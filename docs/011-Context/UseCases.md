---
Documento: UseCases
Módulo: 011-Context
Status: Draft
Versão: 0.1
Última atualização: 2026-07-20
Responsável (RACI-A): Arquiteto do Módulo 011-Context
ADRs relacionados: ADR-0110, ADR-0111, ADR-0112, ADR-0113, ADR-0114, ADR-0115, ADR-0116, ADR-0117, ADR-0118, ADR-0119
RFCs relacionados: RFC-0001, RFC-0011
Depende de: 010-Memory, 017-Model-Router, 022-Policy, 024-Observability, 025-Audit, 021-Security, 006-Kernel, 008-Agent-Lifecycle, 040-Glossary
---

# AIOS — Módulo 011 · Context — Casos de Uso

> Este documento deriva integralmente do [`_DESIGN_BRIEF.md`](./_DESIGN_BRIEF.md)
> (§2, §4, §5, §6, §9) e **NÃO PODE contradizê-lo**. Contratos centrais (URN,
> envelope de evento, envelope de erro, idempotência, correlação, subjects) são
> **reutilizados** de [RFC-0001](../003-RFC/RFC-0001-Architecture-Baseline.md),
> nunca redefinidos. Palavras normativas conforme RFC 2119/8174 (**DEVE**,
> **NÃO DEVE**, **DEVERIA**, **PODE**).

## 1. Convenções e atores

Cada caso de uso segue o formato **ator / pré-condições / fluxo principal /
fluxos alternativos / exceções / pós-condições**, e é rastreável até os
requisitos funcionais ([`FunctionalRequirements.md`](./FunctionalRequirements.md))
da §7.1 do brief. Toda mutação DEVE portar `Idempotency-Key`, `X-AIOS-Tenant` e
`traceparent` (RFC-0001 §5.5, §5.6). Toda operação privilegiada DEVE ser
autorizada pelo `ContextPolicyGuard` junto ao PDP (`022`) em regime *default
deny*.

### 1.1 Atores

| Ator | Tipo | Descrição |
|------|------|-----------|
| **AgentRuntime (007)** | Primário (via API) | Runtime Python que solicita a montagem de contexto antes de cada chamada de LLM; NÃO acessa `010`/`017` diretamente para fins de contexto — sempre via `011`. |
| **Agent-Runtime Orchestrator / Model-Router caller** | Primário | Componente do plano de controle que invoca `assemble`/`compress` em nome do agente (via `007`/`012`). |
| **MemoryService (010)** | Suporte | Fonte do recall seletivo; produtor de eventos de invalidação. |
| **ModelRouter (017)** | Suporte | Fonte de limites de modelo, tokenizer, embedding e sumarização. |
| **PolicyEngine (022)** | Suporte | PDP consultado pelo `ContextPolicyGuard`. |
| **AgentLifecycle (006/008)** | Primário (via evento) | Emite eventos de suspensão/término que disparam expurgo de contexto/cache por agente. |
| **SecurityService (021/025)** | Primário | Dispara expurgo LGPD (RTBF) e consome trilha de auditoria. |
| **Operador de Plataforma** | Secundário | Consulta/edita `BudgetProfile` e observa métricas de compressão/cache. |
| **EvictionManager (interno)** | Interno | Executa TTL/LRU semântico do cache, reagindo a pressão e eventos. |

### 1.2 Índice de casos de uso

| UC | Nome | Ator primário | FR de origem |
|----|------|----------------|--------------|
| UC-001 | Montar contexto ótimo (`assemble`, cache *miss*) | AgentRuntime | FR-001, FR-002, FR-003, FR-004, FR-005, FR-007, FR-009, FR-011 |
| UC-002 | Servir contexto via cache semântico (`assemble`, cache *hit*) | AgentRuntime | FR-006, FR-009, FR-011 |
| UC-003 | Comprimir/sumarizar fragmentos avulsos (`compress`) | AgentRuntime / PlanningEngine (012) | FR-003, FR-004, FR-014 |
| UC-004 | Armazenar resposta no cache semântico (`cache store`) | AgentRuntime / Model-Router caller | FR-006, FR-011 |
| UC-005 | Invalidar cache por evento de mudança de origem | MemoryService (010) / Knowledge (018/019) | FR-008 |
| UC-006 | Invalidar cache manualmente por escopo (`cache/invalidate`) | Operador de Plataforma / SecurityService | FR-008, FR-012 |
| UC-007 | Estimar tokens de um payload (`tokens/estimate`) | AgentRuntime | FR-007 |
| UC-008 | Ler/atualizar `BudgetProfile` (`budgets/{scope}`) | Operador de Plataforma | FR-002 |
| UC-009 | Degradar graciosamente quando `010`/`017` indisponíveis | ContextAssembler (interno) | FR-010 |
| UC-010 | Reagir ao ciclo de vida do agente (suspensão/término) | AgentLifecycle (006/008) | FR-008, FR-012 |
| UC-011 | Expurgar contexto/cache por RTBF | SecurityService (021/025) | FR-012 |

