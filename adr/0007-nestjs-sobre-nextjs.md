# ADR-0007: NestJS sobre Next.js (back/front separados)

- **Data**: 2026-04-18
- **Status**: Aceito

## Contexto

Tentação de adotar Next.js full-stack (RSC + Server Actions) para evitar 2
projetos. Mas: cron jobs, BullMQ, multi-módulo Prisma exigem servidor
long-running, não fazem bom par com edge/serverless do Next.

## Decisão

Backend e frontend permanecem **separados**: NestJS (Railway) + React/Vite (Vercel).
Next.js está fora de escopo, **não sugerir**.

## Alternativas consideradas

- **Next.js full-stack**: descartado — RSC força mental model adicional, cron
  jobs precisam serviço externo (Vercel Cron limita), debug entre client/server
  border é confuso.
- **tRPC + Vite**: considerado — descartado pelo investimento prévio em controllers Nest.

## Consequências

### Positivas

- Separação de concerns clara.
- Backend pode rodar workers, schedulers, queues sem hack.

### Negativas

- 2 deploys, 2 envs, 2 CIs.
- Tipos compartilhados via `openapi-typescript` (ver ADR-0009).

## Referências

- `CLAUDE.md` › "Fora de escopo (não sugerir)"
