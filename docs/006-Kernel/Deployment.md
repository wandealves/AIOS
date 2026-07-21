---
Documento: Deployment
Módulo: 006-Kernel
Status: Draft
Versão: 0.1
Última atualização: 2026-07-20
Responsável (RACI-A): Arquiteto do Módulo 006-Kernel
ADRs relacionados: ADR-0002 (Microsserviços Control/Data Plane), ADR-0003 (.NET Control / Python Runtime), ADR-0004 (NATS como Barramento), ADR-0006 (Redis Estado Quente), ADR-0061 (ACB & OCC), ADR-0065 (Sharding do ACB)
RFCs relacionados: RFC-0001 (baseline)
Depende de: 001-Architecture, 009-Scheduler, 008-Lifecycle, 022-Policy, 020-Communication, 028 (Gateway/Infra), 027 (HA/DR)
---

# 006-Kernel — Deployment

> **Escopo.** Este documento especifica como o Kernel Service (006) é
> empacotado, implantado, escalado e operado: topologia de containers, recursos
> por réplica, dependências externas, health/readiness, estratégia de rollout e
> integração com o Gateway (YARP). Não redefine os contratos centrais de RFC-0001
> nem os modos de falha (ver `FailureRecovery.md`) ou o modelo de escala
> (ver `Scalability.md`) — este documento foca em **como o serviço é colocado em
> produção**, referenciando aqueles onde necessário.

---

## 1. Visão Geral da Topologia

O Kernel Service é um processo **stateless** (.NET 10, AOT quando aplicável),
implantado como múltiplas réplicas idênticas atrás do API Gateway (YARP). Todo
estado autoritativo é externalizado em PostgreSQL (fonte da verdade) e Redis
(estado quente/cache), e toda comunicação assíncrona passa pelo NATS/JetStream.

```
                                   Internet / Rede do Tenant
                                             │  HTTPS (TLS 1.3)
                                             ▼
                              ┌───────────────────────────────┐
                              │   API Gateway (YARP) — 028     │
                              │  - AuthN (OIDC) · Rate-limit    │
                              │  - Roteamento /v1/kernel/* →    │
                              │    pool de réplicas do Kernel   │
                              └───────────────┬─────────────────┘
                                               │ mTLS (interno)
                    ┌──────────────────────────┼──────────────────────────┐
                    ▼                          ▼                          ▼
           ┌─────────────────┐        ┌─────────────────┐        ┌─────────────────┐
           │ Kernel réplica 1 │        │ Kernel réplica 2 │  ...   │ Kernel réplica N │
           │  (006 · .NET 10) │        │  (006 · .NET 10) │        │  (006 · .NET 10) │
           └────────┬─────────┘        └────────┬─────────┘        └────────┬─────────┘
                    │                            │                            │
        ┌───────────┼────────────────────────────┼────────────────────────────┼──────────┐
        ▼           ▼                            ▼                            ▼          ▼
  PostgreSQL HA   Redis Cluster            NATS/JetStream Cluster        022-Policy   009-Scheduler
 (kernel.*, RLS) (ACB quente, cotas)     (KERNEL_LIFECYCLE/QUOTA/AUDIT)     (PDP)      (admissão)
        │                                                                     │            │
        ▼                                                                     ▼            ▼
  (streaming replication,                                              gRPC+mTLS     gRPC+mTLS
   réplica standby p/ RTO/RPO)                                                            │
                                                                                            ▼
                                                                                     008-Lifecycle
                                                                                  (materialização/MinIO)
```

Todas as réplicas do Kernel são **equivalentes e sem afinidade obrigatória**
(afinidade de shard é otimização de cache local, não requisito de correção —
ver `_DESIGN_BRIEF.md` §10 e `Scalability.md`).

---

## 2. Requisitos de Infraestrutura por Réplica

