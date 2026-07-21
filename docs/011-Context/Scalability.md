---
Documento: Scalability (Modelo de Escala e Concorrência)
Módulo: 011-Context
Status: Draft
Versão: 0.1
Última atualização: 2026-07-20
Responsável (RACI-A): Arquiteto do Módulo 011-Context
ADRs relacionados: ADR-0110, ADR-0111, ADR-0112, ADR-0113, ADR-0116
RFCs relacionados: RFC-0001 (baseline), RFC-0011 (proposta)
Depende de: 001-Architecture, 005-Database, 009-Scheduler, 010-Memory, 017-Model-Router, 020-Communication, 026-Cost-Optimizer, 027-Cluster, 040-Glossary
---

# AIOS — Módulo 011 · Context — Scalability

> Este documento deriva da **§10 (Escalabilidade e Concorrência)** do
> `_DESIGN_BRIEF.md` e das metas de escala dos NFRs (§7.2). **NÃO PODE
> contradizê-lo**. Sharding, limites-alvo e política de locks são canônicos.
> Reutiliza os contratos da
> [RFC-0001](../003-RFC/RFC-0001-Architecture-Baseline.md) e o modelo de shard
> `hash(tenant,agent) mod N` de
> [001-Architecture](../001-Architecture/Architecture.md). Palavras normativas
> conforme RFC 2119/8174.

---

## 1. Objetivo e metas de escala

O `ContextService` (011) DEVE escalar de dezenas até **milhões de agentes**
mantendo o cache semântico como caminho dominante de baixo custo, com
particionamento determinístico e reuso agressivo de sumários para conter o
crescimento de chamadas de embedding/inferência de contexto. As metas
canônicas (brief §7.2/§10) são:

| Meta | Valor-alvo | Requisito |
|------|-----------|-----------|
| Latência `CacheLookup` (hit) | p99 ≤ 20 ms | NFR-001 |
| Latência `Assemble` (miss, sem sumarização) | p99 ≤ 150 ms | NFR-002 |
| Latência `Assemble` (miss, com 1 chamada de sumarização) | p99 ≤ 1200 ms | NFR-003 |
| Context Compression Ratio médio | ≥ 0,50 | NFR-004 |
| Cache hit ratio (regime permanente) | ≥ 0,35 | NFR-006 |
| Throughput de `assemble` | ≥ 500 req/s por réplica | NFR-009 |
| Disponibilidade | ≥ 99,95% mensal | NFR-008 |
| Agentes suportados | milhões (via cache L2 particionado + reuso de `SummaryNode`) | brief §10 |

---

## 2. Modelo de escala horizontal

O serviço é **stateless** no plano de controle (.NET 10) e escala
horizontalmente por réplicas atrás do YARP; o estado vive nos backends
particionados. A dimensão crítica é o **cache/estado**, não a computação de
orquestração em si.

```
                         ┌──────────────── YARP / Gateway ────────────────┐
                         │  roteia por shard = hash(tenant_id, agent_id)   │
                         │  mod context.shard.count                        │
                         └───────┬───────────────┬───────────────┬─────────┘
                                 │               │               │
                        ┌────────▼─────┐ ┌───────▼──────┐ ┌──────▼───────┐
                        │ContextSvc r1 │ │ContextSvc r2 │ │ContextSvc rN │  (stateless, N réplicas)
                        └───┬───┬───┬──┘ └──┬───┬───┬───┘ └──┬───┬───┬───┘
                            │   │   │       │   │   │        │   │   │
        ┌───────────────────┘   │   └───────┼───┼───┼────────┘   │   └────────────┐
        ▼                       ▼           ▼   ▼   ▼            ▼                ▼
   ┌─────────┐          ┌──────────────┐  ┌───────────────┐              ┌────────┐
   │ Redis   │          │ PostgreSQL   │  │ pgvector HNSW │              │ MinIO  │
   │ (L1     │          │ shard/tenant │  │ (L2 cache +   │              │ (blobs │
   │ cache,  │          │ (RLS/tenant) │  │  fragments)   │              │  de    │
   │ locks)  │          └──────────────┘  └───────────────┘              │fragmnt)│
   └─────────┘                                                            └────────┘
       ▲                                                                          ▲
       │ estado quente (L1)                              estado durável (L2/bundles/budgets)
       └──────────────────────────────────────────────────────────────────────────┘
```

