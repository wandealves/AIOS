---
Documento: UseCases
Módulo: 020-Communication
Status: Draft
Versão: 0.1
Última atualização: 2026-07-22
Responsável (RACI-A): Arquiteto do Módulo 020-Communication
ADRs relacionados: ADR-0200, ADR-0201, ADR-0202, ADR-0203, ADR-0205, ADR-0206, ADR-0207
RFCs relacionados: RFC-0001, RFC-0200, RFC-0201
Depende de: FunctionalRequirements.md, API.md, StateMachine.md, 021-Security, 022-Policy, 025-Audit
---

# 020-Communication — Casos de Uso

Formato: ator / pré-condições / fluxo principal / fluxos alternativos / exceções /
pós-condições. Códigos de erro conforme `./API.md` §5.

---

## UC-001 — Registrar um subject

- **Ator primário:** Engenheiro de módulo produtor.
- **Requisitos:** FR-002, FR-017.
- **Pré-condições:** módulo registrado; `dataschema` publicado em `../004-API/`.

**Fluxo principal**
1. `POST /v1/comm/subjects` com `domain`, `entity`, `action`, `producer_module`,
   `dataschema`, `stream_ref` e `traffic_class`.
2. O `SubjectRegistry` valida a forma contra RFC-0001 §5.3 e verifica que o
   `dataschema` existe no registro do `004-API`.
3. Verifica que o trio (`domain`,`entity`,`action`) é inédito e que o stream referido
   captura o padrão de subject correspondente.
4. Persiste e emite `aios._platform.comm.subject.registered`.

**Fluxos alternativos**
- **A1** — registro repetido com a mesma `Idempotency-Key`: retorna o registro
  existente, sem duplicar.

**Exceções**
- **E1** — forma inválida (domínio fora da RFC-0001 §5.3, ou subject por instância em
  vez de por tipo) → `AIOS-BUS-0004`.
- **E2** — `dataschema` desconhecido → `AIOS-BUS-0015`.
- **E3** — capability ausente → `AIOS-BUS-0002`.

**Pós-condições:** o subject passa a ser publicável pelo produtor declarado — e por
mais ninguém.

---

## UC-002 — Declarar um stream

- **Ator primário:** Engenheiro de módulo dono do fluxo.
- **Requisitos:** FR-016, FR-017.

**Fluxo principal**
1. `PUT /v1/comm/streams/{name}` com `subjects`, `retention`, `max_age`, `max_bytes`,
   `replicas`, `discard` e `dedupe_window`.
2. PDP autoriza `bus:stream:write`.
3. O `StreamManager` grava a declaração (`declared`) e materializa no JetStream
   (`active`), emitindo `stream.created`.

**Fluxos alternativos**
- **A1** — declaração idêntica repetida: operação nula (idempotente).
- **A2** — mudança **compatível** (aumento de `max_age`/`max_bytes`): aplicada em lugar.
- **A3** — mudança **incompatível** (número de réplicas, chave de particionamento
  lógico): o `StreamManager` cria o novo stream, espelha, drena o antigo e aposenta —
  sem perda (ADR-0203).
- **A4** — aposentadoria: `:drain` leva a `draining` e, após o backlog zerar, a
  `retired`, com evento `stream.retired`.

**Exceções**
- **E1** — `replicas` fora de 1..5 → `AIOS-BUS-0004`.
- **E2** — sobreposição de padrões de subject com outro stream → `AIOS-BUS-0004`
  (uma mensagem capturada por dois streams duplica entrega e confunde retenção).

**Pós-condições:** stream ativo e coerente com a declaração versionada.

---

## UC-003 — Declarar um consumidor durável

- **Ator primário:** Engenheiro de módulo consumidor.
- **Requisitos:** FR-016.

**Fluxo principal**
1. `PUT /v1/comm/streams/{name}/consumers/{durable}` com `ack_policy` (`explicit`),
   `ack_wait`, `max_deliver`, `backoff`, `max_ack_pending`, `filter_subject` e
   `dlq_subject`.
2. O `ConsumerManager` valida coerência (`max_deliver` ≥ 1; `backoff` crescente e com
   tamanho compatível com `max_deliver`) e materializa no JetStream.

**Exceções**
- **E1** — `ack_policy = none` em stream de tráfego de controle → recusado: perder
  confirmação em fluxo de controle é perda silenciosa disfarçada de desempenho.
- **E2** — `max_ack_pending` acima do teto global → `AIOS-BUS-0004`.

