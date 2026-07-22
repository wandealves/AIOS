---
Documento: Deployment
Módulo: 004-API
Status: Draft
Versão: 0.1
Última atualização: 2026-07-21
Responsável (RACI-A): Arquiteto do Módulo 004-API
ADRs relacionados: ADR-0040, ADR-0047
RFCs relacionados: RFC-0001
Depende de: ../027-Cluster/README.md, ../028-Deployment/README.md, ./Configuration.md, ./Scalability.md
---

# 004-API — Implantação

> Topologia de implantação do API Gateway. O Gateway **é operado** pela
> topologia definida em `../028-Deployment/` e `../027-Cluster/` — este
> documento descreve como o módulo se encaixa nessa topologia, não a
> redefine (NR-08 do brief).

## 1. Topologia (ASCII)

```
                               Internet / Rede de Parceiros
                                          │
                              ┌───────────▼───────────┐
                              │  L7 Load Balancer /     │
                              │  Edge Protection (027)  │
                              └───────────┬───────────┘
                                          │ HTTPS/gRPC/SSE
              ┌───────────────────────────┼───────────────────────────┐
              ▼                           ▼                           ▼
      ┌───────────────┐          ┌───────────────┐          ┌───────────────┐
      │ 004-API réplica │          │ 004-API réplica │          │ 004-API réplica │
      │   AZ-1           │          │   AZ-2           │          │   AZ-3           │
      └───────┬───────┘          └───────┬───────┘          └───────┬───────┘
              │ mTLS interno              │ mTLS interno              │ mTLS interno
              └─────────────┬─────────────┴─────────────┬─────────────┘
                             ▼                           ▼
                  ┌────────────────────┐      ┌────────────────────┐
                  │ PostgreSQL HA (api)  │      │ Redis Cluster        │
                  │ (`../005-Database/`) │      │ (`../005-Database/`) │
                  └────────────────────┘      └────────────────────┘
                             │
                             ▼
                  ┌────────────────────┐
                  │ NATS/JetStream       │
                  │ (`../020-Communication/`) │
                  └────────────────────┘
                             │
                             ▼
             upstreams internos (006-019, 021-027) — apenas via mTLS
             a partir da malha do Gateway; NÃO alcançáveis da Internet
```

## 2. Recursos por réplica

| Recurso | Valor recomendado | Racional |
|---------|---------------------|----------|
| CPU | 2 vCPU (request) / 4 vCPU (limit) | Proxy *I/O-bound*; CPU cresce com validação de schema e TLS *termination*. |
| Memória | 1 GiB (request) / 2 GiB (limit) | Cache JWKS, *snapshot* de rotas/versões em memória, buffers de *streams* SSE. |
| GPU | Não aplicável | O Gateway não executa inferência. |
| Conexões de saída (upstream) | Pool dimensionado por `api.gateway.upstream_timeout_ms` e `CircuitBreakerManager` (bulkhead por serviço). | Evita exaustão de conexões por um único upstream lento. |

## 3. Réplicas e anti-affinity

- Mínimo de **3 réplicas**, uma por zona de disponibilidade (AZ), para
  sustentar NFR-003 (≥ 99,99% de disponibilidade) mesmo sob perda de uma AZ.
- **Anti-affinity obrigatória** entre réplicas do mesmo serviço (nunca duas
  réplicas na mesma AZ como única cobertura) — política definida e aplicada
  por `../027-Cluster/`.
- **Autoscaling horizontal** por CPU e por conexões ativas (incluindo
  *streams* SSE), com limite superior determinado pelo orçamento de
  `../028-Deployment/`; o Gateway não impõe limite teórico de réplicas
  (stateless, ver `./Scalability.md`).

## 4. Health e readiness

| Probe | Endpoint | Critério |
|-------|----------|----------|
| Liveness | `GET /v1/healthz` | Processo respondendo; não valida dependências externas. |
| Readiness | `GET /v1/readyz` | *Snapshot* de rotas/versões carregado do `ContractRegistry`; conexão com Redis estabelecida; **não** exige `021`/`022` disponíveis (o Gateway PODE ficar `Ready` e ainda assim negar por `fail_mode=closed`). |

Uma réplica que falha `readyz` **DEVE** ser removida do balanceador
imediatamente; uma réplica que falha `healthz` **DEVE** ser reiniciada pela
orquestração (`../028-Deployment/`).

## 5. Estratégia de rollout

- **Rolling update** padrão, com `maxUnavailable=0` e `maxSurge=1` (nunca
  reduzir capacidade de borda durante o *deploy*, dado que o Gateway é
  ponto único de entrada).
- **Canary opcional** por porcentagem de tráfego roteado à nova revisão,
  coordenado por `../027-Cluster/`; recomendado para mudanças no
  `EdgeRouter`/`AuthNFilter` (caminho crítico).
- Mudanças em `RouteDefinition`/`ApiVersion` (registro de contrato) **NÃO
  DEVEM** exigir *redeploy* do serviço — são aplicadas via evento
  (`aios._platform.api.route.*`, `aios._platform.api.version.*`) que
  invalida o *snapshot* em memória de cada réplica, sem *restart*.

## 6. Papel do YARP como *gateway* de topologia

O `EdgeRouter` (núcleo YARP) é, ele próprio, o *reverse proxy* de borda do
AIOS — não há um segundo gateway físico entre a Internet e o 004-API além
da proteção L7/edge de `../027-Cluster/` (WAF, *DDoS protection*). Essa
proteção de borda de infraestrutura é complementar, não substitui o
`RouteAuthorizer`/`AuthNFilter` do Gateway.

## 7. Dependências de implantação

| Dependência | Ordem de subida | Comportamento se ausente no *boot* |
|--------------|-------------------|--------------------------------------|
| PostgreSQL (`api` schema) | Antes do Gateway aceitar tráfego. | Réplica não fica `Ready` até obter o *snapshot* inicial de rotas/versões. |
| Redis | Antes do Gateway aceitar tráfego. | Réplica não fica `Ready` (rate limit/idempotência exigem Redis). |
| NATS/JetStream | Pode subir em paralelo; consumido de forma assíncrona. | Réplica fica `Ready`, mas eventos de invalidação de *snapshot* ficam pendentes até reconectar (ver `./FailureRecovery.md`). |
| `021-Security` (JWKS) | Independente. | AuthN usa cache local até `api.auth.jwks_cache_ttl_s`; expirado sem religar → `fail_mode=closed`. |
| `022-Policy` (PDP) | Independente. | AuthZ usa cache de decisão; sem decisão fresca → `fail_mode=closed`. |

*Fim de `Deployment.md`.*
