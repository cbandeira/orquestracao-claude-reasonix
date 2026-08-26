# Loop manual Claude Code ⇄ Reasonix — passo a passo

O `ai-flow` continua existindo e continua funcionando. O que muda aqui é quem
dirige: os modelos rodam nas GUIs, na sua frente, e o `aif` faz o papel de
cartório — git, validação e commit. Nada roda escondido em `subprocess`.

---

## O que tem neste pacote

| Arquivo | Onde vai | Para que serve |
|---|---|---|
| `aif` | `~/.local/bin/aif` | o cartório: worktree, semáforo, validação, commit |
| `.claude/commands/planejar.md` | raiz do projeto | slash command `/planejar` |
| `.claude/commands/revisar.md` | raiz do projeto | slash command `/revisar` |
| `.claude/settings.json` | raiz do projeto | statusline com tokens + notificações — **mesclar, não sobrescrever** |
| `.ai/implementer.md` | raiz do projeto | contrato permanente do worker |
| `REASONIX.md` | raiz do projeto | instruções que o Reasonix carrega sozinho |

---

## 1. Instalação, uma vez por máquina

```bash
mkdir -p ~/.local/bin
cp aif ~/.local/bin/aif
chmod +x ~/.local/bin/aif
aif --help
```

Se `aif` não for encontrado, falta `~/.local/bin` no `PATH`.

Dependências: `git`, `bash`, `python3`, `iconv`. O `jq` só é necessário para a
statusline do Claude Code, e o `notify-send` (pacote `libnotify-bin`) só para
as notificações.

## 2. Instalação, uma vez por projeto

```bash
aif install /caminho/do/seu/projeto
```

Rode da raiz deste pacote — ou de qualquer lugar, passando `--from <raiz do
pacote>`. Ele copia os cinco arquivos e **nunca sobrescreve**: o que já existe
no destino fica como está, e ele diz na tela o que pulou.

O `.claude/settings.json` tem tratamento à parte, porque é o único do pacote
que costuma disputar espaço com algo que já existe. Um projeto que já usa
Claude Code guarda as permissões dele nesse arquivo, e copiar por cima apaga
essas permissões em silêncio — você só descobre quando o Claude Code volta a
pedir confirmação para tudo. Se não há conflito, o `install` copia e segue. Se
a statusline e os hooks já estiverem instalados — no `settings.json` ou no
`settings.local.json`, tanto faz, porque o Claude Code soma os dois —, ele diz
onde estão e não mexe em nada. Só quando sobra o que instalar, e o arquivo já
existe, é que ele para e pergunta:

- **1 — `.claude/settings.local.json`**, que não é versionado. É o padrão, e
  evita impor as suas notificações a quem mais mexa no repositório. Se o
  `.gitignore` ainda não ignora esse arquivo, o `install` acrescenta a linha.
- **2 — mesclar no `settings.json` que já existe.** A mesclagem é aditiva e só:
  não remove chave, não troca valor que já está lá, e rodar duas vezes não
  duplica hook.
- **3 — pular**, e você resolve à mão.

Essa checagem é por conteúdo, não por arquivo existir. É o que evita o caso
chato: instalar de novo escolhendo o outro arquivo não sobrescreveria nada —
duplicaria, e cada parada do Claude Code viraria duas notificações.

Depois **ajuste duas coisas**:

- em `REASONIX.md`, o comando de testes do projeto. É a última seção, e o
  `pytest -q` que vem lá é só um exemplo. O comando precisa rodar da raiz da
  worktree sem `cd` antes, terminar sozinho (nada de watch mode) e devolver o
  código de saída certo — é por ele que o worker sabe se pode reportar
  `PRONTO`. Não aponte para uma suíte que exige banco ou container de pé: ela
  falha por falta de infra e o worker reporta `BLOQUEADO` sem que nada esteja
  quebrado.
- em `.ai/implementer.md`, nada — ele é genérico de propósito. Regra permanente
  do projeto vai no `.ai/decisions.md`, que os três papéis leem; o
  `implementer.md` só muda se a regra for sobre como o *worker* se comporta.

Por fim, **commite**. Não é zelo, é requisito: o `aif open` monta a worktree com
`git worktree add`, então ela contém só o que está commitado na branch base. O
`.ai/implementer.md` e o `.ai/decisions.md` o `aif` copia à mão para dentro da
worktree, e por isso sobrevivem sem commit; o `REASONIX.md` e o
`.claude/commands/` não — sem commit, o Reasonix abre a worktree sem as
instruções do projeto e o `/planejar` nem aparece no Claude Code.

```bash
cd /caminho/do/seu/projeto && git add REASONIX.md .ai .claude && git status --short
```

Leia esse `git status` antes de commitar. Tudo que veio do pacote é arquivo
novo, e arquivo novo aparece como `A`. Um `M` ali significa que algo do projeto
mudou — o `install` não faz isso sozinho, mas a sua mesclagem manual pode ter
feito. Desfaça com `git checkout HEAD -- <arquivo>` se não for o que você
queria.

Se o projeto já tem `.ai/decisions.md` do `ai-flow`, ele é aproveitado: o `aif`
copia esse arquivo para dentro de cada worktree, e os três papéis o leem.

## 3. Preparando as duas janelas

A ideia é não ter que caçar janela. Duas opções:

**A — tudo num VS Code só (recomendado).** Instale a extensão do Claude Code e
a extensão do Reasonix (`SivanLiu.reasonix-agent`, que sobe o backend
`reasonix acp` local — a CLI precisa estar instalada antes). Abra a worktree
como pasta e deixe os dois painéis lado a lado.

**B — dois apps desktop.** Claude Code no app desktop e o Reasonix no app dele.
Aí o alt-tab volta, mas as notificações do passo 8 avisam quando é a hora.

