---
Documento: Events
Módulo: 010-Memory
Status: Draft
Versão: 0.1
Última atualização: 2026-07-20
Responsável (RACI-A): Arquiteto do Módulo 010-Memory
ADRs relacionados: ADR-0102, ADR-0103, ADR-0107, ADR-0109
RFCs relacionados: RFC-0001 (baseline), RFC-0100 (a propor)
Depende de: 003-RFC (RFC-0001), 004-API, 011-Context, 020-Communication, 023-Learning, 024-Observability, 025-Audit, 026-Cost-Optimizer, 040-Glossary
---

# 010-Memory — Events

> Catálogo de eventos NATS do `MemoryService`. **Deriva** da §6 do
> `_DESIGN_BRIEF.md` e **NÃO PODE contradizê-lo**. O envelope de evento
> (CloudEvents), a convenção de subjects e a semântica de idempotência são
> **reutilizados** de [RFC-0001 §5.2/§5.3/§5.5](../003-RFC/RFC-0001-Architecture-Baseline.md)
> — aqui apenas os aplicamos. Palavras normativas conforme RFC 2119/8174.

## 1. Convenções de barramento

- **Domínio reservado:** `memory` (brief §0). Subjects seguem
  `aios.<tenant>.memory.<entidade>.<acao>` (RFC-0001 §5.3), com namespace de tenant
  isolando o tráfego (contas NATS por tenant).
- **Envelope:** todo evento usa o envelope CloudEvents 1.0 de RFC-0001 §5.2
  (`specversion`, `id`, `source`, `type`, `subject`, `time`, `tenant`,
  `traceparent`, `datacontenttype`, `dataschema`, `data`).
- **Publicação:** exclusivamente via **Outbox** (`OutboxPublisher`) — linha de banco
  gravada na mesma transação da mutação e relay para NATS/JetStream (atomicidade
  DB+evento, brief §2.2/§9-F8).
- **Entrega:** **at-least-once**. Consumidores DEVEM deduplicar por `event.id`
  (RFC-0001 §5.5). Efeito *exactly-once* obtido por at-least-once + idempotência
  (ver [040-Glossary](../040-Glossary/Glossary.md)).
- **Transporte:** stream JetStream durável; mensagens envenenadas vão para **DLQ**
  após N falhas (brief §9-F10). Ver [020-Communication](../020-Communication/Events.md).

```
 Mutação (remember/consolidate/forget/...)
        │  (mesma transação)
   ┌────▼─────────┐    relay at-least-once    ┌──────────────┐
   │ OutboxEvent   │──────────────────────────▶│ NATS JetStream│──▶ consumidores (dedup por event.id)
   │ status=pending│      (marca published)     │  aios.<t>.mem.*│
   └───────────────┘                            └──────┬────────┘
                                                       │ N falhas
                                                       ▼  DLQ
```

## 2. Versionamento de schema

- Cada evento referencia um `dataschema` versionado
  (`aios://schemas/memory.<entidade>.<acao>/<v>`); a evolução DEVE ser **aditiva**
  e consumidores DEVEM tolerar campos desconhecidos (RFC-0001 §5.7).
- Uma mudança incompatível cria uma nova versão de schema e coexiste com a anterior;
  o registro central fica em [004-API/Events.md](../004-API/Events.md).
- **Minimização (LGPD/GDPR):** o `data` carrega o **mínimo** necessário e referencia
  o item por URN — conteúdo/PII não trafegam no envelope (RFC-0001 §7; brief §12.3;
  ver [Security.md](./Security.md)).

## 3. Eventos emitidos (produtor = 010-Memory)

Derivado da §6.1 do brief. `<t>` = slug do tenant.

