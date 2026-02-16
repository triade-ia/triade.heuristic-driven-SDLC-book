# Análise Baica: Envio de QualiPoints (REQ_FINAL)

## Contexto do Caso

Este documento apresenta a aplicação da heurística **Baica** (Jonatas Faria) sobre o requisito de **Envio de QualiPoints** ([setUp/REQ_FINAL.MD](../../setUp/REQ_FINAL.MD)). A análise investiga o core essencial e resiliente do código em relação a qualquer entrada de dados, focando em boundaries, nulls/vazios, caracteres especiais, formatos inválidos e padrões comuns de falha, identificando gaps de validação, riscos de borda e oportunidades de blindagem do sistema.

## Entradas de Dados Identificadas

Com base no contrato de API e nas regras de negócio:

| Entrada | Tipo | Origem | Uso |
|--------|------|--------|-----|
| `recipient_id` | string | Body (JSON) | Identificador do destinatário |
| `amount` | number | Body (JSON) | Valor em QualiPoints a transferir |
| `user_id` | (implícito) | JWT token | Remetente — nunca do body |
| `idempotency-key` | string | Header | Evitar transferências duplicadas |

---

## 1. Valores Mínimos/Máximos (Boundaries)

**Como o código é construído para validar e reagir aos menores e maiores valores possíveis? Ele previne erros de estouro, subfluxo ou lógica?**

### Situação no Requisito

- **Valor mínimo**: 1 QualiPoint — definido.
- **Valor máximo**: saldo disponível do remetente — definido conceitualmente.
- **Limites para `recipient_id`**: não definidos (tamanho, formato).
- **Limites para `idempotency-key`**: não definidos.

### Questões Aplicadas

- Quais são os limites mínimos e máximos definidos para cada entrada numérica ou de tamanho?
- O código trata explicitamente amount = 0, amount = -1, amount = MAX_INT / valores extremos?
- Há validação de range antes de operações que podem causar overflow ou underflow?
- Strings (`recipient_id`, `idempotency-key`) têm limite de tamanho validado?

### Gaps Identificados

| Gap | Risco | Recomendação |
|-----|--------|---------------|
| Valor máximo absoluto não definido | Overflow em sistemas com saldo muito alto ou tipo numérico limitado; abuso com valores astronômicos | Definir teto máximo global (ex.: 1.000.000 QualiPoints) além do limite “saldo disponível”. Documentar tipo e precisão (ex.: inteiro, 2 decimais). |
| `amount = 0` não citado explicitamente | Tratamento inconsistente (aceitar ou rejeitar) | Documentar: valor 0 é inválido (rejeitar com 422). Incluir na descrição do 422. |
| `amount` negativo não citado | Pode ser aceito por falta de validação | Documentar: valores negativos rejeitados com 422. Incluir na descrição do 422. |
| Tipo de `amount` (inteiro vs decimal) não especificado | Inconsistência (ex.: 1.5 QualiPoints?) e possível problema de precisão | Definir: inteiro não negativo ou decimal com N casas; documentar no contrato. |
| Tamanho máximo de `recipient_id` não definido | Payload muito grande, possível DoS ou comportamento indefinido | Definir limite (ex.: 64 ou 128 caracteres). Rejeitar com 413 ou 422 e mensagem clara. |
| Tamanho máximo de `idempotency-key` não definido | Mesmo risco de payload/DoS | Definir limite (ex.: 128 caracteres). Rejeitar com 413 ou 422. |

### Regras de Validação Sugeridas para Documentação

- `amount`: inteiro (ou decimal com N casas), mínimo 1, máximo min(saldo_remetente, TETO_GLOBAL).
- `amount = 0` ou negativo → 422 Unprocessable Entity.
- `recipient_id`: string, comprimento entre 1 e L caracteres (L a definir).
- `idempotency-key`: string, comprimento entre 1 e L caracteres (L a definir), opcional.

---

## 2. Valores Nulos/Vazios (Nulls/Empty)

**Como o código lida com entradas nulas, vazias ou indefinidas? Ele impede erros de NullPointerException ou lógica corrompida?**

### Situação no Requisito

- Não há menção explícita a tratamento de campos ausentes, null ou string vazia.
- Contrato mostra apenas estrutura esperada do body; não define obrigatoriedade nem comportamento quando campos faltam.

### Questões Aplicadas

- Todos os pontos de entrada tratam null, undefined, string vazia?
- Requisitos especificam o que fazer quando um campo opcional não é enviado?
- Campos obrigatórios estão explícitos?

### Gaps Identificados

