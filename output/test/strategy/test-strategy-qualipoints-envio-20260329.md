# Relatório de Estratégia de Testes: Envio de QualiPoints entre Usuários

**Data**: 2026-03-29
**Requisito**: REQ_INICIAL v2 — Envio de QualiPoints
**Funcionalidade**: Transferência síncrona de QualiPoints entre usuários via API REST, com idempotência, atomicidade e lista de transações recentes.

---

## Análise de Heurísticas

### Heurísticas Identificadas

#### Desenvolvimento
- [x] Baica (Boundaries, Nulls, Special Chars, Formats, Failure Patterns)
- [x] VADER (Values, Authorization, Data Consistency, Error Handling, Rate Limiting)

#### Arquitetura de Código
- [x] CRUD (Create, Read, Update, Delete)
- [x] Count (0, 1, Muitos)

#### Design de Código
- [x] Multi-User (Colisão, Sessão Dupla, Inventário Crítico, UX de Conflito)

#### QA & Testes
- [x] FAILURE (Functional, Appropriate, Impact, Log, UI, Recovery, Emotions)
- [x] EMOTIONS (9 emoções: Alegria, Tristeza, Raiva, Medo, Nojinho, Surpresa, Confiança, Antecipação, Desconfiança)
- [x] FLOOD (picos, concorrência, entrada massiva, degradação, estabilidade)

#### Pós-Deploy
- [x] RCRCRC (Recent, Core, Risky, Configuration-sensitive, Conformance, Complex)
- [x] SFDPOT (Sources, Formats, Dependencies, Pace, Environment, Other, Time)

### Justificativa por Heurística

| Heurística | Justificativa |
|------------|---------------|
| **Baica** | API recebe inputs críticos (amount, recipientId, headers). Necessita validação rigorosa de boundaries (1–10.000), nulls, formatos e padrões de falha (injection, XSS). |
| **VADER** | Endpoint REST `POST /api/v1/transfers` com autenticação Bearer, idempotency-key, contratos de erro. Cobre as 5 dimensões da qualidade de API. |
| **CRUD** | Operação Create (transferência + registro) e Read (lista de transações recentes). Ciclo de vida dos dados com atomicidade e controle de acesso. |
| **Count** | Volumes variáveis: saldo 0/1/N, lista 0/1/10 transações, idempotency-keys acumuladas. Comportamento nos extremos de volume. |
| **Multi-User** | Saldo é recurso compartilhado; transferências simultâneas geram race conditions, deadlocks (A→B e B→A), sessões duplas (mesmo usuário em dois devices). |
| **FAILURE** | Operação financeira exige rollback correto, recovery via retry, error handling em todas as camadas (API, banco, rede). |
| **EMOTIONS** | Transferência de pontos gera ansiedade (medo de débito duplo), frustração (erro sem clareza), confiança (confirmação imediata). UX emocional impacta adoção. |
| **FLOOD** | Sem rate limiting no MVP; picos de requisições podem causar degradação. Concorrência em saldo é ponto crítico sob carga. |
| **RCRCRC** | Feature nova (Recent), operação financeira core (Core), concorrência sem lock definido (Risky), timeout configurável (Configuration-sensitive), HTTPS obrigatório (Conformance), atomicidade multi-tabela (Complex). |
| **SFDPOT** | Sources: App + Painel. Formats: JSON com inteiro. Dependencies: banco relacional, auth. Pace: síncrono 10s timeout. Environment: HTTPS prod. Time: idempotency-key sem TTL definido. |

---

## Estratégia por Camada

### Distribuição Recomendada

| Camada | Testes | % | Foco |
|--------|--------|---|------|
| Unitários | 38 | 55% | Validações de input, regras de negócio, lógica de idempotência |
| Integração | 15 | 22% | Atomicidade no banco, concorrência/locks, queries de listagem |
| Serviço/API | 9 | 13% | Endpoint completo, autenticação, contratos de erro, headers |
| Componentes | 4 | 6% | UI de envio, feedback visual, estados |
| E2E | 3 | 4% | Fluxo completo, retry, consistência cross-channel |
| **Total** | **69** | **100%** | |

### Justificativa da Distribuição

