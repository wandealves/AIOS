---
Documento: Security
Módulo: 004-API
Status: Draft
Versão: 0.1
Última atualização: 2026-07-21
Responsável (RACI-A): Arquiteto do Módulo 004-API
ADRs relacionados: ADR-0008 (global), ADR-0041, ADR-0043, ADR-0048, ADR-0049
RFCs relacionados: RFC-0001 (§6, §7)
Depende de: ../021-Security/README.md, ../022-Policy/README.md, ../025-Audit/README.md
---

# 004-API — Segurança

> Reproduz e detalha `./_DESIGN_BRIEF.md` §12. Este documento **não redefine**
> os contratos de segurança centrais (mTLS, OAuth2/OIDC, envelope de erro) —
> reutiliza `../003-RFC/RFC-0001-Architecture-Baseline.md` §6–§7.

## 1. AuthN / AuthZ

### 1.1 AuthN de borda

O Gateway é o **ponto de validação de AuthN de borda**: valida tokens
OAuth2/OIDC (JWT) contra as chaves publicadas (JWKS) por `021-Security`, que
permanece a **fonte da verdade** de identidade (emissão, rotação de chaves,
revogação). O Gateway **NÃO DEVE** emitir, renovar ou revogar tokens (NR-03
do brief). Comunicação Gateway→upstream **DEVE** usar **mTLS**
(RFC-0001 §6).

### 1.2 AuthZ em duas camadas (defesa em profundidade)

1. O Gateway é **PEP grosso de rota** (`RouteAuthorizer`), consultando o
   **PDP** do `022-Policy` para decidir se tenant/role pode sequer chamar a
   rota — postura **default deny**.
2. O serviço de destino (ex.: `006-Kernel`) permanece o **PEP fino de
   capacidade/recurso**.

O Gateway **NÃO substitui** a autorização fina downstream — é uma camada
adicional, não uma alternativa (ver ADR-0041, a propor).

### 1.3 Isolamento de tenant

O Gateway **NÃO DEVE** aceitar `X-AIOS-Tenant` divergente do `tenant`
contido no token autenticado (RFC-0001 §6) → `AIOS-API-0016`. Upstreams
internos **NÃO DEVEM** ser alcançáveis diretamente da rede pública — apenas
via mTLS a partir do Gateway/malha interna (segmentação de rede,
`../028-Deployment/`), servindo de *backstop* contra *bypass* do PEP de
borda.

## 2. Threat Model STRIDE (detalhado)

| Ameaça (STRIDE) | Vetor no API Gateway | Mitigação |
|-----------------|-----------------------|-----------|
| **S**poofing | Forjar tenant/identidade via token adulterado ou reaproveitado. | Validação de assinatura JWT contra JWKS rotacionado; `X-AIOS-Tenant` casado com claim do token (`AIOS-API-0016`); mTLS interno entre Gateway e upstreams. |
| **T**ampering | Alterar registro de rota/versão para desviar tráfego ou reativar versão retirada. | OCC (`version`), papel `api-registry-admin` exigido, RLS no estado por tenant, auditoria imutável (`025-Audit`) de toda mutação de registro. |
| **R**epudiation | Negar ter feito uma chamada de borda ou uma mutação de registro. | Logs de acesso + `GatewayTelemetry` correlacionados por `trace_id`; eventos `api.route.*`/`api.version.*` auditados via `025-Audit`. |
| **I**nformation disclosure | Vazamento de PII/segredos em `detail` de erro, logs de acesso ou payload agregado. | Envelope RFC-0001 §5.4 sem PII em `detail`; redação de `Authorization`/tokens nos logs; minimização (RFC-0001 §7). |
| **D**enial of service | *Flood* na borda pública (alvo primário de DoS por ser o único ponto de entrada). | Rate limiting por tenant/rota, limite de tamanho de corpo, circuit breaker/bulkhead por upstream, autoscaling de réplicas (`../028-Deployment/`), proteção L7/edge (`../027-Cluster/`). |
| **E**levation of privilege | Contornar o PEP de rota chamando o upstream diretamente, ou escalar *role* via payload manipulado. | Segmentação de rede (upstreams só aceitam mTLS do Gateway/malha interna); *roles* derivadas de claims assinadas, nunca de payload do cliente; PEP fino redundante no upstream. |

## 3. Segredos e chaves

