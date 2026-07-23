---
Documento: Vision
Módulo: 021-Security
Status: Draft
Versão: 0.1
Última atualização: 2026-07-22
Responsável (RACI-A): Arquiteto do Módulo 021-Security
ADRs relacionados: ADR-0001, ADR-0002, ADR-0008, ADR-0010 (globais); ADR-0210, ADR-0211, ADR-0213, ADR-0217 (deste módulo, a propor)
RFCs relacionados: RFC-0001, RFC-0210, RFC-0211
Depende de: 000-Vision, 001-Architecture, 022-Policy, 025-Audit, 005-Database, 020-Communication
---

# 021-Security — Visão do Módulo

> "A segurança está para o AIOS assim como o chaveiro e o subsistema de credenciais
> estão para um sistema operacional endurecido: nenhum processo escolhe sua própria
> identidade, forja sua própria chave ou decide seu próprio isolamento." — analogia
> condutora deste módulo (ver `../040-Glossary/Glossary.md`, termos **mTLS**, **OIDC**,
> **PEP/PDP**, **Sandbox**, **Capability**).

---

## 1. Propósito do Módulo

O **021-Security** é a **raiz de confiança** do AIOS. Ele corresponde à camada
transversal `L-` do Modelo de Camadas de `../001-Architecture/Architecture.md` §6 e
concentra tudo aquilo que os demais módulos **não devem inventar por conta própria**:
identidade, credenciais, chaves, segredos e perfis de isolamento.

A fronteira do módulo é definida por três perguntas distintas que plataformas com
governança frequentemente confundem — e que aqui pertencem a três módulos diferentes:

| Pergunta | Módulo | Artefato |
|----------|--------|----------|
| **Quem é você?** e **com que credencial se prova?** | **021-Security** | Token, certificado, segredo, conta |
| **Você pode fazer isso?** | `../022-Policy/` | Decisão do PDP |
| **O que aconteceu?** | `../025-Audit/` | Trilha imutável |

Disso decorrem as quatro entregas do módulo:

1. **Identidade verificável** — emissor OIDC para humanos e serviços, PKI interna para
   o mTLS obrigatório (RFC-0001 §6), e atestação de workload antes de qualquer emissão.
2. **Credenciais de vida curta** — um ciclo de vida unificado (§4 do brief) para token,
   certificado, *role* de banco e conta NATS, com rotação por sobreposição e revogação
   propagada.
3. **Custódia de segredos e chaves** — cofre versionado com cifragem envelopada,
   KMS com hierarquia KEK/DEK e caminho para HSM.
4. **Isolamento como artefato** — perfis `seccomp`/`cgroups`/egress versionados e
   **assinados**, publicados aqui e aplicados pelo `../007-Agent-Runtime/`.

---

## 2. Problema que o Módulo Resolve

A Visão Global (`../000-Vision/Vision.md`) identifica a **governança fraca (P8)** como
limitação estrutural dos sistemas de agentes atuais. O 021 ataca quatro manifestações
concretas disso:

| Problema | Como o módulo endereça |
|----------|------------------------|
| **Credenciais eternas espalhadas** | Chaves de API em variáveis de ambiente, senhas de banco em manifestos, tokens sem expiração. Aqui, `expires_at` é `NOT NULL` por construção e o TTL mediano é ≤ 24 h — credencial de longa vida é dívida de segurança, e o esquema não permite contraí-la. |
| **Identidade implícita entre serviços** | "Quem está na rede interna é confiável" falha no primeiro contêiner comprometido. Aqui, todo serviço prova identidade por mTLS com certificado emitido **após atestação**, e um workload não consegue assumir a identidade de outro. |
| **Isolamento por configuração dispersa** | O endurecimento de um sandbox acaba dependendo de quem editou o YAML por último. Aqui, o perfil é artefato **assinado e versionado**, verificado pelo runtime antes de aplicar. |
| **Segurança acoplada à autorização** | Quando o mesmo componente decide *quem é* e *o que pode*, mudar uma regra de negócio exige tocar o sistema de identidade. A separação 021/022 permite evoluir política sem mexer na raiz de confiança. |

---

## 3. Escopo

### 3.1 Dentro do escopo (in scope)

