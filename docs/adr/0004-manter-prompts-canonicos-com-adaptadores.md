# ADR-0004 — Manter prompts canônicos com adaptadores por ferramenta

- Status: Accepted
- Data: 2026-09-11

## Contexto

Claude Code, Codex e OpenCode descobrem comandos, Skills e permissões em
formatos diferentes. Copiar integralmente os prompts para cada integração faria
o comportamento divergir com o tempo.

## Decisão

Manter o protocolo completo de planejamento e revisão em
`.claude/commands/`. As demais integrações são adaptadores:

- `.opencode/commands/` usa symlinks relativos para os prompts canônicos e
  `.opencode/agents/` define permissões;
- `.agents/skills/` contém Skills curtas do Codex que carregam os prompts
  canônicos;
- `.codex/config.toml` e `.codex/hooks.json` tratam permissões e notificação,
  sem duplicar o protocolo dos papéis.

## Consequências

- Uma mudança de comportamento é feita em um único corpo de prompt.
- Os adaptadores precisam continuar pequenos e alinhados.
- `aif install` distribui todos os adaptadores, mesmo quando um projeto usa só
  um orquestrador.
- Campos específicos de uma ferramenta permanecem nas suas camadas de
  integração.
