---
Documento: ADR-0009 — Model Router multiobjetivo (custo × qualidade × latência)
Módulo: 002-ADR
Status: Accepted
Versão: 1.0
Última atualização: 2026-07-20
Responsável (RACI-A): Arquitetura-Chefe
Decisores: Steering Committee, FinOps
Módulos afetados: 011, 017, 026
---

# ADR-0009: Model Router multiobjetivo

## Contexto
A inferência é o recurso de maior custo do AIOS. Usar sempre o maior modelo é caro
e lento; usar sempre o menor degrada qualidade. Diferentes provedores/modelos
variam em custo, qualidade, latência e disponibilidade ao longo do tempo.

## Problema
Como escolher, por requisição, o modelo que minimiza custo/latência/energia sem
violar a qualidade mínima e o orçamento?

## Alternativas
1. **Modelo fixo** configurado por agente.
2. **Regras estáticas** (ex.: tarefa X → modelo Y).
3. **Router multiobjetivo**: decide por requisição via política `min(αL+β$+γE)` sujeita a `Q≥Q_min`, com drivers plugáveis por provedor.
4. **Router aprendido (RL)** desde o início.

## Análise
| Critério | Fixo | Regras estáticas | Multiobjetivo | RL puro |
|----------|------|------------------|---------------|---------|
| Custo otimizado | Não | Parcial | Sim | Sim |
| Qualidade garantida | Depende | Parcial | Sim (restrição) | Difícil de garantir |
| Adaptação a disponibilidade | Não | Não | Sim | Sim |
| Explicabilidade | Alta | Alta | Alta | Baixa |
| Complexidade | Baixa | Baixa | Média | Alta |
| Maturidade p/ v1 | — | — | Adequada | Arriscada |

RL puro é difícil de garantir/explicar no v1. O router multiobjetivo com
heurísticas explicáveis entrega ganho imediato e evolui para políticas aprendidas
(v2.0+) sem mudar a interface.

## Escolha
Adotamos um **Model Router multiobjetivo** com função de custo
`min(αL + β$ + γE)` sujeita a `Q ≥ Q_min` e orçamento, com **drivers plugáveis**
por provedor (ADR-0003) e **cache semântico** (011) à frente. Políticas aprendidas
por RL são um caminho evolutivo, não pré-requisito.

## Consequências
**Positivas:** redução de custo/latência com garantia de qualidade; independência
de provedor; evolução para RL sem quebra de contrato.
**Negativas:** necessidade de estimar qualidade/latência por modelo; calibração de
pesos por SLA.
**Neutras:** integra-se ao Cost Optimizer (026) e ao Context/cache (011).

## Riscos
| Risco | Prob. | Impacto | Mitigação |
|-------|-------|---------|-----------|
| Estimativa de qualidade imprecisa | Média | Médio | Feedback online + benchmarks (035); fallback conservador. |
| Provedor indisponível | Média | Alto | Circuit breaker + failover para modelo alternativo. |
| Oscilação de decisão | Baixa | Baixo | Histerese/estabilização na política. |

## Trade-offs
Ganhamos eficiência de custo com qualidade garantida; pagamos com a necessidade de
telemetria de qualidade/custo por modelo e calibração de política.

## Referências
`../000-Vision/Vision.md` §8; `../017-Model-Router/`; `../026-Cost-Optimizer/`.