| Recurso | Valor recomendado (baseline) | Observação |
|---------|-------------------------------|-------------|
| CPU | 2 vCPU (request), 4 vCPU (limit) | Caminho quente é CPU-bound (serialização, validação de schema, cache de decisão). |
| Memória | 1 GiB (request), 2 GiB (limit) | ACBs *hot* residem em Redis, não em heap do Kernel; heap cobre apenas conexões, buffers e cache local de decisão PEP. |
| GPU | Não aplicável | O Kernel não executa inferência; roteamento de modelo é delegado a `017-Model-Router`. |
| Disco | Efêmero, ≤ 1 GiB | Sem estado persistente local; usado apenas para logs de curto prazo antes do envio a Seq/OTel collector. |
| Rede | Baixa latência (mesma região/AZ que PostgreSQL/Redis/NATS) | Latência de rede às dependências é o principal risco ao SLO de `spawn` p99 ≤ 250 ms (NFR-001). |

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
| Variáveis de ambiente | Prefixo `AIOS_KERNEL_` para todas as chaves de configuração (ver `_DESIGN_BRIEF.md` §8); segredos **NÃO DEVEM** ser passados como variável de ambiente em texto puro — usar montagem de arquivo a partir do cofre de segredos (`Security.md` §6). |
| Portas expostas | `8080` (REST, atrás do Gateway), `5001` (gRPC interno, mTLS), `9090` (métricas Prometheus, rede interna de observabilidade). |
| Tags de imagem | Semânticas (`006-kernel:1.4.2`), imutáveis; **NÃO DEVEM** ser sobrescritas (`:latest` proibido em produção). |

---

## 4. Topologia de Implantação — Docker Compose (ambiente de desenvolvimento/integração)

```yaml
services:
  kernel:
    image: registry.internal/aios/006-kernel:${KERNEL_VERSION}
    deploy:
      replicas: 3
      resources:
        limits: { cpus: "4", memory: 2g }
        reservations: { cpus: "2", memory: 1g }
      restart_policy: { condition: on-failure, max_attempts: 5 }
    environment:
      - AIOS_KERNEL_QUOTA_WINDOW=1h
      - AIOS_KERNEL_PEP_FAIL_MODE=closed
      - AIOS_KERNEL_ACB_SHARD_COUNT=64
    secrets:
      - kernel_pg_credentials
      - kernel_redis_credentials
      - kernel_nats_account_jwt
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
      - POSTGRES_DB=aios_kernel
    volumes: [pg-kernel-data:/var/lib/postgresql/data]
    networks: [aios-control-plane]

  redis:
    image: redis:7-alpine
    command: ["redis-server", "--requirepass", "${REDIS_PASSWORD}"]
    networks: [aios-control-plane]

  nats:
    image: nats:2-alpine
    command: ["-js", "-sd", "/data"]
    volumes: [nats-kernel-data:/data]
    networks: [aios-control-plane]

networks:
  aios-control-plane: { driver: bridge }

volumes:
  pg-kernel-data:
  nats-kernel-data:

secrets:
  kernel_pg_credentials: { external: true }
  kernel_redis_credentials: { external: true }
  kernel_nats_account_jwt: { external: true }
```

> Em ambientes de produção, PostgreSQL, Redis e NATS **DEVEM** ser clusters
> gerenciados de alta disponibilidade compartilhados/dedicados (ver `027-HA-DR`),
> não containers efêmeros do Compose — o exemplo acima é para desenvolvimento e
> testes de integração locais.

---

## 5. Réplicas e Afinidade de Shard

| Aspecto | Regra |
|---------|-------|
| Contagem mínima de réplicas em produção | **3** (tolerância a falha de 1 zona sem perda de capacidade de atendimento). |
| Escalonamento horizontal | Baseado em CPU (%), profundidade de fila de syscall e `aios_kernel_syscall_latency_ms` p99 (ver §9). |
| Afinidade de shard | **PODE** ser usada como otimização (réplica prioriza cache local para um subconjunto de `shard_key`), mas **NÃO É** requisito de correção — qualquer réplica **DEVE** poder atender qualquer `tenant`/`agente`, buscando o ACB no PostgreSQL/Redis se não estiver em cache local. |
| Distribuição entre zonas | Réplicas **DEVEM** ser distribuídas em ≥ 2 zonas de disponibilidade (AZ) quando a infraestrutura subjacente suportar, para atender a NFR-005 (disponibilidade ≥ 99,95%). |

---

## 6. Dependências Externas

