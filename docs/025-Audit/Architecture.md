---
Documento: Architecture
Módulo: 025-Audit
Status: Draft
Versão: 0.1
Última atualização: 2026-07-23
Responsável (RACI-A): Arquiteto do Módulo 025-Audit
ADRs relacionados: ADR-0002, ADR-0003, ADR-0004, ADR-0005, ADR-0010, ADR-0251, ADR-0252, ADR-0255, ADR-0259
RFCs relacionados: RFC-0001, RFC-0250, RFC-0251
Depende de: 001-Architecture, 005-Database, 020-Communication, 021-Security, 022-Policy, 024-Observability, 027-Cluster
---

# 025-Audit — Architecture

## 1. Visão C4 — Nível 1 (Contexto)

```
   ┌───────────────────────────────────────────────────────────────────────────┐
   │  TODOS OS MÓDULOS — emitem registro de auditoria (RFC-0001 §5.8)         │
   │  022 decisões · 021 credenciais · 006 syscalls · 009 escalonamento ·     │
   │  005 expurgos · 024 silêncios/SLOs · 026 gastos · 028 rollouts           │
   └──────────────┬───────────────────────────┬────────────────────────────────┘
                  │ eventos (JetStream)       │ API direta (mTLS, Idempotency-Key)
                  ▼                           ▼
   ┌───────────────────────────────────────────────────────────────────────────┐
   │                 025-AUDIT — TRILHA IMUTÁVEL (este módulo)                 │
   │  encadeia · sela · verifica · retém · segura (hold) · apaga (cripto)      │
   │  consulta sob PEP · exporta pacote probatório                            │
   └───┬──────────────┬───────────────┬─────────────────┬─────────────────────┘
       │              │               │                 │
       ▼              ▼               ▼                 ▼
   PostgreSQL      MinIO (WORM)   021-Security      022-Policy
   (schema audit)  object lock    (assinatura,       (PEP de consulta
                                   chaves/titular)    e governança)
       │
       └──▶ NATS aios.*.audit.*  ──▶ 005 (legal hold) · 024 (anomalia) · 029 (investigação)
                                  ──▶ AUDITOR EXTERNO (pacote verificável offline)
```

## 2. Visão C4 — Nível 2 (Contêiner)

| Contêiner | Tecnologia | Papel | Justificativa |
|-----------|-----------|-------|---------------|
| `audit-ingest` | .NET 10 (gRPC + consumidores JetStream) | Ingestão durável, normalização, encadeamento. | Control plane em .NET (ADR-0003); caminho crítico com confirmação síncrona. |
| `audit-sealer` | .NET 10 (worker) | Constrói árvores de Merkle, assina e publica selos. | Trabalho periódico e paralelo por partição, isolado da ingestão. |
| `audit-verifier` | .NET 10 (worker) | Verificação contínua por amostragem e completa sob demanda. | Executa em réplica de leitura para não competir com a ingestão. |
| `audit-api` | .NET 10 (ASP.NET Core + gRPC) | Consulta, governança (*hold*, retenção, apagamento) e exportação. | PEP; superfície de menor volume e maior sensibilidade. |
| `audit-archiver` | .NET 10 (worker) | Arquivamento WORM das partições seladas. | I/O de objeto isolado do caminho quente. |
| `audit-outbox-relay` | .NET 10 (worker) | Publica o outbox em JetStream. | At-least-once sem 2PC (padrão do repositório). |
| PostgreSQL (schema `audit`) | PostgreSQL 16 | Cadeia, selos, *holds*, comprovantes, anomalias. | Unificação relacional (ADR-0005); RLS por `tenant_id`. |
| MinIO | MinIO (S3) com *object lock* | Arquivo **WORM** das partições seladas. | Cópia fora do alcance de `UPDATE`/`DELETE` de banco. |
| NATS / JetStream | NATS 2.x | Ingestão por evento e emissão de `aios.*.audit.*`. | Barramento primário (ADR-0004). |

