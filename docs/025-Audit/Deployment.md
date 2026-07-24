---
Documento: Deployment
Módulo: 025-Audit
Status: Draft
Versão: 0.1
Última atualização: 2026-07-23
Responsável (RACI-A): Arquiteto do Módulo 025-Audit
ADRs relacionados: ADR-0002, ADR-0003, ADR-0004, ADR-0005, ADR-0251, ADR-0255
RFCs relacionados: RFC-0001, RFC-0250
Depende de: Architecture.md, Configuration.md, 021-Security, 027-Cluster, 028-Deployment
---

# 025-Audit — Implantação

## 1. Topologia

```
   eventos (JetStream)                    API direta (mTLS)
          │                                      │
   ┌──────▼──────────────────────────────────────▼──────┐
   │        audit-ingest  (3–10 réplicas, HPA)          │
   │  normaliza · cifra · encadeia · COMMIT DURÁVEL     │
   └──────────────────┬─────────────────────────────────┘
                      ▼
        ┌─────────────────────────────┐
        │  PostgreSQL (schema audit)  │  synchronous_commit = remote_apply
        │  primário + réplica síncrona│  ◀── o que sustenta o RPO 0
        └──────┬───────────────┬──────┘
               │               │ (réplica de leitura)
   ┌───────────▼──────┐   ┌────▼──────────────┐   ┌────────────────────┐
   │  audit-sealer    │   │  audit-verifier   │   │  audit-archiver    │
   │  (1 por partição │   │  (2 réplicas)     │   │  (2 réplicas)      │
   │   lógica, líder) │   │  verificação      │   │  → MinIO WORM      │
   └────────┬─────────┘   │  em réplica       │   └─────────┬──────────┘
            │             └───────────────────┘             │
            ▼ assina                                        ▼
     021-Security (KMS)                            MinIO (object lock)
            ▲
   ┌────────┴─────────┐        ┌──────────────────────────┐
   │    audit-api     │───────▶│ audit-outbox-relay       │──▶ NATS aios.*.audit.*
   │ consulta·gov·exp │  022   │ (líder único)            │
   └──────────────────┘  PEP   └──────────────────────────┘
```

O `audit-verifier` roda contra a **réplica de leitura**: verificar 10⁹ registros
competindo com a ingestão degradaria o p99 da escrita, que é justamente o caminho que
não pode falhar.

## 2. Contêineres e recursos

| Serviço | Imagem base | Réplicas (prod) | CPU (req/lim) | RAM (req/lim) | Observação |
|---------|-------------|-----------------|---------------|---------------|------------|
| `audit-ingest` | .NET 10 (Alpine) | 3 – 10 (HPA) | 1 / 4 vCPU | 2 / 6 GiB | Custo dominante: hash + cifragem + commit síncrono |
| `audit-sealer` | .NET 10 | 2 (eleição de líder por partição) | 1 / 3 vCPU | 2 / 4 GiB | Merkle é CPU-bound |
| `audit-verifier` | .NET 10 | 2 | 2 / 6 vCPU | 4 / 8 GiB | Lê da réplica; picos na verificação completa |
| `audit-api` | .NET 10 | 2 – 4 | 0,5 / 2 vCPU | 1 / 3 GiB | Baixo volume, alta sensibilidade |
| `audit-archiver` | .NET 10 | 2 | 0,5 / 2 vCPU | 1 / 4 GiB | I/O de objeto |
| `audit-outbox-relay` | .NET 10 | 2 (1 ativa) | 0,25 / 1 vCPU | 256 / 512 MiB | Eleição de líder |

**PostgreSQL**: primário + **réplica síncrona** obrigatória
(`aud.db.synchronous_commit = remote_apply`). Sem a réplica síncrona, o RPO 0 é uma
promessa sem lastro.

**MinIO**: bucket com *object lock* em modo *compliance* habilitado **na criação** —
não é possível habilitá-lo depois, e um bucket sem *object lock* tornaria o
arquivamento uma cópia comum.

### 2.1 Autoescala (HPA)

| Serviço | Métrica | Alvo |
|---------|---------|------|
| `audit-ingest` | CPU | 60% |
| `audit-ingest` | `aios_aud_append_latency_ms` p99 | ≤ 20 ms (NFR-001) |
| `audit-ingest` | lag do consumidor JetStream | < 30 s |
| `audit-api` | p95 de consulta | ≤ 2 s (NFR-012) |

