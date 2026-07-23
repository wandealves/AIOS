---
Documento: SequenceDiagrams
Módulo: 021-Security
Status: Draft
Versão: 0.1
Última atualização: 2026-07-22
Responsável (RACI-A): Arquiteto do Módulo 021-Security
ADRs relacionados: ADR-0211, ADR-0213, ADR-0214, ADR-0215, ADR-0216, ADR-0217
RFCs relacionados: RFC-0001, RFC-0210, RFC-0211
Depende de: UseCases.md, StateMachine.md, API.md, Events.md
---

# 021-Security — Diagramas de Sequência

Convenção: `──▶` chamada síncrona, `╌╌▶` mensagem assíncrona (NATS), `◀──` resposta,
`[guard]` condição. Toda chamada carrega os cabeçalhos de correlação da RFC-0001 §5.6.

---

## 1. Caminho feliz — Validação de token (o caminho que NÃO passa pelo 021)

```
 Cliente     004-API (AuthNFilter)      JwksCache        021-Security
    │                │                      │                  │
    │                │  … no boot / ao receber jwks.rotated …   │
    │                │◀───── JWKS (chaves públicas) ────────────┤
    │                │                      │                  │
    ├─ requisição + Bearer JWT ─▶           │                  │
    │                ├─ valida assinatura, exp, iss ──▶│        │
    │                │◀──── válido (local, ~µs) ───────┤        │
    │                ├─ consulta CRL em cache (TTL 10s) │        │
    │                │◀──── não revogado ───────────────┤        │
    │◀─ 200 (roteado ao upstream) ─┤        │                  │
```

Este é o diagrama mais importante do módulo — e o mais fácil de desenhar errado. A
validação acontece **inteiramente no `004-API`**, sem rede até o 021 (ADR-0211). Se
cada requisição do AIOS consultasse este módulo, ele seria ao mesmo tempo o gargalo de
latência e o ponto único de falha total do sistema.

---

## 2. Caminho feliz — Emissão de certificado de workload (UC-003)

```
 Serviço(010)  security-svc  WorkloadAttestor  PDP(022)  security-ca  KMS   PG   NATS
      │             │              │              │           │        │     │     │
      ├ POST /certificates (CSR, purpose=mesh:010) ──────────▶│        │     │     │
      │             │ T-01 Requested             │           │        │     │     │
      │             ├─ atesta (nó, serviço, imagem) ─▶       │        │     │     │
      │             │◀─ prova de atestação ───────┤          │        │     │     │
      │             ├─ Decide(sec:cert:issue) ───────────────▶│        │     │     │
      │             │◀──────── allow ─────────────────────────┤        │     │     │
      │             ├─ IssueWorkloadCert(csr, proof) ─────────▶        │     │     │
      │             │                                         ├─ assina ▶     │     │
      │             │                                         │◀── cert ┤     │     │
      │             │◀──────── certificado (TTL 24h) ─────────┤        │     │     │
      │             ├─ BEGIN; credential(Issued→Active); fingerprint; outbox; COMMIT ▶
      │◀─ certificado (entregue UMA vez) ─┤                    │        │     │     │
      │             │                                                         ╌╌╌▶│
      │             │        aios.<tenant>.security.credential.issued              │
```

O material vai para o solicitante e **não fica recuperável**: o banco guarda
`fingerprint` e referência cifrada (invariante I5). Perdeu o certificado? Reemita — é
mais rápido e mais seguro que manter um caminho de recuperação.

---

## 3. Caminho feliz — Rotação sem downtime (UC-005)

```
 CredentialBroker   PG    Consumidor(005)   Recurso(PostgreSQL)   NATS
        │            │          │                   │              │
        │  … renew_before_ratio (50% do TTL) atingido …            │
        ├─ emite sucessora ─────────────────────────▶              │
        │            │          │      CREATE ROLE memory_rw_v2 ──▶│
        ├─ antiga → Rotating (T-05) ─▶                             │
        │            │                                       ╌╌╌╌▶ │
        │            │   aios.<t>.security.credential.rotating     │
        │            │          │◀── evento ────────────────────────┤
        │            │          ├─ adota a nova credencial          │
        │            │          │                   │              │
        │  ═══ janela de sobreposição: AMBAS funcionam (300 s) ═══ │
        │            │          │                   │              │
        ├─ [sem uso recente da antiga] → Superseded (T-06)         │
        │            │          │      DROP ROLE memory_rw_v1 ────▶│
```

