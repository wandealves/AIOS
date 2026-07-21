---
Documento: Configuration.md
Módulo: 006-Kernel
Status: Draft
Versão: 0.1
Última atualização: 2026-07-20
Responsável (RACI-A): Arquiteto do Módulo 006-Kernel
ADRs relacionados: ADR-0062, ADR-0063, ADR-0065
RFCs relacionados: RFC-0001 (baseline), RFC-0007 (ACB & Quota Model, a propor)
Depende de: 001-Architecture, 003-RFC/RFC-0001, 022-Policy, 026-Cost-Optimizer, 028-Deployment
---

# 006-Kernel — Configuration

> **Escopo.** Catálogo completo das chaves de configuração do Kernel: tipo,
> default, faixa válida, escopo de aplicação (`global`/`tenant`/`agent`),
> se é recarregável em runtime (*hot reload*) e a variável de ambiente
> correspondente. Fonte normativa: `_DESIGN_BRIEF.md` §8. **Nenhum valor aqui
> contradiz o brief** — este documento apenas detalha mecânica de
> carregamento, precedência, validação e exemplos operacionais.

---

## 1. Convenções de configuração do Kernel

| Aspecto | Regra |
|---------|-------|
| Prefixo de env var | `AIOS_KERNEL_` + chave em maiúsculas com `_` no lugar de `.` (ex.: `kernel.spawn.boot_timeout_ms` → `AIOS_KERNEL_SPAWN_BOOT_TIMEOUT_MS`). |
| Formato de arquivo | `appsettings.json` (.NET 10 `IConfiguration`) + overlay por ambiente (`appsettings.Production.json`) + variáveis de ambiente (precedência mais alta) + overrides por tenant em PostgreSQL (`kernel.quota`, tabela de config futura `kernel.tenant_config`). |
| Precedência (maior→menor) | 1) Override por agente (quando aplicável, ex. `quota_override` no `spawn`) → 2) Override por tenant (config em banco) → 3) Variável de ambiente → 4) `appsettings.{Environment}.json` → 5) `appsettings.json` (defaults do brief). |
| Hot reload | Chaves marcadas **sim** são observadas via `IOptionsMonitor<T>` (.NET) e reagem a `aios.<tenant>.policy.bundle.updated`/config change sem reiniciar o processo. Chaves **não** exigem *rolling restart* coordenado. |
| Validação | Toda chave é validada contra a faixa declarada na inicialização (*fail-fast*) e em cada *reload*; valor fora da faixa é rejeitado e o valor anterior é mantido, com log de erro estruturado (`Logging.md`). |
| Escopo `global` | Aplica-se a todas as réplicas do Kernel Service no cluster. |
| Escopo `tenant` | Pode ser sobreposto por tenant (ex.: tenant com SLA premium tem `boot_timeout_ms` maior). |
| Escopo `agent` | Pode ser sobreposto por agente individual no momento do `spawn` (via `quotaOverride`/`labels`), respeitando tetos definidos no escopo `tenant`. |

---

## 2. Catálogo completo de chaves

