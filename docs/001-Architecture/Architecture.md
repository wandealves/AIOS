---
Documento: Architecture (Global)
Módulo: 001-Architecture
Status: Stable
Versão: 1.0
Última atualização: 2026-07-20
Responsável (RACI-A): Arquitetura-Chefe
ADRs relacionados: ADR-0001..ADR-0010
RFCs relacionados: RFC-0001
Depende de: 000-Vision
---

# AIOS — Arquitetura Global de Referência

Este é o documento arquitetural canônico do AIOS. Ele estabelece o **modelo C4**
(Contexto → Contêiner → Componente → Código), o **modelo de deployment**, os
**padrões arquiteturais**, as **fronteiras de responsabilidade** e as **decisões
estruturais** que todos os módulos `002`–`040` reutilizam. Nenhum módulo redefine
o que aqui é estabelecido; módulos apenas *detalham* seu interior.

## Índice

1. Visão e objetivos arquiteturais
2. Atributos de qualidade e táticas
3. C4 Nível 1 — Contexto do Sistema
4. C4 Nível 2 — Contêineres
5. C4 Nível 3 — Componentes do Plano de Controle
6. Modelo de camadas (control plane / data plane / storage)
7. Fluxos ponta-a-ponta (sequência ASCII)
8. Modelo de dados de alto nível
9. Modelo de comunicação (síncrona e assíncrona)
10. Modelo de deployment
11. Multi-tenancy e isolamento
12. Escalabilidade para milhões de agentes
13. Padrões arquiteturais adotados
14. Tecnologias e justificativas
15. Fronteiras e contratos entre módulos
16. Riscos arquiteturais e trade-offs
17. Alternativas descartadas

---

## 1. Objetivos Arquiteturais

A arquitetura é dirigida por sete objetivos, em ordem de precedência:

1. **Escalabilidade elástica** — 1 → 10⁶⁺ agentes, sem reescrita.
2. **Alta disponibilidade** — control plane ≥ 99,95%; sem SPOF.
3. **Isolamento** — entre tenants, entre agentes, entre planos.
4. **Governabilidade** — política, auditoria e compliance como estrutura.
5. **Extensibilidade** — plugins/drivers/protocolos sem tocar no kernel.
6. **Observabilidade** — telemetria e proveniência em todo caminho.
7. **Eficiência de custo** — otimização explícita de custo de inferência.

---

## 2. Atributos de Qualidade e Táticas (ATAM resumido)

| Atributo | Cenário-alvo | Tática arquitetural | Onde |
|----------|--------------|---------------------|------|
| Disponibilidade | Perda de 1 nó de control plane sem downtime perceptível. | Serviços stateless replicados; estado externalizado; health/readiness; failover automático. | `027` |
| Escalabilidade | 10× de agentes com crescimento sub-linear de custo de coordenação. | Particionamento por tenant/shard; comunicação seletiva; cache semântico. | `009`,`020`,`011` |
| Desempenho | p99 de "spawn de agente" ≤ 250 ms; p99 de decisão de scheduling ≤ 20 ms. | Caminho quente sem I/O bloqueante; Redis para estado quente; NATS baixa latência. | `006`,`009` |
| Segurança | Ação privilegiada sem autorização é impossível. | *Default deny*; PEP no gateway + PDP no Policy Engine; mTLS interno. | `021`,`022` |
| Auditabilidade | Reconstruir a proveniência de qualquer decisão de agente. | Event sourcing de decisões + trilha imutável append-only. | `025` |
| Modificabilidade | Adicionar novo provedor de LLM sem deploy do kernel. | Model Router com drivers plugáveis; contrato estável. | `017` |
| Recuperabilidade | Falha de runtime não perde trabalho aceito. | Idempotência + outbox + saga + DLQ; RTO≤15min, RPO≤5min. | `014`,`027` |

---

## 3. C4 — Nível 1: Contexto do Sistema

