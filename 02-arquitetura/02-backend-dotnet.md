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

A `main` ainda está em `initial commit`. O setup de verdade está na branch
**`initial-setup`** (PR aberto): .NET 10, as camadas, Serilog, Dapper + Npgsql, os dois
health checks, `ProblemDetails` em português e os primeiros testes.

Feito (na `initial-setup`):

- [x] **.NET 10** ([R-14](../06-operacao/02-decisoes-e-riscos.md#r-14--net-8-sai-de-suporte-durante-o-projeto-)).
- [x] Scaffold fora: `WeatherForecast`, `Class1.cs`, `UnitTest1.cs`.
- [x] `Ratio.Api.Tests` com `WebApplicationFactory`; `Ratio.Infrastructure.Tests` com Testcontainers.
- [x] Connection string por variável de ambiente (`ConnectionStrings__Ratio`) — **a API
      não sobe sem ela**, em vez de subir e quebrar na primeira consulta.
- [x] CORS com **lista explícita** (`Cors:AllowedOrigins`), nunca `*`.
- [x] `ProblemDetails` com `title`/`detail` em português — inclusive o 400 de validação.
- [x] `/health` e `/health/ready` com os [três estados](#health-check--os-três-estados).
- [x] Log estruturado em arquivo (Serilog), `UseWindowsService()` e `UseForwardedHeaders`.
- [x] Erro de configuração na subida vai **para o arquivo de log** — no serviço Windows
      não há console, e sem isso a TI do cliente ficaria sem diagnóstico.

Falta:

- [ ] **Primeira rota de domínio** (`GET /api/topics`) — hoje só o health existe, e ela
      **nasce de um teste** ([TDD](../07-justificativas/03-tdd.md)).
- [ ] Workflow de CI — **não existe `.github/` no repo**: build + test + pacote de versão.
- [ ] Publicação **self-contained `win-x64`** rodando como **serviço Windows**, escutando
      em `127.0.0.1` atrás do NGINX — ver
      [Implantação no cliente](../06-operacao/04-implantacao-no-cliente.md#backend).

## Pacotes

Versões ficam **centralizadas** em `Ratio/Directory.Packages.props`; nenhum `.csproj`
fixa versão.

| Pacote | Onde | Para quê | Versão |
|---|---|---|---|
| `Swashbuckle.AspNetCore` | Api | Swagger — exposto **só em Development** | 6.6.2 |
| `Npgsql` | Infrastructure | driver Postgres | 9.0.3 |
| `Dapper` | Infrastructure | acesso a dados (ver abaixo) | 2.1.35 |
| `Serilog.AspNetCore` + `Serilog.Sinks.File` | Api | log estruturado, em arquivo com rotação (no cliente não há console) | 9.0.0 / 6.0.0 |
| `Microsoft.Extensions.Hosting.WindowsServices` | Api | rodar como serviço Windows | 10.0.0 |
| `xunit` + `xunit.runner.visualstudio` | testes | framework e `Assert` | 2.9.3 / 2.8.2 |
| `Moq` | testes | dublê; asserção é o `Assert` do xUnit — ver [TDD](../07-justificativas/03-tdd.md#backend--net) | 4.20.72 |
| `Microsoft.AspNetCore.Mvc.Testing` | `Ratio.Api.Tests` | API em memória | 10.0.0 |
| `Testcontainers.PostgreSql` | `Ratio.Infrastructure.Tests` | Postgres real (`postgres:16`, como produção — sem pgvector) no teste | 4.7.0 |
| `Microsoft.Extensions.DependencyInjection` | `Ratio.Infrastructure.Tests` | montar um container de verdade no teste de registro | 10.0.0 |
| `Microsoft.NET.Test.Sdk` + `coverlet.collector` | testes | runner e cobertura | 17.12.0 / 6.0.4 |

**Fora da lista de propósito:** pacote de health check
([por quê](#health-check--os-três-estados)).

**`SSH.NET` não se mexe.** Ninguém o referencia: é **pin transitivo**. O Testcontainers
traz a versão 2024.2.0, que tem duas vulnerabilidades altas conhecidas, e o pin sobe
para 2026.0.0 — tirar a linha faz o `restore` falhar com `NU1903`. Só sai quando o
Testcontainers atualizar a dependência.

### Acesso a dados — Dapper

**Decidido: Dapper, não EF Core** ([D-26](../06-operacao/02-decisoes-e-riscos.md#d-26--acesso-a-dados-com-dapper)). A carga de trabalho é **100% leitura** de
agregados com SQL analítico que precisamos controlar. Um ORM adiciona uma camada de
tradução entre você e a consulta. EF Core faria sentido com escrita transacional rica —
não há: a API não escreve, e a carga é feita por fora
([D-17](../06-operacao/02-decisoes-e-riscos.md#d-17--carga-manual-não-agendada)).

**O schema não é da API.** Tabelas e views do DW são criadas pelas migrations SQL do
pipeline de carga; a API só as lê. Não use EF Migrations nem crie tabela a partir do
backend. O usuário de banco da API deve ter **só `SELECT`** no schema `dw`.

Raspagem, ETL, NLP, migrations e testes de integridade da carga ficam fora deste
repositório e do CI da API. Os testes de integração do backend podem criar schema e
dados mínimos apenas no PostgreSQL descartável de teste, para validar suas consultas.
Isso não executa o pipeline nem altera os bancos de carga, homologação ou produção.

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
| `GET /health` | o processo está de pé | monitoramento |
| `GET /health/ready` | há dado utilizável — e, se não houver, **por quê** ([três estados](#health-check--os-três-estados)) | monitoramento, verificação pós-instalação |
| `GET /api/topics?q=&court=&period=&level=&minStrength=&limit=` | lista de temas | tela de resultados + filtros |
| `GET /api/topics/{code}` | painel do tema: resumo, série anual, por tribunal, por órgão | detalhamento |
| `GET /api/topics/{code}/decisions?court=&outcome=&limit=` | processos que sustentam o tema, com link para a origem | amostra auditável |
| `GET /api/topics/{code}/export?format=csv` | exportação | botão `EXPORTAR CSV` |
| `POST /api/chat` | pergunta em linguagem natural | [chatbot](../01-produto/05-chatbot.md) |

`POST /api/chat` é a única rota de escrita-aparente, e mesmo ela não grava dado de
domínio.

### Health check — os três estados

`GET /health` é *liveness*: responde `200` com o texto `Saudável` enquanto o processo
estiver de pé. **Não toca no banco** — serve só para saber se o serviço Windows caiu.

`GET /health/ready` é *readiness*, e responde a pergunta que importa na instalação:
**há dado utilizável?** Ele distingue **três** situações, porque a ação de quem opera o
servidor é diferente em cada uma:

| Situação | HTTP | `reason` | O que quem opera faz |
|---|---|---|---|
| DW com carga publicada | `200` | — | nada |
| Banco respondeu, mas o `dw` está vazio ou nem existe | `503` | `sem carga publicada` | restaurar o dump da última carga |
| Banco não respondeu — parado, credencial errada, permissão faltando | `503` | `banco inacessível` | consertar o banco; a causa está no log em arquivo |

Corpo do `200`:

```json
{ "status": "Pronto", "lastExtractionAt": "2026-09-15T12:00:00+00:00" }
```

Corpo dos `503` — `application/problem+json`, com `title` e `detail` em português
([Idioma](#idioma)) e o membro de extensão **`reason`**:

```json
{
  "type": "https://tools.ietf.org/html/rfc9110#section-15.6.4",
  "title": "Não está pronto",
  "status": 503,
  "detail": "O banco respondeu, mas não há carga publicada no schema dw. Restaure o dump da última carga.",
  "traceId": "00-db47d42c4a3c205bac87639059c36632-a92174375b146617-00",
  "reason": "sem carga publicada"
}
```

**`reason` é o campo do contrato; `detail` é prosa.** Monitoramento, script de
instalação e a TI do cliente decidem olhando o `reason` — valor curto e estável — sem
casar frase. O `detail` pode ser reescrito a qualquer momento; o `reason` só muda com
aviso. Hoje existem dois: `sem carga publicada` e `banco inacessível`.

**A consulta é indexada.** O readiness é `MAX(extracted_at)` na tabela de fatos, e o
monitoramento bate nele de minuto em minuto: o índice `idx_fact_case_event_extracted_at`
([DW](04-data-warehouse.md#schemas-e-objetos)) troca um seq scan de 535 ms por 0,26 ms.

**`traceId` liga a resposta ao log.** É o mesmo identificador que aparece na linha do
Serilog que registrou a falha — é ele que a TI do cliente manda para nós quando o
`/health/ready` acusar `banco inacessível`.

**Um `503` de banco vazio é esperado numa instalação nova.** O servidor sobe antes de o
dump ser restaurado; o `reason` é o que diz à TI do cliente que falta a carga, e não que
a instalação está quebrada.

**Por que não um pacote de health check.** `AspNetCore.HealthChecks.NpgSql` responde
"o banco aceitou conexão" — o que não separa banco **vazio** de banco **fora do ar**, nem
devolve a data da última carga. Como é exatamente essa distinção que a
[implantação no cliente](../06-operacao/04-implantacao-no-cliente.md) precisa, o endpoint
é escrito à mão sobre a porta `ILastExtractionReader`, e o pacote saiu da lista.

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
