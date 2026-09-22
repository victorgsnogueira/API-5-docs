# API5-Frontend

Leia antes o [`CLAUDE.md`](../CLAUDE.md) global.

## Regras próprias

- Chamadas à API por caminho relativo (`/api/...`); nada de URL absoluta no build.
- Toda resposta da API passa por um schema `zod`: resposta fora do contrato quebra no
  parse, não no meio da tela.
- Testes com Vitest; a API é simulada com MSW enquanto o endpoint real não existe.
- Fontes self-hosted, nenhum recurso de CDN (produção sem internet).
- Nenhum cálculo de número na tela: o frontend exibe o que a API devolve.

## Comandos

Os scripts ficam em `ratio/package.json`: `format:check`, `lint`, `typecheck`, `test:ci`, `build`.
