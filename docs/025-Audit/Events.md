---
Documento: Events
Módulo: 025-Audit
Status: Draft
Versão: 0.1
Última atualização: 2026-07-23
Responsável (RACI-A): Arquiteto do Módulo 025-Audit
ADRs relacionados: ADR-0004, ADR-0010, ADR-0250, ADR-0251, ADR-0254, ADR-0256
RFCs relacionados: RFC-0001, RFC-0250, RFC-0251
Depende de: _DESIGN_BRIEF.md §6, 020-Communication, 004-API (registro de domínios), 005-Database
---

# 025-Audit — Catálogo de Eventos

> Envelope **CloudEvents** e convenção de subjects conforme
> `../003-RFC/RFC-0001-Architecture-Baseline.md` §5.2–§5.3 — **referenciados, não
> redefinidos**. Domínio `<dominio>` = **`audit`**, valor **já previsto** na lista de
> domínios da RFC-0001 §5.3.

## 1. Regra de conteúdo

**Eventos deste módulo carregam referências e provas, nunca o conteúdo auditado.**

A tentação de violar essa regra é maior em `audit.anomaly.detected`: seria conveniente
anexar o registro suspeito para agilizar a investigação. Seria também a forma mais
rápida de publicar, em um stream com múltiplos consumidores, exatamente o conteúdo que
está cifrado por titular dentro da trilha. Quem precisa do conteúdo **consulta sob
PEP** — e essa consulta fica registrada.

## 2. Eventos emitidos

| Subject | `type` | Quando | Stream | Consumidores conhecidos |
|---------|--------|--------|--------|-------------------------|
| `aios.<tenant>.audit.record.sealed` | `aios.audit.record.sealed` | Intervalo selado (T-02). | `AUDIT_SEAL` | `024`, `029`, `032` |
| `aios.<tenant>.audit.anomaly.detected` | `aios.audit.anomaly.detected` | Quebra de cadeia, lacuna, selo atrasado, classe silenciosa, volume ou consulta anômala. | `AUDIT_ANOMALY` | `024`, `029`, `021` |
| `aios.<tenant>.audit.legalhold.applied` | `aios.audit.legalhold.applied` | *Hold* ativado. | `AUDIT_GOVERNANCE` | **`005`**, `010`, `024`, `029` |
| `aios.<tenant>.audit.legalhold.released` | `aios.audit.legalhold.released` | *Hold* liberado. | `AUDIT_GOVERNANCE` | **`005`**, `010`, `024` |
| `aios.<tenant>.audit.erasure.completed` | `aios.audit.erasure.completed` | Apagamento criptográfico ou expurgo concluído, com comprovante. | `AUDIT_GOVERNANCE` | `021`, `005`, `032` |
| `aios.<tenant>.audit.retention.expired` | `aios.audit.retention.expired` | Payload expurgado por fim de retenção (T-06). | `AUDIT_GOVERNANCE` | `029` |
| `aios.<tenant>.audit.export.completed` | `aios.audit.export.completed` | Pacote probatório pronto. | `AUDIT_GOVERNANCE` | `032`, `029` |
| `aios._platform.audit.seal.published` | `aios.audit.seal.published` | Selo de plataforma publicado (âncora global). | `AUDIT_SEAL` | `029` |

> `audit.legalhold.applied` é o evento de maior alcance operacional do módulo: ao
> consumi-lo, o `../005-Database/` **suspende o expurgo** das tabelas afetadas
> (`AIOS-DB-0010`). É o mecanismo pelo qual uma decisão jurídica se propaga a todo o
> plano de dados sem acoplamento direto.

## 3. Schemas (payload `data`)

Envelope completo conforme RFC-0001 §5.2; abaixo apenas o campo `data`.

### 3.1 `aios.audit.record.sealed`

```json
{
  "specversion": "1.0",
  "id": "01J9ZW00000000000000000000",
  "source": "urn:aios:acme:service:audit",
  "type": "aios.audit.record.sealed",
  "subject": "urn:aios:acme:event:01J9ZQ00000000000000000000",
  "time": "2026-07-23T11:03:00.000Z",
  "tenant": "acme",
  "traceparent": "00-4bf92f3577b34da6a3ce929d0e0e4736-00f067aa0ba902b7-01",
  "datacontenttype": "application/json",
  "dataschema": "aios://schemas/audit.record.sealed/1",
  "data": {
    "sealUrn": "urn:aios:acme:event:01J9ZQ00000000000000000000",
    "partitionKey": "acme:07",
    "seqFrom": 418100, "seqTo": 418291, "recordCount": 192,
    "merkleRoot": "sha256:e51b…",
    "sealHash": "sha256:88ca…",
    "prevSealHash": "sha256:2f19…",
    "signedBy": "urn:aios:_platform:key:01J9ZR00000000000000000000",
    "sealedAt": "2026-07-23T11:03:00.000Z"
  }
}
```

