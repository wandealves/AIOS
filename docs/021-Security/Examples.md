---
Documento: Examples
Módulo: 021-Security
Status: Draft
Versão: 0.1
Última atualização: 2026-07-22
Responsável (RACI-A): Arquiteto do Módulo 021-Security
ADRs relacionados: ADR-0211, ADR-0213, ADR-0215, ADR-0217, ADR-0218
RFCs relacionados: RFC-0001, RFC-0210, RFC-0211
Depende de: API.md, Events.md, Configuration.md, 030-CLI, 031-SDK
---

# 021-Security — Exemplos

Exemplos executáveis, do mais simples ao avançado. Endpoints e códigos de erro conforme
`./API.md`.

---

## 1. Hello World — autenticar e usar um token

```bash
# 1. obter token de serviço (client_credentials)
TOKEN=$(curl -sX POST https://api.aios.local/oauth2/token \
  --cert /etc/aios/certs/svc.pem --key /etc/aios/certs/svc-key.pem \
  -d grant_type=client_credentials \
  -d scope="memory:read memory:write" | jq -r .access_token)

# 2. usar o token — a validação acontece NO 004-API, localmente
curl -s https://api.aios.local/v1/memory/items/urn:aios:acme:memory:01J9Z8Q6 \
  -H "Authorization: Bearer $TOKEN" \
  -H "X-AIOS-Tenant: acme"
```

O passo 2 **não** gera chamada ao 021: o `../004-API/` valida a assinatura com o JWKS em
cache (ADR-0211). É por isso que o módulo não é gargalo do caminho quente.

---

## 2. Emitir certificado de workload (com atestação)

```bash
# O serviço gera o par de chaves localmente e envia apenas o CSR
openssl req -new -newkey ec -pkeyopt ec_paramgen_curve:P-256 \
  -nodes -keyout svc-key.pem -out svc.csr \
  -subj "/CN=010-Memory/O=aios"

curl -sX POST https://api.aios.local/v1/security/certificates \
  -H "X-AIOS-Tenant: acme" \
  -H "Idempotency-Key: 01J9ZE0K3N5Q7R9T1V3X5Z7A9C" \
  -H "Content-Type: application/json" \
  -d "{ \"csr\": \"$(base64 -w0 svc.csr)\",
        \"purpose\": \"mesh:010\",
        \"attestationProof\": \"$(cat /run/aios/attestation-token)\" }"
```

```json
{
  "metadata": {
    "urn": "urn:aios:acme:credential:01J9ZE1M4P6R8T0V2X4Z6B8D0F",
    "kind": "workload_cert",
    "state": "Active",
    "purpose": "mesh:010",
    "fingerprint": "sha256:3f8b…c204",
    "notBefore": "2026-07-22T17:00:00Z",
    "expiresAt": "2026-07-23T17:00:00Z"
  },
  "material": { "certificate": "-----BEGIN CERTIFICATE-----\nMIIB…" }
}
```

A **chave privada nunca sai do serviço** — só o CSR trafega. O `attestationProof` vem do
orquestrador e é o que impede outro contêiner de pedir este certificado.

---

## 3. Credencial dinâmica de banco (padrão para todos os módulos)

```csharp
// No boot do serviço — e novamente a 50% do TTL
var cred = await security.IssueCredentialAsync(new IssueRequest {
    PrincipalUrn = myPrincipalUrn,
    Kind         = "db_role",
    Purpose      = "postgres:memory_rw",
    TtlSeconds   = 3600
}, idempotencyKey: Ulid.NewUlid().ToString());

// Usar imediatamente; NÃO persistir em disco nem em variável de ambiente
var connString = $"Host=aios-pgbouncer;Username={cred.Material.Username};" +
                 $"Password={cred.Material.Password};Database=aios";

// Agendar renovação em 50% do TTL (sec.cert.renew_before_ratio)
scheduler.At(cred.Metadata.ExpiresAt - (cred.Metadata.ExpiresAt - DateTime.UtcNow) / 2,
             () => RenewAsync());
```

```csharp
// ❌ ANTIPADRÃO — senha de banco em variável de ambiente, sem expiração
var connString = Environment.GetEnvironmentVariable("PG_CONNECTION_STRING");
```

