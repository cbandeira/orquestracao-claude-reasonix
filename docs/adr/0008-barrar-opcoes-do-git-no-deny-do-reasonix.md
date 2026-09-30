# ADR-0008 — Barrar opções do Git antes do subcomando no deny do Reasonix

- Status: Accepted
- Data: 2026-09-30

## Contexto

O ADR-0006 transformou as proibições do worker em regras `deny` no
`reasonix.toml`, na forma de prefixo: `Bash(git commit:*)`. Ao construir as
travas do worker Claude Code (ADR-0007), vimos que no Claude Code uma regra de
prefixo deixa passar `git -c user.name=x commit`. Esse é justamente o comando
que um modelo tenta quando falta identidade no Git.

O Reasonix tem o mesmo desvio. No código dele (`internal/permission`, v1.39.5),
uma regra terminada em `:*` compara as palavras iniciais do comando, uma a uma,
e não descarta opções globais do Git. Num teste real com `reasonix -p`, em que
nada além do `deny` segura, `git -c user.name=x commit`, `git -C . commit` e
`git --no-pager commit` criaram commits, e só `git commit` foi negado.

A sintaxe também difere da do Claude Code. No Reasonix, um padrão que termina
em ` *` é lido como prefixo, não como curinga, e por isso a forma
`git * commit *` do `.claude/aif-worker.json` não funciona aqui. Um padrão que
não termina em `:*` nem em ` *` vira um curinga sobre o comando inteiro, em que
`*` casa qualquer sequência e `?` casa um caractere.

A mesma versão, v1.39.5 (28/09/2026), mudou duas coisas que o ADR-0006 dá como
fatos:

- Um arquivo de projeto só restringe. O `deny` e o `ask` dele se somam aos da
  config global, em vez de substituí-los, e `mode` e `allow` do projeto são
  ignorados.
- Um "Always allow" dado numa pasta passa a ser gravado em
  `~/.reasonix/project-grants.json`, por pasta, e não mais no `reasonix.toml`
  do projeto.

Até a v1.39.4, os dois fatos do ADR-0006 continuam valendo.

## Decisão

Cada subcomando proibido — `commit`, `push`, `merge`, `rebase` e `reset` —
mantém a regra de prefixo e ganha duas regras de curinga:

- `Bash(git * commit)`, para o subcomando no fim do comando;
- `Bash(git * commit ?*)`, que exige um espaço depois do subcomando e pelo
  menos mais um caractere.

O `?*` evita a forma terminada em ` *` e, por exigir o espaço, não barra um
arquivo cujo nome começa com o subcomando, como `commit_notes.txt`.

O `reasonix.toml` continua local, fora do git e copiado para cada worktree:
versões anteriores à v1.39.5 ainda gravam nele. O `aif install` só avisa sobre
regras `deny` da config global quando não encontra no `PATH` um Reasonix
v1.39.5 ou mais novo. Sem a CLI, por exemplo só com o app desktop, ele avisa
do mesmo jeito, com a ressalva da versão.

## Consequências

- Num teste real com a v1.39.5, foram negados `git -c … commit`,
  `git -C . commit`, `git --no-pager commit`, `git -c … commit` sem
  argumentos, os mesmos desvios para `push`, `merge`, `rebase` e `reset`, e um
  desvio dentro de um comando composto (`cd . && git -C . commit`). Passaram
  `git diff -- commit_notes.txt`, `git log -- reset_utils.txt`,
  `git log --grep=commit`, `git log --no-merges` e `git -C . status`.
- Falso positivo conhecido: `git log --grep commit`, com espaço, é barrado.
  Pela mesma forma, `git stash push` também é. São raros no trabalho de um
  worker e podem ser contornados com outra formulação.
- A mescla por chave do `install` não muda. Um projeto que já tem `deny` o
  mantém até rodar `aif install --force`, que troca só o valor dessa chave e
  guarda um `.bak`.
- `Bash(aif:*)` continua pegando só `aif` chamado pelo nome. Um caminho
  explícito, como `~/.local/bin/aif accept`, não é coberto. Fica como está: o
  `aif accept` ainda revalida o veredito, e o worker não tem motivo para
  chamá-lo por caminho.

## Relações

- Atualiza o [ADR-0006](0006-worker-em-modo-normal-com-contrato-enxuto.md):
  o formato das regras `deny` muda, e as afirmações de que listas não se somam
  e de que o Reasonix grava cada "Always allow" no `reasonix.toml` passam a
  valer só até a v1.39.4.
- Aplica ao Reasonix a proteção que o
  [ADR-0007](0007-escolher-worker-por-tarefa.md) introduziu no worker Claude
  Code, com a sintaxe própria do Reasonix.
