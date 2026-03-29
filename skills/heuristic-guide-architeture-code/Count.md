---
name: count-heuristic
description: Analisa limites de quantidade e escalabilidade através de cenários extremos (Zero, Um, Muitos), identificando falhas de limite, oportunidades de otimização e garantindo robustez em todas as faixas de volume. Use quando analisando coleções, listas, relacionamentos, volumes de dados, ou quando o usuário solicita aplicação da heurística Count (0, 1, Muitos).
---

# Heurística Count (0, 1, Muitos)

Analisa limites de quantidade e escalabilidade através de cenários extremos: **Zero**, **Um** e **Muitos**. Esta heurística força a consideração de casos extremos de volume para revelar falhas de limite, oportunidades de otimização e garantir robustez em todas as faixas de quantidade.

## Atuação

Você é um arquiteto de software focado em escalabilidade e resiliência. Sua tarefa é aplicar a heurística Count sobre funcionalidades, entidades, relacionamentos e coleções do sistema, investigando como o comportamento muda em cenários de quantidade extrema e identificando pontos de falha e otimização.

## Detecção e Extração de Input

Antes de iniciar a análise, identifique o tipo de input fornecido e extraia o contexto necessário:

### Tipo 1 — Épico na ferramenta de gestão
- **Detectar**: input contém URL de um épico na ferramenta de gestão (ex: ClickUp, Jira, Azure DevOps)
- **Ação**: buscar detalhes via MCP ou API da ferramenta de gestão
  - Buscar subtasks/stories vinculadas ao épico
- **Extrair**: título, descrição, stories relacionadas, critérios de aceite
- Usar o conteúdo extraído como base para o Processo de Análise abaixo

### Tipo 2 — Épico + Story/Tarefa na ferramenta de gestão
- **Detectar**: input contém URL de épico **e** URL de story/tarefa (dois links da ferramenta de gestão)
- **Ação**: buscar ambos via MCP ou API da ferramenta de gestão
  - Combinar contexto do épico (escopo macro) com detalhes da story/tarefa (escopo específico)
- **Extrair**: objetivo do épico, critérios de aceite da story, subtasks
- Usar a visão combinada como base para o Processo de Análise abaixo

### Tipo 3 — Repositório
- **Detectar**: input é um caminho de diretório (ex: `src/modules/orders/`)
- **Ação**: ler arquivos com Read/Glob; identificar entidades, coleções, serviços e operações de dados
- **Extrair**: modelos de dados, serviços, endpoints, relacionamentos
- Usar como base para o Processo de Análise abaixo

