---
Documento: ADR-0002 — Microserviços com split Control Plane / Data Plane
Módulo: 002-ADR
Status: Accepted
Versão: 1.0
Última atualização: 2026-07-20
Responsável (RACI-A): Arquitetura-Chefe
Decisores: Steering Committee
Módulos afetados: 001, 006, 007, 009, 027, 028
---

# ADR-0002: Microserviços com separação Control Plane / Data Plane

## Contexto
O AIOS precisa escalar subsistemas de forma independente (o Scheduler tem perfil
de carga distinto do Memory Manager), tolerar falhas parciais e permitir deploy
independente. Agentes são efêmeros e numerosos; o plano de execução tem perfil de
recurso e risco muito diferente do plano de decisão.

## Problema
Qual estilo arquitetural e qual fronteira de decomposição adotar?

## Alternativas
1. **Monólito modular** (.NET único, módulos internos).
2. **Microserviços sem split** (todos os serviços no mesmo plano).
3. **Microserviços com split Control Plane / Data Plane** (controle em serviços .NET; execução de agentes isolada no runtime).
4. **Kubernetes puro**: 1 pod por agente.

## Análise
| Critério | Monólito | µSvc s/ split | µSvc c/ split | K8s 1pod/agente |
|----------|----------|---------------|---------------|-----------------|
| Escala independente | Não | Sim | Sim | Sim |
| Isolamento de falha | Fraco | Médio | Forte | Forte |
| Custo p/ 10⁶ agentes efêmeros | Baixo | Médio | Baixo–Médio | Proibitivo |
| Blast radius de runtime malicioso | Alto | Médio | Baixo (isolado) | Baixo |
| Complexidade operacional | Baixa | Alta | Alta | Muito alta |
| Latência de spawn | Baixa | Média | Baixa (pool) | Alta (pod cold start) |

1 pod/agente é inviável para milhões de agentes efêmeros (cold start ~segundos,
overhead de pod). O split permite tratar o runtime como pool elástico barato e o
controle como serviços stateless replicáveis.

## Escolha
Adotamos **Opção 3**: microserviços orientados a domínio no **Control Plane (.NET 10)**
+ **Data Plane** de Agent Runtimes gerenciados por um **Runtime Supervisor**, com
pool elástico e sandbox por agente. Kubernetes é alvo futuro para *scheduling de
nós*, não para 1 pod/agente.

## Consequências
**Positivas:** escala e falha isoladas; blast radius contido; deploy independente.
**Negativas:** complexidade operacional (mitigada por operadores/defaults);
consistência eventual entre serviços.
**Neutras:** exige contratos estáveis (gRPC/NATS) entre módulos.

## Riscos
| Risco | Prob. | Impacto | Mitigação |
|-------|-------|---------|-----------|
| Sprawl de serviços | Média | Médio | Limitar nº de serviços; bounded contexts claros. |
| Consistência eventual confunde | Média | Médio | CQRS + read-your-writes em Redis; docs claras. |
| Overhead de rede interna | Baixa | Médio | gRPC + NATS de baixa latência; co-locação por AZ. |

## Trade-offs
Ganhamos escalabilidade e isolamento; pagamos com complexidade operacional e a
necessidade de disciplina de contratos e observabilidade.

## Referências
ADR-0001, ADR-0003; `../001-Architecture/Architecture.md` §4, §6, §16.
