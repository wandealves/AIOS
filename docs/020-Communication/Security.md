---
Documento: Security
Módulo: 020-Communication
Status: Draft
Versão: 0.1
Última atualização: 2026-07-22
Responsável (RACI-A): Arquiteto do Módulo 020-Communication
ADRs relacionados: ADR-0008, ADR-0010 (globais); ADR-0200, ADR-0201, ADR-0202, ADR-0207, ADR-0209
RFCs relacionados: RFC-0001, RFC-0200, RFC-0201
Depende de: _DESIGN_BRIEF.md §12, 021-Security, 022-Policy, 025-Audit, 004-API
---

# 020-Communication — Segurança

Contratos centrais (mTLS interno, validação de token no Gateway, `tenant` como
fronteira de isolamento, minimização de dados) são normativos na
`../003-RFC/RFC-0001-Architecture-Baseline.md` §6–§7 e **não são redefinidos aqui**.

---

## 1. Autenticação (AuthN)

| Caminho | Mecanismo | Responsável |
|---------|-----------|-------------|
| Cliente NATS (módulo ou agente) → broker | **NKey/JWT** da conta do tenant + **TLS** | `021-Security` emite; `020` consome |
| Serviço → `aios-comm-svc` (gRPC) | **mTLS** na malha interna (RFC-0001 §6) | `021-Security` |
| Operador → API administrativa | OAuth2/OIDC validado no `../004-API/`; claims propagadas | `021-Security` + `004-API` |
| Par A2A **externo** → barramento | **mTLS** + JWT federado de emissor confiável | `021-Security` (validação), `020` (sessão) |

O **operator JWT** que assina as contas é a raiz de confiança do barramento e é
custodiado **offline** pelo `../021-Security/`. O `020` **não** emite credenciais: ele
solicita, aplica e revoga.

---

## 2. Autorização (AuthZ)

O módulo é **PEP**. Consultam o **PDP** do `../022-Policy/`, com ***default deny***
(RFC-0001 §5.8):

| Capability | Operações | Sensibilidade |
|------------|-----------|---------------|
| `bus:subject:read` / `bus:subject:write` | Ler / registrar / depreciar subject | baixa / média |
| `bus:stream:read` / `bus:stream:write` | Ler / declarar / drenar stream | baixa / **alta** |
| `bus:stream:replay` | Replay de stream | **alta** |
| `bus:consumer:read` / `bus:consumer:write` | Ler / declarar consumidor | baixa / média |
| `bus:dlq:read` / `bus:dlq:replay` / `bus:dlq:discard` | Inspecionar / reinjetar / descartar | média / **alta** / **alta** |
| `bus:group:write` | Criar canal de grupo | média |
| `bus:account:provision` | Provisionar conta de tenant | **alta** |
| `bus:account:export` | Exportar subject entre contas | **crítica** |
| `a2a:session:open` / `:close` / `:read` | Sessões A2A | **alta** / média / baixa |
| `bus:health:read` | Status do barramento | baixa |

Além disso, **cada capacidade negociada** em uma sessão A2A é submetida individualmente
ao PDP (T-04): a autorização é do conjunto exato, não de "abrir sessão" em abstrato.

**Comportamento sob PDP indisponível** (`bus.policy.fail_mode = closed`): operações
administrativas e **novas** sessões A2A são negadas (`AIOS-BUS-0002` /
`AIOS-A2A-0001`); o **tráfego já autorizado continua**. Um barramento que para de
entregar porque o serviço de política caiu transformaria uma falha localizada em queda
total do AIOS.

---

## 3. Isolamento Multi-Tenant

A **conta NATS por tenant** é a fronteira de isolamento e o invariante mais forte do
módulo (NFR-010, ADR-0201).

```
   ┌── conta acme ─────────────┐   ┌── conta globex ───────────┐
   │ subjects: aios.acme.>     │   │ subjects: aios.globex.>   │
   │ credencial: NKey/JWT acme │   │ credencial: NKey/JWT globex│
   └───────────────────────────┘   └───────────────────────────┘
              ▲                                ▲
              └── sem export/import declarado, o roteamento nem
                  considera os subjects da outra conta ──┘
```

Propriedades relevantes:

1. **Invisibilidade, não apenas proibição**: um cliente da conta `acme` que assine
   `aios.>` recebe apenas o tráfego de `acme` — a outra conta não existe do ponto de
   vista do roteamento. Isso é qualitativamente diferente de uma regra de permissão que
   pode ser mal escrita.
2. `tenant` no envelope **DEVE** casar com a conta autenticada, verificado pelo
   `EnvelopeValidator` (`AIOS-BUS-0003`).
3. Cruzar contas exige *export/import* **declarado, autorizado e auditado** (FR-019) —
   é exceção documentada, nunca configuração de rotina.
4. Limites por conta (conexões, assinaturas, payload, taxa) contêm o raio de dano de
   um tenant comprometido.
5. Sessões A2A entre tenants distintos **NÃO DEVEM** ser abertas sem export explícito;
   o `A2AGateway` verifica a compatibilidade de tenant já em T-01.

---

## 4. Autorização de Publicação (anti-falsificação)

Cada subject tem um `producer_module` declarado em `comm.subject_registry`. A
publicação por qualquer outro módulo é recusada com `AIOS-BUS-0010` e auditada.

Sem essa regra, qualquer serviço com credencial válida do tenant poderia publicar
`aios.acme.agent.lifecycle.terminated` e induzir o Kernel, o Scheduler e o Audit a agir
sobre um fato que nunca ocorreu. A verificação de **quem** publica é tão importante
quanto a de **o que** é publicado.

---

## 5. Superfície de Ataque

| Superfície | Exposição | Controle |
|------------|-----------|----------|
| Porta 4222 (clientes NATS) | Malha interna | TLS + NKey/JWT por conta + limites por conta |
| Porta 6222 (rotas de cluster) | Somente entre nós | TLS mútuo; rede segregada |
| Porta 7422 (leaf nodes) | Controlada por `027` | mTLS + filtros de propagação |
| Porta 8222 (monitoramento) | Interna | Sem exposição externa; raspada pelo Prometheus |
| API `/v1/comm` | Interna, via `004-API` | OIDC + PEP/PDP + auditoria |
| gRPC `aios.comm.v1` | Malha interna | mTLS + PEP |
| Canal A2A com par externo | Externa (quando habilitada) | mTLS + JWT federado + capacidades restritas ao perfil da RFC-0201 |
| Payload de mensagem | Interna | Validação de envelope, limite de tamanho, minimização obrigatória |

---

## 6. Threat Model — STRIDE

| Ameaça | Vetor no 020-Communication | Mitigação | Verificação |
|--------|-----------------------------|-----------|-------------|
| **S**poofing | Agente ou serviço publica evento fingindo ser outro módulo (ex.: forjar `policy.decision.allowed` para burlar governança). | `producer_module` declarado por subject (`AIOS-BUS-0010`); conta NATS por tenant; identidade de par A2A verificada por mTLS/JWT (`AIOS-A2A-0005`). | T-SEC-03, T-A2A-03 |
| **T**ampering | Alteração do envelope em trânsito; mudança de declaração de stream para desviar tráfego a um consumidor não autorizado. | TLS em todas as conexões; `EnvelopeValidator` no publish; declaração versionada (OCC) e alterável só via PEP; auditoria em `../025-Audit/`. | T-ENV-01, T-SEC-04 |
| **R**epudiation | Negar ter publicado uma mensagem ou aberto uma sessão A2A com determinado par. | `source` + `traceparent` no envelope; toda sessão registrada em `comm.a2a_session` com capacidades, volume e horários, auditada em `025`. | T-A2A-02 |
| **I**nformation disclosure | Assinar `aios.>` e ler tráfego de outros tenants; PII no payload ou na DLQ; export indevido entre contas. | Conta por tenant (a outra conta é **invisível** ao roteamento); curinga restrito ao escopo autorizado; minimização obrigatória (RFC-0001 §7); DLQ sob RLS com retenção de 30 dias; export exige `bus:account:export` e é auditado. | T-SEC-01, T-SEC-02 |
| **D**enial of service | Produtor inundando o cluster; difusão explosiva para 10⁴ agentes; *slow consumer* travando entrega; payload gigante; excesso de assinaturas. | Cotas por tenant/agente (`AIOS-BUS-0007`), `fanout_limit` (`AIOS-BUS-0008`), `max_ack_pending` (`AIOS-BUS-0014`), `max_payload_bytes` (`AIOS-BUS-0006`), `max_subscriptions_agent`. | T-FLOW-01, T-FLOW-02 |
| **E**levation of privilege | Criar *export* entre contas para exfiltrar dados; ampliar capacidades de uma sessão A2A já autorizada. | Export exige capability crítica e justificativa auditada; capacidades A2A autorizadas item a item e **imutáveis** após `Established` (invariante I4, `AIOS-A2A-0007`). | T-SEC-02, T-A2A-01 |

