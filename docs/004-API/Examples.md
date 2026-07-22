---
Documento: Examples
Módulo: 004-API
Status: Draft
Versão: 0.1
Última atualização: 2026-07-21
Responsável (RACI-A): Arquiteto do Módulo 004-API
ADRs relacionados: ADR-0040, ADR-0044
RFCs relacionados: RFC-0001
Depende de: ./API.md, ../030-CLI/README.md, ../031-SDK/README.md
---

# 004-API — Exemplos

> Exemplos executáveis via CLI (`030`), SDK (`031`) e chamada HTTP direta,
> usando os verbos/endpoints reais definidos em `./API.md`.

## 1. "Hello World" — chamar uma rota *pass-through* via CLI

```bash
# Autentica e configura o CLI (fora do escopo deste módulo; ver ../030-CLI/)
aios auth login --tenant acme

# Cria um agente via o Kernel, passando pelo API Gateway
aios kernel agents spawn --agent-type researcher --priority normal
# Internamente: POST /v1/kernel/agents via o Gateway (004-API), com
# Idempotency-Key e traceparent gerados automaticamente pelo CLI.
```

## 2. Chamada HTTP direta (curl) — caminho feliz

```bash
curl -X POST https://api.aios.example/v1/kernel/agents \
  -H "Authorization: Bearer $TOKEN" \
  -H "X-AIOS-Tenant: acme" \
  -H "Idempotency-Key: $(uuidgen)" \
  -H "Content-Type: application/json" \
  -d '{"agentType": "researcher", "priority": "normal"}'
```

## 3. Repetir uma chamada com segurança (idempotência de borda)

```bash
KEY=$(uuidgen)

# Primeira tentativa — timeout de rede do lado do cliente
curl -X POST https://api.aios.example/v1/kernel/agents \
  -H "Authorization: Bearer $TOKEN" -H "X-AIOS-Tenant: acme" \
  -H "Idempotency-Key: $KEY" -d '{"agentType":"researcher"}' --max-time 1 || true

# Retry seguro com a MESMA chave — o Gateway devolve o mesmo resultado,
# sem criar um segundo agente (ver ./API.md §5, FR-006)
curl -X POST https://api.aios.example/v1/kernel/agents \
  -H "Authorization: Bearer $TOKEN" -H "X-AIOS-Tenant: acme" \
  -H "Idempotency-Key: $KEY" -d '{"agentType":"researcher"}'
```

## 4. Consumir eventos de tarefa via SSE (JavaScript, Web Console)

```javascript
const es = new EventSource(
  `/v1/tasks/${taskUrn}/events`,
  { withCredentials: true } // token via cookie de sessão do Console
);

es.addEventListener("task.execution.progressed", (evt) => {
  const data = JSON.parse(evt.data);
  console.log(`Progresso: ${data.progressPercent}%`);
});

es.addEventListener("task.execution.completed", (evt) => {
  console.log("Tarefa concluída:", JSON.parse(evt.data));
  es.close();
});
```

## 5. Consumir o contrato agregado (descoberta de API)

```bash
curl https://api.aios.example/v1/api/openapi.json | jq '.paths | keys'
```

## 6. Consultar o catálogo de erros (integração de terceiros)

```bash
curl https://api.aios.example/v1/api/errors | jq '.items[] | select(.domain=="API")'
```

## 7. Administração — registrar uma nova rota (papel `api-registry-admin`)

```bash
curl -X POST https://api.aios.example/v1/api/routes \
  -H "Authorization: Bearer $ADMIN_TOKEN" \
  -H "Idempotency-Key: $(uuidgen)" \
  -H "Content-Type: application/json" \
  -d '{
    "moduleOwner": "012-Planning",
    "pathPattern": "/v1/planning/plans/{urn}",
    "methods": ["GET"],
    "protocol": "rest",
    "upstreamCluster": "planning-cluster",
    "apiMajor": 1,
    "authRequired": true,
    "requiredRoles": ["planning-viewer"],
    "schemaRef": "planning.v1#/paths/~1plans~1{urn}"
  }'
```

## 8. Administração — ciclo de vida de versão (SDK Python, exemplo avançado)

```python
from aios_sdk import ApiRegistryClient

client = ApiRegistryClient(base_url="https://api.aios.example", token=admin_token)

# T-01: registrar nova major em Draft (feito pelo pipeline do módulo dono)
client.versions.register(module_owner="012-Planning", major=2,
                          contract_ref="minio://contracts/012-planning/v2/openapi.json")

# T-02: publicar após contract tests aprovados
client.versions.publish(module_owner="012-Planning", major=2,
                         idempotency_key="pub-012-v2-2026-07-21")

# A major 1 é automaticamente Deprecated (T-04) pelo Gateway.
# Após a janela de depreciação (>= 180 dias) e tráfego residual baixo:
client.versions.retire(module_owner="012-Planning", major=1,
                        idempotency_key="retire-012-v1-2027-01-20")
```

## 9. Tratar rejeição de rate limit com *backoff*

```python
import time, random

def call_with_backoff(fn, max_retries=5):
    for attempt in range(max_retries):
        resp = fn()
        if resp.status_code != 429:
            return resp
        retry_after = float(resp.headers.get("Retry-After", "1"))
        time.sleep(retry_after + random.uniform(0, 0.5))
    raise RuntimeError("Rate limit não liberou após retries")
```

*Fim de `Examples.md`.*
