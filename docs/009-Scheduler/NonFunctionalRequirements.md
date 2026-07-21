---
Documento: NonFunctionalRequirements
Módulo: 009-Scheduler
Status: Draft
Versão: 0.1
Última atualização: 2026-07-20
Responsável (RACI-A): Arquiteto do Módulo 009-Scheduler
ADRs relacionados: ADR-0090, ADR-0091, ADR-0092, ADR-0093, ADR-0094, ADR-0095, ADR-0096, ADR-0097
RFCs relacionados: RFC-0001, RFC-0090, RFC-0091
Depende de: 006-Kernel, 022-Policy, 026-Cost-Optimizer, 027-Cluster
---

# 009-Scheduler — Non-Functional Requirements

> Cada requisito é identificado por `NFR-NNN`, organizado por atributo de
> qualidade, com meta numérica (SLO), indicador (SLI) e método de verificação.
> NFR-001 a NFR-012 reproduzem e detalham literalmente o
> [`_DESIGN_BRIEF.md`](./_DESIGN_BRIEF.md) §7.2 — os números não podem divergir
> do brief. NFR-013 a NFR-017 elevam a requisito não-funcional explícito
> compromissos já assumidos em outras seções do brief (§8, §10, §12), sem
> introduzir metas novas que o contradigam.
>
> Palavras normativas conforme RFC 2119 / RFC 8174. Unidades seguem o padrão
> AIOS: latência em `ms` (p50/p95/p99), throughput em `req/s`, custo em `USD`.

---

## Índice por atributo de qualidade

| Atributo | NFRs |
|----------|------|
| Performance (latência/throughput) | NFR-001, NFR-002, NFR-009, NFR-017 |
| Escalabilidade | NFR-004, NFR-014 |
| Disponibilidade e Recuperação | NFR-003, NFR-007, NFR-008 |
| Corretude e Consistência | NFR-005, NFR-011 |
| Fairness | NFR-006 |
| Observabilidade | NFR-010 |
| Segurança | NFR-012, NFR-016 |
| Manutenibilidade | NFR-013 |
| Retenção de Dados | NFR-015 |

---

## Tabela consolidada (metas)

| ID | Atributo | Meta (SLO) | SLI / Método |
|----|----------|-----------|--------------|
| NFR-001 | Latência de decisão | p99 ≤ 20 ms, p50 ≤ 5 ms | Histograma `aios_scheduler_decision_latency_ms` |
| NFR-002 | Throughput | ≥ 50.000 decisões/s por réplica | Load test sustentado |
| NFR-003 | Disponibilidade | ≥ 99,95% | Uptime probe / error budget |
| NFR-004 | Escalabilidade | 1 → 10⁶⁺ agentes | Benchmark de escala |
| NFR-005 | Corretude de admissão | 0 overcommit de slot | Teste de concorrência/chaos |
| NFR-006 | Fairness | `Q_min` ≥ 1% de vazão em janela de 60s | Métrica de fairness por tenant |
| NFR-007 | Durabilidade da decisão | RPO ≤ 5 min | Teste de recuperação |
| NFR-008 | Recuperação (RTO) | RTO ≤ 15 min | Drill de failover |
| NFR-009 | Custo de preempção | overhead ≤ 50 ms; grace default 2s | Métrica `aios_scheduler_preemption_ms` |
| NFR-010 | Observabilidade | 100% decisões com trace/span/evento | Cobertura OTel |
| NFR-011 | Idempotência | 0 admissões duplicadas | Teste de duplicação |
| NFR-012 | Segurança | 100% mutações via PEP→PDP | Auditoria 025 |
| NFR-013 | Manutenibilidade (troca de política) | v1→v2 sem mudança de API/eventos | Teste de contrato |
| NFR-014 | Portabilidade operacional (rebalanceamento) | 0 downtime de admissão durante mudança de N | Chaos test de rebalanceamento |
| NFR-015 | Retenção de dados | Idempotência ≥ 24h; decisão conforme política de retenção | Auditoria de retenção |
| NFR-016 | Isolamento multi-tenant | 0 vazamento cross-tenant | Teste de isolamento RLS/NATS |
| NFR-017 | Agilidade de configuração | Recarga de pesos/config ≤ 5 s | Teste de hot reload |

---

## Detalhamento por atributo

### Performance

#### NFR-001 — Latência de decisão

**Meta:** p99 ≤ 20 ms, p50 ≤ 5 ms para o caminho `Submit` (admissão + score +
enfileiramento), sem I/O bloqueante externo síncrono no caminho quente.

**SLI:** histograma `aios_scheduler_decision_latency_ms` (ver `Metrics.md`),
segmentado por `tenant`, `service_class`, `outcome`.

**Método de verificação:** teste de carga com percentuais fixos de mix de
classes de serviço (`INTERACTIVE`/`BATCH`/`BACKGROUND`/`SYSTEM`), medido em
ambiente representativo (ver `Benchmark.md`).

