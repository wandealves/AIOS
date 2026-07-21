---
Documento: FAQ
Módulo: 008-Agent-Lifecycle
Status: Draft
Versão: 0.1
Última atualização: 2026-07-20
Responsável (RACI-A): Arquiteto do Módulo 008 — Agent Lifecycle
ADRs relacionados: ADR-0080, ADR-0081, ADR-0082, ADR-0083, ADR-0084, ADR-0085, ADR-0086, ADR-0087, ADR-0088, ADR-0089
RFCs relacionados: RFC-0001, RFC-0080
Depende de: 006-Kernel, 007-Agent-Runtime, 009-Scheduler, 010-Memory, 020-Communication, 021-Security, 022-Policy, 024-Observability, 025-Audit, 027-Cluster
---

# 008-Agent-Lifecycle — Perguntas Frequentes (FAQ)

> Este documento esclarece dúvidas recorrentes de escopo, fronteira e uso do
> Agent Lifecycle. Ele **não redefine** contratos já normatizados em
> `../003-RFC/RFC-0001-Architecture-Baseline.md` nem termos já definidos em
> `../040-Glossary/Glossary.md` — apenas os aplica ao contexto deste módulo.
> Para o comportamento formal e autoritativo, consulte `./_DESIGN_BRIEF.md`.

---

## Índice

