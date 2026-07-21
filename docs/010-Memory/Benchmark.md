---
Documento: Benchmark
Módulo: 010-Memory
Status: Draft
Versão: 0.1
Última atualização: 2026-07-20
Responsável (RACI-A): Arquiteto do Módulo 010-Memory
ADRs relacionados: ADR-0101, ADR-0104, ADR-0105, ADR-0108
RFCs relacionados: RFC-0001, RFC-0100
Depende de: 001-Architecture, 003-RFC (RFC-0001), 005-Database, 017-Model-Router, 024-Observability, 026-Cost-Optimizer, 040-Glossary
---

# 010-Memory — Benchmark (Metodologia e Resultados-Alvo)

Este documento define a **metodologia de benchmark** do `MemoryService`: cargas, ambiente,
KPIs, **resultados-alvo** (idênticos aos SLO/NFR da §7.2 do `_DESIGN_BRIEF.md`), baseline e
**reprodutibilidade**. Os KPIs medem as mesmas métricas do catálogo canônico
[Metrics.md](./Metrics.md) (`aios_memory_*`) e alimentam os SLO de [Monitoring.md](./Monitoring.md).

Palavras normativas conforme RFC 2119/8174. Nenhum contrato central é redefinido aqui
(reutiliza [RFC-0001](../003-RFC/RFC-0001-Architecture-Baseline.md)).

---

## 1. Objetivo e princípios

O benchmark DEVE responder, de forma reprodutível, se o módulo cumpre os NFR de desempenho
(NFR-001, NFR-002, NFR-003, NFR-004, NFR-012). Princípios:

1. **Reprodutível:** ambiente, dataset e carga versionados; execução automatizada; *seed* fixa.
2. **Isolado:** ambiente `perf` dedicado, sem *noisy neighbors*; um shard por vez, depois escala.
3. **Comparável:** todo resultado é comparado ao **baseline** registrado (§7); regressão
   > 5% em qualquer KPI de latência ou > 3% no Recall Rate bloqueia release.
4. **Honesto:** medir percentis (p50/p95/p99), nunca só a média; incluir *warm-up* e descartar
   as primeiras N amostras; reportar intervalo de confiança.

---

## 2. Ambiente de referência (SUT)

```
┌──────────────── Ambiente perf (1 shard de referência) ────────────────┐
│  MemoryService (.NET 10)   2 réplicas   4 vCPU / 8 GiB cada            │
│  PostgreSQL 16 + pgvector + Apache AGE  8 vCPU / 32 GiB / NVMe SSD     │
│      └ 1 primário + 1 réplica de leitura (streaming)                  │
│  Redis 7 (Working/hot + idempotência)   2 vCPU / 8 GiB                 │
│  MinIO (blobs/snapshots)                4 vCPU / 16 GiB / SSD          │
│  NATS JetStream                         2 vCPU / 4 GiB                 │
│  Model Router stub (017)  latência fixa p/ embedding: 60 ms ± 5 ms     │
│  Gerador de carga (k6/NBomber)          fora do SUT, rede 10 GbE       │
└───────────────────────────────────────────────────────────────────────┘
```

| Parâmetro | Valor de referência | Origem |
|-----------|---------------------|--------|
| `memory.embedding.dim` | 1024 | §8 brief |
| `memory.hnsw.m` | 16 | §8 brief |
| `memory.hnsw.ef_construction` | 128 | §8 brief |
| `memory.hnsw.ef_search` | 64 | §8 brief |
| `memory.item.inline_max_bytes` | 65536 | §8 brief |
| `memory.recall.default_k` | 10 | §8 brief |
| `memory.backpressure.max_inflight` | 1000 | §8 brief |

O ambiente `perf` DEVE espelhar a topologia de produção (mesma versão de PG/pgvector/AGE)
para que os números sejam extrapoláveis. Divergências DEVEM ser registradas no relatório.

---

## 3. Datasets

| Dataset | Tamanho | Uso |
|---------|---------|-----|
| `DS-SMALL` | 10^6 itens / tenant | latência a frio e a quente. |
| `DS-LARGE` | **10^8 vetores / tenant** | escala vetorial (NFR-012). |
| `DS-EPISODIC` | 10^7 itens particionados por mês | recall temporal/Episodic. |
| `DS-GRAPH` | 10^6 nós / 5·10^6 arestas (AGE) | recall `mode=graph`. |
| `GroundTruthCorpus` | 10^4 consultas rotuladas (consulta→itens relevantes) | Recall Rate / Recall@k (NFR-002). |

Embeddings são gerados por *stub* determinístico (seed por `content_hash`), garantindo o
mesmo dataset entre execuções. Nenhum dado real de tenant é usado (dados sintéticos).

