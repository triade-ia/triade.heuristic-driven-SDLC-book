---
name: analise-failure-transferencia-qualipoints
description: Análise de cenários de falha do requisito de transferência de QualiPoints usando a heurística FAILURE. Identifica gaps de especificação, riscos de impacto e oportunidades de melhoria nas sete dimensões.
---

# Análise FAILURE — Transferência de QualiPoints

**Requisito analisado:** REQ_INICIAL_V2.md — Envio de QualiPoints entre usuários  
**Data:** 2025-02-21  
**Status:** Análise completa com recomendações  
**Aplicação:** Refinamento, codificação e elaboração de testes de cenários de falha

---

## Sumário Executivo

O requisito REQ_INICIAL_V2 define um fluxo síncrono de transferência de QualiPoints com boas práticas de autenticação, validação server-side, atomicidade e idempotência. Entretanto, a análise FAILURE revela **gaps críticos em especificação de falha**, especialmente em:

- **Functional:** comportamento em cascata quando dependências falham; modo degradado não está definido.
- **Appropriate:** alguns cenários de erro não mapeiam a resposta apropriada (ex.: falha de banco durante transação atômica).
- **Impact:** impacto em integridade de dados não é completamente explorado em race conditions.
- **Log:** falta padronização de logs estruturados com contexto distribuído (correlation ID).
- **UI:** comportamento em timeout, estados inconsistentes e mensagens de erro não são totalmente especificadas.
- **Recovery:** algumas falhas temporárias permitem retry, outras deixam o sistema em estado indefinido.
- **Emotions:** mensagens de erro estão presentes, mas tom e transparência em falhas de dependência externas (rede, banco) não são refinados.

Este documento mapeia **70+ cenários de falha** através das sete dimensões e recomenda **refinamentos essenciais** antes de codificação.

---

## 1. FUNCTIONAL — Comportamento em Falha e Isolamento

### 1.1 Cenários de falha e funcionalidade

| Cenário | Descrição | Comportamento esperado (atual) | Gap ou risco |
|---------|-----------|--------|--------|
| **Banco de dados indisponível** | API não consegue conectar ao DB durante `POST /api/v1/transfers` | ? (não especificado) | **CRÍTICO**: Requisito não define: retorna 503 + rollback automático? Qual o estado da transação se DB sai do ar no meio da atomicidade? |
| **Falha ao debitar remetente (constraint violado, deadlock)** | Transação falha na primeira parte (débito) | Rollback + erro apropriado ao cliente | ✓ Esperado (atomicidade), mas não há detalhe de qual código HTTP retornar. |
| **Falha ao creditar destinatário (conta foi bloqueada entre validação e crédito)** | Validação passou, mas ao tentar creditar, conta está bloqueada | Rollback + erro apropriado | ✓ Atomicidade garante, mas **quem notifica o remetente dessa situação?** Mensagem específica? |
| **Falha ao inserir registro de transação (violação de PK, disk full)** | Débito e crédito foram concluídos, mas `INSERT` na tabela de transações falha | ? | **CRÍTICO**: Requisito assume que o banco nunca falha nessa etapa. Mas e se DB fica sem espaço? Rollback ou estado inconsistente? |
| **Timeout durante processamento (API processa mas HTTP connection cai antes de resposta)** | Cliente não recebe 200, presume que não processou | Idempotency-Key garante que retry não duplica | ✓ Tratado, mas app precisa saber disso e reenviar com mesma chave. Sem documentação clara no requisito. |
| **Falha de rede entre cliente e API** | Cliente não consegue enviar a requisição | Cliente tenta novamente com mesma Idempotency-Key | ✓ Esperado, mas timeout é "10 segundos" — como o cliente sabe se foi timeout de rede ou processamento lento? |
| **Validação cascata: destinatário ativo, mas entre validação e crédito é bloqueado** | Account status muda durante transação | Usa snapshot isolado (isolation level) ou falha? | **RISCO**: Requisito não especifica isolation level ou pessimistic lock. Qual isolation level do DB é assumido (READ_COMMITTED, SERIALIZABLE)? |
| **Fallback para sistema legado (se existe)** | Se sistema novo falha, cai para sistema antigo? | Não especificado (MVP não menciona) | N/A para MVP, mas **em escala**, há plano B? |
| **Degradação de serviço: alguns usuários conseguem enviar, outros não (ex.: shard falha)** | Particionamento de BD causa falha parcial | ? | **Não especificado**: é fail-fast (erro para todos) ou parcial (alguns usuários conseguem)? |

### 1.2 Dependências e isolamento de falhas

**Dependências mapeadas:**
- Banco de dados relacional (crítica)
- Serviço de autenticação / token validation (se centralizado)
- Serviço de notificação / auditoria de logs (se async)
- Rede (HTTP client timeout)

**Isolamento atual:**
- API é stateless (escala bem).
- Sem microserviços mencionados; assume BD monolítico.
- Sem menção a circuit breaker, retry automático com backoff, ou fallbacks.

### 1.3 Modo degradado

**Gap:** Requisito não define modo degradado. Exemplos de questões não respondidas:
- Se BD fica lento (timeout em 10s), API retorna 503? A UI exibe "sistema indisponível" ou "tente novamente"?
- Se taxa de erro sobe (ex.: 30% das requisições falham), há limite de taxa (rate limiting) para proteger BD?
- Há health check que detecta BD indisponível e ativa fallback (ex.: modo read-only, cache)?

### 1.4 Recomendações — Functional

