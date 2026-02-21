# Análise VADER: Requisito de Envio de QualiPoints

**Versão:** 1.0  
**Data:** 2025-02-21  
**Requisito analisado:** REQ_INICIAL_V2.md  
**Objetivo:** Validar o requisito de envio de QualiPoints entre usuários contra as 5 dimensões VADER

---

## 1. Values (Valores e Validação de Inputs)

### Status: ✅ BEM ESPECIFICADO

#### 1.1 Tipos de dados e formatos
- ✅ **Valor da transferência:** inteiro, sem casas decimais
- ✅ **IDs:** strings (recipientId)
- ✅ **Headers:** authorization (Bearer token), idempotency-key (UUID/string única)
- ✅ **Content-Type:** JSON obrigatório

#### 1.2 Limites (ranges)
- ✅ **Mínimo:** 1 QualiPoint
- ✅ **Máximo:** 10.000 QualiPoints por transação
- ✅ **Saldo:** não há limite máximo especificado para saldo individual (backlog)

**Validações especificadas:**
- ✅ Valores zero ou negativos são rejeitados (400)
- ✅ Valores fora do intervalo [1, 10000] são rejeitados (400)
- ✅ Campo recipientId obrigatório (ausência retorna 400)
- ✅ Campo amount obrigatório (ausência retorna 400)

#### 1.3 Valores ausentes e de limite

| Campo | Obrigatório | Comportamento |
|-------|------------|---|
| authorization | Sim | Ausência → 401 Unauthorized |
| idempotency-key | Sim | Ausência → não especificado (RISCO) |
| recipientId | Sim | Ausência → 400 Bad Request |
| amount | Sim | Ausência → 400 Bad Request |

#### 1.4 Gaps e Riscos

| Gap | Severidade | Observação |
|-----|-----------|-----------|
| **Idempotency-Key ausente** | 🟡 Média | Requisito marca como "Sim" mas não especifica código de erro (400? 422?). Clarificar resposta. |
| **Formato de recipientId** | 🟡 Média | Não há spec de formato (UUID? CPF? email?). A validação depende da estrutura de ID do projeto. |
| **Tamanho do payload** | 🟡 Média | Não há limite máximo de tamanho de requisição (ex.: array de recipients em batch). MVP é one-to-one, mas proteger contra abuse. |
| **Espaços em branco em strings** | 🟢 Baixa | Trim de recipientId? Comportamento esperado? Recomendação: trim e validar, documentar. |

#### 1.5 Recomendações

- [ ] **Definir código HTTP para idempotency-key ausente:** Sugerir 400 Bad Request com código `MISSING_IDEMPOTENCY_KEY`.
- [ ] **Especificar formato de recipientId:** UUID? Email? User ID interno? Documentar formato esperado e exemplo.
- [ ] **Adicionar limite de tamanho de payload:** Ex.: máximo 10KB por requisição.
- [ ] **Definir comportamento de whitespace:** Trim e validar, ou rejeitar?
- [ ] **Validar estrutura de token:** Formato esperado do Authorization header.

---

## 2. Authorization (Autorização e Autenticação)

### Status: ✅ BEM ESPECIFICADO

#### 2.1 Autenticação
- ✅ **Mecanismo:** Bearer token (implícito; padrão REST)
- ✅ **Identificação:** Remetente extraído do token (não enviado no body)
- ✅ **Resposta sem autenticação:** 401 Unauthorized
- ✅ **HTTPS obrigatório:** Explicitamente mencionado (seção 8.3)

#### 2.2 Autorização e permissões

| Cenário | Status | Código HTTP | Validação |
|---------|--------|------------|-----------|
| Conta remetente bloqueada | ✅ Tratado | 403 Forbidden | `ACCOUNT_NOT_ACTIVE` |
| Conta remetente pendente/em análise | ✅ Tratado | 403 Forbidden | Implícito em "ativo" |
| Destinatário bloqueado | ✅ Tratado | 404 Not Found | `RECIPIENT_NOT_FOUND` |
| Envio para si mesmo | ✅ Tratado | 409 Conflict | `SELF_TRANSFER_NOT_ALLOWED` |
| Token expirado | ⚠️ Não especificado | 401? | Recomendação: 401 Unauthorized |
| Token inválido/formato errado | ✅ Implícito | 401 | Mapeado na tabela de erros |