---

## 2. UC-001 — Montar contexto ótimo (`assemble`, cache *miss*)

- **Ator primário:** AgentRuntime.
- **Atores de suporte:** `ContextApiGateway`, `ContextPolicyGuard`, `PolicyEngine
  (022)`, `ContextAssembler`, `TokenBudgeter`, `TokenCounter`,
  `SelectiveRetriever`, `MemoryClient` (→ `010`), `RelevanceRanker`,
  `RedundancyEliminator`, `HierarchicalCompressor`, `BundleStore`,
  `EventPublisher`.
- **Objetivo:** montar um `ContextBundle` que caiba no `token_budget` do modelo
  alvo, maximizando relevância preservada.
- **Rastreabilidade:** FR-001, FR-002, FR-003, FR-004, FR-005, FR-007, FR-009,
  FR-011; NFR-002, NFR-003, NFR-004, NFR-005, NFR-010.

### 2.1 Pré-condições

1. O chamador DEVE estar autenticado (OIDC/mTLS, RFC-0001 §6) e portar
   `X-AIOS-Tenant`, `Idempotency-Key` e `traceparent`.
2. O `tenant_id` do recurso DEVE coincidir com o contexto autenticado.
3. `model_id` informado é resolvível pelo `017-Model-Router` (limites e
   tokenizer conhecidos ou cacheados).

### 2.2 Fluxo principal (caminho feliz)

1. AgentRuntime envia `POST /v1/context/assemble` (ou gRPC `Assemble`) com
   `{agent_urn, task_urn?, model_id, sections, budget_profile_scope}`.
2. `ContextApiGateway` valida o schema, checa `Idempotency-Key` no registro de
   idempotência e impõe o tenant; o bundle nasce em `RECEIVED`
   ([`StateMachine.md`](./StateMachine.md) §4.1).
3. `ContextPolicyGuard` consulta o PDP (`022`) — decisão *allow*; bundle
   transita `RECEIVED → POLICY_CHECK`.
4. `ContextAssembler` consulta o `SemanticCacheManager` (`CACHE_LOOKUP`); não há
   *hit* válido (`MISS`) — emite `aios.<tenant>.context.window.cache_miss`.
5. `TokenBudgeter` obtém `model_max_tokens`/`reserved_output_ratio` do
   `ModelRouterClient` (`017`) e o `BudgetProfile` efetivo, calculando
   `token_budget` por seção; bundle transita para `BUDGET_ALLOCATED`.
6. `SelectiveRetriever` solicita recall seletivo ao `MemoryClient` (`010`)
   dentro de `context.retrieval.memory_deadline_ms`, limitado a
   `context.retrieval.top_k`; bundle em `RETRIEVING`.
7. `RelevanceRanker` ordena os candidatos por similaridade + recência +
   prioridade; bundle em `RANKING`.
8. `RedundancyEliminator` colapsa near-duplicates (`cos ≥
   context.dedup.cosine_threshold`); `HierarchicalCompressor` aplica
   sumarização hierárquica map-reduce até caber no orçamento restante; bundle
   em `COMPRESSING`.
9. Quando `tokens_in ≤ token_budget`, o bundle transita para `ASSEMBLED`.
10. `BundleStore` persiste `ContextBundle`/`ContextFragment`s (offload a MinIO
    quando aplicável, FR-013); `EventPublisher` grava o outbox e publica
    `aios.<tenant>.context.window.assembled` e
    `aios.<tenant>.context.window.compressed`; bundle transita para `SERVED`.
11. A API retorna `200 OK` com `bundle_id`, `tokens_in`, `compression_ratio`,
    `cache_outcome=MISS`.

### 2.3 Fluxos alternativos

- **A1 — Requisição repetida (idempotência):** se `Idempotency-Key` já existe
  com o mesmo payload, o passo 2 retorna o resultado memoizado (≥ 24 h) sem
  reexecutar 3–10 (FR-011).