```
   eventos (JetStream)          API direta (mTLS)
          │                            │
   ┌──────▼────────────────────────────▼──────┐
   │        audit-ingest  (N réplicas)         │  ACK só após durabilidade
   │  normaliza · cifra payload · encadeia     │
   └──────────────────┬────────────────────────┘
                      ▼
             ┌─────────────────┐        ┌──────────────────┐
             │  PostgreSQL     │◀──────▶│  audit-sealer    │──▶ 021 (assina)
             │  schema audit   │        │  (Merkle + selo) │
             └────────┬────────┘        └────────┬─────────┘
                      │                          ▼
             ┌────────▼────────┐        ┌──────────────────┐
             │ audit-verifier  │        │ audit-archiver   │──▶ MinIO (WORM)
             │ (réplica leitura)│       └──────────────────┘
             └─────────────────┘
                      ▲
             ┌────────┴────────┐
             │   audit-api     │──▶ 022 (PEP) · exportação · legal hold
             └─────────────────┘
                      │
             audit-outbox-relay ──▶ NATS aios.*.audit.*
```

## 3. Visão C4 — Nível 3 (Componente)

```
   eventos JetStream (classes registradas)     API direta /v1/audit (mTLS)
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
   │  └──────────┬───────────┘   └─────────┬──────────┘    │                     │
   │             ▼                          │               │                     │
   │  ┌──────────────────────┐   ┌──────────▼──────────┐   │                     │
   │  │    PayloadCipher     │──▶│   SubjectKeyVault   │───┼─────────────────────┼─▶ 021
   │  └──────────┬───────────┘   └─────────────────────┘   │                     │
   │             ▼                                          │                     │
   │  ┌──────────────────────────────────────────────────┐ │                     │
   │  │   ChainAppender   seq · record_hash · prev_hash   │ │  APPEND-ONLY        │
   │  └──────────┬────────────────────────┬──────────────┘ │                     │
   │             ▼                         ▼                │                     │
   │  ┌──────────────────┐      ┌────────────────────────┐ │                     │
   │  │   SealService    │─────▶│    WormArchiver        │─┼─────────────────────┼─▶ MinIO
   │  └────────┬─────────┘      └────────────────────────┘ │                     │
   │           ▼                                            │                     │
   │  ┌──────────────────┐  ┌──────────────────────┐  ┌────▼──────────────────┐  │
   │  │IntegrityVerifier │  │ CompletenessMonitor  │  │   AnomalyDetector     │  │
   │  └──────────────────┘  └──────────────────────┘  └───────────────────────┘  │
   │  ┌──────────────────┐  ┌──────────────────────┐  ┌───────────────────────┐  │
   │  │ LegalHoldManager │─▶│  RetentionEnforcer   │  │   CryptoShredder      │  │
   │  └──────────────────┘  └──────────────────────┘  └───────────┬───────────┘  │
   │                        ┌──────────────────────────────────────▼───────────┐  │
   │                        │        ErasureReceiptService                     │  │
   │                        └──────────────────────────────────────────────────┘  │
   │  ┌────────────────────────────────────────────────────────────────────────┐ │
   │  │ EventEmitter (Outbox → JetStream) · AuditTelemetry (OTel, sem conteúdo) │ │
   │  └────────────────────────────────────────────────────────────────────────┘ │
   └──────────────────────────────────────────────────────────────────────────────┘
```

### 3.1 Responsabilidade por componente

