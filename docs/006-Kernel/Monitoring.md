---
Documento: Monitoring
Módulo: 006-Kernel
Status: Draft
Versão: 0.1
Última atualização: 2026-07-20
Responsável (RACI-A): Arquiteto do Módulo 006-Kernel
ADRs relacionados: ADR-0010 (Observabilidade e Auditoria por Construção), ADR-0062 (Token-bucket de cotas), ADR-0063 (PEP/PDP e fail-closed), ADR-0066 (Outbox + JetStream)
RFCs relacionados: RFC-0001 (baseline, §5.6 correlação), RFC-0007 (a propor — ACB & Resource Quota Model)
Depende de: 024-Observability, 025-Audit, 022-Policy, 009-Scheduler, 008-Agent-Lifecycle
---

# 006-Kernel — Monitoring

> Este documento define **o que observar, como observar e o que fazer quando algo
> sai do esperado** no Kernel (006). Ele consome a infraestrutura transversal do
> `024-Observability` (Prometheus/Grafana/OTel Collector) e do `025-Audit`, e
> **não redefine** os contratos de correlação (`traceparent`, `X-AIOS-Tenant`) já
> fixados na `../003-RFC/RFC-0001-Architecture-Baseline.md` §5.6. O catálogo formal
> de métricas usadas aqui está em `./Metrics.md`; este documento cobre dashboards,
> golden signals, SLO/SLA/SLI, regras de alerta Prometheus e runbooks.

## 1. Objetivo e Escopo

O monitoramento do Kernel DEVE responder, em tempo real, três perguntas
operacionais:

1. **O Kernel está honrando seus contratos de desempenho e disponibilidade?**
   (NFR-001..NFR-012 do `_DESIGN_BRIEF.md` §7.2).
2. **O PEP está aplicando governança corretamente** (nem permissivo demais —
   risco de segurança — nem restritivo demais — risco de disponibilidade)?
3. **O sistema de cotas, ciclo de vida e entrega de eventos está consistente**
   (sem overshoot de cota, sem ACB preso em estado transitório, sem evento
   perdido entre commit e publish)?

**Escopo.** Este documento cobre exclusivamente a observabilidade **do próprio
serviço Kernel** (os 12 componentes internos do §2 do brief: `SyscallGateway`,
`CapabilityEnforcer`, `PolicyClient`, `AcbStore`, `ResourceQuotaManager`,
`LifecycleCoordinator`, `SchedulerClient`, `LifecycleClient`,
`ResourceBrokerRouter`, `IdempotencyStore`, `EventEmitter`, `CheckpointManager`,
`KernelTelemetry`). Monitoramento interno de `009-Scheduler`, `008-Agent-Lifecycle`
e `022-Policy` é responsabilidade dos próprios módulos — o Kernel monitora apenas
sua **fronteira de integração** com eles (latência de cliente, taxa de erro,
estado do circuit breaker).

## 2. Golden Signals do Kernel

| Golden Signal | Definição no contexto do Kernel | Métrica primária (ver `Metrics.md`) | Meta associada |
|----------------|----------------------------------|--------------------------------------|-----------------|
| **Latência** | Tempo para o Kernel processar uma syscall, do recebimento no `SyscallGateway` até a resposta. Medido por verbo. | `aios_kernel_syscall_latency_ms{verb}`, `aios_kernel_spawn_latency_ms` | NFR-001 (spawn p99 ≤ 250 ms), NFR-002 (controle p99 ≤ 20 ms), NFR-003 (PEP p99 ≤ 8/25 ms) |
| **Tráfego** | Taxa de syscalls recebidas por segundo, por verbo e por tenant. | `aios_kernel_syscall_total` (rate) | NFR-004 (≥ 20.000 syscalls/s por réplica) |
| **Erros** | Fração de syscalls que terminam em `status=error` ou `decision=deny`, por código `AIOS-<DOMINIO>-<NNNN>`. | `aios_kernel_syscall_total{status="error"}`, `aios_kernel_pep_decisions_total{decision="deny"}` | NFR-010 (0 bypass de PEP), orçamento de erro derivado de NFR-005 |
| **Saturação** | Ocupação dos recursos finitos do caminho quente: contadores de cota, fila do Outbox, *circuit breakers* de dependências, conexões Redis/PostgreSQL. | `aios_kernel_quota_consumed_ratio`, `aios_kernel_outbox_pending`, `aios_kernel_circuit_breaker_state` | NFR-009 (overshoot de cota ≤ 1%), NFR-007 (perda de evento = 0) |

## 3. SLOs / SLIs e Orçamento de Erro

