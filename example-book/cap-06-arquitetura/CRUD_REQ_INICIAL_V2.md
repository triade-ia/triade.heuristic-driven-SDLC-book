# Relatório: CRUD — REQ_INICIAL_V2 (QualiPoints)

**Requisito analisado:** Envio de QualiPoints entre usuários (REQ_INICIAL_V2).
**Skill aplicada:** Cap06_arquitetura — CRUD (Create, Read, Update, Delete)
**Data:** 2026-02-18
**Status:** Análise de arquitetura concluída

---

## 1. Entidades Identificadas para Análise CRUD

Antes de aplicar a lente CRUD, mapeamos as entidades de dados envolvidas na funcionalidade de envio de QualiPoints:

| Entidade | Descrição | Tabela / Artefato |
|----------|-----------|-------------------|
| **Transferência (Transação)** | Registro de cada operação de envio de QualiPoints | `transactions` |
| **Saldo (Carteira)** | Saldo atual de QualiPoints de cada usuário | `wallets` / `balances` |
| **Idempotency Key** | Chave de idempotência por intenção de envio | `idempotency_keys` |
| **Usuário** | Dados e status de conta do remetente/destinatário | `users` |
| **Lista de Transações Recentes** | View derivada das últimas 10 transações do usuário | (query sobre `transactions`) |

---

## 2. Análise CRUD por Entidade

### 2.1 Transferência (Transação)

#### Create — Criação da Transação

| Questão | Análise |
|---------|---------|
| Como a transação é criada? | `POST /api/v1/transfers` — dentro de transação atômica de banco após todas as validações passarem |
| Quais validações precedem a inserção? | Autenticação (token válido), saldo suficiente, destinatário ativo e diferente do remetente, valor entre 1–10.000 |
| A criação é idempotente? | **Sim** — controlada pela Idempotency-Key; a mesma intenção jamais gera dois registros |
| Quem gera o `transactionId`? | **Lacuna:** o requisito não especifica se o ID é gerado pelo cliente (UUID no App) ou pelo servidor (banco auto-increment, UUID server-side) |
| Campos obrigatórios da transação? | `senderId` (implícito pelo token), `recipientId`, `amount`, `status = "completed"`, `completedAt` — ver seção 10.4 do requisito |
| Há campos opcionais não especificados? | **Lacuna:** `description`, `category`, `fee` (se houver) e `errorCode` (para transações que falharam) não estão definidos na estrutura da tabela |
| O que acontece em Insert falho (rollback)? | Transação de banco revertida; sem registro na tabela; Idempotency-Key marcada como `Falhou` |
| Rate limiting para criação? | **Lacuna (backlog):** não definido no MVP (seção 4.3 do requisito) |

**Riscos Create:**
- Sem especificação de quem gera o `transactionId`, há risco de conflito entre ID de cliente e ID de servidor.
- Campos da tabela `transactions` não definidos explicitamente — o schema precisa ser formalizado antes do desenvolvimento.

---

#### Read — Leitura de Transações

| Questão | Análise |
|---------|---------|
| Como as transações são lidas? | `GET /api/v1/transfers` (ou equivalente) — não definido explicitamente no contrato da API do requisito |
| Qual endpoint retorna a listagem? | **Lacuna crítica:** o requisito descreve a lista de transações recentes (seção 6.3), mas não define o endpoint de leitura (somente o endpoint de criação `POST /api/v1/transfers` está especificado) |
| Quais filtros são aplicados? | Apenas pelo `user_id` do usuário autenticado — sem filtros avançados no MVP |
| Controle de acesso: quem vê o quê? | Cada usuário vê **apenas suas próprias** transações (como remetente ou destinatário) — correto e especificado |
| O endpoint retorna dados do destinatário/remetente? | **Lacuna:** o corpo de resposta da listagem não está definido — apenas a resposta do `POST` está especificada (seção 10.4) |
| Paginação está implementada? | **Não no MVP** — retorna fixo as últimas 10 transações sem cursor ou offset |
| Há índice para a query de leitura? | **Lacuna:** índice em `(user_id, created_at DESC)` não está especificado |
| A listagem inclui transações com status `Falhou`? | **Lacuna:** o requisito menciona apenas `status: "completed"` na resposta do `POST`; não está claro se transações que falharam aparecem na lista |
| Consistência de leitura após envio? | **App:** atualiza saldo e lista após HTTP 200. **Painel web:** polling/refresh manual (seção 13 do requisito). Eventual consistency aceitável no MVP |

