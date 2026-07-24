---
Documento: Monitoring
Módulo: 024-Observability
Status: Draft
Versão: 0.1
Última atualização: 2026-07-23
Responsável (RACI-A): Arquiteto do Módulo 024-Observability + SRE
ADRs relacionados: ADR-0010, ADR-0242, ADR-0243, ADR-0245, ADR-0247
RFCs relacionados: RFC-0001, RFC-0241
Depende de: Metrics.md, NonFunctionalRequirements.md, FailureRecovery.md, 029-Operations
---

# 024-Observability — Monitoramento

> **Quem observa o observador.** Este documento tem uma dificuldade que nenhum outro
> módulo tem: monitorar o sistema de monitoramento usando o próprio sistema de
> monitoramento é circular. A saída é o `SelfTelemetry`, *scrapeado* **diretamente**
> pelo Prometheus, sem passar pelo pipeline de ingestão (`./Deployment.md` §4). Todos
> os alertas de §4 marcados **[fora de banda]** dependem apenas desse caminho.

## 1. Golden signals

| Sinal | Métrica | Alvo |
|-------|---------|------|
| **Latência** | `aios_obs_ingest_latency_p99` | ≤ 50 ms (NFR-001) |
| **Tráfego** | `rate(aios_obs_received_total[1m])` | ≥ 2.000.000 dp/s por réplica (NFR-003) |
| **Erros** | `aios_obs_drop_ratio` | ≤ 0,001 (NFR-006) |
| **Saturação** | `aios_obs_queue_depth`, `aios_obs_cardinality_pressure` | fila < 60%; cardinalidade < 80% |

> Atenção à leitura: **`aios_obs_dropped_total` não é "erro" no sentido usual**. O
> descarte é comportamento projetado (fail-open, ADR-0243). O que se monitora é a
> **taxa** e sua variação — descarte de 0,05% é operação normal; de 2% é degradação
> anunciada por `telemetry.ingest.degraded`.

## 2. Rastreamento (traces)

O módulo instrumenta o próprio pipeline com spans:

| Span | Atributos |
|------|-----------|
| `obs.ingest.batch` | `signal_kind`, `transport`, `batch_bytes`, `aios.tenant` |
| `obs.admission` | `accepted_count`, `quarantined_count` |
| `obs.cardinality.check` | `tenant`, `budget_ratio` |
| `obs.pii.redact` | `redacted_count` (nunca os valores) |
| `obs.trace.tail_decide` | `decision`, `reason` |
| `obs.slo.evaluate` | `slo_key`, `sli_value`, `burn_rate_1h` |
| `obs.alert.transition` | `rule_key`, `from`, `to` |
| `obs.query` | `store`, `series_scanned`, `duration_ms` |

Amostragem do próprio módulo: `head_sample_ratio` reduzido (1%) por padrão, pois o
volume é altíssimo e os traces relevantes — erro e cauda — são retidos por política
*tail-based* como qualquer outro (ADR-0244).

## 3. SLO / SLI e error budget

| SLO | SLI | Janela | Error budget |
|-----|-----|--------|--------------|
| Disponibilidade de ingestão ≥ 99,9% (NFR-004) | Fração de lotes aceitos | 30 d | 43 min/mês |
| **Disponibilidade de alerta ≥ 99,95%** (NFR-005) | Probe sintético fim a fim: alerta artificial dispara e notifica | 30 d | 21 min/mês |
| p99 de ingestão ≤ 50 ms (NFR-001) | `aios_obs_ingest_latency_p99` | 7 d | 1% das janelas de 5 min |
| Perda ≤ 0,1% (NFR-006) | `aios_obs_drop_ratio` | 30 d | — |
| Frescor ≤ 10 s (NFR-007) | Probe sintético fim a fim | 7 d | 1% das amostras |
| Cobertura de runbook 100% (NFR-016) | `aios_obs_runbook_stale_total = 0` | contínua | **0** (não há budget) |