| Chave | Tipo | Default | Faixa válida | Escopo | Recarregável | Variável de ambiente | Descrição |
|-------|------|---------|--------------|--------|---------------|------------------------|-----------|
| `kernel.spawn.admission_timeout_ms` | int | `200` | 50–2000 | global/tenant | sim | `AIOS_KERNEL_SPAWN_ADMISSION_TIMEOUT_MS` | Timeout de espera pela resposta de admissão do `009-Scheduler` (T-02/T-03). Excedido → `AIOS-KERNEL-0003`. |
| `kernel.spawn.boot_timeout_ms` | int | `5000` | 500–30000 | global/tenant | sim | `AIOS_KERNEL_SPAWN_BOOT_TIMEOUT_MS` | Timeout de materialização do runtime pelo `008-Agent-Lifecycle` (T-04/T-05). Excedido → `AIOS-KERNEL-0004`. |
| `kernel.pep.decision_cache_ttl_ms` | int | `3000` | 0–60000 | global | sim | `AIOS_KERNEL_PEP_DECISION_CACHE_TTL_MS` | TTL do cache de decisões do `CapabilityEnforcer` (PEP). `0` desabilita cache (toda syscall consulta o PDP). |
| `kernel.pep.fail_mode` | enum | `closed` | `{closed, open}` | global/tenant | sim | `AIOS_KERNEL_PEP_FAIL_MODE` | Comportamento quando o PDP (`022-Policy`) está indisponível. `closed` nega (`AIOS-CAP-0003`); `open` permite com alerta — **DEVERIA** permanecer `closed` em produção (postura *default deny*, RFC-0001 §5.8). |
| `kernel.quota.default_tokens` | bigint | `1000000` | ≥ 0 | tenant/agent | sim | `AIOS_KERNEL_QUOTA_DEFAULT_TOKENS` | Cota default de tokens LLM por janela quando não há override explícito. |
| `kernel.quota.default_cost_usd` | numeric | `10.0` | ≥ 0 | tenant/agent | sim | `AIOS_KERNEL_QUOTA_DEFAULT_COST_USD` | Orçamento default por janela; sobreposto por `026-Cost-Optimizer` via evento `cost.budget.updated`. |
| `kernel.quota.syscalls_per_sec` | int | `200` | 1–100000 | agent | sim | `AIOS_KERNEL_QUOTA_SYSCALLS_PER_SEC` | Rate-limit de syscalls por agente aplicado pelo `SyscallGateway`. Excedido → `AIOS-QUOTA-0002`. |
| `kernel.quota.window` | duration | `1h` | 1m–24h | tenant/agent | sim | `AIOS_KERNEL_QUOTA_WINDOW` | Janela de reset das cotas (token-bucket). |
| `kernel.quota.enforcement` | enum | `hard` | `{hard, soft}` | tenant/agent | sim | `AIOS_KERNEL_QUOTA_ENFORCEMENT` | `hard` rejeita ao exceder; `soft` permite e emite `agent.quota.warning`/marca para alerta. |
| `kernel.quota.warning_threshold_pct` | int | `80` | 1–100 | global/tenant | sim | `AIOS_KERNEL_QUOTA_WARNING_THRESHOLD_PCT` | Percentual do limite que dispara `agent.quota.warning` (ver `Events.md` §2.2.11). |
| `kernel.hibernation.idle_ttl_ms` | int | `300000` | 0–86400000 | global/tenant | sim | `AIOS_KERNEL_HIBERNATION_IDLE_TTL_MS` | Tempo de ociosidade (`now - last_active_at`) até `Suspended`→`Hibernated` (T-08). `0` desabilita hibernação automática. |
| `kernel.runtime.heartbeat_deadline_ms` | int | `15000` | 1000–120000 | global | sim | `AIOS_KERNEL_RUNTIME_HEARTBEAT_DEADLINE_MS` | Deadline sem heartbeat do runtime antes de `Running`→`Failed` (T-12). |
| `kernel.acb.shard_count` | int | `64` | 1–4096 | global | **não** | `AIOS_KERNEL_ACB_SHARD_COUNT` | Número de shards lógicos (`hash(tenant,agent) mod N`). Alterar exige migração de reparticionamento (`Database.md` §7) — **NÃO** aplicável via hot reload. |
| `kernel.outbox.relay_interval_ms` | int | `50` | 5–1000 | global | sim | `AIOS_KERNEL_OUTBOX_RELAY_INTERVAL_MS` | Intervalo de polling do relay do Outbox (`EventEmitter`) para publicação no JetStream. |
| `kernel.idempotency.retention_h` | int | `48` | 24–720 | global | sim | `AIOS_KERNEL_IDEMPOTENCY_RETENTION_H` | Retenção de resultados idempotentes; **piso normativo de 24h** por RFC-0001 §5.5 — valor abaixo de 24 é rejeitado na validação. |
| `kernel.acb.optimistic_retry_max` | int | `5` | 0–20 | global | sim | `AIOS_KERNEL_ACB_OPTIMISTIC_RETRY_MAX` | Retentativas automáticas em conflito de OCC (`version`) antes de retornar `AIOS-KERNEL-0009` ao chamador. |
| `kernel.broker.circuit_breaker_threshold` | float | `0.5` | 0–1 | global | sim | `AIOS_KERNEL_BROKER_CIRCUIT_BREAKER_THRESHOLD` | Taxa de erro (janela móvel) que abre o circuit breaker do `ResourceBrokerRouter` por dependência (010/011/012/015/017). |
| `kernel.broker.circuit_breaker_window_s` | int | `30` | 5–300 | global | sim | `AIOS_KERNEL_BROKER_CIRCUIT_BREAKER_WINDOW_S` | Janela móvel usada para calcular a taxa de erro do circuit breaker. |
| `kernel.broker.request_timeout_ms` | int | `8000` | 500–60000 | global/tenant | sim | `AIOS_KERNEL_BROKER_REQUEST_TIMEOUT_MS` | Timeout de chamada do `ResourceBrokerRouter` a um módulo dono de recurso. Excedido → `AIOS-KERNEL-0005`. |
| `kernel.gateway.max_request_body_kb` | int | `256` | 1–10240 | global | sim | `AIOS_KERNEL_GATEWAY_MAX_REQUEST_BODY_KB` | Tamanho máximo de payload aceito pelo `SyscallGateway` (proteção contra abuso). |
| `kernel.checkpoint.max_size_mib` | int | `64` | 1–1024 | global/tenant | sim | `AIOS_KERNEL_CHECKPOINT_MAX_SIZE_MIB` | Tamanho máximo de snapshot aceito pelo `CheckpointManager` antes de rejeitar `checkpoint` com `AIOS-KERNEL-0010`. |
| `kernel.acb.redis_cache_enabled` | bool | `true` | `{true, false}` | global | sim | `AIOS_KERNEL_ACB_REDIS_CACHE_ENABLED` | Habilita a projeção quente do ACB em Redis (`AcbStore`). Desabilitar força leitura direta ao PostgreSQL (modo degradado, maior latência). |