```
                         ┌───────────────────────────────────────────┐
     ┌──────────┐        │                                           │
     │  Dev de  │──REST/ │                                           │        ┌─────────────────┐
     │ Agentes  │  SDK──▶│                                           │──────▶ │  Provedores LLM  │
     └──────────┘        │                                           │ HTTPS  │ (OpenAI, local,  │
     ┌──────────┐        │                  A I O S                  │        │  Llama, etc.)    │
     │   SRE /  │──gRPC─▶│    Artificial Intelligence Operating      │        └─────────────────┘
     │ Operador │        │              System                       │        ┌─────────────────┐
     └──────────┘        │                                           │──MCP──▶│ Ferramentas /    │
     ┌──────────┐        │   (plano de controle + plano de dados +   │        │ Sistemas Externos│
     │ Agente   │──A2A──▶│    persistência + observabilidade)        │        │ (APIs, DBs, SaaS)│
     │ Externo  │        │                                           │        └─────────────────┘
     └──────────┘        │                                           │        ┌─────────────────┐
     ┌──────────┐        │                                           │──OIDC─▶│  IdP Corporativo │
     │ Admin /  │──Web──▶│                                           │◀──────│ (Keycloak/Entra) │
     │ Console  │        └───────────────────────────────────────────┘        └─────────────────┘
     └──────────┘
```

**Atores e sistemas externos**

| Ator/Sistema | Papel | Protocolo |
|--------------|-------|-----------|
| Dev de Agentes | Cria/roda agentes, tools, memória. | REST + SDK |
| SRE/Operador | Opera, escala, observa. | gRPC + CLI + Web |
| Agente Externo | Colabora com agentes do AIOS. | A2A |
| Admin/Console | Governa, audita, configura. | Web (React) |
| Provedores LLM | Executam inferência. | HTTPS (driver Model Router) |
| Ferramentas/Sistemas | Ações no mundo (drivers). | MCP / REST / gRPC |
| IdP Corporativo | Identidade federada. | OAuth2/OIDC |

---

## 4. C4 — Nível 2: Contêineres

```
┌───────────────────────────────────────────────────────────────────────────────────────┐
│                                       AIOS                                              │
│                                                                                         │
│  ┌───────────────────────┐        ┌──────────────────────────────────────────────────┐ │
│  │  Web Console (React)  │        │            API Gateway  (YARP, .NET 10)          │ │
│  │  Container: SPA        │──HTTPS▶│  AuthN/AuthZ · rate-limit · roteamento · versão   │ │
│  └───────────────────────┘        └───────────┬──────────────────────────┬───────────┘ │
│                                      REST/gRPC │                          │ gRPC         │
│   ┌───────────────────────────────────────────▼───────────┐   ┌──────────▼───────────┐  │
│   │             CONTROL PLANE SERVICES (.NET 10)           │   │  Runtime Supervisor  │  │
│   │  ┌─────────┐ ┌──────────┐ ┌────────────┐ ┌──────────┐  │   │      (.NET 10)       │  │
│   │  │ Kernel  │ │Scheduler │ │AgentRegistry│ │  Policy  │  │   │  gerencia pool de    │  │
│   │  │ Service │ │ Service  │ │ /Lifecycle │ │ Engine   │  │   │  Agent Runtimes      │  │
│   │  └─────────┘ └──────────┘ └────────────┘ └──────────┘  │   └───────────┬──────────┘  │
│   │  ┌─────────┐ ┌──────────┐ ┌────────────┐ ┌──────────┐  │               │ gRPC/NATS    │
│   │  │ Memory  │ │ Context  │ │ ModelRouter│ │ ToolMgr  │  │   ┌───────────▼──────────┐  │
│   │  │ Service │ │ Service  │ │  Service   │ │ Service  │  │   │  AGENT RUNTIME (Python)│  │
│   │  └─────────┘ └──────────┘ └────────────┘ └──────────┘  │   │  N réplicas · sandbox  │  │
│   │  ┌─────────┐ ┌──────────┐ ┌────────────┐ ┌──────────┐  │   │  loop ReAct · MCP host │  │
│   │  │Knowledge│ │ Planning │ │  Workflow  │ │ Learning │  │   └────────────────────────┘  │
│   │  │/GraphRAG│ │ /Goals   │ │  Engine    │ │ Engine   │  │                                │
│   │  └─────────┘ └──────────┘ └────────────┘ └──────────┘  │                                │
│   │  ┌─────────┐ ┌──────────┐ ┌────────────┐               │                                │
│   │  │CostOpt. │ │Observab. │ │   Audit    │               │                                │
│   │  │ Service │ │ Collector│ │  Service   │               │                                │
│   │  └─────────┘ └──────────┘ └────────────┘               │                                │
│   └────────────────────────────┬──────────────────────────┘                                │
│                                 │ pub/sub · request/reply                                   │
│   ┌─────────────────────────────▼──────────────────────────────────────────────────────┐   │
│   │                        COMMUNICATION BUS  —  NATS (JetStream)                        │   │
│   └─────────────────────────────┬──────────────────────────────────────────────────────┘   │
│                                 │                                                            │
│   ┌───────────────┐ ┌───────────▼─────────┐ ┌───────────────┐ ┌──────────────────────────┐ │
│   │  PostgreSQL   │ │       Redis         │ │     MinIO     │ │  Observability Stack     │ │
│   │  + pgvector   │ │  cache/estado quente│ │  blobs/artefa.│ │ Prometheus·Grafana·      │ │
│   │  + Apache AGE │ │  locks/rate-limit   │ │               │ │ OTel Collector·Seq       │ │
│   └───────────────┘ └─────────────────────┘ └───────────────┘ └──────────────────────────┘ │
└───────────────────────────────────────────────────────────────────────────────────────────┘
```

