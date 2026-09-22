# Padrão de commits

**Modelo adotado:** convenção semântica de commits, **em inglês**, em todos os
repositórios do projeto.

---

## As regras

1. **Todo commit é escrito em inglês**, em qualquer repositório do projeto — código ou
   documentação.
2. **Cada commit é pequeno, descritivo e objetivo.** Uma mudança com um propósito.
3. **Todo commit começa com um tipo**, seguido de dois-pontos e da descrição.
4. **Nenhum commit leva ferramenta de IA como coautora.** Remova qualquer
   `Co-Authored-By:` atribuído a IA. A regra não proíbe coautoria humana.
   [Por quê](#por-que-não-atribuir-coautoria-a-ia).
5. **O commit tem só a linha do assunto.** Sem corpo descritivo: o que o commit faz cabe
   na descrição curta, e o porquê fica no PR, na task ou nos Docs.

## Tipos

| Tipo | Quando usar |
|---|---|
| `feat:` | nova funcionalidade |
| `fix:` | correção de bug ou comportamento inesperado |
| `refactor:` | melhoria de código sem alterar comportamento |
| `docs:` | atualização de documentação |
| `test:` | adicionar ou alterar testes do projeto: unidade, integração, contrato, ponta a ponta |
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
| `fix: handle empty result set` | `fix: handle empty result set`<br>`Co-Authored-By: AI Assistant <ai@example.com>` | IA registrada como coautora |
| `chore: configure CI pipeline for backend` | `chore: setup, fix tests and add endpoint` | três mudanças num commit só |
| `test: cover empty doctrine state` | `feat: add tests` | teste é `test:`, não `feat:` |

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
configuradas hoje, mas o padrão já as torna possíveis sem reescrever histórico. A versão das releases **não** sai dos commits: vem do label `release:*` do PR — ver [Versionamento e releases](04-versionamento-e-releases.md).

### Por que commits pequenos

**Revisão mais fácil.** Um commit com um propósito se lê em segundos; um commit que mexe
em cinco coisas obriga o revisor a separar mentalmente o que o autor não separou.

**Reversão cirúrgica.** Se uma mudança der problema, dá para desfazer só ela, sem levar
junto o resto.

**Rastreamento do trabalho.** Ver abaixo.

### Por que não atribuir coautoria a IA

Assistentes de IA podem acrescentar `Co-Authored-By:` automaticamente. Neste projeto,
o uso da ferramenta não deve ser registrado como coautoria: a responsabilidade pelo
commit continua com as pessoas que produziram e revisaram a mudança.

**A restrição vale para IA, não para colegas.** Coautoria humana pode ser registrada.
Antes de commitar, remova trailers que atribuam coautoria a uma ferramenta de IA.

**Não haverá validação de coautoria no CI nem hook para bloquear commits por isso.**
A regra é uma orientação para quem prepara o commit e para os assistentes utilizados.

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

- **O teste vai num commit `test:`, e o código que o faz passar vai no `feat:` ou `fix:`
  seguinte.** O histórico mostra o ciclo do TDD: primeiro o teste que falha, depois o
  código que o atende.
- **O commit `test:` fica vermelho sozinho, e isso é esperado.** O CI roda no PR, sobre o
  último commit da branch, então o que precisa estar verde é o PR, não cada commit.
- **Todo `fix:` vem depois do `test:` que reproduz o bug.** Um `fix:` sem teste é um bug
  que pode voltar.

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
