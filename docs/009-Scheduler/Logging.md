---
Documento: Logging
Módulo: 009-Scheduler
Status: Draft
Versão: 0.1
Última atualização: 2026-07-20
Responsável (RACI-A): Arquiteto do Módulo 009-Scheduler
ADRs relacionados: ADR-0090 (admission control), ADR-0094 (preempção compensável), ADR-0095 (backpressure em três níveis), ADR-0097 (outbox transacional)
RFCs relacionados: RFC-0001 (baseline, §5.4 erro, §5.6 correlação)
Depende de: 024-Observability, 025-Audit, 021-Security
---

# 009-Scheduler — Logging

> Este documento define o **contrato de logging estruturado** do Scheduler:
> níveis, campos obrigatórios, catálogo de eventos de log por componente,
> correlação com traces OTel e política de retenção. A pilha usada é
> **Serilog** (produção do log estruturado em .NET 10) → **Seq**
> (armazenamento/consulta), conforme `_DESIGN_BRIEF.md` e stack global.
> Logging **NÃO substitui** a auditoria imutável de `025-Audit` (que persiste a
> trilha regulatória de decisões de admissão/preempção); logging é para
> diagnóstico operacional, com retenção mais curta.

## 1. Objetivo e Princípios

O logging do Scheduler DEVE:

1. Ser **estruturado** (JSON), nunca texto livre sem campos — todo log é uma
   mensagem-modelo (`MessageTemplate` do Serilog) com propriedades tipadas.
2. Ser **correlacionável**: todo log emitido durante o processamento de uma
   `SchedulingRequest` DEVE carregar `trace_id`, `span_id`, `tenant_id`,
   `request_id` e, quando aplicável, `decision_id` (RFC-0001 §5.6).
3. **Nunca conter segredos** nem PII — `payload_ref` (URN/URI) é logado, nunca
   o conteúdo do payload de tarefa (§12.3 do `_DESIGN_BRIEF.md`, minimização de
   dados). `feature_vector` logado em nível `Debug` DEVE ser o vetor
   numérico/categorial de decisão, nunca dado bruto de negócio.
4. Ser **acionável**: cada linha de log em nível `Warning`/`Error` DEVE
   permitir, sozinha, identificar o componente, a fase do pipeline
   (admissão/prioridade/placement/preempção/dispatch), o tenant/tarefa e a
   causa (`reason_code`/`AIOS-SCHED-<NNNN>`).
5. Não duplicar a função da auditoria: decisões de admissão/preempção/mutação
   de política são logadas para diagnóstico **e** persistidas via
   `DecisionJournal`/auditadas via `025-Audit` separadamente (double-write
   intencional, propósitos distintos — diagnóstico vs. proveniência
   regulatória).

## 2. Níveis de Log e Política de Uso

| Nível | Uso no Scheduler | Exemplos |
|-------|----------------|----------|
| `Verbose` | Detalhe de depuração interna, desabilitado em produção por padrão. | Conteúdo bruto do `feature_vector` calculado pelo `FeatureExtractor`. |
| `Debug` | Fluxo interno útil em troubleshooting, habilitável por tenant/shard. | Entrada/saída de `PriorityQueueStore.Enqueue` com score calculado; consulta de cache de orçamento (`CostEstimator`). |
| `Information` | Eventos de negócio normais (uma linha por transição de estado da §4 do `StateMachine.md`, uma linha por decisão). | `Admissão concluída` com `outcome`, `effective_priority`, `latency_ms`. |
| `Warning` | Degradação tolerada ou comportamento anômalo não fatal. | Backpressure em nível `defer`; cache miss de PDP/orçamento acima do limiar; lease de dispatch expirado (reconciliado). |
| `Error` | Falha que impede o processamento da requisição corrente. | `AIOS-SCHED-0012` (falha interna de decisão); falha de publicação de evento. |
| `Fatal` | Falha que compromete o processo/serviço (deve gerar crash controlado + alerta). | Falha irrecuperável de conexão com PostgreSQL/Redis na inicialização; perda de ownership de todos os shards simultaneamente. |

