# RVCAR / Moven — vitrine técnica

SaaS **multi-tenant** de gestão de locação de veículos para motoristas de aplicativo: franquias,
frota, contratos, cobrança semanal, vistorias, manutenção e repasse a investidores. Em produção desde 2026.

Desenvolvido por mim, sozinho, na [BNU Tech](https://bnutech.com.br). **O código é privado**, porque o
sistema roda com dados reais de clientes. Este repositório mostra o que dá para mostrar sem expor
nada: a arquitetura, uma seleção de decisões técnicas (ADRs) e o método de desenvolvimento com IA.

> Na entrevista técnica, posso compartilhar a tela e mostrar o código rodando.

## Escala

Medido no repositório em 01/10/2026:

| | |
|---|---|
| Commits | **2.965** (jan–out/2026) |
| Código | **~182 mil linhas** de TypeScript (95 mil backend, 86 mil frontend), sem contar testes |
| Testes | **446 arquivos** · mais de **6.700 casos** unitários e de integração · **53** cenários E2E (Playwright) |
| ADRs | **60** registros de decisão, técnicos e de regra de negócio |
| Dívida técnica | **939** itens catalogados (`TD-NNN`), cada um com impacto e gatilho |

## Stack

**Backend:** NestJS 11 · TypeScript · Prisma · PostgreSQL · BullMQ/Redis · Swagger/OpenAPI
**Frontend:** React 19 · Vite 7 · PWA (um app instalável por portal) · Tesseract.js (OCR local)
**Infra:** Docker · Railway (API) · Vercel (SPA) · AWS S3 · GitHub Actions · Sentry
**Integrações:** gateway de pagamento (PIX/boleto) e assinatura eletrônica, configurados **por franquia** · Jira Service Management (via MCP)

## Decisões de arquitetura

Seleção de ADRs técnicas, publicadas na íntegra. As de regra de negócio do cliente ficaram de fora.

| ADR | Decisão | Por que é interessante |
|---|---|---|
| [0001](adr/0001-multi-tenant-franquia-id-guard.md) | Multi-tenant por `franquiaId` na aplicação, **não** RLS | Trade-off avaliado, e uma **errata** honesta: o guard citado nunca existiu com aquele nome |
| [0002](adr/0002-valores-monetarios-bigint-centavos.md) | Dinheiro em `BigInt` centavos | Por que nem `Number`, nem `Decimal`, nem string |
| [0003](adr/0003-stack-base.md) | Stack base | NestJS + React + Prisma + Postgres, e o que foi descartado |
| [0005](adr/0005-mobile-pwa-capacitor.md) | PWA + Capacitor, sem React Native | Um codebase para um dev solo |
| [0006](adr/0006-bullmq-fallback-sincrono.md) | Fila com fallback síncrono | Redis caído não pode significar e-mail crítico perdido |
| [0007](adr/0007-nestjs-sobre-nextjs.md) | NestJS em vez de Next.js full-stack | Workers, crons e filas pedem servidor de longa duração |
| [0008](adr/0008-migrations-forward-only.md) | Migrations só para a frente | Como reverter sem `down` |
| [0009](adr/0009-sem-monorepo-openapi-types.md) · [0011](adr/0011-codegen-openapi-typescript.md) | Tipos do front gerados do OpenAPI do back | O spike achou um **type-check que não checava nada**, com 49 erros escondidos |
| [0013](adr/0013-due-diligence-stack-infra-2026.md) | Due diligence de stack: **não mudar** | Decisão de não agir, documentada com dados |
| [0040](adr/0040-wasm-unsafe-eval-na-csp-para-o-ocr-da-cnh.md) | CSP para o OCR da CNH | Funcionalidade 12 dias em produção sem nunca funcionar; três camadas de causa e um **teste que só sabia dizer "passou"** |

> A ADR-0040 tem uma seção resumida: o original detalha a superfície de ataque do sistema em produção.
> As URLs foram substituídas por `<portal>` e `<staging>`. As demais estão como no repositório original.

## Desenvolvimento com IA: o método

Desenvolvo com **Claude Code**. O ganho de velocidade é real, e o risco também: um agente erra com
confiança, apaga o que não devia e vaza o que encontra. O método existe para que a velocidade não
custe a qualidade.

**1. Memória de projeto versionada.** `CLAUDE.md` com invariantes inegociáveis (dinheiro em centavos,
filtro de tenant fail-closed, migrations só para a frente, PII mascarada antes de qualquer log, nenhum
dado real em fixture ou documentação), o que está fora de escopo e o estado atual do projeto.

**2. Sessões com começo e fim.** Mais de 480 sessões. Cada uma abre com um briefing automático do
estado do git e fecha com critérios verificados: nada sem push, passagem de bastão atualizada.

**3. Guardas executáveis, não boas intenções.** 14 hooks próprios, quase todos nascidos de um
incidente concreto:

| Hook | O que barra | Origem |
|---|---|---|
| `agente-sem-banco-de-prod` | Subagente auditor acessando o banco de produção | Um subagente achou credenciais no disco e consultou produção por iniciativa própria |
| `redact-output` | Segredo na **saída** de comando entrando no contexto do agente | As outras camadas só olhavam o que era gravado, não o que era lido |
| `check-secrets` · `check-script-ensaio` | Gravação de segredo, ou de script que imprime segredo, **antes** de tocar o disco | Mesmo motor de detecção do pre-commit e do CI |
| `check-commit-certificado` · `check-pr-certificado` | Commit de código de runtime e entrega de PR sem auditoria do hash exato | Um PR foi entregue sem a revisão que deveria ter passado |
| `check-bash-ensaio` | Padrões de comando que já causaram dano | Cada regra cita a sessão em que falhou; taxa de falso positivo medida contra ~11 mil comandos reais |
| `snapshot-antes-de-editar` · `medir-raio-da-edicao` | Edição sem ponto de retorno; edição grande demais para ser "pontual" | Confirmação manual vira hábito de apertar Enter, e aí não protege nada |
| `validate-doc-refs` | Documentação citando arquivo ou `file:line` que não existe | Agente inventando referências |
| `tsc-on-commit` · `format-on-save` · `session-briefing` · `check-close-criteria` | Commit com erro de tipo; formatação; estado de git assumido errado; sessão fechada pela metade | — |

Dois princípios valem para todos:
- **O bloqueio diz o que fazer no lugar.** Um bloqueio que só diz "não" faz o agente tentar de novo
  com outra sintaxe, igualmente perigosa.
- **Fail-open por construção.** Erro inesperado no próprio hook deixa passar: um portão que derruba a
  sessão inteira num caso que não previu custa mais do que protege. Por isso os testes exercitam o
  caminho real de chamada, não só a função.

**4. CI como portão.** O agente não passa código quebrado porque o pipeline não deixa:

| Workflow | O que faz |
|---|---|
| Backend CI | Lint, formatação, typecheck, **detecção de drift entre schema e migrations**, testes unitários e de integração em Postgres real |
| Frontend CI | Lint, typecheck, testes, build e `audit` |
| E2E | Sobe o backend com banco descartável e roda Playwright |
| gitleaks | Varredura de segredos no diff, com deny-list e padrões de CPF/CNPJ |
| Audit | `pnpm audit` agendado |
| backup-prod-db | Backup do banco de produção para o S3 |

**5. Demanda rastreável.** Os pedidos do cliente chegam por **Jira Service Management**. Cada chamado
vira uma branch `chamado/<CHAVE>-<n>` e é citado nos commits e no pull request: são 45 chamados
rastreados em mais de 200 commits. O **MCP do Atlassian** abre toda sessão listando os chamados abertos
por JQL. Se o MCP estiver indisponível, o agente **avisa em vez de seguir calado**, porque silêncio
seria indistinguível de "não há chamado nenhum".

**6. Decisão escrita.** A IA implementa; a decisão fica registrada em ADR e é minha.

## Contato

[LinkedIn](https://linkedin.com/in/roberto-chagas-dev) · [Perfil no GitHub](https://github.com/betinhochagas)
