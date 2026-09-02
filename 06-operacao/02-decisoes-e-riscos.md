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
[não serve de base](../05-prototipo/01-prototipo-referencia.md).

---

### D-03 · Escopo territorial: TJSP, TJRJ e TJMG

**Decisão.** A aplicação abrange apenas os tribunais de São Paulo, Rio de Janeiro e
Minas Gerais.

**Por quê.** Três estados concentram volume suficiente para o produto fazer sentido, e o
recorte torna viável mapear deep link, investigar repositório de jurisprudência e rodar
carga completa em janela razoável.

**Custo.** A interface **precisa declarar o escopo** — quem lê "82% favorável" assumindo
cobertura nacional tira conclusão errada. Também obriga a
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

### D-06 · Backend em inglês, dados em português

**Decisão.** Todo o backend — classes, métodos, tabelas, colunas, rotas, campos JSON,
logs, commits — em inglês. Os **valores** dos dados permanecem em português.

**Custo.** Traduzir jargão jurídico gera divergência no time. Mitigado por um
[vocabulário PT→EN fechado](../02-arquitetura/02-backend-dotnet.md#idioma) — acrescente
linhas a ele, não invente sinônimos.

---

### D-07 · Hospedagem em VPS Hostinger com Coolify

**Decisão.** VPS na Hostinger, deploy automático via Coolify. CI/CD, documentação e
monitoramento obrigatórios; ferramentas a definir.

**Implicações.** Tudo precisa ser containerizável; health check vira contrato;
configuração por variável de ambiente; e o banco divide recursos com a aplicação, o que
torna backup e agendamento de carga responsabilidade nossa.

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

**Por quê.** Um código mal classificado corromperia todo o favorável/desfavorável do
produto **sem sintoma visível**.

**Custo.** Subcontagem em vez de erro. É a direção certa de errar.

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

## Riscos

Ordenados por impacto.

### R-01 · A maior parte do mockup não tem fonte de dados 🔴

Citação de acórdão, fundamentos invocados, jurisprudência qualificada, doutrina, valor
fixado e relator — **nenhum tem lastro no DataJud**, e nenhuma fonte alternativa está
confirmada. É o risco central do projeto.

**Mitigação.** Decidir por bloco, e cedo: PANGEA, repositório de tribunal, NLP, ou **fora
do escopo**. Ver [Limitações da fonte](../03-dados/04-limitacoes-da-fonte.md).

**Sinal de alerta:** alguém implementar um desses blocos com dado de demonstração e ele
chegar à apresentação.

### R-02 · Três frentes em branco e nenhuma base reaproveitável 🔴

O backend .NET é scaffold, o frontend não existe, e o protótipo
[não serve de base](../05-prototipo/01-prototipo-referencia.md). Na prática, o projeto
começa do zero em código.

**Mitigação.** Priorizar o caminho mais curto até um fluxo ponta a ponta com dado real —
uma fonte, um recorte pequeno, uma tela — antes de ampliar. Fatiar por fluxo vertical,
não por camada.

### R-03 · Modelagem do DW ainda não existe 🔴

O esquema do protótipo está fora, e não há substituto. Tudo depende disso: ETL, API,
telas, chatbot e os testes de integridade que o desafio exige.

**Mitigação.** Sessão de modelagem com o time, começando pela declaração do
[grão](../03-dados/02-modelo-dimensional.md#decisão-2--o-grão-duas-opções-em-aberto), e
percorrendo o [checklist de auditoria](../03-dados/02-modelo-dimensional.md#checklist-de-auditoria--antes-da-primeira-migration)
antes da primeira migration. É o maior desbloqueio disponível hoje.

### R-04 · Fontes candidatas não verificadas 🟠

PANGEA pode não ter API pública. JusBrasil provavelmente não é licenciável. Os
repositórios dos tribunais não têm formato padronizado.

**Mitigação.** Um **spike por fonte**, com resultado escrito em
[Fontes](../03-dados/01-fontes.md). Nenhum bloco de tela pode ser planejado sobre fonte
não verificada.

### R-05 · Granularidade do tema 🟠

Tema = assunto da TPU não separa as cinco teses que o mockup mostra para uma mesma
consulta.

**Mitigação.** Camada semântica **em cima** do assunto TPU, preservando o lastro. Ver
[O que é um tema](../01-produto/03-tema-modelo-conceitual.md).

### R-06 · Escopo de NLP indefinido 🟠

"Normalizar com NLP usando alguma LLM" é intenção, não plano. Três usos possíveis, com
custos muito diferentes, e um deles bloqueado por falta de texto.

**Mitigação.** Escolher **um** uso (recomendação: agrupamento em tema) e cravar o
guarda-corpo: o modelo não conta. Ver [ETL e NLP](../02-arquitetura/05-etl-e-nlp.md).

### R-07 · DevOps é requisito e ainda não existe 🟠

CI/CD, deploy automático, monitoramento e documentação são cobrados explicitamente, e
valem nota. Nada está configurado.

**Mitigação.** Começar cedo e pequeno: `Dockerfile` e pipeline de build já na primeira
semana de código. Containerizar no fim do projeto é onde os prazos morrem. Ver
[DevOps](03-devops-e-infra.md).

### R-08 · Nota de força precisa ser recalibrada para três tribunais 🟠

O componente de cobertura saturava em 6 tribunais. Com escopo em três, ele nunca chega
ao topo e comprime a nota inteira.

**Mitigação.** Recalibrar antes de exibir qualquer nota. Ver
[Força do entendimento](../01-produto/04-forca-do-entendimento.md#cobertura).

### R-09 · Dependência de fonte externa instável 🟡

O DataJud tem limite de taxa, falha intermitente e pode mudar formato ou rotacionar a
chave. Quanto mais fontes, mais superfície.

**Mitigação.** Retry com backoff em todo conector; **alarme quando a carga falhar**; e a
tela tratando "dado velho" com honestidade.

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
| Qual o grão do fato? | time | **tudo** — R-03 |
| STJ e STF entram no escopo? | time + cliente | modelagem, nota de força, telas |
| PANGEA tem API utilizável? | spike | jurisprudência qualificada |
| De onde vem o inteiro teor — ou ele sai do escopo? | time + cliente | metade da aba Base Analítica |
| Doutrina: quais repositórios de artigo? | time | bloco de doutrina |
| Qual uso de NLP entra na entrega? | time | R-06 |
| Qual modelo/provedor de LLM para o chatbot? | time | custo, privacidade |
| Migrations: DbUp ou FluentMigrator? | dev backend | primeira migration |
| Acesso a dados: Dapper ou EF Core? | dev backend | primeira linha de Infrastructure |
| Só produção, ou produção + staging no Coolify? | time | configuração do CI/CD |
| Ferramenta de monitoramento? | time | R-07 |
| Quais assuntos na carga de demonstração? | time | conteúdo da apresentação |
