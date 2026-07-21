---
Documento: Metrics
Módulo: 011-Context
Status: Draft
Versão: 0.1
Última atualização: 2026-07-20
Responsável (RACI-A): Arquiteto do Módulo 011-Context
ADRs relacionados: ADR-0110, ADR-0111, ADR-0112, ADR-0113, ADR-0114
RFCs relacionados: RFC-0001, RFC-0011
Depende de: 001-Architecture, 003-RFC (RFC-0001), 017-Model-Router, 024-Observability, 026-Cost-Optimizer, 040-Glossary
---

# AIOS — Módulo 011 · Context — Metrics (Catálogo de Métricas)

Este documento define o **catálogo canônico de métricas** do `ContextService`, exportadas
via **OpenTelemetry** (OTel Metrics API) e coletadas por **Prometheus** (via OTel Collector,
ver `../024-Observability/`). Todo nome DEVE seguir o padrão `aios_context_<nome>_<unidade>`.
Os valores-alvo (SLO) referenciados aqui são os **mesmos** declarados na §7.2 do
[`_DESIGN_BRIEF.md`](./_DESIGN_BRIEF.md) e em
[NonFunctionalRequirements.md](./NonFunctionalRequirements.md) — divergência é defeito.

Palavras normativas conforme RFC 2119/8174. Termos reutilizam
[Glossary](../040-Glossary/Glossary.md) (**Context Compression Ratio**, **Semantic Cache**,
**Context Window**); contratos de correlação reutilizam
[RFC-0001 §5.6](../003-RFC/RFC-0001-Architecture-Baseline.md) — **não são redefinidos aqui**.

---

## 1. Convenções

| Aspecto | Regra |
|---------|-------|
| Prefixo | Toda métrica DEVE começar com `aios_context_`. |
| Sufixo de unidade | O nome DEVE terminar com a unidade: `_ms`, `_bytes`, `_ratio`, `_total` (counters), `_usd` (custo), ou o substantivo contável (`_entries`, `_tokens`) para gauges/counters de contagem. |
| Tipos | `counter` (monotônico, sufixo `_total`), `gauge` (instantâneo), `histogram` (distribuição, buckets explícitos). |
| Labels obrigatórios | Toda série DEVE ter `tenant`; onde aplicável, `model_id`, `path` e `cache_outcome`. Cardinalidade de `tenant` é limitada por *recording rules* de agregação (§6). |
| Cardinalidade | Labels NÃO DEVEM incluir `agent_urn`, `bundle_id`, `fragment_id` nem `cache_id` crus (explosão de cardinalidade). Correlação por identificador é feita via traces/logs, não via labels de métrica. |
| Buckets de latência (ms) | `2, 5, 10, 20, 30, 50, 75, 100, 150, 250, 400, 600, 900, 1200, 2000, 4000` (cobre os SLO de 20/150/1200 ms). |
| Exemplars | Histogramas de latência DEVEM anexar *exemplars* com `trace_id` (RFC-0001 §5.6) para correlação com traces. |
| Escopo | `resource.attributes`: `service.name=aios-context`, `service.version`, `deployment.environment`. |

Rótulo `result` assume `ok \| error`; quando `error`, a série é complementada por
`aios_context_errors_total{code}` com o código `AIOS-CTX-NNNN` correspondente (ver
[API.md](./API.md), brief §5.3).

---

## 2. Catálogo — Latência e Vazão (golden signals: latency, traffic)

