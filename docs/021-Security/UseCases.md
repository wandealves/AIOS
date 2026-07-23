---
Documento: UseCases
Módulo: 021-Security
Status: Draft
Versão: 0.1
Última atualização: 2026-07-22
Responsável (RACI-A): Arquiteto do Módulo 021-Security
ADRs relacionados: ADR-0211, ADR-0212, ADR-0213, ADR-0214, ADR-0215, ADR-0216, ADR-0217, ADR-0218
RFCs relacionados: RFC-0001, RFC-0210, RFC-0211
Depende de: FunctionalRequirements.md, API.md, StateMachine.md, 022-Policy, 025-Audit, 007-Agent-Runtime
---

# 021-Security — Casos de Uso

Formato: ator / pré-condições / fluxo principal / fluxos alternativos / exceções /
pós-condições. Códigos de erro conforme `./API.md` §5.

---

## UC-001 — Autenticar princípio humano

- **Ator primário:** Operador (via `../030-CLI/` ou `../032-WebConsole/`).
- **Requisitos:** FR-001, FR-013.
- **Pré-condições:** princípio `human` em `status = active`.

**Fluxo principal**
1. Cliente inicia `authorization_code` com **PKCE** em `GET /oauth2/authorize`.
2. O `IdentityProvider` autentica a credencial primária e exige **segundo fator**
   (`sec.authn.mfa_required_for_human = true`).
3. Emite *access token* (TTL 15 min) e *refresh token* (TTL 24 h), assinados com a
   chave corrente.
4. Atualiza `last_authn_at`; registra o evento de autenticação em `../025-Audit/`.

**Fluxos alternativos**
- **A1** — princípio federado: delega ao IdP do tenant (UC-012) e mapeia a identidade
  externa a um princípio interno com escopo reduzido.

**Exceções**
- **E1** — credenciais inválidas → `AIOS-AUTHN-0001`. A resposta **não** distingue
  "princípio inexistente" de "senha incorreta" (RN-06): o motivo real vai apenas para
  a auditoria e para o `ThreatSignalDetector`.
- **E2** — segundo fator ausente → `AIOS-AUTHN-0004`.
- **E3** — excesso de tentativas → `AIOS-AUTHN-0005`; contenção **por princípio**.
- **E4** — princípio suspenso → `AIOS-SEC-0008`.

**Pós-condições:** token de curta duração emitido, validável localmente pelo `../004-API/`.

---

## UC-002 — Autenticar serviço

- **Ator primário:** Serviço do plano de controle.
- **Requisitos:** FR-001, FR-003.

**Fluxo principal**
1. O serviço apresenta seu **certificado de workload** (mTLS) ou executa
   `client_credentials` em `POST /oauth2/token`.
2. O `TokenService` valida a cadeia até a CA intermediária e verifica a lista de
   revogação.
3. Emite token de serviço com escopo derivado do `purpose` do certificado.

**Exceções**
- **E1** — certificado revogado → `AIOS-SEC-0007`.
- **E2** — certificado expirado → `AIOS-AUTHN-0002`; o serviço deveria ter renovado em
  `renew_before_ratio` (0,5 do TTL).
- **E3** — `issuer` desconhecido → `AIOS-AUTHN-0003`.

**Pós-condições:** identidade de serviço estabelecida sem segredo compartilhado.

---

## UC-003 — Emitir certificado de workload com atestação

- **Ator primário:** Serviço em inicialização.
- **Requisitos:** FR-003, FR-007, FR-016.
- **Pré-condições:** workload em execução em nó conhecido pelo `../027-Cluster/`.

**Fluxo principal**
1. `POST /v1/security/certificates` com CSR, `purpose` e `Idempotency-Key` → estado
   `Requested` (T-01).
2. O `WorkloadAttestor` verifica atributos verificáveis do orquestrador (identidade do
   nó, do serviço e da versão da imagem) — **não** aceita autodeclaração.
3. O `PolicyClient` consulta o PDP (`sec:cert:issue`).
4. A `CertificateAuthority` emite com TTL ≤ `sec.cert.workload_ttl_s` (24 h) →
   `Issued` → `Active` (T-02, T-04).
5. Emite `security.credential.issued`; audita.

**Fluxos alternativos**
- **A1** — renovação automática ao atingir `renew_before_ratio`: nova emissão + rotação
  (UC-005), sem interrupção do mTLS.

**Exceções**
- **E1** — atestação falha → `Failed` (T-03) com `AIOS-SEC-0006` e evento
  `attestation.failed`. Este é o principal controle contra um contêiner comprometido
  assumir a identidade de outro serviço.
