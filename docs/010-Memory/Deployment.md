---
Documento: Deployment
Módulo: 010-Memory
Status: Draft
Versão: 0.1
Última atualização: 2026-07-20
Responsável (RACI-A): Arquiteto do Módulo 010-Memory
ADRs relacionados: ADR-0100 (7 camadas → backends físicos), ADR-0101 (embeddings/HNSW), ADR-0106 (blobs MinIO), ADR-0107 (fronteira Runtime↔Memory), ADR-0108 (sharding vetorial/Episodic); ADR-0002 (Microsserviços Control/Data Plane), ADR-0003 (.NET Control / Python Runtime), ADR-0004 (NATS como Barramento), ADR-0006 (Redis Estado Quente) — herdadas
RFCs relacionados: RFC-0001 (baseline)
Depende de: 001-Architecture, 005-Database, 017-Model-Router, 020-Communication, 021-Security, 022-Policy, 024-Observability, 026-Cost-Optimizer, 027-Cluster, 028-Deployment, 040-Glossary
---

# AIOS — Módulo 010 · Memory — Deployment

> **Escopo.** Este documento especifica como o `MemoryService` (010) é
> empacotado, implantado, escalado e operado: topologia de containers,
> recursos por réplica, dependências externas, health/readiness, estratégia
> de *rollout* e integração com o Gateway (YARP). Não redefine os contratos
> centrais de RFC-0001 nem os modos de falha (ver `FailureRecovery.md`) ou o
> modelo de escala/sharding (ver `Scalability.md`) — este documento foca em
> **como o serviço é colocado em produção**, referenciando aqueles onde
> necessário. Palavras normativas conforme RFC 2119/8174.

---

## Índice

1. Visão Geral da Topologia
2. Requisitos de Infraestrutura por Réplica
3. Containerização
4. Topologia de Implantação — Docker Compose
5. Réplicas e Sharding
6. Dependências Externas
7. Health e Readiness
8. Estratégia de Rollout
9. Escalonamento
10. Configuração de Rede e Gateway (YARP)
11. Ambientes
12. Migrações de Banco de Dados e Reindexação
13. Referências

---

## 1. Visão Geral da Topologia

O `MemoryService` é um processo **stateless** (.NET 10), implantado como
múltiplas réplicas idênticas, acessível por gRPC interno (Agent Runtime 007
via Kernel 006, Context Manager 011, Learning 023) e por REST administrativo
via Gateway (YARP). Todo estado autoritativo é externalizado em PostgreSQL
(`MemoryItem`, `MemoryLayerConfig`, `ConsolidationJob`, `ConsolidationVersion`,
`ForgettingPolicy`, `MemoryQuota`, `OutboxEvent`, `IdempotencyRecord` — fonte
da verdade, com `pgvector` e Apache AGE), Redis (Working/Short-Term quente,
`IdempotencyStore`, contadores de cota, locks de consolidação) e MinIO
(blobs grandes e *snapshots* de `ConsolidationVersion`), e toda comunicação
assíncrona passa pelo NATS/JetStream.

