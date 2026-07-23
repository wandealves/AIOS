---
Documento: Events
Módulo: 021-Security
Status: Draft
Versão: 0.1
Última atualização: 2026-07-22
Responsável (RACI-A): Arquiteto do Módulo 021-Security
ADRs relacionados: ADR-0210, ADR-0212, ADR-0213, ADR-0217
RFCs relacionados: RFC-0001, RFC-0210
Depende de: _DESIGN_BRIEF.md §6, 020-Communication, 004-API (registro de dataschema), 025-Audit
---

# 021-Security — Catálogo de Eventos NATS

Envelope CloudEvents e convenção de subjects: `../003-RFC/RFC-0001-Architecture-Baseline.md`
§5.2–§5.3, **não redefinidos aqui**. `<dominio>` = **`security`**, valor a ser
registrado no registro de domínios mantido por `../004-API/` conforme RFC-0001 §8
(ratificação em **ADR-0210**).

> **Nota de estado atual.** Subjects deste domínio **já são consumidos** por
> `../004-API/` (`jwks.rotated`, `token.revoked`), `../010-Memory/`
> (`rtbf.requested`) e `../020-Communication/` (`token.revoked`) conforme seus
> respectivos catálogos. O registro formal do domínio apenas normaliza o que a
> arquitetura já pressupõe.

> **Regra absoluta:** nenhum evento deste módulo transporta material secreto. Eventos
> carregam URN, `fingerprint`, motivo e horário — **nunca** token, chave ou segredo.
> Esta regra é verificada por teste automatizado (FR-018 / NFR-008).

---

## 1. Eventos Emitidos

| Subject | `type` | Quando | Stream |
|---------|--------|--------|--------|
| `aios._platform.security.jwks.rotated` | `aios.security.jwks.rotated` | Chave de assinatura rotacionada; JWKS atualizado (UC-009). | `SEC_PLATFORM` |
| `aios._platform.security.ca.rotated` | `aios.security.ca.rotated` | CA intermediária rotacionada. | `SEC_PLATFORM` |
| `aios._platform.security.sandboxprofile.published` | `aios.security.sandboxprofile.published` | Novo perfil de isolamento publicado (UC-011). | `SEC_PLATFORM` |
| `aios.<tenant>.security.credential.issued` | `aios.security.credential.issued` | Credencial → `Active` (T-04). | `SEC_CREDENTIAL` |
| `aios.<tenant>.security.credential.rotating` | `aios.security.credential.rotating` | Rotação iniciada (T-05) — sinal para o consumidor adotar a nova. | `SEC_CREDENTIAL` |
| `aios.<tenant>.security.token.revoked` | `aios.security.token.revoked` | Credencial → `Revoked` (T-07/T-09). | `SEC_CREDENTIAL` |
| `aios.<tenant>.security.credential.expiring` | `aios.security.credential.expiring` | Credencial próxima da expiração sem rotação (FR-019). | `SEC_CREDENTIAL` |
| `aios.<tenant>.security.principal.disabled` | `aios.security.principal.disabled` | Princípio desabilitado; revogação em cascata (UC-007). | `SEC_CREDENTIAL` |
| `aios.<tenant>.security.authn.anomaly` | `aios.security.authn.anomaly` | Sinal de anomalia de autenticação (UC-014). | `SEC_THREAT` |
| `aios.<tenant>.security.attestation.failed` | `aios.security.attestation.failed` | Workload não comprovou identidade (T-03). | `SEC_THREAT` |
| `aios.<tenant>.security.rtbf.requested` | `aios.security.rtbf.requested` | Pedido de direito ao esquecimento (UC-015). | `SEC_PRIVACY` |
| `aios.<tenant>.security.federation.suspended` | `aios.security.federation.suspended` | Confiança federada suspensa/revogada (UC-013). | `SEC_PLATFORM` |

---

## 2. Streams JetStream

| Stream | Subjects | Política | `max_age` | Réplicas |
|--------|----------|----------|-----------|----------|
| `SEC_PLATFORM` | `aios._platform.security.>` | `limits` | 1 ano | 3 |
| `SEC_CREDENTIAL` | `aios.*.security.credential.>`, `…token.>`, `…principal.>` | `limits` | 1 ano | 3 |
| `SEC_THREAT` | `aios.*.security.authn.>`, `…attestation.>` | `limits` | 90 dias | 3 |
| `SEC_PRIVACY` | `aios.*.security.rtbf.>` | `limits` | 7 anos | 3 |

A retenção de 7 anos de `SEC_PRIVACY` acompanha a de `DB_RETENTION` do
`../005-Database/`: o comprovante de um pedido de esquecimento precisa sobreviver ao
dado esquecido.

Declaração dos streams no `../020-Communication/` (`comm.stream`), conforme
`../020-Communication/Database.md` §3.

