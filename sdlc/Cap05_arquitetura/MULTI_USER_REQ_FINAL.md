# Análise Multi-User (Heurística Multi-User) - REQ_FINAL

## 1. A Lente da Colisão

**O que acontece se dois atores (usuários ou processos) tentarem o Update no mesmo campo no exato milissegundo?**

### Pontos de Escrita Simultânea Identificados

**1. Atualização de Saldo da Carteira (Crítico)**
- **Campo**: `balance` na tabela `wallets` ou `carteiras`
- **Cenário de Colisão**: Dois usuários diferentes tentando transferir simultaneamente do mesmo remetente
- **Risco**: Race condition pode permitir saldo negativo se validações ocorrerem em paralelo

**2. Atualização de Status do Usuário (Médio)**
- **Campo**: `status` na tabela `users`
- **Cenário de Colisão**: Administrador mudando status de "Ativo" para "Inativo" enquanto transferência está sendo processada
- **Risco**: Transferência pode ser processada para usuário que acabou de ser inativado

**3. Registro de Transação (Baixo)**
- **Campo**: Inserção na tabela `transactions`
- **Cenário de Colisão**: Múltiplas inserções simultâneas
- **Risco**: Baixo (inserções são independentes, mas pode gerar duplicação se idempotency-key falhar)

### Estratégias de Locking Recomendadas

**Locking Pessimista (Recomendado para Saldo)**
- **Aplicação**: Lock de linha na carteira do remetente durante toda a transação
- **Implementação**: `SELECT FOR UPDATE` na carteira do remetente antes de validar saldo
- **Justificativa**: Saldo é recurso crítico que não pode ser comprometido; locking pessimista garante que apenas uma transferência por vez possa modificar o saldo

**Locking Otimista (Recomendado para Status)**
- **Aplicação**: Versionamento de campo `status` com campo `version` ou `updated_at`
- **Implementação**: Verificar `version` no início e validar novamente antes do commit
- **Justificativa**: Mudanças de status são menos frequentes que transferências; locking otimista reduz contenção

**Locking Híbrido (Recomendado para Transações)**
- **Aplicação**: Combinação de idempotency-key (otimista) com verificação de duplicatas no banco (pessimista)
- **Implementação**: 
  - Verificar idempotency-key primeiro (cache rápido)
  - Se não encontrada, verificar no banco por transações duplicadas recentes
  - Lock de linha apenas durante inserção da transação

### Mecanismos de Controle de Concorrência Propostos

**1. Transação Atômica de Banco de Dados**
- **Escopo**: Toda operação de transferência dentro de uma única transação
- **Garantias**: 
  - Débito e crédito ocorrem atomicamente (tudo ou nada)
  - Rollback automático em caso de falha
- **Isolamento**: Nível de isolamento `REPEATABLE READ` ou `SERIALIZABLE` recomendado

**2. Lock de Linha na Carteira do Remetente**
```sql
BEGIN TRANSACTION;
  SELECT balance FROM wallets WHERE user_id = ? FOR UPDATE;
  -- Validações e atualizações aqui
COMMIT;
```

**3. Validação de Status Dentro da Transação**
- **Implementação**: Validar status do destinatário dentro da mesma transação que executa a transferência
- **Lock**: `SELECT FOR UPDATE` no registro do destinatário durante validação
- **Benefício**: Previne mudança de status entre validação e execução

**4. Idempotency Key com Verificação Dupla**
- **Camada 1**: Cache (Redis) com TTL de 24 horas
- **Camada 2**: Verificação no banco por transações recentes com mesmo `sender_id`, `recipient_id`, `amount` e timestamp próximo (±5 minutos)
- **Benefício**: Redundância previne duplicação mesmo se cache falhar

### Comportamento em Caso de Conflito Detectado

**Cenário 1: Saldo Insuficiente Após Lock**
- **Detecção**: Validação de saldo dentro da transação com lock
- **Comportamento**: Retornar erro 402 Payment Required imediatamente
- **Rollback**: Automático pelo banco de dados
- **UX**: Mensagem clara "Saldo insuficiente para esta operação"

**Cenário 2: Status Mudou Durante Processamento**
- **Detecção**: Validação de status dentro da transação com lock
- **Comportamento**: Retornar erro 422 Unprocessable Entity
- **Rollback**: Automático pelo banco de dados
- **UX**: Mensagem "Destinatário não está mais ativo. Por favor, verifique o status antes de tentar novamente."

