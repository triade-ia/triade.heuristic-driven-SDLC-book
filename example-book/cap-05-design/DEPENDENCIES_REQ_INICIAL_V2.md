# Relatório: Dependencies — REQ_INICIAL_V2 (QualiPoints)

**Requisito analisado:** Envio de QualiPoints entre usuários (REQ_INICIAL_V2).
**Skill aplicada:** Cap05_design — Dependencies
**Data:** 2026-02-18
**Status:** Análise de design concluída

---

## 1. Mapeamento de Relacionamentos (O "Tem um")

### Relacionamento de Dados

A funcionalidade de envio de QualiPoints envolve as seguintes entidades e seus relacionamentos estruturais:

| Entidade / Agregado | Relacionamento | Entidade Relacionada | Tipo de Referência |
|---------------------|----------------|----------------------|--------------------|
| `Transferência` | TEM UM | `Remetente (Usuário)` | Implícito via token de autenticação (não enviado no body) |
| `Transferência` | TEM UM | `Destinatário (Usuário)` | Por ID (`recipientId` no body) |
| `Transferência` | TEM UM | `Valor` | Inteiro embutido (1–10.000 QualiPoints) |
| `Transferência` | TEM UMA | `Transação (Registro)` | Gerada na confirmação; retorna `transactionId` |
| `Transferência` | TEM UMA | `Idempotency-Key` | Header da requisição; ligada à intenção de envio |
| `Usuário` | TEM UM | `Saldo (Carteira)` | Tabela de saldo/wallet no banco relacional |
| `Usuário` | TEM UMA | `Lista de Transações Recentes` | Coleção derivada (últimas 10, ordenadas por data DESC) |
| `Transação` | TEM UM | `Status` | `completed` após envio bem-sucedido |

**Padrão de acoplamento predominante:** referência por ID entre entidades. Não há denormalização explicitada no requisito, mas a resposta de sucesso (`200 OK`) replica dados do destinatário e valor, sugerindo leitura desnormalizada na resposta.

---

### Dependência de Fluxo

Serviços ou componentes cujo funcionamento é **vital** para que o envio seja concluído com sucesso:

| # | Componente / Serviço | Papel na Operação | Criticidade |
|---|----------------------|-------------------|-------------|
| 1 | **Serviço de Autenticação** | Valida o token Bearer; identifica o remetente | Crítico — sem autenticação, operação bloqueada na origem |
| 2 | **Serviço de Usuários** | Verifica existência e status ativo do destinatário | Crítico — falha impede validação do destinatário |
| 3 | **Serviço de Saldo / Carteira** | Consulta saldo atual; executa débito (remetente) e crédito (destinatário) | Crítico — coração da operação financeira |
| 4 | **Banco de Dados Relacional** | Persiste saldo atualizado + registro de transação de forma atômica | Crítico — sem DB, a atomicidade não existe |
| 5 | **Controle de Idempotência** | Armazena resultado por `Idempotency-Key`; retorna resultado cacheado em retentativa | Alto — ausência gera risco de débito duplo |
| 6 | **App Mobile** | Inicia a requisição; exibe feedback de estado ao usuário | Consumidor — não é serviço interno, mas orquestra UX |
| 7 | **Painel Web** | Consome lista de transações e saldo via mesma API (ou leitura do mesmo banco) | Baixo no fluxo de envio — impacto apenas em consulta |

---

## 2. Matriz de Propagação de Impacto (O "Efeito Dominó")

### Mudança na Origem

Riscos de inconsistência se um dado-raiz for alterado durante ou após a operação:

| Dado Alterado | Risco de Inconsistência | Mitigação Presente no Requisito |
|---------------|-------------------------|----------------------------------|
| **Saldo do remetente** alterado entre a leitura de saldo e o débito (concorrência) | Saldo pode se tornar negativo se duas transferências simultâneas passarem na validação | Transação atômica de banco + contenção via lock; idempotency key não resolve race condition entre requisições distintas |
| **Status do destinatário** alterado entre a validação e o crédito | Crédito pode ser realizado numa conta que acabou de ser bloqueada | Operação atômica: se qualquer passo falhar (inclusive status check interno), há rollback |
| **ID do usuário** alterado (ex.: migração de base) | Referências de `recipientId` e `senderId` nas transações antigas ficam órfãs | Não coberto no requisito (backlog de consistência histórica) |
| **Limite de valor (10.000)** alterado via configuração em produção | Transações aprovadas com valor antigo ficam no extrato sem indicação de limite vigente | Não coberto — sem versionamento de regras de negócio nas transações |

---

### Falha de Vizinho

Comportamento da funcionalidade diante da indisponibilidade de cada dependência:

| Dependência que Falha | Impacto | Comportamento Definido no Requisito |
|-----------------------|---------|--------------------------------------|
| **Serviço de Autenticação** | Morte da funcionalidade — 100% dos envios falham com 401 | Não há degradação graciosa possível; requisito não define fallback |
| **Serviço de Usuários** | Morte da funcionalidade — impossível validar destinatário | 404 retornado; sem fallback ou cache de status |
| **Banco de Dados** | Morte da funcionalidade — sem persistência, sem atomicidade | Retorna 5xx; cliente pode retry com mesma Idempotency-Key |
| **Controle de Idempotência** | Risco de débito duplo em retentativas | Requisito exige idempotência, mas não descreve o mecanismo de armazenamento da key (em memória, banco, Redis) |
| **Painel Web** | Degradação parcial — envio via App não é afetado; apenas consulta | App e painel são canais independentes; impacto isolado |
| **App Mobile** | Sem impacto na API — a API continua funcional para outros clientes | Requisito não menciona outros clientes, mas a API REST é stateless |

---

### Concorrência

Gargalos identificados em recursos compartilhados:

| Cenário de Concorrência | Ponto de Contenção | Risco |
|-------------------------|--------------------|-------|
| **Dois envios simultâneos do mesmo remetente** (duplo clique ou dois dispositivos) | Tabela de saldo/carteira do remetente — lock de escrita | Sem lock adequado, ambas as transações podem ler o mesmo saldo e ambas passarem na validação, gerando saldo negativo |
| **Mesma Idempotency-Key enviada em paralelo** (duas requisições chegando ao mesmo tempo antes de qualquer resultado ser persistido) | Controle de idempotência — race condition na inserção da key | Sem mecanismo de lock na inserção da key (ex.: UNIQUE constraint), ambas as requisições podem processar |
| **Crédito e débito simultâneos no mesmo usuário** (A → B e C → B ao mesmo tempo) | Tabela de saldo/carteira do destinatário | Menos crítico (créditos simultâneos), mas exige serialização correta |
| **Consulta de lista recente durante envio** | Tabela de transações — leitura durante escrita | Baixo risco com isolamento de leitura no banco; no MVP sem filtros avançados o impacto é mínimo |

---

## 3. Gatilhos de Ação Manual (Investigação Complementar) ⚡

Com base nos relacionamentos e dependências identificados, as seguintes heurísticas complementares **devem ser aplicadas** para completar a análise de design:

---

**⚡ CRUD — Persistência de Dados**

Para complementar a análise dos relacionamentos de dados identificados, aplique a heurística **CRUD** nas seguintes entidades:

- **Tabela de Transações:** Quais operações existem (Create na confirmação; Read na listagem)? Há Update ou Delete de transações? Quem tem permissão de cada operação?
- **Tabela de Saldo/Carteira:** O saldo é atualizado diretamente (Update) ou via eventos de débito/crédito registrados (Create em tabela de movimentação)? Isso define o modelo de consistência eventual vs. imediata.
- **Controle de Idempotência:** Como a Idempotency-Key é armazenada (Create) e consultada (Read)? Ela expira (Delete)? Qual o TTL?

---

**⚡ Count (0, 1, Muitos) — Coleções e Volumes**

Para validar os limites e a escalabilidade, aplique a heurística **Count** em:

- **Lista de Transações Recentes:** O requisito define "últimas 10" — mas o que ocorre quando há **0 transações** (conta nova), **1 transação** ou quando o sistema cresce para **milhões de registros**? A query de "últimas 10" usa índice em data/hora?
- **Idempotency Keys armazenadas:** Quantas keys são acumuladas? Há limite ou política de expiração? O que acontece com **0 keys** (primeira requisição), **1 key** (fluxo normal) ou **muitas keys** acumuladas sem TTL?
- **Múltiplos destinatários:** O requisito só permite 1 destinatário por transferência — isso está correto para todos os casos de uso futuros?

---

**⚡ Position (Primeiro, Último, Meio) — Ordenação de Listas**

Para investigar o comportamento da lista de transações recentes, aplique a heurística **Position** em:

- **Primeira transação do usuário:** A listagem retorna corretamente uma lista com 1 item?
- **Última transação (mais recente):** É garantido que a transação recém-criada aparece no topo na próxima consulta (consistência de leitura imediata ou eventual)?
- **Transação do meio:** Se o usuário tem exatamente 10 transações e faz mais uma, a mais antiga é descartada da lista? O requisito diz "últimas 10" — como é tratado o cursor/paginação para transações além da 10ª?

---

**⚡ Selection (Alguns, Nenhum, Todos) — Filtros e Estados Variáveis**

Para garantir a integridade dos filtros de estado, aplique a heurística **Selection** em:

- **Status do usuário (ativo / bloqueado / pendente / em análise):** O requisito cobre `ativo` e `bloqueado`. O que ocorre com os demais estados? Todos os estados não-ativos devem receber o mesmo erro 403, ou há distinção de mensagem?
- **Transações do extrato:** A listagem retorna transações em que o usuário é remetente **E** destinatário ("alguns"), apenas remetente ("nenhum como destinatário"), ou todas independentemente do papel? O requisito menciona "como remetente ou destinatário" — isso está implementado no filtro da query?
- **Conta do destinatário:** O requisito trata destinatário inexistente (404) e inativo (404 com mesmo código). Se a conta existir mas estiver "em análise" ou "pendente", qual o comportamento esperado?

---

## 4. Decisões de Desacoplamento e Resiliência

### Isolamento

Estratégias para impedir que a falha de um elo bloqueie o usuário ou corrompa dados:

| Estratégia | Aplicação no Requisito | Status no MVP |
|------------|------------------------|---------------|
| **Idempotency Key** | Evita débito duplo em retry por timeout/falha de rede | ✅ Definido no requisito |
| **Operação Atômica (Transação de BD)** | Garante que débito, crédito e registro ocorram juntos ou nenhum ocorra | ✅ Definido no requisito |
| **Timeout com resposta explícita (504/408)** | Evita que o cliente fique em espera indefinida; permite retry consciente | ✅ Definido (10 segundos) |
| **Isolamento de canais (App vs. Painel Web)** | Falha no painel não afeta o envio via App | ✅ Arquitetura separada por design |
| **Lock de escrita no saldo** | Evita saldo negativo por concorrência | ⚠️ Implícito na atomicidade, mas mecanismo (pessimistic lock, optimistic lock, SELECT FOR UPDATE) não está especificado |
| **TTL / expiração da Idempotency Key** | Libera armazenamento e define janela de segurança para retry | ❌ Não definido — lacuna no requisito |
| **Circuit Breaker para dependências externas** | Impede cascata de falhas se Serviço de Usuários ou Auth cair | ❌ Não definido — backlog de resiliência |

---

### Contrato

Análise de comunicação síncrona vs. assíncrona por dependência:

| Dependência | Contrato Atual (MVP) | Risco | Evolução Sugerida (Backlog) |
|-------------|----------------------|-------|------------------------------|
| **App → API REST** | Síncrono — cliente aguarda resposta em até 10s | Experiência degradada se processamento demorar; spinner bloqueante | Manter síncrono no MVP; avaliar webhook para notificações futuras |
| **API → Banco de Dados** | Síncrono — transação atômica em tempo real | Gargalo se o banco estiver sob alta carga | Considerar réplica de leitura para listagem de transações |
| **API → Painel Web** | Assíncrono por design — painel faz polling/refresh manual | Dados do painel podem estar desatualizados entre a transação e o refresh | WebSocket ou Server-Sent Events para atualização em tempo real (backlog) |
| **Controle de Idempotência → Armazenamento** | Síncrono — key deve ser verificada/gravada antes do processamento | Race condition se o store de idempotência não tiver garantia de atomicidade | Usar UNIQUE constraint no banco ou lock atômico (ex.: Redis SETNX) |

---

## 5. Riscos de Acoplamento (Consolidado)

| Risco | Severidade | Probabilidade | Mitigação Necessária |
|-------|-----------|---------------|----------------------|
| **Race condition no saldo** — duas transferências simultâneas passam na validação com mesmo saldo | Alta | Média | Especificar mecanismo de lock (pessimistic ou optimistic locking) |
| **Ausência de TTL na Idempotency Key** — keys acumulam indefinidamente | Média | Alta | Definir política de expiração (ex.: 24h ou 7 dias) |
| **Acoplamento forte ao Serviço de Autenticação** — sem fallback ou cache | Alta | Baixa | Aceitável no MVP; monitorar SLA do serviço de auth |
| **Mecanismo de idempotência não especificado** — risco de race condition na inserção da key | Alta | Média | Especificar armazenamento (BD com UNIQUE constraint ou Redis SETNX) |
| **Status intermediários de conta não mapeados** (pendente, em análise) | Média | Alta | Aplicar heurística Selection para cobrir todos os estados |
| **Listagem sem índice explícito** — query "últimas 10 por data" pode ser lenta em produção | Média | Alta | Aplicar heurística Count; garantir índice em `created_at` na tabela de transações |
| **Consistência eventual no painel web** — saldo/extrato desatualizado após envio | Baixa | Alta | Aceitável no MVP; documentar comportamento esperado para o usuário |

---

## Referências

- **Requisito analisado:** [REQ_INICIAL_V2.md](../requisito-revisado/REQ_INICIAL_V2.md)
- **Heurística aplicada:** [Dependencies.md](../../Skill/Cap05_design/Dependencies.md)
- **Próximas heurísticas recomendadas:** CRUD, Count, Position, Selection (ver seção 3)
