---
Documento: UseCases
Módulo: 009-Scheduler
Status: Draft
Versão: 0.1
Última atualização: 2026-07-20
Responsável (RACI-A): Arquiteto do Módulo 009-Scheduler
ADRs relacionados: ADR-0090, ADR-0091, ADR-0092, ADR-0093, ADR-0094, ADR-0095, ADR-0096, ADR-0097
RFCs relacionados: RFC-0001, RFC-0090, RFC-0091
Depende de: 006-Kernel, 022-Policy, 026-Cost-Optimizer, 027-Cluster
---

# 009-Scheduler — Use Cases

> Casos de uso numerados `UC-NNN`, no formato ator / pré-condições / fluxo
> principal / fluxos alternativos / exceções / pós-condições, rastreáveis aos
> requisitos de [`FunctionalRequirements.md`](./FunctionalRequirements.md) e
> [`NonFunctionalRequirements.md`](./NonFunctionalRequirements.md). Todos os
> casos de uso são consistentes com a máquina de estados e os componentes do
> [`_DESIGN_BRIEF.md`](./_DESIGN_BRIEF.md) (§2, §4) — nenhum estado, campo ou
> componente aqui citado diverge do brief.
>
> Convenção de atores: **Kernel** (006, chamador típico de `Submit`),
> **Runtime Supervisor** (007, materializador de decisões e emissor de
> sinais), **Policy Engine/PDP** (022), **Cost-Optimizer** (026),
> **Cluster** (027), **Operador/SRE** (029, via API admin), **Scheduler**
> (o próprio módulo, ator de sistema quando reage a eventos/temporizadores).

---

## UC-001 — Submeter tarefa e obter admissão (caminho feliz)

| Campo | Conteúdo |
|-------|----------|
| **Ator primário** | Kernel (006), em nome de um agente. |
| **Pré-condições** | `tenant_id` autenticado; cota de concorrência não esgotada; orçamento suficiente; capacidade de ao menos um shard/nó; PDP disponível. |
| **Gatilho** | Chamada `Submit(SchedulingRequest)` via gRPC (ou `POST /v1/scheduler/requests`). |
| **Fluxo principal** | 1. `SchedulingGateway` valida envelope, `traceparent`, `X-AIOS-Tenant` e `Idempotency-Key`. <br> 2. Verifica `IdempotencyStore`: nenhum registro prévio para a chave. <br> 3. `AdmissionController` consulta `QuotaLedger` (cota OK), `BudgetClient`→026 (orçamento OK), `CapacityProvider` (capacidade OK) e `PolicyClient`→022 (PDP `allow`). <br> 4. Reserva atômica de slot (`SlotReservation`); transição `RECEIVED → ADMITTED`; evento `task.execution.admitted`. <br> 5. `PriorityCostEvaluator` computa `C = α·L + β·$ + γ·E` e `effective_priority` via `ISchedulingPolicy`. <br> 6. `PriorityQueueStore` enfileira (ZSET `sched:{tenant}:{class}:{shard}`); transição `ADMITTED → QUEUED`. <br> 7. `DispatchLoop` do shard correspondente encontra a tarefa no topo elegível, confirma capacidade, obtém `PlacementTarget` via `PlacementEngine`/`ShardRouter`. <br> 8. Transição `QUEUED → SCHEDULED`; `DecisionJournal` grava a decisão (outbox) e `EventPublisher` emite `task.execution.scheduled`. <br> 9. `RuntimeSupervisorClient` envia `SchedulingDecision` ao Runtime Supervisor (007). <br> 10. Resposta síncrona ao chamador com `SchedulingDecision` (outcome final observável no momento da resposta pode ser `ADMITTED` com decisão em progresso, ou já `SCHEDULED` se o `Submit` aguardar o ciclo — comportamento configurável). |
| **Fluxos alternativos** | A1. Se a fila já tem capacidade livre imediata, os passos 6–9 podem colapsar em um único ciclo de baixa latência (fast path). |
| **Exceções** | E1. Cota excedida → UC-002. E2. Orçamento insuficiente → UC-003. E3. PDP nega → UC-004. E4. Backpressure em nível *reject* → UC-008. |
| **Pós-condições** | Slot reservado; tarefa em `SCHEDULED` (ou `QUEUED` se aguardando capacidade); `SchedulingDecision` persistida; eventos emitidos; p99 ≤ 20 ms (NFR-001). |
| **Requisitos relacionados** | FR-001, FR-002, FR-003, FR-006, FR-007, FR-008, NFR-001, NFR-005. |

