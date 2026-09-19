# Backend .NET

Repositório `API5-Backend`, solução `Ratio/Ratio.slnx`, **ASP.NET Core** sobre
**.NET 8**, `Nullable` e `ImplicitUsings` habilitados em todos os projetos.

> ⚠ **O .NET 8 sai de suporte em 10/11/2026** — antes do fim do projeto. O .NET 10 é
> a LTS vigente (suporte até nov/2028). Migrar é trocar `net8.0` por `net10.0` nos
> `.csproj` enquanto a solução ainda é scaffold; depois custa mais. Ver
> [R-14](../06-operacao/02-decisoes-e-riscos.md#r-14--net-8-sai-de-suporte-durante-o-projeto-).

**O backend é só a API de leitura.** A carga do DW é um
[processo manual](05-etl-e-nlp.md#carga-manual--o-processo), fora do backend — ver
[D-17](../06-operacao/02-decisoes-e-riscos.md#d-17--carga-manual-não-agendada).

## Por que .NET

Escolhido como alternativa ao Java: mais moderno, e a estrutura de solução com
múltiplos projetos torna a separação em camadas uma **restrição do compilador**, não
uma convenção que se erode. Se `Ratio.Domain` não referencia nada, é impossível vazar
acesso a banco para dentro dele.

---

## Idioma

> **Todo o código é escrito em inglês** — classes, métodos, variáveis, tabelas,
> colunas, rotas, chaves do JSON, mensagens de log, comentários, testes e commits.
>
> **Tudo o que a API devolve para ser lido é em português** — os dados, os rótulos,
> as mensagens de erro e de validação.

A linha divisória: **chave é código (inglês), valor é retorno (português).**

| O quê | Idioma | Exemplo |
|---|---|---|
| Identificador C#, tabela, coluna | inglês | `TopicSummary`, `dw.fact_case_event` |
| Rota | inglês | `GET /api/topics/{code}/decisions` |
| **Chave** do JSON | inglês (camelCase) | `"strengthScore"`, `"polarityLabel"` |
| **Valor** de dado | português | `"name": "Inscrição indevida em cadastro de inadimplentes"` |
| Rótulo / enum exposto ao usuário | português | `"level": "Divergente"` |
| Motivo de dado ausente | português | `"reason": "O DataJud não publica o relator."` |
| **Erro** (`ProblemDetails.title` / `detail`) | português | `"title": "Tema não encontrado"` |
| Mensagem de validação | português | `"O parâmetro 'limit' deve estar entre 1 e 100."` |
| Log | inglês | `"Topic {Code} not found"` — log é para o time, não para o usuário |
| Swagger — descrição das rotas | português | é documentação de quem consome a API |

**Por que as chaves ficam em inglês:** elas são código dos dois lados — viram
propriedade C# e tipo TypeScript. Traduzi-las obrigaria a manter dois vocabulários
para a mesma coisa. Já tudo o que o frontend **exibe** chega pronto em português, e o
frontend não traduz nada — é o que garante que a tela e a API digam a mesma frase
(ver a regra do [rótulo de polaridade](../03-dados/05-polaridade-do-resultado.md)).

```csharp
// certo — identificador em inglês, retorno em português
public sealed record TopicSummary(int Code, string Name, int Cases, string StrengthLevel);
// Name = "Atraso de voo"   StrengthLevel = "Em formação"

return Problem(title: "Tema não encontrado", statusCode: 404);

// errado
public sealed record ResumoTema(int Codigo, string Nome, int Processos);
return Problem(title: "Topic not found");     // o usuário lê isso
```

### Vocabulário do domínio — PT → EN

Traduzir jargão jurídico é onde o time vai divergir. Esta tabela fecha a discussão;
acrescente linhas conforme aparecerem, não invente sinônimos.

| Português | Inglês no código | Nota |
|---|---|---|
| processo | `case` | evitar `process` — colide com processo de sistema |
| número do processo (CNJ) | `caseNumber` | 20 dígitos |
| movimentação | `caseEvent` / `movement` | `movement` para o tipo (TPU), `caseEvent` para a ocorrência |
| decisão | `decision` | |
| julgamento | `judgment` | grafia sem "e" |
| acórdão | `appellateDecision` | decisão de colegiado |
| sentença | `ruling` | decisão de juiz singular |
| tribunal | `court` | |
| órgão julgador | `judgingBody` | vara, câmara, turma |
| vara | `trialCourtUnit` | |
| câmara / turma | `panel` | |
| grau / instância | `courtLevel` | `First`, `Second`, `Superior` |
| relator | `reporterJudge` | |
| tema | `topic` | a entidade central do produto |
| assunto (TPU) | `subject` | o código do CNJ que origina o tema |
| classe processual | `caseClass` | |
| procedência | `Granted` | enum `DecisionOutcome` |
| improcedência | `Denied` | |
| procedência em parte | `PartiallyGranted` | |
| extinção | `Dismissed` | sem julgamento de mérito |
| recurso | `appeal` | |
| jurisprudência | `caseLaw` | |
| precedente | `precedent` | |
| súmula | `precedentSummary` | manter `Súmula 385/STJ` como texto do dado |
| tema repetitivo | `bindingPrecedent` | |
| doutrina | `legalDoctrine` | |
| fundamento | `legalGround` | |
| força do entendimento | `strengthScore` | |
| concordância | `agreement` | componente da nota |
| cobertura | `coverage` | |
| recência | `recency` | |

**Regra para termos sem tradução fiel** (*acórdão*, *súmula*, *IRDR*): use a tradução
funcional acima no identificador e mantenha o termo original no **valor** e na
documentação da API. O usuário final nunca vê o identificador.

---

## Estrutura da solução

```
Ratio.Domain            (sem dependências)
   ▲
Ratio.Application  ──> Domain
   ▲          ▲
   │          └── Ratio.Application.Tests  (xUnit)
   │
Ratio.Infrastructure ──> Application, Domain
   ▲          ▲
   │          └── Ratio.Infrastructure.Tests  (xUnit)
   │
Ratio.Api  ──> Application, Infrastructure     (ASP.NET Core Web API)
   ▲
   └── Ratio.Api.Tests  (xUnit + WebApplicationFactory)   ← a criar
```

Dependências invertem para dentro: a Application define a **porta** (interface), a
Infrastructure fornece o **adaptador**. A Api é o host.

### Responsabilidade de cada projeto

| Projeto | Contém | Nunca contém |
|---|---|---|
| `Ratio.Domain` | `Topic`, `Case`, `CaseEvent`, `DecisionOutcome`, o cálculo do `StrengthScore` | SQL, HTTP, atributos de framework |
| `Ratio.Application` | casos de uso (`SearchTopics`, `GetTopicDetail`, `ListDecisions`), DTOs, interfaces de repositório | Npgsql, `HttpClient` |
| `Ratio.Infrastructure` | repositórios de leitura sobre Postgres | regra de negócio, DDL, cliente de fonte externa |
| `Ratio.Api` | controllers, DI, CORS, Swagger, health checks | consulta SQL |

### Multifonte na estrutura

O produto consome [várias fontes](../03-dados/01-fontes.md), mas **a API não fala com
nenhuma delas**: fontes são problema da [carga](05-etl-e-nlp.md). O multifonte chega à
API como **proveniência** — toda resposta diz de que fonte e de quando é o dado, lida das
colunas `source` / `extracted_at` que toda linha do DW carrega.

---

## Estado atual e primeiras tarefas

Todos os projetos existem; nenhum tem código de verdade (um commit, `initial commit`).

- [ ] **Subir para .NET 10** enquanto é scaffold ([R-14](../06-operacao/02-decisoes-e-riscos.md#r-14--net-8-sai-de-suporte-durante-o-projeto-)).
- [ ] **Remover o scaffold `WeatherForecast`** (`Ratio.Api/WeatherForecast.cs` e
      `Controllers/WeatherForecastController.cs`) antes que apareça no Swagger.
- [ ] Substituir os `Class1.cs` e `UnitTest1.cs` — **o primeiro código de cada camada
      nasce de um teste** ([TDD](../07-justificativas/03-tdd.md)).
- [ ] Criar `Ratio.Api.Tests` (testes HTTP com `WebApplicationFactory`).
- [ ] Connection string por variável de ambiente (`ConnectionStrings__Ratio`).
- [ ] CORS com **lista explícita** de origens, nunca `*`.
- [ ] `ProblemDetails` com `title`/`detail` em português para todo erro.
- [ ] Expor `/health` e `/health/ready` para o [monitoramento e a verificação pós-instalação](../06-operacao/03-devops-e-infra.md).
- [ ] Log estruturado (Serilog) — pré-requisito de observabilidade.
- [ ] Workflow de CI — **não existe `.github/` no repo**: build + test + pacote de versão.
- [ ] Publicação **self-contained `win-x64`** rodando como **serviço Windows**, escutando
      em `127.0.0.1` atrás do NGINX, com `UseForwardedHeaders` e log em arquivo — ver
      [Implantação no cliente](../06-operacao/04-implantacao-no-cliente.md#backend).

## Pacotes

Já referenciados:

| Pacote | Onde | Versão |
|---|---|---|
| `Swashbuckle.AspNetCore` | `Ratio.Api` | 6.6.2 |
| `xunit` + `xunit.runner.visualstudio` | testes | 2.5.3 |
| `Microsoft.NET.Test.Sdk` | testes | 17.8.0 |
| `coverlet.collector` | testes | 6.0.0 |

A adicionar:

| Pacote | Onde | Para quê |
|---|---|---|
| `Npgsql` | Infrastructure | driver Postgres |
| `Dapper` | Infrastructure | acesso a dados (ver abaixo) |
| `Serilog.AspNetCore` + `Serilog.Sinks.File` | Api | log estruturado, em arquivo com rotação (no cliente não há console) |
| `Microsoft.Extensions.Hosting.WindowsServices` | Api | rodar como serviço Windows |
| `AspNetCore.HealthChecks.NpgSql` | Api | `/health/ready` |
| `Moq` (≥ 4.20.70) | testes | dublê; asserção é o `Assert` do xUnit — ver [TDD](../07-justificativas/03-tdd.md#backend--net) |
| `Microsoft.AspNetCore.Mvc.Testing` | `Ratio.Api.Tests` | API em memória |
| `Testcontainers.PostgreSql` | `Ratio.Infrastructure.Tests` | Postgres real (`postgres:16`, como produção — sem pgvector) no teste |

### Acesso a dados — Dapper

**Decidido: Dapper, não EF Core** ([D-26](../06-operacao/02-decisoes-e-riscos.md#d-26--acesso-a-dados-com-dapper)). A carga de trabalho é **100% leitura** de
agregados com SQL analítico que precisamos controlar. Um ORM adiciona uma camada de
tradução entre você e a consulta. EF Core faria sentido com escrita transacional rica —
não há: a API não escreve, e a carga é feita por fora
([D-17](../06-operacao/02-decisoes-e-riscos.md#d-17--carga-manual-não-agendada)).

**O schema não é da API.** Tabelas e views do DW são criadas pelas migrations SQL do
pipeline de carga; a API só as lê. Não use EF Migrations nem crie tabela a partir do
backend. O usuário de banco da API deve ter **só `SELECT`** no schema `dw`.

---

## Contrato da API

O esqueleto abaixo foi derivado das telas. Chaves em inglês, valores em português
([Idioma](#idioma)).

> **Referência executável.** O [protótipo de dados de set/2026](../05-prototipo/01-prototipo-referencia.md#protótipo-de-dados--setembro2026)
> (`prototipo-prod - versao 202609/api/main.py`) implementa estas rotas contra o DW
> real — inclusive `polarityLabel`, `strengthScore.components`, `unavailable` e
> `provenance`. Serve para ver o **formato** e as **consultas SQL** que funcionam; o
> código em Python não é para portar linha a linha.

Somente `GET` na camada de consulta.

| Rota | Responde | Alimenta |
|---|---|---|
| `GET /health` | prontidão do processo | monitoramento, verificação pós-instalação |
| `GET /health/ready` | prontidão **real**: há dado utilizável? | monitoramento |
| `GET /api/topics?q=&court=&period=&level=&minStrength=&limit=` | lista de temas | tela de resultados + filtros |
| `GET /api/topics/{code}` | painel do tema: resumo, série anual, por tribunal, por órgão | detalhamento |
| `GET /api/topics/{code}/decisions?court=&outcome=&limit=` | processos que sustentam o tema, com link para a origem | amostra auditável |
| `GET /api/topics/{code}/export?format=csv` | exportação | botão `EXPORTAR CSV` |
| `POST /api/chat` | pergunta em linguagem natural | [chatbot](../01-produto/05-chatbot.md) |

`POST /api/chat` é a única rota de escrita-aparente, e mesmo ela não grava dado de
domínio.

### Campos que as telas exigem

Levantados percorrendo os mockups. Nenhum é opcional se o bloco correspondente entrar
na entrega:

| Campo | Onde aparece |
|---|---|
| `subjectArea` (tag CONSUMIDOR / BANCÁRIO / QUANTUM) | resultado e detalhamento |
| `summary` (as duas linhas de prosa) | resultado |
| `lastDecisionDate` | resultado e detalhamento |
| `strengthScore` **com os componentes abertos** | selo — ver [Força](../01-produto/04-forca-do-entendimento.md) |
| `outcome.polarityLabel` — o **texto** que acompanha o percentual | barra de alinhamento — ver [Polaridade](../03-dados/05-polaridade-do-resultado.md) |
| `unavailable` — para cada bloco sem fonte, o **motivo** em português | blocos vazios da Base analítica |
| contagem por tribunal | painel de filtros |
| `sourceLink` com o tipo do link | amostra auditável |
| `provenance` (fonte + data de extração) | rodapé de toda tela |

Esse último é novo e importante: como o produto é multifonte, **cada resposta deve
dizer de onde o dado veio e quando foi extraído**.

---

## Convenções

- **Inglês no código, português no retorno.** Ver [Idioma](#idioma).
- **TDD.** Nenhum código de produção sem um teste falhando que o peça. Ver
  [TDD](../07-justificativas/03-tdd.md).
- **Nada de dado inventado.** Onde a fonte não tem, devolva vazio. É preferível a
  interface mostrar "sem dado" a mostrar dado que a fonte não entrega.
- **Nenhuma métrica calculada no frontend.** Se um número aparece na tela, veio pronto
  daqui.
- **Toda resposta carrega proveniência.** Multifonte sem proveniência é dado que
  ninguém consegue auditar.
