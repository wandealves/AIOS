---
Documento: SequenceDiagrams
Módulo: 025-Audit
Status: Draft
Versão: 0.1
Última atualização: 2026-07-23
Responsável (RACI-A): Arquiteto do Módulo 025-Audit
ADRs relacionados: ADR-0010, ADR-0251, ADR-0253, ADR-0254, ADR-0255, ADR-0259
RFCs relacionados: RFC-0001, RFC-0250, RFC-0251
Depende de: UseCases.md, API.md, Events.md, StateMachine.md
---

# 025-Audit — Diagramas de Sequência

> Todos os fluxos propagam `traceparent` (W3C Trace Context) e `X-AIOS-Tenant`
> conforme `../003-RFC/RFC-0001-Architecture-Baseline.md` §5.6. Timeouts indicados são
> os defaults de `./Configuration.md`.

## 1. UC-001 — Registro por evento (caminho principal)

```
 022-Policy      NATS/JetStream   AuditIngest    RecordNorm.   PayloadCipher  ChainAppender  PG
     │                 │              │              │              │              │        │
     │ outbox commit   │              │              │              │              │        │
     │────────────────▶│ publica      │              │              │              │        │
     │                 │─────────────▶│ dedupe por event_id         │              │        │
     │                 │              │──────┐       │              │              │        │
     │                 │              │◀─────┘ inédito│              │              │        │
     │                 │              │─────────────▶│ envelope canônico            │        │
     │                 │              │              │─────────────▶│ cifra c/ chave│        │
     │                 │              │              │              │  do titular   │        │
     │                 │              │              │◀─────────────│ + payload_digest       │
     │                 │              │◀─────────────│              │              │        │
     │                 │              │ append ──────────────────────────────────▶│        │
     │                 │              │              │   seq · record_hash · prev_hash       │
     │                 │              │              │              │              │───────▶│
     │                 │              │              │              │              │ COMMIT │
     │                 │              │              │              │              │ durável│
     │                 │              │◀───────────────────────────────────────────│◀───────│
     │                 │◀── ACK ──────│  (SÓ AQUI — invariante I9)                 │        │
     │                 │              │                                            │        │
     │                 │  estado = Received; aguarda selagem (UC-003)              │        │
```

> O `ACK` ao JetStream ocorre **após o commit durável**, nunca antes. Confirmar antes
> significaria que uma queda entre o `ACK` e o commit produziria uma **lacuna
> invisível** — o defeito que este módulo existe para impedir.

## 2. UC-002 (E1) — Banco indisponível: recusar é o correto

```
   Produtor            AuditIngestGateway            PostgreSQL
      │                        │                          │
      │ POST /v1/audit/records │                          │
      │ Idempotency-Key: 01J9…│                          │
      │───────────────────────▶│ append                   │
      │                        │─────────────────────────▶│  ✗ indisponível
      │                        │◀── falha ────────────────│
      │                        │ NÃO confirma             │
      │◀── 503 AIOS-AUD-0005 ──│ retriable=true           │
      │    retryAfterMs: 2000  │                          │
      │                        │                          │
      │ mantém o fato no PRÓPRIO outbox e reenvia         │
      │───────────────────────▶│ (mesmo Idempotency-Key)  │
      │                        │─────────────────────────▶│  ✓ recuperado
      │◀── 201 Created ────────│                          │
      │    seq=418291  record_hash=sha256:7a1c…           │
```

Compare com o `../024-Observability/`, que responde **200 e descarta** sob saturação:
os dois módulos fazem escolhas opostas porque otimizam garantias opostas — telemetria
não pode atrasar o emissor; auditoria não pode perder o fato.

## 3. UC-003 — Selagem de um intervalo

```
  SealService        ChainAppender/PG      021-Security      Outbox/NATS    WormArchiver
       │                    │                    │                │              │
       │ a cada 60 s        │                    │                │              │
       │ seleciona [seq_from, seq_to]            │                │              │
       │───────────────────▶│                    │                │              │
       │◀── record_hash[] ──│                    │                │              │
       │ constrói árvore de Merkle               │                │              │
       │──────┐             │                    │                │              │
       │◀─────┘ merkle_root │                    │                │              │
       │ seal_hash = H(root ‖ intervalo ‖ prev_seal_hash ‖ time)  │              │
       │ assina seal_hash ───────────────────────▶│               │              │
       │◀── assinatura ──────────────────────────│                │              │
       │ TRANSAÇÃO: grava selo + registros → Sealed + outbox      │              │
       │───────────────────▶│                    │                │              │
       │                    │                    │  audit.record.sealed ────────▶│
       │                    │                    │                │─────────────▶│ agenda
       │                    │                    │                │              │ archive
```

