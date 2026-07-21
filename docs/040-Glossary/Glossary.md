---
Documento: Glossary (Global)
Módulo: 040-Glossary
Status: Stable
Versão: 1.0
Última atualização: 2026-07-20
Responsável (RACI-A): Arquitetura-Chefe
ADRs relacionados: —
RFCs relacionados: RFC-0001
Depende de: 000-Vision, 001-Architecture
---

# AIOS — Glossário Canônico

> **Fonte única de verdade terminológica.** Todo documento do AIOS DEVE reutilizar
> estas definições sem redefini-las. Ao introduzir um termo novo, adicione-o aqui.
> Termos em inglês são mantidos quando consagrados na indústria.

## Convenção

- **Termo** *(sigla)* — definição. `→ módulo(s) onde é detalhado`.

---

## A

**A2A (Agent-to-Agent)** — Protocolo aberto de comunicação entre agentes, usado
para colaboração entre agentes do AIOS e com agentes externos. `→ 020`

**ABAC (Attribute-Based Access Control)** — Autorização baseada em atributos
(do sujeito, recurso, ação e ambiente). Complementa o RBAC. `→ 022`

**ACB (Agent Control Block)** — Estrutura que representa um agente gerenciado
(análogo ao PCB — *Process Control Block* — do SO clássico): identidade, estado,
cotas, prioridade, ponteiros de memória/contexto, política. `→ 006, 008`

**Admission Control** — Decisão do Scheduler de aceitar ou rejeitar uma tarefa/
agente com base em cotas, orçamento e capacidade disponível. `→ 009`

**Agent (Agente)** — Entidade autônoma gerenciada pelo AIOS que percebe, raciocina,
usa ferramentas e age para cumprir objetivos. Unidade fundamental, análoga ao
"processo". `→ 007, 008`

**Agent Group** — Conjunto lógico de agentes tratado como unidade para
comunicação seletiva, política e escalonamento; reduz custo de coordenação. `→ 020, 009`

**Agent Runtime** — Ambiente de execução isolado (Python, sandbox) onde o loop
cognitivo do agente roda. Plano de dados. `→ 007`

**AIOS (Artificial Intelligence Operating System)** — Sistema operacional para
agentes de IA; gerenciador de recursos cognitivos e computacionais. `→ 000`

**AOT (Ahead-of-Time compilation)** — Compilação antecipada usada no plano de
controle .NET para reduzir latência de start. `→ 028`

**AZ (Availability Zone)** — Zona de disponibilidade; domínio de falha usado no
modelo de HA. `→ 027`

---

## B

**Backpressure** — Mecanismo que limita a taxa de entrada quando o sistema está
saturado, propagando pressão do consumidor ao produtor. `→ 009, 020`

**Bulkhead** — Padrão de isolamento que particiona recursos para conter falhas
(um compartimento afundado não afunda o navio). `→ 001, 021`

**Benchmark** — Metodologia e conjunto de cargas para medir desempenho e
qualidade de forma reprodutível. `→ 035`

---

## C

**Capability** — Token/permissão que concede a um agente o direito de executar
uma syscall cognitiva ou usar uma ferramenta específica. Base do *default deny*. `→ 006, 021`

**Circuit Breaker** — Padrão que interrompe chamadas a um dependente falho para
permitir sua recuperação. `→ 017, 015`

**Cognitive Resource (Recurso Cognitivo)** — Recurso gerenciado pelo AIOS
específico de IA: memória, contexto, atenção, inferência, conhecimento. `→ 000, 010, 011`

**Cold Agent** — Agente suspenso/hibernado cujo estado está em armazenamento
durável (não em RAM), materializável sob demanda. Chave para escala a milhões. `→ 008, 001`

**Context Window (Janela de Contexto)** — Limite de tokens que um modelo processa
por chamada. Recurso escasso gerenciado pelo Context Manager. `→ 011`

**Context Compression Ratio** — Métrica: redução de tokens preservando desempenho
da tarefa. `→ 011, 024`

**Control Plane (Plano de Controle)** — Conjunto de serviços que decidem e
gerenciam (scheduling, política, registro). Distinto do plano de dados. `→ 001`

**Cost Optimizer** — Subsistema que otimiza o custo de inferência sob restrições
de qualidade/SLA/orçamento. `→ 026`

**CQRS (Command Query Responsibility Segregation)** — Separação de caminhos de
escrita e leitura. `→ 001, 010`

---

## D

**Data Plane (Plano de Dados)** — Onde os agentes executam (Agent Runtime).
Distinto do plano de controle. `→ 001, 007`

**Default Deny** — Postura de segurança em que nada é permitido salvo autorização
explícita. `→ 021, 022`

**DLQ (Dead-Letter Queue)** — Fila para mensagens que falharam repetidamente,
para inspeção/replay. `→ 020, 014`

**DR (Disaster Recovery)** — Recuperação de desastre; processos e metas (RTO/RPO)
para restaurar o sistema após falha grave. `→ 027`

