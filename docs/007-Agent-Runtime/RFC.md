---
Documento: RFC.md
Módulo: 007-Agent-Runtime
Status: Draft
Versão: 0.1
Última atualização: 2026-07-21
Responsável (RACI-A): Arquiteto do Módulo 007 — Agent Runtime
ADRs relacionados: ADR-0072, ADR-0075
RFCs relacionados: RFC-0001, RFC-0070, RFC-0071
Depende de: ../003-RFC/, ../008-Agent-Lifecycle/, ../015-Tool-Manager/, ../021-Security/
---

# 007-Agent-Runtime — Índice de RFCs

> **Escopo deste documento.** Este documento **NÃO** especifica protocolo
> algum por si — é um **índice navegável** das *Requests for Comments*
> (RFC) que especificam contratos consumidos ou propostos por este módulo.
> O conteúdo normativo pleno reside em `../003-RFC/`, no formato IETF
> definido por `../003-RFC/TEMPLATE.md` (Abstract · Status/Terminologia ·
> Motivação · Terminologia · Especificação · Segurança · Privacidade ·
> Registros IANA-like · Compatibilidade · Referências · Apêndices). Este
> arquivo **DEVE** ser mantido sincronizado com `./_DESIGN_BRIEF.md` §11 e
> com `../003-RFC/README.md`.

> **Regra de não-redefinição.** Contratos centrais já especificados pela
> RFC-0001 (URN `urn:aios:<tenant>:<tipo>:<id>`, envelope de evento
> CloudEvents, subjects `aios.<tenant>.<dominio>.<entidade>.<acao>`,
> envelope de erro RFC 7807 `AIOS-<DOMINIO>-<NNNN>`, `Idempotency-Key`,
> cabeçalhos `traceparent`/`X-AIOS-Tenant`) **NÃO SÃO** redefinidos por
> nenhuma RFC deste módulo — apenas **especializados** para o domínio de
> execução cognitiva e sandbox de agente.

---

## 1. RFC Consumida (Baseline)

| RFC | Título | Status | Relação com o Agent Runtime |
|-----|--------|--------|----------------------------------|
| [RFC-0001](../003-RFC/RFC-0001-Architecture-Baseline.md) | AIOS Architecture Baseline & Core Contracts | Accepted | **Baseline consumida, não redefinida.** O Agent Runtime herda integralmente: formato de URN (§5.1), envelope de evento CloudEvents (§5.2), convenção de subjects (§5.3), envelope de erro RFC 7807/`google.rpc.Status` (§5.4), semântica de idempotência (§5.5), cabeçalhos de correlação obrigatórios (§5.6) e regras de versionamento de API (§5.7). Toda RFC específica do módulo (§2 abaixo) DEVE citar a seção pertinente da RFC-0001 em vez de reespecificá-la. |

### 1.1 Mapeamento de Uso da RFC-0001 pelo Agent Runtime

| Seção da RFC-0001 | Contrato | Uso concreto no Agent Runtime |
|--------------------|----------|-----------------------------------|
| §5.1 URN | `urn:aios:<tenant>:<tipo>:<id>` | `agent_urn`, `task_urn`, `tool_urn`, `resolved_model_urn`, `plan_urn` em todas as entidades de `./Database.md` §2. |
| §5.2 Envelope de Evento (CloudEvents) | Estrutura `{id, source, type, time, datacontenttype, subject, tenant, data, traceparent}` | Payload de todo evento de `./Events.md` §2, publicado via outbox local (ADR-0076). |
| §5.3 Convenção de Subjects | `aios.<tenant>.<dominio>.<entidade>.<acao>` | Domínios `agent` e `task`; entidades `runtime`, `execution` (ex.: `aios.<t>.agent.runtime.booted`). |
| §5.4 Envelope de Erro | `type/title/status/detail/instance/code/retriable` (REST); `google.rpc.Status`+`ErrorInfo` (gRPC) | Todo erro do domínio `RUNTIME` (`./API.md` §4). |
| §5.5 Idempotência | `Idempotency-Key`, retenção ≥ 24h | `Boot`/`SubmitTask` (`./API.md` §5). |
| §5.6 Correlação | `traceparent`, `X-AIOS-Tenant` | Extraídos em todo método `aios.runtime.v1`; propagados a todo evento e span OTel (`TelemetryEmitter`). |
| §5.7 Versionamento | Versionamento semântico de API | Pacote gRPC `aios.runtime.v1`; base REST `/v1/runtime` (rede de controle). |

