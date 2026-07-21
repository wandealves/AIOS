---
Documento: ADR-0004 — NATS + JetStream como barramento primário
Módulo: 002-ADR
Status: Accepted
Versão: 1.0
Última atualização: 2026-07-20
Responsável (RACI-A): Arquitetura-Chefe
Decisores: Steering Committee
Módulos afetados: 001, 014, 020, 025
---

# ADR-0004: NATS + JetStream como barramento primário

## Contexto
O AIOS precisa de comunicação assíncrona (eventos de domínio), síncrona leve
(request/reply de controle) e streaming durável (auditoria, event sourcing, DLQ,
workflows). O barramento é caminho quente para milhões de agentes.

## Problema
Qual tecnologia de mensageria adotar como barramento primário?

## Alternativas
1. **Apache Kafka.**
2. **RabbitMQ.**
3. **NATS + JetStream.**
4. **Redis Streams.**

## Análise
| Critério | Kafka | RabbitMQ | NATS+JetStream | Redis Streams |
|----------|-------|----------|----------------|---------------|
| Latência | Média | Média | Muito baixa | Baixa |
| Request/reply nativo | Não | Parcial | Sim | Não |
| Streaming durável/replay | Excelente | Bom | Bom (JetStream) | Limitado |
| Footprint operacional | Pesado | Médio | Leve | Leve |
| Retenção longa (log infinito) | Excelente | Fraco | Bom | Fraco |
| Multi-tenancy/namespaces | Médio | Médio | Forte (accounts) | Fraco |

Kafka é excelente para retenção longa mas pesado e sem request/reply nativo. NATS
entrega latência mínima, footprint pequeno, request/reply nativo e contas
multi-tenant, com JetStream cobrindo durabilidade/replay.

## Escolha
Adotamos **NATS + JetStream** como barramento primário (pub/sub, request/reply,
streams). **Kafka permanece opcional** como *sink* de auditoria/analytics de
retenção muito longa (ADR de módulo em `025-Audit` se necessário).

## Consequências
**Positivas:** baixa latência; um só sistema para os três padrões; multi-tenancy
por contas; operação leve.
**Negativas:** ecossistema de conectores menor que Kafka; retenção massiva exige
JetStream bem dimensionado ou sink externo.
**Neutras:** subjects padronizados `aios.<tenant>.<dominio>.<entidade>.<acao>`.

## Riscos
| Risco | Prob. | Impacto | Mitigação |
|-------|-------|---------|-----------|
| Limite de retenção JetStream | Média | Médio | Sink Kafka/MinIO para long-term audit. |
| Menos conectores prontos | Média | Baixo | Adapters próprios; padrões abertos. |
| Operação de cluster NATS | Baixa | Médio | Quorum 3 nós; runbooks (029). |

## Trade-offs
Ganhamos latência, simplicidade e request/reply nativo; abrimos mão do ecossistema
e da retenção "infinita" de Kafka (recuperados via sink opcional).

## Referências
`../001-Architecture/Architecture.md` §9; `../020-Communication/`; `../025-Audit/`.
