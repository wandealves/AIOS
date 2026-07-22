---
Documento: Architecture
Módulo: 007-Agent-Runtime
Status: Draft
Versão: 0.1
Última atualização: 2026-07-20
Responsável (RACI-A): Arquiteto do Módulo 007 — Agent Runtime
ADRs relacionados: ADR-0070, ADR-0071, ADR-0072, ADR-0073, ADR-0074, ADR-0075, ADR-0076, ADR-0077, ADR-0078, ADR-0079 (deste módulo, a propor); ADR-0001, ADR-0002, ADR-0003 (globais)
RFCs relacionados: RFC-0001 (baseline); RFC-0070 (Runtime↔Supervisor Control Protocol, a propor); RFC-0071 (MCP Host Sandboxing Profile, a propor)
Depende de: 006-Kernel, 008-Agent-Lifecycle, 009-Scheduler, 010-Memory, 011-Context, 012-Planning, 015-Tool-Manager, 017-Model-Router, 020-Communication, 021-Security, 022-Policy, 024-Observability, 025-Audit
---

# 007-Agent-Runtime — Arquitetura

> Este documento detalha a arquitetura interna do Agent Runtime (plano de
> dados, Python) em notação C4 adaptada para ASCII. Ele **não redefine**
> contratos centrais (URN, envelope de evento, envelope de erro, idempotência,
> correlação, subjects) — estes são consumidos de
> `../003-RFC/RFC-0001-Architecture-Baseline.md`. Este documento é derivado de
> `./_DESIGN_BRIEF.md` (fonte única de verdade do módulo) e **não pode
> contradizê-lo**.

## Índice

1. Papel do Agent Runtime na arquitetura global
2. C4 Nível 1 — Contexto do Sistema (Agent Runtime)
3. C4 Nível 2 — Contêineres
4. C4 Nível 3 — Componentes
5. Tabela de responsabilidades por componente
6. Fronteiras e regras de dependência
7. Padrões arquiteturais adotados
8. Tecnologias e justificativas
9. Modelo de camadas e travessia de planos
10. Riscos arquiteturais e trade-offs
11. Alternativas descartadas
12. Referências e ADRs

---

## 1. Papel do Agent Runtime na Arquitetura Global

O Agent Runtime (`007`) é o **plano de dados (data plane)** do AIOS, escrito
em **Python**. É o processo que materializa um agente e conduz seu **loop
cognitivo ReAct** (`perceber → pensar → agir(tool) → observar → repetir`)
dentro de um **sandbox** isolado por processo, hospedando o **MCP host** que
expõe ferramentas ao raciocínio do agente. Cada instância do Agent Runtime
executa **exatamente um agente por vez** — isolamento forte análogo ao
processo de um sistema operacional clássico (ver Glossário: *Agent Runtime*,
*Data Plane*, *Sandbox*, *ReAct*).

O Agent Runtime **não decide nada sobre governança**: ele é **PEP local**
(Policy Enforcement Point) que aplica *capabilities* concedidas no boot e
consulta o PDP (`022-Policy`, via Kernel) quando exigido — nunca é PDP. Ele
**não fala com LLM** exceto através do `ModelRouterClient → 017-Model-Router`,
**não executa ferramenta externa** exceto através do `ToolInvoker →
015-Tool-Manager`, e **não persiste estado de controle** diretamente em
PostgreSQL/pgvector/AGE — toda persistência durável (`AgentSession`,
`ReActStep`, `ToolInvocation`, `ModelCall`, `Checkpoint`) trafega por gRPC/NATS
aos serviços do control plane (`010-Memory`, `025-Audit`) que a possuem. Um
processo separado do control plane, o **Runtime Supervisor** (.NET 10,
coordenado por `008-Agent-Lifecycle`), gerencia o **pool** de instâncias de runtime
(cria/monitora/mata, faz *placement*, aplica cotas de nível de pool); este
módulo especifica apenas o **processo runtime Python** e o **contrato**
(`RuntimeControl`, `AgentExecution`) que ele expõe ao Supervisor.

Essa separação replica, no domínio cognitivo, a fronteira clássica
*user-space* (loop de raciocínio, sandboxed) ↔ *kernel/control-space*
(governança, escalonamento, ciclo de vida) — mas invertida em relação ao
`006-Kernel`: aqui o Agent Runtime **é** o espaço isolado e não-privilegiado, e
o Kernel/Supervisor/Policy compõem o espaço de controle que o supervisiona.

---

## 2. C4 — Nível 1: Contexto do Sistema (Agent Runtime)

