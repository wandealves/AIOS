---
Documento: Logging
Módulo: 008-Agent-Lifecycle
Status: Draft
Versão: 0.1
Última atualização: 2026-07-20
Responsável (RACI-A): Arquiteto do Módulo 008 — Agent Lifecycle
ADRs relacionados: ADR-0080 (FSM canônica), ADR-0084 (Lease/fencing), ADR-0085 (Migração como saga), ADR-0086 (Event sourcing + Outbox), ADR-0089 (Retenção/tombstone)
RFCs relacionados: RFC-0001 (baseline, §5.4 erro, §5.6 correlação)
Depende de: 024-Observability, 025-Audit, 021-Security
---

# 008-Agent-Lifecycle — Logging

> Este documento define o **contrato de logging estruturado** do módulo 008:
> níveis, campos obrigatórios, catálogo de eventos de log por componente,
> correlação com traces OTel e política de retenção. A pilha usada é
> **Serilog** (produção do log estruturado em .NET 10) → **Seq**
> (armazenamento/consulta), conforme stack global. Logging **NÃO substitui** a
> auditoria imutável de `025-Audit` (que persiste a trilha regulatória de
> transições de ciclo de vida); logging é para diagnóstico operacional, com
> retenção mais curta.

## 1. Objetivo e Princípios

O logging do módulo 008 DEVE:

1. Ser **estruturado** (JSON), nunca texto livre sem campos — todo log é uma
   mensagem-modelo (`MessageTemplate` do Serilog) com propriedades tipadas.
2. Ser **correlacionável**: todo log emitido durante o processamento de uma
   transição de ciclo de vida DEVE carregar `trace_id`, `span_id` e
   `tenant_id` (RFC-0001 §5.6).
3. **Nunca conter segredos** (chaves de cifragem, tokens) nem conteúdo de
   working memory — o módulo 008 loga apenas **metadados de estado**
   (`agent_id`, `generation`, `state`, tamanhos, hashes), nunca o payload
   serializado do agente (isso é responsabilidade do 010-Memory quanto a
   conteúdo; o 008 trata a working memory como blob opaco).
4. Ser **acionável**: cada linha de log em nível `Warning`/`Error` DEVE
   permitir, sozinha, identificar o componente, o agente/tenant, a transição
   tentada e a causa.
5. Não duplicar a função da auditoria: toda transição de FSM é logada para
   diagnóstico **e** auditada via `025-Audit` separadamente (double-write
   intencional, propósitos distintos — log é operacional/efêmero, auditoria é
   regulatória/imutável).

## 2. Níveis de Log e Política de Uso

| Nível | Uso no módulo 008 | Exemplos |
|-------|----------------|----------|
| `Verbose` | Detalhe de depuração interna, desabilitado em produção por padrão. | Payload bruto (redigido) enviado ao `SnapshotStore`. |
| `Debug` | Fluxo interno útil em troubleshooting, habilitável por tenant/agente. | Tentativa de aquisição de lease com `fencing_token` esperado. |
| `Information` | Eventos de negócio normais (uma linha por transição de FSM processada). | `Transição concluída` com `from_state`, `to_state`, `trigger`, `latency_ms`. |
| `Warning` | Degradação tolerada ou comportamento anômalo não fatal. | Lease não adquirido (outro coordenador ativo); retry de checkpoint. |
| `Error` | Falha que impede a transição corrente. | `AIOS-LIFECYCLE-0007` (checkpoint corrompido). |
| `Fatal` | Falha que compromete o processo/serviço (deve gerar crash controlado + alerta). | Falha irrecuperável de conexão com PostgreSQL na inicialização do `AcbStore`. |

**Regra de amostragem.** Logs `Information` de transições de alta frequência
(`ready`, `running` em cargas de `spawn` em massa) PODEM ser amostrados (ex.:
1:10) sob carga elevada, mas `Warning`/`Error`/`Fatal` e todo log de **decisão
de hibernação/migração** NÃO DEVEM ser amostrados (100% de captura — requisito
de auditabilidade, NFR-012).

## 3. Campos Obrigatórios (todo evento de log)

