# Tasks — Sprint 1

**Janela:** 07/09 a 27/09/2026 · **9 Stories · 44 SP** · 107 tasks · 396h

Convenções, camadas e o resumo geral em [README](README.md). O padrão da `Iteration` é `Sprint 1` em tudo o que está neste arquivo.

> **A pontuação que ordena a busca nasce nesta sprint.** A US-02 (Sprint 1) precisa do score para ordenar, e a US-06 (Sprint 2) é a que o expõe, valida e mostra. Por isso o cálculo está na task 2.1, e as tasks da US-06 só o testam e o publicam. A dependência no backlog aponta para o futuro, e as tasks a resolvem sem mudar a história.

> **Os filtros saíram para a Sprint 2** (US-04): a lista da Sprint 1 vem sem recorte.

| Bloco | Tasks | Horas |
|---|---:|---:|
| Technical Foundation | 41 | 163h |
| Stories | 66 | 233h |
| **Total** | **107** | **396h** |

**Já entregues antes deste arquivo** (não viram task): `0.10` Set up MSW for the frontend test suite (Test); `0.11` Run the Vitest suite in the frontend CI workflow (Frontend).

---

# Technical Foundation

Issue container. Tasks sem Story-mãe, numeração `0.Y`: o que a sprint precisa para que as Stories tenham onde rodar (schema, pipeline de dados, caminho de carga, esqueleto de tela e entrega).

```
Priority: Must
Estimate: —
Tasks técnicas sem User Story associada: 41 tasks · 163h.
```

### Schema and migrations

| # | Task | Layer | Est. |
|---|---|---|---:|
| 0.6 | Create a read-only database role for the API | Backend | 2h |
| 0.12 | Add the migration runner that applies versioned SQL scripts on startup | Backend | 6h |
| 0.13 | Take a database lock so two starts never migrate at once | Backend | 3h |
| 0.14 | Use one connection string to migrate and another, read-only, to serve | Backend | 3h |
| 0.15 | Create migration V001 with the court, judging-body, class, case, movement, outcome and date dimensions | Backend | 6h |
| 0.16 | Add the subject, theme and doctrine dimensions and their bridges to V001 | Backend | 5h |
| 0.17 | Add the event fact to V001 | Backend | 4h |
| 0.18 | Add the score configuration to V001 | Backend | 3h |
| 0.19 | Apply the migrations in the integration-test fixture | Test | 4h |
| 0.20 | Test that a migrated empty database reports "not loaded" on readiness | Test | 3h |
| 0.69 | Create the current-result aggregate, one result per case | Backend | 4h |
| 0.70 | Create the theme summary aggregate | Backend | 5h |

**Descrições**

- **0.6**
  - `Data:` GRANT de USAGE no schema `dw` e SELECT nas tabelas, nada além
  - `Verifies:` um INSERT com esse papel falha com `permission denied`
- **0.12**
  - `Data: scripts SQL embutidos no assembly, aplicados em ordem; um journal guarda versão e data`
  - `Verifies: a segunda execução não aplica nada`
- **0.13** — `Verifies: duas instâncias subindo juntas aplicam cada migration uma única vez`
- **0.14**
  - `Data:` `ConnectionStrings:RatioMigration` só na subida; `ConnectionStrings:Ratio` para servir
  - `Verifies: a conexão de serviço não escreve; falha de migração impede a subida e é gravada no log em arquivo`
- **0.15**
  - `Data:` chaves substitutas `_sk`; `case_number` único; `court_level`, `secrecy_level` e polaridade com `CHECK`; `source` e `extracted_at` em toda dimensão de conteúdo
  - `Verifies:` aplica num banco vazio que só tem `unaccent` e `pg_trgm`
- **0.16**
  - `Data:` `theme_key` público e único; `bridge_subject_doctrine` com método, score e modelo de embedding
  - `Verifies: ligação de doutrina sem método é rejeitada pelo banco`
