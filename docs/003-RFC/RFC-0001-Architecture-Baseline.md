---
Documento: RFC-0001 — AIOS Architecture Baseline & Core Contracts
Módulo: 003-RFC
Status: Accepted
Versão: 1.0
Última atualização: 2026-07-20
Autores: Arquitetura-Chefe, Steering Committee
Módulos afetados: 000, 001, 004, 006, 009, 020, 021, 022, 024, 025
---

# RFC-0001: AIOS Architecture Baseline & Core Contracts

## 1. Abstract

Esta RFC estabelece a linha de base normativa da arquitetura do AIOS e os
contratos centrais compartilhados por todos os módulos: identidade de recursos,
envelope de eventos, envelope de erro de API, cabeçalhos de correlação, semântica
de idempotência, e o registro de subjects do barramento. Ela consolida as decisões
ADR-0001..ADR-0010 em requisitos verificáveis (RFC 2119). Módulos subsequentes
DEVEM estar em conformidade com esta RFC.

## 2. Status desta RFC e Terminologia

As palavras-chave "DEVE", "NÃO DEVE", "OBRIGATÓRIO", "DEVERIA", "NÃO DEVERIA",
"PODE" e "OPCIONAL" devem ser interpretadas conforme RFC 2119 e RFC 8174.

## 3. Motivação

Com dezenas de módulos e duas plataformas de runtime (.NET/Python), a ausência de
contratos comuns levaria a incompatibilidades de identificadores, eventos, erros e
correlação de telemetria. Esta RFC fixa esses contratos uma única vez, para que
todo módulo os reutilize sem redefinição — garantindo consistência, auditabilidade
e evolução compatível.

## 4. Terminologia

Reutiliza `../040-Glossary/Glossary.md`. Termos-chave: **Tenant**, **Agent**,
**Task**, **ACB**, **Control Plane**, **Data Plane**, **Subject**, **Idempotency**.

## 5. Especificação (normativa)

### 5.1 Identidade de recursos (URN)

Todo recurso gerenciado DEVE possuir um identificador estável no formato URN:

```
urn:aios:<tenant>:<tipo>:<id>
  <tenant> = slug do tenant (a-z0-9-), OBRIGATÓRIO
  <tipo>   = agent | task | memory | tool | model | plan | workflow | policy | event
  <id>     = ULID (Universally Unique Lexicographically Sortable Identifier)
```

Exemplos:
```
urn:aios:acme:agent:01J9Z8Q6H7K2M4N6P8R0S2T4V6
urn:aios:acme:task:01J9Z8Q7B0C2D4E6F8G0H2J4K6
```

- IDs DEVEM ser ULID (ordenáveis por tempo, 128 bits).
- `tenant` DEVE estar presente em todo recurso multi-tenant (isolamento).
- IDs NÃO DEVEM ser reutilizados após exclusão.

### 5.2 Envelope de Evento (NATS)

Todo evento publicado no barramento DEVE usar este envelope JSON (CloudEvents-compatível):

```json
{
  "specversion": "1.0",
  "id": "01J9Z8Q8...",               // ULID único do evento
  "source": "urn:aios:acme:service:scheduler",
  "type": "aios.task.execution.completed",
  "subject": "urn:aios:acme:task:01J9...",
  "time": "2026-07-20T12:34:56.789Z", // RFC 3339, UTC
  "tenant": "acme",
  "traceparent": "00-<trace_id>-<span_id>-01",
  "datacontenttype": "application/json",
  "dataschema": "aios://schemas/task.execution.completed/1",
  "data": { /* payload específico do evento, versionado */ }
}
```

- `type` DEVE seguir `aios.<dominio>.<entidade>.<acao>`.
- `dataschema` DEVE referenciar uma versão de schema registrada.
- `traceparent` DEVE propagar o contexto W3C Trace Context.
- Produtores DEVEM garantir at-least-once; consumidores DEVEM ser idempotentes (§5.5).

### 5.3 Convenção de Subjects NATS

```
aios.<tenant>.<dominio>.<entidade>.<acao>

  <dominio>  = agent | task | memory | context | model | tool | knowledge
             | plan | workflow | policy | audit | cost | cluster
  <entidade> = lifecycle | execution | item | decision | ...
  <acao>     = created | updated | deleted | started | completed | failed | ...
```

Exemplos normativos:
```
aios.acme.agent.lifecycle.spawned
aios.acme.agent.lifecycle.suspended
aios.acme.task.execution.completed
aios.acme.memory.item.consolidated
aios.acme.policy.decision.denied
```

- Wildcards de assinatura seguem NATS (`aios.acme.agent.>`).
- O namespace `<tenant>` DEVE isolar tráfego entre tenants (contas NATS).

### 5.4 Envelope de Erro de API (REST e gRPC)

Toda API DEVE retornar erros neste formato (RFC 7807-compatível para REST):

```json
{
  "type": "https://docs.aios/errors/quota-exceeded",
  "title": "Quota Exceeded",
  "status": 429,
  "code": "AIOS-QUOTA-0001",
  "detail": "Token budget exceeded for tenant 'acme'.",
  "instance": "urn:aios:acme:task:01J9...",
  "traceId": "<trace_id>",
  "timestamp": "2026-07-20T12:34:56.789Z",
  "retriable": true,
  "retryAfterMs": 1500
}
```