| Campo | Tipo | Obrigatório | Descrição |
|-------|------|-------------|-----------|
| `timestamp` | `datetime` (RFC 3339, UTC) | sim | Momento de emissão. |
| `level` | `string` | sim | Um dos níveis da §2. |
| `message_template` | `string` | sim | Template Serilog (ex.: `"Transição {FromState}->{ToState} concluída em {LatencyMs} ms"`). |
| `component` | `string` | sim | Um dos 17 componentes internos (§2 do brief), ex.: `LifecycleCoordinator`, `HibernationController`. |
| `trace_id` | `string` | sim | Extraído de `traceparent` (RFC-0001 §5.6). |
| `span_id` | `string` | sim | Span corrente do OTel. |
| `tenant_id` | `string` | sim | Contexto de isolamento multi-tenant. |
| `agent_id` | `string` (ULID/URN `agent`) | quando aplicável | Identidade do agente-alvo. |
| `generation` | `int64` | quando aplicável | Encarnação corrente do ACB. |
| `from_state` / `to_state` | `string` | em logs de transição FSM | Estados de `LifecycleState` (§4 do brief). |
| `trigger` | `string` | em logs de transição FSM | `spawn`,`ready`,`start`,`suspend`,`resume`,`hibernate`,`wake`,`migrate`,`terminate`,`fail`,`reconcile`. |
| `actor` | `string` | sim em mutações | `api:<sub>`, `scheduler`, `reconciler`, `runtime`. |
| `idempotency_key` | `string` | quando presente na requisição | Correlaciona repetições. |
| `fencing_token` | `int64` | em operações de lease | Token de lease vigente. |
| `checkpoint_id` | `string` | quando aplicável | Identidade do checkpoint envolvido. |
| `migration_id` | `string` | em logs de migração | Identidade da saga. |
| `error_code` | `string` | quando `level ∈ {Error, Fatal}` | `AIOS-LIFECYCLE-<NNNN>`. |
| `latency_ms` | `number` | quando aplicável | Duração da operação logada. |
| `exception` | `object` | quando `level ∈ {Error, Fatal}` | Tipo, mensagem, stack trace (sem PII). |
| `service_version` | `string` | sim | Versão do build do módulo 008 (SemVer + hash de commit). |
| `environment` | `string` | sim | `dev` \| `staging` \| `prod`. |

> **Redação.** O módulo 008 NÃO DEVE logar o conteúdo de `working_memory_ref`
> nem qualquer bytes de payload de checkpoint — apenas ponteiros/metadados
> (`checkpoint_id`, `size_bytes`, `content_hash`). Conteúdo de memória é
> responsabilidade do módulo dono (`010-Memory`).

## 4. Catálogo de Eventos de Log por Componente

> Convenção de nomeação do evento de log (campo lógico `event_name`, usado
> como tag de busca no Seq): `lifecycle.<component_snake>.<ação>`.

