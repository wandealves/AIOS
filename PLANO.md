# AIOS — Plano de Desenvolvimento

> **Documento vivo.** Este é o rastreador oficial do progresso da documentação do AIOS.
> A cada documento ou módulo concluído, **marque o checkbox**, **atualize o contador `(X/26)`**,
> **recalcule a barra de progresso global** e **atualize a data** abaixo.

**Última atualização:** 2026-07-22

**Legenda:** ✅ concluído · 🟨 em andamento · ⬜ pendente

---

## Progresso global

| Métrica | Valor |
|---|---|
| Fundação canônica (000–003, 040) | ✅ 5/5 |
| Módulos técnicos completos (004–039) | **10 / 36** |
| Módulos técnicos em andamento | 0 |
| Documentos técnicos escritos | **258 / 936** (36 módulos × 26 docs) |

```
Fundação   [██████████] 100%   (000, 001, 002, 003, 040)
Módulos    [███░░░░░░░░]  28%   (10 completos / 36 técnicos)
```

---

## Definition of Done (por módulo)

Um módulo `NNN-Nome` só é ✅ quando:

1. **`_DESIGN_BRIEF.md`** escrito (fonte única de verdade interna) — skill `aios-design-brief`.
2. **26 documentos obrigatórios** derivados do brief — skill `aios-module` / `_templates/MODULE_TEMPLATE.md`.
   Nenhum doc pode ser só lista de títulos; cabeçalho de metadados preenchido; diagramas em ASCII.
3. **Auditoria** sem defeitos bloqueantes — skill `aios-doc-review` (contra template + brief + RFC-0001 + Glossário 040).
4. **Consistência:** não redefine termos do `040-Glossary` nem contratos da `003-RFC/RFC-0001`.

**Os 26 documentos:** `Vision`, `Architecture`, `Requirements`, `FunctionalRequirements`,
`NonFunctionalRequirements`, `UseCases`, `SequenceDiagrams`, `ClassDiagrams`, `StateMachine`,
`Database`, `API`, `Events`, `Configuration`, `Deployment`, `Security`, `Monitoring`, `Logging`,
`Metrics`, `Testing`, `Benchmark`, `FailureRecovery`, `Scalability`, `Examples`, `FAQ`, `ADR`, `RFC`.

> As ondas abaixo são a **ordem recomendada de trabalho** (não um bloqueio rígido): a fundação já
> está pronta e todos os módulos referenciam a RFC-0001, então qualquer módulo é escrevível. A ordem
> prioriza fechar o que está aberto e depois os contratos transversais que os demais mais referenciam.

---

## Passo 0 — Fundação + núcleo ✅ CONCLUÍDO

Base terminológica/arquitetural e os primeiros módulos no padrão de ouro.

- [x] ✅ **000-Vision** — visão global, problema, princípios
- [x] ✅ **001-Architecture** — arquitetura de referência (C4/UML ASCII)
- [x] ✅ **002-ADR** — 10 decisões-semente
- [x] ✅ **003-RFC** — RFC-0001 Architecture Baseline (contratos centrais)
- [x] ✅ **040-Glossary** — glossário canônico
- [x] ✅ **006-Kernel** (26/26) — padrão de ouro de referência
- [x] ✅ **008-Agent-Lifecycle** (26/26)
- [x] ✅ **009-Scheduler** (26/26)
- [x] ✅ **010-Memory** (26/26)

---

## Passo 1 — Concluir em andamento + contratos base ✅ CONCLUÍDO

Fechar os módulos já iniciados e escrever os contratos (API/DB) que os demais mais referenciam.

- [x] ✅ **011-Context** (26/26) — `[x]` brief · `[x]` 26 docs · `[x]` auditoria (0 bloqueantes; defeitos Alto/Médio/Baixo corrigidos)
- [x] ✅ **007-Agent-Runtime** (26/26) — `[x]` brief · `[x]` 26 docs · `[x]` auditoria (0 bloqueantes; defeitos Alto/Médio/Baixo corrigidos)
- [x] ✅ **004-API** (26/26) — `[x]` brief · `[x]` 26 docs · `[x]` auditoria (0 bloqueantes; defeitos Maiores/menores corrigidos)
- [x] ✅ **005-Database** (26/26) — `[x]` brief · `[x]` 26 docs · `[x]` auditoria (0 bloqueantes; rastreabilidade T-BKP-03 corrigida)

---

## Passo 2 — Plataforma transversal 🟨

Contratos de comunicação, segurança, política, observabilidade e auditoria referenciados por todo o sistema.

- [x] ✅ **020-Communication** (26/26) — `[x]` brief · `[x]` 26 docs · `[x]` auditoria (0 bloqueantes; recording rule `aios_bus_sessions_rejected` documentada)
- [x] ✅ **021-Security** (26/26) — `[x]` brief · `[x]` 26 docs · `[x]` auditoria (0 bloqueantes)
- [ ] ⬜ **022-Policy** (0/26) — `[ ]` brief · `[ ]` 26 docs · `[ ]` auditoria
- [ ] ⬜ **024-Observability** (0/26) — `[ ]` brief · `[ ]` 26 docs · `[ ]` auditoria
- [ ] ⬜ **025-Audit** (0/26) — `[ ]` brief · `[ ]` 26 docs · `[ ]` auditoria

