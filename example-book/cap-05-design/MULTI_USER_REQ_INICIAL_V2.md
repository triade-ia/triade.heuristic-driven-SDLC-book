# Relatório: Multi-User — REQ_INICIAL_V2 (QualiPoints)

**Requisito analisado:** Envio de QualiPoints entre usuários (REQ_INICIAL_V2).
**Skill aplicada:** Cap05_design — Multi-User (Gestão de Concorrência e Conflitos)
**Data:** 2026-02-18
**Status:** Análise de design concluída

---

## 1. Lente da Colisão

**O que acontece se dois atores tentarem debitar o mesmo saldo no exato milissegundo?**

### Cenário crítico: dois envios simultâneos do mesmo remetente

O remetente possui saldo de **200 QualiPoints** e inicia dois envios distintos (ex.: em dois dispositivos, ou por duplo envio paralelo no App):

- **Requisição A:** envia 150 QP para o Usuário B (`Idempotency-Key: key-A`)
- **Requisição B:** envia 100 QP para o Usuário C (`Idempotency-Key: key-B`)

Sem mecanismo de locking, a sequência de execução pode ser:

| Passo | Requisição A | Requisição B | Estado do Saldo |
|-------|-------------|-------------|-----------------|
| 1 | Lê saldo = 200 | — | 200 |
| 2 | — | Lê saldo = 200 | 200 |
| 3 | Valida: 200 ≥ 150 ✅ | — | 200 |
| 4 | — | Valida: 200 ≥ 100 ✅ | 200 |
| 5 | Debita 150 → grava saldo = 50 | — | 50 |
| 6 | — | Debita 100 → grava saldo = 100 | **100 (corrompido!)** |

**Resultado:** Saldo real deveria ser -50 (operação inválida), mas o banco registra 100 — o débito de A foi sobrescrito pelo débito de B. **Inconsistência crítica.**

### Riscos identificados

| Risco | Origem | Severidade |
|-------|--------|-----------|
| **Race condition no saldo** — duas leituras leem o mesmo valor antes de qualquer escrita | `UPDATE saldo` sem isolamento transacional adequado | Crítica |
| **Lost Update** — a escrita de uma transação sobrescreve a escrita de outra | Ausência de locking na tabela de saldo/carteira | Crítica |
| **Saldo negativo** — ambas as transações passam na validação com o mesmo saldo lido | Validação e débito não são atômicos na mesma operação isolada | Alta |
| **Race condition na Idempotency-Key** — duas requisições com a **mesma** key chegam antes de qualquer resultado ser gravado | Ausência de constraint UNIQUE ou lock atômico no armazenamento da key | Alta |

### Estratégias de locking recomendadas

| Estratégia | Mecanismo | Quando usar |
|-----------|----------|------------|
| **Pessimista (SELECT FOR UPDATE)** | Bloqueia a linha do saldo do remetente na leitura; outras transações aguardam até o commit/rollback | Recomendado para o MVP — operações financeiras exigem consistência forte |
| **Otimista (versioning)** | Lê o saldo com um `version` ou `updated_at`; no UPDATE verifica se o valor não mudou desde a leitura; se mudou, retorna conflito e força retry | Alternativa se a contenção esperada for baixa; adiciona complexidade de retry na aplicação |
| **Híbrida** | SELECT FOR UPDATE no saldo do remetente + versioning para auditoria | Para ambientes de alta concorrência com necessidade de rastreabilidade |

**Recomendação para o MVP:** Usar **locking pessimista** (`SELECT FOR UPDATE` na linha de saldo do remetente) dentro da transação atômica de banco. Simples, seguro e consistente com a operação já definida como atômica no requisito.

---

## 2. Lente da Sessão Dupla

**O sistema suporta um único usuário operando em dispositivos ou abas diferentes sem gerar inconsistência?**

### Cenário: usuário autenticado em dois dispositivos simultaneamente

O requisito define que o remetente é identificado pelo **token de autenticação** (Bearer). Dois dispositivos do mesmo usuário podem possuir tokens válidos ao mesmo tempo, e ambos podem iniciar transferências independentes.

