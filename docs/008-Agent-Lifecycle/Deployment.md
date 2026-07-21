---
Documento: Deployment
Módulo: 008-Agent-Lifecycle
Status: Draft
Versão: 0.1
Última atualização: 2026-07-20
Responsável (RACI-A): Arquiteto do Módulo 008 — Agent Lifecycle
ADRs relacionados: ADR-0080 (Máquina de estados canônica), ADR-0081 (Cold Agent Hibernation em MinIO+PostgreSQL), ADR-0083 (WarmPool/materialização < 250 ms), ADR-0084 (Lease + fencing token), ADR-0086 (Event sourcing + Outbox)
RFCs relacionados: RFC-0001 (baseline)
Depende de: 001-Architecture, 006-Kernel, 007-Agent-Runtime, 009-Scheduler, 020-Communication, 027-Cluster, 028 (Gateway/Infra — quando publicado)
---

# 008-Agent-Lifecycle — Deployment

> **Escopo.** Este documento especifica como o serviço Agent Lifecycle (008)
> é empacotado, implantado, escalado e operado: topologia de containers,
> recursos por réplica, dependências externas, health/readiness, estratégia
> de rollout e integração com o Gateway (YARP). Não redefine os contratos
> centrais de RFC-0001 nem os modos de falha (ver `FailureRecovery.md`) ou o
> modelo de escala (ver `Scalability.md`) — este documento foca em **como o
> serviço é colocado em produção**, referenciando aqueles onde necessário.

---

## 1. Visão Geral da Topologia

O serviço Agent Lifecycle é um processo **stateless** (.NET 10, AOT quando
aplicável), implantado como múltiplas réplicas idênticas atrás do API Gateway
(YARP). Todo estado autoritativo é externalizado em PostgreSQL (ACB,
transições, outbox, índice de checkpoint), Redis (lease distribuído, ACB
quente/projeção) e MinIO (blobs de snapshot/checkpoint). Toda comunicação
assíncrona passa pelo NATS/JetStream.

```
                                   Internet / Rede do Tenant
                                             │  HTTPS (TLS 1.3)
                                             ▼
                              ┌───────────────────────────────┐
                              │   API Gateway (YARP) — 028     │
                              │  - AuthN (OIDC) · Rate-limit    │
                              │  - Roteamento /v1/agents/** →   │
                              │    pool de réplicas 008         │
                              └───────────────┬─────────────────┘
                                               │ mTLS (interno)
                    ┌──────────────────────────┼──────────────────────────┐
                    ▼                          ▼                          ▼
           ┌──────────────────┐      ┌──────────────────┐      ┌──────────────────┐
           │ Lifecycle réplica 1│    │ Lifecycle réplica 2│    │ Lifecycle réplica N│
           │  (008 · .NET 10)  │      │  (008 · .NET 10)  │      │  (008 · .NET 10)  │
           └─────────┬─────────┘      └─────────┬─────────┘      └─────────┬─────────┘
                     │                          │                          │
     ┌───────────────┼──────────────────────────┼──────────────────────────┼────────────┐
     ▼               ▼                          ▼                          ▼            ▼
PostgreSQL HA    Redis Cluster              MinIO Cluster           NATS/JetStream   Dependências
(ACB, transições, (lease/fencing,           (checkpoints/           Cluster           externas:
 outbox, ckpt idx, RLS) ACB hot)              snapshots, por        (lifecycle.*,      006-Kernel
                                               tenant)               consumo 009/007/  009-Scheduler
                                                                      006/014/027)      007-Runtime
                                                                                         027-Cluster
                                                                                         022-Policy
```

Todas as réplicas do módulo 008 são **equivalentes e sem afinidade
obrigatória** (afinidade de shard é otimização de cache local, não requisito
de correção — ver `_DESIGN_BRIEF.md` §10 e `Scalability.md`).

---

## 2. Requisitos de Infraestrutura por Réplica

