# Análise Baica do Requisito REQ_INICIAL_V2

**Requisito:** Envio de QualiPoints entre usuários  
**Data da Análise:** 21 de fevereiro de 2026  
**Heurística:** Baica — Identificando e assegurando o core essencial e resiliente do código  
**Baseado em:** [REQ_INICIAL_V2.md](../../output/requisito-revisado/REQ_INICIAL_V2.md)

---

## 1. Valores Mínimos/Máximos (Boundaries)

### Achados

**✅ Bem definido:**
- Mínimo: 1 QualiPoint
- Máximo: 10.000 QualiPoints por transação
- Rejeição de zero e negativos está explícita

**⚠️ Gaps identificados:**

| Gap | Risco | Recomendação |
|-----|-------|--------------|
| **Saldo máximo do usuário não definido** | Overflow ao crédito acumulado; usuário recebe valores e saldo cresce indefinidamente. Impacto em cálculos, relatórios ou limite de armazenamento. | Definir saldo máximo (ex.: 999.999.999 QualiPoints) e documentar comportamento quando atingido (rejeitar recebimento ou alertar). |
| **Limite por dia/mês mencionado como "backlog"** | Usuário pode explorar múltiplos envios pequenos para contornar limite global; falta controle de taxa. | Documentar no requisito quando essa regra será implementada e qual será o limite inicial (ex.: 50 transações/dia, 100.000 máximo/mês). |
| **Tamanho máximo do ID do destinatário não especificado** | Cliente pode enviar ID muito grande; API deve validar e rejeitar. | Definir comprimento máximo do campo `recipientId` (ex.: "string de até 36 caracteres para UUID"). |
| **Número máximo de transações recentes não parametrizado** | Requisito fixa "últimas 10", mas sem justificativa. E se o usuário tiver >10 transações no mesmo segundo? | Clarificar: as 10 transações são por paginação ou sempre as últimas 10? Documentar ordenação de desempate (ex.: "por timestamp DESC, depois por transaction ID DESC"). |
| **Timeout de 10s é rígido** | Se API nunca responde em exatamente 10s, como diferencia timeout de falha lenta? | Especificar: é 10s para aceitar a requisição ou 10s para responder? Considerar timeout maior para operações atômicas (débito/crédito + insert). |

### Casos de Teste Essenciais (Boundaries)

```
✓ amount = 1 (válido, mínimo)
✓ amount = 10000 (válido, máximo)
✗ amount = 0 (inválido)
✗ amount = -1 (inválido)
✗ amount = 10001 (acima do máximo)
✗ amount = 999999999 (muito grande, potencial overflow)
✓ saldo_remetente = 10000, amount = 10000 (caso limite)
✗ saldo_remetente = 9999, amount = 10000 (insuficiente)
```

---

## 2. Valores Nulos/Vazios (Nulls/Empty)

### Achados

**✅ Bem coberto:**
- Requisição sem `Authorization` retorna 401
- Campo obrigatório `recipientId` e `amount` são explícitos no contrato

**⚠️ Gaps importantes:**

| Gap | Risco | Recomendação |
|-----|-------|--------------|
| **`recipientId` = null / vazio na requisição** | API deve rejeitar com 400 Bad Request; requisito não especifica mensagem clara. | Documentar: "Se `recipientId` for null, vazio ou apenas espaços, API retorna 400 com código `INVALID_RECIPIENT_FORMAT` e mensagem 'ID do destinatário é obrigatório.'" |
| **`amount` = null / não enviado** | Assumir zero? Rejeitar? Requisito não explicita. | Documentar: "Se `amount` for null ou omitido, API retorna 400 com código `INVALID_AMOUNT` e mensagem 'Valor é obrigatório e deve ser um número inteiro.'" |
| **`Idempotency-Key` vazio ou null** | Requisito diz "Idempotency-Key é obrigatório" mas não valida. Se cliente envia vazio, API rejeita? Com qual código? | Documentar: "Se `Idempotency-Key` for vazio, null ou duplicado em menos de X minutos sem sucesso prévio, API retorna 400 com código `INVALID_IDEMPOTENCY_KEY`." |
| **`Authorization` vazio (ex.: `Bearer `)** | API trata como "sem token" (401) ou tenta processar token vazio? | Esclarecer: token vazio é inválido (401), não processável. |
| **Corpo da requisição vazio ou mal-formado JSON** | Requisito assume JSON válido; erro de parsing não está mapeado. | Adicionar resposta: "Se JSON inválido, API retorna 400 com código `INVALID_JSON` e mensagem 'Corpo da requisição é um JSON inválido.'" |
| **Campo `recipientId` = usuário inexistente** | Mapeado (404), mas e se for array vazio `[]` ou objeto `{}`? | Adicionar validação de tipo: `recipientId` deve ser string, não array ou objeto. |
| **Resposta com campos opcionais ausentes** | Requisito não especifica se resposta de sucesso (200) sempre inclui `transactionId`, `completedAt`, etc. Se algum estiver null/ausente, cliente quebra. | Garantir que 200 sempre retorna todos os campos do contrato; opcionais marcam explicitamente com `"field": null` ou documentam omissão. |

