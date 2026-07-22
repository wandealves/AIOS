---
Documento: Configuration
Módulo: 004-API
Status: Draft
Versão: 0.1
Última atualização: 2026-07-21
Responsável (RACI-A): Arquiteto do Módulo 004-API
ADRs relacionados: ADR-0042, ADR-0044, ADR-0047
RFCs relacionados: RFC-0001
Depende de: ./_DESIGN_BRIEF.md §8, ./Deployment.md, ./Scalability.md
---

# 004-API — Chaves de Configuração

> Reproduz e detalha `./_DESIGN_BRIEF.md` §8. Escopo: `global` (cluster),
> `tenant`, `route`. "Recarregável" indica *hot reload* (sem *restart* de
> réplica). Prefixo de variável de ambiente: `AIOS_API_`.

## 1. Tabela consolidada

| Chave | Default | Faixa | Escopo | Recarregável | Descrição |
|-------|---------|-------|--------|--------------|-----------|
| `api.gateway.rate_limit.default_rps` | 100 | 1–100000 | tenant | sim | RPS default por tenant quando sem `RateLimitPolicy` explícita. |
| `api.gateway.rate_limit.default_burst` | 200 | 1–200000 | tenant | sim | Rajada default. |
| `api.gateway.max_body_size_bytes` | 10485760 | 1024–104857600 | global/route | sim | Tamanho máximo de corpo aceito (10 MiB default). |
| `api.gateway.upstream_timeout_ms` | 5000 | 100–60000 | global/route | sim | Timeout aguardando resposta do upstream. |
| `api.gateway.circuit_breaker.error_threshold` | 0.5 | 0–1 | global | sim | Taxa de erro que abre o *breaker* por upstream. |
| `api.gateway.circuit_breaker.min_requests` | 20 | 1–1000 | global | sim | Amostra mínima antes de avaliar abertura. |
| `api.version.coexistence_majors_min` | 2 | 1–5 | global | sim | Nº mínimo de majors coexistentes (RFC-0001 §5.7). |
| `api.version.deprecation_window_days` | 180 | 30–720 | global | sim | Janela mínima entre `Deprecated` e `Retired`. |
| `api.version.retirement_traffic_threshold` | 0.01 | 0–1 | global | sim | Fração máxima de tráfego residual para permitir T-05. |
| `api.auth.jwks_cache_ttl_s` | 300 | 30–3600 | global | sim | TTL do cache de chaves JWKS. |
| `api.auth.fail_mode` | `closed` | {closed, open} | global | sim | Comportamento se `021`/`022` indisponíveis. `closed`=nega. |
| `api.idempotency.ttl_hours` | 24 | 24–720 | global | sim | Retenção de resultados idempotentes (≥ 24h RFC-0001 §5.5). |
| `api.schema_validation.enabled` | true | bool | global/route | sim | Ativa/desativa `SchemaValidator` por rota. |
| `api.cors.allowed_origins` | `[]` | lista | tenant | sim | Origens CORS permitidas por tenant. |
| `api.sse.max_streams_per_replica` | 20000 | 100–200000 | global | sim | Limite de *streams* SSE concorrentes por réplica. |
| `api.sse.heartbeat_interval_s` | 15 | 5–120 | global | sim | Intervalo de *heartbeat* SSE. |
| `api.registry.contract_sync_interval_s` | 30 | 5–300 | global | sim | Frequência de re-agregação do `OpenApiAggregator`. |
| `api.registry.write_role` | `api-registry-admin` | string | global | **não** | Papel exigido para mutar registros de contrato. |

## 2. Variáveis de ambiente correspondentes

Toda chave de escopo `global` é sobrescrevível via variável de ambiente com
prefixo `AIOS_API_`, substituindo `.` por `_` e maiúsculas:

```
AIOS_API_GATEWAY__UPSTREAM_TIMEOUT_MS=5000
AIOS_API_AUTH__FAIL_MODE=closed
AIOS_API_VERSION__COEXISTENCE_MAJORS_MIN=2
AIOS_API_SSE__MAX_STREAMS_PER_REPLICA=20000
```

Chaves de escopo `tenant`/`route` **NÃO DEVEM** ser sobrescritas por
variável de ambiente do processo (afetariam todos os tenants/rotas da
réplica); elas são geridas via `RateLimitPolicy`/`RouteDefinition`
persistidas (`./Database.md`) e carregadas via *snapshot* em memória com
invalidação por evento.

## 3. Notas de operação por chave crítica

- **`api.auth.fail_mode=closed` (default)**: postura de segurança
  estrutural. Alterar para `open` **DEVE** ser uma decisão explícita e
  documentada — degrada a garantia de NFR-008 (0 *bypass* de PEP) e só é
  aceitável em ambientes de teste isolados, nunca em produção.
- **`api.gateway.circuit_breaker.error_threshold` /
  `.min_requests`**: ajustar exige avaliação conjunta com NFR-012
  (isolamento de falha); limiares muito baixos geram *flapping* do
  *breaker*, limiares muito altos atrasam a proteção do *bulkhead*.
- **`api.version.coexistence_majors_min`**: nunca reduzir abaixo de `2` sem
  nova RFC — é a implementação operacional direta da RFC-0001 §5.7.
- **`api.idempotency.ttl_hours`**: nunca configurar abaixo de `24` (mínimo
  normativo da RFC-0001 §5.5); a faixa começa em 24 propositalmente.
- **`api.sse.max_streams_per_replica`**: dimensionado junto a NFR-004
  (≥ 100.000 *streams* SSE concorrentes por réplica); reduzir este valor
  exige aumentar o número de réplicas para manter o SLO agregado.

## 4. Escopo e precedência

Quando uma chave existe em múltiplos escopos aplicáveis (ex.:
`api.gateway.max_body_size_bytes` em `global` e `route`), a precedência é:

```
route (RouteDefinition específica) > tenant (quando aplicável) > global (default do cluster)
```

Esta precedência é resolvida em tempo de requisição pelo componente
correspondente (`RateLimiter`, `SchemaValidator`) sem I/O adicional —
valores de escopo `route`/`tenant` são carregados no mesmo *snapshot* em
memória do `ContractRegistry` (ver `./Architecture.md` §5).

*Fim de `Configuration.md`.*