```
                     gRPC (comando, mTLS)          NATS (heartbeat/eventos, mTLS)
        ┌───────────────────────────────────┐   ┌───────────────────────────────┐
        │      Runtime Supervisor (.NET 10)  │   │  020-Communication (NATS/JS)  │
        │      cria/monitora/mata pool ·     │◀─▶│  transporte de heartbeat e     │
        │      placement · cotas de pool     │   │  eventos de execução           │
        └───────────────────┬────────────────┘   └───────────────┬────────────────┘
                             │ Boot/Suspend/Resume/Kill/Drain      │ publica/consome
                             ▼                                     ▼
        ╔══════════════════════════════ 007-AGENT RUNTIME (Python) ═══════════════════════════╗
        ║       processo isolado por agente · loop ReAct · MCP host · sandbox                  ║
        ╚═══╤════════════╤════════════╤═══════════╤════════════╤════════════╤═══════════╤═════╝
            │ gRPC        │ gRPC        │ gRPC       │ gRPC        │ gRPC       │ gRPC      │ OTLP
            ▼             ▼            ▼            ▼             ▼            ▼           ▼
      010-Memory   011-Context   012-Planning  015-Tool-Manager 017-Model-Router 022-Policy  024-Obs./
      (recall/     (montagem de  (plano/       (única via de     (única via de   (PDP, via  025-Audit
       remember)    contexto)     replan)       execução de tool) inferência LLM) Kernel/006)
```

**Atores e sistemas em fronteira**

| Ator/Sistema | Papel na interação com o Agent Runtime | Protocolo |
|--------------|------------------------------------------|-----------|
| Runtime Supervisor (.NET 10, `008-Agent-Lifecycle`) | Cria/monitora/mata a instância; comanda `Boot/Suspend/Resume/Kill/Drain`; recebe heartbeat. | gRPC (comando) + NATS (heartbeat) |
| `009-Scheduler` (indireto, via Supervisor) | Decide admissão/placement; o runtime nunca é consultado diretamente. | — (fora do escopo deste módulo) |
| `010-Memory` | Recupera/persiste itens de memória (working/short/long/semantic/episodic/procedural). | gRPC |
| `011-Context` | Monta janela de contexto, compressão, cache semântico. | gRPC |
| `012-Planning` | Decompõe objetivo em plano; replaneja em falha/deriva. | gRPC |
| `015-Tool-Manager` | Única via de execução de ferramentas (autorização, métricas, versão). | gRPC/NATS |
| `017-Model-Router` | Única via de inferência LLM (escolha de modelo, streaming, fallback). | gRPC |
| `022-Policy` (via Kernel/`006`) | PDP consultado pelo PEP local (`CapabilityEnforcer`) quando a decisão não é resolvível por *capability* cacheada. | gRPC (indireto) |
| `021-Security` | Fornece perfis de sandbox, segredos escopados por tenant, certificados mTLS. | — (config-time) |
| NATS/JetStream (`020-Communication`) | Barramento de eventos de ciclo de vida/execução; consumo de comandos de lifecycle/política/registro. | NATS/JetStream |
| `024-Observability` / `025-Audit` | Destino de telemetria OTel e trilha de auditoria (via eventos). | OTLP / NATS |

---

## 3. C4 — Nível 2: Contêineres

> Diferente do Kernel (serviço único do control plane), o Agent Runtime é
> publicado como **N processos efêmeros de curta/média duração**, um por
> agente materializado, geridos por um **pool** mantido pelo Runtime
> Supervisor. Não há um "contêiner de aplicação" único — a unidade de
> implantação é a **instância de runtime**.

```
┌───────────────────────────────────────────────────────────────────────────────────┐
│                    007-AGENT RUNTIME — CONTÊINERES                                │
│                                                                                     │
│  ┌───────────────────────────┐  ┌───────────────────────────┐  ┌────────────────┐ │
│  │ Runtime Instance #1        │  │ Runtime Instance #2        │  │  Runtime       │ │
│  │ (Python 3.12+, asyncio,    │  │ (Python 3.12+, asyncio,    │  │  Instance #N   │ │
│  │  1 processo = 1 agente)    │  │  1 processo = 1 agente)    │  │  ...           │ │
│  │  seccomp-bpf + cgroups v2  │  │  seccomp-bpf + cgroups v2  │  │                │ │
│  │  + netns + FS ro/tmpfs     │  │  + netns + FS ro/tmpfs     │  │                │ │
│  └──────────────┬─────────────┘  └──────────────┬─────────────┘  └────────┬───────┘ │
│                 │ outbox local (SQLite/WAL efêmero por processo)          │         │
│                 ▼                                ▼                        ▼         │
│  ┌───────────────────────────────────────────────────────────────────────────────┐ │
│  │   Runtime Supervisor (.NET 10, fora deste módulo) — pool warm_size,           │ │
│  │   placement, autoscaling, cotas de pool, kill/drain de réplicas               │ │
│  └───────────────────────────────────────────────────────────────────────────────┘ │
│                                                                                     │
│  ┌────────────────────┐ ┌────────────────────┐ ┌──────────────────────────────┐   │
│  │  Redis              │ │  NATS/JetStream    │ │  OTel Collector (sidecar/    │   │
│  │  (lease de sessão,  │ │  (heartbeat,       │ │  daemonset) → 024; Serilog   │   │
│  │  exatamente-um-ativo│ │  eventos de        │ │  estruturado → Seq (024)     │   │
│  │  em resume)         │ │  execução)         │ │                              │   │
│  └────────────────────┘ └────────────────────┘ └──────────────────────────────┘   │
└───────────────────────────────────────────────────────────────────────────────────┘
```

