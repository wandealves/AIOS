---
Documento: Vision
Módulo: 005-Database
Status: Draft
Versão: 0.1
Última atualização: 2026-07-22
Responsável (RACI-A): Arquiteto do Módulo 005-Database
ADRs relacionados: ADR-0001, ADR-0002, ADR-0005, ADR-0006, ADR-0008, ADR-0010 (globais); ADR-0050, ADR-0051, ADR-0053, ADR-0055 (deste módulo, a propor)
RFCs relacionados: RFC-0001, RFC-0050, RFC-0051
Depende de: 000-Vision, 001-Architecture, 021-Security, 022-Policy, 025-Audit, 027-Cluster
---

# 005-Database — Visão do Módulo

> "O subsistema de dados está para o AIOS assim como o *filesystem* e o *volume
> manager* estão para um sistema operacional: nenhum processo escolhe como suas
> páginas são gravadas, replicadas ou expiradas — ele apenas escreve, e o sistema
> garante o resto." — analogia condutora deste módulo (ver
> `../040-Glossary/Glossary.md`, termos **Control Plane**, **Tenant**, **Shard**).

---

## 1. Propósito do Módulo

O **005-Database** é o **subsistema de armazenamento** do AIOS: a plataforma de
persistência única sobre a qual todos os módulos do plano de controle (`006`–`019`,
`022`, `023`, `025`, `026`) guardam seu estado autoritativo. Ele corresponde à
camada de dados descrita em `../001-Architecture/Architecture.md` e é implementado
sobre **PostgreSQL 16 + pgvector + Apache AGE** (decisão herdada de **ADR-0005**),
com **PgBouncer** na frente e **MinIO** como destino de WAL/backup.

A tese central do módulo, estabelecida no `_DESIGN_BRIEF.md` §0, cabe em uma frase:

> *o módulo dono decide **o que** persistir; o 005 decide **como** isso é
> fisicamente armazenado, isolado, migrado, particionado, retido, recuperado e
> observado.*

Disso decorrem as quatro entregas do módulo:

1. **Modelo físico consolidado** — um schema por módulo dono, catalogado em
   `platform.schema_registry`, com convenções físicas obrigatórias (URN como PK,
   `tenant_id`, `version` para OCC, `timestamptz` em UTC) impostas por *linter*, não
   por convenção social.
2. **Isolamento multi-tenant no nível físico** — **Row-Level Security** por
   `tenant_id` como rede de segurança independente do código da aplicação.
3. **Evolução governada do esquema** — migrações forward-only no padrão
   *expand/contract*, com validação, ensaio em réplica e aplicação atômica sob
   *advisory lock* global.
4. **Durabilidade e ciclo de vida do dado** — particionamento, retenção, expurgo
   rastreável (LGPD/GDPR), WAL archiving contínuo, PITR e verificação periódica de
   restauração.

---

## 2. Problema que o Módulo Resolve

A Visão Global (`../000-Vision/Vision.md`) identifica limitações estruturais dos
sistemas de agentes atuais. O 005-Database responde a quatro delas na camada de dados:

| Limitação da Visão | Como o subsistema de dados endereça |
|--------------------|--------------------------------------|
| **Ausência de estado durável e auditável** | Agentes que "esquecem" ao reiniciar são um sintoma de estado guardado em processo. Aqui, todo estado de controle tem fonte da verdade transacional, com replicação síncrona no quórum e RPO ≤ 5 min — reiniciar um agente nunca é perder o que ele sabia. |
| **Governança fraca (P8)** | Isolamento entre tenants normalmente depende de cada query lembrar do `WHERE tenant_id = …`. RLS transforma isso em propriedade do banco: um defeito de aplicação vira *zero linhas retornadas*, não vazamento. |
| **Escala para milhões de agentes** | 10⁶ agentes produzem ~10⁹ linhas em tabelas de série. Sem particionamento e retenção por *drop de partição*, o índice ativo cresce sem limite e a latência degrada. O módulo mantém o *working set* proporcional ao presente, não ao histórico. |
| **Evolução sem downtime** | Um sistema com dezenas de módulos evoluindo em paralelo não sobrevive a migrações ad hoc. O pipeline expand/contract garante que schema antigo e novo coexistam durante o rollout, alinhado à regra de coexistência de versões da RFC-0001 §5.7. |

