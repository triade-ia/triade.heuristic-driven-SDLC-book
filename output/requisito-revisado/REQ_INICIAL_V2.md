# Requisito Revisado v2: Envio de QualiPoints entre usuários

**Versão:** 2  
**Baseado em:** REQ_INICIAL + relatórios User Scenarios e WWWWWHKE (Cap 04 — Concepção)  
**Data:** 2025-02-18  
**Status:** Pronto para desenvolvimento (refinamento aplicado)

---

## 1. Descrição e objetivo

O usuário deve poder enviar QualiPoints para outro usuário de forma **rápida e segura**. O sistema oferece tela de envio no App, API REST que processa a operação e painel web para **consulta** (sem aprovação manual no fluxo de envio). A operação é **síncrona**: o usuário espera confirmação na hora, com feedback claro de estado (processando / concluído / falhou).

---

## 2. Contexto do projeto

- **Produto:** Carteira digital QualiPoints.
- **Canais:** App mobile (envio e consulta), painel web (consulta de extrato e saldo).
- **Stack:** Banco de dados relacional, comunicação REST API, HTTPS.
- **Escopo MVP:** Transferência entre usuários; lista simples de transações recentes; sem comprovante em PDF, e-mail automático ou aprovação manual no painel.

---

## 3. Quem (autorização e identidade)

### 3.1 Remetente

- Apenas usuário **autenticado** (logado) pode enviar QualiPoints.
- Conta do remetente deve estar no status **ativo** (contas bloqueadas, pendentes ou em análise não podem enviar).
- Remetente é identificado pelo token de autenticação; não é informado no corpo da requisição.

### 3.2 Destinatário

- Deve ser um **ID de usuário válido** existente na base.
- Conta do destinatário deve estar **ativa** e **elegível para receber** (não bloqueada).
- **Envio para o próprio usuário (mesmo ID do remetente) não é permitido** — a API rejeita com erro específico.

### 3.3 Perfis

- No MVP, apenas o perfil **usuário final** envia e recebe; papéis de admin/suporte para desbloqueio ou conciliação ficam para backlog.

---

## 4. O quê (objeto e limites)

### 4.1 Objeto transferido

- **QualiPoints:** valor em pontos, **inteiro**, sem casas decimais.
- Unidade: 1 QualiPoint = 1 unidade (sem conversão para outra moeda no escopo deste requisito).

### 4.2 Limites por transação

| Regra | Valor |
|-------|--------|
| **Mínimo** | 1 QualiPoint |
| **Máximo** | 10.000 QualiPoints por transação |

Valores **zero** ou **negativos** são rejeitados pela API.

### 4.3 Limites por período (backlog)

- Limite por dia/mês e rate limit (transações por minuto/hora) ficam para refinamento posterior; no MVP não há limite além do máximo por transação.

---

## 5. Quando (temporalidade e concorrência)

### 5.1 Modo de processamento

- **Síncrono:** o cliente (App) envia a requisição e **espera** a resposta da API com o resultado (sucesso ou erro). Não há fila assíncrona para o envio no MVP.

### 5.2 Timeout e expiração

- **Timeout da API:** a API deve responder em até **10 segundos**. Se o processamento exceder esse tempo, a API retorna erro **504 Gateway Timeout** (ou 408 Request Timeout, conforme convenção do projeto).
- **Comportamento do cliente:** em caso de timeout ou falha de rede, o cliente pode **repetir a requisição** usando a **mesma chave de idempotência** (evitando débito duplo). Não há estado "pendente" persistido no servidor para retry posterior pelo usuário; ou a transação foi concluída no momento da chamada, ou o cliente pode tentar de novo com a mesma idempotency key.

### 5.3 Concorrência

- A atualização de saldo (débito do remetente + crédito do destinatário + registro da transação) é **atômica** no servidor (transação de banco de dados ou equivalente), de forma que duas requisições simultâneas do mesmo usuário não resultem em saldo inconsistente. A **idempotency key** evita processar duas vezes o mesmo envio.

---

## 6. Onde (arquitetura e validação)

### 6.1 Onde a regra é validada

- **Toda** validação de negócio (saldo, destinatário, valor, envio para si mesmo) é feita **no servidor (API)**. O App pode exibir feedback de UX (ex.: campo obrigatório), mas a **fonte da verdade** é a API.
- Nenhuma decisão de débito/crédito depende apenas do cliente.

### 6.2 Persistência e consistência

- Dados persistidos em **banco relacional**: saldos (ou tabela de carteira), tabela de transações.
- A operação de envio é **atômica**: débito do remetente, crédito do destinatário e inserção do registro da transação ocorrem na mesma transação de banco (ou mecanismo equivalente). Se qualquer passo falhar, toda a operação é revertida (rollback).

### 6.3 Lista de transações recentes

