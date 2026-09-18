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

## Riscos

Ordenados por impacto. **Status revisado em 15/09/2026**, após a primeira carga
real.

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

### R-02 · Três frentes em branco e nenhuma base reaproveitável 🔴

O backend .NET é scaffold, o frontend não existe, e o protótipo
[não serve de base](../05-prototipo/01-prototipo-referencia.md). Na prática, o projeto
começa do zero em código.

**Mitigação.** Priorizar o caminho mais curto até um fluxo ponta a ponta com dado real —
uma fonte, um recorte pequeno, uma tela — antes de ampliar. Fatiar por fluxo vertical,
não por camada.

### R-03 · Modelagem do DW ✅ *(era 🔴 — resolvido)*

O esquema existe, está carregado e o checklist de auditoria foi percorrido. Grão
declarado (D-13), pontes no lugar, proveniência em toda linha, carga idempotente
verificada. **463.016 linhas de fato.**

**Restam dois itens do checklist:** o modelo não responde às perguntas que
dependem de inteiro teor (R-01), e não há historização de dimensão (R-13).

⚠ **O esquema está no spike `scraping/`, não no `Ratio.Etl` oficial em .NET.**
A modelagem é a mesma; o host é que falta portar.

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
valem nota. Nada está configurado.

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
