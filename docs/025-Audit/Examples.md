---
Documento: Examples
Módulo: 025-Audit
Status: Draft
Versão: 0.1
Última atualização: 2026-07-23
Responsável (RACI-A): Arquiteto do Módulo 025-Audit
ADRs relacionados: ADR-0010, ADR-0251, ADR-0253, ADR-0254, ADR-0255, ADR-0257, ADR-0259
RFCs relacionados: RFC-0001, RFC-0250, RFC-0251
Depende de: API.md, UseCases.md, Configuration.md
---

# 025-Audit — Exemplos

> Exemplos usam o tenant `acme` e os endpoints reais de `./API.md`. O CLI é o do
> `../030-CLI/`; o SDK, o do `../031-SDK/`.

## 1. Hello world — registrar um fato

```bash
curl -sS -X POST https://api.aios.local/v1/audit/records \
  -H "X-AIOS-Tenant: acme" \
  -H "Idempotency-Key: 01J9ZM00000000000000000000" \
  -H "traceparent: 00-4bf92f3577b34da6a3ce929d0e0e4736-00f067aa0ba902b7-01" \
  -H "Content-Type: application/json" \
  -d '{
        "recordClass": "policy.decision",
        "occurredAt": "2026-07-23T10:15:00.123Z",
        "actor": { "urn": "urn:aios:acme:agent:01J9Z8Q6H7K2M4N6P8R0S2T4V6", "kind": "agent" },
        "action": "memory:item:read",
        "resourceUrn": "urn:aios:acme:memory:01J9ZA00000000000000000000",
        "outcome": "denied",
        "decisionRef": "01J9ZB00000000000000000000",
        "payload": { "reasonCode": "explicit_deny", "matchedRules": ["mem.read.cross-agent"] }
      }'
```

```json
HTTP/1.1 201 Created
{ "id": "01J9ZN00000000000000000000", "seq": 418291, "partitionKey": "acme:07",
  "recordHash": "sha256:7a1c9f…", "prevHash": "sha256:c04b1a…",
  "recordedAt": "2026-07-23T10:15:00.184Z", "duplicate": false }
```

O `201` só chega **após o commit durável com replicação confirmada**. Se o produtor
recebeu essa resposta, o fato está na trilha — essa é a promessa do RPO 0.

## 2. Quando o banco não confirma: a resposta correta é recusar

```json
HTTP/1.1 503 Service Unavailable
{ "code": "AIOS-AUD-0005",
  "detail": "Escrita não pôde ser confirmada como durável; reenvie.",
  "retriable": true, "retryAfterMs": 2000 }
```

```csharp
// Lado do produtor: o fato fica no outbox até haver confirmação
var result = await _audit.AppendAsync(record, idempotencyKey: key);
if (!result.Confirmed)
{
    // NÃO descartar. NÃO seguir em frente como se tivesse registrado.
    _outbox.Retain(record, key);       // reenvio com a MESMA chave
    return;
}
_outbox.Remove(key);
```

Compare com o `../024-Observability/`, que responde `200` e descarta sob saturação: os
dois módulos fazem escolhas opostas porque telemetria não pode atrasar o emissor, e
auditoria não pode perder o fato.

## 3. Idempotência: reenviar é seguro

```bash
# Mesma requisição de §1, mesma Idempotency-Key
curl -sS -X POST … -H "Idempotency-Key: 01J9ZM00000000000000000000" …
```

```json
HTTP/1.1 200 OK
{ "id": "01J9ZN00000000000000000000", "seq": 418291,
  "recordHash": "sha256:7a1c9f…", "duplicate": true }
```

Mesmo `seq`, mesmo `recordHash`, `duplicate: true`. Um produtor que reenvia após
timeout não cria um segundo registro do mesmo fato — que é o que produziria uma trilha
com eventos fantasmas.

## 4. Registrar uma classe auditável

Um fato só entra na trilha se sua classe estiver registrada (ADR-0256):

```bash
aios audit class put \
  --key policy.decision \
  --producer 022-Policy \
  --subjects 'aios.*.policy.decision.denied' \
  --retention-days 1825 \
  --legal-basis "Obrigação legal — trilha de autorização (LGPD art. 7º, II)" \
  --contains-pii \
  --expected-rate-per-day 4200000
```

`--expected-rate-per-day` não é estatística decorativa: é o que torna a **ausência**
detectável. Sem ela, um produtor que para de emitir é indistinguível de um sistema sem
atividade (`./Monitoring.md` RB-A-07).

## 5. Provar que um registro está na trilha

```bash
aios audit proof 01J9ZN00000000000000000000
```

