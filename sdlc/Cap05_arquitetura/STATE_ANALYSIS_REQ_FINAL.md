# Análise de Estados (Heurística State Analysis) - REQ_FINAL

## 1. Identificação de Estados (Onde estou?)

### Estados do Usuário

**Estados Válidos:**
- **Ativo**: Usuário pode enviar e receber QualiPoints
- **Inativo**: Usuário não pode receber transferências (validação crítica)
- **Suspenso**: Estado intermediário (possível bloqueio temporário)

**Estados Finais:**
- Não há estados finais explícitos no requisito, mas estados como "Inativo" podem ser considerados finais para operações de transferência

**Estados Transitórios:**
- Mudança de "Ativo" para "Suspenso" pode ser reversível
- Mudança de "Ativo" para "Inativo" pode ser permanente ou temporária (requer definição de negócio)

### Estados da Transação

**Estados Válidos Identificados:**
- **Pendente**: Transação iniciada, aguardando processamento (implícito durante execução)
- **Processando**: Durante a execução da transação atômica
- **Concluída**: Transferência realizada com sucesso (status 201)
- **Falha**: Transação rejeitada por erro (402, 404, 422)

**Estados Finais:**
- **Concluída**: Não pode ser revertida automaticamente (requer operação manual de estorno)
- **Falha**: Estado final - transação não ocorreu

**Estados Transitórios:**
- **Pendente → Processando**: Durante validações e execução
- **Processando → Concluída**: Após sucesso da transação atômica
- **Processando → Falha**: Em caso de erro durante execução

### Estados da Carteira

**Estado Principal:**
- **Saldo Disponível**: Valor numérico que representa pontos disponíveis para transferência

**Invariante Crítico:**
- Saldo nunca pode ser negativo (garantido pela validação de valor máximo = saldo disponível)

## 2. Mapeamento de Transições (Como mudo?)

### Transições de Estado do Usuário

**Ativo → Inativo**
- **Gatilho**: Ação administrativa ou automática (fora do escopo deste requisito)
- **Condições**: Usuário deve existir e estar no estado "Ativo"
- **Ações**: Bloqueio de recebimento de transferências
- **Impacto**: Transações futuras para este usuário serão rejeitadas (404 ou validação de status)

**Ativo → Suspenso**
- **Gatilho**: Ação administrativa ou regra de negócio
- **Condições**: Usuário deve existir e estar no estado "Ativo"
- **Ações**: Bloqueio temporário de operações
- **Impacto**: Similar ao estado "Inativo" para efeitos de transferência

**Suspenso → Ativo**
- **Gatilho**: Reversão administrativa
- **Condições**: Usuário deve estar no estado "Suspenso"
- **Ações**: Restauração de capacidade de receber transferências

### Transições de Estado da Transação

**Pendente → Processando**
- **Gatilho**: Recebimento da requisição POST /api/v1/transactions
- **Condições**: 
  - JWT válido e `user_id` extraído
  - `recipient_id` e `amount` presentes no corpo
  - `idempotency-key` não processada anteriormente (se fornecida)
- **Ações**: 
  - Validação inicial de formato
  - Verificação de idempotência

**Processando → Concluída**
- **Gatilho**: Sucesso de todas as validações e execução da transação atômica
- **Condições**:
  - Saldo do remetente >= `amount`
  - `amount` >= 1 QualiPoint
  - Destinatário existe e está "Ativo"
  - Transação de banco de dados executada com sucesso
- **Ações**:
  - Débito no remetente (dentro da transação)
  - Crédito no destinatário (dentro da transação)
  - Registro da transação com `transaction_id`
  - Retorno de resposta 201 com `new_balance`

**Processando → Falha (402 Payment Required)**
- **Gatilho**: Saldo insuficiente detectado durante validação
- **Condições**: `amount` > saldo disponível do remetente
- **Ações**: Rollback automático (nenhuma alteração no banco)
- **Estado Final**: Transação não ocorreu