```
                                   Rede interna do Control Plane
                                             │
              gRPC + mTLS (interno, sem Gateway externo)
                                             │
                    ┌────────────────────────┼─────────────────────────┐
                    ▼                        ▼                         ▼
           ┌─────────────────┐      ┌──────────────────┐       ┌─────────────────┐
           │ Agent Runtime    │      │ Context Manager   │      │  Learning (023)   │
           │ (007, via 006)   │      │      (011)         │      │  consolidação     │
           └────────┬─────────┘      └─────────┬─────────┘      │  dirigida         │
                    │ remember/recall           │ recall seletivo└────────┬─────────┘
                    ▼                           ▼                         ▼
           ┌──────────────────────────────────────────────────────────────────────┐
           │                     MEMORY SERVICE — pool de réplicas                 │
           │   ┌─────────────────┐  ┌─────────────────┐        ┌─────────────────┐│
           │   │ réplica 1        │  │ réplica 2        │  ...  │ réplica N        ││
           │   │ MemoryApiFacade  │  │ MemoryApiFacade  │       │ MemoryApiFacade  ││
           │   │ +Pep+LayerRouter │  │ +Pep+LayerRouter │       │ +Pep+LayerRouter ││
           │   └────────┬─────────┘  └────────┬─────────┘       └────────┬─────────┘│
           └────────────┼──────────────────────┼──────────────────────────┼─────────┘
                         │                      │                          │
        ┌────────────────┼──────────────────────┼──────────────────────────┼───────────┬─────────────┐
        ▼                ▼                      ▼                          ▼           ▼             ▼
  PostgreSQL HA     Redis Cluster           NATS/JetStream Cluster       MinIO      022-Policy   017-Model-Router
 (+pgvector+AGE,   (Working/Short-Term,     (aios.<t>.memory.*,        (blobs/     (PDP)         (embeddings)
  RLS por tenant)   idempotência, locks,     DLQ, consolidação/jobs)   snapshots)      │              │
                     contadores de cota)                                              ▼              ▼
                                                                                  gRPC+mTLS      gRPC+mTLS
```

Todas as réplicas do `MemoryService` são **funcionalmente equivalentes** e
**stateless**; a diferença entre elas é apenas o roteamento por
`shard = hash(tenant_id, agent_id) mod N` (otimização de localidade de
cache, não requisito de correção — ver `Scalability.md` §2).

---

## 2. Requisitos de Infraestrutura por Réplica

| Recurso | Valor recomendado (baseline) | Observação |
|---------|-------------------------------|-------------|
| CPU | 2 vCPU (request), 4 vCPU (limit) | `RecallEngine` (fusão RRF, re-rank) e serialização/validação de DTOs são os principais consumidores de CPU no caminho quente. |
| Memória | 2 GiB (request), 4 GiB (limit) | Filas/cotas/idempotência residem em Redis; heap cobre conexões, *buffers* de *batching* de embeddings e cache local de decisão PDP/capacidade. |
| GPU | Não aplicável | O `MemoryService` não executa inferência; embeddings são delegados a `017-Model-Router`. |
| Disco | Efêmero, ≤ 1 GiB | Sem estado persistente local; usado apenas para *logs* de curto prazo antes do envio a Seq/OTel *collector*. |
| Rede | Baixa latência (mesma região/AZ que Redis/PostgreSQL/NATS/MinIO) | Latência de rede às dependências é o principal risco aos SLOs de `remember`/`recall` (NFR-001/003). |

> Estes valores são **baseline de dimensionamento inicial**; o
> dimensionamento definitivo por ambiente é validado em `Benchmark.md` e
> ajustado por autoescalonamento (§9). Backends (PostgreSQL/Redis/MinIO/NATS)
> têm dimensionamento próprio, especificado em `../005-Database/Deployment.md`
> e `../027-Cluster/Deployment.md` — não escalam junto com o `MemoryService`.

---

## 3. Containerização

| Aspecto | Especificação |
|---------|-----------------|
| Runtime | .NET 10, imagem de execução mínima (distroless-equivalente). |
| Compilação | AOT (*Ahead-of-Time*) quando suportado pela plataforma alvo, para reduzir tempo de *start-up* e superfície de *runtime* (ver Glossário: AOT). |
| Usuário | Não-root, UID/GID dedicados, sem capacidades Linux adicionais. |
| Sistema de arquivos | Raiz somente-leitura; `/tmp` efêmero para *buffers* de curto prazo. |
| Variáveis de ambiente | Prefixo `AIOS_MEMORY_` para todas as chaves de configuração (`memory.*`, ver `_DESIGN_BRIEF.md` §8 / `Configuration.md`); segredos (DSNs, credenciais) **NÃO DEVEM** ser passados em texto puro — usar montagem de arquivo a partir do cofre de segredos ou `secretRef` (`Security.md` §6). |
| Portas expostas | `5010` (gRPC interno `aios.memory.v1`, mTLS), `8080` (REST admin `/v1/memory/*`, atrás do Gateway), `9090` (métricas Prometheus, rede interna de observabilidade). |
| Tags de imagem | Semânticas (`010-memory:1.2.0`), imutáveis; **NÃO DEVEM** ser sobrescritas (`:latest` proibido em produção). |

