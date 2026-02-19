# Relatório: State Analysis — REQ_INICIAL_V2 (QualiPoints)

**Requisito analisado:** Envio de QualiPoints entre usuários (REQ_INICIAL_V2).
**Skill aplicada:** Cap05_design — State Analysis (Análise de Máquinas de Estado e Transições)
**Data:** 2026-02-18
**Status:** Análise de design concluída

---

## 1. Identificação de Estados (Onde estou?)

O requisito envolve três máquinas de estado distintas que interagem entre si: o **estado da transação**, o **estado da conta do usuário** e o **estado da Idempotency-Key**.

### 1.1 Máquina de Estados da Transação

| Estado | Tipo | Descrição |
|--------|------|-----------|
| **Processando** | Transitório | Requisição recebida pela API; débito/crédito/registro em execução dentro da transação atômica de banco. |
| **Concluída** | Final | Débito do remetente, crédito do destinatário e registro persistidos com sucesso (HTTP 200). |
| **Falhou** | Final | A operação foi rejeitada por regra de negócio (4xx) ou por falha interna (5xx); rollback realizado. |

> **Estados finais:** `Concluída` e `Falhou` são imutáveis — uma transação concluída não pode ser revertida no MVP (estorno fica para backlog); uma transação que falhou não pode ser concluída (apenas uma nova intenção — nova Idempotency-Key — pode ser enviada).

### 1.2 Máquina de Estados da Conta do Usuário

Derivada das restrições das seções 3.1, 3.2 e 3.3 do requisito:

| Estado | Pode Enviar | Pode Receber | Descrição |
|--------|-------------|--------------|-----------|
| **Ativo** | Sim | Sim | Estado normal de operação. |
| **Bloqueado** | Não | Não | Conta impedida de enviar e de receber. |
| **Pendente** | Não | Não definido | Conta ainda não ativa; envio rejeitado. |
| **Em Análise** | Não | Não definido | Conta sob revisão; envio rejeitado. |

> O requisito define os estados da conta apenas pelo efeito nas operações (ativo/bloqueado/pendente/em análise), mas não especifica as transições entre eles — isso é lacuna explicitada na seção 5 deste relatório.

### 1.3 Máquina de Estados da Idempotency-Key

| Estado | Descrição |
|--------|-----------|
| **Inexistente** | Key ainda não foi registrada pela API. |
| **Processando** | Key inserida; transação em execução. Estado intermediário para prevenir race condition na inserção dupla. |
| **Concluída** | Transação executada com sucesso; resultado persistido. |
| **Falhou** | Transação rejeitada; erro persistido junto à key. |

> `Concluída` e `Falhou` são estados finais da key. Uma key `Processando` que nunca avança (crash durante a transação) é um risco tratado na seção 4.

---

## 2. Mapeamento de Transições (Como mudo?)

### 2.1 Transições da Transação

```
[Inexistente] ──(POST /transfers recebido)──► [Processando]
                                                    │
                          ┌─────────────────────────┤
                          │                         │
                (validações passam +          (validação falha
                 commit bem-sucedido)          ou erro interno)
                          │                         │
                          ▼                         ▼
                     [Concluída]               [Falhou]
```

| Transição | Gatilho | Pré-condições | Ações executadas |
|-----------|---------|---------------|------------------|
| `Inexistente → Processando` | POST `/api/v1/transfers` recebido com headers válidos | Token válido; Idempotency-Key ainda não existe | Iniciar transação de banco; registrar key no estado `Processando`; adquirir lock no saldo do remetente |
| `Processando → Concluída` | Commit da transação atômica | Saldo suficiente; destinatário ativo e diferente do remetente; valor entre 1–10.000 | Debitar remetente; creditar destinatário; inserir registro na tabela de transações; atualizar key para `Concluída`; retornar HTTP 200 |
| `Processando → Falhou` | Erro de validação de negócio ou falha interna seguida de rollback | Qualquer pré-condição não atendida (saldo insuficiente, destinatário inválido, conta inativa, etc.) | Rollback de todos os passos; atualizar key para `Falhou`; retornar HTTP 4xx/5xx correspondente |

### 2.2 Transições da Conta do Usuário (implícitas no requisito)

| Transição | Impacto na operação de envio |
|-----------|------------------------------|
| `Ativo → Bloqueado` | Envios futuros rejeitados (HTTP 403 `ACCOUNT_NOT_ACTIVE`); destinatários bloqueados rejeitados (HTTP 404 `RECIPIENT_NOT_FOUND`) |
| `Pendente → Ativo` | Habilita envio |
| `Em Análise → Ativo` | Habilita envio |
| Demais transições | **Não mapeadas no requisito** — lacuna identificada |

