---
Documento: Monitoring
Módulo: 008-Agent-Lifecycle
Status: Draft
Versão: 0.1
Última atualização: 2026-07-20
Responsável (RACI-A): Arquiteto do Módulo 008 — Agent Lifecycle
ADRs relacionados: ADR-0080 (FSM canônica), ADR-0081 (Hibernação em MinIO+PG), ADR-0083 (WarmPool), ADR-0084 (Lease/fencing), ADR-0085 (Migração como saga), ADR-0087 (Reconciliação)
RFCs relacionados: RFC-0001 (baseline, §5.6 correlação), RFC-0080 (a propor — Cold Agent Hibernation & Checkpoint/Restore Protocol)
Depende de: 024-Observability, 025-Audit, 009-Scheduler, 007-Agent-Runtime, 027-Cluster
---

# 008-Agent-Lifecycle — Monitoring

> Este documento define **o que observar, como observar e o que fazer quando algo
> sai do esperado** no Lifecycle Manager (008). Ele consome a infraestrutura
> transversal do `024-Observability` (Prometheus/Grafana/OTel Collector) e do
> `025-Audit`, e **não redefine** os contratos de correlação (`traceparent`,
> `X-AIOS-Tenant`) já fixados na `../003-RFC/RFC-0001-Architecture-Baseline.md`
> §5.6. O catálogo formal de métricas está em `./Metrics.md`; este documento
> cobre dashboards, golden signals, SLO/SLA/SLI, regras de alerta Prometheus e
> runbooks.

## 1. Objetivo e Escopo

O monitoramento do módulo 008 DEVE responder, em tempo real, quatro perguntas
operacionais centrais (derivadas de `_DESIGN_BRIEF.md` §7.2 e §9):

1. **A FSM canônica do agente está sendo honrada nas metas de latência e
   throughput?** (spawn/materialização `p99 ≤ 250 ms`, transições
   `p99 ≤ 20 ms`, checkpoint `p99 ≤ 500 ms`, migração `p99 ≤ 2 s`).
2. **O sistema escala rumo a 10⁶ agentes com ≤ 5% ativos em RAM**, sem
   degradação de latência de materialização (`wake`) à medida que a população
   fria cresce?
3. **A serialização por lease está saudável** (sem contenção excessiva, sem
   split-brain, sem coordenador travado segurando um agente indefinidamente)?
4. **A durabilidade e a integridade dos checkpoints estão garantidas** (RTO ≤
   15 min, RPO ≤ 5 min, `0` restore de checkpoint corrompido não detectado)?

**Escopo.** Este documento cobre exclusivamente a observabilidade **do próprio
serviço 008** — os 17 componentes internos do brief §2 (`LifecycleApiSurface`,
`LifecyclePolicyEnforcer`, `LifecycleCoordinator`, `StateMachineEngine`,
`AcbStore`, `SpawnManager`, `WarmPoolManager`, `HibernationController`,
`CheckpointService`, `SnapshotStore`, `MigrationOrchestrator`, `LeaseManager`,
`ReconciliationController`, `LifecycleEventPublisher`, `LifecycleEventConsumer`,
`TombstoneManager`, `LifecycleTelemetry`). O módulo monitora apenas sua
**fronteira de integração** com o Kernel (006), o Scheduler (009), o Runtime
Supervisor (007), o Memory (010) e o Cluster (027) — latência de cliente, taxa
de erro e estado de circuit breaker dessas dependências.

## 2. Golden Signals do Módulo 008

| Golden Signal | Definição no contexto do 008 | Métrica primária (ver `Metrics.md`) | Meta associada |
|----------------|----------------------------------|--------------------------------------|-----------------|
| **Latência** | Tempo de materialização (`spawn`/`wake`), de decisão de transição, de checkpoint e de migração. | `aios_lifecycle_spawn_duration_ms`, `aios_lifecycle_transition_duration_ms`, `aios_lifecycle_checkpoint_duration_ms`, `aios_lifecycle_migration_duration_ms` | NFR-001, NFR-002, NFR-003, NFR-010 |
| **Tráfego** | Taxa de transições de FSM processadas por segundo, por tipo de gatilho (`spawn`,`suspend`,`hibernate`,`wake`,`migrate`,`terminate`). | `aios_lifecycle_transitions_total` (rate) | NFR-008 (≥ 50.000 transições/s por cluster) |
| **Erros** | Fração de operações que terminam em `AIOS-LIFECYCLE-<NNNN>`, por código. | `aios_lifecycle_operations_total{status="error"}` | orçamento de erro derivado de NFR-007 |
| **Saturação** | Ocupação de recursos finitos do caminho quente: leases em contenção, WarmPool esgotado, backlog do Outbox, pressão de RAM disparando hibernação em massa. | `aios_lifecycle_lease_contention_ratio`, `aios_lifecycle_warmpool_available`, `aios_lifecycle_outbox_pending`, `aios_lifecycle_hibernation_triggered_total{reason="ram_pressure"}` | NFR-004 (≤5% agentes ativos em RAM), NFR-011 |

