# Relatório VADER — Envio de QualiPoints entre Usuários

**Heurística:** VADER (Stuart Ashman)
**Requisito analisado:** `input/qualitpoints.md` — Requisito Revisado v2
**Contexto de entrada:** Épico + Tarefa (requisito detalhado com contrato de API)
**Solicitação:** Refinamento de demandas + Codificação
**Data da análise:** 2026-03-29

---

## Resumo Executivo

O requisito está bem estruturado e cobre boa parte das dimensões VADER. A análise identificou **gaps de especificação** e **riscos** que devem ser endereçados antes ou durante o desenvolvimento. Os principais pontos de atenção são: ausência de rate limiting no MVP, falta de especificação de formatos de validação para campos, e lacunas no tratamento de concorrência e expiração de idempotency keys.

---

## 1. Values (Valores e Validação de Inputs)

### O que o requisito define

| Campo | Tipo | Validação especificada |
|-------|------|----------------------|
| `amount` | integer | 1 a 10.000; zero e negativo rejeitados (400) |
| `recipientId` | string (ID de usuário) | Deve existir, estar ativo, diferente do remetente |
| `Authorization` | header Bearer token | Obrigatório |
| `Idempotency-Key` | header string | Obrigatória, única por intenção |
| `Content-Type` | header | `application/json` |

### Gaps identificados

| # | Gap | Risco | Recomendação |
|---|-----|-------|--------------|
| V1 | **Formato do `recipientId` não especificado** — o requisito diz "string (ID do usuário)" mas não define formato (UUID, numérico, alfanumérico) nem comprimento máximo. | Payloads com strings enormes ou caracteres especiais podem causar problemas de parsing, injeção ou consumo de memória. | Definir formato esperado (ex.: UUID v4, `^[a-f0-9-]{36}$`) e comprimento máximo. Rejeitar com 400 se fora do formato. |
| V2 | **Formato do `Idempotency-Key` não especificado** — diz "UUID" como exemplo, mas não define formato obrigatório nem comprimento máximo. | Chaves arbitrariamente longas podem consumir espaço em banco/cache. Chaves vazias ou com caracteres especiais podem causar comportamento inesperado. | Definir formato (UUID v4) e comprimento máximo (ex.: 128 chars). Rejeitar com 400 se vazio ou fora do formato. |
| V3 | **Tipo do `amount` — comportamento com decimal não especificado** — diz "inteiro" mas não define o que acontece se o cliente enviar `100.5` ou `"100"` (string). | Linguagens/frameworks podem aceitar silenciosamente `100.5` como `100` (truncamento) ou `"100"` como 100 (coerção), levando a comportamento inesperado. | Especificar: rejeitar com 400 se o valor não for estritamente inteiro. Definir comportamento para tipos incorretos (string, boolean, array). |
| V4 | **Campos extras no body não especificados** — não define se a API ignora ou rejeita campos desconhecidos. | Campos extras ignorados podem mascarar erros do cliente (ex.: `"ammount"` em vez de `"amount"`). | Definir política: rejeitar com 400 se houver campos desconhecidos no body (strict parsing), ou documentar que campos extras são ignorados. |
| V5 | **Body vazio ou malformado (JSON inválido)** — não especifica resposta para body vazio ou JSON malformado. | Pode resultar em erro 500 genérico em vez de 400 claro. | Especificar: JSON malformado ou body vazio → 400 com código `INVALID_REQUEST_BODY`. |
| V6 | **`amount` no limite máximo do tipo inteiro** — não define comportamento para valores acima de 10.000 mas dentro do range do inteiro (ex.: 99.999) vs. valores que excedem o tipo (ex.: `9999999999999999999`). | Overflow de inteiro pode causar comportamento indefinido dependendo da linguagem. | Validar range (1-10.000) antes de qualquer operação aritmética. Especificar tipo: int32 é suficiente. |

### Checklist Values

- [x] Tipo de dado definido para `amount` (integer)
- [x] Range definido (1 a 10.000)
- [x] Tratamento de zero e negativo definido
- [ ] Formato de `recipientId` não especificado
- [ ] Formato de `Idempotency-Key` não especificado
- [ ] Comportamento para tipos incorretos (string no lugar de integer) não definido
- [ ] Política para campos extras no body não definida
- [ ] Tratamento de body vazio/malformado não especificado

