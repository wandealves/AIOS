---
Documento: SequenceDiagrams
Módulo: 022-Policy
Status: Draft
Versão: 0.1
Última atualização: 2026-07-23
Responsável (RACI-A): Arquiteto do Módulo 022-Policy
ADRs relacionados: ADR-0008, ADR-0221, ADR-0224, ADR-0226
RFCs relacionados: RFC-0001, RFC-0220, RFC-0221
Depende de: UseCases.md, API.md, Events.md, StateMachine.md
---

# 022-Policy — Diagramas de Sequência

> Todos os fluxos propagam `traceparent` (W3C Trace Context) e `X-AIOS-Tenant`
> conforme `../003-RFC/RFC-0001-Architecture-Baseline.md` §5.6. Timeouts indicados são
> os defaults de `./Configuration.md`.

## 1. UC-001 — Decisão com cache hit no PEP (caminho mais comum)

```
   Agente/Chamador      PEP (ex.: 006 CapabilityEnforcer)        022-Policy
        │                        │                                   │
        │  ação privilegiada     │                                   │
        │───────────────────────▶│                                   │
        │                        │ consulta cache local              │
        │                        │──────┐                            │
        │                        │◀─────┘ hit (TTL válido)           │
        │                        │                                   │
        │◀── prossegue/impede ───│      (nenhuma chamada ao PDP)     │
        │                        │                                   │
```

> Este é o caminho que sustenta o NFR-002: a maioria das decisões **nunca** chega ao
> 022. O TTL que o torna possível é emitido pelo próprio PDP (`decisionTtlMs`).

## 2. UC-001 — Decisão com cache miss (caminho quente completo)

```
  PEP            DecisionGateway   DecisionEngine   AttributeResolver   BundleRuntime  Journal
   │                    │                │                 │                 │           │
   │ Evaluate(req)      │                │                 │                 │           │
   │───────────────────▶│                │                 │                 │           │
   │                    │ valida tenant  │                 │                 │           │
   │                    │ + rate limit   │                 │                 │           │
   │                    │───────┐        │                 │                 │           │
   │                    │◀──────┘ ok     │                 │                 │           │
   │                    │ Decide(req)    │                 │                 │           │
   │                    │───────────────▶│                 │                 │           │
   │                    │                │ ActiveFor(tenant)                 │           │
   │                    │                │──────────────────────────────────▶│           │
   │                    │                │◀── CompiledBundle (v42) ──────────│           │
   │                    │                │ Resolve(required)                 │           │
   │                    │                │────────────────▶│                 │           │
   │                    │                │                 │ cache local     │           │
   │                    │                │                 │──────┐          │           │
   │                    │                │                 │◀─────┘ hit      │           │
   │                    │                │◀── AttributeSet │  (≤ 20 ms/fonte)│           │
   │                    │                │ RBAC + ABAC + combina (§6 FSM)    │           │
   │                    │                │──────┐                            │           │
   │                    │                │◀─────┘ allow + obligations + ttl  │           │
   │                    │◀── Decision ───│                                   │           │
   │◀── DecisionResponse│                │                                   │           │
   │  (p99 ≤ 5 ms)      │                │  grava assíncrono ────────────────────────────▶│
   │                    │                │  (fora do caminho de resposta — S6)           │
   │ cumpre obrigações  │                │                                   │           │
   │ e prossegue        │                │                                   │           │
```

**Falha coberta:** se `Resolve` estourar o timeout de uma fonte `required`, o
`DecisionEngine` produz `indeterminate` → colapsa em `deny`
(`reason_code = attribute_unavailable`) **antes** de responder — ver §5.

## 3. UC-001 (A2) — Decisão por NATS request-reply

```
   006-Kernel            NATS            022-Policy (PDP)
       │                  │                     │
       ├─ request(aios.acme.policy.decision.request, reply=_INBOX.xyz) ▶
       │                  ├─ deliver ──────────▶│
       │                  │                     ├─ avalia (RBAC+ABAC)
       │                  │                     │  traceparent preservado
       │                  │◀─ publish _INBOX.xyz (DecisionResponse) ─┤
       │◀─ resposta ──────┤                     │
       │  (timeout do chamador: bus.request.timeout_ms)              │
```

