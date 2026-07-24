---
Documento: API
Módulo: 025-Audit
Status: Draft
Versão: 0.1
Última atualização: 2026-07-23
Responsável (RACI-A): Arquiteto do Módulo 025-Audit
ADRs relacionados: ADR-0010, ADR-0250, ADR-0251, ADR-0253, ADR-0255, ADR-0257, ADR-0259
RFCs relacionados: RFC-0001, RFC-0250, RFC-0251
Depende de: _DESIGN_BRIEF.md §5, 004-API (catálogo de erros e registros), 021-Security, 022-Policy
---

# 025-Audit — Contratos de API

> Autenticação, envelope de erro, idempotência, correlação e versionamento seguem
> `../003-RFC/RFC-0001-Architecture-Baseline.md` §5.4–§5.7 e §6 — **referenciados, não
> redefinidos**. Pacote gRPC: `aios.audit.v1`. Base REST: `/v1/audit`.

## 1. Superfícies e volume esperado

| Superfície | Uso | Volume | Regime de falha |
|-----------|-----|--------|-----------------|
| **Eventos (JetStream)** | Caminho principal: fatos produzidos pelos módulos. | ~99% | Sem `ACK` até durabilidade; JetStream reentrega |
| **API de ingestão** | Fatos que não são eventos de barramento; comprovantes de expurgo de `005`/`010`/`024`. | ~1% | ***fail-closed***: `503 AIOS-AUD-0005` |
| **API de consulta/prova** | Investigação, auditoria, prova de inclusão. | baixo | ***fail-closed*** (PEP) |
| **API de governança** | *Hold*, retenção, apagamento, exportação. | muito baixo | ***fail-closed*** (PEP + aprovação dupla) |

Este módulo é **fail-closed nas duas pontas** — ao contrário do
`../024-Observability/`, que é *fail-open* na ingestão. A razão está em §2.2.

## 2. Ingestão

### 2.1 Operações

| Operação | REST | gRPC | Idempotente | Capability |
|----------|------|------|-------------|------------|
| Registrar fato | `POST /v1/audit/records` | `AppendRecord` | sim (`Idempotency-Key`) | `aud:record:append` |
| Registrar em lote | `POST /v1/audit/records:batch` | `AppendBatch` | sim | `aud:record:append` |

### 2.2 Regras normativas

| # | Regra |
|---|-------|
| 1 | O produtor **DEVE** autenticar-se por **mTLS** com certificado de workload (`../021-Security/`). |
| 2 | A resposta é devolvida **somente após durabilidade confirmada** (invariante I9). |
| 3 | Não sendo possível confirmar, o serviço **DEVE** responder `503 AIOS-AUD-0005` com `retriable=true` — **nunca** aceitar. |
| 4 | `record_class` **DEVE** estar registrada; classe desconhecida → `AIOS-AUD-0002`. |
| 5 | Repetição do mesmo `Idempotency-Key`/`event_id` devolve o registro já gravado, com o **mesmo `record_hash`**. |
| 6 | O produtor **DEVE** manter o fato em seu outbox até receber confirmação. |

> **Por que `503` é a resposta correta.** No `../024-Observability/`, descartar
> telemetria e responder `200` é o desenho: bloquear o emissor seria pior. Aqui é o
> oposto — aceitar sem durabilidade trocaria uma indisponibilidade **visível** por uma
> lacuna **invisível**, e a lacuna é o defeito que este módulo existe para impedir
> (`./Vision.md` §5).

### 2.3 OpenAPI (recorte normativo)

