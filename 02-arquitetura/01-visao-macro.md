# Visão macro

## As camadas

```
┌─ 1 · FONTES (multifonte) ───────────────────────────────────────────┐
│  DataJud/CNJ      metadado processual em massa          ✅ carregado │
│  Doutrina         DOAJ · SciELO · OAI-PMH, artigo com link  ✅       │
│  Repositórios do TJSP · TJRJ · TJMG   inteiro teor   🔴 bloqueados  │
│  PANGEA / PDPJ    precedentes qualificados           (a investigar) │
│                        escopo: SP · RJ · MG                         │
└──────────────────────────┬──────────────────────────────────────────┘
                           │  batch MANUAL — rodado à mão, do nosso lado
┌─ 2 · CARGA ───────────────▼─────────────────────────────────────────┐
│  Extract    um coletor por fonte ──► raw (JSONB)                    │
│  Transform  achatar ──► staging · traduzir códigos · polaridade     │
│  Normalize  NLP: embeddings · temas + curadoria · doutrina↔tema     │
│  Load       upsert idempotente, com proveniência em toda linha      │
│  Validate   24 testes de integridade ── só sobe se todos passarem   │
│                          pasta scraping/ — Python + SQL  (D-17)     │
└──────────────────────────┬──────────────────────────────────────────┘
                           │  pg_dump / pg_restore do schema dw
┌─ 3 · DATA WAREHOUSE ──────▼─────────────────────────────────────────┐
│  Postgres 16 · ICU pt-BR · modelagem dimensional                    │
│  grão = movimentação processual (D-13) · 463.016 linhas de fato     │
└──────────────────────────┬──────────────────────────────────────────┘
                           │  atualizado ao fim de cada carga
┌─ 4 · AGREGADOS OLAP ──────▼─────────────────────────────────────────┐
│  resultado vigente por processo · resumo por tema · por ano ·       │
│  por tribunal · por órgão julgador                                  │
└──────────────────────────┬──────────────────────────────────────────┘
                           │
┌─ 5 · APLICAÇÃO ───────────▼─────────────────────────────────────────┐
│  API       ASP.NET Core · Ratio.Api   só leitura · retorno em PT    │
│  Web       React · TanStack Router · shadcn/ui · Tailwind            │
│  Chatbot   LLM com tool use sobre os mesmos agregados  (roadmap)    │
└──────────────────────────┬──────────────────────────────────────────┘
                           │
┌─ 6 · DEVOPS ──────────────▼─────────────────────────────────────────┐
│  CI/CD · deploy automático · monitoramento · documentação           │
│  produção: intranet do cliente · Windows Server · NGINX (D-21/22)   │
└─────────────────────────────────────────────────────────────────────┘
```

## Decisões estruturantes

### Multifonte desde o desenho

Nenhuma fonte isolada entrega o que as telas pedem. O ETL nasce com **um conector por
fonte atrás de uma mesma porta** (`ICaseSource`), e acrescentar fonte é registrar uma
classe — não reescrever o pipeline. Ver [Fontes](../03-dados/01-fontes.md).

**Corolário: proveniência é obrigatória.** Toda linha carregada sabe de que fonte veio e
quando. Sem isso, é impossível auditar divergência entre fontes ou reprocessar uma só.

### Escopo territorial: SP, RJ e MG

Só TJSP, TJRJ e TJMG. Isso reduz drasticamente a superfície: três índices no DataJud em
vez de 91, três sistemas de deep link em vez de dezenas, carga completa em janela
razoável.

A interface precisa **declarar esse escopo** — um usuário que assume cobertura nacional
tira conclusão errada.

### Batch manual, não agendado

