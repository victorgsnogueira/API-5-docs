# Ratio — Wiki do projeto

> *Ratio* (latim: razão, fundamento). De **ratio decidendi** — o fundamento
> determinante de uma decisão judicial, aquilo que de fato se repete e vira padrão.

Data Warehouse jurídico com camada de busca: um lugar único onde advogado e juiz
consultam **como um tema costuma ser decidido**, em vez de garimpar processo a
processo em portais de tribunal.

**Escopo:** TJSP, TJRJ e TJMG. **Fontes:** múltiplas (DataJud, PANGEA, repositórios de
tribunal, doutrina). **Stack:** ASP.NET Core + React (TanStack Router, shadcn/ui,
Tailwind) + Postgres 16. **Produção na intranet do cliente**, em Windows
Server atrás de NGINX, só para funcionários dele. **TDD** no backend e no frontend.
**Carga manual** do DW.

Esta wiki é o repositório `API-5-docs`; o código vive em
[outros repositórios](00-visao-geral/03-repositorios.md), clonados lado a lado.
Os caminhos citados aqui (`API5-Backend/…`, `prototipo/…`) pressupõem esse arranjo.

---

## Leia isto antes de começar a codar

| Aviso | Onde |
|---|---|
| ⚠ O `prototipo/` **não é fonte de verdade** — nem o código, nem o contrato, nem a modelagem | [Protótipo](05-prototipo/01-prototipo-referencia.md) |
| ✅ A **modelagem do DW existe e está carregada** — 463.016 linhas de fato | [Modelo dimensional](03-dados/02-modelo-dimensional.md) |
| 🔴 **"Favorável" sem dizer a quem inverte a leitura** — leia antes de exibir percentual | [Polaridade do resultado](03-dados/05-polaridade-do-resultado.md) |
| ⚠ O que depende de **inteiro teor** segue sem fonte — os 4 tribunais estão bloqueados | [Limitações da fonte](03-dados/04-limitacoes-da-fonte.md) |
| ⚠ Código **em inglês**; tudo o que a API devolve (dados, rótulos, erros) **em português** | [Idioma](02-arquitetura/02-backend-dotnet.md#idioma) |
| 🧪 **TDD**: nenhum código de produção sem um teste que falhou antes | [TDD](07-justificativas/03-tdd.md) |
| 🏢 Produção é a **intranet do cliente** (Windows Server, NGINX); ele recebe **só arquivos buildados** | [Implantação no cliente](06-operacao/04-implantacao-no-cliente.md) |
| ⚠ A carga do DW é **manual** e a pasta `scraping/` **não está versionada** | [D-17](06-operacao/02-decisoes-e-riscos.md#d-17--carga-manual-não-agendada) · [R-15](06-operacao/02-decisoes-e-riscos.md#r-15--o-pipeline-de-carga-fica-fora-de-repositório--risco-aceito) |

---

## Por onde começar

| Se você é… | Leia nesta ordem |
|---|---|
| Novo no time | [Problema e solução](00-visao-geral/01-problema-e-solucao.md) → [Glossário](00-visao-geral/02-glossario.md) → [Repositórios](00-visao-geral/03-repositorios.md) |
| Dev backend | [Visão macro](02-arquitetura/01-visao-macro.md) → [Backend .NET](02-arquitetura/02-backend-dotnet.md) → [Modelo dimensional](03-dados/02-modelo-dimensional.md) |
| Dev frontend | [Telas](01-produto/02-telas.md) → [Design system](04-design/01-design-system.md) → [Frontend React](02-arquitetura/03-frontend-react.md) |
| Dev de dados / ETL | [Fontes](03-dados/01-fontes.md) → [ETL e NLP](02-arquitetura/05-etl-e-nlp.md) → [Limitações](03-dados/04-limitacoes-da-fonte.md) → [Polaridade](03-dados/05-polaridade-do-resultado.md) |
| DevOps | [DevOps e infraestrutura](06-operacao/03-devops-e-infra.md) → [Implantação no cliente](06-operacao/04-implantacao-no-cliente.md) → [Ambiente local](06-operacao/01-ambiente-local.md) |
| SM / documentação | [Repositórios](00-visao-geral/03-repositorios.md) → [Decisões e riscos](06-operacao/02-decisoes-e-riscos.md) |
| Qualquer um, antes do primeiro commit | [Padrão de branches](07-justificativas/01-branches.md) → [Padrão de commits](07-justificativas/02-commits.md) → [TDD](07-justificativas/03-tdd.md) |

---

## Índice completo

### 00 · Visão geral
- [Problema e solução](00-visao-geral/01-problema-e-solucao.md) — a dor do cliente e o que o Ratio entrega
- [Glossário jurídico e técnico](00-visao-geral/02-glossario.md) — processo, jurisprudência, doutrina, tema, DW, ETL
- [Repositórios e responsabilidades](00-visao-geral/03-repositorios.md) — os repos e quem mexe em quê

### 01 · Produto
- [Personas e jornada](01-produto/01-personas-e-jornada.md) — advogado e juiz, o que cada um pergunta
- [Telas](01-produto/02-telas.md) — Busca, Resultados, Detalhamento (Resumo / Base Analítica)
- [O que é um "tema"](01-produto/03-tema-modelo-conceitual.md) — a entidade central do produto
- [Força do entendimento](01-produto/04-forca-do-entendimento.md) — a nota 0–100 e por que ela é auditável
- [Chatbot](01-produto/05-chatbot.md) — LLM consultando o DW · **roadmap**

### 02 · Arquitetura
- [Visão macro](02-arquitetura/01-visao-macro.md) — as camadas, da fonte à tela
- [Backend .NET](02-arquitetura/02-backend-dotnet.md) — camadas, idioma, pacotes, contrato da API
- [Frontend React](02-arquitetura/03-frontend-react.md) — stack real do repo, estrutura, rotas, regras
- [Data Warehouse](02-arquitetura/04-data-warehouse.md) — por que Postgres, e **tudo o que está instalado nele**
- [ETL e NLP](02-arquitetura/05-etl-e-nlp.md) — normalização campo a campo, NLP, **processo da carga manual**

### 03 · Dados
- [Fontes de dados](03-dados/01-fontes.md) — DataJud, PANGEA, JusBrasil, tribunais, doutrina
- [Modelo dimensional](03-dados/02-modelo-dimensional.md) — esquema implementado · checklist respondido
- [Agregados OLAP](03-dados/03-agregados-olap.md) — as consultas que alimentam cada tela
- [Limitações da fonte](03-dados/04-limitacoes-da-fonte.md) — **o que não dá para prometer**
- [Polaridade do resultado](03-dados/05-polaridade-do-resultado.md) — **leitura obrigatória antes de exibir qualquer percentual**

### 04 · Design
- [Design system](04-design/01-design-system.md) — tipografia, cor, forma, componentes

### 05 · Protótipo
- [Protótipo — o que é e o que não é](05-prototipo/01-prototipo-referencia.md) — catálogo de armadilhas, não referência

### 06 · Operação
- [Ambiente local](06-operacao/01-ambiente-local.md) — subir tudo na sua máquina
- [Decisões e riscos](06-operacao/02-decisoes-e-riscos.md) — registro de decisões e o que está aberto
- [DevOps e infraestrutura](06-operacao/03-devops-e-infra.md) — CI/CD, pacote de versão, monitoramento
- [Implantação no cliente](06-operacao/04-implantacao-no-cliente.md) — intranet, Windows Server, NGINX, e os manuais que faltam escrever

### 07 · Justificativas
Os padrões e as ferramentas do projeto, e por que foram escolhidos.
- [Padrão de branches](07-justificativas/01-branches.md) — `main`, uma branch por US, uma por task
- [Padrão de commits](07-justificativas/02-commits.md) — convenção semântica, em inglês
- [TDD](07-justificativas/03-tdd.md) — o padrão de desenvolvimento, backend e frontend

---

## Estado atual (2026-09-19)

| Frente | Estado |
|---|---|
| Telas (design) | ✅ mockups fechados em [`Telas/`](Telas/) |
| Escopo, fontes e convenções | ✅ definidos — ver [Decisões](06-operacao/02-decisoes-e-riscos.md) |
| Modelagem do DW | ✅ implementada e carregada — 463.016 linhas de fato |
| Carga (coleta + normalização) | ✅ funcional, **manual** — a pasta `scraping/` fica **fora de repositório**, por decisão ([R-15](06-operacao/02-decisoes-e-riscos.md#r-15--o-pipeline-de-carga-fica-fora-de-repositório--risco-aceito)) |
| NLP / normalização em tema | ✅ 447 assuntos → 408 temas; doutrina ligada a tema |
| Fontes além do DataJud | 🟠 doutrina ✅; jurisprudência dos tribunais bloqueada |
| Backend .NET (`API5-Backend`) | 🟠 setup na branch `initial-setup` (PR aberto): .NET 10, camadas, health check, 17 testes — sem CI e sem rota de domínio |
| Frontend React (`API5-Frontend`) | 🟠 scaffold com design system e CI; nenhuma tela |
| Testes | 🟠 24 de integridade do DW + 17 no backend; **frontend sem Vitest** (nem script `test`) |
| Chatbot | 🔴 roadmap |
| Implantação no cliente | 🔴 requisitos registrados; manuais, spec das máquinas e pacote **a escrever** |
| Deploy, monitoramento | 🔴 não configurado |

## Próximos desbloqueios, em ordem

1. **Configurar os testes do frontend** — Vitest + Testing Library + MSW; hoje o `apps/web`
   não tem nem script `test`, e o backend já está com xUnit + Moq + Testcontainers
   ([TDD](07-justificativas/03-tdd.md)).
2. **Um fluxo vertical fino** — `GET /api/topics` + tela de resultados, ponta a ponta, por TDD.
3. **CI do backend** — build + test + pacote de versão; os testes de integração pedem
   **Docker no runner**. Não há `.github/` no repositório.
4. **Manual de implantação e spec das máquinas** — o cliente instala sozinho, a partir só
   dos arquivos buildados ([Implantação no cliente](06-operacao/04-implantacao-no-cliente.md#documentos-que-precisam-ser-escritos)).

> Saíram desta lista: versionar o `scraping/` (decidido que fica fora) e subir para .NET 10
> (feito). `Dockerfile` também: produção é serviço Windows, não contêiner
> ([D-21](06-operacao/02-decisoes-e-riscos.md#d-21--produção-na-intranet-do-cliente-em-windows-server)) —
> Docker só aparece em teste e na carga.
