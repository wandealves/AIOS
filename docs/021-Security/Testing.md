---
Documento: Testing
Módulo: 021-Security
Status: Draft
Versão: 0.1
Última atualização: 2026-07-22
Responsável (RACI-A): Arquiteto do Módulo 021-Security
ADRs relacionados: ADR-0211, ADR-0212, ADR-0213, ADR-0215, ADR-0216, ADR-0217, ADR-0218, ADR-0219
RFCs relacionados: RFC-0001, RFC-0210, RFC-0211
Depende de: Requirements.md, FunctionalRequirements.md, NonFunctionalRequirements.md
---

# 021-Security — Estratégia de Testes

Os identificadores (`T-XXX-NN`) são os mesmos da matriz de rastreabilidade em
`./Requirements.md` §3.

> **Princípio desta estratégia:** em um módulo de segurança, o teste mais valioso não é
> o que confirma que a operação legítima funciona — é o que confirma que a **ilegítima
> falha**. Boa parte dos casos abaixo verifica ausências: que não existe caminho para
> reler material, para criar credencial eterna, para desrevogar ou para enumerar contas.

---

## 1. Pirâmide de Testes

```
                    ┌────────────────────────┐
                    │ Pentest / Red team     │  trimestral, externo
                    ├────────────────────────┤
                    │ Chaos / DR / cerimônia │  KMS fora, perda de CA, rotação
                    ├────────────────────────┤
                    │ Carga                  │  5k emissões/s, 20k introspecções/s
                    ├────────────────────────┤
                    │ Conformidade OIDC      │  suíte externa de certificação
                    ├────────────────────────┤
                    │ Integração (por PR)    │  PG + KMS simulado + PDP real
                    ├────────────────────────┤
                    │ Unidade (por commit)   │  FSM, TTL, suítes, não-oráculo
                    └────────────────────────┘
```

Testes de integração usam **PostgreSQL real** e **PDP real** (`../022-Policy/` em
contêiner). O KMS é simulado no CI e **real em staging** — a semântica de HSM (latência,
erros, indisponibilidade) não é reproduzível por *mock*.

---

## 2. Testes de Unidade

| ID | Alvo | Verifica |
|----|------|----------|
| T-UNIT-01 | FSM de credencial | Todas as transições válidas (T-01..T-09) e rejeição das inválidas (`AIOS-SEC-0005`). |
| T-UNIT-02 | Guarda de TTL | TTL acima do máximo do tipo é recusado; ausência de TTL usa o default, nunca "infinito". |
| T-UNIT-03 | Governança de suítes | Algoritmo fora de `allowed_suites` → `AIOS-SEC-0011`. |
| T-UNIT-04 | Ciclo de `KeyMaterial` | `retired` não vai a `destroyed` com artefato dependente. |
| T-UNIT-05 | Contenção por princípio | Limiar e janela corretos; contenção **não** vaza para outros princípios. |
| T-UNIT-06 | Escopo federado | `max_scope_expansion = 0` recusa qualquer escopo não listado. |
| T-UNIT-07 | Precedência de configuração | principal/kind > tenant > global > default. |

Cobertura-alvo: **≥ 90%** nos pacotes `Identity`, `Lifecycle` e `Keys` — acima do
padrão do projeto (85%), por serem caminhos de segurança.

---

## 3. Testes de Integração