As fontes não são tempo real (o DataJud tem defasagem de horas a dias) e o produto
mostra tendência, não notícia. A carga é **manual** ([D-17](../06-operacao/02-decisoes-e-riscos.md#d-17--carga-manual-não-agendada)):
roda à mão, passa por curadoria humana e por testes de integridade, e só então sobe
para produção. Cada rodada reprocessa janelas que se sobrepõem — daí a **idempotência
ser requisito**.

### Agregados pré-calculados

A tela de tema abre com várias agregações sobre a tabela fato. Rodar isso a cada request
deixaria a página lenta, e o dado só muda quando o ETL roda.

**Consequência:** o dado exibido é tão fresco quanto a última carga manual. A interface
declara a data de extração — os mockups já fazem isso.

### Busca full-text no próprio Postgres

`to_tsvector('portuguese', …)` com `unaccent` e `pg_trgm` para erro de digitação. Evita
subir um Elasticsearch só para o campo de busca, nesta escala.

### API somente leitura no domínio

O único caminho de gravação de dado de domínio é a carga manual. Simplifica
autenticação, CORS e permissão de banco — o usuário de banco da API só tem `SELECT`. A
exceção aparente é a rota do chatbot, que não grava domínio.

### O chatbot consome os mesmos agregados

Não é um caminho alternativo até o banco: é uma interface de linguagem natural sobre as
consultas que as telas já usam. É isso que garante que ele responda os mesmos números
que a tela mostra. Ver [Chatbot](../01-produto/05-chatbot.md).

### Código em inglês, retorno em português

Todo o código em inglês — identificadores, tabelas, rotas, chaves do JSON. Tudo o que a
API **devolve para ser lido** — dados, rótulos, erros — em português, porque o domínio é
o direito brasileiro e o usuário é advogado e juiz. Detalhe em
[Backend .NET](02-backend-dotnet.md#idioma).

### TDD

Backend e frontend são escritos por TDD. Ver [TDD](../07-justificativas/03-tdd.md).

## Contrato entre as camadas

| Fronteira | Contrato |
|---|---|
| Fonte → carga | payload cru em `raw`, achatado em `staging` antes de tocar o modelo |
| Carga → DW | SQL com upsert; nenhuma regra de negócio no banco além dos agregados; sobe por `pg_restore` só depois dos testes |
| DW → API | os agregados são a interface; a API não agrega sobre o fato em tempo de request |
| API → Web | chaves em inglês, valores e mensagens em português, métricas pré-calculadas, proveniência em toda resposta |
| API → Chatbot | as mesmas consultas, expostas como ferramentas parametrizadas |

## Onde cada peça mora

| Camada | Repositório | Projeto / pasta |
|---|---|---|
| Carga (coleta, NLP, testes de integridade) | ⚠ sem repo — pasta `scraping/` ([R-15](../06-operacao/02-decisoes-e-riscos.md#r-15--o-pipeline-de-carga-não-está-versionado-)) | `scripts/` |
| Migrations do DW | idem | `sql/` |
| API | `API5-Backend` | `Ratio.Api` |
| Web | `API5-Frontend` | `ratio/apps/web` + `ratio/packages/ui` |
| Infra / deploy | a definir | ver [DevOps](../06-operacao/03-devops-e-infra.md) |

## Infraestrutura

| Item | Escolha |
|---|---|
| Banco | Postgres 16 com `pg_trgm` e `unaccent`; `pgvector` só no ambiente de carga ([D-25](../06-operacao/02-decisoes-e-riscos.md#d-25--produção-sem-pgvector-embeddings-ficam-na-carga)) — ver [Data Warehouse](04-data-warehouse.md#o-que-está-instalado-no-banco) |
| Produção | **intranet do cliente**, **Windows Server**, uso só por funcionários — ver [Implantação no cliente](../06-operacao/04-implantacao-no-cliente.md) |
| Proxy reverso | **NGINX** (no lugar do IIS do cliente) |
| Entrega | arquivos buildados: API self-contained `win-x64`, `dist/` do frontend, `nginx.conf`, dump do `dw` |
| Acesso | só restrição de rede, sem login (D-23) |
| Simulação de produção | rede **Tailscale** simulando a intranet (D-24) |
| CI/CD | GitHub Actions (frontend já tem; backend falta) |
| Monitoramento | obrigatório, ferramenta a definir |

Detalhes e pendências: [DevOps e infraestrutura](../06-operacao/03-devops-e-infra.md).
