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

## Proteção da `main` e das branches de US

No `API5-Backend` e no `API5-Frontend`, o fluxo é imposto pelo GitHub com **dois
rulesets** por repositório (Settings → Rulesets): `main rules` e `us* rules`. Os dois
estão com enforcement ativo e **lista de bypass vazia** — as regras valem para todos,
inclusive administradores. Os repositórios têm a mesma configuração; a única diferença é
o check de CI exigido.

Na prática: **ninguém dá push direto na `main` nem nas `usX`**. Todo código entra por
pull request, a partir de uma branch de task.

### `main rules` — a `main`

| Item | Configuração |
|---|---|
| Alvo | só a branch padrão, `main` |
| Pull request obrigatório | sim — não há push direto |
| Aprovações necessárias | 1 |
| Quem aprova | team **Code Reviewers**, em todos os arquivos (padrão `**`) |
| Métodos de merge permitidos | só **merge** (sem squash nem rebase) |
| Exclusão da branch | bloqueada |
| Force push | bloqueado |
| Status checks | obrigatórios (GitHub Actions): o CI do repositório (**`Backend checks`** ou **`Frontend checks`**) e **`Release label`** |
| Branch atualizada antes do merge | exigida — o PR precisa conter a `main` atual |

### `us* rules` — as branches de user story

| Item | Configuração |
|---|---|
| Alvo | `refs/heads/us*` (`us1`, `us2`…) |
| Pull request obrigatório | sim — não há push direto |
| Aprovações necessárias | **0**, sem reviewer obrigatório — a revisão fica no PR `usX` → `main` |
| Métodos de merge permitidos | só **merge** |
| Exclusão da branch | **livre** — a `usX` pode ser apagada depois de mergeada |
| Force push | bloqueado |
| Status checks | obrigatório: o CI do repositório (**`Backend checks`** ou **`Frontend checks`**); **sem** `Release label` |
| Branch atualizada antes do merge | exigida — o PR de task precisa conter a `usX` atual |

### Como fica o fluxo

| PR | O que o GitHub exige para o merge |
|---|---|
| task → `usX` | CI verde e branch atualizada; **sem aprovação** e sem label de release |
| `usX` → `main` | CI verde, `Release label` (um label `release:*` no PR), branch atualizada e **1 aprovação** do team Code Reviewers |

O team **Code Reviewers** (organização `Concord-API`) tem 3 membros: Victor Nogueira
(`victorgsnogueira`), Thiago (`thiagosabreu`) e Richard Leonardo Cordeiro
(`RichardCordeiro`). A aprovação de 1 pessoa do team basta, mas o PR precisa de pelo
menos uma aprovação de um membro do team.

Cada check é o job do `.github/workflows/ci.yml` do repositório: `Backend checks` (job
`tests`) e `Frontend checks` (job `checks`) — ver
[DevOps e infraestrutura](../06-operacao/03-devops-e-infra.md). O nome do check no
ruleset tem que ser idêntico ao `name:` do job; se o job for renomeado, os rulesets
precisam ser atualizados, senão o PR fica esperando um check que nunca roda.

O check `Release label` vem do workflow `release-label.yml`: falha se o PR não tiver
**exatamente um** label `release:*`. É ele que decide se o merge gera uma versão — ver
[Versionamento e releases](04-versionamento-e-releases.md). Ele só roda em PR para a
`main`, e por isso só o `main rules` o exige.

Como a branch do PR precisa estar atualizada com a base, cada merge numa `usX` obriga as
demais tasks abertas a atualizar e rodar o CI de novo. É o preço de testar sempre sobre
o código mais recente.

Não estão ativos: descarte de aprovações antigas quando entram novos commits, exigência
de aprovação do push mais recente por outra pessoa, resolução obrigatória de conversas,
Code Owners, commits assinados e criação de branch sem check (`do_not_enforce_on_create`
desligado).

> ⚠ **O que os rulesets não cobrem.**
> - As branches de task não têm regra própria: podem receber push direto e force push.
> - Sem "descartar aprovações antigas", um PR aprovado pode receber commits novos e
>   ser mergeado sem nova revisão.
> - A criação de uma `usX` nova pode ser recusada por ainda não ter check próprio. Se
>   acontecer, ligar "Do not require status checks on creation" no `us* rules`.
> - Só entram na regra branches cujo nome começa com `us`.

Os repositórios de documentação (`API-5`, `API-5-docs`) não foram verificados aqui.

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
| Pull request de `usX` → `main` | build + testes da aplicação, incluindo integração; exige label `release:*` |
| Merge na `main` | publica uma **release** (tag `vX.Y.Z`, zip e `.sha256`), salvo `release:none` — ver [Versionamento e releases](04-versionamento-e-releases.md) |

Os 24 testes de integridade do DW pertencem à carga manual, executada separadamente.
Não fazem parte do CI do backend ou do frontend. Os testes de integração da API usam
PostgreSQL descartável com schema e dados mínimos preparados pelos próprios testes,
sem executar raspagem, ETL ou NLP e sem acessar a homologação.

> O CI do `API5-Backend` e o do `API5-Frontend` disparam em PR e push para `main` e
> `us*`, então a tabela acima é coberta. O merge na `main` e nas `usX` é bloqueado
> se o check falhar. O `Release label` e a release seguem só a `main`.

## Estado nos repositórios (20/09/2026)

Nenhuma branch `usX` criada ainda em `API5-Backend` ou `API5-Frontend` — o padrão começa
a valer na primeira US. Os dois repositórios têm os rulesets `main rules` e `us* rules`
(ver [Proteção da `main` e das branches de US](#proteção-da-main-e-das-branches-de-us)).
