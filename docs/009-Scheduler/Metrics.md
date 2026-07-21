---
Documento: Metrics
Módulo: 009-Scheduler
Status: Draft
Versão: 0.1
Última atualização: 2026-07-20
Responsável (RACI-A): Arquiteto do Módulo 009-Scheduler
ADRs relacionados: ADR-0090 (admission control), ADR-0091 (score multiobjetivo), ADR-0092 (sharding), ADR-0093 (filas ZSET/aging), ADR-0094 (preempção compensável), ADR-0095 (backpressure em três níveis), ADR-0096 (`ISchedulingPolicy`), ADR-0097 (outbox transacional)
RFCs relacionados: RFC-0001 (baseline); RFC-0090, RFC-0091 (a propor)
Depende de: 024-Observability, 006-Kernel, 022-Policy, 026-Cost-Optimizer, 027-Cluster
---

# 009-Scheduler — Metrics

> Catálogo normativo de métricas OpenTelemetry/Prometheus emitidas pelo
> Scheduler cognitivo. Toda métrica DEVE seguir a convenção
> `aios_<subsistema>_<nome>_<unidade>` (`../_templates/MODULE_TEMPLATE.md`), com
> `<subsistema> = scheduler`. Este catálogo é a fonte usada por `./Monitoring.md`
> (dashboards/alertas) e por `./Benchmark.md` (KPIs de carga). Nenhuma métrica
> aqui introduz componente, estado ou campo que contradiga `_DESIGN_BRIEF.md`.

## 1. Convenções

| Convenção | Regra |
|-----------|-------|
| Prefixo | `aios_scheduler_` para toda métrica emitida pelo serviço Scheduler. |
| Unidade no nome | Sufixo obrigatório: `_ms` (latência), `_total` (contador monotônico), `_ratio` (0–1, podendo exceder 1 em overshoot transitório), sem sufixo para gauges de contagem/enum. |
| Tipo | `counter` (só cresce), `gauge` (sobe/desce), `histogram` (distribuição, gera `_bucket`/`_sum`/`_count`). |
| Labels obrigatórios transversais | `tenant` (quando cardinalidade permitir — ver §12), `service_class` (∈ `INTERACTIVE`\|`BATCH`\|`BACKGROUND`\|`SYSTEM`, cardinalidade fixa de 4), `shard` (quando aplicável, cardinalidade = `scheduler.shards.count`, default 64). |
| Exportação | Via OTel SDK (.NET 10) → OTel Collector → Prometheus remote-write, conforme `../024-Observability/`. |
| Cardinalidade de `task_urn`/`agent_urn` | **NÃO DEVE** ser label de métrica (cardinalidade ilimitada); usar `trace_id`/`decision_id` (logs/traces, ver `./Logging.md`) para granularidade por tarefa individual. |
| Buckets de histograma | Explícitos (não exponenciais automáticos), de forma que os limiares de SLO (5 ms, 20 ms, 50 ms, 250 ms) caiam exatamente numa fronteira, permitindo `histogram_quantile` preciso. |

## 2. Métricas de Admissão (`SchedulingGateway`, `AdmissionController`)

| Métrica | Tipo | Labels | Unidade | Semântica |
|---------|------|--------|---------|-----------|
| `aios_scheduler_requests_total` | counter | `tenant`, `service_class`, `outcome` (`admitted`\|`rejected`) | contagem | Total de `SchedulingRequest` processadas pelo `AdmissionController`. Base do golden signal "tráfego". |
| `aios_scheduler_decision_latency_ms` | histogram | `tenant`, `service_class` | ms | Latência fim-a-fim de `Submit` (recebimento no `SchedulingGateway` → `SchedulingDecision` retornada). SLI de NFR-001 (p99 ≤ 20 ms, p50 ≤ 5 ms). Buckets: 1,2,3,5,8,10,15,20,30,50,100,250. |
| `aios_scheduler_admission_rejected_total` | counter | `tenant`, `reason_code` (`AIOS-SCHED-0001`..`0004`,`0005`,`0007`,`0008`) | contagem | Rejeições de admissão por código de erro. Base para SLI de qualidade/segmentação de causa. |
| `aios_scheduler_requests_in_flight` | gauge | `service_class` | contagem | Requisições `Submit` em processamento simultâneo no pipeline (admissão→placement). Indicador de saturação/concorrência interna. |
| `aios_scheduler_idempotency_hits_total` | counter | `tenant`, `method` (`submit`\|`cancel`\|`preempt`\|`report_signal`) | contagem | Requisições deduplicadas via `Idempotency-Key`/`request_id` (resultado reaproveitado). Contribui para NFR-011. |
| `aios_scheduler_idempotency_conflicts_total` | counter | `tenant`, `method` | contagem | Mesma chave, payload divergente (`AIOS-SCHED-0006`). |
| `aios_scheduler_tenant_mismatch_total` | counter | — | contagem | `X-AIOS-Tenant` divergente do claim autenticado (`AIOS-SCHED-0004`). **DEVE permanecer próximo de 0** — indica tentativa de spoofing ou bug de cliente. |