---

## Passo 3 — Núcleo cognitivo ⬜

Planejamento, objetivos, workflow e roteamento de modelos.

- [ ] ⬜ **012-Planning** (0/26) — `[ ]` brief · `[ ]` 26 docs · `[ ]` auditoria
- [ ] ⬜ **013-Goals** (0/26) — `[ ]` brief · `[ ]` 26 docs · `[ ]` auditoria
- [ ] ⬜ **014-Workflow** (0/26) — `[ ]` brief · `[ ]` 26 docs · `[ ]` auditoria
- [ ] ⬜ **017-Model-Router** (0/26) — `[ ]` brief · `[ ]` 26 docs · `[ ]` auditoria

---

## Passo 4 — Conhecimento & extensão ⬜

Ferramentas, plugins, conhecimento, GraphRAG e aprendizado contínuo.

- [ ] ⬜ **015-Tool-Manager** (0/26) — `[ ]` brief · `[ ]` 26 docs · `[ ]` auditoria
- [ ] ⬜ **016-Plugin-System** (0/26) — `[ ]` brief · `[ ]` 26 docs · `[ ]` auditoria
- [ ] ⬜ **018-Knowledge** (0/26) — `[ ]` brief · `[ ]` 26 docs · `[ ]` auditoria
- [ ] ⬜ **019-GraphRAG** (0/26) — `[ ]` brief · `[ ]` 26 docs · `[ ]` auditoria
- [ ] ⬜ **023-Learning** (0/26) — `[ ]` brief · `[ ]` 26 docs · `[ ]` auditoria

---

## Passo 5 — Escala & operação ⬜

Custo, cluster/HA, implantação e operação.

- [ ] ⬜ **026-Cost-Optimizer** (0/26) — `[ ]` brief · `[ ]` 26 docs · `[ ]` auditoria
- [ ] ⬜ **027-Cluster** (0/26) — `[ ]` brief · `[ ]` 26 docs · `[ ]` auditoria
- [ ] ⬜ **028-Deployment** (0/26) — `[ ]` brief · `[ ]` 26 docs · `[ ]` auditoria
- [ ] ⬜ **029-Operations** (0/26) — `[ ]` brief · `[ ]` 26 docs · `[ ]` auditoria

---

## Passo 6 — Interfaces & experiência ⬜

CLI, SDKs e console web.

- [ ] ⬜ **030-CLI** (0/26) — `[ ]` brief · `[ ]` 26 docs · `[ ]` auditoria
- [ ] ⬜ **031-SDK** (0/26) — `[ ]` brief · `[ ]` 26 docs · `[ ]` auditoria
- [ ] ⬜ **032-WebConsole** (0/26) — `[ ]` brief · `[ ]` 26 docs · `[ ]` auditoria

---

## Passo 7 — Guias, pesquisa & fechamento ⬜

Guias, benchmark, pesquisa científica e roadmap por versões.

- [ ] ⬜ **033-DeveloperGuide** (0/26) — `[ ]` brief · `[ ]` 26 docs · `[ ]` auditoria
- [ ] ⬜ **034-AdministratorGuide** (0/26) — `[ ]` brief · `[ ]` 26 docs · `[ ]` auditoria
- [ ] ⬜ **035-Benchmark** (0/26) — `[ ]` brief · `[ ]` 26 docs · `[ ]` auditoria
- [ ] ⬜ **036-Experiments** (0/26) — `[ ]` brief · `[ ]` 26 docs · `[ ]` auditoria
- [ ] ⬜ **037-Papers** (0/26) — `[ ]` brief · `[ ]` 26 docs · `[ ]` auditoria
- [ ] ⬜ **038-Patents** (0/26) — `[ ]` brief · `[ ]` 26 docs · `[ ]` auditoria
- [ ] ⬜ **039-Roadmap** (0/26) — `[ ]` brief · `[ ]` 26 docs · `[ ]` auditoria

---

## Como atualizar este plano

1. Ao concluir um **documento** de um módulo em andamento, incremente o contador `(X/26)`.
2. Ao concluir uma **etapa** do módulo (brief, 26 docs, auditoria), marque o sub-checkbox correspondente.
3. Quando as três etapas do módulo estiverem feitas, troque o marcador para ✅ e marque o `[x]` do item.
4. Recalcule a linha **"Módulos técnicos completos"** e a **barra de progresso global** no topo.
5. Atualize a **data de "Última atualização"**.

### Fluxo de produção por módulo (skills do repositório)

1. `aios-design-brief` → escrever `docs/NNN-Nome/_DESIGN_BRIEF.md`.
2. `aios-module` → derivar os 26 documentos a partir do brief.
3. `aios-doc-review` → auditar e corrigir defeitos (READ-ONLY: gera relatório).
4. Marcar o módulo como ✅ aqui.
