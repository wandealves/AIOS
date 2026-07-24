---
Documento: StateMachine
Módulo: 025-Audit
Status: Draft
Versão: 0.1
Última atualização: 2026-07-23
Responsável (RACI-A): Arquiteto do Módulo 025-Audit
ADRs relacionados: ADR-0010, ADR-0251, ADR-0252, ADR-0253, ADR-0254, ADR-0255, ADR-0258
RFCs relacionados: RFC-0001, RFC-0250, RFC-0251
Depende de: _DESIGN_BRIEF.md §4, Database.md, Events.md, UseCases.md
---

# 025-Audit — Máquina de Estados

## 1. Qual entidade tem ciclo de vida

A entidade governada por máquina de estados é o **`AuditRecord`** — com uma
particularidade que o distingue de todas as outras FSMs do AIOS: **nenhuma transição
altera o conteúdo do registro**.

O que muda ao longo do ciclo é o **status probatório** (coberto por selo ou não) e a
**legibilidade do payload** (íntegro, indecifrável ou expurgado). Os campos que
sustentam a cadeia — `seq`, `prev_hash`, `record_hash` — são imutáveis em **todos** os
estados, inclusive nos terminais.

Entidade com ciclo secundário: o **`LegalHold`** (§6).

## 2. Estados (`RecordState`)

| Estado | Descrição | Terminal? | Ação de entrada | Ação de saída |
|--------|-----------|-----------|-----------------|---------------|
| `Received` | Encadeado e durável, ainda **não coberto por selo**. | não | Grava `seq`, `prev_hash`, `record_hash`, `recorded_at` | — |
| `Sealed` | Coberto por selo assinado; adulteração retroativa detectável. | não | Grava `seal_ref`; emite `audit.record.sealed` | — |
| `Held` | Sob *legal hold*: retenção e apagamento **suspensos**. | não | Vincula ao `legal_hold`; emite `audit.legalhold.applied` | Emite `audit.legalhold.released` |
| `Shredded` | Chave do titular destruída; payload indecifrável. | **sim** | Emite `audit.erasure.completed` com comprovante | — |
| `Expired` | Retenção legal vencida; payload expurgado. | **sim** | Emite `audit.retention.expired` | — |

## 3. Transições, gatilhos e guardas

| # | De → Para | Gatilho | Guarda (DEVE ser verdadeira) | Erro se violada |
|---|-----------|---------|------------------------------|-----------------|
| T-01 | ∅ → `Received` | Ingestão (evento ou API) | `event_id` inédito no tenant ∧ `record_class` registrada ∧ `prev_hash` = `record_hash` do último `seq` da partição ∧ **escrita durável confirmada**. | `AIOS-AUD-0002`, `AIOS-AUD-0005` |
| T-02 | `Received` → `Sealed` | `SealService` fecha o intervalo | Registro incluído na árvore de Merkle ∧ raiz assinada pela chave corrente do `../021-Security/` ∧ selo encadeado ao anterior. | `AIOS-AUD-0014` |
| T-03 | `Sealed`/`Received` → `Held` | `ApplyLegalHold` | Escopo do *hold* casa com o registro ∧ `approved_by ≠ requested_by` ∧ `legal_basis` declarada. | `AIOS-AUD-0009` |
| T-04 | `Held` → `Sealed` | `ReleaseLegalHold` | Capability `aud:hold:release` ∧ `released_by` registrado ∧ **nenhum outro** *hold* vigente cobre o registro. | `AIOS-AUD-0008` |
| T-05 | `Sealed` → `Shredded` | Apagamento criptográfico (RTBF) | Registro **não** está em `Held` ∧ `subject_ref` presente ∧ chave do titular destruída ∧ comprovante emitido. | `AIOS-AUD-0008` |
| T-06 | `Sealed` → `Expired` | Fim da retenção legal | Registro **não** está em `Held` ∧ `recorded_at + retention_days` < agora ∧ intervalo já arquivado em WORM. | `AIOS-AUD-0008` |
| T-07 | `Received` → `Received` | Reingestão do mesmo `event_id` | Idempotência: nenhum novo registro é criado; a repetição é contabilizada, não gravada. | — |

**Transições que não existem — e por quê:**

| Transição ausente | Por que não existe |
|-------------------|--------------------|
| `Shredded`/`Expired` → qualquer | Terminais absolutos: a chave foi destruída ou o payload expurgado; não há caminho de volta. |
| Qualquer → ∅ (remoção) | Remover um elo tornaria toda a cadeia posterior inverificável (invariante I1). |
| `Received`/`Sealed` → alteração de conteúdo | Append-only (invariante I6); correção é **novo** registro com `outcome = compensated`. |
| `Held` → `Shredded`/`Expired` | O *legal hold* sobrepõe retenção e RTBF (invariante I5). |

## 4. Diagrama de estados (ASCII)

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

## 5. Invariantes