| Componente | `event_name` | Nível | Quando | Campos-chave adicionais |
|------------|--------------|-------|--------|---------------------------|
| `LifecycleApiSurface` | `lifecycle.api_surface.received` | Information | Comando recebido (REST/gRPC), antes de qualquer processamento. | `verb`, `idempotency_key` |
| `LifecycleApiSurface` | `lifecycle.api_surface.rejected_schema` | Warning | Payload inválido ou versão de API incompatível. | `error_code=AIOS-LIFECYCLE-0002` |
| `LifecyclePolicyEnforcer` | `lifecycle.pep.decision` | Information | Toda decisão do PEP (allow/deny), 100% capturado. | `decision`, `permission`, `pdp_latency_ms` |
| `LifecyclePolicyEnforcer` | `lifecycle.pep.denied` | Warning | Operação negada pelo PDP. | `error_code=AIOS-LIFECYCLE-0012` |
| `LifecycleCoordinator` | `lifecycle.coordinator.saga_step` | Information | Cada passo do saga de efeito (spawn/checkpoint/migrate). | `saga_id`, `step`, `status` |
| `LifecycleCoordinator` | `lifecycle.coordinator.saga_compensated` | Warning | Compensação executada após falha de passo. | `saga_id`, `failed_step`, `compensation` |
| `StateMachineEngine` | `lifecycle.fsm.transition` | Information | Transição de estado validada e persistida. | `from_state`, `to_state`, `trigger`, `guard_result` |
| `StateMachineEngine` | `lifecycle.fsm.invalid_transition` | Error | Tentativa de transição não permitida pela FSM. | `error_code=AIOS-LIFECYCLE-0002` |
| `AcbStore` | `lifecycle.acb_store.generation_conflict` | Warning | Escrita rejeitada por `generation`/`fencing_token` obsoleto. | `error_code=AIOS-LIFECYCLE-0013`, `expected_generation`, `actual_generation` |
| `SpawnManager` | `lifecycle.spawn.materialized` | Information | Agente materializado com sucesso (novo ou frio). | `latency_ms`, `warm_pool_hit` (bool) |
| `SpawnManager` | `lifecycle.spawn.timeout` | Error | Materialização excedeu `spawn.timeout_ms`. | `error_code=AIOS-LIFECYCLE-0006` |
| `WarmPoolManager` | `lifecycle.warmpool.exhausted` | Warning | Reserva mínima do pool não pôde ser mantida. | `shard`, `available`, `min_required` |
| `HibernationController` | `lifecycle.hibernation.triggered` | Information | Hibernação iniciada (idle ou pressão de RAM). | `reason` (`idle`\|`ram_pressure`), `idle_seconds` |
| `HibernationController` | `lifecycle.hibernation.completed` | Information | RAM liberada; agente marcado `Hibernated`. | `checkpoint_id`, `storage_tier` |
| `CheckpointService` | `lifecycle.checkpoint.created` | Information | Checkpoint consistente criado. | `checkpoint_id`, `size_bytes`, `content_hash`, `duration_ms` |
| `CheckpointService` | `lifecycle.checkpoint.corrupt` | Error | Hash divergente detectado em restore. | `error_code=AIOS-LIFECYCLE-0007`, `checkpoint_id` |
| `SnapshotStore` | `lifecycle.snapshot_store.io_error` | Error | Falha de I/O em MinIO/PostgreSQL. | `error_code=AIOS-LIFECYCLE-0014` |
| `MigrationOrchestrator` | `lifecycle.migration.phase_transition` | Information | Fase da saga de migração avançou. | `migration_id`, `phase`, `source_node`, `target_node` |
| `MigrationOrchestrator` | `lifecycle.migration.compensated` | Warning | Saga de migração compensada (destino indisponível). | `migration_id`, `error_code=AIOS-LIFECYCLE-0010` |
| `LeaseManager` | `lifecycle.lease.acquired` \| `lifecycle.lease.rejected` | Debug/Warning | Ciclo de vida do lease por agente. | `fencing_token`, `holder`, `ttl_ms` |
| `LeaseManager` | `lifecycle.lease.fencing_rejected` | Error | Escrita com `fencing_token` obsoleto rejeitada. | `error_code=AIOS-LIFECYCLE-0013` |
| `ReconciliationController` | `lifecycle.reconciliation.drift_detected` | Warning | Estado observado diverge do desejado. | `agent_id`, `desired_state`, `observed_state` |
| `ReconciliationController` | `lifecycle.reconciliation.corrected` | Information | Deriva corrigida (ex.: runtime morto → `Failed`). | `correction` |
| `LifecycleEventPublisher` | `lifecycle.outbox.enqueued` \| `lifecycle.outbox.published` | Debug/Information | Ciclo de vida do evento no Outbox. | `event_id`, `subject` |
| `LifecycleEventPublisher` | `lifecycle.outbox.publish_failed` | Error | Falha ao publicar; permanece pendente. | `event_id`, `retry_count` |
| `LifecycleEventConsumer` | `lifecycle.consumer.reactive_transition` | Information | Transição disparada por evento externo (009/007/006). | `source_subject`, `trigger` |
| `TombstoneManager` | `lifecycle.tombstone.purged` | Information | Expurgo de snapshots/checkpoints/ACB de agente terminado. | `agent_id`, `purged_checkpoints`, `retention_days` |

## 5. Correlação com Traces (OpenTelemetry)

- Todo log emitido dentro do processamento de uma transição DEVE ser emitido
  **dentro do span ativo** correspondente, permitindo ao Serilog enriquecer
  automaticamente `trace_id`/`span_id` via o *Activity* do .NET (integração
  OTel↔Serilog padrão da stack, ver `../024-Observability/`).
- O par (`trace_id`, `span_id`) DEVE permitir pivotar do Seq (log) para o
  backend de tracing (Grafana Tempo/Jaeger) em um clique — os dashboards do
  módulo 008 DEVEM incluir *data link* configurado.
- Logs de eventos assíncronos (relay do Outbox, `ReconciliationController`,
  hibernação disparada por pressão de RAM) que não têm uma requisição
  HTTP/gRPC associada DEVEM iniciar um novo trace raiz com
  `source=urn:aios:<tenant>:service:agent-lifecycle` como atributo de span.
- Sagas de longa duração (migração) DEVEM propagar o **mesmo** `trace_id` por
  todas as fases (`quiesce`→`checkpoint`→`transfer`→`materialize`→`cutover`→
  `cleanup`), usando spans filhos por fase — permite reconstruir a linha do
  tempo completa da migração a partir de um único trace.

## 6. Exemplos de Log Estruturado (JSON, formato de saída do sink Seq)

