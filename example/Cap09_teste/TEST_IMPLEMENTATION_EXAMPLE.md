# Exemplo Prático: Implementação de Testes para QualiPoints Transfer API

Este exemplo demonstra a aplicação das skills de teste no requisito de transferência de QualiPoints, usando a estrutura modular em [skills/utils/testes/](../../skills/utils/testes/).

## Workflow Recomendado

1. **Chamar [TEST_STRATEGY.md](../../skills/utils/testes/TEST_STRATEGY.md)** com o requisito e/ou código (ex.: "Função validateAmount e REQ-002 Transferir QualiPoints").
2. **Obter o relatório** gerado em `output/test-strategy-[nome]-[data].md` com heurísticas identificadas, casos de teste por camada e próximos passos.
3. **Implementar os testes** passando o relatório como contexto para os guides: [TEST_UNIT_GUIDE.md](../../skills/utils/testes/TEST_UNIT_GUIDE.md), [TEST_INTEGRATION_GUIDE.md](../../skills/utils/testes/TEST_INTEGRATION_GUIDE.md), [TEST_SERVICE_GUIDE.md](../../skills/utils/testes/TEST_SERVICE_GUIDE.md), [TEST_E2E_GUIDE.md](../../skills/utils/testes/TEST_E2E_GUIDE.md).

## Contexto

**Requisito Original**: REQ-002 Transferir QualiPoints para Outro Usuário

**Funcionalidade**: Permitir que usuários transfiram seus QualiPoints para outros usuários autenticados, com validação de saldo, destinatário válido e registro de transação.

## Análise Inicial com Heurísticas

### Heurísticas Aplicadas

- **Baica**: Validação de inputs (recipient_id, amount)
- **VADER**: Validação de API, autorização, consistência de dados
- **FAILURE**: Tratamento de erros, rollback, mensagens
- **CRUD**: Operações Create (transação), Read (saldo), Update (saldos)
- **Count**: Cenários com 0 QualiPoints, 1 transação, múltiplas transações
- **Chique**: Estado de botões, campos obrigatórios
- **SeenAndHeard**: Feedback de loading, sucesso, erro
- **InputMethod**: Typing no formulário, API payload

## Estratégia de Testes por Camada

### Distribuição de Testes (Pirâmide)

```
Testes E2E (2 testes) - 2%
  └─ Fluxo completo de transferência
  └─ Recuperação de erro

Testes de Serviço (6 testes) - 6%
  └─ POST /api/v1/transactions (sucesso, erros)

Testes de Integração (15 testes) - 15%
  └─ TransactionRepository CRUD
  └─ UserRepository saldo

Testes Unitários (77 testes) - 77%
  └─ Validações (40)
  └─ TransferService (25)
  └─ Utilitários (12)

Total: 100 testes
```

## Implementação dos Testes

### 1. Testes Unitários

#### 1.1 Validação de Amount (Baica - Boundaries)

**TypeScript:**

```typescript
// tests/unit/validators/validateAmount.test.ts
import { validateAmount } from '../../../src/validators/validateAmount';

describe('validateAmount - Baica Boundaries', () => {
  describe('valores mínimos', () => {
    it('deve rejeitar amount = 0', () => {
      const result = validateAmount(0);
      expect(result.valid).toBe(false);
      expect(result.error).toContain('maior que 0');
    });

    it('deve rejeitar amount negativo', () => {
      const result = validateAmount(-100);
      expect(result.valid).toBe(false);
      expect(result.error).toContain('maior que 0');
    });

    it('deve aceitar amount = 1 (mínimo válido)', () => {
      const result = validateAmount(1);
      expect(result.valid).toBe(true);
      expect(result.error).toBeUndefined();
    });
  });

  describe('valores máximos', () => {
    it('deve aceitar 1.000.000 (máximo)', () => {
      const result = validateAmount(1_000_000);
      expect(result.valid).toBe(true);
    });

    it('deve rejeitar amount > 1.000.000', () => {
      const result = validateAmount(1_000_001);
      expect(result.valid).toBe(false);
      expect(result.error).toContain('não pode exceder');
    });
  });

  describe('valores nulos (Baica - Nulls)', () => {
    it('deve rejeitar null', () => {
      const result = validateAmount(null);
      expect(result.valid).toBe(false);
      expect(result.error).toContain('obrigatório');
    });

    it('deve rejeitar undefined', () => {
      const result = validateAmount(undefined);
      expect(result.valid).toBe(false);
    });
  });
});

// Resultado: 8 testes unitários
```