| Recomendação | Prioridade | Impacto |
|--------------|-----------|--------|
| **Especificar isolation level da transação** (READ_COMMITTED vs. SERIALIZABLE) e como evitar race conditions entre validação e crédito | Alta | Previne inconsistência de dados em alta concorrência |
| **Definir comportamento em falha de BD**: rollback automático → 503? Retry interno com backoff? Timeout? | Alta | Determina resiliência e mensagem ao cliente |
| **Incluir circuit breaker ou health check** para detectar BD indisponível e retornar 503 cedo (sem aguardar timeout) | Média | Melhora UX (erro rápido) e protege BD de sobrecarga |
| **Documentar fallback para read-only ou cache** em caso de falha de escrita (backlog, não MVP) | Baixa | Preparar arquitetura para escala futura |

---

## 2. APPROPRIATE — Respostas Apropriadas e Códigos HTTP

### 2.1 Códigos HTTP e mapeamento de erros

O requisito define respostas para 10 cenários (seção 10.4). Análise de adequação:

| Código | Cenário | Apropriado? | Observação |
|--------|---------|-----------|-----------|
| **200 OK** | Envio bem-sucedido | ✓ | Correto; body com `transactionId`, `status: "completed"`, timestamp ISO8601. |
| **400 Bad Request** | Dados inválidos (valor 0, formato errado) | ✓ | Correto; usa código `INVALID_AMOUNT`. |
| **401 Unauthorized** | Token inválido ou expirado | ✓ | Correto; indica problema de autenticação. |
| **403 Forbidden** | Conta bloqueada / não ativa | ✓ | Correto; diferencia "autenticado mas não autorizado". |
| **404 Not Found** | Destinatário não existe/inativo | ✓ | Apropriado (destinatário não existe do ponto de vista do remetente). |
| **409 Conflict** | Envio para si mesmo | ✓ | Correto; conflito de lógica de negócio. |
| **422 Unprocessable Entity** | Saldo insuficiente | ✓ | Apropriado (validação de negócio); diferencia de 400 (formato). |
| **408/504** | Timeout | Parcial | **Gap**: qual usar — 408 (Request Timeout, cliente) ou 504 (Gateway Timeout, servidor)? Se API responde com 504, cliente assume que não processou e pode reenviar com Idempotency-Key. Correto. Mas se é **client-side timeout** (cliente sai), API não responde nada — não é um código HTTP. Requisito confunde os dois. |
| **5xx** | Erro interno | ✓ | Genérico; apropriado para erros não tratados. |
| **Missing: 429** | Rate limit excedido | ✗ | **Gap**: Requisito não menciona rate limit ou 429 (Too Many Requests). Se escala crescer, sem proteção. |
| **Missing: 503** | Serviço indisponível (BD offline, etc.) | ✗ | **Gap crítico**: Não mapeado! Se BD falha, qual código retornar? Requisito deixa em aberto. |

### 2.2 Corpos de resposta (estrutura)

**Sucesso (200):**
```json
{
  "transactionId": "...",
  "amount": 100,
  "recipientId": "...",
  "status": "completed",
  "completedAt": "ISO8601"
}
```
✓ Claro e completo.

**Erro — exemplos do requisito:**
```json
{
  "code": "INSUFFICIENT_BALANCE",
  "message": "Saldo insuficiente.",
  "currentBalance": 50  // opcional
}
```
✓ Bem estruturado, inclui mensagem legível e código máquina.

**Gaps:**
- Não há menção a **request ID** ou **correlation ID** no corpo de resposta. Em caso de erro, seria útil para suporte rastrear a requisição em logs distribuídos.
- Não há diferenciação entre erros **temporários** (retry seguro) vs. **permanentes** (não vale retentar). Exemplo: 503 deve retentar; 409 (envio para si) não. Cliente precisa saber disso (header `Retry-After`?).

### 2.3 Ações do sistema (retry automático, rollback)

- **Rollback em falha atômica:** ✓ Especificado (seção 8.1).
- **Idempotência:** ✓ Especificada (seção 8.2); permite retry seguro.
- **Retry automático na API:** ✗ Não mencionado. Deve a API retentar internamente (ex.: 1 retry em 500ms) antes de retornar 500? Requisito não define.

### 2.4 Recomendações — Appropriate

| Recomendação | Prioridade | Impacto |
|--------------|-----------|--------|
| **Mapear 503 (Serviço indisponível)** para falhas de BD, conectividade ou health check | Alta | Clarifica resposta em falha de dependência crítica |
| **Adicionar 429 (Too Many Requests)** com `Retry-After` header se/quando rate limiting for implementado | Média | Protege API em escala |
| **Incluir `Request-ID` ou `X-Correlation-ID`** no corpo de resposta de erro (5xx) para suporte rastrear requisição | Média | Melhora diagnóstico em produção |
| **Documentar quais erros são "temporários" (retry seguro)** vs. "permanentes" (não retentar automaticamente) | Alta | Guia implementação do cliente |
| **Especificar retry automático interno (ex.: 1 retry em BD com deadlock)** ou deixar tudo para cliente retentar | Média | Define boundary entre responsabilidade API vs. cliente |

---

## 3. IMPACT — Impacto em Dados, Usuário e Negócio

### 3.1 Integridade de dados

| Cenário de falha | Risco de corrupção/perda | Mitigação no requisito | Gap |
|------------------|-------------------------|----------------------|-----|
| **Falha durante débito (ex.: constraint violado)** | Zero débito, zero crédito (rollback atômico) | ✓ Atomicidade garante | Nenhum |
| **Falha durante crédito (ex.: destinatário agora inativo)** | Zero débito, zero crédito (rollback atômico) | ✓ Atomicidade garante | Nenhum |
| **Falha ao inserir transação (ex.: disk full)** | **Débito e crédito podem estar concluídos, registro não inserido** | ✗ **Requisito assume que nunca falha**; sem tratamento | **CRÍTICO**: Saldos mudam mas não há prova/auditoria. |
| **Duplicate UUID de transaction (colisão PK)** | Violação de constraint + erro 500 | ✓ DB rejeita; retry com Idempotency-Key reutiliza transactionId original | Bom, mas qual transactionId retorna em retry? Novo ou original? |
| **Race condition: dois débitos simultâneos do mesmo remetente excedendo saldo** | Possível saldo negativo se isolation inadequado | ✓ Atomicidade isolada (SERIALIZABLE) previne | ✗ **Requisito não especifica isolation level**; DB padrão (READ_COMMITTED) permite race. |
| **Falha em cascata: B credita A, mas A's webhook falha** | A crédito permanece, webhook perdido | N/A (sem webhook no MVP) | N/A |
| **Concorrência: consulta de lista de transações enquanto operação está em andamento** | Usuário vê transação inconsistente ou parcial | Snapshot isolado em leitura? | ✗ Não especificado; leitura suja pode ocorrer. |

