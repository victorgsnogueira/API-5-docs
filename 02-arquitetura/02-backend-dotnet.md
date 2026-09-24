# Backend .NET

Repositório `API5-Backend`, solução `Ratio/Ratio.slnx`, **ASP.NET Core** sobre
**.NET 10** (LTS), `Nullable` e `ImplicitUsings` habilitados em todos os projetos.

**O backend é a API de leitura e o dono do schema `dw`.** Ele cria e evolui o schema por
migration na subida (DbUp, `Ratio.Infrastructure/Migrations/Scripts/V00N__*.sql`), mas
não escreve dado: o dado vem do arquivo de carga do `API5-Pipeline` — ver
[D-17](../06-operacao/02-decisoes-e-riscos.md#d-17--carga-manual-não-agendada) e
[D-36](../06-operacao/02-decisoes-e-riscos.md#d-36--uma-única-credencial-para-a-api-sem-separar-migração-e-leitura).

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
| Identificador C#, tabela, coluna | inglês | `ThemeSummary`, `dw.fact_case_event` |
| Rota | inglês | `GET /api/themes/{key}/cases` |
| **Chave** do JSON | inglês (camelCase) | `"strengthScore"`, `"polarityLabel"` |
| **Valor** de dado | português | `"name": "Inscrição indevida em cadastro de inadimplentes"` |
| Rótulo / enum exposto ao usuário | português | `"level": "Divergente"` |
| Motivo de dado ausente | português | `"reason": "O DataJud não publica o relator."` |
| **Erro** (`ProblemDetails.title` / `detail`) | português | `"title": "Tema não encontrado"` |
| Mensagem de validação | português | `"O parâmetro 'limit' deve estar entre 1 e 100."` |
| Log | inglês | `"Theme {Key} not found"` — log é para o time, não para o usuário |
| Swagger — descrição das rotas | português | é documentação de quem consome a API |

**Por que as chaves ficam em inglês:** elas são código dos dois lados — viram
propriedade C# e tipo TypeScript. Traduzi-las obrigaria a manter dois vocabulários
para a mesma coisa. Já tudo o que o frontend **exibe** chega pronto em português, e o
frontend não traduz nada — é o que garante que a tela e a API digam a mesma frase
(ver a regra do [rótulo de polaridade](../03-dados/05-polaridade-do-resultado.md)).

```csharp
// certo — identificador em inglês, retorno em português
public sealed record ThemeSummary(long Key, string Name, int Cases, string StrengthLevel);
// Name = "Atraso de voo"   StrengthLevel = "Em formação"

return Problem(title: "Tema não encontrado", statusCode: 404);

// errado
public sealed record ResumoTema(int Codigo, string Nome, int Processos);
return Problem(title: "Theme not found");     // o usuário lê isso
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
| tema | `theme` | a entidade central do produto; chave pública `theme_key` |
| assunto (TPU) | `subject` | o código do CNJ que origina o tema; no DW a tabela é `dim_subject` ([D-35](../06-operacao/02-decisoes-e-riscos.md#d-35--o-banco-do-cliente-é-o-dw-um-só-modelo-dw-nos-três-bancos)) |
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
| `Ratio.Domain` | `Theme`, `Case`, `CaseEvent`, `DecisionOutcome`, o cálculo do `StrengthScore` | SQL, HTTP, atributos de framework |
| `Ratio.Application` | casos de uso (`SearchThemes`, `GetThemeDetail`, `ListCases`), DTOs, interfaces de repositório | Npgsql, `HttpClient` |
| `Ratio.Infrastructure` | repositórios de leitura sobre Postgres; as migrations do `dw` (`Migrations/`) | regra de negócio, cliente de fonte externa |
| `Ratio.Api` | controllers, DI, CORS, Swagger, health checks, `Hosting/` (migração na subida) | consulta SQL |

### Multifonte na estrutura

O produto consome [várias fontes](../03-dados/01-fontes.md), mas **a API não fala com
nenhuma delas**: fontes são problema da [carga](05-etl-e-nlp.md). O multifonte chega à
API como **proveniência** — toda resposta diz de que fonte e de quando é o dado, lida das
colunas `source` / `extracted_at` que toda linha do DW carrega.

---

## Estado atual e primeiras tarefas

Estado em 22/09/2026, na `main`:

- [x] **.NET 10** ([R-14](../06-operacao/02-decisoes-e-riscos.md#r-14--net-8-sai-de-suporte-durante-o-projeto-)).
- [x] Scaffold fora: `WeatherForecast`, `Class1.cs`, `UnitTest1.cs`.
- [x] `Ratio.Api.Tests` com `WebApplicationFactory`; `Ratio.Infrastructure.Tests` com Testcontainers.
- [x] Connection string por variável de ambiente (`ConnectionStrings__Ratio`) — **a API
      não sobe sem ela**, em vez de subir e quebrar na primeira consulta.
- [x] CORS com **lista explícita** (`Cors:AllowedOrigins`), nunca `*`.
- [x] `ProblemDetails` com `title`/`detail` em português — inclusive o 400 de validação.
- [x] `/health` e `/health/ready` com os [três estados](#health-check--os-três-estados).
- [x] Log estruturado em arquivo (Serilog), `UseWindowsService()` e `UseForwardedHeaders`.
- [x] **CI e release** (`Backend CI`, `Release label`, `Backend Release`) — ver
      [DevOps](../06-operacao/03-devops-e-infra.md#pipeline-de-cicd--esqueleto) e
      [Versionamento e releases](../07-justificativas/04-versionamento-e-releases.md).
- [x] Erro de configuração na subida vai **para o arquivo de log** — no serviço Windows
      não há console, e sem isso a TI do cliente ficaria sem diagnóstico.
- [x] **Schema `dw` criado pela API na subida** (0.12): `V001` com dimensões, fato,
      pontes, `strength_config`, `case_current_result` e `theme_summary`; lock de
      concorrência; journal em `migrations.schema_versions`.
- [x] Na `us1` (US-01 em andamento): `V002`, configuração de busca `dw.pt_unaccent` (1.1).
- [x] Na `us1`: `V006`, sinônimos de busca `dw.search_synonym` e `dw.expand_query`, com a
      `dw.search_themes` buscando pela consulta expandida (1.4).

Falta:

- [ ] **Primeira rota de domínio** (`GET /api/themes`, task 1.5) — contrato em
      [`GET /api/themes` — contrato da Sprint 1](#get-apithemes--contrato-da-sprint-1).
- [ ] **Pacote de versão completo** — o CI e a release do backend já existem (`.github/workflows/`), mas a release publica só o zip da API; falta juntar com NGINX, dump e manual.
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
| `GET /api/themes?q=&court=&period=&level=&minStrength=&limit=` | lista de temas | tela de resultados + filtros |
| `GET /api/themes/{key}` | painel do tema: resumo, série anual, por tribunal, por órgão | detalhamento |
| `GET /api/themes/{key}/cases?court=&outcome=&limit=` | processos que sustentam o tema, com link para a origem | amostra auditável |
| `GET /api/themes/{key}/export?format=csv` | exportação | botão `EXPORTAR CSV` |
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

### `GET /api/themes` — contrato da Sprint 1

Contrato fechado da lista de temas, para o frontend (0.14, 1.10 a 1.13) desenvolver com
MSW em paralelo à implementação (1.5 a 1.8, 2.3 a 2.5). O frontend valida a resposta com
`zod`; **mudar um campo exige avisar as duas pontas**. Os filtros (`court`, `period`,
`level`, `minStrength`) entram na Sprint 2 (US-04) e não fazem parte deste contrato.

**Requisição**

| Parâmetro | Tipo | Regra |
|---|---|---|
| `q` | texto, opcional | vazio ou ausente → os temas de maior volume (1.7); de 1 a 2 caracteres → `400` (1.6) |
| `limit` | inteiro, opcional | padrão `20`, máximo `100` |

**`200` — lista**

```json
{
  "query": "inscricao indevida",
  "total": 1,
  "themes": [
    {
      "themeKey": 412,
      "name": "Inscrição indevida em cadastro de inadimplentes",
      "subjectArea": "CONSUMIDOR",
      "judgedCount": 144,
      "strengthScore": 78,
      "level": "Dominante",
      "outcome": {
        "upheld": 142,
        "rejected": 2,
        "upheldRatio": 0.9861,
        "polarityLabel": "acolhimento da pretensão do autor"
      },
      "lastDecisionDate": "2026-08-30"
    }
  ]
}
```

| Campo | Regra |
|---|---|
| `themeKey` | chave pública e estável (`dw.dim_theme.theme_key`), **nunca** o `theme_sk` (D-34) |
| `name` | rótulo curado do tema, em português |
| `subjectArea` | tag da área (`CONSUMIDOR`, `BANCARIO`…) ou `null` quando não classificada |
| `judgedCount` | processos com resultado apurado; tema com `0` não aparece (1.8) |
| `strengthScore`, `level` | nota de força e o grau: `Consolidada`, `Dominante`, `Em formação` ou `Divergente` ([Força](../01-produto/04-forca-do-entendimento.md)); a lista vem ordenada por `strengthScore` decrescente, empate por `judgedCount` (2.3, 2.4) |
| `outcome.upheldRatio` | **`null` abaixo do piso de `n`** — a tela mostra só as contagens (D-32) |
| `outcome.polarityLabel` | o texto que diz **a quem** o percentual se refere; nunca exibir percentual sem ele |
| `lastDecisionDate` | `AAAA-MM-DD` ou `null` |

Nenhum número de processo aparece nesta resposta (1.5).

**Nenhum resultado** — `200` com `"total": 0` e `"themes": []`. A mensagem de escopo
("Nenhum tema encontrado para «termo» no escopo TJSP, TJRJ e TJMG.") é montada pelo
frontend (1.13).

**`400` — termo curto demais** — `application/problem+json`:

```json
{
  "type": "https://tools.ietf.org/html/rfc9110#section-15.5.1",
  "title": "Requisição inválida",
  "status": 400,
  "detail": "Digite ao menos 3 caracteres para buscar."
}
```

### `GET /api/themes/{key}` — contrato da Sprint 1

Painel do tema: cabeçalho, Resumo (US-09) e distribuição de desfechos (US-10). `{key}` é o
`theme_key` (D-34). O frontend valida com `zod` e desenvolve contra este JSON com MSW
(9.8 a 9.11, 10.5 a 10.7); **mudar um campo exige avisar as duas pontas**. O campo
`scope` entra pela US-24 e não faz parte deste recorte; o `unavailable` segue a
[seção própria](#unavailable--o-que-não-existe-e-por-quê-us-26) e o `provenance`, a
[dele](#provenance--fonte-e-data-de-extração-us-25).

**`200` — tema**

```json
{
  "themeKey": 412,
  "name": "Inscrição indevida em cadastro de inadimplentes",
  "subjectArea": "CONSUMIDOR",
  "strengthScore": 78,
  "level": "Dominante",
  "caseCount": 203,
  "judgedCount": 144,
  "courtCount": 3,
  "periodStartYear": 2021,
  "periodEndYear": 2026,
  "lastDecisionDate": "2026-08-30",
  "summary": {
    "lead": [
      { "text": "Em " },
      { "ratio": 0.9861, "n": 144, "unit": "decisões" },
      { "text": " julgadas, houve acolhimento da pretensão do autor." }
    ],
    "body": [
      { "text": "As decisões vêm de 3 tribunais, entre 2021 e 2026." }
    ],
    "textOrigin": "template",
    "methodologyVersion": "1.0",
    "generatedAt": "2026-09-23"
  },
  "outcomeBreakdown": [
    {
      "polarityLabel": "acolhimento da pretensão do autor",
      "judged": 144,
      "categories": [
        { "outcome": "Procedente", "count": 100, "ratio": 0.6944 },
        { "outcome": "Parcialmente procedente", "count": 42, "ratio": 0.2917 },
        { "outcome": "Improcedente", "count": 2, "ratio": 0.0139 }
      ]
    }
  ],
  "partialTreatment": "Na nota de força, a procedência em parte conta como acolhimento.",
  "unavailable": [
    {
      "block": "caseLawCitation",
      "reason": "sourceUnavailable",
      "message": "A citação de acórdão depende do inteiro teor da decisão, e os tribunais do escopo bloqueiam a coleta desse texto."
    }
  ]
}
```

| Campo | Regra |
|---|---|
| `caseCount` | processos do tema (`theme_summary.case_count`), julgados ou não |
| `judgedCount` | o `n` da nota (`theme_strength.judged`), o mesmo da lista |
| `courtCount`, `periodStartYear`, `periodEndYear` | `theme_summary`; a tela monta a linha de metadados, sem conta |
| `summary` | texto de `dw.theme_narrative` ([D-38](../06-operacao/02-decisoes-e-riscos.md#d-38--origem-do-texto-do-tema-template-na-carga-curado-por-cima)); `null` quando o tema não tem texto |
| `summary.lead`, `summary.body` | lista de **segmentos**, na ordem de leitura |
| `textOrigin` | `template` ou `curated` |
| `outcomeBreakdown` | uma entrada por família com julgados; mérito e recurso **nunca somados** |
| `categories[].ratio` | `count / judged` com 4 casas; **`null` abaixo do piso de `n`** (D-32) |
| `partialTreatment` | a declaração da [D-39](../06-operacao/02-decisoes-e-riscos.md#d-39--procedência-em-parte-soma-na-nota-separada-na-figura), mostrada junto da fonte da figura |

**Segmentos do texto**

| Segmento | Campos | A tela faz |
|---|---|---|
| texto | `text` | mostra como está |
| proporção | `ratio`, `n`, `unit` | formata `98,6% das 144 decisões`; **`n` é obrigatório**, e sem ele o segmento não é renderizado |
| contagem | `count`, `unit` | mostra `1 decisão`, sem `%`; é o que o template emite abaixo do piso |

Nenhum número do texto é calculado na tela nem produzido por modelo: o `ratio` e o `n` vêm
do `SELECT` da carga (9.4).

**Template (9.4 e 9.5)**

- **Lead:** abre com a proporção de acolhimento na família da nota e o `n` dela, seguida do
  `polarityLabel`: *"Em {ratio} das {n} decisões julgadas, houve {polarityLabel}."* Abaixo
  do piso: *"Há {count} decisão julgada, com {polarityLabel}."*
- **Corpo:** tribunais (`court_count`), período (`period_start_year`–`period_end_year`),
  última decisão e, quando há as duas famílias, mérito e recurso em frases separadas, cada
  uma com o seu `n`. Só slots que existem nos agregados; nada sobre blocos sem fonte.

**`404` — tema não encontrado** — `application/problem+json`, `detail`
`"Tema não encontrado."` (9.7). Uma chave que não é número também responde `404`, só com o
título.

### `unavailable` — o que não existe e por quê (US-26)

Todo bloco da tela que não tem dado vem na resposta como um item de `unavailable`, com o
**motivo** e o **texto em português** que a tela mostra. O frontend nunca escreve o motivo:
ele exibe o que veio (26.4). Um bloco que está em `unavailable` não aparece com valor, nem de
exemplo ([R-01](../06-operacao/02-decisoes-e-riscos.md#r-01--blocos-do-mockup-sem-fonte--era---reduzido-não-eliminado)).

```json
"unavailable": [
  {
    "block": "caseLawCitation",
    "reason": "sourceUnavailable",
    "message": "A citação de acórdão depende do inteiro teor da decisão, e os tribunais do escopo bloqueiam a coleta desse texto."
  }
]
```

| Campo | Regra |
|---|---|
| `block` | chave estável do bloco, em inglês (tabela abaixo) |
| `reason` | um dos **três** motivos; nunca um motivo genérico |
| `message` | o texto que a tela mostra, específico do bloco e do motivo |

**Os três motivos (26.1)**

| `reason` | Quando |
|---|---|
| `sourceUnavailable` | a fonte não fornece o dado (inteiro teor bloqueado, campo que o DataJud não publica) |
| `notLoaded` | o dado existe na fonte, mas ainda não foi carregado (a carga não rodou ou não chegou a este tema) |
| `notApplicable` | o bloco não se aplica a este tema (por exemplo, texto do entendimento para um tema sem julgados) |

**Blocos da Sprint 1 (26.2)**

| `block` | Onde | Motivo | `message` |
|---|---|---|---|
| `caseLawCitation` | citação de acórdão e botão `Inteiro teor · PDF` (9.11) | `sourceUnavailable` | A citação de acórdão depende do inteiro teor da decisão, e os tribunais do escopo bloqueiam a coleta desse texto. |
| `citedDecisions` | marcadores `[1]` e rodapé "Decisões citadas" (9.11) | `sourceUnavailable` | As decisões citadas dependem do inteiro teor, que os tribunais do escopo não liberam para coleta. |
| `amountAwarded` | FIG. 2, valor fixado (10.7) | `sourceUnavailable` | O valor fixado não é campo estruturado no DataJud; ele só existe no inteiro teor da decisão. |
| `reporterJudge` | relator, na Base analítica | `sourceUnavailable` | O DataJud não publica o relator. |
| `summary` | texto do Resumo, quando `summary` é `null` | `notLoaded` ou `notApplicable` | `notLoaded`: O texto deste tema ainda não foi gerado; ele sai na próxima carga. `notApplicable`: Este tema não tem decisões julgadas, então não há entendimento para descrever. |

Os textos ficam no Backend, num só lugar, e a tela os recebe prontos. Mudar um texto não
exige mudar o frontend.

**Link para a explicação (26.6).** Cada aviso leva a `/limitacoes#<âncora>` no frontend:
a âncora é o `block` quando o motivo é `sourceUnavailable` e o próprio `reason` nos outros
dois casos. A página tem uma seção por âncora, mais escopo e atualização. A mensagem
continua sozinha na nota; o link fica fora dela.

### `provenance` — fonte e data de extração (US-25)

Vem em `GET /api/themes` (proveniência global, da `dw.data_provenance`) e em
`GET /api/themes/{key}` (proveniência do tema, da `dw.theme_provenance`). O frontend exibe
no rodapé de cada tela todas as fontes listadas, nunca só a principal.

```json
"provenance": {
  "sources": [
    {
      "block": "cases",
      "source": "datajud",
      "name": "DataJud/CNJ",
      "sourceUrl": "https://www.cnj.jus.br/sistemas/datajud/",
      "extractedAt": "2026-08-28T13:00:00+00:00",
      "count": 12418
    }
  ],
  "methodologyVersion": "1.0"
}
```

| Campo | Regra |
|---|---|
| `block` | `cases` (processos, do fato) ou `doctrine` (artigos, da `dim_doctrine`) |
| `source` | código gravado no DW (`datajud`, `doaj`, `scielo`) |
| `name`, `sourceUrl` | vêm do catálogo de fontes do Backend ([D-40](../06-operacao/02-decisoes-e-riscos.md#d-40--proveniência-nome-e-link-da-fonte-vêm-de-catálogo)); fonte desconhecida sai com o código como nome e `sourceUrl: null` |
| `extractedAt` | `max(extracted_at)` do dado carregado, nunca o relógio do servidor. O frontend mostra o dia em horário de Brasília |
| `count` | processos distintos (`cases`) ou artigos (`doctrine`) |
| `methodologyVersion` | da `strength_config`; `null` se a configuração não foi carregada |

**Bloco sem proveniência não é devolvido (25.4).** Fonte sem código, sem bloco ou sem data
sai da lista. Tema sem proveniência de `cases` volta com `summary: null` e o `unavailable`
de `summary` como `notLoaded`. Lista vazia de fontes faz o frontend esconder o rodapé.
View não populada (`55000`) vira lista vazia, com aviso no log.

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
