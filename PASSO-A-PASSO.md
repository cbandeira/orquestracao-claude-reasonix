# Loop manual orquestrador ⇄ Reasonix — passo a passo

O `ai-flow` continua existindo e continua funcionando. O que muda aqui é quem
dirige: os modelos rodam nas GUIs, na sua frente, e o `aif` faz o papel de
cartório — git, validação, commit e integração. Nada roda escondido em
`subprocess`.

São três papéis e dois agentes. O **orquestrador** planeja e revisa; o
**worker** implementa; você commita e integra. O orquestrador pode ser o Claude
Code ou o OpenCode — os dois leem o mesmo prompt, cada um do seu diretório —, e
o worker aqui é o Reasonix.

---

## O que tem neste pacote

| Arquivo | Onde vai | Para que serve |
|---|---|---|
| `aif` | `~/.local/bin/aif` | o cartório: worktree, semáforo, validação, commit, merge |
| `.claude/commands/planejar.md` | raiz do projeto | o prompt do papel de planejador |
| `.claude/commands/revisar.md` | raiz do projeto | o prompt do papel de revisor |
| `.opencode/commands/*.md` | raiz do projeto | symlinks para os dois de cima |
| `.opencode/agents/planejador.md` | raiz do projeto | as travas de ferramenta do planejador no OpenCode |
| `.opencode/agents/revisor.md` | raiz do projeto | idem, para o revisor |
| `.claude/settings.json` | raiz do projeto | statusline com tokens + notificações — **mesclar, não sobrescrever** |
| `.ai/implementer.md` | raiz do projeto | contrato permanente do worker |
| `REASONIX.md` | raiz do projeto | instruções que o Reasonix carrega sozinho |

Os arquivos de comando são **um só, em dois lugares**. O Claude Code lê
`.claude/commands/`, o OpenCode lê `.opencode/commands/`, e nenhum lê o do
outro — mas o frontmatter carrega os campos das duas ferramentas e cada uma
ignora o que não conhece, então o corpo do prompt não precisa ser duplicado.
Você edita `.claude/commands/planejar.md` e as duas veem a mudança.

Os `agents/` existem porque o OpenCode não tem o campo `allowed-tools` do
Claude Code: lá a restrição de ferramenta mora no agente, e o comando aponta
para ele pelo campo `agent:`. O efeito é o mesmo — o planejador só escreve
`.ai/current-task.md`, o revisor só escreve `.ai/review.json`, e o `bash` de
ambos nega tudo menos os comandos de leitura.

---

## 1. Instalação, uma vez por máquina

```bash
mkdir -p ~/.local/bin
cp aif ~/.local/bin/aif
chmod +x ~/.local/bin/aif
aif --help
```

Se `aif` não for encontrado, falta `~/.local/bin` no `PATH`.

Dependências: `git`, `bash` e `python3`. O `jq` só é necessário para a
statusline do Claude Code, e as notificações usam o que existir — `notify-send`
(pacote `libnotify-bin`) no Linux, `osascript` no macOS.

**Linux e macOS.** O `aif` roda nos dois. Ele não usa nada de bash 4+, então o
bash 3.2 que vem no macOS basta, e evita de propósito as ferramentas cujo
comportamento diverge entre GNU e BSD: nada de `sed -i`, `readlink -f` ou
`iconv //TRANSLIT`. Onde a divergência apareceria — a normalização do slug da
branch —, o trabalho é feito em `python3`, que dá o mesmo resultado nos dois
sistemas e em qualquer locale.

Duas coisas no macOS não vêm de fábrica e você instala à parte: o `python3`
(vem com as Command Line Tools do Xcode) e o `jq`, se quiser a statusline.

## 2. Instalação, uma vez por projeto

```bash
aif install /caminho/do/seu/projeto
```

Rode da raiz deste pacote — ou de qualquer lugar, passando `--from <raiz do
pacote>`. Ele instala os arquivos da tabela acima e **nunca sobrescreve**: o
que já existe no destino fica como está, e ele diz na tela o que pulou. Os dois
`.opencode/commands/` ele cria como symlink relativo; se o sistema de arquivos
não aceitar symlink, ele cai para cópia e avisa.

O `.opencode/` é instalado mesmo que você só use o Claude Code. São quatro
arquivos pequenos e inertes: ferramenta que não lê aquele diretório não é
afetada por ele. Isso é de propósito — trocar de orquestrador depois não deve
exigir reinstalar nada.

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
worktree, e por isso sobrevivem sem commit; o `REASONIX.md`, o
`.claude/commands/` e o `.opencode/` não — sem commit, o Reasonix abre a
worktree sem as instruções do projeto e o `/planejar` nem aparece no
orquestrador.

```bash
cd /caminho/do/seu/projeto && git add REASONIX.md .ai .claude .opencode && git status --short
```

