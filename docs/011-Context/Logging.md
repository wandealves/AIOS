---
Documento: Logging
Módulo: 011-Context
Status: Draft
Versão: 0.1
Última atualização: 2026-07-21
Responsável (RACI-A): Arquiteto do Módulo 011-Context
ADRs relacionados: ADR-0010, ADR-0111, ADR-0116, ADR-0118, ADR-0119
RFCs relacionados: RFC-0001 (baseline), RFC-0011 (proposta)
Depende de: 003-RFC (RFC-0001), 021-Security, 024-Observability, 025-Audit, 040-Glossary
---

# AIOS — Módulo 011 · Context — Logging

> Este documento especifica o **log estruturado** do `ContextService`:
> eventos de log, níveis, campos obrigatórios, correlação e retenção. A
> plataforma de coleta (**Serilog → Seq**, agregação, *sinks*, política de
> retenção da plataforma) pertence a
> [024-Observability](../024-Observability/) — aqui apenas configuramos o que
> é **específico do domínio de janela de contexto e cache semântico**, sem
> redefinir a plataforma. Correlação (`trace_id`/`span_id`/`traceparent`),
> identidade (URN, `tenant_id`) e o envelope de erro RFC 7807 são reutilizados
> de [RFC-0001 §5.6/§6/§7](../003-RFC/RFC-0001-Architecture-Baseline.md).
> Palavras normativas conforme RFC 2119/8174.

---

## 1. Objetivo e princípios

O log estruturado do `011-Context` DEVE permitir, sem acessar o conteúdo
textual de prompts/fragmentos:

1. Reconstruir o caminho de decisão de um `bundle_id` específico
   (`RECEIVED → … → SERVED/REJECTED/FAILED`, [StateMachine.md](./StateMachine.md)).
2. Diagnosticar violação de SLO (latência, compressão, cache hit) sem
   depender exclusivamente de métricas agregadas.
3. Provar, por amostragem, que **nenhum dado sensível** (texto de prompt,
   fragmento, resposta cacheada) trafega em log — apenas identificadores,
   contadores e *scores* (§5).
4. Correlacionar 1:1 com traces OTel ([Monitoring.md](./Monitoring.md)) e com
   eventos de domínio ([Events.md](./Events.md)) através de `trace_id`/
   `event.id`/`bundle_id`/`cache_id`.

> **Princípio normativo:** um log do `011` **NUNCA** é a fonte de verdade de
> auditoria (essa é [025-Audit](../025-Audit/Events.md), brief §1.3 N-08) nem
> de métricas de negócio (essas são [Metrics.md](./Metrics.md)) — o log é o
> elo de **diagnóstico textual** entre os dois.

---

## 2. Níveis de log e semântica no domínio de Context

| Nível | Quando usar no `011` | Exemplo |
|-------|------------------------|---------|
| `Trace` | Detalhe de pipeline interno (por estágio do `ContextAssembler`), habilitado apenas sob *feature flag* de diagnóstico. | Entrada/saída de cada estágio `budget→retrieve→rank→dedup→compress`. |
| `Debug` | Decisões intermediárias não críticas (scores de ranking, candidatos descartados por `top_k`). | `RelevanceRanker` descartou N candidatos abaixo do orçamento remanescente. |
| `Information` | Toda transição terminal de `ContextBundle`/`SemanticCacheEntry` e toda operação de API concluída. | `assemble` concluído com `status=SERVED`, `cache_outcome=MISS`. |
| `Warning` | Degradação, fallback ou condição recuperável acionada. | `DEGRADED` acionado por timeout do `010`; fallback `truncate` por falha de sumarização (`AIOS-CTX-0004`). |
| `Error` | Falha de operação retornada ao chamador (retriable ou não), exceto rejeições determinísticas esperadas. | `AIOS-CTX-0002` sem limites cacheados; falha de persistência do `BundleStore`. |
| `Fatal` | Falha que impede o processo de operar (não a operação isolada). | Falha de *bootstrap* (migração pendente, `pgvector` ausente). |

Regras:

- `REJECTED` por `AIOS-CTX-0001`/`AIOS-CTX-0011`/`AIOS-CTX-0012` (rejeições
  **determinísticas** de contrato) **DEVEM** ser logadas em `Information`
  (são comportamento esperado do sistema, não falha operacional) — apenas
  contabilizadas como erro em `aios_context_errors_total`
  ([Metrics.md](./Metrics.md)), nunca poluindo o canal de `Error`.
