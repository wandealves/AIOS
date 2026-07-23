---
Documento: Events
Módulo: 020-Communication
Status: Draft
Versão: 0.1
Última atualização: 2026-07-22
Responsável (RACI-A): Arquiteto do Módulo 020-Communication
ADRs relacionados: ADR-0004, ADR-0200, ADR-0202, ADR-0204, ADR-0205
RFCs relacionados: RFC-0001, RFC-0200
Depende de: _DESIGN_BRIEF.md §6, 004-API (registro de domínios e dataschema), 025-Audit
---

# 020-Communication — Catálogo de Eventos NATS

Este módulo tem uma posição singular no catálogo de eventos do AIOS: ele **transporta
os eventos de todos os outros módulos** e emite poucos próprios — apenas os que
descrevem o estado do próprio barramento.

O envelope (CloudEvents) e a convenção de subjects são normativos na
`../003-RFC/RFC-0001-Architecture-Baseline.md` §5.2–§5.3 e **não são redefinidos aqui**.
`<dominio>` = **`comm`**, valor novo a ser registrado no registro de domínios mantido
por `../004-API/` conforme RFC-0001 §8 (ratificação em **ADR-0200**).

---

## 1. O Namespace que este Módulo Governa

```
   aios . <tenant> . <dominio> . <entidade> . <acao>
    │        │          │           │           │
    │        │          │           │           └─ created | updated | deleted | …
    │        │          │           └───────────── lifecycle | execution | item | …
    │        │          └───────────────────────── agent | task | memory | context |
    │        │                                     model | tool | knowledge | plan |
    │        │                                     workflow | policy | audit | cost |
    │        │                                     cluster | api | database | comm
    │        └──────────────────────────────────── slug do tenant, ou `_platform`
    └───────────────────────────────────────────── prefixo fixo
```

**Regras de governança aplicadas pelo `SubjectRegistry`** (todas verificáveis):

1. Cada trio (`domain`, `entity`, `action`) é **único** e tem **um** `producer_module`.
2. Um subject identifica um **tipo** de evento, nunca uma **instância** — é proibido
   `aios.acme.agent.01J9Z8Q6….status`. Subjects por instância explodem a cardinalidade
   de roteamento e tornam o catálogo inútil.
3. Todo subject aponta para um `dataschema` registrado em `../004-API/`.
4. Todo subject pertence a exatamente **um** stream (streams não se sobrepõem).
5. Domínios novos (`database`, `comm`) exigem registro via PR + ADR (RFC-0001 §8).

---

## 2. Eventos Emitidos

| Subject | `type` | Quando | Stream |
|---------|--------|--------|--------|
| `aios._platform.comm.subject.registered` | `aios.comm.subject.registered` | Novo subject registrado (UC-001). | `COMM_PLATFORM` |
| `aios._platform.comm.subject.deprecated` | `aios.comm.subject.deprecated` | Subject entrou em depreciação. | `COMM_PLATFORM` |
| `aios._platform.comm.stream.created` | `aios.comm.stream.created` | Stream materializado no JetStream. | `COMM_PLATFORM` |
| `aios._platform.comm.stream.retired` | `aios.comm.stream.retired` | Stream aposentado após drenagem. | `COMM_PLATFORM` |
| `aios._platform.comm.account.provisioned` | `aios.comm.account.provisioned` | Conta NATS criada para um tenant. | `COMM_PLATFORM` |
| `aios._platform.comm.cluster.degraded` | `aios.comm.cluster.degraded` | Perda de nó ou de quórum JetStream. | `COMM_HEALTH` |
| `aios.<tenant>.comm.a2a.established` | `aios.comm.a2a.established` | Sessão A2A → `Established` (T-04). | `COMM_A2A` |
| `aios.<tenant>.comm.a2a.rejected` | `aios.comm.a2a.rejected` | Sessão → `Rejected` (T-05). | `COMM_A2A` |
| `aios.<tenant>.comm.a2a.closed` | `aios.comm.a2a.closed` | Sessão → `Closed` (T-09). | `COMM_A2A` |
| `aios.<tenant>.comm.a2a.failed` | `aios.comm.a2a.failed` | Sessão → `Failed` (T-10). | `COMM_A2A` |
| `aios.<tenant>.comm.delivery.deadlettered` | `aios.comm.delivery.deadlettered` | Mensagem esgotou `max_deliver`. | `COMM_HEALTH` |
| `aios.<tenant>.comm.delivery.replayed` | `aios.comm.delivery.replayed` | Replay autorizado de DLQ ou stream. | `COMM_HEALTH` |
| `aios.<tenant>.comm.consumer.stalled` | `aios.comm.consumer.stalled` | Consumidor parado ou lag acima do limiar. | `COMM_HEALTH` |
| `aios.<tenant>.comm.quota.throttled` | `aios.comm.quota.throttled` | Cota de tráfego aplicada a um produtor. | `COMM_HEALTH` |