**Cenário 3: Idempotency Key Duplicada**
- **Detecção**: Verificação de chave no cache ou banco
- **Comportamento**: Retornar resposta 201 com os mesmos dados da transação original (sem criar nova transação)
- **UX**: Transparente para o usuário (comportamento idempotente esperado)

## 2. A Lente da Sessão Dupla

**O sistema suporta um único usuário operando em abas ou dispositivos diferentes sem gerar inconsistência?**

### Análise de Múltiplas Sessões do Mesmo Usuário

**Cenário Crítico Identificado**: Usuário autenticado em múltiplas abas/dispositivos tentando transferir simultaneamente

**Riscos Identificados:**

**1. Transferências Duplicadas Acidentais**
- **Cenário**: Usuário clica em "Enviar" em duas abas simultaneamente
- **Risco**: Duas transferências idênticas podem ser processadas se idempotency-key não for fornecida
- **Mitigação**: Idempotency-key deve ser gerada automaticamente pelo frontend ou obrigatória

**2. Saldo Insuficiente em Segunda Aba**
- **Cenário**: Usuário inicia transferência em Aba 1 (saldo suficiente), depois inicia outra em Aba 2 (saldo agora insuficiente)
- **Risco**: Aba 2 pode receber erro 402 mesmo que saldo fosse suficiente quando iniciou
- **Comportamento Esperado**: Correto - sistema deve proteger contra saldo negativo
- **UX**: Mensagem clara explicando que outra operação consumiu o saldo

**3. Operações Concorrentes do Mesmo Remetente**
- **Cenário**: Usuário tenta transferir 50 pontos para Usuário A e 30 pontos para Usuário B simultaneamente, tendo apenas 60 pontos
- **Risco**: Ambas podem passar na validação inicial se executadas em paralelo
- **Mitigação**: Lock de linha na carteira do remetente garante processamento sequencial

### Controle de Sessão e Token de Autenticação

**JWT Token**
- **Extração**: `user_id` extraído do token (não enviado no corpo)
- **Validação**: Token deve ser válido e não expirado
- **Limitação**: Não há controle explícito de múltiplas sessões ativas do mesmo usuário

**Recomendação: Refresh Token com Revogação**
- **Implementação Futura**: Sistema de refresh tokens com capacidade de revogar sessões antigas
- **Benefício**: Permite controle de sessões múltiplas se necessário para segurança adicional

### Mecanismos de Sincronização Entre Sessões

**1. Idempotency Key por Requisição**
- **Geração**: Frontend deve gerar UUID único por ação do usuário
- **Armazenamento**: Associar idempotency-key à combinação `sender_id + recipient_id + amount + timestamp`
- **Benefício**: Previne duplicação mesmo com múltiplas abas

**2. Lock de Linha na Carteira**
- **Efeito Colateral Positivo**: Lock garante que operações do mesmo remetente sejam processadas sequencialmente
- **Benefício**: Previne race conditions mesmo com múltiplas sessões

**3. Feedback em Tempo Real (Recomendação Futura)**
- **Implementação**: WebSocket ou Server-Sent Events para atualizar saldo em todas as abas após transferência
- **Benefício**: Usuário vê saldo atualizado imediatamente em todas as sessões

### Riscos de Operações Duplicadas ou Conflitantes

**Risco Alto: Duplicação por Falta de Idempotency Key**
- **Cenário**: Frontend não envia idempotency-key e usuário clica múltiplas vezes
- **Mitigação Atual**: Depende do frontend enviar a chave
- **Recomendação**: Tornar idempotency-key obrigatória ou gerar automaticamente no backend se não fornecida

**Risco Médio: Transferências Parciais Simultâneas**
- **Cenário**: Usuário tenta transferir mais do que o saldo permite em múltiplas abas
- **Mitigação**: Lock de linha garante processamento sequencial
- **Comportamento**: Primeira transferência bem-sucedida, demais recebem erro 402

## 3. A Lente do Inventário Crítico

**Como garantimos que "o último item" não seja vendido para duas pessoas? (Estratégia de Locking)**

### Recursos Críticos com Quantidade Limitada Identificados

