---
Documento: ADR
Módulo: 021-Security
Status: Draft
Versão: 0.1
Última atualização: 2026-07-22
Responsável (RACI-A): Arquiteto do Módulo 021-Security
ADRs relacionados: ADR-0001, ADR-0002, ADR-0006, ADR-0008, ADR-0010; ADR-0210..ADR-0219
RFCs relacionados: RFC-0001, RFC-0210, RFC-0211
Depende de: 002-ADR, _DESIGN_BRIEF.md §11
---

# 021-Security — Índice de ADRs

Faixa reservada: **`ADR-0210`..`ADR-0219`** (regra `NNN × 10`). As ADRs são registradas
em `../002-ADR/` — este documento é o índice com o impacto de cada decisão.

---

## 1. ADRs Globais Herdadas

| ADR | Decisão | Impacto no 021-Security | Status |
|-----|---------|-------------------------|--------|
| ADR-0001 | AIOS é um SO, não um framework | O módulo é o subsistema de credenciais do SO, não uma biblioteca de autenticação. | Accepted |
| ADR-0002 | Microserviços + split control/data plane | Identidade de workload é pré-requisito do mTLS entre serviços. | Accepted |
| ADR-0006 | Redis para estado quente; padrão Outbox | Contadores de contenção em Redis; revogação e evento na mesma transação. | Accepted |
| **ADR-0008** | **Governança por política, *default deny*** | **Define a fronteira deste módulo**: o 021 é PEP de si mesmo; quem decide é o `../022-Policy/`. | Accepted |
| ADR-0010 | Observabilidade e auditoria por construção | Toda emissão, leitura de segredo e revogação é auditada. | Accepted |

---

## 2. ADRs a Propor por Este Módulo

| ADR | Título | Escopo da decisão | Documentos afetados | Status |
|-----|--------|-------------------|---------------------|--------|
| ADR-0210 | Fronteira identidade × autorização (021 × 022) e registro do domínio de eventos `security` | Quem asserta atributos e quem os interpreta; registro do domínio `security` conforme RFC-0001 §8. | `Vision.md`, `Events.md`, `Database.md` | Proposed |
| ADR-0211 | JWT assinado com validação local por JWKS vs. introspecção centralizada | A decisão dominante do módulo: mantê-lo fora do caminho quente, aceitando revogação não instantânea. | `Architecture.md`, `Scalability.md`, `Benchmark.md` | Proposed |
| ADR-0212 | Modelo unificado de credencial | Um ciclo de vida para token, certificado, *role* e conta — em vez de quatro subsistemas divergentes. | `StateMachine.md`, `Database.md`, `ClassDiagrams.md` | Proposed |
| ADR-0213 | Credenciais de vida curta com rotação por sobreposição | Proibição estrutural de credencial sem expiração; janela de sobreposição em vez de coordenação perfeita. | `StateMachine.md`, `Configuration.md`, `Metrics.md` | Proposed |
| ADR-0214 | PKI interna com raiz *offline* e certificados de workload de TTL ≤ 24 h | Hierarquia de CA; o que é recuperável e o que não deveria ser alcançável. | `Architecture.md`, `Deployment.md`, `FailureRecovery.md` | Proposed |
| ADR-0215 | Atestação de workload obrigatória antes da emissão | O controle central contra falsificação de identidade de serviço na rede interna. | `Security.md`, `UseCases.md`, `Configuration.md` | Proposed |
| ADR-0216 | Hierarquia KEK/DEK com cifragem envelopada e caminho para HSM | Rotação da raiz de cifragem sem recifrar o corpus de dados. | `Database.md`, `Benchmark.md`, `FailureRecovery.md` | Proposed |
| ADR-0217 | Perfis de sandbox como artefato assinado e versionado | Isolamento deixa de ser configuração dispersa e vira material verificável pelo `../007-Agent-Runtime/`. | `Database.md`, `API.md`, `Testing.md` | Proposed |
| ADR-0218 | Federação com escopo reduzido | Confiança externa não se converte em confiança interna (`max_scope_expansion = 0`). | `Security.md`, `UseCases.md`, `Configuration.md` | Proposed |
| ADR-0219 | Domínios de erro (`SEC`, `AUTHN`), mensagens não-oráculo e *fail-closed* seletivo | Reserva dos domínios; proibição de oráculo de enumeração; degradação que preserva a validação. | `API.md`, `Logging.md`, `FailureRecovery.md` | Proposed |