| Contêiner/Unidade | Responsabilidade | Persistência | Escala |
|--------------------|-------------------|----------------|--------|
| Runtime Instance (processo Python por agente) | Executa `RuntimeBootstrapper`, `ExecutionStateMachine`, `CognitiveLoopEngine`, `McpHost` e clientes de recurso (§4). Efêmero. | Outbox local SQLite/WAL (apenas até publicação confirmada); nenhum estado durável local. | Horizontal por pool; ≥ 500 sessões concorrentes por nó (NFR-004). |
| Runtime Supervisor (.NET 10, fora do código deste módulo) | Cria/monitora/mata instâncias; mantém pool quente (`warm_size`); *placement*; cotas de pool. | — (control plane, `008-Agent-Lifecycle`). | N réplicas do Supervisor por cluster. |
| Redis | *Lease* distribuído por `session_id` (exatamente-um-runtime-ativo); cache de decisão do PEP local. | Volátil (reconstruível). | Cluster com réplicas. |
| NATS/JetStream | Transporte de heartbeat, eventos de ciclo de vida/execução, comandos consumidos de lifecycle/política/registro. | Streams de `020-Communication`. | Cluster NATS. |
| OTel Collector / Serilog-Seq | Coleta traces/metrics/logs de cada instância; correlação `traceparent`/`tenant`/`agent`. | Exportado a `024-Observability`. | Sidecar por nó/pool. |

---

## 4. C4 — Nível 3: Componentes

> Reproduzido e detalhado a partir de `./_DESIGN_BRIEF.md` §2.2. Este é o
> diagrama de componentes autoritativo de uma **instância** do Agent Runtime.

```
                     ┌───────────────────────────── RUNTIME SUPERVISOR (.NET 10) ─────────────────────┐
                     │      cria/monitora/mata pool · placement · cotas de pool · drain               │
                     └───────────────┬───────────────────────────────────────────────┬───────────────┘
                       gRPC (comando) │                                    NATS (heartbeat/eventos) │
   ╔═══════════════════════════════════▼════════════ AGENT RUNTIME (Python · processo por agente) ═══▼══════════╗
   ║  ┌───────────────────┐   ┌────────────────────┐   ┌──────────────────────┐                                 ║
   ║  │ SupervisorChannel │──▶│ RuntimeBootstrapper │──▶│  SandboxManager      │  seccomp·cgroups·netns·FS ro    ║
   ║  └─────────┬─────────┘   └─────────┬──────────┘   └──────────┬───────────┘                                 ║
   ║            │ heartbeat/health        │ AgentSpec, capabilities  │ isola tudo abaixo                          ║
   ║  ┌─────────▼─────────┐   ┌──────────▼───────────────────────────▼──────────────────────────────────────┐   ║
   ║  │   HealthProbe     │   │              ExecutionStateMachine (§4 do brief) — estado local canônico     │   ║
   ║  └───────────────────┘   └──────────┬───────────────────────────────────────────────┬─────────────┘   ║
   ║                                      │ conduz                                              │ transições       ║
   ║                          ┌───────────▼──────────── CognitiveLoopEngine (ReAct) ───────────▼───────────┐     ║
   ║                          │  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐  ┌──────────────────┐ │     ║
   ║                          │  │ Perception   │─▶│ Reasoning    │─▶│ ActionExecutor│─▶│ Observation      │ │     ║
   ║                          │  │ Module       │  │ Module       │  │              │  │ Collector        │ │     ║
   ║                          │  └──────┬───────┘  └──────┬───────┘  └──────┬───────┘  └────────┬─────────┘ │     ║
   ║                          │         │ percept          │ thought/act     │ tool/answer      │ observe    │     ║
   ║                          └─────────┼──────────────────┼─────────────────┼──────────────────┼───────────┘     ║
   ║        ┌─────────────────┐         │                  │                 │                  │                 ║
   ║        │ QuotaGovernor   │◀── orçamento/passos ────────┴────────┬────────┴──────────────────┘                 ║
   ║        └─────────────────┘                                      │                                             ║
   ║  ┌──────────────────┐  ┌──────────────────┐  ┌─────────────────▼──┐  ┌──────────────┐  ┌──────────────────┐   ║
   ║  │ ContextClient    │  │ MemoryClient     │  │  McpHost           │  │ PlanningClient│  │ CapabilityEnforcer│  ║
   ║  │  → 011-Context   │  │  → 010-Memory    │  │   ┌──────────────┐ │  │ → 012-Planning│  │  PEP (→022-Policy)│  ║
   ║  └──────────────────┘  └──────────────────┘  │   │ ToolInvoker  │─┼──┼─▶ 015-Tool-Mgr└──────────────────┘   ║
   ║  ┌──────────────────┐  ┌──────────────────┐  │   └──────────────┘ │  └──────────────┘                         ║
   ║  │ ModelRouterClient│  │ CheckpointManager│  └────────────────────┘                                          ║
   ║  │  → 017-Model-Rtr │  │  → MinIO/PG(cp)  │                                                                  ║
   ║  └──────────────────┘  └──────────────────┘                                                                  ║
   ║  ┌──────────────────┐  ┌──────────────────┐                                                                  ║
   ║  │ EventPublisher   │  │ TelemetryEmitter │   outbox → NATS/JetStream (020) · OTel → Collector (024)         ║
   ║  └────────┬─────────┘  └────────┬─────────┘                                                                  ║
   ╚═══════════│═════════════════════│════════════════════════════════════════════════════════════════════════════╝
              ▼                     ▼
    aios.<tenant>.agent.runtime.*   OTel Collector (traces/metrics/logs)
    aios.<tenant>.task.execution.*
```

