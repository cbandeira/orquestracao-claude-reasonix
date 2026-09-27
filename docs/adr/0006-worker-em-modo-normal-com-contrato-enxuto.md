# ADR-0006 — Rodar o worker em modo Normal com contrato enxuto

- Status: Accepted
- Data: 2026-09-27

## Contexto

Até a versão 1.2.0, o orquestrador terminava o planejamento e a revisão
reprovada com um bloco `/goal` para colar no Reasonix: um *task contract*
completo — Context, Request, Output format, Constraints e Pause policy —,
cercado por linhas de moldura.

Na prática, o worker entrou em loop algumas vezes em Goal. A documentação do
Reasonix confirma a causa: Goal não tem limite padrão de rodadas, turnos, tempo
nem de rodadas sem progresso, e continua até o próprio modelo julgar a tarefa
concluída ou travada. O teto só existe se alguém configurar
`[agent] goal_token_budget`. O usuário passou a apagar o `/goal` e as linhas de
moldura antes de colar, e a colar só o necessário.

Boa parte do bloco também repetia o que o worker já recebe: o Reasonix carrega
sozinho o `REASONIX.md` do projeto a cada turno, e ele aponta para
`.ai/implementer.md`, que já define formato de saída, política de pausa e as
proibições de commit, push e edição do veredito. Essas proibições, por sua vez,
eram só texto: com a permissão em Full access, nada as fazia valer.

O Reasonix, por fim, varre `.agents/` e `.claude/` do projeto atrás de skills e
comandos, e por isso enxerga os `planejar` e `revisar` instalados para os
orquestradores.

## Decisão

O worker roda em modo Normal, sem Goal e sem Plan, com a permissão em
Workspace access.

O handoff do orquestrador para o worker é uma linha de status fora de um bloco
de código e, dentro dele, só o que se cola: o contrato em
`.ai/current-task.md`, o Request, as Constraints próprias do plano e a linha de
encerramento `PRONTO → devolva ao orquestrador.` Regras permanentes do worker
ficam no `REASONIX.md` e no `.ai/implementer.md`, não no bloco colado.

As proibições do worker passam a ser aplicadas pela ferramenta. `aif install`
mescla no `reasonix.toml` da raiz do projeto regras `[permissions] deny` para
`git commit`, `push`, `merge`, `rebase`, `reset`, qualquer `aif` e escrita em
`.ai/review.json`, e `[skills] disable_implicit_invocation = true`.

Esse arquivo é estado local, não histórico: o Reasonix grava nele cada
"Always allow", com caminho absoluto, e o cria se não existir. Por isso ele
fica no `.gitignore`, o `aif open` o copia para dentro de cada worktree — como
já faz com `.ai/implementer.md` — e o `aif accept` nunca o commita. A mescla é
por chave, para preservar as regras `allow` que o Reasonix já tenha escrito.

## Consequências

- O turno do worker termina quando ele reporta; o limite de tentativas vem do
  `.ai/implementer.md`, não de um orçamento de Goal.
- O bloco colado é curto e não precisa ser editado antes de colar.
- As regras `deny` valem em qualquer modo de permissão, inclusive Full access,
  e em cada trecho de um comando composto.
- No Reasonix, listas não se somam entre arquivos de config: o `deny` do
  projeto substitui o da config global. O `install` avisa quando há regras
  globais para copiar.
- As travas valem só na máquina onde o `install` rodou; não viajam pelo
  repositório para outras pessoas. É o preço de o arquivo ser também onde o
  Reasonix guarda aprovações locais.
- O modelo não invoca skills sozinho no projeto; o usuário ainda as chama com
  `/skill`. Isso substitui o antigo lembrete de desabilitar, na config global,
  as skills de review do Reasonix.
- O ADR-0001 continua válido; muda só o formato do bloco que o usuário
  transfere entre as interfaces.

## Relações

- Complementa o [ADR-0001](0001-separar-orquestracao-implementacao-e-aceite.md)
  e o [ADR-0003](0003-usar-artefatos-como-contrato.md).
