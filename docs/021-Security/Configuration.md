---
Documento: Configuration
Módulo: 021-Security
Status: Draft
Versão: 0.1
Última atualização: 2026-07-22
Responsável (RACI-A): Arquiteto do Módulo 021-Security
ADRs relacionados: ADR-0211, ADR-0213, ADR-0214, ADR-0215, ADR-0216, ADR-0218, ADR-0219
RFCs relacionados: RFC-0001, RFC-0210, RFC-0211
Depende de: _DESIGN_BRIEF.md §8, Deployment.md, 022-Policy, 029-Operations
---

# 021-Security — Configuração

Prefixo de variável de ambiente: **`AIOS_SEC_`**. A chave `sec.token.access_ttl_s`
corresponde a `AIOS_SEC_TOKEN_ACCESS_TTL_S` (pontos → `_`, maiúsculas).

Escopos: `global`, `tenant`, `principal`, `credential_kind`. Os valores abaixo são
idênticos aos de `./_DESIGN_BRIEF.md` §8 — divergência é defeito.

> **Este módulo tem uma particularidade de configuração:** várias chaves são
> **não-recarregáveis** por segurança, não por limitação técnica. Alterar em runtime a
> exigência de atestação ou a rotação da KEK seria um caminho de ataque — se um
> invasor consegue mudar a configuração, deveria pelo menos ter de reiniciar o serviço
> (evento visível) para que a mudança valha.

---

## 1. Catálogo de Chaves

### 1.1 Tokens

| Chave | Tipo | Default | Faixa | Escopo | Recarregável | Descrição |
|-------|------|---------|-------|--------|--------------|-----------|
| `sec.token.access_ttl_s` | int | 900 | 60–3600 | global/tenant | sim | TTL do *access token* (15 min). |
| `sec.token.refresh_ttl_s` | int | 86400 | 3600–2592000 | global/tenant | sim | TTL do *refresh token* (24 h). |
| `sec.token.max_ttl_s` | int | 3600 | 60–86400 | global | sim | Teto absoluto para qualquer token (`AIOS-SEC-0004`). |

O TTL de 15 min do *access token* é o outro lado da moeda de ADR-0211: como a validação
é local, ele **é** a janela máxima de uso de um token revogado antes de a lista de
revogação ser consultada. Ampliá-lo amplia essa janela.

### 1.2 Certificados e credenciais

| Chave | Tipo | Default | Faixa | Escopo | Recarregável | Descrição |
|-------|------|---------|-------|--------|--------------|-----------|
| `sec.cert.workload_ttl_s` | int | 86400 | 3600–604800 | global | sim | TTL de certificado de workload (24 h). |
| `sec.cert.renew_before_ratio` | float | 0.5 | 0.1–0.9 | global | sim | Fração do TTL a partir da qual renova. |
| `sec.credential.rotation_overlap_s` | int | 300 | 0–86400 | credential_kind | sim | Janela de sobreposição em `Rotating` (invariante I2). |
| `sec.credential.db_role_ttl_s` | int | 3600 | 300–86400 | credential_kind | sim | TTL de credencial dinâmica de PostgreSQL. |
| `sec.credential.nats_account_ttl_s` | int | 604800 | 3600–2592000 | credential_kind | sim | TTL de conta NKey/JWT do NATS. |
| `sec.credential.max_active_per_principal` | int | 32 | 1–1024 | principal | sim | Limite de credenciais ativas (`AIOS-SEC-0013`). |

### 1.3 Segredos

| Chave | Tipo | Default | Faixa | Escopo | Recarregável | Descrição |
|-------|------|---------|-------|--------|--------------|-----------|
| `sec.secret.default_lease_ttl_s` | int | 3600 | 60–86400 | tenant | sim | Duração default do *lease* de leitura. |
| `sec.secret.rotation_period_days` | int | 90 | 1–365 | tenant | sim | Periodicidade default de rotação. |

### 1.4 Chaves e criptografia

| Chave | Tipo | Default | Faixa | Escopo | Recarregável | Descrição |
|-------|------|---------|-------|--------|--------------|-----------|
| `sec.key.signing_rotation_days` | int | 90 | 7–730 | global | sim | Rotação da chave de assinatura de token. |
| `sec.key.kek_rotation_days` | int | 365 | 30–1825 | global | **não** | Rotação da KEK (exige cerimônia, UC-010). |
| `sec.key.retire_grace_days` | int | 30 | 1–365 | global | sim | Permanência em `retired` antes de `destroyed`. |
| `sec.key.hsm_enabled` | bool | `false` | {true,false} | global | **não** | Usa HSM para KEK e CA raiz. |
| `sec.key.allowed_suites` | lista | `["ES256","EdDSA","AES-256-GCM"]` | lista | global | sim | Suítes permitidas (`AIOS-SEC-0011`). |