### 3.2 Impacto para o usuário

| Cenário | Impacto negativo | Severidade |
|---------|------------------|-----------|
| **Timeout (504) após enviar 1.000 QualiPoints** | Usuário não sabe se foi enviado. Pode enviar de novo (protegido por Idempotency-Key), mas se esquecer, duplica intencionalmente. | Alta |
| **Saldo insuficiente detectado tarde (após débito parcial)** | Impossível com atomicidade; erro retorna antes de qualquer mudança. | Mitigado ✓ |
| **Mensagem de erro técnica (ex.: "ORA-00001: Unique constraint violated")** | Usuário confuso, sem próximos passos. | Alta |
| **UI trava em estado "Processando" após erro 5xx** | Usuário acha que enviou e espera feedback infinito. | Crítica |
| **Retry automático do cliente causa múltiplos débitos (sem Idempotency-Key)** | Remetente perde QualiPoints. | Crítica |
| **Destinatário bloqueado após validação bem-sucedida** | Crédito não é entregue (rollback). Mensagem apropriada informa? | Alta |

### 3.3 Impacto para o negócio

| Cenário | Impacto | Risco |
|---------|---------|------|
| **Perda de transações não registradas (disk full em DB)** | Inconsistência de saldo, auditoria impossível, reclamações de usuários. | Alto |
| **Taxa de erro alta (> 5%) em horário de pico** | Usuários frustrados, abandono, suporte sobrecarregado. | Alto |
| **Débito duplo por falha de idempotência** | Reembolso manual, perda de confiança, impacto reputacional. | Crítico |
| **Cascata: BD falha, todos os usuários afetados** | Indisponibilidade do serviço, SLA violado. | Crítico |
| **Audit trail incompleto (transações não logadas)** | Impossível rastrear origem de inconsistências, fraude invisível. | Alto |

### 3.4 Recomendações — Impact

| Recomendação | Prioridade | Impacto |
|--------------|-----------|--------|
| **Especificar: o que acontece se INSERT de transação falha após débito/crédito?** (ex.: retry com exponential backoff, rollback, fila de reprocessamento) | Crítica | Previne saldos sem auditoria |
| **Garantir SERIALIZABLE isolation ou pessimistic lock** em débito/crédito para evitar race condition | Alta | Previne saldo negativo em concorrência alta |
| **Implementar idempotência obrigatória no cliente** (gerar UUID antes de enviar, não deixar vazio) | Alta | Previne débito duplo |
| **Documentar estratégia de SLA**: máximo de requisições falhando, timeout máximo, alertas | Média | Define comportamento esperado em escala |
| **Incluir compensação de transação (ex.: reversal, reembolso automático)** em backlog para falhas detectadas offline | Média | Reduz trabalho manual de suporte |

---

## 4. LOG — Logs Estruturados e Diagnóstico

### 4.1 Informações de log esperadas

O requisito menciona "auditoria: toda tentativa de envio (sucesso ou falha)" (seção 8.3) mas **não detalha estrutura de log**. Questões não respondidas:

| Aspecto | Pergunta | Resposta no requisito |
|---------|----------|----------------------|
| **Nível de log** | Sucesso é INFO ou DEBUG? Erro é WARN ou ERROR? 4xx é o quê? | ✗ Não especificado |
| **Campos obrigatórios** | Quais IDs logar (user, transaction, recipient, request)? | Parcial: "idempotency key, usuário, valor, destinatário, resultado" |
| **Contexto distribuído** | Request ID ou Correlation ID para rastrear em microsserviços? | ✗ Não mencionado |
| **Segurança** | Logar token? Logar saldo completo? | ✗ Não especificado |
| **Estrutura** | Log é texto livre, JSON, ou formato estruturado? | ✗ Não especificado |

### 4.2 Cenários de falha e logs

| Cenário | Informação crítica a logar | Formato sugerido |
|---------|--------------------------|-----------------|
| **Sucesso (200)** | request_id, user_id, recipient_id, amount, transaction_id, timestamp, idempotency_key | JSON estruturado; nível INFO |
| **Erro de autenticação (401)** | request_id, tentativa de user_id (se disponível), timestamp | JSON; nível WARN (não é erro do sistema, é cliente não autorizado) |
| **Erro de validação (400, 422)** | request_id, user_id, reason (INVALID_AMOUNT, INSUFFICIENT_BALANCE), timestamp | JSON; nível WARN |
| **Erro de negócio (404, 409, 403)** | request_id, user_id, code (RECIPIENT_NOT_FOUND, SELF_TRANSFER_NOT_ALLOWED), timestamp | JSON; nível WARN |
| **Erro interno (500, 503)** | request_id, user_id, error_message, stack_trace (se apropriado), timestamp, idempotency_key | JSON; nível ERROR; **stack trace em dev, não em prod** |
| **Timeout (504)** | request_id, user_id, duration_ms (tempo até timeout), idempotency_key, timestamp | JSON; nível WARN ou ERROR (depende se é timeout esperado) |
| **Retry via Idempotency-Key** | request_id, idempotency_key, "reusing cached result from", original_timestamp | JSON; nível DEBUG ou INFO |
| **Falha de BD (rollback)** | request_id, user_id, error_code (ex.: deadlock, constraint violation), affected_tables, timestamp | JSON; nível ERROR; stack trace protegido |

