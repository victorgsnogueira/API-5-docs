# Frontend React

Repositório `API5-Frontend`. **Scaffold criado** (commit `842da1b`): monorepo com
design system já mapeado em tokens, roteamento por arquivo e três rotas vazias. Nenhuma
tela real ainda.

## Escopo

Três telas, detalhadas em [Telas](../01-produto/02-telas.md):

| Rota | Tela | API |
|---|---|---|
| `/` | Busca | `GET /api/topics?q=` (sugestões / temas de maior volume) |
| `/busca?q=…` | Resultados + filtros | `GET /api/topics?q=…` |
| `/tema/$code?aba=resumo\|base` | Detalhamento (abas Resumo / Base analítica) | `GET /api/topics/{code}` e `/decisions` |

> **Estado no repo:** existem `routes/index.tsx`, `routes/result.tsx` e
> `routes/details.tsx` — a rota de detalhe ainda **não recebe o código do tema**.
> Renomear para `routes/busca.tsx` e `routes/tema.$code.tsx` na primeira task de tela.

**Rotas em português.** A URL é o que o usuário vê, copia e manda para um colega — é
texto de tela, não identificador. Por isso é a exceção à regra de código em inglês
([D-27](../06-operacao/02-decisoes-e-riscos.md#d-27--rotas-do-frontend-em-português)).
O nome do parâmetro de rota (`$code`) fica em inglês: ele não aparece na URL, só no
código. A **API** continua com rotas em inglês (`/api/topics`).

A aba do detalhamento é **search param validado** (`?aba=base`), não estado local: o
usuário vai querer mandar o link da base analítica para um colega. O TanStack Router
valida o parâmetro com `validateSearch` — aba inválida cai em `resumo`, não em tela
quebrada.

---

## Stack

O que **está no repo** hoje, conferido nos `package.json`:

| Item | Escolha | Versão | Estado |
|---|---|---|---|
| Linguagem | **TypeScript** | ~6 | ✅ |
| UI | **React** | 19 | ✅ |
| Build | **Vite** | 8 | ✅ |
| Roteamento | **TanStack Router** — por arquivo, com `@tanstack/router-plugin` e *auto code splitting* | 1.170 | ✅ |
| Componentes | **shadcn/ui** sobre **Base UI** (`@base-ui/react`, estilo `base-vega`) | shadcn 4 | ✅ só `button` |
| Estilo | **Tailwind CSS v4** (`@tailwindcss/vite`, config CSS-first, sem `tailwind.config.js`) | 4 | ✅ |
| Variantes | `class-variance-authority` + `cn` | — | ✅ |
| Ícones | `lucide-react` | 1.43 | ✅ |
| Fontes | **self-hosted** via Fontsource — Source Serif 4 (variável) e IBM Plex Mono 400/500/600 | — | ✅ |
| Validação | `zod` | 4 | ✅ instalado em `packages/ui` |
| Monorepo | **Turborepo** + npm workspaces (`apps/*`, `packages/*`) | turbo 2.9 | ✅ |
| Qualidade | ESLint 10 + typescript-eslint · Prettier 3 com `prettier-plugin-tailwindcss` | — | ✅ |
| Runtime | Node **≥ 20**, npm 11 | — | ✅ |

E o que **falta** e se recomenda:

| Item | Escolha | Por quê |
|---|---|---|
| Dados do servidor | **TanStack Query** | cache, carregando/erro e revalidação sem reescrever isso em cada tela; integra com o *loader* do TanStack Router (`ensureQueryData`) |
| Contrato | **zod** nos tipos da API | já está instalado; valida a resposta na fronteira — se o backend mudar um campo, quebra no parse, não no meio da tela |
| Testes | **Vitest** + Testing Library + MSW | o padrão de desenvolvimento é [TDD](../07-justificativas/03-tdd.md) |
| Gráficos | **SVG escrito à mão** | as figuras do mockup são barras retangulares sem raio, sem eixo, sem grade e sem tooltip — menos código em SVG puro que configurando uma biblioteca para desligar tudo |

### Por que essa stack

- **TanStack Router** tipa rota, parâmetro e search param de ponta a ponta. Numa tela
  cuja aba mora na URL (`?aba=`), um link errado vira erro de compilação, não 404.
- **shadcn/ui** não é dependência: o código do componente é **copiado** para
  `packages/ui` e passa a ser nosso. É isso que permite dobrar cada componente ao
  [design system](../04-design/01-design-system.md) — sem sombra, raio de 3px, vermelho
  só como sinal — em vez de brigar com o estilo default de uma biblioteca fechada.
- **Tailwind v4** lê os tokens direto de CSS custom properties (`@theme inline`). O
  design system vira **variável CSS uma vez** e todas as classes (`bg-background`,
  `text-primary`) passam a significar a paleta do Ratio.
- **Monorepo** separa o que é design system (`packages/ui`) do que é aplicação
  (`apps/web`). Se um dia houver um segundo app (painel admin, página do chatbot), ele
  reaproveita os componentes sem copiar.

---

## Estrutura

```
API5-Frontend/
├── .github/workflows/ci.yml        lint · typecheck · build   (falta: test)
└── ratio/                          raiz do monorepo (npm workspaces + turbo)
    ├── apps/web/                   a aplicação
    │   ├── components.json         config do shadcn (aponta para packages/ui)
    │   ├── vite.config.ts          tanstackRouter() + react() + tailwindcss(), alias @/
    │   └── src/
    │       ├── main.tsx            createRouter + RouterProvider
    │       ├── routeTree.gen.ts    GERADO pelo plugin — não editar, não revisar
    │       ├── routes/             uma rota por arquivo
    │       │   ├── __root.tsx
    │       │   ├── index.tsx       /
    │       │   ├── result.tsx      → renomear: busca.tsx
    │       │   └── details.tsx     → renomear: tema.$code.tsx
    │       ├── api/                (a criar) client.ts + topics.ts: schemas zod + funções
    │       ├── features/           (a criar) search/ · results/ · topic/
    │       └── components/         componentes de tela
    └── packages/ui/                o design system
        └── src/
            ├── styles/globals.css  ⭐ tokens do design system mapeados para o shadcn
            ├── components/         componentes shadcn, já restilizados
            ├── hooks/
            └── lib/utils.ts
```

### Onde fica cada coisa

| Se é… | Vai em |
|---|---|
| Primitivo visual reutilizável (botão, tabela, chip, tag) | `packages/ui/src/components` — via `npx shadcn add` e depois restilizado |
| Componente do domínio (`StrengthScore`, `AlignmentBar`, `Figure`, `Provenance`, `EmptyState`) | `apps/web/src/components` |
| Tela | `apps/web/src/routes` (a rota) + `apps/web/src/features/<tela>` (o conteúdo) |
| Chamada à API e tipo do contrato | `apps/web/src/api` |

Adicionar um componente shadcn, a partir de `ratio/apps/web`:

```bash
npx shadcn@latest add table
```

Ele cai em `packages/ui/src/components`. **Revise antes de commitar**: o default do
shadcn traz `shadow-xs`, `rounded-md` e estados `dark:` que o design system não usa.

---

## O design system já está nos tokens

`packages/ui/src/styles/globals.css` mapeia a paleta do [design system](../04-design/01-design-system.md)
para as variáveis que o shadcn espera:

| Variável shadcn | Valor | Papel no Ratio |
|---|---|---|
| `--background` | `#F4F1EA` | papel |
| `--card` | `#FFFFFF` | bloco branco |
| `--foreground` | `#14120F` | tinta |
| `--muted-foreground` | `#8B8478` | metadado — **nunca** texto que precisa ser lido |
| `--primary` / `--ring` / `--destructive` | `#A3121A` | o **único vermelho**: sinal, ação, link |
| `--border` | `#DCD6C9` | filete |
| `--chart-1…5` | `#E4CFCF` → `#7B0D14` | escala de dados |
| `--radius` | `3px` | raio máximo |

**Não há modo escuro.** O design system é papel e tinta; o `ThemeProvider` e o
`@custom-variant dark` do template ficaram só para não quebrar o `button.tsx` gerado —
nenhuma classe `.dark` é aplicada. Não construa nada dependendo de `dark:`.

---

## Regras não negociáveis

1. **Não recalcular métrica.** Percentual, nota e mediana vêm prontos da API. O
   frontend formata, não calcula. Formatação numérica em `pt-BR`
   (`Intl.NumberFormat`): 12.418, não 12,418.
2. **Não escrever rótulo de polaridade.** O texto ao lado de um percentual de resultado
   (`polarityLabel`) **vem da API**. O frontend nunca escreve "favorável" — ver
   [Polaridade do resultado](../03-dados/05-polaridade-do-resultado.md).
3. **Toda figura declara a fonte.** O componente `Figure` **exige** a prop `source` no
   tipo — sem fonte, não compila.
4. **Estado vazio é conteúdo, não erro.** Quando a API devolve lista vazia (doutrina,
   fundamentos), a tela diz o que não existe e por quê — o motivo vem da API. Nunca
   placeholder.
5. **Vermelho é sinal.** Reservado a ação, link, valor predominante e ao que o leitor
   precisa conferir. Nunca decoração.
6. **Todo número clicável leva à fonte.** Número de processo abre o tribunal, com o
   rótulo "consultar no tribunal" — nunca "veja a decisão".
7. **Sem sombra, sem gradiente, sem ícone decorativo.** `lucide-react` só para ícone
   com função (fechar, expandir, link externo).
8. **Declarar o escopo e a proveniência.** Toda tela com número diz de onde ele veio e
   quando foi extraído — quem lê "82%" precisa saber que são dois/três estados.

Cada uma dessas regras é um teste que nasce vermelho. Ver [TDD](../07-justificativas/03-tdd.md#o-que-se-testa-primeiro).

---

## Idioma

| O quê | Idioma | Exemplo |
|---|---|---|
| Identificador — componente, função, variável, tipo | **inglês** | `AlignmentBar`, `searchTopics` |
| **Rota** — a URL que o usuário vê, e portanto o nome do arquivo em `routes/` | **português** | `/busca`, `/tema/123?aba=base` |
| Chave do JSON da API (e portanto do tipo TS) | **inglês** | `strengthScore`, `polarityLabel` |
| Texto que o usuário lê — rótulo, título, mensagem, estado vazio | **português** | "Base analítica", "consultar no tribunal" |
| Texto que **vem da API** (rótulo de polaridade, motivo de dado ausente, grau da nota) | **português**, exibido como veio | "Divergente" |
| Descrição de teste e comentário | inglês | `it("renders the polarity label…")` |

Não misture os dois num mesmo identificador (`ScoreForca`, `useTema`).

---

## Configuração

| Variável | Exemplo |
|---|---|
| `VITE_API_URL` | `http://localhost:5000` **só em dev** |

**Em produção a API é chamada por caminho relativo (`/api/...`)**: o NGINX do cliente
serve o frontend e repassa `/api/` para a API, na mesma origem. Como o endereço da
intranet do cliente não é conhecido no build, uma URL fixa obrigaria um build por
cliente. Ver [Implantação no cliente](../06-operacao/04-implantacao-no-cliente.md#frontend).

O backend precisa ter a origem do dev server (`http://localhost:5173`) na lista
explícita de CORS — só em dev.

> `VITE_*` é embutida no bundle **em tempo de build** — não é segredo e não muda depois
> do deploy sem rebuild. Nunca ponha chave ali.

## Comandos

```bash
cd API5-Frontend/ratio
npm install
npm run dev          # turbo dev → vite em http://localhost:5173
npm run lint
npm run typecheck
npm run build
npm run format
```

## Acessibilidade e tipografia

- Fontes **self-hosted** (Fontsource), carregadas por `globals.css` — sem requisição ao
  Google Fonts. **Obrigatório**, não preferência: produção é intranet, e nenhum recurso
  pode vir de CDN. Pilha de fallback real
  (`Source Serif 4, Georgia, serif` / `IBM Plex Mono, ui-monospace, monospace`).
- Contraste: `#8B8478` sobre `#F4F1EA` é o par mais fraco da paleta — reserve-o a
  metadado e legenda.
- Tabela larga rola dentro do próprio contêiner (`overflow-x-auto`); o corpo da página
  nunca rola na horizontal.
- Foco visível em todo controle (`focus-visible:ring-ring`, já no `button`). O campo de
  busca é o primeiro elemento focável.
- Os primitivos do **Base UI** já entregam teclado e ARIA corretos em menu, diálogo e
  abas — use-os em vez de montar `div` clicável.

---

## Referência visual

O [protótipo de dados de set/2026](../05-prototipo/01-prototipo-referencia.md#protótipo-de-dados--setembro2026)
(`prototipo-prod - versao 202609/web`) renderiza as três telas contra o DW real. **Não é
código a portar** (usa React Router e CSS puro), mas mostra como o dado real se
comporta em cada bloco — inclusive os blocos que ficam sem fonte.
