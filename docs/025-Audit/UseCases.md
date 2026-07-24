---
Documento: UseCases
Módulo: 025-Audit
Status: Draft
Versão: 0.1
Última atualização: 2026-07-23
Responsável (RACI-A): Arquiteto do Módulo 025-Audit
ADRs relacionados: ADR-0010, ADR-0251, ADR-0253, ADR-0254, ADR-0255, ADR-0257, ADR-0259
RFCs relacionados: RFC-0001, RFC-0250, RFC-0251
Depende de: FunctionalRequirements.md, StateMachine.md, API.md, Events.md
---

# 025-Audit — Casos de Uso

> Formato: ator · pré-condições · fluxo principal · fluxos alternativos · exceções ·
> pós-condições. Os IDs `UC-NNN` são referenciados por `./Requirements.md` §5 e §6 e
> por `./Testing.md`.

## UC-001 — Registrar um fato por evento (caminho principal)

| Campo | Conteúdo |
|-------|----------|
| **Ator primário** | Qualquer módulo produtor, via JetStream (`../022-Policy/`, `../021-Security/`, `../006-Kernel/`, ...) |
| **Pré-condições** | `record_class` registrada em `audit.auditable_class`; subject listado em `source_subjects` |
| **Requisitos** | FR-001, FR-003, FR-004, FR-019; NFR-002, NFR-009 |

**Fluxo principal**

1. O produtor publica o evento no barramento via seu **outbox transacional**.
2. O consumidor do 025 recebe e o `AuditIngestGateway` verifica o `event_id` contra a
   tabela de dedupe.
3. O `RecordNormalizer` monta o envelope canônico: `actor`, `action`, `resource_urn`,
   `outcome`, `decision_ref`, `trace_id`, `subject_ref`.
4. O `PayloadCipher` cifra o detalhe com a chave do titular (quando a classe tem
   `contains_pii`) e calcula `payload_digest` sobre o texto em claro.
5. O `ChainAppender` obtém `seq` da partição, calcula `record_hash` sobre o conteúdo
   canônico concatenado ao `prev_hash`, e insere.
6. O consumidor confirma (`ACK`) ao JetStream **somente após o commit durável**.
7. O registro fica em `Received` até a próxima selagem (UC-003).

**Fluxos alternativos**

- **A1 — Reentrega do JetStream:** o mesmo `event_id` chega de novo; nenhum registro é
  criado (T-07); o contador de duplicatas é incrementado e o `ACK` é devolvido.
- **A2 — Classe com `contains_pii = false`:** o payload é cifrado com a chave do
  **tenant**, não do titular; não há apagamento criptográfico individual.

**Exceções**

| # | Condição | Resultado |
|---|----------|-----------|
| E1 | `record_class` não registrada | Registro **não** é criado; `AIOS-AUD-0002`; evento vai para DLQ e o dono é notificado |
| E2 | Banco indisponível | **Sem `ACK`**: o JetStream reentrega; nenhum fato é perdido (NFR-006) |
| E3 | `prev_hash` divergente do último `seq` | Escrita abortada; `chain_break` — anomalia crítica (UC-006) |
| E4 | Envelope malformado | `AIOS-AUD-0004`; DLQ; **a rejeição é ela mesma registrada** |

**Pós-condições:** fato encadeado e durável; `seq` sem lacuna; nenhuma alteração
possível a partir daqui (invariante I6).

---

## UC-002 — Registrar um fato por API direta

| Campo | Conteúdo |
|-------|----------|
| **Ator primário** | Produtor fora do barramento (cerimônia manual, sistema externo, comprovante de expurgo de `005`/`010`/`024`) |
| **Pré-condições** | mTLS válido; capability `aud:record:append`; `Idempotency-Key` presente |
| **Requisitos** | FR-001, FR-002; NFR-001, NFR-006, NFR-014 |

**Fluxo principal**

1. O produtor chama `POST /v1/audit/records` com `Idempotency-Key`.
2. O gateway valida mTLS, tenant e envelope.
3. O registro percorre normalização → cifragem → encadeamento (UC-001, passos 3–5).
4. A resposta **só é devolvida após durabilidade confirmada** (invariante I9), com o
   `id`, o `seq` e o `record_hash`.

**Fluxos alternativos**