```yaml
openapi: 3.1.0
info: { title: AIOS Audit API, version: "1.0" }
paths:
  /v1/audit/records:
    post:
      operationId: appendAuditRecord
      parameters:
        - { name: X-AIOS-Tenant,   in: header, required: true, schema: { type: string } }
        - { name: Idempotency-Key, in: header, required: true, schema: { type: string } }
        - { name: traceparent,     in: header, required: true, schema: { type: string } }
      requestBody:
        required: true
        content:
          application/json:
            schema: { $ref: '#/components/schemas/AuditRecordDraft' }
      responses:
        '201':
          description: Registro encadeado e DURÁVEL
          content:
            application/json:
              schema: { $ref: '#/components/schemas/AppendResult' }
        '409': { $ref: '#/components/responses/AuditError' }   # AIOS-AUD-0016
        '422': { $ref: '#/components/responses/AuditError' }   # AIOS-AUD-0002/0004
        '503': { $ref: '#/components/responses/AuditError' }   # AIOS-AUD-0005 (reenvie)
components:
  schemas:
    AuditRecordDraft:
      type: object
      required: [recordClass, occurredAt, actor, action, resourceUrn, outcome]
      properties:
        recordClass: { type: string, example: "policy.decision" }
        occurredAt:  { type: string, format: date-time }
        actor:
          type: object
          required: [urn, kind]
          properties:
            urn:  { type: string, example: "urn:aios:acme:agent:01J9Z8Q6H7K2M4N6P8R0S2T4V6" }
            kind: { type: string, enum: [human, service, agent, external] }
        action:      { type: string, example: "memory:item:read" }
        resourceUrn: { type: string, example: "urn:aios:acme:memory:01J9ZA00000000000000000000" }
        outcome:     { type: string, enum: [allowed, denied, succeeded, failed, compensated] }
        decisionRef: { type: string, description: "decision_id do 022-Policy" }
        subjectRef:  { type: string, description: "titular pseudonimizado; habilita crypto-shredding" }
        payload:     { type: object, additionalProperties: true }
        eventId:     { type: string }
    AppendResult:
      type: object
      required: [id, seq, recordHash, duplicate]
      properties:
        id:         { type: string, example: "01J9ZN00000000000000000000" }
        seq:        { type: integer, format: int64, example: 418291 }
        partitionKey: { type: string, example: "acme:07" }
        recordHash: { type: string, example: "sha256:7a1c9f…" }
        prevHash:   { type: string }
        recordedAt: { type: string, format: date-time }
        duplicate:  { type: boolean }
```

### 2.4 Exemplo

```http
POST /v1/audit/records HTTP/1.1
X-AIOS-Tenant: acme
Idempotency-Key: 01J9ZM00000000000000000000
traceparent: 00-4bf92f3577b34da6a3ce929d0e0e4736-00f067aa0ba902b7-01
Content-Type: application/json

{ "recordClass": "privacy.erasure_receipt",
  "occurredAt": "2026-07-23T11:02:00.000Z",
  "actor": { "urn": "urn:aios:acme:service:database", "kind": "service" },
  "action": "db:erasure:execute",
  "resourceUrn": "urn:aios:acme:policy:01J9ZP00000000000000000000",
  "outcome": "succeeded",
  "subjectRef": "tok:9f2c4e…",
  "payload": { "tablesAffected": 12, "rowsErased": 1842, "receiptHash": "sha256:3b8e…" } }
```

```json
HTTP/1.1 201 Created
{ "id": "01J9ZN00000000000000000000", "seq": 418291, "partitionKey": "acme:07",
  "recordHash": "sha256:7a1c9f…", "prevHash": "sha256:c04b1a…",
  "recordedAt": "2026-07-23T11:02:00.184Z", "duplicate": false }
```

## 3. Consulta e prova

| Operação | REST | gRPC | Idempotente | Capability |
|----------|------|------|-------------|------------|
| Consultar trilha | `POST /v1/audit/records:query` | `QueryRecords` | sim (leitura) | `aud:record:read` |
| Ler registro | `GET /v1/audit/records/{id}` | `GetRecord` | sim (leitura) | `aud:record:read` |
| Prova de inclusão | `GET /v1/audit/records/{id}:proof` | `GetInclusionProof` | sim (leitura) | `aud:record:read` |
| Verificar integridade | `POST /v1/audit/verify` | `VerifyChain` | sim | `aud:integrity:verify` |
| Listar/ler selos | `GET /v1/audit/seals[/{urn}]` | `ListSeals`/`GetSeal` | sim (leitura) | `aud:seal:read` |

> **Toda consulta é registrada** como fato auditável de classe `audit.access`, com
> solicitante, escopo e volume retornado (ADR-0257). Quem consultou a trilha é
> exatamente o que o auditor do auditor precisa saber.