---

## UC-002 — Rejeitar por cota de concorrência excedida

| Campo | Conteúdo |
|-------|----------|
| **Ator primário** | Kernel (006). |
| **Pré-condições** | `sched:quota:{tenant}:concurrency` já no limite `max_concurrency`. |
| **Gatilho** | `Submit(SchedulingRequest)`. |
| **Fluxo principal** | 1. `SchedulingGateway` valida envelope. <br> 2. `AdmissionController` consulta `QuotaLedger`: cota esgotada. <br> 3. Transição `RECEIVED → REJECTED`; nenhuma reserva de slot é criada. <br> 4. Resposta com erro RFC 7807, `code = AIOS-SCHED-0001`, HTTP 429, `retriable = true`. <br> 5. Evento `task.execution.rejected` emitido com `{taskUrn, reasonCode: "AIOS-SCHED-0001", retriable: true}`. |
| **Fluxos alternativos** | A1. Se o cliente possui `deadline_at` distante e a política do tenant permite, o Kernel PODE re-tentar após `retryAfterMs` sugerido (não fornecido neste erro; ver UC-008 para backpressure). |
| **Exceções** | Nenhuma adicional — este é, ele mesmo, um fluxo de exceção de UC-001. |
| **Pós-condições** | Nenhum slot reservado; nenhum efeito colateral em filas; tarefa permanece responsabilidade do chamador (pode re-submeter). |
| **Requisitos relacionados** | FR-001, NFR-005. |

---

## UC-003 — Rejeitar por orçamento insuficiente

| Campo | Conteúdo |
|-------|----------|
| **Ator primário** | Kernel (006); ator secundário Cost-Optimizer (026). |
| **Pré-condições** | Orçamento residual do tenant (via 026) menor que `est_cost_usd`. |
| **Gatilho** | `Submit(SchedulingRequest)`. |
| **Fluxo principal** | 1. `CostEstimator` obtém `est_cost_usd` (fornecido no request ou recalculado). <br> 2. `BudgetClient` consulta orçamento residual em 026 (ou cache local, se dentro da margem de frescor aceitável). <br> 3. Orçamento insuficiente → transição `RECEIVED → REJECTED`, `code = AIOS-SCHED-0002`, HTTP 429, `retriable = true`. <br> 4. Evento `task.execution.rejected` emitido. |
| **Fluxos alternativos** | A1. Se 026 está indisponível, `CostEstimator` usa última estimativa cacheada com margem conservadora (brief §9); se ainda assim exceder o limite, aplica a mesma rejeição. |
| **Exceções** | E1. 026 indisponível e sem cache válido → decisão conservadora de rejeição para orçamentos no limite (degradação graciosa, brief §9). |
| **Pós-condições** | Nenhum slot reservado; nenhum consumo de orçamento debitado. |
| **Requisitos relacionados** | FR-001, FR-002. |

---

## UC-004 — Rejeitar por política (PDP nega admissão)

| Campo | Conteúdo |
|-------|----------|
| **Ator primário** | Kernel (006); ator secundário Policy Engine/PDP (022). |
| **Pré-condições** | Política vigente do tenant proíbe a admissão (ex.: agente suspenso por violação, classe de serviço não autorizada). |
| **Gatilho** | `Submit(SchedulingRequest)`. |
| **Fluxo principal** | 1. `PolicyClient` consulta o PDP (022) com o contexto da requisição (tenant, agente, classe, ação = `admit`). <br> 2. PDP retorna `deny`. <br> 3. Transição `RECEIVED → REJECTED`, `code = AIOS-SCHED-0004`, HTTP 403, `retriable = false`. <br> 4. Evento `task.execution.rejected` emitido; evento de auditoria correlato é gerado por 022/025 (fora do escopo do Scheduler). |
| **Fluxos alternativos** | A1. PDP indisponível (timeout) → tratado como `deny` (*default deny*, RFC-0001 §5.8), mesmo `code = AIOS-SCHED-0004`. |
| **Exceções** | Nenhuma adicional. |
| **Pós-condições** | Nenhum slot reservado; decisão registrada para auditoria com `policy_id`/`policy_version` referenciados. |
| **Requisitos relacionados** | FR-001, FR-020, NFR-012. |

