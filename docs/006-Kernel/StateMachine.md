---
Documento: StateMachine
Módulo: 006-Kernel
Status: Draft
Versão: 0.1
Última atualização: 2026-07-20
Responsável (RACI-A): Arquiteto do Módulo 006-Kernel
ADRs relacionados: ADR-0061, ADR-0062, ADR-0064, ADR-0067 (deste módulo, a propor)
RFCs relacionados: RFC-0001 (baseline); RFC-0007 (ACB & Quota Model, a propor)
Depende de: 006-Kernel/_DESIGN_BRIEF.md, 009-Scheduler, 008-Lifecycle
---

# 006-Kernel — Máquina(s) de Estados

> A máquina de estados canônica deste módulo é a **FSM do Agent Control Block
> (ACB)**, definida em `./_DESIGN_BRIEF.md` §4 e reproduzida aqui sem
> alteração. Este documento também detalha duas sub-máquinas internas que
> servem diretamente a essa FSM: o ciclo de vida de uma **reserva de cota**
> (`ResourceQuotaManager`) e o ciclo de vida de um **resultado idempotente**
> (`IdempotencyStore`). Ambas as sub-máquinas são elaborações consistentes das
> responsabilidades R-04 e R-07 do brief — não introduzem estados do ACB nem
> o contradizem.

## Índice

1. FSM canônica do ACB — estados
2. FSM canônica do ACB — transições, gatilhos e guardas
3. FSM canônica do ACB — diagrama ASCII
4. Invariantes da FSM do ACB
5. Ações de entrada/saída por estado
6. Sub-máquina: ciclo de vida da reserva de cota
7. Sub-máquina: ciclo de vida do resultado idempotente
8. Referências

---

## 1. FSM Canônica do ACB — Estados (`AcbState`)

| Estado | Descrição | Terminal? |
|--------|-----------|-----------|
| `Pending` | ACB criado; aguardando admissão do Scheduler. | não |
| `Admitted` | Scheduler aceitou; slot reservado; aguardando materialização. | não |
| `Running` | Runtime materializado; agente executando loop cognitivo. | não |
| `Suspended` | Agente pausado (voluntário ou por preempção); estado em RAM/quente. | não |
| `Hibernated` | Agente *cold*: estado em armazenamento durável (MinIO/PG); zero RAM. | não |
| `Terminating` | Encerramento em curso (drenagem + compensação). | não |
| `Terminated` | Encerrado; ACB retido para auditoria; recursos liberados. | **sim** |
| `Failed` | Falha irrecuperável na materialização/execução. | **sim** |

---

## 2. FSM Canônica do ACB — Transições, Gatilhos e Guardas

| # | De → Para | Gatilho (syscall/evento) | Guarda (DEVE ser verdadeira) |
|---|-----------|--------------------------|------------------------------|
| T-01 | ∅ → `Pending` | `spawn` | Capability `agent:spawn` concedida ∧ cota `agent_count` do tenant disponível. |
| T-02 | `Pending` → `Admitted` | resposta do Scheduler | Scheduler retornou slot ∧ admissão aprovada (cotas ok). |
| T-03 | `Pending` → `Failed` | timeout/rejeição do Scheduler | Admissão negada ou timeout > `spawn.admission_timeout`. |
| T-04 | `Admitted` → `Running` | confirmação do Lifecycle | Runtime materializado ∧ `runtime_ref` atribuído ∧ contexto restaurado. |
| T-05 | `Admitted` → `Failed` | falha de materialização | Runtime não iniciou dentro de `spawn.boot_timeout`. |
| T-06 | `Running` → `Suspended` | `suspend` / preempção | Capability `agent:suspend` ∧ agente em estado suspensível (sem seção crítica não-idempotente). |
| T-07 | `Suspended` → `Running` | `resume` | Capability `agent:resume` ∧ slot re-reservado no Scheduler. |
| T-08 | `Suspended` → `Hibernated` | política de ociosidade | `now - last_active_at > hibernation.idle_ttl`. |
| T-09 | `Hibernated` → `Admitted` | `resume` / demanda | Checkpoint válido ∧ Scheduler reserva slot (materialização sob demanda, alvo p99 < 250 ms). |
| T-10 | `Running`/`Suspended`/`Hibernated` → `Terminating` | `kill` | Capability `agent:kill` ∧ (chamador = owner ∨ role admin). |
| T-11 | `Terminating` → `Terminated` | drenagem concluída | Compensações executadas ∧ cotas liberadas ∧ evento `terminated` emitido. |
| T-12 | `Running` → `Failed` | falha do runtime | Heartbeat perdido > `runtime.heartbeat_deadline` ∧ retries esgotados. |
| T-13 | `Running`/`Suspended` → (mesmo) | `checkpoint` | Capability `agent:checkpoint`; snapshot durável concluído; atualiza `checkpoint_ref`. |

