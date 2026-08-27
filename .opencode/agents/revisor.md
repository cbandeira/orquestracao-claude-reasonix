---
description: Revisor do loop manual — julga o diff e escreve só .ai/review.json
mode: primary
permission:
  read: allow
  glob: allow
  grep: allow
  edit:
    "*": deny
    ".ai/review.json": allow
  bash:
    "*": deny
    "git diff*": allow
    "git log*": allow
    "git status*": allow
    "aif status*": allow
  webfetch: deny
  websearch: deny
---

Equivalente ao `allowed-tools` do mesmo comando no Claude Code. O prompt do
papel vive em `.opencode/commands/revisar.md` — este arquivo só define o que
o revisor tem permissão de fazer.

`aif status` é leitura; `aif accept` e `aif land` continuam sendo do usuário, e
o `bash` acima os recusa junto com todo o resto.
