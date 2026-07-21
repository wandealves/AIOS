---
Documento: Logging
Módulo: 006-Kernel
Status: Draft
Versão: 0.1
Última atualização: 2026-07-20
Responsável (RACI-A): Arquiteto do Módulo 006-Kernel
ADRs relacionados: ADR-0010 (Observabilidade e Auditoria por Construção), ADR-0063 (PEP/PDP), ADR-0069 (Domínios de erro do Kernel)
RFCs relacionados: RFC-0001 (baseline, §5.4 erro, §5.6 correlação)
Depende de: 024-Observability, 025-Audit, 021-Security
---

# 006-Kernel — Logging

> Este documento define o **contrato de logging estruturado** do Kernel: níveis,
> campos obrigatórios, catálogo de eventos de log por componente, correlação com
> traces OTel e política de retenção. A pilha usada é **Serilog** (produção do
> log estruturado em .NET 10) → **Seq** (armazenamento/consulta), conforme
> `_DESIGN_BRIEF.md` e stack global. Logging **NÃO substitui** a auditoria
> imutável de `025-Audit` (que persiste a trilha regulatória); logging é para
> diagnóstico operacional, com retenção mais curta.

## 1. Objetivo e Princípios

O logging do Kernel DEVE:

1. Ser **estruturado** (JSON), nunca texto livre sem campos — todo log é uma
   mensagem-modelo (`MessageTemplate` do Serilog) com propriedades tipadas.
2. Ser **correlacionável**: todo log emitido durante o processamento de uma
   syscall DEVE carregar `trace_id`, `span_id` e `tenant_id` (RFC-0001 §5.6).
3. **Nunca conter segredos** (tokens, chaves, payload de LLM bruto) nem PII
   além do estritamente necessário — URNs são identificadores opacos.
4. Ser **acionável**: cada linha de log em nível `Warning`/`Error` DEVE permitir,
   sozinha, identificar o componente, a syscall, o agente/tenant e a causa.
5. Não duplicar a função da auditoria: decisões de política e mutações
   privilegiadas são logadas para diagnóstico **e** auditadas via `025-Audit`
   separadamente (double-write intencional, propósitos distintos).

## 2. Níveis de Log e Política de Uso

| Nível | Uso no Kernel | Exemplos |
|-------|----------------|----------|
| `Verbose` | Detalhe de depuração interna, desabilitado em produção por padrão. | Conteúdo de payload de cache do PEP (redigido). |
| `Debug` | Fluxo interno útil em troubleshooting, habilitável por tenant/agente. | Entrada/saída de `AcbStore.TryUpdate` com `version` esperada. |
| `Information` | Eventos de negócio normais (uma linha por syscall processada, por transição de FSM). | `Syscall spawn concluída` com `verb`, `state_to`, `latency_ms`. |
| `Warning` | Degradação tolerada ou comportamento anômalo não fatal. | Cache miss do PEP acima do limiar; retry de OCC. |
| `Error` | Falha que impede o processamento da syscall corrente. | `AIOS-KERNEL-0005` (broker de recurso falhou). |
| `Fatal` | Falha que compromete o processo/serviço (deve gerar crash controlado + alerta). | Falha irrecuperável de conexão com PostgreSQL na inicialização. |

**Regra de amostragem.** Logs `Information` de syscalls de alta frequência
(`get_quota`, `recall`) PODEM ser amostrados (ex.: 1:10) sob carga elevada,
mas `Warning`/`Error`/`Fatal` e todo log de **decisão do PEP** NÃO DEVEM ser
amostrados (100% de captura — requisito de auditabilidade, NFR-011).

## 3. Campos Obrigatórios (todo evento de log)

| Campo | Tipo | Obrigatório | Descrição |
|-------|------|-------------|-----------|
| `timestamp` | `datetime` (RFC 3339, UTC) | sim | Momento de emissão. |
| `level` | `string` | sim | Um dos níveis da §2. |
| `message_template` | `string` | sim | Template Serilog (ex.: `"Syscall {Verb} concluída em {LatencyMs} ms"`). |
| `component` | `string` | sim | Um dos 12 componentes internos (§2.1 do brief), ex.: `SyscallGateway`, `CapabilityEnforcer`. |
| `trace_id` | `string` | sim | Extraído de `traceparent` (RFC-0001 §5.6). |
| `span_id` | `string` | sim | Span corrente do OTel. |
| `tenant_id` | `string` | sim (exceto logs de bootstrap) | Contexto de isolamento multi-tenant. |
| `agent_urn` | `string` | quando aplicável | URN do agente-alvo da syscall. |
| `verb` | `string` | quando aplicável | Um dos 11 verbos de syscall (§5 do brief). |
| `idempotency_key` | `string` | quando presente na requisição | Correlaciona repetições. |
| `decision` | `string` | quando envolve PEP | `allow` \| `deny`. |
| `error_code` | `string` | quando `level ∈ {Error, Fatal}` ou `decision=deny` | `AIOS-<DOMINIO>-<NNNN>`. |
| `latency_ms` | `number` | quando aplicável | Duração da operação logada. |
| `acb_state_from` / `acb_state_to` | `string` | em logs de transição FSM | Estados da §4 do brief. |
| `exception` | `object` | quando `level ∈ {Error, Fatal}` | Tipo, mensagem, stack trace (sem PII). |
| `service_version` | `string` | sim | Versão do build do Kernel (SemVer + hash de commit). |
| `environment` | `string` | sim | `dev` \| `staging` \| `prod`. |

