---
Documento: Configuration
Módulo: 022-Policy
Status: Draft
Versão: 0.1
Última atualização: 2026-07-23
Responsável (RACI-A): Arquiteto do Módulo 022-Policy
ADRs relacionados: ADR-0008, ADR-0221, ADR-0226, ADR-0227, ADR-0228
RFCs relacionados: RFC-0001, RFC-0220, RFC-0221
Depende de: _DESIGN_BRIEF.md §8, NonFunctionalRequirements.md, Deployment.md
---

# 022-Policy — Configuração

Prefixo de variável de ambiente: **`AIOS_POL_`**. A chave `pol.decision.default_ttl_ms`
corresponde a `AIOS_POL_DECISION_DEFAULT_TTL_MS`.

Escopos: `global` (instalação), `tenant`, `bundle`, `attribute_source`. Quando uma
chave existe em mais de um escopo, vale a **mais específica**, limitada pelo teto do
escopo mais geral — um tenant **NÃO DEVE** conseguir configurar-se para além do que a
plataforma permite.

## 1. Chaves de decisão (caminho quente)

| Chave | Tipo | Default | Faixa | Escopo | Recarregável | Descrição |
|-------|------|---------|-------|--------|--------------|-----------|
| `pol.decision.default_ttl_ms` | int | 3000 | 0–60000 | global/tenant | sim | TTL sugerido para `allow` no cache do PEP. |
| `pol.decision.deny_ttl_ms` | int | 1000 | 0–10000 | global/tenant | sim | TTL para `deny`. **DEVE** ser ≤ o de `allow` (revogação rápida). |
| `pol.decision.eval_budget_ms` | int | 50 | 5–500 | global | sim | Orçamento total de avaliação; estouro ⇒ `deny` (`AIOS-POL-0011`, NFR-010). |
| `pol.decision.batch_max_size` | int | 64 | 1–512 | global | sim | Itens por `EvaluateBatch` (`AIOS-POL-0016`). |
| `pol.decision.max_eval_per_s_per_tenant` | int | 100000 | 100–1000000 | tenant | sim | Limite de avaliações (`AIOS-POL-0010`). |
| `pol.obligation.max_per_decision` | int | 8 | 1–32 | global | sim | Obrigações por decisão (`AIOS-POL-0019`). |

> `deny_ttl_ms ≤ default_ttl_ms` é validado na carga da configuração: inverter os dois
> faria a negação durar mais que a permissão no cache, e uma correção de política
> levaria mais tempo para restaurar acesso do que para removê-lo — exatamente o
> oposto do desejável em um incidente.

## 2. Chaves de bundle e publicação

| Chave | Tipo | Default | Faixa | Escopo | Recarregável | Descrição |
|-------|------|---------|-------|--------|--------------|-----------|
| `pol.bundle.max_rules` | int | 10000 | 100–100000 | global | sim | Regras por bundle (`AIOS-POL-0016`). |
| `pol.bundle.max_compiled_mb` | int | 64 | 1–512 | global | sim | Tamanho do artefato compilado. |
| `pol.bundle.signature_required` | bool | `true` | {true,false} | global | **não** | Exige assinatura válida para ativar (`AIOS-POL-0014`, NFR-015). |
| `pol.bundle.propagation_target_s` | int | 10 | 1–120 | global | sim | Meta de propagação do bundle ativo (NFR-004). |
| `pol.bundle.rollback_retention_versions` | int | 10 | 1–100 | tenant | sim | Versões `Superseded` retidas (invariante I6). |
| `pol.bundle.dual_approval_required` | bool | `true` | {true,false} | tenant | **não** | Exige `approved_by ≠ authored_by` (`AIOS-POL-0018`). |
| `pol.conflict.mode` | enum | `strict` | {strict,warn} | tenant | sim | Conflito bloqueia a compilação ou apenas anota. |

## 3. Chaves de teste e simulação

