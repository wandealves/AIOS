---
Documento: StateMachine
Módulo: 022-Policy
Status: Draft
Versão: 0.1
Última atualização: 2026-07-23
Responsável (RACI-A): Arquiteto do Módulo 022-Policy
ADRs relacionados: ADR-0008, ADR-0221, ADR-0223, ADR-0224, ADR-0226, ADR-0227
RFCs relacionados: RFC-0001, RFC-0220, RFC-0221
Depende de: _DESIGN_BRIEF.md §4, Database.md, Events.md, UseCases.md
---

# 022-Policy — Máquina de Estados

## 1. Qual entidade tem ciclo de vida

A entidade governada por máquina de estados neste módulo é o **`PolicyBundle`** — não
a decisão. A decisão é **efêmera**: existe pelo TTL no cache do PEP e é reconstruível
a partir de (`bundle_version`, `DecisionRequest`, `attributes_digest`). O bundle, ao
contrário, é o artefato que precisa de autoria, prova, publicação e reversão.

Tratar política como **release** — e não como linha de configuração — é o que torna
possível responder "o que mudou na segurança do sistema entre ontem e hoje?"
(ADR-0221).

Entidades com ciclo secundário e mais simples: `PolicyException` (§7) e
`SimulationRun` (§8).

## 2. Estados (`BundleState`)

| Estado | Descrição | Terminal? | Ação de entrada | Ação de saída |
|--------|-----------|-----------|-----------------|---------------|
| `Draft` | Em autoria; regras, papéis e vínculos mutáveis. | não | Reserva `bundle_version` | Congela o conteúdo |
| `Validating` | Compilação e análise de conflitos em curso. | não | Agenda compilação | Grava `compiled_digest` ou motivo da rejeição |
| `Validated` | Compila, sem conflito bloqueante; **imutável**. | não | Marca imutabilidade (`AIOS-POL-0020` em edição) | — |
| `Tested` | Casos golden obrigatórios aprovados, cobertura ≥ mínima. | não | Grava `test_coverage` | — |
| `Simulated` | Reavaliado contra tráfego registrado; diferencial disponível. | não | Vincula `simulation_run` | — |
| `Active` | Em vigor. No máximo **um** por `(tenant, scope)`. | não | Grava `activated_at`; emite `policy.bundle.updated`; dispara propagação | Emite evento de troca |
| `Superseded` | Substituído por versão posterior; retido para rollback. | não | Grava `superseded_by` | — |
| `RolledBack` | Estava ativo e foi revertido. | não | Emite `policy.bundle.rolledback` | — |
| `Rejected` | Reprovado na compilação, no conflito ou nos testes. | **sim** | Emite `policy.bundle.rejected` | — |
| `Archived` | Fora da janela de retenção de rollback. | **sim** | Libera o artefato compilado | — |

## 3. Transições, gatilhos e guardas

| # | De → Para | Gatilho | Guarda (DEVE ser verdadeira) | Erro se violada |
|---|-----------|---------|------------------------------|-----------------|
| T-01 | ∅ → `Draft` | `CreateBundle` | Capability `pol:bundle:author` ∧ tenant do autor = tenant do bundle. | `AIOS-POL-0002`, `AIOS-POL-0003` |
| T-02 | `Draft` → `Validating` | `ValidateBundle` | `rule_count` ≤ `pol.bundle.max_rules` ∧ toda regra com `description` não vazia. | `AIOS-POL-0016` |
| T-03 | `Validating` → `Validated` | compilação concluída | Sintaxe válida ∧ referências resolvidas ∧ herança acíclica ∧ (sem conflito ∨ `pol.conflict.mode = warn`). | — |
| T-04 | `Validating` → `Rejected` | compilação falhou | Erro de sintaxe ∨ referência inexistente ∨ conflito em `strict` ∨ complexidade excedida. | `AIOS-POL-0004`, `-0012`, `-0008`, `-0016`, `-0017` |
| T-05 | `Validated` → `Tested` | `RunPolicyTests` | 100% dos casos `is_mandatory` passaram ∧ `test_coverage` ≥ `pol.test.min_coverage`. | `AIOS-POL-0007` |
| T-06 | `Validated` → `Rejected` | testes reprovados | Caso obrigatório falhou ∨ cobertura abaixo do mínimo. | `AIOS-POL-0007` |
| T-07 | `Tested` → `Simulated` | `RunSimulation` | Existe bundle `Active` como linha de base ∧ janela dentro da retenção do journal. | `AIOS-POL-0005` |
| T-08 | `Simulated` → `Active` | `PublishBundle` | Capability `pol:bundle:publish` ∧ `approved_by ≠ authored_by` ∧ assinatura válida ∧ diferencial revisado. | `AIOS-POL-0018`, `-0014` |
| T-09 | `Tested` → `Active` | `PublishBundle` (`skip_simulation`) | **Somente** primeiro bundle do tenant (sem linha de base) ∨ escopo `platform` em *bootstrap* — sempre com aprovação dupla registrada. | `AIOS-POL-0005` |
| T-10 | `Active` → `Superseded` | ativação de versão posterior | Sucessora em `Active` na mesma transação ∧ `superseded_by` preenchido. | `AIOS-POL-0006` |
| T-11 | `Active` → `RolledBack` | `RollbackBundle` | Capability `pol:bundle:rollback` ∧ alvo em `Superseded` dentro da retenção ∧ alvo passa a `Active` na mesma transação. | `AIOS-POL-0015` |
| T-12 | `Superseded` → `Active` | `RollbackBundle` (alvo) | Artefato compilado e assinatura ainda válidos ∧ dentro de `pol.bundle.rollback_retention_versions`. | `AIOS-POL-0014`, `-0015` |
| T-13 | `Superseded`/`RolledBack` → `Archived` | expurgo de retenção | Fora da janela de rollback ∧ nenhuma simulação ou explicação pendente referenciando a versão. | — |
| T-14 | `Draft` → `Rejected` | `DiscardBundle` | Capability `pol:bundle:author`. | `AIOS-POL-0002` |