| Subject | `type` | Gatilho (estado) | Consumidores | Ordenação |
|---------|--------|------------------|--------------|-----------|
| `aios.<t>.memory.item.stored` | `aios.memory.item.stored` | Item → `ACTIVE` | 023, 024, 025 | por `agent_id` |
| `aios.<t>.memory.item.consolidated` | `aios.memory.item.consolidated` | Item promovido/mesclado (`COMMITTED`) | 023, 019, 025 | por `agent_id` |
| `aios.<t>.memory.item.forgotten` | `aios.memory.item.forgotten` | Item → `FORGOTTEN` (tombstone) | 025, 026, 024 | por `agent_id` |
| `aios.<t>.memory.item.recalled` | `aios.memory.item.recalled` | Recall executado (amostrado) | 011, 024 | best-effort |
| `aios.<t>.memory.item.decayed` | `aios.memory.item.decayed` | Item → `DECAYING` | 024, 026 | best-effort |
| `aios.<t>.memory.item.archived` | `aios.memory.item.archived` | Item → `ARCHIVED` (cold/MinIO) | 026, 024 | por `agent_id` |
| `aios.<t>.memory.item.purged` | `aios.memory.item.purged` | Purge físico concluído (RTBF) | 025 (auditoria RTBF) | por `agent_id` |
| `aios.<t>.memory.consolidation.started` | `aios.memory.consolidation.started` | Job → `RUNNING` | 023, 024 | por `job_id` |
| `aios.<t>.memory.consolidation.completed` | `aios.memory.consolidation.completed` | Job `COMMITTED` | 023, 024 | por `job_id` |
| `aios.<t>.memory.consolidation.failed` | `aios.memory.consolidation.failed` | Job `FAILED` | 023, 024, 025 | por `job_id` |
| `aios.<t>.memory.consolidation.rolledback` | `aios.memory.consolidation.rolledback` | Job `ROLLED_BACK` | 023, 025 | por `job_id` |
| `aios.<t>.memory.quota.exceeded` | `aios.memory.quota.exceeded` | Cota de camada estourada | 026, 024 | por `tenant/agent/layer` |

### 3.1 Ordenação e particionamento

- Não há ordem total global. A ordem **relativa por chave** é preservada dentro do
  shard (`hash(tenant_id, agent_id) mod N`, brief §10). Eventos de item usam
  `agent_id` como chave de partição; eventos de job usam `job_id`.
- `item.recalled` e `item.decayed` são **amostrados** (telemetria/custo) e são
  best-effort — consumidores NÃO DEVEM assumir contagem 1:1 com operações.

## 4. Eventos consumidos (010-Memory como consumidor)

Derivado da §6.2 do brief.

| Subject (assinatura) | Produtor | Ação do `MemoryService` |
|----------------------|----------|--------------------------|
| `aios.<t>.learning.consolidation.requested` | 023-Learning | Dispara `ConsolidationJob` dirigido. |
| `aios.<t>.learning.policy.rolledback` | 023-Learning | Reverte consolidação associada (por versão). |
| `aios.<t>.context.recall.requested` | 011-Context | Executa recall seletivo e responde. |
| `aios.<t>.agent.lifecycle.terminated` | 006/008 | Expira Working/Short-Term do agente. |
| `aios.<t>.agent.lifecycle.suspended` | 006/008 | Move estado quente → cold (MinIO/PG). |
| `aios.<t>.policy.decision.updated` | 022-Policy | Recarrega decisões de acesso a memória. |
| `aios.<t>.security.rtbf.requested` | 021/025 | Agenda expurgo (direito ao esquecimento). |

Assinaturas usam wildcards NATS por domínio quando aplicável (ex.:
`aios.acme.agent.lifecycle.>`). O consumo é idempotente: dedup por `event.id` +
idempotência das operações internas (brief §9-F8/F9).

## 5. Schemas JSON (envelope + `data`)

O envelope é o de RFC-0001 §5.2; abaixo detalha-se o `data` por evento.

### 5.1 `memory.item.stored`

```json
{
  "specversion": "1.0",
  "id": "01J9Z8QGEVT0STORED0000000001",
  "source": "urn:aios:acme:service:memory",
  "type": "aios.memory.item.stored",
  "subject": "urn:aios:acme:memory:01J9Z8QCITEMULID0000000001",
  "time": "2026-07-20T12:34:56.789Z",
  "tenant": "acme",
  "traceparent": "00-4bf92f3577b34da6a3ce929d0e0e4736-00f067aa0ba902b7-01",
  "datacontenttype": "application/json",
  "dataschema": "aios://schemas/memory.item.stored/1",
  "data": {
    "urn": "urn:aios:acme:memory:01J9Z8QCITEMULID0000000001",
    "agent_id": "01J9Z8Q6H7K2M4N6P8R0S2T4V6",
    "layer": "semantic",
    "kind": "fact",
    "salience": 0.7,
    "pii": false,
    "content_hash": "sha256:9f2b...",
    "created_at": "2026-07-20T12:34:56.780Z"
  }
}
```