### Casos de Teste Essenciais (Nulls/Empty)

```
✗ recipientId = null → 400 INVALID_RECIPIENT_FORMAT
✗ recipientId = "" → 400 INVALID_RECIPIENT_FORMAT
✗ recipientId = "   " (apenas espaços) → 400 INVALID_RECIPIENT_FORMAT
✗ amount = null → 400 INVALID_AMOUNT
✗ amount omitido do body → 400 INVALID_AMOUNT
✗ Idempotency-Key = "" → 400 INVALID_IDEMPOTENCY_KEY
✗ Authorization omitido → 401 UNAUTHORIZED
✗ Authorization = "Bearer " (vazio) → 401 UNAUTHORIZED
✗ Body = "{invalid json}" → 400 INVALID_JSON
```

---

## 3. Caracteres Especiais (Special Chars)

### Achados

**⚠️ Risco alto — gaps críticos:**

| Gap | Risco | Recomendação |
|-----|-------|--------------|
| **`recipientId` pode conter caracteres especiais, emojis, SQL/scripts** | Se `recipientId` é string sem validação, cliente pode enviar: `"'; DROP TABLE users; --"`, `"<script>alert('xss')</script>"`, emojis, null bytes. Bancos vulneráveis a injeção se usam concatenação. | **Validar whitelist:** `recipientId` deve conter **apenas** alfanuméricos + hífens/underscores (ex.: `^[a-zA-Z0-9_-]+$`). Descartar controles especiais (null bytes, newlines). |
| **Sem sanitização de entrada antes de query** | Requisito diz "validação server-side" mas não especifica se usa parameterização (ORM, prepared statements) ou concatenação. | **Mandatório:** Use ORM ou prepared statements (ex.: Sequelize, Hibernate, parameterized queries) para evitar SQL injection. Nunca concatene `recipientId` em query. |
| **`Idempotency-Key` não validado quanto a formato** | Se é UUID, cliente pode enviar qualquer string; se é SHA-256, requisito não especifica. Servidor aceita `"\x00\x01"` como chave? | Documentar: "Idempotency-Key deve ser string de até 255 caracteres, contendo apenas `[a-zA-Z0-9_-]`. Caracteres especiais e controles são rejeitados com 400." |
| **Log de auditoria pode armazenar entrada sem escape** | Requisito menciona "auditoria: toda tentativa de envio" mas não diz se loga a entrada bruta ou escapada. | Garantir: logs usam estrutura JSON com campos separados (não concatenação) para evitar log injection. Ex.: `{ "user": "alice", "recipient": "bob", "amount": 100 }` em vez de interpolação. |
| **Resposta JSON pode conter `recipientId` de entrada sem escape** | Se API eco a entrada na resposta sem sanitização e cliente renderiza como HTML, risco de XSS no painel web. | Garantir: resposta JSON sempre escapa strings corretamente; cliente renderiza como texto (não HTML) ou usa library de sanitização (ex.: DOMPurify). |

### Casos de Teste Essenciais (Special Chars)

