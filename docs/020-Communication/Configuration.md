---
Documento: Configuration
Módulo: 020-Communication
Status: Draft
Versão: 0.1
Última atualização: 2026-07-22
Responsável (RACI-A): Arquiteto do Módulo 020-Communication
ADRs relacionados: ADR-0201, ADR-0203, ADR-0205, ADR-0206, ADR-0207, ADR-0208, ADR-0209
RFCs relacionados: RFC-0001, RFC-0200, RFC-0201
Depende de: _DESIGN_BRIEF.md §8, Deployment.md, 021-Security (segredos), 026-Cost-Optimizer
---

# 020-Communication — Configuração

Prefixo de variável de ambiente: **`AIOS_BUS_`**. A chave `bus.consumer.max_deliver`
corresponde a `AIOS_BUS_CONSUMER_MAX_DELIVER` (pontos → `_`, maiúsculas).

Escopos: `global` (cluster), `tenant`, `agent`, `stream`. **Recarregável** indica *hot
reload* sem reinício. Os valores abaixo são idênticos aos de `./_DESIGN_BRIEF.md` §8 —
divergência é defeito.

---

## 1. Catálogo de Chaves

### 1.1 Publicação e validação

| Chave | Tipo | Default | Faixa | Escopo | Recarregável | Descrição |
|-------|------|---------|-------|--------|--------------|-----------|
| `bus.publish.max_payload_bytes` | int | 1048576 | 4096–8388608 | global/tenant | sim | Teto de payload (`AIOS-BUS-0006`); acima disso, use referência a objeto. |
| `bus.publish.validate_envelope` | bool | `true` | true\|false | global | sim | Validação do envelope CloudEvents no publish (ADR-0202). |
| `bus.publish.require_registered_subject` | bool | `true` | true\|false | global | sim | Rejeita subject fora do registro (`AIOS-BUS-0004`). |

> **Aviso normativo.** `validate_envelope` e `require_registered_subject` **NÃO DEVEM**
> ser desligados em produção: sem eles, uma mensagem malformada entra no stream e o
> defeito reaparece em todos os consumidores, longe do autor.

### 1.2 Streams

| Chave | Tipo | Default | Faixa | Escopo | Recarregável | Descrição |
|-------|------|---------|-------|--------|--------------|-----------|
| `bus.stream.default_replicas` | int | 3 | 1–5 | global/stream | **não** | Réplicas JetStream; alterar exige migração espelhada (ADR-0203). |
| `bus.stream.default_max_age` | duration | `7d` | 1h–3650d | stream | sim | Retenção temporal default. |
| `bus.stream.default_max_bytes` | int | 53687091200 | 1e9–1e13 | stream | sim | Teto de armazenamento por stream (50 GiB). |
| `bus.stream.dedupe_window` | duration | `2m` | 10s–1h | stream | sim | Janela de dedupe por `event.id` (RFC-0001 §5.5). |
| `bus.stream.discard_policy` | enum | `old` | old\|new | stream | sim | Descarte ao atingir o limite. |

### 1.3 Consumidores e entrega

| Chave | Tipo | Default | Faixa | Escopo | Recarregável | Descrição |
|-------|------|---------|-------|--------|--------------|-----------|
| `bus.consumer.ack_wait_ms` | int | 30000 | 1000–600000 | stream/global | sim | Prazo de ack antes da reentrega. |
| `bus.consumer.max_deliver` | int | 5 | 1–100 | stream/global | sim | Tentativas antes da DLQ. |
| `bus.consumer.backoff_ms` | int[] | `[1000,5000,30000,120000]` | lista crescente | stream | sim | Backoff por tentativa. |
| `bus.consumer.max_ack_pending` | int | 1000 | 1–100000 | stream/global | sim | Mensagens em voo por consumidor (backpressure). |
| `bus.consumer.stall_threshold_ms` | int | 60000 | 5000–3600000 | global | sim | Silêncio até declarar consumidor parado. |

### 1.4 Cotas e contenção