#### 1.2 Validação de Recipient ID (Baica - Special Chars & Formats)

**TypeScript:**

```typescript
// tests/unit/validators/validateRecipientId.test.ts
import { validateRecipientId } from '../../../src/validators/validateRecipientId';

describe('validateRecipientId - Baica Special Chars & Formats', () => {
  describe('valores nulos/vazios', () => {
    it('deve rejeitar null', () => {
      const result = validateRecipientId(null);
      expect(result.valid).toBe(false);
    });

    it('deve rejeitar string vazia', () => {
      const result = validateRecipientId('');
      expect(result.valid).toBe(false);
    });

    it('deve rejeitar apenas espaços', () => {
      const result = validateRecipientId('   ');
      expect(result.valid).toBe(false);
    });
  });

  describe('tamanho (Baica - Boundaries)', () => {
    it('deve aceitar ID com 1 caractere', () => {
      const result = validateRecipientId('a');
      expect(result.valid).toBe(true);
    });

    it('deve aceitar ID com 128 caracteres (máximo)', () => {
      const result = validateRecipientId('a'.repeat(128));
      expect(result.valid).toBe(true);
    });

    it('deve rejeitar ID com 129 caracteres', () => {
      const result = validateRecipientId('a'.repeat(129));
      expect(result.valid).toBe(false);
      expect(result.error).toContain('não pode exceder 128');
    });
  });

  describe('caracteres especiais (Baica - Special Chars)', () => {
    it('deve aceitar letras, números, underscore e hífen', () => {
      const validIds = ['user123', 'user_123', 'user-123', 'user_123-abc'];
      validIds.forEach(id => {
        expect(validateRecipientId(id).valid).toBe(true);
      });
    });

    it('deve rejeitar espaços e caracteres especiais', () => {
      const invalidIds = ['user 123', 'user@123', 'user#123', 'user😀123'];
      invalidIds.forEach(id => {
        expect(validateRecipientId(id).valid).toBe(false);
      });
    });
  });
});

// Resultado: 10 testes unitários
```

#### 1.3 TransferService com Mocks

**TypeScript:**

```typescript
// tests/unit/services/TransferService.test.ts
import { TransferService } from '../../../src/services/TransferService';
import { BalanceRepository } from '../../../src/repositories/BalanceRepository';

describe('TransferService', () => {
  let mockBalanceRepo: jest.Mocked<BalanceRepository>;
  let transferService: TransferService;
  
  beforeEach(() => {
    mockBalanceRepo = {
      getBalance: jest.fn(),
      updateBalance: jest.fn(),
    } as any;
    
    transferService = new TransferService(mockBalanceRepo);
  });
  
  describe('canTransfer', () => {
    it('deve retornar true quando saldo é suficiente', async () => {
      mockBalanceRepo.getBalance.mockResolvedValue(100);
      
      const result = await transferService.canTransfer('user123', 50);
      
      expect(result).toBe(true);
      expect(mockBalanceRepo.getBalance).toHaveBeenCalledWith('user123');
    });
    
    it('deve retornar false quando saldo é insuficiente', async () => {
      mockBalanceRepo.getBalance.mockResolvedValue(30);
      
      const result = await transferService.canTransfer('user123', 50);
      
      expect(result).toBe(false);
    });
    
    it('deve retornar true quando saldo é exatamente igual (Baica - Boundaries)', async () => {
      mockBalanceRepo.getBalance.mockResolvedValue(50);
      
      const result = await transferService.canTransfer('user123', 50);
      
      expect(result).toBe(true);
    });
    
    it('deve retornar false quando saldo é 1 a menos (Baica - Boundaries)', async () => {
      mockBalanceRepo.getBalance.mockResolvedValue(49);
      
      const result = await transferService.canTransfer('user123', 50);
      
      expect(result).toBe(false);
    });
  });
});

// Resultado: 4 testes unitários
```