**Regra de amostragem.** Logs `Information` de operações de alta frequência
(`enqueue`, leituras de `StreamQueueState`) PODEM ser amostrados (ex.: 1:10)
sob carga elevada, mas `Warning`/`Error`/`Fatal` e todo log de **decisão de
admissão, preempção e mutação de política** NÃO DEVEM ser amostrados (100% de
captura — requisito de auditabilidade, NFR-010/NFR-012).

## 3. Campos Obrigatórios (todo evento de log)

| Campo | Tipo | Obrigatório | Descrição |
|-------|------|-------------|-----------|
| `timestamp` | `datetime` (RFC 3339, UTC) | sim | Momento de emissão. |
| `level` | `string` | sim | Um dos níveis da §2. |
| `message_template` | `string` | sim | Template Serilog (ex.: `"Admissão {Outcome} para task {TaskUrn} em {LatencyMs} ms"`). |
| `component` | `string` | sim | Um dos 21 componentes internos (`_DESIGN_BRIEF.md` §2.1), ex.: `AdmissionController`, `PreemptionManager`. |
| `trace_id` | `string` | sim | Extraído de `traceparent` (RFC-0001 §5.6). |
| `span_id` | `string` | sim | Span corrente do OTel. |
| `tenant_id` | `string` | sim | Contexto de isolamento multi-tenant. |
| `request_id` | `string` | sim (exceto logs de bootstrap/reconciliação) | ULID da `SchedulingRequest`. |
| `decision_id` | `string` | quando uma `SchedulingDecision` já foi produzida | ULID da decisão. |
| `task_urn` / `agent_urn` | `string` | quando aplicável | URNs da tarefa/agente-alvo. |
| `service_class` | `string` | quando aplicável | `INTERACTIVE`\|`BATCH`\|`BACKGROUND`\|`SYSTEM`. |
| `outcome` | `string` | em logs de decisão | `ADMITTED`\|`SCHEDULED`\|`REJECTED`\|`PREEMPTED`\|`RESUMED`\|`DEFERRED`. |
| `idempotency_key` | `string` | quando presente na requisição | Correlaciona repetições. |
| `reason_code` | `string` | quando `outcome ∈ {REJECTED}` ou `level ∈ {Error,Fatal}` | `AIOS-SCHED-<NNNN>`. |
| `latency_ms` | `number` | quando aplicável | Duração da operação logada. |
| `shard` | `integer` | quando aplicável | Shard alvo (`ShardRouter`/`PlacementEngine`). |
| `placement_target` | `object` | em logs de placement/dispatch | `{shard, node_id, runtime_pool}`. |
| `effective_priority` | `number` | em logs de `PriorityCostEvaluator` | Prioridade efetiva calculada. |
| `policy_id` / `policy_version` | `string` | em logs de decisão | Identidade da política aplicada (`ISchedulingPolicy`). |
| `exception` | `object` | quando `level ∈ {Error, Fatal}` | Tipo, mensagem, stack trace (sem PII). |
| `service_version` | `string` | sim | Versão do build do Scheduler (SemVer + hash de commit). |
| `environment` | `string` | sim | `dev` \| `staging` \| `prod`. |

> **Redação.** `SchedulingRequest.payload_ref` é logado como referência
> (URN/URI); o conteúdo do blob apontado NÃO DEVE ser lido/logado pelo
> Scheduler. `feature_vector` logado em `Debug` DEVE conter apenas
> features numéricas/categoriais de decisão (carga, fila, custo estimado,
> folga de SLA, histórico) — nunca texto livre de negócio.

## 4. Catálogo de Eventos de Log por Componente

> Convenção de nomeação do evento de log (campo lógico `event_name`, usado
> como tag de busca no Seq): `scheduler.<component_snake>.<ação>`.

