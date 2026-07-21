---
Documento: ClassDiagrams
Módulo: 011-Context
Status: Draft
Versão: 0.1
Última atualização: 2026-07-20
Responsável (RACI-A): Arquiteto do Módulo 011-Context
ADRs relacionados: ADR-0110, ADR-0111, ADR-0112, ADR-0113, ADR-0114, ADR-0116, ADR-0117; ADR-0001..ADR-0010 (herdadas)
RFCs relacionados: RFC-0001 (baseline); RFC-0011 (proposta)
Depende de: 010-Memory, 017-Model-Router, 022-Policy, 024-Observability
---

# 011-Context — ClassDiagrams

> Estruturas, interfaces, contratos e invariantes dos componentes internos do
> `ContextService` descritos em `Architecture.md` e em `_DESIGN_BRIEF.md`
> §2–§3. Os diagramas usam notação UML adaptada para ASCII: `◆──` composição,
> `◇──` agregação, `┈┈▷` dependência/uso ou implementação de interface,
> `──▶` associação dirigida.

## Índice

1. Convenções de notação
2. Diagrama de classes — Pipeline de assembly (`ContextAssembler`)
3. Interface `ICompressionStrategy` e implementações
4. Diagrama de classes — Cache semântico (L1/L2)
5. Diagrama de classes — Budget e tokenização
6. Diagrama de classes — Recuperação, ranking e deduplicação
7. Diagrama de classes — Clientes de dependências externas
8. Diagrama de classes — Persistência e eventos
9. Entidades de dados canônicas (contratos de mensagem)
10. Invariantes globais
11. Relações entre componentes (visão consolidada)

---

## 1. Convenções de Notação

```
┌───────────────────────┐
│      ClassName         │   « estereótipo »  (ex.: «service», «interface», «entity», «value object»)
├───────────────────────┤
│ - campoPrivado: Tipo   │
│ + campoPublico: Tipo   │
├───────────────────────┤
│ + Metodo(args): Retorno│
└───────────────────────┘

  A ──▷ B     : A depende de/usa B
  A ──▶ B     : A tem associação dirigida com B (referência)
  A ◆── B     : A é composto por B (ciclo de vida acoplado)
  A ◇── B     : A agrega B (ciclo de vida independente)
  A ┈┈▷ I     : A implementa a interface I
```

Visibilidade: `+` público, `-` privado, `#` protegido. Tipos seguem convenção
.NET (`string`, `int`, `decimal`, `DateTimeOffset`, `Ulid`,
`IReadOnlyList<T>`, `Task<T>` para métodos assíncronos, `float[]`/`vector`
para embeddings).

---

## 2. Diagrama de Classes — Pipeline de Assembly

```
┌────────────────────────────────────┐
│         ContextApiGateway            │ «service · PEP entrypoint»
├────────────────────────────────────┤
│ - _idempotency: IIdempotencyStore    │
│ - _guard: IContextPolicyGuard        │
├────────────────────────────────────┤
│ + Assemble(req: AssembleRequest,     │
│     idemKey: string):                │
│     Task<ContextBundle>              │
│ + GetBundle(bundleId: Ulid):         │
│     Task<ContextBundle?>             │
│ + Compress(req: CompressRequest):    │
│     Task<CompressResponse>           │
│ + EstimateTokens(req):               │
│     Task<EstimateTokensResponse>     │
└────────────────┬────────────────────┘
                  │ ┈┈▷ valida envelope (traceparent, X-AIOS-Tenant, Idempotency-Key)
                  ▼
┌────────────────────────────────────┐
│        ContextPolicyGuard            │ «service · PEP»
├────────────────────────────────────┤
│ - _pdp: IPolicyClient                │
├────────────────────────────────────┤
│ + Authorize(subject, action,          │
│     resource: Urn): Task<PdpDecision> │
└────────────────┬────────────────────┘
                  │ allow ──▶
                  ▼
┌──────────────────────────────────────────────────────────────────────┐
│                        ContextAssembler                                │ «service · orquestrador»
├──────────────────────────────────────────────────────────────────────┤
│ - _cache: ISemanticCacheManager                                        │
│ - _budgeter: ITokenBudgeter                                            │
│ - _retriever: ISelectiveRetriever                                       │
│ - _ranker: IRelevanceRanker                                            │
│ - _dedup: IRedundancyEliminator                                        │
│ - _compressor: ICompressionStrategy                                    │
│ - _store: IBundleStore                                                 │
│ - _publisher: IEventPublisher                                          │
├──────────────────────────────────────────────────────────────────────┤
│ + AssembleAsync(req: AssembleRequest):                                 │
│     Task<ContextBundle>                                                │
│ - TryServeFromCacheAsync(req): Task<ContextBundle?>                    │
│ - AllocateBudgetAsync(req): Task<TokenBudget>                          │
│ - RetrieveCandidatesAsync(req, budget): Task<IReadOnlyList<ContextFragment>> │
│ - RankAsync(candidates): Task<IReadOnlyList<ContextFragment>>          │
│ - DeduplicateAsync(ranked): Task<IReadOnlyList<ContextFragment>>       │
│ - CompressToFitAsync(deduped, budget): Task<IReadOnlyList<ContextFragment>> │
└────────────────────────────────────────────────────────────────────────┘
```

