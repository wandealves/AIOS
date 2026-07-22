---
Documento: Security
Módulo: 005-Database
Status: Draft
Versão: 0.1
Última atualização: 2026-07-22
Responsável (RACI-A): Arquiteto do Módulo 005-Database
ADRs relacionados: ADR-0008, ADR-0010 (globais); ADR-0051, ADR-0052, ADR-0056, ADR-0059
RFCs relacionados: RFC-0001, RFC-0050
Depende de: _DESIGN_BRIEF.md §12, 021-Security, 022-Policy, 025-Audit, 004-API
---

# 005-Database — Segurança

Contratos de segurança centrais (mTLS interno, validação de token no Gateway,
`tenant` como fronteira de isolamento, minimização de dados) são normativos na
`../003-RFC/RFC-0001-Architecture-Baseline.md` §6–§7 e **não são redefinidos aqui**.

---

## 1. Autenticação (AuthN)

| Caminho | Mecanismo | Responsável |
|---------|-----------|-------------|
| Usuário/operador → API administrativa | OAuth2/OIDC validado no `../004-API/`; claims assinadas propagadas | `021-Security` + `004-API` |
| Serviço → `aios-database-svc` | **mTLS** na malha interna (RFC-0001 §6) | `021-Security` |
| Serviço → PostgreSQL (via PgBouncer) | Credencial da *role* do serviço, obtida do cofre; TLS obrigatório | `021-Security` |
| `aios-database-svc` → MinIO | Chave de acesso do cofre; TLS | `021-Security` |

O módulo **não emite nem valida tokens de usuário final** e **não custodia segredos** —
apenas os consome, com rotação sem downtime (dupla credencial durante a janela).

---

## 2. Autorização (AuthZ)

O módulo é **PEP**. Toda operação administrativa monta um `DecisionRequest` e consulta
o **PDP** do `../022-Policy/`, com ***default deny*** (RFC-0001 §5.8). Negativa →
`AIOS-DB-0002`.

| Capability | Operações | Sensibilidade |
|------------|-----------|---------------|
| `db:migration:submit` | submeter, validar, ensaiar | média |
| `db:migration:apply` | aplicar no primário | **alta** |
| `db:migration:rollback` | reverter migração reversível | alta |
| `db:schema:read` / `db:schema:audit` | ler catálogo / auditar conformidade | baixa |
| `db:retention:write` | alterar TTL / `legal_hold` | **alta** |
| `db:partition:read` / `db:partition:write` | listar / garantir partições | média |
| `db:erasure:execute` | direito ao esquecimento | **alta** |
| `db:backup:read` / `db:backup:verify` | catálogo / *restore drill* | média |
| `db:restore:execute` | PITR | **crítica — exige 2ª aprovação** |
| `db:vector:write` | criar/reindexar índice vetorial | média |
| `db:replication:read` | status de replicação | baixa |

Se o PDP estiver indisponível, o comportamento é ***fail-closed***: a operação é
negada, nunca permitida por omissão.

---

## 3. Privilégio Mínimo no Banco

| *Role* | Concessões | Quem usa |
|--------|-----------|----------|
| `<schema>_rw` | `SELECT/INSERT/UPDATE/DELETE` **apenas** no schema do próprio módulo | Serviços de runtime (`006`…`026`) |
| `<schema>_ro` | `SELECT` no próprio schema | Jobs de leitura, réplicas |
| `platform_ddl` | `CREATE/ALTER/DROP` em todos os schemas | **Somente** o `MigrationEngine`, durante migração autorizada |
| `platform_admin` | `BYPASSRLS` para consultas de plataforma legítimas | `aios-database-svc` (jobs de catálogo/retenção), **auditado** |

Regras normativas:

- Nenhum serviço de runtime **DEVE** ser `SUPERUSER` nem dono das tabelas.
- Nenhuma *role* de runtime **DEVE** ter `CREATE` em schema algum.
- O uso de `platform_admin` (`BYPASSRLS`) **DEVE** ser registrado em `../025-Audit/`
  com finalidade, e é revisado periodicamente.