### 2.3 Transições da Idempotency-Key

| Transição | Gatilho | Ação |
|-----------|---------|------|
| `Inexistente → Processando` | Inserção atômica no início da transação (INSERT com UNIQUE constraint) | Registrar key; prosseguir com o processamento |
| `Processando → Concluída` | Commit bem-sucedido da transação | Persistir resultado (HTTP 200 + corpo) junto à key |
| `Processando → Falhou` | Rollback por erro de negócio | Persistir erro junto à key |
| `Qualquer → retorno cacheado` | Requisição de retry com key já existente | Retornar resultado já persistido sem reprocessar |

---

## 3. Validação de Invariantes (O que não pode acontecer?)

### 3.1 Invariantes da Transação

| # | Invariante | Descrição | Consequência se violada |
|---|-----------|-----------|------------------------|
| I-1 | **Transação concluída é imutável** | `Concluída → Falhou` não é permitida; `Falhou → Concluída` não é permitida. | Crédito ou débito incorreto pós-fato; inconsistência de auditoria. |
| I-2 | **Atomicidade total** | Nunca há débito sem crédito correspondente, e vice-versa. | Saldo do sistema fora de balanço (vazamento ou perda de pontos). |
| I-3 | **Sem estado Processando persistente após resposta** | O estado `Processando` é transitório: ao fim da chamada síncrona, a transação está `Concluída` ou `Falhou`. | "Transação zumbi" — saldo bloqueado sem conclusão. |
| I-4 | **Idempotência** | A mesma Idempotency-Key jamais gera dois débitos. | Débito duplo — perda de pontos para o remetente. |
| I-5 | **Saldo nunca negativo** | O saldo do remetente após débito deve ser ≥ 0. | Crédito sem cobertura — pontos criados do nada. |
| I-6 | **Envio para si mesmo rejeitado** | `recipientId == senderId` sempre resulta em `Falhou` (HTTP 409). | Transação circular sem valor; risco de bug em saldo. |

### 3.2 Invariantes da Conta

| # | Invariante |
|---|-----------|
| I-7 | Conta `Bloqueado` não pode ser remetente nem destinatário. |
| I-8 | Conta `Pendente` ou `Em Análise` não pode ser remetente. |
| I-9 | O estado de conta é verificado no momento do processamento, não no momento do login (evitar TOCTOU). |

> **TOCTOU (Time-of-Check-Time-of-Use):** Um usuário pode estar ativo no momento do login mas ter sua conta bloqueada antes do processamento da transferência. A validação de status deve ocorrer **dentro da transação atômica**, não apenas na autenticação.

---

## 4. Lente da Persistência

### 4.1 Como o estado é armazenado

| Entidade | Localização | Estado persistido | Risco de perda |
|----------|-------------|------------------|----------------|
| **Saldo do remetente** | Tabela `carteira` (banco relacional) | Reflete estado após commit | Baixo — banco com durabilidade ACID |
| **Saldo do destinatário** | Tabela `carteira` (banco relacional) | Reflete estado após commit | Baixo |
| **Registro da transação** | Tabela `transacoes` | Inserido na mesma transação atômica | Baixo |
| **Estado da transação (visão do usuário)** | Inferido da resposta HTTP + tabela de transações | Não há coluna `status` explícita no requisito | **Médio — lacuna: estado intermediário não persistido** |
| **Idempotency-Key** | Tabela `idempotency_keys` (recomendado) | Estado `Processando → Concluída/Falhou` | **Alto — risco de key "Processando" órfã em caso de crash** |

### 4.2 Risco de esquecimento de estado após crash

O cenário mais crítico ocorre quando a API processa a transação de banco com sucesso (commit) mas falha **antes** de retornar a resposta HTTP ao cliente:

```
Commit OK ──► API crasha ──► Cliente não recebe resposta ──► Timeout no cliente
                                    │
                                    ▼
                    Cliente faz retry com mesma Idempotency-Key
                                    │
                                    ▼
                    API encontra key = "Concluída" ──► retorna HTTP 200 (correto)
```

**Resultado:** O mecanismo de idempotência resolve corretamente — desde que a key tenha sido atualizada para `Concluída` **antes** do commit, ou na mesma transação.