| Chave | Tipo | Default | Faixa | Escopo | Recarregável | Descrição |
|-------|------|---------|-------|--------|--------------|-----------|
| `bus.quota.msgs_per_sec_tenant` | int | 50000 | 100–5000000 | tenant | sim | Cota de publicação por tenant (`AIOS-BUS-0007`). |
| `bus.quota.msgs_per_sec_agent` | int | 200 | 1–100000 | agent | sim | Cota de publicação por agente. |
| `bus.quota.bytes_per_sec_tenant` | int | 104857600 | 1e6–1e10 | tenant | sim | Cota de banda por tenant (100 MiB/s). |
| `bus.quota.max_subscriptions_agent` | int | 64 | 1–4096 | agent | sim | Assinaturas simultâneas por agente — força o uso de grupos em vez de malha ponto a ponto. |

### 1.5 Comunicação seletiva

| Chave | Tipo | Default | Faixa | Escopo | Recarregável | Descrição |
|-------|------|---------|-------|--------|--------------|-----------|
| `bus.group.fanout_limit` | int | 256 | 1–10000 | tenant/global | sim | Destinatários por difusão (`AIOS-BUS-0008`). |
| `bus.group.default_scope` | enum | `group` | group\|hierarchy\|broadcast | tenant | sim | Escopo default de propagação. |

### 1.6 Request/reply e A2A

| Chave | Tipo | Default | Faixa | Escopo | Recarregável | Descrição |
|-------|------|---------|-------|--------|--------------|-----------|
| `bus.request.timeout_ms` | int | 5000 | 100–120000 | global/tenant | sim | Timeout de request/reply (`AIOS-BUS-0012`). |
| `bus.a2a.handshake_timeout_ms` | int | 3000 | 500–30000 | global/tenant | sim | Prazo de handshake (T-03). |
| `bus.a2a.idle_timeout_ms` | int | 300000 | 10000–86400000 | tenant/agent | sim | Ociosidade até encerrar sessão (T-08). |
| `bus.a2a.max_sessions_agent` | int | 32 | 1–1024 | agent | sim | Sessões simultâneas por agente (`AIOS-A2A-0006`). |
| `bus.a2a.allow_external_peers` | bool | `false` | true\|false | tenant | sim | Habilita pares A2A externos (exige identidade federada). |

### 1.7 DLQ, política e topologia

| Chave | Tipo | Default | Faixa | Escopo | Recarregável | Descrição |
|-------|------|---------|-------|--------|--------------|-----------|
| `bus.dlq.retention_days` | int | 30 | 1–365 | global | sim | Retenção da quarentena. |
| `bus.dlq.auto_replay` | bool | `false` | true\|false | global | sim | Replay automático da DLQ. **Default desligado**: replay cego repete o defeito (ADR-0205). |
| `bus.policy.fail_mode` | enum | `closed` | closed\|open | global | sim | Comportamento se o PDP estiver indisponível — aplica-se a operações **novas**, não ao tráfego já autorizado. |
| `bus.cluster.leaf_enabled` | bool | `false` | true\|false | global | **não** | Habilita leaf nodes (topologia decidida por `../027-Cluster/`). |

---

## 2. Segredos (nunca em arquivo de configuração)

| Segredo | Origem | Rotação |
|---------|--------|---------|
| NKey/JWT das contas de tenant | Cofre do `../021-Security/` | Automática, sem downtime (dupla credencial na janela). |
| Chave de assinatura do *operator* NATS | Cofre do `../021-Security/` (offline) | Anual, com cerimônia. |
| Certificados mTLS do serviço e dos pares A2A externos | PKI interna (`../021-Security/`) | Conforme política do `021`. |

Segredos **NÃO DEVEM** aparecer em variáveis de ambiente em texto claro nem em logs
(`./Logging.md` §5).

---

## 3. Precedência de Configuração

```
   valor efetivo = agent | stream (mais específico)
                 ▲ tenant
                 ▲ global
                 ▲ default do código (menos específico)
```

Chave de escopo `global` **NÃO PODE** ser sobrescrita por tenant; a tentativa é
rejeitada na validação. Chaves não recarregáveis alteradas em runtime ficam pendentes
até o próximo reinício e aparecem em `aios_bus_config_pending_reload`.

Cotas (`bus.quota.*`) têm uma fonte externa adicional: o evento
`aios.<tenant>.cost.budget.updated` do `../026-Cost-Optimizer/` sobrescreve o valor
efetivo por tenant. Editar a cota manualmente e de forma permanente é antipadrão —
o orçamento pertence ao `026`.

---

## 4. Exemplo — `appsettings.Production.json`