### 4.3 Correlação distribuída

**Gap:** Se o sistema crescer para microsserviços (Auth → API → DB → Audit), como rastrear uma requisição?

- Sugerir: header `X-Correlation-ID` ou `X-Request-ID` passado entre serviços.
- Incluir em todos os logs relacionados à mesma requisição.

### 4.4 Dados sensíveis

**Gap:** Requisito diz "logs não expõem tokens ou dados sensíveis em claro" (seção Log do FAILURE.md), mas não especifica o que logar:

- ✗ Logar `Authorization: Bearer <token>`? NÃO.
- ✓ Logar "authentication attempt by user X"? SIM.
- ✗ Logar saldo completo do usuário? Depende da sensibilidade; melhor não.
- ✓ Logar "saldo insuficiente"? SIM (é fato de negócio relevante, sem expor valor exato).

### 4.5 Recomendações — Log

| Recomendação | Prioridade | Impacto |
|--------------|-----------|--------|
| **Especificar formato de log estruturado (JSON)** com campos obrigatórios: request_id, user_id, timestamp, code, message | Alta | Permite busca/alertas automatizados em log aggregation |
| **Incluir Correlation ID** em header de requisição e propagar em todos os logs | Média | Essencial para diagnóstico em arquitetura distribuída futura |
| **Definir níveis de log**: INFO (sucesso), WARN (cliente error 4xx), ERROR (servidor error 5xx) | Alta | Clarifica severidade e permite alertas |
| **Documentar quais dados NÃO logar** (token, saldo exato, credenciais) e quais SIM (user_id, amount, reason) | Alta | Previne vazamento de dados sensíveis |
| **Implementar log de idempotência**: quando Idempotency-Key é reutilizada, logar "cached result reused" | Média | Ajuda monitorar retry rate |

---

## 5. UI — Interface do Usuário e Feedback

### 5.1 Estados da transação (conforme requisito)

O requisito mapeia 3 estados na UI (seção 11.1):
- **Processando**: loading visível, botão desabilitado.
- **Concluído**: confirmação de sucesso, saldo atualizado.
- **Falhou**: mensagem de erro, permitir retry.

**Análise de suficiência:**

| Estado | Descrição no requisito | Gap ou risco |
|--------|------------------------|-------------|
| **Processando** | "Indicador de carregamento; desabilitar novo envio" | ✓ OK, mas: quanto tempo é "aceitável" sem feedback? Se > 5s, sugerir ao usuário "a operação está demorando"? |
| **Concluído** | "Exibir confirmação de sucesso; atualizar saldo e lista" | ✓ OK, mas: animação de sucesso? Som? Botão "enviar novamente"? |
| **Falhou** | "Exibir mensagem conforme código; permitir retry" | ⚠ Parcial: qual a diferença visual entre erro temporário (retentar) vs. permanente (não faz sentido retentar)? |

### 5.2 Comportamento em timeout e erro de rede

**Gap crítico:** Requisito **não especifica comportamento na UI** em timeout ou erro de rede:

| Cenário | Comportamento esperado | Especificação no requisito |
|---------|----------------------|-------------------------|
| **Cliente espera 10s, API não responde** | UI exibe "Tempo esgotado. Tente novamente?" com botão Retry | ✗ Não especificado |
| **Rede cai enquanto carregando** | UI exibe "Sem conexão. Tente novamente quando conectado" | ✗ Não especificado |
| **Usuário toca "Enviar" duas vezes rapidamente** | Segundo clique é ignorado (botão desabilitado) | ✓ "Desabilitar novo envio até resposta" |
| **Usuário fecha app durante "Processando"** | Ao rearir, app mantém estado (UUID da tentativa) para retentar com mesma Idempotency-Key? | ✗ Não especificado (depende da app, não da API) |
| **API retorna 503 (indisponível) três vezes seguidas** | UI sugere "Tente novamente em alguns instantes" ou "Sistema em manutenção"? | ✗ Não especificado |

### 5.3 Mensagens de erro — tom e clareza

Requisito mapeia mensagens (seção 11.2) com exemplos legíveis:
- "Valor deve ser entre 1 e 10.000 QualiPoints." (claro, sem jargão)
- "Saldo insuficiente. Seu saldo atual não permite este envio." (empático)

**Análise:**

| Mensagem | Ton |ade | Clareza | Gap |
|----------|------|---------|-----|
| "Valor deve ser entre 1 e 10.000 QualiPoints." | Neutro | ✓ Alta | Nenhum |
| "Destinatário não encontrado. Verifique o ID ou o status da conta." | Neutro | ✓ Alta | Bom; oferece próximos passos |
| "Não é permitido enviar QualiPoints para a própria conta." | Neutro | ✓ Alta | Nenhum |
| "Saldo insuficiente. Seu saldo atual não permite este envio." | Empático | ✓ Alta | Bom; não culpabiliza. Mas: "Seu saldo atual é X" seria melhor (informação, não presunção)? |
| "Sua conta não está ativa para envio. Entre em contato com o suporte." | Empático | ✓ Alta | Bom; oferece recurso (suporte) |
| "A operação demorou mais que o esperado. Tente novamente." | Neutro/empático | ⚠ Média | Bom, mas: "Tente novamente ou aguarde alguns segundos"? Não deixa claro se é timeout da API ou rede. |
| "Ocorreu um erro. Tente novamente em instantes." | Empático | ⚠ Média | Genérica; não informa qual erro. Melhor: "Erro temporário do sistema. Tente novamente em alguns instantes." |
| "Sessão expirada. Faça login novamente." | Neutro | ✓ Alta | Claro; oferece próximo passo |

