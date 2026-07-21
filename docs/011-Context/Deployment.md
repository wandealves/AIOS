---
Documento: Deployment
Módulo: 011-Context
Status: Draft
Versão: 0.1
Última atualização: 2026-07-20
Responsável (RACI-A): Arquiteto do Módulo 011-Context
ADRs relacionados: ADR-0110, ADR-0111, ADR-0113, ADR-0117, ADR-0118
RFCs relacionados: RFC-0001 (baseline), RFC-0011 (proposta)
Depende de: 001-Architecture, 005-Database, 010-Memory, 017-Model-Router, 020-Communication, 021-Security, 022-Policy, 024-Observability, 028-Deployment, 040-Glossary
---

# 011-Context — Deployment

> Topologia de implantação do `ContextService`. **Deriva** do brief (§2, §3, §9,
> §10) e **NÃO PODE contradizê-lo**. O `ContextService` roda em **.NET 10** no
> plano de controle, exposto por **YARP** (REST/gRPC). Palavras normativas
> conforme RFC 2119/8174.

## 1. Topologia de implantação (ASCII)

```
                         ┌───────────────────────── AIOS Control Plane ─────────────────────────┐
   Agent Runtime (007)  │  ┌──────────┐    REST /v1/context · gRPC aios.context.v1              │
   Planning (012)  ───▶ │  │  YARP     │──┐  (mTLS interno; TLS externo terminado no gateway)    │
   (OAuth2/OIDC)         │  │ Gateway   │  │                                                       │
                         │  │  (028)    │  │  ┌───────────────────────────────────────────────┐   │
                         │  └──────────┘  └─▶│ context-service (.NET 10) · N réplicas (HPA)    │   │
                         │                    │ ContextApiGateway/PolicyGuard/Assembler/Budgeter│   │
                         │                    │ Retriever/Ranker/Dedup/Compressor/CacheManager  │   │
                         │                    │ BundleStore/EvictionManager/EventPublisher      │   │
                         │                    └───┬───────┬───────────┬──────────┬──────────────┘   │
                         └────────────────────────┼───────┼───────────┼──────────┼──────────────────┘
                                                  │       │           │          │
                                     ┌────────────▼┐ ┌────▼──────┐ ┌──▼───────┐ ┌▼───────────────┐
                                     │ Redis        │ │PostgreSQL │ │  MinIO   │ │ NATS JetStream │
                                     │ (L1 cache,   │ │+pgvector  │ │ (blobs de│ │ (eventos,      │
                                     │  locks,      │ │ RLS/tenant│ │ fragments│ │  domínio       │
                                     │  idempot.)   │ │ (005)     │ │ grandes) │ │  context, 020) │
                                     └──────────────┘ └───────────┘ └──────────┘ └────────────────┘
                                                  ▲                            ▲
                                     Model Router (017) ◀── limites/tokenizer/embedding/summ.
                                     Memory (010) ◀── recall seletivo   Policy (022) ◀── PDP
```

O `context-service` é **stateless** no processo (estado vive nos backends); PODE
escalar horizontalmente. Estado quente do cache semântico (L1) e locks de
*single-flight* ficam em Redis; a fonte da verdade durável (`ContextBundle`,
`ContextFragment`, `SemanticCacheEntry` L2, `BudgetProfile`, `SummaryNode`) fica
em PostgreSQL (+`pgvector`) com RLS por tenant; fragmentos grandes em MinIO;
eventos `aios.<tenant>.context.*` em NATS/JetStream (brief §2, §9, §10).

## 2. Dependências de implantação

| Dependência | Papel | Módulo | Criticidade |
|-------------|-------|--------|-------------|
| PostgreSQL + pgvector | Fonte da verdade durável de bundles/fragments/cache L2/budgets/summaries; índice ANN (HNSW/IVFFLAT) | [005-Database](../005-Database/Database.md) | **hard** (durabilidade) |
| Redis | Cache L1 do cache semântico, locks de *single-flight*, `IdempotencyStore` | 005/027 | **soft** (bypass de cache em falha, FM-03) |
| MinIO | *Offload* de fragmentos/blobs `> context.limits.max_inline_bytes` | brief §2, §3.2 | soft (fragmento mantém inline enquanto viável) |
| NATS/JetStream | Eventos `context.window.*`/`context.cache.*`/`context.budget.*` (Outbox) + eventos consumidos (§6.2) | [020-Communication](../020-Communication/Deployment.md) | hard (Outbox garante RPO evento ≤ 60 s) |
| Model Router (017) | Limites de modelo, registro de tokenizer, embedding e sumarização | [017-Model-Router](../017-Model-Router/API.md) | **soft** (degradação com limites cacheados, FM-01/FM-05) |
| Memory (010) | Recall seletivo de candidatos de contexto | [010-Memory](../010-Memory/API.md) | soft (montagem parcial em `DEGRADED`, FM-02) |
| Policy Engine (022) | PDP para o `ContextPolicyGuard` | [022-Policy](../022-Policy/API.md) | hard (default deny) |
| Gateway/YARP (028) | Terminação TLS, roteamento, rate-limit por tenant | [028-Deployment](../028-Deployment/Deployment.md) | hard |
| Security (021) | OAuth2/OIDC, mTLS, cofre de segredos | [021-Security](../021-Security/Security.md) | hard |
| Observability (024) | OTel/Prometheus/Seq | [024-Observability](../024-Observability/Deployment.md) | soft |
| Audit (025) | Trilha imutável de operações privilegiadas | [025-Audit](../025-Audit/Events.md) | soft (emissão *best-effort*, mas obrigatória) |