> Escalar o `audit-ingest` **não** resolve latência causada por commit síncrono lento:
> nesse caso o sinal é `aios_aud_commit_latency_ms` e a ação é no banco (rede entre
> primário e réplica, I/O), não em réplicas de aplicação.

## 3. Docker Compose (desenvolvimento)

```yaml
services:
  audit-ingest:
    image: aios/audit-ingest:0.1
    environment:
      AIOS_AUD_INGEST_REQUIRE_DURABLE_ACK: "true"   # true em TODOS os ambientes
      AIOS_AUD_CHAIN_PARTITIONS_PER_TENANT: "1"
      AIOS_AUD_DB_SYNCHRONOUS_COMMIT: "remote_apply"
      AIOS_AUD_DB_CONNECTION_STRING: "Host=postgres;Database=aios;Username=audit"
      AIOS_AUD_NATS_URL: "nats://nats:4222"
    depends_on: [postgres, nats]
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8080/healthz/ready"]
      interval: 10s
      timeout: 2s
      retries: 3

  audit-sealer:
    image: aios/audit-sealer:0.1
    environment:
      AIOS_AUD_SEAL_INTERVAL_S: "10"
    depends_on: [postgres]

  audit-verifier:
    image: aios/audit-verifier:0.1
    environment:
      AIOS_AUD_VERIFY_INTERVAL_S: "60"
      AIOS_AUD_VERIFY_SAMPLE_SIZE: "100"
    depends_on: [postgres, minio]

  audit-api:
    image: aios/audit-api:0.1
    environment:
      AIOS_AUD_POLICY_FAIL_MODE: "open"             # apenas local
      AIOS_AUD_HOLD_REQUIRE_DUAL_APPROVAL: "false"  # apenas local
    ports: ["8080:8080", "8081:8081"]

  audit-archiver:
    image: aios/audit-archiver:0.1
    environment:
      AIOS_AUD_WORM_OBJECT_LOCK_DAYS: "1"
      AIOS_AUD_WORM_BUCKET: "aios-audit-dev"
    depends_on: [minio]

  minio:
    image: minio/minio
    command: server /data
    environment:
      MINIO_OBJECT_LOCKING: "on"      # DEVE estar ligado na CRIAÇÃO do bucket
```

## 4. Health e readiness

| Endpoint | Verifica | Falha significa |
|----------|----------|-----------------|
| `GET /healthz/live` | Processo respondendo | Reiniciar contêiner |
| `GET /healthz/ready` (`audit-ingest`) | (a) PostgreSQL **primário e réplica síncrona** alcançáveis; (b) último `record_hash` da partição legível; (c) chave de cifragem disponível | Retirar do balanceamento — **melhor recusar que aceitar sem durabilidade** |
| `GET /healthz/ready` (`audit-api`) | PostgreSQL alcançável; PDP alcançável | Retirar do balanceamento |
| `GET /healthz/ready` (`audit-sealer`) | Chave de assinatura disponível no `../021-Security/` | Selagem adia; **ingestão não é afetada** |
| `GET /healthz/startup` | Migrações aplicadas; *trigger* de imutabilidade presente | Ainda inicializando |

> **Regra crítica de readiness do `audit-ingest`:** se a **réplica síncrona** não
> estiver alcançável, a réplica de aplicação **NÃO DEVE** ficar pronta. Aceitar
> escrita sem replicação confirmada quebraria o RPO 0 silenciosamente — e o produtor
> teria recebido `201` acreditando que o fato estava seguro.
>
> O *startup* também verifica a **presença do `tr_record_immutable`**
> (`./Database.md` §2.1). Subir sem o *trigger* significaria que a imutabilidade
> voltou a ser convenção.

## 5. Segredos e identidade

| Segredo | Origem | Rotação | Injeção |
|---------|--------|---------|---------|
| Credencial PostgreSQL | Credencial dinâmica do `../021-Security/` (`db_role`) | ≤ 1 h | Arquivo em `tmpfs` |
| **Chave de assinatura de selo** | `../021-Security/` (`signing_key`) | 90 d, com sobreposição | **HSM/KMS — nunca sai do 021** |
| Chaves de cifragem por titular | `../021-Security/` (KEK/DEK) | Sob demanda (destruição no RTBF) | Referência apenas; material no KMS |
| Conta NKey/JWT do NATS | `../021-Security/` (`nats_account`) | 7 d | Arquivo em `tmpfs` |
| Certificado mTLS de workload | PKI do `../021-Security/`, após atestação | ≤ 24 h | Socket local do agente de identidade |
| Credencial MinIO (WORM) | `../021-Security/` | 90 d | Arquivo em `tmpfs` |