---

## 3. Streams JetStream do Módulo

| Stream | Subjects | Política | `max_age` | Réplicas |
|--------|----------|----------|-----------|----------|
| `COMM_PLATFORM` | `aios._platform.comm.subject.>`, `…stream.>`, `…account.>` | `limits` | 1 ano | 3 |
| `COMM_HEALTH` | `aios._platform.comm.cluster.>`, `aios.*.comm.delivery.>`, `aios.*.comm.consumer.>`, `aios.*.comm.quota.>` | `limits` | 30 dias | 3 |
| `COMM_A2A` | `aios.*.comm.a2a.>` | `limits` | 90 dias | 3 |

O catálogo completo de streams do AIOS (incluindo os de outros módulos, que este
módulo hospeda) está em `./Database.md` §3.1.

---

## 4. Schemas de Payload

Todo evento usa o envelope da RFC-0001 §5.2. Abaixo apenas o `data`, com o
`dataschema` correspondente.

### 4.1 `aios.comm.a2a.established`

`dataschema`: `aios://schemas/comm.a2a.established/1`

```json
{
  "specversion": "1.0",
  "id": "01J9ZC9V2X4Z6B8D0F2H4K6M8P",
  "source": "urn:aios:acme:service:comm",
  "type": "aios.comm.a2a.established",
  "subject": "urn:aios:acme:a2asession:01J9ZC8U1W3Y5A7C9E1G3J5L7N",
  "time": "2026-07-22T15:10:00.412Z",
  "tenant": "acme",
  "traceparent": "00-4bf92f3577b34da6a3ce929d0e0e4736-00f067aa0ba902b7-01",
  "datacontenttype": "application/json",
  "dataschema": "aios://schemas/comm.a2a.established/1",
  "data": {
    "initiatorUrn": "urn:aios:acme:agent:01J9Z8Q6H7K2M4N6P8R0S2T4V6",
    "peerUrn": "urn:aios:acme:agent:01J9Z8Q7B0C2D4E6F8G0H2J4K6",
    "peerKind": "internal",
    "capabilities": ["task.delegate", "memory.share"],
    "channelSubject": "aios.acme.a2a.session.01J9ZC8U1W3Y5A7C9E1G3J5L7N",
    "handshakeMs": 118
  }
}
```

### 4.2 `aios.comm.delivery.deadlettered`

`dataschema`: `aios://schemas/comm.delivery.deadlettered/1`

```json
{
  "data": {
    "dlqId": "01J9ZCAW3Y5A7C9E1G3J5L7N9Q",
    "originalSubject": "aios.acme.memory.item.consolidated",
    "streamName": "MEMORY_EVENTS",
    "consumerName": "learning-consolidation",
    "deliveryCount": 5,
    "lastError": "ValidationException: campo 'layer' ausente",
    "firstDeliveredAt": "2026-07-22T15:02:10Z",
    "quarantinedAt": "2026-07-22T15:05:41Z"
  }
}
```

> **Minimização (RFC-0001 §7):** `lastError` carrega a **classe** do erro e a mensagem
> técnica — nunca o payload nem dados pessoais. O envelope completo fica em
> `comm.dlq_entry`, sob RLS e com retenção de 30 dias.

### 4.3 `aios.comm.consumer.stalled`

`dataschema`: `aios://schemas/comm.consumer.stalled/1`

```json
{
  "data": {
    "streamName": "KERNEL_LIFECYCLE",
    "consumerName": "learning-lifecycle",
    "consumerModule": "023-Learning",
    "numPending": 48210,
    "numAckPending": 1000,
    "maxAckPending": 1000,
    "lastAckAt": "2026-07-22T14:58:03Z",
    "stalledForMs": 184000
  }
}
```

### 4.4 `aios.comm.quota.throttled`

`dataschema`: `aios://schemas/comm.quota.throttled/1`

```json
{
  "data": {
    "scope": "tenant",
    "subjectUrn": "urn:aios:acme:tenant:acme",
    "limitKind": "msgs_per_sec",
    "limitValue": 50000,
    "observedValue": 68420,
    "windowSeconds": 1,
    "action": "rejected",
    "errorCode": "AIOS-BUS-0007"
  }
}
```

### 4.5 `aios.comm.cluster.degraded`

`dataschema`: `aios://schemas/comm.cluster.degraded/1`

```json
{
  "data": {
    "nodesTotal": 3,
    "nodesHealthy": 2,
    "jetstreamQuorum": true,
    "affectedStreams": [],
    "degradedSince": "2026-07-22T15:20:00Z",
    "severity": "P2"
  }
}
```

`jetstreamQuorum: true` com um nó fora é degradação **sem** impacto de entrega (R=3).
Se o campo virar `false`, a severidade sobe para P1 e a publicação durável passa a
falhar com `AIOS-BUS-0011`.

