---
Documento: Events
Módulo: 022-Policy
Status: Draft
Versão: 0.1
Última atualização: 2026-07-23
Responsável (RACI-A): Arquiteto do Módulo 022-Policy
ADRs relacionados: ADR-0004, ADR-0010, ADR-0220, ADR-0221, ADR-0227
RFCs relacionados: RFC-0001, RFC-0220, RFC-0221
Depende de: _DESIGN_BRIEF.md §6, 020-Communication, 004-API (registro de domínios), 025-Audit
---

# 022-Policy — Catálogo de Eventos

> Envelope **CloudEvents** e convenção de subjects conforme
> `../003-RFC/RFC-0001-Architecture-Baseline.md` §5.2–§5.3 — **referenciados, não
> redefinidos**. Domínio `<dominio>` = **`policy`**, já previsto na RFC-0001 §5.3, cujo
> subject `aios.<tenant>.policy.decision.denied` é exemplo normativo da própria RFC.

## 1. Regra de conteúdo

**Nenhum evento deste módulo transporta valor de atributo marcado como `sensitive`,
nem o conteúdo do recurso avaliado.** Eventos carregam URN, ação, `reason_code`,
`rule_key`, versão de bundle e horários. Essa regra é verificável (FR-023) e vale
inclusive para o payload de `policy.decision.denied`, que é o mais tentador a violá-la
— um evento de negação com o dado que motivou a negação vazaria exatamente o que a
política protege.

## 2. Eventos emitidos

| Subject | `type` | Quando | Stream | Consumidores conhecidos |
|---------|--------|--------|--------|-------------------------|
| `aios.<tenant>.policy.bundle.updated` | `aios.policy.bundle.updated` | Bundle → `Active` (T-08/T-09/T-12). Invalidação **total** de cache. | `POLICY_BUNDLE` | `004`, `006`, `007`, `009`, `010`, `011`, `015`, `017`, `020`, `021` |
| `aios.<tenant>.policy.bundle.rolledback` | `aios.policy.bundle.rolledback` | Bundle ativo revertido (T-11). | `POLICY_BUNDLE` | `024`, `025`, `029` |
| `aios.<tenant>.policy.bundle.rejected` | `aios.policy.bundle.rejected` | Candidato reprovado (T-04/T-06). | `POLICY_BUNDLE` | `024`, `032` |
| `aios.<tenant>.policy.decision.updated` | `aios.policy.decision.updated` | Mudança **granular**: vínculo criado/removido, exceção concedida/revogada, papel alterado. | `POLICY_DECISION` | `004`, `006`, `007`, `011` |
| `aios.<tenant>.policy.decision.denied` | `aios.policy.decision.denied` | Decisão `deny` emitida (amostrada para `no_matching_rule`; **sempre** para `explicit_deny` e `platform_deny`). | `POLICY_DECISION` | `024`, `025` |
| `aios.<tenant>.policy.exception.granted` | `aios.policy.exception.granted` | Waiver concedido. | `POLICY_GOVERNANCE` | `025`, `029`, `032` |
| `aios.<tenant>.policy.exception.expired` | `aios.policy.exception.expired` | Waiver expirou ou foi revogado. | `POLICY_GOVERNANCE` | `025`, `029` |
| `aios.<tenant>.policy.simulation.completed` | `aios.policy.simulation.completed` | Simulação concluída com diferencial. | `POLICY_GOVERNANCE` | `032`, `029` |
| `aios._platform.policy.attribute.degraded` | `aios.policy.attribute.degraded` | Fonte `required` degradada. | `POLICY_GOVERNANCE` | `024`, `029` |

> `bundle.updated` e `decision.updated` são **deliberadamente distintos**: o primeiro
> é a bomba de vácuo (raro, invalida tudo do tenant); o segundo é o bisturi
> (frequente, invalida o afetado). Ter apenas o primeiro faria cada concessão de papel
> derrubar o cache de todos os PEPs do tenant — e o custo apareceria como um salto de
> latência global a cada operação administrativa rotineira.

## 3. Schemas (payload `data`)

Envelope completo conforme RFC-0001 §5.2; abaixo apenas o campo `data`.

### 3.1 `aios.policy.bundle.updated`

