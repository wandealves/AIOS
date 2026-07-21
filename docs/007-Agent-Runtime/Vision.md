---
Documento: Vision
Módulo: 007-Agent-Runtime
Status: Draft
Versão: 0.1
Última atualização: 2026-07-20
Responsável (RACI-A): Arquiteto do Módulo 007 — Agent Runtime
ADRs relacionados: ADR-0070, ADR-0071, ADR-0072, ADR-0073, ADR-0074, ADR-0075, ADR-0076, ADR-0077, ADR-0078, ADR-0079
RFCs relacionados: RFC-0001, RFC-0070, RFC-0071
Depende de: 006-Kernel, 008-Agent-Lifecycle, 009-Scheduler, 010-Memory, 011-Context, 012-Planning, 015-Tool-Manager, 017-Model-Router, 020-Communication, 021-Security, 022-Policy, 024-Observability, 025-Audit
---

# 007-Agent-Runtime — Visão do Módulo

> "O Agent Runtime está para o agente assim como o processo está para o código
> do usuário em um sistema operacional clássico: o espaço isolado, cotado e
> observável dentro do qual o **loop cognitivo** de um agente executa — sem
> jamais tocar diretamente memória, ferramentas, modelos ou o disco fora dos
> contratos que o governam." — analogia condutora deste módulo (ver
> `../040-Glossary/Glossary.md`, termos **Agent Runtime**, **Data Plane**,
> **Sandbox**, **ReAct**, **Cold Agent**).

---

## 1. Propósito do Módulo

