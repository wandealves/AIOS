---
Documento: UseCases
Módulo: 005-Database
Status: Draft
Versão: 0.1
Última atualização: 2026-07-22
Responsável (RACI-A): Arquiteto do Módulo 005-Database
ADRs relacionados: ADR-0050, ADR-0051, ADR-0053, ADR-0054, ADR-0055, ADR-0056, ADR-0057
RFCs relacionados: RFC-0001, RFC-0050, RFC-0051
Depende de: FunctionalRequirements.md, API.md, StateMachine.md, 022-Policy, 025-Audit, 027-Cluster
---

# 005-Database — Casos de Uso

Formato: ator / pré-condições / fluxo principal / fluxos alternativos / exceções /
pós-condições. Todos os códigos de erro citados são os de `./API.md` §5.

---

## UC-001 — Registrar novo objeto de schema

- **Ator primário:** Engenheiro de módulo (via migração).
- **Requisitos:** FR-001.
- **Pré-condições:** módulo dono registrado; migração `Applied` que criou a tabela.

**Fluxo principal**
1. O `MigrationEngine` conclui a aplicação (T-07) de uma migração que cria tabela.
2. O `SchemaRegistry` insere a linha em `platform.schema_registry` com `owner_module`,
   `multi_tenant`, `data_class`, `partition_strategy` e `retention_ref`.
3. O `RlsPolicyManager` confirma no catálogo do PostgreSQL que a RLS está ativa e
   grava `rls_enabled = true`.
4. O `EventEmitter` publica `aios._platform.database.schema.registered`.

**Fluxos alternativos**
- **A1** — a tabela já existe no catálogo: a operação é idempotente e apenas
  atualiza `schema_version`.

**Exceções**
- **E1** — `data_class = 'pii'` sem `retention_ref`: `AIOS-DB-0009`; a migração
  não teria passado no lint (FR-002), logo esta exceção indica bypass e gera alerta P1.

**Pós-condições:** catálogo coerente com o estado real do banco.

---

## UC-002 — Submeter e validar migração

- **Ator primário:** Engenheiro de módulo.
- **Requisitos:** FR-002, FR-017.
- **Pré-condições:** `version` inédita; script `up` disponível.

**Fluxo principal**
1. O ator envia `POST /v1/database/migrations` com `version`, `ownerModule`, `phase`
   e o SQL. O serviço cria a linha em `Draft` (T-01) e calcula `up_sql_digest`.
2. O ator chama `:validate`. O `PolicyClient` consulta o PDP (capability
   `db:migration:submit`).
3. O `DdlConventionValidator` analisa o SQL: PK no padrão URN, `tenant_id` + RLS +
   retenção em tabela multi-tenant, `timestamptz`, índice em FK, ausência de DDL
   perigoso (`DISABLE ROW LEVEL SECURITY`, `SECURITY DEFINER`, `GRANT` de *role*).
4. Sem violação bloqueante → `Validated` (T-02).

**Fluxos alternativos**
- **A1** — migração puramente aditiva e não bloqueante: `blocking_estimate_ms = 0`,
  segue direto para o ensaio.

**Exceções**
- **E1** — violação de convenção → `Failed` (T-03) com `AIOS-MIGRATION-0002`.
- **E2** — `blocking_estimate_ms` > `db.migration.max_block_ms` (200) → `Failed`,
  com orientação de reescrever no padrão expand/contract.
- **E3** — `version` duplicada → `AIOS-MIGRATION-0004`.
- **E4** — capability negada → `AIOS-DB-0002`.

**Pós-condições:** migração em `Validated` ou `Failed`; decisão auditada em `../025-Audit/`.

---

## UC-003 — Ensaiar migração em réplica (*dry-run*)

- **Ator primário:** Engenheiro de módulo / pipeline de CI.
- **Requisitos:** FR-004.
- **Pré-condições:** migração em `Validated`; réplica de ensaio disponível.

**Fluxo principal**
1. `POST /v1/database/migrations/{urn}:dry-run`.
2. O `MigrationEngine` abre transação em uma réplica promovida a ambiente de ensaio,
   executa o script, mede `duration_ms` e o bloqueio observado, e **reverte**.
3. Sucesso → `DryRun` (T-04) com `dry_run_at` preenchido.

**Exceções**
- **E1** — erro de execução ou `db.migration.timeout_ms` excedido → `Failed` (T-05)
  com `AIOS-MIGRATION-0005`.
- **E2** — bloqueio medido acima do orçamento → `Failed`, mesma orientação de E2 do UC-002.