> **Redação.** Campos que poderiam conter PII (ex.: conteúdo de `data` de
> syscalls `remember`/`recall`) NÃO DEVEM ser logados; o Kernel loga apenas
> ponteiros/metadados (ex.: tamanho do payload, namespace de memória), nunca o
> conteúdo — conteúdo é responsabilidade de auditoria do módulo dono (`010`).

## 4. Catálogo de Eventos de Log por Componente

> Convenção de nomeação do evento de log (campo lógico `event_name`, usado como
> tag de busca no Seq): `kernel.<component_snake>.<ação>`.

| Componente | `event_name` | Nível | Quando | Campos-chave adicionais |
|------------|--------------|-------|--------|---------------------------|
| `SyscallGateway` | `kernel.syscall_gateway.received` | Information | Syscall recebida, antes de qualquer processamento. | `verb`, `idempotency_key`, `content_length` |
| `SyscallGateway` | `kernel.syscall_gateway.rejected_schema` | Warning | Payload/versão de ABI inválida. | `error_code=AIOS-SYSCALL-0001`, `abi_version` |
| `SyscallGateway` | `kernel.syscall_gateway.rate_limited` | Warning | Rate-limit por agente excedido. | `error_code=AIOS-QUOTA-0002` |
| `CapabilityEnforcer` | `kernel.pep.decision` | Information | Toda decisão do PEP (allow/deny), 100% capturado. | `decision`, `capability`, `cache` (`hit`/`miss`), `pdp_latency_ms` |
| `CapabilityEnforcer` | `kernel.pep.fail_closed` | Error | PDP indisponível e `fail_mode=closed` negou por segurança. | `error_code=AIOS-CAP-0003` |
| `PolicyClient` | `kernel.policy_client.circuit_open` | Error | Circuit breaker do PDP abriu. | `dependency=022-policy`, `failure_rate` |
| `AcbStore` | `kernel.acb_store.transition` | Information | Transição de estado da FSM persistida com sucesso. | `acb_state_from`, `acb_state_to`, `version` |
| `AcbStore` | `kernel.acb_store.occ_conflict` | Warning | Conflito de concorrência otimista. | `expected_version`, `actual_version`, `retry_count` |
| `AcbStore` | `kernel.acb_store.invalid_transition` | Error | Tentativa de transição não permitida pela FSM. | `error_code=AIOS-KERNEL-0002` |
| `ResourceQuotaManager` | `kernel.quota.reserved` \| `kernel.quota.consumed` \| `kernel.quota.released` | Debug/Information | Ciclo de vida de uma reserva de cota. | `resource`, `amount`, `subject_urn` |
| `ResourceQuotaManager` | `kernel.quota.exceeded` | Warning | Cota `hard` excedida. | `error_code=AIOS-QUOTA-0001`, `resource`, `limit`, `attempted` |
| `LifecycleCoordinator` | `kernel.lifecycle.saga_step` | Information | Cada passo da saga (`spawn`/`kill`/`checkpoint`). | `saga_id`, `step`, `status` |
| `LifecycleCoordinator` | `kernel.lifecycle.saga_compensated` | Warning | Compensação executada após falha de passo. | `saga_id`, `failed_step`, `compensation` |
| `SchedulerClient` | `kernel.scheduler_client.timeout` | Error | Timeout aguardando decisão de admissão. | `error_code=AIOS-KERNEL-0003` |
| `LifecycleClient` | `kernel.lifecycle_client.boot_timeout` | Error | Timeout de materialização do runtime. | `error_code=AIOS-KERNEL-0004` |
| `ResourceBrokerRouter` | `kernel.broker.forwarded` | Debug | Syscall de recurso encaminhada ao módulo dono. | `target_module`, `verb` |
| `ResourceBrokerRouter` | `kernel.broker.failed` | Error | Broker de recurso retornou erro/timeout. | `error_code=AIOS-KERNEL-0005`, `target_module` |
| `IdempotencyStore` | `kernel.idempotency.replay` | Information | Requisição duplicada detectada; resultado reaproveitado. | `idempotency_key` |
| `IdempotencyStore` | `kernel.idempotency.conflict` | Warning | Mesma chave, payload divergente. | `error_code=AIOS-SYSCALL-0002` |
| `EventEmitter` | `kernel.outbox.enqueued` | Debug | Evento gravado no Outbox na mesma transação da mutação. | `event_id`, `subject` |
| `EventEmitter` | `kernel.outbox.published` | Information | Relay publicou o evento no JetStream com sucesso. | `event_id`, `subject`, `relay_latency_ms` |
| `EventEmitter` | `kernel.outbox.publish_failed` | Error | Falha ao publicar; permanece pendente para nova tentativa. | `event_id`, `retry_count` |
| `CheckpointManager` | `kernel.checkpoint.created` | Information | Snapshot durável concluído. | `checkpoint_ref`, `duration_ms` |
| `KernelTelemetry` | `kernel.telemetry.audit_forward_failed` | Error | Falha ao encaminhar evento de auditoria para `025-Audit`. | `error_code` |

