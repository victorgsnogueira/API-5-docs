# Tasks — Sprint 2

**Janela:** 05/10 a 25/10/2026 · **13 Stories · 65 SP** · 88 tasks · 303h

Convenções, camadas e o resumo geral em [README](README.md). O padrão da `Iteration` é `Sprint 2` em tudo o que está neste arquivo.

> **Duas Stories não têm fonte de dado hoje e estão marcadas 🔒:** a **US-19** (fundamentos invocados) e a **US-37** (acórdão por trás de cada afirmação). As duas dependem do inteiro teor, que está bloqueado nos quatro repositórios testados. A primeira task de cada uma é um spike, e a segunda é registrar a decisão; as tasks de implementação só são puxáveis depois que a decisão estiver escrita. O spike é um só, a task 19.1, e a 37.1 depende dele.

> Se a decisão for tirar as duas do escopo, a sprint perde 16 dos 65 SP, e as outras 11 Stories entregam a análise por ano, por tribunal, por órgão e a amostra auditável.

| Bloco | Tasks | Horas |
|---|---:|---:|
| Technical Foundation | 6 | 25h |
| Stories | 82 | 278h |
| **Total** | **88** | **303h** |

---

# Technical Foundation

Issue container. Tasks sem Story-mãe, numeração `0.Y`: o que a sprint precisa para que as Stories tenham onde rodar (schema, pipeline de dados, caminho de carga, esqueleto de tela e entrega).

```
Priority: Must
Estimate: —
Tasks técnicas sem User Story associada: 6 tasks · 25h.
```

### Pipeline and load path

| # | Task | Layer | Est. |
|---|---|---|---:|
| 0.47 | Top up the harvest for themes with fewer than 5 judged cases | ETL | 6h |
| 0.48 | Add the schema version to the load script and refuse a mismatched database | ETL | 4h |
| 0.49 | Add the load manifest with row counts and the extraction date | ETL | 3h |
| 0.52 | Publish the load script as a versioned pipeline release | DevOps | 3h |

**Descrições**

- **0.47**
  - `Data: nova rodada por códigos de assunto dos temas ralos, mantendo o filtro de julgamento`
  - `Verifies: o número de temas com 5 ou mais julgados aumenta, e o aumento é registrado na saída`
- **0.48** — `Verifies: carregar dados de uma versão de schema diferente da do banco falha com mensagem clara, antes de apagar qualquer coisa`
- **0.49** — `Data: versão do schema, data de extração e contagem de linhas de cada tabela`

### Environments

| # | Task | Layer | Est. |
|---|---|---|---:|
| 0.53 | Run the API on the homologation server so it creates the schema and the team loads the data file | DevOps | 4h |
| 0.54 | Add the end-to-end smoke test from search to theme detail | Test | 5h |

**Descrições**

- **0.53**
  - `Data: é o ensaio do que o cliente fará: criar o banco vazio, subir a API, carregar o arquivo`
  - `Verifies:` banco vazio → API sobe e migra → `psql -f` carrega → `/health/ready` passa a informar dado carregado
- **0.54** — `Verifies: busca → lista → detalhe → troca de aba, contra a homologação`

---

# Sprint 2

## US-03 — Como **usuário**, quero ver em cada resultado o score, a área do direito, o título da tese, um resumo curto, os tribunais, o volume, o período, a última decisão e o percentual favorável, para comparar teses antes de abrir qualquer uma delas

```
Priority: Must
Estimate: 8 SP
Epic: E1
State: ready
Depends on: US-01, US-06
Full DoR and acceptance criteria: https://github.com/Concord-API/API-5/blob/main/Docs/Scrum/acceptance-criteria.md#us-03--compare-theses-in-the-result-list
```

| # | Task | Layer | Est. |
|---|---|---|---:|
| 3.1 | Expose score, area, title, summary, courts, volume, period, last decision and upheld percentage per result | Backend | 6h |
| 3.2 | Return the short summary of up to two lines per theme | Backend | 3h |
| 3.3 | Return the court acronyms with the exact `+N` difference | Backend | 3h |
| 3.4 | Build the result item with the score circle and the area tag | Frontend | 6h |
| 3.5 | Build the `AlignmentBar` with the count next to it | Frontend | 4h |
| 3.6 | Render the court chips with the `+N` indicator only when needed | Frontend | 3h |
| 3.7 | Format the numbers in Portuguese and never recompute them | Frontend | 2h |
| 3.8 | Show the count instead of the percentage below the percentage floor | Frontend | 2h |
| 3.9 | Open the theme detail from the result item | Frontend | 2h |
| 3.10 | Test that the item renders every field of the contract | Test | 3h |

