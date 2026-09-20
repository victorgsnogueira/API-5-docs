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
| **Acesso** | só restrição de rede, **sem login** — ver [D-23](02-decisoes-e-riscos.md#d-23--acesso-por-restrição-de-rede-sem-login) |
| **Simulação de produção** | rede **Tailscale** simulando a intranet; sem VPS — ver [D-24](02-decisoes-e-riscos.md#d-24--simulação-da-intranet-numa-rede-tailscale) |
| Deploy | o merge na `main` publica uma release (zip + `.sha256`) — ver [Versionamento e releases](../07-justificativas/04-versionamento-e-releases.md); quem instala é o cliente |
| CI/CD | obrigatório |
| Monitoramento | obrigatório, ferramenta a definir |
| Documentação | obrigatória, formato a definir |

## Ambiente

Produção é a intranet do cliente — ver [Implantação no cliente](04-implantacao-no-cliente.md).
O projeto está sendo rodado numa rede **Tailscale** para simular esse ambiente de
intranet ([D-24](02-decisoes-e-riscos.md#d-24--simulação-da-intranet-numa-rede-tailscale)).

**Health check é contrato, não enfeite.** É por ele que a instalação é verificada. A
API precisa expor:

| Endpoint | Responde | Quando não está bem |
|---|---|---|
| `/health` | o processo está de pé (*liveness*) | não responde |
| `/health/ready` | há dado utilizável no DW, e de quando é a última carga (*readiness*) | `503` com o campo **`reason`** |

O segundo é o que importa de verdade: uma API que sobe apontando para um banco vazio
está "no ar" e inútil.

**Os dois `503` não são a mesma coisa**, e o `reason` é o que os separa:

| `reason` | Significa | Ação |
|---|---|---|
| `sem carga publicada` | o banco respondeu; falta restaurar o dump do `dw` | restaurar a carga |
| `banco inacessível` | o banco não respondeu — parado, credencial, permissão | consertar o banco; a causa está no log em arquivo |

Por isso o monitoramento olha o `reason`, não só o status: um pede uma carga, o outro é
incidente de infraestrutura. Contrato completo em
[Backend .NET](../02-arquitetura/02-backend-dotnet.md#health-check--os-três-estados).

**Configuração fora do código.** Nada de connection string em `appsettings.json`
versionado nem dentro do pacote. A configuração é da máquina onde roda.

### A carga não roda no ambiente de produção

Não há contêiner de ETL nem job agendado. A carga é
[manual](../02-arquitetura/05-etl-e-nlp.md#carga-manual--o-processo): passa pelos 24
testes de integridade, e só então o schema `dw` é restaurado. Produção não fala com
DataJud, DOAJ ou qualquer fonte.

**Backup é responsabilidade nossa** até a entrega: um DW que se perde é uma recarga de
dias contra fontes com limite de taxa. Guarde o dump de cada carga e o `raw`.

## Pipeline de CI/CD — esqueleto

Um pipeline por repositório.

### `API5-Backend`

**Existe** — `.github/workflows/ci.yml` (workflow `Backend CI`, job `Backend checks`),
em `push`/`pull_request` para `main` e `us*`, e também sob demanda (`workflow_dispatch`):

```
push / pull request
  ├── restore com dependências travadas   (--locked-mode)
  ├── verificação de formatação e analyzers  (dotnet format --verify-no-changes)
  ├── build Release
  ├── testes com PostgreSQL descartável (Testcontainers)   ← TDD
  │     └── schema e dados mínimos de teste para validar as consultas da API
  ├── publicação self-contained win-x64
  ├── upload de resultados de teste e cobertura (14 dias)
  └── na main: upload do artefato da API para o pacote de versão (30 dias)
```

O job `Backend checks` é o check **obrigatório** para merge na `main` e nas `usX`,
configurado nos rulesets (ver
[Proteção da `main` e das branches de US](../07-justificativas/01-branches.md#proteção-da-main-e-das-branches-de-us)).

Raspagem, ETL, NLP, migrations da carga e os 24 testes de integridade do DW ficam
fora do repositório e do CI do backend. Os testes da API preparam seu próprio banco
descartável; não dependem da máquina de carga, da Tailscale, da homologação ou de um
dump completo. A API em execução apenas consulta o `dw` publicado.

O pacote entregue ao cliente reúne os artefatos da aplicação, a configuração do NGINX
e um dump previamente validado pelo processo separado de carga. O CI da aplicação
não coleta nem processa dados para gerar esse dump.

### `API5-Frontend`

**Existe** — `.github/workflows/ci.yml` (workflow `Frontend CI`, job `Frontend checks`),
em `push`/`pull_request` para `main` e `us*`, e também sob demanda (`workflow_dispatch`).
A versão do Node vem de `ratio/.node-version`:

```
push / pull request
  ├── npm ci                              (dependências travadas)
  ├── format:check
  ├── lint
  ├── typecheck
  ├── test:ci                             ← TDD (com cobertura)
  ├── build
  ├── upload de resultados e cobertura (14 dias)
  └── na main: upload do dist/ para o pacote de versão (30 dias)
```

O job `Frontend checks` é o check **obrigatório** para merge na `main` e nas `usX`,
configurado nos rulesets (ver
[Proteção da `main` e das branches de US](../07-justificativas/01-branches.md#proteção-da-main-e-das-branches-de-us)).

### Release — backend e frontend

Além do CI, cada repositório tem dois workflows que cuidam da versão, idênticos nos dois.
Detalhes, regra de versão e pontos de atenção em
[Versionamento e releases](../07-justificativas/04-versionamento-e-releases.md).

| Workflow | Job / check | Quando roda | O que faz |
|---|---|---|---|
| `Release label` (`release-label.yml`) | `Release label` | PR para a `main` (aberto, reaberto, novo push, label posto ou tirado) | exige **um** label `release:sprint\|us\|fix\|none` e comenta no PR a versão prevista |
| `Backend Release` / `Frontend Release` (`release.yml`) | `Publish release` | fim do CI, com sucesso, em push na `main` | acha o PR mergeado, calcula a versão pelo label e cria a **GitHub Release** com o zip e o `.sha256` |

O check `Release label` é **obrigatório** nos rulesets. A release publica
`ratio-api-vX.Y.Z-win-x64.zip` (backend, self-contained) e `ratio-web-vX.Y.Z.zip`
(frontend, conteúdo do `dist/`). Isso **não é o pacote de versão completo** do cliente:
faltam `nginx.conf`, dump do `dw`, scripts e manual.

### `API-5-docs` (esta wiki)

Lint de markdown e **verificação de links e âncoras quebrados**. Wiki com link morto
envelhece rápido, e as páginas se referenciam muito entre si.

Não precisa de deploy — o GitHub já renderiza. Se um dia a wiki virar site (MkDocs,
Docusaurus), aí sim vira um site estático a hospedar.

## Testes de integridade do DW — requisito explícito

O desafio pede "testes automatizados validando integridade dos dados e consistência das
consultas". Os 24 testes SQL validam os dados no processo separado de carga manual.
Os testes de integração do backend validam as consultas da API contra PostgreSQL
descartável no CI. São responsabilidades distintas.

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
> a condição para seguir. O script de publicação também os executa no destino.
> Eles não fazem parte do CI do backend ou do frontend.

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
| ~~Runner de CI~~ | decidido: **GitHub Actions**, runner `ubuntu-24.04` | — |
| Controle de migration aplicada na carga separada | mecanismo a definir junto do versionamento do pipeline, fora do backend | antes da primeira migration nova |
| Log estruturado | Serilog + destino a definir | junto com o esqueleto da API |
| Monitoramento | Uptime Kuma (leve, self-host), Grafana + Prometheus (completo), Sentry (erros) | depois do primeiro deploy |
| Backup | o dump de cada carga + o `raw` local, guardados fora da máquina que os gerou | antes da primeira carga que doa perder |
| Gestão de segredos | configuração na máquina do cliente (fora do pacote) | imediato — nada de segredo no repositório nem no pacote |
| Pacote de versão completo | as releases do backend e do frontend são zips separados; falta juntar com `nginx.conf`, dump do `dw`, scripts e manual | antes da primeira entrega ao cliente |
| Documentação | esta wiki + Swagger + README por repo | contínuo |

## Regras que valem desde já

1. **Nenhum segredo no repositório.** Connection string, chave de API, credencial —
   tudo por variável de ambiente.
2. **Só vira pacote de versão o que passou no CI.** Nada de build feito à mão na máquina de alguém. A release só é publicada depois do CI verde na `main`.
3. **Toda migration da carga deve ser versionada separadamente do backend** (pendência R-15). Em produção, o schema chega pelo `pg_restore` da
   carga validada — ninguém roda DDL à mão lá.
4. **A carga é idempotente.** Isso é o que torna reprocessamento seguro.
5. **Nada sobe para produção sem os 24 testes de integridade vazios.**
6. **Nada entra na `main` com teste falhando** — o CI bloqueia o merge. Ver [TDD](../07-justificativas/03-tdd.md).
7. **`Dockerfile` desde cedo.** Containerizar no fim do projeto é onde os prazos morrem.
