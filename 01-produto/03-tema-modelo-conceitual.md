# O que é um "tema"

**Tema é a entidade central do produto.** O usuário busca tema, o resultado é tema, o
detalhamento é de um tema. Processo, precedente, jurisprudência e doutrina são o que
*sustenta* o tema — nunca o que se entrega na busca.

## A definição

> Um tema é um agrupamento de processos, precedentes, jurisprudências e doutrinas de
> tribunais diversos que tratam da **mesma questão jurídica**, normalizados sob um
> único rótulo.

Exemplo canônico:

```
processo A · TJSP · "atraso de voo superior a dez horas"     ┐
processo B · TJRJ · "atraso de voo · pernoite em aeroporto"  ├─> tema: ATRASO DE VOO
processo C · TJMG · "transporte aéreo · atraso · dano moral" ┘
```

Escopo do produto: **TJSP, TJRJ e TJMG**.

Sem essa normalização, o usuário teria de saber de antemão como cada tribunal
escreveu o assunto — que é exatamente o problema que ele veio resolver.

## Duas implementações possíveis

### A · Tema = assunto da TPU  *(o caminho determinístico)*

O DataJud já traz cada processo classificado por códigos de **assunto** da Tabela
Processual Unificada do CNJ. Adotar esse código como tema é gratuito, determinístico e
auditável.

A chave é o código do assunto, e a relação com processo é N:N (um processo tem vários
assuntos) — logo, tabela-ponte.

| Prós | Contras |
|---|---|
| Zero custo de inferência | granularidade é a do CNJ, não a da pergunta do usuário |
| Determinístico e reproduzível | um assunto da TPU pode conter várias teses distintas |
| Vocabulário oficial, defensável | não captura variação de redação nem tese emergente |

O contra é concreto e visível nos mockups: a tela de resultados mostra **cinco teses
diferentes** para a mesma consulta — *dano moral in re ipsa*, *Súmula 385*,
*quantum indenizatório*… Um único código de assunto da TPU não separa isso.

### B · Tema = agrupamento semântico  *(o que os mockups pressupõem)*

Agrupar por similaridade semântica do texto (assunto + classe + ementa, quando
houver), usando embeddings ou uma LLM para rotular o cluster.

| Prós | Contras |
|---|---|
| Granularidade igual à da pergunta real | não determinístico: a mesma carga pode gerar temas diferentes |
| Captura tese emergente | rótulo gerado por modelo precisa de curadoria |
| Separa teses dentro de um mesmo assunto | custo de inferência e latência no ETL |

## ✅ Implementado: A com B por cima (15/09/2026)

A recomendação abaixo foi seguida à risca. Resultado, na recarga cível de
20/09/2026: **1.075 assuntos da TPU → 1.049 temas**, dos quais 23 vieram de
agrupamento semântico (consolidando 49 assuntos) e 1.026 foram mantidos 1:1.

`dim_topic` (assunto bruto) **não foi destruída** — `dim_theme` é uma camada
acima, ligada por `bridge_theme_topic`. A pergunta "de onde saiu esse tema?" tem
resposta em SQL, e há teste que falha se algum tema perder o lastro.

Detalhes do método, dos erros do algoritmo e da curadoria:
[ETL e NLP](../02-arquitetura/05-etl-e-nlp.md#uso-1--agrupar-assuntos-em-tema--maior-valor-começar-por-aqui).

⚠ **O contra da Opção A continua valendo.** Os mockups mostram cinco teses para
uma consulta; o agrupamento por embedding junta variação de **redação**, não
separa **teses** dentro de um mesmo assunto. Para isso seria preciso a ementa —
que depende do inteiro teor, hoje sem fonte.

## Recomendação (seguida)

**Fazer A primeiro, B em cima de A** — e nunca substituir um pelo outro:

```
assunto TPU (determinístico, do CNJ)   ← lastro, sempre presente
        │
        │  camada de agrupamento semântico
        ▼
tema (rótulo do produto)               ← o que aparece na tela
```

Assim o tema **sempre** aponta para códigos de TPU reais. A pergunta "de onde saiu
esse tema?" tem resposta: destes N assuntos, destes M processos. Se o agrupamento
semântico falhar ou for descartado, o produto degrada para a granularidade da TPU em
vez de parar de funcionar.

Discussão da camada semântica: [ETL e NLP](../02-arquitetura/05-etl-e-nlp.md).

## Atributos de um tema

Levantado a partir das telas. Nada disso está implementado — nomes em inglês, porque
o [backend é em inglês](../02-arquitetura/02-backend-dotnet.md#idioma):

| Campo | Origem | Uso na tela | Estado |
|---|---|---|---|
| `code` | chave do tema | identificador na URL | ✅ |
| `name` | rótulo (**valor em português**) | título | ✅ 1.049 temas |
| `subjectArea` | classificação de matéria | tag CONSUMIDOR / BANCÁRIO / QUANTUM | ✅ 94,1% |
| `summary` | prosa curta, gerada e curada | as duas linhas do resultado | 🔴 NLP Uso 3, não feito |
| `caseCount` | contagem distinta via ponte | "12.418 processos" | ✅ |
| `judgedCount` | processos com resultado apurado | denominador de toda métrica | ✅ |
| `courtCount` | contagem distinta | "3 tribunais" e componente de cobertura | ✅ |
| ~~`granted` / `denied`~~ → **`claimUpheld` / `claimRejected`** | agregado por tema | barra de alinhamento | ✅ **renomeado** |
| **`claimPolarityLabel`** | autor dominante da classe | **obrigatório junto do percentual** | ✅ **campo novo** |
| `periodStart` / `periodEnd` | ano mín./máx. | "2021 — 2026" | ✅ |
| `lastDecisionDate` | máx. da data de decisão | "última decisão 21.08.2026" | ✅ |
| `strengthScore` | calculado | [nota /100](04-forca-do-entendimento.md) | ✅ |
| `provenance` | fonte + data de extração | rodapé — o produto é multifonte | ✅ |
| `relevance` | ranking da busca | ordenação (não exibido) | 🔴 |

**`claimPolarityLabel` é campo novo e não opcional.** Sem ele a barra de
alinhamento diz "98% favorável" para um tema penal em que isso significa
condenação. Ver [Polaridade do resultado](../03-dados/05-polaridade-do-resultado.md).

## Regras invioláveis

1. **Todo tema aponta para processos reais.** Um tema sem lastro no fato não existe.
2. **Tema sem julgamento não aparece na busca.** Um tema sem desfecho apurado não
   responde à pergunta do usuário.
3. **O rótulo do tema é auditável.** Se foi gerado por modelo, isso é registrado e
   os assuntos de origem ficam disponíveis.
5. **Todo tema carrega proveniência.** De que fontes vieram os processos que o
   sustentam, e quando foram extraídos.
4. **Ponte, nunca coluna.** A relação processo↔tema é N:N. Colocar `tema_sk` no fato
   duplicaria linhas e inflaria toda contagem.
