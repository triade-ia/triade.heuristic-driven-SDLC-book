# Relatório RCRCRC — Envio de QualiPoints entre Usuários

**Requisito analisado:** REQ QualiPoints v2 — Envio de QualiPoints entre usuários
**Modo de entrada:** Épico + Tarefa (requisito funcional detalhado com contrato de API)
**Data da análise:** 2026-03-29
**Heurística aplicada:** RCRCRC (James Bach)

---

## R — Recent (Recente)

**Foco:** O que está sendo introduzido ou alterado neste ciclo e pode gerar regressão?

O requisito descreve uma **funcionalidade nova** (envio de QualiPoints), o que implica que todo o código é recente. Áreas de atenção:

| Área | Risco de Regressão | Justificativa |
|------|---------------------|---------------|
| **Endpoint POST `/api/v1/transfers`** | **Alto** | Endpoint novo; toda a lógica de débito/crédito/registro é inédita. Qualquer bug aqui impacta diretamente o saldo dos usuários. |
| **Tabela de saldos (carteira)** | **Alto** | Operações de escrita em saldo já existente. Se a tabela de saldos já era usada por outras funcionalidades (consulta, extrato), alterações no schema ou locks podem causar regressão em leituras existentes. |
| **Tabela de transações** | **Médio** | Nova tabela ou novos registros; a lista de transações recentes (últimas 10) depende diretamente desta inserção. |
| **Mecanismo de idempotência** | **Alto** | Introdução de controle por `Idempotency-Key`. Se implementado via tabela ou cache, é componente novo que pode interferir com outros fluxos de API caso compartilhe infraestrutura. |
| **Módulos adjacentes: autenticação e autorização** | **Médio** | O fluxo depende do token Bearer para identificar o remetente. Alterações no middleware de autenticação ou na extração de identidade do token podem afetar outros endpoints existentes. |
| **Lista de transações recentes** | **Médio** | Novo endpoint ou nova query; se compartilhar a mesma API de extrato do painel web, pode afetar a performance ou o contrato de resposta existente. |

**Módulos adjacentes a monitorar:**
- Serviço/módulo de autenticação (token parsing)
- Camada de persistência de saldos (se já existente)
- API do painel web (se consome a mesma tabela de transações)

---

## C — Core (Principal)

**Foco:** Funcionalidades essenciais do produto que, se quebrarem, paralisam o negócio ou afetam a maioria dos usuários.

| Funcionalidade Core | Criticidade | Justificativa |
|---------------------|-------------|---------------|
| **Débito e crédito de saldo** | **Crítica** | Operação financeira direta. Saldo incorreto = perda de confiança do usuário e possível impacto financeiro real. |
| **Atomicidade da transação (débito + crédito + registro)** | **Crítica** | Se a atomicidade falhar, é possível debitar sem creditar (ou vice-versa), gerando inconsistência irrecuperável sem intervenção manual. |
| **Fluxo de autenticação** | **Crítica** | Sem autenticação funcional, nenhum envio é possível. Regressão aqui bloqueia 100% dos usuários. |
| **Validação de saldo suficiente** | **Crítica** | Se falhar, usuários podem enviar pontos que não possuem, gerando saldo negativo. |
| **Consulta de saldo e extrato** | **Alta** | Funcionalidade já existente (painel web). Se o novo fluxo de envio afetar a performance ou a consistência dos dados consultados, impacta todos os usuários do painel. |

**Fluxos Core que devem ser validados após qualquer deploy deste requisito:**
1. Login e obtenção de token
2. Consulta de saldo (antes e depois do envio)
3. Envio completo (débito + crédito + registro)
4. Lista de transações recentes (do remetente e do destinatário)
5. Consulta no painel web (consistência com o App)

---

## R — Risky (Arriscado)

**Foco:** Áreas com histórico de falhas, alta complexidade de integração ou consequências graves em caso de falha.

