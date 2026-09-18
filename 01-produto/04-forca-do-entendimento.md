# Força do entendimento

A nota de 0 a 100 no círculo — a métrica-assinatura do produto. Responde a:
**"quão firme é esse entendimento?"**

> ⚠ **Metodologia proposta, não auditada.** Os pesos e as saturações abaixo vieram de
> raciocínio, não de validação com especialista da área nem de teste empírico. Servem
> como ponto de partida para a discussão — não como fórmula fechada.
>
> ✅ **Implementada em 15/09/2026** sobre a carga real (408 temas), em
> `dw.theme_strength`. Os pesos continuam sendo calibragem não validada — mas
> agora vivem em `dw.strength_config`, uma linha de tabela, alteráveis sem
> mexer em SQL de view.
>
> 🔴 **Antes de exibir qualquer nota, leia
> [Polaridade do resultado](../03-dados/05-polaridade-do-resultado.md).** A carga
> revelou que "% favorável" sem dizer favorável a quem induz à conclusão
> invertida — em matéria penal, procedência é condenação.

## Por que ela existe

Volume não é força. Um tema com 12.418 processos divididos 50/50 é uma **divergência
enorme**, não um entendimento consolidado — e ordenar a busca por volume colocaria
justamente as brigas no topo. A nota existe para ordenar por *utilidade para quem vai
citar*.

## Os quatro componentes

| Componente | Peso | O que mede | Por que esse peso |
|---|---|---|---|
| **Concordância** | 0,45 | o quanto as decisões apontam para o mesmo lado | é literalmente a pergunta do usuário |
| **Volume** | 0,25 | quantos julgamentos sustentam a tese (log) | 10→100 decisões importa mais que 1.000→1.090 |
| **Cobertura** | 0,20 | em quantos tribunais a tese aparece | tese firme em um tribunal só é local, não consolidada |
| **Recência** | 0,10 | se ainda há julgamento recente | tese parada pode ter sido superada — o pior caso para quem cita |

Os pesos somam 1,0. `nota = round(Σ componente × peso × 100)`.

## Cada fórmula

### Concordância

```
majority  = max(granted, denied) / judged
agreement = max(0, (majority - 0.5) * 2)
```

Reescala de [0,5 … 1,0] para [0 … 1]: 50/50 é divergência total (0), 100/0 é
unanimidade (1). Sem a reescala, uma tese empatada já começaria em 0,5 e pareceria
meio consolidada.

