---
Documento: Vision
Módulo: 006-Kernel
Status: Draft
Versão: 0.1
Última atualização: 2026-07-20
Responsável (RACI-A): Arquiteto do Módulo 006-Kernel
ADRs relacionados: ADR-0060, ADR-0061, ADR-0062, ADR-0063, ADR-0064, ADR-0065, ADR-0066, ADR-0067, ADR-0068, ADR-0069
RFCs relacionados: RFC-0001, RFC-0006, RFC-0007
Depende de: 000-Vision, 001-Architecture, 022-Policy, 009-Scheduler, 008-Agent-Lifecycle, 020-Communication, 021-Security, 024-Observability, 025-Audit
---

# 006-Kernel — Visão do Módulo

> "O Kernel está para o agente assim como o kernel do Linux está para o processo:
> a fronteira privilegiada através da qual todo pedido de recurso passa, é
> autenticado contra o contexto correto, autorizado por política, cotado e
> registrado." — analogia condutora deste módulo (ver `../040-Glossary/Glossary.md`,
> termos **Kernel (Cognitivo)** e **ACB**).

---

## 1. Propósito do Módulo

O **006-Kernel** é o **núcleo cognitivo** do AIOS (ver Visão Global,
`../000-Vision/Vision.md` §7.1: "Kernel — análogo ao Kernel Linux — gerencia
ciclo de vida do agente, syscalls cognitivas"). Sua razão de existir é dar ao
AIOS uma **fronteira de confiança única e não contornável**: nenhum agente
acessa memória, contexto, planejamento, ferramentas ou modelos de inferência
diretamente — todo acesso passa por uma **syscall cognitiva** exposta pelo
Kernel, que a autentica, autoriza, cota, executa (via *broker* aos módulos
donos) e audita.

Sem essa fronteira, o AIOS degeneraria em um conjunto de bibliotecas chamadas
ad-hoc por cada agente — exatamente o modelo que a Visão Global rejeita
explicitamente ao afirmar que "frameworks atuais compõem chamadas, mas não
gerenciam recursos, não isolam falhas, não escalonam sob contenção e não
governam" (`../000-Vision/Vision.md` §2). O Kernel é o componente que torna essa
afirmação verdadeira na prática: é onde **governança, cotas e ciclo de vida se
tornam código executável**, não apenas política de intenção.

O módulo entrega três capacidades indissociáveis:

1. **ABI de syscalls cognitivas** — uma superfície estável de 11 verbos
   (`spawn`, `kill`, `suspend`, `resume`, `remember`, `recall`, `plan`,
   `invoke_tool`, `route_model`, `get_quota`, `checkpoint`) exposta via REST
   (externo) e gRPC (interno), versionada e compatível para trás.
2. **ACB (Agent Control Block)** como estrutura de controle autoritativa —
   identidade, estado de ciclo de vida, prioridade, cotas, ponteiros de
   memória/contexto e política — análoga ao PCB de um sistema operacional
   clássico.
3. **Enforcement** de dois tipos de restrição em toda syscall privilegiada:
   **capability** (é permitido? — via PEP consultando o PDP do `022-Policy`) e
   **cota** (há orçamento? — tokens, cpu-ms, memória, custo, ferramentas,
   syscalls/s).

---

## 2. Problema que o Módulo Resolve

A Visão Global identifica oito limitações estruturais dos agentes de IA atuais
(`../000-Vision/Vision.md` §2, P1–P8). O Kernel é a resposta direta a três
delas — e o pré-requisito estrutural para que as demais sejam sequer
governáveis:

| Limitação da Visão | Como o Kernel endereça |
|---------------------|------------------------|
| **P4 — Ausência de gerenciamento de recursos** | O Kernel é o único ponto onde cotas de tokens/cpu-ms/memória/custo/ferramentas/syscalls-por-segundo são reservadas, consumidas e liberadas atomicamente (`ResourceQuotaManager`), por agente e por tenant. Sem isso, um agente com defeito de lógica (loop, prompt injection, ferramenta descontrolada) degrada toda a plataforma — o clássico "noisy neighbor". |
| **P8 — Governança fraca** | O Kernel é o **PEP** (Policy Enforcement Point) de todo o AIOS para ações de agente: nenhuma syscall privilegiada executa sem consulta prévia ao PDP (`022-Policy`), com postura **default deny**. Isso torna "governança" uma propriedade estrutural do sistema, não uma convenção que cada módulo poderia esquecer de aplicar. |
| **P7 — Baixa observabilidade** | Por ser o ponto de passagem obrigatório, o Kernel é também o ponto de instrumentação obrigatório: toda syscall gera trace OTel, métrica e registro de auditoria (`025-Audit`), tornando qualquer ação de agente reconstruível post-mortem. |

Além disso, o Kernel resolve um problema de **engenharia de plataforma** que
antecede os problemas de IA: como coordenar, de forma consistente e
recuperável, o ciclo de vida de um agente que depende de decisões de **três**
subsistemas distintos (admissão do `009-Scheduler`, materialização do
`008-Agent-Lifecycle`, e a própria máquina de estados do ACB) sem que essa
coordenação vire uma fonte de inconsistência distribuída. A resposta é o
padrão **saga com compensação** (ADR-0064) orquestrado pelo
`LifecycleCoordinator`.

### 2.1 Por que não resolver isso em cada módulo de recurso individualmente

Uma alternativa de design seria deixar que `010-Memory`, `012-Planning`,
`015-Tool-Manager` etc. aplicassem capability e cota cada um por si. Essa
alternativa foi descartada porque:

- **Duplicaria a lógica de PEP** em N módulos, com risco de divergência de
  postura de segurança (um módulo esquece de checar; *default deny* deixa de
  ser universal).
- **Fragmentaria o ACB**: sem um registro central de estado de agente, não há
  como responder "este agente está `Suspended`? pode invocar ferramenta?" sem
  consultar N fontes.
- **Impediria contabilidade de cota transversal**: cotas de tokens/custo são
  agregadas por agente e por tenant *através* de múltiplos tipos de recurso;
  um contador por módulo não fecha essa conta de forma atômica.

O Kernel centraliza essas três preocupações (identidade/estado, capability,
cota) e **delega** a execução do recurso em si ao módulo dono — ele é
**broker governado**, nunca executor de lógica de negócio de memória,
planejamento, ferramentas ou inferência (ver Não-Responsabilidade NR-05 no
`_DESIGN_BRIEF.md`).

---

## 3. Escopo

### 3.1 Escopo — Dentro (o Kernel É responsável por)

| Área | Descrição |
|------|-----------|
| ABI de syscalls | Os 11 verbos cognitivos, seus contratos REST/gRPC, versionamento e compatibilidade (`RFC-0006`, a propor). |
| ACB | Modelo de dados, máquina de estados canônica e concorrência otimista do Agent Control Block (`RFC-0007`, a propor). |
| PEP (capability enforcement) | Montagem do contexto de decisão e consulta ao PDP (`022-Policy`) antes de toda syscall privilegiada; *default deny*. |
| Cotas de recurso | Reserva/consumo/liberação atômicos de tokens, cpu-ms, memória, custo, ferramentas e syscalls/s, por agente e por tenant. |
| Coordenação de ciclo de vida | Orquestração de `spawn/suspend/resume/kill/checkpoint` como sagas, delegando admissão ao Scheduler e materialização ao Lifecycle. |
| Brokering de recursos | Encaminhamento de `remember/recall/plan/invoke_tool/route_model` aos módulos donos, após capability + cota aplicados. |
| Emissão de eventos | Publicação de eventos de ciclo de vida e decisão (`aios.<tenant>.agent.*`) via Outbox transacional. |
| Idempotência | Garantia de efeito único por `Idempotency-Key` em toda mutação exposta. |
| Isolamento multi-tenant | Rejeição de qualquer `tenant` divergente do contexto autenticado. |

### 3.2 Escopo — Fora (o Kernel NÃO é responsável por)

Reproduzido do `_DESIGN_BRIEF.md` §1.3 (não redefinido aqui, apenas resumido
para orientar o leitor deste documento de Visão):

| Área excluída | Dono real | Por quê |
|----------------|-----------|---------|
| Loop de raciocínio (ReAct) e sandbox de execução | `007-Agent-Runtime` | O Kernel governa o *acesso a recursos*, não o *pensamento* do agente — separação Plano de Controle / Plano de Dados (Princípio 1 da Visão Global). |
| Decisão de *onde/quando* executar (placement, preempção, admissão de fila) | `009-Scheduler` | O Kernel *pede* um slot; quem decide alocação sob contenção é o Scheduler — evita que o Kernel acumule lógica de otimização que não é sua especialidade. |
| Materialização/hibernação física do processo do agente | `008-Agent-Lifecycle` + Runtime Supervisor | O Kernel mantém o estado *lógico* (ACB); o Lifecycle mantém o estado *físico* (processo, snapshot). |
| Decisão de política (ser PDP) | `022-Policy` | O Kernel é **PEP**, nunca **PDP** — separação estrita evita que regras de negócio de segurança fiquem espalhadas e inconsistentes entre módulos. |
| Execução de memória/contexto/planejamento/ferramentas/inferência | `010`, `011`, `012`, `015`, `017` | O Kernel encaminha; a lógica de domínio pertence a quem entende o recurso. |
| Persistência da trilha de auditoria imutável | `025-Audit` | O Kernel *emite* eventos de decisão; a durabilidade/imutabilidade é responsabilidade especializada do Audit. |
| Identidade federada / emissão-validação de tokens OAuth2/OIDC | `021-Security` + Gateway | O Kernel consome claims já validadas; não deve reimplementar AuthN. |
| Definição de orçamento de custo por tenant | `026-Cost-Optimizer` | O Kernel *aplica* a cota corrente; quem define o valor do orçamento é o Cost Optimizer. |

Esta separação segue diretamente o **Princípio 1** da Visão Global
(Separação Plano de Controle / Plano de Dados) e o padrão arquitetural
"Kernel mínimo, drivers plugáveis" descrito em
`../001-Architecture/Architecture.md`.

---

## 4. Personas Atendidas

O Kernel é majoritariamente um módulo de **infraestrutura interna**: seus
consumidores diretos são outros módulos e, indiretamente, todas as personas da
Visão Global (`../000-Vision/Vision.md` §5). A tabela abaixo detalha a relação
específica de cada persona com o Kernel:

| Persona | Como interage com o Kernel | Necessidade satisfeita |
|---------|------------------------------|--------------------------|
| **Desenvolvedor de Agentes** | Indiretamente, via SDK (`031-SDK`) que traduz chamadas de alto nível em syscalls REST/gRPC do Kernel. | Uma API estável e previsível para *spawn*, memória, planejamento e ferramentas, sem se preocupar com FSM, sagas ou cotas. |
| **Operador de Plataforma (SRE)** | Consome métricas `aios_kernel_*`, dashboards e alertas derivados da telemetria do Kernel; opera o `kernel.*` config. | Visibilidade de saturação (cotas, PEP, latência de spawn) e controles operacionais (fail-mode do PEP, timeouts). |
| **Arquiteto Enterprise** | Define/valida a fronteira PEP↔PDP e os *bindings* de política aplicados via `policy_ref` do ACB. | Garantia estrutural de *default deny* e rastreabilidade de toda decisão de autorização. |
| **CISO / Compliance** | Audita `syscall_log`, eventos `agent.syscall.denied` e a trilha emitida ao `025-Audit`. | Prova de que 100% das ações privilegiadas passaram por autorização e estão registradas (NFR-010, NFR-011 do brief). |
| **Gestor de Custo (FinOps)** | Consome `agent.quota.exceeded`/`agent.quota.warning` e o saldo via `get_quota`; define limites via `026-Cost-Optimizer` que o Kernel aplica. | Contenção efetiva de custo por agente/tenant sem depender de disciplina manual. |
| **Agente de IA (consumidor não-humano)** | Chamador direto (via Agent Runtime) de todos os 11 verbos da ABI. | Um contrato estável, idempotente e previsível para obter qualquer recurso cognitivo. |

---

## 5. Princípios de Design do Kernel

Estes princípios especializam, para este módulo, os 10 princípios invioláveis
da Visão Global (`../000-Vision/Vision.md` §6). Toda ADR do módulo
(`ADR-0060`..`ADR-0069`) DEVE ser avaliada contra eles.

1. **Kernel mínimo, brokerado, nunca executor.** O Kernel NÃO DEVE conter
   lógica de negócio de memória, planejamento, ferramentas ou inferência — ele
   valida, cota, autoriza e encaminha. Isso mantém o núcleo pequeno,
   auditável e substituível por partes (especializa o Princípio 7 — 
   Extensibilidade sem fork — da Visão Global).
2. **Default deny estrutural.** Nenhuma syscall privilegiada DEVE executar
   sem decisão explícita `allow` do PDP. Na ausência de decisão (PDP
   indisponível), o Kernel DEVE **negar com segurança**
   (`kernel.pep.fail_mode=closed` por padrão) — especializa o Princípio 2 da
   Visão Global.
3. **Cotas são atômicas e nunca otimistas.** Reserva/consumo/liberação de
   cota DEVE ocorrer sob primitivas atômicas (token-bucket em Redis via
   script Lua), nunca "verificar depois" — especializa o Princípio 3 (recursos
   cotados e isolados).
4. **Idempotência antes de performance.** Toda mutação exposta DEVE ser seguray
   para repetir; otimizações de latência NÃO DEVEM comprometer essa garantia
   — especializa o Princípio 6 (idempotência e recuperação) e a RFC-0001 §5.5.
5. **Estado autoritativo único (ACB), fonte de verdade única (PostgreSQL).**
   O Redis é *cache quente*, nunca fonte de verdade; qualquer inconsistência
   DEVE resolver a favor do PostgreSQL.
6. **Coordenação por saga, não por transação distribuída.** `spawn`,
   `resume` e `kill` cruzam limites de serviço (Scheduler, Lifecycle); o
   Kernel DEVE modelar essa coordenação como saga compensável, nunca como
   commit de duas fases síncrono.
7. **Observabilidade sem exceção.** 100% das syscalls DEVEM produzir trace,
   métrica e registro de auditoria — não há "modo silencioso" no Kernel,
   nem mesmo para syscalls negadas (que são, aliás, o dado de segurança mais
   valioso).
8. **Multi-tenant por construção, não por convenção.** Toda entidade do
   Kernel carrega `tenant_id` com RLS; nenhum código de aplicação decide
   isolamento — o banco de dados o impõe.
9. **Cold por padrão, quente sob demanda.** Um agente ocioso DEVE poder migrar
   para `Hibernated` (estado durável, zero RAM) e voltar a `Running` sob
   demanda com latência previsível — pré-requisito estrutural para a meta de
   escala V-R1 da Visão Global (10⁶+ agentes).
10. **Fronteiras de responsabilidade são invioláveis.** O Kernel NÃO DEVE
    absorver responsabilidades de Scheduler, Lifecycle, Policy ou dos módulos
    de recurso mesmo quando isso pareceria simplificar um fluxo específico —
    a estabilidade de longo prazo da ABI depende dessas fronteiras.

---

## 6. Relação com a Visão Global do AIOS

O Kernel é o módulo que a Visão Global nomeia explicitamente como o análogo do
"Kernel Linux" na tabela de subsistemas (`../000-Vision/Vision.md` §7.1) e o
primeiro componente citado no diagrama conceitual de 10.000 pés (§7), dentro do
**Plano de Controle (.NET 10)**. Três relações merecem destaque:

- **V-R1 (escala a 10⁶+ agentes)**: o modelo de ACB *hot/cold* (Redis
  quente + PostgreSQL/MinIO frio) e o sharding determinístico
  (`hash(tenant,agent) mod N`) são a contribuição direta do Kernel a essa meta
  — a maioria dos agentes em repouso DEVE poder existir como ACB `Hibernated`
  sem custo de RAM (ver `../000-Vision/Vision.md` §9, V-R1; ver §10 do
  `_DESIGN_BRIEF.md`).
- **V-R4 (auditoria imutável de 100% das ações privilegiadas)**: o Kernel é o
  ponto de aplicação estrutural dessa meta — por ser passagem obrigatória de
  toda syscall, é o lugar correto para garantir cobertura de 100%
  (NFR-010/NFR-011 do brief), em vez de depender de instrumentação
  espalhada por múltiplos módulos.
- **V-R10 (RTO ≤ 15 min, RPO ≤ 5 min do plano de controle)**: o ACB, sendo a
  estrutura de estado mais crítica do plano de controle, herda diretamente
  essa meta (ver `_DESIGN_BRIEF.md` §9); a estratégia de recuperação do
  Kernel (PostgreSQL HA + Outbox) é uma instância concreta da meta de
  disponibilidade da Visão.

O Kernel também opera como o **ponto de tradução** entre a linguagem de
"processo" do SO clássico e a linguagem de "agente cognitivo" do AIOS: onde um
kernel Unix expõe `fork`/`exec`/`wait`/`kill`, o Kernel do AIOS expõe
`spawn`/`resume`/`checkpoint`/`kill` — e onde um kernel Unix gerencia memória
física via MMU, o Kernel do AIOS gerencia ponteiros para memória *cognitiva*
(`memory_ptr`) delegando a implementação ao `010-Memory`. Essa analogia,
descrita na Visão Global (§7.1), é o fio condutor de todo o desenho interno do
módulo.

---

## 7. Não-Objetivos Explícitos (deste módulo)

Além das Não-Responsabilidades já listadas na §3.2 (que definem *fronteira de
propriedade*), o módulo declara explicitamente os seguintes não-objetivos de
**produto/design**, para evitar expansão de escopo (*scope creep*) ao longo do
tempo:

- O Kernel NÃO tem por objetivo se tornar um **motor de regras de negócio**
  genérico; ele DEVE permanecer restrito às 11 syscalls cognitivas definidas
  na ABI (`RFC-0006`, a propor). Novos verbos exigem RFC, não extensão ad-hoc.
- O Kernel NÃO tem por objetivo prover **UI** de qualquer natureza — consumo
  humano do ACB ocorre via `032-WebConsole`, nunca diretamente do Kernel.
- O Kernel NÃO tem por objetivo ser o **cache semântico** de inferência (isso
  é `011-Context`) nem o **otimizador de custo** (isso é
  `026-Cost-Optimizer`) — o Kernel apenas aplica os limites que esses módulos
  definem.
- O Kernel NÃO tem por objetivo suportar *scheduling* cooperativo dentro do
  próprio processo do agente (isso é responsabilidade do loop ReAct em
  `007-Agent-Runtime`); o Kernel enxerga o agente como uma caixa opaca de
  estado de ciclo de vida.
- O Kernel NÃO tem por objetivo, na v1.0, suportar *transações distribuídas
  cruzando tenants* — cada syscall opera estritamente dentro da fronteira de
  um único `tenant_id` (RFC-0001 §6).

---

## 8. Riscos Estratégicos Específicos do Módulo

Complementando os riscos globais da Visão (`../000-Vision/Vision.md` §10),
o Kernel carrega riscos próprios por concentrar tanta responsabilidade
crítica em um único ponto de passagem:

| Risco | Descrição | Mitigação | Referência |
|-------|-----------|-----------|------------|
| RSK-K01 | O Kernel se tornar um **SPOF lógico** por concentrar PEP + cota + FSM. | Serviço stateless replicado; estado externalizado (PostgreSQL HA + Redis); sharding determinístico. | `_DESIGN_BRIEF.md` §9–§10 |
| RSK-K02 | *Overshoot* de cota sob alta concorrência (corrida entre réplicas). | Token-bucket atômico via script Lua no Redis; meta de overshoot ≤ 1% (NFR-009). | `_DESIGN_BRIEF.md` §10 |
| RSK-K03 | Indisponibilidade do PDP (`022-Policy`) travar todo o AIOS. | `fail_mode=closed` configurável + cache de decisão com TTL curto + circuit breaker no `PolicyClient`. | `_DESIGN_BRIEF.md` §9, ADR-0063 |
| RSK-K04 | Inconsistência entre estado do ACB e evento publicado (ex.: crash entre commit e publish). | Padrão Outbox transacional + relay; consumidores deduplicam por `event.id`. | `_DESIGN_BRIEF.md` §9, ADR-0066 |
| RSK-K05 | Crescimento descontrolado do número de ACBs (agentes órfãos nunca terminados). | Hibernação automática por ociosidade (`kernel.hibernation.idle_ttl_ms`) + varredura periódica; retenção de `Terminated` configurável. | `_DESIGN_BRIEF.md` §8 |

---

## 9. Referências

- Visão Global do AIOS: `../000-Vision/Vision.md`
- Arquitetura Global: `../001-Architecture/Architecture.md`
- Glossário: `../040-Glossary/Glossary.md`
- Contratos centrais: `../003-RFC/RFC-0001-Architecture-Baseline.md`
- Design Brief interno (fonte de verdade do módulo): `./_DESIGN_BRIEF.md`
- Índice do módulo: `./README.md`
- Módulos acoplados: `../022-Policy/`, `../009-Scheduler/`, `../008-Agent-Lifecycle/`,
  `../007-Agent-Runtime/`, `../010-Memory/`, `../011-Context/`, `../012-Planning/`,
  `../015-Tool-Manager/`, `../017-Model-Router/`, `../026-Cost-Optimizer/`,
  `../024-Observability/`, `../025-Audit/`, `../020-Communication/`.

---

*Fim de `Vision.md`. Próximo documento na cadeia do módulo:*
`./Architecture.md`.
