---
Documento: UseCases
Módulo: 022-Policy
Status: Draft
Versão: 0.1
Última atualização: 2026-07-23
Responsável (RACI-A): Arquiteto do Módulo 022-Policy
ADRs relacionados: ADR-0008, ADR-0221, ADR-0223, ADR-0224, ADR-0226, ADR-0227, ADR-0228
RFCs relacionados: RFC-0001, RFC-0220, RFC-0221
Depende de: FunctionalRequirements.md, StateMachine.md, API.md, Events.md
---

# 022-Policy — Casos de Uso

> Formato: ator · pré-condições · fluxo principal · fluxos alternativos · exceções ·
> pós-condições. Os IDs `UC-NNN` são referenciados por `./Requirements.md` §5 e por
> `./Testing.md`.

## UC-001 — Avaliar decisão de autorização (caminho quente)

| Campo | Conteúdo |
|-------|----------|
| **Ator primário** | PEP de um módulo (`CapabilityEnforcer` do `../006-Kernel/`, `RouteAuthorizer` do `../004-API/`, etc.) |
| **Atores secundários** | `../021-Security/` (atributos do princípio), `../026-Cost-Optimizer/` (orçamento), `../024-Observability/` (sinais) |
| **Pré-condições** | Chamador autenticado por mTLS; existe bundle em `Active` para o `(tenant, scope)`; `X-AIOS-Tenant` coerente com o contexto autenticado |
| **Requisitos** | FR-001, FR-002, FR-003, FR-004, FR-005, FR-015, FR-017, FR-019; NFR-001, NFR-008, NFR-016 |

**Fluxo principal**

1. O PEP verifica o próprio cache de decisão; havendo entrada válida, **não** chama o PDP.
2. Em *cache miss*, o PEP monta o `DecisionRequest` (`subject`, `action`, `resource`,
   `environment`) e chama `Evaluate` (gRPC) ou publica em
   `aios.<tenant>.policy.decision.request`.
3. O `DecisionGateway` valida o schema, extrai `traceparent`/`X-AIOS-Tenant`
   (RFC-0001 §5.6) e aplica o limite `pol.decision.max_eval_per_s_per_tenant`.
4. O `ActionNormalizer` converte a ação para a forma canônica `<dominio>:<objeto>:<verbo>`.
5. O `AttributeResolver` resolve os atributos exigidos pelas regras candidatas
   (cache local → Redis → fonte externa, com timeout por fonte).
6. O `RbacResolver` calcula os papéis efetivos do sujeito; o `AbacEvaluator` avalia as
   condições das regras candidatas.
7. O `DecisionEngine` combina os resultados pela precedência canônica
   (`./StateMachine.md` §6) e produz `effect`, `reason_code` e `matched_rules[]`.
8. O `ObligationComposer` monta as obrigações e o TTL (`decision_ttl_ms`).
9. A resposta é devolvida ao PEP; o `DecisionJournal` grava **assincronamente**
   (100% dos `deny`, amostra dos `allow`).
10. O PEP aplica a decisão: cumpre as obrigações e prossegue, ou impede a ação.

**Fluxos alternativos**

- **A1 — Exceção vigente:** existe waiver aplicável e a regra `deny` não é
  não-dispensável ⇒ efeito `allow` com `reason_code = exception_applied` (UC-008).
- **A2 — Decisão por barramento:** o chamador usa NATS request-reply; a resposta segue
  o mesmo contrato, com `traceparent` preservado (`../020-Communication/Examples.md`).
- **A3 — Lote:** o chamador agrupa até `pol.decision.batch_max_size` requisições
  (UC-002).

**Exceções**

| # | Condição | Resultado |
|---|----------|-----------|
| E1 | Nenhuma regra casou | `200` com `effect = deny`, `reason_code = no_matching_rule` |
| E2 | Não há bundle `Active` para o tenant | `200` com `deny`, `no_active_bundle` (invariante I5) |
| E3 | Atributo `required` não resolvido | `200` com `deny`, `attribute_unavailable`; erro `AIOS-POL-0009` apenas na superfície administrativa |
| E4 | Orçamento de avaliação estourado | `200` com `deny`, `eval_timeout`; métrica `aios_pol_eval_timeout_total` |
| E5 | `tenant` divergente do contexto autenticado | `403 AIOS-POL-0003` |
| E6 | Limite de avaliações do tenant excedido | `429 AIOS-POL-0010` com `Retry-After` |

