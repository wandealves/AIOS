---
Documento: Metrics
Módulo: 008-Agent-Lifecycle
Status: Draft
Versão: 0.1
Última atualização: 2026-07-20
Responsável (RACI-A): Arquiteto do Módulo 008 — Agent Lifecycle
ADRs relacionados: ADR-0080 (FSM canônica), ADR-0081 (Hibernação), ADR-0082 (Codec de checkpoint), ADR-0083 (WarmPool), ADR-0084 (Lease/fencing), ADR-0085 (Migração/saga), ADR-0086 (Event sourcing/Outbox), ADR-0087 (Reconciliação)
RFCs relacionados: RFC-0001 (baseline)
Depende de: 024-Observability
---

# 008-Agent-Lifecycle — Metrics

> Catálogo normativo de métricas OpenTelemetry/Prometheus emitidas pelo módulo
> 008. Toda métrica DEVE seguir a convenção `aios_<subsistema>_<nome>_<unidade>`
> (`../_templates/MODULE_TEMPLATE.md`), com `<subsistema> = lifecycle`
> (conforme `_DESIGN_BRIEF.md` §0: prefixo `aios_lifecycle_<nome>_<unidade>`).
> Este documento é a fonte usada por `./Monitoring.md` (alertas/dashboards) e
> por `./Benchmark.md` (KPIs de carga). Labels DEVEM incluir `tenant` apenas
> onde a cardinalidade for controlada (ver §9 — cuidados de cardinalidade).

## 1. Convenções

| Convenção | Regra |
|-----------|-------|
| Prefixo | `aios_lifecycle_` para toda métrica emitida pelo serviço 008. |
| Unidade no nome | Sufixo obrigatório: `_ms` (latência), `_total` (contador monotônico), `_ratio` (0..1), `_bytes`, sem sufixo para gauges adimensionais (contagem instantânea). |
| Tipo | `counter` (só cresce), `gauge` (sobe/desce), `histogram` (distribuição, gera `_bucket`/`_sum`/`_count`). |
| Labels obrigatórios transversais | `tenant` (quando cardinalidade permitir — ver §9), `state`/`trigger` (para métricas de FSM), `component` (quando aplicável). |
| Exportação | Via OTel SDK (.NET) → OTel Collector → Prometheus remote-write, conforme `../024-Observability/`. |
| Cardinalidade de `agent_id` | **NÃO DEVE** ser label de métrica (cardinalidade ilimitada); use `trace_id`/logs para granularidade por agente (ver `./Logging.md`). |

## 2. Métricas de FSM e Transições (StateMachineEngine, AcbStore, LifecycleCoordinator)

| Métrica | Tipo | Labels | Unidade | Semântica |
|---------|------|--------|---------|-----------|
| `aios_lifecycle_transitions_total` | counter | `tenant`, `from_state`, `to_state`, `trigger`, `status` (`ok`\|`error`) | contagem | Total de transições processadas (T1..T11 do brief §4.3). Base do golden signal "tráfego"/"erros". |
| `aios_lifecycle_transition_duration_ms` | histogram | `trigger` | ms | Latência de decisão da transição (excl. efeitos I/O). Métrica-alvo de NFR-002. Buckets: 1,2,5,10,15,20,30,50,100,250. |
| `aios_lifecycle_agents_total` | gauge | `tenant`, `state` (∈ `LifecycleState`) | contagem | Nº de agentes correntes por estado. Base de NFR-004 (≤5% ativos em RAM a 10⁶). **Cuidado de cardinalidade:** `tenant` só em painéis agregados de baixa cardinalidade. |
| `aios_lifecycle_generation_total` | counter | `tenant` | contagem | Incrementos de `generation` (spawn/restore/migração). |
| `aios_lifecycle_time_in_state_ms` | histogram | `state` | ms | Tempo decorrido em cada estado antes da transição de saída. |
| `aios_lifecycle_operations_total` | counter | `tenant`, `operation` (`spawn`\|`suspend`\|`resume`\|`hibernate`\|`wake`\|`migrate`\|`checkpoint`\|`restore`\|`terminate`), `status` (`ok`\|`error`) | contagem | Operações de API processadas, particionadas por resultado. |
| `aios_lifecycle_generation_conflict_total` | counter | — | contagem | Escritas rejeitadas por `generation`/`fencing_token` obsoleto (`AIOS-LIFECYCLE-0013`). |

