# ADR-0011: Codegen de tipos back↔front via `openapi-typescript` + plugin `@nestjs/swagger`

- **Data**: 2026-06-06
- **Status**: Aceita
- **Decisor(es)**: Roberto Chagas (solo)
- **Etapa do plan.md**: Etapa 2 — Fundação Técnica (spike codegen)

## Contexto

O invariante #9 exige contrato type-safe back↔front: os tipos do frontend devem ser
**gerados** do OpenAPI do backend (`/docs`), sem monorepo (ADR-0009). O spike da Etapa 2
precisava validar se o `openapi-typescript` cobre **request bodies** (POST/PATCH), com plano
B previsto (`nestjs-zod` / `@anatine/zod-openapi`) caso fosse insuficiente.

## Achados do spike

1. **`openapi-typescript` funciona**, mas o NestJS gerava os schemas dos DTOs **vazios**
   (sem properties) — request bodies saíam sem campos. Causa: faltava o plugin
   `@nestjs/swagger`, que infere os campos a partir dos tipos TS + `class-validator`.
2. **Não foi preciso `nestjs-zod`.** Habilitar o plugin resolveu (20 de 24 schemas passaram
   a ter campos; os 4 restantes são DTOs sem propriedades).
3. O plugin é um **transformer de compilação** — rodando o gerador via `ts-node` ele não é
   aplicado. Solução: pré-gerar `backend/src/metadata.ts` com `PluginMetadataGenerator` e
   carregá-lo via `SwaggerModule.loadPluginMetadata(...)` no gerador.
4. **Descoberta colateral**: o `type-check` do frontend (`tsc --noEmit`) era **vácuo** — o
   `tsconfig.json` raiz é só *references* (`files: []`), então não checava nada (o CI passava
   falsamente). Trocado para **`tsc -b`**, o que revelou **49 erros de tipo pré-existentes**
   (todos corrigidos nesta sessão).

## Decisão

Adotar **`openapi-typescript` + plugin `@nestjs/swagger`**. Fluxo de codegen:

1. Backend: `pnpm openapi:generate` → `generate-metadata.ts` (plugin metadata) +
   `generate-openapi.ts` → escreve `backend/openapi.json` (gitignored).
2. Frontend: `pnpm types:generate` → `openapi-typescript` → `frontend/src/types/api-generated.ts`
   (commitado — é o contrato consumido).
3. O build do frontend (`tsc -b && vite build`) **enforça** o contrato: mudança de DTO no
   backend, após regenerar os tipos, **quebra o build**.

## Alternativas consideradas

- **`nestjs-zod`** / **`@anatine/zod-openapi`** (plano B): descartado — o plugin oficial
  `@nestjs/swagger` já resolve, com menos dependências.
- **Monorepo / tipos compartilhados**: descartado em ADR-0009.

## Consequências

### Positivas
- Contrato type-safe real: renomear campo de DTO no backend → build do frontend falha
  apontando o erro (provado: `LoginDto.email`→`username` quebrou `contract-check.ts`).
- O plugin também enriquece o `/docs` de **produção** (schemas completos — avança o DoD
  "Swagger documenta 100% dos endpoints").
- O `type-check` do frontend agora é **real** (`tsc -b`), pegando regressões de tipo no CI.

### Negativas / trade-offs aceitos
- `backend/openapi.json` e `backend/src/metadata.ts` são artefatos gerados (gitignored);
  exigem regenerar quando a API muda (`pnpm openapi:generate`).
- O codegen exige a infra local de pé (Postgres + Redis) para instanciar o Nest.

### Neutras
- `frontend/src/types/contract-check.ts`: arquivo de "teste de contrato" que exercita os
  tipos gerados e quebra o build se um DTO usado mudar sem regenerar.

## Referências

- ADR-0009 (contrato back↔front sem monorepo)
- `backend/scripts/generate-metadata.ts`, `backend/scripts/generate-openapi.ts`
- `backend/nest-cli.json` (plugin `@nestjs/swagger`)
- Docs Nest: "OpenAPI CLI Plugin" + "loadPluginMetadata" (geração via SWC/ts-node)
