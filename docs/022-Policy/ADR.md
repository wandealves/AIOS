---
Documento: ADR
Módulo: 022-Policy
Status: Draft
Versão: 0.1
Última atualização: 2026-07-23
Responsável (RACI-A): Arquiteto do Módulo 022-Policy
ADRs relacionados: ADR-0001, ADR-0002, ADR-0004, ADR-0005, ADR-0006, ADR-0008, ADR-0010; ADR-0220..ADR-0229
RFCs relacionados: RFC-0001, RFC-0220, RFC-0221
Depende de: 002-ADR, _DESIGN_BRIEF.md §11
---

# 022-Policy — Índice de ADRs

Faixa reservada: **`ADR-0220`..`ADR-0229`** (regra `NNN × 10`). As ADRs são registradas
em `../002-ADR/` — este documento é o índice com o impacto de cada decisão sobre o
módulo.

---

## 1. ADRs globais herdadas

| ADR | Decisão | Impacto no 022-Policy | Status |
|-----|---------|------------------------|--------|
| [ADR-0001](../002-ADR/ADR-0001-Sistema-Operacional-nao-Framework.md) | AIOS é um SO, não um framework | O módulo é o **monitor de referência** do SO, não uma biblioteca de autorização acoplada à aplicação. | Accepted |
| [ADR-0002](../002-ADR/ADR-0002-Microservicos-Control-Data-Plane.md) | Microserviços + split control/data plane | O PDP é serviço do control plane, consultado por PEPs em ambos os planos. | Accepted |
| [ADR-0004](../002-ADR/ADR-0004-NATS-como-Barramento.md) | NATS como barramento primário | Propagação de bundle e invalidação de cache por evento; decisão também por request-reply. | Accepted |
| [ADR-0005](../002-ADR/ADR-0005-PostgreSQL-pgvector-AGE.md) | PostgreSQL unificado | Schema `policy` com RLS; invariante I1 materializada em índice único parcial. | Accepted |
| [ADR-0006](../002-ADR/ADR-0006-Redis-Estado-Quente.md) | Redis para estado quente | Cache de atributos e contadores de limite por tenant. | Accepted |
| **[ADR-0008](../002-ADR/ADR-0008-Governanca-por-Politica-Default-Deny.md)** | **Governança por política, *default deny*** | **Decisão-mãe deste módulo.** Fixa PEP/PDP, *default deny*, política como dado, cache de decisão e a exigência de teste e simulação de política. Todo o 022 é a materialização desta ADR. | Accepted |
| [ADR-0010](../002-ADR/ADR-0010-Observabilidade-Auditoria-por-Construcao.md) | Observabilidade e auditoria por construção | Toda decisão gera trace e trilha correlacionada; explicabilidade obrigatória. | Accepted |

---

## 2. ADRs a propor por este módulo

| ADR | Título | Escopo da decisão | Documentos afetados | Status |
|-----|--------|-------------------|---------------------|--------|
| ADR-0220 | Domínio de erro `POL` e registro do domínio de eventos `policy` | Reserva de `AIOS-POL-0001..0099`; registro de subjects e streams `POLICY_*` conforme RFC-0001 §8. | `API.md`, `Events.md` | Proposed |
| ADR-0221 | Política como artefato de release: bundle compilado, versionado, assinado e reversível | A decisão estruturante do módulo: política é **dado com ciclo de release**, não configuração. Implica imutabilidade após `Validated`, assinatura, rollback e reprodutibilidade da decisão. | `StateMachine.md`, `Database.md`, `Security.md`, `Vision.md` | Proposed |
| ADR-0222 | Avaliação central com bundle em memória vs. avaliação embarcada no PEP | A decisão de desempenho dominante: manter um ponto único de decisão e auditoria, aceitando o custo de rede, mitigado por cache no PEP. Alternativa embarcada preservada sob RFC-0221 para evolução futura. | `Architecture.md`, `Scalability.md`, `Benchmark.md` | Proposed |
| ADR-0223 | Precedência canônica de combinação (*deny-overrides* com prioridade explícita) | Como múltiplas regras casadas se combinam de forma determinística e auditável, em vez de depender da ordem de avaliação. | `StateMachine.md`, `ClassDiagrams.md`, `Testing.md` | Proposed |
| ADR-0224 | Colapso de `indeterminate`/`not_applicable` em `deny` na fronteira do PDP | O PEP vê apenas `allow`/`deny`. Evita que cada módulo invente tratamento próprio para "não sei" — e que algum o trate como "pode". | `API.md`, `StateMachine.md`, `FailureRecovery.md` | Proposed |
| ADR-0225 | Obrigações na resposta e a regra do "PEP que não cumpre, nega" | Permite `allow` condicional (redigir PII, exigir MFA, impor perfil de sandbox) sem que a condição vire comentário. | `API.md`, `Testing.md`, `Security.md` | Proposed |
| ADR-0226 | Gate obrigatório de testes golden e simulação antes da publicação | Torna impossível publicar política sem prova prévia do efeito; o custo é latência de processo, não de runtime. | `StateMachine.md`, `Testing.md`, `UseCases.md` | Proposed |
| ADR-0227 | Exceções (waivers) com prazo máximo e aprovação dupla | Reconhece que a urgência operacional existe e a canaliza para uma válvula com prazo, em vez de deixá-la afrouxar a política permanentemente. | `Database.md`, `Security.md`, `Monitoring.md` | Proposed |
| ADR-0228 | Autogoverno do PDP: bundle-raiz `platform`, separação de funções e cerimônia | Resolve o problema de o PDP autorizar a si mesmo sem criar um caminho de escalada de privilégio. | `Security.md`, `UseCases.md`, `Configuration.md` | Proposed |
| ADR-0229 | Journal de decisões de retenção curta no `022` vs. trilha imutável no `025` | Evita duas fontes da verdade de auditoria com regras de expurgo divergentes; define o que cada um prova. | `Logging.md`, `Database.md`, `Events.md` | Proposed |

---

## 3. Decisões que este módulo **não** toma

| Assunto | Onde a decisão vive |
|---------|---------------------|
| Modelo de identidade, credenciais e atributos assertáveis | `../021-Security/ADR.md` (ADR-0210) |
| Formato e retenção da trilha imutável | `../025-Audit/` |
| Definição de orçamento e contabilidade de custo | `../026-Cost-Optimizer/` |
| Comportamento do PEP sob indisponibilidade do PDP | Módulo do PEP (ex.: `../006-Kernel/ADR.md` — ADR-0063) + RFC-0001 §5.8 |
| Perfis de isolamento (conteúdo e assinatura) | `../021-Security/ADR.md` (ADR-0217) |

O 022 **exige** um perfil de sandbox por obrigação, mas não decide o que o perfil
contém; **lê** o orçamento, mas não decide o limite. Manter essa disciplina é o que
evita que o módulo de política vire o depósito de toda decisão difícil do sistema.

---

## 4. Registro e evolução

- Novas ADRs deste módulo **DEVEM** usar a faixa `ADR-0220..ADR-0229` e ser
  registradas em `../002-ADR/README.md`.
- Esgotada a faixa, a extensão **DEVE** ser negociada com Arquitetura-Chefe — não se
  invade a faixa de outro módulo.
- ADRs não são editadas após `Accepted`: são **substituídas**
  (`Superseded by ADR-XXXX`), conforme `../002-ADR/README.md`.
- Uma decisão que contrarie a RFC-0001 ou o ADR-0008 **NÃO DEVE** ser tomada aqui:
  exige RFC que obsolete a baseline.

*Fim de `ADR.md`.*