Carrega **hashes e intervalos**, jamais registros.

### 3.2 `aios.audit.anomaly.detected`

```json
{ "data": {
    "anomalyId": "01J9ZX00000000000000000000",
    "kind": "sequence_gap",
    "severity": "critical",
    "partitionKey": "acme:07",
    "detail": { "expectedSeq": 418292, "observedSeq": 418295, "missingCount": 3 },
    "detectedAt": "2026-07-23T11:20:00.000Z" } }
```

Para `kind = chain_break`, `detail` traz `recordId`, `expectedHash` e `observedHash` —
**hashes**, não conteúdo. Para `class_silent`, traz `recordClass`, `producerModule`,
`expectedRatePerDay` e `observedRate`.

### 3.3 `aios.audit.legalhold.applied` / `.released`

```json
{ "data": {
    "holdUrn": "urn:aios:acme:event:01J9ZV00000000000000000000",
    "caseRef": "PROC-4821",
    "scope": { "subjectRef": "tok:9f2c4e…",
               "recordClass": ["policy.decision","agent.lifecycle"],
               "from": "2026-01-01T00:00:00Z", "to": "2026-07-23T00:00:00Z" },
    "legalBasis": "Requisição judicial — obrigação legal de preservação",
    "requestedBy": "urn:aios:acme:principal:01J9ZY00000000000000000000",
    "approvedBy":  "urn:aios:acme:principal:01J9ZG00000000000000000000",
    "recordsHeld": 18422,
    "appliedAt": "2026-07-23T11:10:00.000Z",
    "expiresAt": null } }
```

Em `.released`, acrescenta `releasedBy`, `releasedAt` e `releaseReason`.

> `expiresAt: null` é **válido** e esperado (`./StateMachine.md` §6.1). Consumidores
> **NÃO DEVEM** tratar a ausência de prazo como erro nem inferir um prazo próprio: o
> *hold* vale até o `.released` correspondente.

### 3.4 `aios.audit.erasure.completed`

```json
{ "data": {
    "receiptUrn": "urn:aios:acme:event:01J9ZZ00000000000000000000",
    "kind": "crypto_shred",
    "subjectRef": "tok:9f2c4e…",
    "originModule": "025-Audit",
    "recordsAffected": 3184,
    "receiptHash": "sha256:5d7a…",
    "chainRecordId": "01JA0000000000000000000000",
    "executedAt": "2026-07-23T11:30:00.000Z" } }
```

`chainRecordId` aponta para o registro **na própria trilha** que prova o apagamento:
apagar sem deixar prova do apagamento seria indistinguível de nunca ter apagado.

### 3.5 `aios.audit.retention.expired` e `aios.audit.export.completed`

```json
{ "data": {
    "recordClass": "task.execution",
    "purgedCount": 1204831,
    "retentionDays": 1825,
    "oldestOccurredAt": "2021-07-01T00:00:00Z",
    "chainIntact": true,
    "executedAt": "2026-07-23T02:00:00.000Z" } }
```

```json
{ "data": {
    "exportUrn": "urn:aios:acme:event:01JA0100000000000000000000",
    "recordCount": 842391, "sealCount": 4392,
    "packageHash": "sha256:9c1e…",
    "requestedBy": "urn:aios:acme:principal:01J9ZY00000000000000000000",
    "approvedBy":  "urn:aios:acme:principal:01J9ZG00000000000000000000",
    "expiresAt": "2026-07-26T11:00:00.000Z" } }
```

`chainIntact: true` no expurgo por retenção não é decoração: é a confirmação de que a
verificação foi executada **depois** do expurgo e a cadeia continua íntegra (FR-012).

## 4. Eventos consumidos

O módulo assina os subjects declarados em `audit.auditable_class.source_subjects`.
Acrescentar uma classe é o **caminho normativo** para tornar um novo fato auditável
(ADR-0256) — e é isso que torna a **ausência** detectável.

| Subject assinado | Produtor | `record_class` resultante |
|-------------------|----------|---------------------------|
| `aios.<tenant>.policy.decision.denied` | `../022-Policy/` | `policy.decision` |
| `aios.<tenant>.policy.bundle.updated` / `.rolledback` | `../022-Policy/` | `policy.bundle` |
| `aios.<tenant>.policy.exception.granted` / `.expired` | `../022-Policy/` | `policy.exception` |
| `aios.<tenant>.security.credential.issued` | `../021-Security/` | `security.credential` |
| `aios.<tenant>.security.token.revoked` | `../021-Security/` | `security.credential` |
| `aios.<tenant>.security.principal.disabled` | `../021-Security/` | `security.principal` |
| `aios.<tenant>.security.rtbf.requested` | `../021-Security/` | `privacy.rtbf` (**e dispara o apagamento próprio**) |
| `aios.<tenant>.agent.lifecycle.spawned` / `.terminated` | `../006-Kernel/` | `agent.lifecycle` |
| `aios.<tenant>.task.execution.completed` / `.failed` | `../009-Scheduler/` | `task.execution` |
| `aios.<tenant>.telemetry.alert.acknowledged` | `../024-Observability/` | `ops.alert` |
| `aios.<tenant>.cost.budget.exhausted` | `../026-Cost-Optimizer/` | `cost.budget` |
| `aios._platform.deployment.rollout.started` | `../028-Deployment/` | `ops.deployment` |