> **Fronteira crítica (brief §1.2/§2.1):** o Agent Runtime (Python, plano de
> dados) NÃO DEVE acessar Redis/PostgreSQL do módulo diretamente — DEVE passar
> pelo `context-service` via REST/gRPC.

## 3. Recursos e réplicas

Perfil de referência por réplica (control plane, sem GPU — embeddings e
sumarização ficam no `017`):

| Item | Requests | Limits |
|------|----------|--------|
| CPU | 1 vCPU | 2 vCPU |
| Memória | 1 GiB | 2 GiB |
| GPU | — (nenhuma) | — |
| Réplicas (default) | 3 | — |
| Autoscale (HPA) | 3 | 32 |

- **Escala horizontal:** stateless; alvo de HPA por CPU/RSS e por latência p99
  de `assemble`. Sharding determinístico
  `shard = hash(tenant_id, agent_id) mod context.shard.count` (brief §10).
- **Metas de capacidade:** ≥ 500 req/s de `assemble` por réplica (NFR-009);
  `CacheLookup` p99 ≤ 20 ms (NFR-001); `Assemble` p99 ≤ 150 ms sem sumarização
  (NFR-002) e ≤ 1200 ms com 1 chamada de sumarização (NFR-003).
- **Backends** (Redis/PostgreSQL/MinIO/NATS) têm dimensionamento próprio em
  [005-Database](../005-Database/Deployment.md) e
  [027-Cluster](../027-Cluster/Deployment.md); não escalam junto com o
  `context-service`.

## 4. Health e readiness

| Probe | Endpoint | Verifica | Falha ⇒ |
|-------|----------|----------|---------|
| Liveness | `GET /healthz` | processo vivo; sem deadlock | reinício do container |
| Readiness | `GET /readyz` | PostgreSQL alcançável, NATS conectado, migrações aplicadas | fora do balanceador |
| Startup | `GET /startupz` | bootstrap concluído (config, esquema, índice `pgvector` pronto) | atrasa liveness |

Regras:
- Readiness DEVE reprovar se **PostgreSQL** ou **NATS** (dependências *hard*)
  estiverem indisponíveis. Redis, MinIO e `017-Model-Router` indisponíveis
  **não** reprovam readiness — o serviço opera em modo degradado: bypass de
  cache (FM-03), *offload* de blob adiado, ou limites de modelo cacheados
  (FM-01) — consistente com `context.degraded.allow_truncate_fallback=true`.
- Readiness expõe a saúde por dependência para o Gateway/YARP e para os
  dashboards de [Monitoring](../024-Observability/Monitoring.md).

## 5. Compose / manifesto (exemplo)

Trecho ilustrativo (Docker Compose de ambiente local; produção usa o manifesto
de [028-Deployment](../028-Deployment/Deployment.md)):

```yaml
services:
  context-service:
    image: registry.aios.local/aios/context-service:1.0
    deploy:
      replicas: 3
      resources:
        reservations: { cpus: "1.0", memory: 1Gi }
        limits:       { cpus: "2.0", memory: 2Gi }
    environment:
      AIOS_CONTEXT_EMBEDDING_DIM: "1536"
      AIOS_CONTEXT_CACHE_VECTOR_INDEX: "hnsw"
      AIOS_CONTEXT_RETRIEVAL_TOP_K: "32"
      AIOS_CONTEXT_SHARD_COUNT: "64"
      AIOS_CONTEXT_RATELIMIT_ASSEMBLE_RPS_PER_TENANT: "1000"
      # segredos injetados do cofre (não em texto claro) — ver Security.md
      AIOS_CONTEXT_PG_DSN__secretRef: context-pg-dsn
      AIOS_CONTEXT_REDIS_URL__secretRef: context-redis-url
      AIOS_CONTEXT_NATS_URL__secretRef: context-nats-url
      AIOS_CONTEXT_MINIO__secretRef: context-minio-creds
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8080/readyz"]
      interval: 10s
      timeout: 3s
      retries: 3
    depends_on: [postgres, redis, minio, nats]
    networks: [aios-control-plane]

networks:
  aios-control-plane: { external: true }
```

