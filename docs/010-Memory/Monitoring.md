---
Documento: Monitoring
Módulo: 010-Memory
Status: Draft
Versão: 0.1
Última atualização: 2026-07-20
Responsável (RACI-A): Arquiteto do Módulo 010-Memory
ADRs relacionados: ADR-0102, ADR-0104, ADR-0105, ADR-0108
RFCs relacionados: RFC-0001, RFC-0100
Depende de: 001-Architecture, 003-RFC (RFC-0001), 024-Observability, 025-Audit, 026-Cost-Optimizer, 040-Glossary
---

# 010-Memory — Monitoring (Observabilidade Operacional)

Este documento define **dashboards**, **regras de alerta Prometheus**, **SLI/SLO/SLA**,
**golden signals** e **runbooks** do `MemoryService`. Todos os alvos numéricos são
idênticos aos NFR da §7.2 do `_DESIGN_BRIEF.md` (ver
[NonFunctionalRequirements.md](./NonFunctionalRequirements.md)). As métricas citadas
seguem o catálogo canônico de [Metrics.md](./Metrics.md) (`aios_memory_*`).

Palavras normativas conforme RFC 2119/8174. A plataforma de coleta/painéis (Prometheus,
Grafana, Alertmanager, Seq) pertence a [024-Observability](../024-Observability/) — aqui
apenas configuramos o que é específico de Memory, **sem redefinir** a plataforma.

---

## 1. Golden Signals do MemoryService

| Sinal | Pergunta operacional | Métrica-fonte ([Metrics.md](./Metrics.md)) |
|-------|----------------------|--------------------------------------------|
| **Latency** | As operações respondem dentro do SLO? | `aios_memory_remember_duration_ms`, `aios_memory_recall_duration_ms`, `aios_memory_ann_search_duration_ms` |
| **Traffic** | Qual a carga por operação/tenant? | `aios_memory_remember_total`, `aios_memory_recall_total` |
| **Errors** | Qual a taxa/tipos de falha? | `aios_memory_errors_total{code}` |
| **Saturation** | Cotas, filas e backends estão saturando? | `aios_memory_usage_ratio`, `aios_memory_inflight`, `aios_memory_outbox_pending` |
| **Qualidade (negócio)** | A memória está recuperando o que importa? | `aios_memory_recall_rate` (Memory Recall Rate, R11) |

O *golden signal* de negócio **Memory Recall Rate** é tratado como sinal de primeira
classe, ao lado dos quatro clássicos, por ser o KPI-chave do módulo (NFR-002).

---

## 2. SLI, SLO e SLA

### 2.1 Definições (SLI → SLO)

| # | SLI (como se mede) | SLO (alvo) | Janela | NFR |
|---|--------------------|-----------|--------|-----|
| SLO-1 | p99 de `aios_memory_remember_duration_ms{embedding="false"}` | ≤ **30 ms** | 30 d móveis | NFR-001 |
| SLO-2 | p99 de `aios_memory_remember_duration_ms{embedding="true"}` | ≤ **120 ms** | 30 d | NFR-001 |
| SLO-3 | p99 de `aios_memory_recall_duration_ms{mode="vector"}` (hot/Redis) | ≤ **50 ms** | 30 d | NFR-003 |
| SLO-4 | p99 de `aios_memory_recall_duration_ms{mode="hybrid"}` (Semantic/Episodic ANN) | ≤ **150 ms** | 30 d | NFR-003 |
| SLO-5 | p99 de `aios_memory_recall_duration_ms{mode="graph"}` | ≤ **250 ms** | 30 d | NFR-003 |
| SLO-6 | `aios_memory_recall_rate{source="prod_sample"}` | ≥ **0,95** | 7 d | NFR-002 |
| SLO-7 | Disponibilidade = 1 − taxa de erro `retriable`/`5xx` | ≥ **99,95%** | 30 d | NFR-005 |
| SLO-8 | `aios_memory_usage_ratio` | < **1,0** | instantâneo | NFR-009 |
| SLO-9 | p99 de `aios_memory_ann_search_duration_ms` | ≤ **150 ms** | 30 d | NFR-012 |
| SLO-10 | `aios_memory_consolidation_recall_delta_ratio` | ≥ **−0,01** | por job | NFR-011 |

