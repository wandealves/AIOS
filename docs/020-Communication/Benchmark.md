---
Documento: Benchmark
Módulo: 020-Communication
Status: Draft
Versão: 0.1
Última atualização: 2026-07-22
Responsável (RACI-A): Arquiteto do Módulo 020-Communication
ADRs relacionados: ADR-0004, ADR-0204, ADR-0206, ADR-0207
RFCs relacionados: RFC-0001
Depende de: NonFunctionalRequirements.md, Testing.md, Scalability.md, 035-Benchmark
---

# 020-Communication — Benchmark

Metodologia e KPIs para verificar os NFR de desempenho e escala. Os alvos são os de
`./NonFunctionalRequirements.md` — este documento define **como medi-los**. A
metodologia geral do projeto está em `../035-Benchmark/`.

---

## 1. Ambiente de Referência

| Item | Especificação |
|------|---------------|
| Cluster NATS | 3 nós × 8 vCPU, 16 GiB RAM, NVMe local (≥ 100k IOPS), JetStream R=3 |
| Rede | 10 GbE, RTT intra-cluster < 0,3 ms |
| Gerador de carga | 4 nós × 8 vCPU, mesma rede |
| Payload padrão | 512 B (perfil real do tráfego de controle do AIOS) |
| Versões | NATS 2.10, clientes .NET 10 e Python |

Toda medição descarta os **primeiros 2 minutos** (aquecimento e formação de conexões).
Resultados obtidos fora deste ambiente **NÃO DEVEM** ser comparados com estes alvos.

---

## 2. KPIs e Alvos

| KPI | Alvo | NFR | Cenário |
|-----|------|-----|---------|
| p99 publish core (sem ack) | ≤ **2 ms** | NFR-001 | C-1 |
| p99 publish durável (ack R=3) | ≤ **15 ms** | NFR-002 | C-2 |
| p99 request/reply | ≤ **10 ms** | NFR-003 | C-3 |
| Throughput core | ≥ **1.000.000 msg/s** | NFR-004 | T-1 |
| Throughput persistido | ≥ **200.000 msg/s** | NFR-004 | T-2 |
| p99 publish→deliver | ≤ **20 ms** | NFR-008 | C-4 |
| Escala de namespace | ≥ **10⁵** subjects, **10⁴** consumidores | NFR-007 | S-1 |
| Amplificação de fan-out | **1** publicação por difusão | NFR-016 | F-1 |
| Sessões A2A simultâneas | ≥ **10⁴**, p99 handshake ≤ **200 ms** | NFR-015 | A-1 |
| Degradação por *slow consumer* | ≤ **10%** no p99 dos demais | NFR-011 | R-1 |
| Perda sob falha de 1 nó | **0** mensagens persistidas | NFR-006 | R-2 |

---

## 3. Perfil de Carga Sintética AIOS

A carga reproduz o padrão real do AIOS — dominado por mensagens **pequenas, de
controle e sensíveis a latência**, e não por grandes lotes de dados:

| Fluxo | Peso | Padrão | Persistido? |
|-------|------|--------|-------------|
| Eventos de ciclo de vida de agente (`agent.lifecycle.*`) | 30% | pub/sub durável | sim |
| Telemetria de execução (`task.execution.*`) | 25% | pub/sub core | não |
| Consultas ao PDP (`policy.decision.*`) | 20% | request/reply | não |
| Eventos de memória/contexto | 10% | pub/sub durável | sim |
| Difusões para *Agent Group* | 8% | fan-out | não |
| Sessões A2A | 5% | canal privado | não |
| Auditoria (`audit.*`) | 2% | pub/sub durável | sim |

Distribuição de tenants: **Zipf (s = 1,1)** sobre 1.000 tenants — reproduz o cenário de
*hot tenants*, onde a eficácia das cotas por tenant é realmente testada.

---

## 4. Cenários de Latência

### C-1 — Publish core (fire-and-forget)
- 200.000 msg/s, payload 512 B, 15 min.
- **Critério:** p99 ≤ 2 ms; p999 ≤ 10 ms.

### C-2 — Publish durável (ack JetStream R=3)
- 100.000 msg/s em `KERNEL_LIFECYCLE`, 15 min.
- **Critério:** p99 ≤ 15 ms. O custo do ack em relação a C-1 é a medida direta do preço
  da durabilidade, e é a informação que orienta cada módulo a escolher entre stream
  durável e pub/sub efêmero.

### C-3 — Request/reply
- 50.000 req/s com respondente de latência artificial de 1 ms.
- **Critério:** p99 ≤ 10 ms (excluído o 1 ms do respondente).

### C-4 — Publish → deliver
- Carga mista da §3 a 300.000 msg/s.
- **Critério:** p99 do intervalo publish→deliver ≤ 20 ms para consumidores saudáveis.

---

## 5. Cenários de Throughput

### T-1 — Teto core
- Rampa 200k → 1,5M msg/s em degraus de 200k, 5 min por degrau.
- Mede: taxa de erro, p99, CPU dos nós, uso de rede.
- **Critério:** ≥ 1.000.000 msg/s com p99 dentro de C-1 e erro < 0,01%.