**Riscos Read:**
- **Endpoint de leitura não definido no contrato da API** — maior lacuna: o requisito especifica o comportamento (`GET` retorna últimas 10), mas não o endpoint.
- **Corpo da resposta de listagem não definido** — campos retornados, formato de data, inclusão ou não de dados do remetente/destinatário estão ausentes.
- **Transações com falha na lista** — comportamento não definido para transações que foram tentadas mas falharam.

---

#### Update — Atualização de Transações

| Questão | Análise |
|---------|---------|
| Há operação de Update sobre transações? | **Não** — transações concluídas são **imutáveis** por design (seção 8.1: "estados finais `Concluída` e `Falhou` são imutáveis") |
| Existe estorno (reversão de transação)? | **Não no MVP** — estorno fica para backlog (seção 7.1 do requisito via STATE_ANALYSIS) |
| Alguém pode alterar o valor ou destinatário de uma transação? | **Não** — a imutabilidade é invariante de segurança (I-1 da análise STATE_ANALYSIS) |
| Há risco de Update acidental? | Sem endpoint de `PUT/PATCH /api/v1/transfers/:id`, o risco é baixo. Garantir que não há rota exposta inadvertidamente |

**Avaliação Update:** Comportamento correto por design — transações não devem ser atualizadas. A ausência intencional de operação de Update é uma decisão arquitetural adequada para auditoria. **Verificar que nenhuma rota de Update está exposta na API.**

---

#### Delete — Exclusão de Transações

| Questão | Análise |
|---------|---------|
| Há operação de Delete sobre transações? | **Não no MVP** — registros de transação são permanentes para fins de auditoria (seção 8.3 do requisito) |
| Exclusão lógica (soft delete) ou física (hard delete)? | **Não definido** — o requisito não menciona coluna `deleted_at` ou `is_deleted`; assume-se hard-permanent (sem exclusão) |
| Há retenção de dados definida? | **Lacuna:** período de retenção de registros de transação não especificado |
| Há exclusão em cascata se o usuário for deletado? | **Lacuna:** comportamento das transações caso o usuário remetente ou destinatário seja excluído não está definido |

**Riscos Delete:**
- **Sem política de retenção** — em mercados regulados (financeiro, LGPD), a retenção de transações pode ser obrigatória por N anos.
- **Cascade delete não definido** — se um usuário for excluído do sistema, o que acontece com o histórico de transações que o referenciam? Risco de violação de integridade referencial ou perda de histórico de auditoria.

---

### 2.2 Saldo (Carteira)

#### Create — Criação da Carteira

| Questão | Análise |
|---------|---------|
| Quando e como a carteira é criada? | **Fora do escopo do requisito** — o requisito assume que a carteira já existe para usuários ativos |
| Saldo inicial? | **Não definido** neste requisito — escopo de outro módulo |

---

#### Read — Leitura de Saldo

| Questão | Análise |
|---------|---------|
| Onde o saldo é consultado? | **Internamente na API** — antes de autorizar o débito (validação de saldo suficiente, seção 9, regra 4) |
| Há endpoint público de consulta de saldo? | **Lacuna:** o requisito menciona "painel web para consulta de saldo" (seção 1), mas não define o endpoint de leitura de saldo |
| O App exibe saldo atualizado após envio? | **Sim** — após HTTP 200, o App atualiza o saldo e lista de recentes (seção 11.1) |
| Leitura de saldo é consistente com a escrita? | A leitura de saldo para validação ocorre **dentro da transação atômica** — consistência forte. A exibição no App/painel é via refresh/polling — consistência eventual |

