# Loop manual orquestrador ⇄ worker — passo a passo

O `ai-flow` continua existindo e continua funcionando. O que muda aqui é quem
dirige: os modelos rodam nas GUIs, na sua frente, e o `aif` faz o papel de
cartório — git, validação, commit e integração. Nada roda escondido em
`subprocess`.

Para o mapa de componentes e suas fronteiras, consulte `ARQUITETURA.md`. Para
as justificativas históricas das decisões, consulte `docs/adr/`.

São três papéis e dois agentes. O **orquestrador** planeja e revisa; o
**worker** implementa; você commita e integra. O orquestrador pode ser Claude
Code, Codex ou OpenCode — os três seguem o mesmo protocolo. O worker pode ser o
Reasonix, com DeepSeek, ou uma segunda sessão do Claude Code, com Sonnet.

---

## O que tem neste pacote

| Arquivo | Onde vai | Para que serve |
|---|---|---|
| `aif` | `~/.local/bin/aif` | o cartório: worktree, semáforo, validação, commit, merge |
| `.claude/commands/planejar.md` | raiz do projeto | o prompt do papel de planejador |
| `.claude/commands/revisar.md` | raiz do projeto | o prompt do papel de revisor |
| `.agents/skills/planejar/SKILL.md` | raiz do projeto | a Skill `$planejar` do Codex |
| `.agents/skills/revisar/SKILL.md` | raiz do projeto | a Skill `$revisar` do Codex |
| `.codex/config.toml` | raiz do projeto | perfil do orquestrador: repositório somente leitura e `.ai/` gravável |
| `.codex/hooks.json` | raiz do projeto | notificação quando o Codex termina |
| `.codex/hooks/aif-notify.sh` | raiz do projeto | implementação portátil da notificação |
| `.opencode/commands/*.md` | raiz do projeto | symlinks para os dois de cima |
| `.opencode/agents/planejador.md` | raiz do projeto | as travas de ferramenta do planejador no OpenCode |
| `.opencode/agents/revisor.md` | raiz do projeto | idem, para o revisor |
| `.claude/settings.json` | raiz do projeto | statusline com tokens + notificações — **mesclar, não sobrescrever** |
| `.claude/aif-worker.json` | raiz do projeto | `deny` do worker Claude Code para commit/merge/`aif`/veredito, carregado só na sessão dele |
| `.ai/implementer.md` | raiz do projeto | contrato permanente do worker |
| `REASONIX.md` | raiz do projeto | instruções do worker: o Reasonix as carrega sozinho, o worker Claude Code as recebe no system prompt |
| `reasonix.toml` | raiz do projeto, **fora do git** | config local do Reasonix: `deny` para commit/merge/`aif` e skills só por `/skill` |

O corpo de cada comando é **um só**. O Claude Code lê `.claude/commands/`, o
OpenCode chega ao mesmo arquivo por `.opencode/commands/`, e as Skills do Codex
são adaptadores curtos que mandam carregar esse protocolo canônico. Você edita
`.claude/commands/planejar.md` e os três recebem a mudança.

Os `agents/` existem porque o OpenCode não tem o campo `allowed-tools` do
Claude Code: lá a restrição de ferramenta mora no agente, e o comando aponta
para ele pelo campo `agent:`. O efeito é o mesmo — o planejador só escreve
`.ai/current-task.md`, o revisor só escreve `.ai/review.json`, e o `bash` de
ambos nega tudo menos os comandos de leitura. No Codex, o perfil
`aif-orchestrator` lê a worktree e só permite escrita no diretório `.ai/`; os
prompts restringem essa escrita aos dois artefatos do contrato.

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

As pastas `.opencode/`, `.agents/` e `.codex/` são instaladas sempre. São
pequenas e inertes para as ferramentas que não as leem. Isso é de propósito:
trocar de orquestrador na tarefa seguinte não exige reinstalar nada.

