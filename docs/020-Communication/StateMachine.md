---
Documento: StateMachine
Módulo: 020-Communication
Status: Draft
Versão: 0.1
Última atualização: 2026-07-22
Responsável (RACI-A): Arquiteto do Módulo 020-Communication
ADRs relacionados: ADR-0203, ADR-0205, ADR-0207
RFCs relacionados: RFC-0001, RFC-0201
Depende de: _DESIGN_BRIEF.md §4, API.md, Events.md, Database.md
---

# 020-Communication — Máquinas de Estado

A entidade com ciclo de vida governado neste módulo é a **sessão A2A**
(`comm.a2a_session`). Streams possuem um ciclo declarativo simples (§5) e entradas de
DLQ um ciclo de três estados (§6). Subjects, consumidores e canais de grupo são
configurações versionadas por OCC, sem FSM.

Esta é a reprodução detalhada de `./_DESIGN_BRIEF.md` §4; qualquer divergência é
defeito deste documento.

---

## 1. Estados (`A2ASessionState`)

| Estado | Descrição | Ação de entrada | Terminal? |
|--------|-----------|-----------------|-----------|
| `Requested` | Iniciador pediu sessão; par ainda não respondeu. | Aloca `urn`, registra `initiator_urn`/`peer_urn`, arma timeout de handshake. | não |
| `Negotiating` | Par respondeu; capacidades em negociação e submetidas ao PDP. | Registra o conjunto proposto de `capabilities`. | não |
| `Established` | Sessão autorizada e ativa. | Aloca `channel_subject` exclusivo; emite `comm.a2a.established`. | não |
| `Suspended` | Ativa, porém pausada (cota excedida ou capacidade revogada). | Recusa mensagens com `AIOS-A2A-0004`; mantém o canal alocado. | não |
| `Closing` | Encerramento em curso; drenagem das mensagens em voo. | Interrompe novas mensagens; aguarda entrega ou expiração. | não |
| `Closed` | Encerrada normalmente. | Libera o canal; grava `closed_at` e `close_reason`; emite `comm.a2a.closed`. | **sim** |
| `Rejected` | Recusada na negociação. | Grava motivo; emite `comm.a2a.rejected`. | **sim** |
| `Failed` | Encerrada por erro irrecuperável. | Libera o canal; emite `comm.a2a.failed`. | **sim** |

---

## 2. Transições, gatilhos e guardas

| # | De → Para | Gatilho | Guarda (DEVE ser verdadeira) | Ação de saída |
|---|-----------|---------|------------------------------|----------------|
| T-01 | ∅ → `Requested` | `OpenSession` | Capability `a2a:session:open` ∧ `tenant` do par compatível ∧ cota de sessões do agente disponível. | Persiste a sessão. |
| T-02 | `Requested` → `Negotiating` | par respondeu ao handshake | Resposta dentro de `bus.a2a.handshake_timeout_ms` ∧ identidade do par verificada (mTLS/JWT). | Registra proposta de capacidades. |
| T-03 | `Requested` → `Failed` | timeout de handshake | Sem resposta no prazo. | `AIOS-A2A-0002`; emite `comm.a2a.failed`. |
| T-04 | `Negotiating` → `Established` | PDP autorizou o conjunto negociado | Decisão `allow` para **todas** as `capabilities` ∧ canal privado alocado. | Emite `comm.a2a.established`. |
| T-05 | `Negotiating` → `Rejected` | PDP negou ∨ par recusou ∨ capacidades incompatíveis | Motivo registrado em `close_reason`. | `AIOS-A2A-0001`/`AIOS-A2A-0007`. |
| T-06 | `Established` → `Suspended` | cota excedida ∨ `policy.bundle.updated` revoga capacidade | Cota de mensagens/bytes estourada ∨ nova decisão do PDP nega capacidade em uso. | Passa a recusar mensagens. |
| T-07 | `Suspended` → `Established` | cota renovada ∨ reautorização | Nova janela de cota ∨ decisão `allow` do PDP. | Retoma o canal. |
| T-08 | `Established`/`Suspended` → `Closing` | `CloseSession` ∨ ociosidade | Pedido explícito ∨ `now - last_activity_at > bus.a2a.idle_timeout_ms`. | Inicia drenagem. |
| T-09 | `Closing` → `Closed` | drenagem concluída | Mensagens em voo entregues ou expiradas ∧ canal liberado ∧ evento emitido. | Emite `comm.a2a.closed`. |
| T-10 | `Established`/`Suspended`/`Closing` → `Failed` | erro irrecuperável | Par desapareceu além do limite ∨ violação de protocolo ∨ revogação de identidade pelo `021`. | Emite `comm.a2a.failed`. |

Gatilho recebido em estado que não admite a transição retorna `AIOS-A2A-0003` e
**NÃO DEVE** alterar o estado.

---

## 3. Invariantes

| # | Invariante | Como é garantido | Violação |
|---|-----------|------------------|----------|
| I1 | Sessão não-terminal possui **exatamente um** `channel_subject` exclusivo, nunca reutilizado após o estado terminal. | Alocação por ULID na entrada de `Established`; liberação em `Closed`/`Failed`. | Colisão de canal ⇒ incidente P1 |
| I2 | Mensagens só trafegam com estado `Established`; em `Suspended` são **recusadas**, não enfileiradas. | Verificação no `A2AGateway` a cada mensagem. | `AIOS-A2A-0004` |
| I3 | Toda transição terminal emite exatamente um evento `aios.<tenant>.comm.a2a.<acao>` e um registro em `../025-Audit/`. | Outbox transacional + dedupe por `event.id`. | Evento ausente/duplicado ⇒ defeito de relay |
| I4 | `capabilities` **NÃO DEVEM** ser ampliadas após `Established`. | Guarda no `A2AGateway`; ampliação exige **nova** sessão. | Recusa com `AIOS-A2A-0007` |
| I5 | Transição só ocorre com o `version` esperado (OCC). | Camada de persistência. | `AIOS-BUS-0009` |

