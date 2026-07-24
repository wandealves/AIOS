---
Documento: Testing
Módulo: 022-Policy
Status: Draft
Versão: 0.1
Última atualização: 2026-07-23
Responsável (RACI-A): Arquiteto do Módulo 022-Policy + QA
ADRs relacionados: ADR-0008, ADR-0221, ADR-0224, ADR-0225, ADR-0226
RFCs relacionados: RFC-0001, RFC-0220, RFC-0221
Depende de: FunctionalRequirements.md, NonFunctionalRequirements.md, UseCases.md, Requirements.md
---

# 022-Policy — Estratégia de Testes

## 1. Duas camadas que não se confundem

| Camada | O que testa | Quem escreve | Onde vive |
|--------|-------------|--------------|-----------|
| **Testes do módulo** (esta estratégia) | O motor de política: compilação, avaliação, FSM, contratos, falhas. | Equipe do 022 | CI do produto |
| **Testes de política** (casos golden) | A política **de um tenant**: "este papel pode ler esta memória?". | Engenharia de segurança do tenant | `policy.policy_test`, executados pelo `PolicyTestRunner` |

Os dois usam a palavra "teste" e resolvem problemas diferentes. Um motor correto com
política errada nega o que deveria permitir; uma política correta com motor errado faz
o inverso. O gate de publicação (FR-007) cobre a segunda camada; esta seção cobre a
primeira.

## 2. Pirâmide e cobertura-alvo

| Nível | Escopo | Cobertura-alvo | Ferramenta |
|-------|--------|----------------|-----------|
| Unitário | `DecisionEngine`, `RbacResolver`, `AbacEvaluator`, `ObligationComposer`, `ConflictAnalyzer` | ≥ 90% de linhas; **100%** dos ramos de combinação (§4 do `StateMachine`) | xUnit |
| Integração | Compilação + `BundleRuntime` + PostgreSQL + Redis | ≥ 80% | Testcontainers |
| Contrato | PEP↔PDP (gRPC, NATS, REST), eventos | 100% das operações de `./API.md` | Pact + JSON Schema |
| E2E | Autoria → teste → simulação → publicação → decisão → rollback | 100% dos UC-001..UC-015 | Compose + cenários |
| Carga | NFR-001, NFR-002, NFR-012 | — | k6 / NBomber |
| Caos | `./FailureRecovery.md` §7 | 7 experimentos | Chaos harness |

## 3. Testes por fronteira testável

As interfaces de `./ClassDiagrams.md` §2 são os pontos de injeção:

| Fronteira | Duplo de teste | O que se verifica |
|-----------|----------------|-------------------|
| `IAttributeResolver` | *Fake* com latência e falha programáveis | `required` ausente ⇒ `deny`; `optional` ausente ⇒ regra descartada |
| `IBundleRuntime` | Bundle compilado em memória, fixado | Determinismo; troca atômica durante avaliação |
| `IBundleRegistry` | PostgreSQL real (Testcontainers) | FSM, OCC, índice único parcial |
| `ISimulationEngine` | Journal sintético | Diferencial correto; zero efeito em produção |
| `ISelfGovernanceGuard` | Bundle-raiz de teste | Nenhuma rota administrativa sem passar por ele |

## 4. Casos de teste — requisitos funcionais