## 3. Métricas de Prioridade/Custo (`PriorityCostEvaluator`, `FeatureExtractor`, `CostEstimator`, `ISchedulingPolicy`)

| Métrica | Tipo | Labels | Unidade | Semântica |
|---------|------|--------|---------|-----------|
| `aios_scheduler_cost_score` | histogram | `tenant`, `service_class` | ratio (adimensional) | Distribuição de `C = α·L + β·$ + γ·E` calculado por decisão. Buckets: 0.05,0.1,0.2,0.3,0.4,0.5,0.7,1.0. |
| `aios_scheduler_effective_priority` | histogram | `tenant`, `service_class` | pontos (0–100+aging) | Distribuição da prioridade efetiva atribuída (score da ZSET). |
| `aios_scheduler_policy_weight` | gauge | `tenant`, `weight` (`alpha`\|`beta`\|`gamma`) | ratio (0–1) | Peso corrente aplicado por tenant (normalizado, `alpha+beta+gamma≈1`). |
| `aios_scheduler_quality_floor_violations_total` | counter | `tenant`, `service_class` | contagem | `quality_floor` inatingível sob restrições correntes (`AIOS-SCHED-0007`). |
| `aios_scheduler_aging_boost_applied` | histogram | `service_class` | pontos | Boost de prioridade aplicado por aging, até `scheduler.aging.max_boost`. SLI complementar de NFR-006 (fairness). |
| `aios_scheduler_edf_expired_total` | counter | `tenant`, `service_class` | contagem | Tarefas que transicionaram `QUEUED → EXPIRED` por deadline vencido (`AIOS-SCHED-0008`). |
| `aios_scheduler_policy_engine_active` | gauge (info) | `tenant`, `engine` (`heuristic`\|`rl`) | booleano (0/1) | Motor de política ativo por tenant (`scheduler.policy.engine`). Métrica-alvo de NFR-013 (rastreio de qual política decide). |
| `aios_scheduler_feature_extraction_duration_ms` | histogram | — | ms | Duração da extração do `feature_vector` (`FeatureExtractor`). Componente do orçamento de NFR-001. |
| `aios_scheduler_cost_estimator_cache_hit_ratio` | gauge | — | ratio (0–1) | Taxa de acerto do cache local do `CostEstimator` (evita chamada síncrona a 026 no caminho quente). |

## 4. Métricas de Placement (`PlacementEngine`, `ShardRouter`)

| Métrica | Tipo | Labels | Unidade | Semântica |
|---------|------|--------|---------|-----------|
| `aios_scheduler_placement_duration_ms` | histogram | — | ms | Duração da resolução de shard/nó/pool. |
| `aios_scheduler_placement_failures_total` | counter | `tenant`, `reason` (`no_capacity`\|`affinity_conflict`) | contagem | Falhas de placement (`AIOS-SCHED-0010`). |
| `aios_scheduler_shard_load` | gauge | `shard` | contagem | Nº de tarefas correntes (QUEUED+SCHEDULED+RUNNING observado) atribuídas ao shard — detecta desbalanceamento. |
| `aios_scheduler_shard_rebalance_total` | counter | — | contagem | Rebalanceamentos executados após mudança de `scheduler.shards.count` (ADR-0092). |
| `aios_scheduler_shard_rebalance_keys_moved_ratio` | gauge | — | ratio | Fração de chaves remapeadas na última mudança de `N` (alvo ≈ `1/N`, ver NFR-014). |
| `aios_scheduler_affinity_fallback_total` | counter | `tenant` | contagem | Vezes em que uma regra de anti-afinidade foi ignorada por ausência de alternativa com capacidade (fallback documentado). |

## 5. Métricas de Preempção (`PreemptionManager`)

