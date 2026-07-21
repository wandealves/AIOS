---
Documento: FunctionalRequirements
Módulo: 008-Agent-Lifecycle
Status: Draft
Versão: 0.1
Última atualização: 2026-07-20
Responsável (RACI-A): Arquiteto do Módulo 008 — Agent Lifecycle
ADRs relacionados: ADR-0080, ADR-0081, ADR-0082, ADR-0083, ADR-0084, ADR-0085, ADR-0086, ADR-0087, ADR-0089
RFCs relacionados: RFC-0001, RFC-0080 (a propor)
Depende de: `_DESIGN_BRIEF.md` (deste módulo), `Requirements.md`, `../003-RFC/RFC-0001-Architecture-Baseline.md`
---

# 008-Agent-Lifecycle — Functional Requirements

> Convenção de prioridade: **MoSCoW** (`Must` / `Should` / `Could` /
> `Won't-this-release`). Todo FR é mensurável e verificável por um critério de
> aceite objetivo. A numeração é global e sequencial, **idêntica** à do
> `_DESIGN_BRIEF.md` §7.1; não reutilize um ID mesmo se um FR for depreciado
> (marque como `Deprecated` no lugar). "Origem" aponta para a seção do brief
> de onde o requisito deriva. Nenhum FR abaixo contradiz o brief — cada um
> **elabora** o critério de aceite com maior profundidade operacional.

## A. Máquina de Estados Canônica

| ID | Descrição | Prioridade | Critério de aceite | Origem |
|----|-----------|-----------|--------------------|--------|
| FR-001 | O módulo 008 DEVE ser a autoridade única sobre a máquina de estados canônica do agente (`Created → Ready → Running ↔ Suspended → Hibernated → Migrating → Terminated/Failed`, §4 do brief) e o único componente autorizado a persistir transições do ACB. O `StateMachineEngine` DEVE validar toda transição contra a tabela de arestas/guardas (T1–T11) antes de qualquer efeito colateral. | Must | Transição fora das arestas definidas na FSM retorna `AIOS-LIFECYCLE-0002` (HTTP 409) sem qualquer mutação de `AgentControlBlock` nem publicação de evento; teste unitário cobre 100% das combinações `(from_state, trigger)` válidas e inválidas; nenhum outro módulo grava diretamente em `AgentControlBlock.state`. | Brief §1.1 (R1), §4 |

## B. Spawn e Materialização

| ID | Descrição | Prioridade | Critério de aceite | Origem |
|----|-----------|-----------|--------------------|--------|
| FR-002 | O `SpawnManager` DEVE materializar um agente novo ou frio negociando slot com o `009-Scheduler` e runtime com o `007-Agent-Runtime`, atingindo `p99 ≤ 250 ms` do gatilho `spawn`/`wake` até `Running`. | Must | Benchmark de carga sustentada mede `aios_lifecycle_spawn_duration_ms{quantile="0.99"} ≤ 250` e `{quantile="0.5"} ≤ 80` com `WarmPoolManager` ativo (NFR-001); evento `agent.lifecycle.running` emitido ao final; timeout além de `spawn.timeout_ms` retorna `AIOS-LIFECYCLE-0006`. | Brief §1.1 (R3), §2 (SpawnManager) |
| FR-014 | O `WarmPoolManager` DEVE manter um pool de runtimes pré-aquecidos por tenant/shard, respeitando `warmpool.min_per_shard`/`warmpool.max_per_shard`, para reduzir *cold-start* de processo de runtime durante `spawn`/`wake`. | Should | Reserva mínima (`warmpool.min_per_shard`) é mantida sob observação contínua (métrica `aios_lifecycle_warmpool_available`); teste de integração confirma que `spawn`/`wake` com pool disponível não invoca o boot completo de runtime (apenas *attach*), reduzindo a latência medida em relação ao cenário sem pool. | Brief §2 (WarmPoolManager), §10 |

## C. Suspensão e Retomada

| ID | Descrição | Prioridade | Critério de aceite | Origem |
|----|-----------|-----------|--------------------|--------|
| FR-003 | O `LifecycleCoordinator` DEVE executar `suspend` (`Running → Suspended`) e `resume` (`Suspended → Running`) preservando a memória de trabalho do agente (working memory em RAM/Redis), sem perda de estado cognitivo em curso. | Must | Após `resume`, o hash de integridade da working memory (`working_memory_ref`) é idêntico ao capturado imediatamente antes do `suspend`; latência de `resume` medida por `aios_lifecycle_transition_duration_ms{trigger="resume"}` respeita NFR-002; `suspend` sem lease adquirido é rejeitado com `AIOS-LIFECYCLE-0003`. | Brief §1.1 (R4), §4.3 (T4, T5) |

