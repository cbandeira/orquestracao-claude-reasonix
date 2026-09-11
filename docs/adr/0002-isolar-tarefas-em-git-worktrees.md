# ADR-0002 — Isolar cada tarefa em uma Git worktree

- Status: Accepted
- Data: 2026-08-27

## Contexto

Orquestrador e worker precisam operar sobre a mesma árvore sem misturar uma
tarefa em andamento com o checkout principal. Uma worktree dentro do próprio
repositório também faria ferramentas que percorrem a árvore enxergarem uma
segunda cópia do projeto.

## Decisão

Cada `aif open` cria uma branch `ai/<slug>` e uma worktree externa em
`../.ai-flow-worktrees/<projeto>/<slug>/`. Só uma tarefa pode permanecer ativa
por repositório principal.

O estado que localiza essa worktree fica em `.aif/current.json` no checkout
principal. `land` integra e limpa; `drop` descarta e limpa.

## Consequências

- A base permanece separada do trabalho dos agentes.
- Todos os agentes precisam abrir exatamente o caminho da worktree.
- Arquivos do pacote necessários à sessão devem estar commitados antes de
  `open`, porque a nova worktree nasce da branch base.
- O nome do projeto no caminho evita colisões entre repositórios irmãos.
- `land` e `drop` não podem rodar de dentro da worktree que removerão.