- **A1 — Lote:** `POST /v1/audit/records:batch` com até `aud.ingest.batch_max_size`
  (500) registros; todos entram na mesma transação por partição.
- **A2 — Repetição:** a mesma `Idempotency-Key` devolve o registro já gravado, com o
  mesmo `record_hash` (NFR-014).

**Exceções**

| # | Condição | Resultado |
|---|----------|-----------|
| E1 | Durabilidade não confirmada | **`503 AIOS-AUD-0005`, `retriable=true`** — o produtor reenvia do seu outbox |
| E2 | Tenant divergente do certificado | `403 AIOS-AUD-0003` |
| E3 | Lote acima do limite | `422 AIOS-AUD-0015` |
| E4 | Tentativa de alterar registro existente | `409 AIOS-AUD-0016` |

**Pós-condições:** ou o fato está durável e encadeado, ou o produtor sabe que precisa
reenviar. **Não existe estado intermediário silencioso** — que é justamente o defeito
que este desenho impede.

---

## UC-003 — Selar um intervalo

| Campo | Conteúdo |
|-------|----------|
| **Ator primário** | `SealService` (automático, a cada `aud.seal.interval_s`) |
| **Atores secundários** | `../021-Security/` (chave de assinatura) |
| **Pré-condições** | Existem registros em `Received` na partição; chave de assinatura disponível |
| **Requisitos** | FR-005; NFR-004 |

**Fluxo principal**

1. O `SealService` seleciona o intervalo `[seq_from, seq_to]` da partição.
2. Constrói a **árvore de Merkle** sobre os `record_hash` do intervalo.
3. Calcula `seal_hash` sobre (raiz, intervalo, `prev_seal_hash`, horário).
4. Solicita a **assinatura** ao `../021-Security/` com a chave `signing` corrente.
5. Persiste o selo e atualiza os registros para `Sealed` (T-02) na mesma transação.
6. Emite `audit.record.sealed`; o `WormArchiver` agenda o arquivamento (UC-015).

**Fluxos alternativos**

- **A1 — Âncora externa:** com `aud.seal.external_anchor_enabled`, o `seal_hash` é
  também registrado em serviço de terceiro, gravando `external_anchor`.

**Exceções**

| # | Condição | Resultado |
|---|----------|-----------|
| E1 | Chave de assinatura indisponível | Selagem adia; **a ingestão continua**; alerta se exceder o intervalo |
| E2 | Registros em `Received` além de `aud.seal.interval_s` | `seal_overdue` (`./Monitoring.md` RB-A-04) |
| E3 | Rotação de chave durante a selagem | Selo assinado com a chave nova; `signed_by` registra qual foi |

**Pós-condições:** o intervalo é **imutável de forma verificável**; a prova de inclusão
de qualquer registro do intervalo pode ser gerada (UC-013).

---

## UC-004 — Verificar integridade

| Campo | Conteúdo |
|-------|----------|
| **Ator primário** | `IntegrityVerifier` (contínuo) ou auditor (sob demanda) |
| **Pré-condições** | Selos existentes; capability `aud:integrity:verify` para verificação sob demanda |
| **Requisitos** | FR-006; NFR-007, NFR-008 |

**Fluxo principal (contínuo, por amostragem)**

1. A cada `aud.verify.interval_s` (900 s), sorteia `aud.verify.sample_size` (1000)
   registros selados.
2. Recomputa `record_hash` de cada um e confere o encadeamento com o anterior.
3. Confere a prova de Merkle contra a raiz do selo e a assinatura do selo.
4. Registra o resultado; qualquer divergência dispara UC-006.

**Fluxo alternativo (completo, sob demanda)**

- **A1:** `POST /v1/audit/verify` para uma partição/janela executa os 6 passos de
  `./StateMachine.md` §7, incluindo a **comparação com a cópia WORM**. Executa em
  réplica de leitura para não competir com a ingestão.

**Exceções**

| # | Condição | Resultado |
|---|----------|-----------|
| E1 | Divergência de hash | `AIOS-AUD-0011`; anomalia `chain_break` crítica (UC-006) |
| E2 | Selo ausente para o intervalo | `AIOS-AUD-0014` |
| E3 | Verificação excede o orçamento de tempo | Resultado parcial marcado como tal; **nunca** reportado como "íntegro" |

