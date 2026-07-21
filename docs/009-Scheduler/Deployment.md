---
Documento: Deployment
Módulo: 009-Scheduler
Status: Draft
Versão: 0.1
Última atualização: 2026-07-20
Responsável (RACI-A): Arquiteto do Módulo 009-Scheduler
ADRs relacionados: ADR-0090 (Admission control com reserva atômica), ADR-0092 (Sharding determinístico), ADR-0093 (Filas ZSET com leases), ADR-0097 (Outbox transacional); ADR-0002 (Microsserviços Control/Data Plane), ADR-0003 (.NET Control / Python Runtime), ADR-0004 (NATS como Barramento), ADR-0006 (Redis Estado Quente) — herdadas
RFCs relacionados: RFC-0001 (baseline)
Depende de: 001-Architecture, 006-Kernel, 007-Agent-Runtime, 022-Policy, 026-Cost-Optimizer, 027-Cluster, 028-Deployment
---

# 009-Scheduler — Deployment

> **Escopo.** Este documento especifica como o Scheduler Service (009) é
> empacotado, implantado, escalado e operado: topologia de containers,
> recursos por réplica, dependências externas, health/readiness, estratégia de
> rollout e integração com o Gateway (YARP). Não redefine os contratos centrais
> de RFC-0001 nem os modos de falha (ver `FailureRecovery.md`) ou o modelo de
> escala/sharding (ver `Scalability.md`) — este documento foca em **como o
> serviço é colocado em produção**, referenciando aqueles onde necessário.

---

## Índice

1. Visão Geral da Topologia
2. Requisitos de Infraestrutura por Réplica
3. Containerização
4. Topologia de Implantação — Docker Compose
5. Réplicas e Ownership de Shard
6. Dependências Externas
7. Health e Readiness
8. Estratégia de Rollout
9. Escalonamento
10. Configuração de Rede e Gateway (YARP)
11. Ambientes
12. Migrações de Banco de Dados
13. Referências

---

## 1. Visão Geral da Topologia

O Scheduler Service é um processo **stateless** (.NET 10), implantado como
múltiplas réplicas idênticas, acessível por gRPC interno (Kernel 006, Runtime
Supervisor 007) e por REST administrativo via Gateway (YARP). Todo estado
autoritativo é externalizado em PostgreSQL (`DecisionJournal`,
`IdempotencyRecord` — fonte da verdade) e Redis (`PriorityQueueStore`,
`QuotaLedger`, leases — estado quente), e toda comunicação assíncrona passa
pelo NATS/JetStream.

```
                                   Rede interna do Control Plane
                                             │
              gRPC + mTLS (interno, sem Gateway externo)
                                             │
                    ┌────────────────────────┼─────────────────────────┐
                    ▼                        ▼                         ▼
           ┌─────────────────┐      ┌─────────────────┐       ┌─────────────────┐
           │ Kernel (006)     │      │ Runtime Supervisor│      │  Web Console/CLI │
           │ "pede slot"      │      │      (007)         │      │  (032/030)        │
           └────────┬─────────┘      └─────────┬─────────┘      └────────┬─────────┘
                    │ Submit                    │ ReportRuntimeSignal      │ REST via Gateway (YARP)
                    ▼                           ▲                          ▼
           ┌──────────────────────────────────────────────────────────────────────┐
           │                     SCHEDULER SERVICE — pool de réplicas              │
           │   ┌─────────────────┐  ┌─────────────────┐        ┌─────────────────┐│
           │   │ réplica 1        │  │ réplica 2        │  ...  │ réplica N        ││
           │   │ owner shards     │  │ owner shards     │       │ owner shards     ││
           │   │ 0..k (lease)     │  │ k+1..2k (lease)  │       │ ...              ││
           │   └────────┬─────────┘  └────────┬─────────┘       └────────┬─────────┘│
           └────────────┼──────────────────────┼──────────────────────────┼─────────┘
                         │                      │                          │
        ┌────────────────┼──────────────────────┼──────────────────────────┼───────────┐
        ▼                ▼                      ▼                          ▼           ▼
  PostgreSQL HA     Redis Cluster           NATS/JetStream Cluster    022-Policy   026-Cost-Optimizer
 (DecisionJournal,  (ZSET filas, cotas,     (task.execution.*,          (PDP)        (orçamento)
  Idempotency, RLS)  leases de shard/       scheduler.*)
                      dispatch)                                            │              │
                                                                            ▼              ▼
                                                                       gRPC+mTLS      gRPC+mTLS
                                                                                            │
                                                                                            ▼
                                                                                    027-Cluster
                                                                                (inventário/capacidade)
```