## 3. Métricas de Spawn e WarmPool (SpawnManager, WarmPoolManager)

| Métrica | Tipo | Labels | Unidade | Semântica |
|---------|------|--------|---------|-----------|
| `aios_lifecycle_spawn_duration_ms` | histogram | `tenant`, `warm_pool_hit` (`true`\|`false`) | ms | Latência de materialização (cold/new → `Running`). Métrica-alvo de NFR-001. Buckets: 25,50,80,100,150,200,250,350,500,1000,2500. |
| `aios_lifecycle_spawn_timeout_total` | counter | `tenant` | contagem | Materializações que excederam `spawn.timeout_ms` (`AIOS-LIFECYCLE-0006`). |
| `aios_lifecycle_warmpool_available` | gauge | `tenant`, `shard` | contagem | Runtimes pré-aquecidos disponíveis no pool. |
| `aios_lifecycle_warmpool_capacity` | gauge | `tenant`, `shard` | contagem | Capacidade máxima configurada (`warmpool.max_per_shard`). |
| `aios_lifecycle_warmpool_exhausted_total` | counter | `tenant`, `shard` | contagem | Vezes em que a reserva mínima não pôde ser mantida. |
| `aios_lifecycle_warmpool_hit_ratio` | gauge | `tenant` | ratio (0–1) | Fração de `spawn`s atendidos por runtime pré-aquecido (vs. boot completo). |

## 4. Métricas de Hibernação (HibernationController)

| Métrica | Tipo | Labels | Unidade | Semântica |
|---------|------|--------|---------|-----------|
| `aios_lifecycle_hibernation_triggered_total` | counter | `tenant`, `reason` (`idle`\|`ram_pressure`) | contagem | Hibernações iniciadas. Base de alerta de storm (`Monitoring.md`). |
| `aios_lifecycle_hibernation_duration_ms` | histogram | — | ms | Duração do ciclo completo de hibernação (checkpoint + liberação de RAM). |
| `aios_lifecycle_wake_duration_ms` | histogram | `tenant` | ms | Latência de materialização a partir de frio (`wake`). Alvo compartilhado com NFR-001 (`< 250 ms`). |
| `aios_lifecycle_wake_total` | counter | `tenant`, `status` (`ok`\|`error`) | contagem | Materializações de cold agent. |
| `aios_lifecycle_cold_agents_ram_bytes` | gauge | — | bytes | RSS agregado de agentes hibernados (DEVE tender a 0 por agente frio — verificação de FR-004). |
| `aios_lifecycle_hot_index_size_bytes` | gauge | — | bytes | Tamanho do índice quente em Redis por agente frio (alvo NFR-011: ≤ 4 KiB/agente). |

## 5. Métricas de Checkpoint e Snapshot (CheckpointService, SnapshotStore)

| Métrica | Tipo | Labels | Unidade | Semântica |
|---------|------|--------|---------|-----------|
| `aios_lifecycle_checkpoint_duration_ms` | histogram | `tenant`, `codec` (`json+zstd`\|`msgpack+zstd`) | ms | Duração da criação de checkpoint. Métrica-alvo de NFR-003 (≤500ms/64MiB). |
| `aios_lifecycle_checkpoint_total` | counter | `status` (`ok`\|`error`), `consistency` (`consistent`\|`crash`) | contagem | Total de checkpoints executados. |
| `aios_lifecycle_checkpoint_size_bytes` | histogram | — | bytes | Tamanho do blob comprimido do checkpoint. |
| `aios_lifecycle_checkpoint_corrupt_total` | counter | — | contagem | **DEVE permanecer 0.** Checkpoint com `content_hash` divergente detectado no restore (viola NFR-009). |
| `aios_lifecycle_restore_duration_ms` | histogram | `tenant` | ms | Duração da restauração (ACB + working memory) a partir de checkpoint. |
| `aios_lifecycle_restore_total` | counter | `status` (`ok`\|`error`) | contagem | Total de restores executados. |
| `aios_lifecycle_snapshot_io_errors_total` | counter | `backend` (`minio`\|`postgresql`) | contagem | Falhas de I/O no `SnapshotStore` (`AIOS-LIFECYCLE-0014`). |
| `aios_lifecycle_checkpoint_age_seconds` | gauge | — | segundos | Idade do `last_checkpoint_id` mais antigo entre agentes ativos — proxy de RPO (NFR-006). |

