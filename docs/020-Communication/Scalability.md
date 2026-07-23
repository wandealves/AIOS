---
Documento: Scalability
Módulo: 020-Communication
Status: Draft
Versão: 0.1
Última atualização: 2026-07-22
Responsável (RACI-A): Arquiteto do Módulo 020-Communication
ADRs relacionados: ADR-0004, ADR-0201, ADR-0204, ADR-0206, ADR-0207
RFCs relacionados: RFC-0001, RFC-0200
Depende de: _DESIGN_BRIEF.md §10, 027-Cluster, Benchmark.md, 009-Scheduler (agent groups)
---

# 020-Communication — Escalabilidade e Concorrência

Meta central: sustentar **≥ 10⁶ agentes**, **≥ 10⁵ subjects ativos** e **≥ 10⁴
consumidores duráveis** com custo de coordenação **sub-linear** (NFR-007).

Este é o módulo onde a meta de 10⁶ agentes encontra seu obstáculo mais duro. Computação
escala comprando máquinas; **coordenação não**. Uma frota em que todos falam com todos
produz `N²` caminhos e colapsa muito antes de esgotar CPU.

---

## 1. O Problema Quadrático

```
   ❌ malha ponto a ponto                 ✅ hierarquia + grupos
      N agentes → N² caminhos                N agentes → O(N) assinaturas

      A ─── B                             aios.acme.group.squad-7.>
      │ \ / │                                   ▲    ▲    ▲    ▲
      │  X  │                                   │    │    │    │
      C ─── D                                   A    B    C    D
                                          (1 publish → fan-out do broker)

   1.000 agentes:  ~500.000 caminhos        1.000 agentes: 1.000 assinaturas
   10⁶ agentes:    ~5×10¹¹ caminhos         10⁶ agentes:   10⁶ assinaturas
```

A coluna da direita é a razão de existir do `SelectiveCommunicationRouter` e do limite
`max_subscriptions_agent` (64). O limite não é uma restrição arbitrária: ele **impede
estruturalmente** que a topologia da esquerda seja construída, mesmo por engano.

---

## 2. Estratégias por Dimensão

| Dimensão | Estratégia |
|----------|-----------|
| Namespace | Subjects hierárquicos `aios.<tenant>.<dominio>.<entidade>.<acao>` (RFC-0001 §5.3): o filtro acontece no **broker**, no nível exato de interesse, não no consumidor. |
| Isolamento de tenant | **Conta NATS por tenant**: o roteamento nem considera subjects de outra conta — isolamento que também é otimização de dispersão. |
| Comunicação seletiva | Canais de *Agent Group* com `fanout_limit` e `scope`: uma difusão custa **1 publicação** do produtor, qualquer que seja o tamanho do grupo (NFR-016). |
| Contenção de N² | `max_subscriptions_agent` (64) força o uso de grupos e sessões A2A explícitas em vez de malha ponto a ponto. |
| Streams por domínio | Um stream por família de fluxo, **não** por tenant: o número de streams é O(domínios), não O(tenants) — condição para operar com 10⁴ tenants (ADR-0204). |
| Consumidores duráveis | `max_ack_pending` limita o em-voo; paralelismo se obtém com **múltiplas instâncias no mesmo `durable`**, não com janela maior. |
| Backpressure | Cotas por tenant/agente + `max_ack_pending` + `discard policy`: a pressão sobe até o produtor em vez de inchar o broker. |
| Escala horizontal | Cluster NATS de 3+ nós com JetStream R=3; `aios-comm-svc` é stateless (declaração no PostgreSQL, runtime no NATS). |
| Escala geográfica | Leaf nodes e gateways de supercluster com **filtros de propagação** — tráfego local não atravessa regiões (`../027-Cluster/`). |
| Sessões A2A | Canal privado por sessão, encerrado por ociosidade; sessões são recurso **cotado** (`max_sessions_agent` = 32), não recurso livre. |

---

## 3. Por que um stream por domínio, e não por tenant

| Modelo | Streams com 10⁴ tenants × 8 domínios | Consequência |
|--------|--------------------------------------|--------------|
| Por tenant × domínio | **80.000** streams | Inoperável: limites de arquivo, memória do broker, observabilidade ilegível, tempo de reconciliação inviável. |
| **Por domínio** (adotado) | **8** streams | Operável; retenção uniforme por família de fluxo; isolamento garantido pela **conta**, não pelo stream. |

O isolamento entre tenants **não** vem da separação de streams — vem da conta NATS
(§2). Separar streams por tenant seria pagar um custo operacional enorme por uma
garantia que já se tem de outra forma.

**Consequência aceita:** a retenção é por domínio, não por cliente. Expurgo por tenant
no plano de dados é responsabilidade de `../005-Database/`; o barramento apenas expira
por `max_age`.

---

## 4. Curva de Escala Esperada

| Escala | Subjects | Consumidores duráveis | Streams | p99 publish core |
|--------|----------|-----------------------|---------|------------------|
| 10³ agentes | ~10² | ~10² | ~10 | ≤ 2 ms |
| 10⁴ agentes | ~10³ | ~10³ | ~12 | ≤ 2 ms |
| 10⁵ agentes | ~10⁴ | ~10³ | ~15 | ≤ 2 ms |
| **10⁶ agentes** | **~10⁵** | **~10⁴** | **~20** | **≤ 2 ms** |