- Provedor de identidade **OIDC** (humanos e serviços) e publicação de **JWKS**.
- **PKI interna**: CA raiz *offline*, intermediárias, certificados de workload.
- **Atestação de workload** antes da emissão de credencial de serviço.
- **Cofre de segredos** versionado com *lease* e cifragem envelopada.
- **Credenciais dinâmicas**: *roles* de PostgreSQL (`005`), contas NKey/JWT
  (`020`), chaves de MinIO e de provedores de LLM (`017`).
- **KMS**: hierarquia KEK/DEK, rotação, interface HSM.
- **Revogação** e propagação por evento e por lista consultável.
- **Perfis de isolamento** assinados, consumidos pelo `007`.
- **Federação** com IdP externo e âncoras de confiança para pares A2A externos (`020`).
- **Sinais** de anomalia de autenticação e serviços criptográficos padronizados.

### 3.2 Fora do escopo (out of scope)

Lista normativa completa em `_DESIGN_BRIEF.md` §1.3. Os pontos que mais geram confusão:

- **Decidir autorização** — é do `../022-Policy/`. O 021 asserta atributos; o PDP os
  interpreta.
- **Definir o modelo RBAC/ABAC** (quais papéis existem, o que concedem) — também do `022`.
- **Ser PEP de borda** para rotas de API — é do `../004-API/`.
- **Executar o sandbox** (criar namespaces, aplicar cgroups) — é do `../007-Agent-Runtime/`.
  O 021 publica o **perfil**.
- **Persistir a trilha imutável** — é do `../025-Audit/`.
- **Provisionar e operar contas NATS** — é do `../020-Communication/`; o 021 apenas
  **emite** a credencial.
- **Firewall, segmentação de rede e WAF** — são de `../028-Deployment/` e `../027-Cluster/`.
- **Bloquear um sujeito por comportamento** — o 021 sinaliza a anomalia; a decisão é
  política (`022`) ou operacional (`029`).

### 3.3 Fronteira em diagrama

```
   consumidores ──"quem sou eu?"──▶ 021-Security ──claims/certs/segredos──▶ 022-Policy
   (004,005,007,                     (raiz de                               (decide)
    017,020, todos)                   confiança)  ──fatos──▶ 025-Audit (prova)
```

---

## 4. Personas

| Persona | Necessidade | O que o módulo entrega |
|---------|-------------|------------------------|
| **Serviço do plano de controle** | Provar identidade a outro serviço sem senha compartilhada. | Certificado de workload de vida curta, emitido após atestação. |
| **Engenheiro de módulo** | Acessar PostgreSQL/NATS/MinIO sem embutir credencial no código. | Credencial dinâmica com TTL curto e rotação automática por sobreposição. |
| **Autor de agente** | Executar código de terceiros sem comprometer o host. | Perfil de sandbox assinado, aplicado pelo runtime. |
| **Operador de segurança** | Revogar acesso imediatamente ao suspeitar de comprometimento. | Revogação em qualquer estado, propagada em ≤ 30 s. |
| **Administrador do tenant** | Usar o IdP corporativo da organização. | Federação OIDC/SAML com escopo interno reduzido. |
| **Encarregado de dados (DPO)** | Atender pedido de esquecimento e provar controle de acesso. | Gatilho `security.rtbf.requested` + trilha de leitura de segredo. |
| **Auditor** | Verificar que nenhuma credencial é eterna. | `expires_at` obrigatório e métrica de TTL mediano. |

---

## 5. Princípios de Design

| # | Princípio | Consequência prática |
|---|-----------|----------------------|
| P-01 | **Credencial de longa vida é dívida.** | `expires_at` é `NOT NULL`; TTL acima do máximo do tipo é recusado (`AIOS-SEC-0004`). |
| P-02 | **Identidade se prova, não se declara.** | Atestação obrigatória antes de emitir credencial de workload (`AIOS-SEC-0006`). |
| P-03 | **O segredo nunca volta.** | Material privado não é relido após a emissão; guarda-se referência cifrada e `fingerprint`. Perdeu? Reemita. |
| P-04 | **Isolamento é artefato assinado.** | Perfil de sandbox versionado, assinado e imutável — não configuração editável em deploy. |
| P-05 | **Revogação é irreversível.** | `Revoked` não transita para nenhum estado; recuperar acesso exige nova emissão. |
| P-06 | **Erro de autenticação não é oráculo.** | A resposta não distingue "não existe" de "senha errada"; o motivo real vai só para auditoria. |
| P-07 | **Validar não pode depender de mim.** | Tokens são JWT assinados, validados **localmente** pelo `004` com JWKS — o 021 não fica no caminho quente. |
| P-08 | **Falhar fechado no controle, aberto na validação.** | Com o PDP fora, emissão e administração são negadas; **autenticação e validação continuam**. |
| P-09 | **Separação de funções.** | Quem administra princípios não administra chaves; raiz e KEK exigem aprovação dupla. |

