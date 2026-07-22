---
Documento: Configuration
Módulo: 007-Agent-Runtime
Status: Draft
Versão: 0.1
Última atualização: 2026-07-21
Responsável (RACI-A): Arquiteto do Módulo 007 — Agent Runtime
ADRs relacionados: ADR-0073, ADR-0077, ADR-0078
RFCs relacionados: RFC-0001
Depende de: ./_DESIGN_BRIEF.md, ../021-Security/, ../022-Policy/, ../026-Cost-Optimizer/
---

# 007-Agent-Runtime — Configuration

> Todas as chaves abaixo são as **mesmas** definidas no `_DESIGN_BRIEF.md`
> §8, sem alteração de nome, default, faixa ou escopo. Escopo: **G**=global
> (todo o cluster), **T**=por tenant, **A**=por agente (override mais
> específico vence). Variáveis de ambiente são prefixadas `AIOS_RT_`;
> reload é aplicado sem *restart* do processo quando indicado (o
> `ConfigWatcher` interno observa mudanças e reconfigura os componentes
> afetados).

## Índice

1. Fontes de configuração e precedência
2. Catálogo completo de chaves
3. Exemplo de arquivo de configuração (YAML)
4. Recarga em runtime (hot reload)
5. Validação e valores inválidos
6. Referências

---

## 1. Fontes de Configuração e Precedência

Da menor para a maior precedência (a última vence):

1. **Default embutido** no código do runtime (valores desta tabela).
2. **Arquivo de configuração global** montado no processo (`/etc/aios/runtime.yaml`),
   gerido por `021-Security`/infraestrutura (ConfigMap).
3. **Override por tenant**, resolvido no boot via chamada a `022-Policy`/config
   service e reavaliado a cada `aios.<t>.policy.decision.updated` para as
   chaves marcadas `Reload=sim`.
4. **Override por agente**, presente no próprio `AgentSpec` recebido em
   `Boot` (aplica-se apenas às chaves de escopo `A`).
5. **Variável de ambiente** `AIOS_RT_<CHAVE_EM_MAIUSCULAS_COM_UNDERSCORE>`,
   usada primariamente em desenvolvimento/teste local — **NÃO DEVERIA** ser a
   via primária em produção multi-tenant (não é escopada por tenant).

---

## 2. Catálogo Completo de Chaves