---

## 2. RFCs Propostas por Este Módulo

> **Estado de redação.** As RFCs abaixo estão **planejadas** e seu conteúdo
> pretendido está descrito no `./_DESIGN_BRIEF.md` (especialmente §5
> Superfície de API e §12 Segurança), que é a fonte única de verdade até
> que cada RFC seja redigida como arquivo próprio em `../003-RFC/` no
> formato IETF completo e promovida de `Draft` para `Accepted` conforme o
> ciclo de vida `Draft → Discussion → Last Call → Accepted`. Nenhum destes
> arquivos existe fisicamente em `../003-RFC/` no momento desta versão do
> índice.

| RFC | Título | Status | Escopo pretendido |
|-----|--------|--------|-------------------------|
| RFC-0070 | Runtime↔Supervisor Control Protocol | Planned (Draft a redigir) | Especificação normativa completa do protocolo de controle entre o Runtime Supervisor (.NET 10) e o processo Agent Runtime (Python): `aios.runtime.v1.RuntimeControl` (`Boot`/`Suspend`/`Resume`/`Kill`/`Drain`/`GetStatus`) e `aios.runtime.v1.AgentExecution` (`SubmitTask`/`CancelTask`/`StreamSteps`), semântica de cada método, matriz de idempotência, mapeamento de erros, e regras de compatibilidade retroativa entre versões major da ABI. |
| RFC-0071 | MCP Host Sandboxing Profile | Planned (Draft a redigir) | Especificação normativa do perfil de segurança do `McpHost` embarcado no sandbox: superfície MCP exposta, *handshake* com `015-Tool-Manager`, restrições de `seccomp-bpf`/`cgroups v2`/egress aplicáveis especificamente ao processo do host MCP, e regras de evolução aditiva do catálogo de ferramentas sem exigir *restart* de sessão. |

### 2.1 Justificativa de Cada RFC (Por Que É uma RFC e Não uma Seção da `Architecture.md`)

| RFC | Justificativa |
|-----|-----------------|
| RFC-0070 | O protocolo Runtime↔Supervisor é um **contrato de protocolo entre dois planos** (dados/controle), potencialmente mantidos por equipes distintas, consumido também por `009-Scheduler` (indiretamente) — RFCs especificam protocolos inteiros com processo de comentário público (`Draft → Discussion → Last Call`), diferente de uma decisão pontual de ADR. |
| RFC-0071 | O perfil de sandboxing do MCP host é consumido por `015-Tool-Manager` (que precisa saber quais garantias de isolamento pode assumir ao registrar uma ferramenta) e por `021-Security` (que audita o perfil) — mudança de superfície de segurança exige coordenação multi-módulo típica de processo de RFC, não de ADR isolada. |

### 2.2 Estrutura Obrigatória que Cada RFC Deste Módulo DEVE Seguir

Conforme `../003-RFC/README.md`, ambas as RFCs propostas DEVEM conter, nesta
ordem:

1. **Abstract** — resumo de 1 parágrafo do contrato especificado.
2. **Status & terminologia** (RFC 2119/8174) — palavras normativas.
3. **Motivação** — por que o contrato existe (referenciar `_DESIGN_BRIEF.md` §1).
4. **Terminologia** — reuso de `../040-Glossary/Glossary.md`, sem redefinir termos.
5. **Especificação (normativa)** — o contrato em si (protocolo de controle ou perfil de sandboxing).
6. **Considerações de segurança** — referenciar `./Security.md` deste módulo.
7. **Considerações de privacidade (LGPD/GDPR)** — referenciar `_DESIGN_BRIEF.md` §12.3.
8. **Considerações de registro (IANA-like)** — reserva de domínio de erro `RUNTIME`, pacote gRPC `aios.runtime.v1`, subjects `agent.runtime.*`/`task.execution.*`.
9. **Compatibilidade e versionamento** — regras de evolução sem quebra.
10. **Referências** — normativas (RFC-0001) e informativas (ADRs do §2 de `./ADR.md`).
11. **Apêndices** — exemplos de request/response, payloads de evento.

