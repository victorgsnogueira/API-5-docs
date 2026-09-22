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
| 1.4 | Backend | [1.4](1.4.md) | 1.3 |
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

## Pipeline — o que a carga real precisa

| Task | O quê |
|---|---|
| [0.72](https://github.com/Concord-API/API-5/issues/139) | gravar a linha de `dw.strength_config` na carga — sem ela a nota e o piso de `n` saem vazios |
| [0.73](https://github.com/Concord-API/API-5/issues/140) | `REFRESH` das views na ordem de dependência — em ordem alfabética a `theme_strength` quebra a carga |

## Fluxo de toda task

1. Atribua a issue a você no board.
2. Rode `/plano-task <id>` ([como instalar a skill](../../../09-ia/README.md)) e siga o
   plano aprovado.
3. Crie a branch a partir da **`us1`** do repositório da task. O nome é o ID mais o
   título do board com hífens, sem acento nem crase. A `us1` já existe nos três
   repositórios; tasks `0.Y` saem da `us0`.
4. Commits em TDD: `test:` vermelho pelo motivo certo, depois `feat:`/`fix:`. Só a linha
   do assunto, em inglês. Nenhum comentário no código, no SQL ou no teste.
5. PR para a `us1`, só com título. O board se move sozinho.

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
