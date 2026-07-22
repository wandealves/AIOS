---
Documento: Events
Módulo: 005-Database
Status: Draft
Versão: 0.1
Última atualização: 2026-07-22
Responsável (RACI-A): Arquiteto do Módulo 005-Database
ADRs relacionados: ADR-0050, ADR-0053, ADR-0055, ADR-0056
RFCs relacionados: RFC-0001, RFC-0050
Depende de: _DESIGN_BRIEF.md §6, 020-Communication, 004-API (registro de dataschema), 025-Audit
---

# 005-Database — Catálogo de Eventos NATS

O envelope de evento (CloudEvents) e a convenção de subjects são normativos na
`../003-RFC/RFC-0001-Architecture-Baseline.md` §5.2–§5.3 e **não são redefinidos
aqui**. Este documento cataloga os eventos que o módulo **produz** e **consome**.

- `<dominio>` = **`database`**, valor novo a ser registrado no registro de domínios
  mantido por `../004-API/` conforme RFC-0001 §8 (ratificação em **ADR-0050**).
- Entrega **at-least-once** via JetStream; consumidores **DEVEM** deduplicar por
  `event.id` (RFC-0001 §5.5).
- Publicação sempre pelo **Outbox transacional** (`platform.outbox`): o evento e a
  mudança de estado commitam juntos ou não commitam (FR-018).
- Eventos de **plataforma** usam o tenant reservado `_platform` (convenção
  estabelecida em `../004-API/_DESIGN_BRIEF.md` §6); eventos que refletem dado de um
  tenant usam o `<tenant>` real.

---

## 1. Eventos Emitidos

| Subject | `type` | Quando | Stream | Retenção |
|---------|--------|--------|--------|----------|
| `aios._platform.database.migration.applied` | `aios.database.migration.applied` | Migração → `Applied` (T-07). | `DB_PLATFORM` | 1 ano |
| `aios._platform.database.migration.failed` | `aios.database.migration.failed` | Migração → `Failed` (T-03/T-05/T-08). | `DB_PLATFORM` | 1 ano |
| `aios._platform.database.migration.rolledback` | `aios.database.migration.rolledback` | Migração → `RolledBack` (T-09). | `DB_PLATFORM` | 1 ano |
| `aios._platform.database.schema.registered` | `aios.database.schema.registered` | Nova tabela catalogada. | `DB_PLATFORM` | 1 ano |
| `aios._platform.database.partition.created` | `aios.database.partition.created` | Partição futura pré-criada. | `DB_PLATFORM` | 90 dias |
| `aios._platform.database.partition.dropped` | `aios.database.partition.dropped` | Partição expirada removida. | `DB_PLATFORM` | 90 dias |
| `aios._platform.database.backup.completed` | `aios.database.backup.completed` | Backup catalogado. | `DB_PLATFORM` | 1 ano |
| `aios._platform.database.backup.verified` | `aios.database.backup.verified` | *Restore drill* bem-sucedido. | `DB_PLATFORM` | 1 ano |
| `aios._platform.database.replication.lagging` | `aios.database.replication.lagging` | Lag > `db.replication.max_lag_ms`. | `DB_HEALTH` | 30 dias |
| `aios._platform.database.replication.promoted` | `aios.database.replication.promoted` | Réplica promovida a primário. | `DB_HEALTH` | 1 ano |
| `aios._platform.database.compliance.violated` | `aios.database.compliance.violated` | Tabela multi-tenant sem RLS ou sem retenção. | `DB_HEALTH` | 1 ano |
| `aios.<tenant>.database.retention.purged` | `aios.database.retention.purged` | Expurgo por TTL concluído. | `DB_RETENTION` | 7 anos |
| `aios.<tenant>.database.erasure.completed` | `aios.database.erasure.completed` | Direito ao esquecimento concluído. | `DB_RETENTION` | 7 anos |
| `aios.<tenant>.database.storage.threshold` | `aios.database.storage.threshold` | Uso de armazenamento cruzou limiar. | `DB_HEALTH` | 90 dias |

