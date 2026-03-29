---
name: test-integration-guide
description: Guia para implementar testes de integração. Use quando tiver relatório de estratégia (output/test-strategy-*.md) com casos IT-XXX e precisar gerar código que testa repositórios, banco de dados, transações e rollbacks em TypeScript ou Java.
---

# Guia de Testes de Integração

Implemente testes de integração com base no **relatório de estratégia** gerado por [TEST_STRATEGY.md](TEST_STRATEGY.md). Use os casos com prefixo **IT-XXX** para gerar testes que validam interação entre módulos (ex.: repositório + banco de dados).

## Quando Usar Este Guide

- Relatório em `output/` contém casos de teste de integração (IT-XXX)
- Precisa testar: persistência (CRUD), transações, rollback, queries
- Heurísticas: CRUD, VADER (Data Consistency), FAILURE (Functional), Count

## Definição e Escopo

Testes de integração validam a **interação entre dois ou mais módulos** com dependências reais (ex.: banco de dados).

**Características**: Banco real ou test container, 100ms–1s por teste, setup/teardown para estado limpo.

**Quando usar**: Repositórios, CRUD, transações, rollbacks, queries.

**Quando NÃO usar**: Lógica pura (use [TEST_UNIT_GUIDE.md](TEST_UNIT_GUIDE.md)); APIs HTTP completas (use [TEST_SERVICE_GUIDE.md](TEST_SERVICE_GUIDE.md)).

## Como Usar o Relatório

1. Receba o relatório: `@TEST_INTEGRATION_GUIDE.md @output/test-strategy-xxx.md`
2. Localize "Testes de Integração" e casos IT-XXX
3. Para cada caso: setup (conectar/limpar banco), action (chamar repositório/serviço), assert (dados persistidos ou rollback)
4. Mantenha IDs (IT-USER-01, IT-TRANS-01) em comentários

## Estratégias de Banco de Dados

- **In-memory**: SQLite, H2 — rápido; pode divergir de produção
- **Testcontainers**: PostgreSQL/MySQL em Docker — igual a produção; requer Docker
- **Banco de teste**: instância dedicada; limpeza rigorosa entre testes

## Setup e Teardown

**TypeScript (ex.: pg):**

```typescript
beforeAll(async () => {
  pool = new Pool({ /* config test_db */ });
  await pool.query('CREATE TABLE IF NOT EXISTS users (...)');
  repository = new UserRepository(pool);
});
beforeEach(async () => {
  await pool.query('DELETE FROM users');
});
afterAll(async () => {
  await pool.query('DROP TABLE IF EXISTS users');
  await pool.end();
});
```

**Java (Spring Boot Test):**

```java
@SpringBootTest
@ActiveProfiles("test")
@Transactional
class UserRepositoryIT {
  @Autowired
  private UserRepository userRepo;
  @BeforeEach
  void setUp() { userRepo.deleteAll(); }
}
```

## Exemplo 1: CRUD com Repositório (TypeScript)

Casos típicos no relatório: IT-USER-01 (Create), IT-USER-02 (Read), IT-USER-03 (Update), IT-USER-04 (Delete), IT-USER-05 (idempotência delete).

```typescript
describe('UserRepository - Integration (CRUD)', () => {
  let pool: Pool;
  let repository: UserRepository;

  beforeAll(async () => {
    pool = new Pool({ host: 'localhost', database: 'test_db', ... });
    await pool.query(`CREATE TABLE IF NOT EXISTS users (
      id UUID PRIMARY KEY, name VARCHAR(255), email VARCHAR(255) UNIQUE, balance INTEGER, created_at TIMESTAMP
    )`);
    repository = new UserRepository(pool);
  });
  beforeEach(async () => { await pool.query('DELETE FROM users'); });
  afterAll(async () => { await pool.query('DROP TABLE IF EXISTS users'); await pool.end(); });

  describe('Create (CRUD)', () => {
    it('deve criar usuário com dados válidos', async () => {
      const user = await repository.create({ name: 'João', email: 'joao@example.com', balance: 1000 });
      expect(user.id).toBeDefined();
      expect(user.name).toBe('João');
      const found = await repository.findById(user.id);
      expect(found).toEqual(user);
    });
    it('deve rejeitar criação com email duplicado', async () => {
      await repository.create({ name: 'A', email: 'dup@example.com', balance: 100 });
      await expect(repository.create({ name: 'B', email: 'dup@example.com', balance: 200 })).rejects.toThrow();
    });
  });

  describe('Read (CRUD)', () => {
    it('deve retornar null quando usuário não existe', async () => {
      const user = await repository.findById('00000000-0000-0000-0000-000000000000');
      expect(user).toBeNull();
    });
    it('deve encontrar usuário por ID', async () => {
      const created = await repository.create({ name: 'Maria', email: 'maria@example.com', balance: 500 });
      const found = await repository.findById(created.id);
      expect(found).toEqual(created);
    });
  });

  describe('Update (CRUD)', () => {
    it('deve atualizar saldo do usuário', async () => {
      const user = await repository.create({ name: 'Pedro', email: 'pedro@example.com', balance: 1000 });
      const updated = await repository.updateBalance(user.id, 1500);
      expect(updated.balance).toBe(1500);
      const found = await repository.findById(user.id);
      expect(found?.balance).toBe(1500);
    });
  });

  describe('Delete (CRUD)', () => {
    it('deve deletar usuário existente', async () => {
      const user = await repository.create({ name: 'Ana', email: 'ana@example.com', balance: 750 });
      await repository.delete(user.id);
      expect(await repository.findById(user.id)).toBeNull();
    });
    it('não deve falhar ao deletar usuário inexistente (idempotência)', async () => {
      await expect(repository.delete('00000000-0000-0000-0000-000000000000')).resolves.not.toThrow();
    });
  });
});
```