```
✗ recipientId = "'; DROP TABLE users; --" → 400 INVALID_RECIPIENT_FORMAT
✗ recipientId = "<script>alert('xss')</script>" → 400 INVALID_RECIPIENT_FORMAT
✗ recipientId = "user@example.com" (contém @) → 400 ou aceitar (depende do padrão de ID)
✗ recipientId = "用户1" (caracteres Unicode fora de ASCII) → 400 ou aceitar (depende da política de ID)
✗ recipientId = "user\x00hacker" (null byte) → 400 INVALID_RECIPIENT_FORMAT
✗ Idempotency-Key = "'; DROP TABLE; --" → 400 INVALID_IDEMPOTENCY_KEY
✗ Idempotency-Key = "uuïd-with-accents" (acentos) → 400 ou aceitar
✓ Idempotency-Key = "550e8400-e29b-41d4-a716-446655440000" (UUID válido) → aceitar
```

---

## 4. Formatos Inválidos (Invalid Formats)

### Achados

**✅ Bem coberto:**
- `amount` deve ser inteiro (não string, não decimal)
- Headers obrigatórios especificados

**⚠️ Gaps:**

| Gap | Risco | Recomendação |
|-----|-------|--------------|
| **Formato de `recipientId` não padronizado** | Requisito não define: é UUID? E-mail? Número inteiro? String alfanumérica? Se não padronizar, diferentes clientes enviam formatos diferentes. | Documentar: "`recipientId` deve ser UUID (formato `xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx`) **OU** detalhar o padrão local (ex.: 'ID numérico de até 20 dígitos'). Se inválido, retorna 400 `INVALID_RECIPIENT_FORMAT`." |
| **`amount` aceita string ou apenas number?** | Requisito especifica `"integer"` no contrato mas não diz se API aceita `"amount": "100"` (string) ou só `"amount": 100` (number JSON). | Esclarecer: "Se `amount` for string, API tenta converter (ex.: `"100"` → 100) ou rejeita com 400 `INVALID_AMOUNT`? Se é float `100.5`, arredonda, trunca ou rejeita?" |
| **`Authorization` aceita apenas Bearer ou outras schemes?** | Requisito diz `Bearer <token>` mas não trata `Basic`, `Digest` ou formato malformado. | Documentar: "Apenas `Authorization: Bearer <token>` é suportado. Outros schemes retornam 401. Sem o prefixo `Bearer `, retorna 401 `INVALID_AUTHORIZATION_FORMAT`." |
| **Timestamps na resposta — qual formato?** | Campo `completedAt` é "ISO8601" mas não especifica: com Z (UTC)? Milissegundos? Timezone do usuário? | Documentar: "Timestamps são sempre ISO8601 UTC com milissegundos: `2025-02-21T15:30:45.123Z`." |
| **Response Content-Type sempre JSON?** | Requisito menciona `Content-Type: application/json` para requisição mas não garante para resposta. | Documentar: "Resposta HTTP sempre é `Content-Type: application/json`, mesmo em erro." |
| **Caracteres de quebra de linha em `recipientId`** | Cliente envia `"recipientId": "user1\nuser2"` — é múltiplos IDs ou inválido? | Rejeitar: `recipientId` não pode conter `\n`, `\r`, `\t`. Validar com regex `^[a-zA-Z0-9_-]+$` ou equivalente conforme padrão de ID. |

### Casos de Teste Essenciais (Invalid Formats)

```
✗ amount = "100" (string em vez de number) → aceitar com conversão ou 400?
✗ amount = 100.5 (float em vez de int) → rejeitar com 400 ou arredondar?
✗ amount = "abc" (não numérico) → 400 INVALID_AMOUNT
✗ Authorization = "Basic base64..." (scheme errado) → 401 INVALID_AUTHORIZATION_FORMAT
✗ Authorization = "Bearertoken" (sem espaço) → 401 INVALID_AUTHORIZATION_FORMAT
✗ recipientId = "user\n@admin" (contém newline) → 400 INVALID_RECIPIENT_FORMAT
✗ recipientId = "" (apenas espaço) → 400 INVALID_RECIPIENT_FORMAT
✓ amount = 100 (integer válido) → aceitar
✓ Authorization = "Bearer eyJ..." (formato correto) → processar
```

---

## 5. Padrões Comuns de Falha (Common Failure Patterns)

### Achados

**⚠️ Riscos críticos identificados:**