---

## 4. Topologia de Implantação — Docker Compose (ambiente de desenvolvimento/integração)

```yaml
services:
  memory-service:
    image: registry.internal/aios/010-memory:${MEMORY_VERSION}
    deploy:
      replicas: 3
      resources:
        limits: { cpus: "4", memory: 4g }
        reservations: { cpus: "2", memory: 2g }
      restart_policy: { condition: on-failure, max_attempts: 5 }
    environment:
      - AIOS_MEMORY_EMBEDDING_DIM=1024
      - AIOS_MEMORY_HNSW_M=16
      - AIOS_MEMORY_HNSW_EF_CONSTRUCTION=128
      - AIOS_MEMORY_HNSW_EF_SEARCH=64
      - AIOS_MEMORY_RECALL_DEFAULT_K=10
      - AIOS_MEMORY_RECALL_FUSION=rrf
      - AIOS_MEMORY_BACKPRESSURE_MAX_INFLIGHT=1000
      - AIOS_MEMORY_ITEM_INLINE_MAX_BYTES=65536
      - AIOS_MEMORY_BLOB_BUCKET=aios-memory
      - AIOS_MEMORY_CONSOLIDATION_AUTO=true
      - AIOS_MEMORY_RTBF_ENABLED=true
    secrets:
      - memory_pg_credentials
      - memory_redis_credentials
      - memory_minio_credentials
      - memory_nats_account_jwt
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8080/readyz"]
      interval: 10s
      timeout: 3s
      retries: 3
      start_period: 20s
    depends_on:
      postgres: { condition: service_healthy }
      redis: { condition: service_healthy }
      minio: { condition: service_healthy }
      nats: { condition: service_healthy }
    networks: [aios-control-plane]

  postgres:
    image: pgvector/pgvector:pg16
    environment:
      - POSTGRES_DB=aios_memory
    volumes: [pg-memory-data:/var/lib/postgresql/data]
    # Apache AGE habilitada via CREATE EXTENSION age; carregada no boot da imagem
    networks: [aios-control-plane]

  redis:
    image: redis:7-alpine
    command: ["redis-server", "--requirepass", "${REDIS_PASSWORD}"]
    networks: [aios-control-plane]

  minio:
    image: minio/minio:latest
    command: ["server", "/data", "--console-address", ":9001"]
    volumes: [minio-memory-data:/data]
    networks: [aios-control-plane]

  nats:
    image: nats:2-alpine
    command: ["-js", "-sd", "/data"]
    volumes: [nats-memory-data:/data]
    networks: [aios-control-plane]

networks:
  aios-control-plane: { driver: bridge }

volumes:
  pg-memory-data:
  minio-memory-data:
  nats-memory-data:

secrets:
  memory_pg_credentials: { external: true }
  memory_redis_credentials: { external: true }
  memory_minio_credentials: { external: true }
  memory_nats_account_jwt: { external: true }
```

> Em ambientes de produção, PostgreSQL, Redis, MinIO e NATS **DEVEM** ser
> *clusters* gerenciados de alta disponibilidade compartilhados/dedicados
> (ver `../027-Cluster/`), não containers efêmeros do Compose — o exemplo
> acima é para desenvolvimento e testes de integração locais.