| Recurso | Valor recomendado (baseline) | Observação |
|---------|-------------------------------|-------------|
| CPU | 2 vCPU (request), 4 vCPU (limit) | Caminho quente é CPU-bound (validação de FSM, serialização de comando, cache de decisão PEP); serialização/compressão de checkpoint (`zstd`) é intensiva de CPU sob rajada de hibernação. |
| Memória | 1,5 GiB (request), 3 GiB (limit) | ACBs *hot* residem em Redis, não em heap do serviço; heap cobre conexões, buffers de (de)serialização de checkpoint em trânsito (limitado por `checkpoint.max_working_memory_mib`, default 256 MiB) e cache local de decisão PEP. |
| GPU | Não aplicável | O módulo 008 não executa inferência nem loop cognitivo (N1 do brief). |
| Disco | Efêmero, ≤ 2 GiB | Sem estado persistente local; usado como *scratch* transitório de (de)compressão de checkpoint antes do upload/download ao MinIO, e para logs de curto prazo antes do envio a Seq/OTel collector. |
| Rede | Baixa latência (mesma região/AZ que PostgreSQL/Redis/NATS/MinIO) | Latência de rede às dependências é o principal risco ao SLO de `spawn`/`wake` p99 ≤ 250 ms (NFR-001); throughput de rede ao MinIO é crítico para `checkpoint`/`restore` (NFR-003). |

> Estes valores são **baseline de dimensionamento inicial**; o dimensionamento
> definitivo por ambiente é validado em `Benchmark.md` e ajustado por
> autoescalonamento (§9).

---

## 3. Containerização

| Aspecto | Especificação |
|---------|-----------------|
| Runtime | .NET 10, imagem de execução mínima (distroless-equivalente). |
| Compilação | AOT (*Ahead-of-Time*) quando suportado pela plataforma alvo, para reduzir tempo de start-up e superfície de runtime (Glossário: AOT). |
| Usuário | Não-root, UID/GID dedicados, sem capacidades Linux adicionais. |
| Sistema de arquivos | Raiz somente-leitura; `/tmp` efêmero para *scratch* de (de)compressão de checkpoint. |
| Variáveis de ambiente | Prefixo `AIOS_LIFECYCLE_` para todas as chaves de configuração (ver `_DESIGN_BRIEF.md` §8); segredos **NÃO DEVEM** ser passados como variável de ambiente em texto puro — usar montagem de arquivo a partir do cofre de segredos (`Security.md` §6). |
| Portas expostas | `8080` (REST, atrás do Gateway), `5001` (gRPC interno `aios.lifecycle.v1`, mTLS), `9090` (métricas Prometheus, rede interna de observabilidade). |
| Tags de imagem | Semânticas (`008-agent-lifecycle:1.0.0`), imutáveis; **NÃO DEVEM** ser sobrescritas (`:latest` proibido em produção). |

---

## 4. Topologia de Implantação — Docker Compose (ambiente de desenvolvimento/integração)

