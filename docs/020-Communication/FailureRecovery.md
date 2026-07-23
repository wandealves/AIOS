---
Documento: FailureRecovery
Módulo: 020-Communication
Status: Draft
Versão: 0.1
Última atualização: 2026-07-22
Responsável (RACI-A): Arquiteto do Módulo 020-Communication
ADRs relacionados: ADR-0004, ADR-0203, ADR-0205, ADR-0209
RFCs relacionados: RFC-0001, RFC-0200
Depende de: _DESIGN_BRIEF.md §9, 027-Cluster, 029-Operations, Monitoring.md
---

# 020-Communication — Falhas e Recuperação

Metas: **RTO ≤ 15 min**, **RPO ≤ 5 min** para streams duráveis (V-R10 da Visão Global,
NFR-009).

Princípio de degradação: o barramento prefere **rejeitar na entrada com sinal claro de
recuo** a aceitar e perder depois. Ordem de sacrifício: telemetria e tráfego bulk →
replay e operações administrativas → **nunca** o tráfego de controle nem a entrega de
mensagens já aceitas.

---

## 1. FMEA — Modos de Falha

| # | Modo de falha | Detecção | Isolamento | Recuperação | Idempotência/Retry | Severidade |
|---|---------------|----------|-----------|-------------|--------------------|-----------|
| F-01 | Perda de 1 nó NATS | Health do cluster; `aios_bus_cluster_nodes` | Quórum mantido (R=3); clientes reconectam a outro nó | Reingresso automático; ressincronização de streams | Sem perda; publicação com ack é retentável | P2 |
| F-02 | Perda de quórum JetStream | Falha de ack de publicação durável | Publicação durável rejeitada (`AIOS-BUS-0011`); **core pub/sub segue** | Restaurar nós até o quórum; snapshot se necessário | Produtores retêm no Outbox e republicam | **P1** |
| F-03 | *Slow consumer* | `ack_pending` no teto; `ack_latency_ms` crescente | Contenção **daquele** consumidor; demais intactos | Escalar instâncias no mesmo *durable* ou corrigir o processamento | Reentrega natural após ack | P2 |
| F-04 | Consumidor parado | Sem ack por `stall_threshold_ms` | Backlog acumula no stream; evento `consumer.stalled` | Reinício do consumidor; DLQ após `max_deliver` | Backoff exponencial | P2 |
| F-05 | Mensagem envenenada | Reentregas repetidas com o mesmo erro | Vai à DLQ ao esgotar `max_deliver` | Inspeção, correção do consumidor, replay explícito | Replay nunca automático (ADR-0205) | P3 |
| F-06 | Stream atinge `max_bytes` | `stream_limit_ratio` > 0,9 | `discard=old` descarta o mais antigo; `discard=new` rejeita (`AIOS-BUS-0013`) | Aumentar limite ou reduzir `max_age`; investigar o produtor | Produtor recua com backpressure | **P1** |
| F-07 | Produtor inundando (*noisy neighbor*) | Cota por tenant/agente | `AIOS-BUS-0007`; a cota é por tenant, não global | Ajuste de orçamento via `../026-Cost-Optimizer/`; investigação | Retry com backoff | P2 |
| F-08 | Fan-out explosivo | `fanout_limit` excedido | Difusão rejeitada **antes** de executar (`AIOS-BUS-0008`) | Reduzir escopo ou dividir o grupo | Determinístico | P3 |
| F-09 | PDP (`022`) indisponível | Timeout/CB no `PolicyClient` | Bulkhead do PEP | *Fail-closed* para operações **novas**; tráfego já autorizado continua | Retry após meia-abertura do CB | P2 |
| F-10 | Par A2A desaparece | Heartbeat perdido / sem atividade | Sessão isolada | T-10 → `Failed`; canal liberado; evento emitido | Nova sessão exige novo handshake | P3 |
| F-11 | Credencial de conta revogada | `security.token.revoked` | Conta desconectada | Sessões afetadas encerradas; nova credencial pelo `../021-Security/` | Reconexão idempotente | P2 |
| F-12 | Partição leaf/gateway | Health do `BridgeConnector` | Cada lado opera localmente | Reconciliação ao religar; dedupe por `event.id` | At-least-once preservado | P2 |
| F-13 | Perda de dados de stream em um nó | Falha de leitura / réplica fora | Stream isolado nesse nó | Ressincronização a partir das réplicas; snapshot em último caso | Dedupe evita duplicidade | P2 |
| F-14 | PostgreSQL (`comm`) indisponível | Health do serviço | **Tráfego de mensagens não é afetado** | Operações administrativas falham; DLQ acumula em buffer e alerta | Reconciliação idempotente ao voltar | P2 |
| F-15 | `aios-comm-svc` indisponível | Readiness | Tráfego segue (o serviço está fora do caminho de mensagem) | Reinício; reconciliação declarativa | Idempotente | P3 |