---

## 3. Diagrama de precedência de configuração (ASCII)

```
   ┌────────────────────────────────────────────────────────────────────┐
   │  Resolução de valor efetivo para uma chave em uma dada syscall      │
   └────────────────────────────────────────────────────────────────────┘

   spawn(tenant=acme, agent=..., quotaOverride={tokensLimit: 750000})
                              │
                              ▼
        ┌─────────────────────────────────────────┐
        │ 1. Override por AGENTE (quotaOverride,   │  maior precedência
        │    labels no payload da syscall)         │
        └─────────────────┬─────────────────────────┘
                           │ ausente? desce
                           ▼
        ┌─────────────────────────────────────────┐
        │ 2. Override por TENANT (kernel.quota      │
        │    com scope=tenant; kernel.tenant_config)│
        └─────────────────┬─────────────────────────┘
                           │ ausente? desce
                           ▼
        ┌─────────────────────────────────────────┐
        │ 3. Variável de ambiente                   │
        │    (AIOS_KERNEL_QUOTA_DEFAULT_TOKENS)     │
        └─────────────────┬─────────────────────────┘
                           │ ausente? desce
                           ▼
        ┌─────────────────────────────────────────┐
        │ 4. appsettings.{Environment}.json         │
        └─────────────────┬─────────────────────────┘
                           │ ausente? desce
                           ▼
        ┌─────────────────────────────────────────┐
        │ 5. appsettings.json (default do brief)    │  menor precedência
        │    ex.: kernel.quota.default_tokens=1e6   │
        └─────────────────────────────────────────┘
```

---

## 4. Exemplos de configuração por ambiente

### 4.1 `appsettings.json` (defaults — refletem exatamente o brief §8)

```json
{
  "Kernel": {
    "Spawn": {
      "AdmissionTimeoutMs": 200,
      "BootTimeoutMs": 5000
    },
    "Pep": {
      "DecisionCacheTtlMs": 3000,
      "FailMode": "closed"
    },
    "Quota": {
      "DefaultTokens": 1000000,
      "DefaultCostUsd": 10.0,
      "SyscallsPerSec": 200,
      "Window": "01:00:00",
      "Enforcement": "hard",
      "WarningThresholdPct": 80
    },
    "Hibernation": {
      "IdleTtlMs": 300000
    },
    "Runtime": {
      "HeartbeatDeadlineMs": 15000
    },
    "Acb": {
      "ShardCount": 64,
      "OptimisticRetryMax": 5,
      "RedisCacheEnabled": true
    },
    "Outbox": {
      "RelayIntervalMs": 50
    },
    "Idempotency": {
      "RetentionH": 48
    },
    "Broker": {
      "CircuitBreakerThreshold": 0.5,
      "CircuitBreakerWindowS": 30,
      "RequestTimeoutMs": 8000
    },
    "Gateway": {
      "MaxRequestBodyKb": 256
    },
    "Checkpoint": {
      "MaxSizeMib": 64
    }
  }
}
```