| Componente | `event_name` | Nível | Quando | Campos-chave adicionais |
|------------|--------------|-------|--------|---------------------------|
| `SchedulingGateway` | `scheduler.gateway.received` | Information | `SchedulingRequest` recebida, antes de qualquer processamento. | `idempotency_key`, `service_class` |
| `SchedulingGateway` | `scheduler.gateway.tenant_mismatch` | Error | `X-AIOS-Tenant` diverge do claim autenticado. | `reason_code=AIOS-SCHED-0004` |
| `SchedulingGateway` | `scheduler.gateway.invalid_request` | Warning | Payload/URN inválido antes do pipeline. | `reason_code=AIOS-SCHED-0005` |
| `AdmissionController` | `scheduler.admission.decided` | Information | Toda decisão de admissão (allow/deny), 100% capturado. | `outcome`, `reason_code` (quando `REJECTED`) |
| `AdmissionController` | `scheduler.admission.pdp_denied` | Warning | PDP negou admissão. | `reason_code=AIOS-SCHED-0004` |
| `PolicyClient` | `scheduler.policy_client.circuit_open` | Error | Circuit breaker do PDP (022) abriu. | `dependency=022-policy`, `failure_rate` |
| `QuotaLedger` | `scheduler.quota.reserved` \| `.consumed` \| `.released` | Debug/Information | Ciclo de vida de reserva de slot. | `resource=concurrency`, `tenant_id`, `service_class` |
| `QuotaLedger` | `scheduler.quota.exceeded` | Warning | Cota de concorrência excedida. | `reason_code=AIOS-SCHED-0001` |
| `QuotaLedger` | `scheduler.quota.overcommit_detected` | Fatal | Reserva atômica falhou e overcommit foi detectado (nunca deveria ocorrer). | incidente SEV-1 |
| `BudgetClient` | `scheduler.budget.insufficient` | Warning | Orçamento insuficiente (026). | `reason_code=AIOS-SCHED-0002` |
| `PriorityCostEvaluator` | `scheduler.priority.evaluated` | Debug | Score `C=α·L+β·$+γ·E` e prioridade efetiva calculados. | `cost_score`, `effective_priority`, `policy_version` |
| `PriorityCostEvaluator` | `scheduler.priority.quality_floor_violation` | Warning | `quality_floor` inatingível. | `reason_code=AIOS-SCHED-0007` |
| `PriorityCostEvaluator` | `scheduler.priority.edf_expired` | Information | Deadline vencido antes do dispatch (`QUEUED→EXPIRED`). | `reason_code=AIOS-SCHED-0008`, `deadline_at` |
| `PlacementEngine` | `scheduler.placement.resolved` | Debug | Shard/nó/pool escolhido. | `placement_target` |
| `PlacementEngine` | `scheduler.placement.no_capacity` | Warning | Nenhum shard/nó elegível. | `reason_code=AIOS-SCHED-0010` |
| `PlacementEngine` | `scheduler.placement.affinity_fallback` | Warning | Anti-afinidade ignorada por ausência de alternativa. | `affinity` |
| `ShardRouter` | `scheduler.shard_router.rebalanced` | Information | Rebalanceamento de shards após mudança de `N`. | `keys_moved_ratio` |
| `PriorityQueueStore` | `scheduler.queue.enqueued` \| `.dequeued` | Debug | Ciclo de vida da tarefa na ZSET. | `shard`, `effective_priority` |
| `DispatchLoop` | `scheduler.dispatch.scheduled` | Information | `SchedulingDecision` enviada ao Runtime Supervisor. | `decision_id`, `placement_target`, `latency_ms` |
| `DispatchLoop` | `scheduler.dispatch.lease_conflict` | Warning | Conflito de lease de dispatch entre réplicas. | `shard` |
| `PreemptionManager` | `scheduler.preemption.executed` | Information | Vítima preemptada, aviso enviado ao Runtime. | `victim_task_urn`, `initiator_task_urn`, `grace_period_ms` |
| `PreemptionManager` | `scheduler.preemption.denied` | Warning | PDP negou preempção ou ausência de vítima elegível. | `reason` |
| `PreemptionManager` | `scheduler.preemption.cooldown_blocked` | Debug | Vítima candidata bloqueada por cooldown anti-*thrashing*. | `victim_task_urn` |
| `BackpressureController` | `scheduler.backpressure.level_changed` | Warning | Mudança de nível `accept`/`defer`/`reject`. | `shard`, `from_level`, `to_level`, `pressure` |
| `BackpressureController` | `scheduler.backpressure.rejected` | Warning | Admissão negada por saturação. | `reason_code=AIOS-SCHED-0003`, `retry_after_ms` |
| `IdempotencyStore` | `scheduler.idempotency.replay` | Information | Requisição duplicada detectada; resultado reaproveitado. | `idempotency_key` |
| `IdempotencyStore` | `scheduler.idempotency.conflict` | Warning | Mesma chave, payload divergente. | `reason_code=AIOS-SCHED-0006` |
| `DecisionJournal` | `scheduler.decision_journal.recorded` | Information | `SchedulingDecision` persistida (outbox) com sucesso. | `decision_id`, `outcome`, `write_ms` |
| `EventPublisher` | `scheduler.event_publisher.published` | Information | Evento publicado no JetStream com sucesso. | `event_id`, `subject`, `relay_latency_ms` |
| `EventPublisher` | `scheduler.event_publisher.publish_failed` | Error | Falha ao publicar; permanece pendente para nova tentativa. | `event_id`, `retry_count` |
| `ReconciliationWorker` | `scheduler.reconciliation.orphan_requeued` | Warning | Lease expirado sem confirmação; tarefa re-enfileirada. | `request_id`, `shard` |
| `ReconciliationWorker` | `scheduler.reconciliation.drift_corrected` | Warning | Divergência entre estado quente e realidade observada, corrigida. | `dimension` |
| `RuntimeSupervisorClient` | `scheduler.runtime_client.signal_received` | Debug | Sinal de runtime (`started`/`completed`/`failed`/`heartbeat`) recebido. | `signal_id`, `type` |
| `RuntimeSupervisorClient` | `scheduler.runtime_client.timeout` | Error | Timeout aguardando confirmação de materialização. | `dependency=007-runtime-supervisor` |
| `CapacityProvider` | `scheduler.capacity.stale` | Warning | Heartbeat de capacidade além do TTL esperado. | `node_id`, `heartbeat_age_ms` |
| `SchedulerTelemetry` | `scheduler.telemetry.audit_forward_failed` | Error | Falha ao encaminhar evento de proveniência a `025-Audit`. | `error_code` |

