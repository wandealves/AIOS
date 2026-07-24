---
Documento: Design Brief Interno (Fonte Única de Verdade do Módulo)
Módulo: 025-Audit
Status: Draft
Versão: 0.1
Última atualização: 2026-07-23
Responsável (RACI-A): Arquiteto do Módulo 025-Audit
ADRs relacionados: ADR-0001, ADR-0002, ADR-0004, ADR-0005, ADR-0008, ADR-0010 (globais, herdados); ADR-0250..ADR-0259 (deste módulo, a propor)
RFCs relacionados: RFC-0001 (baseline, consumida — não redefinida); RFC-0250 (Audit Record & Chain Integrity Contract), RFC-0251 (Retention, Legal Hold & Crypto-Erasure), a propor por este módulo
Depende de: 001-Architecture, 003-RFC (RFC-0001), 005-Database (schema `audit`), 020-Communication (NATS/JetStream), 021-Security (assinatura, chaves por titular, identidade), 022-Policy (PDP), 024-Observability (fronteira telemetria × prova), 027-Cluster, 028-Deployment, 029-Operations; consumido por **todos** os módulos, que emitem registro de auditoria por construção (RFC-0001 §5.8, ADR-0010)
---

# 025-Audit — Design Brief (Fonte Única de Verdade)

> **Escopo deste documento.** Este brief é a fonte única de verdade **interna** do
> módulo 025-Audit. Os 26 documentos obrigatórios (ver `README.md` e
> `../_templates/MODULE_TEMPLATE.md`) DEVEM ser derivados deste brief e **NÃO PODEM
> contradizê-lo**. Onde um contrato central já existe (URN, envelope de evento,
> envelope de erro, idempotência, correlação, **obrigatoriedade de registro imutável
> para toda ação privilegiada**), este documento **referencia** a
> `../003-RFC/RFC-0001-Architecture-Baseline.md` §5.8 e §7 e **não os redefine**.

> **Palavras normativas** conforme RFC 2119 / RFC 8174: DEVE, NÃO DEVE, DEVERIA,
> NÃO DEVERIA, PODE.

---

## 0. Posição no AIOS e contrato de fronteira

```
   ┌──────────────────────────────────────────────────────────────────────────┐
   │  TODOS OS MÓDULOS — emitem registro de auditoria por construção          │
   │  022 decisões · 021 credenciais · 006 syscalls · 009 escalonamento ·     │
   │  005 expurgos · 024 silêncios e SLOs · 026 gastos · 028 rollouts         │
   └───────────────┬──────────────────────────────────────────────────────────┘
                   │ evento (JetStream) ou escrita direta (API), com Idempotency-Key
                   ▼
   ┌──────────────────────────────────────────────────────────────────────────┐
   │  025-AUDIT — TRILHA IMUTÁVEL (este módulo)                               │
   │  · ENCADEIA por hash e SELA periodicamente (prova de não-adulteração)    │
   │  · verifica INTEGRIDADE e COMPLETUDE (lacuna é anomalia, não silêncio)   │
   │  · aplica RETENÇÃO LEGAL, LEGAL HOLD e APAGAMENTO CRIPTOGRÁFICO         │
   │  · exporta pacote probatório para auditor externo                        │
   │  · NÃO decide, NÃO alerta, NÃO é fonte de métrica operacional            │
   └───────────────┬──────────────────────────────────────────────────────────┘
                   │ prova verificável · comprovantes · legal hold
                   ▼
   ┌──────────────────────────────────────────────────────────────────────────┐
   │  005-DATABASE   — suspende expurgo ao receber `audit.legalhold.applied`  │
   │  024-OBSERVAB.  — telemetria: amostrável e perecível (fronteira ADR-0246)│
   │  029-OPERATIONS — investigação e resposta; auditor externo               │
   └──────────────────────────────────────────────────────────────────────────┘
```

**Fronteira em uma frase:** *o `024-Observability` responde **o que está acontecendo
agora e como o sistema se comporta**; o 025 responde **o que aconteceu e pode ser
provado**.* A telemetria é amostrável, tem retenção curta e pode perder dados sob
pressão; a trilha de auditoria não pode nenhuma das três coisas — e é essa diferença,
não o formato do dado, que justifica dois módulos.

---

## 1. Responsabilidades e Não-Responsabilidades

### 1.1 Missão

O 025-Audit é a **memória probatória** do AIOS — o análogo do *journal* de um sistema
de arquivos com garantia de integridade, elevado a um requisito de conformidade: não
basta que o registro exista, é preciso **provar** que ele não foi alterado desde que
foi escrito, e provar isso a alguém que não confia no operador do sistema.

A primeira tese do módulo é que **imutabilidade declarada não é imutabilidade**. Uma
tabela com "não altere" no comentário é uma tabela alterável por quem tem `UPDATE`. A
imutabilidade aqui é **verificável por terceiros**: cada registro carrega o hash do
anterior (cadeia), selos periódicos publicam a raiz de Merkle assinada, e qualquer
auditor pode recomputar a cadeia e detectar a divergência **sem confiar no AIOS**.

A segunda tese é que **a lacuna é o defeito mais perigoso**. Um registro adulterado
quebra o hash e é detectado; um registro que **nunca chegou** não deixa rastro algum.
Por isso o módulo mantém sequência monotônica por partição e um registro do que
**deveria** chegar — a completude é medida, não presumida.

A terceira tese resolve a tensão que define o módulo: **imutabilidade versus direito
ao esquecimento**. Uma trilha que não pode ser alterada não pode apagar dados
pessoais; a LGPD exige apagar. A saída é o **apagamento criptográfico**: o payload é
cifrado com uma chave por titular; ao exercer o direito, a **chave é destruída**. O
registro permanece na cadeia com hash íntegro — prova-se que **algo** aconteceu, sem
que o conteúdo pessoal seja mais legível. A cadeia sobrevive; o dado pessoal, não.

### 1.2 Responsabilidades (o 025-Audit DEVE)

| # | Responsabilidade |
|---|------------------|
| R-01 | Ingerir registros de auditoria por **evento** (JetStream) e por **escrita direta** (API), com idempotência por `event.id`/`Idempotency-Key`. |
| R-02 | Normalizar todo registro ao **envelope canônico de auditoria** (quem, o quê, sobre o quê, quando, sob qual decisão e correlação). |
| R-03 | **Encadear por hash**: todo registro carrega o hash do anterior na sua partição, formando uma cadeia verificável. |
| R-04 | **Selar periodicamente**: publicar raiz de Merkle assinada (chave do `021-Security`), tornando a adulteração retroativa detectável. |
| R-05 | Verificar **integridade** de forma contínua e sob demanda, e sinalizar qualquer quebra como anomalia de severidade máxima. |
| R-06 | Verificar **completude**: detectar lacuna de sequência e ausência de classe de registro esperada. |
| R-07 | Aplicar **retenção legal** por classe de registro, com base legal declarada. |
| R-08 | Gerir **legal hold**: suspender retenção e apagamento sobre um escopo, emitindo `audit.legalhold.applied` (consumido por `005` e demais donos de dado). |
| R-09 | Executar **apagamento criptográfico** (destruição da chave do titular) ao receber pedido de esquecimento, preservando a cadeia. |
| R-10 | Emitir **comprovante** verificável de expurgo/apagamento, correlacionado ao pedido original. |
| R-11 | Arquivar em **WORM** (objeto com *object lock*) as partições seladas, fora do alcance de operação de banco. |
| R-12 | Servir **consulta** de trilha com PEP (`022-Policy`), escopo por tenant e trilha da própria consulta. |
| R-13 | **Exportar pacote probatório** para auditor externo: registros, selos, cadeia e instruções de verificação independente. |
| R-14 | Detectar **anomalias** de auditoria (quebra de cadeia, lacuna, volume anômalo, acesso incomum) e emitir `audit.anomaly.detected`. |
| R-15 | Ser **PEP** de suas próprias operações administrativas (legal hold, retenção, exportação), com *default deny*. |

### 1.3 Não-Responsabilidades (o 025-Audit NÃO DEVE)

| # | Não-Responsabilidade | Dono real |
|---|----------------------|-----------|
| NR-01 | Ser fonte de **métrica operacional** ou de painel. A trilha é prova, não série temporal. | `024-Observability` |
| NR-02 | **Decidir** autorização. O 025 registra a decisão; quem decide é o PDP. | `022-Policy` |
| NR-03 | **Alertar** o plantão ou rotear notificação. O 025 emite o fato; o roteamento é do observador. | `024-Observability`, `029-Operations` |
| NR-04 | Estabelecer **identidade** ou custodiar chaves. O 025 **usa** chaves; o chaveiro é do `021`. | `021-Security` |
| NR-05 | Executar o **expurgo do dado de negócio** nos módulos donos. O 025 sinaliza e registra o comprovante. | `005-Database`, `010-Memory`, módulo dono |
| NR-06 | Definir a **classificação** de dado (`data_class`) do conteúdo auditado. | módulo dono do dado |
| NR-07 | Interpretar juridicamente a base legal ou decidir se um *hold* é devido. O 025 **aplica** o que foi decidido, com autor registrado. | jurídico/DPO do tenant |
| NR-08 | Substituir o **journal de decisões** do `022` (retenção curta, explicabilidade) por sua trilha. | `022-Policy` |
| NR-09 | Reprocessar ou **corrigir** um registro. Erro se corrige com **novo** registro de compensação. | — (invariante I6) |
| NR-10 | Guardar **conteúdo de negócio** por si (prompt, resposta, item de memória) além do necessário para provar a ação. | módulo dono do dado |
| NR-11 | Investigar incidente ou conduzir resposta. O 025 fornece a evidência. | `029-Operations` |
| NR-12 | Ser barramento: não reencaminha eventos alheios nem serve como fila de integração. | `020-Communication` |

