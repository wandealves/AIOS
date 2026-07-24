---
Documento: FunctionalRequirements
Módulo: 024-Observability
Status: Draft
Versão: 0.1
Última atualização: 2026-07-23
Responsável (RACI-A): Arquiteto do Módulo 024-Observability
ADRs relacionados: ADR-0010, ADR-0241, ADR-0242, ADR-0243, ADR-0244, ADR-0245, ADR-0247, ADR-0249
RFCs relacionados: RFC-0001, RFC-0240, RFC-0241
Depende de: _DESIGN_BRIEF.md §7.1, 021-Security, 022-Policy, 025-Audit
---

# 024-Observability — Requisitos Funcionais

> Todos os requisitos abaixo derivam de `./_DESIGN_BRIEF.md` §7.1 e **NÃO PODEM**
> contradizê-lo. Prioridade em MoSCoW (Must / Should / Could). Cada FR é verificável
> por ao menos um caso de teste em `./Testing.md` e exercitado por ao menos um caso de
> uso em `./UseCases.md`.

## 1. Convenções

- Palavras normativas conforme RFC 2119 / RFC 8174.
- Códigos de erro seguem `AIOS-OBS-<NNNN>` (`./API.md` §6), formato da
  `../003-RFC/RFC-0001-Architecture-Baseline.md` §5.4.
- Nomes de sinal seguem `aios_<subsistema>_<nome>_<unidade>` (métrica) e
  `<componente>.<evento>` (log), conforme RFC-0240.

## 2. Tabela de requisitos funcionais

| ID | Requisito | Prioridade | Critério de aceite | Origem | UC | Teste |
|----|-----------|-----------|--------------------|--------|----|-------|
| FR-001 | O módulo **DEVE** ingerir métricas, traces e logs por OTLP (gRPC e HTTP) sob mTLS. | Must | Lote válido de cada tipo aceito e visível na consulta em ≤ 10 s. | Brief §7.1 | UC-001 | T-OBS-001 |
| FR-002 | O `TelemetryIngestGateway` **NÃO DEVE** aplicar backpressure ao emissor: sob saturação, **DEVE** descartar e contabilizar. | Must | Teste de saturação: latência do emissor inalterada; `aios_obs_dropped_total` reflete o descarte. | Brief §7.1; ADR-0243 | UC-001 | T-OBS-002 |
| FR-003 | O módulo **DEVE** admitir apenas sinais registrados, quarentenando o desconhecido **sem derrubar o lote**. | Must | Sinal não registrado → quarentena + `telemetry.signal.quarantined`; demais sinais do lote persistem. | Brief §7.1; ADR-0241 | UC-002, UC-003 | T-OBS-003 |
| FR-004 | O `SignalRegistry` **DEVE** validar nome, tipo, unidade e labels no registro. | Must | Nome fora de `aios_<subsistema>_<nome>_<unidade>` → `AIOS-OBS-0004`. | Brief §7.1 | UC-002 | T-OBS-004 |
| FR-005 | O `CardinalityGuard` **DEVE** impor orçamento de cardinalidade por tenant e por serviço. | Must | Estouro → ação configurada (`quarantine`/`drop_labels`/`reject`) + `telemetry.cardinality.exceeded`. | Brief §7.1; ADR-0242 | UC-003 | T-OBS-005 |
| FR-006 | O módulo **NÃO DEVE** aceitar label de identidade (`*_urn`, id de entidade) em métrica. | Must | `agent_urn` como label → `AIOS-OBS-0004` no registro; descartado em runtime. | Brief §7.1; ADR-0242 | UC-002, UC-003 | T-OBS-006 |
| FR-007 | O `PiiRedactor` **DEVE** redigir PII na borda, antes de qualquer persistência. | Must | Varredura do armazenamento não encontra PII em sinal `operational`. | Brief §7.1; ADR-0249 | UC-004 | T-OBS-007 |
| FR-008 | O `TraceSampler` **DEVE** aplicar política *tail-based*, retendo **100%** dos traces com erro e das caudas acima do limiar. | Must | Trace com `status=ERROR` ou acima de `obs.trace.tail_keep_slow_ms` sempre presente; demais na taxa configurada. | Brief §7.1; ADR-0244 | UC-005 | T-OBS-008 |
| FR-009 | O `ResourceEnricher` **DEVE** enriquecer todo sinal com atributos de recurso canônicos. | Must | 100% dos sinais com `service.name`, `aios.tenant`, `deployment.environment`. | Brief §7.1 | UC-001 | T-OBS-009 |
| FR-010 | O `SloEngine` **DEVE** calcular SLI e manter o ledger de error budget por SLO e janela. | Must | Ledger reproduz o SLI a partir dos sinais registrados; divergência = defeito. | Brief §7.1; ADR-0245 | UC-006, UC-007 | T-OBS-010 |
| FR-011 | O `AlertEvaluator` **DEVE** avaliar alertas com *multi-window multi-burn-rate* e histerese. | Must | Queima rápida dispara em ≤ 2 min; oscilação abaixo do limiar não dispara. | Brief §7.1; ADR-0245 | UC-007, UC-008 | T-OBS-011 |
| FR-012 | O módulo **DEVE** gerir o ciclo de vida do alerta (`./StateMachine.md`) com deduplicação por `fingerprint`. | Must | Nunca mais de uma instância aberta por `fingerprint` (invariante I1). | Brief §7.1 | UC-008..UC-011 | T-OBS-012 |
| FR-013 | Toda regra de alerta **DEVE** referenciar um runbook válido. | Must | Publicação sem `runbook_urn` → `AIOS-OBS-0008`. | Brief §7.1; ADR-0247 | UC-013 | T-OBS-013 |
| FR-014 | Silêncios **DEVEM** exigir prazo, justificativa e autor. | Must | Silêncio sem `expires_at` ou sem `reason` → `AIOS-OBS-0013`. | Brief §7.1 | UC-010 | T-OBS-014 |
| FR-015 | O módulo **DEVE** continuar avaliando alerta silenciado (silêncio suprime notificação, não observação). | Must | Alerta silenciado que resolve registra `Resolved` normalmente (invariante I3). | Brief §7.1 | UC-010, UC-011 | T-OBS-015 |
| FR-016 | O módulo **DEVE** tratar ausência de dado como `Expired`, **nunca** como `Resolved`. | Must | Alvo removido sem `cluster.node.decommissioned` → `telemetry.alert.nodata` (invariante I6). | Brief §7.1 | UC-011 | T-OBS-016 |
| FR-017 | O `QueryGateway` **DEVE** servir consulta unificada de métricas, traces e logs com isolamento por tenant. | Must | Consulta cross-tenant → `AIOS-OBS-0003`. | Brief §7.1 | UC-012 | T-OBS-017 |
| FR-018 | O módulo **DEVE** atuar como PEP nas consultas e nas operações administrativas. | Must | Sem capability → negado pelo PDP; decisão auditada em `../025-Audit/`. | Brief §7.1 | UC-012, UC-013 | T-OBS-018 |
| FR-019 | O `RetentionManager` **DEVE** aplicar retenção em camadas com *downsampling* programado. | Must | Métrica raw expurgada em 15 d; agregada de 5 min disponível por 90 d. | Brief §7.1; ADR-0248 | UC-014 | T-OBS-019 |
| FR-020 | O módulo **DEVERIA** publicar dashboards e runbooks como código versionado. | Should | Dashboard referenciando sinal não registrado é rejeitado. | Brief §7.1; ADR-0247 | UC-013 | T-OBS-020 |
| FR-021 | O `SelfTelemetry` **DEVE** expor saúde e perda do próprio pipeline por caminho independente. | Must | Com o pipeline saturado, `SelfTelemetry` continua respondendo. | Brief §7.1 | UC-001 | T-OBS-021 |
| FR-022 | O `EventEmitter` **DEVE** emitir todos os eventos de `./Events.md` via Outbox transacional. | Must | Nenhum evento perdido em teste de crash entre commit e publicação. | Brief §7.1 | UC-008 | T-OBS-022 |
| FR-023 | O módulo **DEVE** suportar o direito ao esquecimento sobre a telemetria. | Must | `security.rtbf.requested` → expurgo/pseudonimização comprovada e correlacionada em `../025-Audit/`. | Brief §7.1 | UC-015 | T-OBS-023 |
| FR-024 | O `DashboardRegistry` **DEVERIA** anotar rollouts e publicações de política nas linhas do tempo. | Should | Evento de rollout visível no dashboard em ≤ 30 s. | Brief §7.1 | UC-013 | T-OBS-024 |

