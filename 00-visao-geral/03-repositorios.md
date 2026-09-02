# Repositórios e responsabilidades

O projeto vive em **três repositórios git independentes**, mais um repositório de
protótipo antigo, que **não** é fonte de verdade.

```
D:/Desenvolvimento/fatec/API/API-5/     ← pasta de trabalho (não é repo)
├── API-5/          repo de documentação  ← esta wiki nasceu aqui em Docs/
├── API5-Backend/   repo do backend .NET 8
├── API5-Frontend/  repo do frontend React
└── prototipo/      protótipo antigo (Python/FastAPI) — ⚠ não é referência
```

## API-5 — documentação

| | |
|---|---|
| Conteúdo | documentação do projeto, artefatos de sprint, wiki |
| Responsável | **Scrum Master** do time |
| Estado | README vazio; a documentação técnica está em `Docs/` |

> **Nota sobre o local desta wiki.** Os arquivos foram escritos em
> `<raiz>/Docs/`, que hoje fica **fora** do repositório `API-5/`. Se a intenção é
> versioná-los junto da documentação do SM, mova a pasta `Docs/` para dentro de
> `API-5/`. Os links internos são todos relativos e continuam funcionando.

## API5-Backend — backend .NET 8

Solução `Ratio.slnx`, sete projetos em Clean Architecture:

| Projeto | Papel | Estado |
|---|---|---|
| `Ratio.Domain` | entidades e regras de negócio puras, sem dependência | `Class1.cs` vazio |
| `Ratio.Application` | casos de uso, portas (interfaces); depende só do Domain | `Class1.cs` vazio |
| `Ratio.Infrastructure` | acesso ao Postgres, clientes HTTP, implementações das portas | `Class1.cs` vazio |
| `Ratio.Api` | ASP.NET Core Web API + Swashbuckle (Swagger) | scaffold `WeatherForecast` |
| `Ratio.Etl` | console app da carga (executável agendável) | `Program.cs` scaffold |
| `Ratio.Application.Tests` | xUnit sobre Application e Domain | `UnitTest1.cs` |
| `Ratio.Infrastructure.Tests` | xUnit sobre Infrastructure | `UnitTest1.cs` |

Detalhe das camadas e do contrato da API: [Backend .NET](../02-arquitetura/02-backend-dotnet.md).

**Primeira tarefa real:** remover o scaffold `WeatherForecast` (controller + modelo)
antes que ele vaze para o Swagger de produção.

## API5-Frontend — frontend React

| | |
|---|---|
| Stack | React |
| Estado | **repositório vazio** — nada criado ainda |

Ponto de partida: [Frontend React](../02-arquitetura/03-frontend-react.md) e o
[Design system](../04-design/01-design-system.md).

## prototipo — protótipo antigo

> ⚠ **Não é fonte de verdade.** Foi construído antes das definições atuais de produto,
> escopo, fontes e arquitetura, e a modelagem de banco dele foi criada sem auditoria
> alguma. Serve como catálogo de armadilhas conhecidas das fontes — nada mais.

Ver [Protótipo — o que é e o que não é](../05-prototipo/01-prototipo-referencia.md).

## Convenções entre repositórios

**Idioma.** O **backend inteiro é escrito em inglês** — classes, métodos, tabelas,
colunas, rotas, campos JSON, logs e commits. Os **dados tratados permanecem em
português**, porque o domínio é o direito brasileiro: `topic.name` vale
`"Atraso de voo"`. Vocabulário de tradução PT→EN em
[Backend .NET](../02-arquitetura/02-backend-dotnet.md#idioma).

O frontend acompanha o contrato da API (inglês) e exibe rótulos em português para o
usuário final.

**Contrato.** O backend é a única fonte de verdade do formato JSON. O frontend não
recalcula métrica: se um número aparece na tela, ele veio pronto da API. Isso vale
inclusive para a [nota de força](../01-produto/04-forca-do-entendimento.md), que a
API devolve já decomposta em componentes.

**Nada de dado inventado.** Onde a fonte não tem, a API devolve vazio e a interface
mostra "sem dado". Ver [Limitações da fonte](../03-dados/04-limitacoes-da-fonte.md).