---

## 5. Eventos Consumidos

| Subject assinado | Produtor | Ação do módulo | Consumidor durável |
|-------------------|----------|----------------|--------------------|
| `aios.<tenant>.policy.bundle.updated` | `../022-Policy/` | Invalida o cache do `PolicyClient` e reavalia sessões A2A ativas (pode disparar T-06). | `comm-policy-cache` |
| `aios.<tenant>.security.token.revoked` | `../021-Security/` | Revoga credencial de conta/par; encerra sessões afetadas (T-10). | `comm-revocation` |
| `aios._platform.cluster.topology.changed` | `../027-Cluster/` | Reconfigura leaf nodes/gateways via `BridgeConnector`. | `comm-topology` |
| `aios.<tenant>.cost.budget.updated` | `../026-Cost-Optimizer/` | Atualiza cotas de tráfego do `FlowController`. | `comm-quota` |
| `aios.<tenant>.agent.lifecycle.terminated` | `../006-Kernel/` | Encerra sessões A2A e libera assinaturas do agente extinto (FR-020). | `comm-agent-cleanup` |
| `aios._platform.api.eventschema.registered` | `../004-API/` | Atualiza o cache de `dataschema` do `EnvelopeValidator`. | `comm-schema-cache` |

Todos os consumidores **DEVEM** ser idempotentes: reprocessar um `event.id` já visto é
operação nula.

---

## 6. Semântica de Entrega (o contrato que o módulo oferece aos demais)

| Aspecto | Garantia |
|---------|----------|
| Entrega | **At-least-once** via JetStream. *Exactly-once efetivo* pela combinação Outbox no produtor + dedupe por `event.id` na janela do stream + dedupe no consumidor (RFC-0001 §5.5). |
| Ordenação | Garantida **por stream**. Não há ordem global entre streams; quem depende de ordem entre domínios reconcilia por `time`/`event.id`. |
| Reentrega | `ack_wait` (30 s) + backoff `[1s, 5s, 30s, 2min]` até `max_deliver` (5). |
| DLQ | Esgotadas as tentativas, a mensagem é **quarentenada e inspecionável** — nunca descartada em silêncio. |
| Replay | Por tempo, sequência ou entrada de DLQ, sempre em consumidor isolado e sob capability. |
| Versionamento | `dataschema` versionado; evolução **aditiva**; consumidores toleram campos desconhecidos (RFC-0001 §5.7). |
| Retenção | Explícita por stream (`max_age`). O barramento **não é arquivo**. |
| Privacidade | Payload segue minimização (RFC-0001 §7): URNs opacos e referências, não conteúdo sensível. |

---

## 7. Exemplo — Produtor e Consumidor

**Produzir** (o produtor publica; o barramento valida):

```python
event = {
  "specversion": "1.0",
  "id": ulid(),
  "source": "urn:aios:acme:service:kernel",
  "type": "aios.agent.lifecycle.spawned",     # DEVE casar com o subject
  "subject": agent_urn,
  "time": now_rfc3339(),
  "tenant": "acme",
  "traceparent": current_traceparent(),        # obrigatório (RFC-0001 §5.6)
  "datacontenttype": "application/json",
  "dataschema": "aios://schemas/agent.lifecycle.spawned/1",
  "data": {"agentUrn": agent_urn, "parentUrn": parent_urn, "priority": 5}
}
ack = await js.publish("aios.acme.agent.lifecycle.spawned",
                       json.dumps(event).encode(),
                       headers={"Nats-Msg-Id": event["id"]})  # dedupe por event.id
```

**Consumir** (dedupe obrigatório, ack explícito):

```python
sub = await js.pull_subscribe(
    "aios.acme.agent.lifecycle.spawned",
    durable="audit-lifecycle",
)

for msg in await sub.fetch(batch=100, timeout=5):
    event = json.loads(msg.data)
    if seen(event["id"]):          # at-least-once ⇒ dedupe é do consumidor
        await msg.ack(); continue
    try:
        handle(event)
        mark_seen(event["id"])
        await msg.ack()
    except TransientError:
        await msg.nak(delay=5)     # reentrega antecipada e controlada
    except PermanentError:
        await msg.term()           # vai direto à DLQ, sem gastar 5 tentativas
```

`term()` em erro permanente é uma boa prática subestimada: gastar cinco tentativas em
uma mensagem que jamais será processada só atrasa o backlog.

---

## 8. Referências

- Brief: `./_DESIGN_BRIEF.md` §6
- Contratos de evento: `../003-RFC/RFC-0001-Architecture-Baseline.md` §5.2–§5.3
- Streams e modelo físico: `./Database.md` · API de controle: `./API.md`
- Registro de domínios e schemas: `../004-API/`
