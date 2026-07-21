---
Documento: Architecture
Módulo: 006-Kernel
Status: Draft
Versão: 0.1
Última atualização: 2026-07-20
Responsável (RACI-A): Arquiteto do Módulo 006-Kernel
ADRs relacionados: ADR-0001, ADR-0002, ADR-0003, ADR-0004, ADR-0005, ADR-0006, ADR-0008, ADR-0010 (globais); ADR-0060, ADR-0061, ADR-0062, ADR-0063, ADR-0064, ADR-0065, ADR-0066, ADR-0067, ADR-0068, ADR-0069 (deste módulo, a propor)
RFCs relacionados: RFC-0001 (baseline); RFC-0006 (Cognitive Syscall ABI, a propor); RFC-0007 (ACB & Quota Model, a propor)
Depende de: 001-Architecture, 022-Policy, 009-Scheduler, 008-Lifecycle, 020-Communication, 021-Security, 024-Observability, 025-Audit
---

# 006-Kernel — Arquitetura

> Este documento detalha a arquitetura interna do Kernel (núcleo cognitivo do
> AIOS) em notação C4 adaptada para ASCII. Ele **não redefine** contratos centrais
> (URN, envelope de evento, envelope de erro, idempotência, correlação, subjects) —
> estes são consumidos de `../003-RFC/RFC-0001-Architecture-Baseline.md`. Este
> documento é derivado de `./_DESIGN_BRIEF.md` (fonte única de verdade do módulo)
> e não pode contradizê-lo.

## Índice

1. Papel do Kernel na arquitetura global
2. C4 Nível 1 — Contexto do Sistema (Kernel)
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

## 1. Papel do Kernel na Arquitetura Global

O Kernel (`006`) é o **núcleo cognitivo** do AIOS: o análogo do kernel de um
sistema operacional clássico. Ele expõe a **ABI de syscalls cognitivas**
(`spawn`, `kill`, `suspend`, `resume`, `remember`, `recall`, `plan`,
`invoke_tool`, `route_model`, `get_quota`, `checkpoint`), mantém o **Agent
Control Block (ACB)** como estrutura de controle autoritativa de cada agente,
atua como **Policy Enforcement Point (PEP)** perante o PDP do `022-Policy`,
faz *enforcement* de **cotas de recurso** e **coordena o ciclo de vida** do
agente delegando *placement* ao `009-Scheduler` e materialização ao
`008-Lifecycle`.

O Kernel é a **fronteira de confiança** entre o plano de dados (o loop de
raciocínio do agente, executado no `007-Agent-Runtime`) e o plano de controle
governado (todos os demais módulos de recurso). Nenhum agente acessa memória,
contexto, planejamento, ferramentas ou modelos diretamente — tudo passa pelo
Kernel como **broker governado** (R-08 do brief). Isso replica, no domínio
cognitivo, a fronteira clássica *user-space ↔ kernel-space*.

---

## 2. C4 — Nível 1: Contexto do Sistema (Kernel)

```
                    gRPC (interno, mTLS)                 gRPC (interno, mTLS)
        ┌──────────────────┐                    ┌──────────────────────┐
        │ 007-Agent-Runtime│───── syscalls ────▶│                       │
        │ (plano de dados) │◀──── allow/deny ───│                       │
        └──────────────────┘                    │                       │
                                                  │                       │
        ┌──────────────────┐   REST (externo,   │                       │  gRPC   ┌─────────────────┐
        │  API Gateway      │   via YARP)         │      006-KERNEL       │────────▶│  022-Policy (PDP)│
        │  (021-Security)   │────────────────────▶│  (núcleo cognitivo)   │◀────────│                 │
        └──────────────────┘                    │                       │         └─────────────────┘
                                                  │                       │
        ┌──────────────────┐   admissão/slot     │                       │  materialização  ┌──────────────┐
        │  009-Scheduler    │◀───────────────────│                       │─────────────────▶│ 008-Lifecycle │
        │                   │────────────────────▶│                       │◀─────────────────│               │
        └──────────────────┘   decisão            │                       │  confirmação      └──────────────┘
                                                  │                       │
        ┌──────────────────┐  brokered syscalls  │                       │
        │ 010/011/012/015/  │◀───────────────────│                       │
        │ 017 (recursos)    │────────────────────▶│                       │
        └──────────────────┘  resultado           │                       │
                                                  │                       │
        ┌──────────────────┐   eventos (async)   │                       │  telemetria/audit ┌──────────────┐
        │ NATS/JetStream    │◀───────────────────│                       │─────────────────▶│ 024-Obs./     │
        │ (020-Communication│                     │                       │                   │ 025-Audit     │
        └──────────────────┘                     └───────────────────────┘                    └──────────────┘
```