| Métrica | Tipo | Labels | Unidade | Semântica |
|---------|------|--------|---------|-----------|
| `aios_scheduler_preemptions_total` | counter | `tenant`, `service_class`, `initiator_class` | contagem | Preempções efetivadas (`SCHEDULED`/`RUNNING → PREEMPTED`). |
| `aios_scheduler_preemption_ms` | histogram | — | ms | Overhead de decisão de preempção (seleção de vítima até sinalização de suspensão). SLI de NFR-009 (≤ 50 ms). Buckets: 5,10,20,30,50,75,100,200. |
| `aios_scheduler_preemption_grace_period_ms` | histogram | `service_class` | ms | Distribuição do `grace_period_ms` efetivamente concedido. |
| `aios_scheduler_preemption_resume_total` | counter | `tenant` | contagem | Tarefas retomadas de `PREEMPTED → QUEUED` (`resumed`). |
| `aios_scheduler_preemption_cooldown_blocked_total` | counter | — | contagem | Tentativas de escolher vítima dentro do cooldown, bloqueadas pela histerese anti-*thrashing* (ADR-0094). |
| `aios_scheduler_preemption_denied_total` | counter | `reason` (`pdp_deny`\|`no_victim_eligible`) | contagem | Preempções não realizadas por negação do PDP ou ausência de vítima elegível. |

## 6. Métricas de Filas e Dispatch (`PriorityQueueStore`, `DispatchLoop`)

| Métrica | Tipo | Labels | Unidade | Semântica |
|---------|------|--------|---------|-----------|
| `aios_scheduler_queue_depth` | gauge | `tenant`, `service_class`, `shard` | contagem | Profundidade corrente da ZSET `sched:{tenant}:{class}:{shard}`. Insumo direto do `BackpressureController`. |
| `aios_scheduler_queue_wait_ms` | histogram | `service_class` | ms | Tempo entre `enqueue` e `dispatch` de uma tarefa (tempo em `QUEUED`). |
| `aios_scheduler_queue_oldest_wait_ms` | gauge | `tenant`, `service_class`, `shard` | ms | Idade da tarefa mais antiga elegível na fila — insumo de aging/alerta de starvation. |
| `aios_scheduler_dispatch_duration_ms` | histogram | — | ms | Duração de um ciclo do `DispatchLoop` (retirar topo → revalidar capacidade → enviar decisão). |
| `aios_scheduler_dispatch_lease_conflicts_total` | counter | `shard` | contagem | Conflitos de lease de dispatch (duas réplicas tentando retirar o mesmo shard) — deve ser raro sob ownership correto. |
| `aios_scheduler_dispatch_total` | counter | `tenant`, `service_class`, `outcome` (`scheduled`\|`no_capacity_retry`) | contagem | Total de tentativas de dispatch e seu desfecho imediato. |
| `aios_scheduler_fairness_share_ratio` | gauge | `tenant` | ratio (0–1) | Fração de despachos do tenant sobre o total, em janela móvel de 60s. SLI direto de NFR-006 (`min_share`). |
| `aios_scheduler_starvation_boost_saturated_total` | counter | `tenant` | contagem | Vezes em que o aging atingiu `scheduler.aging.max_boost` sem garantir `min_share` — sinal de fairness sob estresse extremo. |

## 7. Métricas de Backpressure (`BackpressureController`)

| Métrica | Tipo | Labels | Unidade | Semântica |
|---------|------|--------|---------|-----------|
| `aios_scheduler_backpressure_level` | gauge | `shard` | enum (0=accept,1=defer,2=reject) | Nível corrente de saturação do shard. |
| `aios_scheduler_backpressure_pressure_ratio` | gauge | `shard` | ratio (0–1) | Pressão combinada (fila + lag JetStream + capacidade) usada para decidir o nível. |
| `aios_scheduler_backpressure_rejections_total` | counter | `tenant`, `shard` | contagem | Admissões negadas por `AIOS-SCHED-0003` (nível `reject`). |
| `aios_scheduler_backpressure_deferred_total` | counter | `tenant`, `shard` | contagem | Tarefas movidas para `DEFERRED` (nível `defer`). |
| `aios_scheduler_retry_after_ms` | gauge | `shard` | ms | Valor corrente de `Retry-After`/`retryAfterMs` sinalizado aos produtores. |
| `aios_scheduler_backpressure_level_changes_total` | counter | `shard`, `from_level`, `to_level` | contagem | Transições de nível — usado para detectar oscilação/flapping de backpressure. |

## 8. Métricas de Cota e Concorrência (`QuotaLedger`)