| Padrão | Risco | Recomendação |
|--------|-------|--------------|
| **SQL Injection via `recipientId`** | Se API faz query como `SELECT * FROM users WHERE id = '${recipientId}'`, atacante envia `"'; DROP TABLE transfers; --"`. | ✅ **Essencial:** Use **prepared statements** ou ORM com parameterização. **Nunca** concatene entrada em SQL. Exemplo seguro: `db.query("SELECT * FROM users WHERE id = ?", [recipientId])`. |
| **XSS armazenado no histórico de transações** | Se `recipientId` (ou valor de entrada do cliente) é armazenado sem escape e exibido no painel web sem sanitização, atacante injeta `<img src=x onerror="...">`. | ✅ Garantir: entrada é escapada/sanitizada no banco (ou em estrutura JSON) e exibido como texto no painel (não como HTML) ou com Content-Security-Policy (CSP) rígida. |
| **CSRF em POST de transferência** | Requisito não menciona proteção CSRF. Se usuário está logado no painel e visita site malicioso, site pode fazer POST não autorizado. | ✅ Implementar: CSRF token no formulário/App ou usar **SameSite=Strict** em cookie de sessão. Requisito atual carece dessa especificação. |
| **Escalação de privilégio via `recipientId`** | Se `recipientId` não é validado e API confia em outro campo do token para permissão, atacante pode enviar para conta de admin. | ✅ Garantir: servidor **sempre** valida que `recipientId` existe e está ativo **independentemente** do token. Nunca confie em campo do cliente. |
| **Negação de serviço (DoS) — múltiplos envios rápidos** | Sem rate limit global, usuário (ou bot) pode enviar 1000 requisições/segundo, sobrecarregando BD. Requisito menciona rate limit como "backlog". | ✅ No MVP: implementar rate limit básico (ex.: máx 10 transações/minuto por usuário). Documentar no requisito. Sem isso, sistema é vulnerável a DoS. |
| **Negação de serviço — payload grande** | Cliente envia `recipientId` com 1 MB de dados; API processa e falha ou lentifica. | ✅ Validar: máximo de caracteres em `recipientId` (ex.: 36 para UUID) e tamanho total do body (ex.: 1 KB). Retornar 413 Payload Too Large se exceder. |
| **Timing attack em validação de saldo** | Se resposta "insuficiente" (422) é mais rápida que "sucesso" (200), atacante pode deduzir saldo. | ⚠️ Risco baixo, mas: considerar tempo de resposta consistente ou não expor `currentBalance` em erro 422. Requisito atual mostra saldo em erro — rever se é intencional. |
| **Idempotency key reusável em múltiplas contas** | Se dois usuários usam mesma Idempotency-Key, servidor pode retornar resultado de um para o outro. | ✅ Garantir: Idempotency-Key é vinculado a **(usuário, chave)** — mesmo se dois usuários usam `abc123`, são armazenados separadamente. |
| **Replay attack — não validar timestamp** | Se API não valida que Idempotency-Key é "recente", atacante pode armazenar uma resposta bem-sucedida e replayá-la dias depois. | ⚠️ Considerar: expiração de Idempotency-Key (ex.: válida por 24h) para evitar reutilização indefinida. Requisito atual não especifica. |
| **Inconsistência transacional — débito sem crédito** | Se processo falha entre débito e crédito (ex.: crash do servidor), saldo fica inconsistente. | ✅ Obrigatório: usar **transação ACID** no banco — débito, crédito e insert de log ocorrem atomicamente; se qualquer falha, rollback. Requisito menciona isso, mas implementação deve garantir. |
| **Acesso não autorizado ao histórico de outro usuário** | API retorna último 10 transações — mas verifica que usuário é remetente **ou** destinatário? Se não, usuário A vê transações de usuário B. | ✅ Essencial: filtrar histórico por `(sender_id = current_user) OR (recipient_id = current_user)`. Requisito diz "apenas as próprias transações" mas não especifica query. |

### Casos de Teste Essenciais (Common Failure Patterns)

```
✗ recipientId = "'; DROP TABLE users; --" (SQL injection) → 400 INVALID_RECIPIENT_FORMAT
✗ recipientId = "<img src=x onerror='alert(1)'>" (XSS armazenado) → 400 INVALID_RECIPIENT_FORMAT ou escape antes de armazenar
✗ 1000 requisições/segundo de mesmo usuário (DoS) → rate limit retorna 429 Too Many Requests
✗ recipientId = "a" * 1000000 (payload grande) → 413 Payload Too Large
✗ Dois usuários usam mesma Idempotency-Key → cada um recebe seu próprio resultado (não cruzado)
✗ Usuário A tenta GET /api/v1/transfers?user=B (ler histórico de outro) → 403 Forbidden (filtra no servidor)
✗ CSRF: formulário malicioso tenta POST sem CSRF token → rejected ou 403 (conforme implementação)
✓ Requisição dentro de rate limit → processada
✓ Idempotency-Key expirado após 24h → pode reutilizar chave para novo envio
```

