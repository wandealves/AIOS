---
Documento: Examples
Módulo: 020-Communication
Status: Draft
Versão: 0.1
Última atualização: 2026-07-22
Responsável (RACI-A): Arquiteto do Módulo 020-Communication
ADRs relacionados: ADR-0200, ADR-0202, ADR-0205, ADR-0206, ADR-0207
RFCs relacionados: RFC-0001, RFC-0200, RFC-0201
Depende de: API.md, Events.md, Configuration.md, 030-CLI, 031-SDK
---

# 020-Communication — Exemplos

Exemplos executáveis, do mais simples ao avançado. Endpoints e códigos de erro são os de
`./API.md`; o contrato de mensagem é o de `./Events.md`.

---

## 1. Hello World — publicar e consumir um evento

**Passo 1 — registrar o subject** (uma vez, por migração/pipeline):

```bash
aios comm subject register \
  --domain agent --entity lifecycle --action spawned \
  --producer 006-Kernel \
  --dataschema aios://schemas/agent.lifecycle.spawned/1 \
  --stream KERNEL_LIFECYCLE --traffic-class control
# → urn:aios:_platform:subjectdef:01J9ZC5R8T0V2X4Z6B8D0F2H4K
```

**Passo 2 — publicar** (Python, dentro do relay de Outbox):

```python
event = {
  "specversion": "1.0",
  "id": ulid(),                                   # dedupe key
  "source": "urn:aios:acme:service:kernel",
  "type": "aios.agent.lifecycle.spawned",         # DEVE casar com o subject
  "subject": agent_urn,
  "time": now_rfc3339(),
  "tenant": "acme",
  "traceparent": current_traceparent(),
  "datacontenttype": "application/json",
  "dataschema": "aios://schemas/agent.lifecycle.spawned/1",
  "data": {"agentUrn": agent_urn, "parentUrn": parent_urn, "priority": 5}
}

ack = await js.publish(
    "aios.acme.agent.lifecycle.spawned",
    json.dumps(event).encode(),
    headers={"Nats-Msg-Id": event["id"]},         # dedupe por event.id
)
print(ack.stream, ack.seq)   # KERNEL_LIFECYCLE 8412
```

**Passo 3 — consumir** (com ack explícito e dedupe):

```python
sub = await js.pull_subscribe("aios.acme.agent.lifecycle.spawned",
                              durable="audit-lifecycle")

for msg in await sub.fetch(batch=100, timeout=5):
    event = json.loads(msg.data)
    if seen(event["id"]):        # at-least-once ⇒ dedupe é do consumidor
        await msg.ack(); continue
    handle(event)
    mark_seen(event["id"])
    await msg.ack()
```

---

## 2. Declarar stream e consumidor

```bash
# stream (idempotente)
aios comm stream put KERNEL_LIFECYCLE \
  --subjects 'aios.*.agent.lifecycle.>' \
  --retention limits --max-age 30d --max-bytes 50GiB \
  --replicas 3 --discard old --dedupe-window 2m \
  --owner 006-Kernel

# consumidor durável
aios comm consumer put KERNEL_LIFECYCLE audit-lifecycle \
  --module 025-Audit \
  --ack-policy explicit --ack-wait 30s \
  --max-deliver 5 --backoff 1s,5s,30s,2m \
  --max-ack-pending 1000 \
  --dlq-subject aios.acme.comm.delivery.deadlettered
```

Repetir qualquer um dos comandos com os mesmos parâmetros é operação nula. Alterar
`--replicas` dispara migração espelhada, sem perda de backlog (ADR-0203).

---

## 3. O que acontece quando o subject não está registrado

```python
await js.publish("aios.acme.tool.usage.spike", payload)
```

```json
{
  "code": "AIOS-BUS-0004",
  "title": "Subject Not Registered",
  "status": 422,
  "detail": "aios.acme.tool.usage.spike não consta em comm.subject_registry.",
  "retriable": false
}
```

