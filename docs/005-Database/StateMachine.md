---
Documento: StateMachine
Módulo: 005-Database
Status: Draft
Versão: 0.1
Última atualização: 2026-07-22
Responsável (RACI-A): Arquiteto do Módulo 005-Database
ADRs relacionados: ADR-0052, ADR-0053
RFCs relacionados: RFC-0001, RFC-0051
Depende de: _DESIGN_BRIEF.md §4, API.md, Events.md, Database.md
---

# 005-Database — Máquina de Estados

A entidade com ciclo de vida governado neste módulo é a **migração de schema**
(`platform.migration`). As demais entidades de plataforma (`schema_registry`,
`retention_policy`, `partition_policy`, `vector_index`) são configurações versionadas
por OCC (`version`), sem FSM própria. `platform.erasure_request` possui um ciclo
simples, documentado em §5.

Esta é a reprodução detalhada de `./_DESIGN_BRIEF.md` §4; qualquer divergência é
defeito deste documento.

---

## 1. Estados (`MigrationState`)

| Estado | Descrição | Ação de entrada | Terminal? |
|--------|-----------|-----------------|-----------|
| `Draft` | Migração submetida pelo módulo dono; ainda não validada. | Calcula `up_sql_digest`/`down_sql_digest`; grava `owner_module` e `phase`. | não |
| `Validated` | Aprovada pelo `DdlConventionValidator`. | Registra o relatório de lint e `blocking_estimate_ms`. | não |
| `DryRun` | Aplicada com sucesso em réplica de ensaio e revertida. | Preenche `dry_run_at` e `duration_ms` medido. | não |
| `Applying` | Em aplicação no primário, sob *advisory lock* global. | Adquire `pg_advisory_lock`; abre transação DDL. | não |
| `Applied` | Aplicada com sucesso. | Atualiza `platform.schema_registry`; grava evento no Outbox; libera o lock. | **sim** |
| `Failed` | Falhou na validação, no ensaio ou na aplicação. | Preenche `failure_code`; libera o lock se detido. | **sim** |
| `RolledBack` | Migração reversível revertida deliberadamente. | Executa o script `down`; atualiza o registry. | **sim** |
| `Superseded` | Substituída por migração corretiva posterior. | Preenche `superseded_by`. | **sim** |

> Estados terminais **NÃO DEVEM** ser atualizados, exceto pelo preenchimento de
> `superseded_by` (RN-04 em `./FunctionalRequirements.md`).

---

## 2. Transições, gatilhos e guardas

| # | De → Para | Gatilho | Guarda (DEVE ser verdadeira) | Ação de saída |
|---|-----------|---------|------------------------------|----------------|
| T-01 | ∅ → `Draft` | `SubmitMigration` | `version` única ∧ `owner_module` registrado ∧ digest calculado. | Persiste a linha. |
| T-02 | `Draft` → `Validated` | `ValidateMigration` | Lint sem violação bloqueante ∧ toda tabela multi-tenant criada tem `tenant_id` + RLS + retenção declarada. | Anexa relatório de lint. |
| T-03 | `Draft` → `Failed` | validação reprovada | Violação de convenção (`AIOS-MIGRATION-0002`) ∨ `blocking_estimate_ms` > `db.migration.max_block_ms`. | Emite `migration.failed`. |
| T-04 | `Validated` → `DryRun` | `DryRunMigration` | Ensaio concluído em réplica sem erro ∧ `duration_ms` medido. | Reverte a transação de ensaio. |
| T-05 | `Validated`/`DryRun` → `Failed` | erro no ensaio | Script falhou ∨ excedeu `db.migration.timeout_ms`. | Emite `migration.failed`. |
| T-06 | `DryRun` → `Applying` | `ApplyMigration` | Capability `db:migration:apply` ∧ advisory lock obtido ∧ nenhuma outra migração em `Applying` ∧ (janela permitida ∨ migração não bloqueante). | Abre transação DDL. |
| T-07 | `Applying` → `Applied` | commit bem-sucedido | DDL commitado ∧ `schema_registry.schema_version` atualizado ∧ evento no Outbox. | Libera o lock; emite `migration.applied`. |
| T-08 | `Applying` → `Failed` | erro/timeout na aplicação | Transação revertida integralmente ∧ lock liberado ∧ `failure_code` preenchido. | Emite `migration.failed`. |
| T-09 | `Applied` → `RolledBack` | `RollbackMigration` | `reversible = true` ∧ capability `db:migration:rollback` ∧ nenhuma migração posterior dependente aplicada. | Emite `migration.rolledback`. |
| T-10 | `Applied`/`Failed` → `Superseded` | migração corretiva aplicada | Existe migração posterior em `Applied` referenciando esta em `superseded_by`. | Atualiza o vínculo. |

Quando o gatilho é recebido em um estado que não admite a transição, a operação
retorna `AIOS-MIGRATION-0003` (transição inválida) e **NÃO DEVE** alterar o estado.

