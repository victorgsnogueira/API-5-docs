# Guias de implementação

Um guia por task, com o contexto técnico que a descrição do board não carrega: o que já
existe no código, as pegadinhas conferidas no banco e de quem a task depende. O guia não
substitui o `/plano-task`: ele é o que você lê **antes** de pedir o plano.

## US-01 — busca por tema

| Task | Repositório | Guia | Depende de |
|---|---|---|---|
| 1.1 | Backend | [1.1](1.1.md) · feita | — |
| 1.2 | Backend | [1.2](1.2.md) | 1.1 |
| 1.3 | Backend | [1.3](1.3.md) | 1.2 |
| 1.4 | Backend | [1.4](1.4.md) · feita | 1.3 |
| 1.5 | Backend | [1.5](1.5.md) | 1.3, 2.1 (decidido: espera a 2.1) |
| 1.6 | Backend | [1.6](1.6.md) | 1.5 |
| 1.7 | Backend | [1.7](1.7.md) | 1.5 |
| 1.8 | Backend | [1.8](1.8.md) | 1.3 (e 1.7 para a lista sem termo) |
| 1.9 | Pipeline | [1.9](1.9.md) | 1.4 |
| 1.10 | Frontend | [1.10](1.10.md) | 0.14 |
| 1.11 | Frontend | [1.11](1.11.md) | 0.14, 1.10 |
| 1.12 | Frontend | [1.12](1.12.md) | 1.11 |
| 1.13 | Frontend | [1.13](1.13.md) | 1.11 |