### 5.4 Estados inconsistentes

**Risco:** Se a API retorna erro após processing visual começou, a UI pode ficar em estado inconsistente:

- Botão ainda está desabilitado (parece congelado)?
- Saldo desatualizado (mostra valor antigo quando deveria recarregar)?
- Indicador de loading nunca some?

### 5.5 Recomendações — UI

| Recomendação | Prioridade | Impacto |
|--------------|-----------|--------|
| **Especificar timeout visual:** após 5s de carregamento, exibir "A operação está demorando. Aguarde ou tente novamente."** | Alta | Melhora UX em conexão lenta |
| **Diferenciar visualmente erros temporários vs. permanentes:** cor, ícone, botão "Retentar" visível vs. oculto | Alta | Guia usuário sobre próximos passos |
| **Especificar persistência de estado:** se usuário fecha app durante "Processando", ao rearir, mostrar "Verificando status da transferência..." | Média | Evita confusão e retenção de UUID para retry |
| **Adicionar "Seu saldo atual: X" em mensagens de erro de saldo** | Média | Melhora confiança e informação do usuário |
| **Definir comportamento em múltiplos cliques:** botão desabilitado vs. loading com overlay? | Baixa | Melhora UX mas é implementação de app |
| **Especificar animação/som de sucesso** (opcional) para feedback positivo claro | Baixa | Melhora UX mas não é crítico |

---

## 6. RECOVERY — Recuperação Automática e Manual

### 6.1 Recuperação automática (server-side)

O requisito **não especifica retry automático na API**. Questões:

| Cenário | Deve retentar automaticamente? | Impacto |
|---------|-----|--------|
| **Deadlock em BD durante transação** | SIM (temporário, determinístico) | Se API retorna 500, cliente não sabe se pode retentar |
| **Timeout de conexão BD (breve, reconecta)** | SIM | Melhor experiência; menos erros para cliente |
| **Erro de Network I/O (packet loss, reconecta)** | SIM | Retentar 1x internamente melhora taxa de sucesso |
| **Rate limit interno atingido** | SIM, com backoff exponencial | Evita bombardear BD |
| **Falha de write-ahead log (WAL)** | ? (raro, mas crítico) | Depende do DB; assumir que DB lida |

### 6.2 Recuperação manual (client-side)

Requisito menciona: "cliente pode repetir a requisição usando a mesma chave de idempotência" (seção 5.2).

**Análise:**

| Aspecto | Especificado? | Gap |
|--------|--------|-----|
| **Cliente gera Idempotency-Key** | Parcial: "UUID gerado no App" | ✓ OK; mas como garantir que app implementa isso? Documentar em contrato de API |
| **Cliente reenviacom mesma chave** | ✓ Sim (seção 5.2) | ✓ OK |
| **API retorna mesmo resultado** | ✓ Sim (seção 8.2) | ✓ OK; evita débito duplo |
| **Sem estado "pendente" persistido** | ✓ Mencionado (seção 12) | ✓ OK; fluxo é síncrono |
| **Botão "Retentar" na UI** | Implícito ("permitir retry") | ⚠ Como o app sabe qual erro permite retry? Precisa documentar |
| **Preservação de estado (formulário)** | Não mencionado | ✗ Se usuário vê erro 422 (saldo insuficiente), dados do formulário desaparecem ou persistem? |

### 6.3 Recuperação em cenários específicos

| Cenário | Recuperação esperada | Especificação |
|---------|--------|--------|
| **Timeout (504)** | Retentar com mesma Idempotency-Key | ✓ Documentado |
| **Saldo insuficiente (422)** | Usuário aumenta saldo (fora da API) e tenta novamente | ✗ Como? Novo UUID ou mesma Idempotency-Key? (provavelmente novo UUID) |
| **Destinatário não existe (404)** | Verificar ID e tentar com outro | ✗ Não há "sugestão" de contatos ou IDs válidos |
| **Envio para si mesmo (409)** | Selecionar outro destinatário | ✗ UI não valida isso localmente; depende de erro da API |
| **Conta bloqueada (403)** | Contato com suporte | ✓ Documentado ("Entre em contato com o suporte") |
| **Erro genérico 500** | Retentar? Ou aguardar? | ⚠ Requisito sugere "Tente novamente em instantes"; sem clara se retry automático é aconselhado |

### 6.4 Possibilidade de deadlock em retry

**Risco:** Se cliente faz retry manual 5 vezes seguidas (toque repetido), e API não consegue processar, qual o limite?

| Aspecto | Especificado? |
|--------|--------|
| **Máximo de retries do cliente (sem back-off)** | ✗ Não |
| **Máximo de retries da API (interno, com back-off)** | ✗ Não |
| **Rate limiting para proteger de retry storm** | ✗ Não (vide 429 no gap de Appropriate) |

### 6.5 Recomendações — Recovery

| Recomendação | Prioridade | Impacto |
|--------------|-----------|--------|
| **Implementar 1-3 retries automáticos na API** para deadlock, timeout breve, rate limit com backoff exponencial | Alta | Reduz taxa de erro sem duplicar débito |
| **Documentar qual erro permite retry:** 503, 504 → retentar; 422, 409 → não retentar automaticamente | Alta | Guia implementação do cliente |
| **Especificar comportamento em saldo insuficiente:** novo UUID ou mesma Idempotency-Key? | Média | Previne confusão na implementação do app |
| **Adicionar header `Retry-After` na resposta de erro** (ex.: 503, 429) para indicar quando retentar | Média | Melhora eficiência de retry exponencial |
| **Limitar retry automático do cliente a 3 tentativas** com back-off (1s, 2s, 4s) | Média | Evita retry storm sem impacto no usuário |
| **Preservar estado do formulário em erro 422** (saldo insuficiente) para novo envio após recarregar saldo | Baixa | Melhora UX |

