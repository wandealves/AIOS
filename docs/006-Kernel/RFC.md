---
Documento: RFC.md
Módulo: 006-Kernel
Status: Draft
Versão: 0.1
Última atualização: 2026-07-20
Responsável (RACI-A): Arquiteto do Módulo 006-Kernel
ADRs relacionados: ADR-0008, ADR-0060, ADR-0061, ADR-0062, ADR-0063
RFCs relacionados: RFC-0001, RFC-0006, RFC-0007
Depende de: 001-Architecture, 022-Policy, 009-Scheduler, 008-Lifecycle, 003-RFC
---

# 006-Kernel — Índice de RFCs

> **Escopo deste documento.** Este documento **NÃO** especifica protocolo algum
> por si — é um **índice navegável** das *Requests for Comments* (RFC) que
> especificam contratos consumidos ou propostos por este módulo. O conteúdo
> normativo pleno reside em `../003-RFC/`, no formato IETF definido por
> `../003-RFC/TEMPLATE.md` (Abstract · Status/Terminologia · Motivação ·
> Terminologia · Especificação · Segurança · Privacidade · Registros IANA-like ·
> Compatibilidade · Referências · Apêndices). Este arquivo **DEVE** ser mantido
> sincronizado com `../_DESIGN_BRIEF.md` §11 e com `../003-RFC/README.md`.

> **Regra de não-redefinição.** Contratos centrais já especificados pela
> RFC-0001 (URN `urn:aios:<tenant>:<tipo>:<id>`, envelope de evento CloudEvents,
> subjects `aios.<tenant>.<dominio>.<entidade>.<acao>`, envelope de erro RFC 7807
> `AIOS-<DOMINIO>-<NNNN>`, `Idempotency-Key`, cabeçalhos `traceparent`/
> `X-AIOS-Tenant`) **NÃO SÃO** redefinidos por nenhuma RFC deste módulo — apenas
> **especializados** para o domínio de syscalls cognitivas e ACB.

---

## 1. RFC Consumida (Baseline)

| RFC | Título | Status | Relação com o Kernel |
|-----|--------|--------|------------------------|
| [RFC-0001](../003-RFC/RFC-0001-Architecture-Baseline.md) | AIOS Architecture Baseline & Core Contracts | Accepted | **Baseline consumida, não redefinida.** O Kernel herda integralmente: formato de URN (§5.1), envelope de evento CloudEvents (§5.2), convenção de subjects (§5.3), envelope de erro RFC 7807 (§5.4), semântica de idempotência (§5.5), cabeçalhos de correlação obrigatórios (§5.6) e regras de versionamento de API (§5.7). Toda RFC específica do Kernel (§2 abaixo) DEVE citar a seção pertinente da RFC-0001 em vez de reespecificá-la. |

### 1.1 Mapeamento de uso da RFC-0001 pelo Kernel

| Seção da RFC-0001 | Contrato | Uso concreto no Kernel |
|--------------------|----------|--------------------------|
| §5.1 URN | `urn:aios:<tenant>:<tipo>:<id>` | Campo `urn` da tabela `kernel.acb` (`urn:aios:<tenant>:agent:<ULID>`); todo `agent_urn` em `kernel.syscall_log`. |
| §5.2 Envelope de Evento (CloudEvents) | Estrutura `{id, source, type, time, datacontenttype, subject, tenant, data, traceparent}` | Payload de `kernel.outbox.payload` para todos os 12 tipos de evento do §6 do `_DESIGN_BRIEF.md`. |
| §5.3 Convenção de Subjects | `aios.<tenant>.<dominio>.<entidade>.<acao>` | Domínio `agent`; entidades `lifecycle`, `checkpoint`, `quota`, `syscall` (ex.: `aios.<tenant>.agent.lifecycle.spawned`). |
| §5.4 Envelope de Erro (RFC 7807) | `type/title/status/detail/instance/code/retriable` | Toda resposta de erro do Kernel (`API.md` §catálogo), com `code` nos domínios `KERNEL`/`CAP`/`QUOTA`/`SYSCALL`. |
| §5.5 Idempotência | `Idempotency-Key`, retenção ≥ 24h | `IdempotencyStore`; retenção configurável via `kernel.idempotency.retention_h` (default 48h). |
| §5.6 Correlação | `traceparent`, `X-AIOS-Tenant` | Extraídos pelo `SyscallGateway` em toda requisição; propagados a todo evento e span OTel emitido pelo `KernelTelemetry`. |
| §5.7 Versionamento | Versionamento semântico de API | Pacote gRPC `aios.kernel.v1`; base REST `/v1/kernel`. |

