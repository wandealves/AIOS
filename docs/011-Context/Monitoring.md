---
Documento: Monitoring
Módulo: 011-Context
Status: Draft
Versão: 0.1
Última atualização: 2026-07-20
Responsável (RACI-A): Arquiteto do Módulo 011-Context
ADRs relacionados: ADR-0110, ADR-0111, ADR-0112, ADR-0113, ADR-0116, ADR-0118, ADR-0119
RFCs relacionados: RFC-0001, RFC-0011
Depende de: 001-Architecture, 003-RFC (RFC-0001), 010-Memory, 017-Model-Router, 022-Policy, 024-Observability, 025-Audit, 026-Cost-Optimizer, 040-Glossary
---

# AIOS — Módulo 011 · Context — Monitoring (Observabilidade Operacional)

Este documento define **dashboards**, **regras de alerta Prometheus**, **SLI/SLO/SLA**,
**golden signals** e **runbooks** do `ContextService`. Todos os alvos numéricos são
**idênticos** aos NFR da §7.2 do [`_DESIGN_BRIEF.md`](./_DESIGN_BRIEF.md) (ver
[NonFunctionalRequirements.md](./NonFunctionalRequirements.md)); divergência entre este
documento e o brief é considerada defeito de documentação. As métricas citadas seguem o
catálogo canônico de [Metrics.md](./Metrics.md) (`aios_context_*`).

Palavras normativas conforme RFC 2119/8174. A plataforma de coleta/painéis (Prometheus,
Grafana, Alertmanager, Seq) pertence a [024-Observability](../024-Observability/) — aqui
apenas configuramos o que é específico de Context, **sem redefinir** a plataforma.

---

## 1. Golden Signals do ContextService

| Sinal | Pergunta operacional | Métrica-fonte ([Metrics.md](./Metrics.md)) |
|-------|----------------------|--------------------------------------------|
| **Latency** | `assemble`/`cache lookup`/`compress` respondem dentro do SLO? | `aios_context_assemble_duration_ms`, `aios_context_cache_lookup_duration_ms`, `aios_context_compress_duration_ms` |
| **Traffic** | Qual a carga por operação/tenant/modelo? | `aios_context_assemble_total`, `aios_context_cache_lookup_total` |
| **Errors** | Qual a taxa/tipos de falha (`AIOS-CTX-<NNNN>`)? | `aios_context_errors_total{code}` |
| **Saturation** | Orçamento de tokens, cotas de cache e rate limit estão saturando? | `aios_context_token_budget_utilization_ratio`, `aios_context_cache_entries`, `aios_context_ratelimit_rejected_total` |
| **Qualidade (negócio)** | O módulo está comprimindo bem e evitando reinferência? | `aios_context_compression_ratio` (**Context Compression Ratio**), `aios_context_cache_hit_total`/`aios_context_cache_miss_total` (cache hit ratio) |

Os dois sinais de negócio — **Context Compression Ratio** e **cache hit ratio** — são
tratados como sinais de primeira classe, ao lado dos quatro clássicos, por serem os KPIs
que justificam a missão do módulo (brief §1.1, NFR-004/NFR-006).

---

## 2. SLI, SLO e SLA

### 2.1 Definições (SLI → SLO)

