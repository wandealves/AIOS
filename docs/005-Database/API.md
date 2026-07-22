---
Documento: API
Módulo: 005-Database
Status: Draft
Versão: 0.1
Última atualização: 2026-07-22
Responsável (RACI-A): Arquiteto do Módulo 005-Database
ADRs relacionados: ADR-0053, ADR-0055, ADR-0056, ADR-0057, ADR-0059
RFCs relacionados: RFC-0001, RFC-0050, RFC-0051
Depende de: _DESIGN_BRIEF.md §5, 004-API (registro de erros), 021-Security, 022-Policy
---

# 005-Database — Contratos de API

> **Esta é uma API administrativa.** Nenhum dado de domínio trafega por ela: os
> módulos consumidores acessam o PostgreSQL diretamente pelo `ConnectionPoolGateway`,
> sob as *roles*, RLS e limites que este módulo governa (`./Architecture.md` §1).

Autenticação, envelope de erro, idempotência, correlação e versionamento seguem
`../003-RFC/RFC-0001-Architecture-Baseline.md` §5.4–§5.7 e **não são redefinidos aqui**.

- Base REST: `/v1/database` (exposta via `../004-API/`).
- Pacote gRPC: `aios.database.v1`.
- Toda mutação **DEVE** enviar `Idempotency-Key` e os cabeçalhos de correlação
  (`traceparent`, `X-AIOS-Tenant`).
- Operações de plataforma usam o tenant reservado `_platform`.

---

## 1. Superfície de Operações

| Operação | REST | gRPC (rpc) | Idempotente | Capability (PDP) |
|----------|------|-----------|-------------|------------------|
| Submeter migração | `POST /v1/database/migrations` | `SubmitMigration` | sim (key) | `db:migration:submit` |
| Validar migração | `POST /v1/database/migrations/{urn}:validate` | `ValidateMigration` | sim | `db:migration:submit` |
| Ensaiar migração | `POST /v1/database/migrations/{urn}:dry-run` | `DryRunMigration` | sim | `db:migration:submit` |
| Aplicar migração | `POST /v1/database/migrations/{urn}:apply` | `ApplyMigration` | sim (key) | `db:migration:apply` |
| Reverter migração | `POST /v1/database/migrations/{urn}:rollback` | `RollbackMigration` | sim | `db:migration:rollback` |
| Listar migrações | `GET /v1/database/migrations` | `ListMigrations` | sim (leitura) | `db:migration:read` |
| Ler migração | `GET /v1/database/migrations/{urn}` | `GetMigration` | sim (leitura) | `db:migration:read` |
| Ler registro de schema | `GET /v1/database/schemas[/{schema}/{table}]` | `GetSchemaObject` | sim (leitura) | `db:schema:read` |
| Auditar conformidade | `POST /v1/database/schemas:audit` | `AuditSchemas` | sim (leitura) | `db:schema:audit` |
| Definir retenção | `PUT /v1/database/retention-policies/{name}` | `PutRetentionPolicy` | sim | `db:retention:write` |
| Garantir partições | `POST /v1/database/partitions:ensure` | `EnsurePartitions` | sim | `db:partition:write` |
| Listar partições | `GET /v1/database/partitions` | `ListPartitions` | sim (leitura) | `db:partition:read` |
| Solicitar expurgo | `POST /v1/database/erasure-requests` | `RequestErasure` | sim (key) | `db:erasure:execute` |
| Consultar expurgo | `GET /v1/database/erasure-requests/{urn}` | `GetErasureRequest` | sim (leitura) | `db:erasure:read` |
| Listar backups | `GET /v1/database/backups` | `ListBackups` | sim (leitura) | `db:backup:read` |
| Verificar backup | `POST /v1/database/backups/{urn}:verify` | `VerifyBackup` | sim | `db:backup:verify` |
| Restaurar (PITR) | `POST /v1/database/restore` | `RestorePointInTime` | sim (key) | `db:restore:execute` (+ 2ª aprovação) |
| Status de replicação | `GET /v1/database/replication` | `GetReplicationStatus` | sim (leitura) | `db:replication:read` |
| Definir índice vetorial | `PUT /v1/database/vector-indexes/{schema}/{table}/{column}` | `PutVectorIndex` | sim | `db:vector:write` |
| Reindexar vetorial | `POST /v1/database/vector-indexes/{schema}/{table}/{column}:reindex` | `ReindexVector` | sim | `db:vector:write` |

