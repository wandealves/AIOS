---
Documento: FAQ
Módulo: 021-Security
Status: Draft
Versão: 0.1
Última atualização: 2026-07-22
Responsável (RACI-A): Arquiteto do Módulo 021-Security
ADRs relacionados: ADR-0210, ADR-0211, ADR-0212, ADR-0213, ADR-0214, ADR-0215, ADR-0216, ADR-0217, ADR-0218, ADR-0219
RFCs relacionados: RFC-0001, RFC-0210, RFC-0211
Depende de: Vision.md, API.md, Security.md, Examples.md
---

# 021-Security — Perguntas Frequentes

---

## Escopo e fronteiras

**Qual a diferença entre o 021 e o 022-Policy?**
São três perguntas distintas: o **021** responde *quem é você e com que credencial se
prova*; o **022** responde *você pode fazer isso*; o **025** responde *o que aconteceu*.
Separá-las permite evoluir regras de negócio sem tocar na raiz de confiança — e é a
fronteira formalizada em ADR-0210.

**O 021 define quais papéis existem e o que cada um concede?**
Não. Ele **asserta atributos** do princípio (`attributes` em `security.principal`); o
`../022-Policy/` os interpreta. A distinção fica visível no schema: o 021 grava um JSON
de atributos e não sabe o que significam.

**O 021 aplica o sandbox do agente?**
Não. Ele **publica o perfil** assinado; quem cria namespaces e aplica cgroups é o
`../007-Agent-Runtime/`. O 021 fornece o material verificável; o runtime aplica e
verifica.

**Toda requisição do AIOS passa pelo 021?**
Não — e essa é a decisão arquitetural mais importante do módulo (ADR-0211). Tokens são
JWT assinados, validados **localmente** pelo `../004-API/` com o JWKS em cache. O 021
participa da **emissão** e da **revogação**, não da validação.

**Se o 021 cair, o AIOS para?**
Não. Credenciais já emitidas continuam válidas e validáveis. Para de emitir novas e de
autenticar — degradação, não paralisia (`./FailureRecovery.md` §2).

---

## Credenciais

**Por que não posso ter uma credencial sem expiração?**
Porque credencial de longa vida é dívida de segurança: sem expiração, um vazamento é
permanente até alguém notar. `expires_at` é `NOT NULL` no schema — a proibição está no
banco, não numa convenção que alguém possa contornar sob pressão.

**Preciso de um token que dure um ano para uma integração. Como faço?**
Não faz — e provavelmente não precisa. O padrão correto é um princípio `service` com
`client_credentials`, renovando token de 15 min automaticamente. Se a integração não
consegue renovar, o problema está nela, e a solução é um *sidecar* que faça a renovação,
não uma credencial eterna.

**Perdi a credencial que recebi. Como recupero?**
Não recupera: **reemite**. O material é entregue uma única vez (invariante I5) e o banco
guarda apenas `fingerprint` e referência cifrada. Isso é uma propriedade, não uma
limitação — não existe caminho de recuperação para um atacante explorar.

**Revoguei por engano. Como desfaço?**
Não desfaz. `Revoked` é terminal e irreversível (invariante I3). Emita uma nova
credencial — leva segundos e garante que a restauração de acesso passe por atestação,
PDP e auditoria, em vez de por um botão de "desfazer".

**Por que a rotação tem janela de sobreposição?**
Para trocar sem downtime: durante 300 s, a antiga e a nova funcionam, e o consumidor
adota a nova ao receber `credential.rotating`. Sem a janela, toda rotação exigiria
coordenação perfeita entre emissor e consumidor.

**A revogação é instantânea?**
Não, e isso é deliberado. O preço de a validação ser local é a revogação levar até
30 s (NFR-006): evento pelo NATS (< 1 s no caminho feliz) e cache de CRL de 10 s.
Se precisar de efeito imediato, além de revogar, derrube as sessões no
`../020-Communication/`.