> A retenção longa de `DB_RETENTION` é deliberada: o **comprovante** de que um dado
> foi apagado precisa sobreviver ao dado apagado. O evento contém metadados e
> contagens, **nunca** o conteúdo expurgado.

---

## 2. Streams JetStream

| Stream | Subjects | Política | Réplicas |
|--------|----------|----------|----------|
| `DB_PLATFORM` | `aios._platform.database.migration.>`, `…schema.>`, `…partition.>`, `…backup.>` | `limits`, durável, `max_age=1y` | 3 |
| `DB_HEALTH` | `aios._platform.database.replication.>`, `…compliance.>`, `aios.*.database.storage.>` | `limits`, `max_age=90d` | 3 |
| `DB_RETENTION` | `aios.*.database.retention.>`, `aios.*.database.erasure.>` | `limits`, `max_age=7y` | 3 |

Ordenação é garantida **por stream**; consumidores que dependem de ordem entre
domínios diferentes **DEVEM** usar `time`/`event.id` para reconciliar.

---

## 3. Schemas de Payload

Todo evento usa o envelope da RFC-0001 §5.2. Abaixo apenas o `data` de cada tipo, com
seu `dataschema` registrado.

### 3.1 `aios.database.migration.applied`

`dataschema`: `aios://schemas/database.migration.applied/1`

```json
{
  "specversion": "1.0",
  "id": "01J9ZB4Q7S9U1W3Y5A7C9E1G3J",
  "source": "urn:aios:_platform:service:database",
  "type": "aios.database.migration.applied",
  "subject": "urn:aios:_platform:migration:01J9ZB0K3N5Q7R9T1V3X5Z7A9C",
  "time": "2026-07-22T14:03:47.912Z",
  "tenant": "_platform",
  "traceparent": "00-4bf92f3577b34da6a3ce929d0e0e4736-00f067aa0ba902b7-01",
  "datacontenttype": "application/json",
  "dataschema": "aios://schemas/database.migration.applied/1",
  "data": {
    "version": "20260722T140000_add_memory_tag",
    "ownerModule": "010-Memory",
    "phase": "expand",
    "upSqlDigest": "9f2c…a71b",
    "durationMs": 1812,
    "blockingMs": 40,
    "appliedBy": "urn:aios:_platform:service:ci-pipeline",
    "objectsAffected": ["memory.tag"]
  }
}
```

### 3.2 `aios.database.retention.purged`

`dataschema`: `aios://schemas/database.retention.purged/1`

```json
{
  "data": {
    "schemaName": "kernel",
    "tableName": "syscall_log",
    "method": "drop_partition",
    "partition": "kernel.syscall_log_2026_04_23",
    "rowsRemoved": 42104883,
    "bytesReclaimed": 8912896000,
    "policyName": "syscall-log-90d",
    "cutoff": "2026-04-23T00:00:00Z"
  }
}
```

### 3.3 `aios.database.erasure.completed`

`dataschema`: `aios://schemas/database.erasure.completed/1`

```json
{
  "data": {
    "requestUrn": "urn:aios:acme:erasurerequest:01J9ZB3P6R8T0V2X4Z6B8D0F2H",
    "subjectUrn": "urn:aios:acme:agent:01J9Z8Q6H7K2M4N6P8R0S2T4V6",
    "scope": "subject",
    "tablesAffected": ["memory.item", "memory.embedding", "context.window", "graph.vertex"],
    "rowsErased": 18402,
    "tokenizedRows": 37,
    "receiptHash": "sha256:1b8c…44df",
    "completedAt": "2026-07-22T14:22:09.441Z"
  }
}
```

> **Minimização (RFC-0001 §7):** o payload traz identificadores opacos (URN),
> contagens e hashes — nunca o dado pessoal removido.

### 3.4 `aios.database.compliance.violated`

`dataschema`: `aios://schemas/database.compliance.violated/1`

