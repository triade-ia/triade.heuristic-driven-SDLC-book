# Relatório Multi-User — Envio de QualiPoints

**Heurística aplicada:** Multi-User (Gestão de Concorrência e Conflitos)
**Requisito analisado:** Envio de QualiPoints entre usuários (v2)
**Modo de entrada:** Épico + Tarefa (requisito revisado completo)
**Data:** 2026-03-29

---

## 1. Lente da Colisão

**Pergunta-guia:** O que acontece se dois atores tentarem o Update no mesmo campo no exato milissegundo?

### Cenários identificados

| # | Cenário de colisão | Risco | Severidade |
|---|-------------------|-------|------------|
| C1 | Remetente A envia 8.000 pontos para B **e** 5.000 para C simultaneamente. Saldo de A = 10.000. Ambas as requisições leem saldo 10.000, ambas aprovam, resultado: saldo fica -3.000. | **Race condition clássica de saldo** — débito duplo por leitura concorrente do mesmo saldo | **Crítica** |
| C2 | Dois remetentes diferentes (A e B) enviam pontos para o mesmo destinatário C no mesmo instante. Ambos creditam C; se o UPDATE não for serializado, um crédito pode sobrescrever o outro. | **Lost update no saldo do destinatário** | **Crítica** |
| C3 | Remetente A envia para B. B envia para A. Ambos no mesmo instante. Dependendo da ordem de lock, pode haver deadlock (A trava saldo-A → tenta saldo-B; B trava saldo-B → tenta saldo-A). | **Deadlock potencial em transferências cruzadas** | **Alta** |

### Estratégia de locking recomendada

| Estratégia | Aplicação | Justificativa |
|-----------|-----------|---------------|
| **Pessimista (SELECT ... FOR UPDATE)** | Leitura e atualização do saldo do remetente e do destinatário dentro da mesma transação de banco | Para operações financeiras com escrita simultânea em recurso crítico (saldo), locking pessimista é a abordagem mais segura. Evita leitura fantasma e garante serialização. |
| **Ordem fixa de lock** | Sempre adquirir lock no menor userId primeiro, depois no maior | Previne deadlock no cenário C3 (transferências cruzadas). Exemplo: se userA.id < userB.id, sempre lock(userA) → lock(userB), independente de quem é remetente ou destinatário. |

### Pseudocódigo recomendado para atomicidade

```
BEGIN TRANSACTION

  -- Ordem fixa de lock para evitar deadlock
  ids = SORT([remetente_id, destinatario_id])
  SELECT saldo FROM carteira WHERE user_id = ids[0] FOR UPDATE
  SELECT saldo FROM carteira WHERE user_id = ids[1] FOR UPDATE

  -- Validação de saldo (feita DENTRO da transação, após o lock)
  IF saldo_remetente < valor THEN
    ROLLBACK → retornar 422 INSUFFICIENT_BALANCE
  END IF

  UPDATE carteira SET saldo = saldo - valor WHERE user_id = remetente_id
  UPDATE carteira SET saldo = saldo + valor WHERE user_id = destinatario_id
  INSERT INTO transacoes (remetente, destinatario, valor, status, idempotency_key, created_at)

COMMIT
```

### Gaps identificados no requisito

- **GAP-C1:** O requisito menciona "operação atômica" (seções 5.3, 6.2, 8.1), mas **não especifica a estratégia de locking** (pessimista vs. otimista). Recomendação: explicitar `SELECT ... FOR UPDATE` ou equivalente.
- **GAP-C2:** O requisito **não menciona ordem de aquisição de locks** para prevenir deadlocks em transferências cruzadas (A→B e B→A simultâneos). Recomendação: documentar a regra de "lock por menor ID primeiro".
- **GAP-C3:** A validação de saldo deve ocorrer **após** a aquisição do lock, não antes. O requisito não deixa isso explícito.

---

## 2. Lente da Sessão Dupla

**Pergunta-guia:** O sistema suporta um único usuário operando em abas ou dispositivos diferentes sem gerar inconsistência?

### Cenários identificados