O **probe sintético de alerta** merece destaque: uma vez por minuto, uma condição
artificial é criada e o tempo até a notificação é medido. É a única forma de verificar
a capacidade cuja falha é invisível por definição — um sistema de alerta quebrado não
alerta que está quebrado.

## 4. Alertas Prometheus

```yaml
groups:
- name: observability-critical
  rules:
  - alert: ObservabilityAlertPipelineDown          # [fora de banda]
    expr: absent(up{job="alert-evaluator"} == 1) or increase(aios_obs_alert_transitions_total[15m]) == 0
    for: 5m
    labels: { severity: critical, page: immediate }
    annotations:
      summary: "Pipeline de alerta parado — o sistema não consegue avisar sobre nada"
      runbook: "./Monitoring.md#rb-o-01"

  - alert: ObservabilitySelfTelemetryDown          # [fora de banda]
    expr: up{job="obs-self-telemetry"} == 0
    for: 2m
    labels: { severity: critical, page: immediate }
    annotations:
      summary: "Meta-observabilidade indisponível — o observador está cego sobre si"
      runbook: "./Monitoring.md#rb-o-02"

  - alert: ObservabilityIngestDropHigh
    expr: aios_obs_drop_ratio > 0.05
    for: 5m
    labels: { severity: critical }
    annotations:
      summary: "Descarte acima de 5% — decisões estão sendo tomadas sobre telemetria incompleta"
      runbook: "./Monitoring.md#rb-o-04"

  - alert: ObservabilityCardinalityCritical
    expr: aios_obs_cardinality_pressure > 0.95
    for: 5m
    labels: { severity: critical }
    annotations:
      summary: "Orçamento de cardinalidade quase esgotado"
      runbook: "./Monitoring.md#rb-o-03"

- name: observability-warning
  rules:
  - alert: ObservabilityIngestLatencyHigh
    expr: aios_obs_ingest_latency_p99 > 50
    for: 10m
    labels: { severity: warning }
    annotations: { summary: "p99 de ingestão acima do SLO (NFR-001)", runbook: "./Monitoring.md#rb-o-05" }

  - alert: ObservabilityCardinalityPressure
    expr: aios_obs_cardinality_pressure > 0.8
    for: 15m
    labels: { severity: warning }
    annotations: { summary: "Orçamento de cardinalidade acima de 80%", runbook: "./Monitoring.md#rb-o-03" }

  - alert: ObservabilityForbiddenLabelUsed
    expr: increase(aios_obs_forbidden_label_total[1h]) > 0
    for: 0m
    labels: { severity: warning }
    annotations: { summary: "Label de identidade tentado — descritor de sinal precisa de correção", runbook: "./Monitoring.md#rb-o-06" }

  - alert: ObservabilityPiiUndeclared
    expr: increase(aios_obs_pii_undeclared_total[1h]) > 0
    for: 0m
    labels: { severity: warning }
    annotations: { summary: "PII detectada em sinal não declarado como pii", runbook: "./Monitoring.md#rb-o-07" }

  - alert: ObservabilityNotifyLatencyHigh
    expr: aios_obs_notify_latency_p95_critical > 60000
    for: 5m
    labels: { severity: warning }
    annotations: { summary: "Notificação de alerta crítico demorando > 60 s", runbook: "./Monitoring.md#rb-o-08" }

  - alert: ObservabilitySilenceChurnHigh
    expr: increase(aios_obs_silence_created_total[30d]) > 3
    for: 1h
    labels: { severity: warning }
    annotations: { summary: "Mesma regra silenciada repetidamente — limiar errado ou problema ignorado", runbook: "./Monitoring.md#rb-o-09" }

  - alert: ObservabilityRunbookStale
    expr: aios_obs_runbook_stale_total > 0
    for: 24h
    labels: { severity: warning }
    annotations: { summary: "Runbook sem revisão há mais de 180 dias (NFR-016)", runbook: "./Monitoring.md#rb-o-10" }

  - alert: ObservabilityQueryAbortedHigh
    expr: rate(aios_obs_query_aborted_total[10m]) > 0.5
    for: 15m
    labels: { severity: warning }
    annotations: { summary: "Consultas sendo abortadas por orçamento", runbook: "./Monitoring.md#rb-o-11" }

  - alert: ObservabilityTraceFragmented
    expr: rate(aios_obs_trace_fragmented_total[10m]) > 1
    for: 15m
    labels: { severity: warning }
    annotations: { summary: "Traces fragmentados — roteamento por hash(trace_id) inconsistente", runbook: "./Monitoring.md#rb-o-12" }

  - alert: ObservabilityDownsampleLag
    expr: time() - aios_obs_downsample_last_success_timestamp > 7200
    for: 30m
    labels: { severity: warning }
    annotations: { summary: "Downsampling atrasado — retenção longa em risco", runbook: "./Monitoring.md#rb-o-13" }
```

