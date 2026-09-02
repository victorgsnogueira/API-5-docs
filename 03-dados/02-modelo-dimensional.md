# Modelo dimensional

> ## ⚠ Nada aqui está fechado
>
> A modelagem existente no `prototipo/` foi criada **sem auditoria alguma** e não serve
> de base. Esta página descreve o **método** e registra uma **proposta inicial** — que
> precisa ser auditada, discutida e provavelmente refeita antes de virar migration.
>
> O que é sólido aqui é a *disciplina* (definir grão, separar fato de dimensão, resolver
> N:N com ponte). O que é provisório é cada tabela e cada coluna.

## O método, antes das tabelas

Modelagem dimensional é uma sequência de quatro decisões, nesta ordem. Pular a ordem é
o erro clássico.

**1 · Escolher o processo de negócio.** O que queremos medir? Aqui: *como um tema
jurídico vem sendo decidido*.

**2 · Declarar o grão.** O que representa **uma linha** da tabela fato? É a decisão mais
cara de reverter — tudo depois depende dela.

**3 · Identificar as dimensões.** Por quais recortes se pergunta? Tribunal, órgão,
tempo, tema, classe…

**4 · Identificar os fatos (medidas).** O que se conta ou se soma?

## Decisão 2 — o grão: duas opções em aberto

Esta é **a** discussão que o time precisa ter antes de escrever qualquer DDL.

### Opção A · Grão = movimentação processual

Uma linha por evento dentro de um processo.

| Prós | Contras |
|---|---|
| É o menor evento que o DataJud entrega | volume de fato muito maior |
| Permite medir tempo entre etapas e taxa de recurso | a maioria das movimentações é irrelevante para o produto (despacho, conclusão) |
| O resultado do julgamento vive aqui (é uma movimentação) | exige uma camada de agregação para chegar a "resultado do processo" |

### Opção B · Grão = decisão / julgamento

Uma linha por evento de julgamento, filtrando as movimentações que representam
desfecho.

| Prós | Contras |
|---|---|
| Fato bem menor e alinhado à pergunta do produto | perde tempo entre etapas e taxa de recurso |
| Consulta mais direta | se a definição de "o que é julgamento" mudar, recarrega tudo |

### Como decidir

A pergunta a responder: **o produto vai medir tramitação (tempo, recursos) ou só
resultado?** Os mockups atuais só mostram resultado — o que apontaria para B. Mas
"tempo médio até a decisão" é uma métrica que advogado pede, e ela exige A.

Recomendação: **A**, com uma camada agregada por cima que produz o resultado vigente
por processo. É a opção que não fecha porta. Mas isso é recomendação, não decisão
tomada — registrar em [Decisões e riscos](../06-operacao/02-decisoes-e-riscos.md) quando
o time bater o martelo.

## Proposta inicial de esquema