## D. Hibernação de Cold Agents

| ID | Descrição | Prioridade | Critério de aceite | Origem |
|----|-----------|-----------|--------------------|--------|
| FR-004 | O `HibernationController` DEVE hibernar *cold agents* (`Suspended → Hibernated`) serializando ACB + working memory via `CheckpointService`/`SnapshotStore` para MinIO/PostgreSQL, liberando toda a RAM associada ao runtime antes de confirmar a transição. | Must | Após `hibernate`, o RSS do processo de runtime associado é `0`; a transição só é confirmada se o checkpoint resultante tem `consistency=consistent` (invariante INV4); snapshot íntegro (`content_hash` verificável) em MinIO/PG; evento `agent.lifecycle.hibernated` emitido apenas após liberação de RAM. | Brief §1.1 (R5), §4.3 (T6) |
| FR-005 | O `HibernationController`/`SpawnManager` DEVE materializar um *cold agent* sob demanda (`Hibernated → Ready/Running`, gatilho `wake`), restaurando ACB + working memory a partir do `last_checkpoint_id` e incrementando `generation`. | Must | `wake` restaura `generation = generation_anterior + 1`; evento `agent.lifecycle.woken` emitido; agente retoma execução com estado cognitivo equivalente ao momento do checkpoint (teste de round-trip); latência de materialização respeita NFR-001 (`p99 ≤ 250 ms`). | Brief §1.1 (R5), §4.3 (T7) |

## E. Checkpoint e Restore

| ID | Descrição | Prioridade | Critério de aceite | Origem |
|----|-----------|-----------|--------------------|--------|
| FR-006 | O `CheckpointService` DEVE criar checkpoints consistentes de ACB + working memory sob demanda ou periodicamente (`checkpoint.interval_s`), e restaurá-los de forma atômica, com verificação de `content_hash` (sha-256) e cifragem `aes-256-gcm` por tenant. | Must | `POST …/checkpoint` cria um `Checkpoint` com `consistency=consistent` para agente `Running`/`Suspended`; `POST …/restore` reconstrói o ACB e a working memory a partir de um checkpoint válido, byte-a-byte equivalente ao capturado; checkpoint com `content_hash` divergente é rejeitado com `AIOS-LIFECYCLE-0007`; checkpoint inexistente retorna `AIOS-LIFECYCLE-0008`. | Brief §1.1 (R6), §3.3, §4.3 (T6, T7) |

## F. Migração

| ID | Descrição | Prioridade | Critério de aceite | Origem |
|----|-----------|-----------|--------------------|--------|
| FR-007 | O `MigrationOrchestrator` DEVE migrar um agente entre shards/nós como uma **saga** (`quiesce → checkpoint → transfer → materialize → cutover → cleanup`), consumindo decisões de placement do `009-Scheduler`/`027-Cluster`, sem perda de trabalho aceito. | Must | `MigrationJob` conclui a fase `cutover` com o agente materializado no destino (`Ready`/`Running`/`Hibernated`, conforme T9) e a origem invalidada/drenada; falha em qualquer fase aciona compensação (`MigrationJob.status=compensated`) que reativa a origem sem perda de trabalho aceito; latência respeita NFR-010 (`p99 ≤ 2 s`, agente ≤ 64 MiB, mesmo DC). | Brief §1.1 (R7), §3.5, §4.3 (T8, T9) |

## G. Eventos

| ID | Descrição | Prioridade | Critério de aceite | Origem |
|----|-----------|-----------|--------------------|--------|
| FR-008 | O `LifecycleEventPublisher` DEVE emitir todo evento do catálogo `agent.lifecycle.*` (`Events.md`) de forma **transacional** (Outbox) e **idempotente**, garantindo que nenhuma transição de estado seja persistida sem o evento correspondente ser eventualmente publicado. | Must | Teste de *crash injection* entre o commit da transação de transição e a publicação no NATS não resulta em evento perdido (o relay do Outbox reenvia); consumidores deduplicam por `event.id` sem efeito colateral duplicado; nenhum evento é publicado sem a transição correspondente ter sido commitada. | Brief §1.1 (R8), §3.6, §6.1 |

## H. Reconciliação