---

## 2. Decomposição em Componentes Internos

### 2.1 Componentes (PascalCase · responsabilidade · dependências)

| Componente | Responsabilidade | Depende de (interno → externo) |
|------------|------------------|-------------------------------|
| **AuditIngestGateway** | Recebe registros por API direta (mTLS) e por consumidores JetStream; valida envelope, deduplica por `event.id` e **confirma só após durabilidade** (RPO 0). | RecordNormalizer, ChainAppender; ← 020-Communication |
| **RecordNormalizer** | Converte o evento de origem no **envelope canônico de auditoria** (§3.1); resolve `actor`, `action`, `resource`, `decision_ref` e correlação (RFC-0001 §5.6). | AuditableRegistry |
| **AuditableRegistry** | Registro das **classes de registro auditáveis** (`record_class`), sua retenção, base legal e o produtor esperado — insumo da verificação de completude. | → PostgreSQL |
| **ChainAppender** | Atribui `seq` monotônico por partição, calcula `record_hash` sobre o conteúdo canônico e encadeia com `prev_hash`. **Append-only**; nenhum caminho de atualização existe. | → PostgreSQL (`audit.record`) |
| **PayloadCipher** | Cifra o payload com a **chave do titular** (envelope) antes de persistir, permitindo apagamento criptográfico posterior sem tocar a cadeia. | SubjectKeyVault |
| **SubjectKeyVault** | Mapeia titular → chave de cifragem (custodiada no `021-Security`); destrói a chave sob pedido de esquecimento. | → 021-Security |
| **SealService** | Agrega os registros de um intervalo em uma **árvore de Merkle**, assina a raiz e publica o selo; a assinatura vem do `021`. | ChainAppender; → 021-Security |
| **IntegrityVerifier** | Recomputa cadeia e provas de Merkle, contínua (amostragem) e sob demanda (verificação completa); qualquer divergência é incidente máximo. | SealService, WormArchiver |
| **CompletenessMonitor** | Detecta lacuna de `seq`, atraso de selo e ausência de `record_class` esperada; a **lacuna é o defeito mais perigoso** (§1.1). | AuditableRegistry, ChainAppender |
| **LegalHoldManager** | Aplica e libera *holds* por escopo (titular, período, classe, caso); emite `audit.legalhold.applied`/`.released`; **sobrepõe** retenção e apagamento. | EventEmitter; → PostgreSQL |
| **RetentionEnforcer** | Expurga o **payload** de registros fora da retenção legal, preservando `seq`, hashes e metadados mínimos na cadeia. | LegalHoldManager, WormArchiver |
| **CryptoShredder** | Executa o apagamento criptográfico: destrói a chave do titular e marca os registros como `Shredded`, sem quebrar a cadeia. | SubjectKeyVault, LegalHoldManager |
| **ErasureReceiptService** | Emite comprovante verificável (hash, escopo, horário, executor) do apagamento próprio e registra os comprovantes recebidos de `005`/`010`/`024`. | ChainAppender, EventEmitter |
| **WormArchiver** | Arquiva partições seladas em objeto com *object lock* (WORM), fora do alcance de operação de banco; mantém o catálogo de arquivamento. | SealService; → MinIO |
| **QueryService** | Consulta da trilha com PEP, escopo por tenant e **registro da própria consulta** como evento auditável. | PolicyClient, ChainAppender |
| **ExportService** | Monta o **pacote probatório** para auditor externo: registros, selos, provas de Merkle e instruções de verificação independente. | IntegrityVerifier, SealService |
| **AnomalyDetector** | Quebra de cadeia, lacuna, volume anômalo por produtor, padrão incomum de consulta; emite `audit.anomaly.detected`. | CompletenessMonitor, IntegrityVerifier |
| **PolicyClient** | PEP das operações administrativas e de consulta; circuit breaker, cache curto, *fail-closed*. | → 022-Policy |
| **EventEmitter** | Publica eventos CloudEvents via **Outbox transacional** em `audit.outbox`. | → PostgreSQL, NATS/JetStream |
| **AuditTelemetry** | Métricas `aios_aud_*`, spans e logs OTel para `024-Observability` — **sem** expor conteúdo auditado. | → 024-Observability |

### 2.2 Diagrama de Componentes (ASCII)

```
   eventos JetStream (aios.*.>)          API direta /v1/audit (mTLS)
                 │                                    │
   ┌─────────────▼────────────────────────────────────▼─────────────────────────┐
   │                     AUDIT SERVICE (025 · .NET 10)                            │
   │                                                                              │
   │  ┌──────────────────────────┐        ┌──────────────────────────────────┐   │
   │  │   AuditIngestGateway     │        │  QueryService + ExportService    │   │
   │  │ (dedupe · ACK só após    │        │  + PolicyClient (PEP) ───────────┼───┼─▶ 022
   │  │  durabilidade — RPO 0)   │        └────────────────┬─────────────────┘   │
   │  └──────────┬───────────────┘                          │                     │
   │             ▼                                          │                     │
   │  ┌──────────────────────┐   ┌────────────────────┐    │                     │
   │  │  RecordNormalizer    │◀─▶│ AuditableRegistry  │    │                     │
   │  │ (envelope canônico)  │   │ (classes esperadas)│    │                     │
   │  └──────────┬───────────┘   └─────────┬──────────┘    │                     │
   │             ▼                          │               │                     │
   │  ┌──────────────────────┐   ┌──────────▼──────────┐   │                     │
   │  │    PayloadCipher     │──▶│   SubjectKeyVault   │───┼─────────────────────┼─▶ 021
   │  │ (chave por titular)  │   │ (chave por titular) │   │                     │
   │  └──────────┬───────────┘   └─────────────────────┘   │                     │
   │             ▼                                          │                     │
   │  ┌──────────────────────────────────────────────────┐ │                     │
   │  │   ChainAppender   seq · record_hash · prev_hash   │ │  APPEND-ONLY        │
   │  └──────────┬────────────────────────┬──────────────┘ │                     │
   │             ▼                         ▼                │                     │
   │  ┌──────────────────┐      ┌────────────────────────┐ │                     │
   │  │   SealService    │─────▶│    WormArchiver        │─┼─────────────────────┼─▶ MinIO
   │  │ (Merkle+assinat.)│      │ (object lock)          │ │                     │  (WORM)
   │  └────────┬─────────┘      └────────────────────────┘ │                     │
   │           ▼                                            │                     │
   │  ┌──────────────────┐  ┌──────────────────────┐  ┌────▼──────────────────┐  │
   │  │IntegrityVerifier │  │ CompletenessMonitor  │  │   AnomalyDetector     │  │
   │  └──────────────────┘  └──────────────────────┘  └───────────────────────┘  │
   │  ┌──────────────────┐  ┌──────────────────────┐  ┌───────────────────────┐  │
   │  │ LegalHoldManager │─▶│  RetentionEnforcer   │  │   CryptoShredder      │  │
   │  │ (sobrepõe tudo)  │  │ (expurga payload)    │  │ (destrói a chave)     │  │
   │  └──────────────────┘  └──────────────────────┘  └───────────┬───────────┘  │
   │                        ┌──────────────────────────────────────▼───────────┐  │
   │                        │        ErasureReceiptService                     │  │
   │                        └──────────────────────────────────────────────────┘  │
   │  ┌────────────────────────────────────────────────────────────────────────┐ │
   │  │ EventEmitter (Outbox → JetStream)  ·  AuditTelemetry (OTel, sem conteúdo)│ │
   │  └────────────────────────────────────────────────────────────────────────┘ │
   └──────────┬──────────────────────┬───────────────────────┬───────────────────┘
              ▼                      ▼                       ▼
      PostgreSQL (schema audit)   MinIO (WORM)      NATS aios.*.audit.*
```

---

## 3. Modelo de Dados / Entidades Canônicas

> Alinhado à RFC-0001 §5.1 (URN, ULID) e às convenções físicas do
> `../005-Database/Database.md` §2 (CV-01..CV-12). Schema: **`audit`**.
>
> **Regra absoluta deste modelo:** a tabela `audit.record` é **append-only**. Não
> existe caminho de `UPDATE` sobre conteúdo, e `DELETE` só remove **payload** de
> registros fora de retenção — `seq`, `record_hash`, `prev_hash` e metadados mínimos
> permanecem para sempre, sob pena de quebrar a cadeia. Correção de erro é **novo
> registro de compensação**, nunca alteração do original.

### 3.1 Entidade `AuditRecord` — tabela `audit.record` (multi-tenant, RLS, particionada)

O envelope canônico de auditoria. Entidade governada pela FSM da §4.

