---
Documento: ClassDiagrams
Módulo: 025-Audit
Status: Draft
Versão: 0.1
Última atualização: 2026-07-23
Responsável (RACI-A): Arquiteto do Módulo 025-Audit
ADRs relacionados: ADR-0010, ADR-0251, ADR-0252, ADR-0253, ADR-0255, ADR-0258
RFCs relacionados: RFC-0001, RFC-0250, RFC-0251
Depende de: Architecture.md, StateMachine.md, API.md
---

# 025-Audit — Diagramas de Classe e Contratos

> As interfaces abaixo são **conceituais** (este repositório é de documentação, não de
> implementação) e servem para fixar as fronteiras testáveis do módulo, referenciadas
> em `./Testing.md`. Nomes de componentes são idênticos aos de `./Architecture.md` §3.1.

## 1. Visão estrutural do caminho de ingestão

```
   ┌──────────────────────────────────────────────────────────────────────┐
   │                      AuditIngestGateway                              │
   │  + Append(rec: AuditRecordDraft, ctx): AppendResult                  │
   │  + AppendBatch(recs: AuditRecordDraft[], ctx): AppendResult[]        │
   │  - Deduplicate(tenantId, eventId): ExistingRecord?                   │
   │  invariante: confirma SOMENTE após durabilidade (I9, ADR-0255)       │
   └────────────────────────────────┬─────────────────────────────────────┘
                                    │ ◆ (composição)
                                    ▼
   ┌──────────────────────────────────────────────────────────────────────┐
   │                       RecordNormalizer                               │
   │  + Normalize(source: SourceFact, cls: AuditableClass): CanonicalRecord│
   └───────────────┬──────────────────────────────────┬───────────────────┘
                   │                                  │
                   ▼                                  ▼
        ┌────────────────────┐            ┌──────────────────────┐
        │ AuditableRegistry  │            │    PayloadCipher     │
        │ (classes esperadas)│            │  ◇── SubjectKeyVault │
        └────────────────────┘            └──────────┬───────────┘
                                                     ▼
   ┌──────────────────────────────────────────────────────────────────────┐
   │                        ChainAppender                                 │
   │  + Append(rec: CanonicalRecord): ChainedRecord                       │
   │  - NextSeq(partitionKey): long          (monotônico, sem lacuna)     │
   │  - ComputeHash(rec, prevHash): string   (sha256 canônico)            │
   │  invariante: APPEND-ONLY — não existe Update nem Delete de linha     │
   └──────────────────────────────────────────────────────────────────────┘
```

> A ausência de `Update`/`Delete` nesta interface **é** o desenho. Uma capability de
> alteração seria uma capability de apagar evidência (ADR-0258); o modelo a torna
> inexistente, não apenas negada.

## 2. Interfaces do caminho de ingestão

```
interface IAuditIngestGateway {
  AppendResult   Append(AuditRecordDraft rec, ProducerContext ctx);
  AppendResult[] AppendBatch(AuditRecordDraft[] recs, ProducerContext ctx);
  // AppendResult NUNCA é devolvido antes do commit durável (ADR-0255)
}

interface IRecordNormalizer {
  CanonicalRecord Normalize(SourceFact fact, AuditableClass cls);
  // preenche actor · action · resourceUrn · outcome · decisionRef · traceId
}

interface IAuditableRegistry {
  AuditableClass? Resolve(string classKey);
  AuditableClass? ResolveBySubject(string natsSubject);
  AuditableClass  Register(AuditableClassDraft draft);   // valida retenção e base legal
  ExpectedVolume  ExpectedFor(string classKey);          // insumo do CompletenessMonitor
}

interface IPayloadCipher {
  CipheredPayload Encrypt(object payload, SubjectRef? subject, string tenantId);
  object?         Decrypt(CipheredPayload p, AuthContext ctx);  // null se chave destruída
  string          Digest(object payload);                        // sha256 do texto em claro
}

interface ISubjectKeyVault {
  KeyRef  KeyFor(SubjectRef subject, string tenantId);   // cria se inexistente
  void    Destroy(SubjectRef subject, string reason);     // apagamento criptográfico
  bool    IsDestroyed(SubjectRef subject);
}

interface IChainAppender {
  ChainedRecord Append(CanonicalRecord rec);
  long          NextSeq(string partitionKey);
  string        ComputeHash(CanonicalRecord rec, string prevHash);
  string        LastHash(string partitionKey);
  // NÃO EXISTE: Update(...), Delete(...)
}
```

### 2.1 Contrato do envelope canônico (RFC-0250)

