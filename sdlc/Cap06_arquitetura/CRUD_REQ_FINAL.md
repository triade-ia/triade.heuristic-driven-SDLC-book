# Análise CRUD: Sistema de Transferência de QualiPoints

## Contexto do Caso

Este documento apresenta a aplicação da heurística **CRUD** sobre o requisito de transferência de QualiPoints entre usuários. A análise investiga como cada operação fundamental de persistência de dados (Create, Read, Update, Delete) se manifesta no sistema, identificando gaps de implementação, riscos de performance e vulnerabilidades de segurança.

## Entidades Identificadas

O sistema de transferência de QualiPoints envolve as seguintes entidades principais:
- **Carteiras (Wallets)**: Armazenam saldo de QualiPoints por usuário
- **Transações (Transactions)**: Registram histórico de transferências
- **Usuários (Users)**: Contêm status da conta (Ativo/Inativo)

## Análise CRUD por Entidade

### 1. Carteiras (Wallets)

#### Create (Criação)

**Questões levantadas:**
- ❓ Como as carteiras são criadas? São criadas automaticamente no registro do usuário ou sob demanda?
- ❓ Qual o saldo inicial de uma nova carteira? Zero ou há bônus de boas-vindas?
- ❓ Há validação de unicidade (uma carteira por usuário)?
- ❓ Como são tratadas múltiplas criações simultâneas da mesma carteira?

**Gaps identificados:**
- ⚠️ Requisito não especifica processo de criação de carteira
- ⚠️ Não há definição de saldo inicial
- ⚠️ Falta estratégia para prevenir criação duplicada em cenários de alta concorrência

**Recomendações:**
- Implementar criação automática de carteira no registro do usuário (saldo inicial = 0)
- Usar constraint UNIQUE no banco para garantir uma carteira por usuário
- Considerar idempotência na criação (verificar existência antes de criar)

#### Read (Leitura)

**Questões levantadas:**
- ❓ Como o saldo é recuperado? Via endpoint específico ou junto com dados do usuário?
- ❓ Há necessidade de histórico de saldo ou apenas saldo atual?
- ❓ Quais filtros são aplicados? (ex: por usuário, por período)
- ❓ Qual a performance esperada para leitura de saldo?

**Análise do requisito:**
- ✅ Saldo é retornado após transferência (`new_balance` na resposta)
- ⚠️ Não há endpoint explícito para consulta de saldo
- ⚠️ Não há especificação de paginação ou limites

**Recomendações para escala:**
- Implementar endpoint `GET /api/v1/wallets/me` para consulta de saldo
- Considerar caching do saldo em Redis para leituras frequentes (TTL curto: 30-60s)
- Indexar `user_id` na tabela wallets para queries rápidas
- Implementar auditoria de consultas de saldo para compliance

#### Update (Atualização)

**Questões levantadas:**
- ❓ Como o saldo é atualizado durante transferências?
- ❓ Quais campos podem ser atualizados? Apenas `balance` ou há outros campos?
- ❓ Qual o impacto de atualizações simultâneas no mesmo saldo?
- ❓ Há controle de versão ou optimistic locking?

**Análise do requisito:**
- ✅ Requisito especifica atomicidade (transação de banco de dados)
- ✅ Operação síncrona com confirmação em tempo real
- ⚠️ Não especifica estratégia de locking (pessimista vs otimista)
- ⚠️ Não há tratamento explícito de conflitos de concorrência

**Riscos identificados:**
- 🔴 **Race condition crítica**: Duas transferências simultâneas podem passar validação de saldo antes de qualquer débito, permitindo saldo negativo
- 🔴 **Falta de isolamento**: Sem locks adequados, leituras podem ocorrer durante atualizações

**Recomendações:**
- Implementar lock pessimista (`SELECT FOR UPDATE`) na carteira do remetente durante toda a transação
- Validar saldo dentro da transação, não antes
- Implementar timeout de transação (5-10 segundos) para prevenir deadlocks
- Considerar versionamento de saldo para detecção de conflitos

**Em escala:**
- Transações otimizadas para minimizar tempo de bloqueio
- Particionamento de carteiras por shard se volume for muito alto
- Monitoramento de contenção de locks em produção

#### Delete (Exclusão)

**Questões levantadas:**
- ❓ Carteiras podem ser excluídas? Em que cenário?
- ❓ A exclusão é lógica (soft delete) ou física (hard delete)?
- ❓ O que acontece com o saldo ao excluir uma carteira?
- ❓ Há exclusão em cascata quando um usuário é removido?

