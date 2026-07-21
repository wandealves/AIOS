# AIOS — Documentação Técnica

**AIOS · Artificial Intelligence Operating System** — Um sistema operacional para
agentes de IA: gerenciador de recursos cognitivos e computacionais, do agente
único à frota de milhões de agentes distribuídos.

Este repositório é a **documentação oficial** do AIOS, escrita no padrão de
sistemas operacionais/plataformas de infraestrutura (Linux, Kubernetes,
PostgreSQL, Envoy). O objetivo é permitir que engenheiros implementem o sistema
sem tomar decisões arquiteturais não documentadas.

> **Estado atual: Fundação canônica concluída.** A base terminológica e
> arquitetural (`000`, `001`, `040`, `002`, `003`) está escrita e estável. Os
> módulos `004`–`039` estão como **esqueleto + índice** e serão preenchidos no
> fan-out, cada documento conforme `_templates/MODULE_TEMPLATE.md` e em
> conformidade com `003-RFC/RFC-0001`.

> 📋 **Plano de desenvolvimento e progresso:** [`PLANO.md`](PLANO.md) — rastreador vivo
> com o que já foi finalizado e o que falta, dividido em passos (ondas por dependência).

---

## Como navegar

1. **Comece pela Visão** → [`000-Vision/Vision.md`](000-Vision/Vision.md)
2. **Entenda a arquitetura** → [`001-Architecture/Architecture.md`](001-Architecture/Architecture.md)
3. **Consulte o vocabulário** → [`040-Glossary/Glossary.md`](040-Glossary/Glossary.md)
4. **Veja as decisões** → [`002-ADR/README.md`](002-ADR/README.md)
5. **Leia os contratos centrais** → [`003-RFC/RFC-0001-Architecture-Baseline.md`](003-RFC/RFC-0001-Architecture-Baseline.md)
6. **Padrão de documentação** → [`_templates/MODULE_TEMPLATE.md`](_templates/MODULE_TEMPLATE.md)

---

## Índice de Módulos

