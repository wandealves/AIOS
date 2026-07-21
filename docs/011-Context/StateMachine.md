---
Documento: StateMachine
Módulo: 011-Context
Status: Draft
Versão: 0.1
Última atualização: 2026-07-20
Responsável (RACI-A): Arquiteto do Módulo 011-Context
ADRs relacionados: ADR-0110, ADR-0111, ADR-0112, ADR-0116, ADR-0118; ADR-0001..ADR-0010 (herdadas)
RFCs relacionados: RFC-0001 (baseline); RFC-0011 (proposta)
Depende de: 010-Memory, 017-Model-Router, 022-Policy, 024-Observability
---

# 011-Context — StateMachine

> Estas são as máquinas de estado canônicas do módulo, reproduzidas sem
> alteração de `_DESIGN_BRIEF.md` §4. Existem **duas** máquinas
> independentes, com autoridades distintas: (1) o pipeline de montagem de um
> `ContextBundle` (§2–§8) e (2) o ciclo de vida de uma `SemanticCacheEntry`
> (§9–§12). Nenhum documento derivado PODE introduzir estado, transição ou
> guarda que contradiga o exposto aqui — qualquer extensão exige nova ADR e
> atualização do `_DESIGN_BRIEF.md` primeiro.

## Índice

1. Escopo e autoridade das máquinas de estado
2. `ContextBundle` — Estados
3. `ContextBundle` — Diagrama de estados (ASCII)
4. `ContextBundle` — Tabela de transições, gatilhos e guardas
5. `ContextBundle` — Ações de entrada e saída por estado
6. `ContextBundle` — Estados terminais
7. `ContextBundle` — Invariantes da máquina de estados
8. `ContextBundle` — Relação com eventos emitidos
9. `SemanticCacheEntry` — Estados
10. `SemanticCacheEntry` — Diagrama de estados (ASCII)
11. `SemanticCacheEntry` — Tabela de transições, gatilhos e guardas
12. `SemanticCacheEntry` — Estados terminais e invariantes
13. Casos especiais

---

## 1. Escopo e Autoridade das Máquinas de Estado

| Máquina | Estados | Autoridade | Onde a transição é decidida |
|---------|---------|------------|-------------------------------|
| `ContextBundle` (pipeline de montagem) | `RECEIVED`, `POLICY_CHECK`, `CACHE_LOOKUP`, `BUDGET_ALLOCATED`, `RETRIEVING`, `RANKING`, `COMPRESSING`, `ASSEMBLED`, `SERVED`, `REJECTED`, `DEGRADED`, `FAILED`, `EXPIRED` | **Context (011)** | `ContextApiGateway`, `ContextPolicyGuard`, `ContextAssembler` e seus subcomponentes internos. |
| `SemanticCacheEntry` (ciclo de vida do cache) | `PENDING`, `INDEXED`, `SERVING`, `STALE`, `INVALIDATED`, `EVICTED` | **Context (011)** | `SemanticCacheManager`, `EvictionManager`; invalidação também é **reativa** a eventos externos de `010-Memory`/`017-Model-Router`/`018-019-Knowledge` (autoridade do fato gerador é do produtor do evento; a transição em si é sempre executada pelo `011`). |

Ambas as máquinas são de autoridade **integral** do `011-Context` — diferente
de módulos como `009-Scheduler`, o `011` não delega estados a um serviço
externo; dependências externas (`010`, `017`, `022`) apenas **influenciam**
guardas (timeouts, deny, invalidação) sem possuir a transição.

---

## 2. `ContextBundle` — Estados

