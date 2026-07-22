---
Documento: Deployment
Módulo: 007-Agent-Runtime
Status: Draft
Versão: 0.1
Última atualização: 2026-07-21
Responsável (RACI-A): Arquiteto do Módulo 007 — Agent Runtime
ADRs relacionados: ADR-0070, ADR-0071, ADR-0073
RFCs relacionados: RFC-0001
Depende de: ../008-Agent-Lifecycle/, ../020-Communication/, ../021-Security/, ../024-Observability/
---

# 007-Agent-Runtime — Deployment

> **Particularidade deste módulo.** Diferente de um serviço convencional com
> N réplicas homogêneas, o Agent Runtime é implantado como um **pool de
> processos efêmeros**, um por agente materializado, gerido pelo **Runtime
> Supervisor** (fora do código deste módulo). Este documento descreve a
> topologia de implantação da **imagem/runtime Python** que cada instância
> executa, os recursos por instância, as dependências de infraestrutura, e
> como o pool se integra ao restante do AIOS.

## Índice

1. Topologia de implantação (Docker/Compose e produção)
2. Recursos por instância (CPU/RAM/disco)
3. Dependências de infraestrutura
4. Health/Readiness/Liveness
5. Estratégia de rollout
6. Papel do YARP/API Gateway
7. Referências

---

## 1. Topologia de Implantação

### 1.1 Ambiente de desenvolvimento (Docker Compose)

```yaml
# docker-compose.dev.yml (trecho relevante ao Agent Runtime)
services:
  agent-runtime-pool:
    image: aios/agent-runtime:dev
    build:
      context: ./services/agent-runtime
      dockerfile: Dockerfile
    environment:
      AIOS_RT_RUNTIME_POOL_WARM_SIZE: "4"
      AIOS_RT_SANDBOX_PROFILE: "net-restricted"
      NATS_URL: "nats://nats:4222"
      REDIS_URL: "redis://redis:6379/2"
      OTEL_EXPORTER_OTLP_ENDPOINT: "http://otel-collector:4317"
    depends_on: [nats, redis, otel-collector, runtime-supervisor]
    cap_add: ["SYS_ADMIN"]          # necessário para namespaces/seccomp em dev
    security_opt: ["seccomp=unconfined"]  # dev apenas; produção usa perfil real
    deploy:
      replicas: 4   # simula warm_size em dev; produção usa o Supervisor, não replicas estáticas

  runtime-supervisor:
    image: aios/runtime-supervisor:dev   # .NET 10, fora deste módulo
    environment:
      AGENT_RUNTIME_IMAGE: "aios/agent-runtime:dev"
    depends_on: [nats, redis]
```

> **Nota:** em desenvolvimento, `replicas: 4` do Compose é uma
> **aproximação** do pool quente; em produção, o Runtime Supervisor cria e
> destrói processos dinamicamente (não é uma réplica estática de
> orquestrador de contêineres) conforme `runtime.pool.warm_size` e a
> demanda observada.

### 1.2 Topologia de produção (ASCII)

```
                         ┌───────────────────────────────┐
                         │   Runtime Supervisor (.NET 10)  │   N réplicas (HA)
                         │   008-Agent-Lifecycle · 009-Scheduler │
                         └───────────────┬─────────────────┘
                                          │ cria/monitora/mata processos
                    ┌─────────────────────┼─────────────────────┐
                    ▼                     ▼                     ▼
          ┌──────────────────┐  ┌──────────────────┐  ┌──────────────────┐
          │ Nó de execução A   │  │ Nó de execução B   │  │ Nó de execução C   │
          │ ┌───────────────┐ │  │ ┌───────────────┐ │  │ ┌───────────────┐ │
          │ │ Runtime proc 1 │ │  │ │ Runtime proc 1 │ │  │ │ Runtime proc 1 │ │
          │ │ (1 agente)     │ │  │ │ (1 agente)     │ │  │ │ (1 agente)     │ │
          │ ├───────────────┤ │  │ ├───────────────┤ │  │ ├───────────────┤ │
          │ │ Runtime proc 2 │ │  │ │ Runtime proc 2 │ │  │ │ Runtime proc 2 │ │
          │ │ ...            │ │  │ │ ...            │ │  │ │ ...            │ │
          │ │ até            │ │  │ │ até            │ │  │ │ até            │ │
          │ │ max_sessions_  │ │  │ │ max_sessions_  │ │  │ │ max_sessions_  │ │
          │ │ per_node       │ │  │ │ per_node       │ │  │ │ per_node       │ │
          │ └───────────────┘ │  │ └───────────────┘ │  │ └───────────────┘ │
          └─────────┬──────────┘  └─────────┬──────────┘  └─────────┬──────────┘
                    │                        │                        │
                    └────────────┬───────────┴────────────┬───────────┘
                                  ▼                        ▼
                     ┌──────────────────────┐  ┌──────────────────────┐
                     │  NATS/JetStream (020) │  │  Redis (lease/cache)  │
                     └──────────────────────┘  └──────────────────────┘
                                  │
                                  ▼
                     ┌──────────────────────┐
                     │  OTel Collector (024) │──▶ Prometheus/Grafana/Seq
                     └──────────────────────┘
```