| Risco | Severidade | Dimensão de Origem |
|-------|------------|---------------------|
| **Inconsistência de saldo por falha de atomicidade** | **Crítica** | A operação envolve 3 escritas (débito, crédito, registro) que devem ser atômicas. Falha parcial = saldo inconsistente. Testar cenários de falha no meio da transação (ex: banco cai após débito, antes do crédito). |
| **Débito duplo por falha na idempotência** | **Crítica** | Se o mecanismo de `Idempotency-Key` falhar (ex: race condition na verificação), o mesmo envio pode ser processado duas vezes, debitando o remetente em dobro. |
| **Race condition em envios simultâneos** | **Alta** | Dois envios simultâneos do mesmo remetente podem ambos passar na validação de saldo e debitar além do disponível (problema clássico de concorrência). Requer lock otimista ou pessimista no saldo. |
| **Timeout com estado indefinido** | **Alta** | Se a API retorna 504 mas a transação foi efetivada no banco, o cliente faz retry com mesma Idempotency-Key. Se a key não foi gravada (transação commitou mas idempotência não), há risco de débito duplo. |
| **Operações irreversíveis** | **Alta** | Não há menção a mecanismo de estorno no MVP. Um envio errado (destinatário correto porém indesejado) não pode ser revertido pelo usuário. |
| **Saldo negativo por validação insuficiente** | **Alta** | Se a validação de saldo não for feita dentro da mesma transação de banco (ex: SELECT fora do BEGIN...COMMIT), pode haver janela para saldo negativo. |

**Áreas de alto risco inerente:**
- Toda operação que altera saldo (financeiro)
- Mecanismo de idempotência (evitar débito duplo)
- Concorrência de requisições simultâneas do mesmo usuário
- Cenários de timeout/falha parcial

---

## C — Configuration-sensitive (Sensível à Configuração)

**Foco:** Funcionalidades cujo comportamento varia conforme configuração de ambiente, dados, parâmetros ou perfis.

| Variável de Configuração | Risco | Cenários de Teste |
|---------------------------|-------|-------------------|
| **Timeout da API (10s)** | **Médio** | Valor configurável por ambiente? Se staging tem timeout diferente de produção, testes de timeout não são representativos. Validar paridade. |
| **Limite máximo por transação (10.000)** | **Médio** | Valor hardcoded ou configurável? Se configurável, testar com valores diferentes (1, 10.000, 10.001). Se hardcoded, validar que o valor está correto em produção. |
| **Status de conta (ativo/bloqueado/pendente)** | **Alto** | Diferentes status de conta do remetente e destinatário geram comportamentos distintos. Testar todas as combinações: ativo→ativo, ativo→bloqueado, bloqueado→ativo, pendente→ativo. |
| **Variáveis de ambiente (HTTPS, conexão com banco)** | **Médio** | Diferenças entre staging e produção em strings de conexão, certificados HTTPS, configuração de pool de conexões podem causar falhas silenciosas. |
| **Idempotency-Key storage** | **Médio** | Se armazenada em cache (Redis) vs banco: TTL do cache, política de evicção e disponibilidade do cache variam por ambiente. |
| **Multi-tenant (futuro)** | **Baixo (MVP)** | No MVP não há multi-tenant, mas se a infraestrutura for compartilhada, validar isolamento de dados. |

**Cenários de configuração prioritários para teste:**
1. Conta remetente: ativo, bloqueado, pendente, em análise
2. Conta destinatário: ativo, bloqueado, inexistente
3. Valores limítrofes: 0, 1, 9.999, 10.000, 10.001, -1, decimal (2.5)
4. Paridade de configuração staging vs produção (timeout, limites, HTTPS)

---

## C — Conformance (Conformidade)

**Foco:** Requisitos regulatórios, contratos de API, segurança e políticas internas.