| # | Cenário | Risco | Severidade |
|---|---------|-------|------------|
| S1 | Usuário abre o App no celular e no tablet. Em ambos, visualiza saldo 5.000. Inicia envio de 4.000 em cada dispositivo simultaneamente. | **Débito duplo do mesmo usuário** — mesmo cenário de C1, mas originado por sessões múltiplas do mesmo ator | **Crítica** |
| S2 | Usuário envia no App (celular). Abre o painel web. Saldo no painel ainda mostra o valor antigo porque não houve refresh. | **Inconsistência visual entre canais** — problema de UX, não de integridade | **Baixa** |
| S3 | Usuário faz envio no celular. No tablet, a lista de transações recentes não mostra a nova transação até refresh. | **Lista desatualizada em sessão paralela** | **Baixa** |

### Análise

| Aspecto | Status no requisito | Avaliação |
|---------|-------------------|-----------|
| Múltiplas sessões simultâneas | Não mencionado | O requisito não restringe sessões simultâneas. O mecanismo de idempotência protege contra retry, mas **não protege contra duas intenções distintas** do mesmo usuário (duas Idempotency-Keys diferentes, uma por dispositivo). |
| Token de autenticação | Seção 3.1 — identificação por token | Se cada dispositivo tem seu próprio token, ambos podem fazer requisições válidas simultaneamente. Isso é correto e esperado; a proteção é via lock no saldo (Lente da Colisão). |
| Sincronização entre sessões | Seção 13 — polling/refresh manual | Aceitável para MVP. Não há WebSocket nem push de atualização. |

### Gaps identificados no requisito

- **GAP-S1:** O requisito não aborda o cenário de **sessões simultâneas do mesmo usuário gerando intenções distintas** (não é retry — são duas operações legítimas). A proteção existe via lock de saldo (a segunda operação falha por saldo insuficiente), mas o requisito deveria documentar esse comportamento explicitamente.
- **GAP-S2:** Considerar para backlog: **rate limit por usuário** (ex.: máximo N envios por minuto) como camada adicional de proteção contra automação ou abuso em múltiplas sessões.

---

## 3. Lente do Inventário Crítico

**Pergunta-guia:** Como garantimos que "o último item" não seja vendido para duas pessoas?

### Recurso crítico identificado

O **saldo do remetente** é o recurso de inventário crítico. Diferente de estoque físico (que chega a zero e acabou), o saldo pode ser consumido parcialmente, mas o risco é o mesmo: duas operações concorrentes podem "gastar" um saldo que só cobre uma delas.

### Cenários identificados

| # | Cenário | Risco | Severidade |
|---|---------|-------|------------|
| I1 | Saldo = 100. Envio de 100 para B e envio de 100 para C ao mesmo tempo. Sem lock, ambos veem saldo 100, ambos aprovam, saldo fica -100. | **Saldo negativo por consumo concorrente do inventário** | **Crítica** |
| I2 | Saldo = 100. Envio de 60 para B e envio de 60 para C ao mesmo tempo. Sem lock, ambos veem 100, ambos aprovam, saldo fica -20. | **Variante parcial do mesmo problema** | **Crítica** |

### Mecanismos de proteção recomendados

| Camada | Mecanismo | Propósito |
|--------|-----------|-----------|
| **Banco de dados** | `SELECT ... FOR UPDATE` + validação de saldo dentro da transação | Garantia primária: serializa acessos ao saldo |
| **Banco de dados** | Constraint `CHECK (saldo >= 0)` na coluna de saldo | Garantia secundária (defense in depth): mesmo que haja bug no código, o banco rejeita saldo negativo |
| **Aplicação** | Validação de saldo após lock, antes do UPDATE | Garantia na camada de negócio |

### Gaps identificados no requisito

- **GAP-I1:** O requisito menciona "saldo ≥ valor" (seção 9, regra 4), mas **não especifica constraint de banco** para impedir saldo negativo como defesa em profundidade. Recomendação: adicionar `CHECK (saldo >= 0)` na tabela de carteira.
- **GAP-I2:** O requisito não define comportamento quando o saldo é exatamente zero — embora o valor mínimo de envio seja 1, é importante documentar que saldo = 0 resulta em `INSUFFICIENT_BALANCE`.