Cada **nó de execução** hospeda múltiplos processos runtime (um por agente),
até o limite `runtime.pool.max_sessions_per_node` (default 500, NFR-004),
isolados entre si por `SandboxManager` (namespaces/`seccomp-bpf`/`cgroups v2`),
não por contêiner Docker individual por agente — a unidade de isolamento
primária é o **processo do SO**, montado sobre a infraestrutura de
contêiner/VM do nó (compatível com execução em Kubernetes com `securityContext`
apropriado, ou VMs dedicadas, conforme perfil de isolamento do tenant).

---

## 2. Recursos por Instância (CPU/RAM/Disco)

| Recurso | Default (`SandboxProfile: net-restricted`) | Faixa configurável | Observação |
|---------|----------------------------------------------|-----------------------|-------------|
| CPU | 1000 millicores (`sandbox.cpu_millicores`) | 100–8000 | Aplicado via `cgroups v2`; overhead de sandbox ≤ 5% (NFR-002). |
| RAM | 512 MiB (`sandbox.mem_mib`) | 128–8192 | Sessão *cold* (`suspended`) consome ~0 RAM (NFR-010). |
| PIDs | 256 (`sandbox.pids_max`) | 16–4096 | Limite de processos/threads dentro do sandbox (contém *fork bombs*). |
| Disco efêmero (tmpfs) | 128 MiB (`sandbox.fs_writable_mib`) | 0–4096 | Usado pelo outbox local (SQLite/WAL) e escrita temporária de ferramentas. |
| Rede (egress) | Vazio (`sandbox.egress_allowlist=[]`) | Lista `host:porta` | *Default deny*; nenhuma saída de rede sem allowlist explícita. |

**Dimensionamento de nó (referência):** um nó com 32 vCPU / 64 GiB RAM
suporta, no perfil default, até ~500 sessões `busy` simultâneas
(`max_sessions_per_node`) somando CPU/RAM configurados, mais margem para o
próprio SO/agentes de observabilidade — ver `./Scalability.md` §"Caminho de
escala" para o cálculo completo por faixa de população.

---

## 3. Dependências de Infraestrutura

| Dependência | Papel | Criticidade |
|--------------|-------|---------------|
| NATS/JetStream (`020-Communication`) | Heartbeat, eventos de execução/ciclo de vida, comandos consumidos. | **Alta** — indisponibilidade prolongada degrada observabilidade e coordenação de ciclo de vida (mitigado por outbox local, ver `./FailureRecovery.md`). |
| Redis | *Lease* distribuído por sessão; cache de decisão do PEP local. | **Alta** — sem Redis, `Resume` não pode garantir exatamente-um-ativo (ADR-0079); runtime **DEVE** recusar `Resume` em vez de arriscar dupla execução. |
| Runtime Supervisor (`.NET 10`, `008-Agent-Lifecycle`) | Cria/monitora/mata instâncias; pool quente; *placement*. | **Crítica** — sem Supervisor, não há novos `Boot`; sessões já ativas continuam até seu ciclo natural. |
| `010-Memory`, `011-Context`, `012-Planning`, `015-Tool-Manager`, `017-Model-Router` | Recursos governados consumidos pelo loop cognitivo. | **Alta por dependência específica** — cada uma é isolada por *circuit breaker*; indisponibilidade de uma não derruba as demais. |
| `022-Policy` (via Kernel) | PDP consultado pelo `CapabilityEnforcer` em cache-miss. | **Alta** — indisponibilidade aciona postura *fail-closed* (default deny) para decisões não cacheadas. |
| OTel Collector (`024-Observability`) | Exportação de traces/metrics/logs. | **Média** — degradação de observabilidade, não de execução (buffer local com *backpressure* controlado). |
| MinIO (via control plane) | Destino de blobs de `Checkpoint`. | **Alta** — sem MinIO acessível, `Suspend` não completa (retorna `AIOS-RUNTIME-0010`), e uma sessão sob pressão de cota é forçada a `terminating` em vez de `suspended`. |

