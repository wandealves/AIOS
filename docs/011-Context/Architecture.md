---
Documento: Architecture
Módulo: 011-Context
Status: Draft
Versão: 0.1
Última atualização: 2026-07-20
Responsável (RACI-A): Arquiteto do Módulo 011-Context
ADRs relacionados: ADR-0110, ADR-0111, ADR-0112, ADR-0113, ADR-0114, ADR-0115, ADR-0116, ADR-0117, ADR-0118, ADR-0119
RFCs relacionados: RFC-0001, RFC-0011 (proposta)
Depende de: 010-Memory, 017-Model-Router, 024-Observability, 022-Policy, 025-Audit, 020-Communication, 005-Database, 021-Security
---

# AIOS — Módulo 011 · Context — Architecture

> Derivado de `./_DESIGN_BRIEF.md` (§2). Componentes canônicos em `PascalCase`
> conforme brief §2.2; contratos centrais (URN, evento, erro, idempotência,
> correlação, subjects) são da RFC-0001
> (`../003-RFC/RFC-0001-Architecture-Baseline.md`) — **referenciados, não
> redefinidos**. Diagramas SEMPRE em ASCII. Palavras normativas conforme
> RFC 2119/8174.

## 1. Visão geral

O `ContextService` é um serviço do **plano de controle**, implementado em
**.NET 10**, exposto por **REST/gRPC** através do gateway **YARP**, publicando
eventos em **NATS (JetStream)** e persistindo estado em **Redis** (cache quente
L1 e locks), **PostgreSQL** (fonte da verdade, com `pgvector` para embeddings de
fragmentos/cache — **AGE não é usado por este módulo**, pois o grafo de
conhecimento pertence a `018/019-Knowledge`) e **MinIO** (offload de conteúdo
grande). Ele é o análogo cognitivo do subsistema de **gerência de memória
virtual/paginação** de um sistema operacional clássico: decide o que "cabe na
página" (janela de contexto ativa do modelo), o que é "comprimido/compactado"
(sumarização hierárquica) e o que "evita recomputar" (cache semântico L1/L2).
Esta seção apresenta a arquitetura em três níveis C4 (Contexto → Contêiner →
Componente), derivados do brief §2, e não introduz nenhum componente, fronteira
ou tecnologia que o brief não preveja.

O módulo **NÃO** decide qual modelo executa a tarefa nem seus limites de janela
(`017-Model-Router`), **NÃO** persiste memória de longo prazo (`010-Memory`),
**NÃO** executa ferramentas (`015-Tool-Manager`/`007-Agent-Runtime`) e **NÃO**
mantém o grafo de conhecimento (`018/019-Knowledge`) — ver brief §1.3.

## 2. C4 Nível 1 — Contexto

```
                          ┌───────────────────────────────────────────────┐
   Agent Runtime (007) ──▶│                                               │
   / Kernel (006)         │                                               │◀─ Model Router (017)
   (solicita contexto     │             CONTEXT SERVICE (011)             │   (limites/tokenizer/
    p/ chamada de LLM)    │  assemble · compress · cache · budget         │    embedding/summarização)
                          │  gerência de janela de contexto (paginação    │
   Planning (012) ───────▶│  cognitiva: cache/compressão/recuperação)     │──▶ Memory (010)
   (contexto p/ replan)   │                                               │   (recall seletivo)
                          │                                               │
   Communication (020) ──▶│                                               │──▶ Policy Engine (022)
   (invalidação por evento)│                                              │   (PDP: default deny)
                          └────────┬───────────────┬──────────────┬───────┘
   Cost-Optimizer (026) ◀──────────┘               │              └────────▶ Audit (025)
   (tokens/custo evitado)                          │                        (eventos → trilha)
   Security (021) ────────▶ (mTLS/OIDC)             ▼
                                            NATS (JetStream)
                                     Redis · PostgreSQL(+pgvector) · MinIO
                                                     │
                                                     └────────────────────▶ Observability (024)
                                                        (OTel: Context Compression Ratio, cache hit ratio)
```

