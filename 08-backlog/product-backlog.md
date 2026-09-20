### Escala de estimativa

| SP | Significado | Camadas | Incerteza |
| --- | --- | --- | --- |
| **1** | ajuste isolado: um texto, uma formatação | 1 | nenhuma |
| **2** | a última peça de algo que outra story já entregou | 1 | nenhuma |
| **3** | trabalho próprio em uma camada: um bloco, uma tabela, um componente | 1 | nenhuma |
| **5** | duas camadas, ou uma regra que alguém precisa definir | 2 | alguma |
| **8** | atravessa do banco à tela, ou o time nunca fez aquilo | 3+ | real |
| **13** | grande **e** incerta — teto da escala, candidata a quebra no refinamento | todas | alta |

**O que faz subir de degrau:** quantas camadas a story toca (banco → backend →
tela), se há decisão aberta no caminho, se um modelo participa, se exige alteração no banco.

A Story `US-14` pode ser tomada como base de estimativa de ST para as outros, já que é a 3.

---

### Prioridade — MoSCoW

| Prioridade | Significado |
| --- | --- |
| **Must** | Obrigatório para a entrega. Sem este item, o produto não cumpre seu objetivo principal, não atende ao requisito essencial ou não pode ser considerado funcional/avaliável. Deve ser priorizado antes de qualquer outro item. |
| **Should** | Importante para uma entrega completa. O produto ainda funciona sem este item, mas sua ausência gera uma perda relevante de qualidade, usabilidade ou cobertura do objetivo. Deve entrar se houver capacidade disponível após os Must. |
| **Could** | Melhoria desejável. Agrega valor ao produto, mas não compromete a entrega caso fique de fora. É o primeiro grupo a ser reduzido ou adiado quando houver restrição de prazo, equipe ou capacidade. |
| **Won't** | Fora da entrega atual. O item foi analisado e deliberadamente não será desenvolvido neste ciclo/projeto. Deve ser registrado no Fora do escopo para evitar que volte ao backlog como uma demanda inesperada. |

---

## Product Backlog