---

## 5. Tabela de Responsabilidades por Componente

| Componente | Responsabilidade primária | Colaboradores diretos | Dependências externas |
|------------|-----------------------------|--------------------------|--------------------------|
| **RuntimeBootstrapper** | Inicializa o processo, carrega o **AgentSpec**, valida *capabilities*, prepara sandbox, emite `boot→ready`. Alvo cold start ≤ 250 ms com pool quente. | SandboxManager, CapabilityEnforcer, SupervisorChannel | Runtime Supervisor |
| **SandboxManager** | Cria/mantém isolamento: namespaces, `seccomp-bpf`, `cgroups v2`, FS efêmero, egress allowlist; mata em violação. | RuntimeBootstrapper, ExecutionStateMachine | `021-Security` (perfis), `022-Policy` (egress) |
| **ExecutionStateMachine** | Máquina de estados canônica do agente (ver `./StateMachine.md`); fonte da verdade **local** do estado; publica transições. | EventPublisher, TelemetryEmitter, CognitiveLoopEngine | — |
| **CognitiveLoopEngine** | Driver do loop ReAct: sequencia `Perception→Reasoning→Action→Observation`, controla passos/orçamento, dispara replanejamento/reflexão. | PerceptionModule, ReasoningModule, ActionExecutor, ObservationCollector, PlanningClient, QuotaGovernor | — |
| **PerceptionModule** | Monta o *percept* do passo (objetivo, contexto recuperado, observações prévias, estado do plano). | ContextClient, MemoryClient | `010-Memory`, `011-Context` |
| **ReasoningModule** | Transforma *percept* em *thought*/decisão de ação via inferência LLM; produz a próxima ação. **Sempre** via Model Router. | ModelRouterClient, ContextClient | `017-Model-Router` |
| **ActionExecutor** | Executa a ação decidida: invoca tool via MCP host, produz resposta final ou aciona replanejamento; *circuit breaker*/timeout/retry. | McpHost, ToolInvoker, CapabilityEnforcer | `015-Tool-Manager` |
| **ObservationCollector** | Normaliza o resultado da ação em um `ReActStep`, atualiza working memory, alimenta o próximo *percept*. | MemoryClient, CheckpointManager | `010-Memory` |
| **McpHost** | Hospeda o servidor/host MCP dentro do sandbox; expõe catálogo de ferramentas permitido; *handshake*/roteamento MCP. | ToolInvoker, SandboxManager | — (interno ao sandbox) |
| **ToolInvoker** | Proxy de invocação de ferramenta; toda chamada passa por `015-Tool-Manager` (autorização, métricas, versão). | McpHost, CapabilityEnforcer | `015-Tool-Manager` |
| **ModelRouterClient** | Cliente de inferência; envia prompts/mensagens ao `017-Model-Router`, recebe modelo/endpoint escolhido, streaming e fallback. | ReasoningModule | `017-Model-Router` |
| **MemoryClient** | Recupera/persiste itens de memória (working/short/long/semantic/episodic/procedural); nunca toca PostgreSQL diretamente. | PerceptionModule, ObservationCollector | `010-Memory` |
| **ContextClient** | Monta janela de contexto, aplica compressão e cache semântico. | PerceptionModule, ReasoningModule | `011-Context` |
| **PlanningClient** | Solicita plano/decomposição e replanejamento; recebe passos e critérios de sucesso. | CognitiveLoopEngine | `012-Planning` |
| **CheckpointManager** | Serializa/deserializa o **Checkpoint** (working memory, cursor do loop, plano) para MinIO/PG via control plane; habilita suspend/resume (*cold agent*). | ObservationCollector, SupervisorChannel | `010-Memory`, MinIO (via control plane) |
| **QuotaGovernor** | Contabiliza e impõe cotas locais (tokens, USD, wall-clock, passos, fan-out, profundidade); sinaliza esgotamento e aciona suspensão/abort. | CognitiveLoopEngine | `026-Cost-Optimizer` (limites), Kernel (cotas) |
| **CapabilityEnforcer** | PEP local: valida *capability tokens* concedidos no boot; consulta o PDP quando exigido. *Default deny*. | ActionExecutor, ToolInvoker, SandboxManager | `022-Policy` (via Kernel), `021-Security` |
| **EventPublisher** | Publica eventos de domínio no NATS/JetStream com envelope CloudEvents via **outbox** transacional local (SQLite/WAL) para atomicidade. | ExecutionStateMachine, QuotaGovernor, CheckpointManager, CapabilityEnforcer | NATS/JetStream (`020`) |
| **TelemetryEmitter** | Emite traces/metrics/logs OTel com correlação `traceparent`/`tenant`/`agent`; exporta ao OTel Collector. | Todos os componentes acima (transversal) | `024-Observability` |
| **SupervisorChannel** | Canal de controle com o Runtime Supervisor: recebe `boot/suspend/resume/kill/drain`, reporta health/heartbeat e estado. | RuntimeBootstrapper, HealthProbe, ExecutionStateMachine | Runtime Supervisor, `008-Agent-Lifecycle` |
| **HealthProbe** | Endpoints/handlers de liveness, readiness e *startup*; watchdog do loop (detecta *stuck* e deadlock). | ExecutionStateMachine, SupervisorChannel | — |