| # | SLI (como se mede) | SLO (alvo) | Janela | NFR |
|---|--------------------|-----------|--------|-----|
| SLO-1 | p99 de `aios_context_cache_lookup_duration_ms` | ≤ **20 ms** | 30 d móveis | NFR-001 |
| SLO-2 | p99 de `aios_context_assemble_duration_ms{path="miss_no_summarize"}` | ≤ **150 ms** | 30 d | NFR-002 |
| SLO-3 | p99 de `aios_context_assemble_duration_ms{path="miss_summarize"}` | ≤ **1200 ms** | 30 d | NFR-003 |
| SLO-4 | `aios_context_compression_ratio:avg1d` | ≥ **0,50** | 1 d | NFR-004 |
| SLO-5 | Δ Task Completion Rate vs. baseline sem compressão (`035-Benchmark`) | ≥ **−2%** (abs.) | por release | NFR-005 |
| SLO-6 | `aios_context_cache_hit_ratio:avg1d` (regime permanente) | ≥ **0,35** | 1 d | NFR-006 |
| SLO-7 | custo evitado / custo bruto (`aios_context_cache_cost_saved_usd` / custo estimado bruto) | ≥ **0,25** | 7 d | NFR-007 |
| SLO-8 | Disponibilidade = 1 − taxa de erro `retriable`/`5xx` | ≥ **99,95%** | 30 d | NFR-008 |
| SLO-9 | throughput de `assemble` por réplica | ≥ **500 req/s** | pico sustentado | NFR-009 |
| SLO-10 | erro de estimativa de tokens vs. contagem exata | ≤ **2%** | por release | NFR-010 |
| SLO-11 | taxa de falso-*hit* de cache (`aios_context_cache_false_hit_ratio`) | ≤ **0,5%** | 7 d (amostragem) | NFR-011 |
| SLO-12 | RTO / RPO | ≤ **5 min** / ≤ **60 s** | por *drill* | NFR-012 |
| SLO-13 | cobertura de traces OTel em caminhos críticos | **100%** | contínuo | NFR-013 |
| SLO-14 | vazamento de cache entre tenants | **0** | contínuo | NFR-014 |

### 2.2 SLA (compromisso externo)

O SLA publicado ao tenant DEVE ser **mais frouxo** que o SLO interno para preservar
margem operacional:

| Compromisso SLA | Alvo externo | SLO interno correspondente |
|-----------------|--------------|----------------------------|
| Disponibilidade do serviço de contexto | **99,9%/mês** | SLO-8 (99,95%) |
| Latência de `assemble` (miss sem sumarização, p99) | **≤ 250 ms** | SLO-2 (150 ms) |
| Latência de `assemble` (miss com sumarização, p99) | **≤ 2000 ms** | SLO-3 (1200 ms) |
| RPO | **≤ 5 min** | SLO-12 (60 s) |
| RTO | **≤ 15 min** | SLO-12 (5 min) |

### 2.3 Error budget

O *error budget* de disponibilidade (SLO-8 = 99,95%/30 d) equivale a **~21,9 min/mês** de
indisponibilidade tolerada (idêntico ao cálculo de NFR-008 do brief). Política de queima:

| Consumo do budget | Ação |
|-------------------|------|
| < 50% | Operação normal; mudanças liberadas. |
| 50–90% | *Freeze* de mudanças não essenciais; revisão de causas dominantes. |
| > 90% ou *burn rate* > 14,4× por 1 h | Página on-call (P1); congelamento total; runbook RB-09. |

Regra de *burn rate* rápido: `aios_context:error_ratio:rate5m > 0.0072` (14,4× a taxa de
budget) por 5 min → alerta crítico.

---

## 3. Regras de alerta Prometheus

As regras abaixo referenciam as *recording rules* de [Metrics.md](./Metrics.md) §6. Toda
regra crítica DEVE ter *runbook* associado (coluna `runbook`).

