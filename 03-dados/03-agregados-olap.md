# Agregados OLAP

> ⚠ **Proposta, não esquema fechado.** Depende diretamente do
> [modelo dimensional](02-modelo-dimensional.md), que ainda está em aberto. As views
> descritas aqui são um desenho plausível a auditar junto com ele.

## Por que agregar de antemão

A tela de tema abre com três agregações sobre a tabela fato. Rodar isso a cada request
deixa a página lenta — e o dado só muda quando o ETL roda, o que é uma vez por dia.

Alternativas:

| Estratégia | Quando serve |
|---|---|
| **View materializada** com `REFRESH` ao fim do ETL | padrão recomendado: simples, dentro do banco, versionável em migration |
| Tabela de agregado escrita pelo próprio ETL | quando o cálculo é caro demais para um `REFRESH` completo |
| Agregação em tempo de request, com cache | quando o volume é pequeno e a frescura importa |

**Consequência aceita** em qualquer uma das duas primeiras: o dado exibido é tão fresco
quanto a última carga. A interface declara a data de extração — os mockups já fazem
isso (*"extração de 28.08.2026"*).

## Cadeia proposta

```
fact_case_event + dim_movement
        │
        ▼
case_current_result        o desfecho VIGENTE de cada processo
        │
        ├──> topic_summary          selo de força e cabeçalho
        ├──> topic_by_year          série anual / alinhamento por ano
        ├──> topic_by_court         alinhamento por tribunal
        └──> topic_by_judging_body  colegialidade / divergência interna
```

A ordem do `REFRESH` importa: o resultado vigente primeiro; as demais dependem dele.

---

## `case_current_result` — o desfecho vigente

Um processo pode ter vários julgamentos (1ª instância, recurso, embargos). Qual deles é
"o resultado"?

**Proposta:** o **mais recente** — é o entendimento que está de pé hoje, que é o que
juiz e advogado precisam saber.

**A auditar:** essa escolha perde a história de reforma em grau de recurso. Um processo
julgado procedente em 1º grau e improcedente em 2º conta apenas como improcedente. É
uma perda real de informação, e "taxa de reforma em segundo grau" é justamente o tipo de
métrica que interessa ao público-alvo.

Alternativa a considerar: manter o resultado por **grau**, e deixar a tela escolher.
Isso muda o desenho da view e provavelmente das telas.

---

## `topic_summary` — resumo por tema

Alimenta o selo de força, o cabeçalho do detalhamento e a linha de metadados do
resultado de busca.

| Medida | Cálculo |
|---|---|
| processos | contagem distinta via ponte |
| tribunais | contagem distinta de tribunais |
| julgados | processos com resultado apurado |
| favorável / desfavorável | por categoria de resultado |
| ano inicial / final | mín. e máx. do ano do resultado |

**Decisão embutida a auditar:** *procedência em parte conta como favorável?*

A proposta diz que sim — do ponto de vista de quem pergunta "essa tese pega?",
acolhimento parcial é acolhimento. Mas isso perde nuance, e para o tema *quantum* a
diferença entre integral e parcial é exatamente o assunto.

Seja qual for a escolha, ela precisa estar **declarada no SQL**, não escondida na
camada de aplicação — é uma decisão de produto que alguém vai questionar.

---

## `topic_by_year`

`(tema, ano, favorável, desfavorável)`. Alimenta o bloco `ALINHAMENTO POR ANO` da coluna
lateral e a frase de tendência (*"69% → 84% no sentido predominante"*).

Ano sem resultado não deve virar barra vazia no gráfico.

---

## `topic_by_court`

`(tema, tribunal, favorável, desfavorável)`. Alimenta `ALINHAMENTO POR TRIBUNAL`, a
tabela *Comportamento por tribunal* e as contagens do painel de filtros.

Com o escopo em três tribunais, esta view é pequena — o que abre a possibilidade de
calculá-la em tempo de request se a materialização atrapalhar.

---

## `topic_by_judging_body`

`(tema, tribunal, órgão, favorável, desfavorável)`.

A pergunta de colegialidade: *"as câmaras do mesmo tribunal decidem igual entre si?"*

Regra de exibição proposta: só mostrar órgão com mais de um julgamento. Órgão com um
caso só é ruído de vara, não divergência.

---

## O que ainda falta desenhar

As views acima cobrem **processos**. Os mockups também pedem agregação sobre entidades
que o modelo atual nem tem:

| Bloco de tela | Agregado necessário | Depende de |
|---|---|---|
| *Fundamentos invocados* (frequência × taxa de acolhimento) | fundamento × tema × resultado | inteiro teor + NLP |
| *Jurisprudência qualificada* (citado em, seguido) | precedente × tema × aderência | PANGEA ou equivalente |
| *Doutrina invocada* (citações) | doutrina × tema | fonte de doutrina |
| Mediana de valor, P25/P75 | medida numérica no fato | inteiro teor |

Nenhum deles é difícil de agregar; todos dependem de dado que ainda não temos. Ver
[Fontes](01-fontes.md) e [Limitações](04-limitacoes-da-fonte.md).

Note que **mediana e percentil** não são agregações triviais em view materializada
incremental — `percentile_cont` exige o conjunto ordenado inteiro. Se o valor entrar no
escopo, vale pensar nisso cedo.

---

## Como cada tela consome

| Tela / bloco | Agregado |
|---|---|
| Busca — lista de temas | resumo por tema |
| Resultado — nota, volume, período, % favorável | resumo por tema |
| Filtro — contagem por tribunal | por tribunal |
| Detalhe — cabeçalho e selo | resumo por tema |
| Detalhe — alinhamento por ano, `FIG. 1` | por ano |
| Detalhe — alinhamento por tribunal | por tribunal |
| Detalhe — divergência interna | por órgão |
| Detalhe — *Amostra auditável* | consulta ao fato + resultado vigente |
| [Chatbot](../01-produto/05-chatbot.md) | os mesmos agregados, via ferramentas expostas ao modelo |

O chatbot é um consumidor de primeira classe destes agregados — não um caminho
alternativo até o banco. Isso é o que garante que ele responda os mesmos números que a
tela mostra.

---

## Pontos de atenção

- **`REFRESH` sem `CONCURRENTLY` trava leitura.** Para usar `CONCURRENTLY`, cada view
  precisa de índice único. Vale prever isso desde a primeira migration.
- **Agregado não filtra escopo.** O recorte (quais tribunais, quais assuntos) é
  responsabilidade do ETL, não da view.
- **Entre a gravação e o `REFRESH`, a tela mostra o estado anterior.** Isso é aceitável
  — dado consistente e antigo é melhor que dado meio atualizado.
- **Todo agregado precisa de teste de consistência** com a tabela fato. É requisito
  explícito do desafio; ver [DevOps](../06-operacao/03-devops-e-infra.md).