Contrato idêntico ao gRPC (RFC-0220); ver `../020-Communication/Examples.md` para o
lado do barramento.

## 4. UC-003..UC-006 — Do rascunho à publicação

```
  Autor        PolicyApi      BundleCompiler   PolicyTestRunner  SimulationEngine  Registry
    │              │                │                 │                 │             │
    │ CreateBundle │                │                 │                 │             │
    │─────────────▶│ T-01 Draft ────────────────────────────────────────────────────▶│
    │ PutContent   │                │                 │                 │             │
    │─────────────▶│                │                 │                 │             │
    │ Validate     │                │                 │                 │             │
    │─────────────▶│ T-02 Validating│                 │                 │             │
    │              │───────────────▶│ compila+índice  │                 │             │
    │              │                │ ConflictAnalyzer│                 │             │
    │              │                │──────┐          │                 │             │
    │              │                │◀─────┘ 0 conflito bloqueante      │             │
    │              │◀── CompiledBundle (digest) ──────────────────────────────────────▶│
    │◀── Validated (T-03) ──────────│                 │                 │             │
    │ RunTests     │                │                 │                 │             │
    │─────────────▶│────────────────────────────────▶│ golden + cobertura            │
    │              │                │                 │──────┐          │             │
    │              │                │                 │◀─────┘ 100% obrig., 0,94      │
    │◀── Tested (T-05) ─────────────────────────────────────────────────────────────▶│
    │ RunSimulation│                │                 │                 │             │
    │─────────────▶│──────────────────────────────────────────────────▶│ reavalia    │
    │              │                │                 │                 │ janela do   │
    │              │                │                 │                 │ journal     │
    │◀── Simulated (T-07) + diff (flipped_to_deny=12) ──────────────────┤             │
    │              │                │                 │                 │             │
  Aprovador (≠ autor)               │                 │                 │             │
    │ Publish + Idempotency-Key      │                 │                 │             │
    │─────────────▶│ SelfGovernanceGuard: capability + approved_by≠authored_by        │
    │              │ Registry.VerifySignature ───────────────────────────────────────▶│
    │              │ TRANSAÇÃO: T-08 Active + T-10 Superseded + outbox(bundle.updated)│
    │◀── 200 Active (v43) ──────────│                 │                 │             │
```

## 5. Fluxo de falha — fonte de atributo indisponível (UC-011)

```
  PEP        DecisionEngine     AttributeResolver      026-Cost-Optimizer
   │               │                    │                      │
   │ Evaluate      │                    │                      │
   │──────────────▶│ Resolve(budget.*)  │                      │
   │               │───────────────────▶│ GET budget           │
   │               │                    │─────────────────────▶│
   │               │                    │      (sem resposta)  │
   │               │                    │  ⏱ timeout 20 ms     │
   │               │                    │──────┐               │
   │               │                    │◀─────┘ Indeterminate │
   │               │◀── Indeterminate ──│                      │
   │               │ criticality=required ⇒ colapsa em deny    │
   │               │──────┐                                    │
   │               │◀─────┘ deny(attribute_unavailable)        │
   │◀── deny ──────│                                           │
   │  (200, ttl=1s)│                                           │
   │               │ marca fonte degraded                      │
   │               │ emite aios._platform.policy.attribute.degraded ──▶ NATS
```

> A decisão **não** é adiada nem devolvida como erro `5xx` ao PEP: é um `deny` com
> motivo. Devolver erro faria o PEP tratar a indisponibilidade como incidente
> transitório e, na pior implementação, repetir até obter um `allow`.

## 6. Fluxo de falha — publicação que quebra produção e rollback (UC-007)

