---
Documento: FunctionalRequirements
Módulo: 022-Policy
Status: Draft
Versão: 0.1
Última atualização: 2026-07-23
Responsável (RACI-A): Arquiteto do Módulo 022-Policy
ADRs relacionados: ADR-0008, ADR-0221, ADR-0223, ADR-0224, ADR-0225, ADR-0226, ADR-0227, ADR-0228
RFCs relacionados: RFC-0001, RFC-0220, RFC-0221
Depende de: _DESIGN_BRIEF.md §7.1, 021-Security, 025-Audit, 026-Cost-Optimizer
---

# 022-Policy — Requisitos Funcionais

> Todos os requisitos abaixo derivam de `./_DESIGN_BRIEF.md` §7.1 e **NÃO PODEM**
> contradizê-lo. Prioridade em MoSCoW (Must / Should / Could). Cada FR é verificável
> por ao menos um caso de teste em `./Testing.md` e exercitado por ao menos um caso de
> uso em `./UseCases.md`.

## 1. Convenções

- Palavras normativas conforme RFC 2119 / RFC 8174.
- Códigos de erro seguem `AIOS-POL-<NNNN>` (`./API.md` §5), formato da
  `../003-RFC/RFC-0001-Architecture-Baseline.md` §5.4.
- `reason_code` refere-se ao conjunto estável definido em `./_DESIGN_BRIEF.md` §4.7.

## 2. Tabela de requisitos funcionais

| ID | Requisito | Prioridade | Critério de aceite | Origem | UC | Teste |
|----|-----------|-----------|--------------------|--------|----|-------|
| FR-001 | O PDP **DEVE** avaliar um `DecisionRequest` e responder `allow`/`deny` sob *default deny*. | Must | Requisição sem regra casada retorna `deny` com `reason_code = no_matching_rule`. | Brief §7.1; ADR-0008 | UC-001 | T-POL-001 |
| FR-002 | O PDP **DEVE** avaliar RBAC e ABAC no mesmo ciclo, aplicando a precedência canônica de `./_DESIGN_BRIEF.md` §4.6. | Must | Suíte de conformidade cobre os 5 passos de precedência; resultado determinístico em 100% das repetições. | Brief §7.1; ADR-0223 | UC-001 | T-POL-002 |
| FR-003 | Toda decisão **DEVE** conter `reason_code`, `matched_rules[]` e `bundle_version`. | Must | 100% das respostas carregam os três campos; `deny` sem motivo reprova o teste. | Brief §7.1 | UC-001, UC-010 | T-POL-003 |
| FR-004 | O PDP **DEVE** retornar obrigações que o PEP **DEVE** cumprir, respeitando `pol.obligation.max_per_decision`. | Must | Obrigação desconhecida pelo PEP é tratada como `deny` (teste de contrato com `../006-Kernel/`). | Brief §7.1; ADR-0225 | UC-001 | T-POL-004 |
| FR-005 | O PDP **DEVE** retornar TTL de decisão, nunca superior ao configurado, e **NÃO DEVE** emitir TTL de `deny` maior que o de `allow`. | Must | `deny_ttl_ms` ≤ `default_ttl_ms` verificado em todas as respostas. | Brief §7.1 | UC-001 | T-POL-005 |
| FR-006 | O `BundleCompiler` **DEVE** compilar o bundle, validar referências e detectar conflitos e regras inalcançáveis. | Must | Contradição sem prioridade → `AIOS-POL-0008` em `pol.conflict.mode = strict`. | Brief §7.1 | UC-003 | T-POL-006 |
| FR-007 | O módulo **DEVE** barrar a publicação de bundle que não passe nos casos golden obrigatórios. | Must | Falha de caso `is_mandatory` → `AIOS-POL-0007`; bundle permanece fora de `Active`. | Brief §7.1; ADR-0226 | UC-004 | T-POL-007 |
| FR-008 | O `SimulationEngine` **DEVE** reavaliar tráfego registrado contra o bundle candidato e produzir o diferencial. | Must | Relatório com `flipped_to_deny`/`flipped_to_allow` por `rule_key`, sem efeito em produção. | Brief §7.1; ADR-0226 | UC-005 | T-POL-008 |
| FR-009 | A publicação **DEVE** exigir assinatura verificável e aprovação dupla. | Must | `approved_by = authored_by` → `AIOS-POL-0018`; assinatura inválida → `AIOS-POL-0014`. | Brief §7.1; ADR-0228 | UC-006 | T-POL-009 |
| FR-010 | O módulo **DEVE** permitir reverter para a versão anterior dentro da janela de retenção. | Must | Rollback ativa a versão alvo e emite `policy.bundle.updated` + `policy.bundle.rolledback`. | Brief §7.1 | UC-007 | T-POL-010 |
| FR-011 | O `BundleDistributor` **DEVE** propagar o bundle ativo a todas as réplicas e sinalizar por `policy.bundle.updated`. | Must | Nenhuma réplica serve versão anterior após `pol.bundle.propagation_target_s` (NFR-004). | Brief §7.1 | UC-014 | T-POL-011 |
| FR-012 | O módulo **DEVE** emitir `policy.decision.updated` em mudança granular (vínculo, exceção, papel). | Must | PEP invalida apenas o afetado; verificado por teste de contrato com `../006-Kernel/` e `../007-Agent-Runtime/`. | Brief §7.1 | UC-008, UC-009 | T-POL-012 |
| FR-013 | O `AttributeResolver` **DEVE** resolver atributos externos com timeout e criticidade declarada por fonte. | Must | Fonte `required` indisponível ⇒ `deny` com `attribute_unavailable` (`AIOS-POL-0009`). | Brief §7.1 | UC-011 | T-POL-013 |
| FR-014 | O `ExceptionManager` **DEVE** conceder exceções temporárias com prazo máximo, justificativa e aprovador distinto. | Must | `expires_at` sempre preenchido e ≤ `pol.exception.max_ttl_h`; expiração automática observada. | Brief §7.1; ADR-0227 | UC-008 | T-POL-014 |
| FR-015 | O `DecisionJournal` **DEVE** registrar 100% dos `deny` e uma amostra dos `allow`. | Must | Contagem de `deny` no journal = contagem em `aios_pol_decisions_total{effect="deny"}`. | Brief §7.1 | UC-001, UC-010 | T-POL-015 |
| FR-016 | O `ExplainService` **DEVE** explicar qualquer decisão registrada, reproduzindo-a sobre a versão exata do bundle. | Must | `ExplainDecision` devolve o mesmo `effect` e as mesmas `matched_rules`. | Brief §7.1 | UC-010 | T-POL-016 |
| FR-017 | O PDP **DEVE** responder decisões também por NATS request-reply em `aios.<tenant>.policy.decision.request`. | Must | Contrato exercitado com `../020-Communication/` (teste Pact). | Brief §7.1 | UC-001 | T-POL-017 |
| FR-018 | O `SelfGovernanceGuard` **DEVE** autorizar as operações administrativas do próprio módulo pelo bundle-raiz. | Must | Operação sem capability → `AIOS-POL-0002`; bundle-raiz não editável pelo fluxo comum. | Brief §7.1; ADR-0228 | UC-015 | T-POL-018 |
| FR-019 | O PDP **DEVE** negar tudo quando não houver bundle ativo para o `(tenant, scope)`. | Must | Tenant sem `Active` → `deny` com `no_active_bundle` (invariante I5). | Brief §7.1 | UC-001 | T-POL-019 |
| FR-020 | O módulo **DEVERIA** expor as permissões efetivas de um sujeito (RBAC resolvido + vínculos vigentes). | Should | Resultado coerente com decisões reais em amostragem cruzada de ≥ 1.000 pares. | Brief §7.1 | UC-012 | T-POL-020 |
| FR-021 | O `EventEmitter` **DEVE** emitir todos os eventos de `./Events.md` via Outbox transacional. | Must | Nenhum evento perdido em teste de crash entre commit e publicação. | Brief §7.1 | UC-014 | T-POL-021 |
| FR-022 | O `BundleCompiler` **DEVERIA** rejeitar bundle acima dos limites de complexidade configurados. | Should | Excesso de regras/profundidade/tamanho → `AIOS-POL-0016` na compilação, nunca em runtime. | Brief §7.1 | UC-003 | T-POL-022 |
| FR-023 | O módulo **NÃO DEVE** expor valor de atributo `sensitive` em resposta, log, métrica, evento ou journal. | Must | Varredura automatizada não encontra valor sensível; apenas `sha256` em `attributes_digest`. | Brief §7.1 | UC-011 | T-POL-023 |

