---
Documento: RFC.md
Módulo: 008-Agent-Lifecycle
Status: Draft
Versão: 0.1
Última atualização: 2026-07-20
Responsável (RACI-A): Arquiteto do Módulo 008 — Agent Lifecycle
ADRs relacionados: ADR-0080, ADR-0081, ADR-0082, ADR-0083, ADR-0085
RFCs relacionados: RFC-0001, RFC-0080
Depende de: 006-Kernel, 007-Agent-Runtime, 009-Scheduler, 010-Memory, 020-Communication, 021-Security, 022-Policy, 027-Cluster, 003-RFC
---

# 008-Agent-Lifecycle — Índice de RFCs

> **Escopo deste documento.** Este documento **NÃO** especifica protocolo
> algum por si — é um **índice navegável** das *Requests for Comments* (RFC)
> que especificam contratos consumidos ou propostos por este módulo. O
> conteúdo normativo pleno reside em `../003-RFC/`, no formato IETF definido
> por `../003-RFC/TEMPLATE.md` (Abstract · Status/Terminologia · Motivação ·
> Terminologia · Especificação · Segurança · Privacidade · Registros IANA-like
> · Compatibilidade · Referências · Apêndices). Este arquivo **DEVE** ser
> mantido sincronizado com `../_DESIGN_BRIEF.md` §11.2 e com
> `../003-RFC/README.md`.

> **Regra de não-redefinição.** Contratos centrais já especificados pela
> RFC-0001 (URN `urn:aios:<tenant>:<tipo>:<id>`, envelope de evento
> CloudEvents, subjects `aios.<tenant>.<dominio>.<entidade>.<acao>`, envelope
> de erro RFC 7807 `AIOS-<DOMINIO>-<NNNN>`, `Idempotency-Key`, cabeçalhos
> `traceparent`/`X-AIOS-Tenant`) **NÃO SÃO** redefinidos por nenhuma RFC deste
> módulo — apenas **especializados** para o domínio de ciclo de vida do
> agente, hibernação e checkpoint/restore.

---

## 1. RFC Consumida (Baseline)

| RFC | Título | Status | Relação com o Agent Lifecycle |
|-----|--------|--------|-----------------------------------|
| [RFC-0001](../003-RFC/RFC-0001-Architecture-Baseline.md) | AIOS Architecture Baseline & Core Contracts | Accepted | **Baseline consumida, não redefinida.** O módulo 008 herda integralmente: formato de URN (§5.1), envelope de evento CloudEvents (§5.2), convenção de subjects (§5.3), envelope de erro RFC 7807 (§5.4), semântica de idempotência (§5.5) e cabeçalhos de correlação obrigatórios (§5.6). Toda RFC específica do módulo (§2 abaixo) DEVE citar a seção pertinente da RFC-0001 em vez de reespecificá-la. |

### 1.1 Mapeamento de uso da RFC-0001 pelo Agent Lifecycle

| Seção da RFC-0001 | Contrato | Uso concreto no Agent Lifecycle |
|--------------------|----------|-------------------------------------|
| §5.1 URN | `urn:aios:<tenant>:<tipo>:<id>` | `agent_id` de `AgentControlBlock` (`urn:aios:<tenant>:agent:<ULID>`); `checkpoint_id` (`urn:aios:<tenant>:checkpoint:<ULID>`) e `migration_id` (`urn:aios:<tenant>:migration:<ULID>`), tipos propostos ao registro `004-API/Resources.md` (`_DESIGN_BRIEF.md` §0). |
| §5.2 Envelope de Evento (CloudEvents) | Estrutura `{id, source, type, time, datacontenttype, subject, tenant, data, traceparent}` | Payload de `LifecycleOutbox.payload` para os 13 tipos de evento `agent.lifecycle.*` (`Events.md` §6.1). |
| §5.3 Convenção de Subjects | `aios.<tenant>.<dominio>.<entidade>.<acao>` | Domínio `agent`, entidade `lifecycle` (`aios.<tenant>.agent.lifecycle.<acao>`), conforme faixa reservada ao módulo 008. |
| §5.4 Envelope de Erro (RFC 7807) | `type/title/status/detail/instance/code/retriable` | Toda resposta de erro do módulo (`API.md` §catálogo), com `code` no domínio `AIOS-LIFECYCLE-<NNNN>` (faixa 0001–0099). |
| §5.5 Idempotência | `Idempotency-Key`, retenção ≥ 24h | Toda mutação POST/DELETE de `/v1/agents/**` (`spawn`, `suspend`, `resume`, `hibernate`, `wake`, `migrate`, `checkpoint`, `restore`, `terminate`) exige `Idempotency-Key`; mesmo resultado sob repetição (NFR-013). |
| §5.6 Correlação | `traceparent`, `X-AIOS-Tenant` | Propagados por `LifecycleApiSurface`, persistidos em `LifecycleTransition.correlation` e anexados a todo span OTel de `LifecycleTelemetry`. |
| §5.7 Versionamento | Versionamento semântico de API | Pacote gRPC `aios.lifecycle.v1`; base REST `/v1/agents`. |

