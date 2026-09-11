# ADR-0001 — Separar orquestração, implementação e aceite humano

- Status: Accepted
- Data: 2026-08-27

## Contexto

Um único agente que planeja, implementa, revisa e commita avalia o próprio
trabalho e pode abrir o portão que deveria controlá-lo. Também reduz a
visibilidade do usuário sobre mudanças de escopo e sobre o momento em que o
código passa a integrar o histórico.

## Decisão

Separar o fluxo em três autoridades:

- o orquestrador planeja e revisa;
- o Reasonix implementa e testa dentro do contrato;
- o usuário aceita, valida em execução e integra.

O orquestrador não corrige código durante a revisão. O worker não revisa nem
commita. O `aif accept` permanece uma ação explícita do usuário.

## Consequências

- O veredito vem de um modelo diferente daquele que escreveu a implementação.
- O usuário precisa transferir o `/goal` entre interfaces e acionar os portões.
- O processo custa mais turnos, mas torna escopo, responsabilidade e falhas mais
  observáveis.
- Automatizar o ciclo inteiro sem preservar autoridades distintas contrariaria
  esta decisão.