| ID | Requisito | Cenário | Resultado esperado |
|----|-----------|---------|--------------------|
| T-POL-001 | FR-001 | Ação sem regra correspondente | `200`, `effect=deny`, `reason_code=no_matching_rule` |
| T-POL-002 | FR-002 | Matriz dos 5 passos de precedência (§6 `StateMachine`) | Resultado determinístico em 100% das repetições; `deny` vence empate |
| T-POL-003 | FR-003 | Toda decisão da suíte | `reason_code`, `matched_rules`, `bundle_version` presentes |
| T-POL-004 | FR-004 | PEP recebe obrigação `kind` desconhecida | PEP trata como `deny` (teste de contrato com `../006-Kernel/`) |
| T-POL-005 | FR-005 | Comparar TTLs de `allow` e `deny` | `deny_ttl_ms` ≤ `default_ttl_ms` em 100% |
| T-POL-006 | FR-006 | Bundle com `allow` e `deny` de mesma prioridade para o mesmo par | `AIOS-POL-0008` em `strict`; anotação em `warn` |
| T-POL-007 | FR-007 | Caso golden obrigatório falhando | `AIOS-POL-0007`; bundle → `Rejected`; nunca `Active` |
| T-POL-008 | FR-008 | Simulação sobre 10⁵ decisões registradas | Diferencial por `rule_key`; **0** decisão de produção alterada |
| T-POL-009 | FR-009 | Publicar com `approved_by = authored_by` | `AIOS-POL-0018` (barrado também pelo `CHECK` do banco) |
| T-POL-010 | FR-010 | Rollback para v-1 | v anterior `Active`; eventos `rolledback` + `updated`; ≤ 10 s |
| T-POL-011 | FR-011 | Publicar com 3 réplicas ativas | Convergência de `active_bundle_version` em ≤ 10 s |
| T-POL-012 | FR-012 | Criar vínculo de papel | `policy.decision.updated` com `subjectUrn`; PEP invalida **apenas** o sujeito |
| T-POL-013 | FR-013 | Fonte `required` com timeout | `deny`/`attribute_unavailable`; fonte marcada `degraded` |
| T-POL-014 | FR-014 | Waiver com prazo de 200 h | `AIOS-POL-0013`; nenhum waiver sem `expires_at` no banco |
| T-POL-015 | FR-015 | 10.000 decisões (mistura allow/deny) | 100% dos `deny` no journal; `allow` ≈ taxa configurada |
| T-POL-016 | FR-016 | `Explain` de decisão de 20 dias atrás | Mesmo `effect` e mesmas `matched_rules` da original |
| T-POL-017 | FR-017 | Decisão por NATS request-reply | Contrato idêntico ao gRPC; `traceparent` preservado |
| T-POL-018 | FR-018 | Chamada administrativa sem capability; tentativa de rota alternativa | `AIOS-POL-0002`; **nenhuma** rota administrativa alcançável sem o guard |
| T-POL-019 | FR-019 | Tenant sem bundle `Active` | 100% `deny` com `no_active_bundle`; **0** `allow` |
| T-POL-020 | FR-020 | Permissões efetivas × decisões reais | Coerência em amostra de ≥ 1.000 pares |
| T-POL-021 | FR-021 | Crash entre commit e publicação | Evento publicado após recuperação; nenhum perdido |
| T-POL-022 | FR-022 | Bundle com 100.001 regras | `AIOS-POL-0016` na compilação, nunca em runtime |
| T-POL-023 | FR-023 | Atributo `sensitive` em decisão | Valor ausente de resposta, log, métrica, evento e journal; só `sha256` |

## 5. Casos de teste — requisitos não-funcionais

| ID | NFR | Método | Critério |
|----|-----|--------|----------|
| T-POL-101 | NFR-001 | Carga sustentada, atributos em cache | p99 ≤ 5 ms; com atributo remoto ≤ 15 ms |
| T-POL-102 | NFR-002 | Carga por réplica isolada | ≥ 50.000 decisões/s |
| T-POL-103 | NFR-003 | Injeção de falhas + probe | ≥ 99,99% na janela |
| T-POL-104 | NFR-004 | Publicação com N réplicas | Convergência ≤ 10 s; cache do PEP ≤ 5 s |
| T-POL-105 | NFR-005 | Bundle com 10⁴ regras, 10⁵ vínculos | Latência cresce sub-linearmente |
| T-POL-106 | NFR-006 | DR drill | RTO ≤ 15 min; RPO ≤ 5 min |
| T-POL-107 | NFR-007 | Varredura do journal | 0 `deny` sem `reason_code` |
| T-POL-108 | NFR-008 | Varredura + constraint | 0 `allow` sem regra; `ck_allow_has_rule` rejeita a escrita |
| T-POL-109 | NFR-009 | Reprodução de 10⁴ decisões | 100% idênticas |
| T-POL-110 | NFR-010 | Regra artificialmente custosa | 100% respondem em ≤ 50 ms ou `deny` |
| T-POL-111 | NFR-011 | Bundle com cobertura 0,85 | Publicação barrada (`AIOS-POL-0007`) |
| T-POL-112 | NFR-012 | Simulação de 10⁶ com produção sob carga | ≤ 10 min; p99 de produção inalterado |
| T-POL-113 | NFR-013 | Replay de mutações com mesma chave | Efeito único; mesma resposta |
| T-POL-114 | NFR-014 | Amostragem de spans | 100% das decisões com trace; 0 valor sensível |
| T-POL-115 | NFR-015 | Artefato adulterado | Réplica não fica *ready*; alerta dispara |
| T-POL-116 | NFR-016 | Requisição com tenant divergente | `AIOS-POL-0003`; RLS bloqueia mesmo com bypass de aplicação |