## 3. Requisitos por agrupamento funcional

| Grupo | FRs | Documento de detalhamento |
|-------|-----|---------------------------|
| Decisão (caminho quente) | FR-001..FR-005, FR-013, FR-017, FR-019 | `./API.md`, `./SequenceDiagrams.md` |
| Ciclo de vida do bundle | FR-006..FR-011, FR-022 | `./StateMachine.md`, `./Database.md` |
| Governança e exceções | FR-009, FR-014, FR-018 | `./Security.md` |
| Explicabilidade e trilha | FR-003, FR-015, FR-016, FR-023 | `./Logging.md`, `../025-Audit/` |
| Integração assíncrona | FR-012, FR-021 | `./Events.md` |
| Consulta e suporte | FR-020 | `./Examples.md` |

## 4. Exemplo de verificação (FR-001 + FR-003)

Requisição a um tenant cuja política não contém regra para a ação solicitada:

```http
POST /v1/policy/decisions:evaluate HTTP/1.1
X-AIOS-Tenant: acme
Content-Type: application/json

{ "subject": {"urn":"urn:aios:acme:agent:01J9Z8Q6H7K2M4N6P8R0S2T4V6","roles":["agent.basic"]},
  "action": "tool:external-http:invoke",
  "resource": "urn:aios:acme:tool:01J9ZA00000000000000000000",
  "environment": {"tenant":"acme"} }
```

```json
HTTP/1.1 200 OK

{ "effect": "deny",
  "reasonCode": "no_matching_rule",
  "matchedRules": [],
  "bundleVersion": 42,
  "obligations": [],
  "decisionTtlMs": 1000,
  "decisionId": "01J9ZB00000000000000000000",
  "evaluatedAt": "2026-07-23T10:15:00.123Z" }
```

O `deny` é uma resposta `200` — não um erro (`./API.md` §5, nota deliberada).

## 5. Referências

- Fonte: `./_DESIGN_BRIEF.md` §7.1
- Rastreabilidade: `./Requirements.md`
- Não-funcionais: `./NonFunctionalRequirements.md`
- Casos de uso: `./UseCases.md` · Testes: `./Testing.md`

*Fim de `FunctionalRequirements.md`.*