- **Réplicas do serviço** escalam com a taxa de `assemble`/`cache lookup`
  (CPU-bound na fusão/*ranking*/dedup do `ContextAssembler`).
- **Backends** escalam com o volume de dados (bundles, fragments, entradas de
  cache por shard/tenant).
- Qualquer réplica PODE servir qualquer tenant, mas o roteamento por shard
  maximiza localidade de cache L1 (Redis) e reduz *fan-out* de leitura.

---

## 3. Particionamento e sharding

### 3.1 Estratégia canônica (brief §10)

| Dimensão | Estratégia | Chave de partição |
|----------|-----------|--------------------|
| Shard global | `shard = hash(tenant_id, agent_id) mod context.shard.count` | (tenant, agent) |
| Cache L1 (Redis) | Partição por shard, TTL curto (`context.cache.l1_redis_ttl_seconds`) | (tenant, shard) |
| Cache L2 (`pgvector`) | Particionado por tenant; índice HNSW (default) ou IVFFLAT por partição | tenant |
| `ContextBundle`/`ContextFragment` | RLS por `tenant_id`; retenção efêmera (`expires_at`) | tenant |
| `SummaryNode` | Árvore self-FK por origem sumarizada (`source_urn`), TTL de reuso | tenant + `source_urn` |
| Blobs (MinIO) | Prefixo por tenant; referenciados por `content_ref` | tenant |

### 3.2 Diagrama de particionamento

```
tenant "acme"
   ├── agent A ──hash──▶ shard 3  ──▶ Redis shard3 (L1) + PG RLS(acme) + pgvector part(acme)
   ├── agent B ──hash──▶ shard 41 ──▶ Redis shard41(L1) + PG RLS(acme) + pgvector part(acme)
   └── ...

Cache L2 (pgvector, por tenant):
   ctxcache(acme) ── HNSW índice ── prompt_embedding, prompt_hash(fast path)
   └── invalidação por evento (memory.item.consolidated/deleted, model.registry.updated)
```

### 3.3 Rebalanceamento

- Aumentar `N` (`context.shard.count`, não recarregável — brief §8) exige
  *rehash*; DEVE ser feito com migração online por tenant, sem *downtime*
  (janela de dupla-escrita + *cutover*), análoga ao modelo de `010-Memory`.
- Trocar `context.cache.vector_index` (HNSW↔IVFFLAT) ou
  `context.embedding.dimensions` exige reindex/migração — ambas as chaves são
  **não recarregáveis** (brief §8) e não devem coincidir com um rollout de
  código (ver [Deployment.md](./Deployment.md) §6).

---

## 4. Cache semântico como caminho dominante (rumo a milhões de agentes)

A chave para escalar a milhões de agentes é que o **cache semântico absorve a
maior parte do tráfego repetitivo**, evitando reinferência e reduzindo
proporcionalmente as chamadas de embedding/sumarização ao `017-Model-Router`
(brief §10, "Rumo a milhões de agentes").

| Sinal (evento consumido) | Ação de contexto | Efeito de escala |
|---------------------------|-------------------|-------------------|
| `aios.<tenant>.agent.lifecycle.suspended` | *Evict* de cache/contexto por-agente (escopo `agent`) | Libera Redis/pgvector do agente suspenso |
| `aios.<tenant>.agent.lifecycle.terminated` | Expurgo de bundles/fragments efêmeros do agente | Libera estado durável residual |
| `aios.<tenant>.memory.item.consolidated` | Invalida cache/`SummaryNode` dependentes | Evita servir cache stale; força *miss* controlado |
| `aios.<tenant>.model.registry.updated` | Invalida limites/tokenizers cacheados | Realinha `TokenBudgeter` sem *downtime* |

