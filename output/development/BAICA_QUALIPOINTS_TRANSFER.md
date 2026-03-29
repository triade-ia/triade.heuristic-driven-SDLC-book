# Relatório Baica — Envio de QualiPoints entre Usuários

**Requisito analisado:** `input/qualitpoints.md` (Requisito Revisado v2)
**Heurística:** Baica — Identificando e Assegurando o Core Essencial e Resiliente do Código
**Modo de entrada:** Épico (Análise de Requisitos)
**Data da análise:** 2026-03-29

---

## Entradas de dados identificadas

| # | Entrada | Origem | Tipo esperado |
|---|---------|--------|---------------|
| 1 | `amount` | Body JSON | integer (1–10.000) |
| 2 | `recipientId` | Body JSON | string (ID de usuário) |
| 3 | `Authorization` | Header | string (Bearer token) |
| 4 | `Idempotency-Key` | Header | string (UUID) |
| 5 | `Content-Type` | Header | string (application/json) |

---

## 1. Valores Mínimos/Máximos (Boundaries)

### 1.1 Campo `amount`

**Coberto pelo requisito:**
- Mínimo: 1 QualiPoint
- Máximo: 10.000 QualiPoints por transação
- Zero e negativos: rejeitados (400 — `INVALID_AMOUNT`)
- Tipo: inteiro, sem casas decimais

**Gaps identificados:**

| # | Gap | Risco | Recomendação |
|---|-----|-------|--------------|
| B-01 | Requisito não define comportamento para `amount` com casas decimais (ex.: 10.5, 99.99) | API pode aceitar silenciosamente truncando ou arredondando, corrompendo a intenção do usuário | Definir regra explícita: rejeitar valores não inteiros com erro `INVALID_AMOUNT` e mensagem "Valor deve ser um número inteiro entre 1 e 10.000" |
| B-02 | Requisito não define comportamento para valores extremamente grandes (ex.: `MAX_INT`, `2^63-1`, `Number.MAX_SAFE_INTEGER`) | Overflow no servidor ou no banco de dados; comportamento indefinido em diferentes linguagens | Validar que `amount` é inteiro positivo dentro do range 1–10.000 **antes** de qualquer operação; rejeitar valores fora do tipo inteiro (ex.: long overflow) |
| B-03 | Não há definição de limite de saldo máximo da carteira do destinatário | Créditos sucessivos podem estourar limite do tipo numérico no banco | Definir tipo numérico da coluna de saldo com capacidade suficiente e, se aplicável, limite máximo de saldo |
| B-04 | Limite por período (dia/mês) está marcado como backlog, mas não há rate limit por transação/minuto | Abuso por automação: envio massivo de 10.000 pontos repetidamente (dentro do saldo) | Mesmo no MVP, considerar rate limit básico (ex.: máx. 10 transações por minuto por usuário) como proteção contra abuso |
| B-05 | Lista de transações recentes fixa em 10 — sem paginação | Sem risco direto, mas requisito não define o que acontece se o usuário tem menos de 10 transações | Especificar: retorna até 10; se houver menos, retorna quantas existirem (array pode ser vazio) |

### 1.2 Campo `recipientId`

| # | Gap | Risco | Recomendação |
|---|-----|-------|--------------|
| B-06 | Requisito não define comprimento máximo do `recipientId` | Payload com ID absurdamente longo pode causar problemas de memória ou performance na consulta | Definir tamanho máximo (ex.: 36 chars para UUID, ou conforme padrão do ID de usuário do projeto) |

### 1.3 Header `Idempotency-Key`

| # | Gap | Risco | Recomendação |
|---|-----|-------|--------------|
| B-07 | Requisito não define tamanho mínimo/máximo do `Idempotency-Key` | Chaves extremamente longas consomem armazenamento; chaves muito curtas aumentam risco de colisão | Definir: mínimo 16 caracteres, máximo 128 caracteres; formato sugerido UUID v4 |
| B-08 | Requisito não define TTL (tempo de vida) da `Idempotency-Key` no servidor | Armazenamento cresce indefinidamente; chaves antigas nunca expiram | Definir política de expiração (ex.: chaves expiram após 24h ou 72h) |

