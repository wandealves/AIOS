---
Documento: FunctionalRequirements
Módulo: 025-Audit
Status: Draft
Versão: 0.1
Última atualização: 2026-07-23
Responsável (RACI-A): Arquiteto do Módulo 025-Audit
ADRs relacionados: ADR-0010, ADR-0251, ADR-0252, ADR-0253, ADR-0254, ADR-0255, ADR-0256, ADR-0257, ADR-0258, ADR-0259
RFCs relacionados: RFC-0001, RFC-0250, RFC-0251
Depende de: _DESIGN_BRIEF.md §7.1, 021-Security, 022-Policy, 005-Database
---

# 025-Audit — Requisitos Funcionais

> Todos os requisitos abaixo derivam de `./_DESIGN_BRIEF.md` §7.1 e **NÃO PODEM**
> contradizê-lo. Prioridade em MoSCoW (Must / Should / Could). Cada FR é verificável
> por ao menos um caso de teste em `./Testing.md` e exercitado por ao menos um caso de
> uso em `./UseCases.md`.

## 1. Convenções

- Palavras normativas conforme RFC 2119 / RFC 8174.
- Códigos de erro seguem `AIOS-AUD-<NNNN>` (`./API.md` §6), formato da
  `../003-RFC/RFC-0001-Architecture-Baseline.md` §5.4.
- Estados e transições referem-se à FSM de `./StateMachine.md`.

## 2. Tabela de requisitos funcionais

| ID | Requisito | Prioridade | Critério de aceite | Origem | UC | Teste |
|----|-----------|-----------|--------------------|--------|----|-------|
| FR-001 | O módulo **DEVE** ingerir registros por evento (JetStream) e por API direta, com dedupe por `event_id`. | Must | Reprocessar 10⁴ eventos não cria duplicata; `seq` permanece sem lacuna. | Brief §7.1 | UC-001, UC-002 | T-AUD-001 |
| FR-002 | O `AuditIngestGateway` **DEVE** confirmar a escrita **somente após durabilidade**; do contrário, `AIOS-AUD-0005`. | Must | Teste de crash: nenhum registro confirmado é perdido (RPO 0). | Brief §7.1; ADR-0255 | UC-002 | T-AUD-002 |
| FR-003 | O `RecordNormalizer` **DEVE** normalizar todo fato ao envelope canônico de auditoria. | Must | 100% dos registros com `actor`, `action`, `resource_urn`, `outcome`, `trace_id`. | Brief §7.1 | UC-001 | T-AUD-003 |
| FR-004 | O `ChainAppender` **DEVE** encadear por hash: `prev_hash` = `record_hash` do anterior na partição. | Must | Verificação de cadeia passa em 100% dos registros (invariante I2). | Brief §7.1; ADR-0251 | UC-001 | T-AUD-004 |
| FR-005 | O `SealService` **DEVE** selar intervalos periodicamente com raiz de Merkle **assinada**. | Must | Selo em ≤ `aud.seal.interval_s`; assinatura verificável com a chave pública do `../021-Security/`. | Brief §7.1; ADR-0251 | UC-003 | T-AUD-005 |
| FR-006 | O `IntegrityVerifier` **DEVE** verificar integridade continuamente (amostragem) e sob demanda (completa). | Must | Adulteração injetada é detectada em ≤ 1 ciclo de verificação. | Brief §7.1 | UC-004, UC-006 | T-AUD-006 |
| FR-007 | O `CompletenessMonitor` **DEVE** detectar lacuna de sequência e classe silenciosa. | Must | `seq` faltante → `audit.anomaly.detected{kind=sequence_gap}` em ≤ 5 min. | Brief §7.1; ADR-0256 | UC-005 | T-AUD-007 |
| FR-008 | O módulo **DEVE** aplicar retenção legal por classe, com base legal declarada. | Must | Classe `contains_pii` sem `legal_basis` → `AIOS-AUD-0010`. | Brief §7.1 | UC-007, UC-011 | T-AUD-008 |
| FR-009 | O `LegalHoldManager` **DEVE** aplicar e liberar *hold* com aprovação dupla e `case_ref`. | Must | `approved_by = requested_by` → `AIOS-AUD-0009`. | Brief §7.1; ADR-0254 | UC-008, UC-009 | T-AUD-009 |
| FR-010 | O *legal hold* **DEVE** suspender retenção e apagamento no escopo. | Must | Expurgo ou apagamento sob *hold* → `AIOS-AUD-0008`; nenhum payload removido. | Brief §7.1; ADR-0254 | UC-008 | T-AUD-010 |
| FR-011 | O `CryptoShredder` **DEVE** executar apagamento criptográfico destruindo a chave do titular. | Must | Payload torna-se indecifrável; `record_hash` e `payload_digest` inalterados. | Brief §7.1; ADR-0253 | UC-010 | T-AUD-011 |
| FR-012 | O módulo **DEVE** preservar cadeia e metadados em `Shredded` e `Expired`. | Must | Verificação de cadeia continua passando após apagamento (invariante I4). | Brief §7.1 | UC-010, UC-011 | T-AUD-012 |
| FR-013 | Todo apagamento **DEVE** gerar comprovante verificável, ele mesmo registrado na trilha. | Must | `receipt_hash` reproduzível; `chain_record_id` aponta para registro existente. | Brief §7.1 | UC-010 | T-AUD-013 |
| FR-014 | O `WormArchiver` **DEVE** arquivar intervalos selados em objeto com *object lock*. | Must | Tentativa de sobrescrita do objeto é rejeitada pelo armazenamento. | Brief §7.1; ADR-0251 | UC-015 | T-AUD-014 |
| FR-015 | O `QueryService` **DEVE** servir consulta com PEP e escopo por tenant. | Must | Consulta cross-tenant → `AIOS-AUD-0003`. | Brief §7.1 | UC-012 | T-AUD-015 |
| FR-016 | O módulo **DEVE** registrar **toda consulta à trilha** como fato auditável. | Must | Consulta gera registro de classe `audit.access` com solicitante e escopo. | Brief §7.1; ADR-0257 | UC-012 | T-AUD-016 |
| FR-017 | O `ExportService` **DEVE** exportar pacote probatório com registros, selos, provas e instruções de verificação. | Must | Auditor externo verifica o pacote **sem** acesso ao AIOS. | Brief §7.1; ADR-0259 | UC-014 | T-AUD-017 |
| FR-018 | O `AnomalyDetector` **DEVE** detectar e sinalizar anomalias de auditoria. | Must | Quebra de cadeia → `audit.anomaly.detected{kind=chain_break, severity=critical}`. | Brief §7.1 | UC-006 | T-AUD-018 |
| FR-019 | O módulo **NÃO DEVE** permitir alteração de registro (append-only). | Must | Tentativa de alteração → `AIOS-AUD-0016`; correção só por registro de compensação. | Brief §7.1; ADR-0258 | UC-001 | T-AUD-019 |
| FR-020 | O módulo **DEVE** atuar como PEP nas operações administrativas e de consulta. | Must | Sem capability → negado pelo PDP; **a própria negação é auditada**. | Brief §7.1 | UC-012, UC-014 | T-AUD-020 |
| FR-021 | O `EventEmitter` **DEVE** emitir todos os eventos de `./Events.md` via Outbox transacional. | Must | Nenhum evento perdido em teste de crash entre commit e publicação. | Brief §7.1 | UC-008 | T-AUD-021 |
| FR-022 | O `AuditableRegistry` **DEVERIA** registrar classes com produtor e volume esperados. | Should | Classe não registrada na ingestão → `AIOS-AUD-0002`. | Brief §7.1; ADR-0256 | UC-007 | T-AUD-022 |
| FR-023 | O módulo **NÃO DEVE** expor conteúdo auditado em evento, métrica ou log. | Must | Varredura automatizada não encontra payload fora da trilha. | Brief §7.1 | UC-012 | T-AUD-023 |