#### 2.3 Propriedade de recursos

- ✅ **Remetente:** Validado via token; API não permite envio em nome de outro usuário.
- ✅ **Lista de transações:** Filtrada por usuário logado; apenas suas transações (como remetente ou destinatário).
- ✅ **Saldo:** Implicitamente protegido (não há endpoint de consulta de saldo de terceiro definido).

#### 2.4 Gaps e Riscos

| Gap | Severidade | Observação |
|-----|-----------|-----------|
| **Token expirado** | 🟡 Média | Não há especificação explícita. Clarificar: 401? Qual mensagem? |
| **Múltiplas contas ativas (mesmo usuário)** | 🟡 Média | Não há contexto; assumir que usuário = 1 conta. |
| **Admin/Suporte (backlog)** | 🟢 Baixa | Fora do escopo MVP, mas documentado como backlog. |
| **API key vs Bearer** | 🟡 Média | Apenas Bearer token mencionado; há suporte a outro tipo? Clarificar. |

#### 2.5 Recomendações

- [ ] **Especificar comportamento de token expirado:** Exemplo: 401 com `code: "TOKEN_EXPIRED"`.
- [ ] **Validar estrutura e validade de token:** Validação deve ocorrer no servidor antes de processar.
- [ ] **Documentar duração de token:** Quanto tempo um token é válido? (fora do escopo deste requisito, mas contextual).
- [ ] **Adicionar rate limit por usuário:** Não especificado, mas recomendado para evitar brute force de idempotency keys.

---

## 3. Data Consistency (Consistência de Dados)

### Status: ✅ MUITO BEM ESPECIFICADO

#### 3.1 Idempotência

- ✅ **Implementação:** Via Idempotency-Key header (UUID único por intenção).
- ✅ **Comportamento em retentativa:**
  - Primeira chamada: processa, grava, retorna 200.
  - Chamadas subsequentes: retorna **mesmo resultado** (200 ou erro original), sem reprocessar.
- ✅ **Proteção contra débito duplo:** Garantida pela idempotência.
- ✅ **Exemplo de caso de uso:** Timeout ou falha de rede → cliente retentar com mesma chave, sem duplicar débito.

#### 3.2 Atomicidade e transações

- ✅ **Operação atômica:** Débito remetente + crédito destinatário + registro de transação = uma transação DB.
- ✅ **Rollback:** Se qualquer passo falha, toda a operação é revertida; saldos não ficam inconsistentes.
- ✅ **Exemplo tratado:** Dois envios simultâneos do mesmo usuário não resultam em saldo inconsistente.

#### 3.3 Integridade referencial

- ✅ **Destinatário deve existir:** Validação obrigatória (404 se não encontrado).
- ✅ **Destinatário ativo:** Exigido; contas inativas não recebem (404).
- ✅ **Remetente ativo:** Exigido; contas bloqueadas não enviam (403).

#### 3.4 Concorrência

- ✅ **Transações simultâneas:** Protegidas por transação DB (lock, isolation level).
- ✅ **Idempotency key:** Evita processamento duplicado no caso de reenvio acidental (mesmo no cliente).

#### 3.5 Validações de negócio

- ✅ **Saldo suficiente:** Validado server-side antes de debitar.
- ✅ **Envio para si mesmo:** Validado e rejeitado (409 Conflict).
- ✅ **Valor no intervalo [1, 10000]:** Validado (400 se inválido).

#### 3.6 Gaps e Riscos