**Transição de FSM concluída (Suspended → Hibernated):**
```json
{
  "timestamp": "2026-07-20T14:02:11.114Z",
  "level": "Information",
  "message_template": "Transição {FromState}->{ToState} concluída em {LatencyMs} ms",
  "event_name": "lifecycle.fsm.transition",
  "component": "StateMachineEngine",
  "trace_id": "4bf92f3577b34da6a3ce929d0e0e4736",
  "span_id": "00f067aa0ba902b7",
  "tenant_id": "acme",
  "agent_id": "01J9Z8Q6H7K2M4N6P8R0S2T4V6",
  "generation": 3,
  "from_state": "Suspended",
  "to_state": "Hibernated",
  "trigger": "hibernate",
  "actor": "reconciler",
  "checkpoint_id": "01J9Z9A1B2C3D4E5F6G7H8J9K0",
  "latency_ms": 14,
  "service_version": "0.9.1+f1e2d3c",
  "environment": "prod"
}
```

**Falha por checkpoint corrompido:**
```json
{
  "timestamp": "2026-07-20T14:05:47.902Z",
  "level": "Error",
  "message_template": "Checkpoint {CheckpointId} corrompido: hash divergente no restore",
  "event_name": "lifecycle.checkpoint.corrupt",
  "component": "CheckpointService",
  "trace_id": "7a3c1e9f2b4d4e6a8c0f2a4c6e8b0d2f",
  "span_id": "1a2b3c4d5e6f7081",
  "tenant_id": "acme",
  "agent_id": "01J9Z8Q6H7K2M4N6P8R0S2T4V6",
  "generation": 5,
  "checkpoint_id": "01J9Z9A1B2C3D4E5F6G7H8J9K0",
  "error_code": "AIOS-LIFECYCLE-0007",
  "service_version": "0.9.1+f1e2d3c",
  "environment": "prod"
}
```

**Fencing token rejeitado (split-brain evitado):**
```json
{
  "timestamp": "2026-07-20T14:11:03.500Z",
  "level": "Error",
  "message_template": "Escrita rejeitada: fencing_token {ActualToken} obsoleto (esperado {ExpectedToken})",
  "event_name": "lifecycle.lease.fencing_rejected",
  "component": "LeaseManager",
  "trace_id": "9e8d7c6b5a4938271605f4e3d2c1b0a9",
  "span_id": "2b3c4d5e6f708192",
  "tenant_id": "acme",
  "agent_id": "01J9Z8Q6H7K2M4N6P8R0S2T4V6",
  "fencing_token": 41,
  "error_code": "AIOS-LIFECYCLE-0013",
  "exception": {
    "type": "StaleFencingTokenException",
    "message": "expected_generation=42 actual=41"
  },
  "service_version": "0.9.1+f1e2d3c",
  "environment": "prod"
}
```

## 7. Retenção e Ciclo de Vida do Log

| Ambiente | Retenção no Seq | Retenção arquivada (frio) | Observação |
|----------|-------------------|------------------------------|------------|
| `prod` | 30 dias (consulta interativa) | 180 dias em armazenamento frio (MinIO, comprimido) | Alinhado à retenção mínima de auditoria de `025-Audit`, que é superior e regida por LGPD/GDPR. |
| `staging` | 14 dias | não aplicável | — |
| `dev` | 3 dias | não aplicável | — |

- Logs **NÃO DEVEM** ser a fonte de verdade regulatória — essa função é de
  `025-Audit` (retenção própria, imutabilidade, cadeia de custódia).
- A rotação/expurgo de logs no Seq é automatizada; nenhuma operação manual de
  exclusão seletiva é permitida (preservação de integridade forense mínima
  dentro da janela de retenção).
- O expurgo executado pelo `TombstoneManager` (retenção LGPD de agentes
  terminados) DEVE, ele próprio, gerar um log `lifecycle.tombstone.purged` e
  um evento auditável (025) — a remoção de dados de um agente não pode ser
  silenciosa.
- Logs contendo `tenant_id` DEVEM respeitar a mesma fronteira de isolamento
  multi-tenant no controle de acesso ao Seq (RBAC por tenant, ver
  `021-Security`).

## 8. Referências

- Catálogo de métricas (correlacionadas por `trace_id`): `./Metrics.md`
- Dashboards e alertas: `./Monitoring.md`
- Auditoria imutável (fonte regulatória): `../025-Audit/README.md`
- Correlação e cabeçalhos: `../003-RFC/RFC-0001-Architecture-Baseline.md` §5.6
- Catálogo de erros do módulo: `./API.md` §Erros, `_DESIGN_BRIEF.md` §5.3
- Infraestrutura de observabilidade: `../024-Observability/README.md`
