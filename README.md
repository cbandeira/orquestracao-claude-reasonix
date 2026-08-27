# orquestracao

Um loop de desenvolvimento com dois modelos e você no meio. Um agente planeja e
revisa, outro implementa, e um script — o `aif` — faz o papel de cartório: cuida
do git, valida o que os agentes escreveram e diz qual é o próximo passo.

Nada roda escondido. Os dois modelos rodam nas GUIs deles, na sua frente, e
você vê cada turno acontecer. O `aif` não invoca modelo nenhum.

## Por que assim

O ponto do arranjo é que **quem escreve não é quem aprova**. O worker
implementa, o orquestrador revisa e escreve um veredito em `.ai/review.json`, e
o commit só acontece quando o `aif accept` revalida esse veredito por conta
própria — inclusive recusando um `approved` que liste uma questão bloqueante.
Um portão que o próprio revisado abre não é um portão.

E `accept` não é `testado`: a revisão leu o diff, não rodou o código. A
worktree fica de pé justamente para você exercitar a mudança antes de integrar.

## Instalação

```bash
mkdir -p ~/.local/bin && cp aif ~/.local/bin/aif && chmod +x ~/.local/bin/aif
```

```bash
aif install /caminho/do/seu/projeto
```

Depois ajuste o comando de testes em `REASONIX.md` e commite — a worktree só
enxerga o que está commitado.

Dependências: `git`, `bash` e `python3`. Roda em Linux e macOS: não usa nada de
bash 4+ nem ferramenta cujo comportamento mude entre GNU e BSD. No macOS, o
`python3` vem com as Command Line Tools; o `jq`, só se você quiser a statusline.

## O ciclo

```bash
aif open "Adicionar rate limiting no endpoint de login"
cd "$(aif cd)"
```

| Passo | Quem | O quê |
|---|---|---|
| 1 | orquestrador | `/planejar <tarefa>` → escreve `.ai/current-task.md` |
| 2 | worker | cola o bloco `/goal` → implementa e roda os testes |
| 3 | orquestrador | `/revisar` → escreve `.ai/review.json` |
| 4 | você | `aif accept` → valida o veredito e commita |
| 5 | você | testa de verdade, depois `aif land` → mescla e limpa |

Perdeu o fio? `aif status` diz onde você está e qual é o próximo passo.

## Os dois orquestradores

O papel de orquestrador — planejar e revisar — pode ser exercido pelo **Claude
Code** ou pelo **OpenCode**. O prompt de cada papel é um arquivo só: ele vive em
`.claude/commands/`, e o `.opencode/commands/` o alcança por symlink. O
frontmatter carrega os campos das duas ferramentas, e cada uma ignora o que não
conhece.

```bash
export AIF_ORCHESTRATOR=opencode   # ou: claude (padrão)
```

As travas de ferramenta são específicas de cada um — `allowed-tools` no Claude
Code, `.opencode/agents/*.md` no OpenCode — mas têm o mesmo efeito: o planejador
só escreve o contrato, o revisor só escreve o veredito, e nenhum dos dois toca
em git que altere estado. A statusline e as notificações do pacote são só do
Claude Code.

O worker é o **Reasonix**, que lê `REASONIX.md` → `.ai/implementer.md` →
`.ai/current-task.md` e nunca commita.

## Arquivos

| Caminho | Papel |
|---|---|
| `aif` | o cartório: worktree, semáforo, validação, commit, merge |
| `.claude/commands/` | os prompts de `/planejar` e `/revisar` |
| `.opencode/` | symlinks para os mesmos prompts + as travas de ferramenta |
| `.ai/implementer.md` | contrato permanente do worker |
| `REASONIX.md` | o que o Reasonix carrega ao abrir a worktree |
| `.claude/settings.json` | statusline e notificações (Claude Code) |

## Documentação

[PASSO-A-PASSO.md](PASSO-A-PASSO.md) — o passo a passo completo, com os porquês
de cada decisão: por que a worktree fica fora do repositório, por que o
`settings.json` não é copiado por cima, por que `aif accept` é uma tecla sua.
