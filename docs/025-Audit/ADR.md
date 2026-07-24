---
Documento: ADR
Módulo: 025-Audit
Status: Draft
Versão: 0.1
Última atualização: 2026-07-23
Responsável (RACI-A): Arquiteto do Módulo 025-Audit
ADRs relacionados: ADR-0001, ADR-0002, ADR-0004, ADR-0005, ADR-0008, ADR-0010; ADR-0250..ADR-0259
RFCs relacionados: RFC-0001, RFC-0250, RFC-0251
Depende de: 002-ADR, _DESIGN_BRIEF.md §11
---

# 025-Audit — Índice de ADRs

Faixa reservada: **`ADR-0250`..`ADR-0259`** (regra `NNN × 10`). As ADRs são registradas
em `../002-ADR/` — este documento é o índice com o impacto de cada decisão sobre o
módulo.

---

## 1. ADRs globais herdadas

| ADR | Decisão | Impacto no 025-Audit | Status |
|-----|---------|----------------------|--------|
| [ADR-0001](../002-ADR/ADR-0001-Sistema-Operacional-nao-Framework.md) | AIOS é um SO, não um framework | O módulo é o *journal* com garantia de integridade do SO, não uma biblioteca de logging. | Accepted |
| [ADR-0002](../002-ADR/ADR-0002-Microservicos-Control-Data-Plane.md) | Microserviços + split control/data plane | Fatos chegam dos dois planos; a correlação por `trace_id` os costura. | Accepted |
| [ADR-0004](../002-ADR/ADR-0004-NATS-como-Barramento.md) | NATS como barramento primário | ~99% da ingestão chega por JetStream, com `ACK` só após durabilidade. | Accepted |
| [ADR-0005](../002-ADR/ADR-0005-PostgreSQL-pgvector-AGE.md) | PostgreSQL unificado | Schema `audit` com RLS; *trigger* de imutabilidade; particionamento mensal. | Accepted |
| [ADR-0008](../002-ADR/ADR-0008-Governanca-por-Politica-Default-Deny.md) | Governança por política, default deny | *"Toda decisão é auditada (025)"* — o 022 é a maior fonte de fatos; o 025 é PEP das próprias operações. | Accepted |
| **[ADR-0010](../002-ADR/ADR-0010-Observabilidade-Auditoria-por-Construcao.md)** | **Observabilidade e auditoria por construção** | **Decisão-mãe deste módulo.** Fixa a trilha imutável *event-sourced* em todo caminho. Todo o 025 é a materialização da metade "auditoria" desta ADR. | Accepted |

---

## 2. ADRs a propor por este módulo

| ADR | Título | Escopo da decisão | Documentos afetados | Status |
|-----|--------|-------------------|---------------------|--------|
| ADR-0250 | Domínio de erro `AUD` e registro das classes auditáveis (`record_class`) | Reserva de `AIOS-AUD-0001..0099`; registro de subjects e streams `AUDIT_*` conforme RFC-0001 §8. | `API.md`, `Events.md` | Proposed |
| ADR-0251 | Imutabilidade verificável: cadeia de hash + selo de Merkle assinado + WORM | A decisão estruturante: imutabilidade **verificável por terceiros**, não declarada. Adulterar exige comprometer três sistemas distintos. | `StateMachine.md`, `Database.md`, `Security.md`, `Vision.md` | Proposed |
| ADR-0252 | Cadeia particionada por tenant em vez de sequência global única | A decisão de escala dominante: uma cadeia única serializaria toda a escrita do tenant (~4.100 vs. ~52.000 registros/s). | `Scalability.md`, `Benchmark.md`, `Database.md` | Proposed |
| ADR-0253 | Apagamento criptográfico para conciliar imutabilidade e RTBF | Resolve a tensão que define o módulo: cifragem por titular + destruição de chave. O registro permanece; o conteúdo pessoal, não. | `Security.md`, `StateMachine.md`, `Database.md` | Proposed |
| ADR-0254 | *Legal hold* sobrepõe retenção e direito ao esquecimento; *hold* pode não ter prazo | Precedência jurídica explícita (invariante I5) e a única exceção do AIOS à regra de prazo obrigatório. | `StateMachine.md`, `Database.md`, `Events.md` | Proposed |
| ADR-0255 | Confirmação apenas após durabilidade (RPO 0) e `503` como resposta correta | A decisão de disponibilidade dominante: recusar é preferível a aceitar sem garantir. Assimétrica em relação ao `../024-Observability/`, que é fail-open. | `API.md`, `FailureRecovery.md`, `Configuration.md` | Proposed |
| ADR-0256 | Completude declarada: classes auditáveis com produtor e volume esperados | Torna a **ausência** detectável — a lacuna é o defeito que não deixa rastro. | `Database.md`, `Monitoring.md`, `UseCases.md` | Proposed |
| ADR-0257 | Consulta à trilha é fato auditável | Sem isso, ler a trilha seria a única ação privilegiada não auditada. | `API.md`, `Logging.md`, `Security.md` | Proposed |
| ADR-0258 | Correção por registro de compensação, nunca por alteração (append-only) | Uma capability de alteração é uma capability de apagar evidência; o modelo a torna inexistente. | `ClassDiagrams.md`, `Database.md`, `API.md` | Proposed |
| ADR-0259 | Pacote probatório verificável offline por auditor externo | A premissa é que o auditor **não confia** no operador do AIOS. | `API.md`, `Testing.md`, `Examples.md` | Proposed |

