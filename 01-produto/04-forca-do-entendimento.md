# Força do entendimento

A nota de 0 a 100 no círculo — a métrica-assinatura do produto. Responde a:
**"quão firme é esse entendimento?"**

> ⚠ **Metodologia proposta, não auditada.** Os pesos e as saturações abaixo vieram de
> raciocínio, não de validação com especialista da área nem de teste empírico. Servem
> como ponto de partida para a discussão — não como fórmula fechada.

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

**`N` precisa ser recalibrado.** A fórmula original saturava em 6 tribunais, pensando em
cobertura nacional. Com o escopo em **três tribunais** (TJSP, TJRJ, TJMG), nenhum tema
jamais atingiria a saturação — o componente ficaria travado em 0,5 no melhor caso, e a
nota inteira ficaria comprimida.

Opções: saturar em 3 (todos os tribunais do escopo), incluir os superiores na contagem
se eles entrarem, ou substituir cobertura por outro sinal. **Decidir antes de exibir
qualquer nota.**

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
  "score": 77,
  "level": "Dominante",
  "components": {
    "agreement": { "value": 0.640, "weight": 0.45 },
    "volume":    { "value": 1.000, "weight": 0.25 },
    "coverage":  { "value": 0.833, "weight": 0.20 },
    "recency":   { "value": 1.000, "weight": 0.10 }
  },
  "basis": {
    "judged": 12418, "granted": 10679, "denied": 1739,
    "courts": 3, "lastDecisionYear": 2026
  }
}
```

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

## Ajustes em aberto

- **Recalibrar a cobertura para o escopo de três tribunais** — bloqueante, ver acima.
- Os pesos e a saturação de volume (300) são **calibragem, não lei**. Escolhidos por
  raciocínio, sem validação empírica. Vale mostrar a alguém da área antes de exibir.
- Uma tese com 95% de concordância em 8 julgamentos ainda pontua alto em concordância.
  O peso do volume atenua, mas considere um piso mínimo de `julgados` para exibir o
  grau textual.
- A recência usa apenas o **ano** da última decisão, não a densidade recente. Um tema
  com uma decisão em 2026 e nenhuma desde 2019 pontua igual a um julgado toda semana.