---

## 3. Invariantes

| # | Invariante | Como é garantido | Violação |
|---|-----------|------------------|----------|
| I1 | No máximo **uma** migração em `Applying` por cluster. | `pg_advisory_lock` global — nunca convenção de processo. | `AIOS-MIGRATION-0001` |
| I2 | `Applying` é **atômica**: ou a transação DDL commita inteira, ou não há efeito residual. | DDL transacional do PostgreSQL. | Estado inconsistente ⇒ incidente P1 |
| I3 | Migração `contract` com `reversible = false` **NÃO PODE** ir a `RolledBack`. | Guarda de T-09. | `AIOS-MIGRATION-0006` |
| I4 | Toda transição para estado terminal emite **exatamente um** evento `aios._platform.database.migration.<acao>`. | Outbox transacional + dedupe por `event.id`. | Evento duplicado/ausente ⇒ defeito de relay |
| I5 | `up_sql_digest` de uma migração `Applied` é imutável. | Verificação de digest na aplicação. | `AIOS-MIGRATION-0008` |
| I6 | Nenhuma migração alcança `Applied` sem `DryRun` quando `db.migration.require_dry_run = true`. | Guarda de T-06. | `AIOS-MIGRATION-0007` |

---

## 4. Diagrama de estados (ASCII)

```
        SubmitMigration (T-01)
   ∅ ──────────────────────────▶ ┌─────────┐
                                  │  Draft  │
                                  └────┬────┘
                validate (T-02)        │        lint/limite reprovado (T-03)
             ┌────────────────────────┤─────────────────────────────┐
             ▼                         │                             ▼
       ┌───────────┐                   │                       ┌──────────┐
       │ Validated │                   │                       │  Failed  │◀── erro no ensaio (T-05)
       └─────┬─────┘                   │                       └────┬─────┘
             │ dry-run (T-04)          │                            │ forward-fix
             ▼                         │                            │  (T-10)
       ┌───────────┐                   │                            │
       │  DryRun   │                   │                            │
       └─────┬─────┘                   │                            │
             │ apply (T-06, advisory lock global)                   │
             ▼                                                       │
       ┌───────────┐   erro/timeout (T-08)                           │
       │ Applying  │──────────────────────────▶ Failed               │
       └─────┬─────┘                                                 │
             │ commit (T-07)                                         │
             ▼                                                       ▼
       ┌───────────┐   rollback (T-09, só se reversible)   ┌──────────────┐
       │  Applied  │──────────────────────────▶┌──────────┐│  Superseded  │ (terminal)
       └─────┬─────┘                            │RolledBack│└──────────────┘
             │  forward-fix (T-10)              │(terminal)│        ▲
             └──────────────────────────────────┴──────────┴────────┘
```

---

## 5. Ciclo de vida secundário — `ErasureRequest`

Não é uma FSM canônica (sem guardas complexas nem compensação), mas o estado é
persistido em `platform.erasure_request.status` e governa o UC-009.

| Estado | Significado | Próximos |
|--------|-------------|----------|
| `received` | Pedido registrado com `Idempotency-Key`. | `authorized`, `rejected` |
| `authorized` | PDP concedeu `db:erasure:execute`. | `executing`, `rejected` |
| `executing` | Remoção/tokenização em lotes, em ordem segura de FK. | `completed` |
| `completed` | `rows_erased` e `receipt_hash` gravados; evento emitido. | — (terminal) |
| `rejected` | Negado (capability ausente ou `legal_hold` ativo). | — (terminal) |

```
   received ──autorizado──▶ authorized ──inicia──▶ executing ──conclui──▶ completed
      │                          │
      └────── legal_hold ────────┴──────────────▶ rejected   (AIOS-DB-0010)
```

Retomada após interrupção é **idempotente**: o coordenador reprocessa apenas o que
resta do `subject_urn`, e remover o que já não existe é operação nula.

---

## 6. Mapa Estado → Evento → Erro

| Estado alcançado | Evento emitido | Erro típico ao tentar sair indevidamente |
|------------------|----------------|------------------------------------------|
| `Applied` | `aios._platform.database.migration.applied` | `AIOS-MIGRATION-0006` (rollback vedado) |
| `Failed` | `aios._platform.database.migration.failed` | `AIOS-MIGRATION-0003` (transição inválida) |
| `RolledBack` | `aios._platform.database.migration.rolledback` | `AIOS-MIGRATION-0003` |
| `Superseded` | — (implícito na migração corretiva) | `AIOS-MIGRATION-0003` |
| `completed` (erasure) | `aios.<tenant>.database.erasure.completed` | — |

---

## 7. Referências

- Brief: `./_DESIGN_BRIEF.md` §4
- API e catálogo de erros: `./API.md` · Eventos: `./Events.md`
- Casos de uso UC-002..UC-005 e UC-009: `./UseCases.md`
- Sequências correspondentes: `./SequenceDiagrams.md`
