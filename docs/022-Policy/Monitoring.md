---
Documento: Monitoring
Módulo: 022-Policy
Status: Draft
Versão: 0.1
Última atualização: 2026-07-23
Responsável (RACI-A): Arquiteto do Módulo 022-Policy + SRE
ADRs relacionados: ADR-0010, ADR-0008, ADR-0221
RFCs relacionados: RFC-0001
Depende de: Metrics.md, NonFunctionalRequirements.md, FailureRecovery.md, 024-Observability, 029-Operations
---

# 022-Policy — Monitoramento

## 1. Golden signals

| Sinal | Métrica | Alvo |
|-------|---------|------|
| **Latência** | `aios_pol_decision_latency_ms` p99 | ≤ 5 ms (cache) / ≤ 15 ms (atributo remoto) — NFR-001 |
| **Tráfego** | `rate(aios_pol_decisions_total[1m])` | ≤ 50.000/s por réplica — NFR-002 |
| **Erros** | `AIOS-POL-*` por status; `aios_pol_eval_timeout_total` | ≤ 0,1% das requisições |
| **Saturação** | CPU, `aios_pol_bundle_compiled_bytes`, `aios_pol_outbox_pending` | CPU < 60%; outbox < 1.000 |

> Atenção à leitura: **taxa de `deny` não é taxa de erro**. `deny` é o produto normal
> do módulo. O que se monitora é a *variação* da taxa de negação (§4,
> `PolicyDenyRateSpike`), porque um salto indica política recém-publicada com efeito
> não previsto — não indisponibilidade.

## 2. Rastreamento (traces)

100% das decisões geram span OTel (NFR-014), com atributos:

| Atributo do span | Origem |
|------------------|--------|
| `aios.tenant` | `X-AIOS-Tenant` |
| `aios.policy.decision_id` | Resposta |
| `aios.policy.effect` / `reason_code` | Resposta |
| `aios.policy.bundle_version` | `BundleRuntime` |
| `aios.policy.pep_module` | Chamador |
| `aios.policy.attribute_sources` | Fontes consultadas (chaves, **não** valores) |

Spans filhos: `attribute.resolve` (por fonte), `rbac.resolve`, `abac.evaluate`,
`obligation.compose`. O span de journal é **irmão assíncrono**, não filho do caminho
de resposta (invariante estrutural S6, `./ClassDiagrams.md` §5).

## 3. SLO / SLI e error budget

| SLO | SLI | Janela | Error budget |
|-----|-----|--------|--------------|
| Disponibilidade ≥ 99,99% (NFR-003) | Fração de `Evaluate` respondidos sem erro `5xx` | 30 d | 4,3 min/mês |
| p99 ≤ 5 ms (NFR-001) | `aios_pol_decision_latency_p99_cached` | 7 d | 1% das janelas de 5 min |
| Propagação ≤ 10 s (NFR-004) | `aios_pol_bundle_propagation_ms` p99 | 30 d | 1% das publicações |
| Explicabilidade 100% (NFR-007) | `deny` com `reason_code` / total de `deny` | 30 d | **0** (não há budget) |
| *Default deny* (NFR-008) | `aios_pol_allow_without_rule_total` | contínua | **0** (não há budget) |

Consumo de error budget acima de 50% na janela suspende publicações não urgentes de
bundle (política operacional de `../029-Operations/`).

## 4. Alertas Prometheus