| Estado | Significado | Tipo |
|--------|--------------|------|
| `RECEIVED` | Requisição de `assemble` chegou ao `ContextApiGateway`, validada estruturalmente (URN, headers). | Transitório |
| `POLICY_CHECK` | `ContextPolicyGuard` consulta o PDP (`022`) para a operação solicitada. | Transitório |
| `CACHE_LOOKUP` | `SemanticCacheManager` consulta L1/L2 por *fast path* (hash) e *slow path* (similaridade). | Transitório |
| `BUDGET_ALLOCATED` | `TokenBudgeter` resolveu `BudgetProfile` e limites do modelo (`017`); orçamento por seção calculado. | Transitório |
| `RETRIEVING` | `SelectiveRetriever` solicita recall seletivo ao `010-Memory` dentro do *deadline*. | Transitório |
| `RANKING` | `RelevanceRanker` + `RedundancyEliminator` processam os candidatos recuperados. | Transitório |
| `COMPRESSING` | `HierarchicalCompressor` (ou estratégia configurada) reduz tokens para caber no orçamento. | Transitório |
| `ASSEMBLED` | Contexto final montado; `tokens_in ≤ token_budget`; ainda não persistido/entregue. | Transitório |
| `SERVED` | Bundle persistido (`BundleStore`) e evento publicado (*outbox*); resposta entregue ao chamador. | **Terminal** |
| `REJECTED` | Orçamento infeasível (`AIOS-CTX-0001`) ou negação de autorização (`AIOS-CTX-0011`). | **Terminal** |
| `DEGRADED` | Dependência (`010`/`017`) indisponível dentro do *deadline*; contexto parcial/truncado servido. | Transitório (conflui para `ASSEMBLED`/`SERVED`) |
| `FAILED` | Erro não recuperável no pipeline (fora dos ramos previstos de degradação). | **Terminal** |
| `EXPIRED` | `now > expires_at` após `SERVED` (TTL do bundle efêmero). | **Terminal** |

---

## 3. `ContextBundle` — Diagrama de Estados (ASCII)

```
            ┌──────────┐  authZ deny (AIOS-CTX-0011)     ┌──────────┐
 request ─▶ │ RECEIVED │──────────────────────────────▶ │ REJECTED │◀─┐
            └────┬─────┘                                 └──────────┘  │
                 │ validate ok                                        │ budget infeasible
            ┌────▼────────┐  PDP deny                                 │ (AIOS-CTX-0001)
            │ POLICY_CHECK│───────────────────────────────────────────┤
            └────┬────────┘                                           │
                 │ allow                                              │
            ┌────▼────────┐  HIT ─────────► emit cache_hit ──────┐    │
            │ CACHE_LOOKUP│                                      │    │
            └────┬────────┘  MISS/BYPASS                         │    │
                 │ emit cache_miss                               │    │
            ┌────▼──────────────┐  017 down                      │    │
            │ BUDGET_ALLOCATED  │──┐ (fallback default limits)   │    │
            └────┬──────────────┘  │                             │    │
                 │ ok              ▼                             │    │
            ┌────▼──────┐   ┌────────────┐  010 timeout          │    │
            │ RETRIEVING│──▶│  DEGRADED  │◀──(AIOS-CTX-0005)     │    │
            └────┬──────┘   └─────┬──────┘                       │    │
                 │ candidates     │ partial context             │    │
            ┌────▼────┐           │                             │    │
            │ RANKING │           │                             │    │
            └────┬────┘           │                             │    │
            ┌────▼──────────┐     │ compression fail            │    │
            │  COMPRESSING  │─────┤ (AIOS-CTX-0004 → truncate)  │    │
            └────┬──────────┘     │                             │    │
                 │ within budget  │                             │    │
            ┌────▼──────┐◀────────┘                             │    │
            │ ASSEMBLED │──────────────────────────────────────┘    │
            └────┬──────┘                                            │
                 │ deliver + persist + outbox                        │
            ┌────▼───┐   TTL                     unrecoverable error │
            │ SERVED │──────────► EXPIRED        ──────────► FAILED ◀─┘
            └────────┘
```

---

## 4. `ContextBundle` — Tabela de Transições, Gatilhos e Guardas

