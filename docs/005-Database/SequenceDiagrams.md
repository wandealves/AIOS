---
Documento: SequenceDiagrams
Módulo: 005-Database
Status: Draft
Versão: 0.1
Última atualização: 2026-07-22
Responsável (RACI-A): Arquiteto do Módulo 005-Database
ADRs relacionados: ADR-0051, ADR-0053, ADR-0055, ADR-0056
RFCs relacionados: RFC-0001, RFC-0050, RFC-0051
Depende de: UseCases.md, StateMachine.md, API.md, Events.md
---

# 005-Database — Diagramas de Sequência

Todos os diagramas são ASCII. Convenção: `──▶` chamada síncrona, `╌╌▶` mensagem
assíncrona (NATS), `◀──` resposta, `[guard]` condição, `⟲` retry. Toda chamada carrega
os cabeçalhos de correlação da RFC-0001 §5.6.

---

## 1. Caminho feliz — Aplicar migração (UC-004)

```
 Dono   004-API  MigrationEngine  DdlValidator  PDP(022)  PG-primário  Outbox/NATS  025-Audit
  │        │           │              │            │           │            │          │
  │POST    │           │              │            │           │            │          │
  │migrat. │           │              │            │           │            │          │
  ├───────▶│──gRPC────▶│ T-01 Draft   │            │           │            │          │
  │        │           ├─Validate────▶│            │           │            │          │
  │        │           │◀──report ok──┤            │           │            │          │
  │        │           │ T-02 Validated            │           │            │          │
  │        │           ├─dry-run em réplica ───────────────────▶(réplica)   │          │
  │        │           │◀─ duration=1.8s, block=40ms ──────────┤            │          │
  │        │           │ T-04 DryRun  │            │           │            │          │
  │:apply  │           │              │            │           │            │          │
  ├───────▶│──────────▶│─Decide(db:migration:apply)▶│           │            │          │
  │        │           │◀────────── allow ─────────┤           │            │          │
  │        │           ├─ pg_advisory_lock(005) ───────────────▶│            │          │
  │        │           │◀───────── acquired ───────────────────┤            │          │
  │        │           │ T-06 Applying│            │           │            │          │
  │        │           ├─ BEGIN; DDL; UPDATE registry; INSERT outbox; COMMIT ▶          │
  │        │           │◀──────────── ok ──────────────────────┤            │          │
  │        │           │ T-07 Applied │            │           │            │          │
  │        │           ├─ unlock ─────────────────────────────▶│            │          │
  │        │           ├─ audit(apply, allow, digest) ─────────────────────────────────▶│
  │◀─ 200 ─┤◀──────────┤              │            │           │            │          │
  │        │           │              │            │           │ relay ╌╌╌╌▶│          │
  │        │           │       aios._platform.database.migration.applied     │          │
```

**Notas.** O evento só existe porque a linha do Outbox foi commitada na **mesma**
transação do DDL (I4). Se o processo morrer entre o `COMMIT` e a publicação, o relay
reenvia e os consumidores deduplicam por `event.id` (RFC-0001 §5.5).

---

## 2. Caminho feliz — Consulta de dados de módulo (caminho quente)

```
 006-Kernel   PgBouncer      ReplicationSuperv.   PG-primário   PG-réplica
     │            │                  │                 │            │
     ├─ conecta (role kernel_rw, TLS) ▶                │            │
     │            ├─ Acquire(service=006, mode=read) ─▶│            │
     │            │◀─ target=replica (lag=120ms < 1000)┤            │
     ├─ SET app.tenant_id = 'acme' ────────────────────────────────▶│
     ├─ SELECT … WHERE urn = $1 ───────────────────────────────────▶│
     │                                    [RLS filtra por tenant_id]│
     │◀───────────────── linha (p99 ≤ 5 ms, NFR-001) ──────────────┤
```

O serviço `aios-database-svc` **não participa** deste caminho (ver
`./Architecture.md` §1) — ele governa as *roles*, a RLS e os limites, mas não fica
entre a aplicação e o dado.