---

## 2. RFC Proposta por Este Módulo

> **Estado de redação.** A RFC abaixo está **planejada** e seu conteúdo
> pretendido está descrito no `../_DESIGN_BRIEF.md` (especialmente §3 modelo
> de dados de `Checkpoint`/`HibernationRecord`, §4 FSM canônica e §9 modos de
> falha), que é a fonte única de verdade até que a RFC seja redigida como
> arquivo próprio em `../003-RFC/` no formato IETF completo e promovida de
> `Draft` para `Accepted` conforme o ciclo `Draft → Discussion → Last Call →
> Accepted`. O arquivo não existe fisicamente em `../003-RFC/` no momento
> desta versão do índice.

| RFC | Título | Status | Escopo pretendido |
|-----|--------|--------|---------------------|
| RFC-0080 | Cold Agent Hibernation & Checkpoint/Restore Protocol | Planned (Draft a redigir) | Especificação normativa completa do **protocolo de hibernação e checkpoint/restore**: formato de serialização de ACB + working memory (`msgpack+zstd`, AES-256-GCM — ver ADR-0082), semântica de consistência (`consistent` vs. `crash`), versionamento por `generation`, contrato de materialização (`wake`) com meta `p99 ≤ 250 ms`, protocolo de migração (`quiesce → checkpoint → transfer → materialize → cutover → cleanup` — ver ADR-0085), e regras de integridade (`content_hash`) e de rejeição de checkpoint corrompido. Consolida ADR-0080, ADR-0081, ADR-0082, ADR-0083 e ADR-0085. |

### 2.1 Justificativa da RFC (por que é uma RFC e não uma seção da Architecture.md)

| RFC | Justificativa |
|-----|----------------|
| RFC-0080 | O protocolo de hibernação/checkpoint é um **contrato de dados e de wire format** consumido por múltiplos módulos externos ao 008 (`010-Memory` para o ponteiro de working memory, `027-Cluster` para decisões de migração, `021-Security` para o esquema de cifragem por tenant, além de qualquer ferramenta de disaster recovery/backup) — mudanças de formato exigem migração coordenada de dados já persistidos em produção (checkpoints existentes em MinIO/PostgreSQL), típica de processo de RFC com período de `Discussion`/`Last Call`, e não uma decisão isolada de ADR. Além disso, a semântica de consistência (`consistent`/`crash`) e o versionamento por `generation` precisam de especificação normativa única para que restauração entre versões do protocolo permaneça compatível. |

### 2.2 Estrutura obrigatória que a RFC deste módulo DEVE seguir

Conforme `../003-RFC/README.md`, a RFC proposta DEVE conter, nesta ordem:

1. **Abstract** — resumo de 1 parágrafo do protocolo de hibernação/checkpoint/restore especificado.
2. **Status & terminologia** (RFC 2119/8174) — palavras normativas.
3. **Motivação** — por que o protocolo existe (referenciar `_DESIGN_BRIEF.md` §1.1 R5/R6 e §10 — escala a milhões de agentes).
4. **Terminologia** — reuso de `../040-Glossary/Glossary.md` (em particular a entrada *Cold Agent*), sem redefinir termos.
5. **Especificação (normativa)** — formato de serialização, codec, cifragem, hash de integridade, protocolo de `wake`/`restore`, protocolo de migração, versionamento por `generation`, semântica `consistent`/`crash`.
6. **Considerações de segurança** — referenciar `Security.md` deste módulo (STRIDE §12.2 do brief: tampering de checkpoint, information disclosure de working memory).
7. **Considerações de privacidade (LGPD/GDPR)** — referenciar `_DESIGN_BRIEF.md` §12.3 (cifragem, retenção, direito ao esquecimento via `TombstoneManager`).
8. **Considerações de registro (IANA-like)** — reserva de tipos de URN (`checkpoint`, `migration`), enumeração de `codec`/`encryption`/`consistency`.
9. **Compatibilidade e versionamento** — regras de evolução do formato de checkpoint sem quebrar restores antigos (leitura de múltiplas versões de `codec`).
10. **Referências** — normativas (RFC-0001) e informativas (ADR-0080, ADR-0081, ADR-0082, ADR-0083, ADR-0085).
11. **Apêndices** — exemplos completos de payload de checkpoint (metadados), payload de evento `lifecycle.hibernated`/`lifecycle.woken`/`lifecycle.migrated`.