---

## UC-005 — Placement em shard/nó com afinidade

| Campo | Conteúdo |
|-------|----------|
| **Ator primário** | Scheduler (ator de sistema, dentro do `DispatchLoop`). |
| **Pré-condições** | Tarefa admitida e no topo elegível da fila; `affinity`/anti-afinidade declaradas no `SchedulingRequest`. |
| **Gatilho** | Ciclo do `DispatchLoop` para o shard correspondente. |
| **Fluxo principal** | 1. `ShardRouter` calcula `shard = hash(tenant_id, agent_id) mod N`. <br> 2. `PlacementEngine` consulta `CapacityProvider` para nós/pools elegíveis dentro do shard. <br> 3. Aplica regras de afinidade (preferência) e anti-afinidade (exclusão) declaradas. <br> 4. Seleciona `PlacementTarget = {shard, node_id, runtime_pool}` com capacidade residual suficiente. <br> 5. Transição `QUEUED → SCHEDULED`; decisão registrada com `placement_target` preenchido. |
| **Fluxos alternativos** | A1. Nenhum nó satisfaz a anti-afinidade estrita, mas há capacidade em nó não preferido → fallback documentado e logado (aceita colocação subótima em vez de bloquear indefinidamente), salvo se a anti-afinidade for marcada como obrigatória pela política. |
| **Exceções** | E1. Nenhum shard/nó elegível em lugar algum → `code = AIOS-SCHED-0010`, HTTP 503, tarefa retorna para `QUEUED` para nova tentativa no próximo ciclo (ou expira por deadline). |
| **Pós-condições** | `PlacementTarget` determinístico e reproduzível para o mesmo `(tenant_id, agent_id, N)`. |
| **Requisitos relacionados** | FR-003. |

---

## UC-006 — Preemptar tarefa de menor prioridade

| Campo | Conteúdo |
|-------|----------|
| **Ator primário** | Scheduler (`PreemptionManager`), acionado por uma tarefa de maior prioridade sem capacidade livre; ator secundário Runtime Supervisor (007), Policy Engine (022). |
| **Pré-condições** | `scheduler.preemption.enabled = true`; existe tarefa `RUNNING`/`SCHEDULED` de prioridade efetiva inferior; vítima fora do período de cooldown (anti-thrashing). |
| **Gatilho** | `DispatchLoop` não encontra capacidade livre para tarefa de alta prioridade em `QUEUED`. |
| **Fluxo principal** | 1. `PreemptionManager` consulta `PriorityQueueStore`/estado de execução para identificar candidatas a vítima (menor `effective_priority` em execução no shard/nó alvo). <br> 2. Consulta `PolicyClient`→022 para autorizar a preempção (`allow`). <br> 3. Sinaliza suspensão ao `RuntimeSupervisorClient` com `grace_period_ms` (default 2000 ms). <br> 4. Transição da vítima `SCHEDULED/RUNNING → PREEMPTED`; evento `task.execution.preempted` emitido com `{taskUrn, byTaskUrn, gracePeriodMs}`. <br> 5. Runtime Supervisor confirma suspensão (`agent.lifecycle.suspended`); slot é liberado. <br> 6. Tarefa de alta prioridade prossegue para `SCHEDULED` (UC-001, passo 8). |
| **Fluxos alternativos** | A1. Múltiplas vítimas candidatas → seleciona o conjunto mínimo necessário para liberar recurso suficiente. |
| **Exceções** | E1. PDP nega a preempção → tarefa de alta prioridade permanece em `QUEUED` aguardando outra oportunidade (nenhuma preempção ocorre sem `allow`). E2. Vítima em cooldown → excluída da seleção; se não há outra vítima elegível, tarefa de alta prioridade aguarda. |
| **Pós-condições** | Vítima em `PREEMPTED` com re-enfileiramento futuro garantido (UC-007); overhead de decisão ≤ 50 ms (NFR-009). |
| **Requisitos relacionados** | FR-004, NFR-009. |