---

## 7. Criptografia

| Dado | Em trânsito | Em repouso |
|------|-------------|------------|
| Mensagens cliente ↔ broker | TLS obrigatório | Streams JetStream no volume (cifragem do volume, `../028-Deployment/`) |
| Rotas entre nós NATS | TLS mútuo | — |
| Leaf nodes / gateways | mTLS | — |
| Chamadas gRPC de controle | mTLS (RFC-0001 §6) | — |
| Declarações e DLQ (`comm.*`) | TLS ao PostgreSQL | Cifragem de volume + backups cifrados (`../005-Database/`) |
| Credenciais NKey/JWT | TLS | Cofre do `../021-Security/` |

Cifragem **fim a fim de payload** (o broker não conseguir ler a mensagem) **não** é
oferecida por default: ela quebraria a validação de envelope no publish, que é uma das
garantias centrais do módulo (ADR-0202). Quando um caso de uso exigir sigilo do
conteúdo perante a plataforma, o produtor **DEVE** cifrar apenas o campo `data`,
mantendo o envelope legível.

---

## 8. LGPD / GDPR

| Requisito legal | Implementação |
|-----------------|---------------|
| **Minimização** | Payloads carregam URNs opacos e referências, não conteúdo sensível (RFC-0001 §7). O barramento **NÃO DEVE** ser usado para trafegar dado pessoal além do necessário. |
| **Retenção** | `max_age` explícito por stream (default 7 dias); DLQ 30 dias; sessões A2A 90 dias. Retenção longa pertence a `../025-Audit/` e `../005-Database/`. |
| **Direito ao esquecimento** | Mensagens já publicadas **não são reescritas** — o log do JetStream é imutável. Elas expiram pela retenção do stream; o expurgo do dado autoritativo é executado pelo `../005-Database/`. Esta limitação é exatamente a razão de a minimização de payload ser **obrigatória**, e não recomendada. |
| **Segregação** | Contas NATS por tenant; nenhum caminho de assinatura compartilhado entre clientes. |
| **Trilha de colaboração** | Sessões A2A (quem, quando, quais capacidades, quanto volume) são registradas e auditáveis, inclusive com pares externos. |
| **Transferência internacional** | Leaf nodes/gateways entre regiões usam filtros de propagação; quais subjects cruzam fronteira geográfica é decisão registrada em `../027-Cluster/` com avaliação de adequação. |
| **Dados na DLQ** | `comm.dlq_entry` preserva o envelope original e é classificada `confidential`, sob RLS, com retenção curta e `last_error` livre de PII. |

---

## 9. Auditoria

Toda operação privilegiada gera registro em `../025-Audit/` com, no mínimo:
`trace_id`, identidade do chamador, capability avaliada, decisão do PDP, recurso alvo,
resultado e duração. Adicionalmente:

- **Sessões A2A**: par, tipo de par (interno/externo), capacidades concedidas, volume
  trocado, motivo de encerramento.
- **Exports entre contas**: subject, conta de origem e destino, justificativa e autor —
  o único caminho legítimo de tráfego entre tenants.
- **Replay e descarte de DLQ**: quem autorizou, qual mensagem, com qual justificativa.
- **Provisionamento e revogação de contas**: correlacionado com o `../021-Security/`.

---

## 10. Referências

- Brief: `./_DESIGN_BRIEF.md` §12
- Contratos de segurança e privacidade: `../003-RFC/RFC-0001-Architecture-Baseline.md` §6–§7
- Identidade e segredos: `../021-Security/` · Política: `../022-Policy/` · Auditoria: `../025-Audit/`
- Contas e isolamento: `./Deployment.md` §1 · Testes de segurança: `./Testing.md` §8