**Exemplo — prova de inclusão:**

```http
GET /v1/audit/records/01J9ZN00000000000000000000:proof HTTP/1.1
X-AIOS-Tenant: acme
```

```json
{ "recordId": "01J9ZN00000000000000000000",
  "recordHash": "sha256:7a1c9f…",
  "merklePath": [
    { "position": "right", "hash": "sha256:1d0a…" },
    { "position": "left",  "hash": "sha256:9be3…" },
    { "position": "right", "hash": "sha256:44f7…" } ],
  "seal": {
    "urn": "urn:aios:acme:event:01J9ZQ00000000000000000000",
    "merkleRoot": "sha256:e51b…", "sealHash": "sha256:88ca…",
    "prevSealHash": "sha256:2f19…",
    "signature": "MEUCIQ…", "signedBy": "urn:aios:_platform:key:01J9ZR00000000000000000000",
    "sealedAt": "2026-07-23T11:03:00.000Z",
    "wormObject": "s3://aios-audit/acme/07/seal-01J9ZQ.jsonl.zst" },
  "publicKeyRef": "https://api.aios.local/.well-known/jwks.json#01J9ZR…",
  "verifyInstructions": "https://docs.aios/rfc-0250#verification" }
```

O auditor recomputa a raiz a partir do `recordHash` e do `merklePath`, e verifica a
assinatura com a chave pública — **sem consultar o AIOS de novo**.

## 4. Governança e exportação

| Operação | REST | gRPC | Idempotente | Capability |
|----------|------|------|-------------|------------|
| Registrar classe auditável | `PUT /v1/audit/classes/{key}` | `PutAuditableClass` | sim | `aud:class:manage` |
| Aplicar *legal hold* | `POST /v1/audit/legal-holds` | `ApplyLegalHold` | sim (key) | `aud:hold:apply` |
| Liberar *legal hold* | `POST /v1/audit/legal-holds/{urn}:release` | `ReleaseLegalHold` | sim | `aud:hold:release` |
| Listar *holds* ativos | `GET /v1/audit/legal-holds?active=true` | `ListLegalHolds` | sim (leitura) | `aud:hold:read` |
| Executar apagamento | `POST /v1/audit/erasures` | `ExecuteErasure` | sim (key) | `aud:erasure:execute` |
| Ler comprovante | `GET /v1/audit/erasures/{urn}` | `GetErasureReceipt` | sim (leitura) | `aud:erasure:read` |
| Solicitar exportação | `POST /v1/audit/exports` | `RequestExport` | sim (key) | `aud:export:request` |
| Baixar pacote probatório | `GET /v1/audit/exports/{urn}/package` | `DownloadExport` | sim (leitura) | `aud:export:download` |

### 4.1 O pacote probatório

Estrutura entregue ao auditor externo (FR-017, ADR-0259):

```
   package-01J9ZS….zip
   ├── manifest.json          escopo, contagens, package_hash, jurisdição de origem
   ├── records.jsonl.zst      registros em SERIALIZAÇÃO CANÔNICA (RFC-0250)
   ├── seals.jsonl            selos do período, com assinatura e prev_seal_hash
   ├── proofs.jsonl           caminho de Merkle de cada registro até seu selo
   ├── public-keys.jwks       chaves públicas de verificação (inclusive as históricas)
   └── VERIFY.md              procedimento passo a passo, sem depender do AIOS
```

`VERIFY.md` descreve os quatro passos: recomputar `record_hash`, conferir
`prev_hash`, recomputar a raiz de Merkle pelo caminho e verificar a assinatura do
selo. Registros `Shredded` aparecem com metadados e `payload_digest`, e a **ausência
do conteúdo é explicada no manifesto** — não é um dado faltante, é um apagamento
comprovado.

**Exemplo — aplicar *legal hold*:**