- **A2 — Fragmento grande:** no passo 10, fragmento com
  `bytes > context.limits.max_inline_bytes` é externalizado ao MinIO
  (`content_ref`), mantendo `content_inline = NULL` (FR-013).
- **A3 — Reuso de `SummaryNode`:** no passo 8, antes de sumarizar, o
  `HierarchicalCompressor` consulta `SummaryNode`s existentes por similaridade;
  se `similarity ≥ context.cache.reuse_threshold` e não expirado, reutiliza o
  sumário em vez de chamar o modelo de sumarização (FR-014).
- **A4 — Bypass de cache explícito:** se `bypass_cache=true`, o passo 4 é
  ignorado e o fluxo segue direto para `BUDGET_ALLOCATED`.

### 2.4 Exceções

| Condição | Código | HTTP | Efeito |
|----------|--------|------|--------|
| Schema/DTO inválido | `AIOS-CTX-0014` | 400 | Rejeita; nada persistido. |
| Tenant divergente | `AIOS-CTX-0011` | 403 | Rejeita; auditoria (`025`). |
| PDP nega | `AIOS-CTX-0011` | 403 | Rejeita. |
| Payload excede limite | `AIOS-CTX-0013` | 413 | Rejeita. |
| Orçamento infeasível (system+instrução > budget) | `AIOS-CTX-0001` | 422 | `REJECTED`; sugere modelo maior ao `017`. |
| `017` indisponível (limites/tokenizer) sem cache válido | `AIOS-CTX-0002` | 503 | Retriable; ver UC-009. |
| Tokenizer indisponível | `AIOS-CTX-0003` | 503 | Retriable. |
| `010-Memory` timeout de recall | `AIOS-CTX-0005` | 504 | Transita para `DEGRADED`; ver UC-009. |
| Falha de sumarização | `AIOS-CTX-0004` | 500 | Fallback truncate; segue para `ASSEMBLED`. |
| Idempotency-Key com payload divergente | `AIOS-CTX-0012` | 409 | Rejeita. |
| Rate limit por tenant excedido | `AIOS-CTX-0015` | 429 | Retriable com backoff. |

### 2.5 Pós-condições

1. Em sucesso, o bundle é consultável por `GET /v1/context/bundles/{bundle_id}`
   em `state=SERVED` até `expires_at`.
2. Exatamente um evento `aios.<tenant>.context.window.assembled` DEVE ser
   publicado (at-least-once via Outbox).
3. `tokens_in ≤ token_budget` (invariante de FR-001).

---

## 3. UC-002 — Servir contexto via cache semântico (`assemble`, cache *hit*)

- **Ator primário:** AgentRuntime.
- **Atores de suporte:** `ContextApiGateway`, `ContextPolicyGuard`,
  `ContextAssembler`, `SemanticCacheManager`, `EmbeddingClient`,
  `EventPublisher`, `TelemetryEmitter`.
- **Objetivo:** evitar reinferência e nova montagem completa quando uma
  entrada de cache semântico equivalente já existe.
- **Rastreabilidade:** FR-006, FR-009, FR-011; NFR-001, NFR-006, NFR-007,
  NFR-011.

### 3.1 Pré-condições

1. Chamador autenticado, autorizado, `bypass_cache` ausente/`false` e
   `context.cache.enabled=true` para o escopo.
2. Existe ao menos uma `SemanticCacheEntry` indexada (`INDEXED`/`SERVING`) para
   o par (`tenant`, `model_id`, escopo).

### 3.2 Fluxo principal (caminho feliz)

1. AgentRuntime envia `POST /v1/context/assemble` (mesmo contrato do UC-001).
2. Após `POLICY_CHECK`, `ContextAssembler` invoca `SemanticCacheManager.lookup`:
   primeiro por `prompt_hash` (fast path, correspondência exata); na ausência,
   por busca ANN via `prompt_embedding` (`EmbeddingClient` + HNSW em
   `pgvector`, slow path).
3. Encontrada entrada com `similarity ≥ similarity_threshold` e
   `invalidated_at IS NULL`, o bundle transita `CACHE_LOOKUP → SERVED`
   diretamente (bypassa `BUDGET_ALLOCATED..ASSEMBLED`).
4. `SemanticCacheManager` incrementa `hit_count`, atualiza `last_hit_at` e
   contabiliza `tokens_saved`/`cost_saved_usd`; a entrada transita
   `INDEXED → SERVING` ([`StateMachine.md`](./StateMachine.md) §4.2).
