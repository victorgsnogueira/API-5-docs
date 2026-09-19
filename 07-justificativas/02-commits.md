# Padrão de commits

**Modelo adotado:** convenção semântica de commits, **em inglês**, em todos os
repositórios do projeto.

---

## As regras

1. **Todo commit é escrito em inglês**, em qualquer repositório do projeto — código ou
   documentação.
2. **Cada commit é pequeno, descritivo e objetivo.** Uma mudança com um propósito.
3. **Todo commit começa com um tipo**, seguido de dois-pontos e da descrição.

## Tipos

| Tipo | Quando usar |
|---|---|
| `feat:` | nova funcionalidade |
| `fix:` | correção de bug ou comportamento inesperado |
| `refactor:` | melhoria de código sem alterar comportamento |
| `docs:` | atualização de documentação |
| `test:` | teste **sem** mudança de código de produção — cobrir comportamento que já existia |
| `chore:` | tarefas de configuração, build ou manutenção |

## Formato

```
<tipo>: <descrição curta, no imperativo, em inglês>
```

Exemplo:

```
feat: add semantic search by legal topic
```

### Mais exemplos

| ✅ Assim | ❌ Não assim | Problema |
|---|---|---|
| `feat: add topic detail endpoint` | `feat: adicionado endpoint de tema` | português |
| `fix: handle missing court code in DataJud response` | `fix: bug` | não descreve nada |
| `refactor: extract strength score into domain service` | `refactor: changes` | não descreve nada |
| `docs: add branch naming standard` | `update docs` | sem tipo |
| `chore: configure CI pipeline for backend` | `chore: setup, fix tests and add endpoint` | três mudanças num commit só |
| `test: cover empty doctrine state` | `feat: add tests` | teste sem código novo é `test:` |

---

## Por que este padrão

### Por que em inglês

Coerente com a decisão de que [o backend inteiro é escrito em inglês](../02-arquitetura/02-backend-dotnet.md#idioma):
se o código, os nomes e os logs estão em inglês, o histórico que descreve esse código
também está. Misturar idiomas no histórico torna a busca (`git log --grep`) imprevisível.

Vale inclusive para os repositórios de documentação — a regra é uma só para o projeto
inteiro, o que evita a pergunta "neste repo é em qual idioma?".

### Por que um tipo em cada commit

**O histórico vira legível de relance.** Dá para separar, só pelo prefixo, o que é
funcionalidade nova, correção e manutenção — útil em revisão, em retrospectiva de sprint
e na hora de montar o que foi entregue.

**Abre caminho para automação.** A convenção semântica é um formato conhecido por
ferramentas de validação de mensagem, geração de changelog e versionamento. Não estão
configuradas hoje, mas o padrão já as torna possíveis sem reescrever histórico.

### Por que commits pequenos

**Revisão mais fácil.** Um commit com um propósito se lê em segundos; um commit que mexe
em cinco coisas obriga o revisor a separar mentalmente o que o autor não separou.

**Reversão cirúrgica.** Se uma mudança der problema, dá para desfazer só ela, sem levar
junto o resto.

**Rastreamento do trabalho.** Ver abaixo.

### Commit como indicador de contribuição

O **número de commits é usado como indicador de contribuição individual e de progresso
da sprint**, permitindo rastrear o fluxo de trabalho no repositório.

Isso só funciona porque as regras acima valem para todos: commits pequenos e com
propósito tornam a contagem comparável entre pessoas e entre sprints. Por isso a regra
de tamanho anda junto com a revisão — sem ela, o indicador mede volume, não
contribuição.

---

## Relação com o TDD

O projeto segue [TDD](03-tdd.md). No histórico, isso aparece assim:

- **Teste e implementação vão no mesmo commit**, com o tipo da mudança (`feat:`,
  `fix:`). O ciclo vermelho → verde acontece na máquina; o commit registra o par já
  verde. Assim nenhum commit da branch deixa o CI vermelho.
- **`test:`** é só para commit que acrescenta teste sem tocar código de produção.
- **Todo `fix:` traz o teste que reproduz o bug.** Um `fix:` sem teste é um bug que pode
  voltar.

## Estado nos repositórios (19/09/2026)

| Repo | Histórico | Segue o padrão? |
|---|---|---|
| `API5-Frontend` | `feat: initial scaffold with design system tokens and file-based routing` | ✅ |
| `API5-Backend` | `initial commit` | ❌ sem tipo — é o commit inicial; vale daqui em diante |
| `API-5-docs` | `docs: …` desde `550a71c` | ✅ (o `feat: adicionado docs 02-09-2026` é anterior ao padrão) |

## Relação com as branches

O commit descreve **o que** mudou; a branch diz **a que task** a mudança pertence.
Juntos, levam de qualquer linha de código até a task no board. Ver
[Padrão de branches](01-branches.md).