```
   milhões de agentes registrados
            │
   ┌────────┴─────────┐
   │ ativos (quentes) │  ← Redis L1: cache semântico das tarefas recorrentes
   └────────┬─────────┘
            │ suspend/terminate
            ▼
   ┌──────────────────┐   reidrata sob nova assemble()   ┌──────────────────┐
   │ evicted (L1/L2)  │──────────────────────────────────▶│ cache-fill novo  │
   │ liberado         │                                    │ (single-flight)  │
   └──────────────────┘                                    └──────────────────┘
```

**Estimativa (back-of-envelope, brief §10):** 10⁶ agentes ativos × 35% de cache
hit ratio (NFR-006, meta mínima) ⇒ redução de aproximadamente 35% das chamadas
de embedding/inferência de contexto que, de outra forma, chegariam ao
`017-Model-Router`. O reuso agressivo de `SummaryNode` (FR-014, COULD) reduz
adicionalmente a fração de chamadas de sumarização por *miss*.

---

## 5. Concorrência e locks

### 5.1 Modelo de concorrência (brief §10)

| Caminho | Modelo | Justificativa |
|---------|--------|----------------|
| `CacheLookup` (leitura) | Sem lock | Escala com réplicas de leitura; caminho quente (NFR-001) |
| Cache-fill em `MISS` (escrita de `SemanticCacheEntry`) | **Lock distribuído** por `(tenant, cache_key)` (Redis `SET NX PX`) | Evita *thundering herd* quando N requisições concorrentes colidem na mesma chave de similaridade |
| `Assemble` (pipeline budget→retrieve→rank→dedup→compress) | Pipeline assíncrono, sem lock entre estágios | Cada estágio é independente por bundle; sem estado compartilhado mutável |
| `EmbeddingClient`/`RelevanceRanker` | *Batching* de embeddings | Amortiza latência de chamada ao `017` sobre múltiplos fragmentos |
| `HierarchicalCompressor` | Sumarização paralela por nível (*map*), sequencial na *reduction* | *Map-reduce* clássico: nível N processa fragmentos independentes em paralelo; a redução final é sequencial por dependência de dados |
| `BudgetProfile` upsert | *Optimistic concurrency* por `updated_at` | Escrita rara; conflito tratado como última-escrita-vence com auditoria |

> **Regra:** locks distribuídos são usados **apenas** para escrita de cache-fill
> por `(tenant, cache_key)` — nunca no caminho quente de leitura
> (`CacheLookup`). Um lock longo no caminho quente violaria NFR-001.

### 5.2 Fluxo de *single-flight* em cache-fill

```
  N requisições concorrentes, mesma (tenant, cache_key), todas em MISS
            │
   ┌────────▼─────────┐  falha (lock já detido)  ┌───────────────────────────┐
   │ SET NX PX lock    │─────────────────────────▶│ aguarda / observa a chave │
   │ (tenant,cache_key)│                            │ (poll curto ou pub/sub)   │
   └────────┬──────────┘                            └────────────┬──────────────┘
            │ sucesso (dono do lock)                              │ lock liberado
            ▼                                                     ▼
   executa pipeline completo                              lê SemanticCacheEntry
   (retrieve→rank→dedup→compress)                          recém-escrita (HIT)
            │
            ▼
   grava SemanticCacheEntry + libera lock + emite context.window.cache_miss/assembled
```

---

## 6. Backpressure e admissão

O rate-limit por tenant aplica *backpressure* antes da saturação, consistente
com o brief §8/§10:

```
   assemble(...) ───▶ ┌────────────────────────────────────┐
                      │ taxa < context.ratelimit.            │
                      │ assemble_rps_per_tenant (1000)?      │
                      └───────┬───────────────┬──────────────┘
                          sim │               │ não
                              ▼               ▼
                      ┌───────────┐    ┌─────────────────────────┐
                      │ admite    │    │ 429 AIOS-CTX-0015        │
                      │ pipeline  │    │ (RateLimited, retriable) │
                      └─────┬─────┘    └─────────────────────────┘
                            ▼
                  ┌──────────────────────────┐
                  │ orçamento viável?         │──não──▶ 422 AIOS-CTX-0001 (BudgetInfeasible)
                  │ (TokenBudgeter)           │
                  └──────────┬────────────────┘
                             │ sim
                             ▼
                  cache lookup → retrieve → rank → dedup → compress → persist + outbox
```