**Processando → Falha (404 Not Found)**
- **Gatilho**: Destinatário inexistente ou não encontrado
- **Condições**: `recipient_id` não existe no banco de dados
- **Ações**: Rollback automático (nenhuma alteração no banco)
- **Estado Final**: Transação não ocorreu

**Processando → Falha (422 Unprocessable Entity)**
- **Gatilho**: Validação de formato ou regra de negócio falhou
- **Condições**: 
  - `amount` < 1 QualiPoint
  - Formato inválido de dados
  - Destinatário não está "Ativo" (validação de status)
- **Ações**: Rollback automático (nenhuma alteração no banco)
- **Estado Final**: Transação não ocorreu

### Transições de Estado da Carteira

**Saldo Disponível → Saldo Atualizado**
- **Gatilho**: Transação concluída com sucesso
- **Condições**: Transição "Processando → Concluída" bem-sucedida
- **Ações**:
  - Remetente: `saldo_atual = saldo_atual - amount`
  - Destinatário: `saldo_atual = saldo_atual + amount`
- **Invariante**: Saldo nunca fica negativo (garantido pela validação prévia)

## 3. Validação de Invariantes (O que não pode acontecer?)

### Invariantes de Transição de Transação

**❌ Transição Inválida: Falha → Concluída**
- **Risco**: Tentativa de reutilizar uma transação falha como sucesso
- **Prevenção**: Transações com estado "Falha" são finais e não podem ser reprocessadas sem nova requisição

**❌ Transição Inválida: Concluída → Processando**
- **Risco**: Reprocessamento de transação já concluída
- **Prevenção**: Verificação de idempotência via `idempotency-key` previne duplicação

**❌ Transição Inválida: Cancelado → Concluído**
- **Risco**: Estados zumbis (não aplicável ao requisito atual, mas importante para evolução futura)
- **Recomendação**: Se implementar estado "Cancelado", garantir que seja final e não permitir transição para "Concluído"

### Invariantes de Estado do Usuário

**❌ Transição Inválida Durante Processamento**
- **Risco**: Status do destinatário mudar de "Ativo" para "Inativo" entre validação e execução
- **Prevenção**: Validar status dentro da mesma transação de banco de dados que executa a transferência (lock de linha)

**❌ Estado Mutuamente Exclusivo**
- **Risco**: Usuário estar simultaneamente "Ativo" e "Inativo"
- **Prevenção**: Campo de status único com valores enumerados, sem estados compostos

### Invariantes de Carteira

**❌ Saldo Negativo**
- **Risco**: Transferência criar saldo negativo no remetente
- **Prevenção**: Validação de `amount <= saldo_disponível` antes da transação atômica

**❌ Saldo Inconsistente**
- **Risco**: Falha parcial deixar saldo inconsistente (débito sem crédito ou vice-versa)
- **Prevenção**: Transação atômica de banco de dados garante tudo ou nada (rollback automático)

### Estados Órfãos Identificados

**Estado "Suspenso"**
- **Risco**: Estado mencionado mas sem transições claras definidas no requisito
- **Recomendação**: Definir explicitamente:
  - Como usuário entra em "Suspenso"?
  - Como usuário sai de "Suspenso"?
  - "Suspenso" bloqueia envio e recebimento ou apenas recebimento?

**Estado "Pendente" da Transação**
- **Risco**: Se sistema crashar durante "Pendente", transação pode ficar órfã
- **Recomendação**: Implementar timeout e mecanismo de limpeza de transações pendentes antigas

## 4. Lente da Persistência

### Armazenamento de Estados

**Estado do Usuário (Status)**
- **Persistência**: Banco de dados (campo `status` na tabela `users`)
- **Risco de Perda**: Baixo (persistido em banco transacional)
- **Recuperação**: Sistema recupera estado do banco ao iniciar