**Lacuna Read de Saldo:** Endpoint de consulta de saldo não definido no contrato da API. Necessário para que o App exiba o saldo atualizado após o envio.

---

#### Update — Atualização de Saldo

| Questão | Análise |
|---------|---------|
| Como o saldo é atualizado? | **Débito do remetente + Crédito do destinatário** em transação atômica de banco (seção 6.2 do requisito) |
| O Update é direto (UPDATE na tabela de saldo) ou por evento (INSERT em tabela de movimentação)? | **Lacuna arquitetural crítica:** o requisito não define se o saldo é armazenado como valor atual (mutable, UPDATE direto) ou como agregado de eventos de crédito/débito (event sourcing, INSERT apenas) |
| Há controle de versão / optimistic locking no saldo? | **Lacuna:** o mecanismo de lock não está especificado — pessimistic (`SELECT FOR UPDATE`) vs. optimistic (campo `version`) |
| Quais campos são atualizados? | Apenas `balance` (saldo atual) — sem outros campos definidos |
| Há log de cada atualização de saldo? | A inserção da transação na tabela `transactions` funciona como log implícito das movimentações |

**Risco crítico Update de Saldo:** Sem especificação do modelo de dados (saldo mutável vs. event sourcing) e do mecanismo de lock, há risco de:
1. **Saldo negativo** por race condition em envios simultâneos;
2. **Inconsistência de auditoria** se o saldo for atualizado diretamente sem rastreamento de movimentação.

**Recomendação arquitetural:** Avaliar se o modelo de saldo deve ser:
- **Opção A (Simples — adequada para MVP):** Coluna `balance` na tabela `wallets` atualizada por `UPDATE ... WHERE user_id = ? AND balance >= amount` (condição atômica que evita saldo negativo) com `SELECT FOR UPDATE`.
- **Opção B (Event Sourcing — mais robusto):** Tabela de movimentações (`wallet_movements`) com INSERT de débito e crédito; saldo calculado por `SUM(amount)`. Mais auditável, mas mais complexo.

---

#### Delete — Exclusão de Saldo

| Questão | Análise |
|---------|---------|
| Há operação de Delete na carteira/saldo? | **Não no MVP** |
| O que acontece se a conta for encerrada? | **Lacuna:** comportamento do saldo em conta encerrada não está definido |

---

### 2.3 Idempotency Key

#### Create — Inserção da Key

| Questão | Análise |
|---------|---------|
| Como a key é criada? | Inserção atômica no início do processamento — antes do débito/crédito |
| Há UNIQUE constraint na coluna da key? | **Lacuna:** não especificado, mas necessário para prevenir race condition de inserção dupla |
| A key e a transação são criadas na mesma transação atômica? | **Deve ser** — ver análise STATE_ANALYSIS (R-1: lacuna crítica) |
| Quem gera o valor da key? | O cliente (App) gera um UUID ao tocar em "Enviar" (seção 8.2 do requisito) |

---

#### Read — Consulta da Key

| Questão | Análise |
|---------|---------|
| Como a key é consultada? | `SELECT ... WHERE key = ?` — deve ocorrer no início de cada requisição, antes de qualquer processamento |
| Performance da consulta? | Com índice UNIQUE na coluna da key, a consulta é `O(1)` — adequado |
| O resultado cacheado (resposta da primeira execução) está armazenado junto à key? | **Deve estar** — o corpo de resposta HTTP (código + corpo JSON) deve ser persistido junto à key para retornar exatamente a mesma resposta em retry |

---

#### Update — Atualização da Key

| Questão | Análise |
|---------|---------|
| A key é atualizada após o processamento? | **Sim** — `Processando → Concluída` ou `Processando → Falhou`, junto com o resultado persistido |
| O Update ocorre na mesma transação atômica do débito/crédito? | **Deve ocorrer** — crítico para evitar key órfã (ver STATE_ANALYSIS, L-2) |

---

#### Delete — Expiração da Key