## 3. Requisitos por agrupamento funcional

| Grupo | FRs | Documento de detalhamento |
|-------|-----|---------------------------|
| Ingestão e durabilidade | FR-001..FR-003, FR-019 | `./API.md` §2, `./SequenceDiagrams.md` |
| Integridade verificável | FR-004..FR-006, FR-014, FR-018 | `./StateMachine.md` §7, `./Database.md` |
| Completude | FR-007, FR-022 | `./Monitoring.md` |
| Retenção, *hold* e apagamento | FR-008..FR-013 | `./Security.md`, `./Database.md` §11 |
| Consulta e prova | FR-015..FR-017, FR-020, FR-023 | `./API.md` §3 e §4 |
| Integração assíncrona | FR-021 | `./Events.md` |

## 4. Exemplo de verificação (FR-002 + FR-019)

Escrita com o banco indisponível — a resposta correta é **recusar**:

```http
POST /v1/audit/records HTTP/1.1
X-AIOS-Tenant: acme
Idempotency-Key: 01J9ZM00000000000000000000
Content-Type: application/json

{ "recordClass": "policy.decision",
  "occurredAt": "2026-07-23T10:15:00.123Z",
  "actor": { "urn": "urn:aios:acme:agent:01J9Z8Q6H7K2M4N6P8R0S2T4V6", "kind": "agent" },
  "action": "memory:item:read",
  "resourceUrn": "urn:aios:acme:memory:01J9ZA00000000000000000000",
  "outcome": "denied",
  "decisionRef": "01J9ZB00000000000000000000",
  "payload": { "reasonCode": "explicit_deny" } }
```

```json
HTTP/1.1 503 Service Unavailable
Content-Type: application/problem+json

{ "code": "AIOS-AUD-0005",
  "detail": "Escrita não pôde ser confirmada como durável; reenvie.",
  "retriable": true, "retryAfterMs": 2000 }
```

O produtor mantém o fato no seu outbox e reenvia — o mesmo `Idempotency-Key` garante
efeito único. **Aceitar** aqui trocaria uma indisponibilidade visível por uma lacuna
invisível, que é exatamente o defeito que este módulo existe para impedir.

Tentativa de corrigir um registro já gravado:

```json
HTTP/1.1 409 Conflict
{ "code": "AIOS-AUD-0016",
  "detail": "Registro é imutável. Emita um registro de compensação referenciando o original.",
  "retriable": false }
```

## 5. Referências

- Fonte: `./_DESIGN_BRIEF.md` §7.1
- Rastreabilidade: `./Requirements.md`
- Não-funcionais: `./NonFunctionalRequirements.md`
- Casos de uso: `./UseCases.md` · Testes: `./Testing.md`

*Fim de `FunctionalRequirements.md`.*