---

## Identidade de serviço e atestação

**O que é atestação e por que não posso desligá-la?**
É a verificação de que o solicitante **é** o serviço que diz ser, usando atributos
verificáveis do orquestrador (nó, serviço, imagem). Sem ela, bastaria estar na rede
interna para pedir a credencial de qualquer módulo — inclusive a *role* de banco de
outro schema. Por isso `sec.attestation.required` é não-recarregável.

**Recebi `AIOS-SEC-0006` (atestação falhou). O que verifico?**
Compare serviço **declarado** e **observado** no evento `attestation.failed`. Na maioria
das vezes é defeito de deploy (rótulo errado, imagem divergente). Se não for, é
tentativa de assumir identidade alheia — incidente P1.

**Por que certificados de workload duram só 24 h?**
Porque comprometer um certificado de 24 h dá ao atacante, no pior caso, 24 h — e a
renovação automática a 50% do TTL torna o custo operacional próximo de zero. Reduzir
para 1 h multiplicaria a carga no HSM sem ganho proporcional (`./Benchmark.md` §4.4).

---

## Segredos e chaves

**Onde ficam meus segredos?**
Cifrados por envelope (DEK do KMS) em `security.secret`. Um dump do banco **não**
concede acesso a nada: as DEK estão envolvidas por uma KEK que não está no backup
(teste T-LEAK-02).

**Por que a leitura de segredo tem lease?**
Para que a posse seja temporária e rastreável. Cada leitura fica registrada (quem,
quando, qual caminho) — **nunca** o valor. Se o segredo vazar depois, essa trilha diz
por onde investigar.

**Por que rotacionar a KEK não exige recifrar todos os dados?**
Porque a cifragem é **envelopada** (ADR-0216): a KEK envolve as DEK, e as DEK cifram os
dados. Rotacionar a KEK recifra ~10⁴ DEK em ~80 s, sem tocar em um byte de dado
(`./Benchmark.md` §8). Sem isso, a rotação levaria horas e, na prática, nunca seria
feita no prazo.

**Por que uma chave "aposentada" não é destruída logo?**
Porque tokens, perfis e certificados assinados por ela ainda precisam ser
**verificáveis**. Destruir cedo transforma artefatos legítimos em inverificáveis — uma
negação de serviço autoinfligida. `retired` dura 30 dias, e `destroyed` é decisão
deliberada.

---

## Autenticação

**Por que a mensagem de erro não diz se o usuário existe?**
Porque isso seria um **oráculo de enumeração de contas**. `AIOS-AUTHN-0001` é idêntico
para "não existe" e "senha errada", com tempo de resposta constante. O motivo real vai
só para a auditoria e para o `ThreatSignalDetector` (RN-06).

**Fui contido por excesso de tentativas. Isso afeta outros usuários?**
Não. A contenção é **por princípio**, nunca global — do contrário, atacar uma conta
negaria serviço a todas, transformando o controle de segurança em vetor de DoS.

**MFA é obrigatório mesmo para administradores?**
Especialmente para eles. `sec.authn.mfa_required_for_human = true` é o default, e
desligá-lo consta da lista de **configurações proibidas em produção**
(`./Configuration.md` §5).

**Posso usar o IdP da minha empresa?**
Sim (UC-012). A identidade externa mapeia para um princípio interno com **escopo
reduzido**: `sec.federation.max_scope_expansion = 0` por default significa que a
confiança externa não vira privilégio interno adicional. Um IdP comprometido não escala
para administrador do AIOS.

---

## Operação

**O KMS caiu. O AIOS parou?**
Não. Emissão suspensa (`AIOS-SEC-0010`); **validação, mTLS e autenticação com
credenciais existentes continuam**. A fila de emissão drena quando o KMS voltar
(`./FailureRecovery.md` §5.1).