### 1.5 Autenticação

| Chave | Tipo | Default | Faixa | Escopo | Recarregável | Descrição |
|-------|------|---------|-------|--------|--------------|-----------|
| `sec.authn.max_failures_per_min` | int | 5 | 1–100 | principal | sim | Limiar de contenção (`AIOS-AUTHN-0005`, NFR-014). |
| `sec.authn.lockout_s` | int | 300 | 30–86400 | principal | sim | Duração da contenção após o limiar. |
| `sec.authn.mfa_required_for_human` | bool | `true` | {true,false} | tenant | sim | Exige segundo fator para princípios humanos. |
| `sec.authn.clock_skew_s` | int | 60 | 0–300 | global | sim | Tolerância de dessincronia de relógio na validação. |

> **Aviso normativo sobre `clock_skew_s`.** Ampliar essa tolerância **NÃO DEVE** ser
> usado como correção para relógios dessincronizados: cada segundo adicional é um
> segundo a mais em que um token expirado continua aceito. A correção é NTP.

### 1.6 Revogação e atestação

| Chave | Tipo | Default | Faixa | Escopo | Recarregável | Descrição |
|-------|------|---------|-------|--------|--------------|-----------|
| `sec.revocation.propagation_target_s` | int | 30 | 1–300 | global | sim | Meta de propagação (NFR-006). |
| `sec.revocation.list_cache_ttl_s` | int | 10 | 1–60 | global | sim | TTL do cache da lista de revogação nos verificadores. |
| `sec.attestation.required` | bool | `true` | {true,false} | global | **não** | Exige atestação antes de emitir credencial de workload. |

`sec.attestation.required` é não-recarregável **e** não deveria ser `false` em nenhum
ambiente conectado: desligá-la remove o único controle que impede um contêiner
comprometido de solicitar a credencial de outro serviço.

### 1.7 Federação, política e isolamento

| Chave | Tipo | Default | Faixa | Escopo | Recarregável | Descrição |
|-------|------|---------|-------|--------|--------------|-----------|
| `sec.federation.max_scope_expansion` | int | 0 | 0–5 | tenant | sim | Escopos adicionais que identidade federada pode ganhar. **0 = nenhum** (ADR-0218). |
| `sec.policy.fail_mode` | enum | `closed` | closed\|open | global | sim | Comportamento se o PDP estiver indisponível — aplica-se a operações administrativas, **não** à validação. |
| `sec.sandbox.default_profile` | string | `agent-default` | nome | tenant | sim | Perfil aplicado quando o agente não especifica outro. |

---

## 2. Segredos e Material de Bootstrap

| Item | Origem | Rotação |
|------|--------|---------|
| Chave da **CA raiz** | Custódia *offline* / HSM. **Nunca** em arquivo ou variável. | Cerimônia plurianual com aprovação dupla. |
| KEK | HSM (se `hsm_enabled`) ou material envolvido em custódia inicial. | `sec.key.kek_rotation_days` (365), por cerimônia (UC-010). |
| Credencial de banco do próprio módulo (`security_rw`) | Provisionada na instalação — é a **única credencial de bootstrap** do AIOS. | Rotação manual documentada em `../029-Operations/`. |
| Certificado inicial do `aios-security-svc` | Emitido na instalação a partir da raiz. | Renovação automática após o bootstrap. |

A credencial de bootstrap é a resposta à circularidade inevitável: o módulo que emite
credenciais precisa de uma credencial que não pode emitir para si mesmo. Ela é única,
documentada, de rotação manual e monitorada como item de risco permanente.

---

## 3. Precedência de Configuração

```
   valor efetivo = principal | credential_kind (mais específico)
                 ▲ tenant
                 ▲ global
                 ▲ default do código (menos específico)
```

Chave de escopo `global` **NÃO PODE** ser sobrescrita por tenant. Chaves
não-recarregáveis alteradas em runtime ficam pendentes até o reinício e aparecem em
`aios_sec_config_pending_reload`.

