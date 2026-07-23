---
Documento: Deployment
Módulo: 020-Communication
Status: Draft
Versão: 0.1
Última atualização: 2026-07-22
Responsável (RACI-A): Arquiteto do Módulo 020-Communication
ADRs relacionados: ADR-0004, ADR-0201, ADR-0203
RFCs relacionados: RFC-0001, RFC-0200
Depende de: 028-Deployment, 027-Cluster, 021-Security, 005-Database, 024-Observability
---

# 020-Communication — Implantação

Empacotamento **Docker/Compose** (stack de referência do `../028-Deployment/`). Este
documento descreve a topologia do barramento; a topologia global do AIOS pertence ao
`028`, e as decisões de HA/DR multi-região ao `../027-Cluster/`.

---

## 1. Topologia

```
                       ┌────────────────────────────────┐
                       │        004-API (YARP)          │  ← admin, /v1/comm
                       └───────────────┬────────────────┘
                       ┌───────────────▼────────────────┐
                       │   aios-comm-svc  × 2           │  .NET 10, stateless
                       │   (registro, streams, A2A,     │  jobs: ociosidade A2A,
                       │    DLQ, contas, replay)        │        varredura de DLQ
                       └───┬────────────────────────┬───┘
                           │ admin (declarações)    │ PostgreSQL (schema comm)
                           ▼                        ▼ Redis (contadores de cota)
   ┌───────────────────────────────────────────────────────────────────┐
   │            aios-nats × 3   (cluster, JetStream R=3)               │
   │   ┌────────────┐  ┌─────────────┐  ┌──────────────┐               │
   │   │ conta acme │  │ conta globex│  │ conta _platform│             │
   │   └────────────┘  └─────────────┘  └──────────────┘               │
   └───┬─────────────────────┬──────────────────────────┬──────────────┘
       │ 4222 (clientes)     │ 6222 (rotas)             │ 7422 (leaf)
       ▼                     ▼                          ▼
   módulos 006…026     replicação entre nós       aios-nats-leaf (0..N)
   + agentes (007)                                 (habilitado por 027)
```

**Portas.** `4222` clientes (TLS), `6222` rotas de cluster, `7422` leaf nodes, `8222`
monitoramento HTTP (interno, raspado pelo Prometheus). A porta `4222` **NÃO DEVE** ser
exposta fora da malha — pares A2A externos entram pelo `../004-API/` ou por leaf node
dedicado com filtro de propagação.

---

## 2. Contêineres e Recursos

| Contêiner | Imagem base | CPU (req/lim) | RAM (req/lim) | Disco | Réplicas |
|-----------|-------------|---------------|---------------|-------|----------|
| `aios-nats` | `nats:2.10-alpine` | 2 / 8 | 4 GiB / 16 GiB | SSD NVMe ≥ 500 GiB (JetStream) | **3** (quórum) |
| `aios-comm-svc` | `mcr.microsoft.com/dotnet/aspnet:10` | 0,5 / 2 | 512 MiB / 2 GiB | — | 2 |
| `aios-nats-leaf` | `nats:2.10-alpine` | 1 / 4 | 2 GiB / 8 GiB | SSD ≥ 100 GiB | 0..N |

**Regras de dimensionamento.** O número de nós NATS **DEVE** ser ímpar (3 ou 5) para
que o quórum de JetStream seja bem definido — quatro nós não oferecem tolerância maior
que três e dobram o custo de replicação. `max_memory_store` (4 GiB) cobre streams
efêmeros; `max_file_store` (500 GiB) é o teto agregado de todos os streams duráveis e
**DEVE** ser maior que a soma dos `max_bytes` declarados.

---

## 3. Ordem de Inicialização e Dependências

```
   1. cofre de segredos (021) — operator JWT e credenciais de conta
   2. PostgreSQL (005) — schema comm disponível e migrado
   3. aios-nats × 3    — cluster formado, quórum de JetStream estabelecido
   4. aios-comm-svc    — reconcilia declarações (streams/consumidores) e sobe
   5. módulos 006…026  — conectam com a credencial de sua conta
   6. agentes (007)    — conectam via runtime, sob cota por agente
```

Um módulo consumidor **NÃO DEVE** iniciar antes de o `aios-comm-svc` reportar `ready`:
seu consumidor durável pode ainda não ter sido materializado.

A **reconciliação** do passo 4 é declarativa e idempotente: o serviço lê
`comm.stream`/`comm.consumer` e garante que o NATS reflita a declaração — criando o que
falta, aplicando mudanças compatíveis e sinalizando as incompatíveis para migração
espelhada (ADR-0203).

---

## 4. Health e Readiness

| Endpoint / verificação | Contêiner | Critério |
|------------------------|-----------|----------|
| `GET /healthz` (liveness) | `aios-comm-svc` | Processo responde. |
| `GET /readyz` (readiness) | `aios-comm-svc` | Conexão ao NATS **e** ao PostgreSQL OK **e** reconciliação concluída. |
| `GET :8222/healthz` | `aios-nats` | Servidor saudável. |
| `GET :8222/jsz` | `aios-nats` | JetStream ativo, com quórum e streams em `healthy`. |
| `GET :8222/routez` | `aios-nats` | Rotas de cluster estabelecidas com os pares. |

Readiness do `aios-comm-svc` **NÃO DEVE** depender do PDP (`022`): a indisponibilidade
de política nega **operações novas** (`fail-closed`), mas não pode impedir o serviço de
subir e continuar operando o tráfego já autorizado (princípio P-09 de `./Vision.md`).

---

## 5. Estratégia de Rollout