**Atores e sistemas em fronteira**

| Ator/Sistema | Papel na interação com o Kernel | Protocolo |
|--------------|----------------------------------|-----------|
| `007-Agent-Runtime` | Cliente principal: todo agente invoca syscalls via seu runtime. | gRPC (interno) |
| API Gateway (YARP, `021`) | Front door externo: valida OAuth2/OIDC, propaga claims, roteia REST para o Kernel. | REST/HTTPS |
| `022-Policy` (PDP) | Autoridade de decisão consultada pelo PEP do Kernel para toda syscall privilegiada. | gRPC |
| `009-Scheduler` | Decide admissão/placement/preempção; o Kernel apenas solicita. | gRPC |
| `008-Lifecycle` | Materializa/hiberna o processo do agente e executa snapshot/restore. | gRPC |
| `010/011/012/015/017` | Módulos donos de recurso (memória, contexto, planejamento, ferramentas, modelo) para os quais o Kernel encaminha syscalls de recurso. | gRPC |
| NATS/JetStream (`020`) | Barramento de eventos assíncronos de ciclo de vida e decisão. | NATS/JetStream |
| `024-Observability` / `025-Audit` | Destino de telemetria OTel e de registros de auditoria imutáveis. | OTLP / gRPC |

---

## 3. C4 — Nível 2: Contêineres

> O Kernel é publicado como **um único serviço de control plane** (stateless,
> escalado horizontalmente), acompanhado dos armazenamentos que o suportam. Não
> há múltiplos contêineres de aplicação — a subdivisão interna é em
> **componentes** (Nível 3).

```
┌────────────────────────────────────────────────────────────────────────────────┐
│                          006-KERNEL — CONTÊINERES                              │
│                                                                                 │
│  ┌────────────────────────────────────────────────────────────────────────┐    │
│  │        Kernel Service  (.NET 10, AOT, stateless, N réplicas)          │    │
│  │  Expõe: REST /v1/kernel (via YARP) · gRPC aios.kernel.v1 (interno)    │    │
│  └───────────┬───────────────────┬───────────────────┬───────────────────┘    │
│              │                   │                   │                        │
│   ┌──────────▼─────────┐ ┌──────▼──────────┐ ┌───────▼──────────┐             │
│   │   PostgreSQL 16     │ │     Redis       │ │  NATS/JetStream   │             │
│   │  schema `kernel`    │ │  (estado quente:│ │ (Outbox → eventos  │             │
│   │  (ACB, Quota,       │ │  ACB cache,     │ │  de lifecycle/     │             │
│   │  syscall_log,       │ │  token-buckets, │ │  quota/audit)      │             │
│   │  outbox) — RLS por  │ │  decision cache │ │                    │             │
│   │  tenant_id           │ │  do PEP)        │ │                    │             │
│   └─────────────────────┘ └─────────────────┘ └────────────────────┘             │
│                                                                                 │
│   ┌─────────────────────────────────────────────────────────────────────┐     │
│   │  Sidecar de Telemetria: OTel Collector (traces/metrics) → 024;       │     │
│   │  Serilog → Seq (024/017-Logging); ambos fora do caminho crítico.     │     │
│   └─────────────────────────────────────────────────────────────────────┘     │
└────────────────────────────────────────────────────────────────────────────────┘
```