| Gap | Severidade | Observação |
|-----|-----------|-----------|
| **Isolation level DB** | 🟡 Média | Não é especificado (ex.: READ_COMMITTED, SERIALIZABLE). Recomendação: usar SERIALIZABLE ou Optimistic Locking. |
| **Timeout da transação DB** | 🟡 Média | Não há spec; API tem timeout de 10s, mas transação DB? |
| **Limite de retentativas com Idempotency-Key** | 🟡 Média | Quantas vezes pode retentar? Há limpeza de chaves antigas? |
| **Validação no cliente** | 🟢 Baixa | Permitida para UX, mas requisito clara que fonte da verdade é servidor. OK. |
| **Timestamp de transação** | 🟢 Baixa | Não especificado (quem define? servidor ou cliente?). Recomendação: servidor (clock único). |

#### 3.7 Recomendações

- [ ] **Especificar isolation level:** Sugerir SERIALIZABLE ou equivalente para evitar race conditions.
- [ ] **Definir timeout da transação DB:** Ex.: 5 segundos (menor que API timeout de 10s).
- [ ] **Limpeza de Idempotency-Keys:** Política de limpeza (ex.: chaves com >30 dias sem uso). Previne consumo ilimitado de armazenamento.
- [ ] **Timestamp:** Sempre gerado pelo servidor (UTC), não pelo cliente.
- [ ] **Teste de concorrência:** Incluir casos de teste com 2+ transações simultâneas do mesmo usuário.

---

## 4. Error Handling (Tratamento de Erros)

### Status: ✅ EXCELENTE

#### 4.1 Códigos HTTP e casos mapeados

| Código | Caso | Código de erro (code field) | Descrição |
|--------|------|----------|-----------|
| **200 OK** | Sucesso | N/A | `{ transactionId, amount, recipientId, status: "completed", completedAt }` |
| **400 Bad Request** | Valor inválido | `INVALID_AMOUNT` | Valor fora do intervalo ou tipo errado. |
| **401 Unauthorized** | Sem autenticação / token inválido | `UNAUTHORIZED` | Sem token ou token inválido. |
| **403 Forbidden** | Conta bloqueada | `ACCOUNT_NOT_ACTIVE` | Conta remetente não ativa. |
| **404 Not Found** | Destinatário não encontrado | `RECIPIENT_NOT_FOUND` | Destinatário não existe ou inativo. |
| **409 Conflict** | Envio para si mesmo | `SELF_TRANSFER_NOT_ALLOWED` | Mesmo ID de remetente e destinatário. |
| **422 Unprocessable Entity** | Saldo insuficiente | `INSUFFICIENT_BALANCE` | Saldo < amount. Opcional: retornar saldo atual. |
| **408/504** | Timeout | `REQUEST_TIMEOUT` | Mensagem: "Pode tentar novamente com mesma chave." |
| **5xx** | Erro interno | `INTERNAL_ERROR` | Mensagem genérica; sem expor stack trace. |

#### 4.2 Estrutura e mensagens

- ✅ **Formato consistente:** `{ code, message }` em todas as respostas de erro.
- ✅ **Mensagens claras e acionáveis:** Ex.: "Valor deve ser entre 1 e 10.000 QualiPoints."
- ✅ **Sem exposição de detalhes internos:** Stack traces não são retornados.
- ✅ **Mapeamento de mensagens para UI:** Tabela de códigos → mensagens de usuário.

#### 4.3 Validação de inputs

- ✅ **Erros de validação específicos:** Ex.: `INVALID_AMOUNT`, `SELF_TRANSFER_NOT_ALLOWED`.
- ✅ **Resposta 400 vs 422:** 400 para formato/tipo errado; 422 para lógica de negócio inválida.

#### 4.4 Logging e auditoria

- ✅ **Auditoria mencionada (seção 8.3):** Toda tentativa (sucesso ou falha) é logada com: idempotency-key, usuário, valor, destinatário, resultado.
- ✅ **Segurança:** Logs protegidos; sem expor tokens completos.

#### 4.5 Gaps e Riscos

