---
Documento: Testing
Módulo: 020-Communication
Status: Draft
Versão: 0.1
Última atualização: 2026-07-22
Responsável (RACI-A): Arquiteto do Módulo 020-Communication
ADRs relacionados: ADR-0200, ADR-0201, ADR-0202, ADR-0203, ADR-0205, ADR-0206, ADR-0207
RFCs relacionados: RFC-0001, RFC-0200, RFC-0201
Depende de: Requirements.md, FunctionalRequirements.md, NonFunctionalRequirements.md
---

# 020-Communication — Estratégia de Testes

Os identificadores (`T-XXX-NN`) são os mesmos da matriz de rastreabilidade em
`./Requirements.md` §3.

---

## 1. Pirâmide de Testes

```
                    ┌────────────────────────┐
                    │  Chaos / DR (mensal)   │  kill de nó, partição, quórum
                    ├────────────────────────┤
                    │  Carga / Escala        │  1M msg/s, 10⁵ subjects, 10⁴ consumers
                    ├────────────────────────┤
                    │  E2E (CI noturno)      │  produtor→bus→consumidor→DLQ→replay
                    ├────────────────────────┤
                    │  Integração (por PR)   │  cluster NATS real de 3 nós
                    ├────────────────────────┤
                    │  Unidade (por commit)  │  regras de envelope, FSM, cotas
                    └────────────────────────┘
```

Princípio: **testes de barramento usam barramento real**. Nenhuma suíte deste módulo
usa broker em memória ou *mock* de cliente NATS — a semântica testada (ack, reentrega,
quórum, contas, dedupe) não existe fora do NATS.

---

## 2. Testes de Unidade

| ID | Alvo | Verifica |
|----|------|----------|
| T-UNIT-01 | `EnvelopeValidator` | Cada regra isolada: campos obrigatórios, `type` × subject, `tenant` × conta, `dataschema`, tamanho, `traceparent`. |
| T-UNIT-02 | FSM de sessão A2A | Todas as transições válidas (T-01..T-10) e rejeição das inválidas (`AIOS-A2A-0003`). |
| T-UNIT-03 | Forma de subject | Aceita `agent.lifecycle.spawned`; rejeita instância (`agent.01J9….status`), maiúsculas e componentes vazios. |
| T-UNIT-04 | Cálculo de backoff | Sequência crescente e compatível com `max_deliver`. |
| T-UNIT-05 | Janela de cota | Contagem deslizante correta em bordas de janela. |
| T-UNIT-06 | Precedência de configuração | agent/stream > tenant > global > default. |

Cobertura-alvo: **≥ 85%** nos pacotes `Registry`, `A2A` e `Delivery`.

---

## 3. Testes de Integração (cluster NATS real, 3 nós)

| ID | Requisito | Cenário | Critério |
|----|-----------|---------|----------|
| T-REG-01 | FR-002 | Publicar em subject não registrado. | `AIOS-BUS-0004`; nada entra no stream. |
| T-REG-02 | FR-002 | Registrar subject com componente por instância. | Recusado (`AIOS-BUS-0004`). |
| T-ENV-01 | FR-003 | Envelope sem `traceparent` / com `type` incoerente. | `AIOS-BUS-0005` em ambos. |
| T-ENV-02 | FR-003 | `dataschema` não registrado. | `AIOS-BUS-0015`. |
| T-ENV-03 | FR-015 | Payload de 2 MiB. | `AIOS-BUS-0006`. |
| T-BUS-01 | FR-001 | Pub/sub core simples. | Mensagem entregue; p99 dentro de NFR-001. |
| T-BUS-02 | FR-001 | Stream durável com consumidor durável. | Ack registrado; `pending` zera. |
| T-BUS-03 | FR-005 | Publicar o mesmo `event.id` 3× em 10 s. | Uma única mensagem no stream. |
| T-BUS-04 | FR-006 | 10⁵ mensagens sequenciais. | Ordem preservada no consumidor. |
| T-RR-01 | FR-001 | Request/reply com respondente ativo e ausente. | Resposta correlacionada; ausência → `AIOS-BUS-0012`. |
| T-DECL-01 | FR-016 | `PUT` de stream repetido, idêntico. | Idempotente; sem alteração. |
| T-DECL-02 | FR-016 | Mudança incompatível (réplicas 3→5) com backlog pendente. | Migração espelhada; **nenhuma** mensagem perdida (ADR-0203). |
| T-DLQ-01 | FR-007 | Consumidor que nunca confirma. | 5 tentativas com backoff; entrada em `comm.dlq_entry`. |
| T-DLQ-02 | FR-007 | Consumidor chama `term()`. | Vai direto à DLQ, sem gastar as 5 tentativas. |
| T-DLQ-03 | FR-008 | Replay de entrada de DLQ. | Reinjetada com o mesmo `event.id`; evento `delivery.replayed`. |
| T-RPL-01 | FR-008 | Replay de stream por tempo. | Consumidor isolado recebe; consumidor de produção **não** é afetado. |
| T-GRP-01 | FR-012 | Difusão para grupo de 1.000 membros. | 1 publicação, 1.000 entregas (NFR-016). |
| T-GRP-02 | FR-012 | Difusão acima de `fanout_limit`. | `AIOS-BUS-0008`. |
| T-A2A-01 | FR-013 | Sessão completa: open → established → close. | Estados e eventos conforme FSM; tentativa de ampliar capacidades → `AIOS-A2A-0007`. |
| T-A2A-02 | FR-013 | Sessão com capacidade negada pelo PDP. | `Rejected` com `AIOS-A2A-0001`; auditoria registra. |
| T-A2A-03 | FR-014 | Par externo com identidade revogada durante a sessão. | `AIOS-A2A-0005`; sessão → `Failed` (T-10). |
| T-A2A-04 | FR-020 | Sessão ociosa além do timeout; agente terminado. | T-08 automático; assinaturas liberadas ao receber `agent.lifecycle.terminated`. |
| T-OBS-01 | FR-018 | Trace do produtor ao consumidor. | Mesmo `trace_id` nas duas pontas; `traceparent` no envelope. |