**Toda alteração de configuração deste módulo é auditada** em `../025-Audit/` com autor,
valor anterior e novo — inclusive as recarregáveis. Configuração de segurança alterada
sem trilha é indistinguível de comprometimento.

---

## 4. Exemplo — `appsettings.Production.json`

```json
{
  "Aios": {
    "Sec": {
      "Token":      { "AccessTtlS": 900, "RefreshTtlS": 86400, "MaxTtlS": 3600 },
      "Cert":       { "WorkloadTtlS": 86400, "RenewBeforeRatio": 0.5 },
      "Credential": { "RotationOverlapS": 300, "DbRoleTtlS": 3600,
                      "NatsAccountTtlS": 604800, "MaxActivePerPrincipal": 32 },
      "Secret":     { "DefaultLeaseTtlS": 3600, "RotationPeriodDays": 90 },
      "Key":        { "SigningRotationDays": 90, "KekRotationDays": 365,
                      "RetireGraceDays": 30, "HsmEnabled": true,
                      "AllowedSuites": ["ES256", "EdDSA", "AES-256-GCM"] },
      "Authn":      { "MaxFailuresPerMin": 5, "LockoutS": 300,
                      "MfaRequiredForHuman": true, "ClockSkewS": 60 },
      "Revocation": { "PropagationTargetS": 30, "ListCacheTtlS": 10 },
      "Attestation":{ "Required": true },
      "Federation": { "MaxScopeExpansion": 0 },
      "Policy":     { "FailMode": "closed" },
      "Sandbox":    { "DefaultProfile": "agent-default" }
    }
  }
}
```

Equivalente por ambiente:

```bash
export AIOS_SEC_TOKEN_ACCESS_TTL_S=900
export AIOS_SEC_CERT_WORKLOAD_TTL_S=86400
export AIOS_SEC_ATTESTATION_REQUIRED=true
export AIOS_SEC_FEDERATION_MAX_SCOPE_EXPANSION=0
export AIOS_SEC_POLICY_FAIL_MODE=closed
```

---

## 5. Configurações Proibidas em Produção

Estas combinações **NÃO DEVEM** existir em ambiente conectado e são verificadas por
job de conformidade (alerta P1 em `./Monitoring.md`):

| Configuração | Por quê |
|--------------|---------|
| `sec.attestation.required = false` | Remove o controle contra falsificação de identidade de workload. |
| `sec.key.hsm_enabled = false` **com** `ca_root` no banco | A chave da raiz não deve residir em material recuperável de um dump. |
| `sec.authn.mfa_required_for_human = false` | Princípio humano sem segundo fator em plataforma com acesso privilegiado. |
| `sec.federation.max_scope_expansion > 0` sem justificativa registrada | Converte confiança externa em privilégio interno. |
| `sec.token.max_ttl_s > 86400` | Aproxima-se de credencial de longa vida (viola P-01). |
| `sec.policy.fail_mode = open` | Permitiria operações administrativas sem decisão do PDP. |
| `sec.authn.clock_skew_s > 300` | Amplia a janela de aceitação de token expirado. |

---

## 6. Relação Configuração → Requisito

| Chave | Requisito que sustenta |
|-------|------------------------|
| `sec.token.max_ttl_s`, `sec.cert.workload_ttl_s` | FR-007, NFR-010 |
| `sec.credential.rotation_overlap_s` | FR-005, invariante I2 |
| `sec.credential.max_active_per_principal` | `AIOS-SEC-0013`, NFR-007 |
| `sec.secret.default_lease_ttl_s` | FR-008 |
| `sec.key.signing_rotation_days`, `sec.key.kek_rotation_days` | FR-002, FR-009, NFR-011 |
| `sec.key.retire_grace_days` | UC-009, ciclo de `KeyMaterial` |
| `sec.key.allowed_suites` | FR-015 |
| `sec.authn.max_failures_per_min`, `sec.authn.lockout_s` | FR-013, NFR-014 |
| `sec.revocation.propagation_target_s`, `…list_cache_ttl_s` | FR-006, NFR-006 |
| `sec.attestation.required` | FR-003, NFR-008 |
| `sec.federation.max_scope_expansion` | FR-011, ADR-0218 |
| `sec.policy.fail_mode` | FR-016 |
| `sec.sandbox.default_profile` | FR-010, NFR-015 |

---

## 7. Referências

- Brief: `./_DESIGN_BRIEF.md` §8
- Implantação: `./Deployment.md` · Segurança: `./Security.md`
- Cerimônias e runbooks: `../029-Operations/`