---

## 2. OpenAPI (extrato normativo)

```yaml
openapi: 3.1.0
info:
  title: AIOS Database Platform API
  version: "1.0.0"
servers:
  - url: https://api.aios.local/v1/database
components:
  parameters:
    Traceparent:    { name: traceparent,      in: header, required: true,  schema: { type: string } }
    Tenant:         { name: X-AIOS-Tenant,    in: header, required: true,  schema: { type: string } }
    IdempotencyKey: { name: Idempotency-Key,  in: header, required: true,  schema: { type: string } }
  schemas:
    Migration:
      type: object
      required: [urn, version, ownerModule, state, phase, upSqlDigest, createdAt]
      properties:
        urn:                { type: string, example: "urn:aios:_platform:migration:01J9ZB0K3N5Q7R9T1V3X5Z7A9C" }
        version:            { type: string, example: "20260722T140000_add_memory_tag" }
        ownerModule:        { type: string, example: "010-Memory" }
        state:              { type: string, enum: [Draft, Validated, DryRun, Applying, Applied, Failed, RolledBack, Superseded] }
        phase:              { type: string, enum: [expand, migrate, contract] }
        upSqlDigest:        { type: string, description: "SHA-256 do script up" }
        reversible:         { type: boolean, default: false }
        blockingEstimateMs: { type: integer, default: 0 }
        dryRunAt:           { type: string, format: date-time, nullable: true }
        appliedAt:          { type: string, format: date-time, nullable: true }
        failureCode:        { type: string, nullable: true }
    Problem:   # RFC 7807 / RFC-0001 §5.4 — NÃO redefinido aqui
      $ref: "https://docs.aios/schemas/problem.json"
paths:
  /migrations:
    post:
      summary: Submete uma migração (estado Draft, T-01)
      parameters: [ { $ref: '#/components/parameters/Traceparent' },
                    { $ref: '#/components/parameters/Tenant' },
                    { $ref: '#/components/parameters/IdempotencyKey' } ]
      requestBody:
        required: true
        content:
          application/json:
            schema:
              type: object
              required: [version, ownerModule, phase, upSql]
              properties:
                version:     { type: string }
                ownerModule: { type: string }
                phase:       { type: string, enum: [expand, migrate, contract] }
                upSql:       { type: string }
                downSql:     { type: string, nullable: true }
      responses:
        "201": { description: Criada, content: { application/json: { schema: { $ref: '#/components/schemas/Migration' } } } }
        "409": { description: "AIOS-MIGRATION-0004 (version duplicada)" }
        "403": { description: "AIOS-DB-0002 (capability negada)" }
  /migrations/{urn}:apply:
    post:
      summary: Aplica a migração no primário (T-06 → T-07 | T-08)
      responses:
        "200": { description: Aplicada }
        "409": { description: "AIOS-MIGRATION-0001 (lock ocupado) — retriable" }
        "412": { description: "AIOS-MIGRATION-0007 (dry-run ausente)" }
        "504": { description: "AIOS-MIGRATION-0005 (timeout; transação revertida)" }
```

---

## 3. gRPC (extrato do `.proto`)

```protobuf
syntax = "proto3";
package aios.database.v1;

import "google/protobuf/timestamp.proto";

service DatabasePlatform {
  rpc SubmitMigration      (SubmitMigrationRequest)  returns (Migration);
  rpc ValidateMigration    (MigrationRef)            returns (Migration);
  rpc DryRunMigration      (MigrationRef)            returns (Migration);
  rpc ApplyMigration       (ApplyMigrationRequest)   returns (Migration);
  rpc RollbackMigration    (RollbackRequest)         returns (Migration);
  rpc ListMigrations       (ListMigrationsRequest)   returns (ListMigrationsResponse);
  rpc GetSchemaObject      (SchemaObjectRef)         returns (SchemaObject);
  rpc AuditSchemas         (AuditRequest)            returns (ComplianceReport);
  rpc PutRetentionPolicy   (RetentionPolicy)         returns (RetentionPolicy);
  rpc EnsurePartitions     (EnsurePartitionsRequest) returns (PartitionReport);
  rpc RequestErasure       (ErasureRequest)          returns (ErasureStatus);
  rpc ListBackups          (ListBackupsRequest)      returns (ListBackupsResponse);
  rpc VerifyBackup         (BackupRef)               returns (VerificationResult);
  rpc RestorePointInTime   (RestoreRequest)          returns (RestoreOutcome);
  rpc GetReplicationStatus (Empty)                   returns (ReplicationStatus);
  rpc PutVectorIndex       (VectorIndexSpec)         returns (VectorIndexSpec);
  rpc ReindexVector        (VectorIndexRef)          returns (ReindexOutcome);
}

message Migration {
  string urn = 1;
  string version = 2;
  string owner_module = 3;
  string state = 4;                 // MigrationState (StateMachine.md §1)
  string phase = 5;                 // expand | migrate | contract
  string up_sql_digest = 6;
  bool   reversible = 7;
  int32  blocking_estimate_ms = 8;
  google.protobuf.Timestamp dry_run_at = 9;
  google.protobuf.Timestamp applied_at = 10;
  string failure_code = 11;
}

message ApplyMigrationRequest {
  string urn = 1;
  string idempotency_key = 2;       // RFC-0001 §5.5
}
```