**Pós-condições**

- O PEP possui uma decisão com motivo e TTL; nenhuma mudança de estado durável
  ocorreu no 022 além do registro assíncrono no journal.
- `deny` de tipo `explicit_deny`/`platform_deny` gerou `policy.decision.denied`.

---

## UC-002 — Avaliar decisões em lote

| Campo | Conteúdo |
|-------|----------|
| **Ator primário** | PEP que precisa autorizar um conjunto (ex.: `../009-Scheduler/` avaliando N candidatas à preempção) |
| **Pré-condições** | Mesmas de UC-001; tamanho do lote ≤ `pol.decision.batch_max_size` (64) |
| **Requisitos** | FR-001; NFR-002 |

**Fluxo principal**

1. O PEP envia `EvaluateBatch` com N `DecisionRequest` do **mesmo** tenant.
2. O PDP resolve os atributos **uma vez** por sujeito/recurso repetido (deduplicação
   interna) e avalia cada item.
3. Responde um vetor de `DecisionResponse` na mesma ordem, cada um com o próprio
   `decision_id` e TTL.

**Exceções**

| # | Condição | Resultado |
|---|----------|-----------|
| E1 | Lote acima do limite | `422 AIOS-POL-0016` |
| E2 | Itens de tenants diferentes no mesmo lote | `403 AIOS-POL-0003` (lote inteiro rejeitado) |

**Pós-condições:** cada item tem decisão independente; a falha de um item **NÃO DEVE**
transformar-se em `allow` de outro.

---

## UC-003 — Autorar e validar um bundle de política

| Campo | Conteúdo |
|-------|----------|
| **Ator primário** | Engenheiro de segurança do tenant |
| **Pré-condições** | Capability `pol:bundle:author`; tenant do autor = tenant do bundle |
| **Requisitos** | FR-006, FR-022 |

**Fluxo principal**

1. O autor cria o bundle (`CreateBundle`) → estado `Draft` (T-01).
2. Edita regras, papéis e vínculos (`PutBundleContent`); toda regra **DEVE** ter
   `description`.
3. Dispara `ValidateBundle` → `Validating` (T-02).
4. O `BundleCompiler` valida sintaxe, resolve referências de papel/atributo, verifica
   aciclicidade da herança e indexa as regras.
5. O `ConflictAnalyzer` procura contradições, sombreamento e regras inalcançáveis.
6. Sem conflito bloqueante ⇒ `Validated` (T-03). O bundle torna-se **imutável**.

**Fluxos alternativos**

- **A1 — Modo `warn`:** com `pol.conflict.mode = warn`, conflitos viram anotações e a
  compilação prossegue.
- **A2 — Descarte:** o autor abandona o rascunho (`DiscardBundle`, T-14).

**Exceções**

| # | Condição | Resultado |
|---|----------|-----------|
| E1 | Sintaxe/expressão inválida | `422 AIOS-POL-0004`, estado `Rejected` (T-04) |
| E2 | Referência a papel/atributo inexistente | `422 AIOS-POL-0012`, `Rejected` |
| E3 | Herança de papéis cíclica | `409 AIOS-POL-0017`, `Rejected` |
| E4 | Conflito em modo `strict` | `409 AIOS-POL-0008`, `Rejected` |
| E5 | Complexidade acima do limite | `422 AIOS-POL-0016`, `Rejected` |
| E6 | Tentativa de editar bundle fora de `Draft` | `409 AIOS-POL-0020` |

**Pós-condições:** bundle em `Validated` com `compiled_digest` calculado, ou em
`Rejected` com o motivo registrado e `policy.bundle.rejected` emitido.

---

## UC-004 — Executar os testes golden do bundle

| Campo | Conteúdo |
|-------|----------|
| **Ator primário** | Engenheiro de segurança do tenant (ou pipeline de CI do tenant) |
| **Pré-condições** | Bundle em `Validated`; capability `pol:bundle:test` |
| **Requisitos** | FR-007; NFR-011 |

**Fluxo principal**

1. `RunPolicyTests` executa todos os casos de `policy.policy_test` do tenant contra o
   bundle candidato, no `policy-simulator`.
