# Tasks — Sprint 3

**Janela:** 02/11 a 22/11/2026 · **7 Stories · 40 SP** · 57 tasks · 212h

Convenções, camadas e o resumo geral em [README](README.md). O padrão da `Iteration` é `Sprint 3` em tudo o que está neste arquivo.

> **O chatbot (US-34, US-35, US-36) tem uma decisão aberta antes de qualquer código de modelo:** o NFR-12 diz que a aplicação funciona sem internet em tempo de execução, e o desenho do chatbot deixa em aberto qual modelo e qual provedor. As tasks 34.2 e 34.3 resolvem isso; as ferramentas de consulta (34.4 e 34.5) não dependem dela e já são puxáveis, porque o desenho manda testá-las isoladamente, sem modelo no caminho.

| Bloco | Tasks | Horas |
|---|---:|---:|
| Technical Foundation | 14 | 56h |
| Stories | 43 | 156h |
| **Total** | **57** | **212h** |

---

# Technical Foundation

Issue container. Tasks sem Story-mãe, numeração `0.Y`: o que a sprint precisa para que as Stories tenham onde rodar (schema, pipeline de dados, caminho de carga, esqueleto de tela e entrega).

```
Priority: Must
Estimate: —
Tasks técnicas sem User Story associada: 14 tasks · 56h.
```

### Backend delivery

| # | Task | Layer | Est. |
|---|---|---|---:|
| 0.55 | Run the API as a Windows service listening on 127.0.0.1 | Backend | 4h |
| 0.56 | Honour the forwarded headers behind the proxy | Backend | 2h |
| 0.57 | Test that the service starts with the configuration outside the package | Test | 3h |

**Descrições**

- **0.55** — `Data:` `UseWindowsService()`; o acesso é pelo NGINX
- **0.56** — `Verifies: esquema e origem corretos no log atrás do NGINX`
- **0.57** — `Verifies:` connection string e caminhos vêm de `appsettings.Production.json` ou de variável do serviço, nunca do build

### Proxy and package

| # | Task | Layer | Est. |
|---|---|---|---:|
| 0.58 | Write the NGINX configuration with SPA fallback, timeouts and security headers | DevOps | 5h |
| 0.59 | Restrict access by network only | DevOps | 3h |
| 0.60 | Register NGINX as a Windows service | DevOps | 3h |
| 0.61 | Assemble the release package with the API, the frontend build, the NGINX config, the load script and the install scripts | DevOps | 6h |
| 0.68 | Freeze the load script for delivery and record its version and extraction date | ETL | 2h |

**Descrições**

- **0.58** — `Data:` `try_files ... /index.html` para `/tema/123` não dar 404 ao recarregar; TLS com o certificado interno do cliente
- **0.59** — `Data: sem login: quem está na intranet usa; a restrição é da rede`
- **0.61** — `Data: o cliente recebe só arquivos buildados: nenhum SDK, Node, Python ou Git`

### Manuals and rehearsal

| # | Task | Layer | Est. |
|---|---|---|---:|
| 0.62 | Write the deployment manual | Docs | 6h |
| 0.63 | Write the update manual with the rollback path | Docs | 5h |
| 0.64 | Write the machine specification | Docs | 3h |
| 0.65 | Document the release package contents | Docs | 2h |
| 0.66 | Rehearse the full installation on the homologation server from the package only | DevOps | 6h |
| 0.67 | Rebuild the load database from scratch and compare it with the previous load | Test | 6h |

**Descrições**

- **0.62** — `Data: instalar e configurar a partir só dos arquivos buildados: PostgreSQL, banco e usuários vazios, serviço da API, NGINX, verificação`
- **0.63** — `Data: como aplicar uma versão nova e uma carga nova sem perder a anterior`
- **0.64** — `Data: Windows Server, CPU, RAM, disco, portas, certificado e contas de serviço`
- **0.66** — `Verifies: seguindo só o manual, um servidor limpo chega ao produto funcionando`
- **0.67** — `Verifies: contagens e agregados equivalem aos da carga anterior, e cada diferença é explicada`

---

# Sprint 3

## US-27 — Como **usuário**, quero exportar para CSV as decisões que sustentam o tema, para trabalhar os dados fora da ferramenta

```
Priority: Should
Estimate: 5 SP
Epic: E5
State: ready
Depends on: US-15
Full DoR and acceptance criteria: https://github.com/Concord-API/API-5/blob/main/Docs/Scrum/acceptance-criteria.md#us-27--export-the-topics-decisions
```