---

## 3. Escopo

### 3.1 Dentro do escopo (in scope)

- Catálogo de schemas, donos e classificação de dados (`platform.schema_registry`).
- Convenções físicas obrigatórias e seu *linter* de DDL (`DdlConventionValidator`).
- Políticas de RLS por `tenant_id` e *roles* de banco por serviço.
- Pipeline de migração governado (FSM em `StateMachine.md`).
- Particionamento (RANGE temporal, HASH por `tenant_id`), pré-criação e *drop*.
- Retenção, expurgo por TTL e direito ao esquecimento com comprovante.
- Backup full/incremental, WAL archiving, PITR e *restore drill*.
- Governança de `pgvector` (índices HNSW/IVFFlat, dimensão, recall) e de **Apache AGE**.
- Pooling, cota de conexões por serviço, `statement_timeout` e contenção de query.
- Contrato canônico da tabela `outbox` usada pelo padrão Outbox transacional.

### 3.2 Fora do escopo (out of scope)

Ver `_DESIGN_BRIEF.md` §1.3 para a lista normativa completa. Os pontos que mais
geram confusão:

- **A semântica do dado** — o que é um ACB, um item de memória ou um plano pertence
  ao módulo dono (`006`, `010`, `012`, …). O 005 hospeda e cataloga, não interpreta.
- **Estado quente volátil** (contadores de cota, cache de decisão) vive em **Redis**,
  fora deste módulo.
- **Objetos binários** (checkpoints, snapshots) vivem em **MinIO** sob `008`/`028` —
  exceto WAL e backups do próprio PostgreSQL.
- **Decisão de política** é do `022-Policy` (o 005 é PEP, nunca PDP).
- **Estratégia de failover do cluster** é do `027-Cluster`; o 005 executa a parte de dados.
- **Estratégia de embedding e ontologia de grafo** são de `010`/`018`/`019`.

### 3.3 Fronteira em diagrama

```
   módulos donos  ──DDL versionado──▶  005-Database  ──instancia/opera──▶  PostgreSQL 16
   (006…026)      ──queries────────▶  (convenções,                        + pgvector + AGE
                                       migração, RLS,                     PgBouncer · MinIO
                                       retenção, DR)
```

---

## 4. Personas

| Persona | Necessidade | O que o módulo entrega |
|---------|-------------|------------------------|
| **Engenheiro de módulo** (dono de `006`, `010`, …) | Criar e evoluir suas tabelas sem quebrar produção. | Pipeline de migração com lint, *dry-run* e aplicação atômica; erro claro antes do deploy, não depois. |
| **SRE / Operador de plataforma** | Saber se o dado está seguro e se o RPO é real. | Catálogo de backups com janela de PITR contínua, *restore drill* automatizado, métricas de lag e alertas. |
| **Encarregado de dados (DPO)** | Provar conformidade com LGPD/GDPR. | Classificação por tabela, base legal obrigatória em `pii`, expurgo com `receipt_hash`, `legal_hold`. |
| **Arquiteto de segurança** | Garantir que um bug de aplicação não vaze dados entre clientes. | RLS obrigatória e auditada continuamente; privilégio mínimo por *role*; DDL perigoso rejeitado no lint. |
| **Engenheiro de custo** (`026`) | Atribuir custo de armazenamento a quem o consome. | `bytes_estimate` por tabela e cota de armazenamento por tenant. |

---

## 5. Princípios de Design

