---
name: test-service-guide
description: Guia para implementar testes de serviço (API). Use quando tiver relatório de estratégia (output/test-strategy-*.md) com casos API-XXX e precisar gerar testes HTTP para endpoints (autenticação, validação, códigos de erro) em TypeScript (Supertest) ou Java (RestAssured).
---

# Guia de Testes de Serviço (API)

Implemente testes de API com base no **relatório de estratégia** gerado por [TEST_STRATEGY.md](TEST_STRATEGY.md). Use os casos com prefixo **API-XXX** para gerar testes HTTP completos (request/response, status, body).

## Quando Usar Este Guide

- Relatório contém casos de teste de serviço/API (API-XXX)
- Precisa testar: endpoints HTTP, autenticação, validação de body, códigos 4xx/5xx, idempotência, rollback
- Heurísticas: VADER (Values, Authorization, Data Consistency, Error Handling), FAILURE (Appropriate, Functional)

## Definição e Escopo

Testes de serviço validam **APIs HTTP completas** (request → aplicação → banco → response).

**Características**: Requisições HTTP reais, banco de teste ou test container, 500 ms–2 s por teste.

**Quando usar**: Contratos de API, autenticação/autorização, validação de input no endpoint, códigos HTTP e mensagens.

**Quando NÃO usar**: Lógica de validação isolada (unitários); fluxo completo na UI (E2E).

## Como Usar o Relatório

1. Receba o relatório: `@TEST_SERVICE_GUIDE.md @output/test-strategy-xxx.md`
2. Localize "Testes de Serviço" e casos API-XXX
3. Para cada caso: montar request (method, headers, body), executar, assert status e body (e estado no banco quando aplicável)
4. Agrupe por dimensão VADER/FAILURE (Authorization, Values, Data Consistency, Error Handling)

## Ferramentas

**TypeScript**: Supertest + Express/Koa, Jest.  
**Java**: RestAssured, MockMvc (Spring), JUnit 5, Testcontainers.

## Exemplo: POST /api/v1/transactions (VADER + FAILURE)

```typescript
import request from 'supertest';
import { app } from '../../src/app';
import { createTestDatabase, cleanDatabase, closeDatabase } from '../helpers/testDatabase';
import { createTestUser, generateValidToken } from '../helpers/testUser';

describe('POST /api/v1/transactions - API (VADER + FAILURE)', () => {
  let sender, recipient, validToken;

  beforeAll(async () => { await createTestDatabase(); });
  beforeEach(async () => {
    await cleanDatabase();
    sender = await createTestUser({ balance: 1000 });
    recipient = await createTestUser({ balance: 500 });
    validToken = generateValidToken(sender.id);
  });
  afterAll(async () => { await closeDatabase(); });

  describe('VADER - Authorization', () => {
    it('deve rejeitar requisição sem token (401)', async () => {
      const res = await request(app)
        .post('/api/v1/transactions')
        .send({ recipient_id: recipient.id, amount: 100 });
      expect(res.status).toBe(401);
      expect(res.body.error).toContain('não autorizado');
    });
    it('deve rejeitar token inválido (401)', async () => {
      const res = await request(app)
        .post('/api/v1/transactions')
        .set('Authorization', 'Bearer invalid_token')
        .send({ recipient_id: recipient.id, amount: 100 });
      expect(res.status).toBe(401);
    });
  });

  describe('VADER - Values (Validação)', () => {
    it('deve rejeitar amount = 0 (422)', async () => {
      const res = await request(app)
        .post('/api/v1/transactions')
        .set('Authorization', `Bearer ${validToken}`)
        .send({ recipient_id: recipient.id, amount: 0 });
      expect(res.status).toBe(422);
      expect(res.body.error).toContain('maior que 0');
    });
    it('deve rejeitar recipient_id vazio (422)', async () => {
      const res = await request(app)
        .post('/api/v1/transactions')
        .set('Authorization', `Bearer ${validToken}`)
        .send({ recipient_id: '', amount: 100 });
      expect(res.status).toBe(422);
      expect(res.body.error).toContain('obrigatório');
    });
  });

  describe('VADER - Data Consistency', () => {
    it('deve transferir valor com sucesso (201)', async () => {
      const res = await request(app)
        .post('/api/v1/transactions')
        .set('Authorization', `Bearer ${validToken}`)
        .send({ recipient_id: recipient.id, amount: 300 });
      expect(res.status).toBe(201);
      expect(res.body).toHaveProperty('transaction_id');
      expect(res.body.status).toBe('completed');
      expect(res.body.amount).toBe(300);
      const updatedSender = await getUserBalance(sender.id);
      const updatedRecipient = await getUserBalance(recipient.id);
      expect(updatedSender).toBe(700);
      expect(updatedRecipient).toBe(800);
    });
  });

  describe('FAILURE - Error Handling', () => {
    it('deve retornar 402 quando saldo insuficiente', async () => {
      const res = await request(app)
        .post('/api/v1/transactions')
        .set('Authorization', `Bearer ${validToken}`)
        .send({ recipient_id: recipient.id, amount: 5000 });
      expect(res.status).toBe(402);
      expect(res.body.error).toBe('Saldo insuficiente');
      expect(await getUserBalance(sender.id)).toBe(1000);
    });
    it('deve retornar 404 quando destinatário não existe', async () => {
      const res = await request(app)
        .post('/api/v1/transactions')
        .set('Authorization', `Bearer ${validToken}`)
        .send({ recipient_id: 'nonexistent_user', amount: 100 });
      expect(res.status).toBe(404);
      expect(res.body.error).toContain('não encontrado');
    });
  });
});
```

## Integração com Heurísticas

- **VADER - Authorization**: 401 sem token, 401 token inválido/expirado, 403 sem permissão
- **VADER - Values**: 400/422 para body inválido (tipos, obrigatórios, boundaries)
- **VADER - Data Consistency**: 201 + verificação de estado no banco; idempotency-key quando aplicável
- **VADER - Error Handling**: 402, 404, 422, 500 com mensagens claras
- **FAILURE - Appropriate**: Códigos HTTP e mensagens corretos
- **FAILURE - Functional**: Rollback (nenhum efeito colateral em 4xx/5xx)

## Para QAs

Template de caso de API no relatório: ID, endpoint, método, headers, body, status esperado, body esperado, heurística.

## Checklist

- [ ] Authorization (401/403) testada?
- [ ] Validação de input (422) para casos do relatório?
- [ ] Sucesso (201/200) e consistência de dados verificada?
- [ ] Erros de negócio (402, 404) com mensagens claras?
- [ ] Rollback ou nenhuma alteração em caso de erro?
- [ ] Cada API-XXX do relatório implementado?

## Referências

- Estratégia: [TEST_STRATEGY.md](TEST_STRATEGY.md)
- Unitários: [TEST_UNIT_GUIDE.md](TEST_UNIT_GUIDE.md)
- Integração: [TEST_INTEGRATION_GUIDE.md](TEST_INTEGRATION_GUIDE.md)
- VADER: [../../Cap08_desenvolvimento/VADER.md](../../Cap08_desenvolvimento/VADER.md)
- FAILURE: [../../Cap08_desenvolvimento/FAILURE.md](../../Cap08_desenvolvimento/FAILURE.md)
