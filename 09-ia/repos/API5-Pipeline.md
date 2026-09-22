# API5-Pipeline

Leia antes o [`CLAUDE.md`](../CLAUDE.md) global.

## O que é

Roda do nosso lado, nunca no cliente. Coleta do DataJud, transforma em `raw` → `staging`
→ `dw` no **banco de carga** e gera o arquivo de carga que popula homologação e produção.
O `dw` do banco de carga é criado pela API (mesma migration); o pipeline cria só `raw`,
`staging` e `etl`.

## Módulos (`pipeline/`)

| Módulo | Papel |
|---|---|
| `tpu` | recorte cível pela TPU (`data/tpu.json`) |
| `datajud` / `raw_schema` | coleta paginada e idempotente em `raw.datajud_case` |
| `transform` / `staging` | achata o processo em eventos em `staging.case_event` |
| `dw_reference`, `dw_case`, `dw_subject`, `dw_fact` | carga dimensional |
| `dw_movement_polarity`, `dw_case_links` | polaridade e link para o tribunal |
| `theme_area`, `theme_build`, `theme_registry`, `theme_load` | temas com chave estável |
| `dump` / `load_file` | arquivo de carga via `pg_dump` |
| `run` | encadeia tudo |

## Comandos

```bash
pip install -r requirements.txt
ruff check .
pytest
```

## Regras próprias

- Dado curado (TPU, grupos de tema, regras de área) fica em `pipeline/data/`, copiado
  por script do original, nunca redigitado.
- Credenciais só por variável de ambiente (`DATABASE_URL`).
- Commit antes de chamar o `pg_dump`: ele abre outra conexão e trava se a transação
  estiver aberta.
- Nada de chamada de rede em teste automatizado: HTTP é falso nos testes.
