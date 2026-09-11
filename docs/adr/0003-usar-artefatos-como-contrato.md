# ADR-0003 — Usar artefatos em disco como contrato entre agentes

- Status: Accepted
- Data: 2026-08-27

## Contexto

Os agentes rodam em aplicações independentes e não compartilham memória de
sessão. O framework precisa transferir intenção e veredito sem depender de um
SDK, protocolo proprietário ou acesso de um modelo à conversa do outro.

## Decisão

Usar dois artefatos efêmeros na worktree:

- `.ai/current-task.md` contém análise, plano, arquivos, testes, critérios,
  riscos, restrições e premissas;
- `.ai/review.json` contém status, resumo e issues com severidade.

O `aif` valida a estrutura mínima do plano e o schema do veredito. Questões
`CRITICAL`, `HIGH` ou `MEDIUM` sempre bloqueiam `accept`. Os dois arquivos são
ignorados pelo Git e excluídos do commit.

## Consequências

- Qualquer orquestrador capaz de obedecer ao contrato pode participar do fluxo.
- O estado fica inspecionável e editável pelo usuário.
- A compatibilidade depende da estabilidade dos títulos obrigatórios e do
  schema JSON.
- Mudanças no contrato exigem atualização coordenada do CLI, prompts,
  adaptadores e documentação.