2. Cada caso compara `effect` (e `reason_code`, quando declarado) com o esperado.
3. O runner calcula `test_coverage` = fração de regras exercitadas.
4. 100% dos casos `is_mandatory` passando **e** cobertura ≥ `pol.test.min_coverage`
   (0,90) ⇒ `Tested` (T-05).

**Exceções**

| # | Condição | Resultado |
|---|----------|-----------|
| E1 | Caso obrigatório falhou | `422 AIOS-POL-0007`; bundle → `Rejected` (T-06) |
| E2 | Cobertura abaixo do mínimo | `422 AIOS-POL-0007`; relatório aponta as regras não exercitadas |

**Pós-condições:** bundle em `Tested` com `test_coverage` gravado, ou `Rejected`.

---

## UC-005 — Simular o bundle candidato contra tráfego registrado

| Campo | Conteúdo |
|-------|----------|
| **Ator primário** | Engenheiro de segurança do tenant |
| **Pré-condições** | Bundle em `Tested`; existe bundle `Active` como linha de base; janela solicitada dentro de `pol.journal.retention_days` |
| **Requisitos** | FR-008; NFR-012 |

**Fluxo principal**

1. `RunSimulation` cria `policy.simulation_run` com `candidate_bundle` e
   `baseline_bundle`, em estado `running`.
2. O `SimulationEngine` lê as partições do `DecisionJournal` na janela e reavalia cada
   requisição com o bundle candidato, **sem** qualquer efeito em produção.
3. Agrega `flipped_to_deny`, `flipped_to_allow` e o diferencial por `rule_key`.
4. Conclui em `completed` e emite `policy.simulation.completed`; o bundle vai a
   `Simulated` (T-07).

**Fluxos alternativos**

- **A1 — Primeiro bundle do tenant:** não há linha de base; a simulação é pulada e o
  fluxo segue por T-09 com aprovação dupla registrada (UC-006, A1).

**Exceções**

| # | Condição | Resultado |
|---|----------|-----------|
| E1 | Janela fora da retenção do journal | `409 AIOS-POL-0005` (pré-condição de T-07 não satisfeita) |
| E2 | Volume acima de `pol.simulation.max_requests` | Simulação truncada e **marcada** como amostrada no relatório |
| E3 | Falha do simulador | `status = failed`; o bundle permanece em `Tested` |

**Pós-condições:** diferencial disponível para revisão humana; nenhuma decisão de
produção foi afetada (NFR-012).

---

## UC-006 — Publicar (ativar) um bundle

| Campo | Conteúdo |
|-------|----------|
| **Ator primário** | Aprovador de política (papel distinto do autor) |
| **Pré-condições** | Bundle em `Simulated` (ou `Tested`, no caso de *bootstrap*); capability `pol:bundle:publish`; artefato assinado pelo `../021-Security/` |
| **Requisitos** | FR-009; NFR-013, NFR-015 |

**Fluxo principal**

1. O aprovador revisa o diferencial da simulação.
2. Chama `PublishBundle` com `Idempotency-Key` (RFC-0001 §5.5).
3. O `SelfGovernanceGuard` verifica a capability e a regra `approved_by ≠ authored_by`.
4. O `BundleRegistry` verifica a assinatura do artefato compilado.
5. Em **uma transação**: bundle candidato → `Active` (T-08), bundle anterior →
   `Superseded` (T-10), evento `policy.bundle.updated` gravado no outbox (invariante I4).
6. O `BundleDistributor` propaga a nova versão às réplicas (UC-014).

**Fluxos alternativos**

- **A1 — Bootstrap:** primeiro bundle do tenant ou bundle de escopo `platform` em
  instalação inicial ⇒ publicação a partir de `Tested` (T-09), sempre com aprovação
  dupla registrada.

**Exceções**

| # | Condição | Resultado |
|---|----------|-----------|
| E1 | Autor = aprovador | `403 AIOS-POL-0018` |
| E2 | Assinatura ausente ou inválida | `412 AIOS-POL-0014` |
| E3 | Estado de origem inválido | `409 AIOS-POL-0005` |
| E4 | Publicação concorrente | `409 AIOS-POL-0006` (OCC + índice único parcial) |
| E5 | Repetição com a mesma `Idempotency-Key` | Mesmo resultado, sem novo efeito (NFR-013) |

**Pós-condições:** exatamente um bundle `Active` por `(tenant, scope)` (invariante I1);
PEPs invalidam cache ao receber `policy.bundle.updated`.