| Gap | Severidade | Observação |
|-----|-----------|-----------|
| **Estrutura de erro com múltiplos campos inválidos** | 🟡 Média | Requisito não especifica: retorna 400 com um `code` ou lista? Ex.: `{ errors: [{field: "amount", code: "INVALID", message: "..."}] }`. |
| **Retry-After header** | 🟡 Média | Mencionado para 429/503, mas 429 não está na tabela. Clarificar: há rate limiting? Se sim, incluir na tabela. |
| **Diferença entre 401 e 403** | 🟢 Baixa | Bem mapeado (401 = sem autenticação, 403 = conta bloqueada). |
| **Corpo de erro em sucesso (200)** | 🟢 Baixa | Bem definido; contém `transactionId`, `status: "completed"`. |
| **Suporte a idiomas (i18n)** | 🟠 Não especificado | Mensagens em português? English? API retorna código (`code`) e cliente mapeia? Recomendação: retornar `code` e cliente faz i18n. |

#### 4.6 Recomendações

- [ ] **Definir estrutura para múltiplos erros:** Se houver vários campos inválidos, retornar array ou um único erro prioritário?
- [ ] **Adicionar Retry-After:** Para 429 (se houver rate limiting) e 503, incluir header.
- [ ] **Implementar idempotência de erro:** Requisição retentada com mesma Idempotency-Key retorna o mesmo erro da primeira tentativa (se falhou), ou apenas sucesso?
  - Recomendação: retornar o mesmo (código + corpo + status) para garantir observabilidade.
- [ ] **Logging detalhado:** Internamente, logar: timestamp, idempotency-key, user-id, recipient-id, amount, source-balance, request-body (sem token), response-code, execution-time.
- [ ] **Tratamento de erro de banco:** Incluir timeouts, deadlocks, constraint violations. Mapear para 5xx com código genérico.

---

## 5. Rate Limiting & Resource Consumption

### Status: ⚠️ PARCIALMENTE ESPECIFICADO (Escopo MVP)

#### 5.1 Rate limiting

- ❌ **Rate limiting não implementado no MVP:** Seção 4.3 menciona "no MVP não há limite além do máximo por transação."
- ⚠️ **429 Conflict não está na tabela de respostas** (seção 10.4), mas é mencionado em "Retry-After para erros temporários."
- ⚠️ **Limite por período (dia/mês):** Documentado como backlog (seção 4.3).

#### 5.2 Timeouts

- ✅ **API timeout:** 10 segundos (seção 5.2).
- ✅ **Resposta em timeout:** 408 Request Timeout ou 504 Gateway Timeout (conforme convenção).
- ✅ **Mensagem ao cliente:** "Pode tentar novamente com a mesma chave."

#### 5.3 Limites de recursos

- ⚠️ **Limite de payload:** Não especificado. Requisição mínima é ~100 bytes; máxima não definida.
- ⚠️ **Limite de resultados em listagem:** Últimas 10 transações (seção 6.3), OK. Sem paginação no MVP.
- ✅ **Sem upload/download de arquivo:** Fora do escopo (apenas JSON).

#### 5.4 Proteção contra DoS

- ⚠️ **Brute force de idempotency keys:** Não há proteção contra geração aleatória de chaves. Recomendação: rate limit por usuário (requisições/minuto).
- ⚠️ **Listagem de transações:** Sempre retorna últimas 10; sem risco de DoS.
- ⚠️ **Query maliciosa:** Não aplicável (operação simples).

#### 5.5 Gaps e Riscos

| Gap | Severidade | Observação |
|-----|-----------|-----------|
| **Rate limiting por usuário** | 🟡 Média | MVP não tem; backlog. Recomendação: adicionar no futuro (ex.: 10 requisições/minuto por usuário). |
| **Proteção contra listagem frequente** | 🟡 Média | Listagem de transações tem últimas 10, sem paginação. OK para MVP, mas documentar limite de requisições à API de consulta. |
| **Limite de tamanho de payload** | 🟡 Média | Não especificado. Sugerir: máximo 10KB por requisição. |
| **Timeout de resposta do cliente** | 🟡 Média | Requisito menciona "10 segundos" para API, mas App? Recomendação: App espera 15 segundos antes de decidir erro. |
| **Retry exponencial** | 🟠 Não especificado | Cliente deve retentar ou desistir? Recomendação: até 3 tentativas com backoff exponencial (1s, 2s, 4s). |
| **Concorrência de requisições (pipelining)** | 🟡 Média | Cliente pode enviar múltiplas requisições sem aguardar resposta? Comportamento esperado não é claro. |

