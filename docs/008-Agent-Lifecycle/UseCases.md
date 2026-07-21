---
Documento: UseCases
Módulo: 008-Agent-Lifecycle
Status: Draft
Versão: 0.1
Última atualização: 2026-07-20
Responsável (RACI-A): Arquiteto do Módulo 008 — Agent Lifecycle
ADRs relacionados: ADR-0080, ADR-0081, ADR-0082, ADR-0083, ADR-0084, ADR-0085, ADR-0086, ADR-0087, ADR-0089
RFCs relacionados: RFC-0001, RFC-0080 (a propor)
Depende de: `_DESIGN_BRIEF.md` (deste módulo), `FunctionalRequirements.md`, `NonFunctionalRequirements.md`, `../006-Kernel/`, `../007-Agent-Runtime/`, `../009-Scheduler/`, `../022-Policy/`, `../027-Cluster/`
---

# 008-Agent-Lifecycle — Use Cases

> Convenção: cada caso de uso é numerado `UC-NNN`, descreve **ator**,
> **pré-condições**, **fluxo principal**, **fluxos alternativos**,
> **exceções** e **pós-condições**. Os códigos de erro citados
> (`AIOS-LIFECYCLE-<NNNN>`) seguem o catálogo do `_DESIGN_BRIEF.md` §5.3 e o
> envelope RFC 7807 da RFC-0001 §5.4. Os gatilhos/transições (`T1`–`T11`) e
> guardas (`G0`–`G10`) seguem `_DESIGN_BRIEF.md` §4.3. A rastreabilidade
> `UC → FR/NFR → Teste` está em `Requirements.md` §5.

## Índice

| UC | Título | Transição(ões) envolvida(s) |
|----|--------|-------------------------------|
| UC-001 | Spawn de agente novo (fluxo feliz) | T1, T2, T3 |
| UC-002 | Spawn com admissão negada pelo Scheduler | T2 (falha) |
| UC-003 | Timeout de materialização de runtime (boot timeout) | T3 (falha) |
| UC-004 | Suspensão voluntária de agente | T4 |
| UC-005 | Preempção (suspensão involuntária pelo Scheduler) | T4 |
| UC-006 | Retomada de agente suspenso (resume quente) | T5 |
| UC-007 | Hibernação por ociosidade (cold agent) | T6 |
| UC-008 | Materialização de cold agent (wake / cold resume) | T7 |
| UC-009 | Checkpoint sob demanda | self-loop (T6/T7 precursor) |
| UC-010 | Restore a partir de checkpoint (recuperação pós-falha de runtime) | T7 (via reconciliação) |
| UC-011 | Migração de agente entre shards/nós (saga feliz) | T8, T9 |
| UC-012 | Migração com falha do destino (compensação) | T8, T9 (compensado) |
| UC-013 | Encerramento normal do agente (terminate) | T10 |
| UC-014 | Falha irrecuperável do agente (fail) | T11 |
| UC-015 | Reconciliação de deriva (runtime morto sem `Terminated`) | T11 / T7 (reconcile) |
| UC-016 | Conflito de lease (concorrência serializada por agente) | qualquer T |
| UC-017 | Negação de política pelo PEP | qualquer T |
| UC-018 | Repetição idempotente de mutação de ciclo de vida | qualquer T |
| UC-019 | Retenção e expurgo de agente terminado (tombstone/LGPD) | (pós-T10/T11) |
| UC-020 | Consulta de histórico de transições (event-sourced) | (somente leitura) |
| UC-021 | Storm de hibernação sob pressão de RAM (backpressure) | T6 (em lote) |
| UC-022 | Migração em massa por drenagem de nó | T8, T9 (em lote) |

---

## UC-001 — Spawn de agente novo (fluxo feliz)

**Ator primário:** `006-Kernel` (em nome de um agente autenticado, via syscall `spawn`) ou operador via API.

**Atores secundários:** `009-Scheduler`, `007-Agent-Runtime`, `022-Policy` (PDP).

**Pré-condições:**
- P1. O ACB foi criado pelo `006-Kernel` e o evento `agent.control.created` foi consumido (T1, estado `Created`).
- P2. O chamador possui capability `lifecycle:spawn` e cabeçalhos de correlação (`traceparent`, `X-AIOS-Tenant`, `Idempotency-Key`) conforme RFC-0001 §5.6.

**Fluxo principal:**
1. O `LifecycleApiSurface` recebe `POST /v1/agents` (ou `Spawn` gRPC), valida schema/versão e verifica a `Idempotency-Key`.
2. O `LifecyclePolicyEnforcer` monta o `DecisionRequest` e consulta o PDP (`022-Policy`); decisão = `allow` (**G0**).
3. O `LifecycleCoordinator` adquire o lease do agente via `LeaseManager` (fencing token emitido).
4. O `StateMachineEngine` valida a transição `Created → Ready` (**G1**) e o `SpawnManager` solicita admissão ao `009-Scheduler`.
5. O Scheduler responde com slot aprovado; o ACB transita para `Ready` (T2); evento `agent.lifecycle.ready` publicado via Outbox.
6. O `SpawnManager` verifica o `WarmPoolManager`: se houver runtime pré-aquecido no shard, faz *attach*; senão, solicita boot completo ao `007-Agent-Runtime`.
7. O `StateMachineEngine` valida a transição `Ready → Running` (**G3**: slot alocado, runtime provisionado, lease válido) e incrementa `generation`.
8. O ACB transita para `Running`; o `LifecycleEventPublisher` publica `agent.lifecycle.running`.
9. O `LeaseManager` libera o lease ao final da transição.
10. O `LifecycleApiSurface` retorna `202 Accepted` com o ACB serializado (URN, estado, `generation`).
11. O `LifecycleTelemetry` registra span, métrica `aios_lifecycle_spawn_duration_ms` e entrada em `LifecycleTransition` para cada aresta percorrida.

**Fluxos alternativos:**
- FA1 (cold start sem WarmPool disponível): o `SpawnManager` solicita boot completo ao `007-Agent-Runtime`; latência tende ao limite superior de NFR-001 (`p99 ≤ 250 ms`), mas ainda dentro do SLO.
- FA2 (spawn de agente previamente `Hibernated` reaproveitando o mesmo `agent_id`): tratado como **UC-008** (wake), não como este caso de uso.

