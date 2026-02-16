# Análise de Dependências (Heurística Dependencies) - REQ_FINAL

## 1. Mapeamento de Relacionamentos (O "Tem um")

### Relacionamento de Dados

**Usuário → Carteira**
- Cada usuário possui uma carteira (relacionamento 1:1)
- A carteira armazena o saldo atual (campo numérico)
- Referência: `user_id` é a chave primária que conecta Usuário e Carteira

**Transação → Remetente e Destinatário**
- Cada transação referencia dois usuários distintos
- Relacionamento: Transação tem um `sender_id` e um `recipient_id`
- A transação armazena: `transaction_id`, `amount`, `timestamp`, `status`

**Transação → Idempotency Key**
- Cada requisição pode ter uma chave de idempotência (armazenada externamente, ex: Redis)
- Relacionamento: Transação tem zero ou uma chave de idempotência associada

**Usuário → Status da Conta**
- Cada usuário possui um status (ex: "Ativo", "Inativo", "Suspenso")
- Validação crítica: apenas usuários "Ativos" podem receber transferências

### Dependência de Fluxo

**Serviços Vitais para o Sucesso da Operação:**
1. **Serviço de Autenticação**: Validação do JWT e extração do `user_id`
2. **Serviço de Banco de Dados**: Leitura de saldo, validação de destinatário, escrita transacional
3. **Serviço de Validação de Usuário**: Verificação de existência e status do destinatário
4. **Serviço de Idempotência** (opcional mas recomendado): Armazenamento temporário da chave

## 2. Matriz de Propagação de Impacto (O "Efeito Dominó")

### Mudança na Origem

**Risco: Alteração de Saldo Concorrente**
- Se o saldo do remetente for alterado por outra operação simultânea, pode ocorrer race condition
- **Mitigação**: Uso de transações com isolamento adequado (ex: SERIALIZABLE ou SELECT FOR UPDATE)

**Risco: Mudança de Status do Destinatário**
- Se o status do destinatário mudar de "Ativo" para "Inativo" entre a validação e a execução, a transferência pode ser processada incorretamente
- **Mitigação**: Validar status dentro da mesma transação que executa a transferência

### Falha de Vizinho

**Cenário 1: Falha do Serviço de Autenticação**
- **Impacto**: Morte da funcionalidade (sem autenticação, não há operação)
- **Degradação Graciosa**: Não aplicável - autenticação é pré-requisito absoluto

**Cenário 2: Falha do Banco de Dados**
- **Impacto**: Morte da funcionalidade (sem persistência, não há transferência)
- **Degradação Graciosa**: Não aplicável - persistência é crítica

**Cenário 3: Falha do Serviço de Idempotência**
- **Impacto**: Degradação (perda de proteção contra duplicatas)
- **Degradação Graciosa**: Operação pode continuar, mas com risco de duplicação em caso de retentativas

**Cenário 4: Timeout na Validação do Destinatário**
- **Impacto**: Bloqueio da operação
- **Degradação Graciosa**: Implementar timeout e retornar erro específico (503 Service Unavailable) após X segundos

### Concorrência

**Gargalo: Tabela de Carteiras**
- Múltiplas transferências simultâneas do mesmo remetente podem causar contenção
- **Solução**: Locks de linha no nível de banco de dados (SELECT FOR UPDATE)

**Gargalo: Validação de Status do Destinatário**
- Múltiplas requisições validando o mesmo destinatário simultaneamente
- **Solução**: Cache de leitura (ex: Redis) com TTL curto para status de usuário

## 3. Gatilhos de Ação Manual (Investigação Complementar) ⚡

**Para complementar a análise do relacionamento Usuário-Carteira, aplique a heurística CRUD:**
- Validar operações de criação, leitura, atualização e exclusão de carteiras
- Garantir que exclusão de usuário tenha tratamento adequado para carteira associada

**Para validar os limites e a escalabilidade de Transações, aplique a heurística Count (0, 1, Muitos):**
- Validar comportamento quando usuário tem 0 transações (primeira transferência)
- Validar comportamento quando usuário tem muitas transações simultâneas
- Validar limites de volume de transações por usuário/período

**Para investigar o comportamento de histórico de transações, aplique a heurística Position (Primeiro, Último, Meio):**
- Validar ordenação de transações (mais recente primeiro?)
- Validar paginação de histórico (se aplicável)

**Para garantir a integridade da busca de destinatários, aplique a heurística Selection (Alguns, Nenhum, Todos):**
- Validar busca quando nenhum usuário corresponde ao critério
- Validar busca quando múltiplos usuários correspondem (caso de busca por nome/email)
- Validar filtro de status "Ativo" em listagens

## 4. Decisões de Desacoplamento e Resiliência

### Isolamento

**Estratégia 1: Circuit Breaker para Validação de Destinatário**
- Implementar circuit breaker entre o serviço de transações e o serviço de validação de usuário
- Em caso de falha repetida, permitir operação com validação simplificada (apenas verificação de existência no banco local)

**Estratégia 2: Queue Assíncrona para Notificações**
- Desacoplar notificações de confirmação da operação principal
- Se o serviço de notificações falhar, a transferência ainda é concluída

**Estratégia 3: Cache de Status de Usuário**
- Cachear status "Ativo" em memória/Redis com TTL de 30 segundos
- Reduz dependência de consultas síncronas ao banco para validação

### Contrato

**Comunicação Síncrona (Atual)**
- Validação de destinatário: Síncrona (dentro da transação)
- Processamento de transferência: Síncrona
- **Vantagem**: Resposta imediata ao usuário
- **Desvantagem**: Bloqueio em caso de falha de qualquer serviço dependente

**Comunicação Assíncrona (Futura - Escalabilidade)**
- Considerar fila de mensagens para processamento de transferências em alta escala
- Manter endpoint síncrono para casos de baixo volume
- Criar endpoint assíncrono alternativo para operações em lote

**Recomendação de Implementação Gradual:**
1. **Fase 1 (Atual)**: Tudo síncrono com transações de banco
2. **Fase 2**: Adicionar cache de status de usuário
3. **Fase 3**: Implementar circuit breaker para validações externas
4. **Fase 4**: Criar fila assíncrona para processamento em lote (opcional)