> Notação de guarda: `∧` = E lógico. Toda guarda **DEVE** ser avaliada antes da
> efetivação da transição; falha de guarda **NÃO DEVE** alterar `State` nem
> `Version` do ACB (transição rejeitada com `AIOS-KERNEL-0002`).

---

## 3. FSM Canônica do ACB — Diagrama ASCII

```
                    spawn (T-01)
        ∅ ─────────────────────────▶ ┌─────────┐
                                      │ Pending │
                                      └────┬────┘
                     admit (T-02)          │  reject/timeout (T-03)
             ┌────────────────────────────┤────────────────────────┐
             ▼                             │                        ▼
        ┌─────────┐   boot ok (T-04)       │                   ┌────────┐
        │Admitted │───────────────┐        │                   │ Failed │◀── boot fail (T-05)
        └────┬────┘               ▼        │                   └────────┘        ▲
             │             ┌───────────┐   │                                     │ runtime fail
   resume    │             │  Running  │───┼──── checkpoint (T-13, self) ───┐    │  (T-12)
   (T-09)    │             └──┬───┬────┘   │                                │    │
             │        suspend │   │ kill (T-10)                             ▼    │
             │         (T-06) │   └──────────────┐               ┌──────────────┴──┐
             │                ▼                  ▼               │   (self loop)   │
             │          ┌───────────┐      ┌────────────┐        └─────────────────┘
             └──────────│ Suspended │      │Terminating │
                idle    └──┬────┬───┘      └─────┬──────┘
                (T-08)     │    │ resume (T-07)  │ drain done (T-11)
                           ▼    └──────────▶Running     ▼
                     ┌───────────┐            ┌────────────┐
                     │Hibernated │─resume────▶│ Terminated │  (terminal)
                     └───────────┘  (T-09)    └────────────┘
```

**Leitura do diagrama:**
- Setas sólidas = transições dirigidas por syscall explícita do chamador
  (`spawn`, `kill`, `suspend`, `resume`, `checkpoint`).
- Setas rotuladas com eventos externos (`admit`, `boot ok`, `boot fail`,
  `runtime fail`, `drain done`) = transições dirigidas por eventos consumidos
  do `009-Scheduler`/`008-Lifecycle` (ver `./_DESIGN_BRIEF.md` §6.2).
- `checkpoint` (T-13) é um **self-loop**: não muda `State`, apenas atualiza
  `checkpoint_ref` e emite `agent.checkpoint.created`.
- `Terminated` e `Failed` são estados **terminais**: nenhuma transição sai
  deles. Um agente encerrado ou falho só pode ser reconstituído via novo
  `spawn`, criando um **novo** ACB (novo URN).

---

## 4. Invariantes da FSM do ACB

- **I1** — `runtime_ref` não-nulo **se e somente se** `State ∈ {Running,
  Suspended}`.
- **I2** — Todo estado terminal (`Terminated`, `Failed`) **DEVE** preencher
  `terminated_at` e liberar as cotas de escopo `agent`.
- **I3** — Toda transição só ocorre com `version` esperado (OCC); conflito de
  concorrência **DEVE** resultar em retry (até `kernel.acb.optimistic_retry_max`)
  ou em `AIOS-KERNEL-0009`.
- **I4** — Toda transição **DEVE** emitir exatamente um evento
  `aios.<tenant>.agent.lifecycle.<acao>` via Outbox transacional, atomicamente
  com a mutação do ACB (nunca antes, nunca depois, nunca em separado).
- **I5** (derivada) — Não existe transição direta de `Pending` para
  `Suspended`, `Hibernated`, `Terminating` ou `Terminated`: um agente **DEVE**
  atravessar `Admitted`/`Running` antes de qualquer forma de pausa ou
  encerramento coordenado — exceto `kill`, que **PODE** encerrar a partir de
  `Running`, `Suspended` ou `Hibernated` diretamente (T-10), mas nunca a partir
  de `Pending`/`Admitted` sem antes falhar a admissão (T-03/T-05) ou completar
  o boot (T-04).
