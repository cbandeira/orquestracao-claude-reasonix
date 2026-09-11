# ADR-0005 — Escolher o orquestrador por tarefa

- Status: Accepted
- Data: 2026-09-11

## Contexto

O usuário pode preferir Claude Code, Codex ou OpenCode conforme a tarefa. Uma
variável persistente de shell cria estado oculto e torna fácil iniciar uma nova
tarefa com o orquestrador escolhido numa sessão anterior.

Um processo filho também não consegue exportar uma variável para o shell pai.
Portanto, `aif open` não pode tornar um `export` global sem pedir que o usuário
avalie sua saída ou sem modificar arquivos de configuração persistentes.

## Decisão

`aif open` pergunta qual orquestrador será usado e grava a resposta no estado
local da tarefa, em `.aif/current.json`. Todos os comandos posteriores leem
esse valor e mostram a sintaxe correta.

`--orchestrator <nome>` fornece seleção não interativa. A variável
`AIF_ORCHESTRATOR` permanece apenas como padrão inicial compatível com versões
anteriores. Para Codex, o comando de abertura ativa o perfil de permissões só
naquela sessão.

## Consequências

- A escolha não altera `.bashrc`, configuração global ou tarefas futuras.
- Outro terminal recupera o orquestrador correto pelo estado do repositório.
- Automação continua possível sem responder ao menu.
- O estado precisa manter compatibilidade com tarefas antigas que não possuam o
  campo `orchestrator`.
