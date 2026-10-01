# ADR-0006: BullMQ com fallback síncrono via flag `ENABLE_QUEUE`

- **Data**: 2026-04-18
- **Status**: Aceito
- **Etapa do plan.md**: Et 2 (Fundação Técnica)

## Contexto

Operações como envio de email crítico (cobrança, recuperação senha) não podem
ser perdidas se Redis cair. Mas em desenvolvimento, exigir Redis para rodar
qualquer job degrada DX.

## Decisão

`QueueService` aceita flag `ENABLE_QUEUE`:

- `true` (prod): enfileira em BullMQ (Redis).
- `false` (dev/test) **ou Redis indisponível**: executa síncrono inline.

Email crítico **nunca** retorna sucesso se a entrega real falhou — síncrono ou
assíncrono.

## Alternativas consideradas

- **BullMQ obrigatório**: descartado — DX ruim, prod cai se Redis cair.
- **Sem fila**: descartado — Et 2+ tem jobs longos (PDF, cobrança batch).

## Consequências

### Positivas

- Dev sem Redis local funciona.
- Prod resiliente a falhas Redis em emails críticos.

### Negativas

- Complexidade extra no `QueueService`.
- Jobs longos síncronos podem timeout em request HTTP.

## Referências

- `CLAUDE.md` › "Invariantes inegociáveis" #5
