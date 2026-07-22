---
Documento: RFC
Módulo: 005-Database
Status: Draft
Versão: 0.1
Última atualização: 2026-07-22
Responsável (RACI-A): Arquiteto do Módulo 005-Database
ADRs relacionados: ADR-0050, ADR-0051, ADR-0052, ADR-0053
RFCs relacionados: RFC-0001, RFC-0050, RFC-0051
Depende de: 003-RFC, _DESIGN_BRIEF.md §11
---

# 005-Database — Índice de RFCs

RFCs são registradas em `../003-RFC/`. Este documento lista as que afetam o módulo e o
impacto de cada uma.

---

## 1. RFCs Consumidas (baseline)

| RFC | Título | Status | Impacto no 005-Database |
|-----|--------|--------|-------------------------|
| **RFC-0001** | Architecture Baseline & Core Contracts | Accepted | **Baseline herdada, não redefinida.** Fornece: identidade URN/ULID (§5.1) usada como PK; envelope CloudEvents (§5.2) e subjects (§5.3) usados pelo Outbox; envelope de erro `AIOS-<DOMINIO>-<NNNN>` (§5.4) para os domínios `DB` e `MIGRATION`; idempotência (§5.5) em toda mutação administrativa; correlação (§5.6) em logs/traces; versionamento (§5.7) que fundamenta o expand/contract; PEP/PDP e auditoria (§5.8); mTLS e isolamento por tenant (§6); minimização e retenção de dados pessoais (§7); registros IANA-like (§8) onde o domínio `database` será inscrito. |

Nenhum contrato acima é reescrito neste módulo. Sempre que um documento do 005 precisa
de um deles, **referencia a seção da RFC-0001**.

---

## 2. RFCs a Propor por Este Módulo

| RFC | Título | Escopo normativo | Relação com RFC-0001 | Status |
|-----|--------|------------------|----------------------|--------|
| **RFC-0050** | AIOS Data Platform Contract | Convenções físicas obrigatórias (CV-01..CV-12); padrão normativo de RLS; contrato da tabela `outbox`; classificação de dados (`data_class`) e obrigações associadas; catálogo `platform.schema_registry` como registro de propriedade de schema. | Estende a RFC-0001 para a camada física; **não** redefine URN, evento, erro nem correlação. | Proposed |
| **RFC-0051** | AIOS Migration Protocol | Formato de migração (`version`, `phase`, digests); FSM normativa (`MigrationState`, T-01..T-10, invariantes I1..I6); garantias de expand/contract; regras de exclusão mútua e orçamento de bloqueio; semântica de forward-fix e `superseded_by`. | Instancia a regra de coexistência de versões da RFC-0001 §5.7 no plano de dados. | Proposed |

---

## 3. Registros Alimentados por Este Módulo

Conforme RFC-0001 §8, os registros centrais são mantidos por `../004-API/`. O 005
contribui com:

| Registro | Contribuição do 005 | Documento |
|----------|---------------------|-----------|
| Domínios de subject NATS | Novo domínio **`database`** (ADR-0050). | `./Events.md` |
| Códigos de erro | Domínios **`DB`** (0001–0099) e **`MIGRATION`** (0001–0099). | `./API.md` §5 |
| Versões de `dataschema` | `database.migration.applied/1`, `database.migration.failed/1`, `database.migration.rolledback/1`, `database.schema.registered/1`, `database.partition.created/1`, `database.partition.dropped/1`, `database.backup.completed/1`, `database.backup.verified/1`, `database.replication.lagging/1`, `database.replication.promoted/1`, `database.compliance.violated/1`, `database.retention.purged/1`, `database.erasure.completed/1`, `database.storage.threshold/1`. | `./Events.md` §3 |
| Tipos de recurso URN | `migration`, `schemaobject`, `erasurerequest`, `backup` (sob avaliação da arquitetura-chefe, pois a RFC-0001 §5.1 enumera os tipos vigentes). | `./Database.md` §3 |

> **Ponto que exige ratificação.** A RFC-0001 §5.1 enumera os tipos de URN existentes
> (`agent`, `task`, `memory`, `tool`, `model`, `plan`, `workflow`, `policy`, `event`).
> Os quatro tipos acima são **novos** e DEVEM ser registrados via PR + ADR-0050 antes
> de o módulo ser promovido a `Stable`.

---

## 4. Processo

1. Uma especificação de contrato/protocolo **NÃO DEVE** ser criada dentro dos 26
   documentos: ela vira RFC em `../003-RFC/` e é referenciada aqui.
2. RFCs deste módulo usam a numeração alinhada à faixa de ADR (`RFC-0050`, `RFC-0051`, …).
3. Alteração incompatível em RFC publicada exige nova RFC que a *obsolete*, com período
   de coexistência (RFC-0001 §9).
4. Ao mudar o status de uma RFC, atualize esta tabela e o índice em `../003-RFC/`.

---

## 5. Referências

- Baseline: `../003-RFC/RFC-0001-Architecture-Baseline.md`
- Decisões: `./ADR.md` · `../002-ADR/README.md`
- Registros centrais: `../004-API/`
- Brief: `./_DESIGN_BRIEF.md` §11