---

## UC-007 — Reverter para a versão anterior (rollback)

| Campo | Conteúdo |
|-------|----------|
| **Ator primário** | SRE de plantão ou aprovador de política |
| **Pré-condições** | Capability `pol:bundle:rollback`; versão alvo em `Superseded` dentro de `pol.bundle.rollback_retention_versions` |
| **Requisitos** | FR-010; NFR-006 |

**Fluxo principal**

1. Alerta de salto na taxa de `deny` após uma publicação (`./Monitoring.md` §4).
2. O operador chama `RollbackBundle` indicando a versão alvo.
3. Em **uma transação**: bundle atual → `RolledBack` (T-11); alvo → `Active` (T-12);
   eventos `policy.bundle.rolledback` e `policy.bundle.updated` no outbox.
4. Réplicas recarregam o artefato assinado do alvo; PEPs invalidam cache.

**Exceções**

| # | Condição | Resultado |
|---|----------|-----------|
| E1 | Alvo arquivado ou fora da janela | `409 AIOS-POL-0015` |
| E2 | Assinatura do alvo inválida | `412 AIOS-POL-0014` |
| E3 | Alvo não está em `Superseded` | `409 AIOS-POL-0005` |

**Pós-condições:** política anterior em vigor em ≤ `pol.bundle.propagation_target_s`;
o bundle revertido **NÃO** volta a vigorar sem nova publicação.

---

## UC-008 — Conceder e revogar exceção (waiver)

| Campo | Conteúdo |
|-------|----------|
| **Ator primário** | Solicitante (engenheiro/operador) + aprovador distinto |
| **Pré-condições** | Capability `pol:exception:grant`; justificativa preenchida |
| **Requisitos** | FR-012, FR-014 |

**Fluxo principal**

1. O solicitante submete `GrantException` com sujeito, ação, recurso, justificativa e
   prazo desejado.
2. O `ExceptionManager` limita o prazo a `pol.exception.max_ttl_h` (72 h) e exige
   `approved_by ≠ requested_by`.
3. Grava a exceção e emite `policy.exception.granted` **e**
   `policy.decision.updated` (invalidação granular).
4. Nas avaliações seguintes, a exceção converte `deny` em `allow` com
   `reason_code = exception_applied` — **exceto** para regra não-dispensável ou de
   escopo `platform`.
5. No vencimento, a exceção expira automaticamente e emite `policy.exception.expired`.

**Fluxos alternativos**

- **A1 — Revogação antecipada:** `RevokeException` grava `revoked_at` e emite
  `policy.exception.expired` imediatamente.

**Exceções**

| # | Condição | Resultado |
|---|----------|-----------|
| E1 | Prazo solicitado acima do máximo | `403 AIOS-POL-0013`, com o prazo máximo permitido no `detail` |
| E2 | Solicitante = aprovador | `403 AIOS-POL-0018` |
| E3 | Uso de exceção expirada | `deny` com `reason_code = exception_expired` |

**Pós-condições:** exceção vigente por prazo determinado, com trilha em
`../025-Audit/`; nenhuma exceção sem `expires_at` existe no sistema.

---

## UC-009 — Vincular sujeito a papel

| Campo | Conteúdo |
|-------|----------|
| **Ator primário** | Administrador de política do tenant |
| **Pré-condições** | Capability `pol:binding:write`; papel existe no bundle |
| **Requisitos** | FR-012 |

**Fluxo principal**

1. `CreateRoleBinding` com `role_key`, `subject_urn`, `subject_kind`, `scope_pattern` e
   `not_after` opcional.
2. Se o papel for `is_privileged`, o `SelfGovernanceGuard` exige aprovação dupla.
3. O vínculo é gravado e `policy.decision.updated` é emitido para o sujeito afetado.

**Exceções**

| # | Condição | Resultado |
|---|----------|-----------|
| E1 | Papel inexistente | `404 AIOS-POL-0001` |
| E2 | Papel privilegiado sem aprovação dupla | `403 AIOS-POL-0018` |
| E3 | Vínculo duplicado | Idempotente: mesmo resultado, sem novo efeito |

**Pós-condições:** permissões efetivas do sujeito atualizadas em ≤ 5 s nos PEPs
(NFR-004).

---