| ID | Requisito | Cenário | Critério |
|----|-----------|---------|----------|
| T-OIDC-01 | FR-001 | Fluxo `authorization_code` + PKCE completo. | Token válido; `discovery` e `introspect` conformes. |
| T-OIDC-02 | FR-001, RN-06 | Autenticação com princípio inexistente × senha errada. | **Respostas idênticas** e tempo de resposta estatisticamente indistinguível. |
| T-CRED-01 | FR-003 | Emissão de certificado de workload com atestação válida. | `Active`; TTL ≤ 24 h; evento `credential.issued`. |
| T-CRED-02 | FR-004 | Emissão de `db_role` no PostgreSQL. | *Role* existe no banco alvo e expira no TTL. |
| T-CRED-03 | FR-004 | Emissão de `nats_account`. | Conta funciona no `../020-Communication/`. |
| T-CRED-04 | FR-007 | TTL de 30 dias para `access_token`. | `AIOS-SEC-0004`. |
| T-CRED-05 | FR-007 | Tentativa de gravar credencial sem `expires_at` (via camada de dados). | Rejeitado pela *constraint* `NOT NULL` — o teste ataca a camada abaixo da API. |
| T-ATT-01 | FR-003 | Atestação com serviço declarado ≠ observado. | `Failed`; `AIOS-SEC-0006`; evento `attestation.failed`. |
| T-ROT-01 | FR-005 | Rotação com janela de 300 s. | Durante a janela, **ambas** funcionam. |
| T-ROT-02 | FR-005 | Rotações concorrentes do mesmo `purpose`. | Nunca 3 utilizáveis (invariante I2); *advisory lock* serializa. |
| T-ROT-03 | FR-019 | Credencial a 20% do TTL sem rotação. | Evento `credential.expiring`. |
| T-REV-01 | FR-006 | Revogação simples. | `Revoked` + entrada em `revocation` + evento, na mesma transação. |
| T-REV-02 | FR-006, NFR-006 | Revogação com verificadores ativos. | Recusa observada em ≤ 30 s no `004` e no `020`. |
| T-REV-03 | FR-014 | Desabilitar princípio com 5 credenciais ativas. | Todas → `Revoked`; falha parcial reverte tudo. |
| T-REV-04 | RN-04 | Tentativa de reverter revogação. | `AIOS-SEC-0005`; estado inalterado. |
| T-SEC-01 | FR-008 | Escrita e leitura de segredo. | `ciphertext` no banco ≠ valor; leitura retorna o valor correto com lease. |
| T-SEC-02 | FR-008 | Leitura após o lease expirar. | `AIOS-SEC-0014`. |
| T-KEY-01 | FR-002 | Rotação de chave de assinatura. | JWKS com **2** chaves; tokens antigos e novos válidos. |
| T-KEY-02 | FR-009 | Rotação de KEK. | DEK recifradas; **dados não tocados**; segredos continuam legíveis. |
| T-PROF-01 | FR-010 | Publicação de perfil. | Assinatura verificável; evento emitido. |
| T-PROF-02 | FR-010 | Alteração de perfil publicado. | `AIOS-SEC-0012`. |
| T-FED-01 | FR-011 | Autenticação via IdP externo registrado. | Princípio interno mapeado com escopo limitado. |
| T-FED-02 | FR-011 | Token externo pedindo escopo não listado. | `AIOS-AUTHN-0007`. |
| T-FED-03 | FR-012 | Suspensão de confiança A2A. | `federation.suspended`; `020` encerra as sessões. |
| T-THR-01 | FR-013, NFR-014 | 10 falhas em 60 s para um princípio. | Contenção após a 5ª; **outros princípios não afetados**. |
| T-CRY-01 | FR-015 | Assinatura com suíte não permitida. | `AIOS-SEC-0011`. |
| T-PEP-01 | FR-016 | Operação administrativa sem capability. | `AIOS-SEC-0002`; decisão auditada. |
| T-PEP-02 | FR-016, RN-07 | PDP indisponível. | Operações administrativas negadas; **`/oauth2/token` e introspecção continuam**. |
| T-EVT-01 | FR-017 | Crash entre commit e publicação. | Relay reenvia; consumidor deduplica. |
| T-PRIV-01 | FR-020 | Pedido de RTBF. | Princípio desabilitado; `rtbf.requested` emitido; comprovante em `025`. |

---

## 4. Testes de Contrato

| ID | Contrato | Verifica |
|----|----------|----------|
| T-CTR-01 | JWKS ↔ `../004-API/` | O `004` valida tokens deste emissor; recarrega ao receber `jwks.rotated`; aceita tokens da chave anterior durante a sobreposição. |
| T-CTR-02 | `db_role` ↔ `../005-Database/` | Credencial emitida conecta e tem exatamente os privilégios do `purpose` — nem mais, nem menos. |
| T-CTR-03 | Perfil ↔ `../007-Agent-Runtime/` | O runtime **verifica a assinatura** e recusa perfil adulterado (NFR-015). |
| T-CTR-04 | `provider_key` ↔ `../017-Model-Router/` | Chave funciona no provedor e é rotacionada sem interromper inferência. |
| T-CTR-05 | `nats_account` e `token.revoked` ↔ `../020-Communication/` | Conta funciona; revogação encerra conexões e sessões A2A. |
| T-CTR-06 | Claims ↔ `../022-Policy/` | Atributos assertados são consumíveis pelo PDP no formato esperado. |
| T-CTR-07 | `rtbf.requested` ↔ `../010-Memory/`, `../005-Database/` | Evento dispara expurgo nos módulos donos. |
| T-CTR-08 | OpenAPI + proto | Compatibilidade retroativa; `IssuedCredential` × `Credential` mantidos distintos. |