| Gap | Risco | Recomendação |
|-----|--------|---------------|
| Comportamento quando `recipient_id` está ausente | 500 ou comportamento indefinido | Definir: campo obrigatório. Ausência ou null → 400 Bad Request (ou 422) com mensagem "recipient_id é obrigatório". |
| Comportamento quando `amount` está ausente | Idem | Definir: campo obrigatório. Ausência ou null → 400 (ou 422) com mensagem "amount é obrigatório". |
| `recipient_id` string vazia "" | Pode ser tratado como válido e gerar 404 genérico | Definir: string vazia ou só espaços → 422 com mensagem "recipient_id não pode ser vazio". |
| Body vazio ou `{}` | Resposta indefinida | Definir: 400 Bad Request com mensagem indicando campos obrigatórios. |
| `idempotency-key` ausente | Já é opcional; ok não enviar | Documentar: quando ausente, não há garantia de idempotência em retentativas. |
| `amount` como null em JSON | Pode causar exceção ou ser ignorado | Definir: null para amount → 422 com mensagem de formato/valor inválido. |

### Regras de Validação Sugeridas para Documentação

- `recipient_id`: obrigatório; não pode ser null, string vazia ou só espaços.
- `amount`: obrigatório; não pode ser null ou ausente; deve ser número no range válido.
- Body deve ser JSON válido e conter ambos os campos; caso contrário → 400 com mensagem clara.

---

## 3. Caracteres Especiais (Special Chars)

**Como o código sanitiza e valida inputs com caracteres não alfanuméricos, símbolos ou emojis?**

### Situação no Requisito

- `recipient_id` é descrito como string; não há política para caracteres permitidos.
- Não há menção a sanitização para injeção (SQL, comando, etc.) nem a uso em queries ou logs.

### Questões Aplicadas

- Há sanitização para evitar injeção (SQL, comando, etc.)?
- O sistema aceita ou rejeita emojis, Unicode, quebras de linha em `recipient_id`?
- Codificação (UTF-8) é tratada de forma consistente?

### Gaps Identificados

| Gap | Risco | Recomendação |
|-----|--------|---------------|
| Formato de `recipient_id` não definido | UUID, número, email? Aceita qualquer string? | Definir formato (ex.: UUID v4). Rejeitar valores que não batem com o formato → 422 com mensagem "Formato de recipient_id inválido (esperado: UUID)". |
| Caracteres especiais em `recipient_id` | Se for ID interno, caracteres como ', ", \, newline podem causar problemas em log, query ou armazenamento | Usar apenas validação de formato (ex.: UUID). Nunca concatenar em SQL; usar parâmetros/ORM. |
| `idempotency-key` sem política | Caracteres de controle ou muito longos podem afetar armazenamento ou log | Definir: caracteres permitidos (ex.: alfanuméricos e hífen) e tamanho máximo. Rejeitar fora do padrão com 400/422. |
| Uso de `recipient_id` em queries não documentado | Risco de SQL/NoSQL injection se concatenado | Documentar: uso de queries parametrizadas ou ORM; nunca concatenar input em query. |

### Regras de Validação Sugeridas para Documentação

- `recipient_id`: formato definido (ex.: UUID v4); rejeitar caracteres ou formato inválido com 422.
- `idempotency-key`: formato opcional definido (ex.: alfanumérico + hífen), tamanho máximo definido.
- Implementação: sempre usar parâmetros/ORM para qualquer uso de `recipient_id` em banco ou cache.

---

## 4. Formatos Inválidos (Invalid Formats)

**Se o código espera um identificador ou um número, como valida o formato e como rejeita formatos incorretos?**

### Situação no Requisito

- 422 é citado para "valor abaixo do mínimo ou formato inválido", mas não detalha quais formatos são inválidos nem a mensagem retornada.
- Não há especificação de formato para `recipient_id` nem para o tipo numérico de `amount`.

### Questões Aplicadas

- Cada campo com formato definido tem validação explícita?
- Mensagens de erro informam o formato esperado?
- Requisitos especificam formatos aceitos e política de rejeição?

### Gaps Identificados

| Gap | Risco | Recomendação |
|-----|--------|---------------|
| Formato de `recipient_id` não especificado | Clientes enviam email, nome, número; backend pode interpretar diferente | Especificar no contrato: ex. "UUID v4". Incluir exemplo válido e mensagem de erro para formato inválido. |
| `amount` como string ("10") ou tipo errado | Parsing inconsistente (aceitar vs rejeitar) | Definir: tipo number no JSON. String ou tipo errado → 422 com mensagem "amount deve ser um número". |
| `amount` com casas decimais | Se for inteiro, 10.5 pode ser truncado ou rejeitado sem critério | Definir se aceita decimal e quantas casas; caso inteiro, rejeitar decimais com 422. |
| Mensagem genérica para 422 | Cliente não sabe se o problema é valor mínimo, formato ou outro | Documentar códigos/mensagens por caso: valor abaixo do mínimo, valor acima do saldo, formato de recipient_id inválido, formato de amount inválido. |
| Content-Type não exigido | Body não-JSON pode gerar 500 ou comportamento estranho | Exigir `Content-Type: application/json`; caso contrário → 415 ou 400 com mensagem clara. |

