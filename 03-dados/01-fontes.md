# Fontes de dados

O produto é **multifonte por definição**. Nenhuma fonte isolada entrega tudo que as
telas pedem: uma traz metadado processual em massa, outra traz inteiro teor, outra traz
doutrina. A arquitetura de ETL precisa nascer assumindo isso — vários conectores
alimentando o mesmo Data Warehouse.

## Escopo territorial

> **A aplicação abrange apenas os tribunais de São Paulo, Rio de Janeiro e Minas
> Gerais.** TJSP, TJRJ e TJMG.

Isso é decisão de escopo, não limitação técnica, e simplifica muita coisa:

- em vez de 91 índices do DataJud, três;
- em vez de mapear deep link de dezenas de sistemas, três (e-SAJ, PJe, o que o TJRJ usar);
- a carga completa cabe em janela de execução razoável.

**Questão em aberto:** tribunais superiores (STJ, STF) entram? Eles não são "de um
estado", mas julgam recursos vindos dos três — e os mockups mostram STJ na lista de
tribunais e súmulas do STJ na jurisprudência qualificada. Tratar como decisão pendente,
registrada em [Decisões e riscos](../06-operacao/02-decisoes-e-riscos.md).

---

## Fonte 1 · DataJud (CNJ)

Base Nacional de Dados do Poder Judiciário, mantida pelo CNJ. É a fonte de **volume**:
metadado processual em massa.

| | |
|---|---|
| Cobertura | mais de 80 milhões de processos, 91 tribunais (usaremos 3) |
| Tecnologia | Elasticsearch exposto por proxy HTTP |
| Endpoint | `POST https://api-publica.datajud.cnj.jus.br/api_publica_{tribunal}/_search` |
| Autenticação | `Authorization: APIKey <chave>` — chave **pública**, divulgada pelo CNJ |
| Documentação | <https://datajud-wiki.cnj.jus.br/api-publica/> |
| Atualização | **não é tempo real** — de horas a dias, conforme o tribunal |
| Granularidade | metadado processual: capa do processo + movimentações |

### O que entrega

| Campo | Observação |
|---|---|
| Número do processo | 20 dígitos, padrão CNJ. Não é opaco: codifica ano, segmento, tribunal e origem |
| Classe processual | código da TPU — precisa de tradução |
| Assuntos | códigos da TPU, **vários por processo** (relação N:N) |
| Órgão julgador | vara, câmara, turma |
| Grau | 1º grau, 2º grau |
| Datas | ajuizamento/distribuição e arquivamento |
| Nível de sigilo | processos em segredo de justiça |
| **Movimentações** | lista cronológica de eventos, cada um com data e código de tipo |

### Restrições

- **Não indexa CPF/CNPJ** nos campos pesquisáveis (LGPD).
- Teto de **100 documentos por página**; paginação obrigatória.
- Um índice por tribunal — não existe endpoint agregado.
- Falha intermitente e limite de taxa: retry com backoff é obrigatório.
- **Não entrega texto de decisão.** É metadado. Ver [Limitações](04-limitacoes-da-fonte.md).

---

## Fonte 2 · PANGEA / PDPJ

<https://pangeabnp.pdpj.jus.br/>

Plataforma do CNJ dentro do ecossistema **PDPJ** (Plataforma Digital do Poder
Judiciário) — a mesma casa do DataJud, mas com proposta diferente: base nacional de
precedentes.

**Por que interessa.** É a candidata natural para o que o DataJud não dá:

- precedentes qualificados — temas repetitivos, IRDR, súmulas — que alimentam o bloco
  *Jurisprudência qualificada* dos mockups, inclusive a distinção entre efeito
  `Obrigatório`, `Vinculante de fato`, `Regional` e `Persuasivo`;
- vínculo entre precedente e processos sobrestados/afetados;
- possivelmente inteiro teor ou link para ele.

**A investigar antes de assumir qualquer coisa** — nada disso está confirmado:

- [ ] existe API pública, ou só interface web?
- [ ] exige credencial PDPJ / certificado digital?
- [ ] cobre TJSP, TJRJ e TJMG, ou só tribunais superiores?
- [ ] entrega o texto do precedente, ou só a ficha?
- [ ] qual a chave de ligação com o número CNJ do processo — dá para casar com o DataJud?
- [ ] termos de uso permitem armazenar em base própria?

Essa investigação é **tarefa de spike**, com resultado escrito aqui. Enquanto não
houver resposta, nenhum bloco de tela pode depender do PANGEA.

---

## Fonte 3 · JusBrasil — provável descarte

<https://www.jusbrasil.com.br/>

Agregador comercial de jurisprudência, com acervo amplo e busca boa.

**Avaliação honesta: provavelmente não serve.** Os motivos são de licença, não técnicos:

| Barreira | Detalhe |
|---|---|
| É produto comercial | o acervo é o ativo deles; a API pública, quando existe, é limitada e paga |
| Termos de uso | raspagem tipicamente vedada em contrato |
| Proteção anti-bot | esperada em serviço desse porte |
| Dado derivado | é agregador, não fonte primária — a origem oficial continua sendo o tribunal |

