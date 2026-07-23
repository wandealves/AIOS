---
Documento: NonFunctionalRequirements
Módulo: 020-Communication
Status: Draft
Versão: 0.1
Última atualização: 2026-07-22
Responsável (RACI-A): Arquiteto do Módulo 020-Communication
ADRs relacionados: ADR-0004, ADR-0201, ADR-0204, ADR-0206, ADR-0207
RFCs relacionados: RFC-0001, RFC-0200, RFC-0201
Depende de: _DESIGN_BRIEF.md §7.2, 024-Observability, 027-Cluster, 035-Benchmark
---

# 020-Communication — Requisitos Não-Funcionais

Metas derivadas de `./_DESIGN_BRIEF.md` §7.2. **Os valores numéricos abaixo são
normativos** e aparecem idênticos em `./Monitoring.md`, `./Benchmark.md`,
`./FailureRecovery.md`, `./Scalability.md` e `./Configuration.md`.

---

## 1. Tabela de Requisitos Não-Funcionais

| ID | Atributo | Meta (SLO) | SLI / método de verificação |
|----|----------|-----------|-----------------------------|
| NFR-001 | Desempenho (publicação core) | p99 de publish *fire-and-forget* ≤ **2 ms** intra-cluster. | `aios_bus_publish_latency_ms{durable="false"}`; cenário C-1 de `./Benchmark.md`. |
| NFR-002 | Desempenho (publicação durável) | p99 de publish com ack JetStream (R=3) ≤ **15 ms**. | `aios_bus_publish_latency_ms{durable="true"}`; cenário C-2. |
| NFR-003 | Desempenho (request/reply) | p99 ≤ **10 ms** intra-cluster, excluído o processamento do respondente. | `aios_bus_request_latency_ms`; cenário C-3. |
| NFR-004 | Throughput | ≥ **1.000.000 mensagens/s** core e ≥ **200.000 mensagens/s** persistidas, em cluster de 3 nós. | `aios_bus_messages_total` (taxa); cenários T-1 e T-2. |
| NFR-005 | Disponibilidade | ≥ **99,95%** do barramento; perda de 1 nó sem indisponibilidade percebida. | Uptime probe; *error budget* de 21,9 min/mês. |
| NFR-006 | Entrega | Perda de mensagem persistida = **0** sob falha de 1 nó (R=3, quórum mantido). | Chaos test com contagem publicada × consumida. |
| NFR-007 | Escalabilidade | ≥ **10⁶** agentes conectados, ≥ **10⁵** subjects ativos, ≥ **10⁴** consumidores duráveis, com custo de coordenação **sub-linear**. | Teste de escala em `./Scalability.md` §7. |
| NFR-008 | Latência de entrega | p99 do intervalo publish→deliver ≤ **20 ms** para consumidor saudável. | `aios_bus_delivery_latency_ms`. |
| NFR-009 | Recuperação | **RTO ≤ 15 min**, **RPO ≤ 5 min** para streams duráveis. | DR drill; restauração de snapshot JetStream. |
| NFR-010 | Isolamento multi-tenant | **0** vazamento entre contas; **100%** dos tenants em conta própria. | Teste de penetração de subject; auditoria de contas. |
| NFR-011 | Contenção | Um *slow consumer* **NÃO DEVE** degradar o p99 dos demais em mais de **10%**. | Cenário R-1 com consumidor artificialmente lento. |
| NFR-012 | Taxa de DLQ | ≤ **0,01%** das mensagens entregues em regime normal. | `aios_bus_deadletter_total` ÷ `aios_bus_messages_total`. |
| NFR-013 | Observabilidade | **100%** das mensagens com `traceparent` propagado e correlação preservada. | Amostragem de traces ponta a ponta. |
| NFR-014 | Idempotência | Repetições de mutação administrativa produzem efeito único em **100%** dos casos. | Teste de replay com `Idempotency-Key`. |
| NFR-015 | Sessões A2A | ≥ **10⁴** sessões simultâneas por cluster; p99 de handshake ≤ **200 ms**. | `aios_bus_a2a_sessions_active`, `aios_bus_a2a_handshake_ms`. |
| NFR-016 | Eficiência de fan-out | Difusão para um *Agent Group* de 1.000 membros **NÃO DEVE** custar mais que **1** publicação do produtor. | Contagem de publicações × entregas no cenário F-1. |

---

## 2. Orçamento de Erro e Priorização

