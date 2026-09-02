# Data Warehouse

O uso de um DW com modelagem dimensional é **requisito do desafio**, não escolha
livre. O que estava em aberto era a tecnologia.

## A escolha: Postgres

Escolhemos Postgres 16. A dúvida registrada pelo time foi honesta: *"nunca montamos um
DW antes, talvez não seja a melhor opção."* Vale destrinchar, porque a resposta é
**sim, é a opção certa para este projeto** — mas por motivos que convém entender.

### Por que Postgres serve aqui

| Argumento | Detalhe |
|---|---|
| **Escala real do projeto** | O escopo é **três tribunais** (TJSP, TJRJ, TJMG) com recorte deliberado de assuntos. Ordem de grandeza de centenas de milhares a poucos milhões de linhas de fato. Postgres lida com isso sem transpirar |
| **Modelagem dimensional é agnóstica** | Esquema estrela é um *modelo*, não uma tecnologia. Fato + dimensões + tabela-ponte se implementam em Postgres exatamente como se implementam em Redshift |
| **OLAP de verdade** | `GROUP BY … FILTER (WHERE …)`, `DISTINCT ON`, funções de janela, views materializadas — tudo nativo |
| **Full-text em português** | `to_tsvector('portuguese', …)` com `unaccent` e `pg_trgm`. Resolve a busca do produto sem subir um Elasticsearch só para isso |
| **Uma peça de infraestrutura** | Banco, motor de busca e camada OLAP no mesmo processo. Menos coisa para configurar, versionar e explicar na banca |
| **Testável** | Migrations em SQL versionado, subida em Docker, banco descartável no CI |

### Onde Postgres deixaria de servir

Registrado para não haver ilusão. Você trocaria de tecnologia se:

- o volume de fato passasse de **dezenas de milhões** de linhas com agregação
  interativa sobre todas elas (aí a resposta é colunar: DuckDB, ClickHouse, Athena
  sobre Parquet);
- fosse preciso varrer o DataJud inteiro em vez de três tribunais com recorte;
- o requisito incluísse processamento distribuído (Spark/Databricks).

Nada disso está no escopo. **Trocar de tecnologia agora seria complexidade sem ganho** —
e o desafio avalia modelagem dimensional e pipeline, não a marca do banco.

> Se quiser um argumento colunar barato sem trocar de stack: `DuckDB` lê Parquet e
> roda o mesmo SQL analítico, e cabe como camada de exploração ao lado do Postgres.
> Mas isso é otimização; não faça antes de ter o pipeline de pé.

## Extensões exigidas

| Extensão | Para quê |
|---|---|
| `unaccent` | o usuário digita "inscricao" e precisa achar "inscrição" |
| `pg_trgm` | similaridade por trigrama — rede de segurança para erro de digitação |

Ambas entram na primeira migration.

## Configuração relevante

| Item | Valor | Motivo |
|---|---|---|
| `LANG` | `pt_BR.utf8` | faz `ORDER BY` respeitar acento na ordenação de temas |
| Porta no host, em dev | não use 5432 | evita conflito com um Postgres já instalado na máquina |
| Configuração | por variável de ambiente | requisito do [Coolify](../06-operacao/03-devops-e-infra.md) |

> **Migrations precisam de runner de verdade.** Aplicar SQL via
> `docker-entrypoint-initdb.d` só funciona quando o volume está vazio — não serve para
> evoluir esquema com dado dentro. Escolher DbUp ou FluentMigrator, com controle de
> versão aplicada, e rodar no pipeline. Ver
> [DevOps](../06-operacao/03-devops-e-infra.md).

> **Backup é obrigatório.** O banco fica na mesma VPS da aplicação. Um DW perdido é uma
> recarga de dias contra fontes com limite de taxa.

## Camadas dentro do banco

```
tabelas base        fato + dimensões + pontes
      │             gravadas pelo ETL, nunca lidas direto pela API para agregar
      ▼
agregados           resultado vigente -> resumo por tema · por ano · por tribunal · por órgão
      │             atualizados ao fim de cada carga
      ▼
API / chatbot       leem só os agregados
```

> ⚠ **O esquema ainda não está fechado.** Ver
> [Modelo dimensional](../03-dados/02-modelo-dimensional.md) — a modelagem que existe
> no protótipo foi feita sem auditoria e não serve de base.

Detalhe das tabelas: [Modelo dimensional](../03-dados/02-modelo-dimensional.md).
Detalhe das views: [Agregados OLAP](../03-dados/03-agregados-olap.md).

## Testes de integridade

O desafio pede "testes automatizados validando integridade dos dados e consistência das
consultas". A lista está em
[DevOps e infraestrutura](../06-operacao/03-devops-e-infra.md#testes-de-integridade-do-dw--requisito-explícito),
junto com o pipeline que a executa.