Roteamento no YARP (Gateway) — trecho de rota/*cluster*:

```json
{
  "Routes": {
    "memory-rest": { "ClusterId": "memory", "Match": { "Path": "/v1/memory/{**catch-all}" } },
    "memory-grpc": { "ClusterId": "memory-grpc", "Match": { "Path": "/aios.memory.v1.MemoryService/{**catch-all}" } }
  },
  "Clusters": {
    "memory":      { "Destinations": { "d1": { "Address": "http://memory-service:8080/" } }, "HttpRequest": { "ActivityTimeout": "00:00:05" } },
    "memory-grpc": { "Destinations": { "d1": { "Address": "https://memory-service:5010/" } } }
  }
}
```

---

## 5. Réplicas e Sharding

| Aspecto | Regra |
|---------|-------|
| Contagem mínima de réplicas em produção | **3** (tolerância a falha de 1 zona sem perda de capacidade de atendimento). |
| Escalonamento horizontal | Baseado em CPU (%), `aios_memory_inflight` (relativo a `memory.backpressure.max_inflight`) e `aios_memory_remember_duration_ms`/`aios_memory_recall_duration_ms` p99 (ver §9). |
| Natureza *stateless* | Qualquer réplica **PODE** atender qualquer tenant/shard — não há *ownership* de shard por réplica como no Scheduler (009); o `shard` é usado para **particionamento físico de dados** (PostgreSQL/HNSW/AGE), não para afinidade de réplica de serviço. |
| Distribuição entre zonas | Réplicas **DEVEM** ser distribuídas em ≥ 2 zonas de disponibilidade (AZ) quando a infraestrutura subjacente suportar, para atender a NFR-005 (disponibilidade ≥ 99,95%). |
| Rebalanceamento de shards físicos (mudança de `N`) | Mudança de `N` de partição de dados exige migração coordenada (ver `Scalability.md` §3.3 e §9 deste documento); **não** afeta a topologia de réplicas do serviço, apenas o particionamento físico nos backends. |

```
   requests (qualquer tenant/agente) ─▶ [réplica A, B, C — todas equivalentes]
                                              │
                                   shard = hash(tenant_id, agent_id) mod N
                                              │
                     ┌────────────────────────┼────────────────────────┐
                     ▼                        ▼                        ▼
              PG partição 0..k          HNSW partição/tenant      AGE subgrafo/tenant
```

---

## 6. Dependências Externas

| Dependência | Protocolo | Criticidade | Comportamento sob indisponibilidade |
|--------------|-----------|--------------|----------------------------------------|
| PostgreSQL (+pgvector +Apache AGE) | TLS + client cert | **Crítica** (fonte da verdade) | Mutações são rejeitadas (`AIOS-MEM-0031`, 503); leituras servidas por réplica, se disponível; ver `FailureRecovery.md` F2. |
| Redis (`WorkingMemoryStore`, `IdempotencyStore`, `QuotaManager`, locks) | TLS + AUTH | **Alta** (não bloqueante para camadas duráveis) | *Fail-open* para camadas duráveis (PG); Working reconstruível (NFR-008); mutações sem `Idempotency-Key` validada são rejeitadas — nunca contornadas (`FailureRecovery.md` F1). |
| MinIO (`BlobStoreAdapter`) | TLS + chave de acesso S3 | **Média** | Conteúdo permanece inline se ≤ limite; externalização represada e reintentada (`FailureRecovery.md` F4). |
| NATS/JetStream | TLS | **Alta** (garantia de evento) | Eventos retidos no `OutboxEvent` (`status=pending`) até reconexão; `OutboxPublisher` reenvia *at-least-once*; decisões continuam sendo aceitas e persistidas (`FailureRecovery.md` F8). |
| `022-Policy` (PDP) | gRPC + mTLS | **Crítica** para toda operação | *Fail-closed*: toda operação negada (`AIOS-MEM-0004`), ver `Security.md` §3.3. |
| `017-Model-Router` (embeddings) | gRPC + mTLS | **Alta** (não bloqueante) | Item persiste como `INGESTED` sem embedding; *backfill* assíncrono (`FailureRecovery.md` F3). |
| `026-Cost-Optimizer` | gRPC + mTLS (eventos) | **Baixa** (não bloqueante) | Reporte de consumo atrasado; não bloqueia `remember`/`recall`. |
| `023-Learning` | Eventos NATS | **Média** (dirige consolidação) | Ausência de eventos apenas atrasa consolidação dirigida; `RetentionScheduler` mantém consolidação por `threshold`/agendada. |
| `011-Context` | gRPC + mTLS | **Alta** (colaboração de recall) | Falha impede recall seletivo colaborativo; `recall` direto via API continua funcional. |
| Gateway (YARP) | HTTPS/mTLS | **Crítica** para tráfego administrativo externo | Sem Gateway, tráfego REST administrativo não alcança o `MemoryService`; tráfego gRPC interno direto (Runtime/Context/Learning) não é afetado. |
| `025-Audit` | Eventos | **Alta** (não bloqueante) | Auditoria é assíncrona via evento; indisponibilidade de `025` não bloqueia a operação, mas **DEVE** gerar alerta (gap de auditoria é risco de conformidade). |

---

## 7. Health e Readiness

| Endpoint | Verifica | Uso |
|----------|----------|-----|
| `GET /healthz` | Processo vivo (*liveness*): laços internos (Outbox relay, RetentionScheduler) responsivos, sem *deadlock*. **NÃO** verifica dependências externas. | Reinício automático do container se falhar repetidamente. |
| `GET /readyz` | Prontidão (*readiness*): conectividade com PostgreSQL (ping + migração aplicada), NATS (conexão ativa), e circuito do `PolicyClient` não aberto de forma permanente. | Remoção temporária do pool de réplicas do Gateway/roteamento gRPC se falhar — réplica não recebe novo tráfego, mas não é reiniciada. |
| `GET /startupz` | *Bootstrap* concluído: configuração carregada, migrações de schema aplicadas, índice HNSW pronto para consulta. | Atrasa avaliação de *liveness* durante o *warm-up* inicial. |
| `GET /metrics` | Exposição Prometheus (`aios_memory_*`, ver `Metrics.md`). | *Scrape* periódico pela pilha de observabilidade (`024-Observability`). |

**Regras normativas:**

- `readyz` **DEVE** retornar `503` se PostgreSQL ou NATS (dependências
  *hard*) estiverem inacessíveis, mesmo que `healthz` continue `200` — o
  processo está vivo, mas não deve receber tráfego de mutação/recall
  durável.
- `readyz` **NÃO DEVE** depender de Redis, MinIO, `017-Model-Router`,
  `026-Cost-Optimizer` ou `023-Learning` estarem disponíveis — a
  indisponibilidade dessas dependências é tratada por degradação graciosa
  por requisição (`Security.md` §8, `FailureRecovery.md` §6), não por
  remoção da réplica do pool.
- Tempo de `start_period` do *healthcheck* **DEVE** acomodar o *warm-up* de
  *pools* de conexão (PostgreSQL/Redis/MinIO/NATS) e carregamento de cache
  de política/capacidade, tipicamente 15–20 s.

---

## 8. Estratégia de Rollout

| Estratégia | Aplicação | Critério de avanço/rollback |
|------------|-----------|--------------------------------|
| **Rolling update** (*default*) | Atualizações de *patch*/*minor* sem mudança de *schema* de API ou de contrato de evento. | Substituição gradual de réplicas (uma por vez ou por *batch* de 25%), aguardando `readyz=200` antes de prosseguir; *rollback* automático se taxa de erro (`5xx`/`AIOS-MEM-0060`) ultrapassar limiar durante o *rollout*. |
| **Blue-green** | Mudanças de *major* de API (nova versão de `aios.memory.v2`) ou migração de *schema* não aditiva. | Nova geração de réplicas (`v2`) sobe em paralelo à atual (`v1`); tráfego migra via Gateway/roteamento gRPC após validação de *smoke tests*; `v1` mantida por período de coexistência (RFC-0001 §5.7: ≥ 2 *majors*). |
| **Canary** | Mudanças de alto risco em `ISchedulingPolicy`-equivalente de recall (troca de estratégia de fusão `rrf`↔`weighted`, ADR-0104) ou em parâmetros de decaimento/consolidação. | Pequena fração de tráfego (ex.: um tenant piloto) roteada à nova política via configuração por tenant (`memory.recall.fusion`); Memory Recall Rate (NFR-002) comparado à *baseline* antes de expandir. |