| Mecanismo | Chave/limite | Código |
|-----------|--------------|--------|
| Rate-limit por tenant | `context.ratelimit.assemble_rps_per_tenant` (1000) | `AIOS-CTX-0015` |
| Orçamento infeasível | contexto mínimo não cabe em `token_budget` | `AIOS-CTX-0001` |
| Payload de entrada excedente | `context.limits.max_payload_bytes` (4 MiB) | `AIOS-CTX-0013` |
| *Deadline* de recall (`010`) | `context.retrieval.memory_deadline_ms` (120 ms) | `AIOS-CTX-0005` (degrada, não bloqueia) |

Consumo de tokens/custo evitado é reportado ao **026-Cost-Optimizer** via
métricas `aios_context_compression_ratio`/`cost_saved_usd` e eventos
`context.window.cache_hit` (ver [Metrics.md](./Metrics.md), [Events.md](./Events.md)).

---

## 7. Leitura em escala (cache L1/L2)

Para sustentar ≥ 500 req/s por réplica (NFR-009) com p99 baixo (NFR-001/002):

| Técnica | Descrição |
|---------|-----------|
| Cache L1 Redis | Sub-ms; TTL curto (`context.cache.l1_redis_ttl_seconds`, default 300 s); evita busca vetorial em `hit` recorrente |
| Cache L2 `pgvector` (HNSW) | Durável; ANN de baixa latência para o *slow path* de similaridade |
| *Fast path* de hash exato | `prompt_hash` casa prompts idênticos sem custo de busca vetorial |
| `top_k` limitado no recall | `context.retrieval.top_k` (default 32) limita candidatos processados por `RelevanceRanker`/`RedundancyEliminator` |
| Batching de embeddings | `EmbeddingClient` agrupa fragmentos candidatos em uma única chamada ao `017` |

> A eficácia de compressão (**Context Compression Ratio ≥ 0,50**, NFR-004) e a
> preservação de qualidade (Δ Task Completion Rate ≥ -2%, NFR-005) NÃO DEVEM
> ser sacrificadas por escala: `context.compression.max_levels` e
> `min_compression_ratio` são os parâmetros de ajuste, não atalhos para reduzir
> qualidade globalmente.

---

## 8. Eviction e crescimento sub-linear do cache

A eviction roda continuamente e por evento (não em lote isolado, dado o
caráter efêmero do módulo — brief §1.3, nota de fronteira):

- `EvictionManager` combina **TTL** (`context.cache.default_ttl_seconds`),
  **LRU semântico** (`last_hit_at`, pressão de `context.cache.max_entries_per_tenant`)
  e **invalidação *event-driven*** (`memory.item.consolidated/deleted`,
  `knowledge.node.updated`).
- `SummaryNode` expira por `expires_at` própria, independente do ciclo de vida
  do cache de resposta — reuso por similaridade (FR-014) reduz recomputação
  sem violar a fronteira de efemeridade do brief §1.3.
- O efeito combinado é **crescimento sub-linear** do volume de cache por
  tenant: sem eviction, o número de entradas cresceria linearmente com o
  tráfego; com TTL+LRU+event-driven, converge para o *working set* de prompts
  recorrentes.

---

## 9. Limites teóricos e gargalos previstos

