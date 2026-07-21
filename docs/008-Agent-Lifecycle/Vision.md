---
Documento: Vision
Módulo: 008-Agent-Lifecycle
Status: Draft
Versão: 0.1
Última atualização: 2026-07-20
Responsável (RACI-A): Arquiteto do Módulo 008 — Agent Lifecycle
ADRs relacionados: ADR-0080, ADR-0081, ADR-0082, ADR-0083, ADR-0084, ADR-0085, ADR-0086, ADR-0087, ADR-0088, ADR-0089
RFCs relacionados: RFC-0001, RFC-0080
Depende de: 006-Kernel, 007-Agent-Runtime, 009-Scheduler, 010-Memory, 020-Communication, 021-Security, 022-Policy, 024-Observability, 025-Audit, 027-Cluster
---

# 008-Agent-Lifecycle — Visão do Módulo

> "Onde o Kernel decide *que* um agente existe e o Scheduler decide *quem executa
> quando*, o Agent Lifecycle decide *em que estado físico* um agente está agora —
> e garante que ele possa transitar entre `Running` quente e `Hibernated` frio sem
> perder um único byte de trabalho aceito. É o `init`/`systemd` do AIOS." —
> analogia condutora deste módulo (ver `../040-Glossary/Glossary.md`, termos
> **Lifecycle (Ciclo de Vida)**, **ACB** e **Cold Agent**).

---

## 1. Propósito do Módulo

O **008-Agent-Lifecycle** é o **Lifecycle Manager** do AIOS — o subsistema
listado explicitamente na tabela de subsistemas da Visão Global
(`../000-Vision/Vision.md` §7.1) como análogo a `init`/`systemd` de um sistema
operacional clássico, responsável por `spawn/suspend/resume/migrate/kill`. Sua
razão de existir é ser a **autoridade única** sobre a **máquina de estados
canônica** de todo agente gerenciado (`Created → Ready → Running ↔ Suspended →
Hibernated → Migrating → Terminated/Failed`, ver `_DESIGN_BRIEF.md` §4) e o
**único** componente do AIOS autorizado a persistir transições desse estado no
**Agent Control Block (ACB)**.

Sem essa autoridade centralizada, o estado de ciclo de vida de um agente seria
uma verdade distribuída e inconsistente entre o Kernel (que cria o ACB), o
Scheduler (que decide admissão e placement), o Agent Runtime (que efetivamente
executa) e o Cluster (que decide nós saudáveis) — exatamente o tipo de
fragmentação de estado que a Visão Global rejeita ao exigir que "todo evento
relevante emita telemetria e trilha de auditoria imutável" (Princípio 5,
`../000-Vision/Vision.md` §6) e que "toda operação de mutação seja idempotente
ou compensável" (Princípio 6). O módulo 008 é o componente que torna essas
exigências verificáveis para o estado de ciclo de vida especificamente: ele é
**event-sourced** (toda transição é um fato imutável em `LifecycleTransition`),
**serializado** (uma única transição por agente por vez, via `LeaseManager`) e
**reconciliador** (corrige deriva entre estado desejado e observado de forma
contínua).

O módulo entrega três capacidades indissociáveis:

1. **A FSM canônica do agente** — oito estados, dez transições nomeadas com
   gatilho e guarda formal, dois estados terminais absorventes (`Terminated`,
   `Failed`), reproduzida integralmente em `./StateMachine.md` a partir de
   `_DESIGN_BRIEF.md` §4.
2. **Hibernação de *cold agents*** — a capacidade de serializar o ACB e a
   working memory de um agente ocioso para MinIO/PostgreSQL, liberar 100% da
   RAM ocupada, e materializá-lo de volta sob demanda com `p99 ≤ 250 ms`
   (NFR-001) — o mecanismo estrutural que permite ao AIOS sustentar `≥ 10⁶`
   agentes por cluster com `≤ 5%` ativos em RAM simultaneamente (NFR-004).
3. **Checkpoint/restore e migração como sagas compensáveis** — captura
   consistente de estado (ACB + ponteiro de working memory) com integridade
   verificável (`content_hash` sha-256) e cifragem (AES-256-GCM), servindo tanto
   à recuperação após falha (RTO ≤ 15 min, RPO ≤ 5 min) quanto à migração
   planejada de agentes entre shards/nós sem perda de trabalho aceito.