**Racional:** o caminho quente usa apenas Redis (cotas/filas, sub-ms) e caches
locais de orçamento/PDP; chamadas externas síncronas ao 022/026 no caminho
crítico são NÃO DEVIDAS (violaria o SLO).

#### NFR-002 — Throughput

**Meta:** ≥ 50.000 decisões/s por réplica; escala linear ao adicionar réplicas
(cada réplica é *owner* de um subconjunto de shards).

**SLI:** taxa sustentada de `Submit` processados sem degradação de p99 acima
da meta de NFR-001.

**Método de verificação:** teste de carga sustentado (≥ 10 min) com
observação de saturação de CPU/Redis; ver `Benchmark.md`.

#### NFR-009 — Custo de preempção

**Meta:** overhead de decisão de preempção ≤ 50 ms; `grace_period_ms` default
2000 ms, configurável por classe.

**SLI:** `aios_scheduler_preemption_ms` (histograma).

**Método de verificação:** teste de integração que força cenário de
preempção sob contenção e mede o tempo entre seleção de vítima e sinalização
de suspensão.

#### NFR-017 — Agilidade de configuração

**Meta:** alteração de pesos (`α`,`β`,`γ`) ou thresholds de backpressure
propaga para novas decisões em ≤ 5 s, sem reinício do serviço.

**SLI:** tempo entre `PUT /v1/scheduler/policy` (ou atualização de config) e a
primeira decisão subsequente refletir o novo valor.

**Método de verificação:** teste de integração com leitura de config
versionada (cache com TTL curto ou invalidação por evento).

---

### Escalabilidade

#### NFR-004 — Escalabilidade

**Meta:** suportar crescimento de 1 a 10⁶⁺ agentes gerenciados, com
coordenação sub-linear (não O(N²)) via sharding determinístico.

**SLI:** benchmark de escala incremental (ver `Scalability.md`/`Benchmark.md`)
medindo latência/throughput em pontos de 10³, 10⁴, 10⁵, 10⁶ agentes simulados.

**Método de verificação:** benchmark dedicado com aumento progressivo de N
(shards) e réplicas, validando que p99 (NFR-001) se mantém dentro da meta.

#### NFR-014 — Portabilidade operacional (rebalanceamento)

**Meta:** mudança no número de shards (`scheduler.shards.count`) NÃO DEVE
causar interrupção de admissão (0 downtime), usando hashing consistente para
minimizar migração (ADR-0092).

**SLI:** fração de chaves remapeadas ao alterar `N` (deve aproximar-se de
`1/N`, não 100%); indisponibilidade observada durante a migração (deve ser 0).

**Método de verificação:** chaos test que altera `N` sob carga e mede impacto
em latência/erros de admissão.

---

### Disponibilidade e Recuperação

#### NFR-003 — Disponibilidade

**Meta:** ≥ 99,95% de disponibilidade do endpoint de scheduling (control
plane), correspondendo a ≤ ~4h23min de indisponibilidade por ano.

**SLI:** uptime probe do endpoint `Submit`/`GetDecision`; error budget mensal.

**Método de verificação:** monitoramento contínuo (ver `Monitoring.md`) com
alertas de burn rate de error budget.

#### NFR-007 — Durabilidade da decisão

**Meta:** RPO ≤ 5 minutos para decisões críticas (outbox durável em
PostgreSQL + JetStream).

**SLI:** intervalo máximo de dados potencialmente perdidos em teste de
recuperação (diferença entre última decisão confirmada e último checkpoint
durável).

**Método de verificação:** drill de recuperação simulando falha de réplica
com decisões em voo.

#### NFR-008 — Recuperação (RTO)

**Meta:** RTO ≤ 15 minutos após falha de réplica, sem perda de trabalho já
admitido.

**SLI:** tempo decorrido entre falha detectada e retomada plena do serviço
(nova réplica assumindo ownership de shards).

**Método de verificação:** drill de failover coordenado com 027.

---

### Corretude e Consistência

#### NFR-005 — Corretude de admissão

**Meta:** 0 (zero) ocorrências de overcommit de slot além de
`max_concurrency` configurado, sob qualquer condição de concorrência.

**SLI:** contagem de violações detectadas por auditoria de `QuotaLedger` vs.
execuções reais reportadas pelo Runtime Supervisor.

**Método de verificação:** teste de concorrência/chaos com rajadas
simultâneas de `Submit` no limite da cota, validando reservas atômicas
(Lua/CAS no Redis).

#### NFR-011 — Idempotência

**Meta:** 0 (zero) admissões duplicadas sob retry do cliente ou entrega
at-least-once de eventos.

**SLI:** contagem de decisões efetivas por `request_id`/`Idempotency-Key`
(deve ser sempre ≤ 1 admissão ativa).