| Campo | Tipo | Chave/Constraint | Descrição |
|-------|------|------------------|-----------|
| `id` | `char(26)` | parte da PK (ULID) | Identidade do registro. |
| `tenant_id` | `text` | NOT NULL, RLS | Fronteira de isolamento. |
| `partition_key` | `text` | NOT NULL | Partição lógica da cadeia (`<tenant>:<shard>`); a sequência é monotônica **por partição**. |
| `seq` | `bigint` | NOT NULL, UNIQUE(partition_key,seq) | Sequência monotônica sem lacuna. |
| `occurred_at` | `timestamptz` | NOT NULL, chave de partição física | Quando o fato ocorreu (fornecido pelo produtor). |
| `recorded_at` | `timestamptz` | NOT NULL | Quando a trilha o gravou (relógio do 025). |
| `record_class` | `text` | NOT NULL, FK→audit.auditable_class.class_key | Classe do registro (dirige retenção e base legal). |
| `producer_module` | `text` | NOT NULL | Módulo que produziu o fato. |
| `actor` | `jsonb` | NOT NULL | Quem agiu: URN do princípio/agente/serviço + tipo. |
| `action` | `text` | NOT NULL | Ação em forma canônica `<dominio>:<objeto>:<verbo>`. |
| `resource_urn` | `text` | NOT NULL | Sobre o que a ação incidiu (RFC-0001 §5.1). |
| `outcome` | `text` | NOT NULL, CHECK ∈ {allowed, denied, succeeded, failed, compensated} | Resultado do fato. |
| `decision_ref` | `text` | NULL | `decision_id` do `../022-Policy/`, quando a ação passou por PDP. |
| `subject_ref` | `text` | NULL | Titular de dados pessoais envolvido (pseudonimizado) — chave do apagamento criptográfico. |
| `payload_cipher` | `bytea` | NULL | Detalhe do fato, **cifrado** com a chave do titular ou do tenant. |
| `payload_digest` | `text` | NOT NULL | `sha256` do payload em claro — permanece após o apagamento, provando **que houve** conteúdo. |
| `prev_hash` | `text` | NOT NULL | `record_hash` do registro anterior na partição (`GENESIS` no primeiro). |
| `record_hash` | `text` | NOT NULL, UNIQUE | `sha256(canonical(campos imutáveis) ‖ prev_hash)`. |
| `seal_ref` | `text` | NULL, FK→audit.seal.urn | Selo que cobre este registro. |
| `state` | `text` | NOT NULL, CHECK ∈ RecordState | Estado da FSM (§4.1). |
| `trace_id` | `text` | NOT NULL | Correlação W3C (RFC-0001 §5.6). |
| `event_id` | `text` | NULL, UNIQUE(tenant_id,event_id) | `event.id` de origem — dedupe (RFC-0001 §5.5). |

Chave primária: `PRIMARY KEY (tenant_id, occurred_at, id)`.
Índices: `UNIQUE(partition_key, seq)`; `UNIQUE(record_hash)`;
`UNIQUE(tenant_id, event_id)`; `btree(tenant_id, subject_ref)`;
`btree(tenant_id, record_class, occurred_at DESC)`;
`btree(decision_ref)`; `btree(seal_ref)`; `btree(state) WHERE state = 'Received'`.

> **Por que `payload_digest` sobrevive ao apagamento.** Depois de destruída a chave, o
> conteúdo é ilegível — mas o `payload_digest` prova que **existia** um conteúdo com
> aquele hash. Sem ele, o apagamento seria indistinguível de um registro que sempre
> foi vazio, e um adversário poderia alegar que nada havia ali.

### 3.2 Entidade `Seal` — tabela `audit.seal` (multi-tenant, RLS)

| Campo | Tipo | Chave/Constraint | Descrição |
|-------|------|------------------|-----------|
| `urn` | `text` | PK | `urn:aios:<tenant>:event:<ULID>`. |
| `tenant_id` | `text` | NOT NULL, RLS | Isolamento. |
| `partition_key` | `text` | NOT NULL | Partição selada. |
| `seq_from` | `bigint` | NOT NULL | Primeiro `seq` coberto. |
| `seq_to` | `bigint` | NOT NULL, CHECK (seq_to ≥ seq_from) | Último `seq` coberto. |
| `merkle_root` | `text` | NOT NULL | Raiz da árvore de Merkle dos `record_hash` do intervalo. |
| `prev_seal_hash` | `text` | NOT NULL | Hash do selo anterior — os selos também formam cadeia. |
| `seal_hash` | `text` | NOT NULL, UNIQUE | Hash canônico deste selo. |
| `signature` | `bytea` | NOT NULL | Assinatura do `seal_hash` (chave `signing` do `../021-Security/`). |
| `signed_by` | `text` | NOT NULL | URN da chave que assinou. |
| `sealed_at` | `timestamptz` | NOT NULL | Momento da selagem. |
| `external_anchor` | `text` | NULL | Âncora externa opcional (registro em serviço de terceiro), quando o contrato exigir. |
| `worm_object` | `text` | NULL | Objeto WORM que arquiva o intervalo. |

Índices: `PK(urn)`; `UNIQUE(seal_hash)`; `UNIQUE(partition_key, seq_from)`;
`btree(tenant_id, sealed_at DESC)`.

> Os selos formam uma **segunda cadeia**, sobre a cadeia de registros. Adulterar um
> registro exige recomputar sua cadeia, o selo que o cobre, todos os selos seguintes —
> e forjar assinaturas cuja chave privada está no `021-Security`, fora do alcance de
> quem opera o banco.

### 3.3 Entidade `AuditableClass` — tabela `audit.auditable_class` (plataforma)

| Campo | Tipo | Chave/Constraint | Descrição |
|-------|------|------------------|-----------|
| `urn` | `text` | PK | `urn:aios:_platform:event:<ULID>`. |
| `class_key` | `text` | NOT NULL, UNIQUE | Ex.: `policy.decision`, `security.credential`, `db.erasure`. |
| `producer_module` | `text` | NOT NULL | Módulo esperado como produtor. |
| `source_subjects` | `text[]` | NOT NULL default '{}' | Subjects NATS que alimentam a classe. |
| `retention_days` | `int` | NOT NULL | Retenção legal em dias. |
| `legal_basis` | `text` | NOT NULL | Base legal da retenção (LGPD art. 7º / GDPR art. 6º). |
| `contains_pii` | `boolean` | NOT NULL default false | Exige cifragem por titular (apagamento criptográfico). |
| `expected_rate_per_day` | `bigint` | NULL | Volume esperado — insumo da detecção de **ausência**. |
| `status` | `text` | NOT NULL, CHECK ∈ {active, deprecated} | Situação. |

Índices: `PK(urn)`; `UNIQUE(class_key)`; `btree(producer_module, status)`.

> `expected_rate_per_day` existe para detectar o defeito que não deixa rastro: um
> produtor que **para de emitir**. Sem uma expectativa declarada, ausência de registro
> é indistinguível de ausência de atividade.

### 3.4 Entidade `LegalHold` — tabela `audit.legal_hold` (multi-tenant, RLS)

| Campo | Tipo | Chave/Constraint | Descrição |
|-------|------|------------------|-----------|
| `urn` | `text` | PK | `urn:aios:<tenant>:event:<ULID>`. |
| `tenant_id` | `text` | NOT NULL, RLS | Isolamento. |
| `case_ref` | `text` | NOT NULL | Referência do caso (processo, investigação, requisição). |
| `scope` | `jsonb` | NOT NULL | Escopo: `record_class`, `subject_ref`, janela temporal, `resource_urn`. |
| `legal_basis` | `text` | NOT NULL | Fundamento declarado. |
| `requested_by` | `text` | NOT NULL | Solicitante. |
| `approved_by` | `text` | NOT NULL, CHECK (approved_by ≠ requested_by) | Aprovador distinto. |
| `applied_at` | `timestamptz` | NOT NULL | Início do *hold*. |
| `expires_at` | `timestamptz` | NULL | Fim previsto — **PODE ser nulo** (§4.5). |
| `released_at` | `timestamptz` | NULL | Liberação efetiva. |
| `released_by` | `text` | NULL | Quem liberou. |
| `version` | `bigint` | NOT NULL default 0 | OCC. |

Índices: `PK(urn)`; `btree(tenant_id, applied_at DESC)`;
`btree(tenant_id) WHERE released_at IS NULL`; `gin(scope jsonb_path_ops)`.

> **`expires_at` PODE ser nulo aqui** — e é a única entidade do AIOS em que isso é
> permitido, ao contrário do waiver do `../022-Policy/` e do silêncio do
> `../024-Observability/`. A razão é jurídica: um *hold* não expira porque um prazo
> técnico venceu; expira quando o caso encerra. Em compensação, todo *hold* sem prazo
> aparece em revisão periódica obrigatória (§9).

### 3.5 Entidade `SubjectKey` — tabela `audit.subject_key` (multi-tenant, RLS)

| Campo | Tipo | Chave/Constraint | Descrição |
|-------|------|------------------|-----------|
| `urn` | `text` | PK | `urn:aios:<tenant>:event:<ULID>`. |
| `tenant_id` | `text` | NOT NULL, RLS | Isolamento. |
| `subject_ref` | `text` | NOT NULL, UNIQUE(tenant_id,subject_ref) | Titular pseudonimizado. |
| `key_ref` | `text` | NOT NULL | Referência à chave custodiada no `../021-Security/`. **Nunca o material.** |
| `state` | `text` | NOT NULL, CHECK ∈ {active, destroyed} | Situação da chave. |
| `destroyed_at` | `timestamptz` | NULL | Momento da destruição (apagamento criptográfico). |
| `destroy_receipt` | `text` | NULL, FK→audit.erasure_receipt.urn | Comprovante correspondente. |
| `version` | `bigint` | NOT NULL default 0 | OCC. |

Índices: `PK(urn)`; `UNIQUE(tenant_id, subject_ref)`; `btree(state)`.

### 3.6 Entidade `ErasureReceipt` — tabela `audit.erasure_receipt` (multi-tenant, RLS)