A **chave de assinatura de selo** é o segredo mais sensível do módulo — e por isso
**não vive aqui**. O `audit-sealer` envia o `seal_hash` ao `../021-Security/` e recebe
a assinatura; a chave privada nunca entra no processo. Comprometer o 025 não basta
para forjar um selo (invariante estrutural S3).

Rotação da chave de assinatura **não invalida selos antigos**: `signed_by` registra
qual chave assinou cada selo, e o `../021-Security/` mantém as chaves públicas
históricas publicadas — o auditor de 2032 ainda verifica o selo de 2026.

## 6. Estratégia de rollout

| Etapa | Ação | Critério de avanço |
|-------|------|--------------------|
| 1 | Aplicar migrações (expand, CV-11) | *Trigger* de imutabilidade entra **junto** com a tabela (`./Database.md` §13) |
| 2 | Rollout do `audit-ingest` (canário, 1 réplica) | p99 ≤ 20 ms; **0** `AIOS-AUD-0005` fora de janela de falha real |
| 3 | Rollout do `audit-sealer` | Selo produzido no intervalo; assinatura verificável |
| 4 | Rollout de `audit-verifier`, `audit-api`, `audit-archiver` | Verificação contínua passa; **0** `chain_break` |
| 5 | Verificação completa de uma partição | Cadeia íntegra ponta a ponta antes de encerrar o rollout |

**Regra de coexistência de versões:** durante o rollout, réplicas de versões diferentes
**DEVEM** produzir o **mesmo `record_hash`** para a mesma entrada. Qualquer mudança na
serialização canônica (RFC-0250) é mudança de contrato e **NÃO PODE** entrar por
rollout comum — exige nova versão de RFC e um período em que ambas as formas sejam
verificáveis.

> **Rollback de deploy não desfaz registros.** Voltar a imagem anterior é seguro; os
> registros escritos pela versão nova permanecem, encadeados e válidos. O que **não** é
> seguro é voltar a uma versão que calcule `record_hash` de forma diferente — daí a
> regra acima.

## 7. Ordem de bootstrap da instalação

```
   1. 005-Database  → schema `audit`, migrações, TRIGGER de imutabilidade
                      + réplica síncrona configurada (synchronous_commit)
   2. 021-Security  → PKI ativa; chave de assinatura de selo disponível
   3. MinIO         → bucket com OBJECT LOCK habilitado na criação
   4. 020-Comm      → streams AUDIT_SEAL / AUDIT_ANOMALY / AUDIT_GOVERNANCE
   5. 025 ingest    → registra as classes auditáveis de plataforma
   6. demais módulos→ passam a emitir fatos auditáveis
   7. 022-Policy    → PEP de consulta passa a autorizar
```

> Entre os passos 5 e 6, o AIOS já registra a própria instalação. Subir os módulos de
> negócio antes do 025 significa que os fatos do bootstrap — justamente os de maior
> interesse forense, como a primeira publicação de política e a emissão das primeiras
> credenciais — **não teriam registro**. A ordem é parte do modelo de conformidade.

## 8. Dependências de implantação

| Dependência | Tipo | Falha na implantação significa |
|-------------|------|-------------------------------|
| PostgreSQL primário + **réplica síncrona** | **Forte** | `audit-ingest` não fica *ready*; produtores recebem `503` e retêm no outbox |
| `../021-Security/` (chave de assinatura) | Forte para selagem | Ingestão continua; selagem adia |
| MinIO com *object lock* | Forte para arquivamento | Ingestão e selagem continuam; alerta se o atraso exceder o prazo |
| NATS/JetStream | Forte para ingestão por evento | API direta continua; JetStream retém e reentrega |
| `../022-Policy/` | Forte para consulta | Consulta e governança negadas; **ingestão não é afetada** |
| `../024-Observability/` | Fraca | Sobe sem telemetria — **DEVE** alertar |

## 9. Referências

- Arquitetura: `./Architecture.md` · Configuração: `./Configuration.md`
- Falhas: `./FailureRecovery.md` · Escala: `./Scalability.md`
- Plataforma: `../027-Cluster/`, `../028-Deployment/`, `../029-Operations/`
- Segredos e identidade: `../021-Security/`

*Fim de `Deployment.md`.*