| ID | Prioridade | Descrição | Estimativa | Sprint |
| --- | --- | --- | --- | --- |
| **US-01** | Must | Como **usuário**, quero digitar o tema do meu caso em linguagem natural e receber temas jurídicos curados — não uma lista de processos — para descobrir como ele vem sendo decidido sem garimpar um acórdão por vez | 8 | Sprint 1 |
| **US-02** | Must | Como **usuário**, quero os resultados ordenados por quão consolidado está o entendimento, e não por relevância de texto, para encontrar primeiro o que sustenta minha tese | 2 | Sprint 1 |
| **US-09** | Must | Como **usuário**, quero ler o entendimento do tema em prosa, abrindo com o número que responde à pergunta e com a contagem de casos ao lado de cada percentual, para entender o padrão sem abrir uma tabela | 8 | Sprint 1 |
| **US-10** | Must | Como **usuário**, quero ver a distribuição de resultados do tema em uma figura com a fonte declarada, para ver de relance quanto é procedente, parcialmente procedente e improcedente | 5 | Sprint 1 |
| **US-21** | Must | Como **usuário**, quero ver a doutrina relacionada ao tema, com autor, obra e link para o artigo quando houver, para saber o que citar além da jurisprudência | 8 | Sprint 1 |
| **US-24** | Must | Como **usuário**, quero que toda tela deixe claro que os dados cobrem TJSP, TJRJ e TJMG, para não tirar uma conclusão nacional de um percentual que reflete três estados | 2 | Sprint 1 |
| **US-25** | Must | Como **usuário**, quero saber a fonte e a data de extração de todo número que estou vendo, para saber exatamente o que estou citando | 3 | Sprint 1 |
| **US-26** | Must | Como **usuário**, quero que a tela me diga o que não existe e por quê, em vez de mostrar um campo vazio ou um valor plausível, para não construir uma peça sobre dado que não existe | 3 | Sprint 1 |
| **US-03** | Must | Como **usuário**, quero ver em cada resultado o score, a área do direito, o título da tese, um resumo curto, os tribunais, o volume, o período, a última decisão e o percentual favorável, para comparar teses antes de abrir qualquer uma delas | 8 | Sprint 2 |
| **US-04** | Must | Como **usuário**, quero filtrar os resultados por tribunal, período, instância e força mínima, vendo quantos processos cada tribunal tem, para restringir a lista ao meu caso | 5 | Sprint 2 |
| **US-06** | Must | Como **usuário**, quero um score de 0 a 100 dizendo o quanto o entendimento sobre o tema está consolidado, para saber se a tese vale ser defendida ou é uma briga aberta | 8 | Sprint 2 |
| **US-07** | Must | Como **usuário**, quero que o score venha com uma classificação em linguagem que já existe no meio jurídico — Consolidada, Dominante, Em formação, Divergente — para não ter de interpretar uma escala inventada | 2 | Sprint 2 |
| **US-08** | Must | Como **usuário**, quero abrir a composição do score — os quatro componentes, seus pesos e a base de cálculo — para citar a estatística sabendo exatamente de onde ela vem | 3 | Sprint 2 |
| **US-11** | Must | Como **usuário**, quero ver como o alinhamento do tema evoluiu ano a ano, com a frase de tendência, para saber se o entendimento está se firmando ou mudando | 5 | Sprint 2 |
| **US-12** | Must | Como **usuário**, quero ver o alinhamento do tema por tribunal, para saber se a tese se sustenta igual em São Paulo, Rio de Janeiro e Minas Gerais | 3 | Sprint 2 |
| **US-14** | Must | Como **usuário**, quero uma tabela do comportamento de cada tribunal — decisões, alinhamento e data da mais recente — para comparar o meu tribunal com os outros | 3 | Sprint 2 |
| **US-15** | Must | Como **usuário**, quero uma amostra auditável dos processos por trás do tema, com câmara, data e resultado, para conferir os casos antes de citá-los | 5 | Sprint 2 |
| **US-17** | Should | Como **usuário**, quero saber se as câmaras do meu tribunal estão decidindo igual, para identificar divergência interna antes de decidir | 5 | Sprint 2 |
| **US-27** | Should | Como **usuário**, quero exportar para CSV as decisões que sustentam o tema, para trabalhar os dados fora da ferramenta | 5 | Sprint 3 |
| **US-28** | Should | Como **usuário**, quero copiar a citação do tema já com a fonte, a data de extração, o escopo e o `n`, para colar sem redigitar | 3 | Sprint 3 |
| **US-29** | Should | Como **usuário**, quero saber quando os dados foram atualizados pela última vez, e ser avisado quando estiverem defasados, para não me apoiar em um retrato de meses atrás | 3 | Sprint 3 |
| **US-13** | Could | Como **usuário**, quero que a aba que estou vendo seja refletida na URL, para enviar a um colega o link exato da parte que quero mostrar | 5 | Sprint 1 |
| **US-38** | Could | Como **usuário**, quero sugestões de consultas frequentes na tela inicial, para entender que tipo de pergunta a ferramenta responde antes de digitar a minha | 2 | Sprint 2 |
| **US-19** | Could | Como **usuário**, quero ver os fundamentos invocados nas decisões do tema, com a frequência e a taxa de acolhimento de cada um, para escolher o argumento que mais vence e evitar o que sempre perde | 8 | Sprint 2 |
| **US-37** | Could | Como **usuário**, quero ver no texto do entendimento o acórdão que sustenta cada afirmação, com a referência completa e a lista de decisões citadas ao pé, para poder citar a mesma decisão | 8 | Sprint 2 |
| **US-30** | Could | Como **usuário**, quero saber quanto tempo normalmente se leva do ajuizamento à decisão neste tema, para calibrar a expectativa do meu cliente | 8 | Sprint 3 |
| **US-34** | Could | Como **usuário**, quero perguntar a um chatbot sobre um tema em linguagem natural e receber a resposta em prosa com os números, para as perguntas que nenhum filtro de tela responde | 13 | Sprint 3 |
| **US-35** | Could | Como **usuário**, quero que toda resposta do chatbot traga o `n`, a fonte, a data de extração, o escopo e o link para os processos, para poder conferir antes de usar | 5 | Sprint 3 |
| **US-36** | Could | Como **usuário**, quero que o chatbot diga que não sabe quando o dado não está na base, para não receber um número plausível e inventado | 3 | Sprint 3 |

## Distribuição por sprint

| Sprint | Janela | Histórias | Pontos |
| --- | --- | --- | --- |
| **Sprint 1** | 07/09 – 27/09 | 9 | 44 |
| **Sprint 2** | 05/10 – 25/10 | 13 | 65 |
| **Sprint 3** | 02/11 – 22/11 | 7 | 40 |

## Requisitos não funcionais:

- O dado é armazenado em um Data Warehouse com modelagem dimensional (tabela fato, dimensões e tabela-ponte para as relações N:N) com o grão declarado por escrito
- Todos os dados mostrados, tanto analíticos como descritivos devem ter sua fonte (como foi formulado) informada - isso deve ocorrer para todos os temas.
- Migrations versionadas, aplicadas por runner no pipeline nunca SQL à mão em produção
- Existe um pipeline de ETL completo executável de forma independente das outras soluções.
- A carga é idempotente: rodar duas vezes com os mesmos dados não duplica linha nem infla contagem
- ETL roda como job agendado, fora do ciclo de vida da API, em horário de baixo uso
- As perguntas do produto são respondidas por consultas OLAP pré-agregadas, resumo por tema, por ano, por tribunal, por órgão, não por agregação sobre o fato em tempo de request
- A busca full-text é em português, no próprio Postgres, tolerando ausência de acento e erro de digitação
- 

T