- A lista é gerada a partir da **mesma fonte de verdade** (tabela de transações). No MVP: **últimas 10 transações** do usuário autenticado, ordenadas por data/hora mais recente primeiro. Sem filtros avançados no primeiro release.
- **Controle de acesso:** cada usuário vê **apenas as próprias** transações (como remetente ou destinatário); a API filtra por identificador do usuário logado.

---

## 7. Por que (escopo MVP — Keep e Eliminate)

### 7.1 Manter (inegociável)

- Autenticação do usuário para enviar.
- Validação server-side de saldo, destinatário, valor e regra "não enviar para si mesmo".
- Operação atômica (débito + crédito + registro).
- Registro de transação para auditoria e lista recente.
- HTTPS e uso de idempotency key para evitar débito duplo em retentativas.

### 7.2 Eliminar do fluxo de envio no MVP

- **Aprovação manual no painel web** para concluir o envio — no MVP o painel web serve apenas para **consulta** (saldo e extrato). O envio é feito e confirmado no App (e a API).
- Comprovante em PDF, e-mail automático por transação e notificação push ficam para backlog.
- Lista "rica" de transações (muitos filtros, muitos registros) — no MVP apenas últimas 10, sem filtros.

---

## 8. Como (transição de estado e segurança)

### 8.1 Atomicidade

- O sistema só debita o remetente **se** o crédito ao destinatário e o registro da transação forem persistidos com sucesso. Caso contrário, rollback e resposta de erro ao cliente.

### 8.2 Idempotência

- O cliente **deve** enviar um header (ex.: `Idempotency-Key`) com valor único por **intenção de envio** (ex.: UUID gerado no App ao tocar em "Enviar"). A API:
  - Na **primeira** requisição com essa chave: processa e grava o resultado; em sucesso, retorna 200 e o corpo com dados da transação.
  - Em **requisições subsequentes** com a mesma chave: não reprocessa; retorna o **mesmo** resultado da primeira (200 e mesmo corpo, ou o erro que foi retornado na primeira vez), para evitar débito duplo em retry.

### 8.3 Segurança

- Comunicação **HTTPS** obrigatória.
- Autenticação via token (ex.: Bearer) na API; remetente identificado pelo token.
- Auditoria: toda tentativa de envio (sucesso ou falha) deve ser registrada para suporte e análise (ex.: log com idempotency key, usuário, valor, destinatário, resultado).

---

## 9. Regras de negócio (consolidadas)

1. Usuário deve estar **logado** e com conta **ativa** para enviar.
2. É obrigatório informar **ID do destinatário** e **valor** (inteiro, entre 1 e 10.000).
3. **Destinatário** deve existir, estar ativo e ser **diferente** do remetente.
4. **Saldo** do remetente deve ser **suficiente** (saldo ≥ valor). Validação feita na API.
5. Após envio bem-sucedido: saldo do remetente é **debitado**, saldo do destinatário é **creditado**, e a transação é **registrada**.
6. A lista de **transações recentes** exibe as últimas **10** transações do usuário (remetente ou destinatário), ordenadas da mais recente para a mais antiga. Acesso restrito às próprias transações.

---

## 10. Contrato da API de envio

### 10.1 Endpoint e método

- **POST** `/api/v1/transfers` (ou equivalente conforme padrão do projeto).

### 10.2 Headers obrigatórios

| Header | Obrigatório | Descrição |
|--------|-------------|-----------|
| `Authorization` | Sim | Token de autenticação (ex.: `Bearer <token>`). |
| `Idempotency-Key` | Sim | String única por intenção de envio (ex.: UUID). Usado para evitar processamento duplicado. |
| `Content-Type` | Sim | `application/json`. |

### 10.3 Corpo da requisição (body)

```json
{
  "recipientId": "string (ID do usuário destinatário)",
  "amount": "integer (1 a 10000)"
}
```

- **remetente:** implícito pelo token; não enviado no body.

### 10.4 Respostas

| Código HTTP | Situação | Corpo (exemplo) |
|-------------|----------|------------------|
| **200 OK** | Envio processado com sucesso | `{ "transactionId": "...", "amount": 100, "recipientId": "...", "status": "completed", "completedAt": "ISO8601" }` |
| **400 Bad Request** | Dados inválidos (ex.: valor 0, formato errado) | `{ "code": "INVALID_AMOUNT", "message": "Valor deve ser entre 1 e 10000." }` |
| **401 Unauthorized** | Não autenticado ou token inválido | `{ "code": "UNAUTHORIZED", "message": "..." }` |
| **403 Forbidden** | Conta bloqueada / não ativa | `{ "code": "ACCOUNT_NOT_ACTIVE", "message": "..." }` |
| **404 Not Found** | Destinatário não encontrado ou inativo | `{ "code": "RECIPIENT_NOT_FOUND", "message": "Destinatário não encontrado ou inativo." }` |
| **409 Conflict** | Envio para si mesmo | `{ "code": "SELF_TRANSFER_NOT_ALLOWED", "message": "Envio para a própria conta não é permitido." }` |
| **422 Unprocessable Entity** | Saldo insuficiente | `{ "code": "INSUFFICIENT_BALANCE", "message": "Saldo insuficiente.", "currentBalance": 50 }` (opcional) |
| **408/504** | Timeout (cliente ou API) | `{ "code": "REQUEST_TIMEOUT", "message": "Tempo esgotado. Pode tentar novamente com a mesma chave de idempotência." }` |
| **5xx** | Erro interno | `{ "code": "INTERNAL_ERROR", "message": "..." }` |