---

## UC-007 — Resumir tarefa preemptada

| Campo | Conteúdo |
|-------|----------|
| **Ator primário** | Scheduler (`ReconciliationWorker`/`DispatchLoop`); ator secundário Runtime Supervisor (007). |
| **Pré-condições** | Tarefa em estado `PREEMPTED`; capacidade ou prioridade relativa voltou a favorecer sua execução. |
| **Gatilho** | Ciclo de reavaliação periódico, mudança de capacidade (`cluster.capacity.changed`), ou liberação de slot por conclusão de outra tarefa. |
| **Fluxo principal** | 1. Tarefa preemptada é re-enfileirada preservando o aging acumulado (evita penalização dupla). <br> 2. Transição `PREEMPTED → QUEUED`. <br> 3. Ciclo normal de `DispatchLoop` (UC-001, passos 7–9) processa a tarefa quando atinge o topo elegível. <br> 4. Evento `task.execution.resumed` emitido ao ser efetivamente re-despachada. |
| **Fluxos alternativos** | A1. Se a tarefa tem `deadline_at` e este já expirou durante a preempção, ela transita para `EXPIRED` em vez de `QUEUED` (ver UC-010). |
| **Exceções** | Nenhuma adicional. |
| **Pós-condições** | Tarefa retoma progresso sem perda de posição relativa de fairness (aging preservado). |
| **Requisitos relacionados** | FR-004, FR-011. |

---

## UC-008 — Ativar backpressure sob saturação

| Campo | Conteúdo |
|-------|----------|
| **Ator primário** | Scheduler (`BackpressureController`), reagindo a saturação; ator secundário Kernel (006) como produtor afetado. |
| **Pré-condições** | Profundidade de fila, lag de JetStream ou pressão de capacidade cruzam `scheduler.backpressure.defer_threshold` (0.80) ou `reject_threshold` (0.95). |
| **Gatilho** | Avaliação contínua/periódica de saturação pelo `BackpressureController`. |
| **Fluxo principal** | 1. `BackpressureController` calcula nível corrente (`accept`/`defer`/`reject`) combinando sinais de `CapacityProvider`, `PriorityQueueStore` e lag JetStream. <br> 2. Ao cruzar um limiar, publica `scheduler.backpressure.changed` com `{level, retryAfterMs, shard}`. <br> 3. Em nível `reject`, novas admissões (`Submit`) retornam `code = AIOS-SCHED-0003`, HTTP 503, com `retryAfterMs`. <br> 4. Em nível `defer`, tarefas já admitidas podem transitar `QUEUED → DEFERRED` temporariamente, reentrando em `QUEUED` após a janela. |
| **Fluxos alternativos** | A1. Saturação localizada a um único shard não afeta admissão de outros shards (isolamento por partição). |
| **Exceções** | Nenhuma adicional — este é o próprio mecanismo de defesa do sistema. |
| **Pós-condições** | Produtor (Kernel) recebe sinal explícito para desacelerar; sistema evita colapso por sobrecarga; nível reportável via `GET /v1/scheduler/capacity`. |
| **Requisitos relacionados** | FR-005, FR-022, NFR-002. |

---

## UC-009 — Cancelar tarefa em fila

| Campo | Conteúdo |
|-------|----------|
| **Ator primário** | Kernel (006), em nome do agente/usuário que solicitou o cancelamento. |
| **Pré-condições** | Tarefa em `QUEUED` ou `SCHEDULED` (não terminal, não `RUNNING`). |
| **Gatilho** | `Cancel(request_id)` via gRPC ou `DELETE /v1/scheduler/requests/{request_id}`. |
| **Fluxo principal** | 1. `SchedulingGateway` valida a chamada (idempotente). <br> 2. Verifica estado atual da tarefa via `DecisionJournal`/`PriorityQueueStore`. <br> 3. Remove a tarefa da ZSET (se `QUEUED`) ou sinaliza cancelamento pré-dispatch (se `SCHEDULED` e ainda não confirmado `RUNNING`). <br> 4. Libera reserva de slot (`SlotReservation`). <br> 5. Transição para `CANCELLED` (terminal); evento `task.execution.rejected` com `reason_code = cancelled`. |
| **Fluxos alternativos** | A1. Chamar `Cancel` novamente para a mesma tarefa já `CANCELLED` retorna o mesmo resultado (idempotente), sem erro. |
| **Exceções** | E1. Tarefa já em `RUNNING` ou estado terminal (`COMPLETED`/`FAILED`/`EXPIRED`/`REJECTED`) → `code = AIOS-SCHED-0011`, HTTP 409 (autoridade de execução é do 007, o Scheduler não pode cancelar em `RUNNING`). |
| **Pós-condições** | Slot liberado; nenhuma execução ocorre para a tarefa cancelada. |
| **Requisitos relacionados** | FR-015. |