| Requisito de Conformidade | Status | Ação de Teste |
|---------------------------|--------|---------------|
| **LGPD/GDPR — dados pessoais** | **Atenção** | O registro de transação contém IDs de usuários (remetente e destinatário). Verificar se o log de auditoria não expõe dados pessoais além do necessário. Avaliar se há necessidade de anonimização em logs. |
| **Contrato de API (seção 10)** | **Obrigatório** | Validar que todos os códigos HTTP (200, 400, 401, 403, 404, 409, 422, 408/504, 5xx) retornam exatamente o schema documentado. Testar cada código de erro com o corpo esperado. |
| **Segurança — HTTPS** | **Obrigatório** | Validar que a API rejeita requisições HTTP (sem TLS). Não deve ser possível enviar QualiPoints por canal não criptografado. |
| **Segurança — Autenticação** | **Obrigatório** | Testar: sem token, token expirado, token inválido, token de outro usuário. Todos devem retornar 401. |
| **Segurança — Autorização** | **Obrigatório** | Validar que um usuário não pode debitar a conta de outro (remetente vem do token, não do body). Testar tentativa de manipulação do body para alterar remetente. |
| **Auditoria** | **Obrigatório** | Toda tentativa de envio (sucesso e falha) deve gerar registro de auditoria com: idempotency key, usuário, valor, destinatário, resultado. Verificar completude e integridade dos logs. |
| **Controle de acesso na lista de transações** | **Obrigatório** | Validar que usuário A não consegue ver transações do usuário B. Testar IDOR (Insecure Direct Object Reference). |

**Riscos de conformidade:**
- Exposição de saldo de outros usuários via manipulação de parâmetros
- Falta de rate limiting no MVP pode permitir abuso (brute force de IDs de destinatário)
- Ausência de log de auditoria em cenários de falha

---

## C — Complex (Complexo)

**Foco:** Áreas estruturalmente complexas, difíceis de testar exaustivamente e propensas a falhas ocultas.

| Área de Complexidade | Nível | Justificativa |
|----------------------|-------|---------------|
| **Transação atômica (3 operações em 1)** | **Alto** | Débito + crédito + registro em uma única transação de banco. A complexidade está nos cenários de falha parcial, deadlocks e rollback. |
| **Mecanismo de idempotência** | **Alto** | Verificar se a key já existe, decidir se reprocessa ou retorna resultado anterior, tudo dentro da mesma janela de concorrência. Race condition entre verificação e inserção da key. |
| **Concorrência de saldo** | **Alto** | Dois envios simultâneos do mesmo usuário competem pelo mesmo recurso (saldo). Requer estratégia de locking que não cause deadlock nem permita saldo negativo. |
| **Cenários de timeout + idempotência** | **Alto** | Combinação de timeout do cliente + transação que pode ou não ter commitado + retry com mesma key = múltiplos estados possíveis difíceis de testar exaustivamente. |
| **Consistência App ↔ Painel Web** | **Médio** | Ambos consomem a mesma fonte de dados, mas em momentos diferentes. Sem WebSocket no MVP, o painel pode mostrar dados desatualizados. Testar sequência: envio no App → refresh no painel → dados consistentes. |
| **Validação de regras de negócio compostas** | **Médio** | Múltiplas validações em sequência (autenticado? conta ativa? destinatário válido? diferente do remetente? saldo suficiente? valor no range?) — a ordem e a completude importam. |

**Cenários complexos prioritários para teste exploratório:**
1. Envio simultâneo do mesmo remetente para dois destinatários diferentes, com saldo suficiente para apenas um
2. Timeout na API com transação commitada + retry com mesma Idempotency-Key
3. Dois dispositivos do mesmo usuário enviando com Idempotency-Keys diferentes ao mesmo tempo
4. Envio que esgota 100% do saldo seguido de consulta imediata no painel web

---

## Mapa de Prioridades

| Dimensão | Prioridade | Áreas Críticas |
|----------|------------|----------------|
| **R — Recent** | Alta | Endpoint de transferência, mecanismo de idempotência, tabela de saldos |
| **C — Core** | Crítica | Atomicidade débito/crédito, autenticação, validação de saldo |
| **R — Risky** | Crítica | Concorrência de saldo, débito duplo, timeout com estado indefinido |
| **C — Configuration** | Média-Alta | Combinações de status de conta, valores limítrofes, paridade staging/prod |
| **C — Conformance** | Alta | Contrato de API, HTTPS, auditoria, controle de acesso (IDOR) |
| **C — Complex** | Alta | Transação atômica sob falha, idempotência + concorrência, timeout + retry |

---

## Top Riscos de Regressão