**Exceções:**
- E1. Política nega (`allow` não concedido) → ver **UC-017**.
- E2. Lease não adquirido (outra transição em curso) → ver **UC-016**.
- E3. Admissão rejeitada/timeout do Scheduler → ver **UC-002**.
- E4. Timeout de materialização do runtime → ver **UC-003**.
- E5. `Idempotency-Key` já usada com payload idêntico → retorna a resposta original (mesmo HTTP status e corpo), sem novo efeito colateral (ver **UC-018**).

**Pós-condições:**
- Q1. ACB em `Running`, com `runtime_instance_id` não-nulo e `generation` incrementado.
- Q2. Três eventos de lifecycle (`ready`, `running`) publicados e auditados via Outbox.
- Q3. `LifecycleTransition` contém uma entrada por aresta percorrida (`seq` monotônico).

---

## UC-002 — Spawn com admissão negada pelo Scheduler

**Ator primário:** `006-Kernel`, em nome do agente.

**Atores secundários:** `009-Scheduler`.

**Pré-condições:** Mesmas de UC-001 até o passo 3 (lease adquirido, ACB em `Created`).

**Fluxo principal (falha):**
1–3. Idênticos a UC-001.
4. O `SpawnManager` solicita admissão ao `009-Scheduler`.
5. O Scheduler responde com rejeição explícita (cota/capacidade insuficiente) **ou** não responde dentro de `spawn.timeout_ms`.
6. O `LifecycleCoordinator` avalia a guarda de falha (**G9**: erro irrecuperável de admissão) e transita o ACB de `Created` para `Failed` (T11).
7. O `LifecycleEventPublisher` publica `agent.lifecycle.failed` com `reason=admission_rejected` ou `admission_timeout`.
8. O `LeaseManager` libera o lease.
9. O `LifecycleApiSurface` retorna `AIOS-LIFECYCLE-0005` (HTTP 429, `retriable=true`) ao chamador.

**Fluxos alternativos:**
- FA1. Chamador reenvia `spawn` com a mesma `Idempotency-Key`: o resultado original (falha) já foi persistido; o mesmo erro é retornado sem nova tentativa de admissão (ver **UC-018**).
- FA2. Chamador reenvia com **nova** `Idempotency-Key`: tratado como novo spawn (UC-001), sujeito a um novo `agent_id` (o `agent_id` anterior permanece `Failed` — estado terminal absorvente, INV3).

**Exceções:**
- E1. Scheduler retorna erro transitório distinto de rejeição/timeout (ex.: 5xx inesperado): mapeado para `AIOS-LIFECYCLE-0005` também, preservando `retriable=true`.

**Pós-condições:**
- Q1. ACB em estado terminal `Failed`, com `terminated_at` preenchido e `failure_code=AIOS-LIFECYCLE-0005`.
- Q2. Evento `agent.lifecycle.failed` publicado e auditado.
- Q3. Nenhum runtime foi provisionado (invariante: `state` terminal ⇒ `runtime_instance_id = null`).

---

## UC-003 — Timeout de materialização de runtime (boot timeout)

**Ator primário:** `007-Agent-Runtime` (indiretamente, via ausência de confirmação).

**Pré-condições:** ACB em `Ready` (slot do Scheduler já reservado, T2 concluída).

**Fluxo principal (falha):**
1. O `SpawnManager` solicita ao `007-Agent-Runtime` a materialização do runtime (boot completo ou *attach* ao WarmPool).
2. O runtime não confirma início dentro de `spawn.timeout_ms`.
3. O `LifecycleCoordinator` avalia **G9** e executa compensação: solicita ao `009-Scheduler` a liberação do slot reservado.
4. O ACB transita de `Ready` para `Failed` (T11).
5. O `LifecycleEventPublisher` publica `agent.lifecycle.failed` com `reason=boot_timeout`.
6. O `LifecycleApiSurface` retorna `AIOS-LIFECYCLE-0006` (HTTP 504, `retriable=true`).

**Fluxos alternativos:**
- FA1. O runtime responde com falha explícita (não apenas timeout) antes do prazo: mesmo tratamento, `reason=boot_failed`.

**Exceções:**
- E1. Falha ao liberar o slot do Scheduler durante a compensação: registrada como incidente operacional (`Monitoring.md`); o ACB ainda transita para `Failed`; o `ReconciliationController` do Scheduler eventualmente expira o slot órfão.

**Pós-condições:**
- Q1. ACB em `Failed`, `runtime_instance_id` permanece nulo.
- Q2. Slot do Scheduler liberado (ou marcado para expiração).
- Q3. Evento `agent.lifecycle.failed` publicado e auditado.

---

## UC-004 — Suspensão voluntária de agente

**Ator primário:** `006-Kernel`, em nome do agente ou de seu operador.

**Pré-condições:**
- P1. ACB em `Running`.
- P2. Chamador possui capability `lifecycle:suspend`.
- P3. Ausência de operação crítica não-preemptível em curso (guarda **G4**).

**Fluxo principal:**
1. Chamador envia `POST /v1/agents/{id}/suspend` com `Idempotency-Key`.
2. `LifecyclePolicyEnforcer` consulta o PDP → `allow`.
3. `LifecycleCoordinator` adquire o lease; `StateMachineEngine` valida **G4** e transita o ACB para `Suspended` (T4).
4. Working memory é preservada em RAM/Redis (nenhuma serialização durável ainda ocorre neste ponto).
5. `LifecycleEventPublisher` publica `agent.lifecycle.suspended`.
6. `LifecycleApiSurface` retorna `202 Accepted` com o ACB atualizado.

**Fluxos alternativos:**
- FA1. Agente em seção crítica não-preemptível: o Kernel aguarda até um limite curto configurável, ou retorna `AIOS-LIFECYCLE-0002` (guarda não satisfeita) se o tempo se esgotar.

**Exceções:**
- E1. ACB não está em `Running` (ex.: já `Suspended`/`Terminated`) → `AIOS-LIFECYCLE-0002`, exceto se for repetição idempotente exata (UC-018).
- E2. Lease não adquirido → UC-016.
- E3. Política nega → UC-017.

**Pós-condições:**
- Q1. ACB em `Suspended`; `working_memory_ref` preservado.
- Q2. Evento `agent.lifecycle.suspended` publicado e auditado.

---

## UC-005 — Preempção (suspensão involuntária pelo Scheduler)

**Ator primário:** `009-Scheduler` (via evento).