## 3. SLOs / SLIs e Orçamento de Erro

Os SLOs do módulo 008 **DEVEM** ser rastreados como frações de tempo dentro do
orçamento de erro (*error budget*), com janela móvel de 30 dias, alinhados à
`NonFunctionalRequirements.md`.

| SLO | SLI (fonte) | Meta | Janela | Orçamento de erro (30d) |
|-----|-------------|------|--------|---------------------------|
| Disponibilidade do serviço 008 | `up{job="agent-lifecycle"}` combinado com taxa de erro 5xx/gRPC-UNAVAILABLE | ≥ 99,95% (NFR-007) | 30d rolling | 21,6 min de indisponibilidade |
| Latência de spawn/materialização | `histogram_quantile(0.99, aios_lifecycle_spawn_duration_ms)` | p99 ≤ 250 ms, p50 ≤ 80 ms | 30d rolling | ≤ 1% das amostras acima do limiar |
| Latência de decisão de transição | `histogram_quantile(0.99, aios_lifecycle_transition_duration_ms)` | ≤ 20 ms | 30d rolling | ≤ 1% das amostras acima do limiar |
| Latência de checkpoint | `histogram_quantile(0.99, aios_lifecycle_checkpoint_duration_ms)` | ≤ 500 ms (working memory ≤ 64 MiB) | 30d rolling | ≤ 1% das amostras acima do limiar |
| Latência de migração | `histogram_quantile(0.99, aios_lifecycle_migration_duration_ms)` | ≤ 2 s (agente ≤ 64 MiB, mesmo DC) | 30d rolling | ≤ 1% das amostras acima do limiar |
| Escala de agentes frios | `aios_lifecycle_agents_total{state="Hibernated"} / aios_lifecycle_agents_total` | ≥ 95% do total em `Hibernated` a 10⁶ agentes | contínuo | monitorado, não é burn-rate clássico |
| Recuperabilidade (RTO) | Tempo `runtime.exited` → `woken`/`resumed` | ≤ 15 min | contínuo | qualquer violação é revisão de incidente |
| Durabilidade (RPO) | Intervalo entre checkpoints válidos consecutivos | ≤ 5 min | contínuo | qualquer violação é revisão de incidente |
| Integridade de snapshot | `aios_lifecycle_checkpoint_corrupt_total` | 0 (NFR-009) | contínuo | qualquer valor > 0 é incidente SEV-1 |
| Throughput de transições | `rate(aios_lifecycle_transitions_total[1m])` | ≥ 50.000/s por cluster | 30d rolling | degradação sustentada é revisão de capacidade |
| Idempotência | Repetição de `Idempotency-Key` produz efeito duplicado | 0 (NFR-013) | contínuo | qualquer valor > 0 é incidente SEV-1 |

**Multi-window multi-burn-rate.** Alertas de SLO de disponibilidade e latência
DEVEM usar o padrão de duas janelas (curta e longa) para equilibrar
sensibilidade e ruído, conforme detalhado na §5.

## 4. Dashboards (Grafana)

Os dashboards do módulo 008 são provisionados como código (JSON versionado em
`029-Operations`) e publicados no Grafana provido por `024-Observability`. Todo
painel DEVE permitir filtro por `tenant` e por `state`/`trigger` (template
variables).

