---
Documento: Deployment
Módulo: 021-Security
Status: Draft
Versão: 0.1
Última atualização: 2026-07-22
Responsável (RACI-A): Arquiteto do Módulo 021-Security
ADRs relacionados: ADR-0211, ADR-0214, ADR-0215, ADR-0216
RFCs relacionados: RFC-0001, RFC-0211
Depende de: 028-Deployment, 027-Cluster, 005-Database, 020-Communication, 029-Operations
---

# 021-Security — Implantação

Empacotamento **Docker/Compose** (stack de referência do `../028-Deployment/`). Este
módulo tem duas particularidades de implantação que o distinguem dos demais: ele é
**dependência de bootstrap** de todo o AIOS e possui um componente que **não é um
contêiner** (a CA raiz).

---

## 1. Topologia

```
                       ┌────────────────────────────────┐
                       │        004-API (YARP)          │
                       │  valida JWT localmente (JWKS)  │  ← NÃO chama o 021 por requisição
                       └───────────────┬────────────────┘
                                       │ /v1/security (admin) + /.well-known/*
                       ┌───────────────▼────────────────┐
                       │   aios-security-svc  × 3       │  .NET 10, stateless
                       │   OIDC · cofre · revogação ·   │  jobs: rotação, expiração,
                       │   perfis · federação            │        varredura anti-vazamento
                       └───┬────────────────────┬───────┘
                           │ gRPC (mTLS)        │
                           ▼                    ├──▶ PostgreSQL (schema security)
                 ┌──────────────────┐           ├──▶ Redis (contenção authn, cache CRL)
                 │ aios-security-ca │           └──▶ NATS (aios.*.security.*)
                 │  × 2 (isolado)   │
                 └────────┬─────────┘
                          │ intermediária assinada por
                          ▼
                 ╔══════════════════════════════════════╗
                 ║  CA RAIZ — OFFLINE                   ║
                 ║  HSM ou custódia física              ║
                 ║  Sem interface de rede. Sem exceção. ║
                 ╚══════════════════════════════════════╝
```

O `aios-security-ca` roda em **processo e contêiner separados** porque a superfície que
emite certificados não deve compartilhar espaço de memória com a que serve endpoints
OIDC expostos. Um comprometimento do endpoint público não deveria dar acesso direto à
chave da CA intermediária.

---

## 2. Contêineres e Recursos

| Contêiner | Imagem base | CPU (req/lim) | RAM (req/lim) | Réplicas | Observação |
|-----------|-------------|---------------|---------------|----------|------------|
| `aios-security-svc` | `mcr.microsoft.com/dotnet/aspnet:10` | 1 / 4 | 1 GiB / 4 GiB | **3** | Assinatura é o custo dominante de CPU. |
| `aios-security-ca` | `mcr.microsoft.com/dotnet/aspnet:10` | 0,5 / 2 | 512 MiB / 2 GiB | 2 | Rede restrita: apenas gRPC do `security-svc`. |
| HSM / KMS externo | — | — | — | externo | Recomendado em produção (`sec.key.hsm_enabled`). |

**Por que 3 réplicas e não 2.** O SLO deste módulo é 99,99% (NFR-005) — cinco vezes
mais exigente que o dos demais. Com duas réplicas, uma atualização contínua deixa o
sistema em capacidade única durante a janela; com três, sobra margem para uma falha
concomitante.

---

## 3. Ordem de Inicialização — o problema do bootstrap

```
   0. CA raiz: existe previamente (cerimônia de instalação). NÃO sobe.
   1. HSM/KMS externo disponível
   2. PostgreSQL (005) com o schema security migrado
   3. aios-security-ca   → carrega intermediária; pronto para emitir
   4. aios-security-svc  → carrega KEK, chaves de assinatura; publica JWKS
   5. NATS (020)         → usa credencial NKey emitida no passo 4 (ou de bootstrap)
   6. 004-API            → busca JWKS; a partir daqui valida tokens localmente
   7. demais módulos     → obtêm certificados de workload e credenciais dinâmicas
```

Este é o único ponto do AIOS com **dependência circular real**, e ela é resolvida
explicitamente:

- O `021` precisa do PostgreSQL (`005`), que precisa de uma credencial de banco — que
  seria emitida pelo `021`. A saída é a **credencial de bootstrap**
  (`./Configuration.md` §2): provisionada na instalação, única, de rotação manual e
  monitorada como item de risco permanente.
- O `021` publica eventos no NATS (`020`), que precisa de contas emitidas pelo `021`.
  A conta `_platform` inicial também é de bootstrap; as demais são emitidas normalmente
  depois.

Nenhuma dessas exceções é "temporária": elas são inerentes e, por isso, documentadas,
auditadas e revisadas periodicamente em vez de esquecidas.

---

## 4. Health e Readiness

| Endpoint | Contêiner | Critério |
|----------|-----------|----------|
| `GET /healthz` | `aios-security-svc` | Processo responde. |
| `GET /readyz` | `aios-security-svc` | PostgreSQL acessível **e** KMS/HSM respondendo **e** chave de assinatura `active` carregada **e** JWKS publicável. |
| `GET /healthz` | `aios-security-ca` | Intermediária carregada e válida (não expirada). |
| `GET /.well-known/jwks.json` | externo | Deve retornar ≥ 1 chave; usado como *smoke test* pelo `004`. |

Readiness **NÃO DEVE** depender do PDP (`022`): com o PDP fora, operações
administrativas são negadas (*fail-closed*), mas autenticação e validação continuam —
e um serviço que se recusa a subir por isso converteria a falha de política em
indisponibilidade total (princípio P-08).

---

## 5. Estratégia de Rollout

