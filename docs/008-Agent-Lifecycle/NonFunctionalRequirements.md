---
Documento: NonFunctionalRequirements
Módulo: 008-Agent-Lifecycle
Status: Draft
Versão: 0.1
Última atualização: 2026-07-20
Responsável (RACI-A): Arquiteto do Módulo 008 — Agent Lifecycle
ADRs relacionados: ADR-0080, ADR-0081, ADR-0082, ADR-0083, ADR-0084, ADR-0086, ADR-0087, ADR-0089
RFCs relacionados: RFC-0001, RFC-0080 (a propor)
Depende de: `_DESIGN_BRIEF.md` (deste módulo), `Requirements.md`, `../024-Observability/`, `../021-Security/`, `../027-Cluster/`
---

# 008-Agent-Lifecycle — Non-Functional Requirements

> Toda meta abaixo é a **mesma** definida no `_DESIGN_BRIEF.md` §7.2 (para
> `NFR-001`..`NFR-013`) ou uma elaboração não-contraditória derivada das
> demais seções do brief (para `NFR-014`..`NFR-017`). Nenhuma meta numérica
> aqui diverge do brief. SLI = *Service Level Indicator* (o que é medido);
> SLO = *Service Level Objective* (a meta); método de verificação referencia
> `Testing.md`/`Benchmark.md`/`Monitoring.md`.

## 1. Desempenho

| ID | Atributo | Meta (SLO) | SLI / Método de verificação |
|----|----------|-----------|------------------------------|
| NFR-001 | Latência de spawn/materialização | `p99 ≤ 250 ms`, `p50 ≤ 80 ms` (cold → `Running`, com `WarmPoolManager` ativo). | Histograma `aios_lifecycle_spawn_duration_ms`; carga sintética em `Benchmark.md` §"Spawn/Wake Latency". |
| NFR-002 | Latência de decisão de transição | `p99 ≤ 20 ms`, excluindo efeitos colaterais de I/O (checkpoint, chamadas a 007/009). | `aios_lifecycle_transition_duration_ms{trigger}`; load test isolando o `StateMachineEngine` do I/O de efeitos. |
| NFR-003 | Latência de checkpoint | `p99 ≤ 500 ms` para working memory `≤ 64 MiB`. | `aios_lifecycle_checkpoint_duration_ms`; benchmark com payloads sintéticos de 1 MiB a 64 MiB. |

## 2. Escalabilidade

| ID | Atributo | Meta (SLO) | SLI / Método de verificação |
|----|----------|-----------|------------------------------|
| NFR-004 | Escala de agentes geridos | Suportar `≥ 10⁶` agentes por cluster, com `≤ 5%` ativos em RAM simultaneamente (a maioria em `Hibernated`). | `aios_lifecycle_agents_total{state}`; teste de escala com população sintética em `Scalability.md`. |

## 3. Recuperabilidade

| ID | Atributo | Meta (SLO) | SLI / Método de verificação |
|----|----------|-----------|------------------------------|
| NFR-005 | Tempo de recuperação (RTO) | `RTO ≤ 15 min` para restaurar um agente após falha de nó (runtime perdido → `woken`/`resumed`). | Medição de tempo `runtime.exited` → `woken`/`resumed`; *DR drill* trimestral em `FailureRecovery.md`. |

## 4. Durabilidade / RPO

| ID | Atributo | Meta (SLO) | SLI / Método de verificação |
|----|----------|-----------|------------------------------|
| NFR-006 | Ponto de recuperação (RPO) | `RPO ≤ 5 min` de working memory (checkpoint periódico + em `suspend`). | Intervalo entre checkpoints válidos consecutivos (`Checkpoint.created_at`); *chaos test* de perda de working memory não persistida. |

## 5. Disponibilidade

| ID | Atributo | Meta (SLO) | SLI / Método de verificação |
|----|----------|-----------|------------------------------|
| NFR-007 | Disponibilidade do serviço 008 | `≥ 99,95%` de uptime mensal (serviço *stateless*, replicado; janela de 30 dias, excluindo manutenção programada anunciada). | Uptime probe sintético (`/healthz`, `/readyz`); *error budget* mensal calculado em `Monitoring.md`. |

## 6. Throughput

| ID | Atributo | Meta (SLO) | SLI / Método de verificação |
|----|----------|-----------|------------------------------|
| NFR-008 | Throughput de transições | `≥ 50.000 transições/s` por cluster. | `rate(aios_lifecycle_transitions_total[5m])`; benchmark de carga sustentada por ≥ 30 min sem degradação de p99 acima das metas de NFR-002. |

## 7. Integridade

| ID | Atributo | Meta (SLO) | SLI / Método de verificação |
|----|----------|-----------|------------------------------|
| NFR-009 | Integridade de snapshot | `0` restore de checkpoint corrompido não detectado. | Verificação de `content_hash` (sha-256) em 100% dos restores; teste de corrupção injetada confirma rejeição com `AIOS-LIFECYCLE-0007`. |