```http
POST /v1/audit/legal-holds HTTP/1.1
X-AIOS-Tenant: acme
Idempotency-Key: 01J9ZT00000000000000000000

{ "caseRef": "PROC-4821",
  "scope": { "subjectRef": "tok:9f2c4e…", "recordClass": ["policy.decision","agent.lifecycle"],
             "from": "2026-01-01T00:00:00Z", "to": "2026-07-23T00:00:00Z" },
  "legalBasis": "Requisição judicial — obrigação legal de preservação",
  "approvedBy": "urn:aios:acme:principal:01J9ZG00000000000000000000" }
```

```json
HTTP/1.1 201 Created
{ "urn": "urn:aios:acme:event:01J9ZV00000000000000000000",
  "recordsHeld": 18422, "appliedAt": "2026-07-23T11:10:00Z", "expiresAt": null }
```

`expiresAt: null` é **válido** neste módulo (`./StateMachine.md` §6.1): um *hold*
expira quando o caso encerra, não quando um temporizador vence.

## 5. gRPC (`aios.audit.v1`)

```proto
syntax = "proto3";
package aios.audit.v1;

service AuditIngest {
  rpc AppendRecord (AuditRecordDraft)      returns (AppendResult);
  rpc AppendBatch  (AuditRecordDraftBatch) returns (AppendResultBatch);
}

service AuditQuery {
  rpc QueryRecords      (RecordQuery)   returns (RecordPage);
  rpc GetRecord         (RecordRef)     returns (AuditRecord);
  rpc GetInclusionProof (RecordRef)     returns (InclusionProof);
  rpc VerifyChain       (VerifyRequest) returns (VerificationResult);
  rpc ListSeals         (SealQuery)     returns (SealPage);
  rpc GetSeal           (SealRef)       returns (Seal);
}

service AuditGovernance {
  rpc PutAuditableClass (AuditableClass)   returns (AuditableClass);
  rpc ApplyLegalHold    (LegalHoldRequest) returns (LegalHold);
  rpc ReleaseLegalHold  (ReleaseRequest)   returns (LegalHold);
  rpc ListLegalHolds    (HoldQuery)        returns (HoldPage);
  rpc ExecuteErasure    (ErasureRequest)   returns (ErasureReceipt);
  rpc GetErasureReceipt (ReceiptRef)       returns (ErasureReceipt);
  rpc RequestExport     (ExportRequest)    returns (AuditExport);
  rpc DownloadExport    (ExportRef)        returns (stream ExportChunk);
}

// NÃO EXISTEM, por desenho (ADR-0258):
//   rpc UpdateRecord(...)   rpc DeleteRecord(...)
```

Mapeamento de erro gRPC: `status` → `google.rpc.Status` + `ErrorInfo` com o mesmo
`code` (RFC-0001 §5.4).

## 6. Catálogo de códigos de erro

> Formato RFC-0001 §5.4. Domínio reservado: **`AUD`** (0001–0099), registrado em
> `../004-API/Errors.md` conforme RFC-0001 §8 (ratificação em **ADR-0250**).

| Código | HTTP | `retriable` | Significado |
|--------|------|-------------|-------------|
| `AIOS-AUD-0001` | 404 | não | Registro, selo, *hold*, comprovante ou exportação não encontrado. |
| `AIOS-AUD-0002` | 422 | não | `record_class` não registrada em `audit.auditable_class`. |
| `AIOS-AUD-0003` | 403 | não | Tenant divergente do contexto autenticado. |
| `AIOS-AUD-0004` | 422 | não | Envelope inválido (campo obrigatório ausente ou malformado). |
| `AIOS-AUD-0005` | 503 | **sim** | Escrita não confirmada como durável — **o produtor DEVE reenviar**. |
| `AIOS-AUD-0006` | 409 | sim | Conflito de concorrência otimista em operação de governança. |
| `AIOS-AUD-0007` | 409 | não | Transição de estado inválida (viola a FSM). |
| `AIOS-AUD-0008` | **423** | não | Operação bloqueada por **legal hold** vigente. |
| `AIOS-AUD-0009` | 403 | não | Aprovação dupla exigida e ausente (*hold* ou exportação). |
| `AIOS-AUD-0010` | 422 | não | Retenção sem `legal_basis` ou abaixo do piso legal. |
| `AIOS-AUD-0011` | 500 | não | **Quebra de cadeia detectada** — integridade comprometida. |
| `AIOS-AUD-0012` | 409 | não | Registro já apagado (`Shredded`/`Expired`): payload indisponível **por desenho**. |
| `AIOS-AUD-0013` | 429 | sim | Limite de consulta/exportação excedido. |
| `AIOS-AUD-0014` | 412 | não | Selo ausente ou assinatura inválida para o intervalo. |
| `AIOS-AUD-0015` | 422 | não | Escopo de exportação vazio ou acima do máximo. |
| `AIOS-AUD-0016` | 409 | não | Tentativa de alteração de registro (append-only). |