## 5. Correlação com Traces (OpenTelemetry)

- Todo log emitido dentro do processamento de uma `SchedulingRequest` DEVE ser
  emitido **dentro do span ativo** correspondente, permitindo ao Serilog
  enriquecer automaticamente `trace_id`/`span_id` via o *Activity* do .NET
  (integração OTel↔Serilog padrão da stack, ver `../024-Observability/`).
- O par (`trace_id`, `span_id`) DEVE permitir pivotar do Seq (log) para o
  backend de tracing (Grafana Tempo/Jaeger) em um clique — os dashboards do
  Scheduler DEVEM incluir *data link* configurado.
- Logs de eventos assíncronos (relay do outbox, `ReconciliationWorker`,
  reavaliação de backpressure) que não têm uma requisição `Submit` associada
  DEVEM iniciar um novo trace raiz com `source=urn:aios:<tenant>:service:scheduler`
  como atributo de span.
- Todo log de decisão (`scheduler.admission.decided`, `scheduler.dispatch.scheduled`,
  `scheduler.preemption.executed`) DEVE incluir `decision_id`, permitindo
  correlacionar log ↔ `SchedulingDecision` persistida ↔ evento publicado
  (tríade de rastreabilidade exigida por NFR-010).

## 6. Exemplos de Log Estruturado (JSON, formato de saída do sink Seq)

**Admissão concluída (allow, dispatch imediato):**
```json
{
  "timestamp": "2026-07-20T12:00:00.009Z",
  "level": "Information",
  "message_template": "Decisão {Outcome} para task {TaskUrn} em {LatencyMs} ms",
  "event_name": "scheduler.admission.decided",
  "component": "AdmissionController",
  "trace_id": "4bf92f3577b34da6a3ce929d0e0e4736",
  "span_id": "00f067aa0ba902b7",
  "tenant_id": "acme",
  "request_id": "01J9ZA000000000000000010",
  "decision_id": "01J9ZA300000000000000012",
  "task_urn": "urn:aios:acme:task:01J9ZA100000000000000011",
  "agent_urn": "urn:aios:acme:agent:01J9Z8Q6H7K2M4N6P8R0S2T4V6",
  "service_class": "INTERACTIVE",
  "outcome": "SCHEDULED",
  "effective_priority": 68,
  "policy_id": "heuristic-default",
  "policy_version": "heuristic@1",
  "latency_ms": 9.1,
  "service_version": "0.1.0+a1b2c3d",
  "environment": "prod"
}
```

