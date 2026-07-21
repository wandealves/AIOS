---
Documento: Vision
Módulo: 009-Scheduler
Status: Draft
Versão: 0.1
Última atualização: 2026-07-20
Responsável (RACI-A): Arquiteto do Módulo 009-Scheduler
ADRs relacionados: ADR-0090, ADR-0091, ADR-0092, ADR-0093, ADR-0094, ADR-0095, ADR-0096, ADR-0097
RFCs relacionados: RFC-0001, RFC-0090, RFC-0091
Depende de: 006-Kernel, 022-Policy, 026-Cost-Optimizer, 027-Cluster
---

# 009-Scheduler — Vision

> "Assim como o CFS (*Completely Fair Scheduler*) do Linux decide qual processo
> ganha a CPU e por quanto tempo, o Scheduler do AIOS decide qual agente/tarefa
> executa, quando, onde e a que custo — mas fá-lo consciente de orçamento,
> qualidade e SLA, não apenas de *time slices*."

---

## 1. Propósito do Módulo

O **009-Scheduler** é o **scheduler cognitivo** do plano de controle do AIOS.
Ele responde, para cada unidade de trabalho submetida ao sistema, à pergunta
canônica formulada no `../000-Vision/Vision.md` (§8) e especializada aqui:

> **"Qual tarefa/agente executa, quando, onde (placement), com qual prioridade
> e a que custo?"**

Isso ocorre sob restrições simultâneas de **cota** (quantos slots de
concorrência o tenant/agente já consome), **orçamento** (quanto o tenant PODE
gastar em inferência), **SLA/deadline** (quando o resultado é necessário) e
**capacidade** (quantos nós/shards/pools têm folga). O Scheduler é, ao mesmo
tempo:

- O **guardião da admissão** do sistema — nenhuma tarefa entra em execução sem
  passar pelo seu crivo (`AdmissionController`).
- O **árbitro de prioridade e custo** — computa a prioridade efetiva via a
  função multiobjetivo `C = α·L + β·$ + γ·E` herdada do modelo formal de
  otimização do AIOS (`../000-Vision/Vision.md` §8).
- O **agente de placement** — decide em qual shard/nó/pool a tarefa materializa,
  via sharding determinístico.
- O **mecanismo de preempção** — realoca capacidade de tarefas de menor
  prioridade efetiva para tarefas mais urgentes/importantes, de forma segura e
  compensável.
- A **válvula de backpressure** do sistema — o ponto onde a saturação vira
  sinal explícito para os produtores (Kernel, Gateway), em vez de degradação
  silenciosa.

O Scheduler **decide**; ele **não executa**. A materialização da decisão
(*spawn*/*suspend*/*kill* de um Agent Runtime) é responsabilidade exclusiva do
**Runtime Supervisor (`007-Agent-Runtime`)**. Essa separação — decisão vs.
execução — é o mesmo princípio de design que separa, em um SO clássico, o
scheduler do *dispatcher* de baixo nível, e reflete o Princípio de Design nº 1
da Visão Global (Separação Plano de Controle / Plano de Dados).

```
   Kernel(006) ── "pede slot para Task" (gRPC) ─────────▶ Scheduler(009)
   Policy(022) ◀── PDP "pode admitir/preemptar?" ───────  Scheduler(009)
   Cost-Optimizer(026) ◀── orçamento/estimativa custo ──  Scheduler(009)
   Cluster(027) ──── inventário de capacidade/shards ───▶ Scheduler(009)
   Scheduler(009) ── "SchedulingDecision (place/preempt)" ▶ Runtime Supervisor(007)
   Scheduler(009) ── eventos aios.<tenant>.task.execution.* ▶ NATS/JetStream
```

---

## 2. Problema que Resolve

Sem um scheduler cognitivo dedicado, um sistema multi-agente em escala sofre de
falhas estruturais que nenhuma quantidade de capacidade computacional extra
resolve sozinha:

| # | Problema sem o Scheduler | Consequência prática |
|---|---------------------------|-----------------------|
| PS-01 | Admissão implícita (tudo que chega, executa) | *Noisy neighbor*: um tenant/agente consome toda a capacidade e degrada os demais; nenhuma proteção de cota. |
| PS-02 | Prioridade fixa ou inexistente | Tarefas `INTERACTIVE` (usuário aguardando) competem em pé de igualdade com `BATCH`/`BACKGROUND`; latência percebida inaceitável. |
| PS-03 | Custo tratado como efeito colateral, não como restrição de decisão | Orçamento estourado silenciosamente; nenhuma forma de preferir "mais barato, ainda dentro do SLA". |
| PS-04 | Placement ad-hoc ou aleatório | Hot spots de carga; nós saturados enquanto outros ociosos; nenhuma afinidade/anti-afinidade. |
| PS-05 | Ausência de preempção | Sob contenção, a única saída é rejeitar tudo ou deixar enfileirar indefinidamente; nenhuma forma de "ceder espaço" a trabalho mais crítico. |
| PS-06 | Ausência de backpressure explícita | O sistema degrada silenciosamente (latência cresce, filas explodem) até falhar em cascata, em vez de sinalizar `Retry-After` cedo. |
| PS-07 | Nenhuma trilha de decisão | Impossível explicar por que uma tarefa foi aceita/rejeitada/adiada; auditoria e aprendizado (023) ficam sem dado de proveniência. |
| PS-08 | Starvation de tenants pequenos | Tenants de baixo volume nunca progridem quando tenants grandes saturam a fila, sem garantia de vazão mínima. |
| PS-09 | Acoplamento entre política de decisão e API/eventos | Trocar a heurística de scheduling por uma política aprendida (RL) exige reescrever clientes e consumidores de eventos. |

O Scheduler resolve estes problemas com **admission control explícito**,
**score multiobjetivo de prioridade/custo**, **placement determinístico**,
**preempção compensável**, **backpressure em três níveis**, **proveniência de
decisão** e uma **interface de política estável** (`ISchedulingPolicy`) que
desacopla a heurística atual de uma política aprendida futura.

### 2.1 Hipótese central do módulo

> Tratar scheduling não como uma fila FIFO com prioridade estática, mas como um
> **problema de decisão sequencial sob múltiplas restrições** (cota, orçamento,
> SLA, capacidade), resolvido inicialmente por heurística determinística (v1) e,
> evolutivamente, por uma política aprendida (v2, RL) — **sem que a API externa,
> o modelo de eventos ou os contratos de dados mudem**.

---

## 3. Escopo

### 3.1 Escopo — Dentro (In)

| # | Dentro do escopo |
|---|-------------------|
| IN-01 | Admission control de `SchedulingRequest` (cota, orçamento, capacidade, PDP). |
| IN-02 | Cálculo de prioridade/score multiobjetivo `C = α·L + β·$ + γ·E`. |
| IN-03 | Placement determinístico por sharding (`hash(tenant,agent) mod N`), afinidade e anti-afinidade. |
| IN-04 | Preempção segura e compensável de tarefas/agentes de menor prioridade efetiva. |
| IN-05 | Backpressure (accept/defer/reject) integrada a JetStream, com `Retry-After`. |
| IN-06 | Filas de prioridade por (tenant, classe, shard) com aging anti-*starvation*. |
| IN-07 | Enforcement de SLA/deadline (EDF-aware) para tarefas com prazo. |
| IN-08 | Registro de proveniência de cada decisão (features, score, política/versão). |
| IN-09 | Reavaliação/re-*scheduling* diante de mudança de capacidade, preempção externa ou expiração. |
| IN-10 | Interface `ISchedulingPolicy` estável, permitindo evoluir de heurística (v1) para política aprendida RL (v2). |
| IN-11 | Idempotência de decisão por `Idempotency-Key`/`request_id`. |
| IN-12 | Reconciliação entre estado de fila e estado real do Runtime (leases órfãos). |

### 3.2 Escopo — Fora (Out) e Dono Correto

| # | Fora do escopo | Dono correto |
|---|------------------|---------------|
| OUT-01 | Executar o loop cognitivo/ReAct do agente. | `007-Agent-Runtime` |
| OUT-02 | Criar/suspender/matar processos de runtime (materializar a decisão de placement). | `007-Agent-Runtime` (Runtime Supervisor) |
| OUT-03 | Definir/avaliar políticas RBAC/ABAC — o Scheduler é **PEP**, não **PDP**. | `022-Policy` |
| OUT-04 | Escolher o modelo LLM ou falar com provedores de inferência. | `017-Model-Router` |
| OUT-05 | Calcular preço/faturar; ser fonte de verdade de orçamento. | `026-Cost-Optimizer` |
| OUT-06 | Persistir memória cognitiva, contexto ou conhecimento. | `010-Memory` / `011-Context` / `018-Knowledge` / `019-GraphRAG` |
| OUT-07 | Prover HA/replicação de NATS/PostgreSQL/Redis; failover de infraestrutura. | `027-Cluster` |
| OUT-08 | Gerenciar cotas de token dentro da própria inferência (rate de tokens do LLM). | `026-Cost-Optimizer` / `017-Model-Router` |
| OUT-09 | Manter a trilha de auditoria imutável (apenas emitir os eventos que 025 consome). | `025-Audit` |
| OUT-10 | Decidir *o que* a tarefa faz (o plano em si); apenas *quando/onde/prioridade*. | `012-Planning` / `014-Workflow` |