---

## 4. Lente da UX de Conflito

**Pergunta-guia:** Quando a colisão ocorre, o sistema explode com um 500 ou resolve de forma elegante?

### Análise do tratamento de erros no requisito

| Cenário de conflito | Código HTTP | Mensagem para o usuário | Status no requisito |
|---------------------|-------------|------------------------|---------------------|
| Saldo insuficiente (ex.: concorrência drenou o saldo) | 422 | "Saldo insuficiente. Seu saldo atual não permite este envio." | **Coberto** (seção 10.4 e 11.2) |
| Timeout por lock demorado (contenção alta) | 408/504 | "A operação demorou mais que o esperado. Tente novamente." | **Parcialmente coberto** (seção 5.2) |
| Deadlock detectado pelo banco | Não mapeado | Não mapeado | **Não coberto** |
| Retry com mesma Idempotency-Key | 200 (mesmo resultado original) | Transparente para o usuário | **Coberto** (seção 8.2) |
| Retry com Idempotency-Key diferente (duplo clique gerando nova key) | 422 (se saldo drenou) ou 200 (se saldo suficiente) | Depende do estado do saldo | **Parcialmente coberto** |
| Erro interno durante transação | 500 | "Ocorreu um erro. Tente novamente em instantes." | **Coberto** (seção 10.4) |

### Gaps identificados no requisito

- **GAP-U1:** **Deadlock handling não está mapeado.** Se o banco detectar deadlock (ex.: por falta de ordem fixa de lock), o erro propagado ao cliente será um 500 genérico. Recomendação: a aplicação deve capturar exceções de deadlock e fazer retry automático (1-2 tentativas internas), transparente para o cliente. Se persistir, retornar 503 com mensagem "Serviço temporariamente indisponível, tente novamente".
- **GAP-U2:** **Lock wait timeout não está mapeado.** Se um lock demorar além do timeout do banco (ex.: innodb_lock_wait_timeout), o erro pode ser confundido com timeout de API. Recomendação: distinguir timeout de lock (retentável) de timeout de processamento (pode indicar problema maior).
- **GAP-U3:** **Feedback de saldo atualizado após falha.** Quando a segunda requisição concorrente falha por `INSUFFICIENT_BALANCE`, o campo `currentBalance` está marcado como "opcional" no requisito (seção 10.4). Recomendação: tornar obrigatório para que o App atualize o saldo exibido e o usuário entenda o motivo da rejeição.

---

## 5. Consolidação de Riscos e Recomendações

### Mapa de riscos

| ID | Risco | Lente | Severidade | Mitigação |
|----|-------|-------|------------|-----------|
| R1 | Race condition no saldo do remetente | Colisão / Inventário | **Crítica** | `SELECT ... FOR UPDATE` com validação pós-lock |
| R2 | Lost update no saldo do destinatário | Colisão | **Crítica** | Lock no destinatário dentro da mesma transação |
| R3 | Deadlock em transferências cruzadas | Colisão | **Alta** | Ordem fixa de aquisição de lock (menor ID primeiro) |
| R4 | Saldo negativo por bug | Inventário | **Alta** | Constraint `CHECK (saldo >= 0)` no banco |
| R5 | Deadlock não tratado na UX | UX de Conflito | **Média** | Retry automático na aplicação + código de erro específico |
| R6 | Sessões simultâneas com intenções distintas | Sessão Dupla | **Média** | Lock de saldo protege integridade; documentar comportamento |
| R7 | Lock wait timeout confundido com API timeout | UX de Conflito | **Média** | Distinguir tipos de timeout no tratamento de erro |

### Recomendações de implementação

| # | Recomendação | Prioridade |
|---|-------------|------------|
| 1 | Implementar `SELECT ... FOR UPDATE` no saldo de ambos os usuários (remetente e destinatário) dentro de uma única transação | **Must have** |
| 2 | Adotar ordem fixa de lock por menor userId para prevenir deadlocks | **Must have** |
| 3 | Adicionar constraint `CHECK (saldo >= 0)` na tabela de carteira | **Must have** |
| 4 | Validar saldo **após** aquisição do lock, nunca antes | **Must have** |
| 5 | Capturar exceções de deadlock e fazer retry interno (1-2x) antes de retornar erro | **Should have** |
| 6 | Tornar obrigatório o campo `currentBalance` na resposta 422 | **Should have** |
| 7 | Adicionar rate limit por usuário (backlog, mas documentar a necessidade) | **Nice to have** |
| 8 | Distinguir lock timeout de API timeout nos logs e na resposta | **Nice to have** |

