---
Documento: Examples
Módulo: 011-Context
Status: Draft
Versão: 0.1
Última atualização: 2026-07-20
Responsável (RACI-A): Arquiteto do Módulo 011-Context
ADRs relacionados: ADR-0110, ADR-0111, ADR-0112, ADR-0113, ADR-0114, ADR-0115, ADR-0116, ADR-0117, ADR-0118, ADR-0119
RFCs relacionados: RFC-0001, RFC-0011
Depende de: 010-Memory, 017-Model-Router, 024-Observability, 022-Policy, 025-Audit, 020-Communication, 005-Database, 021-Security
---

# 011-Context — Examples

> Todos os exemplos assumem o Gateway YARP em `https://api.aios.local` (REST) e
> o serviço gRPC `aios.context.v1.ContextService` acessível internamente em
> `context.aios.internal:7011`. Endpoints, campos e erros seguem
> `./API.md`/`./_DESIGN_BRIEF.md` §5. Toda mutação usa `Idempotency-Key` e
> propaga `traceparent`/`X-AIOS-Tenant` conforme RFC-0001 §5.6. Os `tenant_id`
> usados (`acme`) e IDs ULID são ilustrativos.

---

## 1. Hello World — montar um contexto (REST)

O exemplo mais simples: montar um `ContextBundle` mínimo para uma chamada de
LLM, deixando o `011-Context` decidir orçamento, recuperação e compressão.

```bash
# Hello world: assembly mínimo via REST
curl -sS -X POST https://api.aios.local/v1/context/assemble \
  -H "Content-Type: application/json" \
  -H "X-AIOS-Tenant: acme" \
  -H "Idempotency-Key: 01J9Z9AA000000000000000A" \
  -H "traceparent: 00-4bf92f3577b34da6a3ce929d0e0e4736-00f067aa0ba902b7-01" \
  -d '{
        "agent_urn": "urn:aios:acme:agent:01J9Z9AB1C2D3E4F5G6H7J8K9M",
        "task_urn": "urn:aios:acme:task:01J9Z9AB1C2D3E4F5G6H7J8K9N",
        "model_id": "claude-large-v3",
        "system_prompt": "Você é um assistente de suporte técnico da Acme.",
        "user_query": "Como faço para redefinir a senha da minha conta?",
        "memory_query_hint": "reset de senha, conta de usuário"
      }'
```

Resposta esperada (caminho feliz — orçamento respeitado, sem cache prévio):

```json
{
  "bundle_id": "01J9Z9AC2D3E4F5G6H7J8K9L0M",
  "tenant_id": "acme",
  "model_id": "claude-large-v3",
  "model_max_tokens": 200000,
  "token_budget": 150000,
  "reserved_output_tokens": 50000,
  "tokens_raw": 8420,
  "tokens_in": 6103,
  "compression_ratio": 0.2751,
  "status": "SERVED",
  "cache_outcome": "MISS",
  "trace_id": "4bf92f3577b34da6a3ce929d0e0e4736",
  "created_at": "2026-07-20T12:00:00.010Z",
  "expires_at": "2026-07-20T12:15:00.010Z"
}
```

> Nota: `status=SERVED` com `cache_outcome=MISS` significa que o pipeline
> completo (retrieve→rank→dedup→compress) foi executado. O conteúdo montado
> (mensagens prontas para o provedor de LLM) é recuperável via
> `GET /v1/context/bundles/{bundle_id}` (§8).

---

## 2. Hello World — mesma montagem via gRPC (grpcurl)

```bash
# Equivalente ao exemplo 1, via gRPC (grpcurl para fins de exemplo/CLI)
grpcurl -plaintext \
  -H "x-aios-tenant: acme" \
  -H "idempotency-key: 01J9Z9AA000000000000000A" \
  -H "traceparent: 00-4bf92f3577b34da6a3ce929d0e0e4736-00f067aa0ba902b7-01" \
  -d '{
        "agent_urn": "urn:aios:acme:agent:01J9Z9AB1C2D3E4F5G6H7J8K9M",
        "task_urn": "urn:aios:acme:task:01J9Z9AB1C2D3E4F5G6H7J8K9N",
        "model_id": "claude-large-v3",
        "system_prompt": "Você é um assistente de suporte técnico da Acme.",
        "user_query": "Como faço para redefinir a senha da minha conta?"
      }' \
  context.aios.internal:7011 aios.context.v1.ContextService/Assemble
```