| Questão | Análise |
|---------|---------|
| As keys são excluídas após certo tempo? | **Lacuna:** sem TTL definido — as keys acumulam indefinidamente (ver COUNT, L-3) |
| A exclusão é lógica (marcar como expirada) ou física? | **Não definido** |
| O que acontece se o cliente tentar retry após a key expirar? | **Lacuna:** o requisito não define o comportamento pós-expiração — nova intenção? Erro 404? |

---

### 2.4 Usuário

#### Create — Criação de Usuário

**Fora do escopo** deste requisito. O requisito assume usuários já existentes.

---

#### Read — Consulta de Usuário

| Questão | Análise |
|---------|---------|
| Para que o usuário é consultado? | **Validação do destinatário:** verificar se `recipientId` existe e está ativo (seção 3.2 do requisito) |
| Quais campos são consultados? | `user_id` (existência) e `status` (ativo/bloqueado) — no mínimo |
| A consulta usa índice? | Consulta por `user_id` (chave primária) — `O(1)`, sem lacuna |
| Há cache do status do usuário? | **Risco:** se o status for cacheado, um bloqueio de conta pode não ser refletido imediatamente — TOCTOU (ver STATE_ANALYSIS, I-9) |
| O remetente também é consultado para validar status? | **Sim** — seção 3.1: remetente deve estar ativo. A verificação é implicit via token, mas o status ativo deve ser validado na API, não apenas na autenticação |

---

#### Update — Atualização de Usuário

**Fora do escopo** deste requisito (bloqueio/ativação de conta fica para admin/suporte — seção 3.3 do requisito).

---

#### Delete — Exclusão de Usuário

**Fora do escopo** deste requisito.

---

## 3. Matriz CRUD Consolidada

| Entidade | C (Create) | R (Read) | U (Update) | D (Delete) | Lacunas Críticas |
|----------|-----------|---------|-----------|-----------|-----------------|
| **Transferência** | ✅ Definido (POST /transfers) | ⚠️ Parcial (comportamento definido, endpoint não especificado) | ✅ Não aplicável (imutável por design) | ✅ Não aplicável (permanente para auditoria) | Endpoint GET ausente, corpo da resposta de listagem não definido |
| **Saldo** | — (fora do escopo) | ⚠️ Parcial (leitura interna OK, endpoint público ausente) | ⚠️ Modelo não definido (UPDATE direto vs. event sourcing, lock não especificado) | — (não aplicável) | Modelo de atualização de saldo e mecanismo de lock não definidos |
| **Idempotency Key** | ✅ Definido conceitualmente | ✅ Definido conceitualmente | ✅ Definido conceitualmente | ❌ Não definido (sem TTL/expiração) | TTL e comportamento pós-expiração ausentes |
| **Usuário** | — (fora do escopo) | ✅ Definido (validação de destinatário) | — (fora do escopo) | — (fora do escopo) | Cascade delete em transações se usuário for excluído |

---

## 4. Análise de Segurança e Controle de Acesso por Operação

| Operação | Quem Pode Executar | Controle Definido | Lacunas |
|----------|-------------------|-------------------|---------|
| **Create Transferência** | Usuário autenticado com conta ativa | ✅ Token Bearer + validação de status | Sem rate limiting no MVP |
| **Read Transações** | Usuário autenticado — apenas suas próprias | ✅ Filtro por `user_id` do token | Endpoint não especificado no contrato |
| **Read Saldo** | Usuário autenticado — apenas seu próprio | ⚠️ Implícito — endpoint não definido | Endpoint ausente no contrato |
| **Update Saldo** | Apenas a API (processo interno) — nunca pelo cliente | ✅ Não há endpoint público de Update de saldo | Mecanismo de lock não definido |
| **Update Idempotency Key** | Apenas a API (processo interno) | ✅ Não há endpoint público de Update de key | — |
| **Delete Idempotency Key** | Job interno de limpeza (a definir) | ❌ Não definido | Sem política de expiração |
| **Create/Update/Delete Usuário** | Fora do escopo | — | Cascade em transações não definido |

**Invariante de segurança crítica:** Nenhuma das operações de escrita (débito, crédito, registro de transação) pode ser iniciada por chamada direta do cliente — todas são consequência de um único `POST /api/v1/transfers` autenticado, processado integralmente no servidor.