| # | Risco | Severidade | Dimensão | Probabilidade |
|---|-------|------------|----------|---------------|
| 1 | **Débito sem crédito (ou vice-versa) por falha de atomicidade** | Crítica | Risky + Complex | Média — depende da implementação de transação de banco |
| 2 | **Débito duplo por falha na idempotência sob concorrência** | Crítica | Risky + Complex | Média-Alta — race condition clássica se não houver lock na Idempotency-Key |
| 3 | **Saldo negativo por race condition em envios simultâneos** | Crítica | Risky + Complex | Alta — sem lock explícito no saldo, dois envios concorrentes podem ambos passar na validação |
| 4 | **Exposição de dados de outros usuários na lista de transações (IDOR)** | Alta | Conformance | Média — depende da implementação do filtro por usuário |
| 5 | **Contrato de API divergente da documentação (códigos/schema)** | Alta | Conformance + Recent | Média — risco natural em API nova; cada código de erro deve ser validado |
| 6 | **Regressão no fluxo de autenticação** | Alta | Core | Baixa-Média — depende se o middleware de auth foi alterado |
| 7 | **Inconsistência App ↔ Painel Web após envio** | Média | Complex + Configuration | Média — sem real-time, timing de refresh pode mostrar dados desatualizados |

---

## Perguntas em Aberto

1. **Idempotência — TTL:** Por quanto tempo a `Idempotency-Key` é mantida? Se expirar, um retry tardio pode reprocessar a transação. Qual o TTL definido?
2. **Locking de saldo:** Qual estratégia será usada para evitar race condition no saldo? Lock pessimista (SELECT FOR UPDATE), lock otimista (versão/timestamp) ou outra?
3. **Armazenamento da Idempotency-Key:** Banco relacional ou cache (Redis)? Se cache, qual a política de evicção e o que acontece se o cache reiniciar?
4. **Rate limiting:** Sem limite por período no MVP — há algum mecanismo anti-abuso (ex: rate limit por IP/token) para evitar brute force ou flood?
5. **Estorno:** Se um envio errado for feito, como será tratado no MVP? Apenas via suporte? Há endpoint de estorno planejado?
6. **Auditoria — formato e destino:** Os logs de auditoria serão gravados no banco, em arquivo, ou em serviço externo (ex: CloudWatch, Datadog)? Qual o schema?
7. **Painel web — endpoint:** A lista de transações recentes no painel web usará o mesmo endpoint do App ou haverá endpoint separado?
8. **Notificação ao destinatário:** No MVP, o destinatário saberá que recebeu pontos apenas ao consultar o saldo/extrato? Não há push notification?

---

## Lacunas de Cobertura

| Lacuna | Impacto | Recomendação |
|--------|---------|--------------|
| **Testes de concorrência** | Alto | Não é possível validar race conditions apenas com testes unitários. Necessário teste de carga com múltiplas requisições simultâneas do mesmo usuário. |
| **Testes de falha parcial de banco** | Alto | Simular falha de banco após débito mas antes do crédito para validar rollback. Requer teste de integração com injeção de falhas. |
| **Testes de idempotência sob concorrência** | Alto | Enviar duas requisições com mesma Idempotency-Key simultaneamente. Apenas uma deve ser processada. |
| **Teste de contrato de API (contract testing)** | Médio | Validar que todos os códigos HTTP e schemas de resposta estão conforme documentação (seção 10). Considerar Pact ou similar. |
| **Teste de segurança (IDOR, token manipulation)** | Alto | Validar que manipulação de parâmetros não permite acessar dados de outros usuários ou debitar conta alheia. |
| **Teste de paridade staging/produção** | Médio | Validar que configurações críticas (timeout, limites, HTTPS) são idênticas entre ambientes. |
| **Teste de timeout end-to-end** | Médio | Simular latência na API > 10s e validar comportamento do cliente (retry com mesma key). |

---

## Recomendações de Teste

### Prioridade 1 — Executar antes do deploy (bloqueante)