---

## UC-010 — Escalonar por deadline (EDF)

| Campo | Conteúdo |
|-------|----------|
| **Ator primário** | Scheduler (`PriorityCostEvaluator`, `DispatchLoop`), reagindo à presença de `deadline_at`. |
| **Pré-condições** | `scheduler.deadline.edf_enabled = true`; `SchedulingRequest.deadline_at` presente. |
| **Gatilho** | Admissão da tarefa (cálculo inicial de prioridade) e verificação periódica de deadlines em fila (`deadline_tick`). |
| **Fluxo principal** | 1. `PriorityCostEvaluator` incorpora a folga até o deadline como componente adicional da prioridade efetiva (mais próxima do deadline → maior urgência). <br> 2. `DispatchLoop` despacha tarefas respeitando essa ordem quando comparável dentro da mesma classe. <br> 3. Verificação periódica (`deadline_tick`) identifica tarefas em `QUEUED` cujo `now > deadline_at`. <br> 4. Transição `QUEUED → EXPIRED` (terminal); libera reserva; evento `task.execution.rejected` com `reason_code` de deadline vencido. |
| **Fluxos alternativos** | A1. Deadline já vencido no momento da admissão → rejeição imediata (`RECEIVED → REJECTED`, `code = AIOS-SCHED-0008`, HTTP 410), sem passar por `QUEUED`. |
| **Exceções** | Nenhuma adicional. |
| **Pós-condições** | Tarefas com deadline mais próximo têm precedência mensurável sobre tarefas comparáveis; nenhuma tarefa expirada é despachada. |
| **Requisitos relacionados** | FR-012. |

---

## UC-011 — Retry após falha do Runtime

| Campo | Conteúdo |
|-------|----------|
| **Ator primário** | Scheduler (`DispatchLoop`/`ReconciliationWorker`), reagindo a sinal de falha; ator secundário Runtime Supervisor (007). |
| **Pré-condições** | Tarefa em `RUNNING`; Runtime Supervisor reporta falha (`ReportRuntimeSignal` ou evento `agent.lifecycle.terminated` com falha). |
| **Gatilho** | Sinal de falha recebido do Runtime Supervisor. |
| **Fluxo principal** | 1. Transição `RUNNING → FAILED`; libera slot; evento `task.execution.failed` com `{taskUrn, errorCode, willRetry}`. <br> 2. Verifica política de retry (`scheduler.retry.max_attempts`, `scheduler.retry.backoff_base_ms`) e se a falha é retriável. <br> 3. Se retriável e tentativas < `max_attempts`: aguarda backoff exponencial com jitter e transita `FAILED → QUEUED` (re-enqueue). <br> 4. Se não retriável ou tentativas esgotadas: permanece em `FAILED` (terminal). |
| **Fluxos alternativos** | A1. Falha classificada como não-retriável pela política do tenant → não há passo 3, encerra em `FAILED`. |
| **Exceções** | Nenhuma adicional. |
| **Pós-condições** | Tarefa retriável reentra no ciclo normal de scheduling preservando `request_id` (idempotência); tentativas contabilizadas para auditoria. |
| **Requisitos relacionados** | FR-011, FR-014. |

---

## UC-012 — Reconciliar lease órfão