## 5. Correlação com Traces (OpenTelemetry)

- Todo log emitido dentro do processamento de uma syscall DEVE ser emitido
  **dentro do span ativo** correspondente, permitindo ao Serilog enriquecer
  automaticamente `trace_id`/`span_id` via o *Activity* do .NET (integração
  OTel↔Serilog padrão da stack, ver `../024-Observability/`).
- O par (`trace_id`, `span_id`) DEVE permitir pivotar do Seq (log) para o
  backend de tracing (Grafana Tempo/Jaeger, conforme `024-Observability`) em
  um clique — os dashboards do Kernel DEVEM incluir *data link* configurado.
- Logs de eventos assíncronos (relay do Outbox, reconciliação de cota) que
  não têm uma requisição HTTP/gRPC associada DEVEM iniciar um novo trace raiz
  com `source=urn:aios:<tenant>:service:kernel` como atributo de span.

## 6. Exemplos de Log Estruturado (JSON, formato de saída do sink Seq)

**Decisão de PEP (allow, cache hit):**
```json
{
  "timestamp": "2026-07-20T14:02:11.114Z",
  "level": "Information",
  "message_template": "Decisão do PEP: {Decision} para capability {Capability}",
  "event_name": "kernel.pep.decision",
  "component": "CapabilityEnforcer",
  "trace_id": "4bf92f3577b34da6a3ce929d0e0e4736",
  "span_id": "00f067aa0ba902b7",
  "tenant_id": "acme",
  "agent_urn": "urn:aios:acme:agent:01J9Z8Q6H7K2M4N6P8R0S2T4V6",
  "verb": "invoke_tool",
  "decision": "allow",
  "capability": "tool:invoke:http-fetch",
  "cache": "hit",
  "pdp_latency_ms": 0,
  "service_version": "1.4.2+a1b2c3d",
  "environment": "prod"
}
```

**Falha por cota excedida:**
```json
{
  "timestamp": "2026-07-20T14:05:47.902Z",
  "level": "Warning",
  "message_template": "Cota excedida: {Resource} limite={Limit} tentado={Attempted}",
  "event_name": "kernel.quota.exceeded",
  "component": "ResourceQuotaManager",
  "trace_id": "7a3c1e9f2b4d4e6a8c0f2a4c6e8b0d2f",
  "span_id": "1a2b3c4d5e6f7081",
  "tenant_id": "acme",
  "agent_urn": "urn:aios:acme:agent:01J9Z8Q6H7K2M4N6P8R0S2T4V6",
  "verb": "route_model",
  "error_code": "AIOS-QUOTA-0001",
  "resource": "tokens",
  "limit": 1000000,
  "attempted": 1004200,
  "service_version": "1.4.2+a1b2c3d",
  "environment": "prod"
}
```

**Falha irrecuperável de materialização (Failed via T-05):**
```json
{
  "timestamp": "2026-07-20T14:11:03.500Z",
  "level": "Error",
  "message_template": "Timeout de materialização do runtime para agente {AgentUrn}",
  "event_name": "kernel.lifecycle_client.boot_timeout",
  "component": "LifecycleClient",
  "trace_id": "9e8d7c6b5a4938271605f4e3d2c1b0a9",
  "span_id": "2b3c4d5e6f708192",
  "tenant_id": "acme",
  "agent_urn": "urn:aios:acme:agent:01J9Z8Q6H7K2M4N6P8R0S2T4V6",
  "verb": "spawn",
  "acb_state_from": "Admitted",
  "acb_state_to": "Failed",
  "error_code": "AIOS-KERNEL-0004",
  "exception": {
    "type": "TimeoutException",
    "message": "boot_timeout_ms=5000 excedido"
  },
  "service_version": "1.4.2+a1b2c3d",
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
- Logs contendo `tenant_id` DEVEM respeitar a mesma fronteira de isolamento
  multi-tenant no controle de acesso ao Seq (RBAC por tenant, ver `021-Security`).

## 8. Referências

- Catálogo de métricas (correlacionadas por `trace_id`): `./Metrics.md`
- Dashboards e alertas: `./Monitoring.md`
- Auditoria imutável (fonte regulatória): `../025-Audit/README.md`
- Correlação e cabeçalhos: `../003-RFC/RFC-0001-Architecture-Baseline.md` §5.6
- Catálogo de erros do Kernel: `./API.md` §Erros, `_DESIGN_BRIEF.md` §5.2
- Infraestrutura de observabilidade: `../024-Observability/README.md`
