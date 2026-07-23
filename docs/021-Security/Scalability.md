---
Documento: Scalability
Módulo: 021-Security
Status: Draft
Versão: 0.1
Última atualização: 2026-07-22
Responsável (RACI-A): Arquiteto do Módulo 021-Security
ADRs relacionados: ADR-0211, ADR-0212, ADR-0213, ADR-0216
RFCs relacionados: RFC-0001, RFC-0210
Depende de: _DESIGN_BRIEF.md §10, 005-Database, 027-Cluster, Benchmark.md
---

# 021-Security — Escalabilidade e Concorrência

Meta: **≥ 10⁶ princípios** e **≥ 10⁷ credenciais ativas** com custo sub-linear
(NFR-007).

---

## 1. A Decisão que Define a Escala

```
   ❌ validação centralizada                ✅ validação local (adotado)
      cada requisição → 021                    JWT assinado + JWKS cacheado
                                               cadeia de CA verificada localmente

      10⁵ req/s no AIOS                        10⁵ req/s no AIOS
      = 10⁵ chamadas/s ao 021                  = ~0 chamadas/s ao 021
      + ~8 ms por requisição                   + ~50 µs por requisição
      021 = gargalo + SPOF total               021 só na EMISSÃO e na REVOGAÇÃO
```

Tudo o mais neste documento decorre de ADR-0211. Um módulo de identidade que participa
de cada requisição precisa escalar com o **tráfego** do sistema; um que participa apenas
da emissão escala com a **rotatividade de credenciais** — ordens de grandeza menor.

O preço dessa escolha é a revogação não ser instantânea, e ele é pago deliberadamente:
TTL de 15 min para tokens, lista de revogação com cache de 10 s, propagação-alvo de 30 s
(NFR-006).

---

## 2. Estratégias por Dimensão

| Dimensão | Estratégia |
|----------|-----------|
| Validação | **Local**, com JWKS cacheado no `../004-API/` e cadeia de CA verificada em memória pelos serviços. Zero rede até o 021. |
| Revogação | Evento (`token.revoked`) + lista consultável com cache de 10 s. Custo O(entradas ativas), não O(histórico), porque a entrada sai da lista quando a credencial expiraria de qualquer forma. |
| Emissão | Serviço **stateless**; réplicas escalam por CPU (assinatura é o custo dominante) ou por capacidade do HSM. |
| Persistência | `security.credential` particionada por **`expires_at`**: as vencidas ficam juntas e o expurgo é `DROP` de partição. |
| Cache de chaves | Chaves de assinatura em memória, invalidadas por evento; DEK em cache de TTL curto, nunca persistidas fora do envelope. |
| Concorrência de rotação | OCC por `version` + *advisory lock* por `purpose`+`principal` — garante a invariante I2 sem serializar rotações independentes. |
| Contenção de autenticação | Contadores por princípio em Redis. Contenção é **por princípio**, nunca global: atacar uma conta não pode negar serviço a todas. |
| Federação | JWKS do IdP externo em cache por tenant; a falha de um IdP externo afeta **apenas** aquele tenant. |
| Emissão em lote | Agentes recebem credenciais no *boot*, em lote pelo `../007-Agent-Runtime/`, evitando 10⁶ chamadas individuais. |

---

## 3. Curva de Escala

| Escala | Princípios | Credenciais ativas | Emissões/s (regime) | p99 emissão |
|--------|-----------|--------------------|---------------------|-------------|
| 10³ agentes | ~10³ | ~10⁴ | ~5 | ≤ 50 ms |
| 10⁴ agentes | ~10⁴ | ~10⁵ | ~50 | ≤ 50 ms |
| 10⁵ agentes | ~10⁵ | ~10⁶ | ~500 | ≤ 50 ms |
| **10⁶ agentes** | **~10⁶** | **~10⁷** | **~5.000** | **≤ 50 ms** |

A taxa de emissão cresce **linearmente** com a frota — não com o tráfego. É essa
distinção que mantém a curva viável: 10⁶ agentes renovando credenciais de 24 h geram
~12 emissões/s de renovação em regime; o pico de 5.000/s ocorre em eventos de *boot* em
massa, e é para ele que o módulo é dimensionado.

---

## 4. Por que o particionamento é por `expires_at`

| Chave de partição | Consequência |
|-------------------|--------------|
| `created_at` (intuitivo) | Credenciais de TTLs diferentes criadas no mesmo dia expiram em dias diferentes → cada partição tem uma mistura de vivas e mortas → expurgo exige `DELETE`. |
| **`expires_at`** (adotado) | Todas as credenciais que morrem no mesmo período ficam **juntas** → expurgo é `DROP` de partição, O(1), sem *bloat*. |

O mesmo raciocínio vale para a lista de revogação: a entrada expira junto com a
credencial, mantendo `aios_sec_revocation_list_size` proporcional às **credenciais
ativas**, não ao histórico de revogações.

---

## 5. Concorrência

