---
name: revisar
description: Revisa a implementação do Reasonix contra o plano ativo do aif, escreve `.ai/review.json` e produz o próximo handoff.
---

# Revisar uma implementação do aif

Leia integralmente `.claude/commands/revisar.md` e siga esse arquivo como o
protocolo canônico deste papel.

Preserve especialmente estas fronteiras:

- o único arquivo que pode ser escrito é `.ai/review.json`;
- não corrija o código e não altere o estado do Git;
- uma reprovação termina com o bloco de correção para o Reasonix, sem `/goal`;
- uma aprovação termina instruindo o usuário a executar `aif accept`.