**Risco real:** Se o crash ocorre **após** o commit do saldo mas **antes** de atualizar a Idempotency-Key:

```
UPDATE saldo ──► COMMIT ──► [CRASH] ──► key ainda em "Processando"
                                              │
                                              ▼
                               Retry do cliente ──► key em estado ambíguo
                               ¿ processar de novo? ¿ retornar erro? ¿ retornar 200?
```

**Mitigação:** A atualização da Idempotency-Key deve estar **dentro da mesma transação atômica** do débito/crédito/registro — garantindo que o estado da key e o estado do saldo são sempre consistentes.

### 4.3 Recuperação de falhas durante transições

| Ponto de falha | Estado resultante | Recuperação |
|----------------|------------------|-------------|
| **Falha antes do BEGIN** | Nenhuma alteração | Retry com mesma key — sem efeito colateral |
| **Falha após BEGIN, antes do commit** | Rollback automático do banco | Key não inserida (ou em `Processando` sem resultado) — ver risco de key órfã |
| **Falha após commit, antes de responder ao cliente** | Transação concluída no banco | Retry com mesma key retorna resultado da primeira execução (200) — correto |
| **Timeout de rede no cliente** | Estado no servidor desconhecido pelo cliente | Cliente usa mesma key; API retorna resultado cacheado — correto |

### 4.4 Key em estado "Processando" órfã

Se a API inseriu a key como `Processando` mas fez rollback antes de atualizá-la para `Concluída` ou `Falhou` (ex.: por deadlock ou exception não tratada), a key fica presa em `Processando`:

- Próximo retry com a mesma key encontra `Processando` — retornar erro ou aguardar?
- **Lacuna no requisito:** não há definição de comportamento para key em estado intermediário persistente.

**Recomendação:** Tratar inserção e atualização da key na mesma transação atômica — se o rollback ocorre, a key é removida junto. Se isso não for possível, definir TTL para estado `Processando` (ex.: após 30 segundos sem atualização, considerar falha).

---

## 5. Lacunas e Riscos Identificados

| # | Lacuna / Risco | Severidade | Origem no Requisito | Recomendação |
|---|----------------|-----------|--------------------|--------------|
| L-1 | **Transições de estado da conta não mapeadas** — o requisito lista os estados (ativo, bloqueado, pendente, em análise) mas não especifica quem e como altera esses estados | Alta | Seções 3.1, 3.2 | Definir quais eventos (admin, suporte, processo automático) causam cada transição; garantir que mudança de estado invalide sessões ativas (TOCTOU) |
| L-2 | **Estado "Processando" da key pode ficar órfão** — crash entre rollback e limpeza da key | Alta | Seção 8.2 | Manter inserção e atualização da key na mesma transação atômica; ou definir TTL para estado intermediário |
| L-3 | **Coluna de status da transação não especificada** — o requisito menciona `"status": "completed"` na resposta, mas não define se há coluna `status` na tabela de transações | Média | Seção 10.4 | Adicionar coluna `status` na tabela de transações com valores `completed` e `failed`; facilita auditoria e lista de recentes |
| L-4 | **Transições de estado da conta durante a transação (TOCTOU)** — conta pode ser bloqueada entre autenticação e processamento | Alta | Seções 3.1, 6.1 | Revalidar status da conta dentro da transação atômica, não apenas no middleware de autenticação |
| L-5 | **Estado "Falhou" sem distinção de causa** — o requisito distingue códigos HTTP, mas a tabela de transações não registra o motivo da falha | Baixa | Seção 8.3 | Registrar o código de erro (ex.: `INSUFFICIENT_BALANCE`) no log de auditoria e, opcionalmente, na tabela de transações |
| L-6 | **TTL da Idempotency-Key não definido** — keys acumulam indefinidamente | Média | Seção 8.2 | Definir política de expiração (ex.: 24h) e job de limpeza periódico |
| L-7 | **Estado da transação não consultável pelo usuário** — sem endpoint de consulta de status por transactionId | Baixa | Seção 10 | Avaliar endpoint `GET /api/v1/transfers/{transactionId}` no backlog para suporte e auditoria |

---

## 6. Diagrama Consolidado de Estados

### 6.1 Transação (visão do usuário no App)