**Pós-condições:** consumidor com política de entrega explícita e caminho de DLQ definido.

---

## UC-004 — Publicar um evento

- **Ator primário:** Módulo produtor (via relay de Outbox).
- **Requisitos:** FR-002, FR-003, FR-004, FR-005, FR-015, FR-018.

**Fluxo principal**
1. O produtor commita estado + linha de `outbox` na mesma transação (contrato do
   `../005-Database/Database.md` §3.9).
2. O relay publica no NATS com o envelope CloudEvents (RFC-0001 §5.2).
3. O `EnvelopeValidator` confere campos obrigatórios, coerência entre `type` e subject,
   `dataschema` registrado e tamanho do payload.
4. O `SubjectRegistry` confirma que o chamador é o `producer_module` declarado.
5. O `FlowController` debita a cota do tenant/agente.
6. A mensagem entra no stream; o ack de JetStream retorna a sequência; o relay marca
   `published = true`.

**Fluxos alternativos**
- **A1** — republicação do mesmo `event.id` dentro de `dedupe_window`: o JetStream
  descarta a duplicata; o resultado é uma única mensagem (FR-005).
- **A2** — publicação *fire-and-forget* (telemetria): sem ack, latência ≤ 2 ms (NFR-001).

**Exceções**
- **E1** — subject não registrado → `AIOS-BUS-0004` (a mensagem permanece no Outbox).
- **E2** — envelope inválido → `AIOS-BUS-0005`.
- **E3** — produtor não autorizado → `AIOS-BUS-0010`.
- **E4** — payload acima do limite → `AIOS-BUS-0006` (envie referência a objeto).
- **E5** — cota excedida → `AIOS-BUS-0007`; o relay recua com backoff.
- **E6** — sem quórum JetStream → `AIOS-BUS-0011`; a mensagem fica no Outbox e é
  republicada depois — nenhuma perda.

**Pós-condições:** evento durável no stream, exatamente uma vez em efeito.

---

## UC-005 — Consumir com ack e reentrega

- **Ator primário:** Módulo consumidor.
- **Requisitos:** FR-001, FR-006, FR-007, FR-018.

**Fluxo principal**
1. O consumidor durável recebe a mensagem com `traceparent` propagado.
2. Deduplica por `event.id` (obrigação do consumidor, RFC-0001 §5.5).
3. Processa e confirma (`ack`).

**Fluxos alternativos**
- **A1** — `nak` explícito com atraso sugerido: reentrega antecipada e controlada.
- **A2** — processamento longo: `in-progress` estende o `ack_wait` sem perder a posse.

**Exceções**
- **E1** — sem ack dentro de `ack_wait` (30 s): reentrega com backoff
  `[1s, 5s, 30s, 2min]`.
- **E2** — `max_deliver` (5) esgotado → DLQ (UC-006).
- **E3** — `max_ack_pending` (1000) atingido: a entrega é contida
  (`AIOS-BUS-0014`) e o evento `consumer.stalled` é emitido — o consumidor lento não
  contamina os demais.

**Pós-condições:** mensagem processada, reentregue ou quarentenada — nunca perdida.

---

## UC-006 — Tratar mensagem envenenada (DLQ)

- **Ator primário:** `DeadLetterManager` (ator de sistema).
- **Requisitos:** FR-007.

**Fluxo principal**
1. A mensagem esgota `max_deliver` no consumidor.
2. O `DeadLetterManager` grava `comm.dlq_entry` com envelope preservado,
   `delivery_count`, `last_error` (sem PII), stream e consumidor de origem.
3. Emite `aios.<tenant>.comm.delivery.deadlettered` e alerta se a taxa cruzar NFR-012.

**Exceções**
- **E1** — DLQ indisponível (falha de escrita em `comm`): a mensagem **NÃO DEVE** ser
  descartada; permanece no stream até o limite de retenção e o incidente é P1.

**Pós-condições:** falha visível e inspecionável, com contexto suficiente para
diagnóstico — o oposto de sumiço silencioso.

---

## UC-007 — Replay de uma entrada de DLQ

- **Ator primário:** SRE ou engenheiro do módulo consumidor.
- **Requisitos:** FR-008, FR-017.
- **Pré-condições:** causa raiz corrigida e implantada.

**Fluxo principal**
1. `GET /v1/comm/dlq` para inspecionar; identifica-se o padrão de falha.
2. `POST /v1/comm/dlq/{id}:replay` com `Idempotency-Key`; PDP autoriza
   `bus:dlq:replay`.