**Pré-condições:** ACB em `Running`; Scheduler decidiu preemptar em favor de agente de maior prioridade.

**Fluxo principal:**
1. O `LifecycleEventConsumer` consome `aios.<tenant>.scheduler.preemption.requested`.
2. O `LifecycleCoordinator` trata o evento como gatilho equivalente a `suspend` (T4), sem exigir nova consulta ao PDP (a decisão de preempção já é uma decisão de política de escalonamento autorizada pelo Scheduler; o 008 apenas aplica).
3. O ACB transita para `Suspended`.
4. O `LifecycleEventPublisher` publica `agent.lifecycle.suspended` com `labels.reason=preemption`.
5. O `LifecycleTelemetry` registra a suspensão com `actor=scheduler` (não uma mutação externa via API).

**Fluxos alternativos:**
- FA1. Agente já `Suspended`/`Terminated` quando o evento de preempção chega: evento é ignorado de forma idempotente (dedupe por `event.id`), sem erro.

**Exceções:**
- E1. Falha ao processar o evento (ex.: ACB não encontrado): registrada em log de erro; evento é reencaminhado para DLQ conforme política do `020-Communication`.

**Pós-condições:**
- Q1. ACB em `Suspended` com `labels.reason=preemption`.
- Q2. Evento de lifecycle publicado e auditado.

---

## UC-006 — Retomada de agente suspenso (resume quente)

**Ator primário:** `006-Kernel` ou o próprio `009-Scheduler` (ao liberar capacidade).

**Pré-condições:**
- P1. ACB em `Suspended`.
- P2. Chamador possui capability `lifecycle:resume`.

**Fluxo principal:**
1. Chamador envia `POST /v1/agents/{id}/resume` com `Idempotency-Key`.
2. `LifecyclePolicyEnforcer` consulta o PDP → `allow`.
3. `LifecycleCoordinator` solicita ao `009-Scheduler`, via `SpawnManager`, a re-reserva de slot.
4. Scheduler confirma slot disponível; `StateMachineEngine` valida **G5** (slot disponível, working memory íntegra, política `run` ainda válida) e transita o ACB para `Running` (T5).
5. `LifecycleEventPublisher` publica `agent.lifecycle.running` (reentrada).
6. `LifecycleApiSurface` retorna `202 Accepted`.

**Fluxos alternativos:**
- FA1. Slot não disponível imediatamente: mesma política de timeout de admissão de UC-002; se esgotado, retorna `AIOS-LIFECYCLE-0005` mantendo o ACB em `Suspended` (não falha o agente, apenas a tentativa de resume).

**Exceções:**
- E1. ACB não está em `Suspended` → tratado como idempotente se repetição exata (UC-018), senão `AIOS-LIFECYCLE-0002`.
- E2. Política nega → UC-017.

**Pós-condições:**
- Q1. ACB em `Running` com `working_memory_ref` preservado.
- Q2. Latência medida por `aios_lifecycle_transition_duration_ms{trigger="resume"}` (NFR-002).

---

## UC-007 — Hibernação por ociosidade (cold agent)

**Ator primário:** `HibernationController` (processo interno, varredura periódica), sem chamador externo direto.

**Pré-condições:**
- P1. ACB em `Suspended`.
- P2. `now - last_active > hibernation.idle_ttl_s` **ou** `hibernation.ram_pressure_threshold_pct` excedido.

**Fluxo principal:**
1. O `HibernationController` identifica ACBs elegíveis (idle além do TTL ou sob pressão de RAM).
2. O `HibernationController` adquire o lease do agente e invoca o `CheckpointService` para garantir um checkpoint `consistent` recente (cria um novo se necessário).
3. Confirmado o checkpoint, o `HibernationController` solicita ao `007-Agent-Runtime` a liberação dos recursos quentes (RAM).
4. O `StateMachineEngine` valida **G6** (idle/pressão + checkpoint `consistent` gravado) e transita o ACB para `Hibernated` (T6); `last_checkpoint_id` atualizado; `cold_since` preenchido.
5. O `LifecycleEventPublisher` publica `agent.lifecycle.hibernated`.

**Fluxos alternativos:**
- FA1. Checkpoint já existente e válido (criado há pouco tempo, dentro de `checkpoint.interval_s`): reaproveitado, sem novo snapshot.

**Exceções:**
- E1. Falha ao criar o checkpoint (MinIO/PG indisponível): hibernação é **adiada** (ACB permanece `Suspended`); erro `AIOS-LIFECYCLE-0014` registrado; nova tentativa na próxima varredura (degradação graciosa, §9 do brief).

**Pós-condições:**
- Q1. ACB em `Hibernated`, RSS do runtime `= 0` (NFR-011), `last_checkpoint_id` válido.
- Q2. Evento `agent.lifecycle.hibernated` publicado e auditado.

---

## UC-008 — Materialização de cold agent (wake / cold resume)

**Ator primário:** `006-Kernel`, disparado por nova demanda de trabalho para o agente.

**Pré-condições:**
- P1. ACB em `Hibernated` com `last_checkpoint_id` válido.
- P2. Chamador possui capability `lifecycle:wake`.

**Fluxo principal:**
1. Chamador envia `POST /v1/agents/{id}/wake`.
2. `LifecyclePolicyEnforcer` consulta o PDP → `allow`.
3. `LifecycleCoordinator` adquire o lease; `CheckpointService` verifica a integridade (`content_hash`) do checkpoint referenciado.
4. `SpawnManager` solicita admissão (placement) ao `009-Scheduler` — a materialização sob demanda equivale a um novo *placement*.
5. `StateMachineEngine` valida **G5** (checkpoint íntegro, slot concedido) e transita o ACB para `Ready`/`Running` (T7), incrementando `generation` e `wake_count`.
6. O `CheckpointService` restaura ACB + working memory a partir do checkpoint; ponteiros `working_memory_ref`/`context_ref` reatribuídos.
7. `LifecycleEventPublisher` publica `agent.lifecycle.woken`.
8. `LifecycleApiSurface` retorna `202 Accepted`.

**Fluxos alternativos:**
- FA1. Nenhum — a restauração é sempre a partir do `last_checkpoint_id` válido mais recente.