| Dashboard | Público-alvo | Painéis principais |
|-----------|--------------|---------------------|
| **Lifecycle Overview** | On-call, SRE | Transições/s por gatilho; latência p50/p95/p99 de spawn/transição/checkpoint/migração; taxa de erro por código `AIOS-LIFECYCLE-*`; réplicas saudáveis; burn rate de SLO. |
| **ACB State Distribution** | Plataforma, Capacity Planning | Distribuição de ACBs por `state` (Created/Ready/Running/Suspended/Hibernated/Migrating/Terminated/Failed); % de agentes frios vs. quentes; tendência de crescimento rumo a 10⁶. |
| **Hibernation & WarmPool** | Plataforma, SRE | `aios_lifecycle_hibernation_triggered_total` por `reason` (`idle`\|`ram_pressure`); latência de `wake`; ocupação do WarmPool por tenant/shard; reservas mín/máx configuradas vs. reais. |
| **Checkpoint & Snapshot** | Plataforma, Segurança | Duração de checkpoint por tamanho de working memory; taxa de `content_hash` divergente; distribuição por `storage_tier` (hot/warm/cold); idade média do `last_checkpoint_id`. |
| **Migration** | Plataforma, Cluster | `MigrationJob` por `phase`/`status`; latência de saga por fase; taxa de compensação; migrações em massa disparadas por `cluster.node.draining`. |
| **Lease & Reconciliation** | Plataforma, SRE | Contenção de lease (`AIOS-LIFECYCLE-0003`/s); fencing token rejeitado/s; lag de reconciliação; ACBs corrigidos por drift/min. |
| **Outbox & Events** | Plataforma, SRE | `aios_lifecycle_outbox_pending`; lag do relay; taxa de publicação por subject `agent.lifecycle.*`; erros de publicação no JetStream. |

### 4.1 Mockup ASCII — Dashboard "Lifecycle Overview"

```
┌───────────────────────────── Lifecycle Overview  (tenant=*, state=*) ─────────────────────────────┐
│  Transições/s por gatilho              │  Latência p99 (ms)                                       │
│  ▇▇▇▇▇▇▇▇▇▇ spawn       3.1k/s         │  spawn        ████████████████░░░░  231 ms  (SLO 250)     │
│  ▇▇▇▇▇▇▇▇ hibernate     2.4k/s         │  transition   ███░░░░░░░░░░░░░░░░░   14 ms  (SLO 20)      │
│  ▇▇▇▇▇▇▇▇▇▇▇ wake       2.9k/s         │  checkpoint   █████████░░░░░░░░░░░  380 ms  (SLO 500)     │
│  ▇▇ migrate             0.3k/s         │  migration    ██████████████░░░░░ 1720 ms  (SLO 2000)     │
├─────────────────────────────────────────┼───────────────────────────────────────────────────────────┤
│  Taxa de erro por código (5m)           │  Burn rate do SLO de disponibilidade                     │
│  LIFECYCLE-0002 (transição inválida) 0.05% │  janela 1h : 0.7x  (ok)                                │
│  LIFECYCLE-0003 (lease ocupado)      0.9%  │  janela 6h : 1.0x  (ok)                                │
│  LIFECYCLE-0007 (checkpoint corrupto) 0.0% │  janela 3d : 0.9x  (ok)                                │
├─────────────────────────────────────────┴───────────────────────────────────────────────────────────┤
│  Réplicas saudáveis: 8/8   % agentes frios: 96.2%   Outbox pending: 118   WarmPool disp.: 71%       │
└───────────────────────────────────────────────────────────────────────────────────────────────────┘
```

## 5. Regras de Alerta (Prometheus)

> `for` é o tempo mínimo em que a condição DEVE persistir antes de disparar,
> evitando *flapping*. Severidade `page` aciona on-call; `ticket` gera item de
> backlog; `info` é apenas painel/registro.