**Pós-condições:** a integridade da amostra ou da janela está provada — ou o incidente
está aberto.

---

## UC-005 — Detectar lacuna ou classe silenciosa

| Campo | Conteúdo |
|-------|----------|
| **Ator primário** | `CompletenessMonitor` (automático) |
| **Pré-condições** | Classes registradas com `expected_rate_per_day` |
| **Requisitos** | FR-007; NFR-005 |

**Fluxo principal**

1. O monitor verifica continuamente a continuidade de `seq` por partição.
2. Compara o volume observado por `record_class` com `expected_rate_per_day`,
   aplicando `aud.completeness.silence_ratio` (0,2).
3. Lacuna de `seq` → `audit.anomaly.detected{kind=sequence_gap}` em ≤
   `aud.completeness.gap_alert_s` (300 s).
4. Volume abaixo do limiar → `audit.anomaly.detected{kind=class_silent}`, apontando o
   `producer_module` esperado.

**Exceções**

| # | Condição | Resultado |
|---|----------|-----------|
| E1 | Lacuna por reinício de partição | Investigação confirma; conclusão registrada em `audit.anomaly.resolution` |
| E2 | Classe legitimamente sem atividade (ex.: fim de semana) | Ajuste do `expected_rate_per_day` — **nunca** desligar a expectativa |
| E3 | Produtor descomissionado | Classe marcada `deprecated`; expectativa zerada com registro do motivo |

**Pós-condições:** a ausência é visível. Sem esta verificação, um produtor que parou de
emitir seria indistinguível de um sistema sem atividade.

---

## UC-006 — Detectar quebra de cadeia (incidente)

| Campo | Conteúdo |
|-------|----------|
| **Ator primário** | `IntegrityVerifier` → `AnomalyDetector` |
| **Atores secundários** | CISO, `../029-Operations/` |
| **Pré-condições** | Verificação em execução |
| **Requisitos** | FR-006, FR-018; NFR-007 |

**Fluxo principal**

1. A verificação encontra `record_hash` recomputado ≠ armazenado, ou `prev_hash`
   inconsistente.
2. `audit.anomaly.detected{kind=chain_break, severity=critical}` é emitido
   imediatamente.
3. A escrita **na partição afetada** é congelada para preservar o estado.
4. Compara-se o registro com a **cópia WORM** e com a raiz de Merkle do **selo
   assinado** — duas fontes independentes do banco.
5. O resultado da comparação determina o escopo: alteração pós-selagem (detectável e
   provável adulteração) ou defeito de escrita anterior à selagem.
6. Acionamento do CISO conforme `./Monitoring.md` RB-A-01.

**Exceções**

| # | Condição | Resultado |
|---|----------|-----------|
| E1 | WORM e selo concordam entre si e divergem do banco | **Adulteração no banco**: a evidência íntegra é a do WORM |
| E2 | WORM ausente (intervalo ainda não arquivado) | Selo assinado ainda decide; janela de risco documentada |
| E3 | Divergência por bug de serialização canônica | Corrigido em código; registros reprocessados por **compensação**, nunca por alteração |

**Pós-condições:** a divergência está caracterizada e a evidência preservada. A trilha
não "se conserta" — a conclusão é registrada como fato novo.

---

## UC-007 — Registrar uma classe auditável

| Campo | Conteúdo |
|-------|----------|
| **Ator primário** | Arquiteto de plataforma / módulo dono |
| **Pré-condições** | Capability `aud:class:manage` |
| **Requisitos** | FR-008, FR-022 |

**Fluxo principal**

1. `PUT /v1/audit/classes/{key}` com `producer_module`, `source_subjects`,
   `retention_days`, `legal_basis`, `contains_pii` e `expected_rate_per_day`.
2. O `AuditableRegistry` valida a retenção contra `aud.retention.min_days` (365) e
   exige `legal_basis` quando `contains_pii`.
3. O consumidor passa a assinar os `source_subjects` declarados.

**Exceções**

| # | Condição | Resultado |
|---|----------|-----------|
| E1 | `contains_pii = true` sem `legal_basis` | `422 AIOS-AUD-0010` |
| E2 | `retention_days` abaixo do piso | `422 AIOS-AUD-0010` com o piso no `detail` |
| E3 | Classe já existe com produtor diferente | `409 AIOS-AUD-0006`; alteração exige revisão |

