# Limitações da fonte

**Leia esta página antes de prometer qualquer coisa para o cliente ou para a banca.**

O produto é [multifonte](01-fontes.md): parte do que falta no DataJud pode vir de
PANGEA ou dos repositórios dos tribunais. Mas **nenhuma dessas fontes está confirmada**,
e enquanto não estiver, o que vale é a lacuna.

Ao construir cada bloco de tela, a decisão precisa ser consciente: *outra fonte, NLP, ou
não entra na entrega.*

---

## O que o DataJud NÃO entrega

### 1 · Texto integral da decisão

O DataJud é **metadado processual**: capa do processo e movimentações. Não há ementa,
não há inteiro teor, não há um parágrafo sequer da decisão.

**Bloqueia:**
- o bloco de citação de acórdão da aba Resumo (fundo bege, filete vermelho);
- o botão `Inteiro teor · PDF`;
- os blocos *Fundamentos invocados* e *Jurisprudência qualificada*;
- qualquer extração por NLP sobre o conteúdo da decisão.

**Saídas possíveis:** integrar os repositórios de jurisprudência do TJSP, TJRJ e TJMG
(sem API padronizada), investigar se o [PANGEA](01-fontes.md#fonte-2--pangea--pdpj)
entrega inteiro teor, ou remover esses blocos do escopo.

### 2 · Doutrina acadêmica

**Nenhuma API pública de tribunal publica doutrina.** É material protegido por direito
autoral, e não é dado judicial.

**Bloqueia:** o bloco *Doutrina invocada* — Cavalieri Filho, Tartuce, Marques,
Schreiber, com posição (majoritária/intermediária/minoritária) e contagem de citações.

**Regra de produto já fechada:** a doutrina nunca é servida como PDF de livro. A
referência é o **artigo, por link** para onde está publicado, ou o **nome da obra** em
texto (autor, título, edição, capítulo). Ver
[Fontes · Doutrina](01-fontes.md#fonte-5--doutrina).

Isso resolve o lado autoral, mas **não** resolve de onde sai a associação entre doutrina
e tema — que continua sem fonte.

### 3 · Relator

O nome do relator não vem de forma estruturada e consistente entre tribunais.

**Bloqueia:** a coluna `RELATOR` da amostra auditável, a referência completa da citação
de acórdão, e qualquer análise de *win rate* por relator.

### 4 · Valor da condenação (quantum)

Não é campo estruturado. O valor fixado está no texto da decisão, quando está.

**Bloqueia:** a coluna `MEDIANA R$` da tabela por tribunal, a `FIG. 2 — VALOR FIXADO`
com P25/mediana/P75, a coluna `VALOR` da amostra auditável, e o tema *Quantum
indenizatório* que aparece nos resultados de busca.

Note que isso também tem impacto de modelagem: o fato hoje **não tem coluna de medida
numérica**. Adicionar valor muda o desenho.

### 5 · Resultado do julgamento como campo

O desfecho existe **apenas** como código de movimentação da TPU (219/220/221).

**Não bloqueia** — é o que a categoria de resultado na dimensão de movimentação
resolve. Mas cria o ponto mais frágil do projeto: um código mal classificado corrompe
silenciosamente toda a apuração de resultado, sem sintoma visível. Daí a regra: só
entra código conferido contra a API; na dúvida, categoria neutra, e categoria neutra não
entra na métrica.

> **✅ Implementado.** 6 códigos conferidos contra a tabela oficial da TPU/CNJ —
> 219/220/221 (mérito) e 237/238/239 (recurso). Os outros **257 códigos** vistos
> na carga entraram como `Neutral, code_verified = false`: contam para volume e
> série temporal, mas **não** para apuração de resultado.

> **🔴 E revelou um problema que esta página não previa.** O código diz se a
> **pretensão** foi acolhida — não quem ganhou. Em matéria penal, "Procedência"
> é **condenação**. Além disso, mérito e recurso têm polaridades **diferentes** e
> não podem ser somados. Página dedicada:
> **[Polaridade do resultado](05-polaridade-do-resultado.md)** — leitura
> obrigatória antes de exibir qualquer percentual.

### 5b · ~~Completude do dado varia por tribunal~~ — era erro nosso *(resolvido em 20/09/2026)*

**O que esta seção dizia:** que 100% das 265.088 movimentações do TJMG vinham com
`dataHora` nulo, que isso era o dado do tribunal e que bloqueava o TJMG inteiro na
tabela fato.

**Estava errado, e o erro era do coletor.** A coleta ordenava por `@timestamp`
**crescente** e parava na cota, então só via os documentos mais antigos do índice —
no TJMG, de 2017/2018, que realmente não têm `dataHora`. Consultando o mesmo índice
em ordem **decrescente**, os documentos recentes têm `dataHora` em 100% das
movimentações. Na recarga de 20/09/2026 o TJMG entrou com **254.251 eventos**.

**A lição continua, corrigida:** a **ordem da coleta é o recorte**. Antes de atribuir
uma lacuna à fonte, vale checar se ela não é da consulta. Verificar completude campo a
campo por tribunal continua valendo — só não foi isso que aconteceu aqui.

### 6 · CPF / CNPJ das partes

Não indexados nos campos pesquisáveis, por LGPD.

**Não bloqueia** nada no escopo atual, e é uma restrição bem-vinda.

---

## Limitações operacionais

| Limitação | Consequência |
|---|---|
| Não é tempo real (horas a dias) | ETL em batch; a tela declara a data de extração |
| Um índice por tribunal | coletar de N tribunais = N chamadas |
| Teto de 100 documentos por página | paginação com `search_after` |
| Limite de taxa e falha intermitente | retry com backoff obrigatório |
| Volume | a carga é sempre um **recorte deliberado**, mesmo com três tribunais |
| Escopo territorial | TJSP, TJRJ e TJMG. **A interface precisa declarar isso** — quem assume cobertura nacional tira conclusão errada de um percentual |
| Processos em segredo de justiça | não abrem para ninguém, com ou sem link |

---

## Links para o processo na origem

Não existe "o site do governo": cada tribunal resolve *deep link* de um jeito — ou não
resolve. Com o escopo em três, são três casos a tratar.

Observado abrindo a URL e **inspecionando o conteúdo** (o código HTTP engana: a página
de erro do e-SAJ também devolve 200).

> ### ⚠ Reverificado em 15/09/2026 — o formato antigo **quebrou**
>
> A recomendação de reverificar estava certa. O conjunto de parâmetros que esta
> página registrava **não funciona mais**: devolve **200** com *"O tipo de
> pesquisa informado é inválido"*. Como a página de erro também é 200, checar
> status code não detecta nada — exatamente a armadilha descrita acima.
>
> **Faltavam dois parâmetros.** O formato que funciona hoje exige também
> `dadosConsulta.localPesquisa.cdLocal` e `dadosConsulta.valorConsultaNuUnificado`.

| Tribunal | Sistema | Resultado (verificado 15/09/2026) |
|---|---|---|
| TJ-SP 1º grau | e-SAJ `cpopg/search.do`, GET | ✅ **abre o processo** — `0000017-52.1994.8.26.0097` devolveu "Execução Fiscal / Foro de Buritama / 1ª Vara", e a classe **bate com o DataJud** |
| TJ-SP 2º grau | e-SAJ `cposg/search.do`, GET | ✅ **abre** — `0000590-16.2021.8.26.0333` devolveu "Apelação Criminal / 2º Grau / Encerrado" |
| TJ-RJ | SPA | ignora o parâmetro da URL → `portal` |
| TJ-MG | PJe/JSF | exige POST com sessão → `portal` |

**Verificação cruzada, não só HTTP 200:** o teste compara o dado da página com o
que o DataJud diz do mesmo processo. É a única forma de saber que o link abriu o
processo *certo*.

O e-SAJ também aplica limite de taxa — uma das consultas de teste voltou com
*"Foram identificadas múltiplas consultas simultâneas"*. Gerar o link é barato;
**validar links em lote não é**, e não deve entrar no ETL.

Daí as três camadas de resposta, que a interface precisa distinguir:

- **`direto`** — deep link real, abre o processo;
- **`portal`** — portal do tribunal, com o número já formatado para colar;
- **`null`** — tribunal não mapeado. Melhor sem link do que link que não resolve.

> **Regra de rótulo.** O texto do link é sempre **"consultar no tribunal"**, nunca
> "veja a decisão". Não é possível garantir por teste que um link abra um processo
> específico, e prometer isso é o tipo de coisa que o usuário descobre na frente do
> cliente dele.

---

## Consequência de produto

Boa parte do que os mockups mostram **não tem lastro hoje**. As opções, por bloco:

| Bloco sem fonte | Opção A | Opção B | Opção C |
|---|---|---|---|
| Citação de acórdão | repositório de jurisprudência do TJSP/TJRJ/TJMG | **PANGEA**, se entregar inteiro teor | remover do escopo |
| Fundamentos invocados | texto + NLP | — | remover do escopo |
| Jurisprudência qualificada | **PANGEA**, se entregar precedentes | texto + NLP | remover do escopo |
| Doutrina invocada | artigo com link (repositórios abertos, DOI) | referência de livro curada, em texto | remover do escopo |
| Valor / quantum | texto + NLP + coluna de medida no fato | — | remover do escopo |
| Relator | raspagem nos três tribunais | inteiro teor + NLP | remover do escopo |

**Não existe opção D:** exibir dado plausível gerado por modelo. O público-alvo cita o
que lê aqui em peça e em decisão; um número inventado destrói o produto e não é um
risco reputacional recuperável.

Onde a fonte não tem, **a API devolve vazio e a tela diz que não há dado** — de
preferência explicando por quê, com link para esta página.