**Estado da Transação**
- **Persistência**: Banco de dados (tabela `transactions` com campos: `transaction_id`, `sender_id`, `recipient_id`, `amount`, `status`, `timestamp`)
- **Risco de Perda**: Crítico durante transição "Processando"
- **Recuperação**: 
  - Se crash ocorrer antes de commit: rollback automático (nenhuma alteração)
  - Se crash ocorrer após commit: transação já está persistida como "Concluída"

**Estado da Carteira (Saldo)**
- **Persistência**: Banco de dados (campo `balance` na tabela `wallets` ou `carteiras`)
- **Risco de Perda**: Crítico - saldo é estado crítico do sistema
- **Recuperação**: 
  - Saldo sempre refletido no banco (não apenas em memória)
  - Transações atômicas garantem consistência

**Idempotency Key**
- **Persistência**: Armazenamento externo recomendado (ex: Redis) com TTL
- **Risco de Perda**: Se perder, permite duplicação de transações
- **Recuperação**: 
  - Redis com persistência habilitada reduz risco
  - TTL adequado (ex: 24 horas) limpa chaves antigas automaticamente

### Mecanismos de Recuperação

**Cenário 1: Crash Durante Processamento (Antes do Commit)**
- **Estado**: Transação em "Processando", mas não commitada
- **Recuperação**: Rollback automático do banco de dados
- **Resultado**: Nenhuma alteração persistida, transação não ocorreu
- **Ação do Sistema**: Usuário pode tentar novamente (idempotência previne duplicação)

**Cenário 2: Crash Após Commit Parcial**
- **Estado**: Débito realizado, mas crédito não commitado (improvável com transação atômica)
- **Recuperação**: Transação atômica previne este cenário
- **Garantia**: Banco de dados garante tudo ou nada

**Cenário 3: Crash Após Commit Completo**
- **Estado**: Transação "Concluída" e persistida
- **Recuperação**: Estado já está correto no banco
- **Ação do Sistema**: Nenhuma ação necessária, estado já está consistente

**Cenário 4: Perda de Idempotency Key**
- **Estado**: Chave de idempotência perdida (Redis down ou expirada)
- **Risco**: Permite duplicação se usuário retentar requisição
- **Mitigação**: 
  - Verificação adicional no banco: buscar transações recentes com mesmo `sender_id`, `recipient_id`, `amount` e timestamp próximo
  - Implementar janela de tempo para detectar duplicatas mesmo sem chave

### Mecanismo de Reconciliação

**Recomendação: Job de Reconciliação Periódico**
- **Objetivo**: Detectar e corrigir inconsistências de estado
- **Frequência**: Diária ou semanal
- **Verificações**:
  1. Soma de todas as transações de crédito - débitos deve igualar saldo atual
  2. Transações "Processando" há mais de X minutos devem ser investigadas
  3. Transações sem estado definido devem ser marcadas como "Falha"

**Recomendação: Log de Auditoria**
- **Objetivo**: Rastreabilidade completa de mudanças de estado
- **Implementação**: Tabela de auditoria registrando:
  - Estado anterior e novo estado
  - Timestamp da mudança
  - Usuário/sistema que causou a mudança
  - Contexto da operação (transaction_id, etc.)

## 5. Diagrama de Estados

### Diagrama de Transição de Transação

```
[Pendente] 
    |
    | (Requisição recebida, validações iniciais OK)
    v
[Processando]
    |
    |-- (Validações OK + Transação DB sucesso) --> [Concluída] (FINAL)
    |
    |-- (Saldo insuficiente) --> [Falha: 402] (FINAL)
    |
    |-- (Destinatário não existe) --> [Falha: 404] (FINAL)
    |
    |-- (Validação de formato/regra) --> [Falha: 422] (FINAL)
```

### Tabela de Transições Válidas

