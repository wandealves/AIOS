---
Documento: Architecture
Módulo: 010-Memory
Status: Draft
Versão: 0.1
Última atualização: 2026-07-20
Responsável (RACI-A): Arquiteto do Módulo 010-Memory
ADRs relacionados: ADR-0007 (herdada), ADR-0100, ADR-0101, ADR-0102, ADR-0103, ADR-0104, ADR-0105, ADR-0106, ADR-0107, ADR-0108, ADR-0109
RFCs relacionados: RFC-0001 (baseline), RFC-0100 (a propor)
Depende de: 001-Architecture, 003-RFC (RFC-0001), 005-Database, 011-Context, 017-Model-Router, 021-Security, 022-Policy, 023-Learning, 024-Observability, 025-Audit, 026-Cost-Optimizer, 040-Glossary
---

# 010-Memory — Arquitetura

> Este documento detalha a arquitetura interna do `MemoryService` (gerenciador de
> memória cognitiva hierárquica do AIOS) em notação C4 adaptada para ASCII. Ele
> **não redefine** contratos centrais (URN, envelope de evento, envelope de erro,
> idempotência, correlação, subjects) — estes são consumidos de
> `../003-RFC/RFC-0001-Architecture-Baseline.md`. Este documento é derivado de
> `./_DESIGN_BRIEF.md` (fonte única de verdade do módulo) e **NÃO PODE**
> contradizê-lo.

## Índice

1. Papel do Memory na arquitetura global
2. C4 Nível 1 — Contexto do Sistema (Memory)
3. C4 Nível 2 — Contêineres
4. C4 Nível 3 — Componentes
5. Tabela de responsabilidades por componente
6. Fronteiras e regras de dependência
7. Padrões arquiteturais adotados
8. Tecnologias e justificativas
9. Mapa canônico das 7 camadas físicas e travessia de planos
10. Riscos arquiteturais e trade-offs
11. Alternativas descartadas
12. Referências e ADRs

---

## 1. Papel do Memory na Arquitetura Global

O `MemoryService` (`010`) é o **gerenciador de memória cognitiva hierárquica** do
AIOS — análogo à MMU + swap + sistema de arquivos de um SO clássico, porém
aplicado a recursos cognitivos. Ele expõe uma **API uniforme de memória**
(`remember`, `recall`, `consolidate`, `forget`) sobre 7 camadas físicas
heterogêneas (Working, Short-Term, Long-Term, Semantic, Procedural, Episodic,
Knowledge Graph), ocultando do chamador a diversidade de backends (Redis,
PostgreSQL+pgvector, Apache AGE, MinIO).

O Memory é consultado pelo `006-Kernel` (via syscalls `remember`/`recall`
brokeradas), dirigido pelo `023-Learning` para consolidação e por políticas de
esquecimento próprias para conter crescimento descontrolado. Ele colabora com
o `011-Context` na recuperação seletiva que compõe a janela de contexto final —
mas **não monta** essa janela (fronteira N3 do brief). Ele **nunca** chama LLM
diretamente: todo embedding é requisitado ao `017-Model-Router` (fronteira R7/N1).