```json
{
  "specversion": "1.0",
  "id": "01J9ZG00000000000000000000",
  "source": "urn:aios:acme:service:policy",
  "type": "aios.policy.bundle.updated",
  "subject": "urn:aios:acme:policy:01J9ZC00000000000000000000",
  "time": "2026-07-23T10:20:00.000Z",
  "tenant": "acme",
  "traceparent": "00-4bf92f3577b34da6a3ce929d0e0e4736-00f067aa0ba902b7-01",
  "datacontenttype": "application/json",
  "dataschema": "aios://schemas/policy.bundle.updated/1",
  "data": {
    "bundleUrn": "urn:aios:acme:policy:01J9ZC00000000000000000000",
    "bundleVersion": 43,
    "previousVersion": 42,
    "scope": "tenant",
    "compiledDigest": "sha256:9c1e…",
    "reason": "publish",
    "activatedAt": "2026-07-23T10:20:00.000Z"
  }
}
```

`reason ∈ {publish, rollback, bootstrap}`. Consumidores **DEVEM** invalidar todo o
cache de decisão do tenant e **DEVERIAM** registrar a versão adotada (métrica
`aios_pol_active_bundle_version` do lado do PEP).

### 3.2 `aios.policy.decision.updated`

```json
{ "data": {
    "changeKind": "role_binding_created",
    "subjectUrn": "urn:aios:acme:principal:01J9ZE00000000000000000000",
    "scopePattern": "urn:aios:acme:memory:*",
    "bundleVersion": 43,
    "effectiveAt": "2026-07-23T10:25:00.000Z" } }
```

`changeKind ∈ {role_binding_created, role_binding_deleted, role_updated,
exception_granted, exception_revoked, exception_expired}`. Consumidores **DEVEM**
invalidar apenas as entradas de cache do `subjectUrn` indicado.

### 3.3 `aios.policy.decision.denied`

```json
{ "data": {
    "decisionId": "01J9ZB00000000000000000000",
    "subjectUrn": "urn:aios:acme:agent:01J9Z8Q6H7K2M4N6P8R0S2T4V6",
    "action": "tool:external-http:invoke",
    "resourceUrn": "urn:aios:acme:tool:01J9ZA00000000000000000000",
    "reasonCode": "explicit_deny",
    "matchedRules": ["tool.external.block-unapproved"],
    "bundleVersion": 43,
    "pepModule": "006-Kernel",
    "decidedAt": "2026-07-23T10:26:00.100Z" } }
```

**Não** contém atributos, claims nem conteúdo do recurso (§1).

### 3.4 `aios.policy.exception.granted` / `.expired`

```json
{ "data": {
    "exceptionUrn": "urn:aios:acme:policy:01J9ZH00000000000000000000",
    "subjectUrn": "urn:aios:acme:principal:01J9ZE00000000000000000000",
    "actionPattern": "db:migration:apply",
    "resourcePattern": "urn:aios:acme:*",
    "requestedBy": "urn:aios:acme:principal:01J9ZE00000000000000000000",
    "approvedBy": "urn:aios:acme:principal:01J9ZK00000000000000000000",
    "effectiveFrom": "2026-07-23T11:00:00.000Z",
    "expiresAt": "2026-07-24T11:00:00.000Z" } }
```

Em `.expired`, acrescenta `"terminatedBy": "expiry" | "revocation"` e `"revokedBy"`
quando aplicável.

### 3.5 `aios.policy.simulation.completed`

```json
{ "data": {
    "simulationUrn": "urn:aios:acme:policy:01J9ZF00000000000000000000",
    "candidateBundleVersion": 43, "baselineBundleVersion": 42,
    "evaluatedCount": 842391, "flippedToDeny": 12, "flippedToAllow": 0,
    "truncated": false,
    "topRules": [ { "ruleKey": "tool.external.block-unapproved", "flippedToDeny": 12 } ] } }
```

### 3.6 `aios.policy.attribute.degraded`

```json
{ "data": {
    "sourceKey": "budget", "providerModule": "026-Cost-Optimizer",
    "criticality": "required", "status": "degraded",
    "failureRate": 0.42, "observedFrom": "2026-07-23T10:30:00.000Z" } }
```

Emitido no tenant reservado `_platform` por ser condição de plataforma, não de tenant.

## 4. Eventos consumidos

