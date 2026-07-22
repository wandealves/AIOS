---
Documento: FAQ
Módulo: 005-Database
Status: Draft
Versão: 0.1
Última atualização: 2026-07-22
Responsável (RACI-A): Arquiteto do Módulo 005-Database
ADRs relacionados: ADR-0005, ADR-0051, ADR-0053, ADR-0055, ADR-0057, ADR-0058, ADR-0059
RFCs relacionados: RFC-0001, RFC-0050, RFC-0051
Depende de: Vision.md, Database.md, API.md, Examples.md
---

# 005-Database — Perguntas Frequentes

---

## Escopo

**O 005 é um serviço que fica entre minha aplicação e o banco?**
Não. O caminho de dados vai **direto** ao PostgreSQL pelo PgBouncer. O
`aios-database-svc` governa convenções, migrações, RLS, retenção e durabilidade — mas
não fica no caminho quente das suas queries (`./Architecture.md` §1). Colocá-lo lá
adicionaria um salto de rede por consulta e um ponto único de falha para toda escrita.

**Quem decide o que vai nas minhas tabelas?**
Você, o módulo dono. O 005 decide **como** aquilo é fisicamente armazenado, isolado,
migrado, particionado, retido e recuperado. A semântica é sua; a física é dele.

**Posso criar uma tabela direto no banco?**
Não. Nenhuma *role* de runtime tem `CREATE`. Toda tabela nasce de uma migração
governada — é o que garante que ela tenha `tenant_id`, RLS, retenção e índices desde o
primeiro dia, em vez de "depois a gente arruma".

**Redis e MinIO são deste módulo?**
Não. Estado quente volátil (Redis) e objetos binários (MinIO) estão fora do escopo —
exceto WAL e backups do próprio PostgreSQL, que são responsabilidade do 005
(`./Vision.md` §3.2).

**Por que eu não posso consultar o banco a partir de um cliente externo?**
Porque não existe caminho externo até o banco: todo acesso de fora passa pelo
`../004-API/`. Expor o PostgreSQL na borda anularia o modelo de autorização inteiro.

---

## Multi-tenancy e RLS

**Por que RLS em vez de um schema (ou banco) por tenant?**
Um schema com 10⁴ tenants é operável; 10⁴ schemas não são — cada migração viraria
O(n) e o catálogo do PostgreSQL degradaria. A RLS entrega o isolamento sem multiplicar
objetos (ADR-0051).

**Se minha query já filtra por `tenant_id`, a RLS é redundante?**
Não. A RLS existe justamente para o dia em que a query **esquecer** o filtro. Ela é
rede de segurança, não otimização — e é o único controle que continua valendo quando
há um bug na aplicação.

**Minha query retornou zero linhas e eu sei que há dados. Por quê?**
Provavelmente a sessão não definiu `app.tenant_id`. Sem ele, a política filtra tudo:
falha fechada por desenho. Use `SET LOCAL app.tenant_id` dentro da transação
(`./Examples.md` §10).

**Existe alguma forma legítima de consultar todos os tenants?**
Sim: a *role* `platform_admin` com `BYPASSRLS`, usada apenas por jobs de plataforma
(catálogo, retenção, auditoria) e **registrada em `../025-Audit/`** com finalidade
declarada. Nunca por um serviço de runtime.

**Posso desligar a RLS em desenvolvimento para agilizar?**
Não deveria. `db.rls.enforce` existe apenas para bootstrap local, e ambientes `dev` do
CI mantêm a RLS ligada — desligá-la mascara exatamente os defeitos que só apareceriam
em produção.

---

## Migrações

**Por que forward-only? Rollback não é mais seguro?**
Rollback de DDL destrutivo é ilusório: assim que a versão nova gravou dados na
estrutura nova, "voltar" perde informação. Por isso migrações destrutivas (fase
`contract`) são irreversíveis por definição, e a correção é uma **nova** migração
(ADR-0053).

**Minha migração foi recusada por `blocking_estimate_ms > 200`. E agora?**
Reescreva no padrão expand/contract: adicione o novo sem tocar no velho, migre os dados
em lotes, e só remova o velho depois que todas as instâncias estiverem atualizadas
(`./Examples.md` §4). O orçamento de 200 ms é o que torna migração compatível com
disponibilidade de 99,95%.

**Por que preciso de *dry-run* se o lint já passou?**
Lint verifica a **forma**; o *dry-run* mede o **custo real** com volume próximo ao de
produção. Um `CREATE INDEX` correto pode ainda assim levar 40 minutos — isso só
aparece no ensaio.

**Duas pipelines aplicaram migração ao mesmo tempo. O que acontece?**
A segunda recebe `AIOS-MIGRATION-0001`. A exclusão mútua é garantida por
`pg_advisory_lock` no próprio banco, não por disciplina operacional (invariante I1).

**Minha migração falhou no meio. O schema ficou inconsistente?**
Não. O PostgreSQL tem DDL transacional: ou tudo commitou, ou nada aconteceu
(invariante I2). O estado vai para `Failed` e você corrige com uma nova migração.

---

## Desempenho