---

## 3. Decisões que este módulo **não** toma

| Assunto | Onde a decisão vive |
|---------|---------------------|
| Quais métricas e alertas existem | `../024-Observability/` |
| Modelo de autorização (papéis, regras, capabilities) | `../022-Policy/` |
| Identidade, custódia de chaves e assinatura | `../021-Security/` |
| Classificação (`data_class`) do conteúdo auditado | Módulo dono do dado |
| Se um *legal hold* é devido; interpretação de base legal | Jurídico/DPO do tenant |
| Expurgo do dado de negócio nos módulos donos | `../005-Database/`, `../010-Memory/`, módulo dono |
| Resposta operacional ao incidente | `../029-Operations/` |
| Convenções físicas de banco (CV-01..CV-12) | `../005-Database/` |

O 025 **aplica** o *hold* que o jurídico decidiu, **registra** a decisão que o 022
tomou, **usa** a chave que o 021 custodia. Manter essa disciplina é o que evita que o
módulo de auditoria vire árbitro de questões que não são dele — e é também o que
permite que a trilha seja imparcial em relação ao que registra.

---

## 4. Relação entre ADR-0246 e este módulo

A **ADR-0246** (fronteira telemetria × auditoria) é proposta pelo
`../024-Observability/`, não por este módulo — mas define metade da identidade do 025.
Ela fixa que telemetria é **amostrável e perecível** e auditoria **não pode ser nenhuma
das duas**, evitando duas fontes da verdade com regras de retenção divergentes.

Consequências diretas aqui: RPO 0 (NFR-006), retenção legal por classe (NFR-010),
imutabilidade verificável (NFR-007) e a proibição de usar a trilha como fonte de
métrica (NR-01).

---

## 5. Registro e evolução

- Novas ADRs deste módulo **DEVEM** usar a faixa `ADR-0250..ADR-0259` e ser
  registradas em `../002-ADR/README.md`.
- Esgotada a faixa, a extensão **DEVE** ser negociada com Arquitetura-Chefe — não se
  invade a faixa de outro módulo.
- ADRs não são editadas após `Accepted`: são **substituídas**
  (`Superseded by ADR-XXXX`), conforme `../002-ADR/README.md`.
- Uma decisão que contrarie a RFC-0001 ou o ADR-0010 **NÃO DEVE** ser tomada aqui:
  exige RFC que obsolete a baseline.
- **ADR-0251 e ADR-0255 têm peso especial:** substituí-las alteraria as garantias de
  integridade e durabilidade que sustentam o valor probatório de todo o histórico já
  registrado. Qualquer substituição precisa definir o tratamento dos registros
  anteriores.

*Fim de `ADR.md`.*