| # | Task | Layer | Est. |
|---|---|---|---:|
| 27.1 | Generate the CSV on the server from the same rows as the screen | Backend | 6h |
| 27.2 | Apply the screen filter to the export | Backend | 3h |
| 27.3 | Write the source, the extraction date and the scope in the file header | Backend | 3h |
| 27.4 | Encode the file for Portuguese spreadsheets | Backend | 3h |
| 27.5 | Declare the row ceiling and return it when exceeded | Backend | 3h |
| 27.6 | Leave the unsourced fields as empty columns | Backend | 2h |
| 27.7 | Add the export button and the ceiling message | Frontend | 4h |
| 27.8 | Test that the file has the same rows as the screen for the same filter | Test | 4h |

**Descrições**

- **27.1** — `Data:` `GET /api/themes/{key}/export?format=csv`, gerado a partir da mesma consulta da amostra; nunca montado no navegador
- **27.4**
  - `Data:` UTF-8 com BOM, separador `;`, vírgula decimal
  - `Verifies: acentos, números e datas abrem corretos no Excel em português`
- **27.7** — `Verifies: acima do teto a interface diz qual é, em vez de falhar sem explicação`

*8 tasks · 28h*

## US-28 — Como **usuário**, quero copiar a citação do tema já com a fonte, a data de extração, o escopo e o `n`, para colar sem redigitar

```
Priority: Should
Estimate: 3 SP
Epic: E5
State: ready
Depends on: US-25
Full DoR and acceptance criteria: https://github.com/Concord-API/API-5/blob/main/Docs/Scrum/acceptance-criteria.md#us-28--copy-the-citation-ready-to-paste
```

| # | Task | Layer | Est. |
|---|---|---|---:|
| 28.1 | Assemble the citation text in one or two plain lines | Backend | 4h |
| 28.2 | Copy the citation to the clipboard with a visible confirmation | Frontend | 4h |
| 28.3 | Test that the citation carries the number, the n, the source, the date and the scope | Test | 3h |
| 28.4 | Test that a block without a source never enters the citation | Test | 2h |

**Descrições**

- **28.1**
  - `Data: sem marcação e sem quebra de linha que force reedição`
  - `Verifies: nada sem lastro entra na citação`

*4 tasks · 13h*

## US-29 — Como **usuário**, quero saber quando os dados foram atualizados pela última vez, e ser avisado quando estiverem defasados, para não me apoiar em um retrato de meses atrás

```
Priority: Should
Estimate: 3 SP
Epic: E5
State: awaiting decision 16
Depends on: US-25
Full DoR and acceptance criteria: https://github.com/Concord-API/API-5/blob/main/Docs/Scrum/acceptance-criteria.md#us-29--know-whether-the-data-is-current
```

| # | Task | Layer | Est. |
|---|---|---|---:|
| 29.1 | Register the decision on the staleness threshold | Backend | 2h |
| 29.2 | Expose the last extraction date and the stale flag | Backend | 3h |
| 29.3 | Show the extraction date on every screen with numbers | Frontend | 2h |
| 29.4 | Show the stale-data warning past the threshold | Frontend | 3h |
| 29.5 | Make the readiness check report the age of the last extraction | Backend | 3h |

**Descrições**

- **29.1** — `Data: decisão 16: a partir de quantos dias o dado é dado como velho`
- **29.5** — `Verifies: uma carga que não chega há dias nunca aparece com cara de dado atual`

*5 tasks · 13h*

## US-30 — Como **usuário**, quero saber quanto tempo normalmente se leva do ajuizamento à decisão neste tema, para calibrar a expectativa do meu cliente

```
Priority: Could
Estimate: 8 SP
Epic: E6
State: awaiting decision 1
Depends on: —
Full DoR and acceptance criteria: https://github.com/Concord-API/API-5/blob/main/Docs/Scrum/acceptance-criteria.md#us-30--know-how-long-it-takes-to-reach-a-decision
```

| # | Task | Layer | Est. |
|---|---|---|---:|
| 30.1 | Create the time-to-decision aggregate from the filing date and the result date | Backend | 5h |
| 30.2 | Exclude the cases without a decision and state it | Backend | 2h |
| 30.3 | Exclude the results dated before the filing and report how many | Backend | 3h |
| 30.4 | Break the time-to-decision down by court | Backend | 4h |
| 30.5 | Show the elapsed time with its n and the "observed, not a forecast" caveat | Frontend | 5h |

**Descrições**