**Pós-condições:** custo real da migração conhecido antes de tocar o primário.

---

## UC-004 — Aplicar migração no primário

- **Ator primário:** Engenheiro de módulo (ou pipeline de deploy).
- **Requisitos:** FR-003, FR-004, FR-017, FR-018.
- **Pré-condições:** migração em `DryRun` (ou `Validated`, se `require_dry_run=false`).

**Fluxo principal**
1. `POST /v1/database/migrations/{urn}:apply` com `Idempotency-Key`.
2. O `PolicyClient` verifica a capability `db:migration:apply` no PDP → `allow`.
3. O `MigrationEngine` adquire o `pg_advisory_lock` global (espera até
   `db.migration.lock_timeout_ms`).
4. Estado → `Applying` (T-06). Em **uma transação**: executa o DDL, atualiza
   `platform.schema_registry`, grava o evento em `platform.outbox` e faz `COMMIT`.
5. Estado → `Applied` (T-07); lock liberado; relay publica
   `aios._platform.database.migration.applied`.

**Fluxos alternativos**
- **A1** — repetição com a mesma `Idempotency-Key`: retorna o resultado original,
  sem reaplicar (RFC-0001 §5.5).
- **A2** — rollout em andamento (`deployment.rollout.started` recebido) e fase
  `contract`: a aplicação é recusada até o fim da janela.

**Exceções**
- **E1** — lock ocupado → `AIOS-MIGRATION-0001` (retriable).
- **E2** — erro/timeout na aplicação → transação revertida integralmente; `Failed`
  (T-08) com `failure_code`; lock liberado.
- **E3** — *dry-run* ausente com exigência ativa → `AIOS-MIGRATION-0007`.
- **E4** — digest divergente do registrado → `AIOS-MIGRATION-0008`.

**Pós-condições:** schema evoluído atomicamente; evento emitido exatamente uma vez
(dedupe por `event.id`).

---

## UC-005 — Reverter migração reversível

- **Ator primário:** SRE de plantão.
- **Requisitos:** FR-005.
- **Pré-condições:** migração em `Applied` com `reversible = true`.

**Fluxo principal**
1. `POST /v1/database/migrations/{urn}:rollback` com justificativa.
2. PDP verifica `db:migration:rollback`; o engine confirma que **não** há migração
   posterior dependente da mesma tabela.
3. Executa o script `down` sob o mesmo advisory lock → `RolledBack` (T-09).
4. Publica `aios._platform.database.migration.rolledback`.

**Exceções**
- **E1** — migração não reversível (fase `contract` destrutiva) ou com dependentes →
  `AIOS-MIGRATION-0006`. A orientação normativa é **forward-fix**: nova migração
  corretiva, que marcará a original como `Superseded` (T-10).

**Pós-condições:** schema no estado anterior; histórico preservado (nada é apagado).

---

## UC-006 — Auditar conformidade de schemas

- **Ator primário:** Arquiteto de segurança / job periódico.
- **Requisitos:** FR-001, FR-006, FR-019, FR-020.
- **Pré-condições:** catálogo populado.

**Fluxo principal**
1. `POST /v1/database/schemas:audit` (ou execução agendada).
2. O serviço compara o catálogo real do PostgreSQL com `platform.schema_registry`:
   RLS ativa em toda tabela `multi_tenant`, retenção declarada, `outbox` com índice
   parcial, classificação `pii` com `legal_basis`.
3. Atualiza `row_estimate`/`bytes_estimate` (insumo de custo para `../026-Cost-Optimizer/`).
4. Divergência → evento `aios._platform.database.compliance.violated` + alerta.

**Fluxos alternativos**
- **A1** — nenhum desvio: emite apenas a métrica de conformidade (100%).

**Exceções**
- **E1** — tabela multi-tenant com RLS desabilitada: incidente **P1**; migração
  emergencial reabilita a política; acesso recente é auditado retroativamente.

**Pós-condições:** desvio detectado em ≤ 1 ciclo de auditoria.

---

## UC-007 — Garantir partições

- **Ator primário:** `PartitionManager` (job agendado).
- **Requisitos:** FR-007.
- **Pré-condições:** `platform.partition_policy` definida para a tabela.

**Fluxo principal**
1. O job verifica quantas partições futuras existem para cada tabela `range_time`.
2. Cria as faltantes até `precreate_ahead` (default 7) e emite
   `aios._platform.database.partition.created`.