---

## 2. Problema que o Módulo Resolve

A Visão Global identifica oito limitações estruturais dos agentes de IA atuais
(`../000-Vision/Vision.md` §2, P1–P8). O módulo 008 é a resposta direta a duas
delas, e o pré-requisito estrutural para a meta de escala mais ambiciosa da
Visão (V-R1):

| Limitação/Meta da Visão | Como o Agent Lifecycle endereça |
|---------------------------|----------------------------------|
| **P4 — Ausência de gerenciamento de recursos** | Um agente que permanece `Running` indefinidamente, mesmo ocioso, consome RAM e slots de execução que poderiam servir outros agentes. O `HibernationController` (§2 do `_DESIGN_BRIEF.md`) converte ociosidade em RAM liberada de forma automática, sem intervenção do desenvolvedor do agente. |
| **P7 — Baixa observabilidade** | Por manter o histórico de transições como log event-sourced append-only (`LifecycleTransition`), o módulo torna reconstruível, para qualquer agente, exatamente quando e por que ele mudou de estado — inclusive o `actor` e o resultado da guarda avaliada. |
| **V-R1 — Suportar de 1 a 10⁶+ agentes concorrentes distribuídos** (`../000-Vision/Vision.md` §9) | Esta é a meta que justifica a existência do módulo como subsistema dedicado (e não como uma tabela de estado dentro do Kernel): sem hibernação de cold agents com índice quente mínimo (`≤ 4 KiB` por ACB frio em Redis, NFR-011) e materialização sob demanda em `< 250 ms`, a meta de 10⁶ agentes exigiria RAM proporcional ao número total de agentes, não ao número de agentes *ativos* — economicamente inviável. |
| **V-R10 — RTO ≤ 15 min, RPO ≤ 5 min do plano de controle** (`../000-Vision/Vision.md` §9) | O par checkpoint/restore do módulo 008 é a implementação concreta dessa meta para o estado de agente: qualquer agente `Running` tem, no máximo, 5 minutos de working memory não recuperável em caso de falha de nó (`checkpoint.interval_s = 120`, mais checkpoint em `suspend`), e a materialização a partir do último checkpoint válido restabelece execução dentro da janela de 15 minutos. |

### 2.1 Por que este problema não é resolvido dentro do Kernel (006) ou do Scheduler (009)

Uma alternativa de design seria deixar que o próprio Kernel persistisse
transições de estado do ACB (já que ele o cria) ou que o Scheduler mantivesse
seu próprio modelo de "agente ativo/inativo" para fins de admissão. Essa
alternativa foi descartada porque:

- **Misturaria "decisão" com "estado"**: o Scheduler decide *quem* executa e
  *onde* (placement); se ele também fosse dono da FSM, a fronteira entre
  "decidir admitir" e "aplicar a admissão" desapareceria, violando a separação
  de responsabilidades que a Visão Global exige entre módulos especializados
  (Princípio 7 — Extensibilidade sem fork).
- **Sobrecarregaria o Kernel com efeitos físicos**: o Kernel (006) é
  deliberadamente um *broker* de syscalls cognitivas e cotas, não um executor
  de sagas de infraestrutura (materialização de runtime, serialização de
  snapshot, transferência entre nós). Colocar essas sagas no Kernel violaria o
  princípio "Kernel mínimo, brokerado, nunca executor" documentado em
  `../006-Kernel/Vision.md` §5.
- **Impediria hibernação em escala**: sem um componente dedicado a decidir
  *quando* hibernar (pressão de RAM, ociosidade) e a orquestrar o
  checkpoint/liberação de forma transacional, a meta de `≤ 5%` de agentes
  ativos em RAM (NFR-004) não seria alcançável de forma confiável — exigiria
  lógica ad-hoc espalhada por múltiplos módulos.

O módulo 008 centraliza essas três preocupações (FSM, hibernação/checkpoint,
migração) e **delega** a decisão de *quem/quando/onde* ao Scheduler (009), a
execução do loop cognitivo ao Agent Runtime (007), e a materialização/durabilidade
de memória de longo prazo ao Memory (010) — ele é **coordenador de estado
físico**, nunca o dono da lógica de negócio desses domínios (ver
Não-Responsabilidades N1–N8 no `_DESIGN_BRIEF.md` §1.2).