---

## 7. EMOTIONS — Mensagens Empáticas e Transparência

### 7.1 Tom das mensagens mapeadas

Requisito mapeia mensagens de erro (seção 11.2). Análise de tom:

| Mensagem | Tom | Culpabiliza? | Transparência | Próximos passos |
|----------|-----|--------|--------|------|
| "Valor deve ser entre 1 e 10.000 QualiPoints." | Neutro | Não | ✓ Claro o que fazer | Reformular input |
| "Saldo insuficiente. Seu saldo atual não permite este envio." | Empático | Não | ✓ Claro por que falhou | Aumentar saldo; retentar depois |
| "Destinatário não encontrado. Verifique o ID ou o status da conta." | Neutro/empático | Não | ✓ Oferece causas possíveis | Verificar ID; contato |
| "Não é permitido enviar QualiPoints para a própria conta." | Neutro | Não | ✓ Claro por que | Selecionar outro destinatário |
| "Sua conta não está ativa para envio. Entre em contato com o suporte." | Empático | Não | ⚠ Por que conta não está ativa? (não explicado) | Contato com suporte |
| "A operação demorou mais que o esperado. Tente novamente." | Empático | Não | ⚠ Não explica se é temporário ou persistente | Retentar (implícito) |
| "Ocorreu um erro. Tente novamente em instantes." | Empático | Não | ✗ Genérica; não explica qual erro | Retentar |
| "Sessão expirada. Faça login novamente." | Neutro | Não | ✓ Claro por que e o que fazer | Login |

### 7.2 Transparência em falhas de dependência

**Gap:** Requisito não mapeia mensagens para falhas de dependência:

| Falha | Mensagem esperada | Atual |
|-------|----------|--------|
| **BD offline (503)** | "Nosso sistema está temporariamente indisponível. Tente novamente em alguns instantes." | ✗ Não mapeada |
| **Serviço de autenticação indisponível (503)** | "Não conseguimos verificar sua autenticação neste momento. Tente novamente." | ✗ Não mapeada |
| **Timeout de rede lenta (504)** | "A conexão está lenta. Pode levar mais tempo. Aguarde ou tente novamente." | ✗ Só menciona "demorou" |

### 7.3 Empoderamento do usuário

**Questões:**

| Aspecto | Pergunta | Resposta no requisito |
|--------|----------|--------|
| **Controle** | Usuário se sente no controle ou à mercê? | Parcialmente: tem botão "Retentar", mas sem clara se vale a pena. |
| **Informação** | Usuário sabe o que aconteceu e por que? | Parcialmente: mensagens explicam, mas algumas são genéricas. |
| **Esperança** | Há esperança de sucesso em retentar ou é perda? | Parcialmente: "Tente novamente" sugere retry; 409 (envio para si) não sugere esperança. |
| **Responsabilidade** | Culpa cai no usuário ou sistema? | ✓ Bem feito: "Valor deve ser..." (culpa no usuário, apropriado), "Saldo insuficiente" (não culpa), "Sistema indisponível" (culpa no sistema, não mapeado). |

### 7.4 Contexto emocional em cenários críticos

| Cenário | Emoção do usuário | Mensagem apropriada | Atual |
|---------|-------|-------|-----|
| **Timeout após enviar 1.000 QualiPoints** | Pânico: "Perdi os QualiPoints?" | "A operação pode ter sido bem-sucedida, mas não conseguimos confirmar. Tente novamente com o botão abaixo (seguro usar a mesma chave)." | ✗ Não tão detalhado |
| **Erro genérico 500** | Frustração: "O que houve?" | "Encontramos um erro temporário. Nosso time foi notificado. Tente novamente em alguns segundos." (indica que foi reportado) | ✗ Genérico demais |
| **Conta bloqueada (403)** | Confusão: "Por quê?" | "Sua conta foi temporariamente bloqueada. Contacte o suporte em [link/telefone]." (oferece recurso) | ✓ OK: "Entre em contato com o suporte" |
| **Idempotência em retry** | Confiança: "É seguro retentar?" | "Você pode retentar com segurança. Não será debitado novamente." (educação sobre idempotência) | ✗ Não documentado para usuário |

### 7.5 Recomendações — Emotions

| Recomendação | Prioridade | Impacto |
|--------------|-----------|--------|
| **Mapear mensagens para falhas de dependência (503, serviço offline)** com tom "problema temporário, tente novamente" | Alta | Reduz pânico do usuário em indisponibilidade |
| **Documentar para o app:** em timeout (504), exibir "Pode ter sido bem-sucedido. Tente novamente (seguro)."** | Alta | Reduz confusão e medo de perda de dados |
| **Adicionar "Sua conta está ativa?" ou explicação** quando 403 (conta não ativa) | Média | Melhora compreensão do problema |
| **Incluir "Ocorreu um erro raro que foi reportado. Tente novamente."** em 500, em vez de genérico | Média | Transmite transparência e confiança |
| **Documentar para o app:** em retry, incluir mensagem "Seguro retentar; você não será debitado novamente." (educação sobre idempotência)** | Média | Reduz ansiedade do usuário |
| **Definir tone of voice do projeto** (ex.: "casual e empático" vs. "formal e técnico") para consistência em todas as mensagens | Baixa | Melhora coerência da experiência |

---

## 8. Resumo de Gaps e Riscos Críticos

### 8.1 Gaps críticos (Deve resolver antes de codificar)

