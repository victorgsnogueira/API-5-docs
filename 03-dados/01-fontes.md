# Fontes de dados

> ## ✅ Investigadas e testadas — 15/09/2026
>
> As fontes abaixo deixaram de ser candidatas: foram **testadas contra os sistemas
> reais**, e o resultado está registrado em cada seção. Resumo:
>
> | Fonte | Veredito | Por quê |
> |---|---|---|
> | DataJud | ✅ **em uso** | 1.086.623 movimentações dos três tribunais, só matéria cível |
> | Doutrina (DOAJ, SciELO, OAI-PMH) | ✅ **em uso** | 52.696 artigos — a lacuna de doutrina foi resolvida |
> | Repositório do TJSP (e-SAJ/CJSG) | ❌ inviável | exige reCAPTCHA v3 |
> | Repositório do TJRJ (eJURIS) | ❌ inviável | Termos de Uso proíbem raspagem |
> | Repositório do TJMG | ❌ inviável | CAPTCHA numérico na busca |
> | STJ (`scon.stj.jus.br`) | ❌ inviável | bloqueio de borda (403) |
> | PANGEA | ⬜ não investigada | |
>
> Detalhe de cada teste nas seções correspondentes. A carga que produziu esses
> números está em `scraping/` (spike), documentada em `scraping/README.md`.

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

### ✅ Verificado em uso (15/09/2026)

Confirmado contra a API real, com 18.378 processos coletados:

| Fato | Detalhe |
|---|---|
| Chave pública funciona | `Authorization: APIKey …`, divulgada no wiki do CNJ |
| Paginação | `from/size` **não alcança além de ~10.000**. Obrigatório `search_after` com `sort` — o teto de 100 por página é só o tamanho do lote |
| `sort` por `_id` | **não funciona** (400). Usar `@timestamp` |
| Latência | 15 a 30 s por página é normal; picos de mais de 60 s |
| Falhas | 504 e 429 frequentes — o retry com backoff não é precaução, é requisito |
| **Movimentos por processo** | média de **43,8** — um recorte de 18 mil processos rende mais de 460 mil linhas de fato |
| Campo `relator` | **não existe no payload**. Confirma a limitação já prevista |

### ⚠ Qualidade do dado varia por tribunal — achado grave

As **265.088 movimentações do TJMG** coletadas têm **100% do campo `dataHora`
nulo**. Não é bug do coletor: é o dado que o tribunal publica nessa API.

