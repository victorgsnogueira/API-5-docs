# Telas

Três telas. Os mockups fechados estão em [`Telas/`](../Telas/); as regras
visuais estão em [Design system](../04-design/01-design-system.md).

---

## 1 · Tela inicial — Busca

![Tela inicial](../Telas/Tela%20Inicial%20-%20Busca.png)

Tela única, centrada, sem navegação. Fundo: papel `#F4F1EA` com a ilustração da
Justiça em meio-tom, quase apagada — presença, não decoração.

| Elemento | Especificação |
|---|---|
| Logotipo | `Rat`**i**`o` em serifa 64px/700, o **i** em itálico vermelho `#A3121A`; filete escuro abaixo |
| Assinatura | *"Precedentes que se repetem viram padrão."* — serifa itálica, apoio |
| Campo de busca | borda 1px `#C9C2B4`, fundo branco, raio 3px; botão **BUSCAR** em vermelho cheio, mono uppercase |
| Sugestões | chips de consultas frequentes, fundo `#FBF9F4`, serifa; ex.: *atraso de voo superior a quatro horas*, *rescisão indireta por atraso salarial*, *tarifa de água · prescrição* |

**Regras.** O placeholder mostra tema em linguagem natural, nunca número de processo —
a tela ensina o que se busca aqui. Busca vazia não é erro: devolve os temas de maior
volume.

**API.** `GET /api/themes?q=` (sem termo, retorna os temas de maior volume).

---

## 2 · Tela de resultados

![Resultados](../Telas/Tela%20de%20Busca%20-%20Resultados.png)

Barra superior branca com o logotipo reduzido (25px) e o campo de busca inline.
Cabeçalho: **"5 teses para _inscrição indevida em cadastro de inadimplentes_"**, com
o controle `FILTROS 3 ▾` e o rótulo `ORDENADO POR FORÇA` à direita.

### Item da lista

```
┌──────┐   ┌────────────┐
│  77  │   │ CONSUMIDOR │  <- matéria: tag preta, mono uppercase
│ /100 │   └────────────┘
└──────┘   Inscrição indevida em cadastro de inadimplentes
  score    gera dano moral in re ipsa            <- serifa 24px/700
 (70px)
           Predomina o entendimento de que a negativação indevida
           dispensa prova do abalo. A exceção é a inscrição
           preexistente legítima, que afasta a indenização.

           [STJ][TJSP][TJMG][+2]  12.418 processos  2021 — 2026
           última decisão 21.08.2026    ========|  82% favorável
```

| Componente | Regra |
|---|---|
| Score | círculo 70px, borda 1px, número mono 27px/600, `/100` abaixo em 9px |
| Matéria | tag preta `#14120F`, mono 10px uppercase |
| Resumo | duas linhas, `#3A342C`, medida máx. 640px |
| Siglas de tribunal | chips mono. Com escopo em TJSP/TJRJ/TJMG, o `+N` raramente aparece |
| Metadados | mono 10–11px, `#8B8478` |
| Barra de alinhamento | 5px, trilha `#EBE6DB`; a fração predominante em vermelho |
| Separador | filete 1px `#DCD6C9` entre itens |
| Hover da linha | fundo `#FBF6F5` |

### Filtros

![Filtros](../Telas/Tela%20de%20Busca%20-%20Filtros.png)

Painel que se abre abaixo do cabeçalho, quatro colunas, sem sombra:

| Coluna | Controle |
|---|---|
| **Tribunal** | checkboxes com contagem à direita (`TJSP 5.412`). O mockup traz `+ 88 tribunais`; **com o escopo em três, esse link não existe** |
| **Período** | três botões exclusivos: `5 anos` (ativo, vermelho cheio) · `2 anos` · `Tudo` |
| **Grau** | checkboxes: 1º grau · 2º grau · Superior |
| **Força mínima** | slider 0–100 com o valor em mono; botão `APLICAR` e link `limpar filtros` |

O checkbox marcado é um quadrado vermelho cheio — sem ícone de "check".

