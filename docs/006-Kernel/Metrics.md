---
Documento: Metrics
Módulo: 006-Kernel
Status: Draft
Versão: 0.1
Última atualização: 2026-07-20
Responsável (RACI-A): Arquiteto do Módulo 006-Kernel
ADRs relacionados: ADR-0010 (Observabilidade), ADR-0061 (ACB/OCC), ADR-0062 (Token-bucket de cotas), ADR-0063 (PEP/PDP), ADR-0065 (Sharding), ADR-0066 (Outbox)
RFCs relacionados: RFC-0001 (baseline)
Depende de: 024-Observability
---

# 006-Kernel — Metrics

> Catálogo normativo de métricas OpenTelemetry/Prometheus emitidas pelo Kernel.
> Toda métrica DEVE seguir a convenção `aios_<subsistema>_<nome>_<unidade>`
> (`../_templates/MODULE_TEMPLATE.md`), com `<subsistema> = kernel`. Este
> documento é a fonte usada por `./Monitoring.md` (alertas/dashboards) e por
> `./Benchmark.md` (KPIs de carga). Labels DEVEM incluir `tenant` apenas onde a
> cardinalidade for controlada (ver §4 — cuidados de cardinalidade).

## 1. Convenções

| Convenção | Regra |
|-----------|-------|
| Prefixo | `aios_kernel_` para toda métrica emitida pelo serviço Kernel. |
| Unidade no nome | Sufixo obrigatório: `_ms` (latência), `_total` (contador monotônico), `_ratio` (0..1), `_bytes`, sem sufixo para gauges adimensionais (contagem instantânea). |
| Tipo | `counter` (só cresce), `gauge` (sobe/desce), `histogram` (distribuição, gera `_bucket`/`_sum`/`_count`). |
| Labels obrigatórios transversais | `tenant` (quando cardinalidade permitir — ver §4), `verb` (para métricas de syscall), `component` (quando aplicável). |
| Exportação | Via OTel SDK (.NET) → OTel Collector → Prometheus remote-write, conforme `../024-Observability/`. |
| Cardinalidade de `agent_urn` | **NÃO DEVE** ser label de métrica (cardinalidade ilimitada); use `trace_id`/logs para granularidade por agente (ver `./Logging.md`). |

## 2. Métricas de Syscall (SyscallGateway / superfície ABI)

| Métrica | Tipo | Labels | Unidade | Semântica |
|---------|------|--------|---------|-----------|
| `aios_kernel_syscall_total` | counter | `tenant`, `verb`, `decision` (`allow`\|`deny`), `status` (`ok`\|`error`) | contagem | Total de syscalls processadas, particionadas por resultado. Base do golden signal "tráfego"/"erros". |
| `aios_kernel_syscall_latency_ms` | histogram | `tenant`, `verb` | ms | Latência fim-a-fim da syscall (recebimento → resposta). Buckets: 1,2,5,10,20,50,100,250,500,1000,2500,5000. |
| `aios_kernel_spawn_latency_ms` | histogram | `tenant` | ms | Latência específica de `spawn` (cold→`Running`). Métrica-alvo de NFR-001. Buckets: 25,50,100,150,200,250,350,500,1000,2500. |
| `aios_kernel_syscall_payload_bytes` | histogram | `verb` | bytes | Tamanho do payload de entrada da syscall (detecção de payloads anômalos). |
| `aios_kernel_syscall_rejected_total` | counter | `tenant`, `reason` (`schema`\|`abi_version`\|`rate_limit`) | contagem | Rejeições antes do PEP (schema inválido, ABI incompatível, rate-limit do agente). |
| `aios_kernel_syscall_in_flight` | gauge | `verb` | contagem | Syscalls em processamento simultâneo (indicador de saturação/concorrência). |

## 3. Métricas de PEP / Capability (CapabilityEnforcer, PolicyClient)