| Componente | Estratégia | Observação |
|------------|-----------|------------|
| `aios-comm-svc` | *Rolling* (1 por vez) | Stateless; a reconciliação é idempotente. |
| `aios-nats` | *Rolling* **um nó por vez**, aguardando `jsz` reportar streams `healthy` | Derrubar dois nós simultaneamente perde o quórum e para a publicação durável. |
| Declaração de stream | Compatível: em lugar. Incompatível: migração espelhada (ADR-0203). | Nunca recriação destrutiva — perderia o backlog não consumido. |
| Registro de subject | Aditivo; depreciação com coexistência ≥ 2 majors. | RFC-0001 §5.7. |
| Leaf nodes | Independente do cluster central | Conforme `../027-Cluster/`. |

**Regra de ouro do rollout do NATS:** entre a parada de um nó e a do seguinte, o
`jsz` **DEVE** reportar todos os streams com réplicas em dia. Ignorar essa espera é a
forma mais comum de transformar uma atualização de rotina em incidente de perda de
quórum.

---

## 6. Volumes e Persistência

| Volume | Montagem | Conteúdo | Backup |
|--------|----------|----------|--------|
| `natsdata-N` | `/data/jetstream` | Streams duráveis do nó N | Snapshot de stream (§7) |
| — | — | Declarações e DLQ | Em PostgreSQL, coberto pelo backup do `../005-Database/` |

O estado durável do barramento tem **duas** naturezas: as **mensagens** (em JetStream,
replicadas R=3) e as **declarações** (em PostgreSQL, com backup e PITR do `005`).
Perder um volume de JetStream é recuperável pela replicação; perder o PostgreSQL sem
backup significaria perder o catálogo — e é por isso que a declaração vive lá, não em
arquivos de configuração do broker.

---

## 7. Backup e Restauração

| Artefato | Método | Frequência | Alvo |
|----------|--------|-----------|------|
| Streams críticos (`AUDIT_TRAIL`, `COMM_PLATFORM`) | `nats stream backup` → MinIO | diária | RPO ≤ 5 min complementado pela replicação R=3 |
| Declarações (`comm.*`) | Backup/PITR do `../005-Database/` | contínuo (WAL) | RPO ≤ 5 min |
| Operator/account JWT | Cofre do `../021-Security/` | conforme política do `021` | — |

Restauração de stream é procedimento de `../029-Operations/`; o RTO alvo é **≤ 15 min**
(NFR-009).

---

## 8. Extrato de `compose.yaml`

```yaml
services:
  aios-nats-1: &nats
    image: nats:2.10-alpine
    command: >
      -c /etc/nats/nats-server.conf
      --name aios-nats-1
      --cluster nats://0.0.0.0:6222
      --routes nats://aios-nats-2:6222,nats://aios-nats-3:6222
      --http_port 8222
    volumes:
      - natsdata-1:/data/jetstream
      - ./nats-server.conf:/etc/nats/nats-server.conf:ro
      - ./certs:/etc/nats/certs:ro
    healthcheck:
      test: ["CMD", "wget", "-qO-", "http://localhost:8222/healthz"]
      interval: 10s
      retries: 6

  aios-nats-2:
    <<: *nats
    command: >
      -c /etc/nats/nats-server.conf --name aios-nats-2
      --cluster nats://0.0.0.0:6222
      --routes nats://aios-nats-1:6222,nats://aios-nats-3:6222 --http_port 8222
    volumes: [ "natsdata-2:/data/jetstream", "./nats-server.conf:/etc/nats/nats-server.conf:ro", "./certs:/etc/nats/certs:ro" ]

  aios-nats-3:
    <<: *nats
    command: >
      -c /etc/nats/nats-server.conf --name aios-nats-3
      --cluster nats://0.0.0.0:6222
      --routes nats://aios-nats-1:6222,nats://aios-nats-2:6222 --http_port 8222
    volumes: [ "natsdata-3:/data/jetstream", "./nats-server.conf:/etc/nats/nats-server.conf:ro", "./certs:/etc/nats/certs:ro" ]

  aios-comm-svc:
    image: aios/comm-svc:0.1
    environment:
      AIOS_BUS_PUBLISH_MAX_PAYLOAD_BYTES: "1048576"
      AIOS_BUS_CONSUMER_MAX_DELIVER: "5"
      AIOS_BUS_QUOTA_MSGS_PER_SEC_TENANT: "50000"
      AIOS_BUS_DLQ_AUTO_REPLAY: "false"
      AIOS_BUS_POLICY_FAIL_MODE: "closed"
      OTEL_EXPORTER_OTLP_ENDPOINT: "http://otel-collector:4317"
    depends_on: [ aios-nats-1, aios-nats-2, aios-nats-3, aios-postgres-primary ]
    deploy: { replicas: 2 }
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8080/readyz"]
      interval: 15s

volumes:
  natsdata-1:
  natsdata-2:
  natsdata-3:
```

---

## 9. Ambientes

| Ambiente | Diferenças |
|----------|-----------|
| `dev` (local) | 1 nó NATS, JetStream R=1, backup desligado. Contas por tenant **permanecem ativas** — desligá-las mascararia erros de isolamento que só apareceriam em produção. |
| `staging` | 3 nós, R=3, volume reduzido; é onde os testes de contenção e de contrato rodam no CI. |
| `prod` | Topologia completa (§1), R=3 em domínios de falha distintos, leaf nodes conforme `../027-Cluster/`. |

---

## 10. Referências

- Topologia global: `../028-Deployment/` · HA/DR e supercluster: `../027-Cluster/`
- Configuração: `./Configuration.md` · Modelo físico: `./Database.md`
- Monitoramento: `./Monitoring.md` · Falhas: `./FailureRecovery.md`
- Runbooks: `../029-Operations/`