---

## 3. Caminho feliz — Retenção por *drop* de partição (UC-008)

```
 RetentionEnforcer  PartitionManager   PG-primário   Outbox/NATS   026-Cost
        │                  │                │             │           │
        ├─ RunCycle() ─────▶                │             │           │
        │  [para cada tabela range_time]    │             │           │
        │                  ├─ corte = now - ttl           │           │
        │                  ├─ [legal_hold?] não           │           │
        │                  ├─ ALTER TABLE … DETACH PARTITION p_2026_04 ▶
        │                  │◀── ok ─────────┤             │           │
        │                  │  … aguarda drop_grace_h (24h) …          │
        │                  ├─ DROP TABLE p_2026_04 ───────▶           │
        │                  │◀── ok, 42.1M linhas, 8.3 GiB ┤           │
        ├─ INSERT outbox(retention.purged) ─▶             │           │
        │                  │                │ relay ╌╌╌╌╌▶│           │
        │                  │   aios.<tenant>.database.retention.purged ╌╌▶│
```

O custo é O(1): não há varredura, não há tuplas mortas, não há *bloat* (NFR-016).

---

## 4. Falha — Lint reprova a migração (UC-002 E1)

```
 Dono   004-API   MigrationEngine   DdlValidator
  │        │            │                │
  ├:validate──────────▶ │                │
  │        │            ├─ Validate ────▶│
  │        │            │                ├ TenantIdRequiredRule ....... ok
  │        │            │                ├ RlsPolicyRequiredRule ...... VIOLADA
  │        │            │                ├ RetentionDeclaredRule ...... VIOLADA
  │        │            │◀── report(2 violações bloqueantes) ─┤
  │        │            │ T-03 → Failed  │
  │◀ 422 AIOS-MIGRATION-0002 ────────────┤
  │        │            │ ╌╌▶ aios._platform.database.migration.failed
```

O erro chega **antes** do deploy, com a lista exata do que falta — o custo de violar a
convenção é um teste vermelho, não um incidente.

---

## 5. Falha — Timeout na aplicação (UC-004 E2)

```
 MigrationEngine     PG-primário      Outbox/NATS      Monitoring
       │                  │                │                │
       │ T-06 Applying    │                │                │
       ├─ BEGIN; ALTER TABLE … ───────────▶│                │
       │  … 15 min (db.migration.timeout_ms = 900000) …      │
       ├─ [timeout] cancel backend ───────▶│                │
       │◀── ROLLBACK completo ─────────────┤                │
       │ T-08 Failed (AIOS-MIGRATION-0005) │                │
       ├─ pg_advisory_unlock ─────────────▶│                │
       ├─ INSERT outbox(migration.failed) ─▶ ╌╌╌╌╌╌╌╌╌╌╌╌╌▶ │ alerta
```

Pela invariante **I2**, não existe estado intermediário: o schema volta exatamente ao
que era. A correção é **forward-fix** (nova migração, T-10), nunca edição do histórico.

---

## 6. Falha — Duas migrações concorrentes (UC-004 E1)

```
 CI-A      MigrationEngine(réplica 1)   PG advisory lock   MigrationEngine(réplica 2)   CI-B
  │                 │                          │                       │                │
  ├─ apply(M1) ────▶│                          │                       │◀──── apply(M2) ┤
  │                 ├─ pg_advisory_lock(005) ─▶│                       │                │
  │                 │◀── acquired ─────────────┤                       │                │
  │                 │                          │◀─ pg_try_advisory_lock ┤                │
  │                 │                          ├── busy ──────────────▶│                │
  │                 │  T-06 Applying (M1)      │        [espera lock_timeout=5s]         │
  │                 │                          │                       │ T-nenhuma       │
  │                 │                          │                       ├─ 409 ──────────▶│
  │                 │                          │                       │ AIOS-MIGRATION-0001 (retriable)
```