| Gap | Dimensão | Impacto | Recomendação |
|-----|----------|--------|-------------|
| **BD falha durante INSERT de transação (após débito/crédito)** | Functional, Impact | Dados inconsistentes (saldo muda, mas transação não registrada) | Definir estratégia: retry com fila, rollback completo, ou aceitação de dados órfãos (com reconciliação) |
| **Isolation level não especificado** | Functional, Impact | Race condition em concorrência alta (saldo negativo possível) | Especificar READ_COMMITTED com pessimistic lock OU SERIALIZABLE |
| **Código HTTP 503 não mapeado para falha de BD** | Appropriate | Cliente não sabe qual erro retornar em DB offline | Adicionar: 503 (Serviço indisponível) para falhas de dependência crítica |
| **Retry automático na API não definido** | Recovery, Appropriate | Taxa de erro alta em BD com deadlock temporário | Definir: API faz 1-3 retries internos com backoff para deadlock, timeout; cliente pode retentar manualmente com Idempotency-Key |
| **Mensagens para falha de BD / serviço offline não mapeadas** | UI, Emotions | Usuário vê erro genérico, acha que é culpa dele | Mapear: "Sistema indisponível. Tente novamente em alguns instantes." |

### 8.2 Gaps médios (Refinar antes de PR)

| Gap | Dimensão | Impacto | Recomendação |
|-----|----------|--------|-------------|
| **Log estruturado não especificado** | Log | Diagnóstico difícil em produção | Especificar: JSON com fields obrigatórios (request_id, user_id, timestamp, code, message) |
| **Rate limiting (429) não mapeado** | Appropriate | API vulnerável a retry storm | Adicionar: 429 (Too Many Requests) com `Retry-After` header (backlog se escala) |
| **Timeout visual na UI** | UI | Usuário acha que congelou após 10s | Especificar: após 5s de "Processando", exibir "A operação está demorando. Aguarde ou tente novamente." |
| **Comportamento em saldo insuficiente (novo UUID ou mesma Idempotency-Key?)** | Recovery, Log | Confusão na implementação do app | Especificar: novo UUID (é intenção diferente de envio) |
| **Correlação distribuída (Correlation ID)** | Log | Impossível rastrear requisição em microsserviços futuros | Documentar: header `X-Correlation-ID` passado entre serviços, incluído em logs |

### 8.3 Gaps baixos (Nice-to-have, backlog)

| Gap | Dimensão | Impacto | Recomendação |
|-----|----------|--------|-------------|
| **Saldo no corpo de erro (422)** | UI, Emotions | Usuário não vê saldo atual | Adicionar: `"currentBalance": 50` no corpo de erro 422 |
| **Animação/som de sucesso** | UI | Feedback positivo menos evidente | Especificar: animação de "✓" + sound (opcional) |
| **Sugestão de contatos (404)** | UI | Usuário não sabe qual ID enviar | Backlog: listar 5 contatos recentes quando recipient_id é inválido |
| **Compensação automática (reversal)** | Recovery, Impact | Reembolso manual de erros detectados offline | Backlog: sistema de reversão automática de transações órfãs |

---

## 9. Matriz de Decisão para Refinamento

Para cada gap, a equipe de produto deve decidir:

| Gap | Prioridade | Decide em | Impacto em Desenvolvimento |
|-----|-----------|-----------|---------------------------|
| **Isolation level** | Crítica | Design review (antes de código) | Muda lock strategy e indice no BD |
| **Retry automático (API)** | Crítica | Design review | Muda tratamento de exceção; implementa retry com backoff |
| **503 em BD offline** | Crítica | Design review | Adiciona health check; mapeia erro no handler |
| **Log estruturado** | Alta | Sprint de setup | Escolher: SLF4J, Logback, JSON; implementar decoradores |
| **Timeout visual (5s)** | Alta | Design/PM | Decisão de UX; implementação no app |
| **Msgs de BD offline** | Alta | PM | Copywriting; validação de tom |
| **Rate limiting (429)** | Média | Backlog / próximo sprint | Implementação de middleware/servlet filter |
| **Correlation ID** | Média | Backlog (quando microsserviços) | Adiciona header em cliente HTTP; middleware em API |

---

## 10. Checklist de Refinamento (baseado em FAILURE)

Antes de mover para desenvolvimento, valide:

### 10.1 Functional
- [ ] Isolation level do BD especificado (READ_COMMITTED com lock OU SERIALIZABLE)
- [ ] Comportamento em BD offline definido (503, retry automático, fallback?)
- [ ] Rollback automático em falha de qualquer etapa (débito, crédito, insert transação)
- [ ] Sem cascata não intencional entre serviços (se houver)

### 10.2 Appropriate
- [ ] Código HTTP 503 adicionado para BD/serviço offline
- [ ] Código HTTP 429 adicionado (backlog se não MVP)
- [ ] Request ID / Correlation ID incluído em corpo de resposta de erro (5xx)
- [ ] Retry automático interno (API) vs. manual (cliente) claramente definido

### 10.3 Impact
- [ ] Estratégia para INSERT de transação falhar após débito/crédito definida
- [ ] Idempotência obrigatória documentada (cliente DEVE enviar UUID, não é opcional)
- [ ] SLA documentado (máximo de requisições falhando, timeout máximo)
- [ ] Plano de compensação / reversal documentado (MVP ou backlog)

### 10.4 Log
- [ ] Formato de log estruturado (JSON) com campos obrigatórios
- [ ] Nível de log (INFO, WARN, ERROR) mapeado por tipo de erro
- [ ] Dados sensíveis definidos (o que NÃO logar: token, saldo exato)
- [ ] Correlação distribuída (Correlation ID) documentada

### 10.5 UI
- [ ] Estados de erro diferenciam temporário (retry) vs. permanente (não retry)
- [ ] Mensagem de timeout visual adicionada (após 5s: "Aguarde ou tente novamente")
- [ ] Preservação de estado em erro (formulário não limpo em 422)
- [ ] Botão "Retentar" claramente visível/identificável