A mensagem **não se perde**: ela permanece no Outbox do produtor. Registre o subject
(§1, passo 1) e o relay publica na varredura seguinte.

---

## 4. Subject por instância — o erro mais caro

```bash
# ❌ ERRADO — um subject por agente
aios comm subject register --domain agent --entity 01J9Z8Q6H7K2M4N6P8R0S2T4V6 --action status
# → 422 AIOS-BUS-0004: componente deve casar ^[a-z][a-z0-9]*$

# ✅ CERTO — um subject por tipo; a instância vai no envelope
aios comm subject register --domain agent --entity status --action changed ...
```

```python
event["subject"] = "urn:aios:acme:agent:01J9Z8Q6H7K2M4N6P8R0S2T4V6"  # a instância aqui
await js.publish("aios.acme.agent.status.changed", ...)               # o tipo aqui
```

Com 10⁶ agentes, a primeira forma criaria 10⁶ subjects — inviabilizando roteamento,
observabilidade e o próprio catálogo.

---

## 5. Difusão para um *Agent Group*

```bash
# declarar o canal
aios comm group channel put squad-7 \
  --subject-prefix aios.acme.group.squad-7 \
  --fanout-limit 256 --scope group --rate-limit 100
```

```csharp
// UMA publicação; o broker faz o fan-out para todos os membros
await bus.PublishAsync(
    subject: "aios.acme.group.squad-7.task.assigned",
    envelope: CloudEvent.Create(
        type: "aios.task.assignment.created",
        tenant: "acme",
        data: new { taskUrn, deadline }));
```

```csharp
// ❌ ANTIPADRÃO — iterar destinatários na aplicação
foreach (var agent in squad.Members)                     // 1.000 publicações
    await bus.PublishAsync($"aios.acme.agent.{agent.Id}.inbox", envelope);
```

O segundo bloco satura o cliente, cria 1.000 subjects por instância e dispara o alerta
`BusFanoutAmplification` (`./Monitoring.md` §3).

---

## 6. Request/reply

```csharp
// Solicitante (006-Kernel consultando o PDP)
var decision = await bus.RequestAsync<DecisionRequest, Decision>(
    subject: "aios.acme.policy.decision.request",
    request: new DecisionRequest(subject: agentUrn, action: "memory:write", resource: memUrn),
    timeout: TimeSpan.FromSeconds(5));      // bus.request.timeout_ms

// Respondente (022-Policy)
await bus.RespondAsync<DecisionRequest, Decision>(
    subject: "aios.acme.policy.decision.request",
    handler: async req => await pdp.EvaluateAsync(req));   // traceparent preservado
```

Sem resposta no prazo: `AIOS-BUS-0012` (retriable). Repetir **apenas** se a operação for
idempotente — uma consulta ao PDP é; um comando de mutação pode não ser.

---

## 7. Sessão A2A entre agentes

```python
# Agente A abre a sessão
session = await comm.a2a.open(
    peer_urn="urn:aios:acme:agent:01J9Z8Q7B0C2D4E6F8G0H2J4K6",
    peer_kind="internal",
    capabilities=["task.delegate", "memory.share"],
    idempotency_key=ulid(),
)
print(session.state, session.channel_subject)
# Established  aios.acme.a2a.session.01J9ZC8U1W3Y5A7C9E1G3J5L7N

# Troca de mensagens no canal privado
await comm.a2a.send(session.urn, {"kind": "delegate", "taskUrn": task_urn})

# Encerramento explícito (ou automático por ociosidade em 300 s)
await comm.a2a.close(session.urn, reason="completed")
```

Tentativa de ampliar capacidades em sessão viva:

```python
await comm.a2a.add_capability(session.urn, "tool.invoke")
# → 422 AIOS-A2A-0007: capacidades imutáveis após Established (invariante I4).
#   Abra uma NOVA sessão com o conjunto desejado.
```

---

## 8. Diagnóstico de consumidor lento

```bash
aios comm consumer status KERNEL_LIFECYCLE learning-lifecycle
```

