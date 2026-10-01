# ADR-0008: Migrations Prisma forward-only (sem `down`)

- **Data**: 2026-04-18
- **Status**: Aceito

## Contexto

Prisma `migrate dev` cria SQL forward-only por padrão (sem `down.sql`). Reverter
schema em produção via "migrate down" geralmente piora a situação (perda de
dados, locks longos).

## Decisão

**Migrations só vão pra frente.** Para reverter:

1. `git revert` do commit que adicionou a migration.
2. Criar nova migration corretiva (drop coluna, etc).
3. Em caso de quebra: restore do dump pré-migration.

**Antes de migration destrutiva** em prod, executar `scripts/backup-prod.ts`.

## Alternativas consideradas

- **Migrations reversíveis**: descartado — Prisma não gera, e SQL down manual
  raramente está testado.

## Consequências

### Positivas

- SQL consistente, sem ambiguidade de "pode ou não rodar down".
- Disciplina: pensar duas vezes antes de mergear migration.

### Negativas

- Recovery requer dump (custo de armazenar backups).

## Referências

- `CLAUDE.md` › "Invariantes inegociáveis" #3
- `scripts/backup-prod.ts` (a criar)
