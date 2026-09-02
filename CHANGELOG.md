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