**Recomendação.** Verificar se há API oficial e sob quais termos, e **não** planejar
raspagem. Se o acesso não for licenciado, descartar e ir direto ao repositório do
tribunal — que é a fonte primária de todo modo, e cuja citação é a que o usuário
final pode conferir.

> Precedente de arquitetura: preferir sempre a **fonte oficial**. O público-alvo cita o
> que lê aqui; citar um agregador comercial é mais fraco do que citar o tribunal.

---

## Fonte 4 · Repositórios de jurisprudência dos tribunais

Como o escopo é SP/RJ/MG, são três alvos concretos, não 91:

| Tribunal | Sistema | O que investigar |
|---|---|---|
| **TJSP** | e-SAJ | consulta de jurisprudência tem endpoint estável? formato do inteiro teor? |
| **TJRJ** | portal próprio + SPA | a busca de jurisprudência é separada da consulta processual |
| **TJMG** | PJe / portal | mesma pergunta |

É a **única via conhecida** para o texto integral do acórdão, e portanto para a citação
de acórdão, os fundamentos invocados e a coluna de valor da condenação.

Dificuldades esperadas: sem API padronizada, formato variando por tribunal, HTML e PDF,
e proteção contra cliente que não é navegador.

---

## Fonte 5 · Doutrina

**Regra fechada do produto:**

> A doutrina **nunca** é servida como PDF de livro. A referência é:
> - **o artigo**, por **link** para onde ele está publicado; ou
> - **o nome da obra**, quando a citação vier de livro — autor, título, edição,
>   capítulo. Texto, não arquivo.

Isso resolve o problema jurídico (livro é obra protegida; hospedar PDF é infração) e
mantém a utilidade: o profissional precisa saber **o que citar**, não receber o livro.

Fontes candidatas para o artigo com link:

| Tipo | Exemplos | Nota |
|---|---|---|
| Repositórios acadêmicos abertos | SciELO, portais de periódicos de faculdades de direito, repositórios institucionais | link estável, acesso aberto |
| Revistas jurídicas com DOI | — | DOI é a referência ideal: estável e citável |
| Publicações de tribunais e escolas judiciais | revistas do STJ, EMERJ, EPM | conteúdo aberto |

Para livro: **apenas os metadados da referência**, curados. Autor, obra, edição,
capítulo — exatamente o que o mockup mostra em *Doutrina invocada*.

**Ainda em aberto:** de onde sai a associação entre doutrina e tema. Extrair citações
doutrinárias do texto do acórdão exige o inteiro teor (Fonte 4) e NLP. Curar
manualmente é possível para uma demonstração e não escala.

---

## Fontes ainda não listadas

O time indicou que haverá outras. Quando entrarem, cada uma precisa responder a este
questionário **antes** de virar conector:

1. Qual pergunta de tela ela responde que nenhuma outra responde?
2. Tem API, ou é raspagem? Se raspagem, os termos de uso permitem?
3. Como se liga ao que já temos — pelo número CNJ, ou por outra chave?
4. Qual a frequência de atualização e o custo de uma carga completa?
5. Se ela cair amanhã, que parte do produto para de funcionar?

A resposta vira uma seção nesta página.

---

## TPU — Tabela Processual Unificada

Não é fonte de dados, é **tabela de tradução**: os códigos padronizados do CNJ para
classe, assunto e movimentação. Tudo que vem do DataJud está codificado por ela.

Para os movimentos, a tradução vai além do nome: cada código precisa virar uma
categoria analítica (resultado favorável, desfavorável, parcial…). É o ponto mais
sensível do pipeline e exige verificação código a código contra a API.
Ver [ETL e NLP](../02-arquitetura/05-etl-e-nlp.md).

---

## Consolidação: o que sustenta cada bloco de tela

| Bloco da tela | Fonte esperada | Estado |
|---|---|---|
| Busca de temas | DataJud (assuntos TPU) + agrupamento | 🟡 fonte existe, agrupamento a definir |
| Nota de força | DW (agregação própria) | 🟡 metodologia proposta, não auditada |
| Comportamento por tribunal | DataJud + TPU | 🟡 fonte existe |
| Amostra auditável (processo, órgão, data, desfecho) | DataJud + TPU | 🟡 fonte existe |
| Link para o processo na origem | mapeamento por tribunal (3 sistemas) | 🟡 só TJSP verificado |
| Série anual / por órgão | DW | 🟡 fonte existe |
| Jurisprudência qualificada | **PANGEA** (a investigar) | 🔴 não confirmada |
| Citação de acórdão · inteiro teor | repositório do tribunal | 🔴 não integrado |
| Fundamentos invocados | inteiro teor + NLP | 🔴 depende do inteiro teor |
| Doutrina invocada | artigo com link / referência de livro curada | 🔴 sem fonte definida |
| Coluna `relator` | inteiro teor ou raspagem | 🔴 sem fonte |
| Coluna `valor` / mediana R$ | inteiro teor + NLP | 🔴 sem fonte |

Detalhamento do que falta e das opções: [Limitações da fonte](04-limitacoes-da-fonte.md).