**Pós-condições:** o fato passa a ser auditável **e** a sua ausência passa a ser
detectável — as duas coisas ao mesmo tempo.

---

## UC-008 — Aplicar *legal hold*

| Campo | Conteúdo |
|-------|----------|
| **Ator primário** | Jurídico / DPO do tenant |
| **Atores secundários** | Aprovador distinto; `../005-Database/` |
| **Pré-condições** | Capability `aud:hold:apply`; `case_ref` e `legal_basis` declarados |
| **Requisitos** | FR-009, FR-010, FR-021; NFR-010 |

**Fluxo principal**

1. O solicitante submete `ApplyLegalHold` com escopo (classe, titular, janela,
   recurso), `case_ref` e `legal_basis`.
2. O `LegalHoldManager` exige `approved_by ≠ requested_by`.
3. Os registros no escopo transitam para `Held` (T-03).
4. Em **uma transação**: *hold* gravado + `audit.legalhold.applied` no outbox.
5. O `../005-Database/` consome o evento e **suspende o expurgo** das tabelas afetadas.
6. Tentativas de expurgo ou apagamento no escopo passam a falhar com
   `AIOS-AUD-0008`.

**Fluxos alternativos**

- **A1 — *Hold* sem prazo:** `expires_at` nulo é permitido (`./StateMachine.md` §6.1);
  entra na revisão obrigatória a cada `aud.hold.review_interval_days` (90).

**Exceções**

| # | Condição | Resultado |
|---|----------|-----------|
| E1 | Solicitante = aprovador | `403 AIOS-AUD-0009` |
| E2 | `legal_basis` ausente | `422 AIOS-AUD-0004` |
| E3 | Pedido de RTBF sobre escopo em *hold* | Recusado com `423 AIOS-AUD-0008`; **a recusa é registrada** e o titular é informado |

**Pós-condições:** nada no escopo é apagado ou expurgado enquanto o *hold* durar — nem
por retenção, nem por direito ao esquecimento.

---

## UC-009 — Liberar *legal hold*

| Campo | Conteúdo |
|-------|----------|
| **Ator primário** | Jurídico / DPO, após encerramento do caso |
| **Pré-condições** | Capability `aud:hold:release` |
| **Requisitos** | FR-009 |

**Fluxo principal**

1. `ReleaseLegalHold` com o motivo do encerramento.
2. Registros voltam a `Sealed` (T-04), **desde que nenhum outro *hold*** os cubra.
3. Emite `audit.legalhold.released`; `../005-Database/` retoma o expurgo.
4. Pedidos de RTBF pendentes sobre o escopo são reavaliados.

**Exceções**

| # | Condição | Resultado |
|---|----------|-----------|
| E1 | Outro *hold* ainda cobre o registro | Permanece em `Held`; a liberação afeta só o *hold* indicado |
| E2 | Sem capability | Negado pelo PDP; a negação é auditada |

**Pós-condições:** retenção e apagamento voltam a operar normalmente, com o histórico
de quem aplicou e quem liberou preservado.

---

## UC-010 — Executar apagamento criptográfico (RTBF)

| Campo | Conteúdo |
|-------|----------|
| **Ator primário** | `CryptoShredder`, disparado por `security.rtbf.requested` |
| **Atores secundários** | `../021-Security/` (destruição de chave), DPO |
| **Pré-condições** | `subject_ref` identificado; nenhum *legal hold* no escopo |
| **Requisitos** | FR-011, FR-012, FR-013; NFR-011 |

**Fluxo principal**

1. O módulo consome `aios.<tenant>.security.rtbf.requested` com o titular.
2. Verifica `audit.legal_hold`: havendo *hold*, o pedido é **recusado e registrado**
   (UC-008 E3).
3. Não havendo, solicita ao `../021-Security/` a **destruição da chave** do titular.
4. `audit.subject_key.state` → `destroyed`; os registros do titular → `Shredded`
   (T-05).
5. O `ErasureReceiptService` emite comprovante com `receipt_hash` e o **registra na
   própria trilha** (`chain_record_id`).
