# Relatório: Count (0, 1, Muitos) — REQ_INICIAL_V2 (QualiPoints)

**Requisito analisado:** Envio de QualiPoints entre usuários (REQ_INICIAL_V2).
**Skill aplicada:** Heurística Count (0, 1, Muitos) — Arquitetura de Código
**Data:** 2026-03-29
**Status:** Análise de arquitetura concluída

**Fonte analisada:** Requisito revisado v2 — [qualitpoints.md](../../input/qualitpoints.md)

---

## 1. Coleções e Volumes Identificados no Requisito

Antes de aplicar os cenários Count, mapeamos todas as coleções, entidades com multiplicidade e volumes definidos (ou ausentes) no requisito:

| Coleção / Entidade | Limite Definido | Observação |
|--------------------|-----------------|------------|
| **Lista de transações recentes** | Máximo 10 por consulta | Fixo no MVP; sem paginação definida |
| **QualiPoints por transação** | Mínimo 1 / Máximo 10.000 | Inteiro; limites explícitos |
| **Idempotency Keys armazenadas** | Não definido | Sem TTL ou política de expiração |
| **Transações simultâneas do mesmo remetente** | Não limitado | Concorrência tratada por atomicidade |
| **Destinatários por transferência** | Exatamente 1 | Não há transferência multi-destinatário |
| **Tentativas de retry por intenção** | Não limitado | Qualquer número com a mesma Idempotency-Key |
| **Usuários ativos no sistema** | Não definido | Sem estimativa de volume na base |
| **Transações históricas por usuário** | Não definido | A listagem retorna apenas as 10 mais recentes |

---

## 2. Análise por Coleção

### 2.1 Lista de Transações Recentes

O requisito define "últimas 10 transações do usuário autenticado, ordenadas por data/hora mais recente primeiro" (seção 6.3).

#### Cenário Zero (0 transações)

| Questão | Análise |
|---------|---------|
| O que acontece quando a lista está vazia? | **Lacuna:** o requisito não define o comportamento da API para um usuário sem nenhuma transação |
| Como a UI se comporta com zero itens? | **Lacuna:** nenhuma mensagem de empty state está especificada para o App ou painel web |
| A API retorna `[]` ou `null`? | **Não definido** — a diferença é importante: `null` pode gerar NullPointerException em clientes sem tratamento adequado |
| Há call-to-action de onboarding? | Não abordado — usuário novo não sabe que pode enviar QualiPoints |

**Risco:** Clientes (App e painel web) sem tratamento explícito de lista vazia podem lançar exceção ao tentar iterar sobre `null`, ou exibir tela em branco sem mensagem orientativa.

**Recomendação:** Especificar resposta explícita `[]` (array vazio) para zero transações, e definir mensagem de empty state para UI ("Você ainda não realizou nenhuma transferência. Envie seus primeiros QualiPoints!").

---

#### Cenário Um (1 transação)

| Questão | Análise |
|---------|---------|
| A listagem funciona corretamente com exatamente um item? | O comportamento de array com 1 elemento deve ser validado — risco de off-by-one em ordenação |
| Há otimização possível para o caso único? | Query com `LIMIT 10` retorna 1 resultado — sem overhead; nenhuma otimização especial necessária |
| A UI exibe corretamente lista com 1 item? | **Lacuna leve:** nenhum layout específico é descrito para lista com 1 item; risco de UI quebrada se o design assumiu mínimo de 2 itens |

**Risco baixo:** O comportamento para 1 item é coberto implicitamente pelo caso geral. Verificar apenas que a query não usa `LIMIT 10 OFFSET 0` de forma que ignore o registro único.

---

#### Cenário Muitos (N transações)

| Questão | Análise |
|---------|---------|
| O que constitui "muitos" neste contexto? | Usuário ativo: dezenas a centenas de transações por mês; sistema maduro: milhões de registros na tabela |
| A query de "últimas 10 por data" usa índice? | **Lacuna crítica:** o requisito não especifica índice em `created_at` (ou `completed_at`) na tabela de transações |
| O que acontece com usuários que têm mais de 10 transações? | A listagem retorna sempre as 10 mais recentes — correto pelo requisito. Porém, não há paginação definida para navegar além das 10 primeiras |
| O tamanho da tabela de transações impacta a consulta? | Sem índice em `(user_id, created_at)`, uma `SELECT ... WHERE user_id = ? ORDER BY created_at DESC LIMIT 10` realiza full scan em tabelas grandes — degradação progressiva |
| Há estratégia de particionamento ou arquivamento? | Não definido — risco a longo prazo com crescimento ilimitado da tabela |

