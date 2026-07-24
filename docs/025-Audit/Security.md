---
Documento: Security
Módulo: 025-Audit
Status: Draft
Versão: 0.1
Última atualização: 2026-07-23
Responsável (RACI-A): Arquiteto do Módulo 025-Audit + CISO
ADRs relacionados: ADR-0008, ADR-0010, ADR-0251, ADR-0253, ADR-0254, ADR-0257, ADR-0258, ADR-0259
RFCs relacionados: RFC-0001, RFC-0250, RFC-0251
Depende de: _DESIGN_BRIEF.md §12, 021-Security, 022-Policy, 005-Database
---

# 025-Audit — Segurança

> A trilha de auditoria é o alvo de maior valor do AIOS para um **atacante interno**:
> quem consegue alterá-la apaga a evidência de tudo o que fez. O modelo de segurança
> deste módulo parte de uma premissa desconfortável e deliberada — **o operador do
> banco não é confiável** — e é por isso que a prova não vive apenas no banco.

## 1. Superfície de ataque

| Superfície | Exposição | Controle primário |
|-----------|-----------|-------------------|
| Ingestão (eventos e API) | Todos os módulos produtores | mTLS + `record_class` registrada + dedupe |
| Acesso direto ao PostgreSQL | Operador de banco, DBA, backup | *Trigger* de imutabilidade + cadeia + selo assinado + WORM |
| Consulta da trilha | Investigador, auditor, DPO | PEP + escopo por tenant + **registro da consulta** |
| Governança (*hold*, retenção, apagamento) | Jurídico, DPO, plataforma | PEP + aprovação dupla + chaves não-recarregáveis |
| Exportação | Auditor externo (via responsável) | Aprovação dupla + escopo revisado + TTL do link |
| Chave de assinatura de selo | `../021-Security/` | **Não reside neste módulo** (invariante S3) |
| Chaves por titular | `../021-Security/` | Referência apenas; material no KMS |

## 2. AuthN / AuthZ

### 2.1 Autenticação

- **Produtores**: **mTLS** com certificado de workload emitido pelo
  `../021-Security/` após atestação (RFC-0001 §6).
- **Humanos**: OIDC validado no Gateway (`../004-API/`) com MFA obrigatório para
  operações de governança.

### 2.2 Autorização

| Regra | Efeito |
|-------|--------|
| Ingestão exige `aud:record:append` | Produtor sem capability é recusado; a recusa é registrada |
| Consulta exige `aud:record:read` | Negação pelo PDP; **a negação também é auditada** |
| Governança exige capability específica por operação | `aud:hold:apply`, `aud:hold:release`, `aud:erasure:execute`, `aud:export:request`, `aud:class:manage` |
| **Nenhuma capability de alteração existe** | Não há `aud:record:update` no modelo (ADR-0258) |
| RLS por `tenant_id` | Segunda camada, independente do código |

### 2.3 Separação de funções

| Operação | Regra | Erro |
|----------|-------|------|
| Aplicar *legal hold* | `approved_by ≠ requested_by` (constraint `ck_hold_approver`) | `AIOS-AUD-0009` |
| Exportar trilha | `approved_by ≠ requested_by` (constraint `ck_export_approver`) | `AIOS-AUD-0009` |
| Aplicar e liberar o mesmo *hold* | **NÃO DEVERIAM** ser a mesma pessoa | Revisão periódica |
| Alterar registro | Ninguém — a operação não existe | `AIOS-AUD-0016` |
| Reduzir retenção ou *object lock* | Exige *deploy* revisado (chave não-recarregável) | — |

> As duas primeiras separações estão no **esquema do banco**, não apenas na aplicação
> (`./Database.md` §5 e §8). Um controle que vive só no código é desligável por um
> *deploy*; um `CHECK` constraint exige uma migração revisada — que, por sua vez, é
> auditada.

### 2.4 Fail-closed nas duas pontas

```
   ┌──────────────────────────────────────────────────────────────────┐
   │ INGESTÃO           fail-CLOSED  (aud.ingest.require_durable_ack) │
   │  · sem durabilidade ⇒ 503 AIOS-AUD-0005; o produtor reenvia      │
   │  racional: aceitar sem durabilidade = LACUNA INVISÍVEL           │
   └──────────────────────────────────────────────────────────────────┘
   ┌──────────────────────────────────────────────────────────────────┐
   │ CONSULTA/GOVERNANÇA fail-CLOSED (aud.policy.fail_mode = closed)  │
   │  · PDP indisponível ⇒ NEGADO                                     │
   │  racional: ninguém lê nem governa a trilha sem autorização       │
   └──────────────────────────────────────────────────────────────────┘
```