- **E2** — PDP nega → `Failed` com `AIOS-SEC-0002`.
- **E3** — TTL solicitado acima do máximo → `AIOS-SEC-0004`.
- **E4** — KMS/HSM indisponível → `AIOS-SEC-0010` (retriable); a emissão entra em fila.

**Pós-condições:** identidade de workload verificável, com expiração curta.

---

## UC-004 — Emitir credencial dinâmica de recurso

- **Ator primário:** Módulo consumidor (`005`, `020`, `017`).
- **Requisitos:** FR-004, FR-007, FR-016.

**Fluxo principal**
1. `POST /v1/security/credentials` com `kind` (`db_role`, `nats_account`,
   `object_key`, `provider_key`), `purpose` e `Idempotency-Key`.
2. Atestação + PDP (como em UC-003).
3. O `CredentialBroker` cria a credencial **no recurso alvo** (por exemplo, `CREATE
   ROLE` com validade no PostgreSQL) e registra metadados em `security.credential`.
4. Entrega o material **uma única vez** (invariante I5) e emite `credential.issued`.

**Fluxos alternativos**
- **A1** — repetição com a mesma `Idempotency-Key`: retorna o mesmo registro, **sem**
  reemitir material.

**Exceções**
- **E1** — recurso alvo indisponível: a emissão falha e **nada** é registrado como
  `Active` — não existe credencial registrada que não funcione.
- **E2** — limite de credenciais ativas do princípio → `AIOS-SEC-0013`.
- **E3** — TTL acima do máximo do tipo → `AIOS-SEC-0004`.

**Pós-condições:** consumidor com credencial de vida curta, sem segredo em manifesto.

---

## UC-005 — Rotacionar credencial sem downtime

- **Ator primário:** `CredentialBroker` (job) ou consumidor.
- **Requisitos:** FR-005, FR-019.

**Fluxo principal**
1. Gatilho: `rotation_period` atingido, `renew_before_ratio` do TTL, ou pedido
   explícito (`:rotate`).
2. Emite a sucessora (UC-003/UC-004) e vincula `rotated_to`; estado da antiga →
   `Rotating` (T-05).
3. Emite `credential.rotating` — sinal para o consumidor adotar a nova.
4. Decorrida `sec.credential.rotation_overlap_s` (300 s) e sem uso recente da antiga,
   estado → `Superseded` (T-06).

**Fluxos alternativos**
- **A1** — consumidor ainda usando a antiga ao fim da janela: a janela é estendida uma
  vez e um alerta é emitido; a antiga **não** é revogada silenciosamente.

**Exceções**
- **E1** — sucessora falha ao ser emitida: a antiga permanece `Active`; a rotação é
  reagendada. Nunca se remove a credencial atual antes de a nova estar utilizável.
- **E2** — rollout em andamento (`deployment.rollout.started`): a rotação é suspensa
  até o fim da janela.
- **E3** — três credenciais utilizáveis do mesmo `purpose`: violação da invariante I2 →
  incidente; o *advisory lock* por `purpose`+`principal` existe para impedi-lo.

**Pós-condições:** credencial substituída sem interrupção do consumidor.

---

## UC-006 — Revogar credencial

- **Ator primário:** Operador de segurança, ou sistema (comprometimento detectado).
- **Requisitos:** FR-006, FR-016, FR-017.

**Fluxo principal**
1. `POST /v1/security/credentials/{urn}:revoke` com motivo.
2. PDP autoriza `sec:credential:revoke`.
3. Em **uma transação**: estado → `Revoked` (T-07), entrada em `security.revocation`,
   linha no Outbox (invariante I4).
4. O relay publica `aios.<tenant>.security.token.revoked`; verificadores
   (`004`, `020`, serviços com cache de CRL) invalidam em ≤ 30 s (NFR-006).

**Fluxos alternativos**
- **A1** — revogação retroativa de credencial já `Superseded` (T-09): usada quando o
  comprometimento é descoberto depois da substituição, pois a antiga pode estar em
  cache de terceiros.

**Exceções**
- **E1** — credencial já `Revoked`: operação idempotente, sem novo evento.
- **E2** — tentativa de "desrevogar" → recusada (invariante I3). Recuperar acesso exige
  **nova** emissão, o que garante que o caminho de recuperação passe pelo PDP e pela
  auditoria.

**Pós-condições:** credencial inutilizável, com trilha do motivo e do autor.

---

## UC-007 — Desabilitar princípio com revogação em cascata

- **Ator primário:** Administrador do tenant (desligamento, incidente).
- **Requisitos:** FR-014, FR-006.