**Contrato de `AssembleAsync`:** DEVE ser idempotente por
`Idempotency-Key`/`request_id` (RFC-0001 §5.5); DEVE retornar
`tokens_in ≤ token_budget` em todo bundle no estado `SERVED` (FR-001); DEVE
respeitar o *deadline* de recall (`context.retrieval.memory_deadline_ms`) sem
bloquear indefinidamente; NÃO DEVE persistir efeitos colaterais fora de uma
única transação (`BundleStore` + *outbox*).

---

## 3. Interface `ICompressionStrategy` e Implementações

```
┌──────────────────────────────────────────────┐
│         «interface» ICompressionStrategy       │
├──────────────────────────────────────────────┤
│ + Method: CompressionMethod { get; }           │
├──────────────────────────────────────────────┤
│ + CompressAsync(fragments:                      │
│     IReadOnlyList<ContextFragment>,             │
│     targetTokens: int):                         │
│     Task<CompressionResult>                     │
└───────────────────┬──────────────────────────┘
                     △ implementa
      ┌──────────────┼──────────────────┐
      │              │                  │
┌──────────────────┐ ┌────────────────────┐ ┌──────────────────────┐
│ HierarchicalCompr.│ │ ExtractiveCompressor│ │ TruncateCompressor    │
│ «strategy·default»│ │ «strategy·opt-in»   │ │ «strategy·fallback»   │
├──────────────────┤ ├────────────────────┤ ├──────────────────────┤
│ - _maxLevels: int  │ │ - _sentenceScorer   │ │ (sem dependências     │
│ - _summModel:      │ │ - _topN: int        │ │  externas)            │
│   IModelRouterClient│ ├────────────────────┤ ├──────────────────────┤
├──────────────────┤ │ + CompressAsync(...) │ │ + CompressAsync(...)  │
│ + CompressAsync(...) │  → seleciona N       │ │  → corta pelo limite  │
│  → map-reduce em     │    sentenças de maior│ │    de tokens,         │
│    _maxLevels níveis;│    score léxico/     │ │    determinístico,    │
│    fallback interno  │    posicional        │ │    sem chamada de LLM │
│    → TruncateCompressor em falha (AIOS-CTX-0004)│└──────────────────────┘
└──────────────────┘ └────────────────────┘
```

**Invariantes da interface (contrato estável — FR-003, ADR-0110, RFC-0011):**

1. `ICompressionStrategy` NÃO DEVE expor tipos específicos de implementação em
   sua assinatura pública — apenas `ContextFragment`, `CompressionResult`,
   `CompressionMethod`.
2. A troca de estratégia (`hierarchical`/`extractive`/`truncate`) DEVE ser
   feita por configuração (`context.compression.method`), NÃO DEVE exigir
   mudança de API externa nem de schema de eventos
   (`ContextFragment.compression_method` absorve a variação).
3. Toda implementação DEVE retornar `CompressionResult.tokensAfter ≤
   targetTokens` — quando não for possível preservando semântica mínima, DEVE
   propagar falha (`AIOS-CTX-0004`) para o chamador acionar o *fallback*
   determinístico de `TruncateCompressor`.