Todas as réplicas do Scheduler são **funcionalmente equivalentes**; a
diferença entre elas é apenas a **posse (ownership) de um subconjunto de
shards** via lease em Redis — uma otimização de localidade de dispatch, não um
requisito de correção (ver `Scalability.md` §3 e `_DESIGN_BRIEF.md` §10).

---

## 2. Requisitos de Infraestrutura por Réplica

| Recurso | Valor recomendado (baseline) | Observação |
|---------|-------------------------------|-------------|
| CPU | 2 vCPU (request), 4 vCPU (limit) | Caminho quente (`Submit`) é CPU-bound: serialização, validação de envelope, cálculo de `C = α·L + β·$ + γ·E`, cache de decisão PDP. |
| Memória | 1 GiB (request), 2 GiB (limit) | Filas e cotas residem em Redis, não em heap do Scheduler; heap cobre conexões, buffers, cache local de decisão e de capacidade. |
| GPU | Não aplicável | O Scheduler não executa inferência; roteamento de modelo é delegado a `017-Model-Router`. |
| Disco | Efêmero, ≤ 1 GiB | Sem estado persistente local; usado apenas para logs de curto prazo antes do envio a Seq/OTel collector. |
| Rede | Baixa latência (mesma região/AZ que Redis/PostgreSQL/NATS) | Latência de rede às dependências é o principal risco ao SLO de `Submit` p99 ≤ 20 ms (NFR-001). |

> Estes valores são **baseline de dimensionamento inicial**; o dimensionamento
> definitivo por ambiente é validado em `Benchmark.md` e ajustado por
> autoescalonamento (§9).

---

## 3. Containerização

| Aspecto | Especificação |
|---------|-----------------|
| Runtime | .NET 10, imagem de execução mínima (distroless-equivalente). |
| Compilação | AOT (*Ahead-of-Time*) quando suportado pela plataforma alvo, para reduzir tempo de start-up e superfície de runtime (ver Glossário: AOT). |
| Usuário | Não-root, UID/GID dedicados, sem capacidades Linux adicionais. |
| Sistema de arquivos | Raiz somente-leitura; `/tmp` efêmero para buffers de curto prazo. |
| Variáveis de ambiente | Prefixo `AIOS_SCHEDULER_` para todas as chaves de configuração (ver `_DESIGN_BRIEF.md` §8); segredos **NÃO DEVEM** ser passados como variável de ambiente em texto puro — usar montagem de arquivo a partir do cofre de segredos (`Security.md` §6). |
| Portas expostas | `5001` (gRPC interno `aios.scheduler.v1`, mTLS), `8080` (REST admin, atrás do Gateway), `9090` (métricas Prometheus, rede interna de observabilidade). |
| Tags de imagem | Semânticas (`009-scheduler:1.2.0`), imutáveis; **NÃO DEVEM** ser sobrescritas (`:latest` proibido em produção). |

---

## 4. Topologia de Implantação — Docker Compose (ambiente de desenvolvimento/integração)