Erros gRPC mapeiam para `google.rpc.Status` + `ErrorInfo` com o mesmo `code`
`AIOS-<DOMINIO>-<NNNN>`, conforme RFC-0001 §5.4.

---

## 4. Paginação, Filtros e Concorrência

| Aspecto | Regra |
|---------|-------|
| Paginação | Cursor opaco: `?pageSize=50&pageToken=<opaco>`; resposta traz `nextPageToken`. Máximo `pageSize` = 200. |
| Ordenação | Padrão: `created_at DESC` (ULID já é ordenável por tempo). |
| Filtros | `?state=Applied&ownerModule=010-Memory&since=2026-07-01T00:00:00Z`. |
| Concorrência | Mutações de configuração usam OCC: header `If-Match: <version>`; divergência → `AIOS-DB-0004` (409, retriable). |
| Idempotência | `Idempotency-Key` persistida por ≥ 24h (RFC-0001 §5.5); repetição com payload divergente → 409. |

---

## 5. Catálogo de Códigos de Erro

Domínios reservados por este módulo: **`DB`** (0001–0099) e **`MIGRATION`**
(0001–0099), registrados no catálogo mantido por `../004-API/` (RFC-0001 §8).

| Código | HTTP | `retriable` | Significado |
|--------|------|-------------|-------------|
| `AIOS-DB-0001` | 404 | não | Objeto de schema/partição/backup não encontrado. |
| `AIOS-DB-0002` | 403 | não | Operação administrativa negada pelo PDP (*default deny*). |
| `AIOS-DB-0003` | 403 | não | Tenant divergente do contexto autenticado (violação de RLS). |
| `AIOS-DB-0004` | 409 | sim | Conflito de concorrência otimista (`version`). |
| `AIOS-DB-0005` | 503 | sim | Primário indisponível; escrita rejeitada. |
| `AIOS-DB-0006` | 503 | sim | Pool de conexões esgotado para o serviço chamador (bulkhead). |
| `AIOS-DB-0007` | 504 | sim | `statement_timeout` excedido; consulta abortada. |
| `AIOS-DB-0008` | 409 | não | Violação de convenção física ao registrar tabela. |
| `AIOS-DB-0009` | 422 | não | Política de retenção sem `legal_basis` para tabela `pii`. |
| `AIOS-DB-0010` | 423 | não | Expurgo bloqueado por `legal_hold` ativo. |
| `AIOS-DB-0011` | 409 | não | Dimensão de coluna vetorial é imutável. |
| `AIOS-DB-0012` | 507 | não | Limite de armazenamento do tenant excedido. |
| `AIOS-DB-0013` | 503 | sim | Lag de replicação acima do limiar; leitura obsoleta recusada. |
| `AIOS-MIGRATION-0001` | 409 | sim | Já existe migração em `Applying` (advisory lock ocupado). |
| `AIOS-MIGRATION-0002` | 422 | não | Migração reprovada no lint de convenções. |
| `AIOS-MIGRATION-0003` | 409 | não | Transição de estado inválida (viola a FSM). |
| `AIOS-MIGRATION-0004` | 409 | não | `version` de migração duplicada. |
| `AIOS-MIGRATION-0005` | 504 | sim | Timeout na aplicação; transação revertida. |
| `AIOS-MIGRATION-0006` | 409 | não | Rollback recusado (não reversível ou com dependentes). |
| `AIOS-MIGRATION-0007` | 412 | não | *Dry-run* obrigatório ausente. |
| `AIOS-MIGRATION-0008` | 409 | não | Digest do script divergente do registrado. |