---

## 3. Schemas de Payload

Todo evento usa o envelope da RFC-0001 §5.2. Abaixo apenas o `data`.

### 3.1 `aios.security.token.revoked`

`dataschema`: `aios://schemas/security.token.revoked/1`

```json
{
  "specversion": "1.0",
  "id": "01J9ZD5R8T0V2X4Z6B8D0F2H4K",
  "source": "urn:aios:acme:service:security",
  "type": "aios.security.token.revoked",
  "subject": "urn:aios:acme:credential:01J9ZD2N5Q7S9U1W3Y5A7C9E1G",
  "time": "2026-07-22T16:12:44.318Z",
  "tenant": "acme",
  "traceparent": "00-4bf92f3577b34da6a3ce929d0e0e4736-00f067aa0ba902b7-01",
  "datacontenttype": "application/json",
  "dataschema": "aios://schemas/security.token.revoked/1",
  "data": {
    "credentialUrn": "urn:aios:acme:credential:01J9ZD2N5Q7S9U1W3Y5A7C9E1G",
    "principalUrn": "urn:aios:acme:principal:01J9Z8Q7B0C2D4E6F8G0H2J4K6",
    "kind": "db_role",
    "fingerprint": "sha256:7c4a…9e21",
    "reason": "compromise",
    "effectiveAt": "2026-07-22T16:12:44Z",
    "credentialExpiresAt": "2026-07-22T17:00:00Z"
  }
}
```

Os consumidores (`004`, `020`, serviços com CRL em cache) invalidam pelo
`fingerprint` — que identifica sem revelar. `credentialExpiresAt` diz por quanto tempo
a entrada precisa permanecer na lista local: depois disso, a credencial expiraria de
qualquer forma.

### 3.2 `aios.security.jwks.rotated`

`dataschema`: `aios://schemas/security.jwks.rotated/1`

```json
{
  "data": {
    "newKid": "01J9ZD1M4P6R8T0V2X4Z6B8D0F",
    "previousKid": "01J9ZC0K3N5Q7R9T1V3X5Z7A9C",
    "algorithm": "ES256",
    "jwksUri": "https://api.aios.local/.well-known/jwks.json",
    "previousRetiresAt": "2026-08-21T16:00:00Z"
  }
}
```

O evento traz apenas **identificadores de chave** — o material público é buscado no
`jwksUri`, e o privado nunca sai do KMS.

### 3.3 `aios.security.credential.rotating`

`dataschema`: `aios://schemas/security.credential.rotating/1`

```json
{
  "data": {
    "credentialUrn": "urn:aios:acme:credential:01J9ZD2N5Q7S9U1W3Y5A7C9E1G",
    "successorUrn": "urn:aios:acme:credential:01J9ZD6S9U1W3Y5A7C9E1G3J5L",
    "kind": "db_role",
    "purpose": "postgres:memory_rw",
    "overlapEndsAt": "2026-07-22T16:20:00Z"
  }
}
```

É o sinal para o consumidor adotar a sucessora **antes** de `overlapEndsAt`. O material
da nova credencial **não** viaja aqui: o consumidor a obtém por emissão própria ou já a
recebeu na resposta de `:rotate`.

### 3.4 `aios.security.attestation.failed`

`dataschema`: `aios://schemas/security.attestation.failed/1`

```json
{
  "data": {
    "requestedPurpose": "mesh:006",
    "declaredService": "006-Kernel",
    "observedService": "099-Unknown",
    "nodeRef": "urn:aios:_platform:node:01J9ZD7T0V2X4Z6B8D0F2H4K6M",
    "sourceAddress": "10.42.7.19",
    "failureKind": "service_identity_mismatch",
    "severity": "P1"
  }
}
```

Este é um dos eventos de segurança mais relevantes do AIOS: divergência entre serviço
declarado e observado indica ou defeito de deploy, ou tentativa de um contêiner assumir
a identidade de outro.

### 3.5 `aios.security.authn.anomaly`

`dataschema`: `aios://schemas/security.authn.anomaly/1`

```json
{
  "data": {
    "principalUrn": "urn:aios:acme:principal:01J9Z8Q6H7K2M4N6P8R0S2T4V6",
    "anomalyKind": "burst_failures",
    "failureCount": 27,
    "windowSeconds": 60,
    "threshold": 5,
    "actionTaken": "lockout",
    "lockoutSeconds": 300
  }
}
```

`actionTaken: "lockout"` é **contenção técnica por princípio**, não decisão de política:
o módulo sinaliza e contém temporariamente; qualquer bloqueio permanente é decisão do
`../022-Policy/` ou de `../029-Operations/` (UC-014 E1).

### 3.6 `aios.security.rtbf.requested`

`dataschema`: `aios://schemas/security.rtbf.requested/1`