```
record AuditRecordDraft {
  string      recordClass;    // registrada em AuditableRegistry
  string      occurredAt;     // RFC 3339 UTC, fornecido pelo produtor
  Actor       actor;          // { urn, kind ∈ {human, service, agent, external} }
  string      action;         // <dominio>:<objeto>:<verbo>
  string      resourceUrn;    // RFC-0001 §5.1
  string      outcome;        // allowed|denied|succeeded|failed|compensated
  string?     decisionRef;    // decision_id do 022-Policy
  string?     subjectRef;     // titular pseudonimizado
  object      payload;        // detalhe do fato — será cifrado
  string?     eventId;        // dedupe
  string      traceparent;    // RFC-0001 §5.6
}

record ChainedRecord {
  string id; long seq; string partitionKey;
  string recordHash; string prevHash; string payloadDigest;
  string recordedAt; RecordState state;
}

record AppendResult { string id; long seq; string recordHash; bool duplicate; }
```

**Invariantes de contrato:**

- (C1) `AppendResult` só é produzido após commit durável (I9). Não há resultado
  "aceito provisoriamente".
- (C2) `recordHash = sha256(canonical(campos imutáveis) ‖ prevHash)` — a serialização
  canônica é normativa (RFC-0250) para que o auditor recompute o mesmo valor.
- (C3) `payloadDigest` é calculado sobre o **texto em claro**, antes da cifragem, e
  **sobrevive** ao apagamento (invariante I4).
- (C4) `duplicate = true` devolve o registro original **com o mesmo `recordHash`**
  (NFR-014).
- (C5) `subjectRef` presente ⇒ payload cifrado com chave **do titular**; ausente ⇒
  chave do tenant.
- (C6) `outcome = compensated` **DEVE** referenciar o registro corrigido no payload —
  é o único mecanismo de correção (ADR-0258).

## 3. Estruturas de integridade e prova

```
   ┌─────────────────────┐  agrega   ┌──────────────────────────┐
   │    SealService      │──────────▶│         Seal             │
   │ + Seal(partition,   │           │ - merkleRoot             │
   │     range): Seal    │           │ - prevSealHash ──────────┼──┐ cadeia
   │ + InclusionProof(   │           │ - sealHash               │  │ de selos
   │     recordId): Proof│           │ - signature / signedBy   │◀─┘
   └──────────┬──────────┘           │ - wormObject             │
              │                       └──────────────────────────┘
              │ verifica                          ▲
              ▼                                   │ compara
   ┌─────────────────────┐            ┌───────────┴──────────┐
   │  IntegrityVerifier  │───────────▶│     WormArchiver     │
   │ + VerifySample(n)   │            │ + Archive(seal)      │
   │ + VerifyFull(range) │            │ + Fetch(sealUrn)     │
   └──────────┬──────────┘            └──────────────────────┘
              │ divergência
              ▼
   ┌─────────────────────┐            ┌──────────────────────┐
   │  AnomalyDetector    │◀───────────│ CompletenessMonitor  │
   │ + Raise(kind, sev)  │            │ + CheckGaps()        │
   └─────────────────────┘            │ + CheckSilence()     │
                                       └──────────────────────┘
```

```
interface ISealService {
  Seal            Seal(string partitionKey, SeqRange range);
  InclusionProof  InclusionProof(string recordId);   // caminho de Merkle + selo
  Seal?           SealCovering(string recordId);
}

interface IIntegrityVerifier {
  VerificationResult VerifySample(int sampleSize);          // contínua
  VerificationResult VerifyFull(string partitionKey, TimeWindow w);  // sob demanda
  // VerifyFull compara com a cópia WORM — a 3ª fonte independente
}

interface ICompletenessMonitor {
  Gap[]      CheckGaps(string partitionKey);
  SilentClass[] CheckSilence();     // volume < expected × silence_ratio
  Overdue[]  CheckSealOverdue();
}

interface ILegalHoldManager {
  LegalHold Apply(LegalHoldRequest req, string approvedBy);  // approvedBy ≠ requestedBy
  void      Release(string holdUrn, string releasedBy, string reason);
  bool      IsHeld(ChainedRecord rec);        // consultado por Retention e CryptoShredder
  LegalHold[] Active(string tenantId);        // insumo da revisão periódica
}

interface IRetentionEnforcer {
  PurgeResult PurgeExpired(string recordClass);   // pula registros em Held (I5)
}

interface ICryptoShredder {
  ErasureReceipt Shred(SubjectRef subject, string tenantId);  // bloqueado por Held
}

interface IExportService {
  AuditExport Request(ExportScope scope, string requestedBy, string approvedBy);
  ExportPackage Build(string exportUrn);   // registros + selos + provas + instruções
}
```