| Alerta | Expressão PromQL (resumida) | `for` | Severidade | Runbook |
|--------|------------------------------|-------|------------|---------|
| `LifecycleSpawnLatencyHigh` | `histogram_quantile(0.99, sum(rate(aios_lifecycle_spawn_duration_ms_bucket[5m])) by (le)) > 250` | 10m | page | RB-LC-01 |
| `LifecycleTransitionLatencyHigh` | `histogram_quantile(0.99, sum(rate(aios_lifecycle_transition_duration_ms_bucket[5m])) by (le)) > 20` | 10m | page | RB-LC-01 |
| `LifecycleCheckpointLatencyHigh` | `histogram_quantile(0.99, sum(rate(aios_lifecycle_checkpoint_duration_ms_bucket[5m])) by (le)) > 500` | 10m | ticket | RB-LC-02 |
| `LifecycleMigrationLatencyHigh` | `histogram_quantile(0.99, sum(rate(aios_lifecycle_migration_duration_ms_bucket[5m])) by (le)) > 2000` | 10m | ticket | RB-LC-03 |
| `LifecycleErrorRateHigh` | `sum(rate(aios_lifecycle_operations_total{status="error"}[5m])) / sum(rate(aios_lifecycle_operations_total[5m])) > 0.01` | 5m | page | RB-LC-04 |
| `LifecycleLeaseContentionHigh` | `sum(rate(aios_lifecycle_lease_rejections_total[5m])) / sum(rate(aios_lifecycle_lease_acquisitions_total[5m])) > 0.05` | 10m | ticket | RB-LC-05 |
| `LifecycleFencingTokenRejectSpike` | `sum(rate(aios_lifecycle_fencing_token_rejections_total[5m])) > 10` | 5m | page (SEV-1) | RB-LC-06 |
| `LifecycleHibernationStorm` | `sum(rate(aios_lifecycle_hibernation_triggered_total{reason="ram_pressure"}[5m])) > 500` | 5m | page | RB-LC-07 |
| `LifecycleCheckpointCorruptionDetected` | `increase(aios_lifecycle_checkpoint_corrupt_total[10m]) > 0` | 0s | page (SEV-1) | RB-LC-08 |
| `LifecycleWarmPoolExhausted` | `aios_lifecycle_warmpool_available / aios_lifecycle_warmpool_capacity < 0.1` | 5m | page | RB-LC-09 |
| `LifecycleOutboxBacklogGrowing` | `aios_lifecycle_outbox_pending > 1000` | 5m | page | RB-LC-10 |
| `LifecycleOutboxLagHigh` | `aios_lifecycle_outbox_lag_ms > 5000` | 5m | page | RB-LC-10 |
| `LifecycleReconciliationDriftHigh` | `sum(rate(aios_lifecycle_reconcile_corrections_total[15m])) > 100` | 15m | ticket | RB-LC-11 |
| `LifecycleAcbStuckInMigrating` | `max(time() - aios_lifecycle_state_entered_timestamp{state="Migrating"}) > 300` | 5m | page | RB-LC-12 |
| `LifecycleAcbStuckInHibernated` (wake nunca concluído) | `max(time() - aios_lifecycle_wake_requested_timestamp) - max(time() - aios_lifecycle_wake_completed_timestamp) > 30` | 5m | ticket | RB-LC-13 |
| `LifecycleMigrationCompensationSpike` | `sum(rate(aios_lifecycle_migration_jobs_total{status="compensated"}[15m])) > 5` | 15m | ticket | RB-LC-03 |
| `LifecycleReplicaDown` | `up{job="agent-lifecycle"} == 0` | 2m | page | RB-LC-14 |
| `LifecycleColdScaleTargetMissed` | `aios_lifecycle_agents_total{state="Hibernated"} / aios_lifecycle_agents_total < 0.95 and aios_lifecycle_agents_total > 100000` | 30m | ticket | RB-LC-15 |
| **Burn-rate SLO (rápido)** | curta: `error_rate_5m > 14.4 * (1 - 0.9995)`; longa: `error_rate_1h > 14.4 * (1 - 0.9995)` (ambas verdadeiras) | 5m | page | RB-LC-04 |
| **Burn-rate SLO (lento)** | curta: `error_rate_1h > 6 * (1 - 0.9995)`; longa: `error_rate_6h > 6 * (1 - 0.9995)` | 30m | ticket | RB-LC-04 |

## 6. Runbooks Associados

> Os runbooks completos (passo-a-passo executável) residem em
> `../029-Operations/Runbooks/` (fonte operacional); este documento mantém
> apenas o resumo e o vínculo alerta → runbook.