---

## 4. Health/Readiness/Liveness

| Endpoint/Sinal | Semântica | Consumidor |
|------------------|-----------|--------------|
| `GET /v1/runtime/healthz` | Processo vivo (event loop `asyncio` respondendo). | Probe de infraestrutura do nó (liveness). |
| `GET /v1/runtime/readyz` | Sandbox preparado, MCP host carregado, pronto para `Boot`/`SubmitTask`. | Runtime Supervisor (decide se marca a instância `idle` no pool quente). |
| `aios.<t>.agent.runtime.heartbeat` (NATS) | Estado (`RuntimeState`), consumo, orçamento correntes. | Runtime Supervisor (watchdog de heartbeat, ver `./UseCases.md` UC-009). |
| Watchdog interno (`watchdog.stuck_timeout_ms`) | Detecta loop travado sem depender de sinal externo. | `HealthProbe` (auto-recuperação/`kill`). |

**Política de liveness/readiness em orquestradores compatíveis com
Kubernetes** (quando o nó de execução é um Pod): `livenessProbe` mapeia
`healthz` com `periodSeconds` curto (ex.: 5s) e `failureThreshold` baixo
(processo travado deve ser reciclado rápido); `readinessProbe` mapeia
`readyz` e é usado pelo Supervisor (não pelo balanceador de carga tradicional
— este módulo não recebe tráfego HTTP de usuário final).

---

## 5. Estratégia de Rollout

- **Rollout de nova versão do runtime** (imagem Python) segue o padrão
  *drain-then-replace*, coordenado pelo Runtime Supervisor:
  1. O Supervisor marca instâncias da versão antiga para `Drain`
     (`RuntimeControl.Drain`) — a instância para de aceitar novas sessões
     (`AIOS-RUNTIME-0013` para tentativas de `Boot`/`SubmitTask` recebidas
     nesse meio-tempo), mas conclui as sessões em andamento.
  2. Sessões `suspended` associadas a instâncias drenadas são naturalmente
     retomadas em instâncias novas no próximo `Resume` (sem coordenação
     especial — o checkpoint é agnóstico à versão do processo, respeitando
     compatibilidade aditiva de schema de checkpoint).
  3. Novas instâncias sobem já na versão nova, populam o pool quente
     (`runtime.pool.warm_size`), e passam a receber `Boot`.
  4. Instâncias antigas em `draining` transitam a `dead` assim que
     esvaziadas e são destruídas.
- **Rollback:** se a nova versão apresentar taxa elevada de
  `AIOS-RUNTIME-0002`/`0008`/`0015` (erros de boot/travamento/sandbox) logo
  após o rollout, o Supervisor interrompe a substituição e retoma a versão
  anterior para novas instâncias — sessões já materializadas na versão nova
  não são revertidas automaticamente (decisão operacional documentada em
  `../029-Operations/`).
- **Coexistência de versões da ABI** (`aios.runtime.v1`/`v2`) segue
  RFC-0001 §5.7 (≥ 2 majors), independentemente do rollout de imagem —
  versão de imagem e versão de ABI evoluem em ritmos possivelmente distintos.

---

## 6. Papel do YARP/API Gateway

O Agent Runtime **NÃO É** exposto pelo API Gateway externo (YARP) em nenhuma
circunstância — é um serviço estritamente interno do plano de dados,
alcançável apenas pelo Runtime Supervisor e por serviços do control plane na
rede interna (mTLS, `021-Security`). Isso reforça a Não-responsabilidade
N-08 do `_DESIGN_BRIEF.md` §1.3: "Expor API pública externa ao usuário
final" pertence a `004-API`/API Gateway, nunca a este módulo. Qualquer
necessidade de expor progresso de execução a um usuário final (ex.:
streaming de passos no WebConsole) **DEVE** passar por um serviço
intermediário do control plane que consome `StreamSteps`/eventos e os
republica sob autenticação de usuário — nunca por acesso direto ao runtime.

---

## 7. Referências

- Recursos e limites de sandbox (fonte): `./_DESIGN_BRIEF.md` §3.2 (`SandboxProfile`), §8.
- Modelo de escala e dimensionamento de nó: `./Scalability.md`.
- Segurança de rede/mTLS: `./Security.md`.
- Observabilidade de saturação de pool: `./Monitoring.md`.
- Runtime Supervisor: `../008-Agent-Lifecycle/`.