**Riscos identificados:**
- **Performance de leitura degradante:** Sem índice composto em `(user_id, created_at DESC)`, a consulta das últimas 10 transações tem custo `O(N)` sobre todos os registros do usuário.
- **Sem paginação no MVP:** O requisito fixa 10 registros. À medida que o produto cresce, a ausência de paginação dificulta a evolução para extrato completo.
- **Sem política de arquivamento:** Tabela de transações cresce indefinidamente.

**Recomendação:** Criar índice composto `(user_id, created_at DESC)` na tabela de transações desde o MVP. Planejar estratégia de paginação (cursor-based) para evolução futura.

---

### 2.2 Idempotency Keys Armazenadas

#### Cenário Zero (0 keys)

| Questão | Análise |
|---------|---------|
| Primeira requisição do sistema (nenhuma key cadastrada) | Comportamento correto: key é inserida e processamento prossegue |
| Query de verificação é otimizada para ausência? | `SELECT ... WHERE key = ?` com UNIQUE constraint retorna rapidamente — sem impacto |

**Sem lacunas no cenário zero.**

---

#### Cenário Um (1 key — fluxo normal de retry)

| Questão | Análise |
|---------|---------|
| Retry com a mesma key encontra o resultado cacheado? | Sim — comportamento correto definido no requisito (seção 8.2) |
| A key em estado "Processando" (crash durante transação) retorna o quê? | **Lacuna:** não há comportamento definido para key presa em estado intermediário |

---

#### Cenário Muitos (N keys acumuladas)

| Questão | Análise |
|---------|---------|
| Quantas keys são acumuladas? | **Não definido** — sem TTL, as keys crescem indefinidamente: 1 key por intenção de envio de cada usuário |
| Há política de expiração? | **Lacuna crítica:** o requisito não define TTL ou job de limpeza |
| Impacto em disco e performance de lookup? | Com UNIQUE constraint e índice no campo da key, o lookup é `O(1)` — performance de leitura adequada mesmo com muitas keys. O risco é de **crescimento ilimitado de armazenamento** |
| O que acontece com keys de tentativas abandonadas? | Sem TTL, keys de transações que falharam permanecem para sempre no banco |

**Risco:** Acúmulo indefinido de Idempotency-Keys gera crescimento de armazenamento proporcional ao volume de transações. Para 1 milhão de transações/mês sem TTL, a tabela acumula milhões de registros por ano.

**Recomendação:** Definir TTL de 24h a 7 dias para keys. Implementar job periódico de limpeza de keys expiradas. Definir o comportamento explícito de retry após expiração da key (nova intenção = nova key).

---

### 2.3 QualiPoints por Transação (Valores)

#### Cenário Zero (0 QualiPoints)

| Questão | Análise |
|---------|---------|
| Valor zero é permitido? | **Não** — API rejeita com HTTP 400 e código `INVALID_AMOUNT` (seção 10.4) |
| O que acontece com `amount: 0`? | Validação server-side retorna erro imediatamente; sem débito ou crédito |

**Sem lacunas — comportamento definido corretamente.**

---

#### Cenário Um (1 QualiPoint — valor mínimo)

| Questão | Análise |
|---------|---------|
| Valor 1 é aceito? | **Sim** — mínimo definido no requisito (seção 4.2) |
| Há tratamento especial para valor mínimo? | Não necessário — a lógica de validação `1 ≤ amount ≤ 10.000` cobre este caso naturalmente |

**Sem lacunas — caso coberto explicitamente pelo requisito.**

---

#### Cenário Muitos (10.000 QualiPoints — valor máximo)

| Questão | Análise |
|---------|---------|
| Valor máximo por transação é 10.000? | **Sim** — definido no requisito (seção 4.2) |
| O que acontece com `amount: 10.001`? | API rejeita com HTTP 400 e código `INVALID_AMOUNT` |
| Há limite de volume total por período? | **Lacuna (backlog):** o requisito reconhece explicitamente que limites por dia/mês ficam para backlog (seção 4.3) |
| Overflow possível em campo de saldo? | **Risco de tipo de dado:** se saldo for armazenado como `INT` (máximo ~2,1 bilhões), e o sistema crescer muito, há risco de overflow. Não especificado no requisito |

**Risco a longo prazo:** Sem limite de volume por período, um usuário pode acumular saldo ilimitado. Verificar o tipo de dado do saldo no banco (recomendado: `BIGINT` ou equivalente).

---

### 2.4 Transações Simultâneas do Mesmo Remetente

#### Cenário Zero (0 envios simultâneos)

Fluxo padrão sem concorrência — coberto pela operação atômica.

---

#### Cenário Um (1 envio em andamento)

Fluxo normal esperado — a transação atômica processa e retorna. Sem lacunas.

---

#### Cenário Muitos (N envios simultâneos do mesmo remetente)

