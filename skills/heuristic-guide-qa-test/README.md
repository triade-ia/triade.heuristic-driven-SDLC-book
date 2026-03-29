# Heuristic Guide — QA & Testes

Guia de heurísticas para a fase de **QA e testes**: explora cenários emocionais do usuário (EMOTIONS), analisa falhas do sistema sob 7 dimensões (FAILURE) e identifica comportamento sob carga massiva (FLOOD).

## Quando usar

- Quando quiser gerar cenários de teste além do caminho feliz
- Quando estiver analisando como o sistema se comporta em situações de falha
- Quando precisar cobrir testes de carga, stress e estabilidade
- Para gerar cenários empáticos baseados em diferentes perfis e estados emocionais do usuário

## Como invocar

```
/heuristic-guide-qa-test
```

Ou especifique a heurística:

```
Aplique EMOTIONS nessa feature: [descrição]
Aplique FAILURE nessa feature: [descrição]
Aplique FLOOD nessa feature: [descrição]
```

## Inputs necessários

| Campo | Descrição | Exemplo |
|-------|-----------|---------|
| Feature ou cenário | Descrição da funcionalidade ou situação a analisar | "Fluxo de compra de ingresso durante venda de alta demanda" |

## O que a skill produz

**EMOTIONS (Priscila Caimi & Jonatas Martins):**
- 9 cenários baseados nas emoções do Inside Out: Alegria, Tristeza, Raiva, Medo, Nojinho, Surpresa, Confiança, Antecipação, Desconfiança
- Cada emoção mapeia um tipo de comportamento de usuário e cenário de teste

**FAILURE (Ben Simo):**
- Análise de falhas em 7 dimensões: Functional, Appropriate, Impact, Log, UI, Recovery, Emotions
- Identificação de gaps no tratamento de erros e recuperação de falhas

**FLOOD:**
- Análise em 5 dimensões: picos de requisições, transações concorrentes, entrada massiva, degradação de performance, estabilidade e recuperação
- Cenários de stress test e comportamento sob sobrecarga

## Exemplo de uso

```
Aplique FLOOD no sistema:
"Venda de ingressos com abertura simultânea para 10.000 usuários no mesmo horário"
```

## Skills relacionadas

- [`/heuristic-guide-pos-deploy`](../heuristic-guide-pos-deploy/) — heurísticas para ambiente pós-deploy
- [`/heuristic-guide-development`](../heuristic-guide-development/) — heurísticas de desenvolvimento (Baica, VADER)
- [`/heuristic-guide-brainstorm`](../heuristic-guide-brainstorm/) — heurísticas de brainstorm inicial
