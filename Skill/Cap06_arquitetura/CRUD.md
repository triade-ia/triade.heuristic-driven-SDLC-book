---
name: crud-heuristic
description: Analisa funcionalidades do sistema através das quatro operações fundamentais de persistência de dados (Create, Read, Update, Delete), focando em validações, performance, segurança e escalabilidade. Use quando analisando requisitos de manipulação de dados, projetando APIs, modelando entidades, ou quando o usuário solicita aplicação da heurística CRUD.
---

# Heurística CRUD

Analisa cada funcionalidade do sistema através da lente das quatro operações básicas de persistência de dados: **Create** (Criar), **Read** (Ler), **Update** (Atualizar) e **Delete** (Excluir).

## Atuação

Você é um arquiteto de software focado em robustez, eficiência e segurança na manipulação de dados. Sua tarefa é aplicar a heurística CRUD sobre funcionalidades do sistema, investigando como cada operação manipula dados e quais são as implicações em diferentes contextos, especialmente em cenários de alto volume e concorrência.

## Processo de Análise

Ao receber uma funcionalidade ou entidade de dados, analise cada uma das quatro operações CRUD:

### Create (Criação)

Questione:
- Como os dados são inseridos no sistema?
- Quais validações são aplicadas antes da inserção?
- Qual o impacto de múltiplas criações simultâneas?
- Há limites de taxa (rate limiting) para criação?
- Como são tratados dados duplicados ou conflitos?

**Em escala, considere:**
- Escritas assíncronas para alto volume (filas de mensagens)
- Write amplification e otimizações de escrita
- Validações distribuídas em sistemas multi-nó

### Read (Leitura)

Questione:
- Como os dados são recuperados?
- Quais filtros e ordenações são usados?
- Qual a performance da leitura para diferentes volumes de dados?
- Há paginação implementada?
- Quais campos são retornados e há controle de acesso?

**Em escala, considere:**
- Indexação adequada para queries frequentes
- Caching (memória, Redis) para dados frequentemente lidos
- Denormalização ou vistas materializadas para leituras complexas
- Sharding/particionamento para distribuir carga de leitura

### Update (Atualização)

Questione:
- Como os dados são modificados?
- Quais campos podem ser atualizados e por quem?
- Qual o impacto de atualizações simultâneas no mesmo dado?
- Há controle de versão ou otimistic locking?
- Como são tratados conflitos de concorrência?

**Em escala, considere:**
- Transações otimizadas para minimizar bloqueios
- Contenção em cenários Multi-User
- Modelo de consistência adequado (forte vs eventual)
- Validações distribuídas

### Delete (Exclusão)

Questione:
- Como os dados são removidos?
- A exclusão é lógica (soft delete) ou física (hard delete)?
- Há exclusão em cascata (cascading delete) em dados relacionados?
- Quais são as implicações de segurança e integridade?
- Há auditoria ou logs de exclusão?

**Em escala, considere:**
- Exclusão em massa eficiente sem bloquear o sistema
- Exclusão lógica vs física para performance
- Background jobs para processamento assíncrono
- Particionamento para facilitar exclusões

## Análise de Segurança e Acesso

Para cada operação CRUD, avalie:
- **Controle de acesso**: Quem pode executar cada operação?
- **Permissões granulares**: Há controle fino sobre campos específicos?
- **Auditoria**: Operações são registradas para compliance?
- **Proteção contra acesso indevido**: Há validação de autorização antes de cada operação?

## Análise de Consistência e Integridade

Questione:
- Qual o modelo de consistência adequado para cada dado e operação (forte, eventual)?
- Como as validações garantem integridade em todos os nós do sistema?
- Como os Deletes afetam referências em outros sistemas?
- Há transações distribuídas quando necessário?

## Implementando Testes para Operações CRUD

Para cada operação CRUD identificada, crie testes que validem o comportamento em diferentes camadas. Consulte [TEST_STRATEGY.md](../../utils/testes/TEST_STRATEGY.md) com seu requisito/código para gerar relatório em `output/`; depois use o relatório com [TEST_INTEGRATION_GUIDE.md](../../utils/testes/TEST_INTEGRATION_GUIDE.md) e [TEST_SERVICE_GUIDE.md](../../utils/testes/TEST_SERVICE_GUIDE.md) para implementação. Detalhes:

### Testes por Operação

