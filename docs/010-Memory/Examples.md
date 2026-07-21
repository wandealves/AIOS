---
Documento: Examples (Exemplos Executáveis)
Módulo: 010-Memory
Status: Draft
Versão: 0.1
Última atualização: 2026-07-20
Responsável (RACI-A): Arquiteto do Módulo 010-Memory
ADRs relacionados: ADR-0007 (herdada), ADR-0100, ADR-0101, ADR-0102, ADR-0103, ADR-0104, ADR-0106
RFCs relacionados: RFC-0001 (baseline), RFC-0100 (a propor)
Depende de: 001-Architecture, 004-API, 011-Context, 017-Model-Router, 040-Glossary
---

# AIOS — Módulo 010 · Memory — Examples

> Exemplos executáveis derivados da **§5 (Superfície de API)** do
> `_DESIGN_BRIEF.md`. Todos os endpoints, campos, códigos de erro e headers são os
> canônicos do brief e da [RFC-0001](../003-RFC/RFC-0001-Architecture-Baseline.md)
> (§5.4 erro, §5.5 idempotência, §5.6 correlação). Nenhum exemplo introduz campo ou
> rota que não exista no brief. Palavras normativas conforme RFC 2119/8174.

---

## 1. Pré-requisitos e convenções

Base REST: `/v1/memory` (RFC-0001 §5.7). Toda **mutação** DEVE enviar os headers
obrigatórios (RFC-0001 §5.6):

| Header | Obrigatório em | Exemplo |
|--------|----------------|---------|
| `Authorization: Bearer <token>` | tudo | OAuth2/OIDC validado no Gateway (021) |
| `X-AIOS-Tenant` | tudo multi-tenant | `acme` |
| `Idempotency-Key` | mutações | `01J9Z8Q6H7K2M4N6P8R0S2T4V6` (ULID) |
| `traceparent` | tudo | `00-<trace_id>-<span_id>-01` |
| `X-AIOS-Api-Version` | opcional | `1` |

Variáveis usadas nos exemplos:

```bash
export AIOS_MEM="https://api.aios.local/v1/memory"
export TOKEN="$(aios auth token)"          # token OIDC do tenant
export TENANT="acme"
idem() { python -c 'import ulid;print(ulid.new())'; }  # gera ULID p/ Idempotency-Key
```

---

## 2. Hello World — `remember` e `recall`

### 2.1 `remember` (REST · cria item em Long-Term)

`POST /v1/memory/items` → `Remember(RememberRequest)`. Idempotente por `Idempotency-Key`.

```bash
curl -sS -X POST "$AIOS_MEM/items" \
  -H "Authorization: Bearer $TOKEN" \
  -H "X-AIOS-Tenant: $TENANT" \
  -H "Idempotency-Key: $(idem)" \
  -H "traceparent: 00-4bf92f3577b34da6a3ce929d0e0e4736-00f067aa0ba902b7-01" \
  -H "Content-Type: application/json" \
  -d '{
        "layer": "long_term",
        "kind": "fact",
        "content": { "text": "O cliente ACME prefere faturas em PDF." },
        "agent_id": "01J9Z8Q7B0C2D4E6F8G0H2J4K6",
        "tags": ["preferencia", "faturamento"],
        "legal_basis": "contract",
        "retention_class": "standard",
        "salience": 0.7
      }'
```

Resposta `201 Created` (item passou a `ACTIVE`; evento `memory.item.stored` emitido):

```json
{
  "urn": "urn:aios:acme:memory:01J9ZB2K5N7Q9S1T3V5W7X9Y1Z",
  "state": "ACTIVE",
  "layer": "long_term",
  "kind": "fact",
  "embedding_model": "urn:aios:acme:model:text-embed-default",
  "created_at": "2026-07-20T12:34:56.789Z"
}
```

> **Nota (F3):** se o Model Router (017) estiver indisponível, o item é aceito com
> `state = INGESTED` (sem embedding) e o embedding é gerado em *backlog* — a escrita
> **não** falha (ver [FailureRecovery.md §5](./FailureRecovery.md)).

### 2.2 `recall` (REST · busca híbrida rankeada)

`POST /v1/memory/recall` → `Recall(RecallRequest)`.

