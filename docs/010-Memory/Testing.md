---
Documento: Testing
Módulo: 010-Memory
Status: Draft
Versão: 0.1
Última atualização: 2026-07-20
Responsável (RACI-A): Arquiteto do Módulo 010-Memory
ADRs relacionados: ADR-0100, ADR-0101, ADR-0102, ADR-0104, ADR-0105, ADR-0107, ADR-0109
RFCs relacionados: RFC-0001, RFC-0100
Depende de: 001-Architecture, 003-RFC (RFC-0001), 005-Database, 011-Context, 017-Model-Router, 021-Security, 022-Policy, 023-Learning, 040-Glossary
---

# 010-Memory — Testing (Estratégia de Testes)

Este documento define a **estratégia de testes** do `MemoryService`: níveis (unit,
integration, contract, e2e, chaos, load), cobertura-alvo, fixtures, ambientes e **gates de
qualidade**. Todo caso de teste DEVE rastrear a um `FR-NNN`/`NFR-NNN` da §7 do
`_DESIGN_BRIEF.md` (ver [FunctionalRequirements.md](./FunctionalRequirements.md) e
[NonFunctionalRequirements.md](./NonFunctionalRequirements.md)).

Palavras normativas conforme RFC 2119/8174. Contratos verificados reutilizam
[RFC-0001](../003-RFC/RFC-0001-Architecture-Baseline.md) e não são redefinidos.

---

## 1. Pirâmide de testes

```
                 ┌───────────────┐
                 │   Chaos/DR    │  raros, ambiente staging-iso  (F1–F10)
                 ├───────────────┤
                 │   Load/Perf   │  por release  (NFR-001/003/004/012)
                 ├───────────────┤
                 │      E2E       │  fluxos ponta-a-ponta reais
                 ├───────────────┤
                 │   Contract     │  API/eventos vs. consumidores (011/023/025/026)
                 ├───────────────┤
                 │  Integration   │  com Redis/PG+pgvector+AGE/MinIO/NATS reais
                 ├───────────────┴──────────────┐
                 │           Unit               │  base larga, rápida, isolada
                 └──────────────────────────────┘
```

| Nível | O que valida | Backends | Frequência | Duração-alvo |
|-------|--------------|----------|------------|--------------|
| **Unit** | Lógica pura de componentes (roteamento, decay, fusão RRF, guardas da FSM). | *mocks/fakes* | cada commit | < 2 min total |
| **Integration** | Componente + backend real (pgvector/HNSW, AGE, Redis, MinIO, NATS). | Testcontainers | cada PR | < 15 min |
| **Contract** | Aderência a RFC-0001 (URN, erro, evento, idempotência) e a schemas de consumidores. | Pact/Verifier | cada PR | < 5 min |
| **E2E** | Fluxos completos `remember→recall→consolidate→forget`. | ambiente efêmero | cada merge em `main` | < 30 min |
| **Load** | SLO de latência/throughput/escala vetorial. | ambiente perf | cada release | ver [Benchmark.md](./Benchmark.md) |
| **Chaos/DR** | Modos de falha §9 do brief e RTO/RPO. | staging isolado | mensal / pré-GA | variável |

---

## 2. Testes unitários

Cobrem lógica determinística **sem** I/O. Componentes-alvo (§2.2 do brief): `LayerRouter`,
`ForgettingEngine` (decay/TTL/LRU), `RecallEngine` (fusão RRF/re-rank), `QuotaManager`,
`MemoryPep` (guardas), máquinas de estado (`MemoryItem`, `ConsolidationJob`).