---

## 3. SDK Python — assembly com tratamento de erro

```python
# Exemplo de uso do AIOS SDK (031-SDK) para montar contexto antes de uma
# chamada de LLM. O SDK encapsula geração de Idempotency-Key, propagação de
# traceparent e parsing do envelope de erro RFC 7807 (RFC-0001 §5.4).

from aios_sdk import ContextClient, AssembleRequest, ContextError

client = ContextClient(tenant="acme")

request = AssembleRequest(
    agent_urn="urn:aios:acme:agent:01J9Z9AD3E4F5G6H7J8K9L0M1N",
    task_urn="urn:aios:acme:task:01J9Z9AD3E4F5G6H7J8K9L0M1P",
    model_id="claude-large-v3",
    system_prompt="Você é um assistente de suporte técnico da Acme.",
    user_query="Meu pedido #48213 não chegou. O que aconteceu?",
    memory_query_hint="pedido 48213, atraso de entrega",
)

try:
    bundle = client.assemble(request)  # Idempotency-Key gerado automaticamente
    print(f"status={bundle.status} tokens_in={bundle.tokens_in} "
          f"ratio={bundle.compression_ratio:.2f} cache={bundle.cache_outcome}")
except ContextError as err:
    # err.code segue o padrão AIOS-CTX-<NNNN> (ver ./API.md §3)
    if err.code == "AIOS-CTX-0001":
        print("Contexto mínimo não cabe no orçamento do modelo; considere um modelo maior.")
    elif err.code == "AIOS-CTX-0005":
        print("Recall de memória expirou; bundle pode ter vindo parcial (DEGRADED).")
    else:
        raise
```

---

## 4. Consultando apenas o cache semântico (sem montar o bundle completo)

Útil quando o chamador quer saber, de forma barata, se uma resposta já
computada existe antes de decidir se vale a pena montar/inferir.

```bash
curl -sS -X POST https://api.aios.local/v1/context/cache/lookup \
  -H "Content-Type: application/json" \
  -H "X-AIOS-Tenant: acme" \
  -d '{
        "model_id": "claude-large-v3",
        "prompt_text": "Como faço para redefinir a senha da minha conta?",
        "cache_scope": "task_type",
        "scope_ref": "support.password_reset"
      }'
```

Resposta em caso de *hit*:

```json
{
  "cache_id": "01J9Z9AE4F5G6H7J8K9L0M1N2P",
  "hit": true,
  "similarity": 0.968,
  "similarity_threshold": 0.92,
  "tokens_saved": 6103,
  "cost_saved_usd": 0.0312,
  "response_ref": "minio://ctx-cache/acme/01J9Z9AE4F5G6H7J8K9L0M1N2P.json"
}
```

Resposta em caso de *miss*:

```json
{
  "hit": false,
  "reason": "NO_CANDIDATE_ABOVE_THRESHOLD"
}
```

---

## 5. Armazenando uma resposta no cache semântico

Chamado tipicamente pelo `007-Agent-Runtime` após uma inferência bem-sucedida,
para que chamadas futuras semanticamente equivalentes possam ser servidas do
cache.

```bash
curl -sS -X POST https://api.aios.local/v1/context/cache/entries \
  -H "Content-Type: application/json" \
  -H "X-AIOS-Tenant: acme" \
  -H "Idempotency-Key: 01J9Z9AF5G6H7J8K9L0M1N2P3Q" \
  -d '{
        "model_id": "claude-large-v3",
        "prompt_text": "Como faço para redefinir a senha da minha conta?",
        "cache_scope": "task_type",
        "scope_ref": "support.password_reset",
        "response_inline": "Para redefinir sua senha, acesse Configurações > Segurança > Redefinir Senha...",
        "ttl_seconds": 3600
      }'
```