T-CTR-08 verifica algo estrutural: que **não existe** operação de leitura retornando
material. A separação entre `IssuedCredential` (com material, só na emissão) e
`Credential` (metadados) é o que torna a invariante I5 verificável em tempo de
compilação para os consumidores.

---

## 5. Testes de Conformidade OIDC

| ID | Verifica |
|----|----------|
| T-CONF-01 | Suíte de certificação OIDC *Basic OP* e *Config OP*. |
| T-CONF-02 | PKCE obrigatório para clientes públicos; `code_challenge_method = S256`. |
| T-CONF-03 | Introspecção (RFC 7662) e revogação (RFC 7009) conformes. |
| T-CONF-04 | `discovery` com todos os campos obrigatórios; `jwks_uri` acessível. |

Conformidade importa porque clientes OIDC genéricos precisam funcionar sem adaptação —
foi essa a justificativa de manter os endpoints padronizados fora da convenção interna
do AIOS (`./API.md` §1).

---

## 6. Testes de Invariante (job contínuo, não apenas CI)

As consultas de `./Database.md` §12 rodam continuamente em produção. Qualquer linha
retornada é **incidente**, não relatório:

| ID | Invariante | Consulta |
|----|-----------|----------|
| T-INV-01 | I1 — nenhuma credencial sem expiração | `credentials_without_expiry = 0` |
| T-INV-02 | I2 — no máximo duas utilizáveis por `purpose` | `GROUP BY … HAVING count(*) > 2` |
| T-INV-03 | Chave `active` única por papel | `uq_key_active_per_role` (verificação de existência do índice) |
| T-INV-04 | Revogação sempre com motivo | `ck_reason_when_revoked` |
| T-INV-05 | Nenhuma configuração proibida ativa | `prohibited_config_active = 0` |

---

## 7. Testes de Contenção e Concorrência

| ID | Cenário | Critério |
|----|---------|----------|
| T-CONC-01 | 100 `IssueCredential` concorrentes com a mesma `Idempotency-Key`. | Uma única credencial; demais retornam a mesma (NFR-013). |
| T-CONC-02 | Rotação e revogação simultâneas da mesma credencial. | Serializadas por OCC; estado final consistente; sem credencial órfã. |
| T-CONC-03 | Força bruta contra um princípio sob carga normal de outros. | Contenção isolada; latência dos demais inalterada. |

---

## 8. Testes de Segurança (os mais importantes deste módulo)

| ID | Requisito | Cenário | Critério |
|----|-----------|---------|----------|
| T-LEAK-01 | FR-018, NFR-008 | Varredura automatizada de **respostas de API, logs, eventos e métricas** por padrões de material secreto (chave privada, JWT, senha, seed NKey). | **Zero** achados. Executa em CI **e** continuamente em produção. |
| T-LEAK-02 | FR-018 | Dump completo do schema `security` sem a KEK. | Nenhum segredo recuperável a partir do dump. |
| T-ENUM-01 | RN-06 | Tentativa de enumerar contas por mensagem e por tempo de resposta. | Respostas idênticas; diferença de tempo dentro do ruído estatístico. |
| T-ATT-02 | STRIDE-S | Contêiner tenta obter credencial de outro serviço. | `AIOS-SEC-0006`; evento e alerta P1. |
| T-PRIV-02 | STRIDE-E | Princípio com `sec:principal:write` tenta emitir credencial privilegiada a si mesmo. | Recusado por falta de `sec:credential:issue` (separação de funções). |
| T-TAMP-01 | STRIDE-T | Perfil de sandbox adulterado após publicação. | O `../007-Agent-Runtime/` recusa aplicar; alerta P1. |
| T-REPL-01 | STRIDE-R | Revogação sem motivo. | Rejeitada pela *constraint* `ck_reason_when_revoked`. |
| T-DOS-01 | STRIDE-D | Inundação de pedidos de emissão. | `AIOS-SEC-0013`; **validação de tokens existentes não é afetada**. |
| T-PENT-01 | — | Pentest externo trimestral com foco em identidade. | Sem achado de severidade alta ou crítica em aberto. |

