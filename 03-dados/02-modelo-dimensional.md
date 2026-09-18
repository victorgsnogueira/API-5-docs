# Modelo dimensional

> ## ✅ Esquema implementado e carregado — 15/09/2026
>
> Deixou de ser proposta. Existe um esquema rodando com **463.016 linhas de
> fato**, e as decisões que esta página listava como em aberto foram tomadas:
>
> | Decisão | Status |
> |---|---|
> | **Grão do fato** | ✅ **movimentação processual** (Opção A) — o DataJud entrega o array `movimentos`, que é a fonte de eventos que a Opção A pressupunha |
> | Chave natural do fato | ✅ `(processo, movimento, timestamp)` — carga idempotente verificada rodando o mesmo lote duas vezes |
> | `dim_topic` | ✅ assunto da TPU (Opção A), **mais** uma camada de tema semântico por cima (`dim_theme`) |
> | `dim_movement.result_category` | ✅ 6 códigos conferidos contra a TPU/CNJ; os outros 257 entram como neutros |
> | Precedentes e doutrina no modelo | ✅ `dim_doctrine` + `bridge_topic_doctrine` implementadas. Precedente segue sem fonte |
> | Proveniência | ✅ `source` + `source_url` + `extracted_at` em toda linha carregada |
>
> **Nova exigência descoberta na carga:**
> [polaridade do resultado](05-polaridade-do-resultado.md) — `dim_movement`
> e `dim_case_class` ganharam colunas que esta página não previa.
>
> A implementação está em `scraping/sql/` (spike), não no `Ratio.Etl` oficial em
> .NET. O esquema é o mesmo; o host é que ainda vai ser portado.
>
> A modelagem do `prototipo/` continua **não servindo de base** — nada dela foi
> aproveitado.

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

### ✅ Decidido: Opção A (movimentação), com agregado por cima

A recomendação foi seguida. O que confirmou a escolha, na prática:

- o DataJud entrega o array `movimentos` por processo — a fonte de eventos que a
  Opção A pressupõe **existe**;
- média de **43,8 movimentos por processo**: 18.378 processos renderam 463.016
  linhas de fato. Volume alto, como o contra previa, mas Postgres absorve sem
  esforço;
- a camada `case_current_result` resolve o "resultado vigente por processo", e é
  sobre ela que todos os agregados de tema rodam.

Houve um período em que a Opção B (grão = decisão) foi adotada, enquanto o
DataJud estava fora do escopo e a única fonte possível seria um repositório de
jurisprudência. A tabela `fact_case_decision` desse desenho **continua existindo
e vazia**, pronta para quando houver inteiro teor. Não são versões concorrentes:
são grãos diferentes para fontes diferentes.

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

## Esquema como implementado

```
                    dim_date
                        │
  dim_court ────────┐   │   ┌────── dim_judging_body
                    ▼   ▼   ▼
 dim_case ──────> fact_case_event <────── dim_movement
    │  │                                      │
    │  │ bridge_case_topic                    ├─ outcome_sk (dim_decision_outcome)
    │  ▼                                      └─ polarity_reference  ← novo
    │ dim_topic (assunto TPU, 447)
    │     │  bridge_theme_topic        ┌── dim_doctrine (52.696)
    │     ▼                            │      ▲
    │  dim_theme (408) ────────────────┘  bridge_topic_doctrine
    │
    └─ dim_case_class ─ claimant_type  ← novo
```

Agregados por cima: `case_current_result` → `theme_summary` · `theme_by_year` ·
`theme_by_court` · `theme_strength` (+ equivalentes no grão de assunto).

## Checklist de auditoria — respondido

- [x] **O grão está declarado por escrito?** Sim: uma linha = uma movimentação.
- [x] **Cada dimensão tem chave natural clara e estável?** Sim — número CNJ,
      código da TPU, sigla do tribunal, nome do assunto, DOI/URL do artigo.
- [x] **Toda relação N:N passa por ponte?** Sim: `bridge_case_topic`,
      `bridge_theme_topic`, `bridge_topic_doctrine`.
- [x] **Proveniência em tudo que é carregado?** Sim, e há teste que falha se
      faltar.
- [x] **A carga é idempotente?** Sim — `natural_key` no fato e
      `UNIQUE(source, payload_hash)` no raw. Verificado reprocessando o mesmo lote.
- [x] **Precedentes e doutrina cabem?** Doutrina sim. **Precedente qualificado
      continua sem entidade e sem fonte** (dependia do PANGEA).
- [ ] **O modelo responde a todas as perguntas das telas?** Não: falta tudo que
      depende de inteiro teor (fundamentos, citação de acórdão, valor, relator).
- [x] **Alguma métrica ficou impossível de calcular?** Sim, e é consequência do
      grão: **tempo entre etapas e taxa de recurso** são calculáveis (o grão de
      evento permite), mas **valor da condenação** não — o fato não tem medida
      numérica, e adicioná-la depende de fonte que não existe.
- [ ] **Dimensões mudam ao longo do tempo? Precisa de SCD?** Ainda não tratado.
      Órgão julgador é renomeado, assunto da TPU é revisado pelo CNJ. Hoje o
      upsert **sobrescreve**, sem histórico. Risco conhecido, não endereçado.
- [x] **Quem fora do time lê o esquema e entende o domínio?** Os nomes seguem o
      [vocabulário PT→EN](../02-arquitetura/02-backend-dotnet.md#idioma), e as
      migrations têm comentário explicando cada decisão não óbvia.

### O que a carga mostrou que o checklist não perguntava

- **Completude do dado varia por tribunal** — o TJMG entrega `dataHora` nulo em
  100% dos movimentos. Um contrato de API igual não garante dado igual.
- **Resultado precisa de polaridade** — saber que a pretensão foi acolhida não
  basta; é preciso saber **de quem**. Ver
  [Polaridade do resultado](05-polaridade-do-resultado.md).

Ambos viraram colunas e constraints, não observações soltas.

## Referências de método

Kimball, *The Data Warehouse Toolkit* — a fonte canônica para grão, star schema, tabela
de ponte e dimensões que mudam lentamente. Vale ler os capítulos 1 a 3 antes de fechar
o esquema; é literatura curta e evita retrabalho caro.
