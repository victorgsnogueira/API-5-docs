# Modelagem dos três bancos

> **Estado em 20/09/2026**, gerado a partir do catálogo do Postgres do banco da carga, então os tipos, as chaves, os índices, as linhas e o preenchimento aqui são os reais. A decisão que originou esta modelagem é a [D-35](../06-operacao/02-decisoes-e-riscos.md#d-35--o-banco-do-cliente-é-o-dw-um-só-modelo-dw-nos-três-bancos). O modelo anterior está descrito em [Modelo dimensional](02-modelo-dimensional.md).

Esta página descreve, tabela por tabela e coluna por coluna, o que cada banco do projeto guarda, para que tipo e para quê. A modelagem já parte do que a fonte **consegue** entregar: onde não há dado, não há coluna.

## Sumário

1. [Os três bancos](#1--os-três-bancos)
2. [Convenções](#2--convenções)
3. [O que os dados permitem, e o que não](#3--o-que-os-dados-permitem-e-o-que-não)
4. [Schema `dw` — o DW entregue](#4--schema-dw--o-dw-entregue)
5. [Schema `etl` — memória da carga](#5--schema-etl--memória-da-carga)
6. [Schemas `raw`, `staging` e `nlp` — só na carga](#6--schemas-raw-staging-e-nlp--só-na-carga)
7. [Homologação e produção](#7--homologação-e-produção)
8. [O que saiu do modelo](#8--o-que-saiu-do-modelo)
9. [Testes de integridade](#9--testes-de-integridade)
10. [Ciclo de carga e ordem de atualização](#10--ciclo-de-carga-e-ordem-de-atualização)

---

## 1 · Os três bancos

| | **Carga** | **Homologação** | **Produção** |
|---|---|---|---|
| Onde roda | contêiner `api5-dw`, máquina do time | contêiner `ratio-homolog`, máquina do time | intranet do cliente, Windows Server |
| Imagem | `pgvector/pgvector:pg16` | `postgres:16` | Postgres 16 nativo |
| Schemas | `dw`, `etl`, `raw`, `staging`, `nlp` | **só `dw`** | **só `dw`** |
| pgvector | sim | não | não |
| Quem escreve | o pipeline (usuário `dw_admin`) | só o publicador (`ratio_loader`) | só o instalador |
| Quem lê | o pipeline | a API (`ratio_api`, somente `SELECT`) | a API (somente `SELECT`) |
| Como ganha estrutura | migrations `001` a `028` | troca atômica do schema `dw` a cada publicação | o mesmo `dw` da homologação |
| Tamanho hoje | 564 MB no `dw` (+ 261 MB `raw`, 509 MB `staging`, 107 MB `nlp`, 1128 kB `etl`) | ~564 MB | ainda não existe |

**A regra de corte é uma só: o que está no schema `dw` vai para a homologação e para a produção; o que está fora dele fica na carga.** Por isso o modelo de homologação e o de produção são **idênticos**, e o banco de carga tem o mesmo `dw` mais os schemas de trabalho. Não existe um segundo modelo a manter, nem transformação entre a carga e o destino.

```
  fontes públicas                  BANCO DA CARGA                                   HOMOLOGAÇÃO / PRODUÇÃO
  ───────────────    ┌────────────────────────────────────────────┐            ┌──────────────────────┐
  DataJud ─────────► │ raw ──► staging ──► dw  ◄── etl (memória)  │            │                      │
  DOAJ / SciELO ───► │                      ▲                     │  publica   │          dw          │
  OAI-PMH ─────────► │                      └── nlp (embeddings)  │ ─────────► │   (mesmo modelo)     │
  TPU do CNJ ──────► │                                            │  só o dw   │                      │
                     └────────────────────────────────────────────┘            └──────────┬───────────┘
                                                                                            │ só leitura
                                                                                            ▼
                                                                                         a API
```

---

## 2 · Convenções

| Assunto | Regra |
|---|---|
| Idioma | nomes de tabelas e colunas em **inglês**; textos e rótulos que o usuário lê, em **português** (D-06) |
| Chaves substitutas | sufixo `_sk`. `bigint` com identidade nas tabelas que crescem (`case_sk`, `event_sk`, `doctrine_sk`), `smallint` nas dimensões pequenas e fixas (tribunal, classe, movimentação, desfecho) |
| Chave pública | só o tema tem: `theme_key`, estável entre cargas. `theme_sk` é interno e é renumerado a cada recarga |
| Chave natural | única e declarada quando existe (`case_number`, `court_code`, `movement_code`, `subject_name`); é ela que torna a carga idempotente |
| Texto | sempre `text`. O Postgres não tem custo em `varchar(n)`, e um limite inventado só serviria para truncar dado de fonte |
| Datas e horas | instantes em `timestamptz`; dias em `date`. O dia de um evento é também uma chave inteira `AAAAMMDD` para `dim_date` |
| Proveniência | toda tabela de fato e toda dimensão de conteúdo carrega `source` e `extracted_at`. A data mostrada ao usuário vem daí, nunca do relógio do servidor (US-25) |
| Nulo | só onde a fonte pode não informar. Coluna sem nenhum valor não existe: foi retirada do modelo (ver [seção 8](#8--o-que-saiu-do-modelo)) |
| Domínios fechados | valores possíveis travados por regra (`CHECK`): grau, sigilo, desfecho, polaridade, área do produto, tipo de link |
| Colunas geradas | `search_vector` e `*_norm` são calculadas pelo banco a partir do nome, para a busca; não se escrevem |
| Vocabulário | **tema** é `theme` (D-34); **assunto** da TPU é `subject`. No banco não existe mais `topic` |

---

## 3 · O que os dados permitem, e o que não

A modelagem só tem coluna para o que a fonte entrega. Esta é a lista honesta do que existe hoje. Os resultados vão de 1976 a 2026, e os ajuizamentos de 1951 a 2026.

| Dado | Existe? | Situação |
|---|---|---|
| Movimentações do processo (data, tipo, órgão) | **sim** | 1.086.623 eventos de 18.002 processos, nos três tribunais (DataJud) |
| Resultado do processo (procedência, improcedência, parcial) | **em parte** | só 5.582 processos (31,0%) têm resultado apurado, e só 6 códigos de movimentação definem resultado. O restante é andamento sem julgamento |
| Quem propôs a pretensão (polaridade) | **em parte** | só 26% das classes definem quem propõe; nos demais o `n` aparece sem inverter o sentido do resultado (D-15) |
| Classe e assunto do processo (TPU) | **sim** | 121 classes e 1.075 assuntos cíveis, com código e área oficiais |
| Data de ajuizamento | **sim** | 100% dos processos. Em 747 processos o resultado é anterior ao ajuizamento e fica fora do tempo até a decisão |
| Órgão julgador | **sim** | 100% dos eventos. 14 nomes de câmara do TJRJ chegam corrompidos na fonte (`�`) e são guardados como recebidos |
| Tema | **derivado** | 1.049 temas por NLP e curadoria; a área do produto está em 56% deles, e os demais usam a área oficial da TPU |
| Doutrina | **em parte** | 52.696 artigos, dos quais 9.649 ligados a algum tema **por similaridade**. É doutrina relacionada, não invocada (D-33) |
| Link para o processo no tribunal | **sim** | por regra de URL. Direto no TJSP; portal no TJRJ e no TJMG |
| Relator | **não** | o DataJud não publica. Não há coluna |
| Inteiro teor e ementa | **não** | bloqueado nos 4 tribunais (CAPTCHA e termos de uso). Não há coluna |
| Valor da condenação | **não** | só existiria no inteiro teor |
| Fundamentos invocados | **não** | exige o texto da decisão |
| Precedentes qualificados (súmula, repetitivo, IRDR) | **não** | sem fonte verificada |
| Processos sigilosos | **não entram** | o DataJud público não os entrega; `secrecy_level` existe para garantir que nunca sejam exibidos |
| Matéria penal | **fora do escopo** | D-29. Conferido por testes |
| Livros de doutrina | **não hospedados** | obra protegida; só artigos, por link |

---

## 4 · Schema `dw` — o DW entregue

O `dw` é um modelo dimensional. **O grão declarado do fato é a movimentação processual** ([D-13](../06-operacao/02-decisoes-e-riscos.md#d-13--grão-do-fato-movimentação-processual-opção-a)): cada linha de `fact_case_event` é um evento datado em um processo. Tudo o que a tela mostra sai de agregados calculados sobre esse fato.

```
                              dim_date
                                 │
  dim_court ──┬──────────────────┼──────────────┬── dim_judging_body
              │                  ▼              │
  dim_case_class ─► dim_case ─► fact_case_event ◄─┴── dim_movement ──► dim_decision_outcome
                       │
                       │ bridge_case_subject
                       ▼
                  dim_subject ── bridge_theme_subject ──► dim_theme
                       │
                       └── bridge_subject_doctrine ──► dim_doctrine

  agregados (calculados do fato):  case_current_result ─► theme_summary · theme_by_year · theme_by_court
                                                          theme_by_judging_body · theme_time_to_decision
                                                          theme_strength · data_provenance
```

### 4.1 · Dimensões

#### `dim_court`

**Dimensão · tribunais do escopo** · tabela · 3 linhas

Os três tribunais que o produto cobre. Tabela fixa: só muda se o escopo mudar (decisão 2 do backlog).

- **Grão:** um tribunal
- **Fonte:** definida por nós (seed)

| Coluna | Tipo | Nulo? | O que guarda | Preenchimento |
|---|---|---|---|---:|
| `court_sk` | `smallint (identidade)` | não | Chave substituta do tribunal. | 100% |
| `court_code` | `text` | não | Sigla do tribunal (`TJSP`, `TJRJ`, `TJMG`). É a chave natural, única, e é o que o coletor usa para escolher o índice do DataJud (`api_publica_tjsp`). | 100% |
| `court_name` | `text` | não | Nome oficial por extenso do tribunal. | 100% |
| `state_uf` | `char(2)` | não | Sigla da unidade da federação, com 2 letras. | 100% |

Restrições:

- chave primária: `PRIMARY KEY (court_sk)`
- única: `UNIQUE (court_code)`

#### `dim_judging_body`

**Dimensão · órgãos julgadores** · tabela · 1.893 linhas

Vara, câmara, turma ou núcleo que decide, com o nome que o DataJud dá a ele. O mesmo nome pode existir em tribunais diferentes, por isso a unicidade é por tribunal. **Limite da fonte:** 14 órgãos do TJRJ (0,7% das câmaras, 7.586 eventos) chegam com o caractere de substituição `�` no nome; guardamos como recebido, sem corrigir por suposição.

- **Grão:** um órgão julgador de um tribunal
- **Fonte:** DataJud (`orgaoJulgador`)

| Coluna | Tipo | Nulo? | O que guarda | Preenchimento |
|---|---|---|---|---:|
| `judging_body_sk` | `bigint (identidade)` | não | Chave substituta do órgão julgador. | 100% |
| `court_sk` | `smallint` | não | Tribunal ao qual o órgão pertence (FK para `dim_court`). | 100% |
| `body_name` | `text` | não | Nome do órgão como a fonte o informa, por exemplo `1ª Câmara de Direito Privado`. Único dentro do tribunal. | 100% |

Restrições:

- chave estrangeira: `FOREIGN KEY (court_sk) REFERENCES dim_court(court_sk)`
- chave primária: `PRIMARY KEY (judging_body_sk)`
- única: `UNIQUE (court_sk, body_name)`

#### `dim_case_class`

**Dimensão · classes processuais** · tabela · 121 linhas

A natureza do processo (Procedimento Comum, Execução Fiscal, Embargos à Execução…), segundo a Tabela Processual Unificada do CNJ. Só classes cíveis existem aqui (D-29). A coluna `claimant_type` diz **quem propõe a pretensão** naquela classe, que é o que permite ler o desfecho sem inverter o sentido (D-15).

- **Grão:** uma classe processual da TPU
- **Fonte:** DataJud (`classe`) + regra nossa para `claimant_type`

| Coluna | Tipo | Nulo? | O que guarda | Preenchimento |
|---|---|---|---|---:|
| `case_class_sk` | `smallint (identidade)` | não | Chave substituta da classe. | 100% |
| `class_name` | `text` | não | Nome da classe na TPU. Único. | 100% |
| `claimant_type` | `text` | sim | Quem propõe a pretensão nessa classe: `fazenda`, `credor`, `defesa`, `autor_particular` ou `acusacao`. **Nulo quando a classe não determina isso** (74% das classes): nada é inferido. | 26,4% |

Restrições:

- regra: `CHECK (((claimant_type IS NULL) OR (claimant_type = ANY (ARRAY['acusacao'::text, 'fazenda'::text, 'credor'::text, 'defesa'::text, 'autor_particular'::text]))))`
- chave primária: `PRIMARY KEY (case_class_sk)`
- única: `UNIQUE (class_name)`

#### `dim_case`

**Dimensão · processos** · tabela · 18.002 linhas

Cada processo coletado. É a dimensão mais usada: liga o fato de movimentação, os assuntos e o resultado atual. Só entram processos cíveis (D-29).

- **Grão:** um processo (número CNJ único)
- **Fonte:** DataJud

| Coluna | Tipo | Nulo? | O que guarda | Preenchimento |
|---|---|---|---|---:|
| `case_sk` | `bigint (identidade)` | não | Chave substituta do processo. | 100% |
| `case_number` | `text` | não | Número CNJ com os 20 dígitos, sem máscara (`NNNNNNNDDAAAAJTROOOO`). Único. | 100% |
| `court_sk` | `smallint` | não | Tribunal do processo (FK para `dim_court`). | 100% |
| `case_class_sk` | `smallint` | sim | Classe processual (FK para `dim_case_class`). Nulo se a classe não estiver mapeada; hoje todos têm. | 100% |
| `court_level` | `text` | não | Grau de jurisdição: `First` (1º grau), `Second` (2º grau), `SpecialCourt` (juizado especial) ou `AppealPanel` (turma recursal). | 100% |
| `secrecy_level` | `smallint` | não | Nível de sigilo, de 0 a 5. O DataJud público só entrega processos públicos, então hoje todos valem 0; a coluna existe para a regra de nunca exibir processo sigiloso (NFR-21). | 100% |
| `filed_at` | `date` | sim | Data de ajuizamento. Base do tempo até a decisão (US-30). | 100% |
| `source` | `text` | não | Fonte do dado (`datajud`). | 100% |
| `extracted_at` | `timestamptz` | não | Quando a coleta trouxe o processo. | 100% |
| `case_number_formatted` | `text` | sim | Número com a máscara do CNJ (`NNNNNNN-DD.AAAA.J.TR.OOOO`), o que o usuário cola no portal do tribunal. | 100% |
| `source_link` | `text` | sim | Endereço para consultar o processo no tribunal. | 100% |
| `source_link_type` | `text` | sim | `direto` quando o link abre o processo (TJSP, e-SAJ) ou `portal` quando só abre o portal do tribunal (TJRJ e TJMG, que não aceitam parâmetro na URL). O rótulo na tela é sempre "consultar no tribunal". | 100% |

Restrições:

- regra: `CHECK ((court_level = ANY (ARRAY['First'::text, 'Second'::text, 'SpecialCourt'::text, 'AppealPanel'::text])))`
- regra: `CHECK (((secrecy_level >= 0) AND (secrecy_level <= 5)))`
- regra: `CHECK (((source_link IS NULL) = (source_link_type IS NULL)))`
- regra: `CHECK (((source_link_type IS NULL) OR (source_link_type = ANY (ARRAY['direto'::text, 'portal'::text]))))`
- chave estrangeira: `FOREIGN KEY (case_class_sk) REFERENCES dim_case_class(case_class_sk)`
- chave estrangeira: `FOREIGN KEY (court_sk) REFERENCES dim_court(court_sk)`
- chave primária: `PRIMARY KEY (case_sk)`
- única: `UNIQUE (case_number)`

Índices:

- `idx_dim_case_source_link_type`: `btree (source_link_type)`

#### `dim_date`

**Dimensão · calendário** · tabela · 32.142 linhas

Calendário de 01/01/1940 a 31/12/2027. Permite agrupar por ano, trimestre e mês sem calcular datas nas consultas.

- **Grão:** um dia
- **Fonte:** gerada

| Coluna | Tipo | Nulo? | O que guarda | Preenchimento |
|---|---|---|---|---:|
| `date_sk` | `integer` | não | Chave inteira no formato `AAAAMMDD` (por exemplo, `20260914`). | 100% |
| `full_date` | `date` | não | A data. Única. | 100% |
| `year` | `smallint` | não | Ano. | 100% |
| `quarter` | `smallint` | não | Trimestre, de 1 a 4. | 100% |
| `month` | `smallint` | não | Mês, de 1 a 12. | 100% |
| `month_name` | `text` | não | Nome do mês em inglês (`January`…). A tradução para a tela é da camada de apresentação. | 100% |
| `day` | `smallint` | não | Dia do mês. | 100% |

Restrições:

- chave primária: `PRIMARY KEY (date_sk)`
- única: `UNIQUE (full_date)`

#### `dim_movement`

**Dimensão · tipos de movimentação** · tabela · 420 linhas

Os 420 tipos de movimentação que apareceram nos processos, com a leitura de cada um. **Só 6 códigos são verificados**, os que de fato definem resultado (procedência, improcedência e procedência em parte, em 1º e 2º grau). Os demais são `Neutral`.

- **Grão:** um código de movimentação da TPU
- **Fonte:** DataJud (`movimentos`) + nomes resolvidos na TPU do CNJ

| Coluna | Tipo | Nulo? | O que guarda | Preenchimento |
|---|---|---|---|---:|
| `movement_sk` | `smallint (identidade)` | não | Chave substituta do tipo de movimentação. | 100% |
| `movement_code` | `integer` | não | Código do CNJ na TPU (`219` é Procedência). Único. | 100% |
| `movement_name` | `text` | não | Nome oficial. Quando a API do DataJud não manda o nome, ele é resolvido na TPU; 3 códigos locais que nem a TPU nomeia são descartados na carga. | 100% |
| `outcome_sk` | `smallint` | não | Desfecho a que o código se traduz (FK para `dim_decision_outcome`). O padrão é `Neutral`. | 100% |
| `code_verified` | `boolean` | não | Verdadeiro só quando o código foi conferido contra a TPU e tem polaridade declarada. Só movimentos verificados definem o resultado atual do processo. | 100% |
| `polarity_reference` | `text` | sim | A quem o desfecho se refere: `pretensao_autor` (o pedido de quem propôs) ou `pretensao_recorrente` (o pedido de quem recorreu). Obrigatório quando `code_verified`; nulo nos outros. Ver D-15. | 1,4% |

Restrições:

- regra: `CHECK (((polarity_reference IS NULL) OR (polarity_reference = ANY (ARRAY['pretensao_autor'::text, 'pretensao_recorrente'::text]))))`
- regra: `CHECK (((NOT code_verified) OR (polarity_reference IS NOT NULL)))`
- chave estrangeira: `FOREIGN KEY (outcome_sk) REFERENCES dim_decision_outcome(outcome_sk)`
- chave primária: `PRIMARY KEY (movement_sk)`
- única: `UNIQUE (movement_code)`

#### `dim_decision_outcome`

**Dimensão · desfechos** · tabela · 5 linhas

Vocabulário fechado de cinco desfechos. Só os três primeiros entram nas métricas (`counts_in_metric`).

- **Grão:** um desfecho possível
- **Fonte:** definida por nós (seed)

| Coluna | Tipo | Nulo? | O que guarda | Preenchimento |
|---|---|---|---|---:|
| `outcome_sk` | `smallint (identidade)` | não | Chave substituta do desfecho. | 100% |
| `outcome_code` | `text` | não | Código: `Granted`, `Denied`, `PartiallyGranted`, `Dismissed` ou `Neutral`. | 100% |
| `outcome_label` | `text` | não | Rótulo em português mostrado ao usuário (Provimento / Procedência, Improvimento / Improcedência, etc.). | 100% |
| `counts_in_metric` | `boolean` | não | Verdadeiro para procedência, improcedência e procedência em parte; falso para extinção sem mérito e movimentos neutros. | 100% |

Restrições:

- chave primária: `PRIMARY KEY (outcome_sk)`
- única: `UNIQUE (outcome_code)`

#### `dim_subject`

**Dimensão · assuntos da TPU** · tabela · 1.075 linhas

O que o processo discute, segundo a TPU do CNJ (`Indenização por Dano Moral`, `Plano de Saúde`…). São 1.075 assuntos cíveis. Cada assunto pertence a um tema (`bridge_theme_subject`). Substitui o antigo `dim_topic`.

- **Grão:** um assunto processual
- **Fonte:** DataJud (`assuntos`)

| Coluna | Tipo | Nulo? | O que guarda | Preenchimento |
|---|---|---|---|---:|
| `subject_sk` | `bigint (identidade)` | não | Chave substituta do assunto. | 100% |
| `subject_name` | `text` | não | Nome do assunto na TPU. Único. | 100% |
| `subject_code` | `integer` | não | Código do assunto na TPU do CNJ. Obrigatório: sem código não dá para saber se o assunto é cível. | 100% |
| `tpu_area` | `text` | sim | Área oficial da TPU sob a qual o assunto está (`DIREITO DO CONSUMIDOR`, `DIREITO CIVIL`…), com 15 valores. | 100% |
| `search_vector` | `tsvector (gerada)` | sim | Coluna gerada: vetor de busca textual em português, sem acento, do nome. | 100% |
| `subject_name_norm` | `text (gerada)` | sim | Coluna gerada: nome em minúsculas e sem acento, usado na busca por similaridade de trigramas. | 100% |

Restrições:

- chave primária: `PRIMARY KEY (subject_sk)`
- única: `UNIQUE (subject_name)`

Índices:

- `idx_dim_subject_fts`: `gin (search_vector)`
- `idx_dim_subject_name_trgm`: `gin (subject_name_norm gin_trgm_ops)`
- `idx_dim_subject_subject_code`: `btree (subject_code)`
- `idx_dim_subject_tpu_area`: `btree (tpu_area)`

#### `dim_theme`

**Dimensão · temas (a entidade central do produto)** · tabela · 1.049 linhas

O tema agrupa assuntos da TPU que dizem a mesma coisa em direito. É o que a busca devolve e o que as telas mostram. O rótulo é curado e auditável, e os assuntos de origem ficam sempre acessíveis.

- **Grão:** um tema
- **Fonte:** NLP + curadoria sobre `dim_subject`

| Coluna | Tipo | Nulo? | O que guarda | Preenchimento |
|---|---|---|---|---:|
| `theme_sk` | `bigint (identidade)` | não | Chave interna do tema. **É renumerada a cada recarga**: nunca exponha na API nem em URL. | 100% |
| `theme_name` | `text` | não | Rótulo do tema. Único. | 100% |
| `subject_area` | `text` | sim | Área do produto (`CIVIL`, `CONSUMIDOR`, `SAUDE`…, 14 valores). **Nulo em 43,8% dos temas**, os de poucos julgados sem curadoria; nulo significa "sem tag do produto", e a tela usa `tpu_area`. | 56,2% |
| `label_origin` | `text` | não | Como o rótulo nasceu. `llm_curated` é rótulo proposto com apoio de modelo e revisado; a regra do produto é que a origem do rótulo é sempre registrada. | 100% |
| `embedding_model` | `text` | sim | Modelo de embeddings que apoiou o agrupamento dos assuntos. | 100% |
| `created_at` | `timestamptz` | não | Quando o tema foi carregado. | 100% |
| `tpu_area` | `text` | sim | Área oficial da TPU predominante entre os assuntos do tema. Preenchida em todos. | 100% |
| `search_vector` | `tsvector (gerada)` | sim | Coluna gerada: vetor de busca textual em português, sem acento, do nome do tema. | 100% |
| `theme_name_norm` | `text (gerada)` | sim | Coluna gerada: nome em minúsculas e sem acento, usado na busca por similaridade de trigramas. | 100% |
| `theme_key` | `bigint` | não | Chave pública do tema, estável entre cargas (D-31). É a que vai na rota `/api/themes/{key}`. | 100% |

Restrições:

- regra: `CHECK (((subject_area IS NULL) OR (subject_area = ANY (ARRAY['ADMINISTRATIVO'::text, 'AMBIENTAL'::text, 'BANCARIO'::text, 'CIVIL'::text, 'CONSUMIDOR'::text, 'EMPRESARIAL'::text, 'FAMILIA'::text, 'IMOBILIARIO'::text, 'PREVIDENCIARIO'::text, 'PROCESSUAL'::text, 'QUANTUM'::text, 'SAUDE'::text, 'TRABALHISTA'::text, 'TRIBUTARIO'::text]))))`
- chave primária: `PRIMARY KEY (theme_sk)`
- única: `UNIQUE (theme_name)`

Índices:

- `idx_dim_theme_fts`: `gin (search_vector)`
- `idx_dim_theme_key`: única, `btree (theme_key)`
- `idx_dim_theme_name_trgm`: `gin (theme_name_norm gin_trgm_ops)`

#### `dim_doctrine`

**Dimensão · doutrina (artigos acadêmicos)** · tabela · 52.696 linhas

Artigos coletados do DOAJ, SciELO e repositórios OAI-PMH (EMERJ, EJEF/TJMG, IndexLaw). São 52.696, mas **só 9.649 estão ligados a algum tema**. Livros não entram: são obra protegida e nunca são hospedados.

- **Grão:** um artigo
- **Fonte:** DOAJ, SciELO, OAI-PMH

| Coluna | Tipo | Nulo? | O que guarda | Preenchimento |
|---|---|---|---|---:|
| `doctrine_sk` | `bigint (identidade)` | não | Chave substituta do artigo. | 100% |
| `title` | `text` | não | Título do artigo. | 100% |
| `authors` | `text` | sim | Autores, em um único texto, como a fonte os entrega. | 99,8% |
| `journal_name` | `text` | sim | Revista ou repositório em que foi publicado. | 100% |
| `publication_year` | `smallint` | sim | Ano de publicação. | 100% |
| `doi` | `text` | sim | Identificador DOI. **Nulo em 46%** dos artigos; único quando existe. | 53,7% |
| `article_url` | `text` | sim | Endereço do artigo na fonte. Pelo menos um entre `doi` e `article_url` sempre existe. | 99,9% |
| `subject_area` | `text` | sim | Classificação temática da fonte, geralmente em inglês (`Law in general. Comparative and uniform law`). **Não é a área do produto.** | 100% |
| `source` | `text` | não | Origem: `doaj`, `scielo`, `oai_emerj`, `oai_ejef_tjmg` ou `oai_indexlaw`. | 100% |
| `extracted_at` | `timestamptz` | não | Quando o artigo foi coletado. | 100% |

Restrições:

- regra: `CHECK (((doi IS NOT NULL) OR (article_url IS NOT NULL)))`
- chave primária: `PRIMARY KEY (doctrine_sk)`
- única: `UNIQUE (doi)`
- única: `UNIQUE (source, article_url)`

### 4.2 · Fato

#### `fact_case_event`

**Fato · movimentação processual** · tabela · 1.086.623 linhas

A tabela fato do DW. Cada linha é um evento no andamento de um processo (distribuição, juntada, sentença, recurso…). Os agregados de decisão (`case_current_result` e tudo que sai dele) são calculados a partir dela. Não tem medida numérica: o que se conta são as próprias linhas.

- **Grão:** **uma movimentação processual**: um evento datado que ocorreu num processo (D-13)
- **Fonte:** DataJud (`movimentos`)

| Coluna | Tipo | Nulo? | O que guarda | Preenchimento |
|---|---|---|---|---:|
| `event_sk` | `bigint (identidade)` | não | Chave substituta do evento. | 100% |
| `case_sk` | `bigint` | não | Processo em que o evento ocorreu (FK para `dim_case`). | 100% |
| `court_sk` | `smallint` | não | Tribunal (FK para `dim_court`). Repete o do processo para filtrar sem junção. | 100% |
| `judging_body_sk` | `bigint` | sim | Órgão julgador do evento (FK para `dim_judging_body`). Nulo se a fonte não informa; hoje sempre informado. | 100% |
| `movement_sk` | `smallint` | não | Tipo de movimentação (FK para `dim_movement`). | 100% |
| `date_sk` | `integer` | sim | Dia do evento (FK para `dim_date`). | 100% |
| `occurred_at` | `timestamptz` | não | Instante exato do evento, com fuso horário. Nunca é inventado: evento sem data é descartado na carga (D-11). | 100% |
| `source` | `text` | não | Fonte do dado (`datajud`). | 100% |
| `source_url` | `text` | não | Endereço do índice do DataJud de onde veio, um por tribunal. | 100% |
| `extracted_at` | `timestamptz` | não | Quando a coleta trouxe o evento. Alimenta a data de extração mostrada ao usuário (US-25). | 100% |
| `natural_key` | `text` | não | `número do processo | código da movimentação | instante`. Único; é o que torna a carga idempotente. | 100% |

Restrições:

- chave estrangeira: `FOREIGN KEY (case_sk) REFERENCES dim_case(case_sk)`
- chave estrangeira: `FOREIGN KEY (court_sk) REFERENCES dim_court(court_sk)`
- chave estrangeira: `FOREIGN KEY (date_sk) REFERENCES dim_date(date_sk)`
- chave estrangeira: `FOREIGN KEY (judging_body_sk) REFERENCES dim_judging_body(judging_body_sk)`
- chave estrangeira: `FOREIGN KEY (movement_sk) REFERENCES dim_movement(movement_sk)`
- chave primária: `PRIMARY KEY (event_sk)`
- única: `UNIQUE (natural_key)`

Índices:

- `idx_fact_case_event_case`: `btree (case_sk)`
- `idx_fact_case_event_court`: `btree (court_sk)`
- `idx_fact_case_event_date`: `btree (date_sk)`
- `idx_fact_case_event_extracted_at`: `btree (extracted_at)`
- `idx_fact_case_event_movement`: `btree (movement_sk)`

### 4.3 · Pontes

#### `bridge_case_subject`

**Ponte · processo ↔ assunto** · tabela · 25.943 linhas

Um processo pode discutir vários assuntos, e um assunto aparece em muitos processos. Sem esta ponte a contagem por tema duplicaria linhas.

- **Grão:** um par processo–assunto
- **Fonte:** DataJud (`assuntos`)

| Coluna | Tipo | Nulo? | O que guarda | Preenchimento |
|---|---|---|---|---:|
| `case_sk` | `bigint` | não | Processo (FK para `dim_case`). | 100% |
| `subject_sk` | `bigint` | não | Assunto (FK para `dim_subject`). | 100% |

Restrições:

- chave estrangeira: `FOREIGN KEY (case_sk) REFERENCES dim_case(case_sk)`
- chave estrangeira: `FOREIGN KEY (subject_sk) REFERENCES dim_subject(subject_sk)`
- chave primária: `PRIMARY KEY (case_sk, subject_sk)`

Índices:

- `idx_bridge_case_subject_subject`: `btree (subject_sk)`

#### `bridge_theme_subject`

**Ponte · tema ↔ assunto** · tabela · 1.075 linhas

Diz quais assuntos formam cada tema. Hoje cada assunto está em um único tema (1.075 assuntos em 1.049 temas), mas o desenho admite N:N.

- **Grão:** um par tema–assunto
- **Fonte:** curadoria de temas

| Coluna | Tipo | Nulo? | O que guarda | Preenchimento |
|---|---|---|---|---:|
| `theme_sk` | `bigint` | não | Tema (FK para `dim_theme`, com exclusão em cascata). | 100% |
| `subject_sk` | `bigint` | não | Assunto (FK para `dim_subject`). | 100% |

Restrições:

- chave estrangeira: `FOREIGN KEY (subject_sk) REFERENCES dim_subject(subject_sk)`
- chave estrangeira: `FOREIGN KEY (theme_sk) REFERENCES dim_theme(theme_sk) ON DELETE CASCADE`
- chave primária: `PRIMARY KEY (theme_sk, subject_sk)`

Índices:

- `idx_bridge_theme_subject_subject`: `btree (subject_sk)`

#### `bridge_subject_doctrine`

**Ponte · assunto ↔ doutrina** · tabela · 13.870 linhas

A doutrina **relacionada** a um assunto, por similaridade semântica de títulos com filtro léxico e limiar declarado (D-33). Não é doutrina citada por decisão nenhuma. Para chegar da doutrina ao tema, passa-se por `bridge_theme_subject`.

- **Grão:** um par assunto–artigo
- **Fonte:** cálculo nosso (embeddings)

| Coluna | Tipo | Nulo? | O que guarda | Preenchimento |
|---|---|---|---|---:|
| `subject_sk` | `bigint` | não | Assunto (FK para `dim_subject`). | 100% |
| `doctrine_sk` | `bigint` | não | Artigo (FK para `dim_doctrine`). | 100% |
| `link_method` | `text` | não | Como a ligação foi feita (`embedding_cosine+lexical`). Obrigatório: ligação sem método não é auditável. | 100% |
| `similarity` | `real` | sim | Cosseno entre os embeddings, de 0,55 (o limiar) a 1,0. | 100% |
| `embedding_model` | `text` | sim | Modelo que produziu os embeddings. | 100% |

Restrições:

- chave estrangeira: `FOREIGN KEY (doctrine_sk) REFERENCES dim_doctrine(doctrine_sk)`
- chave estrangeira: `FOREIGN KEY (subject_sk) REFERENCES dim_subject(subject_sk)`
- chave primária: `PRIMARY KEY (subject_sk, doctrine_sk)`

### 4.4 · Configuração

#### `strength_config`

**Configuração · score de força** · tabela · 1 linhas

Os pesos e parâmetros do score de 0 a 100 e a versão da metodologia. Ficam no banco, e não no código, para o score poder ser auditado e recalculado por um teste independente.

- **Grão:** uma linha só (`id = 1`)
- **Fonte:** definida por nós

| Coluna | Tipo | Nulo? | O que guarda | Preenchimento |
|---|---|---|---|---:|
| `id` | `smallint` | não | Sempre 1: a tabela só admite uma linha. | 100% |
| `weight_agreement` | `real` | não | Peso da concordância no score (0,45). | 100% |
| `weight_volume` | `real` | não | Peso do volume (0,25). | 100% |
| `weight_coverage` | `real` | não | Peso da cobertura de tribunais (0,20). | 100% |
| `weight_recency` | `real` | não | Peso da recência (0,10). Os quatro pesos somam 1,0 (restrição). | 100% |
| `volume_saturation` | `integer` | não | Número de julgados a partir do qual o componente volume satura (300). | 100% |
| `coverage_courts` | `smallint` | não | Quantos tribunais existem no escopo, base da cobertura (3). | 100% |
| `reference_year` | `smallint` | não | Ano de referência da recência. Por padrão, o ano corrente. | 100% |
| `methodology_version` | `text` | não | Versão da metodologia, exibida junto à proveniência (`1.0`, provisória). | 100% |
| `min_judged_for_percentage` | `smallint` | não | Abaixo desse número de julgados a tela mostra a contagem, nunca o percentual (2, D-32). | 100% |

Restrições:

- regra: `CHECK ((abs(((((weight_agreement + weight_volume) + weight_coverage) + weight_recency) - (1.0)::double precision)) < (0.001)::double precision))`
- regra: `CHECK ((id = 1))`
- regra: `CHECK ((min_judged_for_percentage >= 1))`
- chave primária: `PRIMARY KEY (id)`

#### `search_synonym`

**Configuração · sinônimos de busca** · tabela · 4 linhas

Traduz jargão forense que não existe no vocabulário da TPU ("negativação") para o vocabulário dos temas. É curadoria manual e não escala sozinha (D-30).

- **Grão:** um termo
- **Fonte:** curadoria manual

| Coluna | Tipo | Nulo? | O que guarda | Preenchimento |
|---|---|---|---|---:|
| `term` | `text` | não | Termo digitado, normalizado (minúsculo, sem acento). | 100% |
| `expands_to` | `text` | não | Termos com que ele é substituído na busca. | 100% |
| `note` | `text` | sim | Por que a entrada existe. | 100% |

Restrições:

- chave primária: `PRIMARY KEY (term)`

### 4.5 · Agregados

Os agregados são **views materializadas**: o resultado é guardado e só muda quando a carga manda atualizar. As perguntas do produto são respondidas por eles, nunca por agregação sobre o fato em tempo de requisição. A ordem de atualização está na [seção 10](#10--ciclo-de-carga-e-ordem-de-atualização).

#### `case_current_result`

**Agregado · resultado atual do processo** · agregado materializado · 5.582 linhas

Para cada processo, o **último movimento verificado que define resultado**. É a base de todos os agregados de decisão. Só 31% dos processos têm resultado (5.582 de 18.002); o resto é andamento sem julgamento apurado.

- **Grão:** um processo que tem resultado

| Coluna | Tipo | Nulo? | O que guarda | Preenchimento |
|---|---|---|---|---:|
| `case_sk` | `bigint` | sim | Processo. | — |
| `event_sk` | `bigint` | sim | Evento que definiu o resultado. | — |
| `outcome_sk` | `smallint` | sim | Desfecho (procedência, improcedência ou parcial). | — |
| `polarity_reference` | `text` | sim | A quem o desfecho se refere (`pretensao_autor` ou `pretensao_recorrente`). | — |
| `claimant_type` | `text` | sim | Quem propôs a pretensão, pela classe (pode ser nulo). | — |
| `date_sk` | `integer` | sim | Dia do resultado. | — |
| `court_sk` | `smallint` | sim | Tribunal. | — |
| `judging_body_sk` | `bigint` | sim | Órgão julgador. | — |

Índices:

- `idx_case_current_result_case`: única, `btree (case_sk)`
- `idx_case_current_result_polarity`: `btree (polarity_reference)`

#### `theme_summary`

**Agregado · resumo do tema** · agregado materializado · 1.049 linhas

Volumes e contagens do tema: quantos processos, quantos com resultado, quantos acolhidos e rejeitados, período e última decisão. Não guarda percentual: o cálculo é da camada de apresentação (nada é calculado na tela, mas também nada é escondido aqui).

- **Grão:** um tema

| Coluna | Tipo | Nulo? | O que guarda | Preenchimento |
|---|---|---|---|---:|
| `theme_sk` | `bigint` | sim | Tema. | — |
| `theme_name` | `text` | sim | Nome do tema. | — |
| `subject_area` | `text` | sim | Área do produto (pode ser nula). | — |
| `subject_count` | `bigint` | sim | Quantos assuntos formam o tema. | — |
| `case_count` | `bigint` | sim | Quantos processos têm algum assunto do tema. | — |
| `court_count` | `bigint` | sim | Em quantos tribunais há resultado. | — |
| `judged_case_count` | `bigint` | sim | Quantos processos têm resultado. | — |
| `claim_upheld_count` | `bigint` | sim | Processos em que a pretensão de quem propôs foi acolhida (procedência ou parcial). | — |
| `claim_rejected_count` | `bigint` | sim | Processos em que a pretensão de quem propôs foi rejeitada. | — |
| `appeal_upheld_count` | `bigint` | sim | Processos em que a pretensão de quem recorreu foi acolhida. | — |
| `appeal_rejected_count` | `bigint` | sim | Processos em que a pretensão de quem recorreu foi rejeitada. | — |
| `unknown_claimant_count` | `bigint` | sim | Processos com resultado cuja classe não diz quem propôs. | — |
| `period_start_year` | `smallint` | sim | Ano do resultado mais antigo. | — |
| `period_end_year` | `smallint` | sim | Ano do resultado mais recente. | — |
| `last_decision_date` | `date` | sim | Data do resultado mais recente. | — |
| `claimant_breakdown` | `jsonb` | sim | Contagem de processos por tipo de proponente, em JSON. | — |
| `dominant_claimant` | `text` | sim | Tipo de proponente mais frequente. | — |
| `claim_polarity_label` | `text` | sim | Frase que diz o que "acolhido" significa neste tema (por exemplo, "acolhimento da pretensão da Fazenda Pública"). Existe para não mostrar "favorável" sem dizer a quem (D-15). | — |

Índices:

- `idx_theme_summary_pk`: única, `btree (theme_sk)`

#### `theme_by_year`

**Agregado · tema por ano** · agregado materializado · 2.196 linhas

Série anual do alinhamento do tema (US-11).

- **Grão:** um tema em um ano

| Coluna | Tipo | Nulo? | O que guarda | Preenchimento |
|---|---|---|---|---:|
| `theme_sk` | `bigint` | sim | Tema. | — |
| `year` | `smallint` | sim | Ano do resultado. | — |
| `claim_upheld_count` | `bigint` | sim | Pretensões acolhidas no ano. | — |
| `claim_rejected_count` | `bigint` | sim | Pretensões rejeitadas no ano. | — |

Índices:

- `idx_theme_by_year_pk`: única, `btree (theme_sk, year)`

#### `theme_by_court`

**Agregado · tema por tribunal** · agregado materializado · 1.063 linhas

Alinhamento do tema em cada tribunal (US-12 e US-14).

- **Grão:** um tema em um tribunal

| Coluna | Tipo | Nulo? | O que guarda | Preenchimento |
|---|---|---|---|---:|
| `theme_sk` | `bigint` | sim | Tema. | — |
| `court_code` | `text` | sim | Sigla do tribunal. | — |
| `claim_upheld_count` | `bigint` | sim | Pretensões acolhidas. | — |
| `claim_rejected_count` | `bigint` | sim | Pretensões rejeitadas. | — |

Índices:

- `idx_theme_by_court_pk`: única, `btree (theme_sk, court_code)`

#### `theme_by_judging_body`

**Agregado · tema por órgão julgador (novo)** · agregado materializado · 4.770 linhas

Alinhamento de cada câmara ou turma no tema, para o painel "por órgão" e para a divergência interna do tribunal (US-17). Não corta órgãos de poucos casos: a regra de exibição (`min_judged_for_percentage`) é da tela, não do dado. A soma por tribunal fecha exatamente com `theme_by_court` (teste 37).

- **Grão:** um tema em um órgão julgador de um tribunal

| Coluna | Tipo | Nulo? | O que guarda | Preenchimento |
|---|---|---|---|---:|
| `theme_sk` | `bigint` | sim | Tema. | — |
| `court_code` | `text` | sim | Sigla do tribunal. | — |
| `body_name` | `text` | sim | Nome do órgão julgador. | — |
| `case_count` | `bigint` | sim | Processos com resultado naquele órgão. | — |
| `claim_upheld_count` | `bigint` | sim | Pretensões acolhidas. | — |
| `claim_rejected_count` | `bigint` | sim | Pretensões rejeitadas. | — |

Índices:

- `idx_theme_by_judging_body_pk`: única, `btree (theme_sk, court_code, body_name)`

#### `theme_time_to_decision`

**Agregado · tempo até a decisão (novo)** · agregado materializado · 638 linhas

Quantos dias, em geral, entre o ajuizamento e o resultado (US-30). Considera só processos com data de ajuizamento e cujo resultado não seja anterior a ela: 747 processos ficam de fora por essa anomalia da fonte.

- **Grão:** um tema

| Coluna | Tipo | Nulo? | O que guarda | Preenchimento |
|---|---|---|---|---:|
| `theme_sk` | `bigint` | sim | Tema. | — |
| `case_count` | `bigint` | sim | Processos usados no cálculo. | — |
| `p25_days` | `integer` | sim | Primeiro quartil, em dias. | — |
| `median_days` | `integer` | sim | Mediana, em dias. | — |
| `p75_days` | `integer` | sim | Terceiro quartil, em dias. | — |

Índices:

- `idx_theme_time_to_decision_pk`: única, `btree (theme_sk)`

#### `theme_strength`

**Agregado · score de força do entendimento** · agregado materializado · 1.049 linhas

O score de 0 a 100 com os quatro componentes abertos, os pesos e a base de cálculo. Cada componente vem de 0 a 1 e o score é a soma ponderada. É reproduzido do zero por um teste (teste 3 de `014`), então a fórmula não pode mentir.

- **Grão:** um tema

| Coluna | Tipo | Nulo? | O que guarda | Preenchimento |
|---|---|---|---|---:|
| `theme_sk` | `bigint` | sim | Tema. | — |
| `theme_name` | `text` | sim | Nome do tema. | — |
| `subject_area` | `text` | sim | Área do produto (pode ser nula). | — |
| `score` | `integer` | sim | Score de 0 a 100. | — |
| `level` | `text` | sim | Grau em linguagem jurídica: `Consolidada` (90+), `Dominante` (75+), `Em formação` (55+), `Divergente`. | — |
| `agreement_value` | `numeric` | sim | Concordância, de 0 a 1: quão distante de 50/50 está o desfecho. | — |
| `agreement_weight` | `real` | sim | Peso da concordância. | — |
| `volume_value` | `numeric` | sim | Volume, de 0 a 1, em escala logarítmica saturando em `volume_saturation`. | — |
| `volume_weight` | `real` | sim | Peso do volume. | — |
| `coverage_value` | `numeric` | sim | Cobertura, de 0 a 1: tribunais com resultado sobre os do escopo. | — |
| `coverage_weight` | `real` | sim | Peso da cobertura. | — |
| `recency_value` | `numeric` | sim | Recência, de 0 a 1, pelo ano da última decisão. | — |
| `recency_weight` | `real` | sim | Peso da recência. | — |
| `agreement_basis` | `text` | sim | Base da concordância: `pretensao_autor` ou `pretensao_recorrente`. Nunca mistura as duas famílias. | — |
| `dominant_claimant` | `text` | sim | Tipo de proponente mais frequente. | — |
| `claim_polarity_label` | `text` | sim | O que "acolhido" significa neste tema. | — |
| `claimant_breakdown` | `jsonb` | sim | Contagem por tipo de proponente, em JSON. | — |
| `unknown_claimant_count` | `bigint` | sim | Resultados sem proponente identificado. | — |
| `judged` | `bigint` | sim | Julgados na base de cálculo: **o `n` que a tela mostra ao lado de todo percentual**. | — |
| `upheld` | `bigint` | sim | Acolhidos na base de cálculo. | — |
| `rejected` | `bigint` | sim | Rejeitados na base de cálculo. | — |
| `claim_upheld_count` | `bigint` | sim | Pretensões do autor acolhidas. | — |
| `claim_rejected_count` | `bigint` | sim | Pretensões do autor rejeitadas. | — |
| `appeal_upheld_count` | `bigint` | sim | Pretensões do recorrente acolhidas. | — |
| `appeal_rejected_count` | `bigint` | sim | Pretensões do recorrente rejeitadas. | — |
| `courts` | `bigint` | sim | Tribunais com resultado. | — |
| `last_decision_year` | `smallint` | sim | Ano da última decisão. | — |
| `last_decision_date` | `date` | sim | Data da última decisão. | — |
| `volume_saturation` | `integer` | sim | Parâmetro usado no cálculo (cópia de `strength_config`). | — |
| `coverage_courts` | `smallint` | sim | Parâmetro usado no cálculo (cópia de `strength_config`). | — |
| `reference_year` | `smallint` | sim | Parâmetro usado no cálculo (cópia de `strength_config`). | — |

Índices:

- `idx_theme_strength_pk`: única, `btree (theme_sk)`
- `idx_theme_strength_score`: `btree (score DESC)`

#### `data_provenance`

**Agregado · proveniência global** · agregado materializado · 6 linhas

Fonte, data de extração e volume de cada conjunto de dados, para o rodapé da US-25. Era uma view que agregava 1 milhão de linhas a cada chamada (1,15 s); virou agregado materializado, atualizado na carga.

- **Grão:** um par bloco–fonte

| Coluna | Tipo | Nulo? | O que guarda | Preenchimento |
|---|---|---|---|---:|
| `block` | `text` | sim | `casos` ou `doutrina`. | — |
| `source` | `text` | sim | Fonte (`datajud`, `doaj`, `scielo`…). | — |
| `extracted_at` | `timestamptz` | sim | Data mais recente de extração. | — |
| `row_count` | `bigint` | sim | Quantas linhas vieram dessa fonte. | — |

Índices:

- `idx_data_provenance_pk`: única, `btree (block, source)`

### 4.6 · Views

#### `theme_case_export`

**View · processos do tema para amostra e exportação** · view

Alimenta a amostra auditável (US-15) e a exportação em CSV (US-27). Os nomes das colunas já estão em português, prontos para o arquivo.

- **Grão:** um processo com resultado em um tema

| Coluna | Tipo | Nulo? | O que guarda | Preenchimento |
|---|---|---|---|---:|
| `theme_sk` | `bigint` | sim | Tema. | — |
| `theme_key` | `bigint` | sim | Chave pública do tema. | — |
| `theme_name` | `text` | sim | Nome do tema. | — |
| `numero_processo` | `text` | sim | Número CNJ com máscara. | — |
| `tribunal` | `text` | sim | Sigla do tribunal. | — |
| `orgao_julgador` | `text` | sim | Órgão julgador do resultado. | — |
| `classe` | `text` | sim | Classe processual. | — |
| `grau` | `text` | sim | Grau de jurisdição (`First`, `Second`…). | — |
| `data_do_julgamento` | `date` | sim | Data do resultado. | — |
| `resultado` | `text` | sim | Desfecho em português. | — |
| `polarity_reference` | `text` | sim | A quem o desfecho se refere. | — |
| `ajuizado_em` | `date` | sim | Data de ajuizamento. | — |
| `source` | `text` | sim | Fonte do dado. | — |
| `source_link` | `text` | sim | Endereço para consultar no tribunal. | — |
| `source_link_type` | `text` | sim | `direto` ou `portal`. | — |
| `extracted_at` | `timestamptz` | sim | Quando o processo foi coletado. | — |

#### `theme_provenance`

**View · proveniência de um tema** · view

Fonte, data de extração e volume por tema, separando o bloco de casos do de doutrina, mais a versão da metodologia (US-25).

- **Grão:** um par tema–bloco–fonte

| Coluna | Tipo | Nulo? | O que guarda | Preenchimento |
|---|---|---|---|---:|
| `theme_sk` | `bigint` | sim | Tema. | — |
| `block` | `text` | sim | `casos` ou `doutrina`. | — |
| `source` | `text` | sim | Fonte. | — |
| `extracted_at` | `timestamptz` | sim | Data mais recente de extração. | — |
| `row_count` | `bigint` | sim | Quantas linhas (processos ou artigos) sustentam o tema. | — |
| `methodology_version` | `text` | sim | Versão da metodologia. | — |

### 4.7 · Funções e busca

A busca de temas roda no próprio Postgres ([D-30](../06-operacao/02-decisoes-e-riscos.md#d-30--busca-de-temas-em-português)), sem serviço externo.

| Objeto | Assinatura | O que faz |
|---|---|---|
| `dw.pt_unaccent` | configuração de busca textual | Português com `unaccent` aplicado antes do radical. Ao tirar o acento antes, o radical degrada (`indenização` → `indenizaca`); por isso a similaridade de trigramas roda em paralelo e cobre plural e flexão |
| `dw.expand_query` | `(p_query text)` | Aplica os sinônimos de `search_synonym` a uma consulta, palavra por palavra. |
| `dw.norm_pt` | `(txt text)` | Passa o texto para minúsculas e tira os acentos. É a base da busca. |
| `dw.search_themes` | `(p_query text, p_limit integer DEFAULT 20)` | A busca de temas (US-01). Combina busca textual e similaridade de trigramas, considera só temas com pelo menos um julgado, corta abaixo de 0,5 de rank e devolve o tema, a área, o `n`, o score, o grau, o rank e o tipo de correspondência (`texto`, `similaridade` ou `texto+similaridade`). |
| `dw.search_themes_top` | `(p_limit integer DEFAULT 20)` | A busca vazia: os temas de maior volume de julgados. |

---

## 5 · Schema `etl` — memória da carga

Tabelas que a carga precisa lembrar entre uma execução e a próxima, mas que **não fazem parte do que é entregue**. Ficam fora do `dw` justamente para a regra de corte ser por schema, e não por lista de exceções. A recarga (`TRUNCATE`) não toca nelas.

#### `theme_registry`

**Carga · memória das chaves públicas** · tabela · 1.049 linhas

Guarda a `theme_key` de cada tema desde a primeira vez que ele apareceu. **Nunca pode ser truncada nem descartada**: é o que impede o link de um tema de apontar para outro depois de uma recarga (D-31).

- **Grão:** um tema já publicado
- **Fonte:** gerada na carga

| Coluna | Tipo | Nulo? | O que guarda | Preenchimento |
|---|---|---|---|---:|
| `theme_key` | `bigint (identidade)` | não | Chave pública, identidade única e nunca reatribuída. | 100% |
| `theme_name` | `text` | não | Nome do tema a que a chave pertence. Único. | 100% |
| `first_seen` | `timestamptz` | não | Quando o tema apareceu pela primeira vez. | 100% |

Restrições:

- chave primária: `PRIMARY KEY (theme_key)`
- única: `UNIQUE (theme_name)`

#### `theme_area_curation`

**Carga · curadoria da área do produto** · tabela · 45 linhas

A área do produto dos 45 temas com 5 ou mais julgados que vieram sem tag, cada um com a base da decisão. Depende do nome do tema, por isso sobrevive à recarga.

- **Grão:** um tema curado à mão
- **Fonte:** curadoria (com base no dado)

| Coluna | Tipo | Nulo? | O que guarda | Preenchimento |
|---|---|---|---|---:|
| `theme_name` | `text` | não | Nome do tema curado. | 100% |
| `subject_area` | `text` | não | Área do produto atribuída. | 100% |
| `curated_at` | `timestamptz` | não | Quando a curadoria foi feita. | 100% |
| `stretched` | `boolean` | não | Verdadeiro quando a área é uma aproximação, porque o produto não tem área própria (hoje, as duas *Medidas de proteção* do ECA). | 100% |
| `basis` | `text` | não | A evidência da decisão: a classe processual dominante, os assuntos que co-ocorrem ou a raiz da TPU. | 100% |

Restrições:

- chave primária: `PRIMARY KEY (theme_name)`

#### `tpu_scope`

**Carga · recorte cível da TPU** · tabela · 6.140 linhas

Quais classes e assuntos da TPU são cíveis e quais são penais. É o que garante que matéria criminal não entra (D-29), e é conferido pelos testes 25 e 26.

- **Grão:** um código da TPU
- **Fonte:** SGT/CNJ (`tpu/tpu.json`)

| Coluna | Tipo | Nulo? | O que guarda | Preenchimento |
|---|---|---|---|---:|
| `kind` | `text` | não | `classe` ou `assunto`. | 100% |
| `code` | `integer` | não | Código na TPU. | 100% |
| `branch` | `text` | não | `civel` ou `penal`. | 100% |
| `name` | `text` | sim | Nome, quando conhecido (78% preenchido: os códigos penais não trazem nome). | 78,5% |

Restrições:

- regra: `CHECK ((branch = ANY (ARRAY['civel'::text, 'penal'::text])))`
- regra: `CHECK ((kind = ANY (ARRAY['classe'::text, 'assunto'::text])))`
- chave primária: `PRIMARY KEY (kind, code)`

---

## 6 · Schemas `raw`, `staging` e `nlp` — só na carga

O caminho do dado dentro da carga é `raw` → `staging` → `dw`, com o `nlp` ao lado. **Nada disso vai para a homologação nem para a produção.**

### 6.1 · `raw` — o dado como veio

Cópia fiel da fonte, sem transformar. Permite reprocessar sem coletar de novo, e a coleta é idempotente por `(source, payload_hash)`.

#### `datajud_case`

**Bruto · processo do DataJud** · tabela · 18.020 linhas

O dado cru, sem transformação. Serve para reprocessar sem coletar de novo. A coleta é idempotente por `(source, payload_hash)`.

- **Grão:** um documento de processo, exatamente como o DataJud devolveu
- **Fonte:** DataJud

| Coluna | Tipo | Nulo? | O que guarda | Preenchimento |
|---|---|---|---|---:|
| `id` | `bigint (identidade)` | não | Identidade da linha. | 100% |
| `source` | `text` | não | Fonte (`datajud`). | 100% |
| `tribunal` | `text` | não | Tribunal do documento (`tjsp`, `tjrj`, `tjmg`). | 100% |
| `source_url` | `text` | não | Endereço do índice consultado. | 100% |
| `collected_at` | `timestamptz` | não | Quando a coleta trouxe o documento. | 100% |
| `payload_hash` | `text` | não | Hash do conteúdo; com `source`, impede duplicar o mesmo documento. | 100% |
| `payload` | `jsonb` | não | O documento JSON inteiro, incluindo `movimentos` e `assuntos`. | 100% |

Restrições:

- chave primária: `PRIMARY KEY (id)`
- única: `UNIQUE (source, payload_hash)`

Índices:

- `idx_raw_datajud_case_payload`: `gin (payload jsonb_path_ops)`
- `idx_raw_datajud_case_tribunal`: `btree (tribunal)`

#### `doctrine_article`

**Bruto · artigo de doutrina** · tabela · 53.004 linhas

Registro cru de DOAJ, SciELO e OAI-PMH, antes de normalizar.

- **Grão:** um registro de artigo, como a fonte devolveu
- **Fonte:** DOAJ, SciELO, OAI-PMH

| Coluna | Tipo | Nulo? | O que guarda | Preenchimento |
|---|---|---|---|---:|
| `id` | `bigint (identidade)` | não | Identidade da linha. | 100% |
| `source` | `text` | não | Fonte (`doaj`, `scielo`, `oai_emerj`…). | 100% |
| `source_url` | `text` | não | Endereço do registro. | 100% |
| `collected_at` | `timestamptz` | não | Quando foi coletado. | 100% |
| `payload_hash` | `text` | não | Hash do conteúdo; com `source`, impede duplicar. | 100% |
| `payload` | `jsonb` | não | O registro JSON inteiro. | 100% |

Restrições:

- chave primária: `PRIMARY KEY (id)`
- única: `UNIQUE (source, payload_hash)`

Índices:

- `idx_raw_doctrine_collected`: `btree (collected_at DESC)`
- `idx_raw_doctrine_payload`: `gin (payload jsonb_path_ops)`

### 6.2 · `staging` — área de trabalho

Dado achatado e limpo, a meio caminho do `dw`. É descartável: refeito a cada carga.

#### `case_event`

**Área de trabalho · movimentação achatada** · tabela · 1.117.175 linhas

O JSON do DataJud transformado em linhas planas antes de entrar no `dw`. **É apagada e refeita a cada carga.** Tem 1.117.175 linhas contra 1.086.623 no fato: as 30.552 a mais são **repetições do mesmo evento** (mesmo processo, mesmo código e mesmo instante) que a fonte devolve mais de uma vez, e que a chave natural colapsa na entrada no `dw`.

- **Grão:** uma movimentação de um processo, já achatada do JSON
- **Fonte:** `raw.datajud_case`

| Coluna | Tipo | Nulo? | O que guarda | Preenchimento |
|---|---|---|---|---:|
| `id` | `bigint (identidade)` | não | Identidade da linha. | 100% |
| `raw_id` | `bigint` | sim | Documento de origem em `raw.datajud_case`. | 100% |
| `tribunal` | `text` | não | Tribunal. | 100% |
| `case_number` | `text` | não | Número CNJ sem máscara. | 100% |
| `court_level` | `text` | sim | Grau de jurisdição. | 100% |
| `case_class_code` | `integer` | sim | Código da classe na TPU. | 100% |
| `case_class_name` | `text` | sim | Nome da classe. | 100% |
| `judging_body_code` | `text` | sim | Código do órgão julgador. | 100% |
| `judging_body_name` | `text` | sim | Nome do órgão julgador. | 100% |
| `filed_at` | `timestamptz` | sim | Data e hora do ajuizamento. | 100% |
| `secrecy_level` | `smallint` | sim | Nível de sigilo. | 100% |
| `subjects` | `jsonb` | sim | Lista JSON dos assuntos do processo (`codigo` e `nome`). | 100% |
| `movement_code` | `integer` | não | Código da movimentação na TPU. | 100% |
| `movement_name` | `text` | não | Nome da movimentação. | 100% |
| `occurred_at` | `timestamptz` | não | Instante do evento. | 100% |
| `source` | `text` | não | Fonte. | 100% |
| `source_url` | `text` | não | Endereço de origem. | 100% |
| `extracted_at` | `timestamptz` | não | Quando foi coletado. | 100% |

Restrições:

- chave estrangeira: `FOREIGN KEY (raw_id) REFERENCES datajud_case(id)`
- chave primária: `PRIMARY KEY (id)`

Índices:

- `idx_staging_case_event_case_number`: `btree (case_number)`
- `idx_staging_case_event_raw`: `btree (raw_id)`

#### `doctrine_article`

**Área de trabalho · artigo normalizado** · tabela · 52.994 linhas

Artigo com campos separados e limpos, antes de entrar em `dw.dim_doctrine`. Vêm 52.994 do bruto (10 a menos que os 53.004 registros); a deduplicação por DOI e por (fonte, endereço) acontece na entrada no `dw`, que fica com 52.696.

- **Grão:** um artigo, já normalizado
- **Fonte:** `raw.doctrine_article`

| Coluna | Tipo | Nulo? | O que guarda | Preenchimento |
|---|---|---|---|---:|
| `id` | `bigint (identidade)` | não | Identidade da linha. | 100% |
| `raw_id` | `bigint` | sim | Registro de origem em `raw.doctrine_article`. | 100% |
| `title` | `text` | não | Título. | 100% |
| `authors` | `text` | sim | Autores. | 99,8% |
| `journal_name` | `text` | sim | Revista ou repositório. | 100% |
| `publication_year` | `integer` | sim | Ano de publicação. | 100% |
| `doi` | `text` | sim | DOI. | 54,0% |
| `article_url` | `text` | sim | Endereço do artigo. | 99,9% |
| `subject_area` | `text` | sim | Classificação temática da fonte. | 100% |
| `language` | `text` | sim | Idioma declarado pela fonte (65 valores distintos). | 100% |
| `source` | `text` | não | Fonte. | 100% |
| `source_url` | `text` | não | Endereço do registro. | 100% |
| `extracted_at` | `timestamptz` | não | Quando foi coletado. | 100% |
| `processed_at` | `timestamptz` | não | Quando foi normalizado. | 100% |

Restrições:

- chave estrangeira: `FOREIGN KEY (raw_id) REFERENCES doctrine_article(id)`
- chave primária: `PRIMARY KEY (id)`

Índices:

- `idx_staging_doctrine_raw`: `btree (raw_id)`

### 6.3 · `nlp` — vetores

Os embeddings que servem para agrupar assuntos em temas e ligar doutrina a assunto. Existem só na carga porque a produção não tem pgvector ([D-25](../06-operacao/02-decisoes-e-riscos.md#d-25--produção-sem-pgvector-embeddings-ficam-na-carga)).

#### `subject_embedding`

**NLP · vetor do assunto** · tabela · 1.075 linhas

O embedding do nome de cada assunto (384 dimensões). Existe só na carga: produção não tem pgvector (D-25).

- **Grão:** um assunto
- **Fonte:** modelo `paraphrase-multilingual-MiniLM-L12-v2`, local

| Coluna | Tipo | Nulo? | O que guarda | Preenchimento |
|---|---|---|---|---:|
| `subject_sk` | `bigint` | não | Assunto (FK para `dw.dim_subject`). | 100% |
| `embedding` | `vector(384)` | não | Vetor de 384 dimensões. | 100% |
| `embedding_model` | `text` | não | Modelo que gerou o vetor. | 100% |
| `created_at` | `timestamptz` | não | Quando foi gerado. | 100% |

Restrições:

- chave estrangeira: `FOREIGN KEY (subject_sk) REFERENCES dim_subject(subject_sk) ON DELETE CASCADE`
- chave primária: `PRIMARY KEY (subject_sk)`

#### `doctrine_embedding`

**NLP · vetor do artigo** · tabela · 52.696 linhas

O embedding do título de cada artigo. Só na carga.

- **Grão:** um artigo
- **Fonte:** modelo `paraphrase-multilingual-MiniLM-L12-v2`, local

| Coluna | Tipo | Nulo? | O que guarda | Preenchimento |
|---|---|---|---|---:|
| `doctrine_sk` | `bigint` | não | Artigo (FK para `dw.dim_doctrine`). | 100% |
| `embedding` | `vector(384)` | não | Vetor de 384 dimensões. | 100% |
| `embedding_model` | `text` | não | Modelo que gerou o vetor. | 100% |
| `created_at` | `timestamptz` | não | Quando foi gerado. | 100% |

Restrições:

- chave estrangeira: `FOREIGN KEY (doctrine_sk) REFERENCES dim_doctrine(doctrine_sk) ON DELETE CASCADE`
- chave primária: `PRIMARY KEY (doctrine_sk)`

---

## 7 · Homologação e produção

São o mesmo modelo: o schema `dw` da seção 4, sem nenhum dos schemas de trabalho.

| | Homologação | Produção |
|---|---|---|
| Estrutura | recriada a cada publicação, numa única transação (`DROP SCHEMA dw CASCADE` seguido do restore); se falhar, o destino fica como estava | a mesma, pelo instalador |
| Dado | o do banco da carga no momento da publicação | o mesmo, depois de validado na homologação |
| Dono do schema | `ratio_loader` | usuário de carga do cliente |
| Acesso da API | `ratio_api`, só `USAGE` no schema e `SELECT` nas tabelas | equivalente, só leitura |
| Extensões | `pg_trgm` e `unaccent` (o pgvector não é necessário) | as mesmas |
| Testes ao publicar | 37 testes do `dw`, todos têm que vir vazios | os mesmos |

A produção **ainda não existe** e ficará por último: a homologação é o ensaio dela. O que a homologação aprova é exatamente o que a produção recebe.

O DDL completo do `dw` está em `scraping/sql/baseline/dw_schema.sql`, e a cadeia de migrations `001` a `028` construída do zero produz um schema idêntico ao do banco (verificado por comparação, com zero diferença).

---

## 8 · O que saiu do modelo

Tudo o que não faz sentido diante do dado que a fonte entrega, ou que nenhuma história usa, foi retirado.

| Saiu | Por quê |
|---|---|
| `dw.fact_case_decision` e `dw.bridge_decision_topic` | o grão "decisão publicada" dependia do inteiro teor, bloqueado. Estavam vazias desde a criação. Levavam `reporter_judge`, `summary` e `full_text_url`, três colunas sem fonte |
| `dw.topic_summary`, `topic_by_year`, `topic_by_court`, `topic_by_judging_body` | agregados no grão de assunto, que nenhuma rota nem história lê. A câmara por tema, que faltava, virou `theme_by_judging_body` |
| `raw.tjmg_decision` e `staging.case_decision` | sobra da tentativa de raspar decisões do TJMG, que ficou bloqueada. Sempre vazias |
| `dim_case.case_number_legacy` | numeração antiga de processo. Nunca preenchida |
| `dim_topic.is_curated` | constante (todas verdadeiras); não distinguia nada |
| `dim_topic.subject_area` | duplicava a área do tema, que se obtém pela ponte. Duas cópias divergem |
| `dim_theme.cluster_id` | identificador de uma rodada de agrupamento; muda a cada carga e não tem sentido fora dela |
| Índices de trigrama em `dim_doctrine` (58 MB) | serviam para buscar na doutrina, coisa que nenhuma história pede |
| `dw.theme_registry`, `theme_area_curation`, `tpu_scope` | não saíram: **passaram para o schema `etl`**, porque são memória da carga e não fazem parte do que é entregue |

**O que foi renomeado:** `dim_topic` → `dim_subject`, `bridge_case_topic` → `bridge_case_subject`, `bridge_theme_topic` → `bridge_theme_subject`, `bridge_topic_doctrine` → `bridge_subject_doctrine` e `topic_sk` → `subject_sk`. Resolve a colisão de vocabulário da [D-34](../06-operacao/02-decisoes-e-riscos.md#d-34--themes-na-api-theme_key-na-rota): no banco, `topic` era o assunto, e na API era o tema.

**O que entrou:** `theme_by_judging_body` (câmara por tema, para o painel e a US-17), `theme_time_to_decision` (US-30), `data_provenance` como agregado materializado (a view antiga levava 1,15 s por chamada), restrições de domínio e de obrigatoriedade que estavam só na cabeça de quem carrega.

**Uma deriva corrigida no caminho:** no banco, `dim_doctrine` era única por `(source, article_url)`, mas a migration `004` ainda dizia `(source, title, publication_year)`. Alguém alterou o banco à mão e a mudança nunca virou migration. É o tipo de coisa que o runner de migrations versionadas evita.

---

## 9 · Testes de integridade

Cada teste é uma consulta que **tem que devolver zero linhas**. São 37 testes sobre o `dw`, que rodam na carga e em todo destino depois da publicação, e 3 que só fazem sentido na carga, porque dependem do schema `etl`.

### 9.1 · No `dw` — 37 testes (carga, homologação e produção)

| Arquivo | # | O que trava |
|---|---:|---|
| `011 · camada semântica (NLP)` | 1 | Tema sem assunto de origem (regra: todo tema aponta para códigos TPU reais) |
| `011 · camada semântica (NLP)` | 2 | Assunto órfão: existe em dim_subject mas nenhum tema o cobre |
| `011 · camada semântica (NLP)` | 3 | Rótulo gerado por modelo sem marcação de origem (regra: rótulo é auditável) |
| `011 · camada semântica (NLP)` | 4 | Tema sem lastro no fato (regra: tema sem processo real não existe) |
| `011 · camada semântica (NLP)` | 5 | Ligação de doutrina sem score ou sem método (regra: associação auditável) |
| `011 · camada semântica (NLP)` | 6 | Ligação de doutrina abaixo do limiar declarado (0.55) |
| `011 · camada semântica (NLP)` | 7 | Embedding fora de vector(384), ou vetor dentro do schema dw (D-25) |
| `011 · camada semântica (NLP)` | 8 | theme_summary divergindo da contagem no fato (agregado mente?) |
| `014 · força, link e polaridade` | 1 | Nota fora do intervalo 0-100 |
| `014 · força, link e polaridade` | 2 | Componente fora do intervalo 0-1 |
| `014 · força, link e polaridade` | 3 | Score não reproduz a soma ponderada dos componentes (fórmula mente?) |
| `014 · força, link e polaridade` | 4 | Grau textual incompatível com a nota |
| `014 · força, link e polaridade` | 5 | Base de cálculo inconsistente (acolhida + rejeitada <> julgados na base) |
| `014 · força, link e polaridade` | 6 | Cobertura excede o número de tribunais do escopo |
| `014 · força, link e polaridade` | 7 | Pesos não somam 1,0 |
| `014 · força, link e polaridade` | 8 | Processo com link mas sem tipo, ou tipo sem link |
| `014 · força, link e polaridade` | 9 | Tipo de link fora do vocabulário declarado (direto/portal/NULL) |
| `014 · força, link e polaridade` | 10 | Link "direto" que não aponta para o e-SAJ do TJSP (único verificado) |
| `014 · força, link e polaridade` | 11 | Número formatado inconsistente com o número bruto |
| `014 · força, link e polaridade` | 12 | Código conferido sem polaridade declarada |
| `014 · força, link e polaridade` | 13 | Concordância calculada sobre famílias MISTURADAS (erro metodológico) |
| `014 · força, link e polaridade` | 14 | Tema com julgados mas sem rótulo de polaridade (renderizaria "favorável" solto) |
| `014 · força, link e polaridade` | 15 | claimant_type fora do vocabulário declarado |
| `014 · força, link e polaridade` | 16 | Coluna com a palavra "favor" sobrou em algum agregado |
| `021 · escopo cível` | 27 | tema marcado como área PENAL |
| `021 · escopo cível` | 28 | assunto sem código da TPU |
| `021 · escopo cível` | 29 | assunto sem área oficial da TPU |
| `021 · escopo cível` | 30 | fato cujo processo não está no recorte cível |
| `027 · busca e área do produto` | 31 | tema com 5+ julgados sem área do produto |
| `027 · busca e área do produto` | 33 | busca de referência não devolve o tema esperado |
| `027 · busca e área do produto` | 34 | busca fora de escopo devolvendo resultado |
| `027 · busca e área do produto` | 35 | configuração da metodologia sem versão ou sem piso de n |
| `027 · busca e área do produto` | 36 | bloco sem fonte ou sem data de extração na proveniência |
| `029 · modelo v2` | 37 | theme_by_judging_body não fecha com theme_by_court |
| `029 · modelo v2` | 38 | tempo até a decisão fora de ordem ou negativo |
| `029 · modelo v2` | 39 | proveniência divergindo do fato e da doutrina |
| `029 · modelo v2` | 40 | objeto que saiu do modelo ainda presente no dw |

### 9.2 · Só na carga — 3 testes

| # | O que trava |
|---:|---|
| 25 | assunto de ramo penal no dw |
| 26 | processo de classe penal no dw |
| 32 | tema sem chave pública ou com chave de outro tema |

---

## 10 · Ciclo de carga e ordem de atualização

A carga é **manual** ([D-17](../06-operacao/02-decisoes-e-riscos.md#d-17--carga-manual-não-agendada)). O caminho completo, do dado bruto à homologação:

| # | Passo | Onde |
|---:|---|---|
| 1 | Coletar DataJud, DOAJ, SciELO e OAI-PMH | `raw` |
| 2 | Achatar e carregar o modelo dimensional (`transform_load_*`) | `staging` → `dw` |
| 3 | Gerar embeddings, agrupar, curar e carregar os temas; ligar doutrina | `nlp`, `dw` |
| 4 | Aplicar a curadoria de área (`sql/etl/apply_theme_area_curation.sql`) e os links de origem (`sql/etl/apply_source_links.sql`) | `dw` |
| 5 | **Atualizar os agregados, nesta ordem** (`sql/etl/refresh_aggregates.sql`) | `dw` |
| 6 | Rodar os testes (40 na carga) | carga |
| 7 | Publicar (`scripts/publish_dw.sh`) e rodar os 37 testes no destino | homologação |

A ordem do passo 5 importa, porque cada agregado lê o anterior:

```
case_current_result
   └─► theme_summary ─► theme_strength
   └─► theme_by_year
   └─► theme_by_court
   └─► theme_by_judging_body
   └─► theme_time_to_decision
data_provenance  (lê o fato e a doutrina; independe dos demais)
```