| Métrica | Tipo | Labels | Unidade | Semântica |
|---------|------|--------|---------|-----------|
| `aios_kernel_pep_decisions_total` | counter | `tenant`, `decision` (`allow`\|`deny`), `reason` (`policy`\|`tenant_mismatch`\|`pdp_unavailable`) | contagem | Toda decisão do PEP. Base para NFR-010 (0 bypass) e detecção de fail-closed em massa. |
| `aios_kernel_pep_decision_ms` | histogram | `cache` (`hit`\|`miss`) | ms | Latência da decisão de capability. Métrica-alvo de NFR-003 (≤8ms hit / ≤25ms miss). |
| `aios_kernel_pep_cache_hit_ratio` | gauge | — | ratio (0–1) | Taxa de acerto do cache de decisões (`kernel.pep.decision_cache_ttl_ms`). |
| `aios_kernel_pep_bypass_total` | counter | — | contagem | **DEVE permanecer 0.** Qualquer valor > 0 indica syscall privilegiada executada sem consulta ao PDP — incidente de segurança SEV-1. |
| `aios_kernel_policy_client_circuit_breaker_state` | gauge | `dependency="022-policy"` | enum (0=closed,1=half_open,2=open) | Estado do circuit breaker do `PolicyClient`. |
| `aios_kernel_policy_bundle_version` | gauge | — | versão (info metric, valor=1, label=`version`) | Versão do *policy bundle* carregado (invalidado por `policy.bundle.updated`). |

## 4. Métricas de ACB e Ciclo de Vida (AcbStore, LifecycleCoordinator)

| Métrica | Tipo | Labels | Unidade | Semântica |
|---------|------|--------|---------|-----------|
| `aios_kernel_acb_state_count` | gauge | `tenant`, `state` (∈ `AcbState`, §4 do brief) | contagem | Nº de ACBs correntes por estado. Soma total = ACBs vivos do tenant. **Cuidado de cardinalidade:** `tenant` só é incluído em painéis agregados de baixa cardinalidade (nº de tenants ativos), nunca combinado com `agent_urn`. |
| `aios_kernel_acb_transitions_total` | counter | `from_state`, `to_state` | contagem | Transições da FSM (T-01..T-13 do brief §4.2). |
| `aios_kernel_acb_state_entered_timestamp` | gauge | `state` | timestamp unix (por ACB amostrado) | Usado para alertar ACB preso (`now - valor > limiar`), ver `Monitoring.md` RB-K-09. |
| `aios_kernel_acb_time_in_state_ms` | histogram | `state` | ms | Tempo decorrido em cada estado antes da transição de saída. |
| `aios_kernel_occ_conflicts_total` | counter | `entity` (`acb`\|`quota`) | contagem | Conflitos de concorrência otimista (`version` divergente). Métrica-alvo de NFR-009/estabilidade. |
| `aios_kernel_occ_retry_total` | counter | `entity`, `outcome` (`resolved`\|`exhausted`) | contagem | Retentativas de OCC e seu desfecho (`kernel.acb.optimistic_retry_max`). |
| `aios_kernel_saga_steps_total` | counter | `saga` (`spawn`\|`kill`\|`checkpoint`), `step`, `status` (`ok`\|`compensated`\|`failed`) | contagem | Passos de saga do `LifecycleCoordinator`. |
| `aios_kernel_hibernation_transitions_total` | counter | `direction` (`to_hibernated`\|`to_admitted`) | contagem | Transições T-08/T-09 (hibernação/resume de agente *cold*). |
| `aios_kernel_active_agents` | gauge | `tenant` | contagem | ACBs em `Running` ou `Suspended` (excluindo `Hibernated`) — proxy de carga ativa por tenant. |
| `aios_kernel_shard_acb_count` | gauge | `shard` | contagem | ACBs por `shard_key` — detecta desbalanceamento de sharding (ADR-0065). |

## 5. Métricas de Cotas (ResourceQuotaManager)