O `011-Context` é consumido primariamente por `Agent Runtime (007)` (via
`Kernel`/syscalls cognitivas) e por `Planning (012)` sempre que uma chamada de
LLM precisa de um `ContextBundle` dentro do orçamento de tokens do modelo alvo.
Ele nunca chama o provedor de LLM para a inferência final da tarefa — apenas
usa modelos auxiliares (embedding, sumarização) roteados pelo `017`. Ver
`./Vision.md` §3.2 para a fronteira completa in/out de escopo.

## 3. C4 Nível 2 — Contêiner

```
┌───────────────────────── CONTEXT SERVICE (011) · .NET 10 · plano de controle ─────────────────────────┐
│                                                                                                         │
│   ┌───────────────┐      gRPC/REST (YARP)        ┌──────────────────────────────────────────────┐      │
│   │ API Contêiner │◀────────────────────────────│ Ingress / correlação (traceparent, tenant,     │      │
│   │ (Facade + PEP)│                               │  Idempotency-Key)                             │      │
│   └──────┬────────┘                               └──────────────────────────────────────────────┘      │
│          │ comandos validados                                                                           │
│   ┌──────▼───────────────┐   ┌───────────────────────┐   ┌───────────────────────────────────────┐    │
│   │ Contêiner de Assembly │   │ Contêiner de Cache      │   │ Contêiner de Telemetria/Outbox        │    │
│   │ (Budgeter+Retriever+  │   │ Semântico (CacheMgr +   │   │ (TelemetryEmitter, EventPublisher)    │    │
│   │  Ranker+Dedup+Compr.) │   │  EvictionManager, L1/L2)│   └───────────────┬───────────────────────┘    │
│   └──────┬───────────────┘   └──────────┬──────────────┘                   │                            │
│          │                              │                                  ▼                            │
└──────────┼──────────────────────────────┼──────────────────────────── NATS (JetStream) ─────────────────┘
           ▼                              ▼
   Redis (L1 cache/locks)   PostgreSQL (+pgvector, RLS por tenant)         MinIO (blobs de fragmentos)
                            fonte da verdade: bundles, fragments, cache L2, budgets, outbox
```

| Contêiner lógico | Papel | Backends |
|-------------------|-------|----------|
| API (Facade + PEP) | Superfície REST/gRPC, authN/Z, idempotência, versionamento (`/v1`). | Redis (idempotência), `022-Policy` |
| Assembly | Orquestração budget→retrieve→rank→dedup→compress. | Redis, PostgreSQL, `010-Memory`, `017-Model-Router` |
| Cache Semântico | Lookup/store/invalidação/eviction do cache indexado por similaridade. | Redis (L1), PostgreSQL+`pgvector` (L2) |
| Telemetria/Outbox | OTel + publicação de eventos via Outbox transacional. | NATS, PostgreSQL, `024-Observability` |

## 4. C4 Nível 3 — Componente (canônico, brief §2.2)