---

## 3. Decisões e Alternativas Descartadas (resumo)

| ADR | Escolha | Alternativas descartadas | Trade-off aceito |
|-----|---------|--------------------------|------------------|
| ADR-0211 | JWT + validação local | Token opaco + introspecção por requisição | **Revogação não instantânea** (≤ 30 s) — mitigada por TTL curto e cache de CRL. É o trade-off central do módulo. |
| ADR-0212 | Ciclo unificado | Um subsistema por tipo de credencial | Abstração precisa acomodar tipos heterogêneos (resolvido com `kind` + `purpose`). |
| ADR-0213 | TTL obrigatório e curto | Credenciais de longa vida com revogação | Mais emissões e renovações; carga contínua no HSM. |
| ADR-0214 | PKI própria com raiz offline | CA pública; CA embutida em malha de serviço | Operar PKI é responsabilidade séria — daí a cerimônia documentada e ensaiada. |
| ADR-0215 | Atestação obrigatória | Confiança na rede interna | Depende de atributos verificáveis do orquestrador (`027`/`028`); falha de atestação bloqueia deploy legítimo mal configurado. |
| ADR-0216 | Envelope KEK/DEK | Cifragem direta com uma chave | Uma indireção a mais na leitura de segredo (~3 ms de `unwrap`). |
| ADR-0217 | Perfis assinados | Configuração em manifesto de deploy | Publicar perfil vira operação governada, com capability e versão. |
| ADR-0218 | Escopo federado reduzido | Mapear papéis do IdP externo diretamente | Integração com IdP corporativo exige mapeamento explícito de escopos. |
| ADR-0219 | *Fail-closed* seletivo | *Fail-closed* total; *fail-open* | Operações administrativas param quando o PDP cai — aceito para preservar validação e autenticação. |

---

## 4. Relação com Decisões de Outros Módulos

| ADR de outro módulo | Interação |
|---------------------|-----------|
| ADR-0051 (RLS por `tenant_id`, `../005-Database/`) | O 021 emite as *roles* que operam sob RLS; a fronteira de tenant é a mesma nos dois módulos. |
| ADR-0201 (conta NATS por tenant, `../020-Communication/`) | O 021 emite as credenciais NKey/JWT dessas contas; revogá-las encerra o acesso ao barramento. |
| ADR-0207 (perfil A2A, `../020-Communication/`) | As âncoras de confiança de pares externos são registradas aqui (`federation_trust`). |
| ADR-0075 (MCP host no sandbox, `../007-Agent-Runtime/`) | O perfil de isolamento que encaixota o MCP host é publicado por este módulo. |

---

## 5. Processo

1. Uma decisão arquitetural do módulo **NÃO DEVE** ser tomada dentro de um dos 26
   documentos: ela é registrada como ADR em `../002-ADR/` e apenas **referenciada** aqui.
2. Novas ADRs usam a próxima numeração livre em `ADR-0210`..`ADR-0219`.
3. Uma ADR que altere contrato central exige também **RFC** (ver `./RFC.md`).
4. Decisões que afrouxem um controle de segurança (TTL maior, atestação opcional,
   escopo federado ampliado) **DEVEM** ser ADRs explícitas, com análise de risco — nunca
   ajustes de configuração sem registro.
5. Ao mudar o status de uma ADR, atualize esta tabela e `../002-ADR/README.md`.

O ponto 4 é específico deste módulo: em segurança, o afrouxamento silencioso é o modo de
falha mais comum, e ele acontece por configuração, não por decisão registrada.

---

## 6. Referências

- Índice global de decisões: `../002-ADR/README.md`
- Brief (origem das propostas): `./_DESIGN_BRIEF.md` §11
- RFCs do módulo: `./RFC.md`
