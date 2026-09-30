# Registros de decisão arquitetural

Esta pasta registra decisões estruturais do projeto. `ARQUITETURA.md` descreve
o sistema como ele é hoje; os ADRs explicam por que ele chegou a esse desenho.

## Convenção

- Nome: `NNNN-resumo-em-kebab-case.md`.
- Status: `Proposed`, `Accepted`, `Deprecated` ou `Superseded`.
- ADRs aceitos não são reescritos para parecer atuais. Uma decisão nova cria
  outro ADR e aponta qual registro substitui.
- O número seguinte é `0008`.

Cada ADR contém contexto, decisão, consequências e relações com outros
registros. Um registro novo deve usar esta estrutura mínima:

```markdown
# ADR-NNNN — Título

- Status: Proposed
- Data: AAAA-MM-DD

## Contexto

## Decisão

## Consequências
```

## Índice

- [ADR-0001 — Separar orquestração, implementação e aceite humano](0001-separar-orquestracao-implementacao-e-aceite.md)
- [ADR-0002 — Isolar cada tarefa em uma Git worktree](0002-isolar-tarefas-em-git-worktrees.md)
- [ADR-0003 — Usar artefatos em disco como contrato entre agentes](0003-usar-artefatos-como-contrato.md)
- [ADR-0004 — Manter prompts canônicos com adaptadores por ferramenta](0004-manter-prompts-canonicos-com-adaptadores.md)
- [ADR-0005 — Escolher o orquestrador por tarefa](0005-escolher-orquestrador-por-tarefa.md)
- [ADR-0006 — Rodar o worker em modo Normal com contrato enxuto](0006-worker-em-modo-normal-com-contrato-enxuto.md)
- [ADR-0007 — Escolher o worker por tarefa](0007-escolher-worker-por-tarefa.md)