| Subject assinado | Produtor | Ação do módulo |
|-------------------|----------|----------------|
| `aios.<tenant>.security.principal.disabled` | `../021-Security/` | Expira vínculos e exceções do princípio; emite `policy.decision.updated`. |
| `aios.<tenant>.security.token.revoked` | `../021-Security/` | Invalida decisões associadas à credencial revogada. |
| `aios.<tenant>.security.authn.anomaly` | `../021-Security/` | Atualiza o atributo `threat.*` do sujeito (insumo ABAC). |
| `aios.<tenant>.cost.budget.exhausted` | `../026-Cost-Optimizer/` | Atualiza `budget.*`; regras dependentes passam a negar sem esperar o TTL. |
| `aios.<tenant>.agent.lifecycle.terminated` | `../006-Kernel/` | Expira vínculos e exceções do agente extinto. |
| `aios.<tenant>.audit.anomaly.detected` | `../025-Audit/` | Atualiza atributo de risco; PODE disparar reavaliação de `allow` cacheados. |
| `aios._platform.deployment.rollout.started` | `../028-Deployment/` | Suspende publicações automáticas de bundle durante a janela de rollout. |

Todo consumo é **idempotente** por `event.id` (RFC-0001 §5.5). Um
`principal.disabled` reprocessado não pode reabrir vínculos já expirados.

## 5. Semântica de entrega e streams

| Stream | Subjects | Retenção | Réplicas | Semântica |
|--------|----------|----------|----------|-----------|
| `POLICY_BUNDLE` | `aios.*.policy.bundle.*` | 90 dias | 3 | at-least-once; ordem por subject |
| `POLICY_DECISION` | `aios.*.policy.decision.*` | 7 dias | 3 | at-least-once; alto volume, amostrado |
| `POLICY_GOVERNANCE` | `aios.*.policy.exception.*`, `aios.*.policy.simulation.*`, `aios.*.policy.attribute.*` | 365 dias | 3 | at-least-once; retenção longa por governança |

- Publicação via **Outbox transacional** (`policy.outbox`, `./Database.md` §9): a
  transição de estado e o evento são gravados na mesma transação (invariante I4) e
  publicados pelo `policy-outbox-relay`.
- Consumidores **DEVEM** deduplicar por `event.id` e **DEVEM** tolerar campos
  desconhecidos (evolução aditiva, RFC-0001 §5.7).
- `aios.<tenant>.policy.decision.request` **não** é um evento: é um subject de
  **request-reply** (`../020-Communication/`), sem stream e sem retenção.

## 6. Ordenação e reprocessamento

```
   publicação v42 ──▶ publicação v43 ──▶ rollback para v42
        │                   │                    │
        └──── ordem por subject dentro do stream POLICY_BUNDLE ────┘

   Consumidor que processa fora de ordem DEVE comparar `bundleVersion`
   e ignorar evento com versão ANTERIOR à que já adotou, EXCETO quando
   `reason = "rollback"` — nesse caso a versão menor é a correta.
```

> Essa é a única inversão legítima de versão no sistema. Um consumidor que aplique
> "sempre a maior versão" desfaz silenciosamente um rollback — e o rollback existe
> justamente porque a versão maior estava errada.

## 7. Versionamento de schema

| `dataschema` | Versão | Mudanças permitidas |
|--------------|--------|---------------------|
| `aios://schemas/policy.bundle.updated/1` | 1 | Aditivas (novos campos opcionais) |
| `aios://schemas/policy.decision.updated/1` | 1 | Novos valores de `changeKind` (aditivos) |
| `aios://schemas/policy.decision.denied/1` | 1 | Novos `reasonCode` (aditivos) |
| `aios://schemas/policy.exception.granted/1` | 1 | Aditivas |
| `aios://schemas/policy.simulation.completed/1` | 1 | Aditivas |
| `aios://schemas/policy.attribute.degraded/1` | 1 | Aditivas |

Mudança incompatível exige nova versão de schema e coexistência (RFC-0001 §5.7);
registro em `../004-API/Events.md` (RFC-0001 §8).

## 8. Referências

- Fonte única de verdade: `./_DESIGN_BRIEF.md` §6
- Envelope e subjects: `../003-RFC/RFC-0001-Architecture-Baseline.md` §5.2–§5.3
- Barramento e streams: `../020-Communication/`
- Registro de domínios e schemas: `../004-API/Events.md`
- Outbox: `./Database.md` §9 · FSM: `./StateMachine.md`

*Fim de `Events.md`.*