**API.** `GET /api/themes?q=<termo>&court=&period=&level=&minStrength=&limit=` — o
contrato ainda não existe; ver [Backend .NET](../02-arquitetura/02-backend-dotnet.md#contrato-da-api).

---

## 3 · Tela de detalhamento

Cabeçalho comum às duas abas: link `‹ VOLTAR AOS 5 RESULTADOS`, score em círculo
104px com o rótulo `FORÇA DO ENTENDIMENTO`, tag de matéria, título em serifa 34px/700
e a linha de metadados em mono: `12.418 processos · 5 tribunais · 2021 — 2026 ·
última decisão 21.08.2026`.

Na barra superior, à direita: `EXPORTAR CSV` (contorno) e `COPIAR CITAÇÃO`
(vermelho cheio).

Abas: **RESUMO** | **BASE ANALÍTICA** — mono uppercase, ativa sublinhada 2px em
vermelho.

### 3a · Aba Resumo

![Resumo 1](../Telas/Tela%20de%20Detalhamento%20-%20Resumo%201.png)

Layout de duas colunas: artigo (~690px) + coluna lateral de indicadores (~250px).

**Coluna do artigo**

1. **Lead** — serifa 20px, filete vermelho 2px à esquerda, recuo 18px. Abre com o
   número que responde à pergunta: *"Em **82% das 12.418 decisões** analisadas…"*
2. **Corpo** — serifa 17px, entrelinha 1.75. Todo número no texto vem acompanhado do
   *n* e, quando sustenta uma afirmação específica, de marcador de citação `[1]` em
   mono vermelho superscrito, clicável, que leva ao rodapé.
3. **Citação de acórdão** — bloco com fundo `#FBF9F4` e filete vermelho 3px, texto em
   serifa itálica; abaixo, a referência em mono (`REsp 2.043.118/SP · STJ · 3ª Turma ·
   rel. min. Nancy Andrighi · j. 14.08.2026`) e o botão `Inteiro teor · PDF`.
4. **Figuras numeradas no fluxo do texto** — nunca grade de cartões:

   ![Resumo 2](../Telas/Tela%20de%20Detalhamento%20-%20Resumo%202.png)

   - `FIG. 1 — DESFECHO DAS 12.418 DECISÕES` — barras horizontais por desfecho
     (Procedente 7.699 · 62,0% / Parcialmente procedente 2.980 · 24,0% /
     Improcedente 1.739 · 14,0%), com a fonte declarada embaixo.
   - `FIG. 2 — VALOR FIXADO (R$)` — faixa interquartil com P25 `5.000`, mediana
     `8.000` em destaque vermelho e P75 `12.000`; legenda com a ressalva
     *"valores nominais, sem correção"*.
5. **Decisões citadas neste texto** — rodapé numerado `[1] [2] [3]`, cada entrada com
   número do processo em mono vermelho, órgão, relator, data, uma linha de síntese e
   os botões `PDF` e `Citar`:

   ![Resumo 3](../Telas/Tela%20de%20Detalhamento%20-%20Resumo%203.png)
6. **Pé** — `Ver base analítica completa ›` · `Baixar as 12.418 decisões (CSV)` e a
   nota de procedência: *"Texto gerado a partir das tabelas da base analítica. Fonte
   primária: DataJud/CNJ, extração de 28.08.2026 · metodologia v2.1."* — com o produto
   multifonte, essa nota precisa listar **todas** as fontes que alimentaram a página.

**Coluna lateral**

- `ALINHAMENTO POR ANO` — mini barras por ano (21…26), rampa
  `#E4CFCF → #C97C7F → #14120F → #A3121A`, o ano mais recente em vermelho cheio;
  legenda `69% -> 84% no sentido predominante`.
- `ALINHAMENTO POR TRIBUNAL` — lista sigla/percentual; os dois maiores em vermelho.
- Botão `BASE ANALÍTICA` levando à outra aba.

### 3b · Aba Base analítica

![Base analítica 1](../Telas/Tela%20de%20Detalhamento%20-%20Base%20Analitica%201.png)

Coluna única de blocos brancos com borda 1px `#DCD6C9`. Cada bloco tem título em
serifa e, à direita, um rótulo de escopo ou link de expansão.

| Bloco | Colunas | Nota |
|---|---|---|
| **Comportamento por tribunal** | tribunal · decisões · alinhamento · mediana R$ · última | no escopo atual são três linhas; a coluna de mediana [não tem fonte](../03-dados/04-limitacoes-da-fonte.md) |
| **Fundamentos invocados na decisão** | fundamento · natureza · citado em · acolhido · distribuição | rótulo `frequência × taxa de acolhimento`; natureza em serifa itálica (*Lei*, *Jurisprudência*, *Súmula*, *Tese defensiva*) |
| **Jurisprudência qualificada** | precedente · espécie · efeito · citado em · seguido · íntegra | efeito como tag: `Obrigatório` (preta cheia), `Vinculante de fato` / `Regional` / `Persuasivo` (contorno vermelho). Rodapé `DIVERGÊNCIA ABERTA` quando um tribunal foge do padrão |
| **Doutrina invocada** | autor — obra · edição/capítulo · posição · citações | posição como tag: *Corrente majoritária* / *Posição intermediária* / *Corrente minoritária*. **Nunca PDF de livro**: artigo vira link para onde está publicado; livro vira referência em texto — ver [Fontes](../03-dados/01-fontes.md#fonte-5--doutrina) |
| **Amostra auditável** | processo · órgão · relator · data · desfecho · valor · íntegra | número do processo em mono vermelho, clicável; link `abrir as 12.418 decisões` |

![Base analítica 2](../Telas/Tela%20de%20Detalhamento%20-%20Base%20Analitica%202.png)

![Base analítica 3](../Telas/Tela%20de%20Detalhamento%20-%20Base%20Analitica%203.png)

Rodapé geral: `Fonte: DataJud/CNJ, extração de 28.08.2026 · metodologia v2.1 ·
exportar tabelas (CSV)`.

**Regra de tabela.** Texto em serifa, número em mono alinhado à direita, cabeçalho em
mono 10px uppercase `#8B8478`. Barra de distribuição de 5–9px sem raio.

**API.** `GET /api/themes/{key}` (resumo, série, por tribunal, por órgão) e
`GET /api/themes/{key}/cases` (amostra auditável).

---

## Lacuna entre mockup e dado disponível

Estes blocos do detalhamento **não têm fonte hoje** e dependem de trabalho adicional:

| Bloco | Por quê | Fonte candidata |
|---|---|---|
| Citação de acórdão / inteiro teor | DataJud entrega metadado, não texto | repositório do TJSP/TJRJ/TJMG · PANGEA |
| Fundamentos invocados | exige NLP sobre o texto da decisão | depende do inteiro teor |
| Jurisprudência qualificada | precedentes com efeito vinculante | **removido do backlog** ([D-33](../06-operacao/02-decisoes-e-riscos.md#d-33--histórias-removidas-do-backlog-e-doutrina-relacionada)) |
| Doutrina relacionada | não é dado judicial | artigo com link, ligado ao tema por similaridade ([D-33](../06-operacao/02-decisoes-e-riscos.md#d-33--histórias-removidas-do-backlog-e-doutrina-relacionada)) |
| Valor fixado (quantum) | não é campo estruturado | depende do inteiro teor |
| Relator | não vem estruturado e consistente | **removido do backlog** ([D-33](../06-operacao/02-decisoes-e-riscos.md#d-33--histórias-removidas-do-backlog-e-doutrina-relacionada)) |

Nenhuma dessas fontes está confirmada. Antes de construir cada bloco, decida: **outra
fonte, extração por NLP, ou não entra na entrega.** Mockup não é promessa de dado.
Ver [Limitações da fonte](../03-dados/04-limitacoes-da-fonte.md).

## Duas coisas que faltam nos mockups

**1 · Declaração de escopo.** Os dados cobrem **TJSP, TJRJ e TJMG**. Um usuário que lê
"82% favorável" assumindo cobertura nacional tira conclusão errada. Precisa estar
visível — no cabeçalho do resultado, no rodapé do detalhamento, ou em ambos.

**2 · Proveniência por fonte.** O rodapé atual diz *"Fonte: DataJud/CNJ"*. Com o produto
[multifonte](../03-dados/01-fontes.md), cada bloco pode vir de origem diferente, e o
rodapé precisa refletir isso — quem cita precisa saber de onde veio cada número.

## Chatbot

Não aparece nos mockups e está no roadmap. Onde ele entra na interface é
[decisão em aberto](05-chatbot.md#pontos-em-aberto).