| Contêiner | Responsabilidade | Persistência | Escala |
|-----------|-------------------|---------------|--------|
| Kernel Service | Hospeda todos os componentes internos (§4); expõe REST/gRPC; stateless. | Nenhuma local (tudo externalizado). | Horizontal, por CPU/fila; sem afinidade obrigatória (§9 do brief). |
| PostgreSQL (`kernel` schema) | Fonte da verdade de ACB, Quota (limites), `syscall_log`, `outbox`. | Durável, particionado por `tenant_id`. | Réplicas de leitura + streaming replication (HA). |
| Redis | Projeção quente do ACB (leitura sub-ms), token-buckets de cota, cache de decisão do PEP, locks de FSM com TTL curto. | Volátil (reconstruível a partir do PostgreSQL). | Cluster com réplicas. |
| NATS/JetStream | Transporte durável de eventos publicados pelo Outbox relay. | Streams `KERNEL_LIFECYCLE`, `KERNEL_QUOTA`, `KERNEL_AUDIT`. | Cluster NATS (`020`). |

---

## 4. C4 — Nível 3: Componentes

> Reproduzido e detalhado a partir de `./_DESIGN_BRIEF.md` §2.2. Este é o
> diagrama de componentes autoritativo do Kernel.

```
                         REST (externo) / gRPC (interno)  · via API Gateway (YARP)
                                          │
   ┌──────────────────────────────────────▼──────────────────────────────────────┐
   │                          KERNEL SERVICE (006 · .NET 10)                     │
   │                                                                              │
   │   ┌───────────────────┐        ┌───────────────────┐                        │
   │   │  SyscallGateway    │───────▶│ IdempotencyStore  │◀── Redis + PostgreSQL  │
   │   │ (valida/despacha)  │        └───────────────────┘                        │
   │   └─────────┬──────────┘                                                     │
   │             │ (toda syscall privilegiada)                                    │
   │             ▼                                                                 │
   │   ┌───────────────────┐   consulta PDP   ┌───────────────┐                    │
   │   │ CapabilityEnforcer │────────────────▶│ PolicyClient  │──▶ 022-Policy (PDP)│
   │   │      (PEP)         │◀─── allow/deny ─│ (CB + cache)  │                    │
   │   └─────────┬──────────┘                 └───────────────┘                    │
   │             │ allow                                                           │
   │      ┌──────┴───────────────────────────┬───────────────────────────┐        │
   │      ▼                                   ▼                           ▼        │
   │ ┌─────────────────┐   ┌───────────────────────────┐   ┌───────────────────┐  │
   │ │ ResourceQuota    │   │  LifecycleCoordinator     │   │ ResourceBroker    │  │
   │ │ Manager          │◀─▶│ (FSM ACB · sagas)          │   │ Router            │  │
   │ │ (reserva/consumo)│   └───┬──────────────┬─────────┘   │ remember/recall/  │  │
   │ └───────┬──────────┘       │              │             │ plan/invoke_tool/ │  │
   │         │        ┌─────────▼──┐   ┌───────▼────────┐    │ route_model       │  │
   │         │        │SchedulerCli│   │ LifecycleCli   │    └────────┬──────────┘  │
   │         │        └─────┬──────┘   └───────┬────────┘             │             │
   │  ┌──────▼─────────┐    │                  │        ┌─────────────▼───────────┐│
   │  │  AcbStore       │    │                  │        │ CheckpointManager      ││
   │  │ (PG + Redis,    │    │                  │        └─────────────┬───────────┘│
   │  │  optimistic v)  │    │                  │                      │            │
   │  └──────┬──────────┘    │                  │                      │            │
   │         │                │                  │        ┌────────────▼───────────┐│
   │         └────────────────┴──────────────────┴───────▶│    EventEmitter        ││
   │                                                       │  (Outbox → JetStream)  ││
   │  ┌───────────────────────────────────────────────┐   └────────────┬───────────┘│
   │  │ KernelTelemetry (OTel spans/metrics/logs + audit)│               │           │
   │  └───────────────────────────────────────────────┘                │           │
   └──────────────┬───────────────┬───────────────┬──────────────────────┼───────────┘
                  ▼               ▼               ▼                       ▼
            009-Scheduler   008-Lifecycle   010/011/012/015/017   NATS aios.<tenant>.agent.*
```