---

## 4. Isolamento Multi-Tenant

A **RLS** por `tenant_id` é a fronteira física de isolamento e o invariante mais forte
do módulo (NFR-011). O padrão está em `./Database.md` §4. Propriedades relevantes para
segurança:

1. `FORCE ROW LEVEL SECURITY` aplica a política **inclusive ao dono da tabela**.
2. `WITH CHECK` impede *escrever* linha de outro tenant, não apenas lê-la.
3. Sessão sem `app.tenant_id` vê **zero linhas** — falha fechada, nunca aberta.
4. `tenant` divergente do contexto autenticado → `AIOS-DB-0003` (RFC-0001 §6).
5. Grafos AGE são isolados por grafo (`graph_<tenant>`), com acesso concedido só à
   *role* correspondente (FR-014).
6. Auditoria contínua (`POST /v1/database/schemas:audit`) detecta RLS desabilitada e
   emite `aios._platform.database.compliance.violated` com severidade **P1**.

---

## 5. Superfície de Ataque

| Superfície | Exposição | Controle |
|------------|-----------|----------|
| API administrativa `/v1/database` | Interna, via `004-API` | OIDC + PEP/PDP + auditoria |
| gRPC `aios.database.v1` | Malha interna | mTLS + PEP |
| Porta do PgBouncer | Malha interna | TLS + credencial por *role* + bulkhead |
| Porta do PostgreSQL | **Não exposta** fora da malha | `pg_hba` restrito ao PgBouncer e às réplicas |
| SQL de migração | Entrada controlada pelo dono do schema | Lint (CV-10), revisão, *dry-run*, capability `db:migration:apply` |
| Consulta Cypher (AGE) | Via módulo `019` | `max_traversal_depth`, `statement_timeout` |
| Artefatos de backup em MinIO | Objeto remoto | Cifragem em repouso e em trânsito, checksum, credencial dedicada |

**DDL proibido em migração** (regra `NoDangerousDdlRule`, CV-10 em `./Database.md`):
`DISABLE ROW LEVEL SECURITY`, `SECURITY DEFINER`, `CREATE/ALTER ROLE`, `GRANT`,
`ALTER SYSTEM`, `COPY … PROGRAM`, alteração global de `search_path`.

---

## 6. Threat Model — STRIDE

| Ameaça | Vetor no 005-Database | Mitigação | Verificação |
|--------|------------------------|-----------|-------------|
| **S**poofing | Serviço se conecta com credencial de outro serviço/tenant para ler dados alheios. | mTLS + *role* por serviço + credenciais rotacionadas no cofre (`021`); `tenant` da sessão casado com o contexto autenticado (`AIOS-DB-0003`). | T-SEC-02 |
| **T**ampering | Alteração direta de linhas, do histórico de migrações ou de políticas de retenção para encobrir ação. | Privilégio mínimo (sem DDL/`UPDATE` fora do schema próprio); `platform.migration` imutável em estado terminal (I5); toda operação administrativa auditada em `025`; checksums de página (`--data-checksums`). | T-SEC-04 |
| **R**epudiation | Negar ter aplicado uma migração destrutiva ou expurgado dados. | `applied_by` + `up_sql_digest` na migração; `receipt_hash` no expurgo; auditoria imutável com `trace_id` em `../025-Audit/`. | T-ERA-02 |
| **I**nformation disclosure | Vazamento entre tenants por query defeituosa, backup mal protegido ou PII em log de consulta. | **RLS** independente da aplicação; backups cifrados em repouso e em trânsito; o `QueryGovernor` **NÃO DEVE** registrar valores de parâmetros de consultas sobre tabelas `pii` — apenas o texto normalizado. | T-SEC-01 |
| **D**enial of service | Consulta de custo explosivo (travessia de grafo, *seq scan* em 10⁹ linhas), esgotamento de conexões, enchimento de disco. | `statement_timeout` (30 s), `db.graph.max_traversal_depth` (6), bulkhead de conexões (50/serviço), cota de armazenamento por tenant (`AIOS-DB-0012`), rejeição precoce com `Retry-After`. | T-CONN-02 |
| **E**levation of privilege | Migração maliciosa que cria função `SECURITY DEFINER`, desabilita RLS ou concede *role*. | `DdlConventionValidator` rejeita esse DDL (CV-10); aplicação exige `db:migration:apply`; auditoria contínua compara catálogo real × `schema_registry`. | T-MIG-03, T-SEC-03 |