## 4. Abrindo uma tarefa

Na raiz do repositório (não na worktree):

```bash
aif open "Adicionar rate limiting no endpoint de login"
```

Isso cria a branch `ai/adicionar-rate-limiting-no-endpoint-de-login`, a worktree
em `../.ai-flow-worktrees/<slug>/`, e imprime o caminho. Mesma convenção do
`ai-flow` — os dois convivem sem brigar, desde que você não tenha tarefa ativa
nos dois ao mesmo tempo.

```bash
cd "$(aif cd)"
```

Abra os dois agentes **nessa pasta**. Isso importa: é a worktree que isola a
tarefa, e é lá que os artefatos vivem.

## 5. Planejar (Claude Code)

No Claude Code, dentro da worktree:

```
/planejar Adicionar rate limiting no endpoint de login
```

Ele lê o código, escreve `.ai/current-task.md` e termina imprimindo um bloco
`/goal ...` pronto. **Leia o plano antes de seguir** — é o momento mais barato
de corrigir rumo. Editar o `current-task.md` à mão é uso previsto; o validador
só exige que as seções `## Plan` e `## Acceptance criteria` continuem lá, com
esses nomes em inglês.

Confira quando quiser:

```bash
aif status
```

## 6. Implementar (Reasonix)

Copie o bloco que o Claude Code imprimiu e cole no Reasonix. É um *task
contract* no formato que ele espera: contexto, request, formato de saída,
restrições e política de pausa.

Deixe rodando. Ele lê `REASONIX.md` → `.ai/implementer.md` → `.ai/current-task.md`,
implementa, roda os testes e encerra com:

```
PRONTO → peça ao Claude Code revisar.
```

Se aparecer `CONFLITO DE PLANO`, não force: volte ao Claude Code e replaneje.
É o worker dizendo que o mapa não bate com o território.

## 7. Revisar (Claude Code)

De volta ao Claude Code, na mesma sessão:

```
/revisar
```

Ele lê o diff contra o plano, escreve `.ai/review.json` e termina de um dos
dois jeitos:

- **REPROVADO** — imprime um bloco `/goal` de correção. Cola no Reasonix, volta
  ao passo 6. O worker vai tratar só o que é `CRITICAL`, `HIGH` e `MEDIUM`.
- **APROVADO** — manda você rodar `aif accept`.

## 8. Commitar (você)

```bash
aif accept
```

O `aif` revalida o `review.json` do zero — schema, campos obrigatórios,
severidades — e aplica a regra que o `ai-flow` já aplicava: **um veredito
`approved` que lista uma questão bloqueante vale como `changes_required`**.
Só depois disso ele commita, na branch da tarefa. Sem push, sem merge, sem
tocar na base.

Por que não deixar o Claude Code commitar sozinho? Porque foi ele que escreveu
o veredito. Um portão que o próprio revisado abre não é um portão — é
decoração. `aif accept` é uma tecla; a independência vale a tecla.

Depois:

```bash
git diff main...ai/<slug>       # confira você mesmo
git worktree remove "<caminho>" # limpa quando terminar
```

---

## Vendo tokens e sendo avisado

O `.claude/settings.json` do pacote faz duas coisas:

**Statusline** com modelo, projeto, custo acumulado e tokens da sessão — algo
como `[Opus] tap-list | $0.42 | 53k tok`. Os nomes dos campos que a statusline
recebe podem mudar entre versões; se a linha vier vazia, confira a referência
em `code.claude.com/docs/en/hooks`. Para consulta pontual, `/context` e `/cost`
resolvem sem configurar nada.

**Hooks `Stop` e `Notification`** disparam `notify-send` quando o Claude termina
ou quando precisa de você. Os hooks disparam igual no terminal, nas extensões de
IDE, no app desktop e na web — então funciona em qualquer das duas montagens do
passo 3. Em macOS, troque `notify-send` por `osascript -e 'display notification …'`
ou um `afplay`.

No lado do Reasonix, o app desktop já mostra o loop de ferramentas, as
aprovações e os checkpoints por turno. Como você vai estar olhando, use o modo
de aprovação interativo em vez de `--permission-mode auto`: leitura, escrita e
shell pedem separadamente, e o sandbox da workspace ainda limita o alcance.
É uma postura melhor do que a do modo automático, não pior.

---

## O semáforo, para quando você perder o fio

```bash
aif status
```

| Estado | Próximo passo |
|---|---|
| plano ausente ou inválido | Claude Code: `/planejar` |
| plano ok, sem mudanças de código | Reasonix: cole o bloco `/goal` |
| mudanças de código, sem revisão | Claude Code: `/revisar` |
| revisão exige mudanças | Reasonix: cole o bloco de correção |
| revisão aprovada | `aif accept` |

Comandos completos: `install`, `open`, `cd`, `status`, `verify`, `accept`, `drop`.

---

## Se você quiser trazer isso para dentro do próprio ai-flow

Hoje o `ai-flow` não serve como cartório deste loop por um motivo específico:
`step_review()` chama `artifacts.clear_review()` **antes** de invocar o revisor,
então ele apaga o `review.json` que a GUI acabou de escrever. O `step_plan()`
já faz o certo — checa `has_valid_plan()` e reaproveita o plano existente sem
chamar o planejador.

A mudança é simétrica e pequena: um `has_valid_review()` em `ArtifactStore` e
um curto-circuito no começo do `step_review()`, igual ao do plano. Feito isso,
`ai-flow resume` passa a fechar a tarefa lendo o que as GUIs produziram, e o
`aif` vira redundante.

É, aliás, a primeira tarefa perfeita para rodar neste loop.
