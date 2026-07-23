---
Documento: Requirements
Módulo: 021-Security
Status: Draft
Versão: 0.1
Última atualização: 2026-07-22
Responsável (RACI-A): Arquiteto do Módulo 021-Security
ADRs relacionados: ADR-0008, ADR-0210, ADR-0211, ADR-0212, ADR-0213, ADR-0214, ADR-0215, ADR-0216, ADR-0217, ADR-0218, ADR-0219
RFCs relacionados: RFC-0001, RFC-0210, RFC-0211
Depende de: FunctionalRequirements.md, NonFunctionalRequirements.md, UseCases.md, Testing.md
---

# 021-Security — Índice de Requisitos e Rastreabilidade

Consolida os requisitos do módulo e estabelece a **matriz de rastreabilidade**
Requisito → Caso de Uso → Teste. Não introduz requisitos novos.

---

## 1. Stakeholders e Interesses

| Stakeholder | Interesse primário | Requisitos que o representam |
|-------------|--------------------|------------------------------|
| Serviço do plano de controle | Provar identidade sem segredo compartilhado. | FR-003, FR-004, NFR-002 |
| Engenheiro de módulo | Acessar recursos sem credencial embutida no código. | FR-004, FR-005, FR-008, NFR-003 |
| Autor de agente | Executar código de terceiros com isolamento garantido. | FR-010, NFR-015 |
| Operador de segurança | Revogar acesso imediatamente e provar controle. | FR-006, FR-014, NFR-006 |
| Administrador do tenant | Usar o IdP corporativo com escopo controlado. | FR-011, FR-012 |
| Encarregado de dados (DPO) | Atender pedido de esquecimento; rastrear acesso a segredo. | FR-020, NFR-012 |
| Auditor | Provar que nenhuma credencial é eterna e que tudo é rastreável. | FR-007, FR-016, NFR-010 |
| SRE / Operador de plataforma | Bootstrap confiável e recuperação de desastre. | NFR-005, NFR-009, NFR-011 |

---

## 2. Índice Consolidado

### 2.1 Requisitos Funcionais (20)

| Faixa | Tema | IDs |
|-------|------|-----|
| Identidade e autenticação | OIDC, JWKS, federação | FR-001, FR-002, FR-011 |
| Emissão de credencial | Certificados, credenciais dinâmicas, atestação | FR-003, FR-004, FR-007 |
| Ciclo de vida | Rotação, revogação, cascata | FR-005, FR-006, FR-014, FR-019 |
| Segredos e chaves | Cofre, KMS, criptografia | FR-008, FR-009, FR-015 |
| Isolamento | Perfis assinados de sandbox | FR-010 |
| Federação externa | IdP, âncoras A2A | FR-011, FR-012 |
| Detecção | Anomalias de autenticação | FR-013 |
| Governança e privacidade | PEP, eventos, não-vazamento, RTBF | FR-016, FR-017, FR-018, FR-020 |

### 2.2 Requisitos Não-Funcionais (15)

| Categoria | IDs |
|-----------|-----|
| Desempenho | NFR-001, NFR-002, NFR-003, NFR-004 |
| Disponibilidade e recuperação | NFR-005, NFR-009 |
| Segurança | NFR-006, NFR-008, NFR-010, NFR-011, NFR-014, NFR-015 |
| Escalabilidade | NFR-007 |
| Observabilidade | NFR-012 |
| Manutenibilidade | NFR-013 |

### 2.3 Casos de Uso (15)

`UC-001` a `UC-015`, especificados em `./UseCases.md`.

---

## 3. Matriz de Rastreabilidade — FR → UC → Teste

| FR | Caso(s) de uso | Caso(s) de teste (`./Testing.md`) | NFR associado |
|----|----------------|-----------------------------------|---------------|
| FR-001 | UC-001, UC-002 | T-OIDC-01, T-OIDC-02 | NFR-001 |
| FR-002 | UC-009 | T-KEY-01 (sobreposição de JWKS) | NFR-011 |
| FR-003 | UC-003 | T-CRED-01, T-ATT-01 | NFR-002 |
| FR-004 | UC-004 | T-CRED-02, T-CRED-03 | NFR-002, NFR-004 |
| FR-005 | UC-005 | T-ROT-01, T-ROT-02 (invariante I2) | NFR-013 |
| FR-006 | UC-006 | T-REV-01, T-REV-02 (propagação) | NFR-006 |
| FR-007 | UC-004 | T-CRED-04 (TTL excessivo), T-CRED-05 (sem expiração) | NFR-010 |
| FR-008 | UC-008 | T-SEC-01, T-SEC-02 (lease expirado) | NFR-003, NFR-008 |
| FR-009 | UC-010 | T-KEY-02 (rotação de KEK) | NFR-011 |
| FR-010 | UC-011 | T-PROF-01, T-PROF-02 (imutabilidade) | NFR-015 |
| FR-011 | UC-012 | T-FED-01, T-FED-02 (escopo reduzido) | NFR-008 |
| FR-012 | UC-013 | T-FED-03 (suspensão encerra A2A) | — |
| FR-013 | UC-014 | T-THR-01 (contenção por princípio) | NFR-014 |
| FR-014 | UC-007 | T-REV-03 (cascata) | NFR-006 |
| FR-015 | UC-008 | T-CRY-01 (suíte não permitida) | — |
| FR-016 | UC-003, UC-006, UC-010 | T-PEP-01 (default deny), T-PEP-02 (PDP fora) | NFR-005 |
| FR-017 | UC-006, UC-009 | T-EVT-01 (crash entre commit e publish) | NFR-012 |
| FR-018 | todos | T-LEAK-01 (varredura de resposta/log/evento/métrica) | NFR-008 |
| FR-019 | UC-005 | T-ROT-03 (alerta de expiração) | NFR-010 |
| FR-020 | UC-015 | T-PRIV-01 (gatilho RTBF) | NFR-012 |