**Gaps identificados:**
- ⚠️ Requisito não aborda exclusão de carteiras
- ⚠️ Não há política definida para encerramento de contas

**Recomendações:**
- Implementar exclusão lógica (soft delete) com flag `deleted_at`
- Manter histórico de carteiras excluídas para auditoria
- Definir política: transferir saldo restante antes de exclusão ou converter em crédito administrativo
- Implementar exclusão em cascata apenas após confirmação explícita

---

### 2. Transações (Transactions)

#### Create (Criação)

**Questões levantadas:**
- ❓ Como as transações são registradas no sistema?
- ❓ Quais validações são aplicadas antes de criar o registro?
- ❓ Qual o impacto de múltiplas criações simultâneas?
- ❓ Há limites de taxa (rate limiting) para criação de transações?

**Análise do requisito:**
- ✅ Requisito especifica idempotência via `idempotency-key`
- ✅ Transação é criada dentro de operação atômica
- ✅ Resposta inclui `transaction_id` (UUID)

**Gaps identificados:**
- ⚠️ Não especifica estrutura completa do registro de transação
- ⚠️ Não define campos obrigatórios além de `transaction_id`
- ⚠️ Falta estratégia para lidar com alto volume de transações

**Recomendações:**
- Estrutura mínima de transação:
  - `transaction_id` (UUID, primary key)
  - `sender_id` (referência ao usuário remetente)
  - `recipient_id` (referência ao usuário destinatário)
  - `amount` (valor transferido)
  - `created_at` (timestamp)
  - `idempotency_key` (para prevenção de duplicatas)
  - `status` (pending, completed, failed)
- Implementar verificação de idempotência em duas camadas:
  - Cache Redis com TTL de 24 horas (performance)
  - Verificação no banco por transações recentes duplicadas (confiabilidade)
- Considerar rate limiting por usuário (ex: máximo de 100 transferências/hora)
- Indexar `idempotency_key` para queries rápidas de verificação

**Em escala:**
- Para alto volume: considerar escritas assíncronas em fila de mensagens
- Particionar tabela de transações por data (partitioning mensal)
- Implementar arquivamento de transações antigas (> 1 ano)

#### Read (Leitura)

**Questões levantadas:**
- ❓ Como o histórico de transações é recuperado?
- ❓ Quais filtros são aplicados? (por usuário, por período, por status)
- ❓ Há paginação implementada?
- ❓ Qual a performance esperada para leitura de histórico?

**Gaps identificados:**
- ⚠️ Requisito não especifica endpoint para consulta de histórico
- ⚠️ Não há definição de filtros ou ordenação
- ⚠️ Falta especificação de paginação

**Recomendações:**
- Implementar endpoint `GET /api/v1/transactions` com:
  - Filtros: `sender_id`, `recipient_id`, `status`, `date_from`, `date_to`
  - Ordenação: `created_at DESC` (mais recentes primeiro)
  - Paginação: `limit` (padrão: 20) e `offset` ou cursor-based
- Indexar campos de filtro frequente:
  - `(sender_id, created_at)` para histórico do remetente
  - `(recipient_id, created_at)` para histórico do destinatário
  - `created_at` para queries por período
- Implementar caching para consultas frequentes (ex: últimas 10 transações do usuário)
- Considerar vistas materializadas para relatórios complexos

**Em escala:**
- Sharding por `user_id` se volume for muito alto
- Implementar read replicas para distribuir carga de leitura
- Cache de transações recentes em Redis (últimas 24 horas)

#### Update (Atualização)

**Questões levantadas:**
- ❓ Transações podem ser atualizadas após criação?
- ❓ Quais campos podem ser modificados? (ex: status, notas)
- ❓ Há cenários de correção ou estorno de transações?
- ❓ Como são tratados conflitos de atualização simultânea?

**Análise do requisito:**
- ⚠️ Requisito não aborda atualização de transações
- ⚠️ Não há especificação de estorno ou cancelamento

**Recomendações:**
- Implementar atualização apenas de campos não-críticos:
  - `notes` (observações administrativas)
  - `status` (apenas em casos específicos: pending → failed)
- **Não permitir** atualização de: `amount`, `sender_id`, `recipient_id`, `created_at`
- Para estornos: criar nova transação reversa em vez de atualizar existente
- Implementar controle de versão (`version` field) para detecção de conflitos
- Registrar auditoria de todas as atualizações

#### Delete (Exclusão)

