---
Documento: StateMachine
Módulo: 024-Observability
Status: Draft
Versão: 0.1
Última atualização: 2026-07-23
Responsável (RACI-A): Arquiteto do Módulo 024-Observability
ADRs relacionados: ADR-0010, ADR-0241, ADR-0242, ADR-0245, ADR-0247
RFCs relacionados: RFC-0001, RFC-0240, RFC-0241
Depende de: _DESIGN_BRIEF.md §4, Database.md, Events.md, UseCases.md
---

# 024-Observability — Máquina de Estados

## 1. Qual entidade tem ciclo de vida

A entidade governada por máquina de estados neste módulo é o **`AlertInstance`** — o
produto final do módulo e a única coisa que ele entrega a um ser humano.

Séries temporais não têm ciclo de vida (têm **retenção**, `./Database.md` §11); SLOs
mudam por **versão**; mas o alerta nasce, é notificado, reconhecido, eventualmente
silenciado e resolvido — e cada transição tem consequência operacional real: alguém é
acordado, alguém assume, alguém deixa de ser notificado.

Entidade com ciclo secundário: o **`SignalDescriptor`** (§5), cujo estado governa a
admissão da telemetria.

## 2. Estados (`AlertState`)

| Estado | Descrição | Terminal? | Ação de entrada | Ação de saída |
|--------|-----------|-----------|-----------------|---------------|
| `Pending` | Condição verdadeira, mas ainda dentro de `for_duration`. | não | Registra `started_at` e `fingerprint` | — |
| `Firing` | Condição sustentada; notificação entregue ou em entrega. | não | Emite `telemetry.alert.firing`; agenda escalonamento | Cancela escalonamento |
| `Acknowledged` | Um operador assumiu; escalonamento suspenso. | não | Grava `acknowledged_by`/`acknowledged_at`; emite `alert.acknowledged` | — |
| `Silenced` | Coberto por silêncio vigente; não notifica, **continua avaliando**. | não | Grava `silenced_until` | Reavalia a condição |
| `Resolved` | Condição deixou de ser verdadeira. | **sim** | Grava `resolved_at`; emite `telemetry.alert.resolved` | — |
| `Expired` | Condição não pôde ser avaliada por falta de dado. | **sim** | Emite `telemetry.alert.nodata` | — |

## 3. Transições, gatilhos e guardas

| # | De → Para | Gatilho | Guarda (DEVE ser verdadeira) | Erro se violada |
|---|-----------|---------|------------------------------|-----------------|
| T-01 | ∅ → `Pending` | Condição da regra tornou-se verdadeira | Regra em `status = active` ∧ sinais da `expr` **registrados** ∧ nenhuma instância aberta com o mesmo `fingerprint`. | `AIOS-OBS-0002` |
| T-02 | `Pending` → `Firing` | `for_duration` decorrida com condição contínua | Condição verdadeira em **todas** as avaliações da janela ∧ regra possui `runbook_urn` válido. | `AIOS-OBS-0008` |
| T-03 | `Pending` → `Resolved` | Condição cessou antes de `for_duration` | Sem notificação emitida — o alerta **nunca** alcançou um humano. | — |
| T-04 | `Firing` → `Acknowledged` | `AcknowledgeAlert` | Capability `obs:alert:ack` ∧ `acknowledged_by` identificado. | `AIOS-OBS-0003` |
| T-05 | `Firing`/`Acknowledged` → `Silenced` | Silêncio vigente casa com os labels | Silêncio com `expires_at` no futuro ∧ `reason` preenchida. | `AIOS-OBS-0013` |
| T-06 | `Silenced` → `Firing` | Silêncio expirou ou foi revogado | Reavaliação confirma que a condição ainda é verdadeira. | — |
| T-07 | `Firing`/`Acknowledged`/`Silenced` → `Resolved` | Condição deixou de ser verdadeira | Confirmada por ≥ 1 janela de avaliação (evita *flapping*). | — |
| T-08 | `Firing` → `Firing` | `escalation_after` decorrido sem reconhecimento | `escalation_level` < máximo configurado; incrementa o nível. | — |
| T-09 | `Pending`/`Firing`/`Acknowledged`/`Silenced` → `Expired` | Dado ausente além de `obs.alert.no_data_timeout_s` | Ausência sustentada de dado — **ausência não é sucesso**. | — |