| # | Princípio | Consequência prática |
|---|-----------|----------------------|
| P-01 | **A convenção é imposta pela máquina.** | Tabela multi-tenant sem `tenant_id`, sem RLS ou sem retenção declarada **NÃO DEVE** ser criada — a migração falha com `AIOS-MIGRATION-0002`. |
| P-02 | **RLS é rede de segurança, não otimização.** | O isolamento **NÃO DEVE** depender do `WHERE` da aplicação. Desabilitar RLS é violação P1 detectada por auditoria contínua. |
| P-03 | **Forward-only.** | Não se reescreve histórico de migração; corrige-se com uma nova migração. Rollback existe apenas para migrações declaradamente reversíveis. |
| P-04 | **Expirar é *drop*, não `DELETE`.** | Retenção sobre tabelas de série usa partição temporal; custo O(1), sem *bloat*, sem varredura. |
| P-05 | **Backup só existe se restaurar.** | Um backup não verificado por *restore drill* é tratado como inexistente (FR-012). |
| P-06 | **Falhar fechado.** | Sob dependência indisponível, o módulo prefere rejeitar escrita, recusar leitura obsoleta e negar operação administrativa a servir dado incorreto ou sem governança. |
| P-07 | **O working set é o presente.** | Particionamento e retenção mantêm o índice ativo pequeno mesmo com 10⁹ linhas históricas. |
| P-08 | **Privilégio mínimo no banco.** | Nenhum serviço de runtime é `SUPERUSER` nem dono das tabelas; DDL só sob a *role* do `MigrationEngine`, durante migração autorizada. |

---

## 6. Não-Objetivos Explícitos

1. **Não é um ORM nem uma camada de acesso a dados.** O módulo não fornece
   repositórios de domínio; cada módulo acessa o PostgreSQL com sua própria camada,
   sob as credenciais e convenções que o 005 governa.
2. **Não é um data warehouse.** Consultas analíticas de longo alcance pertencem ao
   `035-Benchmark`/pipelines externos; o cluster operacional é dimensionado para
   OLTP com busca vetorial.
3. **Não é um serviço de consulta para clientes externos.** Não há acesso direto ao
   banco a partir da borda — todo acesso externo passa por `../004-API/`.
4. **Não implementa lógica de negócio no banco.** Sem *stored procedures* de domínio;
   funções SQL limitam-se a utilidades de plataforma (particionamento, RLS, expurgo).
5. **Não decide multi-região.** Topologia geográfica e DR entre regiões são de
   `../027-Cluster/`; o 005 fornece as primitivas (replicação, PITR, restauração por tenant).

---

## 7. Relação com a Visão Global

O `../000-Vision/Vision.md` define o AIOS como um sistema operacional para agentes,
com metas de escala (10⁶ agentes), governança (toda ação privilegiada autorizada e
auditada) e recuperabilidade (**RTO ≤ 15 min, RPO ≤ 5 min**, requisito V-R10). O
005-Database é o módulo onde essas três metas se tornam propriedades físicas:

- **Escala** → particionamento + retenção + réplicas de leitura (`Scalability.md`).
- **Governança** → RLS + privilégio mínimo + PEP em toda operação administrativa
  (`Security.md`).
- **Recuperabilidade** → replicação síncrona + WAL archiving + PITR verificado
  (`FailureRecovery.md`).

Nenhum contrato central é redefinido aqui: identidade de recurso (URN), envelope de
evento, envelope de erro, idempotência, correlação e subjects vêm de
`../003-RFC/RFC-0001-Architecture-Baseline.md`; o vocabulário vem de
`../040-Glossary/Glossary.md`.

---

## 8. Critérios de Sucesso do Módulo

| # | Critério | Verificação |
|---|----------|-------------|
| S-01 | Zero vazamento entre tenants em auditoria e teste de penetração de RLS. | NFR-011, `Testing.md` |
| S-02 | Nenhuma migração causou indisponibilidade não planejada em 12 meses. | NFR-010, `Monitoring.md` |
| S-03 | Janela de PITR contínua, sem lacuna de LSN, com *restore drill* verde. | FR-011/FR-012, `FailureRecovery.md` |
| S-04 | Latência p99 estável (≤ 5 ms por PK) com 10⁹ linhas históricas em partições. | NFR-001/NFR-007, `Benchmark.md` |
| S-05 | 100% das tabelas `pii` com base legal declarada e TTL ativo. | FR-002/FR-009, `Security.md` |

---

## 9. Referências

- Visão global: `../000-Vision/Vision.md`
- Arquitetura global: `../001-Architecture/Architecture.md`
- Contratos centrais: `../003-RFC/RFC-0001-Architecture-Baseline.md`
- Glossário: `../040-Glossary/Glossary.md`
- Fonte única de verdade deste módulo: `./_DESIGN_BRIEF.md`
- Arquitetura do módulo: `./Architecture.md` · Modelo físico: `./Database.md`