#### 5.6 Recomendações

- [ ] **Implementar rate limiting para MVP (ou clarificar como está in backlog):**
  - Sugerir: 10 transações/minuto por usuário.
  - Retornar 429 com `X-RateLimit-Remaining`, `X-RateLimit-Reset`.
  - Incluir 429 na tabela de respostas (seção 10.4).

- [ ] **Definir limite de tamanho de payload:** Ex.: 10KB. Retornar 413 Payload Too Large se excedido.

- [ ] **Especificar estratégia de retry no cliente:**
  - Máximo 3 tentativas.
  - Backoff exponencial: 1s, 2s, 4s.
  - Usar mesma Idempotency-Key.

- [ ] **Timeout de cliente (App):** Sugerir 15 segundos (5s buffer acima do timeout da API).

- [ ] **Documentar comportamento de pipelining:** Permitir ou não múltiplas requisições simultâneas?

---

## 6. Checklist de Conformidade VADER

| Dimensão | Status | Gaps críticos | Recomendações |
|-----------|--------|---|---|
| **Values** | ✅ Bom | Idempotency-Key ausente (código HTTP não definido); formato de recipientId não especificado. | Definir código de erro; especificar formato. |
| **Authorization** | ✅ Excelente | Token expirado não mencionado explicitamente. | Clarificar comportamento de token expirado. |
| **Data Consistency** | ✅ Excelente | Isolation level DB e limpeza de chaves de idempotência não especificados. | Especificar isolation level; política de limpeza. |
| **Error Handling** | ✅ Excelente | Estrutura de múltiplos erros de validação não definida; Retry-After não está na tabela. | Clarificar estrutura multi-erro; adicionar 429. |
| **Rate Limiting** | ⚠️ Parcial | Rate limiting não implementado (MVP); proteção contra brute force de chaves. | Especificar backlog; considerar adicionar rate limit por usuário. |

---

## 7. Riscos Identificados (Resumido)

### Críticos (🔴 Deve resolver antes de dev)
- Nenhum identificado. Requisito é maduro e bem especificado.

### Altos (🟠 Resolver antes de deploy)
1. **Idempotency-Key ausente:** Código HTTP não definido. Deve ser 400.
2. **Rate limiting MVP:** Documentar claramente se é MVP sem rate limiting ou se há uma estratégia.
3. **Isolation level DB:** Especificar SERIALIZABLE ou equivalente para evitar race conditions.

### Médios (🟡 Resolver antes de produção)
1. Formato de recipientId não especificado.
2. Token expirado sem código de erro definido.
3. Estrutura de erro com múltiplos campos inválidos.
4. Limpeza de Idempotency-Keys (política de retenção).
5. Limite de tamanho de payload não definido.
6. Comportamento de pipelining não definido.

### Baixos (🟢 Nice to have / backlog)
1. Mensagens em idiomas diferentes (i18n).
2. Timestamps de transação gerados pelo servidor (já é implícito).
3. Testes de concorrência (será feito em teste).

---

## 8. Recomendações Consolidadas

### Antes do desenvolvimento (curto prazo)
1. Criar documento **API Specification (OpenAPI/Swagger)** que resolva todos os gaps acima (especialmente Idempotency-Key ausente, rate limiting, isolation level).
2. Definir **matriz de testes VADER** (test strategy) baseada neste requisito:
   - **Values:** testes de validação (válido, inválido, limite).
   - **Authorization:** testes de autenticação e permissão.
   - **Data Consistency:** testes de idempotência, atomicidade, concorrência.
   - **Error Handling:** testes de códigos HTTP e mensagens.
   - **Rate Limiting:** testes de timeouts (quando implementado).

