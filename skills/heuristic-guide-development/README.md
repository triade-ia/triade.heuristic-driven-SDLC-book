# Heuristic Guide — Desenvolvimento

Guia de heurísticas para a fase de **desenvolvimento**: explora valores limite e entradas inválidas (Baica) e analisa qualidade de APIs em 5 dimensões (VADER).

## Quando usar

- Quando estiver testando endpoints de API ou formulários em desenvolvimento
- Quando precisar cobrir entradas limítrofes, nulas, caracteres especiais e formatos inválidos
- Quando quiser verificar autorização, consistência de dados e tratamento de erros em APIs
- Para identificar vulnerabilidades de segurança (SQL injection, XSS) nos campos de entrada

## Como invocar

```
/heuristic-guide-development
```

Ou especifique a heurística:

```
Aplique Baica nessa feature: [descrição]
Aplique VADER nessa feature: [descrição]
```

## Inputs necessários

| Campo | Descrição | Exemplo |
|-------|-----------|---------|
| Feature ou endpoint | Descrição da funcionalidade ou endpoint a analisar | "Endpoint POST /events para criação de evento" |

## O que a skill produz

**Baica (Jonatas Faria):**
- Análise em 5 dimensões: Boundaries (limites), Nulls/Empty (nulos/vazios), Special Chars (caracteres especiais), Invalid Formats (formatos inválidos), Common Failure Patterns (SQL injection, XSS)
- Cenários de teste para cada dimensão

**VADER (Stuart Ashman):**
- Análise de qualidade da API em 5 aspectos: Values (valores), Authorization (autorização), Data Consistency (consistência), Error Handling (erros), Rate Limiting (limites)
- Identificação de vulnerabilidades e gaps de cobertura

## Exemplo de uso

```
Aplique VADER no endpoint:
"POST /panel/events — cria um evento associado a uma organização, requer Bearer token"
```

## Skills relacionadas

- [`/heuristic-guide-architeture-code`](../heuristic-guide-architeture-code/) — heurísticas de arquitetura (Count, CRUD)
- [`/heuristic-guide-qa-test`](../heuristic-guide-qa-test/) — heurísticas para execução de testes
- [`/create-api-test`](../create-api-test/) — para criar os testes de API automatizados
