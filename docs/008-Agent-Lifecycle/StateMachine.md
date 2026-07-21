---
Documento: StateMachine
Módulo: 008-Agent-Lifecycle
Status: Draft
Versão: 0.1
Última atualização: 2026-07-20
Responsável (RACI-A): Arquiteto do Módulo 008 — Agent Lifecycle
ADRs relacionados: ADR-0080, ADR-0081, ADR-0084, ADR-0085, ADR-0087
RFCs relacionados: RFC-0001 (baseline); RFC-0080 (Cold Agent Hibernation & Checkpoint/Restore Protocol, a propor)
Depende de: 008-Agent-Lifecycle/_DESIGN_BRIEF.md, 006-Kernel, 009-Scheduler
---

# 008-Agent-Lifecycle — Máquina(s) de Estados

> A máquina de estados canônica deste módulo é a **FSM do ciclo de vida do
> agente**, definida em `./_DESIGN_BRIEF.md` §4 e reproduzida aqui sem
> alteração — o módulo 008 é a **autoridade única** sobre esta FSM (R1 do
> brief). Este documento também detalha duas sub-máquinas internas que
> servem diretamente a essa FSM: o ciclo de vida de uma **saga de migração**
> (`MigrationJob`) e o ciclo de vida de uma **lease de coordenação**
> (`LeaseManager`). Ambas as sub-máquinas são elaborações consistentes das
> responsabilidades R7 e R10 do brief — não introduzem estados do agente nem
> o contradizem.

## Índice

1. FSM canônica do ciclo de vida — estados
2. FSM canônica do ciclo de vida — diagrama ASCII
3. FSM canônica do ciclo de vida — transições, gatilhos e guardas
4. Invariantes da FSM canônica
5. Ações de entrada/saída por estado
6. Sub-máquina: ciclo de vida da saga de migração (`MigrationJob`)
7. Sub-máquina: ciclo de vida da lease de coordenação (`Lease`)
8. Referências

---

## 1. FSM Canônica do Ciclo de Vida — Estados (`LifecycleState`)

| Estado | Descrição | RAM alocada? | Terminal? |
|--------|-----------|--------------|-----------|
| `Created` | ACB criado pelo Kernel (`006`); ainda não admitido. | não | não |
| `Ready` | Admitido pelo Scheduler (`009`); aguardando slot/execução. | reservada | não |
| `Running` | Runtime (`007`) executando o loop cognitivo. | sim | não |
| `Suspended` | Pausado; memória de trabalho preservada em RAM/Redis. | sim (reduzida) | não |
| `Hibernated` | *Cold agent*: estado serializado em MinIO/PostgreSQL; **RAM liberada**. | não | não |
| `Migrating` | Em migração entre shards/nós (saga ativa, ver §6). | transitória | não |
| `Terminated` | Encerrado normalmente; ACB tombstoned. | não | **sim** |
| `Failed` | Encerrado por falha irrecuperável. | não | **sim** |

---

## 2. FSM Canônica do Ciclo de Vida — Diagrama ASCII

> Reprodução exata de `./_DESIGN_BRIEF.md` §4.2.

```
                    spawn(new)
        ┌────────┐  [G1]      ┌────────┐  start[G3]   ┌─────────┐
   ●───▶│Created │───────────▶│ Ready  │─────────────▶│ Running │
        └────────┘   admit    └────────┘              └────┬────┘
             │       [G2]          ▲                       │  ▲
             │                     │ resume/wake[G5]       │  │ resume[G5]
             │ fail[G9]            │                  suspend│  │
             ▼                     │                    [G4] │  │
        ┌────────┐           ┌──────────┐  hibernate[G6]  ┌──▼──┴────┐
        │ Failed │◀──────────│Hibernated│◀────────────────│Suspended │
        └────────┘  fail[G9] └────┬─────┘   (idle)        └────┬─────┘
             ▲                    │ wake[G5]                    │
             │                    │  (materialize <250ms)       │
             │            migrate[G7]                    migrate[G7]
             │              ┌───────────────┐                   │
             └──────fail────│   Migrating   │◀──────────────────┘
                     [G9]   └───────┬───────┘
                                    │ cutover[G8]
                                    ▼   (→ Ready|Running|Hibernated no destino)
                             ┌────────────┐
              terminate[G10] │ Terminated │  ● (estado terminal)
      (de qualquer não-term.)└────────────┘
```