3. Identifica partições além do TTL, faz `DETACH`, aguarda `db.partition.drop_grace_h`
   (default 24) e então `DROP`, emitindo `partition.dropped`.

**Exceções**
- **E1** — `legal_hold` ativo na política de retenção: o `DROP` é suspenso e a
  partição permanece anexada, com alerta informativo.
- **E2** — falha ao criar partição (disco cheio): `AIOS-DB-0012`; alerta P1 —
  ausência de partição futura significa **falha de escrita** ao virar o período.

**Pós-condições:** sempre há destino para a escrita do próximo período.

---

## UC-008 — Executar retenção por TTL

- **Ator primário:** `RetentionEnforcer` (job a cada `db.retention.run_interval_min`).
- **Requisitos:** FR-008, FR-010, FR-018.

**Fluxo principal**
1. Para cada tabela com política, calcula o corte por `basis_column` e `ttl`.
2. Método `drop_partition` (preferencial) → delega ao `PartitionManager` (UC-007).
3. Método `delete_batch` → remove em lotes de `batch_size` (default 10.000),
   respeitando `statement_timeout`.
4. Emite `aios.<tenant>.database.retention.purged` com volume expurgado.

**Exceções**
- **E1** — `legal_hold` → nenhuma remoção; registro do conflito (FR-010).
- **E2** — janela de execução excedida: o job pausa e retoma no próximo ciclo (o
  progresso é idempotente por natureza).

**Pós-condições:** nenhum dado além do TTL declarado, salvo sob *hold*.

---

## UC-009 — Atender pedido de direito ao esquecimento

- **Ator primário:** Encarregado de dados (DPO), via `004-API`.
- **Requisitos:** FR-009, FR-010, FR-017.
- **Pré-condições:** titular identificado por `subject_urn`.

**Fluxo principal**
1. `POST /v1/database/erasure-requests` com `subject_urn`, `scope` e `Idempotency-Key`
   → estado `received`.
2. PDP autoriza `db:erasure:execute` → `authorized`.
3. O `ErasureCoordinator` resolve o conjunto de tabelas que referenciam o titular
   (incluindo colunas `pgvector` e vértices/arestas AGE) e grava em `tables_affected`.
4. Executa remoção em ordem segura de FK, em lotes; onde a remoção física é vedada por
   obrigação legal, aplica **tokenização irreversível** → `executing`.
5. Conclui: `rows_erased`, `receipt_hash`, estado `completed`, evento
   `aios.<tenant>.database.erasure.completed` e registro em `../025-Audit/`.

**Exceções**
- **E1** — `legal_hold` ativo → `rejected` com `AIOS-DB-0010`.
- **E2** — interrupção no meio: o pedido permanece em `executing` e é retomado de
  forma idempotente pelo restante do `subject_urn`.
- **E3** — capability negada → `AIOS-DB-0002`.

**Pós-condições:** dado pessoal removido ou tokenizado, com comprovante verificável.

---

## UC-010 — Executar e verificar backup

- **Ator primário:** `BackupOrchestrator` (agendado).
- **Requisitos:** FR-011, FR-012, FR-018.

**Fluxo principal**
1. A cada `db.backup.full_interval_h` (24) executa backup full; WAL é arquivado
   continuamente (limite de `db.backup.wal_archive_timeout_s` = 60 s).
2. Cataloga em `platform.backup_catalog` com `checksum`, `lsn_start`/`lsn_end` e
   `expires_at`; emite `backup.completed`.
3. A cada `db.backup.verify_interval_h` (168) restaura o backup em ambiente isolado,
   valida integridade e emite `backup.verified`.

**Exceções**
- **E1** — falha no *restore drill* → backup marcado como não verificado, alerta **P1**
  e novo backup full imediato. Um backup não verificado **NÃO DEVE** ser considerado
  válido para fins de RTO/RPO.
- **E2** — arquivamento de WAL sem progresso → o RPO está degradado; alerta imediato.

**Pós-condições:** janela de PITR contínua e comprovadamente restaurável.

---

## UC-011 — Restaurar a um ponto no tempo (PITR)

- **Ator primário:** SRE de plantão, com aprovação dupla.
- **Requisitos:** FR-011, FR-017.
- **Pré-condições:** incidente declarado; alvo (`timestamp` ou LSN) definido.

**Fluxo principal**
1. `POST /v1/database/restore` com alvo, escopo e `Idempotency-Key`.
2. PDP verifica `db:restore:execute` **e** a segunda aprovação (capability distinta).
3. Restaura o último full anterior ao alvo e reaplica WAL até o LSN escolhido.
4. Reconfigura o `ConnectionPoolGateway` e reabre a escrita; registra tudo em `025`.