---

## 3. Escopo

### 3.1 Escopo — Dentro (o módulo 008 É responsável por)

Reproduzido e resumido a partir das Responsabilidades R1–R12 do
`_DESIGN_BRIEF.md` §1.1:

| Área | Descrição |
|------|-----------|
| Máquina de estados canônica | Autoridade única sobre `LifecycleState` e suas transições; único componente que persiste mudança de estado no ACB (R1). |
| Agent Control Block (estado de ciclo de vida) | Mantém o estado de ciclo de vida, `generation` e seu versionamento dentro do ACB — o Kernel (006) é dono da *criação* e das syscalls cognitivas (R2). |
| Spawn / materialização | Materializa agentes novos ou frios com `p99 ≤ 250 ms`, negociando slot com o Scheduler e runtime com o Runtime Supervisor (R3). |
| Suspensão / retomada | `suspend`/`resume` preservando working memory (R4). |
| Hibernação de cold agents | Serializa ACB + working memory para MinIO/PostgreSQL, libera RAM, habilita escala a 10⁶ (R5). |
| Checkpoint / restore | Captura e restauração consistentes de ACB + working memory, base de RTO ≤ 15 min (R6). |
| Migração | Orquestra migração entre shards/nós consumindo decisões de placement do Scheduler/Cluster (R7). |
| Emissão de eventos | Publica `aios.<tenant>.agent.lifecycle.*` de forma transacional e idempotente (R8). |
| Reconciliação | Corrige deriva entre estado desejado e observado continuamente (R9). |
| Leases/locks distribuídos | Garante transição serial por agente (R10). |
| Superfície de API | Expõe REST/gRPC de ciclo de vida e histórico event-sourced (R11). |
| Retenção e expurgo | Aplica tombstone e expurgo LGPD/GDPR de agentes terminados (R12). |

### 3.2 Escopo — Fora (o módulo 008 NÃO é responsável por)

Reproduzido do `_DESIGN_BRIEF.md` §1.2 (não redefinido aqui, apenas resumido
para orientar o leitor deste documento de Visão):

| Área excluída | Dono real | Por quê |
|----------------|-----------|---------|
| Loop cognitivo (ReAct, uso de tools) | `007-Agent-Runtime` | O 008 apenas inicia/pausa/encerra o processo de runtime — não pensa por ele (N1). |
| Quem executa, quando e onde (prioridade, custo, admissão, placement) | `009-Scheduler` | O 008 *aplica* a decisão de placement, não a *toma* (N2). |
| Syscalls cognitivas e cotas | `006-Kernel` + `026-Cost-Optimizer` | O 008 lê cotas por referência para admitir uma materialização, mas não as gerencia (N3). |
| Persistência/consolidação de camadas de memória | `010-Memory` | O 008 serializa apenas o *ponteiro/snapshot da working memory* como parte do checkpoint (N4). |
| Decisões de política (RBAC/ABAC) | `022-Policy` | O 008 atua como PEP consultando o PDP, nunca decide política (N5). |
| Provisão física de nós/pods, autoscaling, replicação de storage | `027-Cluster` + `028-Deployment` | O 008 consome decisões de infraestrutura, não as gera (N6). |
| Roteamento de modelos LLM e execução de tools | `017-Model-Router` + `015-Tool-Manager` | Fora de escopo do ciclo de vida (N7). |
| Trilha de auditoria imutável | `025-Audit` | O 008 *emite* eventos; a durabilidade imutável é do Audit (N8). |

Esta separação segue diretamente o **Princípio 1** da Visão Global (Separação
Plano de Controle / Plano de Dados) e a fronteira 008×006×009×007 formalizada
em `ADR-0088` (a propor).

---

## 4. Personas Atendidas

O módulo 008 é majoritariamente um módulo de **infraestrutura interna de
plataforma**: seus consumidores diretos são outros módulos (Kernel, Scheduler,
Runtime Supervisor) e ferramentas de operação. A tabela abaixo detalha a
relação de cada persona da Visão Global (`../000-Vision/Vision.md` §5) com o
Agent Lifecycle:

| Persona | Como interage com o módulo | Necessidade satisfeita |
|---------|------------------------------|--------------------------|
| **Desenvolvedor de Agentes** | Indiretamente, via SDK (`031-SDK`) que traduz `suspend`/`resume`/`checkpoint`/`restore` de alto nível em chamadas REST/gRPC de ciclo de vida. | Um agente que fica ocioso por horas não deve consumir orçamento de RAM nem custo — a hibernação é transparente ao código do agente. |
| **Operador de Plataforma (SRE)** | Consome métricas `aios_lifecycle_*`, dashboards de agentes por estado, opera migração em massa via evento `cluster.node.draining`. | Visibilidade de saturação (agentes ativos vs. frios), controle de migração planejada durante manutenção de nós. |
| **Arquiteto Enterprise** | Define/valida a fronteira 008×006×009×007 e os SLOs de RTO/RPO exigidos por contrato. | Garantia estrutural de que hibernação/migração não introduzem perda de trabalho aceito. |
| **CISO / Compliance** | Audita eventos `agent.lifecycle.terminated`/`failed`, verifica retenção e expurgo LGPD de checkpoints (`TombstoneManager`). | Prova de que agentes terminados são expurgados dentro do prazo de retenção configurado e que IDs não são reutilizados. |
| **Gestor de Custo (FinOps)** | Observa a proporção de agentes `Hibernated` vs. `Running` como proxy de eficiência de RAM/custo de infraestrutura. | Confirmação de que o *WarmPool* e a hibernação estão convertendo ociosidade em economia real, não apenas latência de wake maior. |
| **Agente de IA (consumidor indireto)** | Nunca chama o módulo 008 diretamente — é chamado *sobre* ele (o Kernel dispara `spawn`/`suspend`/`checkpoint` em seu nome). | Continuidade de execução (retomar exatamente de onde parou) sem precisar saber que foi hibernado, suspenso ou migrado. |
| **Cluster/HA Operator (`027`)** | Dispara migração em massa de agentes ao drenar um nó (`cluster.node.draining`), consome `MigrationJob.status`. | Capacidade de esvaziar um nó com segurança antes de manutenção/desligamento, sem interromper trabalho aceito. |

---

## 5. Princípios de Design do Módulo

Estes princípios especializam, para o módulo 008, os dez princípios invioláveis
da Visão Global (`../000-Vision/Vision.md` §6). Toda ADR do módulo
(`ADR-0080`..`ADR-0089`) DEVE ser avaliada contra eles.

1. **A FSM é um contrato imutável, não uma sugestão.** Estados, transições e
   guardas definidos em `_DESIGN_BRIEF.md` §4 são canônicos; nenhuma
   implementação PODE introduzir estado ou transição não documentados sem
   nova ADR (`ADR-0080`). Especializa o Princípio 9 (Compatibilidade futura).
2. **Uma transição por agente por vez, sem lock global.** O
   `LeaseManager` garante serialização por `agent_id` via Redis
   (`SET NX PX` + fencing token), nunca por lock distribuído de escopo amplo —
   especializa o Princípio 3 (recursos cotados e isolados) e viabiliza escala
   horizontal por shard (`ADR-0084`).
3. **Cold por padrão, quente sob demanda.** Um agente ocioso DEVE poder migrar
   para `Hibernated` (zero RAM) e retornar a `Running` com latência previsível
   (`p99 ≤ 250 ms`) — este é o princípio de design mais determinante do
   módulo, pré-requisito estrutural da meta V-R1 da Visão Global
   (`ADR-0081`, `ADR-0083`).
4. **Nenhuma hibernação, migração ou terminação sem checkpoint consistente
   prévio.** As invariantes INV4 do `_DESIGN_BRIEF.md` §4.3 (`Hibernated`/
   `Migrating` exigem checkpoint `consistent` válido) NÃO PODEM ser
   relaxadas — especializa o Princípio 6 (idempotência e recuperação).
5. **Event sourcing como fonte de proveniência.** Toda transição é um fato
   append-only em `LifecycleTransition`, nunca sobrescrito — o estado atual
   (projeção em `AcbStore`) é sempre derivável do log, nunca o inverso
   (`ADR-0086`).
6. **Migração é saga compensável, nunca transação distribuída síncrona.**
   `checkpoint → transfer → materialize → cutover` DEVE poder reverter em
   qualquer fase sem perda de trabalho aceito na origem (`ADR-0085`).