| Estado Origem | Estado Destino | Gatilho | Condições | Ações |
|---------------|----------------|---------|-----------|-------|
| Pendente | Processando | POST /api/v1/transactions | JWT válido, dados presentes, idempotência OK | Iniciar validações |
| Processando | Concluída | Validações OK | Saldo suficiente, destinatário ativo, transação DB OK | Débito + Crédito + Commit |
| Processando | Falha (402) | Validação de saldo | Saldo < amount | Rollback |
| Processando | Falha (404) | Validação de destinatário | Destinatário não existe | Rollback |
| Processando | Falha (422) | Validação de formato/regra | amount < 1 ou destinatário inativo | Rollback |

## 6. Identificação de Riscos de Estados Inconsistentes

### Risco Crítico 1: Race Condition em Validação de Status
- **Cenário**: Status do destinatário muda de "Ativo" para "Inativo" entre validação e execução
- **Impacto**: Transferência para usuário inativo é processada incorretamente
- **Mitigação**: Validar status dentro da mesma transação de banco com lock de linha (SELECT FOR UPDATE)

### Risco Crítico 2: Race Condition em Saldo
- **Cenário**: Múltiplas transferências simultâneas do mesmo remetente
- **Impacto**: Saldo pode ficar negativo se validações ocorrerem em paralelo
- **Mitigação**: Lock de linha na carteira do remetente durante toda a transação

### Risco Médio 3: Transação Órfã em "Processando"
- **Cenário**: Sistema crasha durante processamento, transação fica em estado indefinido
- **Impacto**: Transação pode ficar travada indefinidamente
- **Mitigação**: 
  - Timeout de transação no banco de dados
  - Job de limpeza que marca transações "Processando" antigas como "Falha"

### Risco Médio 4: Perda de Idempotency Key
- **Cenário**: Redis falha ou chave expira antes do processamento completo
- **Impacto**: Permite duplicação de transações em retentativas
- **Mitigação**: 
  - Verificação adicional no banco por duplicatas recentes
  - Persistência de Redis habilitada
  - TTL adequado (não muito curto)

### Risco Baixo 5: Estado "Suspenso" Indefinido
- **Cenário**: Estado mencionado mas sem regras claras de transição
- **Impacto**: Comportamento inconsistente do sistema
- **Mitigação**: Definir explicitamente regras de negócio para estado "Suspenso"

## 7. Recomendações de Heurísticas Complementares

**Para validar operações CRUD sobre estados de usuário, aplique a heurística CRUD:**
- Garantir que mudanças de status sejam auditadas
- Validar que exclusão de usuário tenha tratamento adequado para transações pendentes

**Para validar limites de transições, aplique a heurística Count (0, 1, Muitos):**
- Validar comportamento quando usuário tem 0 transações (primeira transferência)
- Validar comportamento quando usuário tem muitas transações simultâneas
- Validar limites de transições por período (rate limiting)

**Para garantir integridade temporal, aplique a heurística Time (Antes, Durante, Depois):**
- Validar que validação de status ocorra "Durante" a transação (não antes)
- Validar timeout adequado para transações "Processando"
- Validar TTL de idempotency keys

**Para investigar casos extremos de estado, aplique a heurística Selection (Alguns, Nenhum, Todos):**
- Validar comportamento quando nenhum usuário está "Ativo" (cenário de manutenção)
- Validar comportamento quando todos os usuários estão "Ativo" (cenário normal)
- Validar filtros de status em listagens e buscas

## 8. Objetivo Final Alcançado

Este relatório fornece um mapa completo do ciclo de vida dos estados no sistema de transferência de QualiPoints, identificando:

✅ **Todos os estados válidos** (Usuário, Transação, Carteira)
✅ **Todas as transições possíveis** com gatilhos, condições e ações
✅ **Invariantes críticas** que previnem estados inconsistentes
✅ **Mecanismos de persistência** e recuperação de falhas
✅ **Riscos identificados** com mitigações propostas
✅ **Recomendações** de heurísticas complementares para análise aprofundada

O sistema está arquitetado para garantir **Integridade de Fluxo** através de transações atômicas e validações dentro da mesma transação, prevenindo estados inconsistentes que gerariam necessidade de suporte manual caro.
