---
Documento: FunctionalRequirements
Módulo: 009-Scheduler
Status: Draft
Versão: 0.1
Última atualização: 2026-07-20
Responsável (RACI-A): Arquiteto do Módulo 009-Scheduler
ADRs relacionados: ADR-0090, ADR-0091, ADR-0092, ADR-0093, ADR-0094, ADR-0095, ADR-0096, ADR-0097
RFCs relacionados: RFC-0001, RFC-0090, RFC-0091
Depende de: 006-Kernel, 022-Policy, 026-Cost-Optimizer, 027-Cluster
---

# 009-Scheduler — Functional Requirements

> Cada requisito é identificado por `FR-NNN`, classificado em prioridade MoSCoW
> (**MUST**/**SHOULD**/**COULD**/**WON'T**), mensurável e verificável, com
> critério de aceite explícito e origem rastreável ao
> [`_DESIGN_BRIEF.md`](./_DESIGN_BRIEF.md). Nenhum requisito aqui contradiz o
> brief; FR-001 a FR-015 reproduzem e detalham o brief §7.1. FR-016 a FR-025
> derivam de outras seções do brief (§5, §6, §8, §12) elevadas a requisito
> funcional explícito, sem introduzir componente, estado ou campo novo.
>
> Palavras normativas conforme RFC 2119 / RFC 8174. Erros referenciados usam o
> catálogo `AIOS-SCHED-<NNNN>` (brief §5.4, detalhado em `API.md`).

---

## Convenção de prioridade MoSCoW

| Nível | Significado |
|-------|-------------|
| **MUST** | Requisito obrigatório para o módulo ser considerado funcional (bloqueia release). |
| **SHOULD** | Fortemente recomendado; ausência DEVE ser justificada e registrada como débito técnico. |
| **COULD** | Desejável, não bloqueante. |
| **WON'T** | Explicitamente fora do escopo desta versão (registrado para evitar retrabalho de discussão). |

---

## Tabela consolidada

| ID | Requisito | Prioridade | Origem |
|----|-----------|------------|--------|
| FR-001 | Admission Control multi-critério | MUST | Brief §1 R-01, §7.1 |
| FR-002 | Score multiobjetivo e prioridade efetiva | MUST | Brief §1 R-02, §7.1 |
| FR-003 | Placement determinístico por sharding | MUST | Brief §1 R-03, §7.1 |
| FR-004 | Preempção segura e compensável | MUST | Brief §1 R-04, §7.1 |
| FR-005 | Backpressure com sinalização ao produtor | MUST | Brief §1 R-05, §7.1 |
| FR-006 | Filas de prioridade particionadas | MUST | Brief §1 R-06, §7.1 |
| FR-007 | Emissão do ciclo de eventos de decisão | MUST | Brief §1 R-07, §6.1, §7.1 |
| FR-008 | Idempotência de decisão | MUST | Brief §1 R-08, §7.1 |
| FR-009 | Fairness e anti-starvation | MUST | Brief §1 R-09, §7.1 |
| FR-010 | Registro de proveniência da decisão | MUST | Brief §1 R-10, §7.1 |
| FR-011 | Reavaliação e re-scheduling | MUST | Brief §1 R-11, §7.1 |
| FR-012 | Enforcement de SLA/deadline (EDF) | SHOULD | Brief §1 R-12, §7.1 |
| FR-013 | Estabilidade de interface de política v1→v2 | MUST | Brief §1 R-13, §7.1 |
| FR-014 | Reconciliação de estado (filas × runtime) | MUST | Brief §1, §9 |
| FR-015 | Cancelamento de tarefa | SHOULD | Brief §5.2/§5.3, §7.1 |
| FR-016 | Superfície gRPC/REST estável | MUST | Brief §5 |
| FR-017 | Consulta de proveniência e observabilidade de filas | SHOULD | Brief §5.2 |
| FR-018 | Recarga de configuração em runtime | SHOULD | Brief §8 |
| FR-019 | Isolamento multi-tenant | MUST | Brief §12.1 |
| FR-020 | Enforcement PEP→PDP | MUST | Brief §12.1, RFC-0001 §5.8 |
| FR-021 | Minimização de dados (LGPD/GDPR) | MUST | Brief §12.3 |
| FR-022 | Sinalização de níveis de backpressure | MUST | Brief §10 |
| FR-023 | Rebalanceamento de shards coordenado | SHOULD | Brief §10 |
| FR-024 | Configuração de vazão mínima por tenant | MUST | Brief §8, §7.2 NFR-006 |
| FR-025 | Tratamento de eventos não processáveis (DLQ) | SHOULD | Brief §9 |

---

## Detalhamento

### FR-001 — Admission Control multi-critério

**Descrição normativa:** O Scheduler DEVE avaliar cada `SchedulingRequest`
recebida via `Submit` contra quatro critérios, na ordem: (1) cota de
concorrência (`QuotaLedger`), (2) orçamento disponível (`BudgetClient`→026),
(3) capacidade residual de shard/nó (`CapacityProvider`), e (4) autorização do
PDP (`PolicyClient`→022). A requisição só transita `RECEIVED → ADMITTED` se
**todas** as guardas forem verdadeiras; caso contrário transita para
`REJECTED` com `reason_code` apropriado (`AIOS-SCHED-0001`..`0004`).

**Prioridade:** MUST.

**Critério de aceite:**
- Dado quota de concorrência esgotada, quando `Submit` é chamado, então a
  resposta é `REJECTED` com `code = AIOS-SCHED-0001`, HTTP 429, `retriable = true`.
- Dado orçamento insuficiente para `est_cost_usd`, quando `Submit` é chamado,
  então `code = AIOS-SCHED-0002`, HTTP 429, `retriable = true`.
- Dado backpressure em nível *reject*, então `code = AIOS-SCHED-0003`, HTTP 503,
  com `retryAfterMs` preenchido.
- Dado PDP nega, então `code = AIOS-SCHED-0004`, HTTP 403, `retriable = false`.
- Dado todos os critérios satisfeitos, então outcome `ADMITTED` é emitido em
  ≤ 20 ms p99 (NFR-001) e o slot é reservado atomicamente (sem overcommit,
  NFR-005).

### FR-002 — Score multiobjetivo e prioridade efetiva

**Descrição normativa:** O Scheduler DEVE computar, para cada tarefa admitida,
o custo multiobjetivo `C = α·L + β·$ + γ·E` (L = latência estimada, $ = custo
estimado, E = energia/consumo estimado), com `alpha + beta + gamma ≈ 1`
normalizados por tenant/classe (`scheduler.weights.*`), e derivar a
`effective_priority` respeitando `quality_floor` (`Q ≥ Q_min`), a classe de
serviço (`service_class`) e o orçamento remanescente.

**Prioridade:** MUST.

**Critério de aceite:**
- Dado pesos `α=0.5, β=0.3, γ=0.2` e uma tarefa com estimativas `L,$,E`, então
  `cost_score` calculado é determinístico e reproduzível a partir do
  `feature_vector` persistido (auditável).
- Dado `quality_floor` inatingível sob as restrições correntes, então a
  decisão é `REJECTED` com `code = AIOS-SCHED-0007` (HTTP 422).
- A prioridade efetiva DEVE ser monotônica em relação a `C` (menor `C` → maior
  urgência) e DEVE incorporar o termo de aging descrito em FR-009.

### FR-003 — Placement determinístico por sharding

**Descrição normativa:** O Scheduler DEVE escolher o shard de destino via
`shard = hash(tenant_id, agent_id) mod N` (`ShardRouter`), e o nó/pool de
runtime dentro do shard via `PlacementEngine`, respeitando `affinity`/
anti-afinidade declaradas em `SchedulingRequest.affinity` e a capacidade
residual reportada por `CapacityProvider`.

**Prioridade:** MUST.

**Critério de aceite:**
- Dado o mesmo par `(tenant_id, agent_id)` e o mesmo `N`, o shard calculado é
  sempre idêntico (função pura, sem estado).
- Dado nenhum shard/nó elegível (capacidade zero em todos), então a decisão é
  `REJECTED` com `code = AIOS-SCHED-0010` (HTTP 503).
- Dado uma dica de anti-afinidade, o `PlacementTarget` escolhido NÃO DEVE
  colocar a tarefa no mesmo nó que a entidade indicada como incompatível,
  salvo se nenhum outro nó tiver capacidade (fallback documentado e logado).

### FR-004 — Preempção segura e compensável

**Descrição normativa:** O Scheduler DEVE, quando uma tarefa de maior
prioridade efetiva não encontra capacidade livre, selecionar vítima(s) de
menor prioridade efetiva em execução, obter autorização do PDP, e sinalizar
suspensão ao Runtime Supervisor com um `grace_period_ms` configurável antes de
liberar o slot.

**Prioridade:** MUST.

**Critério de aceite:**
- Nenhuma preempção ocorre sem uma consulta prévia ao PDP retornando `allow`.
- O evento `aios.<tenant>.task.execution.preempted` é emitido com
  `{taskUrn, byTaskUrn, gracePeriodMs}` antes da suspensão efetiva ser
  confirmada.
- O overhead de decisão de preempção é ≤ 50 ms (NFR-009); o `grace_period_ms`
  default é 2000 ms e é configurável por classe.
- Tarefas recém-agendadas (dentro do cooldown, ADR-0094) NÃO DEVEM ser
  escolhidas como vítima, prevenindo *thrashing*.

### FR-005 — Backpressure com sinalização ao produtor

**Descrição normativa:** O Scheduler DEVE calcular um nível de saturação
(`accept`/`defer`/`reject`) combinando profundidade de fila, lag de JetStream e
pressão de capacidade, e propagar esse sinal ao produtor (Kernel/Gateway) via
código de erro e cabeçalho `Retry-After`/`retryAfterMs` quando em nível
`reject`.

**Prioridade:** MUST.

**Critério de aceite:**
- Dado pressão ≥ `scheduler.backpressure.reject_threshold` (default 0.95),
  então novas admissões retornam `AIOS-SCHED-0003` com `retryAfterMs` presente.
- Dado pressão entre `defer_threshold` (0.80) e `reject_threshold`, a tarefa é
  aceita porém pode ser diferida (`DEFERRED`) antes de reentrar em `QUEUED`.
- O evento `aios.<tenant>.scheduler.backpressure.changed` é emitido a cada
  transição de nível.

### FR-006 — Filas de prioridade particionadas

**Descrição normativa:** O Scheduler DEVE manter filas de execução como Redis
ZSET, particionadas por `(tenant, service_class, shard)`, com score igual à
`effective_priority` (menor valor = mais urgente), permitindo `enqueue`,
`dequeue` (com lease), `peek` e `aging` em tempo O(log n).

**Prioridade:** MUST.

**Critério de aceite:**
- Cada chave ZSET segue o padrão `sched:{tenant}:{class}:{shard}` (brief §3.3).
- O `DispatchLoop` retira sempre o membro de menor score elegível (capacidade
  disponível) sem *lock* global — apenas lease por shard.
- Uma tarefa nunca aparece em mais de uma ZSET simultaneamente (invariante de
  unicidade de fila).

### FR-007 — Emissão do ciclo de eventos de decisão

**Descrição normativa:** O Scheduler DEVE publicar, no envelope CloudEvents da
RFC-0001 §5.2, todos os eventos listados no brief §6.1
(`admitted`, `rejected`, `scheduled`, `preempted`, `resumed`, `completed`,
`failed`, `backpressure.changed`, `decision.recorded`) no subject
`aios.<tenant>.task.execution.<acao>` ou `aios.<tenant>.scheduler.<entidade>.<acao>`.

**Prioridade:** MUST.

**Critério de aceite:**
- 100% das transições de estado que geram um dos oito outcomes de
  `SchedulingDecision` produzem exatamente um evento correspondente (ou mais,
  em caso de re-tentativa idempotente controlada) — ver `Events.md`.
- Todo evento publicado valida contra o `dataschema` registrado (ver
  `Events.md` de batch posterior) e carrega `traceparent`.
- Publicação ocorre via outbox transacional (`DecisionJournal` →
  `EventPublisher`), garantindo atomicidade entre gravação da decisão e
  publicação (padrão *Outbox*, RFC-0001 §5.2, ADR-0097).

### FR-008 — Idempotência de decisão

**Descrição normativa:** O Scheduler DEVE garantir que uma mesma
`SchedulingRequest`, identificada por `Idempotency-Key` (equivalente a
`request_id`), produza no máximo uma admissão efetiva, mesmo sob repetição
(retry do cliente, entrega at-least-once).

**Prioridade:** MUST.

**Critério de aceite:**
- Repetir `Submit` com a mesma `Idempotency-Key` e mesmo payload retorna o
  `response_snapshot` gravado na primeira execução, sem nova reserva de slot.
- Repetir com a mesma chave e payload divergente retorna
  `AIOS-SCHED-0006` (HTTP 409).
- `IdempotencyRecord` é retido por, no mínimo, `scheduler.idempotency.ttl_hours`
  (default 24h), conforme RFC-0001 §5.5.

### FR-009 — Fairness e anti-starvation

**Descrição normativa:** O Scheduler DEVE aplicar *aging* (boost de prioridade
proporcional ao tempo em fila, até `scheduler.aging.max_boost`) para garantir
que todo tenant/classe ativo obtenha uma vazão mínima (`scheduler.fairness.min_share`)
em uma janela de observação configurável (default 60 s).

**Prioridade:** MUST.

**Critério de aceite:**
- Sob carga adversarial (um tenant satura a fila), tenants de baixa prioridade
  ainda recebem ≥ `min_share` de despachos na janela (NFR-006).
- O boost de aging é limitado por `scheduler.aging.max_boost` (não permite que
  uma tarefa antiga ultrapasse indefinidamente a prioridade das demais).

### FR-010 — Registro de proveniência da decisão

**Descrição normativa:** O Scheduler DEVE persistir, para cada
`SchedulingDecision`, o vetor de features (`feature_vector`), os pesos
aplicados (`alpha`,`beta`,`gamma`), a identidade e versão da política
(`policy_id`,`policy_version`) e o resultado (`outcome`), de forma append-only
em PostgreSQL, habilitando auditoria (025) e treino futuro (023).

**Prioridade:** MUST.

**Critério de aceite:**
- 100% das decisões (`ADMITTED`,`SCHEDULED`,`REJECTED`,`PREEMPTED`,`RESUMED`,
  `DEFERRED`) têm um registro correspondente em `SchedulingDecision` (NFR-010).
- Registros são imutáveis (sem `UPDATE`/`DELETE` fora de expurgo LGPD
  auditado — ver FR-021).

### FR-011 — Reavaliação e re-scheduling

**Descrição normativa:** O Scheduler DEVE reprogramar tarefas em fila (ou já
despachadas) diante de: mudança de capacidade (evento de 027), preempção
externa, expiração de lease (Runtime não confirma), ou expiração de deadline.

**Prioridade:** MUST.

**Critério de aceite:**
- Ao receber `aios.<tenant>.cluster.capacity.changed`, o `CapacityProvider` é
  atualizado e o próximo ciclo do `DispatchLoop` reflete a nova capacidade em
  ≤ 1 heartbeat (`scheduler.capacity.heartbeat_ttl_ms`).
- Lease de dispatch expirado (`scheduler.lease.ttl_ms`) resulta em
  re-enfileiramento automático pelo `ReconciliationWorker`, sem duplicar
  execução (idempotência preservada).

### FR-012 — Enforcement de SLA/deadline (EDF)

**Descrição normativa:** Quando `scheduler.deadline.edf_enabled=true` e a
`SchedulingRequest` carrega `deadline_at`, o Scheduler DEVERIA aplicar
*Earliest-Deadline-First* como componente adicional da prioridade efetiva, e
DEVE mover a tarefa para `EXPIRED` quando `now > deadline_at` antes do
despacho.

**Prioridade:** SHOULD.

**Critério de aceite:**
- Duas tarefas de mesma classe e custo, com deadlines diferentes, são
  despachadas na ordem do deadline mais próximo primeiro.
- Uma tarefa cujo deadline vence em `QUEUED` transita para `EXPIRED` (terminal)
  e emite `rejected` com `reason_code` de deadline.

### FR-013 — Estabilidade de interface de política v1→v2

**Descrição normativa:** O Scheduler DEVE encapsular toda lógica de decisão de
prioridade/custo atrás da interface `ISchedulingPolicy`, permitindo substituir
`HeuristicPolicy` (v1) por `LearnedPolicy` (v2, RL) via
`scheduler.policy.engine` sem alterar a superfície de API (§5) ou o catálogo de
eventos (§6).

**Prioridade:** MUST.

**Critério de aceite:**
- Trocar `scheduler.policy.engine` de `heuristic` para `rl` não requer
  mudança de contrato gRPC/REST nem de schema de evento (validado por teste de
  contrato).
- `policy_version` é registrado em toda `SchedulingDecision`, permitindo
  auditoria de qual política decidiu.

### FR-014 — Reconciliação de estado (filas × runtime real)

**Descrição normativa:** O `ReconciliationWorker` DEVE, periodicamente,
comparar o estado das filas/leases com sinais reportados pelo Runtime
Supervisor, expirando leases órfãos e corrigindo divergências (ex.: tarefa
`SCHEDULED` sem confirmação de `RUNNING` além do TTL de lease).

**Prioridade:** MUST.

**Critério de aceite:**
- Nenhuma tarefa permanece em `SCHEDULED` além de `scheduler.lease.ttl_ms` sem
  ação corretiva (re-enqueue ou marcação de falha).
- A reconciliação não introduz execução duplicada (idempotência por
  `request_id` preservada).

### FR-015 — Cancelamento de tarefa

**Descrição normativa:** O Scheduler DEVERIA aceitar `Cancel(request_id)` para
tarefas em `QUEUED` ou `SCHEDULED`, liberando reserva de slot e emitindo
`rejected` com `reason_code = cancelled`.

**Prioridade:** SHOULD.

**Critério de aceite:**
- Cancelar uma tarefa em `RUNNING` retorna `AIOS-SCHED-0011` (conflito de
  estado terminal/aplicável), pois a autoridade de execução é do 007.
- Cancelar uma tarefa já em estado terminal é idempotente (retorna o mesmo
  resultado, sem erro).

### FR-016 — Superfície gRPC/REST estável

**Descrição normativa:** O Scheduler DEVE expor os métodos gRPC
(`Submit`,`Cancel`,`Preempt`,`GetDecision`,`ReportRuntimeSignal`,
`StreamQueueState`) do serviço `aios.scheduler.v1.SchedulerService` e as rotas
REST equivalentes (brief §5.3), versionadas por `/v1/...`.

**Prioridade:** MUST.

**Critério de aceite:** ver `API.md` (contrato OpenAPI/proto completo,
documento de batch posterior). Toda mutação aceita `Idempotency-Key`; toda
chamada propaga `traceparent`/`X-AIOS-Tenant`.

### FR-017 — Consulta de proveniência e observabilidade de filas

**Descrição normativa:** O Scheduler DEVERIA expor `GetDecision(request_id)`
para consulta pontual de proveniência e `StreamQueueState` para observabilidade
em tempo real do estado das filas por tenant/classe/shard.

**Prioridade:** SHOULD.

**Critério de aceite:** `GetDecision` para um `request_id` inexistente retorna
`AIOS-SCHED-0009` (HTTP 404). `StreamQueueState` é somente leitura e não
consome slot de admissão.

### FR-018 — Recarga de configuração em runtime

**Descrição normativa:** Chaves marcadas "Recarregável = Sim" no brief §8
(pesos α/β/γ, cotas, thresholds de backpressure, etc.) DEVEM poder ser
atualizadas sem reinício do serviço, respeitando autorização via 022.

**Prioridade:** SHOULD.

**Critério de aceite:** Uma alteração de `scheduler.weights.alpha` via
`PUT /v1/scheduler/policy` reflete em novas decisões em ≤ 5 s (NFR-017), sem
interromper requisições em voo.

### FR-019 — Isolamento multi-tenant

**Descrição normativa:** Toda entidade persistida DEVE carregar `tenant_id`
com Row-Level Security ativo; filas Redis DEVEM ser particionadas por tenant;
subjects NATS DEVEM usar o namespace `aios.<tenant>.*`.

**Prioridade:** MUST.

**Critério de aceite:** Uma consulta/decisão de um tenant nunca retorna ou
influencia dados de outro tenant (testado via teste de isolamento RLS e
teste de namespace NATS).

### FR-020 — Enforcement PEP→PDP

**Descrição normativa:** Toda operação de admissão, preempção ou mutação de
política DEVE passar pelo `SchedulingGateway` (PEP) e ser autorizada pelo PDP
(022) segundo a postura *default deny* (RFC-0001 §5.8).

**Prioridade:** MUST.

**Critério de aceite:** 100% das mutações privilegiadas têm uma chamada PDP
correspondente auditável (NFR-012); indisponibilidade do PDP resulta em
negação (`AIOS-SCHED-0004`), nunca em admissão silenciosa.

### FR-021 — Minimização de dados (LGPD/GDPR)

**Descrição normativa:** O Scheduler NÃO DEVE persistir payload de tarefa
inline; DEVE usar `payload_ref` (URN/URI para MinIO). O `feature_vector` NÃO
DEVE conter PII bruta.

**Prioridade:** MUST.

**Critério de aceite:** Varredura de schema/amostras de `SchedulingRequest` e
`SchedulingDecision` não encontra campos de payload de negócio ou PII bruta
fora de `payload_ref`.

### FR-022 — Sinalização de níveis de backpressure

**Descrição normativa:** O `BackpressureController` DEVE expor três níveis
(`accept`,`defer`,`reject`) com limiares configuráveis
(`scheduler.backpressure.defer_threshold`,`reject_threshold`) e publicar
mudança de nível como evento.

**Prioridade:** MUST. (Detalha FR-005.)

**Critério de aceite:** ver FR-005; adicionalmente, o nível corrente é
consultável via `GET /v1/scheduler/capacity`.

### FR-023 — Rebalanceamento de shards coordenado

**Descrição normativa:** Quando `N` (`scheduler.shards.count`) muda, o
Scheduler DEVERIA aplicar hashing consistente para minimizar migração de
tarefas entre shards, coordenando a mudança com o Cluster (027).

**Prioridade:** SHOULD.

**Critério de aceite:** Uma mudança de `N` de 64 para 65 remapeia uma fração
próxima de `1/N` das chaves (não todas), verificável por teste de
rebalanceamento (ADR-0092).

### FR-024 — Configuração de vazão mínima por tenant

**Descrição normativa:** O Scheduler DEVE permitir configurar
`scheduler.fairness.min_share` por tenant, garantindo o piso de vazão usado
por FR-009.

**Prioridade:** MUST.

**Critério de aceite:** Alterar `min_share` de um tenant específico não afeta
o `min_share` de outros tenants (isolamento de configuração).

### FR-025 — Tratamento de eventos não processáveis (DLQ)

**Descrição normativa:** Eventos consumidos que não puderem ser processados
após `scheduler.retry.max_attempts` DEVERIAM ser encaminhados a uma DLQ do
JetStream para inspeção e replay manual.

**Prioridade:** SHOULD.

**Critério de aceite:** Um evento malformado consumido (ex.:
`cluster.capacity.changed` com schema inválido) não trava o consumidor
principal e é encontrável na stream DLQ correspondente.

---

## Requisitos WON'T (explicitamente fora de escopo nesta versão)

| ID | Requisito descartado | Justificativa | Dono correto |
|----|------------------------|----------------|--------------|
| WON'T-01 | Execução do loop cognitivo do agente | Não-responsabilidade N-01 do brief | 007-Agent-Runtime |
| WON'T-02 | Cálculo/faturamento de preço | Não-responsabilidade N-05 do brief | 026-Cost-Optimizer |
| WON'T-03 | Avaliação de políticas RBAC/ABAC | Não-responsabilidade N-03 do brief (Scheduler é PEP, não PDP) | 022-Policy |
| WON'T-04 | HA/replicação de NATS/PG/Redis | Não-responsabilidade N-07 do brief | 027-Cluster |

---

*Fim de `FunctionalRequirements.md` — Módulo 009-Scheduler.*