## UC-010 — Explicar uma decisão registrada

| Campo | Conteúdo |
|-------|----------|
| **Ator primário** | SRE de plantão, auditor ou desenvolvedor do agente afetado |
| **Pré-condições** | `decision_id` dentro de `pol.journal.retention_days`; capability `pol:decision:explain` |
| **Requisitos** | FR-003, FR-015, FR-016; NFR-007, NFR-009 |

**Fluxo principal**

1. O ator chama `ExplainDecision` com o `decision_id` (devolvido na resposta original).
2. O `ExplainService` carrega o registro do journal e o **artefato compilado da
   `bundle_version` exata** usada na decisão.
3. Reproduz a avaliação com os atributos identificados pelo `attributes_digest`.
4. Devolve `effect`, `reason_code`, caminho de regras avaliadas (casadas e
   descartadas, com o motivo do descarte) e obrigações emitidas.

**Exceções**

| # | Condição | Resultado |
|---|----------|-----------|
| E1 | `decision_id` fora da retenção | `404 AIOS-POL-0001`; orienta consultar `../025-Audit/` |
| E2 | Bundle da decisão já `Archived` sem artefato | `409 AIOS-POL-0015` (viola a expectativa da invariante I6 — incidente) |
| E3 | Tenant divergente | `403 AIOS-POL-0003` |

**Pós-condições:** decisão explicada de forma **idêntica** à original (NFR-009);
consulta registrada em auditoria.

---

## UC-011 — Degradação de fonte de atributo

| Campo | Conteúdo |
|-------|----------|
| **Ator primário** | `AttributeResolver` (automático) |
| **Atores secundários** | `../021-Security/`, `../026-Cost-Optimizer/`, `../024-Observability/` |
| **Pré-condições** | Fonte registrada em `policy.attribute_source` com `criticality` declarada |
| **Requisitos** | FR-013, FR-023; NFR-010 |

**Fluxo principal**

1. Uma regra candidata exige o atributo `budget.remaining_usd`.
2. O `AttributeResolver` tenta cache local → Redis → `../026-Cost-Optimizer/`, com
   timeout `pol.attribute.resolve_timeout_ms` (20 ms).
3. A fonte não responde no prazo.
4. Sendo `criticality = required`, o efeito interno é `indeterminate`, colapsado em
   **`deny`** com `reason_code = attribute_unavailable` (ADR-0224).
5. A fonte é marcada `degraded` e `policy.attribute.degraded` é emitido.

**Fluxos alternativos**

- **A1 — Atributo `optional`:** a regra que o exige é descartada; a avaliação
  prossegue com as demais.
- **A2 — Valor em cache ainda válido:** usa-se o valor cacheado, sem chamada externa.

**Exceções**

| # | Condição | Resultado |
|---|----------|-----------|
| E1 | Soma dos timeouts excede o orçamento de avaliação | `deny` com `eval_timeout` |
| E2 | Fonte devolve valor sensível | Valor **nunca** é gravado no journal — apenas o `sha256` (FR-023) |

**Pós-condições:** nenhuma decisão foi tomada com atributo faltante interpretado como
"permitido"; a degradação é observável em `./Monitoring.md`.

---

## UC-012 — Consultar permissões efetivas de um sujeito

| Campo | Conteúdo |
|-------|----------|
| **Ator primário** | Desenvolvedor de agente, administrador do tenant |
| **Pré-condições** | Capability `pol:subject:read` |
| **Requisitos** | FR-020 |

**Fluxo principal**

1. `GetEffectivePermissions` para o `subject_urn`.
2. O `RbacResolver` resolve papéis diretos, herdados e derivados de grupo, e agrega as
   permissões concedidas, aplicando `scope_pattern` e `not_after`.
3. Devolve a lista `{action_pattern, resource_pattern, origem}` e as exceções vigentes.

**Exceções**

| # | Condição | Resultado |
|---|----------|-----------|
| E1 | Sujeito sem vínculo | Lista vazia — **não** é erro; `deny` é o comportamento esperado |
| E2 | Tenant divergente | `403 AIOS-POL-0003` |

**Pós-condições:** o consultante conhece o alcance do sujeito. A resposta é uma
projeção do RBAC — **não** substitui a avaliação (regras ABAC dependem de contexto).

---

## UC-013 — Registrar ou atualizar fonte de atributo

