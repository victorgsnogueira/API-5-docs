# Ambiente local

## Layout na máquina

```
D:/Desenvolvimento/fatec/API/API-5/
├── API-5/          documentação do SM   (repo)
├── API5-Backend/   backend .NET 8       (repo)
├── API5-Frontend/  frontend React       (repo, vazio)
├── Docs/           esta wiki            (repo)
└── prototipo/      protótipo antigo     (repo) — ⚠ não é referência
```

São **cinco repositórios git independentes**. A pasta que os contém não é um repo — é só
a convenção de tê-los lado a lado, que os caminhos citados na wiki pressupõem.

Para montar o ambiente do zero:

```bash
mkdir API-5 && cd API-5
git clone https://github.com/Concord-API/API-5.git
git clone https://github.com/Concord-API/API5-Backend.git
git clone https://github.com/Concord-API/API5-Frontend.git
git clone https://github.com/victorgsnogueira/API-5-docs.git Docs
```

O protótipo só se você [precisar consultá-lo](#protótipo-antigo--só-se-você-precisar-consultá-lo).

---

## Backend .NET

### Pré-requisitos

.NET SDK 8, Docker Desktop. Visual Studio 2022 ou VS Code com C# Dev Kit.

### Comandos

```bash
cd API5-Backend/Ratio
dotnet restore
dotnet build
dotnet run --project Ratio.Api
dotnet test
```

### Banco

Suba um Postgres local em contêiner. **Não use a porta 5432** se você já tem um Postgres
instalado na máquina:

```bash
docker run -d --name ratio-db -p 5433:5432 -e POSTGRES_DB=ratio -e POSTGRES_USER=ratio -e POSTGRES_PASSWORD=ratio -e LANG=pt_BR.utf8 postgres:16-alpine
```

Connection string para desenvolvimento:

```
Host=localhost;Port=5433;Database=ratio;Username=ratio;Password=ratio
```

**Por variável de ambiente, nunca em `appsettings.json` versionado** — é a mesma
disciplina que o [Coolify](03-devops-e-infra.md) exige em produção.

Não há connection string configurada ainda; ver
[Backend .NET](../02-arquitetura/02-backend-dotnet.md#estado-atual-e-primeiras-tarefas).

### Rodar o ETL

`Ratio.Etl` é console app:

```bash
dotnet run --project Ratio.Etl
```

Ele **não** roda junto com a API, de propósito — ver
[ETL](../02-arquitetura/05-etl-e-nlp.md#agendamento).

---

## Frontend React

Repositório vazio. Para começar:

```bash
cd API5-Frontend
npm create vite@latest . -- --template react-ts
npm install
npm run dev
```

Sobe em `http://localhost:5173`. Acrescente essa origem à lista explícita de CORS do
backend. Estrutura sugerida: [Frontend React](../02-arquitetura/03-frontend-react.md).

---

## Protótipo antigo — só se você precisar consultá-lo

> ⚠ O `prototipo/` **não é fonte de verdade**. Não desenvolva contra ele, não porte
> código dele, não aponte o frontend novo para a API dele. Ver
> [Protótipo](../05-prototipo/01-prototipo-referencia.md).

Se ainda assim precisar rodá-lo para inspecionar comportamento de alguma fonte:

```bash
cd prototipo/backend
cp .env.example .env
docker compose up -d
curl http://localhost:8000/api/saude
```

Sobe Postgres na 5433 e FastAPI na 8000; Swagger em <http://localhost:8000/docs>. O
banco sobe vazio — a carga é comando à parte, descrito no `README.md` dele.

Derrubar quando terminar, para não competir por porta com o ambiente novo:

```bash
cd prototipo/backend && docker compose down
```

---

## Variáveis de ambiente

O que o backend .NET vai precisar. Nada disso está configurado ainda:

| Variável | Para quê |
|---|---|
| `ConnectionStrings__Ratio` | Postgres |
| `Cors__AllowedOrigins` | lista explícita, nunca `*` |
| `DataJud__ApiKey` | chave pública do CNJ, sobrescritível |
| `Etl__Courts` | `tjsp,tjrj,tjmg` |
| `Etl__Subjects` | códigos de assunto da TPU do recorte |
| `Etl__LimitPerCombination` | teto por combinação |

Em produção, todas injetadas pelo [Coolify](03-devops-e-infra.md). **Nenhum segredo no
repositório.**

---

## Problemas comuns

| Sintoma | Causa provável | Solução |
|---|---|---|
| API sobe mas `/health/ready` falha | banco vazio ou inacessível | conferir connection string e se a carga rodou |
| Porta 5433 ocupada | outro contêiner ou Postgres local | trocar a porta publicada |
| Frontend com erro de CORS | origem fora da lista | acrescentar em `Cors__AllowedOrigins` e reiniciar a API |
| Migration nova não aplicou | runner não configurado | ver [DevOps](03-devops-e-infra.md) |
| Busca não acha com acento | `unaccent` não instalado | conferir se a migration de extensões rodou |
| ETL falhando intermitente | limite de taxa do DataJud | esperado; o cliente precisa de retry com backoff |
| Contêiner do protótipo brigando por porta | ele ficou de pé | `docker compose down` na pasta dele |