**Por que minha consulta ficou lenta depois de meses em produção?**
Verifique se a tabela tem política de partição e retenção. Sem elas, o índice ativo
cresce indefinidamente. Com particionamento temporal + retenção por `DROP`, o *working
set* permanece proporcional ao presente (`./Scalability.md` §3).

**Posso aumentar `statement_timeout` para minha consulta pesada?**
Quase nunca é a resposta certa. Uma consulta legítima e recorrente acima de 30 s
indica índice faltante ou modelagem inadequada — abra uma migração de índice. O
timeout global protege todos os tenants (UC-015 E1).

**Por que HNSW e não IVFFlat?**
HNSW oferece latência estável sem etapa de treino, o que casa com um sistema onde os
vetores chegam continuamente. IVFFlat continua disponível para conjuntos muito grandes
com tolerância a *build* (ADR-0057).

**Meu recall está abaixo de 0,95. O que ajusto primeiro?**
`ef_search` — é recarregável e troca latência por qualidade sem reindexar
(`./Benchmark.md` §5). Só depois considere `m` maior, que exige reindexação.

**Por que não posso mudar a dimensão do meu vetor?**
Porque o índice e os dados existentes assumem a dimensão declarada. Trocar o modelo de
embedding exige nova coluna + migração de dados (`AIOS-DB-0011`).

---

## Operação

**Recebi `AIOS-DB-0006` (pool esgotado). Aumento o pool?**
Primeiro verifique se o serviço não está segurando conexões além do necessário
(transação longa, consulta sem índice). O bulkhead está protegendo os outros serviços;
elevá-lo apenas transfere a saturação para o primário.

**A leitura devolveu `AIOS-DB-0013`. É erro meu?**
Não: a réplica estava com lag acima do limiar e o sistema recusou servir dado obsoleto.
Repita apontando ao primário — o próprio pool costuma fazer isso automaticamente
quando `read_from_replica` está ativo.

**Temos backup diário. Isso basta para o RPO de 5 minutos?**
Não. O RPO é sustentado pelo **WAL archiving contínuo** (`archive_timeout = 60 s`), não
pelo backup full. Se o arquivamento parar, o RPO degrada imediatamente, mesmo com
backups em dia (F-09 em `./FailureRecovery.md`).

**Por que um backup "não verificado" não conta?**
Porque a única prova de que um backup funciona é ter restaurado a partir dele. Sem
*restore drill* bem-sucedido, ele é tratado como inexistente para fins de RTO/RPO
(FR-012).

**Fiquei sem partições futuras. É grave?**
Sim — é P1. Sem partição para o período corrente, a **escrita falha**. Execute
`POST /v1/database/partitions:ensure` e investigue por que o job não rodou (F-15).

---

## Conformidade

**Como comprovo que um dado pessoal foi apagado?**
Pelo `receipt_hash` do pedido de expurgo, registrado em `../025-Audit/` junto com as
tabelas afetadas e a contagem de linhas (UC-009). O comprovante sobrevive ao dado — por
isso o stream `DB_RETENTION` tem retenção de 7 anos.

**E se a lei me obrigar a apagar e outra obrigação me impedir?**
`legal_hold` prevalece: o expurgo é recusado com `AIOS-DB-0010` e o conflito fica
**registrado**. Esse tipo de colisão é decisão jurídica, não algo que o sistema deva
resolver silenciosamente.

**Se não dá para apagar fisicamente, o que acontece?**
Aplica-se tokenização irreversível, mantendo a integridade referencial, e isso consta
do comprovante (`./Security.md` §8).

**Por que minha tabela `pii` não passou no lint?**
Falta `legal_basis` na política de retenção (`AIOS-DB-0009`). Dado pessoal sem base
legal declarada não entra em produção.

---

## Armadilhas Comuns

| Armadilha | Sintoma | Correção |
|-----------|---------|----------|
| `SET` fora de transação com `pool_mode=transaction` | Comportamento intermitente, RLS "não funciona" | Use `SET LOCAL` dentro da transação (`./Examples.md` §10). |
| `ADD COLUMN … DEFAULT <volátil>` | Migração reescreve a tabela inteira | Adicione a coluna sem default e faça backfill em lotes. |
| `DELETE` massivo para expurgo | *Bloat*, autovacuum sobrecarregado | Use particionamento temporal + `drop_partition`. |
| FK sem índice | `DELETE` no lado referenciado varre a tabela | CV-05 — o lint rejeita. |
| `timestamp` sem timezone | Erro de fuso em retenção e partição | CV-04 — sempre `timestamptz` em UTC. |
| Confiar no `WHERE tenant_id` da aplicação | Vazamento em caso de bug | RLS obrigatória (CV-02). |
| `LISTEN/NOTIFY` para eventos | Não funciona com pooling em modo `transaction` | Use o padrão Outbox → NATS (`./Events.md`). |
| Índice criado sem `CONCURRENTLY` | Escrita bloqueada na produção | CV-12 — o lint rejeita. |

---

## Referências

- Visão e escopo: `./Vision.md` · Modelo físico: `./Database.md`
- Exemplos: `./Examples.md` · API: `./API.md`
- Segurança e LGPD: `./Security.md` · Operação: `../029-Operations/`