**Leitura do diagrama:**

- Setas rotuladas com `[Gn]` referenciam as guardas da tabela §3, que
  **DEVEM** ser avaliadas como verdadeiras antes da efetivação da transição.
- `spawn(new)` (T1) é disparado por `aios.<tenant>.agent.control.created`,
  consumido de `006-Kernel` (evento reativo, não syscall direta do 008).
- `admit` (T2), `start` (T3) e `migrate`/`cutover` (T8/T9) são parcialmente
  dirigidos por decisões externas (`009-Scheduler`, `027-Cluster`)
  consumidas via evento, e parcialmente por comando explícito via API
  (`./API.md`).
- `Terminated` e `Failed` são estados **terminais** (INV3): nenhuma
  transição de saída existe. Um agente encerrado ou falho só é reconstituído
  via novo `spawn`, criando um **novo** ACB (novo URN, `agent_id` nunca
  reutilizado — ADR-0089).
- A seta de `fail[G9]` a partir de `Migrating` reflete que uma migração
  irrecuperável (saga travada além de `migration.saga_timeout_ms` sem
  compensação bem-sucedida) pode, em último caso, resultar em `Failed` — ver
  §6.3 para a via preferencial de compensação (`Compensated`, que retorna o
  agente ao estado anterior, não a `Failed`).

---

## 3. FSM Canônica do Ciclo de Vida — Transições, Gatilhos e Guardas

> Reprodução exata de `./_DESIGN_BRIEF.md` §4.3.

| # | De → Para | Gatilho | Guarda (DEVE ser verdadeira) |
|---|-----------|---------|------------------------------|
| T1 | `∅ → Created` | `spawn`/`acb.created` (006) | **G0**: ACB válido, tenant autenticado, política permite `create`. |
| T2 | `Created → Ready` | `admit` (009) | **G1**: Scheduler admitiu (cotas/orçamento ok); política `run` permitida. |
| T3 | `Ready → Running` | `start` | **G3**: slot alocado, Runtime (007) provisionado, lease adquirida, `generation` incrementado. |
| T4 | `Running → Suspended` | `suspend` | **G4**: lease válida; ausência de operação crítica não-preemptível em curso. |
| T5 | `Suspended → Running` | `resume` | **G5**: slot disponível, working memory íntegra, política `run` ainda válida. |
| T6 | `Suspended → Hibernated` | `hibernate` | **G6**: idle ≥ `idle_ttl` **ou** pressão de RAM; checkpoint `consistent` gravado com sucesso. |
| T7 | `Hibernated → Ready/Running` | `wake`/`materialize` | **G5**: checkpoint íntegro (hash ok), Scheduler concede slot; `generation` incrementado. |
| T8 | `Running/Suspended → Migrating` | `migrate` | **G7**: decisão de placement de 009/027; lease adquirida; destino saudável. |
| T9 | `Migrating → Ready/Running/Hibernated` | `cutover` | **G8**: materialização no destino confirmada; origem drenada/invalidada. |
| T10 | `qualquer não-terminal → Terminated` | `terminate` | **G10**: política `terminate` permitida; efeitos colaterais drenados; checkpoint final opcional. |
| T11 | `qualquer não-terminal → Failed` | `fail` | **G9**: erro irrecuperável (timeout de saga, corrupção de checkpoint, runtime perdido além de `max_recovery`). |

Notação de guarda: cada `Gn` é avaliada por `IStateMachineEngine.Evaluate`
(`./ClassDiagrams.md` §3.4) antes de qualquer efeito colateral; falha de
guarda **NÃO DEVE** alterar `State`, `Generation` ou `FencingToken` do ACB
(transição rejeitada com `AIOS-LIFECYCLE-0002`).

---

## 4. Invariantes da FSM Canônica

> Reproduzidas literalmente de `./_DESIGN_BRIEF.md` §4.3, elaboradas com
> detalhe operacional:

- **INV1** — Uma transição por agente por vez. Garantido por `LeaseManager`
  (`SET NX PX` em Redis + `FencingToken` monotônico persistido no ACB, ver
  §7). Nenhum coordenador **DEVE** efetivar uma transição sem deter a lease
  vigente do `agent_id`.
- **INV2** — Toda transição **DEVE** escrever `LifecycleTransition` **e**
  `LifecycleOutbox` na mesma transação de banco de dados (padrão Outbox).
  Nunca uma sem a outra.
- **INV3** — Estados terminais (`Terminated`, `Failed`) são absorventes:
  nenhuma transição de saída é permitida a partir deles. `IStateMachineEngine`
  **DEVE** rejeitar qualquer `trigger` recebido para um agente já terminal
  com `AIOS-LIFECYCLE-0004`.
- **INV4** — `Hibernated` e `Migrating` **DEVEM** possuir um `Checkpoint` com
  `Consistency = Consistent` válido (hash verificado) **antes** de:
  (a) liberar RAM do runtime de origem (entrada em `Hibernated`); ou
  (b) invalidar/drenar a origem (fase `Cutover` de `Migrating`, T9).
- **INV5** (derivada) — Não existe transição direta de `Created` para
  `Suspended`, `Hibernated`, `Migrating` ou `Terminated`: um agente **DEVE**
  atravessar `Ready`/`Running` antes de qualquer forma de pausa, hibernação
  ou migração — exceto `terminate` (T10), que **PODE** encerrar a partir de
  qualquer estado não-terminal, incluindo `Created`/`Ready`.
- **INV6** (derivada) — Um agente em `Hibernated` **NÃO DEVE** consumir cota
  de `cpu_ms`/`mem_mib` (zero RAM); apenas cotas de armazenamento/retenção
  aplicam-se nesse estado (consistente com I6/N3 do brief).
- **INV7** (derivada) — `wake_count` **DEVE** incrementar em toda transição
  `Hibernated → Ready/Running` (T7); é a métrica-base de
  `aios_lifecycle_agents_total{state="hibernated"}` e de
  `HibernationRecord.LastWakeLatencyMs`.

---

## 5. Ações de Entrada/Saída por Estado

| Estado | Ação de entrada (*on-enter*) | Ação de saída (*on-exit*) |
|--------|-------------------------------|------------------------------|
| `Created` | Registrar ACB no módulo 008 (reflexo de `acb.created` do 006); publicar `agent.lifecycle.created`. | — |
| `Ready` | Registrar `placement_shard`/slot recebido do Scheduler; publicar `agent.lifecycle.ready`. | — |
| `Running` | Atribuir `runtime_instance_id`; incrementar `generation`; restaurar `working_memory_ref`/`context_ref`; publicar `agent.lifecycle.running`. | Cancelar temporizadores de admissão pendentes. |
| `Suspended` | Preservar working memory em RAM/Redis; iniciar temporizador `hibernation.idle_ttl_s`; publicar `agent.lifecycle.suspended`. | Cancelar temporizador de ociosidade. |
| `Hibernated` | Confirmar `last_checkpoint_id` `consistent`; liberar `runtime_instance_id` (→ `null`); registrar `cold_since`; publicar `agent.lifecycle.hibernated`. | — |
| `Migrating` | Criar/associar `MigrationJob`; iniciar temporizador `migration.saga_timeout_ms`; publicar `agent.lifecycle.migrating`. | Cancelar temporizador de saga ao concluir/compensar. |
| `Terminated` | Preencher `terminated_at`; liberar cotas de escopo agente; publicar `agent.lifecycle.terminated`; agendar `TombstoneManager`. | — (terminal) |
| `Failed` | Preencher `terminated_at`/`failure_code`; liberar cotas; publicar `agent.lifecycle.failed`; registrar causa em auditoria (`025`). | — (terminal) |

---

## 6. Sub-máquina: Ciclo de Vida da Saga de Migração (`MigrationJob`)

> Elaboração interna de `MigrationOrchestrator` (R7, FR-007). Não é uma FSM
> do agente — rege as **fases internas** de uma saga de migração enquanto o
> agente permanece em `LifecycleState = Migrating` (T8→T9 da FSM canônica).
> Cada `MigrationJob` é identificado por `migration_id` (ULID).

