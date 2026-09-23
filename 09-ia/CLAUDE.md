# Ratio — contexto para desenvolver com IA

Contexto global do projeto, válido nos três repositórios de código. Os detalhes de cada
repositório ficam em [`repos/`](repos/). A fonte de verdade é esta wiki
(`https://github.com/victorgsnogueira/API-5-docs`); em caso de conflito, a wiki vence.

## O projeto

Data Warehouse jurídico com busca por **tema**, não por processo. Escopo: TJSP, TJRJ e
TJMG, só matéria cível. Produção é a **intranet do cliente** (Windows Server, NGINX, sem
internet no servidor), que recebe só arquivos buildados.

| Repositório | O quê | Stack |
|---|---|---|
| `Concord-API/API5-Backend` | API REST que lê o `dw` e cria o schema via migration | .NET 10, Dapper, Npgsql, DbUp, xUnit, Testcontainers |
| `Concord-API/API5-Frontend` | tela de busca e de tema | React, TanStack Router/Query, zod, Vitest, MSW |
| `Concord-API/API5-Pipeline` | coleta, transforma e gera o arquivo de carga do `dw` | Python 3.12, psycopg2, pytest, Testcontainers |
| `Concord-API/API-5` | backlog e critérios de aceite da org | — |

## Como o trabalho anda

- **Toda mudança nasce de uma task do board** (projeto "API V Tasks", org `Concord-API`).
  A task tem ID `X.Y`: `X` é a US, `0.Y` é o Technical Foundation. A descrição tem
  `Data:` (o que fazer) e `Verifies:` (como provar). Detalhe em `08-backlog/tasks/`.
- **Branches:** `main` ← `usX` (uma por US, sem zero à esquerda; `us0` é o Technical
  Foundation, permanente) ← `<id>-<Titulo-da-task-com-hifens>`, sem acento nem barra.
- **Uma task é uma branch e um merge só**, mesmo que a implementação tenha várias
  etapas. A `usX` só vai pra `main` quando a US inteira termina; a `us0` vai quando uma
  task do Foundation termina.
- **O card vai para In Progress quando o commit 1 é liberado** (a skill `plano-task` faz
  isso); daí em diante o board se move sozinho pelo nome da branch: PR aberto → Review,
  PR mergeado → Done.
- **PR só com título**, sem descrição. Ninguém dá push direto na `main` nem nas `usX`.

## Commits

- Em inglês, **só a linha do assunto**, sem corpo, sem `Co-Authored-By` de IA.
- Tipos: `feat`, `fix`, `refactor`, `docs`, `test` (qualquer teste), `chore`.
- **TDD obrigatório:** um commit `test:` com o teste vermelho pelo motivo certo, depois o
  `feat:`/`fix:` que o faz passar. O que precisa estar verde é o PR, não cada commit.

## Código

- **Sem comentário nenhum** — nem em código, nem em SQL, nem em teste. Pendência vai
  pra task, pro PR ou pros Docs.
- Código em inglês; tudo que a API devolve ao usuário (dados, rótulos, erros) em
  **português**.
- Antes de escrever uma asserção sobre comportamento de banco ou biblioteca, **confira
  rodando** — não suponha.

## Banco

- O `dw` é **criado pela API** (DbUp, `Ratio.Infrastructure/Migrations/Scripts/V00N__*.sql`)
  e **populado pelo arquivo de carga** do pipeline (`TRUNCATE` + `COPY` + `REFRESH`).
- **Nunca edite uma migration já lançada**; toda mudança é uma `V00N` nova, segura contra
  dado que já existe (sem `NOT NULL` sem `DEFAULT`, sem `CHECK` que o dado atual quebre).
- A API usa **uma credencial só** e **não gerencia usuário nem grant** (D-36).
- As extensões `unaccent` e `pg_trgm` são instaladas por quem provisiona o banco, no
  schema `public`.

## Decisões que não se reabrem sem conversa

- Chave pública do tema (`theme_key`) estável entre cargas.
- Abaixo do piso de `n`, mostra contagem, nunca percentual (D-32).
- **Nenhuma chamada de LLM dentro da carga automática**; LLM só na descoberta offline de
  temas, com revisão humana. Narrativa por template (D-37).
- Nenhum código penal entra no `dw`.

Todas as decisões: `06-operacao/02-decisoes-e-riscos.md`.

## Planejar antes de codar

Antes de tocar em código, apresente o plano da task no formato do skill
[`plano-task`](skills/plano-task/SKILL.md): branch, arquivos novos, o que muda em cada
arquivo e a lista de commits. Só implemente depois de aprovado.