| Dependência | Protocolo | Criticidade | Comportamento sob indisponibilidade |
|--------------|-----------|--------------|----------------------------------------|
| PostgreSQL (fonte da verdade do ACB/Quota/Outbox) | TLS + client cert | **Crítica** | Kernel não consegue mutar estado; syscalls mutantes falham com `503`/`AIOS-KERNEL-0005`-equivalente; leituras cacheadas em Redis podem continuar por curto prazo. |
| Redis (ACB quente, contadores de cota, lock distribuído) | TLS + AUTH | **Crítica** | Fallback a limites conservadores do PostgreSQL (nega dúvidas); ver `FailureRecovery.md`. |
| NATS/JetStream (Outbox relay, eventos consumidos) | TLS | **Alta** | Eventos ficam retidos no Outbox (`kernel.outbox`) até reconexão; syscalls continuam sendo aceitas e persistidas (não bloqueante). |
| `022-Policy` (PDP) | gRPC + mTLS | **Crítica** para syscalls privilegiadas | `fail_mode=closed` nega syscalls privilegiadas (`Security.md` §3.3). |
| `009-Scheduler` | gRPC + mTLS | **Alta** (bloqueia `spawn`/`resume`) | `spawn`/`resume` falham com `AIOS-KERNEL-0003`; demais syscalls não afetadas. |
| `008-Lifecycle` | gRPC + mTLS | **Alta** (bloqueia materialização) | `spawn` falha em `AIOS-KERNEL-0004` (timeout de boot); `suspend`/`kill` de agentes já `Running` não afetados. |
| `010/011/012/015/017` (brokers de recurso) | gRPC + mTLS | **Média**, isolada por dependência | Circuit breaker isola; apenas a syscall daquele recurso específico falha (`AIOS-KERNEL-0005`). |
| Gateway (YARP) | HTTPS/mTLS | **Crítica** para tráfego externo | Sem Gateway, tráfego externo não alcança o Kernel; tráfego gRPC interno direto não é afetado. |
| `025-Audit` | gRPC/eventos | **Alta** (não bloqueante) | Auditoria é assíncrona via evento; indisponibilidade de `025` não bloqueia a syscall, mas **DEVE** gerar alerta (gap de auditoria é um risco de conformidade). |

---

## 7. Health e Readiness

| Endpoint | Verifica | Uso |
|----------|----------|-----|
| `GET /healthz` | Processo vivo (liveness): loop de eventos responsivo, sem deadlock. **NÃO** verifica dependências externas. | Reinício automático do container se falhar repetidamente. |
| `GET /readyz` | Prontidão (readiness): conectividade com PostgreSQL (ping), Redis (ping), NATS (conexão ativa), e circuito do `PolicyClient` não aberto de forma permanente. | Remoção temporária do pool de réplicas do Gateway se falhar — réplica não recebe novo tráfego, mas não é reiniciada. |
| `GET /metrics` | Exposição Prometheus (`aios_kernel_*`, ver `Metrics.md`). | Scrape periódico pela pilha de observabilidade (`024-Observability`). |

**Regras normativas:**

- `readyz` **DEVE** retornar `503` se qualquer dependência crítica (PostgreSQL,
  Redis) estiver inacessível, mesmo que `healthz` continue `200` — o processo
  está vivo, mas não deve receber tráfego.
- `readyz` **NÃO DEVE** depender do PDP (`022-Policy`) estar disponível — a
  indisponibilidade do PDP é tratada por *fail-closed* por syscall (`Security.md`
  §3.3), não por remoção da réplica do pool (leituras não privilegiadas devem
  continuar funcionando).
- Tempo de `start_period` do healthcheck **DEVE** acomodar o *warm-up* de pools
  de conexão (PostgreSQL/Redis/NATS) e carregamento inicial de *policy bundle*
  em cache, tipicamente 10–20 s.

---

## 8. Estratégia de Rollout

| Estratégia | Aplicação | Critério de avanço/rollback |
|------------|-----------|--------------------------------|
| **Rolling update** (default) | Atualizações de patch/minor sem mudança de schema de API. | Substituição gradual de réplicas (uma por vez ou por *batch* de 25%), aguardando `readyz=200` antes de prosseguir; rollback automático se taxa de erro (`5xx`) ultrapassar limiar durante o rollout. |
| **Blue-green** | Mudanças de *major* de API (nova versão de ABI de syscall, RFC-0006) ou migração de schema de banco não aditiva. | Nova geração de réplicas (`v2`) sobe em paralelo à atual (`v1`); tráfego migra via Gateway após validação de *smoke tests*; `v1` mantida por período de coexistência (RFC-0001 §5.7: ≥ 2 majors). |
| **Canary** | Mudanças de alto risco em lógica de PEP, cotas ou FSM do ACB. | Pequena fração de tráfego (ex.: 5%) roteada à nova versão via YARP; métricas de erro/latência comparadas à baseline antes de expandir. |

**Regras de compatibilidade:**

- Migrações de schema PostgreSQL **DEVEM** ser aditivas e retrocompatíveis
  durante o rollout (coluna nova nullable, sem `DROP`/`RENAME` destrutivo na
  mesma release) — ver `Database.md` para a estratégia de migração completa.
- Mudança incompatível de API (REST/gRPC) **DEVE** incrementar a versão major
  (`/v2/kernel`, pacote `aios.kernel.v2`) e coexistir com a versão anterior por
  ≥ 2 majors (RFC-0001 §5.7), nunca substituição *in place*.