Comprovantes de expurgo executados por `../005-Database/`, `../010-Memory/` e
`../024-Observability/` chegam pela **API direta** (`./API.md` §2) e são registrados
como `privacy.erasure_receipt` com `origin_module` preservado.

Todo consumo é **idempotente** por `event.id` (RFC-0001 §5.5): a reentrega do
JetStream é esperada e não duplica registro (invariante I7).

## 5. Semântica de entrega e streams

| Stream | Subjects | Retenção | Réplicas | Semântica |
|--------|----------|----------|----------|-----------|
| `AUDIT_SEAL` | `aios.*.audit.record.*`, `aios.*.audit.seal.*` | 1825 dias | 5 | at-least-once; base da verificação histórica |
| `AUDIT_ANOMALY` | `aios.*.audit.anomaly.*` | 1825 dias | 5 | at-least-once; histórico de integridade |
| `AUDIT_GOVERNANCE` | `aios.*.audit.{legalhold,erasure,retention,export}.*` | 1825 dias | 5 | at-least-once; prova de governança |

- **5 réplicas** e retenção de 5 anos, contra as 3 réplicas usuais dos demais módulos:
  o custo adicional é justificado porque estes eventos são, eles mesmos, elementos de
  prova.
- Publicação via **Outbox transacional** (`audit.outbox`, `./Database.md` §9): a
  transição e o evento são gravados na mesma transação.
- Consumidores **DEVEM** deduplicar por `event.id` e **DEVEM** tolerar campos
  desconhecidos (RFC-0001 §5.7).

> **O stream não é a trilha.** A retenção de 5 anos aqui é conveniência de integração;
> a prova está em `audit.record`, nos selos assinados e na cópia WORM. Um consumidor
> que dependa do stream para reconstruir a trilha está usando a ferramenta errada — e
> perderá tudo o que exceder a retenção.

## 6. Ordenação e reprocessamento

```
   record.sealed(seq 1..192) ──▶ record.sealed(seq 193..410) ──▶ …
        │                              │
        └──── ordem por partitionKey dentro do stream AUDIT_SEAL ────┘

   Consumidor que processe fora de ordem DEVE usar (partitionKey, seqTo)
   como chave de progresso, e IGNORAR selo com seqTo já processado.
   Selos NUNCA são reemitidos com conteúdo diferente: sealHash é único.
```

Para `legalhold.applied`/`.released`, a ordenação importa mais: aplicar depois de
liberar deixaria o `../005-Database/` com o expurgo suspenso indevidamente. O
consumidor **DEVE** usar `holdUrn` + `appliedAt`/`releasedAt` e ignorar evento cujo
efeito já esteja refletido.

## 7. Versionamento de schema

| `dataschema` | Versão | Mudanças permitidas |
|--------------|--------|---------------------|
| `aios://schemas/audit.record.sealed/1` | 1 | Aditivas (novos campos opcionais) |
| `aios://schemas/audit.anomaly.detected/1` | 1 | Novos `kind` (aditivos) |
| `aios://schemas/audit.legalhold.applied/1` | 1 | Aditivas |
| `aios://schemas/audit.erasure.completed/1` | 1 | Novos `kind` (aditivos) |
| `aios://schemas/audit.retention.expired/1` | 1 | Aditivas |
| `aios://schemas/audit.export.completed/1` | 1 | Aditivas |

Mudança incompatível exige nova versão de schema e coexistência (RFC-0001 §5.7);
registro em `../004-API/Events.md` (RFC-0001 §8).

> A **serialização canônica** do `record_hash` (RFC-0250) é caso à parte: alterá-la
> invalidaria a verificação de **todo o histórico**. Ela só muda com nova versão da
> RFC e um período em que ambas as formas são verificáveis (`./RFC.md` §5).

## 8. Referências

- Fonte única de verdade: `./_DESIGN_BRIEF.md` §6
- Envelope e subjects: `../003-RFC/RFC-0001-Architecture-Baseline.md` §5.2–§5.3
- Barramento e streams: `../020-Communication/`
- Registro de domínios e schemas: `../004-API/Events.md`
- Outbox: `./Database.md` §9 · FSM: `./StateMachine.md`

*Fim de `Events.md`.*