**Contêineres e responsabilidades**

| Contêiner | Tecnologia | Responsabilidade | Estado |
|-----------|-----------|------------------|--------|
| API Gateway | YARP / .NET 10 | Entrada única, authN/Z, rate-limit, versão. | Stateless |
| Control Plane Services | .NET 10 | Lógica de kernel/scheduler/memória/etc. | Stateless (estado externalizado) |
| Runtime Supervisor | .NET 10 | Cria/monitora/mata Agent Runtimes; cotas. | Stateful (efêmero) |
| Agent Runtime | Python | Executa o loop cognitivo do agente em sandbox. | Efêmero por agente |
| Communication Bus | NATS + JetStream | Pub/sub, request/reply, streams duráveis. | Stateful (stream) |
| PostgreSQL | PG16 + pgvector + AGE | Registro, memória vetorial, grafo, auditoria. | Fonte da verdade |
| Redis | Redis 7 | Cache quente, locks, filas rápidas, rate-limit. | Volátil/replicado |
| MinIO | MinIO (S3) | Artefatos, checkpoints, blobs grandes. | Durável |
| Observability | OTel/Prometheus/Grafana/Seq | Métricas, traces, logs. | Time-series |

Racional das escolhas em §14 e ADRs `0001`–`0010`.

---

## 5. C4 — Nível 3: Componentes do Plano de Controle (exemplo: Kernel + Scheduler)

```
┌──────────────────────── KERNEL SERVICE (006) ─────────────────────────┐
│                                                                        │
│  ┌────────────────┐   ┌────────────────┐   ┌───────────────────────┐   │
│  │ Syscall Gateway│──▶│ Capability     │──▶│ Agent Control Block   │   │
│  │ (API cognitiva)│   │ Enforcer (PEP) │   │ Store (ACB)           │   │
│  └───────┬────────┘   └───────┬────────┘   └───────────┬───────────┘   │
│          │                    │ consulta PDP           │               │
│          │                    ▼                        │               │
│          │            ┌───────────────┐                │               │
│          │            │ Policy Client │──▶ Policy Engine(022)          │
│          │            └───────────────┘                │               │
│          ▼                                              ▼               │
│  ┌────────────────┐                          ┌───────────────────────┐ │
│  │ Resource Quota │                          │ Lifecycle Coordinator │ │
│  │ Manager        │◀────── cotas ───────────▶│ (spawn/suspend/kill)  │ │
│  └───────┬────────┘                          └───────────┬───────────┘ │
│          │ emite eventos NATS                            │ pede slot   │
└──────────┼───────────────────────────────────────────────┼────────────┘
           ▼                                                ▼
   agent.lifecycle.*                              ┌──────────────────────┐
                                                  │  SCHEDULER (009)     │
                                                  │ ┌──────────────────┐ │
                                                  │ │ Admission Ctrl   │ │
                                                  │ ├──────────────────┤ │
                                                  │ │ Priority/Cost    │ │
                                                  │ │ Policy Evaluator │ │
                                                  │ ├──────────────────┤ │
                                                  │ │ Placement Engine │ │
                                                  │ ├──────────────────┤ │
                                                  │ │ Preemption Mgr   │ │
                                                  │ └──────────────────┘ │
                                                  └──────────────────────┘
```