**Questões levantadas:**
- ❓ Transações podem ser excluídas?
- ❓ A exclusão é lógica ou física?
- ❓ Há implicações de integridade referencial?
- ❓ Como são tratadas exclusões em massa?

**Recomendações:**
- **Não permitir exclusão física** de transações (requisito de auditoria e compliance)
- Implementar exclusão lógica apenas para casos administrativos específicos
- Manter histórico completo para relatórios e auditoria
- Implementar arquivamento de transações antigas (> 2 anos) em storage frio
- Para exclusões administrativas: registrar motivo e usuário responsável

---

### 3. Usuários (Users)

#### Create (Criação)

**Questões levantadas:**
- ❓ Como usuários são criados no sistema?
- ❓ Qual o status inicial? (deve ser "Ativo" ou há período de verificação?)
- ❓ Há validação de unicidade de email/ID?

**Gaps identificados:**
- ⚠️ Requisito não especifica processo de criação de usuário
- ⚠️ Não define status inicial

**Recomendações:**
- Status inicial deve ser validado conforme política de negócio
- Garantir criação automática de carteira associada ao criar usuário
- Implementar validação de unicidade de identificadores

#### Read (Leitura)

**Questões levantadas:**
- ❓ Como o status do usuário é verificado durante transferência?
- ❓ Há endpoint para consulta de usuário?
- ❓ Quais campos são retornados?

**Análise do requisito:**
- ✅ Requisito especifica validação de status "Ativo" antes de transferência
- ⚠️ Não especifica como o status é recuperado

**Recomendações:**
- Validar status dentro da transação com lock (`SELECT FOR UPDATE`)
- Implementar endpoint `GET /api/v1/users/{id}` para consulta
- Indexar campo `status` para queries rápidas
- Considerar caching de status de usuários ativos (lista em Redis)

#### Update (Atualização)

**Questões levantadas:**
- ❓ Como o status do usuário é atualizado?
- ❓ O que acontece com transferências pendentes quando status muda?
- ❓ Há controle de concorrência na atualização de status?

**Riscos identificados:**
- 🔴 **Race condition**: Status pode mudar durante processamento de transferência
- 🔴 **Inconsistência**: Transferência pode ser processada para usuário que acabou de ser desativado

**Recomendações:**
- Validar status dentro da transação de transferência com lock
- Implementar controle de versão para detecção de mudanças simultâneas
- Notificar usuário sobre transferências pendentes ao desativar conta
- Implementar fila de processamento para transferências de usuários com status em transição

#### Delete (Exclusão)

**Questões levantadas:**
- ❓ Usuários podem ser excluídos?
- ❓ O que acontece com carteira e transações associadas?
- ❓ Há exclusão em cascata?

**Recomendações:**
- Implementar exclusão lógica (soft delete) com flag `deleted_at`
- Manter histórico completo de transações mesmo após exclusão
- Não permitir exclusão física de usuários com transações ativas
- Implementar processo de arquivamento após período de retenção legal

---

## Análise de Segurança e Acesso

### Controle de Acesso por Operação CRUD

| Operação | Entidade | Controle Necessário | Status |
|----------|----------|---------------------|--------|
| Create | Wallet | Apenas sistema (criação automática) | ⚠️ Não especificado |
| Read | Wallet | Usuário autenticado (apenas própria carteira) | ⚠️ Não especificado |
| Update | Wallet | Apenas sistema (durante transferências) | ✅ Implícito |
| Delete | Wallet | Administrador | ⚠️ Não especificado |
| Create | Transaction | Usuário autenticado (apenas como remetente) | ✅ JWT especificado |
| Read | Transaction | Usuário autenticado (próprias transações) | ⚠️ Não especificado |
| Update | Transaction | Administrador (casos específicos) | ⚠️ Não especificado |
| Delete | Transaction | Não permitido | ⚠️ Não especificado |
| Read | User | Usuário autenticado (validação de destinatário) | ✅ Validado no requisito |
| Update | User | Administrador ou próprio usuário (campos específicos) | ⚠️ Não especificado |

**Gaps de segurança identificados:**
- ⚠️ Falta especificação de controle de acesso granular
- ⚠️ Não há definição de roles/permissões
- ⚠️ Falta validação explícita de autorização antes de cada operação

**Recomendações:**
- Implementar middleware de autorização que valide permissões antes de cada operação CRUD
- Validar que usuário só pode acessar/modificar seus próprios dados
- Implementar auditoria de todas as operações CRUD para compliance
- Considerar RBAC (Role-Based Access Control) para diferentes níveis de acesso

