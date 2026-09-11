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
| 1 | orquestrador | `/planejar <tarefa>` ou `$planejar <tarefa>` → escreve `.ai/current-task.md` |
| 2 | worker | cola o bloco `/goal` → implementa e roda os testes |
| 3 | orquestrador | `/revisar` ou `$revisar` → escreve `.ai/review.json` |
| 4 | você | `aif accept` → valida o veredito e commita |
| 5 | você | testa de verdade, depois `aif land` → mescla e limpa |

Perdeu o fio? `aif status` diz onde você está e qual é o próximo passo.

## Os três orquestradores

O papel de orquestrador — planejar e revisar — pode ser exercido pelo **Claude
Code**, pelo **Codex** ou pelo **OpenCode**. O prompt de cada papel continua
sendo um arquivo só em `.claude/commands/`: o OpenCode o alcança por symlink e
as Skills do Codex o carregam como protocolo canônico.

```bash
aif open "Adicionar rate limiting"
# escolha: 1 Claude Code, 2 Codex ou 3 OpenCode
```

A escolha vale somente para a tarefa aberta, fica em `.aif/current.json` e é
reutilizada pelo `aif status`. Não altera `.bashrc` nem exige `export`. Para
automação, use `aif open --orchestrator codex "<tarefa>"`.

As travas são específicas de cada ferramenta: `allowed-tools` no Claude Code,
um perfil de permissões em `.codex/config.toml` no Codex e
`.opencode/agents/*.md` no OpenCode. O planejador só escreve o contrato, o
revisor só escreve o veredito, e o estado do Git permanece somente leitura.
Claude Code e Codex têm notificações; a statusline do pacote é exclusiva do
Claude Code.

O worker é o **Reasonix**, que lê `REASONIX.md` → `.ai/implementer.md` →
`.ai/current-task.md` e nunca commita.

## Arquivos

| Caminho | Papel |
|---|---|
| `aif` | o cartório: worktree, semáforo, validação, commit, merge |
| `.claude/commands/` | os prompts de `/planejar` e `/revisar` |
| `.agents/skills/` | as Skills `$planejar` e `$revisar` do Codex |
| `.codex/` | perfil de escrita restrito a `.ai/` e notificação do Codex |
| `.opencode/` | symlinks para os mesmos prompts + as travas de ferramenta |
| `.ai/implementer.md` | contrato permanente do worker |
| `REASONIX.md` | o que o Reasonix carrega ao abrir a worktree |
| `.claude/settings.json` | statusline e notificações (Claude Code) |
| `ARQUITETURA.md` | componentes, limites, estado e fluxo do sistema |
| `docs/adr/` | decisões arquiteturais e suas justificativas |
| `CHANGELOG.md` | o que mudou em cada versão do pacote |

## Versão

O pacote inteiro — script, prompts e contratos — tem um número só, porque as
peças evoluem juntas.

```bash
aif version
```

[CHANGELOG.md](CHANGELOG.md) diz o que mudou em cada uma e quando reinstalar o
pacote nos projetos que já usam o loop.

## Documentação

- [PASSO-A-PASSO.md](PASSO-A-PASSO.md) — manual completo de instalação e uso.
- [ARQUITETURA.md](ARQUITETURA.md) — componentes, contratos, estado, fronteiras
  e fluxo do sistema.
- [docs/adr/](docs/adr/) — decisões arquiteturais e suas justificativas
  históricas.
- [CHANGELOG.md](CHANGELOG.md) — mudanças de cada versão do pacote.