| De | Para | Gatilho | Guarda (DEVE ser verdadeira) | Ação/Evento |
|----|------|---------|--------------------------------|--------------|
| `RECEIVED` | `POLICY_CHECK` | `assemble` | Payload válido; `tenant` do token == `X-AIOS-Tenant`. | Registra `IdempotencyStore.TryBegin`. |
| `RECEIVED`/`POLICY_CHECK` | `REJECTED` | validação/authZ | Payload malformado (`AIOS-CTX-0014`) ou `tenant` divergente (`AIOS-CTX-0011`) ou PDP deny (`AIOS-CTX-0011`). | Emite `aios.<tenant>.context.window.rejected`. |
| `POLICY_CHECK` | `CACHE_LOOKUP` | `pdp_decision` | PDP `allow` (`022`). | — |
| `CACHE_LOOKUP` | `SERVED` | `cache_hit` | `similarity ≥ similarity_threshold` (slow path) OU `prompt_hash` exato (fast path) E entrada não invalidada. | Persiste bundle com `cache_outcome=HIT`; emite `aios.<tenant>.context.window.cache_hit`. |
| `CACHE_LOOKUP` | `BUDGET_ALLOCATED` | `cache_miss`/`bypass` | `MISS` OU `bypass_cache=true` OU cache indisponível (`AIOS-CTX-0006`, `cache_outcome=BYPASS`). | Emite `aios.<tenant>.context.window.cache_miss`. |
| `BUDGET_ALLOCATED` | `RETRIEVING` | `budget_ok` | `token_budget > 0` E alocação viável (`Σ pesos = 1.0`). | Aloca `TokenBudget` por seção. |
| `BUDGET_ALLOCATED` | `REJECTED` | `budget_infeasible` | `tokens(system+instruction) > token_budget` mínimo viável. | `AIOS-CTX-0001`; emite `context.window.rejected`. |
| `*` (`BUDGET_ALLOCATED`) | `BUDGET_ALLOCATED` (com fallback) | `model_router_down` | `017` indisponível para limites/tokenizer. | Usa limites cacheados; `AIOS-CTX-0002` se sem cache disponível → `REJECTED`. |
| `RETRIEVING` | `RANKING` | `candidates_received` | Recall retornou dentro do *deadline* (`context.retrieval.memory_deadline_ms`). | — |
| `RETRIEVING`/`COMPRESSING` | `DEGRADED` | `dependency_timeout` | `010` (recall) ou `017` (sumarização) excede o *deadline*. | `AIOS-CTX-0005` (recall) ou segue para *fallback* de compressão; monta contexto parcial (system+history). |
| `RANKING` | `COMPRESSING` | `ranked_deduped` | `RelevanceRanker` + `RedundancyEliminator` concluídos. | — |
| `COMPRESSING` | `ASSEMBLED` | `within_budget` | `tokens_in ≤ token_budget`. | Emite `aios.<tenant>.context.window.compressed`. |
| `COMPRESSING` | `DEGRADED` | `compression_fail` | Falha do modelo de sumarização (`017`). | `AIOS-CTX-0004`; aplica `TruncateCompressor` (*fallback* determinístico). |
| `DEGRADED` | `ASSEMBLED` | `fallback_applied` | Contexto parcial/truncado já respeita `token_budget`. | — |
| `ASSEMBLED` | `SERVED` | `persist_and_publish` | Persistência (`BundleStore`) + publicação *outbox* confirmadas. | Emite `aios.<tenant>.context.window.assembled`. |
| `ASSEMBLED`/`*` | `FAILED` | `unrecoverable_error` | Erro fora dos ramos previstos (ex.: falha de persistência após retries). | Sem evento de sucesso; erro 5xx propagado ao chamador. |
| `SERVED` | `EXPIRED` | `ttl_tick` | `now > expires_at`. | Remoção lógica; bundle não mais servível via `GetBundle`. |