---

## 5. Análise de Consistência por Operação

| Operação | Modelo de Consistência | Justificativa | Risco |
|----------|----------------------|---------------|-------|
| **Create Transferência + Update Saldo** | **Forte (ACID)** — transação atômica de banco | Débito + crédito + registro devem ser atômicos (seção 6.2) | ✅ Correto por design |
| **Read Transações (listagem)** | **Eventual** — polling/refresh | App atualiza após HTTP 200; painel atualiza no próximo refresh (seção 13) | ✅ Aceitável no MVP, documentar para o usuário |
| **Read Saldo (painel web)** | **Eventual** — polling | Painel não reflete em tempo real | ✅ Aceitável no MVP |
| **Idempotency Key** | **Forte** — deve estar na mesma transação | Key e saldo devem ser consistentes após crash | ⚠️ Risco de key órfã se não for atômica (ver STATE_ANALYSIS) |

---

## 6. Lacunas e Riscos Consolidados

| # | Lacuna / Risco | Operação | Severidade | Recomendação |
|---|----------------|----------|------------|--------------|
| L-1 | **Endpoint de leitura de transações não definido** | Read | Alta | Definir `GET /api/v1/transfers` (ou similar) com autenticação e retorno das últimas 10 transações |
| L-2 | **Corpo de resposta da listagem não definido** | Read | Alta | Especificar campos retornados: `transactionId`, `amount`, `senderId`, `recipientId`, `status`, `createdAt` |
| L-3 | **Endpoint de leitura de saldo não definido** | Read | Alta | Definir `GET /api/v1/balance` (ou similar) para App e painel web |
| L-4 | **Modelo de atualização de saldo não definido** | Update | Alta | Decidir entre UPDATE direto com `SELECT FOR UPDATE` vs. event sourcing antes do desenvolvimento |
| L-5 | **Mecanismo de lock no saldo não especificado** | Update | Alta | Especificar pessimistic locking (`SELECT FOR UPDATE`) ou optimistic locking com campo `version` |
| L-6 | **TTL e Delete da Idempotency Key não definidos** | Delete | Média | Definir política de expiração (24h–7 dias) e comportamento pós-expiração |
| L-7 | **Schema da tabela `transactions` não formalizado** | Create / Read | Média | Definir colunas: `id`, `sender_id`, `recipient_id`, `amount`, `status`, `idempotency_key`, `created_at`, `completed_at` |
| L-8 | **Cascade delete de transações não definido** | Delete (indireto) | Baixa (backlog) | Definir comportamento das transações se o usuário remetente ou destinatário for excluído |
| L-9 | **Transações com status `Falhou` na listagem** | Read | Média | Definir se transações rejeitadas aparecem na lista de recentes (atualmente o requisito cita apenas status `completed`) |
| L-10 | **Geração de `transactionId` não definida** | Create | Média | Especificar se o ID é gerado pelo servidor (UUID v4) ou pelo banco (auto-increment); recomendar UUID v4 server-side |

---

## 7. Cenários de Teste Derivados da Heurística CRUD

### 7.1 Create (Criação)

```typescript
// Unitário — validações antes de criar
it('deve rejeitar criação com amount = 0 antes de acessar o banco', () => {
  const validator = new TransferValidator();
  const result = validator.validate({ recipientId: 'user-2', amount: 0 });
  expect(result.isValid).toBe(false);
  expect(result.errorCode).toBe('INVALID_AMOUNT');
});

// Integração — inserção no banco
it('deve inserir transação na tabela transactions após débito e crédito', async () => {
  await transfer(user1, user2, 100);
  const tx = await db.query('SELECT * FROM transactions WHERE sender_id = ?', [user1.id]);
  expect(tx).toHaveLength(1);
  expect(tx[0].amount).toBe(100);
  expect(tx[0].status).toBe('completed');
});

// Serviço — API POST completa
it('deve criar transferência via POST /api/v1/transfers com sucesso', async () => {
  const response = await post('/api/v1/transfers', {
    recipientId: user2.id,
    amount: 100
  }, { 'Idempotency-Key': uuid() });
  expect(response.status).toBe(200);
  expect(response.body.transactionId).toBeDefined();
  expect(response.body.status).toBe('completed');
});

// Unicidade — idempotência no Create
it('deve retornar o mesmo transactionId em retentativa com mesma key (CRUD Create idempotente)', async () => {
  const key = uuid();
  const first = await post('/api/v1/transfers', { recipientId: user2.id, amount: 100 }, { 'Idempotency-Key': key });
  const retry = await post('/api/v1/transfers', { recipientId: user2.id, amount: 100 }, { 'Idempotency-Key': key });
  expect(retry.body.transactionId).toBe(first.body.transactionId);
  // Verificar que o saldo foi debitado apenas uma vez
  expect(await getBalance(user1)).toBe(initialBalance - 100);
});
```