## 5. Runbooks

| ID | Alerta | Procedimento |
|----|--------|--------------|
| **RB-O-01** | `ObservabilityAlertPipelineDown` | 1) **P1 imediato**: enquanto durar, nenhum alerta do AIOS inteiro dispara. 2) Verificar `alert-evaluator` (pods, logs, conexão com PostgreSQL e Alertmanager). 3) Se o PostgreSQL estiver fora, ativar o modo de avaliação em memória (degradado, sem persistência de instância). 4) Comunicar o plantão de que a detecção automática está suspensa e ativar vigilância manual dos dashboards. 5) Só encerrar o incidente após o **probe sintético** voltar a passar. |
| **RB-O-02** | `ObservabilitySelfTelemetryDown` | 1) **P1**: o observador está cego sobre si mesmo — não é possível saber se o pipeline está perdendo dado. 2) Verificar o *scrape* direto (`job="obs-self-telemetry"`), que **não** passa pelo pipeline. 3) Se o endpoint está de pé mas o Prometheus não o alcança, o problema é de rede/descoberta, não do módulo. 4) Enquanto durar, tratar toda métrica de ingestão como **não confiável**. |
| **RB-O-03** | `ObservabilityCardinalityPressure` / `Critical` | 1) Identificar o `signal_name` de maior crescimento por `aios_obs_active_series`. 2) Identificar o `owner_module`. 3) Verificar `aios_obs_forbidden_label_total` e `aios_obs_labels_dropped_total` — costuma haver um label novo introduzido por deploy recente. 4) Quarentenar o sinal infrator (`obs:signal:manage`) e notificar o dono. 5) **Não** aumentar `max_series_per_tenant` como primeira ação: isso adia o problema e aumenta o custo permanentemente. |
| **RB-O-04** | `ObservabilityIngestDropHigh` | 1) Verificar `reason` em `aios_obs_dropped_total`: `queue_full` → escalar `otel-gateway`; `storage_down` → ir ao armazenamento afetado; `quarantined` → ir para RB-O-03. 2) Confirmar que o `otel-agent` está bufferizando (o descarte no agente é pior: perde na origem). 3) Comunicar às equipes que a telemetria do período está incompleta — decisões sobre a janela afetada precisam disso. |
| **RB-O-05** | `ObservabilityIngestLatencyHigh` | 1) Verificar `aios_obs_queue_depth` e CPU do gateway. 2) Checar `aios_obs_batch_bytes` — lotes maiores que o usual indicam emissor mal configurado. 3) Checar latência do `PiiRedactor` (`obs.pii.redact` spans): regra de redação custosa é causa comum. 4) Escalar réplicas se a saturação for confirmada. |
| **RB-O-06** | `ObservabilityForbiddenLabelUsed` | 1) Identificar `label` e `owner_module`. 2) Abrir issue para o módulo dono: a granularidade que ele quer pertence a trace/log, não a métrica. 3) Verificar se o descritor precisa de `forbidden_labels` explícito. 4) **Não** adicionar o label a `allowed_labels` sem análise de cardinalidade. |
| **RB-O-07** | `ObservabilityPiiUndeclared` | 1) Tratar como **incidente de privacidade**. 2) Identificar o sinal e o `owner_module`. 3) Confirmar que a redação heurística atuou (o dado **não** deve ter sido persistido). 4) Corrigir o `data_class` do descritor para `pii` em nova versão. 5) Se houver suspeita de persistência, acionar expurgo e o DPO. |
| **RB-O-08** | `ObservabilityNotifyLatencyHigh` | 1) Verificar Alertmanager (fila, integrações, rate limit do canal). 2) Verificar `aios_obs_outbox_pending`. 3) Enquanto durar, o plantão **DEVE** monitorar o dashboard de alertas abertos diretamente. |
| **RB-O-09** | `ObservabilitySilenceChurnHigh` | 1) Listar os silêncios da `rule_key` em `telemetry.silence`. 2) Decidir com o dono do serviço: o limiar está errado (corrigir a regra) **ou** o problema é real e está sendo ignorado (abrir issue de correção). 3) **Não** renovar o silêncio uma quarta vez sem uma das duas ações. |
| **RB-O-10** | `ObservabilityRunbookStale` | 1) Listar runbooks com `last_reviewed_at` > 180 d. 2) Atribuir revisão ao `owner_module`. 3) Runbook desatualizado é pior que ausente: dá instruções que não funcionam mais durante um incidente. |
| **RB-O-11** | `ObservabilityQueryAbortedHigh` | 1) Identificar o tenant e a consulta. 2) Verificar se um painel está agregando ao vivo o que deveria ser *recording rule*. 3) Ajustar o painel antes de aumentar o orçamento. |
| **RB-O-12** | `ObservabilityTraceFragmented` | 1) Se houver rollout de `otel-gateway` em curso, é **esperado e temporário** (`./Deployment.md` §6) — silenciar pela janela. 2) Caso contrário, verificar o roteamento por `hash(trace_id)` no balanceador. |
| **RB-O-13** | `ObservabilityDownsampleLag` | 1) Verificar o job de *downsampling* e o espaço no objeto (MinIO). 2) Enquanto durar, consultas de janela longa caem para a camada disponível. 3) Se persistir além da retenção `raw` (15 d), há **perda definitiva** de resolução histórica — escalar como P1. |