O requisito combina **validação de input** (peso em unitários), **persistência atômica e concorrência** (peso em integração), **contrato HTTP completo** (peso em serviço) e **jornada do usuário** (componentes + E2E). A função de validação de amount e recipientId concentra a maior parte dos cenários em unitários. A atomicidade de débito+crédito+registro e os race conditions exigem integração real com banco. O endpoint `POST /api/v1/transfers` com seus 8 códigos de resposta e headers obrigatórios demanda testes de serviço. UI e E2E cobrem a experiência emocional e fluxos críticos de ponta a ponta.

---

## Casos de Teste Detalhados

### Testes Unitários (38)

#### Módulo: validateAmount
**Heurística: Baica — Boundaries**

1. **UT-AMT-01**: Rejeitar amount = 0
   - Input: `{ amount: 0 }`
   - Output esperado: `{ valid: false, code: "INVALID_AMOUNT" }`
   - Heurística: Baica — Boundaries (limite inferior)
   - Prioridade: P0

2. **UT-AMT-02**: Rejeitar amount negativo
   - Input: `{ amount: -1 }`
   - Output esperado: `{ valid: false, code: "INVALID_AMOUNT" }`
   - Heurística: Baica — Boundaries
   - Prioridade: P0

3. **UT-AMT-03**: Aceitar amount = 1 (mínimo válido)
   - Input: `{ amount: 1 }`
   - Output esperado: `{ valid: true }`
   - Heurística: Baica — Boundaries (limite inferior exato)
   - Prioridade: P0

4. **UT-AMT-04**: Aceitar amount = 10000 (máximo válido)
   - Input: `{ amount: 10000 }`
   - Output esperado: `{ valid: true }`
   - Heurística: Baica — Boundaries (limite superior exato)
   - Prioridade: P0

5. **UT-AMT-05**: Rejeitar amount = 10001
   - Input: `{ amount: 10001 }`
   - Output esperado: `{ valid: false, code: "INVALID_AMOUNT" }`
   - Heurística: Baica — Boundaries (acima do máximo)
   - Prioridade: P0

6. **UT-AMT-06**: Rejeitar amount decimal (ex: 99.5)
   - Input: `{ amount: 99.5 }`
   - Output esperado: `{ valid: false, code: "INVALID_AMOUNT" }`
   - Heurística: Baica — Formats (inteiro obrigatório)
   - Prioridade: P1

7. **UT-AMT-07**: Aceitar amount intermediário (ex: 5000)
   - Input: `{ amount: 5000 }`
   - Output esperado: `{ valid: true }`
   - Heurística: Count — 1 (valor típico)
   - Prioridade: P2

**Heurística: Baica — Nulls/Empty**

8. **UT-AMT-08**: Rejeitar amount = null
   - Input: `{ amount: null }`
   - Output esperado: `{ valid: false, code: "INVALID_AMOUNT" }`
   - Heurística: Baica — Nulls
   - Prioridade: P0

9. **UT-AMT-09**: Rejeitar amount = undefined (campo ausente)
   - Input: `{}` (sem campo amount)
   - Output esperado: `{ valid: false, code: "INVALID_AMOUNT" }`
   - Heurística: Baica — Nulls
   - Prioridade: P0

10. **UT-AMT-10**: Rejeitar amount = "" (string vazia)
    - Input: `{ amount: "" }`
    - Output esperado: `{ valid: false, code: "INVALID_AMOUNT" }`
    - Heurística: Baica — Nulls/Formats
    - Prioridade: P1

**Heurística: Baica — Formats / Special Chars**

11. **UT-AMT-11**: Rejeitar amount como string numérica ("100")
    - Input: `{ amount: "100" }`
    - Output esperado: `{ valid: false, code: "INVALID_AMOUNT" }` (ou coerção controlada)
    - Heurística: Baica — Formats (type coercion)
    - Prioridade: P1

12. **UT-AMT-12**: Rejeitar amount com overflow (Number.MAX_SAFE_INTEGER)
    - Input: `{ amount: 9007199254740991 }`
    - Output esperado: `{ valid: false, code: "INVALID_AMOUNT" }`
    - Heurística: Baica — Boundaries (overflow)
    - Prioridade: P1

#### Módulo: validateRecipient
**Heurística: Baica — Nulls + VADER — Values**

13. **UT-RCP-01**: Rejeitar recipientId = null
    - Input: `{ recipientId: null }`
    - Output esperado: `{ valid: false, code: "RECIPIENT_NOT_FOUND" }`
    - Heurística: Baica — Nulls
    - Prioridade: P0

