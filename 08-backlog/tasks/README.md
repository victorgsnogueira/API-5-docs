# Tasks do projeto

Quebra do [Product Backlog](../product-backlog.md) em tasks, separada por sprint, no padrão que o time já usa: uma **Story** por User Story, **sub-issues `X.Y`** por trabalho técnico e um bloco **`Technical Foundation`** (`0.Y`) para o que não tem Story-mãe. **Este conjunto substitui o `tasks.md` anterior**, que ainda descrevia as 12 Stories da versão antiga da Sprint 1.

| Arquivo | Conteúdo |
|---|---|
| [`sprint-1.md`](sprint-1.md) | Sprint 1 · 07/09 a 27/09 · 9 Stories · 44 SP · 107 tasks |
| [`sprint-2.md`](sprint-2.md) | Sprint 2 · 05/10 a 25/10 · 13 Stories · 65 SP · 88 tasks |
| [`sprint-3.md`](sprint-3.md) | Sprint 3 · 02/11 a 22/11 · 7 Stories · 40 SP · 57 tasks |

## Como ler

- **Título da task:** curto, imperativo, em inglês (o código é em inglês — [Idioma](../../02-arquitetura/02-backend-dotnet.md#idioma)). Sem tag de camada no texto: camada é o campo `Layer`.
- **Descrição:** só quando carrega informação que o título não dá, em tópicos: `Data:` (o que o dado ou o contrato tem), `Message:` (texto exato mostrado ao usuário), `Verifies:` (o que o teste prova) e `Depends:` (outras tasks).
- **Estimativa:** em horas, sempre **menos de 8h**. Nenhuma task passa de 6h. São propostas para refinar com o time, não compromissos.
- **`Layer`:** `Backend` · `Frontend` · `ETL` · `Test`, as quatro do padrão do time, mais **`DevOps`** (CI, empacotamento, implantação) e **`Docs`** (manuais e páginas de metodologia), acrescentadas porque há trabalho que não cabe nas quatro. `ETL` é o pipeline de dados; o schema do banco é `Backend`, porque nasce no repositório do backend.
- **Numeração:** `X.Y` pertence à Story `US-X`; `0.Y` é Technical Foundation, sem Story-mãe. Os IDs são únicos no projeto inteiro.
- **🔒:** Story sem fonte de dado hoje. A primeira task é um spike, a segunda registra a decisão, e as de implementação só são puxáveis depois dela.
- **Link da Story:** leva aos critérios de aceite e à DoR dela, no repositório do projeto.

## Resumo

| Sprint | Stories | SP | Tasks | Horas | Foundation | Stories (tasks) |
|---|---:|---:|---:|---:|---:|---:|
| Sprint 1 | 9 | 44 | 107 | 396h | 41 (163h) | 66 (233h) |
| Sprint 2 | 13 | 65 | 88 | 303h | 6 (25h) | 82 (278h) |
| Sprint 3 | 7 | 40 | 57 | 212h | 14 (56h) | 43 (156h) |
| **Total** | **29** | **149** | **252** | **911h** | | |

Horas por camada:

| Camada | Sprint 1 | Sprint 2 | Sprint 3 | Total |
|---|---:|---:|---:|---:|
| Backend | 145h | 107h | 100h | 352h |
| Frontend | 99h | 108h | 30h | 237h |
| ETL | 124h | 41h | 2h | 167h |
| Test | 21h | 34h | 38h | 93h |
| DevOps | 7h | 7h | 26h | 40h |
| Docs | 0h | 6h | 16h | 22h |

> **Compare esses totais com a capacidade real do time antes de puxar a Sprint 1.** A Sprint 1 concentra a base do projeto (schema, pipeline, carga e esqueleto de tela) e é a que mais pesa em horas; a soma das horas de uma sprint não é a soma dos story points dela, porque parte do trabalho está no bloco Technical Foundation e não pertence a nenhuma Story.

## O que está por trás das tasks

As tasks seguem a arquitetura decidida na [D-35](../../06-operacao/02-decisoes-e-riscos.md#d-35--o-banco-do-cliente-é-o-dw-um-só-modelo-dw-nos-três-bancos) e descrita em [Modelagem dos três bancos](../../03-dados/06-modelagem-dos-bancos.md):

| Peça | Onde vive | O que as tasks fazem com ela |
|---|---|---|
| **Schema do `dw`** | migrations SQL no repositório do backend | criam o banco na subida da API, no cliente e na homologação; o schema nasce em `V001` e cresce migration a migration |
| **Pipeline** | repositório próprio e versionado | monta os dados no banco de carga, sprint a sprint, até chegar ao resultado completo |
| **Dados** | um arquivo SQL gerado na carga | vocês carregam na homologação e o cliente carrega na produção, na mão, com `psql -f` |

## Como o pipeline cresce

O pipeline não chega pronto: cada sprint acrescenta só o que as Stories dela precisam, e a última fecha a prova de que o conjunto reproduz uma carga completa.

| Sprint | O que o pipeline passa a entregar | Tasks |
|---|---|---|
| **1** | repositório e CI; banco de carga a partir das migrations; escopo cível pela TPU; coleta do DataJud por tribunal, estratificada por área e só com processos julgados; carga idempotente das dimensões e do fato; polaridade e links; embeddings locais, clusters, curadoria e temas com chave estável; doutrina (DOAJ, SciELO, OAI-PMH) e ligação por similaridade; texto do tema; arquivo de carga com os agregados atualizados | 0.21 a 0.44, 1.9, 9.4, 9.5, 21.1 a 21.5 |
| **2** | segunda rodada de coleta para os temas com poucos julgados; versão do schema e manifesto no arquivo de carga; sugestões de busca; fundamentos do inteiro teor, se a decisão liberar | 0.47 a 0.49, 0.52, 38.1, 19.1 a 19.5 |
| **3** | congelamento do arquivo de carga para a entrega; reconstrução do zero comparada com a carga anterior | 0.67, 0.68 |

## Decisões que travam tasks

A numeração é a do backlog. Cada uma tem uma task cujo entregável é escrever a decisão em [Decisões e riscos](../../06-operacao/02-decisoes-e-riscos.md).

| Decisão | O que decide | Task | Precisa estar tomada |
|---|---|---|---|
| 20 | fonte do texto do tema: curado, e o dado registra a origem | 9.1 | Sprint 1 |
| 3 | tratamento da procedência em parte na figura | 10.1 | Sprint 1 |
| 2 | STJ e STF entram no escopo? | 4.1 | planejamento da Sprint 2 (05/10) |
| 4 | recalibração da cobertura do score | 6.1 | Sprint 2 |
| 14 | mínimo de julgados para exibir o grau textual | 7.1 | Sprint 2 |
| — | regra que marca uma câmara como divergente | 17.2 | Sprint 2 |
| — | inteiro teor: fonte, viabilidade ou remoção | 19.1, 19.2, 37.1 | Sprint 2 |
| 16 | limiar para dar o dado como velho | 29.1 | Sprint 3 |
| 18 | onde o chatbot aparece | 34.1 | Sprint 3 |
| — | modelo e hospedagem do chatbot, diante do NFR-12 | 34.2, 34.3 | Sprint 3 |

## De onde vieram os IDs de fundação anteriores

| ID anterior | Situação |
|---|---|
| `0.4` índice único em todo agregado | absorvido: cada task que cria um agregado já exige o índice único (0.69, 0.70, 2.1, 10.2, 11.1, 12.1, 17.1, 30.1) |
| `0.5` refresh dos agregados no fim da carga | absorvido na 0.25, o arquivo de carga |
| `0.6` papel somente leitura | mantido |
| `0.7` renomear as rotas | mantido; a rota passou a receber `$key`, não `$code` |
| `0.8`, `0.9` TanStack Query, cliente e schemas | mantidos |
| `0.10`, `0.11` MSW e Vitest no CI | já entregues |

As tasks das Stories foram renumeradas, e o que mudou de história saiu do conjunto: US-05, US-18, US-22 e US-23 não existem mais no backlog.
