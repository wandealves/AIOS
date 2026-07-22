---
Documento: Examples
Módulo: 005-Database
Status: Draft
Versão: 0.1
Última atualização: 2026-07-22
Responsável (RACI-A): Arquiteto do Módulo 005-Database
ADRs relacionados: ADR-0051, ADR-0053, ADR-0055, ADR-0057
RFCs relacionados: RFC-0001, RFC-0050, RFC-0051
Depende de: API.md, Database.md, Configuration.md, 030-CLI, 031-SDK
---

# 005-Database — Exemplos

Exemplos executáveis, do mais simples ao mais avançado. Endpoints e códigos de erro são
os de `./API.md`; as convenções físicas são as de `./Database.md` §2.

---

## 1. Hello World — criar uma tabela corretamente

Um módulo (`010-Memory`) quer uma tabela nova. O DDL abaixo passa no lint porque
respeita CV-01 (PK URN), CV-02 (`tenant_id` + RLS), CV-03 (OCC), CV-04 (`timestamptz`),
CV-05 (índice em FK) e CV-07 (retenção declarada).

```sql
-- migrations/20260722T140000_add_memory_tag.sql   (phase = expand)
CREATE TABLE memory.tag (
    urn        text        PRIMARY KEY,
    tenant_id  text        NOT NULL,
    item_urn   text        NOT NULL REFERENCES memory.item(urn) ON DELETE CASCADE,
    name       text        NOT NULL,
    version    bigint      NOT NULL DEFAULT 0,
    created_at timestamptz NOT NULL DEFAULT now(),
    updated_at timestamptz NOT NULL DEFAULT now(),
    CONSTRAINT uq_memory_tag UNIQUE (tenant_id, item_urn, name)
);

CREATE INDEX ix_memory_tag_item ON memory.tag (item_urn);   -- CV-05

ALTER TABLE memory.tag ENABLE ROW LEVEL SECURITY;
ALTER TABLE memory.tag FORCE  ROW LEVEL SECURITY;
CREATE POLICY p_memory_tag_tenant ON memory.tag
    USING      (tenant_id = current_setting('app.tenant_id', true))
    WITH CHECK (tenant_id = current_setting('app.tenant_id', true));
```

Declaração de retenção que acompanha a migração:

```bash
curl -sX PUT https://api.aios.local/v1/database/retention-policies/memory-tag-365d \
  -H "X-AIOS-Tenant: _platform" \
  -H "Content-Type: application/json" \
  -d '{ "ttl": "365d", "basisColumn": "created_at", "method": "delete_batch",
        "batchSize": 10000 }'
```

---

## 2. Ciclo completo de uma migração (CLI)

```bash
# 1. submeter
aios db migration submit \
  --file migrations/20260722T140000_add_memory_tag.sql \
  --owner 010-Memory --phase expand --reversible
# → urn:aios:_platform:migration:01J9ZB0K3N5Q7R9T1V3X5Z7A9C   state=Draft

# 2. validar (lint de convenções)
aios db migration validate urn:…:01J9ZB0K3N5Q7R9T1V3X5Z7A9C
# → state=Validated   blocking_estimate=15ms

# 3. ensaiar em réplica
aios db migration dry-run urn:…:01J9ZB0K3N5Q7R9T1V3X5Z7A9C
# → state=DryRun   duration=1.8s  measured_block=40ms  ✔ dentro de 200ms

# 4. aplicar
aios db migration apply urn:…:01J9ZB0K3N5Q7R9T1V3X5Z7A9C \
  --idempotency-key 01J9ZB1M4P6R8T0V2X4Z6B8D0F
# → state=Applied  duration=1812ms
#   evento aios._platform.database.migration.applied publicado

# 5. conferir o catálogo
aios db schema get memory.tag
# → owner_module=010-Memory  multi_tenant=true  rls_enabled=true
#   data_class=internal      retention=memory-tag-365d
```

---

## 3. O que acontece quando você esquece a RLS

```sql
-- DDL incompleto (sem policy, sem retenção)
CREATE TABLE tools.invocation (
    urn text PRIMARY KEY, tenant_id text NOT NULL, payload jsonb NOT NULL
);
```

```bash
aios db migration validate urn:…:01J9ZC1N4P6R8T0V2X4Z6B8D0F
```

```json
{
  "code": "AIOS-MIGRATION-0002",
  "title": "Migration Rejected by DDL Convention Lint",
  "status": 422,
  "detail": "tools.invocation: sem POLICY de RLS (RlsPolicyRequiredRule); retenção não declarada (RetentionDeclaredRule); coluna created_at ausente (TimestamptzRule).",
  "retriable": false
}
```

O custo do erro é um teste vermelho em 3 segundos — não um vazamento entre clientes
descoberto meses depois.

---

## 4. Expand/Contract — renomear uma coluna sem downtime

```sql
-- Passo 1 — 20260722T090000_expand_memory_content   (expand, reversível)
ALTER TABLE memory.item ADD COLUMN content text;
```
```csharp
// A aplicação passa a escrever nas duas colunas e ler de content com fallback
item.Content = value;
item.Body    = value;                      // temporário
var text = row.Content ?? row.Body;        // leitura tolerante
```
```sql
-- Passo 2 — 20260723T090000_migrate_memory_content   (migrate)
-- backfill em lotes, respeitando statement_timeout
UPDATE memory.item SET content = body
 WHERE content IS NULL AND urn IN (
   SELECT urn FROM memory.item WHERE content IS NULL LIMIT 10000
 );

-- Passo 3 — 20260801T090000_contract_memory_body    (contract, NÃO reversível)
ALTER TABLE memory.item DROP COLUMN body;
```

