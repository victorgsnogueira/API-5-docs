# API5-Backend

Leia antes o [`CLAUDE.md`](../CLAUDE.md) global.

## Estrutura

| Projeto | Papel |
|---|---|
| `Ratio.Domain` | tipos de domínio |
| `Ratio.Application` | portas (`Abstractions/`, ex.: `IDatabaseMigrator`, `ILastExtractionReader`) |
| `Ratio.Infrastructure` | adapters: Dapper/Npgsql, `Migrations/` (DbUp + scripts `.sql` embutidos) |
| `Ratio.Api` | host: `Program.cs`, `Controllers/`, `Hosting/` (hosted services, ex.: migração na subida) |

- Dapper não é ORM: classes são formato de consulta. Cálculo de score fica no banco, não em C#.
- `Ratio.Api` é o host; o que roda no ciclo de vida da aplicação fica em `Hosting/`.

## Comandos

```bash
cd Ratio
dotnet restore Ratio.slnx --locked-mode
dotnet format Ratio.slnx --verify-no-changes --no-restore
dotnet build Ratio.slnx --configuration Release --no-restore
dotnet test Ratio.slnx --configuration Release --no-build --no-restore
```

Testes de infraestrutura precisam do Docker rodando (Testcontainers, `postgres:16` com ICU `pt-BR`).

## Regras próprias

- Pacote novo: versão em `Directory.Packages.props`, referência sem versão no `.csproj`,
  e os `packages.lock.json` atualizados (o CI restaura com `--locked-mode`).
- `TreatWarningsAsErrors` está ligado.
- Migration nova entra na lista ordenada `Scripts` de `DatabaseMigratorTests`.
- Configuração só por variável de ambiente (`ConnectionStrings__Ratio`); nada de senha em arquivo.
