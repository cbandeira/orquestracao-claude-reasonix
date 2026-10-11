# ADR-0009 — Usar Haiku com esforço médio no worker Claude Code

- Status: Accepted
- Data: 2026-10-10

## Contexto

O ADR-0007 pôs o worker Claude Code em Sonnet com esforço médio. O argumento
contra o worker Claude era a cota: ele é o papel que mais consome tokens, e a
cota é a mesma do orquestrador. O Haiku 5.5 custa uma fração do Sonnet por
token, aceita os níveis de esforço de `low` a `max` e tem como padrão o
`medium`. O plano chega pronto do orquestrador, e o revisor confere o
resultado. Por isso o worker não precisa do modelo mais capaz da casa.

Ao verificar o comando, vimos que o `--effort medium` nem sempre vale. Nos
testes, capturamos a requisição real do Claude Code com um servidor local:

- Com o Claude Code 2.1.285, o alias `haiku` abre o Haiku 4.5. Esse modelo não
  tem esforço, e a requisição sai com `budget_tokens` de 32k, sem aviso. No
  2.1.295, o mesmo alias já abre o Haiku 5.5.
- Com o ID completo `claude-haiku-5-5`, as duas versões enviam o Haiku 5.5 com
  `effort: medium`. A mais antiga só avisa `unrecognized_model`.
- `CLAUDE_CODE_EFFORT_LEVEL`, se estiver exportada no shell, vence o
  `--effort`: com `CLAUDE_CODE_EFFORT_LEVEL=high`, a requisição saiu com
  `effort: high` apesar de `--effort medium`.

## Decisão

O worker Claude Code passa a usar o Haiku 5.5 com esforço médio, e o comando
impresso pelo `aif` fica assim:

```bash
CLAUDE_CODE_EFFORT_LEVEL="${AIF_WORKER_EFFORT:-medium}" \
  claude --model "${AIF_WORKER_MODEL:-claude-haiku-5-5}" \
  --permission-mode acceptEdits \
  --settings .claude/aif-worker.json \
  --append-system-prompt-file REASONIX.md
```

- **Modelo pelo ID completo.** O alias depende da versão do Claude Code. O ID
  completo funciona igual em todas as versões testadas. Trocar de geração
  passa a ser uma edição explícita no pacote.
- **Esforço pela variável, sem `--effort`.** A variável no começo do comando
  vale só para a sessão do worker e se sobrepõe a qualquer valor exportado no
  shell. Com ela, a flag fica redundante. A sintaxe `VAR=valor comando`
  funciona em bash e zsh, no macOS e no Linux.
- **Médio, não alto.** O `high` reduz as vezes em que o Haiku dá uma mudança
  como pronta sem rodar os testes, mas gasta mais. O contrato do worker já
  manda rodar o comando de testes e proíbe reportar sucesso com teste
  vermelho, e o revisor pega o que escapar. Quem quiser outro nível usa
  `AIF_WORKER_EFFORT`.

O menu continua com duas opções. A segunda passa a ser `Claude Code +
claude-haiku-5-5 (esforço medium)`. O Sonnet continua disponível por
`AIF_WORKER_MODEL=sonnet`.

## Consequências

- O worker Claude gasta bem menos cota por tarefa. O argumento do ADR-0007 a
  favor do Reasonix, o de poupar cota, fica mais fraco, mas o Reasonix
  continua sendo o padrão e ainda é o worker de outra família de modelo.
- Uma revisão `changes_required` custa um `/revisar` do orquestrador. Se o
  Haiku em `medium` gerar reprovações demais, a correção é subir o
  `AIF_WORKER_EFFORT` ou mudar o padrão, sem mexer no fluxo.
- Um `maxEffortLevel` nas settings, ou um teto da organização, ainda pode
  rebaixar o esforço. Ele nunca sobe acima de `medium`.
- Tarefas já abertas com o worker Claude guardam só `worker: claude`. O
  `aif status` delas passa a imprimir o comando novo.
- Um Claude Code antigo avisa `unrecognized_model`. O aviso é inofensivo, mas
  vale manter o CLI atualizado.

## Relações

- Substitui o item "Modelo e esforço" do
  [ADR-0007](0007-escolher-worker-por-tarefa.md). O resto daquele registro
  continua valendo.
