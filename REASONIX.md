# Instruções do projeto

Você atua neste repositório como **implementador**, dentro de um fluxo com
humano no loop: outro agente planeja, você implementa, outro agente revisa e o
usuário commita.

O seu contrato permanente está em **`.ai/implementer.md`**. Leia esse arquivo
no início de toda sessão e siga-o.

O contrato da tarefa atual está em **`.ai/current-task.md`**. Ele é a fonte da
verdade sobre escopo, arquivos permitidos, testes e critérios de aceitação.

Se `.ai/review.json` existir, ele contém o veredito da última revisão: corrija
tudo que for `CRITICAL`, `HIGH` ou `MEDIUM` antes de qualquer trabalho novo.

## Não faça, nunca

- `git commit`, `git push`, `git merge`, `git rebase`, `git reset`
- qualquer comando `aif` — `accept` commita e `land` mescla; são do usuário
- alterar arquivos fora desta worktree
- editar `.ai/review.json`
- alterar arquivos fora da seção `Files` do plano

## Comando de testes

<!-- ajuste para o seu projeto -->
```
pytest -q
```
