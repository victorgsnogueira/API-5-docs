# Padrão de desenvolvimento: TDD

**Modelo adotado:** *Test-Driven Development* — o teste é escrito **antes** do código
que o faz passar. Vale para o **backend** (`API5-Backend`) e para o **frontend**
(`API5-Frontend`).

---

## O ciclo

```
   🔴 RED        escreva um teste que descreve o comportamento desejado
                 e veja-o FALHAR pelo motivo certo
        │
        ▼
   🟢 GREEN      escreva o MÍNIMO de código que o faz passar
        │
        ▼
   🔵 REFACTOR   melhore o código com a rede de testes verde
        │
        └──────► próximo comportamento
```

Três regras que tornam o ciclo real, e não teatro:

1. **Nenhum código de produção sem um teste falhando que o peça.** Endpoint, caso de
   uso, componente, formatação — tudo começa por um teste vermelho.
2. **Ver o teste falhar é obrigatório.** Um teste que nunca foi visto vermelho pode estar
   passando por acidente (asserção errada, mock que devolve qualquer coisa).
3. **Refatorar só no verde.** Mudança de comportamento e mudança de estrutura não
   acontecem ao mesmo tempo.

---

## Backend — .NET

### Ferramentas

| Papel | Ferramenta | Estado no repo |
|---|---|---|
| Framework de teste | **xUnit** | ✅ referenciado (2.5.3 — atualizar) |
| Cobertura | **coverlet** | ✅ referenciado |
| Asserção | **`Assert` do próprio xUnit** | ✅ já vem com o xUnit |
| Dublê de teste | **Moq** | a adicionar |
| Teste de API em memória | **`Microsoft.AspNetCore.Mvc.Testing`** (`WebApplicationFactory`) | a adicionar |
| Postgres real em teste | **Testcontainers** (`Testcontainers.PostgreSql`) com a imagem `pgvector/pgvector:pg16` | a adicionar |

> **Sem biblioteca de asserção extra.** O `Assert` do xUnit (`Assert.Equal`,
> `Assert.DoesNotContain`, `Assert.Throws`…) cobre o que o projeto precisa. Não
> adicione FluentAssertions — a partir da v8 ele tem licença comercial.

> **Moq: fixe versão ≥ 4.20.70.** A 4.20.0 embutiu o SponsorLink (coleta de e-mail
> no build); foi removido nas versões seguintes.

> **Por que Testcontainers e não SQLite/in-memory.** O produto depende de coisas que só o
> Postgres tem: `unaccent`, `pg_trgm`, `vector`, views materializadas, `FILTER (WHERE …)`.
> Um banco falso passaria nos testes e quebraria em produção. O teste de repositório
> sobe o **mesmo** Postgres que roda em produção.

### Onde cada teste mora