---

## 📋 Checklist de Análise Baica

- [ ] **Boundaries**: Máximo saldo do usuário não definido; limite por dia/mês é "backlog"
- [ ] **Nulls/Empty**: Mensagens de erro para campos obrigatórios ausentes (recipientId, amount) devem ser documentadas
- [ ] **Special Chars**: ⚠️ **Crítico** — Validar `recipientId` com whitelist; usar prepared statements/ORM; não concatenar em SQL
- [ ] **Formatos**: Padrão de `recipientId` (UUID? numérico?) deve ser especificado; conversão de `amount` (string → int) deve ser explícita
- [ ] **Common Failures**: ⚠️ **Crítico** — Rate limit no MVP é ausente (documenta como backlog); CSRF não mencionado; Idempotency-Key vinculada a (user, key)?; acesso a histórico validado?
- [ ] **Requisitos**: Gaps em validação documentados e comunicados ao time de dev antes de implementar
- [ ] **Testes**: Incluir casos para limites, null/vazio, caracteres especiais, formatos inválidos e padrões de falha

---

## 🎯 Recomendações Prioritárias (por risco)

### 🔴 Crítico — Não iniciar desenvolvimento sem clareza:

1. **SQL Injection:** Documentar que `recipientId` é validado com whitelist e query usa prepared statements/ORM.
2. **Rate Limiting:** Definir limite por minuto/hora no MVP (ex.: máx 10 transações/minuto por usuário) ou deixar claro como será escalável sem limite.
3. **Idempotency:** Esclarecer que chave é vinculada a **(user_id, idempotency_key)** para evitar cruzamento entre usuários.
4. **Acesso ao Histórico:** Documenter filtro exato — API valida que usuário logado é remetente **OU** destinatário antes de retornar cada transação.

### 🟡 Alto — Incorporar antes de liberar para QA:

5. **Formato de `recipientId`:** Padronizar (UUID, email, numérico?) e documentar regex de validação.
6. **Validação de Tipos:** Se `amount` vem como string `"100"`, API aceita com conversão ou rejeita com 400?
7. **Mensagens de Erro:** Mapear todas as validações (campos obrigatórios, formato, range) com códigos e mensagens específicas.
8. **CSRF Protection:** Adicionar ao escopo do MVP ou documentar como será mitigado (ex.: SameSite=Strict em cookies).

### 🟢 Médio — Refinar antes de release:

9. **Saldo Máximo:** Definir limite e comportamento quando atingido.
10. **Timestamp ISO8601:** Especificar com precisão (UTC, milissegundos, etc.).
11. **Logging:** Garantir que logs usam estrutura JSON, não concatenação, para evitar log injection.

---

## Resumo Executivo

O requisito REQ_INICIAL_V2 apresenta uma **base sólida** com atomicidade, autenticação e idempotência bem articuladas. No entanto, existem **gaps críticos de validação e segurança** que devem ser resolvidos **antes da implementação** para garantir robustez:

| Dimensão | Status | Prioridade | Ação |
|----------|--------|-----------|------|
| Boundaries | ⚠️ Parcial | 🟡 Alto | Definir saldo máximo e limite por período |
| Nulls/Empty | ⚠️ Parcial | 🟡 Alto | Documentar tratamento de campos obrigatórios |
| Special Chars | 🔴 Crítico | 🔴 Crítico | Validar whitelist; usar parameterização em SQL |
| Formatos | ⚠️ Parcial | 🟡 Alto | Padronizar `recipientId`; esclarecer conversão de `amount` |
| Common Failures | 🔴 Crítico | 🔴 Crítico | Rate limit, CSRF, Idempotency (user, key), Acesso a histórico |

**Recomendação:** Incorporar gaps críticos (especialmente segurança e rate limit) no escopo do MVP antes de iniciar desenvolvimento.
