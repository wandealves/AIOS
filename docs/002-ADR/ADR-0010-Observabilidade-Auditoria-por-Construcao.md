---
Documento: ADR-0010 — Observabilidade e auditoria por construção
Módulo: 002-ADR
Status: Accepted
Versão: 1.0
Última atualização: 2026-07-20
Responsável (RACI-A): Arquitetura-Chefe, SRE-Lead, CISO
Decisores: Steering Committee
Módulos afetados: 024, 025, e todos os módulos
---

# ADR-0010: Observabilidade e auditoria por construção

## Contexto
Sistemas de agentes autônomos são difíceis de depurar e auditar. Enterprise e
compliance (LGPD/GDPR) exigem reconstruir a proveniência de qualquer decisão.
Adicionar observabilidade depois é caro e incompleto ("se não é observável, não
existe").

## Problema
Como garantir que todo caminho relevante seja observável e que toda ação
privilegiada seja auditável de forma imutável, sem depender de esforço posterior?

## Alternativas
1. **Logging opcional** por serviço (adicionado conforme necessidade).
2. **Métricas + logs** padronizados, sem tracing distribuído.
3. **OTel end-to-end (traces+metrics+logs) + auditoria event-sourced imutável** como requisito transversal obrigatório.

## Análise
| Critério | Logging opcional | Métricas+logs | OTel + audit event-sourced |
|----------|------------------|---------------|-----------------------------|
| Depuração distribuída | Fraca | Média | Forte (trace correlacionado) |
| Proveniência de decisão | Não | Parcial | Total |
| Compliance (LGPD/GDPR) | Difícil | Parcial | Nativo (trilha imutável) |
| Custo de adição posterior | Alto | Médio | Zero (por construção) |
| Overhead runtime | Baixo | Baixo | Médio (mitigável por sampling) |

## Escolha
Adotamos **observabilidade e auditoria por construção** como requisito
transversal: **OpenTelemetry** (traces, métricas, logs) em todo serviço com
`trace_id/span_id/tenant_id/agent_id` correlacionados; **auditoria event-sourced
imutável** (append-only) para toda ação privilegiada e toda decisão de agente
(*decision provenance*). Nenhum serviço é considerado "pronto" sem telemetria e
auditoria (Definition of Done).

## Consequências
**Positivas:** depuração e auditoria completas; compliance nativa; proveniência de
decisões; SLOs mensuráveis.
**Negativas:** overhead de runtime (mitigado por sampling adaptativo); custo de
armazenamento de telemetria/auditoria.
**Neutras:** métricas, logs e eventos são padronizados por
`024-Observability`/`025-Audit` e reusados por todos os módulos.

## Riscos
| Risco | Prob. | Impacto | Mitigação |
|-------|-------|---------|-----------|
| Custo de armazenamento | Média | Médio | Retenção em camadas; sampling; sink frio (MinIO/Kafka). |
| Overhead de tracing | Média | Médio | Sampling adaptativo; spans essenciais. |
| Vazamento de dados sensíveis em logs | Média | Alto | Redação/PII scrubbing obrigatória (021/025). |

## Trade-offs
Ganhamos depurabilidade, auditabilidade e compliance; pagamos com overhead de
runtime e custo de armazenamento de telemetria.

## Referências
`../000-Vision/Vision.md` §6; `../024-Observability/`; `../025-Audit/`.