```bash
curl -sS -X POST "$AIOS_MEM/recall" \
  -H "Authorization: Bearer $TOKEN" \
  -H "X-AIOS-Tenant: $TENANT" \
  -H "traceparent: 00-4bf92f3577b34da6a3ce929d0e0e4736-00f067aa0ba902b8-01" \
  -H "Content-Type: application/json" \
  -d '{
        "query": "como o cliente quer receber faturas?",
        "agent_id": "01J9Z8Q7B0C2D4E6F8G0H2J4K6",
        "layer": ["long_term", "semantic"],
        "kind": ["fact"],
        "k": 5,
        "min_score": 0.30,
        "mode": "hybrid",
        "include_forgotten": false
      }'
```

Resposta `200 OK` (`RecallResult` rankeado; evento `memory.item.recalled` amostrado):

```json
{
  "results": [
    {
      "urn": "urn:aios:acme:memory:01J9ZB2K5N7Q9S1T3V5W7X9Y1Z",
      "score": 0.92,
      "layer": "long_term",
      "kind": "fact",
      "content": { "text": "O cliente ACME prefere faturas em PDF." },
      "signals": { "relevance": 0.90, "recency": 0.65, "salience": 0.70 }
    }
  ],
  "count": 1,
  "mode": "hybrid",
  "degraded": false,
  "next_cursor": null
}
```

---

## 3. SDK (Python · runtime do agente via gRPC/NATS)

O Agent Runtime (Python, plano de dados) **NÃO DEVE** acessar PostgreSQL/Redis
diretamente — DEVE passar pela API do `MemoryService` (brief §1.2, ADR-0107). O SDK
encapsula gRPC (`aios.memory.v1`).

```python
from aios.memory.v1 import MemoryClient  # cliente gRPC gerado do proto aios.memory.v1
import ulid

# tenant/traceparent/token são injetados pelo runtime (contexto do agente)
mem = MemoryClient.from_agent_context()

# remember: idempotente por idempotency_key
res = mem.remember(
    layer="semantic",
    kind="fact",
    content={"text": "pgvector usa HNSW para busca ANN"},
    tags=["infra", "vetorial"],
    legal_basis="legitimate_interest",
    retention_class="standard",
    salience=0.6,
    idempotency_key=str(ulid.new()),
)
print(res.urn, res.state)          # urn:aios:acme:memory:... ACTIVE

# recall: modo vetorial explícito nas camadas ANN
hits = mem.recall(
    query="qual índice o pgvector usa?",
    layer=["semantic", "episodic"],
    k=10,
    mode="vector",
    min_score=0.30,
)
for h in hits.results:
    print(f"{h.score:.2f}  {h.content['text']}")
```

> **Idempotência prática:** para reintentar com segurança após timeout, **reuse a
> mesma `idempotency_key`** — o servidor retorna o resultado memoizado
> (`IdempotencyStore`, ≥ 24h). Uma chave nova cria um novo item.

---

## 4. Consolidação dirigida (`consolidate`) e rollback

A consolidação promove itens entre camadas de forma **versionada e reversível**
(FR-005, ADR-0102). Normalmente é disparada pelo `023-Learning`
(`learning.consolidation.requested`), mas PODE ser manual.

### 4.1 Disparar consolidação

`POST /v1/memory/consolidate` → retorna `job_id` (execução assíncrona).

```bash
JOB=$(curl -sS -X POST "$AIOS_MEM/consolidate" \
  -H "Authorization: Bearer $TOKEN" -H "X-AIOS-Tenant: $TENANT" \
  -H "Idempotency-Key: $(idem)" -H "Content-Type: application/json" \
  -d '{
        "agent_id": "01J9Z8Q7B0C2D4E6F8G0H2J4K6",
        "from_layer": "short_term",
        "to_layer": "long_term",
        "trigger": "manual"
      }' | jq -r '.job_id')
echo "job=$JOB"
```

### 4.2 Acompanhar o job

`GET /v1/memory/jobs/{id}` → estado do `ConsolidationJob` (StateMachine §4.2 do brief).

```bash
curl -sS "$AIOS_MEM/jobs/$JOB" \
  -H "Authorization: Bearer $TOKEN" -H "X-AIOS-Tenant: $TENANT" | jq
```

