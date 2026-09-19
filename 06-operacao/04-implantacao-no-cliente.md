# Implantação no cliente

> **Produção é a intranet do cliente.** O Ratio roda nos servidores do próprio cliente,
> em **Windows Server**, e é usado **exclusivamente pelos funcionários dele** — não é um
> site público. Ver [D-21](02-decisoes-e-riscos.md#d-21--produção-na-intranet-do-cliente-em-windows-server).
>
> Esta página registra **o que a implantação exige** e **os documentos que precisam ser
> escritos**. Ela não é o manual de implantação.

---

## As premissas

| Premissa | Consequência no projeto |
|---|---|
| Roda na **intranet do cliente** | nada depende de internet no servidor: fontes self-hosted, sem CDN, sem chamada externa em tempo de execução |
| Servidores **Windows Server** | todo artefato entregue tem de rodar em Windows — API, frontend, banco e proxy |
| **Nós damos a spec** das máquinas | a especificação é um entregável nosso, não uma pergunta ao cliente |
| O cliente recebe **só arquivos buildados** | nenhum SDK, Node, npm, Python, Git ou IDE nas máquinas dele |
| Uso **só por funcionários** | **restrição de rede, sem login** — quem está na intranet usa ([abaixo](#acesso-só-de-funcionários)) |
| Proxy reverso: **NGINX**, não IIS | ver [D-22](02-decisoes-e-riscos.md#d-22--nginx-como-proxy-reverso-no-lugar-do-iis) |

---

## Desenho no servidor do cliente

```
        estação do funcionário (navegador, rede interna)
                          │  https://ratio.<intranet-do-cliente>
                          ▼
   ┌─ Windows Server ──────────────────────────────────────────┐
   │                                                            │
   │  NGINX  (serviço Windows)              porta 443           │
   │   ├── /        → arquivos estáticos do frontend (dist/)    │
   │   └── /api/    → proxy para a API em 127.0.0.1:5000        │
   │                                                            │
   │  Ratio.Api  (serviço Windows, self-contained)  127.0.0.1   │
   │                          │                                 │
   │  PostgreSQL 16 + pgvector   (serviço Windows)  127.0.0.1   │
   │                                                            │
   └────────────────────────────────────────────────────────────┘
                          ▲
                          │  dump do schema dw, entregue junto com a versão
                 carga manual, feita por NÓS, fora do cliente
```

Pontos que o desenho fixa:

- **Uma única origem.** O NGINX serve o frontend e repassa `/api/` para a API. O
  navegador só vê um endereço — **não há CORS** em produção, e o frontend chama `/api`
  por caminho relativo.
- **Só o NGINX escuta na rede.** API e banco escutam em `127.0.0.1`; ninguém na
  intranet fala com eles direto.
- **A carga não acontece no cliente.** A [carga manual](../02-arquitetura/05-etl-e-nlp.md#carga-manual--o-processo)
  roda do nosso lado; o cliente recebe o dump do schema `dw` já validado e o restaura.
  O servidor do cliente nunca fala com DataJud, DOAJ ou qualquer fonte.
- Se banco e aplicação ficam na **mesma máquina ou em duas** é decisão da
  [especificação](#documentos-que-precisam-ser-escritos).

---

## O que precisa mudar no projeto por causa disso

### Backend

| Item | Por quê |
|---|---|
| Publicar **self-contained** para `win-x64` (`dotnet publish -r win-x64 --self-contained`) | o cliente não instala runtime .NET — o executável leva o runtime junto |
| Rodar como **serviço Windows** (`Microsoft.Extensions.Hosting.WindowsServices`, `UseWindowsService()`) | sobe com a máquina e reinicia sozinho, sem ninguém logado |
| Escutar só em `127.0.0.1` | o acesso é pelo NGINX |
| Honrar `X-Forwarded-*` (`UseForwardedHeaders`) | atrás de proxy, para log e esquema (`https`) corretos |
| Configuração por `appsettings.Production.json` **fora do pacote** ou variável de ambiente do serviço | connection string e caminhos são do cliente, não do build |
| Log em **arquivo** com rotação (Serilog `File`) | no servidor do cliente não há console para ler log |
| CORS: desnecessário em produção | origem única; manter lista explícita só para dev |

### Frontend

| Item | Por quê |
|---|---|
| Chamar a API por caminho **relativo** (`/api/...`) | o endereço da intranet do cliente não é conhecido no build; com `VITE_API_URL` fixo, cada cliente exigiria um build próprio |
| Fontes **self-hosted** | ✅ já é assim (Fontsource) — sem internet, Google Fonts não carregaria |
| Nenhum recurso de CDN | mesma razão |
| Roteamento do lado do cliente | o NGINX precisa de `try_files … /index.html` para `/topics/123` não dar 404 ao recarregar |

> **Links para os tribunais saem da intranet.** O "consultar no tribunal" abre o e-SAJ /
> portal do tribunal no navegador do funcionário. Funciona se a estação tiver acesso à
> internet; se a rede do cliente bloquear, o link não abre. Confirmar com o cliente.

### Banco

| Item | Por quê |
|---|---|
| **PostgreSQL 16 nativo para Windows** como serviço | sem Docker nas máquinas do cliente |
| **pgvector compilado para Windows** | ⚠ não vem no instalador padrão — ver [R-16](02-decisoes-e-riscos.md#r-16--pgvector-e-locale-no-postgres-para-windows-) |
| Locale **ICU `pt-BR`** na criação do cluster (`initdb --locale-provider=icu --icu-locale=pt-BR`) | mesma ordenação com acento que temos hoje |
| `unaccent` e `pg_trgm` | vêm no `contrib` do instalador Windows |
| Papel `ratio_api` só com `SELECT` | a API não escreve ([Data Warehouse](../02-arquitetura/04-data-warehouse.md#papéis--o-que-falta-para-produção)) |

### Proxy — NGINX

| Item | Por quê |
|---|---|
| NGINX para Windows como **serviço** (via WinSW ou NSSM) | o NGINX não se registra como serviço Windows sozinho |
| TLS com o **certificado interno** do cliente | é intranet: a CA é a do cliente, não Let's Encrypt |
| `client_max_body_size`, timeouts e cabeçalhos de segurança definidos | configuração entregue pronta, não "a ajustar" |

---

## Acesso só de funcionários

**Decidido: restrição de rede, sem login** ([D-23](02-decisoes-e-riscos.md#d-23--acesso-por-restrição-de-rede-sem-login)).
Quem alcança o servidor pela intranet do cliente usa o sistema; não há tela de login,
sessão, usuário nem integração com Active Directory.

O que isso implica:

- **A segurança de acesso é da rede do cliente.** O Ratio não controla quem entra; o
  manual de implantação precisa dizer isso com todas as letras, para a TI do cliente
  não publicar o servidor para fora da intranet.
- **Só a porta do NGINX fica exposta.** API e banco em `127.0.0.1` continuam
  obrigatórios — sem login, nenhuma outra porta pode ser alcançável.
- **A troca de IIS por NGINX é neutra.** Como não há autenticação Windows integrada a
  preservar, o NGINX substitui o IIS sem perda.
- **Não há dado de usuário.** Nada de LGPD sobre quem consulta — o sistema não sabe.
  (Se o [chatbot](../01-produto/05-chatbot.md) entrar, a pergunta digitada volta a ser
  dado a tratar.)

---

## O que deixa de valer

| Antes (VPS Hostinger + Coolify — descartada) | Agora (intranet do cliente) |
|---|---|
| deploy automático no merge da `main` | o CI **gera o pacote de versão**; quem instala é o cliente, seguindo o manual |
| contêineres Docker | serviços Windows |
| TLS automático do Coolify | certificado interno do cliente, configurado no NGINX |
| variáveis injetadas pelo Coolify | arquivo de configuração / variável do serviço, na máquina do cliente |
| monitoramento nosso | o cliente opera; nós entregamos health check e log em arquivo |

> ⚠ **Conflito com o desafio.** O desafio cobra "deploy automático". Com a produção na
> mão do cliente, o deploy automático só pode existir no ambiente **nosso** de
> homologação — que simula a intranet numa rede Tailscale
> ([D-24](02-decisoes-e-riscos.md#d-24--homologação-simulada-numa-rede-tailscale-sem-vps)).
> Ver [R-17](02-decisoes-e-riscos.md#r-17--deploy-automático-exigido-pelo-desafio-x-produção-no-cliente-).

---

## Documentos que precisam ser escritos

Registrados aqui para não se perderem. **Nenhum existe ainda.**

| Documento | Para quem | O que precisa ter |
|---|---|---|
| **Manual de implantação** | equipe de TI do cliente | como instalar e configurar, a partir **só dos arquivos buildados**: PostgreSQL + pgvector, restauração do dump, serviço da API, NGINX + certificado, verificação pelo `/health/ready`; sem nenhuma ferramenta de desenvolvimento |
| **Manual de atualização** | TI do cliente | como aplicar uma versão nova e **uma carga nova** (restore do dump) sem perder a anterior; como voltar atrás |
| **Especificação das máquinas** | cliente | versão do Windows Server, CPU, RAM, disco, se banco e aplicação ficam juntos ou separados, portas, certificado, contas de serviço |
| **Conteúdo do pacote de versão** | nós e o cliente | o que vai no zip: API publicada, `dist/` do frontend, `nginx.conf`, dump do `dw`, scripts de instalação, versão e data da carga |

O pacote e os manuais andam juntos: **toda versão entregue leva o manual da própria
versão.**