```yaml
services:
  agent-lifecycle:
    image: registry.internal/aios/008-agent-lifecycle:${LIFECYCLE_VERSION}
    deploy:
      replicas: 3
      resources:
        limits: { cpus: "4", memory: 3g }
        reservations: { cpus: "2", memory: 1.5g }
      restart_policy: { condition: on-failure, max_attempts: 5 }
    environment:
      - AIOS_LIFECYCLE_SPAWN_TARGET_P99_MS=250
      - AIOS_LIFECYCLE_HIBERNATION_IDLE_TTL_S=300
      - AIOS_LIFECYCLE_CHECKPOINT_CODEC=msgpack+zstd
      - AIOS_LIFECYCLE_CHECKPOINT_ENCRYPTION=aes-256-gcm
      - AIOS_LIFECYCLE_LEASE_TTL_MS=5000
    secrets:
      - lifecycle_pg_credentials
      - lifecycle_redis_credentials
      - lifecycle_minio_credentials
      - lifecycle_nats_account_jwt
      - lifecycle_checkpoint_kms_key_ref
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
      minio: { condition: service_healthy }
    networks: [aios-control-plane]

  postgres:
    image: pgvector/pgvector:pg16
    environment:
      - POSTGRES_DB=aios_lifecycle
    volumes: [pg-lifecycle-data:/var/lib/postgresql/data]
    networks: [aios-control-plane]

  redis:
    image: redis:7-alpine
    command: ["redis-server", "--requirepass", "${REDIS_PASSWORD}"]
    networks: [aios-control-plane]

  minio:
    image: minio/minio:latest
    command: ["server", "/data", "--console-address", ":9001"]
    environment:
      - MINIO_ROOT_USER=${MINIO_ROOT_USER}
      - MINIO_ROOT_PASSWORD=${MINIO_ROOT_PASSWORD}
    volumes: [minio-lifecycle-data:/data]
    networks: [aios-control-plane]

  nats:
    image: nats:2-alpine
    command: ["-js", "-sd", "/data"]
    volumes: [nats-lifecycle-data:/data]
    networks: [aios-control-plane]

networks:
  aios-control-plane: { driver: bridge }

volumes:
  pg-lifecycle-data:
  minio-lifecycle-data:
  nats-lifecycle-data:

secrets:
  lifecycle_pg_credentials: { external: true }
  lifecycle_redis_credentials: { external: true }
  lifecycle_minio_credentials: { external: true }
  lifecycle_nats_account_jwt: { external: true }
  lifecycle_checkpoint_kms_key_ref: { external: true }
```

> Em ambientes de produção, PostgreSQL, Redis, MinIO e NATS **DEVEM** ser
> clusters gerenciados de alta disponibilidade compartilhados/dedicados (ver
> `027-Cluster`), não containers efêmeros do Compose — o exemplo acima é para
> desenvolvimento e testes de integração locais.

---

## 5. Réplicas e Afinidade de Shard

| Aspecto | Regra |
|---------|-------|
| Contagem mínima de réplicas em produção | **3** (tolerância a falha de 1 zona sem perda de capacidade de atendimento). |
| Escalonamento horizontal | Baseado em CPU (%), profundidade de fila de materialização/hibernação e `aios_lifecycle_spawn_duration_ms`/`aios_lifecycle_transition_duration_ms` p99 (ver §9). |
| Afinidade de shard | **PODE** ser usada como otimização (réplica prioriza cache local para um subconjunto de `placement_shard`), mas **NÃO É** requisito de correção — qualquer réplica **DEVE** poder atender qualquer `tenant`/agente, buscando o ACB no PostgreSQL/Redis se não estiver em cache local (`_DESIGN_BRIEF.md` §10). |
| Distribuição entre zonas | Réplicas **DEVEM** ser distribuídas em ≥ 2 zonas de disponibilidade (AZ) quando a infraestrutura subjacente suportar, para atender a NFR-007 (disponibilidade ≥ 99,95%). |

---

## 6. Dependências Externas