### Antes da produção (médio prazo)
1. Implementar auditoria completa (logging de todas as tentativas).
2. Implementar limpeza de Idempotency-Keys (ex.: chaves com >30 dias).
3. Testes de carga para validar timeout de 10s e comportamento sob pico de tráfego.
4. Documentação de monitoramento e alertas (ex.: alerta se % de 5xx > 1%).

### Backlog (futuro)
1. Rate limiting por usuário (ex.: 10 transações/minuto).
2. Limite por período (dia/mês) por usuário.
3. Notificações push/e-mail em caso de falha.
4. Aprovação manual no painel web (documentado como backlog no requisito).
5. Comprovante em PDF e e-mail automático.

---

## 9. Conclusão

O requisito **REQ_INICIAL_V2.md é de alta qualidade** e está bem especificado nas dimensões **Values, Authorization, Data Consistency e Error Handling**. A dimensão **Rate Limiting** é o ponto mais fraco (não há rate limiting no MVP), mas está documentado como backlog.

**Recomendação:** Proceder para fase de design e implementação, resolvendo os **gaps de especificação** (Idempotency-Key código HTTP, isolation level, limite de payload) através de documento OpenAPI ou similar antes de começar a codificação.

A aplicação rigorosa da heurística VADER durante testes reduzirá significativamente riscos de **segurança, consistência de dados e escalabilidade** em produção.

---

## Apêndice A: Checklist para Desenvolvimento

Use este checklist durante implementação para garantir conformidade VADER:

### Values
- [ ] Validar tipo e formato de recipientId
- [ ] Validar tipo inteiro de amount
- [ ] Rejeitar amount < 1 ou > 10000 (400)
- [ ] Rejeitar amount zero ou negativo (400)
- [ ] Rejeitar requisição sem recipientId (400)
- [ ] Rejeitar requisição sem amount (400)
- [ ] Rejeitar requisição sem Authorization header (401)
- [ ] Definir código de erro para Idempotency-Key ausente (400)
- [ ] Trim e validar recipientId (ou especificar comportamento)

### Authorization
- [ ] Extrair e validar token do Authorization header
- [ ] Rejeitar token inválido (401)
- [ ] Rejeitar token expirado (401 com código TOKEN_EXPIRED)
- [ ] Validar que conta remetente está ativa (403 se não)
- [ ] Validar que conta destinatário existe e está ativa (404 se não)
- [ ] Rejeitar envio para si mesmo (409)
- [ ] Usar HTTPS em todas as requisições

### Data Consistency
- [ ] Validar saldo remetente >= amount
- [ ] Usar transação DB para débito + crédito + registro (SERIALIZABLE ou equivalente)
- [ ] Implementar Idempotency-Key: armazenar resposta da primeira requisição
- [ ] Retornar mesma resposta em retentativa (sem reprocessar)
- [ ] Rollback em falha (qualquer passo falha, toda transação falha)
- [ ] Usar timestamp gerado pelo servidor (UTC)
- [ ] Validar integridade referencial (destinatário existe)

### Error Handling
- [ ] Retornar 400 para INVALID_AMOUNT
- [ ] Retornar 401 para UNAUTHORIZED (sem token ou inválido)
- [ ] Retornar 403 para ACCOUNT_NOT_ACTIVE (remetente bloqueado)
- [ ] Retornar 404 para RECIPIENT_NOT_FOUND
- [ ] Retornar 409 para SELF_TRANSFER_NOT_ALLOWED
- [ ] Retornar 422 para INSUFFICIENT_BALANCE (com saldo atual opcional)
- [ ] Retornar 408 ou 504 para REQUEST_TIMEOUT
- [ ] Retornar 5xx genérico para INTERNAL_ERROR (sem expor stack)
- [ ] Estrutura consistente: `{ code, message }`
- [ ] Logar todas as tentativas (sucesso e falha) com idempotency-key, user, amount, resultado
- [ ] Não expor detalhes internos em respostas de erro