- `code` DEVE seguir `AIOS-<DOMINIO>-<NNNN>` e ser estável (catálogo em `004-API`).
- gRPC DEVE mapear `status` → `google.rpc.Status` + `ErrorInfo` com o mesmo `code`.
- `retriable` DEVE indicar se o cliente PODE repetir.

### 5.5 Idempotência

- Toda operação de mutação exposta externamente DEVE aceitar o cabeçalho
  `Idempotency-Key` (ULID/UUID fornecido pelo cliente).
- O servidor DEVE persistir o resultado por chave por, no mínimo, 24h e retornar o
  mesmo resultado em repetições.
- Consumidores de eventos DEVEM deduplicar por `event.id`.

### 5.6 Cabeçalhos de correlação (obrigatórios)

| Cabeçalho | Descrição |
|-----------|-----------|
| `traceparent` / `tracestate` | W3C Trace Context (OTel). OBRIGATÓRIO. |
| `X-AIOS-Tenant` | Tenant do chamador. OBRIGATÓRIO em APIs multi-tenant. |
| `X-AIOS-Agent` | URN do agente, quando aplicável. |
| `Idempotency-Key` | Ver §5.5. OBRIGATÓRIO em mutações. |
| `X-AIOS-Api-Version` | Versão de API solicitada (§9). |

### 5.7 Versionamento de API

- APIs REST DEVEM ser versionadas por caminho (`/v1/...`) e header `X-AIOS-Api-Version`.
- Mudanças incompatíveis DEVEM incrementar a versão major e coexistir por ≥ 2 majors (V-R9).
- gRPC DEVE usar pacotes versionados (`aios.kernel.v1`).
- Eventos DEVEM versionar `dataschema`; consumidores DEVEM tolerar campos desconhecidos (evolução aditiva).

### 5.8 Requisitos transversais (derivados de ADR-0008/0010)

- Toda ação privilegiada DEVE passar por um PEP e ser autorizada pelo PDP (022) — *default deny*.
- Toda ação privilegiada e toda decisão de agente DEVEM emitir registro de auditoria imutável (025).
- Todo serviço DEVE emitir traces/metrics/logs OTel com os campos de correlação (§5.6).

## 6. Considerações de Segurança

- Comunicação interna entre serviços DEVE usar mTLS (021).
- Tokens de acesso DEVEM ser validados no Gateway (OAuth2/OIDC) e propagados como claims assinadas.
- `tenant` em URNs e subjects é fronteira de isolamento; serviços NÃO DEVEM aceitar `tenant` divergente do contexto autenticado.
- Envelopes de erro NÃO DEVEM vazar dados sensíveis em `detail`.

## 7. Considerações de Privacidade (LGPD/GDPR)

- Payloads de eventos e logs DEVEM aplicar minimização de dados; PII DEVE ser redigida ou tokenizada.
- Itens de memória com dados pessoais DEVEM carregar metadados de base legal e retenção (010/025).
- O direito ao esquecimento DEVE ser suportado por operação de expurgo rastreável.

## 8. Registros do AIOS (IANA-like)

Esta RFC cria os seguintes registros, mantidos em `../004-API/`:

| Registro | Conteúdo | Doc |
|----------|----------|-----|
| Tipos de recurso (URN `<tipo>`) | agent, task, memory, tool, model, plan, workflow, policy, event | 004-API/Resources.md |
| Domínios de subject | agent, task, memory, ... | 004-API/Events.md |
| Códigos de erro | `AIOS-<DOMINIO>-<NNNN>` | 004-API/Errors.md |
| Versões de schema de evento | `dataschema` → versão | 004-API/Events.md |

Novos valores DEVEM ser registrados via PR + RFC/ADR de módulo.

## 9. Compatibilidade e Versionamento

Ver §5.7. Esta RFC é a v1 da baseline. Alterações incompatíveis exigem nova RFC
que *obsolete* esta (`Obsoleted by RFC-XXXX`), mantendo período de coexistência.

## 10. Referências

**Normativas:** RFC 2119, RFC 8174, RFC 3339, RFC 7807, W3C Trace Context;
ADR-0001..ADR-0010; `../001-Architecture/Architecture.md`.
**Informativas:** CloudEvents 1.0; `../000-Vision/Vision.md`; `../040-Glossary/Glossary.md`.

## 11. Apêndices

### A. Exemplo — publicar evento de conclusão de tarefa (Python runtime)

```python
event = {
  "specversion": "1.0",
  "id": ulid(),
  "source": "urn:aios:acme:service:runtime",
  "type": "aios.task.execution.completed",
  "subject": task_urn,
  "time": now_rfc3339(),
  "tenant": "acme",
  "traceparent": current_traceparent(),
  "datacontenttype": "application/json",
  "dataschema": "aios://schemas/task.execution.completed/1",
  "data": {"taskId": task_urn, "status": "SUCCEEDED", "costUsd": 0.0123, "tokens": 4210}
}
await nats.publish("aios.acme.task.execution.completed", json.dumps(event).encode())
```

### B. Exemplo — resposta de erro REST (429)

```
HTTP/1.1 429 Too Many Requests
Content-Type: application/problem+json
Retry-After: 2

{ "type":"https://docs.aios/errors/quota-exceeded","title":"Quota Exceeded",
  "status":429,"code":"AIOS-QUOTA-0001","detail":"Token budget exceeded.",
  "traceId":"4bf92f3577b34da6a3ce929d0e0e4736","retriable":true,"retryAfterMs":1500 }
```