| Campo | Conteúdo |
|-------|----------|
| **Ator primário** | Scheduler (`ReconciliationWorker`), ator de sistema periódico. |
| **Pré-condições** | Tarefa em `SCHEDULED` cujo lease de dispatch (`scheduler.lease.ttl_ms`) expirou sem confirmação de início (`RUNNING`) pelo Runtime Supervisor. |
| **Gatilho** | Ciclo periódico do `ReconciliationWorker` ou expiração de TTL detectada em leitura. |
| **Fluxo principal** | 1. `ReconciliationWorker` varre leases com TTL expirado em `PriorityQueueStore`/`DecisionJournal`. <br> 2. Para cada lease órfão, verifica se o Runtime Supervisor confirma (heartbeat) ou não a materialização. <br> 3. Se não confirmada: libera a reserva de slot associada e transita a tarefa `SCHEDULED → QUEUED` (re-schedule), preservando `request_id` para evitar duplicação. <br> 4. Registra a reconciliação como evento interno de auditoria (proveniência). |
| **Fluxos alternativos** | A1. Runtime Supervisor confirma materialização atrasada (falso positivo de expiração) → reconciliação detecta e não duplica; mantém `RUNNING`. |
| **Exceções** | Nenhuma adicional. |
| **Pós-condições** | Nenhum trabalho é perdido silenciosamente; nenhuma execução duplicada ocorre (idempotência preservada); RTO ≤ 15 min (NFR-008). |
| **Requisitos relacionados** | FR-014, NFR-008. |

---

## UC-013 — Submissão duplicada (idempotência)

| Campo | Conteúdo |
|-------|----------|
| **Ator primário** | Kernel (006), re-tentando `Submit` após timeout de rede (comportamento at-least-once esperado). |
| **Pré-condições** | Uma `SchedulingRequest` com a mesma `Idempotency-Key`/`request_id` já foi processada anteriormente (com sucesso ou falha registrada). |
| **Gatilho** | Segunda chamada `Submit` com a mesma `Idempotency-Key`. |
| **Fluxo principal** | 1. `SchedulingGateway` consulta `IdempotencyStore` e encontra registro existente para a chave. <br> 2. Compara o payload da nova chamada com o payload original. <br> 3. Se idêntico: retorna o `response_snapshot` da primeira execução, sem nova reserva de slot nem novo enfileiramento. <br> 4. Nenhum novo evento de admissão é emitido (a idempotência é transparente ao chamador). |
| **Fluxos alternativos** | Nenhum. |
| **Exceções** | E1. Payload divergente para a mesma chave → `code = AIOS-SCHED-0006`, HTTP 409 (conflito de idempotência). |
| **Pós-condições** | Exatamente uma admissão efetiva por `request_id`, independentemente do número de retries (NFR-011). |
| **Requisitos relacionados** | FR-008, NFR-011. |

---

## UC-014 — Garantir fairness/anti-starvation

| Campo | Conteúdo |
|-------|----------|
| **Ator primário** | Scheduler (`PriorityCostEvaluator`/`PriorityQueueStore`), ator de sistema contínuo. |
| **Pré-condições** | Múltiplos tenants/classes competindo pelas mesmas filas/shards; um ou mais tenants com volume desproporcional de submissões. |
| **Gatilho** | Avaliação contínua de aging a cada ciclo de fila e monitoramento de vazão por tenant/classe. |
| **Fluxo principal** | 1. Cada tarefa em `QUEUED` acumula um boost de prioridade proporcional ao tempo de espera (`scheduler.aging.factor_per_sec`), limitado a `scheduler.aging.max_boost`. <br> 2. O `DispatchLoop` considera o score já ajustado por aging ao escolher o topo elegível. <br> 3. Métrica de fairness por tenant é calculada em janela de 60 s; se um tenant cai abaixo de `scheduler.fairness.min_share`, seu aging é acelerado (ou sua fila priorizada) até restabelecer o piso. |
| **Fluxos alternativos** | A1. Sistema em baixa contenção (todas as filas vazias/rasas) → aging tem efeito desprezível, comportamento reduz-se a priorização normal por `C`. |
| **Exceções** | Nenhuma adicional — a garantia é probabilística/estatística sobre a janela, não uma exceção pontual. |
| **Pós-condições** | Todo tenant ativo obtém ≥ `min_share` de vazão na janela observada (NFR-006); nenhuma tarefa fica presa indefinidamente. |
| **Requisitos relacionados** | FR-009, FR-024, NFR-006. |

---

## UC-015 — Atualizar pesos de política (α/β/γ)