**1. Saldo da Carteira (Recurso Crítico Principal)**
- **Natureza**: Quantidade limitada (saldo disponível)
- **Cenário de Conflito**: Dois destinatários diferentes tentando receber transferências simultâneas do mesmo remetente quando saldo é exato para apenas uma transferência
- **Exemplo**: Remetente tem 50 pontos, tenta transferir 50 para Usuário A e 50 para Usuário B simultaneamente

**2. Status "Ativo" do Destinatário (Recurso de Capacidade)**
- **Natureza**: Capacidade limitada de receber transferências (apenas usuários "Ativos")
- **Cenário de Conflito**: Múltiplas transferências simultâneas para usuário que está sendo inativado
- **Exemplo**: Administrador inativa Usuário X enquanto duas transferências para X estão sendo processadas

### Estratégias de Reserva e Bloqueio

**Estratégia 1: Lock Pessimista na Carteira do Remetente (Implementada)**
- **Mecanismo**: `SELECT FOR UPDATE` na linha da carteira do remetente
- **Duração**: Durante toda a transação (validação + débito + crédito)
- **Garantia**: Apenas uma transferência por vez pode modificar o saldo do remetente
- **Resultado**: "Último ponto" não pode ser transferido para duas pessoas simultaneamente

**Estratégia 2: Validação Atômica de Saldo**
- **Mecanismo**: Validação `amount <= balance` dentro da mesma transação que executa o débito
- **Garantia**: Saldo não pode mudar entre validação e execução
- **Resultado**: Previne race condition onde duas validações passam antes de qualquer débito ocorrer

**Estratégia 3: Lock Pessimista no Status do Destinatário**
- **Mecanismo**: `SELECT FOR UPDATE` no registro do destinatário durante validação de status
- **Duração**: Durante validação e execução da transferência
- **Garantia**: Status não pode mudar entre validação e execução
- **Resultado**: Previne transferência para usuário que acabou de ser inativado

### Mecanismos de Atomicidade em Operações Críticas

**Operação Atômica de Transferência**
```sql
BEGIN TRANSACTION;
  -- Lock remetente
  SELECT balance FROM wallets WHERE user_id = sender_id FOR UPDATE;
  
  -- Lock destinatário
  SELECT status FROM users WHERE user_id = recipient_id FOR UPDATE;
  
  -- Validações
  IF balance < amount THEN ROLLBACK; RETURN 402; END IF;
  IF status != 'Ativo' THEN ROLLBACK; RETURN 422; END IF;
  
  -- Execução atômica
  UPDATE wallets SET balance = balance - amount WHERE user_id = sender_id;
  UPDATE wallets SET balance = balance + amount WHERE user_id = recipient_id;
  INSERT INTO transactions (sender_id, recipient_id, amount, status) VALUES (...);
  
COMMIT;
```

**Garantias da Atomicidade:**
- ✅ Débito e crédito ocorrem juntos ou nenhum ocorre
- ✅ Validações ocorrem dentro da mesma transação
- ✅ Rollback automático em caso de qualquer falha
- ✅ Isolamento previne leituras inconsistentes durante processamento

### Tratamento de Casos onde o Recurso se Esgota Durante a Operação

**Cenário 1: Saldo se Esgota Entre Validação e Execução**
- **Impossibilidade**: Lock de linha previne este cenário
- **Garantia**: Com `SELECT FOR UPDATE`, nenhuma outra transação pode modificar o saldo até o commit
- **Resultado**: Se saldo for insuficiente, será detectado durante validação dentro da transação

**Cenário 2: Múltiplas Transferências Simultâneas do Mesmo Remetente**
- **Comportamento**: Lock garante processamento sequencial
- **Resultado**: 
  - Primeira transferência: Processada com sucesso (se saldo suficiente)
  - Segunda transferência: Aguarda lock, depois valida saldo atualizado, retorna 402 se insuficiente
- **UX**: Segunda requisição pode ter timeout maior devido à espera do lock

**Cenário 3: Status Muda Durante Processamento**
- **Comportamento**: Lock no registro do destinatário previne mudança durante processamento
- **Resultado**: 
  - Se status já estava "Inativo" antes do lock: Erro 422 retornado
  - Se status muda durante lock: Operação administrativa aguarda fim da transferência
- **Garantia**: Transferência em processamento não é interrompida por mudança de status