---

## 5. Tabela de Responsabilidades por Componente

| Componente | Responsabilidade primária | Colaboradores diretos | Dependências externas |
|------------|---------------------------|------------------------|-------------------------|
| **SyscallGateway** | *Front door* da ABI: recebe REST/gRPC, valida schema/versão de ABI, extrai `traceparent`/`X-AIOS-Tenant`/`Idempotency-Key`, aplica rate-limit por agente, despacha ao handler correto. | CapabilityEnforcer, IdempotencyStore, ResourceBrokerRouter, LifecycleCoordinator | API Gateway (YARP) |
| **CapabilityEnforcer** | PEP: monta `DecisionRequest` (sujeito, ação, recurso, ambiente) por syscall privilegiada; aplica *default deny*; cacheia decisões (TTL curto). | PolicyClient, AcbStore | — |
| **PolicyClient** | Cliente resiliente do PDP (`022`): gRPC com circuit breaker, cache de decisões, *fail-closed* configurável, carrega *policy bundles* versionados. | CapabilityEnforcer | `022-Policy` |
| **AcbStore** | CRUD autoritativo do ACB; projeção quente (Redis) + fonte da verdade (PostgreSQL); OCC via `version`; leitura *read-your-writes*. | CapabilityEnforcer, LifecycleCoordinator, ResourceQuotaManager | PostgreSQL, Redis |
| **ResourceQuotaManager** | Reserva/consumo/liberação atômicos de cotas por agente/tenant; token-bucket/leaky-bucket; sinaliza `AIOS-QUOTA-*`. | AcbStore, ResourceBrokerRouter, LifecycleCoordinator | Redis (contadores), PostgreSQL (limites), `026` (orçamento) |
| **LifecycleCoordinator** | Implementa a FSM do ACB (§4 do brief); orquestra `spawn/suspend/resume/kill/checkpoint` como sagas com compensação. | AcbStore, SchedulerClient, LifecycleClient, EventEmitter | — |
| **SchedulerClient** | Cliente do `009-Scheduler`: submete admissão/placement, recebe slot/preempção; timeout + retry idempotente. | LifecycleCoordinator | `009-Scheduler` |
| **LifecycleClient** | Cliente do `008-Lifecycle`: solicita materialização, hibernação, snapshot/restore. | LifecycleCoordinator, CheckpointManager | `008-Lifecycle` |
| **ResourceBrokerRouter** | Encaminha `remember/recall/plan/invoke_tool/route_model` ao módulo dono após capability+cota; bulkhead e circuit breaker por dependência. | CapabilityEnforcer, ResourceQuotaManager | `010`, `011`, `012`, `015`, `017` |
| **IdempotencyStore** | Persiste resultado por `Idempotency-Key` (≥24h); deduplica repetições. | SyscallGateway | Redis (quente), PostgreSQL (durável) |
| **EventEmitter** | Publica CloudEvents no NATS/JetStream via Outbox transacional; at-least-once + ordenação por stream; dedup por `event.id`. | LifecycleCoordinator, ResourceQuotaManager, CheckpointManager, CapabilityEnforcer | PostgreSQL (outbox), NATS/JetStream |
| **CheckpointManager** | Coordena `checkpoint`: coleta ponteiros de memória/contexto, dispara snapshot durável via Lifecycle→MinIO, registra `checkpoint_ref`. | AcbStore, LifecycleClient, EventEmitter | `008-Lifecycle` → MinIO |
| **KernelTelemetry** | Instrumentação transversal OTel (spans por syscall, métricas `aios_kernel_*`, logs Serilog) e emissão de auditoria. | Todos os componentes acima (cross-cutting) | `024-Observability`, `025-Audit` |

