---
Documento: SequenceDiagrams
Módulo: 020-Communication
Status: Draft
Versão: 0.1
Última atualização: 2026-07-22
Responsável (RACI-A): Arquiteto do Módulo 020-Communication
ADRs relacionados: ADR-0200, ADR-0202, ADR-0205, ADR-0206, ADR-0207
RFCs relacionados: RFC-0001, RFC-0200, RFC-0201
Depende de: UseCases.md, StateMachine.md, API.md, Events.md
---

# 020-Communication — Diagramas de Sequência

Convenção: `──▶` chamada síncrona, `╌╌▶` mensagem assíncrona (NATS), `◀──` resposta,
`[guard]` condição, `⟲` retry. Toda mensagem carrega os cabeçalhos de correlação da
RFC-0001 §5.6.

---

## 1. Caminho feliz — Publicação com Outbox e entrega durável (UC-004, UC-005)

```
 006-Kernel  Outbox(PG)  Relay  EnvelopeVal.  SubjRegistry  NATS/JetStream  025-Audit
     │           │         │         │             │              │            │
     ├─ BEGIN; muda ACB; INSERT outbox; COMMIT ▶   │              │            │
     │           │◀─ poll ─┤         │             │              │            │
     │           │         ├─ validate(env) ──────▶│              │            │
     │           │         │         ├─ type↔subject? tenant↔conta? schema? ok │
     │           │         │         ├─ produtor autorizado? ────▶│            │
     │           │         │         │◀──── sim (006-Kernel) ─────┤            │
     │           │         ├─ publish aios.acme.agent.lifecycle.spawned ──────▶│
     │           │         │◀────────── ack (stream=KERNEL_LIFECYCLE, seq=8412) │
     │           │◀─ published=true ─┤             │              │            │
     │           │         │         │             │              ├─ deliver ─▶│
     │           │         │         │             │              │◀── ack ────┤
     │           │         │         │             │              │            │
```

**Notas.** O evento só existe porque a linha de Outbox foi commitada na **mesma**
transação da mudança de estado. Se o relay morrer antes de publicar, ele republica; o
`dedupe_window` do stream (2 min) descarta a duplicata pelo `event.id`, produzindo
efeito exatamente-uma-vez sem coordenação distribuída (RFC-0001 §5.5).

---

## 2. Caminho feliz — Request/reply entre serviços (UC-015)

```
 006-Kernel        NATS         022-Policy(PDP)
     │              │                  │
     ├─ request(aios.acme.policy.decision.request, reply=_INBOX.xyz) ▶
     │              ├─ deliver ───────▶│
     │              │                  ├─ avalia (RBAC+ABAC)
     │              │◀─ publish _INBOX.xyz (allow) ─┤
     │◀─ resposta (p99 ≤ 10 ms, NFR-003) ───────────┤
     │              │                  │
     │  [sem resposta em bus.request.timeout_ms = 5s → AIOS-BUS-0012]
```

O barramento não conhece a semântica da decisão — apenas correlaciona a resposta com o
`reply-to` gerado e propaga o `traceparent`, de modo que a consulta ao PDP apareça no
mesmo trace da syscall que a originou.

---

## 3. Caminho feliz — Difusão para *Agent Group* (UC-013)

```
 Coordenador  SelectiveRouter   NATS      agente-1 … agente-1000
      │             │             │            │
      ├─ publish aios.acme.group.squad-7.task.assigned ▶
      │             ├─ scope=group? fanout=1000 ≤ limit? ok
      │             ├──────────── 1 publicação ──▶│
      │             │             ├─ fan-out do broker ─────▶ (1000 entregas)
      │             │             │            │
      │  custo do produtor: 1 publicação (NFR-016), independente do tamanho do grupo
```

Este é o diagrama que justifica o módulo existir: a alternativa ingênua — o
coordenador publicar mil vezes, uma por destinatário — transforma coordenação em
gargalo assim que a frota cresce.

---