O invariante I4 merece nota: permitir ampliação em sessão viva transformaria a
autorização inicial num cheque em branco — a decisão do PDP passaria a valer para um
conjunto de capacidades que ele nunca avaliou.

---

## 4. Diagrama de estados (ASCII)

```
        OpenSession (T-01)
   ∅ ───────────────────────▶ ┌───────────┐
                               │ Requested │
                               └─────┬─────┘
             handshake ok (T-02)     │     timeout (T-03)
          ┌─────────────────────────┤────────────────────────┐
          ▼                          │                        ▼
   ┌─────────────┐                   │                  ┌──────────┐
   │ Negotiating │                   │                  │  Failed  │◀── erro irrecuperável
   └──────┬──────┘                   │                  └──────────┘        (T-10)
          │ PDP allow (T-04)         │  PDP deny /            ▲
          │                          │  par recusa (T-05)     │
          │                          ▼                        │
          │                   ┌────────────┐                  │
          │                   │  Rejected  │ (terminal)       │
          │                   └────────────┘                  │
          ▼                                                    │
   ┌─────────────┐  cota/revogação (T-06)  ┌─────────────┐    │
   │ Established │────────────────────────▶│  Suspended  │────┘
   │             │◀────────────────────────│             │
   └──────┬──────┘  reautorização (T-07)   └──────┬──────┘
          │                                        │
          │ close / ociosidade (T-08)              │ close (T-08)
          ▼                                        ▼
   ┌─────────────┐        drenagem concluída (T-09)      ┌──────────┐
   │   Closing   │──────────────────────────────────────▶│  Closed  │ (terminal)
   └─────────────┘                                        └──────────┘
```

---

## 5. Ciclo declarativo de `StreamDefinition`

| Estado | Significado | Próximos |
|--------|-------------|----------|
| `declared` | Declaração registrada em `comm.stream`, ainda não materializada no NATS. | `active` |
| `active` | Stream existe no JetStream e aceita publicação. | `draining` |
| `draining` | Não aceita novas publicações; consumidores drenam o backlog. | `retired` |
| `retired` | Removido do NATS; declaração retida para histórico. | — (terminal) |

```
   declared ──materializa──▶ active ──:drain──▶ draining ──backlog=0──▶ retired
                               │
                               └── mudança incompatível (ADR-0203):
                                   cria novo stream → espelha → drena o antigo
```

Mudanças **compatíveis** (aumentar `max_age`, `max_bytes`) são aplicadas em lugar.
Mudanças **incompatíveis** (número de réplicas, conjunto de subjects) exigem a
migração espelhada acima — nunca a recriação destrutiva do stream, que perderia o
backlog não consumido.

---

## 6. Ciclo de `DeadLetterEntry`

| Estado | Significado | Próximos |
|--------|-------------|----------|
| `quarantined` | Mensagem esgotou `max_deliver`; envelope preservado e inspecionável. | `replayed`, `discarded` |
| `replayed` | Reinjetada no subject original após autorização. | — (terminal) |
| `discarded` | Descartada deliberadamente, com justificativa auditada. | — (terminal) |

```
   quarantined ──:replay (PDP + Idempotency-Key)──▶ replayed   (terminal)
        │
        └────── :discard (justificativa obrigatória) ──▶ discarded (terminal)
        │
        └────── expiração (bus.dlq.retention_days = 30) ──▶ removida do catálogo
```

Não existe transição automática de `quarantined` para `replayed`:
`bus.dlq.auto_replay` é `false` por default (ADR-0205), porque replay antes da
correção reproduz exatamente o mesmo defeito.

---

## 7. Mapa Estado → Evento → Erro

| Estado alcançado | Evento emitido | Erro típico ao tentar sair indevidamente |
|------------------|----------------|------------------------------------------|
| `Established` (A2A) | `aios.<tenant>.comm.a2a.established` | `AIOS-A2A-0007` (ampliar capacidades) |
| `Suspended` (A2A) | — (interno; visível em `GetSession`) | `AIOS-A2A-0004` (mensagem recusada) |
| `Rejected` (A2A) | `aios.<tenant>.comm.a2a.rejected` | `AIOS-A2A-0003` (transição inválida) |
| `Closed` (A2A) | `aios.<tenant>.comm.a2a.closed` | `AIOS-A2A-0003` |
| `Failed` (A2A) | `aios.<tenant>.comm.a2a.failed` | `AIOS-A2A-0003` |
| `active` (stream) | `aios._platform.comm.stream.created` | `AIOS-BUS-0004` (declaração inválida) |
| `retired` (stream) | `aios._platform.comm.stream.retired` | `AIOS-BUS-0001` |
| `quarantined` (DLQ) | `aios.<tenant>.comm.delivery.deadlettered` | — |
| `replayed` (DLQ) | `aios.<tenant>.comm.delivery.replayed` | `AIOS-BUS-0002` (sem capability) |

---

## 8. Referências

- Brief: `./_DESIGN_BRIEF.md` §4
- API e catálogo de erros: `./API.md` · Eventos: `./Events.md`
- Casos de uso UC-002, UC-006, UC-007, UC-011, UC-012: `./UseCases.md`
- Sequências correspondentes: `./SequenceDiagrams.md`