| ID | Invariante | Como é garantida |
|----|-----------|------------------|
| I1 | **Nenhum registro é removido da cadeia.** `seq`, `prev_hash` e `record_hash` permanecem em todos os estados. | Sem operação de `DELETE` de linha; o expurgo remove apenas `payload_cipher` (`./Database.md` §2). Remover um elo tornaria a cadeia posterior inverificável. |
| I2 | `prev_hash` do registro `seq = N` é **exatamente** o `record_hash` de `seq = N-1` na mesma partição. | Calculado pelo `ChainAppender` na transação de escrita; verificado continuamente. Divergência é `chain_break` — anomalia **crítica**. |
| I3 | A sequência por partição é **monotônica e sem lacuna**. | `UNIQUE(partition_key, seq)` + sequência dedicada; lacuna é `sequence_gap` — o defeito mais perigoso (`./Vision.md` §5). |
| I4 | `Shredded` e `Expired` preservam `seq`, hashes, `payload_digest`, `occurred_at`, `record_class` e `outcome`. | Some o **conteúdo**, nunca a **prova de que houve** um. O `payload_digest` impede que o apagamento seja confundido com um registro vazio. |
| I5 | Registro em `Held` **NÃO DEVE** transitar para `Shredded` nem `Expired`. | Guarda de T-05/T-06 + `AIOS-AUD-0008`. Precedência jurídica documentada, não escolha técnica. |
| I6 | **Append-only**: nenhum caminho de `UPDATE` sobre `actor`, `action`, `resource_urn`, `outcome`, `payload_digest` ou campos de cadeia. | Ausência de operação na API (`AIOS-AUD-0016`) + *trigger* de banco; correção é registro de compensação. |
| I7 | Ingestão é **idempotente** por `(tenant_id, event_id)`. | `UNIQUE(tenant_id, event_id)` (RFC-0001 §5.5). |
| I8 | Todo registro é selado em ≤ `aud.seal.interval_s`. | `CompletenessMonitor`; excedente é `seal_overdue`. |
| I9 | A escrita só é confirmada **após durabilidade**. | `aud.ingest.require_durable_ack` (não-recarregável); do contrário, `AIOS-AUD-0005`. |

> **Por que I4 preserva o `payload_digest`.** Depois de destruída a chave, o conteúdo é
> ilegível — mas o digest prova que **existia** um conteúdo com aquele hash. Sem ele, o
> apagamento seria indistinguível de um registro que sempre foi vazio, e um adversário
> poderia alegar que nada havia ali.

## 6. Ciclo de vida do `LegalHold`

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

### 6.1 A exceção deliberada: *hold* pode não ter prazo

| Artefato | `expires_at` | Justificativa |
|----------|--------------|---------------|
| Waiver do `../022-Policy/` | **Obrigatório** (72 h) | Exceção sem prazo é política nova aprovada por ninguém. |
| Silêncio do `../024-Observability/` | **Obrigatório** (72 h) | Silêncio permanente é o alerta excluído sem decisão. |
| **`LegalHold` do 025** | **PODE ser nulo** | Um *hold* não expira porque um prazo técnico venceu; expira quando o **caso encerra**. |

O controle compensatório é a **revisão periódica obrigatória**
(`aud.hold.review_interval_days`, default 90) de todos os *holds* ativos, mais a
exigência de `case_ref` e `legal_basis` no momento da aplicação.

## 7. Verificação de integridade (não é FSM)

```
   verificação contínua (amostragem)          verificação completa (sob demanda)
   ────────────────────────────────           ─────────────────────────────────
   a cada aud.verify.interval_s (900 s):      para uma partição/janela:
     · sorteia aud.verify.sample_size (1000)    1. recomputa record_hash de cada registro
     · recomputa record_hash                    2. confere prev_hash encadeado
     · confere prova de Merkle no selo          3. recomputa a raiz de Merkle
     · confere assinatura do selo               4. verifica assinatura do selo (chave 021)
                                                5. confere cadeia de selos (prev_seal_hash)
   qualquer divergência ⇒ chain_break          6. compara com a cópia WORM
   (severidade CRÍTICA, alerta imediato)
```

O passo 6 fecha o modelo: mesmo com acesso total ao PostgreSQL, um adversário não
consegue reescrever a cópia arquivada em objeto com *object lock*, nem forjar a
assinatura cuja chave privada está no `../021-Security/`. **Adulterar exige comprometer
três sistemas distintos.**

### 7.1 O que cada camada detecta

| Camada | Detecta | Não detecta sozinha |
|--------|---------|---------------------|
| `prev_hash` (cadeia) | Alteração de um registro isolado | Reescrita completa e consistente da cadeia |
| Selo assinado | Reescrita da cadeia após a selagem | Adulteração na janela antes do selo |
| Cadeia de selos | Remoção ou substituição de um selo inteiro | — |
| Cópia WORM | Divergência entre banco e arquivo | Adulteração antes do arquivamento |
| `sequence_gap` | **Registro que nunca chegou** | Registro alterado (o hash cobre isso) |

Nenhuma camada é suficiente sozinha; juntas, a janela de ataque é o intervalo entre a
escrita e a selagem (≤ 60 s) — e mesmo nela o `prev_hash` já detecta alteração local.

## 8. Interação entre as máquinas

```
   AuditRecord: Received → Sealed
                              │
                              ├── audit.record.sealed ──▶ prova disponível para consulta
                              │
   LegalHold: Requested → Active
                              │
                              ├── audit.legalhold.applied ──▶ 005-Database suspende expurgo
                              │                              025 bloqueia T-05 e T-06
                              ▼
                          Released ──▶ audit.legalhold.released ──▶ expurgo pode retomar

   security.rtbf.requested (021) ──▶ T-05 (Shredded), SE não houver hold
                                     └──▶ audit.erasure.completed + comprovante na trilha
```

> A seta mais importante do diagrama é a que **não** existe: `Held` nunca chega a
> `Shredded`. Um pedido de esquecimento sobre registro sob *hold* é recusado com
> `AIOS-AUD-0008`, e a recusa — como toda operação aqui — é ela mesma registrada.

## 9. Referências

- Fonte única de verdade: `./_DESIGN_BRIEF.md` §4
- Modelo físico: `./Database.md` · Eventos: `./Events.md`
- Fluxos: `./SequenceDiagrams.md` · Casos de uso: `./UseCases.md`
- Erros: `./API.md` §6 · Segurança: `./Security.md`

*Fim de `StateMachine.md`.*