Os princípios P-07 e P-08 merecem nota conjunta: eles existem porque o 021 é
dependência de *bootstrap* de todo o sistema. Um desenho em que cada requisição
consulta o serviço de segurança transforma-o simultaneamente em gargalo e em ponto
único de falha total.

---

## 6. Não-Objetivos Explícitos

1. **Não é o PDP.** Não avalia regras de negócio, papéis ou atributos para decidir
   acesso — isso é `../022-Policy/`.
2. **Não é um WAF nem um IDS de rede.** Detecta anomalias na camada de **identidade**;
   inspeção de tráfego pertence à infraestrutura.
3. **Não é um SIEM.** Emite sinais e fatos; correlação e retenção longa são de
   `../025-Audit/` e `../024-Observability/`.
4. **Não é um gerenciador de senhas de usuário final.** O foco é identidade de
   plataforma; credenciais de usuário final de aplicações do tenant são do tenant.
5. **Não cifra colunas de negócio.** Fornece as chaves; a decisão de cifrar e a
   classificação do dado são do módulo dono do schema.
6. **Não responde a incidentes.** Sinaliza e revoga sob comando; a resposta operacional
   é de `../029-Operations/`.

---

## 7. Relação com a Visão Global

O `../000-Vision/Vision.md` estabelece que **nenhuma ação privilegiada ocorre sem
autorização** e que o sistema deve ser auditável ponta a ponta. Essa promessa tem uma
pré-condição raramente explicitada: **autorizar exige saber quem está pedindo**. O 021
é onde essa pré-condição é satisfeita.

- **Governança** → identidade verificável + credenciais rastreáveis + PEP em toda
  operação administrativa (`Security.md`).
- **Escala** → validação local por JWKS, sem chamada por requisição (`Scalability.md`).
- **Recuperabilidade** → **RTO ≤ 15 min, RPO ≤ 5 min**, com CA raiz recuperável de
  custódia *offline* (`FailureRecovery.md`).

Nenhum contrato central é redefinido: mTLS interno e validação de token no Gateway vêm
de `../003-RFC/RFC-0001-Architecture-Baseline.md` §6; URN, envelope de evento e de erro,
idempotência e correlação vêm das §5.1–§5.6.

---

## 8. Critérios de Sucesso do Módulo

| # | Critério | Verificação |
|---|----------|-------------|
| S-01 | Zero credenciais sem expiração em produção; TTL mediano ≤ 24 h. | NFR-010, `Metrics.md` |
| S-02 | Zero ocorrências de material secreto em log, evento, métrica ou resposta. | NFR-008, `Testing.md` §8 |
| S-03 | Revogação efetiva em ≤ 30 s em todos os verificadores. | NFR-006, `Benchmark.md` |
| S-04 | 100% dos perfis de sandbox consumidos com assinatura verificada. | NFR-015, contrato com `../007-Agent-Runtime/` |
| S-05 | Disponibilidade ≥ 99,99% — o AIOS não sobe sem este módulo. | NFR-005, `Monitoring.md` |
| S-06 | Nenhuma emissão de credencial de workload sem atestação bem-sucedida. | FR-003, `Security.md` |

---

## 9. Referências

- Visão global: `../000-Vision/Vision.md`
- Arquitetura global: `../001-Architecture/Architecture.md`
- Contratos centrais: `../003-RFC/RFC-0001-Architecture-Baseline.md`
- Glossário: `../040-Glossary/Glossary.md`
- Fonte única de verdade deste módulo: `./_DESIGN_BRIEF.md`
- Arquitetura do módulo: `./Architecture.md` · Segurança: `./Security.md`
