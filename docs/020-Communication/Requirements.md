---
Documento: Requirements
Módulo: 020-Communication
Status: Draft
Versão: 0.1
Última atualização: 2026-07-22
Responsável (RACI-A): Arquiteto do Módulo 020-Communication
ADRs relacionados: ADR-0004, ADR-0200, ADR-0201, ADR-0202, ADR-0203, ADR-0204, ADR-0205, ADR-0206, ADR-0207, ADR-0208, ADR-0209
RFCs relacionados: RFC-0001, RFC-0200, RFC-0201
Depende de: FunctionalRequirements.md, NonFunctionalRequirements.md, UseCases.md, Testing.md
---

# 020-Communication — Índice de Requisitos e Rastreabilidade

Consolida os requisitos do módulo e estabelece a **matriz de rastreabilidade**
Requisito → Caso de Uso → Teste. Não introduz requisitos novos: os FR estão em
`./FunctionalRequirements.md` e os NFR em `./NonFunctionalRequirements.md`, ambos
derivados de `./_DESIGN_BRIEF.md` §7.

---

## 1. Stakeholders e Interesses

| Stakeholder | Interesse primário | Requisitos que o representam |
|-------------|--------------------|------------------------------|
| Engenheiro de módulo produtor | Publicar sem perder mensagem nem quebrar consumidores. | FR-002, FR-003, FR-005, FR-015, NFR-001, NFR-002 |
| Engenheiro de módulo consumidor | Não perder trabalho nem travar por mensagem ruim. | FR-006, FR-007, FR-008, FR-011, NFR-008, NFR-012 |
| Autor de agente | Colaborar com outros agentes sem infraestrutura própria. | FR-012, FR-013, FR-014, FR-020, NFR-015, NFR-016 |
| SRE / Operador de plataforma | Saber quem satura o barramento e recuperar rápido. | FR-010, FR-011, FR-016, NFR-004, NFR-005, NFR-009 |
| Arquiteto de segurança | Isolamento entre tenants e ausência de eventos forjados. | FR-004, FR-009, FR-017, FR-019, NFR-010 |
| Encarregado de dados (DPO) | Saber o que trafega e por quanto tempo. | FR-013 (trilha A2A), NFR-013 |
| Engenheiro de custo (`026`) | Atribuir custo de tráfego a quem o gera. | FR-010, NFR-004 |

---

## 2. Índice Consolidado

### 2.1 Requisitos Funcionais (20)

| Faixa | Tema | IDs |
|-------|------|-----|
| Namespace e validação | Registro de subjects, envelope, produtor declarado | FR-002, FR-003, FR-004 |
| Transporte | Pub/sub, request/reply, streams, ordem, dedupe | FR-001, FR-005, FR-006 |
| Entrega e recuperação | Reentrega, DLQ, replay | FR-007, FR-008 |
| Isolamento | Conta por tenant, export entre contas | FR-009, FR-019 |
| Contenção | Cotas, *slow consumer*, payload | FR-010, FR-011, FR-015 |
| Comunicação seletiva | Canais de grupo, fan-out | FR-012 |
| A2A | Sessões internas e externas, encerramento | FR-013, FR-014, FR-020 |
| Governança | Declaração idempotente, PEP, correlação | FR-016, FR-017, FR-018 |

### 2.2 Requisitos Não-Funcionais (16)

| Categoria | IDs |
|-----------|-----|
| Desempenho | NFR-001, NFR-002, NFR-003, NFR-004, NFR-008 |
| Disponibilidade e recuperação | NFR-005, NFR-006, NFR-009 |
| Escalabilidade | NFR-007, NFR-016 |
| Segurança | NFR-010 |
| Contenção e confiabilidade | NFR-011, NFR-012 |
| Observabilidade | NFR-013 |
| Manutenibilidade | NFR-014 |
| A2A | NFR-015 |

### 2.3 Casos de Uso (15)

`UC-001` a `UC-015`, especificados em `./UseCases.md`.

---

## 3. Matriz de Rastreabilidade — FR → UC → Teste