### 6.1 Fases (`MigrationPhase`)

| Fase | Descrição | Reversível sem perda? |
|------|-----------|------------------------|
| `Quiesce` | Corte de novas operações de escrita no agente na origem; drena syscalls em voo. | sim |
| `Checkpoint` | `CheckpointService.Create` gera snapshot `Consistent` da origem. | sim |
| `Transfer` | Transferência do blob de checkpoint para o storage acessível ao destino (mesmo bucket multi-região ou réplica). | sim |
| `Materialize` | `SpawnManager.Materialize` reconstrói o agente no shard/nó destino a partir do checkpoint transferido. | sim (destino ainda não é autoritativo) |
| `Cutover` | Destino confirmado saudável e sincronizado; origem invalidada; destino passa a ser autoritativo. | **não** (ponto de não-retorno) |
| `Cleanup` | Remoção de recursos remanescentes na origem (runtime, locks, cache local). | sim (idempotente) |

### 6.2 Diagrama ASCII

```
              Start(target)
   ∅ ──────────────────────────▶ ┌──────────┐
                                  │ Quiesce  │
                                  └────┬─────┘
                    drenagem ok        │  timeout/erro
                                       ▼
                                 ┌──────────┐
                                 │Checkpoint │──────────────┐
                                 └────┬─────┘                │
                       checkpoint ok  │                       │
                                       ▼                       │
                                 ┌──────────┐                  │
                                 │ Transfer  │──────────────┐  │
                                 └────┬─────┘                │  │
                     transferência ok │                       │  │
                                       ▼                       │  │
                                 ┌────────────┐                │  │
                                 │ Materialize │─────────────┐ │  │
                                 └─────┬──────┘               │ │  │
              destino saudável e sync  │                       │ │  │
                                       ▼                       │ │  │
                                 ┌──────────┐                  │ │  │
                                 │ Cutover  │ (ponto de não-   │ │  │
                                 └────┬─────┘  retorno)         │ │  │
                            cutover ok │                        │ │  │
                                       ▼                        │ │  │
                                 ┌──────────┐                   │ │  │
                                 │ Cleanup  │                   │ │  │
                                 └────┬─────┘                   │ │  │
                                      ▼                         │ │  │
                              Status=Succeeded (terminal)        │ │  │
                                                                  │ │  │
   Quiesce/Checkpoint/Transfer/Materialize ── falha/timeout ─────┴─┴──┘
                                       │
                                       ▼
                              ┌────────────────┐
                              │  Compensate()   │
                              │ (reativa origem)│
                              └───────┬────────┘
                                      ▼
                          Status=Compensated (terminal)

   (falha após Cutover, sem via de compensação) ──▶ Status=Failed (terminal, raro)
```

### 6.3 Transições e guardas

| De → Para | Gatilho | Guarda |
|-----------|---------|--------|
| `∅ → Quiesce` | `IMigrationOrchestrator.Start(agentId, target)` | Lease adquirida; decisão de placement válida de `009`/`027`; destino reporta saudável. |
| `Quiesce → Checkpoint` | Drenagem de syscalls em voo concluída | `now - quiesce_started < migration.saga_timeout_ms`. |
| `Checkpoint → Transfer` | `CheckpointService.Create` retorna sucesso | `Checkpoint.Consistency = Consistent`. |
| `Transfer → Materialize` | Blob replicado/acessível ao destino | `ContentHash` verificado no destino. |
| `Materialize → Cutover` | `SpawnManager.Materialize` no destino confirma `RuntimeInstanceId` saudável | Destino sincronizado (sem drift de `generation`). |
| `Cutover → Cleanup` | Origem invalidada com sucesso | Nenhuma escrita pendente na origem. |
| `Cleanup → Status=Succeeded` | Recursos de origem liberados | — |
| `{Quiesce,Checkpoint,Transfer,Materialize} → Compensate()` | Falha ou `migration.saga_timeout_ms` excedido | Origem ainda autoritativa (fase < `Cutover`). |
| `Compensate() → Status=Compensated` | Origem reativada com sucesso | Agente retorna ao `LifecycleState` anterior a `Migrating` (T9 aborta, sem cutover). |
| `Cutover falha (raro, sem compensação possível) → Status=Failed` | Falha após ponto de não-retorno sem origem reativável | Aciona T11 (`fail[G9]`) da FSM canônica. |

