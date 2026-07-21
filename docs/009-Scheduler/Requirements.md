---
Documento: Requirements
Módulo: 009-Scheduler
Status: Draft
Versão: 0.1
Última atualização: 2026-07-20
Responsável (RACI-A): Arquiteto do Módulo 009-Scheduler
ADRs relacionados: ADR-0090, ADR-0091, ADR-0092, ADR-0093, ADR-0094, ADR-0095, ADR-0096, ADR-0097
RFCs relacionados: RFC-0001, RFC-0090, RFC-0091
Depende de: 006-Kernel, 022-Policy, 026-Cost-Optimizer, 027-Cluster
---

# 009-Scheduler — Requirements

> **Autoridade normativa.** Este documento consolida e indexa os requisitos do
> Scheduler (009). Ele **não redefine** contratos centrais — reutiliza URN, o
> envelope de evento CloudEvents, o envelope de erro RFC 7807, `Idempotency-Key`
> e a correlação `traceparent`/`X-AIOS-Tenant` definidos em
> [RFC-0001](../003-RFC/RFC-0001-Architecture-Baseline.md), e a terminologia de
> [Glossary.md](../040-Glossary/Glossary.md). A fonte única de verdade dos
> requisitos detalhados é o [`_DESIGN_BRIEF.md`](./_DESIGN_BRIEF.md) (§7); este
> documento organiza, indexa e rastreia esses requisitos — não os contradiz.
>
> Palavras normativas (DEVE, NÃO DEVE, DEVERIA, NÃO DEVERIA, PODE) seguem
> RFC 2119 / RFC 8174.

---

## 1. Propósito e Escopo do Documento

Este documento é o **índice consolidado de requisitos** do módulo 009-Scheduler:
ele aponta para os requisitos funcionais detalhados em
[`FunctionalRequirements.md`](./FunctionalRequirements.md), os requisitos
não-funcionais detalhados em [`NonFunctionalRequirements.md`](./NonFunctionalRequirements.md)
e os casos de uso em [`UseCases.md`](./UseCases.md), estabelecendo a **matriz de
rastreabilidade** entre requisito → caso de uso → estratégia de teste (detalhada
em `Testing.md`, de batch posterior).

O Scheduler é o **scheduler cognitivo** do plano de controle do AIOS: decide
*quem* executa, *quando*, *onde* (placement), com *qual prioridade* e a *que
custo*, sob restrições de cota, orçamento, SLA e capacidade — análogo ao CFS do
Linux, porém consciente de custo e qualidade (ver `_DESIGN_BRIEF.md` §0).

### 1.1 Fora de escopo deste documento

Este documento **NÃO** especifica: a topologia de deployment (`Deployment.md`),
o modelo físico de dados (`Database.md`), os contratos de API linha a linha
(`API.md`), o catálogo de eventos (`Events.md`) ou as métricas (`Metrics.md`).
Ele referencia esses documentos onde a rastreabilidade o exige.

---

## 2. Stakeholders e Papéis

| Stakeholder | Papel | Interesse no módulo |
|-------------|-------|----------------------|
| Arquiteto do Módulo 009-Scheduler | RACI-A (Accountable) | Consistência do design com o brief e a RFC-0001. |
| Kernel (006) | Consumidor primário (cliente `Submit`) | Precisa de decisão de admissão/placement em p99 ≤ 20 ms para honrar syscalls cognitivas. |
| Runtime Supervisor (007) | Executor da decisão | Recebe `SchedulingDecision` e materializa spawn/suspend/kill; reporta sinais de runtime. |
| Policy Engine / PDP (022) | Autoridade de autorização | Decide admissão/preempção; Scheduler é PEP, nunca PDP. |
| Cost-Optimizer (026) | Fonte de orçamento/estimativa | Fornece `est_cost_usd`/orçamento residual para o score `C`. |
| Cluster (027) | Fonte de capacidade/topologia | Fornece inventário de shards/nós e eventos de mudança de capacidade. |
| Audit (025) | Consumidor de proveniência | Consome `scheduler.decision.recorded` para trilha imutável. |
| Learning (023) | Consumidor de features | Consome `feature_vector` para treino da política aprendida (v2 RL). |
| Operador/SRE (029) | Operação e observabilidade | Depende de `Monitoring.md`/`Metrics.md`/runbooks para operar o serviço. |
| Tenant (cliente final da plataforma) | Usuário indireto | Espera fairness (`Q_min`), SLA e custo previsível. |

