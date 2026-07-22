---
Documento: NonFunctionalRequirements
Módulo: 007-Agent-Runtime
Status: Draft
Versão: 0.1
Última atualização: 2026-07-21
Responsável (RACI-A): Arquiteto do Módulo 007 — Agent Runtime
ADRs relacionados: ADR-0070, ADR-0071, ADR-0072, ADR-0073, ADR-0074, ADR-0076, ADR-0077, ADR-0079
RFCs relacionados: RFC-0001, RFC-0070, RFC-0071
Depende de: `_DESIGN_BRIEF.md` (deste módulo), `Requirements.md`, `../024-Observability/`, `../021-Security/`
---

# 007-Agent-Runtime — Non-Functional Requirements

> Toda meta abaixo é a **mesma** definida no `_DESIGN_BRIEF.md` §7.2 (para
> `NFR-001`..`NFR-012`) ou uma elaboração não-contraditória derivada das
> demais seções do brief (para `NFR-013`..`NFR-016`, já anunciadas no sumário
> de `Requirements.md` §4). Nenhuma meta numérica aqui diverge do brief. SLI =
> *Service Level Indicator* (o que é medido); SLO = *Service Level Objective*
> (a meta); método de verificação referencia `Testing.md`/`Benchmark.md`/
> `Monitoring.md`.

## 1. Desempenho

| ID | Atributo | Meta (SLO) | SLI / Método de verificação |
|----|----------|-----------|------------------------------|
| NFR-001 | Cold start | p99 de boot→ready **≤ 250 ms** com pool quente; p99 **≤ 900 ms** sem pool quente. | Histograma `aios_runtime_cold_start_ms` (p99); carga sintética com `runtime.pool.warm_size=0` vs. `>0` em `Benchmark.md`. |
| NFR-002 | Overhead de sandbox | Overhead de CPU do sandbox **≤ 5%**; overhead de latência por passo **≤ 8 ms**, ambos comparados a execução sem `seccomp`/`cgroups`/namespaces. | `aios_runtime_sandbox_overhead_ratio`; comparação A/B com baseline instrumentada em `Benchmark.md`. |
| NFR-003 | Latência de passo do loop | p95 de latência de passo (excluindo tempo de LLM/tool) **≤ 15 ms**. | `aios_runtime_step_latency_ms` (p95), excluindo `latency_ms` atribuído a `ModelCall`/`ToolInvocation` do mesmo passo. |

## 2. Escalabilidade

| ID | Atributo | Meta (SLO) | SLI / Método de verificação |
|----|----------|-----------|------------------------------|
| NFR-004 | Densidade de sessões concorrentes | **≥ 500** sessões concorrentes por nó (agentes ativos leves); **10⁶+** agentes majoritariamente *cold*, materializáveis sob demanda. | Contagem de sessões em `RuntimeState=busy` por nó; taxa de sucesso/latência de `resume` sob população sintética de 10⁵+ sessões `suspended` (`Scalability.md`). |

## 3. Disponibilidade

| ID | Atributo | Meta (SLO) | SLI / Método de verificação |
|----|----------|-----------|------------------------------|
| NFR-005 | Disponibilidade do pool | Runtime pool disponível **≥ 99,9%**; falha de instância NÃO DEVE perder trabalho já aceito. | `aios_runtime_availability_ratio`; *chaos test* que mata instâncias em `busy` e verifica retomada via checkpoint sem perda de tarefa aceita. |

## 4. Recuperabilidade

| ID | Atributo | Meta (SLO) | SLI / Método de verificação |
|----|----------|-----------|------------------------------|
| NFR-006 | RTO/RPO de sessão | **RTO ≤ 30 s** para retomar sessão suspensa; **RPO ≤ 1 passo** do loop (última observação persistida). | Medição do tempo entre `Resume` e estado ativo restaurado; contagem de passos perdidos em *crash injection* entre checkpoints. |

## 5. Throughput

| ID | Atributo | Meta (SLO) | SLI / Método de verificação |
|----|----------|-----------|------------------------------|
| NFR-007 | Throughput de eventos | **≥ 5.000 msg/s** de eventos de execução por nó, sem *backpressure* observável no outbox. | `aios_runtime_events_published_total` (taxa); load test de outbox/publish em `Benchmark.md`. |

## 6. Segurança

| ID | Atributo | Meta (SLO) | SLI / Método de verificação |
|----|----------|-----------|------------------------------|
| NFR-008 | Cobertura de PEP e isolamento de sandbox | **100%** das ações privilegiadas passam pelo `CapabilityEnforcer` (PEP); **0** escapes de sandbox confirmados em *pentest*. | Auditoria de cobertura (`025-Audit`) cruzando ações privilegiadas com decisões do PEP; relatório de *pentest*/teste de escape de sandbox (`Security.md`). |

## 7. Observabilidade