| Componente | Estratégia | Observação |
|------------|-----------|------------|
| `aios-security-svc` | *Rolling*, 1 réplica por vez, com `readyz` verde antes da próxima | Nunca abaixo de 2 réplicas ativas. |
| `aios-security-ca` | *Rolling*, 1 por vez | A emissão de certificados pode aguardar segundos; a validação não é afetada. |
| Rotação de chave de assinatura | Publicar → assinar → aposentar (UC-009) | **Nunca** assinar antes de publicar. |
| Rotação de KEK | Cerimônia com aprovação dupla (UC-010) | Fora de janela de rollout. |
| Migração de schema `security` | Expand/contract via `../005-Database/` | Fase `contract` só após todas as réplicas atualizadas. |
| Novo perfil de sandbox | Publicação aditiva (nova versão) | Perfis antigos permanecem consultáveis para auditoria. |

O módulo consome `aios._platform.deployment.rollout.started` e **suspende rotações de
credencial** durante a janela: rotacionar credencial enquanto instâncias antigas e novas
coexistem multiplica as combinações de falha sem necessidade.

---

## 6. Rede e Isolamento

| Fluxo | Origem → Destino | Protocolo | Restrição |
|-------|------------------|-----------|-----------|
| OIDC público | `004-API` → `security-svc` | HTTPS | Único fluxo alcançável a partir da borda. |
| Administração | `004-API` → `security-svc` | gRPC/mTLS | Sob capability do PDP. |
| Emissão de certificado | `security-svc` → `security-ca` | gRPC/mTLS | **Apenas** o `security-svc` alcança a CA. |
| KMS | `security-svc`, `security-ca` → HSM | TLS/PKCS#11 | Rede segregada. |
| Dados | `security-svc` → PostgreSQL | TLS | *Role* `security_rw`; nenhum outro módulo tem acesso ao schema. |
| Eventos | `security-svc` → NATS | TLS + NKey | Conta `_platform`. |

O `aios-security-ca` **não** tem rota de entrada a partir da borda, nem de outros
módulos: sua única origem autorizada é o `security-svc`.

---

## 7. Volumes e Persistência

| Volume | Conteúdo | Backup |
|--------|----------|--------|
| — (nenhum no `security-svc`) | Serviço é *stateless*. | — |
| Material da intermediária | Em HSM ou montado como segredo do orquestrador. | Custódia, não backup convencional. |
| Schema `security` | Princípios, credenciais, segredos cifrados, chaves (metadados). | Backup/PITR do `../005-Database/` |

O backup do schema `security` **não** compromete confidencialidade: os segredos estão
cifrados por envelope e as chaves privadas ou estão envolvidas pela KEK (que não está
no backup) ou vivem no HSM. Restaurar o banco sem a KEK produz dados inúteis — o que é
exatamente o comportamento desejado.

---

## 8. Extrato de `compose.yaml`

```yaml
services:
  aios-security-ca:
    image: aios/security-ca:0.1
    environment:
      AIOS_SEC_KEY_HSM_ENABLED: "true"
      AIOS_SEC_CERT_WORKLOAD_TTL_S: "86400"
      AIOS_SEC_ATTESTATION_REQUIRED: "true"
    secrets: [ ca_intermediate_ref ]        # referência ao HSM, não o material
    networks: [ aios-security-internal ]     # rede isolada
    deploy: { replicas: 2 }
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8080/healthz"]
      interval: 15s

  aios-security-svc:
    image: aios/security-svc:0.1
    environment:
      AIOS_SEC_TOKEN_ACCESS_TTL_S: "900"
      AIOS_SEC_TOKEN_MAX_TTL_S: "3600"
      AIOS_SEC_KEY_SIGNING_ROTATION_DAYS: "90"
      AIOS_SEC_AUTHN_MFA_REQUIRED_FOR_HUMAN: "true"
      AIOS_SEC_REVOCATION_PROPAGATION_TARGET_S: "30"
      AIOS_SEC_FEDERATION_MAX_SCOPE_EXPANSION: "0"
      AIOS_SEC_POLICY_FAIL_MODE: "closed"
      OTEL_EXPORTER_OTLP_ENDPOINT: "http://otel-collector:4317"
    depends_on: [ aios-security-ca, aios-postgres-primary ]
    networks: [ aios-security-internal, aios-mesh ]
    deploy: { replicas: 3 }
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8080/readyz"]
      interval: 15s

networks:
  aios-security-internal:
    internal: true            # sem rota para fora
  aios-mesh: {}
```

---

## 9. Ambientes

| Ambiente | Diferenças |
|----------|-----------|
| `dev` (local) | 1 réplica; CA raiz de desenvolvimento gerada localmente; `hsm_enabled=false`. **`attestation.required` permanece `true`** com atestador simulado — desligá-la mascararia defeitos de identidade que só apareceriam em produção. |
| `staging` | Topologia de produção com HSM simulado; cerimônias de rotação ensaiadas aqui antes de produção. |
| `prod` | Topologia completa (§1); HSM real; CA raiz em custódia física; `sec.key.hsm_enabled = true`. |

As cerimônias (rotação de KEK, recuperação de raiz) **DEVEM** ser ensaiadas em
`staging` antes da primeira execução em produção — o momento de descobrir um passo
faltante no procedimento não é durante um incidente.

---

## 10. Referências

- Topologia global: `../028-Deployment/` · HA/DR: `../027-Cluster/`
- Configuração: `./Configuration.md` · Modelo físico: `./Database.md`
- Monitoramento: `./Monitoring.md` · Falhas e cerimônias: `./FailureRecovery.md`
- Runbooks e cerimônias: `../029-Operations/`