---

## 4. Testes de Contrato

| ID | Contrato | Verifica |
|----|----------|----------|
| T-CTR-01 | OpenAPI `/v1/comm` | Requisição/resposta conformes; erros no envelope RFC 7807 (RFC-0001 §5.4). |
| T-CTR-02 | `aios.comm.v1` (proto) | Compatibilidade retroativa: nenhum campo removido ou renumerado. |
| T-CTR-03 | Envelope de mensagem | Todo evento emitido valida contra seu `dataschema`; campos desconhecidos são tolerados. |
| T-CTR-04 | Idempotência (NFR-014) | Repetição de `OpenSession`/`ReplayDeadLetter` com a mesma `Idempotency-Key` → efeito único. |
| T-CTR-05 | Contrato de subject | Todo subject declarado por outros módulos (`006`, `009`, `010`, `022`, `025`) existe no registro e aponta para stream ativo. |

T-CTR-05 é um teste **de integração entre módulos**: ele falha quando um módulo declara
em sua documentação um evento que nunca foi registrado no barramento — a divergência
mais comum entre "o que o doc diz" e "o que o sistema faz".

---

## 5. Testes de Isolamento e Governança de Namespace

| ID | Cenário | Critério |
|----|---------|----------|
| T-NS-01 | Registrar dois subjects com o mesmo (domain, entity, action). | Segundo recusado. |
| T-NS-02 | Declarar dois streams com padrões de subject sobrepostos. | Recusado (`AIOS-BUS-0004`, invariante C-02). |
| T-NS-03 | Depreciar subject em uso. | Publicação segue funcionando durante a coexistência; evento `subject.deprecated` emitido. |

---

## 6. Testes de Entrega sob Falha

| ID | Cenário | Critério |
|----|---------|----------|
| T-DEL-01 | Crash do produtor entre `COMMIT` e publicação. | Relay republica; dedupe evita duplicidade (efeito exatamente-uma-vez). |
| T-DEL-02 | Crash do consumidor após processar e antes do ack. | Reentrega; dedupe do consumidor evita efeito duplicado. |
| T-DEL-03 | NATS indisponível por 5 min durante publicação. | Mensagens permanecem no Outbox; publicadas ao religar; **zero perda**. |

---

## 7. Testes de Contenção e Concorrência

| ID | Requisito | Cenário | Critério |
|----|-----------|---------|----------|
| T-FLOW-01 | FR-010 | Tenant publica 2× a cota por 5 min. | `AIOS-BUS-0007` com `Retry-After`; evento `quota.throttled`; **outros tenants sem degradação**. |
| T-FLOW-02 | FR-011, NFR-011 | Um consumidor propositalmente lento no mesmo stream de um saudável. | Lento é contido (`AIOS-BUS-0014`); p99 do saudável degrada **≤ 10%**. |
| T-FLOW-03 | NFR-007 | Agente tenta abrir 100 assinaturas (limite 64). | Recusado a partir da 65ª; força uso de canal de grupo. |
| T-CONC-01 | NFR-014 | 100 `OpenSession` concorrentes com a mesma `Idempotency-Key`. | Uma única sessão criada; demais retornam a mesma. |

---

## 8. Testes de Segurança