### Rate Limiting & Resource Consumption
- [ ] Implementar timeout de API (10 segundos)
- [ ] Documentar (ou implementar) rate limiting por usuário
- [ ] Documentar (ou implementar) limite de tamanho de payload (ex.: 10KB)
- [ ] Especificar estratégia de retry no cliente (3 tentativas, backoff exponencial)
- [ ] Implementar limpeza de Idempotency-Keys (política de retenção)

---

## Apêndice B: Matriz de Testes Baseada em VADER

A seguir, exemplos de testes a serem implementados em cada camada (unit, integration, service, E2E):

### Values — Testes de Validação

**Unit / Integration:**
- Validar INVALID_AMOUNT (valor 0, negativo, > 10000)
- Validar tipo inteiro (rejeitar string, float)
- Validar MISSING_AMOUNT (body sem amount)
- Validar MISSING_RECIPIENT_ID (body sem recipientId)
- Validar formato de recipientId (UUID? Email?)

**Service (API):**
- POST /api/v1/transfers com amount=0 → 400 INVALID_AMOUNT
- POST /api/v1/transfers com amount=10001 → 400 INVALID_AMOUNT
- POST /api/v1/transfers com amount="abc" → 400 INVALID_AMOUNT
- POST /api/v1/transfers sem body → 400 BAD_REQUEST
- POST /api/v1/transfers sem Authorization → 401 UNAUTHORIZED

### Authorization — Testes de Autenticação e Autorização

**Unit / Integration:**
- Validar token (formato, expiração, assinatura)
- Validar status de conta (ativa, bloqueada, pendente)

**Service (API):**
- POST sem Authorization header → 401 UNAUTHORIZED
- POST com token inválido → 401 UNAUTHORIZED
- POST com token expirado → 401 com TOKEN_EXPIRED
- POST com conta remetente bloqueada → 403 ACCOUNT_NOT_ACTIVE
- POST com destinatário inativo → 404 RECIPIENT_NOT_FOUND
- POST para si mesmo → 409 SELF_TRANSFER_NOT_ALLOWED

### Data Consistency — Testes de Idempotência, Atomicidade, Concorrência

**Integration / Service:**
- POST com Idempotency-Key K1 e amount=100 → 200, retorna transactionId T1
- POST com mesma K1 e amount=100 → 200, retorna transactionId T1 (mesma transação, não duplicado)
- Verificar que saldo foi debitado apenas uma vez
- POST com amount=50 e saldo=40 → 422 INSUFFICIENT_BALANCE
- Verificar que saldo remetente não foi debitado (rollback)
- Dois POSTs simultâneos com K1 e K2 → ambos retornam sucesso, saldos consistentes

**E2E:**
- Usuário envia 100 para outro, confirma na App
- Painel web mostra transação na lista (após refresh)
- Saldos em ambos os lados estão corretos

### Error Handling — Testes de Códigos HTTP e Mensagens

**Service (API):**
- Verificar código HTTP para cada erro (400, 401, 403, 404, 409, 422, 5xx)
- Verificar estrutura `{ code, message }` em todas as respostas
- Verificar mensagem é clara e acionável (não expõe stack)
- Verificar sem informações sensíveis (tokens, caminhos internos)

### Rate Limiting & Resource Consumption — Testes de Timeout e Limites

**Service (API) / Load:**
- POST com timeout >= 10s → 408/504 REQUEST_TIMEOUT
- Múltiplos POSTs rápidos (quando rate limiting for implementado) → 429 Too Many Requests
- Verificar headers X-RateLimit-* presentes (quando implementado)
- Teste de carga: 100 requisições/segundo → API responde sem degradação severa

---

## Versão e Histórico

| Versão | Data | Autor | Mudanças |
|--------|------|-------|----------|
| 1.0 | 2025-02-21 | Análise VADER | Primeira análise completa do REQ_INICIAL_V2 com 5 dimensões VADER |