**Regras de compatibilidade:**

- Migrações de *schema* PostgreSQL **DEVEM** ser aditivas e retrocompatíveis
  durante o *rollout* (coluna nova *nullable*, sem `DROP`/`RENAME`
  destrutivo na mesma *release*) — ver `Database.md` para a estratégia de
  migração completa.
- Mudança incompatível de API (REST/gRPC) **DEVE** incrementar a versão
  *major* (`/v2/memory`, pacote `aios.memory.v2`) e coexistir com a versão
  anterior por ≥ 2 *majors* (RFC-0001 §5.7), nunca substituição *in place*.
- Alteração de `memory.hnsw.m`/`memory.embedding.dim` **NÃO É**
  recarregável em *runtime* (§8 do brief) — exige reindexação online (§12) e
  **NÃO DEVE** ser combinada com o mesmo passo de um *rollout* de código
  (janela de manutenção dedicada).
- Nenhuma *release* **DEVE** ser promovida a 100% do tráfego sem que os
  SLOs de `NonFunctionalRequirements.md` (latência, Memory Recall Rate,
  taxa de erro) permaneçam dentro da meta durante a janela de *canary*/
  *blue-green*.

Sequência de *deploy* padrão:

```
1. aplicar migração DB (expand)          → compatível com versão de código atual
2. rolling update memory-service         → maxUnavailable=0, readiness drena réplica
3. verificar SLI (recall_rate, p99, erros) → gate de qualidade (Monitoring.md)
4. aplicar contract (remoções tardias)   → somente após todas as réplicas na nova versão
```

