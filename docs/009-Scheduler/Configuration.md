---
Documento: Configuration.md
Módulo: 009-Scheduler
Status: Draft
Versão: 0.1
Última atualização: 2026-07-20
Responsável (RACI-A): Arquiteto do Módulo 009-Scheduler
ADRs relacionados: ADR-0090, ADR-0091, ADR-0092, ADR-0094, ADR-0095, ADR-0096 (a propor)
RFCs relacionados: RFC-0001 (baseline); RFC-0091 (a propor)
Depende de: 003-RFC/RFC-0001, 022-Policy, 027-Cluster
---

# 009-Scheduler — Configuration

> **Escopo.** Catálogo canônico de todas as chaves de configuração do
> Scheduler: tipo, default, faixa válida, escopo, recarregabilidade em
> runtime e variável de ambiente correspondente. As chaves e seus defaults
> **DEVEM** ser idênticos aos definidos em `./_DESIGN_BRIEF.md` §8 — este
> documento os detalha, não os redefine nem os contradiz.

---

## 1. Convenções

| Aspecto | Regra |
|---------|-------|
| Formato de arquivo | `appsettings.json` (.NET 10, `IOptions<T>`/`IOptionsMonitor<T>`), seções em `PascalCase` mapeando as chaves em `snake.case` deste catálogo (ex.: `scheduler.weights.alpha` → `Scheduler:Weights:Alpha`). |
| Override por ambiente | Variável de ambiente com separador duplo underscore (convenção .NET): `Scheduler__Weights__Alpha`. Todas as variáveis usam o prefixo `AIOS_` quando providas via orquestrador de cluster (`AIOS_SCHEDULER__WEIGHTS__ALPHA`) — ambas as formas são aceitas, a segunda tem precedência. |
| Precedência de resolução | `default embutido` < `appsettings.json` < variável de ambiente < *override* por tenant (armazenado em `scheduler.policy_weight`, ver `Database.md`) < *override* por classe de serviço (mais específico vence). |
| Recarregável (`Hot-reload`) | Chaves marcadas "Sim" são observadas via `IOptionsMonitor<T>` (mudança em arquivo/ConfigMap) ou invalidadas por evento `aios.<tenant>.scheduler.config.updated` (interno, não listado em `Events.md` por ser de escopo operacional, não de domínio). Chaves "Não" exigem *rolling restart* coordenado. |
| Validação | Toda chave é validada na carga (`IValidateOptions<T>`); violação de faixa **impede o start** do serviço (fail-fast) — exceto *overrides* por tenant/classe, que são validados no momento do `PUT /v1/scheduler/policy` (rejeitados com `AIOS-SCHED-0005`, ver `API.md`). |
| Autorização de mudança | Toda alteração de chave com escopo `tenant`/`tenant/classe` DEVE passar por PEP (`SchedulingGateway`) → PDP (022) — *default deny* (RFC-0001 §5.8). Chaves de escopo `global`/`cluster` são alteradas apenas via *deployment* (`Deployment.md`), não em runtime por API. |

---

## 2. Catálogo completo de chaves