| Situação | Comportamento esperado | Risco |
|----------|----------------------|-------|
| **Dois envios diferentes** (keys distintas) do mesmo remetente em paralelo | Ambas devem ser processadas se o saldo total for suficiente | Race condition no saldo — ver Lente da Colisão |
| **Mesmo envio** (mesma `Idempotency-Key`) enviado em paralelo de dois dispositivos | A API deve processar **uma** vez e retornar o mesmo resultado para ambas | Race condition na inserção da key — pode processar duas vezes se não houver lock atômico na key |
| **Consulta de saldo** em um dispositivo enquanto o outro realiza um envio | Saldo exibido pode estar desatualizado | Aceitável no MVP com refresh manual; sem WebSocket não há atualização em tempo real |
| **Envio simultâneo** que somado excede o saldo, mas individualmente é válido | Sem locking, ambos podem passar na validação | Saldo pode ficar negativo — mitigado com SELECT FOR UPDATE |

### Controle de sessão e token

O requisito não especifica:
- Se múltiplos tokens simultâneos para o mesmo usuário são permitidos
- Se há limite de sessões ativas por usuário
- Se o token de um dispositivo invalida o token do outro

**Lacuna identificada:** Sem controle de sessões concorrentes explícito, a defesa contra operações paralelas do mesmo usuário recai inteiramente sobre o locking no banco de dados.

### Sincronização entre sessões

| Canal | Sincronização após envio | Status no MVP |
|-------|-------------------------|--------------|
| **App (mesmo dispositivo)** | Saldo e lista de recentes atualizados após resposta 200 | ✅ Definido no requisito |
| **App (dispositivo B)** | Sem sincronização ativa — depende de refresh manual | ⚠️ Não definido — risco de exibição de saldo desatualizado |
| **Painel Web** | Polling ou refresh manual | ✅ Aceitável no MVP, conforme seção 13 do requisito |

---

## 3. Lente do Inventário Crítico

**Como garantir que o saldo não seja debitado além do disponível para dois usuários simultâneos?**

No contexto de QualiPoints, o **saldo do remetente** é o "inventário crítico" — recurso limitado que não pode ser vendido (debitado) para dois destinatários ao mesmo tempo se o total exceder o disponível.

### Estratégia de reserva e bloqueio

| Etapa | Ação | Mecanismo recomendado |
|-------|------|-----------------------|
| **1. Leitura com bloqueio** | Ler o saldo do remetente travando a linha para escrita | `SELECT saldo FROM carteira WHERE usuario_id = :id FOR UPDATE` |
| **2. Validação** | Verificar se `saldo ≥ amount` | Dentro da transação, com o lock ativo |
| **3. Débito** | Atualizar o saldo: `saldo = saldo - amount` | `UPDATE carteira SET saldo = saldo - :amount WHERE usuario_id = :id` |
| **4. Crédito** | Atualizar o saldo do destinatário | `UPDATE carteira SET saldo = saldo + :amount WHERE usuario_id = :recipientId` |
| **5. Registro** | Inserir o registro da transação | `INSERT INTO transacoes (...)` |
| **6. Commit** | Liberar todos os locks | `COMMIT` — locks liberados automaticamente |

**Comportamento quando o saldo se esgota durante operação concorrente:**

- Requisição A adquire o lock, valida saldo suficiente, debita e faz commit.
- Requisição B tenta adquirir o lock, aguarda. Após o commit de A, lê o saldo atualizado (agora insuficiente) e retorna **422 INSUFFICIENT_BALANCE**.
- **Resultado correto:** nenhum débito acima do saldo disponível. O lock pessimista garante serialização.

### Riscos no armazenamento da Idempotency-Key

O "inventário crítico" secundário é a **unicidade da Idempotency-Key**:

| Cenário | Risco | Mitigação |
|---------|-------|-----------|
| Duas requisições com a **mesma key** chegam antes de qualquer resultado ser persistido | Sem constraint UNIQUE, ambas podem processar | `UNIQUE constraint` na coluna `idempotency_key` + captura de `DuplicateKeyException` para retornar o resultado já gravado |
| Duas requisições com a **mesma key** — uma em processamento, outra aguardando | A segunda deve receber o resultado da primeira, não um novo processamento | Lock por key (ex.: `SELECT ... FOR UPDATE` na tabela de idempotência ou `Redis SETNX`) |
| Keys sem TTL acumulam indefinidamente | Degradação de performance em leitura/escrita da tabela | ❌ Não definido no requisito — lacuna: definir política de expiração (ex.: 24h) |

---

## 4. Lente da UX de Conflito

**Quando a colisão ocorre, o sistema retorna um erro 500 genérico ou resolve de forma elegante?**