| Métrica | Tipo | Unidade | Labels | Semântica | SLO/NFR ligado |
|---------|------|---------|--------|-----------|----------------|
| `aios_context_assemble_duration_ms` | histogram | ms | `tenant`, `path` (`cache_hit\|miss_no_summarize\|miss_summarize\|degraded`) | Duração de `Assemble()` fim-a-fim, do `RECEIVED` ao `SERVED`/`REJECTED`. | NFR-002 (p99 ≤ 150 ms `miss_no_summarize`), NFR-003 (p99 ≤ 1200 ms `miss_summarize`) |
| `aios_context_cache_lookup_duration_ms` | histogram | ms | `tenant`, `backend` (`l1_redis\|l2_pgvector`) | Duração isolada do `CacheLookup` (L1 e, se necessário, L2). | NFR-001 (p99 ≤ 20 ms) |
| `aios_context_retrieval_duration_ms` | histogram | ms | `tenant` | Duração do recall seletivo ao `010-Memory` (`SelectiveRetriever`). | NFR-002/003 (parcela) |
| `aios_context_compress_duration_ms` | histogram | ms | `tenant`, `method` (`hierarchical\|extractive\|truncate`) | Duração do `HierarchicalCompressor` (inclui chamada de sumarização quando aplicável). | NFR-003 (parcela) |
| `aios_context_token_count_duration_ms` | histogram | ms | `tenant`, `model_id` | Latência de contagem/estimativa de tokens (`TokenCounter`). | NFR-010 (parcela) |
| `aios_context_embedding_duration_ms` | histogram | ms | `tenant`, `result` | Latência da chamada ao Model Router (017) para embedding. | NFR-001/002 (parcela) |
| `aios_context_assemble_total` | counter | total | `tenant`, `model_id`, `outcome` (`served\|rejected\|failed\|degraded`) | Contagem de operações `assemble` concluídas por desfecho. | NFR-009 (base de throughput) |
| `aios_context_cache_lookup_total` | counter | total | `tenant`, `model_id`, `outcome` (`hit\|miss\|bypass`) | Contagem de operações de lookup de cache. | NFR-006 (base do hit ratio) |
| `aios_context_compress_total` | counter | total | `tenant`, `method`, `result` | Contagem de invocações de compressão. | FR-003 |

Throughput é derivado por `rate(aios_context_assemble_total[1m])` /
`rate(aios_context_cache_lookup_total[1m])`.

---

## 3. Catálogo — Compressão e Cache Semântico (golden signal de negócio)

| Métrica | Tipo | Unidade | Labels | Semântica | SLO/NFR ligado |
|---------|------|---------|--------|-----------|----------------|
| `aios_context_compression_ratio` | histogram | ratio [0..1] | `tenant`, `method` | **Context Compression Ratio** por bundle: `1 − tokens_in/tokens_raw` (idêntico ao campo `compression_ratio` de `ContextBundle`). Métrica-chave do módulo (R-03). | **NFR-004 (média ≥ 0,50)** |
| `aios_context_cache_hit_total` | counter | total | `tenant`, `model_id`, `cache_scope` (`tenant\|agent\|task_type`) | *Hits* de cache semântico (fast-path `prompt_hash` ou slow-path `prompt_embedding`). | NFR-006 |
| `aios_context_cache_miss_total` | counter | total | `tenant`, `model_id`, `reason` | *Misses* de cache semântico. | NFR-006 |
| `aios_context_cache_similarity` | histogram | ratio [0..1] | `tenant`, `outcome` (`hit\|miss`) | Distribuição do score de similaridade observado no lookup (saúde do `similarity_threshold`). | ADR-0112 |
| `aios_context_cache_tokens_saved_total` | counter | tokens | `tenant`, `model_id` | Tokens evitados por *hits* de cache (soma de `tokens_saved`). | NFR-007 |
| `aios_context_cache_cost_saved_usd` | counter | usd | `tenant`, `model_id` | Custo estimado evitado por *hits* de cache (soma de `cost_saved_usd`). | **NFR-007 (≥ 0,25 do custo bruto)** |
| `aios_context_cache_false_hit_ratio` | gauge | ratio [0..1] | `tenant`, `source` (`bench\|prod_sample`) | Fração de *hits* avaliados como semanticamente inadequados (amostragem/avaliação offline). | **NFR-011 (≤ 0,005)** |
| `aios_context_dedup_collapsed_total` | counter | total | `tenant` | Fragmentos colapsados pelo `RedundancyEliminator` (`cosine ≥ context.dedup.cosine_threshold`). | FR-004 |
| `aios_context_task_completion_delta_ratio` | gauge | ratio [-1..1] | `tenant`, `benchmark_run_id` | Δ Task Completion Rate vs. baseline sem compressão (medido em `035-Benchmark`, apenas exposto aqui). | NFR-005 (≥ −0,02) |

