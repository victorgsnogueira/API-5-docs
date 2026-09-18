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
| **Hospedagem** | VPS na **Hostinger** |
| **Orquestração / deploy** | **Coolify** |
| Deploy | automático |
| CI/CD | obrigatório |
| Monitoramento | obrigatório, ferramenta a definir |
| Documentação | obrigatória, formato a definir |

## Coolify — o que isso implica

Coolify é uma plataforma self-hosted de deploy (um PaaS que roda na sua própria VPS).
Duas consequências práticas para o desenvolvimento:

**1 · Tudo precisa ser containerizável.** API, ETL e frontend rodam como contêineres.
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

```
                    VPS Hostinger
   ┌──────────────────────────────────────────────┐
   │  Coolify                                     │
   │   ├── ratio-api        (ASP.NET Core)        │
   │   ├── ratio-web        (React, estático)     │
   │   ├── ratio-db         (Postgres)            │
   │   └── ratio-etl        (job agendado)        │
   │                                              │
   │  proxy reverso + TLS  (gerenciado pelo Coolify)
   └──────────────────────────────────────────────┘
              ▲                        │
              │ deploy automático      │ ETL sai para as fontes
        GitHub Actions            DataJud · PANGEA · tribunais
```

### O ETL não é um serviço web

`Ratio.Etl` é console app. No Coolify, isso vira um **job agendado** (cron), não um
serviço com porta. Rodar a carga dentro do processo da API é erro: acopla a carga ao
ciclo de vida do servidor e impede rodar uma carga manual sem reiniciar tudo.

### O banco na mesma VPS

Simples e barato, e serve para o projeto. Duas consequências a encarar:

- **backup é responsabilidade nossa.** Um DW que se perde é uma recarga de dias contra
  fontes com limite de taxa. Backup automatizado do volume é item obrigatório.
- **recursos são compartilhados.** Uma carga pesada de ETL compete com a API pelo mesmo
  CPU. Agendar a carga em horário de baixo uso.

## Pipeline de CI/CD — esqueleto

Um pipeline por repositório.

### `API5-Backend`

```
push / pull request
  ├── restore + build                     (falha rápida)
  ├── testes unitários                    (Domain, Application)
  ├── testes de integração                (Postgres de serviço)
  │     └── inclui os testes de integridade do DW (abaixo)
  ├── análise estática / lint
  └── build da imagem Docker
        └── na main: publica e dispara deploy no Coolify
```

### `API5-Frontend`

```
push / pull request
  ├── install + typecheck + lint
  ├── testes
  └── build
        └── na main: publica e dispara deploy no Coolify
```

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

> ⚠ **Ainda não rodam no CI.** Hoje são arquivos `.sql` executados à mão contra
> o banco local. Entrar no pipeline é tarefa pendente — junto com o port do ETL
> para o `Ratio.Etl` em .NET.

## Monitoramento — a definir

O que precisa ser respondido, independentemente da ferramenta escolhida:

| Pergunta | Sinal necessário |
|---|---|
| A API está de pé e respondendo rápido? | uptime + latência por rota |
| A carga de ontem rodou? | **alarme quando o ETL falha ou não roda** |
| O dado está velho? | idade da última extração, exposta em `/health/ready` |
| Alguma fonte mudou de comportamento? | taxa de erro por fonte |
| O banco está saudável? | espaço em disco, conexões, duração do `REFRESH` |
| O que aconteceu quando deu erro? | log estruturado com correlação de requisição |

O alarme de ETL é o mais importante: uma carga que falha silenciosamente por uma semana
deixa o produto exibindo dado velho **com aparência de dado atual**.

## Decisões pendentes

| Escolha | Opções a considerar | Quando decidir |
|---|---|---|
| Runner de CI | GitHub Actions (provável, pelos repos) | antes do primeiro merge relevante |
| Runner de migration | DbUp, FluentMigrator | antes da primeira migration |
| Log estruturado | Serilog + destino a definir | junto com o esqueleto da API |
| Monitoramento | Uptime Kuma (leve, self-host), Grafana + Prometheus (completo), Sentry (erros) | depois do primeiro deploy |
| Backup | dump agendado + destino externo à VPS | antes da primeira carga que doa perder |
| Gestão de segredos | variáveis do Coolify | imediato — nada de segredo no repositório |
| Ambientes | só produção, ou produção + staging? | antes de configurar o Coolify |
| Documentação | esta wiki + Swagger + README por repo | contínuo |

## Regras que valem desde já

1. **Nenhum segredo no repositório.** Connection string, chave de API, credencial —
   tudo por variável de ambiente.
2. **Deploy só do que passou no CI.** Sem `git push` para a VPS.
3. **Toda migration é versionada e roda no pipeline**, nunca à mão em produção.
4. **A carga é idempotente.** Isso é o que torna deploy e reprocessamento seguros.
5. **`Dockerfile` desde cedo.** Containerizar no fim do projeto é onde os prazos morrem.
