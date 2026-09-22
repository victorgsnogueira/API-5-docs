# Ambiente local

## Layout na máquina

```
D:/Desenvolvimento/fatec/API/API-5/
├── API-5/          backlog e critérios de aceite da org   (repo)
├── API5-Backend/   API .NET 10, dona do schema dw         (repo)
├── API5-Frontend/  frontend React                          (repo)
├── API5-Pipeline/  coleta e arquivo de carga do dw         (repo)
├── Docs/           esta wiki                               (repo)
├── scraping/       fase de descoberta — ⚠ histórico, não é mais o pipeline
└── prototipo/      protótipo antigo     (repo) — ⚠ não é referência
```

A pasta que os contém não é um repo — é só a convenção de tê-los lado a lado, que os
caminhos citados na wiki pressupõem. A `scraping/` foi o pipeline da fase de descoberta;
o pipeline de verdade agora é o `API5-Pipeline`.

Para montar o ambiente do zero:

```bash
mkdir API-5 && cd API-5
git clone https://github.com/Concord-API/API-5.git
git clone https://github.com/Concord-API/API5-Backend.git
git clone https://github.com/Concord-API/API5-Frontend.git
git clone https://github.com/Concord-API/API5-Pipeline.git
git clone https://github.com/victorgsnogueira/API-5-docs.git Docs
```

Vai usar um assistente de IA? O contexto pronto está em [Desenvolver com IA](../09-ia/README.md).

---

## Como o banco funciona agora

- A **API cria o schema `dw`** sozinha, na subida, por migration (DbUp). Ninguém aplica
  SQL à mão.
- O **dado vem do arquivo de carga** gerado pelo `API5-Pipeline` (`TRUNCATE` + `COPY` +
  `REFRESH`, numa transação), aplicado **depois** que a API criou o schema.
- O banco só precisa existir, com as extensões `unaccent` e `pg_trgm` no schema `public`.

---

## Backend .NET

### Pré-requisitos

.NET SDK 10, Docker Desktop (os testes de integração sobem Postgres via Testcontainers).
Visual Studio 2022 ou VS Code com C# Dev Kit.

### Banco local

Um `postgres:16` com locale ICU `pt-BR` (igual à produção, sem pgvector):

```bash
docker run -d --name ratio-dev --restart unless-stopped -p 5433:5432 -v ratio_dev_data:/var/lib/postgresql/data -e POSTGRES_DB=ratio -e POSTGRES_USER=ratio -e POSTGRES_PASSWORD=<senha> -e POSTGRES_INITDB_ARGS="--locale-provider=icu --icu-locale=pt-BR --encoding=UTF8 --locale=C.utf8" postgres:16
docker exec ratio-dev psql -U ratio -d ratio -c "CREATE EXTENSION IF NOT EXISTS unaccent; CREATE EXTENSION IF NOT EXISTS pg_trgm;"
```

### Comandos

```bash
cd API5-Backend/Ratio
export ConnectionStrings__Ratio="Host=localhost;Port=5433;Database=ratio;Username=ratio;Password=<senha>"
dotnet run --project Ratio.Api      # na primeira subida cria o schema dw
dotnet test Ratio.slnx              # o ciclo do TDD — ver 07-justificativas/03-tdd.md
```