**Provisório.** Nomes em inglês, porque o backend será em inglês
([ver convenção](../02-arquitetura/02-backend-dotnet.md#idioma)); os *valores* seguem em
português, porque o dado é do direito brasileiro.

```
                          dim_date
                              │
      dim_court ──────────┐   │   ┌────── dim_judging_body
                          ▼   ▼   ▼
    dim_case ────────> fact_case_event <──── dim_movement
         │                                        │
         │ bridge_case_topic                      └─ result_category
         ▼
     dim_topic
         ▲
    dim_class ──> (liga em dim_case)
```

### Tabela fato (proposta)

| Coluna | Papel |
|---|---|
| `case_sk`, `movement_sk`, `date_sk`, `court_sk`, `judging_body_sk` | chaves para as dimensões |
| `occurred_at` | timestamp do evento |
| chave natural | `(case_sk, movement_sk, occurred_at)` — para a carga ser idempotente |

Fato de eventos, sem medida numérica: a métrica é contagem, e o significado do evento
vem da dimensão de movimentação.

> **A ser auditado:** se o produto precisar exibir **valor da condenação**, o fato passa
> a ter medida numérica e o desenho muda. Isso depende de conseguirmos o inteiro teor —
> ver [Limitações](04-limitacoes-da-fonte.md).

### Dimensões (proposta)

| Tabela | Chave natural | Papel |
|---|---|---|
| `dim_court` | sigla | TJSP, TJRJ, TJMG (+ superiores, se entrarem no escopo) |
| `dim_topic` | a definir | **a entidade central do produto** — ver abaixo |
| `dim_class` | código TPU | classe processual |
| `dim_judging_body` | (tribunal, código) | vara/câmara/turma — responde à colegialidade |
| `dim_date` | data | ano, mês, trimestre |
| `dim_movement` | código TPU | traduz código em categoria de resultado |
| `dim_case` | número CNJ | atributos estáveis do processo |
| `bridge_case_topic` | (processo, tema) | resolve o N:N — um processo tem vários assuntos |

**A ponte não é opcional.** Um processo tem vários assuntos; colocar o tema como coluna
do fato duplicaria linhas e inflaria toda contagem por tema. Esse é um dos poucos
pontos que não precisam de auditoria — é erro conhecido de modelagem dimensional.

### `dim_topic` — a chave natural está em aberto

Se o tema for o assunto da TPU, a chave é o código. Se for um agrupamento semântico, a
chave é gerada e precisa de rastreabilidade até os códigos de origem. Ver
[O que é um tema](../01-produto/03-tema-modelo-conceitual.md).

Isso muda a tabela. Decidir antes de escrever a migration.

### `dim_movement.result_category` — o ponto mais sensível

O DataJud não publica resultado de julgamento como campo; ele só existe como código de
movimentação. Essa coluna é a tradução, e um código mal classificado corrompe
silenciosamente todo o favorável/desfavorável do produto.

Regra proposta: **só entra código cujo nome foi conferido contra a API.** Na dúvida,
categoria neutra — e categoria neutra não entra na métrica. Errar para menos é
recuperável; errar para mais, não.

## Multifonte: o que a modelagem precisa suportar

O produto consome [várias fontes](01-fontes.md). O modelo precisa absorver isso sem
reforma:

- **Proveniência.** Toda linha carregada deve saber de onde veio (DataJud, PANGEA,
  repositório do TJSP…) e quando. Sem isso, é impossível auditar divergência entre
  fontes ou reprocessar uma fonte só.
- **Chave de casamento.** O número CNJ é o candidato natural para ligar registros de
  fontes diferentes ao mesmo processo. Confirmar que todas as fontes o expõem.
- **Entidades que não são processo.** Precedente qualificado (súmula, tema repetitivo,
  IRDR) e doutrina não são processos e não cabem no fato de eventos processuais.
  Provavelmente pedem dimensões e pontes próprias — `dim_precedent`,
  `bridge_topic_precedent`, `dim_doctrine`, `bridge_topic_doctrine`.

Essa última é uma lacuna real da proposta atual: ela modela **processos**, e os mockups
pedem também **precedentes** e **doutrina** ligados ao tema.

## Checklist de auditoria — antes da primeira migration

- [ ] O grão está declarado por escrito e o time concorda?
- [ ] Cada dimensão tem chave natural clara e estável?
- [ ] Toda relação N:N passa por ponte?
- [ ] Há coluna de proveniência (fonte + data de carga) em tudo que é carregado?
- [ ] A carga é idempotente? Qual é a chave natural que garante isso?
- [ ] Precedentes e doutrina cabem no modelo, ou faltam entidades?
- [ ] O modelo responde a todas as perguntas das telas? (percorrer mockup a mockup)
- [ ] Existe alguma métrica que o modelo torna impossível de calcular?
- [ ] Dimensões mudam ao longo do tempo? Precisamos de historização (SCD)?
- [ ] Quem fora do time consegue ler o esquema e entender o domínio?

## Referências de método

Kimball, *The Data Warehouse Toolkit* — a fonte canônica para grão, star schema, tabela
de ponte e dimensões que mudam lentamente. Vale ler os capítulos 1 a 3 antes de fechar
o esquema; é literatura curta e evita retrabalho caro.