## 6. Métricas de Migração (MigrationOrchestrator)

| Métrica | Tipo | Labels | Unidade | Semântica |
|---------|------|--------|---------|-----------|
| `aios_lifecycle_migration_duration_ms` | histogram | `tenant` | ms | Duração fim-a-fim da saga de migração (quiesce→cutover). Métrica-alvo de NFR-010 (≤2s, ≤64MiB, mesmo DC). |
| `aios_lifecycle_migration_jobs_total` | counter | `status` (`pending`\|`running`\|`succeeded`\|`failed`\|`compensated`) | contagem | Total de `MigrationJob` por desfecho. |
| `aios_lifecycle_migration_phase_duration_ms` | histogram | `phase` (`quiesce`\|`checkpoint`\|`transfer`\|`materialize`\|`cutover`\|`cleanup`) | ms | Duração de cada fase individual da saga. |
| `aios_lifecycle_migration_concurrent` | gauge | `node` | contagem | Migrações concorrentes por nó (verificação de `migration.max_concurrent_per_node`). |
| `aios_lifecycle_migration_batch_total` | counter | `trigger` (`manual`\|`node_draining`) | contagem | Migrações em lote disparadas por drenagem de nó (Cluster/027). |

## 7. Métricas de Lease e Reconciliação (LeaseManager, ReconciliationController)

| Métrica | Tipo | Labels | Unidade | Semântica |
|---------|------|--------|---------|-----------|
| `aios_lifecycle_lease_acquisitions_total` | counter | — | contagem | Leases adquiridos com sucesso. |
| `aios_lifecycle_lease_rejections_total` | counter | — | contagem | Tentativas de aquisição rejeitadas (`AIOS-LIFECYCLE-0003`, outro coordenador ativo). |
| `aios_lifecycle_lease_contention_ratio` | gauge | — | ratio | `rejections / (rejections + acquisitions)` em janela móvel — base de alerta de contenção. |
| `aios_lifecycle_fencing_token_rejections_total` | counter | — | contagem | Escritas rejeitadas por fencing token obsoleto (`AIOS-LIFECYCLE-0013`) — indicador de coordenador rogue/split-brain. |
| `aios_lifecycle_reconcile_cycle_duration_ms` | histogram | — | ms | Duração de um ciclo completo de reconciliação (`reconcile.interval_ms`). |
| `aios_lifecycle_reconcile_corrections_total` | counter | `correction_type` (`runtime_dead`\|`checkpoint_orphan`\|`migration_stuck`\|`state_drift`) | contagem | Correções aplicadas pelo `ReconciliationController`. |
| `aios_lifecycle_reconcile_drift_agents` | gauge | — | contagem | Nº de agentes com estado desejado ≠ observado no ciclo corrente. |

## 8. Métricas de Eventos, Outbox e Retenção (LifecycleEventPublisher/Consumer, TombstoneManager)

| Métrica | Tipo | Labels | Unidade | Semântica |
|---------|------|--------|---------|-----------|
| `aios_lifecycle_outbox_pending` | gauge | — | contagem | Mensagens no Outbox aguardando publicação. Métrica-alvo de saturação. |
| `aios_lifecycle_outbox_lag_ms` | gauge | — | ms | Idade da mensagem pendente mais antiga no Outbox. |
| `aios_lifecycle_outbox_published_total` | counter | `subject` | contagem | Eventos `agent.lifecycle.*` publicados com sucesso no JetStream. |
| `aios_lifecycle_outbox_publish_errors_total` | counter | `subject`, `reason` | contagem | Falhas de publicação (retentáveis). |
| `aios_lifecycle_outbox_lost_total` | counter | — | contagem | **DEVE permanecer 0** (FR-008). Incrementado apenas se detecção de perda irrecuperável ocorrer. |
| `aios_lifecycle_consumer_events_total` | counter | `source_subject`, `status` (`processed`\|`deduped`\|`error`) | contagem | Eventos consumidos de 006/007/009/014/027 e sua ação disparada. |
| `aios_lifecycle_idempotency_hits_total` | counter | `operation` | contagem | Requisições deduplicadas via `Idempotency-Key` (resultado reaproveitado). |
| `aios_lifecycle_tombstone_purged_total` | counter | `tenant` | contagem | Agentes/checkpoints expurgados por retenção/LGPD (`TombstoneManager`). |
| `aios_lifecycle_replicas_ready` | gauge | — | contagem | Réplicas saudáveis do serviço 008 (via readiness probe). |
| `aios_lifecycle_up` | gauge (info) | — | booleano (0/1) | Liveness do processo (padrão OTel/Prometheus `up`). |