| Questão | Análise |
|---------|---------|
| Duas requisições com IDs diferentes chegam ao mesmo tempo para o mesmo remetente? | Ambas passam pela autenticação; ambas leem o mesmo saldo; qual processa primeiro? |
| O lock de saldo é adequado para N requisições simultâneas? | **Lacuna:** o mecanismo de lock (pessimistic `SELECT FOR UPDATE`, optimistic locking com versão) não está especificado |
| N requisições com a mesma Idempotency-Key chegam simultaneamente? | Race condition na inserção da key — sem UNIQUE constraint, ambas podem ser inseridas e processadas |
| O rate limiting previne abuso? | **Lacuna (backlog):** o requisito menciona rate limit como backlog (seção 4.3) |

**Risco de saldo negativo:** Sem especificação do mecanismo de lock, duas transferências simultâneas do mesmo remetente podem passar na validação de saldo com o mesmo valor lido, resultando em saldo negativo.

---

### 2.5 Destinatários por Transferência

#### Cenário Zero (0 destinatários)

`recipientId` ausente no body -> HTTP 400 (validação de campo obrigatório). Coberto pelo contrato da API.

---

#### Cenário Um (1 destinatário — obrigatório por design)

Fluxo esperado e único permitido no MVP. Sem lacunas.

---

#### Cenário Muitos (N destinatários)

Não permitido no MVP — o requisito restringe a exatamente 1 destinatário por transação. Transferências em lote ficam para backlog. **Sem lacunas para o MVP.**

---

## 3. Bugs de Limite Identificados

| # | Bug Potencial | Cenário Count | Severidade | Origem |
|---|---------------|---------------|------------|--------|
| B-1 | **Divisão por zero na média de saldo** | Zero transações | Alta | Qualquer cálculo de média sobre lista vazia sem tratamento retorna NaN/Infinity |
| B-2 | **Off-by-one na listagem** | Um item | Média | Ordenação `DESC LIMIT 10` pode excluir o item mais recente se `created_at` for igual em duas transações (tie-breaking não definido) |
| B-3 | **Saldo negativo em N envios simultâneos** | Muitos simultâneos | Alta | Ausência de especificação de lock no saldo permite race condition |
| B-4 | **Overflow no tipo de dado do saldo** | Muitos créditos | Baixa (longo prazo) | Tipo de dado do saldo não especificado |
| B-5 | **Acúmulo ilimitado de Idempotency-Keys** | Muitas keys | Média | Sem TTL definido — crescimento de armazenamento proporcional ao volume |
| B-6 | **Query sem índice degrada progressivamente** | Muitas transações | Alta (produção) | Ausência de índice em `(user_id, created_at)` na tabela de transações |

---

## 4. Cenários de Teste Derivados da Heurística Count

A heurística Count (0, 1, Muitos) fornece cenários-chave para estruturar testes em todas as camadas. Para gerar o relatório completo de estratégia de testes e implementar os casos de teste, consulte [TEST_STRATEGY.md](../../docs/guides/TEST_STRATEGY.md) com o requisito e esta análise como input.

### 4.1 Cenários de Zero (ausência)

- API retorna `[]` (array vazio) para usuário sem transações — não `null`
- Transferência com `amount: 0` retorna HTTP 400 com código `INVALID_AMOUNT`
- UI exibe empty state com mensagem orientativa quando não há transações
- Transferência com saldo zero retorna HTTP 422 com código `INSUFFICIENT_BALANCE`
- `recipientId` ausente retorna HTTP 400

### 4.2 Cenários de Um (caso singular)

- Lista de transações com exatamente 1 item exibe corretamente
- Transferência de valor mínimo (1 QualiPoint) é aceita e processada
- Retry com mesma Idempotency-Key retorna resultado cacheado (mesmo corpo e código HTTP)
- Transferência que drena saldo para exatamente 0 é concluída com sucesso

### 4.3 Cenários de Muitos (alto volume)

- Lista retorna apenas 10 transações quando usuário possui 50+ no histórico
- Performance da query de listagem com 100k transações na tabela (< 200ms)
- Transferência com valor máximo (10.000 QualiPoints) é aceita
- Valor acima do máximo (10.001) é rejeitado com HTTP 400
- N envios simultâneos do mesmo remetente não resultam em saldo negativo
- Lookup de Idempotency-Key permanece performático com milhões de keys no banco
- Ordenação das transações é determinística mesmo com timestamps iguais

> **Para implementação dos testes:** Utilize os guides em [docs/guides/](../../docs/guides/) — TEST_UNIT_GUIDE, TEST_INTEGRATION_GUIDE, TEST_SERVICE_GUIDE, TEST_COMPONENT_GUIDE e TEST_E2E_GUIDE — com os cenários acima como input.