7. **Reconciliação contínua, não reativa apenas a comandos.** O
   `ReconciliationController` compara estado desejado × observado em loop
   independente de qualquer chamada de API — deriva (ex.: runtime morto sem
   `Terminated`) DEVE ser corrigida sem intervenção humana dentro de
   `reconcile.interval_ms` (`ADR-0087`).
8. **Observabilidade sem exceção.** 100% das transições DEVEM produzir trace
   OTel, evento e registro correlacionado (NFR-012) — não há "modo silencioso"
   para nenhuma transição, incluindo falhas.
9. **Multi-tenant e isolamento por construção.** Toda entidade do módulo
   carrega `tenant_id` com RLS; sharding determinístico
   (`hash(tenant_id,agent_id) mod N`) nunca cruza fronteira de tenant.
10. **Fronteiras de responsabilidade são invioláveis.** O módulo 008 NÃO DEVE
    absorver responsabilidades de Scheduler, Kernel, Memory ou Cluster mesmo
    quando isso pareceria simplificar um fluxo específico — a estabilidade de
    longo prazo do contrato entre módulos depende dessas fronteiras
    (`ADR-0088`).

---

## 6. Relação com a Visão Global do AIOS

O módulo 008 é nomeado explicitamente na tabela de subsistemas da Visão Global
como **Lifecycle Manager**, análogo a `init`/`systemd`
(`../000-Vision/Vision.md` §7.1). Três relações merecem destaque:

- **V-R1 (escala a 10⁶+ agentes)**: a combinação hibernação + WarmPool +
  sharding determinístico é a contribuição direta e mais visível do módulo a
  essa meta — a maioria dos agentes em repouso DEVE existir como ACB
  `Hibernated` sem custo de RAM, com índice quente mínimo (`≤ 4 KiB`, NFR-011)
  em Redis apenas para roteamento e wake rápido (ver `_DESIGN_BRIEF.md` §10).
- **V-R6 (memória hierárquica com consolidação e esquecimento)**: o módulo 008
  não implementa memória, mas é o ponto onde a working memory é *congelada* e
  *descongelada* de forma consistente a cada hibernação/wake — uma dependência
  direta e crítica de `010-Memory` para que a "amnésia" (P1 da Visão Global)
  não se manifeste como efeito colateral de hibernação.
- **V-R10 (RTO ≤ 15 min, RPO ≤ 5 min)**: o par checkpoint/restore do módulo é
  a instância concreta dessa meta para o estado de agente — o
  `ReconciliationController` detecta runtime morto e materializa a partir do
  último checkpoint válido dentro da janela de recuperação (ver
  `_DESIGN_BRIEF.md` §9).

O módulo também opera como o **ponto de tradução** entre a linguagem de
`fork`/`exec`/`wait`/`kill` de um kernel Unix e a linguagem de ciclo de vida
cognitivo do AIOS: onde `systemd` gerencia unidades com `start`/`stop`/
`restart`/`enable`, o módulo 008 gerencia agentes com `spawn`/`suspend`/
`hibernate`/`wake`/`migrate`/`terminate` — preservando, ao contrário de um
processo Unix comum, a capacidade de *congelar indefinidamente* um agente sem
custo de RAM e retomá-lo depois com estado idêntico. Essa capacidade de
hibernação profunda é o que diferencia o AIOS de um simples orquestrador de
processos e é o fio condutor de todo o desenho interno do módulo.

---

## 7. Não-Objetivos Explícitos (deste módulo)

Além das Não-Responsabilidades já listadas na §3.2 (que definem *fronteira de
propriedade*), o módulo declara explicitamente os seguintes não-objetivos de
**produto/design**, para evitar expansão de escopo (*scope creep*) ao longo do
tempo:

- O módulo 008 NÃO tem por objetivo se tornar um **orquestrador de workflows**
  genérico; sagas determinísticas de múltiplos passos de negócio são
  responsabilidade do `014-Workflow`. As sagas internas do módulo 008
  (`spawn`, migração) existem apenas para coordenar o *estado físico* de um
  agente, não fluxos de negócio arbitrários.