---

## Análise de Consistência e Integridade

### Modelo de Consistência

**Operações críticas que requerem consistência forte:**
- ✅ Transferência de QualiPoints (débito + crédito atômico)
- ✅ Validação de saldo (deve ocorrer dentro da transação)
- ✅ Validação de status do destinatário (deve ocorrer dentro da transação)

**Operações que podem usar consistência eventual:**
- 📊 Relatórios e estatísticas (podem usar dados ligeiramente desatualizados)
- 📊 Histórico de transações (leitura pode usar read replica)

**Recomendações:**
- Usar transações ACID para operações críticas (transferências)
- Implementar read replicas para operações de leitura não-críticas
- Considerar eventual consistency apenas para relatórios e analytics
- Implementar validações distribuídas se sistema for multi-nó

### Integridade Referencial

**Dependências identificadas:**
- Transaction → User (sender_id, recipient_id)
- Transaction → Wallet (implicitamente através de user_id)
- Wallet → User (user_id)

**Recomendações:**
- Implementar foreign keys no banco de dados
- Definir comportamento de cascata apropriado:
  - `ON DELETE RESTRICT` para transações (não permitir exclusão de usuário com transações)
  - `ON DELETE CASCADE` apenas para dados não-críticos
- Validar integridade antes de operações de exclusão

---

## Checklist de Análise CRUD

### Create (Criação)
- [x] Validações identificadas (saldo, status, limites)
- [x] Tratamento de duplicatas (idempotency-key)
- [ ] Rate limiting especificado
- [x] Estratégia de criação de carteira definida
- [ ] Processo de criação de usuário especificado

### Read (Leitura)
- [ ] Endpoints de consulta especificados
- [ ] Paginação implementada
- [ ] Indexação definida
- [ ] Caching estratégico planejado
- [ ] Performance estimada para diferentes volumes

### Update (Atualização)
- [x] Controle de concorrência identificado (locks necessários)
- [x] Validações dentro da transação
- [ ] Campos atualizáveis definidos
- [ ] Controle de versão implementado
- [ ] Estratégia de resolução de conflitos

### Delete (Exclusão)
- [x] Tipo de exclusão definido (soft delete recomendado)
- [ ] Política de cascata especificada
- [ ] Auditoria de exclusões
- [ ] Processo de arquivamento
- [ ] Integridade referencial garantida

### Segurança
- [x] Controle de acesso granular identificado
- [ ] Permissões por operação CRUD definidas
- [ ] Auditoria de operações planejada
- [ ] Validação de autorização antes de cada operação

### Escalabilidade
- [x] Otimizações para alto volume identificadas
- [x] Estratégias de caching definidas
- [x] Particionamento considerado
- [ ] Sharding planejado se necessário
- [ ] Read replicas para distribuir carga

### Consistência
- [x] Modelo de consistência adequado identificado
- [x] Transações ACID para operações críticas
- [ ] Validações distribuídas se multi-nó
- [ ] Estratégia de resolução de conflitos

---

## Gaps de Implementação Identificados

### Críticos (Devem ser resolvidos antes do desenvolvimento)

1. **Estratégia de Locking**
   - Definir uso de locks pessimistas (`SELECT FOR UPDATE`) para carteiras
   - Especificar ordem de aquisição de locks para prevenir deadlocks
   - Definir timeout de transação

2. **Endpoints de Consulta**
   - Especificar endpoint para consulta de saldo (`GET /api/v1/wallets/me`)
   - Especificar endpoint para histórico de transações (`GET /api/v1/transactions`)
   - Definir filtros, ordenação e paginação

3. **Controle de Acesso**
   - Definir permissões por operação CRUD
   - Especificar validação de autorização antes de cada operação
   - Definir roles e níveis de acesso

4. **Estrutura de Dados**
   - Especificar schema completo de transações
   - Definir campos obrigatórios e opcionais
   - Especificar índices necessários

### Importantes (Devem ser resolvidos durante desenvolvimento)

5. **Criação de Carteira**
   - Definir processo de criação automática
   - Especificar saldo inicial
   - Implementar idempotência na criação

6. **Exclusão Lógica**
   - Definir política de soft delete para todas as entidades
   - Especificar processo de arquivamento
   - Definir período de retenção

7. **Auditoria**
   - Especificar quais operações devem ser auditadas
   - Definir formato de logs de auditoria
   - Planejar armazenamento de logs

