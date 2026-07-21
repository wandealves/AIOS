---
Documento: NonFunctionalRequirements
Módulo: 006-Kernel
Status: Draft
Versão: 0.1
Última atualização: 2026-07-20
Responsável (RACI-A): Arquiteto do Módulo 006-Kernel
ADRs relacionados: ADR-0060, ADR-0061, ADR-0062, ADR-0063, ADR-0064, ADR-0065, ADR-0066, ADR-0067, ADR-0069
RFCs relacionados: RFC-0001, RFC-0006, RFC-0007
Depende de: `_DESIGN_BRIEF.md` (deste módulo), `Requirements.md`, `../024-Observability/`, `../021-Security/`
---

# 006-Kernel — Non-Functional Requirements

> Toda meta abaixo é a **mesma** definida no `_DESIGN_BRIEF.md` §7.2 (para
> `NFR-001`..`NFR-012`) ou uma elaboração não-contraditória derivada das
> demais seções do brief (para `NFR-013`..`NFR-018`). Nenhuma meta numérica
> aqui diverge do brief. SLI = *Service Level Indicator* (o que é medido);
> SLO = *Service Level Objective* (a meta); método de verificação referencia
> `Testing.md`/`Benchmark.md`/`Monitoring.md`.

## 1. Desempenho

| ID | Atributo | Meta (SLO) | SLI / Método de verificação |
|----|----------|-----------|------------------------------|
| NFR-001 | Latência de `spawn` | p99 ≤ **250 ms** no caminho quente (agente `cold`→`Running`, incluindo admissão + materialização). | Histograma `aios_kernel_spawn_latency_ms`; carga sintética em `Benchmark.md` §"Spawn Latency". |
| NFR-002 | Latência de syscall de controle | p99 de `suspend`/`resume`/`kill`/`get_quota` ≤ **20 ms**. | `aios_kernel_syscall_latency_ms{verb}`; load test com mix representativo de verbos. |
| NFR-003 | Latência de decisão de capability (PEP) | p99 ≤ **8 ms** com cache hit; p99 ≤ **25 ms** com cache miss (chamada ao PDP). | `aios_kernel_pep_decision_ms{cache_hit}`; teste isolado do `PolicyClient` com PDP mockado e real. |

## 2. Capacidade e Throughput

| ID | Atributo | Meta (SLO) | SLI / Método de verificação |
|----|----------|-----------|------------------------------|
| NFR-004 | Throughput por réplica | ≥ **20.000 syscalls/s** (mix de controle: `get_quota`, `suspend`, `resume`, decisão PEP em cache) por réplica do Kernel Service, sob CPU/memória definidos em `Deployment.md`. | Benchmark de carga sustentada por ≥ 30 min sem degradação de p99 acima das metas de NFR-001/002; ver `Benchmark.md`. |

## 3. Disponibilidade

| ID | Atributo | Meta (SLO) | SLI / Método de verificação |
|----|----------|-----------|------------------------------|
| NFR-005 | Disponibilidade do serviço | ≥ **99,95%** de uptime mensal do control plane do Kernel (janela de 30 dias, excluindo manutenção programada anunciada). | Uptime probe sintético (`/healthz`, `/readyz`); *error budget* mensal calculado em `Monitoring.md`. |

## 4. Escalabilidade

| ID | Atributo | Meta (SLO) | SLI / Método de verificação |
|----|----------|-----------|------------------------------|
| NFR-006 | Escala de ACBs geridos | Suportar ≥ **10⁶** ACBs simultaneamente registrados (majoritariamente `Hibernated`), com custo de infraestrutura crescendo **sub-linearmente** em relação ao número total de ACBs. | Teste de escala com população sintética de ACBs em `Scalability.md`; medição de custo de RAM/CPU por 10⁵ ACBs adicionais. |

## 5. Durabilidade e Recuperação

| ID | Atributo | Meta (SLO) | SLI / Método de verificação |
|----|----------|-----------|------------------------------|
| NFR-007 | Durabilidade de eventos | **0** eventos perdidos sob crash do processo do Kernel entre commit de estado e publicação (padrão Outbox). | *Chaos test* que mata o processo entre commit e publish; verificação de que o relay reenviou 100% dos eventos pendentes. |
| NFR-008 | Recuperação de desastre | **RTO ≤ 15 min**, **RPO ≤ 5 min** para o estado de controle (ACB, cotas, outbox). | *DR drill* trimestral com restauração de réplica PostgreSQL HA; medição de tempo até o Kernel voltar a aceitar syscalls e de perda de dados desde o último ponto de replicação confirmado. |

## 6. Consistência

| ID | Atributo | Meta (SLO) | SLI / Método de verificação |
|----|----------|-----------|------------------------------|
| NFR-009 | Consistência de cota sob concorrência | *Overshoot* de cota com `enforcement=hard` ≤ **1%** do limite configurado, mesmo sob alta concorrência. | Teste de carga concorrente (N *workers* simultâneos próximos ao limite) contra o token-bucket Redis; comparação do consumo real registrado versus o limite. |

## 7. Segurança

| ID | Atributo | Meta (SLO) | SLI / Método de verificação |
|----|----------|-----------|------------------------------|
| NFR-010 | Cobertura de enforcement | **100%** das ações privilegiadas passam pelo PEP; **0** caminhos de *bypass* identificados. | Pentest anual + revisão de código automatizada (regra estática: todo handler de syscall privilegiada DEVE invocar `CapabilityEnforcer`); auditoria de cobertura cruzando `syscall_log.decision` com o catálogo de verbos privilegiados. |

