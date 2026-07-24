---
Documento: Logging
Módulo: 022-Policy
Status: Draft
Versão: 0.1
Última atualização: 2026-07-23
Responsável (RACI-A): Arquiteto do Módulo 022-Policy
ADRs relacionados: ADR-0010, ADR-0008, ADR-0229
RFCs relacionados: RFC-0001
Depende de: Metrics.md, Database.md (decision_journal), 024-Observability, 025-Audit
---

# 022-Policy — Logging

Log estruturado com **Serilog → Seq** (`../024-Observability/`), correlacionado por
`trace_id`/`span_id` (W3C Trace Context, RFC-0001 §5.6).

## 1. Três destinos distintos — não confundir

| Destino | O que registra | Retenção | Fonte da verdade para |
|---------|----------------|----------|-----------------------|
| **Log de aplicação** (Serilog→Seq) | Operação do serviço: erros, transições, degradações. | 14 dias | Diagnóstico técnico |
| **`policy.decision_journal`** (PostgreSQL) | Decisões: 100% dos `deny`, amostra dos `allow`. | `pol.journal.retention_days` (30 d) | Explicabilidade e simulação |
| **Trilha imutável** (`../025-Audit/`) | Fatos de governança e decisão, imutáveis. | Definida pelo `025` | Prova legal e auditoria |

> São **três** coisas diferentes com aparência de log. O erro comum é usar o log de
> aplicação para explicar decisões — o que produz respostas que dependem de o nível de
> log estar ligado e da retenção de 14 dias. A explicabilidade (NFR-007, NFR-009) é
> responsabilidade do **journal**, que é banco, tem constraint (`ck_allow_has_rule`) e
> é consultável por `decision_id` (ADR-0229).

## 2. Campos obrigatórios em todo log

| Campo | Origem | Exemplo |
|-------|--------|---------|
| `timestamp` | RFC 3339 UTC | `2026-07-23T10:15:00.123Z` |
| `level` | Serilog | `Information` |
| `trace_id` / `span_id` | OTel | `4bf92f3577b34da6a3ce929d0e0e4736` |
| `tenant_id` | `X-AIOS-Tenant` | `acme` |
| `service` | Fixo | `022-policy` |
| `component` | Componente emissor (PascalCase) | `BundleDistributor` |
| `event` | Chave estável do evento de log (§3) | `policy.bundle.activated` |
| `bundle_version` | Quando aplicável | `43` |

## 3. Catálogo de eventos de log

| Componente | `event` | Nível | Quando | Campos adicionais |
|------------|---------|-------|--------|-------------------|
| `DecisionGateway` | `policy.decision.rejected_tenant` | Warning | `tenant` divergente (`AIOS-POL-0003`) | `pep_module`, `claimed_tenant` |
| `DecisionGateway` | `policy.decision.rate_limited` | Warning | `AIOS-POL-0010` | `pep_module`, `current_rate` |
| `DecisionEngine` | `policy.decision.evaluated` | **Debug** | Toda decisão | `decision_id`, `effect`, `reason_code`, `latency_us` |
| `DecisionEngine` | `policy.decision.denied` | Information | `explicit_deny` / `platform_deny` | `decision_id`, `matched_rules`, `pep_module` |
| `DecisionEngine` | `policy.decision.eval_timeout` | Warning | Orçamento estourado | `decision_id`, `stage`, `elapsed_ms` |
| `DecisionEngine` | `policy.decision.allow_without_rule` | **Error** | `allow` sem regra casada (**nunca deveria ocorrer**) | `decision_id`, `pep_module` |
| `AttributeResolver` | `policy.attribute.timeout` | Warning | Timeout de fonte | `source_key`, `criticality`, `timeout_ms` |
| `AttributeResolver` | `policy.attribute.source_degraded` | Error | Fonte `required` marcada `degraded` | `source_key`, `provider_module`, `failure_rate` |
| `BundleCompiler` | `policy.bundle.compiled` | Information | Compilação bem-sucedida | `rule_count`, `compiled_bytes`, `duration_ms` |
| `BundleCompiler` | `policy.bundle.compile_failed` | Warning | `AIOS-POL-0004`/`-0012`/`-0016`/`-0017` | `error_code`, `rule_key` |
| `ConflictAnalyzer` | `policy.bundle.conflict_detected` | Warning | Contradição/sombreamento/inalcançável | `kind`, `rule_keys` |
| `PolicyTestRunner` | `policy.test.failed` | Warning | Caso golden reprovado | `test_key`, `expected`, `actual` |
| `SimulationEngine` | `policy.simulation.completed` | Information | Simulação concluída | `evaluated_count`, `flipped_to_deny`, `flipped_to_allow`, `truncated` |
| `BundleRegistry` | `policy.bundle.transition` | Information | Transição da FSM | `from`, `to`, `transition_id` (T-NN) |
| `BundleRegistry` | `policy.bundle.signature_invalid` | **Error** | Assinatura inválida (`AIOS-POL-0014`) | `stage`, `signed_by` |
| `BundleDistributor` | `policy.bundle.activated` | Information | Bundle → `Active` | `previous_version`, `reason` |
| `BundleDistributor` | `policy.bundle.propagated` | Information | Réplica adotou a versão | `replica`, `propagation_ms` |
| `ExceptionManager` | `policy.exception.granted` | Information | Waiver concedido | `exception_urn`, `requested_by`, `approved_by`, `expires_at` |
| `ExceptionManager` | `policy.exception.expired` | Information | Waiver expirou/revogado | `exception_urn`, `terminated_by` |
| `SelfGovernanceGuard` | `policy.selfgov.denied` | Warning | Operação administrativa negada (`AIOS-POL-0002`) | `operation`, `principal` |
| `SelfGovernanceGuard` | `policy.selfgov.dual_approval_rejected` | Warning | SoD violada (`AIOS-POL-0018`) | `operation`, `authored_by`, `approved_by` |
| `DecisionJournal` | `policy.journal.dropped` | **Error** | Registro descartado | `reason`, `effect` |
| `EventEmitter` | `policy.outbox.publish_failed` | Warning | Falha de publicação | `subject`, `attempt` |

