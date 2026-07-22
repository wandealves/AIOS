---
Documento: Testing
Módulo: 007-Agent-Runtime
Status: Draft
Versão: 0.1
Última atualização: 2026-07-21
Responsável (RACI-A): Arquiteto do Módulo 007 — Agent Runtime
ADRs relacionados: ADR-0071, ADR-0076, ADR-0077, ADR-0078, ADR-0079
RFCs relacionados: RFC-0001, RFC-0070
Depende de: ./FunctionalRequirements.md, ./NonFunctionalRequirements.md, ./UseCases.md, ./Requirements.md
---

# 007-Agent-Runtime — Testing

> Estratégia de testes do Agent Runtime, ligada à matriz de rastreabilidade
> `FR/NFR → UC → Teste` de `./Requirements.md` §5. Toda categoria abaixo
> **DEVE** ser executada em CI antes de qualquer *release* que altere o
> comportamento do módulo (gate de qualidade, §6).

## Índice

1. Pirâmide de testes e categorias
2. Testes unitários
3. Testes de integração
4. Contract tests (gRPC + eventos)
5. Testes end-to-end (e2e)
6. Chaos tests
7. Load tests
8. Fixtures e ambientes
9. Critérios de gate de release
10. Referências

---

## 1. Pirâmide de Testes e Categorias

```
              ▲
             ╱ ╲     e2e (poucos, caros, alta confiança de fluxo completo)
            ╱───╲
           ╱ chaos╲   resiliência sob falha injetada
          ╱─────────╲
         ╱ contract    ╲  gRPC (RuntimeControl/AgentExecution) + eventos NATS
        ╱───────────────╲
       ╱   integration     ╲  componente + dependência real/mock (010/011/012/015/017)
      ╱───────────────────────╲
     ╱        unit                ╲  lógica pura por componente (muitos, rápidos, baratos)
    ╱───────────────────────────────╲
```

| Categoria | Cobertura-alvo | Frequência de execução |
|-----------|-------------------|----------------------------|
| Unit | ≥ 85% de linha nos componentes de domínio (NFR-011) | A cada commit (CI). |
| Integration | 100% dos fluxos de `./UseCases.md` com dependência real/mock | A cada PR. |
| Contract | 100% dos métodos `RuntimeControl`/`AgentExecution` (NFR-015) | A cada PR e antes de *release*. |
| E2E | Fluxos felizes UC-001, UC-003/004, UC-007/008 | Nightly + antes de *release*. |
| Chaos | UC-005, UC-008, UC-009, UC-012, UC-013, UC-015 | Semanal + antes de *release* major. |
| Load | Metas de NFR-001..NFR-004, NFR-007 | Antes de *release*; contínuo em staging (baixa intensidade). |

---

## 2. Testes Unitários

| Componente | Foco de teste | Exemplo de caso |
|-------------|-------------------|---------------------|
| `ExecutionStateMachine` | Todas as transições/guardas de `./StateMachine.md` §2. | Transição `T4` só ocorre com **G2** (capability concedida); transição inválida é rejeitada. |
| `QuotaGovernor` | Cálculo de `is_over_budget()`, `check_budget()` por dimensão. | `consumed.tokens_used = budget.max_tokens` → `exhausted=true`. |
| `CapabilityEnforcer` | Postura *default deny*; cache TTL; invalidação. | Cache-miss sem PDP acessível → `deny` (fail-closed). |
| `CheckpointManager` | Cálculo/validação de `checksum`. | Checksum recalculado diverge do armazenado → `restore()` rejeita. |
| `ToolInvoker`/`ModelRouterClient` | Lógica de *circuit breaker*/retry/backoff isolada de rede real (mocks). | N falhas consecutivas abre o breaker exatamente no limiar configurado. |
| `ObservationCollector` | Normalização de resultado e redação de PII. | PII sintética em `observation` é redigida antes de retornar o `ReActStep`. |

Fixtures usam *fakes* determinísticos para `010`/`011`/`012`/`015`/`017`
(sem I/O real), garantindo testes rápidos e sem *flakiness* de rede.

---

## 3. Testes de Integração