---

## 5. `ContextBundle` — Ações de Entrada e Saída por Estado

| Estado | Ação de entrada (*on-enter*) | Ação de saída (*on-exit*) |
|--------|--------------------------------|-------------------------------|
| `RECEIVED` | Validar envelope (URN, `traceparent`, `X-AIOS-Tenant`, `Idempotency-Key`); iniciar `TryBegin` de idempotência. | Encerrar checagem de idempotência para o *path* corrente. |
| `POLICY_CHECK` | Consultar `ContextPolicyGuard` → PDP (`022`). | — |
| `CACHE_LOOKUP` | Calcular `prompt_hash` e (se necessário) `prompt_embedding`; consultar L1 então L2. | Registrar `CacheOutcome` para telemetria. |
| `BUDGET_ALLOCATED` | Resolver `BudgetProfile` efetivo; obter limites do modelo (`ModelRouterClient`). | Congelar `TokenBudget` para o restante do pipeline. |
| `RETRIEVING` | Disparar `SelectiveRetriever.RetrieveAsync` com *deadline*. | — |
| `RANKING` | Executar `RelevanceRanker` seguido de `RedundancyEliminator`. | — |
| `COMPRESSING` | Selecionar `ICompressionStrategy` conforme `context.compression.method`. | Registrar `compression_ratio` parcial por fragmento. |
| `ASSEMBLED` | Validar `tokens_in ≤ token_budget` (guarda final). | — |
| `SERVED` | Persistir bundle+fragmentos (`BundleStore.SaveAsync`); publicar evento via *outbox*. | *(terminal)* |
| `DEGRADED` | Registrar motivo de degradação (`reason_code`) para auditoria/telemetria. | Prosseguir para `ASSEMBLED` com contexto parcial/truncado. |
| `REJECTED` | Persistir motivo de rejeição; liberar quaisquer reservas de idempotência pendentes com o resultado de erro. | *(terminal)* |
| `FAILED` | Registrar erro não recuperável; emitir log crítico correlacionado (`trace_id`). | *(terminal)* |
| `EXPIRED` | Marcar bundle como não servível; elegível para expurgo físico por retenção. | *(terminal)* |

---

## 6. `ContextBundle` — Estados Terminais

Os estados terminais desta máquina são: **`SERVED`, `REJECTED`, `FAILED`,
`EXPIRED`**. Uma vez em estado terminal, nenhuma nova transição é permitida —
uma nova chamada para o mesmo `bundle_id` DEVE retornar o bundle existente
(`GetBundle`) ou, para `assemble` repetido com a mesma `Idempotency-Key`, o
mesmo resultado (RFC-0001 §5.5). `SERVED` transiciona para `EXPIRED` apenas
por decurso de TTL — não representa falha, mas fim do ciclo de vida efêmero
do bundle (brief §1.3: `ContextBundle` nunca é memória canônica).

---

## 7. `ContextBundle` — Invariantes da Máquina de Estados

| # | Invariante |
|---|-----------|
| SM-CTX-01 | Um `ContextBundle` NÃO DEVE estar em mais de um estado simultaneamente (máquina determinística por `bundle_id`). |
| SM-CTX-02 | Uma transição para `SERVED` DEVE coincidir com exatamente uma persistência + publicação de evento (transação *outbox* única) — nunca duas. |
| SM-CTX-03 | Toda transição para `SERVED` DEVE satisfazer `tokens_in ≤ token_budget` — a guarda `within_budget` NÃO É opcional (FR-001). |
| SM-CTX-04 | `DEGRADED` NUNCA é terminal — sempre converge para `ASSEMBLED` (contexto parcial) ou, em falha irrecuperável, para `FAILED`. |
| SM-CTX-05 | Transições que dependem do PDP (`POLICY_CHECK`) DEVEM negar por padrão (*default deny*) se o Policy Engine (`022`) estiver indisponível (RFC-0001 §5.8). |
| SM-CTX-06 | `CACHE_LOOKUP → SERVED` (via `HIT`) NÃO DEVE ocorrer se a entrada de cache estiver com `invalidated_at` preenchido. |
| SM-CTX-07 | Toda transição relevante DEVE ser acompanhada da emissão do evento correspondente no envelope RFC-0001 §5.2, publicado atomicamente com a persistência via *outbox*. |
| SM-CTX-08 | `EXPIRED` só é alcançável a partir de `SERVED` — um bundle nunca expira antes de ser servido (bundles não servidos terminam em `REJECTED`/`FAILED`). |

