# Repositórios e responsabilidades

O projeto vive em **repositórios git independentes**, mais um protótipo antigo que não é
referência.

```
D:/Desenvolvimento/fatec/API/API-5/     ← pasta de trabalho (não é repo)
├── API-5/          documentação do SM        Concord-API/API-5
├── API5-Backend/   backend .NET 8            Concord-API/API5-Backend
├── API5-Frontend/  frontend React            Concord-API/API5-Frontend
├── Docs/           esta wiki                 victorgsnogueira/API-5-docs
└── prototipo/      protótipo antigo ⚠        victorgsnogueira/prototipo
```

> **Dois repositórios de documentação.** `API-5` (do SM) e `API-5-docs` (esta wiki)
> cobrem território vizinho. Vale combinar a divisão antes que a mesma informação passe
> a existir nos dois e as versões divirjam. Sugestão: `API-5` fica com artefatos de
> gestão — sprints, atas, entregas, apresentações — e `API-5-docs` com documentação
> técnica e de produto.
>
> **Nota de organização:** os três repositórios de projeto estão na organização
> **Concord-API**; a wiki e o protótipo estão na conta pessoal
> **victorgsnogueira**. Documentação de time em conta pessoal é ponto de atrito
> previsível (acesso, permissão, continuidade). Considere transferir `API-5-docs` para a
> organização.

---

## Docs — esta wiki

| | |
|---|---|
| Remoto | `victorgsnogueira/API-5-docs` |
| Conteúdo | documentação técnica e de produto, mockups em [`Telas/`](../Telas/) |
| Estado | ✅ versionado e publicado |

### Como manter

- **A wiki acompanha a decisão, não a sucede.** Decidiu algo em reunião? A decisão vira
  uma entrada em [Decisões e riscos](../06-operacao/02-decisoes-e-riscos.md) no mesmo dia.
- **Pergunta em aberto é conteúdo.** Registrar "não sabemos ainda" vale mais que deixar
  a página em silêncio — é o que impede alguém de inventar a resposta sozinho.
- **Todo link é relativo.** Isso mantém a wiki navegável tanto no GitHub quanto no disco.
- **Link quebrado envelhece a wiki rápido.** Verificação automática de links entra no
  pipeline; ver [DevOps](../06-operacao/03-devops-e-infra.md).

Como a pasta virou repositório próprio, os caminhos citados aqui (`API5-Backend/…`,
`prototipo/…`) apontam para **fora** deste repositório — eles pressupõem os quatro
clones lado a lado, como no diagrama acima.

---

## API-5 — documentação do SM

| | |
|---|---|
| Remoto | `Concord-API/API-5` |
| Responsável | **Scrum Master** do time |
| Estado | README vazio, um commit inicial |

---

## API5-Backend — backend .NET 8

| | |
|---|---|
| Remoto | `Concord-API/API5-Backend` |
| Estado | solução criada, camadas em branco (scaffold) |

Solução `Ratio.slnx`, sete projetos em Clean Architecture:

| Projeto | Papel | Estado |
|---|---|---|
| `Ratio.Domain` | entidades e regras de negócio puras, sem dependência | `Class1.cs` vazio |
| `Ratio.Application` | casos de uso, portas (interfaces); depende só do Domain | `Class1.cs` vazio |
| `Ratio.Infrastructure` | acesso ao Postgres, clientes das fontes, migrations | `Class1.cs` vazio |
| `Ratio.Api` | ASP.NET Core Web API + Swashbuckle (Swagger) | scaffold `WeatherForecast` |
| `Ratio.Etl` | console app da carga (agendável) | `Program.cs` scaffold |
| `Ratio.Application.Tests` | xUnit sobre Application e Domain | `UnitTest1.cs` |
| `Ratio.Infrastructure.Tests` | xUnit sobre Infrastructure | `UnitTest1.cs` |

Detalhe das camadas, do idioma e do contrato da API:
[Backend .NET](../02-arquitetura/02-backend-dotnet.md).

**Primeira tarefa real:** remover o scaffold `WeatherForecast` (controller + modelo)
antes que ele vaze para o Swagger de produção.

---

## API5-Frontend — frontend React

| | |
|---|---|
| Remoto | `Concord-API/API5-Frontend` |
| Estado | **vazio** — sem nenhum commit |

Ponto de partida: [Frontend React](../02-arquitetura/03-frontend-react.md) e o
[Design system](../04-design/01-design-system.md).

---

## prototipo — protótipo antigo

| | |
|---|---|
| Remoto | `victorgsnogueira/prototipo` |
| Estado | funcional, mas **fora do projeto** |

> ⚠ **Não é fonte de verdade.** Foi construído antes das definições atuais de produto,
> escopo, fontes e arquitetura, e a modelagem de banco dele foi criada sem auditoria
> alguma. Serve como catálogo de armadilhas conhecidas das fontes — nada mais.

Ver [Protótipo — o que é e o que não é](../05-prototipo/01-prototipo-referencia.md).

---

## Convenções entre repositórios

**Idioma.** O **backend inteiro é escrito em inglês** — classes, métodos, tabelas,
colunas, rotas, campos JSON, logs e commits. Os **dados tratados permanecem em
português**, porque o domínio é o direito brasileiro: `topic.name` vale
`"Atraso de voo"`. Vocabulário de tradução PT→EN em
[Backend .NET](../02-arquitetura/02-backend-dotnet.md#idioma).

O frontend acompanha o contrato da API (inglês) e exibe rótulos em português para o
usuário final. Esta wiki é escrita em português.

**Contrato.** O backend é a única fonte de verdade do formato JSON. O frontend não
recalcula métrica: se um número aparece na tela, ele veio pronto da API — inclusive a
[nota de força](../01-produto/04-forca-do-entendimento.md), que vem já decomposta em
componentes.

**Nada de dado inventado.** Onde a fonte não tem, a API devolve vazio e a interface
mostra "sem dado". Ver [Limitações da fonte](../03-dados/04-limitacoes-da-fonte.md).

**Proveniência.** O produto é [multifonte](../03-dados/01-fontes.md): toda resposta diz
de que fonte veio o dado e quando foi extraído.

**CI/CD em todos.** Cada repositório tem seu pipeline, inclusive este.
Ver [DevOps e infraestrutura](../06-operacao/03-devops-e-infra.md).