**Driver** — Extensão que integra um recurso externo (modelo, ferramenta) ao AIOS
por contrato estável, análogo a driver de dispositivo. `→ 015, 017`

---

## E

**Embedding** — Vetor denso que representa semântica de um item; armazenado em
`pgvector`. `→ 010, 019`

**Episodic Memory (Memória Episódica)** — Memória de eventos/experiências
específicas com contexto temporal. `→ 010`

**Event Sourcing** — Padrão em que o estado é derivado de uma sequência imutável
de eventos; base da auditoria. `→ 001, 025`

**Exactly-once / At-least-once** — Semânticas de entrega de mensagem. O AIOS usa
at-least-once + idempotência para efeito exactly-once. `→ 020`

---

## F

**Fan-out** — Número de destinatários de uma mensagem/decisão; limitado para
evitar custo O(N²) de coordenação. `→ 020, 009`

**FMEA (Failure Mode and Effects Analysis)** — Análise sistemática de modos de
falha e efeitos. `→ */FailureRecovery.md`

**Forgetting (Esquecimento controlado)** — Política de remoção/decaimento de itens
de memória para evitar crescimento descontrolado. `→ 010`

---

## G

**Goal (Objetivo)** — Estado-alvo declarativo com critérios de sucesso e
restrições que dirige o planejamento. `→ 013`

**GraphRAG** — Recuperação aumentada por geração baseada em grafo de conhecimento
(Apache AGE), superando RAG puramente vetorial. `→ 019`

**gRPC** — Framework RPC de alto desempenho usado na comunicação interna. `→ 004`

---

## H

**HA (High Availability)** — Alta disponibilidade; capacidade de operar apesar de
falhas de componentes. `→ 027`

**Hot State (Estado quente)** — Estado acessado com alta frequência, mantido em
Redis para latência sub-ms. `→ 001, 010`

---

## I

**Idempotency (Idempotência)** — Propriedade de uma operação produzir o mesmo
efeito se aplicada uma ou várias vezes. Obrigatória em mutações. `→ 001, 004`

**Inference (Inferência)** — Execução de um modelo de IA para produzir saída; o
recurso de maior custo. `→ 017, 026`

**IdP (Identity Provider)** — Provedor de identidade federada (OIDC). `→ 021`

---

## J

**JetStream** — Camada de streaming durável do NATS (persistência, replay,
consumidores). `→ 020`

---

## K

**Kernel (Cognitivo)** — Núcleo do AIOS: expõe syscalls cognitivas, gerencia ACB,
cotas e ciclo de vida. `→ 006`

**Knowledge Graph (Grafo de Conhecimento)** — Representação em nós/arestas do
conhecimento consolidado; base do GraphRAG. `→ 018, 019`

**Knowledge Reuse Rate** — Métrica: frequência de reutilização de conhecimento já
consolidado. `→ 023, 024`

---

## L

**LGPD / GDPR** — Leis de proteção de dados (Brasil / UE); requisitos de
privacidade, consentimento e direitos do titular. `→ 021, 025`

**Lifecycle (Ciclo de Vida)** — Estados e transições de um agente
(spawn→ready→running→suspended→terminated). `→ 008`

**LLM Router / Model Router** — Subsistema que escolhe o modelo por
custo/qualidade/latência/disponibilidade. `→ 017`

**Long-Term Memory (Memória de Longo Prazo)** — Memória persistente consolidada.
`→ 010`

---

## M

**MCP (Model Context Protocol)** — Protocolo aberto para expor ferramentas e
contexto a modelos/agentes. `→ 015`

**Memory Recall Rate** — Métrica: fração de informação relevante recuperada
quando necessária. `→ 010, 024`

**mTLS (mutual TLS)** — TLS com autenticação de ambas as partes; usado entre
serviços internos. `→ 021`

**Multi-tenancy** — Capacidade de servir múltiplos tenants isolados na mesma
plataforma. `→ 001, 021`

---

## N

**NATS** — Barramento de mensageria de baixa latência do AIOS. `→ 020`

**NFR (Non-Functional Requirement)** — Requisito não-funcional (qualidade). `→ */NonFunctionalRequirements.md`

---

## O

**Outbox** — Padrão transacional que garante atomicidade entre escrita no banco e
publicação de evento. `→ 001, 020`

**OpenTelemetry (OTel)** — Padrão de telemetria (traces, métricas, logs). `→ 024`

**OIDC (OpenID Connect)** — Camada de identidade sobre OAuth2. `→ 021`

---

## P

**PEP / PDP (Policy Enforcement/Decision Point)** — PEP intercepta e aplica;
PDP decide. Base da governança por política. `→ 022`

**pgvector** — Extensão PostgreSQL para busca vetorial (ANN). `→ 005, 010`

**Placement** — Decisão de onde (qual runtime/shard/nó) executar um agente/tarefa.
`→ 009, 027`

**Planning Engine** — Subsistema que decompõe objetivos em planos e replaneja. `→ 012`