```json
{
  "data": {
    "requestUrn": "urn:aios:acme:rtbfrequest:01J9ZD8U1W3Y5A7C9E1G3J5L7N",
    "subjectPrincipalUrn": "urn:aios:acme:principal:01J9Z8Q6H7K2M4N6P8R0S2T4V6",
    "scope": "subject",
    "requestedAt": "2026-07-22T16:30:00Z",
    "targets": ["010-Memory", "005-Database", "025-Audit"]
  }
}
```

Nenhum dado pessoal do titular viaja no evento — apenas o URN opaco. Os módulos donos
resolvem o que apagar (RFC-0001 §7).

---

## 4. Eventos Consumidos

| Subject assinado | Produtor | Ação do módulo | Consumidor durável |
|-------------------|----------|----------------|--------------------|
| `aios.<tenant>.policy.bundle.updated` | `../022-Policy/` | Invalida o cache do `PolicyClient`; reavalia emissões pendentes. | `sec-policy-cache` |
| `aios.<tenant>.agent.lifecycle.terminated` | `../006-Kernel/` | Revoga as credenciais emitidas ao agente extinto. | `sec-agent-cleanup` |
| `aios._platform.cluster.node.decommissioned` | `../027-Cluster/` | Revoga certificados de workload do nó removido. | `sec-node-cleanup` |
| `aios.<tenant>.audit.anomaly.detected` | `../025-Audit/` | Correlaciona com sinais próprios do `ThreatSignalDetector`. | `sec-threat-correlation` |
| `aios._platform.deployment.rollout.started` | `../028-Deployment/` | Suspende rotações durante a janela de rollout. | `sec-rollout-guard` |

Todos os consumidores **DEVEM** ser idempotentes: reprocessar um `event.id` já visto é
operação nula (RFC-0001 §5.5).

---

## 5. Contrato de Consumo — o que os outros módulos esperam

Este módulo é **produtor de eventos críticos de segurança**; falhar em emiti-los tem
consequências diretas em outros módulos:

| Evento | Consumidor | O que quebra se não for emitido |
|--------|-----------|----------------------------------|
| `token.revoked` | `../004-API/`, `../020-Communication/` | Credencial revogada continuaria aceita até expirar — a revogação viraria ficção. |
| `jwks.rotated` | `../004-API/` | Tokens assinados pela nova chave seriam rejeitados até o TTL do cache expirar. |
| `sandboxprofile.published` | `../007-Agent-Runtime/` | Agentes continuariam executando sob perfil antigo, ignorando endurecimento. |
| `credential.rotating` | consumidores de credencial dinâmica | Adoção tardia da sucessora e possível interrupção ao fim da janela. |
| `rtbf.requested` | `../010-Memory/`, `../005-Database/` | Pedido de esquecimento não se propagaria — falha de conformidade. |
| `principal.disabled` | vários | Acesso residual de princípio encerrado. |

É por isso que **todos** são emitidos via Outbox transacional (FR-017): a mudança de
estado e o evento commitam juntos ou não commitam.

---

## 6. Semântica de Entrega

| Aspecto | Garantia |
|---------|----------|
| Entrega | At-least-once via JetStream; efeito único por Outbox + dedupe por `event.id`. |
| Ordenação | Por stream. Revogações de um mesmo princípio são serializadas pela transação. |
| Latência-alvo | `token.revoked` deve alcançar os verificadores em ≤ **30 s** (NFR-006). |
| Versionamento | `dataschema` versionado; evolução aditiva (RFC-0001 §5.7). |
| Privacidade | Sem material secreto, sem PII: apenas URN, `fingerprint`, motivo e horários. |
| Falha de publicação | Evento permanece em `security.outbox` com `published = false`; relay reenvia. |

---

## 7. Exemplo — Consumidor de revogação (padrão para verificadores)

```python
sub = await js.subscribe("aios.acme.security.token.revoked",
                         durable="api-revocation", manual_ack=True)

async for msg in sub.messages:
    event = json.loads(msg.data)
    if seen(event["id"]):            # dedupe obrigatório
        await msg.ack(); continue
    d = event["data"]
    # guarda até a expiração natural — antes disso, esquecer = desfazer a revogação
    revocation_cache.add(d["fingerprint"], until=d["credentialExpiresAt"])
    mark_seen(event["id"])
    await msg.ack()
```

O `until` é essencial: manter a entrada apenas enquanto a credencial ainda seria válida
mantém o cache pequeno **sem** abrir janela de reaceitação.

---

## 8. Referências

- Brief: `./_DESIGN_BRIEF.md` §6
- Contratos de evento: `../003-RFC/RFC-0001-Architecture-Baseline.md` §5.2–§5.3
- Barramento e streams: `../020-Communication/` · Registro de schemas: `../004-API/`
- API: `./API.md` · Sequências: `./SequenceDiagrams.md`