| Mecanismo | Onde | Motivo |
|-----------|------|--------|
| OCC (`version`) | Todas as entidades | Conflitos são raros; retry é mais barato que lock. |
| *Advisory lock* por `purpose`+`principal` | Rotação | Garante a invariante I2 (nunca três utilizáveis) sem serializar rotações de propósitos distintos. |
| Índice único parcial | `security.key` (`active` por papel) | Impede duas chaves ativas para o mesmo papel — a escolha de qual usar seria não-determinística. |
| `Idempotency-Key` | Emissão | Repetir sem a chave produziria uma **segunda credencial válida** — ampliação silenciosa da superfície. |
| Cache de decisão do PDP | `PolicyClient` | TTL curto; evita consultar o `022` a cada emissão em lote. |

---

## 6. Escala Horizontal e Limites

| Componente | Escala | Limite prático |
|------------|--------|----------------|
| `aios-security-svc` | Horizontal (stateless) | Escala até o limite do PostgreSQL/KMS. |
| `aios-security-ca` | Horizontal | Limitado pela capacidade de assinatura do HSM. |
| **HSM** | **Vertical / por dispositivo** | **Gargalo estrutural da emissão.** |
| Lista de revogação | — | Proporcional às credenciais ativas, não ao histórico. |
| PostgreSQL (`security`) | Conforme `../005-Database/` | Particionado; leitura em réplica para consultas de catálogo. |

Caminho de crescimento quando o HSM satura, em ordem de preferência:

```
   1. Aumentar TTL de credenciais de baixo risco  → menos renovações por unidade de tempo
   2. Emissão em lote no boot de agentes          → menos chamadas individuais
   3. Cache de DEK com TTL maior                  → menos operações de unwrap
   4. HSM adicional / cluster de HSM              → mais capacidade de assinatura
   5. Chave de assinatura em memória para tokens  → último recurso; reduz a garantia
```

O passo 5 é explicitamente o **último**: ele troca segurança por capacidade e só deveria
ser considerado com decisão registrada em ADR, nunca como ajuste operacional.

---

## 7. Teste de Escala (T-SCALE-01)

| Parâmetro | Valor |
|-----------|-------|
| Princípios | 10⁶ (10⁴ serviços, 10⁶ agentes, 10³ humanos) |
| Credenciais ativas | 10⁷ |
| Credenciais históricas | 10⁹ em partições por `expires_at` |
| Lista de revogação | 10⁵ entradas ativas |
| Carga | perfil da §3 de `./Benchmark.md`, 50.000 req/s no `004` |
| Duração | 24 h |

**Critérios:** p99 de emissão e introspecção estáveis; expurgo de partições sem impacto
em latência; lista de revogação proporcional às ativas; **zero** chamadas ao 021 no
caminho de validação.

---

## 8. Limites Conhecidos e Mitigações

| Limite | Impacto | Mitigação |
|--------|---------|-----------|
| Revogação não instantânea | Janela de uso até o TTL/cache | TTL de 15 min; cache de 10 s; propagação ≤ 30 s (NFR-006). Aceito conscientemente em troca da escala (§1). |
| HSM como gargalo de emissão | Teto de emissões/s | Ordem de crescimento da §6; dimensionamento medido em `./Benchmark.md` §5 (T-3). |
| Bootstrap circular | Credencial inicial não emitida pelo próprio módulo | Credencial de bootstrap única, documentada, de rotação manual e monitorada como risco permanente (`./Deployment.md` §3). |
| Cache de JWKS desatualizado | Tokens novos rejeitados | Publicar a chave **antes** de assinar com ela; sobreposição de 30 dias; evento `jwks.rotated`. |
| 10⁶ princípios em uma tabela | Custo de consulta | Índices por `(tenant_id, status)`; consultas de catálogo em réplica de leitura. |
| Contenção por princípio consome Redis | Memória proporcional a princípios sob ataque | TTL curto nos contadores; contenção é por princípio, e o conjunto sob ataque é pequeno. |

---

## 9. Rumo a Milhões

A combinação que sustenta 10⁶ agentes:

1. **Validação local** — o módulo não participa do caminho quente (a decisão dominante).
2. **TTL curto com renovação distribuída** — a carga de emissão é contínua e previsível,
   não concentrada.
3. **Emissão em lote no boot** — 10⁶ agentes não geram 10⁶ chamadas individuais.
4. **Particionamento por `expires_at`** — expurgo O(1); o *working set* é o presente.
5. **Lista de revogação proporcional às ativas** — não cresce com o histórico.
6. **Contenção por princípio** — um ataque a uma conta não degrada as demais.
7. **Serviço stateless** — réplicas escalam livremente; o estado está no banco e no HSM.

Nenhum desses sete é otimização tardia: todos decorrem de decisões tomadas no desenho
(ADR-0211, ADR-0212, ADR-0213, ADR-0216) e são verificáveis por métrica contínua.

---

## 10. Referências

- Brief: `./_DESIGN_BRIEF.md` §10
- Medição: `./Benchmark.md` · Falhas sob carga: `./FailureRecovery.md`
- Modelo físico e particionamento: `./Database.md` · `../005-Database/`
- Topologia: `./Deployment.md` · `../027-Cluster/`