Os SLOs do Kernel **DEVEM** ser rastreados como frações de tempo dentro do
orçamento de erro (*error budget*), com janela móvel de 30 dias, alinhados à
`NonFunctionalRequirements.md`.

| SLO | SLI (fonte) | Meta | Janela | Orçamento de erro (30d) |
|-----|-------------|------|--------|---------------------------|
| Disponibilidade do serviço Kernel | `up{job="kernel"}` combinado com taxa de erro 5xx/gRPC-UNAVAILABLE | ≥ 99,95% (NFR-005) | 30d rolling | 21,6 min de indisponibilidade |
| Latência de `spawn` | `histogram_quantile(0.99, aios_kernel_spawn_latency_ms)` | ≤ 250 ms | 30d rolling | ≤ 1% das amostras acima do limiar |
| Latência de syscalls de controle | `histogram_quantile(0.99, aios_kernel_syscall_latency_ms{verb=~"suspend|resume|kill|get_quota"})` | ≤ 20 ms | 30d rolling | ≤ 1% das amostras acima do limiar |
| Decisão do PEP (cache hit) | `histogram_quantile(0.99, aios_kernel_pep_decision_ms{cache="hit"})` | ≤ 8 ms | 30d rolling | ≤ 1% das amostras acima do limiar |
| Decisão do PEP (cache miss) | `histogram_quantile(0.99, aios_kernel_pep_decision_ms{cache="miss"})` | ≤ 25 ms | 30d rolling | ≤ 1% das amostras acima do limiar |
| Consistência de cota | `aios_kernel_quota_overshoot_ratio` | ≤ 1% (NFR-009) | 7d rolling | conforme burn-rate abaixo |
| Durabilidade de eventos | `aios_kernel_outbox_lost_total` | 0 eventos perdidos (NFR-007) | contínuo | qualquer valor > 0 é incidente SEV-1 |
| Cobertura de auditoria/PEP | `aios_kernel_pep_bypass_total` | 0 (NFR-010) | contínuo | qualquer valor > 0 é incidente SEV-1 |

**Multi-window multi-burn-rate.** Alertas de SLO de disponibilidade e latência
DEVEM usar o padrão de duas janelas (curta e longa) para equilibrar sensibilidade
e ruído, conforme detalhado na §5.

## 4. Dashboards (Grafana)

Os dashboards do Kernel são provisionados como código (JSON versionado em
`029-Operations`) e publicados no Grafana provido por `024-Observability`. Todo
painel DEVE permitir filtro por `tenant` e por `verb` (template variables).

| Dashboard | Público-alvo | Painéis principais |
|-----------|--------------|---------------------|
| **Kernel Overview** | On-call, SRE | RPS por verbo; latência p50/p95/p99 por verbo; taxa de erro; taxa de `deny`; réplicas saudáveis; burn rate de SLO. |
| **ACB Lifecycle** | Time de plataforma | Distribuição de ACBs por estado (`Pending/Admitted/Running/Suspended/Hibernated/Terminating/Terminated/Failed`); taxa de transições por par `(from,to)`; conflitos de OCC/s; tempo médio em cada estado. |
| **Quota & Cost** | FinOps, SRE | Consumo de cota por tenant (tokens/cpu-ms/custo/tools/syscalls); nº de `agent.quota.exceeded`/min; top-N tenants por consumo; razão de overshoot. |
| **PEP / Policy** | Segurança, Compliance | Decisões allow/deny por segundo; latência de decisão (cache hit/miss); taxa de *cache hit* do `CapabilityEnforcer`; estado do circuit breaker do `PolicyClient`; eventos `agent.syscall.denied`. |
| **Outbox & Events** | Plataforma, SRE | `aios_kernel_outbox_pending`; lag do relay (`aios_kernel_outbox_lag_ms`); taxa de publicação por subject; erros de publicação no JetStream. |
| **Broker Health** | Plataforma, SRE | Latência/erro por dependência (`010/011/012/015/017`); estado dos circuit breakers (`ResourceBrokerRouter`); taxa de *bulkhead rejection*. |

### 4.1 Mockup ASCII — Dashboard "Kernel Overview"