---

## 5. Lacunas e Riscos Consolidados

| # | Lacuna / Risco | Cenário Count | Severidade | Recomendação |
|---|----------------|---------------|------------|--------------|
| L-1 | **Empty state da lista de transações não especificado** | Zero | Média | Definir resposta `[]` na API e mensagem de empty state na UI |
| L-2 | **Índice em `(user_id, created_at)` não especificado** | Muitos | Alta | Adicionar índice composto desde o MVP para evitar degradação progressiva |
| L-3 | **TTL da Idempotency-Key não definido** | Muitos | Média | Definir política de expiração (24h a 7 dias) e job de limpeza |
| L-4 | **Mecanismo de lock no saldo não especificado** | Muitos (simultâneos) | Alta | Especificar pessimistic locking (`SELECT FOR UPDATE`) ou optimistic locking (versão/etag) |
| L-5 | **Sem paginação para além das 10 transações** | Muitos | Média (backlog) | Planejar cursor-based pagination para evolução do extrato completo |
| L-6 | **Tipo de dado do saldo não especificado** | Muitos (longo prazo) | Baixa | Usar `BIGINT` para saldo; evitar overflow futuro |
| L-7 | **Sem rate limiting por usuário/período** | Muitos (simultâneos) | Média (backlog) | Implementar rate limiting no MVP ou definir prioridade no backlog |
| L-8 | **Tie-breaking em `created_at` não definido** | Um / Muitos | Baixa | Definir campo de ordenação secundário (ex.: `id ASC`) quando timestamps forem iguais |

---

## 6. Recomendações por Prioridade

### Para o MVP (inegociável)

| # | Recomendação | Justificativa |
|---|-------------|---------------|
| R-1 | Criar índice composto `(user_id, created_at DESC)` na tabela de transações | Previne degradação progressiva de performance da listagem com crescimento de dados |
| R-2 | Especificar que a API retorna `[]` (array vazio, não `null`) para zero transações | Evita NullPointerException em clientes; padroniza contrato da API |
| R-3 | Especificar mecanismo de lock no saldo (pessimistic ou optimistic locking) | Previne saldo negativo em envios simultâneos do mesmo remetente |
| R-4 | Usar tipo `BIGINT` (ou equivalente) para saldo no banco | Previne overflow a longo prazo |

### Para o Backlog

| # | Recomendação |
|---|-------------|
| R-5 | Definir TTL da Idempotency-Key (24h-7 dias) e job de limpeza periódico |
| R-6 | Planejar cursor-based pagination para extrato completo (além das 10 transações) |
| R-7 | Implementar rate limiting por usuário (N transações por minuto/hora) |
| R-8 | Definir campo de ordenação secundário para tie-breaking em `created_at` |
| R-9 | Definir estratégia de arquivamento de transações antigas (particionamento por data ou tabela de histórico) |

---

## 7. Heurísticas Complementares Recomendadas

### CRUD
**Conexão com Count:** O cenário "Muitos" das transações relaciona diretamente com as operações de Read (query de listagem) e Create (inserção atômica). A heurística CRUD complementa a análise com foco em segurança de acesso (quem pode ler quais transações), controle de concorrência nas escritas e imutabilidade dos registros.

**Recomendação:** Aplicar **CRUD** para garantir que: (a) apenas o próprio usuário lê suas transações (Read com controle de acesso); (b) não há operações de Update ou Delete sobre transações concluídas; (c) a criação da transação está protegida por lock adequado.

### FAILURE
**Conexão com Count:** O cenário "Muitos" de envios simultâneos e o comportamento "Zero" de keys no banco após limpeza são pontos de falha críticos.

**Recomendação:** Aplicar **FAILURE** para mapear sistematicamente o que acontece quando há N envios simultâneos e apenas alguns são bem-sucedidos, e quando o job de limpeza de keys é executado durante uma transação em andamento.

### Position (Primeiro, Último, Meio)
**Conexão com Count:** A listagem das "últimas 10" é diretamente relacionada à posição dos registros — primeiro (mais recente), último (décimo mais recente) e meio.

**Recomendação:** Aplicar **Position** para garantir que a transação recém-criada aparece imediatamente no topo da listagem, que a 11a transação não aparece, e que o tie-breaking em `created_at` é determinístico.

---

## Referências

- **Requisito analisado:** [qualitpoints.md](../../input/qualitpoints.md)
- **Skill aplicada:** [Count.md](../../skills/heuristic-guide-architeture-code/Count.md)
- **Estratégia de testes:** [TEST_STRATEGY.md](../../docs/guides/TEST_STRATEGY.md)
- **Próximas heurísticas recomendadas:** CRUD, FAILURE, Position (ver seção 7)
