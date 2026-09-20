# Padrão de branches

**Modelo adotado:** Feature Branching, com uma branch por user story.

Vale para os repositórios de código (`API5-Backend`, `API5-Frontend`). Os repositórios
de documentação seguem [um padrão próprio](#repositórios-de-documentação).

---

## A estrutura

```
main                                          estável · recebe a US concluída
 │
 ├── us1                                      base da user story 1
 │    ├── RATIO-35-0.1-Implementar-documentação-API
 │    ├── RATIO-35-0.2-…
 │    └── RATIO-36-0.1-…
 │
 ├── us2                                      base da user story 2
 │    └── …
 │
 └── us3
      └── …
```

Três níveis, cada um com um papel:

| Nível | Nome | Nasce de | Volta para | Quando |
|---|---|---|---|---|
| Principal | `main` | — | — | sempre existe |
| User story | `us1`, `us2`, `us3`… | `main` | `main` | ao concluir a US, após revisão e aprovação |
| Task | `<task-pai>-<subtask>-<nome-da-task>` | a branch da US | a branch da US | ao concluir a task |

### `main`

Branch principal e **estável** do projeto. Recebe merge a cada conclusão de uma user
story, **após revisão e aprovação**. Ninguém trabalha direto nela.

### `usX` — uma por user story

Cada user story tem sua própria branch base (`us1`, `us2`, `us3`…). É nela que se
integram todas as funcionalidades desenvolvidas durante aquele ciclo.

### Branch de task

Para cada nova funcionalidade ou correção, cria-se uma branch específica **a partir da
branch da US** a que a task pertence. O nome vem da task no board:

```
<chave-da-task-pai>-<número-da-subtask>-<Nome-da-task-com-hífen-no-lugar-de-espaço>
```

Exemplo:

```
RATIO-35-0.1-Implementar-documentação-API
│        │   └─ nome da task, espaços viram hífen
│        └──── subtask
└───────────── task pai no board
```

> **Observação.** O git aceita acento em nome de branch, mas algumas ferramentas não
> lidam bem com isso — URLs de pull request, terminais do Windows, gatilhos de CI. Se
> aparecer problema, a saída é grafar sem acento (`Implementar-documentacao-API`).

---

## Repositórios de documentação

As branches dos repositórios de documentação **não são ligadas a tasks**, então não há
chave de board para usar. O padrão é:

```
RATIO-DOCS-<Descrição-do-que-está-sendo-feito>
```

Exemplos:

```
RATIO-DOCS-Definition-of-Done
RATIO-DOCS-Product-Backlog
```

Vale para `API-5` e `API-5-docs` (esta wiki). Ver
[Repositórios](../00-visao-geral/03-repositorios.md).

---

## Por que este modelo

### Por que uma branch por user story

**A US é a unidade de entrega do time.** A sprint é planejada, acompanhada e apresentada
em user stories — faz sentido que o repositório seja organizado na mesma unidade. A
`main` passa a refletir exatamente o conjunto de USs entregues, nem mais nem menos.

**Integração antes da `main`.** As tasks de uma mesma US costumam depender umas das
outras (o endpoint precisa do repositório, a tela precisa do endpoint). A branch da US é
o lugar onde elas se encontram e são testadas juntas, sem expor a `main` a uma US pela
metade.

**`main` sempre apresentável.** Como só entra US concluída e revisada, a `main` pode ser
demonstrada a qualquer momento — o que importa num projeto com entregas avaliadas por
sprint.

### Por que uma branch por task, nomeada pelo board

**Rastreabilidade de ponta a ponta.** O nome da branch leva a chave da task
(`RATIO-35`). De qualquer commit dá para chegar à task no board, e da task ao código —
sem planilha de controle paralela.

**Trabalho isolado.** Cada pessoa trabalha na sua task sem pisar no código de outra. Um
problema numa task não bloqueia as demais da mesma US.

**Revisão em porções pequenas.** Uma branch por task gera pull requests do tamanho de
uma task — revisáveis de verdade, em vez de um diff gigante no fim da US.

### Por que um padrão diferente para documentação

Documentação não passa pelo board como task, então não há chave para usar. O prefixo
`RATIO-DOCS-` resolve duas coisas: deixa claro, pelo nome, que a branch é de
documentação, e mantém a mesma raiz `RATIO-` do resto do projeto.

---

## Relação com o CI/CD

A estrutura de branches define onde o pipeline roda. Ver
[DevOps e infraestrutura](../06-operacao/03-devops-e-infra.md).

| Evento | O que o pipeline deve fazer |
|---|---|
| Pull request de task → `usX` | build + testes — **teste falhando bloqueia o merge** ([TDD](03-tdd.md)) |
| Pull request de `usX` → `main` | build + testes da aplicação, incluindo integração |
| Merge na `main` | gera o **pacote de versão** para o cliente |

Os 24 testes de integridade do DW pertencem à carga manual, executada separadamente.
Não fazem parte do CI do backend ou do frontend. Os testes de integração da API usam
PostgreSQL descartável com schema e dados mínimos preparados pelos próprios testes,
sem executar raspagem, ETL ou NLP e sem acessar a homologação.

> ⚠ O CI do `API5-Frontend` hoje dispara só em PR para `main`. Para cobrir a tabela
> acima, o gatilho precisa incluir as branches `us*`. O `API5-Backend` ainda não tem CI.

## Estado nos repositórios (19/09/2026)

`API5-Backend` e `API5-Frontend` têm só a `main`, com um commit cada. Nenhuma branch
`usX` criada ainda — o padrão começa a valer na primeira US.