```
                         ┌──────────────────────────────────┐
                         │         [Processando]            │
                         │  (indicador de loading ativo;    │
                         │   botão de envio desabilitado)   │
                         └──────────────┬───────────────────┘
                                        │
               ┌────────────────────────┴─────────────────────────┐
               │                                                   │
               ▼                                                   ▼
   ┌─────────────────────┐                           ┌────────────────────────┐
   │    [Concluída]      │                           │       [Falhou]         │
   │  HTTP 200           │                           │  HTTP 4xx / 5xx        │
   │  Exibir sucesso;    │                           │  Exibir mensagem;      │
   │  atualizar saldo e  │                           │  permitir retry com    │
   │  lista de recentes  │                           │  mesma Idempotency-Key │
   └─────────────────────┘                           └────────────────────────┘
           (final)                                           (final)
```

### 6.2 Idempotency-Key (visão interna da API)

```
[Inexistente] ──(INSERT atômico)──► [Processando]
                                         │
                      ┌──────────────────┤
                      │                  │
               (commit OK)          (rollback)
                      │                  │
                      ▼                  ▼
              [Concluída]           [Falhou]
              (resultado           (erro
               persistido)          persistido)
                      │                  │
                      └──────┬───────────┘
                             │
                      (retry com mesma key)
                             │
                             ▼
                   Retornar resultado cacheado
                   (sem reprocessar)
```

---

## 7. Recomendações por Prioridade

### Para o MVP (inegociável)

| # | Recomendação | Justificativa |
|---|-------------|---------------|
| R-1 | Inserir e atualizar a Idempotency-Key **dentro da mesma transação atômica** do débito/crédito/registro | Previne key órfã em `Processando` após crash |
| R-2 | Revalidar o status da conta do remetente **dentro da transação atômica** (não apenas no middleware de autenticação) | Previne TOCTOU — conta pode ser bloqueada entre autenticação e processamento |
| R-3 | Adicionar coluna `status` na tabela de transações com valores `completed` / `failed` | Habilita lista de recentes com status correto e auditoria |
| R-4 | Definir comportamento explícito para key em estado `Processando` persistente (ex.: considerar falha após TTL de 30s) | Previne bloqueio indefinido em retry |

### Para o Backlog

| # | Recomendação |
|---|-------------|
| R-5 | Definir e documentar as transições de estado da conta do usuário (quem bloqueia, quem ativa, quais eventos disparam cada transição) |
| R-6 | Definir TTL da Idempotency-Key (ex.: 24h) e job de limpeza |
| R-7 | Avaliar endpoint `GET /api/v1/transfers/{transactionId}` para consulta de status por ID |
| R-8 | Registrar código de erro no log de auditoria e, opcionalmente, na tabela de transações para rastreabilidade de falhas |

---

## 8. Heurísticas Complementares Recomendadas ⚡

### Multi-User (Concorrência)
**Já aplicada:** Ver [MULTI_USER_REQ_INICIAL_V2.md](./MULTI_USER_REQ_INICIAL_V2.md).
**Conexão:** Os locks de concorrência (`SELECT FOR UPDATE`) são o mecanismo que garante a transição `Processando → Concluída` sem race condition no saldo.

### CRUD
**Quando aplicar:** Para detalhar as operações de criação e consulta de transações, garantindo que o estado `Concluída` seja sempre lido da mesma fonte de verdade (tabela de transações) tanto no App quanto no painel web.

**Recomendação:** "Aplicar a heurística **CRUD** para mapear as operações de Create (inserção de transação), Read (lista de recentes, consulta de saldo) e verificar que não existem operações de Update ou Delete não intencionais sobre transações concluídas."

### FAILURE
**Quando aplicar:** Para detalhar os cenários de falha das transições de estado — em especial crash durante `Processando`, timeout de lock e rollback por deadlock.

**Recomendação:** "Aplicar a heurística **FAILURE** para mapear sistematicamente cada ponto de falha nas transições de estado e garantir que a recuperação em cada cenário resulta em estado consistente."

---

## Referências

- **Requisito analisado:** [REQ_INICIAL_V2.md](../requisito-revisado/REQ_INICIAL_V2.md)
- **Heurística aplicada:** [StateAnalysis.md](../../Skill/Cap05_design/StateAnalysis.md)
- **Análise complementar já realizada:** [MULTI_USER_REQ_INICIAL_V2.md](./MULTI_USER_REQ_INICIAL_V2.md)
- **Análise complementar já realizada:** [DEPENDENCIES_REQ_INICIAL_V2.md](./DEPENDENCIES_REQ_INICIAL_V2.md)
- **Próximas heurísticas recomendadas:** CRUD, FAILURE (ver seção 8)
