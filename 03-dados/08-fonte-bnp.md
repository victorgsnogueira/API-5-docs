# Fonte para os precedentes qualificados: Pangea/BNP (spike 0.113)

**Situação em 16/11/2026.** A US-41 (precedentes qualificados do tema) depende de uma fonte
verificada (DoR 7). Hoje o DW só tem os **temas repetitivos do STJ** (`dw.dim_precedent`,
carregada do CSV de dados abertos do STJ). Este spike verificou se o **Banco Nacional de
Precedentes (BNP)** do CNJ, pesquisado pelo **Pangea/BNP**, serve como fonte para o resto.

## As três portas do BNP

| Porta | Endereço | Para quem | Acesso |
|---|---|---|---|
| Site de pesquisa Pangea/BNP | `pangeabnp.pdpj.jus.br` | público ("ferramenta pública", diz o manual) | navegador, sem login |
| API interna do Pangea | `pangeabnp.pdpj.jus.br/api/v1` | o próprio site (Angular) | sem documentação pública, sem termos de uso para acesso automatizado |
| API do BNP (webservice "BNP integração") | produção `bnp-sempj.cloud.pje.jus.br`; homologação `bnp-integracao.stg.cloud.pje.jus.br` | os **tribunais**, que enviam e consultam os precedentes | **SSO da PDPJ** com `client_id` e `client_secret` (`grant_type=client_credentials`); a credencial é pedida **por chamado ao CNJ** (`suporteti.cnj.jus.br`) |

O Pangea é do TRT4 para o CNJ: um job lê a API do BNP com usuário e senha próprios e indexa
os precedentes num OpenSearch, que o site pesquisa.

## O que o BNP guarda

Pela Portaria CNJ nº 116/2022, cada tribunal envia, por tema:

- número do tema, categoria (Recurso Repetitivo, IRR, **IAC**, PUIL, **IRDR**, Representativo
  da TNU) e situação (Admitido, Mérito Julgado, Transitado em Julgado, **Cancelado**,
  **Sobrestado** etc.);
- **questão submetida a julgamento** e **tese firmada**;
- textos da decisão e do acórdão de admissão, da ementa e do acórdão de mérito e da decisão
  de suspensão;
- relator, órgão julgador, classe e **número dos processos paradigma**, no número único CNJ
  (Resolução CNJ nº 65/2008), e os processos sobrestados.

A API ainda tem métodos de **Recurso Repetitivo** (STJ/TST) e **Repercussão Geral** (STF).
**Súmulas não estão** entre as categorias da Portaria.

## As perguntas que a wiki deixou

| Pergunta (`01-fontes.md`) | Resposta |
|---|---|
| Existe API pública, ou só interface web? | **Só a interface web é pública.** A API oficial exige credencial; a API interna do site não é documentada nem liberada para uso de terceiros |
| Exige credencial PDPJ / certificado? | **Sim**: token SSO da PDPJ com `client_secret`, concedido por chamado ao CNJ e pensado para tribunais |
| Cobre TJSP, TJRJ e TJMG? | **Em tese, sim**: a Portaria obriga os TJs a enviar IRDR e IAC. O volume por tribunal não foi conferido, porque isso exige a API |
| Entrega o texto ou só a ficha? | **Os dois**: questão, tese, situação, ementas e inteiro teor dos acórdãos |
| Liga com o número CNJ do DataJud? | **Sim**: processos paradigma e sobrestados vêm no número único |
| Os termos permitem armazenar em base própria? | **Não há termos publicados.** Não existe licença de dados abertos nem regra para acesso automatizado; o Swagger de produção nem está publicado (aponta para o exemplo "petstore") |

## O que o BNP traria além do DW

| Hoje no DW (`dim_precedent`) | O BNP acrescentaria |
|---|---|
| temas repetitivos do STJ, com tese, questão, situação e órgão | **IRDR e IAC do TJSP, do TJRJ e do TJMG**, que é a maior lacuna da US-41 |
| — | Repercussão Geral do STF |
| — | situação detalhada (sobrestamento, cancelamento) e processos sobrestados por tema |
| — | ementas e acórdãos de admissão e de mérito |

## Estabilidade do endpoint

- O endereço de produção está na FAQ oficial, e mudanças na API são avisadas por lista de
  e-mail aos tribunais (`l-bnp.integracao`).
- O sistema antigo (BNPR) continua no ar até a migração terminar, sem prazo.
- Sem a especificação publicada, não dá para afirmar contrato estável para quem não é
  tribunal.

## Conclusão

**O BNP é a fonte certa, mas hoje não é uma fonte verificada para o Ratio.** O dado existe e
é o que a US-41 precisa, inclusive a ligação pelo número CNJ. Só que a única forma
automatizada documentada exige uma credencial que o CNJ concede a tribunais, e não há termos
que autorizem coletar pelo site público ou pela API interna dele. Este spike **não coletou
dados** do Pangea nem da API do BNP.

## Recomendação do spike

1. **Pedir acesso ao CNJ**, aos gerentes do projeto BNP indicados na documentação, explicando
   o uso (projeto acadêmico, só leitura, só matéria cível de TJSP, TJRJ, TJMG, STJ e STF). Se
   houver credencial ou autorização por escrito, o BNP vira a fonte da US-41.
2. **Enquanto isso, não comprometer a US-41 inteira.** Um recorte possível com fontes
   oficiais já abertas:
   - STJ: os temas repetitivos que o DW já tem;
   - STF: a Repercussão Geral pelos dados abertos do programa Corte Aberta (mesma família de
     fonte do [spike 0.93](07-fonte-stf.md)), a verificar;
   - IRDR e IAC dos TJs: as páginas do NUGEPNAC de cada tribunal (o TJSP oferece exportar a
     planilha dos temas). Cada tribunal precisa ter os termos conferidos antes; o TJRJ proíbe
     robôs no portal (ver `01-fontes.md`).
3. **Consequência para as tasks:** a 41.1 ("da fonte verificada") fica bloqueada até a
   resposta do CNJ, ou é reescrita por origem (STF pelo Corte Aberta, TJs pelo NUGEPNAC),
   cada uma com o seu spike curto de termos de uso.

Fontes consultadas em 16/11/2026: documentação Pangea/BNP da PDPJ
(`docs.pdpj.jus.br/servicos-negociais/pangeabnp/`), manual de utilização do Pangea/BNP
(`docs.pdpj.jus.br/servicos-negociais/BNP-Pangea/`), configuração pública do site
(`pangeabnp.pdpj.jus.br/assets/config/env.js`), Swaggers de produção e de homologação do
BNP, Portaria CNJ nº 116/2022 (`atos.cnj.jus.br/atos/detalhar/4475`), portal Corte Aberta do
STF e páginas do NUGEPNAC do TJSP.