```
┌──────────────────────────────── CONTEXT SERVICE (011) ─────────────────────────────────┐
│                                                                                          │
│   REST/gRPC (X-AIOS-Tenant, Idempotency-Key, traceparent)                               │
│        │                                                                                │
│  ┌─────▼──────────────┐        ┌──────────────────────┐                                 │
│  │ ContextApiGateway  │───PEP─▶│ ContextPolicyGuard    │──▶ Policy Engine (022)          │
│  │ (validação/authZ)  │        └──────────────────────┘                                 │
│  └─────┬──────────────┘                                                                 │
│        │ assemble()                                                                     │
│  ┌─────▼───────────────────────────────────────────────────────────────────────┐        │
│  │                        ContextAssembler (orquestrador)                       │        │
│  │                                                                              │        │
│  │  1.┌──────────────┐  2.┌────────────────────┐   ──cache lookup──┐            │        │
│  │    │ TokenBudgeter│    │ SemanticCacheManager│◀───────────────── ┘           │        │
│  │    └──────┬───────┘    └──────┬──────────────┘   HIT? ─────────▶ serve+emit  │        │
│  │           │ limites           │ L1 Redis / L2 pgvector                       │        │
│  │  ┌────────▼───────┐  3.┌──────▼──────────┐  4.┌──────────────────┐          │        │
│  │  │ TokenCounter   │    │SelectiveRetriever│──▶│ RelevanceRanker  │          │        │
│  │  └────────────────┘    └──────┬──────────┘   └────────┬─────────┘          │        │
│  │           ▲                   │ recall              5.│                     │        │
│  │           │             ┌─────▼──────────┐   ┌────────▼──────────┐          │        │
│  │           │             │ RedundancyElim.│──▶│HierarchicalCompr. │          │        │
│  │           │             └────────────────┘   └────────┬──────────┘          │        │
│  │  ┌────────┴────────┐         ▲   ▲               6.   │ assembled            │        │
│  │  │ EmbeddingClient │─────────┘   │            ┌────────▼──────────┐          │        │
│  │  └────────┬────────┘             │            │ BundleStore       │──▶ MinIO │        │
│  │           │                      │            └────────┬──────────┘ (blobs) │        │
│  └───────────┼──────────────────────┼─────────────────────┼──────────────────┘        │
│              │                      │                       │ outbox                    │
│  ┌───────────▼────────┐  ┌──────────▼────────┐   ┌─────────▼─────────┐                  │
│  │ ModelRouterClient  │  │ MemoryClient       │   │ EventPublisher    │──▶ NATS/JS       │
│  │ (limites/tokenizer/│  │ (recall seletivo)  │   │ (context.window.*)│  aios.<t>.context│
│  │  embedding/summ.)  │  └──────────┬─────────┘   └────────────────────┘                 │
│  └───────────┬────────┘             │                                                     │
│              │                 ┌────▼───────┐   ┌──────────────────┐                     │
│              ▼                 │ EvictionMgr│   │ TelemetryEmitter │──▶ OTel/Prom/Seq     │
│      Model Router (017)        └────────────┘   └──────────────────┘                     │
│                                                                                            │
│  Persistência: PostgreSQL(+pgvector) · Redis(L1 cache/locks) · MinIO(blobs)                │
└────────────────────────────────────────────────────────────────────────────────────────────┘
```

**Padrão arquitetural do fluxo:** `Gateway → PEP → Orquestrador → (cache
lookup) → retrieve → rank → dedup → compress → store → outbox/eventos`,
alinhado ao padrão canônico da Arquitetura Global (§5 de
`../001-Architecture/Architecture.md`) — idêntico em forma ao `_DESIGN_BRIEF.md`
§2.2, reproduzido aqui sem divergência.

## 5. Catálogo de componentes internos

Cada componente é `PascalCase` e canônico (brief §2.1). Responsabilidades e
dependências abaixo NÃO PODEM divergir do brief.