## 4. UC-006 — Quebra de cadeia: três fontes independentes

```
 IntegrityVerifier      PostgreSQL        Selo assinado        WORM (MinIO)      CISO
        │                   │                   │                   │             │
        │ recomputa record_hash                 │                   │             │
        │──────────────────▶│                   │                   │             │
        │◀── divergente ────│                   │                   │             │
        │ CONGELA escrita da partição           │                   │             │
        │ audit.anomaly.detected{chain_break, critical} ────────────────────────▶│
        │                   │                   │                   │             │
        │ compara com a raiz de Merkle do selo  │                   │             │
        │──────────────────────────────────────▶│                   │             │
        │◀── selo concorda com o registro ORIGINAL                   │             │
        │ compara com a cópia arquivada         │                   │             │
        │──────────────────────────────────────────────────────────▶│             │
        │◀── WORM concorda com o selo ──────────────────────────────│             │
        │                                                            │             │
        │ CONCLUSÃO: banco adulterado; evidência íntegra = WORM + selo             │
        │───────────────────────────────────────────────────────────────────────▶│
```

Adulterar a trilha exigiria reescrever o banco, **recomputar todos os selos
posteriores**, forjar assinaturas cuja chave privada está no `../021-Security/` **e**
sobrescrever objetos com *object lock* — três sistemas com controles distintos.

## 5. UC-008 + UC-010 — *Legal hold* bloqueia o direito ao esquecimento

```
 Jurídico   LegalHoldMgr   Outbox/NATS   005-Database   021-Security   CryptoShredder  Titular
    │            │              │              │              │              │           │
    │ ApplyLegalHold(case=PROC-4821, escopo)   │              │              │           │
    │───────────▶│ approved_by ≠ requested_by  │              │              │           │
    │            │ registros → Held (T-03)     │              │              │           │
    │            │ TRANSAÇÃO: hold + outbox    │              │              │           │
    │            │─────────────▶│ audit.legalhold.applied     │              │           │
    │            │              │─────────────▶│ SUSPENDE expurgo            │           │
    │            │              │              │              │              │           │
    │  ... dias depois ...      │              │              │              │           │
    │            │              │              │              │              │           │
    │            │              │  security.rtbf.requested ◀──│──────────────│           │
    │            │              │─────────────────────────────────────────▶│ verifica    │
    │            │◀────────────────────────── consulta hold ───────────────│  hold       │
    │            │── HOLD VIGENTE ────────────────────────────────────────▶│             │
    │            │                                             RECUSA (AIOS-AUD-0008)    │
    │            │  ↳ a recusa é REGISTRADA na trilha como fato             │───────────▶│
    │            │     titular é informado da obrigação legal de preservação            │
```

## 6. UC-010 — Apagamento criptográfico (sem *hold*)

```
 021-Security    NATS    CryptoShredder   SubjectKeyVault   021 (KMS)   ChainAppender  Outbox
      │            │           │                │              │              │           │
      │ rtbf.requested         │                │              │              │           │
      │───────────▶│──────────▶│ verifica hold: nenhum         │              │           │
      │            │           │───────────────▶│ destrói chave│              │           │
      │            │           │                │─────────────▶│ DESTROY      │           │
      │            │           │                │◀── ok ───────│              │           │
      │            │           │ registros do titular → Shredded (T-05)       │           │
      │            │           │──────────────────────────────────────────────▶│          │
      │            │           │   payload_cipher permanece (ILEGÍVEL)         │          │
      │            │           │   seq · record_hash · payload_digest INTACTOS │          │
      │            │           │ comprovante → registro NA PRÓPRIA TRILHA ─────▶│          │
      │            │           │──────────────────────────────────────────────────────────▶│
      │            │◀── audit.erasure.completed ────────────────────────────────────────────│
      │            │                                                                       │
      │  verificação pós-apagamento: cadeia CONTINUA passando (FR-012)                     │
```

