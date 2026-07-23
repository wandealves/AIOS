---
Documento: Vision
Módulo: 020-Communication
Status: Draft
Versão: 0.1
Última atualização: 2026-07-22
Responsável (RACI-A): Arquiteto do Módulo 020-Communication
ADRs relacionados: ADR-0001, ADR-0002, ADR-0004, ADR-0008, ADR-0010 (globais); ADR-0200, ADR-0201, ADR-0206, ADR-0207 (deste módulo, a propor)
RFCs relacionados: RFC-0001, RFC-0200, RFC-0201
Depende de: 000-Vision, 001-Architecture, 021-Security, 022-Policy, 025-Audit, 027-Cluster
---

# 020-Communication — Visão do Módulo

> "O barramento está para o AIOS assim como *sockets*, *pipes* e sinais estão para um
> sistema operacional: nenhum processo escolhe como sua mensagem é roteada, isolada ou
> reentregue — ele apenas envia, e o sistema garante o resto." — analogia condutora
> deste módulo (ver `../040-Glossary/Glossary.md`, termos **NATS**, **JetStream**,
> **Backpressure**, **A2A**, **Agent Group**).

---

## 1. Propósito do Módulo

O **020-Communication** é o **subsistema de IPC** do AIOS: o barramento leste-oeste
por onde todo módulo do plano de controle e todo agente conversa com os demais. Ele
corresponde à camada **L1.5 BARRAMENTO** do Modelo de Camadas de
`../001-Architecture/Architecture.md` §6 e é implementado sobre **NATS 2.x +
JetStream** (decisão fundadora **ADR-0004**).

A tese do módulo, estabelecida no `_DESIGN_BRIEF.md` §0, cabe em uma frase:

> *o módulo produtor decide **o que** dizer e a quem interessa; o 020 decide **como** a
> mensagem trafega, é isolada, entregue, reentregue, reproduzida e contida.*

Disso decorrem as quatro entregas do módulo:

1. **Namespace governado** — o registro `comm.subject_registry` define quais subjects
   existem, quem pode publicar em cada um e qual `dataschema` os descreve. Publicar
   fora do registro é erro, não improviso.
2. **Garantias de entrega** — at-least-once com dedupe por `event.id`, ordem por
   stream, reentrega com backoff, DLQ inspecionável e replay controlado.
3. **Isolamento multi-tenant** — uma **conta NATS por tenant**: a fronteira
   `aios.<tenant>.…` é de segurança, não de organização.
4. **Custo de coordenação sub-linear** — comunicação seletiva por *Agent Group*,
   limites de fan-out e sessões A2A explícitas, para que 10⁶ agentes não produzam
   tráfego quadrático.

---

## 2. Problema que o Módulo Resolve

A Visão Global (`../000-Vision/Vision.md`) identifica limitações estruturais dos
sistemas de agentes atuais. O 020 responde a quatro delas na camada de transporte:

| Limitação da Visão | Como o barramento endereça |
|--------------------|-----------------------------|
| **Coordenação que não escala** | Frameworks de agentes tendem a conectar todos com todos: `N` agentes produzem `N²` caminhos e o sistema morre por volume de coordenação muito antes de morrer por computação. Aqui, difusão é **hierárquica e seletiva** — uma publicação chega a mil membros de um grupo ao custo de uma publicação (NFR-016). |
| **Acoplamento síncrono** | Chamadas diretas entre serviços fazem a falha de um derrubar a cadeia inteira. O barramento desacopla no tempo (streams duráveis) e no espaço (o produtor não conhece o consumidor). |
| **Perda silenciosa de trabalho** | Sem reentrega e DLQ, uma mensagem que falha simplesmente desaparece. Aqui ela é reentregue com backoff e, se persistir, vai para quarentena **inspecionável** — nunca some. |
| **Governança fraca (P8)** | Um barramento aberto permite que qualquer serviço publique qualquer coisa. Aqui, cada subject tem produtor declarado (`AIOS-BUS-0010`) e cada envelope é validado no publish — falsificar um evento de domínio alheio é estruturalmente impossível. |

---

## 3. Escopo

### 3.1 Dentro do escopo (in scope)