---

## 8. `ContextBundle` — Relação com Eventos Emitidos

| Estado alcançado | Evento (`Events.md` §6.1) |
|--------------------|------------------------------|
| `CACHE_LOOKUP` (HIT) | `aios.<tenant>.context.window.cache_hit` |
| `CACHE_LOOKUP` (MISS/BYPASS) | `aios.<tenant>.context.window.cache_miss` |
| `COMPRESSING` (aplicada) | `aios.<tenant>.context.window.compressed` |
| `SERVED` | `aios.<tenant>.context.window.assembled` |
| `REJECTED` | `aios.<tenant>.context.window.rejected` |

---

## 9. `SemanticCacheEntry` — Estados

| Estado | Significado | Tipo |
|--------|--------------|------|
| `PENDING` | Entrada solicitada via `StoreAsync`; ainda não indexada em L1/L2. | Transitório |
| `INDEXED` | Embedding calculado e persistido no L2 (`pgvector`); disponível para *lookup*. | Transitório |
| `SERVING` | Entrada usada em ao menos um `HIT`; `hit_count > 0`, `last_hit_at` atualizado. | Transitório |
| `STALE` | TTL decorrido ou pressão de LRU semântico detectada; ainda fisicamente presente mas não deve mais ser servida como `HIT` fresco. | Transitório |
| `INVALIDATED` | Invalidada por evento de mudança da origem (`010`/`017`/`018-019`) ou invalidação manual. | Transitório |
| `EVICTED` | Removida fisicamente de L1/L2. | **Terminal** |

---

## 10. `SemanticCacheEntry` — Diagrama de Estados (ASCII)

```
 store() ─▶ PENDING ──embedding+persist──▶ INDEXED ──hit──▶ SERVING
                                              │                │
                             TTL/decay ───────┼────────────────┘
                                              ▼
                                            STALE ──event/ttl──▶ EVICTED
                                              ▲
                     memory.item.consolidated │  (invalidação event-driven)
                              ────────────────┴──────────────▶ INVALIDATED ─▶ EVICTED
```

---

## 11. `SemanticCacheEntry` — Tabela de Transições, Gatilhos e Guardas

| De | Para | Gatilho | Guarda | Ação/Evento |
|----|------|---------|---------|--------------|
| `PENDING` | `INDEXED` | `store` | Embedding calculado com sucesso; escrita L2 confirmada. | Grava `prompt_hash`+`prompt_embedding`+`response_ref`. |
| `INDEXED` | `SERVING` | `lookup_hit` | `similarity ≥ similarity_threshold` OU `prompt_hash` exato. | Incrementa `hit_count`; atualiza `last_hit_at`; emite `aios.<tenant>.context.window.cache_hit` (no bundle correspondente). |
| `INDEXED`/`SERVING` | `STALE` | `ttl_decay` | `now > created_at + ttl_seconds` OU pressão de LRU semântico (`context.cache.max_entries_per_tenant` excedido). | `EvictionManager` marca como candidata a remoção. |
| `INDEXED`/`SERVING` | `INVALIDATED` | evento de origem | Recebimento de `memory.item.consolidated`/`memory.item.deleted`/`model.registry.updated`/`knowledge.node.updated` referenciando `scope_ref`/proveniência da entrada. | `EvictionManager.OnSourceChangedAsync`; emite `aios.<tenant>.context.cache.invalidated`. |
| `STALE` | `EVICTED` | `evict` | TTL expirado ou seleção por LRU semântico. | Remove de L1 e L2; emite `aios.<tenant>.context.cache.evicted` (`cause=ttl`\|`lru`). |
| `INVALIDATED` | `EVICTED` | `evict` | Sempre (invalidação já decidiu a remoção). | Remove de L1 e L2; emite `aios.<tenant>.context.cache.evicted` (`cause=invalidated`). |