---

## 6. Fronteiras e Regras de Dependência

- O Kernel **DEVE** tratar `009-Scheduler` e `008-Lifecycle` como **serviços
  externos consultados**, nunca como bibliotecas internas — reforça NR-02/NR-03
  do brief (o Kernel *pede*, não *decide onde/quando* nem *materializa*).
- O Kernel **NÃO DEVE** implementar lógica de negócio dos módulos de recurso
  (`010`, `011`, `012`, `015`, `017`); o `ResourceBrokerRouter` **DEVE** apenas
  aplicar capability + cota e encaminhar (NR-05, R-08).
- O Kernel **NÃO DEVE** decidir política; toda decisão de autorização **DEVE**
  atravessar `PolicyClient` até o PDP do `022-Policy` (NR-04, R-03).
- Toda dependência externa do Kernel (`022`, `009`, `008`, `010/011/012/015/017`)
  **DEVE** estar isolada por *circuit breaker* e *bulkhead* dedicados, para que a
  falha de uma dependência não degrade as demais (Padrão Bulkhead, §7).
- O Kernel **DEVE** ser *stateless* ao nível do processo: todo estado que
  sobrevive a um restart de réplica **DEVE** residir em PostgreSQL/Redis/NATS,
  nunca em memória de processo não externalizada.

---

## 7. Padrões Arquiteturais Adotados

| Padrão | Onde é aplicado | Motivação |
|--------|-------------------|-----------|
| **PEP/PDP (Policy Enforcement/Decision Point)** | `CapabilityEnforcer` (PEP) consulta `022-Policy` (PDP) via `PolicyClient`. | Separação entre aplicação e decisão de política (ADR-0008); *default deny* uniforme. |
| **Broker governado** | `ResourceBrokerRouter` encaminha syscalls de recurso sem executar a lógica do recurso. | Mantém o Kernel fino e substituível; cada módulo de recurso evolui independentemente (R-08, NR-05). |
| **Saga com compensação** | `LifecycleCoordinator` orquestra `spawn`/`kill` como sequência de passos compensáveis (reserva de slot → materialização → confirmação). | Transação distribuída sem 2PC entre Kernel, Scheduler e Lifecycle (ADR-0064). |
| **Transactional Outbox** | `EventEmitter` grava o evento na mesma transação do estado e um relay publica no JetStream. | Atomicidade entre persistência e publicação sem *dual write* (R-07, ADR-0066). |
| **Optimistic Concurrency Control (OCC)** | `AcbStore`, campo `version` do ACB. | Evita locks pessimistas no caminho quente; conflitos raros resolvidos por retry idempotente (ADR-0061). |
| **Token-bucket atômico (Lua/Redis)** | `ResourceQuotaManager`. | Enforcement de cota sob alta concorrência sem *round-trip* ao PostgreSQL a cada syscall (ADR-0062). |
| **Circuit Breaker** | `PolicyClient`, `SchedulerClient`, `LifecycleClient`, `ResourceBrokerRouter`. | Isola falha de dependência externa; evita cascata (ADR-0002, ADR global de resiliência). |
| **Bulkhead** | Pools/limites dedicados por dependência dentro do `ResourceBrokerRouter` e clientes gRPC. | Um recurso lento (ex.: `017-Model-Router`) não esgota threads/conexões usadas por outro (ex.: `010-Memory`). |
| **Idempotency-Key + deduplicação** | `IdempotencyStore` em toda mutação; `EventEmitter` deduplica por `event.id`. | Efeito *exactly-once* sobre transporte *at-least-once* (RFC-0001 §5.5). |
| **Sharding determinístico** | `AcbStore`: `shard = hash(tenant_id, agent_id) mod N`. | Localidade de cache e distribuição de carga previsível rumo a 10⁶⁺ ACBs (ADR-0065). |
| **CQRS parcial** | Leitura de ACB via Redis (projeção quente); escrita via PostgreSQL com OCC. | Latência de leitura sub-ms sem sacrificar consistência de escrita. |

