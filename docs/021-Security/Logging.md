---
Documento: Logging
Módulo: 021-Security
Status: Draft
Versão: 0.1
Última atualização: 2026-07-22
Responsável (RACI-A): Arquiteto do Módulo 021-Security
ADRs relacionados: ADR-0010, ADR-0213, ADR-0219
RFCs relacionados: RFC-0001
Depende de: 024-Observability, 025-Audit, Security.md, Metrics.md
---

# 021-Security — Log Estruturado

Stack: **Serilog → Seq**, com exportação OTel (`../024-Observability/`). Todo log é
estruturado (JSON). Campos de correlação conforme RFC-0001 §5.6, **não redefinidos aqui**.

> **A regra que governa este documento:** o log deste módulo registra **fatos sobre
> credenciais**, nunca **credenciais**. Um sistema de logs que recebe material secreto
> torna-se, ele próprio, um cofre — sem cifragem, com retenção diferente, com acesso
> mais amplo e sem trilha de leitura. É o pior lugar possível para um segredo.

---

## 1. Campos Obrigatórios

| Campo | Origem | Obrigatório | Observação |
|-------|--------|-------------|------------|
| `timestamp` | serviço | sim | RFC 3339, UTC. |
| `level` | serviço | sim | `Verbose`…`Fatal`. |
| `message_template` | Serilog | sim | Template, não a string interpolada. |
| `trace_id` / `span_id` | `traceparent` | sim | Correlação com traces e auditoria. |
| `tenant_id` | requisição | sim em operação com tenant | `_platform` em operação de plataforma. |
| `service` | config | sim | `aios-security-svc` \| `aios-security-ca`. |
| `component` | código | sim | `TokenService`, `CertificateAuthority`, … (PascalCase). |
| `operation` | código | sim | `IssueCredential`, `RevokeCredential`, … |
| `principal_urn` | contexto | quando aplicável | URN opaco — **nunca** nome de usuário ou e-mail. |
| `credential_urn` | contexto | quando aplicável | URN da credencial afetada. |
| `fingerprint` | contexto | quando aplicável | `sha256` — identifica sem revelar. |
| `purpose` | contexto | em emissão/rotação | Propósito declarado. |
| `error_code` | catálogo | em falha | `AIOS-SEC-*` / `AIOS-AUTHN-*`. |
| `duration_ms` | código | em operação concluída | Correlaciona com métricas. |
| `outcome` | código | sim | `ok` \| `denied` \| `failed`. |

Note que `principal_urn` e `fingerprint` **podem** aparecer no log (ao contrário do que
ocorre nas métricas, `./Metrics.md` §0): logs têm acesso restrito e retenção controlada,
e sem esses campos seria impossível investigar um incidente. A restrição nas métricas é
por cardinalidade e por exposição em painéis.

---

## 2. Níveis — critério de uso

| Nível | Quando usar | Exemplos neste módulo |
|-------|-------------|-----------------------|
| `Verbose` | Diagnóstico fino; **desligado** em produção. | Passos internos da validação de cadeia de certificado. |
| `Debug` | Fluxo interno útil em investigação. | Seleção de chave de assinatura; cache de JWKS. |
| `Information` | Fato relevante e esperado. | Credencial emitida; rotação concluída; perfil publicado. |
| `Warning` | Anomalia que ainda não é falha, ou fato de segurança relevante. | Contenção por força bruta; leitura de segredo; revogação; escopo federado negado. |
| `Error` | Operação falhou e exige atenção. | Atestação reprovada; KMS indisponível; propagação de revogação atrasada. |
| `Fatal` | Serviço não pode continuar. | Chave de assinatura `active` ausente na inicialização. |

**Nota sobre `Warning` para operações bem-sucedidas.** Revogação, leitura de segredo e
publicação de perfil são registradas em `Warning` mesmo quando bem-sucedidas. Não é
erro de classificação: são eventos que **merecem atenção em qualquer revisão de log**,
e o nível é o mecanismo mais simples de garantir que apareçam.

---

## 3. Eventos de Log Canônicos