| Chave | Default | Faixa | Escopo | Reload | Variável de ambiente |
|-------|---------|-------|--------|--------|-------------------------|
| `runtime.pool.warm_size` | 32 | 0–1024 | G/T | sim | `AIOS_RT_RUNTIME_POOL_WARM_SIZE` |
| `runtime.pool.max_sessions_per_node` | 500 | 1–5000 | G | sim | `AIOS_RT_RUNTIME_POOL_MAX_SESSIONS_PER_NODE` |
| `runtime.cold_start.target_ms` | 250 | 50–2000 | G | não | `AIOS_RT_RUNTIME_COLD_START_TARGET_MS` |
| `loop.max_steps` | 50 | 1–1000 | T/A | sim | `AIOS_RT_LOOP_MAX_STEPS` |
| `loop.max_recursion_depth` | 8 | 1–64 | T/A | sim | `AIOS_RT_LOOP_MAX_RECURSION_DEPTH` |
| `loop.max_tool_fanout` | 8 | 1–64 | T/A | sim | `AIOS_RT_LOOP_MAX_TOOL_FANOUT` |
| `loop.step_timeout_ms` | 30000 | 1000–600000 | T/A | sim | `AIOS_RT_LOOP_STEP_TIMEOUT_MS` |
| `loop.wall_clock_budget_ms` | 900000 | 1000–86400000 | T/A | sim | `AIOS_RT_LOOP_WALL_CLOCK_BUDGET_MS` |
| `budget.max_tokens` | 200000 | 100–10000000 | T/A | sim | `AIOS_RT_BUDGET_MAX_TOKENS` |
| `budget.max_cost_usd` | 5.00 | 0.01–1000 | T/A | sim | `AIOS_RT_BUDGET_MAX_COST_USD` |
| `model.default_timeout_ms` | 60000 | 1000–600000 | T | sim | `AIOS_RT_MODEL_DEFAULT_TIMEOUT_MS` |
| `model.fallback_enabled` | true | bool | T | sim | `AIOS_RT_MODEL_FALLBACK_ENABLED` |
| `tool.invocation_timeout_ms` | 30000 | 1000–300000 | T/A | sim | `AIOS_RT_TOOL_INVOCATION_TIMEOUT_MS` |
| `tool.circuit_breaker.error_threshold` | 5 | 1–100 | T | sim | `AIOS_RT_TOOL_CIRCUIT_BREAKER_ERROR_THRESHOLD` |
| `tool.retry.max_attempts` | 3 | 0–10 | T/A | sim | `AIOS_RT_TOOL_RETRY_MAX_ATTEMPTS` |
| `tool.retry.backoff_base_ms` | 200 | 10–10000 | T | sim | `AIOS_RT_TOOL_RETRY_BACKOFF_BASE_MS` |
| `sandbox.profile` | `net-restricted` | `default\|net-restricted\|no-net\|custom` | T/A | não | `AIOS_RT_SANDBOX_PROFILE` |
| `sandbox.cpu_millicores` | 1000 | 100–8000 | T/A | não | `AIOS_RT_SANDBOX_CPU_MILLICORES` |
| `sandbox.mem_mib` | 512 | 128–8192 | T/A | não | `AIOS_RT_SANDBOX_MEM_MIB` |
| `sandbox.pids_max` | 256 | 16–4096 | T/A | não | `AIOS_RT_SANDBOX_PIDS_MAX` |
| `sandbox.fs_writable_mib` | 128 | 0–4096 | T/A | não | `AIOS_RT_SANDBOX_FS_WRITABLE_MIB` |
| `sandbox.egress_allowlist` | `[]` | lista `host:porta` | T/A | sim | `AIOS_RT_SANDBOX_EGRESS_ALLOWLIST` |
| `checkpoint.enabled` | true | bool | T | sim | `AIOS_RT_CHECKPOINT_ENABLED` |
| `checkpoint.interval_steps` | 5 | 1–100 | T/A | sim | `AIOS_RT_CHECKPOINT_INTERVAL_STEPS` |
| `checkpoint.max_size_mib` | 64 | 1–1024 | T | sim | `AIOS_RT_CHECKPOINT_MAX_SIZE_MIB` |
| `events.outbox.flush_interval_ms` | 100 | 10–5000 | G | sim | `AIOS_RT_EVENTS_OUTBOX_FLUSH_INTERVAL_MS` |
| `events.outbox.max_retries` | 10 | 0–100 | G | sim | `AIOS_RT_EVENTS_OUTBOX_MAX_RETRIES` |
| `heartbeat.interval_ms` | 2000 | 500–30000 | G | sim | `AIOS_RT_HEARTBEAT_INTERVAL_MS` |
| `watchdog.stuck_timeout_ms` | 120000 | 5000–600000 | G | sim | `AIOS_RT_WATCHDOG_STUCK_TIMEOUT_MS` |
| `telemetry.sampling_ratio` | 1.0 | 0.0–1.0 | G/T | sim | `AIOS_RT_TELEMETRY_SAMPLING_RATIO` |
| `pii.redaction.enabled` | true | bool | G/T | sim | `AIOS_RT_PII_REDACTION_ENABLED` |
| `runtime.pep.decision_cache_ttl_ms` | 5000 | 0–60000 | T | sim | `AIOS_RT_RUNTIME_PEP_DECISION_CACHE_TTL_MS` |

> **Nota de rastreabilidade.** `runtime.pep.decision_cache_ttl_ms` instancia o
> padrão "PEP com cache de decisão + invalidação por evento" decidido no brief
> (§2.1, componente `CapabilityEnforcer`; ADR-0078 a propor). A chave está
> registrada na tabela de chaves do `_DESIGN_BRIEF.md` §8 (fonte única de
> verdade), com default/faixa/escopo idênticos aos desta tabela.

### 2.1 Descrição e efeito colateral de cada chave