## 4. Relações entre componentes

| Relação | Tipo | Semântica |
|---------|------|-----------|
| `AuditIngestGateway` ◆── `RecordNormalizer` | Composição | Não há rota de ingestão que ignore a normalização. |
| `RecordNormalizer` ──▶ `AuditableRegistry` | Dependência | Classe desconhecida rejeita o fato (`AIOS-AUD-0002`). |
| `PayloadCipher` ◇── `SubjectKeyVault` | Agregação | O cofre é compartilhado; a chave vive no `../021-Security/`. |
| `ChainAppender` ⊘ `Update`/`Delete` | **Ausência** de operação | Append-only por construção (ADR-0258). |
| `SealService` ──▶ `021-Security` | Dependência | A chave privada de assinatura **não** está neste módulo. |
| `IntegrityVerifier` ──▶ `WormArchiver` | Dependência | A verificação completa exige a terceira fonte. |
| `RetentionEnforcer` ──▶ `LegalHoldManager` | Dependência | Consulta obrigatória antes de qualquer expurgo (I5). |
| `CryptoShredder` ──▶ `LegalHoldManager` | Dependência | Mesma guarda: *hold* bloqueia o apagamento. |
| `QueryService` ──▶ `ChainAppender` | Dependência | A **consulta gera um registro** (ADR-0257). |
| `ExportService` ◆── `IntegrityVerifier` | Composição | Nada é exportado sem verificação prévia. |

## 5. Invariantes estruturais

| ID | Invariante | Consequência do rompimento |
|----|-----------|----------------------------|
| S1 | Não existe operação de alteração ou remoção de registro em nenhuma interface. | Uma capability de alteração é uma capability de apagar evidência. |
| S2 | O `AppendResult` só existe após commit durável. | Lacuna invisível (NFR-006 cai). |
| S3 | A chave privada de assinatura de selo **não** reside neste módulo. | Comprometer o 025 bastaria para forjar selos. |
| S4 | `RetentionEnforcer` e `CryptoShredder` consultam `LegalHoldManager` antes de agir. | Destruição de prova sob obrigação legal (NFR-010 cai). |
| S5 | `IntegrityVerifier.VerifyFull` compara com o WORM. | A verificação passaria a confiar na mesma fonte que deveria auditar. |
| S6 | `QueryService` sempre gera registro de acesso. | A consulta à trilha seria a única ação privilegiada não auditada. |
| S7 | `payloadDigest` é calculado antes da cifragem e nunca recalculado. | Após o apagamento, não haveria prova de que houve conteúdo. |

## 6. Modelo de domínio (entidades persistidas)

```
   ┌────────────────────┐ *      1 ┌──────────────────────┐
   │    AuditRecord     │──────────│   AuditableClass     │
   │ - seq / prevHash   │  class   │ - retentionDays      │
   │ - recordHash       │          │ - legalBasis         │
   │ - payloadCipher    │          │ - containsPii        │
   │ - payloadDigest    │          │ - expectedRatePerDay │
   │ - state (FSM §2)   │          └──────────────────────┘
   └───┬────────┬───────┘
       │ sealRef│ subjectRef
       ▼        ▼
   ┌────────┐ ┌──────────────┐        ┌────────────────────┐
   │  Seal  │ │  SubjectKey  │───────▶│  ErasureReceipt    │
   │ merkle │ │ - state      │ destroy│ - receiptHash      │
   │ signat.│ │ - keyRef     │        │ - chainRecordId ───┼──▶ AuditRecord
   └───┬────┘ └──────────────┘        └────────────────────┘
       │ wormObject
       ▼
   ┌────────────┐        ┌────────────────┐      ┌──────────────┐
   │ WORM obj.  │        │   LegalHold    │      │ AuditAnomaly │
   │ object lock│        │ - caseRef      │      │ - kind       │
   └────────────┘        │ - expiresAt(∅) │      │ - severity   │
                          └────────────────┘      └──────────────┘
   ┌────────────────────┐
   │    AuditExport     │──inclui──▶ AuditRecord + Seal + InclusionProof
   └────────────────────┘
```

Detalhamento físico (tipos, constraints, índices) em `./Database.md`.

## 7. Referências

- Componentes: `./Architecture.md` §3.1 · FSM: `./StateMachine.md`
- Contratos externos: `./API.md` · `./Events.md`
- Fonte única de verdade: `./_DESIGN_BRIEF.md` §2 e §5
- Testes das fronteiras: `./Testing.md` §3

*Fim de `ClassDiagrams.md`.*