Cada módulo detalha seus próprios componentes internos em seu `Architecture.md`;
este diagrama fixa o **padrão** (Gateway → PEP → estado → coordenador → eventos).

---

## 6. Modelo de Camadas

```
  L4  EXPERIÊNCIA      WebConsole(032) · CLI(030) · SDK(031)
  ────────────────────────────────────────────────────────────
  L3  BORDA            API Gateway (YARP) · AuthN/Z · versão · rate-limit
  ────────────────────────────────────────────────────────────
  L2  CONTROLE         Kernel·Scheduler·Registry·Lifecycle·Policy·Memory·
       (.NET 10)       Context·ModelRouter·ToolMgr·Knowledge·Planning·Goals·
                       Workflow·Learning·CostOpt·Observability·Audit
  ────────────────────────────────────────────────────────────
  L1.5 BARRAMENTO      NATS (JetStream) — pub/sub, request/reply, streams
  ────────────────────────────────────────────────────────────
  L1  DADOS            Agent Runtime (Python) — sandbox, loop cognitivo, MCP host
  ────────────────────────────────────────────────────────────
  L0  RECURSOS         PostgreSQL(+pgvector+AGE) · Redis · MinIO · LLMs externos
  ────────────────────────────────────────────────────────────
  L-  TRANSVERSAL      Security(021) · Observability(024) · Audit(025) · Cluster/HA/DR(027)
```

**Regras de dependência (invioláveis):**
- Uma camada só depende da camada imediatamente inferior ou de serviços transversais.
- O plano de dados NUNCA acessa o banco diretamente para estado de controle; ele passa pelos serviços do plano de controle via NATS/gRPC.
- Nenhum componente fala com LLM diretamente exceto via **Model Router (017)**.
- Nenhum componente executa ferramenta externa exceto via **Tool Manager (015)** → Runtime.

---

## 7. Fluxo Ponta-a-Ponta (sequência ASCII) — "Executar uma tarefa em um agente"

```
Cliente   Gateway   Kernel   Policy   Scheduler  Registry  RuntimeSup  Runtime   ModelRouter  Memory
  │  POST    │         │        │         │          │          │         │           │          │
  │ /tasks   │         │        │         │          │          │         │           │          │
  ├─────────▶│ authN/Z │        │         │          │          │         │           │          │
  │          ├────────▶│ spawn? │         │          │          │         │           │          │
  │          │         ├───────▶│ PDP     │          │          │         │           │          │
  │          │         │◀───────┤ allow   │          │          │         │           │          │
  │          │         ├─ cotas ok ───────▶ admite   │          │         │           │          │
  │          │         │         │        ├─ place ──▶ registra │         │           │          │
  │          │         │         │        │          ├─ cria ──▶│ start   │           │          │
  │          │         │         │        │          │          ├────────▶│ boot      │          │
  │          │         │         │        │          │          │         ├─ recupera contexto ─▶│
  │          │         │         │        │          │          │         │◀── contexto ─────────┤
  │          │         │         │        │          │          │         ├─ escolhe modelo ─────▶
  │          │         │         │        │          │          │         │◀── modelo/endpoint ──┤
  │          │         │         │        │          │          │         ├─ loop ReAct (tools) ─┐
  │          │         │         │        │          │          │         │◀─────────────────────┘
  │          │         │         │        │          │          │         ├─ persiste memória ──▶│
  │◀── 202 Accepted (task_id, stream) ────────────────────────────────────┤ resultado           │
  │          │         │         │        │          │          │         │ agent.task.completed │
  └── SSE/stream de eventos ◀────── NATS ────────────────────────────────────────────────────────┘
```

