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

Sem copiar nada para dentro do repositório de código:

- **Contexto global:** copie o conteúdo de `CLAUDE.md` (e o do repositório em que vai
  trabalhar) para o seu `~/.claude/CLAUDE.md`, ou peça ao assistente, no começo da
  sessão, para ler estes arquivos pelo link do GitHub.
- **Skill:** copie a pasta `skills/plano-task/` para `~/.claude/skills/plano-task/`. Depois
  é só chamar `/plano-task <id da task>`.

Quando uma regra mudar, mude aqui primeiro; é esta pasta que os outros assistentes leem.