| Chave | Tipo | Default | Faixa válida | Escopo | Recarregável | Variável de ambiente |
|-------|------|---------|---------------|--------|---------------|------------------------|
| `scheduler.shards.count` | int | `64` | 1–4096 | global/cluster | **Não** (rebalanceamento coordenado com 027) | `Scheduler__Shards__Count` |
| `scheduler.decision.timeout_ms` | int | `15` | 1–100 | global | Sim | `Scheduler__Decision__TimeoutMs` |
| `scheduler.weights.alpha` | double | `0.5` | 0.0–1.0 | tenant | Sim | `Scheduler__Weights__Alpha` |
| `scheduler.weights.beta` | double | `0.3` | 0.0–1.0 | tenant | Sim | `Scheduler__Weights__Beta` |
| `scheduler.weights.gamma` | double | `0.2` | 0.0–1.0 | tenant | Sim | `Scheduler__Weights__Gamma` |
| `scheduler.quality_floor` | double | `0.7` | 0.0–1.0 | tenant/classe | Sim | `Scheduler__QualityFloor` |
| `scheduler.max_concurrency.per_tenant` | int | `1000` | 1–1000000 | tenant | Sim | `Scheduler__MaxConcurrency__PerTenant` |
| `scheduler.max_concurrency.per_class.INTERACTIVE` | int | `400` | 0–1000000 | tenant | Sim | `Scheduler__MaxConcurrency__PerClass__Interactive` |
| `scheduler.max_concurrency.per_class.BATCH` | int | `400` | 0–1000000 | tenant | Sim | `Scheduler__MaxConcurrency__PerClass__Batch` |
| `scheduler.max_concurrency.per_class.BACKGROUND` | int | `150` | 0–1000000 | tenant | Sim | `Scheduler__MaxConcurrency__PerClass__Background` |
| `scheduler.max_concurrency.per_class.SYSTEM` | int | `50` | 0–1000000 | tenant | Sim | `Scheduler__MaxConcurrency__PerClass__System` |
| `scheduler.fairness.min_share` | double | `0.01` | 0.0–1.0 | tenant | Sim | `Scheduler__Fairness__MinShare` |
| `scheduler.aging.factor_per_sec` | double | `1.0` | 0.0–100.0 | classe | Sim | `Scheduler__Aging__FactorPerSec` |
| `scheduler.aging.max_boost` | int | `40` | 0–100 | classe | Sim | `Scheduler__Aging__MaxBoost` |
| `scheduler.preemption.enabled` | bool | `true` | `true`\|`false` | global | Sim | `Scheduler__Preemption__Enabled` |
| `scheduler.preemption.grace_period_ms` | int | `2000` | 0–60000 | classe | Sim | `Scheduler__Preemption__GracePeriodMs` |
| `scheduler.preemption.cooldown_ms` | int | `5000` | 0–120000 | classe | Sim | `Scheduler__Preemption__CooldownMs` |
| `scheduler.backpressure.defer_threshold` | double | `0.80` | 0.0–1.0 | shard | Sim | `Scheduler__Backpressure__DeferThreshold` |
| `scheduler.backpressure.reject_threshold` | double | `0.95` | 0.0–1.0 | shard | Sim | `Scheduler__Backpressure__RejectThreshold` |
| `scheduler.backpressure.retry_after_ms` | int | `1500` | 100–60000 | global | Sim | `Scheduler__Backpressure__RetryAfterMs` |
| `scheduler.deadline.edf_enabled` | bool | `true` | `true`\|`false` | tenant | Sim | `Scheduler__Deadline__EdfEnabled` |
| `scheduler.retry.max_attempts` | int | `3` | 0–10 | classe | Sim | `Scheduler__Retry__MaxAttempts` |
| `scheduler.retry.backoff_base_ms` | int | `500` | 50–60000 | classe | Sim | `Scheduler__Retry__BackoffBaseMs` |
| `scheduler.policy.engine` | enum | `heuristic` | `heuristic`\|`rl` | global/tenant | Sim (interface estável, `ISchedulingPolicy`) | `Scheduler__Policy__Engine` |
| `scheduler.idempotency.ttl_hours` | int | `24` | 24–168 | global | Sim | `Scheduler__Idempotency__TtlHours` |
| `scheduler.capacity.heartbeat_ttl_ms` | int | `5000` | 1000–60000 | global | Sim | `Scheduler__Capacity__HeartbeatTtlMs` |
| `scheduler.lease.ttl_ms` | int | `10000` | 1000–120000 | global | Sim | `Scheduler__Lease__TtlMs` |
| `scheduler.redis.zset_prefix` | string | `sched` | `[a-z0-9_-]+` | global | **Não** | `Scheduler__Redis__ZsetPrefix` |

Pesos DEVEM satisfazer `alpha + beta + gamma ≈ 1` (tolerância ±0.01,
verificado por `CHECK` no PostgreSQL — ver `Database.md` §2.5 — e revalidado
na leitura pelo `PolicyWeightsProvider`, com normalização automática se a soma
divergir dentro da tolerância).

---

## 3. Detalhamento por chave (semântica e impacto operacional)

### 3.1 `scheduler.shards.count` (N)

Define o número de partições determinísticas usadas por `ShardRouter`
(`shard = hash(tenant_id, agent_id) mod N`, R-03/FR-003). Mudança de `N`
**NÃO É recarregável em runtime isoladamente** — exige rebalanceamento
coordenado com `027-Cluster` (hashing consistente, ADR-0092 a propor) para
minimizar migração de filas ativas; é aplicada via *deployment* controlado
(`Deployment.md`), nunca via `PUT` de API.

### 3.2 `scheduler.decision.timeout_ms`

Orçamento de tempo interno que o `AdmissionController`/`PriorityCostEvaluator`
têm para produzir uma decisão antes de aplicar fallback conservador
(`AIOS-SCHED-0012`). Relacionado a NFR-001 (p99 ≤ 20 ms) — este valor é o
teto do *budget* interno, não o SLO externo completo (que inclui rede).

### 3.3 `scheduler.weights.alpha` / `.beta` / `.gamma`

Pesos do score multiobjetivo `C = α·L + β·$ + γ·E` (R-02/FR-002). Escopo
`tenant`: cada tenant PODE ter pesos próprios, persistidos em
`scheduler.policy_weight` (`Database.md` §2.5) e alteráveis via
`PUT /v1/scheduler/policy` (`API.md` §3), sempre atrás de PDP (022).
Mudança é recarregada por `PriorityCostEvaluator` via cache com invalidação
por evento `aios.<tenant>.policy.decision.updated` (consumido, ver
`Events.md` §3.7) ou por *pull* periódico (fallback).

