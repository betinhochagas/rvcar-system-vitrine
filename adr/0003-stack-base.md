# ADR-0003: Stack base — NestJS + React + Prisma + Postgres

- **Data**: 2026-04-18
- **Status**: Aceito
- **Decisor(es)**: Roberto Chagas

## Contexto

Solo developer com experiência prévia em Node/TS. Necessidade de produtividade
com type safety e ecossistema maduro.

## Decisão

- **Backend**: NestJS 11 (TypeScript, decorators, DI).
- **Frontend**: React 19 + Vite 7 (SPA).
- **ORM**: Prisma 5 com migrations forward-only (ver ADR-0008).
- **DB**: PostgreSQL 14→16 (Railway).

## Alternativas consideradas

- **Next.js full-stack**: descartado (ver ADR-0007).
- **Drizzle ORM**: descartado — Prisma tem ecossistema mais maduro e DX superior.
- **Express puro**: descartado — falta estrutura para projeto multi-módulo.

## Consequências

### Positivas

- Type safety end-to-end.
- DI nativa do Nest facilita testes.
- Prisma migrate gera SQL determinístico.

### Negativas

- Curva de aprendizado decorators do Nest.
- Prisma não é leve em cold-start (mitigado por Railway).

## Referências

- `CLAUDE.md` › "Invariantes inegociáveis"