---

## 7. Criptografia

| Dado | Em trânsito | Em repouso |
|------|-------------|------------|
| Conexões cliente ↔ PgBouncer ↔ PostgreSQL | TLS obrigatório | — |
| Replicação primário → réplica | TLS | — |
| Volumes `pgdata-*` | — | Cifragem do volume (`../028-Deployment/`) |
| WAL e backups em MinIO | TLS | Cifragem no bucket + checksum SHA-256 por artefato |
| Chamadas de serviço (gRPC) | mTLS (RFC-0001 §6) | — |

Cifragem em nível de coluna **NÃO** é usada por padrão: ela quebraria índices e busca
vetorial. Onde há necessidade de proteção adicional a um campo específico, o dono do
schema **DEVE** usar tokenização (mantendo a referência) e declarar o campo como `pii`.

---

## 8. LGPD / GDPR

| Requisito legal | Implementação |
|-----------------|---------------|
| **Minimização** | Payloads de eventos e logs contêm URNs opacos, contagens e hashes — nunca o dado pessoal (RFC-0001 §7, `./Events.md` §3.3). |
| **Classificação** | `schema_registry.data_class` ∈ {`public`, `internal`, `confidential`, `pii`}. |
| **Base legal** | Tabela `pii` **DEVE** declarar `legal_basis` na política de retenção; sem ela, a migração falha (`AIOS-DB-0009`). |
| **Retenção** | TTL declarado por tabela; expurgo preferencial por *drop de partição*; o registro do expurgo é auditável e não contém o dado expurgado. |
| **Direito ao esquecimento** | `POST /v1/database/erasure-requests` resolve todas as tabelas que referenciam o titular (incluindo vetores e vértices AGE), remove em ordem segura de FK e emite `erasure.completed` com `receipt_hash` (UC-009). |
| **Impossibilidade legal de apagar** | Aplica-se **tokenização irreversível** em vez de exclusão, registrada no comprovante. |
| **Legal hold** | `legal_hold` suspende qualquer expurgo (`AIOS-DB-0010`); a obrigação de preservar prevalece sobre a de apagar, e o conflito é registrado, nunca resolvido em silêncio. |
| **Segregação** | RLS + partição por tenant; backups herdam o escopo lógico e suportam restauração parcial por tenant. |
| **Portabilidade** | Exportação por tenant a partir de réplica, sob capability específica e auditoria. |

---

## 9. Auditoria

Toda operação administrativa gera registro em `../025-Audit/` com, no mínimo:
`trace_id`, identidade do chamador (URN), capability avaliada e decisão do PDP,
recurso alvo, resultado e duração. Adicionalmente:

- Migrações registram `up_sql_digest` e `applied_by` (não repudiabilidade).
- Expurgos registram `receipt_hash`, `rows_erased` e `tables_affected`.
- Uso de `platform_admin` (`BYPASSRLS`) é registrado com finalidade declarada.
- Restaurações PITR registram alvo, aprovadores e RTO medido.

---

## 10. Referências

- Brief: `./_DESIGN_BRIEF.md` §12
- Contratos de segurança e privacidade: `../003-RFC/RFC-0001-Architecture-Baseline.md` §6–§7
- Identidade e segredos: `../021-Security/` · Política: `../022-Policy/` · Auditoria: `../025-Audit/`
- Modelo físico e RLS: `./Database.md` §4 · Testes de segurança: `./Testing.md` §8