## 8. Observabilidade

| ID | Atributo | Meta (SLO) | SLI / Método de verificação |
|----|----------|-----------|------------------------------|
| NFR-011 | Cobertura de telemetria | **100%** das syscalls emitem trace OTel com `trace_id`, `tenant_id`, `agent_urn` e `verb` correlacionados. | Inspeção amostral de spans em produção/staging; regra de lint de instrumentação no CI (nenhum handler novo sem `Activity`/span). |

## 9. Idempotência

| ID | Atributo | Meta (SLO) | SLI / Método de verificação |
|----|----------|-----------|------------------------------|
| NFR-012 | Efeito único sob repetição | **100%** das mutações repetidas com a mesma `Idempotency-Key` e mesmo payload produzem efeito único (sem duplicação de estado, cota ou evento). | Teste de *replay*/duplicação automatizado por verbo mutável; verificação de contagem de eventos publicados e de deltas de cota antes/depois da repetição. |

## 10. Manutenibilidade

| ID | Atributo | Meta (SLO) | SLI / Método de verificação |
|----|----------|-----------|------------------------------|
| NFR-013 | Acoplamento e complexidade | Cada componente interno (§2.1 do brief) DEVE ser testável isoladamente via *fakes*/*mocks* das dependências externas (PDP, Scheduler, Lifecycle, brokers), com complexidade ciclomática média por método ≤ **10** e sem dependência circular entre componentes. | Análise estática no CI (linter de complexidade); revisão de arquitetura confirma ausência de ciclos no grafo de dependências de `Architecture.md`. |

## 11. Compatibilidade e Portabilidade

| ID | Atributo | Meta (SLO) | SLI / Método de verificação |
|----|----------|-----------|------------------------------|
| NFR-014 | Compatibilidade de ABI | Mudanças incompatíveis na ABI de syscalls DEVEM coexistir por **≥ 2 versões major** antes da remoção da versão anterior (RFC-0001 §5.7). | Teste de compatibilidade que executa a suíte de contract test da versão `N-1` contra o serviço que já expõe `N`; changelog de depreciação em `API.md`. |

## 12. Testabilidade

| ID | Atributo | Meta (SLO) | SLI / Método de verificação |
|----|----------|-----------|------------------------------|
| NFR-015 | Cobertura de contract test | **100%** dos 11 verbos + `GetAcb` cobertos por contract test (REST e gRPC) antes de qualquer *release*; cobertura de linha ≥ **80%** nos componentes de domínio (`AcbStore`, `ResourceQuotaManager`, `LifecycleCoordinator`, `CapabilityEnforcer`). | Relatório de cobertura no pipeline de CI (`Testing.md`); gate de *release* bloqueia merge abaixo do limiar. |

## 13. Conformidade Regulatória (LGPD/GDPR)

| ID | Atributo | Meta (SLO) | SLI / Método de verificação |
|----|----------|-----------|------------------------------|
| NFR-016 | Minimização e direito ao esquecimento | **100%** dos eventos e logs emitidos pelo Kernel usam identificadores opacos (URN) sem PII adicional; `kill` seguido de solicitação de expurgo DEVE completar a remoção rastreável do ACB (mantendo apenas metadados de auditoria exigidos por lei) em até o prazo definido pela política de privacidade vigente. | Auditoria de amostragem de payloads de eventos/logs em busca de PII; teste de expurgo fim-a-fim coordenado com `025-Audit`. |

## 14. Eficiência de Custo Operacional

| ID | Atributo | Meta (SLO) | SLI / Método de verificação |
|----|----------|-----------|------------------------------|
| NFR-017 | Densidade de agentes hibernados | Um agente em estado `Hibernated` DEVE consumir **~0 MiB de RAM ativa** (apenas metadados do ACB em cache, se houver) e ser materializável sob demanda dentro da meta de NFR-001; o custo marginal de manter 10⁵ ACBs adicionais majoritariamente `Hibernated` DEVE crescer sub-linearmente com o total de ACBs. | Medição de RAM por réplica correlacionada com `count(ACB where state='Hibernated')`; teste de escala em `Scalability.md`. |

## 15. Resiliência

| ID | Atributo | Meta (SLO) | SLI / Método de verificação |
|----|----------|-----------|------------------------------|
| NFR-018 | Recuperação automática de dependências | *Circuit breakers* por dependência (`PolicyClient`, `SchedulerClient`, `LifecycleClient`, brokers de recurso) DEVEM abrir dentro de **1 janela de avaliação** ao ultrapassar `kernel.broker.circuit_breaker_threshold` e tentar meia-abertura automaticamente, sem intervenção manual, restaurando o tráfego normal em até **5 min** após a dependência voltar a responder corretamente. | *Chaos test* de indisponibilidade/recuperação de cada dependência; medição do tempo entre a dependência voltar a responder e o breaker fechar (`aios_kernel_circuit_breaker_state`). |

## Notas

- Metas de NFR-001 a NFR-012 são **idênticas** às do `_DESIGN_BRIEF.md` §7.2;
  qualquer divergência futura DEVE ser corrigida primeiro no brief e depois
  propagada aqui.
- NFR-013 a NFR-018 cobrem os atributos de qualidade exigidos pelo esqueleto
  do `MODULE_TEMPLATE.md` (manutenibilidade) e riscos identificados no brief
  (§9 Modos de Falha, §10 Escalabilidade, §12.3 LGPD/GDPR) que não possuíam
  NFR numerado explícito.
- Rastreabilidade completa (`NFR → UC → Teste`) em `Requirements.md` §5.
