# ADR-0040: `'wasm-unsafe-eval'` na CSP — o OCR da CNH nunca funcionou em produção

- **Data**: 2026-08-20
- **Status**: **Aceita** (implementada na s279)
- **Decisor(es)**: Roberto Chagas (dev), com medição na origem de produção
- **Etapa do plan.md**: Et 6 (documentos) — leitura da CNH no cadastro de motorista

## Contexto

A tela de cadastro de motorista tem um botão **"Preencher pela CNH"** que lê o PDF da
Carteira Digital de Trânsito e pré-preenche os campos. A leitura é **OCR local** (Tesseract,
escolhido na s206 justamente para a CNH não sair do navegador) — e Tesseract é **WebAssembly**.

Em **2026-08-20 (s279)** o dev relatou que a tela mostrava *"Falha ao processar o PDF"* em
produção e que o arquivo não subia. A causa foi **medida na origem real**:

| Medição | Resultado |
| ------- | --------- |
| `WebAssembly.instantiate` em `<portal>` | 🔴 `CompileError: ...violates the following Content Security policy directive` |
| Mesmo código em origem **sem** CSP restritiva (controle positivo) | ✅ permitido — o instrumento enxerga |
| Worker same-origin | ✅ permitido |
| Worker `blob:` | ⚠️ **medido errado na 1ª passagem — ver §Correção** |
| Os 6 assets `/tesseract/*` em produção | ✅ servidos, MIME correto, tamanhos corretos |
| Arquivo inexistente (controle negativo) | 200 + `text/html` — a SPA mascara 404, por isso a régua olhou **conteúdo** |

A CSP de produção era `script-src 'self'`, **sem** `'wasm-unsafe-eval'`.

⚠️ **Nunca funcionou em produção.** A CSP estrita entrou em **2026-05-08** (`1aa44ad`) e o
`vercel.json` teve sua última alteração em **2026-05-16**; o OCR entrou em **2026-08-08**
(`ffec393`). A medição da s206 que aprovou o OCR rodou no **dev server, que não envia CSP** —
é o modo de falha clássico "passou em dev". O staging carrega a **mesma** CSP, então também
nunca funcionou lá.

## Decisão

Duas diretivas, não uma — a segunda foi descoberta **depois** de a primeira ir ao ar
(ver §Correção):

```
script-src 'self'  →  script-src 'self' 'wasm-unsafe-eval'
(ausente)          →  worker-src 'self' blob:
```

## Correção — o bloqueio tinha DOIS degraus, e eu declarei o segundo aberto sem prová-lo

Com `'wasm-unsafe-eval'` no ar em staging, o dev testou com uma CNH real e a leitura
**continuou falhando**. O console — que passou a existir na mesma sessão — deu a causa:

> *Creating a worker from `blob:https://<staging>/…` violates the
> following Content Security Policy directive: `"script-src 'self' 'wasm-unsafe-eval'"`.
> Note that `'worker-src'` was not explicitly set, so `'script-src'` is used as a
> fallback. The action has been blocked.*

O `tesseract.js` cria seu worker a partir de uma **blob URL**. Sem `worker-src`, o
navegador cai no `script-src`, que não permite `blob:`.

🔴 **A 1ª medição desta ADR afirmou que worker `blob:` era permitido. Estava errada, e o
erro foi de INSTRUMENTO.** O teste envolvia `new Worker(blobUrl)` em `try/catch` — e o
bloqueio por CSP **não lança sincronamente**: o construtor retorna, a violação é
disparada como evento, e o `catch` nunca roda. O `try/catch` **não conseguia** produzir
um veredito negativo; ele só sabia dizer "permitido".

Medido de novo, na mesma origem, com os dois instrumentos lado a lado:

| Instrumento | Veredito |
| ----------- | -------- |
| `try/catch` em volta de `new Worker(blob)` (o da 1ª passagem) | "PERMITIDO" — **mente** |
| Ouvinte de `securitypolicyviolation` | `violatedDirective: "worker-src"`, `blockedURI: "blob"` ⇒ **BLOQUEADO** |

⇒ **Lição de método**: um controle que só sabe produzir um dos dois resultados não é
controle. Antes de aceitar "permitido", pergunte *"este instrumento seria capaz de dizer
'bloqueado'?"*.

### ⚠️ A CSP nova NÃO chega ao usuário no deploy — chega quando o service worker atualiza

Depois de o header novo estar comprovadamente no ar, a verificação **continuou
acusando bloqueio**. A causa não era a CSP:

| O que | Valor |
| ----- | ----- |
| Política que o **documento** estava usando (`event.originalPolicy`) | a **antiga**, sem `worker-src` |
| Política que o **servidor** mandava no mesmo instante (`fetch(location, {cache:'reload'})`) | a **nova**, com `worker-src 'self' blob:` |
| `navigator.serviceWorker.controller` | **presente** |

O app é um **PWA**: o Workbox faz *precache* do `index.html` **com os headers da
resposta**, CSP inclusive. Enquanto o service worker antigo controlar a aba, ele serve
aquele documento — e o navegador aplica a CSP **gravada no cache**, não a do servidor.

Provado por eliminação: **desregistrando o service worker** e recarregando, o mesmo
worker de blob passou a responder (`postMessage(21)` → `42`), com **zero** violações.

