---
Documento: FAQ
Módulo: 009-Scheduler
Status: Draft
Versão: 0.1
Última atualização: 2026-07-20
Responsável (RACI-A): Arquiteto do Módulo 009-Scheduler
ADRs relacionados: ADR-0090, ADR-0091, ADR-0092, ADR-0093, ADR-0094, ADR-0095, ADR-0096, ADR-0097
RFCs relacionados: RFC-0001, RFC-0090, RFC-0091
Depende de: 006-Kernel, 022-Policy, 026-Cost-Optimizer, 027-Cluster
---

# 009-Scheduler — FAQ

> Este documento reúne perguntas frequentes, armadilhas comuns e
> esclarecimentos de escopo sobre o Scheduler cognitivo. Ele NÃO substitui
> `./_DESIGN_BRIEF.md`; em caso de conflito aparente, o brief prevalece.

---

## 1. Conceituais

### 1.1 O Scheduler é um orquestrador de workflows?

**Não.** O Scheduler decide **quando/onde/com qual prioridade** uma tarefa já
definida executa. Ele não decompõe objetivos em passos, não gerencia sagas nem
compensações de longo prazo — isso é responsabilidade de `012-Planning` e
`014-Workflow`. Uma armadilha comum é tentar usar o campo `affinity` de
`SchedulingRequest` para expressar dependências entre tarefas (ex.: "rode B
depois de A"); esse tipo de dependência DEVE ser modelado no Workflow Engine, e
cada passo submetido individualmente ao Scheduler quando estiver pronto para
execução.

### 1.2 Qual a diferença entre "prioridade" (`priority_hint`) e "prioridade
efetiva" (`effective_priority`)?

`priority_hint` é uma **sugestão não vinculante** do chamador (0–100, default
50) enviada em `SchedulingRequest`. `effective_priority` é o valor **computado**
pelo `PriorityCostEvaluator` a partir de `C = α·L + β·$ + γ·E`, ajustado por
classe de serviço, folga de deadline (EDF) e aging anti-*starvation* — é esse
valor, e não o *hint*, que determina a posição na fila (score do ZSET). Um
`priority_hint` alto NÃO garante execução imediata se custo, orçamento ou
capacidade não permitirem.

### 1.3 O Scheduler decide qual modelo de LLM usar?

**Não.** Essa é a responsabilidade do `017-Model-Router`. O Scheduler decide
*quando/onde* a tarefa executa; o Model Router decide, dentro da execução, qual
modelo atende a tarefa sob custo/qualidade/latência. O `est_cost_usd` /
`est_tokens` que o Scheduler recebe em `SchedulingRequest` é uma **estimativa**
fornecida por `026-Cost-Optimizer` (que por sua vez pode consultar 017), não uma
decisão do próprio Scheduler.

### 1.4 O Scheduler é o PDP (Policy Decision Point)?

**Não.** O `SchedulingGateway` é o **PEP** (Policy Enforcement Point) do
módulo: ele intercepta toda `SchedulingRequest`/`PreemptRequest` e consulta o
**PDP real, que é o `022-Policy`**, via `PolicyClient`. O Scheduler nunca
implementa suas próprias regras de RBAC/ABAC; ele apenas aplica o veredito do
PDP (*default deny* — RFC-0001 §5.8). Se 022 estiver indisponível, a admissão
é negada por padrão (erro `AIOS-SCHED-0004`), nunca liberada por omissão.

### 1.5 Um agente que está "suspenso" (`Cold Agent`) precisa de tratamento
especial no Scheduler?

Não no sentido de lógica de decisão: uma `SchedulingRequest` para um agente
frio entra na fila normalmente, com prioridade efetiva calculada da mesma
forma. A única diferença é operacional — a *materialização* do placement
(rehidratar o agente) é do `007-Agent-Runtime` e pode levar até ~250 ms; o
Scheduler apenas garante o slot e o placement, sem modelar o custo de
"descongelamento" como uma dimensão separada da função `C`.

---

## 2. Admissão e Cotas

### 2.1 O que acontece se o orçamento do tenant estiver esgotado no meio do
processamento de uma `SchedulingRequest`?

A verificação de orçamento é feita **no momento da admissão**, contra a
estimativa (`est_cost_usd`) fornecida por `026-Cost-Optimizer`. Se insuficiente,
a requisição é rejeitada com `AIOS-SCHED-0002` (429, retriável). O Scheduler
**não** monitora o gasto real durante a execução da tarefa — isso é
continuamente reconciliado por 026; o Scheduler apenas reage a
`aios.<tenant>.cost.budget.updated` para refletir orçamento atualizado em
admissões futuras.

### 2.2 Por que uma `SchedulingRequest` idêntica, reenviada duas vezes, não
gera duas execuções?

Porque toda mutação do Scheduler (`Submit`, `Cancel`, `Preempt`,
`ReportRuntimeSignal`) é idempotente por `Idempotency-Key`/`request_id`
(herdado de RFC-0001 §5.5). O `IdempotencyStore` retorna o **mesmo**
`SchedulingDecision` para a mesma chave, sem nova reserva de slot nem nova
admissão (NFR-011). Isso é essencial porque o transporte é *at-least-once* —
retries de rede DEVEM ser seguros por construção.

**Armadilha comum:** reenviar a mesma `SchedulingRequest` com o **mesmo**
`Idempotency-Key` mas com **payload diferente** (ex.: `priority_hint` mudou)
não é tratado como uma nova decisão silenciosa — é um conflito de
idempotência e retorna `AIOS-SCHED-0006` (409). Se o payload precisa mudar,
use uma nova `Idempotency-Key`.

### 2.3 Uma tarefa pode ser admitida mesmo com capacidade zero em todos os
shards?

Não diretamente para execução, mas **sim para a fila**. Admissão (`ADMITTED`)
verifica cota, orçamento e política — não exige capacidade disponível *agora*.
A falta de capacidade é tratada no *dispatch* (transição `QUEUED → SCHEDULED`),
onde a tarefa aguarda até haver folga, sujeita a backpressure e ao aging.
Apenas quando **nenhum shard elegível** existe (ex.: afinidade impossível de
satisfazer) o erro é `AIOS-SCHED-0010` (503).

### 2.4 O que é reservado durante a admissão e quando é liberado?

Um slot é **reservado atomicamente** (`sched:reservation:{request_id}` com TTL)
no momento em que `AdmissionController` aceita a requisição, antes mesmo do
enfileiramento. A reserva é confirmada quando a tarefa é efetivamente
despachada (`SCHEDULED`) ou liberada em caso de `REJECTED`/`CANCELLED`/`EXPIRED`.
Esse desenho evita a janela de corrida em que duas requisições concorrentes
veem "1 slot livre" e ambas o consomem — a causa clássica de *overcommit*.

---

## 3. Prioridade, Custo e Fairness

### 3.1 Os pesos `α`, `β`, `γ` são globais ou por tenant?

São configuráveis **por tenant** (`scheduler.weights.alpha/beta/gamma`),
recarregáveis em runtime, e DEVEM satisfazer `α + β + γ ≈ 1` (normalizados na
leitura). Um tenant que prioriza latência sobre custo (ex.: aplicações
interativas) pode configurar `α` alto; um tenant *batch*-heavy tipicamente
aumenta `β`. Alterações de peso DEVEM ser autorizadas via `022-Policy` — não
são um parâmetro livre de auto-serviço sem controle.

### 3.2 Um tenant grande pode "afogar" um tenant pequeno na fila?

**Não, por design.** `scheduler.fairness.min_share` (default 1%) garante que
todo tenant ativo obtenha vazão mínima em uma janela de 60 s (NFR-006). O
mecanismo prático é o **aging**: o score de espera de uma tarefa é
incrementado ao longo do tempo (`scheduler.aging.factor_per_sec`, limitado a
`scheduler.aging.max_boost`), o que eventualmente a eleva ao topo da fila
mesmo competindo com tenants de maior volume.

### 3.3 O que é EDF e quando ele se aplica?

*Earliest Deadline First* — quando `deadline_at` está presente na
`SchedulingRequest` e `scheduler.deadline.edf_enabled=true` (default), a
prioridade efetiva incorpora a folga até o prazo, favorecendo tarefas cujo
deadline está mais próximo, mesmo que sua classe de serviço nominal seja
inferior. Se o deadline já venceu no momento da admissão, a requisição é
rejeitada imediatamente com `AIOS-SCHED-0008` (410) — nunca fica presa na fila
apenas para expirar depois (`EXPIRED`).

### 3.4 O que significa "qualidade mínima" (`quality_floor` / `Q_min`) na
prática do Scheduler?

É uma restrição, não um objetivo a maximizar: o Scheduler só aceita
configurações de placement/prioridade que **não violem** `Q ≥ Q_min` (herdado
do modelo formal de otimização da Visão Global). Se, sob as restrições atuais
de custo/capacidade, o `quality_floor` requisitado for inatingível, a resposta
é `AIOS-SCHED-0007` (422) — o Scheduler nunca "silenciosamente" entrega
qualidade abaixo do piso solicitado.

---

## 4. Placement e Sharding

### 4.1 Por que sharding por `hash(tenant_id, agent_id) mod N` e não round-robin?

Porque precisamos de **determinismo**: a mesma dupla (tenant, agente) sempre
mapeia para o mesmo shard, o que permite que o estado relacionado a esse
agente (fila, cotas, afinidade) permaneça localizado, evitando coordenação
cruzada entre réplicas. Round-robin distribuiria carga de forma mais
uniforme no curto prazo, mas quebraria a localidade necessária para
`PriorityQueueStore` operar sem locks globais (ver `Scalability.md`).

### 4.2 O que acontece quando o número de shards (`N`) muda?

`scheduler.shards.count` **não é recarregável isoladamente** — mudar `N`
exige rebalanceamento coordenado com `027-Cluster` usando *hashing consistente*
para minimizar migração de estado. Um `N` alterado sem essa coordenação
quebraria o determinismo do `ShardRouter` e poderia levar a filas "órfãs" (uma
tarefa cujo shard antigo já não é servido por nenhuma réplica).

### 4.3 Afinidade/anti-afinidade é uma garantia rígida?

Afinidade/anti-afinidade (`affinity` em `SchedulingRequest`) é um **critério de
placement**, avaliado pelo `PlacementEngine` dentro dos shards/nós já
determinados pelo `ShardRouter` e sujeitos à capacidade residual real. Se a
afinidade solicitada não puder ser satisfeita por nenhum destino elegível, o
resultado é `AIOS-SCHED-0010` (nenhum shard/nó elegível) — não é
"melhor esforço" silencioso.

---

## 5. Preempção e Backpressure

### 5.1 Preempção mata a tarefa vítima?

**Não.** Preempção é, por design, **compensável**: a vítima recebe aviso e um
*grace period* (`scheduler.preemption.grace_period_ms`, default 2000 ms) antes
da suspensão efetiva; seu estado é preservado e ela retorna a `QUEUED` para ser
retomada quando capacidade/prioridade permitirem (transição
`PREEMPTED → QUEUED` preservando o aging acumulado). Matar definitivamente uma
tarefa é uma operação distinta (`Cancel`), não uma consequência de preempção.

### 5.2 Preempção pode entrar em loop (a preempta b, b preempta a)?

Esse fenômeno — *thrashing* de preempção — é mitigado por **histerese**:
cooldown por tarefa e proteção de vítimas recém-agendadas contra nova
preempção imediata (ver modos de falha em `_DESIGN_BRIEF.md` §9 e ADR-0094). A
métrica `aios_scheduler_preemption_ms`/taxa de preempções por segundo é
monitorada como sinal de alerta caso a histerese não seja suficiente.

### 5.3 Qual a diferença entre os três níveis de backpressure?

| Nível | Limiar (default) | Comportamento |
|-------|-------------------|-----------------|
| `accept` | pressão < 0.80 | Admissão segue fluxo normal. |
| `defer` | 0.80 ≤ pressão < 0.95 | Tarefas elegíveis são adiadas (`DEFERRED`) e reagendadas; sinaliza `Retry-After` mais curto. |
| `reject` | pressão ≥ 0.95 | Novas admissões são recusadas com `AIOS-SCHED-0003` (503, retriável) e `Retry-After` mais longo. |

Isso permite que produtores (Kernel, Gateway) recuem de forma **gradual** em
vez de sofrerem falhas abruptas apenas quando o sistema já colapsou.

### 5.4 O cliente deve implementar lógica própria de retry ao receber
`Retry-After`?

Sim — o Scheduler **sinaliza** a janela recomendada; ele não a impõe via
bloqueio de conexão. Clientes (Kernel, SDKs) DEVEM respeitar `Retry-After`
(ou o campo equivalente `retryAfterMs` do envelope de erro RFC 7807herdado de
RFC-0001 §5.4) com *backoff* e *jitter*, para evitar que um "rejeitar em massa"
vire uma nova onda sincronizada de retries (efeito manada).

---

## 6. Eventos, API e Integração

### 6.1 Por que `Submit` retorna `SchedulingDecision` e não apenas um "OK"?

Porque a submissão **é** uma decisão: mesmo no caminho feliz, o chamador
precisa saber o `decision_id`, o `outcome` (`ADMITTED`/`SCHEDULED`/...), a
`effective_priority` e, quando aplicável, o `placement_target`. Reduzir a
resposta a um booleano esconderia informação necessária para observabilidade e
para o próprio cliente decidir se deve aguardar, cancelar ou re-submeter com
outros parâmetros.

### 6.2 Quais eventos devo consumir para saber que minha tarefa está de fato
rodando?

`aios.<tenant>.task.execution.scheduled` indica que o Scheduler despachou a
decisão ao Runtime Supervisor — **não** que o agente já está rodando. A
confirmação real vem indiretamente: o Scheduler observa
`aios.<tenant>.agent.lifecycle.spawned` (produzido por 007) para transitar seu
próprio modelo interno de `SCHEDULED` para `RUNNING`. Consumidores externos que
precisam da confirmação definitiva de execução DEVEM assinar os eventos de
`007-Agent-Runtime`, não inferir "rodando" apenas a partir de `scheduled`.

### 6.3 O evento `scheduler.decision.recorded` duplica `task.execution.*`?

Não — eles têm propósitos diferentes. `task.execution.{admitted,scheduled,...}`
comunica o **resultado funcional** (o que aconteceu com a tarefa) para
consumidores operacionais. `scheduler.decision.recorded` carrega a
**proveniência completa** da decisão (política/versão, `feature_vector`,
referência) para consumo por `025-Audit` e `023-Learning` — um propósito de
auditoria/treino, não de fluxo funcional. Um consumidor típico de fluxo de
negócio NÃO precisa assinar `decision.recorded`.

### 6.4 Posso usar REST para o caminho quente de submissão em produção?

`POST /v1/scheduler/requests` existe e é funcionalmente equivalente a `Submit`
gRPC, mas o caminho **recomendado para produção interna** é gRPC
(`aios.scheduler.v1.SchedulerService.Submit`), por overhead de serialização
menor e alinhamento com a meta de p99 ≤ 20 ms. REST é preferencial para uso
administrativo, observabilidade e integrações externas via Gateway.

---

## 7. Segurança e Multi-tenancy

### 7.1 Um chamador pode submeter uma tarefa "em nome de" outro tenant?

Não. `X-AIOS-Tenant` DEVE casar com o tenant do contexto autenticado
(claim assinada, RFC-0001 §5.6); qualquer divergência é rejeitada com
`AIOS-SCHED-0004`, independentemente do conteúdo de `tenant_id` dentro do
payload. Essa regra é herdada de RFC-0001 §6 e não é uma decisão local do
Scheduler.

### 7.2 Preempção dirigida (`Preempt` administrativo) exige alguma role
especial?

Sim — `scheduler-admin`, distinta de `scheduler-submit` (usada para
`Submit`/`Cancel`). Toda preempção, dirigida ou automática, passa pelo PDP
(022) antes de ser executada; não existe caminho de preempção que ignore o
PEP do `SchedulingGateway`.

### 7.3 O `feature_vector` de uma decisão pode conter dados pessoais (PII)?

**Não deve.** Por princípio de minimização (RFC-0001 §7), o `feature_vector`
carrega sinais numéricos derivados (carga, fila, custo estimado, folga de
SLA, histórico agregado) — nunca payload bruto da tarefa. O payload em si
nunca trafega pelo Scheduler; `payload_ref` aponta para MinIO, e é
responsabilidade do produtor da tarefa garantir que esse payload, quando
referenciado por sistemas a jusante, siga as políticas de LGPD/GDPR do
`021-Security`/`025-Audit`.

---

## 8. Evolução e Extensibilidade

### 8.1 Trocar `scheduler.policy.engine` de `heuristic` para `rl` muda a API
externa?

**Não pode mudar**, por desenho: `ISchedulingPolicy` é a interface estável que
isola a heurística v1 (`HeuristicPolicy`) de uma futura política aprendida v2
(`LearnedPolicy`, RL). API gRPC/REST, schema de eventos e formato de
`SchedulingDecision` permanecem idênticos — apenas a lógica interna que produz
`effective_priority`/`placement_target` muda. Essa é a garantia central de
FR-013 e do RFC-0091.

### 8.2 Como uma política aprendida (v2) seria auditada da mesma forma que a
heurística (v1)?

Da mesma maneira: toda decisão, independentemente da política que a produziu,
grava `policy_id`/`policy_version` (`heuristic@1` vs. `rl@<hash>`) e
`feature_vector` no `DecisionJournal`. Isso significa que a proveniência de
uma decisão de RL é tão auditável quanto a de uma heurística — o contrato de
auditoria não é relaxado para acomodar aprendizado.

### 8.3 Adicionar uma nova classe de serviço (além de `INTERACTIVE`, `BATCH`,
`BACKGROUND`, `SYSTEM`) é uma mudança compatível?