| Chave | Tipo | Default | Faixa | Escopo | Recarregável | Descrição |
|-------|------|---------|-------|--------|--------------|-----------|
| `pol.test.min_coverage` | decimal | 0.90 | 0.0–1.0 | tenant | sim | Cobertura mínima de regras para publicar (NFR-011). |
| `pol.test.required_before_publish` | bool | `true` | {true,false} | global | **não** | Torna o gate de teste obrigatório. |
| `pol.simulation.required_before_publish` | bool | `true` | {true,false} | tenant | sim | Exige simulação quando há bundle ativo (T-08). |
| `pol.simulation.max_requests` | int | 1000000 | 1000–10000000 | tenant | sim | Teto de requisições reavaliadas; excedente marca `truncated`. |

> `pol.test.required_before_publish` e `pol.bundle.dual_approval_required` são
> **não-recarregáveis** de propósito: são os dois controles que impedem política sem
> prova e política sem revisão de entrar em vigor. Se pudessem ser desligados em
> runtime, seriam desligados exatamente no momento de pressa em que mais importam.

## 4. Chaves de atributos

| Chave | Tipo | Default | Faixa | Escopo | Recarregável | Descrição |
|-------|------|---------|-------|--------|--------------|-----------|
| `pol.attribute.resolve_timeout_ms` | int | 20 | 1–200 | attribute_source | sim | Timeout de resolução por fonte. |
| `pol.attribute.cache_ttl_s` | int | 30 | 0–3600 | attribute_source | sim | TTL do cache de atributo. |
| `pol.attribute.fail_mode` | enum | `closed` | {closed,open} | global | sim | Comportamento com fonte `required` indisponível. |

`pol.attribute.fail_mode = open` **NÃO DEVERIA** ser habilitado em produção sem
exceção documentada em ADR: ele transforma "não consegui verificar" em "pode".

## 5. Chaves de governança e journal

| Chave | Tipo | Default | Faixa | Escopo | Recarregável | Descrição |
|-------|------|---------|-------|--------|--------------|-----------|
| `pol.exception.max_ttl_h` | int | 72 | 1–720 | tenant | sim | Prazo máximo de um waiver (`AIOS-POL-0013`). |
| `pol.exception.require_dual_approval` | bool | `true` | {true,false} | tenant | **não** | Aprovador distinto do solicitante. |
| `pol.journal.allow_sample_rate` | decimal | 0.05 | 0.0–1.0 | tenant | sim | Amostragem de `allow` no journal. |
| `pol.journal.retention_days` | int | 30 | 1–365 | tenant | sim | Retenção do journal (a prova longa é do `../025-Audit/`). |
| `pol.deny_event.sample_rate` | decimal | 0.1 | 0.0–1.0 | tenant | sim | Amostragem de `policy.decision.denied` para `no_matching_rule`. |
| `pol.selfgov.root_bundle_immutable` | bool | `true` | {true,false} | global | **não** | Bundle-raiz só muda por cerimônia (`./Security.md` §2.3). |

> **`deny` no journal não é configurável.** `pol.journal.allow_sample_rate` amostra
> apenas `allow`; 100% dos `deny` são gravados (FR-015, NFR-007). Amostrar negações
> tornaria impossível responder "por que meu agente foi bloqueado?" — o caso em que
> a explicabilidade mais importa.
>
> Da mesma forma, `pol.deny_event.sample_rate` amostra **apenas** `no_matching_rule`
> (alto volume, baixa informação); `explicit_deny` e `platform_deny` são sempre
> emitidos.

## 6. Infraestrutura e dependências

| Chave | Tipo | Default | Escopo | Recarregável | Descrição |
|-------|------|---------|--------|--------------|-----------|
| `pol.db.connection_string` | secret | — | global | **não** | PostgreSQL (schema `policy`); credencial dinâmica do `../021-Security/`. |
| `pol.db.command_timeout_ms` | int | 3000 | global | sim | Timeout de comando (fora do caminho quente). |
| `pol.redis.connection_string` | secret | — | global | **não** | Cache de atributos e contadores de limite. |
| `pol.nats.url` | string | — | global | **não** | Barramento (`../020-Communication/`). |
| `pol.nats.request_subject_prefix` | string | `aios` | global | **não** | Prefixo dos subjects (RFC-0001 §5.3). |
| `pol.outbox.poll_interval_ms` | int | 200 | global | sim | Intervalo do `policy-outbox-relay`. |
| `pol.otel.endpoint` | string | — | global | **não** | Coletor OTel (`../024-Observability/`). |
| `pol.audit.emit_enabled` | bool | `true` | global | **não** | Emissão da trilha para `../025-Audit/` (ADR-0010). |