| Dependência | Protocolo | Criticidade | Comportamento sob indisponibilidade |
|--------------|-----------|--------------|----------------------------------------|
| PostgreSQL (fonte da verdade do ACB, transições, outbox, índice de checkpoint) | TLS + client cert | **Crítica** | Mutações de ciclo de vida falham; leituras cacheadas em Redis podem continuar por curto prazo. Ver `FailureRecovery.md`. |
| Redis (lease/fencing token, ACB quente) | TLS + AUTH | **Crítica** | `LeaseManager` não consegue serializar transições com segurança; módulo **DEVE** recusar novas mutações mutuamente exclusivas (fail-closed) até recuperação. |
| MinIO (blobs de checkpoint/snapshot) | TLS + credencial S3 | **Alta** (bloqueia hibernação/migração/restore) | `hibernate`/`checkpoint`/`restore`/`migrate` falham com `AIOS-LIFECYCLE-0014`; agentes já quentes continuam operando (`Running`/`Suspended` não afetados). |
| NATS/JetStream (Outbox relay, eventos consumidos) | TLS | **Alta** | Eventos ficam retidos no `LifecycleOutbox` até reconexão; transições continuam sendo aceitas e persistidas (não bloqueante). |
| `022-Policy` (PDP) | gRPC + mTLS | **Crítica** para mutações | Fail-closed nega mutações (`Security.md` §3.3). |
| `009-Scheduler` | gRPC + mTLS | **Alta** (bloqueia `spawn`/`wake`/`migrate`) | Falha com `AIOS-LIFECYCLE-0005`/timeout; demais transições não afetadas. |
| `007-Agent-Runtime` | gRPC + mTLS | **Alta** (bloqueia materialização) | `spawn`/`wake` falham com `AIOS-LIFECYCLE-0006` (timeout de materialização); `suspend`/`terminate` de agentes já materializados não afetados. |
| `006-Kernel` | gRPC + mTLS | **Alta** (origem do `acb.created`) | Sem novos ACBs sendo criados; agentes já geridos pelo 008 continuam sua FSM normalmente. |
| `010-Memory` (ponteiro de working memory) | gRPC + mTLS | **Média** | `checkpoint`/`restore` de working memory podem degradar; ACB e metadados de ciclo de vida continuam consistentes. |
| `027-Cluster` (placement de migração) | gRPC + mTLS | **Média** (bloqueia `migrate`) | `migrate` falha com `AIOS-LIFECYCLE-0010`; agente permanece na origem, sem perda de trabalho. |
| Gateway (YARP) | HTTPS/mTLS | **Crítica** para tráfego externo | Sem Gateway, tráfego externo não alcança o módulo; gRPC interno direto não é afetado. |
| `025-Audit` | gRPC/eventos | **Alta** (não bloqueante) | Auditoria é assíncrona via evento; indisponibilidade não bloqueia a transição, mas **DEVE** gerar alerta (gap de auditoria). |

---

## 7. Health e Readiness

| Endpoint | Verifica | Uso |
|----------|----------|-----|
| `GET /healthz` | Processo vivo (liveness): loop de eventos responsivo, sem deadlock. **NÃO** verifica dependências externas. | Reinício automático do container se falhar repetidamente. |
| `GET /readyz` | Prontidão (readiness): conectividade com PostgreSQL (ping), Redis (ping), MinIO (ping de bucket), NATS (conexão ativa), e circuito do `PolicyClient` não aberto de forma permanente. | Remoção temporária do pool de réplicas do Gateway se falhar — réplica não recebe novo tráfego, mas não é reiniciada. |
| `GET /metrics` | Exposição Prometheus (`aios_lifecycle_*`, ver `Metrics.md`). | Scrape periódico pela pilha de observabilidade (`024-Observability`). |

**Regras normativas:**

- `readyz` **DEVE** retornar `503` se qualquer dependência crítica
  (PostgreSQL, Redis) estiver inacessível, mesmo que `healthz` continue `200`
  — o processo está vivo, mas não deve receber tráfego.
- `readyz` **DEVERIA** refletir degradação de MinIO (dependência alta, não
  crítica) como um sinal de saúde reduzida sem necessariamente remover a
  réplica do pool — mutações que não dependem de storage (`suspend`/`resume`
  de agentes já quentes) **DEVEM** continuar operando.
- `readyz` **NÃO DEVE** depender do PDP (`022-Policy`) estar disponível — a
  indisponibilidade do PDP é tratada por *fail-closed* por comando
  (`Security.md` §3.3), não por remoção da réplica do pool (leituras não
  privilegiadas devem continuar funcionando).
- Tempo de `start_period` do healthcheck **DEVE** acomodar o *warm-up* de
  pools de conexão (PostgreSQL/Redis/MinIO/NATS) e o *rehydrate* inicial do
  `WarmPoolManager`, tipicamente 15–30 s.