---

## 8. Tecnologias e Justificativas

| Tecnologia | Uso no Kernel | Justificativa | Alternativa descartada |
|------------|-----------------|----------------|--------------------------|
| **.NET 10 (AOT)** | Runtime do Kernel Service. | Baixa latência de start, tipagem forte para a ABI de syscalls, alinhado ao control plane do AIOS (ADR-0003). | JVM (maior *footprint* de start); Go (ecossistema gRPC/OTel do .NET já padronizado no control plane). |
| **gRPC** (`aios.kernel.v1`) | Comunicação interna com `007`, `009`, `008`, `010/011/012/015/017`, `022`. | Contrato fortemente tipado, streaming, baixa latência — essencial para o caminho quente de syscalls (NFR-002/003). | REST interno (overhead de serialização/latência maior). |
| **REST (via YARP)** | Superfície externa `/v1/kernel`. | Interoperabilidade com SDKs de terceiros e ferramentas de operação; RFC-0001 §5.7 exige versionamento por caminho. | GraphQL (complexidade desnecessária para uma ABI de comandos). |
| **PostgreSQL 16** | Fonte da verdade do ACB, Quota, `syscall_log`, Outbox. | RLS nativo por `tenant_id`, transações ACID para Outbox, particionamento maduro (ADR-0005). | MongoDB (sem RLS nativo equivalente; transações multi-doc menos maduras para o padrão Outbox). |
| **Redis** | Projeção quente do ACB, token-buckets, cache de decisão do PEP, locks de FSM. | Latência sub-ms, scripts Lua atômicos para token-bucket, TTL nativo (ADR-0006). | Memcached (sem scripts atômicos, sem estruturas ricas). |
| **NATS/JetStream** | Transporte de eventos de lifecycle/quota/audit. | Baixa latência, *streams* duráveis, *replay*, alinhado ao barramento único do AIOS (ADR-0004). | Kafka (operação mais pesada para a escala inicial; NATS já é o barramento padrão do AIOS). |
| **MinIO** (via `008-Lifecycle`) | Destino de snapshots de checkpoint (indireto, via `LifecycleClient`). | Object storage compatível S3, já padrão do AIOS para estado durável de agentes *cold*. | Sistema de arquivos local (não escalável, sem durabilidade multi-nó). |
| **OpenTelemetry + Prometheus + Grafana + Serilog/Seq** | `KernelTelemetry`: spans por syscall, métricas `aios_kernel_*`, logs correlacionados. | Padrão transversal do AIOS (ADR-0010); correlação `trace_id`/`tenant_id` obrigatória (RFC-0001 §5.6). | APM proprietário (vendor lock-in, sem padrão aberto). |

---

## 9. Modelo de Camadas e Travessia de Planos

```
   PLANO DE DADOS                         PLANO DE CONTROLE                    ARMAZENAMENTO
 ┌───────────────────┐   syscall (gRPC)  ┌───────────────────────────┐   ┌─────────────────────┐
 │ 007-Agent-Runtime │──────────────────▶│         006-KERNEL         │──▶│ PostgreSQL (ACB,    │
 │  (loop cognitivo,  │◀──────────────────│  (PEP + FSM + broker de   │◀──│ Quota, log, outbox)  │
 │   sandbox Python)  │  allow/deny/result│    recurso governado)     │   └─────────────────────┘
 └───────────────────┘                    └──────────┬────────────────┘   ┌─────────────────────┐
                                                       │                    │ Redis (hot state)   │
                                                       │ syscalls de        └─────────────────────┘
                                                       │ recurso governadas ┌─────────────────────┐
                                                       ▼                    │ NATS/JetStream       │
                                     010/011/012/015/017 (donos do recurso) └─────────────────────┘
```

O Kernel é a **única travessia legítima** entre o plano de dados (onde o agente
raciocina) e o plano de controle (onde os recursos são governados). Essa
travessia única é o que torna o *capability enforcement* e o *quota
enforcement* universais e auditáveis (R-03, R-04, R-09).