| ID | Descrição | Prioridade | Critério de aceite | Origem |
|----|-----------|-----------|--------------------|--------|
| FR-009 | O `ReconciliationController` DEVE reconciliar continuamente o estado desejado (`desired_state`) contra o estado observado do agente (padrão *controller/reconcile*), detectando e corrigindo derivas como runtime morto sem transição para `Terminated`. | Must | Ausência de heartbeat do `007-Agent-Runtime` além de `reconcile.runtime_dead_after_ms` é detectada e reconciliada dentro de `reconcile.interval_ms` do ciclo seguinte; teste de *chaos* que mata o runtime sem sinalizar saída confirma que o ACB converge ao estado correto (`Ready`/`Running` via restore do checkpoint, ou `Failed` se além de `max_recovery`) sem intervenção manual. | Brief §1.1 (R9), §9 |

## I. Concorrência

| ID | Descrição | Prioridade | Critério de aceite | Origem |
|----|-----------|-----------|--------------------|--------|
| FR-010 | O `LeaseManager` DEVE garantir que, no máximo, um coordenador execute uma transição por agente por vez, usando lease distribuído (Redis `SET NX PX`) com `fencing_token` monotônico persistido no ACB. | Must | Duas requisições concorrentes de mutação sobre o mesmo `agent_id`: uma adquire o lease e sucede; a outra recebe `AIOS-LIFECYCLE-0003` (HTTP 409, `retriable=true`); escrita com `fencing_token` obsoleto (coordenador antigo/rogue) é rejeitada com `AIOS-LIFECYCLE-0013`, independentemente de o lease ter expirado. | Brief §1.1 (R10), §3.6, §4.3 (INV1) |

## J. Retenção e Expurgo

| ID | Descrição | Prioridade | Critério de aceite | Origem |
|----|-----------|-----------|--------------------|--------|
| FR-011 | O `TombstoneManager` DEVE aplicar retenção (`retention.terminated_ttl_days`, `retention.checkpoint_ttl_days`) e expurgo de agentes `Terminated`/`Failed`, suportando o direito ao esquecimento (LGPD/GDPR) sobre snapshots/checkpoints, sem jamais reutilizar um `agent_id` expurgado. | Must | Expurgo além do prazo de retenção remove os blobs em MinIO e os metadados sensíveis em PostgreSQL, preservando apenas os metadados de auditoria exigidos por lei; evento auditável é emitido via `025-Audit`; tentativa de recriar o mesmo `agent_id` após expurgo é rejeitada estruturalmente (URN nunca reemitido, RFC-0001 §5.1). | Brief §1.1 (R12), §12.3 |

## K. Observabilidade da API

| ID | Descrição | Prioridade | Critério de aceite | Origem |
|----|-----------|-----------|--------------------|--------|
| FR-012 | O `LifecycleApiSurface` DEVE expor o histórico de transições de ciclo de vida (event-sourced, `LifecycleTransition`) por agente, paginado e ordenado por `seq`. | Must | `GET /v1/agents/{id}/lifecycle` retorna a sequência completa de transições ordenada por `seq` monotônico por agente, com paginação por cursor; nenhuma transição registrada em `LifecycleTransition` está ausente da resposta paginada completa. | Brief §1.1 (R11), §3.2, §5.1 |

## L. Segurança (PEP)

| ID | Descrição | Prioridade | Critério de aceite | Origem |
|----|-----------|-----------|--------------------|--------|
| FR-013 | O `LifecyclePolicyEnforcer` DEVE interceptar toda mutação de ciclo de vida e consultar o PDP do `022-Policy` antes de qualquer efeito colateral, aplicando *default deny*. | Must | Requisição de mutação sem autorização do PDP retorna `AIOS-LIFECYCLE-0012` (HTTP 403) sem qualquer alteração de `AgentControlBlock`, sem consumo de lease e sem publicação de evento de sucesso; teste de integração cobre as ações `lifecycle:spawn`, `:suspend`, `:resume`, `:hibernate`, `:wake`, `:migrate`, `:checkpoint`, `:restore`, `:terminate`. | Brief §1.1, §12.1 |

## Notas de rastreabilidade

- A relação completa `FR-NNN → UC-NNN → Teste` está em `Requirements.md` §5.
- A numeração `FR-001`..`FR-014` e as descrições-base são **idênticas** às do
  `_DESIGN_BRIEF.md` §7.1; os critérios de aceite acima são **elaborações**
  operacionais, não requisitos novos fora de escopo.
- Alterações a este catálogo (adição, depreciação) DEVEM ser acompanhadas de
  atualização da matriz de rastreabilidade em `Requirements.md` §5.
