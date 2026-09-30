# ADR-0007 — Escolher o worker por tarefa

- Status: Accepted
- Data: 2026-09-30

## Contexto

Até aqui o worker era sempre o Reasonix com DeepSeek. É um custo pequeno, mas
é dinheiro a mais para quem já paga uma assinatura do Claude. Pela assinatura,
um worker Claude Code com Sonnet não custa nada a mais em dinheiro. Ele gasta
cota.

Essa cota é a mesma do orquestrador. O worker é o papel que mais consome
tokens: lê código, edita, roda testes, lê a saída e tenta de novo. Um worker
Claude que esgota a janela de uso no meio de uma tarefa trava junto o
`/revisar` do Opus. O DeepSeek tem o efeito oposto: tira a parte pesada da
cota. Ele também é de outra família de modelo, e o orquestrador tem mais
chance de pegar um erro de um autor com vieses diferentes dos seus.

Nenhum dos dois workers é melhor em todas as situações. A escolha depende da
cota disponível e do tamanho da tarefa, e isso muda de uma tarefa para outra.
É a mesma situação que o ADR-0005 resolveu para o orquestrador.

## Decisão

`aif open` pergunta o worker logo depois do orquestrador, com duas opções:

1. **Reasonix + DeepSeek**: o padrão, com o comportamento atual.
2. **Claude Code + Sonnet**.

A resposta vai para o campo `worker` de `.aif/current.json`. `--worker
reasonix|claude` faz a seleção sem menu. `AIF_WORKER` só muda o padrão inicial
do menu. Tarefas antigas sem esse campo são lidas como `reasonix`.

O worker Claude roda em outra sessão do Claude Code, aberta na worktree com o
comando que o `aif` imprime:

```bash
claude --model "${AIF_WORKER_MODEL:-sonnet}" --effort "${AIF_WORKER_EFFORT:-medium}" \
  --permission-mode acceptEdits \
  --settings .claude/aif-worker.json \
  --append-system-prompt-file REASONIX.md
```

- **Contrato.** Os dois workers usam o mesmo contrato. O Reasonix lê o
  `REASONIX.md` sozinho, e o Claude o recebe por
  `--append-system-prompt-file`. Nos dois casos o arquivo aponta para
  `.ai/implementer.md`. O nome continua `REASONIX.md` para não quebrar
  projetos já instalados, em que ele guarda o comando de testes editado pelo
  usuário.
- **Travas.** As proibições do ADR-0006 viram regras `permissions.deny` em
  `.claude/aif-worker.json`: `git commit`, `push`, `merge`, `rebase`, `reset`,
  qualquer `aif`, escrita em `.ai/review.json` e invocação de `planejar` e
  `revisar`. Uma regra `Bash(git commit *)` casa pelo começo do comando e
  deixa passar `git -c user.name=x commit`, justamente o que um modelo tenta
  quando falta identidade no Git. Por isso cada subcomando também ganha a forma
  `Bash(git * commit *)`. Uma regra `Edit(...)` cobre todas as ferramentas de
  escrita de arquivo. As regras não entram em `.claude/settings.json`: esse
  arquivo também vale para o orquestrador Claude Code na mesma worktree, e um
  deny de escrita em `.ai/review.json` ali bloquearia o próprio `/revisar`.
  Com `--settings`, as regras valem só na sessão do worker.
- **Modo de permissão.** `acceptEdits`: edições passam direto, e comandos de
  shell, como os testes, pedem aprovação. As regras deny valem em qualquer
  modo.
- **Modelo e esforço.** Sonnet com esforço médio por padrão. `AIF_WORKER_MODEL`
  e `AIF_WORKER_EFFORT` sobrescrevem sem editar o pacote.
- **Handoff.** O bloco que o orquestrador imprime é o mesmo para os dois
  workers e passa a dizer "cole na sessão do worker". O orquestrador não sabe qual
  worker foi escolhido, porque o estado fica no checkout principal. Quem diz
  onde colar é o `aif status`.

## Consequências

- Com o worker Claude, uma tarefa não gera gasto fora da assinatura, mas gasta
  mais cota. Por isso o Reasonix continua como padrão, e o Claude fica para
  quando houver folga de cota.
- Diferente do `reasonix.toml`, o `.claude/aif-worker.json` não guarda estado
  local. É arquivo do pacote: o `install` o copia, ele é commitado e as travas
  do worker Claude valem em qualquer máquina que clone o projeto.
- Um "sempre permitir" na sessão do worker Claude grava
  `.claude/settings.local.json` dentro da worktree. Esse arquivo é estado
  local, como o `.reasonix/`. `aif open` o põe no `.gitignore` e `aif accept`
  nunca o commita.
- Com Claude orquestrando e Claude implementando, as duas sessões usam o mesmo
  projeto `.claude/`. Os hooks de notificação disparam nas duas, o que é
  desejável. A statusline também aparece nas duas.
- Qualquer orquestrador combina com qualquer worker. As instruções de worker
  no `/planejar` e no `/revisar` precisam de redação neutra nos três
  adaptadores (Claude Code, Codex e OpenCode).
- O estado precisa aceitar tarefas antigas sem o campo `worker`.
- As formas `Bash(git * <subcomando> *)` também barram comandos inofensivos
  que contêm a palavra, como `git log --grep commit` ou `git stash push`. É um
  falso positivo raro no trabalho de um worker, e ele pode ser contornado com
  outra formulação.

## Relações

- Aplica ao worker o mesmo desenho do
  [ADR-0005](0005-escolher-orquestrador-por-tarefa.md).
- Mantém o [ADR-0006](0006-worker-em-modo-normal-com-contrato-enxuto.md) para
  o Reasonix. Para o worker Claude, o equivalente das travas do
  `reasonix.toml` é o `.claude/aif-worker.json`.
- Preserva o [ADR-0001](0001-separar-orquestracao-implementacao-e-aceite.md):
  continua não havendo um mesmo agente que implementa e aprova, ainda que os
  dois possam ser da mesma família de modelo.
