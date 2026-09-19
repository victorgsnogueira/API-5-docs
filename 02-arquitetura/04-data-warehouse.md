# Data Warehouse

O uso de um DW com modelagem dimensional é **requisito do desafio**, não escolha
livre. O que estava em aberto era a tecnologia.

## A escolha: Postgres

Escolhemos Postgres 16. A dúvida registrada pelo time foi honesta: *"nunca montamos um
DW antes, talvez não seja a melhor opção."* Vale destrinchar, porque a resposta é
**sim, é a opção certa para este projeto** — mas por motivos que convém entender.

### Por que Postgres serve aqui

| Argumento | Detalhe |
|---|---|
| **Escala real do projeto** | O escopo é **três tribunais** (TJSP, TJRJ, TJMG) com recorte deliberado de assuntos. Ordem de grandeza de centenas de milhares a poucos milhões de linhas de fato. Postgres lida com isso sem transpirar |
| **Modelagem dimensional é agnóstica** | Esquema estrela é um *modelo*, não uma tecnologia. Fato + dimensões + tabela-ponte se implementam em Postgres exatamente como se implementam em Redshift |
| **OLAP de verdade** | `GROUP BY … FILTER (WHERE …)`, `DISTINCT ON`, funções de janela, views materializadas — tudo nativo |
| **Full-text em português** | `to_tsvector('portuguese', …)` com `unaccent` e `pg_trgm`. Resolve a busca do produto sem subir um Elasticsearch só para isso |
| **Uma peça de infraestrutura** | Banco, motor de busca e camada OLAP no mesmo processo. Menos coisa para configurar, versionar e explicar na banca |
| **Testável** | Migrations em SQL versionado, subida em Docker, banco descartável no CI |

### Onde Postgres deixaria de servir

Registrado para não haver ilusão. Você trocaria de tecnologia se:

- o volume de fato passasse de **dezenas de milhões** de linhas com agregação
  interativa sobre todas elas (aí a resposta é colunar: DuckDB, ClickHouse, Athena
  sobre Parquet);
- fosse preciso varrer o DataJud inteiro em vez de três tribunais com recorte;
- o requisito incluísse processamento distribuído (Spark/Databricks).

Nada disso está no escopo. **Trocar de tecnologia agora seria complexidade sem ganho** —
e o desafio avalia modelagem dimensional e pipeline, não a marca do banco.

> Se quiser um argumento colunar barato sem trocar de stack: `DuckDB` lê Parquet e
> roda o mesmo SQL analítico, e cabe como camada de exploração ao lado do Postgres.
> Mas isso é otimização; não faça antes de ter o pipeline de pé.

## O que está instalado no banco

Inventário do contêiner `api5-dw` — o **banco da carga** —, conferido direto no
catálogo do Postgres (`pg_extension`, `pg_database`, `pg_settings`) em 19/09/2026.

### Carga × produção

