# Agregados OLAP

> ## Atualização de 20/09/2026 — [D-35](../06-operacao/02-decisoes-e-riscos.md#d-35--o-banco-do-cliente-é-o-dw-um-só-modelo-dw-nos-três-bancos)
>
> Os agregados no grão de **assunto** (`topic_summary`, `topic_by_year`, `topic_by_court`,
> `topic_by_judging_body`) **foram removidos**: nenhuma rota nem história os lê. O que a tela consome
> são os agregados no grão de **tema**, e ganharam dois: `theme_by_judging_body` (câmara por tema) e
> `theme_time_to_decision` (tempo até a decisão). `data_provenance` virou agregado materializado.
> As seções abaixo sobre `topic_*` ficam como registro do desenho original; as definições vigentes,
> coluna por coluna, estão em [Modelagem dos três bancos](06-modelagem-dos-bancos.md#45--agregados).

> ## ✅ Implementados — 15/09/2026
>
> A cadeia proposta aqui foi construída e roda sobre a carga real. Mudou uma
> coisa importante no contrato das colunas:
>
> | Antes (proposta) | Agora (implementado) |
> |---|---|
> | `favorable_count` | `claim_upheld_count` |
> | `unfavorable_count` | `claim_rejected_count` |
> | mérito e recurso somados | colunas separadas: `claim_*` e `appeal_*` |
> | — | `dominant_claimant`, `claimant_breakdown`, `claim_polarity_label` |
>
> **A palavra "favorável" saiu do schema**, e há um teste que falha se ela
> voltar. Motivo em [Polaridade do resultado](05-polaridade-do-resultado.md):
> "favorável" sem dizer a quem faz o produto exibir a conclusão invertida em
> matéria penal.

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
        ├──> topic_summary          selo de força e cabeçalho              (removido na D-35)
        ├──> topic_by_year          série anual / alinhamento por ano       (removido na D-35)
        ├──> topic_by_court         alinhamento por tribunal                (removido na D-35)
        └──> topic_by_judging_body  colegialidade / divergência interna     (removido na D-35)
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

## Agregados no grão de TEMA — novos

A cadeia original parava no assunto. Com a
[camada semântica](../02-arquitetura/05-etl-e-nlp.md#uso-1--agrupar-assuntos-em-tema--maior-valor-começar-por-aqui),
existe um nível acima, que é o que a tela consome:

```
case_current_result
      └──> theme_summary · theme_by_year · theme_by_court · theme_by_judging_body
           theme_time_to_decision · theme_strength
data_provenance   (lê o fato e a doutrina)
```

`theme_strength` é a [nota de força](../01-produto/04-forca-do-entendimento.md),
com os componentes e a polaridade abertos.

## O que ainda falta desenhar

| Bloco de tela | Agregado necessário | Depende de | Estado |
|---|---|---|---|
| *Doutrina relacionada* | doutrina × tema | fonte de doutrina | 🟢 **feito** — 13.870 ligações, por similaridade ([D-33](../06-operacao/02-decisoes-e-riscos.md#d-33--histórias-removidas-do-backlog-e-doutrina-relacionada)) |
| *Fundamentos invocados* | fundamento × tema × resultado | inteiro teor + NLP | 🔴 |
| Mediana de valor, P25/P75 | medida numérica no fato | inteiro teor | 🔴 |

Os dois que faltam dependem do **inteiro teor**, que é a lacuna estrutural que
sobrou depois da investigação de fontes. Ver
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
