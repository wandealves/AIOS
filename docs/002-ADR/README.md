---
Documento: ADR Index
Módulo: 002-ADR
Status: Stable
Versão: 1.0
Última atualização: 2026-07-20
Responsável (RACI-A): Arquitetura-Chefe
---

# AIOS — Architecture Decision Records (ADR)

Registro imutável de decisões arquiteturais. Cada ADR captura **uma** decisão
significativa, seu contexto e consequências. ADRs não são editadas após
`Accepted`; são **substituídas** (`Superseded by ADR-XXXX`).

## Como escrever uma ADR

Use `TEMPLATE.md`. Toda ADR contém, obrigatoriamente:
**Contexto · Problema · Alternativas · Análise · Escolha · Consequências · Riscos · Trade-offs.**

## Estados

`Proposed` → `Accepted` → (`Deprecated` | `Superseded`).

## Índice

| ADR | Título | Status | Decisão (1 linha) |
|-----|--------|--------|-------------------|
| [0001](ADR-0001-Sistema-Operacional-nao-Framework.md) | AIOS é um SO, não um framework | Accepted | Construir plano de controle próprio, não usar biblioteca de orquestração como núcleo. |
| [0002](ADR-0002-Microservicos-Control-Data-Plane.md) | Microserviços + split control/data plane | Accepted | Separar plano de controle (.NET) do plano de dados (runtime) em microserviços. |
| [0003](ADR-0003-DotNet-Control-Python-Runtime.md) | .NET 10 no controle, Python no runtime | Accepted | Control plane em .NET 10; Agent Runtime em Python. |
| [0004](ADR-0004-NATS-como-Barramento.md) | NATS como barramento primário | Accepted | NATS+JetStream em vez de Kafka como bus principal. |
| [0005](ADR-0005-PostgreSQL-pgvector-AGE.md) | PostgreSQL + pgvector + Apache AGE | Accepted | Unificar relacional, vetorial e grafo em PostgreSQL. |
| [0006](ADR-0006-Redis-Estado-Quente.md) | Redis para estado quente e locks | Accepted | Redis para cache, rate-limit e locks distribuídos. |
| [0007](ADR-0007-Memoria-Hierarquica.md) | Memória hierárquica de 7 camadas | Accepted | Working→Short→Long→Semantic→Procedural→Episodic→Graph. |
| [0008](ADR-0008-Governanca-por-Politica-Default-Deny.md) | Governança por política, default deny | Accepted | PEP/PDP central; nada permitido sem autorização explícita. |
| [0009](ADR-0009-Model-Router-Multiobjetivo.md) | Model Router multiobjetivo | Accepted | Roteamento por custo×qualidade×latência sob restrição. |
| [0010](ADR-0010-Observabilidade-Auditoria-por-Construcao.md) | Observabilidade e auditoria por construção | Accepted | OTel + trilha imutável event-sourced em todo caminho. |

## Numeração

ADRs são numeradas sequencialmente (`ADR-NNNN`), nunca reutilizadas. Fan-out de
módulos adicionará ADRs específicas (ex.: ADR-0011+ para decisões de memória,
scheduler, etc.), sempre linkadas do `ADR.md` do módulo correspondente.