3. A mensagem é reinjetada no subject original com o mesmo `event.id` (consumidores
   deduplicam se já a processaram parcialmente).
4. Estado → `replayed`; evento `delivery.replayed` emitido.

**Fluxos alternativos**
- **A1** — descarte deliberado: `:discard` com justificativa obrigatória, auditada.

**Exceções**
- **E1** — replay antes da correção: repete o mesmo defeito. Por isso
  `bus.dlq.auto_replay` é `false` por default (ADR-0205) — replay é decisão, não
  automatismo.

**Pós-condições:** mensagem reprocessada ou explicitamente descartada, com trilha.

---

## UC-008 — Replay de stream para recuperação

- **Ator primário:** SRE / engenheiro em investigação.
- **Requisitos:** FR-008, FR-017.

**Fluxo principal**
1. `POST /v1/comm/streams/{name}:replay` com alvo por tempo (`since`) ou sequência, e
   filtro opcional de subject.
2. PDP autoriza `bus:stream:replay`.
3. O `ReplayService` cria um consumidor **isolado e efêmero**, entrega ao destino
   indicado e o remove ao final.

**Exceções**
- **E1** — alvo fora da janela de retenção do stream → `AIOS-BUS-0001`.
- **E2** — tentativa de replay para o consumidor de produção: recusada. Replay usa
  consumidor isolado, para não reprocessar em quem já está saudável.

**Pós-condições:** dados reprocessados sem perturbar a produção.

---

## UC-009 — Provisionar conta de tenant

- **Ator primário:** Operador de plataforma (onboarding de tenant).
- **Requisitos:** FR-009, FR-017.

**Fluxo principal**
1. `POST /v1/comm/accounts` com o slug do tenant.
2. PDP autoriza `bus:account:provision`.
3. O `TenantAccountManager` cria a conta NATS, solicita ao `../021-Security/` a
   emissão de NKey/JWT, aplica limites default (conexões, assinaturas, payload) e
   emite `account.provisioned`.

**Exceções**
- **E1** — conta já existente: operação idempotente, retorna a existente.
- **E2** — cofre indisponível: provisionamento falha; **não** se cria conta sem
  credencial gerenciada.

**Pós-condições:** tenant isolado no roteamento desde o primeiro byte.

---

## UC-010 — Exportar subject entre contas

- **Ator primário:** Arquiteto de plataforma.
- **Requisitos:** FR-019, FR-017.
- **Pré-condições:** justificativa de negócio registrada.

**Fluxo principal**
1. `POST /v1/comm/accounts/{tenant}/exports` declarando subject, conta de destino e
   direção.
2. PDP autoriza `bus:account:export` (capability de **alta sensibilidade**).
3. O `TenantAccountManager` cria o *export/import* no NATS e registra em
   `../025-Audit/` com justificativa, autor e escopo.

**Exceções**
- **E1** — capability ausente → `AIOS-BUS-0002`.
- **E2** — subject com `data_class` sensível: recusado por política; a exportação
  cruza a fronteira de tenant e por isso é tratada como exceção, nunca como rotina.

**Pós-condições:** único caminho legítimo de tráfego entre contas, com trilha.

---

## UC-011 — Abrir sessão A2A entre agentes internos

- **Ator primário:** Agente (via `../007-Agent-Runtime/`).
- **Requisitos:** FR-013, FR-017, FR-020.

**Fluxo principal**
1. `POST /v1/comm/a2a/sessions` com `peer_urn`, capacidades desejadas e
   `Idempotency-Key` → `Requested` (T-01).
2. O par responde ao handshake dentro de `bus.a2a.handshake_timeout_ms` (3 s) →
   `Negotiating` (T-02).
3. O `PolicyClient` submete **cada** capacidade ao PDP; todas autorizadas →
   `Established` (T-04), com canal privado `aios.<t>.a2a.session.<ULID>`.
4. Os agentes trocam mensagens no canal; volume contabilizado em `message_count` e
   `bytes_total`.
5. Encerramento explícito ou por ociosidade → `Closing` → `Closed` (T-08, T-09), com
   evento `comm.a2a.closed`.

**Fluxos alternativos**
- **A1** — cota de tráfego estourada → `Suspended` (T-06); mensagens recusadas com
  `AIOS-A2A-0004` até a renovação (T-07).
- **A2** — `policy.bundle.updated` revoga capacidade em uso → `Suspended` (T-06).