---

## 3. Rastreabilidade RFC → Componente → Requisito

| RFC | Componente(s) primário(s) (ver `Architecture.md` §2) | Requisito(s) relacionado(s) | ADR(s) de suporte |
|-----|---------------------------------------------------------|--------------------------------|----------------------|
| RFC-0001 (baseline) | `LifecycleApiSurface`, `LifecycleEventPublisher`, `LifecycleTelemetry` | FR-008, FR-012, FR-013, NFR-012, NFR-013 | ADR-0004, ADR-0008, ADR-0010 |
| RFC-0080 (proposta) | `CheckpointService`, `SnapshotStore`, `HibernationController`, `MigrationOrchestrator` | FR-004, FR-005, FR-006, FR-007, NFR-001, NFR-003, NFR-004, NFR-006, NFR-009, NFR-010, NFR-011 | ADR-0080, ADR-0081, ADR-0082, ADR-0083, ADR-0085 |

---

## 4. Impacto de Mudanças Futuras na RFC-0001 sobre o Agent Lifecycle

| Cenário de mudança na RFC-0001 | Impacto esperado no Agent Lifecycle | Mitigação |
|----------------------------------|------------------------------------------|-----------|
| Alteração do formato de URN (§5.1) | Migração de `agent_id`/`checkpoint_id`/`migration_id` em `AgentControlBlock`, `Checkpoint`, `MigrationJob` e em todo evento `agent.lifecycle.*` já persistido/publicado. | Migração versionada com dupla-leitura (RFC-0001 §9); ADR de suporte antes de aplicar; `generation` já isola encarnações, facilitando a transição. |
| Alteração do envelope de evento CloudEvents (§5.2) | Todo payload de `LifecycleOutbox` precisa ser revalidado contra o novo schema; consumidores de 006/007/009/014/027 precisam de compatibilidade. | Versionamento de schema por `dataschema` (`Events.md` §6); consumidores toleram campos aditivos (regra de compatibilidade aditiva). |
| Alteração da semântica de idempotência (§5.5) | `Idempotency-Key` em `spawn`/`hibernate`/`wake`/`migrate`/`restore`/`terminate` pode precisar de ajuste de janela de retenção. | Retenção já configurável; hot-reload de configuração (`Configuration.md`). |
| Novo cabeçalho de correlação obrigatório (§5.6) | `LifecycleApiSurface` e `LifecycleCoordinator` precisam extrair e propagar o novo cabeçalho para `LifecycleTransition.correlation` e para spans OTel. | Extração de cabeçalhos é centralizada em `LifecycleApiSurface` (baixo *fan-out* de mudança). |
| Alteração do envelope de erro RFC 7807 (§5.4) | Catálogo `AIOS-LIFECYCLE-0001`..`0015` precisa remapear campos adicionais (ex.: novo campo obrigatório). | Catálogo de erro versionado junto com `API.md`; testes de contrato cobrem o envelope. |

---

## 5. Processo de Promoção (Draft → Accepted)

1. O autor redige o arquivo `RFC-0080-<slug>.md` em `../003-RFC/` seguindo a
   estrutura obrigatória da §2.2 e o `../003-RFC/TEMPLATE.md`.
2. A especificação **NÃO PODE** contradizer o `../_DESIGN_BRIEF.md` deste
   módulo nem redefinir contratos já centrais na RFC-0001; qualquer
   necessidade de divergência exige atualizar o brief primeiro.
3. Ciclo de comentário público interno: `Draft` → `Discussion` (revisão dos
   módulos consumidores listados na §3 — em particular `010-Memory`,
   `021-Security` e `027-Cluster`) → `Last Call` (janela final de objeções) →
   `Accepted`.
4. Ao ser aceita, o `Status` muda para `Accepted` no arquivo da RFC e neste
   índice; `../003-RFC/README.md` global é atualizado.
5. RFCs aceitas evoluem apenas por **RFC sucessora** que a torna `Obsoleted by
   RFC-XXXX` — nunca por edição retroativa do texto normativo.

---

## 6. Referências

- Índice global de RFCs: `../003-RFC/README.md`
- Template de RFC: `../003-RFC/TEMPLATE.md`
- Contrato baseline: `../003-RFC/RFC-0001-Architecture-Baseline.md`
- Fonte de verdade do módulo: `./_DESIGN_BRIEF.md` §3, §4, §11.2
- Índice de ADRs do módulo: `./ADR.md`
- API do módulo: `./API.md`
- Eventos do módulo: `./Events.md`
- Modelo de dados do módulo: `./Database.md`

*Fim do índice de RFCs do módulo 008-Agent-Lifecycle.*