---

## 4. Perfis de carga

| Perfil | Mix | Concorrência | Duração | Objetivo |
|--------|-----|--------------|---------|----------|
| **L1 — Remember sustentado** | 100% `remember` (inline) | rampa até saturação | 15 min | NFR-001 (inline), NFR-004 (≥ 5.000/s) |
| **L2 — Remember c/ embedding** | 100% `remember` (com embedding) | rampa | 15 min | NFR-001 (≤ 120 ms) |
| **L3 — Recall vetorial** | 100% `recall mode=vector` sobre `DS-LARGE` | rampa até 2.000/s | 15 min | NFR-003 (≤ 150 ms), NFR-004, NFR-012 |
| **L4 — Recall híbrido** | `recall mode=hybrid` | rampa | 15 min | NFR-003 (≤ 150 ms) |
| **L5 — Recall grafo** | `recall mode=graph` sobre `DS-GRAPH` | rampa | 15 min | NFR-003 (≤ 250 ms) |
| **L6 — Mix realista** | 60% recall / 35% remember / 5% consolidate | estável no SLO | 60 min | comportamento sob mix produtivo |
| **L7 — Soak** | mix L6 | carga nominal | 24 h | vazamentos, decay, crescimento de cota |
| **L8 — Stress/backpressure** | `remember` acima de `max_inflight` | além de 5.000/s | 10 min | F7 (backpressure `AIOS-MEM-0011`) |

Cada perfil aplica *warm-up* de 2 min (descarta amostras) antes da janela de medição.

---

## 5. KPIs e resultados-alvo

Os alvos são os **mesmos** dos NFR (§7.2 brief) e SLO ([Monitoring.md](./Monitoring.md) §2).

| KPI | Métrica ([Metrics.md](./Metrics.md)) | Alvo | NFR/SLO |
|-----|--------------------------------------|------|---------|
| Latência `remember` inline (p99) | `aios_memory_remember_duration_ms{embedding="false"}` | ≤ **30 ms** | NFR-001 / SLO-1 |
| Latência `remember` c/ embedding (p99) | `aios_memory_remember_duration_ms{embedding="true"}` | ≤ **120 ms** | NFR-001 / SLO-2 |
| Latência `recall` hot/vector (p99) | `aios_memory_recall_duration_ms{mode="vector"}` | ≤ **50 ms** | NFR-003 / SLO-3 |
| Latência `recall` hybrid (p99) | `aios_memory_recall_duration_ms{mode="hybrid"}` | ≤ **150 ms** | NFR-003 / SLO-4 |
| Latência `recall` graph (p99) | `aios_memory_recall_duration_ms{mode="graph"}` | ≤ **250 ms** | NFR-003 / SLO-5 |
| Latência ANN isolada (p99) | `aios_memory_ann_search_duration_ms` | ≤ **150 ms** | NFR-012 / SLO-9 |
| Throughput `remember` | `rate(aios_memory_remember_total)` | ≥ **5.000/s** por shard | NFR-004 |
| Throughput `recall` | `rate(aios_memory_recall_total)` | ≥ **2.000/s** por shard | NFR-004 |
| **Memory Recall Rate** | `aios_memory_recall_rate{source="bench"}` | ≥ **0,95** | NFR-002 / SLO-6 |
| Recall@10 ANN vs. exata | `aios_memory_recall_at_k_ratio{k="10"}` | ≥ **0,95** | NFR-002 |
| Escala vetorial | `aios_memory_vectors` sustentado | ≥ **10^8/tenant** | NFR-012 |
| Δ Recall pós-consolidação | `aios_memory_consolidation_recall_delta_ratio` | ≥ **−0,01** | NFR-011 / SLO-10 |
| Usage ratio sob soak | `aios_memory_usage_ratio` | < **1,0** | NFR-009 / SLO-8 |

---

## 6. Procedimento de execução (reprodutível)

```bash
# 1) Provisiona ambiente perf com config de referência (§2)
make perf-env-up  CONFIG=bench/reference.yaml

# 2) Semeia datasets determinísticos (seed fixa)
memctl bench seed --dataset DS-LARGE --tenant acme --seed 42

# 3) Executa um perfil (ex.: L3 — recall vetorial), coletando métricas OTel
k6 run bench/profiles/L3_recall_vector.js \
   --out experimental-prometheus-rw \
   --summary-export=results/L3_$(git rev-parse --short HEAD).json

# 4) Mede Recall Rate offline contra ground truth
memctl bench recall-rate --corpus GroundTruthCorpus --k 10 \
   --out results/recall_rate.json

# 5) Compara ao baseline e aplica gate de regressão
memctl bench compare --baseline baselines/v0.1.json \
   --current results/ --max-latency-regression 0.05 --max-recall-drop 0.03
```