A fronteira acima é normativa: nenhum documento derivado deste módulo PODE
atribuir ao Scheduler uma responsabilidade listada em "Fora do escopo", e
nenhum outro módulo PODE reimplementar admission control, placement ou
preempção de tarefas — isso duplicaria autoridade e quebraria a garantia de
não-*overcommit* (NFR-005).

---

## 4. Personas Atendidas

| Persona | Necessidade em relação ao Scheduler | Como o módulo atende |
|---------|--------------------------------------|------------------------|
| **Desenvolvedor de Agentes** | Submeter tarefas e obter uma decisão previsível e rápida (p99 ≤ 20 ms). | `Submit` gRPC/REST idempotente, com `service_class`/`priority_hint`/`deadline_at` opcionais. |
| **Operador de Plataforma (SRE)** | Entender por que o sistema está rejeitando/adiando trabalho, e agir sobre isso. | Eventos `scheduler.backpressure.changed`, endpoint `GET /v1/scheduler/capacity`, métricas de fila e preempção. |
| **Arquiteto Enterprise** | Garantir que nenhuma tarefa execute sem autorização de política e sem cota respeitada. | `AdmissionController` como PEP obrigatório de 022; reservas atômicas sem *overcommit*. |
| **Gestor de Custo (FinOps)** | Impedir que tarefas excedam orçamento e priorizar o custo-benefício. | Peso `β` na função `C`, integração com `026-Cost-Optimizer`, erro `AIOS-SCHED-0002`. |
| **Cientista de IA / Pesquisador (RL)** | Evoluir a política de decisão sem quebrar consumidores existentes. | Interface `ISchedulingPolicy` (v1 heurística → v2 aprendida) e proveniência (`feature_vector`) como dado de treino. |
| **CISO / Compliance** | Auditar toda decisão de admissão/preempção. | `DecisionJournal` append-only + evento `scheduler.decision.recorded` consumido por `025-Audit`. |
| **Tenant de baixo volume** | Não ser eternamente preterido por tenants grandes. | Fairness/anti-*starvation* (`min_share`, aging) — NFR-006. |
| **Agente de IA (consumidor não-humano)** | Submeter e cancelar tarefas via contrato estável, incluindo A2A/MCP a jusante. | API gRPC/REST versionada (`aios.scheduler.v1`), idempotente, com envelope de erro padrão. |

---

## 5. Princípios de Design do Scheduler

Estes princípios especializam, para o domínio de scheduling, os princípios
invioláveis da `../000-Vision/Vision.md` (§6). Toda ADR do módulo (faixa
ADR-0090–0099) é avaliada contra eles.

1. **Decisão é barata; execução é cara.** O caminho quente de `Submit` NÃO DEVE
   depender de I/O bloqueante externo síncrono (cotas/capacidade em Redis,
   orçamento/PDP em cache de curta duração) — meta p99 ≤ 20 ms (NFR-001).
2. **Nunca *overcommit*.** Toda reserva de slot é atômica (script Lua/CAS em
   Redis); zero admissões além de `max_concurrency` sob concorrência (NFR-005).
3. **Custo e qualidade são restrições de primeira classe, não pós-processamento.**
   A prioridade efetiva nasce de `C = α·L + β·$ + γ·E` sob `Q ≥ Q_min`, jamais
   de um número mágico fixo por cliente.
4. **Preempção é compensável, nunca destrutiva.** Toda vítima recebe aviso e
   *grace period*; o estado é preservado para retomada (`PREEMPTED → QUEUED`).
5. **Backpressure é explícita e antecipada.** O sistema sinaliza saturação
   (`defer`/`reject` + `Retry-After`) antes de degradar silenciosamente ou
   colapsar em cascata.
6. **Fairness é uma garantia, não um acidente.** Todo tenant/classe ativo tem
   vazão mínima garantida por aging; nenhum tenant grande pode afogar outro.
7. **Toda decisão é explicável e auditável.** Score, pesos, política/versão e
   *feature vector* são persistidos como proveniência (`DecisionJournal`),
   nunca apenas o resultado final.
8. **A política de decisão é substituível sem quebra de contrato.** A
   interface `ISchedulingPolicy` isola heurística (v1) de política aprendida
   (v2, RL); API, eventos e schema de dados permanecem estáveis.