```json
{ "recordId": "01J9ZN00000000000000000000",
  "recordHash": "sha256:7a1c9f…",
  "merklePath": [
    { "position": "right", "hash": "sha256:1d0a…" },
    { "position": "left",  "hash": "sha256:9be3…" },
    { "position": "right", "hash": "sha256:44f7…" } ],
  "seal": { "urn": "urn:aios:acme:event:01J9ZQ00000000000000000000",
            "merkleRoot": "sha256:e51b…", "sealHash": "sha256:88ca…",
            "signature": "MEUCIQ…",
            "signedBy": "urn:aios:_platform:key:01J9ZR00000000000000000000",
            "sealedAt": "2026-07-23T11:03:00Z" },
  "publicKeyRef": "https://api.aios.local/.well-known/jwks.json#01J9ZR…" }
```

Com esses três elementos — `recordHash`, `merklePath` e o selo assinado — qualquer
pessoa recomputa a raiz e verifica a assinatura **sem consultar o AIOS de novo**.

## 6. Verificar a integridade da cadeia

```bash
# Verificação completa de uma partição, comparando com a cópia WORM
aios audit verify --partition acme:07 --from 2026-07-01 --to 2026-07-23 --full
```

```
Partição acme:07 · 418.291 registros · 4.392 selos
  [1/6] recomputando record_hash ............... OK  (418.291/418.291)
  [2/6] conferindo prev_hash encadeado ......... OK
  [3/6] recomputando raízes de Merkle .......... OK  (4.392 selos)
  [4/6] verificando assinaturas ................ OK  (2 chaves: 01J9ZR…, 01J9ZS…)
  [5/6] conferindo cadeia de selos ............. OK
  [6/6] comparando com cópia WORM .............. OK  (amostra de 1%)

  ✔ CADEIA ÍNTEGRA   duração: 3 min 12 s
```

O passo 4 mostrando **duas chaves** é o comportamento esperado após uma rotação: cada
selo registra `signed_by`, e as chaves públicas históricas continuam publicadas pelo
`../021-Security/`. O selo de 2026 permanece verificável em 2032.

## 7. Aplicar um *legal hold*

```bash
aios audit hold apply \
  --case PROC-4821 \
  --subject tok:9f2c4e… \
  --class policy.decision,agent.lifecycle \
  --from 2026-01-01 --to 2026-07-23 \
  --legal-basis "Requisição judicial — obrigação legal de preservação" \
  --approved-by urn:aios:acme:principal:01J9ZG00000000000000000000
```

```json
{ "urn": "urn:aios:acme:event:01J9ZV00000000000000000000",
  "recordsHeld": 18422, "appliedAt": "2026-07-23T11:10:00Z", "expiresAt": null }
```

`expiresAt: null` é **válido aqui** e só aqui (`./StateMachine.md` §6.1): um *hold*
expira quando o caso encerra, não quando um temporizador vence. Em troca, ele entra na
revisão obrigatória a cada 90 dias.

O evento `audit.legalhold.applied` faz o `../005-Database/` **suspender o expurgo** das
tabelas no escopo — uma decisão jurídica se propagando ao plano de dados sem
acoplamento direto.

## 8. Direito ao esquecimento — e quando ele é recusado

```bash
# Disparado por security.rtbf.requested do 021; ou manualmente:
aios audit erasure execute --subject tok:9f2c4e…
```

**Caso 1 — sem *hold*:**

```json
{ "receiptUrn": "urn:aios:acme:event:01J9ZZ00000000000000000000",
  "kind": "crypto_shred", "recordsAffected": 3184,
  "receiptHash": "sha256:5d7a…",
  "chainRecordId": "01JA0000000000000000000000",
  "executedAt": "2026-07-23T11:30:00Z" }
```

**Caso 2 — com *hold* vigente:**

```json
HTTP/1.1 423 Locked
{ "code": "AIOS-AUD-0008",
  "detail": "Apagamento bloqueado: legal hold PROC-4821 vigente sobre o escopo.",
  "retriable": false }
```

A recusa **também vira registro na trilha**. O titular é informado da obrigação legal
de preservação — e existe prova de que o pedido foi feito, quando e por quê foi
recusado.

## 9. O que sobra depois do apagamento

```bash
aios audit query --subject tok:9f2c4e… --limit 1
```

```json
{ "records": [ {
    "id": "01J9ZN00000000000000000000", "seq": 418291,
    "occurredAt": "2026-07-23T10:15:00.123Z",
    "recordClass": "policy.decision", "producerModule": "022-Policy",
    "actor": { "urn": "urn:aios:acme:agent:01J9Z8…", "kind": "agent" },
    "action": "memory:item:read",
    "resourceUrn": "urn:aios:acme:memory:01J9ZA…",
    "outcome": "denied",
    "payload": null,
    "payloadDigest": "sha256:b21f…",
    "state": "Shredded",
    "note": "AIOS-AUD-0012: payload apagado por direito ao esquecimento" } ] }
```

