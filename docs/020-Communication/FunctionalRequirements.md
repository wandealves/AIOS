---
Documento: FunctionalRequirements
Módulo: 020-Communication
Status: Draft
Versão: 0.1
Última atualização: 2026-07-22
Responsável (RACI-A): Arquiteto do Módulo 020-Communication
ADRs relacionados: ADR-0200, ADR-0201, ADR-0202, ADR-0203, ADR-0204, ADR-0205, ADR-0206, ADR-0207, ADR-0208, ADR-0209
RFCs relacionados: RFC-0001, RFC-0200, RFC-0201
Depende de: _DESIGN_BRIEF.md §7.1, 021-Security, 022-Policy, 025-Audit, 026-Cost-Optimizer
---

# 020-Communication — Requisitos Funcionais

Requisitos derivados de `./_DESIGN_BRIEF.md` §7.1. Prioridade em **MoSCoW**. Palavras
normativas conforme RFC 2119. Todo FR é verificável por ao menos um caso de teste em
`./Testing.md`.

---

## 1. Tabela de Requisitos Funcionais

| ID | Requisito | Prioridade | Critério de aceite | Origem |
|----|-----------|-----------|--------------------|--------|
| FR-001 | O módulo DEVE operar pub/sub, request/reply e streams duráveis sobre NATS + JetStream. | Must | Os três padrões exercitados em teste de integração com cluster real de 3 nós. | Brief §1.2 R-01 |
| FR-002 | O módulo DEVE rejeitar publicação em subject fora do registro ou fora da convenção da RFC-0001 §5.3. | Must | Subject não registrado ou malformado → `AIOS-BUS-0004`; ambos os casos cobertos por teste. | Brief §1.2 R-02 |
| FR-003 | O módulo DEVE validar o envelope CloudEvents no momento da publicação. | Must | Campo obrigatório ausente, `type` incoerente com o subject ou `dataschema` desconhecido → `AIOS-BUS-0005` / `AIOS-BUS-0015`. | Brief §1.2 R-04 |
| FR-004 | O módulo DEVE garantir que apenas o `producer_module` declarado publique em cada subject. | Must | Publicação por outro módulo → `AIOS-BUS-0010`; tentativa auditada. | Brief §12.1 |
| FR-005 | O módulo DEVE garantir entrega at-least-once com dedupe por `event.id` dentro de `dedupe_window`. | Must | Publicação duplicada na janela resulta em **uma** mensagem no stream. | Brief §1.2 R-05, RFC-0001 §5.5 |
| FR-006 | O módulo DEVE preservar a ordem das mensagens por stream. | Must | Sequência de 10⁵ mensagens chega em ordem ao consumidor durável. | Brief §1.2 R-05 |
| FR-007 | O módulo DEVE reentregar mensagens não confirmadas com backoff e enviá-las à DLQ ao esgotar `max_deliver`. | Must | Consumidor que nunca confirma gera entrada em `comm.dlq_entry` e evento `delivery.deadlettered`. | Brief §1.2 R-10 |
| FR-008 | O módulo DEVE permitir replay de DLQ e de stream (por tempo ou sequência) em consumidor isolado. | Must | Replay não perturba consumidores de produção; evento `delivery.replayed` emitido. | Brief §1.2 R-10 |
| FR-009 | O módulo DEVE isolar tenants em contas NATS distintas. | Must | Credencial de `acme` não publica nem assina em `aios.globex.>`; tentativa → `AIOS-BUS-0003`. | Brief §1.2 R-06 |
| FR-010 | O módulo DEVE aplicar cotas de tráfego por tenant e por agente, com sinalização de recuo. | Must | Excesso → `AIOS-BUS-0007` e evento `quota.throttled`. | Brief §1.2 R-07 |
| FR-011 | O módulo DEVE detectar e conter *slow consumers*. | Must | `max_ack_pending` excedido → `AIOS-BUS-0014` e evento `consumer.stalled`; demais consumidores não degradam. | Brief §1.2 R-07 |
| FR-012 | O módulo DEVE prover canais de *Agent Group* com limite de fan-out e escopo de difusão. | Must | Difusão acima de `fanout_limit` → `AIOS-BUS-0008`. | Brief §1.2 R-08 |
| FR-013 | O módulo DEVE estabelecer, autorizar e encerrar sessões A2A conforme a FSM de `./StateMachine.md`. | Must | Sessão sem capability → `AIOS-A2A-0001`; transição inválida → `AIOS-A2A-0003`. | Brief §1.2 R-09, §4 |
| FR-014 | O módulo DEVERIA suportar par A2A **externo** com identidade verificada por mTLS/JWT federado. | Should | Par com identidade revogada → `AIOS-A2A-0005` e encerramento (T-10). | Brief §1.2 R-09 |
| FR-015 | O módulo DEVE rejeitar payload acima de `bus.publish.max_payload_bytes`. | Must | Payload maior → `AIOS-BUS-0006`, com orientação de usar referência a objeto. | Brief §1.3 NR-08 |
| FR-016 | O módulo DEVE declarar streams e consumidores de forma idempotente e versionada. | Must | `PUT` repetido não altera estado; mudança incompatível migra sem perda (ADR-0203). | Brief §4.4 |
| FR-017 | O módulo DEVE consultar o PDP antes de toda operação administrativa e de toda abertura de sessão A2A. | Must | Sem capability → `AIOS-BUS-0002` / `AIOS-A2A-0001`; decisão auditada em `../025-Audit/`. | Brief §1.2 R-11 |
| FR-018 | O módulo DEVE propagar `traceparent` em 100% das mensagens e emitir telemetria OTel. | Must | Trace contínuo do produtor ao consumidor em teste ponta a ponta. | Brief §1.2 R-12 |
| FR-019 | O módulo DEVE permitir *export* de subject entre contas somente por declaração explícita e autorizada. | Must | Export sem capability → `AIOS-BUS-0002`; export concedido é registrado em `../025-Audit/`. | Brief §12.1 |
| FR-020 | O módulo DEVERIA encerrar automaticamente sessões A2A ociosas e liberar assinaturas de agentes extintos. | Should | Ociosidade > `bus.a2a.idle_timeout_ms` → T-08; `agent.lifecycle.terminated` limpa recursos do agente. | Brief §6.2 |