| Chave | Componente afetado | Efeito ao mudar |
|-------|-----------------------|--------------------|
| `runtime.pool.warm_size` | Runtime Supervisor (fora deste módulo) | Tamanho do pool quente; afeta diretamente `cold_start_ms` (NFR-001). Não é recarregado *pelo* runtime — é lido no boot do Supervisor. |
| `runtime.pool.max_sessions_per_node` | `RuntimeBootstrapper` (rejeita `Boot` além do limite) | Limite de densidade por nó (NFR-004); excesso retorna `AIOS-RUNTIME-0001` do lado do Supervisor. |
| `runtime.cold_start.target_ms` | Métrica de referência apenas (não aplica limite rígido) | Usado como *label* de comparação em `Benchmark.md`; **não recarregável** pois é apenas meta de referência fixada no design. |
| `loop.max_steps` | `QuotaGovernor` | Reduzir o valor pode interromper sessões em andamento no próximo passo avaliado (nunca retroativamente). |
| `loop.max_recursion_depth` | `QuotaGovernor` | Limita profundidade de sub-planejamento/replanejamento aninhado. |
| `loop.max_tool_fanout` | `QuotaGovernor`, `ActionExecutor` | Limita paralelismo de invocação de tools por passo. |
| `loop.step_timeout_ms` | `CognitiveLoopEngine` | Timeout por passo individual (perceive/think/act/observe); não confundir com `watchdog.stuck_timeout_ms` (nível de loop). |
| `loop.wall_clock_budget_ms` | `QuotaGovernor` | Orçamento total de tempo de parede da sessão. |
| `budget.max_tokens` / `budget.max_cost_usd` | `QuotaGovernor` | Cotas locais consumidas por `ModelCall`/`ToolInvocation`; esgotamento aciona UC-007. |
| `model.default_timeout_ms` | `ModelRouterClient` | Timeout de cada chamada de inferência. |
| `model.fallback_enabled` | `ModelRouterClient` | Habilita/desabilita fallback automático de modelo (UC-012). |
| `tool.invocation_timeout_ms` | `ToolInvoker` | Timeout de cada invocação de tool. |
| `tool.circuit_breaker.error_threshold` | `ToolInvoker` | Nº de falhas na janela de avaliação que abre o *circuit breaker* (UC-013, NFR-016). |
| `tool.retry.max_attempts` / `tool.retry.backoff_base_ms` | `ToolInvoker` | Política de retry com backoff exponencial + *jitter*. |
| `sandbox.profile` | `SandboxManager` | Seleciona o `SandboxProfile` aplicado no boot; mudança exige novo `Boot` (não recarregável em sessão ativa — trocar o perfil de isolamento de um processo já isolado não é seguro). |
| `sandbox.cpu_millicores`/`mem_mib`/`pids_max`/`fs_writable_mib` | `SandboxManager` (`cgroups v2`) | Limites físicos aplicados no boot; não recarregáveis por definição de `cgroups`. |
| `sandbox.egress_allowlist` | `SandboxManager` | Lista de `host:porta` permitidos; recarregável via reconciliação de política de rede, sem novo boot completo (aplicação incremental de regras de firewall/eBPF). |
| `checkpoint.enabled` | `CheckpointManager` | Desabilitar remove a via de suspensão graciosa; esgotamento de cota passa a encerrar a sessão diretamente. |
| `checkpoint.interval_steps` | `CheckpointManager` | Frequência de checkpoints incrementais; afeta diretamente o RPO (NFR-006). |
| `checkpoint.max_size_mib` | `CheckpointManager` | Limite de tamanho do blob de working memory antes de acionar compressão/erro. |
| `events.outbox.flush_interval_ms` | `EventPublisher` | Frequência de tentativa de publicação do outbox local; afeta latência de propagação de eventos (não a durabilidade). |
| `events.outbox.max_retries` | `EventPublisher` | Tentativas antes de considerar o evento "preso" (alerta operacional). |
| `heartbeat.interval_ms` | `SupervisorChannel`/`HealthProbe` | Frequência de heartbeat; afeta a velocidade de detecção de instância `dead` pelo Supervisor. |
| `watchdog.stuck_timeout_ms` | `HealthProbe` | Tempo sem progresso antes de considerar o loop travado (UC-015). |
| `telemetry.sampling_ratio` | `TelemetryEmitter` | Taxa de amostragem de traces (1.0 = 100%); reduzir custo de observabilidade sob alta escala sem perder métricas agregadas. |
| `pii.redaction.enabled` | `ObservationCollector`, `EventPublisher` | Desabilitar é **desaconselhado em produção**; usado apenas em ambientes de teste isolados (NFR-013). |
| `runtime.pep.decision_cache_ttl_ms` | `CapabilityEnforcer` | TTL do cache local de decisões de *capability*; reduzir aumenta chamadas ao PDP (mais atual, mais latência); aumentar reduz latência do caminho quente às custas de atualidade (mitigado pela invalidação por evento, ADR-0078). |