```
┌─────────────────────────────── Kernel Overview  (tenant=*, verb=*) ───────────────────────────────┐
│  RPS por verbo                         │  Latência p99 por verbo (ms)                             │
│  ▇▇▇▇▇▇▇▇▇▇ spawn        1.2k/s        │  spawn        ██████████████████░░  212 ms  (SLO 250)     │
│  ▇▇▇▇▇▇▇▇▇▇▇▇▇▇ get_quota 4.8k/s       │  get_quota    ███░░░░░░░░░░░░░░░░░   6 ms   (SLO 20)      │
│  ▇▇▇▇▇▇ suspend           0.6k/s        │  suspend      █████░░░░░░░░░░░░░░░  11 ms   (SLO 20)      │
├─────────────────────────────────────────┼───────────────────────────────────────────────────────────┤
│  Taxa de erro / deny (janela 5m)        │  Burn rate do SLO de disponibilidade                     │
│  status=error   0.02%                   │  janela 1h : 0.8x  (ok)                                  │
│  decision=deny  1.4%   (esperado > 0)   │  janela 6h : 1.1x  (ok)                                  │
│                                          │  janela 3d : 0.9x  (ok)                                  │
├─────────────────────────────────────────┴───────────────────────────────────────────────────────────┤
│  Réplicas saudáveis: 6/6      Outbox pending: 42      CB(PDP)=closed    CB(Scheduler)=closed        │
└───────────────────────────────────────────────────────────────────────────────────────────────────┘
```

## 5. Regras de Alerta (Prometheus)

> `for` é o tempo mínimo em que a condição DEVE persistir antes de disparar,
> evitando *flapping*. Severidade `page` aciona on-call; `ticket` gera item de
> backlog; `info` é apenas painel/registro.

| Alerta | Expressão PromQL (resumida) | `for` | Severidade | Runbook |
|--------|------------------------------|-------|------------|---------|
| `KernelSpawnLatencyHigh` | `histogram_quantile(0.99, sum(rate(aios_kernel_spawn_latency_ms_bucket[5m])) by (le)) > 250` | 10m | page | RB-K-01 |
| `KernelControlLatencyHigh` | `histogram_quantile(0.99, sum(rate(aios_kernel_syscall_latency_ms_bucket{verb=~"suspend|resume|kill|get_quota"}[5m])) by (le)) > 20` | 10m | page | RB-K-01 |
| `KernelErrorRateHigh` | `sum(rate(aios_kernel_syscall_total{status="error"}[5m])) / sum(rate(aios_kernel_syscall_total[5m])) > 0.01` | 5m | page | RB-K-02 |
| `KernelPepFailClosedSpike` | `sum(rate(aios_kernel_pep_decisions_total{decision="deny",reason="pdp_unavailable"}[5m])) > 5` | 5m | page | RB-K-03 |
| `KernelPepLatencyHigh` | `histogram_quantile(0.99, sum(rate(aios_kernel_pep_decision_ms_bucket{cache="hit"}[5m])) by (le)) > 8` | 10m | ticket | RB-K-04 |
| `KernelQuotaOvershoot` | `aios_kernel_quota_overshoot_ratio > 0.01` | 15m | ticket | RB-K-05 |
| `KernelOutboxBacklogGrowing` | `aios_kernel_outbox_pending > 1000` | 5m | page | RB-K-06 |
| `KernelOutboxLagHigh` | `aios_kernel_outbox_lag_ms > 5000` | 5m | page | RB-K-06 |
| `KernelOccConflictRateHigh` | `sum(rate(aios_kernel_occ_conflicts_total[5m])) > 50` | 10m | ticket | RB-K-07 |
| `KernelBrokerCircuitOpen` | `aios_kernel_circuit_breaker_state{dependency=~"010|011|012|015|017"} == 2` | 1m | page | RB-K-08 |
| `KernelSchedulerCircuitOpen` | `aios_kernel_circuit_breaker_state{dependency="009-scheduler"} == 2` | 1m | page | RB-K-08 |
| `KernelAcbStuckInPending` | `max(time() - aios_kernel_acb_state_entered_timestamp{state="Pending"}) > 5` | 1m | page | RB-K-09 |
| `KernelAcbStuckInTerminating` | `max(time() - aios_kernel_acb_state_entered_timestamp{state="Terminating"}) > 60` | 5m | ticket | RB-K-09 |
| `KernelReplicaDown` | `up{job="kernel"} == 0` | 2m | page | RB-K-10 |
| `KernelPepBypassDetected` | `aios_kernel_pep_bypass_total > 0` | 0s | page (SEV-1) | RB-K-11 |
| `KernelEventLossDetected` | `increase(aios_kernel_outbox_lost_total[10m]) > 0` | 0s | page (SEV-1) | RB-K-06 |
| **Burn-rate SLO (multi-janela)** | curta: `error_rate_5m > 14.4 * (1 - 0.9995)`; longa: `error_rate_1h > 14.4 * (1 - 0.9995)` (ambas verdadeiras) | 5m | page | RB-K-02 |
| **Burn-rate SLO (lento)** | curta: `error_rate_1h > 6 * (1 - 0.9995)`; longa: `error_rate_6h > 6 * (1 - 0.9995)` | 30m | ticket | RB-K-02 |

