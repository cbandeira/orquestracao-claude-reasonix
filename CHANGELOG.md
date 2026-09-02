# Changelog

Versões do pacote inteiro — o script `aif`, os prompts em `.claude/` e
`.opencode/`, o `REASONIX.md` e o `settings.json`. Eles evoluem juntos: um
contrato novo costuma exigir validação nova, então não faz sentido versionar
cada peça em separado.

A versão que está rodando: `aif version`. A que um projeto recebeu fica no
rodapé do `aif install`.

Semântica:

- **MAJOR** — quebra o fluxo ou o formato dos artefatos em `.ai/`. Um pacote já
  instalado precisa ser atualizado à mão antes da próxima tarefa.
- **MINOR** — acrescenta comando, campo ou validação, sem invalidar o que
  existe. Reinstalar é opcional.
- **PATCH** — corrige comportamento, mensagem ou portabilidade.

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