## Exemplo 2: Transação com Rollback (Java)

Caso típico: IT-TRANS-01 — rollback quando saldo insuficiente.

```java
@SpringBootTest
@ActiveProfiles("test")
@DisplayName("TransferTransactionService - Integration")
class TransferTransactionServiceIT {
    @Autowired
    private UserRepository userRepo;
    @Autowired
    private EntityManager entityManager;
    private TransferTransactionService transferService;

    @BeforeEach
    void setUp() {
        userRepo.deleteAll();
        transferService = new TransferTransactionService(userRepo, entityManager);
    }

    @Test
    @Transactional
    @DisplayName("Deve transferir valor entre usuários com sucesso")
    void shouldTransferBetweenUsers() {
        User sender = userRepo.save(new User("s@test.com", "Sender", 1000));
        User recipient = userRepo.save(new User("r@test.com", "Recipient", 500));
        entityManager.flush();
        transferService.transfer(sender.getId(), recipient.getId(), 300);
        entityManager.flush();
        entityManager.clear();
        assertEquals(700, userRepo.findById(sender.getId()).orElseThrow().getBalance());
        assertEquals(800, userRepo.findById(recipient.getId()).orElseThrow().getBalance());
    }

    @Test
    @Transactional
    @DisplayName("Deve fazer rollback quando saldo insuficiente (FAILURE - Functional)")
    void shouldRollbackWhenInsufficientBalance() {
        User sender = userRepo.save(new User("s@test.com", "Sender", 100));
        User recipient = userRepo.save(new User("r@test.com", "Recipient", 500));
        entityManager.flush();
        assertThrows(IllegalStateException.class, () ->
            transferService.transfer(sender.getId(), recipient.getId(), 300));
        entityManager.clear();
        assertEquals(100, userRepo.findById(sender.getId()).orElseThrow().getBalance());
        assertEquals(500, userRepo.findById(recipient.getId()).orElseThrow().getBalance());
    }
}
```

## Integração com Heurísticas

- **CRUD**: Create (sucesso + constraint), Read (encontrado + não encontrado), Update, Delete (sucesso + idempotência)
- **VADER - Data Consistency**: Transação atômica, rollback em falha, integridade referencial
- **FAILURE - Functional**: Rollback verificado; estado consistente após erro
- **Count**: Lista vazia (0), um registro (1), paginação (muitos)

## Para QAs

Use o relatório para casos de teste de integração: ID, componentes testados, pré-condições, passos, resultado esperado, heurística.

## Checklist

- [ ] Setup/teardown limpa dados antes de cada teste?
- [ ] Testes independentes (não dependem de ordem)?
- [ ] Transações e rollbacks testados quando aplicável?
- [ ] Constraints (unique, not null, FK) verificadas?
- [ ] CRUD coberto conforme relatório?
- [ ] Cada IT-XXX do relatório implementado?
- [ ] Performance: testes < 1s cada?

## Referências

- Estratégia: [TEST_STRATEGY.md](TEST_STRATEGY.md)
- Unitários: [TEST_UNIT_GUIDE.md](TEST_UNIT_GUIDE.md)
- Serviço: [TEST_SERVICE_GUIDE.md](TEST_SERVICE_GUIDE.md)
- Heurística CRUD: [CRUD.md](../../skills/heuristic-guide-architeture-code/CRUD.md)