Se já existir `.codex/config.toml`, o instalador acrescenta somente o perfil
`aif-orchestrator`; um perfil com esse nome que já exista é preservado. Em
`.codex/hooks.json`, os hooks são mesclados por conteúdo e não são duplicados.
O perfil não vira padrão do projeto: o comando que o `open` imprime o ativa
somente naquela sessão do Codex.

O `reasonix.toml` é diferente dos outros: é estado local, como o `.reasonix/`,
e fica fora do git. O Reasonix o lê por cima de `~/.reasonix/config.toml` e,
até a v1.39.4, **também escreve nele** — cada "Always allow" que você clica
vira uma regra `allow` ali, com caminho absoluto, e o arquivo é criado se não
existir. Commitado, ele sujaria o diff de toda tarefa. A partir da v1.39.5, o
Reasonix grava esses "Always allow" em `~/.reasonix/project-grants.json`, por
pasta, e ignora `mode` e `allow` de um arquivo de projeto: o `reasonix.toml`
só pode restringir. O `aif` o mantém fora do git do mesmo jeito, porque as
versões anteriores continuam gravando nele. Por isso o `install` o põe no
`.gitignore` (e avisa se ele já estiver commitado), o `aif open` o copia da raiz
para dentro de cada worktree, como faz com o `.ai/implementer.md`, e o
`aif accept` nunca o commita. O do pacote traz duas chaves:

- `[permissions] deny` barra `git commit`, `push`, `merge`, `rebase`, `reset`,
  qualquer `aif` e escrita em `.ai/review.json`. É o "Não faça, nunca" do
  `REASONIX.md` aplicado pela ferramenta, não só pedido ao modelo: `deny` vence
  em qualquer modo de permissão, inclusive Full access, e é checado em cada
  trecho de um comando composto. Uma regra como `git commit:*` compara as
  palavras iniciais do comando e deixaria passar `git -c user.name=x commit`,
  `git -C . commit` ou `git --no-pager commit`. Por isso cada subcomando tem
  também `git * commit` e `git * commit ?*`, que o Reasonix lê como curinga.
  O `?*` exige um espaço depois do subcomando, e um arquivo como
  `commit_notes.txt` não é barrado. O efeito colateral é raro:
  `git log --grep commit` também é barrado, e `--grep=commit` passa.
- `[skills] disable_implicit_invocation = true` impede o modelo de descobrir e
  invocar skills sozinho — você ainda as chama com `/skill`. Isso tira do worker
  as skills de review embutidas do Reasonix, que duplicariam o `/revisar` sem ter
  o plano, e também os `planejar`/`revisar` de `.agents/skills/`: o Reasonix
  varre `.agents/` e `.claude/` do projeto atrás de skills e comandos.

Se o projeto já tiver um `reasonix.toml` — o caso comum é um que o próprio
Reasonix escreveu, só com `allow` —, o `install` acrescenta o `deny` e o
`disable_implicit_invocation` que faltam e não mexe no resto. Uma dessas chaves
que já exista fica como está, com aviso; com `--force`, ele troca só o valor
dela e guarda o arquivo anterior em `.bak`. Um projeto instalado antes da
versão 1.4.1 do pacote mantém o `deny` antigo, sem os curingas: rode
`aif install --force` para trocá-lo.

Atenção a um detalhe que depende da versão do Reasonix. Até a v1.39.4, listas
não se somam entre arquivos: o `deny` do projeto **substitui** o da config
global dentro dele. Se você mantém regras `deny` lá, o `install` avisa, e você
as copia para o `reasonix.toml`. A partir da v1.39.5, as duas listas se somam,
e o `install` não avisa quando encontra essa versão no `PATH`.

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
worktree, e por isso sobrevivem sem commit — e o `reasonix.toml` também, que
nem deve ser commitado; o `REASONIX.md`, o `.claude/commands/`, `.agents/`,
`.codex/` e `.opencode/` não — sem commit, a
worktree abre sem as instruções e o comando de planejar não aparece no
orquestrador.