---

## 2. RFCs Propostas por Este Módulo

> **Estado de redação.** As RFCs abaixo estão **planejadas** e seu conteúdo
> pretendido está descrito no `../_DESIGN_BRIEF.md` (especialmente §5 ABI de
> syscalls e §3–§4 modelo de dados/FSM do ACB), que é a fonte única de verdade
> até que cada RFC seja redigida como arquivo próprio em `../003-RFC/` no
> formato IETF completo e promovida de `Draft` para `Accepted` conforme o ciclo
> de vida `Draft → Discussion → Last Call → Accepted`. Nenhum destes arquivos
> existe fisicamente em `../003-RFC/` no momento desta versão do índice.

| RFC | Título | Status | Escopo pretendido |
|-----|--------|--------|---------------------|
| RFC-0006 | Cognitive Syscall ABI | Planned (Draft a redigir) | Especificação normativa completa dos **11 verbos de syscall cognitiva** (`spawn`, `kill`, `suspend`, `resume`, `remember`, `recall`, `plan`, `invoke_tool`, `route_model`, `get_quota`, `checkpoint`): assinatura REST + gRPC (`aios.kernel.v1`), semântica de cada verbo, matriz de idempotência, catálogo de erros associado, regras de compatibilidade retroativa da ABI entre versões maiores. |
| RFC-0007 | ACB & Resource Quota Model | Planned (Draft a redigir) | Especificação normativa do **schema canônico do ACB** (campos, invariantes I1–I4), da **máquina de estados** (`AcbState`, transições T-01..T-13, guardas), e do **modelo de cotas** (dimensões, janelas, modos `hard`/`soft`, semântica de reserva/consumo/liberação atômica). |

### 2.1 Justificativa de cada RFC (por que é uma RFC e não uma seção da Architecture.md)

| RFC | Justificativa |
|-----|----------------|
| RFC-0006 | A ABI de syscalls é um **contrato de protocolo** consumido por múltiplos clientes externos ao módulo (`007-Agent-Runtime`, SDKs de terceiros, `020-Communication`/MCP-A2A bridges) — RFCs especificam protocolos inteiros com processo de comentário público (`Draft → Discussion → Last Call`), diferente de uma decisão pontual de ADR. |
| RFC-0007 | O schema do ACB e sua FSM são consumidos por `009-Scheduler` (leitura de `priority`, `quota_ref`) e `008-Lifecycle` (leitura de `runtime_ref`, `checkpoint_ref`) — uma mudança de schema exige coordenação multi-módulo típica de processo de RFC, não de ADR isolada. |

### 2.2 Estrutura obrigatória que cada RFC deste módulo DEVE seguir

Conforme `../003-RFC/README.md`, ambas as RFCs propostas DEVEM conter, nesta ordem:

1. **Abstract** — resumo de 1 parágrafo do contrato especificado.
2. **Status & terminologia** (RFC 2119/8174) — palavras normativas.
3. **Motivação** — por que o contrato existe (referenciar `_DESIGN_BRIEF.md` §1).
4. **Terminologia** — reuso de `../040-Glossary/Glossary.md`, sem redefinir termos.
5. **Especificação (normativa)** — o contrato em si (ABI ou schema/FSM).
6. **Considerações de segurança** — referenciar `Security.md` deste módulo.
7. **Considerações de privacidade (LGPD/GDPR)** — referenciar `_DESIGN_BRIEF.md` §12.3.
8. **Considerações de registro (IANA-like)** — reserva de domínios de erro, verbos, subjects.
9. **Compatibilidade e versionamento** — regras de evolução sem quebra.
10. **Referências** — normativas (RFC-0001) e informativas (ADRs do §2 de `ADR.md`).
11. **Apêndices** — exemplos de request/response, payloads de evento.