| Campo | Conteúdo |
|-------|----------|
| **Ator primário** | Arquiteto de plataforma |
| **Pré-condições** | Capability `pol:attribute:manage` |
| **Requisitos** | FR-013 |

**Fluxo principal**

1. `PutAttributeSource` com `source_key`, `provider_module`, `criticality`,
   `resolve_timeout_ms`, `cache_ttl_s` e `sensitive`.
2. O módulo valida que o `provider_module` existe e que o timeout cabe no orçamento de
   avaliação.
3. Registra e passa a considerar a fonte nas próximas compilações e avaliações.

**Exceções**

| # | Condição | Resultado |
|---|----------|-----------|
| E1 | `resolve_timeout_ms` > `pol.decision.eval_budget_ms` | `422 AIOS-POL-0012` |
| E2 | Regras ativas referenciam fonte que se pretende remover | `409 AIOS-POL-0008` |

**Pós-condições:** fonte disponível para uso em condições ABAC; atributo `sensitive`
protegido em toda a cadeia de observabilidade.

---

## UC-014 — Propagar bundle e invalidar caches dos PEPs

| Campo | Conteúdo |
|-------|----------|
| **Ator primário** | `BundleDistributor` (automático, após UC-006 ou UC-007) |
| **Atores secundários** | Todas as réplicas `policy-api`; todos os PEPs do tenant |
| **Pré-condições** | Bundle em `Active`; evento no outbox |
| **Requisitos** | FR-011, FR-021; NFR-004, NFR-015 |

**Fluxo principal**

1. O `policy-outbox-relay` publica `aios.<tenant>.policy.bundle.updated` no
   JetStream (stream `POLICY_BUNDLE`).
2. Cada réplica `policy-api` carrega o artefato, **verifica a assinatura** e troca o
   `BundleRuntime` atomicamente.
3. Cada PEP consumidor invalida **todo** o cache de decisão do tenant afetado.
4. A métrica `aios_pol_active_bundle_version` converge em todas as réplicas em
   ≤ 10 s (NFR-004).

**Exceções**

| # | Condição | Resultado |
|---|----------|-----------|
| E1 | Réplica com assinatura inválida | Réplica recusa servir e sai do balanceamento (`./FailureRecovery.md` §3) |
| E2 | Réplica não converge no prazo | Alerta `PolicyBundleVersionSkew`; réplica retirada |
| E3 | Evento duplicado | Dedupe por `event.id` (RFC-0001 §5.5) |

**Pós-condições:** nenhuma réplica serve versão anterior após a janela; nenhum PEP
decide com cache anterior à ativação.

---

## UC-015 — Alterar o bundle-raiz de plataforma (autogoverno)

| Campo | Conteúdo |
|-------|----------|
| **Ator primário** | Arquitetura-chefe + CISO (cerimônia) |
| **Pré-condições** | `pol.selfgov.root_bundle_immutable = true`; dois aprovadores distintos; janela de manutenção |
| **Requisitos** | FR-018 |

**Fluxo principal**

1. A proposta de alteração do bundle `scope = platform` é registrada com justificativa
   e revisão fora do caminho de API comum.
2. Dois princípios distintos com capability `pol:bundle:publish` de plataforma aprovam.
3. O bundle-raiz candidato percorre a FSM completa (validação, testes, simulação),
   como qualquer outro.
4. A ativação é registrada em `../025-Audit/` com os dois aprovadores.

**Exceções**

| # | Condição | Resultado |
|---|----------|-----------|
| E1 | Tentativa de alterar pelo fluxo comum de tenant | `403 AIOS-POL-0002` |
| E2 | Um único aprovador | `403 AIOS-POL-0018` |
| E3 | Bundle-raiz que negaria a própria administração | Barrado pelos casos golden obrigatórios de plataforma (`AIOS-POL-0007`) |

**Pós-condições:** a política que governa o PDP mudou por cerimônia auditável; regras
`deny` de plataforma seguem com precedência absoluta sobre qualquer tenant.

---

## Referências

- Fonte única de verdade: `./_DESIGN_BRIEF.md`
- Fluxos detalhados: `./SequenceDiagrams.md` · FSM: `./StateMachine.md`
- Contratos: `./API.md` · `./Events.md`
- Rastreabilidade: `./Requirements.md` §5 e §6

*Fim de `UseCases.md`.*
