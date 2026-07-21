---
Documento: Logging
Módulo: 010-Memory
Status: Draft
Versão: 0.1
Última atualização: 2026-07-20
Responsável (RACI-A): Arquiteto do Módulo 010-Memory
ADRs relacionados: ADR-0102, ADR-0105, ADR-0109
RFCs relacionados: RFC-0001, RFC-0100
Depende de: 001-Architecture, 003-RFC (RFC-0001), 021-Security, 024-Observability, 025-Audit, 040-Glossary
---

# 010-Memory — Logging (Log Estruturado)

Este documento define o **log estruturado** do `MemoryService`, emitido via **Serilog**
e enviado ao **Seq** (agregador central de [024-Observability](../024-Observability/)).
Todo log DEVE ser estruturado (JSON), correlacionável por `trace_id`/`span_id`/`tenant_id`
(RFC-0001 §5.6) e respeitar a minimização de PII (RFC-0001 §7). Log **não** é auditoria:
a trilha imutável pertence a [025-Audit](../025-Audit/) (N6 do brief).

Palavras normativas conforme RFC 2119/8174. Termos reutilizam
[Glossary](../040-Glossary/Glossary.md); a correlação reutiliza
[RFC-0001 §5.6](../003-RFC/RFC-0001-Architecture-Baseline.md) — **não redefinidos aqui**.

---

## 1. Princípios

1. **Estruturado sempre.** O sink de produção DEVE ser JSON (Serilog
   `CompactJsonFormatter`); mensagens usam *message templates* com propriedades nomeadas,
   nunca interpolação de string.
2. **Correlação obrigatória.** Todo evento de log DEVE conter `trace_id`, `span_id` e
   `tenant_id` (quando houver contexto). Eles vêm do `Activity`/`traceparent` (OTel).
3. **Sem PII crua.** Conteúdo de `MemoryItem` (`content`), embeddings, e valores marcados
   `pii=true` NÃO DEVEM aparecer em log — apenas identificadores, hashes e metadados.
4. **Log ≠ evento de domínio ≠ auditoria.** Logs servem diagnóstico; eventos de domínio
   (§6 do brief) integram módulos; auditoria (025) é a verdade legal. Uma ação pode gerar
   os três, mas cada um com propósito distinto.
5. **Amostragem no caminho quente.** Logs `Debug`/`Verbose` do caminho de `recall`/`remember`
   DEVERIAM ser amostrados para não dominar o volume (§5).

---

## 2. Níveis de log (mapeamento Serilog)

| Nível Serilog | Uso no MemoryService | Exemplos |
|---------------|----------------------|----------|
| `Verbose` | Passo-a-passo de fusão/re-rank, scores intermediários. Amostrado (< 1%). | Contribuição RRF por fonte no recall. |
| `Debug` | Decisões de roteamento, escolha inline vs. blob, cache de embedding. Amostrado. | `LayerRouter` escolheu `semantic`. |
| `Information` | Ciclo de vida normal de operações e jobs. | `remember` concluído, `ConsolidationJob` iniciado. |
| `Warning` | Degradação graciosa e condições recuperáveis. | Grafo indisponível → fallback hybrid (F5); backpressure ativo (F7). |
| `Error` | Falha de operação/job com impacto ao chamador. | `AIOS-MEM-0030` (Model Router falhou); rollback de consolidação (F6). |
| `Fatal` | Falha que impede o serviço de operar. | Perda de conexão a todos os backends duráveis. |

Nível padrão de produção: `Information`. `Debug`/`Verbose` habilitáveis por tenant/escopo
via *log level override* dinâmico (sem redeploy).

---

## 3. Campos obrigatórios (schema de log)

Todo registro de log DEVE conter os campos **base**; operações e jobs acrescentam os campos
de contexto. Campos seguem `snake_case`.

### 3.1 Campos base (todo log)