Como o fato de movimentação exige timestamp real (e
[D-11](../06-operacao/02-decisoes-e-riscos.md#d-11--nada-de-dado-inventado)
proíbe inventar data), **o TJMG ficou fora da tabela fato**. TJSP e TJRJ não
têm o problema.

**Consequência de produto:** o escopo declarado é de três tribunais, mas a base
efetiva hoje tem **dois**. Isso afeta diretamente o componente de cobertura da
[nota de força](../01-produto/04-forca-do-entendimento.md) e precisa ser
declarado na interface. Qualquer conector novo deve checar a completude campo a
campo por tribunal, não assumir que o contrato da API vale igual para todos.

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

> ### 🔴 Investigada e **bloqueada nos quatro alvos** — 15/09/2026
>
> Esta era a única via conhecida para o inteiro teor. **Nenhum dos quatro
> tribunais testados é acessível** por raspagem lícita e sem contornar proteção
> anti-bot.

Como o escopo é SP/RJ/MG, eram três alvos concretos (mais o STJ, testado como
alternativa). Resultado de cada um, **testado contra o sistema real**:

| Tribunal | Sistema | O que o teste mostrou | Veredito |
|---|---|---|---|
| **TJSP** | e-SAJ / CJSG | O formulário carrega **reCAPTCHA v3** (`grecaptcha.execute`) e tem um captcha secundário próprio (`captchaControleAcesso.do`). O campo `recaptcha_response_token` é obrigatório inclusive na consulta simples | ❌ |
| **TJRJ** | eJURIS (ASP.NET) | Tecnicamente o mais fácil: HTML server-side, sem captcha aparente. Mas o [Termo de Uso](https://www.tjrj.jus.br/lgpd/termo-de-uso) **proíbe expressamente** "robôs, spiders ou scrapers… sem permissão expressa por escrito deste Tribunal" | ❌ |
| **TJMG** | Struts (`www5`) | Sem captcha na página do formulário — mas a **busca real** devolve **401 + desafio de CAPTCHA numérico** (`captcha.svl`). A decisão é server-side (`ValidacaoCaptchaAction.exibirCaptcha`) | ❌ |
| **STJ** | `scon.stj.jus.br` | **403** de borda/WAF, mesmo com headers de navegador | ❌ |

### Por que não basta "tentar mais"

O TJMG foi testado a fundo porque o captcha ali é **condicional** (existe uma
função que decide se exibe). Testadas e descartadas: sessão nova por requisição,
pausa de 5 minutos, busca estruturada sem palavra-chave, endpoint de número
único. Todas continuaram caindo no captcha.

A conclusão veio do ecossistema: os raspadores open-source que **de fato** leem o
TJMG (ex.: [`jtrecenti/juscraper`](https://github.com/jtrecenti/juscraper))
declaram no código que resolvem o captcha automaticamente por OCR. Ou seja: **não
existe caminho que evite o captcha** — existe caminho que o resolve.

### O que isso significa para o produto

Os blocos que dependiam de inteiro teor continuam **sem fonte**: citação de
acórdão, fundamentos invocados, valor da condenação e relator.

Caminhos reais, em ordem de custo:

1. **Autorização escrita do TJRJ** — é a única barreira puramente contratual;
   um ofício resolve, e o eJURIS é tecnicamente simples.
2. **Convênio/API institucional** com os tribunais, via FATEC.
3. **Resolução automatizada de captcha** (OCR ou serviço) — prática comum na
   comunidade de legal tech brasileira, mas é **decisão do time e da
   coordenação**, com implicações jurídicas que não cabe a um desenvolvedor
   tomar sozinho.
4. **Remover os blocos do escopo** e entregar o que o DataJud sustenta.

---

## Fonte 5 · Doutrina

**Regra fechada do produto:**

> A doutrina **nunca** é servida como PDF de livro. A referência é:
> - **o artigo**, por **link** para onde ele está publicado; ou
> - **o nome da obra**, quando a citação vier de livro — autor, título, edição,
>   capítulo. Texto, não arquivo.

Isso resolve o problema jurídico (livro é obra protegida; hospedar PDF é infração) e
mantém a utilidade: o profissional precisa saber **o que citar**, não receber o livro.

### ✅ Resolvida — 52.696 artigos carregados (15/09/2026)

Três fontes abertas, todas com API ou protocolo padronizado, **sem captcha,
sem bloqueio de robots.txt e sem restrição de Termos de Uso encontrada**:

| Fonte | Acesso | Artigos | Observação |
|---|---|---:|---|
| **DOAJ** | API REST pública, sem autenticação | 44.132 | Teto de **1.000 resultados por consulta** — contornado particionando por termo jurídico × ano de publicação. O endpoint OAI-PMH deles responde mas devolve vazio |
| **OAI-PMH** — rede [Index Law](https://www.indexlaw.org) (46 revistas), Revista da EMERJ (TJRJ), Revista EJEF (TJMG) | OAI-PMH 2.0, Dublin Core | 6.754 | Sem teto: pagina por `resumptionToken` até esgotar. Melhor fonte para volume previsível |
| **SciELO** | ArticleMeta API | 2.118 | 6 revistas de Direito na coleção Brasil, achadas varrendo as 427 revistas indexadas |

Descartada: **Repositório da USP** — o `robots.txt` bloqueia explicitamente
agentes de IA. Respeitado.

Para livro: **apenas os metadados da referência**, curados. Autor, obra, edição,
capítulo — exatamente o que o mockup mostra em *Doutrina invocada*.

### ✅ A associação doutrina ↔ tema também foi resolvida

Era a pergunta em aberto desta página. A resposta **não** depende do inteiro
teor: é associação semântica entre o título do artigo e o assunto da TPU,
com score gravado e limiar declarado. **13.870 ligações**, cobrindo 9.649 artigos.

**Só embedding não serve** — e isso é a lição que vale registrar. Na primeira
tentativa, o tema *Indenização por Dano Moral* atraiu artigos de **filosofia
moral** com score mais alto que os do instituto jurídico:

| Artigo | Similaridade |
|---|---:|
| A CALIBRAÇÃO DA MORAL PELO POSITIVISMO | 0,903 |
| MORAL E CONTEMPORANEIDADE | 0,901 |
| *Dano Moral: Aspectos Históricos e de Quantificação* (o correto) | *0,854* |

Falso-amigo semântico: **"moral" (filosofia) × "dano moral" (instituto)**. Como
o erro pontuava **mais alto** que o acerto, subir o limiar não resolveria.

A solução é **híbrida**: exige proximidade semântica **e** presença léxica dos
termos distintivos do assunto no título. Isso rejeitou **176.709 candidatos
falsos** e derrubou a cobertura de 67,6% para 13,7% — que é a direção certa de
errar, pela mesma lógica do D-10: artigo sem tema é recuperável, artigo no tema
errado não é.

Detalhes em [ETL e NLP](../02-arquitetura/05-etl-e-nlp.md#nlp--onde-o-modelo-entra).

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

Atualizado após a carga real de 15/09/2026.

| Bloco da tela | Fonte | Estado |
|---|---|---|
| Busca de temas | DataJud (assuntos TPU) + agrupamento semântico | 🟢 **1.049 temas carregados**, com lastro em assunto real |
| Nota de força | DW (agregação própria) | 🟢 **implementada**, com os 4 componentes abertos. R-08 (cobertura) decidido: satura em 3 |
| Comportamento por tribunal | DataJud + TPU | 🟢 **implementado** — mas só 2 tribunais têm dado (ver TJMG acima) |
| Amostra auditável (processo, órgão, data, desfecho) | DataJud + TPU | 🟢 **18.002 processos** |
| Link para o processo na origem | mapeamento por tribunal | 🟢 **100% dos processos**: TJSP `direto` (reverificado), TJRJ `portal` |
| Série anual / por órgão | DW | 🟢 **implementado** |
| Doutrina invocada | DOAJ + SciELO + OAI-PMH | 🟢 **52.696 artigos**, 13.870 ligados a tema |
| **Polaridade do resultado** | DataJud + classe processual | 🟢 **implementada** — ver [Polaridade](05-polaridade-do-resultado.md). **Nova exigência**, descoberta na carga |
| Jurisprudência qualificada | **PANGEA** (não investigada) | 🔴 sem fonte |
| Citação de acórdão · inteiro teor | repositório do tribunal | 🔴 **bloqueado nos 4 tribunais** (captcha/ToS) |
| Fundamentos invocados | inteiro teor + NLP | 🔴 depende do inteiro teor |
| Coluna `relator` | inteiro teor ou raspagem | 🔴 **confirmado ausente** no payload do DataJud |
| Coluna `valor` / mediana R$ | inteiro teor + NLP | 🔴 sem fonte |

O que mudou de 🔴 para 🟢 veio de fonte nova (doutrina) ou de trabalho de ETL
que faltava. O que continua 🔴 depende de **inteiro teor**, que é a única
lacuna estrutural que sobrou.

Detalhamento do que falta e das opções: [Limitações da fonte](04-limitacoes-da-fonte.md).
