---
Documento: SequenceDiagrams
Módulo: 010-Memory
Status: Draft
Versão: 0.1
Última atualização: 2026-07-20
Responsável (RACI-A): Arquiteto do Módulo 010-Memory
ADRs relacionados: ADR-0100, ADR-0101, ADR-0102, ADR-0103, ADR-0104, ADR-0106, ADR-0107, ADR-0109
RFCs relacionados: RFC-0001, RFC-0100
Depende de: 001-Architecture, 003-RFC (RFC-0001), 011-Context, 017-Model-Router, 021-Security, 022-Policy, 023-Learning, 024-Observability, 025-Audit, 026-Cost-Optimizer, 040-Glossary
---

# Diagramas de Sequência — Módulo 010-Memory

> Deriva das APIs (§5) e eventos (§6) do
> [`_DESIGN_BRIEF.md`](./_DESIGN_BRIEF.md). Contratos de correlação (`traceparent`,
> `X-AIOS-Tenant`, `Idempotency-Key`) e envelopes seguem
> [RFC-0001](../003-RFC/RFC-0001-Architecture-Baseline.md) §5.4–§5.6 — **reutilizados,
> não redefinidos**. Todos os diagramas são **ASCII**. Convenção: `──▶` chamada
> síncrona; `- ->` resposta/retorno; `~~▶` mensagem assíncrona (NATS/JetStream);
> `[guarda]` condição; `⌛` timeout. Palavras normativas RFC 2119/8174.

## 1. Participantes canônicos

| Alias | Componente | Papel |
|-------|-----------|-------|
| RT | AgentRuntime / ContextService (011) | Chamador da API |
| FAC | MemoryApiFacade | Fachada REST/gRPC |
| PEP | MemoryPep | PEP + tenant guard + idempotência |
| PDP | Policy Engine (022) | Autorização |
| IDS | IdempotencyStore (Redis) | Deduplicação |
| LR | LayerRouter | Classificação/roteamento |
| QM | QuotaManager | Cotas/backpressure |
| EMB | EmbeddingClient → Model Router (017) | Embeddings |
| LS | LayerStores (Redis/PG/pgvector/AGE) | Persistência por camada |
| BLB | BlobStoreAdapter (MinIO) | Blobs grandes |
| RE | RecallEngine | Recall híbrido + RRF |
| CE | ConsolidationEngine | Consolidação |
| CVM | ConsolidationVersionManager | Snapshot/rollback |
| FE | ForgettingEngine | Decay/expiry/purge |
| RS | RetentionScheduler | Agendamento de jobs |
| OBX | OutboxPublisher → NATS/JetStream | Eventos de domínio |
| TEL | MemoryTelemetry | OTel/métricas |

---

## 2. `remember` — caminho feliz (com embedding e blob)

Fluxo de UC-001. Timeout alvo: p99 ≤ 120 ms com embedding (NFR-001).

```
RT        FAC       PEP      PDP/IDS     LR        QM       EMB(017)   LS        BLB      OBX/NATS
│  POST    │         │          │        │         │         │         │         │          │
│ /items   │         │          │        │         │         │         │         │          │
│ +Idem-Key│         │          │        │         │         │         │         │          │
├─────────▶│         │          │        │         │         │         │         │          │
│          │ authz + │          │        │         │         │         │         │          │
│          │ idemp.  │          │        │         │         │         │         │          │
│          ├────────▶│ check key│        │         │         │         │         │          │
│          │         ├─────────▶│(Redis) │         │         │         │         │          │
│          │         │◀ - - miss│        │         │         │         │         │          │
│          │         ├─────────▶│ decide │         │         │         │         │          │
│          │         │◀ - allow-│        │         │         │         │         │          │
│          │◀ - ok - │          │        │         │         │         │         │          │
│          │ route   │          │        │         │         │         │         │          │
│          ├───────────────────────────▶│         │         │         │         │          │
│          │         │          │        │ reserve │         │         │         │          │
│          │         │          │        ├────────▶│[cota ok]│         │         │          │
│          │         │          │        │◀ - ok - │         │         │         │          │
│          │         │          │        │ [bytes>inline_max]│         │         │          │
│          │         │          │        ├──────────────────────────────────────▶│ put(hash)│
│          │         │          │        │◀ - - - - - - - - - - - - - - content_ref │        │
│          │         │          │        │ embed(content)   │         │         │          │
│          │         │          │        ├──────────────────▶│         │         │          │
│          │         │          │        │◀ - - vector(D) - -│         │         │          │
│          │         │          │        │ persist(item, embedding, content_ref) │          │
│          │         │          │        ├────────────────────────────▶│         │          │
│          │         │          │        │        │  INGESTED→ACTIVE   │         │          │
│          │         │          │        │◀ - - - - - - - - - - - ok - -│         │          │
│          │         │          │        │ outbox row + publish          │        │          │
│          │         │          │        ├──────────────────────────────────────────────────▶│
│          │         │          │        │        │         │  ~~▶ aios.<t>.memory.item.stored │
│          │◀ - - - - - - - - - 201 {urn,state=ACTIVE} - - - │         │         │          │
│◀ 201 - - │         │          │        │         │         │         │         │          │
```

