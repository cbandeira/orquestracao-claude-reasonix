# Changelog

Versões do pacote inteiro — o script `aif`, os prompts em `.claude/` e
`.opencode/`, `.agents/`, `.codex/`, o `REASONIX.md` e o `settings.json`. Eles
evoluem juntos: um contrato novo costuma exigir validação nova, então não faz
sentido versionar cada peça em separado.

A versão que está rodando: `aif version`. A que um projeto recebeu fica no
rodapé do `aif install`.

Semântica:

- **MAJOR** — quebra o fluxo ou o formato dos artefatos em `.ai/`. Um pacote já
  instalado precisa ser atualizado à mão antes da próxima tarefa.
- **MINOR** — acrescenta comando, campo ou validação, sem invalidar o que
  existe. Reinstalar é opcional.
- **PATCH** — corrige comportamento, mensagem ou portabilidade.

## 1.3.0 — 2026-09-27

- O handoff para o Reasonix deixa de usar `/goal`. No Reasonix, Goal não tem
  limite padrão de rodadas, turnos, tempo nem de rodadas sem progresso, e já
  prendeu o worker em loop. `/planejar` e `/revisar` agora imprimem uma linha de
  status e, dentro de um bloco de código, só o que se cola: o request, as
  restrições próprias do plano e a linha `PRONTO → devolva ao orquestrador.`
  Sem linhas de moldura e sem repetir as regras permanentes, que o worker já
  recebe do `REASONIX.md`. Ver ADR-0006.
- `aif install` mescla no `reasonix.toml` da raiz do projeto regras
  `[permissions] deny` para `git commit`, `push`, `merge`, `rebase`, `reset`,
  qualquer `aif` e escrita em `.ai/review.json`. O "Não faça, nunca" do
  `REASONIX.md` passa a ser aplicado pelo Reasonix em qualquer modo de
  permissão, inclusive Full access. A mescla é por chave: um arquivo que o
  próprio Reasonix escreveu, só com `allow`, ganha o que falta e mantém o
  resto; uma chave já existente só é trocada com `--force`, com backup `.bak`.
  O `install` avisa quando a config global tem regras `deny`, porque o `deny`
  do projeto as substitui.
- O mesmo arquivo liga `[skills] disable_implicit_invocation = true`, que
  impede o modelo de invocar skills sozinho — inclusive os
  `planejar`/`revisar` de `.agents/skills/`, que o Reasonix também enxerga. O
  item do checklist que mandava desabilitar as skills de review na config
  global sai.
- O `reasonix.toml` passa a ser tratado como estado local, como o `.reasonix/`:
  o Reasonix grava nele cada "Always allow", com caminho absoluto. O `install`
  e o `open` o põem no `.gitignore`, o `open` o copia para dentro da worktree,
  o `accept` nunca o commita e o `status` não o conta como mudança de código.
  Antes, um "Always allow" dado pelo worker podia entrar no commit da tarefa.
- `aif install` avisa quando o `.gitignore` do projeto cobre arquivos do
  pacote (um `.agents/`, por exemplo), e, depois de um `--force`, lembra de
  tirar os `*.bak` do stage.
- O checklist do `install` acompanha a barra atual do Reasonix: no **+**, não
  selecionar Goal nem Plan (fica Normal); permissão em Workspace access, não
  Full access. Saem as menções a Standard, Auto, YOLO e Delivery.
- `aif status` diz "em modo Normal: cole o bloco" em vez de "cole o bloco
  /goal".

Para receber os prompts novos num projeto já instalado, rode
`aif install --force` e reaplique o comando de testes a partir de
`REASONIX.md.bak`.

## 1.2.0 — 2026-09-11

- Codex passa a ser um orquestrador de primeira classe, com Skills `$planejar`
  e `$revisar`, perfil de permissões que restringe escrita a `.ai/` e hook de
  notificação ao terminar.
- `aif install` instala e mescla os artefatos do Codex sem sobrescrever
  configurações ou hooks existentes do projeto.
- `aif open` pergunta, a cada tarefa, entre Claude Code, Codex e OpenCode. A
  escolha fica somente no estado da tarefa, funciona entre terminais e não
  altera configuração persistente do shell.
- `aif open --orchestrator <nome>` permite seleção não interativa e
  `AIF_ORCHESTRATOR` continua definindo o padrão inicial por compatibilidade.
- `aif status` passa a recuperar o orquestrador salvo e mostra a sintaxe certa:
  `/planejar`/`/revisar` ou `$planejar`/`$revisar`.

## 1.1.0 — 2026-09-05

- `aif install` ganha `--force`: atualiza arquivos que já existem no destino
  em vez de mantê-los, guardando a versão anterior em `*.bak` — inclusive o
  comando de testes customizado na última seção do `REASONIX.md`, que precisa
  ser reaplicado depois. Sem `--force`, o comportamento continua o de sempre
  (mantém o que já existe).
- `.ai/implementer.md`: o worker para depois de 3 tentativas seguidas de
  corrigir um teste vermelho e reporta `BLOQUEADO`, em vez de tentar
  indefinidamente. Sem teto, um plano ruim ou o worker travado num loop
  gastava tokens sem limite.

## 1.0.3 — 2026-09-01

- `aif install` ganha um quarto item no checklist "Falta você": ajustar a
  barra do Reasonix — execução em Standard (não Goal), aprovação de
  ferramenta em Auto (não YOLO), perfil de trabalho em Standard. YOLO
  desliga a aprovação de ferramenta inteira, deixando o "não faça nunca" do
  `REASONIX.md` sem trava técnica; Goal e Delivery empurram o modelo a
  continuar ou revisar além do escopo do plano.

## 1.0.2 — 2026-09-01

- `aif install` ganha um terceiro item no checklist "Falta você": desabilitar
  as skills de review do Reasonix (`~/.reasonix/config.toml` →
  `disabled_skills`). É lembrete de configuração de máquina, não de tarefa —
  por isso fica no `install`, que roda uma vez por projeto, e não no `open`,
  que roda uma vez por tarefa.

## 1.0.1 — 2026-09-01

- Corrige a linha de status do worker: era `PRONTO → peça a revisão ao
  orquestrador`, e o Reasonix a lia como instrução para rodar sua skill
  embutida de review (gastando tokens num trabalho que o `/revisar` já faz).
  Agora é `PRONTO → devolva ao orquestrador`, em `.ai/implementer.md`,
  `.claude/commands/planejar.md` e `.claude/commands/revisar.md`. O `aif` não
  valida esse texto — ele lê `.ai/review.json` — então nenhum comando muda de
  comportamento.

## 1.0.0 — 2026-08-27

Primeira versão numerada. Consolida o loop manual completo, já em uso:

- `aif open` cria branch, worktree e o esqueleto do contrato, e avisa quando o
  pacote ainda não está commitado na base.
- `aif status` como semáforo único do fluxo; `aif verify` e `aif accept`
  validam `.ai/review.json`.
- `aif land` integra a tarefa em um comando: mescla, remove a worktree e apaga
  a branch. `aif drop` descarta.
- `aif install` copia o pacote para um projeto sem sobrescrever nada.
- Orquestrador plugável via `AIF_ORCHESTRATOR` (Claude Code ou OpenCode), com
  os dois lendo o mesmo prompt.
- Portabilidade macOS: slug em `python3` (independe do locale e do iconv) e
  dispatch tolerante ao bash 3.2.