---

## 3. Rastreabilidade RFC → Componente → Requisito

| RFC | Componente(s) primário(s) (ver `./Architecture.md` §5) | Requisito(s) relacionado(s) | ADR(s) de suporte |
|-----|----------------------------------------------------------|----------------------------------|-------------------------|
| RFC-0001 (baseline) | `EventPublisher`, `TelemetryEmitter`, `SupervisorChannel` | FR-011, FR-012, NFR-009, NFR-012 | ADR-0076, ADR-0010 (global) |
| RFC-0070 (proposta) | `SupervisorChannel`, `CognitiveLoopEngine` (via `AgentExecution`) | FR-001, FR-009, FR-011, FR-013, NFR-014, NFR-015 | ADR-0072, ADR-0073, ADR-0079 |
| RFC-0071 (proposta) | `McpHost`, `SandboxManager`, `ToolInvoker` | FR-004, FR-007, NFR-002, NFR-008, NFR-014 | ADR-0071, ADR-0075 |

---

## 4. Impacto de Mudanças Futuras na RFC-0001 sobre o Agent Runtime

| Cenário de mudança na RFC-0001 | Impacto esperado no Agent Runtime | Mitigação |
|-------------------------------------|----------------------------------------|-----------|
| Alteração do formato de URN (§5.1) | Migração de todo `agent_urn`/`task_urn`/`tool_urn`/`resolved_model_urn` em `runtime.*` (`./Database.md`). | Migração versionada com dupla-leitura (RFC-0001 §9); ADR de suporte antes de aplicar. |
| Alteração do envelope de evento CloudEvents (§5.2) | Todo payload de `./Events.md` §2 precisa ser revalidado contra o novo schema. | Versionamento de `dataschema`; consumidores toleram campos aditivos (`./Events.md` §6). |
| Alteração da semântica de idempotência (§5.5) | Janela de retenção de `Idempotency-Key` de `Boot`/`SubmitTask` pode precisar de ajuste. | Retenção já ≥ mínimo exigido; ajuste centralizado em um único componente. |
| Novo cabeçalho de correlação obrigatório (§5.6) | `SupervisorChannel`/`AgentExecution` precisam extrair e propagar o novo cabeçalho. | Extração de cabeçalhos é centralizada, baixo *fan-out* de mudança. |

---

## 5. Processo de Promoção (Draft → Accepted)

1. O autor redige o arquivo `RFC-00NN-<slug>.md` em `../003-RFC/` seguindo a
   estrutura obrigatória da §2.2 e o `../003-RFC/TEMPLATE.md`.
2. A especificação **NÃO PODE** contradizer o `./_DESIGN_BRIEF.md` deste
   módulo nem redefinir contratos já centrais na RFC-0001; qualquer
   necessidade de divergência exige atualizar o brief primeiro.
3. Ciclo de comentário público interno: `Draft` → `Discussion` (revisão dos
   módulos consumidores listados na §3) → `Last Call` (janela final de
   objeções) → `Accepted`.
4. Ao ser aceita, o `Status` muda para `Accepted` no arquivo da RFC e neste
   índice; `../003-RFC/README.md` global é atualizado.
5. RFCs aceitas evoluem apenas por **RFC sucessora** que a torna
   `Obsoleted by RFC-XXXX` — nunca por edição retroativa do texto
   normativo.

---

## 6. Referências

- Índice global de RFCs: `../003-RFC/README.md`
- Template de RFC: `../003-RFC/TEMPLATE.md`
- Contrato baseline: `../003-RFC/RFC-0001-Architecture-Baseline.md`
- Fonte de verdade do módulo: `./_DESIGN_BRIEF.md` §5, §11
- Índice de ADRs do módulo: `./ADR.md`
- API do módulo: `./API.md`
- Eventos do módulo: `./Events.md`

*Fim do índice de RFCs do módulo 007-Agent-Runtime.*
