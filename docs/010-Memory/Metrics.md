---
Documento: Metrics
Módulo: 010-Memory
Status: Draft
Versão: 0.1
Última atualização: 2026-07-20
Responsável (RACI-A): Arquiteto do Módulo 010-Memory
ADRs relacionados: ADR-0101, ADR-0102, ADR-0104, ADR-0105, ADR-0108
RFCs relacionados: RFC-0001, RFC-0100
Depende de: 001-Architecture, 003-RFC (RFC-0001), 024-Observability, 026-Cost-Optimizer, 040-Glossary
---

# 010-Memory — Metrics (Catálogo de Métricas)

Este documento define o **catálogo canônico de métricas** do `MemoryService`, exportadas
via **OpenTelemetry** (OTel Metrics API) e coletadas por **Prometheus** (via OTel
Collector, ver `../024-Observability/`). Todo nome DEVE seguir o padrão
`aios_memory_<nome>_<unidade>`. Os valores-alvo (SLO) referenciados aqui são os
**mesmos** declarados na §7.2 do `_DESIGN_BRIEF.md` e em
[NonFunctionalRequirements.md](./NonFunctionalRequirements.md) — divergência é defeito.

Palavras normativas conforme RFC 2119/8174. Termos reutilizam
[Glossary](../040-Glossary/Glossary.md); contratos de correlação reutilizam
[RFC-0001 §5.6](../003-RFC/RFC-0001-Architecture-Baseline.md) — **não são redefinidos aqui**.

---

## 1. Convenções

| Aspecto | Regra |
|---------|-------|
| Prefixo | Toda métrica DEVE começar com `aios_memory_`. |
| Sufixo de unidade | O nome DEVE terminar com a unidade: `_ms`, `_bytes`, `_ratio`, `_total` (counters), ou o substantivo contável (`_items`, `_vectors`) para gauges. |
| Tipos | `counter` (monotônico, sufixo `_total`), `gauge` (instantâneo), `histogram` (distribuição, buckets explícitos). |
| Labels obrigatórios | Toda série DEVE ter `tenant`; onde aplicável, `layer` e `mode`. Cardinalidade de `tenant` é limitada por *recording rules* de agregação (§5). |
| Cardinalidade | Labels NÃO DEVEM incluir `agent_id`, `item_id` nem `session_id` crus (explosão de cardinalidade). Agregação por agente é feita em métricas de negócio amostradas. |
| Buckets de latência (ms) | `5, 10, 20, 30, 50, 75, 100, 120, 150, 200, 250, 400, 800, 1600` (cobre os SLO de 30/50/120/150/250 ms). |
| Exemplars | Histogramas de latência DEVEM anexar *exemplars* com `trace_id` (RFC-0001 §5.6) para correlação com traces. |
| Escopo | `resource.attributes`: `service.name=aios-memory`, `service.version`, `deployment.environment`. |

Rótulo `result` assume `ok \| error`; quando `error`, a série é complementada por
`aios_memory_errors_total{code}` com o código `AIOS-MEM-NNNN` correspondente
(ver [API.md](./API.md) §5.2 do brief).

---

## 2. Catálogo — Latência e Vazão (golden signals: latency, traffic)