14. **UT-RCP-02**: Rejeitar recipientId = "" (vazio)
    - Input: `{ recipientId: "" }`
    - Output esperado: `{ valid: false, code: "RECIPIENT_NOT_FOUND" }`
    - Heurística: Baica — Nulls
    - Prioridade: P0

15. **UT-RCP-03**: Rejeitar recipientId = senderId (auto-envio)
    - Input: `{ recipientId: "user-123", senderId: "user-123" }`
    - Output esperado: `{ valid: false, code: "SELF_TRANSFER_NOT_ALLOWED" }`
    - Heurística: VADER — Values (regra de negócio)
    - Prioridade: P0

16. **UT-RCP-04**: Rejeitar recipientId com caracteres especiais (SQL injection)
    - Input: `{ recipientId: "'; DROP TABLE users;--" }`
    - Output esperado: Rejeição ou sanitização segura
    - Heurística: Baica — Special Chars (injection)
    - Prioridade: P0

17. **UT-RCP-05**: Rejeitar recipientId inexistente
    - Input: `{ recipientId: "user-nonexistent" }` (mock: não existe)
    - Output esperado: `{ valid: false, code: "RECIPIENT_NOT_FOUND" }`
    - Heurística: VADER — Values
    - Prioridade: P0

18. **UT-RCP-06**: Rejeitar recipientId com conta inativa/bloqueada
    - Input: `{ recipientId: "user-blocked" }` (mock: status bloqueado)
    - Output esperado: `{ valid: false, code: "RECIPIENT_NOT_FOUND" }`
    - Heurística: VADER — Authorization
    - Prioridade: P1

#### Módulo: validateBalance
**Heurística: Count — 0/1/N + Baica — Boundaries**

19. **UT-BAL-01**: Rejeitar envio quando saldo = 0
    - Input: saldo remetente = 0, amount = 1
    - Output esperado: `{ valid: false, code: "INSUFFICIENT_BALANCE" }`
    - Heurística: Count — Zero
    - Prioridade: P0

20. **UT-BAL-02**: Aceitar envio quando saldo = amount (exato)
    - Input: saldo = 100, amount = 100
    - Output esperado: `{ valid: true }`
    - Heurística: Baica — Boundaries (limite exato)
    - Prioridade: P0

21. **UT-BAL-03**: Rejeitar envio quando saldo < amount
    - Input: saldo = 50, amount = 100
    - Output esperado: `{ valid: false, code: "INSUFFICIENT_BALANCE" }`
    - Heurística: Count — Boundary
    - Prioridade: P0

22. **UT-BAL-04**: Aceitar envio quando saldo > amount
    - Input: saldo = 10000, amount = 1
    - Output esperado: `{ valid: true }`
    - Heurística: Count — Muitos
    - Prioridade: P1

#### Módulo: validateSender
**Heurística: VADER — Authorization**

23. **UT-SND-01**: Rejeitar remetente com conta inativa
    - Input: sender.status = "blocked"
    - Output esperado: `{ valid: false, code: "ACCOUNT_NOT_ACTIVE" }`
    - Heurística: VADER — Authorization
    - Prioridade: P0

24. **UT-SND-02**: Rejeitar remetente com conta pendente
    - Input: sender.status = "pending"
    - Output esperado: `{ valid: false, code: "ACCOUNT_NOT_ACTIVE" }`
    - Heurística: VADER — Authorization
    - Prioridade: P1

25. **UT-SND-03**: Aceitar remetente com conta ativa
    - Input: sender.status = "active"
    - Output esperado: `{ valid: true }`
    - Heurística: VADER — Authorization
    - Prioridade: P0

#### Módulo: TransferService (lógica de negócio com mocks)
**Heurística: CRUD — Create + FAILURE — Functional**

26. **UT-TRF-01**: Transferência bem-sucedida debita remetente e credita destinatário
    - Input: sender.balance=500, amount=100, recipient válido
    - Output esperado: sender.balance=400, recipient.balance+=100, transação registrada
    - Heurística: CRUD — Create (operação completa)
    - Prioridade: P0

27. **UT-TRF-02**: Transferência falha se qualquer etapa falhar (rollback)
    - Input: mock de falha no crédito ao destinatário
    - Output esperado: débito revertido, nenhuma transação registrada, erro retornado
    - Heurística: FAILURE — Functional (rollback)
    - Prioridade: P0

28. **UT-TRF-03**: Idempotency-key já processada retorna mesmo resultado
    - Input: idempotencyKey = "key-123" (já existe com resultado 200)
    - Output esperado: mesmo response da primeira execução, sem reprocessar
    - Heurística: VADER — Data Consistency (idempotência)
    - Prioridade: P0