| Componente | Responsabilidade | Depende de |
|------------|-------------------|------------|
| **ContextApiGateway** | Superfície REST/gRPC do módulo (`/v1`); validação de payload, authN (claims OIDC), invocação do PEP, verificação de `Idempotency-Key`, versionamento de API. | `ContextPolicyGuard`, `ContextAssembler`, `SemanticCacheManager`, `021-Security` |
| **ContextPolicyGuard** | PEP local: consulta o PDP (`022`) para operações privilegiadas (assemble para outro agente, upsert de `BudgetProfile`, invalidação em massa); *default deny*. | `022-Policy` |
| **ContextAssembler** | Orquestrador do pipeline de montagem; coordena `budget → cache lookup → retrieve → rank → dedup → compress → persist → emit`; é a única via de mutação de estado de um `ContextBundle`. | Todos os componentes internos abaixo |
| **TokenBudgeter** | Calcula e aloca o orçamento de tokens por seção (system/instrução/memória/histórico/tools) a partir de `BudgetProfile` e dos limites do modelo alvo. | `ModelRouterClient`, `TokenCounter` |
| **TokenCounter** | Conta/estima tokens por tokenizer do modelo alvo, via registro de tokenizers exposto pelo `017`. | `ModelRouterClient` |
| **SelectiveRetriever** | Solicita recall seletivo (top-K, `context.retrieval.top_k`) ao `010-Memory` dentro de um *deadline* configurável; coleta candidatos de contexto. | `MemoryClient` |
| **RelevanceRanker** | Rankeia/scoreia fragmentos candidatos combinando similaridade semântica, recência e prioridade declarada da fonte. | `EmbeddingClient`, `SelectiveRetriever` |
| **RedundancyEliminator** | Detecta e remove *near-duplicates* e sobreposição semântica entre fragmentos (`cosine ≥ context.dedup.cosine_threshold`). | `EmbeddingClient` |
| **HierarchicalCompressor** | Compressão/sumarização *map-reduce* multinível; extrai `SummaryNode`; aplica *fallback* determinístico de truncamento em falha. | `ModelRouterClient` (sumarização), `TokenCounter` |
| **SemanticCacheManager** | Lookup/store/invalidação do cache semântico em dois níveis (Redis L1 + `pgvector` L2), chave por similaridade (`prompt_embedding`) e por *fast path* (`prompt_hash`). | `EmbeddingClient`, Redis, PostgreSQL |
| **EmbeddingClient** | Gera embeddings de prompts/fragmentos para chave de cache e para ranking; nunca chama LLM de inferência final. | `017-Model-Router` (modelo de embedding) |
| **EvictionManager** | Política de eviction/expiração do cache (TTL + LRU semântico + invalidação *event-driven*). | `SemanticCacheManager`, Redis |
| **BundleStore** | Persistência de `ContextBundle`/`ContextFragment`; *offload* de blobs grandes (`> context.limits.max_inline_bytes`) ao MinIO. | PostgreSQL, MinIO |
| **EventPublisher** | Publica eventos `aios.<tenant>.context.*` via *outbox* transacional (DB + NATS). | NATS/JetStream, `BundleStore` |
| **ModelRouterClient** | Cliente gRPC para limites de modelo, registro de tokenizer e endpoints de embedding/sumarização. | `017-Model-Router` |
| **MemoryClient** | Cliente gRPC/NATS para recall seletivo no `010-Memory`. | `010-Memory` |
| **TelemetryEmitter** | Instrumentação OTel (traces/metrics/logs Serilog→Seq); expõe `aios_context_*` e **Context Compression Ratio**. | `024-Observability` |

## 6. Fronteiras e contratos

| Fronteira | Regra normativa | Referência |
|-----------|------------------|------------|
| Runtime/Kernel ↔ Context | Chamadores **NÃO DEVEM** acessar Redis/PostgreSQL do módulo diretamente; DEVEM usar REST/gRPC. | brief §1.2, ADR-0113 |
| Context ↔ Model Router (017) | Limites de modelo, tokenizer, embedding e sumarização **DEVEM** vir do `017`; o módulo **NÃO DEVE** chamar provedor de LLM para inferência final da tarefa. | brief §1.3 N-01/N-03, ADR-0114/ADR-0115 |
| Context ↔ Memory (010) | Recall seletivo é **solicitado** ao `010`; o `011` **NÃO DEVE** persistir, consolidar ou decidir esquecimento de memória. | brief §1.3 N-02, ADR-0116 |
| Context ↔ Policy (022) | Toda operação privilegiada **DEVE** passar pelo `ContextPolicyGuard` → PDP (*default deny*). | RFC-0001 §5.8 |
| Context ↔ Audit (025) | O módulo **emite** eventos auditáveis; a trilha imutável é do `025`. | brief §1.3 N-08 |
| Sumarização efêmera vs. memória canônica | `SummaryNode` gerado aqui **PODE** ser cache de curta duração; **NÃO DEVE** ser tratado como memória canônica de longo prazo. | brief §1.3 (nota de fronteira) |
| Identificadores/eventos/erros | URN, envelope CloudEvents, envelope RFC 7807, idempotência, correlação. | RFC-0001 §5 |

