---
Documento: Events
Módulo: 004-API
Status: Draft
Versão: 0.1
Última atualização: 2026-07-21
Responsável (RACI-A): Arquiteto do Módulo 004-API
ADRs relacionados: ADR-0043, ADR-0045, ADR-0048
RFCs relacionados: RFC-0001 (§5.2–§5.3, §8), RFC-0041
Depende de: ../020-Communication/README.md, ./StateMachine.md, ./Database.md
---

# 004-API — Catálogo de Eventos NATS

> Envelope CloudEvents e convenção de subjects: `../003-RFC/RFC-0001-Architecture-Baseline.md`
> §5.2–§5.3 (`aios.<tenant>.<dominio>.<entidade>.<acao>`). `<dominio>` deste
> módulo = `api`. Entrega **at-least-once** via JetStream; consumidores
> **DEVEM** deduplicar por `event.id`.
>
> **Convenção especializada deste módulo**: eventos de **ciclo de vida do
> registro de contrato** (rota, versão, código de erro, schema de evento) são
> **metadados de plataforma**, não eventos de um tenant específico — eles
> usam o tenant reservado **`_platform`** (`aios._platform.api.*`),
> ratificado em ADR-0048. Eventos que refletem **atividade por tenant** (rate
> limit excedido, autenticação rejeitada, violação de contrato) usam o
> `<tenant>` real do chamador.

## 1. Eventos emitidos (produtor: 004-API)

| Subject | `type` | Quando | Stream (JetStream) |
|---------|--------|--------|---------------------|
| `aios._platform.api.route.registered` | `aios.api.route.registered` | Nova `RouteDefinition` criada (UC-009). | `API_REGISTRY` (durável) |
| `aios._platform.api.route.updated` | `aios.api.route.updated` | Rota atualizada. | `API_REGISTRY` |
| `aios._platform.api.route.disabled` | `aios.api.route.disabled` | Rota desativada. | `API_REGISTRY` |
| `aios._platform.api.version.published` | `aios.api.version.published` | `ApiVersion` → `Published` (T-02, UC-010). | `API_REGISTRY` |
| `aios._platform.api.version.deprecated` | `aios.api.version.deprecated` | `ApiVersion` → `Deprecated` (T-04, UC-011). | `API_REGISTRY` |
| `aios._platform.api.version.retired` | `aios.api.version.retired` | `ApiVersion` → `Retired` (T-05, UC-012). | `API_REGISTRY` |
| `aios._platform.api.errorcode.registered` | `aios.api.errorcode.registered` | Novo código `AIOS-<DOMINIO>-<NNNN>` registrado. | `API_REGISTRY` |
| `aios._platform.api.eventschema.registered` | `aios.api.eventschema.registered` | Novo `dataschema` registrado. | `API_REGISTRY` |
| `aios.<tenant>.api.request.rejected` | `aios.api.request.rejected` | AuthN/AuthZ/rate-limit/schema rejeitou (amostrado, sinal de segurança — UC-002, UC-003, UC-004, UC-006). | `API_SECURITY` |
| `aios.<tenant>.api.contract.violation` | `aios.api.contract.violation` | Payload não conforme ao schema registrado (UC-006). | `API_SECURITY` |
| `aios.<tenant>.api.ratelimit.breached` | `aios.api.ratelimit.breached` | Limiar sustentado de rate-limit excedido (sinal de saturação/abuso — UC-004). | `API_SECURITY` |

## 2. Eventos consumidos

| Subject assinado | Produtor | Ação do Gateway |
|-------------------|----------|------------------|
| `aios.<tenant>.policy.decision.updated` | `022-Policy` | Invalida cache de decisões do `RouteAuthorizer`. |
| `aios._platform.security.jwks.rotated` | `021-Security` | Recarrega `JwksCache` do `AuthNFilter`. |
| `aios.<tenant>.security.token.revoked` | `021-Security` | Invalida sessões/tokens ativos correspondentes na borda. |
| `aios._platform.cluster.capacity.changed` | `027-Cluster` | Atualiza saúde de upstream para `CircuitBreakerManager`. |
| `aios._platform.<module>.contract.published` | Cada módulo upstream | Dispara `OpenApiAggregator` a re-agregar o contrato unificado (UC-015). |

## 3. Schema de payload (`data`) por evento

### 3.1 `aios.api.route.registered` / `.updated` / `.disabled`