**Exceções**
- **E1** — alvo fora da janela de PITR → `AIOS-DB-0001`.
- **E2** — artefato corrompido (checksum divergente) → aborta e escala para o backup
  anterior verificado.

**Pós-condições:** cluster consistente no ponto escolhido; **RTO ≤ 15 min**.

---

## UC-012 — Promover réplica em failover

- **Ator primário:** `027-Cluster` (ator de sistema).
- **Requisitos:** FR-016.

**Fluxo principal**
1. `027` publica `aios._platform.cluster.failover.requested`.
2. O `ReplicationSupervisor` escolhe a réplica com menor lag, coordena a promoção e
   reconfigura o PgBouncer para o novo primário.
3. Emite `aios._platform.database.replication.promoted`.

**Exceções**
- **E1** — nenhuma réplica dentro do lag aceitável: promoção é feita mesmo assim sob
  decisão explícita do `027`, com registro do RPO efetivamente perdido.

**Pós-condições:** escrita restabelecida; réplicas antigas reconstruídas.

---

## UC-013 — Especificar e reindexar índice vetorial

- **Ator primário:** Engenheiro de `010-Memory` / `018-Knowledge`.
- **Requisitos:** FR-013, FR-014.

**Fluxo principal**
1. `PUT /v1/database/vector-indexes/{schema}/{table}/{column}` com `dimensions`,
   `index_type` (default `hnsw`), `distance_op` e `params`.
2. O `VectorIndexManager` cria o índice com `CREATE INDEX CONCURRENTLY`.
3. Job periódico mede Recall@10 contra busca exata amostrada (meta ≥ 0,95, NFR-006).
4. Quando parâmetros mudam, `:reindex` executa `REINDEX ... CONCURRENTLY`.

**Exceções**
- **E1** — tentativa de alterar `dimensions` → `AIOS-DB-0011`; a mudança exige nova
  coluna e migração de dados.
- **E2** — recall abaixo da meta → alerta e recomendação de aumentar `ef_search`
  (troca latência por qualidade).

**Pós-condições:** índice ativo com qualidade medida.

---

## UC-014 — Conceder acesso e aplicar bulkhead de conexões

- **Ator primário:** Serviço consumidor (`006`, `010`, …).
- **Requisitos:** FR-015, FR-016.

**Fluxo principal**
1. O serviço conecta ao PgBouncer com a credencial de sua *role*, obtida do cofre (`021`).
2. O `ConnectionPoolGateway` aplica `max_connections_per_service` (default 50) e
   `statement_timeout` (default 30.000 ms), e roteia leitura elegível para réplica.
3. A sessão fixa o `tenant_id`, ativando a RLS.

**Exceções**
- **E1** — cota de conexões esgotada → `AIOS-DB-0006` com `Retry-After`.
- **E2** — lag da réplica acima do limiar → `AIOS-DB-0013` ou redirecionamento ao primário.
- **E3** — `tenant` divergente do contexto autenticado → `AIOS-DB-0003`.

**Pós-condições:** nenhum serviço consegue esgotar o pool dos demais.

---

## UC-015 — Conter consulta patológica

- **Ator primário:** `QueryGovernor` (ator de sistema).
- **Requisitos:** FR-015.

**Fluxo principal**
1. Consulta ultrapassa `db.query.slow_threshold_ms` (500) → o plano é capturado.
2. Ultrapassando `statement_timeout` (30 s), a consulta é abortada com `AIOS-DB-0007`.
3. O governor correlaciona por `queryid`, detecta regressão de plano e recomenda
   índice ou `ANALYZE` no relatório do schema.

**Fluxos alternativos**
- **A1** — travessia AGE acima de `db.graph.max_traversal_depth` (6): rejeitada antes
  de executar, como contenção de DoS.

**Exceções**
- **E1** — consulta legítima e recorrente acima do orçamento: o dono do schema DEVE
  abrir migração de índice; ajustar o timeout global **NÃO DEVERIA** ser a resposta.

**Pós-condições:** um tenant não degrada a latência dos demais.

---

## Referências

- Requisitos: `./FunctionalRequirements.md` · `./NonFunctionalRequirements.md`
- Sequências: `./SequenceDiagrams.md` · FSM: `./StateMachine.md` · API: `./API.md`
- Brief: `./_DESIGN_BRIEF.md`