**Exceções:**
- E1. Checkpoint corrompido (`content_hash` divergente) → `AIOS-LIFECYCLE-0007` (HTTP 422); ACB permanece `Hibernated`; se existir checkpoint anterior válido, o sistema pode tentá-lo conforme política operacional; se nenhum, ver **UC-014** (fail).
- E2. Admissão negada/timeout do Scheduler: mesmo tratamento de UC-002, porém o ACB permanece `Hibernated` (não vai a `Failed` diretamente, pois o agente não estava em execução ativa) — nova tentativa de `wake` é possível.

**Pós-condições:**
- Q1. ACB em `Running` com estado cognitivo equivalente ao momento do checkpoint; `generation` e `wake_count` incrementados.
- Q2. Latência de materialização dentro da meta de NFR-001 (`p99 ≤ 250 ms`).

---

## UC-009 — Checkpoint sob demanda

**Ator primário:** `006-Kernel` (a pedido do agente/operador) ou processo interno (`HibernationController`, `checkpoint.interval_s`).

**Pré-condições:**
- P1. ACB em `Running` ou `Suspended`.
- P2. Chamador possui capability `lifecycle:checkpoint`.

**Fluxo principal:**
1. Chamador (ou processo interno) envia `POST /v1/agents/{id}/checkpoint` com `Idempotency-Key`.
2. `LifecyclePolicyEnforcer` consulta o PDP → `allow` (dispensável para gatilho interno periódico, conforme configuração).
3. `CheckpointService` coleta os ponteiros atuais (`working_memory_ref`, `context_ref`) do ACB.
4. `CheckpointService` serializa (codec `checkpoint.codec`), cifra (`checkpoint.encryption`) e persiste o blob via `SnapshotStore` (MinIO) + metadados (PostgreSQL), calculando `content_hash`.
5. O ACB é atualizado com o novo `last_checkpoint_id` (operação atômica; não constitui uma transição de estado, é uma atualização de metadado dentro do estado corrente).
6. `LifecycleEventPublisher` publica `agent.lifecycle.checkpointed`.
7. `LifecycleApiSurface` retorna `201 Created` com o `checkpoint_id`.

**Fluxos alternativos:**
- FA1. Checkpoint disparado internamente pela hibernação (UC-007) ou pela política periódica: mesmo fluxo, sem chamador externo; auditado com `actor=hibernation-controller`/`checkpoint-scheduler`.

**Exceções:**
- E1. Falha ao persistir o snapshot (MinIO/PG indisponível) → `AIOS-LIFECYCLE-0014` (HTTP 503, `retriable=true`); `last_checkpoint_id` anterior permanece válido (atomicidade da atualização do ponteiro — sem estado parcial).
- E2. Falha de serialização/deserialização → `AIOS-LIFECYCLE-0011` (HTTP 500).
- E3. Política nega → UC-017.

**Pós-condições:**
- Q1. `last_checkpoint_id` do ACB aponta para um snapshot íntegro (`consistency=consistent`) e recuperável.
- Q2. Evento `agent.lifecycle.checkpointed` publicado e auditado.

---

## UC-010 — Restore a partir de checkpoint (recuperação pós-falha de runtime)

**Ator primário:** `ReconciliationController` (gatilho interno) ou operador via API (`restore` explícito com `checkpointId`).

**Pré-condições:**
- P1. ACB em `Running` sem heartbeat do `007-Agent-Runtime` além de `reconcile.runtime_dead_after_ms` (caso interno) **ou** chamador solicita `restore` explícito com um `checkpointId` específico.
- P2. Existe ao menos um `Checkpoint` com `consistency ∈ {consistent, crash}` associado ao agente.

**Fluxo principal:**
1. O `ReconciliationController` detecta ausência de heartbeat e sinaliza o `LifecycleCoordinator` (ou o chamador invoca `POST /v1/agents/{id}/restore`).
2. `LifecycleCoordinator` adquire o lease; `CheckpointService` seleciona o checkpoint alvo (o mais recente `consistent`, salvo indicação explícita de `checkpointId`).
3. `CheckpointService` verifica `content_hash`; se íntegro, restaura ACB + working memory.
4. `SpawnManager` solicita novo slot ao `009-Scheduler` e novo runtime ao `007-Agent-Runtime`.
5. `StateMachineEngine` valida **G5** e transita o ACB para `Ready`/`Running` (T7), incrementando `generation`.
6. `LifecycleEventPublisher` publica `agent.lifecycle.restored` e, na sequência, `agent.lifecycle.running`.
7. `LifecycleApiSurface`/`LifecycleTelemetry` registram o evento com `reason=runtime_lost` (caso interno) ou `reason=explicit_restore`.

**Fluxos alternativos:**
- FA1. `restore` explícito para um `checkpointId` anterior ao mais recente (rollback intencional, ex.: para depuração): permitido via API, desde que o checkpoint pertença ao mesmo `agent_id`/`tenant_id`.

**Exceções:**
- E1. Checkpoint corrompido/inacessível → `AIOS-LIFECYCLE-0007`; se nenhum checkpoint válido restar → transição para `Failed` (T11) com `failure_code=AIOS-LIFECYCLE-0007` (ver **UC-014**).
- E2. Checkpoint inexistente para o `checkpointId` informado → `AIOS-LIFECYCLE-0008` (HTTP 404).
- E3. Tempo total de recuperação excede `recovery.rto_budget_s` → incidente operacional escalado (`Monitoring.md`), sem violar a garantia de eventual convergência.

**Pós-condições:**
- Q1. ACB em `Running` com estado equivalente ao do checkpoint restaurado; `generation` incrementado.
- Q2. RTO observado registrado em `aios_lifecycle_migration_duration_ms`/métrica de recuperação equivalente, verificado contra NFR-005 (`RTO ≤ 15 min`).

---

## UC-011 — Migração de agente entre shards/nós (saga feliz)

**Ator primário:** `009-Scheduler`/`027-Cluster` (decisão de placement), executada pelo `MigrationOrchestrator`.

**Pré-condições:**
- P1. ACB em `Running` ou `Suspended`.
- P2. Decisão de placement de destino recebida (`aios.<tenant>.scheduler.placement.decided` ou `cluster.node.draining`).
- P3. Destino saudável (health check aprovado).