| Segredo/chave | Origem | Rotação | Onde é consumido |
|-----------------|--------|---------|--------------------|
| Chaves JWKS de validação | `021-Security` (publicadas via endpoint JWKS). | Conforme política de `021`; Gateway recarrega via evento `aios._platform.security.jwks.rotated` e/ou TTL de cache. | `AuthNFilter` / `JwksCache`. |
| Certificados mTLS (Gateway↔upstream) | Autoridade certificadora interna (`021-Security`/`../028-Deployment/`). | Rotação automática antes da expiração (padrão AIOS). | `EdgeRouter`, todos os clientes gRPC internos. |
| Credenciais PostgreSQL/Redis | Gerenciador de segredos da plataforma. | Rotação periódica sem *downtime* (conexões renovadas em *hot reload*). | `ContractRegistry`, `RateLimiter`, `IdempotencyRelay`. |

O Gateway **NÃO DEVE** armazenar segredos de longa duração em configuração
estática; toda credencial é injetada em tempo de execução pelo gerenciador
de segredos da plataforma (fora do escopo deste módulo).

## 4. TLS / mTLS

- **Externo (cliente→Gateway)**: TLS 1.2+ obrigatório; TLS 1.3 preferencial.
  Terminação de TLS ocorre no `EdgeRouter`.
- **Interno (Gateway→upstream)**: **mTLS obrigatório** em toda chamada
  (RFC-0001 §6) — o Gateway apresenta certificado de cliente identificando-se
  como `004-API` a cada upstream, e valida o certificado de servidor do
  upstream.
- **Gateway→`021`/`022`**: mTLS, mesmo padrão.

## 5. Sandbox

Não aplicável diretamente — o Gateway não executa código de terceiros nem
carrega *plugins* dinâmicos de módulo; toda extensão de comportamento
(novas rotas, novos códigos de erro) ocorre via **dados** (registros no
`ContractRegistry`), não via código carregado dinamicamente. Isso elimina
uma classe inteira de superfície de ataque de sandbox-escape relevante para
módulos de execução de agente (`007-Agent-Runtime`).

## 6. LGPD / GDPR

- **Minimização**: logs de acesso e eventos de borda **NÃO DEVEM** conter
  corpo de requisição/resposta com PII por padrão; apenas metadados (rota,
  status, latência, `trace_id`, `tenant_id`) são logados estruturadamente
  (RFC-0001 §7). Cabeçalhos sensíveis (`Authorization`, cookies) **DEVEM**
  ser redigidos nos logs (ver `./Logging.md`).
- **Retenção**: registros de idempotência de borda têm TTL configurável
  (`api.idempotency.ttl_hours`, mínimo 24h); logs de acesso seguem política
  de retenção definida em `../024-Observability/`/`../025-Audit/`, não neste
  módulo.
- **Direito ao esquecimento**: o Gateway **NÃO É** o dono de dados
  pessoais — não persiste payloads de negócio; expurgo de dados de
  agente/usuário é coordenado pelos módulos donos (`010-Memory`,
  `025-Audit`) e não requer ação neste módulo além de invalidar caches de
  sessão associados ao tenant/usuário.
- **Segregação**: `tenant_id` + RLS no estado por tenant
  (`RateLimitPolicy`, `IdempotencyRecord` — ver `./Database.md`); CORS e
  política de origem configuráveis por tenant (`api.cors.allowed_origins`)
  para suportar requisitos de residência/soberania de dados.

## 7. Controles de segurança — resumo

| Controle | Componente | NFR relacionado |
|----------|------------|-------------------|
| Validação de JWT contra JWKS | `AuthNFilter` | NFR-008 |
| Consulta ao PDP com *default deny* | `RouteAuthorizer` | NFR-008 |
| Rejeição de `X-AIOS-Tenant` divergente | `AuthNFilter` | NFR-008, FR-014 |
| Rate limiting/quotas de borda | `RateLimiter` | NFR-007 |
| Normalização de erro sem vazamento de PII | `ErrorTranslator` | RFC-0001 §7 |
| Auditoria de rejeições e violações | `GatewayTelemetry` → `025-Audit` | NFR-009 |
| Segmentação de rede (upstream só via mTLS do Gateway) | `../028-Deployment/` | NFR-008 |

## 8. Riscos e alternativas

Ver `./Vision.md` §8 (riscos estratégicos) para os riscos RSK-A01..RSK-A05.
A alternativa de deixar AuthN/AuthZ inteiramente a cargo de cada módulo
upstream foi descartada (ver `./Architecture.md` §11) por fragmentar a
postura de segurança de borda.

*Fim de `Security.md`.*