**Create (Criação):**
- **Unitários**: Validação de regras de negócio antes de criar, geração de IDs, defaults
- **Integração**: Inserção no banco, violação de constraints (unique, not null, FK), criação com relacionamentos
- **Serviço**: API POST com validação completa, autenticação, autorização
- **E2E**: Fluxo completo de criação através da UI, validação de campos

**Read (Leitura):**
- **Unitários**: Lógica de filtros, ordenação, transformações de dados
- **Integração**: Queries do banco, joins, paginação com dados reais
- **Serviço**: API GET com filtros, paginação, autorização (usuário só vê seus dados)
- **E2E**: Listagens, buscas, visualização de detalhes

**Update (Atualização):**
- **Unitários**: Validação de mudanças, merge de dados parciais
- **Integração**: Update no banco, concorrência (optimistic locking), versionamento
- **Serviço**: API PUT/PATCH com validação, autorização, idempotência
- **E2E**: Edição através da UI, preservação de dados não alterados

**Delete (Exclusão):**
- **Unitários**: Validação de permissão, lógica de soft vs hard delete
- **Integração**: Deleção no banco, cascata, integridade referencial
- **Serviço**: API DELETE com autorização, idempotência, auditoria
- **E2E**: Exclusão através da UI com confirmação

### Camadas de Teste Recomendadas

**Distribuição por operação:**

| Operação | Unitários | Integração | Serviço | E2E |
|----------|-----------|------------|---------|-----|
| Create | Validações | Inserção + constraints | API POST completa | Formulário |
| Read | Filtros/ordenação | Queries + joins | API GET completa | Listagem/busca |
| Update | Merge de dados | Concorrência | API PUT/PATCH | Edição |
| Delete | Regras | Cascata | API DELETE | Exclusão + confirmação |

### Exemplo de Cobertura CRUD em Testes

Para uma entidade User com operações CRUD completas:

**Create:**
```typescript
// Unitários
it('deve validar email antes de criar')
it('deve gerar ID único')

// Integração
it('deve inserir usuário no banco')
it('deve rejeitar email duplicado')

// Serviço
it('deve criar usuário via POST /users')
it('deve retornar 422 com email inválido')

// E2E
it('deve criar usuário através do formulário')
```

**Read:**
```typescript
// Integração
it('deve buscar usuário por ID')
it('deve retornar null quando não existe')
it('deve paginar lista de usuários')

// Serviço
it('deve retornar usuário via GET /users/:id')
it('deve retornar 404 quando não existe')
it('deve listar usuários com paginação')

// E2E
it('deve visualizar perfil do usuário')
it('deve buscar usuários por nome')
```

**Update:**
```typescript
// Unitários
it('deve validar mudanças de email')

// Integração
it('deve atualizar apenas campos alterados')
it('deve falhar em conflito de versão (concorrência)')

// Serviço
it('deve atualizar via PATCH /users/:id')
it('deve retornar 409 em conflito de versão')

// E2E
it('deve editar perfil e salvar mudanças')
it('deve mostrar erro em conflito')
```

**Delete:**
```typescript
// Integração
it('deve fazer soft delete (marcar como deleted)')
it('deve ser idempotente (não falha se já deletado)')

// Serviço
it('deve deletar via DELETE /users/:id')
it('deve retornar 204 ou 200')

// E2E
it('deve deletar com confirmação em modal')
```

Os guides em [utils/testes/](../../utils/testes/) fornecem exemplos em TypeScript e Java para cada operação.

## Checklist de Análise CRUD

Ao aplicar a heurística, verifique:

- [ ] **Create**: Validações, tratamento de duplicatas, rate limiting
- [ ] **Read**: Performance, indexação, caching, paginação
- [ ] **Update**: Controle de concorrência, validações, consistência
- [ ] **Delete**: Tipo de exclusão, cascata, auditoria, integridade
- [ ] **Segurança**: Controle de acesso granular para cada operação
- [ ] **Escalabilidade**: Otimizações para alto volume e concorrência
- [ ] **Consistência**: Modelo adequado para cada contexto

## Objetivo Final

Garantir que cada operação CRUD seja robusta, eficiente e segura, identificando gaps de implementação, riscos de performance e vulnerabilidades de segurança antes da codificação. A análise deve focar especialmente em cenários de escala, onde operações triviais podem se tornar gargalos catastróficos.