---

## 10. Riscos Arquiteturais e Trade-offs

| Risco/Trade-off | Descrição | Mitigação |
|-------------------|-----------|-----------|
| Kernel como ponto único de travessia | Toda syscall passa pelo Kernel — risco de gargalo de throughput. | *Stateless* + escala horizontal + caminho quente sem I/O bloqueante (NFR-004: ≥20.000 syscalls/s por réplica). |
| Latência adicional do PEP em cada syscall privilegiada | Consulta ao PDP soma latência ao caminho crítico. | Cache de decisões com TTL curto (`kernel.pep.decision_cache_ttl_ms`); meta p99 ≤ 8 ms com cache (NFR-003). |
| Acoplamento a disponibilidade do Scheduler/Lifecycle no `spawn` | Falha externa pode travar admissão de novos agentes. | Saga com timeout + compensação (ACB→`Failed`); circuit breaker por dependência (§9 do brief). |
| Overshoot de cota sob alta concorrência | Token-bucket em Redis pode, sob corrida extrema, permitir pequeno excesso. | Reconciliação periódica com PostgreSQL; meta de overshoot ≤ 1% (NFR-009). |
| Crescimento do `syscall_log` / Outbox | Alto volume de syscalls gera grande volume de linhas. | Particionamento por tempo/tenant (ver `Database.md`), retenção configurável, *archival* para *cold storage*. |

---

## 11. Alternativas Descartadas

| Alternativa | Por que foi descartada |
|-------------|--------------------------|
| Kernel decidir política localmente (embutir PDP) | Violaria a separação PEP/PDP (ADR-0008); acoplaria evolução de política ao deploy do Kernel; ver ADR-0063. |
| Agentes acessarem `010/011/012/015/017` diretamente, sem broker | Elimina o ponto único de *capability enforcement* e cota; impossibilita auditoria centralizada (NR-05, R-08). |
| Locks pessimistas de linha para o ACB | Reduziria o throughput do caminho quente; substituído por OCC com `version` (ADR-0061). |
| 2PC (two-phase commit) entre Kernel↔Scheduler↔Lifecycle | Custo de latência e disponibilidade incompatível com NFR-001 (p99 spawn ≤ 250 ms); substituído por saga com compensação (ADR-0064). |
| Kafka como barramento de eventos do Kernel | Introduziria um segundo barramento no AIOS além do NATS já padronizado (ADR-0004), aumentando custo operacional sem ganho proporcional na escala inicial. |

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
  `../002-ADR/ADR-0005-PostgreSQL-pgvector-AGE.md`,
  `../002-ADR/ADR-0006-Redis-Estado-Quente.md`,
  `../002-ADR/ADR-0008-Governanca-por-Politica-Default-Deny.md`,
  `../002-ADR/ADR-0010-Observabilidade-Auditoria-por-Construcao.md`
- ADRs do módulo (a propor, faixa `ADR-0060`–`ADR-0069`): ver `./ADR.md` para o
  índice detalhado — ABI de syscalls (ADR-0060), modelo do ACB/OCC (ADR-0061),
  token-bucket de cota (ADR-0062), fronteira PEP/PDP (ADR-0063), saga de ciclo
  de vida (ADR-0064), sharding do ACB (ADR-0065), Outbox+JetStream (ADR-0066),
  hibernação/checkpoint (ADR-0067), brokering de recursos (ADR-0068), domínios
  de erro (ADR-0069).
- RFCs do módulo (a propor): ver `./RFC.md`.
- Componentes acoplados: `../022-Policy/`, `../009-Scheduler/`,
  `../008-Lifecycle/`, `../007-Agent-Runtime/`, `../010-Memory/`,
  `../011-Context/`, `../012-Planning/`, `../015-Tool-Manager/`,
  `../017-Model-Router/`, `../026-Cost-Optimizer/`, `../024-Observability/`,
  `../025-Audit/`, `../020-Communication/`.

*Fim do documento `Architecture.md` do módulo 006-Kernel.*