```json
{
  "job_id": "01J9ZC0AA1BB2CC3DD4EE5FF6G",
  "state": "COMMITTED",
  "from_layer": "short_term",
  "to_layer": "long_term",
  "version_id": 42,
  "stats": { "promoted": 128, "merged": 12, "recall_rate_before": 0.951, "recall_rate_after": 0.958 }
}
```

### 4.3 Rollback (defesa contra *catastrophic forgetting*)

Se a consolidação regredir o **Memory Recall Rate** (NFR-011), o *rollback* é
**automático**; ainda assim é possível reverter manualmente:

```bash
curl -sS -X POST "$AIOS_MEM/consolidate/$JOB/rollback" \
  -H "Authorization: Bearer $TOKEN" -H "X-AIOS-Tenant: $TENANT" \
  -H "Idempotency-Key: $(idem)"
```

- Sucesso → estado restaurado à `ConsolidationVersion` anterior; evento
  `memory.consolidation.rolledback`.
- Se a versão não existe ou já está ativa → `409 AIOS-MEM-0042`.

---

## 5. Esquecimento controlado (`forget`) e RTBF

### 5.1 `forget` por política (decay/lru/quota/ttl)

`POST /v1/memory/forget` → aplica política; retorna contagem afetada (FR-006).

```bash
curl -sS -X POST "$AIOS_MEM/forget" \
  -H "Authorization: Bearer $TOKEN" -H "X-AIOS-Tenant: $TENANT" \
  -H "Idempotency-Key: $(idem)" -H "Content-Type: application/json" \
  -d '{
        "scope": "agent",
        "agent_id": "01J9Z8Q7B0C2D4E6F8G0H2J4K6",
        "strategy": "decay",
        "params": { "decay_threshold": 0.15 }
      }'
```

```json
{ "affected": 37, "strategy": "decay", "state_transition": "DECAYING→FORGET_PENDING" }
```

> Itens em `legal_hold` **não** são esquecidos por esta via → `423 AIOS-MEM-0050`
> (salvo RTBF autorizado).

### 5.2 Direito ao esquecimento (RTBF · expurgo direcionado)

`DELETE /v1/memory/items/{id}` → `PurgeItem` (FR-011, LGPD/GDPR).

```bash
curl -sS -X DELETE "$AIOS_MEM/items/01J9ZB2K5N7Q9S1T3V5W7X9Y1Z" \
  -H "Authorization: Bearer $TOKEN" -H "X-AIOS-Tenant: $TENANT" \
  -H "Idempotency-Key: $(idem)"
```

Resposta `202 Accepted` (`AIOS-MEM-0051` — expurgo aceito e agendado; assíncrono):

```json
{ "code": "AIOS-MEM-0051", "status": "accepted", "detail": "Purge agendado (RTBF)." }
```

O expurgo remove **vetor + blob + tombstone** após o `memory.forget.grace_period`
(default 7d); emite `memory.item.purged` para auditoria (025).

---

## 6. Blob grande, stats e listagem de camadas

### 6.1 Externalização de blob (FR-009, ADR-0106)

Conteúdo acima de `memory.item.inline_max_bytes` (default 64 KiB) é externalizado
para MinIO automaticamente — o cliente **não** muda a chamada; apenas envia o
conteúdo. A resposta traz `content_ref` em vez de `content` inline:

```json
{
  "urn": "urn:aios:acme:memory:01J9ZD5...",
  "state": "ACTIVE",
  "content_ref": "s3://aios-memory/acme/sha256/9f86d0818...",
  "content_hash": "9f86d0818...",
  "layer": "episodic"
}
```

### 6.2 Estatísticas e Memory Recall Rate

`GET /v1/memory/stats`:

```bash
curl -sS "$AIOS_MEM/stats?agent_id=01J9Z8Q7B0C2D4E6F8G0H2J4K6" \
  -H "Authorization: Bearer $TOKEN" -H "X-AIOS-Tenant: $TENANT" | jq
```

```json
{
  "memory_recall_rate": 0.958,
  "usage": {
    "long_term":  { "items": 12840, "bytes": 41231872, "usage_ratio": 0.12 },
    "semantic":   { "items": 98211, "bytes": 88123392, "usage_ratio": 0.41, "vectors": 98211 }
  }
}
```