| FR | Caso(s) de uso | Caso(s) de teste (`./Testing.md`) | NFR associado |
|----|----------------|-----------------------------------|---------------|
| FR-001 | UC-004, UC-005, UC-015 | T-BUS-01, T-BUS-02, T-RR-01 | NFR-001, NFR-003 |
| FR-002 | UC-001, UC-004 | T-REG-01, T-REG-02 | — |
| FR-003 | UC-004 | T-ENV-01, T-ENV-02 | — |
| FR-004 | UC-004 | T-SEC-03 | NFR-010 |
| FR-005 | UC-004 | T-BUS-03 (dedupe) | NFR-002 |
| FR-006 | UC-005 | T-BUS-04 (ordem) | NFR-008 |
| FR-007 | UC-005, UC-006 | T-DLQ-01, T-DLQ-02 | NFR-012 |
| FR-008 | UC-007, UC-008 | T-DLQ-03, T-RPL-01 | NFR-009 |
| FR-009 | UC-009 | T-SEC-01 (penetração de subject) | NFR-010 |
| FR-010 | UC-014 | T-FLOW-01 | NFR-004, NFR-011 |
| FR-011 | UC-014 | T-FLOW-02 (contenção) | NFR-011 |
| FR-012 | UC-013 | T-GRP-01, T-GRP-02 | NFR-016 |
| FR-013 | UC-011 | T-A2A-01, T-A2A-02 | NFR-015 |
| FR-014 | UC-012 | T-A2A-03 (par externo revogado) | NFR-010 |
| FR-015 | UC-004 | T-ENV-03 | NFR-001 |
| FR-016 | UC-002, UC-003 | T-DECL-01, T-DECL-02 (migração sem perda) | NFR-014 |
| FR-017 | UC-002, UC-010, UC-011 | T-SEC-04 (default deny) | NFR-010 |
| FR-018 | UC-004, UC-005 | T-OBS-01 (trace ponta a ponta) | NFR-013 |
| FR-019 | UC-010 | T-SEC-02 | NFR-010 |
| FR-020 | UC-011, UC-012 | T-A2A-04 (ociosidade e limpeza) | NFR-015 |

---

## 4. Matriz NFR → Método de Verificação

| NFR | Método | Documento |
|-----|--------|-----------|
| NFR-001, NFR-002, NFR-003 | Medição de latência sob carga controlada | `./Benchmark.md` §4 |
| NFR-004 | Rampa de throughput até saturação | `./Benchmark.md` §5 |
| NFR-005 | Uptime probe + *error budget* | `./Monitoring.md` §4 |
| NFR-006 | Chaos test (kill de nó com quórum mantido) | `./FailureRecovery.md` §6 |
| NFR-007 | Teste de escala (10⁵ subjects, 10⁴ consumidores) | `./Scalability.md` §7 |
| NFR-008 | Histograma publish→deliver | `./Benchmark.md` §5 |
| NFR-009 | DR drill com snapshot JetStream | `./FailureRecovery.md` §5 |
| NFR-010 | Pentest de subject + auditoria de contas | `./Security.md` §6 |
| NFR-011 | Cenário de contenção com consumidor lento | `./Testing.md` §7 |
| NFR-012 | Razão DLQ/entregues em produção | `./Monitoring.md` §3 |
| NFR-013 | Amostragem de traces ponta a ponta | `./Logging.md` §7 |
| NFR-014 | Teste de replay com `Idempotency-Key` | `./Testing.md` §4 |
| NFR-015 | Carga de sessões A2A simultâneas | `./Benchmark.md` §7 |
| NFR-016 | Contagem publicações × entregas em difusão | `./Benchmark.md` §6 |

---

## 5. Requisitos Herdados (não redefinidos aqui)

| Origem | Requisito herdado | Aplicação no módulo |
|--------|-------------------|---------------------|
| RFC-0001 §5.1 | URN com ULID | `urn` das entidades de `comm` e identidade de agentes em sessões A2A |
| RFC-0001 §5.2 | Envelope CloudEvents | Validado no publish (`EnvelopeValidator`) |
| **RFC-0001 §5.3** | **Convenção de subjects** `aios.<tenant>.<dominio>.<entidade>.<acao>` | **Contrato central deste módulo**: o registro `comm.subject_registry` o aplica, não o redefine |
| RFC-0001 §5.4 | Envelope de erro | Domínios `BUS` e `A2A` em `./API.md` |
| RFC-0001 §5.5 | Idempotência e dedupe por `event.id` | `dedupe_window` do stream; mutações administrativas |
| RFC-0001 §5.6 | Cabeçalhos de correlação | `traceparent` em toda mensagem |
| RFC-0001 §5.7 | Versionamento e evolução aditiva | Depreciação de subject com coexistência ≥ 2 majors |
| RFC-0001 §5.8 | PEP/PDP com *default deny* e auditoria | FR-017, FR-019 |
| RFC-0001 §6–§7 | mTLS, isolamento por tenant, minimização | `./Security.md` |
| `../000-Vision/Vision.md` V-R10 | RTO ≤ 15 min, RPO ≤ 5 min | NFR-009 |

---

## 6. Glossário Local

Este módulo **não define termos próprios**. *NATS*, *JetStream*, *Backpressure*,
*A2A*, *Agent Group*, *DLQ*, *Bulkhead*, *Tenant* e *Control Plane* são os do
`../040-Glossary/Glossary.md`. Termos específicos do NATS (subject, stream, consumidor
durável, ack, conta, leaf node, supercluster) são usados com o significado padrão da
documentação oficial do NATS 2.x e estão contextualizados em `./Architecture.md` e
`./Events.md`.

---

## 7. Referências

- Requisitos funcionais: `./FunctionalRequirements.md`
- Requisitos não-funcionais: `./NonFunctionalRequirements.md`
- Casos de uso: `./UseCases.md` · Testes: `./Testing.md`
- Brief: `./_DESIGN_BRIEF.md` · Contratos: `../003-RFC/RFC-0001-Architecture-Baseline.md`
