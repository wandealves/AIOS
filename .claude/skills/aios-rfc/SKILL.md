---
name: aios-rfc
description: Procedimento canônico para autorar uma RFC técnica do AIOS no estilo IETF, em pt-BR — convenção de nome/numeração, estrutura de seções numeradas (Abstract, Terminologia/RFC2119, Motivação, Especificação normativa, Schemas/Contratos, Segurança, Privacidade, Compatibilidade/Versionamento, Registros, Referências), regra de não redefinir os contratos da RFC-0001, checklist de qualidade e esqueleto preenchível. Use ao criar ou revisar uma RFC.
---

# Skill: Autorar uma RFC do AIOS (estilo IETF)

Guia operacional para produzir uma RFC normativa em conformidade com o repositório
AIOS. Tudo em **português (pt-BR)**; contratos verificáveis, não código de produção.

## 1. Passos do procedimento

1. **Ler o template**: `docs/003-RFC/TEMPLATE.md` (fonte da verdade do front-matter
   e da numeração de seções).
2. **Ler a baseline** `docs/003-RFC/RFC-0001-Architecture-Baseline.md` por completo —
   calibrar profundidade e formatação; identificar os contratos que **não** se redefine.
3. **Listar RFCs existentes** (`docs/003-RFC/RFC-*.md`) — achar o próximo número livre.
4. **Consultar o glossário** `docs/040-Glossary/Glossary.md` — reutilizar termos.
5. **Escrever o arquivo** com o esqueleto (§4), preenchendo todas as seções.
6. **Verificar** o checklist de qualidade (§5).

## 2. Numeração e nome de arquivo

- Próxima RFC livre, **4 dígitos** zero-padded (`RFC-0002`, `RFC-0003`, …).
- Nunca reutilizar número (mesmo `Obsoleted`).
- Arquivo: `docs/003-RFC/RFC-NNNN-Titulo.md` — título em pt-BR, palavras separadas
  por hífen, sem acentos no nome do arquivo.
- Exemplos: `RFC-0002-Kernel-Syscall-ABI.md`,
  `RFC-0003-Scheduler-Cognitivo-Protocolo.md`.

## 3. Regra de não redefinição da RFC-0001