---

## 9. Escalonamento

| Sinal | Ação | Referência |
|-------|------|--------------|
| CPU média de réplica > 70% por 5 min | Adicionar réplica. | Autoescalonamento horizontal padrão. |
| `aios_memory_remember_duration_ms`/`aios_memory_recall_duration_ms` p99 acima da meta (NFR-001/003) | Adicionar réplica; investigar dependência lenta (PDP/Model Router/PostgreSQL). | `Scalability.md` §7 (orçamento de latência). |
| `aios_memory_inflight` próximo de `memory.backpressure.max_inflight` | Adicionar réplica; revisar limite de *backpressure*. | `Scalability.md` §6 (backpressure). |
| `aios_memory_outbox_pending` crescente | Investigar `OutboxPublisher`/conectividade NATS; aumentar paralelismo do relay. | `FailureRecovery.md` F8. |
| `aios_memory_usage_ratio` próximo de 1,0 por tenant | Acionar poda (`ForgettingEngine`); revisar cotas com `026-Cost-Optimizer`. | `Scalability.md` §6, NFR-009. |
| CPU média de réplica < 30% por 15 min | Remover réplica (respeitando mínimo de 3 em produção). | Autoescalonamento horizontal padrão. |

O modelo detalhado de escala (sharding, particionamento vetorial/temporal,
limites teóricos de *throughput* e caminho para 10⁸ vetores/tenant) é
especificado em `./Scalability.md` — este documento cobre apenas os
gatilhos operacionais de infraestrutura.

---

## 10. Configuração de Rede e Gateway (YARP)