| Componente | Responsabilidade | Depende de |
|------------|------------------|-----------|
| **AuditIngestGateway** | Ingestão por API e por evento; dedupe por `event_id`; **confirma só após durabilidade**. | RecordNormalizer, ChainAppender |
| **RecordNormalizer** | Converte o fato de origem no envelope canônico (`./Database.md` §2). | AuditableRegistry |
| **AuditableRegistry** | Classes auditáveis: produtor esperado, subjects de origem, retenção, base legal, volume esperado. | → PostgreSQL |
| **ChainAppender** | `seq` monotônico por partição, `record_hash`, `prev_hash`. Append-only. | → PostgreSQL |
| **PayloadCipher** | Cifra o payload com a chave do titular antes de persistir. | SubjectKeyVault |
| **SubjectKeyVault** | Titular → chave (custodiada no `../021-Security/`); destruição sob RTBF. | → 021-Security |
| **SealService** | Árvore de Merkle por intervalo, assinatura da raiz, cadeia de selos. | ChainAppender; → 021-Security |
| **IntegrityVerifier** | Recomputa cadeia, provas e assinaturas; compara com a cópia WORM. | SealService, WormArchiver |
| **CompletenessMonitor** | Lacuna de `seq`, selo atrasado, classe silenciosa. | AuditableRegistry, ChainAppender |
| **LegalHoldManager** | Aplica/libera *hold*; emite `audit.legalhold.applied`/`.released`. | EventEmitter |
| **RetentionEnforcer** | Expurga **payload** fora da retenção, preservando cadeia. | LegalHoldManager, WormArchiver |
| **CryptoShredder** | Destrói a chave do titular; marca registros como `Shredded`. | SubjectKeyVault, LegalHoldManager |
| **ErasureReceiptService** | Comprovantes próprios e recebidos de `005`/`010`/`024`. | ChainAppender, EventEmitter |
| **WormArchiver** | Arquiva intervalos selados em objeto com *object lock*. | SealService; → MinIO |
| **QueryService** | Consulta sob PEP; **registra a própria consulta** como fato auditável. | PolicyClient, ChainAppender |
| **ExportService** | Pacote probatório verificável offline. | IntegrityVerifier, SealService |
| **AnomalyDetector** | `chain_break`, `sequence_gap`, `seal_overdue`, `class_silent`, `volume_anomaly`, `query_pattern`. | CompletenessMonitor, IntegrityVerifier |
| **PolicyClient** | PEP; *fail-closed*. | → 022-Policy |
| **EventEmitter** | Outbox transacional → JetStream. | → PostgreSQL, NATS |
| **AuditTelemetry** | Métricas `aios_aud_*`, spans e logs — **sem** conteúdo auditado. | → 024-Observability |

## 4. Fronteiras arquiteturais

| Fronteira | Contrato | Documento |
|-----------|----------|-----------|
| Produtores → 025 | Envelope canônico de auditoria (RFC-0250) | `./API.md` §2 |
| 025 → `005-Database` | `audit.legalhold.applied` suspende expurgo | `./Events.md` |
| 025 → auditor externo | Pacote probatório verificável offline (RFC-0250) | `./API.md` §4 |
| 025 ↔ `021-Security` | Chave de assinatura de selo; chaves por titular | `../021-Security/` |
| 025 → `022-Policy` | PEP de consulta e governança | `./Security.md` |
| 025 → `024-Observability` | Métricas e anomalias; **nunca** conteúdo auditado | `./Metrics.md` |
| 025 ↔ WORM (MinIO) | *Object lock* por `aud.worm.object_lock_days` | `./Deployment.md` |

> A fronteira decisiva é com o `../024-Observability/` (ADR-0246): **telemetria é
> amostrável e perecível; auditoria não**. Um sistema que usa a mesma loja para as
> duas acaba com uma auditoria que perde eventos ou uma telemetria cara demais.

## 5. Padrões arquiteturais adotados

| Padrão | Onde | Por quê | ADR |
|--------|------|---------|-----|
| **Event sourcing / append-only log** | `ChainAppender` | O estado é a sequência de fatos; nada se altera. | ADR-0010 |
| **Hash chain** | `record_hash`/`prev_hash` | Alteração local quebra tudo o que vem depois. | ADR-0251 |
| **Merkle tree + selo assinado** | `SealService` | Prova de inclusão compacta e verificável offline. | ADR-0251 |
| **WORM (object lock)** | `WormArchiver` | Cópia fora do alcance do operador de banco. | ADR-0251 |
| **Cadeia particionada** | `partition_key` | Evita ponto único de serialização na escrita. | ADR-0252 |
| **Crypto-shredding** | `PayloadCipher` + `CryptoShredder` | Concilia imutabilidade e direito ao esquecimento. | ADR-0253 |
| **Durable-ack (fail-closed na escrita)** | `AuditIngestGateway` | Melhor recusar que perder; o produtor tem outbox. | ADR-0255 |
| **Registro de classes esperadas** | `AuditableRegistry` | Torna a **ausência** detectável. | ADR-0256 |
| **Transactional Outbox** | `EventEmitter` | Fato e evento na mesma transação. | ADR-0004 |
| **Compensating record** | `outcome = compensated` | Corrigir sem alterar. | ADR-0258 |