| Métrica | Tipo | Labels | Unidade | Semântica |
|---------|------|--------|---------|-----------|
| `aios_kernel_quota_consumed_ratio` | gauge | `tenant`, `scope` (`tenant`\|`agent`), `resource` (`tokens`\|`cpu_ms`\|`mem_mib`\|`cost_usd`\|`tools`\|`syscalls_per_sec`) | ratio (0–1+) | Consumo corrente / limite da janela. Valor > 1 indica overshoot transitório (deve convergir a ≤1 após enforcement). |
| `aios_kernel_quota_reserved_total` / `_consumed_total` / `_released_total` | counter | `resource`, `scope` | unidade do recurso (tokens, ms, MiB, USD*1e4, contagem) | Ciclo de vida de reserva atômica de cota (reserva→consumo→liberação). |
| `aios_kernel_quota_exceeded_total` | counter | `tenant`, `resource`, `enforcement` (`hard`\|`soft`) | contagem | Vezes que o limite foi excedido. Gatilho de `agent.quota.exceeded`/`agent.quota.warning`. |
| `aios_kernel_quota_overshoot_ratio` | gauge | `resource` | ratio | `(consumo_real - limite) / limite` quando positivo, medido pós-reconciliação. Métrica-alvo de NFR-009 (≤1%). |
| `aios_kernel_quota_reconciliation_duration_ms` | histogram | — | ms | Duração da reconciliação periódica Redis↔PostgreSQL dos contadores de cota. |
| `aios_kernel_quota_reconciliation_drift` | gauge | `resource` | unidade do recurso | Diferença absoluta entre contador Redis e saldo reconciliado no PostgreSQL antes do ajuste. |

## 6. Métricas de Eventos / Outbox (EventEmitter)

| Métrica | Tipo | Labels | Unidade | Semântica |
|---------|------|--------|---------|-----------|
| `aios_kernel_outbox_pending` | gauge | — | contagem | Mensagens no Outbox aguardando publicação (`published=false`). Métrica-alvo de saturação. |
| `aios_kernel_outbox_lag_ms` | gauge | — | ms | Idade da mensagem pendente mais antiga no Outbox. |
| `aios_kernel_outbox_published_total` | counter | `subject` | contagem | Eventos publicados com sucesso no JetStream. |
| `aios_kernel_outbox_publish_errors_total` | counter | `subject`, `reason` | contagem | Falhas de publicação (retentáveis). |
| `aios_kernel_outbox_lost_total` | counter | — | contagem | **DEVE permanecer 0** (NFR-007). Incrementado apenas se detecção de perda irrecuperável ocorrer (auditoria de reconciliação). |
| `aios_kernel_outbox_relay_duration_ms` | histogram | — | ms | Duração de cada ciclo do relay (`kernel.outbox.relay_interval_ms`). |
| `aios_kernel_idempotency_hits_total` | counter | `verb` | contagem | Requisições deduplicadas via `Idempotency-Key` (resultado reaproveitado). |
| `aios_kernel_idempotency_conflicts_total` | counter | `verb` | contagem | Mesma chave com payload divergente (`AIOS-SYSCALL-0002`). |

## 7. Métricas de Brokers de Recurso (ResourceBrokerRouter)

| Métrica | Tipo | Labels | Unidade | Semântica |
|---------|------|--------|---------|-----------|
| `aios_kernel_broker_latency_ms` | histogram | `target_module` (`010`\|`011`\|`012`\|`015`\|`017`), `verb` | ms | Latência de encaminhamento ao módulo dono do recurso. |
| `aios_kernel_broker_errors_total` | counter | `target_module`, `verb` | contagem | Falhas retornadas pelo broker de recurso (`AIOS-KERNEL-0005`). |
| `aios_kernel_circuit_breaker_state` | gauge | `dependency` (`009-scheduler`\|`008-lifecycle`\|`010`\|`011`\|`012`\|`015`\|`017`\|`022-policy`) | enum (0=closed,1=half_open,2=open) | Estado do circuit breaker por dependência externa. |
| `aios_kernel_bulkhead_rejected_total` | counter | `dependency` | contagem | Rejeições por saturação do compartimento de isolamento (bulkhead). |