**Total de testes unitários neste exemplo: 22 (na prática seriam ~77)**

---

### 2. Testes de Integração

#### 2.1 UserRepository - CRUD com PostgreSQL

**TypeScript:**

```typescript
// tests/integration/repositories/UserRepository.test.ts
import { Pool } from 'pg';
import { UserRepository } from '../../../src/repositories/UserRepository';

describe('UserRepository - Integration (CRUD)', () => {
  let pool: Pool;
  let repository: UserRepository;
  
  beforeAll(async () => {
    pool = new Pool({
      host: 'localhost',
      port: 5432,
      database: 'test_db',
      user: 'test_user',
      password: 'test_pass'
    });
    
    await pool.query(`
      CREATE TABLE IF NOT EXISTS users (
        id UUID PRIMARY KEY,
        name VARCHAR(255) NOT NULL,
        email VARCHAR(255) NOT NULL UNIQUE,
        balance INTEGER NOT NULL DEFAULT 0,
        created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
      )
    `);
    
    repository = new UserRepository(pool);
  });
  
  beforeEach(async () => {
    await pool.query('DELETE FROM users');
  });
  
  afterAll(async () => {
    await pool.query('DROP TABLE IF EXISTS users');
    await pool.end();
  });
  
  describe('Create (CRUD)', () => {
    it('deve criar usuário com dados válidos', async () => {
      const user = await repository.create({
        name: 'João Silva',
        email: 'joao@example.com',
        balance: 1000
      });
      
      expect(user.id).toBeDefined();
      expect(user.name).toBe('João Silva');
      expect(user.balance).toBe(1000);
      
      // Verificar persistência
      const found = await repository.findById(user.id);
      expect(found).toEqual(user);
    });
    
    it('deve rejeitar email duplicado', async () => {
      await repository.create({
        name: 'User 1',
        email: 'duplicate@example.com',
        balance: 100
      });
      
      await expect(
        repository.create({
          name: 'User 2',
          email: 'duplicate@example.com',
          balance: 200
        })
      ).rejects.toThrow(); // Unique constraint violation
    });
  });
  
  describe('Read (CRUD)', () => {
    it('deve retornar null quando usuário não existe (Count - Zero)', async () => {
      const user = await repository.findById('00000000-0000-0000-0000-000000000000');
      expect(user).toBeNull();
    });
    
    it('deve encontrar usuário por ID (Count - Um)', async () => {
      const created = await repository.create({
        name: 'Maria',
        email: 'maria@example.com',
        balance: 500
      });
      
      const found = await repository.findById(created.id);
      expect(found).toEqual(created);
    });
  });
  
  describe('Update (CRUD)', () => {
    it('deve atualizar saldo do usuário', async () => {
      const user = await repository.create({
        name: 'Pedro',
        email: 'pedro@example.com',
        balance: 1000
      });
      
      const updated = await repository.updateBalance(user.id, 1500);
      
      expect(updated.balance).toBe(1500);
      
      // Verificar persistência
      const found = await repository.findById(user.id);
      expect(found?.balance).toBe(1500);
    });
  });
  
  describe('Delete (CRUD)', () => {
    it('deve deletar usuário existente', async () => {
      const user = await repository.create({
        name: 'Ana',
        email: 'ana@example.com',
        balance: 750
      });
      
      await repository.delete(user.id);
      
      const found = await repository.findById(user.id);
      expect(found).toBeNull();
    });
    
    it('deve ser idempotente ao deletar inexistente', async () => {
      await expect(
        repository.delete('00000000-0000-0000-0000-000000000000')
      ).resolves.not.toThrow();
    });
  });
});

// Resultado: 8 testes de integração
```

**Total de testes de integração neste exemplo: 8 (na prática seriam ~15)**

---

### 3. Testes de Serviço (API)

#### 3.1 POST /api/v1/transactions

**TypeScript:**

