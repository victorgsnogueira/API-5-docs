# Glossário

Metade do time não é da área jurídica e metade não é da área de dados. Este é o
vocabulário mínimo para as duas conversas.

## Direito

**Processo** — a ação judicial em si, identificada por um número único de 20 dígitos
(padrão CNJ). É a entidade rastreada; o análogo de um "contrato" em outro domínio.
O número não é opaco: ele codifica sequencial, dígito verificador, ano, segmento de
justiça, tribunal e unidade de origem.

**Movimentação** — cada evento dentro de um processo ("recebido", "conclusos",
"sentença proferida", "recurso interposto"), com data e código de tipo. É o menor
evento que a fonte entrega e o **grão do nosso fato**.

**Decisão judicial** — quando o juiz ou o tribunal decide algo dentro do processo.
Um processo pode ter várias ao longo do tempo (1ª instância, recurso, embargos).

**Acórdão** — decisão de um órgão colegiado (câmara, turma), por oposição à sentença,
que é do juiz singular.

**Jurisprudência** — o padrão que emerge quando várias decisões sobre o mesmo tema
seguem a mesma linha. **Não é um dado bruto**: é descoberta por agregação. É
exatamente o que o DW produz.

**Precedente** — uma decisão específica usada como referência para casos futuros.

**Súmula** — enunciado curto que consolida entendimento reiterado de um tribunal
(ex.: *Súmula 385/STJ*). Na prática funciona como regra.

**Tema repetitivo / IRDR** — mecanismos que fixam tese de observância obrigatória
para casos idênticos. Distinguem-se do precedente meramente persuasivo.

**Doutrina** — opinião e interpretação de juristas acadêmicos (livros, artigos). Não
vem de tribunal; é dado de enriquecimento e contexto.

**Tribunal** — a instituição que julga (STF, STJ, TJSP, TRF…). Vira dimensão.

**Órgão julgador** — a vara, câmara ou turma dentro do tribunal. Permite responder à
pergunta de colegialidade: *"as câmaras do mesmo tribunal decidem igual entre si?"*

**Grau** — instância: 1º grau (juiz singular), 2º grau (tribunal estadual/regional),
superior (STJ/STF).

**Procedência / improcedência / procedência em parte** — o desfecho do pedido:
acolhido, rejeitado, acolhido parcialmente. No DataJud isso **não existe como campo** —
só como código de movimentação. Ver [Modelo dimensional](../03-dados/02-modelo-dimensional.md).

**Quantum** — o valor fixado na condenação.

**TPU (Tabela Processual Unificada)** — as tabelas de códigos padronizadas do CNJ
para classe processual, assunto e tipo de movimentação. Tudo no DataJud vem
codificado por ela; traduzir código → nome legível é trabalho do ETL.

**Ratio decidendi** — o fundamento determinante de uma decisão, o que nela de fato
vincula casos futuros. Dá nome ao projeto.

## Dados

**Data Warehouse (DW)** — banco modelado para análise agregada, não para operação.
Otimizado para "quantos, quanto, em que proporção", não para "inserir um registro".

**Modelagem dimensional / esquema estrela** — uma tabela **fato** central (os eventos
mensuráveis) cercada por tabelas **dimensão** (os recortes: tribunal, tempo, tema…).
O nome vem do desenho: a fato no centro, as dimensões em volta.

**Grão** — o que representa **uma linha** da tabela fato. Definir o grão é a primeira
decisão de qualquer DW e a mais difícil de reverter. O nosso: *uma movimentação
processual*.

**Chave substituta (SK, *surrogate key*)** — chave artificial e sequencial da dimensão
(`tribunal_sk`), independente da chave natural da fonte (a sigla `TJSP`).

**Tabela-ponte** — resolve relação N:N entre fato e dimensão. Um processo pode ter
vários assuntos; sem a ponte, a contagem por tema duplicaria linhas e inflaria toda
métrica.

**OLAP** — consultas analíticas por múltiplos recortes ("favorável por tribunal por
ano"). É o tipo de pergunta que o produto faz o tempo todo.

**View materializada** — o resultado de uma consulta agregada gravado em disco e
atualizado sob comando. Usada porque agregar milhões de linhas de fato a cada
request deixaria a tela lenta.

**ETL** — *Extract* (buscar na fonte), *Transform* (limpar, traduzir códigos,
padronizar, deduplicar), *Load* (gravar nas tabelas do DW).

**Idempotência** — rodar a carga duas vezes com os mesmos dados não duplica linha nem
infla contagem. Requisito, não conforto: o agendamento diário reprocessa janelas que
se sobrepõem.

**NLP** — processamento de linguagem natural. Aqui: normalizar textos e assuntos
heterogêneos em um mesmo **tema**, e extrair fundamentos e valores de texto livre.

**Tema** — a entidade central do produto. Ver [O que é um tema](../01-produto/03-tema-modelo-conceitual.md).