---

## 6. Fronteiras e Regras de Dependência

Herdadas de `001-Architecture` §6 e reproduzidas do brief §2.2 (invioláveis):

- Nenhum componente fala com LLM exceto **ModelRouterClient → 017**.
- Nenhum componente executa ferramenta externa exceto **ToolInvoker → 015**.
- Nenhum componente acessa PostgreSQL/pgvector/AGE diretamente; estado de
  controle trafega por gRPC/NATS aos serviços do control plane.
- Toda ação privilegiada passa pelo **CapabilityEnforcer** (PEP) — *default
  deny*.
- O Agent Runtime **NÃO DEVE** implementar scheduling, admissão, preempção,
  gestão de pool, treino/seleção de provedor de LLM, registro de ferramentas,
  definição de política RBAC/ABAC, event store/CQRS de leitura global,
  orquestração de sagas multi-agente, ou consolidação de aprendizado de longo
  prazo — cada uma dessas é responsabilidade de outro módulo (ver brief §1.3,
  N-01..N-10).
- O Agent Runtime **DEVE** ser tratado, do ponto de vista do Supervisor e dos
  módulos de recurso, como um **processo não-confiável por padrão**: toda
  interação de saída é mediada por um cliente com *circuit breaker*/timeout, e
  toda interação de entrada exige *capability* previamente concedida.
- Um processo de runtime **DEVE** executar **exatamente um agente por vez**
  (isolamento forte); não há multiplexação de agentes dentro do mesmo processo
  do sistema operacional.

---

## 7. Padrões Arquiteturais Adotados