**Fluxo principal:**
1. O `LifecycleEventConsumer` consome a decisão de placement; o `MigrationOrchestrator` cria um `MigrationJob` (`phase=quiesce`).
2. `LifecycleCoordinator` adquire o lease; `StateMachineEngine` valida **G7** (decisão de placement válida, lease adquirido, destino saudável) e transita o ACB para `Migrating` (T8).
3. Fase `quiesce`: o agente para de aceitar novo trabalho na origem (drenagem suave).
4. Fase `checkpoint`: `CheckpointService` cria um checkpoint `consistent` de ACB + working memory.
5. Fase `transfer`: o blob do checkpoint é replicado/tornado acessível ao destino (mesma região de MinIO ou replicação cross-region conforme `027-Cluster`).
6. Fase `materialize`: `SpawnManager` materializa o agente no nó/shard de destino a partir do checkpoint transferido.
7. Fase `cutover`: `StateMachineEngine` valida **G8** (materialização no destino confirmada) e transita o ACB para o estado apropriado no destino (`Ready`/`Running`/`Hibernated`, T9), atualizando `placement_shard`/`placement_node`.
8. Fase `cleanup`: a origem é invalidada (runtime antigo encerrado, lease liberado).
9. `LifecycleEventPublisher` publica `agent.lifecycle.migrating` (no início) e `agent.lifecycle.migrated` (ao final).
10. `MigrationJob.status = succeeded`.

**Fluxos alternativos:**
- FA1. Agente `Suspended` (sem runtime ativo) migrando: a fase `quiesce` é trivial (nenhum trabalho em curso); as demais fases seguem normalmente, terminando em `Suspended` no destino.
- FA2. Migração para `Hibernated` no destino (ex.: destino é um shard de baixa prioridade): fase `cutover` transita para `Hibernated` em vez de `Ready`/`Running`.

**Exceções:**
- E1. Destino se torna insalubre durante a saga → ver **UC-012**.
- E2. `migration.saga_timeout_ms` excedido em qualquer fase → compensação (ver **UC-012**).

**Pós-condições:**
- Q1. ACB materializado no destino (`placement_shard`/`placement_node` atualizados), origem invalidada.
- Q2. `MigrationJob.status = succeeded`; latência total registrada em `aios_lifecycle_migration_duration_ms` (NFR-010).
- Q3. Eventos `migrating` e `migrated` publicados e auditados.

---

## UC-012 — Migração com falha do destino (compensação)

**Ator primário:** `MigrationOrchestrator`, reagindo à indisponibilidade do destino.

**Pré-condições:** `MigrationJob` em andamento (fase `transfer` ou `materialize`); ACB em `Migrating`.

**Fluxo principal (falha):**
1. O destino falha ao confirmar a materialização dentro de `migration.saga_timeout_ms`, ou um *health check* prévio à fase `materialize` reporta o destino insalubre.
2. O `MigrationOrchestrator` avalia **G9** (erro irrecuperável da saga) e inicia a compensação: invalida qualquer estado parcial criado no destino, sem afetar a origem (que ainda não foi limpa nesta fase).
3. O `StateMachineEngine` transita o ACB de volta ao estado anterior à migração (`Running`/`Suspended`/`Hibernated`, reativando a origem) — não é uma transição `T11` automática, pois a falha é do destino, não do agente.
4. `MigrationJob.status = compensated`; `phase` registrado no ponto de falha para diagnóstico.
5. `LifecycleEventPublisher` publica `agent.lifecycle.failed` com `reason=migration_target_unavailable` **apenas se** a origem também não puder ser reativada (caso raro); caso contrário, nenhum evento de falha do agente é emitido — apenas telemetria operacional da migração.
6. `LifecycleApiSurface` retorna, para uma chamada síncrona de acompanhamento, `AIOS-LIFECYCLE-0010` (HTTP 502, `retriable=true`).

**Fluxos alternativos:**
- FA1. Falha detectada ainda na fase `quiesce`/`checkpoint` (antes de qualquer transferência): compensação trivial, ACB retorna ao estado anterior sem qualquer limpeza de destino necessária.

**Exceções:**
- E1. Origem também se tornou indisponível durante a tentativa de compensação (cenário de dupla falha): ACB transita para `Failed` (T11) com `failure_code=AIOS-LIFECYCLE-0010`; escalado como incidente crítico.

**Pós-condições:**
- Q1. Agente **nunca perde trabalho aceito** — a origem permanece autoritativa até o `cutover` ser confirmado (invariante de design da saga, brief §1.1 R7).
- Q2. `MigrationJob.status = compensated`, disponível para diagnóstico e nova tentativa futura.

---

## UC-013 — Encerramento normal do agente (terminate)

**Ator primário:** `006-Kernel`, operador humano ou o próprio sistema (encerramento administrativo).

**Pré-condições:**
- P1. ACB em qualquer estado não-terminal (`Created`, `Ready`, `Running`, `Suspended`, `Hibernated`, `Migrating`).
- P2. Chamador possui capability `lifecycle:terminate`.

**Fluxo principal:**
1. Chamador envia `DELETE /v1/agents/{id}` com `Idempotency-Key`.
2. `LifecyclePolicyEnforcer` consulta o PDP → `allow`.
3. `LifecycleCoordinator` adquire o lease; `StateMachineEngine` valida **G10** (efeitos colaterais drenados, checkpoint final opcional) e transita o ACB para `Terminated` (T10).
4. Se `Running`/`Suspended`, o runtime é encerrado graciosamente via `007-Agent-Runtime`; se `Hibernated`, nenhum runtime precisa ser parado.
5. `terminated_at` é preenchido; `TombstoneManager` é notificado para iniciar a contagem de retenção.
6. `LifecycleEventPublisher` publica `agent.lifecycle.terminated`.
7. `LifecycleApiSurface` retorna `202 Accepted`.

**Fluxos alternativos:**
- FA1. Encerramento a partir de `Migrating`: aguarda a conclusão segura da fase corrente da saga (ou aciona compensação, UC-012) antes de terminar, para não deixar o agente materializado em dois lugares simultaneamente.
- FA2. Encerramento com checkpoint final solicitado explicitamente: `CheckpointService` cria um último checkpoint antes da transição, preservando o estado para fins de auditoria/investigação (não para retomada, já que `Terminated` é absorvente).

**Exceções:**
- E1. ACB já em `Terminated`/`Failed` (terminal): tratado como idempotente (UC-018) se houver `Idempotency-Key` correspondente; caso contrário, retorna o estado atual sem erro adicional além de `AIOS-LIFECYCLE-0004` se a intenção explícita era mutar um estado terminal.
- E2. Política nega → UC-017.