29. **UT-TRF-04**: Idempotency-key com erro anterior retorna mesmo erro
    - Input: idempotencyKey = "key-456" (já existe com resultado 422)
    - Output esperado: mesmo erro 422 da primeira execução
    - Heurística: VADER — Data Consistency
    - Prioridade: P1

#### Módulo: validateHeaders
**Heurística: VADER — Error Handling**

30. **UT-HDR-01**: Rejeitar requisição sem Authorization header
    - Input: headers sem Authorization
    - Output esperado: 401 Unauthorized
    - Heurística: VADER — Authorization
    - Prioridade: P0

31. **UT-HDR-02**: Rejeitar requisição sem Idempotency-Key header
    - Input: headers sem Idempotency-Key
    - Output esperado: 400 Bad Request
    - Heurística: VADER — Error Handling
    - Prioridade: P0

32. **UT-HDR-03**: Rejeitar requisição com Content-Type != application/json
    - Input: Content-Type: text/plain
    - Output esperado: 400 ou 415 Unsupported Media Type
    - Heurística: VADER — Error Handling
    - Prioridade: P1

33. **UT-HDR-04**: Rejeitar token expirado
    - Input: Authorization: Bearer expired-token
    - Output esperado: 401 Unauthorized
    - Heurística: VADER — Authorization
    - Prioridade: P0

34. **UT-HDR-05**: Rejeitar token inválido (formato incorreto)
    - Input: Authorization: Bearer malformed
    - Output esperado: 401 Unauthorized
    - Heurística: VADER — Authorization
    - Prioridade: P1

#### Módulo: listRecentTransactions
**Heurística: CRUD — Read + Count — 0/1/N**

35. **UT-LST-01**: Retornar lista vazia quando usuário não tem transações
    - Input: userId com 0 transações
    - Output esperado: `{ transactions: [], count: 0 }`
    - Heurística: Count — Zero
    - Prioridade: P1

36. **UT-LST-02**: Retornar 1 transação quando usuário tem exatamente 1
    - Input: userId com 1 transação
    - Output esperado: `{ transactions: [tx1], count: 1 }`
    - Heurística: Count — Um
    - Prioridade: P1

37. **UT-LST-03**: Retornar no máximo 10 transações ordenadas por data desc
    - Input: userId com 15 transações
    - Output esperado: 10 transações, mais recente primeiro
    - Heurística: Count — Muitos + CRUD — Read
    - Prioridade: P0

38. **UT-LST-04**: Retornar apenas transações do usuário logado
    - Input: userId = "user-A", existem transações de user-A e user-B
    - Output esperado: apenas transações onde user-A é remetente ou destinatário
    - Heurística: CRUD — Read (controle de acesso)
    - Prioridade: P0

---

### Testes de Integração (15)

#### Módulo: TransferRepository (banco de dados)
**Heurística: CRUD — Create + Multi-User — Colisão**

1. **IT-TRF-01**: Transferência atômica persiste débito + crédito + registro em uma transação
   - Input: sender.balance=500, amount=100, recipient válido
   - Output esperado: 3 operações commitadas; consulta confirma saldos e registro
   - Heurística: CRUD — Create (atomicidade)
   - Prioridade: P0

2. **IT-TRF-02**: Rollback completo quando crédito ao destinatário falha
   - Input: simular falha no UPDATE do destinatário
   - Output esperado: saldo do remetente inalterado, sem registro de transação
   - Heurística: FAILURE — Functional (rollback real no banco)
   - Prioridade: P0

3. **IT-TRF-03**: Rollback completo quando registro da transação falha
   - Input: simular falha no INSERT da transação
   - Output esperado: saldos inalterados
   - Heurística: FAILURE — Functional
   - Prioridade: P0

4. **IT-TRF-04**: Idempotency-key impede débito duplo no banco
   - Input: mesma idempotencyKey enviada 2x concorrentemente
   - Output esperado: apenas 1 débito efetivado
   - Heurística: VADER — Data Consistency (idempotência real)
   - Prioridade: P0

**Heurística: Multi-User — Colisão + Count — Muitos**