---

## 3. Exemplo de Arquivo de Configuração (YAML)

```yaml
# /etc/aios/runtime.yaml — default global, sobrescrito por tenant/agente
runtime:
  pool:
    warm_size: 32
    max_sessions_per_node: 500
  cold_start:
    target_ms: 250

loop:
  max_steps: 50
  max_recursion_depth: 8
  max_tool_fanout: 8
  step_timeout_ms: 30000
  wall_clock_budget_ms: 900000

budget:
  max_tokens: 200000
  max_cost_usd: 5.00

model:
  default_timeout_ms: 60000
  fallback_enabled: true

tool:
  invocation_timeout_ms: 30000
  circuit_breaker:
    error_threshold: 5
  retry:
    max_attempts: 3
    backoff_base_ms: 200

sandbox:
  profile: net-restricted
  cpu_millicores: 1000
  mem_mib: 512
  pids_max: 256
  fs_writable_mib: 128
  egress_allowlist: []

checkpoint:
  enabled: true
  interval_steps: 5
  max_size_mib: 64

events:
  outbox:
    flush_interval_ms: 100
    max_retries: 10

heartbeat:
  interval_ms: 2000

watchdog:
  stuck_timeout_ms: 120000

telemetry:
  sampling_ratio: 1.0

pii:
  redaction:
    enabled: true

# Override por tenant (exemplo — tenant "acme" com orçamento mais generoso)
tenants:
  acme:
    budget:
      max_tokens: 500000
      max_cost_usd: 15.00
    sandbox:
      egress_allowlist:
        - "api.internal-tool.acme.com:443"
```

---

## 4. Recarga em Runtime (Hot Reload)

- O `ConfigWatcher` interno assina mudanças de configuração por tenant
  (evento `aios.<t>.policy.decision.updated` cobre também *bundles* de
  configuração operacional quando aplicável, e um canal de config dedicado
  fora do escopo deste módulo cobre config pura).
- Chaves marcadas `Reload=sim` são aplicadas ao **próximo passo do loop**
  (nunca no meio de um passo em andamento), evitando estado inconsistente.
- Chaves marcadas `Reload=não` (ex.: `sandbox.*`, `runtime.cold_start.target_ms`)
  só têm efeito em uma **nova** materialização (`Boot`); uma sessão ativa
  nunca troca de perfil de sandbox em execução — trocar limites de
  isolamento de um processo já em execução comprometeria a garantia de
  isolamento por construção (Princípio 1, `./Vision.md` §5).

---

## 5. Validação e Valores Inválidos

- Toda chave é validada contra sua **faixa** no carregamento (boot e
  reload); um valor fora da faixa é rejeitado e o runtime mantém o valor
  anterior válido, emitindo alerta operacional (não aplica o config
  potencialmente perigoso).
- Chaves de escopo `A` (por agente) só são aceitas se dentro da faixa
  **e** dentro do limite máximo permitido pelo escopo `T` (por tenant) —
  um agente não pode se auto-conceder um orçamento maior que o teto do
  tenant.
- Validação de `sandbox.egress_allowlist` inclui checagem de formato
  `host:porta` e recusa de *wildcards* amplos (`*:*`) fora de perfis
  explicitamente marcados como `custom` e aprovados por `022-Policy`.

---

## 6. Referências

- Fonte canônica das chaves: `./_DESIGN_BRIEF.md` §8.
- Componentes afetados por cada chave: `./Architecture.md` §5.
- Metas de SLO que dependem destas chaves: `./NonFunctionalRequirements.md`.
- Perfis de sandbox referenciados por `sandbox.profile`: `./Security.md`, `../021-Security/`.