1. **Teste de atomicidade:** Envio completo (happy path) → validar que saldo do remetente diminuiu, saldo do destinatário aumentou, transação registrada — tudo ou nada.
2. **Teste de idempotência:** Enviar mesma requisição (mesma Idempotency-Key) 2x → validar que o saldo foi debitado apenas 1x e a resposta é idêntica.
3. **Teste de concorrência de saldo:** 2 envios simultâneos do mesmo remetente com saldo para apenas 1 → validar que apenas 1 é processado e o saldo não fica negativo.
4. **Teste de contrato de API:** Validar cada código HTTP (200, 400, 401, 403, 404, 409, 422, 504) com o schema documentado.
5. **Teste de segurança básica:** Requisição sem token (401), token inválido (401), tentativa de IDOR na lista de transações.

### Prioridade 2 — Executar no dia do deploy (importante)

6. **Testes de valores limítrofes:** 0, 1, -1, 10.000, 10.001, decimal, string, null.
7. **Testes de combinação de status de conta:** Todas as combinações remetente/destinatário (ativo, bloqueado, pendente, inexistente).
8. **Teste de envio para si mesmo:** Validar rejeição com 409.
9. **Teste de rollback:** Simular falha durante a transação (se possível com injeção de falhas) → validar que nenhum saldo foi alterado.
10. **Teste de consistência App ↔ Painel:** Enviar no App → consultar no painel web → dados devem estar consistentes.

### Prioridade 3 — Automatizar a médio prazo

11. **Teste de carga:** Simular N usuários enviando simultaneamente para validar comportamento sob stress (locks, deadlocks, timeouts).
12. **Contract testing automatizado:** Implementar Pact ou similar para validar contrato de API continuamente.
13. **Teste de segurança automatizado:** SAST/DAST para detectar IDOR, SQL injection nos parâmetros de recipientId e amount.
14. **Teste de resiliência:** Chaos engineering no banco de dados para validar comportamento de rollback e retry.

---

## Decisão de Deploy

### Avaliação: **Requer mitigação pré-deploy**

| Critério | Status |
|----------|--------|
| Funcionalidade financeira (débito/crédito) | Risco inerente alto |
| Atomicidade validada | Deve ser confirmada antes do deploy |
| Idempotência validada sob concorrência | Deve ser confirmada antes do deploy |
| Contrato de API validado | Deve ser confirmado antes do deploy |
| Segurança básica (auth, IDOR) | Deve ser confirmada antes do deploy |

**Recomendação:** Não deployar até que os testes de **Prioridade 1** (itens 1-5) sejam executados e aprovados. Os riscos de inconsistência financeira (débito duplo, saldo negativo, atomicidade) são graves o suficiente para justificar esta exigência.

**Pós-deploy — monitoramento reforçado:**
- Monitorar saldo total do sistema (soma de todos os saldos deve permanecer constante — conservação de pontos)
- Alertas para transações com `Idempotency-Key` duplicada (indicativo de retry, mas também de possível falha)
- Alertas para respostas 5xx no endpoint de transferência
- Dashboard de latência do endpoint (P95, P99) para detectar degradação antes de timeouts

---

## Checklist de Análise RCRCRC

- [x] **Recent:** As mudanças recentes foram mapeadas; módulos adjacentes afetados foram identificados
- [x] **Core:** As funcionalidades essenciais do negócio foram listadas e incluídas no escopo de regressão
- [x] **Risky:** O histórico de incidentes e as áreas de alto risco inerente foram consultados e considerados
- [x] **Configuration-sensitive:** Variações de status de conta, valores limítrofes e ambientes foram avaliadas
- [x] **Conformance:** Requisitos de LGPD, contrato de API, segurança e auditoria foram verificados
- [x] **Complex:** Áreas de alta complexidade técnica (atomicidade, concorrência, idempotência) foram priorizadas
- [x] **Interações entre dimensões:** Risky + Complex (concorrência + atomicidade), Conformance + Recent (contrato de API nova)
- [x] **Lacunas de automação:** Testes de concorrência, falha parcial e contract testing identificados como gaps
- [x] **Top 3 riscos priorizados:** Atomicidade, débito duplo e saldo negativo têm plano de teste concreto
- [x] **Decisão de deploy:** Requer mitigação pré-deploy (testes Prioridade 1 obrigatórios)

---

*Análise gerada com base na heurística RCRCRC (James Bach) aplicada ao requisito QualiPoints v2.*
*Referência: Bach, J. (n.d.). RCRCRC Regression Testing Heuristic. Satisfice Inc.*
