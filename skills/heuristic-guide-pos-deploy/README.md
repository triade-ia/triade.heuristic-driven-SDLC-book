# Heuristic Guide — Pós-Deploy

Guia de heurísticas para **ambientes pós-deploy**: prioriza testes de regressão por categorias de risco (RCRCRC) e realiza análise sistemática de risco em 7 dimensões para produção (SFDPOT).

## Quando usar

- Após um deploy em produção ou homologação
- Quando precisar decidir quais testes de regressão executar primeiro após uma mudança
- Para análise de risco de uma nova feature em ambiente operacional
- Para investigar bugs reportados em produção com abordagem sistemática

## Como invocar

```
/heuristic-guide-pos-deploy
```

Ou especifique a heurística:

```
Aplique RCRCRC nesse deploy: [descrição das mudanças]
Aplique SFDPOT nessa feature: [descrição]
```

## Inputs necessários

| Campo | Descrição | Exemplo |
|-------|-----------|---------|
| Mudanças do deploy | Descrição das alterações feitas no deploy | "Adicionado módulo de cupom de desconto; refatorado cálculo de preço" |
| Feature ou componente | Funcionalidade a analisar com SFDPOT | "Sistema de pagamento com gateway externo" |

## O que a skill produz

**RCRCRC (James Bach):**
- Lista priorizada de testes por categoria: Recent (recentes), Core (core do sistema), Risky (arriscados), Configuration-sensitive (sensíveis a config), Conformance (conformidade), Complex (complexos)
- Guia de quais testes executar primeiro pós-deploy

**SFDPOT (James Bach):**
- Análise em 7 dimensões: Sources, Formats, Dependencies, Pace, Environment, Other, Time
- Mapa de riscos priorizados por severidade (impacto × probabilidade)
- Perguntas em aberto e lacunas de cobertura
- Recomendação de deploy: bloqueante, requer atenção ou apenas monitoramento

## Exemplo de uso

```
Aplique SFDPOT na feature recém-deployada:
"Integração com gateway de pagamento Pagar.me — processa pagamentos de ingresso em tempo real"
```

## Skills relacionadas

- [`/heuristic-guide-qa-test`](../heuristic-guide-qa-test/) — heurísticas para fase de testes (EMOTIONS, FAILURE, FLOOD)
- [`/debug-e2e-failure`](../debug-e2e-failure/) — para debugar falhas E2E encontradas pós-deploy
- [`/e2e-failure-root-cause`](../e2e-failure-root-cause/) — análise completa de causa raiz