A pirâmide segue as camadas da [solução](../02-arquitetura/02-backend-dotnet.md#estrutura-da-solução):

| Camada | Tipo de teste | Projeto | Toca banco? |
|---|---|---|---|
| `Ratio.Domain` | unitário puro — cálculo da nota de força, polaridade, regras | `Ratio.Application.Tests` | não |
| `Ratio.Application` | unitário — casos de uso com repositório substituído (Moq) | `Ratio.Application.Tests` | não |
| `Ratio.Infrastructure` | integração — repositório contra Postgres em Testcontainers | `Ratio.Infrastructure.Tests` | **sim** |
| `Ratio.Api` | integração — HTTP em memória com `WebApplicationFactory` | `Ratio.Api.Tests` *(a criar)* | opcional |

### Exemplo do ciclo — a regra da polaridade

A regra mais cara do produto ([Polaridade](../03-dados/05-polaridade-do-resultado.md)) é
exatamente o tipo de coisa que se trava com teste **antes**:

```csharp
// 🔴 RED — escrito antes de PolarityLabel existir
public class PolarityLabelTests
{
    [Fact]
    public void Prosecution_claims_are_never_labelled_as_favorable()
    {
        var label = PolarityLabel.For(ClaimantType.Prosecution);

        Assert.DoesNotContain("favorável", label);
        Assert.Equal("acolhimento da pretensão acusatória (procedência = condenação)", label);
    }
}
```

E com dublê, num caso de uso da Application:

```csharp
[Fact]
public async Task Unknown_topic_returns_not_found()
{
    var repository = new Mock<ITopicRepository>();
    repository.Setup(r => r.GetByCodeAsync(999, It.IsAny<CancellationToken>()))
              .ReturnsAsync((TopicDetail?)null);
    var useCase = new GetTopicDetail(repository.Object);

    var result = await useCase.ExecuteAsync(999, CancellationToken.None);

    Assert.True(result.IsNotFound);
    repository.Verify(r => r.GetByCodeAsync(999, It.IsAny<CancellationToken>()), Times.Once);
}
```

Note o padrão do projeto: **identificador em inglês, valor em português** — o teste
confere a string que a API devolve ao usuário. Ver [Idioma](../02-arquitetura/02-backend-dotnet.md#idioma).

### Nomenclatura

- Classe: `<ClasseSobTeste>Tests` — `StrengthScoreTests`.
- Método: frase em inglês com `_`, descrevendo o comportamento —
  `Coverage_saturates_at_three_courts`, não `Test1`.
- Um comportamento por teste. Arrange / Act / Assert separados por linha em branco.

### Comandos

```bash
cd API5-Backend/Ratio
dotnet test                                              # tudo
dotnet test --filter "FullyQualifiedName~StrengthScore"  # um recorte
dotnet test --collect:"XPlat Code Coverage"              # com cobertura
```

---

## Frontend — React

### Ferramentas

| Papel | Ferramenta | Por quê |
|---|---|---|
| Runner | **Vitest** | nativo do Vite (já é o build do repo); mesma config, mesmos aliases (`@/`, `@workspace/ui`) |
| DOM | **jsdom** | ambiente de browser para o Vitest |
| Componente | **React Testing Library** + `@testing-library/user-event` + `@testing-library/jest-dom` | testa o que o usuário vê e faz, não detalhe de implementação |
| API falsa | **MSW** (Mock Service Worker) | intercepta `fetch` na rede; o componente não sabe que é teste |
| Rotas | `createMemoryHistory` do **TanStack Router** | renderiza a rota real com URL controlada — testa `?tab=analytics` de verdade |
| E2E *(opcional, fim de sprint)* | **Playwright** | fluxo Busca → Resultados → Tema contra a API real |

**Nada disso está instalado ainda.** O CI do frontend hoje roda lint, typecheck e build
— falta o passo de teste. Ver [DevOps](../06-operacao/03-devops-e-infra.md#pipeline-de-cicd--esqueleto).

### O que se testa primeiro

As [regras não negociáveis do frontend](../02-arquitetura/03-frontend-react.md#regras-não-negociáveis)
são, cada uma, um teste que nasce vermelho:

| Regra | Teste |
|---|---|
| Não recalcular métrica | o componente exibe `82%` quando a API manda `0.82` — e não calcula a partir de contagens |
| Formatação pt-BR | `12418` vira `12.418`, não `12,418` |
| Figura exige fonte | `Figure` sem a prop `source` **não compila** (teste de tipo) |
| Estado vazio é conteúdo | lista vazia de doutrina renderiza o **motivo**, não uma tabela vazia |
| Nunca "% favorável" | `AlignmentBar` renderiza o rótulo que veio da API; o texto "favorável" não aparece |
| Aba na URL | navegar para `/topics/1234?tab=analytics` abre a Base analítica |

### Exemplo do ciclo

```tsx
// 🔴 RED — AlignmentBar.test.tsx, escrito antes do componente
it("renders the polarity label sent by the API, never 'favorável'", () => {
  render(
    <AlignmentBar
      share={0.986}
      label="acolhimento da pretensão acusatória (procedência = condenação)"
    />
  )

  expect(screen.getByText(/98,6%/)).toBeInTheDocument()
  expect(screen.getByText(/pretensão acusatória/)).toBeInTheDocument()
  expect(screen.queryByText(/favorável/i)).not.toBeInTheDocument()
})
```

### Convenções

- Arquivo de teste ao lado do componente: `AlignmentBar.tsx` + `AlignmentBar.test.tsx`.
- Descrição do teste em inglês; o texto buscado na tela, em português (é o que o usuário lê).
- Busque por **papel e texto** (`getByRole`, `getByText`), nunca por classe CSS ou
  `data-testid` quando houver alternativa acessível — de brinde, o teste cobra
  acessibilidade.
- Componentes do `packages/ui` (shadcn) não precisam de teste próprio para o que é do
  shadcn; testa-se o **uso** deles nas telas.

### Comandos *(depois de configurado)*

```bash
cd API5-Frontend/ratio
npm run test          # turbo test → vitest run em cada workspace
npx vitest            # modo watch, dentro de apps/web — o do dia a dia no TDD
```

---

## Pipeline de dados

O [pipeline de carga](../02-arquitetura/05-etl-e-nlp.md#carga-manual--o-processo) é
Python e fica fora do escopo formal de TDD (que vale para backend e frontend). Ele tem
duas redes próprias:

- **24 testes de integridade em SQL** contra o DW, obrigatórios a cada carga;
- funções puras de transformação (`clean()`, extratores, mapa de polaridade) são
  candidatas naturais a **pytest** — recomendado, não exigido.

---

## TDD, commits e branches

- **Teste e implementação vão no mesmo commit** (`feat:` ou `fix:`). A branch nunca
  recebe um commit vermelho — o histórico de cada commit compila e passa.
- Commit **só de teste** (cobrir comportamento que já existia, sem mudar código) usa o
  tipo `test:`. Ver [Padrão de commits](02-commits.md).
- **Todo `fix:` começa por um teste que reproduz o bug.** O teste falha, o fix o faz
  passar, e o bug não volta sem alguém perceber.
- O pull request de task → `usX` **não é aprovado com teste falhando** — o CI bloqueia.
  Ver [Padrão de branches](01-branches.md#relação-com-o-cicd).

---

## Por que TDD

### Porque o produto é feito de regras que erram em silêncio

Os erros mais caros encontrados até aqui **não quebraram nada visível**: a palavra
"favorável" invertendo o sentido de temas penais, a doutrina de filosofia moral ligada a
*dano moral*, o agregado que duplicava linhas por produto cartesiano. Cada um produzia
uma tela bonita e errada. Só um teste que afirma o comportamento esperado pega esse tipo
de erro — e escrevê-lo **antes** obriga a decidir o comportamento antes de codar.

### Porque o público cita o que lê

O usuário é advogado e juiz. Um percentual errado na tela vira argumento em petição.
Teste é a forma de provar que o número exibido é o número calculado.

### Porque trava o contrato entre as duas pontas

A API devolve texto em português e o frontend não recalcula nada. Um teste de cada lado
sobre o mesmo contrato (`polarityLabel`, `strengthScore.components`) é o que impede
backend e frontend de divergirem sem ninguém notar.

### Porque é requisito do desafio

O desafio cobra "testes automatizados validando integridade dos dados e consistência das
consultas". TDD faz a cobertura crescer junto com o código, em vez de virar uma tarefa
de fim de sprint que nunca acontece.