| Métrica | Tipo | Unidade | Labels | Semântica | SLO/NFR ligado |
|---------|------|---------|--------|-----------|----------------|
| `aios_memory_remember_duration_ms` | histogram | ms | `tenant`, `layer`, `embedding` (`true\|false`) | Duração de `remember()` fim-a-fim (validação→persistência→embedding). | NFR-001 (p99 ≤ 30 ms inline; ≤ 120 ms c/ embedding) |
| `aios_memory_recall_duration_ms` | histogram | ms | `tenant`, `mode` (`vector\|lexical\|graph\|hybrid`) | Duração de `recall()` fim-a-fim por modo de busca. | NFR-003 (p99 ≤ 50/150/250 ms) |
| `aios_memory_ann_search_duration_ms` | histogram | ms | `tenant`, `layer` (`semantic\|episodic`) | Latência isolada da busca ANN pgvector/HNSW. | NFR-012 (p99 ANN ≤ 150 ms) |
| `aios_memory_consolidation_duration_ms` | histogram | ms | `tenant`, `from_layer`, `to_layer` | Duração de um `ConsolidationJob` (SNAPSHOTTING→COMMITTED). | NFR-011 |
| `aios_memory_embedding_duration_ms` | histogram | ms | `tenant`, `result` | Latência da chamada ao Model Router (017) para embedding. | NFR-001 (parcela) |
| `aios_memory_remember_total` | counter | total | `tenant`, `layer`, `kind`, `result` | Contagem de operações `remember`. Base de throughput. | NFR-004 (≥ 5.000/s por shard) |
| `aios_memory_recall_total` | counter | total | `tenant`, `mode`, `result` | Contagem de operações `recall`. Base de throughput. | NFR-004 (≥ 2.000/s por shard) |
| `aios_memory_forget_total` | counter | total | `tenant`, `strategy` (`ttl\|decay\|lru\|quota\|rtbf`), `result` | Contagem de operações de esquecimento. | FR-006 |
| `aios_memory_purge_total` | counter | total | `tenant`, `scope` (`item\|agent\|tenant`) | Expurgos físicos (RTBF) concluídos. | FR-011 |

Throughput é derivado por `rate(aios_memory_remember_total[1m])` /
`rate(aios_memory_recall_total[1m])`.

---

## 3. Catálogo — Qualidade de Recuperação (golden signal de negócio)

| Métrica | Tipo | Unidade | Labels | Semântica | SLO/NFR ligado |
|---------|------|---------|--------|-----------|----------------|
| `aios_memory_recall_rate` | gauge | ratio [0..1] | `tenant`, `source` (`bench\|prod_sample`) | **Memory Recall Rate**: fração de consultas cujo item relevante esperado apareceu no top-k. Métrica-chave do módulo (R11). | **NFR-002 (≥ 0,95)** |
| `aios_memory_recall_at_k_ratio` | gauge | ratio [0..1] | `tenant`, `k`, `layer` | Recall@k da busca ANN vs. busca exata (qualidade do índice HNSW). | NFR-002 (Recall@10 ≥ 0,95) |
| `aios_memory_recall_results_items` | histogram | items | `tenant`, `mode` | Distribuição do nº de resultados retornados por `recall`. | FR-002 |
| `aios_memory_recall_score` | histogram | ratio [0..1] | `tenant`, `mode` | Distribuição do score de relevância do top-1 (saúde do re-rank/RRF). | FR-002, ADR-0104 |
| `aios_memory_consolidation_recall_delta_ratio` | gauge | ratio [-1..1] | `tenant` | Δ do Recall Rate pós-consolidação vs. pré (dispara rollback se < −0,01). | NFR-011 |

> `aios_memory_recall_rate` DEVE ser calculada em dois regimes: (a) offline no
> [Benchmark.md](./Benchmark.md) contra *ground truth*; (b) online por amostragem em
> produção (evento `memory.item.recalled` amostrado, §6.1 do brief). O alerta de SLO
> usa o valor de produção.

---

## 4. Catálogo — Saturação, Cotas e Recursos (golden signal: saturation)

| Métrica | Tipo | Unidade | Labels | Semântica | SLO/NFR ligado |
|---------|------|---------|--------|-----------|----------------|
| `aios_memory_usage_ratio` | gauge | ratio [0..] | `tenant`, `layer`, `dimension` (`items\|bytes\|vectors`) | Uso corrente / cota. DEVE permanecer < 1,0. | NFR-009 |
| `aios_memory_items` | gauge | items | `tenant`, `layer`, `state` | Nº de itens por camada e estado da FSM (§4.1 do brief). | NFR-009 |
| `aios_memory_bytes` | gauge | bytes | `tenant`, `layer` | Bytes armazenados por camada (inline + refs). | NFR-009 |
| `aios_memory_vectors` | gauge | vectors | `tenant`, `layer` | Nº de vetores indexados (HNSW). | NFR-012 (≥ 10^8/tenant) |
| `aios_memory_inflight` | gauge | requests | (global) | Escritas em voo; comparar a `memory.backpressure.max_inflight` (1000). | NFR-004, F7 |
| `aios_memory_backpressure_active` | gauge | bool [0\|1] | `tenant`, `layer` | 1 quando admissão de escrita está represada. | F7 |
| `aios_memory_quota_exceeded_total` | counter | total | `tenant`, `layer`, `dimension` | Rejeições `AIOS-MEM-0010/0011` por estouro de cota. | NFR-009, F7 |
| `aios_memory_blob_offloaded_total` | counter | total | `tenant`, `layer` | Conteúdos externalizados para MinIO (> `inline_max_bytes`). | FR-009 |
| `aios_memory_blob_size_bytes` | histogram | bytes | `tenant` | Distribuição de tamanho de blobs externalizados. | FR-009 |