### 4.2 Override via variáveis de ambiente (tenant premium, produção)

```bash
# Tenant "acme" opera com SLA mais folgado para materialização de agentes
# e cota de tokens maior — aplicado como override de ambiente do deployment
# dedicado (ver 028-Deployment/Deployment.md para isolamento por tenant).
export AIOS_KERNEL_SPAWN_BOOT_TIMEOUT_MS=8000
export AIOS_KERNEL_QUOTA_DEFAULT_TOKENS=5000000
export AIOS_KERNEL_QUOTA_ENFORCEMENT=hard
export AIOS_KERNEL_PEP_FAIL_MODE=closed
```

### 4.3 Override por syscall (`spawn` com `quotaOverride`)

```json
POST /v1/kernel/agents
{
  "priority": 2,
  "policyRef": "urn:aios:acme:policy:high-priority-agent",
  "quotaOverride": {
    "tokensLimit": 2000000,
    "costUsdLimit": 25.0,
    "syscallsPerSec": 500
  },
  "labels": { "slaClass": "premium" }
}
```

O `quotaOverride` acima **DEVE** ser validado pelo `ResourceQuotaManager`
contra o teto definido no escopo `tenant` (não pode exceder o orçamento
definido por `026-Cost-Optimizer`); excesso solicitado retorna
`AIOS-KERNEL-0010` (payload inválido — pedido de cota acima do teto do
tenant) antes mesmo de tentar `spawn`.

---

## 5. Chaves não recarregáveis — impacto operacional

| Chave | Por que não é recarregável | Procedimento para alterar |
|-------|-------------------------------|------------------------------|
| `kernel.acb.shard_count` | `shard_key` é uma coluna `GENERATED` no PostgreSQL calculada como `hash(...) % N` (ver `Database.md` §2.2); mudar `N` em runtime deixaria `shard_key` inconsistente entre linhas antigas e novas. | Executar migração de reparticionamento controlada (recalcular `shard_key` para todas as linhas, redistribuir cache Redis por afinidade), documentada como runbook em `../029-Operations/`; requer janela de manutenção ou migração *online* em duas fases (dupla escrita + backfill). |

Todas as demais chaves listadas no §2 são observadas via
`IOptionsMonitor<T>` e aplicadas **sem reinício de processo**; a propagação
para todas as réplicas do Kernel Service ocorre por invalidação de cache
local acionada por evento interno de configuração (não um subject NATS
público — mecanismo interno ao processo de configuração do control plane).

---

## 6. Validação e comportamento em falha

- Na inicialização, o Kernel Service **DEVE** validar todas as chaves contra
  as faixas do §2; violação de faixa em qualquer chave **DEVE** impedir o
  boot do serviço (*fail-fast*), com log estruturado indicando a chave e o
  valor rejeitado.
- Em *hot reload*, um valor fora da faixa **DEVE** ser rejeitado mantendo o
  valor efetivo anterior, emitindo log de nível `Error` e métrica
  `aios_kernel_config_reload_rejected_total{key}` (ver `Metrics.md`).
- `kernel.idempotency.retention_h` **DEVE** ser rejeitado se configurado
  abaixo de 24 (piso normativo da RFC-0001 §5.5), independentemente da
  origem do valor (env var, banco ou arquivo).
- `kernel.pep.fail_mode = open` **DEVERIA** emitir um alerta de segurança
  contínuo (`Monitoring.md`) enquanto ativo, dado que relaxa a postura
  *default deny* exigida pela RFC-0001 §5.8.

---

## 7. Referências

- Fonte normativa das chaves e seus defaults: `./_DESIGN_BRIEF.md` §8.
- Consumo das chaves pelos componentes: `./Architecture.md` (componentes `SyscallGateway`, `CapabilityEnforcer`, `ResourceQuotaManager`, `LifecycleCoordinator`, `EventEmitter`, `CheckpointManager`).
- Impacto de `kernel.acb.shard_count` no modelo físico: `./Database.md` §4, §7.
- Consumo de `kernel.pep.fail_mode`/`kernel.pep.decision_cache_ttl_ms` no fluxo do PEP: `./SequenceDiagrams.md`.
- Métricas de configuração/reload: `./Metrics.md`.
- Topologia de deployment e overlays por ambiente: `../028-Deployment/`.
