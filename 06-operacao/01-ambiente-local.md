# Ambiente local

## Layout na máquina

```
D:/Desenvolvimento/fatec/API/API-5/
├── API-5/          documentação do SM   (repo)
├── API5-Backend/   backend .NET 8       (repo)
├── API5-Frontend/  frontend React       (repo)
├── Docs/           esta wiki            (repo)
├── scraping/       pipeline de carga do DW   ⚠ FORA de repositório
├── prototipo-prod - versao 202609/   protótipo de dados (fora de repositório)
└── prototipo/      protótipo antigo     (repo) — ⚠ não é referência
```

São **cinco repositórios git independentes**, mais duas pastas soltas. A `scraping/` é
o pipeline que produziu o DW — **não está versionada** em lugar nenhum
([R-15](02-decisoes-e-riscos.md#r-15--o-pipeline-de-carga-não-está-versionado-)). A pasta que os contém não é um repo — é só
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

.NET SDK 8 (10 quando o [R-14](02-decisoes-e-riscos.md#r-14--net-8-sai-de-suporte-durante-o-projeto-) for resolvido),
Docker Desktop (os testes de integração sobem Postgres via Testcontainers). Visual Studio
2022 ou VS Code com C# Dev Kit.

### Comandos

```bash
cd API5-Backend/Ratio
dotnet restore
dotnet build
dotnet run --project Ratio.Api
dotnet test          # o ciclo do TDD — ver 07-justificativas/03-tdd.md
```

### Banco

O banco da carga roda em contêiner (Postgres 16 + pgvector, porque o NLP grava
embeddings) com locale ICU `pt-BR`. Produção não tem pgvector — ver
[Carga × produção](../02-arquitetura/04-data-warehouse.md#carga--produção--os-três-bancos). Inventário completo do que está instalado em
[Data Warehouse](../02-arquitetura/04-data-warehouse.md#o-que-está-instalado-no-banco).

```bash
docker run -d --name api5-dw --restart unless-stopped -p 5432:5432 -v api5_dw_data:/var/lib/postgresql/data -e POSTGRES_DB=api5_dw -e POSTGRES_USER=dw_admin -e POSTGRES_PASSWORD=<senha> -e POSTGRES_INITDB_ARGS="--locale-provider=icu --icu-locale=pt-BR --encoding=UTF8 --locale=C.utf8" pgvector/pgvector:pg16
```

Depois, aplicar as migrations (criam schemas e extensões `vector`, `pg_trgm`,
`unaccent`):

```bash
for f in scraping/sql/0*.sql; do docker exec -i api5-dw psql -U dw_admin -d api5_dw -v ON_ERROR_STOP=1 < "$f"; done
```

> Se já houver um Postgres instalado na máquina ocupando a 5432, publique em outra
> porta (`-p 5433:5432`) e ajuste a connection string.

Connection string para desenvolvimento:

```
Host=localhost;Port=5432;Database=api5_dw;Username=dw_admin;Password=<senha>
```

**Por variável de ambiente, nunca em `appsettings.json` versionado** — é a mesma
disciplina que a [implantação no cliente](04-implantacao-no-cliente.md) exige — a
configuração de produção é da máquina dele, não do build.

**Banco vazio?** Ou roda a [carga manual](../02-arquitetura/05-etl-e-nlp.md#carga-manual--o-processo)
inteira, ou restaura um dump de quem já tem a base:

```bash
docker cp dw.dump api5-dw:/tmp/dw.dump
docker exec api5-dw pg_restore -U dw_admin -d api5_dw --clean --if-exists -n dw /tmp/dw.dump
```

### Só vai desenvolver API ou frontend?

**Não precisa do banco da carga.** Aponte para a **homologação**, pela rede Tailscale do
time, com o usuário somente leitura:

```
Host=<nome-da-máquina-na-tailnet>;Port=5433;Database=ratio;Username=ratio_api;Password=<pedir ao time>
```

É o mesmo `dw` que vai para produção, sem pgvector e sem superusuário — se a API
funciona ali, funciona no cliente. Senha nunca vai para o repositório nem para esta wiki.

Para trabalhar offline, suba um `postgres:16` local e restaure o dump do `dw` (~15 MB)
do mesmo jeito que o [`publish_dw.sh`](../02-arquitetura/05-etl-e-nlp.md#subir-para-produção) faz.

### Rodar a carga

A carga não faz parte do backend. É o pipeline Python em `scraping/`, rodado à mão —
passo a passo em [Carga manual](../02-arquitetura/05-etl-e-nlp.md#carga-manual--o-processo).

Pré-requisitos: **Python 3.12** e

```bash
pip install psycopg2-binary requests beautifulsoup4 lxml sentence-transformers scikit-learn numpy
```

O `sentence-transformers` baixa o modelo (`paraphrase-multilingual-MiniLM-L12-v2`,
~470 MB) na primeira execução e depois roda offline, em CPU.

> No Windows use `python -u` nos coletores longos — sem isso a saída fica em buffer e o
> log parece travado.

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

O que o backend .NET vai precisar. Nada disso está configurado ainda:

| Variável | Para quê |
|---|---|
| `ConnectionStrings__Ratio` | Postgres — em produção, o papel **somente leitura** `ratio_api` |
| `Cors__AllowedOrigins` | lista explícita, nunca `*` |

Frontend:

| Variável | Para quê |
|---|---|
| `VITE_API_URL` | URL da API — embutida no build |

A chave do DataJud e os parâmetros de recorte (tribunais, assuntos, teto) são do
**pipeline de carga**, não do backend — hoje são argumentos dos scripts.

Em produção, definidas na máquina do cliente ([Implantação no cliente](04-implantacao-no-cliente.md)). **Nenhum segredo no
repositório.**

---

## Problemas comuns

| Sintoma | Causa provável | Solução |
|---|---|---|
| API sobe mas `/health/ready` falha | banco vazio ou inacessível | conferir connection string; restaurar um dump |
| `docker` não conecta (`dockerDesktopLinuxEngine`) | Docker Desktop fechado | abrir o Docker Desktop e `docker start api5-dw` |
| Porta 5433 ocupada | outro contêiner ou Postgres local | trocar a porta publicada |
| Frontend com erro de CORS | origem fora da lista | acrescentar em `Cors__AllowedOrigins` e reiniciar a API |
| Migration nova não aplicou | não há controle de versão aplicada | aplicar o arquivo com `psql -v ON_ERROR_STOP=1` |
| Busca não acha com acento | `unaccent` não instalado | conferir se a migration de extensões rodou |
| Coletor do DataJud falhando intermitente (504/429) | limite de taxa | esperado; o coletor tem retry com backoff — rode de novo, é idempotente |
| Coletor parece travado no Windows | saída em buffer | `python -u` |
| Contêiner do protótipo brigando por porta | ele ficou de pé | `docker compose down` na pasta dele |
