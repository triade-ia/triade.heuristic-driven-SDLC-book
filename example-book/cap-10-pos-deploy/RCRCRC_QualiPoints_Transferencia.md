# Análise RCRCRC — Envio de QualiPoints entre Usuários

**Heurística aplicada:** RCRCRC (Recent, Core, Risky, Configuration-sensitive, Conformance, Complex)
**Requisito analisado:** REQ_INICIAL_V2 — Envio de QualiPoints entre usuários
**Data:** 2026-02-22
**Fase:** Cap10 — Pós-Deploy (MVP inicial da funcionalidade)
**Contexto:** Primeiro deploy da funcionalidade de transferência de QualiPoints. Toda a feature é "recente". A análise foca em onde concentrar esforços de regressão e validação pós-deploy.

---

## R — Recent (Recente)

> O que foi introduzido neste deploy? Essas são as áreas com maior probabilidade de introduzir regressões.

**Contexto:** Este é o deploy inicial da funcionalidade de transferência. Todos os componentes abaixo foram criados ou modificados para suportar o fluxo de envio.

### Áreas diretamente introduzidas neste deploy

| Área | Descrição |
|------|-----------|
| Endpoint `POST /api/v1/transfers` | Novo endpoint de transferência — toda a lógica de orquestração é nova |
| Lógica de débito/crédito atômico | Transação de banco que cobre débito do remetente, crédito do destinatário e inserção do registro |
| Mecanismo de Idempotency-Key | Lógica de detecção e retorno de requisições duplicadas com a mesma chave |
| Validações server-side | Regras de negócio novas: saldo, destinatário, valor, envio para si mesmo |
| Tabela/modelo de transações | Nova entidade persistida a cada envio bem-sucedido |
| Endpoint de transações recentes | Listagem das últimas 10 transações do usuário logado |
| Integração da autenticação com o novo endpoint | O token Bearer é interpretado para identificar o remetente |

### Módulos adjacentes com risco de efeito colateral

- **Módulo de saldo (carteira):** Toda lógica de leitura e escrita de saldo foi expandida para suportar débito e crédito transacionais. Consultas de saldo pré-existentes podem ter sido afetadas.
- **Módulo de autenticação/autorização:** A extração do identificador do usuário a partir do token agora é usada em mais um contexto. Qualquer inconsistência no parsing do token afeta diretamente o remetente identificado.
- **Listagem de transações:** Se já existia alguma listagem anterior de atividade do usuário, a nova tabela de transações pode ter alterado o schema ou a query subjacente.

### Questões críticas sobre o "recente"

- A adição da tabela de transações exigiu migrations de banco. As migrations foram aplicadas corretamente em produção?
- O mecanismo de idempotência usa armazenamento separado (tabela, cache)? Esse armazenamento foi provisionado corretamente no ambiente de produção?
- A lógica de rollback (atomicidade) foi testada em ambiente equivalente ao de produção, ou apenas em staging com dados sintéticos?

---

## C — Core (Principal)

> Quais funcionalidades essenciais do produto podem ser afetadas? Se quebrarem, a maioria dos usuários é impactada ou o negócio para.

### Funcionalidades core do produto QualiPoints

| Funcionalidade | Criticidade | Motivo |
|---------------|-------------|--------|
| Autenticação e sessão do usuário | Crítica | Sem login, nenhuma operação funciona — o endpoint de transferência exige token válido |
| Consulta de saldo | Alta | Exibida na tela de envio; usada na validação server-side; impactada pelo débito/crédito |
| Envio de QualiPoints (novo fluxo core) | Crítica | É o próprio objeto deste deploy — qualquer falha aqui é o defeito central |
| Listagem de transações recentes | Alta | Confirmação visual pós-envio; se a lista não atualizar, o usuário não sabe se a operação ocorreu |
| Idempotência (evitar débito duplo) | Crítica | Falha aqui resulta em perda financeira real para o usuário — debitar duas vezes é inaceitável |