Este é o **único módulo do AIOS fail-closed nas duas pontas** — o oposto exato do
`../024-Observability/`, que é *fail-open* na ingestão. A diferença não é estilística:
telemetria perdida degrada visibilidade; **registro de auditoria perdido destrói
prova**, e de forma que ninguém percebe.

## 3. Defesa em profundidade da imutabilidade

```
   camada 1 — APLICAÇÃO
   └─ nenhuma operação de Update/Delete existe nas interfaces (S1)
        │ contornável por: acesso direto ao banco
        ▼
   camada 2 — BANCO
   └─ trigger tr_record_immutable rejeita UPDATE de campo e DELETE de linha
        │ contornável por: superusuário que remove o trigger
        ▼
   camada 3 — CADEIA DE HASH
   └─ prev_hash/record_hash: alteração local quebra tudo o que vem depois
        │ contornável por: recomputar toda a cadeia posterior
        ▼
   camada 4 — SELO ASSINADO
   └─ raiz de Merkle assinada com chave que NÃO está neste módulo
        │ contornável por: comprometer também o 021-Security
        ▼
   camada 5 — CÓPIA WORM
   └─ object lock no MinIO: sobrescrita rejeitada pelo armazenamento
        │ contornável por: comprometer também o armazenamento de objetos
        ▼
   Adulteração sem detecção exige comprometer TRÊS SISTEMAS DISTINTOS,
   com credenciais, planos de controle e operadores diferentes.
```

Cada camada isolada é contornável; o valor está na composição. E a camada 3 tem uma
propriedade que as demais não têm: mesmo comprometida, ela **deixa rastro** — a
divergência de hash é detectável para sempre.

## 4. Integridade e cadeia de custódia

| Controle | Como |
|----------|------|
| Encadeamento | `prev_hash` = `record_hash` do anterior (invariante I2). |
| Selagem | Raiz de Merkle assinada a cada `aud.seal.interval_s` (60 s). |
| Cadeia de selos | `prev_seal_hash` — remover um selo inteiro também é detectável. |
| Verificação contínua | Amostragem a cada 900 s; adulteração detectada em ≤ 1 ciclo (NFR-007). |
| Verificação completa | Compara com o **WORM** — a terceira fonte independente. |
| Serialização canônica | Normativa (RFC-0250): o auditor recomputa o **mesmo** `record_hash`. |
| Proveniência | `producer_module` conferido contra o `AuditableRegistry`; `actor` derivado das claims, não de campo livre. |

## 5. Proteção do conteúdo auditado

| Item | Regra |
|------|-------|
| Payload | Cifrado por titular (`contains_pii`) ou por tenant; nunca em claro no banco. |
| Eventos | Carregam hashes, intervalos e referências — **nunca** o conteúdo (`./Events.md` §1). |
| Métricas | Sem `subject_ref`, `resource_urn` ou `record_hash` como label (`./Metrics.md` §6). |
| Logs | Registram `record_class`, `seq` e códigos — jamais o payload (`./Logging.md` §6). |
| Exportação | Escopo revisado por aprovador distinto; conteúdo apagado aparece como `payload_digest` com explicação. |
| Consulta | Decifra apenas o que o solicitante pode ver; **a consulta é registrada**. |

## 6. STRIDE

| Ameaça | Vetor no 025-Audit | Mitigação | Verificação |
|--------|--------------------|-----------|-------------|
| **S**poofing | Serviço falso injeta registros forjados para criar álibi ou incriminar outro princípio. | mTLS com certificado atestado; `producer_module` conferido contra o `AuditableRegistry`; `actor` derivado das claims propagadas, não de campo livre do payload. | T-AUD-001; `AIOS-AUD-0003` monitorado. |
| **T**ampering | **A ameaça central.** Operador com acesso ao banco altera ou remove registros para apagar evidência. | As cinco camadas de §3: interface sem mutação, *trigger*, cadeia de hash, selo assinado fora do módulo, cópia WORM. | T-AUD-019; injeção controlada (T-AUD-107). |
| **R**epudiation | Negar ter aplicado um *hold*, exportado a trilha ou executado um apagamento. | `requested_by`/`approved_by`/`executed_by` obrigatórios; **toda consulta é auditada** (ADR-0257); comprovantes com hash reproduzível e registro na própria trilha. | T-AUD-016; auditoria cruzada. |
| **I**nformation disclosure | Consulta de auditoria como porta lateral para dados que a API de negócio não exporia; exportação com escopo excessivo. | Payload cifrado por titular; consulta sob PEP com capability; exportação com aprovação dupla e escopo revisado; conteúdo nunca em evento/métrica/log (FR-023). | T-AUD-023; T-AUD-115. |
| **D**enial of service | Inundação de registros por um produtor; consulta ou exportação varrendo 10⁹ registros. | Limite por tenant; orçamento de consulta (`AIOS-AUD-0013`) e exportação (`AIOS-AUD-0015`). **A recusa preserva a trilha** — melhor `503` que perda. | T-AUD-102; teste de carga. |
| **E**levation of privilege | Obter capability de apagamento e destruir evidência sob o disfarce de RTBF; reduzir retenção para expurgar prova. | *Legal hold* **sobrepõe** apagamento (I5, `AIOS-AUD-0008`); `aud.retention.min_days` e `aud.worm.object_lock_days` não-recarregáveis; todo apagamento gera comprovante que é **ele mesmo** um registro; `Shredded` preserva `payload_digest`. | T-AUD-010; T-AUD-110. |