---

## 12. `SemanticCacheEntry` — Estados Terminais e Invariantes

**Estado terminal:** `EVICTED`. Uma vez evicted, o `cache_id` NÃO DEVE ser
reutilizado; uma nova entrada para o mesmo prompt/escopo recebe novo
`cache_id` (RFC-0001 §5.1 — IDs não são reutilizados).

| # | Invariante |
|---|-----------|
| SM-CACHE-01 | `INDEXED→SERVING` requer `similarity ≥ threshold` — nunca um `HIT` abaixo do limiar configurado. |
| SM-CACHE-02 | `INDEXED`/`SERVING→INVALIDATED` DEVE ser disparado em ≤ 5 s após o evento de mudança da origem (FR-008). |
| SM-CACHE-03 | `STALE`/`INVALIDATED→EVICTED` é executado pelo `EvictionManager` — nenhum outro componente remove entradas de cache diretamente. |
| SM-CACHE-04 | Expurgo LGPD (`memory.item.deleted`) DEVE resultar em `INVALIDATED→EVICTED` rastreável (evento + auditoria), nunca em remoção silenciosa. |
| SM-CACHE-05 | `hit_count` é monotonicamente crescente até `EVICTED` — nunca decrementado. |

---

## 13. Casos Especiais

### 13.1 Degradação graciosa como estado transitório, não terminal

`DEGRADED` é deliberadamente modelado como **transitório**: o objetivo do
`011-Context` é sempre entregar um contexto viável, mesmo que parcial
(`context.degraded.allow_truncate_fallback=true`). Isso reflete o princípio
de FR-010 — degradação de dependência não é motivo de falha do chamador.

### 13.2 Cache invalidado durante `CACHE_LOOKUP` em andamento

Se uma entrada de cache transiciona para `INVALIDATED` (evento externo)
exatamente durante uma consulta `CACHE_LOOKUP` em andamento (corrida), a
guarda de SM-CTX-06 garante que o `ContextBundle` NÃO DEVE aceitar `HIT`
sobre uma entrada com `invalidated_at` preenchido — o *lookup* deve tratar
essa condição como `MISS`, mesmo que a leitura tenha começado antes da
invalidação (checagem de `invalidated_at IS NULL` na mesma consulta que lê
similaridade).

### 13.3 `BYPASS` como resultado distinto de `MISS`

`cache_outcome=BYPASS` (cache indisponível, `AIOS-CTX-0006`, ou
`bypass_cache=true` explícito no request) segue o mesmo ramo de transição
que `MISS` (`CACHE_LOOKUP→BUDGET_ALLOCATED`), mas é registrado
separadamente na telemetria — permite distinguir "não havia nada
cacheável" de "cache estava fora do ar" para fins de SLO (NFR-006) sem
alterar a máquina de estados do bundle.

### 13.4 Truncamento como *fallback* dentro de `COMPRESSING`

A falha de sumarização (`AIOS-CTX-0004`) NÃO transiciona para `FAILED` — o
pipeline permanece em `COMPRESSING`/`DEGRADED` e aplica
`TruncateCompressor` como estratégia de recuperação determinística,
preservando a garantia de que todo `ContextBundle` `SERVED` respeita o
orçamento de tokens (INV SM-CTX-03), mesmo sob falha do modelo de
sumarização.