### 5.2 `memory.item.consolidated`

```json
{
  "type": "aios.memory.item.consolidated",
  "dataschema": "aios://schemas/memory.item.consolidated/1",
  "data": {
    "urn": "urn:aios:acme:memory:01J9Z8QHCONSITEM0000000001",
    "from_layer": "short_term",
    "to_layer": "long_term",
    "consolidation_version": 7,
    "parent_ids": ["01J9Z8...A", "01J9Z8...B"],
    "job_id": "01J9Z8QEJOBULID00000000001"
  }
}
```

### 5.3 `memory.item.forgotten` e `memory.item.purged`

```json
{
  "type": "aios.memory.item.forgotten",
  "dataschema": "aios://schemas/memory.item.forgotten/1",
  "data": {
    "urn": "urn:aios:acme:memory:01J9Z8QCITEMULID0000000001",
    "strategy": "decay",
    "tombstoned_at": "2026-07-20T13:00:00.000Z",
    "grace_period": "7d"
  }
}
```

```json
{
  "type": "aios.memory.item.purged",
  "dataschema": "aios://schemas/memory.item.purged/1",
  "data": {
    "urn": "urn:aios:acme:memory:01J9Z8QCITEMULID0000000001",
    "reason": "rtbf",
    "purged_vectors": true,
    "purged_blob": true
  }
}
```

### 5.4 `memory.consolidation.*`

```json
{
  "type": "aios.memory.consolidation.completed",
  "dataschema": "aios://schemas/memory.consolidation.completed/1",
  "data": {
    "job_id": "01J9Z8QEJOBULID00000000001",
    "agent_id": "01J9Z8Q6H7K2M4N6P8R0S2T4V6",
    "from_layer": "short_term",
    "to_layer": "long_term",
    "version_id": 7,
    "recall_rate_before": 0.96,
    "recall_rate_after": 0.97,
    "stats": { "promoted": 120, "merged": 8, "deduped": 3 }
  }
}
```

### 5.5 `memory.quota.exceeded`

```json
{
  "type": "aios.memory.quota.exceeded",
  "dataschema": "aios://schemas/memory.quota.exceeded/1",
  "data": {
    "tenant": "acme",
    "agent_id": "01J9Z8Q6H7K2M4N6P8R0S2T4V6",
    "layer": "semantic",
    "limit_items": 1000000,
    "used_items": 1000001,
    "usage_ratio": 1.0
  }
}
```

## 6. Correlação com a máquina de estados

Cada evento emitido corresponde a uma transição canônica do `MemoryItem` (brief §4.1)
ou do `ConsolidationJob` (brief §4.2). Rastreabilidade:

| Transição de estado | Evento emitido |
|---------------------|----------------|
| `INGESTED → ACTIVE` | `item.stored` |
| `CONSOLIDATING → CONSOLIDATED` (job `COMMITTED`) | `item.consolidated` + `consolidation.completed` |
| `ACTIVE/CONSOLIDATED → DECAYING` | `item.decayed` |
| `CONSOLIDATED → ARCHIVED` | `item.archived` |
| `* → FORGOTTEN` | `item.forgotten` |
| `FORGOTTEN → PURGED` | `item.purged` |
| Job `→ RUNNING` / `FAILED` / `ROLLED_BACK` | `consolidation.started` / `.failed` / `.rolledback` |

## 7. Riscos e alternativas

| Risco / decisão | Mitigação / alternativa |
|-----------------|--------------------------|
| Duplicata de evento (at-least-once). | Dedup obrigatória por `event.id`; consumidores idempotentes. Alternativa (exactly-once nativo) descartada por custo/acoplamento. |
| Perda de evento em crash pós-commit. | Outbox transacional + relay; RPO de evento = 0 (brief §9-F8). |
| Vazamento de PII em `data`. | Minimização; só URN/metadados no envelope; conteúdo permanece em PG/MinIO sob RLS. |
| Volume alto de `item.recalled`/`decayed`. | Amostragem configurável; best-effort; não usado para lógica de negócio. |
| Poison message trava consumidor. | DLQ JetStream + alerta + inspeção manual (brief §9-F10). |

Decisões governadas por `ADR-0102` (consolidação versionada), `ADR-0103`
(esquecimento) e `RFC-0100` (contrato de consolidação/esquecimento) — ver
[ADR.md](./ADR.md) e [RFC.md](./RFC.md).