- O módulo 008 NÃO tem por objetivo prover **UI** de qualquer natureza —
  consumo humano do histórico de ciclo de vida ocorre via `032-WebConsole`,
  nunca diretamente do módulo.
- O módulo 008 NÃO tem por objetivo decidir **política de retenção legal**
  (prazos, bases legais de LGPD/GDPR) — ele **aplica** a retenção configurada
  (`retention.terminated_ttl_days`, `retention.checkpoint_ttl_days`), mas a
  definição desses prazos é responsabilidade de governança/compliance
  (`021-Security`, `025-Audit`).
- O módulo 008 NÃO tem por objetivo, na v1.0, suportar migração de agente
  **entre tenants** — cada `MigrationJob` opera estritamente dentro da
  fronteira de um único `tenant_id` (RFC-0001 §6), assim como toda transição
  da FSM.
- O módulo 008 NÃO tem por objetivo se tornar uma **fila de tarefas** genérica
  — ele gerencia o ciclo de vida do *agente* (a entidade que executa tarefas),
  não o ciclo de vida de tarefas individuais (isso é `014-Workflow`).

---

## 8. Riscos Estratégicos Específicos do Módulo

Complementando os riscos globais da Visão (`../000-Vision/Vision.md` §10),
o módulo 008 carrega riscos próprios por ser o guardião do estado físico de
todo agente do AIOS:

| Risco | Descrição | Mitigação | Referência |
|-------|-----------|-----------|------------|
| RSK-L01 | *Storm* de materialização (muitos `wake`/`spawn` simultâneos) esgotando RAM ou capacidade do Scheduler. | Backpressure via `migration.max_concurrent_per_node`, WarmPool com reserva limitada, admissão controlada pelo Scheduler (009); storm é enfileirado, não derruba o nó. | `_DESIGN_BRIEF.md` §9, §10 |
| RSK-L02 | Checkpoint corrompido tornando um agente irrecuperável. | `content_hash` (sha-256) verificado no restore; fallback ao checkpoint anterior; se nenhum válido existir, transição controlada para `Failed` com auditoria — nunca um estado indefinido. | `_DESIGN_BRIEF.md` §9, NFR-009 |
| RSK-L03 | Migração travada por queda do nó destino, deixando o agente em limbo. | Saga com `saga_timeout_ms`; compensação reativa a origem sem perda de trabalho aceito (`MigrationJob = compensated`). | `_DESIGN_BRIEF.md` §9, ADR-0085 |
| RSK-L04 | *Split-brain* de coordenadores (dois processos do módulo 008 acreditando ter a lease do mesmo agente). | `fencing_token` monotônico no ACB; escrita com token obsoleto é rejeitada (`AIOS-LIFECYCLE-0013`). | `_DESIGN_BRIEF.md` §9, ADR-0084 |
| RSK-L05 | Crescimento descontrolado de agentes nunca terminados (órfãos), inflando índice quente e custo de storage frio indefinidamente. | Hibernação automática por ociosidade + retenção/expurgo (`TombstoneManager`) com IDs nunca reutilizados. | `_DESIGN_BRIEF.md` §1.1 (R12), §12.3 |
| RSK-L06 | Vazamento de dados pessoais em snapshots/checkpoints de working memory. | Cifragem em repouso por tenant (AES-256-GCM), minimização de payload de evento (sem conteúdo de memória), expurgo rastreável sob solicitação. | `_DESIGN_BRIEF.md` §12.3 |

---

## 9. Referências

- Visão Global do AIOS: `../000-Vision/Vision.md`
- Arquitetura Global: `../001-Architecture/Architecture.md`
- Glossário: `../040-Glossary/Glossary.md`
- Contratos centrais: `../003-RFC/RFC-0001-Architecture-Baseline.md`
- Design Brief interno (fonte de verdade do módulo): `./_DESIGN_BRIEF.md`
- Índice do módulo: `./README.md`
- Módulos acoplados: `../006-Kernel/`, `../007-Agent-Runtime/`,
  `../009-Scheduler/`, `../010-Memory/`, `../020-Communication/`,
  `../021-Security/`, `../022-Policy/`, `../024-Observability/`,
  `../025-Audit/`, `../027-Cluster/`.

---

*Fim de `Vision.md`. Próximo documento na cadeia do módulo:*
`./Architecture.md`.
