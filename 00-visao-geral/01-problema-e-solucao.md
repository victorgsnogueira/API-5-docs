# Problema e solução

## A dor do cliente

Advogados e juízes precisam saber **como um determinado tema vem sendo decidido**
antes de agir:

- o **advogado** precisa disso para escolher a tese que sustenta, calibrar o pedido
  (inclusive o valor) e antecipar o argumento da outra parte;
- o **juiz** precisa disso para saber o que o próprio tribunal e as instâncias
  superiores já firmaram sobre a matéria.

Hoje esse trabalho é manual e caro:

1. o dado está espalhado em dezenas de portais — só nos três estados do nosso escopo
   são três sistemas diferentes (e-SAJ no TJSP, portal próprio no TJRJ, PJe no TJMG),
   mais bases nacionais (DataJud, PANGEA) e repositórios de jurisprudência próprios;
2. não existe visão agregada: para saber "quanto costuma ser fixado de dano moral
   nesse caso no TJSP", a pessoa abre acórdão por acórdão;
3. o resultado desse esforço acaba virando **planilha de Excel montada à mão**, que
   não se atualiza, não é auditável e morre com quem a montou.

O cliente pediu, em uma frase: **centralizar esses dados em um único lugar onde
advogados e juízes possam consultar e apoiar suas análises.**

## A solução ofertada

Um site com a ergonomia de um buscador: a pessoa digita o tema em linguagem natural
("inscrição indevida em cadastro de inadimplentes") e recebe de volta **temas
jurídicos apurados** — não uma lista de processos.

Ao abrir um tema, ela vê duas coisas, em duas abas:

- **Resumo** — o entendimento em prosa, com os números embutidos no texto e cada
  afirmação apoiada em decisão citável;
- **Base analítica** — as tabelas por trás do resumo: comportamento por tribunal,
  fundamentos invocados e taxa de acolhimento, jurisprudência qualificada, doutrina
  e uma amostra auditável de processos com link para a fonte oficial.

Detalhe do que cada tela mostra: [Telas](../01-produto/02-telas.md).

## O que torna isso diferente de uma busca de jurisprudência

Buscadores jurídicos existentes devolvem **documentos**. O Ratio devolve **padrão**:

| Busca tradicional | Ratio |
|---|---|
| "achei 12.418 acórdãos" | "em 82% dos 12.418, o pedido foi acolhido" |
| ordenado por relevância textual | ordenado por [força do entendimento](../01-produto/04-forca-do-entendimento.md) |
| você lê para descobrir a tendência | a tendência é o resultado; os documentos são a prova |

O núcleo técnico que sustenta isso é um **Data Warehouse dimensional** — o dado
precisa estar modelado para agregação, não para leitura documento a documento.

## Restrições do desafio acadêmico

O projeto é uma API (Aprendizagem por Projeto Integrado) da FATEC, com requisitos
fixos que valem tanto quanto o produto:

- **uso obrigatório de Data Warehouse** com modelagem dimensional (fatos e dimensões)
  e consultas OLAP;
- **pipeline de ETL** completo (extração, transformação, carga);
- **boas práticas de DevOps** — testes automatizados validando integridade dos dados
  e consistência das consultas, CI/CD, versionamento.

A camada de produto (busca, telas, chatbot) é construída **em cima** desse núcleo,
não no lugar dele.

## Limite honesto do escopo

A fonte primária (DataJud/CNJ) entrega **metadado processual**, não texto integral de
decisão nem doutrina. Boa parte do que aparece nos mockups — citação de acórdão,
doutrina invocada, valor fixado — depende de fonte adicional ou de extração por NLP
que ainda não existe. Isso está catalogado sem eufemismo em
[Limitações da fonte](../03-dados/04-limitacoes-da-fonte.md) e é o principal risco
do projeto.