Ordem no backend: **1.2 → 1.3 → 1.4**, e em paralelo a **2.1** (US-02); depois **1.5**, e
então 1.6, 1.7 e 1.8. O frontend não espera o backend: trabalha contra o
[contrato de `GET /api/themes`](../../../02-arquitetura/02-backend-dotnet.md#get-apithemes--contrato-da-sprint-1)
com MSW, mas precisa da **0.14** (esqueleto) antes.

## US-02 — ordem por força

| Task | Repositório | Guia | Depende de |
|---|---|---|---|
| 2.1 | Backend | [2.1](2.1.md) | 0.12 (a carga real também precisa da 0.72 e da 0.73) |
| 2.2 | Backend (testes) | [2.2](2.2.md) | 2.1 |
| 2.3 | Backend | [2.3](2.3.md) | 1.3, 2.1 |
| 2.4 | Backend | [2.4](2.4.md) | 2.3 |
| 2.5 | Backend | [2.5](2.5.md) | 1.5, 2.3 |
| 2.6 | Frontend | [2.6](2.6.md) | 1.11 |

A **2.1 é o gargalo das duas US**: a 1.5, a 2.3 e a 2.5 esperam por ela. Comece por ela
junto com a 1.2.

## US-09 — entendimento em prosa

A origem do texto está decidida na [D-38](../../../06-operacao/02-decisoes-e-riscos.md#d-38--origem-do-texto-do-tema-template-na-carga-curado-por-cima), e o formato da resposta, com o template e
os segmentos, está no [contrato de `GET /api/themes/{key}`](../../../02-arquitetura/02-backend-dotnet.md#get-apithemeskey--contrato-da-sprint-1). As tasks seguem esses dois
documentos.

| Task | Repositório | Guia | Depende de |
|---|---|---|---|
| 9.3 | Backend | [9.3](9.3.md) | — |
| 9.4 | Pipeline | [9.4](9.4.md) | 9.3 |
| 9.5 | Pipeline | [9.5](9.5.md) | 9.4 |
| 9.6 | Backend | [9.6](9.6.md) | 9.3 |
| 9.7 | Backend | [9.7](9.7.md) | 9.6 |
| 9.8 | Frontend | [9.8](9.8.md) | — (contra o contrato, com MSW) |
| 9.9 | Frontend | [9.9](9.9.md) | 9.8 |
| 9.10 | Frontend | [9.10](9.10.md) | 9.9 |
| 9.11 | Frontend | [9.11](9.11.md) | 9.9, US-26 (26.1, 26.2, 26.4) |

A **9.3 é o gargalo do Backend e do Pipeline**: a 9.4 e a 9.6 esperam pela tabela. As três
frentes andam juntas: Backend (9.3 → 9.6 → 9.7), Pipeline (9.4 → 9.5, depois da migration da
9.3 na `main`) e Frontend (9.8 → 9.9 → 9.10, contra o contrato com MSW, desde já). A 9.11
espera o contrato de `unavailable` da US-26.

## US-10 — distribuição em figura

O tratamento da procedência em parte está decidido na [D-39](../../../06-operacao/02-decisoes-e-riscos.md#d-39--procedência-em-parte-soma-na-nota-separada-na-figura), e o campo
`outcomeBreakdown` já está no [contrato](../../../02-arquitetura/02-backend-dotnet.md#get-apithemeskey--contrato-da-sprint-1).

| Task | Repositório | Guia | Depende de |
|---|---|---|---|
| 10.2 | Backend | [10.2](10.2.md) | — |
| 10.3 | Backend | [10.3](10.3.md) | 10.2, 9.6 |
| 10.4 | Frontend | [10.4](10.4.md) | — |
| 10.5 | Frontend | [10.5](10.5.md) | 10.4, 9.9 |
| 10.6 | Frontend | [10.6](10.6.md) | 10.5 |
| 10.7 | Frontend | [10.7](10.7.md) | 9.9, US-26 (26.1, 26.2, 26.4) |

A **10.2 e a 10.4 não dependem de nada** e podem começar já.

## Pipeline — o que a carga real precisa

| Task | O quê |
|---|---|
| [0.72](https://github.com/Concord-API/API-5/issues/139) | gravar a linha de `dw.strength_config` na carga — sem ela a nota e o piso de `n` saem vazios |
| [0.73](https://github.com/Concord-API/API-5/issues/140) | `REFRESH` das views na ordem de dependência — em ordem alfabética a `theme_strength` quebra a carga |

## Fluxo de toda task

1. Atribua a issue a você no board.
2. Rode `/plano-task <id>` ([como instalar a skill](../../../09-ia/README.md)) e siga o
   plano aprovado.
3. Crie a branch a partir da **`usN`** da story, no repositório da task (`us9` para a
   9.Y, `us10` para a 10.Y). O nome é o ID mais o título do board com hífens, sem acento
   nem crase. Se a `usN` ainda não existir, ela nasce da `main`; tasks `0.Y` saem da `us0`.
4. Commits em TDD: `test:` vermelho pelo motivo certo, depois `feat:`/`fix:`. Só a linha
   do assunto, em inglês. Nenhum comentário no código, no SQL ou no teste.
5. PR para a `usN`, só com título. O board se move sozinho.
6. Decisão ou contrato não vira task: é escrito direto nesta documentação, e as tasks
   apontam para ele.

## Regras que valem para todas as tasks de banco

- **Migration nova a cada mudança**, `V00N__descricao.sql` em
  `Ratio.Infrastructure/Migrations/Scripts/`. Nunca edite uma já mergeada. Se duas tasks
  pegarem o mesmo número, quem mergear depois renumera antes do merge.
- Toda migration nova entra na lista `Scripts` de `DatabaseMigratorTests.cs`, senão esse
  teste quebra.
- **Não semeie dado em migration.** O arquivo de carga faz `TRUNCATE` em todas as tabelas
  do `dw`; o que a migration inserir some na primeira carga. Dado vem do pipeline; nos
  testes, cada teste insere o seu.
- Testes de banco seguem `PortugueseTextSearchTests.cs`: `PostgresFixture` cria um banco
  novo, o `DatabaseMigrator` aplica tudo, e o teste insere o que precisa.
- As métricas por tema estão nas views materializadas `dw.case_current_result` e
  `dw.theme_summary`. Depois de inserir dado no teste, rode
  `REFRESH MATERIALIZED VIEW dw.case_current_result` e depois
  `REFRESH MATERIALIZED VIEW dw.theme_summary`, nessa ordem.
- `theme_sk` é interno e muda a cada carga; o que sai da API é `theme_key`.