---

## 2. Valores Nulos/Vazios (Nulls/Empty)

### 2.1 Campo `amount`

| # | Gap | Risco | Recomendação |
|---|-----|-------|--------------|
| N-01 | Requisito não define comportamento para `amount` ausente, null ou undefined no body | Erro genérico 500 ou comportamento indefinido se o campo não for enviado | Validar presença obrigatória; retornar 400 `INVALID_AMOUNT` com mensagem "Campo amount é obrigatório" |
| N-02 | Não define comportamento para body JSON vazio `{}` | Servidor pode falhar ao tentar ler campos inexistentes | Validar que body contém ambos os campos obrigatórios antes de processar |

### 2.2 Campo `recipientId`

| # | Gap | Risco | Recomendação |
|---|-----|-------|--------------|
| N-03 | Requisito não define comportamento para `recipientId` como string vazia `""` ou apenas espaços `"   "` | Consulta ao banco com string vazia pode retornar resultado inesperado ou erro genérico | Validar: rejeitar string vazia e strings compostas apenas por whitespace com erro 400 específico (ex.: `INVALID_RECIPIENT`) |
| N-04 | Não define comportamento para `recipientId` null ou ausente | Mesmo risco de N-01 | Validar presença obrigatória; retornar 400 com mensagem "Campo recipientId é obrigatório" |

### 2.3 Headers

| # | Gap | Risco | Recomendação |
|---|-----|-------|--------------|
| N-05 | Requisito não define comportamento para `Idempotency-Key` ausente | Sem idempotência, risco de débito duplo em retry | Rejeitar requisição sem `Idempotency-Key` com 400 e mensagem clara: "Header Idempotency-Key é obrigatório" |
| N-06 | Não define comportamento para `Idempotency-Key` vazia (`""`) | Todas as requisições sem chave seriam tratadas como "mesma intenção" | Rejeitar chave vazia com 400 |
| N-07 | Requisito não define comportamento para `Authorization` ausente vs. token inválido vs. token expirado | Mensagens genéricas dificultam debug pelo cliente | Diferenciar: ausente → 401 "Token não fornecido"; inválido → 401 "Token inválido"; expirado → 401 "Sessão expirada" |

---

## 3. Caracteres Especiais (Special Chars)

### 3.1 Campo `recipientId`

| # | Gap | Risco | Recomendação |
|---|-----|-------|--------------|
| S-01 | Requisito não define charset permitido para `recipientId` | Se aceitar caracteres arbitrários, risco de injection em consultas ao banco | Definir whitelist de caracteres: alfanuméricos e hífens (se UUID) ou apenas numéricos (se ID sequencial). Rejeitar qualquer caractere fora do padrão |
| S-02 | Não menciona sanitização contra SQL injection no `recipientId` | Concatenação de string em query pode ser explorada: `recipientId: "'; DROP TABLE users;--"` | **Obrigatório**: usar queries parametrizadas ou ORM. Nunca concatenar `recipientId` em SQL |
| S-03 | Não menciona sanitização contra NoSQL injection | Se backend usar MongoDB ou similar, operadores como `{"$gt": ""}` podem ser injetados | Validar tipo string primitiva; rejeitar objetos no campo `recipientId` |

### 3.2 Campo `amount`

| # | Gap | Risco | Recomendação |
|---|-----|-------|--------------|
| S-04 | Requisito não define comportamento para `amount` como string (ex.: `"abc"`, `"10"`, `"1e5"`) | Parsing implícito pode aceitar `"1e5"` como 100.000, ultrapassando o máximo | Validar tipo estritamente: rejeitar qualquer valor que não seja integer nativo do JSON. Rejeitar notação científica |

### 3.3 Header `Idempotency-Key`

