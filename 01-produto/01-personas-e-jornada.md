# Personas e jornada

## As duas personas

### Advogado — precisa **sustentar** uma posição

Chega com um caso concreto e uma pergunta prática:

- *"Essa tese pega? Em quantos por cento dos casos ela é acolhida?"*
- *"Quanto costuma ser fixado? Vale pedir mais e arriscar sucumbência?"*
- *"Se eu defendo, qual é o argumento com maior taxa de sucesso — e qual é o que
  perde sempre?"*
- *"Que acórdão eu cito na petição?"*

O que ele precisa levar embora: **um número que ele possa citar e um processo que
ele possa abrir**. Estatística que não dá para auditar, ele não usa — porque a outra
parte vai conferir.

### Juiz / desembargador — precisa **decidir** com coerência

- *"O que o meu tribunal já firmou sobre isso?"*
- *"As câmaras estão decidindo igual entre si, ou há divergência interna?"*
- *"O STJ já se pronunciou? Há tema repetitivo ou súmula vinculando?"*
- *"O valor que eu ia fixar está dentro da faixa praticada?"*

A pergunta de colegialidade ("as câmaras conflitam?") é o que justifica a dimensão
uma dimensão de órgão julgador e um agregado por órgão — não é enfeite analítico.

## O que as duas têm em comum

Ambas **citam** o que leem, em peça ou em decisão. Isso impõe três requisitos que
atravessam o produto inteiro:

1. **Todo número é rastreável.** Nenhum indicador aparece sem o `n` e sem a fonte.
2. **Todo processo é abrível.** O número do processo vem com link para o tribunal de
   origem. Onde não dá para garantir o deep link, o rótulo é "consultar no tribunal",
   nunca "veja a decisão". Ver [links para a fonte](../05-prototipo/01-prototipo-referencia.md#sobre-links-para-o-processo-na-origem).
3. **A metodologia é aberta.** A nota de força vem com os componentes e os pesos
   expostos. Ver [Força do entendimento](04-forca-do-entendimento.md).

## Jornada principal

```
  ①  Tela inicial
      digita "inscrição indevida em cadastro de inadimplentes"
                          │
                          ▼
  ②  Resultados — 5 temas, ordenados por força
      cada um com: nota /100, matéria, título, resumo, tribunais,
      volume, período, última decisão, % favorável
      filtros: tribunal · período · grau · força mínima
                          │  clica no tema
                          ▼
  ③  Detalhamento
      ├─ aba RESUMO          o entendimento em prosa, com figuras
      │                      numeradas e decisões citadas ao pé
      └─ aba BASE ANALÍTICA  as tabelas por trás do resumo
                          │
                          ▼
  ④  Saída
      exportar CSV · copiar citação · abrir o processo no tribunal
```

## Momentos que decidem se o produto serve

| Momento | O que não pode acontecer |
|---|---|
| Busca | devolver processo em vez de tema — a pessoa não veio procurar um número |
| Resultado | tema com 3 decisões aparecendo com nota alta e parecendo consolidado |
| Detalhamento | número no texto sem `n` e sem fonte declarada |
| Citação | link que abre uma página de erro do tribunal e não o processo |

## Escopo de cobertura — o que dizer ao usuário

Os dados cobrem **TJSP, TJRJ e TJMG**. Isso precisa estar visível: um advogado que
assume cobertura nacional tira conclusão errada de um percentual que só reflete três
estados. A tela declara o escopo, e o [chatbot](05-chatbot.md) também.

## Roadmap

- **Chatbot** — assistente que responde em linguagem natural consultando o DW. Está
  planejado; ver [Chatbot](05-chatbot.md). Só faz sentido depois que os agregados
  estiverem estáveis.

## Fora do escopo

- **Alerta de mudança de entendimento** ("me avise se essa tese virar").
- **Comparação entre dois temas** lado a lado.
