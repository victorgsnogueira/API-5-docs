# Chatbot

> **Planejado, não construído.** Está no roadmap do produto; nada foi implementado.
> Esta página fixa o desenho antes que alguém comece pelo lado errado.

## O que é

Um assistente que responde perguntas em linguagem natural **usando o nosso Data
Warehouse como fonte** — não a memória do modelo, não a internet.

```
"como o TJSP vem decidindo negativação indevida nos últimos dois anos?"
                          │
                          ▼
              LLM interpreta a pergunta
                          │
                          ▼
         consulta os agregados do DW (OLAP)
                          │
                          ▼
   resposta em prosa, com os números e a fonte de cada um
```

## Por que ele faz sentido aqui

O DW já responde a essas perguntas — mas hoje só através de uma tela, com filtros
fixos. Muita pergunta legítima não cabe em nenhum filtro:

- *"esse tema é mais favorável em SP ou em MG?"*
- *"a tendência mudou depois de 2024?"*
- *"que câmara do TJSP diverge das outras nesse assunto?"*

Cada uma dessas é uma consulta OLAP. O chatbot é **uma interface de linguagem natural
sobre as consultas que já existem** — não um caminho novo até o dado.

## A regra que define tudo

> **O modelo interpreta a pergunta e redige a resposta. O modelo não conta.**
> **Todo número vem de `SELECT`.**

Isso é o mesmo princípio que vale para o resto do produto
([D-08](../06-operacao/02-decisoes-e-riscos.md)), e aqui ele é ainda mais crítico:
uma LLM produz números plausíveis com total fluência, e o público-alvo **cita** o que
lê. Um percentual alucinado numa petição é dano que não se desfaz.

## Arquitetura recomendada — *tool use*, não texto no prompt

Duas abordagens possíveis, e uma delas é claramente melhor.

### ❌ Enfiar dados no prompt

Buscar um monte de linhas, colar no contexto e pedir para o modelo analisar.

Problemas: não escala com o volume; o modelo faz aritmética sobre o texto (e erra);
impossível auditar de onde veio o número.

### ✅ Expor consultas como ferramentas

O modelo não recebe dados: recebe um conjunto de **funções** que ele pode chamar, cada
uma correspondendo a uma consulta parametrizada e já validada.

```
searchTopics(query, court?, period?)          -> lista de temas
getTopicSummary(topicCode)                    -> volume, alinhamento, período
getTopicByYear(topicCode)                     -> série anual
getTopicByCourt(topicCode)                    -> recorte por tribunal
getTopicByJudgingBody(topicCode, court)       -> colegialidade
listDecisions(topicCode, court?, outcome?)    -> processos com link para a origem
```

O modelo escolhe a ferramenta, o backend executa o SQL, o modelo redige a resposta com
o resultado que voltou. Vantagens:

- **o número é sempre de `SELECT`** — a ferramenta é a única via até o dado;
- **auditável** — dá para logar exatamente quais ferramentas foram chamadas com quais
  parâmetros e o que retornaram;
- **escala** — o volume de dados não passa pelo contexto;
- **reaproveita** — são as mesmas consultas que as telas usam.

> Ferramenta nenhuma deve aceitar SQL livre vindo do modelo. Consultas parametrizadas,
> parâmetros validados, e um teto de linhas em cada uma.

## Requisitos de resposta

Toda resposta do chatbot precisa carregar o mesmo rigor das telas:

| Requisito | Por quê |
|---|---|
| **O `n` junto de todo percentual** | "82%" sem base não é informação |
| **A fonte e a data de extração** | o produto é multifonte; o usuário precisa saber de onde veio |
| **Link para os processos que sustentam** | ele vai querer conferir e citar |
| **Dizer "não sei" quando o DW não tem** | a tentação de completar a lacuna é exatamente o risco |
| **Escopo declarado** | os dados cobrem TJSP, TJRJ e TJMG — a resposta não pode sugerir cobertura nacional |

Esse último ponto merece atenção: perguntado sobre "os tribunais brasileiros", o
chatbot precisa responder sobre os três que temos, **dizendo que são três**.

## O que ele não é

- **Não dá aconselhamento jurídico.** Ele descreve o que os dados mostram. "Em 82% dos
  casos o pedido foi acolhido" é dado; "você deve entrar com essa ação" não é o produto.
- **Não inventa jurisprudência.** Se o tema não está no DW, a resposta é que não está.
- **Não substitui as telas.** A tela é melhor para exploração visual e para citar; o
  chat é melhor para perguntas que não cabem em filtro.
- **Não é o núcleo do desafio acadêmico.** O núcleo é DW + ETL + OLAP + DevOps. O
  chatbot é camada de produto em cima disso, e só faz sentido depois que os agregados
  estiverem estáveis.

## Ordem de construção

1. DW carregado e agregados estáveis;
2. consultas expostas como ferramentas, testadas isoladamente (sem LLM no caminho);
3. camada de LLM com *tool use*;
4. log de auditoria de toda conversa: pergunta, ferramentas chamadas, dados retornados,
   resposta;
5. avaliação — um conjunto de perguntas com resposta conhecida, verificando que os
   números batem com o `SELECT` direto.

O passo 5 não é opcional. Sem ele, não há como afirmar que o chatbot não está mentindo.

## Pontos em aberto

| Pergunta | Impacto |
|---|---|
| Qual modelo / provedor? | custo, latência, se o dado sai da nossa infraestrutura |
| O que vai no log de auditoria, e por quanto tempo? | LGPD, e a pergunta do usuário pode conter dado do caso dele |
| Há memória de conversa entre sessões? | privacidade e complexidade |
| Custo por pergunta e teto de uso | um chat aberto ao público sem teto é conta imprevisível |
| Ele aparece em todas as telas, ou só no detalhamento? | os mockups atuais não mostram chat |