| Campo | Tipo | Origem | Obrigatório | Descrição |
|-------|------|--------|-------------|-----------|
| `timestamp` | RFC 3339 (UTC) | Serilog | DEVE | Instante do evento. |
| `level` | string | Serilog | DEVE | `Verbose..Fatal`. |
| `message` | string | template | DEVE | Mensagem renderizada. |
| `message_template` | string | Serilog | DEVE | Template original (agrupamento no Seq). |
| `service_name` | string | resource | DEVE | `aios-memory`. |
| `service_version` | string | resource | DEVE | Versão do build. |
| `environment` | string | resource | DEVE | `dev\|staging\|prod`. |
| `trace_id` | hex(32) | OTel `Activity` | DEVE¹ | Correlação W3C (RFC-0001 §5.6). |
| `span_id` | hex(16) | OTel `Activity` | DEVE¹ | Span corrente. |
| `tenant_id` | string(slug) | contexto autenticado | DEVE² | Fronteira de isolamento. |

¹ DEVE quando há `Activity` ativa (todo caminho de request/consumo).
² DEVE quando há contexto de tenant (ausente apenas em logs de bootstrap).

### 3.2 Campos de contexto de operação

| Campo | Aplicável a | Descrição |
|-------|-------------|-----------|
| `operation` | toda API | `remember\|recall\|consolidate\|forget\|purge\|get_item\|get_stats`. |
| `agent_id` | ops com agente | ULID do agente (identificador, nunca conteúdo). |
| `session_id` | Working/Short-Term | ULID da sessão. |
| `memory_item_id` | ops por item | ULID/URN do item (`urn:aios:<t>:memory:<id>`). |
| `layer` | remember/recall | camada física (`working..kg`). |
| `kind` | remember | `fact\|event\|skill\|...`. |
| `mode` | recall | `vector\|lexical\|graph\|hybrid`. |
| `result` | toda API | `ok\|error`. |
| `error_code` | falhas | `AIOS-MEM-NNNN` (ver [API.md](./API.md)). |
| `retriable` | falhas | bool (RFC-0001 §5.4). |
| `duration_ms` | toda API | Latência da operação (espelha [Metrics.md](./Metrics.md)). |
| `idempotency_key` | mutações | Chave (hash, não valor cru se sensível). |
| `job_id` | consolidação | ULID do `ConsolidationJob`. |
| `version_id` | consolidação | `ConsolidationVersion` afetada. |
| `content_hash` | remember/blob | SHA-256 (permite correlação sem expor conteúdo). |
| `quota_dimension` | cota | `items\|bytes\|vectors`. |
| `legal_basis` | RTBF/retention | base legal (LGPD/GDPR) — metadado, nunca o dado pessoal. |

---

## 4. Regras de privacidade e redação (LGPD/GDPR)

| Regra | Norma |
|-------|-------|
| `content` de `MemoryItem` | NÃO DEVE ser logado. Registrar apenas `content_hash` e tamanho. |
| Vetores/embeddings | NÃO DEVEM ser logados (sem valor diagnóstico, alto volume). |
| Campos `pii=true` | DEVEM ser redigidos/tokenizados antes de qualquer log. |
| `idempotency_key` sensível | DEVERIA ser registrada como hash. |
| Erros | `detail` do envelope RFC 7807 NÃO DEVE conter dado sensível (RFC-0001 §6). |
| RTBF | O expurgo NÃO remove logs de diagnóstico já minimizados; a prova legal do expurgo é o evento `memory.item.purged` auditado por 025, não o log. |

Um *Serilog Enricher* dedicado (`PiiRedactionEnricher`) DEVE aplicar redação antes do sink.

---

## 5. Retenção e roteamento

| Nível | Sink primário | Retenção | Amostragem |
|-------|---------------|----------|------------|
| `Information`+ | Seq (quente) | **30 dias** | integral |
| `Warning`/`Error`/`Fatal` | Seq + arquivamento frio (MinIO/S3) | **180 dias** | integral |
| `Debug`/`Verbose` | Seq (quente), habilitado sob demanda | **7 dias** | ≤ 1% no caminho quente |
| Logs correlacionados a auditoria | — | ver [025-Audit](../025-Audit/) | — |

Retenção de log é **independente** da retenção de `MemoryItem` (§8 do brief) e da trilha de
auditoria. A rotação/arquivamento pertence a [024-Observability](../024-Observability/).

---

## 6. Exemplos