**Fronteira crítica:** o Agent Runtime (Python, plano de dados) **NÃO DEVE**
acessar PostgreSQL/Redis/MinIO diretamente para memória — **DEVE** passar pela
API do `MemoryService` via gRPC/NATS, brokerada pelo Kernel (regra de camadas de
`../001-Architecture/Architecture.md` §6; ver `./_DESIGN_BRIEF.md` "Fronteira
crítica").

---

## 2. C4 — Nível 1: Contexto do Sistema (Memory)

```
                gRPC (interno, brokerado pelo Kernel)         REST/gRPC (interno, via YARP)
    ┌──────────────────┐                              ┌──────────────────────┐
    │   006-Kernel      │───── remember/recall/... ───▶│                       │
    │ (broker de recurso│◀──── resultado/erro ─────────│                       │
    │  governado)       │                              │                       │
    └──────────────────┘                              │                       │
                                                         │                       │
    ┌──────────────────┐   recall dirigido/seletivo     │                       │  gRPC   ┌──────────────────┐
    │   011-Context      │◀──────────────────────────▶│    010-MEMORY          │────────▶│  022-Policy (PDP)  │
    │ (janela de contexto)│                             │  (memória cognitiva)   │◀────────│                    │
    └──────────────────┘                              │                       │         └──────────────────┘
                                                         │                       │
    ┌──────────────────┐   consolidate() dirigido       │                       │  embeddings     ┌──────────────────┐
    │   023-Learning      │──────────────────────────▶│                       │───────────────▶│ 017-Model-Router   │
    │ (política de        │◀── rollback/eventos ───────│                       │◀───────────────│                    │
    │  aprendizado)        │                             │                       │  vetores        └──────────────────┘
    └──────────────────┘                              │                       │
                                                         │                       │
    ┌──────────────────┐   eventos (async)              │                       │  telemetria/audit  ┌──────────────────┐
    │ NATS/JetStream     │◀───────────────────────────│                       │──────────────────▶│ 024-Obs./025-Audit │
    │ (020-Communication) │────────────────────────────▶│                       │                    └──────────────────┘
    └──────────────────┘                              └───────────────────────┘
                                                                  │
                                                                  │  reporta consumo
                                                                  ▼
                                                         ┌──────────────────┐
                                                         │ 026-Cost-Optimizer │
                                                         └──────────────────┘
```

**Atores e sistemas em fronteira**

| Ator/Sistema | Papel na interação com o Memory | Protocolo |
|--------------|----------------------------------|-----------|
| `006-Kernel` | Cliente principal: encaminha syscalls `remember`/`recall`/`consolidate`/`forget` brokeradas após capability+cota. | gRPC (interno) |
| `011-Context` | Consumidor de recuperação seletiva: emite `context.recall.requested`, recebe `RecallResult` para compor a janela (mas o Memory não monta a janela — N3). | gRPC/NATS |
| `023-Learning` | Autoridade que dirige consolidação (`learning.consolidation.requested`) e reversão (`learning.policy.rolledback`); o Memory apenas executa. | NATS (evento) |
| `022-Policy` (PDP) | Autoridade de decisão consultada pelo `MemoryPep` para toda operação privilegiada. | gRPC |
| `017-Model-Router` | Único caminho legítimo para obtenção de embeddings (R7/N1 — o Memory nunca chama LLM diretamente). | gRPC |
| NATS/JetStream (`020`) | Barramento de eventos de domínio (`item.stored`, `consolidated`, `forgotten`, ...) e de consumo (`agent.lifecycle.*`, `policy.decision.updated`, `security.rtbf.requested`). | NATS/JetStream |
| `024-Observability` / `025-Audit` | Destino de telemetria OTel (Memory Recall Rate incluída) e de registros de auditoria imutáveis. | OTLP / gRPC |
| `026-Cost-Optimizer` | Recebe relatório de consumo (itens/bytes/vetores) por tenant/agente/camada; não decide cobrança (N9). | NATS (evento) |
| API Gateway (YARP, `021`) | Front door externo para clientes autorizados (operação/SDK) além do caminho brokerado pelo Kernel. | REST/HTTPS |

---

## 3. C4 — Nível 2: Contêineres

> O `MemoryService` é publicado como **um único serviço de control plane**
> (stateless ao nível de processo, escalado horizontalmente), acompanhado dos
> armazenamentos físicos que materializam as 7 camadas.

```
┌──────────────────────────────────────────────────────────────────────────────────┐
│                          010-MEMORY — CONTÊINERES                                │
│                                                                                    │
│  ┌──────────────────────────────────────────────────────────────────────────┐    │
│  │        Memory Service  (.NET 10, AOT, stateless, N réplicas)             │    │
│  │  Expõe: REST /v1/memory (via YARP) · gRPC aios.memory.v1 (interno)       │    │
│  └───────┬───────────────┬────────────────┬───────────────┬─────────────────┘    │
│          │               │                │               │                       │
│   ┌──────▼──────┐ ┌──────▼───────────┐ ┌──▼─────────────┐ ┌▼──────────────────┐  │
│   │  Redis        │ │ PostgreSQL 16     │ │ Apache AGE      │ │  MinIO             │  │
│   │  (Working/    │ │ (+ pgvector HNSW) │ │ (extensão do    │ │ (blobs grandes,    │  │
│   │  Short-Term    │ │ schema `memory`;  │ │ mesmo cluster   │ │  snapshots de       │  │
│   │  hot state,    │ │ RLS por tenant_id;│ │ PostgreSQL;     │ │  ConsolidationVer-  │  │
│   │  quotas quentes,│ │ Long-Term/        │ │ grafo de        │ │  sion; content-      │  │
│   │  locks de        │ │ Semantic/Procedural│ │ memória do      │ │  addressed)         │  │
│   │  consolidação)   │ │ /Episodic; outbox) │ │ agente)         │ │                     │  │
│   └───────────────┘ └──────────────────┘ └────────────────┘ └────────────────────┘  │
│                                                                                    │
│   ┌────────────────────────────────────────────────────────────────────────┐     │
│   │  Sidecar de Telemetria: OTel Collector (traces/metrics) → 024;           │     │
│   │  Serilog → Seq (024/017-Logging); fora do caminho crítico.               │     │
│   └────────────────────────────────────────────────────────────────────────┘     │
└──────────────────────────────────────────────────────────────────────────────────┘
```

| Contêiner | Responsabilidade | Persistência | Escala |
|-----------|-------------------|---------------|--------|
| Memory Service | Hospeda todos os componentes internos (§4); expõe REST/gRPC; stateless. | Nenhuma local (tudo externalizado). | Horizontal por shard `hash(tenant_id, agent_id) mod N`; sem afinidade obrigatória. |
| Redis | Working/Short-Term (hot), contadores de cota quentes, locks distribuídos de consolidação por (tenant, agente), cache de resultado de recall. | Volátil (Working reconstruível; Short-Term com projeção durável em PG). | Cluster com réplicas. |
| PostgreSQL (+pgvector, schema `memory`) | Fonte da verdade de `MemoryItem`, `MemoryLayerConfig`, `ConsolidationJob`, `ConsolidationVersion`, `ForgettingPolicy`, `MemoryQuota`, `OutboxEvent`, `IdempotencyRecord`. RLS por `tenant_id`. Índice HNSW para Semantic/Episodic. | Durável, particionado (Episodic por tempo). | Réplicas de leitura (CQRS) + streaming replication (HA, `027`). |
| Apache AGE (extensão sobre o mesmo cluster PostgreSQL) | `KnowledgeEdge`/nós de memória consolidada do agente (openCypher); travessia para `mode=graph`. | Durável, junto ao cluster PG. | Escala com o cluster PostgreSQL; travessia isolada por *bulkhead*. |
| MinIO | Blobs grandes (`content_ref`), snapshots frios de `ConsolidationVersion`. | Durável (S3), content-addressed por hash. | Escala S3-compatível padrão do AIOS. |
| NATS/JetStream | Transporte durável de eventos de domínio publicados pelo Outbox relay. | Streams `MEMORY_ITEM`, `MEMORY_CONSOLIDATION`, `MEMORY_QUOTA`. | Cluster NATS (`020`). |

---

## 4. C4 — Nível 3: Componentes

> Reproduzido e detalhado a partir de `./_DESIGN_BRIEF.md` §2.1. Este é o
> diagrama de componentes autoritativo do Memory.

```
┌──────────────────────────────── MEMORY SERVICE (010) · .NET 10 ────────────────────────────────┐
│                                                                                                  │
│   REST/gRPC (YARP)                                                                               │
│        │                                                                                          │
│   ┌────▼─────────────┐   consulta PDP   ┌──────────────────┐                                     │
│   │ MemoryApiFacade  │─────────────────▶│ MemoryPep        │──▶ Policy Engine (022)              │
│   │ remember/recall/ │                  │ (PEP + tenant    │                                     │
│   │ consolidate/forget│                 │  guard + idemp.) │──▶ IdempotencyStore (Redis)         │
│   └────┬─────────────┘                  └──────────────────┘                                     │
│        │ comando validado                                                                        │
│   ┌────▼───────────┐        ┌────────────────────┐        ┌───────────────────────┐             │
│   │ LayerRouter    │───────▶│ QuotaManager       │───────▶│ EmbeddingClient       │──▶ Model    │
│   │ (classifica +  │        │ (cotas/tenant/     │        │ (gera embedding via   │   Router    │
│   │  roteia p/camada)│      │  camada, backpress)│        │  017; cache dim)      │   (017)     │
│   └────┬───────────┘        └────────────────────┘        └───────────────────────┘             │
│        │                                                                                          │
│   ┌────▼──────────────────────── LAYER STORES (Strategy por camada) ──────────────────────────┐ │
│   │ WorkingMemoryStore   ShortTermMemoryStore   LongTermMemoryStore   SemanticMemoryStore      │ │
│   │   (Redis, TTL)         (Redis+PG, sessão)     (PG, durável)         (PG+pgvector/HNSW)      │ │
│   │ ProceduralMemoryStore  EpisodicMemoryStore   KnowledgeGraphAdapter                          │ │
│   │   (PG, skills)          (PG+pgvector, temporal)  (Apache AGE / openCypher)                  │ │
│   └────┬───────────────────────┬───────────────────────┬───────────────────┬───────────────────┘ │
│        │                       │                       │                   │                      │
│   ┌────▼─────────┐   ┌─────────▼────────┐   ┌──────────▼─────────┐  ┌──────▼──────────┐          │
│   │ RecallEngine │   │ ConsolidationEngine│ │ ForgettingEngine   │  │ BlobStoreAdapter│          │
│   │ (hybrid +    │   │ (promoção versio-  │ │ (decay/expiry/     │  │ (MinIO; conteúdo│          │
│   │  rerank +    │   │  nada, dirigida    │ │  purge, cotas,     │  │  grande + hash) │          │
│   │  RRF fusion) │   │  por 023)          │ │  RTBF)             │  └─────────────────┘          │
│   └──────────────┘   └─────────┬──────────┘ └────────┬───────────┘                               │
│                                │ snapshot/rollback    │ tombstone                                 │
│                      ┌─────────▼──────────┐  ┌────────▼──────────┐   ┌────────────────────────┐  │
│                      │ ConsolidationVersion│ │ RetentionScheduler│   │ OutboxPublisher        │  │
│                      │ Manager (rollback)  │ │ (jobs/cron)       │   │ (DB→NATS at-least-once)│  │
│                      └─────────────────────┘ └───────────────────┘   └───────────┬────────────┘  │
│                                                                                   │               │
│   ┌───────────────────────────────────────────────────────────────────────┐      │               │
│   │ MemoryTelemetry (OTel: métricas/traces/logs · Memory Recall Rate)      │      │               │
│   └───────────────────────────────────────────────────────────────────────┘      ▼               │
└──────────────────────────────────────────────────────────────────────────── NATS (JetStream) ───┘
        │ Redis            │ PostgreSQL (+pgvector +AGE)          │ MinIO
        ▼                  ▼                                       ▼
   estado quente      fonte da verdade (RLS por tenant)      blobs/checkpoints
```

---

## 5. Tabela de Responsabilidades por Componente

| Componente | Responsabilidade primária | Colaboradores diretos | Dependências externas |
|------------|---------------------------|------------------------|-------------------------|
| **MemoryApiFacade** | Ponto único das 4 operações canônicas (`remember/recall/consolidate/forget`) + `getItem/getStats`; valida DTOs, aplica versionamento de API (`X-AIOS-Api-Version`), traduz para comandos internos. | MemoryPep, LayerRouter, RecallEngine, ConsolidationEngine, ForgettingEngine | API Gateway (YARP), `006-Kernel` |
| **MemoryPep** | Policy Enforcement Point: autoriza cada operação junto ao PDP (`022`), impõe `tenant` do contexto autenticado (default deny se divergente — `AIOS-MEM-0005`), valida `Idempotency-Key`. | Policy Engine (022), IdempotencyStore, `021-Security` | `022-Policy` |
| **IdempotencyStore** | Persiste resultado por `Idempotency-Key` por ≥ 24h (RFC-0001 §5.5); deduplica mutações. | Redis (quente), PostgreSQL (durável) | — |
| **LayerRouter** | Classifica o item (por `kind`/hint) e roteia para a(s) camada(s) corretas; decide inline vs. blob; aciona QuotaManager e EmbeddingClient conforme a camada. | QuotaManager, EmbeddingClient, LayerStores, BlobStoreAdapter | — |
| **QuotaManager** | Contabiliza itens/bytes/vetores por (tenant, agente, camada); aplica cotas e *backpressure*; reporta consumo ao `026`. | Redis (contadores quentes), PostgreSQL, Cost-Optimizer (026) | `026-Cost-Optimizer` |
| **EmbeddingClient** | Solicita embeddings ao Model Router (`017`) com *batching*, retry idempotente, cache de dimensão; nunca chama LLM diretamente (N1). | Model Router (017) | `017-Model-Router` |
| **WorkingMemoryStore** | Memória de trabalho volátil do raciocínio corrente; Redis com TTL curto; escopo por agente/sessão. | Redis | — |
| **ShortTermMemoryStore** | Memória recente de sessão; Redis (hot) com projeção durável em PostgreSQL (CQRS). | Redis, PostgreSQL | — |
| **LongTermMemoryStore** | Memória persistente consolidada; PostgreSQL durável. | PostgreSQL | — |
| **SemanticMemoryStore** | Fatos/conceitos gerais; PostgreSQL + `pgvector` (índice HNSW) para busca ANN. | PostgreSQL+pgvector | — |
| **ProceduralMemoryStore** | Habilidades/rotinas "como fazer"; PostgreSQL (estrutura + versão de skill). | PostgreSQL | — |
| **EpisodicMemoryStore** | Eventos/experiências com contexto temporal; PostgreSQL + `pgvector`; particionado por tempo. | PostgreSQL+pgvector | — |
| **KnowledgeGraphAdapter** | Nós/arestas de memória consolidada do agente em Apache AGE (openCypher); travessia para recall. | PostgreSQL+Apache AGE | — |
| **BlobStoreAdapter** | Externaliza conteúdo > limite inline em MinIO; content-addressed (hash), deduplicação, TTL alinhado à retenção. | MinIO | — |
| **RecallEngine** | Recuperação híbrida (lexical + ANN + grafo), fusão RRF, re-rank por relevância/recência/saliência/escopo; produz `RecallResult` rankeado; calcula Memory Recall Rate. | LayerStores, KnowledgeGraphAdapter, EmbeddingClient, `011-Context` | `011-Context` |
| **ConsolidationEngine** | Promove/mescla/deduplica itens entre camadas sob direção do `023`; grava snapshot versionado antes de mutar. | ConsolidationVersionManager, LayerStores, `023-Learning`, OutboxPublisher | `023-Learning` |
| **ConsolidationVersionManager** | Versiona cada consolidação; permite `rollback` a uma versão anterior (mitiga *catastrophic forgetting*). | PostgreSQL, MinIO (snapshots frios) | — |
| **ForgettingEngine** | Aplica decaimento (score), expiração (TTL), expurgo (RTBF) e poda por cota; cria tombstones. | LayerStores, QuotaManager, RetentionScheduler, OutboxPublisher | — |
| **RetentionScheduler** | Agenda jobs de decaimento/consolidação/expurgo (cron interno + reação a eventos); garante idempotência de jobs. | ForgettingEngine, ConsolidationEngine, NATS | `020-Communication` |
| **OutboxPublisher** | Publica eventos de domínio via padrão Outbox (linha DB + publicação NATS at-least-once). | PostgreSQL (outbox), NATS/JetStream | `020-Communication` |
| **MemoryTelemetry** | Instrumenta OTel (traces/metrics/logs Serilog→Seq); expõe `aios_memory_*` e Memory Recall Rate. | OpenTelemetry, `024-Observability` | `024-Observability`, `025-Audit` |

---

## 6. Fronteiras e Regras de Dependência

- O Memory **DEVE** tratar `017-Model-Router` como **serviço externo
  consultado**, nunca como biblioteca embutida; **NÃO DEVE**, em nenhuma
  circunstância, chamar um LLM diretamente (N1, R7).
- O Memory **NÃO DEVE** decidir **quando/como** aprender uma política de
  consolidação; **DEVE** apenas executar o `ConsolidationJob` solicitado pelo
  `023-Learning` (N2) — a inteligência de "o que promover" pertence ao 023, a
  mecânica de "como promover com segurança" (versionamento, rollback,
  integridade) pertence ao Memory.
- O Memory **NÃO DEVE** montar a janela de contexto final nem realizar
  compressão de prompt/cache semântico de inferência (N3) — apenas fornece
  `RecallResult` rankeado ao `011-Context`, que é quem compõe a janela.
- O Memory **NÃO DEVE** persistir o grafo de conhecimento de domínio público
  nem o GraphRAG de recuperação global (N4) — o `KnowledgeGraphAdapter`
  gerencia apenas o **grafo de memória do agente** (nós/arestas privados por
  tenant/agente), distinto do grafo global de `018/019-Knowledge`.
- O Memory **NÃO DEVE** decidir autorização (PDP); atua exclusivamente como
  PEP consultando `022-Policy` via `MemoryPep` (N5, *default deny*).
- O Memory **NÃO DEVE** manter trilha de auditoria imutável própria; apenas
  emite eventos de domínio que o `025-Audit` consome e torna imutáveis (N6).
- Toda dependência externa do Memory (`017`, `022`, `023`, `026`) **DEVE**
  estar isolada por *circuit breaker* e *bulkhead* dedicados, para que a falha
  de uma dependência (ex.: `017` indisponível) não degrade outras operações
  (ex.: `recall` sobre itens já persistidos).
- O Memory **DEVE** ser *stateless* ao nível de processo: todo estado que
  sobrevive a um restart de réplica **DEVE** residir em Redis/PostgreSQL/AGE/
  MinIO/NATS, nunca em memória de processo não externalizada.

---

## 7. Padrões Arquiteturais Adotados

| Padrão | Onde é aplicado | Motivação |
|--------|-------------------|-----------|
| **PEP/PDP (Policy Enforcement/Decision Point)** | `MemoryPep` (PEP) consulta `022-Policy` (PDP). | Separação entre aplicação e decisão de política (ADR-0008 herdada); *default deny* uniforme (§12.1 do brief). |
| **Strategy** | `LayerStores` (`WorkingMemoryStore`, `ShortTermMemoryStore`, ..., `KnowledgeGraphAdapter`) implementam uma interface comum por camada, cada uma com backend/política físicos próprios. | Permite adicionar/trocar backend de uma camada sem alterar `LayerRouter`/`RecallEngine` (ADR-0100). |
| **CQRS (Command Query Responsibility Segregation)** | Escrita via `MemoryApiFacade`→`LayerRouter` (caminho de comando); leitura via `RecallEngine` sobre réplicas de leitura PG + cache Redis. | Latência de leitura menor sem sacrificar consistência de escrita nas camadas duráveis (glossário — CQRS; brief §10). |
| **Event Sourcing parcial (linhagem)** | `MemoryItem.parent_ids` e `ConsolidationVersion` mantêm a linhagem de cada consolidação/merge. | Rastreabilidade de proveniência (glossário — Provenance) e suporte a rollback determinístico (ADR-0102). |
| **Transactional Outbox** | `OutboxPublisher` grava o evento na mesma transação do estado e um relay publica no JetStream. | Atomicidade entre persistência e publicação sem *dual write* (R9). |
| **Versionamento + Rollback (Snapshot/Restore)** | `ConsolidationVersionManager` grava snapshot antes de toda `CONSOLIDATING`; `RollbackConsolidation` restaura versão anterior. | Defesa primária contra *catastrophic forgetting* (R4, NFR-011, ADR-0102). |
| **Hybrid Search + Reciprocal Rank Fusion (RRF)** | `RecallEngine` funde resultados lexical + vetorial (pgvector/HNSW) + grafo (AGE) por RRF (ou `weighted`, configurável). | Qualidade de recuperação superior a um único modo isolado; base do Memory Recall Rate (NFR-002, ADR-0104). |
| **Content-Addressed Storage** | `BlobStoreAdapter` usa `content_hash` (sha256) como chave de deduplicação em MinIO. | Evita duplicação de blobs grandes; simplifica expurgo determinístico (ADR-0106). |
| **Backpressure + Quota Accounting** | `QuotaManager` nega/represa escrita ao exceder cota por (tenant, agente, camada). | Contém crescimento descontrolado sem degradar tenants vizinhos (R6, R5, NFR-009, ADR-0105). |
| **Idempotency-Key + Deduplicação de Eventos** | `IdempotencyStore` em toda mutação; consumidores de eventos deduplicam por `event.id`. | Efeito *exactly-once* sobre transporte *at-least-once* (RFC-0001 §5.5). |
| **Sharding determinístico** | Camadas vetoriais e Episodic particionadas por `shard = hash(tenant_id, agent_id) mod N`; Episodic também particionado por tempo (`RANGE` mensal). | Localidade de índice HNSW e distribuição de carga previsível rumo a 10⁸ vetores/tenant (NFR-012, ADR-0108). |
| **Lock distribuído por escopo** | `ConsolidationEngine` serializa jobs concorrentes por (tenant, agente) via lock Redis. | Evita consolidações concorrentes conflitantes sobre o mesmo escopo (brief §10). |

---

## 8. Tecnologias e Justificativas

| Tecnologia | Uso no Memory | Justificativa | Alternativa descartada |
|------------|-----------------|----------------|--------------------------|
| **.NET 10 (AOT)** | Runtime do Memory Service. | Baixa latência de start, tipagem forte para a superfície gRPC/REST, alinhado ao control plane do AIOS (ADR-0003 global). | JVM (maior *footprint* de start); Go (ecossistema gRPC/OTel do .NET já padronizado no control plane). |
| **gRPC** (`aios.memory.v1`) | Comunicação interna com `006-Kernel`, `011-Context`, `023-Learning`, `017-Model-Router`, `022-Policy`. | Contrato fortemente tipado, streaming, baixa latência — essencial ao caminho quente de `remember`/`recall` (NFR-001/003). | REST interno (overhead de serialização/latência maior). |
| **REST (via YARP)** | Superfície externa `/v1/memory`. | Interoperabilidade com SDKs de terceiros e ferramentas de operação; RFC-0001 §5.7 exige versionamento por caminho. | GraphQL (complexidade desnecessária para 4 operações canônicas + auxiliares). |
| **PostgreSQL 16 (schema `memory`)** | Fonte da verdade de `MemoryItem` e demais entidades (§3 do brief); Long-Term/Semantic/Procedural/Episodic. | RLS nativo por `tenant_id`, transações ACID para Outbox, particionamento maduro (ADR-0005 herdada). | MongoDB (sem RLS nativo equivalente; sem extensão vetorial/grafo integrada ao mesmo motor transacional). |
| **pgvector (HNSW)** | Índice vetorial das camadas Semantic e Episodic. | ANN de alta qualidade (Recall@10 ≥ 0,95) integrado ao mesmo motor transacional do PostgreSQL, evitando um segundo *datastore* dedicado (ADR-0101). | Motor vetorial dedicado externo (Milvus/Pinecone) — custo operacional de um segundo sistema de consistência e RLS próprios, sem benefício líquido na escala-alvo (ADR-0101 documenta o *trade-off*). |
| **Apache AGE (openCypher)** | `KnowledgeGraphAdapter`: grafo de memória do agente. | Extensão nativa do PostgreSQL para openCypher — reaproveita RLS/transações/backup do mesmo cluster (ADR-0005 herdada). | Neo4j dedicado (segundo motor de persistência, sem RLS por tenant nativo, custo operacional adicional). |
| **Redis** | Working/Short-Term (hot), contadores de cota, cache de decisão/resultado, locks distribuídos. | Latência sub-ms, TTL nativo, scripts Lua atômicos para contabilização de cota sob concorrência (ADR-0006 herdada). | Memcached (sem estruturas ricas nem scripts atômicos; sem locks distribuídos nativos). |
| **MinIO** | `BlobStoreAdapter` (conteúdo > limite inline) e snapshots de `ConsolidationVersion`. | Object storage compatível S3, já padrão do AIOS para estado durável; content-addressed simplifica deduplicação e expurgo (ADR-0106). | Sistema de arquivos local (não escalável, sem durabilidade multi-nó). |
| **NATS/JetStream** | Transporte de eventos de domínio (`item.stored`, `consolidated`, `forgotten`, ...). | Baixa latência, *streams* duráveis, *replay*, barramento único padrão do AIOS (ADR-0004 herdada). | Kafka (operação mais pesada; NATS já é o barramento padrão do AIOS). |
| **OpenTelemetry + Prometheus + Grafana + Serilog/Seq** | `MemoryTelemetry`: spans por operação, métricas `aios_memory_*` incluindo Memory Recall Rate, logs correlacionados. | Padrão transversal do AIOS (ADR-0010 herdada); correlação `trace_id`/`tenant_id` obrigatória (RFC-0001 §5.6). | APM proprietário (vendor lock-in, sem padrão aberto). |

---

## 9. Mapa Canônico das 7 Camadas Físicas e Travessia de Planos

```
   PLANO DE DADOS                       PLANO DE CONTROLE                       ARMAZENAMENTO
 ┌───────────────────┐  syscall (gRPC) ┌────────────────────────┐   ┌─────────────────────────┐
 │ 007-Agent-Runtime │────brokerado───▶│      006-Kernel         │   │ Redis                    │
 │  (loop cognitivo,  │  por 006        │  (PEP + broker recurso) │   │  Working (15 min TTL)     │
 │   sandbox Python)  │◀────────────────│                        │   │  Short-Term hot (24h)     │
 └───────────────────┘  allow/deny/    └───────────┬────────────┘   └─────────────────────────┘
                          resultado                 │ remember/recall/            ▲
                                                       │ consolidate/forget          │ hot path
                                                       ▼                             │
                                          ┌─────────────────────────┐               │
                                          │      010-MEMORY          │───────────────┘
                                          │  (LayerRouter + Stores + │
                                          │   RecallEngine + ...)    │──────────────┐
                                          └───────────┬─────────────┘               │
                                                        │                            ▼
                                    ┌───────────────────┼───────────────────┐  PostgreSQL+pgvector+AGE
                                    ▼                   ▼                   ▼  Long-Term / Semantic /
                          embeddings (gRPC)     eventos (NATS)    blobs (S3) Procedural / Episodic / KG
                                    │                   │                   │
                                    ▼                   ▼                   ▼
                          017-Model-Router      020-Communication          MinIO
```

| Camada | Backend | Persistência | TTL default | Escopo típico |
|--------|---------|--------------|-------------|---------------|
| Working | Redis | Efêmera (RAM) | 15 min | sessão/agente |
| Short-Term | Redis (hot) + PostgreSQL (proj.) | Volátil→durável | 24 h | sessão |
| Long-Term | PostgreSQL | Durável | 90 dias (config.) | agente/tenant |
| Semantic | PostgreSQL + pgvector (HNSW) | Durável | sem TTL (decay) | agente/tenant |
| Procedural | PostgreSQL | Durável | sem TTL | agente/tenant |
| Episodic | PostgreSQL + pgvector (part. tempo) | Durável | 365 dias (config.) | agente |
| Knowledge Graph | Apache AGE (openCypher) | Durável | sem TTL | agente/tenant |
| Blobs grandes | MinIO | Durável (S3) | alinhado à retenção | qualquer camada |

O Memory é a **única travessia legítima** entre o plano de dados (onde o
agente raciocina) e os backends físicos de memória. Essa travessia única —
sempre brokerada pelo `006-Kernel` — é o que torna o *capability enforcement*,
o *quota enforcement* e a auditoria de acesso à memória universais (R6, R10,
NFR-010, NFR-013).

---

## 10. Riscos Arquiteturais e Trade-offs

| Risco/Trade-off | Descrição | Mitigação |
|-------------------|-----------|-----------|
| Único motor (PostgreSQL) para relacional + vetorial + grafo | Acopla a escala vetorial/grafo à escala transacional do PostgreSQL. | Sharding determinístico por (tenant, agente); réplicas de leitura para recall; caminho de migração para índice vetorial externo previsto em ADR-0101/ADR-0108 caso a escala-alvo (10⁸ vetores/tenant) exija. |
| Consolidação incorreta pode degradar qualidade de recall | Promoção/merge mal calibrado pode reduzir Memory Recall Rate pós-consolidação. | `ConsolidationVersionManager` versiona toda mudança; NFR-011 exige rollback automático se Recall Rate cair > 0,01 vs. pré-consolidação. |
| Crescimento descontrolado de itens/vetores por tenant | Sem poda ativa, uso tende a crescer indefinidamente. | `ForgettingEngine` + `QuotaManager` aplicam decaimento/TTL/LRU/cota continuamente (NFR-009); `RetentionScheduler` agenda jobs periódicos. |
| Latência do PEP em cada operação privilegiada | Consulta ao PDP soma latência ao caminho crítico de `remember`/`recall`. | Cache de decisões com TTL curto no `MemoryPep`; meta p99 dentro de NFR-001/003 já contempla esse custo. |
| Dependência de disponibilidade do Model Router para `remember` com embedding | Falha do `017` bloquearia a geração de embedding. | Item persiste em `INGESTED` sem embedding e é enfileirado para geração em backlog (F3 do brief); não bloqueia a escrita do conteúdo. |
| Reidratação de itens `ARCHIVED` sob recall adiciona latência de cauda | Item frio movido para MinIO precisa ser trazido de volta para responder a um recall. | Reidratação transparente (FR-012); latência de cauda aceita como custo do modelo hot/cold (NFR-003 aplica-se ao caminho quente predominante). |

---

## 11. Alternativas Descartadas

| Alternativa | Por que foi descartada |
|-------------|--------------------------|
| Motor vetorial dedicado externo (Milvus/Pinecone/Weaviate) em vez de pgvector | Introduziria um segundo sistema de consistência e RLS por tenant a manter, sem integração transacional com o restante do `MemoryItem`; ADR-0101 documenta a análise completa. |
| Neo4j dedicado para o Knowledge Graph de memória do agente | Segundo motor de persistência fora do cluster PostgreSQL padrão do AIOS, sem RLS nativo por tenant e com custo operacional adicional; Apache AGE reaproveita a infraestrutura já padronizada (ADR-0005 herdada). |
| Agentes acessarem PostgreSQL/Redis/MinIO diretamente para memória (sem broker) | Elimina o ponto único de *capability enforcement* e cota sobre memória; impossibilita auditoria centralizada e viola a fronteira crítica do brief (regra de camadas de `001-Architecture` §6; ADR-0107). |
| Esquecimento apenas por TTL fixo (sem decaimento por saliência/uso) | Não diferencia itens importantes de itens triviais; poda cega degradaria qualidade de recall; substituído por política combinada TTL+decay+LRU+cota+RTBF (ADR-0103). |
| Consolidação sem versionamento (mutação direta das camadas) | Sem mecanismo de rollback, uma consolidação incorreta causaria *catastrophic forgetting* irreversível; substituído por snapshot versionado obrigatório antes de toda mutação (ADR-0102). |
| Embeddings gerados localmente pelo próprio Memory Service | Violaria a fronteira de responsabilidade do Model Router (N1) e duplicaria lógica de seleção de modelo/custo já centralizada em `017`; ADR-0101 reforça o caminho único via `EmbeddingClient`. |

---

## 12. Referências e ADRs

- Fonte de verdade do módulo: `./_DESIGN_BRIEF.md`
- Arquitetura global: `../001-Architecture/Architecture.md`
- Contratos centrais: `../003-RFC/RFC-0001-Architecture-Baseline.md`
- Glossário: `../040-Glossary/Glossary.md`
- ADRs globais consumidas: `../002-ADR/ADR-0003-DotNet-Control-Python-Runtime.md`,
  `../002-ADR/ADR-0004-NATS-como-Barramento.md`,
  `../002-ADR/ADR-0005-PostgreSQL-pgvector-AGE.md`,
  `../002-ADR/ADR-0006-Redis-Estado-Quente.md`,
  `../002-ADR/ADR-0008-Governanca-por-Politica-Default-Deny.md`,
  `../002-ADR/ADR-0010-Observabilidade-Auditoria-por-Construcao.md`
- ADRs do módulo (a propor, faixa `ADR-0100`–`ADR-0109`): ver `./ADR.md` para o
  índice detalhado — modelo de 7 camadas (ADR-0100), embeddings/HNSW (ADR-0101),
  consolidação versionada/rollback (ADR-0102), políticas de esquecimento
  (ADR-0103), fusão de recall híbrido (ADR-0104), cotas/backpressure
  (ADR-0105), externalização de blobs (ADR-0106), fronteira Runtime↔Memory
  (ADR-0107), sharding vetorial/temporal (ADR-0108), retenção legal/RTBF
  (ADR-0109).
- RFCs do módulo (a propor): ver `./RFC.md` — RFC-0100 (Memory Consolidation &
  Forgetting Contract).
- Componentes acoplados: `../006-Kernel/`, `../011-Context/`, `../017-Model-Router/`,
  `../021-Security/`, `../022-Policy/`, `../023-Learning/`, `../024-Observability/`,
  `../025-Audit/`, `../026-Cost-Optimizer/`.

*Fim do documento `Architecture.md` do módulo 010-Memory.*