```
  Monitoring        SRE           PolicyApi        Registry        PEPs (todos)
      │              │                │                │                │
      │ alerta PolicyDenyRateSpike (deny +400% pós v43)                  │
      │─────────────▶│                │                │                │
      │              │ GET simulations/{urn}  (revisa o diferencial)     │
      │              │───────────────▶│                │                │
      │              │ RollbackBundle(target=v42)      │                │
      │              │───────────────▶│ SelfGovernanceGuard: pol:bundle:rollback
      │              │                │ TRANSAÇÃO: T-11 v43→RolledBack  │
      │              │                │            T-12 v42→Active      │
      │              │                │            outbox: bundle.rolledback
      │              │                │                    + bundle.updated
      │              │◀── 200 ────────│                │                │
      │              │                │ propaga (≤ 10 s) ───────────────▶│ invalidam
      │              │                │                │                │ cache total
      │◀── taxa de deny normaliza ─────────────────────────────────────────┤
```

RTO da operação: **≤ 15 min** (NFR-006), dominado pelo tempo de decisão humana — a
execução técnica é uma transação e uma propagação de ≤ 10 s.

## 7. Fluxo de falha — PDP indisponível visto pelo PEP

```
   PEP (006 CapabilityEnforcer)        022-Policy
        │                                   │
        │ Evaluate                          │
        │──────────────────────────────────▶│ (sem resposta)
        │  ⏱ timeout                        │
        │──────┐                            │
        │◀─────┘ circuit breaker abre        │
        │                                   │
        │ fail_mode = closed (default)      │
        │──────┐                            │
        │◀─────┘ nega a syscall             │
        │  AIOS-CAP-0003 (503, retriable)   │
        │                                   │
        │ (meia-abertura periódica) ───────▶│  ← recuperação automática
```

O comportamento do PEP sob indisponibilidade do PDP é **do PEP** e está normatizado em
`../006-Kernel/Security.md` §3.3 e em RFC-0001 §5.8; o 022 apenas garante que a
ausência de resposta nunca seja interpretável como `allow`.

## 8. Concessão de exceção e invalidação granular (UC-008)

```
  Solicitante   Aprovador   ExceptionManager    Outbox/NATS        PEPs
      │             │              │                 │               │
      │ GrantException (justificativa, prazo 24h)     │               │
      │────────────────────────────▶│                │               │
      │             │              │ valida prazo ≤ 72h              │
      │             │ aprova       │ e approved_by ≠ requested_by    │
      │             │─────────────▶│                 │               │
      │             │              │ grava Exception(Active)         │
      │             │              │ outbox: exception.granted       │
      │             │              │       + decision.updated ──────▶│
      │             │              │                 │──────────────▶│ invalida
      │             │              │                 │               │ SÓ o sujeito
      │◀── 201 ─────────────────────┤                │               │
      │                            │                 │               │
      │  ... 24 h depois ...       │ expira automaticamente          │
      │                            │ outbox: exception.expired ─────▶│ invalida
```

## 9. Explicação de decisão (UC-010)

```
   SRE            ExplainService      DecisionJournal    BundleRegistry   DecisionEngine
    │                   │                    │                 │                │
    │ Explain(decId)    │                    │                 │                │
    │──────────────────▶│ busca registro     │                 │                │
    │                   │───────────────────▶│                 │                │
    │                   │◀── record(v42, attributes_digest) ───│                │
    │                   │ carrega artefato da v42 ────────────▶│                │
    │                   │◀── CompiledBundle(v42) ──────────────│                │
    │                   │ reavalia (mesmo motor, modo sombra) ─────────────────▶│
    │                   │◀── mesma decisão (NFR-009) ──────────────────────────│
    │◀── Explanation ───│                                                       │
    │  (regras casadas, regras descartadas + motivo, obrigações)                │
```

## 10. Referências

- Casos de uso: `./UseCases.md` · FSM: `./StateMachine.md`
- Contratos: `./API.md` · `./Events.md`
- Falhas: `./FailureRecovery.md` · Observabilidade: `./Monitoring.md`
- Convenções de diagrama: `../_templates/MODULE_TEMPLATE.md`

*Fim de `SequenceDiagrams.md`.*
