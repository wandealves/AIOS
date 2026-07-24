---
Documento: API
Módulo: 024-Observability
Status: Draft
Versão: 0.1
Última atualização: 2026-07-23
Responsável (RACI-A): Arquiteto do Módulo 024-Observability
ADRs relacionados: ADR-0010, ADR-0240, ADR-0241, ADR-0242, ADR-0243, ADR-0247
RFCs relacionados: RFC-0001, RFC-0240, RFC-0241
Depende de: _DESIGN_BRIEF.md §5, 004-API (catálogo de erros e registros), 021-Security, 022-Policy
---

# 024-Observability — Contratos de API

> Autenticação, envelope de erro, idempotência, correlação e versionamento seguem
> `../003-RFC/RFC-0001-Architecture-Baseline.md` §5.4–§5.7 e §6 — **referenciados, não
> redefinidos**. Pacote gRPC: `aios.observability.v1`. Base REST: `/v1/observability`.

## 1. Três superfícies, três contratos

| Superfície | Protocolo | Uso | Regime de falha |
|-----------|-----------|-----|-----------------|
| **Ingestão** | **OTLP** (padrão externo) | Emissão de métricas, traces e logs por todos os módulos. | ***fail-open*** (ADR-0243) |
| **Consulta** | REST/gRPC | PromQL, TraceQL, logs; painéis e investigação. | ***fail-closed*** (PEP) |
| **Administração** | REST/gRPC | Registro de sinais, SLOs, regras, silêncios, dashboards, retenção. | ***fail-closed*** (PEP) |

A ingestão usa o protocolo **OTLP** e não a convenção interna porque é o contrato que
os SDKs de todos os módulos já falam — obrigar cada emissor a implementar uma API
própria do AIOS transformaria a instrumentação em trabalho de integração.

## 2. Ingestão (OTLP)

### 2.1 Endpoints

| Endpoint | Protocolo | Sinal |
|----------|-----------|-------|
| `:4317` `/opentelemetry.proto.collector.metrics.v1.MetricsService/Export` | OTLP gRPC | Métricas |
| `:4317` `/opentelemetry.proto.collector.trace.v1.TraceService/Export` | OTLP gRPC | Traces |
| `:4317` `/opentelemetry.proto.collector.logs.v1.LogsService/Export` | OTLP gRPC | Logs |
| `:4318` `POST /v1/{metrics,traces,logs}` | OTLP HTTP | Equivalente HTTP |

### 2.2 Regras normativas

| # | Regra |
|---|-------|
| 1 | O emissor **DEVE** autenticar-se por **mTLS** com certificado de workload (`../021-Security/`). |
| 2 | O `tenant` **DEVE** vir do atributo de recurso `aios.tenant`; divergente da identidade do certificado ⇒ `AIOS-OBS-0003`. |
| 3 | Os atributos de recurso `service.name` e `deployment.environment` **DEVEM** estar presentes; os demais são injetados pelo `ResourceEnricher`. |
| 4 | O `traceparent` **DEVE** ser propagado (RFC-0001 §5.6) — sem ele não há pivô log↔trace. |
| 5 | A resposta **NÃO DEVE** aplicar backpressure: sob saturação, o gateway responde sucesso e **descarta**, contabilizando em `aios_obs_dropped_total` (ADR-0243). |
| 6 | Sinal não registrado **NÃO DEVE** derrubar o lote: apenas aquele sinal é quarentenado, reportado por **evento e contador**, não por erro de transporte. |

### 2.3 Exemplo — métrica registrada e desconhecida no mesmo lote

```json
POST /v1/metrics                                   (OTLP HTTP, mTLS)
{ "resourceMetrics": [ {
    "resource": { "attributes": [
        {"key":"service.name","value":{"stringValue":"kernel"}},
        {"key":"deployment.environment","value":{"stringValue":"production"}},
        {"key":"aios.tenant","value":{"stringValue":"acme"}} ] },
    "scopeMetrics": [ { "metrics": [
        { "name": "aios_kernel_syscall_latency_ms", "histogram": { "dataPoints": [ ] } },
        { "name": "kernel_temp_debug_counter",      "sum":       { "dataPoints": [ ] } } ] } ] } ] }
```

```
HTTP/1.1 200 OK
{ "partialSuccess": { "rejectedDataPoints": 1,
                      "errorMessage": "kernel_temp_debug_counter: unregistered signal (quarantined)" } }

Efeitos:
  · aios_kernel_syscall_latency_ms → persistida
  · kernel_temp_debug_counter      → quarentenada
      + aios_obs_signal_quarantined_total{reason="unregistered"} +1
      + evento aios._platform.telemetry.signal.quarantined
```

