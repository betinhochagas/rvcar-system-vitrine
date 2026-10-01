# ADR-0001: Multi-tenant via `franquiaId` + Guard (não RLS)

- **Data**: 2026-04-18
- **Status**: Aceito
- **Decisor(es)**: Roberto Chagas
- **Etapa do plan.md**: Et 4 (Franquias + Investidor)

## Contexto

Portal serve múltiplas franquias. Dados de uma franquia jamais podem vazar
para outra. Postgres oferece Row-Level Security (RLS), mas RLS adiciona
complexidade operacional (policies por tabela, debug difícil, perdas em logs).

## Decisão

Multi-tenant implementado em **camada de aplicação**: toda query Prisma filtra
por `franquiaId` extraído do JWT, validado por `FranquiaGuard` Nest. RLS do
Postgres **não** será usado.

> ⚠️ **Errata (s405, 2026-09-17)**: `FranquiaGuard` nunca existiu com esse nome. O que implementa esta decisão é
> `tenantWhere` em `backend/src/common/tenant/tenant-scope.ts` (fail-closed; 74 arquivos em `backend/src` o usam) +
> os guards de `backend/src/common/guards/` (`PermissaoGuard`, `NivelGuard`, `EscopoGlobalGuard`). A decisão (aplicação,
> não RLS) permanece. O invariante 2 do `CLAUDE.md` foi corrigido em `d8adaf09`; `schema.prisma` ainda cita o nome em
> 3 comentários (`:205`, `:656`, `:2638`) — troca exige commit certificado (`backend/prisma`), fica para o próximo TD.

## Alternativas consideradas

- **RLS Postgres**: descartado — debug difícil, performance de logs degradada,
  acoplamento com Postgres específico.
- **Schemas separados por tenant**: descartado — não escala (>50 franquias).

## Consequências

### Positivas

- Debug e logs simples.
- Migrations únicas, sem multiplicar por tenant.
- Disciplina via Guard centralizado.

### Negativas / trade-offs aceitos

- **Risco de bug**: query sem filtro = vazamento. Mitigação: ESLint custom rule + code review obrigatório em controller.
- Performance: índice composto `(franquiaId, ...)` em todas as tabelas tenant.

## Referências

- `backend/src/common/guards/franquia.guard.ts` (a ser criado em Et 4)
- `CLAUDE.md` › "Invariantes inegociáveis" #2
