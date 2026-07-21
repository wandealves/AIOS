---
Documento: Events
Módulo: 011-Context
Status: Draft
Versão: 0.1
Última atualização: 2026-07-20
Responsável (RACI-A): Arquiteto do Módulo 011-Context
ADRs relacionados: ADR-0111, ADR-0112, ADR-0116, ADR-0119
RFCs relacionados: RFC-0001 (baseline), RFC-0011 (proposta)
Depende de: 003-RFC (RFC-0001), 004-API, 010-Memory, 017-Model-Router, 018/019-Knowledge, 020-Communication, 022-Policy, 024-Observability, 025-Audit, 040-Glossary
---

# 011-Context — Events

> Catálogo de eventos NATS do `ContextService`. **Deriva** da §6 do
> `_DESIGN_BRIEF.md` e **NÃO PODE contradizê-lo**. O envelope de evento
> (CloudEvents), a convenção de subjects e a semântica de idempotência são
> **reutilizados** de [RFC-0001 §5.2/§5.3/§5.5](../003-RFC/RFC-0001-Architecture-Baseline.md)
> — aqui apenas os aplicamos. Palavras normativas conforme RFC 2119/8174.

## 1. Convenções de barramento

- **Domínio reservado:** `context` (brief §6). Subjects seguem
  `aios.<tenant>.context.<entidade>.<acao>` (RFC-0001 §5.3), com namespace de
  tenant isolando o tráfego (contas NATS por tenant).
- **Envelope:** todo evento usa o envelope CloudEvents 1.0 de RFC-0001 §5.2
  (`specversion`, `id`, `source`, `type`, `subject`, `time`, `tenant`,
  `traceparent`, `datacontenttype`, `dataschema`, `data`).
- **Publicação:** exclusivamente via **Outbox** (`EventPublisher`) — linha de
  banco gravada na mesma transação da persistência do `ContextBundle`
  (transição `ASSEMBLED→SERVED`) ou da mutação de cache/budget, e relay para
  NATS/JetStream (atomicidade DB+evento, brief §2.1/§9-FM07).
- **Entrega:** **at-least-once**. Consumidores DEVEM deduplicar por
  `event.id` (RFC-0001 §5.5). Efeito *exactly-once* obtido por at-least-once +
  idempotência (ver [040-Glossary](../040-Glossary/Glossary.md)).
- **Transporte:** stream JetStream durável; mensagens envenenadas vão para
  **DLQ** após N falhas. Ver [020-Communication](../020-Communication/Events.md).

```
 Pipeline ContextAssembler (ASSEMBLED → SERVED, ou mutação de cache/budget)
        │  (mesma transação lógica)
   ┌────▼─────────┐    relay at-least-once    ┌──────────────┐
   │ OutboxEvent   │──────────────────────────▶│ NATS JetStream│──▶ consumidores (dedup por event.id)
   │ status=pending│      (marca published)     │ aios.<t>.context.*│
   └───────────────┘                            └──────┬────────┘
                                                       │ N falhas
                                                       ▼  DLQ
```

## 2. Versionamento de schema

- Cada evento referencia um `dataschema` versionado
  (`aios://schemas/context.<entidade>.<acao>/<v>`); a evolução DEVE ser
  **aditiva** e consumidores DEVEM tolerar campos desconhecidos (RFC-0001
  §5.7).
- Uma mudança incompatível cria uma nova versão de schema e coexiste com a
  anterior; o registro central fica em
  [004-API/Events.md](../004-API/Events.md).
- **Minimização (LGPD/GDPR):** o `data` carrega o **mínimo** necessário
  (identificadores, contadores, scores) e referencia fragmentos por URN — o
  conteúdo textual de fragmentos/respostas cacheadas **NÃO DEVE** trafegar no
  envelope de evento, apenas em `content_ref`/`response_ref` sob controle de
  acesso (RFC-0001 §7; brief §12.3; ver [Security.md](./Security.md)).

## 3. Eventos emitidos (produtor = 011-Context)

Derivado da §6.1 do brief. `<t>` = slug do tenant.