O lote **não** é rejeitado: uma métrica de depuração esquecida em um serviço não pode
apagar toda a telemetria dele.

## 3. Consulta

| Operação | REST | gRPC | Idempotente | Capability |
|----------|------|------|-------------|------------|
| Métricas (PromQL) | `POST /v1/observability/query/metrics` | `QueryMetrics` | sim (leitura) | `obs:query:metrics` |
| Traces (filtro) | `POST /v1/observability/query/traces` | `QueryTraces` | sim (leitura) | `obs:query:traces` |
| Trace por id | `GET /v1/observability/traces/{traceId}` | `GetTrace` | sim (leitura) | `obs:query:traces` |
| Logs | `POST /v1/observability/query/logs` | `QueryLogs` | sim (leitura) | `obs:query:logs` |
| SLO e budget | `GET /v1/observability/slos/{key}` | `GetSlo` | sim (leitura) | `obs:slo:read` |

```yaml
openapi: 3.1.0
info: { title: AIOS Observability API, version: "1.0" }
paths:
  /v1/observability/query/metrics:
    post:
      operationId: queryMetrics
      parameters:
        - { name: X-AIOS-Tenant, in: header, required: true, schema: { type: string } }
        - { name: traceparent,   in: header, required: true, schema: { type: string } }
      requestBody:
        required: true
        content:
          application/json:
            schema:
              type: object
              required: [query, from, to]
              properties:
                query: { type: string, example: 'histogram_quantile(0.99, rate(aios_kernel_syscall_latency_ms_bucket[5m]))' }
                from:  { type: string, format: date-time }
                to:    { type: string, format: date-time }
                step:  { type: string, example: "30s" }
      responses:
        '200': { description: Série(s) resultante(s) }
        '403': { $ref: '#/components/responses/ObsError' }   # AIOS-OBS-0003
        '429': { $ref: '#/components/responses/ObsError' }   # AIOS-OBS-0010
        '503': { $ref: '#/components/responses/ObsError' }   # AIOS-OBS-0011
```

**Exemplo — pivô log → trace:**

```http
POST /v1/observability/query/logs HTTP/1.1
X-AIOS-Tenant: acme
{ "filter": "event = 'policy.decision.denied' and level = 'Information'",
  "from": "2026-07-23T10:00:00Z", "to": "2026-07-23T10:15:00Z", "limit": 50 }
```

```json
{ "entries": [ { "timestamp": "2026-07-23T10:12:03.441Z",
                 "service": "022-policy", "event": "policy.decision.denied",
                 "traceId": "4bf92f3577b34da6a3ce929d0e0e4736",
                 "tenantId": "acme", "reasonCode": "explicit_deny" } ] }
```

```http
GET /v1/observability/traces/4bf92f3577b34da6a3ce929d0e0e4736 HTTP/1.1
```

O `traceId` do log é o mesmo do trace (RFC-0001 §5.6): a investigação é um salto, não
uma correlação manual por horário.

## 4. Administração

| Operação | REST | gRPC | Idempotente | Capability |
|----------|------|------|-------------|------------|
| Registrar sinal | `POST /v1/observability/signals` | `RegisterSignal` | sim (key) | `obs:signal:register` |
| Ler/listar sinais | `GET /v1/observability/signals[/{name}]` | `GetSignal`/`ListSignals` | sim (leitura) | `obs:signal:read` |
| Quarentenar/reabilitar | `POST /v1/observability/signals/{name}:quarantine` / `:reinstate` | `SetSignalState` | sim | `obs:signal:manage` |
| Orçamento de cardinalidade | `PUT /v1/observability/cardinality-budgets/{scope}` | `PutCardinalityBudget` | sim | `obs:cardinality:manage` |
| Publicar SLO | `POST /v1/observability/slos` | `PutSlo` | sim (key) | `obs:slo:write` |
| Publicar regra de alerta | `PUT /v1/observability/alert-rules/{key}` | `PutAlertRule` | sim | `obs:alert:write` |
| Reconhecer alerta | `POST /v1/observability/alerts/{id}:ack` | `AcknowledgeAlert` | sim | `obs:alert:ack` |
| Criar silêncio | `POST /v1/observability/silences` | `CreateSilence` | sim (key) | `obs:silence:write` |
| Revogar silêncio | `POST /v1/observability/silences/{urn}:revoke` | `RevokeSilence` | sim | `obs:silence:write` |
| Publicar dashboard | `POST /v1/observability/dashboards` | `PutDashboard` | sim | `obs:dashboard:write` |
| Publicar runbook | `PUT /v1/observability/runbooks/{key}` | `PutRunbook` | sim | `obs:runbook:write` |
| Definir retenção | `PUT /v1/observability/retention/{tier}` | `PutRetentionPolicy` | sim | `obs:retention:manage` |

