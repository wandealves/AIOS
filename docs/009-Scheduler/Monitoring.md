---
Documento: Monitoring
Módulo: 009-Scheduler
Status: Draft
Versão: 0.1
Última atualização: 2026-07-20
Responsável (RACI-A): Arquiteto do Módulo 009-Scheduler
ADRs relacionados: ADR-0090 (admission control), ADR-0093 (filas ZSET/aging), ADR-0094 (preempção compensável), ADR-0095 (backpressure em três níveis), ADR-0097 (outbox transacional)
RFCs relacionados: RFC-0001 (baseline, §5.6 correlação); RFC-0090, RFC-0091 (a propor)
Depende de: 024-Observability, 025-Audit, 006-Kernel, 007-Agent-Runtime, 022-Policy, 026-Cost-Optimizer, 027-Cluster
---

# 009-Scheduler — Monitoring

> Este documento define **o que observar, como observar e o que fazer quando
> algo sai do esperado** no Scheduler cognitivo (009). Consome a infraestrutura
> transversal de `024-Observability` (Prometheus/Grafana/OTel Collector) e de
> `025-Audit`, e **não redefine** os contratos de correlação (`traceparent`,
> `X-AIOS-Tenant`) já fixados na `../003-RFC/RFC-0001-Architecture-Baseline.md`
> §5.6. O catálogo formal de métricas usado aqui está em `./Metrics.md`; este
> documento cobre dashboards, golden signals, SLO/SLA/SLI, regras de alerta
> Prometheus e runbooks.

## 1. Objetivo e Escopo

O monitoramento do Scheduler DEVE responder, em tempo real, quatro perguntas
operacionais:

1. **O Scheduler está honrando seus contratos de desempenho e disponibilidade?**
   (NFR-001..NFR-017 de `NonFunctionalRequirements.md`).
2. **A admissão está correta e segura** — nem permite overcommit de slot (risco
   de corretude/segurança), nem rejeita além do necessário (risco de
   disponibilidade/negócio)?
3. **As filas estão saudáveis** (sem starvation de tenant, sem crescimento
   descontrolado, sem preempção em *thrashing*)?
4. **A cadeia de decisão→proveniência→evento está íntegra** (toda decisão tem
   trace, toda decisão é persistida, todo evento é publicado sem perda)?

**Escopo.** Este documento cobre exclusivamente a observabilidade **do próprio
serviço Scheduler** (os componentes internos do `_DESIGN_BRIEF.md` §2:
`SchedulingGateway`, `AdmissionController`, `PriorityCostEvaluator`,
`ISchedulingPolicy`, `FeatureExtractor`, `PlacementEngine`, `ShardRouter`,
`PreemptionManager`, `PriorityQueueStore`, `DispatchLoop`,
`BackpressureController`, `QuotaLedger`, `CapacityProvider`, `CostEstimator`,
`PolicyClient`, `BudgetClient`, `RuntimeSupervisorClient`, `IdempotencyStore`,
`DecisionJournal`, `EventPublisher`, `SchedulerTelemetry`,
`ReconciliationWorker`). O monitoramento interno de `022-Policy`,
`026-Cost-Optimizer`, `027-Cluster` e `007-Agent-Runtime` é responsabilidade
dos próprios módulos — o Scheduler monitora apenas a sua **fronteira de
integração** com eles (latência de cliente, taxa de erro, estado do circuit
breaker).

## 2. Golden Signals do Scheduler

| Golden Signal | Definição no contexto do Scheduler | Métrica primária (ver `Metrics.md`) | Meta associada |
|----------------|----------------------------------|--------------------------------------|-----------------|
| **Latência** | Tempo para produzir uma `SchedulingDecision`, do recebimento no `SchedulingGateway` até a resposta de `Submit`. | `aios_scheduler_decision_latency_ms` | NFR-001 (p99 ≤ 20 ms, p50 ≤ 5 ms) |
| **Tráfego** | Taxa de `SchedulingRequest` recebidas por segundo, por tenant e classe de serviço. | `aios_scheduler_requests_total` (rate) | NFR-002 (≥ 50.000 decisões/s por réplica) |
| **Erros** | Fração de requisições que terminam em `outcome=REJECTED` por causa não-esperada (excluindo rejeições de negócio esperadas, ex.: cota do tenant esgotada por design), segmentada por `AIOS-SCHED-<NNNN>`. | `aios_scheduler_admission_rejected_total`, `aios_scheduler_events_publish_errors_total` | Orçamento de erro derivado de NFR-003; 0 para `AIOS-SCHED-0012` (falha interna) |
| **Saturação** | Ocupação dos recursos finitos do caminho quente: profundidade de fila, consumo de cota, pressão de backpressure, outbox pendente. | `aios_scheduler_queue_depth`, `aios_scheduler_quota_consumed_ratio`, `aios_scheduler_backpressure_pressure_ratio`, `aios_scheduler_outbox_pending` | NFR-005 (0 overcommit), NFR-007 (RPO ≤ 5 min) |

