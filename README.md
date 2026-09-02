# Ratio — Wiki do projeto

> *Ratio* (latim: razão, fundamento). De **ratio decidendi** — o fundamento
> determinante de uma decisão judicial, aquilo que de fato se repete e vira padrão.

Data Warehouse jurídico com camada de busca: um lugar único onde advogado e juiz
consultam **como um tema costuma ser decidido**, em vez de garimpar processo a
processo em portais de tribunal.

**Escopo:** TJSP, TJRJ e TJMG. **Fontes:** múltiplas (DataJud, PANGEA, repositórios de
tribunal, doutrina). **Stack:** .NET 8 + React + Postgres, em VPS Hostinger com Coolify.

---

## Leia isto antes de começar a codar

| Aviso | Onde |
|---|---|
| ⚠ O `prototipo/` **não é fonte de verdade** — nem o código, nem o contrato, nem a modelagem | [Protótipo](05-prototipo/01-prototipo-referencia.md) |
| ⚠ A **modelagem do DW ainda não existe** — a do protótipo foi feita sem auditoria | [Modelo dimensional](03-dados/02-modelo-dimensional.md) |
| ⚠ Boa parte dos mockups **não tem fonte de dados** confirmada | [Limitações da fonte](03-dados/04-limitacoes-da-fonte.md) |
| ⚠ O backend é escrito **em inglês**; os dados ficam em português | [Idioma](02-arquitetura/02-backend-dotnet.md#idioma) |

---

## Por onde começar

| Se você é… | Leia nesta ordem |
|---|---|
| Novo no time | [Problema e solução](00-visao-geral/01-problema-e-solucao.md) → [Glossário](00-visao-geral/02-glossario.md) → [Repositórios](00-visao-geral/03-repositorios.md) |
| Dev backend | [Visão macro](02-arquitetura/01-visao-macro.md) → [Backend .NET](02-arquitetura/02-backend-dotnet.md) → [Modelo dimensional](03-dados/02-modelo-dimensional.md) |
| Dev frontend | [Telas](01-produto/02-telas.md) → [Design system](04-design/01-design-system.md) → [Frontend React](02-arquitetura/03-frontend-react.md) |
| Dev de dados / ETL | [Fontes](03-dados/01-fontes.md) → [ETL e NLP](02-arquitetura/05-etl-e-nlp.md) → [Limitações](03-dados/04-limitacoes-da-fonte.md) |
| DevOps | [DevOps e infraestrutura](06-operacao/03-devops-e-infra.md) → [Ambiente local](06-operacao/01-ambiente-local.md) |
| SM / documentação | [Repositórios](00-visao-geral/03-repositorios.md) → [Decisões e riscos](06-operacao/02-decisoes-e-riscos.md) |

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
- [Backend .NET](02-arquitetura/02-backend-dotnet.md) — camadas, idioma, contrato da API
- [Frontend React](02-arquitetura/03-frontend-react.md) — estrutura, rotas, consumo da API
- [Data Warehouse](02-arquitetura/04-data-warehouse.md) — por que Postgres, e o que isso custa
- [ETL e NLP](02-arquitetura/05-etl-e-nlp.md) — conectores, normalização e onde a LLM entra

### 03 · Dados
- [Fontes de dados](03-dados/01-fontes.md) — DataJud, PANGEA, JusBrasil, tribunais, doutrina
- [Modelo dimensional](03-dados/02-modelo-dimensional.md) — método e proposta · **a auditar**
- [Agregados OLAP](03-dados/03-agregados-olap.md) — as consultas que alimentam cada tela
- [Limitações da fonte](03-dados/04-limitacoes-da-fonte.md) — **o que não dá para prometer**

### 04 · Design
- [Design system](04-design/01-design-system.md) — tipografia, cor, forma, componentes

### 05 · Protótipo
- [Protótipo — o que é e o que não é](05-prototipo/01-prototipo-referencia.md) — catálogo de armadilhas, não referência

### 06 · Operação
- [Ambiente local](06-operacao/01-ambiente-local.md) — subir tudo na sua máquina
- [Decisões e riscos](06-operacao/02-decisoes-e-riscos.md) — registro de decisões e o que está aberto
- [DevOps e infraestrutura](06-operacao/03-devops-e-infra.md) — CI/CD, Coolify, monitoramento

---

## Estado atual (2026-09-02)

| Frente | Estado |
|---|---|
| Telas (design) | ✅ mockups fechados em [`Docs/Telas/`](Telas/) |
| Escopo, fontes e convenções | ✅ definidos — ver [Decisões](06-operacao/02-decisoes-e-riscos.md) |
| Modelagem do DW | 🔴 **não existe** — a do protótipo não serve |
| Backend .NET (`API5-Backend`) | 🔴 solução criada, camadas em branco |
| Frontend React (`API5-Frontend`) | 🔴 repositório vazio |
| ETL | 🔴 não iniciado |
| Fontes além do DataJud | 🔴 não verificadas |
| NLP / normalização em tema | 🔴 não iniciado |
| Chatbot | 🔴 roadmap |
| CI/CD, deploy, monitoramento | 🔴 não configurado |

## Próximos desbloqueios, em ordem

1. **Fechar o grão e a modelagem do DW** — trava tudo o mais ([R-03](06-operacao/02-decisoes-e-riscos.md#r-03--modelagem-do-dw-ainda-não-existe-)).
2. **Spike por fonte** — PANGEA, repositórios do TJSP/TJRJ/TJMG, doutrina.
3. **Um fluxo vertical fino** — uma fonte, um recorte, uma tela, ponta a ponta.
4. **`Dockerfile` + pipeline de build**, na mesma semana em que o código começa.