**Connection string só por variável de ambiente**, nunca em `appsettings.json`
versionado. Uma credencial só, que também migra ([D-36](02-decisoes-e-riscos.md#d-36--uma-única-credencial-para-a-api-sem-separar-migração-e-leitura)).

Com o banco recém-criado, `/health/ready` responde `503` com `reason: sem carga
publicada` — é o esperado até o arquivo de carga ser aplicado.

### Dado para desenvolver

> ⚠ **Ainda não há arquivo de carga do schema novo distribuído para o time.** O dump
> antigo é do modelo da fase de descoberta e **não serve**: a API recusa subir num `dw`
> que ela não criou. O primeiro arquivo sai da task `0.71` (comando do pipeline); onde
> ele fica disponível para o time ainda será decidido. Até lá, os testes de cada
> repositório preparam o próprio banco descartável e não precisam de carga.

Quando existir, aplicar é um comando, com a API já tendo subido uma vez:

```bash
docker exec -i ratio-dev psql -U ratio -d ratio -v ON_ERROR_STOP=1 < ratio-load.sql
```

> **Não aponte a API nova para a homologação por enquanto.** A homologação ainda está no
> modelo da fase de descoberta, sem o journal de migration; a API nova não sobe ali. Ela
> é refeita no fluxo novo na task `0.53` (Sprint 2).

---

## Pipeline de carga

```bash
cd API5-Pipeline
pip install -r requirements.txt
ruff check .
pytest                                # Docker ligado — os testes sobem Postgres descartável
```

O pipeline roda do nosso lado, nunca no cliente. Ele usa o **banco de carga** (com os
schemas de trabalho `raw`, `staging` e `etl`), cujo `dw` é criado pela mesma API.
Configuração só por variável de ambiente (`DATABASE_URL`). Módulos e regras em
[`API5-Pipeline`](../09-ia/repos/API5-Pipeline.md).

---

## Frontend React

### Pré-requisitos

Node **20+** e npm 11.

### Comandos

```bash
cd API5-Frontend/ratio
npm install
npm run dev          # http://localhost:5173
npm run lint
npm run typecheck
npm run build
```

É um monorepo Turborepo: os comandos na raiz `ratio/` rodam em todos os workspaces
(`apps/web`, `packages/ui`). Acrescente `http://localhost:5173` à lista explícita de CORS
do backend. Estrutura e stack: [Frontend React](../02-arquitetura/03-frontend-react.md).

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

Backend:

| Variável | Para quê |
|---|---|
| `ConnectionStrings__Ratio` | Postgres — uma credencial só, que também migra ([D-36](02-decisoes-e-riscos.md#d-36--uma-única-credencial-para-a-api-sem-separar-migração-e-leitura)) |
| `Cors__AllowedOrigins` | lista explícita, nunca `*` |

Pipeline:

| Variável | Para quê |
|---|---|
| `DATABASE_URL` | banco de carga |

Frontend:

| Variável | Para quê |
|---|---|
| `VITE_API_URL` | URL da API — embutida no build |

A chave do DataJud e os parâmetros de recorte (tribunais, áreas, cotas) são do
**pipeline de carga**, não do backend — chegam por variável de ambiente na task `0.71`.

Em produção, definidas na máquina do cliente ([Implantação no cliente](04-implantacao-no-cliente.md)). **Nenhum segredo no
repositório.**

---

## Problemas comuns

| Sintoma | Causa provável | Solução |
|---|---|---|
| API sobe mas `/health/ready` dá `503` | o campo `reason` diz qual | `sem carga publicada` → aplicar o arquivo de carga; `banco inacessível` → conferir connection string e se o contêiner está de pé |
| `docker` não conecta (`dockerDesktopLinuxEngine`) | Docker Desktop fechado | abrir o Docker Desktop e `docker start ratio-dev` |
| Porta 5433 ocupada | outro contêiner ou Postgres local | trocar a porta publicada |
| Frontend com erro de CORS | origem fora da lista | acrescentar em `Cors__AllowedOrigins` e reiniciar a API |
| API não sobe: log crítico de migration | o `dw` já existia sem ter sido criado pela API, ou uma migration quebrou no meio | banco novo e vazio; nunca criar o `dw` à mão |
| API não sobe: `text search dictionary "public.unaccent" does not exist` | extensão fora do schema `public` | `CREATE EXTENSION unaccent` no `public` |
| Coletor do DataJud falhando intermitente (504/429) | limite de taxa | esperado; o coletor tem retry com backoff — rode de novo, é idempotente |
| Coletor parece travado no Windows | saída em buffer | `python -u` |
| Contêiner do protótipo brigando por porta | ele ficou de pé | `docker compose down` na pasta dele |