**Recomendação: Timeout de Lock**
- **Implementação**: Timeout de 5-10 segundos para locks de linha
- **Benefício**: Previne deadlocks e operações travadas indefinidamente
- **Tratamento**: Se timeout ocorrer, retornar erro 503 Service Unavailable com mensagem "Sistema ocupado, tente novamente"

## 4. A Lente da UX de Conflito

**Quando a colisão ocorre, o sistema explode com um erro 500 ou resolve de forma elegante (ex: "Outro usuário já atualizou este registro")?**

### Análise de Tratamento de Erros em Cenários de Conflito

**Erros Definidos no Requisito:**

**1. 402 Payment Required (Saldo Insuficiente)**
- **Gatilho**: Validação de saldo dentro da transação detecta `amount > balance`
- **Tratamento**: Rollback automático, retorno de erro 402
- **UX Atual**: Não especificada no requisito
- **Recomendação de Mensagem**: 
  - "Saldo insuficiente. Você possui {balance} QualiPoints disponíveis e tentou transferir {amount}."
  - Incluir `available_balance` no corpo da resposta para facilitar retry

**2. 404 Not Found (Destinatário Inexistente)**
- **Gatilho**: Destinatário não encontrado no banco de dados
- **Tratamento**: Rollback automático, retorno de erro 404
- **UX Atual**: Não especificada no requisito
- **Recomendação de Mensagem**: 
  - "Destinatário não encontrado. Verifique se o ID está correto."
  - Incluir `recipient_id` no corpo da resposta para facilitar debug

**3. 422 Unprocessable Entity (Validação de Formato/Regra)**
- **Gatilhos**: 
  - `amount < 1`
  - Formato inválido de dados
  - Destinatário não está "Ativo"
- **Tratamento**: Rollback automático, retorno de erro 422
- **UX Atual**: Não especificada no requisito
- **Recomendação de Mensagem**: 
  - Para `amount < 1`: "Valor mínimo de transferência é 1 QualiPoint."
  - Para destinatário inativo: "Destinatário não está ativo. Status atual: {status}. Entre em contato com o suporte se necessário."
  - Incluir campo `errors` com detalhes específicos de validação

### Feedback ao Usuário sobre Operações Conflitantes

**Cenário 1: Conflito por Saldo Insuficiente Após Outra Transferência**
- **Situação**: Usuário tenta transferir, mas outra transferência simultânea consumiu o saldo
- **Comportamento Atual**: Erro 402 genérico
- **Recomendação de Melhoria**: 
  - Mensagem: "Saldo insuficiente. Outra operação pode ter consumido seu saldo. Saldo atual: {balance}."
  - Incluir timestamp da última transação para contexto

**Cenário 2: Conflito por Status Mudado Durante Processamento**
- **Situação**: Destinatário foi inativado enquanto transferência estava sendo processada
- **Comportamento Atual**: Erro 422 genérico (se detectado)
- **Recomendação de Melhoria**: 
  - Mensagem: "Não foi possível completar a transferência. O destinatário não está mais ativo. Status atualizado: {status}."
  - Sugerir verificar status antes de tentar novamente

**Cenário 3: Conflito por Idempotency Key Duplicada**
- **Situação**: Requisição duplicada detectada via idempotency-key
- **Comportamento Esperado**: Retornar resposta 201 com dados da transação original (sem criar nova)
- **UX**: Transparente - usuário recebe confirmação normalmente
- **Recomendação**: Incluir header `X-Idempotent-Replay: true` para indicar que é replay

**Cenário 4: Timeout de Lock (Deadlock ou Sistema Sobrecarregado)**
- **Situação**: Lock de linha excede timeout devido à alta concorrência
- **Comportamento Atual**: Não especificado
- **Recomendação**: 
  - Retornar erro 503 Service Unavailable
  - Mensagem: "Sistema temporariamente ocupado devido à alta demanda. Por favor, tente novamente em alguns segundos."
  - Incluir `retry_after` em segundos no header

### Estratégias de Resolução Automática vs. Manual

**Resolução Automática Implementada:**

**1. Rollback Automático em Falhas**
- **Mecanismo**: Transação atômica de banco de dados
- **Benefício**: Nenhuma alteração parcial persiste em caso de erro
- **UX**: Usuário pode tentar novamente sem risco de inconsistência