5. `EventPublisher` publica `aios.<tenant>.context.window.cache_hit` (com
   `similarity`, `tokensSaved`, `costSavedUsd`).
6. A API retorna `200 OK` com `cache_outcome=HIT` e o payload cacheado
   (`response_ref` resolvido inline ou via MinIO).

### 3.3 Fluxos alternativos

- **A1 — Fast path exato:** quando `prompt_hash` coincide, a busca ANN é
  evitada por completo, atingindo a meta de latência mais agressiva
  (NFR-001).
- **A2 — Múltiplos candidatos ANN:** entre candidatos acima do limiar, o de
  maior `similarity` é escolhido; empate é resolvido por `last_hit_at` mais
  recente (LRU semântico).

### 3.4 Exceções

| Condição | Código | HTTP | Efeito |
|----------|--------|------|--------|
| Redis/pgvector indisponível | `AIOS-CTX-0006` | 503 | Bypass de cache; segue como UC-001 (`cache_outcome=BYPASS`). |
| Embedding indisponível para busca ANN | `AIOS-CTX-0009` | 503 | Fallback a `prompt_hash` apenas (FM-05, brief §9). |
| Entrada expirada entre lookup e leitura | `AIOS-CTX-0007` (interno) | — | Tratada como `MISS`; segue UC-001. |

### 3.5 Pós-condições

1. Nenhuma nova chamada de sumarização/embedding de fragmentos é realizada
   (economia de tokens/custo, NFR-006/NFR-007).
2. Evento `context.window.cache_hit` publicado; métricas de *hit ratio*
   atualizadas.
3. `hit_count` e `last_hit_at` da `SemanticCacheEntry` refletem o acesso.

---

## 4. UC-003 — Comprimir/sumarizar fragmentos avulsos (`compress`)

- **Ator primário:** AgentRuntime ou PlanningEngine (`012`).
- **Atores de suporte:** `ContextApiGateway`, `HierarchicalCompressor`,
  `RedundancyEliminator`, `TokenCounter`, `SummarizationClient` (via `017`).
- **Objetivo:** comprimir um conjunto de fragmentos fornecido diretamente pelo
  chamador (fora do fluxo completo de `assemble`), por exemplo para resumir um
  histórico longo antes de persistir em memória.
- **Rastreabilidade:** FR-003, FR-004, FR-014; NFR-004, NFR-005.

### 4.1 Pré-condições

1. Chamador autenticado com `X-AIOS-Tenant`, `traceparent`, `Idempotency-Key`.
2. Ao menos um fragmento de entrada é fornecido (`fragments[]`, cada um com
   `content`/`content_ref` e `role`).

### 4.2 Fluxo principal (caminho feliz)

1. Chamador envia `POST /v1/context/compress` com `{fragments[], target_tokens,
   method?}`.
2. `RedundancyEliminator` deduplicará fragmentos semanticamente equivalentes
   (`cos ≥ context.dedup.cosine_threshold`).
3. `HierarchicalCompressor` aplica sumarização map (por lote) → reduce
   (hierárquico) até `tokens_final ≤ target_tokens`, respeitando
   `context.compression.max_levels`.
4. `TokenCounter` valida a contagem final.
5. A API retorna `200 OK` com os fragmentos comprimidos, `tokens_before`,
   `tokens_after` e `compression_ratio`.

### 4.3 Fluxos alternativos

- **A1 — `method=extractive`:** aplica apenas seleção extrativa de sentenças,
  sem chamar modelo de sumarização (mais barato, menor ratio).
- **A2 — `method=truncate`:** trunca deterministicamente por prioridade/ordem,
  sem custo de inferência (usado também como fallback de FM-04).
- **A3 — Reuso de `SummaryNode`:** igual a UC-001/A3.

### 4.4 Exceções

| Condição | Código | HTTP | Efeito |
|----------|--------|------|--------|
| `fragments[]` vazio/schema inválido | `AIOS-CTX-0014` | 400 | Rejeita. |
| Falha do modelo de sumarização | `AIOS-CTX-0004` | 500 | Fallback truncate; resposta ainda `200` com `method_used=truncate`. |
| `target_tokens` inalcançável mesmo truncando ao mínimo semântico | `AIOS-CTX-0001` | 422 | Rejeita. |

### 4.5 Pós-condições

1. `tokens_after ≤ target_tokens` (salvo rejeição explícita).
2. `SummaryNode`s produzidos são persistidos como cache de curta duração
   (`expires_at`), nunca como memória canônica (fronteira do brief §1.2).

---