5. **IT-MU-01**: Race condition — 2 envios simultâneos do mesmo remetente com saldo insuficiente para ambos
   - Input: saldo=150, envio1=100, envio2=100, concorrentes
   - Output esperado: 1 sucesso + 1 INSUFFICIENT_BALANCE; saldo final consistente
   - Heurística: Multi-User — Colisão (race condition de saldo)
   - Prioridade: P0

6. **IT-MU-02**: Deadlock prevention — transferências cruzadas A→B e B→A simultâneas
   - Input: userA envia 50 para userB, userB envia 30 para userA, concorrentes
   - Output esperado: ambas concluem sem deadlock; saldos finais consistentes
   - Heurística: Multi-User — Colisão (deadlock)
   - Prioridade: P0

7. **IT-MU-03**: Lock ordering — múltiplas transferências para mesmo destinatário (hot row)
   - Input: 5 remetentes enviam simultaneamente para mesmo destinatário
   - Output esperado: todas processadas corretamente, saldo final = soma dos créditos
   - Heurística: Multi-User — Inventário Crítico
   - Prioridade: P1

8. **IT-MU-04**: Sessão dupla — mesmo usuário envia de 2 devices simultâneos
   - Input: mesmo senderId, 2 requisições com idempotencyKeys diferentes
   - Output esperado: ambas processadas se saldo permite; idempotência por key individual
   - Heurística: Multi-User — Sessão Dupla
   - Prioridade: P1

#### Módulo: TransactionListRepository
**Heurística: CRUD — Read + Count — 0/1/N**

9. **IT-LST-01**: Query de listagem com índice performa em < 100ms com 100k registros
   - Input: tabela com 100.000 transações, query para userId específico
   - Output esperado: retorno em < 100ms com 10 registros
   - Heurística: Count — Muitos (performance)
   - Prioridade: P1

10. **IT-LST-02**: Controle de acesso — query não retorna transações de outro usuário
    - Input: userId=A consulta; existem transações de A e B
    - Output esperado: apenas transações de A
    - Heurística: CRUD — Read (segurança)
    - Prioridade: P0

#### Módulo: IdempotencyKeyRepository
**Heurística: VADER — Data Consistency + Count — Muitos**

11. **IT-IDP-01**: Persistência e lookup de idempotency-key
    - Input: salvar key "abc-123" com resultado, depois buscar
    - Output esperado: resultado recuperado corretamente
    - Heurística: VADER — Data Consistency
    - Prioridade: P0

12. **IT-IDP-02**: Concorrência — 2 requisições com mesma key simultaneamente
    - Input: mesma key processada por 2 threads
    - Output esperado: apenas 1 cria registro; outra recebe resultado existente
    - Heurística: Multi-User — Colisão
    - Prioridade: P0

#### Módulo: BalanceRepository
**Heurística: Count — 0 + CRUD — Update**

13. **IT-BAL-01**: Saldo nunca fica negativo após transferência
    - Input: saldo=1, amount=1
    - Output esperado: saldo=0, transferência registrada
    - Heurística: Count — Zero (saldo resultante zero)
    - Prioridade: P0

14. **IT-BAL-02**: Validação de saldo ocorre dentro da transação (após lock)
    - Input: saldo=100, 2 envios de 100 concorrentes
    - Output esperado: 1 sucesso, 1 erro; saldo final = 0 (nunca negativo)
    - Heurística: Multi-User — Inventário Crítico
    - Prioridade: P0

15. **IT-BAL-03**: Crédito ao destinatário incrementa corretamente sob concorrência
    - Input: 3 remetentes enviam 10 cada para mesmo destinatário
    - Output esperado: saldo destinatário += 30
    - Heurística: Count — Muitos + Multi-User — Colisão
    - Prioridade: P1

---

### Testes de Serviço / API (9)

#### Endpoint: POST /api/v1/transfers
**Heurística: VADER — 5 dimensões + FAILURE — Appropriate**

1. **API-TRF-01**: Envio bem-sucedido retorna 200 com corpo completo
   - Input: headers válidos, `{ recipientId: "user-B", amount: 100 }`
   - Output esperado: 200, `{ transactionId, amount: 100, recipientId: "user-B", status: "completed", completedAt: ISO8601 }`
   - Heurística: VADER — Values (contrato de sucesso)
   - Prioridade: P0

2. **API-TRF-02**: Amount inválido retorna 400 com código INVALID_AMOUNT
   - Input: `{ recipientId: "user-B", amount: 0 }`
   - Output esperado: 400, `{ code: "INVALID_AMOUNT", message: "Valor deve ser entre 1 e 10000." }`
   - Heurística: VADER — Error Handling
   - Prioridade: P0