4. `HierarchicalCompressor` DEVE registrar, para cada nível, o `SummaryNode`
   correspondente (parent/child), permitindo reconstrução da árvore de
   sumarização (`Database.md` §`SummaryNode`).
5. `TruncateCompressor` NÃO DEVE chamar nenhum modelo externo — é o único
   caminho de compressão que funciona mesmo com `017-Model-Router`
   completamente indisponível (degradação graciosa, FM-04).

---

## 4. Diagrama de Classes — Cache Semântico (L1/L2)

```
┌────────────────────────────────────┐
│        SemanticCacheManager          │ «service»
├────────────────────────────────────┤
│ - _l1: IRedisCacheStore              │
│ - _l2: IPgVectorCacheStore           │
│ - _embedding: IEmbeddingClient        │
│ - _lock: IDistributedLock             │
├────────────────────────────────────┤
│ + LookupAsync(prompt: string,          │
│     modelId: string, scope: CacheScope): │
│     Task<CacheLookupResult>            │
│ + StoreAsync(entry: SemanticCacheEntry): │
│     Task<Ulid>                          │
│ + InvalidateAsync(cacheId: Ulid):       │
│     Task                                │
│ + InvalidateByScopeAsync(scope,          │
│     sourceUrn: Urn?): Task<int>          │
│ - TryFastPathAsync(promptHash):          │
│     Task<SemanticCacheEntry?>            │  // exact match
│ - TrySlowPathAsync(embedding,             │
│     threshold: decimal):                  │
│     Task<SemanticCacheEntry?>             │  // ANN + similarity ≥ threshold
└────────────────┬────────────────────┘
                  │ ◇── usa
                  ▼
┌────────────────────────────────────┐      ┌────────────────────────────────────┐
│           EvictionManager             │      │        «value object» CacheLookupResult│
├────────────────────────────────────┤      ├────────────────────────────────────┤
│ - _l1: IRedisCacheStore               │      │ + Outcome: CacheOutcome (HIT|MISS|BYPASS)│
│ - _l2: IPgVectorCacheStore             │      │ + Entry: SemanticCacheEntry?          │
├────────────────────────────────────┤      │ + Similarity: decimal?                │
│ + EvictExpiredAsync(): Task<int>       │      │ + TokensSaved: int?                    │
│ + EvictByLruAsync(tenantId,             │      │ + CostSavedUsd: decimal?               │
│     maxEntries: int): Task<int>         │      └────────────────────────────────────┘
│ + OnSourceChangedAsync(sourceUrn: Urn):  │
│     Task<int>  // invalidação event-driven│
└────────────────────────────────────┘

CacheOutcome = HIT | MISS | BYPASS
CacheScope   = TENANT | AGENT | TASK_TYPE
```

**Invariantes de cache (NFR-006, NFR-011, ADR-0111, ADR-0112, ADR-0119):**

1. `LookupAsync` DEVE tentar `TryFastPathAsync` (hash exato) antes de
   `TrySlowPathAsync` (ANN por similaridade) — o *fast path* é O(1) e NÃO
   DEVE gerar chamada de embedding.
2. `TrySlowPathAsync` DEVE retornar `HIT` **apenas** quando
   `similarity ≥ entry.similarity_threshold` — abaixo disso é `MISS`
   (nunca um `HIT` aproximado silencioso).
3. Toda leitura/escrita de cache DEVE ser namespaced por `tenant_id`
   (chave Redis prefixada, RLS no L2) — nenhuma consulta PODE atravessar
   tenants (INV de isolamento, NFR-014).
4. `StoreAsync` DEVE adquirir lock distribuído (`SET NX PX`) por
   `(tenant, cache_key)` antes de popular o L2, evitando *thundering herd*
   de cache-fill concorrente sobre a mesma chave (*single-flight*).
5. `OnSourceChangedAsync` DEVE ser chamado em reação aos eventos consumidos
   de `010`/`017`/`018-019`/`022` (`Events.md` §6.2) e DEVE invalidar toda
   entrada cujo `scope_ref`/proveniência referencie o `sourceUrn` afetado
   dentro de 5 s (FR-008).
6. `EvictByLruAsync` NÃO DEVE remover entradas com `last_hit_at` dentro da
   janela de retenção mínima configurada, salvo quando
   `hit_count = 0 ∧ ttl_seconds` expirado (política combinada TTL+LRU
   semântico, ADR-0116).