| Campo | Tipo | Chave/Constraint | Descrição |
|-------|------|------------------|-----------|
| `urn` | `text` | PK | `urn:aios:<tenant>:event:<ULID>`. |
| `tenant_id` | `text` | NOT NULL, RLS | Isolamento. |
| `kind` | `text` | NOT NULL, CHECK ∈ {crypto_shred, payload_purge, external} | Natureza do apagamento. |
| `subject_ref` | `text` | NULL | Titular, quando aplicável. |
| `origin_module` | `text` | NOT NULL | `025` (próprio) ou o módulo que reportou (`005`, `010`, `024`). |
| `scope` | `jsonb` | NOT NULL | Escopo apagado. |
| `records_affected` | `bigint` | NOT NULL default 0 | Comprovante quantitativo. |
| `executed_at` | `timestamptz` | NOT NULL | Execução. |
| `executed_by` | `text` | NOT NULL | Executor. |
| `receipt_hash` | `text` | NOT NULL, UNIQUE | Hash do comprovante — o que o solicitante guarda. |
| `chain_record_id` | `char(26)` | NOT NULL | Registro de auditoria que **prova** o comprovante (auto-referência à trilha). |

Índices: `PK(urn)`; `UNIQUE(receipt_hash)`; `btree(tenant_id, subject_ref)`;
`btree(origin_module, executed_at DESC)`.

> O comprovante de apagamento é, ele mesmo, um registro na trilha (`chain_record_id`).
> Apagar sem deixar prova do apagamento seria indistinguível de nunca ter apagado.

### 3.7 Entidade `AuditExport` — tabela `audit.export` (multi-tenant, RLS)

| Campo | Tipo | Chave/Constraint | Descrição |
|-------|------|------------------|-----------|
| `urn` | `text` | PK | `urn:aios:<tenant>:event:<ULID>`. |
| `tenant_id` | `text` | NOT NULL, RLS | Isolamento. |
| `scope` | `jsonb` | NOT NULL | Escopo exportado (classe, janela, titular, caso). |
| `requested_by` | `text` | NOT NULL | Solicitante (auditor interno/externo). |
| `approved_by` | `text` | NOT NULL, CHECK (approved_by ≠ requested_by) | Aprovador distinto. |
| `record_count` | `bigint` | NOT NULL default 0 | Registros incluídos. |
| `seal_count` | `int` | NOT NULL default 0 | Selos incluídos. |
| `package_hash` | `text` | NULL, UNIQUE | Hash do pacote gerado. |
| `package_object` | `text` | NULL | Objeto (MinIO) com o pacote probatório. |
| `status` | `text` | NOT NULL, CHECK ∈ {requested, building, ready, failed, expired} | Situação. |
| `expires_at` | `timestamptz` | NOT NULL | Validade do link de download. |

Índices: `PK(urn)`; `UNIQUE(package_hash)`; `btree(tenant_id, status)`;
`btree(expires_at)`.

### 3.8 Entidade `AuditAnomaly` — tabela `audit.anomaly` (multi-tenant, RLS)

| Campo | Tipo | Chave/Constraint | Descrição |
|-------|------|------------------|-----------|
| `id` | `char(26)` | PK (ULID) | Identidade. |
| `tenant_id` | `text` | NOT NULL, RLS | Isolamento. |
| `kind` | `text` | NOT NULL, CHECK ∈ {chain_break, sequence_gap, seal_overdue, class_silent, volume_anomaly, query_pattern} | Natureza. |
| `severity` | `text` | NOT NULL, CHECK ∈ {critical, high, medium} | Severidade. |
| `detail` | `jsonb` | NOT NULL | Evidência (partição, `seq` esperado/observado, classe, produtor). |
| `detected_at` | `timestamptz` | NOT NULL | Detecção. |
| `resolved_at` | `timestamptz` | NULL | Encerramento após investigação. |
| `resolution` | `text` | NULL | Conclusão registrada. |

Índices: `PK(id)`; `btree(tenant_id, kind, detected_at DESC)`;
`btree(severity) WHERE resolved_at IS NULL`.

### 3.9 Relações (ASCII)

```
   audit.auditable_class(1) ──rege──< audit.record(*)
                                          │  │
                     seq/prev_hash ───────┘  │ seal_ref
                     (cadeia por partição)   ▼
                                        audit.seal(*) ──[prev_seal_hash]──▶ audit.seal
                                             │                              (cadeia de selos)
                                             └──▶ WORM (MinIO object lock)

   audit.record(*) ──[subject_ref]──▶ audit.subject_key(1) ──destruída──▶ Shredded
        │
        ├──[decision_ref]──▶ 022-Policy (journal de decisões, retenção curta)
        └──[chain_record_id]◀── audit.erasure_receipt(*)

   audit.legal_hold(*) ══sobrepõe══╳ retenção e apagamento de audit.record
   audit.export(*) ──inclui──▶ audit.record(*) + audit.seal(*) + provas de Merkle
   audit.anomaly(*) ◀──detecta── IntegrityVerifier · CompletenessMonitor
```

---

## 4. Máquina de Estados Canônica do `AuditRecord`

> A entidade governada por máquina de estados é o **registro de auditoria** — com uma
> particularidade que o distingue de todas as outras FSMs do AIOS: **nenhuma transição
> altera o conteúdo do registro**. O que muda é o *status probatório* (selado ou não)
> e a *legibilidade do payload* (íntegro, indecifrável ou expurgado). Os campos que
> sustentam a cadeia — `seq`, `prev_hash`, `record_hash` — são imutáveis em **todos**
> os estados, inclusive nos terminais.

### 4.1 Estados (`RecordState`)

| Estado | Descrição | Terminal? |
|--------|-----------|-----------|
| `Received` | Encadeado e durável, ainda **não coberto por selo**. | não |
| `Sealed` | Coberto por selo assinado; adulteração retroativa é detectável. | não |
| `Held` | Sob *legal hold*: retenção e apagamento **suspensos**. | não |
| `Shredded` | Chave do titular destruída; payload indecifrável. `payload_digest` e cadeia intactos. | **sim** |
| `Expired` | Retenção legal vencida; payload expurgado. Metadados mínimos e cadeia intactos. | **sim** |

### 4.2 Transições, gatilhos e guardas

| # | De → Para | Gatilho | Guarda (DEVE ser verdadeira) |
|---|-----------|---------|------------------------------|
| T-01 | ∅ → `Received` | Ingestão (evento ou API) | `event_id` inédito no tenant ∧ `record_class` registrada em `audit.auditable_class` ∧ `prev_hash` = `record_hash` do último `seq` da partição ∧ escrita **durável** confirmada. |
| T-02 | `Received` → `Sealed` | `SealService` fecha o intervalo | Registro incluído na árvore de Merkle ∧ raiz assinada pela chave corrente do `../021-Security/` ∧ selo encadeado ao anterior. |
| T-03 | `Sealed`/`Received` → `Held` | `ApplyLegalHold` | Escopo do *hold* casa com o registro ∧ `approved_by ≠ requested_by` ∧ `legal_basis` declarada. |
| T-04 | `Held` → `Sealed` | `ReleaseLegalHold` | Capability `aud:hold:release` ∧ `released_by` registrado ∧ nenhum outro *hold* vigente cobre o registro. |
| T-05 | `Sealed` → `Shredded` | Apagamento criptográfico (RTBF) | Registro **não** está em `Held` ∧ `subject_ref` presente ∧ chave do titular destruída ∧ comprovante emitido. |
| T-06 | `Sealed` → `Expired` | Fim da retenção legal | Registro **não** está em `Held` ∧ `recorded_at + retention_days` < agora ∧ intervalo já arquivado em WORM. |
| T-07 | `Received` → `Received` | Reingestão do mesmo `event_id` | Idempotência: nenhum novo registro é criado; a repetição é contabilizada, não gravada. |

> **Não existe transição de saída de `Shredded` nem de `Expired`, e não existe
> transição que remova o registro da cadeia.** A "exclusão" no AIOS é sempre exclusão
> de **conteúdo**, nunca de **prova de existência**.

### 4.3 Diagrama de estados (ASCII)

```
        ingestão (T-01)          selo assinado (T-02)
   ∅ ──────────────────▶ ┌──────────┐ ─────────────────▶ ┌──────────┐
                          │ Received │                     │  Sealed  │
                          └────┬─────┘                     └────┬─────┘
                               │  ▲ reingestão idempotente        │
                               │  └── (T-07, nada é gravado)      │
                               │                                  │
                    legal hold │ (T-03)              legal hold   │ (T-03)
                               ▼                                  ▼
                          ┌─────────────────────────────────────────┐
                          │                  Held                    │
                          │  retenção e apagamento SUSPENSOS        │
                          └────────────────────┬────────────────────┘
                                               │ liberação (T-04)
                                               ▼
                                          ┌──────────┐
                                          │  Sealed  │
                                          └────┬─────┘
                              RTBF (T-05)      │      fim de retenção (T-06)
                    ┌──────────────────────────┴──────────────────────────┐
                    ▼                                                      ▼
             ┌─────────────┐                                       ┌─────────────┐
             │  Shredded   │ (terminal)                            │   Expired   │ (terminal)
             │ payload     │                                       │ payload     │
             │ indecifrável│                                       │ expurgado   │
             └─────────────┘                                       └─────────────┘
                    │                                                      │
                    └──── seq · prev_hash · record_hash · payload_digest ───┘
                                       PERMANECEM (invariante I4)
```

### 4.4 Invariantes

