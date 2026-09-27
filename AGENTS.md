# Repository Guidelines

## Project Structure & Module Organization

This repository packages a human-in-the-loop development workflow. The `aif`
file is the executable Bash CLI and the primary implementation. `README.md`
is the concise overview; `PASSO-A-PASSO.md` explains the full operational
flow; `ARQUITETURA.md` describes the current architecture; `docs/adr/` records
architectural decisions; and `CHANGELOG.md` records releases. `REASONIX.md` and
`.ai/implementer.md` define the worker contract, and `reasonix.toml` enforces
its prohibitions as Reasonix `deny` rules. Agent prompts live in
`.claude/commands/`; `.opencode/commands/` and `.opencode/agents/` provide the
OpenCode integration, while `.agents/skills/` and `.codex/` provide the Codex
integration. Keep matching behavior aligned across all three orchestrators.
Update `ARQUITETURA.md` when component boundaries or invariants change, and
add an ADR for new structural decisions instead of rewriting accepted ADRs.

## Build, Test, and Development Commands

There is no build step or committed automated test suite. Validate changes to
the CLI before submitting:

```bash
bash -n aif        # syntax-check the Bash script
./aif --help       # smoke-test command parsing and help text
./aif version      # confirm the packaged version
```

For changes to `open`, `accept`, `land`, or `drop`, exercise the complete flow
in a disposable Git repository. Do not use `aif land` or `aif drop` against a
working repository unless that lifecycle is the behavior being tested.

## Coding Style & Naming Conventions

Target Bash compatible with macOS and Linux; do not introduce Bash 4+-only
features or GNU-specific command behavior. Follow the existing script style:
two-space indentation, `snake_case` function and local-variable names, quoted
path expansions, and small functions such as `cmd_accept`. Keep user-facing
messages in Portuguese, matching the current CLI and documentation. Use
Python 3 only where Bash would make JSON handling unsafe or unclear.

## Testing and Workflow Artifacts

Run the relevant manual lifecycle after changing validation or Git behavior.
`aif accept` must revalidate `.ai/review.json` and must not commit
`.ai/current-task.md`, `.ai/review.json`, `.reasonix/`, or `reasonix.toml`. These files are
per-worktree runtime state, not package history. Keep `REASONIX.md`'s test
command as an editable template for installed projects.

## Commits and Pull Requests

Use concise Conventional Commit-style subjects already used in history, for
example `feat: adicionar comando` or `fix: preservar artefatos efêmeros`.
Keep each commit focused. Pull requests should state the user-visible workflow
change, name the commands manually exercised, and call out compatibility or
documentation updates. Include terminal output only when it clarifies a
behavioral change; screenshots are generally unnecessary for this CLI package.