```json
{
  "numPending": 48210,
  "numAckPending": 1000,
  "maxAckPending": 1000,
  "numRedelivered": 412,
  "stalled": true,
  "lastAckAt": "2026-07-22T14:58:03Z"
}
```

Leitura: `numAckPending` no teto e `stalled: true` significam que o **consumidor** não
está confirmando — o barramento já o contém (`AIOS-BUS-0014`), protegendo os demais. A
ação é no consumidor: escalar instâncias no mesmo `durable` ou corrigir o processamento.
Aumentar `maxAckPending` apenas adia o problema.

---

## 9. Triagem e replay de DLQ

```bash
# 1. inspecionar padrões
aios comm dlq list --stream MEMORY_EVENTS --since 24h --group-by error
# ERRO                                              QTD   CONSUMIDOR
# ValidationException: campo 'layer' ausente        142   learning-consolidation
# TimeoutException: upstream 017 indisponível         8   learning-consolidation

# 2. corrigir a causa e implantar  (NÃO pule esta etapa)

# 3. replay em lote pequeno primeiro
aios comm dlq replay --stream MEMORY_EVENTS --error 'campo .layer. ausente' --limit 10
# → 10 replayed; monitorar antes de liberar o restante

# 4. o que for irrecuperável, descartar com justificativa
aios comm dlq discard 01J9ZCAW3Y5A7C9E1G3J5L7N9Q \
  --reason "evento de schema v0 descontinuado em 2026-06; sem consumidor"
```

`bus.dlq.auto_replay` é `false` por default exatamente para tornar o passo 2
inevitável.

---

## 10. Verificar isolamento entre tenants

```bash
# Credencial do tenant acme tentando ler tráfego de globex
nats sub 'aios.globex.>' --creds /etc/aios/creds/acme.creds
# → nenhuma mensagem: a conta globex é INVISÍVEL ao roteamento da conta acme

nats sub 'aios.>' --creds /etc/aios/creds/acme.creds
# → apenas tráfego de acme, mesmo com o curinga mais amplo possível
```

Esse é o teste T-SEC-01 de `./Testing.md`. Note a diferença em relação a um modelo de
permissões: aqui não há regra a ser mal escrita — a outra conta simplesmente não existe
do ponto de vista do roteador.

---

## 11. Armadilhas Comuns

| Armadilha | Sintoma | Correção |
|-----------|---------|----------|
| Subject por instância | Explosão de cardinalidade; catálogo inútil | Tipo no subject, instância no envelope (§4). |
| Iterar destinatários na aplicação | Cliente saturado; alerta de amplificação | Canal de *Agent Group* (§5). |
| Usar o barramento como banco | Replay para "descobrir o estado atual" | Estado autoritativo em `../005-Database/`. |
| Payload grande | `AIOS-BUS-0006`; latência de terceiros degradada | Referência a objeto em MinIO (ADR-0208). |
| `ack_policy: none` em fluxo de controle | Perda silenciosa disfarçada de desempenho | `explicit` sempre em `traffic_class = control`. |
| Não deduplicar no consumidor | Efeito duplicado sob reentrega | Dedupe por `event.id` é obrigação do consumidor (RFC-0001 §5.5). |
| Aumentar `max_ack_pending` para "resolver" lag | Problema adiado e maior | Escalar instâncias no mesmo `durable` (§8). |
| Replay antes de corrigir a causa | Mesma mensagem volta à DLQ | Triagem primeiro (§9). |
| Repetir imediatamente após `AIOS-BUS-0007` | Realimenta a saturação | Respeitar `Retry-After`. |
| Gastar 5 tentativas em erro permanente | Backlog atrasado | `msg.term()` envia direto à DLQ. |

---

## 12. Referências

- API de controle: `./API.md` · Contrato de mensagem: `./Events.md`
- Configuração: `./Configuration.md` · CLI: `../030-CLI/` · SDKs: `../031-SDK/`
- Perguntas frequentes: `./FAQ.md`