```yaml
groups:
- name: policy-critical
  rules:
  - alert: PolicyAllowWithoutRule
    expr: increase(aios_pol_allow_without_rule_total[5m]) > 0
    for: 0m
    labels: { severity: critical, page: immediate }
    annotations:
      summary: "allow emitido sem regra casada — default deny rompido"
      runbook: "./Monitoring.md#rb-p-01"

  - alert: PolicySignatureVerificationFailed
    expr: increase(aios_pol_signature_verification_failures_total[5m]) > 0
    for: 0m
    labels: { severity: critical, page: immediate }
    annotations:
      summary: "Falha de verificação de assinatura de bundle"
      runbook: "./Monitoring.md#rb-p-02"

  - alert: PolicyNoActiveBundle
    expr: increase(aios_pol_no_active_bundle_total[5m]) > 0
    for: 2m
    labels: { severity: critical, page: immediate }
    annotations:
      summary: "Tenant sem bundle ativo: tudo está sendo negado"
      runbook: "./Monitoring.md#rb-p-03"

  - alert: PolicyBundleVersionSkew
    expr: aios_pol_bundle_version_skew > 0
    for: 10m      # > pol.bundle.propagation_target_s (10 s) com folga operacional
    labels: { severity: critical }
    annotations:
      summary: "Réplicas servindo versões diferentes de bundle"
      runbook: "./Monitoring.md#rb-p-04"

- name: policy-warning
  rules:
  - alert: PolicyDecisionLatencyHigh
    expr: aios_pol_decision_latency_p99_cached > 5
    for: 10m
    labels: { severity: warning }
    annotations: { summary: "p99 de decisão acima do SLO (NFR-001)", runbook: "./Monitoring.md#rb-p-05" }

  - alert: PolicyDenyRateSpike
    expr: |
      (aios_pol_deny_ratio - aios_pol_deny_ratio offset 1h) > 0.20
    for: 5m
    labels: { severity: warning }
    annotations: { summary: "Salto na taxa de negação — provável publicação com efeito não previsto", runbook: "./Monitoring.md#rb-p-06" }

  - alert: PolicyEvalTimeoutHigh
    expr: rate(aios_pol_eval_timeout_total[5m]) > 1
    for: 5m
    labels: { severity: warning }
    annotations: { summary: "Avaliações estourando o orçamento de tempo (NFR-010)", runbook: "./Monitoring.md#rb-p-07" }

  - alert: PolicyAttributeSourceDegraded
    expr: aios_pol_attribute_source_status{status="degraded"} == 1
    for: 5m
    labels: { severity: warning }
    annotations: { summary: "Fonte de atributo degradada — decisões podem cair em deny", runbook: "./Monitoring.md#rb-p-08" }

  - alert: PolicyExceptionChurnHigh
    expr: increase(aios_pol_exception_grants_total{outcome="granted"}[30d]) > 3
    for: 1h
    labels: { severity: warning }
    annotations: { summary: "Waivers renovados repetidamente — deveria virar regra revisada", runbook: "./Monitoring.md#rb-p-09" }

  - alert: PolicyJournalLagHigh
    expr: histogram_quantile(0.99, rate(aios_pol_journal_write_lag_ms_bucket[5m])) > 5000
    for: 10m
    labels: { severity: warning }
    annotations: { summary: "Journal atrasado — explicabilidade recente em risco", runbook: "./Monitoring.md#rb-p-10" }

  - alert: PolicyOutboxBacklog
    expr: aios_pol_outbox_pending > 1000
    for: 5m
    labels: { severity: warning }
    annotations: { summary: "Outbox acumulando — invalidação de cache atrasada", runbook: "./Monitoring.md#rb-p-11" }

  - alert: PolicyRateLimitedHigh
    expr: rate(aios_pol_rate_limited_total[5m]) > 10
    for: 10m
    labels: { severity: warning }
    annotations: { summary: "Tenant atingindo o limite de avaliações", runbook: "./Monitoring.md#rb-p-12" }
```

## 5. Runbooks