---

## 3. Escopo Funcional (Responsabilidades vs. Não-Responsabilidades)

O escopo funcional do módulo é fixado no brief (§1) e reproduzido aqui apenas
como referência de rastreabilidade — a definição normativa vive no brief.

### 3.1 Dentro do escopo (resumo, ver brief §1.1 para o detalhe R-01..R-13)

| Área | Requisitos relacionados |
|------|--------------------------|
| Admission Control | FR-001, FR-006, FR-008 |
| Prioridade/Custo (score multiobjetivo) | FR-002, FR-012, FR-013 |
| Placement (sharding determinístico) | FR-003 |
| Preempção | FR-004 |
| Backpressure | FR-005, FR-022 |
| Filas de prioridade e fairness | FR-006, FR-009, FR-024 |
| Eventos e proveniência | FR-007, FR-010, FR-017 |
| Idempotência | FR-008, FR-011 |
| Reconciliação e re-scheduling | FR-011, FR-014, FR-023 |
| Extensibilidade de política (v1→v2 RL) | FR-013, FR-018 |
| Segurança/LGPD transversal | FR-020, FR-021 |

### 3.2 Fora do escopo (não-responsabilidades — ver brief §1.2 N-01..N-10)

O Scheduler **NÃO DEVE**: executar o loop cognitivo do agente (007), materializar
placement (007), avaliar políticas RBAC/ABAC (022), escolher modelo LLM (017),
calcular/faturar custo (026), persistir memória/contexto (010/011/018/019),
prover HA de infraestrutura (027), gerenciar cotas de token de inferência
(026/017), manter trilha de auditoria imutável (025, apenas emite eventos) ou
decidir *o que* a tarefa faz (012/014).

### 3.3 Diagrama de contexto (referência)

```
   Kernel(006) ── SchedulingRequest (gRPC) ────────────▶ Scheduler(009)
   Policy(022) ◀── "pode admitir/preemptar?" (PDP) ────  Scheduler(009)
   Cost-Optimizer(026) ◀── orçamento/estimativa custo ─  Scheduler(009)
   Cluster(027) ──── inventário capacidade/shards ─────▶ Scheduler(009)
   Scheduler(009) ── SchedulingDecision (place/preempt) ▶ Runtime Supervisor(007)
   Scheduler(009) ── aios.<tenant>.task.execution.* ────▶ NATS/JetStream ──▶ Audit(025)/Learning(023)
```

---

## 4. Requisitos Funcionais — Índice

Detalhamento completo (ID, descrição normativa, prioridade MoSCoW, critério de
aceite, origem) em [`FunctionalRequirements.md`](./FunctionalRequirements.md).
Índice resumido:

| ID | Resumo | Prioridade |
|----|--------|------------|
| FR-001 | Admissão/rejeição por cota, orçamento, capacidade e PDP | MUST |
| FR-002 | Score multiobjetivo `C = α·L + β·$ + γ·E` e prioridade efetiva | MUST |
| FR-003 | Placement determinístico `hash(tenant,agent) mod N` | MUST |
| FR-004 | Preempção segura com grace period e autorização PDP | MUST |
| FR-005 | Backpressure com `Retry-After` sob saturação | MUST |
| FR-006 | Filas de prioridade Redis ZSET por (tenant, classe, shard) | MUST |
| FR-007 | Emissão de eventos do ciclo de decisão | MUST |
| FR-008 | Idempotência por `Idempotency-Key`/`request_id` | MUST |
| FR-009 | Anti-starvation via aging e vazão mínima | MUST |
| FR-010 | Registro de proveniência da decisão | MUST |
| FR-011 | Re-scheduling sob mudança de capacidade/preempção/expiração | MUST |
| FR-012 | Suporte a EDF para tarefas com `deadline_at` | SHOULD |
| FR-013 | Interface `ISchedulingPolicy` estável (v1 heurística → v2 RL) | MUST |
| FR-014 | Reconciliação de filas × runtime real e leases órfãos | MUST |
| FR-015 | Cancelamento de tarefas em QUEUED/SCHEDULED | SHOULD |
| FR-016 | Superfície gRPC/REST conforme brief §5 | MUST |
| FR-017 | Consulta de proveniência (`GetDecision`) e observabilidade de filas (`StreamQueueState`) | SHOULD |
| FR-018 | Recarga em runtime de pesos/config sem reinício | SHOULD |
| FR-019 | Isolamento multi-tenant (RLS + namespace NATS + filas por tenant) | MUST |
| FR-020 | Enforcement PEP→PDP em toda mutação privilegiada | MUST |
| FR-021 | Minimização de dados (payload por referência, sem PII bruta em features) | MUST |
| FR-022 | Sinalização de níveis de backpressure (accept/defer/reject) | MUST |
| FR-023 | Rebalanceamento de shards coordenado com Cluster (027) | SHOULD |
| FR-024 | Configuração de vazão mínima (`fairness.min_share`) por tenant | MUST |
| FR-025 | Tratamento de eventos não processáveis via DLQ | SHOULD |

---

## 5. Requisitos Não-Funcionais — Índice

Detalhamento completo (atributo, meta SLO, SLI/método) em
[`NonFunctionalRequirements.md`](./NonFunctionalRequirements.md). Índice resumido:

| ID | Atributo | Meta |
|----|----------|------|
| NFR-001 | Latência de decisão | p99 ≤ 20 ms, p50 ≤ 5 ms |
| NFR-002 | Throughput | ≥ 50.000 decisões/s por réplica |
| NFR-003 | Disponibilidade | ≥ 99,95% |
| NFR-004 | Escalabilidade | 1 → 10⁶⁺ agentes |
| NFR-005 | Corretude de admissão | 0 overcommit de slot |
| NFR-006 | Fairness | `Q_min` ≥ 1% de vazão em janela de 60s |
| NFR-007 | Durabilidade da decisão | RPO ≤ 5 min |
| NFR-008 | Recuperação | RTO ≤ 15 min |
| NFR-009 | Custo de preempção | overhead ≤ 50 ms; grace default 2s |
| NFR-010 | Observabilidade | 100% das decisões com trace/span/evento |
| NFR-011 | Idempotência | 0 admissões duplicadas |
| NFR-012 | Segurança | 100% das mutações via PEP→PDP |
| NFR-013 | Manutenibilidade | Troca de política v1→v2 sem mudança de API/eventos |
| NFR-014 | Portabilidade operacional | Rebalanceamento de shards sem downtime de admissão |
| NFR-015 | Retenção de dados | Idempotência ≥ 24h; decisões conforme política de retenção |
| NFR-016 | Isolamento multi-tenant | 0 vazamento cross-tenant em filas/decisões |
| NFR-017 | Agilidade de configuração | Recarga de pesos/config ≤ 5 s sem reinício |

---

## 6. Casos de Uso Relacionados — Índice

Detalhamento completo (ator, pré-condições, fluxos, exceções, pós-condições) em
[`UseCases.md`](./UseCases.md). Índice resumido:

| ID | Título | FR/NFR relacionados |
|----|--------|----------------------|
| UC-001 | Submeter tarefa e obter admissão (caminho feliz) | FR-001, FR-002, FR-006, NFR-001 |
| UC-002 | Rejeitar por cota de concorrência excedida | FR-001, NFR-005 |
| UC-003 | Rejeitar por orçamento insuficiente | FR-001, FR-002 |
| UC-004 | Rejeitar por política (PDP nega admissão) | FR-020, NFR-012 |
| UC-005 | Placement em shard/nó com afinidade | FR-003 |
| UC-006 | Preemptar tarefa de menor prioridade | FR-004, NFR-009 |
| UC-007 | Resumir tarefa preemptada | FR-004, FR-011 |
| UC-008 | Ativar backpressure sob saturação | FR-005, FR-022 |
| UC-009 | Cancelar tarefa em fila | FR-015 |
| UC-010 | Escalonar por deadline (EDF) | FR-012 |
| UC-011 | Retry após falha do Runtime | FR-011, FR-014 |
| UC-012 | Reconciliar lease órfão | FR-014, NFR-008 |
| UC-013 | Submissão duplicada (idempotência) | FR-008, NFR-011 |
| UC-014 | Garantir fairness/anti-starvation | FR-009, FR-024, NFR-006 |
| UC-015 | Atualizar pesos de política (α/β/γ) | FR-018, FR-020 |
| UC-016 | Consultar proveniência da decisão | FR-017 |
| UC-017 | Rebalancear shards sob mudança de capacidade (027) | FR-011, FR-023 |
| UC-018 | Trocar política v1 (heurística) → v2 (RL) sem downtime | FR-013, NFR-013 |

---

## 7. Matriz de Rastreabilidade (Requisito → Caso de Uso → Teste)

A coluna "Teste" referencia a estratégia definida em `Testing.md` (documento de
batch posterior); os tipos citados aqui (unit/integration/contract/load/chaos)
são os DEVERÃO ser detalhados naquele documento sem contradição a esta matriz.

| Requisito | Caso(s) de Uso | Tipo de teste esperado |
|-----------|-----------------|--------------------------|
| FR-001 / NFR-005 | UC-001, UC-002, UC-003, UC-004 | integration + chaos (concorrência de admissão) |
| FR-002 | UC-001, UC-010 | unit (score) + contract (política) |
| FR-003 | UC-005 | unit (hash determinístico) + integration (afinidade) |
| FR-004 / NFR-009 | UC-006, UC-007 | integration + load (preempção sob carga) |
| FR-005 / FR-022 | UC-008 | load (saturação) + chaos |
| FR-006 | UC-001, UC-014 | unit (ZSET) + integration |
| FR-007 | UC-001..UC-011 | contract (schema de evento) |
| FR-008 / NFR-011 | UC-013 | integration (retry/duplicação) |
| FR-009 / FR-024 / NFR-006 | UC-014 | load (fairness multi-tenant) |
| FR-010 | UC-016 | contract (proveniência) |
| FR-011 / FR-014 | UC-011, UC-012, UC-017 | chaos (falha de Runtime/Redis) |
| FR-012 | UC-010 | unit (EDF) + integration |
| FR-013 / NFR-013 | UC-018 | contract (interface `ISchedulingPolicy`) |
| FR-015 | UC-009 | integration |
| FR-016 | UC-001..UC-016 | contract (OpenAPI/proto) |
| FR-017 | UC-016 | integration (streaming) |
| FR-018 | UC-015 | integration (hot reload) |
| FR-019 / NFR-016 | UC-001, UC-014 | integration (isolamento RLS) |
| FR-020 / NFR-012 | UC-004, UC-006, UC-015 | contract (PEP→PDP) + security review |
| FR-021 | UC-001 | security review (minimização) |
| FR-023 | UC-017 | chaos (mudança de N) |
| FR-025 | (transversal a eventos) | integration (DLQ replay) |

---

## 8. Premissas e Restrições