### Caminhos críticos que precisam ser validados após qualquer deploy futuro

1. **Fluxo completo de envio:** login → informar destinatário e valor → confirmar → feedback de sucesso → saldo atualizado → transação na lista.
2. **Fluxo de retry seguro:** enviar → simular timeout → reenviar com mesma Idempotency-Key → verificar que o saldo foi debitado apenas uma vez.
3. **Consulta de saldo após envio:** o saldo do remetente exibido no App deve refletir o débito imediatamente após o retorno do 200.

---

## R — Risky (Arriscado)

> Onde a probabilidade de falha é maior ou o impacto de uma falha é severo?

### Riscos inerentes ao domínio

| Risco | Probabilidade | Severidade | Justificativa |
|-------|--------------|------------|---------------|
| Débito sem crédito (falha parcial na transação atômica) | Média | Crítica | Se o rollback não funcionar corretamente, o remetente perde saldo sem que o destinatário receba |
| Duplo débito por falha na Idempotency-Key | Média | Crítica | Retry legítimo do cliente (timeout de rede) + falha no mecanismo de idempotência = débito duplicado |
| Race condition no saldo | Média | Alta | Dois envios simultâneos do mesmo remetente podem ler o mesmo saldo antes de qualquer débito ser persistido |
| Identificação errada do remetente via token | Baixa | Crítica | Erro no parsing do token pode debitar a conta errada — impacto de segurança e financeiro |
| Timeout não tratado corretamente (504/408) | Média | Alta | A transação pode ter sido concluída no servidor, mas o cliente não recebeu resposta — usuário tenta de novo sem a mesma Idempotency-Key |
| Falha silenciosa no registro de transação | Baixa | Alta | Débito e crédito ocorrem, mas o registro não é persistido — operação invisível para auditoria e lista recente |
| Destinatário recebe saldo sem que remetente seja debitado | Baixa | Alta | Falha no rollback na direção oposta |

### Operações irreversíveis que exigem atenção especial

- O débito do saldo do remetente é **irreversível** sem um fluxo de estorno (não previsto no MVP).
- O registro da transação na tabela de auditoria deve ser imutável — qualquer lógica de update ou delete nessa tabela é um risco.

### Histórico de risco (MVP — sem histórico, mas risco estrutural)

Por ser o primeiro deploy, não há histórico de incidentes. O risco se concentra nas áreas de maior complexidade técnica (atomicidade + idempotência + concorrência), que são justamente as mais difíceis de testar exaustivamente antes do go-live.

---

## C — Configuration-sensitive (Sensível à Configuração)

> O comportamento muda conforme configurações de ambiente, dados ou flags?

### Variáveis de ambiente e configuração críticas

| Configuração | Risco de Divergência Staging → Produção |
|-------------|----------------------------------------|
| Timeout da API (10 segundos) | Em produção, latência de banco e carga real podem alterar o comportamento de timeout — staging pode não replicar isso |
| Conexão com banco de dados (pool, isolamento de transação) | O nível de isolamento (`REPEATABLE READ`, `SERIALIZABLE`) afeta diretamente o comportamento do saldo concorrente |
| Armazenamento da Idempotency-Key (Redis, banco, memória) | Se usar cache em memória, restarts do serviço em produção invalidam chaves — cliente pode sofrer duplo débito |
| HTTPS / TLS | Certificado e redirecionamentos HTTP→HTTPS devem estar ativos e corretos em produção |
| Configuração de autenticação (segredo do token, expiração) | Diferença de configuração entre ambientes pode tornar tokens de staging inválidos em produção ou vice-versa |

### Variações por perfil de usuário

No MVP, apenas o perfil **usuário final** envia e recebe. Contudo, é importante validar:
- Usuários com conta **bloqueada** recebem 403 (não conseguem enviar)?
- Usuários com conta **pendente ou em análise** são corretamente bloqueados?
- O comportamento é idêntico para contas recém-criadas (com saldo zero)?