| SLO | Orçamento mensal | Política ao esgotar |
|-----|------------------|---------------------|
| Disponibilidade 99,95% (NFR-005) | 21,9 min | Congelamento de mudanças de topologia e de declarações de stream não críticas. |
| Perda zero de mensagem persistida (NFR-006) | **0 violações** | Incidente P1; investigação de quórum e de configuração de réplicas. |
| Isolamento (NFR-010) | **0 violações** | Incidente P1; revogação de credenciais e auditoria retroativa de tráfego. |
| Taxa de DLQ ≤ 0,01% (NFR-012) | 0,01% das mensagens | Acima disso, o consumidor responsável entra em revisão obrigatória. |

Regra de priorização sob conflito: **isolamento > entrega > disponibilidade >
latência > throughput**. Perder isolamento é incidente de segurança; perder latência é
degradação. Quando duas metas colidem, a de maior precedência vence e a decisão é
registrada em `../025-Audit/`.

---

## 3. Atributos por Categoria

### 3.1 Desempenho
NFR-001 a NFR-004 e NFR-008 definem o envelope. O serviço `aios-comm-svc` **NÃO DEVE**
estar no caminho de mensagem: produtores e consumidores falam com o NATS diretamente
(`./Architecture.md` §1), evitando dois saltos de rede por mensagem.

### 3.2 Escalabilidade
NFR-007 e NFR-016 são o coração do módulo. A meta não é apenas "aguentar 10⁶ agentes",
é fazê-lo com custo de coordenação **sub-linear** — o que exige que o tráfego seja
hierárquico e seletivo, não ponto a ponto (`./Scalability.md` §3).

### 3.3 Confiabilidade
NFR-006, NFR-009 e NFR-012 formam a cadeia de garantias: nada se perde (R=3 + Outbox),
o que falha é reentregue, e o que persiste falhando é quarentenado de forma
inspecionável.

### 3.4 Segurança
NFR-010 é o invariante mais forte. Conta NATS por tenant significa que o roteamento
**nem considera** subjects de outra conta — o isolamento não depende de a permissão
estar corretamente escrita.

### 3.5 Observabilidade
NFR-013 exige `traceparent` em 100% das mensagens. Sem isso, um evento que atravessa
seis módulos vira seis incidentes independentes em vez de um trace.

---

## 4. Verificação — exemplo de consulta de SLI

```promql
# NFR-001 — p99 de publish core
histogram_quantile(0.99,
  sum by (le) (rate(aios_bus_publish_latency_ms_bucket{durable="false"}[15m]))
) < 2

# NFR-008 — p99 de entrega
histogram_quantile(0.99,
  sum by (le) (rate(aios_bus_delivery_latency_ms_bucket[15m]))
) < 20

# NFR-012 — taxa de DLQ
sum(rate(aios_bus_deadletter_total[1h])) / sum(rate(aios_bus_messages_total[1h])) < 0.0001

# NFR-016 — amplificação de fan-out (deve ser 1 publicação por difusão)
sum(rate(aios_bus_group_publish_total[5m])) /
sum(rate(aios_bus_group_broadcast_total[5m])) <= 1
```

---

## 5. Rastreabilidade

| NFR | Requisitos funcionais relacionados | Documento de verificação |
|-----|-----------------------------------|--------------------------|
| NFR-001, NFR-002, NFR-003, NFR-004 | FR-001, FR-003 | `./Benchmark.md` §4 |
| NFR-005, NFR-006, NFR-009 | FR-005, FR-006 | `./FailureRecovery.md` |
| NFR-007, NFR-016 | FR-012 | `./Scalability.md` |
| NFR-008 | FR-001, FR-007 | `./Benchmark.md` §5 |
| NFR-010 | FR-009, FR-019 | `./Security.md` §6 |
| NFR-011 | FR-010, FR-011 | `./Testing.md` §7 |
| NFR-012 | FR-007, FR-008 | `./Monitoring.md` |
| NFR-013 | FR-018 | `./Logging.md` |
| NFR-014 | FR-016 | `./Testing.md` §4 |
| NFR-015 | FR-013, FR-014, FR-020 | `./Benchmark.md` §7 |

---

## 6. Referências

- Brief: `./_DESIGN_BRIEF.md` §7.2
- Requisitos funcionais: `./FunctionalRequirements.md`
- Monitoramento: `./Monitoring.md` · Métricas: `./Metrics.md`