**O PDP caiu. O 021 para de autenticar?**
Não. `fail_mode=closed` nega **operações administrativas**; `/oauth2/token` e a
introspecção continuam. Negar tudo converteria uma falha de política em queda total do
AIOS (princípio P-08).

**Por que 3 réplicas e não 2?**
Porque o SLO é 99,99% — 4,4 min/mês, cinco vezes mais apertado que o dos demais módulos.
Com duas réplicas, uma atualização contínua deixa o sistema em capacidade única.

**Como o 021 obtém sua própria credencial de banco?**
Pela **credencial de bootstrap**: única, provisionada na instalação, de rotação manual e
monitorada como risco permanente (`./Deployment.md` §3). É a resposta explícita à
circularidade inevitável — o módulo que emite credenciais não pode emitir a primeira
para si mesmo.

**Onde fica a CA raiz?**
*Offline*, em HSM ou custódia física, sem interface de rede. Ela assina apenas as
intermediárias, em cerimônia com aprovação dupla. Comprometer uma intermediária é
recuperável; a raiz não deveria ser alcançável remotamente para ser comprometida.

**Com que frequência ensaiamos as cerimônias?**
Rotação de KEK: semestral em staging. Recuperação de CA raiz: anual em staging. O
momento de descobrir um passo faltante no procedimento não é durante um comprometimento.

---

## Conformidade

**Como funciona o direito ao esquecimento?**
O 021 é o **gatilho**: desabilita o princípio (revogação em cascata) e emite
`security.rtbf.requested`, consumido por `../010-Memory/`, `../005-Database/` e
`../025-Audit/`. O expurgo do dado de negócio é dos módulos donos (UC-015).

**Posso apagar tudo sobre um usuário?**
Não tudo: registros exigidos por obrigação legal (quem acessou o quê, quando) são
**tokenizados**, não apagados, e o conflito entre a obrigação de apagar e a de preservar
é registrado — nunca resolvido em silêncio.

**RTBF vale para identidades de serviço?**
Não. É recusado para princípios `service` e `agent`: não há titular pessoal
(UC-015 E2).

**Que dado pessoal o módulo guarda?**
O mínimo para operar identidade: `display_name` e `external_ref`. `security.principal` é
a única tabela `pii` do módulo e declara base legal, conforme CV-08 do
`../005-Database/`.

---

## Armadilhas Comuns

| Armadilha | Sintoma | Correção |
|-----------|---------|----------|
| Persistir o material recebido | Credencial eterna sem trilha | Use em memória; renove a 50% do TTL. |
| Emitir sem `Idempotency-Key` | Segunda credencial válida criada em silêncio | Sempre envie a chave. |
| Ignorar `credential.rotating` | Interrupção ao fim da janela | Assine o evento e troque a quente. |
| Esperar revogação instantânea | Uso residual até o TTL | Trade-off consciente; derrube sessões no `020` se precisar de efeito imediato. |
| Ampliar `clock_skew_s` | Token expirado aceito por mais tempo | Corrija o NTP. |
| Desligar `attestation.required` | Qualquer contêiner pede credencial alheia | Corrija a atestação; é não-recarregável por segurança. |
| Aplicar perfil sem verificar assinatura | Isolamento não garantido | Verifique antes de aplicar (NFR-015). |
| Concentrar `sec:principal:write` e `sec:key:manage` no mesmo princípio | Escalada sem segundo par de olhos | Separação de funções (`./Security.md` §2). |
| Logar o cabeçalho `Authorization` | Token no sistema de logs | Removido antes da serialização (`./Logging.md` §5). |
| Tratar log como auditoria | Investigação depende de sistema volátil | Prova está em `../025-Audit/`; log é diagnóstico. |

---

## Referências

- Visão e escopo: `./Vision.md` · Segurança do módulo: `./Security.md`
- Exemplos: `./Examples.md` · API: `./API.md`
- Falhas e cerimônias: `./FailureRecovery.md` · Operação: `../029-Operations/`
