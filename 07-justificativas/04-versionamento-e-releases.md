# Versionamento e releases

**Modelo adotado:** cada pull request para a `main` declara, por **label**, se gera uma
versão e de que tamanho. O merge publica a release sozinho. Vale para os repositórios de
código (`API5-Backend`, `API5-Frontend`), com o mesmo mecanismo nos dois.

---

## O label `release:*`

Todo PR para a `main` precisa ter **exatamente um** destes labels:

| Label | Efeito na versão | Exemplo | Quando usar |
|---|---|---|---|
| `release:sprint` | sobe o **major**; zera minor e patch | `v1.4.2` → `v2.0.0` | entrega da sprint |
| `release:us` | sobe o **minor**; zera patch | `v1.4.2` → `v1.5.0` | user story concluída (PR `usX` → `main`) |
| `release:fix` | sobe o **patch** | `v1.4.2` → `v1.4.3` | correção pontual |
| `release:none` | **não publica** release | — | mudança sem efeito para o cliente (ajuste de CI, documentação) |

Sem nenhuma tag `vX.Y.Z` no repositório, a conta parte de `v0.0.0`: o primeiro
`release:us` gera `v0.1.0`, e o primeiro `release:sprint` gera `v1.0.0`.

A coluna "Quando usar" é uma sugestão a partir dos nomes (a wiki trata a US como a
unidade de entrega); o que o script impõe é só o efeito na versão. Zero labels ou mais de
um faz o PR **falhar** no check `Release label`.

As tags contam **por repositório**: backend e frontend têm numeração independente.
`v1.2.0` do backend não corresponde a `v1.2.0` do frontend.

---

## O fluxo, do PR à release

```
PR para a main (aberto, reaberto, novo push, label posto ou tirado)
  └── workflow "Release label"  →  check `Release label`
        ├── 0 ou 2+ labels release:*   ✗ falha; o bot comenta o motivo
        ├── release:none               ✓ comenta "não cria release"
        └── sprint / us / fix          ✓ comenta "publica vX.Y.Z"   (número provisório)

merge na main
  └── push na main dispara o CI ("Backend CI" / "Frontend CI")
        └── CI verde → workflow "Backend Release" / "Frontend Release"
              ├── o commit já tem tag v*?           → não faz nada
              ├── não há PR mergeado para o commit? → não faz nada
              ├── label release:none?               → não faz nada
              └── senão: recalcula a versão, builda, zipa e cria a GitHub Release
```

**O número do comentário é provisório.** Ele é calculado a partir da última tag e só fica
definitivo no merge. Dois PRs pendentes com `release:us` mostram a mesma versão prevista;
quem entra primeiro leva o número, e o segundo recebe o seguinte. As releases são
serializadas (`concurrency: backend-release` / `frontend-release`, sem cancelar).

**Só publica depois do CI verde na `main`.** Se o CI da `main` falhar, não há release
daquele merge; a mudança só sai publicada na release do próximo PR que entrar.

---

## O que cada release contém

A release é criada com `gh release create`, com notas geradas automaticamente pelo
GitHub (`--generate-notes`), apontando para o commit exato do merge.

| Repositório | Arquivos anexados | Conteúdo |
|---|---|---|
| `API5-Backend` | `ratio-api-vX.Y.Z-win-x64.zip` e `.zip.sha256` | API publicada **self-contained** para Windows x64, com `Version=X.Y.Z` embutida no binário |
| `API5-Frontend` | `ratio-web-vX.Y.Z.zip` e `.zip.sha256` | conteúdo de `ratio/apps/web/dist` (assets de produção) |

O `.sha256` permite ao cliente conferir a integridade do zip antes de instalar.

> ⚠ **A release ainda não é o pacote de versão.** O
> [pacote entregue ao cliente](../06-operacao/04-implantacao-no-cliente.md#documentos-que-precisam-ser-escritos)
> também leva `nginx.conf`, o dump do `dw`, scripts de instalação e o manual. Hoje a
> release publica só o zip da API **ou** o do frontend. Juntar tudo num pacote está
> pendente.

---

## Como os workflows são configurados

Os três arquivos existem nos dois repositórios, com o mesmo padrão (em `.github/`):

| Arquivo | Workflow | Job / check | Dispara em | Permissões |
|---|---|---|---|---|
| `workflows/ci.yml` | `Backend CI` / `Frontend CI` | `Backend checks` / `Frontend checks` | PR e push em `main` e `us*` | leitura |
| `workflows/release-label.yml` | `Release label` | `Release label` | PR para `main` | leitura, e escrita em PR (para comentar) |
| `workflows/release.yml` | `Backend Release` / `Frontend Release` | `Publish release` | fim do CI em push na `main` | escrita em conteúdo (tag e release) |
| `scripts/release-version.sh` | — | — | chamado pelos dois workflows acima | — |

O `release-label.yml` e o `release-version.sh` são **idênticos** nos dois repositórios.
Mudar a regra de versão exige mudar nos dois.

O script lê os labels, exige um único `release:*` e calcula a próxima versão a partir da
última tag que seja exatamente `vX.Y.Z` (tags fora desse formato são ignoradas).

---

## Pontos de atenção

- **Os labels precisam existir em cada repositório** (`release:sprint`, `release:us`,
  `release:fix`, `release:none`). Isto não foi verificado; sem eles, não há o que marcar
  no PR e o check `Release label` falha.
- **`Release label` só roda em PR para a `main`.** O ruleset `us* rules` não o exige, então
  PR de task → `usX` não precisa de label `release:*`.
- **O check é obrigatório para o merge na `main`.** O ruleset `main rules` exige os dois:
  o CI do repositório e o `Release label`. PR sem label não é mergeado.
- **Não há release sem PR.** O workflow procura o PR mergeado do commit; como a `main`
  só aceita merge por PR, isso é o esperado.
- **Um commit que já tem tag `v*` não gera outra release.**