**Exemplo — registro de sinal:**

```http
POST /v1/observability/signals HTTP/1.1
X-AIOS-Tenant: _platform
Idempotency-Key: 01J9ZD00000000000000000000
Content-Type: application/json

{ "signalKind": "metric",
  "name": "aios_kernel_syscall_latency_ms",
  "ownerModule": "006-Kernel",
  "metricType": "histogram",
  "unit": "ms",
  "allowedLabels": ["verb", "component", "tenant"],
  "forbiddenLabels": ["agent_urn", "task_urn"],
  "bucketBoundaries": [5, 20, 50, 250, 1000],
  "dataClass": "operational",
  "retentionTier": "raw" }
```

```json
HTTP/1.1 201 Created
{ "urn": "urn:aios:_platform:event:01J9ZE00000000000000000000",
  "name": "aios_kernel_syscall_latency_ms", "descriptorVersion": 1,
  "state": "Registered" }
```

Os buckets `[5, 20, 50, 250, 1000]` alinham-se aos limiares de SLO do
`../006-Kernel/`: buckets automáticos não caem nas fronteiras do objetivo, e
`histogram_quantile` sobre eles devolve um número que **parece** um p99.

## 5. gRPC (`aios.observability.v1`)

```proto
syntax = "proto3";
package aios.observability.v1;

service ObservabilityQuery {
  rpc QueryMetrics (MetricQuery) returns (MetricResult);
  rpc QueryTraces  (TraceQuery)  returns (TraceResult);
  rpc GetTrace     (TraceRef)    returns (Trace);
  rpc QueryLogs    (LogQuery)    returns (LogResult);
  rpc GetSlo       (SloRef)      returns (SloStatus);
}

service ObservabilityAdmin {
  rpc RegisterSignal       (SignalDescriptorDraft) returns (SignalDescriptor);
  rpc GetSignal            (SignalRef)             returns (SignalDescriptor);
  rpc ListSignals          (ListSignalsRequest)    returns (ListSignalsResponse);
  rpc SetSignalState       (SignalStateRequest)    returns (SignalDescriptor);
  rpc PutCardinalityBudget (CardinalityBudget)     returns (CardinalityBudget);
  rpc PutSlo               (SloDefinition)         returns (SloDefinition);
  rpc PutAlertRule         (AlertRule)             returns (AlertRule);
  rpc AcknowledgeAlert     (AlertRef)              returns (AlertInstance);
  rpc CreateSilence        (Silence)               returns (Silence);
  rpc RevokeSilence        (SilenceRef)            returns (Empty);
  rpc PutDashboard         (Dashboard)             returns (Dashboard);
  rpc PutRunbook           (Runbook)               returns (Runbook);
  rpc PutRetentionPolicy   (RetentionPolicy)       returns (RetentionPolicy);
}

message SloStatus {
  string  slo_key         = 1;
  int32   slo_version     = 2;
  double  sli_value       = 3;
  double  objective       = 4;
  double  budget_consumed = 5;   // 0..1
  double  burn_rate_1h    = 6;
  double  burn_rate_6h    = 7;
  bool    breached        = 8;
  string  evaluated_at    = 9;   // RFC 3339 UTC
}
```

Mapeamento de erro gRPC: `status` → `google.rpc.Status` + `ErrorInfo` com o mesmo
`code` (RFC-0001 §5.4).

## 6. Catálogo de códigos de erro

> Formato RFC-0001 §5.4. Domínio reservado: **`OBS`** (0001–0099), registrado em
> `../004-API/Errors.md` conforme RFC-0001 §8 (ratificação em **ADR-0240**).