Leia esse `git status` antes de commitar. Tudo que veio do pacote é arquivo
novo, e arquivo novo aparece como `A`. Um `M` ali significa que algo do projeto
mudou — o `install` não faz isso sozinho, mas a sua mesclagem manual pode ter
feito. Desfaça com `git checkout HEAD -- <arquivo>` se não for o que você
queria.

Se você esquecer, o `aif open` avisa antes de criar a worktree e pergunta se
quer seguir assim mesmo. Ele lista o que ficou de fora e distingue os dois
casos — arquivo nunca commitado e arquivo commitado com alteração pendente.
Vale prestar atenção nesse aviso: o sintoma aparece longe da causa. A worktree
nasce sem os comandos, e o que você vê é o orquestrador abrindo sem encontrar
o `/planejar`, sem nada apontando para o commit que faltou.

Se o projeto já tem `.ai/decisions.md` do `ai-flow`, ele é aproveitado: o `aif`
copia esse arquivo para dentro de cada worktree, e os três papéis o leem.

## 3. Escolhendo o orquestrador, e preparando as duas janelas

O orquestrador é quem roda `/planejar` e `/revisar`. Claude Code e OpenCode
servem igual: leem o mesmo prompt, escrevem os mesmos `.ai/current-task.md` e
`.ai/review.json`, e o `aif` valida os dois do mesmo jeito. Nada no cartório
sabe qual você usou.

Diga ao `aif` qual é, e ele passa a citar o nome certo no semáforo:

```bash
export AIF_ORCHESTRATOR=opencode   # ou: claude (padrão)
```

Isso muda só o texto que o `aif` imprime — nenhuma validação depende disso.

Duas diferenças reais entre eles, para você escolher com os olhos abertos:

- **As travas de ferramenta.** No Claude Code elas vêm do `allowed-tools` do
  próprio comando; no OpenCode, do agente em `.opencode/agents/`. A do OpenCode
  é mais apertada num ponto: ela restringe *quais caminhos* podem ser escritos,
  então o planejador literalmente não consegue tocar em código. O
  `allowed-tools` do Claude Code concede `Write` sem restrição de destino — a
  regra "não escreva código" ali é o prompt pedindo, não o harness impedindo.
- **A statusline e as notificações** do passo 8 são do Claude Code. O OpenCode
  não lê `.claude/settings.json`; se você orquestrar por lá, perde o aviso de
  "terminou" e precisa olhar a janela.

Escolhido isso, a ideia é não ter que caçar janela. Duas montagens:

**A — tudo num VS Code só (recomendado).** Instale a extensão do orquestrador
que você escolheu e a extensão do Reasonix (`SivanLiu.reasonix-agent`, que sobe
o backend `reasonix acp` local — a CLI precisa estar instalada antes). Abra a
worktree como pasta e deixe os dois painéis lado a lado.

**B — dois apps desktop.** O orquestrador no app dele e o Reasonix no app dele.
Aí o alt-tab volta, mas as notificações do passo 8 avisam quando é a hora — se
o orquestrador for o Claude Code.

## 4. Abrindo uma tarefa

```bash
aif open "Adicionar rate limiting no endpoint de login"
```

Os comandos do `aif` funcionam de qualquer lugar do repositório, **inclusive de
dentro da worktree da tarefa** — ele sempre resolve a worktree principal, que é
onde o estado mora. As duas exceções são `land` e `drop`, que recusam rodar de
dentro da worktree que iriam remover.

Isso cria a branch `ai/adicionar-rate-limiting-no-endpoint-de-login`, a worktree
em `../.ai-flow-worktrees/<projeto>/<slug>/`, e imprime o caminho. Mesma
convenção de branch do `ai-flow` — os dois convivem sem brigar, desde que você
não tenha tarefa ativa nos dois ao mesmo tempo.

O `<projeto>` no meio do caminho não é enfeite. O diretório fica **fora** do
repositório, porque uma worktree dentro dele é uma segunda cópia do projeto
dentro de si mesmo, e tudo que varre a árvore passa a varrer as duas — um
`eslint .`, o contexto de um `docker build`, um script que inventaria
dependências. Mas ficar fora significa que projetos irmãos, sob o mesmo
diretório-pai, dividem o mesmo `.ai-flow-worktrees/`. Sem o nome do projeto no
caminho, duas tarefas com a mesma descrição em repositórios diferentes disputam
o mesmo diretório — e o `git worktree add` falha *depois* de criar a branch,
deixando uma branch órfã que faz a tentativa seguinte reclamar da coisa errada.

Se preferir outro lugar, `AIF_WORKTREE_DIR` aceita qualquer caminho, relativo à
raiz do repositório ou absoluto.

```bash
cd "$(aif cd)"
```

Abra os dois agentes **nessa pasta**. Isso importa: é a worktree que isola a
tarefa, e é lá que os artefatos vivem.

## 5. Planejar (orquestrador)

No orquestrador, dentro da worktree:

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

