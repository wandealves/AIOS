---
Documento: Deployment
Módulo: 022-Policy
Status: Draft
Versão: 0.1
Última atualização: 2026-07-23
Responsável (RACI-A): Arquiteto do Módulo 022-Policy
ADRs relacionados: ADR-0002, ADR-0003, ADR-0004, ADR-0005, ADR-0006, ADR-0222
RFCs relacionados: RFC-0001, RFC-0221
Depende de: Architecture.md, Configuration.md, 021-Security, 027-Cluster, 028-Deployment
---

# 022-Policy — Implantação

## 1. Topologia

```
                       ┌──────────────────────────────┐
      PEPs (gRPC/mTLS) │      YARP / API Gateway      │  REST externo (/v1/policy)
      ────────────────▶│         (004-API)            │◀──────────────── console/CLI
                       └───────────────┬──────────────┘
                                       │ mTLS
                ┌──────────────────────┼──────────────────────┐
                ▼                      ▼                      ▼
        ┌───────────────┐     ┌───────────────┐      ┌───────────────┐
        │ policy-api #1 │     │ policy-api #2 │      │ policy-api #3 │
        │ BundleRuntime │     │ BundleRuntime │      │ BundleRuntime │
        │  (v43 em RAM) │     │  (v43 em RAM) │      │  (v43 em RAM) │
        └───────┬───────┘     └───────┬───────┘      └───────┬───────┘
                └─────────────────────┼──────────────────────┘
                                      │
         ┌────────────────┬───────────┼────────────┬────────────────┐
         ▼                ▼           ▼            ▼                ▼
   ┌───────────┐   ┌───────────┐  ┌────────┐  ┌──────────┐  ┌──────────────┐
   │PostgreSQL │   │  Redis    │  │ NATS   │  │ 021 (PKI,│  │ 024 OTel     │
   │schema     │   │ cache attr│  │JetStrm │  │ segredos)│  │ collector    │
   │ policy    │   │ + limites │  │POLICY_*│  └──────────┘  └──────────────┘
   └───────────┘   └───────────┘  └────────┘

   ┌──────────────────────┐        ┌────────────────────────┐
   │  policy-simulator    │        │  policy-outbox-relay   │
   │  (1–2 réplicas)      │        │  (1 réplica ativa)     │
   │  pool DEDICADO       │        │  outbox → JetStream    │
   └──────────────────────┘        └────────────────────────┘
```

O `policy-simulator` roda em **pool dedicado** para que a carga analítica não dispute
CPU com o caminho quente (NFR-012). O `policy-outbox-relay` opera com eleição de líder
(uma instância publica; as demais ficam em espera) — publicar em paralelo produziria
duplicatas que os consumidores teriam de deduplicar por `event.id` desnecessariamente.

## 2. Contêineres e recursos

| Serviço | Imagem base | Réplicas (prod) | CPU (req/lim) | RAM (req/lim) | Observação |
|---------|-------------|-----------------|---------------|---------------|------------|
| `policy-api` | .NET 10 runtime (Alpine) | 3 (mín.) – 20 (HPA) | 1 / 4 vCPU | 2 / 6 GiB | RAM dominada pelo bundle compilado (≤ 64 MiB/bundle) + índices |
| `policy-simulator` | .NET 10 runtime | 1 – 2 | 2 / 8 vCPU | 4 / 12 GiB | Job-like; escala sob demanda |
| `policy-outbox-relay` | .NET 10 runtime | 2 (1 ativa) | 0,25 / 1 vCPU | 256 / 512 MiB | Eleição de líder |

Sem GPU. O custo dominante do `policy-api` é **CPU** (avaliação de expressões
compiladas), não I/O — coerente com a decisão de manter o bundle em memória
(ADR-0222).

### 2.1 Autoescala (HPA)

| Métrica | Alvo | Comentário |
|---------|------|-----------|
| CPU | 60% | Sinal primário |
| `aios_pol_decision_latency_ms` p99 | 5 ms (NFR-001) | Escala antes de violar o SLO |
| `aios_pol_decisions_total` por réplica | 40.000/s | 80% do limite de NFR-002 |

Escalar **não** ajuda quando a causa é fonte de atributo lenta: nesse caso o sinal
correto é `aios_pol_attribute_resolve_ms` e a ação é isolar a fonte, não adicionar
réplicas (`./FailureRecovery.md` §3).

## 3. Docker Compose (desenvolvimento)

```yaml
services:
  policy-api:
    image: aios/policy-api:0.1
    environment:
      AIOS_POL_DECISION_EVAL_BUDGET_MS: "50"
      AIOS_POL_ATTRIBUTE_FAIL_MODE: "open"       # apenas local (Configuration §9)
      AIOS_POL_CONFLICT_MODE: "warn"
      AIOS_POL_TEST_REQUIRED_BEFORE_PUBLISH: "false"
      AIOS_POL_DB_CONNECTION_STRING: "Host=postgres;Database=aios;Username=policy"
      AIOS_POL_REDIS_CONNECTION_STRING: "redis:6379"
      AIOS_POL_NATS_URL: "nats://nats:4222"
      AIOS_POL_OTEL_ENDPOINT: "http://otel-collector:4317"
    depends_on: [postgres, redis, nats]
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8080/healthz/ready"]
      interval: 10s
      timeout: 2s
      retries: 3
    ports: ["8080:8080", "8081:8081"]   # 8080 REST, 8081 gRPC

  policy-simulator:
    image: aios/policy-simulator:0.1
    environment:
      AIOS_POL_SIMULATION_MAX_REQUESTS: "100000"
    depends_on: [postgres]

  policy-outbox-relay:
    image: aios/policy-outbox-relay:0.1
    environment:
      AIOS_POL_OUTBOX_POLL_INTERVAL_MS: "200"
    depends_on: [postgres, nats]
```

