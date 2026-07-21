---
Documento: ClassDiagrams
Módulo: 010-Memory
Status: Draft
Versão: 0.1
Última atualização: 2026-07-20
Responsável (RACI-A): Arquiteto do Módulo 010-Memory
ADRs relacionados: ADR-0100, ADR-0101, ADR-0102, ADR-0104, ADR-0105, ADR-0106, ADR-0107
RFCs relacionados: RFC-0001, RFC-0100
Depende de: 001-Architecture, 003-RFC (RFC-0001), 005-Database, 011-Context, 017-Model-Router, 022-Policy, 023-Learning, 026-Cost-Optimizer, 040-Glossary
---

# Diagramas de Classe / Estruturas — Módulo 010-Memory

> Deriva do modelo de dados (§3) e da decomposição em componentes (§2) do
> [`_DESIGN_BRIEF.md`](./_DESIGN_BRIEF.md). Tipos externos (URN, envelopes) seguem
> [RFC-0001](../003-RFC/RFC-0001-Architecture-Baseline.md) — **reutilizados, não
> redefinidos**. Todos os diagramas são **ASCII**. Componentes em `PascalCase`;
> palavras normativas RFC 2119/8174. A assinatura das interfaces é ilustrativa
> (pseudo-.NET/C#), não binária.

## 1. Visão geral de estrutura (composição do MemoryService)

Notação: `◆──` composição (o todo possui a parte); `◇──` agregação; `- ->`
dependência; `<|--` implementação de interface.

```
                         ┌───────────────────────┐
                         │    MemoryApiFacade     │  (fachada REST/gRPC)
                         │  remember/recall/      │
                         │  consolidate/forget/   │
                         │  getItem/getStats      │
                         └───────────┬───────────┘
                                     │ ◆
        ┌──────────────┬─────────────┼───────────────┬────────────────┐
        ▼              ▼             ▼                ▼                ▼
 ┌────────────┐ ┌────────────┐ ┌──────────┐  ┌────────────────┐ ┌────────────────┐
 │ MemoryPep  │ │ LayerRouter│ │RecallEng.│  │ConsolidationEng│ │ ForgettingEng. │
 └─────┬──────┘ └─────┬──────┘ └────┬─────┘  └───────┬────────┘ └───────┬────────┘
       │ - ->         │ ◆           │ - ->           │ ◆                │ - ->
       ▼              ▼             ▼                ▼                  ▼
 ┌────────────┐ ┌────────────┐ ┌──────────────┐ ┌──────────────────┐ ┌──────────────────┐
 │ PolicyEng. │ │QuotaManager│ │IKnowledge    │ │Consolidation     │ │RetentionScheduler│
 │ (022, ext) │ │            │ │GraphAdapter  │ │VersionManager    │ │                  │
 └────────────┘ └─────┬──────┘ └──────────────┘ └──────────────────┘ └──────────────────┘
 ┌────────────┐       │ - ->
 │Idempotency │       ▼
 │Store(Redis)│ ┌────────────┐   ┌────────────────┐   ┌──────────────┐   ┌───────────────┐
 └────────────┘ │EmbeddingCl.│   │ IMemoryLayer   │   │BlobStore     │   │OutboxPublisher│
                │ (→017)     │   │ Store (iface)  │   │Adapter(MinIO)│   │ (→NATS)       │
                └────────────┘   └───────┬────────┘   └──────────────┘   └───────────────┘
                                         │ <|--  (Strategy por camada)
   ┌──────────────┬──────────────┬───────┴──────┬──────────────┬──────────────┬───────────┐
   ▼              ▼              ▼              ▼              ▼              ▼           ▼
Working      ShortTerm       LongTerm       Semantic      Procedural     Episodic   Knowledge
MemoryStore  MemoryStore     MemoryStore    MemoryStore   MemoryStore    MemoryStore GraphAdapter
(Redis,TTL)  (Redis+PG)      (PG)           (PG+pgvector) (PG)           (PG+pgvec/  (Apache AGE)
                                            HNSW)                        part.tempo)
```

`MemoryTelemetry` é uma dependência transversal (OTel) usada por todos os componentes
e é omitida do diagrama por clareza.

---

## 2. Entidade central `MemoryItem` e entidades de domínio

```
┌───────────────────────────── MemoryItem ─────────────────────────────┐
│ + id: Ulid                     «PK, URN=urn:aios:<tenant>:memory:<id>»│
│ + tenantId: string             «NOT NULL, RLS»                        │
│ + agentId: Ulid?                                                      │
│ + sessionId: Ulid?                                                    │
│ + layer: MemoryLayer           «working|short_term|long_term|         │
│                                  semantic|procedural|episodic|kg»     │
│ + kind: MemoryKind             «fact|event|skill|observation|         │
│                                  reflection|summary|edge»             │
│ + state: MemoryState           «ver StateMachine.md §2»               │
│ + content: Json?               «inline se ≤ inline_max_bytes»         │
│ + contentRef: string?          «s3://... quando externalizado»        │
│ + contentHash: byte[]          «sha256, dedup content-addressed»      │
│ + embedding: Vector?           «vector(D), HNSW em semantic/episodic» │
│ + embeddingModel: string?                                            │
│ + salience: float [0..1]       «default 0.5»                          │
│ + decayScore: float [0..1]                                           │
│ + accessCount: long            «default 0»                            │
│ + lastAccessAt: DateTimeOffset?                                      │
│ + sourceUrn: string?           «URN de origem»                       │
│ + consolidationVersion: long?  «FK→ConsolidationVersion»             │
│ + parentIds: Ulid[]?           «linhagem de consolidação»            │
│ + legalBasis: LegalBasis       «consent|contract|                    │
│                                  legitimate_interest|legal_obligation»│
│ + retentionClass: RetentionClass «ephemeral|standard|extended|       │
│                                    legal_hold»                        │
│ + pii: bool                    «default false»                       │
│ + tags: string[]               «GIN»                                 │
│ + metadata: Json                                                     │
│ + expiresAt: DateTimeOffset?                                        │
│ + createdAt: DateTimeOffset                                         │
│ + updatedAt: DateTimeOffset                                         │
│ + tombstonedAt: DateTimeOffset?                                     │
├───────────────────────────────────────────────────────────────────┤
│ + transitionTo(next: MemoryState): void   «valida FSM §4.1»         │
│ + isPurgeable(): bool         «false se legal_hold salvo RTBF»      │
│ + isInline(): bool            «content != null && contentRef==null» │
└───────────────────────────────────────────────────────────────────┘
        │ n..1                              │ 0..1
        │ consolidationVersion              │ parentIds (self-assoc.)
        ▼                                   ▼
┌────────────────────────┐        ┌────────────────────┐
│ ConsolidationVersion   │        │  MemoryItem (origem)│
│ + versionId: long «PK» │        └────────────────────┘
│ + tenantId: string     │
│ + agentId: Ulid?       │        ┌────────────────────────┐
│ + snapshotRef: string  │◀──1..n─│  ConsolidationJob      │
│   «MinIO»              │        │ + id: Ulid «PK»        │
│ + createdAt: DateTime  │        │ + tenantId: string     │
│ + parentVersion: long? │        │ + agentId: Ulid?       │
│ + active: bool         │        │ + trigger: JobTrigger  │
└────────────────────────┘        │   «learning|scheduled| │
                                  │    manual|threshold»   │
┌────────────────────────┐        │ + fromLayer: MemoryLayer│
│ MemoryLayerConfig      │        │ + toLayer: MemoryLayer │
│ + tenantId «PK»        │        │ + state: JobState      │
│ + layer   «PK»         │        │   «ver StateMachine §3»│
│ + ttlDefault: Duration │        │ + versionId: long?     │
│ + maxItems: long       │        │ + startedAt/finishedAt │
│ + maxBytes: long       │        │ + stats: Json          │
│ + decayHalfLife:Duration│       └────────────────────────┘
│ + consolidationThresh. │
│ + hnswM: int           │        ┌────────────────────────┐
│ + hnswEf: int          │        │ ForgettingPolicy       │
└────────────────────────┘        │ + id: Ulid «PK»        │
                                  │ + tenantId             │
┌────────────────────────┐        │ + scope: PolicyScope   │
│ MemoryQuota            │        │   «tenant|agent|layer» │
│ + tenantId «PK comp.»  │        │ + strategy: ForgetStrat│
│ + agentId  «PK comp.»  │        │   «ttl|decay|lru|      │
│ + layer    «PK comp.»  │        │    quota|rtbf»         │
│ + limitItems/limitBytes│        │ + params: Json         │
│ + usedItems/usedBytes  │        │ + enabled: bool        │
│ + updatedAt            │        └────────────────────────┘
└────────────────────────┘

┌────────────────────────┐  ┌────────────────────────┐  ┌────────────────────────┐
│ KnowledgeEdge (AGE)    │  │ OutboxEvent            │  │ IdempotencyRecord      │
│ + srcId / dstId        │  │ + id: Ulid «PK»        │  │ + key: string «PK»     │
│ + relType: string      │  │ + tenantId             │  │ + tenantId             │
│ + tenantId             │  │ + subject: string      │  │ + operation: string    │
│ + weight: float        │  │ + payload: Json        │  │ + responseHash: byte[] │
│ + properties: Json     │  │ + status: pending|     │  │ + result: Json         │
└────────────────────────┘  │   published            │  │ + expiresAt (≥24h)     │
                            │ + createdAt            │  └────────────────────────┘
                            └────────────────────────┘

┌──────────── RecallQuery (transitório) ────────────┐  ┌──── RecallResult (transitório) ────┐
│ + query: string                                    │  │ + items: MemoryItem[]              │
│ + agentId: Ulid?                                    │  │ + scores: float[]                 │
│ + layers: MemoryLayer[]                             │  │ + fusion: rrf|weighted            │
│ + kinds: MemoryKind[]                               │  │ + degraded: bool  «grafo off»     │
│ + tags: string[]                                    │  │ + cursor: string?                 │
│ + k: int  «default memory.recall.default_k»         │  │ + recallRate: float               │
│ + minScore: float «default memory.recall.min_score» │  └───────────────────────────────────┘
│ + timeRange: (from,to)?                             │
│ + includeForgotten: bool «default false»            │
│ + mode: vector|lexical|graph|hybrid                 │
└────────────────────────────────────────────────────┘
```

---

## 3. Interfaces públicas e contratos

### 3.1 `IMemoryService` (superfície canônica — §5 do brief)

```
interface IMemoryService {
  // Idempotente por Idempotency-Key (RFC-0001 §5.5). Erros: 0001,0003,0004,0005,
  // 0010,0011,0020,0021,0030,0031.
  Task<RememberResult> Remember(RememberRequest req, CorrelationCtx ctx);

  // Somente leitura. Erros: 0001,0004,0005,0021,0030,0031.
  Task<RecallResult> Recall(RecallQuery query, CorrelationCtx ctx);

  // Dispara ConsolidationJob assíncrono; retorna job_id. Erros: 0004,0010,0040.
  Task<JobRef> Consolidate(ConsolidateRequest req, CorrelationCtx ctx);

  // Aplica ForgettingPolicy. Erros: 0004,0050 (legal_hold),0031.
  Task<ForgetResult> Forget(ForgetRequest req, CorrelationCtx ctx);

  Task<MemoryItem> GetItem(Ulid id, CorrelationCtx ctx);           // 0002,0004,0005
  Task<Accepted>   PurgeItem(Ulid id, CorrelationCtx ctx);          // 0002,0004,0051(202)
  Task<MemoryStats> GetStats(StatsRequest req, CorrelationCtx ctx);
  Task<JobState>   GetJob(Ulid jobId, CorrelationCtx ctx);
  Task<RollbackResult> RollbackConsolidation(Ulid jobId, CorrelationCtx ctx); // 0042
}
```

`CorrelationCtx` transporta `traceparent`, `X-AIOS-Tenant` e `Idempotency-Key`
(RFC-0001 §5.6) — **não redefine** esses contratos, apenas os carrega.

### 3.2 `IMemoryLayerStore` (Strategy por camada — §2, §3.3)

```
interface IMemoryLayerStore {
  MemoryLayer Layer { get; }
  Task<MemoryItem> Put(MemoryItem item, CorrelationCtx ctx);   // respeita RLS
  Task<MemoryItem?> Get(Ulid id, CorrelationCtx ctx);
  Task<IReadOnlyList<Candidate>> Search(RecallQuery q, CorrelationCtx ctx);
  Task Tombstone(Ulid id, CorrelationCtx ctx);
  Task Purge(Ulid id, CorrelationCtx ctx);                     // remove vetor/blob
  StoreCapabilities Capabilities { get; }  // {vector?, graph?, ttl?, durable?}
}
```

Implementações e capacidades:

| Implementação | Backend | vector | graph | ttl | durable |
|---------------|---------|:------:|:-----:|:---:|:-------:|
| `WorkingMemoryStore` | Redis | não | não | sim | não |
| `ShortTermMemoryStore` | Redis + PG | não | não | sim | sim |
| `LongTermMemoryStore` | PostgreSQL | não | não | não¹ | sim |
| `SemanticMemoryStore` | PG + pgvector (HNSW) | sim | não | não² | sim |
| `ProceduralMemoryStore` | PostgreSQL | não | não | não | sim |
| `EpisodicMemoryStore` | PG + pgvector (part. tempo) | sim | não | sim³ | sim |
| `KnowledgeGraphAdapter` | Apache AGE | não | sim | não | sim |

¹ TTL default 90d por config, sem expiração automática de campo. ² sem TTL (decay).
³ TTL default 365d por config.

### 3.3 Interfaces de apoio

```
interface IEmbeddingClient {              // → Model Router (017), nunca LLM direto
  Task<Vector> Embed(string content, CorrelationCtx ctx);   // batching + retry idemp.
  int Dim { get; }                        // = memory.embedding.dim
}

interface IQuotaManager {
  Task<Reservation> Reserve(QuotaKey k, long items, long bytes); // 0010/0011
  Task Release(QuotaKey k, long items, long bytes);
  Task<Usage> Usage(QuotaKey k);          // reporta a 026
}

interface IOutboxPublisher {              // padrão Outbox (DB→NATS at-least-once)
  Task Enqueue(OutboxEvent e, DbTx tx);   // mesma transação da mutação
}

interface IConsolidationVersionManager {
  Task<ConsolidationVersion> Snapshot(ConsolidateRequest req);   // antes de mutar
  Task<RollbackResult> Rollback(long versionId);                 // 0042
}
```

---

## 4. Invariantes de domínio (normativas)

| # | Invariante | Enforcement |
|---|------------|-------------|
| INV-1 | Todo `MemoryItem` DEVE ter `tenant_id` não nulo e é acessível apenas sob RLS do tenant. | RLS PostgreSQL; `MemoryPep`. |
| INV-2 | `content` XOR `content_ref`: conteúdo é inline **ou** externalizado, nunca ambos nulos para item com corpo. | `isInline()`; `LayerRouter`. |
| INV-3 | Se `content_ref != null`, então `content_hash` DEVE estar preenchido (content-addressed). | `BlobStoreAdapter`. |
| INV-4 | `embedding` só existe em camadas `semantic`/`episodic`; `dim(embedding)` DEVE == `memory.embedding.dim`, senão `AIOS-MEM-0021`. | `SemanticMemoryStore`/`EpisodicMemoryStore`. |
| INV-5 | `layer` DEVE ser compatível com `kind`, senão `AIOS-MEM-0020`. | `LayerRouter`. |
| INV-6 | Transição para `CONSOLIDATING` DEVE ter `ConsolidationVersion` gravada antes (§4.1 inv. ii). | `ConsolidationEngine` + `CVM`. |
| INV-7 | Item em `retention_class=legal_hold` NÃO DEVE ir a `PURGED` salvo RTBF autorizado (§4.1 inv. i). | `ForgettingEngine`; `isPurgeable()`. |
| INV-8 | `FORGOTTEN` mantém tombstone + `parent_ids` (linhagem) até `PURGED` (§4.1 inv. iii). | `ForgettingEngine`. |
| INV-9 | `salience`, `decay_score` ∈ [0..1]; `access_count` ≥ 0. | CHECK constraints (ver [`Database.md`](./Database.md)). |
| INV-10 | Toda mutação emite exatamente um `OutboxEvent` na mesma transação (atomicidade DB+evento). | `OutboxPublisher`. |
| INV-11 | `MemoryItem.transitionTo` só aceita transições declaradas na FSM (§4.1). | `transitionTo()`; ver [`StateMachine.md`](./StateMachine.md). |
| INV-12 | `used_items/used_bytes` de `MemoryQuota` NÃO DEVEM exceder os limites (backpressure a partir daí). | `QuotaManager`. |

---

## 5. Enumerações canônicas

```
enum MemoryLayer   { working, short_term, long_term, semantic, procedural, episodic, kg }
enum MemoryKind    { fact, event, skill, observation, reflection, summary, edge }
enum MemoryState   { INGESTED, ACTIVE, CONSOLIDATING, CONSOLIDATED, ARCHIVED,
                     DECAYING, FORGET_PENDING, FORGOTTEN, PURGED, FAILED }
enum JobState      { PENDING, REJECTED, SNAPSHOTTING, RUNNING, VALIDATING,
                     COMMITTED, ROLLING_BACK, ROLLED_BACK, FAILED }
enum JobTrigger    { learning, scheduled, manual, threshold }
enum LegalBasis    { consent, contract, legitimate_interest, legal_obligation }
enum RetentionClass{ ephemeral, standard, extended, legal_hold }
enum ForgetStrategy{ ttl, decay, lru, quota, rtbf }
enum PolicyScope   { tenant, agent, layer }
enum RecallMode    { vector, lexical, graph, hybrid }
```

> Estes valores são canônicos e idênticos aos usados em
> [`Database.md`](./Database.md) (tipos `enum`), [`StateMachine.md`](./StateMachine.md)
> e [`API.md`](./API.md). Qualquer divergência é defeito de documentação.
