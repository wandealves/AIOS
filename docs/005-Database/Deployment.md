---
Documento: Deployment
Módulo: 005-Database
Status: Draft
Versão: 0.1
Última atualização: 2026-07-22
Responsável (RACI-A): Arquiteto do Módulo 005-Database
ADRs relacionados: ADR-0005, ADR-0053, ADR-0056, ADR-0059
RFCs relacionados: RFC-0001, RFC-0050
Depende de: 028-Deployment, 027-Cluster, 021-Security, 020-Communication, 024-Observability
---

# 005-Database — Implantação

Empacotamento **Docker/Compose** (stack de referência do `../028-Deployment/`).
Este documento descreve a topologia do subsistema de dados e o papel de cada
contêiner; a topologia global do AIOS pertence ao `028`.

---

## 1. Topologia

```
                       ┌────────────────────────────────┐
                       │        004-API (YARP)          │  ← única porta externa
                       └───────────────┬────────────────┘
                                       │ /v1/database (admin)
                       ┌───────────────▼────────────────┐
                       │  aios-database-svc  × 2        │  .NET 10, stateless
                       │  (ativo/passivo por advisory   │  jobs: retenção, partição,
                       │   lock; ambos servem leitura)  │  backup, auditoria
                       └───┬───────────┬────────────┬───┘
                           │           │            │
   módulos 006…026 ────────┼───────────┘            │ eventos (Outbox → JetStream)
        │                  ▼                        ▼
        │        ┌────────────────────┐      ┌────────────┐
        └───────▶│ aios-pgbouncer × 2 │      │ NATS (020) │
                 │  mode=transaction  │      └────────────┘
                 └───┬────────────┬───┘
          escrita    │            │  leitura elegível
                     ▼            ▼
   ┌───────────────────────┐   ┌────────────────────────────┐
   │ aios-postgres-primary │──▶│ aios-postgres-replica × 2+  │  streaming
   │ PG16 + pgvector + AGE │   │ (RO, dry-run, candidatas)   │
   └───────────┬───────────┘   └────────────────────────────┘
               │ WAL contínuo + backup full/incremental
               ▼
        ┌──────────────┐
        │ MinIO (028)  │  s3://aios-backups/{full,incr,wal}
        └──────────────┘
```

---

## 2. Contêineres e Recursos

| Contêiner | Imagem base | CPU (req/lim) | RAM (req/lim) | Disco | Réplicas |
|-----------|-------------|---------------|---------------|-------|----------|
| `aios-database-svc` | `mcr.microsoft.com/dotnet/aspnet:10` | 0,5 / 2 | 512 MiB / 2 GiB | — | 2 |
| `aios-postgres-primary` | `postgres:16` + `pgvector` + `age` | 4 / 16 | 16 GiB / 64 GiB | SSD NVMe, ≥ 1 TiB | 1 |
| `aios-postgres-replica` | idem | 2 / 8 | 8 GiB / 32 GiB | SSD NVMe, ≥ 1 TiB | ≥ 2 |
| `aios-pgbouncer` | `pgbouncer:1.22` | 0,25 / 1 | 128 MiB / 512 MiB | — | 2 |
| `aios-wal-archiver` | *sidecar* do primário | 0,25 / 1 | 128 MiB / 512 MiB | buffer 20 GiB | 1 |

**Regras de dimensionamento.** `shared_buffers` ≈ 25% da RAM do contêiner;
`effective_cache_size` ≈ 60%; `work_mem` conservador (busca vetorial e ordenação são
os maiores consumidores). GPU **não é usada** por este módulo.

---

## 3. Ordem de Inicialização e Dependências

```
   1. MinIO (028)  +  cofre de segredos (021)
   2. aios-postgres-primary   → healthy (pg_isready)
   3. aios-postgres-replica   → em streaming, lag < max_lag_ms
   4. aios-pgbouncer          → pools abertos
   5. aios-database-svc       → aplica migrações pendentes (se autorizado) e sobe
   6. demais módulos (006…026) → conectam via PgBouncer
```

Um módulo consumidor **NÃO DEVE** iniciar antes de o `aios-database-svc` reportar
`ready`, pois a migração pendente pode ser pré-requisito do seu schema.

---

## 4. Health e Readiness

| Endpoint | Contêiner | Critério |
|----------|-----------|----------|
| `GET /healthz` (liveness) | `aios-database-svc` | Processo responde; sem *deadlock* interno. |
| `GET /readyz` (readiness) | `aios-database-svc` | Conexão ao primário OK **e** nenhuma migração pendente obrigatória **e** PDP alcançável (ou `fail-closed` assumido). |
| `pg_isready` | `postgres-*` | Aceita conexões. |
| `SELECT pg_is_in_recovery()` | réplicas | Deve ser `true` em réplica; `false` no primário. |
| `SHOW POOLS` | `pgbouncer` | Pools ativos e sem fila persistente. |