⇒ **Toda mudança de header de segurança tem propagação em DOIS tempos** neste projeto:
(1) o deploy, que muda o que o servidor manda; (2) a atualização do SW, que muda o que o
usuário recebe. Como o `vite-plugin-pwa` está em `registerType: 'prompt'`
(`frontend/vite.config.ts`), o SW novo **fica em espera** e só assume quando o usuário
aceita o aviso de atualização — ou faz *hard reload*, ou fecha todas as abas do app.

⚠️ Isto vale para **apertar** a política também: um endurecimento de CSP não protege
quem ainda está com o SW velho. Ao mudar header de segurança, **não conclua pelo `curl`**
— o `curl` não passa pelo service worker.

### Por que `worker-src 'self' blob:` e não `blob:` no `script-src`

Pôr `blob:` no `script-src` autorizaria **qualquer** script de blob na página, incluindo
o documento principal. `worker-src` restringe a permissão ao que precisa dela: a criação
de workers. O `script-src` segue `'self' 'wasm-unsafe-eval'` — sem `blob:`, sem `eval()`.

Risco residual: quem já conseguisse executar JS na página poderia criar um worker de
blob. Como no caso do `'wasm-unsafe-eval'`, isso é **pós-exploração** — depende de vencer
antes a diretiva que não mudou.

### Por que não é `'unsafe-eval'`

São diretivas diferentes, e a distinção é o núcleo desta ADR. **Medido no navegador**, na
mesma origem, com as duas CSPs:

| CSP | WebAssembly | `eval()` de JavaScript |
| --- | ----------- | ---------------------- |
| `script-src 'self'` (antes) | 🔴 bloqueado | bloqueado |
| `script-src 'self' 'wasm-unsafe-eval'` (agora) | ✅ permitido | ✅ **segue bloqueado** |

`'wasm-unsafe-eval'` permite **compilar WebAssembly** e nada mais. Não permite script inline,
não permite script de origem externa, não permite `eval()`.

### Por que não restringir a diretiva só à rota do cadastro

O `vercel.json` aceita header por caminho, mas isto é uma **SPA**: todas as rotas são servidas
pelo mesmo `index.html`, e a CSP que vale é a do **documento carregado**. Um usuário que
chegasse em `/motoristas` e navegasse para `/motoristas/novo` pelo roteador **não** buscaria o
documento de novo, então continuaria sob a CSP da rota anterior. O resultado seria "funciona se
você recarregar nesta URL, falha se você navegar até ela" — pior que o problema.

## Efeito sobre a proteção contra XSS

> *Seção resumida na versão pública: o original detalha a superfície de ataque do sistema em produção.*

A CSP estrita também é mitigação de XSS. Ela continua de pé: o que bloqueia XSS é `script-src 'self'` recusando script inline e de terceiro, e nada disso muda aqui. Para compilar WASM, um atacante já precisaria estar executando JavaScript na página — exatamente o que a diretiva inalterada bloqueia. `'wasm-unsafe-eval'` é agravante de pós-exploração, não vetor de entrada.

## Alternativas consideradas

| Alternativa | Por que não |
| ----------- | ----------- |
| **Remover o OCR da CNH** | Resolve por subtração, mas joga fora uma feature pedida e já construída, cuja única falha é de configuração. Digitação manual da CNH é o que se queria evitar. |
| **Fazer o OCR no backend** | Tira a CNH do navegador e a manda para o servidor — o oposto da razão pela qual o OCR local foi escolhido na s206 (a CNH não sai da máquina do operador). Também custaria CPU no Railway. |
| **API paga do SERPRO (Datavalid)** | Exige e-CNPJ e tem custo por consulta. Continua sendo o caminho se o OCR se mostrar impreciso. |
| **`'unsafe-eval'`** | Resolveria também, liberando `eval()` de JS junto — desnecessariamente mais largo, e aí sim enfraqueceria a proteção contra XSS. |

## Consequências

- O autofill da CNH passa a **poder** rodar em produção e staging. ⚠️ Isto **não** é promessa de
  que o OCR lê bem uma CNH qualquer: a medição da s206 tem **base de 1 amostra**. O que esta ADR
  garante é que ele deixa de ser impedido pelo navegador.
- Vale para **todos** os deploys que usam `frontend/vercel.json` — portal, staging e investidor.
- A entrada da CNH já foi tornada **resiliente** na mesma sessão: falha de leitura não descarta
  mais o PDF, que é anexado como documento de qualquer forma. Essa correção é independente desta
  ADR e continua valendo se o OCR falhar por outro motivo.

## Como verificar que continua valendo

```
curl -sI https://<portal>/ | grep -i content-security-policy
```

Deve conter `script-src 'self' 'wasm-unsafe-eval'` **e** `worker-src 'self' blob:`. E, no navegador, na origem de produção:
`WebAssembly.instantiate(new Uint8Array([0,0x61,0x73,0x6d,1,0,0,0]))` deve resolver, enquanto
`eval('1+1')` deve continuar lançando.

## Referências

- TD-152 — assets do Tesseract servidos da própria origem porque a CSP bloqueia o jsDelivr.
- ADR-0016 — não relacionada; citada só para não confundir com o eixo do seguro.
- `frontend/vite.config.ts` — `tesseractAssetsPlugin`, que serve `/tesseract/*`.
- `frontend/src/utils/cnhParser.ts` — `extractCnhFromPdf`, o consumidor do WASM.