Notas:
- Idempotência (RFC-0001 §5.5): repetição da mesma `Idempotency-Key` com payload igual retorna o resultado memoizado sem re-persistir.
- `TEL` registra `aios_memory_remember_duration_ms` em todo o span (omitido no traço por clareza).

---

## 3. `remember` — caminho de falha (Model Router indisponível → INGESTED)

Modo de falha F3 (brief §9): item persistido sem embedding, gerado em backlog.

```
RT        FAC       PEP       LR        EMB(017)      LS         RS/backlog
│ POST     │         │         │         │             │             │
├─────────▶│ authz ok│         │         │             │             │
│          ├────────▶│ allow   │         │             │             │
│          ├───────────────────▶│ embed  │             │             │
│          │         │         ├────────▶│ ⌛ timeout / 5xx           │
│          │         │         │◀ - err - │ AIOS-MEM-0030             │
│          │         │         │ [degrada: persiste sem embedding]    │
│          │         │         ├──────────────────────▶│ state=INGESTED
│          │         │         │◀ - - - - - - - - ok - -│             │
│          │         │         ├──────────────────────────────────────▶│ enfileira
│          │         │         │        embedding pendente (retry backoff+jitter)
│          │◀ - - - - 201 {urn,state=INGESTED} - - - - -│             │
│◀ 201 - - │         │         │         │             │             │
   ...                                                                │
   (backlog gera embedding e promove INGESTED→ACTIVE, emite item.stored)
```

Se o item excede N retries idempotentes, transita para `FAILED` (terminal recuperável;
retry posterior via `FAILED→INGESTED` com backoff — ver [`StateMachine.md`](./StateMachine.md) §2).

---

## 4. `recall` — caminho feliz (híbrido: vetorial + lexical + grafo + RRF)

Fluxo de UC-002. Timeout alvo por `mode` (NFR-003): p99 ≤ 50 ms (hot) / 150 ms (ANN) / 250 ms (grafo).

```
RT        FAC       PEP       RE         EMB(017)   LS(pgvector) LS(lexical) KGA(AGE)   TEL
│ POST     │         │         │          │           │            │          │         │
│ /recall  │         │         │          │           │            │          │         │
├─────────▶│ authz   │         │          │           │            │          │         │
│          ├────────▶│ allow+  │          │           │            │          │         │
│          │         │ tenant  │          │           │            │          │         │
│          │◀ - ok - │(RLS)    │          │           │            │          │         │
│          ├───────────────────▶│ plan(mode=hybrid)   │            │          │         │
│          │         │         │ embed(query)          │            │          │         │
│          │         │         ├─────────▶│           │            │          │         │
│          │         │         │◀ - qvec -│           │            │          │         │
│          │         │         │ ANN search (HNSW, ef_search)       │          │         │
│          │         │         ├──────────────────────▶│           │          │         │
│          │         │         │◀ - - - - - top-N vec - │           │          │         │
│          │         │         │ lexical search         │           │          │         │
│          │         │         ├───────────────────────────────────▶│         │          │
│          │         │         │◀ - - - - - - - top-N lex - - - - - -│         │          │
│          │         │         │ graph traverse (openCypher)         │         │          │
│          │         │         ├─────────────────────────────────────────────▶│         │
│          │         │         │◀ - - - - - - - - - - connected items - - - - -│         │
│          │         │         │ RRF fusion + rerank (relev/recência/saliência/escopo)   │
│          │         │         │ apply k, min_score                 │          │         │
│          │         │         ├────────────────────────────────────────────────────────▶│ recall_rate
│          │         │         │ ~~▶ aios.<t>.memory.item.recalled (amostrado)            │
│          │◀ - - - - RecallResult[] {items, scores} - -│           │          │         │
│◀ 200 - - │         │         │          │           │            │          │         │
```

Reidratação (FR-012): se um candidato está `ARCHIVED`, `RE` aciona `BLB` para reidratar
o conteúdo frio antes de compor o `RecallResult` (`ARCHIVED→ACTIVE`).