| Caso (exemplos) | Verifica | Rastreia |
|-----------------|----------|----------|
| Roteamento por `kind`→`layer`; `kind` incompatível → `AIOS-MEM-0020`. | FR-004 | `LayerRouter` |
| Fusão RRF combina lexical+vetor+grafo mantendo ordenação estável. | FR-002, ADR-0104 | `RecallEngine` |
| Decaimento: `decay_score < threshold` ⇒ `DECAYING`. | FR-006 | `ForgettingEngine` |
| Transição inválida da FSM é rejeitada (ex.: `PURGED→ACTIVE`). | §4.1 brief | FSM `MemoryItem` |
| `legal_hold` bloqueia purge salvo RTBF autorizado → `AIOS-MEM-0050`. | FR-011 | invariante (i) |
| Snapshot obrigatório antes de `CONSOLIDATING` (invariante ii). | FR-005 | `ConsolidationEngine` |

Exemplo (xUnit, .NET 10):

```csharp
[Fact] // FR-004: kind incompatível com a camada deve falhar com AIOS-MEM-0020
public void Route_IncompatibleKindForLayer_ThrowsMem0020()
{
    var router = new LayerRouter(_quota, _embed, _stores);
    var item = new MemoryItem { Kind = MemoryKind.Skill, LayerHint = MemoryLayer.Working };
    var ex = Assert.Throws<MemoryException>(() => router.Route(item));
    Assert.Equal("AIOS-MEM-0020", ex.Code);
}
```

---

## 3. Testes de integração

Usam **backends reais efêmeros** via Testcontainers: PostgreSQL (com `pgvector` + `Apache AGE`),
Redis, MinIO, NATS/JetStream. Validam o que *mocks* não conseguem: índice HNSW, RLS,
travessia openCypher, TTL Redis, Outbox→NATS.

| Caso | Verifica | Rastreia |
|------|----------|----------|
| Inserção + busca ANN HNSW retorna vizinhos corretos; Recall@10 ≥ 0,95 vs. exata. | FR-003, NFR-002 | `SemanticMemoryStore` |
| RLS: consulta com `tenant=A` nunca retorna linha de `tenant=B`. | NFR-010 | isolamento |
| Blob > `inline_max_bytes` (65536) vira `content_ref` MinIO + `content_hash`. | FR-009 | `BlobStoreAdapter` |
| Outbox: commit DB gera exatamente um evento publicado (at-least-once + dedup). | R9, F8 | `OutboxPublisher` |
| TTL Working (900 s) expira item; Short-Term projeta em PG (CQRS). | §3.3 brief | `WorkingMemoryStore` |
| `mode=graph` retorna itens conectados via AGE; AGE fora ⇒ `AIOS-MEM-0031`. | FR-013, F5 | `KnowledgeGraphAdapter` |
| Idempotência: `remember` repetido com mesma key retorna resultado memoizado. | RFC-0001 §5.5, F9 | `IdempotencyStore` |

---

## 4. Testes de contrato

Garantem que a superfície pública **não regride** e adere aos contratos centrais.

| Contrato | Método | Contraparte |
|----------|--------|-------------|
| REST/gRPC (`aios.memory.v1`) | Snapshot de OpenAPI/proto + *breaking-change gate*. | Clientes ([API.md](./API.md)) |
| Envelope de erro `AIOS-MEM-NNNN` RFC 7807 | Todo erro do catálogo (§5.2 brief) produz o envelope correto. | RFC-0001 §5.4 |
| Envelope de evento CloudEvents | Cada evento emitido valida contra `dataschema` registrado. | RFC-0001 §5.2 |
| Correlação | Toda resposta/evento carrega `trace_id`/`tenant`. | RFC-0001 §5.6 |
| Consumidores | `memory.item.consolidated`→023/019; `.forgotten`/`.purged`→025; `.recalled`→011; `.quota.exceeded`→026. | §6.1 brief (Pact) |
| Eventos consumidos | Reagir a `learning.consolidation.requested`, `context.recall.requested`, `security.rtbf.requested`, `agent.lifecycle.*`. | §6.2 brief |

Exemplo (validação de evento contra schema):