| # | Premissa/Restrição |
|---|---------------------|
| P-01 | O Runtime Supervisor (007) é a única autoridade que materializa spawn/suspend/kill; o Scheduler apenas decide. |
| P-02 | O Policy Engine (022) está disponível com cache de curta duração; indisponibilidade implica *default deny* (RFC-0001 §5.8). |
| P-03 | O Cost-Optimizer (026) fornece estimativas com latência aceitável para o caminho quente (cache local usado quando necessário). |
| P-04 | O Cluster (027) é a fonte de verdade de topologia/capacidade; o Scheduler mantém apenas uma visão em cache (`CapacityProvider`). |
| P-05 | Redis e PostgreSQL possuem HA/replicação providos pelo 027 — o Scheduler não implementa failover de infraestrutura. |
| P-06 | O número de shards `N` (`scheduler.shards.count`) muda com baixa frequência e de forma coordenada (não é hot-path). |
| P-07 | Toda `SchedulingRequest` chega com `traceparent` e (quando mutação) `Idempotency-Key` válidos, conforme RFC-0001 §5.5/§5.6. |

---

## 9. Riscos e Mitigações

| Risco | Impacto | Mitigação | Requisito relacionado |
|-------|---------|-----------|--------------------------|
| Overcommit de slots sob corrida de admissão | Violação de SLA/custo | Reserva atômica (Lua/CAS) no Redis | FR-001, NFR-005 |
| Starvation de tenants de baixa prioridade | Insatisfação/SLA quebrado | Aging + `min_share` garantido | FR-009, FR-024, NFR-006 |
| Thrashing de preempção | Degradação de throughput | Histerese + cooldown por tarefa (ADR-0094) | FR-004, NFR-009 |
| Indisponibilidade do PDP (022) travando admissão | Rejeições em massa (fail-safe) | *Default deny* documentado e comunicado como comportamento esperado | FR-020, NFR-012 |
| Mudança de política v1→v2 introduzir regressão de qualidade | Score `C` incoerente, SLA quebrado | Interface `ISchedulingPolicy` estável + testes de contrato + canary por tenant | FR-013, NFR-013 |
| Rebalanceamento de shards causar picos de latência | Violação de NFR-001 durante migração | Hashing consistente, coordenação com 027, janela de baixo tráfego | FR-023, NFR-014 |
| Vazamento de PII em `feature_vector`/eventos | Violação LGPD/GDPR | Minimização por design, `payload_ref`, revisão de segurança | FR-021 |

---

## 10. Glossário Local (referência)

Este documento reutiliza sem redefinir os termos de
[`Glossary.md`](../040-Glossary/Glossary.md): **Admission Control**, **Agent**,
**Backpressure**, **Cold Agent**, **Control Plane**, **Fan-out**, **Idempotency**,
**Outbox**, **PEP/PDP**, **Placement**, **Preemption**, **Quota**, **Shard**,
**Task**, **Tenant**. Termos específicos do domínio de scheduling (score `C`,
classe de serviço, EDF, aging) estão definidos normativamente no
[`_DESIGN_BRIEF.md`](./_DESIGN_BRIEF.md) §1–§4 e não são redefinidos aqui.

---

## 11. Referências Cruzadas

- Fonte única de verdade do módulo: [`_DESIGN_BRIEF.md`](./_DESIGN_BRIEF.md)
- Contratos centrais: [`../003-RFC/RFC-0001-Architecture-Baseline.md`](../003-RFC/RFC-0001-Architecture-Baseline.md)
- Glossário: [`../040-Glossary/Glossary.md`](../040-Glossary/Glossary.md)
- Requisitos detalhados: [`FunctionalRequirements.md`](./FunctionalRequirements.md), [`NonFunctionalRequirements.md`](./NonFunctionalRequirements.md)
- Casos de uso: [`UseCases.md`](./UseCases.md)
- Módulos dependentes: [`../006-Kernel/`](../006-Kernel/), [`../022-Policy/`](../022-Policy/), [`../026-Cost-Optimizer/`](../026-Cost-Optimizer/), [`../027-Cluster/`](../027-Cluster/)

---

*Fim de `Requirements.md` — Módulo 009-Scheduler.*