---

## 5. `recall` — caminho de falha (grafo indisponível → degradação graciosa)

Modo de falha F5 (brief §9): `mode=hybrid` degrada sem grafo; `mode=graph` falha.

```
RT        FAC       RE        LS(pgvector) KGA(AGE)
│ POST     │         │          │            │
│ /recall  │         │          │            │
│ mode=    │         │          │            │
│ hybrid   │         │          │            │
├─────────▶│─────────▶│ ANN + lexical ok     │
│          │         ├─────────▶│            │
│          │         │◀ - ok - -│            │
│          │         │ graph traverse         │
│          │         ├──────────────────────▶│ ⌛/erro
│          │         │◀ - AIOS-MEM-0031 - - - │
│          │         │ [degrada: fundir só ANN+lexical]
│          │◀ - - - - 200 RecallResult[] (sem grafo, flag degraded=true)
│◀ 200 - - │         │          │            │
```

```
RT        FAC       RE        KGA(AGE)
│ POST /recall mode=graph │     │
├─────────▶│─────────▶│ traverse│
│          │         ├────────▶│ erro
│          │         │◀ - err - │
│          │◀ - - - - 503 AIOS-MEM-0031 (retriable)
│◀ 503 - - │         │          │
```

---

## 6. `consolidate` — caminho feliz (dirigido por 023, versionado)

Fluxo de UC-003. Snapshot antes de mutar é invariante ([`StateMachine.md`](./StateMachine.md) §3).

```
023       NATS      RS        CE        CVM(PG/MinIO) LS        OBX/NATS       jobFSM
│ ~~▶ aios.<t>.learning.consolidation.requested       │         │             │
├─────────▶│─────────▶│ create ConsolidationJob        │         │             │ PENDING
│          │         │ admit + lock(tenant,agent)      │         │             │
│          │         ├────────▶│                        │         │             │
│          │         │         │ snapshot pré-imagem     │         │             │
│          │         │         ├────────▶│ write ConsolidationVersion            │ SNAPSHOTTING
│          │         │         │◀ - ver - │              │         │             │
│          │         │         │ promote/merge/dedup      │        │             │
│          │         │         ├─────────────────────────▶│ ACTIVE→CONSOLIDATING │ RUNNING
│          │         │         │ ~~▶ consolidation.started ───────▶│             │
│          │         │         │◀ - - - - - - - - - - ok - │        │             │
│          │         │         │ validate (Recall Rate pós ≥ pré−0,01)           │ VALIDATING
│          │         │         │ [ok] mark version active │        │             │
│          │         │         ├─────────────────────────▶│ CONSOLIDATING→CONSOLIDATED
│          │         │         │ ~~▶ item.consolidated + consolidation.completed ─▶│ COMMITTED
│◀ - - - - status via GetJob(job_id) - - - - - - - - - - -│        │             │
```

---

## 7. `consolidate` — caminho de falha (regressão de Recall Rate → rollback automático)

Modo de falha F6 (brief §9) e NFR-011.

```
RS        CE        CVM        LS        OBX/NATS       jobFSM
│─────────▶│ RUNNING  │          │         │             │ RUNNING
│          │ promote  │          │         │             │
│          ├─────────────────────▶│ mutações aplicadas   │
│          │ validate │          │         │             │ VALIDATING
│          │ [Recall Rate pós < pré−0,01]  │             │
│          │ rollback │          │         │             │ ROLLING_BACK
│          ├─────────▶│ restore versão anterior           │
│          │◀ - ok - -│          │         │             │
│          ├─────────────────────▶│ CONSOLIDATING→ACTIVE  │
│          │ ~~▶ consolidation.failed / consolidation.rolledback ─▶│ FAILED / ROLLED_BACK
```

Erros associados: `AIOS-MEM-0040` (conflito de versão concorrente), `AIOS-MEM-0041`
(falha de consolidação com rollback aplicado). Rollback manual explícito (UC-004):
`POST /consolidate/{jobId}/rollback`; versão inexistente/já ativa → `AIOS-MEM-0042`.

---

## 8. `forget` / decaimento — caminho feliz (TTL/decay → FORGOTTEN → PURGED)

Fluxo de UC-005. Grace period configurável por `memory.forget.grace_period`.

