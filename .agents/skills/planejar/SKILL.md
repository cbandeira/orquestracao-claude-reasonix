---
name: planejar
description: Planeja uma tarefa do fluxo aif e escreve o contrato que será implementado pelo worker (Reasonix ou Claude Code). Use dentro de uma worktree aberta por `aif open`.
---

# Planejar uma tarefa do aif

Leia integralmente `.claude/commands/planejar.md` e siga esse arquivo como o
protocolo canônico deste papel.

Considere o texto que acompanha a invocação `$planejar` como `$ARGUMENTS` do
protocolo. Use a descrição mostrada por `aif status` se a invocação não trouxer
texto suficiente.

Preserve especialmente estas fronteiras:

- o único arquivo que pode ser escrito é `.ai/current-task.md`;
- código-fonte e estado do Git são somente leitura;
- o resultado final na tela é a linha de status e o bloco para colar no
  worker, sem `/goal`.