```bash
cd /caminho/do/seu/projeto && git add REASONIX.md .ai .claude .agents .codex .opencode; git status --short
```

Se você rodou o `install` com `--force`, os `*.bak` que ele deixou ao lado dos
arquivos atualizados entram junto no `git add`. Eles não são do projeto: tire-os
do stage com `git reset -q -- '*.bak'` antes de commitar. E se o `.gitignore` do
projeto cobrir alguma pasta do pacote — um `.agents/`, por exemplo —, o
`git add` reclama dela; o `install` avisa quando isso acontece, porque um
arquivo novo ali nunca chegaria à worktree.

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
o comando de planejar, sem nada apontando para o commit que faltou.

Se o projeto já tem `.ai/decisions.md` do `ai-flow`, ele é aproveitado: o `aif`
copia esse arquivo para dentro de cada worktree, e os três papéis o leem.

## 3. Escolhendo o orquestrador, e preparando as duas janelas

O orquestrador é quem planeja e revisa. Os três escrevem os mesmos
`.ai/current-task.md` e `.ai/review.json`, e o `aif` valida todos do mesmo
jeito. A escolha acontece ao abrir cada tarefa:

```bash
aif open "Adicionar rate limiting no endpoint de login"

Qual orquestrador será usado nesta tarefa?

  1  Claude Code
  2  Codex
  3  OpenCode
```

O valor fica em `.aif/current.json` até `land` ou `drop`, portanto outro
terminal continua vendo a escolha correta. Não há alteração em `.bashrc` nem
export persistente. `AIF_ORCHESTRATOR` continua aceito como padrão da seleção;
para scripts, pule a pergunta explicitamente:

```bash
aif open --orchestrator codex "Adicionar rate limiting no endpoint de login"
```

Logo depois vem a pergunta do worker:

```
Qual worker vai implementar?

  1  Reasonix + DeepSeek
  2  Claude Code + sonnet (esforço medium)
```

O padrão é o Reasonix. Os dois cumprem o mesmo contrato, e a diferença é de
custo e de viés. O Reasonix cobra à parte, mas pouco, e tira a parte que mais
consome tokens da sua cota do Claude. O worker Claude Code não custa nada além
da assinatura, mas gasta a mesma cota do orquestrador: se ele esgotar a janela
de uso no meio da tarefa, o `/revisar` fica esperando junto. E um revisor pega
mais erro de um autor de outra família de modelo. Use o Claude quando tiver
folga de cota e o Reasonix nas tarefas longas. `AIF_WORKER` muda o padrão do
menu, `--worker claude` pula a pergunta, e `AIF_WORKER_MODEL` e
`AIF_WORKER_EFFORT` trocam o modelo e o esforço do worker Claude (padrão
`sonnet` e `medium`). Ver ADR-0007.

Diferenças reais entre os orquestradores, para você escolher com os olhos abertos:

- **As travas de ferramenta.** No Claude Code elas vêm do `allowed-tools` do
  próprio comando; no OpenCode, do agente em `.opencode/agents/`. A do OpenCode
  é mais apertada num ponto: ela restringe *quais caminhos* podem ser escritos,
  então o planejador literalmente não consegue tocar em código. O
  `allowed-tools` do Claude Code concede `Write` sem restrição de destino — a
  regra "não escreva código" ali é o prompt pedindo. No Codex, o perfil
  `aif-orchestrator` bloqueia código e Git e permite escrita só em `.ai/`.
- **A chamada.** Claude Code e OpenCode usam `/planejar` e `/revisar`; Codex
  usa `$planejar` e `$revisar`.
- **Status e notificações.** Claude Code tem statusline e notificações. Codex
  recebe notificação pelo hook `Stop`. OpenCode não lê nenhum desses arquivos.

Escolhido isso, a ideia é não ter que caçar janela. Duas montagens:

Com o worker Claude Code, são duas sessões do Claude na mesma pasta: abra o
worker num terminal separado, com o comando que o `aif open` imprime (passo 6).
Com o Reasonix:

**A — tudo num VS Code só (recomendado).** Instale a extensão do orquestrador
que você escolheu e a extensão do Reasonix (`SivanLiu.reasonix-agent`, que sobe
o backend `reasonix acp` local — a CLI precisa estar instalada antes). Abra a
worktree como pasta e deixe os dois painéis lado a lado.

**B — dois apps desktop.** O orquestrador no app dele e o Reasonix no app dele.
Aí o alt-tab volta, mas as notificações avisam quando é a hora no Claude Code e
no Codex.

## 4. Abrindo uma tarefa

```bash
aif open "Adicionar rate limiting no endpoint de login"
```

Depois da seleção, o `aif` imprime o comando exato para abrir o orquestrador
e, com o worker Claude Code, também o do worker. Para Codex, será:

```bash
codex -c 'default_permissions="aif-orchestrator"'
```

Na primeira abertura de cada worktree, confirme que você confia no projeto;
sem essa confiança o Codex deliberadamente não carrega configuração e Skills
locais. O perfil exige Codex 0.138.0 ou mais recente.

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

No Claude Code ou OpenCode, dentro da worktree:

```
/planejar Adicionar rate limiting no endpoint de login
```

No Codex:

```
$planejar Adicionar rate limiting no endpoint de login
```

Ele lê o código, escreve `.ai/current-task.md` e termina imprimindo uma linha
de status e um bloco pronto para colar no worker. **Leia o plano antes de seguir** — é o momento mais barato
de corrigir rumo. Editar o `current-task.md` à mão é uso previsto; o validador
só exige que as seções `## Plan` e `## Acceptance criteria` continuem lá, com
esses nomes em inglês.

Confira quando quiser:

```bash
aif status
```

## 6. Implementar (worker)

`aif status` diz qual worker esta tarefa usa. O bloco que o orquestrador
imprime é o mesmo para os dois.

### Com o worker Claude Code

Num terminal separado, dentro da worktree, abra o worker com o comando que o
`aif open` imprimiu — o `aif status` repete esse comando enquanto for a vez
do worker:

```bash
claude --model sonnet --effort medium --permission-mode acceptEdits \
  --settings .claude/aif-worker.json --append-system-prompt-file REASONIX.md
```

Cada parte tem motivo:

- `--append-system-prompt-file REASONIX.md` entrega ao Claude o mesmo contrato
  que o Reasonix lê sozinho, e ele manda ler `.ai/implementer.md`.
- `--settings .claude/aif-worker.json` carrega as travas só nesta sessão:
  `deny` para commit, push, merge, rebase, reset — inclusive com opções
  antes do subcomando, como `git -c … commit` —, qualquer `aif`, escrita em
  `.ai/review.json` e as skills `planejar` e `revisar`. Se estivessem no
  `settings.json` do projeto, travariam também o `/revisar` de um orquestrador
  Claude Code na mesma worktree.
- `--permission-mode acceptEdits` deixa as edições passarem direto e pede sua
  aprovação para comandos de shell, como os testes. Um "sempre permitir" grava
  `.claude/settings.local.json` na worktree, que o `aif accept` nunca commita.

Cole o bloco e deixe rodando. Quando o worker encerrar com `PRONTO`, volte ao
orquestrador. Na correção de uma reprovação, cole o bloco novo na mesma
sessão.

### Com o Reasonix

Antes de colar, confira a barra do Reasonix:

- no **+**, não selecione Goal nem Plan — sem nada selecionado, o modo é Normal;
- na permissão, **Workspace access**, não Full access nem Read only.

Depois copie o bloco que o orquestrador imprimiu — só o que está dentro do
bloco de código — e cole no Reasonix. É um *task contract* enxuto: o request, as
restrições próprias do plano e a linha de encerramento. O resto — não commitar,
ficar dentro de `Files`, formato do relatório, quando pausar — o worker já
recebe do `REASONIX.md`, que o Reasonix carrega sozinho a cada turno.