---

## 2. Authorization (Autorização)

### O que o requisito define

| Aspecto | Especificação |
|---------|--------------|
| Autenticação | Bearer token no header `Authorization` |
| Remetente | Identificado pelo token (não enviado no body) |
| Conta ativa | Remetente deve ter conta ativa |
| Destinatário ativo | Destinatário deve ter conta ativa e elegível |
| Self-transfer | Proibido (409) |
| 401 | Token ausente ou inválido |
| 403 | Conta bloqueada/não ativa |
| HTTPS | Obrigatório |

### Gaps identificados

| # | Gap | Risco | Recomendação |
|---|-----|-------|--------------|
| A1 | **Token expirado — código HTTP não especificado** — o requisito lista 401 para "não autenticado ou token inválido" mas não diferencia token expirado de token malformado. | O cliente não sabe se deve re-autenticar (token expirado) ou se há erro de implementação (token malformado). | Retornar 401 em ambos os casos, mas com códigos distintos: `TOKEN_EXPIRED` vs `TOKEN_INVALID`. Isso permite ao App redirecionar para login ou exibir erro técnico. |
| A2 | **Tipo de token não especificado** — diz "Bearer" mas não define JWT, opaque token ou outro. | Impacta decisões de implementação: JWT permite validação local (sem DB hit), opaque exige lookup. Afeta performance e arquitetura. | Definir tipo de token (recomendação: JWT com expiração curta + refresh token). |
| A3 | **Permissões/roles não detalhadas no token** — diz "perfil usuário final" mas não define se o token carrega role/scope. | Se no futuro admin/suporte forem adicionados, a ausência de role no token exige retrabalho. | Mesmo no MVP, incluir campo de role/scope no token (`user`, para futuro `admin`, `support`). Validar na API que role = `user` para este endpoint. |
| A4 | **Auditoria de tentativas — nível de detalhe** — diz "toda tentativa de envio (sucesso ou falha) deve ser registrada" mas não define campos do log nem retenção. | Logs insuficientes dificultam investigação de incidentes; logs excessivos consomem recursos. | Definir campos mínimos do log: `timestamp`, `userId`, `recipientId`, `amount`, `idempotencyKey`, `httpStatus`, `errorCode`, `ip`, `userAgent`. Definir retenção (ex.: 90 dias). |
| A5 | **Rate limiting de autenticação não mencionado** — não há proteção contra brute force de tokens. | Atacante pode tentar múltiplos tokens em sequência para encontrar um válido. | Implementar rate limiting por IP para requisições com 401 (ex.: max 10 falhas de autenticação por minuto por IP). |

### Checklist Authorization

- [x] Autenticação definida (Bearer token)
- [x] Remetente pelo token (não pelo body) — boa prática
- [x] Validação de conta ativa
- [x] Self-transfer bloqueado
- [x] HTTPS obrigatório
- [x] Códigos 401 e 403 definidos
- [ ] Tipo de token não especificado (JWT vs opaque)
- [ ] Token expirado não diferenciado de token inválido
- [ ] Campos de auditoria/log não especificados
- [ ] Rate limiting de autenticação não definido

---

## 3. Data Consistency (Consistência de Dados)

### O que o requisito define

| Aspecto | Especificação |
|---------|--------------|
| Atomicidade | Débito + crédito + registro em transação única de banco; rollback se qualquer parte falhar |
| Idempotência | `Idempotency-Key` obrigatória; mesma key → mesma resposta sem reprocessar |
| Concorrência | Transação atômica de banco evita saldo inconsistente |
| Fonte da verdade | Banco relacional (saldos + transações) |
| Consistência App/Painel | Mesma API/banco; polling ou refresh manual no MVP |

### Gaps identificados