O segundo bloco é exatamente o que este módulo existe para eliminar: uma credencial sem
dono identificado, sem expiração e sem caminho de revogação.

---

## 4. O que acontece ao pedir credencial eterna

```bash
curl -sX POST https://api.aios.local/v1/security/credentials \
  -H "X-AIOS-Tenant: acme" -H "Idempotency-Key: 01J9ZE2N5Q7S9U1W3Y5A7C9E1G" \
  -d '{ "principalUrn": "urn:aios:acme:principal:01J9Z8Q7",
        "kind": "access_token", "purpose": "api:read", "ttlSeconds": 31536000 }'
```

```json
{
  "code": "AIOS-SEC-0004",
  "status": 422,
  "title": "Requested TTL Exceeds Maximum For Credential Kind",
  "detail": "ttlSeconds=31536000 excede sec.token.max_ttl_s=3600 para kind=access_token.",
  "retriable": false
}
```

Não existe parâmetro, cabeçalho ou modo de compatibilidade que contorne isso. Elevar o
teto exige alterar configuração global, sob capability própria e com registro auditado —
porque a exceção "só desta vez" é exatamente como credenciais eternas nascem.

---

## 5. Rotação sem downtime (visão do consumidor)

```python
# O consumidor assina o evento de rotação e adota a sucessora ANTES do fim da janela
async for msg in rotation_sub.messages:
    event = json.loads(msg.data)
    d = event["data"]
    if d["purpose"] != "postgres:memory_rw":
        await msg.ack(); continue

    new_cred = await security.get_credential(d["successorUrn"])
    pool.swap_credentials(new_cred)          # troca a quente, sem derrubar conexões
    log.info("credencial rotacionada, janela até %s", d["overlapEndsAt"])
    await msg.ack()
```

Durante a janela de 300 s, **as duas credenciais funcionam** (invariante I2). Se o
consumidor ainda estiver usando a antiga ao fim da janela, ela é estendida uma vez e um
alerta é emitido — a credencial em uso não é cortada silenciosamente.

---

## 6. Revogação de emergência

```bash
# 1. revogar
aios sec credential revoke urn:aios:acme:credential:01J9ZE1M \
  --reason compromise \
  --detail "chave observada em repositório público"

# 2. confirmar a propagação (meta: 30 s)
aios sec revocation status --fingerprint sha256:3f8b…c204
# → revoked=true  propagated_to=[004-API, 020-Comm, 3 serviços]  elapsed=1.8s

# 3. auditar todo uso desde a emissão
aios audit query --fingerprint sha256:3f8b…c204 --since 7d
```

Tentar reverter:

```bash
aios sec credential unrevoke urn:aios:acme:credential:01J9ZE1M
# → Erro: comando inexistente.
#   Revoked é terminal e irreversível (invariante I3). Emita uma nova credencial.
```

A ausência do comando não é omissão da CLI: é o reflexo de não existir tal transição na
FSM. Recuperar acesso passa obrigatoriamente por nova emissão — com atestação, decisão
do PDP e trilha.

---

## 7. Ler segredo com lease

```python
lease = await security.read_secret("llm/anthropic/api-key")
client = AnthropicClient(api_key=lease.value)

# Renovar antes do fim do lease; NÃO persistir o valor
scheduler.at(lease.expires_at - timedelta(minutes=5), renew_secret)
```

```python
# ❌ ANTIPADRÃO
with open("/etc/secrets/anthropic.key", "w") as f:   # persistiu → agora é um segredo
    f.write(lease.value)                              # sem expiração, sem trilha
```

A leitura fica registrada na auditoria (quem, quando, qual caminho, qual lease) — mas
**nunca** o valor. Se o segredo vazar depois, essa trilha é o que permite descobrir por
onde.

---

## 8. Publicar e consumir perfil de sandbox

```bash
aios sec sandbox publish agent-network-restricted \
  --seccomp @seccomp-strict.json \
  --cgroups '{"cpuMs":1000,"memoryMiB":256,"pidsMax":32}' \
  --namespaces pid,net,mnt,ipc,uts,user \
  --egress-allow api.anthropic.com:443
# → agent-network-restricted v1 publicado e assinado
#   evento aios._platform.security.sandboxprofile.published
```

Do lado do `../007-Agent-Runtime/`:

```python
profile = await security.get_sandbox_profile("agent-network-restricted", version=1)

if not crypto.verify(profile.body, profile.signature, key_id=profile.signed_by):
    raise SandboxProfileTampered(profile.name)   # RECUSA iniciar o agente

sandbox.apply(profile)   # só aplica depois de verificar
```

O runtime **verifica antes de aplicar** (NFR-015). Perfil não verificado significa
isolamento não garantido — e um agente que executa código de terceiros sem isolamento
garantido não deve executar.

Tentar alterar um perfil publicado:

```bash
aios sec sandbox update agent-network-restricted --version 1 --cgroups '{"memoryMiB":1024}'
# → 409 AIOS-SEC-0012: perfil publicado é imutável. Publique a versão 2.
```

---

## 9. Federar o IdP corporativo do tenant

```bash
aios sec federation put https://login.acme-corp.com \
  --kind idp \
  --jwks-uri https://login.acme-corp.com/.well-known/jwks.json \
  --allowed-scopes "console:read,console:write"
```

Usuário do IdP externo pedindo escopo além do acordado:

```json
{
  "code": "AIOS-AUTHN-0007",
  "status": 403,
  "title": "Requested Scope Not Allowed For Federated Identity",
  "detail": "Escopo 'admin:all' não consta em allowed_scopes do issuer https://login.acme-corp.com.",
  "retriable": false
}
```

Com `sec.federation.max_scope_expansion = 0` (default), confiança externa **não** se
converte em privilégio interno adicional (ADR-0218). Um IdP comprometido não escala para
administrador do AIOS.

---

## 10. Verificar a postura de segurança

```bash
aios sec posture
```

```
CREDENCIAIS
  ativas.......................  8.412.334
  sem expiração................          0   ✔ (deve ser 0)
  TTL mediano..................       18 h   ✔ (meta ≤ 24 h)
  expirando sem rotação........          3   ⚠

CHAVES
  signing (active).............     41 dias  ✔ (rotação em 90)
  kek (active).................    203 dias  ✔ (rotação em 365)
  chaves em retired............          2

REVOGAÇÃO
  lista ativa..................        847
  não propagadas...............          0   ✔
  p99 de propagação............      2,1 s   ✔ (meta ≤ 30 s)

CONFORMIDADE
  achados de vazamento.........          0   ✔
  falhas de assinatura de perfil          0   ✔
  configurações proibidas......          0   ✔
```

As linhas marcadas com "deve ser 0" são as **cinco métricas de postura**
(`./Metrics.md` §8): qualquer valor diferente de zero é incidente, não tendência.

---

## 11. Armadilhas Comuns

| Armadilha | Sintoma | Correção |
|-----------|---------|----------|
| Persistir o material recebido | Vira credencial eterna sem trilha | Use em memória; renove a 50% do TTL (§3). |
| Guardar o segredo lido em arquivo | Cópia sem expiração nem auditoria de leitura | Renove o lease; nunca persista (§7). |
| Repetir emissão sem `Idempotency-Key` | Segunda credencial válida criada silenciosamente | Sempre envie a chave (RFC-0001 §5.5). |
| Ignorar `credential.rotating` | Interrupção ao fim da janela de sobreposição | Assine o evento e troque a quente (§5). |
| Ampliar `clock_skew_s` para "corrigir" expiração | Amplia a aceitação de token expirado | Corrija o NTP. |
| Tratar `AIOS-AUTHN-0001` como "usuário não existe" | Suposição errada — a resposta é idêntica nos dois casos | O motivo real está só na auditoria (RN-06). |
| Aplicar perfil de sandbox sem verificar assinatura | Isolamento não garantido | Verifique antes de aplicar (§8). |
| Desligar `attestation.required` para "destravar" um deploy | Remove o controle contra falsificação de identidade | Corrija a atestação; o parâmetro é não-recarregável por isso. |
| Esperar revogação instantânea | Janela até o TTL/cache | É trade-off consciente de ADR-0211; para efeito imediato, além de revogar, derrube as sessões no `../020-Communication/`. |

---

## 12. Referências

- API: `./API.md` · Eventos: `./Events.md` · Configuração: `./Configuration.md`
- CLI: `../030-CLI/` · SDKs: `../031-SDK/`
- Perguntas frequentes: `./FAQ.md`