| `operation` | Nível | Campos específicos | Correspondência |
|-------------|-------|--------------------|-----------------|
| `AuthenticatePrincipal` | Information / Warning | `kind`, `flow`, `mfa_used`, `outcome` | UC-001, UC-002 |
| `AuthenticationFailed` | Warning | `reason_internal` (**só no log**, nunca na resposta), `source_address` | UC-001 E1, RN-06 |
| `LockoutApplied` | Warning | `principal_urn`, `failure_count`, `lockout_s` | UC-014 |
| `IssueCredential` | Information | `kind`, `purpose`, `fingerprint`, `ttl_s`, `attested` | UC-003, UC-004 |
| `AttestationFailed` | **Error** | `declared_service`, `observed_service`, `node_ref`, `failure_kind` | UC-003 E1 |
| `RotateCredential` | Information | `credential_urn`, `successor_urn`, `overlap_s`, `trigger` | UC-005 |
| `RotationOverlapExtended` | Warning | `credential_urn`, `reason` (uso recente) | UC-005 A1 |
| `RevokeCredential` | **Warning** | `credential_urn`, `fingerprint`, `reason`, `revoked_by` | UC-006 |
| `RevocationPropagated` | Information | `credential_urn`, `propagation_ms` | NFR-006 |
| `DisablePrincipal` | **Warning** | `principal_urn`, `credentials_revoked`, `reason` | UC-007 |
| `ReadSecret` | **Warning** | `path`, `version_no`, `lease_s` — **nunca o valor** | UC-008 |
| `PutSecret` | Information | `path`, `version_no` | UC-008 |
| `RotateKey` | **Warning** | `role`, `new_kid`, `previous_kid`, `approvers[]` | UC-009, UC-010 |
| `PublishSandboxProfile` | **Warning** | `name`, `profile_version`, `signed_by`, `diff_summary` | UC-011 |
| `PutFederationTrust` | **Warning** | `issuer`, `kind`, `allowed_scopes`, `approved_by` | UC-012, UC-013 |
| `FederationScopeDenied` | Warning | `issuer`, `requested_scope` | UC-012 E2 |
| `PolicyDecision` | Information / Warning | `capability`, `decision`, `deny_reason` | FR-016 |
| `ConfigurationChanged` | **Warning** | `key`, `previous_value`, `new_value`, `changed_by` | `./Configuration.md` §3 |
| `LeakScanFinding` | **Error** | `surface`, `pattern_kind`, `artifact_ref` — **nunca o material encontrado** | FR-018 |

---

## 4. Exemplos

### 4.1 Emissão de credencial

```json
{
  "timestamp": "2026-07-22T16:00:00.412Z",
  "level": "Information",
  "message_template": "Credencial {Kind} emitida para {PrincipalUrn} com TTL {TtlS}s",
  "trace_id": "4bf92f3577b34da6a3ce929d0e0e4736",
  "span_id": "00f067aa0ba902b7",
  "tenant_id": "acme",
  "service": "aios-security-svc",
  "component": "CredentialBroker",
  "operation": "IssueCredential",
  "principal_urn": "urn:aios:acme:principal:01J9Z8Q7B0C2D4E6F8G0H2J4K6",
  "credential_urn": "urn:aios:acme:credential:01J9ZD2N5Q7S9U1W3Y5A7C9E1G",
  "kind": "db_role",
  "purpose": "postgres:memory_rw",
  "fingerprint": "sha256:7c4a…9e21",
  "ttl_s": 3600,
  "attested": true,
  "duration_ms": 62,
  "outcome": "ok"
}
```

Não há campo com a senha, o certificado ou a chave. O `fingerprint` permite correlacionar
esta emissão com qualquer uso ou revogação posterior — que é tudo de que a investigação
precisa.

### 4.2 Falha de autenticação (com o motivo real apenas no log)

```json
{
  "timestamp": "2026-07-22T16:03:11.907Z",
  "level": "Warning",
  "message_template": "Autenticação falhou para {PrincipalRef}",
  "trace_id": "9c1a2b3c4d5e6f708192a3b4c5d6e7f8",
  "tenant_id": "acme",
  "component": "IdentityProvider",
  "operation": "AuthenticationFailed",
  "principal_urn": "urn:aios:acme:principal:01J9Z8Q6H7K2M4N6P8R0S2T4V6",
  "reason_internal": "bad_secret",
  "source_address": "203.0.113.44",
  "error_code": "AIOS-AUTHN-0001",
  "outcome": "failed"
}
```

`reason_internal` distingue `principal_not_found` de `bad_secret` — informação essencial
para investigar, e que **jamais** aparece na resposta HTTP (RN-06, `./API.md` §5.1).
A resposta ao cliente é sempre idêntica, com tempo constante.

### 4.3 Atestação reprovada

```json
{
  "timestamp": "2026-07-22T16:05:41.220Z",
  "level": "Error",
  "message_template": "Atestação falhou: serviço declarado {Declared} × observado {Observed}",
  "trace_id": "1a2b3c4d5e6f708192a3b4c5d6e7f809",
  "tenant_id": "acme",
  "component": "WorkloadAttestor",
  "operation": "AttestationFailed",
  "requested_purpose": "mesh:006",
  "declared_service": "006-Kernel",
  "observed_service": "099-Unknown",
  "node_ref": "urn:aios:_platform:node:01J9ZD7T0V2X4Z6B8D0F2H4K6M",
  "source_address": "10.42.7.19",
  "failure_kind": "service_identity_mismatch",
  "error_code": "AIOS-SEC-0006",
  "outcome": "denied"
}
```