```json
{
  "cache_id": "01J9Z9AG6H7J8K9L0M1N2P3Q4R",
  "status": "INDEXED",
  "ttl_seconds": 3600,
  "created_at": "2026-07-20T12:03:00.000Z"
}
```

---

## 6. Comprimindo um conjunto de fragmentos avulsos (sem `assemble` completo)

Útil quando um chamador já tem seus próprios fragmentos (por exemplo, um lote
de resultados de ferramenta) e só quer aplicar a compressão hierárquica do
`011` antes de incluí-los manualmente em outro pipeline.

```bash
curl -sS -X POST https://api.aios.local/v1/context/compress \
  -H "Content-Type: application/json" \
  -H "X-AIOS-Tenant: acme" \
  -H "Idempotency-Key: 01J9Z9AH7J8K9L0M1N2P3Q4R5S" \
  -d '{
        "model_id": "claude-large-v3",
        "target_tokens": 2000,
        "method": "hierarchical",
        "fragments": [
          {"source_kind": "tool_result", "content": "Log completo do pedido #48213: criado em 2026-07-15, pago em 2026-07-15, separado em 2026-07-16, sem movimentação desde então..."},
          {"source_kind": "tool_result", "content": "Histórico de tickets do cliente: 3 tickets anteriores sobre atraso de entrega em outras compras..."}
        ]
      }'
```

```json
{
  "tokens_before": 5230,
  "tokens_after": 1890,
  "compression_ratio": 0.6386,
  "method": "hierarchical",
  "summary_nodes": [
    {"node_id": "01J9Z9AJ8K9L0M1N2P3Q4R5S6T", "level": 1, "tokens": 1890}
  ]
}
```

---

## 7. Estimando tokens antes de decidir o que enviar

```bash
curl -sS -X POST https://api.aios.local/v1/context/tokens/estimate \
  -H "Content-Type: application/json" \
  -H "X-AIOS-Tenant: acme" \
  -d '{
        "model_id": "claude-large-v3",
        "text": "Como faço para redefinir a senha da minha conta?"
      }'
```

```json
{
  "model_id": "claude-large-v3",
  "estimated_tokens": 11,
  "exact": true,
  "tokenizer_source": "017-model-router"
}
```

Se o tokenizer exato do modelo estiver indisponível, o campo `exact` indica
uma estimativa aproximada (erro-alvo ≤ 2%, NFR-010):

```json
{
  "model_id": "claude-large-v3",
  "estimated_tokens": 11,
  "exact": false,
  "tokenizer_source": "heuristic_fallback"
}
```

---

## 8. Recuperando um bundle já montado

```bash
curl -sS https://api.aios.local/v1/context/bundles/01J9Z9AC2D3E4F5G6H7J8K9L0M \
  -H "X-AIOS-Tenant: acme"
```

```json
{
  "bundle_id": "01J9Z9AC2D3E4F5G6H7J8K9L0M",
  "status": "SERVED",
  "cache_outcome": "MISS",
  "tokens_raw": 8420,
  "tokens_in": 6103,
  "compression_ratio": 0.2751,
  "fragments": [
    {"fragment_id": "01J9Z9AK9L0M1N2P3Q4R5S6T7U", "source_kind": "system", "ordinal": 0, "tokens_final": 42, "included": true},
    {"fragment_id": "01J9Z9AL0M1N2P3Q4R5S6T7U8V", "source_kind": "memory", "ordinal": 1, "tokens_final": 310, "relevance_score": 0.91, "included": true},
    {"fragment_id": "01J9Z9AM1N2P3Q4R5S6T7U8V9W", "source_kind": "memory", "ordinal": 2, "tokens_final": 0, "relevance_score": 0.79, "included": false}
  ]
}
```

Se o `bundle_id` for inválido ou já tiver expirado (`EXPIRED`):

```json
{
  "type": "https://docs.aios/errors/bundle-not-found",
  "title": "Bundle Not Found",
  "status": 404,
  "code": "AIOS-CTX-0007",
  "detail": "bundle_id does not exist or has expired.",
  "traceId": "9f1e5c2a7b3d4e6f8091a2b3c4d5e6f7",
  "timestamp": "2026-07-20T12:20:00.000Z",
  "retriable": false
}
```