(I1) **Nenhum registro é removido da cadeia.** `seq`, `prev_hash` e `record_hash`
permanecem em todos os estados, inclusive terminais. Remover um elo tornaria toda a
cadeia posterior inverificável.
(I2) `prev_hash` do registro de `seq = N` é **exatamente** o `record_hash` do registro
de `seq = N-1` na mesma partição. Divergência é `chain_break` — anomalia crítica, não
degradação.
(I3) A sequência por partição é **monotônica e sem lacuna**. Lacuna é
`sequence_gap` — o defeito mais perigoso, porque um registro que nunca chegou não
deixa rastro (§1.1).
(I4) `Shredded` e `Expired` preservam `seq`, hashes, `payload_digest`, `occurred_at`,
`record_class` e `outcome`. Some o **conteúdo**, nunca a **prova de que houve** um.
(I5) Registro em `Held` **NÃO DEVE** transitar para `Shredded` nem `Expired`. O *legal
hold* sobrepõe retenção e direito ao esquecimento — e essa precedência é decisão
jurídica documentada, não escolha técnica.
(I6) **Append-only**: não existe caminho de `UPDATE` sobre `actor`, `action`,
`resource_urn`, `outcome`, `payload_digest` ou campos de cadeia. Erro se corrige com
**novo registro** de `outcome = compensated` referenciando o original.
(I7) Ingestão é **idempotente** por `(tenant_id, event_id)`: reprocessar não duplica
(RFC-0001 §5.5).
(I8) Todo registro é selado em ≤ `aud.seal.interval_s`; registro em `Received` além
desse prazo é `seal_overdue`.
(I9) A escrita só é confirmada ao produtor **após durabilidade** — o RPO deste módulo
é **0** para registro confirmado (§7.2, NFR-006).

### 4.5 Ciclo de vida do `LegalHold`

| Estado | Significado | Próximos |
|--------|-------------|----------|
| `Requested` | Solicitado, aguardando aprovador distinto. | `Active`, `Denied` |
| `Active` | Vigente; suspende retenção e apagamento no escopo. | `Released` |
| `Denied` | Recusado pelo aprovador. | — (terminal) |
| `Released` | Liberado após encerramento do caso, com autor registrado. | — (terminal) |

```
   Requested ──aprova──▶ Active ──libera (caso encerrado)──▶ Released (terminal)
       │
       │ recusa
       ▼
   Denied (terminal)
```

Ao contrário do waiver do `../022-Policy/` e do silêncio do `../024-Observability/`,
o *hold* **PODE** não ter prazo: ele expira quando o caso encerra, não quando um
temporizador vence. O controle compensatório é a **revisão periódica obrigatória** de
*holds* ativos (§9) e a exigência de `case_ref` e `legal_basis`.

### 4.6 Verificação de integridade (não é FSM)

```
   verificação contínua (amostragem)          verificação completa (sob demanda)
   ────────────────────────────────           ─────────────────────────────────
   a cada aud.verify.interval_s:              para uma partição/janela:
     · sorteia N registros selados              1. recomputa record_hash de cada registro
     · recomputa record_hash                    2. confere prev_hash encadeado
     · confere prova de Merkle no selo          3. recomputa a raiz de Merkle
     · confere assinatura do selo               4. verifica assinatura do selo (chave 021)
                                                5. confere cadeia de selos (prev_seal_hash)
   qualquer divergência ⇒ chain_break          6. compara com a cópia WORM
   (severidade CRÍTICA, alerta imediato)
```

O passo 6 é o que fecha o modelo: mesmo com acesso total ao PostgreSQL, um adversário
não consegue reescrever a cópia arquivada em objeto com *object lock*, nem forjar a
assinatura cuja chave privada está no `../021-Security/`.

---

## 5. Superfície de API (REST + gRPC)

> Autenticação/erros/idempotência/versionamento seguem RFC-0001 §5.4–§5.7. Pacote
> gRPC: `aios.audit.v1`. Base REST: `/v1/audit`.
>
> A **maior parte** do tráfego de entrada não passa por esta API: chega por
> **eventos** no JetStream (§6.2). A escrita direta existe para fatos que não são
> naturalmente eventos de barramento (ex.: resultado de uma cerimônia manual) e para
> produtores fora do plano de controle.

### 5.1 Ingestão

| Operação | REST | gRPC | Idempotente | Capability |
|----------|------|------|-------------|------------|
| Registrar fato | `POST /v1/audit/records` | `AppendRecord` | sim (`Idempotency-Key`) | `aud:record:append` |
| Registrar em lote | `POST /v1/audit/records:batch` | `AppendBatch` | sim | `aud:record:append` |

Regras normativas da ingestão:

- O produtor **DEVE** autenticar-se por mTLS com certificado de workload (`021`).
- A resposta só é devolvida **após durabilidade** confirmada (invariante I9) — este
  módulo é deliberadamente **fail-closed** na escrita: melhor o produtor reter no
  outbox e reenviar do que a trilha aceitar e perder.
- `record_class` **DEVE** estar registrada; classe desconhecida → `AIOS-AUD-0002`.
- Repetição do mesmo `event_id` devolve o registro já gravado (T-07), sem duplicar.

### 5.2 Consulta e prova

| Operação | REST | gRPC | Idempotente | Capability |
|----------|------|------|-------------|------------|
| Consultar trilha | `POST /v1/audit/records:query` | `QueryRecords` | sim (leitura) | `aud:record:read` |
| Ler registro | `GET /v1/audit/records/{id}` | `GetRecord` | sim (leitura) | `aud:record:read` |
| Obter prova de inclusão | `GET /v1/audit/records/{id}:proof` | `GetInclusionProof` | sim (leitura) | `aud:record:read` |
| Verificar integridade | `POST /v1/audit/verify` | `VerifyChain` | sim | `aud:integrity:verify` |
| Listar/ler selos | `GET /v1/audit/seals[/{urn}]` | `ListSeals`/`GetSeal` | sim (leitura) | `aud:seal:read` |

> **Toda consulta à trilha é, ela mesma, um fato auditável** (`audit.access.queried`).
> Quem consultou o quê é exatamente o tipo de informação que um auditor precisa — e a
> ausência desse registro seria a lacuna mais conveniente de todas.

### 5.3 Governança

| Operação | REST | gRPC | Idempotente | Capability |
|----------|------|------|-------------|------------|
| Registrar classe auditável | `PUT /v1/audit/classes/{key}` | `PutAuditableClass` | sim | `aud:class:manage` |
| Aplicar legal hold | `POST /v1/audit/legal-holds` | `ApplyLegalHold` | sim (key) | `aud:hold:apply` |
| Liberar legal hold | `POST /v1/audit/legal-holds/{urn}:release` | `ReleaseLegalHold` | sim | `aud:hold:release` |
| Listar holds ativos | `GET /v1/audit/legal-holds?active=true` | `ListLegalHolds` | sim (leitura) | `aud:hold:read` |
| Executar apagamento criptográfico | `POST /v1/audit/erasures` | `ExecuteErasure` | sim (key) | `aud:erasure:execute` |
| Ler comprovante | `GET /v1/audit/erasures/{urn}` | `GetErasureReceipt` | sim (leitura) | `aud:erasure:read` |
| Solicitar exportação | `POST /v1/audit/exports` | `RequestExport` | sim (key) | `aud:export:request` |
| Baixar pacote probatório | `GET /v1/audit/exports/{urn}/package` | `DownloadExport` | sim (leitura) | `aud:export:download` |

### 5.4 Catálogo de códigos de erro

> Formato RFC-0001 §5.4. Domínio reservado: **`AUD`** (0001–0099), registrado no
> catálogo mantido por `../004-API/` conforme RFC-0001 §8 (ratificação em **ADR-0250**).

| Código | HTTP | `retriable` | Significado |
|--------|------|-------------|-------------|
| `AIOS-AUD-0001` | 404 | não | Registro, selo, *hold*, comprovante ou exportação não encontrado. |
| `AIOS-AUD-0002` | 422 | não | `record_class` não registrada em `audit.auditable_class`. |
| `AIOS-AUD-0003` | 403 | não | Tenant divergente do contexto autenticado. |
| `AIOS-AUD-0004` | 422 | não | Envelope de registro inválido (campo obrigatório ausente ou malformado). |
| `AIOS-AUD-0005` | 503 | **sim** | Escrita não pôde ser confirmada como durável — **o produtor DEVE reenviar** (I9). |
| `AIOS-AUD-0006` | 409 | sim | Conflito de concorrência otimista (`version`) em operação de governança. |
| `AIOS-AUD-0007` | 409 | não | Transição de estado inválida (viola a FSM §4). |
| `AIOS-AUD-0008` | **423** | não | Operação bloqueada por **legal hold** vigente (invariante I5). |
| `AIOS-AUD-0009` | 403 | não | Aprovação dupla exigida e ausente (*hold* ou exportação). |
| `AIOS-AUD-0010` | 422 | não | Retenção sem `legal_basis` para classe com `contains_pii`. |
| `AIOS-AUD-0011` | 500 | não | **Quebra de cadeia detectada** — integridade comprometida; incidente crítico. |
| `AIOS-AUD-0012` | 409 | não | Registro já apagado (`Shredded`/`Expired`): payload indisponível por desenho. |
| `AIOS-AUD-0013` | 429 | sim | Limite de consulta/exportação excedido. |
| `AIOS-AUD-0014` | 412 | não | Selo ausente ou assinatura inválida para o intervalo solicitado. |
| `AIOS-AUD-0015` | 422 | não | Escopo de exportação vazio ou excedendo o máximo permitido. |
| `AIOS-AUD-0016` | 409 | não | Tentativa de alteração de registro (append-only, invariante I6). |

> **Nota deliberada sobre `AIOS-AUD-0005`.** Este é o único módulo do AIOS em que
> devolver `503` na escrita é a **resposta correta**: o produtor tem outbox e vai
> reenviar. Aceitar o registro sem durabilidade confirmada trocaria uma
> indisponibilidade visível por uma **lacuna invisível** — e a lacuna é justamente o
> defeito que este módulo existe para tornar impossível.

---

## 6. Catálogo de Eventos NATS