Durante a janela existem **exatamente duas** credenciais utilizáveis (invariante I2). A
antiga só é removida depois de comprovadamente ociosa — se ainda houver uso, a janela é
estendida uma vez e um alerta é emitido, em vez de a credencial ser cortada.

---

## 4. Caminho feliz — Revogação e propagação (UC-006)

```
 Operador  security-svc   PG    NATS    004-API   020-Comm   Serviços
     │          │          │      │         │         │          │
     ├ :revoke (motivo=compromise) ▶        │         │          │
     │          ├─ PDP allow ────▶│         │         │          │
     │          ├─ BEGIN; state=Revoked; revocation; outbox; COMMIT ▶ (I4)
     │◀─ 200 ───┤          │      │         │         │          │
     │          │          │ ╌╌╌▶ │         │         │          │
     │          │   aios.<t>.security.token.revoked   │          │
     │          │          │      ├────────▶│ invalida cache      │
     │          │          │      ├──────────────────▶│ encerra sessões/conexões
     │          │          │      ├─────────────────────────────▶│ atualiza CRL local
     │          │          │      │         │         │          │
     │   ◀════ efeito observável em ≤ 30 s (NFR-006) ═══════════▶│
```

A gravação da revogação e a publicação do evento ocorrem na **mesma transação**
(invariante I4). Se o processo morrer entre as duas, o relay do Outbox reenvia — uma
revogação registrada sem propagação seria um dos defeitos mais perigosos possíveis
neste módulo.

---

## 5. Falha — Atestação reprovada (UC-003 E1)

```
 Contêiner suspeito   security-svc   WorkloadAttestor   NATS   025-Audit
        │                  │                │             │        │
        ├ POST /certificates (purpose=mesh:006) ─────────▶│        │
        │                  │ T-01 Requested │             │        │
        │                  ├─ atesta ──────▶│             │        │
        │                  │                ├ identidade do nó: ok │
        │                  │                ├ serviço declarado: 006 │
        │                  │                ├ serviço observado: 099 ✗
        │                  │◀─ FALHA ───────┤             │        │
        │                  │ T-03 Failed    │             │        │
        │◀ 403 AIOS-SEC-0006 ┤              │             │        │
        │                  ├─ audita (tentativa, origem, divergência) ──────▶│
        │                  │            ╌╌╌▶│ aios.<t>.security.attestation.failed
```

Este é o controle central contra um contêiner comprometido assumir a identidade de
outro serviço na rede interna. Sem atestação, bastaria estar na rede para pedir a
credencial de qualquer módulo — inclusive a *role* de banco de outro schema.

---

## 6. Falha — KMS indisponível (UC-003 E4)

```
 Serviço   security-svc   KMS/HSM   004-API   Serviços já autenticados
    │           │            ✗          │              │
    ├ IssueCredential ──────▶(timeout)  │              │
    │           │ T-03 Failed           │              │
    │◀ 503 AIOS-SEC-0010 (retriable) ───┤              │
    │           │                        │              │
    │           │   ⚠ EMISSÃO suspensa   │              │
    │           │                        │              │
    │           │   ✓ VALIDAÇÃO continua ├─ JWKS em cache ─▶ (tokens seguem válidos)
    │           │   ✓ mTLS continua      │              ├─ cadeia da CA em cache ▶
    │           │                        │              │
    │  … KMS restaurado → fila de emissão drena (idempotente por Idempotency-Key) …
```

A degradação é **graciosa por desenho**: perder o KMS impede emitir credenciais
**novas**, mas não invalida as existentes. Um desenho em que a validação dependesse do
KMS transformaria essa falha em queda total do AIOS.

---

## 7. Falha — PDP indisponível (fail-closed seletivo)

```
 Operador   security-svc   PolicyClient   PDP(022)   Cliente com token válido
     │           │              │             ✗                 │
     ├ :rotate ──▶              │                               │
     │           ├─ Decide ────▶│──── timeout ────▶             │
     │           │◀─ CB aberto: Deny (fail-closed) ─┤            │
     │◀ 403 AIOS-SEC-0002 ──────┤                               │
     │           │                                              │
     │           │   ✓ /oauth2/token continua atendendo ────────┤
     │           │   ✓ introspecção continua respondendo         │
```

