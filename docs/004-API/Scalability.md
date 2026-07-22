---
Documento: Scalability
Módulo: 004-API
Status: Draft
Versão: 0.1
Última atualização: 2026-07-21
Responsável (RACI-A): Arquiteto do Módulo 004-API
ADRs relacionados: ADR-0040, ADR-0044
RFCs relacionados: RFC-0001
Depende de: ./_DESIGN_BRIEF.md §10, ./Deployment.md, ./NonFunctionalRequirements.md
---

# 004-API — Escalabilidade e Concorrência

> Reproduz e detalha `./_DESIGN_BRIEF.md` §10 (fonte única de verdade).

## 1. Modelo de escala

O Gateway é **puramente horizontal**: por ser um proxy *stateless*, qualquer
réplica atende qualquer requisição, sem *sharding* de estado por tenant ou
agente. Isso o diferencia estruturalmente de módulos de domínio como
`006-Kernel` (que fragmenta ACBs por `hash(tenant,agent) mod N`) — o
004-API **não fragmenta estado por shard**.

```
        L7 Load Balancer (../027-Cluster/)
                    │
    ┌───────────────┼───────────────┐
    ▼               ▼               ▼
┌────────┐     ┌────────┐     ┌────────┐      (N réplicas, sem afinidade,
│Réplica 1│     │Réplica 2│ ...│Réplica N│       escala linear até o limite
└────┬───┘     └────┬───┘     └────┬───┘       de conexões de saída/upstream)
     │               │               │
     └───────────────┴───────────────┘
                    │ estado externalizado
     ┌──────────────┼──────────────┐
     ▼               ▼               ▼
  Redis          PostgreSQL       NATS/JetStream
```

## 2. Particionamento

| Dimensão | Estratégia |
|----------|-----------|
| Particionamento | Não aplicável a este módulo (proxy *stateless* — ver `./_DESIGN_BRIEF.md` §10, nota de decisão ADR-0040). |
| Estado externalizado | Contadores de rate limit e cache de idempotência em **Redis**; registros de contrato em **PostgreSQL**; *snapshot* de rotas/versões carregado em memória por réplica com invalidação por evento (`aios._platform.api.*`). |
| Réplicas | Escala horizontal pura atrás do balanceador L7, com *anti-affinity* por zona (`../028-Deployment/`). Sem afinidade obrigatória. |

## 3. Concorrência

- **Registro de contrato** (`RouteDefinition`, `ApiVersion`): OCC (campo
  `version`) — mutações são de baixa frequência (administrativas), sem
  necessidade de *locks* distribuídos.
- **Rate limit**: *token-bucket* atômico via *script* Lua no Redis, evitando
  condição de corrida entre réplicas concorrentes acessando o mesmo bucket
  de `(tenant, rota)`.
- **Idempotência**: reserva atômica (`SETNX`-equivalente) no Redis antes de
  encaminhar ao upstream, com confirmação durável no PostgreSQL —
  garante efeito único mesmo sob requisições concorrentes com a mesma
  `Idempotency-Key`.

## 4. Backpressure

| Mecanismo | Componente | Efeito |
|-----------|------------|--------|
| Rejeição antecipada por cota | `RateLimiter` | `429` por tenant/rota antes de consumir recursos de *downstream*. |
| Isolamento de upstream saturado | `CircuitBreakerManager` | `503` imediato sem tentar a chamada de rede, poupando *threads*/conexões. |
| Limite de *streams* por réplica | `SseStreamRelay` | `503` acima de `api.sse.max_streams_per_replica`, sinalizando ao balanceador redirecionar para outra réplica/escalar. |

## 5. Caminho quente

AuthN com cache JWKS local, AuthZ com cache de decisão de TTL curto, rate
limit em Redis (sub-ms) — sem I/O bloqueante síncrono desnecessário no
caminho `EdgeRouter`→upstream. Alvo de overhead p99 ≤ 5 ms (NFR-001), ver
`./Benchmark.md`.

## 6. SSE / conexões longas

Cada réplica multiplexa suas próprias conexões SSE, assinando os subjects
NATS relevantes por tenant sob demanda; **não há afinidade** de
cliente-réplica exigida — qualquer réplica PODE assumir a assinatura de um
tenant/tarefa a qualquer momento, o que simplifica *rolling updates* (uma
réplica sendo desligada apenas fecha suas conexões SSE; o cliente
reconecta a outra réplica, sem perda de eventos graças à durabilidade do
JetStream).

## 7. Limites teóricos

| Limite | Valor | Origem |
|--------|-------|--------|
| *Streams* SSE por réplica | `api.sse.max_streams_per_replica` (default 20.000, faixa até 200.000) | Restrição operacional de memória de conexão, não arquitetural. |
| Throughput por réplica | ≥ 150.000 req/s (NFR-002) | Restrição de CPU/*I/O* medida em `./Benchmark.md`. |
| Número de réplicas | Sem limite teórico (stateless) | Restrito apenas pelo orçamento de `../028-Deployment/` e pela capacidade agregada de `021`/`022`/upstreams. |

## 8. Rumo a milhões (de agentes, indiretamente)

O volume de requisições do Gateway escala com o número de
**operadores/integrações** (humanos, CLIs, dashboards, SDKs) e com picos de
administração (ex.: *spawn* em lote), **não linearmente** com o número de
agentes ativos — a carga de execução de agentes trafega majoritariamente
**dentro** do plano de controle (gRPC interno, NATS), não através deste
Gateway. Ver `../001-Architecture/Architecture.md` §11. Consequentemente, a
meta de escala V-R1 da Visão Global (10⁶+ agentes) **não** impõe ao 004-API
o mesmo requisito de escala de estado que impõe a `006-Kernel`/`010-Memory`
— o Gateway escala por **conexões e throughput de borda**, não por número
de entidades gerenciadas.

*Fim de `Scalability.md`.*