| # | Gap | Risco | Recomendação |
|---|-----|-------|--------------|
| S-05 | Requisito não define caracteres permitidos no `Idempotency-Key` | Caracteres de controle, null bytes ou sequências especiais podem corromper armazenamento ou logs | Definir: aceitar apenas alfanuméricos, hífens e underscores. Rejeitar outros caracteres |

### 3.4 Mensagens de erro (saída)

| # | Gap | Risco | Recomendação |
|---|-----|-------|--------------|
| S-06 | Requisito não menciona escape de dados do usuário em mensagens de erro | Se a mensagem de erro ecoar o input (ex.: "Destinatário 'X' não encontrado"), há risco de XSS refletido | Nunca incluir input do usuário em mensagens de erro sem escape. Usar mensagens genéricas com códigos de erro |

---

## 4. Formatos Inválidos (Invalid Formats)

### 4.1 Campo `recipientId`

| # | Gap | Risco | Recomendação |
|---|-----|-------|--------------|
| F-01 | Requisito diz "ID de usuário válido" mas não define o formato do ID (UUID? numérico? alfanumérico?) | Sem validação de formato, qualquer string passa para a consulta ao banco | Definir formato esperado (ex.: UUID v4 = `^[0-9a-f]{8}-[0-9a-f]{4}-4[0-9a-f]{3}-[89ab][0-9a-f]{3}-[0-9a-f]{12}$`) e validar antes de consultar o banco |
| F-02 | Não define se `recipientId` é case-sensitive | "ABC-123" e "abc-123" podem ser tratados como IDs diferentes | Definir política: normalizar para lowercase antes de consultar, ou manter case-sensitive |

### 4.2 Campo `amount`

| # | Gap | Risco | Recomendação |
|---|-----|-------|--------------|
| F-03 | Requisito define "inteiro" mas JSON não distingue int de float nativamente | Valor `100.0` em JSON é válido e pode ser interpretado como float | Validar server-side que o valor é inteiro sem parte decimal: `amount % 1 === 0` |

### 4.3 Header `Idempotency-Key`

| # | Gap | Risco | Recomendação |
|---|-----|-------|--------------|
| F-04 | Requisito sugere "UUID" mas não exige formato específico | Chaves fracas (ex.: "1", "2", "3") aumentam risco de colisão | Recomendar UUID v4 na documentação; opcionalmente validar formato UUID no servidor |

### 4.4 Resposta `completedAt`

| # | Gap | Risco | Recomendação |
|---|-----|-------|--------------|
| F-05 | Requisito diz "ISO8601" mas não especifica timezone | Clientes em diferentes fusos podem interpretar horários incorretamente | Definir: sempre retornar em UTC com sufixo `Z` (ex.: `2026-03-29T14:30:00Z`) |

---

## 5. Padrões Comuns de Falha (Common Failure Patterns)

### 5.1 Injection

| # | Gap | Risco | Recomendação |
|---|-----|-------|--------------|
| P-01 | Requisito não menciona explicitamente queries parametrizadas | SQL injection via `recipientId` ou campos de log | **Crítico**: toda interação com banco deve usar queries parametrizadas ou ORM. Documentar como regra de implementação |
| P-02 | Não menciona validação de JSON schema do body | Campos extras no body podem ser interpretados pelo framework (mass assignment) | Validar body contra schema estrito; ignorar ou rejeitar campos não esperados |

### 5.2 XSS

| # | Gap | Risco | Recomendação |
|---|-----|-------|--------------|
| P-03 | Painel web exibe lista de transações — se dados do `recipientId` ou outros campos contêm HTML/JS, podem ser renderizados | XSS armazenado no painel web | Escapar todos os dados vindos do banco antes de renderizar no painel. Implementar Content-Security-Policy |

### 5.3 CSRF

| # | Gap | Risco | Recomendação |
|---|-----|-------|--------------|
| P-04 | Requisito não menciona proteção CSRF | Se o painel web permite ações futuras (backlog), formulários sem CSRF token podem ser explorados | Para API REST com Bearer token: CSRF é mitigado se o token não é enviado automaticamente (ex.: não está em cookie). Documentar que o token deve ser enviado via header, não cookie |