O passo 3 só é aplicado depois de **todas** as instâncias da aplicação estarem na
versão nova — e é recusado se houver rollout em andamento (`./Events.md` §4).

---

## 5. Busca vetorial com isolamento por tenant

```python
import psycopg

with psycopg.connect(DSN) as conn:          # DSN aponta para o PgBouncer
    with conn.cursor() as cur:
        cur.execute("SET app.tenant_id = %s", ("acme",))   # ativa a RLS
        cur.execute("SET hnsw.ef_search = %s", (64,))      # default do AIOS
        cur.execute(
            """
            SELECT urn, embedding <=> %s::vector AS distance
              FROM memory.item
             ORDER BY embedding <=> %s::vector
             LIMIT 10
            """,
            (query_vec, query_vec),
        )
        for urn, distance in cur.fetchall():
            print(urn, round(distance, 4))
```

Note que **não há `WHERE tenant_id = …`**: a RLS já o aplica. Se a aplicação esquecer
o `SET app.tenant_id`, o resultado é **zero linhas** — falha fechada, nunca vazamento.

---

## 6. Registrar e reindexar um índice vetorial

```bash
# registrar a especificação (dimensão é imutável depois disso)
aios db vector put memory.item.embedding \
  --dimensions 1536 --index hnsw --distance cosine --m 16 --ef-construction 64

# medir qualidade contra busca exata
aios db vector recall memory.item.embedding --sample 10000
# → recall@10 = 0.962   p99 = 38ms   ✔ NFR-006 e NFR-003

# depois de alterar m: reindexação online
aios db vector reindex memory.item.embedding
# → REINDEX CONCURRENTLY em andamento; escrita não bloqueada
```

Tentativa de mudar a dimensão:

```bash
aios db vector put memory.item.embedding --dimensions 3072
# → 409 AIOS-DB-0011: dimensão imutável; crie nova coluna e migre os dados
```

---

## 7. Direito ao esquecimento ponta a ponta

```bash
# 1. solicitar
aios db erasure request \
  --subject urn:aios:acme:agent:01J9Z8Q6H7K2M4N6P8R0S2T4V6 \
  --scope subject --idempotency-key 01J9ZB2N5Q7S9U1W3Y5A7C9E1G
# → urn:aios:acme:erasurerequest:01J9ZB3P6R8T0V2X4Z6B8D0F2H  status=authorized

# 2. acompanhar
aios db erasure get urn:aios:acme:erasurerequest:01J9ZB3P6R8T0V2X4Z6B8D0F2H
# → status=completed  rows_erased=18402  tokenized=37
#   tables=[memory.item, memory.embedding, context.window, graph.vertex]
#   receipt_hash=sha256:1b8c…44df
```

Sob `legal_hold`:

```json
{
  "code": "AIOS-DB-0010",
  "status": 423,
  "title": "Erasure Blocked by Legal Hold",
  "detail": "Política 'audit-7y' está sob legal hold; expurgo suspenso.",
  "retriable": false
}
```

A obrigação de preservar vence a de apagar, e o conflito fica registrado.

---

## 8. Operação: verificar backup e simular restauração

```bash
# catálogo e janela de PITR
aios db backup list --limit 5
# KIND         CREATED              SIZE     VERIFIED             LSN RANGE
# full         2026-07-22T02:00Z    412 GiB  2026-07-22T04:11Z    3F/00000000..3F/A8012C40
# incremental  2026-07-22T14:00Z     18 GiB  —                    3F/A8012C40..3F/C1004410

# restore drill sob demanda
aios db backup verify urn:aios:_platform:backup:01J9ZB5R8T0V2X4Z6B8D0F2H4K
# → restaurado em ambiente isolado; integridade OK; duração 9m12s
#   evento database.backup.verified publicado

# janela de PITR disponível
aios db replication status
# primary: aios-postgres-primary  (rw)
# replica-1: lag 78ms   read-eligible=true
# replica-2: lag 122ms  read-eligible=true
# rpo_observed: 34s     ✔ (meta ≤ 300s)
```

---

## 9. Auditoria de conformidade

```bash
aios db schema audit
```

```json
{
  "rlsCoverageRatio": 0.994,
  "violations": [
    { "schema": "tools", "table": "invocation",
      "violations": ["RLS_DISABLED", "RETENTION_MISSING"],
      "ownerModule": "015-Tool-Manager", "severity": "P1" }
  ],
  "tablesAudited": 168,
  "storageBytesTotal": 1043118915584
}
```

Cobertura diferente de **1,0** é incidente P1, não métrica de tendência
(`./Monitoring.md` §3).

---

## 10. Armadilha comum — estado de sessão com `pool_mode=transaction`

```csharp
// ❌ ERRADO: assume que a conexão é sua entre comandos
await conn.ExecuteAsync("SET app.tenant_id = 'acme'");
await conn.ExecuteAsync("SELECT * FROM memory.item");   // pode cair em OUTRA conexão

// ✅ CERTO: tudo dentro da mesma transação
await using var tx = await conn.BeginTransactionAsync();
await conn.ExecuteAsync("SET LOCAL app.tenant_id = 'acme'", transaction: tx);
var rows = await conn.QueryAsync<MemoryItem>("SELECT * FROM memory.item", transaction: tx);
await tx.CommitAsync();
```

`SET LOCAL` dentro da transação é a forma correta: a transação é a unidade de posse da
conexão no modo `transaction` (verificado por T-CONN-03 em `./Testing.md`).

---

## 11. Referências

- API: `./API.md` · Modelo físico: `./Database.md` · Configuração: `./Configuration.md`
- CLI: `../030-CLI/` · SDKs: `../031-SDK/`
- Perguntas frequentes: `./FAQ.md`