3. **API-TRF-03**: Sem token retorna 401 UNAUTHORIZED
   - Input: sem header Authorization
   - Output esperado: 401, `{ code: "UNAUTHORIZED" }`
   - Heurística: VADER — Authorization
   - Prioridade: P0

4. **API-TRF-04**: Conta inativa retorna 403 ACCOUNT_NOT_ACTIVE
   - Input: token de usuário com conta bloqueada
   - Output esperado: 403, `{ code: "ACCOUNT_NOT_ACTIVE" }`
   - Heurística: VADER — Authorization
   - Prioridade: P0

5. **API-TRF-05**: Destinatário inexistente retorna 404 RECIPIENT_NOT_FOUND
   - Input: `{ recipientId: "nonexistent", amount: 100 }`
   - Output esperado: 404, `{ code: "RECIPIENT_NOT_FOUND" }`
   - Heurística: VADER — Values
   - Prioridade: P0

6. **API-TRF-06**: Auto-envio retorna 409 SELF_TRANSFER_NOT_ALLOWED
   - Input: recipientId = id do remetente autenticado
   - Output esperado: 409, `{ code: "SELF_TRANSFER_NOT_ALLOWED" }`
   - Heurística: VADER — Values
   - Prioridade: P0

7. **API-TRF-07**: Saldo insuficiente retorna 422 INSUFFICIENT_BALANCE
   - Input: amount maior que saldo do remetente
   - Output esperado: 422, `{ code: "INSUFFICIENT_BALANCE" }`
   - Heurística: VADER — Values + FAILURE — Appropriate
   - Prioridade: P0

8. **API-TRF-08**: Retry com mesma Idempotency-Key retorna mesmo resultado (200)
   - Input: mesma requisição enviada 2x com mesma key
   - Output esperado: 200 idêntico em ambas, 1 único débito
   - Heurística: VADER — Data Consistency (idempotência end-to-end)
   - Prioridade: P0

9. **API-TRF-09**: Retry com mesma Idempotency-Key retorna mesmo erro (422)
   - Input: key que resultou em 422 na primeira vez
   - Output esperado: 422 idêntico, sem tentativa de reprocessar
   - Heurística: VADER — Data Consistency
   - Prioridade: P1

---

### Testes de Componentes / UI (4)

#### Componente: TransferForm
**Heurística: EMOTIONS + FAILURE — UI**

1. **COMP-TRF-01**: Exibir loading indicator enquanto aguarda resposta da API
   - Input: usuário clica "Enviar", API demora 2s
   - Output esperado: spinner/indicator visível, botão desabilitado
   - Heurística: EMOTIONS — Antecipação (feedback de processamento)
   - Prioridade: P0

2. **COMP-TRF-02**: Exibir mensagem de sucesso e atualizar saldo após 200
   - Input: API retorna 200
   - Output esperado: mensagem de confirmação, saldo atualizado, lista atualizada
   - Heurística: EMOTIONS — Alegria (feedback positivo)
   - Prioridade: P0

3. **COMP-TRF-03**: Preservar campos do formulário após erro e exibir mensagem amigável
   - Input: API retorna 422 (saldo insuficiente)
   - Output esperado: campos mantêm valores, mensagem "Saldo insuficiente..." exibida
   - Heurística: EMOTIONS — Tristeza + FAILURE — UI (preservação de dados)
   - Prioridade: P0

4. **COMP-TRF-04**: Prevenir duplo clique no botão Enviar
   - Input: usuário clica "Enviar" 2x rapidamente
   - Output esperado: apenas 1 requisição enviada; botão desabilitado no primeiro clique
   - Heurística: EMOTIONS — Medo (prevenção de débito duplo)
   - Prioridade: P0

---

### Testes E2E (3)

#### Fluxo: Envio completo de QualiPoints
**Heurística: FAILURE — Recovery + EMOTIONS — Confiança + RCRCRC**

1. **E2E-TRF-01**: Fluxo feliz — login → enviar → confirmação → saldo atualizado → lista atualizada
   - Input: usuário logado com saldo suficiente, destinatário válido
   - Output esperado: transferência concluída, saldo debitado, transação aparece na lista
   - Heurística: RCRCRC — Core (fluxo principal)
   - Prioridade: P0