## 3. SLOs / SLIs e Orçamento de Erro

Os SLOs do Scheduler **DEVEM** ser rastreados como frações de tempo dentro do
orçamento de erro (*error budget*), com janela móvel de 30 dias, alinhados à
`NonFunctionalRequirements.md`.

| SLO | SLI (fonte) | Meta | Janela | Orçamento de erro (30d) |
|-----|-------------|------|--------|---------------------------|
| Disponibilidade do endpoint de scheduling | `aios_scheduler_up` combinado com taxa de erro 5xx/gRPC-UNAVAILABLE em `Submit`/`GetDecision` | ≥ 99,95% (NFR-003) | 30d rolling | 21,6 min de indisponibilidade |
| Latência de decisão | `histogram_quantile(0.99, aios_scheduler_decision_latency_ms)` | ≤ 20 ms (p99), ≤ 5 ms (p50) | 30d rolling | ≤ 1% das amostras acima do limiar |
| Throughput sustentado | `rate(aios_scheduler_requests_total[1m])` por réplica | ≥ 50.000 decisões/s | contínuo (validado por benchmark de release) | — (meta de capacidade, não de erro) |
| Corretude de admissão | `aios_scheduler_quota_overcommit_total` | 0 (NFR-005) | contínuo | qualquer valor > 0 é incidente SEV-1 |
| Fairness | `aios_scheduler_fairness_share_ratio` por tenant | ≥ `scheduler.fairness.min_share` (default 1%) em janela de 60s | 60s rolling | violações toleradas apenas sob pico documentado |
| Overhead de preempção | `histogram_quantile(0.99, aios_scheduler_preemption_ms)` | ≤ 50 ms (NFR-009) | 30d rolling | ≤ 1% das amostras acima do limiar |
| Durabilidade da decisão | `aios_scheduler_events_lost_total` | 0 eventos perdidos (NFR-007) | contínuo | qualquer valor > 0 é incidente SEV-1 |
| Cobertura de proveniência | `aios_scheduler_decisions_recorded_total` / `aios_scheduler_requests_total` | 100% (NFR-010) | 5m rolling | qualquer desvio sustentado é ticket |
| Idempotência | `aios_scheduler_idempotency_conflicts_total` sobre total de replays | 0 admissões duplicadas (NFR-011) | contínuo | qualquer overcommit associado é SEV-1 |
| Isolamento multi-tenant | `aios_scheduler_tenant_mismatch_total` | 0 (NFR-016) | contínuo | qualquer valor > 0 é incidente de segurança |

**Multi-window multi-burn-rate.** Alertas de SLO de disponibilidade e latência
DEVEM usar o padrão de duas janelas (curta e longa) para equilibrar
sensibilidade e ruído, conforme detalhado na §5.

## 4. Dashboards (Grafana)

Os dashboards do Scheduler são provisionados como código (JSON versionado em
`029-Operations`) e publicados no Grafana provido por `024-Observability`. Todo
painel DEVE permitir filtro por `tenant`, `service_class` e `shard` (template
variables).