**Backpressure em nível reject (rejeição por saturação):**
```json
{
  "timestamp": "2026-07-20T12:01:00.000Z",
  "level": "Warning",
  "message_template": "Admissão rejeitada por backpressure: shard {Shard} pressão={Pressure}",
  "event_name": "scheduler.backpressure.rejected",
  "component": "BackpressureController",
  "trace_id": "4bf92f3577b34da6a3ce929d0e0e4736",
  "span_id": "1a2b3c4d5e6f7081",
  "tenant_id": "acme",
  "request_id": "01J9ZA000000000000000020",
  "task_urn": "urn:aios:acme:task:01J9ZA100000000000000021",
  "service_class": "BATCH",
  "outcome": "REJECTED",
  "shard": 17,
  "reason_code": "AIOS-SCHED-0003",
  "retry_after_ms": 1500,
  "service_version": "0.1.0+a1b2c3d",
  "environment": "prod"
}
```

**Preempção executada (vítima escolhida, grace period concedido):**
```json
{
  "timestamp": "2026-07-20T12:03:00.000Z",
  "level": "Information",
  "message_template": "Preempção executada: vítima {VictimTaskUrn} por {InitiatorTaskUrn}, grace={GracePeriodMs}ms",
  "event_name": "scheduler.preemption.executed",
  "component": "PreemptionManager",
  "trace_id": "9e8d7c6b5a4938271605f4e3d2c1b0a9",
  "span_id": "2b3c4d5e6f708192",
  "tenant_id": "acme",
  "decision_id": "01J9ZA300000000000000032",
  "task_urn": "urn:aios:acme:task:01J9ZA100000000000000030",
  "service_class": "BATCH",
  "outcome": "PREEMPTED",
  "grace_period_ms": 2000,
  "service_version": "0.1.0+a1b2c3d",
  "environment": "prod"
}
```

## 7. Retenção e Ciclo de Vida do Log

| Ambiente | Retenção no Seq | Retenção arquivada (frio) | Observação |
|----------|-------------------|------------------------------|------------|
| `prod` | 30 dias (consulta interativa) | 180 dias em armazenamento frio (MinIO, comprimido) | Retenção de proveniência regulatória (`SchedulingDecision`) é superior e regida por `Database.md`/`025-Audit`/LGPD. |
| `staging` | 14 dias | não aplicável | — |
| `dev` | 3 dias | não aplicável | — |

- Logs **NÃO DEVEM** ser a fonte de verdade regulatória — essa função é do
  `DecisionJournal` (PostgreSQL, append-only) e de `025-Audit`.
- A rotação/expurgo de logs no Seq é automatizada; nenhuma operação manual de
  exclusão seletiva é permitida (preservação de integridade forense mínima
  dentro da janela de retenção).
- Logs contendo `tenant_id` DEVEM respeitar a mesma fronteira de isolamento
  multi-tenant no controle de acesso ao Seq (RBAC por tenant, ver
  `021-Security`).
- Direito ao esquecimento: expurgo por `tenant_id`/`agent_urn` propagado e
  auditado (coordenado com `025-Audit`) DEVE também remover/anonimizar
  entradas de log associadas dentro da janela de retenção corrente
  (`_DESIGN_BRIEF.md` §12.3).

## 8. Referências

- Catálogo de métricas (correlacionadas por `trace_id`): `./Metrics.md`
- Dashboards e alertas: `./Monitoring.md`
- Auditoria imutável (fonte regulatória): `../025-Audit/README.md`
- Correlação e cabeçalhos: `../003-RFC/RFC-0001-Architecture-Baseline.md` §5.6
- Catálogo de erros do Scheduler: `./API.md` §5
- Máquina de estados (base das transições logadas): `./StateMachine.md`
- Infraestrutura de observabilidade: `../024-Observability/README.md`