2. **E2E-TRF-02**: Fluxo de retry — envio com timeout → retry com mesma key → sucesso sem débito duplo
   - Input: simular timeout na primeira tentativa, retry com mesma Idempotency-Key
   - Output esperado: apenas 1 débito, confirmação no retry
   - Heurística: FAILURE — Recovery + EMOTIONS — Confiança
   - Prioridade: P0

3. **E2E-TRF-03**: Consistência cross-channel — envio no App → saldo atualizado no painel web
   - Input: envio via App, depois consulta no painel web (refresh)
   - Output esperado: saldo e lista de transações consistentes entre App e painel
   - Heurística: SFDPOT — Sources (múltiplos canais) + RCRCRC — Core
   - Prioridade: P1

---

## Análise Pós-Deploy (RCRCRC + SFDPOT)

### RCRCRC — Priorização de Regressão

| Dimensão | Aplicação | Prioridade |
|----------|-----------|------------|
| **Recent** | Feature nova, sem histórico em produção — monitorar de perto | P0 |
| **Core** | Transferência é a funcionalidade principal da carteira | P0 |
| **Risky** | Concorrência de saldo sem lock definido; idempotency-key sem TTL | P0 |
| **Configuration-sensitive** | Timeout de 10s, limites de 1–10.000, HTTPS obrigatório | P1 |
| **Conformance** | HTTPS, autenticação Bearer, idempotência — compliance financeiro | P1 |
| **Complex** | Atomicidade multi-tabela (3 operações), concorrência, idempotência | P0 |

**Recomendação**: Após cada deploy, executar obrigatoriamente E2E-TRF-01, E2E-TRF-02, API-TRF-01, API-TRF-08, IT-MU-01, IT-TRF-01.

### SFDPOT — Análise de Risco em Produção

| Dimensão | Risco identificado | Mitigação via teste |
|----------|-------------------|---------------------|
| **Sources** | App e Painel consomem mesma API; divergência possível | E2E-TRF-03 |
| **Formats** | JSON com inteiro; coerção de tipo pode variar entre clients | UT-AMT-11 |
| **Dependencies** | Banco relacional (single point of failure); auth service | IT-TRF-02, IT-TRF-03 |
| **Pace** | Síncrono com timeout 10s; sem rate limit no MVP | FLOOD (cenários futuros) |
| **Environment** | HTTPS obrigatório; certificados, DNS, load balancer | Checklist de infra |
| **Other** | Idempotency-key sem TTL = crescimento infinito de storage | IT-IDP-01 + monitoramento |
| **Time** | Timezone em completedAt; ordenação de lista por timestamp | UT-LST-03 |

---

## Próximos Passos para Implementação

### 1. Implementar Testes Unitários (38 testes)
- **Consulte**: [TEST_UNIT_GUIDE.md](../../docs/guides/TEST_UNIT_GUIDE.md)
- **Passe este relatório** como contexto: `@TEST_UNIT_GUIDE.md @output/test/strategy/test-strategy-qualipoints-envio-20260329.md`
- **Casos**: UT-AMT-01 a UT-AMT-12, UT-RCP-01 a UT-RCP-06, UT-BAL-01 a UT-BAL-04, UT-SND-01 a UT-SND-03, UT-TRF-01 a UT-TRF-04, UT-HDR-01 a UT-HDR-05, UT-LST-01 a UT-LST-04
- **Arquivos sugeridos**:
  - `output/test/unit/validators/validateAmount.test.ts`
  - `output/test/unit/validators/validateRecipient.test.ts`
  - `output/test/unit/validators/validateBalance.test.ts`
  - `output/test/unit/validators/validateSender.test.ts`
  - `output/test/unit/validators/validateHeaders.test.ts`
  - `output/test/unit/services/transferService.test.ts`
  - `output/test/unit/services/listTransactions.test.ts`

### 2. Implementar Testes de Integração (15 testes)
- **Consulte**: [TEST_INTEGRATION_GUIDE.md](../../docs/guides/TEST_INTEGRATION_GUIDE.md)
- **Passe este relatório** como contexto
- **Casos**: IT-TRF-01 a IT-TRF-04, IT-MU-01 a IT-MU-04, IT-LST-01 a IT-LST-02, IT-IDP-01 a IT-IDP-02, IT-BAL-01 a IT-BAL-03
- **Arquivos sugeridos**:
  - `output/test/integration/repositories/transferRepository.test.ts`
  - `output/test/integration/concurrency/multiUser.test.ts`
  - `output/test/integration/repositories/transactionListRepository.test.ts`
  - `output/test/integration/repositories/idempotencyKeyRepository.test.ts`
  - `output/test/integration/repositories/balanceRepository.test.ts`