---

## 6. Estratégias de Locking Recomendadas

| Componente | Estratégia | Detalhe |
|-----------|-----------|---------|
| **Saldo (carteira)** | Pessimista (`FOR UPDATE`) | Operação financeira crítica — não pode tolerar conflito otimista com retry do cliente |
| **Idempotency Key** | Insert com `UNIQUE` constraint + `ON CONFLICT DO NOTHING` (ou equivalente) | Primeira requisição insere; requisições subsequentes detectam duplicata e retornam resultado armazenado |
| **Registro de transação** | Insert simples dentro da transação | Protegido pelo escopo transacional já existente |

---

## 7. Identificação de Race Conditions e Deadlocks

### Race Conditions

| ID | Descrição | Ponto no código | Prevenção |
|----|-----------|----------------|-----------|
| RC1 | Leitura de saldo fora do lock permite aprovação de duas operações que juntas excedem o saldo | Service layer — antes do UPDATE de saldo | Ler saldo com `FOR UPDATE` |
| RC2 | Duas requisições com Idempotency-Keys diferentes processadas em paralelo para o mesmo remetente | Endpoint POST /transfers | Lock no saldo serializa naturalmente |
| RC3 | Crédito ao destinatário sem lock pode perder update se outro remetente credita simultaneamente | UPDATE saldo destinatário | `FOR UPDATE` no destinatário também |

### Deadlocks

| ID | Descrição | Condição | Prevenção |
|----|-----------|----------|-----------|
| DL1 | Transferência cruzada: A→B e B→A simultaneamente, cada um trava "seu" saldo primeiro | Duas transações adquirindo locks em ordens inversas | Ordem fixa: sempre lock no menor userId primeiro |
| DL2 | Lock em tabela de transações + lock em carteira em ordens diferentes entre fluxos | Múltiplos fluxos de negócio acessando as mesmas tabelas | Padronizar ordem de acesso: carteira → transações (nunca o inverso) |

---

## 8. Gatilhos de Heurísticas Complementares

### State Analysis
**Recomendação:** Para garantir que as transições de estado da transação (processando → concluído / falhou) sejam atômicas e resistentes a concorrência, aplique a heurística **State Analysis**. O requisito define estados na seção 11.1, mas são estados de UX (no App), não estados persistidos no servidor. Se no futuro a transação tiver estados intermediários persistidos (ex.: "pendente"), a análise de transição de estado se torna crítica.

### Count (0, 1, Muitos)
**Recomendação:** Para validar o volume esperado de acessos simultâneos e dimensionar os mecanismos de locking (ex.: pool de conexões, timeout de lock, particionamento de tabela de saldo), aplique a heurística **Count (0, 1, Muitos)**. Perguntas a responder:
- Quantos envios simultâneos são esperados no pico?
- Qual o throughput máximo de transações por segundo?
- Quantos usuários distintos podem enviar para o mesmo destinatário simultaneamente?

---

## 9. Resumo Executivo

O requisito de Envio de QualiPoints **já contempla** os fundamentos de concorrência (atomicidade, idempotência, rollback), mas apresenta **gaps na especificação da estratégia de locking** que, se não endereçados na implementação, podem resultar em race conditions críticas (saldo negativo, débito duplo) e deadlocks.

**Ações prioritárias antes do desenvolvimento:**

1. Especificar `SELECT ... FOR UPDATE` como estratégia de locking para saldos
2. Definir ordem fixa de aquisição de locks (menor userId primeiro)
3. Adicionar constraint `CHECK (saldo >= 0)` como defesa em profundidade
4. Mapear tratamento de deadlock e lock timeout na aplicação

Com essas adições, o requisito estará robusto para cenários de alta concorrência no MVP.
