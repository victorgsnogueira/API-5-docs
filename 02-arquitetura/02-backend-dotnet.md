# Backend .NET

Repositório `API5-Backend`, solução `Ratio/Ratio.slnx`, **.NET 8**, `Nullable` e
`ImplicitUsings` habilitados em todos os projetos.

## Por que .NET

Escolhido como alternativa ao Java: mais moderno, e a estrutura de solução com
múltiplos projetos torna a separação em camadas uma **restrição do compilador**, não
uma convenção que se erode. Se `Ratio.Domain` não referencia nada, é impossível vazar
acesso a banco para dentro dele.

---

## Idioma

> **Todo o backend é escrito em inglês** — classes, métodos, variáveis, tabelas,
> colunas, rotas, campos JSON, mensagens de log, comentários e commits.
>
> **Os dados permanecem em português**, porque o domínio é o direito brasileiro. O
> valor de um campo `topic.name` é `"Inscrição indevida em cadastro de inadimplentes"`,
> e continua assim.

A linha divisória é simples: **identificador é inglês, conteúdo é português.**

```csharp
// certo
public sealed record TopicSummary(int Code, string Name, int Cases, int Judged);
// name = "Atraso de voo"        ← dado, fica em português

// errado
public sealed record ResumoTema(int Codigo, string Nome, int Processos);
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
Ratio.Etl  ──> Application, Infrastructure     (console app, OutputType=Exe)
```

Dependências invertem para dentro: a Application define a **porta** (interface), a
Infrastructure fornece o **adaptador**. A Api e o Etl são apenas hosts — dois pontos de
entrada para o mesmo núcleo.

### Responsabilidade de cada projeto

| Projeto | Contém | Nunca contém |
|---|---|---|
| `Ratio.Domain` | `Topic`, `Case`, `CaseEvent`, `DecisionOutcome`, o cálculo do `StrengthScore` | SQL, HTTP, atributos de framework |
| `Ratio.Application` | casos de uso (`SearchTopics`, `GetTopicDetail`, `ListDecisions`), DTOs, interfaces de repositório | Npgsql, `HttpClient` |
| `Ratio.Infrastructure` | repositórios sobre Postgres, clientes das fontes externas, migrations | regra de negócio |
| `Ratio.Api` | controllers, DI, CORS, Swagger, health checks | consulta SQL |
| `Ratio.Etl` | orquestração da carga: ler config, iterar fontes, gravar, atualizar agregados | regra de transformação (essa vive na Application) |

### Multifonte na estrutura

O produto consome [várias fontes](../03-dados/01-fontes.md), e elas vão entrar em
tempos diferentes. Desenhe para isso desde o começo:

```csharp
// Ratio.Application — a porta
public interface ICaseSource
{
    string Name { get; }                                  // "datajud", "pangea", …
    IAsyncEnumerable<RawCase> FetchAsync(SourceQuery query, CancellationToken ct);
}
```

Cada fonte é uma implementação em `Ratio.Infrastructure`. O `Ratio.Etl` itera as fontes
registradas; acrescentar uma fonte é registrar uma classe, não reescrever o pipeline.

---

## Estado atual e primeiras tarefas

Todos os projetos existem; nenhum tem código de verdade.

- [ ] **Remover o scaffold `WeatherForecast`** (`Ratio.Api/WeatherForecast.cs` e
      `Controllers/WeatherForecastController.cs`) antes que apareça no Swagger.
- [ ] Substituir os `Class1.cs` de Domain, Application e Infrastructure.
- [ ] Definir connection string em `appsettings.json` + variável de ambiente (o
      `appsettings.json` atual só tem `Logging` e `AllowedHosts`).
- [ ] Configurar CORS com **lista explícita** de origens, nunca `*`.
- [ ] Escolher e configurar o acesso a dados (ver abaixo).
- [ ] Adicionar runner de migration (DbUp ou FluentMigrator).
- [ ] Expor `/health` e `/health/ready` para o [monitoramento e o Coolify](../06-operacao/03-devops-e-infra.md).
- [ ] Configurar log estruturado (Serilog) — pré-requisito de observabilidade.

## Pacotes já referenciados

| Pacote | Onde | Versão |
|---|---|---|
| `Swashbuckle.AspNetCore` | `Ratio.Api` | 6.6.2 |
| `xunit` + `xunit.runner.visualstudio` | testes | 2.5.3 |
| `Microsoft.NET.Test.Sdk` | testes | 17.8.0 |
| `coverlet.collector` | testes | 6.0.0 |

**Faltando:** driver Postgres (`Npgsql`), runner de migration, Serilog, e a decisão
entre Dapper e EF Core.

### Acesso a dados — recomendação

**Dapper (ou Npgsql direto), não EF Core.** A carga de trabalho é ~100% leitura de
agregados com SQL analítico que precisamos controlar. Um ORM adiciona uma camada de
tradução entre você e a consulta. EF Core faria sentido com escrita transacional rica —
não há: a única escrita é o ETL, em lote.

---

## Contrato da API

**A definir.** O contrato do protótipo **não** serve de base — ver
[Protótipo](../05-prototipo/01-prototipo-referencia.md). O que segue é o esqueleto
derivado das telas, em inglês, para o time preencher.

Somente `GET` na camada de consulta.

| Rota | Responde | Alimenta |
|---|---|---|
| `GET /health` | prontidão do processo | Coolify, monitoramento |
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
| contagem por tribunal | painel de filtros |
| `sourceLink` com o tipo do link | amostra auditável |
| `provenance` (fonte + data de extração) | rodapé de toda tela |

Esse último é novo e importante: como o produto é multifonte, **cada resposta deve
dizer de onde o dado veio e quando foi extraído**.

---

## Convenções

- **Inglês no código, português no dado.** Ver [Idioma](#idioma).
- **Nada de dado inventado.** Onde a fonte não tem, devolva vazio. É preferível a
  interface mostrar "sem dado" a mostrar dado que a fonte não entrega.
- **Nenhuma métrica calculada no frontend.** Se um número aparece na tela, veio pronto
  daqui.
- **Toda resposta carrega proveniência.** Multifonte sem proveniência é dado que
  ninguém consegue auditar.