## 5. UC-004 — Armazenar resposta no cache semântico (`cache store`)

- **Ator primário:** AgentRuntime / Model-Router caller (após obter a resposta
  final do LLM via `017`).
- **Atores de suporte:** `ContextApiGateway`, `SemanticCacheManager`,
  `EmbeddingClient`, `BundleStore` (MinIO), `EventPublisher`.
- **Objetivo:** indexar uma resposta de LLM já obtida para reuso futuro por
  similaridade.
- **Rastreabilidade:** FR-006, FR-011; NFR-006, NFR-007.

### 5.1 Pré-condições

1. Chamador autenticado; `context.cache.enabled=true` para o escopo.
2. Resposta a cachear e o prompt normalizado que a originou estão disponíveis.

### 5.2 Fluxo principal (caminho feliz)

1. Chamador envia `POST /v1/context/cache/entries` com
   `{model_id, prompt_normalized, response, cache_scope, scope_ref?,
   ttl_seconds?, similarity_threshold?}`.
2. `SemanticCacheManager` calcula `prompt_hash` (normalização determinística) e
   solicita `prompt_embedding` ao `EmbeddingClient` (`017`).
3. A entrada nasce em `PENDING`; após persistir embedding + `response_ref`
   (inline ou MinIO), transita para `INDEXED`
   ([`StateMachine.md`](./StateMachine.md) §4.2).
4. A API retorna `201 Created` com `cache_id`.

### 5.3 Fluxos alternativos

- **A1 — Requisição repetida (idempotência):** mesma `Idempotency-Key` e
  payload retorna o mesmo `cache_id` sem reindexar (FR-011).
- **A2 — Embedding indisponível:** entrada é indexada apenas por
  `prompt_hash` (fast path exato); busca ANN não a encontrará até reindexação
  em backlog (FM-05).

### 5.4 Exceções

| Condição | Código | HTTP | Efeito |
|----------|--------|------|--------|
| Schema inválido | `AIOS-CTX-0014` | 400 | Rejeita. |
| Cache backend indisponível | `AIOS-CTX-0006` | 503 | Retriable. |
| Embedding indisponível | `AIOS-CTX-0009` | 503 | Ver A2 (fallback hash-only). |
| Idempotency-Key com payload divergente | `AIOS-CTX-0012` | 409 | Rejeita. |

### 5.5 Pós-condições

1. Entrada consultável por `CacheLookup`/`assemble` subsequentes com prompt
   equivalente.
2. `ttl_seconds` e `similarity_threshold` efetivos refletem os defaults de
   configuração quando omitidos.

---

## 6. UC-005 — Invalidar cache por evento de mudança de origem

- **Ator primário:** MemoryService (`010`) ou Knowledge (`018/019`), via
  evento.
- **Atores de suporte:** `SemanticCacheManager`, `EvictionManager`,
  `EventPublisher`.
- **Objetivo:** manter a coerência do cache quando o item de origem (memória
  ou conhecimento) que o fundamenta muda.
- **Rastreabilidade:** FR-008; NFR-011.

### 6.1 Pré-condições

1. Existe ao menos uma `SemanticCacheEntry`/`SummaryNode` cuja proveniência
   referencia o `source_urn` afetado.

### 6.2 Fluxo principal (caminho feliz)

1. Chega `aios.<tenant>.memory.item.consolidated` (ou `.deleted`, ou
   `aios.<tenant>.knowledge.node.updated`).
2. `SemanticCacheManager` localiza entradas com proveniência sobreposta ao
   `source_urn` do evento.
3. As entradas afetadas transitam `INDEXED/SERVING → INVALIDATED`
   ([`StateMachine.md`](./StateMachine.md) §4.2).
4. `EventPublisher` publica `aios.<tenant>.context.cache.invalidated` por
   entrada afetada.
5. `EvictionManager` agenda a remoção física (`INVALIDATED → EVICTED`).

### 6.3 Fluxos alternativos

- **A1 — Item deletado (RTBF em cascata):** `memory.item.deleted` invalida
  **e** aciona expurgo imediato (sem esperar TTL), unindo-se ao fluxo de
  UC-011.

### 6.4 Exceções

| Condição | Efeito |
|----------|--------|
| Evento duplicado | Consumidor deduplica por `event.id` (idempotente); sem efeito colateral adicional. |
| Nenhuma entrada correspondente | No-op; nenhum evento de invalidação emitido. |

### 6.5 Pós-condições