- **I6** (derivada) — Um ACB em `Hibernated` **NÃO DEVE** consumir cota de
  `cpu_ms`/`mem_mib` (zero RAM); apenas cotas de armazenamento/retenção
  aplicam-se nesse estado.

---

## 5. Ações de Entrada/Saída por Estado

| Estado | Ação de entrada (*on-enter*) | Ação de saída (*on-exit*) |
|--------|-------------------------------|------------------------------|
| `Pending` | Persistir ACB (`Version=0`); reservar cota `agent_count`; publicar `agent.lifecycle.spawned`. | — |
| `Admitted` | Registrar `slot` recebido do Scheduler; iniciar temporizador `spawn.boot_timeout_ms`; publicar `agent.lifecycle.admitted`. | Cancelar temporizador de boot. |
| `Running` | Atribuir `runtime_ref`; restaurar `memory_ptr`/`context_ptr`; iniciar heartbeat; publicar `agent.lifecycle.running`; atualizar `last_active_at`. | Parar heartbeat local (mantido pelo Runtime Supervisor). |
| `Suspended` | Persistir estado quente; iniciar temporizador `hibernation.idle_ttl_ms`; publicar `agent.lifecycle.suspended`. | Cancelar temporizador de ociosidade. |
| `Hibernated` | Confirmar `checkpoint_ref` válido; liberar `runtime_ref` (→ `null`); publicar `agent.lifecycle.hibernated`. | — |
| `Terminating` | Iniciar drenagem (finalizar syscalls em voo); disparar compensação de reservas pendentes; publicar evento interno de início de drenagem (não externo). | — |
| `Terminated` | Preencher `terminated_at`; liberar cota de escopo `agent`; publicar `agent.lifecycle.terminated`. | — (terminal) |
| `Failed` | Preencher `terminated_at`; liberar cota de escopo `agent`; publicar `agent.lifecycle.failed`; registrar causa de falha em auditoria. | — (terminal) |

---

## 6. Sub-máquina: Ciclo de Vida da Reserva de Cota

> Elaboração interna de `ResourceQuotaManager` (R-04, FR-004). Não é uma FSM do
> ACB; rege o ciclo de vida de uma **reserva individual** de cota associada a
> uma syscall (ex.: `remember`, `invoke_tool`, `route_model`). Cada reserva é
> identificada por um `reservationId` (ULID) e vive tipicamente por
> milissegundos.

### 6.1 Estados

| Estado | Descrição | Terminal? |
|--------|-----------|-----------|
| `Requested` | Pedido de reserva recebido do `ResourceBrokerRouter`. | não |
| `Reserved` | Token-bucket decrementado atomicamente (Lua/Redis); aguardando confirmação da execução do broker. | não |
| `Committed` | Execução do broker de recurso concluída com sucesso; consumo confirmado (não reversível). | **sim** |
| `Released` | Execução falhou ou foi cancelada; quantidade devolvida ao saldo. | **sim** |
| `Expired` | Reserva não confirmada/liberada dentro do TTL de reserva; liberada automaticamente pelo reconciliador. | **sim** |

### 6.2 Diagrama ASCII

```
                    reserve() ok
        ∅ ─────────────────────────▶ ┌───────────┐
                                       │ Requested │
                                       └─────┬─────┘
                              token-bucket ok │  token-bucket excedido
                       ┌──────────────────────┤  (AIOS-QUOTA-0001/0002/0003)
                       ▼                       ▼
                 ┌───────────┐           ┌───────────┐
                 │ Reserved  │           │ Released  │ (terminal, rejeição imediata)
                 └──┬─────┬──┘           └───────────┘
        commit() ok │     │ release() (falha/cancelamento do broker)
                     │     └────────────────────────────┐
                     ▼                                    ▼
              ┌────────────┐                        ┌───────────┐
              │ Committed  │ (terminal)              │ Released  │ (terminal)
              └────────────┘                        └───────────┘

        Reserved ──(TTL de reserva expirado, sem commit/release)──▶ Expired (terminal)
```

### 6.3 Transições e guardas