T-LEAK-02 é particularmente valioso: ele prova que a decisão de cifragem envelopada
(ADR-0216) atinge seu objetivo — um backup do banco de segurança, se vazado, não
concede acesso a nada.

---

## 9. Testes de Carga

| ID | Requisito | Carga | Critério |
|----|-----------|-------|----------|
| T-LOAD-01 | NFR-002, NFR-004 | 5.000 emissões de token/s por 15 min. | p99 ≤ 50 ms; sem erro sustentado. |
| T-LOAD-02 | NFR-001, NFR-004 | 20.000 introspecções/s. | p99 ≤ 10 ms. |
| T-LOAD-03 | NFR-002 | 500 emissões de certificado/s (com atestação). | p99 ≤ 150 ms. |
| T-LOAD-04 | NFR-003 | 2.000 leituras de segredo/s. | p99 ≤ 20 ms. |
| T-SCALE-01 | NFR-007 | 10⁶ princípios, 10⁷ credenciais ativas. | Latência estável; expurgo por partição funcionando. |

---

## 10. Testes de Caos, DR e Cerimônias

| ID | Cenário | Critério |
|----|---------|----------|
| T-CHAOS-01 | KMS/HSM indisponível por 10 min. | Emissão suspensa (`AIOS-SEC-0010`); **validação e mTLS existentes continuam**; fila drena ao voltar. |
| T-CHAOS-02 | PDP indisponível. | Operações administrativas negadas; autenticação e introspecção continuam (T-PEP-02). |
| T-CHAOS-03 | PostgreSQL (`security`) indisponível. | Emissão e revogação param; **validação por JWKS no `004` continua**. |
| T-CHAOS-04 | Perda de 2 das 3 réplicas. | Serviço continua com 1 réplica; alerta P1; nenhuma credencial invalidada. |
| T-CHAOS-05 | Relógio de um nó com 10 min de desvio. | Validação recusa dentro da tolerância de `clock_skew_s` (60 s); alerta de NTP. |
| T-DR-01 | Restauração do schema `security` a partir de backup. | RTO ≤ 15 min; segredos legíveis **apenas** com a KEK. |
| T-CER-01 | **Cerimônia de rotação de KEK** (UC-010) ensaiada em staging. | DEK recifradas; dados intactos; aprovação dupla registrada. |
| T-CER-02 | **Cerimônia de recuperação da CA raiz** ensaiada em staging. | Nova intermediária emitida; *cross-signing* durante a transição; mTLS restabelecido. |

As duas cerimônias (T-CER-01, T-CER-02) **DEVEM** ser ensaiadas antes da primeira
execução em produção. O momento de descobrir um passo faltante no procedimento não é
durante um comprometimento de CA.

---

## 11. Ambientes e Gates

| Ambiente | Uso | Configuração |
|----------|-----|--------------|
| CI efêmero | unidade, integração, contrato, conformidade | PostgreSQL + PDP reais em contêiner; KMS simulado; CA de desenvolvimento. `attestation.required = true` com atestador simulado. |
| `staging` | carga, caos, cerimônias, pentest | HSM real (ou simulador fiel); topologia de produção. |
| `prod` | testes de invariante contínuos; DR drill | Somente verificação; nenhuma injeção de falha. |

**Gates de merge:**
1. Unidade + integração verdes; cobertura ≥ 90% nos pacotes de segurança.
2. **T-LEAK-01 verde** — zero achados de material sensível.
3. **T-ENUM-01 verde** — sem oráculo de enumeração.
4. Testes de contrato (T-CTR-01..08) verdes.
5. Conformidade OIDC verde.
6. Testes de invariante (T-INV-01..05) verdes.

Sem estes seis, o PR **NÃO DEVE** ser mesclado. Um PR que quebre T-LEAK-01 ou T-ENUM-01
é tratado como incidente de segurança, não como build vermelho.

---

## 12. Referências

- Rastreabilidade: `./Requirements.md` §3
- Requisitos: `./FunctionalRequirements.md` · `./NonFunctionalRequirements.md`
- Carga e KPIs: `./Benchmark.md` · Falhas e cerimônias: `./FailureRecovery.md`
- Invariantes no banco: `./Database.md` §12
