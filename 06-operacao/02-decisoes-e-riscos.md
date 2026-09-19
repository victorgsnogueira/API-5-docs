# Decisões e riscos

## Registro de decisões

Formato leve: o que foi decidido, por quê, e o que isso custa.

---

### D-01 · Data Warehouse em Postgres

**Decisão.** Postgres com modelagem dimensional, em vez de stack colunar (Athena/Parquet,
Databricks, DuckDB).

**Por quê.** Com escopo em três tribunais, o recorte é de centenas de milhares a poucos
milhões de linhas de fato — Postgres resolve com folga. Modelagem dimensional é agnóstica
de tecnologia. E o Postgres ainda resolve a busca full-text em português, eliminando um
Elasticsearch da infraestrutura.

**Custo.** Deixaria de servir acima de dezenas de milhões de linhas com agregação
interativa sobre todas elas. Não está no escopo.

Detalhes: [Data Warehouse](../02-arquitetura/04-data-warehouse.md).

---

### D-02 · Backend em .NET 8

**Decisão.** ASP.NET Core 8, Clean Architecture em sete projetos.

**Por quê.** Alternativa moderna ao Java; a estrutura de solução torna a separação em
camadas uma restrição do compilador, não uma convenção que se erode.

**Custo.** Começar do zero: o protótipo antigo está em Python e
[não serve de base](../05-prototipo/01-prototipo-referencia.md). E o .NET 8 sai de
suporte antes do fim do projeto — ver [R-14](#r-14--net-8-sai-de-suporte-durante-o-projeto-).

---

### D-03 · Escopo territorial: TJSP, TJRJ e TJMG

**Decisão.** A aplicação abrange apenas os tribunais de São Paulo, Rio de Janeiro e
Minas Gerais.

**Por quê.** Três estados concentram volume suficiente para o produto fazer sentido, e o
recorte torna viável mapear deep link, investigar repositório de jurisprudência e rodar
carga completa em janela razoável.

**Custo.** A interface **precisa declarar o escopo** — quem lê um percentual assumindo
cobertura nacional tira conclusão errada. Pior na prática: a base efetiva hoje tem
**dois** tribunais, não três (R-12). Também obriga a
[recalibrar o componente de cobertura](../01-produto/04-forca-do-entendimento.md#cobertura)
da nota de força.

**Em aberto.** STJ e STF entram? Não são "de um estado", mas julgam recursos dos três, e
os mockups os exibem.

---

### D-04 · Produto multifonte

**Decisão.** Consumir várias fontes — DataJud, PANGEA, repositórios dos tribunais,
doutrina — não apenas uma.

**Por quê.** Nenhuma fonte isolada entrega o que as telas pedem.

**Custo.** ETL com um conector por fonte atrás de uma mesma porta; reconciliação entre
fontes; e **proveniência obrigatória em toda linha** — sem ela não há como auditar
divergência nem reprocessar uma fonte só.

Ver [Fontes](../03-dados/01-fontes.md).

---

### D-05 · Doutrina nunca é PDF de livro

**Decisão.** A referência doutrinária é o **artigo, por link** para onde está publicado,
ou o **nome da obra** em texto (autor, título, edição, capítulo).

**Por quê.** Livro é obra protegida; hospedar PDF é infração. E o profissional precisa
saber **o que citar**, não receber o arquivo.

**Custo.** Nenhum relevante. Ver [Fontes · Doutrina](../03-dados/01-fontes.md#fonte-5--doutrina).

---

### D-06 · Código em inglês, retorno da API em português

**Decisão.** Todo o código — backend, frontend e pipeline: classes, métodos, tabelas,
colunas, rotas, chaves do JSON, logs, testes, commits — em inglês. **Tudo o que a API
devolve para ser lido é em português**: valores de dado, rótulos, motivos de dado
ausente, mensagens de erro e de validação. *(Revisada em 19/09/2026: antes falava só em
"dados em português"; agora cobre explicitamente erro e rótulo.)*

**Regra de bolso.** Chave é código (inglês); valor é retorno (português). Detalhe em
[Idioma](../02-arquitetura/02-backend-dotnet.md#idioma).

**Custo.** Traduzir jargão jurídico gera divergência no time. Mitigado por um
[vocabulário PT→EN fechado](../02-arquitetura/02-backend-dotnet.md#idioma) — acrescente
linhas a ele, não invente sinônimos.

---

### D-07 · Hospedagem em VPS Hostinger com Coolify

**Decisão.** VPS na Hostinger, deploy automático via Coolify. CI/CD, documentação e
monitoramento obrigatórios; ferramentas a definir.

**Implicações.** Tudo precisa ser containerizável; health check vira contrato;
configuração por variável de ambiente; e backup do banco é responsabilidade nossa. (A
carga não roda na VPS — [D-17](#d-17--carga-manual-não-agendada).)

Ver [DevOps e infraestrutura](03-devops-e-infra.md).

---

### D-08 · O protótipo não é fonte de verdade

**Decisão.** A pasta `prototipo/` não serve como base para nada — nem contrato, nem
modelagem, nem código. Em particular, **a modelagem de banco dele foi criada sem
auditoria alguma**.

**Por quê.** Foi construída antes das definições atuais de produto, escopo, fontes e
arquitetura.

**Custo.** Perde-se trabalho já feito. O ganho é não herdar decisão que ninguém revisou.
O que sobra dele é o catálogo de armadilhas das fontes —
[ver](../05-prototipo/01-prototipo-referencia.md).

---

### D-09 · Agregados pré-calculados

**Decisão.** Agregados atualizados ao fim do ETL, em vez de agregação a cada request.

**Por quê.** Várias agregações sobre a tabela fato por abertura de tela, e o dado só muda
uma vez por dia.

**Custo.** O dado é tão fresco quanto a última carga. A interface declara a data de
extração.

---

### D-10 · Código de movimentação não conferido não entra na métrica

**Decisão.** Só entra no mapa da TPU código cujo nome foi verificado contra a API. Na
dúvida, categoria neutra — e categoria neutra não conta.

**Por quê.** Um código mal classificado corromperia toda a apuração de resultado do
produto **sem sintoma visível**.

**Custo.** Subcontagem em vez de erro. É a direção certa de errar.

**✅ Aplicado:** 6 códigos conferidos contra a TPU/CNJ; **257 entraram como
neutros**. A regra rendeu mais exclusão do que inclusão, como esperado.

---

### D-11 · Nada de dado inventado

**Decisão.** Onde a fonte não tem, a API devolve vazio e a tela diz "sem dado". O modelo
pode rotular, agrupar e redigir — **nunca contar**. Todo número vem de `SELECT`.

**Por quê.** O público-alvo cita o que lê aqui em peça e em decisão. Um número alucinado
não é um erro recuperável.

**Custo.** Telas com blocos vazios em vez de telas bonitas.

Vale igualmente para o [chatbot](../01-produto/05-chatbot.md).

---

### D-12 · Chatbot no roadmap, sobre os mesmos agregados

**Decisão.** Haverá um chatbot que usa LLM para consultar e responder a partir do DW.
Ele consome as **mesmas consultas** que as telas, expostas como ferramentas — não um
caminho alternativo até o banco.

**Por quê.** É o que garante que ele responda os mesmos números que a tela mostra, e que
cada número seja auditável.

**Custo.** Só faz sentido depois que os agregados estiverem estáveis. Ver
[Chatbot](../01-produto/05-chatbot.md).

---

### D-13 · Grão do fato: movimentação processual (Opção A)

**Decisão.** Uma linha de `fact_case_event` = uma movimentação. O resultado
vigente por processo sai de um agregado por cima (`case_current_result`).

**Por quê.** O DataJud entrega o array `movimentos`, que é a fonte de eventos que
a Opção A pressupunha. Não fecha porta: permite tempo entre etapas e taxa de
recurso, que a Opção B perderia.

**Custo.** Volume alto — média de **43,8 movimentos por processo**; 18.378
processos renderam 463.016 linhas. Postgres absorve sem esforço.

**Nota.** `fact_case_decision` (grão = decisão publicada) existe e está vazia,
para quando houver inteiro teor. São grãos diferentes para fontes diferentes,
não versões concorrentes.

---

### D-14 · Cobertura da nota de força satura em 3 tribunais

**Decisão.** `N = 3` — todos os tribunais do escopo declarado. Resolve o R-08,
que era bloqueante.

**Por quê.** Cobertura mede "a tese aparece em quantos tribunais do universo
coberto", e o universo declarado do produto é três.

**Custo.** `N` precisa mudar se STF/STJ entrarem. Por isso vive em
`dw.strength_config`, não cravado na fórmula.

---

### D-15 · A palavra "favorável" não existe no schema

**Decisão.** Contagens de resultado se chamam **pretensão acolhida/rejeitada**
(`claim_upheld` / `claim_rejected`), sempre acompanhadas de **quem é o autor**.
Mérito e recurso nunca são somados.

**Por quê.** Descoberto na primeira carga real: em matéria penal — a **maior
área da base** — "procedência" é condenação. "98% favorável" faria um advogado
ler exatamente o contrário do que o dado diz.

**Custo.** Contrato de API mais verboso: todo percentual carrega um rótulo de
polaridade junto. É o preço de não induzir o usuário a erro.

Ver [Polaridade do resultado](../03-dados/05-polaridade-do-resultado.md).

---

### D-16 · Doutrina ligada a tema por método híbrido, não só embedding

**Decisão.** A associação doutrina↔assunto exige proximidade semântica **e**
presença léxica dos termos distintivos. Limiar e score gravados em cada linha.

**Por quê.** Embedding puro ligou artigos de *filosofia moral* ao tema *dano
moral* com score **maior** que os artigos corretos. Subir o limiar não
resolveria — o erro pontuava mais alto que o acerto.

**Custo.** Cobertura caiu de 67,6% para 13,7%. É a direção certa de errar:
artigo sem tema é recuperável, artigo no tema errado não é (mesma lógica do D-10).

---

### D-17 · Carga manual, não agendada

**Decisão.** Não existe carga diária nem job agendado. A raspagem é feita **à mão**,
quando o time decide atualizar a base: coletar → normalizar (NLP + curadoria) →
agregar → validar (24 testes) → subir para produção por `pg_dump`/`pg_restore` do
schema `dw`. Exatamente o processo que produziu a base atual. *(19/09/2026)*

**Por quê.** O pipeline tem um passo humano que não automatiza — a
[curadoria de temas](../02-arquitetura/05-etl-e-nlp.md#por-que-o-passo-4-é-humano)
rejeitou 37 de 69 clusters. As fontes têm limite de taxa e a coleta leva horas. E o
produto mostra **tendência de julgamento**, não notícia: dado de semanas atrás não muda
a leitura.

**Consequências.**
- o pipeline fica em **Python** (onde está o NLP); o `Ratio.Etl` em .NET perde o papel —
  recomendação: remover da solução;
- a API passa a ser **somente leitura** também no banco (papel `ratio_api` só com
  `SELECT`);
- some o contêiner de ETL e o alarme de "carga não rodou";
- a **data da extração** vira a única defesa contra dado velho — declarada na tela e em
  `/health/ready`.

**Custo.** A base só se atualiza quando alguém roda. E o pipeline mora hoje numa pasta
fora de repositório — [R-15](#r-15--o-pipeline-de-carga-não-está-versionado-).

---

### D-18 · TDD no backend e no frontend

**Decisão.** Todo código de produção, nos dois repositórios de aplicação, nasce de um
teste que falhou antes. Backend: xUnit (com o `Assert` nativo) + Moq + Testcontainers
(Postgres real). Frontend: Vitest + Testing Library + MSW. *(19/09/2026)*

**Por quê.** Os erros mais caros encontrados até aqui — "favorável" invertendo temas
penais, filosofia moral ligada a *dano moral*, agregado duplicando por produto
cartesiano — **não quebravam nada visível**. Só teste que afirma o comportamento pega
esse tipo de erro.

**Custo.** Ritmo inicial mais lento, e o CI precisa de Docker para os testes de
integração. Ver [TDD](../07-justificativas/03-tdd.md).

---

### D-19 · Frontend: TanStack Router + shadcn/ui + Tailwind

**Decisão.** React 19 + Vite + TypeScript, **TanStack Router** (rotas por arquivo),
**shadcn/ui** sobre Base UI, **Tailwind CSS v4**, em monorepo Turborepo. Substitui a
recomendação anterior desta wiki (React Router + CSS puro). *(19/09/2026 — registrando
o que o scaffold do `API5-Frontend` já adotou.)*

**Por quê.** O TanStack Router tipa rota e search param (a aba do detalhe mora na URL).
O shadcn copia o código do componente para o repo — dá para dobrá-lo ao design system
em vez de brigar com uma biblioteca fechada. E o Tailwind v4 lê os tokens do design
system direto de CSS custom properties.

**Custo.** O default do shadcn traz sombra, raio e modo escuro que o design system
proíbe — **todo componente adicionado precisa ser revisado** antes de commitar. Ver
[Frontend React](../02-arquitetura/03-frontend-react.md).

---

### D-20 · Imagem do banco: `pgvector/pgvector:pg16`, locale ICU `pt-BR`

**Decisão.** Dev, CI (Testcontainers) e produção usam a mesma imagem, com
`--locale-provider=icu --icu-locale=pt-BR`. Extensões: `vector`, `pg_trgm`,
`unaccent`. *(Formaliza o que foi feito na primeira carga.)*

**Por quê.** A imagem oficial não traz o pgvector; e o locale de sistema `pt_BR.utf8`
não existe na imagem Debian — o ICU é o que faz o `ORDER BY` respeitar acento.

**Custo.** `default_text_search_config` continua `english` — todo `to_tsvector` precisa
dizer `'portuguese'`. Inventário completo em
[Data Warehouse](../02-arquitetura/04-data-warehouse.md#o-que-está-instalado-no-banco).

---

## Riscos

Ordenados por impacto. **Status revisado em 15/09/2026**, após a primeira carga
real; R-14 e R-15 acrescentados em 19/09/2026.

### R-14 · .NET 8 sai de suporte durante o projeto 🟠

O suporte do .NET 8 termina em **10/11/2026**. Depois disso, sem correção de
segurança — num produto hospedado na internet.

**Mitigação.** Migrar para **.NET 10** (LTS, suporte até nov/2028) **agora**, enquanto a
solução é scaffold: trocar `net8.0` por `net10.0` nos sete `.csproj` e atualizar os
pacotes de teste. Depois do primeiro código real, a mesma troca custa uma sprint de
regressão.

### R-15 · O pipeline de carga não está versionado 🟠

A pasta `scraping/` — migrations do DW, coletores, NLP, curadoria de temas e os 24
testes de integridade — **não está em nenhum repositório git**. É o único lugar onde o
schema do DW existe. Um disco perdido leva o DW junto, e ninguém além de quem tem a
pasta consegue rodar a carga ([D-17](#d-17--carga-manual-não-agendada)).

**Mitigação.** Versionar já. Sugestão: pasta `pipeline/` dentro do `API5-Backend` — o
schema fica ao lado da API que o lê, e o CI do backend passa a aplicar as mesmas
migrations nos testes de integração. Alternativa: repositório próprio
(`API5-Pipeline`) na organização Concord-API.

### R-01 · Blocos do mockup sem fonte 🟠 *(era 🔴 — reduzido, não eliminado)*

**Resolvido:** a **doutrina** deixou de ser lacuna (52.696 artigos, 9.186 ligados
a tema, via DOAJ/SciELO/OAI-PMH).

**Continua sem fonte, e agora com motivo confirmado:** citação de acórdão,
fundamentos invocados, valor fixado e relator — todos dependem de **inteiro
teor**, e a investigação mostrou que **os quatro tribunais testados estão
bloqueados** (TJSP e TJMG por captcha, TJRJ por Termos de Uso, STJ por WAF).
Jurisprudência qualificada dependia do PANGEA, não investigado.

Relator tem agora uma confirmação dura: **não existe no payload do DataJud**.

**Mitigação.** As opções viraram concretas: ofício ao TJRJ (única barreira
puramente contratual), convênio institucional, resolução automatizada de captcha
(**decisão da coordenação, não de desenvolvedor**), ou remover do escopo.

**Sinal de alerta:** alguém implementar um desses blocos com dado de demonstração e ele
chegar à apresentação.

---

### R-12 · Base efetiva tem 2 tribunais, não 3 🟠 *(novo)*

100% das 265.088 movimentações do TJMG vindas do DataJud têm `dataHora` **nulo**.
Sem timestamp não há evento, e inventar data violaria o D-11 — então o TJMG
ficou fora da tabela fato.

**Impacto direto:** o componente de cobertura da nota trava, e a distribuição
ficou **0 temas "Consolidados"**. A interface declara escopo de três tribunais
enquanto a base tem dois.

**Mitigação.** Declarar a cobertura efetiva na tela; reprocessar se o TJMG
corrigir o campo (os processos já estão no `raw`); e **todo conector novo checa
completude campo a campo por tribunal**.

---

### R-13 · Dimensões sem historização (SCD) 🟡 *(novo)*

Órgão julgador é renomeado, assunto da TPU é revisado pelo CNJ. Hoje o upsert
**sobrescreve** o atributo, sem guardar histórico. Um agregado recalculado depois
de uma renomeação muda retroativamente, sem rastro.

**Mitigação.** Não endereçado. Decidir se alguma dimensão precisa de SCD tipo 2
antes que a base fique grande demais para migrar.

### R-02 · Duas frentes em scaffold 🟠 *(era 🔴 — o DW saiu do zero)*

O DW está carregado e validado. O backend .NET é scaffold, e o frontend tem scaffold
com o design system nos tokens, mas nenhuma tela.

**Mitigação.** Priorizar o caminho mais curto até um fluxo ponta a ponta com dado real —
uma fonte, um recorte pequeno, uma tela — antes de ampliar. Fatiar por fluxo vertical,
não por camada.

### R-03 · Modelagem do DW ✅ *(era 🔴 — resolvido)*

O esquema existe, está carregado e o checklist de auditoria foi percorrido. Grão
declarado (D-13), pontes no lugar, proveniência em toda linha, carga idempotente
verificada. **463.016 linhas de fato.**

**Restam dois itens do checklist:** o modelo não responde às perguntas que
dependem de inteiro teor (R-01), e não há historização de dimensão (R-13).

O esquema vive nas migrations de `scraping/sql/`, e com o [D-17](#d-17--carga-manual-não-agendada)
é ali que ele fica — não há mais port para o `Ratio.Etl`. Falta versioná-lo
([R-15](#r-15--o-pipeline-de-carga-não-está-versionado-)).

### R-04 · Fontes candidatas não verificadas 🟠

PANGEA pode não ter API pública. JusBrasil provavelmente não é licenciável. Os
repositórios dos tribunais não têm formato padronizado.

**Mitigação.** Um **spike por fonte**, com resultado escrito em
[Fontes](../03-dados/01-fontes.md). Nenhum bloco de tela pode ser planejado sobre fonte
não verificada.

### R-05 · Granularidade do tema 🟠 *(mitigado em parte)*

A camada semântica foi construída: 447 assuntos → 408 temas, com o lastro da TPU
preservado.

**Mas o risco não sumiu.** O agrupamento por embedding junta variação de
**redação** ("Indenização por Dano Moral" / "Indenizaçao por Dano Moral"); ele
**não separa teses** dentro de um mesmo assunto, que é o que o mockup mostra.
Para isso seria preciso a ementa — que depende do inteiro teor (R-01).

### R-06 · Escopo de NLP ✅ *(era 🟠 — definido e implementado)*

Escolhido e feito o **Uso 1** (agrupar assuntos em tema), mais um Uso 4 que não
estava previsto (ligar doutrina a tema). O guarda-corpo está travado por teste:
uma consulta recalcula a contagem direto do fato e falha se o agregado divergir.

**Uso 2** (extrair do inteiro teor) segue bloqueado por R-01. **Uso 3** (redigir
o resumo) não foi feito.

### R-07 · DevOps é requisito e ainda não existe 🟠

CI/CD, deploy automático, monitoramento e documentação são cobrados explicitamente, e
valem nota. Só o frontend tem CI (lint, typecheck, build — sem teste). Backend sem
workflow, nada de deploy nem monitoramento.

**Mitigação.** Começar cedo e pequeno: `Dockerfile` e pipeline de build já na primeira
semana de código. Containerizar no fim do projeto é onde os prazos morrem. Ver
[DevOps](03-devops-e-infra.md).

### R-08 · Recalibração da nota de força ✅ *(era 🟠 — decidido)*

`N = 3` (D-14), parametrizado em `dw.strength_config`. A nota está implementada
com os componentes abertos.

⚠ **Mas a compressão continua**, por outro motivo: com o TJMG fora da base
(R-12), quase todo tema vê 1–2 tribunais. Resultado: **0 temas "Consolidados"**.
É o dado, não a fórmula.

### R-09 · Dependência de fonte externa instável 🟡

O DataJud tem limite de taxa, falha intermitente e pode mudar formato ou rotacionar a
chave. Quanto mais fontes, mais superfície.

**Mitigação.** Retry com backoff em todo coletor (implementado no do DataJud); coletores
idempotentes, que podem ser rodados de novo; e a tela declarando a data da extração.
Com a carga manual, a falha acontece na frente de quem opera — não em silêncio.

### R-10 · Deep link para o processo só funciona em parte 🟡

Entre os três tribunais do escopo, só o TJSP abriu o processo por URL nos testes
anteriores.

**Mitigação.** Três camadas de resposta (`direto` / `portal` / sem link) e o rótulo
"consultar no tribunal", nunca "veja a decisão". Ver
[Limitações](../03-dados/04-limitacoes-da-fonte.md#links-para-o-processo-na-origem).

### R-11 · Custo e privacidade do chatbot 🟡

Chat aberto ao público sem teto é conta imprevisível, e a pergunta do usuário pode conter
dado do caso dele.

**Mitigação.** Definir teto de uso, política de log e retenção antes de expor. Ver
[Chatbot](../01-produto/05-chatbot.md#pontos-em-aberto).

---

## Perguntas em aberto

| Pergunta | Quem decide | Bloqueia |
|---|---|---|
| ~~Qual o grão do fato?~~ | ✅ D-13 | — |
| STJ e STF entram no escopo? | time + cliente | modelagem, nota de força, telas |
| PANGEA tem API utilizável? | spike | jurisprudência qualificada |
| De onde vem o inteiro teor — ou ele sai do escopo? | time + cliente | metade da aba Base Analítica |
| ~~Doutrina: quais repositórios de artigo?~~ | ✅ DOAJ, SciELO, OAI-PMH | — |
| ~~Qual uso de NLP entra na entrega?~~ | ✅ Usos 1 e 4 (R-06) | — |
| Qual modelo/provedor de LLM para o chatbot? | time | custo, privacidade |
| Onde versionar o pipeline de carga (`scraping/`)? | time | R-15 |
| Remover o `Ratio.Etl` da solução? | dev backend | limpeza da solução |
| Migrar para .NET 10 agora? | dev backend | R-14 |
| Controle de migration aplicada: tabela própria ou DbUp lendo os SQL? | dev backend | primeira migration nova |
| Acesso a dados: Dapper ou EF Core? *(recomendação: Dapper — a API só lê)* | dev backend | primeira linha de Infrastructure |
| Com que frequência rodar a carga manual? | time | frescor do dado exibido |
| Só produção, ou produção + staging no Coolify? | time | configuração do CI/CD |
| Ferramenta de monitoramento? | time | R-07 |
| Quais assuntos na carga de demonstração? | time | conteúdo da apresentação |