### 6.3 Config vigente por camada

`GET /v1/memory/layers` retorna `MemoryLayerConfig` efetiva (TTL, cotas, HNSW):

```bash
curl -sS "$AIOS_MEM/layers" -H "Authorization: Bearer $TOKEN" -H "X-AIOS-Tenant: $TENANT" | jq '.[].layer'
```

---

## 7. Tratamento de erros (RFC 7807)

Todos os erros seguem o envelope RFC 7807 (RFC-0001 §5.4). Exemplo de cota excedida
(`AIOS-MEM-0010`, retriable):

```
HTTP/1.1 429 Too Many Requests
Content-Type: application/problem+json
Retry-After: 2

{
  "type": "https://docs.aios/errors/mem-quota-exceeded",
  "title": "Memory Quota Exceeded",
  "status": 429,
  "code": "AIOS-MEM-0010",
  "detail": "Cota de itens da camada 'semantic' excedida para o agente.",
  "traceId": "4bf92f3577b34da6a3ce929d0e0e4736",
  "retriable": true,
  "retryAfterMs": 2000
}
```

Padrão de reintento no cliente (respeita `retriable`/`retryAfterMs` + backoff):

```python
import time, random

def with_retry(op, idem_key, max_attempts=5):
    for n in range(max_attempts):
        resp = op(idempotency_key=idem_key)   # MESMA key em todas as tentativas
        if resp.ok or not resp.error.retriable:
            return resp
        delay = min(30.0, 0.1 * 2**n) * (0.8 + 0.4*random.random())  # backoff + jitter
        time.sleep(resp.error.retry_after_ms/1000 if resp.error.retry_after_ms else delay)
    raise RuntimeError("esgotou tentativas")
```

### 7.1 Referência rápida de erros usados nos exemplos

| Código | HTTP | Retriable | Onde aparece |
|--------|------|-----------|--------------|
| `AIOS-MEM-0003` | 409 | não | reuso de `Idempotency-Key` com payload divergente |
| `AIOS-MEM-0010` | 429 | sim | cota de camada excedida (§7) |
| `AIOS-MEM-0011` | 429 | sim | backpressure (fila de escrita saturada) |
| `AIOS-MEM-0020` | 422 | não | camada incompatível com `kind` |
| `AIOS-MEM-0042` | 409 | não | rollback impossível (§4.3) |
| `AIOS-MEM-0050` | 423 | não | item em `legal_hold` (§5.1) |
| `AIOS-MEM-0051` | 202 | — | RTBF aceito e agendado (§5.2) |

---

## 8. Fluxo completo comentado (do "hello" ao avançado)

```
1. remember(fact, short_term)          → ACTIVE            [evento stored]
2. ... uso repetido eleva access_count/salience ...
3. consolidate(short_term→long_term)   → job COMMITTED     [evento consolidated]
   └─ SNAPSHOTTING grava ConsolidationVersion (pré-imagem) antes de mutar
4. recall(query, mode=hybrid)          → RecallResult rankeado (RRF)  [recalled amostrado]
5. (regressão de Recall Rate?)         → rollback automático → ROLLED_BACK  [rolledback]
6. forget(scope=agent, strategy=decay) → itens → FORGET_PENDING → FORGOTTEN  [forgotten]
7. DELETE item (RTBF)                   → 202 → purge após grace period → PURGED  [purged]
```

Cada passo é idempotente, emite evento via Outbox (*at-least-once*) e é
observável por trace OTel correlacionado (NFR-013).

---

## 9. Referências

- [_DESIGN_BRIEF.md §5](./_DESIGN_BRIEF.md) — API canônica (endpoints, erros).
- [API.md](./API.md) — contratos OpenAPI/proto completos.
- [RFC-0001 §5.4/§5.5/§5.6](../003-RFC/RFC-0001-Architecture-Baseline.md) — erro, idempotência, correlação.
- [StateMachine.md](./StateMachine.md) — estados exercitados nos exemplos.
- [FailureRecovery.md](./FailureRecovery.md) — comportamento sob falha (F3, cota).
- [FAQ.md](./FAQ.md) — armadilhas comuns nesses fluxos.