| Subject | `type` | Gatilho (estado) | Consumidores | Ordenação |
|---------|--------|-------------------|---------------|-----------|
| `aios.<t>.context.window.assembled` | `aios.context.window.assembled` | Bundle montado (`ASSEMBLED→SERVED`) | 024, 026, 035 | por `agent_urn` |
| `aios.<t>.context.window.compressed` | `aios.context.window.compressed` | Compressão aplicada a fragmentos (`COMPRESSING`) | 024, 023 | por `bundle_urn` |
| `aios.<t>.context.window.cache_hit` | `aios.context.window.cache_hit` | Lookup resolvido por cache (`CACHE_LOOKUP→SERVED`) | 024, 026 | por `agent_urn` |
| `aios.<t>.context.window.cache_miss` | `aios.context.window.cache_miss` | Lookup sem hit válido (`CACHE_LOOKUP→BUDGET_ALLOCATED`) | 024 | best-effort |
| `aios.<t>.context.window.rejected` | `aios.context.window.rejected` | Orçamento infeasível/authZ deny (`*→REJECTED`) | 024, 025 | por `agent_urn` |
| `aios.<t>.context.cache.evicted` | `aios.context.cache.evicted` | Entrada removida (TTL/LRU/expurgo, `→EVICTED`) | 024, 025 | por `cache_id` |
| `aios.<t>.context.cache.invalidated` | `aios.context.cache.invalidated` | Entrada invalidada por mudança de origem (`→INVALIDATED`) | 025 | por `cache_id` |
| `aios.<t>.context.budget.updated` | `aios.context.budget.updated` | `BudgetProfile` alterado (upsert) | 024, 025 | por `profile_id` |

### 3.1 Ordenação e particionamento

- Não há ordem total global. A ordem **relativa por chave** é preservada
  dentro do shard (`hash(tenant_id, agent_id) mod context.shard.count`, brief
  §10). Eventos de janela (`window.*`) usam `agent_urn` como chave de
  partição; eventos de cache (`cache.*`) usam `cache_id`; eventos de budget
  usam `profile_id`.
- `window.cache_miss` é emitido em **melhor esforço** (telemetria de taxa de
  acerto) — consumidores NÃO DEVEM assumir contagem 1:1 com chamadas de
  `assemble` sob alta carga (amostragem configurável, ver
  [Configuration.md](./Configuration.md)).

## 4. Eventos consumidos (011-Context como consumidor)

Derivado da §6.2 do brief.

| Subject (assinatura) | Produtor | Ação do `ContextService` |
|------------------------|----------|-----------------------------|
| `aios.<t>.memory.item.consolidated` | 010-Memory | Invalida/atualiza cache e `SummaryNode` que dependem do item (`source_urn` casado). |
| `aios.<t>.memory.item.deleted` | 010-Memory | Invalida cache + expurgo rastreável (LGPD/RTBF). |
| `aios.<t>.model.registry.updated` | 017-Model-Router | Atualiza limites/tokenizers cacheados; invalida budget derivado do modelo. |
| `aios.<t>.knowledge.node.updated` | 018/019-Knowledge | Invalida cache derivado de fragmentos de conhecimento (`source_kind=knowledge`). |
| `aios.<t>.agent.lifecycle.suspended` | 006/008 | Evict de contexto/cache com escopo `agent` (opcional, configurável). |
| `aios.<t>.agent.lifecycle.terminated` | 006/008 | Expurgo de `ContextBundle`/`ContextFragment` efêmeros do agente. |
| `aios.<t>.policy.decision.updated` | 022-Policy | Recarrega limites/decisões de política aplicáveis ao `ContextPolicyGuard`. |

Assinaturas usam wildcards NATS por domínio quando aplicável (ex.:
`aios.acme.agent.lifecycle.>`). O consumo é idempotente: dedup por `event.id`
+ idempotência das operações internas (brief §9-FM07).

## 5. Schemas JSON (envelope + `data`)

O envelope é o de RFC-0001 §5.2; abaixo detalha-se o `data` por evento.

### 5.1 `context.window.assembled`

```json
{
  "specversion": "1.0",
  "id": "01J9Z8QMEVT0ASSEMBLED0001",
  "source": "urn:aios:acme:service:context",
  "type": "aios.context.window.assembled",
  "subject": "urn:aios:acme:context:01J9Z8QHBUNDLE0000000001",
  "time": "2026-07-20T12:34:56.789Z",
  "tenant": "acme",
  "traceparent": "00-4bf92f3577b34da6a3ce929d0e0e4736-00f067aa0ba902b7-01",
  "datacontenttype": "application/json",
  "dataschema": "aios://schemas/context.window.assembled/1",
  "data": {
    "bundleId": "01J9Z8QHBUNDLE0000000001",
    "agentUrn": "urn:aios:acme:agent:01J9Z8Q6H7K2M4N6P8R0S2T4V6",
    "modelId": "gpt-x-mini",
    "tokensRaw": 5230,
    "tokensIn": 2380,
    "compressionRatio": 0.5450,
    "budget": 12000
  }
}
```

### 5.2 `context.window.compressed`

```json
{
  "type": "aios.context.window.compressed",
  "dataschema": "aios://schemas/context.window.compressed/1",
  "data": {
    "bundleId": "01J9Z8QHBUNDLE0000000001",
    "method": "abstractive",
    "tokensBefore": 940,
    "tokensAfter": 210,
    "ratio": 0.7766
  }
}
```

### 5.3 `context.window.cache_hit` e `context.window.cache_miss`