---

## 5. Catálogo — Erros, Eventos e Backends (golden signal: errors)

| Métrica | Tipo | Unidade | Labels | Semântica | SLO/NFR ligado |
|---------|------|---------|--------|-----------|----------------|
| `aios_memory_errors_total` | counter | total | `code` (`AIOS-MEM-NNNN`), `operation`, `retriable` | Erros por código canônico e operação. Base do *error budget*. | NFR-005 |
| `aios_memory_embedding_requests_total` | counter | total | `tenant`, `result` | Chamadas ao Model Router (017); `result=error` alimenta `AIOS-MEM-0030`. | FR-010, F3 |
| `aios_memory_events_published_total` | counter | total | `subject`, `result` | Eventos de domínio publicados (§6.1 do brief). | R9 |
| `aios_memory_outbox_pending` | gauge | events | (global) | Linhas `OutboxEvent` em `pending` (lag de publicação). | F8 |
| `aios_memory_outbox_publish_lag_ms` | histogram | ms | (global) | Tempo entre commit no DB e publicação no NATS. | F8 |
| `aios_memory_dlq_messages_total` | counter | total | `subject` | Mensagens movidas para DLQ JetStream (poison, F10). | F10 |
| `aios_memory_consolidation_rollback_total` | counter | total | `tenant`, `reason` (`recall_regression\|conflict\|error`) | Rollbacks de consolidação (F6). | NFR-011 |
| `aios_memory_backend_up` | gauge | bool [0\|1] | `backend` (`redis\|postgres\|pgvector\|age\|minio\|nats`) | Saúde de cada dependência de dados. | F1–F5, NFR-005 |
| `aios_memory_backend_call_duration_ms` | histogram | ms | `backend`, `op` | Latência de chamadas aos backends (diagnóstico). | NFR-001/003 |

### 5.1 Recording rules de agregação (reduzem cardinalidade de `tenant`)

```yaml
groups:
  - name: aios_memory_aggregations
    interval: 30s
    rules:
      # Latência p99 de remember por camada (todos os tenants)
      - record: aios_memory:remember_duration_ms:p99
        expr: histogram_quantile(0.99, sum by (le, layer, embedding) (rate(aios_memory_remember_duration_ms_bucket[5m])))
      # Latência p99 de recall por modo
      - record: aios_memory:recall_duration_ms:p99
        expr: histogram_quantile(0.99, sum by (le, mode) (rate(aios_memory_recall_duration_ms_bucket[5m])))
      # Error budget: taxa de erro 5xx-equivalente
      - record: aios_memory:error_ratio:rate5m
        expr: |
          sum(rate(aios_memory_errors_total{retriable="true"}[5m]))
          / clamp_min(sum(rate(aios_memory_remember_total[5m])) + sum(rate(aios_memory_recall_total[5m])), 1)
```

---

## 6. Exemplo — instrumentação OTel (.NET 10)