### Feature flags

O requisito não menciona feature flags. Se o deploy foi feito com a feature ativa para 100% dos usuários desde o início, não há risco de variação por flag. Contudo, se houver algum mecanismo de rollout gradual não documentado, é necessário validar em todos os grupos.

---

## C — Conformance (Conformidade)

> A funcionalidade respeita padrões regulatórios, contratos de API, segurança e políticas internas?

### Contrato de API

O requisito especifica um contrato explícito (seção 10 do REQ_INICIAL_V2). Após o deploy, validar:

| Aspecto do Contrato | O que verificar |
|--------------------|-----------------|
| Status codes | 200, 400, 401, 403, 404, 409, 422, 408/504, 5xx — cada situação retorna o código correto? |
| Códigos de erro no body | `INVALID_AMOUNT`, `RECIPIENT_NOT_FOUND`, `SELF_TRANSFER_NOT_ALLOWED`, `INSUFFICIENT_BALANCE`, `ACCOUNT_NOT_ACTIVE`, `REQUEST_TIMEOUT`, `INTERNAL_ERROR`, `UNAUTHORIZED` — presentes e com o campo `code` correto? |
| Formato do body de sucesso | `transactionId`, `amount`, `recipientId`, `status: "completed"`, `completedAt` (ISO8601) — todos presentes e no formato correto? |
| Header `Idempotency-Key` obrigatório | A API rejeita requisições sem esse header? Com qual código? |
| Header `Content-Type: application/json` | A API rejeita requisições com content-type incorreto? |

### Segurança

| Requisito de Segurança | Conformidade esperada |
|-----------------------|----------------------|
| HTTPS obrigatório | Requisições em HTTP puro devem ser recusadas ou redirecionadas para HTTPS |
| Autenticação Bearer | Requisições sem `Authorization` retornam 401; tokens expirados ou inválidos retornam 401 |
| Controle de acesso às transações | Usuário A não pode ver transações do usuário B — a API filtra pelo token, não por parâmetro na URL |
| Auditoria | Toda tentativa de envio (sucesso ou falha) deve gerar registro — verificar logs ou tabela de auditoria |

### LGPD / Privacidade

O produto lida com dados de usuários (ID, saldo, histórico de transações). Pontos de atenção:
- Os logs de auditoria contêm dados pessoais? Há retenção e acesso controlados?
- A listagem de transações recentes expõe apenas os dados do próprio usuário (sem dados de terceiros além do `recipientId` necessário)?

---

## C — Complex (Complexo)

> Quais partes têm alta complexidade técnica e são mais propensas a falhas ocultas?

### Operação atômica (maior complexidade técnica do requisito)

A operação de envio exige três passos em uma única transação de banco:
1. Débito do saldo do remetente
2. Crédito do saldo do destinatário
3. Inserção do registro da transação

**Falhas ocultas possíveis:**
- O banco garante atomicidade? O nível de isolamento da transação está configurado corretamente?
- Em caso de deadlock entre duas transferências simultâneas (usuário A → B e usuário B → A ao mesmo tempo), o sistema faz rollback e retorna erro? Ou bloqueia indefinidamente até o timeout?
- Se o passo 3 (inserção do registro) falhar após os passos 1 e 2 terem sido escritos em buffer (mas antes do commit), o rollback cobre os três?

### Mecanismo de Idempotência

**Cenários complexos:**
- Primeira requisição processa e grava o resultado; segunda requisição com a mesma chave chega **enquanto a primeira ainda está processando** — race condition no mecanismo de idempotência.
- Primeira requisição retorna erro 5xx (erro interno); segunda requisição com a mesma chave deve retornar o mesmo erro 5xx (não reprocessar). Esse comportamento está implementado?
- A chave de idempotência tem TTL (tempo de expiração)? Se sim, o que acontece com uma chave expirada — é tratada como nova requisição ou como erro?