1. [Conceitos gerais](#1-conceitos-gerais)
2. [Fronteiras de responsabilidade](#2-fronteiras-de-responsabilidade)
3. [Máquina de estados e ACB](#3-máquina-de-estados-e-acb)
4. [Hibernação de cold agents](#4-hibernação-de-cold-agents)
5. [Checkpoint e restore](#5-checkpoint-e-restore)
6. [Migração entre shards e nós](#6-migração-entre-shards-e-nós)
7. [Concorrência, leases e idempotência](#7-concorrência-leases-e-idempotência)
8. [Eventos e integração assíncrona](#8-eventos-e-integração-assíncrona)
9. [Erros e diagnóstico](#9-erros-e-diagnóstico)
10. [Desempenho e escala](#10-desempenho-e-escala)
11. [Segurança, multi-tenant e LGPD](#11-segurança-multi-tenant-e-lgpd)
12. [Armadilhas comuns (pitfalls)](#12-armadilhas-comuns-pitfalls)

---

## 1. Conceitos gerais

### 1.1 O que é, em uma frase, o Agent Lifecycle?

É o **Lifecycle Manager** do AIOS — análogo a `init`/`systemd` de um sistema
operacional clássico — a autoridade única sobre a máquina de estados canônica
de todo agente gerenciado, responsável por spawn/materialização, suspensão,
hibernação de *cold agents*, checkpoint/restore e migração, garantindo que
nenhum trabalho aceito seja perdido em qualquer uma dessas transições. Ver
`./Vision.md` §1.

### 1.2 Por que existe um módulo separado do Kernel (006) só para isso?

Porque a fronteira entre "estado *lógico*" e "estado *físico*" de um agente é
uma separação de responsabilidade deliberada: o Kernel (006) é dono da
*criação* do ACB e das syscalls cognitivas; o Agent Lifecycle é dono do
*estado de ciclo de vida* dentro do ACB e de seu versionamento por
`generation`. Misturar as duas responsabilidades sobrecarregaria o Kernel com
sagas de infraestrutura (materialização de runtime, serialização de snapshot,
transferência entre nós) que não são sua especialidade. Ver `./Vision.md` §2.1
e `_DESIGN_BRIEF.md` §1.1 (R2).

### 1.3 O módulo executa o loop cognitivo do agente?

**Não.** O módulo nunca executa raciocínio, uso de ferramentas ou qualquer
lógica de domínio do agente — isso é `007-Agent-Runtime`. O módulo 008 apenas
**inicia**, **pausa**, **hiberna** e **encerra** o processo de runtime que
hospeda esse loop. Ver Não-Responsabilidade N1 no `_DESIGN_BRIEF.md` §1.2.

### 1.4 Quais são exatamente os oito estados da FSM?

`Created`, `Ready`, `Running`, `Suspended`, `Hibernated`, `Migrating`,
`Terminated` (terminal) e `Failed` (terminal). Diagrama completo, guardas e
invariantes em `./StateMachine.md` e `_DESIGN_BRIEF.md` §4.

---

## 2. Fronteiras de responsabilidade

### 2.1 O módulo decide *onde* e *quando* um agente executa?

**Não.** Quem decide prioridade, custo, admissão e placement é o
`009-Scheduler`. O módulo 008 **aplica** a decisão de placement (grava
`placement_shard`/`placement_node` no ACB e materializa o runtime no destino
indicado), mas nunca a **toma**. Ver Não-Responsabilidade N2 e a transição T2
(`Created → Ready`, gatilho `admit`) em `_DESIGN_BRIEF.md` §4.3.

### 2.2 O módulo persiste ou consolida camadas de memória?

**Não.** Consolidação de working/short/long/semantic/procedural/episodic
memory é responsabilidade exclusiva de `010-Memory`. O módulo 008 serializa
apenas o **ponteiro/snapshot da working memory** (`working_memory_ref`) como
parte do checkpoint — nunca materializa memória de longo prazo. Ver
Não-Responsabilidade N4.

### 2.3 O módulo decide se uma operação é permitida (RBAC/ABAC)?

**Não.** O `LifecyclePolicyEnforcer` é **PEP** (Policy Enforcement Point):
monta o contexto de decisão e consulta o **PDP** do `022-Policy` antes de toda
mutação, com postura *default deny*. O módulo nunca hospeda regra de política.
Ver Não-Responsabilidade N5 e `_DESIGN_BRIEF.md` §12.1.

### 2.4 O módulo provisiona nós/pods ou faz autoscaling de infraestrutura?

**Não.** Provisão física de nós, autoscaling e replicação de storage são
responsabilidade do `027-Cluster` e do `028-Deployment`. O módulo 008 consome
decisões de infraestrutura (por exemplo, `cluster.node.draining` para
disparar migração em massa), mas não as gera. Ver Não-Responsabilidade N6.

### 2.5 O módulo é a trilha de auditoria imutável do AIOS?

**Não.** O módulo 008 **emite** eventos de ciclo de vida
(`aios.<tenant>.agent.lifecycle.*`); a persistência **imutável e durável**
dessa trilha é responsabilidade do `025-Audit`. Essa separação evita que o
módulo acumule responsabilidade de armazenamento de longo prazo que não é seu
domínio. Ver Não-Responsabilidade N8.

### 2.6 Quem materializa fisicamente o processo do Agent Runtime — o módulo 008 ou o 007?

O módulo 008 **solicita** a materialização (via `SpawnManager`, negociando com
o Runtime Supervisor de `007-Agent-Runtime`), e é o **007** quem efetivamente
cria/monitora/encerra o processo isolado onde o loop cognitivo roda. O módulo
008 reage ao resultado dessa solicitação (sucesso → `Running`; timeout/falha →
compensação ou `Failed`).

---

## 3. Máquina de estados e ACB

### 3.1 Qual a diferença entre `Suspended` e `Hibernated`?

`Suspended` é um estado **quente**: o agente está pausado, mas a working
memory ainda reside em RAM/Redis (RAM reduzida, não zero) e a retomada
(`resume`, T5) é rápida. `Hibernated` é um estado **frio**: o ACB e a working
memory foram serializados para MinIO/PostgreSQL e a **RAM foi liberada
integralmente** — retomar exige materialização sob demanda (T7, `wake`), com
meta `p99 ≤ 250 ms` (NFR-001). Ver tabela de estados em
`_DESIGN_BRIEF.md` §4.1.

### 3.2 O que é `generation` e por que ele importa?

É um contador monotônico de encarnação, incrementado a cada spawn, restore ou
migração de um agente (`_DESIGN_BRIEF.md` §3). Ele existe para distinguir, de
forma inequívoca, *qual* execução física de um agente um checkpoint ou evento
se refere — essencial quando um agente é materializado, falha, e é
rematerializado várias vezes ao longo de sua vida lógica.

### 3.3 Um agente pode ir direto de `Created` para `Running`?

**Não.** A FSM exige passar por `Ready` (T2, `admit`) antes de `Running` (T3,
`start`) — o Scheduler precisa confirmar admissão (cotas/orçamento) antes de
qualquer materialização de runtime. Pular esse passo violaria a guarda **G3**
de T3, que exige slot alocado e lease adquirido. Ver `_DESIGN_BRIEF.md` §4.2–4.3.

### 3.4 Os estados terminais (`Terminated`, `Failed`) têm alguma transição de saída?

**Não, por invariante.** A **INV3** do `_DESIGN_BRIEF.md` §4.3 declara que
estados terminais são absorventes: nenhuma transição de saída existe. Um
agente `Terminated` ou `Failed` não pode ser "reanimado" — para retomar
trabalho equivalente, um **novo** `spawn` deve ser emitido, opcionalmente
referenciando o último `checkpoint_id` válido para restaurar contexto.

### 3.5 O que acontece com `runtime_instance_id` quando um agente atinge um estado terminal?

Ele DEVE ser `null` — a invariante (ii) do ACB (`_DESIGN_BRIEF.md` §3.1)
exige que `state` terminal implique `desired_state` terminal e
`runtime_instance_id = null`. Nenhum runtime físico permanece associado a um
agente terminado.

### 3.6 Quem é o dono do campo `state` do ACB — o módulo 008 ou o Kernel (006)?

O módulo 008. Embora o Kernel (006) seja dono da *criação* do ACB (R2), o
008 é o **único** componente autorizado a persistir transições de `state`
(R1) — qualquer escrita direta de `state` fora do `StateMachineEngine` do
módulo 008 é uma violação de contrato.

---

## 4. Hibernação de cold agents

### 4.1 Quando um agente é hibernado automaticamente?

Quando está `Suspended` e ocioso por `hibernation.idle_ttl_s` (padrão 300 s) —
**ou** quando há pressão de RAM acima de
`hibernation.ram_pressure_threshold_pct` (padrão 85%). Ambos os gatilhos
disparam a transição T6 (`Suspended → Hibernated`), sujeita à guarda **G6**:
checkpoint `consistent` gravado com sucesso antes de liberar RAM. Ver
`_DESIGN_BRIEF.md` §4.3, §8.

### 4.2 A hibernação é sempre automática, ou pode ser solicitada explicitamente?

Ambas as formas existem: o `HibernationController` dispara automaticamente
por ociosidade/pressão de RAM, e a API expõe `POST /v1/agents/{id}/hibernate`
para hibernação sob demanda (por exemplo, um orquestrador que sabe de
antemão que um agente ficará ocioso por um longo período). Ver `./API.md`
(quando disponível) e `_DESIGN_BRIEF.md` §5.1.

### 4.3 Um cold agent ocupa quanto de RAM?

Por design, **zero** RAM de working memory/runtime — o estado inteiro vive em
MinIO (blob) e PostgreSQL (metadados). O único custo de RAM residual é o
**índice quente mínimo** em Redis (`≤ 4 KiB` por ACB frio, NFR-011), mantido
apenas para roteamento e wake rápido, não para reconstrução de estado.

### 4.4 Qual a meta de latência para acordar (`wake`) um cold agent?

`p99 ≤ 250 ms`, o mesmo alvo do `spawn` de um agente novo (NFR-001), porque a
materialização de um cold agent é tratada como **deserialização + attach de
memória**, não como boot completo de processo — o `WarmPoolManager` elimina o
cold-start do processo de runtime em si.

### 4.5 Como o módulo evita que um "storm" de wakes simultâneos derrube um nó?

Via **backpressure**: a admissão de materializações é limitada por
`migration.max_concurrent_per_node` e por sinais do Scheduler (009); um pico
de `wake` é **enfileirado**, não descartado nem processado sem controle. Ver
`_DESIGN_BRIEF.md` §9 (modo de falha "Storm de hibernação").

### 4.6 Um agente hibernado pode ser terminado diretamente, sem acordar primeiro?

Sim — a transição T10 (`terminate`) é válida a partir de **qualquer** estado
não-terminal, incluindo `Hibernated`, conforme o diagrama de estados
(`_DESIGN_BRIEF.md` §4.2). Não é necessário materializar o agente apenas para
encerrá-lo.

---

## 5. Checkpoint e restore

### 5.1 Um checkpoint captura o quê, exatamente?

O **ACB** (identidade, estado, `generation`, ponteiros) e o **ponteiro da
working memory** (`working_memory_ref`) — nunca o conteúdo de memória de
longo prazo (isso é responsabilidade de `010-Memory`, referenciado por
ponteiro). O blob serializado (`codec: msgpack+zstd` por padrão) é gravado em
MinIO; metadados e índice, em PostgreSQL. Ver `_DESIGN_BRIEF.md` §3.3.

### 5.2 Qual a diferença entre um checkpoint `consistent` e um `crash`?

`consistent` significa que o checkpoint foi capturado com o agente
**quiesced** (parado de forma controlada, sem operação em curso) — é o único
tipo aceito como pré-requisito para hibernação ou migração (invariante INV4).
`crash` significa que o checkpoint foi capturado ou inferido sob falha (por
exemplo, o último checkpoint válido antes de um runtime morrer
inesperadamente) — usado apenas para recuperação, nunca para hibernação
planejada.

### 5.3 Com que frequência checkpoints são criados automaticamente?

A cada `checkpoint.interval_s` (padrão 120 s) para agentes `Running`, além de
um checkpoint obrigatório em toda transição para `Suspended` ou `Hibernated`.
Essa cadência é o que sustenta a meta **RPO ≤ 5 min** (NFR-006).

### 5.4 O que acontece se um `restore` detectar um checkpoint corrompido?

O módulo rejeita a operação com `AIOS-LIFECYCLE-0007` (422, não retriável) e
tenta o checkpoint anterior disponível para o mesmo agente; se nenhum
checkpoint íntegro existir, o agente transiciona para `Failed` (T11) com
auditoria completa do evento. Nunca há um restore "parcial" ou silenciosamente
degradado. Ver `_DESIGN_BRIEF.md` §5.3, §9.

### 5.5 Checkpoints são cifrados?

Sim, sempre — `AES-256-GCM` com chave por tenant (delegada a
`021-Security`), configuração fixa e não recarregável
(`checkpoint.encryption`). Isso é obrigatório porque a working memory PODE
conter dados pessoais, sujeitos a LGPD/GDPR. Ver `_DESIGN_BRIEF.md` §3.3, §12.3.

### 5.6 Um `restore` explícito muda o estado do agente imediatamente?

Sim — `POST /v1/agents/{id}/restore` (referenciando um `checkpointId`)
reconstrói o ACB e a working memory a partir do checkpoint indicado e
incrementa `generation`. O agente segue para `Ready`/`Running` conforme
disponibilidade de slot (Scheduler), analogamente a um `wake` de cold agent.

---

## 6. Migração entre shards e nós

### 6.1 O que dispara uma migração?

Duas fontes: uma decisão de placement do `009-Scheduler`/`027-Cluster` (por
exemplo, rebalanceamento de carga), ou o evento
`aios.<tenant>.cluster.node.draining`, que dispara migração em massa dos
agentes hospedados no nó a ser drenado. Ver `_DESIGN_BRIEF.md` §6.2.

### 6.2 Quais são as fases de uma migração (`MigrationJob`)?

`quiesce → checkpoint → transfer → materialize → cutover → cleanup`, conforme
`_DESIGN_BRIEF.md` §3.5. Cada fase é uma etapa da saga orquestrada pelo
`MigrationOrchestrator`, compensável em caso de falha.

### 6.3 O agente perde trabalho durante a migração?

**Não deveria** — essa é uma invariante de projeto (NFR-010 exige migração
sem perda de trabalho *aceito*). A origem só é invalidada **depois** que o
`cutover` confirma materialização bem-sucedida no destino (guarda **G8**,
transição T9). Se o destino falhar antes do cutover, a saga compensa e a
origem permanece ativa (RSK-L03 em `./Vision.md` §8).

### 6.4 O que acontece se o nó de destino cair no meio de uma migração?

O `MigrationJob` detecta timeout via `migration.saga_timeout_ms` (padrão
120000 ms) e aciona compensação: invalida o destino, mantém/reativa a origem,
e marca `MigrationJob.status = compensated`. O cliente recebe
`AIOS-LIFECYCLE-0010` (502, retriable) se consultou o estado durante a falha.
Ver `_DESIGN_BRIEF.md` §9.

### 6.5 Quantas migrações podem ocorrer simultaneamente em um mesmo nó?

Até `migration.max_concurrent_per_node` (padrão 8), para não saturar rede e
storage durante drenagem em massa de um nó. Excedido esse limite, novas
migrações são enfileiradas, não rejeitadas.

### 6.6 Qual a meta de latência de uma migração?

`p99 ≤ 2 s` para um agente de até 64 MiB dentro do mesmo datacenter
(NFR-010), sem perda de trabalho aceito.

---

## 7. Concorrência, leases e idempotência

### 7.1 Duas requisições podem transicionar o mesmo agente ao mesmo tempo?

Não com sucesso simultâneo — o `LeaseManager` garante que **no máximo um**
coordenador possua a lease de um `agent_id` por vez (invariante INV1). A
segunda requisição concorrente recebe `AIOS-LIFECYCLE-0003` (409, retriável).
Ver FR-010 e `_DESIGN_BRIEF.md` §4.3.

### 7.2 O módulo usa lock distribuído global?

**Não.** Apenas leases de granularidade **por agente** (Redis
`SET NX PX` + fencing token), nunca um lock global — isso é o que permite ao
módulo escalar horizontalmente por shard sem contenção cruzada entre agentes
distintos. Ver `_DESIGN_BRIEF.md` §10, princípio de design 2 em `./Vision.md`.

### 7.3 O que é o `fencing_token` e por que ele existe?

É um valor monotônico associado a cada lease concedida, persistido no ACB.
Ele existe para impedir que um coordenador "atrasado" (por exemplo, que
perdeu a lease por timeout mas ainda tenta escrever) corrompa o estado — uma
escrita com `fencing_token` obsoleto é sempre rejeitada
(`AIOS-LIFECYCLE-0013`), mesmo que a rede a entregue fora de ordem.

### 7.4 Toda mutação exige `Idempotency-Key`?

Sim — todas as mutações da API (`spawn`, `suspend`, `resume`, `hibernate`,
`wake`, `migrate`, `checkpoint`, `restore`, `terminate`) exigem
`Idempotency-Key`, conforme `_DESIGN_BRIEF.md` §5.1 e RFC-0001 §5.5.
Leituras (`GET`) não exigem, por não terem efeito colateral a deduplicar.

### 7.5 O que acontece se eu reenviar a mesma `Idempotency-Key`?

O resultado da primeira execução efetiva é retornado sem que a operação
ocorra novamente e sem publicar um segundo evento (NFR-013: repetição com
mesma chave ⇒ mesmo resultado, zero efeito duplicado). Isso vale mesmo que a
transição já tenha avançado o estado do agente adiante do que a chamada
original solicitava.

### 7.6 O que acontece se um coordenador do módulo 008 falhar no meio de uma transição?

A lease expira (TTL curto, `lease.ttl_ms`); um novo coordenador relê o log de
`LifecycleTransition` e o estado de saga em curso, e retoma ou compensa a
operação — o `fencing_token` garante que o coordenador antigo, se voltar a
escrever, seja rejeitado. Ver `_DESIGN_BRIEF.md` §9 ("Crash do coordenador").

---

## 8. Eventos e integração assíncrona

### 8.1 Que garantia de entrega o módulo oferece para seus eventos?

**At-least-once**, via padrão **Outbox transacional** + NATS/JetStream — a
mesma transação de banco que grava a `LifecycleTransition` grava a entrada de
outbox, garantindo que nenhum evento aceito seja perdido mesmo em crash entre
commit e publicação. Consumidores DEVEM deduplicar por `event.id` (RFC-0001
§5.2/§5.5). Ver `_DESIGN_BRIEF.md` §1.1 (R8), §6.

### 8.2 Quais eventos o módulo produz?

Treze eventos sob `aios.<tenant>.agent.lifecycle.<acao>`: `created`, `ready`,
`running`, `suspended`, `resumed`, `hibernated`, `woken`, `migrating`,
`migrated`, `checkpointed`, `restored`, `terminated`, `failed`. Catálogo
completo com schema mínimo em `./Events.md` e `_DESIGN_BRIEF.md` §6.1.

### 8.3 Quais eventos o módulo consome, e de quem?

`agent.control.created` (Kernel, 006), `scheduler.placement.decided` e
`scheduler.preemption.requested` (Scheduler, 009), `agent.runtime.heartbeat` e
`agent.runtime.exited` (Runtime Supervisor, 007), `task.execution.completed`
(Workflow/Task, 014) e `cluster.node.draining` (Cluster, 027). Ver
`_DESIGN_BRIEF.md` §6.2.

### 8.4 O payload de um evento de ciclo de vida contém o conteúdo da working memory?

**Não, nunca.** O `data` mínimo carrega apenas metadados de estado
(`agentId`, `tenantId`, `generation`, `fromState`, `toState`, `trigger`,
`reason?`, `checkpointId?`, `placement?`) — nunca conteúdo de memória. Isso é
uma exigência explícita de minimização de dados (`_DESIGN_BRIEF.md` §12.3).

### 8.5 Os eventos do módulo são ordenados?

Sim, por agente — a coluna `seq` em `LifecycleTransition` garante ordem
monotônica por `(agent_id, seq)`, e a publicação via outbox preserva essa
ordem dentro do mesmo agente. Não há garantia de ordem *entre* agentes
diferentes.

---

## 9. Erros e diagnóstico

### 9.1 Que faixa de código de erro pertence a este módulo?

`AIOS-LIFECYCLE-<NNNN>`, faixa reservada `0001–0099`. Catálogo completo em
`./API.md` e `_DESIGN_BRIEF.md` §5.3.

### 9.2 Recebi `AIOS-LIFECYCLE-0002`. O que significa?

Uma transição inválida para o estado atual foi solicitada — por exemplo,
tentar `resume` em um agente `Terminated` (estado terminal, sem transições de
saída). Consulte `./StateMachine.md` para as transições permitidas a partir
do estado atual do ACB.

### 9.3 Recebi `AIOS-LIFECYCLE-0003`. Devo tratar como erro definitivo?

**Não** — é `retriable: true`. Significa que outra transição está em curso
para o mesmo agente (lease não adquirida). Retente com backoff exponencial e
jitter; a operação concorrente provavelmente já terá liberado a lease.

### 9.4 Recebi `AIOS-LIFECYCLE-0013`. O que aconteceu?

O `generation`/`fencing_token` usado na requisição está obsoleto — outra
transição avançou o agente adiante do estado que a requisição assumia.
Releia o estado atual (`GET /v1/agents/{id}`) antes de retentar; não é
seguro simplesmente reenviar a mesma requisição.

### 9.5 Recebi `AIOS-LIFECYCLE-0006` (timeout de materialização). O agente ficou em estado inconsistente?

Não — a saga de spawn/wake é compensável: se a materialização exceder o
timeout configurado, o `SpawnManager` reverte a reserva de slot e a
transição resultante é determinística (tipicamente `Failed`, T11, guarda
G9), nunca um estado indefinido.

---

## 10. Desempenho e escala

### 10.1 Qual a meta de latência para `spawn`/materialização?

`p99 ≤ 250 ms`, `p50 ≤ 80 ms` (cold → `Running`, com WarmPool ativo),
NFR-001. Para a **decisão** de transição isolada (excluindo efeitos de I/O),
a meta é `p99 ≤ 20 ms` (NFR-002).

### 10.2 Quantos agentes o módulo suporta por cluster?

Meta de `≥ 10⁶` agentes por cluster, com `≤ 5%` ativos em RAM simultaneamente
(NFR-004) — a maioria em estado `Hibernated`, com índice quente mínimo em
Redis. Ver `./Vision.md` §1 e `_DESIGN_BRIEF.md` §10.

### 10.3 Qual o throughput de transições que o módulo sustenta?

`≥ 50.000 transições/s` por cluster (NFR-008), viabilizado pelo desacoplamento
entre o caminho quente de transição (CQRS, escrita append-only) e a
publicação de eventos (Outbox + JetStream assíncrono).

### 10.4 Como o sharding funciona neste módulo?

Determinístico: `shard = hash(tenant_id, agent_id) mod N`
(`../001-Architecture/Architecture.md` §12). Cada shard tem coordenadores
próprios; o estado (ACB/transições) é particionado por shard no PostgreSQL —
nenhum lock global cruza shards.

### 10.5 O que garante que um tenant não degrade o desempenho de outro?

Admissão controlada pelo Scheduler (009), backpressure por
`migration.max_concurrent_per_node`, e o fato de que leases são por agente
(não por tenant) — um tenant com muitos agentes concorrentes não bloqueia
transições de outro tenant.

---

## 11. Segurança, multi-tenant e LGPD

### 11.1 Toda mutação passa por autorização?

Sim — o `LifecyclePolicyEnforcer` é PEP para **toda** mutação, com postura
*default deny*: ausência de decisão explícita `allow` do PDP resulta em
`AIOS-LIFECYCLE-0012` (403). Ver FR-013 e `_DESIGN_BRIEF.md` §12.1.

### 11.2 Um tenant pode acessar/transicionar o agente de outro tenant?

Não — toda entidade carrega `tenant_id` com RLS no PostgreSQL, e qualquer
`tenant` divergente do contexto autenticado (`X-AIOS-Tenant`) é rejeitado
antes de qualquer outra lógica, conforme RFC-0001 §6.

### 11.3 Como funciona o "direito ao esquecimento" para um agente terminado?

O `TombstoneManager` executa expurgo rastreável do ACB, das transições, dos
checkpoints e dos snapshots ao término do prazo de retenção
(`retention.terminated_ttl_days`, `retention.checkpoint_ttl_days`) ou sob
solicitação explícita, emitindo evento auditável ao `025-Audit`. O ID do
agente **nunca** é reutilizado após expurgo. Ver `_DESIGN_BRIEF.md` §12.3.

### 11.4 Snapshots de checkpoint podem conter dados pessoais?

Sim, PODEM — a working memory serializada pode incluir PII. Por isso todo
checkpoint é cifrado em repouso (AES-256-GCM, chave por tenant) e os eventos
de ciclo de vida **nunca** carregam o conteúdo, apenas metadados. Ver
`_DESIGN_BRIEF.md` §12.3.

### 11.5 A comunicação entre o módulo e seus clientes internos é criptografada?

Sim, via **mTLS** entre serviços internos (delegado ao `021-Security`); o
módulo não implementa sua própria camada de transporte seguro.

---

## 12. Armadilhas comuns (pitfalls)

| Armadilha | Por que é um erro | Como evitar |
|-----------|--------------------|-------------|
| Assumir que `Suspended` e `Hibernated` têm a mesma latência de retomada. | `Hibernated` envolve materialização a frio (rehidratar de armazenamento durável); é substancialmente mais lento que `resume` de um agente `Suspended`, ainda que dentro da meta de 250 ms p99. | Não usar hibernação para agentes que precisam retomar em latência de milissegundos consistente — nesse caso, manter em `Suspended`. |
| Reaproveitar a mesma `Idempotency-Key` para operações logicamente diferentes (por exemplo, `hibernate` e depois `terminate` do mesmo agente). | Gera comportamento indefinido de deduplicação — a chave deve corresponder a uma única *intenção* de mutação. | Gerar uma chave nova (ULID/UUID) por intenção de mutação, nunca reaproveitar entre operações distintas. |
| Tratar `AIOS-LIFECYCLE-0003` (lease não adquirida) como erro definitivo. | É `retriable: true` — a operação concorrente provavelmente terminará em milissegundos. | Retentar com backoff exponencial e jitter, nunca abortar a operação do usuário por esse código. |
| Presumir que um `restore` explícito é necessário após queda de runtime. | O `ReconciliationController` já materializa automaticamente a partir do último checkpoint válido quando detecta runtime morto (heartbeat ausente); chamar `restore` manualmente é redundante na maioria dos casos. | Confiar na reconciliação automática; usar `restore` explícito apenas para cenários de recuperação assistida ou auditoria. |
| Ignorar `checkpointId` ao investigar uma falha. | O histórico de transições (`GET .../lifecycle`) referencia `checkpointId` em cada evento relevante — sem ele, é difícil correlacionar exatamente qual snapshot foi usado em cada materialização. | Sempre correlacionar `checkpointId` do evento com o catálogo de checkpoints (`GET .../checkpoints`) ao investigar incidentes. |
| Assumir que eventos de ciclo de vida chegam exatamente uma vez. | A garantia é *at-least-once*; duplicatas são esperadas sob retry/falha de rede. | Consumidores DEVEM deduplicar por `event.id`, nunca assumir entrega única (RFC-0001 §5.5). |
| Chamar diretamente o Kernel (006), o Scheduler (009) ou o Memory (010) para "acelerar" uma transição de ciclo de vida. | Quebra a autoridade única do módulo 008 sobre a FSM (R1) e pode deixar o ACB em estado inconsistente com o estado físico real do runtime. | Sempre transicionar o agente através da API do módulo 008 (`/v1/agents/{id}/...`), nunca por escrita direta em outro módulo. |
| Configurar `hibernation.idle_ttl_s` muito baixo "para economizar RAM mais rápido". | Aumenta a frequência de wake/hibernate, elevando latência percebida pelo consumidor e custo de I/O de checkpoint sem ganho proporcional de economia. | Ajustar o TTL considerando o padrão real de uso do agente; monitorar `aios_lifecycle_agents_total{state}` antes de reduzir agressivamente. |

---

## Referências

- Fonte de verdade do módulo: `./_DESIGN_BRIEF.md`
- Visão do módulo: `./Vision.md`
- Máquina de estados: `./StateMachine.md`
- Contratos centrais: `../003-RFC/RFC-0001-Architecture-Baseline.md`
- Glossário: `../040-Glossary/Glossary.md`
- Exemplos práticos: `./Examples.md`

*Fim de `FAQ.md`.*