A linha **F-15** merece destaque: como o serviço de controle não está no caminho de
mensagem (`./Architecture.md` §1), sua queda **não interrompe o barramento**. Perde-se
a capacidade de declarar streams, abrir sessões A2A e operar a DLQ — não a de entregar
mensagens. Essa propriedade é consequência direta da decisão de arquitetura, e é o que
diferencia um barramento de um proxy.

---

## 2. Detecção — Sinais Primários

| Sinal | Fonte | Falhas que revela |
|-------|-------|-------------------|
| `aios_bus_jetstream_quorum` | serviço/broker | F-02 |
| `aios_bus_cluster_nodes{state="healthy"}` | broker | F-01, precursor de F-02 |
| `aios_bus_consumer_ack_pending` / `aios_bus_consumers_stalled` | broker | F-03, F-04 |
| `aios_bus_deadletter_total` | serviço | F-05 |
| `aios_bus_stream_limit_ratio` | broker | F-06 |
| `aios_bus_quota_throttled_total` | serviço | F-07 |
| `aios_bus_group_fanout_rejected_total` | serviço | F-08 |
| `aios_bus_a2a_sessions_total{outcome="failed"}` | serviço | F-10, F-11 |
| `aios_bus_leaf_connections` | serviço | F-12 |
| `aios_bus_outbox_pending` | serviço | F-14 |

---

## 3. Isolamento e Bulkheads

```
   ┌── conta acme ──┐  ┌── conta globex ─┐  ┌── conta _platform ─┐
   │ cota msg/s     │  │ cota msg/s      │  │ cota msg/s         │  ← bulkhead por tenant
   │ cota bytes/s   │  │ cota bytes/s    │  │                    │
   └───────┬────────┘  └────────┬────────┘  └─────────┬──────────┘
           └────────────────────┼─────────────────────┘
                        cluster NATS (R=3)
                                │
        ┌───────────────────────┼───────────────────────┐
   max_ack_pending        fanout_limit           max_payload_bytes
   (consumidor lento)     (difusão explosiva)    (mensagem gigante)
        │                       │                       │
        └─── cada limite contém uma classe distinta de falha ───┘
```

Nenhuma dessas linhas de defesa depende do bom comportamento do chamador — é essa a
diferença entre um limite documentado e um limite **aplicado**.

---

## 4. Estratégia de Retry

| Erro | `retriable` | Política do cliente |
|------|-------------|---------------------|
| `AIOS-BUS-0007` (cota) | sim | Respeitar `Retry-After`; **não** aumentar concorrência. |
| `AIOS-BUS-0008` (fan-out) | sim | Repetir não resolve sem reduzir o escopo da difusão. |
| `AIOS-BUS-0011` (sem quórum) | sim | Backoff exponencial 100 ms → 30 s; a mensagem permanece no Outbox. |
| `AIOS-BUS-0012` (timeout de request) | sim | Repetir **apenas** se a operação for idempotente. |
| `AIOS-BUS-0014` (*slow consumer*) | sim | Não é ação do produtor: corrigir o consumidor. |
| `AIOS-BUS-0009` (conflito OCC) | sim | Reler, reaplicar, repetir (máx. 5). |
| `AIOS-A2A-0004` (sessão suspensa) | sim | Aguardar reautorização; não reabrir sessão nova imediatamente. |
| `AIOS-BUS-0002/-0003/-0004/-0005/-0006/-0010`, `AIOS-A2A-0001/-0005/-0007` | **não** | Corrigir a causa; repetir não resolve. |