```json
{
  "data": {
    "schemaName": "tools",
    "tableName": "invocation",
    "violations": ["RLS_DISABLED", "RETENTION_MISSING"],
    "ownerModule": "015-Tool-Manager",
    "detectedAt": "2026-07-22T03:15:00Z",
    "severity": "P1"
  }
}
```

### 3.5 `aios.database.replication.lagging`

`dataschema`: `aios://schemas/database.replication.lagging/1`

```json
{
  "data": {
    "replica": "aios-postgres-replica-2",
    "lagMs": 4820,
    "thresholdMs": 1000,
    "readEligible": false,
    "lastLsn": "3F/A8012C40"
  }
}
```

---

## 4. Eventos Consumidos

| Subject assinado | Produtor | Ação do módulo | Consumidor durável |
|-------------------|----------|----------------|--------------------|
| `aios._platform.cluster.failover.requested` | `../027-Cluster/` | `ReplicationSupervisor` coordena promoção e reconfigura o pool. | `db-failover` |
| `aios._platform.deployment.rollout.started` | `../028-Deployment/` | Congela migrações `contract` durante a janela de rollout. | `db-rollout-guard` |
| `aios.<tenant>.policy.bundle.updated` | `../022-Policy/` | Invalida o cache de decisões do `PolicyClient`. | `db-policy-cache` |
| `aios.<tenant>.cost.budget.updated` | `../026-Cost-Optimizer/` | Atualiza o limite de armazenamento do tenant (`AIOS-DB-0012`). | `db-storage-quota` |
| `aios.<tenant>.audit.legalhold.applied` | `../025-Audit/` | Ativa `legal_hold` e suspende expurgos das tabelas afetadas. | `db-legal-hold` |
| `aios.<tenant>.memory.item.forgotten` | `../010-Memory/` | Enfileira expurgo físico do item e de seus vetores. | `db-forget-relay` |

Todos os consumidores **DEVEM** ser idempotentes: reprocessar um `event.id` já visto
é operação nula.

---

## 5. Semântica de Entrega e Versionamento

| Aspecto | Regra |
|---------|-------|
| Entrega | At-least-once (JetStream). *Exactly-once efetivo* pela combinação Outbox + dedupe por `event.id`. |
| Ordenação | Garantida por stream. Eventos de migração de uma mesma tabela são serializados pelo advisory lock (I1). |
| Versionamento | `dataschema` versionado (`/1`, `/2`, …); evolução **aditiva**. Consumidores **DEVEM** tolerar campos desconhecidos (RFC-0001 §5.7). |
| Registro | Novos `dataschema` são registrados via `../004-API/` (RFC-0001 §8). |
| Falha de publicação | Evento permanece em `platform.outbox` com `published = false`; o relay reenvia até sucesso. |
| Privacidade | Payloads seguem minimização (RFC-0001 §7): sem PII, sem SQL de migração, sem conteúdo expurgado. |

---

## 6. Exemplo — Assinatura de um consumidor

```python
# Consumidor de sinais de conformidade (ex.: 025-Audit ou pipeline de segurança)
sub = await js.subscribe(
    "aios._platform.database.compliance.violated",
    durable="audit-db-compliance",
    manual_ack=True,
)

async for msg in sub.messages:
    event = json.loads(msg.data)
    if seen(event["id"]):          # dedupe obrigatório por event.id
        await msg.ack(); continue
    d = event["data"]
    if d["severity"] == "P1":
        page_oncall(f'{d["schemaName"]}.{d["tableName"]}: {",".join(d["violations"])}')
    mark_seen(event["id"])
    await msg.ack()
```

---

## 7. Referências

- Brief: `./_DESIGN_BRIEF.md` §6
- Contratos de evento: `../003-RFC/RFC-0001-Architecture-Baseline.md` §5.2–§5.3
- Barramento: `../020-Communication/` · Registro de schemas: `../004-API/`
- API: `./API.md` · Sequências: `./SequenceDiagrams.md`
