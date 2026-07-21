---
Documento: StateMachine
Módulo: 010-Memory
Status: Draft
Versão: 0.1
Última atualização: 2026-07-20
Responsável (RACI-A): Arquiteto do Módulo 010-Memory
ADRs relacionados: ADR-0102, ADR-0103, ADR-0109
RFCs relacionados: RFC-0001, RFC-0100
Depende de: 001-Architecture, 003-RFC (RFC-0001), 023-Learning, 025-Audit, 040-Glossary
---

# Máquinas de Estado — Módulo 010-Memory

> Reproduz e detalha as máquinas de estado da §4 do
> [`_DESIGN_BRIEF.md`](./_DESIGN_BRIEF.md): `MemoryItem` (§4.1) e `ConsolidationJob`
> (§4.2). Estados, transições, gatilhos, guardas, ações e invariantes são **canônicos**
> e idênticos aos usados em [`ClassDiagrams.md`](./ClassDiagrams.md),
> [`SequenceDiagrams.md`](./SequenceDiagrams.md), [`UseCases.md`](./UseCases.md) e
> [`Database.md`](./Database.md). Diagramas **ASCII**; palavras normativas RFC 2119/8174.
> Cada transição de estado observável emite evento de domínio via Outbox (§6 do brief).

## 1. Convenções

- **Gatilho:** o evento/comando que provoca a avaliação da transição.
- **Guarda:** condição booleana que DEVE ser verdadeira para a transição ocorrer.
- **Ação:** efeito colateral executado atomicamente com a transição (persistência +
  `OutboxEvent`, RFC-0001 §5.2).
- Estados **terminais** não têm transições de saída (salvo retry declarado).

---

## 2. Máquina de estado do `MemoryItem`

### 2.1 Diagrama (ASCII)

```
                remember()                     consolidate() (023)
    ┌──────────┐  ok    ┌────────┐  reforço/uso  ┌────────────────┐   promovido   ┌──────────────┐
──▶ │ INGESTED │───────▶│ ACTIVE │◀─────────────▶│ CONSOLIDATING  │──────────────▶│ CONSOLIDATED │
    └──────────┘        └───┬────┘               └───────┬────────┘               └──────┬───────┘
       │ falha embedding    │ decay/TTL                  │ falha/conflito                │ decay
       ▼                    ▼                            ▼ rollback                      ▼
    ┌────────┐          ┌──────────┐                 (volta ACTIVE)               ┌──────────────┐
    │ FAILED │          │ DECAYING │◀──────────────────────────────────────────── │  ARCHIVED    │
    └───┬────┘          └────┬─────┘   move p/ frio (MinIO)                        │ (cold/MinIO) │
        │ retry              │ score<θ  ou  expires_at  ou  RTBF  ou  cota         └──────┬───────┘
        │ (backoff)          ▼                                                             │ reidrata
        └────▶INGESTED  ┌───────────────┐   forget() confirma   ┌───────────┐             │
                        │ FORGET_PENDING│──────────────────────▶│ FORGOTTEN │◀────────────┘
                        └───────────────┘                       │(tombstone)│
                                                    grace period ⌛ │ purge físico + RTBF
                                                                    ▼
                                                              ┌──────────┐
                                                              │  PURGED  │ (terminal)
                                                              └──────────┘
```

### 2.2 Estados