---

## 5. Diagrama de Classes — Budget e Tokenização

```
┌────────────────────────────────────┐
│            TokenBudgeter              │ «service»
├────────────────────────────────────┤
│ - _profiles: IBudgetProfileStore       │
│ - _modelRouter: IModelRouterClient      │
│ - _counter: ITokenCounter                │
├────────────────────────────────────┤
│ + AllocateAsync(req: AssembleRequest):   │
│     Task<TokenBudget>                     │
│ - ResolveProfile(scope, scopeRef):        │
│     Task<BudgetProfile>                    │
│ - ValidateAllocation(profile):             │
│     void  // soma dos pesos == 1.0         │
└────────────────┬────────────────────┘
                  │ ◇── usa
                  ▼
┌────────────────────────────────────┐       ┌────────────────────────────────────┐
│            TokenCounter               │       │  «value object» TokenBudget           │
├────────────────────────────────────┤       ├────────────────────────────────────┤
│ - _tokenizers: ITokenizerRegistry       │       │ + ModelId: string                       │
├────────────────────────────────────┤       │ + ModelMaxTokens: int                   │
│ + CountAsync(text: string,               │       │ + ReservedOutputTokens: int              │
│     modelId: string): Task<int>          │       │ + TotalBudget: int                       │
│ + EstimateAsync(bytes: long,             │       │ + PerSection: IReadOnlyDictionary<        │
│     modelId: string): Task<int>          │       │     ContextSectionKind, int>              │
└────────────────────────────────────┘       └────────────────────────────────────┘

┌────────────────────────────────────┐
│    «entity» BudgetProfile             │
├────────────────────────────────────┤
│ + ProfileId: Ulid          «PK»        │
│ + TenantId: string          «RLS»       │
│ + Scope: BudgetScope (TENANT|AGENT|TASK_TYPE) │
│ + ScopeRef: string?                      │
│ + ReservedOutputRatio: decimal (0.05-0.60) │
│ + AllocSystem/Instruction/Memory/History/  │
│   Tools: decimal                          │
│ + MinCompressionRatio: decimal              │
│ + UpdatedAt: DateTimeOffset                  │
└────────────────────────────────────┘

ContextSectionKind = SYSTEM | INSTRUCTION | MEMORY | HISTORY | TOOLS
```

**Invariantes de budget (FR-002, NFR-010, ADR-0113):**

1. `AllocSystem + AllocInstruction + AllocMemory + AllocHistory + AllocTools
   = 1.0` (± 0.001 de tolerância de ponto flutuante) — violação rejeita a
   escrita com `AIOS-CTX-0008` (`ValidateAllocation`).
2. `TokenBudget.TotalBudget = ModelMaxTokens × (1 − ReservedOutputRatio)` —
   nunca DEVE exceder o limite retornado por `ModelRouterClient`.
3. `TokenCounter.EstimateAsync` DEVE ter erro ≤ 2% em relação à contagem
   exata do tokenizer do provedor (NFR-010); quando o tokenizer exato não
   está disponível, a estimativa DEVE ser sinalizada como aproximada nos
   metadados internos do `ContextFragment`.
4. `ResolveProfile` DEVE aplicar precedência `task_type > agent > tenant`
   (mais específico vence) e cair no perfil `tenant` *default* quando nenhum
   perfil mais específico existir.

---

## 6. Diagrama de Classes — Recuperação, Ranking e Deduplicação

