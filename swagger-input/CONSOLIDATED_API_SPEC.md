# Especificação Consolidada da API: Envio de QualiPoints (v1.0)

## 1. Informações Gerais da API

- **Título**: API de Transferência de QualiPoints
- **Descrição**: API REST que permite usuários autenticados transferirem pontos (QualiPoints) de sua carteira para outros usuários ativos. Garante integridade do saldo, rastreabilidade e operação atômica. Operação síncrona com confirmação em tempo real.
- **Versão**: 1.0.0
- **Servidor**: `https://api.exemplo.com` (placeholder; ajustar conforme ambiente)
- **Base path**: `/api/v1`

## 2. Autenticação e Segurança

- **Tipo**: Bearer JWT (HTTP Bearer)
- **Header**: `Authorization: Bearer <token>`
- **Regra**: O `user_id` do remetente deve ser extraído do token; nunca enviado no corpo da requisição.
- **Escopo**: Todos os endpoints requerem autenticação (exceto documentação explícita em contrário).

## 3. Endpoints

### 3.1 POST /api/v1/transactions — Transferir QualiPoints

Transferir QualiPoints para outro usuário. Operação síncrona e atômica.

**Headers:**
- `Authorization` (required): Bearer JWT
- `idempotency-key` (required): string (UUID). Chave para garantir idempotência e evitar transferências duplicadas em cliques repetidos ou retentativas de rede. Em caso de requisição duplicada com mesma chave, retornar 201 com os mesmos dados da transação original (sem criar nova). Opcional: incluir header `X-Idempotent-Replay: true` na resposta quando for replay.

**Request Body (application/json):**
- `recipient_id` (string, required): ID do usuário destinatário
- `amount` (number, required): Quantidade de QualiPoints. Mínimo: 1. Máximo: saldo disponível do remetente.

**Resposta 201 Created:**
- `transaction_id` (string, format: uuid): Identificador único da transação
- `new_balance` (number): Novo saldo do remetente após a transferência

**Códigos de Erro:**
- **402 Payment Required**: Saldo insuficiente. Mensagem sugerida: "Saldo insuficiente. Você possui {balance} QualiPoints disponíveis." Incluir no corpo: `available_balance`, `requested_amount`, `shortfall` quando útil.
- **404 Not Found**: Destinatário inexistente. Mensagem: "Destinatário não encontrado. Verifique se o ID está correto." Incluir `recipient_id` no corpo.
- **422 Unprocessable Entity**: Valor abaixo do mínimo (1), formato inválido ou destinatário não está "Ativo". Mensagens: "Valor mínimo de transferência é 1 QualiPoint." ou "Destinatário não está ativo. Status atual: {status}."
- **401 Unauthorized**: Token inválido ou expirado
- **503 Service Unavailable**: Timeout de lock ou sistema sobrecarregado. Incluir header `Retry-After` em segundos quando aplicável.

**Regras de Negócio:**
- Valor mínimo: 1 QualiPoint
- Valor máximo: saldo disponível do remetente
- Destinatário deve existir e estar com status "Ativo"
- Operação atômica: transação de banco de dados; rollback automático se crédito no destinatário falhar
- Transação síncrona: confirmação em tempo real

---

### 3.2 GET /api/v1/wallets/me — Consultar Saldo Próprio

Consultar saldo da carteira do usuário autenticado.

**Headers:** `Authorization` (required): Bearer JWT

**Resposta 200 OK:**
- `user_id` (string): ID do usuário
- `balance` (number): Saldo atual em QualiPoints
- `updated_at` (string, format: date-time): Data/hora da última atualização

**Códigos de Erro:**
- **401 Unauthorized**: Token inválido ou expirado

**Nota:** Recomendação das análises: cache com TTL curto (30–60s) para leituras frequentes.

---

### 3.3 GET /api/v1/transactions — Histórico de Transações

Listar transações do usuário autenticado (como remetente ou destinatário). Paginação obrigatória.

**Headers:** `Authorization` (required): Bearer JWT

**Query Parameters:**
- `limit` (integer, optional): Número de itens por página. Padrão: 20. Máximo: 50.
- `offset` (integer, optional): Deslocamento para paginação. Padrão: 0.
- `start_date` (string, format: date, optional): Filtro data inicial (YYYY-MM-DD)
- `end_date` (string, format: date, optional): Filtro data final (YYYY-MM-DD)
- `sort` (string, optional): Campo de ordenação. Valores: `created_at`. Padrão: `created_at`
- `order` (string, optional): Ordem. Valores: `asc`, `desc`. Padrão: `desc`

**Resposta 200 OK:**
- Array de objetos **Transaction** (ver Schemas)
- Metadados de paginação: `total`, `limit`, `offset`, `has_more`

**Comportamento:** Quando não há transações, retornar array vazio `[]` (não `null`).

**Códigos de Erro:**
- **401 Unauthorized**: Token inválido ou expirado
- **429 Too Many Requests**: Rate limit excedido (quando implementado)