- Nenhuma release **DEVE** ser promovida a 100% do tráfego sem que os SLOs de
  `NonFunctionalRequirements.md` (latência, taxa de erro) permaneçam dentro da
  meta durante a janela de canary/blue-green.

---

## 9. Escalonamento

| Sinal | Ação | Referência |
|-------|------|--------------|
| CPU média de réplica > 70% por 5 min | Adicionar réplica. | Autoescalonamento horizontal padrão. |
| `aios_kernel_syscall_latency_ms{quantile="0.99"}` acima da meta de NFR-001/NFR-002 | Adicionar réplica; investigar dependência lenta (PDP/Scheduler/PostgreSQL). | `Scalability.md` §7 (limites teóricos). |
| Fila/backlog do Outbox (`kernel.outbox` com `published=false` crescente) | Aumentar paralelismo do relay ou investigar partição do NATS. | `FailureRecovery.md` (partição do NATS). |
| CPU média de réplica < 30% por 15 min | Remover réplica (respeitando mínimo de 3 em produção). | Autoescalonamento horizontal padrão. |

O modelo detalhado de escala (sharding, hot/cold state, limites teóricos de ACBs
e syscalls/s) é especificado em `./Scalability.md` — este documento cobre apenas
os gatilhos operacionais de infraestrutura.

---

## 10. Configuração de Rede e Gateway (YARP)

| Aspecto | Especificação |
|---------|-----------------|
| Roteamento | YARP roteia `/v1/kernel/*` (REST externo) ao pool de réplicas do Kernel via *round-robin* com *health-aware* (remove réplicas com `readyz` falho). |
| Rate-limit de borda | O Gateway aplica um rate-limit *coarse* por tenant/IP como primeira linha de defesa; o rate-limit fino por agente (`syscalls_per_sec`) é responsabilidade do Kernel (`ResourceQuotaManager`) — ambos coexistem em camadas (*defense in depth*). |
| Terminação TLS | TLS externo terminado no Gateway; tráfego Gateway→Kernel é mTLS interno (não é HTTP puro). |
| Cabeçalhos propagados | `traceparent`, `X-AIOS-Tenant`, `X-AIOS-Agent`, `Idempotency-Key`, `X-AIOS-Api-Version` (RFC-0001 §5.6) — o Gateway **DEVE** preservar/gerar esses cabeçalhos antes de encaminhar ao Kernel. |
| gRPC interno | Não passa pelo YARP externo; consumido diretamente pela malha de serviços internos (mTLS ponto-a-ponto ou via *service mesh sidecar*, conforme `028`). |

---

## 11. Ambientes

| Ambiente | Réplicas mínimas | Dados | Uso |
|----------|--------------------|-------|-----|
| `dev` | 1 | Sintéticos, resetáveis | Desenvolvimento local (Docker Compose, §4). |
| `staging` | 3 | Mascarados/sintéticos representativos de produção | Validação de rollout, testes de carga (`Benchmark.md`), *chaos drills* (`FailureRecovery.md`). |
| `production` | ≥ 3, distribuídas em ≥ 2 AZ | Reais, sob RLS/mTLS/backup completo | Operação real; sujeito a todos os SLOs de `NonFunctionalRequirements.md`. |

---

## 12. Migrações de Banco de Dados

- Migrações de schema **DEVEM** ser versionadas e aplicadas por ferramenta de
  migração declarativa (ex.: EF Core Migrations / Flyway) executada como um
  passo de *pre-deploy* separado do rollout de réplicas da aplicação.
- Toda migração **DEVE** ser testada quanto a compatibilidade com a versão de
  código **anterior** (para suportar rolling update sem downtime) antes de ser
  aplicada em produção.
- Detalhes de DDL, índices e particionamento: `./Database.md`.

---

## 13. Referências

- Arquitetura geral e diagrama de componentes: `./_DESIGN_BRIEF.md` §2.
- Modelo de escala e sharding: `./Scalability.md`.
- Modos de falha e recuperação: `./FailureRecovery.md`.
- Postura de segurança e hardening: `./Security.md`.
- Contratos centrais (correlação, versionamento de API): `../003-RFC/RFC-0001-Architecture-Baseline.md` §5.6–§5.7.
- Alta disponibilidade e disaster recovery (transversal): `../027-HA-DR/`.
- Gateway e infraestrutura de borda: `../028-Gateway-Infra/`.
- Observabilidade (métricas/probes): `../024-Observability/`.