> **Invariante da sub-máquina:** toda `MigrationJob` **DEVE** alcançar
> exatamente um estado terminal (`Succeeded`, `Compensated` ou, em casos
> raros e pós-`Cutover`, `Failed`) — nenhuma migração permanece `Running`
> indefinidamente; o `ReconciliationController` age como rede de segurança
> ao detectar `migration.saga_timeout_ms` excedido sem avanço de fase (I8 de
> `./ClassDiagrams.md`).

---

## 7. Sub-máquina: Ciclo de Vida da Lease de Coordenação (`Lease`)

> Elaboração interna de `LeaseManager` (R10, FR-010, INV1). Rege o ciclo de
> vida de uma lease individual associada a um `agent_id`, garantindo que no
> máximo um coordenador execute uma transição por vez.

### 7.1 Estados

| Estado | Descrição | Terminal? |
|--------|-----------|-----------|
| `Free` | Nenhum coordenador detém a lease para o `agent_id`. | não |
| `Held` | Lease detida por um `holder`, com `fencing_token` vigente e TTL ativo. | não |
| `Expired` | TTL da lease venceu sem renovação; tratada como `Free` para novos pedidos. | não (reentra em `Free`) |

### 7.2 Diagrama ASCII

```
                    Acquire() — SET NX PX ok
        ┌──────┐ ─────────────────────────────▶ ┌──────┐
        │ Free │                                 │ Held │
        └──────┘ ◀───────────────────────────────└──┬───┘
             ▲          Release() explícito           │
             │                                          │ Renew() antes do TTL
             │                                          ▼
             │                                    ┌──────┐
             └────────────────────────────────────│ Held │ (TTL renovado,
                    TTL expira sem Renew()          └──────┘  mesmo fencing_token)
                    (→ Expired, tratado como Free)
```

### 7.3 Transições e guardas

| De → Para | Gatilho | Guarda |
|-----------|---------|--------|
| `Free → Held` | `ILeaseManager.Acquire(agentId, holder, ttlMs)` | Nenhuma chave `aios:{tenant}:lc:lease:{agent_id}` presente, ou presente e `ExpiresAt < now`. |
| `Held → Held` (renovação) | `ILeaseManager.Renew(lease)` | Chamador é o `holder` atual **e** `fencing_token` confere. |
| `Held → Free` | `ILeaseManager.Release(lease)` | Chamador é o `holder` atual (liberação explícita ao final da transição). |
| `Held → Expired → Free` | `now - lastRenewAt > lease.ttl_ms` sem `Renew()` | Nenhuma (expiração passiva; detectada no próximo `Acquire`). |

> **Invariante da sub-máquina:** o `fencing_token` **DEVE** ser
> monotonicamente crescente a cada novo `Acquire` bem-sucedido sobre o mesmo
> `agent_id`; toda escrita em `IAcbStore.TryUpdate` (`./ClassDiagrams.md`
> §4) **DEVE** apresentar um `fencing_token` igual ou mais recente que o
> persistido no ACB, protegendo contra um coordenador antigo (split-brain)
> que ainda acredite deter a lease — escritas com token obsoleto **DEVEM**
> ser rejeitadas com `AIOS-LIFECYCLE-0013`.

---

## 8. Referências

- Fonte de verdade: `./_DESIGN_BRIEF.md` §4
- Fluxos que disparam estas transições: `./SequenceDiagrams.md`
- Estruturas de dados: `./ClassDiagrams.md`
- Catálogo de eventos emitidos por transição: `./_DESIGN_BRIEF.md` §6.1
  (detalhado em `./Events.md`)
- Catálogo de erros referenciados nas guardas: `./_DESIGN_BRIEF.md` §5.3
  (detalhado em `./API.md`)
- Contratos centrais: `../003-RFC/RFC-0001-Architecture-Baseline.md`

*Fim do documento `StateMachine.md` do módulo 008-Agent-Lifecycle.*