## 4. Diagrama de estados (ASCII)

```
        CreateBundle (T-01)
   ∅ ──────────────────────▶ ┌─────────┐ ── DiscardBundle (T-14) ──┐
                              │  Draft  │                            │
                              └────┬────┘                            │
                    Validate (T-02)│                                 │
                                   ▼                                 │
                            ┌────────────┐   falha (T-04)            │
                            │ Validating │──────────────────────────▶│
                            └─────┬──────┘                           ▼
                       compila ok │(T-03)                     ┌────────────┐
                                  ▼                           │  Rejected  │ (terminal)
                            ┌────────────┐  testes falham     └────────────┘
                            │ Validated  │──────(T-06)───────────────▲
                            └─────┬──────┘                           │
                     testes ok (T-05)                                │
                                  ▼                                  │
                            ┌────────────┐                           │
                            │   Tested   │───── bootstrap (T-09) ────┼──┐
                            └─────┬──────┘                           │  │
                    simulação (T-07)                                 │  │
                                  ▼                                  │  │
                            ┌────────────┐                           │  │
                            │ Simulated  │                           │  │
                            └─────┬──────┘                           │  │
                     publica (T-08)                                  │  │
                                  ▼                                  │  ▼
                            ┌────────────┐  nova versão (T-10)  ┌──────────────┐
                            │   Active   │─────────────────────▶│  Superseded  │
                            └─────┬──────┘                      └──────┬───────┘
                     rollback (T-11)                    rollback alvo (T-12)
                                  ▼                                   │
                            ┌────────────┐                            │
                            │ RolledBack │                            │
                            └─────┬──────┘                            │
                                  │      expurgo de retenção (T-13)   │
                                  └───────────────┬───────────────────┘
                                                  ▼
                                          ┌────────────┐
                                          │  Archived  │ (terminal)
                                          └────────────┘
```

## 5. Invariantes

| ID | Invariante | Como é garantida |
|----|-----------|------------------|
| I1 | Existe **no máximo um** bundle `Active` por `(tenant_id, scope)`. | Índice único parcial `UNIQUE(tenant_id, scope) WHERE state='Active'` (`./Database.md` §3.1) — garantia física, não apenas de aplicação. |
| I2 | O bundle é **imutável** a partir de `Validated`. | Edição rejeitada com `AIOS-POL-0020`; alteração exige nova `bundle_version`. |
| I3 | `Active` só é alcançável de `Simulated` (T-08), `Tested` (T-09, *bootstrap*) ou `Superseded` (T-12, rollback). | Guardas da FSM + teste de contrato; publicar o não-testado é o defeito que este módulo existe para impedir. |
| I4 | Toda transição para `Active` grava a mudança **e** o evento `policy.bundle.updated` na mesma transação. | Outbox transacional (`./Events.md` §5). |
| I5 | Sem bundle `Active` para o `(tenant, scope)`, o PDP responde `deny` a tudo, com `reason_code = no_active_bundle`. | Verificação na entrada do `DecisionEngine`; teste T-POL-019. |
| I6 | Versões em `Superseded` retêm o artefato compilado íntegro enquanto estiverem na janela de rollback. | Expurgo só em T-13; sem o artefato, nem rollback nem explicação de decisões passadas são possíveis. |
| I7 | Toda transição ocorre sob OCC pelo `version` esperado. | Conflito → `AIOS-POL-0006`. |

