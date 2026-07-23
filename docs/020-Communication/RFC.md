---
Documento: RFC
Módulo: 020-Communication
Status: Draft
Versão: 0.1
Última atualização: 2026-07-22
Responsável (RACI-A): Arquiteto do Módulo 020-Communication
ADRs relacionados: ADR-0004, ADR-0200, ADR-0201, ADR-0202, ADR-0207
RFCs relacionados: RFC-0001, RFC-0200, RFC-0201
Depende de: 003-RFC, _DESIGN_BRIEF.md §11
---

# 020-Communication — Índice de RFCs

RFCs são registradas em `../003-RFC/`. Este documento lista as que afetam o módulo e o
impacto de cada uma.

---

## 1. RFCs Consumidas (baseline)

| RFC | Título | Status | Impacto no 020-Communication |
|-----|--------|--------|------------------------------|
| **RFC-0001** | Architecture Baseline & Core Contracts | Accepted | **Baseline herdada, não redefinida.** É a RFC mais central para este módulo, porque **três** de seus contratos são exatamente o que o barramento aplica: o **envelope de evento** (§5.2), a **convenção de subjects** (§5.3) e a **idempotência/dedupe por `event.id`** (§5.5). Além deles: URN (§5.1) para identidades de sessão e entidades; envelope de erro (§5.4) para os domínios `BUS` e `A2A`; correlação (§5.6) propagada em toda mensagem; versionamento (§5.7) para depreciação de subject; PEP/PDP e auditoria (§5.8); mTLS e isolamento por tenant (§6); minimização (§7); registros IANA-like (§8), onde o domínio `comm` será inscrito. |

O módulo **não reescreve** nenhum desses contratos: ele é o ponto do sistema onde eles
deixam de ser texto e passam a ser **verificados em tempo de execução**.

---

## 2. RFCs a Propor por Este Módulo

| RFC | Título | Escopo normativo | Relação com RFC-0001 | Status |
|-----|--------|------------------|----------------------|--------|
| **RFC-0200** | AIOS Bus Contract | Registro de subjects (esquema, unicidade, produtor declarado, classe de tráfego); declaração de streams e consumidores (retenção, réplicas, ack, `max_deliver`, backoff, `max_ack_pending`); garantias de entrega e ordem; semântica de DLQ e replay; backpressure e cotas; contrato do relay de Outbox. | Aplica e detalha §5.2, §5.3 e §5.5; **não** os redefine. | Proposed |
| **RFC-0201** | A2A Profile for AIOS | Handshake, negociação de capacidades, FSM de sessão (`A2ASessionState`, T-01..T-10, invariantes I1..I5), formato do canal privado, perfil restrito para pares **externos** federados, encerramento e trilha. | Especializa o protocolo A2A aberto para o contexto do AIOS, sob o modelo de autorização de §5.8. | Proposed |

---

## 3. Registros Alimentados por Este Módulo

Conforme RFC-0001 §8, os registros centrais são mantidos por `../004-API/`. O 020
contribui com:

| Registro | Contribuição do 020 | Documento |
|----------|---------------------|-----------|
| Domínios de subject NATS | Novo domínio **`comm`** (ADR-0200). Além disso, o `comm.subject_registry` é a **implementação operacional** deste registro para todos os módulos: o que não está lá não é publicável. | `./Events.md` |
| Códigos de erro | Domínios **`BUS`** (0001–0099) e **`A2A`** (0001–0099). | `./API.md` §5 |
| Versões de `dataschema` | `comm.subject.registered/1`, `comm.subject.deprecated/1`, `comm.stream.created/1`, `comm.stream.retired/1`, `comm.account.provisioned/1`, `comm.cluster.degraded/1`, `comm.a2a.established/1`, `comm.a2a.rejected/1`, `comm.a2a.closed/1`, `comm.a2a.failed/1`, `comm.delivery.deadlettered/1`, `comm.delivery.replayed/1`, `comm.consumer.stalled/1`, `comm.quota.throttled/1`. | `./Events.md` §4 |
| Tipos de recurso URN | `subjectdef`, `a2asession`, `groupchannel` (sob avaliação da arquitetura-chefe: a RFC-0001 §5.1 enumera os tipos vigentes). | `./Database.md` |

> **Ponto que exige ratificação.** Os três tipos de URN acima são **novos** frente à
> enumeração da RFC-0001 §5.1 (`agent`, `task`, `memory`, `tool`, `model`, `plan`,
> `workflow`, `policy`, `event`) e **DEVEM** ser registrados via PR + ADR-0200 antes de o
> módulo ser promovido a `Stable`. O mesmo vale para o domínio de subject `comm` —
> observe que o `005-Database` levantou pendência equivalente para `database`, o que
> sugere que a arquitetura-chefe deveria decidir os dois na mesma revisão da RFC-0001.

---

## 4. Relação com RFCs de Outros Módulos

| RFC | Módulo | Interação |
|-----|--------|-----------|
| RFC-0050 (Data Platform Contract) | `../005-Database/` | Define o contrato físico da tabela `outbox`, que o `PublishGateway` consome como fonte de publicação, e as convenções do schema `comm`. |
| RFC-0006 (Cognitive Syscall ABI) | `../006-Kernel/` | Os eventos de ciclo de vida emitidos pelo Kernel são os principais fluxos transportados; seus subjects são registrados aqui. |
| RFC-0070 (Runtime↔Supervisor Control Protocol) | `../007-Agent-Runtime/` | Agentes acessam o barramento através do runtime; as cotas por agente aplicam-se a esse caminho. |

---

## 5. Processo

1. Uma especificação de contrato/protocolo **NÃO DEVE** ser criada dentro dos 26
   documentos: ela vira RFC em `../003-RFC/` e é referenciada aqui.
2. RFCs deste módulo usam a numeração alinhada à faixa de ADR (`RFC-0200`, `RFC-0201`, …).
3. Alteração incompatível em RFC publicada exige nova RFC que a *obsolete*, com período
   de coexistência (RFC-0001 §9).
4. Ao mudar o status de uma RFC, atualize esta tabela e o índice em `../003-RFC/`.

---

## 6. Referências

- Baseline: `../003-RFC/RFC-0001-Architecture-Baseline.md`
- Decisões: `./ADR.md` · `../002-ADR/README.md`
- Registros centrais: `../004-API/`
- Brief: `./_DESIGN_BRIEF.md` §11