```
┌────────────────────────────────────┐
│         SelectiveRetriever            │ «service»
├────────────────────────────────────┤
│ - _memory: IMemoryClient               │
├────────────────────────────────────┤
│ + RetrieveAsync(agentUrn: Urn,          │
│     query: string, topK: int,           │
│     deadlineMs: int):                    │
│     Task<IReadOnlyList<ContextFragment>>  │
└────────────────┬────────────────────┘
                  │ candidatos ──▶
                  ▼
┌────────────────────────────────────┐
│           RelevanceRanker              │ «service»
├────────────────────────────────────┤
│ - _embedding: IEmbeddingClient          │
├────────────────────────────────────┤
│ + RankAsync(fragments,                   │
│     query: string):                       │
│     Task<IReadOnlyList<ContextFragment>>  │
│ - Score(fragment, query):                  │
│     decimal  // w1·similaridade+w2·recência+w3·prioridade │
└────────────────┬────────────────────┘
                  │ ranked ──▶
                  ▼
┌────────────────────────────────────┐
│        RedundancyEliminator            │ «service»
├────────────────────────────────────┤
│ - _embedding: IEmbeddingClient          │
│ - _threshold: decimal                    │
├────────────────────────────────────┤
│ + DeduplicateAsync(ranked):               │
│     Task<IReadOnlyList<ContextFragment>>  │
│ - IsNearDuplicate(a, b): bool              │  // cosine ≥ context.dedup.cosine_threshold
└────────────────────────────────────┘

┌────────────────────────────────────┐
│           EmbeddingClient              │ «adapter · gRPC client»
├────────────────────────────────────┤
│ + EmbedAsync(text: string):              │
│     Task<float[]>                         │
│ + EmbedBatchAsync(texts:                  │
│     IReadOnlyList<string>):                │
│     Task<IReadOnlyList<float[]>>           │
└────────────────────────────────────┘
```

**Invariantes de retrieval/ranking/dedup (FR-004, FR-005, ADR-0113):**

1. `RetrieveAsync` NÃO DEVE bloquear além de `deadlineMs`
   (`context.retrieval.memory_deadline_ms`, default 120 ms) — no timeout,
   DEVE retornar os candidatos parciais já recebidos e sinalizar
   `DEGRADED` ao chamador (`AIOS-CTX-0005`).
2. `RankAsync` DEVE preservar `ContextFragment.relevance_score` calculado de
   forma determinística para o mesmo `(fragment, query, pesos)` — auditável.
3. `DeduplicateAsync` DEVE colapsar fragmentos com `cosine ≥
   context.dedup.cosine_threshold` (default 0.95) preservando o de maior
   `relevance_score`; NÃO DEVE eliminar um item semanticamente único mesmo
   que textualmente semelhante abaixo do limiar.
4. `EmbeddingClient` NÃO DEVE conter lógica de ranking ou cache — é
   estritamente um *adapter* para o `017-Model-Router`.

---

## 7. Diagrama de Classes — Clientes de Dependências Externas

```
┌────────────────────────────────────┐   ┌────────────────────────────────────┐
│         ModelRouterClient             │   │           MemoryClient                │
│       «adapter · gRPC client»          │   │        «adapter · gRPC/NATS client»    │
├────────────────────────────────────┤   ├────────────────────────────────────┤
│ + GetModelLimitsAsync(modelId):         │   │ + RecallAsync(agentUrn: Urn,           │
│     Task<ModelLimits>                    │   │     query: string, topK: int):         │
│ + GetTokenizerAsync(modelId):            │   │     Task<IReadOnlyList<ContextFragment>>│
│     Task<ITokenizer>                     │   └────────────────────────────────────┘
│ + EmbedAsync(text): Task<float[]>         │
│ + SummarizeAsync(text, targetTokens,       │   ┌────────────────────────────────────┐
│     modelHint: string): Task<string>       │   │           PolicyClient                 │
└────────────────────────────────────┘   │        «adapter · gRPC client»         │
                                           ├────────────────────────────────────┤
                                           │ + Authorize(subject, action,           │
                                           │     resource): Task<PdpDecision>       │
                                           └────────────────────────────────────┘
```

**Invariantes de fronteira externa:**

1. Nenhum destes *clients* DEVE conter lógica de montagem/compressão de
   contexto — apenas tradução de contrato (adapter) para o serviço remoto.
2. Toda chamada DEVE propagar `traceparent`/`X-AIOS-Tenant` (RFC-0001 §5.6).
3. Sob timeout/indisponibilidade, cada cliente DEVE aplicar a estratégia de
   `_DESIGN_BRIEF.md` §9 (ex.: `ModelRouterClient` → limites cacheados,
   `PolicyClient` → *default deny*, `MemoryClient` → contexto parcial/
   `DEGRADED`).
4. `ModelRouterClient.SummarizeAsync` DEVE receber um `modelHint` (roteado
   para modelo barato, `context.compression.summarization_model_hint`) —
   o `011` NÃO DEVE escolher o modelo diretamente, apenas sugerir a classe
   de custo/qualidade desejada.