Prova-se **que** a ação ocorreu, **quem** a fez, **sobre o quê** e **com que
resultado**. O `payloadDigest` prova que existiu um conteúdo com aquele hash — sem o
qual o apagamento seria indistinguível de um registro que sempre foi vazio.

E a cadeia continua verificável:

```bash
aios audit verify --partition acme:07 --full
#   ✔ CADEIA ÍNTEGRA   (com 3.184 registros Shredded)
```

## 10. Investigar um incidente

```bash
# 1. Do decision_id de um 403 reportado pelo usuário
aios audit query --decision-ref 01J9ZB00000000000000000000
```

```json
{ "records": [ { "id": "01J9ZN…", "seq": 418291, "outcome": "denied",
                 "action": "memory:item:read",
                 "actor": { "urn": "urn:aios:acme:agent:01J9Z8…" },
                 "traceId": "4bf92f3577b34da6a3ce929d0e0e4736",
                 "payload": { "reasonCode": "explicit_deny" } } ] }
```

```bash
# 2. Toda a atividade daquele agente na janela
aios audit query --actor urn:aios:acme:agent:01J9Z8… --from -2h

# 3. Pivotar para o trace completo (correlação RFC-0001 §5.6)
aios obs query trace 4bf92f3577b34da6a3ce929d0e0e4736
```

Cada uma dessas três consultas gerou um registro `audit.access` na trilha, com o
solicitante e o escopo. Quem investigou também está registrado (ADR-0257).

## 11. Exportar para um auditor externo

```bash
aios audit export request \
  --class policy.decision,security.credential \
  --from 2026-01-01 --to 2026-06-30 \
  --justification "Auditoria SOC 2 Tipo II — período H1/2026" \
  --approved-by urn:aios:acme:principal:01J9ZG00000000000000000000
```

```json
{ "urn": "urn:aios:acme:event:01JA0100000000000000000000",
  "recordCount": 842391, "sealCount": 4392,
  "packageHash": "sha256:9c1e…",
  "status": "ready", "expiresAt": "2026-07-26T11:00:00Z" }
```

O auditor baixa e verifica **em ambiente isolado, sem rede para o AIOS**:

```bash
unzip package-01JA01….zip && cd package-01JA01…
cat VERIFY.md

aios-audit-verify --package .          # validador de referência (RFC-0250)
```

```
  [1/5] recomputando record_hash ............... OK  (842.391)
  [2/5] conferindo prev_hash encadeado ......... OK
  [3/5] recomputando raízes de Merkle .......... OK  (4.392)
  [4/5] verificando assinaturas ................ OK  (3 chaves públicas do pacote)
  [5/5] conferindo cadeia de selos ............. OK

  ✔ PACOTE ÍNTEGRO — verificado sem qualquer acesso ao AIOS
    3.184 registros marcados como Shredded (apagamento comprovado; ver manifest.json)
```

A última linha é importante: registros apagados **não** aparecem como dados faltantes,
e sim como apagamentos comprovados, com o comprovante correspondente no manifesto.

## 12. Simular uma adulteração (teste adversarial)

```bash
# Ambiente de homologação. Remove o trigger e altera um registro.
psql -c "DROP TRIGGER tr_record_immutable ON audit.record;"
psql -c "UPDATE audit.record SET outcome='allowed' WHERE id='01J9ZN00000000000000000000';"

# Aguardar o próximo ciclo de verificação (900 s) ou forçar:
aios audit verify --partition acme:07 --full
```

```
  [1/6] recomputando record_hash ............... FALHA
        seq 418291: esperado sha256:7a1c9f…  observado sha256:e4b201…
  [6/6] comparando com cópia WORM .............. DIVERGENTE

  ✘ QUEBRA DE CADEIA DETECTADA
    A cópia WORM e o selo assinado CONCORDAM entre si e DIVERGEM do banco.
    Conclusão: adulteração no banco. Evidência íntegra: WORM + selo.
```

Alterar o registro exigiria também recomputar toda a cadeia posterior, todos os selos
seguintes, forjar assinaturas cuja chave está no `../021-Security/` **e** sobrescrever
objetos com *object lock* — três sistemas com credenciais e operadores distintos
(`./Security.md` §3).

## 13. Referências

- Contratos: `./API.md` · Eventos: `./Events.md`
- Casos de uso: `./UseCases.md` · Configuração: `./Configuration.md`
- CLI e SDK: `../030-CLI/`, `../031-SDK/`
- FAQ: `./FAQ.md`

*Fim de `Examples.md`.*