### T-2 — Teto persistido
- Rampa 50k → 400k msg/s persistidas (R=3).
- **Critério:** ≥ 200.000 msg/s com p99 dentro de C-2 e **zero** perda.

### T-3 — Tamanho de payload
| Payload | Throughput core esperado | Observação |
|---------|--------------------------|------------|
| 128 B | ~1,4M msg/s | limitado por CPU/sintaxe de protocolo |
| **512 B** | **~1,0M msg/s** | **perfil de referência** |
| 4 KiB | ~350k msg/s | limitado por rede |
| 64 KiB | ~25k msg/s | já degrada latência de terceiros |
| 1 MiB (teto) | ~1,5k msg/s | admissível apenas como exceção |

A curva acima é o argumento empírico para `bus.publish.max_payload_bytes` = 1 MiB e para
o padrão "referência a objeto" (ADR-0208): mensagens grandes não apenas são lentas —
elas consomem a capacidade de todos os demais fluxos que compartilham o cluster.

---

## 6. Cenário de Fan-out (F-1)

- Grupos de 10, 100 e 1.000 membros; 1.000 difusões por segundo.
- Mede `aios_bus_group_publish_total ÷ aios_bus_group_broadcast_total`.

| Tamanho do grupo | Publicações do produtor | Entregas | Razão |
|------------------|-------------------------|----------|-------|
| 10 | 1.000/s | 10.000/s | **1** |
| 100 | 1.000/s | 100.000/s | **1** |
| 1.000 | 1.000/s | 1.000.000/s | **1** |

- **Critério:** razão constante em **1** (NFR-016). Qualquer valor acima indica que o
  produtor está iterando destinatários na aplicação — o antipadrão que
  `SelectiveCommunicationRouter` existe para eliminar.

**Contraste de referência** (medido para justificar o desenho): a mesma difusão feita
por iteração no produtor gera 1.000 publicações/s, satura o cliente e adiciona ~40 ms de
cauda ao p99 do grupo.

---

## 7. Cenário A2A (A-1)

- Abertura de 10.000 sessões em 5 min; 100 mensagens por sessão.
- **Critério:** p99 de handshake ≤ 200 ms (NFR-015); memória do `aios-comm-svc` estável;
  encerramento por ociosidade limpa 100% dos canais.

| Métrica | Alvo |
|---------|------|
| p99 handshake (par interno) | ≤ 200 ms |
| p99 handshake (par externo, mTLS) | ≤ 400 ms |
| Sessões simultâneas sustentadas | ≥ 10.000 |
| Canais órfãos após encerramento | **0** |

---

## 8. Cenários de Escala e Resiliência

### S-1 — Escala de namespace
- 10⁵ subjects registrados, 10⁴ consumidores duráveis, 10⁶ conexões lógicas.
- **Critério:** p99 de publish estável; memória do broker dentro do envelope de
  `./Deployment.md` §2; tempo de resolução de subject não cresce com o namespace.

### R-1 — Contenção por *slow consumer*
- Um consumidor com 500 ms de latência artificial por mensagem, no mesmo stream de um
  consumidor saudável.
- **Critério:** p99 do saudável degrada **≤ 10%** (NFR-011); o lento é contido em
  `max_ack_pending`.

### R-2 — Falha de nó
- Kill de 1 nó sob 300k msg/s persistidas.
- **Critério:** **0** mensagem perdida; p99 retorna ao normal em ≤ 60 s; quórum mantido.

### R-3 — Perda de quórum
- Kill de 2 nós.
- **Critério:** publicação durável falha rápido com `AIOS-BUS-0011` (sem espera longa);
  produtores retêm no Outbox; ao restaurar, o backlog drena sem duplicidade efetiva.

---

## 9. Reprodutibilidade

| Requisito | Detalhe |
|-----------|---------|
| Versionamento | Scripts de carga versionados junto às declarações de stream. |
| Sementes fixas | Distribuição Zipf e geração de payload com semente registrada. |
| Isolamento | Nenhuma outra carga no cluster durante a medição. |
| Registro | Cada execução publica: versão do NATS, configuração efetiva, hash do script, resultados brutos. |
| Comparação | Regressão > 10% em qualquer KPI **DEVE** bloquear a promoção da versão. |

---

## 10. Baseline e Histórico

| Versão | Data | p99 core | p99 durável | msg/s core | Fan-out | Observação |
|--------|------|----------|-------------|------------|---------|------------|
| 0.1 (alvo) | — | ≤ 2 ms | ≤ 15 ms | ≥ 1.000.000 | 1 | Metas de projeto; primeira medição pendente do ambiente de referência. |

Preenchida a cada release, serve de baseline para detectar regressão.

---

## 11. Referências

- Metas: `./NonFunctionalRequirements.md` · Testes: `./Testing.md`
- Escala: `./Scalability.md` · Metodologia global: `../035-Benchmark/`
- Configuração e topologia: `./Configuration.md` · `./Deployment.md`
