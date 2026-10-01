# ADR-0009: Sem monorepo — tipos via `openapi-typescript` do `/docs`

- **Data**: 2026-04-18
- **Status**: Aceito

## Contexto

Backend e frontend separados (ADR-0007) precisam compartilhar tipos de
DTOs/responses. Opções: monorepo (pnpm workspace + pacote `@rvcar/types`),
geração de tipos a partir do OpenAPI, ou duplicar manualmente.

## Decisão

Sem monorepo. Tipos do frontend gerados a partir do `/docs` (Swagger/OpenAPI
exposto pelo Nest) via `openapi-typescript`, salvos em
`frontend/src/types/api-generated.ts`.

## Alternativas consideradas

- **Monorepo pnpm**: descartado — overhead de configuração para 2 projetos
  apenas, complica deploy Vercel/Railway.
- **Pacote npm publicado**: descartado — solo, sem registry privado.
- **Duplicar manualmente**: descartado — drift inevitável.

## Consequências

### Positivas

- Tipos sempre alinhados com contrato HTTP real (não com o que dev "achou").
- Vercel e Railway permanecem deploys simples.

### Negativas

- Comando `pnpm gen:types` precisa rodar no frontend ao mudar DTO no backend.
- Sem refactor cross-project automático (rename DTO no back não atualiza front
  até regenerar).

## Referências

- `CLAUDE.md` › "Invariantes inegociáveis" #9
- `frontend/src/types/api-generated.ts`
