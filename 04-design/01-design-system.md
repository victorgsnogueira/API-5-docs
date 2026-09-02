# Design system

## Conceito

**Impresso acadêmico-jurídico.** Papel envelhecido, serifa para texto longo,
monoespaçada para todo dado auditável, um único vermelho para o que exige ação ou
conferência.

> **Artigo com dados — não dashboard.**

## Princípios

**1 · Artigo, não dashboard.** O dado sustenta a prosa. Figuras entram no fluxo do
texto, numeradas (`FIG. 1`, `FIG. 2`), com fonte declarada. **Nunca grade de cartões.**

**2 · Todo número é rastreável.** Sem rótulos inventados — nada de "T-0431" ou
"consolidada" solta. Só processo, órgão, relator, data, *n* e fonte.

**3 · Vermelho é sinal, não decoração.** Reservado a ação, link, valor predominante e
ao que o leitor precisa conferir.

---

## Tipografia

### Famílias

| Família | Pesos | Uso |
|---|---|---|
| **Source Serif 4** | 400, 600, 700, 400 itálico | títulos, texto corrido, citação de acórdão (itálico) |
| **IBM Plex Mono** | 400, 500, 600 | números, siglas de tribunal, nº de processo, datas, rótulos, metadados, botões, abas |

A divisão não é estética: **serifa é prosa, mono é dado auditável.** Se o leitor pode
precisar conferir, está em mono.

### Escala

| Papel | Família | Tamanho / peso | Tratamento |
|---|---|---|---|
| Logotipo | Serif | 64px (busca) / 25px (barra) · 700 | tracking −.015em, "i" itálico vermelho |
| Título de tese (detalhe) | Serif | 34px · 700 · lh 1.18 | tracking −.014em, `text-wrap: pretty` |
| Título em lista | Serif | 24px · 700 · lh 1.24 | medida máx. 640px |
| Lead do artigo | Serif | 20px · 400 · lh 1.65 | filete vermelho 2px à esquerda, recuo 18px |
| Corpo do artigo | Serif | 17px · 400 · lh 1.75 | coluna de 690px, cor `#241F1A` |
| Citação de acórdão | Serif itálico | 17.5px · 400 · lh 1.6 | fundo `#FBF9F4`, filete vermelho 3px |
| Célula de tabela | Serif / Mono | 13.5px / 13px · 400 | texto em serifa; número em mono, alinhado à direita |
| Cabeçalho de tabela e rótulos | Mono | 10px · 400 · uppercase | tracking .10–.16em, cor `#8B8478` |
| Botão / aba | Mono | 10.5–12px · 400 · uppercase | tracking .12–.14em |
| Legenda e fonte | Mono ou serif itálico | 10.5–12.5px · 400 | cor `#8B8478`, sempre **sob** a figura |
| Marcador de citação `[1]` | Mono | 11px · `vertical-align: super` | cor `#A3121A`, clicável |

---

## Cores

### Papel e superfícies

| Token | Hex | Uso |
|---|---|---|
| Papel | `#F4F1EA` | fundo de todas as telas |
| Superfície | `#FFFFFF` | barra superior, tabelas, blocos |
| Realce quente | `#FBF9F4` | citação de acórdão, chip de sugestão |
| Hover de linha | `#FBF6F5` | linha de resultado sob o cursor |

### Tinta

| Token | Hex | Uso |
|---|---|---|
| Tinta | `#14120F` | títulos, filete de seção, série escura em gráficos |
| Corpo | `#241F1A` | corpo do artigo |
| Secundário | `#3A342C` | texto secundário, resumos de lista |
| Apoio | `#6E675C` | apoio, natureza do fundamento |
| Metadado | `#8B8478` | metadado, legenda, rótulo uppercase |

### Bordas

| Token | Hex | Uso |
|---|---|---|
| Média | `#C9C2B4` | campo de busca, divisórias fortes |
| Bloco | `#DCD6C9` | borda de bloco e de lista |
| Interna | `#EBE6DB` | trilha de barra, divisória interna de tabela |

### Vermelho de sinal

| Token | Hex | Uso |
|---|---|---|
| Sinal | `#A3121A` | ação, link, valor predominante, score forte |
| Hover | `#7B0D14` | hover do vermelho |
| Intermediário | `#C97C7F` | série intermediária em gráfico |
| Antigo | `#E4CFCF` | série mais antiga / faixa interquartil |

### Escala de dados

Rampa única de quatro passos:

```
#E4CFCF  ->  #C97C7F  ->  #14120F  ->  #A3121A
 antigo      interm.       escuro      recente/predominante
```

O vermelho cheio marca **sempre** o valor mais recente ou predominante.

### Proibido

Gradiente · sombra · verde/amarelo de semáforo · um segundo vermelho · emoji · ícone
decorativo.

---

## Forma