### 10.6 Recovery
- [ ] Retry automático (API): condições e backoff definidos
- [ ] Retry manual (cliente): documentação clara de qual erro permite retry
- [ ] Header `Retry-After` incluído em 503, 429
- [ ] Comportamento em saldo insuficiente (novo UUID vs. mesma Idempotency-Key) claro

### 10.7 Emotions
- [ ] Mensagens de BD offline / serviço indisponível (503) mapeadas
- [ ] Mensagem de timeout (504) mais detalhada ("Pode ter funcionado; tente novamente")
- [ ] Tom consistente em todas as mensagens (empático, não culpabilizador)
- [ ] Próximos passos claros em cada erro (ação ou contato)

---

## 11. Casos de Teste Sugeridos (baseados em FAILURE)

### 11.1 Testes unitários (Functional, Appropriate)

```gherkin
Scenario: BD retorna deadlock durante débito
  Given API recebeu requisição válida
  When BD lança SQLException (deadlock)
  Then API retenta 1x internamente
  And sucesso na segunda tentativa
  And retorna 200 com transactionId

Scenario: BD indisponível (connection timeout)
  Given API recebeu requisição válida
  When BD não responde em 2s (circuit open)
  Then API retorna 503
  And corpo contém "UNAVAILABLE" ou similar

Scenario: INSERT de transação falha após débito/crédito
  Given débito e crédito bem-sucedidos
  When INSERT de transação lança constraint violation
  Then rollback completo (débito e crédito revertidos)
  And retorna 500 com correlation ID
```

### 11.2 Testes integração (Impact, Log, Recovery)

```gherkin
Scenario: Race condition em concorrência alta
  Given dois envios simultâneos do mesmo user (saldo = 100)
  And ambos tentam enviar 60 QualiPoints cada
  When duas requisições chegam em paralelo
  Then apenas uma sucede (saldo final = 40 ou 100)
  And não há saldo negativo ou duplicado

Scenario: Idempotência em retry
  Given primeira requisição com Idempotency-Key X
  When retorna 200 com transactionId A
  And segunda requisição com mesma chave X
  Then retorna 200 com transactionId A (cache)
  And DB não tem dois registros

Scenario: Log estruturado em erro 500
  Given erro interno durante processamento
  When API retorna 500
  Then log contém JSON com:
    - request_id: UUID
    - user_id: ID do usuário
    - error_code: código específico
    - timestamp: ISO8601
    - message: descrição legível (sem stack trace em prod)
```

### 11.3 Testes E2E (UI, Recovery, Emotions)

```gherkin
Scenario: Timeout após 10s — UI não trava
  Given cliente envia requisição
  When API não responde em 10s
  Then UI exibe "Tempo esgotado. Tente novamente?"
  And botão "Retentar" está habilitado
  And loading indicator desaparece

Scenario: Recuperação em saldo insuficiente
  Given usuário tenta enviar 100, saldo = 50
  When API retorna 422 INSUFFICIENT_BALANCE
  Then UI exibe "Saldo insuficiente"
  And campos de envio permanecem preenchidos
  And usuário pode aumentar saldo e retentar

Scenario: Mensagem de BD offline é empática
  Given BD está offline (503)
  When usuário tenta enviar
  Then UI exibe "Sistema temporariamente indisponível. Tente novamente em alguns instantes."
  And tom é reassegurador, não técnico
  And botão "Retentar" é visível
```

---

## 12. Próximos Passos

### 12.1 Ordem recomendada

1. **Design Review (1-2h):** Discutir gaps críticos com PM, tech lead, DBA
   - Isolation level do BD
   - Retry automático (API)
   - Comportamento em BD offline (503)
   - Estratégia para INSERT falhando

2. **Refinamento de Requisito (1-2h):** Atualizar REQ_INICIAL_V2 com decisões
   - Adicionar seções: "Isolation e Concorrência", "Retry Automático", "Comportamento em BD Offline"
   - Mapear códigos HTTP 503, 429
   - Atualizar mensagens de erro (BD offline, timeout detalhado)

3. **Design de Teste (2-3h):** Criar TEST_STRATEGY baseado neste documento
   - Mapear quais cenários FAILURE testar em cada camada (unit, integration, service, E2E)
   - Priorizar: race condition, timeout, idempotência, logs

4. **Implementação (3-4 sprints):**
   - Camada 1: BD, atomicidade, isolation
   - Camada 2: API com retry automático e 503
   - Camada 3: Logs estruturados
   - Camada 4: App com UI resiliente e mensagens mapeadas

5. **Testes (paralelo com implementação):** Usar TEST_STRATEGY para guiar TDD/BDD

---

## 13. Conclusão

O requisito REQ_INICIAL_V2 é **sólido em arquitetura geral** (atomicidade, idempotência, autenticação), mas **deixa abertos cenários críticos de falha** que podem resultar em inconsistência de dados, UX confusa e taxa de erro alta em produção.

Aplicar a heurística **FAILURE** revelou **70+ cenários** e permitiu priorizar **11 gaps críticos/médios** que devem ser refinados antes de codificação.

**Benefício:** Equipe entra no desenvolvimento com clareza sobre como o sistema falha, o que logar, como recuperar e como comunicar com o usuário — resultando em **sistema mais robusto, diagnóstico mais rápido e experiência mais confiável**.

---

## 14. Referências

- **FAILURE Heuristic:** Ben Simo, exploratory testing of failure scenarios
- **Requisito base:** REQ_INICIAL_V2.md — Envio de QualiPoints entre usuários
- **Conceitos aplicados:**
  - Atomicidade em transações (ACID)
  - Isolation levels (READ_COMMITTED, SERIALIZABLE)
  - Idempotência e Idempotency-Key (RFC 9110)
  - Retry automático com backoff exponencial
  - Circuit breaker pattern (para fallback em BD offline)
  - Logs estruturados (JSON, correlation ID)
  - Resiliência em microsserviços (timeout, circuit breaker, health check)