**Pós-condições:**
- Q1. ACB em `Terminated` (terminal), retido para auditoria conforme `retention.terminated_ttl_days`.
- Q2. Evento `agent.lifecycle.terminated` publicado exatamente uma vez.
- Q3. `desired_state = Terminated` (invariante: estado terminal ⇒ `desired_state` terminal).

---

## UC-014 — Falha irrecuperável do agente (fail)

**Ator primário:** `LifecycleCoordinator`/`ReconciliationController`, reagindo a um erro que nenhuma compensação resolve.

**Pré-condições:** ACB em qualquer estado não-terminal; condição de falha irrecuperável identificada (timeout de saga esgotado além de tentativas, corrupção de checkpoint sem alternativa, runtime perdido além de `max_recovery`).

**Fluxo principal:**
1. O componente detector (varia por cenário: `SpawnManager`, `CheckpointService`, `MigrationOrchestrator` ou `ReconciliationController`) identifica que a condição de falha atende à guarda **G9**.
2. `LifecycleCoordinator` adquire o lease (ou já o detém, se a falha ocorre durante uma saga em andamento) e transita o ACB para `Failed` (T11) a partir de qualquer estado não-terminal.
3. `failure_code` é preenchido com o `AIOS-LIFECYCLE-<NNNN>` correspondente à causa raiz.
4. `terminated_at` é preenchido.
5. `LifecycleEventPublisher` publica `agent.lifecycle.failed` com o `reason` detalhado.
6. `TombstoneManager` é notificado para iniciar a contagem de retenção (mesmo tratamento de agentes `Terminated`).

**Fluxos alternativos:**
- FA1. Falha detectada durante `Hibernated` (checkpoint corrompido sem alternativa, UC-008/E1): mesmo tratamento, a partir de `Hibernated`.
- FA2. Falha detectada durante `Migrating` sem possibilidade de reativar origem nem destino (UC-012/E1): mesmo tratamento, com `failure_code=AIOS-LIFECYCLE-0010`.

**Exceções:**
- E1. Nenhuma — este é, ele próprio, um caminho terminal bem-definido de outros casos de uso.

**Pós-condições:**
- Q1. ACB em `Failed` (terminal, absorvente — INV3); `runtime_instance_id = null`.
- Q2. Evento `agent.lifecycle.failed` publicado e auditado, com `failure_code` rastreável.
- Q3. Operador/SRE pode investigar via `GET /v1/agents/{id}/lifecycle` (histórico completo de transições, UC-020).

---

## UC-015 — Reconciliação de deriva (runtime morto sem `Terminated`)

**Ator primário:** `ReconciliationController` (loop periódico, `reconcile.interval_ms`).

**Pré-condições:** ACB em `Running` (`desired_state = Running`) sem heartbeat do `007-Agent-Runtime` recebido há mais de `reconcile.runtime_dead_after_ms`.

**Fluxo principal:**
1. O `ReconciliationController` varre periodicamente os ACBs com `desired_state ≠ state` ou com sinais de deriva (última heartbeat antiga).
2. Detecta que o `agent_id` não emite heartbeat além do limiar configurado.
3. Aciona o fluxo de recuperação equivalente a **UC-010** (restore a partir do último checkpoint válido) para tentar reconciliar o estado observado ao estado desejado.
4. Se a recuperação for bem-sucedida dentro de `recovery.rto_budget_s`, o ACB converge para `Ready`/`Running` (via `wake`/`restore`).
5. Se a recuperação falhar ou exceder o orçamento de RTO, o ACB é encaminhado ao fluxo de **UC-014** (`Failed`).

**Fluxos alternativos:**
- FA1. Deriva detectada em `Migrating` travada além de `migration.saga_timeout_ms`: reconciliação aciona a compensação de **UC-012** em vez do restore de UC-010.
- FA2. Deriva detectada em `Hibernated` com checkpoint órfão (referência a snapshot inexistente): reconciliação sinaliza `AIOS-LIFECYCLE-0008` e escalona para investigação, sem transição automática (evita agravar a corrupção).