```yaml
groups:
  - name: aios_context_slo_alerts
    rules:
      # SLO-1 — latência de cache lookup
      - alert: ContextCacheLookupLatencyHigh
        expr: aios_context:cache_lookup_duration_ms:p99 > 20
        for: 10m
        labels: { severity: warning, module: "011-context", runbook: RB-02 }
        annotations:
          summary: "p99 CacheLookup acima de 20 ms (NFR-001)"
          description: "p99={{ $value }}ms backend={{ $labels.backend }}."

      # SLO-2/3 — latência de assemble por caminho
      - alert: ContextAssembleLatencyHigh
        expr: |
          (aios_context:assemble_duration_ms:p99{path="miss_no_summarize"} > 150)
          or (aios_context:assemble_duration_ms:p99{path="miss_summarize"} > 1200)
        for: 10m
        labels: { severity: warning, module: "011-context", runbook: RB-01 }
        annotations: { summary: "p99 assemble acima do SLO por caminho (NFR-002/003)" }

      # SLO-4 — Context Compression Ratio (negócio)
      - alert: ContextCompressionRatioBelowSLO
        expr: aios_context_compression_ratio:avg1d < 0.50
        for: 6h
        labels: { severity: warning, module: "011-context", runbook: RB-03 }
        annotations:
          summary: "Context Compression Ratio < 0,50 (NFR-004)"
          description: "ratio={{ $value }} para tenant {{ $labels.tenant }}."

      # SLO-6 — cache hit ratio (negócio)
      - alert: ContextCacheHitRatioBelowSLO
        expr: aios_context_cache_hit_ratio:avg1d < 0.35
        for: 6h
        labels: { severity: warning, module: "011-context", runbook: RB-04 }
        annotations: { summary: "Cache hit ratio < 0,35 em regime permanente (NFR-006)" }

      # SLO-8 — error budget burn (rápido)
      - alert: ContextErrorBudgetBurnFast
        expr: aios_context:error_ratio:rate5m > 0.0072
        for: 5m
        labels: { severity: critical, module: "011-context", runbook: RB-09 }
        annotations: { summary: "Queima rápida do error budget (>14,4x) — NFR-008" }

  - name: aios_context_saturation_alerts
    rules:
      # Orçamento infeasível
      - alert: ContextBudgetInfeasibleSpike
        expr: increase(aios_context_errors_total{code="AIOS-CTX-0001"}[15m]) > 20
        for: 0m
        labels: { severity: warning, module: "011-context", runbook: RB-05 }
        annotations: { summary: "Pico de BudgetInfeasible (AIOS-CTX-0001) — revisar BudgetProfile/model_id" }

      # Rate limit
      - alert: ContextRateLimitedSpike
        expr: increase(aios_context_ratelimit_rejected_total[5m]) > 100
        for: 5m
        labels: { severity: warning, module: "011-context", runbook: RB-10 }
        annotations: { summary: "Rejeições por rate limit (AIOS-CTX-0015) acima do esperado" }

      # Cache próximo da cota de tenant
      - alert: ContextCacheEntriesNearQuota
        expr: aios_context_cache_entries / on(tenant) aios_context_cache_max_entries > 0.9
        for: 15m
        labels: { severity: warning, module: "011-context", runbook: RB-04 }
        annotations: { summary: "Cache do tenant {{ $labels.tenant }} > 90% da cota (context.cache.max_entries_per_tenant)" }

      # Outbox / eventos
      - alert: ContextOutboxLag
        expr: aios_context_outbox_pending > 1000
        for: 5m
        labels: { severity: warning, module: "011-context", runbook: RB-08 }
        annotations: { summary: "Outbox pendente > 1000 — lag de publicação (FM-07)" }

  - name: aios_context_backend_alerts
    rules:
      - alert: ContextBackendDown
        expr: aios_context_backend_up == 0
        for: 1m
        labels: { severity: critical, module: "011-context", runbook: RB-06 }
        annotations: { summary: "Backend {{ $labels.backend }} indisponível (FM-01..FM-05)" }

      - alert: ContextDegradedModeActive
        expr: increase(aios_context_degraded_total[15m]) > 50
        for: 0m
        labels: { severity: warning, module: "011-context", runbook: RB-06 }
        annotations: { summary: "Aumento de bundles em DEGRADED (FM-02) — recall do 010 lento/indisponível" }

      - alert: ContextCompressionFallbackSpike
        expr: increase(aios_context_summarization_fallback_total[15m]) > 30
        for: 0m
        labels: { severity: warning, module: "011-context", runbook: RB-07 }
        annotations: { summary: "Fallback de truncamento frequente (AIOS-CTX-0004, FM-04)" }

      - alert: ContextCacheFalseHitRatioHigh
        expr: aios_context_cache_false_hit_ratio > 0.005
        for: 30m
        labels: { severity: critical, module: "011-context", runbook: RB-11 }
        annotations: { summary: "Taxa de falso-hit do cache semântico > 0,5% (NFR-011, FM-08)" }
```

---

## 4. Dashboards (Grafana)

Cada painel referencia apenas métricas de [Metrics.md](./Metrics.md). Layout canônico:

