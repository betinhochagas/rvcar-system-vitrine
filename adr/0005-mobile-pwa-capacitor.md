# ADR-0005: Mobile — PWA + Capacitor (sem React Native)

- **Data**: 2026-04-18
- **Status**: Aceito
- **Etapa do plan.md**: Et 8+

## Contexto

App para motoristas precisa de versão mobile. Solo, sem capacidade de manter 2
codebases (web + native).

## Decisão

App mobile = mesmo codebase React do portal, empacotado como PWA. Para acesso a
recursos nativos (push, câmera, geolocalização background), usar **Capacitor**
para gerar binário iOS/Android quando necessário.

## Alternativas consideradas

- **React Native**: descartado — codebase paralela, manutenção 2x.
- **Flutter**: descartado — outra linguagem (Dart), curva extra.
- **PWA puro**: descartado — push iOS limitado, instalação friccional.

## Consequências

### Positivas

- Um codebase, um time mental, deploy unificado.
- Update OTA (web) sem passar por App Store.

### Negativas

- Performance inferior a native em casos extremos (não esperados aqui).
- Apple Developer ($99/ano) ainda necessário para iOS via Capacitor.

## Referências

- `documentation/ROADMAP-03-APP-MOBILE.md`