**Exceções:**
- E1. Múltiplos ACBs elegíveis simultaneamente (ex.: após queda de um nó inteiro): reconciliação aplica *backpressure* — não dispara todas as recuperações de uma vez, respeitando `migration.max_concurrent_per_node` e a admissão do `009-Scheduler` (ver **UC-021**/**UC-022** para o caso em lote).

**Pós-condições:**
- Q1. Nenhum agente permanece indefinidamente com `state ≠ desired_state` além de `reconcile.interval_ms` × tentativas configuradas.
- Q2. Toda ação de reconciliação é registrada em `LifecycleTransition` com `actor=reconciler`.

---

## UC-016 — Conflito de lease (concorrência serializada por agente)

**Ator primário:** Duas (ou mais) requisições concorrentes de mutação sobre o **mesmo** `agent_id` (ex.: `suspend` e `checkpoint` quase simultâneos).

**Pré-condições:** ACB existente; nenhuma requisição ainda detém o lease do agente.

**Fluxo principal:**
1. Requisição A chega ao `LifecycleCoordinator` e adquire o lease via `LeaseManager` (`SET NX PX` no Redis), recebendo um `fencing_token`.
2. Requisição B chega quase simultaneamente e tenta adquirir o mesmo lease; a operação `SET NX` falha (chave já existe).
3. O `LifecycleCoordinator` rejeita B imediatamente com `AIOS-LIFECYCLE-0003` (HTTP 409, `retriable=true`), sem enfileirar nem bloquear.
4. Requisição A prossegue sua transição normalmente e libera o lease ao final (ou o lease expira por TTL, `lease.ttl_ms`, se A travar).
5. O chamador de B pode reenviar a requisição após um curto backoff.

**Fluxos alternativos:**
- FA1. Requisição A trava (crash do coordenador) sem liberar o lease: o lease expira por TTL (`lease.ttl_ms`); um novo coordenador pode adquiri-lo com um `fencing_token` mais alto; qualquer escrita tardia do coordenador travado com o token antigo é rejeitada com `AIOS-LIFECYCLE-0013`.
- FA2. B reenvia com a mesma `Idempotency-Key` de uma tentativa anterior que já teve sucesso (não a que colidiu): tratado como repetição idempotente (UC-018), não como novo conflito.

**Exceções:**
- E1. Split-brain (dois coordenadores acreditam deter o lease simultaneamente, por falha de rede/particionamento): a escrita com `fencing_token` mais antigo é rejeitada por `AcbStore` com `AIOS-LIFECYCLE-0013`; apenas o token mais novo persiste (INV1).

**Pós-condições:**
- Q1. No máximo uma transição por agente é aplicada por vez (INV1); nenhuma escrita perdida (*lost update*).
- Q2. Taxa de conflitos observável via métrica de rejeições `AIOS-LIFECYCLE-0003`/`0013`.

---

## UC-017 — Negação de política pelo PEP

**Ator primário:** Qualquer chamador de uma ação de ciclo de vida sem a capability necessária.

**Pré-condições:** Chamador autenticado (claims OIDC válidas), mas sem a capability requerida pela ação (`lifecycle:spawn`, `:suspend`, `:resume`, `:hibernate`, `:wake`, `:migrate`, `:checkpoint`, `:restore`, `:terminate`).

**Fluxo principal:**
1. Chamador envia qualquer mutação de ciclo de vida.
2. `LifecycleApiSurface` despacha ao `LifecyclePolicyEnforcer`.
3. `LifecyclePolicyEnforcer` monta o `DecisionRequest` e consulta o PDP (`022-Policy`) via gRPC.
4. PDP retorna `deny`.
5. O módulo **não executa** nenhum efeito colateral: nenhum lease é adquirido, nenhuma transição é avaliada, nenhuma cota (referência) é consumida.
6. `LifecycleApiSurface` retorna `AIOS-LIFECYCLE-0012` (HTTP 403).
7. `LifecycleEventPublisher` publica um evento de auditoria de negação (via `025-Audit`, não um evento de lifecycle propriamente dito, já que nenhuma transição ocorreu).

**Fluxos alternativos:**
- FA1. PDP indisponível: postura *default deny* aplicada de forma equivalente (§9 do brief — degradação graciosa mantém agentes quentes, mas não admite **novas** mutações sem decisão válida).

**Exceções:**
- E1. Nenhuma — a negação **é** o caminho de exceção de outros casos de uso, mas é ela mesma bem-definida e não-excepcional do ponto de vista do PEP.

**Pós-condições:**
- Q1. Nenhum efeito colateral no ACB, lease ou eventos de sucesso.
- Q2. Negação auditada com o sujeito, a ação e o `agent_id` alvo (NFR-016).

---

## UC-018 — Repetição idempotente de mutação de ciclo de vida

**Ator primário:** `006-Kernel`/chamador, reenviando uma requisição após timeout de rede ou retry automático de cliente.

**Pré-condições:** Uma mutação de ciclo de vida foi previamente processada com uma `Idempotency-Key` específica, dentro da janela de retenção (≥ 24h, RFC-0001 §5.5).

**Fluxo principal (payload idêntico):**
1. Chamador reenvia a mesma requisição (mesma ação, mesmo payload, mesma `Idempotency-Key`).
2. `LifecycleApiSurface` consulta o registro de idempotência **antes** de qualquer outro processamento (antes até da consulta ao PDP).
3. Resultado previamente persistido é encontrado e retornado **byte-a-byte idêntico** (mesmo HTTP status, mesmo corpo).
4. Nenhuma nova consulta ao PDP, nenhum novo lease, nenhuma nova transição, nenhum novo evento é publicado.

**Fluxos alternativos:**
- FA1. Repetição de uma mutação que originalmente falhou (ex.: UC-002, UC-003): o mesmo erro é retornado novamente, de forma consistente.

**Exceções:**
- E1. Mesma `Idempotency-Key`, payload **divergente**: rejeitado (conflito de idempotência, mapeado a um erro de contrato de API — ver `API.md`); resultado original não é alterado.
- E2. `Idempotency-Key` expirada (fora da janela de retenção) e reenviada: tratada como nova invocação (novo processamento completo).

**Pós-condições:**
- Q1. Efeito único garantido: para qualquer número de repetições dentro da janela de retenção com payload idêntico, o estado do sistema é o mesmo que após a primeira execução bem-sucedida (NFR-013).

---

## UC-019 — Retenção e expurgo de agente terminado (tombstone/LGPD)

**Ator primário:** `TombstoneManager` (processo periódico) ou operador/DPO via solicitação de exercício do direito ao esquecimento.

**Pré-condições:**
- P1. ACB em `Terminated`/`Failed` há mais de `retention.terminated_ttl_days` (expurgo automático) **ou** solicitação explícita de expurgo antecipado por direito ao esquecimento.
- P2. Checkpoints associados além de `retention.checkpoint_ttl_days`.

**Fluxo principal:**
1. O `TombstoneManager` identifica agentes elegíveis para expurgo (varredura periódica) ou recebe uma solicitação explícita de expurgo.
2. Para cada agente elegível, o `TombstoneManager` remove os blobs em MinIO (`SnapshotStore`) referenciados por seus `Checkpoint`s.
3. Os metadados sensíveis em PostgreSQL são removidos ou anonimizados, preservando apenas os metadados de auditoria mínimos exigidos por lei (ex.: `agent_id`, `tenant_id`, `terminated_at`, sem conteúdo de working memory).
4. O `agent_id` é marcado estruturalmente como **não-reutilizável** (o URN nunca é reemitido, RFC-0001 §5.1).
5. Um evento auditável de expurgo é publicado e registrado via `025-Audit`.

**Fluxos alternativos:**
- FA1. Solicitação de expurgo antecipado por titular de dados (antes do prazo padrão de retenção): mesmo fluxo, disparado fora do ciclo periódico, com rastreamento da base legal da solicitação.

**Exceções:**
- E1. Falha ao remover um blob em MinIO (indisponibilidade transitória): reagendado para nova tentativa; o agente permanece marcado como "expurgo pendente" até confirmação completa.
- E2. Tentativa de reutilizar um `agent_id` expurgado (ex.: bug de cliente reenviando um `spawn` com URN antigo): rejeitada estruturalmente pela camada de identidade (o Kernel nunca reemite URN de agente removido).

**Pós-condições:**
- Q1. Nenhum blob de snapshot/checkpoint do agente permanece em MinIO além do prazo de retenção.
- Q2. Evento de expurgo auditável, rastreável e não-repudiável (via `025-Audit`).
- Q3. `agent_id` permanentemente não-reutilizável.

---

## UC-020 — Consulta de histórico de transições (event-sourced)

**Ator primário:** Operador/SRE, ferramenta de auditoria, ou o próprio agente (introspecção do seu histórico).

**Pré-condições:**
- P1. ACB existe (qualquer estado, incluindo terminal, para fins de auditoria histórica).
- P2. Chamador possui capability de leitura sobre o `agent_id` (própria ou administrativa).

**Fluxo principal:**
1. Chamador envia `GET /v1/agents/{id}/lifecycle` (paginado).
2. `LifecyclePolicyEnforcer` avalia a leitura (pode usar decisão em cache para leituras de baixo risco).
3. `AcbStore` consulta a projeção `LifecycleTransition` ordenada por `seq` monotônico, aplicando o cursor de paginação.
4. `LifecycleApiSurface` retorna `200 OK` com a página de transições (`from_state`, `to_state`, `trigger`, `actor`, `reason`, `created_at`).

**Fluxos alternativos:**
- FA1. Consulta filtrada por intervalo de tempo ou por `trigger` específico (ex.: apenas transições de `migrate`): variação de query params sobre o mesmo endpoint.

**Exceções:**
- E1. `agent_id`/`tenant_id` não encontrado → `AIOS-LIFECYCLE-0001` (HTTP 404).
- E2. Tenant divergente (URN de outro tenant) → `AIOS-LIFECYCLE-0012` (tratado como negação de política, sem revelar a existência do recurso alheio).

**Pós-condições:**
- Q1. Nenhuma mutação de estado (leitura pura).
- Q2. Resposta reflete o histórico completo e imutável, sem lacunas de `seq`.

---

## UC-021 — Storm de hibernação sob pressão de RAM (backpressure)

**Ator primário:** `HibernationController`, reagindo a `hibernation.ram_pressure_threshold_pct` excedido em um nó/shard.

**Pré-condições:** Pressão de RAM detectada acima do limiar configurado; múltiplos agentes `Suspended` elegíveis para hibernação simultaneamente.

**Fluxo principal:**
1. `LifecycleTelemetry`/monitoramento de infraestrutura sinaliza pressão de RAM acima do limiar.
2. `HibernationController` prioriza os candidatos por `priority_class` e tempo de ociosidade (agentes `best_effort`/mais ociosos primeiro).
3. Hibernações são processadas em lote, respeitando um orçamento de concorrência (para não saturar o `SnapshotStore`/MinIO com escrita simultânea).
4. Cada hibernação segue o fluxo de **UC-007** individualmente (checkpoint → liberação de RAM → `Hibernated`).
5. Simultaneamente, o `SpawnManager` aplica *backpressure* na admissão de **novos** spawns/wakes no shard afetado, coordenando com o `009-Scheduler`, até a pressão de RAM normalizar.

**Fluxos alternativos:**
- FA1. Pressão de RAM normaliza antes de todos os candidatos serem processados: o lote restante é cancelado; agentes não processados permanecem `Suspended`.

**Exceções:**
- E1. Mesmo sob backpressure, o `SnapshotStore` (MinIO/PG) fica indisponível: hibernações são adiadas (UC-007/E1); alerta operacional elevado, pois a pressão de RAM persiste sem alívio.

**Pós-condições:**
- Q1. A pressão de RAM é aliviada de forma priorizada, sem "queda" abrupta do nó (nenhum OOM-kill não controlado de runtime).
- Q2. Nenhum spawn/wake é indevidamente derrubado por saturação — a admissão é apenas atrasada (fila, não descarte).

---

## UC-022 — Migração em massa por drenagem de nó

**Ator primário:** `027-Cluster` (via evento `cluster.node.draining`), executado pelo `MigrationOrchestrator`.

**Pré-condições:** Um nó do cluster foi marcado para drenagem (manutenção planejada, decomissionamento, ou rebalanceamento); múltiplos agentes materializados nesse nó.

**Fluxo principal:**
1. O `LifecycleEventConsumer` consome `aios.<tenant>.cluster.node.draining`.
2. `MigrationOrchestrator` enumera todos os agentes com `placement_node` igual ao nó em drenagem.
3. Para cada agente, uma migração (**UC-011**) é agendada, respeitando `migration.max_concurrent_per_node` para não saturar a rede/storage do nó de origem e do(s) destino(s).
4. O Scheduler/Cluster decide o destino de cada agente individualmente (rebalanceamento de placement); o `MigrationOrchestrator` apenas aplica.
5. Agentes migram em ondas controladas até que nenhum `placement_node` aponte mais para o nó em drenagem.
6. O `027-Cluster` é notificado (fora do escopo do 008) de que o nó está livre para ser removido/mantido.

**Fluxos alternativos:**
- FA1. Um subconjunto de agentes está `Hibernated` no nó em drenagem: para esses, a migração é apenas uma atualização de metadado de `placement` no `SnapshotStore`/`AcbStore` (mover a referência do blob, se necessário), sem materialização — muito mais barata que migrar um agente `Running`.

**Exceções:**
- E1. Algumas migrações individuais falham e são compensadas (UC-012): o `MigrationOrchestrator` re-enfileira essas migrações para nova tentativa, sem bloquear o progresso das demais.
- E2. Nenhum destino saudável disponível para parte dos agentes: essas migrações permanecem pendentes; alerta operacional é elevado se o prazo de drenagem do nó estiver próximo do vencimento.

**Pós-condições:**
- Q1. Todos os agentes anteriormente no nó em drenagem estão materializados (ou hibernados, com metadado atualizado) em outro nó, sem perda de trabalho aceito.
- Q2. O throughput de migração em lote respeita `migration.max_concurrent_per_node`, mantendo a latência individual de cada migração dentro de NFR-010 mesmo sob carga de lote.

---

## Notas finais

- Todos os fluxos de exceção referenciam o catálogo de erros
  `AIOS-LIFECYCLE-<NNNN>` definido no `_DESIGN_BRIEF.md` §5.3, sem redefini-lo.
- Todo caso de uso que resulta em mutação de estado publica exatamente um
  (ou mais, em sequência) evento(s) correspondente(s) do catálogo em
  `_DESIGN_BRIEF.md` §6.1, via Outbox transacional (invariante INV2 da FSM).
- A tabela de rastreabilidade `UC → FR/NFR → Teste` está consolidada em
  `Requirements.md` §5 e DEVE ser mantida sincronizada com este documento.