## 8. Métricas de Checkpoint e Telemetria (CheckpointManager, KernelTelemetry)

| Métrica | Tipo | Labels | Unidade | Semântica |
|---------|------|--------|---------|-----------|
| `aios_kernel_checkpoint_duration_ms` | histogram | `tenant` | ms | Duração do `checkpoint` (coleta de ponteiros + snapshot durável). |
| `aios_kernel_checkpoint_total` | counter | `status` (`ok`\|`error`) | contagem | Total de checkpoints executados. |
| `aios_kernel_audit_forward_total` | counter | `status` (`ok`\|`error`) | contagem | Encaminhamento de registros de auditoria a `025-Audit`. |
| `aios_kernel_replicas_ready` | gauge | — | contagem | Réplicas saudáveis do serviço Kernel (via readiness probe). |
| `aios_kernel_up` | gauge (info) | — | booleano (0/1) | Liveness do processo (padrão OTel/Prometheus `up`). |

## 9. Mapeamento Métrica → Requisito Não-Funcional

| NFR | Métrica(s) usada(s) como SLI |
|-----|-------------------------------|
| NFR-001 (spawn p99 ≤ 250 ms) | `aios_kernel_spawn_latency_ms` |
| NFR-002 (controle p99 ≤ 20 ms) | `aios_kernel_syscall_latency_ms{verb=~"suspend\|resume\|kill\|get_quota"}` |
| NFR-003 (PEP p99 ≤ 8/25 ms) | `aios_kernel_pep_decision_ms{cache}` |
| NFR-004 (throughput ≥ 20k/s) | `rate(aios_kernel_syscall_total[1m])` |
| NFR-005 (disponibilidade ≥ 99,95%) | `aios_kernel_up`, `aios_kernel_replicas_ready` |
| NFR-006 (≥10⁶ ACBs) | `aios_kernel_acb_state_count{state="Hibernated"}`, `aios_kernel_shard_acb_count` |
| NFR-007 (perda de evento = 0) | `aios_kernel_outbox_lost_total` |
| NFR-009 (overshoot ≤ 1%) | `aios_kernel_quota_overshoot_ratio` |
| NFR-010 (0 bypass de PEP) | `aios_kernel_pep_bypass_total` |
| NFR-011 (100% syscalls com trace) | correlação via `trace_id` em todas as métricas + `Logging.md` |
| NFR-012 (idempotência 100%) | `aios_kernel_idempotency_hits_total` vs. `aios_kernel_idempotency_conflicts_total` |

## 10. Cuidados de Cardinalidade

- `tenant` DEVE ser incluído apenas em métricas cuja cardinalidade de tenants
  ativos seja operacionalmente administrável (dezenas a poucas centenas); em
  ambientes com milhares de tenants, agregações por tenant DEVEM usar
  *recording rules* do Prometheus em vez de séries brutas de alta cardinalidade.
- `agent_urn` **NÃO DEVE** ser label — usar `trace_id` (logs/traces) para
  investigação por agente individual.
- `verb` tem cardinalidade fixa e baixa (11 valores) — seguro como label em
  todas as métricas de syscall.
- Histogramas DEVEM usar buckets explícitos (não exponenciais automáticos) para
  garantir que os limiares de SLO (ex.: 20 ms, 250 ms) caiam exatamente em uma
  fronteira de bucket, permitindo `histogram_quantile` preciso nos limiares.

## 11. Referências

- Dashboards e regras de alerta que consomem este catálogo: `./Monitoring.md`
- Metodologia de benchmark e KPIs-alvo: `./Benchmark.md`
- Correlação de logs: `./Logging.md`
- NFRs de origem: `./NonFunctionalRequirements.md`, `_DESIGN_BRIEF.md` §7.2
- Infraestrutura OTel/Prometheus: `../024-Observability/README.md`
