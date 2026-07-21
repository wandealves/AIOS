---
Documento: ADR-0007 — Memória hierárquica de sete camadas
Módulo: 002-ADR
Status: Accepted
Versão: 1.0
Última atualização: 2026-07-20
Responsável (RACI-A): Arquitetura-Chefe
Decisores: Steering Committee, Cientista-Chefe
Módulos afetados: 010, 011, 018, 019, 023
---

# ADR-0007: Memória hierárquica de sete camadas

## Contexto
A abordagem dominante (janela de contexto + vector DB + resumo) sofre de perda de
contexto, recuperação imperfeita, ausência de prioridades e de esquecimento
controlado (ver `../especificacao.md` §3). O AIOS propõe memória como recurso
gerenciado, análogo à hierarquia de memória de um SO (registradores→cache→RAM→disco).

## Problema
Qual modelo de memória adotar para superar as limitações do RAG plano e permitir
consolidação, priorização e esquecimento?

## Alternativas
1. **RAG plano**: janela de contexto + um vector store + resumo.
2. **Duas camadas**: curto prazo (buffer) + longo prazo (vetorial).
3. **Hierarquia de sete camadas**: Working → Short → Long → Semantic → Procedural → Episodic → Knowledge Graph, cada uma com política própria de retenção/consolidação/esquecimento.

## Análise
| Critério | RAG plano | Duas camadas | Sete camadas |
|----------|-----------|--------------|--------------|
| Retenção diferenciada | Não | Parcial | Sim |
| Consolidação (curto→longo) | Não | Manual | Política automática |
| Esquecimento controlado | Não | Não | Sim (por camada) |
| Memória procedural/episódica | Não | Não | Sim |
| Complexidade | Baixa | Média | Alta |
| Potencial científico | Baixo | Médio | Alto |

A hierarquia de sete camadas endereça diretamente P1/P6 da Visão e habilita
métricas cognitivas (Memory Recall Rate, Knowledge Reuse Rate). A complexidade é
gerida por interfaces uniformes por camada.

## Escolha
Adotamos a **hierarquia de sete camadas** (Working, Short-Term, Long-Term,
Semantic, Procedural, Episodic, Knowledge Graph), cada uma com políticas próprias
de retenção, consolidação e esquecimento, sob uma API de Memory (010) uniforme.
Consolidação é dirigida pelo Learning Engine (023).

## Consequências
**Positivas:** priorização, consolidação e esquecimento explícitos; suporte a
memória procedural/episódica; base de diferenciação científica.
**Negativas:** maior complexidade de implementação e tuning de políticas.
**Neutras:** cada camada mapeia a um store físico (Redis/PG+pgvector/AGE/MinIO).

## Riscos
| Risco | Prob. | Impacto | Mitigação |
|-------|-------|---------|-----------|
| Crescimento descontrolado | Média | Alto | Cotas + esquecimento por camada; Cost Optimizer. |
| Catastrophic forgetting | Média | Alto | Consolidação versionada + rollback (023). |
| Overhead de recuperação | Média | Médio | Cache semântico (011); recuperação seletiva. |

## Trade-offs
Ganhamos capacidade cognitiva e diferenciação; pagamos com complexidade de
implementação e necessidade de tuning de políticas por camada.

## Referências
`../especificacao.md` §5.3; `../010-Memory/`; `../011-Context/`; `../023-Learning/`.