1. Invalidação concluída em ≤ 5 s após o recebimento do evento (FR-008).
2. Consultas subsequentes a essas entradas resultam em `MISS`, nunca em *hit*
   obsoleto.

---

## 7. UC-006 — Invalidar cache manualmente por escopo (`cache/invalidate`)

- **Ator primário:** Operador de Plataforma ou SecurityService (`021/025`).
- **Atores de suporte:** `ContextPolicyGuard`, `SemanticCacheManager`,
  `EventPublisher`.
- **Objetivo:** permitir invalidação administrativa (fora do fluxo
  event-driven) por escopo — tenant, agente ou `source_urn`.
- **Rastreabilidade:** FR-008, FR-012.

### 7.1 Pré-condições

1. Chamador autorizado pelo PDP (`022`) para operação privilegiada de
   invalidação em massa.

### 7.2 Fluxo principal

1. `POST /v1/context/cache/invalidate` com `{scope, scope_ref, reason}`.
2. `ContextPolicyGuard` confirma autorização (*default deny*).
3. `SemanticCacheManager` invalida todas as entradas do escopo; emite
   `context.cache.evicted` por lote com `cause=manual`/`cause=rtbf`.
4. A API retorna `202 Accepted` com a contagem de entradas afetadas.

### 7.3 Fluxos alternativos

- **A1 — Invalidação específica:** `DELETE /v1/context/cache/entries/{cache_id}`
  invalida uma única entrada (sem necessidade de `scope`).

### 7.4 Exceções

| Condição | Código | HTTP |
|----------|--------|------|
| PDP nega | `AIOS-CTX-0011` | 403 |
| `cache_id` inexistente | `AIOS-CTX-0007` | 404 |
| Idempotency-Key com payload divergente | `AIOS-CTX-0012` | 409 |

### 7.5 Pós-condições

1. Entradas do escopo não são mais servidas como *hit*.
2. Auditoria (`025`) registra o ator, o motivo e o escopo da invalidação.

---

## 8. UC-007 — Estimar tokens de um payload (`tokens/estimate`)

- **Ator primário:** AgentRuntime (ex.: antes de decidir dividir uma
  requisição).
- **Atores de suporte:** `TokenCounter`, `ModelRouterClient` (`017`).
- **Objetivo:** contar/estimar tokens de um payload para um `model_id`, sem
  montar um bundle.
- **Rastreabilidade:** FR-007; NFR-010.

### 8.1 Pré-condições

1. `model_id` informado é resolvível (tokenizer conhecido ou estimável).

### 8.2 Fluxo principal

1. `POST /v1/context/tokens/estimate` com `{model_id, payload}`.
2. `TokenCounter` usa o tokenizer exato via `017` quando disponível; caso
   contrário aplica estimador calibrado (erro ≤ 2%).
3. A API retorna `{tokens, exact: true|false}`.

### 8.3 Fluxos alternativos

- **A1 — Tokenizer indisponível:** retorna estimativa com `exact=false` e
  aviso nos metadados, em vez de falhar (a menos que
  `require_exact=true` seja explicitado).

### 8.4 Exceções

| Condição | Código | HTTP |
|----------|--------|------|
| `model_id` desconhecido pelo `017` | `AIOS-CTX-0002` | 503 |
| Tokenizer indisponível com `require_exact=true` | `AIOS-CTX-0003` | 503 |

### 8.5 Pós-condições

1. Nenhuma mutação de estado; operação somente leitura.

---

## 9. UC-008 — Ler/atualizar `BudgetProfile` (`budgets/{scope}`)

- **Ator primário:** Operador de Plataforma.
- **Atores de suporte:** `ContextPolicyGuard`, `BundleStore`,
  `EventPublisher`.
- **Objetivo:** consultar ou ajustar a política de alocação de tokens por
  seção para um escopo (`tenant`/`agent`/`task_type`).
- **Rastreabilidade:** FR-002.

### 9.1 Pré-condições

1. Chamador autorizado pelo PDP a administrar `BudgetProfile` do escopo.

### 9.2 Fluxo principal

1. `GET /v1/context/budgets/{scope}` retorna o perfil efetivo (com *fallback*
   para o perfil de tenant se o escopo mais específico não existir).
2. `PUT /v1/context/budgets/{scope}` com os novos pesos atualiza o perfil;
   valida `Σ alloc_* = 1.0`.
3. `EventPublisher` publica `aios.<tenant>.context.budget.updated`.

### 9.3 Fluxos alternativos

- **A1 — Escopo inexistente na leitura:** retorna o perfil de tenant
  (default) com um indicador `inherited=true`.

