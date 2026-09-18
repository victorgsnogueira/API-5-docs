# Polaridade do resultado

> **Leia antes de exibir qualquer percentual de "favorável" na tela.**
>
> Esta página nasceu de um erro encontrado na primeira carga real de dados. Não
> era hipótese: o produto **exibiria a conclusão invertida** para a maior área
> da base.

## O problema, em uma frase

> **"Favorável" não quer dizer nada sem dizer "favorável a quem".**

Os códigos da TPU não dizem quem ganhou. Dizem se a **pretensão foi acolhida** —
e quem pediu varia de processo para processo.

## Como isso apareceu

Na carga inicial (DataJud, TJSP + TJRJ), o tema *Tráfico e posse de drogas*
apareceu no topo com **98% "favorável"**.

São **condenações**. O autor da ação penal é o Ministério Público; "Procedência"
(código 219) significa que a pretensão **acusatória** foi acolhida.

Um advogado lendo "98% favorável" e assumindo "favorável ao réu" — que é a
leitura natural de quem defende — tira a conclusão exatamente contrária ao que
o dado diz. E matéria penal era a **maior área da carga**: 496 de 1.641
julgados.

## As três camadas do problema

### 1 · Quem é o autor muda o sentido do resultado

| Classe processual | Quem propõe | "Procedência" significa |
|---|---|---|
| Ação Penal | Ministério Público | **condenação** do réu |
| Execução Fiscal | Fazenda Pública | o contribuinte **paga** |
| Monitória, Busca e Apreensão, Despejo | credor | o devedor **perde** |
| Procedimento Comum, Juizado Especial | autor particular | o autor **ganha** |
| **Embargos à Execução, Habeas Corpus** | **a parte defensiva** | **o devedor/réu ganha** |

A última linha é uma **inversão**: em *Embargos à Execução* quem propõe é o
executado. Procedência ali favorece o devedor — direção oposta à de uma
*Execução Fiscal*, onde o autor é a Fazenda. Duas linhas do mesmo agregado
podem apontar para lados contrários.

### 2 · Duas polaridades diferentes não podem ser somadas

Este é um erro **metodológico**, não de rótulo:

| Família de códigos | A polaridade é relativa a |
|---|---|
| 219 / 220 / 221 (mérito) | quem **propôs** a ação |
| 237 / 238 / 239 (recurso) | quem **recorreu** |

Quem recorre costuma ser **quem perdeu embaixo**. Logo, um "provimento" aponta,
com frequência, na direção **oposta** de uma "procedência". Somar as duas
famílias numa única taxa produz um número que não significa nada.

### 3 · A palavra "favorável" é a origem do erro

Enquanto a coluna se chamar `favorable_count`, alguém vai renderizar
"X% favorável" sem qualificar. O nome do campo carrega uma promessa que o dado
não cumpre.

## A regra fechada

> 1. O schema **não usa a palavra "favorável"**. Usa **pretensão acolhida**
>    (`claim_upheld`) e **pretensão rejeitada** (`claim_rejected`).
> 2. Toda contagem de resultado vem acompanhada de **quem é o autor dominante**.
> 3. As famílias de mérito e de recurso **nunca são somadas**. São colunas
>    separadas.
> 4. Todo código de movimentação conferido **declara a que sua polaridade se
>    refere**. Código sem polaridade declarada não entra em métrica — mesma
>    lógica do [D-10](../06-operacao/02-decisoes-e-riscos.md#d-10--código-de-movimentação-não-conferido-não-entra-na-métrica).
> 5. A interface **nunca** exibe um percentual de resultado sem a frase de
>    polaridade junto.

## Como está implementado

| Onde | O quê |
|---|---|
| `dim_movement.polarity_reference` | `pretensao_autor` ou `pretensao_recorrente`. Constraint: código conferido sem polaridade não passa |
| `dim_case_class.claimant_type` | `acusacao`, `fazenda`, `credor`, `defesa`, `autor_particular` — derivado do nome da classe, revisável. `NULL` quando não dá para afirmar |
| `theme_summary.claim_upheld_count` / `claim_rejected_count` | mérito, separado |
| `theme_summary.appeal_upheld_count` / `appeal_rejected_count` | recurso, separado |
| `theme_summary.dominant_claimant` / `claimant_breakdown` | quem propõe os processos do tema |
| `theme_summary.claim_polarity_label` | **a frase pronta para a tela** |
| `theme_strength.agreement_basis` | qual família foi usada no cálculo da concordância |

### A frase que a tela deve usar

`claim_polarity_label` já vem pronta, para ninguém ter que inventar o texto:

| Autor dominante | Frase |
|---|---|
| `acusacao` | acolhimento da pretensão acusatória (procedência = condenação) |
| `fazenda` | acolhimento da pretensão da Fazenda Pública |
| `credor` | acolhimento da pretensão do credor/exequente |
| `defesa` | acolhimento da pretensão da parte defensiva (embargante/impetrante) |
| `autor_particular` | acolhimento da pretensão do autor |
| *(não identificado)* | acolhimento da pretensão de quem propôs (autor não identificado) |

## O que a correção revelou

Números que estavam escondidos dentro de um "% favorável" agregado:

| Autor dominante | Temas | Pretensão acolhida | Rejeitada |
|---|---:|---:|---:|
| autor particular | 193 | 877 | 312 |
| acusação | 57 | 526 | 15 |
| credor | 12 | 89 | 9 |
| **fazenda** | 7 | **7** | **19** |
| **defesa** | 6 | **16** | **20** |

A Fazenda Pública **perde mais do que ganha** nesta amostra (7 × 19). Antes da
correção, esse sinal estava somado com condenações criminais no mesmo
percentual — invisível.

## Testes que travam a regra

Em `scraping/sql/014_strength_link_tests.sql`, cada um devolve zero linhas
quando está certo:

- **12** — código conferido sem polaridade declarada;
- **13** — concordância calculada sobre famílias misturadas;
- **14** — tema com julgados mas sem rótulo de polaridade;
- **15** — `claimant_type` fora do vocabulário;
- **16** — **alguma coluna do schema voltou a ter a palavra "favor"**.

O teste 16 existe para impedir a reintrodução do problema.

## Limite conhecido

Para a família de **recurso**, o DataJud **não informa quem recorreu**. A
polaridade fica irreduzivelmente ambígua: sabemos que a pretensão do recorrente
foi ou não acolhida, mas não quem ele é. Por isso a família de recurso fica
separada e, na carga atual, representa 5 de 1.641 julgados.

Se o inteiro teor entrar no escopo, o polo recursal vira extraível. Enquanto
isso: separado e declarado, nunca somado.

---

Ver também: [Força do entendimento](../01-produto/04-forca-do-entendimento.md) ·
[Agregados OLAP](03-agregados-olap.md) ·
[Limitações da fonte](04-limitacoes-da-fonte.md)