| Padrão | Onde é aplicado | Motivação |
|--------|-------------------|-----------|
| **Processo isolado por agente (bulkhead de processo)** | Cada `RuntimeInstance` executa um único agente; falha/violação de um não contamina outro. | Isolamento forte análogo a processos de SO; contém *noisy neighbors* e escapes de sandbox (ADR-0070). |
| **Sandbox por syscall de SO (seccomp-bpf + cgroups v2 + namespaces)** | `SandboxManager`. | *Default deny* de rede/FS/syscalls; base de segurança do módulo (R-06, ADR-0071). |
| **PEP/PDP (Policy Enforcement/Decision Point)** | `CapabilityEnforcer` (PEP local) consulta o PDP de `022-Policy` via Kernel quando a decisão não é resolvível por *capability* cacheada. | Separação entre aplicação e decisão de política (herdado de ADR-0008); *default deny* uniforme (ADR-0078). |
| **ReAct assíncrono (`perceber→pensar→agir→observar→repetir`)** | `CognitiveLoopEngine` sobre `asyncio`. | Padrão de raciocínio consagrado (Glossário: *ReAct*); não-bloqueante para I/O de LLM/tool/memória (ADR-0077). |
| **Broker governado (nunca executor direto)** | `ToolInvoker → 015`, `ModelRouterClient → 017`, `MemoryClient → 010`, `ContextClient → 011`, `PlanningClient → 012`. | Mantém o runtime fino e substituível; cada módulo de recurso evolui independentemente (R-03/R-04/R-05). |
| **MCP host embarcado no sandbox** | `McpHost`. | Ferramentas expostas ao raciocínio sem sair do perímetro de isolamento; alternativa a *sidecar* externo avaliada e descartada (ADR-0075). |
| **Transactional Outbox (local, efêmero)** | `EventPublisher` grava em outbox SQLite/WAL local na mesma "transação" lógica que muda o estado; relay assíncrono publica no NATS. | Atomicidade estado↔evento sem acesso direto a banco durável do control plane (R-08, ADR-0076). |
| **Circuit Breaker + Retry com backoff/jitter** | `ToolInvoker`, `ModelRouterClient`, clientes de `010/011/012`. | Degradação graciosa perante falhas transitórias de dependência (R-13, FR-014). |
| **Checkpoint/hibernação (*cold agent*)** | `CheckpointManager` serializa working memory + cursor + plano; permite `suspended`/*resume*. | Base para escala a 10⁶+ agentes majoritariamente *cold* (R-09, ADR-0074). |
| **Pool quente pré-forkado (*warm pool*)** | Runtime Supervisor (fora deste módulo) mantém `warm_size` instâncias prontas; `RuntimeBootstrapper` completa o boot em ≤ 250 ms. | Meta de cold start agressiva (NFR-001, ADR-0073). |
| **Lease distribuído por sessão (Redis)** | Garantia de **exatamente um** runtime ativo por `session_id`, adquirido antes de `resume`. | Evita dupla execução de um mesmo agente em *split-brain* (ADR-0079). |
| **PEP com cache de decisão + invalidação por evento** | `CapabilityEnforcer` cacheia decisões e invalida ao consumir `aios.<t>.policy.decision.updated`. | Reduz latência do caminho quente sem sacrificar atualidade de política (ADR-0078). |

---

## 8. Tecnologias e Justificativas

| Tecnologia | Uso no Agent Runtime | Justificativa | Alternativa descartada |
|------------|-----------------------|------------------|---------------------------|
| **Python 3.12+ / asyncio** | Runtime de todos os componentes internos; loop ReAct não-bloqueante. | Ecossistema dominante para orquestração de LLM/ferramentas (MCP, SDKs de modelo); alinhado à decisão de plataforma dual .NET/Python (ADR-0003). | Node.js (ecossistema de tooling MCP menos maduro no domínio cognitivo); Go (sem paridade de bibliotecas de IA/LLM no momento da decisão). |
| **`seccomp-bpf`** | Allowlist de syscalls do processo do agente (`SandboxManager`). | Granularidade fina de restrição de syscalls do kernel do SO, nativa em Linux, baixo overhead (NFR-002: ≤ 5%). | AppArmor/SELinux isolados (menos expressivos para *allowlist* de syscall fina combinada com `cgroups`). |
| **`cgroups v2`** | Limites de CPU/RAM/PIDs por agente (`SandboxManager`, `SandboxProfile`). | Padrão de isolamento de recursos do kernel Linux; contém *noisy neighbors* (NFR-008/012). | Limites em nível de aplicação apenas (sem garantia do kernel, contorna-se por bugs de aplicação). |
| **Linux namespaces (pid/net/mnt/ipc/uts)** | Isolamento de processo/rede/FS por agente. | Base de isolamento leve, sem overhead de virtualização completa; permite `warm pool` com cold start ≤ 250 ms (NFR-001). | gVisor/Firecracker (microVM): overhead de boot maior, avaliado e descartado como padrão-default em favor de namespaces+seccomp+cgroups (ADR-0071); permanece candidato para perfis de isolamento máximo por tenant. |
| **gRPC (`aios.runtime.v1`)** | `RuntimeControl`/`AgentExecution` (Supervisor/control plane ↔ Runtime); clientes de `010/011/012/015/017`. | Contrato fortemente tipado, streaming (essencial para `SubmitTask`/`StreamSteps`), baixa latência no caminho quente. | REST interno (overhead de serialização/latência maior para o volume de chamadas do loop). |
| **NATS/JetStream** | Heartbeat, eventos de execução/ciclo de vida, comandos de lifecycle/política/registro consumidos. | Barramento único do AIOS (ADR-0004); *streams* duráveis, *replay*, baixa latência. | Kafka (operação mais pesada; NATS já padronizado). |
| **SQLite (WAL) efêmero, local ao processo** | Outbox transacional local do `EventPublisher`. | Persistência local leve e transacional sem depender de um banco durável externo dentro do sandbox (que violaria a fronteira "sem acesso direto a PostgreSQL"). | Fila em memória pura (perderia eventos em crash do processo antes do flush); acesso direto a PostgreSQL do sandbox (violaria fronteira de acesso, R-05). |
| **Redis (via control plane)** | *Lease* distribuído por sessão para *resume* exatamente-uma-vez. | Latência sub-ms, `SETNX`/TTL nativos, já padrão do AIOS para estado quente (ADR-0006). | ZooKeeper/etcd (infraestrutura adicional não padronizada no AIOS). |
| **MinIO (indireto, via control plane)** | Destino de blobs de `Checkpoint` (working memory serializada). | Object storage compatível S3, já padrão do AIOS para estado durável de agentes *cold*. | Sistema de arquivos local (não sobrevive à morte do processo/nó, incompatível com hibernação). |
| **MCP (Model Context Protocol)** | Protocolo do `McpHost` para expor ferramentas ao raciocínio. | Protocolo aberto padronizado para contexto/ferramentas de agentes (Glossário: *MCP*); interoperável com `015-Tool-Manager`. | Protocolo proprietário por ferramenta (fragmentaria a integração; perderia interoperabilidade). |
| **OpenTelemetry + Prometheus + Grafana + Serilog/Seq** | `TelemetryEmitter`: spans por passo ReAct, métricas `aios_runtime_*`, logs correlacionados. | Padrão transversal do AIOS (ADR-0010); correlação `traceparent`/`tenant`/`agent` obrigatória (RFC-0001 §5.6). | APM proprietário (vendor lock-in, sem padrão aberto). |

---

## 9. Modelo de Camadas e Travessia de Planos

```
   PLANO DE DADOS (isolado, não-confiável por padrão)        PLANO DE CONTROLE (governado)         ARMAZENAMENTO
 ┌──────────────────────────────────┐   gRPC (mediado por PEP)  ┌──────────────────────────┐   ┌───────────────────────┐
 │      007-AGENT RUNTIME             │──────────────────────────▶│ 010/011/012/015/017       │──▶│ PostgreSQL/pgvector/AGE│
 │  (loop ReAct, MCP host, sandbox)   │◀──────────────────────────│ (donos do recurso)         │◀──│ (via serviços donos)   │
 └───────────────┬────────────────────┘  resultado/observação      └──────────────────────────┘   └───────────────────────┘
                 │ gRPC (comando) / NATS (heartbeat)
                 ▼
        ┌──────────────────────────┐   consulta PDP    ┌──────────────┐
        │ Runtime Supervisor (.NET) │──────────────────▶│ 022-Policy    │
        │ 008-Agent-Lifecycle / 009-Sched.│◀──────────────────│ (via 006)     │
        └──────────────────────────┘                    └──────────────┘
                 │ outbox local → publica
                 ▼
        ┌──────────────────────────┐
        │  NATS/JetStream (020)     │──▶ 024-Observability / 025-Audit
        └──────────────────────────┘
```

O Agent Runtime é a **única travessia legítima** entre o espaço onde o agente
raciocina (não-confiável por padrão, contido por sandbox) e os módulos de
recurso do plano de controle. Assim como o Kernel é a fronteira de confiança
entre agente e recurso governado no plano de controle, o Agent Runtime é a
fronteira de confiança entre o **código/decisão do agente** (potencialmente
adversarial ou defeituoso) e **qualquer efeito real no mundo** (tool, custo,
rede, dado persistido) — daí a exigência de PEP local + sandbox + cotas como
camadas independentes e redundantes de contenção (defesa em profundidade).

---

## 10. Riscos Arquiteturais e Trade-offs

| Risco/Trade-off | Descrição | Mitigação |
|-------------------|-----------|-----------|
| Overhead de sandbox por agente | `seccomp-bpf`/`cgroups`/namespaces somam latência/CPU a cada processo. | Meta explícita de overhead ≤ 5% CPU / ≤ 8 ms por passo (NFR-002); perfis de sandbox configuráveis (`sandbox.profile`). |
| Cold start sob pool frio | Sem `warm pool`, boot completo (namespaces+seccomp+cgroups+AgentSpec) pode ultrapassar a meta agressiva. | Pool quente (`runtime.pool.warm_size`); meta dupla: p99 ≤ 250 ms com pool quente, ≤ 900 ms sem pool quente (NFR-001). |
| Outbox local efêmero pode perder eventos não publicados em crash do processo | O SQLite/WAL local não sobrevive à morte do nó/processo se o outbox ainda não tiver sido persistido no host. | `events.outbox.flush_interval_ms` curto (default 100 ms); consumidores deduplicam por `event.id`; RPO ≤ 1 passo do loop (NFR-006). |
| Dupla execução de sessão em cenários de *split-brain* do Supervisor | Dois processos poderiam, em teoria, tentar retomar a mesma sessão simultaneamente. | *Lease* distribuído (Redis) por `session_id`; `resume` só ocorre com aquisição do lease (ADR-0079, brief §10). |
| Acoplamento de latência a `015-Tool-Manager`/`017-Model-Router` | O loop ReAct é bloqueado pela resposta de tool/LLM externos ao sandbox. | *Circuit breaker*/timeout/retry; degradação graciosa (fallback de modelo, redução de contexto) (R-13, FR-014). |
| Watchdog anti-*stuck* pode interromper um passo legitimamente longo | Falso positivo do watchdog encerraria um passo válido de alta latência. | `watchdog.stuck_timeout_ms` configurável (default 120000 ms); replanejamento antes de kill definitivo (FR-016). |
| Superfície de erro ampla (15 códigos `AIOS-RUNTIME-*`) aumenta complexidade de tratamento no cliente | Consumidores do runtime precisam mapear cada código a uma ação (retry, fallback, abort). | Catálogo estável e documentado (`./API.md`); `retriable` explícito por código conforme RFC-0001 §5.4. |