| ID | Alerta | Procedimento |
|----|--------|--------------|
| **RB-P-01** | `PolicyAllowWithoutRule` | 1) Tratar como **incidente de segurança P1**. 2) Identificar `pep_module` e `bundle_version` pelo journal. 3) Congelar publicações do tenant. 4) Verificar integridade do artefato (`compiled_digest` vs. registro). 5) Se divergente, **rollback imediato** (UC-007) e acionar `../021-Security/`. 6) Revisar o motor de combinação contra os casos golden. |
| **RB-P-02** | `PolicySignatureVerificationFailed` | 1) Verificar se houve rotação de chave de assinatura no `../021-Security/` (evento `security.jwks.rotated`). 2) Se sim, recarregar a chave pública e reverificar. 3) Se não, tratar como **adulteração**: isolar a réplica, preservar o artefato para perícia, acionar CISO. A réplica já se retirou do balanceamento (readiness). |
| **RB-P-03** | `PolicyNoActiveBundle` | 1) Confirmar em `policy.policy_bundle` se existe registro `Active` para o tenant. 2) Se foi arquivamento indevido, ativar a última versão íntegra por rollback (T-12). 3) Se é tenant novo, publicar o bundle inicial (T-09, bootstrap). **Não** desabilitar o *default deny* como contorno. |
| **RB-P-04** | `PolicyBundleVersionSkew` | 1) Identificar as réplicas divergentes por `aios_pol_active_bundle_version{replica}`. 2) Retirá-las do balanceamento. 3) Verificar consumo do stream `POLICY_BUNDLE` e o `policy-outbox-relay`. 4) Forçar recarga do registro. 5) Reintegrar apenas após convergência. |
| **RB-P-05** | `PolicyDecisionLatencyHigh` | 1) Verificar `path`: se `remote_attrs` dominar, a causa é fonte de atributo (ir para RB-P-08). 2) Checar `aios_pol_bundle_rules` e `aios_pol_bundle_compiled_bytes` — bundle recém-publicado maior? 3) Checar CPU/GC. 4) Escalar réplicas só se a saturação for confirmada; escalar não resolve regra patológica. |
| **RB-P-06** | `PolicyDenyRateSpike` | 1) Correlacionar o horário com `policy.bundle.updated`. 2) Abrir o diferencial da simulação correspondente (`GET /v1/policy/simulations/{urn}`) e comparar `flipped_to_deny` com o observado. 3) Se o efeito não estava previsto, **rollback** (UC-007). 4) Se estava previsto e é desejado, silenciar o alerta com anotação e prazo. |
| **RB-P-07** | `PolicyEvalTimeoutHigh` | 1) Identificar o `stage` predominante. 2) Se for `attribute`, ir para RB-P-08. 3) Se for avaliação, procurar regra com condição custosa (`aios_pol_compile_duration_ms` e complexidade do último bundle). 4) Corrigir a regra em nova versão — **não** aumentar `eval_budget_ms` como "correção". |
| **RB-P-08** | `PolicyAttributeSourceDegraded` | 1) Identificar `source_key` e `provider_module`. 2) Verificar a saúde do módulo provedor. 3) Avaliar se a criticidade está correta: atributo marcado `required` que poderia ser `optional` transforma degradação alheia em negação própria. 4) **Não** mudar `fail_mode` para `open` como contorno. |
| **RB-P-09** | `PolicyExceptionChurnHigh` | 1) Listar os waivers do par (sujeito, ação). 2) Levar à revisão de política: o padrão indica regra faltante ou regra errada. 3) Abrir bundle candidato com a regra corrigida (UC-003..UC-006). |
| **RB-P-10** | `PolicyJournalLagHigh` | 1) Verificar saúde do PostgreSQL e o tamanho do buffer de escrita. 2) Confirmar que **nenhum `deny` foi descartado** (`aios_pol_journal_dropped_total`); descarte de `deny` é incidente. 3) Reduzir temporariamente `pol.journal.allow_sample_rate` para aliviar. |
| **RB-P-11** | `PolicyOutboxBacklog` | 1) Verificar o líder do `policy-outbox-relay`. 2) Checar conectividade com NATS e limites do stream. 3) Enquanto houver backlog, os PEPs podem estar com cache desatualizado — considerar reduzir TTL globalmente até normalizar. |
| **RB-P-12** | `PolicyRateLimitedHigh` | 1) Identificar o tenant. 2) Verificar se o PEP daquele tenant está com cache desabilitado ou TTL zerado. 3) Ajustar `max_eval_per_s_per_tenant` só após confirmar que o tráfego é legítimo. |

## 6. Dashboards

| Painel | Conteúdo | Público |
|--------|----------|---------|
| **Policy — Decisão** | p50/p95/p99 por `path` e `surface`; decisões/s; taxa de `deny` por `pep_module`; top `reason_code` | SRE |
| **Policy — Saúde do bundle** | `active_bundle_version` por réplica; propagação p99; transições da FSM; conflitos detectados | SRE + segurança |
| **Policy — Atributos** | Latência e `outcome` por `source_key`; cache hit ratio; fontes degradadas | SRE |
| **Policy — Governança** | Cobertura de teste; diferencial de simulações recentes; waivers ativos e concessões; rejeições por SoD | Segurança + auditoria |
| **Policy — Conformidade** | `allow_without_rule` (deve ser 0); `deny` sem motivo (deve ser 0); assinatura verificada | Auditoria |

## 7. Referências

- Métricas: `./Metrics.md` · Logs: `./Logging.md`
- Metas: `./NonFunctionalRequirements.md` · Falhas: `./FailureRecovery.md`
- Plantão e escalonamento: `../029-Operations/` · Plataforma: `../024-Observability/`

*Fim de `Monitoring.md`.*
