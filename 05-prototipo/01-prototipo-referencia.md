# Protótipo — o que é e o que não é

> ## ⚠ O protótipo NÃO é fonte de verdade
>
> A pasta `prototipo/` foi construída **antes** das definições atuais de produto,
> escopo, fontes e arquitetura. Nada nela foi auditado. Em particular:
>
> - **a modelagem do banco foi criada sem auditoria alguma** e não serve de base para
>   o Data Warehouse real;
> - o contrato de API, os nomes, as métricas e as regras de negócio antecedem as
>   decisões que valem hoje;
> - o escopo dele (91 tribunais, fonte única) contradiz o escopo atual (SP/RJ/MG,
>   multifonte);
> - o backend dele está em português; o nosso será em inglês.
>
> **Em caso de divergência entre o protótipo e esta wiki, a wiki vence.** Em caso de
> dúvida sobre algo que a wiki não cobre, a resposta é *decidir*, não copiar do
> protótipo.

## Então para que ele serve

Para três coisas, e só:

**1 · Prova de que a fonte responde.** O DataJud foi chamado de verdade, com dado real
voltando. Isso elimina a dúvida "será que dá para fazer?" — dá.

**2 · Catálogo de armadilhas conhecidas.** O protótipo tropeçou em coisas que a
documentação do CNJ não avisa, e cada tropeço vale como aviso — não como código a
copiar. Ver [a lista abaixo](#armadilhas-que-o-protótipo-encontrou).

**3 · Demonstração visual.** O HTML/CSS mostra o
[design system](../04-design/01-design-system.md) aplicado, útil como referência de
aparência.

## O que tem dentro

```
prototipo/
├── index.html            três telas em HTML estático
├── assets/               api.js · app.js · data.js · styles.css
├── arte/                 ilustração da Justiça
└── backend/              FastAPI + Postgres  (Python)
    ├── app/              main.py · forca.py · links.py
    ├── etl/              datajud.py · carga.py · tpu.py
    ├── db/migrations/    esquema — NÃO AUDITADO
    └── tests/            21 casos, sem banco e sem rede
```

Stack: FastAPI, Postgres 16, psycopg 3, httpx, pytest. Frontend em HTML/CSS/JS puro.

## Armadilhas que o protótipo encontrou

Isto é o conteúdo de valor real. Cada item é uma hora que o time não precisa perder de
novo — mas **cada um deve ser reverificado** antes de virar decisão, porque nada aqui
passou por auditoria.

### Sobre a API do DataJud

| Achado | Implicação |
|---|---|
| Um índice por tribunal, sem endpoint agregado | coletar de N tribunais = N chamadas |
| Teto de 100 documentos por página | paginação obrigatória; `from`/`size` estoura |
| Falha intermitente e limite de taxa | retry com backoff não é luxo |
| O JSON vem inconsistente entre tribunais | um campo ora é objeto, ora string, ora ausente — achatar antes de tocar o banco |

### Sobre a TPU

O DataJud **não publica resultado de julgamento como campo**. O desfecho só existe como
código de movimentação. O protótipo mapeou três códigos conferindo o nome que a própria
API devolve:

| Código | Nome devolvido pela API | Interpretação |
|---|---|---|
| 219 | Procedência | acolhido |
| 220 | Improcedência | rejeitado |
| 221 | Procedência em Parte | acolhido em parte |

**Como usar isto:** como ponto de partida da verificação, não como mapa pronto. O
mapeamento definitivo precisa de auditoria própria, e a regra que o protótipo adotou
merece ser mantida: código não conferido não entra na métrica. Um código mal
classificado corrompe o resultado do produto inteiro sem sintoma visível.

### Sobre links para o processo na origem

Verificado abrindo a URL e **inspecionando o conteúdo** — o código HTTP engana, porque
a página de erro do e-SAJ também devolve 200:

| Tribunal | Sistema | Resultado observado |
|---|---|---|
| TJ-SP | e-SAJ, GET com parâmetros | abriu o processo (classe, assunto e vara conferidos) |
| TJ-RJ | SPA Angular | ignora o parâmetro da URL |
| TJ-MG | PJe/JSF | exige POST com sessão |

Como o escopo agora é exatamente esses três tribunais, este achado é diretamente
relevante — e a conclusão de produto vale: o rótulo do link é **"consultar no
tribunal"**, nunca "veja a decisão". Não dá para garantir por teste que um link abra um
processo específico.

### Sobre os testes

21 casos que rodam sem banco e sem rede. **Quase todo caso do arquivo de teste do ETL
saiu de uma falha real contra a API do DataJud** — cada um documenta uma inconsistência
que a fonte realmente produz. Vale **ler** os casos para saber o que testar; não vale
portar cegamente, porque testam um modelo de dados que não é o nosso.

## O que explicitamente não reaproveitar

| Item | Motivo |
|---|---|
| **`db/migrations/*.sql`** | modelagem criada sem auditoria — o DW real parte da [modelagem própria](../03-dados/02-modelo-dimensional.md), não daqui |
| O contrato de API | anterior às definições atuais; o nosso é em inglês e tem campos e filtros que o protótipo não tem |
| A nota de força como está | metodologia plausível, **não auditada**; ver [Força do entendimento](../01-produto/04-forca-do-entendimento.md) |
| O escopo de carga (91 tribunais) | hoje são três: TJSP, TJRJ, TJMG |
| A premissa de fonte única | o produto é [multifonte](../03-dados/01-fontes.md) |
| Nomes em português no código | o backend será [em inglês](../02-arquitetura/02-backend-dotnet.md#idioma) |
| `assets/data.js` | dados de demonstração |
| O painel de chat | casca sem backend; o [chatbot real](../01-produto/05-chatbot.md) é outra coisa |

## Divergências de vocabulário

O protótipo chama de **"tese"** o que hoje chamamos de **"tema"**. Também não tem tag
de matéria, resumo em prosa, filtros na tela de resultados, nem a aba Resumo.

Ao ler o código do protótipo, traduza mentalmente — e não deixe o vocabulário antigo
voltar para a wiki ou para o código novo.

## Recomendação

Trate `prototipo/` como um **caderno de campo**: útil para saber onde o terreno é
irregular, inútil como planta da casa. Leia antes de integrar uma fonte, releia quando
uma chamada ao DataJud falhar de um jeito estranho — e não copie e cole nada.

---

## Protótipo de dados — setembro/2026

Outra coisa, com outro propósito. Pasta `prototipo-prod - versao 202609/`, na pasta de
trabalho (fora de repositório).

| | Protótipo antigo (`prototipo/`) | Protótipo de dados (`…versao 202609/`) |
|---|---|---|
| Para quê | primeira tentativa de produto | **ver o dado real do DW numa tela** |
| Banco | modelagem própria, sem auditoria | **o DW oficial** (`api5-dw`), só leitura |
| Contrato | divergente | o [contrato documentado](../02-arquitetura/02-backend-dotnet.md#contrato-da-api), em inglês, com retorno em português |
| Vale como referência? | ❌ | ✅ para **formato de resposta, consultas SQL e comportamento do dado na tela** |
| Vale como código a portar? | ❌ | ❌ — API em FastAPI; web em React Router + CSS puro, não na stack oficial |

O que ele revelou sobre o dado (entidades HTML cruas na doutrina, temas com um tribunal
só, o tamanho real dos blocos sem fonte) está no `README.md` dele.

Rodar: `api/` com `python -m uvicorn main:app --port 5000`; `web/` com `npm run dev`
(porta 5173). Precisa do `api5-dw` de pé.