A RFC-0001 já fixa contratos globais. Sua RFC **os referencia** (ex.: "conforme
RFC-0001 §5.2"), nunca os redefine:

| Contrato | Onde | Referência |
|----------|------|-----------|
| Identidade URN `urn:aios:<tenant>:<tipo>:<id>` | RFC-0001 §5.1 | referenciar |
| Envelope de evento CloudEvents (NATS) | RFC-0001 §5.2 | referenciar |
| Subjects `aios.<tenant>.<dominio>.<entidade>.<acao>` | RFC-0001 §5.3 | referenciar |
| Envelope de erro `AIOS-<DOMINIO>-<NNNN>` (RFC 7807) | RFC-0001 §5.4 | referenciar |
| Idempotência (`Idempotency-Key`, dedup por `event.id`) | RFC-0001 §5.5 | referenciar |
| Cabeçalhos de correlação (W3C Trace Context) | RFC-0001 §5.6 | referenciar |
| Versionamento (path/header/gRPC/`dataschema`) | RFC-0001 §5.7 | referenciar |
| PEP/PDP, auditoria imutável, OTel | RFC-0001 §5.8 | referenciar |

Especifique somente o que é **novo** para o domínio da RFC. Alterar a RFC-0001
exige uma RFC que a **obsolete** (`Obsoleted by RFC-XXXX`) com coexistência — nunca
uma redefinição silenciosa dentro de outra RFC.

## 4. Estrutura de seções (estilo IETF, do TEMPLATE.md)

Ordem e numeração obrigatórias:

1. **Abstract** — 1 parágrafo do que a RFC especifica.
2. **Status desta RFC e Terminologia** — cláusula RFC 2119/8174.
3. **Motivação** — por que a especificação é necessária; problema a resolver.
4. **Terminologia** — termos específicos; reutilizar `../040-Glossary/Glossary.md`.
5. **Especificação (normativa)** — o coração da RFC: formatos, mensagens, máquinas
   de estado, algoritmos, **schemas**. Use subseções numeradas (5.1, 5.2, …).
   Empregue palavras normativas RFC 2119 em cada obrigação verificável.
   Inclua os contratos formais adequados ao domínio:
   - **Schema JSON** de payloads (com exemplo válido).
   - **Protocol Buffers/gRPC**: `service`/`message`, pacote versionado `aios.<mod>.v1`.
   - **OpenAPI/REST** quando houver superfície HTTP.
   - **Diagramas ASCII** de sequência/estado/componente.
6. **Considerações de Segurança** — ameaças, authN/Z, mTLS, cripto.
7. **Considerações de Privacidade (LGPD/GDPR)** — minimização, base legal, retenção,
   direito ao esquecimento.
8. **Registros do AIOS (IANA-like)** — enumerações/registros que a RFC cria ou estende.
9. **Compatibilidade e Versionamento** — impacto em consumidores; evolução aditiva.
10. **Referências** — **Normativas** / **Informativas** (RFCs IETF externas + caminhos
    relativos a RFC-0001, ADRs, módulos).
11. **Apêndices** — exemplos completos.

## 5. Checklist de qualidade

- [ ] Front-matter completo (Documento, Módulo `003-RFC`, Status, Versão, Última
      atualização, Autores, Módulos afetados).
- [ ] Palavras normativas RFC 2119/8174 usadas com rigor, só onde há obrigação.
- [ ] Todas as seções numeradas presentes, na ordem do template.
- [ ] Contratos formais concretos (JSON Schema / proto / OpenAPI) com exemplos válidos.
- [ ] **Não redefine** contratos da RFC-0001; apenas os referencia (§ correta).
- [ ] Segurança e Privacidade preenchidas com substância.
- [ ] Compatibilidade/versionamento tratados (evolução aditiva; obsolescência se preciso).
- [ ] Termos do glossário reutilizados; nada redefinido.
- [ ] Referências normativas e informativas completas; caminhos relativos resolvem.
- [ ] Próximo número de RFC livre; nome de arquivo na convenção.

## 6. Esqueleto preenchível

```markdown
---
Documento: RFC-NNNN — <Título>
Módulo: 003-RFC
Status: Draft
Versão: 1.0
Última atualização: AAAA-MM-DD
Autores: <nomes/papéis>
Módulos afetados: <NNN, ...>
---

# RFC-NNNN: <Título>

## 1. Abstract
<Resumo de 1 parágrafo do que a RFC especifica.>

## 2. Status desta RFC e Terminologia
As palavras-chave "DEVE", "NÃO DEVE", "OBRIGATÓRIO", "DEVERIA", "NÃO DEVERIA",
"PODE" e "OPCIONAL" devem ser interpretadas conforme RFC 2119 e RFC 8174.

## 3. Motivação
<Por que esta especificação é necessária. Problema a resolver.>

## 4. Terminologia
<Termos específicos; reusar `../040-Glossary/Glossary.md`.>

## 5. Especificação (normativa)

### 5.1 <Sub-contrato>
<Formato/mensagem/estado/algoritmo. Palavras normativas RFC 2119.>

```json
{ "exemplo": "schema JSON válido do payload" }
```

### 5.2 <Contrato gRPC> (se aplicável)
```proto
syntax = "proto3";
package aios.<modulo>.v1;
// service / message versionados
```

<Referenciar contratos globais em vez de redefini-los, ex.: identidade conforme
RFC-0001 §5.1; envelope de erro conforme RFC-0001 §5.4.>

## 6. Considerações de Segurança
<Ameaças, mitigação, authN/Z, mTLS, cripto.>

## 7. Considerações de Privacidade (LGPD/GDPR)
<Dados pessoais, minimização, base legal, retenção, expurgo.>

## 8. Registros do AIOS (IANA-like)
<Registros/enumerações que esta RFC cria ou estende.>

## 9. Compatibilidade e Versionamento
<Impacto em consumidores; estratégia de evolução aditiva.>

## 10. Referências
**Normativas:** RFC 2119, RFC 8174, ...; `../003-RFC/RFC-0001-Architecture-Baseline.md`.
**Informativas:** ...; `../040-Glossary/Glossary.md`.

## 11. Apêndices
<Exemplos completos.>
```