Os dois bancos **não são iguais**, de propósito
([D-25](../06-operacao/02-decisoes-e-riscos.md#d-25--produção-sem-pgvector-embeddings-ficam-na-carga)):

| | Banco da carga (`api5-dw`) | Produção (cliente) |
|---|---|---|
| Onde | contêiner, na máquina de quem roda a carga | **PostgreSQL 16 nativo em Windows Server** |
| Schemas | `raw`, `staging`, `dw`, `nlp` | **só `dw`** |
| Extensões | `vector`, `pg_trgm`, `unaccent` | **`pg_trgm`, `unaccent`** — sem pgvector |
| Embeddings | sim — é onde o NLP trabalha | **não** |
| Locale | ICU `pt-BR` | ICU `pt-BR` |

Os embeddings só servem para **produzir** o dado (clusterizar assuntos, ligar doutrina a
tema). O resultado — `dim_theme`, `bridge_theme_topic`, `bridge_topic_doctrine` com o
`similarity` gravado — é tabela comum. A API nunca consulta um vetor. Por isso o pgvector
fica só na carga, e produção roda o Postgres do instalador Windows padrão.

Os embeddings ficam em `nlp.topic_embedding` e `nlp.doctrine_embedding`
(migration `018_move_embeddings_to_nlp.sql`), chaveados pela SK da dimensão. O schema
`dw` não tem nenhuma coluna vetorial — e o teste de integridade 7 falha se voltar a ter.

> ✅ **Verificado em 19/09/2026:** `pg_dump -Fc -n dw` (14,7 MB) restaurado num
> `postgres:16` **sem pgvector**, com ICU `pt-BR`: mesmas contagens (463.016 fatos,
> 52.696 artigos, 408 temas, 9.186 ligações), as 9 views materializadas populadas, busca
> com `unaccent` funcionando e **os 24 testes de integridade vazios**. Só é preciso criar
> `pg_trgm` e `unaccent` antes do restore.

### Imagem e versão

| Item | Valor |
|---|---|
| Imagem | **`pgvector/pgvector:pg16`** — Postgres oficial + pgvector já compilado |
| Postgres | **16.15** (Debian 12 / bookworm) |
| Tamanho atual | ~844 MB (463.016 linhas de fato + 52.696 artigos + embeddings) |

> **Banco da carga:** `pgvector/pgvector:pg16` — a imagem oficial não traz o pgvector.
> **Testcontainers dos [testes](../07-justificativas/03-tdd.md) da API:** `postgres:16`
> (Debian, não Alpine), porque a API testa contra o que produção tem — sem pgvector.

### Extensões instaladas

| Extensão | Versão | Para quê | Onde é usada |
|---|---|---|---|
| **`vector`** (pgvector) — ⚠ **só no banco da carga** | 0.8.6 | embeddings da [camada semântica](05-etl-e-nlp.md#uso-1--agrupar-assuntos-em-tema--maior-valor-começar-por-aqui) | `dim_topic.embedding` e `dim_doctrine.embedding`, **`vector(384)`** — 447 + 52.696 vetores |
| **`pg_trgm`** | 1.6 | similaridade por trigrama — tolera erro de digitação na busca | índices GIN em `dim_doctrine.title`, `dim_doctrine.subject_area`, `fact_case_decision.summary`; `similarity()` na busca de temas |
| **`unaccent`** | 1.1 | "inscricao" acha "inscrição" | busca de temas |
| `plpgsql` | 1.0 | linguagem de função (padrão do Postgres) | — |

As três primeiras estão em `scraping/sql/001_schemas_extensions.sql`. `vector` não
estava prevista: entrou quando o agrupamento semântico saiu do papel — guardar os
embeddings no próprio banco é o que torna o agrupamento reproduzível e auditável sem
reprocessar texto. O pgvector também traz `halfvec`, `sparsevec` e os métodos de
índice **HNSW** e **IVFFlat** — disponíveis, ainda não usados.

### Disponíveis na imagem, não instaladas

Vêm no `contrib` da imagem; basta `CREATE EXTENSION`. Nenhuma é necessária hoje.

| Extensão | Quando instalar |
|---|---|
| **`pg_stat_statements`** | **recomendada** para o [monitoramento](../06-operacao/03-devops-e-infra.md#monitoramento--a-definir): mostra quais consultas da API são lentas. Exige `shared_preload_libraries = 'pg_stat_statements'` e restart |
| `btree_gin` / `btree_gist` | se um índice composto precisar misturar coluna comum com trigrama/JSONB |
| `pgcrypto` | se algum dia houver hash ou UUID gerado no banco |
| `citext`, `fuzzystrmatch`, `intarray`, `uuid-ossp`, `pg_prewarm` | sem uso previsto |

PostGIS **não** vem na imagem — e não há dado geográfico que o justifique.

### Locale e texto

| Item | Valor | Por quê |
|---|---|---|
| Provedor de locale | **ICU**, `pt-BR` | `ORDER BY` respeita acento e ordena "Ação" junto de "Acao" — verificado na prática |
| `LC_COLLATE` / `LC_CTYPE` | `C.utf8` | base do cluster; a ordenação vem do ICU |
| Encoding | `UTF8` | |
| Collations ICU disponíveis | `pt-BR-x-icu`, `pt-x-icu`, … | para `COLLATE` explícito quando precisar |
| Configuração de full-text `portuguese` | ✅ disponível | stemming em português |
| `default_text_search_config` | ⚠ **`english`** | ver abaixo |
| Fuso (`TimeZone`) | `UTC` | o banco grava em UTC; converter para `America/Sao_Paulo` só na exibição |

Criado com:

```
POSTGRES_INITDB_ARGS="--locale-provider=icu --icu-locale=pt-BR --encoding=UTF8 --locale=C.utf8"
```

> A imagem Debian não tem o locale de sistema `pt_BR.utf8` — por isso ICU, e não
> `LANG=pt_BR.utf8` (que a versão anterior desta página recomendava e não funciona).

> ⚠ **`to_tsvector()` sem configuração usa inglês.** O padrão do cluster é `english`.
> Sempre passe a configuração explicitamente — `to_tsvector('portuguese', …)` — ou
> fixe no banco: `ALTER DATABASE api5_dw SET default_text_search_config = 'portuguese';`.
> Hoje nenhuma consulta usa full-text (a busca é `unaccent` + `ILIKE` + `similarity()`),
> então nada quebrou — mas vai quebrar no primeiro `tsvector` escrito sem a config.

### Schemas e objetos

| Schema | Conteúdo | Tabelas | Índices |
|---|---|---|---|
| `raw` | payload cru (JSONB) por fonte — `datajud_case`, `doctrine_article`, `tjmg_decision` | 3 | 12 (inclui GIN no `payload`) |
| `staging` | DTO achatado — `case_event`, `case_decision`, `doctrine_article` | 3 | 8 |
| `dw` | modelo dimensional: 2 fatos, 10 dimensões, 4 pontes, `strength_config` | 17 | 55 |
| `dw` | **9 views materializadas** — `case_current_result`, `topic_summary`, `topic_by_year`, `topic_by_court`, `topic_by_judging_body`, `theme_summary`, `theme_by_year`, `theme_by_court`, `theme_strength` | — | índice único em cada (permite `REFRESH … CONCURRENTLY`) |
| `public` | só as extensões | — | — |

Detalhe das tabelas: [Modelo dimensional](../03-dados/02-modelo-dimensional.md). Das
views: [Agregados OLAP](../03-dados/03-agregados-olap.md).

### Índice vetorial — não se aplica a produção

Os embeddings **não têm índice HNSW/IVFFlat**, e só existem no banco da carga. Com ~53 mil vetores de 384 dimensões e
uso só na carga (clusterizar, ligar doutrina), a busca exata é rápida o bastante e dá o
resultado **exato**, que é o que a curadoria precisa.

Se um dia a busca semântica entrar no caminho de uma requisição (busca por significado,
chatbot), isso **reverte o D-25**: produção passa a precisar do pgvector compilado para
Windows. Nesse caso:

```sql
CREATE INDEX ON dw.dim_doctrine USING hnsw (embedding vector_cosine_ops);
```

### Parâmetros do servidor

Todos no **padrão da imagem**:

| Parâmetro | Valor | Nota |
|---|---|---|
| `shared_buffers` | 128 MB | subir para ~25% da RAM do servidor em produção |
| `work_mem` | 4 MB | baixo para `REFRESH` dos agregados; na carga, `SET work_mem = '256MB'` na sessão |
| `maintenance_work_mem` | 64 MB | idem para `CREATE INDEX` |
| `max_connections` | 100 | sobra — a API usa pool |
| `shared_preload_libraries` | vazio | precisa de `pg_stat_statements` para monitorar |

### Contêiner local

| Item | Valor |
|---|---|
| Nome | `api5-dw` |
| Banco / usuário | `api5_dw` / `dw_admin` (**superusuário** — só para dev) |
| Porta | `5432` no host |
| Volume | `api5_dw_data` → `/var/lib/postgresql/data` |
| Reinício | `unless-stopped` |

Comando de criação em [Ambiente local](../06-operacao/01-ambiente-local.md#banco).

### Papéis — o que falta para produção

Hoje existe **um único usuário, superusuário**. Em produção:

| Papel | Permissão | Quem usa |
|---|---|---|
| `ratio_api` | `USAGE` + `SELECT` no schema `dw`, nada mais | a API (`ConnectionStrings__Ratio`) |
| `ratio_loader` | dono dos schemas `raw`, `staging`, `dw` | o `pg_restore` da [carga manual](05-etl-e-nlp.md#subir-para-produção) |

A API é somente leitura por desenho; o banco deve garantir isso, não só o código.

## Configuração relevante

| Item | Valor | Motivo |
|---|---|---|
| Locale | ICU `pt-BR` na criação do banco | `ORDER BY` com acento |
| Configuração | por variável de ambiente | a configuração de produção é da máquina do cliente ([Implantação](../06-operacao/04-implantacao-no-cliente.md)) |

> **Migrations são SQL numerado** (`scraping/sql/001…018`), aplicadas em ordem com
> `psql -v ON_ERROR_STOP=1`. Aplicar via `docker-entrypoint-initdb.d` só funciona com
> volume vazio — não serve para evoluir esquema com dado dentro. Falta um controle de
> "qual migration já rodou" (tabela de versão, ou DbUp lendo os mesmos arquivos).

> **Backup é obrigatório** — e com a carga manual ele tem duas partes: o `dump` do
> schema `dw` que sobe para produção, e o **`raw` local**, que é o que permite
> reprocessar sem voltar às fontes com limite de taxa.

## Camadas dentro do banco

```
raw → staging      área de trabalho da carga — fica local, não sobe para produção
      │
tabelas base        fato + dimensões + pontes
      │             gravadas pela carga manual, nunca lidas direto pela API para agregar
      ▼
agregados           resultado vigente -> resumo por tema · por ano · por tribunal · por órgão
      │             atualizados ao fim de cada carga (REFRESH MATERIALIZED VIEW)
      ▼
API / chatbot       leem só os agregados
```

Detalhe das tabelas: [Modelo dimensional](../03-dados/02-modelo-dimensional.md).
Detalhe das views: [Agregados OLAP](../03-dados/03-agregados-olap.md).

## Testes de integridade

O desafio pede "testes automatizados validando integridade dos dados e consistência das
consultas". A lista está em
[DevOps e infraestrutura](../06-operacao/03-devops-e-infra.md#testes-de-integridade-do-dw--requisito-explícito),
junto com o pipeline que a executa.