> `aios_context_compression_ratio` e `aios_context_task_completion_delta_ratio` DEVEM ser
> reportadas **juntas** em qualquer painel/relatório de release — otimizar compressão sem
> vigiar a qualidade da tarefa é defeito de release (gate de qualidade, [Testing.md](./Testing.md)).
> `aios_context_cache_hit_ratio` é uma métrica **derivada** (não emitida diretamente),
> calculada por *recording rule* (§6.1) a partir de `cache_hit_total`/`cache_miss_total`.

---

## 4. Catálogo — Saturação, Cotas e Orçamento (golden signal: saturation)

| Métrica | Tipo | Unidade | Labels | Semântica | SLO/NFR ligado |
|---------|------|---------|--------|-----------|----------------|
| `aios_context_token_budget_utilization_ratio` | histogram | ratio [0..1+] | `tenant`, `section` (`system\|instruction\|memory\|history\|tools`) | `tokens_final_seção / (alloc_seção × token_budget)`; DEVE permanecer ≤ 1,0 em bundles `SERVED`. | FR-002 |
| `aios_context_cache_entries` | gauge | entries | `tenant`, `layer` (`l1_redis\|l2_pgvector`) | Nº de entradas de cache correntes por camada e tenant. | NFR-009 |
| `aios_context_cache_max_entries` | gauge | entries | `tenant` | Cota configurada (`context.cache.max_entries_per_tenant`), para cálculo de utilização. | NFR-009 |
| `aios_context_retrieval_topk_truncated_total` | counter | total | `tenant` | Recalls em que candidatos excederam `context.retrieval.top_k` e foram truncados antes do ranking. | FR-005 |
| `aios_context_budget_infeasible_total` | counter | total | `tenant`, `model_id` | Rejeições por orçamento infeasível (`AIOS-CTX-0001`). Espelha `aios_context_errors_total{code="AIOS-CTX-0001"}`. | FR-001, RB-05 |
| `aios_context_ratelimit_rejected_total` | counter | total | `tenant` | Rejeições por limite de taxa (`AIOS-CTX-0015`). | NFR-009 |
| `aios_context_inflight` | gauge | requests | (global) | Requisições `assemble`/`cache lookup` em voo. | NFR-009 |
| `aios_context_degraded_total` | counter | total | `tenant`, `reason` (`memory_timeout\|model_router_down`) | Bundles montados em estado `DEGRADED` (contexto parcial/limites cacheados). | FM-01, FM-02 |
| `aios_context_summarization_fallback_total` | counter | total | `tenant` | Fallback de truncamento determinístico acionado por falha de sumarização (`AIOS-CTX-0004`). | FM-04 |

---

## 5. Catálogo — Erros, Eventos e Backends (golden signal: errors)

| Métrica | Tipo | Unidade | Labels | Semântica | SLO/NFR ligado |
|---------|------|---------|--------|-----------|----------------|
| `aios_context_errors_total` | counter | total | `code` (`AIOS-CTX-NNNN`), `operation`, `retriable` | Erros por código canônico e operação. Base do *error budget*. | NFR-008 |
| `aios_context_token_estimate_error_ratio` | gauge | ratio [0..1] | `tenant`, `model_id` | Erro relativo entre contagem estimada e contagem exata do provedor (amostragem). | **NFR-010 (≤ 0,02)** |
| `aios_context_events_published_total` | counter | total | `subject`, `result` | Eventos de domínio publicados (§6.1 do brief). | R-08 |
| `aios_context_outbox_pending` | gauge | events | (global) | Linhas de outbox em `pending` (lag de publicação). | FM-07 |
| `aios_context_outbox_publish_lag_ms` | histogram | ms | (global) | Tempo entre commit no DB e publicação no NATS. | FM-07 |
| `aios_context_dlq_messages_total` | counter | total | `subject` | Mensagens movidas para DLQ JetStream (evento poison). | FM-07 |
| `aios_context_backend_up` | gauge | bool [0\|1] | `backend` (`redis\|postgres\|pgvector\|minio\|nats\|model_router\|memory`) | Saúde de cada dependência. | FM-01..FM-06, NFR-008 |
| `aios_context_backend_call_duration_ms` | histogram | ms | `backend`, `op` | Latência de chamadas aos backends (diagnóstico). | NFR-001/002/003 |
| `aios_context_cache_evicted_total` | counter | total | `tenant`, `cause` (`ttl\|lru\|event\|manual`) | Entradas de cache removidas pelo `EvictionManager`. | ADR-0116 |
| `aios_context_cache_invalidated_total` | counter | total | `tenant`, `source_domain` (`memory\|model\|knowledge\|policy`) | Invalidações *event-driven* por mudança de origem. | FR-008 |