Em **retentativa com mesma Idempotency-Key**: a API retorna o **mesmo** código e corpo da primeira resposta para essa chave (200 ou erro), sem reprocessar.

---

## 11. Estados da transação e feedback ao usuário

### 11.1 Estados (do ponto de vista do usuário no App)

| Estado | Descrição | Ação na UI |
|--------|-----------|------------|
| **Processando** | Requisição enviada, aguardando resposta da API | Exibir indicador de carregamento; desabilitar novo envio até resposta. |
| **Concluído** | API retornou 200 | Exibir confirmação de sucesso; atualizar saldo e lista de recentes. |
| **Falhou** | API retornou 4xx/5xx ou timeout | Exibir mensagem conforme código (ver mensagens de erro); permitir retry (com mesma Idempotency-Key). |

Não há estado "pendente" persistido no servidor: ou a transação foi concluída na chamada, ou o usuário pode tentar novamente.

### 11.2 Mensagens de erro mapeadas (para exibição no App)

| Código da API | Mensagem sugerida para o usuário |
|---------------|----------------------------------|
| `INVALID_AMOUNT` | "Valor deve ser entre 1 e 10.000 QualiPoints." |
| `RECIPIENT_NOT_FOUND` | "Destinatário não encontrado. Verifique o ID ou o status da conta." |
| `SELF_TRANSFER_NOT_ALLOWED` | "Não é permitido enviar QualiPoints para a própria conta." |
| `INSUFFICIENT_BALANCE` | "Saldo insuficiente. Seu saldo atual não permite este envio." |
| `ACCOUNT_NOT_ACTIVE` | "Sua conta não está ativa para envio. Entre em contato com o suporte." |
| `REQUEST_TIMEOUT` | "A operação demorou mais que o esperado. Tente novamente." |
| `INTERNAL_ERROR` | "Ocorreu um erro. Tente novamente em instantes." |
| `UNAUTHORIZED` | "Sessão expirada. Faça login novamente." |

---

## 12. Comportamento em falha de rede ou timeout

- Se o **cliente** não receber resposta (timeout de rede, conexão caiu): o usuário pode **tentar novamente**. O App deve reenviar a requisição com a **mesma Idempotency-Key** usada na tentativa anterior. A API não processa duas vezes; retorna o resultado já persistido (200 ou o erro original).
- Não há "transação pendente" no servidor para consulta posterior: o fluxo é síncrono com retry seguro via idempotência.
- Opcional (backlog): notificação push ou e-mail em caso de falha após N tentativas; no MVP basta exibir a mensagem na tela e permitir retry.

---

## 13. Consistência entre App e painel web

- Após um envio bem-sucedido no App, o saldo e a lista de transações devem refletir a nova operação quando o usuário **atualizar** o painel web (refresh) ou quando o painel buscar os dados novamente. No MVP não é obrigatório tempo real (WebSocket); **polling** ou refresh manual é aceitável. A **fonte da verdade** é o banco; App e painel consomem a mesma API (ou APIs que leem do mesmo banco).

---

## 14. Edge cases tratados (resumo)

- **Destinatário inexistente ou inativo:** API retorna 404 e código `RECIPIENT_NOT_FOUND`.
- **Valor zero ou negativo / fora do intervalo:** API retorna 400 e código `INVALID_AMOUNT`.
- **Valor maior que o saldo:** API retorna 422 e código `INSUFFICIENT_BALANCE`.
- **Envio para si mesmo:** API retorna 409 e código `SELF_TRANSFER_NOT_ALLOWED`.
- **Timeout ou falha de rede:** Cliente pode retry com mesma Idempotency-Key; API não duplica débito.
- **Duplo clique / múltiplos envios rápidos:** Idempotency-Key evita processar a mesma intenção duas vezes.
- **Listagem de transações:** Apenas as do usuário logado; controle de acesso na API.

---

## 15. Referência aos relatórios de concepção

Este requisito v2 incorpora as melhorias identificadas em:

- **User Scenarios (REQ_INICIAL):** personas (Ana, Roberto, Carla, Bruno), situações de uso, lacunas de feedback/estados, validações, idempotência, escopo de transações recentes e consistência app/painel.
- **WWWWWHKE (REQ_INICIAL):** questionamentos Who/What/When/Where/Why/How, Keep/Eliminate, contrato de API, regras de valor, timeout, atomicidade e lista de priorização.

Com isso, o requisito está **revisado e pronto para desenvolvimento**, atendendo ao mínimo inegociável e reduzindo ambiguidade e riscos técnicos antes da codificação.