```json
{
  "specversion": "1.0",
  "id": "01J9Z8Q8R0VTEEVT0000000001",
  "source": "urn:aios:_platform:service:api-gateway",
  "type": "aios.api.route.registered",
  "subject": "urn:aios:_platform:route:01J9Z8QAR0VTE0000000000001",
  "time": "2026-07-21T12:05:00.000Z",
  "tenant": "_platform",
  "traceparent": "00-...-01",
  "datacontenttype": "application/json",
  "dataschema": "aios://schemas/api.route.registered/1",
  "data": {
    "routeId": "01J9Z8QAR0VTE0000000000001",
    "moduleOwner": "006-Kernel",
    "pathPattern": "/v1/kernel/agents/{urn}",
    "methods": ["GET", "DELETE"],
    "protocol": "rest",
    "apiMajor": 1,
    "status": "active"
  }
}
```

### 3.2 `aios.api.version.published` / `.deprecated` / `.retired`

```json
{
  "specversion": "1.0",
  "id": "01J9Z8QBVEREVT000000000001",
  "source": "urn:aios:_platform:service:api-gateway",
  "type": "aios.api.version.published",
  "subject": "urn:aios:_platform:apiversion:006-Kernel:2",
  "time": "2026-07-21T12:05:00.000Z",
  "tenant": "_platform",
  "traceparent": "00-...-01",
  "datacontenttype": "application/json",
  "dataschema": "aios://schemas/api.version.published/1",
  "data": {
    "moduleOwner": "006-Kernel",
    "major": 2,
    "state": "Published",
    "contractRef": "minio://contracts/006-kernel/v2/openapi.json"
  }
}
```

### 3.3 `aios.api.request.rejected` (sinal de segurança, por tenant)

```json
{
  "specversion": "1.0",
  "id": "01J9Z8QCREJEVT000000000001",
  "source": "urn:aios:acme:service:api-gateway",
  "type": "aios.api.request.rejected",
  "subject": "urn:aios:acme:route:01J9Z8QAR0VTE0000000000001",
  "time": "2026-07-21T12:06:00.000Z",
  "tenant": "acme",
  "traceparent": "00-...-01",
  "datacontenttype": "application/json",
  "dataschema": "aios://schemas/api.request.rejected/1",
  "data": {
    "reason": "AIOS-API-0002",
    "path": "/v1/kernel/agents",
    "method": "POST",
    "clientIp": "203.0.113.7",
    "sampled": true
  }
}
```

## 4. Semântica de entrega e ordenação

- **Entrega**: at-least-once via JetStream (`API_REGISTRY`, `API_SECURITY`),
  conforme RFC-0001 §5.2. Consumidores **DEVEM** deduplicar por `event.id`.
- **Ordenação**: eventos de `api.version.*` para o mesmo `(module_owner,
  major)` **DEVEM** ser publicados na ordem das transições da FSM (T-01 até
  T-06 conforme aplicável) — garantido pela publicação via Outbox
  transacional imediatamente após o commit da transição (invariante I4 de
  `./StateMachine.md`).
- **Atomicidade**: toda mutação de registro (rota/versão/erro/schema) usa o
  padrão **Outbox transacional** (`api.outbox`, ver `./Database.md`): o
  evento é gravado na mesma transação da mutação e um *relay* assíncrono
  publica no JetStream, garantindo `NFR-005` (perda de evento de registro =
  0 sob *crash*).
- **Versionamento de schema**: cada `dataschema` referenciado acima é, por
  sua vez, uma entrada em `api.event_schema` (RFC-0001 §8) — evolução de
  payload segue a mesma regra de aditividade da RFC-0001 §5.7 (consumidores
  DEVEM tolerar campos desconhecidos).
- **Amostragem de sinais de segurança**: `api.request.rejected` PODE ser
  amostrado sob alto volume de rejeições (ex.: ataque de força bruta) para
  não sobrecarregar o `API_SECURITY` stream; a decisão de amostragem é
  configurável e documentada em `./Configuration.md`. `api.ratelimit.breached`
  e `api.contract.violation` **NÃO DEVEM** ser amostrados (sinal de baixo
  volume e alto valor forense).

## 5. Consistência com o restante do módulo

Nomes de subject, `type` e `dataschema` são **idênticos** aos usados em
`./StateMachine.md` (transições) e `./API.md` (operações que os disparam).

*Fim de `Events.md`.*