Fluxo de falha (ex.: cota excedida na admissão) e compensações estão em
`../009-Scheduler/SequenceDiagrams.md` e `../006-Kernel/FailureRecovery.md`.

---

## 8. Modelo de Dados de Alto Nível

```
        ┌────────────┐        ┌────────────┐        ┌────────────┐
        │  Tenant    │1──────*│   Agent    │1──────*│  Task      │
        └────────────┘        └─────┬──────┘        └─────┬──────┘
                                    │1                    │1
                            *┌──────▼──────┐        *┌────▼──────┐
                             │ MemoryItem  │         │  Event    │ (event-sourced)
                             │ (vetor+meta)│         └───────────┘
                             └──────┬──────┘
                                    │*  arestas (Apache AGE)
                             ┌──────▼──────┐
                             │ KnowledgeNode│───(GraphRAG)
                             └─────────────┘
```

- **Tenant**: unidade de isolamento, cobrança e política.
- **Agent**: entidade gerenciada (ACB — Agent Control Block).
- **Task**: unidade de trabalho escalonável.
- **MemoryItem**: item de memória (embedding em `pgvector` + metadados de camada).
- **KnowledgeNode/Edge**: grafo de conhecimento em `Apache AGE`.
- **Event**: fonte da verdade de decisões (event sourcing) e base da auditoria.

Detalhe físico por módulo em cada `Database.md`; visão consolidada em
`../005-Database/`.

---

## 9. Modelo de Comunicação

| Padrão | Uso | Tecnologia | Garantia |
|--------|-----|------------|----------|
| Request/Reply síncrono | APIs externas, chamadas de baixa latência control→control. | gRPC (interno), REST (externo) via YARP | Timeout + retry idempotente |
| Pub/Sub assíncrono | Eventos de domínio (`agent.*`, `task.*`, `memory.*`). | NATS core | At-least-once (efêmero) |
| Stream durável | Auditoria, event sourcing, workflows, DLQ. | NATS JetStream | At-least-once + replay ordenado por stream |
| Cache/estado quente | ACB quente, rate-limit, locks distribuídos. | Redis | TTL + replicação |
| A2A (agente↔agente) | Colaboração inter-agente e com agentes externos. | A2A sobre NATS/HTTP | Negociada por capacidade |
| MCP (agente↔tool) | Invocação de ferramentas. | MCP | Definida pelo tool driver |

**Convenção de subjects NATS:** `aios.<dominio>.<entidade>.<acao>` — ex.:
`aios.agent.lifecycle.spawned`, `aios.task.execution.completed`. Versionamento de
schema por `Events.md` de cada módulo + registro central em `../004-API/Events.md`.

---

## 10. Modelo de Deployment (visão; detalhe em 028)

```
┌──────────────────────────── CLUSTER (Docker Compose → K8s no futuro) ─────────────────────────┐
│                                                                                                │
│  ZONA A (AZ-1)                         ZONA B (AZ-2)                    ZONA C (AZ-3, quorum)   │
│  ┌────────────────┐                    ┌────────────────┐               ┌──────────────────┐   │
│  │ Gateway x2     │                    │ Gateway x2     │               │ NATS quorum node │   │
│  │ ControlPlane xN│                    │ ControlPlane xN│               │ PG witness       │   │
│  │ Runtime pool   │                    │ Runtime pool   │               └──────────────────┘   │
│  │ NATS node      │                    │ NATS node      │                                      │
│  │ PG primary     │◀── streaming rep ─▶│ PG standby     │                                      │
│  │ Redis primary  │◀── replicação ────▶│ Redis replica  │                                      │
│  └────────────────┘                    └────────────────┘                                      │
│                                                                                                │
│  Balanceador L7 (YARP) → Gateways · Anti-affinity por AZ · autoscaling do Runtime pool         │
└────────────────────────────────────────────────────────────────────────────────────────────────┘
```

- **Stateless services**: escalam horizontalmente por réplica; sem afinidade.
- **Runtime pool**: autoscaling guiado pelo Scheduler/CostOptimizer (fila + custo).
- **Stateful (PG/Redis/NATS/MinIO)**: HA por replicação + quorum; ver `027`.

---

## 11. Multi-tenancy e Isolamento

