---
Documento: ADR-0008 — Governança por política com default deny (PEP/PDP)
Módulo: 002-ADR
Status: Accepted
Versão: 1.0
Última atualização: 2026-07-20
Responsável (RACI-A): Arquitetura-Chefe, CISO
Decisores: Steering Committee, Segurança
Módulos afetados: 006, 015, 017, 021, 022, 025
---

# ADR-0008: Governança por política com *default deny* (PEP/PDP)

## Contexto
Agentes autônomos executam ações potencialmente perigosas (ferramentas, gastos,
acesso a dados). Ambientes Enterprise exigem autorização granular, auditável e
centralizada, com compliance (LGPD/GDPR). Autorização espalhada pelo código é
inauditável e insegura.

## Problema
Como impor autorização consistente, granular e auditável a todas as ações
privilegiadas de agentes e usuários?

## Alternativas
1. **Checagens de permissão *ad hoc*** no código de cada serviço.
2. **RBAC simples** centralizado.
3. **PEP/PDP com RBAC + ABAC** e *default deny*: pontos de aplicação (PEP) em cada serviço consultam um ponto de decisão (PDP) central.

## Análise
| Critério | Ad hoc | RBAC simples | PEP/PDP RBAC+ABAC |
|----------|--------|--------------|-------------------|
| Granularidade | Baixa | Média | Alta (atributos) |
| Auditabilidade | Fraca | Média | Forte (decisão central logada) |
| Consistência | Fraca | Média | Forte |
| Default deny | Improvável | Possível | Nativo |
| Evolução de política sem deploy | Não | Difícil | Sim (política como dado) |

RBAC sozinho não expressa contexto (custo acumulado, sensibilidade do dado,
horário). ABAC adiciona atributos. PEP/PDP centraliza a decisão, tornando-a
auditável e evoluível sem redeploy.

## Escolha
Adotamos **PEP/PDP com RBAC + ABAC** e postura **default deny**. Todo serviço
possui PEP; o **Policy Engine (022)** é o PDP. O Gateway aplica a primeira camada;
o Kernel aplica na fronteira de syscalls cognitivas. Toda decisão é auditada (025).

## Consequências
**Positivas:** autorização granular, consistente e auditável; políticas como dado;
compliance facilitada.
**Negativas:** latência de decisão (mitigada por cache de decisão); complexidade
de modelagem de políticas.
**Neutras:** política é versionada e testável (`022-Policy/Testing.md`).

## Riscos
| Risco | Prob. | Impacto | Mitigação |
|-------|-------|---------|-----------|
| PDP vira gargalo/SPOF | Média | Alto | PDP replicado + cache de decisão local no PEP. |
| Políticas mal escritas bloqueiam/liberam demais | Média | Alto | Testes de política; simulação/dry-run; revisão. |
| Latência adicional | Média | Médio | Cache de decisão com TTL curto; avaliação local. |

## Trade-offs
Ganhamos segurança, consistência e auditabilidade; pagamos com latência de decisão
e a disciplina de modelar e testar políticas.

## Referências
`../000-Vision/Vision.md` §6; `../021-Security/`; `../022-Policy/`; `../025-Audit/`.
