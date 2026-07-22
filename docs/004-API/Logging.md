---
Documento: Logging
Módulo: 004-API
Status: Draft
Versão: 0.1
Última atualização: 2026-07-21
Responsável (RACI-A): Arquiteto do Módulo 004-API
ADRs relacionados: ADR-0010 (global)
RFCs relacionados: RFC-0001 (§5.6, §7)
Depende de: ../024-Observability/README.md, ./Security.md
---

# 004-API — Log Estruturado

> Todo log do Gateway é emitido via Serilog → Seq (padrão transversal do
> AIOS, `ADR-0010`), correlacionado por `trace_id`/`span_id`/`tenant_id`
> conforme RFC-0001 §5.6. Este documento define os **eventos de log** do
> módulo — não redefine o transporte/armazenamento (`../024-Observability/`).

## 1. Níveis de log

| Nível | Uso no Gateway |
|-------|------------------|
| `Trace` | Detalhe de resolução de rota por réplica (desligado em produção por padrão). |
| `Debug` | Decisões de cache (JWKS/decisão do PDP/idempotência) hit/miss. |
| `Information` | Ciclo de vida de requisição bem-sucedida (uma linha por requisição, amostrável sob alto volume). |
| `Warning` | Rejeições de AuthN/AuthZ/rate-limit/schema; *circuit breaker* mudando de estado. |
| `Error` | Falha de upstream não conforme (`AIOS-API-0014`); timeout (`AIOS-API-0006`); falha de publicação de evento (retry esgotado). |
| `Fatal` | Falha de *boot* (não consegue carregar *snapshot* inicial de `ContractRegistry`). |

## 2. Campos obrigatórios em todo log de requisição

| Campo | Descrição | Obrigatório |
|-------|-----------|--------------|
| `timestamp` | RFC 3339, UTC. | sim |
| `trace_id` / `span_id` | W3C Trace Context (RFC-0001 §5.6). | sim |
| `tenant_id` | Tenant do chamador (ou `_platform` para operações de registro). | sim |
| `route_id` | Identidade da `RouteDefinition` resolvida (quando aplicável). | sim (após resolução) |
| `method` / `path` / `api_major` | Metadados da requisição. | sim |
| `status_code` | Status HTTP de resposta. | sim |
| `error_code` | `AIOS-API-<NNNN>` quando aplicável. | condicional |
| `latency_ms` | Overhead do Gateway (não inclui tempo do upstream). | sim |
| `upstream_cluster` | Destino roteado (quando aplicável). | sim (após roteamento) |
| `idempotency_key_hash` | Hash da chave (nunca a chave em claro, quando sensível). | condicional |

## 3. Campos explicitamente **proibidos** em log padrão

- Corpo (`body`) de requisição/resposta com dados de negócio — minimização
  por padrão (RFC-0001 §7).
- Cabeçalho `Authorization` (token) em claro — **DEVE** ser redigido
  (`Authorization: [REDACTED]`).
- Cookies de sessão em claro.
- Chaves JWKS privadas (nunca aplicável ao Gateway, que só consome chaves
  públicas).

Estes campos PODEM ser habilitados **apenas** em ambiente de depuração
isolado (nunca produção), com aprovação explícita registrada — controle
transversal de `../025-Audit/`.

## 4. Exemplo de linha de log estruturado

```json
{
  "timestamp": "2026-07-21T12:06:00.123Z",
  "level": "Warning",
  "message": "Request rejected: AuthN failed",
  "trace_id": "4bf92f3577b34da6a3ce929d0e0e4736",
  "span_id": "00f067aa0ba902b7",
  "tenant_id": "acme",
  "route_id": "01J9Z8QAR0VTE0000000000001",
  "method": "POST",
  "path": "/v1/kernel/agents",
  "api_major": 1,
  "status_code": 401,
  "error_code": "AIOS-API-0002",
  "latency_ms": 0.8,
  "component": "AuthNFilter"
}
```

## 5. Correlação e amostragem

- Todo log de requisição **DEVE** carregar o mesmo `trace_id` do span OTel
  correspondente (ver `./Monitoring.md`), permitindo pivotar de log para
  trace em uma única consulta.
- Logs de nível `Information` (requisições bem-sucedidas) PODEM ser
  amostrados sob volume muito alto (> 50.000 req/s por réplica), preservando
  100% dos logs `Warning`/`Error`/`Fatal` — decisão configurável em
  `../024-Observability/`.

## 6. Retenção

Retenção de logs segue a política transversal de `../024-Observability/` e
`../025-Audit/` — este módulo não define retenção própria, apenas os campos
e níveis emitidos. Ver `./Security.md` §6 para a distinção entre log
operacional (aqui) e trilha de auditoria imutável (`025-Audit`).

*Fim de `Logging.md`.*