---

## 8. Diagrama de Classes — Persistência e Eventos

```
┌────────────────────────────────────┐
│            BundleStore                │ «service»
├────────────────────────────────────┤
│ - _db: NpgsqlConnection                 │
│ - _blob: IBlobStoreAdapter               │
├────────────────────────────────────┤
│ + SaveAsync(bundle: ContextBundle,       │
│     fragments: IReadOnlyList<ContextFragment>): │
│     Task                                  │
│ + GetAsync(bundleId: Ulid):               │
│     Task<ContextBundle?>                  │
│ - OffloadIfLargeAsync(fragment):           │
│     Task<ContextFragment>  // > max_inline_bytes → MinIO
└────────────────┬────────────────────┘
                  │ ◆── compõe
                  ▼
┌────────────────────────────────────┐
│           EventPublisher               │ «service · outbox»
├────────────────────────────────────┤
│ + PublishAsync(subject: string,          │
│     evt: CloudEvent): Task                │  // transação única: INSERT + outbox row
└────────────────────────────────────┘

┌────────────────────────────────────┐
│         IdempotencyStore               │ «service»
├────────────────────────────────────┤
│ + TryBeginAsync(idemKey: string,          │
│     requestHash: string):                  │
│     Task<IdempotencyLease>                  │  // Redis (rápido) + PostgreSQL (≥24h)
│ + CompleteAsync(idemKey: string,           │
│     response: JsonDocument): Task            │
│ + GetIfExistsAsync(idemKey: string):        │
│     Task<JsonDocument?>                      │
└────────────────────────────────────┘

┌────────────────────────────────────┐
│           TelemetryEmitter              │ «service · cross-cutting»
├────────────────────────────────────┤
│ + RecordAssembly(bundle: ContextBundle): void │
│ + RecordCacheOutcome(result:              │
│     CacheLookupResult): void               │
│ + StartSpan(name: string): IDisposable      │
└────────────────────────────────────┘
```

**Invariantes de persistência/eventos (FR-009, FR-011, FR-013):**

1. `SaveAsync` e `PublishAsync` DEVEM ocorrer em uma única transação
   *outbox*: se a escrita em PostgreSQL falhar, o evento NÃO DEVE ser
   publicado; se a publicação falhar após *commit*, o *outbox* DEVE reter
   o evento para republicação com *backoff* (*at-least-once*, FM-07).
2. `OffloadIfLargeAsync` DEVE mover para MinIO qualquer `content_inline`
   cujo tamanho exceda `context.limits.max_inline_bytes` (default 16 KiB),
   substituindo por `content_ref` — NÃO DEVE manter as duas formas
   simultaneamente para o mesmo fragmento.
3. `IdempotencyStore.TryBeginAsync` com `requestHash` divergente para a
   mesma `idemKey` DEVE falhar com `AIOS-CTX-0012` (conflito de
   idempotência) — nunca reexecuta o pipeline nesse caso.
4. `TelemetryEmitter.RecordAssembly` DEVE calcular e expor
   `compression_ratio = 1 − tokens_in/tokens_raw` (Context Compression
   Ratio) para todo bundle `SERVED`, mesmo em caminhos de `HIT`.

---

## 9. Entidades de Dados Canônicas (Contratos de Mensagem)