O ponto essencial: o número de **streams** cresce com o número de **domínios de
negócio**, não com o de agentes ou tenants. É isso que mantém a curva plana — e é a
razão de ADR-0204 ser uma decisão de escalabilidade, não apenas de organização.

---

## 5. Concorrência

| Mecanismo | Onde | Motivo |
|-----------|------|--------|
| OCC (`version`) | Declarações (`comm.*`) | Conflito é raro; retry é mais barato que lock. |
| Fila por consumidor durável | Entrega | Múltiplas instâncias no mesmo `durable` competem pelas mensagens — paralelismo sem coordenação explícita. |
| `max_ack_pending` | Entrega | Limita o em-voo por consumidor; é o *bulkhead* que impede um lento de arrastar os demais. |
| Dedupe por `event.id` | Publicação e consumo | Torna at-least-once suficiente: não é preciso protocolo de commit distribuído para obter efeito único. |
| Sem estado compartilhado no `aios-comm-svc` | Serviço de controle | Réplicas escalam sem coordenação; a única serialização é a reconciliação declarativa. |

---

## 6. Backpressure

```
   produtor ──▶ cota (tenant/agente) ──▶ NATS ──▶ max_ack_pending ──▶ consumidor
      ▲                 │                            │
      │  AIOS-BUS-0007  │                            │ AIOS-BUS-0014
      └── Retry-After ──┘                            └── contenção daquele consumidor

   stream cheio ──▶ discard=old (perde histórico)  |  discard=new (AIOS-BUS-0013)
```

A rejeição é **precoce e explícita**. É preferível recusar 5% das publicações de um
produtor saturado a degradar o p99 de todos os tenants. `Retry-After` orienta o cliente
a recuar — repetir imediatamente apenas realimenta a saturação.

A escolha de `discard` é uma decisão de negócio do dono do stream: `old` prioriza
disponibilidade de escrita e aceita perder histórico; `new` prioriza integridade do
histórico e faz a escrita falhar. Streams de auditoria usam `new`; telemetria usa `old`.

---

## 7. Teste de Escala (T-SCALE-01)

| Parâmetro | Valor |
|-----------|-------|
| Tenants | 1.000 (Zipf s = 1,1) |
| Agentes simulados | 10⁶ (conexões lógicas via runtimes) |
| Subjects registrados | 10⁵ |
| Consumidores duráveis | 10⁴ |
| Sessões A2A simultâneas | 10⁴ |
| Carga | perfil da §3 de `./Benchmark.md`, 500k msg/s |
| Duração | 24 h |

**Critérios:** p99 de publish core estável ao longo das 24 h; memória do broker dentro
do envelope de `./Deployment.md` §2; tempo de resolução de subject **não** cresce com o
tamanho do namespace; nenhuma difusão com amplificação > 1.

---

## 8. Limites Conhecidos e Mitigações

| Limite | Impacto | Mitigação |
|--------|---------|-----------|
| Cluster único como domínio de falha | Perda de quórum para todo o AIOS | R=3 em domínios de falha distintos; supercluster e leaf nodes via `../027-Cluster/`. |
| Streams grandes concentram I/O em poucos nós | *Hot shard* de disco | Dividir por domínio e revisar `max_age`; NATS não reparticiona stream automaticamente. |
| `max_bytes` fixo por stream | Descarte ou rejeição ao encher | Alerta em 90% (`./Monitoring.md`); revisão periódica de retenção. |
| Retenção uniforme por domínio | Não há TTL por tenant | Aceito conscientemente (§3); expurgo por titular é do `../005-Database/`. |
| Payload limitado a 1 MiB | Casos de uso com dados grandes | Padrão "referência a objeto" (ADR-0208): envia-se o ponteiro a MinIO. |
| Ordem apenas por stream | Não há ordem global | Consumidores que cruzam domínios reconciliam por `time`/`event.id` — ordem global custaria a escala. |
| Sessões A2A consomem memória do serviço | Teto de sessões simultâneas | Cota por agente, timeout de ociosidade, encerramento automático (FR-020). |

---

## 9. Rumo a Milhões

A combinação que sustenta 10⁶ agentes:

1. **Hierarquia de subjects** — o consumidor assina exatamente o que precisa; o filtro
   é do broker.
2. **Conta por tenant** — o roteamento sequer avalia o tráfego alheio.
3. **Canais de grupo** — difusão custa 1 publicação, independentemente do tamanho.
4. **Limite de assinaturas por agente** — impede a construção da malha quadrática.
5. **Streams por domínio** — número de streams independe de tenants e agentes.
6. **Backpressure com sinal explícito** — saturação vira recuo do produtor, não colapso.
7. **Leaf nodes com filtro** — geografia não multiplica tráfego local.

Nenhum desses sete é otimização tardia: todos são propriedades do desenho do namespace
e das cotas, aplicadas no momento em que o subject é registrado e a conta provisionada.
O custo de coordenação do AIOS cresce com o número de **grupos e domínios** — não com o
quadrado do número de agentes.

---

## 10. Referências

- Brief: `./_DESIGN_BRIEF.md` §10
- Medição: `./Benchmark.md` · Falhas sob carga: `./FailureRecovery.md`
- Topologia e HA: `./Deployment.md` · `../027-Cluster/`
- *Agent Groups* (composição): `../008-Agent-Lifecycle/`, `../009-Scheduler/`