## 3. Requisitos por agrupamento funcional

| Grupo | FRs | Documento de detalhamento |
|-------|-----|---------------------------|
| Ingestão e admissão | FR-001..FR-006, FR-009, FR-021 | `./API.md` §2, `./SequenceDiagrams.md` |
| Privacidade e amostragem | FR-007, FR-008, FR-023 | `./Security.md`, `./Configuration.md` |
| SLO e alerta | FR-010..FR-016, FR-022 | `./StateMachine.md`, `./Monitoring.md` |
| Consulta e autorização | FR-017, FR-018 | `./API.md` §3, `./Security.md` |
| Governança como código | FR-013, FR-020, FR-024 | `./Database.md`, `./Examples.md` |
| Retenção | FR-019 | `./Database.md` §11 |

## 4. Exemplo de verificação (FR-003 + FR-005)

Emissor envia, no mesmo lote, uma métrica registrada e uma desconhecida:

```json
POST /v1/metrics  (OTLP HTTP)
{ "resourceMetrics": [ { "resource": { "attributes": [
      {"key":"service.name","value":{"stringValue":"kernel"}},
      {"key":"aios.tenant","value":{"stringValue":"acme"}} ] },
    "scopeMetrics": [ { "metrics": [
      { "name": "aios_kernel_syscall_latency_ms", "histogram": { } },
      { "name": "kernel_temp_debug_counter",      "sum": { } } ] } ] } ] }
```

```
HTTP/1.1 200 OK          ← o lote NÃO é rejeitado (FR-003, fail-open)

Efeitos observáveis:
  · aios_kernel_syscall_latency_ms  → persistida normalmente
  · kernel_temp_debug_counter       → quarentenada
      + aios_obs_signal_quarantined_total{reason="unregistered"} +1
      + evento aios._platform.telemetry.signal.quarantined
```

Um sinal desconhecido **não** invalida os demais do lote: rejeitar o lote inteiro
faria uma métrica de depuração esquecida em um serviço apagar toda a telemetria dele.

## 5. Referências

- Fonte: `./_DESIGN_BRIEF.md` §7.1
- Rastreabilidade: `./Requirements.md`
- Não-funcionais: `./NonFunctionalRequirements.md`
- Casos de uso: `./UseCases.md` · Testes: `./Testing.md`

*Fim de `FunctionalRequirements.md`.*
