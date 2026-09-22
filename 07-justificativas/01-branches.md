# Padrão de branches

**Modelo adotado:** Feature Branching, com uma branch por user story.

Vale para os repositórios de código (`API5-Backend`, `API5-Frontend`). Os repositórios
de documentação seguem [um padrão próprio](#repositórios-de-documentação).

---

## A estrutura

```
main                                          estável · recebe a US concluída
 │
 ├── us0                                      Technical Foundation (tasks 0.Y)
 │    ├── 0.12-Create-the-dw-schema-migration-and-apply-it-on-API-startup
 │    └── 0.14-Set-up-the-frontend-skeleton-against-the-API-contract
 │
 ├── us1                                      base da US-01
 │    ├── 1.1-Add-the-Portuguese-text-search-configuration-with-accent-removal
 │    └── 1.5-Create-GET-api-themes-returning-themes-never-cases
 │
 └── us9                                      base da US-09
      └── …
```

Três níveis, cada um com um papel:

| Nível | Nome | Nasce de | Volta para | Quando |
|---|---|---|---|---|
| Principal | `main` | — | — | sempre existe |
| User story | `us0`, `us1`, `us2`… | `main` | `main` | ao concluir a US, após revisão e aprovação |
| Task | `<id-da-task>-<Nome-da-task>` | a branch da US | a branch da US | ao concluir a task |

### `main`

Branch principal e **estável** do projeto. Recebe merge a cada conclusão de uma user
story, **após revisão e aprovação**. Ninguém trabalha direto nela.

### `usX` — uma por user story

Cada user story tem sua própria branch base, com o número da US sem zero à esquerda: a
US-01 é `us1`, a US-09 é `us9`, a US-21 é `us21`. É nela que se integram todas as tasks
daquela US.

**`us0` é o Technical Foundation.** As tasks `0.Y` (schema, pipeline, esqueleto do
frontend) não pertencem a nenhuma US, mas precisam de uma base protegida como as outras.
O número 0 casa com a numeração delas, e o ruleset `us*` já cobre a branch.

### Branch de task

Para cada nova funcionalidade ou correção, cria-se uma branch específica **a partir da
branch da US** a que a task pertence. O nome é o ID da task seguido do título dela no
board:

```
<id-da-task>-<Nome-da-task-com-hífen-no-lugar-de-espaço>
```

Exemplo:

```
0.12-Create-the-dw-schema-migration-and-apply-it-on-API-startup
│    └─ título da task, espaços viram hífen
└────── ID da task: 0.12 é do Technical Foundation (us0)
```

**O nome não repete a US.** O ID já diz a que US a task pertence (`1.1` é da US-01,
`9.4` é da US-09, `0.Y` é do Technical Foundation) e é único no projeto inteiro. Também
não leva o número da issue: a referência é o ID da task, que é o mesmo no board e nas
[tasks do projeto](../08-backlog/tasks/README.md).

Ao montar o nome:

- sem acento, crase, aspas ou barra: `` `GET /api/themes` `` vira `GET-api-themes`;
- sem espaço: cada espaço vira um hífen;
- o título pode ser encurtado se ficar longo demais, desde que o ID continue no começo.

> **Por que sem acento e sem símbolo.** O git aceita, mas algumas ferramentas não lidam
> bem com isso: URLs de pull request, terminais do Windows, gatilhos de CI.

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
| Branch atualizada antes do merge | **desligada** no `API5-Backend` e no `API5-Pipeline`; ainda ligada no `API5-Frontend` — ver a nota abaixo |

### `us* rules` — as branches de user story

| Item | Configuração |
|---|---|
| Alvo | `refs/heads/us*` (`us0`, `us1`, `us2`…) |
| Pull request obrigatório | sim — não há push direto |
| Aprovações necessárias | **0**, sem reviewer obrigatório — a revisão fica no PR `usX` → `main` |
| Métodos de merge permitidos | só **merge** |
| Exclusão da branch | **livre** — a `usX` pode ser apagada depois de mergeada |
| Force push | bloqueado |
| Status checks | obrigatório: o CI do repositório (**`Backend checks`** ou **`Frontend checks`**); **sem** `Release label` |
| Branch atualizada antes do merge | **desligada** no `API5-Backend` e no `API5-Pipeline`; ainda ligada no `API5-Frontend` — ver a nota abaixo |

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

> **Por que "branch atualizada antes do merge" foi desligada no `API5-Backend` e no
> `API5-Pipeline`.** A `us0` é permanente — recebe merge de `main` e manda merge pra
> `main` repetidas vezes ao longo do projeto, uma por task do Technical Foundation.
> Cada merge `us0 → main` cria um commit de merge novo na `main`, que a `us0` só
> carrega de volta se alguém sincronizar `main → us0` **antes** de começar a próxima
> task. Isso já não aconteceu uma vez, e travou os dois lados ao mesmo tempo: a `us0`
> não conseguia mergear na `main` porque a `main` tinha um commit que ela não tinha, e
> mergear `main` de volta na `us0` exigia o oposto — nenhum admin conseguia passar por
> cima, porque a lista de bypass está vazia. A saída foi desligar essa exigência: os
> checks continuam obrigatórios sobre o commit real do PR, só deixou de exigir que a
> branch já contenha, letra por letra, o último commit da outra ponta. Isso é
> recomendação do próprio GitHub para branches de integração de longa duração como a
> `us0`, que recebem merge nos dois sentidos — diferente de uma `usX` normal, que só
> recebe de `main` e só devolve uma vez, ao terminar.
>
> **O `API5-Frontend` ainda não passou por isso** porque a `us0` dele só recebeu uma
> task até agora. Fica pendente desligar lá também, antes que o mesmo problema apareça.

Como a branch do PR precisa estar atualizada com a base **nos repositórios onde essa
exigência continua ligada**, cada merge numa `usX` obriga as demais tasks abertas a
atualizar e rodar o CI de novo.

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

**Rastreabilidade de ponta a ponta.** O nome da branch começa pelo ID da task
(`0.12`, `1.1`). De qualquer commit dá para chegar à task no board, e da task ao código —
sem planilha de controle paralela.

**Trabalho isolado.** Cada pessoa trabalha na sua task sem pisar no código de outra. Um
problema numa task não bloqueia as demais da mesma US.

**Revisão em porções pequenas.** Uma branch por task gera pull requests do tamanho de
uma task — revisáveis de verdade, em vez de um diff gigante no fim da US.

> **Uma task é uma unidade só, mesmo que o trabalho dela tenha várias fases.** Uma task
> grande (a `0.13`, por exemplo, com 99h) pode ser implementada em várias etapas
> internas, cada uma com seus próprios commits `test:`/`feat:`. Isso **não** vira várias
> branches nem vários merges pra `usX`: é tudo na mesma branch de task, e ela só
> mergeia pra `usX`/`us0` **uma vez**, quando a task inteira está pronta — do mesmo
> jeito que a `usX` só mergeia pra `main` quando a US inteira termina. Mergear uma fase
> por vez fecha a issue da task antes da hora e obriga a reabrir na mão.

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

Os 37 testes de integridade do DW pertencem à carga manual, executada separadamente.
Não fazem parte do CI do backend ou do frontend. Os testes de integração da API usam
PostgreSQL descartável com schema e dados mínimos preparados pelos próprios testes,
sem executar raspagem, ETL ou NLP e sem acessar a homologação.

> O CI do `API5-Backend` e o do `API5-Frontend` disparam em PR e push para `main` e
> `us*`, então a tabela acima é coberta. O merge na `main` e nas `usX` é bloqueado
> se o check falhar. O `Release label` e a release seguem só a `main`.

## Estado nos repositórios (22/09/2026)

Os três repositórios de código (`API5-Backend`, `API5-Frontend`, `API5-Pipeline`) têm os
rulesets `main rules` e `us* rules` (ver [Proteção da `main` e das branches de
US](#proteção-da-main-e-das-branches-de-us)). O Technical Foundation (`us0`) já entregou
tasks em todos os três:

| Repositório | Tasks já mergeadas na `us0` | Na `main`? |
|---|---|---|
| `API5-Backend` | `0.69`, `0.12` | sim |
| `API5-Frontend` | `0.69` | sim |
| `API5-Pipeline` | `0.13` | esperando aprovação |

Nenhuma branch `usX` (US de verdade, `us1` em diante) foi criada ainda — o padrão começa
a valer na primeira US.