## 4. Falha — Subject não registrado (UC-004 E1)

```
 Produtor    Relay    SubjRegistry   NATS     Outbox(PG)
     │         │           │           │          │
     ├─ COMMIT (estado + outbox) ──────────────────▶
     │         ├─ resolve(aios.acme.tool.usage.spike) ▶
     │         │◀──── não registrado ──┤           │
     │         │  AIOS-BUS-0004        │           │
     │         ├─ NÃO publica; mantém published=false ─────▶
     │         │  (alerta ao dono do módulo)       │
```

A mensagem **não se perde**: ela permanece no Outbox. O produtor registra o subject
(UC-001) e o relay publica na varredura seguinte. A falha aponta para o autor do
defeito, não para N consumidores.

---

## 5. Falha — Mensagem envenenada até a DLQ (UC-006)

```
 NATS/JetStream   Consumidor(025)   DeadLetterMgr   PG(comm)     NATS
       │                │                 │            │           │
       ├─ deliver #1 ──▶│ (exceção)       │            │           │
       │  … ack_wait 30s, backoff 1s …    │            │           │
       ├─ deliver #2 ──▶│ (exceção)       │            │           │
       │  … backoff 5s / 30s / 2min …     │            │           │
       ├─ deliver #5 ──▶│ (exceção)       │            │           │
       ├── max_deliver(5) esgotado ──────▶│            │           │
       │                │                 ├─ INSERT dlq_entry ────▶│
       │                │                 │  (envelope, last_error, delivery_count)
       │                │                 ├─ INSERT outbox ───────▶│
       │                │                 │            │  ╌╌╌╌╌╌╌▶│
       │                │                 │   aios.acme.comm.delivery.deadlettered
```

O consumidor não trava: a mensagem sai do fluxo após a quinta tentativa e o backlog
volta a andar. O defeito fica **visível** em `comm.dlq_entry`, com o envelope
preservado para diagnóstico.

---

## 6. Falha — *Slow consumer* contido (UC-014 E1)

```
 Produtores   NATS   Consumidor-lento   Consumidor-saudável   FlowController
     │          │           │                    │                 │
     ├─ 5k msg/s ─▶        │                    │                 │
     │          ├─ deliver ─▶ (não confirma)     │                 │
     │          │  pending: 200 … 800 … 1000     │                 │
     │          ├─ pending = max_ack_pending ───────────────────────▶│
     │          │  [entrega contida para ESTE consumidor]           │
     │          │           │                    │                 │
     │          ├─ deliver ──────────────────────▶ (segue normal)   │
     │          │           │                    │  p99 intacto     │
     │          │           │                    │  (degradação ≤ 10%, NFR-011)
     │          │  ╌╌╌╌╌╌╌╌ aios.acme.comm.consumer.stalled ╌╌╌╌╌╌▶ alerta
```

A contenção é **por consumidor**, não por stream: quem está lento para de receber, e
quem está saudável continua no ritmo. Sem esse bulkhead, um único consumidor
defeituoso arrastaria a latência de todos os demais.

---

## 7. Falha — Perda de nó e quórum JetStream mantido

```
 Produtor    nats-1     nats-2     nats-3    Consumidor   027-Cluster
     │          ✗          │          │           │            │
     ├─ publish (ack) ─────┼─────────▶│           │            │
     │          │          │  [quórum 2/3 mantido]│            │
     │◀── ack (seq) ───────┤          │           │            │
     │          │          ├─ deliver ────────────▶            │
     │          │          │          │           │            │
     │  ╌╌ aios._platform.comm.cluster.degraded ╌╌╌╌╌╌╌╌╌╌╌╌╌╌▶│
     │          │  (nat-1 reingressa e ressincroniza streams)   │
```

Com R=3, a perda de um nó é **transparente** para publicação e entrega (NFR-006). Se o
quórum se perde (2 nós fora), a publicação **durável** passa a falhar com
`AIOS-BUS-0011` — e os produtores retêm no Outbox, sem perda.

---