```yaml
services:
  scheduler:
    image: registry.internal/aios/009-scheduler:${SCHEDULER_VERSION}
    deploy:
      replicas: 3
      resources:
        limits: { cpus: "4", memory: 2g }
        reservations: { cpus: "2", memory: 1g }
      restart_policy: { condition: on-failure, max_attempts: 5 }
    environment:
      - AIOS_SCHEDULER_SHARDS_COUNT=64
      - AIOS_SCHEDULER_WEIGHTS_ALPHA=0.5
      - AIOS_SCHEDULER_WEIGHTS_BETA=0.3
      - AIOS_SCHEDULER_WEIGHTS_GAMMA=0.2
      - AIOS_SCHEDULER_BACKPRESSURE_DEFER_THRESHOLD=0.80
      - AIOS_SCHEDULER_BACKPRESSURE_REJECT_THRESHOLD=0.95
      - AIOS_SCHEDULER_POLICY_ENGINE=heuristic
    secrets:
      - scheduler_pg_credentials
      - scheduler_redis_credentials
      - scheduler_nats_account_jwt
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8080/healthz"]
      interval: 10s
      timeout: 3s
      retries: 3
      start_period: 15s
    depends_on:
      postgres: { condition: service_healthy }
      redis: { condition: service_healthy }
      nats: { condition: service_healthy }
    networks: [aios-control-plane]

  postgres:
    image: pgvector/pgvector:pg16
    environment:
      - POSTGRES_DB=aios_scheduler
    volumes: [pg-scheduler-data:/var/lib/postgresql/data]
    networks: [aios-control-plane]

  redis:
    image: redis:7-alpine
    command: ["redis-server", "--requirepass", "${REDIS_PASSWORD}"]
    networks: [aios-control-plane]

  nats:
    image: nats:2-alpine
    command: ["-js", "-sd", "/data"]
    volumes: [nats-scheduler-data:/data]
    networks: [aios-control-plane]

networks:
  aios-control-plane: { driver: bridge }

volumes:
  pg-scheduler-data:
  nats-scheduler-data:

secrets:
  scheduler_pg_credentials: { external: true }
  scheduler_redis_credentials: { external: true }
  scheduler_nats_account_jwt: { external: true }
```

> Em ambientes de produção, PostgreSQL, Redis e NATS **DEVEM** ser clusters
> gerenciados de alta disponibilidade compartilhados/dedicados (ver
> `027-Cluster`), não containers efêmeros do Compose — o exemplo acima é para
> desenvolvimento e testes de integração locais.

---

## 5. Réplicas e Ownership de Shard

| Aspecto | Regra |
|---------|-------|
| Contagem mínima de réplicas em produção | **3** (tolerância a falha de 1 zona sem perda de capacidade de atendimento). |
| Escalonamento horizontal | Baseado em CPU (%), profundidade agregada de fila e `aios_scheduler_decision_latency_ms` p99 (ver §9). |
| Ownership de shard | Cada réplica assume, via **lease em Redis com TTL** (`scheduler.lease.ttl_ms`), a posse exclusiva de um subconjunto de shards `0..N-1` para fins de `DispatchLoop` — isso evita contenção de dispatch (duas réplicas tentando retirar o mesmo topo de fila) sem exigir lock global. |
| Correção sob perda de réplica | Se uma réplica falha, seu lease de shard expira e **outra réplica assume automaticamente** o shard órfão — nenhuma tarefa fica presa a uma réplica específica; qualquer réplica **PODE** atender qualquer tenant/shard se necessário (ownership é otimização, não requisito de correção). |
| Distribuição entre zonas | Réplicas **DEVEM** ser distribuídas em ≥ 2 zonas de disponibilidade (AZ) quando a infraestrutura subjacente suportar, para atender a NFR-003 (disponibilidade ≥ 99,95%). |
| Rebalanceamento de shards (mudança de `N`) | `scheduler.shards.count` **NÃO É** recarregável isoladamente em runtime — mudança de `N` exige rebalanceamento coordenado com `027-Cluster` (hashing consistente, ADR-0092), tipicamente executado como uma operação de manutenção planejada (ver `Scalability.md` §9). |

```
   requests ─▶ [réplica A owner shards 0..31] ─▶ ZSET sched:{t}:{c}:{0..31} ─▶ dispatch
   requests ─▶ [réplica B owner shards 32..63]─▶ ZSET sched:{t}:{c}:{32..63}─▶ dispatch
                         ▲ ownership lease (Redis, TTL) ▲   backpressure feedback ◀── JetStream lag
```