Roteamento no YARP (Gateway 028) — trecho de cluster/rota:

```json
{
  "Routes": {
    "context-rest": { "ClusterId": "context", "Match": { "Path": "/v1/context/{**catch-all}" } },
    "context-grpc": { "ClusterId": "context-grpc", "Match": { "Path": "/aios.context.v1.ContextService/{**catch-all}" } }
  },
  "Clusters": {
    "context":      { "Destinations": { "d1": { "Address": "http://context-service:8080/" } }, "HttpRequest": { "ActivityTimeout": "00:00:02" } },
    "context-grpc": { "Destinations": { "d1": { "Address": "https://context-service:8443/" } } }
  }
}
```

> O `ActivityTimeout` do cluster REST (2 s) é deliberadamente curto para
> refletir o SLO de `Assemble` sem sumarização (NFR-002 ≤ 150 ms) mais margem;
> a rota de `assemble` com sumarização usa um cluster dedicado com timeout
> alinhado a NFR-003 (≤ 1200 ms).

## 6. Estratégia de rollout

| Aspecto | Regra |
|---------|-------|
| Estratégia | **Rolling update** com `maxSurge=1`, `maxUnavailable=0`; drena via readiness. |
| Migrações de schema | Aplicadas antes do rollout, compatíveis para trás (expand/contract). Ver [Database.md](./Database.md). |
| Reindex vetorial (`pgvector`) | Alterações em `context.cache.vector_index`/`context.embedding.dimensions` **não são recarregáveis** (brief §8) e exigem reindex/migração dedicada — nunca no mesmo passo de um rollout de código. |
| Compatibilidade de API | Coexistência de ≥ 2 majors (RFC-0001 §5.7); `v1` e futuras versões servem lado a lado. |
| Outbox durante deploy | `EventPublisher` reprocessa eventos pendentes no start; nenhum evento perdido (RPO evento ≤ 60 s, FM-07). |
| Rollback de deploy | Imagem anterior reimplantável; estado nos backends é compatível (expand/contract garante). |
| RTO/RPO alvo | RTO ≤ 5 min; RPO ≤ 60 s — NFR-012. |
| Disponibilidade alvo | ≥ 99,95% (NFR-008). |

Sequência de deploy:

```
1. aplicar migração DB (expand)         → compatível com versão atual
2. rolling update context-service       → maxUnavailable=0, readiness drena
3. verificar SLI (compression_ratio,    → gate de qualidade (Monitoring)
   cache hit ratio, p99 assemble)
4. aplicar contract (remoções tardias)  → após todas as réplicas na nova versão
```

## 7. Riscos e alternativas

| Risco / decisão | Mitigação / alternativa |
|-------------------|--------------------------|
| Backend durável (PostgreSQL) é dependência *hard* e SPOF potencial. | Réplica streaming + failover (`027`); readiness reprova; RTO ≤ 5 min / RPO ≤ 60 s (NFR-012, FM-10). |
| Redis indisponível durante deploy afeta cache L1. | Bypass de cache (`cache_outcome=BYPASS`) sem bloquear readiness (FM-03); serviço segue pelo caminho `MISS`. |
| Reindex vetorial concorrente com rollout satura o nó. | Reindex nunca no mesmo passo do rollout; janela dedicada; `context.cache.vector_index`/`context.embedding.dimensions` não recarregáveis (brief §8). |
| Sharding fixo dificulta rebalanceamento a milhões de agentes. | `context.shard.count` (não recarregável) previsto para migração online por tenant; caminho evolutivo alinhado a `ADR-0111`. |
| `017-Model-Router` degradado durante rollout aumenta latência de sumarização. | Circuit breaker + *fallback* de truncamento determinístico (`AIOS-CTX-0004`); rollout não depende de `017` estar saudável para completar. |

Decisões governadas por `ADR-0110` (compressão hierárquica), `ADR-0111` (cache
em dois níveis), `ADR-0113` (token budgeting), `ADR-0117` (offload MinIO) e
`ADR-0118` (degradação graciosa) — ver [ADR.md](./ADR.md).