### Regras de Validação Sugeridas para Documentação

- **Request**: `Content-Type: application/json` obrigatório.
- **recipient_id**: string no formato UUID v4 (exemplo e regex ou link para spec).
- **amount**: number; inteiro positivo ou decimal com N casas (conforme decisão de negócio).
- **422**: body com `code` e `message` por tipo de erro (ex.: `AMOUNT_BELOW_MINIMUM`, `INVALID_RECIPIENT_FORMAT`).

---

## 5. Padrões Comuns de Falha (Common Failure Patterns)

**Como o código se protege contra vulnerabilidades conhecidas (SQL injection, XSS, escalação de privilégios, etc.)?**

### Situação no Requisito

- JWT e extração de `user_id` do token (nunca do body) estão definidos — alinhado com auth.
- Atomicidade via transação está definida.
- Idempotência via header está definida.
- Não há menção a CSRF, rate limiting, nem a como `recipient_id` é usado em banco/queries.

### Questões Aplicadas

- Há proteção contra SQL/NoSQL injection?
- Autenticação e autorização são verificadas em todos os pontos sensíveis, independentemente do input?
- Há validação de CSRF ou equivalente em APIs sensíveis?
- Há rate limiting ou proteção contra abuso?

### Gaps Identificados

| Gap | Risco | Recomendação |
|-----|--------|---------------|
| Uso de `recipient_id` em queries não documentado | SQL/NoSQL injection se concatenado | Documentar: uso exclusivo de queries parametrizadas ou ORM. Incluir na especificação técnica. |
| Ausência de rate limiting | Abuso, muitas requisições por usuário ou por IP | Documentar: rate limiting por usuário (e opcionalmente por IP). Resposta 429 com Retry-After quando excedido. |
| CSRF em chamadas a partir do Painel Web | Se a API for chamada por browser, CSRF pode permitir transferência indesejada | Para chamadas browser: tokens CSRF ou SameSite cookies; para apenas SPA + JWT em header, avaliar risco e documentar decisão. |
| Confiança em dados do body apenas para negócio | Já está correto não confiar em user_id do body | Manter: remetente sempre do JWT; validar que o token corresponde ao usuário autenticado. |
| Idempotency key reutilizada por outro usuário | Um usuário poderia reutilizar key de outro (se key não for atada ao usuário) | Documentar: idempotency-key deve ser validada no escopo do usuário (mesmo usuário + mesma key = idempotência). |
| Log/saída contendo `recipient_id` ou dados sensíveis | Risco de vazamento em log ou resposta de erro | Documentar: não logar corpo completo em produção; em erros, não retornar dados sensíveis de outros usuários. |

### Regras de Segurança Sugeridas para Documentação

- **Injection**: Nunca concatenar `recipient_id` (ou qualquer input) em SQL/NoSQL; usar sempre parâmetros/ORM.
- **Auth**: Remetente sempre identificado pelo JWT; ignorar qualquer `user_id` ou equivalente no body.
- **Idempotency**: Escopo por usuário autenticado; mesma key + mesmo usuário = mesma resposta.
- **Rate limiting**: Definir limite por usuário (e opcionalmente por IP); retornar 429 quando excedido.
- **Respostas de erro**: Não expor dados de outros usuários; mensagens genéricas quando apropriado (ex.: 404 para destinatário inexistente sem revelar se o ID existe ou não, conforme política de privacidade).

---

## Resumo dos Gaps de Validação

| Dimensão | Gaps principais |
|----------|------------------|
| **Boundaries** | Teto máximo global para amount; amount = 0 e negativo; tipo e precisão de amount; tamanho máximo de recipient_id e idempotency-key. |
| **Nulls/Empty** | Campos obrigatórios explícitos; tratamento de ausência, null e string vazia para recipient_id e amount; body vazio ou incompleto. |
| **Special Chars** | Formato de recipient_id (ex.: UUID); política para idempotency-key; uso parametrizado em queries. |
| **Formatos** | Formato de recipient_id e amount no contrato; mensagens de 422 por tipo de erro; Content-Type obrigatório. |
| **Failure Patterns** | Queries parametrizadas/ORM documentado; rate limiting; escopo de idempotency por usuário; CSRF quando aplicável; não vazar dados em log/erro. |

---

## Sugestões de Regras e Mensagens para o Contrato

### Códigos de erro sugeridos (422)

