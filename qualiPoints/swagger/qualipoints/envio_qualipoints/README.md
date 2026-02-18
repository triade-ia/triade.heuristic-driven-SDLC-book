# Documentação OpenAPI — Envio de QualiPoints

## Nome da Funcionalidade

**Envio de QualiPoints** — Transferência de pontos (QualiPoints) entre usuários autenticados, com operação atômica e idempotência.

## Resumo da Análise

- **Endpoints**: 3
  - `POST /api/v1/transactions` — Transferir QualiPoints
  - `GET /api/v1/transactions` — Histórico de transações (paginação)
  - `GET /api/v1/wallets/me` — Consultar saldo próprio
- **Recursos/paths**: Transações, Carteiras
- **Schemas em components**: 6 (TransferRequest, TransferResponse, Transaction, Wallet, ErrorResponse, TransactionsListResponse)
- **Autenticação**: Bearer JWT (global em todas as operações)

## Tipo de Autenticação

- **Bearer JWT**: Header `Authorization: Bearer <token>`
- O `user_id` do remetente é extraído do token; não deve ser enviado no corpo da requisição.
- Header opcional recomendado para transferências: `idempotency-key` (UUID) para evitar duplicatas.

## Como Visualizar

1. **Swagger Editor**: Abra [Swagger Editor](https://editor.swagger.io/), use *File → Import file* e selecione `openapi.yaml`.
2. **Swagger UI**: Sirva o arquivo com qualquer servidor que suporte Swagger UI (ex.: `docker run -p 8080:8080 -e SWAGGER_JSON=/openapi.yaml -v $(pwd)/openapi.yaml:/openapi.yaml swaggerapi/swagger-ui`) ou use extensões de IDE.
3. **Validação**: No Swagger Editor, verifique se não há erros de sintaxe ou schema na aba de mensagens.

## Estrutura dos Arquivos

```
swagger/envio_qualipoints/
├── openapi.yaml   # Especificação OpenAPI 3.0.3
└── README.md      # Este arquivo
```

## Gaps ou Ambiguidades

- **Rate limiting**: Endpoint `GET /transactions` documenta 429, mas limites concretos (ex.: requisições/minuto) devem ser definidos na implementação.
- **Servidor**: URL base `https://api.exemplo.com` é placeholder; ajustar em `servers.url` ou por ambiente.
- **Filtros de histórico**: Campos `sender_id`/`recipient_id` como filtros opcionais não foram incluídos na primeira versão; podem ser adicionados se a API evoluir.

## Sugestões de Melhoria

- **REST**: Manter convenção de plural para recursos (`/transactions`, `/wallets/me`).
- **Versionamento**: Base path `/api/v1` já suporta evolução; manter compatibilidade em v1 ao adicionar campos opcionais.
- **Exemplos**: Incluir mais exemplos de `ErrorResponse` para 404 e 422 no YAML quando necessário para testes.
- **Paginação**: Considerar cursor-based em vez de offset para grandes volumes (evitar offset muito alto).

## Próximos Passos

1. Validar `openapi.yaml` no [Swagger Editor](https://editor.swagger.io/).
2. Ajustar `servers.url` conforme ambiente (dev/staging/prod).
3. Usar o contrato para gerar clientes (OpenAPI Generator) ou validar respostas em testes.