| Código | HTTP | `retriable` | Significado |
|--------|------|-------------|-------------|
| `AIOS-OBS-0001` | 404 | não | Sinal, SLO, regra, alerta, silêncio, dashboard ou runbook não encontrado. |
| `AIOS-OBS-0002` | 422 | não | Sinal não registrado ou em quarentena. |
| `AIOS-OBS-0003` | 403 | não | Tenant divergente da identidade do emissor/consultante. |
| `AIOS-OBS-0004` | 422 | não | Nome, tipo, unidade ou label fora da convenção (RFC-0240). |
| `AIOS-OBS-0005` | 429 | sim | Orçamento de cardinalidade excedido para o escopo. |
| `AIOS-OBS-0006` | 409 | sim | Conflito de concorrência otimista (`version`). |
| `AIOS-OBS-0007` | 409 | não | Transição de estado inválida (alerta ou sinal). |
| `AIOS-OBS-0008` | 422 | não | Regra de alerta sem runbook válido. |
| `AIOS-OBS-0009` | 422 | não | SLO/dashboard referencia sinal não registrado, ou objetivo fora de (0,1). |
| `AIOS-OBS-0010` | 429 | sim | Limite de custo/tempo de consulta excedido. |
| `AIOS-OBS-0011` | 503 | sim | Armazenamento de telemetria indisponível (consulta). |
| `AIOS-OBS-0012` | 413 | não | Lote OTLP acima do tamanho máximo aceito. |
| `AIOS-OBS-0013` | 403 | não | Silêncio sem prazo, sem justificativa ou acima do máximo. |
| `AIOS-OBS-0014` | 409 | não | Descritor de sinal imutável: registre nova versão. |
| `AIOS-OBS-0015` | 422 | não | Retenção incompatível com a classificação do dado (`pii` sem base legal). |

**Envelope (RFC 7807), exemplo:**

```json
HTTP/1.1 422 Unprocessable Entity
Content-Type: application/problem+json

{ "type": "https://docs.aios/errors/signal-convention-violation",
  "title": "Signal Convention Violation",
  "status": 422,
  "code": "AIOS-OBS-0004",
  "detail": "Label 'agent_urn' é de identidade e não pode ser usado em métrica.",
  "instance": "urn:aios:_platform:event:01J9ZE00000000000000000000",
  "traceId": "4bf92f3577b34da6a3ce929d0e0e4736",
  "timestamp": "2026-07-23T10:20:00.000Z",
  "retriable": false }
```

> **Nota deliberada sobre a ingestão.** Os códigos acima valem para **consulta e
> administração**. Na **ingestão**, problemas de admissão viram **contador e evento**,
> nunca erro de transporte devolvido ao emissor (ADR-0243). Um `429` na ingestão faria
> o SDK do serviço observado gastar CPU em retry justamente quando o sistema está sob
> pressão — transformando a falha do observador em falha do observado.

## 7. Idempotência, paginação e versionamento

| Aspecto | Regra |
|---------|-------|
| Idempotência | `Idempotency-Key` obrigatório em mutações administrativas; resultado persistido ≥ 24 h (RFC-0001 §5.5); repetição devolve o mesmo resultado (NFR-015). |
| Ingestão | OTLP é *at-least-once* do lado do emissor; a deduplicação de datapoint é por `(nome, labels, timestamp)`. |
| Consulta | Leitura pura: repetir é sempre seguro; não exige chave. |
| Paginação | `pageToken`/`pageSize` (máx. 500) em `ListSignals` e nas consultas de log. |
| Versionamento | Caminho `/v1` + `X-AIOS-Api-Version`; gRPC em `aios.observability.v1` (RFC-0001 §5.7). |
| Evolução | Novos `signal_kind`, `retention_tier` e severidades são **aditivos**; clientes **DEVEM** tolerar valores desconhecidos. |

## 8. Limites operacionais

| Limite | Valor | Chave | Erro |
|--------|-------|-------|------|
| Tamanho do lote OTLP | 4 MiB | `obs.ingest.max_batch_mb` | `AIOS-OBS-0012` |
| Séries por tenant | 1.000.000 | `obs.cardinality.max_series_per_tenant` | `AIOS-OBS-0005` |
| Séries por sinal | 10.000 | `obs.cardinality.max_series_per_signal` | `AIOS-OBS-0005` |
| Duração de consulta | 30 s | `obs.query.max_duration_s` | `AIOS-OBS-0010` |
| Séries varridas por consulta | 5.000.000 | `obs.query.max_series_scanned` | `AIOS-OBS-0010` |
| Prazo de silêncio | 72 h | `obs.alert.max_silence_h` | `AIOS-OBS-0013` |
| Página máxima | 500 | — | `AIOS-OBS-0004` |

## 9. Referências

- Fonte única de verdade: `./_DESIGN_BRIEF.md` §5
- Contratos centrais: `../003-RFC/RFC-0001-Architecture-Baseline.md` §5.4–§5.7
- Registros (erros, domínios): `../004-API/Errors.md`, `../004-API/Events.md`
- Eventos: `./Events.md` · Estruturas: `./ClassDiagrams.md` · Exemplos: `./Examples.md`

*Fim de `API.md`.*