O **007-Agent-Runtime** é o **plano de dados (data plane)** do AIOS — o único
lugar do sistema onde o pensamento de um agente efetivamente acontece. É um
processo **Python**, um por agente materializado, que executa o **loop
cognitivo ReAct** (`perceber → pensar → agir → observar → repetir`), hospeda o
**MCP host** que expõe ferramentas ao raciocínio do agente, e nunca invoca uma
ferramenta ou um modelo de inferência por conta própria — toda ferramenta passa
por `015-Tool-Manager`, toda inferência passa por `017-Model-Router` (ver
Visão Global, `../000-Vision/Vision.md` §7, "PLANO DE DADOS — AGENT RUNTIME
(Python · 007)").

Um processo separado do plano de controle, o **Runtime Supervisor** (.NET 10,
coordenado por `008-Agent-Lifecycle` e `009-Scheduler`), cria, monitora e mata
o **pool** de processos runtime, aplica *placement* e cotas de nível de pool.
Este módulo especifica **apenas o processo runtime Python** e o **contrato**
(`RuntimeControl`/`AgentExecution`, `aios.runtime.v1`) que ele expõe ao
Supervisor — a implementação do Supervisor em si é responsabilidade
compartilhada com `008-Agent-Lifecycle`/`009-Scheduler` (ver
`_DESIGN_BRIEF.md` §1.1).

O módulo entrega quatro capacidades indissociáveis:

1. **Materialização e ciclo de execução do agente** — a partir de um
   **AgentSpec** entregue pelo Supervisor, o `RuntimeBootstrapper` prepara
   sandbox, valida *capabilities* e conduz o agente até `done`/`failed`/
   `suspended` (`RuntimeInstance`/`AgentSession`, ver `_DESIGN_BRIEF.md` §3).
2. **O loop cognitivo ReAct** — `CognitiveLoopEngine` sequencia
   `Perception → Reasoning → Action → Observation`, delegando replanejamento a
   `012-Planning` e produzindo, a cada passo, um `ReActStep` imutável e
   auditável.
3. **Isolamento de sandbox por agente** — namespaces de processo,
   `seccomp-bpf`, `cgroups v2`, filesystem read-only + escrita efêmera, e
   *default deny* de rede — um agente comprometido, malicioso ou com defeito
   de lógica **não pode** escapar de seu compartimento.
4. **Enforcement local de capability e cota** — o runtime é um **PEP**
   (Policy Enforcement Point) para toda ação privilegiada (ferramenta,
   egress, syscall cognitiva), e um **governador de cota** local (tokens,
   custo, wall-clock, passos do loop) que suspende ou aborta ao esgotar o
   orçamento.

---

## 2. Problema que o Módulo Resolve

A Visão Global identifica oito limitações estruturais dos agentes de IA atuais
(`../000-Vision/Vision.md` §2, P1–P8). O Agent Runtime é a resposta direta a
quatro delas — porque é o único componente do AIOS onde o "pensamento" do
agente de fato ocorre, e portanto o único lugar onde essas limitações podem
ser corrigidas *em tempo de execução*, não apenas em tempo de design:

| Limitação da Visão | Como o Agent Runtime endereça |
|---------------------|--------------------------------|
| **P2 — Planejamento frágil para tarefas de longa duração** | O `CognitiveLoopEngine` não tenta "adivinhar" um plano melhor sozinho — ele **detecta** falha recuperável ou deriva de objetivo e delega replanejamento a `012-Planning` (transição `thinking→reflecting→thinking`), com limites explícitos de passos/profundidade/fan-out que impedem loops improdutivos infinitos. |
| **P4 — Ausência de gerenciamento de recursos** | O runtime aplica **isolamento forte** (`SandboxManager`: seccomp/cgroups/netns/FS) e **cotas locais** (`QuotaGovernor`: tokens/USD/wall-clock/passos) por agente — um agente com defeito de lógica ou sob prompt injection não pode consumir recursos além do orçamento nem degradar outros agentes no mesmo nó ("noisy neighbor" contido por construção). |
| **P7 — Baixa observabilidade** | Por ser o ponto onde toda decisão cognitiva ocorre, o runtime é também o ponto de instrumentação obrigatório: cada `ReActStep` gera trace OTel, métrica e evento NATS correlacionados por `traceparent`/`tenant`/`agent`, tornando qualquer raciocínio de agente reconstruível post-mortem. |
| **P8 — Governança fraca** | O runtime é um **PEP local**: nenhuma ferramenta é invocada, nenhum egress de rede ocorre e nenhuma syscall cognitiva privilegiada executa sem uma *capability* validada — concedida no boot ou confirmada dinamicamente pelo PDP via Kernel/`022-Policy`. *Default deny* estrutural, não convenção. |

Além disso, o Agent Runtime resolve um problema de **engenharia de
plataforma** que antecede os problemas de IA propriamente ditos: como executar
milhões de "processos cognitivos" de forma segura, barata e recuperável, sem
que a maioria deles consuma RAM continuamente. A resposta é o modelo de
**agente efêmero e *cold-storable***: um agente ocioso é checkpointado
(`CheckpointManager`) para MinIO/PostgreSQL via control plane, libera toda a
RAM do processo, e é rematerializado sob demanda com continuidade de working
memory e cursor de loop — o pré-requisito estrutural direto da meta V-R1 da
Visão Global (10⁶+ agentes).

### 2.1 Por que um processo Python isolado, e não uma thread dentro do Kernel/.NET

Uma alternativa de design seria executar o loop cognitivo como uma rotina
gerenciada dentro do próprio `006-Kernel` (.NET), evitando a complexidade de um
segundo runtime e de comunicação entre processos. Essa alternativa foi
descartada (ADR-0070) porque:

- **Isolamento de falhas exige limites de processo, não de thread.** Um bug de
  raciocínio, uma ferramenta que trava, ou uma tentativa de prompt injection
  que force execução de código arbitrário **precisam** de uma fronteira que o
  sistema operacional aplique (namespaces, seccomp, cgroups) — isso não existe
  dentro de um mesmo processo .NET compartilhado por todos os agentes.
- **O ecossistema de IA (LLM SDKs, MCP, bibliotecas de raciocínio) é
  majoritariamente Python.** Reimplementar esse ecossistema em .NET geraria
  atrito permanente de manutenção sem ganho correspondente — a Visão Global já
  reserva Python para o plano de dados justamente por isso
  (`../000-Vision/Vision.md` §13, ADR-0003).
- **Falha no plano de dados não pode derrubar o plano de controle.** Um
  processo runtime que trava, vaza memória ou é morto por violação de sandbox
  DEVE afetar **apenas aquele agente** — nunca o Kernel, o Scheduler ou
  qualquer outra sessão. Um modelo de thread compartilhada viola essa
  propriedade estrutural (Princípio 1 da Visão Global).
- **Cold start e hibernação exigem controle fino de processo.** Pool quente
  pré-forkado, *copy-on-write* de intérprete e liberação total de RAM em
  hibernação (ADR-0073/ADR-0074) só são viáveis no nível de processo do
  sistema operacional, não no nível de objeto gerenciado de uma VM .NET.

O Agent Runtime, portanto, centraliza execução cognitiva isolada e **delega**
tudo o que não é "pensar": scheduling ao `009-Scheduler`, gerenciamento de pool
ao Runtime Supervisor (`008-Agent-Lifecycle`), registro/permissão de
ferramentas ao `015-Tool-Manager`, escolha de modelo ao `017-Model-Router`, e
decisão de política ao `022-Policy` — ele é **executor governado**, nunca
fonte de verdade de nenhum desses domínios (ver Não-Responsabilidades N-01..
N-10 no `_DESIGN_BRIEF.md` §1.3).

---

## 3. Escopo

### 3.1 Escopo — Dentro (o Agent Runtime É responsável por)

| Área | Descrição |
|------|-----------|
| Materialização e ciclo de execução | Boot a partir do `AgentSpec`, condução até estado terminal (`done`/`failed`) ou quiescente (`suspended`); alvo de cold start p99 ≤ 250 ms com pool quente. |
| Loop cognitivo ReAct | `perceber → pensar → agir → observar → repetir`, com replanejamento delegado a `012-Planning` em falha recuperável ou deriva de objetivo. |
| Host MCP | Hospedagem do servidor/host MCP dentro do sandbox, exposição do catálogo de ferramentas permitido, roteamento de chamadas MCP. |
| Mediação de ferramentas | Toda invocação de ferramenta obrigatoriamente via `015-Tool-Manager` — nunca execução direta. |
| Mediação de inferência | Toda chamada de LLM obrigatoriamente via `017-Model-Router` — nunca acesso direto a provedor. |
| Recuperação/persistência de memória e contexto | Via `010-Memory`/`011-Context` — nunca acesso direto a PostgreSQL/pgvector/AGE para estado de controle. |
| Isolamento de sandbox | Namespaces, `seccomp-bpf`, `cgroups v2` (CPU/RAM/PIDs), FS read-only + escrita efêmera, *default deny* de rede salvo egress explicitamente permitido. |
| PEP local | Validação de *capability* concedida no boot para toda ação privilegiada; consulta ao PDP via Kernel/Policy quando exigido; *default deny*. |
| Telemetria e eventos | Traces/metrics/logs OTel e eventos NATS/JetStream de ciclo de vida e execução, via outbox transacional local. |
| Checkpoint e hibernação | Serialização de working memory + cursor do loop + plano, habilitando suspend/resume (*cold agent*). |
| Cotas locais | Tokens, custo USD, wall-clock, passos do loop, profundidade de recursão, fan-out de tools; abort/suspend ao esgotar. |
| Canal de controle | Health/liveness/readiness e `RuntimeControl` (gRPC + NATS) para o Supervisor comandar `boot/suspend/resume/kill/drain`. |
| Idempotência e deduplicação | Aceitação de tarefas idempotente por `Idempotency-Key`; deduplicação de eventos consumidos por `event.id`. |
| Degradação graciosa | Fallback de modelo, redução de contexto, *circuit breaker* em tools/LLM, replanejamento em falha recuperável. |
| Redação de PII | Minimização de dados em payloads de evento/log conforme metadados de sensibilidade de `010-Memory`. |

### 3.2 Escopo — Fora (o Agent Runtime NÃO é responsável por)

Reproduzido do `_DESIGN_BRIEF.md` §1.3 (não redefinido aqui, apenas resumido
para orientar o leitor deste documento de Visão):

| Área excluída | Dono real | Por quê |
|----------------|-----------|---------|
| Onde/quando/com que prioridade um agente executa (scheduling, admissão, preempção) | `009-Scheduler` | O runtime executa; quem decide alocação sob contenção é o Scheduler — o runtime é uma "caixa opaca" de execução vista de fora. |
| Gerenciamento do pool de runtimes (criar/monitorar/matar réplicas, autoscaling, cotas de pool) | Runtime Supervisor (`008-Agent-Lifecycle`) | O runtime especifica seu próprio contrato de controle, mas não decide quantas réplicas de si mesmo devem existir. |
| Treinar/fine-tunar modelos ou escolher provedor de LLM por custo/qualidade | `017-Model-Router`, `026-Cost-Optimizer` | O runtime *usa* modelos por meio de um contrato estável; a otimização de escolha é especialidade de outro módulo. |
| Registrar/versionar ferramentas, resolver permissões globais ou implementar drivers de tool | `015-Tool-Manager` | O runtime *invoca* ferramentas já registradas e autorizadas; não é um "driver manager". |
| Fonte da verdade de memória, contexto, conhecimento ou auditoria | `010`, `011`, `018/019`, `025` | O runtime é produtor/consumidor efêmero, nunca dono de dado durável. |
| Definir políticas RBAC/ABAC | `022-Policy` | O runtime é PEP, nunca PDP — aplica decisões, não as inventa. |
| Persistir event store ou implementar CQRS/projeções de leitura globais | Serviços do control plane | O runtime publica eventos; a durabilidade/projeção pertence ao control plane. |
| Expor API pública externa ao usuário final | `004-API`, API Gateway (YARP) | O runtime é chamado pelo control plane; nunca é alcançado diretamente por um usuário externo. |
| Orquestrar workflows determinísticos/sagas de múltiplos agentes | `014-Workflow` | O runtime executa **um** agente; coordenação entre agentes é de outra camada. |
| Consolidar aprendizado/*reflexion* de longo prazo em novas políticas | `023-Learning` | O runtime observa e reage no curto prazo; consolidação de longo prazo é domínio especializado. |

Esta separação segue diretamente o **Princípio 1** da Visão Global
(Separação Plano de Controle / Plano de Dados) e a analogia "processo do
sistema operacional" descrita em `../000-Vision/Vision.md` §7.1.

---

## 4. Personas Atendidas

O Agent Runtime é majoritariamente um **serviço interno do plano de dados**:
não expõe API pública, não é acessado diretamente por um usuário final, e
seus consumidores diretos são o Runtime Supervisor e outros serviços do
control plane. Ainda assim, cada persona da Visão Global
(`../000-Vision/Vision.md` §5) tem uma relação específica e concreta com este
módulo:

| Persona | Como interage com o Agent Runtime | Necessidade satisfeita |
|---------|------------------------------------|--------------------------|
| **Desenvolvedor de Agentes** | Indiretamente, via SDK (`031-SDK`) e ferramentas registradas em `015-Tool-Manager`; o comportamento observável do agente (respostas, uso de ferramentas) é produzido aqui. | Execução previsível do loop de raciocínio, com erros e telemetria compreensíveis, sem precisar entender sandbox, FSM ou outbox. |
| **Operador de Plataforma (SRE)** | Consome métricas `aios_runtime_*`, dashboards de cold start/overhead de sandbox/watchdog, e opera `runtime.*` config (pool quente, timeouts, cotas). | Visibilidade de saturação de pool, latência de boot, taxa de sandbox violation, e controles operacionais claros. |
| **Arquiteto Enterprise** | Define/valida perfis de `SandboxProfile` (egress allowlist, limites de cgroups) por tenant, e revisa a fronteira PEP local ↔ PDP (`022-Policy`). | Garantia estrutural de isolamento e de que nenhuma ação privilegiada escapa do enforcement. |
| **CISO / Compliance** | Audita eventos `sandbox_violation`, `quota_exceeded` e a trilha de `ReActStep` (via `025-Audit`); revisa a política de redação de PII em `thought`/`observation`. | Prova de que 100% das ações do agente passaram por PEP e estão registradas de forma auditável. |
| **Gestor de Custo (FinOps)** | Consome `agent.runtime.quota_exceeded` e o consumo agregado (`ModelCall.cost_usd`, `ToolInvocation.cost_usd`); define orçamentos aplicados pelo `QuotaGovernor` via `026-Cost-Optimizer`. | Contenção efetiva de custo por agente/tenant no ponto exato onde o custo é gerado (inferência e ferramentas). |
| **Runtime Supervisor / Scheduler / Lifecycle (consumidores não-humanos)** | Chamadores diretos do contrato `aios.runtime.v1` (`RuntimeControl`, `AgentExecution`) para `boot/suspend/resume/kill/drain` e submissão de tarefas. | Um contrato estável, idempotente e observável para comandar o ciclo de vida físico de um agente. |
| **Agente de IA (ocupante do runtime)** | É o próprio "processo" que o runtime hospeda — perceberá, raciocinará e agirá inteiramente dentro deste módulo. | Um ambiente isolado, com acesso governado a memória/contexto/ferramentas/modelos, capaz de suspender/retomar sem perder progresso. |

---

## 5. Princípios de Design do Agent Runtime

Estes princípios especializam, para este módulo, os 10 princípios invioláveis
da Visão Global (`../000-Vision/Vision.md` §6). Toda ADR do módulo
(`ADR-0070`..`ADR-0079`) DEVE ser avaliada contra eles.

1. **Isolamento forte por padrão, não por configuração.** Todo agente
   materializado DEVE executar em seu próprio processo, com `seccomp-bpf`,
   `cgroups v2` e namespaces aplicados desde o boot — não existe modo
   "confiável" que pule sandboxing. Especializa o Princípio 3 (recursos
   cotados e isolados) da Visão Global.
2. **Broker duplo, nunca executor direto.** O runtime NÃO DEVE invocar uma
   ferramenta sem passar por `015-Tool-Manager`, nem invocar um modelo sem
   passar por `017-Model-Router` — mesmo quando tecnicamente possível na rede
   interna. Isso mantém autorização, métricas e versionamento centralizados
   fora do runtime.
3. **Default deny estrutural (PEP local).** Nenhuma ação privilegiada (tool,
   egress, syscall cognitiva) DEVE executar sem uma *capability* válida — seja
   concedida no boot, seja confirmada dinamicamente pelo PDP. Na ausência de
   decisão, o runtime DEVE negar. Especializa o Princípio 2 da Visão Global.
4. **Efêmero e *cold-storable* por construção.** Todo agente DEVE ser
   suspensível a qualquer momento com checkpoint íntegro, liberando RAM e
   sendo retomável sem perda de continuidade — pré-requisito estrutural para
   a meta de escala V-R1 (10⁶+ agentes).
5. **Loop assíncrono, mas sempre limitado.** O loop cognitivo DEVE ser
   não-bloqueante (`asyncio`) e DEVE respeitar limites explícitos de passos,
   profundidade de recursão e fan-out de tools — um loop sem limite é, por
   definição, um vetor de P2 (planejamento frágil) da Visão Global.
6. **Outbox antes de qualquer efeito observável externo.** Nenhuma transição
   de estado DEVE ser considerada "efetivada" para o mundo externo antes de o
   evento correspondente estar gravado, ainda que localmente, no outbox
   transacional — garante atomicidade estado↔evento mesmo sob crash.
7. **Degradação graciosa antes de falha dura.** Diante de falha recuperável
   (timeout de tool, erro de modelo, deriva de objetivo), o runtime DEVE
   preferir fallback/replanejamento a abortar a sessão — falhar é o último
   recurso, não o primeiro.
8. **Observabilidade sem exceção.** 100% dos passos do loop DEVEM produzir
   trace, métrica e evento — inclusive passos que terminam em erro ou
   negação, que são o dado de segurança mais valioso. Especializa o
   Princípio 5 da Visão Global.
9. **Multi-tenant e cotado por construção.** Toda entidade e todo evento
   carrega `tenant_id`; nenhuma lógica de aplicação decide isolamento — o
   contexto autenticado e a política o impõem, nunca uma convenção de código.
10. **Fronteiras de responsabilidade são invioláveis.** O runtime NÃO DEVE
    absorver responsabilidades de Scheduler, Lifecycle, Tool Manager, Model
    Router ou Policy mesmo quando isso pareceria simplificar um fluxo
    específico — a estabilidade e a auditabilidade de longo prazo dependem
    dessas fronteiras.

---

## 6. Relação com a Visão Global do AIOS

O Agent Runtime é o módulo nomeado explicitamente na Visão Global como
"PLANO DE DADOS — AGENT RUNTIME (Python · 007)" no diagrama conceitual de
10.000 pés (`../000-Vision/Vision.md` §7) e na tabela de subsistemas §7.1
(análogo "Processo/thread"). Três relações merecem destaque:

- **V-R1 (escala a 10⁶+ agentes concorrentes distribuídos)**: o modelo de
  agente efêmero e *cold-storable* (checkpoint em MinIO/PostgreSQL via
  control plane + liberação total de RAM em hibernação) é a contribuição
  direta do runtime a essa meta — a maioria dos agentes DEVE poder existir
  como sessão `suspended` sem custo de RAM, materializável sob demanda em
  ≤ 250 ms (ver `_DESIGN_BRIEF.md` §10, NFR-004/NFR-006/NFR-010).
- **V-R3 (isolamento forte entre tenants e entre agentes)**: o runtime é o
  ponto de aplicação estrutural dessa meta — sandbox por processo (seccomp,
  cgroups, namespaces, FS efêmero, egress *default deny*) garante que um
  agente comprometido não afete outro agente nem outro tenant no mesmo nó
  físico (NFR-008/NFR-012 do brief).
- **V-R4 (auditoria imutável de 100% das ações privilegiadas)**: cada
  `ReActStep`, `ToolInvocation` e `ModelCall` é registrado de forma
  reconstruível, com correlação `traceparent`/`tenant`/`agent` — o runtime
  alimenta diretamente `025-Audit` com a granularidade fina de que essa meta
  depende.

O Agent Runtime também opera como o **ponto de tradução** entre a linguagem
de "processo" do SO clássico e a linguagem de "agente cognitivo" do AIOS: onde
um kernel Unix agenda *quanta* de CPU para um processo, o `CognitiveLoopEngine`
agenda *passos* de raciocínio para um agente; onde um processo Unix é
suspenso para *swap* em disco, uma `AgentSession` é suspensa para checkpoint
em MinIO. Essa analogia, descrita na Visão Global (§7.1), é o fio condutor de
todo o desenho interno do módulo (ver `./Architecture.md`).

---

## 7. Não-Objetivos Explícitos (deste módulo)

Além das Não-Responsabilidades já listadas na §3.2 (que definem *fronteira de
propriedade*), o módulo declara explicitamente os seguintes não-objetivos de
**produto/design**, para evitar expansão de escopo (*scope creep*) ao longo do
tempo:

- O Agent Runtime NÃO tem por objetivo se tornar um **orquestrador
  multi-agente**; ele executa exatamente **um** agente por processo. Fluxos
  determinísticos entre múltiplos agentes pertencem a `014-Workflow`.
- O Agent Runtime NÃO tem por objetivo implementar sua própria lógica de
  **seleção de modelo por custo/qualidade** — ele consulta
  `017-Model-Router` e aceita a decisão recebida, no máximo aplicando
  fallback quando o Router sinaliza indisponibilidade.
- O Agent Runtime NÃO tem por objetivo manter um **registro de ferramentas**
  próprio nem cache de permissões de longo prazo — o catálogo de ferramentas
  do MCP host é recarregado a partir de `015-Tool-Manager` a cada atualização
  (`aios.<t>.tool.registry.updated`), nunca definido localmente.
- O Agent Runtime NÃO tem por objetivo ser a **fonte de verdade durável** de
  nenhuma entidade — `AgentSession`, `ReActStep`, `ToolInvocation`,
  `ModelCall` e `Checkpoint` residem, de forma autoritativa e durável, no
  control plane (PostgreSQL com RLS); o runtime mantém apenas cópias
  efêmeras em sandbox.
- O Agent Runtime NÃO tem por objetivo, na v1.0, suportar **paralelismo
  intra-agente sem limite** — o fan-out de invocação de ferramentas é sempre
  limitado por `loop.max_tool_fanout`, mesmo quando a infraestrutura
  subjacente permitiria mais.

---

## 8. Riscos Estratégicos Específicos do Módulo

Complementando os riscos globais da Visão (`../000-Vision/Vision.md` §10), o
Agent Runtime carrega riscos próprios por hospedar código de raciocínio
potencialmente adversarial (prompt injection) e por ser, ao mesmo tempo, o
componente mais numeroso da plataforma (um processo por agente ativo):

| Risco | Descrição | Mitigação | Referência |
|-------|-----------|-----------|------------|
| RSK-A01 | **Escape de sandbox** por syscall proibida ou uso indevido de ferramenta para acessar recursos fora do compartimento. | `seccomp-bpf` allowlist restritiva + `cgroups v2` + namespaces + egress *default deny*; kill imediato em violação (`sandbox_violation`, `AIOS-RUNTIME-0015`). | `_DESIGN_BRIEF.md` §9, §12.2, ADR-0071 |
| RSK-A02 | **Cold start acima da meta** sob pico de demanda, degradando a experiência de agentes recém-materializados. | Pool quente (`runtime.pool.warm_size`) + *copy-on-write* de intérprete pré-forkado (ADR-0073); meta p99 ≤ 250 ms com pool quente. | `_DESIGN_BRIEF.md` §7.2 (NFR-001), ADR-0073 |
| RSK-A03 | **Noisy neighbor**: um agente com defeito de lógica (loop, fan-out excessivo, prompt injection) degrada outros agentes no mesmo nó. | `cgroups` (CPU/RAM/PIDs) por agente; cotas locais de tokens/USD/wall-clock/passos; `max_sessions_per_node`. | `_DESIGN_BRIEF.md` §10, §12.2 |
| RSK-A04 | **Checkpoint corrompido** impedindo retomada correta de um agente hibernado. | Checksum sha256 versionado; rejeição de resume com checkpoint inválido, retomada do checkpoint anterior íntegro (`AIOS-RUNTIME-0010`). | `_DESIGN_BRIEF.md` §9, ADR-0074 |
| RSK-A05 | **Dupla execução** de uma mesma sessão em caso de corrida entre réplicas do runtime (ex.: resume concorrente). | Lease distribuído (Redis) por `session_id`; `resume` só ocorre com aquisição do lease — exatamente um runtime ativo por sessão. | `_DESIGN_BRIEF.md` §10, ADR-0079 |
| RSK-A06 | **Loop travado/deadlock** consumindo recursos indefinidamente sem produzir progresso observável. | Watchdog (`watchdog.stuck_timeout_ms`) detecta *stuck*; tenta replanejar, senão mata a sessão (`AIOS-RUNTIME-0008`). | `_DESIGN_BRIEF.md` §9, ADR-0077 |
| RSK-A07 | **Vazamento de PII** em `thought`/`observation` propagado a eventos/logs. | Redação de PII conforme metadados de sensibilidade de `010-Memory` (`pii.redaction.enabled`); `detail` de erro nunca carrega dado sensível. | `_DESIGN_BRIEF.md` §12.3 |

---

## 9. Referências

- Visão Global do AIOS: `../000-Vision/Vision.md`
- Arquitetura Global: `../001-Architecture/Architecture.md`
- Glossário: `../040-Glossary/Glossary.md`
- Contratos centrais: `../003-RFC/RFC-0001-Architecture-Baseline.md`
- Design Brief interno (fonte de verdade do módulo): `./_DESIGN_BRIEF.md`
- Índice do módulo: `./README.md`
- Módulos acoplados: `../006-Kernel/`, `../008-Agent-Lifecycle/`,
  `../009-Scheduler/`, `../010-Memory/`, `../011-Context/`,
  `../012-Planning/`, `../015-Tool-Manager/`, `../017-Model-Router/`,
  `../020-Communication/`, `../021-Security/`, `../022-Policy/`,
  `../024-Observability/`, `../025-Audit/`.

---

*Fim de `Vision.md`. Próximo documento na cadeia do módulo:*
`./Architecture.md`.