**Método de verificação:** teste de duplicação (replay do mesmo `Submit` N
vezes) e teste de replay de evento (`event.id` repetido).

---

### Fairness

#### NFR-006 — Fairness

**Meta:** todo tenant/classe ativo obtém vazão ≥ `scheduler.fairness.min_share`
(default 1%) em uma janela de observação de 60 segundos, mesmo sob carga
adversarial de outros tenants.

**SLI:** métrica de fairness por tenant (razão entre despachos do tenant e
despachos totais na janela).

**Método de verificação:** teste de carga multi-tenant com um tenant
dominante e verificação de que tenants minoritários não ficam abaixo do piso.

---

### Observabilidade

#### NFR-010 — Observabilidade

**Meta:** 100% das decisões (`SchedulingDecision`) têm trace/span OTel
correspondente e evento de proveniência (`scheduler.decision.recorded`)
publicado.

**SLI:** cobertura de instrumentação (razão entre decisões com span/evento e
total de decisões).

**Método de verificação:** auditoria de amostragem de traces em Seq/Grafana e
verificação de correlação `trace_id` ↔ `decision_id`.

---

### Segurança

#### NFR-012 — Segurança

**Meta:** 100% das mutações privilegiadas (admissão, preempção, atualização
de política) passam por PEP (`SchedulingGateway`) e são autorizadas pelo PDP
(022) — *default deny*.

**SLI:** razão entre mutações auditadas com decisão PDP correspondente e
total de mutações (auditoria via 025).

**Método de verificação:** revisão de segurança (STRIDE, brief §12.2) e teste
de contrato que simula indisponibilidade do PDP, validando negação.

#### NFR-016 — Isolamento multi-tenant

**Meta:** 0 (zero) vazamento de dados/decisões entre tenants (filas, decisões,
eventos, cotas).

**SLI:** contagem de violações detectadas em teste de isolamento (RLS,
namespace NATS, chaves Redis por tenant).

**Método de verificação:** teste de isolamento dedicado (tentativa de acesso
cross-tenant deve falhar consistentemente).

---

### Manutenibilidade

#### NFR-013 — Manutenibilidade (troca de política)

**Meta:** substituir a política de decisão (`heuristic` → `rl`, via
`scheduler.policy.engine`) NÃO DEVE exigir mudança na superfície de API (§5)
nem no catálogo de eventos (§6).

**SLI:** resultado de teste de contrato (schema de request/response e de
evento) executado contra ambas as implementações de `ISchedulingPolicy`.

**Método de verificação:** suíte de testes de contrato executada em pipeline
de CI para cada implementação de política.

---

### Retenção de Dados

#### NFR-015 — Retenção de dados

**Meta:** registros de idempotência retidos por ≥ 24h (`scheduler.idempotency.ttl_hours`,
default 24, faixa 24–168); decisões (`SchedulingDecision`) retidas conforme
política de retenção definida em `Database.md`, com expurgo rastreável por
`tenant_id`/`agent_urn` (LGPD/GDPR, brief §12.3).

**SLI:** verificação de TTL efetivo em amostragem de registros; auditoria de
expurgo executado vs. solicitado.

**Método de verificação:** teste de expiração de TTL e teste de expurgo
(direito ao esquecimento) coordenado com 025.

---

## Matriz de dependência entre NFRs e componentes (brief §2.1)

| NFR | Componente(s) principal(is) responsável(is) |
|-----|-----------------------------------------------|
| NFR-001/002 | `SchedulingGateway`, `AdmissionController`, `PriorityCostEvaluator` |
| NFR-003/007/008 | `DecisionJournal`, `EventPublisher`, `ReconciliationWorker` |
| NFR-004/014 | `ShardRouter`, `PlacementEngine`, `CapacityProvider` |
| NFR-005/011 | `QuotaLedger`, `IdempotencyStore` |
| NFR-006 | `PriorityQueueStore`, `PriorityCostEvaluator` (aging) |
| NFR-009 | `PreemptionManager` |
| NFR-010 | `SchedulerTelemetry` |
| NFR-012/016 | `SchedulingGateway` (PEP), `PolicyClient` |
| NFR-013 | `ISchedulingPolicy`, `HeuristicPolicy`, `LearnedPolicy` |
| NFR-015 | `IdempotencyStore`, `DecisionJournal` |

---

## Referências Cruzadas

- Metas numéricas de origem: [`_DESIGN_BRIEF.md`](./_DESIGN_BRIEF.md) §7.2
- Requisitos funcionais correlatos: [`FunctionalRequirements.md`](./FunctionalRequirements.md)
- Índice consolidado e matriz de rastreabilidade: [`Requirements.md`](./Requirements.md)
- Contratos centrais: [`../003-RFC/RFC-0001-Architecture-Baseline.md`](../003-RFC/RFC-0001-Architecture-Baseline.md)

---

*Fim de `NonFunctionalRequirements.md` — Módulo 009-Scheduler.*