| Estado | Tipo | Descrição | Ação de entrada |
|--------|------|-----------|-----------------|
| `INGESTED` | inicial | Item aceito e persistido; embedding pode estar pendente. | Reserva cota; agenda embedding se aplicável. |
| `ACTIVE` | estável | Item vivo, consultável por recall. | Emite `item.stored`; atualiza `QuotaManager`. |
| `CONSOLIDATING` | transitório | Sob promoção/mescla entre camadas. | Grava `ConsolidationVersion` (pré-imagem); emite `consolidation.started`. |
| `CONSOLIDATED` | estável | Promovido/mesclado; versão ativa. | Emite `item.consolidated`; atualiza `parent_ids`. |
| `ARCHIVED` | estável (frio) | Conteúdo movido para MinIO; metadados em PG. | Emite `item.archived`; libera armazenamento quente. |
| `DECAYING` | transitório | `decay_score` abaixo do limiar; candidato a esquecer. | Emite `item.decayed`. |
| `FORGET_PENDING` | transitório | Marcado para esquecer; aguarda confirmação. | Cria tombstone (`tombstoned_at`). |
| `FORGOTTEN` | estável (lógico) | Esquecido logicamente; tombstone + linhagem preservados. | Emite `item.forgotten`. |
| `PURGED` | **terminal** | Removido fisicamente (vetor + blob + linha). | Emite `item.purged`; auditoria RTBF (025). |
| `FAILED` | **terminal recuperável** | Erro de persistência/embedding após N retries. | Registra causa; alerta. Retry → `INGESTED`. |

### 2.3 Tabela de transições

| # | De → Para | Gatilho | Guarda | Ação / evento |
|---|-----------|---------|--------|---------------|
| T1 | `INGESTED→ACTIVE` | escrita persistida + embedding pronto (se aplicável) | cota disponível; PDP allow; embedding ok | emite `item.stored` |
| T2 | `INGESTED→FAILED` | erro de persistência/embedding | após N retries idempotentes | registra causa; alerta |
| T3 | `FAILED→INGESTED` | retry | backoff exponencial + jitter | reprocessa |
| T4 | `ACTIVE→CONSOLIDATING` | `consolidate()` (023) ou `consolidation_threshold` atingido | `ConsolidationVersion` gravada antes (INV-6) | emite `consolidation.started` |
| T5 | `CONSOLIDATING→CONSOLIDATED` | promoção/merge concluídos | integridade validada | emite `item.consolidated` |
| T6 | `CONSOLIDATING→ACTIVE` | falha/conflito de consolidação | rollback à versão anterior aplicado | emite `consolidation.rolledback` |
| T7 | `CONSOLIDATED→DECAYING` | tempo/uso reduzem `decay_score` | `decay_score < decay_threshold` | emite `item.decayed` |
| T8 | `ACTIVE→DECAYING` | job de decaimento | `decay_score < θ` **ou** `expires_at` vencido | emite `item.decayed` |
| T9 | `CONSOLIDATED→ARCHIVED` | frieza (baixo acesso) | move conteúdo p/ MinIO; mantém metadados | emite `item.archived` |
| T10 | `ARCHIVED→ACTIVE` | recall exige reidratação | reidrata de MinIO/PG | atualiza `last_access_at` |
| T11 | `DECAYING→FORGET_PENDING` | política de esquecimento elegível | `strategy` casa (`ttl`/`decay`/`lru`/`quota`/`rtbf`) | cria tombstone |
| T12 | `*→FORGET_PENDING` | `forget(policy)` **ou** RTBF (LGPD/GDPR) | não em `legal_hold` (exceto RTBF autorizado) | cria tombstone |
| T13 | `FORGET_PENDING→FORGOTTEN` | confirmação | tombstone criado | emite `item.forgotten` |
| T14 | `FORGOTTEN→PURGED` | fim do *grace period* (`memory.forget.grace_period`) | grace vencido | purge físico + remove blob/vetor; emite `item.purged` |

> `*` em T12 abrange `ACTIVE`, `CONSOLIDATED`, `DECAYING` e `ARCHIVED` como origens
> possíveis de `forget()`/RTBF.

### 2.4 Invariantes (normativas)

- **INV-A (legal hold):** item em `retention_class=legal_hold` **NÃO DEVE** transitar a
  `PURGED` salvo RTBF autorizado (T12/T14 sob RTBF).
- **INV-B (snapshot antes de mutar):** toda transição para `CONSOLIDATING` (T4) **DEVE**
  ter uma `ConsolidationVersion` gravada antes.
- **INV-C (linhagem no esquecimento):** `FORGOTTEN` **DEVE** preservar tombstone +
  `parent_ids` para auditoria até `PURGED`.