> Envelope CloudEvents e convenção de subjects: RFC-0001 §5.2–§5.3. `<dominio>` =
> **`audit`**, valor **já previsto** na lista de domínios da RFC-0001 §5.3. Entrega
> at-least-once; dedupe por `event.id`.
>
> **Regra absoluta:** eventos deste módulo carregam **referências e provas**, nunca o
> conteúdo auditado. Um evento de `audit.record.sealed` transporta hashes e
> intervalos; quem precisa do conteúdo consulta a trilha sob PEP.

### 6.1 Eventos emitidos (produtor: 025-Audit)

| Subject | `type` | Quando | Stream |
|---------|--------|--------|--------|
| `aios.<tenant>.audit.record.sealed` | `aios.audit.record.sealed` | Intervalo selado (T-02); carrega `merkle_root` e assinatura. | `AUDIT_SEAL` |
| `aios.<tenant>.audit.anomaly.detected` | `aios.audit.anomaly.detected` | Quebra de cadeia, lacuna, selo atrasado, classe silenciosa, volume ou consulta anômala. | `AUDIT_ANOMALY` |
| `aios.<tenant>.audit.legalhold.applied` | `aios.audit.legalhold.applied` | *Hold* ativado — **consumido por `005-Database`** para suspender expurgo. | `AUDIT_GOVERNANCE` |
| `aios.<tenant>.audit.legalhold.released` | `aios.audit.legalhold.released` | *Hold* liberado; expurgo pode retomar. | `AUDIT_GOVERNANCE` |
| `aios.<tenant>.audit.erasure.completed` | `aios.audit.erasure.completed` | Apagamento criptográfico ou expurgo de payload concluído, com comprovante. | `AUDIT_GOVERNANCE` |
| `aios.<tenant>.audit.retention.expired` | `aios.audit.retention.expired` | Payload expurgado por fim de retenção (T-06). | `AUDIT_GOVERNANCE` |
| `aios.<tenant>.audit.export.completed` | `aios.audit.export.completed` | Pacote probatório pronto para o auditor. | `AUDIT_GOVERNANCE` |
| `aios._platform.audit.seal.published` | `aios.audit.seal.published` | Selo de plataforma publicado (âncora global). | `AUDIT_SEAL` |

### 6.2 Eventos consumidos

O módulo assina os subjects declarados em `audit.auditable_class.source_subjects`. A
tabela abaixo é o conjunto inicial; **acrescentar uma classe é o caminho normativo**
para tornar um novo fato auditável (§1.1, tese da completude declarada).

| Subject assinado | Produtor | `record_class` resultante |
|-------------------|----------|---------------------------|
| `aios.<tenant>.policy.decision.denied` | `022-Policy` | `policy.decision` |
| `aios.<tenant>.policy.bundle.updated` / `.rolledback` | `022-Policy` | `policy.bundle` |
| `aios.<tenant>.policy.exception.granted` / `.expired` | `022-Policy` | `policy.exception` |
| `aios.<tenant>.security.credential.issued` / `token.revoked` | `021-Security` | `security.credential` |
| `aios.<tenant>.security.principal.disabled` | `021-Security` | `security.principal` |
| `aios.<tenant>.security.rtbf.requested` | `021-Security` | `privacy.rtbf` (**e dispara o apagamento próprio**) |
| `aios.<tenant>.agent.lifecycle.spawned` / `.terminated` | `006-Kernel` | `agent.lifecycle` |
| `aios.<tenant>.task.execution.completed` / `.failed` | `009-Scheduler` | `task.execution` |
| `aios.<tenant>.telemetry.alert.acknowledged` | `024-Observability` | `ops.alert` |
| `aios.<tenant>.cost.budget.exhausted` | `026-Cost-Optimizer` | `cost.budget` |
| `aios._platform.deployment.rollout.started` | `028-Deployment` | `ops.deployment` |

Comprovantes de expurgo executados por `005-Database`, `010-Memory` e
`024-Observability` chegam pela **API direta** (§5.1) e são registrados como
`privacy.erasure_receipt`, com `origin_module` preservado.

---

## 7. Requisitos Funcionais e Não-Funcionais

### 7.1 Requisitos Funcionais (FR)

| ID | Requisito | Prioridade | Critério de aceite |
|----|-----------|-----------|--------------------|
| FR-001 | Ingerir registros por evento (JetStream) e por API direta, com dedupe por `event.id`. | Must | Reprocessar 10⁴ eventos não cria duplicata; `seq` sem lacuna. |
| FR-002 | Confirmar a escrita **somente após durabilidade**; do contrário, `AIOS-AUD-0005`. | Must | Teste de crash: nenhum registro confirmado é perdido (RPO 0). |
| FR-003 | Normalizar todo fato ao envelope canônico de auditoria. | Must | 100% dos registros com `actor`, `action`, `resource_urn`, `outcome`, `trace_id`. |
| FR-004 | Encadear por hash: `prev_hash` = `record_hash` do anterior na partição. | Must | Verificação de cadeia passa em 100% dos registros (invariante I2). |
| FR-005 | Selar intervalos periodicamente com raiz de Merkle **assinada**. | Must | Selo em ≤ `aud.seal.interval_s`; assinatura verificável com a chave pública do `021`. |
| FR-006 | Verificar integridade continuamente (amostragem) e sob demanda (completa). | Must | Adulteração injetada é detectada em ≤ 1 ciclo de verificação. |
| FR-007 | Detectar lacuna de sequência e classe silenciosa. | Must | `seq` faltante → `audit.anomaly.detected{kind=sequence_gap}` em ≤ 5 min. |
| FR-008 | Aplicar retenção legal por classe, com base legal declarada. | Must | Classe `contains_pii` sem `legal_basis` → `AIOS-AUD-0010`. |
| FR-009 | Aplicar e liberar *legal hold* com aprovação dupla e `case_ref`. | Must | `approved_by = requested_by` → `AIOS-AUD-0009`. |
| FR-010 | *Legal hold* **suspende** retenção e apagamento no escopo. | Must | Expurgo ou apagamento sob *hold* → `AIOS-AUD-0008`; nenhum payload removido. |
| FR-011 | Executar apagamento criptográfico destruindo a chave do titular. | Must | Payload torna-se indecifrável; `record_hash` e `payload_digest` inalterados. |
| FR-012 | Preservar cadeia e metadados em `Shredded` e `Expired`. | Must | Verificação de cadeia continua passando após apagamento (invariante I4). |
| FR-013 | Emitir comprovante verificável de todo apagamento, ele mesmo registrado na trilha. | Must | `receipt_hash` reproduzível; `chain_record_id` aponta para registro existente. |
| FR-014 | Arquivar intervalos selados em WORM (objeto com *object lock*). | Must | Tentativa de sobrescrita do objeto é rejeitada pelo armazenamento. |
| FR-015 | Servir consulta com PEP e escopo por tenant. | Must | Consulta cross-tenant → `AIOS-AUD-0003`. |
| FR-016 | Registrar **toda consulta à trilha** como fato auditável. | Must | Consulta gera registro `audit.access` com solicitante e escopo. |
| FR-017 | Exportar pacote probatório com registros, selos, provas e instruções de verificação. | Must | Auditor externo verifica o pacote **sem** acesso ao AIOS. |
| FR-018 | Detectar e sinalizar anomalias de auditoria. | Must | Quebra de cadeia → `audit.anomaly.detected{kind=chain_break, severity=critical}`. |
| FR-019 | Recusar qualquer alteração de registro (append-only). | Must | Tentativa de `UPDATE` → `AIOS-AUD-0016`; correção só por registro de compensação. |
| FR-020 | Atuar como PEP nas operações administrativas e de consulta. | Must | Sem capability → negado pelo PDP; a própria negação é auditada. |
| FR-021 | Emitir todos os eventos de §6.1 via Outbox transacional. | Must | Nenhum evento perdido em teste de crash entre commit e publicação. |
| FR-022 | Registrar classes auditáveis com produtor esperado e volume esperado. | Should | Classe sem registro na origem → `AIOS-AUD-0002` na ingestão. |
| FR-023 | Não expor conteúdo auditado em evento, métrica ou log. | Must | Varredura automatizada não encontra payload fora da trilha. |

### 7.2 Requisitos Não-Funcionais (NFR) com SLO/SLI