```
┌──────────────────────────────────────────────┐
│      «entity» ContextBundle                    │
├──────────────────────────────────────────────┤
│ + BundleId: Ulid                «PK»            │
│ + TenantId: string               «RLS»           │
│ + AgentUrn: Urn                                  │
│ + TaskUrn: Urn?                                   │
│ + ModelId: string                                 │
│ + ModelMaxTokens: int                              │
│ + TokenBudget: int                                  │
│ + ReservedOutputTokens: int                          │
│ + TokensIn: int                                       │
│ + TokensRaw: int                                       │
│ + CompressionRatio: decimal(5,4)                        │
│ + Status: BundleStatus (enum)                            │
│ + CacheOutcome: CacheOutcome?                              │
│ + IdempotencyKey: string          «UQ(tenant,key)»           │
│ + BudgetProfileId: Ulid            «FK→BudgetProfile»          │
│ + TraceId: string                                                │
│ + CreatedAt: DateTimeOffset                                       │
│ + ExpiresAt: DateTimeOffset                                        │
│ + DataSchema: string                                                 │
└──────────────────────────────────────────────┘

┌──────────────────────────────────────────────┐
│      «entity» ContextFragment                  │
├──────────────────────────────────────────────┤
│ + FragmentId: Ulid              «PK»            │
│ + TenantId: string                «RLS»           │
│ + BundleId: Ulid                  «FK→ContextBundle»│
│ + SourceKind: FragmentSourceKind (enum)              │
│ + SourceUrn: Urn?                                     │
│ + Role: FragmentRole (enum)                            │
│ + Ordinal: int                                          │
│ + TokensRaw: int                                          │
│ + TokensFinal: int                                          │
│ + RelevanceScore: decimal(6,5)                                │
│ + CompressionMethod: CompressionMethod (enum)                   │
│ + Included: bool                                                  │
│ + Embedding: float[1536]           «pgvector»                       │
│ + ContentInline: string?                                              │
│ + ContentRef: string?                                                   │
│ + CreatedAt: DateTimeOffset                                                │
└──────────────────────────────────────────────┘

┌──────────────────────────────────────────────┐
│      «entity» SemanticCacheEntry               │
├──────────────────────────────────────────────┤
│ + CacheId: Ulid                  «PK»            │
│ + TenantId: string                 «RLS»           │
│ + CacheScope: CacheScope (enum)                     │
│ + ScopeRef: string?                                   │
│ + ModelId: string                                       │
│ + PromptHash: string                «UQ(tenant,model,hash)»│
│ + PromptEmbedding: float[1536]        «pgvector HNSW»      │
│ + SimilarityThreshold: decimal(4,3)                          │
│ + ResponseRef: string                                          │
│ + TokensSaved: int                                                │
│ + CostSavedUsd: decimal(12,6)                                       │
│ + HitCount: long                                                      │
│ + TtlSeconds: int                                                       │
│ + CreatedAt: DateTimeOffset                                                │
│ + LastHitAt: DateTimeOffset?                                                 │
│ + InvalidatedAt: DateTimeOffset?                                                │
└──────────────────────────────────────────────┘

┌──────────────────────────────────────────────┐
│      «entity» SummaryNode                       │
├──────────────────────────────────────────────┤
│ + NodeId: Ulid                    «PK»            │
│ + TenantId: string                  «RLS»           │
│ + SourceUrn: Urn                                       │
│ + Level: int                                             │
│ + ParentId: Ulid?                    «FK self»              │
│ + Tokens: int                                                  │
│ + ContentRef: string                                              │
│ + Embedding: float[1536]              «pgvector»                    │
│ + ExpiresAt: DateTimeOffset                                            │
└──────────────────────────────────────────────┘

BundleStatus = RECEIVED | POLICY_CHECK | CACHE_LOOKUP | BUDGET_ALLOCATED
             | RETRIEVING | RANKING | COMPRESSING | ASSEMBLED | SERVED
             | REJECTED | DEGRADED | FAILED | EXPIRED
FragmentSourceKind = SYSTEM | INSTRUCTION | MEMORY | KNOWLEDGE | HISTORY | TOOL_RESULT
FragmentRole       = SYSTEM | USER | ASSISTANT | TOOL
CompressionMethod  = NONE | EXTRACTIVE | ABSTRACTIVE | TRUNCATE | DEDUP
```

`BudgetProfile` ◇── `ContextBundle` ◆── `ContextFragment` (agregação 1→N: um
perfil se aplica a muitos bundles; um bundle compõe muitos fragmentos, cujo
ciclo de vida é acoplado ao bundle — apagar o bundle apaga seus fragmentos).
`ContextBundle` ──▶ `SemanticCacheEntry` (associação por `cache_outcome`,
ciclo de vida independente). `SummaryNode` ◆── `SummaryNode` (árvore
self-referenciada da sumarização hierárquica). Ver `Database.md` para DDL
completo e `StateMachine.md` para o mapeamento status↔transições.

---

## 10. Invariantes Globais