## 9. Mapeamento Métrica → Requisito Não-Funcional

| NFR | Métrica(s) usada(s) como SLI |
|-----|-------------------------------|
| NFR-001 (spawn p99 ≤ 250 ms) | `aios_lifecycle_spawn_duration_ms`, `aios_lifecycle_wake_duration_ms` |
| NFR-002 (transição p99 ≤ 20 ms) | `aios_lifecycle_transition_duration_ms` |
| NFR-003 (checkpoint p99 ≤ 500 ms) | `aios_lifecycle_checkpoint_duration_ms` |
| NFR-004 (≥10⁶ agentes, ≤5% ativos em RAM) | `aios_lifecycle_agents_total{state}`, `aios_lifecycle_cold_agents_ram_bytes` |
| NFR-005 (RTO ≤ 15 min) | tempo entre `aios_lifecycle_consumer_events_total{source_subject="agent.runtime.exited"}` e `aios_lifecycle_wake_total`/`aios_lifecycle_transitions_total{to_state="Running"}` |
| NFR-006 (RPO ≤ 5 min) | `aios_lifecycle_checkpoint_age_seconds` |
| NFR-007 (disponibilidade ≥ 99,95%) | `aios_lifecycle_up`, `aios_lifecycle_replicas_ready` |
| NFR-008 (throughput ≥ 50.000 transições/s) | `rate(aios_lifecycle_transitions_total[1m])` |
| NFR-009 (0 restore corrompido não detectado) | `aios_lifecycle_checkpoint_corrupt_total` |
| NFR-010 (migração p99 ≤ 2 s) | `aios_lifecycle_migration_duration_ms` |
| NFR-011 (ACB frio ≤ 4 KiB em Redis) | `aios_lifecycle_hot_index_size_bytes` |
| NFR-012 (100% transições com trace) | correlação via `trace_id` em todas as métricas + `Logging.md` |
| NFR-013 (idempotência, 0 efeito duplicado) | `aios_lifecycle_idempotency_hits_total` vs. reprocessamento observado |

## 10. Cuidados de Cardinalidade

- `tenant` DEVE ser incluído apenas em métricas cuja cardinalidade de tenants
  ativos seja operacionalmente administrável (dezenas a poucas centenas); em
  ambientes com milhares de tenants, agregações por tenant DEVEM usar
  *recording rules* do Prometheus em vez de séries brutas de alta
  cardinalidade.
- `agent_id` **NÃO DEVE** ser label — usar `trace_id` (logs/traces) para
  investigação por agente individual.
- `state`/`trigger` têm cardinalidade fixa e baixa (8 estados, 11 gatilhos) —
  seguros como labels em todas as métricas de FSM.
- `shard` tem cardinalidade limitada pelo número de shards configurados
  (`N` do `hash(tenant_id, agent_id) mod N`) — seguro como label em métricas
  de WarmPool e migração.
- Histogramas DEVEM usar buckets explícitos (não exponenciais automáticos)
  para garantir que os limiares de SLO (ex.: 20 ms, 250 ms, 500 ms, 2000 ms)
  caiam exatamente em uma fronteira de bucket, permitindo `histogram_quantile`
  preciso nos limiares.

## 11. Referências

- Dashboards e regras de alerta que consomem este catálogo: `./Monitoring.md`
- Metodologia de benchmark e KPIs-alvo: `./Benchmark.md`
- Correlação de logs: `./Logging.md`
- NFRs de origem: `./NonFunctionalRequirements.md`, `_DESIGN_BRIEF.md` §7.2
- Infraestrutura OTel/Prometheus: `../024-Observability/README.md`