---

## 9. Consultando e atualizando um `BudgetProfile`

```bash
# Lê o BudgetProfile efetivo do escopo "task_type=support"
curl -sS "https://api.aios.local/v1/context/budgets/task_type:support" \
  -H "X-AIOS-Tenant: acme"
```

```json
{
  "profile_id": "01J9Z9AN2P3Q4R5S6T7U8V9W0X",
  "scope": "task_type",
  "scope_ref": "support",
  "reserved_output_ratio": 0.25,
  "alloc_system": 0.05,
  "alloc_instruction": 0.10,
  "alloc_memory": 0.45,
  "alloc_history": 0.30,
  "alloc_tools": 0.10,
  "min_compression_ratio": 0.30,
  "updated_at": "2026-07-01T09:00:00.000Z"
}
```

```bash
# Atualiza o perfil para dar mais peso à memória (suporte a clientes recorrentes)
curl -sS -X PUT https://api.aios.local/v1/context/budgets/task_type:support \
  -H "Content-Type: application/json" \
  -H "X-AIOS-Tenant: acme" \
  -H "Authorization: Bearer <token-com-role-context-admin>" \
  -H "Idempotency-Key: 01J9Z9AO3Q4R5S6T7U8V9W0X1Y" \
  -d '{
        "scope": "task_type",
        "scope_ref": "support",
        "reserved_output_ratio": 0.25,
        "alloc_system": 0.05,
        "alloc_instruction": 0.10,
        "alloc_memory": 0.55,
        "alloc_history": 0.20,
        "alloc_tools": 0.10,
        "min_compression_ratio": 0.30
      }'
```

Se os pesos não somarem 1.0, a resposta é um erro de validação, não um ajuste
automático:

```json
{
  "type": "https://docs.aios/errors/invalid-budget-profile",
  "title": "Invalid Budget Profile",
  "status": 422,
  "code": "AIOS-CTX-0008",
  "detail": "alloc_system + alloc_instruction + alloc_memory + alloc_history + alloc_tools must equal 1.0 (got 1.05).",
  "traceId": "1a2b3c4d5e6f70819293a4b5c6d7e8f9",
  "timestamp": "2026-07-20T12:22:00.000Z",
  "retriable": false
}
```

---

## 10. Consumindo eventos de janela de contexto (Python + NATS)

```python
# Assina os eventos de contexto do tenant "acme" para acompanhar eficácia
# de compressão e de cache em tempo real.
import asyncio
import json
from nats.aio.client import Client as NATS

async def main():
    nc = NATS()
    await nc.connect("nats://nats.aios.internal:4222")

    async def handler(msg):
        event = json.loads(msg.data.decode())
        # Consumidores DEVEM deduplicar por event.id (at-least-once, RFC-0001 §5.2)
        print(f"[{event['type']}] subject={event['subject']} id={event['id']} "
              f"data={event['data']}")

    await nc.subscribe("aios.acme.context.window.>", cb=handler)
    await nc.subscribe("aios.acme.context.cache.>", cb=handler)

    await asyncio.sleep(3600)

asyncio.run(main())
```

Saída típica ao longo de algumas montagens:

```
[aios.context.window.cache_miss]   subject=urn:aios:acme:context:01J9...L id=01J9Z9AP...
[aios.context.window.assembled]    subject=urn:aios:acme:context:01J9...L id=01J9Z9AQ...
[aios.context.window.cache_hit]    subject=urn:aios:acme:context:01J9...M id=01J9Z9AR...
[aios.context.cache.evicted]       subject=urn:aios:acme:ctxcache:01J9...P id=01J9Z9AS...
```

---

## 11. Lidando com degradação (`010-Memory` indisponível) — cliente resiliente