```csharp
// MemoryTelemetry.cs — instrumentação canônica (nomes = catálogo acima)
private static readonly Meter Meter = new("aios.memory", "0.1");

private static readonly Histogram<double> RememberDurationMs =
    Meter.CreateHistogram<double>("aios_memory_remember_duration_ms", unit: "ms");
private static readonly Counter<long> RememberTotal =
    Meter.CreateCounter<long>("aios_memory_remember_total", unit: "1");
private static readonly ObservableGauge<double> RecallRate =
    Meter.CreateObservableGauge("aios_memory_recall_rate",
        () => new Measurement<double>(_recallRateSampler.Current,
            new("tenant", tenant), new("source", "prod_sample")));

public async Task<RememberResult> RememberAsync(MemoryItem item, CancellationToken ct)
{
    var sw = Stopwatch.StartNew();
    var tags = new TagList { { "tenant", item.TenantId }, { "layer", item.Layer },
                             { "embedding", item.NeedsEmbedding } };
    try
    {
        var result = await _router.RouteAsync(item, ct);
        RememberTotal.Add(1, new TagList(tags) { { "kind", item.Kind }, { "result", "ok" } });
        return result;
    }
    catch (MemoryException ex)
    {
        RememberTotal.Add(1, new TagList(tags) { { "kind", item.Kind }, { "result", "error" } });
        MemoryErrors.Add(1, new("code", ex.Code), new("operation", "remember"),
                            new("retriable", ex.Retriable));
        throw;
    }
    finally
    {
        // exemplar com trace_id anexado automaticamente pelo SDK
        RememberDurationMs.Record(sw.Elapsed.TotalMilliseconds, tags);
    }
}
```

Exemplo de exposição Prometheus (scrape `/metrics`):

```
# HELP aios_memory_remember_duration_ms Duração de remember() fim-a-fim
# TYPE aios_memory_remember_duration_ms histogram
aios_memory_remember_duration_ms_bucket{tenant="acme",layer="semantic",embedding="true",le="120"} 48213
aios_memory_remember_duration_ms_sum{tenant="acme",layer="semantic",embedding="true"} 3.1e6
aios_memory_remember_duration_ms_count{tenant="acme",layer="semantic",embedding="true"} 49001
# TYPE aios_memory_recall_rate gauge
aios_memory_recall_rate{tenant="acme",source="prod_sample"} 0.968
# TYPE aios_memory_usage_ratio gauge
aios_memory_usage_ratio{tenant="acme",layer="semantic",dimension="vectors"} 0.72
```

---

## 7. Rastreabilidade métrica → NFR

| NFR | Métrica(s) primária(s) |
|-----|------------------------|
| NFR-001 | `aios_memory_remember_duration_ms`, `aios_memory_embedding_duration_ms` |
| NFR-002 | `aios_memory_recall_rate`, `aios_memory_recall_at_k_ratio` |
| NFR-003 | `aios_memory_recall_duration_ms`, `aios_memory_ann_search_duration_ms` |
| NFR-004 | `aios_memory_remember_total`, `aios_memory_recall_total`, `aios_memory_inflight` |
| NFR-005 | `aios_memory_errors_total`, `aios_memory_backend_up` |
| NFR-009 | `aios_memory_usage_ratio`, `aios_memory_items`, `aios_memory_bytes` |
| NFR-011 | `aios_memory_consolidation_recall_delta_ratio`, `aios_memory_consolidation_rollback_total` |
| NFR-012 | `aios_memory_vectors`, `aios_memory_ann_search_duration_ms` |
| NFR-013 | *exemplars* com `trace_id` em todos os histogramas |

---

## 8. Riscos e alternativas

- **Cardinalidade por `tenant`.** Risco de explosão com milhares de tenants. Mitigação:
  *recording rules* de agregação (§5.1) e limite de séries por *metric relabeling*;
  métricas por agente NÃO DEVEM usar `agent_id` como label — usam amostragem via evento.
- **Custo de exemplars.** Exemplars aumentam payload de scrape. Mitigação: amostragem de
  exemplars (1 a cada N observações).
- **Alternativa descartada:** métricas puramente *pull* sem OTel Collector — rejeitada por
  não correlacionar com traces/logs (ver ADR-0101 e `../024-Observability/`).

Ver também: [Monitoring.md](./Monitoring.md), [Logging.md](./Logging.md),
[Benchmark.md](./Benchmark.md), [NonFunctionalRequirements.md](./NonFunctionalRequirements.md).
