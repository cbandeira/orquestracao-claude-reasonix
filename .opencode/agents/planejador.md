---
description: Planejador do loop manual — lê o código e escreve só .ai/current-task.md
mode: primary
permission:
  read: allow
  glob: allow
  grep: allow
  edit:
    "*": deny
    ".ai/current-task.md": allow
  bash:
    "*": deny
    "git diff*": allow
    "git log*": allow
    "git status*": allow
    "ls*": allow
  webfetch: deny
  websearch: deny
---

Equivalente ao `allowed-tools` do mesmo comando no Claude Code. O prompt do
papel vive em `.opencode/commands/planejar.md` — este arquivo só define o que
o planejador tem permissão de fazer.

A regra que importa é a da seção `edit`: o planejador escreve o contrato e
nada mais. `bash` nega tudo por padrão e reabre só os quatro comandos de
leitura, porque no OpenCode a última regra que casa é a que vale.