```csharp
[Fact] // RFC-0001 §5.2: evento consolidated adere ao dataschema registrado
public async Task ConsolidatedEvent_MatchesRegisteredSchema()
{
    var evt = await CaptureOutboxEvent("aios.acme.memory.item.consolidated");
    Assert.Equal("aios.memory.item.consolidated", evt.Type);
    Assert.True(SchemaRegistry.Validate(evt.DataSchema, evt.Data));
    Assert.Matches(@"^00-[0-9a-f]{32}-[0-9a-f]{16}-\d{2}$", evt.TraceParent);
}
```

---

## 5. Testes E2E

Exercitam jornadas completas em ambiente efêmero com todos os backends e um *stub* do
Model Router (017) e do Policy Engine (022).

| UC / Jornada | Passos | Rastreia |
|--------------|--------|----------|
| Ciclo de vida do item | `remember` → `recall` → `consolidate` → `forget` → `purge` (RTBF). | FR-001/002/005/006/011 |
| Consolidação dirigida por 023 | Evento `learning.consolidation.requested` → job COMMITTED → `consolidated`. | FR-005 |
| Rollback anti-regressão | Consolidação que baixa Recall Rate > 0,01 dispara rollback automático. | FR-007, NFR-011 |
| RTBF ponta-a-ponta | `security.rtbf.requested` → purge + evento `purged` + auditoria 025. | FR-011, LGPD |
| Reidratação | Item `ARCHIVED` (MinIO) reidratado transparentemente no `recall`. | FR-012 |

---

## 6. Testes de caos e DR

Injetam as falhas da §9 do brief e verificam degradação graciosa e RTO/RPO.

| Experimento | Falha injetada | Resultado esperado | Rastreia |
|-------------|----------------|--------------------|----------|
| Redis kill | derruba Redis (Working/hot) | fail-open p/ camadas duráveis; Working reconstruível; RTO ≤ 1 min. | F1, NFR-008 |
| PG failover | mata primário PostgreSQL | failover p/ standby; RPO ≤ 5 min; RTO ≤ 15 min; Outbox reprocessa. | F2, NFR-006/007 |
| Model Router down | 017 indisponível | item fica `INGESTED` sem embedding; backlog processa depois; sem perda. | F3 |
| MinIO down | S3 indisponível | conteúdo inline se possível; blob represado; retry idempotente por hash. | F4 |
| AGE down | grafo falha | recall degrada p/ hybrid sem grafo; `mode=graph`→`AIOS-MEM-0031`. | F5 |
| Consolidação regressiva | força queda de Recall Rate | rollback automático à versão anterior; evento `rolledback`. | F6, NFR-011 |
| Flood de escrita | supera `max_inflight` (1000) | backpressure `AIOS-MEM-0011`; poda disparada; sem *crash*. | F7 |
| Poison message | evento malformado | move p/ DLQ JetStream; alerta; serviço estável. | F10 |

---

## 7. Testes de carga

Delegados a [Benchmark.md](./Benchmark.md) (metodologia, cargas, KPIs). O gate de release
DEVE verificar, no mínimo:

| Alvo | Fonte |
|------|-------|
| p99 `remember` ≤ 30 ms (inline) / ≤ 120 ms (embedding). | NFR-001 |
| p99 `recall` ≤ 50/150/250 ms por `mode`. | NFR-003 |
| ≥ 5.000 `remember`/s e ≥ 2.000 `recall`/s por shard. | NFR-004 |
| ≥ 10^8 vetores/tenant com p99 ANN ≤ 150 ms. | NFR-012 |

---

## 8. Fixtures e dados de teste

