---
Documento: RFC
Módulo: 004-API
Status: Draft
Versão: 0.1
Última atualização: 2026-07-21
Responsável (RACI-A): Arquiteto do Módulo 004-API
ADRs relacionados: ADR-0042, ADR-0043, ADR-0045
RFCs relacionados: RFC-0001, RFC-0040, RFC-0041
Depende de: ../003-RFC/README.md
---

# 004-API — Índice de RFCs

> RFCs relacionadas a este módulo, no ciclo de vida definido em
> `../003-RFC/README.md` (`Draft` → `Discussion` → `Last Call` → `Accepted`
> → `Obsoleted by RFC-XXXX`).

## 1. RFC consumida (baseline)

| RFC | Título | Status | Impacto no módulo |
|-----|--------|--------|-----------------------|
| [RFC-0001](../003-RFC/RFC-0001-Architecture-Baseline.md) | AIOS Architecture Baseline & Core Contracts | Accepted | **Baseline** integralmente consumida — este módulo **opera** (não redefine) URN, envelope de evento, envelope de erro, idempotência, correlação, versionamento e é o **guardião operacional** dos registros de §8 (tipos de recurso, domínios de subject, catálogo de erros, versões de `dataschema`). |

## 2. RFCs a propor por este módulo

| RFC | Título proposto | Status | Relação |
|-----|-------------------|--------|-----------|
| RFC-0040 | API Gateway Contract & Versioning Policy | A propor | Especifica formalmente as regras completas de `/vN`, header `X-AIOS-Api-Version`, coexistência mínima de majors, política de depreciação (janela ≥ 180 dias), regras de retirada e o formato do contrato agregado servido em `/v1/api/openapi.json`. Formaliza o comportamento já descrito em `./StateMachine.md` e `./API.md`. |
| RFC-0041 | Central Contract Registry Schema | A propor | Especifica formalmente o esquema dos registros de RFC-0001 §8 mantidos por este módulo: tipos de recurso URN (`api.resource_type`), domínios de subject NATS, catálogo de códigos de erro (`api.error_code`), versões de `dataschema` de evento (`api.event_schema`) — incluindo o processo de governança (PR + ADR/RFC de módulo) para novas entradas. Formaliza o comportamento já descrito em `./Database.md` e `./API.md` §1. |

## 3. Processo de evolução

Toda mudança **incompatível** ao contrato deste módulo (ex.: alterar o
formato do envelope de erro estendido, mudar a semântica de uma transição
da FSM de `ApiVersion`) **DEVE** passar por uma nova RFC que declare
`Obsoleted by RFC-XXXX` sobre a RFC anterior relevante, mantendo o período
de coexistência exigido por RFC-0001 §5.7/§9. Mudanças **aditivas** e
retrocompatíveis (nova operação de `RegistryService`, novo código de erro no
domínio `API`) **PODEM** ser incorporadas via atualização incremental de
`RFC-0040`/`RFC-0041` uma vez aceitas, sem exigir uma RFC nova.

*Fim de `RFC.md`.*