### 6.1 A ameaça específica: apagamento como disfarce

O ataque mais plausível **não** é adulterar a cadeia — é **usar o RTBF legítimo como
pretexto** para destruir evidência de má conduta. Controles em camadas:

1. **`Legal hold` prevalece** (I5): sob investigação, nenhum apagamento ocorre, e a
   tentativa é registrada.
2. **`payload_digest` sobrevive** (I4): prova-se que **houve** conteúdo com aquele
   hash — o apagamento não pode ser confundido com "nunca existiu nada".
3. **Metadados sobrevivem**: `actor`, `action`, `resource_urn`, `outcome` e
   `occurred_at` permanecem. Apaga-se o **detalhe**, não o **fato**.
4. **Comprovante obrigatório**: quem apagou, quando, com que escopo — na própria trilha.
5. **PEP + capability específica**: `aud:erasure:execute` é separada de todas as
   demais.

Um adversário que apague tudo o que pode ainda deixa, na cadeia, o registro de que
agiu, sobre o quê e quando — e o registro do próprio apagamento.

## 7. LGPD / GDPR

| Item | Tratamento |
|------|-----------|
| **A tensão central** | Imutabilidade × direito ao esquecimento são incompatíveis se "esquecer" significar "remover o registro". Resolve-se com **apagamento criptográfico** (ADR-0253). |
| Cifragem por titular | Payload cifrado com chave individual quando a classe tem `contains_pii`. |
| Apagamento | Destruição da chave no `../021-Security/`; registro permanece `Shredded`, cadeia íntegra. |
| Prova de existência | `payload_digest` sobrevive: prova que **algo** aconteceu, sem revelar o quê. |
| Base legal | `legal_basis` obrigatória em **toda** classe (`./Database.md` §4). |
| *Legal hold* vs. RTBF | O *hold* **prevalece** — obrigação legal de preservação sobrepõe o pedido. O titular é informado, e a recusa é registrada com `case_ref`. |
| Comprovante | `receipt_hash` + `chain_record_id`; o titular recebe prova verificável. |
| Minimização | O payload contém o mínimo para provar a ação; conteúdo de negócio é do módulo dono (NR-10). |
| Segregação | RLS por tenant; partição lógica de cadeia por tenant; exportação nunca cruza a fronteira. |
| Transferência internacional | O WORM segue a região do tenant (`../027-Cluster/`); o pacote declara a jurisdição de origem. |

## 8. Controles e verificação contínua

| Controle | Frequência | Evidência |
|----------|-----------|-----------|
| Cadeia íntegra (amostragem) | A cada 15 min | `aios_aud_chain_break_total = 0` (NFR-007) |
| Verificação completa de uma partição | Mensal | Relatório de verificação |
| Comparação banco × WORM | Mensal (com a verificação completa) | Divergência = P1 |
| *Trigger* de imutabilidade presente | A cada *startup* e diariamente | `./Deployment.md` §4 |
| Revisão de *holds* ativos sem prazo | A cada 90 dias | `aud.hold.review_interval_days` |
| Revisão de capabilities de consulta e apagamento | Trimestral | `GetEffectivePermissions` no `../022-Policy/` |
| Nenhum conteúdo fora da trilha | Contínua | T-AUD-023, T-AUD-116 |
| Teste de restauração do pacote probatório | Semestral | Verificação offline por terceiro |
| Pentest da superfície de consulta e governança | Semestral | Relatório de `../029-Operations/` |

## 9. Referências

- Fonte única de verdade: `./_DESIGN_BRIEF.md` §12
- Identidade, chaves e assinatura: `../021-Security/`
- Autorização: `../022-Policy/` · Telemetria (fronteira): `../024-Observability/`
- Contratos: `../003-RFC/RFC-0001-Architecture-Baseline.md` §6 e §7
- Modelo físico: `./Database.md` · Alertas: `./Monitoring.md`

*Fim de `Security.md`.*