Modelo: **isolamento lógico forte com opção de isolamento físico** por tenant.

| Nível | Mecanismo |
|-------|-----------|
| Dados | `tenant_id` obrigatório em toda linha; Row-Level Security no PostgreSQL; namespace por tenant no NATS (`aios.<tenant>.*`). |
| Execução | Sandbox por agente + cotas por tenant (Runtime Supervisor). |
| Rede | mTLS entre serviços; segmentação por tenant no gateway. |
| Segredos | Escopados por tenant no Security Manager (021). |
| Custo | Orçamento e cotas por tenant no Cost Optimizer (026). |
| Físico (opcional) | Pools de runtime dedicados / bancos dedicados para tenants sensíveis. |

---

## 12. Escalabilidade para Milhões de Agentes

Estratégia central: **agentes efêmeros e "cold-storable"** + **particionamento** +
**comunicação seletiva**.

```
  1 → 10^3 agentes     : nó único / Compose; runtime pool pequeno.
  10^3 → 10^5 agentes   : múltiplos runtime pools; sharding de memória por tenant;
                          NATS cluster; PG com read-replicas + partições.
  10^5 → 10^6+ agentes  : agentes majoritariamente suspensos ("cold"), materializados
                          sob demanda (spawn < 250ms); sharding por (tenant, hash(agent));
                          Scheduler distribuído; Cost Optimizer prioriza modelos menores.
```

Táticas-chave (detalhe em `*/Scalability.md`):
- **Suspensão/hibernação de agentes** — estado em MinIO/PG; RAM só para agentes ativos.
- **Sharding determinístico** — `shard = hash(tenant_id, agent_id) mod N`.
- **Comunicação seletiva** — limites de fan-out; agrupamento (agent groups); evita O(N²).
- **Cache semântico** — Context/Memory reduzem chamadas repetidas a LLM.
- **Backpressure** — JetStream + limites de admissão no Scheduler.

Limites teóricos e filas em `../009-Scheduler/Scalability.md` e `../027-Cluster/`.

---

## 13. Padrões Arquiteturais Adotados

| Padrão | Onde | Motivo |
|--------|------|--------|
| Control Plane / Data Plane split | global | Isolar falhas; escalar independentemente. |
| Microserviços orientados a domínio | control plane | Modificabilidade e deploy independente. |
| Event Sourcing (decisões) | Kernel, Scheduler, Audit | Proveniência e auditoria imutável. |
| CQRS (leitura/escrita separadas) | Registry, Memory, Knowledge | Escala de leitura; projeções otimizadas. |
| Saga / compensação | Workflow, Task | Consistência sem 2PC distribuído. |
| Outbox transacional | todo serviço que publica evento | Atomicidade DB+evento. |
| Sidecar/Driver | ToolMgr, ModelRouter | Extensibilidade sem tocar no core. |
| PEP/PDP (policy) | Gateway + Policy Engine | *Default deny* centralizado. |
| Bulkhead + Circuit Breaker | chamadas a LLM/tools | Contenção de falhas. |
| Ambassador (Gateway) | YARP | Entrada única, versionamento. |

---

## 14. Tecnologias e Justificativas

| Tecnologia | Papel | Justificativa (resumo; ADR) |
|------------|-------|-----------------------------|
| **.NET 10** | Plano de controle | Alto desempenho, tipagem forte, async maduro, AOT, ecossistema Enterprise. (ADR-0003) |
| **Python** | Agent Runtime | Ecossistema de IA/ML e MCP mais rico; isolado no plano de dados. (ADR-0003) |
| **React** | Web Console | Padrão de mercado para SPA operacional. (032) |
| **NATS + JetStream** | Barramento | Baixa latência, footprint pequeno, request/reply nativo, streams duráveis. (ADR-0004) |
| **PostgreSQL** | Fonte da verdade | ACID, maturidade, extensões. (ADR-0005) |
| **pgvector** | Memória vetorial | Busca ANN no mesmo DB; menos peças móveis. (ADR-0005) |
| **Apache AGE** | Grafo de conhecimento | openCypher sobre PostgreSQL; unifica relacional+grafo. (ADR-0005) |
| **Redis** | Cache/estado quente/locks | Latência sub-ms; estruturas ricas. (ADR-0006) |
| **MinIO** | Object storage | S3-compatível on-prem; blobs/checkpoints. |
| **YARP** | API Gateway | Reverse proxy .NET nativo, extensível. |
| **OpenTelemetry** | Telemetria | Padrão vendor-neutral de traces/metrics/logs. |
| **Prometheus/Grafana** | Métricas/dashboards | Padrão de mercado; alerting. |
| **Serilog/Seq** | Logs estruturados | Ecossistema .NET; consulta estruturada. |
| **Docker/Compose** | Empacotamento/dev | Reprodutibilidade; caminho para K8s. |
| **gRPC/REST/MCP/A2A** | Contratos | Interno perf (gRPC), externo compat (REST), tools (MCP), agentes (A2A). |