### 3.4 `scheduler.quality_floor` (Q_min)

Piso de qualidade aceitável por tenant/classe (`quality_floor` em
`SchedulingRequest` NÃO PODE ser inferior a este piso administrativo — se for,
a requisição individual pode ainda ser aceita se `quality_floor` da requisição
for **maior ou igual** ao piso; o piso é o mínimo institucional, não o teto).
Violação estrutural (impossível de atingir sob restrições correntes de
custo/latência) resulta em `AIOS-SCHED-0007` (422).

### 3.5 `scheduler.max_concurrency.per_tenant` / `.per_class.*`

Limites *hard* de concorrência, aplicados atomicamente por `QuotaLedger`
(Redis, `Database.md` §5.2). Redução do limite em runtime **NÃO** preempta
tarefas já em execução — apenas impede novas admissões acima do novo teto
(efeito gradual, sem *thrashing*).

### 3.6 `scheduler.fairness.min_share`

Vazão mínima garantida por tenant/classe em janela de 60s (R-09/FR-009,
NFR-006). Trabalha em conjunto com `scheduler.aging.*` — o
`PriorityCostEvaluator` eleva a prioridade efetiva de tarefas de tenants
abaixo do `min_share` até `aging.max_boost`.

### 3.7 `scheduler.aging.factor_per_sec` / `.max_boost`

Taxa de incremento de prioridade por segundo de espera na fila e teto do
incremento (anti-starvation, R-09). Aplicado por classe de serviço — classes
`BACKGROUND` tipicamente toleram `factor_per_sec` maior (aging mais agressivo)
para evitar inanição sob carga de classes prioritárias.

### 3.8 `scheduler.preemption.enabled` / `.grace_period_ms` / `.cooldown_ms`

Controla se `PreemptionManager` está ativo (`enabled=false` desativa
completamente R-04/FR-004 — uso apenas em ambientes de teste/debug, NÃO
DEVERIA ser `false` em produção). `grace_period_ms` é o tempo de aviso antes
da suspensão efetiva (compensável). `cooldown_ms` é o período mínimo entre
duas preempções da mesma tarefa (histerese anti-*thrashing*, ADR-0094 a
propor).

### 3.9 `scheduler.backpressure.defer_threshold` / `.reject_threshold` / `.retry_after_ms`

Limiares de pressão (0–1) que definem os três níveis de backpressure
(`accept < defer(0.80) < reject(0.95)`, R-05/FR-005). `retry_after_ms` é o
valor default sugerido no cabeçalho `Retry-After`/campo `retryAfterMs`
quando o nível é `reject` (`AIOS-SCHED-0003`). `defer_threshold` DEVE ser
estritamente menor que `reject_threshold` (validado no load da configuração).

### 3.10 `scheduler.deadline.edf_enabled`

Ativa a lógica *Earliest-Deadline-First* para requisições com `deadline_at`
(R-12/FR-012, SHOULD). Quando `false`, `deadline_at` ainda é respeitado para
expiração (`EXPIRED`), mas não influencia a ordenação de prioridade.

### 3.11 `scheduler.retry.max_attempts` / `.backoff_base_ms`

Política de retry para tarefas que falham e são retriáveis
(`FAILED → QUEUED`, §4.3 da `StateMachine.md`). Backoff exponencial com
jitter: `delay = backoff_base_ms * 2^attempt * jitter(0.5–1.5)`.

### 3.12 `scheduler.policy.engine`

Seleciona a implementação de `ISchedulingPolicy` (R-13/FR-013):
`heuristic` (v1, `HeuristicPolicy`) ou `rl` (v2, `LearnedPolicy`). A troca
**NÃO DEVE** alterar a superfície de API/eventos — apenas `policyId`/
`policyVersion`/`featureVector` na `SchedulingDecision` (ver `API.md` §8,
RFC-0091 a propor). Escopo `global/tenant`: PODE ser habilitado
progressivamente por tenant (*canary* de política aprendida).

### 3.13 `scheduler.idempotency.ttl_hours`

Piso mínimo de retenção de `IdempotencyRecord` (RFC-0001 §5.5 exige ≥ 24h).
Aplicado tanto ao TTL em Redis (`sched:idem:*`) quanto à coluna `expires_at`
em `scheduler.idempotency_record` (`Database.md` §2.4).

### 3.14 `scheduler.capacity.heartbeat_ttl_ms`