- `Warning`/`Error` **DEVEM** incluir `code` (`AIOS-CTX-<NNNN>`) e
  `retriable` como campos estruturados, nunca apenas na mensagem livre.

---

## 3. Schema de campos obrigatórios

Todo evento de log estruturado (JSON, um objeto por linha — compatível com o
*sink* Seq via `024`) **DEVE** conter os campos abaixo quando aplicável ao
contexto da operação:

| Campo | Tipo | Obrigatório | Descrição |
|-------|------|:---:|-----------|
| `@t` (timestamp) | `datetime` (UTC, ISO 8601) | sim | Instante de emissão. |
| `@l` (level) | enum (§2) | sim | Nível do evento. |
| `@m` (message) | string (template) | sim | Mensagem com *placeholders* nomeados (Serilog message templates), nunca concatenação de string. |
| `service` | `"aios-context"` | sim | Nome do serviço (`resource.attributes.service.name`, RFC-0001 §5.6). |
| `service_version` | string | sim | Versão da imagem em execução. |
| `tenant_id` | string | sim | Fronteira de isolamento — RLS/namespace (RFC-0001 §5.1). |
| `trace_id` | string (hex 32) | sim | Extraído de `traceparent` (RFC-0001 §5.6). |
| `span_id` | string (hex 16) | sim | Span OTel corrente. |
| `bundle_id` | ULID | quando aplicável | `ContextBundle` em curso. |
| `cache_id` | ULID | quando aplicável | `SemanticCacheEntry` em curso. |
| `agent_urn` | URN | quando aplicável | Agente solicitante. |
| `component` | `PascalCase` | sim | Componente emissor (`ContextAssembler`, `TokenBudgeter`, `SemanticCacheManager`, …, [Architecture.md](./Architecture.md) §2). |
| `operation` | string | sim | `assemble`\|`compress`\|`cache_lookup`\|`cache_store`\|`cache_invalidate`\|`tokens_estimate`\|`budget_upsert`. |
| `outcome` | string | quando terminal | `served`\|`rejected`\|`failed`\|`degraded`\|`hit`\|`miss`\|`bypass`. |
| `code` | `AIOS-CTX-NNNN` | em erro/aviso | Código canônico ([API.md](./API.md) §5.3). |
| `retriable` | bool | em erro | Espelha a classificação de [FailureRecovery.md](./FailureRecovery.md) §3.2. |
| `duration_ms` | number | quando aplicável | Duração da operação/estágio (espelha o histograma correspondente, [Metrics.md](./Metrics.md)). |
| `idempotency_key` | string | em mutação | Correlação de replays (RFC-0001 §5.5). |
| `exception` | objeto | em `Error`/`Fatal` | *Stack trace* estruturado (sem PII — ver §5). |

> **Nunca** incluir como campo estruturado: `content_inline`, `prompt`,
> `response_ref` (conteúdo), `prompt_embedding`/`embedding` (vetor bruto),
> ou qualquer texto de fragmento/instrução do usuário (§5).

---

## 4. Eventos de log por operação (catálogo)

Cada linha da tabela corresponde a um evento de log emitido pelo componente
indicado, com o `message template` canônico (Serilog).