---

## 6. Recording rules de agregação (reduzem cardinalidade de `tenant`)

```yaml
groups:
  - name: aios_context_aggregations
    interval: 30s
    rules:
      # Latência p99 de cache lookup por backend (todos os tenants)
      - record: aios_context:cache_lookup_duration_ms:p99
        expr: histogram_quantile(0.99, sum by (le, backend) (rate(aios_context_cache_lookup_duration_ms_bucket[5m])))
      # Latência p99 de assemble por caminho
      - record: aios_context:assemble_duration_ms:p99
        expr: histogram_quantile(0.99, sum by (le, path) (rate(aios_context_assemble_duration_ms_bucket[5m])))
      # Context Compression Ratio médio diário
      - record: aios_context_compression_ratio:avg1d
        expr: avg_over_time(aios_context_compression_ratio_sum[1d]) / avg_over_time(aios_context_compression_ratio_count[1d])
      # Cache hit ratio (derivado; agregação diária)
      - record: aios_context_cache_hit_ratio:avg1d
        expr: |
          sum(increase(aios_context_cache_hit_total[1d]))
          / clamp_min(sum(increase(aios_context_cache_hit_total[1d])) + sum(increase(aios_context_cache_miss_total[1d])), 1)
      # Error budget: taxa de erro retriable/5xx-equivalente
      - record: aios_context:error_ratio:rate5m
        expr: |
          sum(rate(aios_context_errors_total{retriable="true"}[5m]))
          / clamp_min(sum(rate(aios_context_assemble_total[5m])) + sum(rate(aios_context_cache_lookup_total[5m])), 1)
```

---

## 7. Exemplo — instrumentação OTel (.NET 10)

```csharp
// ContextTelemetry.cs — instrumentação canônica (nomes = catálogo acima)
private static readonly Meter Meter = new("aios.context", "0.1");

private static readonly Histogram<double> AssembleDurationMs =
    Meter.CreateHistogram<double>("aios_context_assemble_duration_ms", unit: "ms");
private static readonly Counter<long> AssembleTotal =
    Meter.CreateCounter<long>("aios_context_assemble_total", unit: "1");
private static readonly Histogram<double> CompressionRatio =
    Meter.CreateHistogram<double>("aios_context_compression_ratio", unit: "1");
private static readonly Counter<long> CacheHitTotal =
    Meter.CreateCounter<long>("aios_context_cache_hit_total", unit: "1");

public async Task<ContextBundle> AssembleAsync(AssembleRequest req, CancellationToken ct)
{
    var sw = Stopwatch.StartNew();
    var tags = new TagList { { "tenant", req.TenantId }, { "model_id", req.ModelId } };
    string path = "miss_no_summarize";
    try
    {
        var bundle = await _assembler.RunAsync(req, ct);
        path = bundle.CacheOutcome == "HIT" ? "cache_hit"
             : bundle.UsedSummarization ? "miss_summarize"
             : bundle.Status == "DEGRADED" ? "degraded" : "miss_no_summarize";

        AssembleTotal.Add(1, new TagList(tags) { { "outcome", bundle.Status.ToLower() } });
        CompressionRatio.Record(bundle.CompressionRatio, new TagList(tags) { { "method", bundle.CompressionMethod } });
        if (bundle.CacheOutcome == "HIT")
            CacheHitTotal.Add(1, new TagList(tags) { { "cache_scope", bundle.CacheScope } });

        return bundle;
    }
    catch (ContextException ex)
    {
        ContextErrors.Add(1, new("code", ex.Code), new("operation", "assemble"),
                              new("retriable", ex.Retriable));
        throw;
    }
    finally
    {
        // exemplar com trace_id anexado automaticamente pelo SDK
        AssembleDurationMs.Record(sw.Elapsed.TotalMilliseconds, new TagList(tags) { { "path", path } });
    }
}
```