**Fluxo principal**
1. `POST /v1/security/principals/{urn}:disable` com motivo.
2. `principal.status = disabled`.
3. Todas as credenciais em estado não-terminal do princípio → `Revoked` (T-07), em lote
   transacional com suas entradas de revogação.
4. Emite `principal.disabled` e um `token.revoked` por credencial.

**Exceções**
- **E1** — princípio com credenciais em uso por serviços críticos: a operação prossegue
  (desabilitar é decisão deliberada), mas o alerta é emitido para operação — não se
  posterga revogação de segurança por conveniência.
- **E2** — falha parcial na cascata: a transação é revertida; o princípio **não** fica
  desabilitado com credenciais vivas.

**Pós-condições:** nenhum acesso remanescente do princípio.

---

## UC-008 — Escrever e ler segredo

- **Ator primário:** Módulo consumidor.
- **Requisitos:** FR-008, FR-015, FR-018.

**Fluxo principal (escrita)**
1. `PUT /v1/security/secrets/{path}` com o valor.
2. O `SecretStore` gera/usa a DEK corrente, cifra por envelope e grava **nova versão**
   (versões são imutáveis).

**Fluxo principal (leitura)**
1. `POST /v1/security/secrets/{path}:read` → PDP autoriza `sec:secret:read`.
2. Retorna o valor decifrado **com um lease** (`sec.secret.default_lease_ttl_s`).
3. Registra a leitura na auditoria: quem, quando, qual caminho, qual lease — **nunca**
   o valor.

**Exceções**
- **E1** — leitura fora do lease válido → `AIOS-SEC-0014`.
- **E2** — suíte criptográfica não permitida na escrita → `AIOS-SEC-0011`.
- **E3** — segredo inexistente → `AIOS-SEC-0001`.

**Pós-condições:** segredo disponível ao consumidor autorizado, com trilha de acesso.

---

## UC-009 — Rotacionar chave de assinatura e publicar JWKS

- **Ator primário:** `KeyManagementService` (job) ou operador.
- **Requisitos:** FR-002, FR-009, FR-017.

**Fluxo principal**
1. Gatilho: `sec.key.signing_rotation_days` (90 dias) atingido.
2. Nova chave criada em `pending` → promovida a `active`; a anterior vai a `rotating`
   (passa a **apenas verificar**).
3. `JwksPublisher` publica **ambas** as chaves públicas e emite
   `security.jwks.rotated`.
4. O `../004-API/` recarrega o `JwksCache` ao receber o evento (ou por TTL).
5. Após `sec.key.retire_grace_days` (30 dias) sem token pendente assinado pela antiga,
   ela vai a `retired`.

**Exceções**
- **E1** — `004` com cache desatualizado: tokens novos falhariam. É exatamente por isso
  que a chave nova é publicada **antes** de ser usada para assinar, e a antiga
  permanece publicada.
- **E2** — destruição prematura da chave antiga: tokens legítimos tornam-se
  inverificáveis. `destroyed` é decisão deliberada, nunca automática.

**Pós-condições:** rotação sem invalidar token algum.

---

## UC-010 — Rotacionar a KEK (cerimônia)

- **Ator primário:** Dois operadores de segurança (aprovação dupla).
- **Requisitos:** FR-009, FR-016.
- **Pré-condições:** janela programada; `../029-Operations/` notificado.

**Fluxo principal**
1. Dois operadores autorizam (capabilities distintas; separação de funções).
2. Nova KEK gerada (em HSM se `sec.key.hsm_enabled`).
3. As **DEK** são recifradas com a nova KEK — os **dados não são tocados** (é essa a
   razão da cifragem envelopada, ADR-0216).
4. KEK anterior → `rotating` → `retired`, e só depois `destroyed`.

**Exceções**
- **E1** — falha no meio da recifragem: DEK ainda envolvidas pela KEK antiga continuam
  legíveis; o processo é retomável e idempotente por DEK.
- **E2** — aprovação única → `AIOS-SEC-0002`.

**Pós-condições:** raiz de cifragem renovada sem recifrar o corpus de dados.

---

## UC-011 — Publicar perfil de sandbox

- **Ator primário:** Engenheiro de segurança.
- **Requisitos:** FR-010, FR-016.

**Fluxo principal**
1. `POST /v1/security/sandbox-profiles` com `seccomp`, `cgroups`, `namespaces` e
   `egress_allowlist`.
2. PDP autoriza `sec:sandbox:publish`.
3. O perfil é **assinado** com a chave `signing` e publicado com `profile_version`
   incrementada; emite `sandboxprofile.published`.