Procedência em parte somando com procedência é uma decisão
[ainda em aberto](../03-dados/03-agregados-olap.md#topic_summary--resumo-por-tema).

### Volume

```
volume = min(1, log10(1 + judged) / log10(1 + 300))
```

Logarítmico e saturando em **300 julgamentos**. Acima disso, mais decisões não tornam
a tese mais consolidada — só mais litigada.

### Cobertura

```
coverage = min(1, courts / N)
```

> ### ✅ R-08 decidido: **N = 3** (15/09/2026)
>
> A fórmula original saturava em 6 tribunais, pensando em cobertura nacional.
> Com o escopo em três, nenhum tema atingiria a saturação e a nota inteira
> ficaria comprimida.
>
> **Adotada a primeira opção listada: saturar em 3**, todos os tribunais do
> escopo declarado. Cobertura mede "a tese aparece em quantos tribunais do
> universo coberto", e o universo declarado do produto é 3.
>
> `N` vive em `dw.strength_config.coverage_courts`, **não cravado na fórmula** —
> se STF/STJ entrarem no escopo, muda numa linha.

⚠ **Efeito colateral real na carga atual.** Como o
[TJMG ficou fora da tabela fato](../03-dados/04-limitacoes-da-fonte.md#5b--completude-do-dado-varia-por-tribunal),
quase todo tema vê 1 ou 2 tribunais. A cobertura trava em 0,333–0,667 e puxa a
nota para baixo: dos 292 temas com julgamento, a distribuição ficou **213 "Em
formação", 75 "Divergente", 4 "Dominante", 0 "Consolidada"**.

Não é defeito da fórmula — é o dado disponível. Some quando o TJMG entrar. Mas
**exibir "0 temas consolidados" na banca sem essa explicação é péssimo**, então
a interface precisa declarar a cobertura efetiva.

### Recência

```
age = currentYear - lastDecisionYear
age <= 1  ->  1.0
age >= 6  ->  0.0
else      ->  1 - (age - 1) / 5
```

Sem julgamento há 6 anos, a tese é tratada como possivelmente parada.

## Vocabulário do grau

A nota vem acompanhada de um rótulo em linguagem que já existe no meio jurídico — não
uma escala inventada:

| Nota | Grau |
|---|---|
| ≥ 90 | **Consolidada** |
| 75 – 89 | **Dominante** |
| 55 – 74 | **Em formação** |
| < 55 | **Divergente** |

## Auditabilidade — a regra que não se negocia

A API **não** devolve só o número. Devolve cada componente, com seu peso e com a base
de cálculo:

```json
{
  "score": 80,
  "level": "Dominante",
  "components": {
    "agreement": { "value": 0.972, "weight": 0.45 },
    "volume":    { "value": 0.872, "weight": 0.25 },
    "coverage":  { "value": 0.333, "weight": 0.20 },
    "recency":   { "value": 0.800, "weight": 0.10 }
  },
  "basis": {
    "judged": 144, "claimUpheld": 142, "claimRejected": 2,
    "courts": 1, "lastDecisionYear": 2024
  },
  "polarity": {
    "agreementBasis": "pretensao_autor",
    "dominantClaimant": "acusacao",
    "label": "acolhimento da pretensão acusatória (procedência = condenação)"
  }
}
```

Exemplo real da carga (*Tráfico e posse de drogas*). Três mudanças em relação ao
esboço original desta página:

1. `granted`/`denied` viraram **`claimUpheld`/`claimRejected`** — a palavra
   "favorável" saiu do contrato;
2. o bloco **`polarity`** é novo e **obrigatório**: sem ele, "142 de 144" é
   ambíguo. Ver [Polaridade do resultado](../03-dados/05-polaridade-do-resultado.md);
3. `agreementBasis` declara sobre qual família de códigos a concordância foi
   calculada — mérito e recurso **nunca** são somados.

Campos em inglês, valores de domínio (`level`) em português — ver
[Idioma](../02-arquitetura/02-backend-dotnet.md#idioma).

Um juiz ou advogado **não cita estatística que não consegue auditar**. "A IA calculou
81" é exatamente o tipo de número que um profissional descarta. Por isso a nota é
determinística, derivada só de agregação sobre o DW, e nenhum componente é opinião de
modelo. Vale igualmente quando a nota for citada pelo [chatbot](05-chatbot.md).

## Consequências para o frontend

- O círculo mostra o número, mas a decomposição precisa estar acessível — tooltip,
  painel ou seção da base analítica. Não esconda os componentes.
- O frontend **não recalcula** a nota. Ela vem pronta.
- Se não há julgamento apurado, o tema não aparece na busca.

## Testes que travam a fórmula

`scraping/sql/014_strength_link_tests.sql` — cada um devolve zero linhas quando
está certo. Os que importam:

- **3** — recalcula o score **do zero** a partir dos componentes e falha se
  divergir do valor exposto. A fórmula não pode mentir sem o CI perceber;
- **4** — grau textual incompatível com a faixa da nota;
- **7** — pesos que não somam 1,0;
- **13** — concordância calculada sobre famílias de polaridade misturadas;
- **14** — tema com julgados mas sem rótulo de polaridade.

## Ajustes em aberto

- ~~Recalibrar a cobertura para três tribunais~~ → **decidido: N = 3** (ver acima).
- Os pesos e a saturação de volume (300) são **calibragem, não lei**. Escolhidos por
  raciocínio, sem validação empírica. Vale mostrar a alguém da área antes de exibir.
  Ficam em `dw.strength_config`.
- Uma tese com 95% de concordância em 8 julgamentos ainda pontua alto em concordância.
  O peso do volume atenua, mas considere um piso mínimo de `julgados` para exibir o
  grau textual. **Concreto na carga atual:** *Roubo Majorado* pontua 74 com
  **8 julgados**.
- A recência usa apenas o **ano** da última decisão, não a densidade recente. Um tema
  com uma decisão em 2026 e nenhuma desde 2019 pontua igual a um julgado toda semana.
- **Novo:** "procedência em parte" está somando com procedência no
  `claim_upheld_count`. Continua sendo
  [decisão em aberto](../03-dados/03-agregados-olap.md#topic_summary--resumo-por-tema),
  mas agora tem consequência medida: 338 dos 1.636 julgados são parciais (21%).