- Operação do cluster **NATS + JetStream**: pub/sub, request/reply, streams duráveis.
- Registro e governança do **namespace de subjects** (conforme RFC-0001 §5.3).
- Ciclo de vida de **streams** e **consumidores duráveis** (retenção, ack, backoff).
- **Validação do envelope CloudEvents** no momento da publicação.
- **Contas NATS por tenant**, *exports/imports* explícitos, revogação.
- **Backpressure e cotas** de tráfego por tenant e por agente; contenção de *slow consumers*.
- **Comunicação seletiva**: canais de *Agent Group*, escopo e limite de fan-out.
- **A2A**: sessões entre agentes internos e externos, com FSM, negociação e trilha.
- **DLQ** e **replay** (por tempo, sequência ou entrada de quarentena).
- Conectividade entre clusters (leaf nodes, gateways), conforme decisão do `027`.

### 3.2 Fora do escopo (out of scope)

Lista normativa completa em `_DESIGN_BRIEF.md` §1.3. Os pontos que mais geram confusão:

- **A semântica dos eventos** — o significado de `agent.lifecycle.spawned` pertence ao
  `006-Kernel`, não ao barramento. O 020 transporta e valida a **forma**.
- **Os contratos centrais** — envelope, subjects, idempotência e correlação são da
  RFC-0001. O 020 os **aplica**, não os redefine.
- **Ser fonte da verdade** — o barramento é transporte, não banco. Estado autoritativo
  vive em `../005-Database/`.
- **O padrão Outbox dentro dos produtores** — o 020 define o contrato de publicação; a
  tabela e o relay pertencem ao módulo produtor.
- **Blobs grandes** — payloads acima do limite são rejeitados (`AIOS-BUS-0006`);
  envie referência a objeto.
- **Tráfego norte-sul** (REST/gRPC externo) é do `../004-API/`; **MCP** de ferramentas
  é do `../007-Agent-Runtime/` e `../015-Tool-Manager/`.
- **Topologia geográfica e failover de cluster** são do `../027-Cluster/`.

### 3.3 Fronteira em diagrama

```
   produtores (006…026) ──publish──▶  020-Communication  ──opera──▶  NATS + JetStream
   consumidores         ◀─deliver──   (registro, contas,             contas por tenant
   agentes (A2A)        ◀─sessão───    cotas, DLQ, A2A)              streams R=3
```

---

## 4. Personas

| Persona | Necessidade | O que o módulo entrega |
|---------|-------------|------------------------|
| **Engenheiro de módulo produtor** | Publicar um evento sem se preocupar com entrega. | Registro de subject, validação no publish e garantia at-least-once com dedupe. |
| **Engenheiro de módulo consumidor** | Não perder mensagem nem travar por uma mensagem ruim. | Consumidor durável com ack explícito, backoff e DLQ inspecionável. |
| **Autor de agente** | Falar com outro agente sem abrir socket próprio. | Sessões A2A governadas e canais de *Agent Group*. |
| **SRE / Operador de plataforma** | Saber quem está inundando o barramento e por quê. | Cotas por tenant/agente, métricas `aios_bus_*`, eventos de saturação e DLQ. |
| **Arquiteto de segurança** | Garantir que um tenant não veja tráfego de outro. | Conta NATS por tenant; *export* entre contas só por declaração autorizada e auditada. |
| **Encarregado de dados (DPO)** | Saber o que trafega e por quanto tempo. | Retenção explícita por stream, minimização obrigatória de payload, trilha de sessões A2A. |

---

## 5. Princípios de Design

| # | Princípio | Consequência prática |
|---|-----------|----------------------|
| P-01 | **Subject não registrado não existe.** | Publicação fora do registro falha com `AIOS-BUS-0004`; o catálogo do barramento é sempre verdadeiro. |
| P-02 | **Um subject, um produtor.** | Só o `producer_module` declarado publica; forjar evento alheio é estruturalmente impedido (`AIOS-BUS-0010`). |
| P-03 | **A conta é a fronteira, não o prefixo.** | Isolamento por conta NATS, não por convenção de nome — um erro de curinga não vira vazamento. |
| P-04 | **Valide na entrada.** | Envelope conferido no publish: uma mensagem malformada nunca entra no stream e contamina consumidores. |
| P-05 | **Nada desaparece em silêncio.** | Reentrega com backoff e, esgotadas as tentativas, DLQ inspecionável. Replay é **explícito** por default. |
| P-06 | **A pressão sobe até o produtor.** | Cotas e `max_ack_pending` fazem o produtor recuar (`AIOS-BUS-0007`) em vez de o broker inchar. |
| P-07 | **Fan-out é do broker, não do agente.** | Uma difusão para 1.000 membros custa 1 publicação (NFR-016). |
| P-08 | **Transporte não é arquivo.** | Streams têm `max_age` explícito; retenção longa pertence a `../025-Audit/` e `../005-Database/`. |
| P-09 | **Falhar fechado no controle, aberto no tráfego.** | Com o PDP indisponível, novas sessões e operações administrativas são negadas — mas o tráfego já autorizado continua fluindo. |