## 8. Sessão A2A entre agentes internos (UC-011)

```
 agente-A   A2AGateway   PDP(022)   SessionStore   NATS      agente-B   025-Audit
     │           │           │            │          │           │          │
     ├ OpenSession(peer=B, caps=[task.delegate, memory.share]) ▶ │          │
     │           │ T-01 Requested         │          │           │          │
     │           ├─ handshake ────────────────────────────────▶ │           │
     │           │◀──────────── resposta (≤ 3s) ────────────────┤           │
     │           │ T-02 Negotiating       │          │           │          │
     │           ├─ Decide(task.delegate) ▶          │           │          │
     │           │◀──── allow ────────────┤          │           │          │
     │           ├─ Decide(memory.share) ─▶          │           │          │
     │           │◀──── allow ────────────┤          │           │          │
     │           │ T-04 Established       │          │           │          │
     │           ├─ aloca canal aios.acme.a2a.session.01J9ZC… ──▶│          │
     │           ├─ persiste ────────────────────────▶          │           │
     │           ├─ audita (caps, par, tenant) ──────────────────────────────▶│
     │◀─ 201 + channelSubject ─┤          │          │           │          │
     │           │             │          │          │           │          │
     │═══ mensagens no canal privado (contabilizadas em message_count) ═════│
     │           │             │          │          │           │          │
     │           │  … ociosidade > 300s → T-08 Closing → T-09 Closed …      │
     │           │  ╌╌ aios.acme.comm.a2a.closed ╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌▶│
```

Cada capacidade é autorizada **individualmente**. Depois de `Established`, o conjunto
é imutável (invariante I4) — ampliar exige nova sessão, com nova avaliação do PDP.

---

## 9. Falha — Capacidade revogada durante sessão ativa (T-06)

```
 022-Policy    NATS     A2AGateway   agente-A   agente-B
      │          │           │           │          │
      ├╌ policy.bundle.updated ╌▶        │          │
      │          │           ├─ reavalia sessões ativas
      │          │           │  [memory.share agora negado]
      │          │           │ T-06 → Suspended     │
      │          │           │           │          │
      │          │           │◀─ mensagem ─┤        │
      │          │           ├─ AIOS-A2A-0004 ────▶ │
      │          │           │  (recusada, NÃO enfileirada — invariante I2)
      │          │           │           │          │
      │          │  [reautorização posterior → T-07 Established]
```

Recusar em vez de enfileirar é deliberado: uma fila de mensagens aguardando
reautorização acumularia trabalho que talvez nunca seja permitido, e entregaria em
lote no momento da liberação — exatamente quando o sistema menos espera.

---

## 10. Replay controlado de DLQ (UC-007)

```
 SRE   004-API   DeadLetterMgr   PDP(022)   PG(comm)   NATS     Consumidor
  │       │            │             │          │        │           │
  ├ GET /v1/comm/dlq ─▶│             │          │        │           │
  │◀─ 3 entradas, mesmo last_error ──┤          │        │           │
  │  (causa raiz corrigida e implantada)        │        │           │
  ├ POST /dlq/{id}:replay ──────────▶│          │        │           │
  │       │            ├─ Decide(bus:dlq:replay) ▶       │           │
  │       │            │◀──── allow ─┤          │        │           │
  │       │            ├─ republica no subject original (mesmo event.id) ─▶│
  │       │            ├─ status=replayed ─────▶│        │           │
  │       │            │  ╌╌ aios.acme.comm.delivery.replayed ╌╌╌╌╌╌▶│
  │◀─ 200 ─┤            │             │          │        │           │
```

O `event.id` é preservado no replay: se o consumidor já havia processado parcialmente
a mensagem antes de falhar, sua deduplicação evita efeito duplicado.

---

## 11. Referências

- Casos de uso: `./UseCases.md` · FSM: `./StateMachine.md`
- API e erros: `./API.md` · Eventos: `./Events.md`
- Falhas e recuperação: `./FailureRecovery.md`