| # | Módulo | Foco | Status |
|---|--------|------|--------|
| 000 | [Vision](000-Vision/) | Visão global, problema, princípios | ✅ Stable |
| 001 | [Architecture](001-Architecture/) | Arquitetura de referência (C4/UML) | ✅ Stable |
| 002 | [ADR](002-ADR/) | Decisões arquiteturais (10 seed) | ✅ Stable |
| 003 | [RFC](003-RFC/) | Propostas técnicas (RFC-0001) | ✅ Stable |
| 004 | [API](004-API/) | REST/gRPC/eventos/versionamento | 🟨 Skeleton |
| 005 | [Database](005-Database/) | Modelo físico PG+pgvector+AGE | 🟨 Skeleton |
| 006 | [Kernel](006-Kernel/) | Núcleo cognitivo, syscalls, ACB | 🟨 Skeleton |
| 007 | [Agent-Runtime](007-Agent-Runtime/) | Plano de dados (Python), sandbox | 🟨 Skeleton |
| 008 | [Agent-Lifecycle](008-Agent-Lifecycle/) | Ciclo de vida do agente | 🟨 Skeleton |
| 009 | [Scheduler](009-Scheduler/) | Scheduler cognitivo | 🟨 Skeleton |
| 010 | [Memory](010-Memory/) | Memória hierárquica (7 camadas) | 🟨 Skeleton |
| 011 | [Context](011-Context/) | Janela de contexto, cache semântico | 🟨 Skeleton |
| 012 | [Planning](012-Planning/) | Motor de planejamento | 🟨 Skeleton |
| 013 | [Goals](013-Goals/) | Gerência de objetivos | 🟨 Skeleton |
| 014 | [Workflow](014-Workflow/) | Motor de workflow / sagas | 🟨 Skeleton |
| 015 | [Tool-Manager](015-Tool-Manager/) | Driver manager de ferramentas (MCP) | 🟨 Skeleton |
| 016 | [Plugin-System](016-Plugin-System/) | Extensões carregáveis | 🟨 Skeleton |
| 017 | [Model-Router](017-Model-Router/) | Roteamento multiobjetivo de LLMs | 🟨 Skeleton |
| 018 | [Knowledge](018-Knowledge/) | Gerência de conhecimento | 🟨 Skeleton |
| 019 | [GraphRAG](019-GraphRAG/) | Recuperação aumentada por grafo | 🟨 Skeleton |
| 020 | [Communication](020-Communication/) | Barramento NATS, A2A | 🟨 Skeleton |
| 021 | [Security](021-Security/) | AuthN/Z, mTLS, sandbox, LGPD/GDPR | 🟨 Skeleton |
| 022 | [Policy](022-Policy/) | Policy Engine (PDP), RBAC/ABAC | 🟨 Skeleton |
| 023 | [Learning](023-Learning/) | Aprendizado contínuo | 🟨 Skeleton |
| 024 | [Observability](024-Observability/) | OTel, métricas, traces, logs | 🟨 Skeleton |
| 025 | [Audit](025-Audit/) | Auditoria imutável | 🟨 Skeleton |
| 026 | [Cost-Optimizer](026-Cost-Optimizer/) | Otimização de custo de inferência | 🟨 Skeleton |
| 027 | [Cluster](027-Cluster/) | HA, DR, sharding, failover | 🟨 Skeleton |
| 028 | [Deployment](028-Deployment/) | Docker/Compose, topologia, YARP | 🟨 Skeleton |
| 029 | [Operations](029-Operations/) | Runbooks, on-call, upgrades | 🟨 Skeleton |
| 030 | [CLI](030-CLI/) | Interface de linha de comando | 🟨 Skeleton |
| 031 | [SDK](031-SDK/) | SDKs .NET/Python | 🟨 Skeleton |
| 032 | [WebConsole](032-WebConsole/) | Console operacional (React) | 🟨 Skeleton |
| 033 | [DeveloperGuide](033-DeveloperGuide/) | Guia do desenvolvedor | 🟨 Skeleton |
| 034 | [AdministratorGuide](034-AdministratorGuide/) | Guia do administrador | 🟨 Skeleton |
| 035 | [Benchmark](035-Benchmark/) | Metodologia e resultados | 🟨 Skeleton |
| 036 | [Experiments](036-Experiments/) | Plano experimental | 🟨 Skeleton |
| 037 | [Papers](037-Papers/) | Fundamentação científica | 🟨 Skeleton |
| 038 | [Patents](038-Patents/) | Propriedade intelectual | 🟨 Skeleton |
| 039 | [Roadmap](039-Roadmap/) | Roadmap por versões | 🟨 Skeleton |
| 040 | [Glossary](040-Glossary/) | Glossário canônico | ✅ Stable |

Legenda: ✅ Stable · 🟨 Skeleton (índice + 26 docs planejados) · ⬜ Planned (doc individual).

---

## Padrões de documentação

- **Idioma:** Português (pt-BR); termos técnicos consagrados em inglês.
- **Palavras normativas:** RFC 2119 / RFC 8174 (DEVE, NÃO DEVE, DEVERIA, PODE).
- **Diagramas:** ASCII (C4, UML, sequência, estado, componente, implantação).
- **Cada módulo:** 26 documentos obrigatórios (ver `_templates/MODULE_TEMPLATE.md`).
- **Consistência:** nenhum termo é redefinido; reutiliza-se o glossário e a RFC-0001.

## Stack tecnológica de referência

Control plane **.NET 10** · Agent runtime **Python** · Frontend **React** ·
Mensageria **NATS/JetStream** · Dados **PostgreSQL + pgvector + Apache AGE** ·
Estado quente **Redis** · Objetos **MinIO** · Observabilidade
**OpenTelemetry/Prometheus/Grafana/Serilog/Seq** · Gateway **YARP** · Contratos
**REST/gRPC/MCP/A2A** · Empacotamento **Docker/Compose**.

## Contribuição

Ver `033-DeveloperGuide/` (a preencher no fan-out). Novos termos → `040-Glossary`.
Novas decisões → `002-ADR`. Novas especificações de protocolo/contrato → `003-RFC`.