| Campo | Conteúdo |
|-------|----------|
| **Ator primário** | Operador/SRE (029) ou sistema de governança de tenant, via API administrativa. |
| **Pré-condições** | Ator autenticado com role `scheduler-admin`; autorização do PDP (022) para a operação. |
| **Gatilho** | `PUT /v1/scheduler/policy` com novos valores de `alpha`,`beta`,`gamma` (ou outros parâmetros recarregáveis do brief §8) para um tenant/classe. |
| **Fluxo principal** | 1. `SchedulingGateway` (PEP) valida a chamada e consulta o PDP para autorizar a mutação de política. <br> 2. Valida que `alpha + beta + gamma ≈ 1` (normalização). <br> 3. Persiste a nova configuração (versão incrementada) e invalida caches locais relevantes. <br> 4. Novas decisões de `PriorityCostEvaluator` para o tenant/classe passam a usar os novos pesos em ≤ 5 s (NFR-017). |
| **Fluxos alternativos** | A1. Pesos fora da faixa válida (0–1) ou soma muito distante de 1 → requisição rejeitada com `AIOS-SCHED-0005` (request inválida). |
| **Exceções** | E1. PDP nega a mutação → `code = AIOS-SCHED-0004`, HTTP 403. |
| **Pós-condições** | Decisões subsequentes refletem os novos pesos; decisões passadas (já persistidas) mantêm os pesos originais para auditoria (imutabilidade, FR-010). |
| **Requisitos relacionados** | FR-018, FR-020, NFR-017. |

---

## UC-016 — Consultar proveniência da decisão

| Campo | Conteúdo |
|-------|----------|
| **Ator primário** | Operador/SRE (029), sistema de auditoria (025) ou sistema de aprendizado (023). |
| **Pré-condições** | `request_id` conhecido e pertencente ao tenant autenticado. |
| **Gatilho** | `GetDecision(request_id)` via gRPC ou `GET /v1/scheduler/requests/{request_id}`. |
| **Fluxo principal** | 1. `SchedulingGateway` valida a chamada e o escopo de tenant. <br> 2. Consulta `DecisionJournal` (PostgreSQL) pelo `request_id`. <br> 3. Retorna `SchedulingDecision` completa: `outcome`, `effective_priority`, `cost_score`, pesos, `placement_target`, `policy_id`/`policy_version`, `feature_vector`, `decided_at`. |
| **Fluxos alternativos** | Nenhum. |
| **Exceções** | E1. `request_id` inexistente ou pertencente a outro tenant → `code = AIOS-SCHED-0009`, HTTP 404. |
| **Pós-condições** | Nenhum efeito colateral (operação de leitura pura). |
| **Requisitos relacionados** | FR-017, FR-010. |

---

## UC-017 — Rebalancear shards sob mudança de capacidade (027)

| Campo | Conteúdo |
|-------|----------|
| **Ator primário** | Cluster (027), publicando mudança de topologia; ator de sistema Scheduler (`CapacityProvider`, `ShardRouter`). |
| **Pré-condições** | Mudança planejada e coordenada em `scheduler.shards.count` (N) ou na topologia de nós/pools disponíveis. |
| **Gatilho** | Evento `aios.<tenant>.cluster.capacity.changed` consumido pelo Scheduler. |
| **Fluxo principal** | 1. `CapacityProvider` atualiza sua visão de capacidade/topologia. <br> 2. Se `N` mudou, `ShardRouter` aplica a nova função de hashing consistente, remapeando apenas a fração necessária de chaves (~1/N). <br> 3. `DispatchLoop`s afetados retomam operação normal usando a nova topologia; ownership de shards por réplica é renegociado via leases Redis. <br> 4. Tarefas em `QUEUED` permanecem visíveis (nenhuma perda), mesmo que migrem de partição. |
| **Fluxos alternativos** | A1. Mudança de capacidade sem alteração de `N` (apenas nós entrando/saindo dentro dos shards existentes) → apenas `CapacityProvider` é atualizado, sem remapeamento de chaves. |
| **Exceções** | E1. Mudança não coordenada (abrupta) pode causar picos temporários de latência — mitigado por hashing consistente e janela de baixo tráfego recomendada (ADR-0092). |
| **Pós-condições** | 0 downtime de admissão (NFR-014); distribuição de carga entre shards se mantém equilibrada. |
| **Requisitos relacionados** | FR-011, FR-023, NFR-014. |