| # | Gap | Risco | Recomendação |
|---|-----|-------|--------------|
| D1 | **Expiração da idempotency key não definida** — o requisito diz que a mesma key retorna o mesmo resultado, mas não define por quanto tempo a key é válida. | Keys acumulam infinitamente no banco/cache, consumindo recursos. Ou, se expiradas silenciosamente, o cliente pode receber resposta diferente da original após expiração. | Definir TTL para idempotency keys (ex.: 24h). Após expiração, a key pode ser reutilizada. Documentar comportamento: requisição com key expirada é tratada como nova. |
| D2 | **Armazenamento da idempotency key não especificado** — não define onde/como a key e seu resultado são armazenados. | Se em memória: perde-se em restart. Se em banco: qual tabela? Se em cache distribuído: qual estratégia? | Definir: tabela `idempotency_keys` com campos `key`, `user_id`, `response_status`, `response_body`, `created_at`, `expires_at`. Ou cache Redis com TTL. |
| D3 | **Mecanismo de lock para concorrência não especificado** — diz "transação de banco de dados" mas não define se usa `SELECT FOR UPDATE` (pessimistic), versionamento (optimistic) ou outro. | `SELECT FOR UPDATE` pode causar deadlocks em alta concorrência. Optimistic locking pode causar falhas frequentes sob carga. | Definir estratégia: pessimistic locking (`SELECT FOR UPDATE` no saldo do remetente) com timeout curto, ou optimistic locking com retry automático (max 3 tentativas). |
| D4 | **Ordem de lock não definida** — ao atualizar saldos de dois usuários, a ordem de lock importa para evitar deadlocks (ex.: usuário A envia para B enquanto B envia para A simultaneamente). | Deadlock: requisição 1 trava saldo de A e espera B; requisição 2 trava saldo de B e espera A. | Sempre adquirir locks em ordem determinística (ex.: pelo menor ID de usuário primeiro). Documentar essa decisão. |
| D5 | **Idempotency key + erro — comportamento ambíguo** — diz "retorna o mesmo resultado da primeira vez (200 ou erro)". Mas e se o primeiro erro foi 500 (transiente)? Retornar 500 para retries com mesma key impede o cliente de completar a operação. | Se a primeira tentativa falhou com 500 e a key "travou" esse resultado, o cliente precisa gerar nova key e tentar novamente, sem saber se a operação foi ou não processada. | Definir: para erros transientes (5xx), a idempotency key **não** deve ser "consumida". Apenas resultados definitivos (2xx, 4xx) são persistidos como resultado da key. Requisições com key que resultou em 5xx podem ser reprocessadas. |
| D6 | **Validação de saldo — momento exato** — diz "saldo ≥ valor" mas não define se a validação é feita antes do lock ou dentro da transação. | Se validar antes do lock e executar depois, o saldo pode ter mudado entre a verificação e o débito (TOCTOU — Time of Check to Time of Use). | Validar saldo **dentro** da transação, após adquirir lock no saldo do remetente. |

### Checklist Data Consistency

- [x] Atomicidade definida (transação de banco)
- [x] Idempotência definida (Idempotency-Key)
- [x] Fonte da verdade definida (banco relacional)
- [x] Rollback definido em caso de falha
- [ ] TTL da idempotency key não definido
- [ ] Armazenamento da idempotency key não especificado
- [ ] Mecanismo de lock (pessimistic/optimistic) não definido
- [ ] Ordem de lock para evitar deadlock não definida
- [ ] Comportamento de idempotency key em erro transiente (5xx) ambíguo
- [ ] Momento da validação de saldo (antes ou dentro do lock) não especificado

---

## 4. Error Handling (Tratamento de Erros)

### O que o requisito define

| Código HTTP | Situação | Código de erro |
|-------------|----------|---------------|
| 200 | Sucesso | — |
| 400 | Dados inválidos | `INVALID_AMOUNT` |
| 401 | Não autenticado | `UNAUTHORIZED` |
| 403 | Conta não ativa | `ACCOUNT_NOT_ACTIVE` |
| 404 | Destinatário não encontrado/inativo | `RECIPIENT_NOT_FOUND` |
| 409 | Self-transfer | `SELF_TRANSFER_NOT_ALLOWED` |
| 422 | Saldo insuficiente | `INSUFFICIENT_BALANCE` |
| 408/504 | Timeout | `REQUEST_TIMEOUT` |
| 5xx | Erro interno | `INTERNAL_ERROR` |

### Gaps identificados