### 3. Implementar Testes de Serviço/API (9 testes)
- **Consulte**: [TEST_SERVICE_GUIDE.md](../../docs/guides/TEST_SERVICE_GUIDE.md)
- **Passe este relatório** como contexto
- **Casos**: API-TRF-01 a API-TRF-09
- **Arquivos sugeridos**:
  - `output/test/service/api/transfers.test.ts`

### 4. Implementar Testes de Componentes (4 testes)
- **Consulte**: [TEST_COMPONENT_GUIDE.md](../../docs/guides/TEST_COMPONENT_GUIDE.md)
- **Passe este relatório** como contexto
- **Casos**: COMP-TRF-01 a COMP-TRF-04
- **Arquivos sugeridos**:
  - `output/test/component/TransferForm.test.tsx`

### 5. Implementar Testes E2E (3 testes)
- **Consulte**: [TEST_E2E_GUIDE.md](../../docs/guides/TEST_E2E_GUIDE.md)
- **Passe este relatório** como contexto
- **Casos**: E2E-TRF-01 a E2E-TRF-03
- **Arquivos sugeridos**:
  - `output/test/e2e/transfers.spec.ts`

---

## Comandos para Executar Testes

```bash
# Unitários
npm test -- --testPathPattern=output/test/unit

# Integração (requer banco de dados)
npm test -- --testPathPattern=output/test/integration

# Serviço/API (requer servidor rodando)
npm test -- --testPathPattern=output/test/service

# Componentes
npm test -- --testPathPattern=output/test/component

# E2E (requer ambiente completo)
npx playwright test output/test/e2e/

# Todos
npm test
```

---

## Referências

### Skills de Heurísticas

| Fase | Skill | Relatório gerado |
|------|-------|-----------------|
| Desenvolvimento | [`/heuristic-guide-development`](../../skills/heuristic-guide-development/) | [BAICA](../../output/development/BAICA_QUALIPOINTS_TRANSFER.md), [VADER](../../output/development/VADER_QUALITPOINTS_TRANSFER.md) |
| Arquitetura | [`/heuristic-guide-architeture-code`](../../skills/heuristic-guide-architeture-code/) | [CRUD](../../output/architeture-code/CRUD_QUALIPOINTS_ENVIO.md), [Count](../../output/architeture-code/COUNT_REQ_INICIAL_V2.md) |
| Design | [`/heuristic-guide-desing-code`](../../skills/heuristic-guide-desing-code/) | [Multi-User](../../output/design-code/MULTI_USER_QUALIPOINTS_ENVIO.md) |
| QA & Testes | [`/heuristic-guide-qa-test`](../../skills/heuristic-guide-qa-test/) | [FAILURE](../../output/qa-test/FAILURE_QUALIPOINTS_ENVIO.md), [EMOTIONS](../../output/qa-test/EMOTIONS-qualipoints-transferencia-20260329.md), [FLOOD](../../output/qa-test/FLOOD_ANALYSIS_QUALIPOINTS.md) |
| Pós-Deploy | [`/heuristic-guide-pos-deploy`](../../skills/heuristic-guide-pos-deploy/) | Análise incluída neste relatório (seção RCRCRC + SFDPOT) |

### Guides de Implementação

| Camada | Guide |
|--------|-------|
| Unitários | [TEST_UNIT_GUIDE.md](../../docs/guides/TEST_UNIT_GUIDE.md) |
| Integração | [TEST_INTEGRATION_GUIDE.md](../../docs/guides/TEST_INTEGRATION_GUIDE.md) |
| Componentes | [TEST_COMPONENT_GUIDE.md](../../docs/guides/TEST_COMPONENT_GUIDE.md) |
| Serviço/API | [TEST_SERVICE_GUIDE.md](../../docs/guides/TEST_SERVICE_GUIDE.md) |
| E2E | [TEST_E2E_GUIDE.md](../../docs/guides/TEST_E2E_GUIDE.md) |

---

**IMPORTANTE**: Use este relatório como contexto ao chamar os guides!

Exemplo: `@TEST_UNIT_GUIDE.md @output/test/strategy/test-strategy-qualipoints-envio-20260329.md`
"Implemente os testes unitários listados no relatório (UT-AMT-01 a UT-LST-04)"