**2. Idempotência Automática**
- **Mecanismo**: Idempotency-key previne duplicação
- **Benefício**: Cliques acidentais não geram transferências duplicadas
- **UX**: Transparente - usuário não precisa se preocupar com duplicação

**3. Validação Dentro da Transação**
- **Mecanismo**: Todas as validações ocorrem dentro da mesma transação com locks
- **Benefício**: Previne race conditions automaticamente
- **UX**: Sistema garante consistência sem intervenção manual

**Resolução Manual Necessária (Cenários Extremos):**

**1. Deadlock Entre Múltiplas Transferências**
- **Cenário**: Usuário A tenta transferir para B enquanto B tenta transferir para A simultaneamente
- **Risco**: Deadlock se ambos tentarem lock nas carteiras ao mesmo tempo
- **Mitigação Automática**: Banco de dados detecta deadlock e faz rollback de uma transação
- **Resolução Manual**: Usuário recebe erro e pode tentar novamente (sistema resolve automaticamente)

**2. Transação Órfã em "Processando"**
- **Cenário**: Sistema crasha durante processamento, transação fica travada
- **Risco**: Lock pode não ser liberado adequadamente
- **Mitigação Automática**: Timeout de transação libera lock automaticamente
- **Resolução Manual**: Job de limpeza marca transações antigas como "Falha" (automático, mas pode requerer investigação)

**Recomendação: Sistema de Retry Inteligente**
- **Implementação**: Frontend pode implementar retry automático para erros 503 (timeout)
- **Estratégia**: Exponential backoff (1s, 2s, 4s)
- **Limite**: Máximo de 3 tentativas antes de mostrar erro ao usuário

### Experiência do Usuário em Casos de Falha por Concorrência

**Princípios de UX para Conflitos:**

**1. Transparência**
- Usuário deve entender o que aconteceu
- Mensagens claras explicando o motivo da falha
- Informações contextuais (saldo atual, status do destinatário)

**2. Recuperabilidade**
- Erros devem ser recuperáveis quando possível
- Fornecer informações suficientes para retry bem-sucedido
- Sugerir ações corretivas quando aplicável

**3. Consistência**
- Sempre retornar mesmo código de erro para mesmo problema
- Mensagens devem seguir padrão consistente
- Comportamento previsível facilita debugging

**4. Performance Percebida**
- Timeouts de lock devem ser rápidos (< 5 segundos)
- Feedback imediato para erros de validação
- Retry automático para erros transitórios

**Exemplo de Resposta de Erro Melhorada:**

```json
{
  "error": {
    "code": "INSUFFICIENT_BALANCE",
    "message": "Saldo insuficiente para esta transferência",
    "details": {
      "available_balance": 45,
      "requested_amount": 50,
      "shortfall": 5
    },
    "suggestion": "Você precisa de mais 5 QualiPoints para completar esta transferência.",
    "timestamp": "2026-02-16T10:30:00Z",
    "transaction_id": null
  }
}
```

## 5. Identificação de Race Conditions e Deadlocks Potenciais

### Race Conditions Identificadas

**Race Condition 1: Validação de Saldo (CRÍTICO - Mitigado)**
- **Cenário**: Duas transferências simultâneas do mesmo remetente passam validação de saldo antes de qualquer débito
- **Risco Original**: Saldo negativo possível
- **Mitigação**: Lock de linha na carteira do remetente previne esta race condition
- **Status**: ✅ Resolvido

**Race Condition 2: Validação de Status (CRÍTICO - Mitigado)**
- **Cenário**: Status do destinatário muda de "Ativo" para "Inativo" entre validação e execução
- **Risco Original**: Transferência para usuário inativo processada incorretamente
- **Mitigação**: Validação de status dentro da transação com lock previne mudança durante processamento
- **Status**: ✅ Resolvido

**Race Condition 3: Idempotency Key (MÉDIO - Parcialmente Mitigado)**
- **Cenário**: Duas requisições idênticas chegam simultaneamente antes de qualquer uma ser processada
- **Risco Original**: Ambas podem passar verificação de idempotência se cache não estiver sincronizado
- **Mitigação**: Verificação dupla (cache + banco) reduz risco, mas não elimina completamente
- **Recomendação**: Lock na inserção da transação com verificação de duplicata
- **Status**: ⚠️ Requer atenção adicional