É uma mudança que DEVE ser tratada como evolução de schema, não como *hotfix*:
`service_class` é um enum que afeta pesos e política de preempção. Adicionar
um valor novo é aditivo (compatível) se consumidores existentes tolerarem
valores desconhecidos (RFC-0001 §5.7); redefinir o significado de uma classe
existente é incompatível e exige nova versão de RFC/ADR do módulo.

---

## 9. Armadilhas Comuns (resumo rápido)

| Armadilha | Por que é um erro | Correção |
|-----------|---------------------|----------|
| Tratar `priority_hint` como prioridade final | É apenas uma sugestão; a decisão real é `effective_priority`. | Observar `SchedulingDecision.effective_priority`, não o *hint* enviado. |
| Reenviar `SchedulingRequest` com novo payload e mesma `Idempotency-Key` | Gera conflito (`AIOS-SCHED-0006`), não atualização silenciosa. | Gerar nova `Idempotency-Key` para requisições semanticamente diferentes. |
| Assumir que `scheduled` significa "já executando" | `scheduled` é o despacho ao Runtime; a confirmação real vem de 007. | Assinar `agent.lifecycle.spawned` para confirmação de execução. |
| Ignorar `Retry-After` sob backpressure | Gera efeito manada e agrava a saturação. | Implementar backoff com jitter respeitando o valor sinalizado. |
| Modelar dependências entre tarefas via `affinity` | `affinity` é sobre placement físico, não sobre ordem lógica. | Usar `014-Workflow` para modelar dependências e submeter cada passo quando pronto. |
| Assumir que aumentar `priority_hint` contorna orçamento insuficiente | Prioridade não substitui a restrição de orçamento (`Q_min`/`B`). | Ajustar orçamento via `026` ou aceitar deferimento/rejeição. |
| Mudar `scheduler.shards.count` sem coordenar com `027-Cluster` | Quebra o determinismo do sharding e pode gerar filas órfãs. | Sempre coordenar rebalanceamento com hashing consistente. |

---

## 10. Referências

- Design Brief: `./_DESIGN_BRIEF.md`
- Visão do módulo: `./Vision.md`
- Máquina de estados: `./StateMachine.md`
- Contratos centrais: `../003-RFC/RFC-0001-Architecture-Baseline.md`
- Glossário: `../040-Glossary/Glossary.md`

---

*Fim do FAQ do Módulo 009-Scheduler.*