**Policy Engine** — Subsistema que avalia políticas RBAC/ABAC (PDP). `→ 022`

**Preemption (Preempção)** — Suspensão de um agente de menor prioridade para
liberar recurso. `→ 009`

**Procedural Memory (Memória Procedural)** — Memória de "como fazer"
(habilidades/rotinas aprendidas). `→ 010`

**Provenance (Proveniência)** — Registro rastreável de como uma decisão/resultado
foi produzido. `→ 025`

---

## Q

**Quota (Cota)** — Limite de recurso (tokens, CPU, memória, custo, ferramentas)
por agente/tenant. `→ 006, 026`

---

## R

**RAG (Retrieval-Augmented Generation)** — Geração aumentada por recuperação de
contexto. GraphRAG é sua evolução baseada em grafo. `→ 018, 019`

**RBAC (Role-Based Access Control)** — Autorização por papéis. `→ 022`

**ReAct** — Padrão de raciocínio "Reason + Act" (pensar→agir→observar→repetir). `→ 007, 012`

**Reflexion** — Técnica de autoavaliação para melhorar desempenho por reflexão
sobre falhas. `→ 023`

**Runtime Supervisor** — Serviço do control plane que cria/monitora/encerra Agent
Runtimes e aplica cotas. `→ 007, 001`

**RTO / RPO** — *Recovery Time/Point Objective*: metas de tempo/perda de dados na
recuperação. `→ 027`

**Row-Level Security (RLS)** — Mecanismo do PostgreSQL para isolamento por linha
(por `tenant_id`). `→ 005, 021`

---

## S

**Saga** — Padrão de transação distribuída por passos compensáveis. `→ 014`

**Sandbox** — Ambiente isolado e restrito onde o agente executa código/ferramentas
com segurança. `→ 007, 021`

**Scheduler (Cognitivo)** — Subsistema que decide qual agente executa, quando, com
qual prioridade/custo. `→ 009`

**Semantic Cache (Cache Semântico)** — Cache indexado por similaridade semântica
que evita reinferências. `→ 011`

**Semantic Memory (Memória Semântica)** — Memória de fatos/conceitos gerais,
independente de episódio. `→ 010`

**Shard** — Partição horizontal determinística do estado/carga
(`hash(tenant,agent) mod N`). `→ 001, 027`

**Short-Term Memory (Memória de Curto Prazo)** — Memória recente de sessão, acima
da working memory e abaixo da long-term. `→ 010`

**SLI/SLO/SLA** — Indicador/Objetivo/Acordo de nível de serviço. `→ 024, 027`

**SPOF (Single Point of Failure)** — Ponto único de falha; eliminado por design. `→ 027`

**STRIDE** — Framework de *threat modeling* (Spoofing, Tampering, Repudiation,
Information disclosure, Denial of service, Elevation of privilege). `→ 021`

**Syscall Cognitiva** — Chamada de sistema exposta pelo Kernel a agentes
(ex.: `remember`, `recall`, `plan`, `invoke_tool`). `→ 006`

---

## T

**Task (Tarefa)** — Unidade de trabalho escalonável atribuída a um agente. `→ 009, 014`

**Task Completion Rate** — Métrica: % de tarefas concluídas com sucesso. `→ 024, 035`

**Tenant** — Cliente/organização isolada; unidade de política, cobrança e
isolamento. `→ 001, 021`

**Tool (Ferramenta)** — Capacidade externa que o agente invoca (API, DB, ação),
integrada como driver via MCP. `→ 015`

**Tool Manager** — Subsistema "driver manager" de ferramentas: registro,
permissões, métricas, versão. `→ 015`

**Trace / Span** — Unidade de rastreamento distribuído (OpenTelemetry). `→ 024`

---

## W

**Working Memory (Memória de Trabalho)** — Memória mais volátil e imediata do
raciocínio corrente do agente. `→ 010`

**Workflow Engine** — Subsistema de fluxos determinísticos e sagas. `→ 014`

---

## Y

**YARP (Yet Another Reverse Proxy)** — Reverse proxy .NET usado como API Gateway. `→ 001, 028`

---

## Métricas Cognitivas (referência rápida)

| Métrica | Definição | Módulo |
|---------|-----------|--------|
| Memory Recall Rate | Info relevante recuperada / necessária | 010, 024 |
| Context Compression Ratio | Redução de tokens preservando desempenho | 011 |
| Task Completion Rate | Tarefas concluídas / iniciadas | 024, 035 |
| Agent Collaboration Efficiency | Trabalho útil / interação entre agentes | 020 |
| Reasoning Cost per Successful Task | Custo total / tarefa concluída | 026 |
| Knowledge Reuse Rate | Reusos de conhecimento / consultas | 023 |
| Recovery Time After Failure | Tempo até retomar execução pós-falha | 027 |

---

*Termos ausentes devem ser adicionados via PR referenciando o módulo de origem.
Ver `../033-DeveloperGuide/DocumentationCI.md`.*