```
┌───────────────────── AIOS · Context · Visão Geral (011) ────────────────────┐
│ [SLO] Compression Ratio: 0.54 ✓  Cache Hit: 0.41 ✓  Disp.: 99.97% ✓         │
│       Budget 30d restante: 68%                                              │
├──────────────────────────────┬───────────────────────────────────────────────┤
│ Latency: cache lookup p99    │ Latency: assemble p99 por path (miss/summ.)  │
│ (linha vs. limite 20ms)      │ (linha vs. limites 150/1200ms)               │
├──────────────────────────────┼───────────────────────────────────────────────┤
│ Traffic: assemble/s          │ Errors: rate por code (AIOS-CTX-*) empilhado │
│ cache lookup/s por tenant    │ (heatmap)                                    │
├──────────────────────────────┼───────────────────────────────────────────────┤
│ Saturation: budget           │ Cache: entries/tenant vs. cota; eviction/s   │
│ utilization por seção        │ por causa (ttl/lru/event)                    │
├──────────────────────────────┼───────────────────────────────────────────────┤
│ Backends up/down (redis/pg/  │ Outbox pending + publish_lag_ms p99 + DLQ    │
│ minio/nats/017/010)          │                                               │
├──────────────────────────────┴───────────────────────────────────────────────┤
│ Negócio: Compression Ratio (histograma) · Cache Hit Ratio · Cost Saved USD  │
│ · Δ Task Completion Rate (link 035-Benchmark)                                │
└───────────────────────────────────────────────────────────────────────────────┘
```

| Dashboard | Público | Painéis-chave |
|-----------|---------|---------------|
| **Context Overview** | On-call / SRE | SLOs, golden signals, budget de erro. |
| **Context Compression & Quality** | Time do módulo / 035-Benchmark | `compression_ratio`, `cache_hit_ratio`, falso-*hit*, Δ Task Completion Rate. |
| **Context Cache Capacity** | SRE / 026-Cost-Optimizer | `cache_entries`, cota por tenant, `cost_saved_usd`, `tokens_saved`. |
| **Context Backends** | SRE / DBA | saúde e latência de Redis/PostgreSQL(+pgvector)/MinIO/NATS/017/010. |
| **Context Events/Outbox** | SRE | `events_published_total`, `outbox_pending`, `publish_lag_ms`, DLQ. |

---

## 5. Runbooks

Cada alerta crítico aponta para um runbook. Formato: **sintoma → diagnóstico → ação →
verificação**. Runbooks completos versionados em `../024-Observability/runbooks/`.

| ID | Alerta / Cenário | Diagnóstico inicial | Ação primária | Verificação |
|----|------------------|---------------------|---------------|-------------|
| **RB-01** | Latência `assemble` alta | Isolar por `path`; checar `retrieval_duration_ms` (010) e `compress_duration_ms` (sumarização via 017). | Se recall lento: cf. RB-06. Se sumarização lenta: revisar `summarization_model_hint` (roteamento a modelo barato). | p99 ≤ 150/1200 ms. |
| **RB-02** | Latência `cache lookup` alta | Isolar por `backend` (L1 Redis vs. L2 pgvector); checar HNSW `ef_search`. | Verificar saúde do Redis; ajustar índice vetorial se L2 dominante. | p99 ≤ 20 ms. |
| **RB-03** | Compression Ratio < 0,50 | Amostrar `BudgetProfile`/bundles recentes; checar `compression_method` efetivo. | Revisar `min_compression_ratio`/`max_levels`; investigar regressão no `HierarchicalCompressor`. | ratio ≥ 0,50 sem queda de Δ Task Completion Rate. |
| **RB-04** | Cache hit ratio baixo / cota próxima | Ver `similarity_threshold` e padrões de prompt por tenant; checar `cache_entries` vs. cota. | Ajustar `similarity_threshold`/`reuse_threshold`; acionar `EvictionManager` para liberar cota. | hit ratio ≥ 0,35; `cache_entries` < 90% cota. |
| **RB-05** | Pico de `BudgetInfeasible` (AIOS-CTX-0001) | Verificar `model_id`/`token_budget` vs. tamanho mínimo de `system+instruction`. | Sugerir modelo com janela maior ao `017`; revisar `BudgetProfile` do tenant/agente. | Queda de `AIOS-CTX-0001` para nível basal. |
| **RB-06** | Backend/dependência indisponível ou `DEGRADED` alto | Qual `backend`? `010`/`017` fora do deadline? | Circuit breaker isola dependência; fallback com contexto parcial (`DEGRADED`) ou limites cacheados. | `backend_up=1`; `degraded_total` volta ao basal. |
| **RB-07** | Fallback de truncamento frequente (AIOS-CTX-0004) | Checar erro do modelo de sumarização (`017`); taxa de `summarization_fallback_total`. | Trocar `summarization_model_hint`; investigar disponibilidade do modelo barato. | Fallback volta ao nível basal; Compression Ratio recupera. |
| **RB-08** | Outbox lag / DLQ crescendo | `outbox_pending`, `publish_lag_ms`; NATS saudável? | Reiniciar relay Outbox; drenar DLQ após correção da causa raiz. | `outbox_pending` baixo; DLQ estável. |
| **RB-09** | Error budget burn rápido | Top `errors_total{code}`. | Congelar mudanças; mitigar causa dominante (ex.: `017` instável). | *burn rate* < 14,4×. |
| **RB-10** | Rejeições por rate limit (AIOS-CTX-0015) | Identificar tenant/agente ofensor; checar padrão de tráfego. | Revisar `assemble_rps_per_tenant`; investigar *retry storm* de cliente. | Rejeições voltam ao nível basal. |
| **RB-11** | Falso-*hit* de cache acima do SLO (NFR-011) | Amostragem/avaliação offline por tenant/`model_id`; correlacionar com `similarity_threshold` recente. | Elevar `similarity_threshold`; invalidar cluster suspeito (FM-08); quarentena de entradas. | `cache_false_hit_ratio` ≤ 0,5%. |