## 6. Testes de contrato

| Contrato | Contraparte | Verificação |
|----------|-------------|-------------|
| `DecisionRequest`/`DecisionResponse` (RFC-0220) | `../006-Kernel/`, `../004-API/`, `../009-Scheduler/`, `../010-Memory/` | Pact bidirecional; campos obrigatórios; `effect` restrito a `{allow,deny}` |
| Obrigações | `../007-Agent-Runtime/` (`sandbox_profile`), `../017-Model-Router/` (`limit_tokens`) | Obrigação desconhecida ⇒ `deny` |
| `policy.bundle.updated` / `.decision.updated` | Todos os PEPs | Schema + semântica de invalidação (total × granular) |
| Request-reply de decisão | `../020-Communication/` | Subject, timeout, preservação de `traceparent` |
| Atributos | `../021-Security/` (princípio), `../026-Cost-Optimizer/` (orçamento) | Forma e criticidade dos atributos consumidos |
| Trilha | `../025-Audit/` | Campos mínimos do fato de decisão |

## 7. Fixtures e ambientes

| Fixture | Conteúdo |
|---------|----------|
| `bundle-minimal` | 3 regras, 1 papel — testes de unidade |
| `bundle-realistic` | 800 regras, 40 papéis, 5.000 vínculos — testes de latência |
| `bundle-stress` | 10.000 regras, 10⁵ vínculos — NFR-005 |
| `bundle-conflicting` | Contradições e sombreamentos deliberados — `ConflictAnalyzer` |
| `bundle-pathological` | Condições de custo alto — orçamento de avaliação |
| `journal-1M` | 10⁶ decisões sintéticas — simulação (NFR-012) |

| Ambiente | Uso | Diferenças de produção |
|----------|-----|------------------------|
| Local (Compose) | Unidade/integração | `fail_mode=open`, `conflict.mode=warn`, gates desligados |
| CI efêmero | Todos os níveis exceto carga longa | Igual a produção nas chaves de governança |
| Homologação | E2E, carga, caos | **Idêntico** a produção (`./Configuration.md` §9) |

## 8. Gates de qualidade (CI)

Um PR **NÃO DEVE** ser mesclado se qualquer item falhar:

- [ ] Unitários ≥ 90%; **100%** dos ramos de combinação de decisão.
- [ ] Nenhum teste de contrato quebrado (Pact).
- [ ] `T-POL-008`, `T-POL-019` e `T-POL-108` verdes — os três que provam o *default deny*.
- [ ] `T-POL-023` verde — nenhum valor sensível em telemetria.
- [ ] Benchmark de latência sem regressão > 10% vs. baseline (`./Benchmark.md`).
- [ ] Migrações validadas (CV-11/CV-12) e reversíveis.
- [ ] Nenhum `TODO` nos 26 documentos do módulo.

## 9. Teste da política do tenant (gate de publicação)

Independente do CI do produto, todo bundle candidato passa por:

```
   Validated ──[PolicyTestRunner]──▶ 100% dos casos is_mandatory
                                     ∧ coverage ≥ 0,90
                                            │
                                            ▼
                                         Tested ──[SimulationEngine]──▶ diferencial
                                                                            │
                                                            revisão humana  ▼
                                                                        Simulated → Active
```

Um caso golden mínimo por tenant **DEVERIA** incluir: (a) um `allow` esperado do papel
mais comum; (b) um `deny` esperado de acesso cross-agente; (c) um `deny` esperado de
ação privilegiada por papel não privilegiado; (d) o comportamento esperado sob
orçamento esgotado.

## 10. Referências

- Requisitos: `./FunctionalRequirements.md`, `./NonFunctionalRequirements.md`
- Rastreabilidade: `./Requirements.md` §5 · Casos de uso: `./UseCases.md`
- Fronteiras testáveis: `./ClassDiagrams.md` §2 · Caos: `./FailureRecovery.md` §7
- Desempenho: `./Benchmark.md`

*Fim de `Testing.md`.*