Toda mutação administrativa repetida **DEVE** reusar a mesma `Idempotency-Key`
(RFC-0001 §5.5). Toda republicação **DEVE** preservar o `event.id` original, para que a
dedupe funcione.

---

## 5. Procedimentos de Recuperação

### 5.1 Perda de quórum de JetStream (F-02)

```
 1. Confirmar quantos nós estão fora (aios_bus_cluster_nodes, :8222/jsz).
 2. NÃO reiniciar nós saudáveis — isso agrava a perda de quórum.
 3. Restaurar o mínimo de nós para o quórum (2 de 3); aguardar streams "healthy".
 4. Verificar aios_bus_outbox_pending dos produtores: o backlog deve drenar sozinho.
 5. Confirmar ausência de duplicidade efetiva (dedupe por event.id).
 6. Post-mortem: por que dois nós caíram juntos? (domínio de falha compartilhado?)
```
Durante a janela, o **core pub/sub continua operando** — apenas a publicação durável
falha. Alvo: quórum restabelecido em **≤ 5 min**; total dentro de RTO ≤ 15 min.

### 5.2 Triagem de DLQ (F-05)

```
 1. Agrupar por last_error (consulta em ./Database.md §10).
 2. Classificar: defeito do consumidor | mensagem malformada | dependência externa.
 3. Corrigir a causa e implantar.
 4. SÓ ENTÃO: replay (UC-007), em lote pequeno primeiro.
 5. Se a mensagem for irrecuperável: :discard com justificativa auditada.
```
Replay antes da correção reproduz o incidente — é por isso que `auto_replay` é `false`.

### 5.3 Restauração de stream (F-13, DR)

```
 1. Identificar o stream e a janela perdida.
 2. Preferir ressincronização a partir das réplicas (não há perda com R=3).
 3. Se todas as réplicas foram perdidas: restaurar snapshot de MinIO.
 4. Solicitar aos produtores a republicação a partir do Outbox, quando disponível.
 5. Validar contagem e ausência de lacuna de sequência; registrar RTO/RPO medidos.
```

---

## 6. Testes de Recuperação

| Exercício | Frequência | Critério |
|-----------|-----------|----------|
| Chaos: kill de 1 nó sob carga | Mensal | 0 mensagem persistida perdida (NFR-006); p99 normaliza em ≤ 60 s. |
| Chaos: perda de quórum | Trimestral | Falha rápida e explícita; backlog drena sem duplicidade. |
| Partição de leaf node | Trimestral | Operação local preservada; reconciliação correta ao religar. |
| Simulação de PDP indisponível | Trimestral | Novas sessões negadas; **tráfego contínuo** (T-SEC-04). |
| DR drill: restauração de `AUDIT_TRAIL` | Trimestral | RTO ≤ 15 min, RPO ≤ 5 min. |
| Exercício de triagem de DLQ | Semestral | Equipe corrige a causa e faz replay sem reincidência. |

---

## 7. Degradação Graciosa — Ordem de Sacrifício

```
   1º  Telemetria e tráfego bulk        (amostragem agressiva; classe `telemetry`)
   2º  Replay e operações administrativas
   3º  Difusões de grupo não críticas
   4º  Sessões A2A novas                (as existentes seguem)
   ─────────────────── nunca sacrificado ───────────────────
   ·  Entrega de mensagens já aceitas (ack dado ao produtor)
   ·  Tráfego de controle (classe `control`)
   ·  Isolamento entre contas de tenant
```

A classe de tráfego (`traffic_class` em `comm.subject_registry`) não é decorativa: ela é
o que torna essa ordem **executável** em vez de aspiracional — o `FlowController` reduz
primeiro o que está marcado como `telemetry` e `bulk`.

---

## 8. Referências

- Brief: `./_DESIGN_BRIEF.md` §9
- Alertas e runbooks: `./Monitoring.md` · `../029-Operations/`
- HA/DR e topologia: `../027-Cluster/` · Testes: `./Testing.md` §10
- Erros: `./API.md` §5