- **INV-D (dimensão de embedding):** ao entrar em `ACTIVE` com embedding,
  `dim(embedding)` **DEVE** == `memory.embedding.dim`, senão `AIOS-MEM-0021`.
- **INV-E (idempotência):** T3 (`FAILED→INGESTED`) **DEVE** ser idempotente por
  `Idempotency-Key` (RFC-0001 §5.5).

### 2.5 Matriz de estados-alvo permitidos

Legenda: `X` = transição permitida; vazio = proibida.

```
De \ Para    ING ACT CONSg CONSd ARCH DEC FPND FORG PURG FAIL
INGESTED      -   X                                        X
ACTIVE            -   X               X   X   (X)*
CONSOLIDATING     X   -    X
CONSOLIDATED          -    -    X     X       (X)*
ARCHIVED          X                   -       (X)*
DECAYING                                  -   X
FORGET_PEND                                       -   X
FORGOTTEN                                             -   X
PURGED                                                    -
FAILED        X                                              -
```

`(X)*` = transição via `forget()`/RTBF (T12) para `FORGET_PENDING`.

### 2.6 Mapeamento estado → evento NATS

| Estado alcançado | Evento emitido (§6.1 do brief) |
|------------------|-------------------------------|
| `ACTIVE` (via T1) | `aios.<t>.memory.item.stored` |
| `CONSOLIDATED` | `aios.<t>.memory.item.consolidated` |
| `DECAYING` | `aios.<t>.memory.item.decayed` |
| `ARCHIVED` | `aios.<t>.memory.item.archived` |
| `FORGOTTEN` | `aios.<t>.memory.item.forgotten` |
| `PURGED` | `aios.<t>.memory.item.purged` |

---

## 3. Máquina de estado do `ConsolidationJob`

### 3.1 Diagrama (ASCII)

```
  PENDING ──admissão──▶ SNAPSHOTTING ──ok──▶ RUNNING ──ok──▶ VALIDATING ──ok──▶ COMMITTED (terminal)
     │                       │ falha            │ falha           │ inconsistência
     │ rejeitado             ▼                  ▼                 ▼
     ▼                    FAILED ◀───────── ROLLING_BACK ◀───────┘
  REJECTED (terminal)                          │ ok
                                               ▼
                                          ROLLED_BACK (terminal)
```

### 3.2 Estados

| Estado | Tipo | Descrição / guarda | Ação de entrada |
|--------|------|--------------------|-----------------|
| `PENDING` | inicial | Enfileirado; aguarda janela/admissão do `RetentionScheduler`. | Adquire lock por `(tenant, agente)`. |
| `REJECTED` | **terminal** | Cota/política negam. | Emite motivo; libera lock (`AIOS-MEM-0010`). |
| `SNAPSHOTTING` | transitório | Grava `ConsolidationVersion` (pré-imagem) — obrigatório antes de mutar. | Persiste snapshot (MinIO/PG). |
| `RUNNING` | transitório | Promove/mescla/deduplica itens `from_layer`→`to_layer`. | Itens → `CONSOLIDATING`; emite `consolidation.started`. |
| `VALIDATING` | transitório | Verifica integridade/linhagem/Recall Rate pós-consolidação. | Compara Recall Rate (NFR-011). |
| `COMMITTED` | **terminal** | Versão marcada ativa; itens `CONSOLIDATED`. | Emite `item.consolidated` + `consolidation.completed`; libera lock. |
| `ROLLING_BACK` | transitório | Restaura versão anterior (falha ou regressão de Recall Rate). | Reverte itens → `ACTIVE`. |
| `ROLLED_BACK` | **terminal** | Estado revertido. | Emite `consolidation.rolledback`; libera lock. |
| `FAILED` | **terminal** | Erro irrecuperável após rollback seguro; para inspeção. | Emite `consolidation.failed`; alerta. |

### 3.3 Tabela de transições