```python
# Exemplo avançado: cliente que trata explicitamente o estado DEGRADED como
# sucesso parcial, em vez de tratá-lo como falha de assembly.
from aios_sdk import ContextClient, AssembleRequest, ContextError

client = ContextClient(tenant="acme")
request = AssembleRequest(
    agent_urn="urn:aios:acme:agent:01J9Z9AT4S6T7U8V9W0X1Y2Z3A",
    model_id="claude-large-v3",
    system_prompt="Você é um assistente de suporte técnico da Acme.",
    user_query="Qual o status do meu chamado #9911?",
    memory_query_hint="chamado 9911",
)

try:
    bundle = client.assemble(request)
    if bundle.status == "SERVED" and bundle.cache_outcome == "MISS":
        print("Montagem completa (pipeline integral executado).")
    print(f"tokens_in={bundle.tokens_in} ratio={bundle.compression_ratio:.2f}")
except ContextError as err:
    if err.code == "AIOS-CTX-0005":
        # 010-Memory não respondeu a tempo; o SDK ainda pode ter recebido
        # um bundle parcial em vez de uma exceção, dependendo da versão do SDK.
        print("Recall de memória expirou; considerando fallback sem memória.")
    elif err.code == "AIOS-CTX-0006":
        print("Backend de cache indisponível; bypass de cache aplicado (miss forçado).")
    else:
        raise
```

---

## 12. Invalidando cache manualmente por escopo

Requer a *role* `context-admin`; a operação passa pelo PDP (`022`) antes de
ser executada (não é auto-serviço livre).

```bash
curl -sS -X POST https://api.aios.local/v1/context/cache/invalidate \
  -H "Content-Type: application/json" \
  -H "X-AIOS-Tenant: acme" \
  -H "Authorization: Bearer <token-com-role-context-admin>" \
  -H "Idempotency-Key: 01J9Z9AU5T7U8V9W0X1Y2Z3A4B" \
  -d '{
        "cache_scope": "task_type",
        "scope_ref": "support.password_reset",
        "reason": "atualização de política de suporte"
      }'
```

```json
{
  "invalidated_count": 14,
  "cache_scope": "task_type",
  "scope_ref": "support.password_reset",
  "invalidated_at": "2026-07-20T12:30:00.000Z"
}
```

---

## 13. Ciclo completo — do "hello world" ao avançado (fluxo único)

Resumo integrando os exemplos anteriores num único fluxo mental, do ponto de
vista do `007-Agent-Runtime` usando o SDK Python:

```python
from aios_sdk import ContextClient, AssembleRequest, ContextError

client = ContextClient(tenant="acme")

# 1. Montar contexto (hello world)
req = AssembleRequest(
    agent_urn="urn:aios:acme:agent:01J9Z9AV6U8V9W0X1Y2Z3A4B5C",
    model_id="claude-large-v3",
    system_prompt="Você é um assistente de suporte técnico da Acme.",
    user_query="Preciso cancelar minha assinatura.",
    memory_query_hint="cancelamento de assinatura",
)
bundle = client.assemble(req)
print(bundle.status, bundle.cache_outcome, bundle.tokens_in)

# 2. Se veio de cache, usar a resposta cacheada diretamente (ver exemplo 4)
if bundle.cache_outcome == "HIT":
    response = client.get_cached_response(bundle.bundle_id)
else:
    # 3. Caso contrário, invocar o 017-Model-Router com o conteúdo montado
    #    (fora do escopo do 011; ilustrativo)
    response = call_model_router(bundle)
    # 4. Armazenar a resposta no cache semântico para reuso futuro (ver exemplo 5)
    client.cache_store(model_id=req.model_id, prompt_text=req.user_query,
                        response_inline=response, ttl_seconds=3600)

# 5. Consultar o bundle a qualquer momento para auditoria (ver exemplo 8)
snapshot = client.get_bundle(bundle.bundle_id)
print(snapshot.compression_ratio)
```

---

## 14. Referências

- Contratos completos de API: `./API.md`
- Catálogo de eventos: `./Events.md`
- Catálogo de erros: `./API.md` §3 / `_DESIGN_BRIEF.md` §5.3
- Máquina de estados: `./StateMachine.md`
- Configuração (limiares, TTLs, budgets): `./Configuration.md`

---

*Fim dos Exemplos do Módulo 011-Context.*
