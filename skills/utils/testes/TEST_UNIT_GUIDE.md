---
name: test-unit-guide
description: Guia para implementar testes unitários. Use quando tiver um relatório de estratégia (output/test-strategy-*.md) e precisar gerar código de testes unitários em TypeScript ou Java. Leia os casos do relatório e implemente cada um.
---

# Guia de Testes Unitários

Implemente testes unitários com base no **relatório de estratégia** gerado por [TEST_STRATEGY.md](TEST_STRATEGY.md). Quando receber um relatório em `output/test-strategy-*.md`, use os casos listados para gerar o código exato dos testes.

## Quando Usar Este Guide

- Você já tem um relatório em `output/` com casos de teste unitários (prefixo UT-XXX)
- Precisa implementar testes para funções de validação, serviços com mocks, ou lógica pura
- Heurísticas aplicadas: Baica, VADER (Values), Count, FAILURE (mensagens de erro)

## Definição e Escopo

Testes unitários validam a menor unidade testável **isoladamente**, sem dependências reais (banco, rede, APIs).

**Características**: Isolamento total, rápidos (< 50ms), determinísticos, independentes.

**Quando usar**: Validações, cálculos, transformações, funções que recebem input (Baica).

**Quando NÃO usar**: Persistência (use [TEST_INTEGRATION_GUIDE.md](TEST_INTEGRATION_GUIDE.md)), APIs HTTP (use [TEST_SERVICE_GUIDE.md](TEST_SERVICE_GUIDE.md)).

## Como Usar o Relatório

1. **Receba o relatório** como contexto: `@TEST_UNIT_GUIDE.md @output/test-strategy-validateAmount-20260217.md`
2. **Localize a seção** "Testes Unitários" e os casos com prefixo UT-
3. **Para cada caso** no relatório, implemente um `it()` ou `@Test` com:
   - Nome descritivo (ex: "deve rejeitar amount = 0")
   - Arrange: input conforme relatório
   - Act: chamar função sob teste
   - Assert: output esperado conforme relatório
4. **Mantenha os IDs** dos casos (UT-VAL-01, etc.) em comentário para rastreabilidade

## Padrão AAA e Frameworks

**Estrutura**: Arrange (preparar) → Act (executar) → Assert (verificar).

**TypeScript**: Jest ou Vitest. `describe` / `it`, `expect()`.

**Java**: JUnit 5, Mockito. `@Test`, `@DisplayName`, `assertEquals` / `assertTrue` / `assertFalse`.

## Exemplo 1: Validação de Amount (Baica - Boundaries)

Relatório pode listar: UT-VAL-01 (amount=0), UT-VAL-02 (amount=-1), UT-VAL-03 (amount=1), UT-VAL-04 (amount=1_000_000), UT-VAL-05 (amount=1_000_001), UT-VAL-06 (null), UT-VAL-07 (undefined).

**TypeScript (Jest):**

```typescript
import { validateAmount } from '../../src/validators/validateAmount';

describe('validateAmount - Baica Boundaries', () => {
  describe('valores mínimos', () => {
    it('deve rejeitar amount = 0', () => {
      const result = validateAmount(0);
      expect(result.valid).toBe(false);
      expect(result.error).toContain('maior que 0');
    });
    it('deve rejeitar amount negativo', () => {
      expect(validateAmount(-1).valid).toBe(false);
    });
    it('deve aceitar amount = 1 (limite mínimo válido)', () => {
      expect(validateAmount(1).valid).toBe(true);
    });
  });
  describe('valores máximos', () => {
    it('deve aceitar 1.000.000', () => {
      expect(validateAmount(1_000_000).valid).toBe(true);
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
      expect(validateAmount(undefined).valid).toBe(false);
    });
  });
});
```

**Java (JUnit 5):**