### 2.2 SLA (compromisso externo)

O SLA publicado ao tenant DEVE ser **mais frouxo** que o SLO interno para preservar
margem operacional:

| Compromisso SLA | Alvo externo | SLO interno correspondente |
|-----------------|--------------|----------------------------|
| Disponibilidade do serviço de memória | **99,9%/mês** | SLO-7 (99,95%) |
| Latência de `recall` (hybrid, p99) | **≤ 250 ms** | SLO-4 (150 ms) |
| RPO (camadas duráveis) | **≤ 15 min** | NFR-006 (5 min) |
| RTO | **≤ 30 min** | NFR-007 (15 min) |

### 2.3 Error budget

O *error budget* de disponibilidade (SLO-7 = 99,95%/30 d) equivale a **~21,6 min/mês**
de indisponibilidade. Política de queima:

| Consumo do budget | Ação |
|-------------------|------|
| < 50% | Operação normal; mudanças liberadas. |
| 50–90% | *Freeze* de mudanças não essenciais; revisão de causas. |
| > 90% ou *burn rate* > 14,4× por 1 h | Página on-call (P1); congelamento total; runbook RB-07. |

Regra de *burn rate* rápido: `aios_memory:error_ratio:rate5m > 0.0072`
(14,4× a taxa de budget) por 5 min → alerta crítico.

---

## 3. Regras de alerta Prometheus

As regras abaixo referenciam as *recording rules* de [Metrics.md](./Metrics.md) §5.1.
Toda regra crítica DEVE ter *runbook* associado (coluna `runbook`).

```yaml
groups:
  - name: aios_memory_slo_alerts
    rules:
      # SLO-1/2 — latência de remember
      - alert: MemoryRememberLatencyHigh
        expr: aios_memory:remember_duration_ms:p99{embedding="false"} > 30
        for: 10m
        labels: { severity: warning, module: "010-memory", runbook: RB-01 }
        annotations:
          summary: "p99 remember (inline) acima de 30 ms (NFR-001)"
          description: "p99={{ $value }}ms para {{ $labels.layer }}."

      - alert: MemoryRememberEmbedLatencyHigh
        expr: aios_memory:remember_duration_ms:p99{embedding="true"} > 120
        for: 10m
        labels: { severity: warning, module: "010-memory", runbook: RB-01 }
        annotations: { summary: "p99 remember (com embedding) acima de 120 ms (NFR-001)" }

      # SLO-3/4/5 — latência de recall por modo
      - alert: MemoryRecallLatencyHigh
        expr: |
          (aios_memory:recall_duration_ms:p99{mode="vector"} > 50)
          or (aios_memory:recall_duration_ms:p99{mode="hybrid"} > 150)
          or (aios_memory:recall_duration_ms:p99{mode="graph"} > 250)
        for: 10m
        labels: { severity: warning, module: "010-memory", runbook: RB-02 }
        annotations: { summary: "p99 recall acima do SLO por modo (NFR-003)" }

      # SLO-6 — Memory Recall Rate (negócio)
      - alert: MemoryRecallRateLow
        expr: aios_memory_recall_rate{source="prod_sample"} < 0.95
        for: 30m
        labels: { severity: critical, module: "010-memory", runbook: RB-03 }
        annotations:
          summary: "Memory Recall Rate < 0,95 (NFR-002)"
          description: "Recall Rate={{ $value }} para tenant {{ $labels.tenant }}."

      # SLO-7 — error budget burn (rápido)
      - alert: MemoryErrorBudgetBurnFast
        expr: aios_memory:error_ratio:rate5m > 0.0072
        for: 5m
        labels: { severity: critical, module: "010-memory", runbook: RB-07 }
        annotations: { summary: "Queima rápida do error budget (>14,4x) — NFR-005" }

  - name: aios_memory_saturation_alerts
    rules:
      # SLO-8 — cota
      - alert: MemoryQuotaNearLimit
        expr: aios_memory_usage_ratio > 0.9
        for: 15m
        labels: { severity: warning, module: "010-memory", runbook: RB-04 }
        annotations: { summary: "Cota {{ $labels.layer }}/{{ $labels.dimension }} > 90% (NFR-009)" }

      - alert: MemoryQuotaExceeded
        expr: increase(aios_memory_quota_exceeded_total[5m]) > 0
        for: 0m
        labels: { severity: warning, module: "010-memory", runbook: RB-04 }
        annotations: { summary: "Escrita negada por cota (AIOS-MEM-0010/0011)" }

      - alert: MemoryBackpressureActive
        expr: max(aios_memory_backpressure_active) == 1
        for: 5m
        labels: { severity: warning, module: "010-memory", runbook: RB-05 }
        annotations: { summary: "Backpressure ativo — fila de escrita saturada (F7)" }

      # Outbox / eventos
      - alert: MemoryOutboxLag
        expr: aios_memory_outbox_pending > 1000
        for: 5m
        labels: { severity: warning, module: "010-memory", runbook: RB-06 }
        annotations: { summary: "Outbox pendente > 1000 — lag de publicação (F8)" }

      - alert: MemoryDlqGrowing
        expr: increase(aios_memory_dlq_messages_total[15m]) > 0
        for: 0m
        labels: { severity: warning, module: "010-memory", runbook: RB-06 }
        annotations: { summary: "Mensagens em DLQ JetStream (F10)" }

  - name: aios_memory_backend_alerts
    rules:
      - alert: MemoryBackendDown
        expr: aios_memory_backend_up == 0
        for: 1m
        labels: { severity: critical, module: "010-memory", runbook: RB-08 }
        annotations: { summary: "Backend {{ $labels.backend }} indisponível (F1–F5)" }

      - alert: MemoryConsolidationRollbackSpike
        expr: increase(aios_memory_consolidation_rollback_total[1h]) > 3
        for: 0m
        labels: { severity: warning, module: "010-memory", runbook: RB-09 }
        annotations: { summary: "Rollbacks de consolidação frequentes (F6, NFR-011)" }
```