| ID | Atributo | Meta (SLO) | SLI / método de verificação |
|----|----------|-----------|-----------------------------|
| NFR-001 | Desempenho (escrita) | p99 de `AppendRecord` (durável) ≤ **20 ms**. | `aios_aud_append_latency_ms`. |
| NFR-002 | Throughput | ≥ **50.000 registros/s** por réplica. | Benchmark `Benchmark.md`. |
| NFR-003 | Disponibilidade | ≥ **99,95%**. | Uptime probe; *error budget* mensal. |
| NFR-004 | Latência de selagem | **100%** dos registros selados em ≤ **60 s**. | `aios_aud_seal_lag_s`. |
| NFR-005 | Completude | **100%** dos eventos de classes registradas viram registro; lacuna detectada em ≤ **5 min**. | Contagem produtor × trilha; `aios_aud_sequence_gap_total`. |
| NFR-006 | Durabilidade | **RPO = 0** para registro confirmado; **RTO ≤ 15 min**. | Teste de crash; DR drill. |
| NFR-007 | Integridade | **100%** da cadeia verificável; adulteração detectada em ≤ **1 ciclo** de verificação (default 15 min). | Injeção controlada; `aios_aud_chain_break_total = 0`. |
| NFR-008 | Verificação completa | Cadeia de **10⁹** registros verificável em ≤ **4 h**. | Benchmark de verificação. |
| NFR-009 | Escalabilidade | ≥ **10⁹** registros por tenant e ≥ **10⁶** agentes produtores. | Teste de escala `Scalability.md`. |
| NFR-010 | Retenção | **100%** de aderência à retenção legal por classe; **0** expurgo sob *legal hold*. | Auditoria de expurgo. |
| NFR-011 | Apagamento criptográfico | Efetivo em ≤ **24 h** do pedido; **0** payload legível após a destruição da chave. | Teste de decifragem pós-apagamento. |
| NFR-012 | Latência de consulta | p95 de consulta por `subject_ref`/`decision_ref` ≤ **2 s** em 10⁹ registros. | `aios_aud_query_latency_ms`. |
| NFR-013 | Exportação | Pacote de **10⁶** registros gerado em ≤ **30 min** e verificável offline. | Benchmark de exportação. |
| NFR-014 | Idempotência | Repetições produzem efeito único em **100%** dos casos. | Teste de replay. |
| NFR-015 | Isolamento de tenant | **0** consultas ou exportações que atravessem a fronteira de tenant. | RLS + validação estrutural; `AIOS-AUD-0003`. |
| NFR-016 | Confidencialidade | **0** ocorrências de conteúdo auditado em evento, métrica ou log. | Varredura contínua (FR-023). |
| NFR-017 | Observabilidade | **100%** das operações com trace OTel correlacionado. | Cobertura de spans (`../024-Observability/`). |

---

## 8. Chaves de Configuração Principais

> Escopo: `global`, `tenant`, `record_class`. Recarregável indica *hot reload*.
> Prefixo de env var: **`AIOS_AUD_`**.

| Chave | Default | Faixa | Escopo | Recarregável | Descrição |
|-------|---------|-------|--------|--------------|-----------|
| `aud.ingest.require_durable_ack` | `true` | {true,false} | global | **não** | Confirma só após durabilidade (invariante I9, `AIOS-AUD-0005`). |
| `aud.ingest.batch_max_size` | 500 | 1–5000 | global | sim | Registros por lote na API. |
| `aud.ingest.max_rate_per_s_per_tenant` | 100000 | 100–1000000 | tenant | sim | Limite de ingestão por tenant. |
| `aud.chain.partitions_per_tenant` | 16 | 1–256 | tenant | **não** | Partições da cadeia; alterar exige migração (a sequência é por partição). |
| `aud.chain.hash_algorithm` | `sha256` | {sha256,sha3-256} | global | **não** | Algoritmo de hash da cadeia. |
| `aud.seal.interval_s` | 60 | 10–3600 | global | sim | Intervalo de selagem (NFR-004). |
| `aud.seal.signing_key_ref` | — | — | global | **não** | Chave de assinatura de selo (custodiada no `../021-Security/`). |
| `aud.seal.external_anchor_enabled` | `false` | {true,false} | tenant | sim | Ancoragem externa do selo, quando o contrato exigir. |
| `aud.verify.interval_s` | 900 | 60–86400 | global | sim | Ciclo de verificação contínua por amostragem (NFR-007). |
| `aud.verify.sample_size` | 1000 | 10–100000 | global | sim | Registros por ciclo de verificação. |
| `aud.completeness.gap_alert_s` | 300 | 30–86400 | global | sim | Prazo para sinalizar lacuna (NFR-005). |
| `aud.completeness.silence_ratio` | 0.2 | 0.0–1.0 | record_class | sim | Fração do volume esperado abaixo da qual a classe é considerada silenciosa. |
| `aud.retention.default_days` | 1825 | 30–7300 | record_class | sim | Retenção legal default (5 anos). |
| `aud.retention.min_days` | 365 | 30–7300 | global | **não** | Piso absoluto: nenhuma classe pode reter menos. |
| `aud.hold.review_interval_days` | 90 | 7–365 | tenant | sim | Periodicidade da revisão obrigatória de *holds* ativos. |
| `aud.hold.require_dual_approval` | `true` | {true,false} | global | **não** | Aprovador distinto do solicitante (`AIOS-AUD-0009`). |
| `aud.erasure.max_latency_h` | 24 | 1–720 | tenant | sim | Prazo para efetivar o apagamento criptográfico (NFR-011). |
| `aud.worm.object_lock_days` | 1825 | 30–7300 | global | **não** | Duração do *object lock* no arquivamento WORM. |
| `aud.worm.archive_after_s` | 3600 | 60–86400 | global | sim | Prazo entre selagem e arquivamento. |
| `aud.query.max_duration_s` | 30 | 1–300 | tenant | sim | Orçamento de tempo por consulta (`AIOS-AUD-0013`). |
| `aud.query.max_records` | 100000 | 100–10000000 | tenant | sim | Teto de registros por consulta. |
| `aud.export.max_records` | 1000000 | 1000–100000000 | tenant | sim | Teto por exportação (`AIOS-AUD-0015`). |
| `aud.export.package_ttl_h` | 72 | 1–720 | tenant | sim | Validade do link do pacote probatório. |
| `aud.export.require_dual_approval` | `true` | {true,false} | tenant | **não** | Aprovador distinto para exportação. |
| `aud.policy.fail_mode` | `closed` | {closed,open} | global | sim | PEP de consulta e administração — *fail-closed*. |

> **`aud.retention.min_days` e `aud.worm.object_lock_days` são não-recarregáveis por
> desenho.** São os dois valores que um operador sob pressão teria incentivo para
> reduzir a fim de "liberar espaço" — e reduzi-los em runtime destruiria prova que a
> lei exige preservar.

---

## 9. Modos de Falha e Recuperação

| Modo de falha | Detecção | Isolamento | Recuperação | Idempotência/Retry |
|---------------|----------|-----------|-------------|--------------------|
| PostgreSQL (`audit`) indisponível | Health | Escrita **recusada** com `AIOS-AUD-0005` | Failover do `005`; produtores reenviam do outbox | Dedupe por `event_id` |
| Produtor indisponível ou parado | `class_silent` (volume abaixo do esperado) | Classe específica | Investigar o produtor; **não** relaxar a expectativa | — |
| Lacuna de sequência (`sequence_gap`) | `CompletenessMonitor` | Partição afetada | Investigação **P1**: identificar se houve perda ou remoção; registrar conclusão em `audit.anomaly.resolution` | — |
| **Quebra de cadeia (`chain_break`)** | `IntegrityVerifier` | Partição afetada | **Incidente de segurança máximo**: congelar escrita da partição, comparar com WORM e com o selo assinado, acionar CISO | — |
| Selo atrasado (`seal_overdue`) | Registros em `Received` além do intervalo | Selagem | Verificar `SealService` e disponibilidade da chave no `021` | Selagem idempotente |
| Chave de assinatura indisponível | Falha ao selar | Selagem para; **ingestão continua** | Rotação/recuperação no `021`; selagem retroativa do intervalo pendente | Idempotente |
| MinIO (WORM) indisponível | Health | Arquivamento atrasa; cadeia e selos seguem | Retomar arquivamento; alerta se exceder `archive_after_s` | Idempotente por intervalo |
| NATS indisponível | Lag de consumidor | Ingestão por evento para; API direta segue | Drenagem do backlog; JetStream retém | At-least-once + dedupe |
| PDP (022) indisponível | Timeout/CB no `PolicyClient` | Consulta e administração negadas (`fail_mode=closed`); **ingestão continua** | Retry após meia-abertura | Retriable |
| Tentativa de expurgo sob *hold* | `LegalHoldManager` | Operação recusada (`AIOS-AUD-0008`) | Nenhuma — o comportamento é o correto; o pedido é registrado | — |
| *Hold* esquecido em aberto | Revisão periódica (`aud.hold.review_interval_days`) | — | Revisão com jurídico; liberação registrada | — |
| Apagamento criptográfico falho | Verificação pós-apagamento (decifragem ainda possível) | Titular específico | Reexecutar destruição de chave; **P1 de privacidade** | Idempotente |
| Exportação vazando escopo | Revisão de escopo na aprovação dupla | Exportação recusada | Reemitir com escopo correto | — |
| Volume anômalo de consulta | `AnomalyDetector` (`query_pattern`) | Sinal, não bloqueio | Investigação; a política de bloqueio é do `022` | — |

**Metas de recuperação:** **RTO ≤ 15 min** e **RPO = 0** para registro confirmado. O
RPO zero é possível porque a confirmação só ocorre após durabilidade (I9) e porque os
produtores mantêm outbox: uma indisponibilidade do 025 **atrasa** a trilha, não a
perde.

Degradação graciosa em ordem: sacrifica-se primeiro **exportação**, depois
**consulta**, depois **verificação contínua**, depois **selagem**, preservando por
último a **ingestão durável** — porque um sistema que não sela pode selar depois; um
sistema que não ingere perde o fato para sempre.

> **A falha que este módulo não pode ter.** Todas as outras falhas produzem atraso ou
> indisponibilidade. Apenas duas produzem dano irreversível: **perder um registro**
> (por aceitar sem durabilidade) e **não detectar uma adulteração**. As invariantes
> I9 e I2 existem exatamente para essas duas.

---

## 10. Escalabilidade e Concorrência