- **30.1**
  - `Data:` grão de movimentação, decisão 1 já tomada ([D-13](../../06-operacao/02-decisoes-e-riscos.md#d-13--grão-do-fato-movimentação-processual-opção-a)); mediana e quartis em dias
  - `Verifies: índice único no agregado`
- **30.3** — `Data: há processos com resultado anterior ao ajuizamento na fonte: são excluídos e contados, nunca corrigidos`

*5 tasks · 19h*

## US-34 — Como **usuário**, quero perguntar a um chatbot sobre um tema em linguagem natural e receber a resposta em prosa com os números, para as perguntas que nenhum filtro de tela responde

```
Priority: Could
Estimate: 13 SP
Epic: E7
State: awaiting decision 18
Depends on: US-01, US-09
Full DoR and acceptance criteria: https://github.com/Concord-API/API-5/blob/main/Docs/Scrum/acceptance-criteria.md#us-34--ask-in-natural-language
```

| # | Task | Layer | Est. |
|---|---|---|---:|
| 34.1 | Register the decision on where the chatbot appears | Frontend | 2h |
| 34.2 | Spike: how the chatbot reaches a language model from the client's intranet | Backend | 6h |
| 34.3 | Register the decision on the model and its hosting | Backend | 2h |
| 34.4 | Expose the screen queries as validated tools with no free SQL | Backend | 6h |
| 34.5 | Validate the parameters and cap the rows of every tool | Backend | 3h |
| 34.6 | Create `POST /api/chat` running the tool loop | Backend | 6h |
| 34.7 | Instruct the model to describe the data and never give advice | Backend | 3h |
| 34.8 | Log every tool call with its parameters and result | Backend | 3h |
| 34.9 | Build the chat panel where the decision places it | Frontend | 6h |
| 34.10 | Test that the same question returns identical numbers on the screen and in the chat | Test | 5h |

**Descrições**

- **34.1** — `Data: decisão 18`
- **34.2**
  - `Data: o NFR-12 diz que a aplicação funciona sem internet em tempo de execução, e o desenho do chatbot deixa em aberto qual modelo e qual provedor: custo, latência e se o dado sai da infraestrutura`
  - `Data: as saídas a comparar: modelo hospedado no servidor do cliente; chamada externa por um host liberado; ou tirar o chatbot do pacote`
- **34.3**
  - `Depends: 34.2. As tasks abaixo, da 34.6 em diante, só são puxáveis depois dela`
  - `Data:` a task é entregável: vira entrada em [Decisões e riscos](../../06-operacao/02-decisoes-e-riscos.md)
- **34.4**
  - `Data:` `searchThemes`, `getThemeSummary`, `getThemeByYear`, `getThemeByCourt`, `getThemeByJudgingBody`, `listCases`; as mesmas consultas das telas
  - `Verifies: testadas isoladamente, sem modelo no caminho`
- **34.7**
  - `Verifies: "o pedido foi acolhido em 82% dos casos" é dado; "entre com essa ação" não é o produto`
  - `Data: pergunta de matéria criminal recebe a declaração de escopo, não uma resposta`

*10 tasks · 42h*

## US-35 — Como **usuário**, quero que toda resposta do chatbot traga o `n`, a fonte, a data de extração, o escopo e o link para os processos, para poder conferir antes de usar

```
Priority: Could
Estimate: 5 SP
Epic: E7
State: ready
Depends on: US-34
Full DoR and acceptance criteria: https://github.com/Concord-API/API-5/blob/main/Docs/Scrum/acceptance-criteria.md#us-35--be-able-to-check-what-the-chatbot-answered
```

| # | Task | Layer | Est. |
|---|---|---|---:|
| 35.1 | Attach the n, the source, the extraction date and the scope to every number in an answer | Backend | 5h |
| 35.2 | Attach the path to the cases behind each statement | Backend | 3h |
| 35.3 | Answer a question about "the Brazilian courts" for the courts in scope, saying how many | Backend | 3h |
| 35.4 | Build the evaluation suite of known-answer questions | Test | 6h |
| 35.5 | Run the evaluation suite in CI as a precondition for exposing the chatbot | DevOps | 3h |
| 35.6 | Render the provenance and the links inside the chat answer | Frontend | 4h |

**Descrições**

- **35.3** — `Verifies: nunca sugere cobertura nacional`
- **35.4** — `Data: compara cada resposta com a consulta direta`

*6 tasks · 24h*

## US-36 — Como **usuário**, quero que o chatbot diga que não sabe quando o dado não está na base, para não receber um número plausível e inventado

```
Priority: Could
Estimate: 3 SP
Epic: E7
State: ready
Depends on: US-34
Full DoR and acceptance criteria: https://github.com/Concord-API/API-5/blob/main/Docs/Scrum/acceptance-criteria.md#us-36--get-i-dont-know-instead-of-an-invented-number
```

| # | Task | Layer | Est. |
|---|---|---|---:|
| 36.1 | Answer that the data is not in the base when a tool returns nothing | Backend | 3h |
| 36.2 | Refuse to invent case law for a theme that does not exist | Backend | 3h |
| 36.3 | Say that a block is outside the data scope and why | Backend | 2h |
| 36.4 | Fail the evaluation suite when a number has no matching tool result | Test | 5h |
| 36.5 | Test the refusal cases: nonexistent theme, unsourced block and criminal question | Test | 4h |

*5 tasks · 17h*