```java
@DisplayName("AmountValidator - Baica Boundaries")
class AmountValidatorTest {
    @Test
    @DisplayName("Deve rejeitar amount = 0")
    void shouldRejectZero() {
        var result = AmountValidator.validate(0);
        assertFalse(result.isValid());
        assertTrue(result.getError().contains("maior que 0"));
    }
    @Test
    @DisplayName("Deve aceitar amount = 1 (limite mínimo válido)")
    void shouldAcceptMinimumValid() {
        var result = AmountValidator.validate(1);
        assertTrue(result.isValid());
        assertNull(result.getError());
    }
    @Test
    @DisplayName("Deve rejeitar amount > 1.000.000")
    void shouldRejectAboveMaximum() {
        var result = AmountValidator.validate(1_000_001);
        assertFalse(result.isValid());
        assertTrue(result.getError().contains("exceder"));
    }
    @Test
    @DisplayName("Deve rejeitar null")
    void shouldRejectNull() {
        var result = AmountValidator.validate(null);
        assertFalse(result.isValid());
        assertTrue(result.getError().contains("obrigatório"));
    }
}
```

## Exemplo 2: Service com Mocks

Quando o relatório incluir testes de serviço com dependências, use mocks.

**TypeScript (Jest):**

```typescript
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

  it('deve retornar true quando saldo é exatamente igual ao amount', async () => {
    mockBalanceRepo.getBalance.mockResolvedValue(50);
    const result = await transferService.canTransfer('user123', 50);
    expect(result).toBe(true);
  });
});
```

**Java (JUnit 5 + Mockito):**

```java
@Mock
private BalanceRepository balanceRepo;
private TransferService transferService;

@BeforeEach
void setUp() {
    MockitoAnnotations.openMocks(this);
    transferService = new TransferService(balanceRepo);
}

@Test
@DisplayName("Deve retornar true quando saldo é suficiente")
void shouldReturnTrueWhenBalanceIsSufficient() {
    when(balanceRepo.getBalance("user123")).thenReturn(100);
    boolean result = transferService.canTransfer("user123", 50);
    assertTrue(result);
    verify(balanceRepo, times(1)).getBalance("user123");
}

@Test
@DisplayName("Deve retornar false quando saldo é insuficiente")
void shouldReturnFalseWhenBalanceIsInsufficient() {
    when(balanceRepo.getBalance("user123")).thenReturn(30);
    assertFalse(transferService.canTransfer("user123", 50));
}
```

## Integração com Heurísticas (Resumo)

- **Baica**: Boundaries (0, -1, min, max, max+1), Nulls (null, undefined, ""), Special Chars, Formats.
- **VADER**: Values (tipos, formatos), Authorization (funções de permissão).
- **Count**: 0 (lista vazia), 1 (um item), Muitos (lista grande, performance).
- **FAILURE**: Mensagens de erro claras, exceções apropriadas, sem corromper estado.

## Para QAs

Use o relatório para desenhar casos de teste manuais ou automatizados. Cada UT-XXX no relatório deve ter: ID, cenário, entrada, saída esperada, heurística, prioridade (P0/P1/P2).

## Checklist

- [ ] Relatório de estratégia foi usado como fonte dos casos?
- [ ] Cada caso do relatório virou um `it()` / `@Test`?
- [ ] Isolamento: dependências mockadas?
- [ ] Padrão AAA seguido?
- [ ] Nomes dos testes claros e alinhados ao relatório?
- [ ] Assertions específicas (não genéricas)?
- [ ] Testes independentes e rápidos (< 50ms)?

## Referências

- Estratégia e relatório: [TEST_STRATEGY.md](TEST_STRATEGY.md)
- Integração: [TEST_INTEGRATION_GUIDE.md](TEST_INTEGRATION_GUIDE.md)
- Serviço/API: [TEST_SERVICE_GUIDE.md](TEST_SERVICE_GUIDE.md)
- Heurística Baica: [../../Cap08_desenvolvimento/Baica.md](../../Cap08_desenvolvimento/Baica.md)