> **Por que I5 é uma invariante e não um default:** um tenant sem política não é um
> tenant liberado. Se a ausência de bundle produzisse `allow`, a forma mais rápida de
> desabilitar a segurança do AIOS seria apagar um registro.

## 6. Precedência canônica de combinação (ciclo de decisão)

A decisão **não** tem máquina de estados persistida, mas tem um algoritmo
determinístico — a mesma entrada produz sempre a mesma saída (NFR-009):

```
   1. bundle ausente/inativo?  ──sim──▶ deny (no_active_bundle)   [fim]
                │ não
                ▼
   2. deny explícito de escopo `platform` casou?  ──sim──▶ deny (platform_deny)  [fim]
                │ não
                ▼
   3. menor `priority` decide; empate ⇒ deny vence allow
                │
                ▼
   4. exceção (waiver) vigente e regra dispensável?  ──sim──▶ allow (exception_applied)
                │ não
                ▼
   5. nenhuma regra casou?  ──sim──▶ deny (no_matching_rule)
                │ não
                ▼
              allow (matched_allow) + obrigações + TTL
```

### 6.1 Efeitos internos e efeito externo

| Efeito interno | Efeito entregue ao PEP | `reason_code` típico |
|----------------|------------------------|----------------------|
| `allow` | `allow` | `matched_allow`, `exception_applied` |
| `deny` | `deny` | `explicit_deny`, `platform_deny` |
| `not_applicable` | **`deny`** | `no_matching_rule` |
| `indeterminate` | **`deny`** | `attribute_unavailable`, `eval_timeout`, `no_active_bundle` |

O PEP **nunca** vê `not_applicable` nem `indeterminate` (ADR-0224).

## 7. Ciclo de vida da `PolicyException` (waiver)

| Estado | Significado | Próximos |
|--------|-------------|----------|
| `Pending` | Solicitada, aguardando aprovador distinto. | `Active`, `Denied` |
| `Active` | Vigente entre `effective_from` e `expires_at`. | `Expired`, `Revoked` |
| `Denied` | Recusada pelo aprovador. | — (terminal) |
| `Revoked` | Cancelada antes do prazo (`revoked_at`). | — (terminal) |
| `Expired` | Prazo alcançado. | — (terminal) |

```
   Pending ──aprova──▶ Active ──expires_at──▶ Expired (terminal)
      │                   │
      │ recusa            └──revoga──▶ Revoked (terminal)
      ▼
   Denied (terminal)
```

Regra normativa: **nenhuma exceção existe sem `expires_at`**
(`./Database.md` §3.6). `Revoked` e `Expired` emitem `policy.exception.expired`.

## 8. Ciclo de vida do `SimulationRun`

| Estado | Significado | Próximos |
|--------|-------------|----------|
| `running` | Reavaliação em curso no `policy-simulator`. | `completed`, `failed` |
| `completed` | Diferencial disponível; emite `policy.simulation.completed`. | — (terminal) |
| `failed` | Falha do simulador; o bundle permanece em `Tested`. | — (terminal) |

Uma simulação **NÃO DEVE**, em nenhum estado, alterar decisões de produção — é o
requisito que permite executá-la sobre tráfego real (FR-008).

## 9. Interação entre as máquinas

```
   PolicyBundle: Draft → … → Active
                              │
                              ├── emite policy.bundle.updated ──▶ PEPs invalidam TUDO
                              │
   PolicyException: Pending → Active
                              │
                              └── emite policy.decision.updated ─▶ PEPs invalidam o AFETADO

   SimulationRun: running → completed ──▶ habilita T-07 (Tested → Simulated)
```

> `bundle.updated` é a bomba de vácuo (raro, invalida tudo do tenant);
> `decision.updated` é o bisturi (frequente, invalida o afetado). Ter apenas o
> primeiro faria cada concessão de papel derrubar o cache de todos os PEPs.

## 10. Referências

- Fonte única de verdade: `./_DESIGN_BRIEF.md` §4
- Modelo físico: `./Database.md` · Eventos: `./Events.md`
- Fluxos: `./SequenceDiagrams.md` · Casos de uso: `./UseCases.md`
- Erros: `./API.md` §5

*Fim de `StateMachine.md`.*