| Dashboard | Público-alvo | Painéis principais |
|-----------|--------------|---------------------|
| **Scheduler Overview** | On-call, SRE | RPS por classe de serviço; latência p50/p95/p99 de decisão; taxa de rejeição por código `AIOS-SCHED-*`; réplicas saudáveis; burn rate de SLO. |
| **Admission & Cost** | Time de plataforma, FinOps | Taxa de admissão/rejeição por tenant; distribuição de `cost_score`; pesos α/β/γ ativos por tenant; violações de `quality_floor`; motor de política ativo (`heuristic`/`rl`). |
| **Queues & Fairness** | SRE, Plataforma | Profundidade de fila por (tenant, classe, shard); idade da tarefa mais antiga; `fairness_share_ratio` por tenant (top/bottom-N); boost de aging saturado. |
| **Placement & Shards** | Plataforma | Carga por shard (`shard_load`); falhas de placement; fallback de afinidade; progresso de rebalanceamento de shards. |
| **Preemption** | SRE, Plataforma | Preempções/s por classe; overhead de decisão (p99); grace period efetivo; bloqueios por cooldown (anti-*thrashing*); preempções negadas pelo PDP. |
| **Backpressure** | SRE, on-call | Nível de backpressure por shard (accept/defer/reject); pressão combinada; `Retry-After` sinalizado; oscilação de nível (flapping). |
| **Decision Journal & Events** | Plataforma, SRE | `outbox_pending`; lag do outbox; taxa de publicação por subject; erros de publicação no JetStream; cobertura de proveniência. |
| **Dependency Health** | Plataforma, SRE | Latência/erro por dependência (022/026/027/007); estado dos circuit breakers; idade do heartbeat de capacidade. |

### 4.1 Mockup ASCII — Dashboard "Scheduler Overview"

```
┌────────────────────────── Scheduler Overview  (tenant=*, class=*, shard=*) ───────────────────────┐
│  RPS por classe de serviço              │  Latência p99 de decisão (ms)                           │
│  ▇▇▇▇▇▇▇▇▇▇ INTERACTIVE  22.4k/s        │  INTERACTIVE  ████████░░░░░░░░░░  12 ms  (SLO 20)        │
│  ▇▇▇▇▇▇▇▇ BATCH          14.1k/s        │  BATCH        ██████░░░░░░░░░░░░   9 ms  (SLO 20)        │
│  ▇▇▇▇ BACKGROUND          6.2k/s        │  BACKGROUND   █████░░░░░░░░░░░░░   7 ms  (SLO 20)        │
├──────────────────────────────────────────┼───────────────────────────────────────────────────────┤
│  Taxa de rejeição (janela 5m)            │  Burn rate do SLO de disponibilidade                    │
│  AIOS-SCHED-0001 (cota)     0.6%         │  janela 1h : 0.7x  (ok)                                 │
│  AIOS-SCHED-0002 (orçamento) 0.1%        │  janela 6h : 0.9x  (ok)                                 │
│  AIOS-SCHED-0003 (backpressure) 0.0%     │  janela 3d : 1.0x  (ok)                                 │
├──────────────────────────────────────────┴───────────────────────────────────────────────────────┤
│  Réplicas saudáveis: 8/8   Outbox pending: 12   CB(Policy)=closed   CB(CostOpt)=closed   CB(RTSup)=closed │
└────────────────────────────────────────────────────────────────────────────────────────────────────┘
```

## 5. Regras de Alerta (Prometheus)

> `for` é o tempo mínimo em que a condição DEVE persistir antes de disparar,
> evitando *flapping*. Severidade `page` aciona on-call; `ticket` gera item de
> backlog; `info` é apenas painel/registro.