### Exemplo de execução — RB-04 (Cache hit ratio baixo)

```bash
# 1) Confirmar o tenant e a janela
promtool query instant http://prometheus:9090 \
  'aios_context_cache_hit_ratio:avg1d{tenant="acme"} < 0.35'

# 2) Checar se a cota de cache do tenant está saturada
promtool query instant http://prometheus:9090 \
  'aios_context_cache_entries{tenant="acme"} / aios_context_cache_max_entries{tenant="acme"}'

# 3) Se cota saturada -> acionar eviction manual (LRU semântico) para liberar espaço
curl -XPOST https://context.aios/v1/context/cache/invalidate \
  -H "X-AIOS-Tenant: acme" -H "Idempotency-Key: $(ulid)" \
  -d '{"scope":"tenant","reason":"quota_pressure"}'

# 4) Se cota normal -> revisar similarity_threshold (pode estar conservador demais)
curl -XPUT https://context.aios/v1/context/budgets/tenant \
  -H "X-AIOS-Tenant: acme" -H "Idempotency-Key: $(ulid)" \
  -d '{"similarity_threshold": 0.90}'

# 5) Validar retorno do hit ratio ao SLO em 6h
```

---

## 6. Integração e correlação

- Todo alerta DEVE incluir `trace_id` via *exemplars* (RFC-0001 §5.6) para saltar do
  painel ao trace no [024-Observability](../024-Observability/).
- Eventos operacionais relevantes (`window.rejected`, `cache.evicted`, `cache.invalidated`)
  também são emitidos como eventos de domínio (§6.1 do brief) e auditados por
  [025-Audit](../025-Audit/); monitoramento **não** substitui a trilha de auditoria.
- Métricas de custo/economia (`cost_saved_usd`, `tokens_saved`) são compartilhadas com
  [026-Cost-Optimizer](../026-Cost-Optimizer/) para orçamento de custo global.
- A Δ Task Completion Rate (NFR-005/SLO-5) é medida pelo `035-Benchmark`; este documento
  apenas expõe o painel — o cálculo pertence ao módulo de benchmark global.

## 7. Riscos e alternativas

- **Falsos positivos de latência** em janelas de baixo tráfego. Mitigação: `for: 10m` e
  agregação de 5 min; suprimir alertas quando `rate(assemble_total) < N`.
- **Compression Ratio amostrado por tenant pequeno** pode ter variância alta. Mitigação:
  agregação diária (`:avg1d`) e confirmação pelo bench offline ([Benchmark.md](./Benchmark.md)).
- **Alternativa descartada:** alertar por *threshold* estático de latência absoluta em vez
  de percentis — rejeitada por não refletir cauda (usar `histogram_quantile`).
- **Alternativa descartada:** medir cache hit ratio apenas em janela instantânea — rejeitada
  por oscilar demais em baixo volume; adotada janela de 1 d com recording rule (§6 de
  [Metrics.md](./Metrics.md)).