### Concorrência de saldo

O requisito menciona proteção contra "duas requisições simultâneas do mesmo usuário". Cenários de teste:
- Usuário envia 2 requisições simultâneas com Idempotency-Keys **diferentes** mas mesmo remetente e mesmo saldo — apenas uma deve ser bem-sucedida, a segunda deve retornar 422 (saldo insuficiente).
- Não é suficiente ler o saldo antes de debitar; o lock deve ser adquirido antes da leitura ou a operação deve usar `SELECT FOR UPDATE` (ou equivalente).

### Timeout e consistência (estado ambíguo)

O fluxo mais complexo do ponto de vista do cliente:
1. Cliente envia requisição
2. Servidor processa e **commita** a transação
3. Servidor tenta enviar a resposta, mas a conexão caiu
4. Cliente não recebeu resposta — não sabe se a transação ocorreu
5. Cliente reenvia com a mesma Idempotency-Key → deve receber 200 com o resultado original

Se o mecanismo de idempotência não persistir o resultado **antes** de enviar a resposta, ou se a persistência da chave ocorrer só após o sucesso da resposta, esse cenário pode falhar silenciosamente.

---

## 1. Mapa de Prioridades

| Dimensão | Áreas Identificadas | Prioridade |
|----------|--------------------|-----------:|
| **Recent** | Endpoint de transferência, lógica atômica, idempotência, tabela de transações, listagem recente | Alta |
| **Core** | Fluxo completo de envio, consulta de saldo pós-envio, retry seguro com idempotência, autenticação | Crítica |
| **Risky** | Débito sem crédito, duplo débito, race condition de saldo, timeout com transação já commitada | Crítica |
| **Configuration-sensitive** | Timeout, isolamento de transação no banco, armazenamento da Idempotency-Key, HTTPS | Alta |
| **Conformance** | Contrato de API (status codes, códigos de erro, formato do body), segurança, auditoria | Alta |
| **Complex** | Atomicidade sob falha parcial, race condition na idempotência, concorrência de saldo, timeout + inconsistência de estado | Crítica |

---

## 2. Top Riscos de Regressão

| # | Risco | Severidade | Probabilidade | Dimensão(ões) |
|---|-------|-----------|--------------|---------------|
| 1 | Duplo débito por falha no mecanismo de Idempotency-Key em retry | Crítica | Média | Risky + Complex |
| 2 | Falha parcial na transação atômica (débito sem crédito, ou crédito sem registro) | Crítica | Média | Risky + Complex |
| 3 | Race condition: duas requisições simultâneas com saldo suficiente para apenas uma debitam o saldo duas vezes | Crítica | Média | Risky + Complex |
| 4 | Estado ambíguo pós-timeout: transação commitada no servidor, cliente não sabe e reenvia sem Idempotency-Key | Alta | Alta | Complex + Conformance |
| 5 | Consulta de saldo não reflete o débito imediatamente após o envio bem-sucedido | Alta | Média | Core + Recent |
| 6 | Contrato de API quebrado: código de erro ou status HTTP incorreto em cenários de falha | Alta | Baixa | Conformance |
| 7 | Usuário com conta bloqueada consegue enviar porque a validação de status não está no servidor | Alta | Baixa | Core + Conformance |
| 8 | Log de auditoria não registra tentativas com falha (apenas sucessos) | Média | Média | Conformance |
| 9 | Armazenamento da Idempotency-Key em memória — chaves perdidas após restart do serviço em produção | Alta | Baixa | Configuration-sensitive |
| 10 | Listagem de transações retorna transações de outros usuários por falha no filtro por token | Crítica | Baixa | Conformance + Core |

---

## 3. Perguntas em Aberto

1. **Armazenamento da Idempotency-Key:** Onde as chaves são persistidas (banco de dados, Redis, memória)? Qual o TTL da chave? O que acontece se a chave expirar e o cliente reenviar?