4. O `../007-Agent-Runtime/` busca o perfil, **verifica a assinatura** e o aplica.

**Exceções**
- **E1** — alterar perfil já publicado → `AIOS-SEC-0012`. Perfis são imutáveis por
  versão; mudança gera nova versão, o que preserva a capacidade de auditar sob qual
  isolamento um agente executou em qualquer momento do passado.
- **E2** — assinatura inválida na verificação pelo `007`: o runtime **recusa** iniciar
  o agente — perfil não verificado significa isolamento não garantido (NFR-015).

**Pós-condições:** isolamento distribuído como artefato verificável.

---

## UC-012 — Federar IdP externo do tenant

- **Ator primário:** Administrador do tenant.
- **Requisitos:** FR-011, FR-016.

**Fluxo principal**
1. `PUT /v1/security/federation/{issuer}` com `jwks_uri` e `allowed_scopes`.
2. PDP autoriza `sec:federation:write`.
3. O `FederationConnector` valida a acessibilidade do `jwks_uri` e registra a confiança.
4. Autenticações via aquele `issuer` passam a mapear para princípios internos
   `kind = human` com **escopo limitado** a `allowed_scopes`.

**Exceções**
- **E1** — `issuer` não registrado → `AIOS-AUTHN-0006`.
- **E2** — token externo pedindo escopo não permitido → `AIOS-AUTHN-0007`. Por default,
  `sec.federation.max_scope_expansion = 0`: confiança externa **não** se converte em
  confiança interna adicional (ADR-0218).

**Pós-condições:** SSO corporativo sem ampliar a superfície de privilégio.

---

## UC-013 — Registrar âncora de confiança de par A2A externo

- **Ator primário:** Administrador do tenant, a pedido do `../020-Communication/`.
- **Requisitos:** FR-012.

**Fluxo principal**
1. `PUT /v1/security/federation/{issuer}` com `kind = a2a_peer` e `trust_anchor`.
2. O `020` passa a aceitar sessões A2A com pares cuja identidade encadeia até a âncora.

**Exceções**
- **E1** — suspensão/revogação da confiança: emite `federation.suspended`; o `020`
  encerra as sessões A2A com aquele par (T-10 da FSM do `020`).

**Pós-condições:** federação entre organizações com ponto único de revogação.

---

## UC-014 — Detectar e sinalizar anomalia de autenticação

- **Ator primário:** `ThreatSignalDetector` (ator de sistema).
- **Requisitos:** FR-013.

**Fluxo principal**
1. Detecta padrão anômalo: rajada de falhas, uso de credencial revogada, uso fora da
   origem/escopo de emissão.
2. Aplica **contenção por princípio** (`AIOS-AUTHN-0005`, `lockout_s`).
3. Emite `aios.<tenant>.security.authn.anomaly` para `024`/`025`.

**Exceções**
- **E1** — o módulo **NÃO DEVE** decidir bloqueio permanente nem alterar política: a
  decisão é do `../022-Policy/` ou de `../029-Operations/`. Embutir essa decisão aqui
  criaria um segundo PDP informal, fora da governança.

**Pós-condições:** sinal disponível para quem tem autoridade para decidir.

---

## UC-015 — Atender pedido de direito ao esquecimento

- **Ator primário:** Encarregado de dados (DPO).
- **Requisitos:** FR-020, FR-014.

**Fluxo principal**
1. Pedido registrado; o módulo identifica o princípio do titular.
2. Desabilita o princípio com revogação em cascata (UC-007).
3. Emite `aios.<tenant>.security.rtbf.requested`, consumido por `../010-Memory/`,
   `../005-Database/` e `../025-Audit/`.
4. Após a confirmação dos módulos donos, expurga os atributos pessoais do princípio,
   preservando apenas o metadado exigido por obrigação legal.

**Exceções**
- **E1** — obrigação de retenção conflitante (`legal_hold` no `005`): o expurgo do dado
  de negócio é suspenso e o conflito registrado; o princípio permanece desabilitado.
- **E2** — pedido para princípio de serviço: recusado — RTBF aplica-se a titulares
  pessoais, não a identidades de workload.

**Pós-condições:** acesso encerrado e expurgo coordenado com trilha verificável.

---

## Referências

- Requisitos: `./FunctionalRequirements.md` · `./NonFunctionalRequirements.md`
- Sequências: `./SequenceDiagrams.md` · FSM: `./StateMachine.md` · API: `./API.md`
- Brief: `./_DESIGN_BRIEF.md`