---

## 15. Fronteiras e Contratos entre Módulos

Regra: **módulos se comunicam apenas por contratos publicados** (API/Events),
nunca por acesso direto ao estado interno de outro módulo.

```
  Kernel(006) ── syscalls ─▶ todos          ModelRouter(017) ── única via p/ LLM
  Scheduler(009) ── decisões ─▶ RuntimeSup   ToolMgr(015) ── única via p/ tools
  Policy(022) ── PDP ─▶ todo PEP             Memory(010)/Context(011) ─▶ Runtime
  Audit(025) ── consome eventos de todos     Observability(024) ── transversal
```

Matriz completa de dependências permitidas: `../001-Architecture/DependencyMatrix.md`
(a ser gerado na fase de fan-out). Violações são detectadas por lint arquitetural
(`033-DeveloperGuide/ArchLint.md`).

---

## 16. Riscos Arquiteturais e Trade-offs

| Risco/Trade-off | Descrição | Decisão |
|-----------------|-----------|---------|
| Complexidade de microserviços vs. monólito | Mais peças móveis. | Aceito: isolamento e escala independentes valem a complexidade; mitigado por operadores e defaults (ADR-0002). |
| Consistência eventual (event-driven) | Leituras podem ver estado atrasado. | Aceito: CQRS + versionamento; leituras críticas via read-your-writes no Redis. |
| Dois runtimes de linguagem (.NET/Python) | Custo cognitivo/operacional. | Aceito: fronteira clara control/data plane; contratos gRPC/NATS (ADR-0003). |
| pgvector vs. vector DB dedicado | Escala de bilhões de vetores. | Aceito p/ v1; caminho de migração para índice externo previsto (ADR-0005). |
| NATS vs. Kafka | Menos garantias de retenção longa que Kafka. | Aceito: JetStream cobre casos; Kafka opcional para audit sink (ADR-0004). |

---

## 17. Alternativas Descartadas

| Alternativa | Motivo da recusa | ADR |
|-------------|------------------|-----|
| Monólito modular único | Não escala independentemente; falha acoplada. | ADR-0002 |
| Kubernetes como runtime de agente (1 pod/agente) | Granularidade/custo inviáveis para 10⁶ agentes efêmeros. | ADR-0002 |
| Núcleo em Python | GIL/perf/tipagem inferiores para control plane crítico. | ADR-0003 |
| Kafka como bus primário | Latência/operabilidade; request/reply não nativo. | ADR-0004 |
| Vector DB dedicado (Milvus/Weaviate) como padrão | Mais peças móveis no MVP; pgvector suficiente p/ v1. | ADR-0005 |
| Sem plano de controle (biblioteca) | Sem governança/HA/cotas — contraria a Visão. | ADR-0001 |

---

## 18. Referências

- Visão: `../000-Vision/Vision.md`
- Decisões: `../002-ADR/` (ADR-0001..0010)
- Propostas: `../003-RFC/RFC-0001-Architecture-Baseline.md`
- Contratos: `../004-API/`
- Banco: `../005-Database/`
- Glossário: `../040-Glossary/Glossary.md`

*Próximo na cadeia canônica:* especificações de núcleo — `../006-Kernel/`,
`../009-Scheduler/`, `../007-Agent-Runtime/`, `../010-Memory/`, `../011-Context/`.