### 4.4 Leitura de segredo

```json
{
  "timestamp": "2026-07-22T16:07:02.115Z",
  "level": "Warning",
  "message_template": "Segredo {Path} v{VersionNo} lido por {PrincipalUrn}",
  "trace_id": "2b3c4d5e6f708192a3b4c5d6e7f80912",
  "tenant_id": "acme",
  "component": "SecretStore",
  "operation": "ReadSecret",
  "principal_urn": "urn:aios:acme:principal:01J9Z8Q7B0C2D4E6F8G0H2J4K6",
  "path": "llm/anthropic/api-key",
  "version_no": 4,
  "lease_s": 3600,
  "duration_ms": 11,
  "outcome": "ok"
}
```

O `path` aparece; o **valor** nunca. Saber que a chave do provedor foi lida — por quem e
quando — é exatamente o que uma investigação de vazamento precisa.

---

## 5. Regras de Privacidade e Confidencialidade

| Regra | Detalhe |
|-------|---------|
| **Sem material secreto** | Token, chave privada, senha, valor de segredo, CSR assinado e conteúdo de certificado **NÃO DEVEM** aparecer em nenhum nível, nem truncados, nem mascarados parcialmente (mascarar parcialmente ainda vaza entropia). |
| **Sem PII** | Identificação por URN opaco. Nome, e-mail e identificador externo do IdP **NÃO DEVEM** ser logados. |
| **`reason_internal` é exclusivo do log** | Nunca ecoado em resposta de API (RN-06). |
| **Sem conteúdo de perfil completo** | `PublishSandboxProfile` registra `diff_summary`, não o JSON inteiro do perfil. |
| **Achados de varredura sem o achado** | `LeakScanFinding` registra `surface` e `pattern_kind`; nunca o material encontrado — do contrário, o log de vazamento seria mais um vazamento. |
| **Sem cabeçalho `Authorization`** | Requisições logadas têm o cabeçalho removido antes da serialização. |

Violações são incidente de segurança (`SecMaterialLeakDetected`, P1), não defeito de
formatação — e a primeira ação é **rotacionar o material exposto**, antes mesmo de
expurgar o log.

---

## 6. Retenção e Amostragem

| Categoria | Nível | Retenção em Seq | Amostragem |
|-----------|-------|-----------------|------------|
| Emissão, rotação, revogação | Information+ | 1 ano | 100% (nunca amostrado) |
| Leitura de segredo, chaves, perfis, federação, configuração | Warning+ | 1 ano | 100% |
| Falhas de autenticação | Warning | 90 dias | 100% até 100/min por origem; acima disso, agregado (evita que uma força bruta vire uma tempestade de logs) |
| Atestação reprovada | Error | 1 ano | 100% |
| Fluxo interno | Debug | 7 dias | 100% em incidente, 1% em regime |
| Diagnóstico fino | Verbose | desligado em produção | — |

**Log não substitui auditoria.** A trilha imutável de operações privilegiadas vive em
`../025-Audit/` (`./Security.md` §9). O log é para diagnóstico — mais rico, mais
volátil, com acesso mais amplo; a auditoria é para **prova** — imutável, encadeada, com
acesso restrito e retenção legal. Confundi-los é o erro que faz uma investigação
depender de um sistema que alguém pode ter apagado.

---

## 7. Correlação

```
   requisição OIDC ──traceparent──▶ span "IssueCredential"
        │                              │
        │                              ├─ span "WorkloadAttestor.Attest"
        │                              ├─ span "PolicyClient.Decide"      (022)
        │                              ├─ span "Kms.Sign"
        │                              └─ span "Outbox.Enqueue"
        ▼
   log (trace_id, fingerprint) ──▶ Seq
        │                                métrica (exemplar) ──▶ Prometheus
        └──────────── auditoria (trace_id, fingerprint) ──▶ 025-Audit
```

O `fingerprint` é o segundo eixo de correlação, específico deste módulo: ele liga a
**emissão**, todo **uso** subsequente registrado pelos verificadores e a eventual
**revogação** — sem que nenhum desses registros contenha o material.

---

## 8. Referências

- Métricas: `./Metrics.md` · Alertas: `./Monitoring.md`
- Segurança e auditoria: `./Security.md` · `../025-Audit/`
- Correlação (contrato): `../003-RFC/RFC-0001-Architecture-Baseline.md` §5.6