**Envelope (RFC 7807), exemplo:**

```json
HTTP/1.1 423 Locked
Content-Type: application/problem+json

{ "type": "https://docs.aios/errors/legal-hold-active",
  "title": "Legal Hold Active",
  "status": 423,
  "code": "AIOS-AUD-0008",
  "detail": "Apagamento bloqueado: legal hold PROC-4821 vigente sobre o escopo.",
  "instance": "urn:aios:acme:event:01J9ZV00000000000000000000",
  "traceId": "4bf92f3577b34da6a3ce929d0e0e4736",
  "timestamp": "2026-07-23T11:15:00.000Z",
  "retriable": false }
```

> `423 Locked` para o *legal hold* é deliberado e alinha-se ao `AIOS-DB-0010` do
> `../005-Database/`, que usa o mesmo status para expurgo bloqueado por *hold*. Um
> `403` sugeriria falta de permissão; o recurso está **preservado por obrigação
> legal**, e a distinção importa para quem recebe a resposta.

## 7. Idempotência, paginação e versionamento

| Aspecto | Regra |
|---------|-------|
| Idempotência | `Idempotency-Key` obrigatório na ingestão e nas mutações de governança; resultado persistido ≥ 24 h (RFC-0001 §5.5); repetição devolve o mesmo `record_hash` (NFR-014). |
| Eventos | Dedupe por `event_id` (RFC-0001 §5.5); reentrega do JetStream é esperada. |
| Consulta | Leitura pura: repetir é seguro — **mas cada consulta gera um registro de acesso** (ADR-0257). |
| Paginação | `pageToken`/`pageSize` (máx. 1000) em `QueryRecords`, `ListSeals` e `ListLegalHolds`; token opaco e estável. |
| Versionamento | Caminho `/v1` + `X-AIOS-Api-Version`; gRPC em `aios.audit.v1` (RFC-0001 §5.7). |
| Evolução | Novos `record_class`, `outcome` e `anomaly.kind` são **aditivos**. A **serialização canônica** do `record_hash` (RFC-0250) **NÃO PODE** mudar sem nova versão de RFC: alterá-la invalidaria a verificação de todo o histórico. |

## 8. Limites operacionais

| Limite | Valor | Chave | Erro |
|--------|-------|-------|------|
| Registros por lote | 500 | `aud.ingest.batch_max_size` | `AIOS-AUD-0015` |
| Ingestão por tenant | 100.000/s | `aud.ingest.max_rate_per_s_per_tenant` | `AIOS-AUD-0013` |
| Duração de consulta | 30 s | `aud.query.max_duration_s` | `AIOS-AUD-0013` |
| Registros por consulta | 100.000 | `aud.query.max_records` | `AIOS-AUD-0013` |
| Registros por exportação | 1.000.000 | `aud.export.max_records` | `AIOS-AUD-0015` |
| Validade do pacote | 72 h | `aud.export.package_ttl_h` | `AIOS-AUD-0001` |
| Página máxima | 1000 | — | `AIOS-AUD-0004` |

## 9. Referências

- Fonte única de verdade: `./_DESIGN_BRIEF.md` §5
- Contratos centrais: `../003-RFC/RFC-0001-Architecture-Baseline.md` §5.4–§5.7
- Registros (erros, domínios): `../004-API/Errors.md`, `../004-API/Events.md`
- Eventos: `./Events.md` · Estruturas: `./ClassDiagrams.md` · Exemplos: `./Examples.md`

*Fim de `API.md`.*