---

## 2. Regras de Negócio Transversais

| # | Regra | Aplicação |
|---|-------|-----------|
| RN-01 | Toda mutação administrativa **DEVE** aceitar `Idempotency-Key` e produzir efeito único (RFC-0001 §5.5). | FR-016, FR-013, FR-008 |
| RN-02 | Toda operação privilegiada e toda sessão A2A **DEVEM** ser auditadas em `../025-Audit/`. | FR-013, FR-017, FR-019 |
| RN-03 | Nenhum documento ou operação **PODE** redefinir contratos centrais (envelope, subjects, idempotência, correlação) — eles vêm da RFC-0001. | Todos |
| RN-04 | Um subject **DEVE** identificar um **tipo** de evento, nunca uma instância (é proibido `…agent.<ULID>.status`). | FR-002 |
| RN-05 | Replay **NÃO DEVE** ser automático por default: `bus.dlq.auto_replay = false`, porque replay cego reproduz o mesmo defeito. | FR-008 |
| RN-06 | Sob PDP indisponível, o módulo nega **operações novas** (`fail-closed`) mas **NÃO DEVE** interromper o tráfego já autorizado. | FR-017 |

---

## 3. Exemplo de Verificação — FR-004

```bash
# 025-Audit tenta publicar em um subject cujo produtor declarado é o 006-Kernel
nats pub aios.acme.agent.lifecycle.terminated \
  --creds /etc/aios/creds/audit.creds \
  '{"specversion":"1.0","id":"01J9ZC4Q7S9U1W3Y5A7C9E1G3J",
    "source":"urn:aios:acme:service:audit",
    "type":"aios.agent.lifecycle.terminated","tenant":"acme", ... }'
```

```json
{
  "code": "AIOS-BUS-0010",
  "title": "Producer Not Authorized For Subject",
  "status": 409,
  "detail": "Subject aios.acme.agent.lifecycle.terminated tem producer_module=006-Kernel; chamador é 025-Audit.",
  "retriable": false
}
```

Sem essa regra, qualquer serviço com credencial do tenant poderia forjar um evento de
ciclo de vida e induzir os demais módulos a agir sobre um fato que nunca ocorreu.

---

## 4. Rastreabilidade

A matriz FR → UC → Teste está em `./Requirements.md` §3. Cada FR aparece literalmente,
com o mesmo identificador, em `./UseCases.md`, `./Testing.md` e — quando tem meta
numérica — em `./NonFunctionalRequirements.md`.

---

## 5. Referências

- Brief: `./_DESIGN_BRIEF.md` §7.1
- Requisitos não-funcionais: `./NonFunctionalRequirements.md`
- Casos de uso: `./UseCases.md` · Testes: `./Testing.md`
- Contratos centrais: `../003-RFC/RFC-0001-Architecture-Baseline.md`
