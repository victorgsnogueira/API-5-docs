# Fonte para as decisões do STF (spike 0.93)

**Situação em 22/10/2026.** O DataJud cobre TST, TSE, STJ e STM, mas **não o STF**, que
não é subordinado ao CNJ. Este spike levantou a fonte que existe e o que ela permite. A
decisão de incluir ou não o STF é a task 0.94.

## A fonte: decisões do programa Corte Aberta

| Item | O que foi encontrado |
|---|---|
| Quem publica | O próprio STF, pelo programa **Corte Aberta** (Resolução STF n. 774/2022) |
| Onde | Painel de decisões em `transparencia.stf.jus.br/extensions/decisoes/decisoes.html`, com download em CSV e XLSX; o portal tem um repositório de "Dados Abertos" |
| Cópia tratada | Base dos Dados, tabela `basedosdados.br_stf_corte_aberta.decisoes` (1,44 GB), gratuita até jan/2025 |
| Cobertura | Decisões desde **2000**; a fonte original é atualizada **mensalmente** |
| Termos de uso | Dados abertos, publicados para consulta e exportação pelo próprio programa |
| Acesso automatizado | Sem API documentada. O caminho é baixar a base (CSV) e carregar no pipeline, como uma fonte nova ao lado do DataJud |

## Colunas da base de decisões

`ano`, `classe`, `numero`, `relator`, `link`, `subgrupo_andamento`, `andamento`,
`observacao_andamento_decisao`, `modalidade_julgamento`, `tipo_julgamento`,
`meio_tramitacao`, `indicador_tramitacao`, `assunto_processo`, `ramo_direito`,
`data_autuacao`, `data_decisao`, `data_baixa_processo`.

## O que dá e o que não dá para fazer

| Pergunta do produto | Resposta |
|---|---|
| Recortar só o cível? | **Sim**, por `ramo_direito` e `assunto_processo` |
| Saber o resultado da decisão? | **Em parte.** O resultado vem em texto (`andamento`, `subgrupo_andamento`) e precisa de um mapeamento verificado para acolhido e rejeitado, como foi feito com os códigos de movimento do DataJud |
| Saber quem julgou? | **Sim**: `relator`, `tipo_julgamento` (monocrática ou colegiada) e `modalidade_julgamento` |
| Ligar ao processo do TJ ou do STJ pelo número CNJ? | **Não.** Não há o número único de 20 dígitos nem o tribunal de origem: `numero` é o número do processo no STF, lido junto com a `classe` |
| Ligar ao tema? | Só pelo texto do assunto. O assunto não vem com o código da TPU, então o tema precisaria de um mapeamento por nome |
| Link para o usuário conferir? | **Sim**, a coluna `link` leva à página do processo no portal do STF |

## Consequências para a decisão 0.94

- Incluir o STF é possível, mas como **instância à parte**: cada processo do STF conta por
  si. A regra da D-41 (o processo julgado no TJ e no STJ conta uma vez, no tribunal da
  última decisão) **não alcança o STF**, porque não há como achar o mesmo processo.
- O trabalho é de uma fonte nova: carregar o CSV, mapear `andamento` para resultado,
  mapear o assunto para tema e semear o STF na dimensão de tribunal só quando houver dado.
- Enquanto o STF não entrar, a cobertura máxima de um tema continua em **4 de 5**
  tribunais, que é o efeito já previsto da decisão de `coverage_courts = 5`.

**Recomendação do spike:** não incluir o STF na Sprint 2. Abrir as tasks da fonte nova
para uma sprint seguinte, se o PO quiser o STF no escopo.

Fontes consultadas em 22/10/2026: o painel de decisões do Corte Aberta
(`transparencia.stf.jus.br`) e a página do conjunto na Base dos Dados
(`basedosdados.org/dataset/b46bb892-3273-434d-9335-f502b8656ef1`).
