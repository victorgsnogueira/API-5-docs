# DevOps e infraestrutura

> **Requisito obrigatório do projeto.** DevOps aqui não é "nice to have": o desafio
> exige CI/CD, deploy automático, documentação, testes automatizados validando
> integridade dos dados, e monitoramento. Vale nota tanto quanto o produto.
>
> **Os detalhes ainda serão definidos.** Esta página fixa o que já está decidido e
> lista o que precisa ser escolhido, para que nada seja esquecido — não para fechar
> ferramenta agora.

## O que já está decidido

| Item | Decisão |
|---|---|
| **Estratégia de branch** | feature branching, uma branch por US — ver [Padrão de branches](../07-justificativas/01-branches.md) |
| **Padrão de commit** | convenção semântica, em inglês — ver [Padrão de commits](../07-justificativas/02-commits.md) |
| **Padrão de desenvolvimento** | **TDD**, backend e frontend — ver [TDD](../07-justificativas/03-tdd.md) |
| **Carga do DW** | **manual**, do nosso lado; produção recebe o dump validado — ver [D-17](02-decisoes-e-riscos.md#d-17--carga-manual-não-agendada) |
| **Produção** | **intranet do cliente, Windows Server** — ver [D-21](02-decisoes-e-riscos.md#d-21--produção-na-intranet-do-cliente-em-windows-server) e [Implantação no cliente](04-implantacao-no-cliente.md) |
| **Proxy reverso** | **NGINX** (o cliente usa IIS; trocamos) — ver [D-22](02-decisoes-e-riscos.md#d-22--nginx-como-proxy-reverso-no-lugar-do-iis) |
| **Entrega** | pacote de **arquivos buildados** + manual; o cliente instala |
| Homologação / demonstração | VPS Hostinger + Coolify? — **em aberto** ([R-17](02-decisoes-e-riscos.md#r-17--deploy-automático-exigido-pelo-desafio-x-produção-no-cliente-)) |
| Deploy automático | só no ambiente nosso; em produção, o CI gera o pacote |
| CI/CD | obrigatório |
| Monitoramento | obrigatório, ferramenta a definir |
| Documentação | obrigatória, formato a definir |

## Coolify — o que isso implica

> ⚠ **Vale só se a VPS continuar como homologação/demonstração.** Produção é a intranet
> do cliente, em Windows Server, sem Docker — ver [Implantação no cliente](04-implantacao-no-cliente.md).
> Os pontos 2 e 3 abaixo (health check, configuração fora do código) valem nos dois
> ambientes.

Coolify é uma plataforma self-hosted de deploy (um PaaS que roda na sua própria VPS).
Consequências práticas para o desenvolvimento:

**1 · Tudo precisa ser containerizável.** API e frontend rodam como contêineres.
Escrever `Dockerfile` para cada um é tarefa de desenvolvimento, não de infraestrutura, e
deve acontecer cedo — descobrir na véspera que a aplicação não containeriza é o tipo de
surpresa cara.

**2 · Health check é contrato, não enfeite.** Coolify usa health check para saber se um
deploy subiu. A API precisa expor:

| Endpoint | Responde |
|---|---|
| `/health` | o processo está de pé (*liveness*) |
| `/health/ready` | há dado utilizável no DW (*readiness*) |

O segundo é o que importa de verdade: uma API que sobe apontando para um banco vazio
está "no ar" e inútil.

**3 · Configuração por variável de ambiente.** Nada de connection string em
`appsettings.json` versionado. Coolify injeta as variáveis; o código lê do ambiente.

## Desenho da infraestrutura

> Este é o desenho do ambiente **nosso** (homologação/demonstração, se mantido). O de
> **produção** está em [Implantação no cliente](04-implantacao-no-cliente.md#desenho-no-servidor-do-cliente).

```
                    VPS Hostinger
   ┌──────────────────────────────────────────────┐
   │  Coolify                                     │
   │   ├── ratio-api        (ASP.NET Core)        │
   │   ├── ratio-web        (React, estático)     │
   │   └── ratio-db         (pgvector/pgvector:pg16)
   │                                              │
   │  proxy reverso + TLS  (gerenciado pelo Coolify)
   └──────────────────────────────────────────────┘
          ▲                          ▲
          │ deploy automático        │ pg_restore do schema dw
    GitHub Actions             carga manual, na máquina de quem opera
                               (coleta → NLP → testes → dump)
```

### A carga não roda na VPS

Não há contêiner de ETL nem job agendado. A carga é
[manual](../02-arquitetura/05-etl-e-nlp.md#carga-manual--o-processo): roda numa máquina
do time contra um Postgres local, passa pelos 24 testes de integridade, e só então o
schema `dw` é restaurado em produção numa transação única. Produção nunca fala com
DataJud, DOAJ ou qualquer fonte.

### O banco na mesma VPS

Simples e barato, e serve para o projeto. Duas consequências a encarar:

- **backup é responsabilidade nossa.** Um DW que se perde é uma recarga de dias contra
  fontes com limite de taxa. Backup automatizado do volume é item obrigatório.
- **o restore é o único momento pesado.** Como a carga roda fora, a VPS só sente o
  `pg_restore` — alguns minutos, numa transação. Faça fora do horário de uso.

## Pipeline de CI/CD — esqueleto

Um pipeline por repositório.

### `API5-Backend`

**Não existe workflow ainda** (o repo não tem `.github/`).

```
push / pull request
  ├── restore + build                     (falha rápida)
  ├── testes unitários                    (Domain, Application)      ← TDD
  ├── testes de integração                (Testcontainers: pgvector/pgvector:pg16)
  │     └── aplicam as migrations SQL e rodam os testes de integridade do DW
  ├── análise estática / lint
  └── publicação self-contained win-x64
        └── na main: gera o pacote de versão (API + web + nginx.conf + dump do dw)
```

### `API5-Frontend`

**Existe** — `.github/workflows/ci.yml`, Node 20, em `push`/`pull_request` para `main`:

```
push / pull request
  ├── npm ci
  ├── lint                  ✅
  ├── typecheck             ✅
  ├── test                  ❌ falta — vitest run  (TDD)
  └── build                 ✅
        └── na main: dist/ entra no pacote de versão   ❌ falta
```

> O gatilho hoje é só para `main`. Com o [padrão de branches](../07-justificativas/01-branches.md),
> PR de task entra em `usX` — acrescentar `us*` em `branches:` para o CI rodar nesses PRs.

### `API-5-docs` (esta wiki)

Lint de markdown e **verificação de links e âncoras quebrados**. Wiki com link morto
envelhece rápido, e as páginas se referenciam muito entre si.

Não precisa de deploy — o GitHub já renderiza. Se um dia a wiki virar site (MkDocs,
Docusaurus), aí sim entra no Coolify como estático.

## Testes de integridade do DW — requisito explícito

O desafio pede "testes automatizados validando integridade dos dados e consistência das
consultas". Isso é diferente de teste unitário, e roda no CI contra um Postgres real.

Cada um é uma consulta que retorna zero linhas quando está tudo certo.

### ✅ Escritos e passando — 24 consultas (15/09/2026)

Da lista original:

- [x] recarregar o mesmo lote **não** aumenta a contagem da tabela fato (idempotência);
- [x] nenhum registro do fato aponta para dimensão inexistente;
- [x] cada processo tem no máximo um resultado vigente;
- [x] nenhuma linha carregada está sem proveniência (fonte + data de extração);
- [x] os agregados não contêm tema sem lastro no fato;
- [x] todo código de movimentação com categoria de resultado está no mapa verificado;
- [ ] a soma dos agregados por ano bate com o resumo do mesmo tema;
- [ ] a soma dos agregados por tribunal bate com o resumo do mesmo tema.

*(Os dois últimos ainda não têm consulta escrita — o teste equivalente que existe
compara o `theme_summary` com a contagem recalculada direto do fato.)*

E mais 18 que a implementação mostrou serem necessários:

| Arquivo | Cobre |
|---|---|
| `scraping/sql/011_nlp_integrity_tests.sql` (8) | tema sem assunto de origem · assunto órfão · rótulo sem marcação de origem · ligação de doutrina sem score/método · ligação abaixo do limiar declarado · embedding com dimensão errada · **agregado divergindo do fato** |
| `scraping/sql/014_strength_link_tests.sql` (16) | nota fora de 0-100 · componente fora de 0-1 · **score que não reproduz a soma ponderada** · grau incompatível com a nota · pesos que não somam 1 · link sem tipo · tipo fora do vocabulário · número CNJ malformado · **código sem polaridade declarada** · **concordância sobre famílias misturadas** · **coluna com a palavra "favor" no schema** |

### Os três que mais valem

1. **O agregado mente?** — recalcula a contagem direto da tabela fato e compara
   com a view materializada. É o que trava o
   [D-11](02-decisoes-e-riscos.md#d-11--nada-de-dado-inventado) na prática: o
   modelo não conta, e há prova disso rodando.
2. **A fórmula mente?** — recalcula a nota de força do zero a partir dos
   componentes expostos e falha se divergir do valor publicado.
3. **A palavra "favorável" voltou?** — varre o `information_schema` e falha se
   alguma coluna do schema `dw` voltar a usá-la. Impede a reintrodução do
   [problema de polaridade](../03-dados/05-polaridade-do-resultado.md).

> **Onde rodam.** São o **passo 6 da [carga manual](../02-arquitetura/05-etl-e-nlp.md#carga-manual--o-processo)**
> — obrigatórios antes de qualquer dump subir para produção. Nenhuma linha retornada é
> a condição para seguir. No CI do backend, rodam nos testes de integração contra o
> Postgres do Testcontainers (pendente, junto com o workflow).

## Monitoramento — a definir

O que precisa ser respondido, independentemente da ferramenta escolhida:

| Pergunta | Sinal necessário |
|---|---|
| A API está de pé e respondendo rápido? | uptime + latência por rota |
| Quando foi a última carga? | data da última extração, exposta em `/health/ready` e no rodapé das telas |
| Alguma fonte mudou de comportamento? | taxa de erro por fonte |
| O banco está saudável? | espaço em disco, conexões; consultas lentas via `pg_stat_statements` |
| O que aconteceu quando deu erro? | log estruturado com correlação de requisição |

Com a carga manual não há "carga que falhou em silêncio" — ela falha na frente de quem
roda. O risco que sobra é o **dado envelhecer sem ninguém notar**: por isso a data da
extração aparece na tela, não só no log.

## Decisões pendentes

| Escolha | Opções a considerar | Quando decidir |
|---|---|---|
| Runner de CI | GitHub Actions (provável, pelos repos) | antes do primeiro merge relevante |
| Controle de migration aplicada | tabela de versão própria, ou DbUp lendo os SQL de `scraping/sql` | antes da primeira migration nova |
| Log estruturado | Serilog + destino a definir | junto com o esqueleto da API |
| Monitoramento | Uptime Kuma (leve, self-host), Grafana + Prometheus (completo), Sentry (erros) | depois do primeiro deploy |
| Backup | o dump de cada carga + o `raw` local, guardados fora da VPS | antes da primeira carga que doa perder |
| Gestão de segredos | configuração na máquina do cliente (fora do pacote) | imediato — nada de segredo no repositório nem no pacote |
| Ambientes | produção no cliente; homologação nossa (VPS + Coolify)? | R-17 |
| Documentação | esta wiki + Swagger + README por repo | contínuo |

## Regras que valem desde já

1. **Nenhum segredo no repositório.** Connection string, chave de API, credencial —
   tudo por variável de ambiente.
2. **Só vira pacote de versão o que passou no CI.** Nada de build feito à mão na máquina de alguém.
3. **Toda migration é versionada.** Em produção, o schema chega pelo `pg_restore` da
   carga validada — ninguém roda DDL à mão lá.
4. **A carga é idempotente.** Isso é o que torna reprocessamento seguro.
5. **Nada sobe para produção sem os 24 testes de integridade vazios.**
6. **Nada entra na `main` com teste falhando** — o CI bloqueia o merge. Ver [TDD](../07-justificativas/03-tdd.md).
7. **`Dockerfile` desde cedo.** Containerizar no fim do projeto é onde os prazos morrem.