**Exceções**
- **E1** — PDP nega → `Rejected` (T-05) com `AIOS-A2A-0001`.
- **E2** — sem resposta ao handshake → `Failed` (T-03) com `AIOS-A2A-0002`.
- **E3** — cota de sessões do agente excedida → `AIOS-A2A-0006`.
- **E4** — tentativa de ampliar capacidades após `Established` → recusada
  (invariante I4); é necessária **nova** sessão.

**Pós-condições:** colaboração entre agentes governada, cotada e auditável.

---

## UC-012 — Sessão A2A com par externo

- **Ator primário:** Agente interno; par de outra organização.
- **Requisitos:** FR-014, FR-017.
- **Pré-condições:** `bus.a2a.allow_external_peers = true` para o tenant.

**Fluxo principal**
1. Mesmo fluxo do UC-011, com verificação adicional de identidade federada do par
   (mTLS + JWT emitido por emissor confiável, validado com o `../021-Security/`).
2. As capacidades concedidas a par externo são um **subconjunto** das internas,
   conforme perfil da RFC-0201.

**Exceções**
- **E1** — identidade não verificável ou revogada → `AIOS-A2A-0005`; sessões
  existentes com esse par são encerradas (T-10).
- **E2** — capacidade fora do perfil externo → `AIOS-A2A-0007`.
- **E3** — `allow_external_peers = false` → `AIOS-A2A-0001`.

**Pós-condições:** federação possível sem abrir o barramento interno a terceiros.

---

## UC-013 — Difusão para um *Agent Group*

- **Ator primário:** Agente coordenador ou módulo `009-Scheduler`.
- **Requisitos:** FR-012.
- **Pré-condições:** canal de grupo declarado (`PUT /v1/comm/groups/{group}/channel`).

**Fluxo principal**
1. O produtor publica **uma** vez no `subject_prefix` do canal.
2. O `SelectiveCommunicationRouter` valida `scope` e `fanout_limit`.
3. O broker realiza o fan-out aos assinantes do grupo — **1 publicação, N entregas**
   (NFR-016).

**Fluxos alternativos**
- **A1** — `scope = hierarchy`: propaga também a subgrupos declarados.

**Exceções**
- **E1** — membros acima de `fanout_limit` (256) → `AIOS-BUS-0008`. A orientação é
  dividir o grupo ou elevar o limite conscientemente: uma difusão para 10⁴ agentes é
  quase sempre um erro de modelagem, não uma necessidade.
- **E2** — cota do canal excedida → `AIOS-BUS-0007`.

**Pós-condições:** coordenação em massa com custo O(1) para o produtor.

---

## UC-014 — Aplicar cota e conter *slow consumer*

- **Ator primário:** `FlowController` (ator de sistema).
- **Requisitos:** FR-010, FR-011.

**Fluxo principal**
1. Cada publicação debita contadores por tenant e por agente (janela deslizante em Redis).
2. Excedida a cota, novas publicações recebem `AIOS-BUS-0007` com `Retry-After`, e o
   evento `quota.throttled` é emitido.
3. Em paralelo, consumidores com `pending` acima de `max_ack_pending` têm a entrega
   contida e geram `consumer.stalled`.

**Exceções**
- **E1** — produtor ignora o recuo e insiste: cotas continuam rejeitando; o custo é do
  produtor, não do barramento.
- **E2** — cota legítima insuficiente: ajuste vem do `../026-Cost-Optimizer/` via
  `cost.budget.updated`, não por edição manual permanente.

**Pós-condições:** um tenant barulhento não degrada os demais (NFR-011).

---

## UC-015 — Request/reply entre serviços

- **Ator primário:** Módulo do plano de controle (ex.: `006` consultando `022`).
- **Requisitos:** FR-001.

**Fluxo principal**
1. O solicitante publica com `reply-to` gerado e aguarda até
   `bus.request.timeout_ms` (5 s).
2. O respondente processa e publica a resposta no `reply-to`, preservando
   `traceparent`.

**Exceções**
- **E1** — sem resposta no prazo → `AIOS-BUS-0012` (retriable). O solicitante decide:
  repetir (se idempotente) ou degradar.
- **E2** — nenhum respondente assinando o subject: falha imediata em vez de espera
  inútil pelo timeout.

**Pós-condições:** chamada síncrona sobre transporte assíncrono, com timeout explícito
e correlação preservada.

---

## Referências

- Requisitos: `./FunctionalRequirements.md` · `./NonFunctionalRequirements.md`
- Sequências: `./SequenceDiagrams.md` · FSM: `./StateMachine.md` · API: `./API.md`
- Brief: `./_DESIGN_BRIEF.md`