| Dimensão | Estratégia |
|----------|-----------|
| Cadeia particionada | A sequência é monotônica **por partição** (`aud.chain.partitions_per_tenant`, default 16), não globalmente. Uma cadeia única por tenant seria um ponto de serialização absoluto. |
| Escrita | Append em partição escolhida por `hash(actor, resource)`, garantindo distribuição estável e afinidade de verificação. |
| Concorrência de `seq` | Sequência por partição obtida com *advisory lock* curto ou sequência dedicada; contenção limitada a 1/N do tráfego. |
| Selagem | Paralela por partição; a árvore de Merkle é construída por intervalo, sem bloquear a escrita. |
| Verificação | Amostragem contínua (barata) + verificação completa sob demanda, executada em réplica de leitura para não competir com a ingestão. |
| Particionamento físico | `audit.record` particionada por `RANGE(occurred_at)` mensal, além da partição lógica de cadeia. |
| Arquivamento | Partições seladas vão para objeto WORM; consultas antigas leem do arquivo, não do banco quente. |
| Consulta | Índices por `subject_ref`, `decision_ref` e `(record_class, occurred_at)`; orçamento de tempo e de registros por consulta. |
| Isolamento multi-tenant | RLS + partição lógica por tenant; um tenant volumoso não desloca a cadeia de outro. |
| Rumo a milhões | 10⁶ agentes ⇒ ~10⁹ registros/ano por tenant grande. Sustentado por: cadeia particionada, selagem paralela, arquivamento WORM do histórico frio e consulta indexada pelos dois eixos que importam (titular e decisão). |

```
   ❌ cadeia única global                  ✅ cadeia por partição
      todo append serializa                   16 cadeias independentes por tenant
      (a trilha vira o gargalo do AIOS)       selagem e verificação paralelas
```

---

## 11. ADRs e RFCs a Propor

> **Faixa de ADR reservada: `ADR-0250`..`ADR-0259`** (regra 025×10). Registrar em
> `../002-ADR/`. **ADR-0010** (observabilidade e auditoria por construção) é herdada e
> é a decisão-mãe deste módulo; **ADR-0008** define que toda ação privilegiada gera
> decisão auditável.

| ADR | Título proposto |
|-----|-----------------|
| ADR-0250 | Domínio de erro `AUD` e registro das classes auditáveis (`record_class`). |
| ADR-0251 | Imutabilidade verificável: cadeia de hash + selo de Merkle assinado + WORM. |
| ADR-0252 | Cadeia particionada por tenant em vez de sequência global única. |
| ADR-0253 | Apagamento criptográfico (destruição de chave por titular) para conciliar imutabilidade e RTBF. |
| ADR-0254 | *Legal hold* sobrepõe retenção e direito ao esquecimento; *hold* pode não ter prazo. |
| ADR-0255 | Confirmação de escrita apenas após durabilidade (RPO 0) e `503` como resposta correta. |
| ADR-0256 | Completude declarada: classes auditáveis com produtor e volume esperados. |
| ADR-0257 | Consulta à trilha é fato auditável. |
| ADR-0258 | Correção por registro de compensação, nunca por alteração (append-only). |
| ADR-0259 | Pacote probatório verificável offline por auditor externo. |

| RFC | Título proposto | Relação |
|-----|-----------------|---------|
| RFC-0001 | Architecture Baseline & Core Contracts | **Baseline** (consumida, não redefinida) — §5.8 exige registro imutável para toda ação privilegiada; §7 fixa privacidade. |
| RFC-0250 | AIOS Audit Record & Chain Integrity Contract (envelope canônico, encadeamento, Merkle, selo, prova de inclusão, verificação offline). | A propor por este módulo. |
| RFC-0251 | AIOS Retention, Legal Hold & Crypto-Erasure (classes, base legal, *hold*, apagamento criptográfico, comprovante). | A propor por este módulo. |

---

## 12. Decisões de Segurança

> A trilha de auditoria é o alvo de maior valor do AIOS para um atacante interno: quem
> consegue alterá-la apaga a evidência de tudo o que fez. O modelo de segurança deste
> módulo parte da premissa de que **o operador do banco não é confiável** — e é por
> isso que a prova não vive apenas no banco.

### 12.1 AuthN / AuthZ

- **AuthN de produtores**: mTLS com certificado de workload emitido pelo
  `../021-Security/` após atestação (RFC-0001 §6).
- **AuthN de humanos**: OIDC validado no Gateway (`../004-API/`) com MFA.
- **AuthZ**: o 025 é **PEP**. Consulta, exportação, *hold*, retenção e apagamento
  exigem capability avaliada pelo **PDP** do `../022-Policy/`, com ***default deny***.
- **Separação de funções**: aplicar *hold* e liberar *hold* **NÃO DEVERIAM** ser a
  mesma pessoa; exportação exige aprovador distinto do solicitante; **ninguém** tem
  capability de alterar registro — ela não existe no modelo.
- **Fail-closed** em consulta e administração; a **ingestão** também é fail-closed
  (recusa em vez de aceitar sem durabilidade) — este módulo é fail-closed nas duas
  pontas, ao contrário do `../024-Observability/`.
- **Isolamento de tenant**: RLS em todas as tabelas; partição lógica de cadeia por
  tenant; `AIOS-AUD-0003` para tenant divergente.

### 12.2 Threat Model STRIDE (resumido)

| Ameaça (STRIDE) | Vetor no 025-Audit | Mitigação |
|-----------------|---------------------|-----------|
| **S**poofing | Serviço falso injeta registros forjados para criar um álibi ou incriminar outro princípio. | mTLS com certificado atestado; `producer_module` conferido contra o `AuditableRegistry`; `actor` derivado das claims propagadas, não de campo livre. |
| **T**ampering | **A ameaça central.** Operador com acesso ao banco altera ou remove registros para apagar evidência. | Cadeia de hash (I2) + selo de Merkle **assinado** com chave fora do banco (`021`) + cópia **WORM** com *object lock* + verificação contínua. Alterar exige recomputar cadeia, todos os selos posteriores e forjar assinaturas — em três sistemas distintos. |
| **R**epudiation | Negar ter aplicado um *hold*, exportado a trilha ou executado um apagamento. | `requested_by`/`approved_by`/`executed_by` obrigatórios; **toda consulta é auditada** (FR-016); comprovantes com hash reproduzível. |
| **I**nformation disclosure | Uso da consulta de auditoria como porta lateral para dados que a API de negócio não exporia; exportação com escopo excessivo. | Payload **cifrado por titular**; consulta sob PEP com capability por classe; exportação com aprovação dupla e escopo revisado; conteúdo nunca em evento/métrica/log (FR-023). |
| **D**enial of service | Inundação de registros por um produtor; consulta ou exportação que varre 10⁹ registros. | Limite por tenant; orçamento de consulta (`AIOS-AUD-0013`) e de exportação (`AIOS-AUD-0015`); a **recusa preserva a trilha** — melhor `503` que perda. |
| **E**levation of privilege | Obter capability de apagamento e usá-la para destruir evidência sob o disfarce de RTBF; reduzir retenção para expurgar prova. | *Legal hold* **sobrepõe** apagamento (I5, `AIOS-AUD-0008`); `aud.retention.min_days` não-recarregável; todo apagamento gera comprovante que é **ele mesmo** um registro na trilha; `Shredded` preserva `payload_digest`, provando que houve conteúdo. |

### 12.3 LGPD / GDPR

- **A tensão central e sua resolução**: imutabilidade e direito ao esquecimento são
  incompatíveis se "esquecer" significar "remover o registro". O AIOS resolve com
  **apagamento criptográfico** (ADR-0253): o payload é cifrado por titular e a chave é
  destruída. Prova-se que **algo** aconteceu (hash, classe, horário, resultado), sem
  que o **conteúdo pessoal** permaneça legível.
- **Minimização**: o registro guarda o mínimo necessário para provar a ação;
  conteúdo de negócio pertence ao módulo dono (NR-10).
- **Base legal e retenção**: obrigatórias por `record_class`; classe com
  `contains_pii` sem `legal_basis` é recusada (`AIOS-AUD-0010`).
- **Legal hold vs. RTBF**: o *hold* **prevalece** — obrigação legal de preservação
  sobrepõe o pedido de apagamento. O titular é informado da existência do *hold*, e a
  decisão é registrada com `case_ref` e `legal_basis`.
- **Comprovante**: todo apagamento gera comprovante verificável, correlacionado ao
  pedido original e registrado na própria trilha (FR-013).
- **Segregação**: RLS por tenant; partição lógica de cadeia por tenant; exportação
  jamais cruza a fronteira.
- **Transferência internacional**: o arquivamento WORM segue a região do tenant
  (`../027-Cluster/`); o pacote probatório declara a jurisdição de origem.

---

## 13. Referências

- Visão: `../000-Vision/Vision.md`
- Arquitetura: `../001-Architecture/Architecture.md`
- Glossário: `../040-Glossary/Glossary.md` (*Event Sourcing* — reutilizado, não redefinido)
- Contratos centrais: `../003-RFC/RFC-0001-Architecture-Baseline.md` (§5.5 idempotência,
  §5.6 correlação, §5.8 auditoria obrigatória, §7 privacidade)
- Decisão-mãe: `../002-ADR/ADR-0010-Observabilidade-Auditoria-por-Construcao.md`
- Governança: `../002-ADR/ADR-0008-Governanca-por-Politica-Default-Deny.md`
- Template de módulo: `../_templates/MODULE_TEMPLATE.md`
- Índice do módulo: `./README.md`
- Módulos acoplados: `../004-API/`, `../005-Database/`, `../006-Kernel/`,
  `../009-Scheduler/`, `../010-Memory/`, `../020-Communication/`, `../021-Security/`,
  `../022-Policy/`, `../024-Observability/`, `../026-Cost-Optimizer/`,
  `../027-Cluster/`, `../028-Deployment/`, `../029-Operations/`.

*Fim do Design Brief interno do módulo 025-Audit. Este documento governa a geração dos
26 documentos obrigatórios; qualquer conflito entre um documento gerado e este brief é
um defeito do documento gerado.*