---

## 4. Dashboards (Grafana)

Cada painel referencia apenas métricas de [Metrics.md](./Metrics.md). Layout canônico:

```
┌───────────────────── AIOS · Memory · Visão Geral (010) ─────────────────────┐
│ [SLO] Recall Rate: 0.968 ✓   Disp.: 99.97% ✓   Budget 30d restante: 61%     │
├──────────────────────────────┬──────────────────────────────────────────────┤
│ Latency remember p50/p95/p99 │ Latency recall p99 por mode (vector/hybrid/g)│
│ (linha vs. limites 30/120ms) │ (linha vs. limites 50/150/250ms)             │
├──────────────────────────────┼──────────────────────────────────────────────┤
│ Traffic: remember/s recall/s │ Errors: rate por code (AIOS-MEM-*) empilhado │
│ por tenant (top 10)          │ (heatmap)                                    │
├──────────────────────────────┼──────────────────────────────────────────────┤
│ Saturation: usage_ratio por  │ Backends up/down (redis/pg/pgvector/age/     │
│ layer/dimension (barras)     │ minio/nats) + backend_call_duration p99      │
├──────────────────────────────┼──────────────────────────────────────────────┤
│ Consolidation: jobs ativos,  │ Outbox pending + publish_lag_ms p99 +        │
│ rollback_total, recall_delta │ DLQ messages                                 │
└──────────────────────────────┴──────────────────────────────────────────────┘
```

| Dashboard | Público | Painéis-chave |
|-----------|---------|---------------|
| **Memory Overview** | On-call / SRE | SLOs, golden signals, budget. |
| **Memory Recall Quality** | Time do módulo / 023 | `recall_rate`, `recall_at_k_ratio`, `recall_score`, delta de consolidação. |
| **Memory Capacity** | SRE / 026-Cost | `usage_ratio`, `items`, `bytes`, `vectors` por tenant; projeção de esgotamento. |
| **Memory Backends** | SRE / DBA | saúde e latência de Redis/PG/pgvector/AGE/MinIO/NATS. |
| **Memory Events/Outbox** | SRE | `events_published_total`, `outbox_pending`, `publish_lag_ms`, DLQ. |

---

## 5. Runbooks

Cada alerta crítico aponta para um runbook. Formato: **sintoma → diagnóstico → ação →
verificação**. Runbooks completos versionados em `../024-Observability/runbooks/`.