---

## 11. Alternativas Descartadas

| Alternativa | Por que foi descartada |
|-------------|--------------------------|
| Threads/*workers* compartilhados executando múltiplos agentes no mesmo processo | Elimina o isolamento forte de sandbox por agente (namespaces/seccomp/cgroups não se aplicam por thread); um agente comprometido afetaria todos os demais no processo — contraria R-06/NR de isolamento; ver ADR-0070. |
| gVisor/Firecracker (microVM) como mecanismo-padrão de sandbox | Overhead de boot incompatível com a meta de cold start ≤ 250 ms (NFR-001) para o caso comum; mantido como opção para perfis de isolamento máximo por tenant, não como padrão global; ver ADR-0071. |
| MCP host como *sidecar* externo ao sandbox | Exigiria travessia de rede/IPC adicional fora do perímetro isolado para cada chamada de ferramenta, aumentando latência e superfície de ataque; embarcado no próprio sandbox preserva contenção; ver ADR-0075. |
| Acesso direto do runtime a PostgreSQL/pgvector/AGE para persistir `ReActStep`/`Checkpoint` | Violaria a fronteira de acesso do plano de dados (R-05); acoplaria o runtime a credenciais de banco dentro de um processo não-confiável por padrão; toda persistência durável passa pelos serviços donos (`010`, `025`). |
| Runtime como PDP local (decidir política sem consultar `022-Policy`) | Violaria a separação PEP/PDP herdada de ADR-0008; acoplaria evolução de política ao deploy de cada instância de runtime; ver ADR-0078. |
| Fila em memória pura (sem outbox transacional local) para eventos de execução | Perderia eventos pendentes em qualquer crash do processo antes da publicação, violando a meta de RPO ≤ 1 passo (NFR-006); substituído por outbox SQLite/WAL local (ADR-0076). |

---

## 12. Referências e ADRs

- Fonte de verdade do módulo: `./_DESIGN_BRIEF.md`
- Arquitetura global: `../001-Architecture/Architecture.md`
- Contratos centrais: `../003-RFC/RFC-0001-Architecture-Baseline.md`
- Glossário: `../040-Glossary/Glossary.md`
- ADRs globais consumidas: `../002-ADR/ADR-0001-Sistema-Operacional-nao-Framework.md`,
  `../002-ADR/ADR-0002-Microservicos-Control-Data-Plane.md`,
  `../002-ADR/ADR-0003-DotNet-Control-Python-Runtime.md`,
  `../002-ADR/ADR-0004-NATS-como-Barramento.md`,
  `../002-ADR/ADR-0006-Redis-Estado-Quente.md`,
  `../002-ADR/ADR-0008-Governanca-por-Politica-Default-Deny.md`,
  `../002-ADR/ADR-0010-Observabilidade-Auditoria-por-Construcao.md`
- ADRs do módulo (a propor, faixa `ADR-0070`–`ADR-0079`): ver `./ADR.md` para o
  índice detalhado — processo Python por agente (ADR-0070), mecanismo de
  sandbox (ADR-0071), protocolo Runtime↔Supervisor (ADR-0072), estratégia de
  cold start (ADR-0073), modelo de checkpoint/hibernação (ADR-0074), MCP host
  embarcado (ADR-0075), outbox transacional local (ADR-0076), loop ReAct
  assíncrono (ADR-0077), PEP local com cache (ADR-0078), lease distribuído por
  sessão (ADR-0079).
- RFCs do módulo (a propor): ver `./RFC.md` — RFC-0070 (Runtime↔Supervisor
  Control Protocol), RFC-0071 (MCP Host Sandboxing Profile).
- Componentes acoplados: `../006-Kernel/`, `../008-Agent-Lifecycle/`,
  `../009-Scheduler/`, `../010-Memory/`, `../011-Context/`, `../012-Planning/`,
  `../015-Tool-Manager/`, `../017-Model-Router/`, `../020-Communication/`,
  `../021-Security/`, `../022-Policy/`, `../024-Observability/`,
  `../025-Audit/`, `../026-Cost-Optimizer/`.

*Fim do documento `Architecture.md` do módulo 007-Agent-Runtime.*