| # | Operação/estágio | Nível | Componente | Template |
|---|---|---|---|---|
| L-01 | Início de `assemble` | `Information` | `ContextApiGateway` | `"Assemble started bundle={BundleId} agent={AgentUrn} model={ModelId}"` |
| L-02 | Fim de `assemble` (`SERVED`) | `Information` | `ContextAssembler` | `"Assemble served bundle={BundleId} outcome={Outcome} tokensIn={TokensIn} budget={TokenBudget} ratio={CompressionRatio} durationMs={DurationMs}"` |
| L-03 | Rejeição determinística | `Information` | `ContextAssembler` | `"Assemble rejected bundle={BundleId} code={Code} reason={Reason}"` |
| L-04 | `CacheLookup` resultado | `Information` | `SemanticCacheManager` | `"Cache lookup outcome={Outcome} backend={Backend} similarity={Similarity} durationMs={DurationMs}"` |
| L-05 | Compressão aplicada | `Information` | `HierarchicalCompressor` | `"Compression applied bundle={BundleId} method={Method} tokensBefore={TokensBefore} tokensAfter={TokensAfter}"` |
| L-06 | Fallback de truncamento | `Warning` | `HierarchicalCompressor` | `"Summarization failed, truncate fallback bundle={BundleId} code={Code}"` |
| L-07 | Degradação ativada | `Warning` | `ContextAssembler` | `"Degraded mode bundle={BundleId} cause={Cause} code={Code}"` |
| L-08 | Circuit breaker aberto | `Warning` | `ModelRouterClient`/`MemoryClient` | `"Circuit breaker opened dependency={Dependency} failureRate={FailureRate}"` |
| L-09 | Falha não-retriable | `Error` | `ContextApiGateway` | `"Request failed operation={Operation} code={Code} retriable=false"` |
| L-10 | Falha retriable esgotada | `Error` | (cliente do dependente) | `"Retries exhausted dependency={Dependency} attempts={Attempts} code={Code}"` |
| L-11 | Eviction de cache | `Information` | `EvictionManager` | `"Cache entry evicted cacheId={CacheId} cause={Cause}"` |
| L-12 | Invalidação event-driven | `Information` | `SemanticCacheManager` | `"Cache invalidated cacheId={CacheId} sourceUrn={SourceUrn} latencyMs={LatencyMs}"` |
| L-13 | Publicação de evento (outbox) | `Debug` | `EventPublisher` | `"Event published subject={Subject} eventId={EventId}"` |
| L-14 | Lag de outbox detectado | `Warning` | `EventPublisher` | `"Outbox pending threshold exceeded pending={PendingCount}"` |
| L-15 | Conflito de idempotência | `Information` | `ContextApiGateway` | `"Idempotency conflict key={IdempotencyKey} operation={Operation}"` |
| L-16 | Expurgo LGPD/RTBF | `Information` | `BundleStore`/`SemanticCacheManager` | `"RTBF purge completed sourceUrn={SourceUrn} entriesEvicted={Count}"` |
| L-17 | Autorização negada (PEP) | `Warning` | `ContextPolicyGuard` | `"Authorization denied operation={Operation} reason={Reason}"` |
| L-18 | Falha de *bootstrap* | `Fatal` | processo | `"Bootstrap failed reason={Reason}"` |

### 4.1 Exemplo — linha de log JSON (L-02, sucesso)

```json
{
  "@t": "2026-07-21T14:02:11.348Z",
  "@l": "Information",
  "@m": "Assemble served bundle=01J9ZC0Q7S9U2W4Y6A8C0E2G4J outcome=served tokensIn=8990 budget=12000 ratio=0.5199 durationMs=132.4",
  "service": "aios-context",
  "service_version": "1.0.0+a1b2c3d",
  "tenant_id": "acme",
  "trace_id": "4bf92f3577b34da6a3ce929d0e0e4736",
  "span_id": "00f067aa0ba902b7",
  "bundle_id": "01J9ZC0Q7S9U2W4Y6A8C0E2G4J",
  "agent_urn": "urn:aios:acme:agent:01J9Z8Q6H7K2M4N6P8R0S2T4V6",
  "component": "ContextAssembler",
  "operation": "assemble",
  "outcome": "served",
  "duration_ms": 132.4,
  "idempotency_key": "01J9ZC0P5R7T2V4X6Z8B0D2F4H"
}
```

### 4.2 Exemplo — linha de log JSON (L-07, degradação)

```json
{
  "@t": "2026-07-21T14:05:03.912Z",
  "@l": "Warning",
  "@m": "Degraded mode bundle=01J9ZC1F... cause=memory_timeout code=AIOS-CTX-0005",
  "service": "aios-context",
  "tenant_id": "acme",
  "trace_id": "7cf03a4488c45eb7b4df030e1f1f5847",
  "span_id": "11a178bb1ca913c8",
  "bundle_id": "01J9ZC1F1234567890ABCDEF01",
  "component": "ContextAssembler",
  "operation": "assemble",
  "outcome": "degraded",
  "code": "AIOS-CTX-0005",
  "retriable": true,
  "duration_ms": 118.6
}
```

---

## 5. Minimização de dados e PII (redação estrutural)

Derivado de [Security.md](./Security.md) §3/§4 e RFC-0001 §6/§7 — aplicado
aqui ao subsistema de logging:

- Logs **NÃO DEVEM** conter `content_inline`, texto de `system`/
  `instruction`/`history` fornecido pelo agente, texto de `response_ref`
  cacheado, nem o vetor bruto de `embedding`/`prompt_embedding`.
- `message template` **DEVE** usar apenas *placeholders* que resolvem para
  identificadores (URN/ULID), contadores, *scores*, códigos e durações — nunca
  para o valor de `content_inline`/`response_ref`. Isso é aplicado por
  *policy* de *code review* e por um *scrubber* de log (`ForbiddenFields`)
  configurado no *sink* Serilog, que redige (`"[REDACTED]"`) qualquer campo
  fora da lista branca da §3 antes de sair do processo.
- Envelopes de exceção (`exception`) **DEVEM** passar pelo mesmo *scrubber*
  antes de serialização — mensagens de exceção de bibliotecas de terceiros
  PODEM conter fragmentos de payload por acidente (ex.: erro de
  desserialização); o *scrubber* trunca `Message`/`Data` de exceção a um
  tamanho máximo e remove padrões que se assemelhem a PII (e-mail, CPF,
  cartão) por *regex* de defesa em profundidade — controle complementar, não
  substituto da regra estrutural acima.
- Verificação: *scan* periódico automatizado dos últimos N dias de log
  (amostragem) procurando por padrões de PII e por campos fora da lista
  branca da §3 — ver [Testing.md](./Testing.md) (`test_logging_no_pii_leak`).

```
 Serilog pipeline (ContextService)
        │
   ┌────▼───────────────┐
   │ Enrichers           │  tenant_id, trace_id, span_id, service, service_version
   └────┬───────────────┘
   ┌────▼───────────────┐
   │ ForbiddenFields      │  remove/redige content_inline, response_ref, embedding, prompt*
   │ Scrubber             │  trunca + regex de defesa em profundidade em exception.Message
   └────┬───────────────┘
   ┌────▼───────────────┐
   │ Sink: Seq (024)      │  + arquivo local rotativo (buffer de contingência)
   └─────────────────────┘
```

---

## 6. Correlação com traces, métricas e eventos

- `trace_id`/`span_id` **DEVEM** ser idênticos aos anexados como *exemplar*
  nos histogramas de [Metrics.md](./Metrics.md) §1 e ao `traceparent`
  propagado nas chamadas a `010`/`017`/`022` (RFC-0001 §5.6) — um operador
  DEVE poder saltar de um log de erro (`Error`, `code=AIOS-CTX-0002`)
  diretamente ao trace correspondente no painel de
  [024-Observability](../024-Observability/).
- `bundle_id`/`cache_id` correlacionam log ↔ evento de domínio
  ([Events.md](./Events.md)): o log L-02 (`Assemble served`) e o evento
  `context.window.assembled` compartilham `bundle_id` e `trace_id` — nenhum
  dos dois substitui o outro (log é diagnóstico textual; evento é contrato
  de integração; ver princípio §1).
- `bundle_id`/`cache_id`/`event.id` **NÃO SÃO** usados como *label* de
  métrica Prometheus (explosão de cardinalidade, [Metrics.md](./Metrics.md)
  §1) — a correlação de alta cardinalidade vive **apenas** em logs/traces.

---

## 7. Volume, amostragem e retenção

| Aspecto | Regra |
|---------|-------|
| Nível mínimo em produção | `Information` (golden path completo); `Trace`/`Debug` habilitados por *feature flag* de diagnóstico, com TTL automático da flag (≤ 24h) para evitar esquecimento ligado. |
| Amostragem de `Debug` | Amostrado (ex.: 1 em N requisições) mesmo quando a *flag* está ativa, para não saturar o `024-Observability` sob carga (`context.telemetry.*`, [Configuration.md](./Configuration.md) §5.9 — mesmo mecanismo de amostragem usado para `context.window.cache_miss`). |
| Retenção — quente (consultável, Seq) | 30 dias, alinhada à política padrão de [024-Observability](../024-Observability/); nenhuma exceção específica de retenção estendida para `011` (dados são efêmeros por natureza — brief §1.3). |
| Retenção — fria (arquivo, se aplicável) | Conforme política global de `024`; o `011-Context` não define retenção própria (evita duplicar governança). |
| Correlação com RTBF | Logs que referenciam um `source_urn` expurgado (FR-012) **NÃO** são reprocessados retroativamente (já não continham PII, apenas identificadores) — a obrigação de expurgo recai sobre dados duráveis (PostgreSQL/MinIO/cache), não sobre o log histórico, que é estritamente identificador+métrica. |
| Volume esperado | Dominado por L-01/L-02/L-04 em `Information` — ordem de grandeza de 1 evento por operação de API concluída; picos absorvidos pelo *buffering* assíncrono do *sink* Serilog. |