### 9.4 Exceções

| Condição | Código | HTTP |
|----------|--------|------|
| Pesos não somam 1,0 | `AIOS-CTX-0008` | 422 |
| PDP nega | `AIOS-CTX-0011` | 403 |
| Idempotency-Key com payload divergente | `AIOS-CTX-0012` | 409 |

### 9.5 Pós-condições

1. Novas montagens (`assemble`) para o escopo passam a usar o perfil
   atualizado imediatamente (sem necessidade de reinício).

---

## 10. UC-009 — Degradar graciosamente quando `010`/`017` indisponíveis

- **Ator primário:** `ContextAssembler` (interno, disparado durante UC-001).
- **Atores de suporte:** `ModelRouterClient`, `MemoryClient`,
  `TelemetryEmitter`, `EventPublisher`.
- **Objetivo:** entregar um contexto viável mesmo quando dependências
  críticas degradam, em vez de falhar a chamada do agente.
- **Rastreabilidade:** FR-010; NFR-008, NFR-012.

### 10.1 Pré-condições

1. Uma montagem (`assemble`) está em curso (estados `BUDGET_ALLOCATED` ou
   `RETRIEVING`/`COMPRESSING`).
2. `context.degraded.allow_truncate_fallback=true` para o escopo.

### 10.2 Fluxo principal

1. `017-Model-Router` não responde dentro do deadline do circuit breaker:
   `TokenBudgeter` usa o último `model_max_tokens`/tokenizer cacheado válido;
   se não houver cache, retorna `AIOS-CTX-0002`.
2. `010-Memory` excede `context.retrieval.memory_deadline_ms`: o bundle
   transita para `DEGRADED`; a montagem prossegue apenas com
   `system`+`instruction`+`history` já disponíveis localmente.
3. O bundle segue para `ASSEMBLED → SERVED` com `degraded=true` nos
   metadados, respeitando ainda o orçamento.
4. `EventPublisher` publica o evento terminal correspondente
   (`window.assembled` com `degraded=true`); `TelemetryEmitter` incrementa o
   contador de degradações.

### 10.3 Fluxos alternativos

- **A1 — Falha de sumarização durante compressão:** aplica fallback
  `truncate` determinístico (FM-04) sem necessariamente marcar `DEGRADED`
  (a montagem ainda respeita o orçamento).
- **A2 — Ambas as dependências indisponíveis:** o bundle é servido apenas com
  o conteúdo fornecido diretamente na requisição (`sections.system`/
  `instruction`), se isso já satisfizer o mínimo semântico; caso contrário,
  `AIOS-CTX-0001`.

### 10.4 Exceções

| Condição | Código | HTTP | Efeito |
|----------|--------|------|--------|
| `017` indisponível sem cache de limites | `AIOS-CTX-0002` | 503 | Rejeita (retriable). |
| Orçamento infeasível mesmo em modo degradado | `AIOS-CTX-0001` | 422 | `REJECTED`. |

### 10.5 Pós-condições

1. O chamador recebe um bundle utilizável (`SERVED`, `degraded=true`) sempre
   que o mínimo semântico couber no orçamento.
2. Métricas de degradação (`aios_context_degraded_total`) refletem o
   incidente para acionamento de runbooks (`./Monitoring.md`).

---

## 11. UC-010 — Reagir ao ciclo de vida do agente (suspensão/término)

- **Ator primário:** AgentLifecycle (`006/008`), via evento.
- **Atores de suporte:** `BundleStore`, `SemanticCacheManager`,
  `EvictionManager`.
- **Objetivo:** expurgar contexto/cache efêmero associado a um agente
  suspenso ou terminado, evitando acúmulo desnecessário.
- **Rastreabilidade:** FR-008, FR-012; NFR-014.

### 11.1 Pré-condições

1. Chega `aios.<tenant>.agent.lifecycle.suspended` ou `.terminated`.

### 11.2 Fluxo principal

1. **Suspenso:** `EvictionManager` marca como candidatos a *eviction*
   prioritário (LRU semântico) os `SemanticCacheEntry` com `cache_scope=agent`
   referentes ao agente; `ContextBundle`s ativos permanecem até `expires_at`.
2. **Terminado:** `BundleStore` expira imediatamente todos os
   `ContextBundle`s remanescentes do agente; `SemanticCacheManager` invalida
   entradas de escopo `agent` associadas.
3. `EventPublisher` publica `aios.<tenant>.context.cache.evicted` por entrada
   afetada (`cause=agent_terminated`/`agent_suspended`).