```json
{
  "type": "aios.context.window.cache_hit",
  "dataschema": "aios://schemas/context.window.cache_hit/1",
  "data": {
    "cacheId": "01J9Z8QNCACHE0000000001",
    "similarity": 0.973,
    "tokensSaved": 5230,
    "costSavedUsd": 0.0187
  }
}
```

```json
{
  "type": "aios.context.window.cache_miss",
  "dataschema": "aios://schemas/context.window.cache_miss/1",
  "data": {
    "modelId": "gpt-x-mini",
    "reason": "similarity_below_threshold"
  }
}
```

### 5.4 `context.window.rejected`

```json
{
  "type": "aios.context.window.rejected",
  "dataschema": "aios://schemas/context.window.rejected/1",
  "data": {
    "bundleId": "01J9Z8QPBUNDLE0000000003",
    "code": "AIOS-CTX-0001",
    "reason": "budget_infeasible: min context exceeds token budget"
  }
}
```

### 5.5 `context.cache.evicted` e `context.cache.invalidated`

```json
{
  "type": "aios.context.cache.evicted",
  "dataschema": "aios://schemas/context.cache.evicted/1",
  "data": {
    "cacheId": "01J9Z8QNCACHE0000000001",
    "cause": "ttl_expired"
  }
}
```

```json
{
  "type": "aios.context.cache.invalidated",
  "dataschema": "aios://schemas/context.cache.invalidated/1",
  "data": {
    "cacheId": "01J9Z8QNCACHE0000000001",
    "sourceUrn": "urn:aios:acme:memory:01J9Z8QCITEMULID0000000001"
  }
}
```

### 5.6 `context.budget.updated`

```json
{
  "type": "aios.context.budget.updated",
  "dataschema": "aios://schemas/context.budget.updated/1",
  "data": {
    "profileId": "01J9Z8QQPROFILE0000000001",
    "scope": "tenant"
  }
}
```

## 6. Correlação com a máquina de estados

Cada evento emitido corresponde a uma transição canônica do `ContextBundle`
(brief §4.1) ou do `SemanticCacheEntry` (brief §4.2). Rastreabilidade:

| Transição de estado | Evento emitido |
|------------------------|-------------------|
| `CACHE_LOOKUP → SERVED` (hit) | `window.cache_hit` |
| `CACHE_LOOKUP → BUDGET_ALLOCATED` (miss/bypass) | `window.cache_miss` |
| `COMPRESSING` (método aplicado) | `window.compressed` |
| `ASSEMBLED → SERVED` (persistência + outbox confirmadas) | `window.assembled` |
| `* → REJECTED` (orçamento infeasível ou authZ deny) | `window.rejected` |
| `SemanticCacheEntry`: `STALE/INVALIDATED → EVICTED` | `cache.evicted` |
| `SemanticCacheEntry`: `INDEXED/SERVING → INVALIDATED` | `cache.invalidated` |
| `BudgetProfile` upsert confirmado | `budget.updated` |

Ver [StateMachine.md](./StateMachine.md) para a reprodução completa das
máquinas de estado que originam estas transições.

## 7. Riscos e alternativas

| Risco / decisão | Mitigação / alternativa |
|-----------------|--------------------------|
| Duplicata de evento (at-least-once). | Dedup obrigatória por `event.id`; consumidores idempotentes. Alternativa (exactly-once nativo) descartada por custo/acoplamento (mesma decisão de `010-Memory`). |
| Perda de evento em crash pós-commit. | Outbox transacional + relay; RPO de evento ≤ 60 s (brief §9-FM07). |
| Vazamento de conteúdo de fragmento/resposta cacheada em `data`. | Minimização; só identificadores/scores/contadores no envelope; conteúdo permanece em PostgreSQL/MinIO sob RLS e `content_ref`/`response_ref` controlado. |
| Volume alto de `window.cache_miss` sob baixa taxa de acerto. | Amostragem configurável (`context.telemetry.*`, ver [Configuration.md](./Configuration.md)); não usado para lógica de negócio, apenas telemetria/SLI de NFR-006. |
| Falso-hit de cache propagado silenciosamente a consumidores. | `window.cache_hit` carrega `similarity` explícito, permitindo auditoria offline (NFR-011) e correlação com `cache.invalidated` subsequente em caso de detecção de poisoning (FM-08). |
| Poison message trava consumidor. | DLQ JetStream + alerta + inspeção manual, espelhando `010-Memory`. |

Decisões governadas por `ADR-0111` (cache semântico em dois níveis),
`ADR-0112` (limiar de similaridade), `ADR-0116` (invalidação/eviction) e
`ADR-0119` (isolamento de cache por tenant) — ver [ADR.md](./ADR.md) e
[RFC.md](./RFC.md).
