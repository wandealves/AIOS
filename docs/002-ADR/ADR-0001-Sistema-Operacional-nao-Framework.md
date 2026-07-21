---
Documento: ADR-0001 — AIOS é um Sistema Operacional, não um Framework
Módulo: 002-ADR
Status: Accepted
Versão: 1.0
Última atualização: 2026-07-20
Responsável (RACI-A): Arquitetura-Chefe
Decisores: Steering Committee
Módulos afetados: 000, 001, 006, 009, 021, 022, 027
---

# ADR-0001: AIOS é um Sistema Operacional, não um Framework

## Contexto
Existem frameworks maduros de orquestração de agentes (LangGraph, AutoGen, CrewAI,
Semantic Kernel, OpenAI Agents SDK). Eles operam como bibliotecas *in-process*:
compõem chamadas a LLMs e ferramentas, mas não gerenciam recursos, não isolam
falhas, não escalonam sob contenção nem impõem governança. Os requisitos do AIOS
(1→10⁶ agentes, HA ≥99,95%, isolamento multi-tenant, auditoria imutável,
compliance) são de **sistema operacional/plataforma**, não de biblioteca.

## Problema
Devemos construir o AIOS *sobre* um framework de orquestração existente (como
núcleo), ou construir um plano de controle próprio com abstrações de SO
(kernel, scheduler, cotas, isolamento, política)?

## Alternativas
1. **Adotar um framework como núcleo** (ex.: LangGraph) e adicionar features ao redor.
2. **Construir um plano de controle próprio** (SO para agentes) e integrar frameworks apenas como *bibliotecas do runtime* do agente.
3. **Híbrido**: núcleo próprio para controle, mas expor grafos de framework como primeira classe.

## Análise
| Critério | (1) Framework-núcleo | (2) SO próprio | (3) Híbrido |
|----------|----------------------|----------------|-------------|
| Escala a 10⁶ agentes | Ruim (single-process) | Boa | Média |
| Isolamento/cotas | Ausente | Nativo | Parcial |
| HA / SPOF | Não previsto | Projetado | Parcial |
| Governança/auditoria | Fraca | Estrutural | Média |
| Lock-in | Alto | Baixo | Médio |
| Esforço inicial | Baixo | Alto | Alto |
| Alinhamento à Visão | Baixo | Total | Médio |

Frameworks resolvem *composição de raciocínio*, não *gerência de recursos*. Usá-los
como núcleo herdaria seus limites estruturais (P1–P8 da Visão).

## Escolha
Adotamos a **Opção 2**: o AIOS é um **sistema operacional para agentes** com plano
de controle próprio. Frameworks de orquestração podem ser usados **dentro** do
Agent Runtime como bibliotecas, nunca como núcleo do sistema.

## Consequências
**Positivas:** abstrações de SO (kernel, scheduler, cotas, isolamento) de primeira
classe; escala, HA e governança projetadas; sem lock-in de framework.
**Negativas:** maior esforço inicial; necessidade de reimplementar conveniências.
**Neutras:** desenvolvedores podem trazer seu framework favorito no runtime.

## Riscos
| Risco | Prob. | Impacto | Mitigação |
|-------|-------|---------|-----------|
| Reinventar a roda | Média | Médio | Reusar padrões de SO/distribuídos consagrados; SDK ergonômico (031). |
| Curva de adoção | Média | Médio | Compatibilidade com MCP/A2A e SDKs familiares. |

## Trade-offs
Ganhamos escala, isolamento e governança de nível Enterprise; abrimos mão da
velocidade inicial de "pegar uma lib pronta".

## Referências
`../000-Vision/Vision.md` §4, §13; `../001-Architecture/Architecture.md` §1.