| Aspecto | Especificação |
|---------|-----------------|
| Roteamento | YARP roteia `/v1/memory/*` (REST administrativo) ao pool de réplicas do `MemoryService` via *round-robin* com *health-aware* (remove réplicas com `readyz` falho). |
| Tráfego gRPC interno | Não passa pelo YARP externo; Agent Runtime (007, via Kernel 006), Context Manager (011) e Learning (023) chamam diretamente via mTLS ponto-a-ponto ou *service mesh sidecar* (conforme `028`). |
| Rate-limit de borda | O Gateway aplica um *rate-limit coarse* por tenant/IP como primeira linha de defesa para tráfego REST administrativo; o *rate-limit* fino de admissão (`QuotaManager`, `memory.backpressure.max_inflight`) é responsabilidade do `MemoryService` — ambos coexistem em camadas (*defense in depth*). |
| Terminação TLS | TLS externo terminado no Gateway para tráfego REST; tráfego Gateway→`MemoryService` e gRPC interno direto são mTLS (não HTTP puro). |
| Cabeçalhos propagados | `traceparent`, `X-AIOS-Tenant`, `X-AIOS-Agent`, `Idempotency-Key`, `X-AIOS-Api-Version` (RFC-0001 §5.6) — o Gateway **DEVE** preservar/gerar esses cabeçalhos antes de encaminhar ao `MemoryService`. |
| `Retry-After` | Respostas `AIOS-MEM-0010`/`0011` (cota/*backpressure*) **DEVEM** propagar `retryAfterMs`/`Retry-After` de ponta a ponta através do Gateway sem serem removidos por *middleware* intermediário. |

---

## 11. Ambientes

| Ambiente | Réplicas mínimas | Dados | Uso |
|----------|--------------------|-------|-----|
| `dev` | 1 | Sintéticos, resetáveis | Desenvolvimento local (Docker Compose, §4). |
| `staging` | 3 | Mascarados/sintéticos representativos de produção | Validação de *rollout*, testes de carga (`Benchmark.md`), *chaos drills* (`FailureRecovery.md`), testes de reindexação HNSW e migração de *schema*. |
| `production` | ≥ 3, distribuídas em ≥ 2 AZ | Reais, sob RLS/mTLS/*backup* completo | Operação real; sujeito a todos os SLOs de `NonFunctionalRequirements.md`. |

---

## 12. Migrações de Banco de Dados e Reindexação

- Migrações de *schema* **DEVEM** ser versionadas e aplicadas por
  ferramenta de migração declarativa (ex.: EF Core Migrations / Flyway)
  executada como um passo de *pre-deploy* separado do *rollout* de réplicas
  da aplicação.
- Toda migração **DEVE** ser testada quanto a compatibilidade com a versão
  de código **anterior** (para suportar *rolling update* sem *downtime*)
  antes de ser aplicada em produção; detalhes de DDL, índices,
  particionamento e retenção estão em `./Database.md`.
- **Reindexação HNSW** (mudança de `memory.hnsw.m`/`memory.hnsw.ef_construction`
  ou de `memory.embedding.dim`) **DEVE** ser executada como operação
  **online**, sem *downtime* de recall (índice novo construído em paralelo
  e trocado atomicamente por partição/tenant), em janela de manutenção
  dedicada e **nunca** no mesmo passo de um *rollout* de código
  (`Scalability.md` §3.3).
- Alterações no grafo Apache AGE (evolução de tipos de aresta/propriedades)
  **DEVEM** seguir a mesma disciplina de expansão aditiva antes de contração
  destrutiva.
- Migrações de *schema* de blob/*bucket* MinIO (ex.: reorganização de
  prefixo) **DEVEM** preservar `content_hash` como identidade estável,
  nunca invalidando referências (`content_ref`) já persistidas em
  `MemoryItem`.

---

## 13. Referências

- Arquitetura geral e diagrama de componentes: `./Architecture.md`, `./_DESIGN_BRIEF.md` §2.
- Modelo de escala, sharding e particionamento: `./Scalability.md`.
- Modos de falha e recuperação: `./FailureRecovery.md`.
- Postura de segurança e *hardening*: `./Security.md`.
- Modelo físico, DDL, índices e particionamento: `./Database.md`.
- Contratos centrais (correlação, versionamento de API): `../003-RFC/RFC-0001-Architecture-Baseline.md` §5.6–§5.7.
- Alta disponibilidade e infraestrutura de dados: `../005-Database/`, `../027-Cluster/`.
- Gateway e infraestrutura de borda: `../028-Deployment/`.
- Observabilidade (métricas/*probes*): `../024-Observability/`.
- Glossário: `../040-Glossary/Glossary.md` (termos: Hot State, Cold Agent, Shard, AOT, HA).