| Métrica | Tipo | Labels | Unidade | Semântica |
|---------|------|--------|---------|-----------|
| `aios_scheduler_quota_consumed_ratio` | gauge | `tenant`, `service_class` | ratio (0–1+) | Consumo corrente / `max_concurrency` configurado. Valor > 1 indica overshoot transitório (DEVE convergir a ≤1). |
| `aios_scheduler_concurrency_inuse` | gauge | `tenant`, `service_class` | contagem | Slots concorrentes em uso (`sched:quota:{tenant}:{class}:inflight`). |
| `aios_scheduler_quota_reservations_total` | counter | `tenant`, `outcome` (`reserved`\|`released`\|`aborted`) | contagem | Ciclo de vida de reservas atômicas de slot. |
| `aios_scheduler_quota_exceeded_total` | counter | `tenant`, `service_class` | contagem | Cota de concorrência excedida (`AIOS-SCHED-0001`). |
| `aios_scheduler_quota_overcommit_total` | counter | — | contagem | **DEVE permanecer 0** (NFR-005). Qualquer valor > 0 indica falha na reserva atômica — incidente SEV-1. |
| `aios_scheduler_budget_insufficient_total` | counter | `tenant` | contagem | Orçamento insuficiente para custo estimado (`AIOS-SCHED-0002`, via `BudgetClient`). |

## 9. Métricas de Decisão/Outbox/Eventos (`DecisionJournal`, `EventPublisher`)

| Métrica | Tipo | Labels | Unidade | Semântica |
|---------|------|--------|---------|-----------|
| `aios_scheduler_decision_journal_write_ms` | histogram | — | ms | Duração da gravação transacional da `SchedulingDecision` em PostgreSQL. |
| `aios_scheduler_outbox_pending` | gauge | — | contagem | Decisões gravadas aguardando publicação (`published=false`). Métrica-alvo de saturação/durabilidade (NFR-007). |
| `aios_scheduler_outbox_lag_ms` | gauge | — | ms | Idade da mensagem pendente mais antiga no outbox. |
| `aios_scheduler_events_published_total` | counter | `subject` | contagem | Eventos publicados com sucesso no JetStream (`task.execution.*`, `scheduler.*`). |
| `aios_scheduler_events_publish_errors_total` | counter | `subject`, `reason` | contagem | Falhas de publicação (retentáveis). |
| `aios_scheduler_events_lost_total` | counter | — | contagem | **DEVE permanecer 0** (NFR-007). Perda irrecuperável detectada por reconciliação — incidente SEV-1. |
| `aios_scheduler_decisions_recorded_total` | counter | `tenant`, `outcome` | contagem | Total de `SchedulingDecision` persistidas por outcome. SLI de NFR-010 (100% das decisões com proveniência). |

## 10. Métricas de Reconciliação (`ReconciliationWorker`)

| Métrica | Tipo | Labels | Unidade | Semântica |
|---------|------|--------|---------|-----------|
| `aios_scheduler_reconciliation_duration_ms` | histogram | — | ms | Duração de um ciclo de reconciliação (filas × runtime real). |
| `aios_scheduler_leases_expired_total` | counter | `shard` | contagem | Leases de dispatch/ownership expirados sem confirmação. |
| `aios_scheduler_orphan_requeued_total` | counter | `tenant` | contagem | Tarefas `SCHEDULED` sem confirmação de `RUNNING` além do TTL, re-enfileiradas (`ReconciliationWorker`). |
| `aios_scheduler_reconciliation_drift_total` | counter | `dimension` (`queue`\|`quota`\|`capacity`) | contagem | Divergências detectadas e corrigidas entre estado quente (Redis) e realidade observada. |

## 11. Métricas de Dependências e Disponibilidade (`PolicyClient`, `BudgetClient`, `CapacityProvider`, `RuntimeSupervisorClient`, `SchedulerTelemetry`)

| Métrica | Tipo | Labels | Unidade | Semântica |
|---------|------|--------|---------|-----------|
| `aios_scheduler_dependency_latency_ms` | histogram | `dependency` (`022-policy`\|`026-cost-optimizer`\|`027-cluster`\|`007-runtime-supervisor`) | ms | Latência de chamada às dependências externas. |
| `aios_scheduler_dependency_errors_total` | counter | `dependency` | contagem | Erros/timeouts em chamadas às dependências externas. |
| `aios_scheduler_circuit_breaker_state` | gauge | `dependency` | enum (0=closed,1=half_open,2=open) | Estado do circuit breaker por dependência. |
| `aios_scheduler_capacity_free_slots` | gauge | `shard`, `node_id` | contagem | Slots livres reportados por heartbeat (`CapacitySnapshot`). |
| `aios_scheduler_capacity_heartbeat_age_ms` | gauge | `node_id` | ms | Idade do último heartbeat de capacidade — detecta staleness de `CapacityProvider`. |
| `aios_scheduler_up` | gauge (info) | — | booleano (0/1) | Liveness do processo (padrão OTel/Prometheus `up`). |
| `aios_scheduler_replicas_ready` | gauge | — | contagem | Réplicas saudáveis do serviço Scheduler (readiness probe). |
| `aios_scheduler_shard_ownership_active` | gauge | `shard` | booleano (0/1) | Indica se esta réplica é *owner* corrente do shard (lease ativo em Redis). |

