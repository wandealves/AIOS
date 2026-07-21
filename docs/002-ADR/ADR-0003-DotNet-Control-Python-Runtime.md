---
Documento: ADR-0003 — .NET 10 no Control Plane, Python no Agent Runtime
Módulo: 002-ADR
Status: Accepted
Versão: 1.0
Última atualização: 2026-07-20
Responsável (RACI-A): Arquitetura-Chefe
Decisores: Steering Committee
Módulos afetados: 001, 006, 007, 017, 031
---

# ADR-0003: .NET 10 no Control Plane, Python no Agent Runtime

## Contexto
O plano de controle tem requisitos de desempenho, tipagem forte, concorrência
madura e operação Enterprise. O runtime de agentes precisa do ecossistema de IA
(bibliotecas de LLM, MCP, frameworks de raciocínio), majoritariamente em Python.

## Problema
Qual(is) linguagem(ns) usar para o control plane e para o runtime dos agentes?

## Alternativas
1. **Tudo em Python.**
2. **Tudo em .NET.**
3. **.NET 10 no control plane + Python no runtime** (poliglota por fronteira).
4. **Go/Rust no control plane + Python no runtime.**

## Análise
| Critério | Tudo Python | Tudo .NET | .NET+Python | Go/Rust+Python |
|----------|-------------|-----------|-------------|----------------|
| Perf/latência control plane | Fraco (GIL) | Forte | Forte | Muito forte |
| Ecossistema IA no runtime | Forte | Fraco | Forte | Forte |
| Tipagem/segurança | Fraca | Forte | Forte | Forte |
| Maturidade async | Média | Alta | Alta | Alta |
| Produtividade Enterprise | Média | Alta | Alta | Média |
| Curva/hiring | — | — | 2 stacks | 2 stacks + curva Rust |

Python no control plane sofre com GIL e tipagem para serviços críticos de baixa
latência. .NET no runtime perderia o ecossistema de IA. Go/Rust dariam perf, mas
.NET 10 já entrega desempenho (AOT), maturidade Enterprise e produtividade
superiores para a equipe-alvo.

## Escolha
Adotamos **Opção 3**: **Control Plane em .NET 10**; **Agent Runtime em Python**. A
fronteira é estrita (contratos gRPC/NATS). O Model Router (017) abstrai provedores
de LLM independentemente da linguagem.

## Consequências
**Positivas:** melhor ferramenta para cada plano; runtime aproveita o ecossistema
de IA; control plane rápido e tipado.
**Negativas:** duas stacks (build, CI, hiring, observabilidade).
**Neutras:** SDK (031) oferecido em ambas para clientes.

## Riscos
| Risco | Prob. | Impacto | Mitigação |
|-------|-------|---------|-----------|
| Custo de duas stacks | Alta | Médio | Fronteira clara; contratos versionados; OTel comum. |
| Divergência de modelos de erro | Média | Médio | Envelope de erro padronizado (`004-API`). |
| Serialização entre planos | Média | Baixo | Protobuf/JSON schema compartilhado. |

## Trade-offs
Ganhamos "a ferramenta certa para cada trabalho"; pagamos com a complexidade de
operar e contratar para duas plataformas.

## Referências
ADR-0002; `../001-Architecture/Architecture.md` §14; `../007-Agent-Runtime/`.