---

## 8. Exemplo — configuração Serilog (ilustrativa)

```csharp
// Program.cs — configuração ilustrativa de logging estruturado
Log.Logger = new LoggerConfiguration()
    .MinimumLevel.Information()
    .Enrich.WithProperty("service", "aios-context")
    .Enrich.WithProperty("service_version", ThisAssembly.InformationalVersion)
    .Enrich.FromLogContext() // tenant_id, trace_id, span_id via middleware de correlação (RFC-0001 §5.6)
    .Filter.With(new ForbiddenFieldsScrubber(
        forbidden: new[] { "content_inline", "response_ref", "embedding", "prompt_embedding", "prompt" }))
    .WriteTo.Seq(seqServerUrl, restrictedToMinimumLevel: LogEventLevel.Information)
    .WriteTo.Console(new CompactJsonFormatter()) // stdout, coletado por 024
    .CreateLogger();

// Exemplo de emissão (L-02)
_logger.LogInformation(
    "Assemble served bundle={BundleId} outcome={Outcome} tokensIn={TokensIn} budget={TokenBudget} ratio={CompressionRatio} durationMs={DurationMs}",
    bundle.BundleId, "served", bundle.TokensIn, bundle.TokenBudget, bundle.CompressionRatio, sw.Elapsed.TotalMilliseconds);
```

---

## 9. Riscos e alternativas

| Risco / decisão | Mitigação / alternativa |
|-------------------|--------------------------|
| Vazamento acidental de fragmento/prompt via mensagem de exceção de terceiros. | `ForbiddenFieldsScrubber` trunca e aplica regex de defesa em profundidade sobre `exception.Message`/`Data` (§5); *code review* proíbe *placeholders* fora da lista branca. |
| Volume de log em `Debug`/`Trace` satura o `024-Observability` sob incidente prolongado. | Amostragem obrigatória (§7) + TTL automático de *feature flag* de diagnóstico. |
| Rejeições determinísticas esperadas (`AIOS-CTX-0001`/`0011`/`0012`) poluindo alertas de erro. | Logadas em `Information`, não em `Error` (§2); contabilizadas apenas como métrica rotulada por `code` ([Metrics.md](./Metrics.md)). |
| Operador confunde log com trilha de auditoria formal. | Princípio explícito (§1): log nunca é fonte de verdade de auditoria; toda operação privilegiada tem registro dedicado em [025-Audit](../025-Audit/Events.md). |
| **Alternativa descartada:** logar o payload completo da requisição para facilitar depuração. | Rejeitada — viola minimização de dados (RFC-0001 §7, brief §12.3); a combinação `bundle_id`+trace OTel já permite reconstrução do caminho de decisão sem reter conteúdo. |
| **Alternativa descartada:** nível `Debug` como padrão em produção. | Rejeitada — volume/custo de observabilidade desproporcional; `Information` já cobre 100% das transições terminais exigidas por auditoria/diagnóstico (FR-009). |

---

## 10. Ver também

- [Monitoring.md](./Monitoring.md) — correlação de alertas ↔ *runbooks* ↔ logs.
- [Metrics.md](./Metrics.md) — métricas e *exemplars* correlacionados por `trace_id`.
- [Events.md](./Events.md) — eventos de domínio correlacionados por `bundle_id`/`cache_id`.
- [Security.md](./Security.md) — minimização de PII e gestão de segredos.
- [FailureRecovery.md](./FailureRecovery.md) — classificação `retriable`/códigos de erro.
- [Configuration.md](./Configuration.md) — `context.telemetry.*` (amostragem).
- [Testing.md](./Testing.md) — `test_logging_no_pii_leak`, cobertura de eventos de log.
- Glossário: [Correlation ID, PII, Idempotency](../040-Glossary/Glossary.md).