`policy.decision.evaluated` é **Debug** por desenho: em 50.000 decisões/s por réplica,
registrá-lo em `Information` produziria mais log do que o sistema consegue transportar
e não acrescentaria nada — a decisão já está no journal, com constraint e consulta.

## 4. Registro de decisões (o journal)

O `DecisionJournal` (`./Database.md` §7) grava, **assincronamente** e fora do caminho
de resposta:

| Política | Regra |
|----------|-------|
| `deny` | **100%**, sem amostragem, sem configuração que desligue (FR-015). |
| `allow` | Amostrado por `pol.journal.allow_sample_rate` (5%). |
| Conteúdo | URN, ação, `reason_code`, `matched_rules`, `bundle_version`, `attributes_digest`, `latency_us`, `pep_module`, `trace_id`. |
| Proibido | Valores de atributo `sensitive`, claims, conteúdo do recurso (FR-023). |

Consulta típica de plantão:

```sql
SELECT decided_at, subject_urn, action, reason_code, matched_rules, bundle_version
  FROM policy.decision_journal
 WHERE tenant_id = 'acme'
   AND effect = 'deny'
   AND decided_at >= now() - interval '15 minutes'
 ORDER BY decided_at DESC
 LIMIT 100;
```

Para uma decisão específica, prefira `ExplainDecision` (`./API.md` §3.4): ele reproduz
a avaliação sobre a versão exata do bundle, em vez de mostrar apenas o resultado.

## 5. Correlação ponta a ponta

```
   PEP (006-Kernel)                022-Policy                   025-Audit
        │ traceparent: 00-4bf9…-00f0…-01                             │
        ├─ span: kernel.syscall.spawn                                │
        │        │                                                    │
        │        └─▶ span: policy.evaluate  ────────────────────────▶ │
        │              ├─ span: attribute.resolve{source=principal}   │
        │              ├─ span: rbac.resolve                          │
        │              ├─ span: abac.evaluate                         │
        │              └─ (assíncrono) journal.write                  │
        │                                                             │
        └─ log kernel: decision_id=01J9ZB…  ──── mesmo decision_id ──▶ trilha
```

O `decision_id` é a chave que costura os três destinos (§1): aparece na resposta ao
PEP, no log de aplicação, no journal e na trilha imutável. Um plantonista que tenha o
`decision_id` de um `deny` consegue, em três consultas, saber **o que** foi negado,
**por qual regra** e **sob qual versão de política**.

## 6. Níveis e o que NÃO logar

| Nível | Uso |
|-------|-----|
| `Error` | Rompimento de invariante (`allow_without_rule`, assinatura inválida, journal descartado), fonte `required` degradada. |
| `Warning` | Negações administrativas, timeouts, conflitos, testes reprovados, limites atingidos. |
| `Information` | Transições de bundle, ativações, propagações, waivers, simulações. |
| `Debug` | Decisões individuais, resolução de atributo por chamada. |
| `Verbose` | Não usado em produção. |

**Nunca em log:** valores de atributo `sensitive`, claims completas, conteúdo do
recurso, `connection string`, material de assinatura. A varredura automatizada de
FR-023 cobre o pipeline de log tanto quanto o de eventos e métricas.

## 7. Retenção e expurgo

| Destino | Retenção | Expurgo |
|---------|----------|---------|
| Seq (log de aplicação) | 14 dias | Automático |
| `policy.decision_journal` | 30 dias (`pol.journal.retention_days`) | `DROP PARTITION` diário |
| `policy.policy_exception` | 1 ano após `expires_at` | Job de manutenção |
| Trilha imutável | Definida pelo `../025-Audit/` | Fora deste módulo |

Direito ao esquecimento: pseudonimização de `subject_urn` no journal ao consumir
`security.principal.disabled` (`./Security.md` §6) — o registro do fato permanece; a
associação com a pessoa, não.

## 8. Referências

- Métricas: `./Metrics.md` · Alertas e runbooks: `./Monitoring.md`
- Journal (modelo físico): `./Database.md` §7 · Explicação: `./API.md` §3.4
- Trilha imutável: `../025-Audit/` · Plataforma: `../024-Observability/`

*Fim de `Logging.md`.*