```typescript
// tests/api/transfer.api.test.ts
import request from 'supertest';
import { app } from '../../src/app';
import { createTestDatabase, cleanDatabase } from '../helpers/testDatabase';
import { createTestUser, generateValidToken } from '../helpers/testUser';

describe('POST /api/v1/transactions - API (VADER + FAILURE)', () => {
  let sender, recipient, validToken;
  
  beforeAll(async () => {
    await createTestDatabase();
  });
  
  beforeEach(async () => {
    await cleanDatabase();
    
    sender = await createTestUser({ balance: 1000 });
    recipient = await createTestUser({ balance: 500 });
    validToken = generateValidToken(sender.id);
  });
  
  describe('VADER - Authorization', () => {
    it('deve rejeitar requisição sem token (401)', async () => {
      const response = await request(app)
        .post('/api/v1/transactions')
        .send({ recipient_id: recipient.id, amount: 100 });
      
      expect(response.status).toBe(401);
    });
    
    it('deve aceitar token válido (200/201)', async () => {
      const response = await request(app)
        .post('/api/v1/transactions')
        .set('Authorization', `Bearer ${validToken}`)
        .send({ recipient_id: recipient.id, amount: 100 });
      
      expect(response.status).toBe(201);
    });
  });
  
  describe('VADER - Values (Validação)', () => {
    it('deve rejeitar amount = 0 (422)', async () => {
      const response = await request(app)
        .post('/api/v1/transactions')
        .set('Authorization', `Bearer ${validToken}`)
        .send({ recipient_id: recipient.id, amount: 0 });
      
      expect(response.status).toBe(422);
      expect(response.body.error).toContain('maior que 0');
    });
    
    it('deve rejeitar recipient_id vazio (422)', async () => {
      const response = await request(app)
        .post('/api/v1/transactions')
        .set('Authorization', `Bearer ${validToken}`)
        .send({ recipient_id: '', amount: 100 });
      
      expect(response.status).toBe(422);
    });
  });
  
  describe('VADER - Data Consistency', () => {
    it('deve transferir e atualizar saldos corretamente (201)', async () => {
      const response = await request(app)
        .post('/api/v1/transactions')
        .set('Authorization', `Bearer ${validToken}`)
        .send({ recipient_id: recipient.id, amount: 300 });
      
      expect(response.status).toBe(201);
      expect(response.body.transaction_id).toBeDefined();
      expect(response.body.status).toBe('completed');
      
      // Verificar saldos atualizados
      const updatedSender = await getUserBalance(sender.id);
      const updatedRecipient = await getUserBalance(recipient.id);
      
      expect(updatedSender).toBe(700); // 1000 - 300
      expect(updatedRecipient).toBe(800); // 500 + 300
    });
  });
  
  describe('FAILURE - Error Handling', () => {
    it('deve retornar 402 quando saldo insuficiente', async () => {
      const response = await request(app)
        .post('/api/v1/transactions')
        .set('Authorization', `Bearer ${validToken}`)
        .send({ recipient_id: recipient.id, amount: 5000 });
      
      expect(response.status).toBe(402);
      expect(response.body.error).toBe('Saldo insuficiente');
      
      // Verificar que rollback aconteceu
      const unchangedBalance = await getUserBalance(sender.id);
      expect(unchangedBalance).toBe(1000);
    });
    
    it('deve retornar 404 quando destinatário não existe', async () => {
      const response = await request(app)
        .post('/api/v1/transactions')
        .set('Authorization', `Bearer ${validToken}`)
        .send({ recipient_id: 'nonexistent', amount: 100 });
      
      expect(response.status).toBe(404);
      expect(response.body.error).toContain('não encontrado');
    });
  });
});

// Resultado: 8 testes de serviço/API
```

**Total de testes de serviço neste exemplo: 8 (na prática seriam ~6)**

---

### 4. Testes E2E

#### 4.1 Fluxo Completo de Transferência

**TypeScript (Playwright):**