### 7.2 Read (Leitura)

```typescript
// Integração — query de listagem
it('deve retornar apenas as 10 transações mais recentes do usuário (CRUD Read)', async () => {
  await createManyTransfers(user1, user2, 15, 10); // 15 transfers de 10 pontos
  const txs = await transactionRepository.findRecentByUser(user1.id, 10);
  expect(txs).toHaveLength(10);
  expect(txs[0].createdAt).toBeAfter(txs[9].createdAt); // Mais recente primeiro
});

// Serviço — controle de acesso
it('usuário A não deve ver transações do usuário B (CRUD Read — controle de acesso)', async () => {
  await createTransfer(user2, user3, 50);
  const response = await getAs(user1, '/api/v1/transfers');
  expect(response.body).not.toContainEqual(
    expect.objectContaining({ senderId: user2.id })
  );
});

// Serviço — resposta para zero transações
it('deve retornar [] para usuário sem transações (CRUD Read — zero)', async () => {
  const response = await getAs(newUser, '/api/v1/transfers');
  expect(response.status).toBe(200);
  expect(response.body).toEqual([]);
});

// E2E — visualização da lista
it('deve exibir as 10 transações mais recentes no App (CRUD Read E2E)', async ({ page }) => {
  await page.goto('/transactions');
  const rows = page.locator('[data-testid="transaction-item"]');
  await expect(rows).toHaveCount(10);
});
```

### 7.3 Update (Atualização) — Verificar Imutabilidade

```typescript
// Segurança — transação não pode ser atualizada
it('não deve existir endpoint PUT/PATCH /api/v1/transfers/:id (CRUD Update — ausente por design)', async () => {
  const response = await put(`/api/v1/transfers/${transactionId}`, { amount: 999 });
  expect(response.status).toBe(404); // Rota não existe
});

// Integração — saldo atômico: débito + crédito juntos ou nenhum
it('deve reverter débito e crédito em caso de falha no registro da transação (CRUD Update — rollback)', async () => {
  const initialSenderBalance = await getBalance(user1);
  const initialRecipientBalance = await getBalance(user2);

  // Simular falha ao inserir na tabela de transações
  jest.spyOn(transactionRepository, 'create').mockRejectedValueOnce(new Error('DB Error'));

  await expect(transferService.send(user1, user2, 100)).rejects.toThrow();

  // Saldos devem estar inalterados
  expect(await getBalance(user1)).toBe(initialSenderBalance);
  expect(await getBalance(user2)).toBe(initialRecipientBalance);
});
```

### 7.4 Delete (Exclusão) — Verificar Permanência

```typescript
// Segurança — transação não pode ser deletada
it('não deve existir endpoint DELETE /api/v1/transfers/:id (CRUD Delete — ausente por design)', async () => {
  const response = await deleteRequest(`/api/v1/transfers/${transactionId}`);
  expect(response.status).toBe(404); // Rota não existe
});

// Integração — transação permanece após qualquer operação
it('a transação concluída deve persistir mesmo após novo envio (CRUD Delete — imutável)', async () => {
  const { transactionId } = await transfer(user1, user2, 100);
  await transfer(user1, user2, 50); // Novo envio

  const original = await transactionRepository.findById(transactionId);
  expect(original).not.toBeNull();
  expect(original.amount).toBe(100);
});
```