| # | Gap | Risco | Recomendação |
|---|-----|-------|--------------|
| E1 | **Estrutura de erro padronizada incompleta** — os exemplos mostram `{ "code": "...", "message": "..." }` mas não definem se todos os erros seguem essa estrutura. Alguns exemplos incluem `currentBalance` (422), outros não. | Consumidores da API não podem confiar numa estrutura consistente para parsear erros. | Definir estrutura padrão: `{ "code": "string", "message": "string", "details": {} }`. O campo `details` é opcional e varia por erro (ex.: `currentBalance` no 422). Documentar que `code` e `message` estão **sempre** presentes. |
| E2 | **`Idempotency-Key` ausente — código HTTP não especificado** — o header é obrigatório mas não define o que acontece se não for enviado. | Sem tratamento, a API pode processar sem idempotência (risco de débito duplo) ou retornar 500 genérico. | Retornar 400 com código `MISSING_IDEMPOTENCY_KEY` e mensagem "Header Idempotency-Key é obrigatório." |
| E3 | **`Content-Type` incorreto — código HTTP não especificado** — o header é obrigatório (`application/json`) mas não define resposta para `text/plain` ou ausente. | Pode resultar em erro de parsing inesperado (500). | Retornar 415 (Unsupported Media Type) com código `UNSUPPORTED_MEDIA_TYPE`. |
| E4 | **Múltiplos erros de validação — comportamento não definido** — se `amount` é inválido E `recipientId` está ausente, a API retorna um erro ou todos? | Se retorna apenas o primeiro, o cliente corrige um campo, envia de novo e descobre outro erro — UX ruim. | Definir: retornar **todos** os erros de validação de uma vez. Usar array em `details`: `{ "code": "VALIDATION_ERROR", "message": "...", "details": { "fields": [{"field": "amount", "code": "INVALID_AMOUNT", "message": "..."}, ...] } }`. |
| E5 | **`recipientId` ausente — código de erro não especificado** — o requisito define `RECIPIENT_NOT_FOUND` (404) para destinatário inexistente, mas não define código para campo `recipientId` ausente no body. | Busca no banco com null/vazio pode causar erro inesperado. | Retornar 400 com código `MISSING_RECIPIENT_ID` (ou incluir no array de validação — ver E4). |
| E6 | **Exposição de `currentBalance` no erro 422** — marcado como "(opcional)" mas expor saldo em resposta de erro pode ser risco de segurança se a requisição for interceptada. | Informação financeira exposta. Mesmo com HTTPS, logs de proxy, ferramentas de debug do cliente ou cache podem registrar o body de erro. | Avaliar se `currentBalance` é necessário na resposta. Se sim, garantir que só é retornado quando a requisição é autenticada e o saldo pertence ao remetente (já garantido pelo token). Considerar omitir no MVP. |
| E7 | **408 vs 504 — ambiguidade** — o requisito lista "408/504" como alternativas, sem definir qual usar. | Semânticas diferentes: 408 = cliente demorou a enviar; 504 = gateway não recebeu resposta do upstream. Uso incorreto confunde clientes e ferramentas de monitoramento. | Definir: usar **504** quando a API (ou serviço interno) exceder o timeout de 10s. Reservar 408 apenas se houver timeout de recebimento do body do cliente. |

### Checklist Error Handling

- [x] Códigos HTTP definidos para cenários principais
- [x] Códigos de erro (`code`) definidos
- [x] Mensagens mapeadas para o App
- [x] Separação entre erro do cliente (4xx) e erro do servidor (5xx)
- [ ] Estrutura padronizada de resposta de erro não formalizada
- [ ] Idempotency-Key ausente não tratado
- [ ] Content-Type incorreto não tratado
- [ ] Múltiplos erros de validação simultâneos não definidos
- [ ] `recipientId` ausente no body não tratado explicitamente
- [ ] 408 vs 504 ambíguo

---

## 5. Rate Limiting & Resource Consumption (Limite de Taxa e Consumo de Recursos)

### O que o requisito define

| Aspecto | Especificação |
|---------|--------------|
| Rate limiting | **Backlog** — não implementado no MVP |
| Timeout | API responde em até 10 segundos |
| Paginação | Lista de transações: últimas 10, sem paginação |
| Limites por transação | Máximo 10.000 QualiPoints |
| Limites por período | Backlog |

### Gaps identificados