| Propriedade | Regra |
|---|---|
| **Raio** | 3px em botões, chips, tags, campo de busca e blocos. Círculo perfeito só no score |
| **Borda** | 1px sempre. **Nenhuma sombra em nenhum estado** |
| **Filete de seção** | 1px `#14120F` sob o título; 1px `#DCD6C9` entre itens de lista |
| **Aba ativa** | sublinhado 2px `#A3121A`, texto `#14120F`. Inativa: `#8B8478`, borda transparente |
| **Barra de dados** | altura 5–9px, trilha `#EBE6DB`, **sem raio** |
| **Score** | círculo 70px (lista) / 104px (detalhe), borda 1px, número em mono 27px/42px 600, `/100` abaixo em 9–10px |

---

## Tokens CSS

```css
:root {
  /* papel e superfícies */
  --papel:        #F4F1EA;
  --superficie:   #FFFFFF;
  --realce:       #FBF9F4;
  --hover-linha:  #FBF6F5;

  /* tinta */
  --tinta:        #14120F;
  --corpo:        #241F1A;
  --secundario:   #3A342C;
  --apoio:        #6E675C;
  --metadado:     #8B8478;

  /* bordas */
  --borda-media:  #C9C2B4;
  --borda-bloco:  #DCD6C9;
  --borda-interna:#EBE6DB;

  /* vermelho de sinal */
  --sinal:        #A3121A;
  --sinal-hover:  #7B0D14;
  --serie-media:  #C97C7F;
  --serie-antiga: #E4CFCF;

  /* tipografia */
  --serif: "Source Serif 4", Georgia, "Times New Roman", serif;
  --mono:  "IBM Plex Mono", ui-monospace, SFMono-Regular, Consolas, monospace;

  /* forma */
  --raio: 3px;
  --coluna-artigo: 690px;
  --medida-titulo: 640px;
}
```

---

## Componentes

### Score

```
   ╭─────╮
  │  77   │   círculo, borda 1px
  │  /100 │   número: mono 27px (lista) / 42px (detalhe), peso 600
   ╰─────╯    "/100": 9–10px, cor metadado
```

Cor do número acompanha a força: vermelho `#A3121A` em nota alta, tinta `#14120F` nas
demais. No detalhe, vem com o rótulo `FORÇA DO ENTENDIMENTO` abaixo, mono uppercase.

### Tag de matéria

Retângulo preenchido `#14120F`, texto branco, mono 10px uppercase, tracking .12em,
raio 3px. Ex.: `CONSUMIDOR`, `BANCÁRIO`, `QUANTUM`.

### Tag de efeito (jurisprudência qualificada)

| Efeito | Tratamento |
|---|---|
| `Obrigatório` | preenchida `#14120F`, texto branco |
| `Vinculante de fato`, `Regional`, `Persuasivo` | contorno 1px `#A3121A`, texto vermelho, fundo transparente |

O preenchimento sólido é reservado ao que **vincula de direito** — a hierarquia visual
espelha a hierarquia jurídica.

### Chip de tribunal

Contorno 1px `#DCD6C9`, fundo branco, mono 11px, raio 3px. Excedente vira `+N`.

### Barra de alinhamento

Trilha `#EBE6DB`, altura 5px, sem raio. A fração predominante em `#A3121A`; a
complementar fica na trilha. Percentual à direita, em mono.

### Figura

Bloco no fluxo do texto, largura da coluna do artigo:

```
FIG. 1 — DESFECHO DAS 12.418 DECISÕES                    2021–2026
─────────────────────────────────────────────────────────────────
Procedente                                     7.699 ·  62,0%
████████████████████████████████░░░░░░░░░░░░░░░
...
Fonte: DataJud/CNJ · 12.418 acórdãos de 2º grau e instâncias superiores
```

Rótulo em mono uppercase à esquerda, escopo à direita, filete, conteúdo, e **a fonte
sempre embaixo**. Sem fonte declarada, a figura não vai ao ar.

### Tabela

- cabeçalho: mono 10px uppercase, `#8B8478`, divisória 1px `#EBE6DB` abaixo;
- texto em serifa 13.5px; número em **mono 13px alinhado à direita**;
- sem zebra, sem borda vertical;
- valores que exigem conferência (percentual predominante, número de processo) em
  vermelho.

### Botões

| Tipo | Tratamento |
|---|---|
| Primário | fundo `#A3121A`, texto branco, mono uppercase (`BUSCAR`, `COPIAR CITAÇÃO`, `APLICAR`) |
| Secundário | contorno 1px, fundo branco (`EXPORTAR CSV`, `BASE ANALÍTICA`) |
| Link | texto vermelho, sem borda (`limpar filtros`, `ver todos os 5`) |

---

## Checklist de revisão visual

Antes de dar uma tela por pronta:

- [ ] nenhuma sombra, nenhum gradiente;
- [ ] um único vermelho, e ele significa ação / link / predominante;
- [ ] todo número em mono, todo texto corrido em serifa;
- [ ] toda figura tem rótulo numerado e fonte declarada;
- [ ] todo número no texto tem o *n* ao lado;
- [ ] nenhum rótulo que não venha do dado;
- [ ] tabela larga rola dentro do próprio contêiner, não na página;
- [ ] o corpo do artigo não passa de 690px de medida.