| Fixture | Descrição |
|---------|-----------|
| `TenantSeed` | 3 tenants isolados (`acme`, `globex`, `initech`) para testes de RLS/NFR-010. |
| `EmbeddingStub` | Model Router (017) determinístico: vetor pseudo-aleatório estável por `content_hash` (reprodutível). |
| `GroundTruthCorpus` | Corpus rotulado (consulta→itens relevantes) para medir `aios_memory_recall_rate`/Recall@k. |
| `PolicyStub` | PDP (022) configurável (`allow`/`deny`) para testar `MemoryPep` e default deny. |
| `LayerConfigMatrix` | Combinações de `MemoryLayerConfig` (TTL, HNSW `m`/`ef`, cotas) por camada. |
| `BlobFactory` | Gera conteúdos acima/abaixo de `inline_max_bytes` para exercitar externalização. |

Todo dado de teste com aparência de PII DEVE ser sintético; nenhum dado real de tenant é
usado. Embeddings de teste são reprodutíveis (seed fixa) para estabilidade do bench.

---

## 9. Ambientes

| Ambiente | Uso | Backends |
|----------|-----|----------|
| `local` | unit + integration (Testcontainers). | contêineres efêmeros. |
| `ci` | unit + integration + contract em cada PR. | contêineres efêmeros no runner. |
| `staging-iso` | e2e + chaos + DR isolados. | cluster dedicado, dados sintéticos. |
| `perf` | load/benchmark reprodutível. | espelha topologia de produção (ver [Benchmark.md](./Benchmark.md)). |

---

## 10. Gates de qualidade (Definition of Done de teste)

Um PR só é mergeável quando **todos** os gates passam:

| Gate | Critério | Bloqueia merge? |
|------|----------|-----------------|
| Cobertura de linha (unit+integration) | ≥ **85%** global; ≥ **95%** em `RecallEngine`, `ForgettingEngine`, `ConsolidationEngine`, FSMs. | DEVE |
| Cobertura de mutação (componentes críticos) | ≥ **70%** (Stryker.NET) nas FSMs e guardas de segurança. | DEVE |
| Contract tests | 100% verdes; nenhum *breaking change* não versionado (RFC-0001 §5.7). | DEVE |
| Rastreabilidade | Todo `FR/NFR` com ≥ 1 teste ligado (matriz em [Requirements.md](./Requirements.md)). | DEVE |
| RLS/isolamento | Suite NFR-010 (fuzz de tenant) 100% verde. | DEVE |
| Segurança | `MemoryPep` default-deny + STRIDE (§12.2 brief) cobertos; SAST sem *high*. | DEVE |
| Load smoke | p99 dentro do SLO em carga reduzida (gate rápido do bench). | DEVERIA |
| Chaos | Suite de caos mensal 100% verde antes de GA. | DEVE (pré-GA) |
| Lint/build | build limpo; analisadores sem *warning* elevado a erro. | DEVE |

Exemplo de matriz de rastreabilidade (extrato):

| Requisito | Nível(is) de teste |
|-----------|--------------------|
| FR-003 (ANN) | Unit (fusão) + Integration (HNSW) + Load (NFR-012). |
| FR-007 (rollback) | Unit (versão) + E2E (regressão) + Chaos (F6). |
| FR-011 (RTBF) | Integration (purge) + E2E (evento+auditoria) + Contract (025). |
| NFR-002 (Recall Rate) | Integration (Recall@k) + Load/Bench (ground truth). |
| NFR-010 (isolamento) | Integration (RLS) + fuzz de tenant. |

## 11. Riscos e alternativas

- **Flakiness de índice vetorial** (HNSW é aproximado). Mitigação: embeddings de teste
  reprodutíveis (seed) e tolerância estatística no `Recall@k` (≥ 0,95, não igualdade).
- **Custo de e2e/chaos.** Mitigação: agendar chaos mensal/pré-GA; e2e por merge com ambiente
  efêmero de vida curta.
- **Alternativa descartada:** testar só com *mocks* de backend — rejeitada por não cobrir
  RLS, HNSW e AGE (defeitos que só aparecem com backend real → Testcontainers).

Ver também: [Benchmark.md](./Benchmark.md), [Metrics.md](./Metrics.md),
[FailureRecovery.md](./FailureRecovery.md), [Requirements.md](./Requirements.md).