## 6. Alternativas descartadas

| Alternativa | Por que foi descartada | ADR |
|-------------|------------------------|-----|
| **Tabela append-only sem cadeia** (só convenção) | Imutabilidade não verificável: quem tem `UPDATE` reescreve a história sem deixar rastro. | ADR-0251 |
| **Assinar cada registro individualmente** | Custo de assinatura por registro inviável a 50.000/s; o selo agrega o mesmo efeito com uma assinatura por intervalo. | ADR-0251 |
| **Sequência global única por tenant** | Ponto de serialização absoluto na escrita; a trilha viraria o gargalo do AIOS. | ADR-0252 |
| **Blockchain público como âncora primária** | Latência, custo e dependência externa incompatíveis com 50.000 registros/s. Mantido como **âncora opcional** (`aud.seal.external_anchor_enabled`). | ADR-0251 |
| **Apagar registros para atender RTBF** | Destruiria a cadeia e a prova de que a ação ocorreu. | ADR-0253 |
| **Não apagar nada (imutabilidade absoluta)** | Viola a LGPD; não é uma opção. | ADR-0253 |
| **Aceitar escrita sem durabilidade confirmada** | Troca indisponibilidade visível por **lacuna invisível** — o pior defeito possível aqui. | ADR-0255 |
| **Reutilizar o armazenamento do `024`** | Telemetria é amostrável e perecível; auditoria não pode ser nenhuma das duas. | ADR-0246 |
| **Permitir correção de registro por operador** | Uma capability de alteração é uma capability de apagar evidência. | ADR-0258 |

## 7. Decisões estruturais e seus riscos

| Decisão | Risco assumido | Mitigação |
|---------|----------------|-----------|
| Confirmação síncrona e durável | Latência maior na escrita (p99 ≤ 20 ms) | Lote na API; partições paralelas; o custo é aceito por desenho (NFR-001) |
| Cadeia particionada | Não há ordem total entre partições | A ordem que importa é **causal** (`trace_id`, `occurred_at`), não total; cada partição é verificável isoladamente |
| Selo periódico (60 s) | Janela em que a adulteração ainda não é detectável por selo | `prev_hash` já detecta alteração local; o selo fecha a janela contra reescrita completa da cadeia |
| Crypto-shredding | Perda da chave torna o registro ilegível para sempre | É o efeito desejado; o `payload_digest` preserva a prova de existência |
| WORM com *object lock* longo | Custo de armazenamento e impossibilidade de correção | Retenção legal exige; `aud.retention.min_days` impede redução por conveniência |
| PEP no caminho de consulta | Consulta indisponível se o PDP cair | *Fail-closed* deliberado: a trilha não é lida por ninguém não autorizado, nem em incidente |

## 8. Referências

- Fonte única de verdade: `./_DESIGN_BRIEF.md` (§2 componentes, §10 escalabilidade)
- Arquitetura global: `../001-Architecture/Architecture.md`
- Contratos centrais: `../003-RFC/RFC-0001-Architecture-Baseline.md`
- Decisões: `../002-ADR/ADR-0010-Observabilidade-Auditoria-por-Construcao.md`, `./ADR.md`
- Detalhamento: `./ClassDiagrams.md`, `./SequenceDiagrams.md`, `./Database.md`,
  `./API.md`, `./Events.md`, `./Scalability.md`

*Fim de `Architecture.md`.*