**Descrições**

- **3.1**
  - `Data:` o percentual vem com a frase de polaridade do tema (`claim_polarity_label`): a palavra "favorável" não existe no schema ([D-15](../../06-operacao/02-decisoes-e-riscos.md#d-15--a-palavra-favorável-não-existe-no-schema))
  - `Depends: 2.1, 9.4`
- **3.4** — `Data:` círculo de 70px, número em mono, `/100` abaixo; tag de matéria em preto
- **3.5** — `Verifies:` o percentual carrega o seu `n` na mesma linha
- **3.6** — `Verifies: com todos os tribunais visíveis o indicador não é renderizado`

*10 tasks · 34h*

## US-04 — Como **usuário**, quero filtrar os resultados por tribunal, período, instância e força mínima, vendo quantos processos cada tribunal tem, para restringir a lista ao meu caso

```
Priority: Must
Estimate: 5 SP
Epic: E1
State: awaiting decision 2
Depends on: US-03
Full DoR and acceptance criteria: https://github.com/Concord-API/API-5/blob/main/Docs/Scrum/acceptance-criteria.md#us-04--filter-results-to-the-shape-of-my-case
```

| # | Task | Layer | Est. |
|---|---|---|---:|
| 4.1 | Register the decision on the scope of STJ and STF | Backend | 2h |
| 4.2 | Accept court, period, instance and minimum strength on `GET /api/themes` | Backend | 6h |
| 4.3 | Return the case count per court with the filter options | Backend | 4h |
| 4.4 | Offer only the instances that exist in the loaded scope | Backend | 3h |
| 4.5 | Build the filter bar with the court counts | Frontend | 5h |
| 4.6 | Reflect the active filters in the URL | Frontend | 4h |
| 4.7 | Clear every filter and rebuild the list | Frontend | 2h |
| 4.8 | Describe the applied narrowing when nothing matches | Frontend | 3h |
| 4.9 | Test that the filtering happens in the API, never in the browser | Test | 3h |

**Descrições**

- **4.1**
  - `Data: decisão 2, prevista para o planejamento da Sprint 2 (05/10)`
  - `Data:` a task é entregável: vira entrada em [Decisões e riscos](../../06-operacao/02-decisoes-e-riscos.md)
- **4.2** — `Verifies: os filtros se combinam e são aplicados pela API; a lista nunca é filtrada no navegador`
- **4.4** — `Data:` grau vem do dado (`First`, `Second`, `SpecialCourt`, `AppealPanel`); "Superior" não aparece com o escopo em tribunais estaduais
- **4.6** — `Verifies: uma visão filtrada pode ser compartilhada por link`
- **4.8** — `Data: o estado vazio diz quais filtros estão ativos e oferece limpá-los`

*9 tasks · 32h*

## US-06 — Como **usuário**, quero um score de 0 a 100 dizendo o quanto o entendimento sobre o tema está consolidado, para saber se a tese vale ser defendida ou é uma briga aberta

```
Priority: Must
Estimate: 8 SP
Epic: E2
State: awaiting decision 4
Depends on: —
Full DoR and acceptance criteria: https://github.com/Concord-API/API-5/blob/main/Docs/Scrum/acceptance-criteria.md#us-06--the-0-to-100-score
```

| # | Task | Layer | Est. |
|---|---|---|---:|
| 6.1 | Register the decision on the coverage recalibration | Backend | 2h |
| 6.2 | Publish the reference example of the score in the wiki | Docs | 3h |
| 6.3 | Test that the score reproduces the published reference example | Test | 4h |
| 6.4 | Test the agreement edge cases: an even split is 0 and a unanimous topic is 1 | Test | 3h |
| 6.5 | Test that volume saturates and that a stale thesis has zero recency | Test | 3h |
| 6.6 | Withhold the score of a theme with no computed judgment | Backend | 2h |
| 6.7 | Expose the score in the list and detail contracts | Backend | 3h |
| 6.8 | Build the `StrengthScore` circle component | Frontend | 4h |
| 6.9 | Record the known limitations where the score is reached | Frontend | 2h |

**Descrições**

- **6.1** — `Data: decisão 4: enquanto não for tomada, o score não é exibido; hoje a cobertura considera 3 tribunais`
- **6.2** — `Data: um exemplo numérico completo, componente por componente`
- **6.7** — `Data: o cálculo já existe desde a task 2.1; aqui ele passa a fazer parte do contrato público`
- **6.9** — `Data: a recência usa só o ano da última decisão, não a densidade recente`

*9 tasks · 26h*

## US-07 — Como **usuário**, quero que o score venha com uma classificação em linguagem que já existe no meio jurídico — Consolidada, Dominante, Em formação, Divergente — para não ter de interpretar uma escala inventada

```
Priority: Must
Estimate: 2 SP
Epic: E2
State: awaiting decision 14
Depends on: US-06
Full DoR and acceptance criteria: https://github.com/Concord-API/API-5/blob/main/Docs/Scrum/acceptance-criteria.md#us-07--the-grade-in-legal-language
```

| # | Task | Layer | Est. |
|---|---|---|---:|
| 7.1 | Register the decision on the minimum judgments for the grade | Backend | 2h |
| 7.2 | Return the grade only above the minimum judgments | Backend | 3h |
| 7.3 | Show the grade next to the score | Frontend | 3h |
| 7.4 | Show the score without the textual grade below the minimum | Frontend | 2h |
| 7.5 | Test the band boundaries at 55, 75 and 90 | Test | 2h |

**Descrições**

- **7.1** — `Data: decisão 14: uma tese com 95% de concordância em 8 julgamentos não pode tomar emprestada uma autoridade que 8 casos não sustentam`
- **7.2** — `Data: faixas: Consolidada 90+, Dominante 75+, Em formação 55+, Divergente abaixo`

*5 tasks · 12h*

## US-08 — Como **usuário**, quero abrir a composição do score — os quatro componentes, seus pesos e a base de cálculo — para citar a estatística sabendo exatamente de onde ela vem

```
Priority: Must
Estimate: 3 SP
Epic: E2
State: ready
Depends on: US-06
Full DoR and acceptance criteria: https://github.com/Concord-API/API-5/blob/main/Docs/Scrum/acceptance-criteria.md#us-08--audit-the-scores-composition
```

| # | Task | Layer | Est. |
|---|---|---|---:|
| 8.1 | Expose the composition of the score: components, weights and basis | Backend | 4h |
| 8.2 | Build the score breakdown panel | Frontend | 5h |
| 8.3 | Link the methodology and the weights from the panel | Frontend | 2h |
| 8.4 | Write the methodology page | Docs | 3h |

**Descrições**

- **8.1** — `Data: quatro componentes com valor e peso, mais julgados, acolhidos, rejeitados, número de tribunais e ano da última decisão`
- **8.2** — `Verifies: nada é recalculado na tela: o total e os componentes chegam prontos`

*4 tasks · 14h*

## US-11 — Como **usuário**, quero ver como o alinhamento do tema evoluiu ano a ano, com a frase de tendência, para saber se o entendimento está se firmando ou mudando

```
Priority: Must
Estimate: 5 SP
Epic: E3
State: ready
Depends on: US-09
Full DoR and acceptance criteria: https://github.com/Concord-API/API-5/blob/main/Docs/Scrum/acceptance-criteria.md#us-11--see-the-alignment-evolve-over-time
```

| # | Task | Layer | Est. |
|---|---|---|---:|
| 11.1 | Create the yearly alignment aggregate per theme | Backend | 4h |
| 11.2 | Expose the yearly series ordered oldest to newest | Backend | 4h |
| 11.3 | Return the trend sentence values from the aggregate | Backend | 3h |
| 11.4 | Build the yearly alignment bars in the side column | Frontend | 5h |
| 11.5 | Highlight the most recent year and skip the years with no result | Frontend | 3h |
| 11.6 | Hide the trend sentence when there is data in a single year | Frontend | 2h |

**Descrições**

- **11.1** — `Verifies: índice único no agregado`
- **11.3** — `Data: "X% → Y% no sentido predominante", ambos vindos do agregado; não há frase com dados em um único ano`
- **11.5** — `Verifies: ano sem resultado não tem barra vazia`

*6 tasks · 21h*

## US-12 — Como **usuário**, quero ver o alinhamento do tema por tribunal, para saber se a tese se sustenta igual em São Paulo, Rio de Janeiro e Minas Gerais

```
Priority: Must
Estimate: 3 SP
Epic: E3
State: ready
Depends on: US-09
Full DoR and acceptance criteria: https://github.com/Concord-API/API-5/blob/main/Docs/Scrum/acceptance-criteria.md#us-12--see-the-alignment-per-court
```

| # | Task | Layer | Est. |
|---|---|---|---:|
| 12.1 | Create the alignment-by-court aggregate | Backend | 3h |
| 12.2 | Expose the alignment per court with its n | Backend | 3h |
| 12.3 | Test that the courts add up to the theme total | Test | 3h |
| 12.4 | Build the alignment-by-court list | Frontend | 4h |
| 12.5 | Show a court without judgments as having no data | Frontend | 2h |

**Descrições**

- **12.1** — `Verifies: índice único no agregado`
- **12.5** — `Verifies: tribunal sem julgamento não aparece com zero por cento`

*5 tasks · 15h*

## US-14 — Como **usuário**, quero uma tabela do comportamento de cada tribunal — decisões, alinhamento e data da mais recente — para comparar o meu tribunal com os outros

```
Priority: Must
Estimate: 3 SP
Epic: E4
State: ready
Depends on: US-12
Full DoR and acceptance criteria: https://github.com/Concord-API/API-5/blob/main/Docs/Scrum/acceptance-criteria.md#us-14--compare-each-courts-behaviour
```

| # | Task | Layer | Est. |
|---|---|---|---:|
| 14.1 | Expose the behaviour per court: decisions, alignment and last decision | Backend | 4h |
| 14.2 | Build the behaviour table with the visual rules | Frontend | 5h |
| 14.3 | Scroll a wide table inside its container | Frontend | 2h |
| 14.4 | Flag the amount column as unsourced | Frontend | 2h |

**Descrições**

- **14.2** — `Data: texto em serifa, número em mono alinhado à direita, cabeçalho em versalete, sem zebra e sem borda vertical`
- **14.3** — `Verifies: a página nunca rola na horizontal`
- **14.4** — `Verifies: a mediana de valor depende do inteiro teor: a coluna não aparece ou aparece sinalizada, nunca com valor`

*4 tasks · 13h*

## US-15 — Como **usuário**, quero uma amostra auditável dos processos por trás do tema, com câmara, data e resultado, para conferir os casos antes de citá-los

```
Priority: Must
Estimate: 5 SP
Epic: E4
State: ready
Depends on: US-09
Full DoR and acceptance criteria: https://github.com/Concord-API/API-5/blob/main/Docs/Scrum/acceptance-criteria.md#us-15--check-the-sample-of-cases-behind-the-topic
```

| # | Task | Layer | Est. |
|---|---|---|---:|
| 15.1 | Create the case view behind the theme sample | Backend | 3h |
| 15.2 | Expose `GET /api/themes/{key}/cases` with a stable ordering | Backend | 5h |
| 15.3 | Return "N of M decisions" with the sample | Backend | 3h |
| 15.4 | Flag a sealed case and hide its data | Backend | 3h |
| 15.5 | Return the reporter judge and amount columns empty and flagged | Backend | 2h |
| 15.6 | Build the auditable sample table | Frontend | 6h |
| 15.7 | Link every case number to the court with the formatted number | Frontend | 3h |
| 15.8 | Test that the ordering is the same between two calls | Test | 2h |

**Descrições**

- **15.1** — `Data: número formatado, tribunal, órgão julgador, classe, grau, data e resultado, mais fonte e link`
- **15.4** — `Data:` `secrecy_level` maior que 0; o DataJud público só entrega processos públicos, mas a regra existe
- **15.5** — `Data: o DataJud não publica o relator, e o valor depende do inteiro teor`
- **15.7** — `Data:` o rótulo é sempre "consultar no tribunal", nunca "veja a decisão"; `direto` abre o processo, `portal` abre o portal com o número para colar

*8 tasks · 27h*

## US-17 — Como **usuário**, quero saber se as câmaras do meu tribunal estão decidindo igual, para identificar divergência interna antes de decidir

```
Priority: Should
Estimate: 5 SP
Epic: E4
State: ready
Depends on: US-14
Full DoR and acceptance criteria: https://github.com/Concord-API/API-5/blob/main/Docs/Scrum/acceptance-criteria.md#us-17--know-whether-the-courts-panels-diverge
```

| # | Task | Layer | Est. |
|---|---|---|---:|
| 17.1 | Create the alignment-by-judging-body aggregate | Backend | 4h |
| 17.2 | Register the rule that flags a divergent panel | Backend | 3h |
| 17.3 | Expose the panels per court with their n, hiding a panel with one judgment | Backend | 4h |
| 17.4 | Test that a court's panels add up to the court total | Test | 3h |
| 17.5 | Build the panel block with the divergence flag | Frontend | 5h |

**Descrições**

- **17.1** — `Verifies: a soma dos órgãos de um tribunal fecha com o total do tribunal`
- **17.2**
  - `Data: o que é "divergir do padrão do próprio tribunal" precisa de limiar declarado`
  - `Data:` a task é entregável: vira entrada em [Decisões e riscos](../../06-operacao/02-decisoes-e-riscos.md)
- **17.3** — `Verifies: um caso isolado é ruído, não divergência`

*5 tasks · 19h*

## US-38 — Como **usuário**, quero sugestões de consultas frequentes na tela inicial, para entender que tipo de pergunta a ferramenta responde antes de digitar a minha

```
Priority: Could
Estimate: 2 SP
Epic: E1
State: ready
Depends on: US-01
Full DoR and acceptance criteria: https://github.com/Concord-API/API-5/blob/main/Docs/Scrum/acceptance-criteria.md#us-38--query-suggestions-on-the-home-screen
```

| # | Task | Layer | Est. |
|---|---|---|---:|
| 38.1 | Compute the frequent-query suggestions from the loaded data | ETL | 4h |
| 38.2 | Expose the suggestions | Backend | 3h |
| 38.3 | Render the suggestion chips and run the search on click | Frontend | 3h |
| 38.4 | Hide the suggestions area when there are none | Frontend | 2h |

**Descrições**

- **38.1** — `Data: temas de maior volume, com título em linguagem natural; nunca uma lista fixa no código`
- **38.4** — `Verifies: sem sugestões não há espaço vazio nem texto de erro`

*4 tasks · 12h*

## US-19 🔒 — Como **usuário**, quero ver os fundamentos invocados nas decisões do tema, com a frequência e a taxa de acolhimento de cada um, para escolher o argumento que mais vence e evitar o que sempre perde

```
Priority: Could
Estimate: 8 SP
Epic: E4
State: source to verify
Depends on: —
Full DoR and acceptance criteria: https://github.com/Concord-API/API-5/blob/main/Docs/Scrum/acceptance-criteria.md#us-19--pick-the-argument-that-wins-most
```

| # | Task | Layer | Est. |
|---|---|---|---:|
| 19.1 | Spike: check whether the full text of decisions is obtainable for the three courts | ETL | 6h |
| 19.2 | Register the decision: source, extraction feasibility or removal | ETL | 2h |
| 19.3 | Extract the grounds from the decision text | ETL | 6h |
| 19.4 | Store in which decisions each ground was identified | ETL | 4h |
| 19.5 | Count frequency and acceptance rate over the data, never from the model | ETL | 4h |
| 19.6 | Expose the grounds block | Backend | 4h |
| 19.7 | Render the grounds table | Frontend | 5h |

**Descrições**

- **19.1**
  - `Data:` os quatro repositórios foram testados e estão fechados ([Fontes](../../03-dados/01-fontes.md#fonte-4--repositórios-de-jurisprudência-dos-tribunais)); as saídas conhecidas, em ordem de custo: ofício ao TJRJ, convênio institucional, decisão da coordenação sobre automatizar o CAPTCHA, ou remover o bloco
  - `Data: é o mesmo spike que destrava a US-37`
- **19.2** — `Data:` a task é entregável: vira entrada em [Decisões e riscos](../../06-operacao/02-decisoes-e-riscos.md)
- **19.3** — `Depends: 19.2`
- **19.4** — `Verifies: todo fundamento extraído automaticamente é rastreável até as decisões`

*7 tasks · 31h*

## US-37 🔒 — Como **usuário**, quero ver no texto do entendimento o acórdão que sustenta cada afirmação, com a referência completa e a lista de decisões citadas ao pé, para poder citar a mesma decisão

```
Priority: Could
Estimate: 8 SP
Epic: E4
State: source to verify
Depends on: —
Full DoR and acceptance criteria: https://github.com/Concord-API/API-5/blob/main/Docs/Scrum/acceptance-criteria.md#us-37--see-the-ruling-behind-each-statement
```

| # | Task | Layer | Est. |
|---|---|---|---:|
| 37.1 | Register the decision on the full-text route for the citation markers | ETL | 2h |
| 37.2 | Attach a citation marker to a statement backed by a specific case | Backend | 5h |
| 37.3 | Return the cited-decisions list with case number, panel, date and synthesis | Backend | 4h |
| 37.4 | Test that every cited decision exists in the base | Test | 3h |
| 37.5 | Render the citation markers and the cited-decisions footer | Frontend | 6h |
| 37.6 | Show no dead button where the full text is missing | Frontend | 2h |

**Descrições**

- **37.1** — `Depends: 19.1: é a mesma investigação`
- **37.2** — `Depends: 37.1`
- **37.4** — `Verifies: nenhuma citação é gerada sem lastro`

*6 tasks · 22h*

