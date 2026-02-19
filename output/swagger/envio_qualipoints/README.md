# Swagger — Envio de QualiPoints entre usuários

**Gerado a partir de:** `output/requisito-revisado/REQ_INICIAL_V2.md`
**Versão da especificação:** 2.0.0
**Data:** 2025-02-18
**OpenAPI:** 3.0.3

---

## Resumo da Análise

| Item | Detalhe |
|------|---------|
| **Endpoints** | 2 |
| **Recursos/paths** | `/api/v1/transfers`, `/api/v1/transfers/recent` |
| **Schemas em components** | 6 (`TransferRequest`, `TransferResponse`, `TransactionItem`, `RecentTransactionsResponse`, `ErrorResponse`, `InsufficientBalanceError`) |
| **Autenticação** | Bearer JWT (global) |
| **Header de idempotência** | `Idempotency-Key` (UUID, obrigatório no POST) |
| **Códigos de resposta mapeados** | 200, 400, 401, 403, 404, 408, 409, 422, 500, 504 |

---

## Endpoints

### `POST /api/v1/transfers`
Envia QualiPoints do usuário autenticado para outro usuário.

- **Auth:** Bearer JWT (remetente identificado pelo token)
- **Header obrigatório:** `Idempotency-Key` (UUID único por intenção de envio)
- **Body:** `recipientId` (string) + `amount` (integer, 1–10.000)
- **Sucesso:** 200 com dados da transação (`transactionId`, `amount`, `recipientId`, `status`, `completedAt`)
- **Timeout:** API responde em até 10 s; caso contrário, 504

| Código | Situação |
|--------|----------|
| 200 | Transferência concluída |
| 400 | Valor inválido (`INVALID_AMOUNT`) |
| 401 | Token ausente ou inválido (`UNAUTHORIZED`) |
| 403 | Conta do remetente bloqueada (`ACCOUNT_NOT_ACTIVE`) |
| 404 | Destinatário não encontrado/inativo (`RECIPIENT_NOT_FOUND`) |
| 408 | Timeout do cliente (`REQUEST_TIMEOUT`) |
| 409 | Envio para a própria conta (`SELF_TRANSFER_NOT_ALLOWED`) |
| 422 | Saldo insuficiente (`INSUFFICIENT_BALANCE`) |
| 500 | Erro interno (`INTERNAL_ERROR`) |
| 504 | Gateway timeout — API excedeu 10 s (`REQUEST_TIMEOUT`) |

### `GET /api/v1/transfers/recent`
Lista as últimas 10 transações do usuário autenticado (enviadas e recebidas), ordenadas da mais recente para a mais antiga.

- **Auth:** Bearer JWT
- **Sem filtros no MVP**
- **Controle de acesso:** cada usuário vê apenas suas próprias transações

| Código | Situação |
|--------|----------|
| 200 | Lista retornada com sucesso |
| 401 | Token ausente ou inválido |
| 500 | Erro interno |

---

## Autenticação

- **Tipo:** Bearer JWT
- **Header:** `Authorization: Bearer <token>`
- Aplicada **globalmente** (todas as operações exigem autenticação).
- O remetente é identificado pelo token; não é enviado no body da requisição.

---

## Como Visualizar

1. Acesse [Swagger Editor](https://editor.swagger.io/)
2. Clique em **File → Import file** e selecione `openapi.yaml`
3. Explore os endpoints, schemas e exemplos de request/response

Alternativa via CLI:
```bash
# Com swagger-cli instalado
npx @apidevtools/swagger-cli validate openapi.yaml

# Servir localmente com Swagger UI (Docker)
docker run -p 8080:8080 -e SWAGGER_JSON=/api/openapi.yaml \
  -v $(pwd):/api swaggerapi/swagger-ui
```

---

## Gaps e Ambiguidades Identificados

| # | Gap | Impacto | Sugestão |
|---|-----|---------|----------|
| 1 | **Formato do `recipientId`** não especificado no requisito (UUID, e-mail, apelido?) | Médio | Definir tipo e formato antes do desenvolvimento |
| 2 | **Formato do `transactionId`** na resposta não especificado | Baixo | Adotar UUID v4 como padrão |
| 3 | **Path do endpoint de listagem** não definido explicitamente no requisito | Baixo | Confirmado como `/api/v1/transfers/recent`; validar com o time |
| 4 | **Código HTTP de sucesso no POST** — o requisito usa 200 OK, mas REST convencional usaria 201 Created | Baixo | O requisito define explicitamente 200; manter para aderir ao contrato definido |
| 5 | **Campos do `TransactionItem`** para a listagem não detalhados no requisito | Médio | Campos inferidos (`senderId`, `recipientId`, `direction`, `status`, `completedAt`); confirmar com o time |
| 6 | **Limite diário/mensal e rate limit** explicitamente deixados para backlog | Baixo | Adicionar endpoints e schemas quando o backlog for refinado |
| 7 | **Saldo atual do remetente** em `InsufficientBalanceError` é marcado como opcional no requisito (`currentBalance`) | Baixo | Definir se será sempre retornado ou apenas quando disponível |

---

## Sugestões de Melhoria (REST)

- **Versionamento no servidor:** considerar `https://api.qualipoints.com/v1` como base URL em vez de repetir `/v1` nos paths.
- **201 Created no POST:** convenção REST para criação de recursos; o requisito adota 200 OK — manter consistência com o contrato definido.
- **Paginação futura:** ao evoluir `GET /api/v1/transfers/recent`, adicionar `page`, `limit` e `total` para suportar mais de 10 transações.
- **Campo `direction`:** facilita o consumo pela UI (saber se foi enviado ou recebido sem comparar IDs).

---

## Baseado em

- Requisito: `output/requisito-revisado/REQ_INICIAL_V2.md`
- Skill: `Skill/utils/builSwagger/SWAGGER_API_DOC.md` (Modo 1 — Análise de Requisitos)