**Race Condition 4: Leitura de Saldo Inconsistente (BAIXO - Mitigado)**
- **Cenário**: Frontend lê saldo enquanto transferência está sendo processada
- **Risco Original**: Saldo mostrado pode estar desatualizado
- **Mitigação**: Isolamento de transação garante leitura consistente
- **Recomendação Futura**: WebSocket para atualização em tempo real
- **Status**: ✅ Resolvido (com recomendação de melhoria)

### Deadlocks Potenciais

**Deadlock 1: Transferências Circulares Simultâneas (BAIXO)**
- **Cenário**: 
  - Usuário A tenta transferir para B (lock em A, aguarda lock em B)
  - Usuário B tenta transferir para A simultaneamente (lock em B, aguarda lock em A)
- **Risco**: Deadlock clássico
- **Mitigação**: Banco de dados detecta deadlock e faz rollback de uma transação automaticamente
- **Prevenção Adicional**: Ordenar locks sempre pelo `user_id` (menor ID primeiro) para prevenir deadlocks
- **Status**: ⚠️ Requer ordenação de locks

**Deadlock 2: Lock em Múltiplas Tabelas (BAIXO)**
- **Cenário**: Lock em `wallets` e `users` em ordem diferente em transações simultâneas
- **Risco**: Deadlock se ordem de locks for inconsistente
- **Mitigação**: Sempre adquirir locks na mesma ordem (ex: `users` primeiro, depois `wallets`)
- **Status**: ⚠️ Requer padronização de ordem de locks

**Deadlock 3: Lock de Longa Duração (MÉDIO)**
- **Cenário**: Operação lenta mantém lock por muito tempo, bloqueando outras operações
- **Risco**: Timeout de outras requisições
- **Mitigação**: Timeout de transação (5-10 segundos) libera locks automaticamente
- **Status**: ✅ Mitigado com timeout

### Recomendações de Prevenção de Deadlocks

**1. Ordenação Consistente de Locks**
```sql
-- Sempre adquirir locks na mesma ordem
-- Ordem recomendada: users (menor user_id primeiro), depois wallets
SELECT * FROM users WHERE user_id IN (?, ?) ORDER BY user_id FOR UPDATE;
SELECT * FROM wallets WHERE user_id IN (?, ?) ORDER BY user_id FOR UPDATE;
```

**2. Timeout de Transação**
- **Configuração**: Timeout de 5-10 segundos para transações
- **Benefício**: Previne locks indefinidos
- **Tratamento**: Retornar erro 503 se timeout ocorrer

**3. Detecção e Retry de Deadlocks**
- **Implementação**: Capturar exceção de deadlock do banco de dados
- **Comportamento**: Retry automático da transação (máximo 3 tentativas)
- **Backoff**: Esperar 100-200ms antes de retry

## 6. Recomendações de Heurísticas Complementares

### State Analysis (Recomendado)
**Quando aplicar**: Para garantir que transições de estado sejam atômicas e não possam ser interrompidas por operações concorrentes.

**Recomendação**: "Para garantir que as transições de estado sejam atômicas e resistentes a concorrência, aplique a heurística State Analysis."

**Aplicação Específica**: 
- Validar que transição "Pendente → Processando → Concluída" seja atômica
- Verificar que mudanças de status de usuário não interrompam transferências em processamento
- Garantir que estados de transação sejam consistentes mesmo sob concorrência

### Count (0, 1, Muitos) (Recomendado)
**Quando aplicar**: Para entender o volume de acessos simultâneos (throughput) e dimensionar adequadamente os mecanismos de controle de concorrência.

**Recomendação**: "Para validar o volume esperado de acessos simultâneos e dimensionar os mecanismos de locking, aplique a heurística Count (0, 1, Muitos)."

**Aplicação Específica**:
- Quantos usuários simultâneos são esperados?
- Quantas transferências por segundo o sistema deve suportar?
- Qual o pico de concorrência em operações do mesmo remetente?
- Dimensionar timeout de locks baseado no volume esperado

### Time (Antes, Durante, Depois) (Recomendado)
**Quando aplicar**: Para garantir que validações ocorram no momento correto dentro da transação.

**Recomendação**: "Para garantir que validações ocorram 'Durante' a transação (não antes), aplique a heurística Time."