6. Emite `audit.erasure.completed` em ≤ `aud.erasure.max_latency_h` (24 h).

**Fluxos alternativos**

- **A1 — Titular sem registros:** comprovante com `records_affected = 0` — resultado
  válido e igualmente registrado.

**Exceções**

| # | Condição | Resultado |
|---|----------|-----------|
| E1 | *Legal hold* vigente | `423 AIOS-AUD-0008`; titular informado da obrigação legal de preservação |
| E2 | Payload ainda decifrável após a destruição | **P1 de privacidade**: reexecutar; investigar cópia de chave |
| E3 | Consulta posterior ao payload apagado | `409 AIOS-AUD-0012` — indisponível **por desenho** |

**Pós-condições:** o conteúdo pessoal é ilegível; `seq`, `record_hash` e
`payload_digest` permanecem, e a **cadeia continua verificável** (FR-012). Prova-se que
algo aconteceu, sem revelar o quê.

---

## UC-011 — Expurgar payload por fim de retenção

| Campo | Conteúdo |
|-------|----------|
| **Ator primário** | `RetentionEnforcer` (automático) |
| **Pré-condições** | `recorded_at + retention_days` vencido; intervalo arquivado em WORM |
| **Requisitos** | FR-008, FR-012; NFR-010 |

**Fluxo principal**

1. O job identifica registros fora da retenção da sua `record_class`.
2. Verifica *legal hold*: havendo, **pula** o registro (invariante I5).
3. Confirma que o intervalo já está arquivado em WORM.
4. Remove `payload_cipher`; mantém `seq`, hashes, `payload_digest`, `occurred_at`,
   `record_class` e `outcome`. Estado → `Expired` (T-06).
5. Emite `audit.retention.expired`.

**Exceções**

| # | Condição | Resultado |
|---|----------|-----------|
| E1 | *Hold* vigente | Registro preservado; contabilizado no relatório de retenção |
| E2 | Intervalo não arquivado | Expurgo adiado até o arquivamento (a cópia WORM é a última linha de defesa) |
| E3 | Tentativa de reduzir `aud.retention.min_days` em runtime | Ignorada: chave **não-recarregável** |

**Pós-condições:** o custo de armazenamento decresce sem que a **prova de existência**
seja perdida — a cadeia permanece completa e verificável.

---

## UC-012 — Consultar a trilha

| Campo | Conteúdo |
|-------|----------|
| **Ator primário** | Investigador, auditor interno, DPO |
| **Atores secundários** | `../022-Policy/` (PDP) |
| **Pré-condições** | Capability `aud:record:read`; tenant coerente |
| **Requisitos** | FR-015, FR-016, FR-020, FR-023; NFR-012, NFR-015, NFR-016 |

**Fluxo principal**

1. O ator consulta por `subject_ref`, `decision_ref`, `trace_id`, `record_class` ou
   janela temporal.
2. O `QueryService` monta o `DecisionRequest` e consulta o PDP.
3. Autorizado, aplica escopo por tenant e os orçamentos
   (`aud.query.max_duration_s`, `aud.query.max_records`).
4. Decifra o payload dos registros que o solicitante pode ver.
5. **Registra a própria consulta** como fato auditável (classe `audit.access`), com
   solicitante, escopo e volume retornado.

**Exceções**

| # | Condição | Resultado |
|---|----------|-----------|
| E1 | Consulta cross-tenant | `403 AIOS-AUD-0003` |
| E2 | Orçamento excedido | `429 AIOS-AUD-0013` |
| E3 | Registro `Shredded`/`Expired` | Metadados retornados; payload `null` com `AIOS-AUD-0012` no detalhe do item |
| E4 | PDP indisponível | Consulta **negada** (`fail_mode = closed`); a ingestão continua |

**Pós-condições:** o investigador tem a evidência; e o fato de tê-la consultado também
está na trilha — que é exatamente o que o auditor do auditor precisa.

---

## UC-013 — Obter prova de inclusão

| Campo | Conteúdo |
|-------|----------|
| **Ator primário** | Auditor (interno ou externo) |
| **Pré-condições** | Registro selado; capability `aud:record:read` |
| **Requisitos** | FR-005, FR-006 |

**Fluxo principal**