---

## 6. Dependências Externas

| Dependência | Protocolo | Criticidade | Comportamento sob indisponibilidade |
|--------------|-----------|--------------|----------------------------------------|
| Redis (`PriorityQueueStore`, `QuotaLedger`, leases) | TLS + AUTH | **Crítica** | Scheduler entra em modo conservador: novas admissões rejeitadas (`AIOS-SCHED-0003`, retriável); nenhuma reserva/dispatch prossegue sem garantia atômica (ver `FailureRecovery.md`). |
| PostgreSQL (`DecisionJournal`, `IdempotencyRecord`) | TLS + client cert | **Crítica** | Decisões não podem ser persistidas de forma durável; `Submit` degrada para `503`/`AIOS-SCHED-0012`; eventos não são publicados sem o registro correspondente (outbox). |
| NATS/JetStream (publicação de eventos, consumo de sinais) | TLS | **Alta** | Eventos ficam retidos no outbox (`DecisionJournal`) até reconexão; decisões continuam sendo aceitas e persistidas (não bloqueante para o caminho de admissão). |
| `022-Policy` (PDP) | gRPC + mTLS | **Crítica** para admissão/preempção | *Fail-closed*: operações privilegiadas negadas (`AIOS-SCHED-0004`), ver `Security.md` §3.3. |
| `026-Cost-Optimizer` | gRPC + mTLS | **Alta** | Fallback a última estimativa cacheada + margem conservadora; apenas orçamentos no limite são rejeitados (`AIOS-SCHED-0002`). |
| `027-Cluster` (inventário de capacidade) | gRPC/eventos + mTLS | **Alta** | Snapshot expirado é tratado como capacidade zero para o shard afetado (fail-safe); placement para aquele shard falha com `AIOS-SCHED-0010`. |
| `007-Agent-Runtime` (Runtime Supervisor) | gRPC + mTLS | **Alta** (bloqueia confirmação de dispatch) | `ReconciliationWorker` expira lease de dispatch órfão e reenfileira; tarefa não fica presa em `SCHEDULED` indefinidamente. |
| Gateway (YARP) | HTTPS/mTLS | **Crítica** para tráfego administrativo externo | Sem Gateway, tráfego REST administrativo não alcança o Scheduler; tráfego gRPC interno direto (Kernel/Runtime Supervisor) não é afetado. |
| `025-Audit` | eventos | **Alta** (não bloqueante) | Auditoria é assíncrona via evento; indisponibilidade de `025` não bloqueia a decisão, mas **DEVE** gerar alerta (gap de auditoria é risco de conformidade). |

---

## 7. Health e Readiness

| Endpoint | Verifica | Uso |
|----------|----------|-----|
| `GET /healthz` | Processo vivo (liveness): `DispatchLoop` e loop de eventos responsivos, sem deadlock. **NÃO** verifica dependências externas. | Reinício automático do container se falhar repetidamente. |
| `GET /readyz` | Prontidão (readiness): conectividade com Redis (ping), PostgreSQL (ping), NATS (conexão ativa), e circuito do `PolicyClient` não aberto de forma permanente. | Remoção temporária do pool de réplicas do Gateway/roteamento gRPC se falhar — réplica não recebe novo tráfego, mas não é reiniciada. |
| `GET /metrics` | Exposição Prometheus (`aios_scheduler_*`, ver `Metrics.md`). | Scrape periódico pela pilha de observabilidade (`024-Observability`). |

**Regras normativas:**

- `readyz` **DEVE** retornar `503` se qualquer dependência crítica (Redis,
  PostgreSQL) estiver inacessível, mesmo que `healthz` continue `200` — o
  processo está vivo, mas não deve receber tráfego de admissão.
