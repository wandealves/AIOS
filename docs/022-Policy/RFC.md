---
Documento: RFC
Módulo: 022-Policy
Status: Draft
Versão: 0.1
Última atualização: 2026-07-23
Responsável (RACI-A): Arquiteto do Módulo 022-Policy
ADRs relacionados: ADR-0008, ADR-0220, ADR-0221, ADR-0222, ADR-0223, ADR-0224, ADR-0225
RFCs relacionados: RFC-0001, RFC-0220, RFC-0221
Depende de: 003-RFC, _DESIGN_BRIEF.md §11
---

# 022-Policy — Índice de RFCs

RFCs são registradas em `../003-RFC/`. Este documento lista as que afetam o módulo e o
impacto de cada uma.

---

## 1. RFCs consumidas (baseline)

| RFC | Título | Status | Impacto no 022-Policy |
|-----|--------|--------|------------------------|
| **RFC-0001** | [Architecture Baseline & Core Contracts](../003-RFC/RFC-0001-Architecture-Baseline.md) | Accepted | **Baseline herdada, não redefinida.** A **§5.8** é o contrato que este módulo torna executável: *toda ação privilegiada DEVE passar por um PEP e ser autorizada pelo PDP (022) — default deny*. Além dela: URN (§5.1) para bundles, regras, papéis e exceções; envelope CloudEvents (§5.2–§5.3) para o domínio `policy`, cujo subject `aios.<tenant>.policy.decision.denied` é **exemplo normativo da própria RFC**; envelope de erro (§5.4) para o domínio `POL`; idempotência (§5.5) nas mutações administrativas; correlação (§5.6); versionamento (§5.7); mTLS e fronteira de tenant (§6); minimização de dados pessoais (§7); registros IANA-like (§8). |

O módulo **não reescreve** nenhum desses contratos. A RFC-0001 §5.8 diz *que* toda ação
privilegiada passa pelo PDP; este módulo especifica **como a decisão é formada,
provada, versionada e explicada**.

---

## 2. RFCs a propor por este módulo

| RFC | Título | Escopo normativo | Relação com RFC-0001 | Status |
|-----|--------|------------------|----------------------|--------|
| **RFC-0220** | AIOS Policy Decision Contract | Esquema de `DecisionRequest`/`DecisionResponse` (JSON, proto e payload de request-reply); conjunto **fechado e estável** de `reason_code`; catálogo de `obligation.kind` e a regra normativa "PEP que não sabe cumprir, nega"; semântica do `decision_ttl_ms` (incluindo `deny_ttl ≤ allow_ttl`); precedência canônica de combinação; colapso de `indeterminate`/`not_applicable`; garantias de determinismo e reprodutibilidade da decisão. | Especializa §5.8 para o plano de decisão; consome §5.1, §5.4, §5.5, §5.6 e §5.7 sem redefini-los. | Proposed |
| **RFC-0221** | AIOS Policy Bundle Format & Distribution | Formato do bundle-fonte (regra, papel, vínculo, condição ABAC como AST); regras de compilação e indexação; algoritmo de detecção de conflito (contradição, sombreamento, inalcançável); formato e assinatura do artefato compilado; protocolo de distribuição e convergência entre réplicas; retenção mínima para rollback e reprodutibilidade; **perfil opcional de avaliação embarcada no PEP** (alternativa preservada em ADR-0222). | Usa o envelope de evento (§5.2–§5.3) para a propagação e a §6 para a assinatura/integridade. | Proposed |

---

## 3. RFCs de outros módulos que este módulo consome

| RFC | Módulo | Por que importa aqui |
|-----|--------|----------------------|
| RFC-0210 | `../021-Security/` | Define os **atributos assertáveis** do princípio — o insumo de sujeito que o PDP interpreta. A fronteira entre asserção (021) e interpretação (022) está fixada ali e em ADR-0210. |
| RFC-0211 | `../021-Security/` | Identidade de workload e PKI: sustenta o mTLS que autentica cada PEP e a assinatura do artefato de bundle. |
| RFC-0200 / RFC-0201 | `../020-Communication/` | Contrato do barramento: subjects, request-reply e streams JetStream usados na propagação e na decisão por mensagem. |

---

## 4. Registro e evolução

- A numeração `RFC-0220`/`RFC-0221` segue a convenção de faixa por módulo já adotada
  por `../020-Communication/` (RFC-0200/0201) e `../021-Security/` (RFC-0210/0211).
- Novos `reason_code`, `obligation.kind` e valores de `changeKind` são **aditivos** e
  registrados na RFC-0220; remoção ou renomeação exige nova versão da RFC e período de
  coexistência (RFC-0001 §5.7 e §9).
- Alteração incompatível do formato do bundle exige nova versão de RFC-0221 **e**
  recompilação/republicação dos bundles antes do rollout — o motor novo **NÃO DEVE**
  reinterpretar artefato antigo (`./Deployment.md` §6).
- Nenhuma RFC deste módulo **PODE** contradizer a RFC-0001; alteração da baseline exige
  RFC que a obsolete, com período de coexistência.

*Fim de `RFC.md`.*