```typescript
// tests/e2e/transfer-flow.e2e.ts
import { test, expect } from '@playwright/test';

test.describe('Fluxo de Transferência E2E', () => {
  
  test.beforeEach(async ({ page }) => {
    await page.goto('http://localhost:3000/login');
    await page.fill('input[name="email"]', 'sender@test.com');
    await page.fill('input[name="password"]', 'password123');
    await page.click('button[type="submit"]');
    await page.waitForURL('**/dashboard');
  });
  
  test('deve completar transferência com sucesso', async ({ page }) => {
    // 1. Navegar para transferência
    await page.click('a[href="/transfer"]');
    
    // 2. Preencher formulário (InputMethod - Typing)
    await page.fill('input[name="recipient_id"]', 'recipient@test.com');
    await page.fill('input[name="amount"]', '500');
    
    // 3. Chique - Verificar que botão está habilitado
    const submitButton = page.locator('button[type="submit"]');
    await expect(submitButton).toBeEnabled();
    
    // 4. Submeter
    await submitButton.click();
    
    // 5. SeenAndHeard - Verificar loading
    await expect(page.locator('[role="status"]')).toContainText(/Processando/i);
    
    // 6. FAILURE - Emotions: Verificar sucesso
    await expect(page.locator('[role="alert"]')).toContainText(/sucesso/i, { timeout: 10000 });
    
    // 7. Verificar transação no histórico
    await expect(page).toHaveURL(/.*transactions/);
    const transactionRow = page.locator('table tr').first();
    await expect(transactionRow).toContainText('recipient@test.com');
    await expect(transactionRow).toContainText('500');
  });
  
  test('deve permitir recuperação após erro', async ({ page }) => {
    await page.goto('http://localhost:3000/transfer');
    
    // Preencher com amount inválido
    await page.fill('input[name="recipient_id"]', 'recipient@test.com');
    await page.fill('input[name="amount"]', '0');
    await page.click('button[type="submit"]');
    
    // FAILURE - Appropriate: Mensagem de erro
    await expect(page.locator('[role="alert"]')).toContainText(/maior que 0/i);
    
    // FAILURE - Recovery: Corrigir e resubmeter
    await page.fill('input[name="amount"]', '500');
    await page.click('button[type="submit"]');
    
    // Deve ter sucesso
    await expect(page.locator('[role="alert"]')).toContainText(/sucesso/i);
  });
});

// Resultado: 2 testes E2E
```

**Total de testes E2E neste exemplo: 2**

---

## Resumo da Cobertura

### Distribuição Final

| Camada | Testes Exemplo | % |
|--------|----------------|---|
| Unitários | 22 | 56% |
| Integração | 8 | 21% |
| Serviço | 8 | 21% |
| E2E | 2 | 5% |
| **Total** | **40** | **100%** |

### Heurísticas Cobertas

- ✅ **Baica**: 18 testes (boundaries, nulls, special chars)
- ✅ **VADER**: 8 testes (authorization, values, data consistency, error handling)
- ✅ **FAILURE**: 6 testes (functional, appropriate, recovery, emotions)
- ✅ **CRUD**: 5 testes (create, read, update, delete)
- ✅ **Count**: 3 testes (0, 1, muitos)
- ✅ **Chique**: 1 teste (habilitar/desabilitar)
- ✅ **SeenAndHeard**: 2 testes (loading, feedback de sucesso/erro)
- ✅ **InputMethod**: 2 testes (typing)

### Benefícios Alcançados

1. **Confiança para Refatorar**: Testes abrangentes permitem mudanças seguras
2. **Documentação Viva**: Testes documentam comportamento esperado
3. **Prevenção de Regressões**: Bugs não retornam após correção
4. **Feedback Rápido**: Testes unitários executam em <1s
5. **Cobertura Holística**: Todas as heurísticas aplicadas em código executável

## Próximos Passos

Para expandir esta suíte de testes:

1. Adicionar testes de componentes (React Testing Library) para o formulário de transferência
2. Adicionar testes de performance (Count - Muitos) com 10.000+ transações
3. Adicionar testes de concorrência (VADER - Data Consistency) com múltiplas transferências simultâneas
4. Adicionar testes de acessibilidade (SeenAndHeard) com jest-axe
5. Adicionar testes de rate limiting (VADER) na API

---

**Consulte [TEST_STRATEGY.md](../../skills/utils/testes/TEST_STRATEGY.md) para gerar o relatório e os guides em [skills/utils/testes/](../../skills/utils/testes/) para implementação de cada camada.**