## 6. Runbooks Associados

> Os runbooks completos (passo-a-passo executável) residem em
> `../029-Operations/Runbooks/` (fonte operacional); este documento mantém apenas
> o resumo e o vínculo alerta → runbook.

| ID | Runbook | Gatilho | Resumo dos passos |
|----|---------|---------|---------------------|
| RB-K-01 | Latência elevada de syscall | `KernelSpawnLatencyHigh`, `KernelControlLatencyHigh` | 1) Verificar saturação de CPU/GC do pod. 2) Checar latência do `AcbStore` (Redis/PostgreSQL). 3) Checar CB do Scheduler/Lifecycle. 4) Escalar réplicas se saturação confirmada. |
| RB-K-02 | Taxa de erro elevada | `KernelErrorRateHigh`, burn-rate | 1) Segmentar erros por código `AIOS-*`. 2) Se concentrado em `AIOS-KERNEL-0005`, investigar broker de recurso downstream. 3) Se `AIOS-KERNEL-0009`, ver RB-K-07. |
| RB-K-03 | Fail-closed do PEP em massa | `KernelPepFailClosedSpike` | 1) Confirmar indisponibilidade do PDP (`022-Policy`). 2) Verificar `kernel.pep.fail_mode`. 3) Escalar para dono do `022-Policy`. 4) Comunicar impacto (todas as syscalls privilegiadas negadas). |
| RB-K-04 | Degradação do cache de decisão | `KernelPepLatencyHigh` | 1) Checar taxa de *cache hit*. 2) Checar TTL (`kernel.pep.decision_cache_ttl_ms`). 3) Checar latência do PDP em cache miss. |
| RB-K-05 | Overshoot de cota | `KernelQuotaOvershoot` | 1) Identificar `subject_urn` com overshoot. 2) Verificar concorrência no token-bucket Redis. 3) Confirmar reconciliação com PostgreSQL. Ver ADR-0062. |
| RB-K-06 | Backlog/perda do Outbox | `KernelOutboxBacklogGrowing`, `KernelOutboxLagHigh`, `KernelEventLossDetected` | 1) Checar saúde do NATS/JetStream. 2) Checar o *relay* do Outbox (processo/thread vivo?). 3) Se `lost_total > 0`, abrir incidente SEV-1 imediatamente (viola NFR-007). |
| RB-K-07 | Conflitos de OCC elevados | `KernelOccConflictRateHigh` | 1) Identificar ACBs quentes (mesmo `urn` sob concorrência). 2) Avaliar necessidade de lock distribuído pontual (ver `_DESIGN_BRIEF.md` §10). |
| RB-K-08 | Circuit breaker aberto (dependência) | `KernelBrokerCircuitOpen`, `KernelSchedulerCircuitOpen` | 1) Confirmar saúde do dependente. 2) Verificar bulkhead isolando o impacto. 3) Aguardar *half-open*; escalar ao dono do módulo dependente. |
| RB-K-09 | ACB preso em estado transitório | `KernelAcbStuckInPending`, `KernelAcbStuckInTerminating` | 1) Consultar `syscall_log` do ACB. 2) Verificar timeout de admissão/drenagem. 3) Forçar transição administrativa (com auditoria) se saga travada. |
| RB-K-10 | Réplica fora do ar | `KernelReplicaDown` | 1) Checar readiness/liveness. 2) Checar dependências críticas (PostgreSQL/Redis). 3) Rolling restart se aplicável. |
| RB-K-11 | Bypass de PEP detectado | `KernelPepBypassDetected` | 1) Congelar deploys. 2) Isolar a rota que causou o bypass. 3) Auditoria completa via `025-Audit`. Incidente de segurança SEV-1. |

## 7. Referências

- Catálogo de métricas: `./Metrics.md`
- Logging estruturado e correlação: `./Logging.md`
- Estratégia de testes de observabilidade: `./Testing.md`
- Metodologia de carga/benchmark: `./Benchmark.md`
- Modos de falha e recuperação: `./FailureRecovery.md`
- Infraestrutura transversal: `../024-Observability/README.md`
- Auditoria imutável: `../025-Audit/README.md`
- Contratos de correlação: `../003-RFC/RFC-0001-Architecture-Baseline.md` §5.6
- Design Brief (fonte de verdade): `./_DESIGN_BRIEF.md` §7.2, §9