9. **Sharding determinístico antes de escala vertical.** Crescimento de carga é
   absorvido por mais shards/réplicas com estado externalizado, não por
   instâncias monolíticas maiores.
10. **Segurança por padrão.** Nenhuma admissão ou preempção ocorre sem PDP
    (`022-Policy`) — *default deny* herdado de RFC-0001 §5.8.

---

## 6. Não-Objetivos Explícitos

O Scheduler explicitamente **NÃO** busca, nesta versão nem em evoluções
previstas dentro do escopo deste módulo:

- **Não é um orquestrador de workflows.** Não decompõe objetivos em planos nem
  gerencia sagas — isso é `012-Planning` e `014-Workflow`. O Scheduler recebe
  tarefas já definidas e decide apenas quando/onde/com que prioridade executá-las.
- **Não é a fonte de verdade de custo.** Ele consome estimativas de
  `026-Cost-Optimizer`; nunca calcula preço de inferência de forma independente.
- **Não gerencia o ciclo de vida completo do agente.** Materializar
  *spawn*/*suspend*/*resume*/*kill* é do `007-Agent-Runtime`; o Scheduler apenas
  emite a decisão e observa sinais de confirmação.
- **Não é um mecanismo de rate-limiting de borda.** Rate-limit por IP/API-key é
  responsabilidade do Gateway (YARP) e de `021-Security`; o Scheduler aplica
  cotas de *concorrência* e *orçamento* internas, não limites de tráfego HTTP.
- **Não substitui o Cluster Manager.** Não provê HA/replicação de
  infraestrutura nem decide topologia física de nós — consome o inventário de
  `027-Cluster` como entrada.
- **Não é, na v1, uma política aprendida.** `ISchedulingPolicy` v1 é uma
  heurística determinística e auditável; a política aprendida (RL, v2) é um
  não-objetivo desta versão, preparada apenas como extensão futura sem
  quebra de interface (RFC-0091).

---

## 7. Relação com a Visão Global do AIOS

O Scheduler é uma materialização direta do modelo formal de otimização
apresentado em `../000-Vision/Vision.md` (§8):

```
    minimizar   C = α·L + β·$ + γ·E
    sujeito a   Q ≥ Q_min        (qualidade mínima)
                A ≥ A_min        (disponibilidade / SLA)
                $_acumulado ≤ B  (orçamento por tenant/agente)
```

Na tabela de subsistemas da Visão Global (§7.1), o Scheduler é o análogo
direto do **CFS** (*Completely Fair Scheduler*) do Linux, mas — diferente de um
scheduler de CPU — ele **conhece custo monetário e qualidade** como dimensões
de primeira classe da decisão, e não apenas tempo de CPU. Ele contribui
diretamente para os seguintes requisitos-chave da Visão Global:

| Requisito de Visão | Como o Scheduler contribui |
|---------------------|------------------------------|
| V-R1 — suportar 1 → 10⁶⁺ agentes concorrentes | Sharding determinístico + estado externalizado + réplicas *stateless* (`Scalability.md`). |
| V-R2 — disponibilidade do plano de controle ≥ 99,95% | NFR-003; sem SPOF; degradação graciosa (`FailureRecovery.md`). |
| V-R5 — roteamento consciente de custo/qualidade | Score `C = α·L + β·$ + γ·E`, integração com 026/017. |
| V-R9 — compatibilidade retroativa de APIs | `ISchedulingPolicy` isola evolução de política da API pública. |
| V-R10 — RTO ≤ 15 min, RPO ≤ 5 min | NFR-007/008, outbox transacional (`DecisionJournal`). |

O Scheduler também é o ponto onde o Risco Estratégico **RSK-03** (coordenação
multi-agente O(N²)) e **RSK-04** (gargalo/SPOF no plano de controle) da Visão
Global são mitigados na prática: filas particionadas por shard evitam
contenção cruzada, e a ausência de locks globais no caminho quente evita que o
Scheduler se torne, ele próprio, o gargalo do sistema.

---

## 8. Referências

- Design Brief (fonte da verdade interna): `./_DESIGN_BRIEF.md`
- Visão Global: `../000-Vision/Vision.md`
- Arquitetura do módulo: `./Architecture.md`
- Contratos centrais: `../003-RFC/RFC-0001-Architecture-Baseline.md`
- Glossário: `../040-Glossary/Glossary.md`
- Dependências: `../006-Kernel/`, `../022-Policy/`, `../026-Cost-Optimizer/`, `../027-Cluster/`

---

*Fim do documento de Visão do Módulo 009-Scheduler. Próximo na cadeia:*
`./Architecture.md`.