1. `GET /v1/audit/records/{id}:proof`.
2. O serviço devolve: `record_hash`, o **caminho de Merkle** até a raiz, o selo
   (`merkle_root`, `seal_hash`, assinatura, `signed_by`) e a referência da chave
   pública.
3. O auditor recomputa a raiz a partir do `record_hash` e do caminho, e verifica a
   assinatura **com a chave pública** — sem consultar o AIOS de novo.

**Exceções**

| # | Condição | Resultado |
|---|----------|-----------|
| E1 | Registro ainda em `Received` | `412 AIOS-AUD-0014`: sem selo, não há prova de inclusão — apenas o encadeamento |
| E2 | Chave de assinatura rotacionada | O selo indica `signed_by`; a chave pública histórica continua publicada pelo `../021-Security/` |

**Pós-condições:** o auditor tem uma prova **matemática** de que aquele registro estava
na trilha no momento da selagem, verificável de forma independente.

---

## UC-014 — Exportar pacote probatório

| Campo | Conteúdo |
|-------|----------|
| **Ator primário** | Auditor externo (via responsável interno) |
| **Atores secundários** | Aprovador distinto do solicitante |
| **Pré-condições** | Capabilities `aud:export:request` e `aud:export:download` |
| **Requisitos** | FR-017, FR-020; NFR-013, NFR-015 |

**Fluxo principal**

1. `RequestExport` com escopo (classe, janela, titular, caso) e justificativa.
2. Aprovação dupla (`approved_by ≠ requested_by`); o escopo é revisado.
3. O `ExportService` monta o pacote: registros, selos do período, provas de Merkle,
   chaves públicas de verificação e **instruções de verificação offline**.
4. Calcula `package_hash`, grava o objeto e emite `audit.export.completed`.
5. O auditor baixa dentro de `aud.export.package_ttl_h` (72 h) e verifica **sem acesso
   ao AIOS**.

**Exceções**

| # | Condição | Resultado |
|---|----------|-----------|
| E1 | Solicitante = aprovador | `403 AIOS-AUD-0009` |
| E2 | Escopo acima de `aud.export.max_records` | `422 AIOS-AUD-0015` |
| E3 | Escopo vazio | `422 AIOS-AUD-0015` |
| E4 | Registros `Shredded` no escopo | Incluídos com metadados e `payload_digest`; a ausência do conteúdo é **explicada no pacote** |
| E5 | Link expirado | `404 AIOS-AUD-0001`; nova solicitação necessária |

**Pós-condições:** o auditor pode provar a integridade da trilha sem confiar no
operador do AIOS. A exportação, por sua vez, também está na trilha.

---

## UC-015 — Arquivar em WORM

| Campo | Conteúdo |
|-------|----------|
| **Ator primário** | `WormArchiver` (automático) |
| **Pré-condições** | Intervalo selado; passou `aud.worm.archive_after_s` (3600 s) |
| **Requisitos** | FR-014; NFR-007 |

**Fluxo principal**

1. O arquivador serializa o intervalo selado (registros + selo) em formato canônico.
2. Grava o objeto no MinIO com ***object lock*** por `aud.worm.object_lock_days`
   (1825).
3. Registra `worm_object` no selo correspondente.
4. A cópia passa a ser a **terceira fonte independente** para a verificação
   (`./StateMachine.md` §7).

**Exceções**

| # | Condição | Resultado |
|---|----------|-----------|
| E1 | MinIO indisponível | Arquivamento adia; cadeia e selos seguem; alerta se exceder o prazo |
| E2 | Tentativa de sobrescrever objeto arquivado | **Rejeitada pelo armazenamento** (*object lock*) — é o comportamento desejado |
| E3 | Divergência entre banco e objeto na verificação | `chain_break` (UC-006); o objeto WORM é a evidência íntegra |

**Pós-condições:** existe uma cópia da evidência fora do alcance de qualquer operação
de banco — a camada que torna a adulteração impraticável mesmo com acesso total ao
PostgreSQL.

---

## Referências

- Fonte única de verdade: `./_DESIGN_BRIEF.md`
- Fluxos detalhados: `./SequenceDiagrams.md` · FSM: `./StateMachine.md`
- Contratos: `./API.md` · `./Events.md`
- Rastreabilidade: `./Requirements.md` §5 e §6

*Fim de `UseCases.md`.*