## 7. Padrões arquiteturais adotados

| Padrão | Onde | Motivo |
|--------|------|--------|
| **Pipeline (Chain of Responsibility)** | `ContextAssembler` (budget→retrieve→rank→dedup→compress) | Cada estágio é isolado, testável e substituível sem acoplar os demais. |
| **Strategy** | `HierarchicalCompressor` (`compression_method`: hierarchical/extractive/truncate) | Troca de algoritmo de compressão por configuração (`context.compression.method`) sem mudar contrato externo. |
| **Cache-aside em dois níveis (L1/L2)** | `SemanticCacheManager` (Redis L1 + `pgvector` L2) | Latência sub-ms no *hot path*; durabilidade e recall vetorial no L2. |
| **Outbox transacional** | `EventPublisher` | Atomicidade entre persistência do bundle e publicação de evento, entrega *at-least-once*. |
| **PEP/PDP** | `ContextPolicyGuard` + Policy (`022`) | Autorização externa, *default deny* (RFC-0001 §5.8). |
| **Idempotency key** | `ContextApiGateway` (registro por chave) | Deduplicação de mutações (`assemble`, `cache store`, `budget upsert`) — RFC-0001 §5.5. |
| **Single-flight de cache-fill** | `SemanticCacheManager` (lock Redis `SET NX PX`) | Evita *thundering herd* em `miss` concorrente sobre a mesma chave de similaridade. |
| **Circuit breaker** | `ModelRouterClient`, `MemoryClient` | Isola falhas de dependências (`010`/`017`) e habilita degradação graciosa. |
| **Sharding determinístico** | Cache/bundles (`hash(tenant_id, agent_id) mod context.shard.count`) | Escala horizontal sem afinidade; consistente com brief §10. |

## 8. Stack tecnológica e justificativas

| Tecnologia | Uso no módulo | Justificativa | Alternativa descartada / ADR |
|------------|-----------------|----------------|-------------------------------|
| **.NET 10** | Serviço do plano de controle | Baixa latência, AOT, ecossistema do plano de controle AIOS; caminho quente de `assemble` exige p99 ≤ 150 ms (NFR-002). | Python (é plano de dados) — `../001-Architecture/` |
| **Redis** | Cache L1 do cache semântico, locks de *single-flight*, idempotência | TTL nativo, latência sub-ms, evita busca vetorial no caminho de `hit` frequente. | — |
| **PostgreSQL** | Fonte da verdade de `ContextBundle`/`ContextFragment`/`SemanticCacheEntry`/`BudgetProfile`, RLS por tenant | Durabilidade, transações para *outbox*, RLS multi-tenant nativo. | ADR-0111 |
| **pgvector (HNSW/IVFFLAT)** | Busca ANN de fragmentos e chave de similaridade do cache L2 | ANN dentro do PostgreSQL, sem 2º *datastore* dedicado; HNSW default por latência, IVFFLAT alternativa configurável para datasets menores. | ADR-0111, ADR-0112 |
| **MinIO (S3)** | Offload de fragmentos/blobs grandes (`content_ref`) | *Content-addressed*, barato, desacopla o PostgreSQL de payloads grandes. | Inline no PostgreSQL — ADR-0117 |
| **NATS JetStream** | Eventos `context.window.*`/`context.cache.*`/`context.budget.*` | *At-least-once*, contas por tenant, consumo assíncrono de invalidação. | — |
| **YARP** | Ingress REST/gRPC | Gateway do plano de controle .NET, consistente com o restante do AIOS. | — |
| **OpenTelemetry / Serilog→Seq** | Traces/metrics/logs | Correlação obrigatória (RFC-0001 §5.6); cobertura 100% em caminhos críticos (NFR-013). | — |