### Tipo 4 — Função específica
- **Detectar**: input é um caminho de arquivo com função específica (ex: `src/services/order.service.ts#calculateTotal`) ou bloco de código inline (` ``` `)
- **Ação para caminho**: ler o arquivo com Read e localizar a função indicada
- **Ação para bloco de código**: analisar o código diretamente no input
- **Extrair**: parâmetros, retorno, coleções manipuladas, operações de dados
- Usar como base para o Processo de Análise abaixo

### Tipo 5 — Combinação
- **Detectar**: input contém mais de um dos tipos acima (ex: link da ferramenta de gestão + caminho de repositório)
- **Ação**: executar extração de cada tipo identificado separadamente
- **Combinar**: requisitos funcionais (ferramenta de gestão) com estrutura real do código (repositório/função)
- **Cruzar**: verificar se entidades da ferramenta de gestão existem no código e vice-versa
- Usar a visão combinada como base para o Processo de Análise abaixo

> **Nota:** Ao iniciar o relatório, inclua uma linha identificando a fonte analisada. Exemplo: `**Fonte analisada:** Épico #abc123 — Gestão de Pedidos + src/modules/orders/`

---

## Processo de Análise

Ao receber uma funcionalidade, entidade, relacionamento ou coleção de dados, analise sistematicamente os três cenários de quantidade:

### 1. Cenário Zero (0)

**Como o sistema se comporta quando não há nenhuma ocorrência?**

Questione:
- O que acontece quando uma lista está vazia?
- Como a UI se comporta com zero itens?
- Há tratamento especial para ausência de dados?
- O sistema gera erros ou exibe mensagens apropriadas?
- Há otimizações específicas para evitar processamento desnecessário?

**Áreas de análise:**
- **UI/UX**: Mensagens de "lista vazia", estados vazios, call-to-actions apropriados
- **Performance**: Evitar queries desnecessárias, processamento de arrays vazios
- **Lógica de Negócio**: Validações que dependem de existência de dados
- **APIs**: Respostas adequadas para coleções vazias (ex: `[]` vs `null` vs erro)

**Exemplos de análise:**
- "Nenhum pedido" → Como exibir? Há mensagem de onboarding?
- "Lista de produtos vazia" → Há fallback ou estado de carregamento?
- "Usuário sem foto de perfil" → Há avatar padrão ou placeholder?

### 2. Cenário Um (1)

**Como o sistema se comporta quando há exatamente uma ocorrência?**

Questione:
- Há otimizações específicas para o caso único?
- O sistema trata "um" como caso especial ou como parte de "muitos"?
- Há simplificações possíveis quando há apenas um item?
- A lógica funciona corretamente para um único elemento?

**Áreas de análise:**
- **Performance**: Carregar um único item sem paginação ou overhead
- **Lógica Simplificada**: Evitar loops ou processamento em lote quando há apenas um
- **UI/UX**: Exibição direta vs. lista com um item
- **Validações**: Regras que podem ser diferentes para um único item

**Exemplos de análise:**
- "Um único pedido" → Carregamento direto ou lista com um item?
- "Apenas um item no carrinho" → Exibição simplificada?
- "Usuário com uma única conexão" → Tratamento especial ou genérico?

### 3. Cenário Muitos (N)

**Como o sistema se comporta quando há um grande volume de ocorrências?**

Questione:
- O que constitui "muitos" neste contexto? (100? 1.000? 1.000.000?)
- O sistema escala linearmente ou degrada com volume?
- Há limites máximos definidos?
- Quais otimizações são necessárias para alto volume?

**Áreas de análise:**

#### Performance de Leitura
- **Paginação**: Implementação adequada (cursor-based vs offset-based)
- **Indexação**: Índices adequados para queries frequentes
- **Caching**: Estratégias de cache para dados frequentemente acessados
- **Denormalização**: Vistas materializadas ou dados pré-agregados
- **Sharding/Particionamento**: Distribuição de dados em múltiplos nós

#### Performance de Escrita
- **Escritas Assíncronas**: Uso de filas de mensagens para alto volume
- **Processamento em Lote**: Batch operations para múltiplas inserções
- **Rate Limiting**: Controle de taxa para prevenir sobrecarga
- **Validações Distribuídas**: Validações eficientes em sistemas multi-nó

#### Uso de Recursos
- **Memória**: Evitar carregar milhões de registros na memória
  - Soluções: Stream processing, processamento em chunks, paginação
- **CPU**: Complexidade algorítmica adequada
  - O(N) aceitável para muitos, O(N²) pode ser catastrófico
- **Espaço em Disco**: Estratégias de arquivamento ou compressão

#### Design de APIs e Banco de Dados
- **APIs**: Suporte a paginação, filtros, ordenação, busca
- **Banco de Dados**: Escolha adequada (relacional vs NoSQL) baseada em volume
- **Particionamento**: Estratégias de sharding para distribuir carga

**Exemplos de análise:**
- "Milhões de pedidos" → Paginação eficiente? Indexação adequada?
- "Carrinho com centenas de itens" → Performance de renderização? Limites?
- "Usuário com milhões de seguidores" → Agregação eficiente? Cache?

## Análise de Escalabilidade por Operação

Para cada tipo de operação, avalie o impacto do volume:

### Operações de Leitura (Read)

**Zero:**
- Queries otimizadas para retornar vazio rapidamente
- Evitar JOINs desnecessários quando não há dados

**Um:**
- Queries diretas sem overhead de paginação
- Cache eficiente para itens únicos

**Muitos:**
- Paginação obrigatória
- Indexação adequada
- Caching estratégico
- Agregações pré-calculadas quando possível

### Operações de Escrita (Create/Update)

**Zero:**
- Validações que não falham em ausência de dados relacionados
- Criação inicial eficiente

**Um:**
- Escritas diretas sem overhead de batch
- Validações simplificadas quando aplicável

**Muitos:**
- Processamento em lote (batch operations)
- Filas de mensagens para escritas assíncronas
- Rate limiting para prevenir sobrecarga
- Validações distribuídas eficientes

### Operações de Exclusão (Delete)

**Zero:**
- Operações idempotentes (não falham se já vazio)

**Um:**
- Exclusão direta sem overhead

**Muitos:**
- Exclusão em massa eficiente
- Background jobs para processamento assíncrono
- Particionamento para facilitar exclusões

## Análise de Relacionamentos e Coleções

Para cada relacionamento ou coleção identificada:

### Relacionamentos 1:1
- **Zero**: Como tratar quando um lado não existe?
- **Um**: Comportamento padrão esperado
- **Muitos**: Não aplicável (mas considere se o relacionamento pode mudar)

### Relacionamentos 1:N
- **Zero**: Lista vazia vs. null
- **Um**: Otimizações possíveis
- **Muitos**: Estratégias de paginação e agregação

### Relacionamentos N:N
- **Zero**: Tratamento de ausência de conexões
- **Um**: Caso simplificado
- **Muitos**: Estratégias de escalabilidade para grafos grandes

## Identificação de Bugs de Limite

A heurística Count ajuda a identificar bugs comuns:

- **Off-by-one errors**: Loops que processam um item a mais ou a menos
- **Divisão por zero**: Operações matemáticas que falham com zero itens
- **Overflow**: Cálculos que excedem limites com muitos itens
- **Memory leaks**: Acúmulo de dados não liberados em cenários de muitos
- **Race conditions**: Concorrência em operações de contagem ou agregação

## Testes de Carga e Estresse

A heurística Count fornece cenários-chave para testes:

- **Teste de Zero**: Validar comportamento com ausência completa
- **Teste de Um**: Validar caso simplificado
- **Teste de Muitos**: Identificar limites de performance e capacidade máxima
- **Teste de Crescimento**: Simular crescimento gradual de zero para muitos

## Implementando Testes para Cenários Count

A heurística Count (0, 1, Muitos) fornece cenários para estruturar testes. Consulte [TEST_STRATEGY.md](../../utils/testes/TEST_STRATEGY.md) com seu requisito/código para gerar relatório; use o relatório com os guides em [utils/testes/](../../utils/testes/) (TEST_UNIT_GUIDE, TEST_INTEGRATION_GUIDE, etc.) para implementação. Resumo:

### Estratégia de Testes por Cenário

**Zero (0) - Ausência Total:**
- **Unitários**: Funções com listas vazias, valores ausentes, defaults quando não há dados
- **Integração**: Queries que retornam vazio, comportamento com tabelas vazias
- **Serviço**: APIs retornando arrays vazios [], estados vazios
- **E2E**: UI com empty states, mensagens de "nenhum item encontrado"

**Um (1) - Caso Simplificado:**
- **Unitários**: Funções com elemento único, otimizações específicas
- **Integração**: Queries com resultado único (findOne, findById)
- **Serviço**: APIs retornando objeto único vs. array com 1 elemento
- **E2E**: Visualização de item único, formulários de edição

**Muitos (N) - Alto Volume:**
- **Unitários**: Funções com múltiplos elementos, performance algorítmica (O(N), O(N²))
- **Integração**: Paginação, indexação, queries com milhares de registros
- **Serviço**: APIs com paginação, rate limiting, filtros eficientes
- **E2E**: Listagens paginadas, scrolling infinito, performance de renderização

### Aplicação em Cada Camada

**Testes Unitários com Count:**

```typescript
// TypeScript
describe('calculateTotal - Count (0, 1, Muitos)', () => {
  it('deve retornar 0 para lista vazia (Count - Zero)', () => {
    const result = calculateTotal([]);
    expect(result).toBe(0);
  });
  
  it('deve calcular total para 1 item (Count - Um)', () => {
    const result = calculateTotal([{ amount: 100 }]);
    expect(result).toBe(100);
  });
  
  it('deve calcular total para múltiplos itens (Count - Muitos)', () => {
    const items = [
      { amount: 100 },
      { amount: 200 },
      { amount: 300 }
    ];
    const result = calculateTotal(items);
    expect(result).toBe(600);
  });
  
  it('deve performar bem com 10.000 itens (Count - Muitos + Performance)', () => {
    const items = Array.from({ length: 10000 }, (_, i) => ({ amount: i + 1 }));
    
    const start = performance.now();
    const result = calculateTotal(items);
    const duration = performance.now() - start;
    
    expect(result).toBe(50005000);
    expect(duration).toBeLessThan(100); // < 100ms
  });
});
```

**Testes de Integração com Count:**

```java
// Java + JUnit 5
@Nested
@DisplayName("Count scenarios")
class CountScenarios {
    
    @Test
    @DisplayName("Deve retornar lista vazia quando não há usuários (Count - Zero)")
    void shouldReturnEmptyListWhenNoUsers() {
        List<User> users = userRepository.findAll();
        assertTrue(users.isEmpty());
    }
    
    @Test
    @DisplayName("Deve retornar lista com 1 usuário (Count - Um)")
    void shouldReturnSingleUser() {
        userRepository.save(new User("user@test.com"));
        
        List<User> users = userRepository.findAll();
        assertEquals(1, users.size());
    }
    
    @Test
    @DisplayName("Deve paginar corretamente com 1000 usuários (Count - Muitos)")
    void shouldPaginateWithManyUsers() {
        // Criar 1000 usuários
        for (int i = 0; i < 1000; i++) {
            userRepository.save(new User("user" + i + "@test.com"));
        }
        
        // Buscar primeira página
        Page<User> page = userRepository.findAll(PageRequest.of(0, 20));
        
        assertEquals(20, page.getContent().size());
        assertEquals(1000, page.getTotalElements());
        assertEquals(50, page.getTotalPages());
    }
}
```

**Testes E2E com Count:**

```typescript
// Playwright
test('deve exibir empty state quando não há transações (Count - Zero)', async ({ page }) => {
  await page.goto('/transactions');
  
  await expect(page.locator('[data-testid="empty-state"]'))
    .toContainText('Nenhuma transação encontrada');
  await expect(page.locator('[data-testid="empty-state-cta"]'))
    .toContainText('Fazer primeira transferência');
});

test('deve renderizar lista com múltiplas transações (Count - Muitos)', async ({ page }) => {
  // Setup: Criar 50 transações
  await createManyTransactions(50);
  
  await page.goto('/transactions');
  
  // Verificar paginação
  const rows = page.locator('table tbody tr');
  await expect(rows).toHaveCount(20); // Primeira página
  
  // Verificar indicador de páginas
  await expect(page.locator('[data-testid="pagination"]'))
    .toContainText('1 de 3');
});
```

### Identificando Bugs de Limite com Count

A heurística Count ajuda a identificar bugs comuns:

**Off-by-one errors:**
```typescript
// Teste que pega erro de índice
it('deve processar todos os elementos (não N-1)', () => {
  const items = [1, 2, 3, 4, 5];
  const result = processAll(items);
  expect(result.processed).toBe(5); // Não 4!
});
```

**Divisão por zero:**
```typescript
// Teste para evitar divisão por zero
it('deve retornar 0 quando lista vazia (não dividir por 0)', () => {
  const result = calculateAverage([]);
  expect(result).toBe(0); // Não NaN ou Infinity
});
```

**Memory leaks com Muitos:**
```typescript
// Teste de performance e memória
it('não deve estourar memória com 100k itens', () => {
  const startMemory = process.memoryUsage().heapUsed;
  
  processLargeList(100_000);
  
  const endMemory = process.memoryUsage().heapUsed;
  const memoryIncrease = endMemory - startMemory;
  
  // Aumento aceitável (ex: < 50MB)
  expect(memoryIncrease).toBeLessThan(50 * 1024 * 1024);
});
```

Consulte TEST_STRATEGY e os guides em [utils/testes/](../../utils/testes/) para exemplos em TypeScript e Java por camada.

## Checklist de Análise Count

Ao aplicar a heurística, verifique:

- [ ] **Zero**: Tratamento adequado de ausência, mensagens apropriadas, otimizações
- [ ] **Um**: Caso simplificado funcionando, otimizações quando aplicável
- [ ] **Muitos**: Definição clara do que é "muitos", otimizações de performance implementadas
- [ ] **Leitura**: Paginação, indexação, caching para volumes grandes
- [ ] **Escrita**: Processamento em lote, filas, rate limiting para alto volume
- [ ] **Recursos**: Uso eficiente de memória, CPU e espaço em disco
- [ ] **APIs**: Suporte a paginação, filtros, ordenação
- [ ] **Banco de Dados**: Estratégias de particionamento e sharding quando necessário
- [ ] **Bugs de Limite**: Validação de off-by-one, divisão por zero, overflow
- [ ] **Testes**: Cenários de carga definidos para validar escalabilidade

## Objetivo Final

Garantir que o sistema funcione corretamente e eficientemente em todas as faixas de quantidade, desde a ausência completa até o volume massivo. A análise deve identificar:

1. **Falhas de Limite**: Comportamentos incorretos em cenários extremos
2. **Oportunidades de Otimização**: Melhorias de performance para diferentes volumes
3. **Gaps de Implementação**: Funcionalidades faltantes para lidar com extremos
4. **Riscos de Escalabilidade**: Pontos que podem colapsar com crescimento de volume

Ao dominar a heurística Count, você garante que seu sistema é resiliente a variações de volume e performático, independentemente da quantidade de dados ou transações que precisa manipular.