| Recurso | Limite prático (por unidade) | Gargalo | Mitigação / caminho evolutivo |
|---------|-------------------------------|---------|---------------------------------|
| Entradas de cache por tenant | `context.cache.max_entries_per_tenant` (default 10⁶, faixa 10³–10⁸) | Tamanho do índice HNSW em RAM | Eviction agressiva (TTL+LRU); partição por tenant |
| Throughput `assemble` por réplica | ~500 req/s (NFR-009) | CPU de *ranking*/dedup/orquestração | Mais réplicas (HPA); *batch* de embeddings |
| Latência `CacheLookup` | p99 ≤ 20 ms (NFR-001) | Round-trip Redis + eventual fallback a `pgvector` | L1 dominante; L2 só no *miss* de L1 |
| Latência com sumarização | p99 ≤ 1200 ms (NFR-003) | Chamada síncrona ao modelo de sumarização (`017`) | Roteamento a modelo barato (`summarization_model_hint`); *map* paralelo |
| Redis (L1) | RAM do cluster | *Working set* de cache ativo | TTL curto; eviction por pressão |
| `pgvector` (L2) | Tamanho do índice HNSW por tenant | Memória/latência de busca ANN | IVFFLAT alternativo em datasets menores; partição por tenant |
| MinIO | Capacidade S3 | Throughput de objeto | Offload só acima de `max_inline_bytes`; retenção alinhada à efemeridade |

---

## 10. Riscos e alternativas

| Risco | Impacto | Mitigação | Alternativa descartada |
|-------|---------|-----------|---------------------------|
| *Hot tenant* satura um shard de cache | Contenção localizada, `assemble` mais lento | Rate-limit por tenant; sharding por (tenant, agent); `max_entries_per_tenant` | Shard único global (rejeitado: SPOF/limite) |
| Índice HNSW L2 estoura RAM em alto volume de entradas | p99 de `CacheLookup`/`Assemble` degrada | Partição por tenant; IVFFLAT como alternativa configurável; eviction agressiva | Busca vetorial exata (rejeitado: viola NFR-001) |
| *Rehash* ao crescer `context.shard.count` causa *downtime* | Indisponibilidade durante rebalanceamento | Migração online por tenant (dupla-escrita + *cutover*) | *Stop-the-world* (rejeitado: viola NFR-008) |
| Lock de cache-fill vira gargalo sob alta concorrência na mesma chave | Cache-fill enfileira | Lock apenas por `(tenant, cache_key)`; TTL curto de lock; *single-flight* libera assim que grava | Lock global de cache (rejeitado: serializa tudo) |
| Cache L1 (Redis) inconsistente com L2 (pgvector) após invalidação | *Hit* stale servido brevemente | Invalidação prioritária por evento + TTL curto de L1 (`l1_redis_ttl_seconds`) | Cache sem invalidação por evento (rejeitado: viola NFR-011/§12.3) |
| Reuso de `SummaryNode` amplifica erro de sumarização antiga | Degradação silenciosa de qualidade | `reuse_threshold` (0,95) conservador + TTL de reuso; verificação de fidelidade no `HierarchicalCompressor` | Reuso irrestrito sem limiar (rejeitado: viola NFR-005) |

---

## 11. Rastreabilidade

| Requisito | Coberto por (esta doc) |
|-----------|---------------------------|
| NFR-001 (latência cache hit) | §7, §9 |
| NFR-002/003 (latência assemble) | §2, §7, §9 |
| NFR-004 (compression ratio) | §7, §8 |
| NFR-006 (cache hit ratio) | §4, §7, §8 |
| NFR-009 (throughput por réplica) | §2, §3, §9 |
| FR-006 (cache semântico) | §4, §5.2, §7 |
| FR-008 (invalidação por evento) | §4, §8 |
| FR-014 (reuso de `SummaryNode`) | §8, §10 |

---

## 12. Referências

- [_DESIGN_BRIEF.md §10](./_DESIGN_BRIEF.md) — fonte única (sharding, locks,
  backpressure, cache).
- [001-Architecture §12](../001-Architecture/Architecture.md) — modelo de
  shard global.
- [ADR.md](./ADR.md) — ADR-0110/0111/0112/0113/0116.
- [FailureRecovery.md](./FailureRecovery.md) — *bulkhead*, degradação, RTO/RPO.
- [Database.md](./Database.md) — particionamento físico e índices.
- [Configuration.md](./Configuration.md) — chaves `context.shard.*`,
  `context.cache.*`, `context.ratelimit.*`.
- Glossário: [Shard, Semantic Cache, Backpressure, Hot State](../040-Glossary/Glossary.md).