| # | Invariante | Onde é garantida |
|---|-----------|---------------------|
| INV-01 | Todo registro (`ContextBundle`, `ContextFragment`, `SemanticCacheEntry`, `BudgetProfile`, `SummaryNode`) carrega `tenant_id` válido; isolamento por RLS + namespace de cache. | `ContextApiGateway`, PostgreSQL RLS, Redis keyspace prefixado por tenant. |
| INV-02 | `agent_urn`/`task_urn`/`source_urn` DEVEM ser URNs válidas (`urn:aios:<tenant>:<tipo>:<id>`, RFC-0001 §5.1). | Validação em `ContextApiGateway`. |
| INV-03 | Um `ContextBundle` no estado `SERVED` DEVE satisfazer `tokens_in ≤ token_budget`. | `ContextAssembler.CompressToFitAsync`. |
| INV-04 | `BudgetProfile`: soma dos pesos de alocação DEVE ser 1.0 (± tolerância). | `TokenBudgeter.ValidateAllocation`. |
| INV-05 | `ICompressionStrategy` é a única via de redução de tokens de fragmentos — nenhum componente comprime fora dela. | `ContextAssembler`. |
| INV-06 | `SemanticCacheEntry.HIT` só ocorre com `similarity ≥ similarity_threshold` ou `prompt_hash` exato. | `SemanticCacheManager`. |
| INV-07 | `ContextFragment.Embedding` e `SemanticCacheEntry.PromptEmbedding` têm dimensão fixa (`context.embedding.dimensions`, default 1536) por instalação. | `EmbeddingClient`, schema PostgreSQL. |
| INV-08 | Nenhum componente do pipeline persiste conteúdo sensível em claro em campos de evento/erro — apenas `content_ref`/identificadores. | `EventPublisher`, `TelemetryEmitter` (RFC-0001 §6). |
| INV-09 | `ContextBundle` é efêmero: todo bundle carrega `expires_at`; nenhum bundle é retido como memória canônica. | `BundleStore`, brief §1.3 (fronteira de compressão vs. memória). |
| INV-10 | Toda mutação externa (`Assemble`, `CacheStore`, `UpsertBudget`) é idempotente por `Idempotency-Key`. | `ContextApiGateway` + `IdempotencyStore`. |

---

## 11. Relações entre Componentes (Visão Consolidada)

```
ContextApiGateway ◆── IdempotencyStore
ContextApiGateway ──▶ ContextPolicyGuard
ContextApiGateway ──▶ ContextAssembler

ContextPolicyGuard ◇── PolicyClient

ContextAssembler ◇── SemanticCacheManager
ContextAssembler ◇── TokenBudgeter
ContextAssembler ◇── SelectiveRetriever
ContextAssembler ◇── RelevanceRanker
ContextAssembler ◇── RedundancyEliminator
ContextAssembler ◆── ICompressionStrategy
HierarchicalCompressor ┈┈▷ ICompressionStrategy
ExtractiveCompressor   ┈┈▷ ICompressionStrategy
TruncateCompressor     ┈┈▷ ICompressionStrategy
ContextAssembler ◆── BundleStore
ContextAssembler ◇── TelemetryEmitter

TokenBudgeter ◇── ModelRouterClient
TokenBudgeter ◇── TokenCounter
TokenCounter ◇── ModelRouterClient

SelectiveRetriever ◇── MemoryClient
RelevanceRanker ◇── EmbeddingClient
RedundancyEliminator ◇── EmbeddingClient
HierarchicalCompressor ◇── ModelRouterClient

SemanticCacheManager ◇── EmbeddingClient
SemanticCacheManager ◇── EvictionManager
SemanticCacheManager ◆── "Redis (L1)"
SemanticCacheManager ◆── "PostgreSQL+pgvector (L2)"

BundleStore ◇── "MinIO (blob offload)"
BundleStore ◆── EventPublisher
EventPublisher ──▶ "NATS/JetStream"

TelemetryEmitter - - ▷ (todos os componentes, instrumentação transversal)
```

Legenda de leitura: `◆──` indica que o ciclo de vida do componente à direita é
gerenciado pelo componente à esquerda (composição); `◇──` indica dependência
injetada (agregação, ciclo de vida independente, tipicamente via DI container
do .NET). Nenhuma seta cruza a fronteira do módulo sem passar por um dos
*clients* de §7 — reforçando a fronteira descrita em `Architecture.md` §6.