- `readyz` **NÃO DEVE** depender do PDP (`022-Policy`), do Cost-Optimizer
  (`026`) ou do Cluster (`027`) estarem disponíveis — a indisponibilidade
  dessas dependências é tratada por *fail-safe* por requisição (`Security.md`
  §3.3, `FailureRecovery.md`), não por remoção da réplica do pool (leituras
  não privilegiadas, ex. `GetDecision`, devem continuar funcionando).
- Tempo de `start_period` do healthcheck **DEVE** acomodar o *warm-up* de
  pools de conexão (Redis/PostgreSQL/NATS), aquisição inicial de lease de
  shard e carregamento de cache de capacidade, tipicamente 10–20 s.

---

## 8. Estratégia de Rollout

| Estratégia | Aplicação | Critério de avanço/rollback |
|------------|-----------|--------------------------------|
| **Rolling update** (default) | Atualizações de patch/minor sem mudança de schema de API ou de contrato de fila. | Substituição gradual de réplicas (uma por vez ou por *batch* de 25%), aguardando `readyz=200` e reaquisição de lease de shard antes de prosseguir; rollback automático se taxa de erro (`5xx`/`AIOS-SCHED-0012`) ultrapassar limiar durante o rollout. |
| **Blue-green** | Mudanças de *major* de API (nova versão de `aios.scheduler.v2`) ou migração de schema de `DecisionJournal` não aditiva. | Nova geração de réplicas (`v2`) sobe em paralelo à atual (`v1`); tráfego migra via Gateway/roteamento gRPC após validação de *smoke tests*; `v1` mantida por período de coexistência (RFC-0001 §5.7: ≥ 2 majors). |
| **Canary** | Mudanças de alto risco em `ISchedulingPolicy` (ex.: troca `heuristic`→`rl`, ADR-0096), pesos de score, ou lógica de `PreemptionManager`. | Pequena fração de tráfego (ex.: 5% dos tenants, ou um tenant piloto) roteada à nova política via `scheduler.policy.engine` por tenant; métricas de fairness/latência/custo comparadas à baseline antes de expandir. |

**Regras de compatibilidade:**

- Migrações de schema PostgreSQL **DEVEM** ser aditivas e retrocompatíveis
  durante o rollout (coluna nova nullable, sem `DROP`/`RENAME` destrutivo na
  mesma release) — ver `Database.md` para a estratégia de migração completa.
- Mudança incompatível de API (REST/gRPC) **DEVE** incrementar a versão major
  (`/v2/scheduler`, pacote `aios.scheduler.v2`) e coexistir com a versão
  anterior por ≥ 2 majors (RFC-0001 §5.7), nunca substituição *in place*.
- Troca de `ISchedulingPolicy` (v1↔v2) **NÃO DEVE** exigir mudança de contrato
  de API/eventos (FR-013, ADR-0096) — é uma mudança de configuração
  (`scheduler.policy.engine`), habilitando rollout independente do rollout de
  código.
- Nenhuma release **DEVE** ser promovida a 100% do tráfego sem que os SLOs de
  `NonFunctionalRequirements.md` (latência, fairness, taxa de erro)
  permaneçam dentro da meta durante a janela de canary/blue-green.

---

## 9. Escalonamento

| Sinal | Ação | Referência |
|-------|------|--------------|
| CPU média de réplica > 70% por 5 min | Adicionar réplica. | Autoescalonamento horizontal padrão. |
| `aios_scheduler_decision_latency_ms{quantile="0.99"}` acima da meta (NFR-001) | Adicionar réplica; investigar dependência lenta (PDP/Cost-Optimizer/Redis). | `Scalability.md` §8 (limites teóricos). |
| Profundidade de fila (ZSET) crescente por shard sem dispatch correspondente | Investigar réplica *owner* daquele shard (lease travado/reiniciando); considerar redistribuir ownership. | `FailureRecovery.md` (lease órfão). |
| Backlog do outbox (`DecisionJournal` com `published=false` crescente) | Aumentar paralelismo do relay de publicação ou investigar partição do NATS. | `FailureRecovery.md` (partição do NATS). |
| CPU média de réplica < 30% por 15 min | Remover réplica (respeitando mínimo de 3 em produção). | Autoescalonamento horizontal padrão. |