Segredos **NÃO DEVEM** vir de arquivo em imagem: são injetados em runtime pelo
`../021-Security/` (`./Deployment.md` §5).

## 7. Precedência e validação

```
   default do produto
        ↓ sobrescrito por
   configuração global da instalação
        ↓ sobrescrito por
   configuração do tenant        (limitada pelo teto global)
        ↓ sobrescrito por
   configuração de bundle / attribute_source
```

Validações na carga (falham o *startup*, não o primeiro pedido):

| # | Validação | Erro |
|---|-----------|------|
| V-01 | `pol.decision.deny_ttl_ms` ≤ `pol.decision.default_ttl_ms` | *startup* recusado |
| V-02 | Soma dos `resolve_timeout_ms` das fontes `required` ≤ `pol.decision.eval_budget_ms` | `AIOS-POL-0012` no registro da fonte |
| V-03 | `pol.test.min_coverage` ∈ [0,1] e, se `required_before_publish`, > 0 | *startup* recusado |
| V-04 | `pol.exception.max_ttl_h` ≤ teto global | valor truncado ao teto + alerta |
| V-05 | Chave não-recarregável alterada em runtime | ignorada + log `Warning` + métrica |

## 8. Exemplo — `appsettings.Production.json`

```json
{
  "Policy": {
    "Decision": { "DefaultTtlMs": 3000, "DenyTtlMs": 1000, "EvalBudgetMs": 50,
                  "BatchMaxSize": 64, "MaxEvalPerSPerTenant": 100000 },
    "Bundle":   { "MaxRules": 10000, "MaxCompiledMb": 64, "SignatureRequired": true,
                  "PropagationTargetS": 10, "RollbackRetentionVersions": 10,
                  "DualApprovalRequired": true },
    "Test":     { "MinCoverage": 0.90, "RequiredBeforePublish": true },
    "Simulation": { "RequiredBeforePublish": true, "MaxRequests": 1000000 },
    "Attribute":  { "FailMode": "closed" },
    "Exception":  { "MaxTtlH": 72, "RequireDualApproval": true },
    "Journal":    { "AllowSampleRate": 0.05, "RetentionDays": 30 },
    "SelfGov":    { "RootBundleImmutable": true }
  }
}
```

Equivalente por variável de ambiente:

```bash
AIOS_POL_DECISION_EVAL_BUDGET_MS=50
AIOS_POL_BUNDLE_SIGNATURE_REQUIRED=true
AIOS_POL_TEST_MIN_COVERAGE=0.90
AIOS_POL_ATTRIBUTE_FAIL_MODE=closed
AIOS_POL_EXCEPTION_MAX_TTL_H=72
```

## 9. Perfis por ambiente

| Chave | Produção | Homologação | Desenvolvimento |
|-------|----------|-------------|-----------------|
| `pol.attribute.fail_mode` | `closed` | `closed` | `open` (apenas local) |
| `pol.conflict.mode` | `strict` | `strict` | `warn` |
| `pol.test.required_before_publish` | `true` | `true` | `false` |
| `pol.simulation.required_before_publish` | `true` | `true` | `false` |
| `pol.journal.allow_sample_rate` | 0.05 | 0.5 | 1.0 |
| `pol.bundle.dual_approval_required` | `true` | `true` | `false` |

> Um ambiente de desenvolvimento com `fail_mode = open` é aceitável; um ambiente de
> homologação com ele **não** é — homologação existe para reproduzir o comportamento
> de produção, inclusive o comportamento sob falha.

## 10. Referências

- Fonte única de verdade: `./_DESIGN_BRIEF.md` §8
- Metas associadas: `./NonFunctionalRequirements.md`
- Erros: `./API.md` §6 · Implantação: `./Deployment.md`
- Segredos: `../021-Security/` · Observabilidade: `./Monitoring.md`

*Fim de `Configuration.md`.*