### Tratamento de conflitos por cenário

| Cenário de Conflito | Código HTTP | Código de Erro | Mensagem ao Usuário | Comportamento |
|--------------------|------------|----------------|---------------------|---------------|
| **Saldo insuficiente por concorrência** (outro envio do mesmo usuário debitou primeiro) | 422 | `INSUFFICIENT_BALANCE` | "Saldo insuficiente. Seu saldo atual não permite este envio." | ✅ Definido no requisito |
| **Retry com mesma Idempotency-Key** (operação já processada) | 200 | — | Mesmo corpo da resposta original | ✅ Definido no requisito |
| **Race condition na Idempotency-Key** (duas requisições idênticas simultâneas) | 200 | — | Retornar resultado da primeira; descartar processamento da segunda | ⚠️ Definido no comportamento, mas mecanismo de garantia não especificado |
| **Timeout por contenção de lock** (requisição aguardando lock por muito tempo) | 504 | `REQUEST_TIMEOUT` | "A operação demorou mais que o esperado. Tente novamente." | ✅ Definido — timeout de 10 segundos |
| **Deadlock no banco de dados** (transferências cruzadas A→B e B→A simultâneas) | 5xx | `INTERNAL_ERROR` | "Ocorreu um erro. Tente novamente em instantes." | ⚠️ Não explicitado — deadlock não está mapeado no requisito |

### Cenário de deadlock: transferências cruzadas simultâneas

Cenário raro mas possível com locking pessimista:

- **Requisição X:** A → B — adquire lock no saldo de A, aguarda lock no saldo de B
- **Requisição Y:** B → A — adquire lock no saldo de B, aguarda lock no saldo de A

**Resultado:** Deadlock. O banco de dados detecta e mata uma das transações (rollback automático), que retorna 5xx. O cliente pode fazer retry com a mesma Idempotency-Key.

**Mitigação:** Adquirir locks sempre na mesma ordem (ex.: pelo ID do usuário em ordem crescente) — elimina deadlocks em transferências cruzadas.

### Feedback ao usuário: estado de conflito

| Estado | Exibição no App | Ação permitida |
|--------|----------------|----------------|
| **Processando** | Indicador de loading; botão de envio desabilitado | Nenhuma (aguardar resposta) |
| **Conflito resolvido com sucesso** (retry idempotente retornou 200) | Confirmação de sucesso | Ver extrato atualizado |
| **Saldo insuficiente após conflito** | Mensagem de erro clara com saldo atual (se disponível na resposta 422) | Tentar com valor menor ou aguardar crédito |
| **Timeout por contenção** | Mensagem orientando retry | Retry com mesma Idempotency-Key |

---

## 5. Riscos de Concorrência Identificados

| # | Risco | Severidade | Probabilidade | Status no Requisito | Mitigação Necessária |
|---|-------|-----------|---------------|--------------------|--------------------|
| 1 | **Race condition no saldo** — duas requisições distintas leem o mesmo saldo antes de qualquer escrita | Crítica | Alta | ⚠️ Implícito na atomicidade, mecanismo não especificado | Especificar `SELECT FOR UPDATE` no saldo do remetente |
| 2 | **Lost Update** — escrita de uma transação sobrescreve escrita de outra | Crítica | Alta | ⚠️ Não endereçado | Garantido indiretamente pelo lock pessimista |
| 3 | **Race condition na Idempotency-Key** — duas requisições idênticas simultâneas processam duas vezes | Alta | Média | ⚠️ Comportamento definido, mecanismo não especificado | `UNIQUE constraint` + tratamento de `DuplicateKeyException` |
| 4 | **Deadlock em transferências cruzadas** (A→B e B→A simultâneas) | Alta | Baixa | ❌ Não mapeado | Adquirir locks por ordem de ID; tratar rollback automático do banco |
| 5 | **Saldo exibido desatualizado em sessão dupla** | Baixa | Alta | ✅ Aceitável no MVP (polling/refresh) | Documentar comportamento; WebSocket em backlog |
| 6 | **Idempotency-Keys sem expiração** — acúmulo ilimitado | Média | Alta | ❌ Não definido | Definir TTL (ex.: 24h) e job de limpeza |
| 7 | **Contenção de lock em alta concorrência** — usuário muito ativo gera fila de espera | Média | Média | ⚠️ Timeout de 10s definido, mas contenção não mapeada | Monitorar P99 de latência; avaliar locking otimista se contenção crescer |