| De → Para | Gatilho | Guarda |
|-----------|---------|--------|
| ∅ → `Requested` | `IResourceQuotaManager.Reserve(subjectUrn, request)` | `Quota` ativa para `subjectUrn`. |
| `Requested` → `Reserved` | Script Lua decrementa token-bucket com sucesso | Saldo disponível ≥ quantidade solicitada. |
| `Requested` → `Released` | Script Lua rejeita (saldo insuficiente) | — (rejeição imediata, sem alocação). |
| `Reserved` → `Committed` | `IResourceQuotaManager.Commit(reservation)` | `reservation` existe e não expirou. |
| `Reserved` → `Released` | `IResourceQuotaManager.Release(reservation)` | `reservation` existe (mesmo expirada — idempotente). |
| `Reserved` → `Expired` | Reconciliador periódico detecta TTL vencido sem commit/release | `now - reservedAt > quota.reservation_ttl`. |

> **Invariante da sub-máquina:** toda `Reserved` **DEVE** transicionar para
> exatamente um estado terminal (`Committed`, `Released` ou `Expired`); reservas
> penduradas indefinidamente são impossíveis por construção (reconciliação
> periódica garante `Expired` como rede de segurança), preservando NFR-009
> (overshoot de cota ≤ 1%).

---

## 7. Sub-máquina: Ciclo de Vida do Resultado Idempotente

> Elaboração interna de `IdempotencyStore` (R-07, FR-006, RFC-0001 §5.5).
> Rege o ciclo de vida de um resultado associado a um `Idempotency-Key` por
> `tenant_id`.

### 7.1 Estados

| Estado | Descrição | Terminal? |
|--------|-----------|-----------|
| `New` | Nenhum resultado prévio encontrado para a chave. | não |
| `InProgress` | Syscall em execução para esta chave; resultado ainda não persistido. | não |
| `Completed` | Resultado persistido e disponível para *replay* determinístico. | **sim** (até expirar) |
| `ConflictDetected` | Nova requisição com mesma chave mas `payloadHash` divergente. | **sim** |

### 7.2 Diagrama ASCII

```
            primeira chamada com a chave
    ∅ ────────────────────────────────────▶ ┌─────┐
                                             │ New │
                                             └──┬──┘
                             inicia execução    │
                                                 ▼
                                          ┌────────────┐
                                          │ InProgress  │
                                          └──────┬──────┘
                    execução concluída (sucesso ou erro)
                                                 ▼
                                          ┌────────────┐
                                          │ Completed  │ (terminal até `retention_h` expirar)
                                          └────────────┘

    repetição com mesma chave, mesmo payloadHash ──▶ retorna Completed (replay, sem reexecutar)
    repetição com mesma chave, payloadHash diferente ──▶ ┌───────────────────┐
                                                          │ ConflictDetected   │ (AIOS-SYSCALL-0002)
                                                          └───────────────────┘
```

### 7.3 Transições e guardas

| De → Para | Gatilho | Guarda |
|-----------|---------|--------|
| ∅ → `New` | Primeira chamada com `Idempotency-Key` inédita | `TryGetCachedResult` retorna vazio. |
| `New` → `InProgress` | `SyscallGateway` inicia execução | — |
| `InProgress` → `Completed` | Execução termina (sucesso ou erro) e `SaveResult` é chamado | Retenção mínima de 24h (`kernel.idempotency.retention_h ≥ 24`). |
| `Completed` → `Completed` (replay) | Nova chamada com mesma `key` e mesmo `payloadHash` | Resultado cacheado retornado sem reexecução. |
| `Completed`/`InProgress` → `ConflictDetected` | Nova chamada com mesma `key` e `payloadHash` divergente | Rejeição imediata. |
| `Completed` → ∅ (expiração) | `now - completedAt > retention_h` | Reconciliador/TTL do armazenamento remove a entrada. |

---

## 8. Referências

- Fonte de verdade: `./_DESIGN_BRIEF.md` §4
- Fluxos que disparam estas transições: `./SequenceDiagrams.md`
- Estruturas de dados: `./ClassDiagrams.md`
- Catálogo de eventos emitidos por transição: `./_DESIGN_BRIEF.md` §6.1
  (a detalhar em `./Events.md`)
- Catálogo de erros referenciados nas guardas: `./_DESIGN_BRIEF.md` §5.2
  (a detalhar em `./API.md`)
- Contratos centrais: `../003-RFC/RFC-0001-Architecture-Baseline.md`

*Fim do documento `StateMachine.md` do módulo 006-Kernel.*