- `RECIPIENT_ID_REQUIRED`: recipient_id ausente ou vazio.
- `AMOUNT_REQUIRED`: amount ausente ou null.
- `AMOUNT_BELOW_MINIMUM`: valor &lt; 1.
- `AMOUNT_INVALID_TYPE`: amount não é número.
- `AMOUNT_NEGATIVE`: valor negativo.
- `AMOUNT_ABOVE_MAX`: valor &gt; saldo (ou 402 dedicado).
- `RECIPIENT_ID_INVALID_FORMAT`: formato de recipient_id inválido (ex.: não é UUID).
- `PAYLOAD_TOO_LARGE`: recipient_id ou idempotency-key excede tamanho máximo (se usar 413, manter consistente).

### Exemplo de corpo de erro (422)

```json
{
  "error": "Unprocessable Entity",
  "code": "AMOUNT_BELOW_MINIMUM",
  "message": "O valor deve ser no mínimo 1 QualiPoint."
}
```

### Exemplo de contrato enriquecido (trecho)

- **recipient_id**: string, obrigatório, formato UUID v4, 1–36 caracteres.
- **amount**: number, obrigatório, inteiro, mínimo 1, máximo min(saldo_remetente, TETO_GLOBAL).
- **idempotency-key**: string, opcional, 1–128 caracteres, alfanumérico + hífen.
- **Content-Type**: application/json obrigatório.

---

## Casos de Teste Sugeridos (Baica)

### Boundaries

- amount = 0 → 422.
- amount = -1 → 422.
- amount = 1 (mínimo) → 201.
- amount = saldo exato do remetente → 201.
- amount = saldo + 1 → 402.
- amount muito grande (ex.: 1e20 ou MAX_SAFE_INTEGER) → 422 ou 402 conforme teto.
- recipient_id com 0 caracteres → 422.
- recipient_id com tamanho acima do limite → 422/413.

### Nulls/Empty

- Body vazio → 400.
- Body `{}` → 400 ou 422 com indicação de campos obrigatórios.
- recipient_id ausente → 422 (RECIPIENT_ID_REQUIRED).
- recipient_id = null → 422.
- recipient_id = "" → 422.
- amount ausente → 422 (AMOUNT_REQUIRED).
- amount = null → 422.

### Formatos / Caracteres

- recipient_id não UUID (ex.: "abc", "123") → 422.
- recipient_id com caracteres especiais (quotes, backslash) → 422.
- amount como string "10" → 422 (AMOUNT_INVALID_TYPE).
- amount booleano ou array → 422.
- Content-Type diferente de application/json → 415 ou 400.

### Failure Patterns

- Request sem JWT → 401.
- recipient_id de outro usuário (não tentar transferir “para si” se for regra) → conforme regra de negócio (200/201 ou 422).
- Uso de idempotency-key: mesma key duas vezes → mesma resposta (201 + mesmo transaction_id).
- Rate limiting: após N requests no período → 429.

---

## Checklist Baica Aplicado ao Requisito

- [ ] **Boundaries**: Teto máximo global para amount; tratamento de 0 e negativo; tipo e precisão de amount; tamanho máximo de recipient_id e idempotency-key.
- [ ] **Nulls/Empty**: recipient_id e amount obrigatórios; tratamento de null, ausente e string vazia; body vazio/incompleto.
- [ ] **Special Chars**: Formato de recipient_id (ex.: UUID); política para idempotency-key; queries parametrizadas/ORM.
- [ ] **Formatos**: Formato de recipient_id e amount no contrato; mensagens 422 por tipo; Content-Type obrigatório.
- [ ] **Failure Patterns**: Queries parametrizadas documentadas; rate limiting; idempotency por usuário; auth sempre pelo JWT; não vazar dados em log/erro.
- [ ] **Requisitos**: Gaps acima documentados no REQ_FINAL ou em anexo de validação.
- [ ] **Testes**: Casos para limites, null/vazio, formatos inválidos e idempotência/rate limit.

---

## Conclusão

A aplicação da heurística Baica ao requisito de Envio de QualiPoints permitiu identificar **gaps de validação** em todas as cinco dimensões (boundaries, nulls/vazios, caracteres especiais, formatos inválidos e padrões comuns de falha). O requisito já estabelece boas bases (JWT, atomicidade, idempotência, limites mínimos e máximos conceituais), mas precisa ser complementado com:

1. **Definições explícitas** de limites (teto global, tamanhos máximos, formato de identificadores).
2. **Comportamento documentado** para null, ausente e vazio em cada campo.
3. **Mensagens e códigos de erro** específicos para cada tipo de falha de validação.
4. **Regras de segurança** documentadas (queries parametrizadas, rate limiting, escopo de idempotency).

Incorporar essas regras ao requisito e ao contrato da API reduz riscos de comportamento indefinido, falhas em produção e vulnerabilidades, alinhando o sistema à heurística Baica desde a especificação.