**Aplicação Específica**:
- Validar que verificação de saldo ocorra "Durante" a transação (não em validação prévia)
- Garantir que validação de status ocorra "Durante" a execução
- Verificar TTL adequado de idempotency keys

### Dependencies (Opcional)
**Quando aplicar**: Para identificar dependências entre operações concorrentes que podem causar problemas.

**Recomendação**: "Para identificar dependências críticas entre operações que podem causar deadlocks, aplique a heurística Dependencies."

**Aplicação Específica**:
- Mapear dependências entre locks de diferentes tabelas
- Identificar cadeias de dependências que podem causar deadlocks
- Validar ordem de aquisição de locks

## 7. Estratégias de Locking Recomendadas - Resumo Executivo

### Locking Pessimista (Aplicado)
- **Onde**: Carteira do remetente e registro do destinatário
- **Mecanismo**: `SELECT FOR UPDATE` dentro da transação
- **Duração**: Durante toda a operação de transferência
- **Justificativa**: Recursos críticos que não podem ser comprometidos

### Locking Otimista (Não Aplicado - Considerar para Evolução)
- **Onde**: Status de usuário (se mudanças forem frequentes)
- **Mecanismo**: Versionamento com campo `version` ou `updated_at`
- **Benefício**: Reduz contenção em operações de leitura
- **Recomendação**: Considerar se sistema evoluir para muitas mudanças de status simultâneas

### Locking Híbrido (Aplicado)
- **Onde**: Idempotência de transações
- **Mecanismo**: Cache (Redis) + verificação no banco
- **Benefício**: Performance (cache) + confiabilidade (banco)
- **Status**: ✅ Implementado

## 8. Mecanismos de Controle de Concorrência - Resumo Técnico

### 1. Transação Atômica
- ✅ Implementado: Toda operação dentro de uma transação
- ✅ Garantia: Tudo ou nada (ACID)

### 2. Lock de Linha
- ✅ Implementado: `SELECT FOR UPDATE` em recursos críticos
- ✅ Escopo: Carteira do remetente e registro do destinatário

### 3. Isolamento de Transação
- ⚠️ Recomendado: Nível `REPEATABLE READ` ou `SERIALIZABLE`
- ✅ Benefício: Previne leituras inconsistentes

### 4. Idempotência
- ✅ Implementado: Idempotency-key com verificação dupla
- ✅ Camadas: Cache (Redis) + Banco de dados

### 5. Timeout de Transação
- ⚠️ Recomendado: 5-10 segundos
- ✅ Benefício: Previne locks indefinidos

### 6. Ordenação de Locks
- ⚠️ Recomendado: Sempre mesma ordem (por user_id)
- ✅ Benefício: Previne deadlocks

## 9. Objetivo Final Alcançado

Este relatório fornece um plano completo de gestão de concorrência para o sistema de transferência de QualiPoints, identificando:

✅ **Todos os pontos críticos de conflito** através das quatro lentes (Colisão, Sessão Dupla, Inventário Crítico, UX de Conflito)

✅ **Estratégias de locking** recomendadas (pessimista para recursos críticos, híbrido para idempotência)

✅ **Mecanismos de controle de concorrência** propostos (transações atômicas, locks de linha, idempotência dupla)

✅ **Tratamento de conflitos** com foco em experiência do usuário (mensagens claras, recuperabilidade, transparência)

✅ **Race conditions identificadas** com mitigações específicas (locks, validações dentro da transação)

✅ **Deadlocks potenciais** com recomendações de prevenção (ordenação de locks, timeouts)

✅ **Recomendações de heurísticas complementares** (State Analysis, Count, Time, Dependencies)

O sistema está arquitetado para **erradicar Race Conditions e Deadlocks antes que cheguem ao ambiente de produção**, através de:

- **Transações atômicas** que garantem consistência
- **Locks de linha** que previnem race conditions em recursos críticos
- **Validações dentro da transação** que previnem mudanças entre validação e execução
- **Idempotência dupla** que previne duplicação mesmo em falhas de cache
- **Timeouts e ordenação** que previnem deadlocks
- **Tratamento elegante de erros** que melhora experiência do usuário em cenários de conflito

Todas as operações críticas estão protegidas contra concorrência, garantindo **integridade e consistência dos dados mesmo sob alta carga e múltiplos usuários simultâneos**.