O modelo detalhado de escala (sharding, ownership, limites teóricos de
decisões/s e caminho para 10⁶⁺ agentes) é especificado em `./Scalability.md`
— este documento cobre apenas os gatilhos operacionais de infraestrutura.

---

## 10. Configuração de Rede e Gateway (YARP)

| Aspecto | Especificação |
|---------|-----------------|
| Roteamento | YARP roteia `/v1/scheduler/*` (REST administrativo) ao pool de réplicas do Scheduler via *round-robin* com *health-aware* (remove réplicas com `readyz` falho). |
| Tráfego gRPC interno | Não passa pelo YARP externo; Kernel (006) e Runtime Supervisor (007) chamam diretamente via mTLS ponto-a-ponto ou *service mesh sidecar* (conforme `028`). |
| Rate-limit de borda | O Gateway aplica um rate-limit *coarse* por tenant/IP como primeira linha de defesa para tráfego REST administrativo; o rate-limit fino de admissão (`QuotaLedger`, `BackpressureController`) é responsabilidade do Scheduler — ambos coexistem em camadas (*defense in depth*). |
| Terminação TLS | TLS externo terminado no Gateway para tráfego REST; tráfego Gateway→Scheduler e gRPC interno direto são mTLS (não HTTP puro). |
| Cabeçalhos propagados | `traceparent`, `X-AIOS-Tenant`, `X-AIOS-Agent`, `Idempotency-Key`, `X-AIOS-Api-Version` (RFC-0001 §5.6) — o Gateway **DEVE** preservar/gerar esses cabeçalhos antes de encaminhar ao Scheduler. |
| `Retry-After` | Respostas `AIOS-SCHED-0003` (backpressure) **DEVEM** propagar `retryAfterMs`/`Retry-After` de ponta a ponta através do Gateway sem serem removidos por *middleware* intermediário. |

---

## 11. Ambientes

| Ambiente | Réplicas mínimas | Dados | Uso |
|----------|--------------------|-------|-----|
| `dev` | 1 | Sintéticos, resetáveis | Desenvolvimento local (Docker Compose, §4). |
| `staging` | 3 | Mascarados/sintéticos representativos de produção | Validação de rollout, testes de carga (`Benchmark.md`), *chaos drills* (`FailureRecovery.md`), testes de rebalanceamento de shard. |
| `production` | ≥ 3, distribuídas em ≥ 2 AZ | Reais, sob RLS/mTLS/backup completo | Operação real; sujeito a todos os SLOs de `NonFunctionalRequirements.md`. |

---

## 12. Migrações de Banco de Dados

- Migrações de schema **DEVEM** ser versionadas e aplicadas por ferramenta de
  migração declarativa (ex.: EF Core Migrations / Flyway) executada como um
  passo de *pre-deploy* separado do rollout de réplicas da aplicação.
- Toda migração **DEVE** ser testada quanto a compatibilidade com a versão de
  código **anterior** (para suportar rolling update sem downtime) antes de ser
  aplicada em produção — em particular, `SchedulingDecision` é append-only
  (event sourcing), de modo que migrações **NÃO DEVEM** alterar dados
  históricos, apenas adicionar colunas/índices de forma aditiva.
- Detalhes de DDL, índices, particionamento e retenção: `./Database.md`.

---

## 13. Referências

- Arquitetura geral e diagrama de componentes: `./Architecture.md`, `./_DESIGN_BRIEF.md` §2.
- Modelo de escala, sharding e ownership de shard: `./Scalability.md`.
- Modos de falha e recuperação: `./FailureRecovery.md`.
- Postura de segurança e hardening: `./Security.md`.
- Contratos centrais (correlação, versionamento de API): `../003-RFC/RFC-0001-Architecture-Baseline.md` §5.6–§5.7.
- Alta disponibilidade e inventário de capacidade: `../027-Cluster/`.
- Gateway e infraestrutura de borda: `../028-Deployment/`.
- Observabilidade (métricas/probes): `../024-Observability/`.