Invariante **I1** garantida pelo banco, não por convenção operacional.

---

## 7. Falha — Perda do primário e promoção (UC-012)

```
 027-Cluster   ReplicationSuperv.   PG-primário   PG-réplica-1   PgBouncer   Consumidores
      │               │                  ✗             │             │            │
      │               ├─ health check ───▶(sem resposta)│             │            │
      │◀╌ database.replication.lagging ╌╌┤              │             │            │
      ├╌ cluster.failover.requested ╌╌╌▶ │              │             │            │
      │               ├─ escolhe menor lag (réplica-1, 80 ms) ───────▶│            │
      │               ├─ pg_promote() ──────────────────▶             │            │
      │               │◀────────── promovido ───────────┤             │            │
      │               ├─ reconfigura upstream ───────────────────────▶│            │
      │               │                                 │             ├─ escritas  │
      │               │                                 │             │   voltam ─▶│
      │◀╌ database.replication.promoted ╌┤              │             │            │
```

Durante a janela, escritas recebem `AIOS-DB-0005` (retriable) e leituras elegíveis
seguem servidas por réplicas — degradação graciosa em vez de indisponibilidade total.

---

## 8. Direito ao esquecimento (UC-009)

```
 DPO   004-API  ErasureCoordinator  PDP(022)  SchemaRegistry  PG   025-Audit  NATS
  │       │            │                │           │          │        │       │
  ├ POST ▶│───────────▶│ status=received│           │          │        │       │
  │       │            ├─ Decide(db:erasure:execute)▶          │        │       │
  │       │            │◀──── allow ────┤           │          │        │       │
  │       │            │ status=authorized          │          │        │       │
  │       │            ├─ resolve tabelas do titular ─────────▶│        │       │
  │       │            │◀─ [memory.item, memory.embedding, graph.vertex,…] ─────┤
  │       │            │ [legal_hold em alguma? não]│          │        │       │
  │       │            │ status=executing           │          │        │       │
  │       │            ├─ DELETE em lotes, ordem segura de FK ▶│        │       │
  │       │            │◀── rows=18.402 ───────────────────────┤        │       │
  │       │            ├─ receipt_hash = sha256(...) ───────────────────▶│       │
  │       │            │ status=completed           │          │        │       │
  │◀ 200 ─┤◀───────────┤                            │          │        │  ╌╌╌╌▶│
  │       │            │        aios.<tenant>.database.erasure.completed        │
```

Se `legal_hold` estivesse ativo, o fluxo pararia em `rejected` com `AIOS-DB-0010`, e o
conflito entre preservar e apagar seria **registrado**, nunca resolvido silenciosamente.

---

## 9. Restauração a um ponto no tempo (UC-011)

```
 SRE   004-API  BackupOrchestrator  PDP(022)  MinIO   PG-restaurado  PgBouncer  025-Audit
  │       │            │                │        │          │            │          │
  ├POST  ▶│───────────▶│─ Decide(db:restore:execute) ▶      │            │          │
  │restore│            │◀── allow (+2ª aprovação) ┤         │            │          │
  │       │            ├─ seleciona full ≤ alvo ─▶│         │            │          │
  │       │            │◀── artefato + checksum ok┤         │            │          │
  │       │            ├─ restore base ──────────────────────▶           │          │
  │       │            ├─ replay WAL até LSN alvo ──────────▶│           │          │
  │       │            │◀────────── consistente ─────────────┤           │          │
  │       │            ├─ reaponta upstream ────────────────────────────▶│          │
  │       │            ├─ registra alvo, operador, duração ─────────────────────────▶│
  │◀ 200 ─┤◀───────────┤  RTO medido: 9m42s (≤ 15 min, NFR-009)          │          │
```

---

## 10. Referências

- Casos de uso: `./UseCases.md` · FSM: `./StateMachine.md`
- API e erros: `./API.md` · Eventos: `./Events.md`
- Falhas e recuperação: `./FailureRecovery.md`