### 5.4 Autenticação e Autorização

| # | Gap | Risco | Recomendação |
|---|-----|-------|--------------|
| P-05 | Requisito diz "remetente identificado pelo token" mas não detalha validação do token (assinatura, expiração, revogação) | Token forjado ou expirado pode ser aceito | Validar assinatura, expiração e revogação do token em toda requisição |
| P-06 | Não menciona proteção contra enumeração de usuários via `recipientId` | Atacante pode testar IDs sequencialmente para descobrir usuários válidos | Retornar mesma mensagem genérica para "não encontrado" e "inativo" (já está assim: `RECIPIENT_NOT_FOUND`). Considerar rate limit em respostas 404 |

### 5.5 Concorrência e Race Conditions

| # | Gap | Risco | Recomendação |
|---|-----|-------|--------------|
| P-07 | Requisito define atomicidade mas não especifica nível de isolamento da transação no banco | Dirty reads ou phantom reads podem causar inconsistência de saldo | Definir nível de isolamento: `SERIALIZABLE` ou `REPEATABLE READ` para a transação de transferência. Ou usar `SELECT ... FOR UPDATE` no saldo |
| P-08 | Não define comportamento para race condition na `Idempotency-Key` (duas requisições com mesma chave chegam simultaneamente) | Débito duplo se ambas passarem pela checagem de idempotência antes de uma gravar | Usar constraint única no banco para `Idempotency-Key` + lock otimista ou `INSERT ... ON CONFLICT` |

### 5.6 Denial of Service

| # | Gap | Risco | Recomendação |
|---|-----|-------|--------------|
| P-09 | Requisito não define tamanho máximo do body JSON | Payload de vários MB pode consumir memória do servidor | Limitar tamanho do body (ex.: máx. 1 KB para este endpoint) |
| P-10 | Sem rate limit no MVP | Automação pode enviar milhares de requisições por segundo | Implementar rate limit básico mesmo no MVP (ex.: 60 req/min por usuário, 10 req/min para 4xx) |

### 5.7 Logging e Auditoria

| # | Gap | Risco | Recomendação |
|---|-----|-------|--------------|
| P-11 | Requisito menciona auditoria mas não define o que **não** deve ser logado | Tokens, dados sensíveis podem ser logados inadvertidamente | Definir: nunca logar `Authorization` header completo; logar apenas hash ou últimos 4 caracteres do token. Logar `Idempotency-Key`, `amount`, `recipientId`, resultado e timestamp |

---

## Checklist de Análise Baica

- [x] **Boundaries**: Limites de `amount` definidos (1–10.000); **gaps em**: decimais, overflow, saldo máximo, rate limit, tamanho de `recipientId` e `Idempotency-Key`
- [x] **Nulls/Empty**: Ausência tratada implicitamente; **gaps em**: body vazio, campos ausentes, string vazia, `Idempotency-Key` vazia, diferenciação de erros de auth
- [x] **Special Chars**: Não especificado no requisito; **gaps em**: charset de `recipientId`, sanitização contra injection, escape em mensagens de erro
- [x] **Formatos**: Formato de `amount` definido como inteiro; **gaps em**: formato de `recipientId`, float vs int em JSON, formato de `Idempotency-Key`, timezone de `completedAt`
- [x] **Common Failure Patterns**: Idempotência e atomicidade definidos; **gaps em**: queries parametrizadas, XSS no painel, race condition em idempotência, rate limit, tamanho de payload, logging seguro

---

## Resumo de Gaps por Criticidade

### Críticos (implementar antes do MVP)

| ID | Dimensão | Gap |
|----|----------|-----|
| P-01 | Failure Patterns | Queries parametrizadas não mencionadas explicitamente |
| S-02 | Special Chars | Sanitização contra SQL injection no `recipientId` |
| P-07 | Failure Patterns | Nível de isolamento da transação não definido |
| P-08 | Failure Patterns | Race condition na `Idempotency-Key` |
| N-05 | Nulls/Empty | Comportamento para `Idempotency-Key` ausente |