---

## UC-018 — Trocar política v1 (heurística) → v2 (RL) sem downtime

| Campo | Conteúdo |
|-------|----------|
| **Ator primário** | Operador/SRE (029), com apoio do time de Learning (023). |
| **Pré-condições** | `LearnedPolicy` (v2) validada offline e disponível; `scheduler.policy.engine` configurável em runtime. |
| **Gatilho** | Alteração de `scheduler.policy.engine` de `heuristic` para `rl` (globalmente ou por tenant, via canary). |
| **Fluxo principal** | 1. Operador altera a configuração para um subconjunto de tenants (canary). <br> 2. `PriorityCostEvaluator` passa a invocar `LearnedPolicy` através da interface `ISchedulingPolicy`, sem alterar a superfície de API/eventos. <br> 3. `FeatureExtractor` fornece o mesmo contrato de features usado por `HeuristicPolicy`, garantindo compatibilidade. <br> 4. Decisões subsequentes registram `policy_version = rl@<hash>` para auditoria e comparação. <br> 5. Métricas de qualidade/latência são comparadas entre tenants em `heuristic` e `rl` (A/B); rollout expande gradualmente. |
| **Fluxos alternativos** | A1. Detecção de regressão (ex.: aumento de rejeições, violação de `Q_min`) → rollback imediato para `heuristic` via a mesma chave de configuração, sem necessidade de deploy. |
| **Exceções** | Nenhuma adicional — a própria interface `ISchedulingPolicy` existe para tornar essa troca segura. |
| **Pós-condições** | Nenhuma mudança de contrato externo observável pelos consumidores (Kernel, Runtime Supervisor); proveniência distingue claramente qual política decidiu cada tarefa. |
| **Requisitos relacionados** | FR-013, NFR-013. |

---

## Diagrama de contexto dos casos de uso (ASCII, referência)

```
                         ┌─────────────────────────────┐
   Kernel(006) ─────────▶│  UC-001 Submit (feliz)       │
                         │  UC-002 Rejeitar (cota)      │
                         │  UC-003 Rejeitar (orçamento) │
                         │  UC-004 Rejeitar (política)  │
                         │  UC-009 Cancelar             │
                         │  UC-013 Duplicata (idemp.)   │
                         │  UC-016 GetDecision           │
                         └───────────┬─────────────────┘
                                     │
                                     ▼
                      ┌───────────────────────────────┐
                      │   SCHEDULER (009) — núcleo     │
                      │  UC-005 Placement               │
                      │  UC-006 Preempção                │
                      │  UC-007 Resume pós-preempção      │
                      │  UC-008 Backpressure              │
                      │  UC-010 EDF/deadline               │
                      │  UC-011 Retry pós-falha              │
                      │  UC-012 Reconciliação de lease         │
                      │  UC-014 Fairness/anti-starvation        │
                      │  UC-017 Rebalanceamento de shards         │
                      │  UC-018 Troca de política (v1→v2)           │
                      └───────────┬─────────────────┬───────────────┘
                                  │                 │
                                  ▼                 ▼
                     Runtime Supervisor(007)   Policy(022)/Cost-Optimizer(026)/Cluster(027)
                                  │
                                  ▼
                      UC-015 Atualizar pesos (Operador/SRE 029, via 022)
```

---

## Referências Cruzadas

- Máquina de estados subjacente: [`_DESIGN_BRIEF.md`](./_DESIGN_BRIEF.md) §4 (base para `StateMachine.md`, batch posterior).
- Requisitos funcionais: [`FunctionalRequirements.md`](./FunctionalRequirements.md)
- Requisitos não-funcionais: [`NonFunctionalRequirements.md`](./NonFunctionalRequirements.md)
- Matriz de rastreabilidade: [`Requirements.md`](./Requirements.md) §7
- Contratos centrais: [`../003-RFC/RFC-0001-Architecture-Baseline.md`](../003-RFC/RFC-0001-Architecture-Baseline.md)

---

*Fim de `UseCases.md` — Módulo 009-Scheduler.*