---

## 8. Recomendações por Prioridade

### Para o MVP (inegociável)

| # | Recomendação | Justificativa |
|---|-------------|---------------|
| R-1 | Definir endpoint `GET /api/v1/transfers` com autenticação e retorno das últimas 10 transações | Sem este endpoint o painel web e a atualização pós-envio no App não funcionam |
| R-2 | Definir endpoint `GET /api/v1/balance` (ou equivalente) para leitura de saldo | O App precisa exibir saldo atualizado após envio |
| R-3 | Formalizar schema da tabela `transactions` (colunas, tipos de dado, constraints) | Base para desenvolvimento do backend |
| R-4 | Especificar mecanismo de lock no saldo (pessimistic `SELECT FOR UPDATE` recomendado para MVP) | Previne race condition e saldo negativo |
| R-5 | Definir UNIQUE constraint na coluna da Idempotency-Key | Previne race condition na inserção dupla de key |
| R-6 | Definir corpo da resposta da listagem de transações | Contrato de API incompleto sem esta especificação |

### Para o Backlog

| # | Recomendação |
|---|-------------|
| R-7 | Definir TTL da Idempotency-Key e job de limpeza periódico |
| R-8 | Definir comportamento de cascade quando usuário é excluído (transações ficam com `sender_id` inválido?) |
| R-9 | Definir política de retenção de registros de transação (compliance LGPD) |
| R-10 | Avaliar migração para event sourcing no saldo para auditoria histórica completa |
| R-11 | Implementar endpoint de paginação para extrato completo (`GET /api/v1/transfers?cursor=...`) |
| R-12 | Definir se transações com status `Falhou` aparecem na listagem de recentes |

---

## 9. Heurísticas Complementares Recomendadas

### Count (0, 1, Muitos)
**Conexão com CRUD:** As operações de Read (listagem das últimas 10) e Create (volume de inserções simultâneas) são diretamente impactadas pelos cenários de volume.

**Recomendação:** Aplicar **Count** para garantir que a listagem funciona corretamente com 0 transações (empty state), 1 transação (caso base) e N transações (indexação, performance). Ver [COUNT_REQ_INICIAL_V2.md](./COUNT_REQ_INICIAL_V2.md).

### State Analysis (Máquinas de Estado)
**Conexão com CRUD:** O fluxo de Create da transação percorre estados (Processando → Concluída/Falhou). A operação de Update do saldo é condicional ao estado do processo atômico.

**Recomendação:** As lacunas de Update de saldo (mecanismo de lock) e Create da Idempotency-Key (estado intermediário órfão) foram aprofundadas na análise State Analysis — ver [STATE_ANALYSIS_REQ_INICIAL_V2.md](../cap-05-design/STATE_ANALYSIS_REQ_INICIAL_V2.md).

### FAILURE
**Conexão com CRUD:** Falhas nas operações de Create (rollback) e Update (lock timeout, deadlock) são os cenários mais críticos de resiliência.

**Recomendação:** Aplicar **FAILURE** para mapear o comportamento de cada operação CRUD em cenário de falha — especialmente o que acontece se o banco cair durante a transação atômica, ou se o lock de saldo expirar.

---

## Referências

- **Requisito analisado:** [REQ_INICIAL_V2.md](../requisito-revisado/REQ_INICIAL_V2.md)
- **Skill aplicada:** [CRUD.md](../../Skill/Cap06_arquitetura/CRUD.md)
- **Análises complementares já realizadas:** [STATE_ANALYSIS_REQ_INICIAL_V2.md](../cap-05-design/STATE_ANALYSIS_REQ_INICIAL_V2.md), [DEPENDENCIES_REQ_INICIAL_V2.md](../cap-05-design/DEPENDENCIES_REQ_INICIAL_V2.md), [COUNT_REQ_INICIAL_V2.md](./COUNT_REQ_INICIAL_V2.md)
- **Próximas heurísticas recomendadas:** FAILURE, Position (ver seção 9)
