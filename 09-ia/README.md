# Desenvolver com IA

Contexto pronto para quem usa um assistente de IA (Claude Code ou outro) nos repositórios
do projeto. **Nada disso vai para dentro dos repositórios de código** — nenhum `.claude/`,
`CLAUDE.md` ou `AGENTS.md` nos repos do `Concord-API`. Fica só aqui, e cada um aponta o
seu assistente para cá.

| Arquivo | Para quê |
|---|---|
| [`CLAUDE.md`](CLAUDE.md) | contexto global: repositórios, fluxo de task/branch/commit, regras de código e de banco, decisões fechadas |
| [`repos/API5-Backend.md`](repos/API5-Backend.md) | estrutura, comandos e regras do backend |
| [`repos/API5-Frontend.md`](repos/API5-Frontend.md) | regras do frontend |
| [`repos/API5-Pipeline.md`](repos/API5-Pipeline.md) | módulos, comandos e regras do pipeline |
| [`skills/plano-task/SKILL.md`](skills/plano-task/SKILL.md) | skill que apresenta o plano de uma task antes de codar |

## Como usar com o Claude Code

Sem copiar nada para dentro do repositório de código. Os comandos abaixo são do
PowerShell, no Windows.

### 1. GitHub CLI

A skill lê a task no board com o `gh`, usando a conta de quem roda: atribui a task a você
e move o card para `In Progress` quando o desenvolvimento começa. Basta ser membro da org
`Concord-API` e dar ao `gh` o escopo de projetos (passo abaixo); o resto do caminho do
card (Review e Done) é a automação que faz.

```powershell
winget install --id GitHub.cli
```

Feche e abra o terminal, e faça o login:

```powershell
gh auth login
```

Responda **GitHub.com** → **HTTPS** → **Yes** (usar nas credenciais do git) →
**Login with a web browser** e cole o código na página que abrir.

Depois libere o escopo de projetos, para a skill conseguir mover o card:

```powershell
gh auth refresh -s project
```

Confira:

```powershell
gh issue list -R Concord-API/API-5 --search "1.2" --json number,title
```

Tem que voltar a task 1.2. Se der erro de acesso, a conta não está na org ou o login foi
feito com outra conta.

### 2. Esta wiki ao lado dos repositórios

A skill lê o detalhe da task em `Docs/08-backlog/tasks/`, então a wiki fica clonada numa
pasta `Docs`, na mesma pasta dos repositórios de código:

```powershell
git clone https://github.com/victorgsnogueira/API-5-docs Docs
```

### 3. Skill e contexto

```powershell
New-Item -ItemType Directory -Force "$HOME\.claude\skills"
Copy-Item -Recurse Docs\09-ia\skills\plano-task "$HOME\.claude\skills\plano-task"
```

Abra uma sessão nova do Claude Code e chame `/plano-task <id da task>`.

Para o assistente conhecer as regras, copie o conteúdo de `CLAUDE.md` (e o do
repositório em que vai trabalhar) para o seu `~/.claude/CLAUDE.md`, ou peça ao
assistente, no começo da sessão, para ler estes arquivos.

Quando a skill mudar aqui, rode `git pull` na pasta `Docs` e copie a pasta de novo.

Quando uma regra mudar, mude aqui primeiro; é esta pasta que os outros assistentes leem.