## 6. Dashboards

| Painel | Conteúdo | Público |
|--------|----------|---------|
| **Obs — Pipeline** | Recebidos/s, descartados/s por `reason`, profundidade de fila, p99 de ingestão, frescor | SRE |
| **Obs — Cardinalidade** | Séries ativas por tenant e por sinal, pressão de orçamento, quarentenas, labels descartados | Plataforma |
| **Obs — Alertas e SLO** | Alertas abertos por severidade/estado, latência de notificação, SLI e burn rate por SLO, budget consumido | SRE + donos de serviço |
| **Obs — Privacidade** | Atributos redigidos, PII não declarada, comprovantes de expurgo | DPO + segurança |
| **Obs — Retenção e custo** | Volume por camada, duração do downsampling, expurgos, ocupação de armazenamento | Plataforma |
| **Obs — Meta [fora de banda]** | Saúde do `SelfTelemetry`, probe sintético de alerta, probe de frescor | SRE |

## 7. Referências

- Métricas: `./Metrics.md` · Logs: `./Logging.md`
- Metas: `./NonFunctionalRequirements.md` · Falhas: `./FailureRecovery.md`
- Plantão e escalonamento: `../029-Operations/`
- Runbooks versionados: `./_DESIGN_BRIEF.md` §3.8 (entidade `Runbook`)

*Fim de `Monitoring.md`.*