### Desejáveis (Podem ser implementados em iterações futuras)

8. **Rate Limiting**
   - Definir limites por usuário
   - Especificar estratégia de throttling
   - Implementar mensagens de erro apropriadas

9. **Caching**
   - Definir estratégia de cache para saldos
   - Especificar TTL adequado
   - Planejar invalidação de cache

10. **Particionamento/Sharding**
    - Avaliar necessidade baseada em volume esperado
    - Planejar estratégia de distribuição de dados
    - Definir critérios de particionamento

---

## Riscos de Performance Identificados

### Alto Risco

1. **Contenção de Locks em Alta Concorrência**
   - **Cenário**: Múltiplos usuários transferindo simultaneamente
   - **Impacto**: Timeouts e degradação de performance
   - **Mitigação**: Otimizar tempo de transação, implementar timeout adequado, considerar particionamento

2. **Queries de Histórico sem Paginação**
   - **Cenário**: Usuário com milhares de transações
   - **Impacto**: Timeout e consumo excessivo de memória
   - **Mitigação**: Implementar paginação obrigatória, limitar número máximo de resultados

### Médio Risco

3. **Falta de Indexação**
   - **Cenário**: Queries frequentes sem índices adequados
   - **Impacto**: Queries lentas, especialmente em alto volume
   - **Mitigação**: Indexar campos de filtro frequente, monitorar performance de queries

4. **Validação de Idempotência Ineficiente**
   - **Cenário**: Verificação apenas no banco sem cache
   - **Impacto**: Latência adicional em cada requisição
   - **Mitigação**: Implementar cache Redis para verificação rápida

### Baixo Risco

5. **Leitura de Saldo sem Cache**
   - **Cenário**: Consultas frequentes de saldo
   - **Impacto**: Carga desnecessária no banco
   - **Mitigação**: Implementar cache com TTL curto (30-60s)

---

## Recomendações Prioritárias

### Fase 1: Implementação Inicial (MVP)

1. ✅ Implementar locks pessimistas para carteiras durante transferências
2. ✅ Validar saldo e status dentro da transação
3. ✅ Implementar idempotência em duas camadas (cache + banco)
4. ✅ Criar endpoints básicos de consulta com paginação
5. ✅ Implementar controle de acesso básico (usuário acessa apenas próprios dados)

### Fase 2: Robustez e Performance

6. Implementar caching estratégico (saldos, status de usuários)
7. Adicionar rate limiting por usuário
8. Implementar auditoria completa de operações
9. Otimizar queries com índices adequados
10. Implementar exclusão lógica para todas as entidades

### Fase 3: Escalabilidade

11. Avaliar necessidade de read replicas
12. Implementar particionamento de transações por data
13. Planejar estratégia de sharding se volume justificar
14. Implementar arquivamento de dados antigos

---

## Heurísticas Complementares Recomendadas

- **Multi-User**: Já aplicada - garantir que operações simultâneas sejam seguras
- **State Analysis**: Analisar transições de estado de transações (pending → completed → failed)
- **Dependencies**: Mapear dependências entre entidades para garantir integridade
- **Time (Antes, Durante, Depois)**: Validar que operações ocorram no momento correto
- **Count (0, 1, Muitos)**: Dimensionar sistema baseado em volume esperado de transações

---

## Conclusão

A aplicação da heurística CRUD revelou **gaps significativos** no requisito original, especialmente relacionados a:

1. **Operações de Leitura**: Falta de especificação de endpoints de consulta
2. **Controle de Concorrência**: Necessidade explícita de locks para prevenir race conditions
3. **Segurança**: Falta de especificação detalhada de controle de acesso
4. **Escalabilidade**: Ausência de estratégias para alto volume

A análise CRUD complementa a análise Multi-User anterior, focando especificamente nas operações de persistência de dados e suas implicações em diferentes contextos. Juntas, essas heurísticas fornecem uma visão completa dos requisitos técnicos necessários para implementar um sistema robusto, seguro e escalável.

**Lição Principal**: A heurística CRUD força a consideração explícita de todas as operações fundamentais sobre dados, revelando gaps que poderiam passar despercebidos. Cada operação CRUD representa um ponto potencial de falha, vulnerabilidade ou gargalo de performance que deve ser projetado e testado adequadamente.

---

**Próximos Passos**: Resolver os gaps críticos identificados antes de iniciar o desenvolvimento, especialmente relacionados a estratégia de locking, endpoints de consulta e controle de acesso.