---

## 6. Exemplos

### 6.1 Aplicar migração (caminho feliz)

```http
POST /v1/database/migrations/urn:aios:_platform:migration:01J9ZB0K3N5Q7R9T1V3X5Z7A9C:apply HTTP/1.1
Host: api.aios.local
Authorization: Bearer <token>
X-AIOS-Tenant: _platform
Idempotency-Key: 01J9ZB1M4P6R8T0V2X4Z6B8D0F
traceparent: 00-4bf92f3577b34da6a3ce929d0e0e4736-00f067aa0ba902b7-01
```

```http
HTTP/1.1 200 OK
Content-Type: application/json

{
  "urn": "urn:aios:_platform:migration:01J9ZB0K3N5Q7R9T1V3X5Z7A9C",
  "version": "20260722T140000_add_memory_tag",
  "ownerModule": "010-Memory",
  "state": "Applied",
  "phase": "expand",
  "upSqlDigest": "9f2c…a71b",
  "reversible": true,
  "blockingEstimateMs": 40,
  "dryRunAt": "2026-07-22T13:52:11Z",
  "appliedAt": "2026-07-22T14:03:47Z",
  "durationMs": 1812
}
```

### 6.2 Migração reprovada no lint

```http
HTTP/1.1 422 Unprocessable Entity
Content-Type: application/problem+json

{
  "type": "https://docs.aios/errors/migration-lint-failed",
  "title": "Migration Rejected by DDL Convention Lint",
  "status": 422,
  "code": "AIOS-MIGRATION-0002",
  "detail": "memory.tag: tabela multi-tenant sem POLICY de RLS; retenção não declarada.",
  "instance": "urn:aios:_platform:migration:01J9ZB0K3N5Q7R9T1V3X5Z7A9C",
  "traceId": "4bf92f3577b34da6a3ce929d0e0e4736",
  "timestamp": "2026-07-22T13:40:02.114Z",
  "retriable": false
}
```

### 6.3 Advisory lock ocupado

```http
HTTP/1.1 409 Conflict
Retry-After: 5
Content-Type: application/problem+json

{
  "type": "https://docs.aios/errors/migration-in-progress",
  "title": "Another Migration Is Being Applied",
  "status": 409,
  "code": "AIOS-MIGRATION-0001",
  "detail": "Migração 20260722T090000_expand_memory_content está em Applying.",
  "traceId": "4bf92f3577b34da6a3ce929d0e0e4736",
  "retriable": true,
  "retryAfterMs": 5000
}
```

### 6.4 Solicitar expurgo (direito ao esquecimento)

```bash
curl -sX POST https://api.aios.local/v1/database/erasure-requests \
  -H "X-AIOS-Tenant: acme" \
  -H "Idempotency-Key: 01J9ZB2N5Q7S9U1W3Y5A7C9E1G" \
  -H "Content-Type: application/json" \
  -d '{ "subjectUrn": "urn:aios:acme:agent:01J9Z8Q6H7K2M4N6P8R0S2T4V6",
        "scope": "subject" }'
```

```json
{
  "urn": "urn:aios:acme:erasurerequest:01J9ZB3P6R8T0V2X4Z6B8D0F2H",
  "status": "authorized",
  "tablesAffected": ["memory.item", "memory.embedding", "context.window", "graph.vertex"],
  "rowsErased": 0,
  "requestedAt": "2026-07-22T14:10:00Z"
}
```

---

## 7. Versionamento e Compatibilidade

- Versionamento por caminho (`/v1/…`) e header `X-AIOS-Api-Version` (RFC-0001 §5.7).
- Mudanças incompatíveis incrementam a major e coexistem por ≥ 2 majors.
- Campos novos são **aditivos**; clientes **DEVEM** tolerar campos desconhecidos.
- Os códigos de erro deste módulo são estáveis: um código **NÃO DEVE** mudar de
  significado; novos casos recebem números novos, registrados via `../004-API/`.

---

## 8. Referências

- Brief: `./_DESIGN_BRIEF.md` §5 · Casos de uso: `./UseCases.md`
- FSM: `./StateMachine.md` · Eventos: `./Events.md` · Exemplos: `./Examples.md`
- Contratos centrais: `../003-RFC/RFC-0001-Architecture-Baseline.md`
- Registro de erros: `../004-API/`