Por que não Goal: no Reasonix, Goal não tem limite padrão de rodadas, turnos,
tempo nem de rodadas sem progresso. Ele segue até o próprio modelo julgar a
tarefa concluída ou travada — e já prendeu o worker em loop. Em modo Normal, o
turno acaba quando o worker reporta, e o `.ai/implementer.md` limita a 3 as
tentativas de consertar um teste vermelho. Se um dia você usar Goal mesmo
assim, ponha um teto na config global:

```toml
# ~/.reasonix/config.toml
[agent]
goal_token_budget = 20000000
```

Plan também fica de fora: ele faz o Reasonix escrever e confirmar um plano
próprio, e o plano desta tarefa já existe e foi lido por você.

Por que Workspace access: Full access pula a aprovação de ferramenta, e aí a
única trava que sobra é o `deny` do `reasonix.toml`. Read only não deixa o
worker escrever.

Deixe rodando. Ele lê `REASONIX.md` → `.ai/implementer.md` → `.ai/current-task.md`,
implementa, roda os testes e encerra com:

```
PRONTO → devolva ao orquestrador.
```

Se aparecer `CONFLITO DE PLANO`, não force: volte ao orquestrador e replaneje.
É o worker dizendo que o mapa não bate com o território.

## 7. Revisar (orquestrador)

De volta ao Claude Code ou OpenCode, na mesma sessão:

```
/revisar
```

No Codex:

```
$revisar
```

Ele lê o diff contra o plano, escreve `.ai/review.json` e termina de um dos
dois jeitos:

- **REPROVADO** — imprime um bloco de correção. Cola no worker, volta
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
`.ai/review.json`, o `.reasonix/`, o `reasonix.toml` e o
`.claude/settings.local.json`: plano, veredito e estado local dos apps são
efêmeros e por worktree — commitá-los faz a tarefa seguinte nascer
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

Claude Code e Codex recebem notificações quando terminam. O OpenCode não lê os
arquivos de hook do pacote e precisa ser acompanhado pela própria janela.

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
qualquer das duas montagens do passo 3. O `.codex/hooks.json` fornece ao Codex
uma notificação `Stop` equivalente, sem a statusline de custo/tokens.

No lado do Reasonix, o app desktop já mostra o loop de ferramentas, as
aprovações e os checkpoints por turno. Workspace access aprova sozinho o
trabalho comum dentro da worktree, e o que importa continua travado: o `deny`
do `reasonix.toml` barra commit, merge e `aif` em qualquer modo, e o sandbox
limita a escrita à worktree. Na CLI, o equivalente é
`reasonix --permission-mode auto`.

---

## O semáforo, para quando você perder o fio

```bash
aif status
```

| Estado | Próximo passo |
|---|---|
| plano ausente ou inválido | orquestrador: `/planejar` ou `$planejar` |
| plano ok, sem mudanças de código | worker (Reasonix em modo Normal, ou Claude Code): cole o bloco |
| mudanças de código, sem revisão | orquestrador: `/revisar` ou `$revisar` |
| revisão exige mudanças | worker: cole o bloco de correção |
| revisão aprovada | `aif accept` |
| aceita, integração pendente | teste de verdade, depois `aif land` |

Comandos completos: `install`, `open`, `cd`, `status`, `verify`, `accept`,
`land`, `drop`.

Variáveis de ambiente: `AIF_BRANCH_PREFIX` (padrão `ai`), `AIF_WORKTREE_DIR`
(padrão `../.ai-flow-worktrees`), `AIF_COMMIT_PREFIX` (padrão `feat`) e
`AIF_ORCHESTRATOR` (padrão inicial `claude`; aceita `codex` e `opencode`),
`AIF_WORKER` (padrão inicial `reasonix`; aceita `claude`), `AIF_WORKER_MODEL`
(padrão `sonnet`) e `AIF_WORKER_EFFORT` (padrão `medium`). O orquestrador e o
worker efetivos são escolhidos e salvos por `aif open`.

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