## 4. Diagrama de estados (ASCII)

```
        condição verdadeira (T-01)
   ∅ ─────────────────────────────▶ ┌───────────┐
                                     │  Pending  │
                                     └─────┬─────┘
              condição cessa (T-03)        │ for_duration decorrida (T-02)
        ┌────────────────────────────────  ┤
        │                                  ▼
        │                            ┌───────────┐  escalonamento (T-08)
        │                            │  Firing   │──────────┐
        │                            └──┬───┬────┘◀─────────┘
        │              ack (T-04)       │   │  silêncio casa (T-05)
        │            ┌──────────────────┘   └──────────────┐
        │            ▼                                      ▼
        │     ┌──────────────┐   silêncio (T-05)     ┌────────────┐
        │     │ Acknowledged │──────────────────────▶│  Silenced  │
        │     └──────┬───────┘                        └─────┬──────┘
        │            │                    expira/revoga (T-06) │
        │            │                        ┌───────────────┘
        │            │                        ▼ (condição ainda verdadeira)
        │            │                    ┌───────────┐
        │            │                    │  Firing   │
        │            │                    └───────────┘
        │  condição cessa (T-07)  │            │
        └────────────┬────────────┴────────────┘
                     ▼
              ┌────────────┐          sem dado (T-09)     ┌───────────┐
              │  Resolved  │ (terminal)  ───────────────▶ │  Expired  │ (terminal)
              └────────────┘                               └───────────┘
```

## 5. Invariantes

| ID | Invariante | Como é garantida |
|----|-----------|------------------|
| I1 | Existe **no máximo uma** instância aberta (`Pending`, `Firing`, `Acknowledged`, `Silenced`) por `(tenant_id, fingerprint)`. | Índice único parcial (`./Database.md` §6). Duas instâncias do mesmo alerta é ruído — e ruído leva a silêncio. |
| I2 | Nenhuma transição para `Firing` ocorre sem `runbook_urn` válido na regra. | Guarda de T-02 + `NOT NULL` em `telemetry.alert_rule.runbook_urn`. |
| I3 | `Silenced` **NÃO DEVE** interromper a avaliação. | O `AlertEvaluator` continua avaliando; o `AlertRouter` é que suprime a notificação. Silêncio que parasse a observação esconderia a **resolução** tanto quanto o problema. |
| I4 | `Resolved` e `Expired` são terminais; a recorrência cria **nova** instância. | Transições da §3; o histórico de recorrência é o dado mais útil na análise pós-incidente. |
| I5 | Toda transição para `Firing` e para `Resolved` emite evento via Outbox **na mesma transação**. | `./Events.md` §5 (padrão Outbox). |
| I6 | `Expired` **NÃO DEVE** ser tratado como `Resolved`. | Estado distinto + evento distinto (`alert.nodata`). Falta de dado é falha do observador, não melhora do observado. |
| I7 | Toda transição ocorre sob OCC pelo `version` esperado. | Conflito → `AIOS-OBS-0006`. |

> **Por que I6 tem um estado próprio.** A forma mais comum de um sistema de alerta
> mentir é interpretar "parei de receber dados" como "o problema acabou". O alvo que
> caiu deixa de reportar exatamente a métrica que provaria que ele caiu.

## 6. Avaliação de SLO e *burn rate* (não é FSM)

O `SloEngine` avalia continuamente, sem estado durável além do ledger
(`./Database.md` §5):

