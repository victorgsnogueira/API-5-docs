# Frontend React

Repositório `API5-Frontend` — **hoje vazio**. Esta página é o ponto de partida, não a
documentação de algo que já existe.

## Escopo

Três telas, detalhadas em [Telas](../01-produto/02-telas.md):

| Rota | Tela | API |
|---|---|---|
| `/` | Busca | `GET /api/topics?q=` (sugestões / temas de maior volume) |
| `/busca?q=…` | Resultados + filtros | `GET /api/topics?q=…` |
| `/tema/:code` | Detalhamento (abas Resumo / Base analítica) | `GET /api/topics/{code}` e `/decisions` |

A URL é voltada ao usuário e fica em português; o **contrato da API é em inglês**, como
todo o [backend](02-backend-dotnet.md#idioma).

A aba do detalhamento deve ser refletida na URL (`?aba=resumo` ou `?aba=base`) — o
usuário vai querer mandar o link da base analítica para um colega.

## Recomendação de stack

| Item | Escolha | Por quê |
|---|---|---|
| Build | **Vite** | padrão atual para SPA React; build estático que o Coolify serve bem |
| Linguagem | **TypeScript** | o contrato da API tem objetos aninhados (`strengthScore.components`) — tipar evita erro silencioso na tela |
| Roteamento | **React Router** | três rotas, nada exótico |
| Dados | **TanStack Query** | cache, estado de carregando/erro e revalidação sem escrever isso à mão em cada tela |
| Estilo | **CSS puro com custom properties** | o [design system](../04-design/01-design-system.md) é enxuto e muito específico; um framework de utilitários brigaria com ele |
| Gráficos | **SVG escrito à mão** | são três formas simples — barra horizontal, mini-barras por ano, faixa interquartil. Uma biblioteca traria estilo default para brigar com o design |

Sobre os gráficos: as figuras do mockup (`FIG. 1`, `FIG. 2`) são barras retangulares
sem raio, sem eixo, sem grade e sem tooltip. Isso é menos código em SVG puro do que
configurando Recharts para desligar tudo.

## Estrutura sugerida

```
src/
  main.tsx
  App.tsx                  rotas
  api/
    client.ts              fetch base, URL da API por env
    topics.ts              tipos + funções (searchTopics, getTopic, listDecisions)
  telas/
    Busca.tsx
    Resultados.tsx
    Tema.tsx
      abas/Resumo.tsx
      abas/BaseAnalitica.tsx
  componentes/
    Marca.tsx              logotipo (64px e 25px)
    CampoBusca.tsx
    Score.tsx              círculo 70px / 104px
    TagMateria.tsx
    ChipTribunal.tsx
    BarraAlinhamento.tsx
    Figura.tsx             moldura numerada: rótulo, conteúdo, fonte declarada
    Tabela.tsx             cabeçalho mono, número em mono à direita
    PainelFiltros.tsx
    AvisoEscopo.tsx        "dados de TJSP, TJRJ e TJMG"
    Proveniencia.tsx       fonte + data de extração
  estilos/
    tokens.css             cores, tipografia, espaçamento
    base.css
```

## Regras não negociáveis

1. **Não recalcular métrica.** Percentual, nota e mediana vêm prontos da API. O
   frontend formata, não calcula. Formatação numérica em `pt-BR`
   (`Intl.NumberFormat`): 12.418, não 12,418.
2. **Toda figura declara a fonte.** O componente `Figura` deve **exigir** a prop de
   fonte — se não há fonte, não há figura.
3. **Estado vazio é conteúdo, não erro.** Quando a API devolve lista vazia (doutrina,
   fundamentos), a tela diz o que não existe e por quê, com link para
   [Limitações da fonte](../03-dados/04-limitacoes-da-fonte.md). Nunca inventa
   placeholder.
4. **Vermelho é sinal.** Reservado a ação, link, valor predominante e ao que o leitor
   precisa conferir. Nunca decoração.
5. **Todo número clicável leva à fonte.** Número de processo abre o tribunal; marcador
   `[1]` leva ao rodapé de decisões citadas.
6. **Sem sombra, sem gradiente, sem ícone decorativo.** Ver
   [Design system](../04-design/01-design-system.md).
7. **Declarar o escopo e a proveniência.** Os dados cobrem **TJSP, TJRJ e TJMG** e vêm
   de [várias fontes](../03-dados/01-fontes.md). Toda tela com número diz de onde ele
   veio e quando foi extraído — quem lê "82%" precisa saber que são três estados.

## Nomenclatura

Identificadores em inglês, acompanhando a API (`topics.ts`, `searchTopics`,
`strengthScore`). O **texto visível ao usuário é em português** — rótulos, títulos e
mensagens. Componentes com nome em português são aceitáveis onde espelham vocabulário
de tela (`PainelFiltros`), mas nada de misturar os dois num mesmo identificador.

## Configuração

| Variável | Exemplo |
|---|---|
| `VITE_API_URL` | `http://localhost:5000` em dev; injetada pelo Coolify em produção |

O backend precisa ter a origem do dev server (`http://localhost:5173`) na lista
explícita de CORS.

## Acessibilidade e tipografia

- As fontes vêm do Google Fonts: **Source Serif 4** (400, 600, 700 + itálico) e
  **IBM Plex Mono** (400, 500, 600). Sempre com pilha de fallback real
  (`Source Serif 4, Georgia, serif` / `IBM Plex Mono, ui-monospace, monospace`).
- Contraste: `#8B8478` sobre `#F4F1EA` é o par mais fraco da paleta — reserve-o a
  metadado e legenda, nunca a texto que precisa ser lido.
- Tabela larga rola dentro do próprio contêiner (`overflow-x: auto`); o corpo da
  página nunca rola na horizontal.
- Foco visível em todo controle. O campo de busca é o primeiro elemento focável.