---

## 8. Estratégia de Rollout

| Estratégia | Aplicação | Critério de avanço/rollback |
|------------|-----------|--------------------------------|
| **Rolling update** (default) | Atualizações de patch/minor sem mudança de schema de API ou da FSM. | Substituição gradual de réplicas (uma por vez ou por *batch* de 25%), aguardando `readyz=200` antes de prosseguir; rollback automático se taxa de erro (`5xx`)/`AIOS-LIFECYCLE-*` ultrapassar limiar durante o rollout. |
| **Blue-green** | Mudanças de *major* de API (nova versão do pacote `aios.lifecycle.v2`), mudança no formato/codec de checkpoint (ADR-0082), ou migração de schema de banco não aditiva. | Nova geração de réplicas (`v2`) sobe em paralelo à atual (`v1`); tráfego migra via Gateway após validação de *smoke tests* (incluindo `checkpoint`/`restore` de ida e volta); `v1` mantida por período de coexistência (RFC-0001 §5.7: ≥ 2 majors). |
| **Canary** | Mudanças de alto risco na `StateMachineEngine`, no `LeaseManager` (fencing) ou nas políticas de hibernação/WarmPool. | Pequena fração de tráfego (ex.: 5%) roteada à nova versão via YARP; métricas de erro/latência (incluindo p99 de `spawn`/`checkpoint`) comparadas à baseline antes de expandir. |

**Regras de compatibilidade:**

- Migrações de schema PostgreSQL **DEVEM** ser aditivas e retrocompatíveis
  durante o rollout (coluna nova nullable, sem `DROP`/`RENAME` destrutivo na
  mesma release) — ver `Database.md` para a estratégia completa.
- Mudança incompatível de API (REST/gRPC) **DEVE** incrementar a versão
  major (`/v2/agents`, pacote `aios.lifecycle.v2`) e coexistir com a versão
  anterior por ≥ 2 majors (RFC-0001 §5.7), nunca substituição *in place*.
- Mudança no **codec de checkpoint** (`checkpoint.codec`, não recarregável
  em runtime — `_DESIGN_BRIEF.md` §8) **DEVE** ser tratada como mudança
  major: o `CheckpointService` **DEVE** conseguir **ler** checkpoints no
  codec anterior (compatibilidade retroativa de leitura) por, no mínimo, o
  período de retenção de checkpoints vigente (`retention.checkpoint_ttl_days`),
  mesmo após passar a **escrever** no novo codec.
- Nenhuma release **DEVE** ser promovida a 100% do tráfego sem que os SLOs
  de `NonFunctionalRequirements.md` (latência de spawn/checkpoint/migração,
  taxa de erro) permaneçam dentro da meta durante a janela de canary/blue-green.

---

## 9. Escalonamento

| Sinal | Ação | Referência |
|-------|------|--------------|
| CPU média de réplica > 70% por 5 min | Adicionar réplica. | Autoescalonamento horizontal padrão. |
| `aios_lifecycle_spawn_duration_ms{quantile="0.99"}` acima da meta de NFR-001 | Adicionar réplica; investigar `WarmPoolManager`/dependência lenta (Scheduler/Runtime). | `Scalability.md` §7. |
| `aios_lifecycle_checkpoint_duration_ms{quantile="0.99"}` acima da meta de NFR-003 | Adicionar réplica; investigar throughput de rede ao MinIO. | `Scalability.md` §7. |
| Fila/backlog de hibernação (agentes elegíveis a `hibernate` acumulando) | Aumentar paralelismo do `HibernationController` ou investigar pressão de storage. | `FailureRecovery.md`. |
| Fila/backlog do Outbox (`LifecycleOutbox` com `published=false` crescente) | Aumentar paralelismo do relay ou investigar partição do NATS. | `FailureRecovery.md`. |
| CPU média de réplica < 30% por 15 min | Remover réplica (respeitando mínimo de 3 em produção). | Autoescalonamento horizontal padrão. |