---

## 3. Rastreabilidade RFC → Componente → Requisito

| RFC | Componente(s) primário(s) (ver `Architecture.md` §2) | Requisito(s) relacionado(s) | ADR(s) de suporte |
|-----|---------------------------------------------------------|--------------------------------|----------------------|
| RFC-0001 (baseline) | `SyscallGateway`, `EventEmitter`, `IdempotencyStore`, `KernelTelemetry` | FR-006, FR-007, FR-012, NFR-011, NFR-012 | ADR-0004, ADR-0010 |
| RFC-0006 (proposta) | `SyscallGateway`, `ResourceBrokerRouter` | FR-001, FR-008 | ADR-0060, ADR-0068 |
| RFC-0007 (proposta) | `AcbStore`, `ResourceQuotaManager`, `LifecycleCoordinator` | FR-003, FR-004, FR-005, FR-010 | ADR-0061, ADR-0062, ADR-0064 |

---

## 4. Impacto de Mudanças Futuras na RFC-0001 sobre o Kernel

| Cenário de mudança na RFC-0001 | Impacto esperado no Kernel | Mitigação |
|----------------------------------|------------------------------|-----------|
| Alteração do formato de URN (§5.1) | Migração de `kernel.acb.urn` e de todo `agent_urn` em `syscall_log`/`outbox`. | Migração versionada com dupla-leitura (RFC-0001 §9); ADR de suporte antes de aplicar. |
| Alteração do envelope de evento CloudEvents (§5.2) | Todo payload de `kernel.outbox` precisa ser revalidado contra o novo schema. | Versionamento de schema de evento (`Events.md` §versionamento); consumidores toleram campos aditivos. |
| Alteração da semântica de idempotência (§5.5) | `IdempotencyStore` e retenção (`kernel.idempotency.retention_h`) podem precisar de ajuste de janela. | Retenção configurável já ≥ mínimo exigido; hot-reload de config. |
| Novo cabeçalho de correlação obrigatório (§5.6) | `SyscallGateway` precisa extrair e propagar o novo cabeçalho. | Extração de cabeçalhos é centralizada em um único componente (baixo *fan-out* de mudança). |

---

## 5. Processo de Promoção (Draft → Accepted)

1. O autor redige o arquivo `RFC-00NN-<slug>.md` em `../003-RFC/` seguindo a
   estrutura obrigatória da §2.2 e o `../003-RFC/TEMPLATE.md`.
2. A especificação **NÃO PODE** contradizer o `../_DESIGN_BRIEF.md` deste módulo
   nem redefinir contratos já centrais na RFC-0001; qualquer necessidade de
   divergência exige atualizar o brief primeiro.
3. Ciclo de comentário público interno: `Draft` → `Discussion` (revisão dos
   módulos consumidores listados na §3) → `Last Call` (janela final de
   objeções) → `Accepted`.
4. Ao ser aceita, o `Status` muda para `Accepted` no arquivo da RFC e neste
   índice; `../003-RFC/README.md` global é atualizado.
5. RFCs aceitas evoluem apenas por **RFC sucessora** que a torna
   `Obsoleted by RFC-XXXX` — nunca por edição retroativa do texto normativo.

---

## 6. Referências

- Índice global de RFCs: `../003-RFC/README.md`
- Template de RFC: `../003-RFC/TEMPLATE.md`
- Contrato baseline: `../003-RFC/RFC-0001-Architecture-Baseline.md`
- Fonte de verdade do módulo: `./_DESIGN_BRIEF.md` §5, §11
- Índice de ADRs do módulo: `./ADR.md`
- API do módulo: `./API.md`
- Eventos do módulo: `./Events.md`

*Fim do índice de RFCs do módulo 006-Kernel.*