| ID | Alerta / Cenário | Diagnóstico inicial | Ação primária | Verificação |
|----|------------------|---------------------|---------------|-------------|
| **RB-01** | Latência `remember` alta | Ver `backend_call_duration_ms` (PG) e `embedding_duration_ms` (017). | Se embedding: cf. F3 (enfileirar sem embedding). Se PG: cf. RB-08. | p99 volta a ≤ 30/120 ms. |
| **RB-02** | Latência `recall` alta | Isolar por `mode`; checar `ann_search_duration_ms` e `ef_search`. | Ajustar `memory.hnsw.ef_search` (T); escalar réplicas de leitura. | p99 ≤ 50/150/250 ms. |
| **RB-03** | Recall Rate < 0,95 | Correlacionar com consolidação recente (`recall_delta_ratio`). | Se pós-consolidação: acionar rollback (RB-09); senão revisar `min_score`/fusão. | Recall Rate ≥ 0,95. |
| **RB-04** | Cota perto/estourada | Ver `usage_ratio` por dimensão; identificar tenant/camada. | Acionar poda (ForgettingEngine); revisar cota com 026. | `usage_ratio` < 0,9. |
| **RB-05** | Backpressure ativo | `inflight` vs. `max_inflight` (1000). | Escalar shard; aumentar `max_inflight` (G) com cautela. | `backpressure_active`=0. |
| **RB-06** | Outbox lag / DLQ | `outbox_pending`, `publish_lag_ms`; NATS saudável? | Reiniciar relay Outbox; drenar DLQ após correção. | `outbox_pending` baixo; DLQ estável. |
| **RB-07** | Error budget burn | Top `errors_total{code}`. | Congelar mudanças; mitigar causa dominante. | *burn rate* < 14,4×. |
| **RB-08** | Backend down | Qual `backend`? Aplicar tabela de falhas §9 do brief (F1–F5). | Failover PG (F2); fail-open Redis (F1); degradar grafo (F5). | `backend_up`=1. |
| **RB-09** | Rollbacks de consolidação | `consolidation_rollback_total{reason}`. | Pausar `memory.consolidation.auto` (T); investigar regras 023. | Sem novos rollbacks. |

### Exemplo de execução — RB-03 (Recall Rate baixo)

```bash
# 1) Confirmar o tenant e a janela
promtool query instant http://prometheus:9090 \
  'aios_memory_recall_rate{source="prod_sample"} < 0.95'

# 2) Verificar se houve consolidação com regressão
promtool query instant http://prometheus:9090 \
  'aios_memory_consolidation_recall_delta_ratio{tenant="acme"} < -0.01'

# 3) Se regressão -> rollback da última consolidação (idempotente por version_id)
curl -XPOST https://memory.aios/v1/memory/consolidate/$JOB_ID/rollback \
  -H "X-AIOS-Tenant: acme" -H "Idempotency-Key: $(ulid)"

# 4) Validar retorno do Recall Rate ao SLO em 30 min
```

---

## 6. Integração e correlação

- Todo alerta DEVE incluir `trace_id` via *exemplars* (RFC-0001 §5.6) para saltar do
  painel ao trace no [024-Observability](../024-Observability/).
- Eventos operacionais relevantes (rollback, purge/RTBF, quota.exceeded) também são
  emitidos como eventos de domínio (§6.1 do brief) e auditados por
  [025-Audit](../025-Audit/); monitoramento **não** substitui a trilha de auditoria.
- Métricas de custo/capacidade são compartilhadas com
  [026-Cost-Optimizer](../026-Cost-Optimizer/) para orçamento.

## 7. Riscos e alternativas

- **Falsos positivos de latência** em janelas de baixo tráfego. Mitigação: `for: 10m` e
  agregação de 5 min; suprimir alertas quando `rate(remember_total) < N`.
- **Recall Rate amostrado** pode ter variância. Mitigação: janela de 7 d (SLO-6) e
  confirmação pelo bench offline ([Benchmark.md](./Benchmark.md)).
- **Alternativa descartada:** alertar por *threshold* estático de latência absoluta em vez
  de percentis — rejeitada por não refletir cauda (usar `histogram_quantile`).