| ID | Requisito | Cenário | Critério |
|----|-----------|---------|----------|
| T-SEC-01 | FR-009, NFR-010 | Credencial de `acme` assina `aios.>` e `aios.globex.>`. | Recebe **apenas** tráfego de `acme`; a conta `globex` é invisível ao roteamento. |
| T-SEC-02 | FR-019 | Tentar export entre contas sem capability; e com capability. | Sem: `AIOS-BUS-0002`. Com: criado e **auditado** com justificativa. |
| T-SEC-03 | FR-004 | Módulo `025` publica em subject cujo produtor é `006`. | `AIOS-BUS-0010`; tentativa auditada e alertada. |
| T-SEC-04 | FR-017 | PDP indisponível: abrir sessão A2A **e** publicar em fluxo já autorizado. | Sessão negada (*fail-closed*); publicação **continua funcionando** (princípio P-09). |
| T-SEC-05 | LGPD | Inspecionar logs de um fluxo com payload sensível. | Nenhum log contém o campo `data`; apenas `event_id`, subject e métricas. |

T-SEC-04 é o teste que protege contra a falha mais perigosa de desenho deste módulo:
transformar a indisponibilidade do serviço de política em queda total do barramento.

---

## 9. Testes de Carga e Escala

| ID | Requisito | Carga | Critério |
|----|-----------|-------|----------|
| T-LOAD-01 | NFR-001, NFR-004 | 1.000.000 msg/s core, payload 512 B, 30 min. | p99 publish ≤ 2 ms; sem erro sustentado. |
| T-LOAD-02 | NFR-002, NFR-004 | 200.000 msg/s persistidas (R=3). | p99 publish com ack ≤ 15 ms. |
| T-LOAD-03 | NFR-003 | 50.000 request/reply por segundo. | p99 ≤ 10 ms. |
| T-SCALE-01 | NFR-007 | 10⁵ subjects registrados, 10⁴ consumidores duráveis, 10⁶ conexões lógicas de agentes. | Latência estável; memória do broker dentro do envelope de `./Deployment.md`. |
| T-SCALE-02 | NFR-016 | Difusão a grupos de 10, 100 e 1.000 membros. | Publicações do produtor constantes em 1, independentemente do tamanho. |
| T-SCALE-03 | NFR-015 | 10⁴ sessões A2A simultâneas. | p99 de handshake ≤ 200 ms; memória estável. |

Metodologia detalhada em `./Benchmark.md`.

---

## 10. Testes de Caos e DR

| ID | Cenário | Critério |
|----|---------|----------|
| T-CHAOS-01 | Kill de 1 nó NATS sob carga (quórum mantido). | **0** mensagem persistida perdida (NFR-006); clientes reconectam; p99 volta ao normal em ≤ 60 s. |
| T-CHAOS-02 | Kill de 2 nós (quórum perdido). | Publicação durável falha com `AIOS-BUS-0011`; produtores retêm no Outbox; ao restaurar, backlog drena sem perda. |
| T-CHAOS-03 | Partição de rede entre leaf node e cluster central. | Cada lado opera localmente; ao religar, streams espelhados reconciliam com dedupe por `event.id`. |
| T-CHAOS-04 | PostgreSQL (`comm`) indisponível. | Tráfego de mensagens **não** é afetado; operações administrativas falham; DLQ acumula em memória até o limite e então alerta. |
| T-CHAOS-05 | Perda do volume de JetStream de um nó. | Ressincronização a partir das réplicas; sem perda. |
| T-DR-01 | DR drill: restauração de `AUDIT_TRAIL` a partir de snapshot. | **RTO ≤ 15 min**, **RPO ≤ 5 min** (NFR-009). |

---

## 11. Ambientes, Fixtures e Gates

| Ambiente | Uso | Configuração |
|----------|-----|--------------|
| CI efêmero | unidade, integração, contrato | Cluster NATS de 3 nós em contêiner, JetStream R=3, contas `acme`/`globex`/`_platform`. |
| `staging` | carga reduzida, contenção, contrato entre módulos | Topologia de produção com volume ~10%. |
| `prod` | drills de DR e verificação de quórum | Somente operações de leitura/verificação. |

**Gates de merge:**
1. Unidade + integração verdes (com cluster real).
2. T-CTR-05 verde: todo subject documentado por outros módulos está registrado.
3. Contrato sem quebra retroativa (OpenAPI + proto).
4. Testes de isolamento (T-SEC-01) verdes.
5. Cobertura ≥ 85% nos pacotes críticos.

Sem estes cinco, o PR **NÃO DEVE** ser mesclado.

---

## 12. Referências

- Rastreabilidade: `./Requirements.md` §3
- Requisitos: `./FunctionalRequirements.md` · `./NonFunctionalRequirements.md`
- Carga e KPIs: `./Benchmark.md` · Falhas: `./FailureRecovery.md`