| Fluxo (UC) | Dependências reais/mockadas | O que é verificado |
|-------------|----------------------------------|------------------------|
| UC-001/UC-002 (Boot) | `SandboxManager` real (seccomp/cgroups em contêiner de teste privilegiado), `CapabilityEnforcer` com PDP mock. | `cold_start_ms` medido; evento `booted` publicado; falhas de sandbox mapeadas ao código correto. |
| UC-003/UC-004 (loop completo) | Mocks de `010`/`011`/`012`/`015`/`017` com latência simulada. | Sequência completa de `ReActStep`; `step_cursor` incrementado corretamente; PII redigida. |
| UC-005 (replanejamento) | Mock de `012-Planning` retornando plano válido/inválido. | Transições `reflecting↔thinking`/`failed` conforme guarda **G6**/**G7**. |
| UC-007/UC-008 (checkpoint/resume) | MinIO real (via control plane de teste), Redis real (lease). | Round-trip de checkpoint; RTO/RPO medidos; *lease* previne dupla execução. |
| UC-010 (sandbox violation) | Sandbox real com perfil restritivo de teste. | Syscall proibida é bloqueada e reportada; nenhum efeito fora do sandbox. |
| UC-011 (capability denied) | `CapabilityEnforcer` com PDP mock retornando `deny`. | Ação não executada; `AIOS-RUNTIME-0003` retornado. |
| UC-017 (isolamento de tenant) | Duas sessões de tenants distintos em paralelo. | Nenhum vazamento de dado entre tenants em nenhuma leitura/mutação. |

---

## 4. Contract Tests (gRPC + Eventos)

- **gRPC:** suíte executa cada método de `RuntimeControl` e `AgentExecution`
  (`./API.md` §2) contra um runtime real em ambiente isolado, validando
  schema de request/response, mapeamento de erro (`google.rpc.Status` +
  `ErrorInfo`), e idempotência (`Boot`/`SubmitTask` repetidos).
- **Eventos:** suíte valida que todo evento emitido (`./Events.md` §2)
  respeita o envelope CloudEvents (RFC-0001 §5.2) e o `dataschema`
  registrado; consumidores simulados verificam deduplicação por `event.id`.
- **Compatibilidade de ABI (NFR-014, UC-018):** a mesma suíte de contract
  test da versão `N-1` é executada contra uma instância que já expõe a
  versão `N`, confirmando evolução aditiva sem quebra.

---

## 5. Testes End-to-End (E2E)

| Cenário E2E | Componentes reais envolvidos |
|----------------|-----------------------------------|
| Agente simples do "hello world" ao `done` | Runtime completo + Model Router/Tool Manager de teste (respostas determinísticas) + NATS/Redis/MinIO reais em ambiente de staging. |
| Suspensão por cota → retomada bem-sucedida | Runtime completo + checkpoint real em MinIO + nova instância retomando. |
| Rollout de nova versão sem perda de sessão | Duas versões de imagem coexistindo; `Drain` de uma, `Boot` na outra; sessões suspensas retomadas na nova versão. |

---

## 6. Chaos Tests

| Experimento | UC correspondente | O que é validado |
|----------------|------------------------|------------------------|
| Kill do processo runtime entre checkpoints | UC-008 | RTO ≤ 30s, RPO ≤ 1 passo (NFR-006); nenhuma dupla execução. |
| Indisponibilidade total do Model Router | UC-012 | Fallback funcional; sem chamada direta a provedor (FR-003 preservado mesmo sob stress). |
| Indisponibilidade total do Tool Manager | UC-013 | Circuit breaker abre/fecha conforme NFR-016. |
| Perda de heartbeat / partição de rede | UC-009 | Supervisor recria instância; sessão retomada dentro do RTO. |
| Injeção de passo travado (loop infinito simulado) | UC-015 | Watchdog dispara dentro de `watchdog.stuck_timeout_ms`; recuperação ou kill correto. |
| PDP indisponível durante decisão de capability | UC-011 (variante) | Postura *fail-closed*: toda decisão não cacheada é negada. |

---

## 7. Load Tests

| Teste | Meta verificada | Ferramenta/Abordagem |
|-------|---------------------|---------------------------|
| Cold start sob rajada | NFR-001 (p99 ≤ 250ms/900ms) | Rajada sintética de `Boot` com/sem pool quente; ver `./Benchmark.md`. |
| Throughput de eventos | NFR-007 (≥ 5.000 msg/s/nó) | Geração sintética de passos ReAct em paralelo até saturação do outbox. |
| Densidade de sessões | NFR-004 (≥ 500 sessões/nó) | Materialização progressiva de sessões até o limite `max_sessions_per_node`. |
| Overhead de sandbox | NFR-002 (≤ 5% CPU / ≤ 8ms) | Comparação A/B com/sem `seccomp`/`cgroups` no mesmo workload. |

---

## 8. Fixtures e Ambientes

| Ambiente | Uso | Isolamento |
|-----------|-----|---------------|
| `test-unit` | Testes unitários, sem I/O real. | Processo local, fakes determinísticos. |
| `test-integration` | Testes de integração com dependências reais mínimas (Redis, MinIO, NATS locais). | Contêineres efêmeros por execução de CI. |
| `staging` | E2E, chaos, load em escala reduzida. | Réplica da topologia de produção, tenants sintéticos isolados. |
| `perf-lab` | Benchmarks formais (`./Benchmark.md`). | Ambiente dedicado, sem tráfego concorrente de outros times. |

Fixtures principais: `AgentSpec` sintético mínimo, `SandboxProfile` de
teste (`net-restricted` reduzido), catálogo MCP de ferramentas
determinísticas (eco, falha simulada, timeout simulado), respostas de
`017-Model-Router` gravadas (*golden responses*) para reprodutibilidade.

---

## 9. Critérios de Gate de Release

- [ ] Cobertura de linha ≥ 85% nos componentes de domínio (NFR-011).
- [ ] 100% dos métodos `RuntimeControl`/`AgentExecution` cobertos por
      contract test (NFR-015).
- [ ] Nenhum teste de segurança (escape de sandbox, capability bypass)
      falhando (NFR-008).
- [ ] Suíte de chaos executada sem regressão de RTO/RPO (NFR-006).
- [ ] Load test confirma metas de NFR-001..NFR-004, NFR-007 no ambiente
      `perf-lab` antes de *release* major.
- [ ] Teste de compatibilidade de ABI `N-1`/`N` sem quebra (NFR-014).

---

## 10. Referências

- Matriz de rastreabilidade completa: `./Requirements.md` §5.
- Metas numéricas verificadas: `./NonFunctionalRequirements.md`.
- Metodologia formal de benchmark: `./Benchmark.md`.
- Modos de falha exercitados por chaos tests: `./FailureRecovery.md`.