---

## 4. Schemas (Modelos)

### TransferRequest
- `recipient_id` (string, required): ID do usuário destinatário
- `amount` (number, required, minimum: 1): Quantidade de QualiPoints

### TransferResponse
- `transaction_id` (string, format: uuid): ID único da transação
- `new_balance` (number): Novo saldo do remetente

### Transaction
- `transaction_id` (string, format: uuid)
- `sender_id` (string): ID do remetente
- `recipient_id` (string): ID do destinatário
- `amount` (number): Valor transferido
- `status` (string, enum): `pending`, `processing`, `completed`, `failed`
- `created_at` (string, format: date-time): Data/hora da criação

### Wallet
- `user_id` (string): ID do usuário
- `balance` (number): Saldo em QualiPoints
- `updated_at` (string, format: date-time): Última atualização

### ErrorResponse (respostas de erro)
- `error` (object):
  - `code` (string): Código do erro (ex: INSUFFICIENT_BALANCE, RECIPIENT_NOT_FOUND)
  - `message` (string): Mensagem legível
  - `details` (object, optional): Dados adicionais (ex: available_balance, requested_amount)
  - `suggestion` (string, optional): Sugestão de ação
  - `timestamp` (string, format: date-time, optional)
  - `transaction_id` (string, optional): null quando não há transação

### TransactionsListResponse
- `data` (array of Transaction): Lista de transações
- `total` (integer): Total de registros
- `limit` (integer): Limite por página
- `offset` (integer): Offset atual
- `has_more` (boolean): Indica se há mais páginas

---

## 5. Códigos de Status Resumidos

| Código | Descrição |
|--------|-----------|
| 200 | OK — Consulta (saldo, histórico) |
| 201 | Created — Transferência realizada |
| 400 | Bad Request — Requisição malformada |
| 401 | Unauthorized — Token inválido ou ausente |
| 402 | Payment Required — Saldo insuficiente |
| 404 | Not Found — Destinatário inexistente |
| 422 | Unprocessable Entity — Validação (valor mínimo, formato, status do destinatário) |
| 429 | Too Many Requests — Rate limit excedido |
| 503 | Service Unavailable — Timeout ou sistema ocupado |

---

## 6. Exemplos

### Exemplo Request — POST /api/v1/transactions
```json
{
  "recipient_id": "usr_abc123",
  "amount": 50
}
```

### Exemplo Response 201 — POST /api/v1/transactions
```json
{
  "transaction_id": "550e8400-e29b-41d4-a716-446655440000",
  "new_balance": 450
}
```

### Exemplo Response 402
```json
{
  "error": {
    "code": "INSUFFICIENT_BALANCE",
    "message": "Saldo insuficiente para esta transferência",
    "details": {
      "available_balance": 45,
      "requested_amount": 50,
      "shortfall": 5
    },
    "suggestion": "Você precisa de mais 5 QualiPoints para completar esta transferência.",
    "timestamp": "2026-02-16T10:30:00Z",
    "transaction_id": null
  }
}
```

### Exemplo Response 200 — GET /api/v1/wallets/me
```json
{
  "user_id": "usr_abc123",
  "balance": 500,
  "updated_at": "2026-02-16T10:00:00Z"
}
```

### Exemplo Response 200 — GET /api/v1/transactions
```json
{
  "data": [
    {
      "transaction_id": "550e8400-e29b-41d4-a716-446655440000",
      "sender_id": "usr_abc123",
      "recipient_id": "usr_xyz789",
      "amount": 50,
      "status": "completed",
      "created_at": "2026-02-16T10:30:00Z"
    }
  ],
  "total": 1,
  "limit": 20,
  "offset": 0,
  "has_more": false
}
```

---

## 7. Validações e Regras Documentadas

- **amount**: number, mínimo 1; máximo = saldo do remetente (validado no servidor)
- **recipient_id**: string; deve existir e ter status "Ativo"
- **idempotency-key**: UUID; recomendado para todas as requisições POST de transferência
- Estados de transação: Pendente → Processando → Concluída ou Falha
- Usuário destinatário: apenas status "Ativo" pode receber transferências

---

## 8. Referências das Análises

Consolidado a partir de:
- setUp/REQ_FINAL.MD (contrato mínimo, regras, erros)
- example/Cap04_concepcao/USER_SCENARIOS_REQ_INICIAL.md (personas, mensagens de erro)
- example/Cap05_design/MULTI_USER_REQ_FINAL.md (idempotência, conflitos, mensagens)
- example/Cap05_design/STATE_ANALYSIS_REQ_FINAL.md (estados da transação)
- example/Cap06_arquitetura/COUNT_REQ_FINAL.md (paginação, endpoints de consulta)
- example/Cap06_arquitetura/CRUD_REQ_FINAL.md (schemas, endpoints GET wallets/me e transactions)