| # | De → Para | Gatilho | Guarda | Ação / evento |
|---|-----------|---------|--------|---------------|
| J1 | `PENDING→SNAPSHOTTING` | admissão pelo scheduler | lock adquirido; cota/política permitem | inicia snapshot |
| J2 | `PENDING→REJECTED` | admissão negada | cota/política negam | `AIOS-MEM-0010`; libera lock |
| J3 | `SNAPSHOTTING→RUNNING` | snapshot gravado | `ConsolidationVersion` persistida (INV-B) | inicia mutação; emite `consolidation.started` |
| J4 | `SNAPSHOTTING→FAILED` | erro ao snapshotar | — | `consolidation.failed` |
| J5 | `RUNNING→VALIDATING` | mutações concluídas | promoção/merge sem erro | inicia validação |
| J6 | `RUNNING→ROLLING_BACK` | erro/conflito durante mutação | conflito de versão concorrente (`AIOS-MEM-0040`) | inicia reversão |
| J7 | `VALIDATING→COMMITTED` | validação ok | Recall Rate pós ≥ pré − 0,01 (NFR-011) | marca versão ativa; `item.consolidated` + `consolidation.completed` |
| J8 | `VALIDATING→ROLLING_BACK` | inconsistência/regressão | Recall Rate pós < pré − 0,01 | inicia reversão |
| J9 | `ROLLING_BACK→ROLLED_BACK` | reversão bem-sucedida | pré-imagem restaurada | `consolidation.rolledback` |
| J10 | `ROLLING_BACK→FAILED` | reversão falha | rollback impossível/parcial (`AIOS-MEM-0041`) | `consolidation.failed`; inspeção manual |

### 3.4 Invariantes (normativas)

- **INV-J1 (serialização):** jobs concorrentes para o mesmo `(tenant, agente)` **DEVEM**
  ser serializados por lock distribuído (Redis); jobs são idempotentes por `version_id`.
- **INV-J2 (snapshot obrigatório):** `RUNNING` só é alcançável após `SNAPSHOTTING`
  concluído (J3) — nenhuma mutação sem pré-imagem.
- **INV-J3 (sem regressão):** um job só chega a `COMMITTED` se a Recall Rate não regredir
  além de 0,01 (NFR-011); caso contrário, rollback automático (J8).
- **INV-J4 (terminalidade):** `COMMITTED`, `REJECTED`, `ROLLED_BACK` e `FAILED` são
  terminais e liberam o lock.

### 3.5 Correlação Job ↔ MemoryItem

| Estado do Job | Estado dos itens envolvidos | Evento de item |
|---------------|----------------------------|----------------|
| `RUNNING` | `ACTIVE → CONSOLIDATING` (T4) | — (job emite `consolidation.started`) |
| `COMMITTED` | `CONSOLIDATING → CONSOLIDATED` (T5) | `item.consolidated` |
| `ROLLING_BACK`/`ROLLED_BACK` | `CONSOLIDATING → ACTIVE` (T6) | — (job emite `consolidation.rolledback`) |
| `FAILED` | `CONSOLIDATING → ACTIVE` (T6, após rollback seguro) | — (job emite `consolidation.failed`) |

---

## 4. Exemplo de trajetória de vida (item semântico)

```
remember(fact, semantic)                → INGESTED
  embedding ok, cota ok                 → ACTIVE            [item.stored]
consolidate() (023, threshold=500)      → CONSOLIDATING     [consolidation.started]
  merge + validação ok                  → CONSOLIDATED      [item.consolidated]
30d sem acesso, decay_score<0,15        → DECAYING          [item.decayed]
ForgettingPolicy(strategy=decay)        → FORGET_PENDING
confirmação                             → FORGOTTEN         [item.forgotten]
grace_period=7d expira                  → PURGED (terminal) [item.purged]
```

Trajetória alternativa com falha de consolidação:

```
ACTIVE → CONSOLIDATING → (Recall Rate regride) → [job ROLLING_BACK] → ACTIVE
                                                  [consolidation.rolledback]
```

> Rastreabilidade completa em [`UseCases.md`](./UseCases.md) (UC-001, UC-003, UC-005) e
> fluxos em [`SequenceDiagrams.md`](./SequenceDiagrams.md) §6–§8. Modos de falha
> associados em [`FailureRecovery.md`](./FailureRecovery.md) (F3, F6).