Toda execução DEVE registrar: SHA do commit, versão de imagens, config aplicada, dataset e
seed. O relatório é anexado ao release (rastreabilidade). Métricas fluem para
[024-Observability](../024-Observability/) via OTel (mesmo pipeline de produção).

---

## 7. Baseline e comparação

O **baseline** é o conjunto de resultados aprovado de uma versão de referência; toda
execução é comparada a ele. Exemplo de baseline `v0.1` (valores-alvo; a preencher com a
primeira medição real de referência):

| KPI | Baseline v0.1 (alvo) | Regressão que bloqueia |
|-----|----------------------|------------------------|
| `remember` inline p99 | 22 ms (≤ 30) | > +5% ou > 30 ms |
| `remember` embedding p99 | 98 ms (≤ 120) | > +5% ou > 120 ms |
| `recall` vector p99 | 41 ms (≤ 50) | > +5% ou > 50 ms |
| `recall` hybrid p99 | 128 ms (≤ 150) | > +5% ou > 150 ms |
| `recall` graph p99 | 210 ms (≤ 250) | > +5% ou > 250 ms |
| throughput `remember`/shard | 5.400/s (≥ 5.000) | < 5.000/s |
| throughput `recall`/shard | 2.300/s (≥ 2.000) | < 2.000/s |
| Memory Recall Rate | 0,962 (≥ 0,95) | queda > 3% ou < 0,95 |
| Recall@10 | 0,958 (≥ 0,95) | < 0,95 |

> Os números da coluna "Baseline v0.1 (alvo)" são estimativas de projeto coerentes com os
> SLO; a **primeira execução de referência** os substitui pelos valores medidos, que passam
> a ser o baseline oficial. Nenhum valor pode ficar **pior** que o SLO do brief.

Regra de gate (idêntica a [Testing.md](./Testing.md) §7): regressão de latência > 5%,
queda de Recall Rate > 3%, ou qualquer KPI abaixo do SLO ⇒ release bloqueado.

---

## 8. Sensibilidade e tuning (experimentos auxiliares)

Experimentos que informam ADR-0101/0104/0108 (parâmetros de índice e fusão):

| Experimento | Variável | Efeito medido |
|-------------|----------|---------------|
| Varredura de `ef_search` | 16 → 512 | trade-off latência ANN × Recall@10 (calibra `memory.hnsw.ef_search`). |
| Varredura de `m` / `ef_construction` | 8–64 / 32–512 | custo de build × qualidade (requer reindex — não recarregável). |
| Fusão `rrf` vs. `weighted` | `memory.recall.fusion` | impacto no Recall Rate e latência (ADR-0104). |
| Partição Episodic | mensal vs. semanal | latência de recall temporal × custo de manutenção. |
| Sharding | N shards | linearidade do throughput (`hash(tenant,agent) mod N`). |

Exemplo de resultado de sensibilidade (`ef_search`):

```
ef_search |  p99 ANN (ms) | Recall@10
   16      |      38       |   0.91
   64      |      92       |   0.958   <- default de referência
  128      |     140       |   0.972
  256      |     210       |   0.981   (excede SLO de 150 ms -> não recomendado default)
```

## 9. Reprodutibilidade e integridade

- **Determinismo:** embeddings por *stub* com seed fixa; datasets versionados; sem relógio
  de parede em decisões de teste.
- **Ambiente pinado:** imagens por *digest*; config de referência versionada em `bench/`.
- **Isolamento:** um experimento por vez no ambiente `perf`; medir só após *warm-up*.
- **Auditável:** cada relatório carrega SHA, config, dataset, seed e ambiente — anexado ao
  release para rastreabilidade (RFC-0001 §5.8).

## 10. Riscos e alternativas

- **Stub de embedding ≠ modelo real.** O tempo real do Model Router (017) varia; por isso L2
  fixa a latência do stub (60 ms) e mede a **parcela do MemoryService**; a latência real de
  embedding é validada separadamente com 017.
- **Divergência perf↔prod.** Mitigação: espelhar topologia/versões e registrar desvios.
- **Alternativa descartada:** medir apenas média — rejeitada por mascarar cauda; usar
  p50/p95/p99 e intervalo de confiança.

Ver também: [Metrics.md](./Metrics.md), [Monitoring.md](./Monitoring.md),
[Testing.md](./Testing.md), [NonFunctionalRequirements.md](./NonFunctionalRequirements.md),
[Scalability.md](./Scalability.md).