## 8. Desempenho de Migração

| ID | Atributo | Meta (SLO) | SLI / Método de verificação |
|----|----------|-----------|------------------------------|
| NFR-010 | Latência de migração | `p99 ≤ 2 s` (agente `≤ 64 MiB`, mesmo datacenter) sem perda de trabalho aceito. | `aios_lifecycle_migration_duration_ms`; load test de migração em `Benchmark.md`. |

## 9. Eficiência de RAM

| ID | Atributo | Meta (SLO) | SLI / Método de verificação |
|----|----------|-----------|------------------------------|
| NFR-011 | Overhead de hibernação em RAM | ACB frio ocupa `≤ 4 KiB` no índice quente (Redis). | Tamanho médio da projeção fria por agente `Hibernated`, medido em produção/staging; teste de escala em `Scalability.md`. |

## 10. Observabilidade

| ID | Atributo | Meta (SLO) | SLI / Método de verificação |
|----|----------|-----------|------------------------------|
| NFR-012 | Cobertura de telemetria | `100%` das transições emitem trace OTel + evento + log correlacionado (`trace_id`, `tenant_id`, `agent_id`). | Inspeção amostral de spans; regra de lint de instrumentação no CI (nenhum handler de transição novo sem `Activity`/span). |

## 11. Idempotência

| ID | Atributo | Meta (SLO) | SLI / Método de verificação |
|----|----------|-----------|------------------------------|
| NFR-013 | Efeito único sob repetição | Repetição de mutação com mesmo `Idempotency-Key` ⇒ mesmo resultado, `0` efeito duplicado. | Teste de *replay*/duplicação automatizado por ação mutável (`suspend`, `resume`, `hibernate`, `wake`, `migrate`, `checkpoint`, `restore`, `terminate`); verificação de deltas de estado/evento antes/depois da repetição. |

## 12. Manutenibilidade

| ID | Atributo | Meta (SLO) | SLI / Método de verificação |
|----|----------|-----------|------------------------------|
| NFR-014 | Acoplamento e testabilidade | Cada componente interno (§2 do brief) DEVE ser testável isoladamente via *fakes*/*mocks* das dependências externas (Scheduler, Runtime Supervisor, Policy Engine, Cluster), sem dependência circular entre componentes. | Análise estática no CI (linter de dependência); revisão de arquitetura confirma ausência de ciclos no grafo de `Architecture.md`. |

## 13. Testabilidade

| ID | Atributo | Meta (SLO) | SLI / Método de verificação |
|----|----------|-----------|------------------------------|
| NFR-015 | Cobertura de contract test | `100%` das transições da FSM (T1–T11) e dos 11 verbos de API (REST/gRPC) cobertos por contract test antes de qualquer *release*; cobertura de linha `≥ 80%` nos componentes de domínio (`StateMachineEngine`, `AcbStore`, `CheckpointService`, `LeaseManager`). | Relatório de cobertura no pipeline de CI (`Testing.md`); gate de *release* bloqueia merge abaixo do limiar. |

## 14. Segurança

| ID | Atributo | Meta (SLO) | SLI / Método de verificação |
|----|----------|-----------|------------------------------|
| NFR-016 | Cobertura de enforcement | `100%` das mutações de ciclo de vida passam pelo `LifecyclePolicyEnforcer` (PEP); `0` caminhos de *bypass* identificados. | Pentest anual + revisão de código automatizada (regra estática: todo handler de mutação DEVE invocar o PEP); auditoria de cobertura cruzando `LifecycleTransition.actor`/`guard_result` com o catálogo de ações privilegiadas. |

## 15. Conformidade Regulatória (LGPD/GDPR)

| ID | Atributo | Meta (SLO) | SLI / Método de verificação |
|----|----------|-----------|------------------------------|
| NFR-017 | Expurgo rastreável e não-reutilização de ID | `100%` dos agentes `Terminated`/`Failed` além do prazo de retenção (`retention.terminated_ttl_days`) DEVEM ter snapshots/checkpoints expurgados de forma rastreável; `0` reutilização de `agent_id` após expurgo. | Teste de expurgo fim-a-fim coordenado com `025-Audit`; auditoria de amostragem confirma ausência de blobs órfãos em MinIO além do prazo de retenção. |

## Notas

- Metas de `NFR-001` a `NFR-013` são **idênticas** às do `_DESIGN_BRIEF.md`
  §7.2; qualquer divergência futura DEVE ser corrigida primeiro no brief e
  depois propagada aqui.
- `NFR-014` a `NFR-017` cobrem os atributos de qualidade exigidos pelo
  esqueleto do `MODULE_TEMPLATE.md` (manutenibilidade, testabilidade,
  segurança, conformidade) que não possuíam NFR numerado explícito no brief,
  mas são diretamente derivados de suas seções §2 (componentes), §5 (PEP),
  §9 (falhas) e §12.3 (LGPD).
- Rastreabilidade completa (`NFR → UC → Teste`) em `Requirements.md` §5.