Operações **novas** são negadas; autenticação e validação de credenciais já emitidas
continuam (princípio P-08). A alternativa — negar tudo — converteria uma falha do
serviço de política em indisponibilidade total da plataforma.

---

## 8. Rotação de chave de assinatura e JWKS (UC-009)

```
 KeyMgmtService   JwksPublisher   NATS    004-API (JwksCache)   Clientes
       │                │           │              │                │
       ├ nova chave (pending) ──────┤              │                │
       ├ publica AMBAS as públicas ─▶              │                │
       │                │      ╌╌╌╌▶│              │                │
       │                │  aios._platform.security.jwks.rotated     │
       │                │           ├─────────────▶│ recarrega      │
       │                │           │              │                │
       ├ nova → active (passa a ASSINAR)           │                │
       │                │           │              ├ valida tokens novos ✓
       │                │           │              ├ valida tokens antigos ✓
       │                │           │              │                │
       │  … 30 dias (retire_grace_days) …          │                │
       ├ antiga → retired (só verifica)            │                │
```

A ordem é essencial: a chave nova é **publicada antes de ser usada para assinar**, e a
antiga permanece publicada. Inverter isso produziria uma janela em que tokens válidos
seriam rejeitados por caches ainda não atualizados.

---

## 9. Publicação e verificação de perfil de sandbox (UC-011)

```
 Eng. Segurança  security-svc   PDP   KMS(signing)   NATS   007-Runtime   Agente
       │              │           │        │           │         │           │
       ├ POST /sandbox-profiles ─▶│        │           │         │           │
       │              ├─ Decide ─▶│        │           │         │           │
       │              │◀─ allow ──┤        │           │         │           │
       │              ├─ assina perfil ───▶│           │         │           │
       │              │◀── assinatura ─────┤           │         │           │
       │              ├─ publica v3 (imutável) ────────┤         │           │
       │              │                          ╌╌╌╌▶│         │           │
       │              │   aios._platform.security.sandboxprofile.published    │
       │              │                                ├────────▶│           │
       │              │                                │         ├ busca v3  │
       │              │◀───────────── GET perfil ──────────────  ┤           │
       │              ├─── perfil + assinatura ──────────────────▶│           │
       │              │                                          ├ VERIFICA  │
       │              │                                          │ [ok] aplica seccomp,
       │              │                                          │ cgroups, netns ──▶│
       │              │                                          │           │ (agente inicia)
       │              │                                          │           │
       │              │                     [assinatura inválida] ├─ RECUSA iniciar
```

O runtime **verifica antes de aplicar** (NFR-015). Perfil não verificado significa
isolamento não garantido — e um agente que executa código de terceiros sem isolamento
garantido não deve executar.

---

## 10. Direito ao esquecimento (UC-015)

```
 DPO   security-svc   PG    NATS    010-Memory   005-Database   025-Audit
  │         │          │      │          │             │            │
  ├ :disable(principal) ▶     │          │             │            │
  │         ├─ cascata: N credenciais → Revoked (T-07) │            │
  │         │          │ ╌╌╌▶ │ token.revoked × N      │            │
  │         │          │ ╌╌╌▶ │ principal.disabled     │            │
  │         ├─ emite rtbf.requested ─────▶             │            │
  │         │          │      ├─────────▶│ expurga itens do titular │
  │         │          │      ├───────────────────────▶│ erasure_request
  │         │          │      │          │             ├─ receipt_hash ──▶│
  │         │          │      │◀── confirmações ───────┤            │
  │         ├─ expurga atributos pessoais do princípio │            │
  │◀ comprovante ──────┤      │          │             │            │
```

O 021 é o **gatilho e o coordenador de identidade**; o expurgo do dado de negócio é
executado pelos módulos donos (`010`, `005`), com comprovante registrado em `025`.

---

## 11. Referências

- Casos de uso: `./UseCases.md` · FSM: `./StateMachine.md`
- API e erros: `./API.md` · Eventos: `./Events.md`
- Falhas e recuperação: `./FailureRecovery.md`