O modelo detalhado de escala (sharding, hot/cold state, WarmPool, limites
teóricos de agentes e transições/s) é especificado em `./Scalability.md` —
este documento cobre apenas os gatilhos operacionais de infraestrutura.

---

## 10. Configuração de Rede e Gateway (YARP)

| Aspecto | Especificação |
|---------|-----------------|
| Roteamento | YARP roteia `/v1/agents/**` (REST externo, incluindo subrecursos `suspend`/`resume`/`hibernate`/`wake`/`migrate`/`checkpoint`/`restore`) ao pool de réplicas do módulo 008 via *round-robin* com *health-aware* (remove réplicas com `readyz` falho). |
| Rate-limit de borda | O Gateway aplica um rate-limit *coarse* por tenant/IP como primeira linha de defesa; o rate-limit fino de admissão de `spawn`/`wake` é responsabilidade conjunta do módulo 008 e do Scheduler (009) — ambos coexistem em camadas (*defense in depth*). |
| Terminação TLS | TLS externo terminado no Gateway; tráfego Gateway→008 é mTLS interno (não é HTTP puro). |
| Cabeçalhos propagados | `traceparent`, `X-AIOS-Tenant`, `X-AIOS-Agent`, `Idempotency-Key`, `X-AIOS-Api-Version` (RFC-0001 §5.6) — o Gateway **DEVE** preservar/gerar esses cabeçalhos antes de encaminhar ao módulo 008. |
| gRPC interno | Não passa pelo YARP externo; consumido diretamente pela malha de serviços internos (006, 007, 009, 027) via mTLS ponto-a-ponto. |

---

## 11. Ambientes

| Ambiente | Réplicas mínimas | Dados | Uso |
|----------|--------------------|-------|-----|
| `dev` | 1 | Sintéticos, resetáveis | Desenvolvimento local (Docker Compose, §4). |
| `staging` | 3 | Mascarados/sintéticos representativos de produção | Validação de rollout, testes de carga (`Benchmark.md`), *chaos drills* (`FailureRecovery.md`), incluindo simulação de storm de hibernação/wake. |
| `production` | ≥ 3, distribuídas em ≥ 2 AZ | Reais, sob RLS/mTLS/cifragem de checkpoint completa | Operação real; sujeito a todos os SLOs de `NonFunctionalRequirements.md`. |

---

## 12. Migrações de Banco de Dados

- Migrações de schema **DEVEM** ser versionadas e aplicadas por ferramenta de
  migração declarativa (ex.: EF Core Migrations / Flyway) executada como um
  passo de *pre-deploy* separado do rollout de réplicas da aplicação.
- Toda migração **DEVE** ser testada quanto a compatibilidade com a versão de
  código **anterior** (para suportar rolling update sem downtime) antes de
  ser aplicada em produção.
- Migrações que alteram particionamento por `tenant_id`/`placement_shard`
  **DEVEM** ser executadas como operação planejada e observável, nunca como
  efeito colateral silencioso de um deploy de aplicação.
- Detalhes de DDL, índices e particionamento: `./Database.md`.

---

## 13. Referências

- Arquitetura geral e diagrama de componentes: `./_DESIGN_BRIEF.md` §2.
- Modelo de escala e sharding: `./Scalability.md`.
- Modos de falha e recuperação: `./FailureRecovery.md`.
- Postura de segurança e hardening: `./Security.md`.
- Contratos centrais (correlação, versionamento de API): `../003-RFC/RFC-0001-Architecture-Baseline.md` §5.6–§5.7.
- Cluster/HA/DR (transversal): `../027-Cluster/`.
- Gateway e infraestrutura de borda: `../028-Gateway-Infra/` (quando publicado).
- Observabilidade (métricas/probes): `../024-Observability/`.
- Runtime supervisionado: `../007-Agent-Runtime/`.
