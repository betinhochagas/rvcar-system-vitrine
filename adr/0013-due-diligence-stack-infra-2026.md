# ADR-0013: Due diligence de stack/infra (jun/2026) — seguir como está

- **Data**: 2026-06-14
- **Status**: Aceito
- **Etapa do plan.md**: revisão estratégica (após assinatura do contrato, retomada do projeto)

## Contexto

O projeto foi iniciado sem pesquisa aprofundada de stack/infra. Com o contrato
assinado e o projeto retomado "a todo vapor", foi feita uma revisão objetiva
(com dados atualizados de jun/2026) para decidir se dá para continuar com a
stack, a infraestrutura e o plano como estão — **antes** de investir nas etapas
pesadas (Financeiro/fiscal).

## Decisão

**Continuar com a stack, infra e plano atuais.** As escolhas são mainstream,
atuais e bem suportadas; trocar agora seria desperdício e contraria o ADR-0007.
O risco real do projeto é **operacional** (dev solo + sistema que lida com
dinheiro), não tecnológico.

### Verificações (dados de jun/2026)

- **Backend**: NestJS **11.1.26** é o major atual (v12 previsto p/ Q3/2026, com
  ESM + troca de toolchain). Projeto está no major corrente. ✅
- **Frontend**: React **19.2**, Vite 7, Node 22, TS 5.x — todos atuais. ✅
- **ORM**: Prisma **5.22** está **2 majors atrás** (atual 7.8; v7 é a recomendada
  p/ produção). Única dívida de versão relevante → ver TD-021. ⚠️
- **Infra/banco**: Railway passou a ter **PITR** (janela de 7 dias, pgBackRest +
  WAL) no Pro → cobre perda de dados financeiros. ✅
- **Infra/disponibilidade**: Railway **não oferece SLA formal** nos planos padrão
  e teve histórico de instabilidade em 2025-26 → ver TD-022. ⚠️
- **Pagamentos**: Asaas confirmado (PCI-DSS, PIX/boleto/cartão, split, webhooks)
  — sólido em 2026. ✅
- **Plano**: matriz-first (ADR-0012) é a sequência correta. ✅

## Alternativas consideradas

- **Trocar a stack** (Next.js/Drizzle/outro): descartado — sem ganho, alto custo,
  contraria ADR-0007. A stack atual não é o gargalo.
- **Migrar de infra agora** (Render/Fly/Northflank): descartado por ora — Railway
  + PITR atende a matriz na escala atual. Mantém-se como plano B documentado (ver
  TD-022). O app é portável (Docker + Postgres + Redis padrão).

## Consequências

### Ações antes de o Financeiro lidar com dinheiro real

1. **Upgrade Prisma 5 → 7** (TD-021) — barato agora, caro depois.
2. **Backup financeiro redundante** — fechar `backup-prod.ts → S3` (TD-013), além
   do PITR do Railway. Dois cofres, não um.
3. **Gatilho de decisão sobre o Railway** (TD-022) — migrar se o contrato exigir
   SLA de uptime ou se houver incidente que afete o cliente.
4. **Gate de revisão do módulo financeiro** (TD-023) — `/security-review` +
   validação do contador antes do go-live (mitiga bus factor = 1).

### Risco aceito

- **Bus factor = 1**: dev solo num sistema fiscal/financeiro. Não eliminável
  sozinho; mitigado por ADRs/testes/docs/CI já em vigor + gate do TD-023.
- **Railway sem SLA**: aceito na escala atual; revisável via TD-022.

## Referências

- ADR-0007 (stack não muda em 2026) · ADR-0012 (matriz-first)
- TD-013 (backup S3), TD-021 (Prisma 7), TD-022 (Railway SLA/plano B), TD-023 (gate financeiro)
- Fontes (jun/2026): prisma.io/changelog · github.com/nestjs/nest/releases ·
  docs.railway.com/volumes/point-in-time-recovery · asaas.com/api-de-pagamentos