```
   SLI(janela) ──▶ budget_consumed = (1 - SLI) / (1 - objective)
                          │
                          ├─ burn_rate_1h ≥ 14,4  ∧ burn_rate_6h ≥ 6  ⇒ alerta CRITICAL
                          ├─ burn_rate_6h ≥ 6     ∧ burn_rate_24h ≥ 3 ⇒ alerta WARNING
                          └─ budget_consumed ≥ política ⇒ telemetry.budget.exhausted
                                                          (consumido por 028 e 022)
```

A abordagem **multi-janela/multi-taxa** existe para separar "queima rápida" de
"degradação lenta":

| Abordagem | Falha característica |
|-----------|----------------------|
| Só limiar absoluto (`SLI < objetivo`) | Avisa quando o budget já acabou — tarde demais para agir. |
| Só taxa instantânea | Dispara a cada oscilação — ruído que leva ao silêncio. |
| **Multi-janela/multi-taxa** | Janela curta detecta rápido; janela longa confirma que não é ruído. |

## 7. Ciclo de vida do `SignalDescriptor`

| Estado | Significado | Próximos |
|--------|-------------|----------|
| `Proposed` | Sinal declarado pelo módulo dono, aguardando validação de convenção. | `Registered`, `Rejected` |
| `Registered` | Admitido; **descritor imutável** a partir daqui. | `Quarantined`, `Deprecated` |
| `Quarantined` | Excedeu cardinalidade ou violou convenção em runtime: **descartado na ingestão**. | `Registered`, `Retired` |
| `Rejected` | Reprovado na validação (nome, label proibido, unidade ausente). | — (terminal) |
| `Deprecated` | Substituído por versão nova; aceito durante a janela de migração. | `Retired` |
| `Retired` | Não aceito; séries expurgadas conforme retenção. | — (terminal) |

```
   Proposed ──valida──▶ Registered ──cardinalidade estourou──▶ Quarantined
      │                    │    ▲                                   │
      │ falha              │    └──── corrigido e reabilitado ──────┘
      ▼                    │ nova versão                            │
   Rejected           Deprecated ──────janela de migração──────▶ Retired
   (terminal)                                                  (terminal)
```

**Invariantes do `SignalDescriptor`:**
(S1) O descritor é **imutável** a partir de `Registered`; alteração exige nova
`descriptor_version` (`AIOS-OBS-0014`).
(S2) `Quarantined` é **reversível de propósito**: a resposta correta a uma explosão de
cardinalidade é descartar o sinal e avisar o dono, não perdê-lo para sempre nem
derrubar o coletor tentando absorvê-lo.
(S3) Sinal em `Quarantined` ou `Retired` **NÃO DEVE** ser referenciado por SLO ativo
nem por dashboard publicado — publicação que o referencie é rejeitada
(`AIOS-OBS-0009`).
(S4) A transição `Proposed → Registered` valida a convenção da RFC-0240; não há
caminho que persista série de sinal nunca validado.

## 8. Interação entre as máquinas

```
   SignalDescriptor: Proposed → Registered
                                   │
                                   ├── habilita SLO que o referencia (S3)
                                   │
                                   └── Quarantined ──▶ regras que dependem do sinal
                                                        deixam de avaliar ⇒ T-09
                                                        (AlertInstance → Expired)

   AlertInstance: Pending → Firing ──emite──▶ 029-Operations (plantão)
                                    └──────▶ 025-Audit (fato de governança)
```

> A seta mais sutil do diagrama: quarentenar um sinal por cardinalidade **cega** os
> alertas que dependem dele. Por isso a quarentena emite evento próprio
> (`telemetry.signal.quarantined`) e os alertas afetados vão a `Expired` — e não a
> `Resolved`. Um sinal descartado não é um problema resolvido.

## 9. Referências

- Fonte única de verdade: `./_DESIGN_BRIEF.md` §4
- Modelo físico: `./Database.md` · Eventos: `./Events.md`
- Fluxos: `./SequenceDiagrams.md` · Casos de uso: `./UseCases.md`
- Erros: `./API.md` §6 · Alertas e runbooks: `./Monitoring.md`

*Fim de `StateMachine.md`.*