Exemplo de exposição Prometheus (scrape `/metrics`):

```
# HELP aios_context_assemble_duration_ms Duração de assemble() fim-a-fim
# TYPE aios_context_assemble_duration_ms histogram
aios_context_assemble_duration_ms_bucket{tenant="acme",path="miss_summarize",le="1200"} 40122
aios_context_assemble_duration_ms_sum{tenant="acme",path="miss_summarize"} 2.9e7
aios_context_assemble_duration_ms_count{tenant="acme",path="miss_summarize"} 41200
# TYPE aios_context_compression_ratio histogram
aios_context_compression_ratio_sum{tenant="acme",method="hierarchical"} 21504.3
aios_context_compression_ratio_count{tenant="acme",method="hierarchical"} 39872
# TYPE aios_context_cache_hit_total counter
aios_context_cache_hit_total{tenant="acme",model_id="gpt-router-a",cache_scope="agent"} 18321
```

---

## 8. Rastreabilidade métrica → NFR

| NFR | Métrica(s) primária(s) |
|-----|------------------------|
| NFR-001 | `aios_context_cache_lookup_duration_ms` |
| NFR-002 | `aios_context_assemble_duration_ms{path="miss_no_summarize"}` |
| NFR-003 | `aios_context_assemble_duration_ms{path="miss_summarize"}`, `aios_context_compress_duration_ms` |
| NFR-004 | `aios_context_compression_ratio` |
| NFR-005 | `aios_context_task_completion_delta_ratio` |
| NFR-006 | `aios_context_cache_hit_total`, `aios_context_cache_miss_total` |
| NFR-007 | `aios_context_cache_cost_saved_usd`, `aios_context_cache_tokens_saved_total` |
| NFR-008 | `aios_context_errors_total`, `aios_context_backend_up` |
| NFR-009 | `aios_context_assemble_total`, `aios_context_inflight`, `aios_context_cache_entries` |
| NFR-010 | `aios_context_token_estimate_error_ratio` |
| NFR-011 | `aios_context_cache_false_hit_ratio` |
| NFR-012 | `aios_context_backend_up`, `aios_context_outbox_pending` |
| NFR-013 | *exemplars* com `trace_id` em todos os histogramas |
| NFR-014 | ausência de séries cruzadas de `tenant` (verificado por auditoria de RLS, não por métrica) |

---

## 9. Riscos e alternativas

- **Cardinalidade por `tenant` × `model_id`.** Risco de explosão com milhares de tenants e
  dezenas de modelos. Mitigação: *recording rules* de agregação (§6) e limite de séries por
  *metric relabeling*; métricas por `bundle_id`/`fragment_id`/`cache_id` NÃO DEVEM existir
  como labels — apenas em traces/logs.
- **Custo de exemplars.** Exemplars aumentam o payload de scrape. Mitigação: amostragem de
  exemplars (1 a cada N observações) no caminho quente (`cache lookup`).
- **Falso-*hit* difícil de medir online.** A métrica `aios_context_cache_false_hit_ratio`
  depende de amostragem/avaliação offline (LLM-juiz ou humana); o valor de produção tem
  atraso inerente. Mitigação: complementar com o bench determinístico do
  [Benchmark.md](./Benchmark.md).
- **Alternativa descartada:** métricas puramente *pull* sem OTel Collector — rejeitada por
  não correlacionar com traces/logs (ver `../024-Observability/`).

Ver também: [Monitoring.md](./Monitoring.md), [Logging.md](./Logging.md),
[Benchmark.md](./Benchmark.md), [NonFunctionalRequirements.md](./NonFunctionalRequirements.md).
