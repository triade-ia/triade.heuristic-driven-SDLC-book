# Especificação Técnica: Envio de QualiPoints (v1.0)

## 1. Resumo do Valor

Permitir que usuários autenticados transfiram pontos de sua carteira para outros usuários ativos, garantindo a integridade do saldo e a rastreabilidade da operação.

## 2. Definições Técnicas (O "Como")

### Protocolo de Segurança
Autenticação via JWT (o `user_id` deve ser extraído do token, nunca enviado manualmente no corpo da requisição).

### Atomicidade (Banco de Dados)
A operação deve usar uma transação de banco de dados. Se o crédito no destinatário falhar, o débito no remetente deve sofrer rollback automático.

### Idempotência
A API deve aceitar um `idempotency-key` no header para evitar transferências duplicadas em caso de cliques repetidos ou retentativas de rede.

## 3. Regras de Negócio Inegociáveis

### Limites
- Valor mínimo de 1 QualiPoint
- O valor máximo é o saldo disponível do remetente

### Status do Destinatário
O ID do destinatário deve ser validado antes do processamento; ele precisa existir e estar com a conta em estado "Ativo".

### Sincronicidade
A transação é síncrona. O usuário recebe a confirmação (Sucesso ou Erro) em tempo real.

## 4. Contrato de API (Mínimo)

### Endpoint
```
POST /api/v1/transactions
```

### Request
```json
{
  "recipient_id": "string",
  "amount": "number"
}
```

### Resposta de Sucesso (201)
```json
{
  "transaction_id": "uuid",
  "new_balance": "number"
}
```

### Tratamento de Erros

- **402 Payment Required**: Saldo insuficiente
- **404 Not Found**: Destinatário inexistente
- **422 Unprocessable Entity**: Valor abaixo do mínimo ou formato inválido

## 5. Fora do Escopo (Decisões de Economia)

- ❌ Geração de comprovantes em PDF (substituído por log em tela)
- ❌ Confirmação obrigatória via Painel Web (será apenas visualização de histórico)
