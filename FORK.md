# FORK.md — a automação deste fork

Este repositório (`techigorsilva-hub/DeskcommCRM`) é um fork de
[`melgarafael/DeskcommCRM`](https://github.com/melgarafael/DeskcommCRM). Ele acompanha o original
**versão por versão** e publica as próprias versões (tag, imagens no GHCR, release) para a VPS
instalar.

Tudo que é deste fork mora em arquivos que o original nunca toca, e por isso nunca entram em
conflito quando uma versão nova chega:

| Arquivo | O que faz |
|---|---|
| `.fork/config.sh` | **O único arquivo com dados próprios**: repositório original, este fork, comandos do projeto |
| `.fork/comum.sh` | Funções compartilhadas (buscar o original, versões, changelog, arquivos do fork, próxima tag) |
| `.fork/sincronizar.sh` | Confere se o original lançou versão nova; abre a tarefa (Issue) e, se der, o PR |
| `.fork/publicar.sh` | Cria a tag, espera o build das imagens e cria a release |
| `.fork/versao-original` | A versão do original já incorporada (ex.: `v1.86.0`) |
| `.github/workflows/fork-sincronizar.yml` | Roda o `sincronizar.sh` toda segunda-feira |
| `.github/workflows/fork-publicar.yml` | Roda o `publicar.sh` quando `.fork/versao-original` muda na `main` |
| `FORK.md` | Este arquivo |

Os workflows declaram `jobs: # fork`. O comentário é deliberado: os testes de governança do
original (`tests/unit/gatilho-dos-jobs-de-entrega.test.ts` e irmãos) mapeiam os jobs dos
workflows dele e reprovam job desconhecido; os do fork ficam fora dessa varredura, sem que nenhum
arquivo do original precise ser editado.

Fora de `.fork/`, o fork mudou de propósito só o endereço das imagens — de `ghcr.io/melgarafael`
para `ghcr.io/techigorsilva-hub` — em `hostgator-setup-kit/_common.sh` (`IMG_NS`),
`docker-compose.prod.yml` e `.env.hostgator.example`. Esses arquivos podem receber mudança do
original e aparecer como "ponto de atenção" na tarefa semanal.

---

## O ciclo semanal

1. **Segunda-feira, 08:17 de Brasília** (`cron: "17 11 * * 1"`), o `fork-sincronizar` pergunta ao
   original qual é a última versão **lançada** (`releases/latest`, a mesma pergunta que o
   `update.sh` faz — tag sem release não conta). Para rodar fora de hora: aba Actions →
   *Fork — sincronizar com o original* → *Run workflow*.
2. Se for igual a `.fork/versao-original`, termina: "Nada novo".
3. Se houver versão nova, abre uma **Issue** com o rótulo `sincronizacao` contendo:
   - as versões novas e o trecho do `CHANGELOG.md` do original;
   - os **pontos de atenção**: arquivos que o fork modificou e o original também mudou;
   - o que o original marca como *Requer atenção* para quem opera a VPS;
   - colisão de número de migration, se houver (ver abaixo).

   A Issue é aberta por `github-actions[bot]` (não pela chave do dono) para que a notificação
   chegue ao dono — o GitHub não avisa ninguém do que a própria pessoa fez.
4. Faz o merge da tag do original numa branch à parte, `sincronizar/vX.Y.Z`. **Nunca mexe na
   `main`.**
   - **Junção limpa:** grava a versão nova em `.fork/versao-original`, empurra a branch e abre o
     PR (que fecha a Issue ao entrar).
   - **Conflito:** a Issue lista os arquivos em conflito e traz um pedido pronto para colar no
     Claude resolver.
5. Uma tarefa por vez: uma versão mais nova fecha a Issue e o PR antigos (já estão incluídos).
6. Você espera os testes do PR (~30 min) e aprova o merge.
7. O merge muda `.fork/versao-original` na `main`, o que dispara o `fork-publicar`:
   tag → build das imagens (15–25 min) → release. **A release só nasce depois das imagens**, então
   a VPS nunca enxerga uma versão sem imagem.
8. Atualize a VPS: `bash hostgator-setup-kit/update.sh` (ou pela tela: menu → rodapé →
   "Nova versão").

Modo seco, para ver o que aconteceria sem criar nada:

```bash
SINCRONIZAR_SECO=1 bash .fork/sincronizar.sh
```

```bash
PUBLICAR_SECO=1 bash .fork/publicar.sh
```

---

## As versões

O fork usa **os mesmos números do original**, com tags próprias:

- **Versão do original incorporada** → tag com o mesmo número. Ex.: o original lança `v1.87.0`,
  o fork publica `v1.87.0` (o original com as modificações do fork).
- **Correção do fork entre duas versões do original** → o patch seguinte livre. Ex.: `v1.87.1`,
  depois `v1.87.2`. Para publicar: aba Actions → *Fork — publicar versão* → *Run workflow*.
- Se o original depois lançar o número que o fork já usou (ex.: `v1.87.1`), o fork pula para o
  próximo livre (`proxima_tag_livre` em `comum.sh`).

As tags do original ficam num espaço separado, `refs/upstream-tags/*`, **nunca** em `refs/tags/*`:
senão `v1.87.0` significaria duas coisas no mesmo repositório.

Na primeira sincronização, `preparar_historico_do_fork` copia para o fork as tags do original até a
versão incorporada (os mesmos commits, sem mudança) e cria a release inicial
`vX.Y.Z (original, sem modificações)`. Parte dos testes do original
(`scripts/conferir-isolamento-do-kit.sh`) compara o kit com a última release e com tags antigas, e
precisa desse histórico.

**O `publicar.sh` recusa publicar** quando a `main` contém commits do original que ainda não saíram
em versão lançada. O fork só entrega versões do original, nunca o meio do caminho entre duas. Se
acontecer: espere a próxima sincronização, que traz a versão que contém esses commits.

---

## Migrations do fork: faixa 9001+

O original numera as migrations em sequência (`<timestamp>_<NNNN>_<slug>.sql`) e o próximo número
dele é imprevisível para nós. Por isso **toda migration criada neste fork usa `NNNN` a partir de
`9001`** (`9001`, `9002`, …), longe da faixa do original:

```text
supabase/migrations/20261021120000_9001_meu_ajuste.sql
```

Para ver o próximo número livre do fork:

```bash
ls supabase/migrations | sed -nE 's#^[0-9]{14}_(9[0-9]{3})_.*#\1#p' | sort | tail -1
```

O resto da doutrina de migrations do `CLAUDE.md` vale igual: linha `-- manifest:` no cabeçalho,
apêndice idempotente no `supabase/baseline.sql`, `revoke` nas funções novas.

Mesmo assim, o `sincronizar.sh` compara os números das migrations novas do original com os das
migrations do fork e, se coincidirem, a Issue traz a seção **🛑 Colisão de número de migration**:
renumere a sua e revise o apêndice correspondente no `baseline.sql` antes do merge.

---

## A chave FORK_TOKEN

Os dois workflows param no primeiro passo sem o segredo `FORK_TOKEN`. O token padrão do Actions
(`GITHUB_TOKEN`) não serve porque:

- o GitHub **proíbe** que ele envie commits que mexem em `.github/workflows/` — e as versões do
  original mexem;
- uma tag empurrada por ele **não dispara** outro workflow, então o build das imagens não
  começaria.

Como criar:

1. GitHub → Settings → Developer settings → **Fine-grained personal access tokens** → *Generate
   new token*.
2. *Repository access*: só `techigorsilva-hub/DeskcommCRM`.
3. *Permissions* (Repository): **Contents**, **Pull requests**, **Issues** e **Workflows** como
   *Read and write*; **Actions** como *Read and write* (para disparar e acompanhar o build das
   imagens).
4. No repositório: Settings → Secrets and variables → Actions → *New repository secret*, nome
   `FORK_TOKEN`, valor = o token.

Quando a chave vencer, os workflows falham no passo *Conferir a chave FORK_TOKEN* com a mensagem
que aponta para esta seção — gere outra e substitua o segredo.

---

## Trocar a VPS do original para o fork

A VPS decide o que instalar por duas coisas: o `origin` do clone (de onde vêm as tags e a
`releases/latest`) e o endereço das imagens (`IMG_NS` em `hostgator-setup-kit/_common.sh`). Depois
da troca, as duas apontam para o fork.

Pré-requisitos:

- o fork já tem pelo menos uma release publicada **com imagens** (a primeira sincronização com o
  original publica a primeira);
- os pacotes no GHCR (`deskcommcrm`, `deskcomm-worker`, `deskcomm-scheduler`,
  `deskcomm-voice-agent`) estão **públicos** — a VPS baixa sem login. Confira em
  github.com/techigorsilva-hub?tab=packages → pacote → *Package settings* → *Change visibility*.

Na VPS, dentro da pasta do CRM:

```bash
git remote set-url origin https://github.com/techigorsilva-hub/DeskcommCRM.git
```

```bash
bash hostgator-setup-kit/update.sh
```

O `update.sh` faz backup, busca as tags do fork, troca para a última release dele e **relê o
`_common.sh` depois do checkout** — então já usa o `IMG_NS` do fork e regrava `APP_IMAGE`,
`WORKER_IMAGE` e `SCHEDULER_IMAGE` no `.env` com `ghcr.io/techigorsilva-hub/...:<versão>`. Não é
preciso editar o `.env` à mão.

Para voltar ao original: o mesmo `git remote set-url` com
`https://github.com/melgarafael/DeskcommCRM.git` e o mesmo `update.sh`. A versão do original
precisa ser igual ou mais nova que a do fork instalada.

Depois de qualquer troca, confira: `/` responde **200** e `/app` responde **307**
(ver `docs/runbooks/deploy.md`).