| # | Gap | Risco | Severidade | Recomendação |
|---|-----|-------|-----------|--------------|
| R1 | **Rate limiting ausente no MVP** — explicitamente postergado para backlog. | API vulnerável a abuso: um script pode enviar milhares de requisições por segundo, causando DoS, drenando saldo rapidamente (se comprometido) ou sobrecarregando o banco. | **Alto** | Implementar rate limiting básico mesmo no MVP: ex.: max 10 requisições/minuto por usuário no endpoint de transferência. É uma proteção fundamental. Comunicar via headers `X-RateLimit-Limit`, `X-RateLimit-Remaining`, `X-RateLimit-Reset` e código 429. |
| R2 | **Limite de transações por período ausente** — sem limite diário/mensal. | Conta comprometida pode ter todo o saldo drenado em segundos via múltiplas transações de 10.000. | **Alto** | Implementar ao menos um limite diário básico no MVP (ex.: max 50.000 QualiPoints/dia ou max 50 transações/dia). É uma camada de proteção contra fraude. |
| R3 | **Tamanho máximo do payload não definido** — não especifica limite para o body da requisição. | Payloads enormes (MBs) podem consumir memória e CPU do servidor. | **Médio** | Definir limite de payload (ex.: 1 KB para este endpoint — o body tem ~50 bytes). Configurar no servidor/load balancer. Retornar 413 (Payload Too Large) se excedido. |
| R4 | **Lista de transações sem paginação real** — diz "últimas 10", mas não define comportamento se o cliente enviar parâmetros de paginação ou se o requisito evoluir. | Sem estrutura de paginação desde o início, adicionar depois exige mudança no contrato da API (breaking change). | **Baixo** | Mesmo retornando apenas 10 fixos no MVP, já retornar com estrutura paginada: `{ "data": [...], "pagination": { "limit": 10, "hasMore": true/false } }`. Facilita evolução sem breaking change. |
| R5 | **Timeout do lado do servidor — mecanismo não especificado** — diz "responder em até 10 segundos" mas não define se é timeout do servidor, do load balancer ou configuração de connection pool. | Sem implementação explícita, requisições lentas podem ficar penduradas e esgotar threads/conexões. | Configurar timeout em múltiplas camadas: servidor (handler timeout = 10s), banco (query timeout = 5s), connection pool (max wait = 3s). |
| R6 | **Sem proteção contra account takeover em massa** — sem rate limiting + sem limite de transações = se um atacante comprometer um lote de contas, pode drenar todas rapidamente. | Fraude em escala. | **Alto** | Combinar R1 (rate limiting) + R2 (limite diário) + alerta para transações acima de threshold (ex.: > 5.000 QualiPoints ou > 10 transações/hora por conta). |

### Checklist Rate Limiting & Resource Consumption

- [x] Timeout definido (10 segundos)
- [x] Limite por transação definido (10.000)
- [ ] Rate limiting ausente no MVP — **risco alto**
- [ ] Limite por período ausente — **risco alto**
- [ ] Tamanho de payload não definido
- [ ] Paginação não estruturada (fixo em 10)
- [ ] Mecanismo de timeout no servidor não especificado

---

## Consolidação de Gaps por Severidade

### Críticos (devem ser endereçados antes do desenvolvimento)

| # | Dimensão | Gap | Justificativa |
|---|----------|-----|---------------|
| R1 | Rate Limiting | Rate limiting ausente no MVP | Proteção fundamental contra abuso e DoS |
| R2 | Rate Limiting | Limite de transações por período ausente | Proteção contra fraude e drenagem de saldo |
| D5 | Data Consistency | Idempotency key + erro 5xx — comportamento ambíguo | Pode travar operações para o cliente |
| D3 | Data Consistency | Mecanismo de lock não especificado | Deadlocks e race conditions em produção |

### Altos (devem ser endereçados durante o desenvolvimento)

| # | Dimensão | Gap | Justificativa |
|---|----------|-----|---------------|
| V1 | Values | Formato de `recipientId` não especificado | Risco de injeção e payloads maliciosos |
| V2 | Values | Formato de `Idempotency-Key` não especificado | Consumo de recursos com keys arbitrárias |
| D1 | Data Consistency | TTL da idempotency key não definido | Acúmulo infinito de keys |
| D4 | Data Consistency | Ordem de lock não definida | Deadlocks em transferências cruzadas |
| D6 | Data Consistency | Momento da validação de saldo não especificado | Vulnerabilidade TOCTOU |
| E2 | Error Handling | Idempotency-Key ausente não tratado | Risco de débito duplo |
| A5 | Authorization | Rate limiting de autenticação ausente | Brute force de tokens |