## 12. Mapeamento Métrica → Requisito Não-Funcional

| NFR | Métrica(s) usada(s) como SLI |
|-----|-------------------------------|
| NFR-001 (decisão p99 ≤ 20ms / p50 ≤ 5ms) | `aios_scheduler_decision_latency_ms` |
| NFR-002 (throughput ≥ 50k/s) | `rate(aios_scheduler_requests_total[1m])` |
| NFR-003 (disponibilidade ≥ 99,95%) | `aios_scheduler_up`, `aios_scheduler_replicas_ready` |
| NFR-004 (escala 10⁶⁺ agentes) | `aios_scheduler_shard_load`, benchmark de escala (`Benchmark.md`) |
| NFR-005 (0 overcommit) | `aios_scheduler_quota_overcommit_total` |
| NFR-006 (fairness ≥ min_share) | `aios_scheduler_fairness_share_ratio`, `aios_scheduler_starvation_boost_saturated_total` |
| NFR-007 (RPO ≤ 5 min) | `aios_scheduler_outbox_lag_ms`, `aios_scheduler_events_lost_total` |
| NFR-008 (RTO ≤ 15 min) | `aios_scheduler_replicas_ready`, drill de failover (`FailureRecovery.md`) |
| NFR-009 (preempção ≤ 50 ms) | `aios_scheduler_preemption_ms` |
| NFR-010 (100% decisões com proveniência) | `aios_scheduler_decisions_recorded_total` vs. `aios_scheduler_requests_total` |
| NFR-011 (0 admissões duplicadas) | `aios_scheduler_idempotency_hits_total` vs. `aios_scheduler_idempotency_conflicts_total` |
| NFR-012 (100% via PEP→PDP) | `aios_scheduler_dependency_latency_ms{dependency="022-policy"}`, auditoria 025 |
| NFR-013 (troca de política sem quebra) | `aios_scheduler_policy_engine_active` + testes de contrato (`Testing.md`) |
| NFR-014 (rebalanceamento sem downtime) | `aios_scheduler_shard_rebalance_keys_moved_ratio` |
| NFR-016 (isolamento multi-tenant) | `aios_scheduler_tenant_mismatch_total` |
| NFR-017 (hot reload ≤ 5 s) | tempo entre atualização de `aios_scheduler_policy_weight` e efeito observado |

## 13. Cuidados de Cardinalidade

- `tenant` DEVE ser incluído apenas em métricas cuja cardinalidade de tenants
  ativos seja operacionalmente administrável; em ambientes com milhares de
  tenants, agregações por tenant DEVEM usar *recording rules* do Prometheus em
  vez de séries brutas de alta cardinalidade.
- `task_urn`/`agent_urn`/`request_id`/`decision_id` **NÃO DEVEM** ser labels —
  usar `trace_id` (logs/traces, `./Logging.md`) para investigação por tarefa
  individual.
- `shard` tem cardinalidade fixa e conhecida (`scheduler.shards.count`, default
  64, máx. 4096) — seguro como label em métricas de fila/placement/backpressure,
  mas DEVE ser reavaliado se `N` crescer além de algumas centenas (usar
  agregação por faixa de shard nesse caso).
- `service_class` tem cardinalidade fixa e baixa (4 valores) — seguro como
  label em qualquer métrica.
- Histogramas de latência DEVEM usar buckets explícitos alinhados aos
  limiares de SLO (5, 20, 50, 250 ms) para permitir `histogram_quantile`
  preciso nos limiares usados por `Monitoring.md`.

## 14. Referências

- Dashboards e regras de alerta que consomem este catálogo: `./Monitoring.md`
- Metodologia de benchmark e KPIs-alvo: `./Benchmark.md`
- Correlação de logs: `./Logging.md`
- NFRs de origem: `./NonFunctionalRequirements.md`, `_DESIGN_BRIEF.md` §7.2
- Catálogo de erros correlacionado: `./API.md` §5
- Catálogo de eventos correlacionado: `./Events.md`, `_DESIGN_BRIEF.md` §6
- Infraestrutura OTel/Prometheus/Grafana: `../024-Observability/README.md`