```json
{
  "Aios": {
    "Bus": {
      "Publish":  { "MaxPayloadBytes": 1048576, "ValidateEnvelope": true,
                    "RequireRegisteredSubject": true },
      "Stream":   { "DefaultReplicas": 3, "DefaultMaxAge": "7d",
                    "DefaultMaxBytes": 53687091200, "DedupeWindow": "2m",
                    "DiscardPolicy": "old" },
      "Consumer": { "AckWaitMs": 30000, "MaxDeliver": 5,
                    "BackoffMs": [1000, 5000, 30000, 120000],
                    "MaxAckPending": 1000, "StallThresholdMs": 60000 },
      "Quota":    { "MsgsPerSecTenant": 50000, "MsgsPerSecAgent": 200,
                    "BytesPerSecTenant": 104857600, "MaxSubscriptionsAgent": 64 },
      "Group":    { "FanoutLimit": 256, "DefaultScope": "group" },
      "Request":  { "TimeoutMs": 5000 },
      "A2A":      { "HandshakeTimeoutMs": 3000, "IdleTimeoutMs": 300000,
                    "MaxSessionsAgent": 32, "AllowExternalPeers": false },
      "Dlq":      { "RetentionDays": 30, "AutoReplay": false },
      "Policy":   { "FailMode": "closed" },
      "Cluster":  { "LeafEnabled": false }
    }
  }
}
```

Equivalente por ambiente:

```bash
export AIOS_BUS_PUBLISH_MAX_PAYLOAD_BYTES=1048576
export AIOS_BUS_CONSUMER_MAX_DELIVER=5
export AIOS_BUS_QUOTA_MSGS_PER_SEC_TENANT=50000
export AIOS_BUS_A2A_ALLOW_EXTERNAL_PEERS=false
export AIOS_BUS_DLQ_AUTO_REPLAY=false
```

---

## 5. Configuração do NATS (baseline)

Parâmetros do broker, coerentes com as chaves acima e com `./Deployment.md`:

```conf
# nats-server.conf (extrato)
max_payload: 1MB                    # = bus.publish.max_payload_bytes
max_connections: 65536
max_subscriptions: 0                # sem teto global; o limite é por conta/agente
write_deadline: "10s"

jetstream {
  store_dir: "/data/jetstream"
  max_memory_store: 4GB
  max_file_store: 500GB
}

cluster {
  name: "aios"
  routes: ["nats://aios-nats-1:6222", "nats://aios-nats-2:6222", "nats://aios-nats-3:6222"]
}

tls {
  cert_file: "/etc/nats/certs/server.pem"
  key_file:  "/etc/nats/certs/server-key.pem"
  ca_file:   "/etc/nats/certs/ca.pem"
  verify:    true                   # mTLS
}

# Contas por tenant são geradas pelo TenantAccountManager a partir do operator JWT;
# NÃO são escritas manualmente neste arquivo.
```

---

## 6. Relação Configuração → Requisito

| Chave | Requisito que sustenta |
|-------|------------------------|
| `bus.publish.validate_envelope`, `…require_registered_subject` | FR-002, FR-003 |
| `bus.publish.max_payload_bytes` | FR-015, NFR-001 |
| `bus.stream.default_replicas` | NFR-006 (perda zero) |
| `bus.stream.dedupe_window` | FR-005 |
| `bus.consumer.ack_wait_ms`, `…max_deliver`, `…backoff_ms` | FR-007, NFR-012 |
| `bus.consumer.max_ack_pending`, `…stall_threshold_ms` | FR-011, NFR-011 |
| `bus.quota.*` | FR-010, NFR-004, NFR-011 |
| `bus.quota.max_subscriptions_agent` | NFR-007 (contenção de N²) |
| `bus.group.fanout_limit`, `…default_scope` | FR-012, NFR-016 |
| `bus.request.timeout_ms` | NFR-003 |
| `bus.a2a.*` | FR-013, FR-014, FR-020, NFR-015 |
| `bus.dlq.auto_replay`, `…retention_days` | FR-008 |
| `bus.policy.fail_mode` | FR-017 |

---

## 7. Referências

- Brief: `./_DESIGN_BRIEF.md` §8
- Implantação e topologia: `./Deployment.md`
- Segurança e contas: `./Security.md` · Escala: `./Scalability.md`