---

## 4. Matriz NFR → Método de Verificação

| NFR | Método | Documento |
|-----|--------|-----------|
| NFR-001, NFR-002, NFR-003 | Medição de latência sob carga | `./Benchmark.md` §4 |
| NFR-004 | Rampa de throughput | `./Benchmark.md` §5 |
| NFR-005 | Uptime probe + *error budget* de 4,4 min | `./Monitoring.md` §4 |
| NFR-006 | Medição fim a fim de propagação de revogação | `./Benchmark.md` §6 |
| NFR-007 | Teste de escala (10⁶ princípios, 10⁷ credenciais) | `./Scalability.md` §7 |
| NFR-008 | Varredura automatizada anti-vazamento | `./Testing.md` §8 |
| NFR-009 | DR drill + cerimônia de recuperação de raiz | `./FailureRecovery.md` §5 |
| NFR-010 | Métrica contínua de TTL e de credenciais sem expiração | `./Metrics.md` |
| NFR-011 | Idade de chave comparada à política | `./Monitoring.md` §3 |
| NFR-012 | Cobertura de spans + ausência de material sensível | `./Logging.md` §5 |
| NFR-013 | Teste de replay com `Idempotency-Key` | `./Testing.md` §4 |
| NFR-014 | Cenário de força bruta com contenção por princípio | `./Testing.md` §7 |
| NFR-015 | Teste de contrato com `../007-Agent-Runtime/` | `./Testing.md` §5 |

---

## 5. Requisitos Herdados (não redefinidos aqui)

| Origem | Requisito herdado | Aplicação no módulo |
|--------|-------------------|---------------------|
| **RFC-0001 §6** | **mTLS obrigatório entre serviços; tokens validados no Gateway; `tenant` como fronteira de isolamento; erros sem dados sensíveis** | **Contrato central deste módulo**: o 021 é quem emite o material que torna essas exigências executáveis. |
| RFC-0001 §5.1 | URN com ULID | `principal`, `credential`, `secret`, `key`, `sandboxprofile` |
| RFC-0001 §5.2–§5.3 | Envelope CloudEvents e subjects | `./Events.md` (domínio `security`) |
| RFC-0001 §5.4 | Envelope de erro `AIOS-<DOMINIO>-<NNNN>` | Domínios `SEC` e `AUTHN` |
| RFC-0001 §5.5 | Idempotência | Toda emissão, rotação e revogação |
| RFC-0001 §5.6 | Cabeçalhos de correlação | `./Logging.md` |
| RFC-0001 §5.8 | PEP/PDP com *default deny*; auditoria de ação privilegiada | FR-016 |
| RFC-0001 §7 | Minimização e retenção de dados pessoais | `./Security.md` §8 |
| `../000-Vision/Vision.md` V-R10 | RTO ≤ 15 min, RPO ≤ 5 min | NFR-009 |

---

## 6. Contratos com Módulos Consumidores

Este módulo é consumido por vários outros, e esses acoplamentos são requisitos
verificáveis por teste de contrato (T-CTR-* em `./Testing.md`):

| Consumidor | O que consome | Verificação |
|------------|---------------|-------------|
| `../004-API/` | JWKS para validação local de JWT; evento `jwks.rotated` | T-CTR-01 |
| `../005-Database/` | Credencial dinâmica de *role* PostgreSQL | T-CTR-02 |
| `../007-Agent-Runtime/` | Perfis de sandbox assinados; segredos escopados por tenant | T-CTR-03 |
| `../017-Model-Router/` | Chaves de provedores de LLM | T-CTR-04 |
| `../020-Communication/` | Contas NKey/JWT do NATS; evento `token.revoked`; âncoras A2A | T-CTR-05 |
| `../022-Policy/` | Claims e atributos assertáveis do princípio | T-CTR-06 |
| `../010-Memory/`, `../005-Database/` | Evento `security.rtbf.requested` | T-CTR-07 |

---

## 7. Glossário Local

Este módulo **não define termos próprios**. *mTLS*, *OIDC*, *PEP/PDP*, *RBAC*, *ABAC*,
*Sandbox*, *Capability* e *Tenant* são os do `../040-Glossary/Glossary.md`. Termos
técnicos de criptografia e PKI (KEK, DEK, JWKS, CRL, atestação, HSM, PKCE) são usados
com o significado padrão das especificações correspondentes (RFC 7517, RFC 7636,
RFC 7662, RFC 7009) e estão contextualizados em `./Security.md` e `./Database.md`.

---

## 8. Referências

- Requisitos funcionais: `./FunctionalRequirements.md`
- Requisitos não-funcionais: `./NonFunctionalRequirements.md`
- Casos de uso: `./UseCases.md` · Testes: `./Testing.md`
- Brief: `./_DESIGN_BRIEF.md` · Contratos: `../003-RFC/RFC-0001-Architecture-Baseline.md`