> **Apache AGE não é utilizado por este módulo.** Ainda que a stack global do
> AIOS inclua AGE, o `011-Context` **NÃO DEVE** manter grafo de conhecimento —
> essa responsabilidade é exclusiva de `018/019-Knowledge` (brief §1.3 N-05).

## 9. Reservas do módulo (brief §0/§11)

| Recurso | Faixa reservada |
|---------|-------------------|
| Domínio de subject NATS | `context` (`aios.<tenant>.context.<entidade>.<acao>`) |
| Domínio de código de erro | `AIOS-CTX-0001`..`AIOS-CTX-0015` |
| Faixa de ADR | `ADR-0110`..`ADR-0119` |
| RFC de módulo | `RFC-0011` (proposta) |
| Tipos de URN | `context` (bundle), `ctxfrag` (fragmento), `ctxcache` (entrada de cache) |
| Pacote gRPC | `aios.context.v1` |
| Prefixo de métrica | `aios_context_*` |
| Namespace de config | `context.*` |

## 10. Alternativas descartadas (resumo)

- **Cache semântico apenas em Redis** (sem L2 durável) — descartado por perder
  o cache em restart/eviction de memória e por não escalar para o volume de
  entradas exigido por `context.cache.max_entries_per_tenant`; L2 em
  `pgvector` mantém durabilidade e permite ANN em escala — ADR-0111.
- **Truncamento simples como único método de compressão** — descartado como
  padrão por degradar excessivamente a **Task Completion Rate**; adotada
  sumarização hierárquica *map-reduce* como padrão, com truncamento como
  *fallback* de falha, não como estratégia primária — ADR-0110.
- **Cache chaveado apenas por hash exato do prompt** — descartado como único
  mecanismo por não capturar prompts semanticamente equivalentes; adicionado
  o *slow path* por `prompt_embedding`/similaridade — ADR-0111, ADR-0112.
- **Grafo de conhecimento embutido no módulo** — descartado; conhecimento
  consolidado pertence a `018/019-Knowledge`; o `011` apenas **consome**
  fragmentos de conhecimento como candidatos de contexto quando fornecidos.
- **Chamada direta a provedor de LLM para sumarização** — descartada; toda
  chamada de modelo (embedding/sumarização) é roteada pelo `017-Model-Router`,
  preservando um único ponto de política de custo/qualidade — ADR-0114/ADR-0115.

Todas as decisões acima estão indexadas em `./ADR.md` e não são decididas
localmente neste documento.

## 11. Riscos arquiteturais

| Risco | Impacto | Mitigação | Referência |
|-------|---------|-----------|------------|
| Falso-*hit* de cache semântico entre tarefas divergentes | Resposta inadequada servida (qualidade) | `similarity_threshold` conservador + auditoria de amostragem (NFR-011) + `ADR-0112`/`ADR-0119` | brief §12.2 (STRIDE — Information disclosure) |
| Degradação de dependência (`010`/`017`) sob carga | Contexto parcial/estimativa imprecisa | Degradação graciosa (`DEGRADED`), *circuit breaker*, limites cacheados | brief §9 FM-01/FM-02 |
| Crescimento não controlado do cache L2 | Custo de armazenamento, latência ANN | `context.cache.max_entries_per_tenant`, `EvictionManager` (TTL+LRU semântico) | NFR-009, ADR-0116 |
| Vazamento de contexto entre tenants via cache | Violação de isolamento (LGPD) | RLS por `tenant_id` + namespace de cache por tenant + testes de fuzz (NFR-014) | brief §12.2/§12.3, ADR-0119 |
| Sumarização introduz alucinação/perda semântica | Queda de Task Completion Rate | Verificação de fidelidade no `HierarchicalCompressor`, meta mínima de compressão configurável | NFR-005, ADR-0115 |
