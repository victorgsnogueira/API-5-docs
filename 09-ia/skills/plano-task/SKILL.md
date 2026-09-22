---
name: plano-task
description: Use quando o usuário chamar /plano-task (com ou sem o ID/número da task) para apresentar, antes de qualquer código, o plano de uma task do projeto Ratio (API-5) — nome da branch, estrutura de arquivos novos, o que muda em cada arquivo e a lista de commits.
---

# Plano da task

Entrega o plano de implementação de uma task **antes de tocar em qualquer arquivo**. O usuário aprova ou ajusta; só depois começa o desenvolvimento.

## Regras

- **Não crie, edite nem commite nada** ao rodar esta skill. Criar as branches locais é permitido só se o usuário pedir.
- **Atribua a issue a quem está rodando a skill**, logo depois de achá-la no board: `gh issue edit <número> -R Concord-API/API-5 --add-assignee @me`. É a única mudança que a skill faz; se a issue já tiver outro responsável, não o remova e avise no início do plano.
- Escreva em português, direto, sem enrolação.
- Se não vier o ID da task, use a task em andamento na conversa; se não houver, pergunte.

## Antes de montar o plano

1. **Leia a task no board**: `gh issue list -R Concord-API/API-5 --search "<id>" --json number,title,body,labels` e o detalhe em `Docs/08-backlog/tasks/` (repo pessoal `API-5-docs`).
2. **Leia o guia da task**, se existir: `Docs/08-backlog/tasks/guias/<id>.md`, e o `guias/README.md` (fluxo, dependências e regras de banco). As decisões e os alertas do guia valem para o plano: não as reabra, e se o código atual contradisser o guia, aponte a divergência nos pontos para decidir. Se uma dependência listada no guia ainda não estiver mergeada, diga isso logo no início do plano.
3. **Leia o código atual** do repositório afetado (backend `API5-Backend`, frontend `API5-Frontend` ou pipeline) — os arquivos que o plano vai tocar e os testes vizinhos. O plano descreve mudanças sobre o que existe, não sobre suposição.
4. **Respeite decisões já tomadas** na conversa e na memória (ex.: a API não gerencia usuários nem grants do banco).

## Formato da resposta

Siga exatamente estas seções, nesta ordem.

### 1. Task e branch

```
<id> <título da task>                      (#<número da issue>) · atribuída a @<login>

main
 └── us<N>                                  (us0 = Technical Foundation)
      └── <id>-<Titulo-da-task-com-hifens>
```

Nome da branch: ID da task + título do board, espaços viram hífen, sem acento, crase, aspas ou barra, sem referência à US.

### 2. Estrutura de arquivos novos

Árvore só com os caminhos relevantes, cada arquivo com um comentário curto à direita dizendo o que ele é:

```
Ratio/
├── Ratio.Application/Abstractions/
│   └── IAlgo.cs                      interface: Metodo(CancellationToken)
└── Ratio.Infrastructure.Tests/Algo/
    └── AlgoTests.cs                  cenário A, cenário B
```

### 3. O que muda em cada arquivo

Tabela com **todos** os arquivos tocados — novos e alterados:

| Arquivo | Novo/Alterado | O que muda |
|---|---|---|
| `Ratio.Api/Program.cs` | Alterado | chama `X` depois do `Build()`; falha → log crítico e não sobe |

Para arquivo novo, diga o que ele contém; para alterado, diga exatamente o que entra e o que sai. Testes: liste os cenários cobertos.

### 4. Commits

Lista numerada, na ordem em que vão acontecer, com o total no título (`### 4. Commits (5)`):

1. `chore: ...`
2. `test: ...`
3. `feat: ...`

Padrão:
- Tipos: `feat`, `fix`, `refactor`, `docs`, `test` (qualquer teste: unidade, integração, contrato, e2e), `chore`.
- TDD: cada comportamento vira um par `test:` → `feat:`/`fix:`.
- Só a linha do assunto, em inglês, curta. Sem corpo, sem `Co-Authored-By` de IA.
- Commits pequenos; nada de um commit gigante com a task inteira.

### 5. Pontos para decidir

Só o que realmente depende do usuário, cada um com a minha recomendação. Se não houver, escreva "Nenhum".

Termine com uma pergunta curta pedindo aprovação para começar pelo commit 1.

## Lembretes de implementação (para quando for aprovado)

- Código sem comentários; SQL sem comentários.
- Pendências são avisadas na conversa, nunca em comentário no código.
- Sem push até o usuário mandar.