| ID | Runbook | Gatilho | Resumo dos passos |
|----|---------|---------|---------------------|
| RB-LC-01 | Latência elevada de spawn/transição | `LifecycleSpawnLatencyHigh`, `LifecycleTransitionLatencyHigh` | 1) Checar ocupação do WarmPool (`SpawnManager`). 2) Checar latência do `AcbStore` (Redis/PostgreSQL). 3) Checar CB do Scheduler (009)/Runtime Supervisor (007). 4) Escalar réplicas se saturação de CPU confirmada. |
| RB-LC-02 | Checkpoint lento | `LifecycleCheckpointLatencyHigh` | 1) Checar tamanho médio de working memory serializada. 2) Checar latência do MinIO/PostgreSQL (`SnapshotStore`). 3) Validar codec (`checkpoint.codec`) e compressão. |
| RB-LC-03 | Migração lenta ou compensada | `LifecycleMigrationLatencyHigh`, `LifecycleMigrationCompensationSpike` | 1) Verificar saúde do nó/shard destino (027). 2) Checar `migration.saga_timeout_ms`. 3) Investigar rede entre origem/destino. 4) Confirmar que a compensação não deixou o agente órfão. |
| RB-LC-04 | Taxa de erro elevada | `LifecycleErrorRateHigh`, burn-rate | 1) Segmentar erros por `AIOS-LIFECYCLE-*`. 2) Se concentrado em `-0005` (admissão negada), correlacionar com Scheduler (009). 3) Se `-0014` (storage indisponível), ver RB-LC-10. |
| RB-LC-05 | Contenção de lease elevada | `LifecycleLeaseContentionHigh` | 1) Identificar agentes com maior taxa de transições concorrentes. 2) Checar `lease.ttl_ms`/`lease.renew_ms`. 3) Avaliar se clientes estão retransmitindo comandos duplicados sem backoff. |
| RB-LC-06 | Rejeição de fencing token (possível split-brain) | `LifecycleFencingTokenRejectSpike` | 1) Congelar deploys. 2) Identificar coordenadores concorrentes (logs por `holder`). 3) Confirmar isolamento de rede/partição. Incidente de integridade SEV-1. |
| RB-LC-07 | Storm de hibernação por pressão de RAM | `LifecycleHibernationStorm` | 1) Checar `hibernation.ram_pressure_threshold_pct`. 2) Verificar se admissão (009) está sendo aplicada com backpressure. 3) Priorizar hibernação por `priority_class`. |
| RB-LC-08 | Corrupção de checkpoint detectada | `LifecycleCheckpointCorruptionDetected` | 1) Isolar o(s) `checkpoint_id` afetado(s). 2) Tentar checkpoint anterior válido. 3) Auditoria completa via `025-Audit`. Incidente SEV-1 (viola NFR-009). |
| RB-LC-09 | WarmPool esgotado | `LifecycleWarmPoolExhausted` | 1) Checar `warmpool.max_per_shard`. 2) Avaliar aumento temporário de reserva. 3) Correlacionar com pico de `spawn` (W2 do `Benchmark.md`). |
| RB-LC-10 | Backlog/perda do Outbox | `LifecycleOutboxBacklogGrowing`, `LifecycleOutboxLagHigh` | 1) Checar saúde do NATS/JetStream. 2) Checar o *relay* do Outbox (processo vivo?). 3) Se detectar perda irrecuperável, abrir incidente SEV-1 (viola FR-008). |
| RB-LC-11 | Reconciliação corrigindo excesso de deriva | `LifecycleReconciliationDriftHigh` | 1) Identificar padrão comum (ex.: runtime morto sem heartbeat). 2) Checar `reconcile.runtime_dead_after_ms`. 3) Correlacionar com incidentes do Runtime Supervisor (007). |
| RB-LC-12 | ACB preso em `Migrating` | `LifecycleAcbStuckInMigrating` | 1) Consultar `MigrationJob.phase`. 2) Verificar timeout de saga. 3) Forçar compensação administrativa (com auditoria) se travado. |
| RB-LC-13 | `wake` não concluído | `LifecycleAcbStuckInHibernated` | 1) Checar integridade do checkpoint (`content_hash`). 2) Checar admissão do Scheduler (009). 3) Verificar timeout de materialização. |
| RB-LC-14 | Réplica fora do ar | `LifecycleReplicaDown` | 1) Checar readiness/liveness. 2) Checar dependências críticas (PostgreSQL/Redis/MinIO). 3) Rolling restart se aplicável. |
| RB-LC-15 | Meta de escala fria não atingida | `LifecycleColdScaleTargetMissed` | 1) Revisar `hibernation.idle_ttl_s`. 2) Verificar se `HibernationController` está processando a fila de candidatos a hibernação. 3) Revisão de capacidade com Scheduler (009). |

## 7. Referências

- Catálogo de métricas: `./Metrics.md`
- Logging estruturado e correlação: `./Logging.md`
- Estratégia de testes de observabilidade: `./Testing.md`
- Metodologia de carga/benchmark: `./Benchmark.md`
- Modos de falha e recuperação: `./FailureRecovery.md`
- Infraestrutura transversal: `../024-Observability/README.md`
- Auditoria imutável: `../025-Audit/README.md`
- Contratos de correlação: `../003-RFC/RFC-0001-Architecture-Baseline.md` §5.6
- Design Brief (fonte de verdade): `./_DESIGN_BRIEF.md` §7.2, §9, §10