O princípio P-09 merece nota: um barramento que para de entregar mensagens porque o
serviço de política ficou indisponível transforma uma falha localizada em queda total.
A escolha é negar **o que é novo** e preservar **o que já foi autorizado**.

---

## 6. Não-Objetivos Explícitos

1. **Não é um Kafka.** Não há intenção de reter meses de histórico para
   reprocessamento analítico; a retenção default é de 7 dias e o alvo é **latência**,
   não arquivamento (ADR-0004).
2. **Não é um banco de dados.** Consultar estado pelo barramento (replay para
   reconstruir "o valor atual") é antipadrão; use `../005-Database/`.
3. **Não é um transporte de arquivos.** Payload é limitado a 1 MiB por default; blobs
   vão para MinIO e trafega-se a referência.
4. **Não implementa lógica de domínio.** Sem transformação de payload, sem
   roteamento por conteúdo de negócio, sem enriquecimento.
5. **Não decide topologia multi-região.** Configura leaf nodes e gateways conforme o
   que o `../027-Cluster/` decidir.
6. **Não é a trilha de auditoria.** Entrega os eventos; a prova imutável é do
   `../025-Audit/`.

---

## 7. Relação com a Visão Global

O `../000-Vision/Vision.md` define o AIOS como um sistema operacional para agentes,
com metas de escala (10⁶ agentes), governança (nada privilegiado sem autorização) e
recuperabilidade (**RTO ≤ 15 min, RPO ≤ 5 min**). O 020 é onde a **primeira** dessas
metas encontra seu obstáculo mais duro:

- **Escala** → comunicação seletiva, hierarquia de subjects, conta por tenant e
  limites de fan-out (`Scalability.md`).
- **Governança** → produtor declarado por subject, PEP em toda operação administrativa
  e em toda sessão A2A (`Security.md`).
- **Recuperabilidade** → JetStream R=3, DLQ, replay e snapshot (`FailureRecovery.md`).

Nenhum contrato central é redefinido aqui: URN, envelope CloudEvents, **convenção de
subjects**, envelope de erro, idempotência e correlação vêm de
`../003-RFC/RFC-0001-Architecture-Baseline.md`; o vocabulário vem de
`../040-Glossary/Glossary.md`.

---

## 8. Critérios de Sucesso do Módulo

| # | Critério | Verificação |
|---|----------|-------------|
| S-01 | Zero vazamento entre contas de tenant em teste de penetração de subject. | NFR-010, `Testing.md` §8 |
| S-02 | Zero perda de mensagem persistida sob falha de 1 nó. | NFR-006, `FailureRecovery.md` |
| S-03 | Difusão para grupo de 1.000 membros ao custo de 1 publicação. | NFR-016, `Benchmark.md` §6 |
| S-04 | Taxa de DLQ ≤ 0,01% em regime normal, com 100% das entradas inspecionáveis. | NFR-012, `Monitoring.md` |
| S-05 | p99 de publish core ≤ 2 ms e de entrega ≤ 20 ms com 10⁵ subjects ativos. | NFR-001, NFR-008, `Benchmark.md` |
| S-06 | 100% das mensagens com `traceparent` propagado ponta a ponta. | NFR-013, `Logging.md` |

---

## 9. Referências

- Visão global: `../000-Vision/Vision.md`
- Arquitetura global: `../001-Architecture/Architecture.md`
- Contratos centrais: `../003-RFC/RFC-0001-Architecture-Baseline.md`
- Glossário: `../040-Glossary/Glossary.md`
- Fonte única de verdade deste módulo: `./_DESIGN_BRIEF.md`
- Arquitetura do módulo: `./Architecture.md` · Eventos: `./Events.md`