### 6.1 `recall` bem-sucedido (Information, JSON compacto)

```json
{
  "timestamp": "2026-07-20T12:34:56.789Z",
  "level": "Information",
  "message_template": "recall {Mode} concluído para {TenantId}: {ResultCount} itens em {DurationMs}ms",
  "message": "recall hybrid concluído para acme: 8 itens em 63ms",
  "service_name": "aios-memory",
  "service_version": "0.1.0",
  "environment": "prod",
  "trace_id": "4bf92f3577b34da6a3ce929d0e0e4736",
  "span_id": "00f067aa0ba902b7",
  "tenant_id": "acme",
  "operation": "recall",
  "agent_id": "01J9Z8Q6H7K2M4N6P8R0S2T4V6",
  "mode": "hybrid",
  "result": "ok",
  "result_count": 8,
  "duration_ms": 63
}
```

### 6.2 Falha de embedding (Error, F3 / `AIOS-MEM-0030`)

```json
{
  "timestamp": "2026-07-20T12:35:10.101Z",
  "level": "Error",
  "message_template": "remember para {TenantId} enfileirado sem embedding: Model Router indisponível ({ErrorCode})",
  "message": "remember para acme enfileirado sem embedding: Model Router indisponível (AIOS-MEM-0030)",
  "service_name": "aios-memory",
  "trace_id": "1a2b3c...", "span_id": "0af1...", "tenant_id": "acme",
  "operation": "remember", "layer": "semantic", "kind": "fact",
  "memory_item_id": "urn:aios:acme:memory:01J9ZB...",
  "content_hash": "sha256:9f86d0...", "result": "error",
  "error_code": "AIOS-MEM-0030", "retriable": true, "duration_ms": 118
}
```

### 6.3 Configuração Serilog (.NET 10)

```csharp
Log.Logger = new LoggerConfiguration()
    .MinimumLevel.Information()
    .Enrich.FromLogContext()
    .Enrich.WithProperty("service_name", "aios-memory")
    .Enrich.With<TraceContextEnricher>()   // trace_id/span_id do Activity (OTel)
    .Enrich.With<TenantEnricher>()         // tenant_id do contexto autenticado
    .Enrich.With<PiiRedactionEnricher>()   // redação LGPD/GDPR (§4)
    .WriteTo.Seq(serverUrl: cfg["seq:url"], apiKey: cfg["seq:apiKey"],
                 controlLevelSwitch: _levelSwitch)   // override dinâmico por tenant
    .CreateLogger();

// Uso — sempre message template com propriedades nomeadas:
_log.Information("recall {Mode} concluído para {TenantId}: {ResultCount} itens em {DurationMs}ms",
                 mode, tenantId, results.Count, sw.ElapsedMilliseconds);
```

---

## 7. Correlação entre pilares de observabilidade

```
 Request ──▶ [Activity/traceparent]  RFC-0001 §5.6
             │
    ┌────────┼─────────────────┬──────────────────────┐
    ▼        ▼                 ▼                      ▼
  Trace    Métrica          Log (Serilog→Seq)     Evento de domínio
 (OTel)  (aios_memory_*)    trace_id/span_id/       (aios.<t>.memory.*)
                             tenant_id                     │
                                                           ▼
                                                    Auditoria (025)
```

`trace_id` é a **chave de junção**: do painel ([Monitoring.md](./Monitoring.md)) ao trace,
do trace aos logs no Seq, e do log ao evento/auditoria. Todos compartilham `tenant_id`.

## 8. Riscos e alternativas

- **Volume no caminho quente.** `recall`/`remember` em alta vazão (NFR-004) podem inundar o
  Seq. Mitigação: amostragem de `Debug`/`Verbose` (§5) e `Information` conciso.
- **Vazamento de PII por template mal escrito.** Mitigação: revisão de *message templates*
  no gate de CI e `PiiRedactionEnricher` como rede de segurança.
- **Alternativa descartada:** logs em texto livre não estruturado — rejeitada por impedir
  correlação e busca no Seq (viola RFC-0001 §5.8).

Ver também: [Metrics.md](./Metrics.md), [Monitoring.md](./Monitoring.md),
[Security.md](./Security.md).