---

## 6. Mecanismos de Controle de Concorrência Propostos

### Para o MVP (inegociável)

```
BEGIN TRANSACTION;
  -- 1. Adquirir lock no saldo do remetente
  SELECT saldo FROM carteira WHERE usuario_id = :senderId FOR UPDATE;

  -- 2. Validar saldo suficiente
  IF saldo < :amount THEN
    ROLLBACK;
    RETURN 422 INSUFFICIENT_BALANCE;
  END IF;

  -- 3. Verificar/inserir Idempotency-Key (com UNIQUE constraint)
  INSERT INTO idempotency_keys (key, status) VALUES (:key, 'processing')
    ON CONFLICT (key) DO NOTHING;
  -- Se inserção retornou 0 linhas, a key já existe → retornar resultado cacheado

  -- 4. Debitar remetente
  UPDATE carteira SET saldo = saldo - :amount WHERE usuario_id = :senderId;

  -- 5. Creditar destinatário
  UPDATE carteira SET saldo = saldo + :amount WHERE usuario_id = :recipientId;

  -- 6. Registrar transação
  INSERT INTO transacoes (...) VALUES (...);

  -- 7. Atualizar Idempotency-Key com resultado
  UPDATE idempotency_keys SET status = 'completed', resultado = :response
    WHERE key = :key;

COMMIT;
```

### Ordem de aquisição de locks (prevenção de deadlock)

```
-- Sempre adquirir locks em ordem crescente de usuario_id
-- Independente de quem é remetente ou destinatário

IF senderId < recipientId THEN
  LOCK senderId FIRST, THEN recipientId;
ELSE
  LOCK recipientId FIRST, THEN senderId;
END IF;
```

---

## 7. Gatilhos de Ação Manual (Investigação Complementar) ⚡

### State Analysis
**Quando aplicar:** Para verificar se as transições de estado da transação (`processando → concluído → falhou`) são atômicas e não podem ser interrompidas por operações concorrentes.

**Recomendação:** "Para garantir que as transições de estado sejam atômicas e resistentes a concorrência, aplique a heurística **State Analysis**. Em especial, verificar o estado intermediário 'processando' da Idempotency-Key e garantir que nunca há leitura de resultado parcial."

### Count (0, 1, Muitos)
**Quando aplicar:** Para dimensionar os mecanismos de controle de concorrência conforme o volume esperado de acessos simultâneos.

**Recomendação:** "Para validar o volume esperado de acessos simultâneos e dimensionar os mecanismos de locking, aplique a heurística **Count (0, 1, Muitos)**. Em especial: 0 transações simultâneas (baseline), 1 transação (fluxo normal), muitas transações (usuário com alto volume ou campanha de envio em massa)."

---

## 8. Decisões de Concorrência (Consolidado)

| Decisão | Recomendação | Prioridade |
|---------|-------------|-----------|
| **Mecanismo de locking** | Locking pessimista (`SELECT FOR UPDATE`) no saldo do remetente | MVP — crítico |
| **Ordem de locks** | Sempre adquirir por `usuario_id` crescente para prevenir deadlock | MVP — crítico |
| **Unicidade da Idempotency-Key** | `UNIQUE constraint` no banco + tratamento de exceção de duplicidade | MVP — crítico |
| **TTL da Idempotency-Key** | Definir expiração (ex.: 24h) e job de limpeza periódico | MVP — importante |
| **Sessões concorrentes** | Documentar que múltiplos dispositivos são suportados; defesa via locking no banco | MVP — aceitável |
| **Tratamento de deadlock** | Capturar erro de deadlock do banco e retornar 5xx; cliente faz retry com mesma key | MVP — necessário |
| **Locking otimista** | Avaliar como alternativa se a contenção crescer em produção | Backlog |
| **Atualização em tempo real entre sessões** | WebSocket ou SSE para sincronizar saldo entre dispositivos | Backlog |

---

## Referências

- **Requisito analisado:** [REQ_INICIAL_V2.md](../requisito-revisado/REQ_INICIAL_V2.md)
- **Heurística aplicada:** [MultiUser.md](../../Skill/Cap05_design/MultiUser.md)
- **Análise complementar já realizada:** [DEPENDENCIES_REQ_INICIAL_V2.md](./DEPENDENCIES_REQ_INICIAL_V2.md)
- **Próximas heurísticas recomendadas:** State Analysis, Count (ver seção 7)