## 7. UC-012 — Consulta: a própria consulta vira registro

```
  Investigador     QueryService    PolicyClient   022-Policy   PG (trilha)   ChainAppender
       │                │                │             │            │              │
       │ query(subject_ref=tok:9f2c…)    │             │            │              │
       │───────────────▶│ DecisionRequest│             │            │              │
       │                │───────────────▶│  Evaluate   │            │              │
       │                │                │────────────▶│            │              │
       │                │                │◀── allow ───│            │              │
       │                │◀───────────────│             │            │              │
       │                │ escopo por tenant + orçamento│            │              │
       │                │──────────────────────────────────────────▶│              │
       │                │◀── registros (payload decifrado) ─────────│              │
       │◀── resultado ──│                │             │            │              │
       │                │ REGISTRA A CONSULTA (classe audit.access) ──────────────▶│
       │                │   quem · escopo · volume · quando                        │
```

> A última seta é a que muitos sistemas de auditoria esquecem. Quem consultou a trilha
> — e com que escopo — é exatamente o tipo de informação que um auditor precisa, e a
> ausência desse registro seria a lacuna mais conveniente de todas (ADR-0257).

## 8. UC-014 — Exportação e verificação offline pelo auditor

```
 Auditor   Responsável   ExportService   IntegrityVerifier   MinIO      (fora do AIOS)
    │           │              │                 │             │              │
    │ solicita  │              │                 │             │              │
    │──────────▶│ RequestExport (escopo, justificativa)        │              │
    │           │─────────────▶│ aprovação dupla │             │              │
    │           │              │ coleta registros + selos + provas de Merkle  │
    │           │              │────────────────▶│ verifica antes de exportar │
    │           │              │◀── íntegro ─────│             │              │
    │           │              │ monta pacote + chaves públicas + INSTRUÇÕES  │
    │           │              │──────────────────────────────▶│ grava objeto │
    │           │◀── package_hash ──────────────│              │              │
    │◀── link (TTL 72 h) ──────│                │              │              │
    │                                                          │              │
    │ baixa e verifica SEM ACESSO AO AIOS ────────────────────────────────────▶│
    │   1. recomputa record_hash de cada registro                              │
    │   2. recomputa a raiz de Merkle pelo caminho fornecido                   │
    │   3. verifica a assinatura do selo com a chave pública                   │
    │   4. confere a cadeia de selos (prev_seal_hash)                          │
    │◀────────────────── verificação independente concluída ──────────────────│
```

## 9. Fluxo de falha — chave de assinatura indisponível

```
   SealService          021-Security        AuditIngestGateway     Monitoring
        │                    │                       │                  │
        │ assina seal_hash   │                       │                  │
        │───────────────────▶│  ✗ KMS indisponível   │                  │
        │◀── falha ──────────│                       │                  │
        │ SELAGEM ADIADA     │                       │                  │
        │                    │                       │                  │
        │  ✅ A INGESTÃO CONTINUA ──────────────────▶│ registros seguem │
        │     (encadeados, em Received)              │  sendo gravados  │
        │                    │                       │                  │
        │ registros em Received > aud.seal.interval_s│                  │
        │───────────────────────────────────────────────────────────────▶│ seal_overdue
        │                    │                       │                  │
        │  ... KMS restabelecido ...                 │                  │
        │───────────────────▶│ assina o intervalo pendente (retroativo) │
        │◀── assinatura ─────│                       │                  │
```

A hierarquia de degradação (`./FailureRecovery.md` §5) aparece aqui: perde-se
temporariamente a **prova de selagem**, nunca o **fato**. Um registro não selado ainda
está encadeado — e o `prev_hash` já detecta alteração local.

## 10. Referências

- Casos de uso: `./UseCases.md` · FSM: `./StateMachine.md`
- Contratos: `./API.md` · `./Events.md`
- Falhas: `./FailureRecovery.md` · Segurança: `./Security.md`
- Convenções de diagrama: `../_templates/MODULE_TEMPLATE.md`

*Fim de `SequenceDiagrams.md`.*