2. **Nível de isolamento da transação:** Qual o nível de isolamento configurado no banco de dados para a operação de envio? `READ COMMITTED`, `REPEATABLE READ` ou `SERIALIZABLE`? Como o sistema lida com deadlocks?

3. **Race condition na idempotência:** A implementação usa lock otimista ou pessimista para evitar que duas requisições simultâneas com a mesma Idempotency-Key sejam processadas ao mesmo tempo?

4. **Comportamento do timeout do lado do servidor:** Se o processamento interno levar mais de 10 segundos, o que acontece com a transação de banco em andamento? É feito rollback antes de retornar 504, ou a transação pode ter sido commitada?

5. **Migrations de banco:** As migrations necessárias para a tabela de transações e o armazenamento de Idempotency-Keys foram aplicadas com sucesso em produção e validadas antes de ativar o endpoint?

6. **Rollout:** A feature foi ativada para 100% dos usuários ou há rollout gradual? Se gradual, como o tráfego é roteado e o que acontece com usuários fora do grupo de rollout?

7. **Auditoria de falhas:** Os logs de auditoria registram tentativas de envio que falham com 4xx (ex.: saldo insuficiente, destinatário inválido), ou apenas as bem-sucedidas?

8. **Expiração de token durante o envio:** Se o token expirar entre o início da requisição e o processamento no servidor, qual é o comportamento? 401 antes do débito ocorrer?

---

## 4. Lacunas de Cobertura

| Área sem cobertura adequada | Risco exposto |
|----------------------------|--------------|
| Testes de carga/concorrência: múltiplos usuários enviando simultaneamente para o mesmo destinatário | Race condition no crédito do destinatário |
| Testes de carga/concorrência: mesmo remetente enviando duas requisições simultâneas com Idempotency-Keys diferentes | Race condition no saldo do remetente |
| Testes de chaos/injeção de falha no step 2 (crédito) e step 3 (inserção do registro) da transação atômica | Verificação do rollback parcial |
| Testes de timeout controlado: simular o servidor demorar exatamente 10s e verificar o estado da transação | Estado ambíguo pós-timeout |
| Testes de retry com mesma Idempotency-Key enquanto a primeira requisição ainda está processando | Race condition no mecanismo de idempotência |
| Testes de contrato (contract testing) para todos os cenários de erro da API | Garantia de conformidade do contrato |
| Testes de acesso cruzado: usuário A tentando ver transações do usuário B via manipulação de endpoint | Verificação do controle de acesso |
| Testes de restart do serviço durante processamento de requisição | Consistência de estado após reinício |

---

## 5. Recomendações de Teste

### Imediato (executar agora, pós-deploy)

1. **Smoke test do fluxo completo:** Executar um envio real ponta-a-ponta em produção com valores baixos (ex.: 1 QualiPoint) — verificar 200, débito do remetente, crédito do destinatário, registro na lista recente.

2. **Validar todos os cenários de erro do contrato:** Testar cada código de erro esperado (400, 401, 403, 404, 409, 422) com payloads específicos — confirmar que o `code` e o `message` no body correspondem ao especificado.

3. **Retry com mesma Idempotency-Key:** Após um envio bem-sucedido (200), reenviar a exata mesma requisição com a mesma chave — confirmar que o retorno é idêntico (mesmo 200, mesmo `transactionId`) e que o saldo NÃO foi debitado novamente.

4. **Validar consulta de saldo pós-envio:** Após o envio, consultar o saldo do remetente e do destinatário — confirmar que o débito e o crédito estão refletidos.

5. **Validar filtro de acesso às transações:** Com dois usuários distintos, confirmar que cada um vê apenas as próprias transações na listagem.

### Curto prazo (próximos ciclos de deploy)

6. **Testes de concorrência de saldo:** Enviar duas requisições simultâneas com Idempotency-Keys diferentes do mesmo remetente com saldo suficiente para apenas uma — verificar que apenas uma é bem-sucedida e o saldo não vai negativo.