### 11.3 Fluxos alternativos

- **A1 — Agente retomado (resume):** entradas de escopo `agent` ainda não
  evictadas continuam elegíveis a *hit* normalmente.

### 11.4 Exceções

| Condição | Efeito |
|----------|--------|
| Evento duplicado | Consumidor deduplica por `event.id`. |
| Nenhum recurso associado ao agente | No-op. |

### 11.5 Pós-condições

1. Recursos efêmeros de um agente terminado não persistem além do
   necessário, alinhado à natureza efêmera de `ContextBundle`/cache por
   escopo `agent` (brief §1.2).

---

## 12. UC-011 — Expurgar contexto/cache por RTBF (direito ao esquecimento)

- **Ator primário:** SecurityService (`021/025`).
- **Atores de suporte:** `ContextPolicyGuard`, `BundleStore`,
  `SemanticCacheManager`, `EventPublisher`.
- **Objetivo:** remover de forma rastreável bundles e entradas de cache que
  referenciam dados de um sujeito, atendendo LGPD/GDPR.
- **Rastreabilidade:** FR-012; NFR-014.

### 12.1 Pré-condições

1. Existe base legal para o expurgo (RTBF autorizado por `021`).
2. Alvo identificado por `source_urn`/agente/tenant.

### 12.2 Fluxo principal

1. Gatilho: `POST /v1/context/cache/invalidate` com `{scope: "source_urn",
   scope_ref}` (UC-006) ou evento `aios.<tenant>.memory.item.deleted`
   (UC-005/A1).
2. `ContextPolicyGuard` autoriza o expurgo RTBF.
3. `SemanticCacheManager` invalida e agenda remoção física das entradas
   afetadas; `BundleStore` remove bundles residuais que referenciam o
   `source_urn`.
4. `EventPublisher` publica `aios.<tenant>.context.cache.evicted` com
   `cause=rtbf`.
5. A auditoria (`025`) consome o evento e registra o expurgo.

### 12.3 Fluxos alternativos

- **A1 — RTBF por agente/tenant:** o expurgo cobre todos os recursos do
  escopo, em lote (mesmo fluxo de UC-006).

### 12.4 Exceções

| Condição | Código | HTTP |
|----------|--------|------|
| Autorização RTBF ausente | `AIOS-CTX-0011` | 403 |
| Backend indisponível | `AIOS-CTX-0006` | 503 |

### 12.5 Pós-condições

1. Conteúdo referenciado pelo `source_urn` não é mais servido por *hit* de
   cache nem recuperável via bundles residuais.
2. Evento `context.cache.evicted` (`cause=rtbf`) consumido pelo Audit (`025`).

---

## 13. Matriz de rastreabilidade UC → FR → evento

| UC | FR | Evento(s) emitido(s) | Estado(s) afetado(s) |
|----|----|-----------------------|------------------------|
| UC-001 | FR-001, FR-002, FR-003, FR-004, FR-005, FR-007, FR-009, FR-011 | `window.cache_miss`, `window.compressed`, `window.assembled` | `RECEIVED→…→ASSEMBLED→SERVED` |
| UC-002 | FR-006, FR-009, FR-011 | `window.cache_hit` | `CACHE_LOOKUP→SERVED`; `INDEXED→SERVING` |
| UC-003 | FR-003, FR-004, FR-014 | `window.compressed` | — (sem bundle persistente) |
| UC-004 | FR-006, FR-011 | — | `PENDING→INDEXED` |
| UC-005 | FR-008 | `cache.invalidated` | `INDEXED/SERVING→INVALIDATED` |
| UC-006 | FR-008, FR-012 | `cache.evicted` | `*→INVALIDATED/EVICTED` |
| UC-007 | FR-007 | — | — |
| UC-008 | FR-002 | `budget.updated` | — |
| UC-009 | FR-010 | `window.assembled` (`degraded=true`) | `RETRIEVING/COMPRESSING→DEGRADED→ASSEMBLED` |
| UC-010 | FR-008, FR-012 | `cache.evicted` | `*→EVICTED` |
| UC-011 | FR-012 | `cache.evicted` (`cause=rtbf`) | `*→INVALIDATED→EVICTED` |

> Referências: fluxos detalhados em [`SequenceDiagrams.md`](./SequenceDiagrams.md);
> estados e transições em [`StateMachine.md`](./StateMachine.md); contratos de
> API em [`API.md`](./API.md); eventos em [`Events.md`](./Events.md).
