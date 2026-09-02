# Visão macro

## As camadas

```
┌─ 1 · FONTES (multifonte) ───────────────────────────────────────────┐
│  DataJud/CNJ      metadado processual em massa                      │
│  PANGEA / PDPJ    precedentes qualificados      (a investigar)      │
│  Repositórios do TJSP · TJRJ · TJMG   inteiro teor  (a investigar)  │
│  Doutrina         artigo com link · referência de livro (a definir) │
│  … outras fontes ainda não listadas                                 │
│                        escopo: SP · RJ · MG                         │
└──────────────────────────┬──────────────────────────────────────────┘
                           │  batch (as fontes não são tempo real)
┌─ 2 · ETL ─────────────────▼─────────────────────────────────────────┐
│  Extract    um conector por fonte, atrás de uma mesma porta         │
│  Transform  achatar · traduzir códigos · normalizar em tema (NLP)   │
│  Load       upsert idempotente, com proveniência em toda linha      │
│                                    projeto: Ratio.Etl (console app) │
└──────────────────────────┬──────────────────────────────────────────┘
                           │
┌─ 3 · DATA WAREHOUSE ──────▼─────────────────────────────────────────┐
│  Postgres · modelagem dimensional                                   │
│  ⚠ o esquema ainda NÃO está fechado — ver 03-dados                  │
└──────────────────────────┬──────────────────────────────────────────┘
                           │  atualizado ao fim de cada carga
┌─ 4 · AGREGADOS OLAP ──────▼─────────────────────────────────────────┐
│  resultado vigente por processo · resumo por tema · por ano ·       │
│  por tribunal · por órgão julgador                                  │
└──────────────────────────┬──────────────────────────────────────────┘
                           │
┌─ 5 · APLICAÇÃO ───────────▼─────────────────────────────────────────┐
│  API       ASP.NET Core 8 · Ratio.Api        (código em inglês)     │
│  Web       React                                                     │
│  Chatbot   LLM com tool use sobre os mesmos agregados  (roadmap)    │
└──────────────────────────┬──────────────────────────────────────────┘
                           │
┌─ 6 · DEVOPS ──────────────▼─────────────────────────────────────────┐
│  CI/CD · deploy automático · monitoramento · documentação           │
│  VPS Hostinger + Coolify                                            │
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

### Batch, não streaming

As fontes não são tempo real (o DataJud tem defasagem de horas a dias). Carga agendada,
reprocessando janelas que se sobrepõem — daí a **idempotência ser requisito**.

### Agregados pré-calculados

A tela de tema abre com várias agregações sobre a tabela fato. Rodar isso a cada request
deixaria a página lenta, e o dado só muda quando o ETL roda.

**Consequência:** o dado exibido é tão fresco quanto a última carga. A interface declara
a data de extração — os mockups já fazem isso.

### Busca full-text no próprio Postgres

`to_tsvector('portuguese', …)` com `unaccent` e `pg_trgm` para erro de digitação. Evita
subir um Elasticsearch só para o campo de busca, nesta escala.

### API somente leitura no domínio

O único caminho de gravação de dado de domínio é o ETL. Simplifica autenticação, CORS e
permissão de banco. A exceção aparente é a rota do chatbot, que não grava domínio.

### O chatbot consome os mesmos agregados

Não é um caminho alternativo até o banco: é uma interface de linguagem natural sobre as
consultas que as telas já usam. É isso que garante que ele responda os mesmos números
que a tela mostra. Ver [Chatbot](../01-produto/05-chatbot.md).

### Código em inglês, dado em português

Backend inteiro em inglês — identificadores, tabelas, rotas, campos JSON. Os **valores**
seguem em português, porque o domínio é o direito brasileiro. Vocabulário de tradução em
[Backend .NET](02-backend-dotnet.md#idioma).

## Contrato entre as camadas

| Fronteira | Contrato |
|---|---|
| Fonte → ETL | resposta bruta achatada em um DTO antes de tocar o banco |
| ETL → DW | SQL com upsert; nenhuma regra de negócio no banco além dos agregados |
| DW → API | os agregados são a interface; a API não agrega sobre o fato em tempo de request |
| API → Web | JSON em inglês, métricas pré-calculadas, proveniência em toda resposta |
| API → Chatbot | as mesmas consultas, expostas como ferramentas parametrizadas |

## Onde cada peça mora

| Camada | Repositório | Projeto / pasta |
|---|---|---|
| ETL | `API5-Backend` | `Ratio.Etl`, lógica em `Ratio.Application` |
| Migrations | `API5-Backend` | `Ratio.Infrastructure` |
| API | `API5-Backend` | `Ratio.Api` |
| Web | `API5-Frontend` | — |
| Infra / deploy | a definir | ver [DevOps](../06-operacao/03-devops-e-infra.md) |

## Infraestrutura

| Item | Escolha |
|---|---|
| Banco | Postgres (com `unaccent` e `pg_trgm`) |
| Contêineres | Docker — tudo containerizável é requisito do Coolify |
| Hospedagem | VPS **Hostinger** |
| Deploy | **Coolify**, automático |
| CI/CD | obrigatório, ferramenta a definir |
| Monitoramento | obrigatório, ferramenta a definir |

Detalhes e pendências: [DevOps e infraestrutura](../06-operacao/03-devops-e-infra.md).