- **0.17**
  - `Data:` grão = movimentação processual ([D-13](../../06-operacao/02-decisoes-e-riscos.md#d-13--grão-do-fato-movimentação-processual-opção-a)); `natural_key` única
  - `Verifies: inserir o mesmo evento duas vezes falha na chave natural`
- **0.18**
  - `Data:` pesos 0,45 / 0,25 / 0,20 / 0,10; `min_judged_for_percentage` = 2; `methodology_version`
  - `Verifies: pesos que não somam 1,0 são rejeitados`
- **0.19** — `Verifies: o Testcontainers sobe com o mesmo schema que a API cria em produção`
- **0.20** — `Verifies:` `/health/ready` distingue banco vazio de banco indisponível — os três estados
- **0.69**
  - `Data: o último movimento verificado que define resultado (procedência, improcedência ou parcial); só cerca de 31% dos processos têm resultado, o resto é andamento sem julgamento`
  - `Verifies:` índice único por processo, para o `REFRESH CONCURRENTLY` não bloquear a leitura
- **0.70**
  - `Data: processos, processos com resultado, acolhidos e rejeitados nas duas polaridades, período e última decisão; nenhuma coluna com a palavra "favorável"`
  - `Verifies: a contagem do agregado bate com a contagem direta no fato (teste de integridade)`

### Pipeline repository and load path

| # | Task | Layer | Est. |
|---|---|---|---:|
| 0.21 | Create the pipeline repository with the branch, commit and CI standards | DevOps | 4h |
| 0.22 | Read the database credentials from environment variables | ETL | 2h |
| 0.23 | Create the load database from the same migrations plus the pipeline schemas | ETL | 5h |
| 0.24 | Seed the courts, outcomes and calendar reference data | ETL | 3h |
| 0.25 | Generate the load script with truncate, copy and aggregate refresh in one transaction | ETL | 6h |
| 0.26 | Verify the load script on an empty migrated database | Test | 4h |
| 0.27 | Run the integrity tests in the pipeline CI | DevOps | 3h |

**Descrições**

- **0.21** — `Data: as mesmas convenções do projeto: branches por task, commits em inglês só com o assunto`
- **0.22** — `Verifies: nenhum segredo no repositório — varredura no CI`
- **0.23**
  - `Data:` o `dw` vem das migrations do backend; `etl`, `raw`, `staging` e `nlp` são do pipeline e nunca vão para a homologação nem para a produção; pgvector só aqui
  - `Verifies:` o schema `dw` da carga é idêntico ao que a API cria (comparação de `pg_dump -s`)
- **0.24**
  - `Data: 3 tribunais, 5 desfechos e o calendário de 1940 a 2027`
  - `Verifies: rodar duas vezes não duplica`
- **0.25**
  - `Data: TRUNCATE, COPY das tabelas e REFRESH dos agregados na ordem, dentro de uma transação`
  - `Verifies: falha no meio não deixa o banco pela metade`
  - `Data:` o arquivo é o que a homologação e o cliente carregam com `psql -f`
- **0.26**
  - `Verifies: banco vazio migrado + script + testes de integridade = todos vazios`
  - `Data:` os agregados chegam vazios depois de uma restauração só de dados; o `REFRESH` no fim do script é o que os popula

### Base data pipeline

| # | Task | Layer | Est. |
|---|---|---|---:|
| 0.28 | Download the TPU tables and derive the civil scope | ETL | 5h |
| 0.29 | Harvest DataJud civil cases per court, newest first, only cases with a judgment | ETL | 6h |
| 0.30 | Stratify the harvest quota by area of law | ETL | 4h |
| 0.31 | Store the raw documents idempotently by payload hash | ETL | 3h |
| 0.32 | Flatten each document into events and subjects | ETL | 5h |
| 0.33 | Resolve the movement names the source omits from the TPU | ETL | 3h |
| 0.34 | Load the case, class, judging-body and subject dimensions | ETL | 5h |
| 0.35 | Load the event fact idempotently by natural key | ETL | 4h |
| 0.36 | Guard the civil scope in the query, the transform and the load | ETL | 4h |
| 0.37 | Apply the movement semantics and the claimant rules after the load | ETL | 4h |
| 0.38 | Apply the source link per court after the load | ETL | 3h |

**Descrições**

- **0.28**
  - `Data: classes e assuntos cíveis e penais separados pela raiz da TPU, nunca por palavra no nome`
  - `Verifies: nenhum código penal entra na lista cível`
- **0.29**
  - `Data:` `terms` na classe cível, `must_not` nos assuntos penais, `terms` em `movimentos.codigo` [219, 220, 221, 237, 238, 239], ordenação por `@timestamp` decrescente
  - `Verifies:` todo processo coletado traz ao menos um julgamento e `dataHora` preenchida
- **0.30**
  - `Data: cota por área do direito, e não pelo que o tribunal mais processa`
  - `Verifies: nenhuma área concentra a amostra (a Execução Fiscal chegou a 70%)`
- **0.31** — `Verifies: coletar de novo não duplica documento`
- **0.32** — `Data:` `assuntos` pode vir como lista aninhada; evento sem `dataHora` é descartado, nunca datado por suposição ([D-11](../../06-operacao/02-decisoes-e-riscos.md#d-11--nada-de-dado-inventado))
- **0.33** — `Data: códigos sem nome que nem a TPU nomeia são descartados e contados na saída`
- **0.34**
  - `Data:` `subject_code` obrigatório: assunto sem código não entra
  - `Verifies: carregar duas vezes não duplica dimensão`
- **0.35**
  - `Data:` `número do processo | código da movimentação | instante`
  - `Verifies: o mesmo lote carregado duas vezes deixa a contagem igual`
- **0.36** — `Verifies:` a carga aborta se um assunto ou uma classe penal chegar ao `dw`
- **0.37**
  - `Data:` os 6 códigos verificados com sua polaridade; classe → quem propõe (`fazenda`, `credor`, `defesa`, `autor_particular`, `acusacao`)
  - `Verifies: código conferido sem polaridade falha; classe sem regra fica sem proponente, nunca inferido`
- **0.38**
  - `Data: TJSP com link direto (e-SAJ); TJRJ e TJMG com link de portal, mais o número formatado`
  - `Verifies: todo link tem tipo, e todo tipo tem link`

### Themes

| # | Task | Layer | Est. |
|---|---|---|---:|
| 0.39 | Generate the subject embeddings locally | ETL | 4h |
| 0.40 | Propose the subject clusters for review | ETL | 4h |
| 0.41 | Record the theme curation by subject name | ETL | 6h |
| 0.42 | Load the themes and their bridge to subjects | ETL | 4h |
| 0.43 | Assign a stable public key to every theme | ETL | 4h |
| 0.44 | Curate the product area of the themes without a tag, recording the basis | ETL | 4h |

**Descrições**

- **0.39** — `Data:` modelo `paraphrase-multilingual-MiniLM-L12-v2`, sem chamada a API externa ([D-25](../../06-operacao/02-decisoes-e-riscos.md#d-25--produção-sem-pgvector-embeddings-ficam-na-carga))
- **0.40** — `Data: a clusterização propõe, quem decide é a curadoria: em uma carga, 37 de 69 clusters foram rejeitados`
- **0.41**
  - `Data:` por nome de assunto, não por `cluster_id`, que muda a cada execução
  - `Verifies: assunto sem decisão fica 1:1, degradando para a granularidade da TPU`
- **0.42** — `Verifies: nenhum tema existe sem assunto de origem; nenhum assunto fica sem tema`
- **0.43**
  - `Data:` a chave é atribuída na primeira vez que o nome aparece e nunca reatribuída; o registro sobrevive à recarga ([D-31](../../06-operacao/02-decisoes-e-riscos.md#d-31--chave-pública-do-tema))
  - `Verifies: recarregar com os temas em outra ordem mantém todas as chaves`
- **0.44**
  - `Data: a base de cada decisão fica registrada: classe processual dominante, assuntos que co-ocorrem ou raiz da TPU`
  - `Verifies: todo tema com 5 ou mais julgados tem área do produto`

### Frontend

| # | Task | Layer | Est. |
|---|---|---|---:|
| 0.7 | Rename the frontend routes to `busca.tsx` and `tema.$key.tsx` | Frontend | 2h |
| 0.8 | Add TanStack Query wired to the router loader | Frontend | 4h |
| 0.9 | Create the API client and the zod schemas of the contract | Frontend | 5h |
| 0.45 | Call the API through relative paths with a development proxy | Frontend | 2h |
| 0.46 | Add the Portuguese number and date formatters | Frontend | 3h |

**Descrições**

- **0.7** — `Data:` a rota de detalhe recebe a chave pública do tema (`/tema/$key`, [D-34](../../06-operacao/02-decisoes-e-riscos.md#d-34--themes-na-api-theme_key-na-rota))
- **0.9** — `Verifies: resposta fora do contrato quebra no parse, não no meio da tela`
- **0.45** — `Data:` `/api/...` sem `VITE_API_URL`: o endereço do cliente não é conhecido no build
- **0.46**
  - `Data:` `12.418` e `21.08.2026`
  - `Verifies: nenhum número é recalculado na tela, só formatado`

---

# Sprint 1

## US-01 — Como **usuário**, quero digitar o tema do meu caso em linguagem natural e receber temas jurídicos curados — não uma lista de processos — para descobrir como ele vem sendo decidido sem garimpar um acórdão por vez

```
Priority: Must
Estimate: 8 SP
Epic: E1
State: ready
Depends on: —
Full DoR and acceptance criteria: https://github.com/Concord-API/API-5/blob/main/Docs/Scrum/acceptance-criteria.md#us-01--natural-language-topic-search
```

| # | Task | Layer | Est. |
|---|---|---|---:|
| 1.1 | Add the Portuguese text-search configuration with accent removal | Backend | 3h |
| 1.2 | Add the generated search columns and their indexes to the theme dimension | Backend | 3h |
| 1.3 | Create the theme search function combining text and trigram similarity | Backend | 6h |
| 1.4 | Create the synonym table and expand the query with it | Backend | 4h |
| 1.5 | Create `GET /api/themes` returning themes, never cases | Backend | 6h |
| 1.6 | Reject a query shorter than 3 characters with the Portuguese message | Backend | 2h |
| 1.7 | Return the highest-volume themes when the query is empty | Backend | 3h |
| 1.8 | Exclude themes without judged cases from the search | Backend | 2h |
| 1.9 | Curate the synonym seed from real queries | ETL | 3h |
| 1.10 | Build the home screen with the search field | Frontend | 6h |
| 1.11 | Wire the results route to `GET /api/themes` | Frontend | 5h |
| 1.12 | Reflect the search term in the URL | Frontend | 3h |
| 1.13 | Show the count message and the empty message with the declared scope | Frontend | 3h |

**Descrições**

- **1.1**
  - `Data:` `unaccent` aplicado antes do radical; o radical degrada (`indenização` → `indenizaca`), por isso plural e flexão ficam com a similaridade de trigramas
  - `Verifies: "inscricao indevida" e "inscrição indevida" devolvem o mesmo tema`
- **1.2** — `Data: vetor de busca e nome normalizado gerados pelo banco; índices GIN de texto e de trigrama`
- **1.3**
  - `Data: corta abaixo de 0,5 de rank, medido: fora de escopo chega a 0,36 e os acertos ficam entre 0,62 e 2,65`
  - `Verifies: "contrato de arrendamento de satélite" devolve vazio; "negativacao indevda" recupera o tema por similaridade`
- **1.4**
  - `Data: jargão forense fora do vocabulário da TPU ("negativação" → inclusão indevida em cadastro de inadimplentes)`
  - `Verifies: uma frase em linguagem natural com o jargão acha o tema`
- **1.5**
  - `Verifies: a resposta é lista de temas; nenhum número de processo aparece no resultado da busca`
  - `Data:` chave pública `themeKey` na resposta, nunca o `theme_sk`
- **1.6** — `Message: "Digite ao menos 3 caracteres para buscar."`
- **1.7** — `Verifies: busca vazia não é erro`
- **1.8** — `Verifies:` tema sem desfecho apurado não aparece — regra 2 de [tema](../../01-produto/03-tema-modelo-conceitual.md)
- **1.9** — `Data: a lista é curadoria manual, guardada no pipeline e enviada no arquivo de carga`
- **1.10** — `Data: o placeholder mostra tema em linguagem natural, nunca número de processo`
- **1.12** — `Verifies: recarregar a página ou abrir o link reproduz a mesma busca`
- **1.13** — `Message: "N temas encontrados para «termo»." e "Nenhum tema encontrado para «termo» no escopo TJSP, TJRJ e TJMG."`

*13 tasks · 49h*

## US-02 — Como **usuário**, quero os resultados ordenados por quão consolidado está o entendimento, e não por relevância de texto, para encontrar primeiro o que sustenta minha tese

```
Priority: Must
Estimate: 2 SP
Epic: E1
State: ready
Depends on: US-01, US-06
Full DoR and acceptance criteria: https://github.com/Concord-API/API-5/blob/main/Docs/Scrum/acceptance-criteria.md#us-02--results-ordered-by-strength-of-understanding
```

| # | Task | Layer | Est. |
|---|---|---|---:|
| 2.1 | Create the strength aggregate with the four components | Backend | 6h |
| 2.2 | Test that the score is recomputed from zero and matches the exposed value | Test | 4h |
| 2.3 | Order the theme list by strength score, descending | Backend | 3h |
| 2.4 | Break ties by decision volume | Backend | 2h |
| 2.5 | Return the judged count next to every score | Backend | 2h |
| 2.6 | Show the ordering label and the volume next to the score | Frontend | 3h |

**Descrições**

- **2.1**
  - `Data: concordância, volume, cobertura e recência com os pesos da configuração; a concordância nunca mistura pretensão do autor com a do recorrente`
  - `Depends: 0.18, 0.69, 0.70. O cálculo nasce aqui porque a ordenação precisa dele; a US-06 (Sprint 2) o valida, o expõe e o mostra`
- **2.2** — `Verifies: a fórmula não pode mentir: o teste recalcula a soma ponderada e falha se divergir`
- **2.3** — `Verifies: relevância textual não decide a ordem; ela só decide quem entra na lista`
- **2.5** — `Verifies: alto volume com decisões divididas não fica no topo só pelo volume`
- **2.6**
  - `Message: rótulo "ORDENADO POR FORÇA"`
  - `Data: a linha completa do item chega na US-03; aqui basta score e volume`

*6 tasks · 20h*

## US-09 — Como **usuário**, quero ler o entendimento do tema em prosa, abrindo com o número que responde à pergunta e com a contagem de casos ao lado de cada percentual, para entender o padrão sem abrir uma tabela

```
Priority: Must
Estimate: 8 SP
Epic: E3
State: ready
Depends on: US-06
Full DoR and acceptance criteria: https://github.com/Concord-API/API-5/blob/main/Docs/Scrum/acceptance-criteria.md#us-09--read-the-understanding-in-prose
```

| # | Task | Layer | Est. |
|---|---|---|---:|
| 9.1 | Register the decision on the source of the prose | Backend | 2h |
| 9.2 | Define the narrative template with slots for figures and basis | Backend | 4h |
| 9.3 | Add the theme narrative table with the text origin | Backend | 3h |
| 9.4 | Generate the narrative from the aggregates in the load | ETL | 6h |
| 9.5 | Write counts, not percentages, below the percentage floor | ETL | 3h |
| 9.6 | Expose header, lead and body in `GET /api/themes/{key}` | Backend | 5h |
| 9.7 | Return 404 with a Portuguese message for an unknown theme key | Backend | 2h |
| 9.8 | Build the theme header with score, area tag and metadata line | Frontend | 5h |
| 9.9 | Build the article column of the Resumo tab | Frontend | 6h |
| 9.10 | Print the judged count next to every percentage in the text | Frontend | 3h |
| 9.11 | Replace the unsourced blocks with an explanation of their absence | Frontend | 3h |

**Descrições**

- **9.1**
  - `Data: decisão 20: enquanto não há geração automática, o resumo é o texto curado, e o dado registra a origem`
  - `Data:` a task é entregável: vira entrada em [Decisões e riscos](../../06-operacao/02-decisoes-e-riscos.md)
- **9.2** — `Data:` o lead abre com o número que responde à pergunta, com seu `n` ("Em 82% das 12.418 decisões analisadas…")
- **9.3** — `Data:` `text_origin` (`template` ou `curated`), versão da metodologia e data de geração
- **9.4** — `Verifies: nenhum número do texto é produzido por modelo: todos vêm de SELECT`
- **9.5**
  - `Data:` abaixo de `min_judged_for_percentage` (2) o texto diz "1 decisão", nunca "100%"
  - `Verifies: nenhum percentual sobre n = 1`
- **9.7** — `Message: "Tema não encontrado."`
- **9.8** — `Data: a classificação em palavra (Consolidada, Dominante…) entra com a US-07`
- **9.10** — `Verifies: percentual sem n não é renderizado`
- **9.11** — `Data: citação de acórdão, botão de inteiro teor, marcadores de citação e rodapé de decisões citadas não aparecem, e o lugar deles explica por quê`

*11 tasks · 42h*

## US-10 — Como **usuário**, quero ver a distribuição de resultados do tema em uma figura com a fonte declarada, para ver de relance quanto é procedente, parcialmente procedente e improcedente

```
Priority: Must
Estimate: 5 SP
Epic: E3
State: awaiting decision 3
Depends on: US-09
Full DoR and acceptance criteria: https://github.com/Concord-API/API-5/blob/main/Docs/Scrum/acceptance-criteria.md#us-10--see-the-outcome-distribution-as-a-figure
```

| # | Task | Layer | Est. |
|---|---|---|---:|
| 10.1 | Register the decision on partially upheld claims | Backend | 2h |
| 10.2 | Create the outcome distribution aggregate by theme and polarity | Backend | 5h |
| 10.3 | Expose the outcome breakdown separated by category | Backend | 4h |
| 10.4 | Build the `Figure` component requiring the source prop | Frontend | 4h |
| 10.5 | Render FIG. 1 as horizontal bars with count and percentage | Frontend | 5h |
| 10.6 | Show counts only, with no percentage, below the percentage floor | Frontend | 2h |
| 10.7 | Replace the amount figure with an explanation of the missing source | Frontend | 2h |

**Descrições**

- **10.1** — `Data: decisão 3: o tratamento da procedência em parte é declarado onde o número é calculado`
- **10.2**
  - `Data: o agregado do tema soma a parcial com a procedência; a figura precisa dos três desfechos separados`
  - `Verifies:` índice único no agregado, para o `REFRESH CONCURRENTLY` não bloquear a leitura
- **10.3** — `Data: procedente, parcialmente procedente, improcedente; mérito e recurso nunca somados`
- **10.4** — `Verifies:` figura sem `source` não compila e não é renderizada
- **10.5** — `Data: barras retangulares, sem raio, sem eixo, sem grade, sem tooltip; fonte declarada abaixo`
- **10.7** — `Verifies: a figura de valor não aparece, e o lugar dela explica por quê`

*7 tasks · 24h*

## US-21 — Como **usuário**, quero ver a doutrina relacionada ao tema, com autor, obra e link para o artigo quando houver, para saber o que citar além da jurisprudência

```
Priority: Must
Estimate: 8 SP
Epic: E4
State: ready
Depends on: —
Full DoR and acceptance criteria: https://github.com/Concord-API/API-5/blob/main/Docs/Scrum/acceptance-criteria.md#us-21--know-what-to-cite-beyond-case-law
```

| # | Task | Layer | Est. |
|---|---|---|---:|
| 21.1 | Harvest doctrine from DOAJ | ETL | 4h |
| 21.2 | Harvest doctrine from SciELO | ETL | 4h |
| 21.3 | Harvest doctrine from the OAI-PMH repositories | ETL | 5h |
| 21.4 | Normalize and load the doctrine dimension idempotently | ETL | 5h |
| 21.5 | Link doctrine to subjects by semantic similarity above the declared threshold | ETL | 6h |
| 21.6 | Expose the related doctrine block | Backend | 5h |
| 21.7 | Return the unavailable reason when a theme has no doctrine above the threshold | Backend | 2h |
| 21.8 | Render the doctrine table with author, work and article link | Frontend | 5h |
| 21.9 | Declare on the block that the link is by similarity, with its threshold | Frontend | 2h |
| 21.10 | Render a book entry as a text reference without a PDF | Frontend | 2h |

**Descrições**

- **21.3** — `Data: EMERJ, EJEF/TJMG e IndexLaw`
- **21.4**
  - `Data: único por DOI e por (fonte, endereço); pelo menos um dos dois sempre existe`
  - `Verifies: carregar duas vezes não duplica artigo`
- **21.5**
  - `Data: cosseno ≥ 0,55 mais filtro léxico; método, score e modelo gravados em cada ligação`
  - `Verifies: nenhuma ligação sem score ou sem método`
- **21.6** — `Data:` é doutrina **relacionada**, ordenada por similaridade; nenhuma entrada é apresentada como citada por tribunal ([D-33](../../06-operacao/02-decisoes-e-riscos.md#d-33--histórias-removidas-do-backlog-e-doutrina-relacionada))
- **21.10** — `Verifies:` nenhum PDF de livro é servido — regra fechada em [Fontes](../../03-dados/01-fontes.md)

*10 tasks · 40h*

## US-24 — Como **usuário**, quero que toda tela deixe claro que os dados cobrem TJSP, TJRJ e TJMG, para não tirar uma conclusão nacional de um percentual que reflete três estados

```
Priority: Must
Estimate: 2 SP
Epic: E5
State: ready
Depends on: —
Full DoR and acceptance criteria: https://github.com/Concord-API/API-5/blob/main/Docs/Scrum/acceptance-criteria.md#us-24--know-the-coverage-scope
```

| # | Task | Layer | Est. |
|---|---|---|---:|
| 24.1 | Serve the declared scope from a single source | Backend | 3h |
| 24.2 | Return the scope in every API response | Backend | 3h |
| 24.3 | Show the scope on the results header and the detail footer, without interaction | Frontend | 3h |

**Descrições**

- **24.1**
  - `Data: tribunais lidos da dimensão de tribunal; matéria: cível`
  - `Verifies: mudar o escopo muda a declaração sem tocar no texto das telas`
- **24.3** — `Verifies: nenhuma tela com percentual sem declaração de escopo; nunca só em tooltip`

*3 tasks · 9h*

## US-25 — Como **usuário**, quero saber a fonte e a data de extração de todo número que estou vendo, para saber exatamente o que estou citando

```
Priority: Must
Estimate: 3 SP
Epic: E5
State: ready
Depends on: —
Full DoR and acceptance criteria: https://github.com/Concord-API/API-5/blob/main/Docs/Scrum/acceptance-criteria.md#us-25--know-the-source-and-date-of-every-number
```

| # | Task | Layer | Est. |
|---|---|---|---:|
| 25.1 | Create the global provenance aggregate | Backend | 3h |
| 25.2 | Create the per-theme provenance view | Backend | 3h |
| 25.3 | Expose provenance in every response | Backend | 4h |
| 25.4 | Omit a block that has no provenance | Backend | 3h |
| 25.5 | Build the `Provenance` component | Frontend | 3h |
| 25.6 | List every source that fed the page in the footer | Frontend | 4h |
| 25.7 | Test that the extraction date comes from the data, not from the clock | Test | 3h |

**Descrições**

- **25.1**
  - `Data: fonte, data de extração e volume por conjunto de dados`
  - `Verifies: a data vem do dado carregado, nunca do relógio do servidor`
- **25.2** — `Data:` blocos `casos` e `doutrina` separados, mais a versão da metodologia
- **25.3** — `Data:` `source`, `sourceUrl` e `extractedAt`, lidos das colunas que toda linha do DW carrega
- **25.4** — `Verifies: bloco sem procedência não é devolvido: procedência é requisito, não enfeite`
- **25.6** — `Verifies: produto multifonte: o rodapé lista todas as fontes, não só o DataJud`
- **25.7** — `Verifies: página renderizada hoje de uma carga da semana passada mostra a data da carga`

*7 tasks · 23h*

## US-26 — Como **usuário**, quero que a tela me diga o que não existe e por quê, em vez de mostrar um campo vazio ou um valor plausível, para não construir uma peça sobre dado que não existe

```
Priority: Must
Estimate: 3 SP
Epic: E5
State: ready
Depends on: —
Full DoR and acceptance criteria: https://github.com/Concord-API/API-5/blob/main/Docs/Scrum/acceptance-criteria.md#us-26--see-no-data-explained-instead-of-an-empty-field
```

| # | Task | Layer | Est. |
|---|---|---|---:|
| 26.1 | Define the three unavailable reasons in the response contract | Backend | 3h |
| 26.2 | Return the unavailable reason for each sourceless block | Backend | 4h |
| 26.3 | Return an empty list, not an error, when there is nothing to list | Backend | 2h |
| 26.4 | Build the `EmptyState` component reading the reason from the API | Frontend | 4h |
| 26.5 | Test that no sourceless block renders a placeholder value | Test | 3h |
| 26.6 | Link the unavailable explanation to the limitations page | Frontend | 2h |

**Descrições**

- **26.1** — `Data: a fonte não fornece; o dado ainda não foi carregado; não se aplica a este tema — cada um com seu texto, nunca a mensagem genérica`
- **26.2** — `Message: reason → "O DataJud não publica o relator." — texto em português, vindo da API`
- **26.4** — `Verifies: o frontend nunca escreve o motivo: ele exibe o que veio`

*6 tasks · 18h*

## US-13 — Como **usuário**, quero que a aba que estou vendo seja refletida na URL, para enviar a um colega o link exato da parte que quero mostrar

```
Priority: Could
Estimate: 5 SP
Epic: E3
State: ready
Depends on: US-09
Full DoR and acceptance criteria: https://github.com/Concord-API/API-5/blob/main/Docs/Scrum/acceptance-criteria.md#us-13--share-the-link-to-the-tab-i-am-on
```

| # | Task | Layer | Est. |
|---|---|---|---:|
| 13.1 | Validate the tab search param, defaulting to `resumo` | Frontend | 3h |
| 13.2 | Update the URL when the active tab changes | Frontend | 2h |
| 13.3 | Make the browser back button return to the previous tab | Frontend | 3h |

**Descrições**

- **13.1** — `Data:` `?aba=resumo | base`; valor inválido cai em `resumo`, nunca em tela quebrada
- **13.3** — `Verifies: voltar leva à aba anterior, não para fora do tema`

*3 tasks · 8h*