### Altos (implementar no MVP)

| ID | Dimensão | Gap |
|----|----------|-----|
| B-01 | Boundaries | Comportamento para `amount` decimal |
| B-02 | Boundaries | Proteção contra overflow de inteiro |
| S-04 | Special Chars | `amount` como string ou notação científica |
| F-01 | Formats | Formato do `recipientId` não definido |
| N-01/N-02 | Nulls/Empty | Campos ausentes e body vazio |
| P-09 | Failure Patterns | Tamanho máximo do body |
| P-10 | Failure Patterns | Rate limit básico |

### Médios (recomendado para MVP)

| ID | Dimensão | Gap |
|----|----------|-----|
| B-06 | Boundaries | Comprimento máximo do `recipientId` |
| B-07/B-08 | Boundaries | Tamanho e TTL da `Idempotency-Key` |
| N-03 | Nulls/Empty | `recipientId` vazio ou whitespace |
| S-01 | Special Chars | Charset permitido para `recipientId` |
| P-03 | Failure Patterns | XSS no painel web |
| P-06 | Failure Patterns | Enumeração de usuários |
| P-11 | Failure Patterns | Política de logging seguro |
| F-05 | Formats | Timezone de `completedAt` |

### Baixos (backlog)

| ID | Dimensão | Gap |
|----|----------|-----|
| B-03 | Boundaries | Saldo máximo da carteira |
| B-04 | Boundaries | Rate limit por período |
| B-05 | Boundaries | Lista com menos de 10 transações |
| F-02 | Formats | Case sensitivity do `recipientId` |
| F-04 | Formats | Validação de formato UUID no `Idempotency-Key` |
| P-04 | Failure Patterns | CSRF (mitigado por Bearer token via header) |
| S-06 | Special Chars | Escape em mensagens de erro |

---

## Regras de Validação Sugeridas (consolidadas)

```
amount:
  - tipo: integer (rejeitar float, string, null, undefined)
  - range: 1 ≤ amount ≤ 10000
  - erro: 400 INVALID_AMOUNT "Valor deve ser um número inteiro entre 1 e 10.000."

recipientId:
  - tipo: string não vazia
  - formato: [definir conforme padrão de ID do projeto — ex.: UUID v4]
  - comprimento: [definir — ex.: máx. 36 caracteres]
  - charset: [definir whitelist — ex.: alfanuméricos + hífens]
  - diferente do ID do remetente (extraído do token)
  - erro formato: 400 INVALID_RECIPIENT "ID do destinatário inválido."
  - erro ausente: 400 INVALID_RECIPIENT "Campo recipientId é obrigatório."
  - erro self: 409 SELF_TRANSFER_NOT_ALLOWED

Idempotency-Key (header):
  - obrigatório
  - tipo: string não vazia
  - comprimento: 16–128 caracteres
  - charset: alfanuméricos, hífens, underscores
  - TTL: [definir — ex.: 72 horas]
  - erro: 400 MISSING_IDEMPOTENCY_KEY "Header Idempotency-Key é obrigatório."

Body JSON:
  - tamanho máximo: 1 KB
  - campos aceitos: apenas "recipientId" e "amount"
  - campos extras: ignorar ou rejeitar

Rate limit:
  - MVP: 60 req/min por usuário autenticado
  - erro: 429 RATE_LIMIT_EXCEEDED
```

---

## Próximos Passos

1. **Refinamento do requisito**: Incorporar as regras de validação sugeridas e os gaps críticos/altos no documento de requisito antes de iniciar a codificação
2. **Implementação**: Ao codificar, aplicar as 5 dimensões Baica em cada ponto de entrada (controller/handler do `POST /api/v1/transfers` e `GET /api/v1/transactions`)
3. **Testes**: Usar este relatório como base para gerar casos de teste por camada — consultar `TEST_STRATEGY` com este relatório como contexto

---

**Referência:** Faria, Jonatas. Heurística Baica — Identificando e assegurando o core essencial e resiliente do código.