Readiness **NÃO DEVE** depender de MinIO: falha de backup degrada o RPO e gera alerta,
mas não derruba o plano de dados.

---

## 5. Estratégia de Rollout

| Componente | Estratégia | Observação |
|------------|-----------|------------|
| `aios-database-svc` | *Rolling* (1 por vez) | Stateless; o advisory lock impede duas instâncias aplicando migração. |
| Migrações de schema | **Expand → deploy → Migrate → deploy → Contract** | Fase `contract` só após todas as instâncias da aplicação estarem na versão nova. |
| `aios-pgbouncer` | *Rolling*, drenando conexões | Clientes reconectam com backoff. |
| `aios-postgres-primary` (upgrade menor) | Failover planejado: promover réplica atualizada | Coordenado com `../027-Cluster/`. |
| `aios-postgres-primary` (upgrade maior) | Replicação lógica para novo cluster + *cutover* | Janela planejada, ensaiada em *dry-run*. |

Durante um rollout, o módulo consome
`aios._platform.deployment.rollout.started` e **congela migrações `contract`** até o
fim da janela (`./Events.md` §4).

---

## 6. Volumes e Persistência

| Volume | Montagem | Conteúdo | Backup |
|--------|----------|----------|--------|
| `pgdata-primary` | `/var/lib/postgresql/data` | Dados do primário | Via WAL + full para MinIO |
| `pgdata-replica-N` | idem | Réplica | Reconstruível a partir do primário |
| `walbuffer` | `/var/lib/postgresql/walbuffer` | Buffer de WAL antes do envio | — (transitório) |

Volumes **DEVEM** estar em armazenamento cifrado em repouso (`./Security.md` §7).

---

## 7. Extrato de `compose.yaml`

```yaml
services:
  aios-postgres-primary:
    image: aios/postgres:16-pgvector-age
    environment:
      POSTGRES_DB: aios
      POSTGRES_INITDB_ARGS: "--data-checksums"
    command: >
      postgres
        -c synchronous_commit=on
        -c wal_level=replica
        -c archive_mode=on
        -c archive_timeout=60
        -c archive_command='/usr/local/bin/archive-to-minio %p %f'
        -c statement_timeout=30s
        -c idle_in_transaction_session_timeout=60s
        -c lock_timeout=5s
        -c shared_preload_libraries=pg_stat_statements
    volumes: [ "pgdata-primary:/var/lib/postgresql/data" ]
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U aios"]
      interval: 10s
      retries: 6

  aios-pgbouncer:
    image: aios/pgbouncer:1.22
    environment:
      POOL_MODE: transaction
      DEFAULT_POOL_SIZE: "50"          # db.pool.max_connections_per_service
      MAX_CLIENT_CONN: "5000"
    depends_on: [ aios-postgres-primary ]

  aios-database-svc:
    image: aios/database-svc:0.1
    environment:
      AIOS_DB_MIGRATION_REQUIRE_DRY_RUN: "true"
      AIOS_DB_MIGRATION_MAX_BLOCK_MS: "200"
      AIOS_DB_REPLICATION_MAX_LAG_MS: "1000"
      AIOS_DB_BACKUP_WAL_ARCHIVE_TIMEOUT_S: "60"
      OTEL_EXPORTER_OTLP_ENDPOINT: "http://otel-collector:4317"
    depends_on: [ aios-pgbouncer, nats ]
    deploy: { replicas: 2 }
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8080/readyz"]
      interval: 15s

volumes:
  pgdata-primary:
```

---

## 8. Ambientes

| Ambiente | Diferenças |
|----------|-----------|
| `dev` (local) | 1 primário, 0 réplicas, backup desligado, `require_dry_run=false`. RLS **permanece ligada** — desligar mascara defeitos que só aparecem em produção. |
| `staging` | Topologia idêntica à produção, com 1 réplica e volume reduzido; é onde o *dry-run* de migração roda no CI. |
| `prod` | Topologia completa (§1), réplica síncrona no quórum, *restore drill* semanal. |

---

## 9. Referências

- Topologia global: `../028-Deployment/` · HA/DR: `../027-Cluster/`
- Configuração: `./Configuration.md` · Modelo físico: `./Database.md`
- Monitoramento: `./Monitoring.md` · Falhas: `./FailureRecovery.md`