Janela de frescor de `CapacitySnapshot` (Redis, `Database.md` §5.3). Um nó
sem heartbeat dentro desta janela é considerado **fora de serviço** pelo
`CapacityProvider` e excluído do `PlacementEngine` até novo heartbeat.

### 3.15 `scheduler.lease.ttl_ms`

TTL do lease de dispatch por tarefa (`sched:meta:{task_urn}.lease_until`) e do
lease de *ownership* de shard por réplica. Expiração sem confirmação
(`agent.lifecycle.spawned` não observado) aciona `ReconciliationWorker` para
re-enfileirar (evita trabalho perdido sob falha de réplica/Runtime).

### 3.16 `scheduler.redis.zset_prefix`

Prefixo de todas as chaves Redis do módulo (`Database.md` §5). **NÃO
recarregável** — mudança em runtime quebraria a correspondência com chaves já
gravadas; alterado apenas em *deployment* com migração de dados coordenada.

---

## 4. Escopos de configuração — semântica

| Escopo | Significado | Quem aplica/autoriza |
|--------|--------------|------------------------|
| `global/cluster` | Único valor para todo o Scheduler; requer coordenação com 027-Cluster. | Deployment/SRE. |
| `global` | Único valor por *deployment* do Scheduler (todos os tenants); recarregável sem coordenação externa. | Operador do módulo (config file/ConfigMap). |
| `tenant` | Um valor por tenant, com fallback para o default global se ausente. | Tenant admin via `PUT /v1/scheduler/policy`, autorizado por PDP (022). |
| `tenant/classe` | Um valor por (tenant, `service_class`), mais específico que `tenant`. | Idem, com `service_class` no payload. |
| `classe` | Um valor por `service_class`, aplicado a todos os tenants (política de produto, não por cliente). | Operador do módulo (config file). |
| `shard` | Um valor por shard (tipicamente uniforme, mas a arquitetura permite ajuste fino por shard sobrecarregado). | Operador do módulo/SRE. |

---

## 5. Exemplo — `appsettings.json` (trecho)

```json
{
  "Scheduler": {
    "Shards": { "Count": 64 },
    "Decision": { "TimeoutMs": 15 },
    "Weights": { "Alpha": 0.5, "Beta": 0.3, "Gamma": 0.2 },
    "QualityFloor": 0.7,
    "MaxConcurrency": {
      "PerTenant": 1000,
      "PerClass": { "Interactive": 400, "Batch": 400, "Background": 150, "System": 50 }
    },
    "Fairness": { "MinShare": 0.01 },
    "Aging": { "FactorPerSec": 1.0, "MaxBoost": 40 },
    "Preemption": { "Enabled": true, "GracePeriodMs": 2000, "CooldownMs": 5000 },
    "Backpressure": { "DeferThreshold": 0.80, "RejectThreshold": 0.95, "RetryAfterMs": 1500 },
    "Deadline": { "EdfEnabled": true },
    "Retry": { "MaxAttempts": 3, "BackoffBaseMs": 500 },
    "Policy": { "Engine": "heuristic" },
    "Idempotency": { "TtlHours": 24 },
    "Capacity": { "HeartbeatTtlMs": 5000 },
    "Lease": { "TtlMs": 10000 },
    "Redis": { "ZsetPrefix": "sched" }
  }
}
```

## 6. Exemplo — override por variável de ambiente (produção)

```bash
export AIOS_SCHEDULER__WEIGHTS__ALPHA=0.6
export AIOS_SCHEDULER__WEIGHTS__BETA=0.25
export AIOS_SCHEDULER__WEIGHTS__GAMMA=0.15
export AIOS_SCHEDULER__BACKPRESSURE__REJECT_THRESHOLD=0.90
```

## 7. Exemplo — override por tenant via API governada

```
PUT /v1/scheduler/policy
X-AIOS-Tenant: acme
Idempotency-Key: 01J9ZC000000000000000100
{
  "alpha": 0.4,
  "beta": 0.4,
  "gamma": 0.2,
  "qualityFloor": 0.85,
  "engine": "heuristic"
}

→ 200 OK   (persistido em scheduler.policy_weight; trilha em policy_weight_history;
             evento de auditoria emitido para 025 por meio do fluxo de PUT autorizado)
```

---

## 8. Referências

- Fonte de verdade do módulo: `./_DESIGN_BRIEF.md` §8.
- Superfície de API que lê/escreve política em runtime: `./API.md` §3
  (`GET`/`PUT /v1/scheduler/policy`).
- Persistência de pesos e histórico: `./Database.md` §2.5–2.6.
- Contratos centrais (autorização, correlação): `../003-RFC/RFC-0001-Architecture-Baseline.md` §5.6, §5.8.
- Modos de falha associados a limites/timeouts: `./FailureRecovery.md`.
- Modelo de escala (impacto de `shards.count` e leases): `./Scalability.md`.