| Alerta | Expressão PromQL (resumida) | `for` | Severidade | Runbook |
|--------|------------------------------|-------|------------|---------|
| `SchedulerDecisionLatencyHigh` | `histogram_quantile(0.99, sum(rate(aios_scheduler_decision_latency_ms_bucket[5m])) by (le)) > 20` | 10m | page | RB-S-01 |
| `SchedulerThroughputBelowFloor` | `sum(rate(aios_scheduler_requests_total[5m])) < 5000 and predict_linear(...) < ...` (uso relativo: queda ≥ 50% vs. baseline móvel) | 10m | ticket | RB-S-01 |
| `SchedulerAdmissionErrorRateHigh` | `sum(rate(aios_scheduler_admission_rejected_total{reason_code="AIOS-SCHED-0012"}[5m])) / sum(rate(aios_scheduler_requests_total[5m])) > 0.005` | 5m | page | RB-S-02 |
| `SchedulerQuotaOvercommitDetected` | `increase(aios_scheduler_quota_overcommit_total[5m]) > 0` | 0s | page (SEV-1) | RB-S-03 |
| `SchedulerBackpressureRejectSustained` | `avg(aios_scheduler_backpressure_pressure_ratio) by (shard) > 0.95` | 10m | page | RB-S-04 |
| `SchedulerBackpressureFlapping` | `sum(increase(aios_scheduler_backpressure_level_changes_total[10m])) by (shard) > 10` | 10m | ticket | RB-S-04 |
| `SchedulerQueueDepthGrowing` | `deriv(aios_scheduler_queue_depth[10m]) > 0` sustentado + `aios_scheduler_queue_depth > 10000` | 10m | page | RB-S-05 |
| `SchedulerOldestWaitHigh` | `aios_scheduler_queue_oldest_wait_ms > 60000` | 5m | ticket | RB-S-05 |
| `SchedulerFairnessViolation` | `aios_scheduler_fairness_share_ratio < scheduler_fairness_min_share` (junção com config) | 5m | ticket | RB-S-06 |
| `SchedulerPreemptionOverheadHigh` | `histogram_quantile(0.99, sum(rate(aios_scheduler_preemption_ms_bucket[5m])) by (le)) > 50` | 10m | ticket | RB-S-07 |
| `SchedulerPreemptionThrashing` | `sum(rate(aios_scheduler_preemptions_total[5m])) > 100 and sum(rate(aios_scheduler_preemption_cooldown_blocked_total[5m])) > 50` | 10m | page | RB-S-07 |
| `SchedulerOutboxBacklogGrowing` | `aios_scheduler_outbox_pending > 1000` | 5m | page | RB-S-08 |
| `SchedulerOutboxLagHigh` | `aios_scheduler_outbox_lag_ms > 5000` | 5m | page | RB-S-08 |
| `SchedulerEventLossDetected` | `increase(aios_scheduler_events_lost_total[10m]) > 0` | 0s | page (SEV-1) | RB-S-08 |
| `SchedulerDependencyCircuitOpen` | `aios_scheduler_circuit_breaker_state{dependency=~"022-policy|026-cost-optimizer|027-cluster|007-runtime-supervisor"} == 2` | 1m | page | RB-S-09 |
| `SchedulerCapacityStale` | `aios_scheduler_capacity_heartbeat_age_ms > scheduler_capacity_heartbeat_ttl_ms * 2` | 2m | page | RB-S-09 |
| `SchedulerTenantMismatchDetected` | `increase(aios_scheduler_tenant_mismatch_total[5m]) > 0` | 0s | page (SEV-1) | RB-S-10 |
| `SchedulerReplicaDown` | `aios_scheduler_up == 0` | 2m | page | RB-S-11 |
| `SchedulerShardOwnershipGap` | `sum(aios_scheduler_shard_ownership_active) < scheduler_shards_count` | 2m | page | RB-S-11 |
| **Burn-rate SLO (rápido)** | curta: `error_rate_5m > 14.4 * (1 - 0.9995)`; longa: `error_rate_1h > 14.4 * (1 - 0.9995)` (ambas verdadeiras) | 5m | page | RB-S-02 |
| **Burn-rate SLO (lento)** | curta: `error_rate_1h > 6 * (1 - 0.9995)`; longa: `error_rate_6h > 6 * (1 - 0.9995)` | 30m | ticket | RB-S-02 |

## 6. Runbooks Associados

> Os runbooks completos (passo-a-passo executável) residem em
> `../029-Operations/Runbooks/` (fonte operacional); este documento mantém
> apenas o resumo e o vínculo alerta → runbook.