7. **Injeção de falha na transação atômica:** Em ambiente de staging com injeção de falha no passo de crédito ou registro, verificar que o rollback é completo e o saldo do remetente não foi debitado.

8. **Testes de timeout:** Simular latência artificial no servidor acima de 10 segundos — verificar que o cliente recebe 504/408 e que a transação não foi commitada (ou, se foi, que o retry com a mesma Idempotency-Key retorna o resultado correto).

### Médio prazo (automatizar)

9. **Contract tests automatizados:** Automatizar a validação do contrato de API para cada cenário (todos os status codes e códigos de erro), executar em cada pipeline de CI antes do deploy.

10. **Testes de segurança (OWASP):** Verificar ausência de IDOR (Insecure Direct Object Reference) na listagem de transações; validar que manipular `recipientId` no body não permite acesso a dados não autorizados.

---

## 6. Decisão de Deploy

> **Status atual:** MVP inicial — este deploy já ocorreu. A avaliação abaixo orienta o monitoramento pós-deploy e a decisão de manter, reverter ou mitigar.

### Avaliação de Risco Consolidada

**Nível de risco:** 🟠 **Alto — Deploy com monitoramento reforçado e mitigações imediatas**

### Condições para manter o deploy ativo

| Condição | Status |
|----------|--------|
| Smoke test do fluxo completo bem-sucedido em produção | A verificar |
| Todos os cenários de erro retornam os códigos e `code` corretos | A verificar |
| Retry com mesma Idempotency-Key não gera débito duplo | A verificar (crítico) |
| Saldo refletido corretamente após envio | A verificar |
| Filtro de acesso às transações funcionando | A verificar |

### Condições para rollback imediato

- Qualquer evidência de débito sem crédito ou crédito sem débito em produção.
- Qualquer evidência de duplo débito em retry.
- Saldo do usuário indo abaixo de zero.
- Falha na autenticação afetando o remetente identificado.

### Monitoramento recomendado (primeiras 48h)

- Alert em erros 5xx no endpoint `POST /api/v1/transfers` (threshold: > 1% de requisições).
- Alert em discrepâncias de saldo: `SUM(débitos) ≠ SUM(créditos)` na tabela de transações.
- Monitorar latência do endpoint — proximidade com o limite de 10s indica risco de timeout em produção.
- Monitorar tentativas de retry (mesma Idempotency-Key) — pico de retries indica problemas de timeout ou conectividade.

---

## Checklist de Análise RCRCRC

- [x] **Recent:** As mudanças recentes foram mapeadas; módulos adjacentes afetados foram identificados
- [x] **Core:** As funcionalidades essenciais do negócio foram listadas e incluídas no escopo de regressão
- [x] **Risky:** As áreas de alto risco inerente foram identificadas e priorizadas
- [x] **Configuration-sensitive:** Variações de ambiente, timeout e armazenamento de idempotência foram avaliadas
- [x] **Conformance:** Contrato de API, segurança, auditoria e privacidade foram verificados
- [x] **Complex:** Atomicidade, idempotência concorrente, race condition de saldo e timeout foram analisados
- [x] **Interações entre dimensões:** Dimensões sobrepostas identificadas (Risky + Complex, Core + Conformance)
- [x] **Lacunas de automação:** Áreas sem cobertura adequada documentadas
- [x] **Top 3 riscos priorizados:** Duplo débito, falha atômica e race condition têm planos de teste concretos
- [x] **Decisão de deploy:** Recomendação emitida — deploy com monitoramento reforçado e smoke tests obrigatórios

---

*Análise gerada com base no REQ_INICIAL_V2 (Envio de QualiPoints entre usuários) aplicando a heurística RCRCRC de James Bach.*
*Referência: Bach, J. (n.d.). RCRCRC Regression Testing Heuristic. Satisfice Inc.*
