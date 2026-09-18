# ETL e NLP

> Nada disso está implementado. O `prototipo/` tem um ETL funcional, mas ele
> [não é fonte de verdade](../05-prototipo/01-prototipo-referencia.md) — serve como
> catálogo de armadilhas conhecidas, não como código a portar.

## O desenho: um pipeline, vários conectores

O produto é [multifonte](../03-dados/01-fontes.md). O ETL precisa nascer assumindo isso:

```
   DataJud ─┐
   PANGEA  ─┤
   TJSP    ─┼──> [ mesma porta: ICaseSource ] ──> Transform ──> Load ──> DW
   TJRJ    ─┤                                                     │
   TJMG    ─┤                                            proveniência
   …       ─┘                                        (fonte + data de extração)
```

Acrescentar uma fonte é registrar uma implementação, não reescrever o pipeline.
Interface sugerida em [Backend .NET](02-backend-dotnet.md#multifonte-na-estrutura).

**Escopo:** TJSP, TJRJ e TJMG. Todo conector é escrito para esses três.

---

## Extract

### DataJud — o que já se sabe

A API é um Elasticsearch atrás de um proxy:

```
POST https://api-publica.datajud.cnj.jus.br/api_publica_{tribunal}/_search
Authorization: APIKey <chave pública do CNJ>
```

| Fato | Consequência no desenho |
|---|---|
| Um índice por tribunal, sem endpoint agregado | três chamadas, uma por tribunal |
| Teto de 100 documentos por página | paginação obrigatória |
| Chave **pública**, divulgada pelo CNJ no wiki | não é segredo, mas vai por variável de ambiente |
| Limite de taxa e falha intermitente | retry com backoff |
| Não é tempo real | batch; streaming não traria nada |

Documentação: <https://datajud-wiki.cnj.jus.br/api-publica/>

### Filtrar na origem

A carga é sempre um **recorte deliberado**. Mesmo com três tribunais, o volume total é
grande demais para "carregar tudo". O filtro (assuntos, período) vai no corpo da
consulta, não depois.

### Demais fontes

PANGEA, repositórios dos tribunais e doutrina ainda estão em investigação. Cada uma
precisa responder ao questionário em [Fontes](../03-dados/01-fontes.md#fontes-ainda-não-listadas)
antes de virar conector.

---

## Transform

### 1 · Achatar

O JSON das fontes é aninhado e inconsistente — no DataJud, um mesmo campo ora vem
objeto, ora string, ora ausente, variando entre tribunais. Cada conector produz um DTO
plano antes de qualquer coisa tocar o banco. Todo tratamento defensivo mora aí.

### 2 · Traduzir a TPU

Códigos numéricos viram nome legível e, no caso das movimentações, viram **categoria de
resultado** — que é de onde sai o favorável/desfavorável do produto.

O DataJud **não publica resultado de julgamento como campo**: ele só existe como
movimentação. Construir esse mapa é trabalho de auditoria, código a código, conferindo o
nome que a própria API devolve.

> **A regra mais importante do ETL:** código não conferido cai em categoria neutra, e
> categoria neutra **não entra na métrica**. Na dúvida, fora. Um código mal classificado
> corromperia toda a apuração de resultado do produto sem nenhum sintoma visível.

> **✅ Feito.** 6 códigos conferidos contra a tabela oficial da TPU/CNJ — não só
> contra o nome que a API devolve:
>
> | Código | Nome | Categoria | Polaridade |
> |---|---|---|---|
> | 219 | Procedência | acolhida | pretensão do autor |
> | 220 | Improcedência | rejeitada | pretensão do autor |
> | 221 | Procedência em Parte | parcial | pretensão do autor |
> | 237 | Provimento | acolhida | **pretensão do recorrente** |
> | 238 | Provimento em Parte | parcial | **pretensão do recorrente** |
> | 239 | Não-Provimento | rejeitada | **pretensão do recorrente** |
>
> Os outros **257 códigos** vistos na carga entraram como neutros. Contam para
> volume e série temporal; não contam para resultado.

🔴 **A verificação revelou algo que esta seção não previa:** os códigos têm
**polaridades diferentes** — 219/220/221 se referem a quem propôs, 237/238/239 a
quem recorreu. Somá-los é erro metodológico. Toda a discussão está em
[Polaridade do resultado](../03-dados/05-polaridade-do-resultado.md), e a
dimensão de movimentação ganhou uma coluna `polarity_reference` com constraint:
**código conferido sem polaridade declarada não entra no banco**.

### 3 · Reconciliar entre fontes

Novo, e não trivial. Quando duas fontes falam do mesmo processo:

- **chave de casamento** — o número CNJ é o candidato natural; confirmar que todas as
  fontes o expõem;
- **precedência** — se DataJud e repositório do tribunal divergem sobre a mesma data,
  qual vence? Definir e registrar;
- **complementaridade** — o caso comum não é conflito, é uma fonte trazendo campo que a
  outra não tem (metadado do DataJud + inteiro teor do tribunal);
- **proveniência por campo** — quando o dado de um registro vem de fontes diferentes,
  saber qual campo veio de onde deixa de ser luxo.

### 4 · Normalizar em tema

Ver [NLP](#nlp--onde-o-modelo-entra) e
[O que é um tema](../01-produto/03-tema-modelo-conceitual.md).

---

## Load

### Idempotência é requisito

Rodar a carga duas vezes com os mesmos dados **não** pode duplicar linha nem inflar
contagem. O agendamento reprocessa janelas que se sobrepõem, e um fato duplicado
apareceria na tela como decisão a mais.

Como se garante: upsert por chave natural nas dimensões, e chave natural bem escolhida
no fato. Qual é essa chave depende do
[grão](../03-dados/02-modelo-dimensional.md#decisão-2--o-grão-duas-opções-em-aberto),
que ainda está em discussão.

### Proveniência em toda linha

Fonte e data de extração. É o que permite auditar, reprocessar uma fonte só, e exibir no
rodapé da tela de onde veio o dado.

### Fechamento da carga

Atualizar os agregados, na ordem de dependência. Ver
[Agregados OLAP](../03-dados/03-agregados-olap.md).

---

## Agendamento

`Ratio.Etl` é console app (`OutputType=Exe`). No Coolify, isso vira **job agendado**.

**Não** transforme em `BackgroundService` dentro da API: acoplar a carga ao ciclo de
vida do servidor web impede rodar uma carga manual sem reiniciar a API, e faz a carga
competir com o tráfego.

Agendar em horário de baixo uso — o banco é compartilhado com a API na mesma VPS.

**Alarme obrigatório quando a carga falha ou não roda.** Uma carga que falha
silenciosamente por uma semana deixa o produto exibindo dado velho com aparência de dado
atual. Ver [DevOps](../06-operacao/03-devops-e-infra.md).

---

## NLP — onde o modelo entra

Três usos possíveis, em ordem de valor por custo. Nenhum implementado.

### Uso 1 · Agrupar assuntos em tema  *(maior valor, começar por aqui)*

> **✅ Implementado — 15/09/2026.** 447 assuntos da TPU → **408 temas**.

**Problema.** Os mockups mostram cinco teses distintas para uma mesma consulta. Um
código de assunto da TPU não separa isso.

**Abordagem implementada.** Embeddings locais (`paraphrase-multilingual-MiniLM-L12-v2`,
CPU, sem chave de API) gravados em `pgvector`; clusterização aglomerativa por
distância de cosseno; e o rótulo escrito na curadoria.

| Etapa | Resultado |
|---|---|
| Embeddings | 447 assuntos + 52.696 títulos de doutrina |
| Clusterização (`threshold` 0,20) | 69 clusters multi-assunto **candidatos** |
| Curadoria | **32 aceitos, 37 rejeitados** |
| Temas | 408 (32 de merge + 376 mantidos 1:1) |
| Área jurídica | 384 de 408 (94,1%) |

**Requisito cumprido.** `dim_topic` (assunto bruto) não é destruída: `dim_theme`
é uma camada **por cima**, ligada por `bridge_theme_topic`. Se o agrupamento for
descartado, o produto degrada para a granularidade da TPU em vez de parar.

#### A clusterização sozinha produz lixo — a curadoria não é formalidade

Erros reais observados, todos rejeitados na revisão:

- `Furto | Roubo | Ameaça | Liminar | Injúria | Desacato | Mútuo | Férias | Marca`
  — embedding de nome curto e genérico colapsa tudo numa lixeira;
- `DIREITO TRIBUTÁRIO | DIREITO PENAL | DIREITO PREVIDENCIÁRIO` — são marcadores
  de **área**, não temas;
- `Nota Promissória | Citação`, `Imissão | Mandato`, `Compromisso | Consórcio`
  — juridicamente distintos.

E acertos que justificam o esforço: `Indenização por Dano Moral` +
`Indenizaçao por Dano Moral` (erro de digitação na fonte),
`Repetição de indébito` + `Repetição do Indébito`, `Planos de Saúde` +
`Planos de saúde`.

**A curadoria é registrada por NOME de assunto, não por `cluster_id`** — id de
cluster muda a cada execução do algoritmo. Assim a decisão é reproduzível e
revisável por alguém que nunca rodou o modelo.

### Uso 2 · Extrair fundamentos, precedentes e valores do inteiro teor

**Problema.** Os blocos *Fundamentos invocados*, *Jurisprudência qualificada* e a coluna
de valor exigem o **texto** da decisão.

**Bloqueio.** O DataJud não entrega texto. A pergunta a responder **antes** de escrever
qualquer código é: *de onde vem o inteiro teor?* — repositório do TJSP/TJRJ/TJMG, PANGEA,
ou nada. Ver [Limitações](../03-dados/04-limitacoes-da-fonte.md).

### Uso 3 · Redigir o texto da aba Resumo

**Abordagem.** Geração a partir **exclusivamente** das tabelas da base analítica — o
próprio mockup declara isso no rodapé.

**Guarda-corpo obrigatório.** Todo número no texto tem de existir em uma linha do DW, e
toda decisão citada tem de existir na dimensão de processos. Um resumo que alucina um
percentual destrói a credibilidade do produto — o público-alvo **cita** o que lê aqui.

### Doutrina — regra de produto que vale para o NLP

Se e quando a doutrina for extraída de texto, o resultado é **referência**, nunca
arquivo:

- artigo → **link** para onde está publicado;
- livro → **autor, obra, edição, capítulo** em texto.

Nunca PDF de livro. Ver [Fontes](../03-dados/01-fontes.md#fonte-5--doutrina).

### Uso 4 · Ligar doutrina a tema  *(não estava previsto, e funcionou)*

> **✅ Implementado.** 9.186 ligações, cobrindo 7.242 dos 52.696 artigos.

[Fontes](../03-dados/01-fontes.md#fonte-5--doutrina) listava "de onde sai a
associação entre doutrina e tema" como pergunta sem resposta, presumindo que
dependeria do inteiro teor. **Não depende**: dá para associar semanticamente o
título do artigo ao assunto da TPU.

#### A lição: só embedding não serve

Primeira tentativa, com embedding puro: 87.383 ligações, 67,6% de cobertura.
Número bonito, resultado errado. Para *Indenização por Dano Moral*, o modelo
trouxe **filosofia moral** pontuando acima do instituto jurídico:

| Artigo | Similaridade |
|---|---:|
| A CALIBRAÇÃO DA MORAL PELO POSITIVISMO | 0,903 |
| MORAL E CONTEMPORANEIDADE | 0,901 |
| *Dano Moral: Aspectos Históricos e de Quantificação* (o correto) | *0,854* |

Falso-amigo semântico: **"moral" (filosofia) × "dano moral" (instituto)**. E como
o erro pontuava **mais alto** que o acerto, **subir o limiar não resolveria** —
foi preciso mudar o método, não o parâmetro.

**Solução híbrida:** exigir proximidade semântica **e** presença léxica dos
termos distintivos do assunto no título (≥2 termos quando o assunto tem 2+).
Rejeitou **176.709 candidatos falsos**; a cobertura caiu para 13,7%.

Essa queda é a **direção certa de errar**, pela mesma lógica do D-10: artigo sem
tema é recuperável, artigo no tema errado não é. Toda ligação grava
`similarity`, `link_method` e `embedding_model` — reauditável e apertável depois.

Resíduo conhecido: títulos com os dois termos mas assunto diferente ainda passam
(ex.: *"PROSTITUIÇÃO: O IMPASSE ENTRE O LEGALISMO MORAL E O PRINCÍPIO DO DANO"*).

### O princípio comum — e ao chatbot

> O modelo pode **rotular**, **agrupar** e **redigir**. Nunca pode **contar**.
> Todo número vem de `SELECT`.

Mesma regra vale para o [chatbot](../01-produto/05-chatbot.md), pelo mesmo motivo.

**Como isso está travado na prática:** nenhum número do produto passa pelo
modelo — `theme_summary`, `theme_strength` e companhia são `SELECT` sobre o
fato. E há um teste que **recalcula a contagem direto do fato e compara com o
agregado materializado**, falhando se divergirem. A regra deixou de ser
promessa e virou consulta que roda.