| ID | Atributo | Meta (SLO) | SLI / Método de verificação |
|----|----------|-----------|------------------------------|
| NFR-009 | Cobertura de telemetria | **100%** dos passos do loop DEVEM produzir trace/span e correlação `traceparent`/`tenant`/`agent`. | Inspeção amostral de spans em staging/produção; regra de lint de instrumentação no CI (nenhum handler de fase do loop sem `Span`). |

## 8. Eficiência de Custo

| ID | Atributo | Meta (SLO) | SLI / Método de verificação |
|----|----------|-----------|------------------------------|
| NFR-010 | Custo de memória por sessão | Overhead de memória por sessão ociosa **≤ 30 MiB**; sessão *cold* (`suspended`, checkpointada) consome **0** RAM. | `aios_runtime_session_rss_mib`; medição de RAM total por nó correlacionada com `count(sessions where state='suspended')`. |

## 9. Manutenibilidade

| ID | Atributo | Meta (SLO) | SLI / Método de verificação |
|----|----------|-----------|------------------------------|
| NFR-011 | Cobertura de testes e contratos | Cobertura de testes **≥ 85%**; contratos gRPC (`RuntimeControl`/`AgentExecution`) e eventos testados por *contract tests*. | Relatório de cobertura de linha no CI; suíte de contract test executada a cada *release* (`Testing.md`). |

## 10. Isolamento Multi-tenant

| ID | Atributo | Meta (SLO) | SLI / Método de verificação |
|----|----------|-----------|------------------------------|
| NFR-012 | Isolamento entre tenants | **Nenhum** vazamento cross-tenant; `tenant_id` DEVE ser validado contra o contexto autenticado em toda operação. | Teste de isolamento dedicado (sessão de tenant A não enxerga/afeta dados de tenant B); RLS aplicada no control plane para as entidades duráveis. |

## 11. Conformidade Regulatória (LGPD/GDPR)

| ID | Atributo | Meta (SLO) | SLI / Método de verificação |
|----|----------|-----------|------------------------------|
| NFR-013 | Minimização e expurgo rastreável | **100%** dos payloads de evento/log com `pii.redaction.enabled=true` DEVEM ter PII redigida; expurgo de checkpoints/working memory sob comando de esquecimento DEVE ser rastreável e auditável. | Auditoria de amostragem de PII em payloads publicados; teste de expurgo fim-a-fim coordenado com `010-Memory`/`025-Audit` (`Security.md` §12.3 do brief). |

## 12. Compatibilidade e Portabilidade

| ID | Atributo | Meta (SLO) | SLI / Método de verificação |
|----|----------|-----------|------------------------------|
| NFR-014 | Compatibilidade de ABI/catálogo MCP | ABI gRPC `aios.runtime.v1` DEVE coexistir por **≥ 2 majors** (RFC-0001 §5.7); o `McpHost` DEVE tolerar evolução aditiva do catálogo de ferramentas sem *restart* obrigatório. | Teste de compatibilidade `N-1`/`N` da ABI; teste de recarga do catálogo MCP a partir de `aios.<t>.tool.registry.updated` sem interrupção de sessões ativas. |

## 13. Testabilidade

| ID | Atributo | Meta (SLO) | SLI / Método de verificação |
|----|----------|-----------|------------------------------|
| NFR-015 | Cobertura de contract test da ABI | **100%** dos métodos de `RuntimeControl` e `AgentExecution` cobertos por contract test antes de qualquer *release*. | Relatório de cobertura de contract test no pipeline de CI; gate de *release* bloqueia merge com método não coberto. |

## 14. Resiliência

| ID | Atributo | Meta (SLO) | SLI / Método de verificação |
|----|----------|-----------|------------------------------|
| NFR-016 | Recuperação de *circuit breaker* | *Circuit breaker* de tool/LLM DEVE abrir em **≤ 1 janela de avaliação** (`tool.circuit_breaker.error_threshold`) e fechar em **≤ 5 min** após a dependência voltar a responder corretamente. | *Chaos test* de indisponibilidade/recuperação de `015-Tool-Manager`/`017-Model-Router`; medição do tempo entre recuperação da dependência e fechamento do breaker. |

## Notas

- Metas de NFR-001 a NFR-012 são **idênticas** às do `_DESIGN_BRIEF.md` §7.2;
  qualquer divergência futura DEVE ser corrigida primeiro no brief e depois
  propagada aqui.
- NFR-013 a NFR-016 elaboram atributos de qualidade exigidos pelo esqueleto do
  `MODULE_TEMPLATE.md` (conformidade, compatibilidade, testabilidade,
  resiliência) que o brief cobre em outras seções (§9, §12.3) mas sem NFR
  numerado explícito — sua numeração é a mesma já anunciada em
  `Requirements.md` §4, sem contradição com o brief.
- Rastreabilidade completa (`NFR → UC → Teste`) em `Requirements.md` §5.