Copie o bloco que o orquestrador imprimiu e cole no Reasonix. É um *task
contract* no formato que ele espera: contexto, request, formato de saída,
restrições e política de pausa.

Deixe rodando. Ele lê `REASONIX.md` → `.ai/implementer.md` → `.ai/current-task.md`,
implementa, roda os testes e encerra com:

```
PRONTO → devolva ao orquestrador.
```

Se aparecer `CONFLITO DE PLANO`, não force: volte ao orquestrador e replaneje.
É o worker dizendo que o mapa não bate com o território.

## 7. Revisar (orquestrador)

De volta ao orquestrador, na mesma sessão:

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
tocar na base. Ficam de fora do commit o `.ai/current-task.md`, o
`.ai/review.json` e o `.reasonix/`: plano, veredito e estado local do app do
worker são efêmeros e por worktree — commitá-los faz a tarefa seguinte nascer
com o plano e o `approved` da anterior.

Por que não deixar o orquestrador commitar sozinho? Porque foi ele que escreveu
o veredito. Um portão que o próprio revisado abre não é um portão — é
decoração. `aif accept` é uma tecla; a independência vale a tecla.

## 9. Testar (você, de verdade)

`accept` não é `testado`. A revisão **leu o diff**; ela não rodou o código, não
subiu o serviço, não tocou no banco. Tudo que o portão garante é que um segundo
modelo olhou a mudança e não achou nada bloqueante — o que não é pouco, mas
também não é execução.

A worktree continua de pé justamente para isto. Vá até ela e exercite a
mudança como ela vai rodar em produção: mesma configuração, mesmos dados,
mesmos serviços externos.

```bash
cd "$(aif cd)"
git diff main...ai/<slug>   # o que exatamente vai entrar
<a suíte de testes do projeto>
<o sistema rodando de verdade>
```

## 10. Integrar (você)

```bash
aif land
```

Um comando, três coisas: mescla a branch da tarefa na base, remove a worktree e
apaga a branch. Rode da raiz do repositório — o `aif` recusa se você estiver
*dentro* da worktree que sumiria debaixo dos seus pés.

Ele não exige árvore limpa: o git já recusa sozinho um `checkout` ou um `merge`
que atropelaria trabalho local. Se der conflito, o `land` **desfaz o merge e
não muda nada**, mostrando o motivo e sugerindo o rebase:

```bash
git -C "$(aif cd)" rebase main   # resolva os conflitos lá
aif land                         # e tente de novo
```

Nada disso empurra para o `origin`. O `land` termina imprimindo o `git push`
sugerido e para ali — a última porta antes do código sair da sua máquina
continua sendo sua.

Se em vez de integrar você quiser jogar fora, `aif drop` remove worktree e
branch (com aviso, se a tarefa já tinha sido aceita).

---

## Vendo tokens e sendo avisado

Esta seção inteira vale só se o orquestrador for o Claude Code — o OpenCode não
lê `.claude/settings.json` e o pacote não traz equivalente para ele.

O `.claude/settings.json` do pacote faz duas coisas:

**Statusline** com modelo, projeto, custo acumulado e tokens da sessão — algo
como `[Opus] tap-list | $0.42 | 53k tok`. Os nomes dos campos que a statusline
recebe podem mudar entre versões; se a linha vier vazia, confira a referência
em `code.claude.com/docs/en/hooks`. Para consulta pontual, `/context` e `/cost`
resolvem sem configurar nada.

**Hooks `Stop` e `Notification`** avisam quando o Claude termina ou quando
precisa de você. Cada hook procura primeiro o `notify-send` do Linux e cai para
o `osascript` do macOS, então o mesmo `settings.json` serve nos dois; se não
achar nenhum dos dois, não faz nada e não atrapalha. Os hooks disparam igual no
terminal, nas extensões de IDE, no app desktop e na web — então funciona em
qualquer das duas montagens do passo 3.

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
| plano ausente ou inválido | orquestrador: `/planejar` |
| plano ok, sem mudanças de código | Reasonix: cole o bloco `/goal` |
| mudanças de código, sem revisão | orquestrador: `/revisar` |
| revisão exige mudanças | Reasonix: cole o bloco de correção |
| revisão aprovada | `aif accept` |
| aceita, integração pendente | teste de verdade, depois `aif land` |

Comandos completos: `install`, `open`, `cd`, `status`, `verify`, `accept`,
`land`, `drop`.

Variáveis de ambiente: `AIF_BRANCH_PREFIX` (padrão `ai`), `AIF_WORKTREE_DIR`
(padrão `../.ai-flow-worktrees`), `AIF_COMMIT_PREFIX` (padrão `feat`) e
`AIF_ORCHESTRATOR` (padrão `claude`; aceita `opencode`).

Enquanto houver uma tarefa aceita e não integrada, o `aif open` recusa abrir
outra — feche o ciclo com `land` ou `drop` primeiro.

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