```
RS(cron)  FE        LS        QM        OBX/NATS       itemFSM
│ tick    │          │         │         │             │
├────────▶│ select elegíveis (decay_score<θ | expires_at | lru | quota)
│         ├─────────▶│ read candidates    │             │
│         │◀ - ids - │         │         │             │
│         │ create tombstone   │         │             │ DECAYING→FORGET_PENDING
│         ├─────────▶│ mark    │         │             │
│         │ confirm forget     │         │             │ FORGET_PENDING→FORGOTTEN
│         ├─────────▶│         │         │             │
│         │ ~~▶ aios.<t>.memory.item.forgotten ────────▶│
│         │ release quota      │         │             │
│         ├────────────────────▶│ dec used_items/bytes │
│         │  ... grace_period ⌛ ...       │             │
│         │ physical purge (vetor + blob + row)          │ FORGOTTEN→PURGED
│         ├─────────▶│ delete  │         │             │
│         │ ~~▶ aios.<t>.memory.item.purged ───────────▶│
```

---

## 9. `forget` — caminho de falha (item sob legal_hold)

Invariante §4.1: `legal_hold` NÃO DEVE ir a `PURGED` salvo RTBF autorizado.

```
RT/RS     FAC       FE        LS
│ POST /forget (strategy=ttl/decay/lru/quota) │
├────────▶│─────────▶│ select candidates      │
│         │         ├─────────▶│ item.retention_class=legal_hold
│         │         │◀ - hit - │
│         │         │ [bloqueia] AIOS-MEM-0050 (423)
│         │◀ - - - - 423 {skipped: [id], reason: legal_hold}
│◀ 423 - -│         │          │
```

RTBF autorizado (UC-006) é a única via para purgar `legal_hold`:

```
021/025   NATS      FE        BLB(MinIO) LS        OBX/NATS       itemFSM
│ ~~▶ aios.<t>.security.rtbf.requested   │         │             │
├────────▶│────────▶│ authorize RTBF      │         │             │
│         │         │ tombstone           │         │             │ FORGET_PENDING→FORGOTTEN
│         │         ├────────────────────────────▶│ delete blob   │
│         │         ├─────────────────────▶│ delete vector+row    │ FORGOTTEN→PURGED
│         │         │ ~~▶ aios.<t>.memory.item.purged ────────────▶│ (audit 025)
│◀ - 202 AIOS-MEM-0051 (aceito, assíncrono) - - - - │             │
```

---

## 10. Evento consumido — término de agente (assíncrono)

Fluxo de UC-009. Consumidor DEVE deduplicar por `event.id` (RFC-0001 §5.2).

```
006/008   NATS(JetStream)   MemorySvc   WMS/STS(Redis)   OBX/NATS
│ ~~▶ aios.<t>.agent.lifecycle.terminated │             │
├────────▶│─────────────────▶│ dedup(event.id)           │
│         │                 ├────────────▶│ expire Working/Short-Term
│         │                 │◀ - - ok - - │             │
│         │                 │ ~~▶ aios.<t>.memory.item.forgotten ─▶│
│         │                 │ ack          │             │
```

Falha F1 (Redis indisponível): Working é reconstruível (NFR-008); a operação degrada
graciosamente e as camadas duráveis permanecem íntegras. Mensagem envenenada após N
falhas vai para a DLQ do JetStream (brief §9 F10).

---

## 11. Rastreabilidade

| Diagrama | UC | FR/NFR | Evento(s) | Erro(s) |
|----------|----|--------|-----------|---------|
| §2 remember feliz | UC-001 | FR-001/004/009/010; NFR-001 | `item.stored` | — |
| §3 remember falha | UC-001 | FR-010; NFR-001 | `item.stored` (backlog) | `AIOS-MEM-0030` |
| §4 recall feliz | UC-002 | FR-002/003/012/013; NFR-002/003 | `item.recalled` | — |
| §5 recall falha | UC-002 | FR-013; NFR-003 | — | `AIOS-MEM-0031` |
| §6 consolidate feliz | UC-003 | FR-005/014; NFR-011 | `consolidation.started/completed`, `item.consolidated` | — |
| §7 consolidate falha | UC-003/004 | FR-007; NFR-011 | `consolidation.failed/rolledback` | `AIOS-MEM-0040/0041/0042` |
| §8 forget feliz | UC-005 | FR-006; NFR-009 | `item.forgotten/purged` | — |
| §9 forget falha/RTBF | UC-005/006 | FR-006/011 | `item.purged` | `AIOS-MEM-0050/0051` |
| §10 término agente | UC-009 | FR-006/012; NFR-008 | `item.forgotten` | — |

> Ver também [`API.md`](./API.md), [`Events.md`](./Events.md),
> [`StateMachine.md`](./StateMachine.md) e [`FailureRecovery.md`](./FailureRecovery.md).