### Médios (recomendados para qualidade)

| # | Dimensão | Gap |
|---|----------|-----|
| V3 | Values | Tipo `amount` com decimal — comportamento indefinido |
| V4 | Values | Campos extras no body — política não definida |
| V5 | Values | Body vazio/malformado — resposta não definida |
| E1 | Error Handling | Estrutura de erro não padronizada formalmente |
| E4 | Error Handling | Múltiplos erros de validação — comportamento não definido |
| E7 | Error Handling | 408 vs 504 ambíguo |
| R3 | Rate Limiting | Tamanho máximo de payload não definido |
| R5 | Rate Limiting | Mecanismo de timeout no servidor não especificado |
| A1 | Authorization | Token expirado não diferenciado de inválido |
| A4 | Authorization | Campos de auditoria/log não especificados |

### Baixos (boas práticas)

| # | Dimensão | Gap |
|---|----------|-----|
| V6 | Values | Overflow de inteiro para valores extremos |
| A2 | Authorization | Tipo de token (JWT vs opaque) não definido |
| A3 | Authorization | Role/scope no token não definido |
| E3 | Error Handling | Content-Type incorreto não tratado |
| E5 | Error Handling | `recipientId` ausente — código específico não definido |
| E6 | Error Handling | Exposição de `currentBalance` no erro 422 |
| R4 | Rate Limiting | Paginação sem estrutura para evolução |

---

## Pontos Positivos do Requisito

O requisito demonstra maturidade em vários aspectos VADER:

1. **Atomicidade bem definida** — débito + crédito + registro em transação única com rollback.
2. **Idempotência como requisito obrigatório** — `Idempotency-Key` no header evita débito duplo.
3. **Remetente pelo token** — não confiar em dados do body para identificar o remetente (boa prática de segurança).
4. **Self-transfer bloqueado** — regra de negócio explícita com código de erro dedicado.
5. **Códigos HTTP bem mapeados** — cada cenário de erro tem código HTTP e código de erro internos.
6. **Mensagens mapeadas para o App** — tabela de código API → mensagem de usuário.
7. **Edge cases documentados** — seção dedicada com cenários tratados.
8. **HTTPS obrigatório** — segurança na comunicação.
9. **Validação server-side** — explicitamente definida como fonte da verdade.

---

## Recomendações para Implementação

### Ordem sugerida de implementação por dimensão VADER

1. **Authorization** — configurar autenticação JWT/Bearer e middleware de validação de conta ativa antes de qualquer lógica.
2. **Values** — implementar validação de input (tipos, formatos, ranges) como primeira camada no handler.
3. **Data Consistency** — implementar transação atômica com lock, idempotência com TTL.
4. **Error Handling** — padronizar estrutura de erro e garantir que nenhum cenário retorne 500 genérico.
5. **Rate Limiting** — adicionar rate limiting básico (por usuário e por IP) mesmo no MVP.

### Testes VADER recomendados para `POST /api/v1/transfers`

| Dimensão | Casos de teste prioritários |
|----------|-----------------------------|
| **Values** | amount=0, amount=-1, amount=10001, amount=null, amount="abc", amount=100.5, recipientId="" vazio, recipientId=null, body vazio, JSON malformado, campos extras |
| **Authorization** | sem header Auth, token inválido, token expirado, conta bloqueada, conta pendente |
| **Data Consistency** | mesma idempotency-key 2x (sucesso), mesma key após 5xx, duas transferências simultâneas do mesmo remetente, transferência cruzada A→B e B→A simultâneas, saldo exato (saldo=amount) |
| **Error Handling** | todos os códigos HTTP documentados, estrutura consistente em cada erro, sem stack trace em 5xx, múltiplos erros de validação |
| **Rate Limiting** | (quando implementado) exceder limite → 429, headers X-RateLimit presentes, retry após Retry-After |

---

*Análise gerada com a Heurística VADER — Stuart Ashman. Validando APIs para qualidade e escala.*