| ID | Runbook | Gatilho | Resumo dos passos |
|----|---------|---------|---------------------|
| RB-S-01 | Latência/throughput degradado | `SchedulerDecisionLatencyHigh`, `SchedulerThroughputBelowFloor` | 1) Verificar saturação de CPU/GC da réplica. 2) Checar latência de Redis (`PriorityQueueStore`/`QuotaLedger`). 3) Checar caches de `PolicyClient`/`BudgetClient` (miss rate alto força chamada síncrona). 4) Escalar réplicas se saturação confirmada. |
| RB-S-02 | Taxa de erro elevada | `SchedulerAdmissionErrorRateHigh`, burn-rate | 1) Segmentar erros por `AIOS-SCHED-*`. 2) Se concentrado em `0012` (falha interna), investigar exceções recentes/deploy. 3) Se `0004` em massa, ver RB-S-09 (PDP indisponível). |
| RB-S-03 | Overcommit de slot detectado | `SchedulerQuotaOvercommitDetected` | 1) Congelar deploys. 2) Isolar tenant/shard afetado. 3) Auditar scripts Lua de reserva atômica no Redis. 4) Abrir incidente SEV-1; auditoria completa via `025-Audit`. |
| RB-S-04 | Backpressure sustentado/oscilante | `SchedulerBackpressureRejectSustained`, `SchedulerBackpressureFlapping` | 1) Verificar causa raiz de saturação (fila, lag JetStream, capacidade). 2) Se capacidade real insuficiente, escalar Runtime Supervisor/Cluster (007/027). 3) Se oscilação sem causa real, revisar `scheduler.backpressure.defer_threshold`/`reject_threshold` (histerese insuficiente). |
| RB-S-05 | Crescimento de fila / starvation | `SchedulerQueueDepthGrowing`, `SchedulerOldestWaitHigh` | 1) Identificar tenant/classe/shard dominante. 2) Verificar `DispatchLoop` (lease travado?). 3) Verificar capacidade residual do shard. 4) Considerar rebalanceamento manual se shard específico. |
| RB-S-06 | Violação de fairness | `SchedulerFairnessViolation` | 1) Identificar tenant abaixo de `min_share`. 2) Verificar aging/boost saturado. 3) Avaliar aumento pontual de `min_share`/cota via 022 se legítimo. |
| RB-S-07 | Overhead/thrashing de preempção | `SchedulerPreemptionOverheadHigh`, `SchedulerPreemptionThrashing` | 1) Verificar cooldown/histerese (ADR-0094) configurado corretamente. 2) Identificar par de tarefas em ciclo de preempção mútua. 3) Ajustar `scheduler.preemption.grace_period_ms` se aplicável. |
| RB-S-08 | Backlog/perda de outbox/eventos | `SchedulerOutboxBacklogGrowing`, `SchedulerOutboxLagHigh`, `SchedulerEventLossDetected` | 1) Checar saúde do NATS/JetStream. 2) Checar processo/thread do `EventPublisher` (relay vivo?). 3) Se `events_lost_total > 0`, abrir incidente SEV-1 imediatamente (viola NFR-007). |
| RB-S-09 | Dependência externa degradada | `SchedulerDependencyCircuitOpen`, `SchedulerCapacityStale` | 1) Confirmar saúde do dependente (022/026/027/007). 2) Verificar modo de falha aplicado (`_DESIGN_BRIEF.md` §9 — fail-safe/default-deny). 3) Escalar ao dono do módulo dependente. |
| RB-S-10 | Divergência de tenant detectada | `SchedulerTenantMismatchDetected` | 1) Congelar a rota/cliente que originou a divergência. 2) Investigar possível spoofing ou bug de propagação de claim. 3) Auditoria completa via `025-Audit` — incidente de segurança. |
| RB-S-11 | Réplica fora do ar / gap de ownership de shard | `SchedulerReplicaDown`, `SchedulerShardOwnershipGap` | 1) Checar readiness/liveness. 2) Confirmar redistribuição de ownership de shard entre réplicas remanescentes. 3) Rolling restart se aplicável; validar RTO ≤ 15 min (NFR-008). |

## 7. Referências

- Catálogo de métricas: `./Metrics.md`
- Logging estruturado e correlação: `./Logging.md`
- Estratégia de testes de observabilidade: `./Testing.md`
- Metodologia de carga/benchmark: `./Benchmark.md`
- Modos de falha e recuperação: `_DESIGN_BRIEF.md` §9 (`FailureRecovery.md`, quando produzido)
- Infraestrutura transversal: `../024-Observability/README.md`
- Auditoria imutável: `../025-Audit/README.md`
- Contratos de correlação: `../003-RFC/RFC-0001-Architecture-Baseline.md` §5.6
- Design Brief (fonte de verdade): `./_DESIGN_BRIEF.md` §7.2, §9, §10
- NFRs de origem: `./NonFunctionalRequirements.md`
