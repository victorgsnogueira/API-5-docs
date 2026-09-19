# ETL e NLP

> **Implementado e carregado** (set/2026): 463.016 linhas de fato, 52.696 artigos de
> doutrina, 408 temas. O pipeline é o conjunto de scripts em `scraping/` — Python + SQL —
> e **roda à mão**, não agendado. Ver [Carga manual](#carga-manual--o-processo) e
> [D-17](../06-operacao/02-decisoes-e-riscos.md#d-17--carga-manual-não-agendada).

## O desenho: um pipeline, vários conectores, três camadas no banco

```
   DataJud  ─┐                         ┌─ raw.*       payload cru (JSONB), um por fonte
   DOAJ     ─┤   harvest_*.py          │              idempotente: UNIQUE(source, payload_hash)
   SciELO   ─┼──────────────────────► ─┤
   OAI-PMH  ─┘   (um coletor por fonte) ├─ staging.*   DTO achatado, formato comum
                                        │              tratamento defensivo mora aqui
                 transform_load_*.py   ─┤
                                        └─ dw.*        modelo dimensional (fato + dims + pontes)
                                                       + camada semântica (NLP)
                 nlp_*.py              ──►             + agregados (views materializadas)
```

Acrescentar uma fonte é escrever um coletor que grava em `raw` e um extrator que achata
para `staging`. O que está em `dw` não muda.

**Escopo:** TJSP, TJRJ e TJMG. Todo coletor é escrito para esses três.

### Stack do pipeline

| Peça | O quê |
|---|---|
| Linguagem | **Python 3.12** |
| Banco | `psycopg2-binary` (carga em lote com `execute_values`) |
| HTTP | `requests`, com retry e backoff |
| Parsing | `beautifulsoup4` + `lxml` (OAI-PMH / HTML), `html.unescape` para entidade |
| Embeddings | `sentence-transformers` com `paraphrase-multilingual-MiniLM-L12-v2` — **local, CPU, sem chave de API**, 384 dimensões |
| Clusterização | `scikit-learn` (aglomerativo, cosseno) + `numpy` |
| Schema | migrations SQL numeradas (`scraping/sql/001…017`) |

> **Por que Python e não .NET.** O ecossistema de NLP (sentence-transformers, torch,
> scikit-learn) é Python. Com a carga manual, reescrever em .NET seria refazer o que
> funciona sem ganho — a API só lê o resultado.

## Extract

### DataJud — o que já se sabe

A API é um Elasticsearch atrás de um proxy:

```
POST https://api-publica.datajud.cnj.jus.br/api_publica_{tribunal}/_search
Authorization: APIKey <chave pública do CNJ>
```

| Fato | Consequência no desenho |
|---|---|
| Um índice por tribunal, sem endpoint agregado | três chamadas, uma por tribunal |
| Teto de 100 documentos por página | paginação obrigatória |
| Chave **pública**, divulgada pelo CNJ no wiki | não é segredo, mas vai por variável de ambiente |
| Limite de taxa e falha intermitente | retry com backoff |
| Não é tempo real | batch; streaming não traria nada |

Documentação: <https://datajud-wiki.cnj.jus.br/api-publica/>

### Filtrar na origem

A carga é sempre um **recorte deliberado**. Mesmo com três tribunais, o volume total é
grande demais para "carregar tudo". O filtro (assuntos, período) vai no corpo da
consulta, não depois.

### Demais fontes

Doutrina entrou (DOAJ, SciELO, OAI-PMH). Repositórios de jurisprudência dos tribunais
estão bloqueados por captcha/WAF — ver [Fontes](../03-dados/01-fontes.md) e
[Limitações](../03-dados/04-limitacoes-da-fonte.md). Toda fonte nova responde ao
questionário em [Fontes](../03-dados/01-fontes.md#fontes-ainda-não-listadas) antes de
virar coletor.

---

## Transform

### O que "normalizar" significa, campo a campo

Resumo de tudo o que a carga faz entre o payload cru e o dado que a API lê. Cada linha
é um passo que existe no código hoje.

| O quê | De → para | Onde | Regra |
|---|---|---|---|
| Texto | `&#8220;` / `&amp;` / espaço duplo → texto limpo | `clean()` no extrator de doutrina; `fix_html_entities.py` no dado já carregado | decodifica entidade HTML até estabilizar (há dupla codificação). ⚠ falta no extrator do DataJud |
| Número CNJ | 20 dígitos → `NNNNNNN-DD.AAAA.J.TR.OOOO` | `012_source_links.sql` | validado por teste (malformado = falha) |
| Data | ISO com `Z` ou `yyyyMMddHHmmss` → `dim_date` | `transform_load_datajud.py` | `dim_date` cobre 1940+; data fora do calendário não derruba a linha |
| Grau | `G1` / `G2` / `GRAU_UNICO` → `First` / `Second` / `Superior` | `COURT_LEVEL_MAP` | código fora do mapa passa como veio — não é inventado |
| Movimentação | código TPU → categoria de resultado + **polaridade** | `dim_movement` | só 6 códigos conferidos; o resto é neutro ([D-10](../06-operacao/02-decisoes-e-riscos.md#d-10--código-de-movimentação-não-conferido-não-entra-na-métrica)) |
| Classe processual | classe → **quem propõe** (`claimant_type`) | `dim_case_class` | acusação · fazenda · credor · defesa · autor particular ([Polaridade](../03-dados/05-polaridade-do-resultado.md)) |
| Assunto | variações de redação → **tema** | NLP Uso 1 + curadoria | "Indenização por Dano Moral" = "Indenizaçao por Dano Moral" |
| Tema | → **área jurídica** (`subject_area`) | NLP + mapa exato | 94,1% dos temas com área |
| Doutrina | artigo → tema | NLP Uso 4 | semântico **e** léxico ([D-16](../06-operacao/02-decisoes-e-riscos.md#d-16--doutrina-ligada-a-tema-por-método-híbrido-não-só-embedding)) |
| Processo | número → **link na origem** | `012_source_links.sql` | `direto` (TJSP) · `portal` (TJRJ, TJMG) · sem link |
| Tema | contagens → **nota de força** 0–100 | `013`/`017` | componentes gravados; teste recalcula a soma |

### 1 · Achatar

O JSON das fontes é aninhado e inconsistente — no DataJud, um mesmo campo ora vem
objeto, ora string, ora ausente, variando entre tribunais. Cada conector produz um DTO
plano antes de qualquer coisa tocar o banco. Todo tratamento defensivo mora aí.

### 2 · Traduzir a TPU

Códigos numéricos viram nome legível e, no caso das movimentações, viram **categoria de
resultado** — que é de onde sai o favorável/desfavorável do produto.

O DataJud **não publica resultado de julgamento como campo**: ele só existe como
movimentação. Construir esse mapa é trabalho de auditoria, código a código, conferindo o
nome que a própria API devolve.

> **A regra mais importante do ETL:** código não conferido cai em categoria neutra, e
> categoria neutra **não entra na métrica**. Na dúvida, fora. Um código mal classificado
> corromperia toda a apuração de resultado do produto sem nenhum sintoma visível.

> **✅ Feito.** 6 códigos conferidos contra a tabela oficial da TPU/CNJ — não só
> contra o nome que a API devolve:
>
> | Código | Nome | Categoria | Polaridade |
> |---|---|---|---|
> | 219 | Procedência | acolhida | pretensão do autor |
> | 220 | Improcedência | rejeitada | pretensão do autor |
> | 221 | Procedência em Parte | parcial | pretensão do autor |
> | 237 | Provimento | acolhida | **pretensão do recorrente** |
> | 238 | Provimento em Parte | parcial | **pretensão do recorrente** |
> | 239 | Não-Provimento | rejeitada | **pretensão do recorrente** |
>
> Os outros **257 códigos** vistos na carga entraram como neutros. Contam para
> volume e série temporal; não contam para resultado.

🔴 **A verificação revelou algo que esta seção não previa:** os códigos têm
**polaridades diferentes** — 219/220/221 se referem a quem propôs, 237/238/239 a
quem recorreu. Somá-los é erro metodológico. Toda a discussão está em
[Polaridade do resultado](../03-dados/05-polaridade-do-resultado.md), e a
dimensão de movimentação ganhou uma coluna `polarity_reference` com constraint:
**código conferido sem polaridade declarada não entra no banco**.

### 3 · Reconciliar entre fontes

Novo, e não trivial. Quando duas fontes falam do mesmo processo:

- **chave de casamento** — o número CNJ é o candidato natural; confirmar que todas as
  fontes o expõem;
- **precedência** — se DataJud e repositório do tribunal divergem sobre a mesma data,
  qual vence? Definir e registrar;
- **complementaridade** — o caso comum não é conflito, é uma fonte trazendo campo que a
  outra não tem (metadado do DataJud + inteiro teor do tribunal);
- **proveniência por campo** — quando o dado de um registro vem de fontes diferentes,
  saber qual campo veio de onde deixa de ser luxo.

### 4 · Normalizar em tema

Ver [NLP](#nlp--onde-o-modelo-entra) e
[O que é um tema](../01-produto/03-tema-modelo-conceitual.md).

---

## Load

### Idempotência é requisito

Rodar a carga duas vezes com os mesmos dados **não** pode duplicar linha nem inflar
contagem. Toda carga manual reprocessa janelas que se sobrepõem à anterior, e um fato
duplicado apareceria na tela como decisão a mais.

Como se garante: `UNIQUE(source, payload_hash)` no `raw`, upsert por chave natural nas
dimensões, e chave natural no fato (grão = movimentação, [D-13](../06-operacao/02-decisoes-e-riscos.md#d-13--grão-do-fato-movimentação-processual-opção-a)).
Verificado: recarregar o mesmo lote não muda a contagem.

### Proveniência em toda linha

Fonte e data de extração. É o que permite auditar, reprocessar uma fonte só, e exibir no
rodapé da tela de onde veio o dado.

### Fechamento da carga

Atualizar os agregados, na ordem de dependência. Ver
[Agregados OLAP](../03-dados/03-agregados-olap.md).

---

## Carga manual — o processo

**Não há carga agendada.** A raspagem é feita **à mão**, quando o time decide
atualizar a base: coletar → normalizar → validar → subir. Exatamente o processo que
produziu a base atual.

```
   1 COLETAR      harvest_*.py            fontes ──► raw.*
   2 ACHATAR      transform_load_*.py     raw ──► staging ──► dw (fato + dimensões)
   3 NORMALIZAR   nlp_*.py                embeddings · temas · área · doutrina↔tema
   4 REVISAR      curadoria humana        clusters novos aceitos/rejeitados por nome
   5 AGREGAR      REFRESH MATERIALIZED    na ordem de dependência
   6 VALIDAR      24 testes de integridade   ── qualquer linha retornada = PARA
   7 SUBIR        pg_dump ──► pg_restore  local ──► produção
   8 REGISTRAR    data, fontes, contagens   no README da carga e na Decisões
```

### Passo a passo

```bash
# 0. schema — só se houver migration nova
for f in scraping/sql/0*.sql; do docker exec -i api5-dw psql -U dw_admin -d api5_dw -v ON_ERROR_STOP=1 < "$f"; done

# 1. coletar (idempotente — pode rodar de novo)
python -u scraping/scripts/harvest_datajud.py tjsp,tjrj,tjmg 25000
python -u scraping/scripts/harvest_doaj.py
python -u scraping/scripts/harvest_scielo.py
python -u scraping/scripts/harvest_oai.py

# 2. achatar e carregar no modelo dimensional
python scraping/scripts/transform_load_doctrine.py
python scraping/scripts/transform_load_datajud.py

# 3. normalizar (NLP)
python scraping/scripts/nlp_embed.py                  # embeddings -> pgvector
python scraping/scripts/nlp_cluster_subjects.py 0.20  # PROPÕE clusters
python scraping/scripts/nlp_curate_themes.py          # 4. aplica a curadoria revisada
python scraping/scripts/nlp_load_themes.py            # dim_theme + bridge_theme_topic
python scraping/scripts/nlp_link_doctrine.py 0.55 10  # doutrina -> tema (semântico + léxico)

# 5. agregar — a ordem importa
#    case_current_result → topic_* → theme_summary/by_year/by_court → theme_strength

# 6. validar — todas as consultas devem voltar VAZIAS
docker exec -i api5-dw psql -U dw_admin -d api5_dw < scraping/sql/011_nlp_integrity_tests.sql
docker exec -i api5-dw psql -U dw_admin -d api5_dw < scraping/sql/014_strength_link_tests.sql
```

Comandos completos (inclusive a lista de `REFRESH`) em `scraping/README.md`.

### Por que o passo 4 é humano

A clusterização **propõe**; ela não decide. Na carga atual, de 69 clusters candidatos
**37 foram rejeitados** na revisão (`Furto | Roubo | Ameaça | Liminar…` num balde só).
Se aparecer assunto novo, alguém lê os clusters novos e registra a decisão **por nome**
em `nlp_curate_themes.py`. Assunto sem decisão fica 1:1 — degrada para a granularidade
da TPU, não quebra.

### Subir para produção

A carga roda **na máquina de quem opera**, contra o Postgres local; produção só recebe o
resultado validado:

```bash
# local — depois do passo 6 passar
docker exec api5-dw pg_dump -U dw_admin -d api5_dw -Fc -n dw -f /tmp/dw.dump
docker cp api5-dw:/tmp/dw.dump ./dw.dump

# produção — restore do schema dw inteiro, numa transação
pg_restore --clean --if-exists --single-transaction -n dw -d "$PROD_URL" dw.dump
```

- **Só o schema `dw` sobe.** `raw`, `staging` e `nlp` (embeddings) são área de
  trabalho, ficam locais (o `raw` é o que permite reprocessar sem voltar às fontes —
  mantenha backup dele). Produção **não tem pgvector** ([D-25](../06-operacao/02-decisoes-e-riscos.md#d-25--produção-sem-pgvector-embeddings-ficam-na-carga)).
- `--single-transaction`: se o restore falhar no meio, produção continua com a base
  anterior, inteira.
- Produção **nunca** roda coletor nem script de NLP.
- **Produção é o servidor do cliente** ([D-21](../06-operacao/02-decisoes-e-riscos.md#d-21--produção-na-intranet-do-cliente-em-windows-server)):
  o dump vai **no pacote de versão**, e o restore é feito pela TI do cliente, seguindo o
  manual de atualização ([Implantação no cliente](../06-operacao/04-implantacao-no-cliente.md#documentos-que-precisam-ser-escritos)).

### O que muda por não ser agendado

| Antes (agendado) | Agora (manual) |
|---|---|
| job no Coolify | nenhum contêiner de ETL em produção |
| alarme "a carga não rodou" | não se aplica — a data da última carga é **declarada** na tela e em `/health/ready` |
| carga competindo com a API por CPU | não há — a carga roda do nosso lado, fora do servidor de produção |
| falha silenciosa por dias | falha na frente de quem está rodando; o passo 6 impede subir base inconsistente |

**Continua valendo:** idempotência, proveniência em toda linha, e a regra de que o
produto declara **a data da extração** — dado manual envelhece do mesmo jeito.

## NLP — onde o modelo entra

Três usos possíveis, em ordem de valor por custo. Nenhum implementado.

### Uso 1 · Agrupar assuntos em tema  *(maior valor, começar por aqui)*

> **✅ Implementado — 15/09/2026.** 447 assuntos da TPU → **408 temas**.

**Problema.** Os mockups mostram cinco teses distintas para uma mesma consulta. Um
código de assunto da TPU não separa isso.

**Abordagem implementada.** Embeddings locais (`paraphrase-multilingual-MiniLM-L12-v2`,
CPU, sem chave de API) gravados em `pgvector`; clusterização aglomerativa por
distância de cosseno; e o rótulo escrito na curadoria.

| Etapa | Resultado |
|---|---|
| Embeddings | 447 assuntos + 52.696 títulos de doutrina |
| Clusterização (`threshold` 0,20) | 69 clusters multi-assunto **candidatos** |
| Curadoria | **32 aceitos, 37 rejeitados** |
| Temas | 408 (32 de merge + 376 mantidos 1:1) |
| Área jurídica | 384 de 408 (94,1%) |

**Requisito cumprido.** `dim_topic` (assunto bruto) não é destruída: `dim_theme`
é uma camada **por cima**, ligada por `bridge_theme_topic`. Se o agrupamento for
descartado, o produto degrada para a granularidade da TPU em vez de parar.

#### A clusterização sozinha produz lixo — a curadoria não é formalidade

Erros reais observados, todos rejeitados na revisão:

- `Furto | Roubo | Ameaça | Liminar | Injúria | Desacato | Mútuo | Férias | Marca`
  — embedding de nome curto e genérico colapsa tudo numa lixeira;
- `DIREITO TRIBUTÁRIO | DIREITO PENAL | DIREITO PREVIDENCIÁRIO` — são marcadores
  de **área**, não temas;
- `Nota Promissória | Citação`, `Imissão | Mandato`, `Compromisso | Consórcio`
  — juridicamente distintos.

E acertos que justificam o esforço: `Indenização por Dano Moral` +
`Indenizaçao por Dano Moral` (erro de digitação na fonte),
`Repetição de indébito` + `Repetição do Indébito`, `Planos de Saúde` +
`Planos de saúde`.

**A curadoria é registrada por NOME de assunto, não por `cluster_id`** — id de
cluster muda a cada execução do algoritmo. Assim a decisão é reproduzível e
revisável por alguém que nunca rodou o modelo.

### Uso 2 · Extrair fundamentos, precedentes e valores do inteiro teor

**Problema.** Os blocos *Fundamentos invocados*, *Jurisprudência qualificada* e a coluna
de valor exigem o **texto** da decisão.

**Bloqueio.** O DataJud não entrega texto. A pergunta a responder **antes** de escrever
qualquer código é: *de onde vem o inteiro teor?* — repositório do TJSP/TJRJ/TJMG, PANGEA,
ou nada. Ver [Limitações](../03-dados/04-limitacoes-da-fonte.md).

### Uso 3 · Redigir o texto da aba Resumo

**Abordagem.** Geração a partir **exclusivamente** das tabelas da base analítica — o
próprio mockup declara isso no rodapé.

**Guarda-corpo obrigatório.** Todo número no texto tem de existir em uma linha do DW, e
toda decisão citada tem de existir na dimensão de processos. Um resumo que alucina um
percentual destrói a credibilidade do produto — o público-alvo **cita** o que lê aqui.

### Doutrina — regra de produto que vale para o NLP

Se e quando a doutrina for extraída de texto, o resultado é **referência**, nunca
arquivo:

- artigo → **link** para onde está publicado;
- livro → **autor, obra, edição, capítulo** em texto.

Nunca PDF de livro. Ver [Fontes](../03-dados/01-fontes.md#fonte-5--doutrina).

### Uso 4 · Ligar doutrina a tema  *(não estava previsto, e funcionou)*

> **✅ Implementado.** 9.186 ligações, cobrindo 7.242 dos 52.696 artigos.

[Fontes](../03-dados/01-fontes.md#fonte-5--doutrina) listava "de onde sai a
associação entre doutrina e tema" como pergunta sem resposta, presumindo que
dependeria do inteiro teor. **Não depende**: dá para associar semanticamente o
título do artigo ao assunto da TPU.

#### A lição: só embedding não serve

Primeira tentativa, com embedding puro: 87.383 ligações, 67,6% de cobertura.
Número bonito, resultado errado. Para *Indenização por Dano Moral*, o modelo
trouxe **filosofia moral** pontuando acima do instituto jurídico:

| Artigo | Similaridade |
|---|---:|
| A CALIBRAÇÃO DA MORAL PELO POSITIVISMO | 0,903 |
| MORAL E CONTEMPORANEIDADE | 0,901 |
| *Dano Moral: Aspectos Históricos e de Quantificação* (o correto) | *0,854* |

Falso-amigo semântico: **"moral" (filosofia) × "dano moral" (instituto)**. E como
o erro pontuava **mais alto** que o acerto, **subir o limiar não resolveria** —
foi preciso mudar o método, não o parâmetro.

**Solução híbrida:** exigir proximidade semântica **e** presença léxica dos
termos distintivos do assunto no título (≥2 termos quando o assunto tem 2+).
Rejeitou **176.709 candidatos falsos**; a cobertura caiu para 13,7%.

Essa queda é a **direção certa de errar**, pela mesma lógica do D-10: artigo sem
tema é recuperável, artigo no tema errado não é. Toda ligação grava
`similarity`, `link_method` e `embedding_model` — reauditável e apertável depois.

Resíduo conhecido: títulos com os dois termos mas assunto diferente ainda passam
(ex.: *"PROSTITUIÇÃO: O IMPASSE ENTRE O LEGALISMO MORAL E O PRINCÍPIO DO DANO"*).

### O princípio comum — e ao chatbot

> O modelo pode **rotular**, **agrupar** e **redigir**. Nunca pode **contar**.
> Todo número vem de `SELECT`.

Mesma regra vale para o [chatbot](../01-produto/05-chatbot.md), pelo mesmo motivo.

**Como isso está travado na prática:** nenhum número do produto passa pelo
modelo — `theme_summary`, `theme_strength` e companhia são `SELECT` sobre o
fato. E há um teste que **recalcula a contagem direto do fato e compara com o
agregado materializado**, falhando se divergirem. A regra deixou de ser
promessa e virou consulta que roda.