## 4. Health e readiness

| Endpoint | Verifica | Falha significa |
|----------|----------|-----------------|
| `GET /healthz/live` | Processo respondendo | Reiniciar contêiner |
| `GET /healthz/ready` | (a) bundle ativo carregado e **assinatura verificada**; (b) PostgreSQL alcançável; (c) Redis alcançável; (d) NATS conectado | Retirar do balanceamento |
| `GET /healthz/startup` | Migrações aplicadas + primeiro bundle carregado | Ainda inicializando |

> **Regra crítica de readiness:** uma réplica que não conseguiu **verificar a
> assinatura** do bundle ativo **NÃO DEVE** ficar pronta (NFR-015). Servir decisões
> com artefato não verificado é pior do que não servir: a indisponibilidade é
> detectada em segundos; a política adulterada, talvez nunca.
>
> Da mesma forma, uma réplica cuja `aios_pol_active_bundle_version` divirja da versão
> ativa no registro por mais de `pol.bundle.propagation_target_s` **DEVE** sair do
> balanceamento (`./FailureRecovery.md` §3).

## 5. Segredos e identidade

| Segredo | Origem | Rotação | Injeção |
|---------|--------|---------|---------|
| Credencial PostgreSQL | Credencial dinâmica do `../021-Security/` (`db_role`) | ≤ 1 h | Arquivo em `tmpfs` |
| Credencial Redis | `../021-Security/` | 90 d | Variável de ambiente |
| Conta NKey/JWT do NATS | `../021-Security/` (`nats_account`) | 7 d | Arquivo em `tmpfs` |
| Certificado mTLS de workload | PKI do `../021-Security/`, após atestação | ≤ 24 h | Socket local do agente de identidade |
| Chave de verificação de assinatura de bundle | JWKS/chave pública do `../021-Security/` | por rotação | Cache em memória, recarregado por evento |

Nenhum segredo é embutido em imagem ou em código (`../021-Security/` §12.3).

## 6. Estratégia de rollout

| Etapa | Ação | Critério de avanço |
|-------|------|--------------------|
| 1 | Aplicar migrações (expand, CV-11) | DDL < 200 ms; sem bloqueio |
| 2 | Rollout **canário**: 1 réplica na nova versão | p99 ≤ 5 ms e taxa de `deny` estável por 10 min |
| 3 | Rollout progressivo (25% → 50% → 100%) | Sem alerta `PolicyDenyRateSpike` nem `PolicyBundleVersionSkew` |
| 4 | Contract das migrações | Após 2 releases sem rollback |

**Regra de coexistência de versão de código:** durante o rollout, réplicas de versões
diferentes de *software* **DEVEM** servir a **mesma** versão de *bundle*. Mudança no
formato do artefato compilado exige recompilar e republicar o bundle antes do rollout
— nunca reinterpretar o artefato antigo com o motor novo (isso quebraria NFR-009).

**Rollback de deploy** é independente do rollback de política (`./UseCases.md` UC-007):
o primeiro troca a imagem; o segundo troca o bundle. Confundi-los durante um incidente
é o erro operacional mais provável — o runbook em `./Monitoring.md` §5 separa
explicitamente os dois.

## 7. Ordem de bootstrap da instalação

```
   1. 005-Database  → schema `policy` criado, migrações aplicadas
   2. 021-Security  → PKI ativa, chave de assinatura disponível
   3. 020-Comm      → streams POLICY_BUNDLE / POLICY_DECISION / POLICY_GOVERNANCE
   4. 022-Policy    → sobe, publica o BUNDLE-RAIZ (scope=platform) por T-09
                      (bootstrap, aprovação dupla registrada)
   5. demais módulos→ PEPs passam a consultar o PDP
```

> Antes do passo 4, **não há política ativa** — e, pela invariante I5, o PDP nega
> tudo. Isso é intencional: um AIOS que sobe sem política não sobe permissivo, sobe
> inerte. A ordem de bootstrap é, portanto, parte do modelo de segurança e está
> refletida nas dependências de implantação de `../028-Deployment/`.

## 8. Dependências de implantação

| Dependência | Tipo | Falha na implantação significa |
|-------------|------|-------------------------------|
| PostgreSQL (`policy`) | Forte | Não sobe (readiness falha) |
| `../021-Security/` (PKI + chave de assinatura) | Forte | Não sobe: não é possível verificar bundle |
| NATS/JetStream | Forte para propagação | Sobe, mas propagação e invalidação degradam |
| Redis | Fraca | Sobe; cache de atributo em memória local, com mais chamadas externas |
| `../024-Observability/` | Fraca | Sobe sem telemetria — **DEVE** alertar |
| `../025-Audit/` | Forte por conformidade | Sobe, mas `pol.audit.emit_enabled` desligado exige exceção documentada |

## 9. Referências

- Arquitetura: `./Architecture.md` · Configuração: `./Configuration.md`
- Falhas: `./FailureRecovery.md` · Escala: `./Scalability.md`
- Plataforma: `../027-Cluster/`, `../028-Deployment/`, `../029-Operations/`
- Segredos e identidade: `../021-Security/`

*Fim de `Deployment.md`.*
