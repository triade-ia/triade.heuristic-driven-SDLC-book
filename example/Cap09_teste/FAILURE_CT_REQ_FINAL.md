# Casos de Teste FAILURE: Envio de QualiPoints (REQ_FINAL)

## Contexto do Caso

Este documento apresenta os **casos de teste** baseados na heurística **FAILURE** (Ben Simo) para o requisito de **Envio de QualiPoints** ([setUp/REQ_FINAL.MD](../../setUp/REQ_FINAL.MD)). O objetivo é validar o comportamento do sistema em cenários de falha nas sete dimensões: **Functional**, **Appropriate**, **Impact**, **Log**, **UI**, **Recovery** e **Emotions**, garantindo que as falhas ocorram de forma controlada, transparente e recuperável.

## Entradas e Contrato em Análise

| Entrada | Tipo | Origem | Uso |
|--------|------|--------|-----|
| `recipient_id` | string | Body (JSON) | Identificador do destinatário |
| `amount` | number | Body (JSON) | Valor em QualiPoints a transferir |
| `user_id` | (implícito) | JWT token | Remetente — nunca do body |
| `idempotency-key` | string | Header | Evitar transferências duplicadas |

**Endpoint:** `POST /api/v1/transactions`  
**Fluxo:** Transferência síncrona com transação atômica (débito + crédito); rollback se crédito no destinatário falhar. Resposta em tempo real (Sucesso ou Erro).  
**Cliente:** API e Painel Web (visualização de histórico; envio com formulário).

---

## 1. Functional (Funcional)

**Objetivo:** Verificar que o sistema mantém consistência dos dados em falha, que não há débito sem crédito e que falhas de dependência não causam cascata.

| ID | Cenário | Pré-condições | Passos | Resultado esperado | Critério de sucesso |
|----|---------|----------------|--------|---------------------|---------------------|
| F-FUN-01 | Rollback quando crédito no destinatário falha | Ambiente onde o crédito no destinatário pode falhar (ex.: constraint, destinatário inativado durante transação) | 1. Disparar transferência válida. 2. Simular ou provocar falha no crédito ao destinatário após débito. | Rollback automático; saldo do remetente inalterado; nenhuma alteração persistida. | Saldo do remetente igual ao anterior; nenhuma linha de transação criada ou transação criada com rollback; resposta de erro (5xx ou 4xx conforme implementação). |
| F-FUN-02 | Erro 402/404/422 — nenhuma alteração em saldo | Remetente com saldo conhecido | 1. Enviar requisição que resulte em 402 (saldo insuficiente), 404 (destinatário inexistente) ou 422 (valor inválido). 2. Verificar saldo do remetente após a resposta. | Nenhum débito é aplicado em caso de erro de validação ou negócio. | Saldo do remetente inalterado após 402, 404 e 422. |
| F-FUN-03 | Banco indisponível — resposta definida | Banco de dados indisponível ou inacessível | 1. Simular indisponibilidade do banco. 2. Enviar POST /api/v1/transactions. 3. Verificar status e corpo da resposta. | API retorna 503 (ou 500) sem executar débito; nenhuma alteração em saldos. | Status 503 ou 500; corpo sem indicar que a transferência foi processada; saldos inalterados. |
| F-FUN-04 | Falha em uma requisição não afeta outras | Múltiplas requisições de usuários diferentes | 1. Disparar transferência do usuário A que falha (ex.: 402). 2. Em paralelo ou em sequência, disparar transferência válida do usuário B. | Falha da requisição de A não bloqueia ou corrompe a de B; isolamento por requisição. | Transferência de B concluída com 201; saldos de B e destinatário de B corretos. |

---

## 2. Appropriate (Apropriado)

**Objetivo:** Garantir que códigos HTTP e mensagens de erro sejam adequados ao contexto e acionáveis.

| ID | Cenário | Pré-condições | Passos | Resultado esperado | Critério de sucesso |
|----|---------|----------------|--------|---------------------|---------------------|
| F-APP-01 | 402 — saldo insuficiente | Saldo do remetente menor que amount | 1. Enviar amount maior que o saldo. 2. Verificar status e corpo da resposta. | Status 402; corpo com mensagem clara e, se aplicável, código (ex.: INSUFFICIENT_BALANCE). | HTTP 402; mensagem em linguagem clara (ex.: "Saldo insuficiente para esta transferência."); estrutura de erro consistente. |
| F-APP-02 | 404 — destinatário inexistente ou inativo | recipient_id inexistente ou conta inativa | 1. Enviar com recipient_id inexistente ou inativo. 2. Verificar status e corpo. | Status 404; mensagem indicando que o destinatário não foi encontrado ou não está ativo. | HTTP 404; mensagem clara (ex.: "Destinatário não encontrado ou não está ativo."). |
| F-APP-03 | 422 — valor abaixo do mínimo ou formato inválido | amount = 0, negativo ou não numérico | 1. Enviar amount = 0, negativo ou tipo inválido. 2. Verificar status e corpo. | Status 422; mensagem ou detalhes por campo indicando a regra violada. | HTTP 422; mensagem ou campo de erro indicando valor mínimo (1 QualiPoint) ou formato; corpo estruturado (ex.: detalhes por campo). |
| F-APP-04 | 400 — requisição malformada | Body inválido (ex.: JSON malformado, campos obrigatórios ausentes) | 1. Enviar body sem recipient_id ou amount, ou JSON inválido. 2. Verificar status. | Status 400; mensagem indicando que a requisição é inválida. | HTTP 400; mensagem genérica de requisição inválida. |
| F-APP-05 | 401 — token ausente ou inválido | Sem Authorization ou token expirado/inválido | 1. Enviar sem header Authorization ou com token inválido/expirado. 2. Verificar status. | Status 401; não processar transferência. | HTTP 401; nenhuma alteração em saldos. |
| F-APP-06 | 500/503 — mensagem genérica sem detalhes internos | Erro interno ou serviço indisponível | 1. Provocar 500 ou 503. 2. Verificar corpo da resposta. | Mensagem genérica (ex.: "Erro interno" ou "Serviço temporariamente indisponível"); sem stack trace ou detalhes de implementação. | Corpo não expõe stack trace, caminhos de arquivo ou mensagens de exceção internas; mensagem adequada ao cliente. |

---

## 3. Impact (Impacto)

**Objetivo:** Garantir que em falha não haja corrupção de dados, perda de trabalho do usuário ou impacto não mitigado.

| ID | Cenário | Pré-condições | Passos | Resultado esperado | Critério de sucesso |
|----|---------|----------------|--------|---------------------|---------------------|
| F-IMP-01 | Nenhum débito em 402, 404, 422 | Saldo do remetente conhecido | 1. Provocar 402, 404 e 422. 2. Consultar saldo do remetente após cada resposta. | Saldo permanece igual ao anterior; usuário não perde pontos. | Saldo inalterado em todos os três cenários. |
| F-IMP-02 | Idempotency-key evita duplicata em retentativa | Transferência válida já enviada com idempotency-key K | 1. Enviar transferência com idempotency-key K; receber 201. 2. Reenviar mesma requisição com mesma key K (ex.: retry após timeout). 3. Verificar resposta e saldos. | Segunda resposta 201 com mesmo transaction_id; apenas uma transferência efetiva. | Uma única transferência no banco; mesmo transaction_id na segunda resposta; saldo debitado uma vez. |
| F-IMP-03 | Preservação de estado no Painel após 4xx | Painel com formulário preenchido | 1. Preencher recipient_id e amount. 2. Enviar e receber 402 ou 404 ou 422. 3. Verificar tela. | Campos permanecem preenchidos; usuário pode corrigir e retentar sem refazer tudo. | recipient_id e amount preservados; mensagem de erro exibida; botão de reenviar disponível. |
| F-IMP-04 | Resposta 500/503 — orientação de retry | API ou Painel retorna 500 ou 503 | 1. Receber 500 ou 503. 2. Verificar se a mensagem sugere retentar. | Mensagem indica problema temporário e sugere "Tente novamente em instantes" (ou equivalente). | Corpo de erro ou mensagem no Painel com tom de problema temporário e sugestão de retry; se aplicável, header Retry-After. |

---

## 4. Log (Logs)

**Objetivo:** Garantir que falhas sejam registradas com contexto suficiente para diagnóstico, sem expor dados sensíveis.

| ID | Cenário | Pré-condições | Passos | Resultado esperado | Critério de sucesso |
|----|---------|----------------|--------|---------------------|---------------------|
| F-LOG-01 | Falha 4xx/5xx registrada com contexto | Logs acessíveis (ambiente de teste ou staging) | 1. Provocar 402, 404, 422 ou 500. 2. Verificar entradas de log correspondentes. | Cada falha gera registro com nível adequado (ex.: WARN para 4xx, ERROR para 5xx) e contexto (request_id ou correlation_id, user_id se seguro, código/status). | Log contém identificador de requisição (request_id/correlation_id), status/código de erro; suficiente para correlacionar com a chamada. |
| F-LOG-02 | Logs não expõem JWT nem dados sensíveis | Idem | 1. Provocar qualquer erro. 2. Buscar nos logs por token, senha ou PII. | JWT, senhas e dados pessoais não aparecem em claro nos logs. | Nenhum log com token completo, senha ou PII em texto claro. |
| F-LOG-03 | 500 — identificador para suporte | Resposta 500 retornada ao cliente | 1. Provocar 500. 2. Verificar se a resposta inclui identificador de correlação (ex.: request_id) e se o mesmo existe no log. | Cliente recebe identificador (ex.: no corpo ou header) para informar ao suporte; mesmo ID no log. | Corpo de erro ou header contém request_id (ou similar); mesmo valor registrado no log para rastreio. |

---

## 5. UI (Interface do Usuário)

**Objetivo:** Garantir que a interface reflita o estado de falha de forma clara e permaneça utilizável.

| ID | Cenário | Pré-condições | Passos | Resultado esperado | Critério de sucesso |
|----|---------|----------------|--------|---------------------|---------------------|
| F-UI-01 | Loading durante POST (Painel) | Painel de envio | 1. Clicar em enviar. 2. Observar até a resposta. | Indicador de carregamento visível e/ou botão desabilitado durante o request. | Botão desabilitado e/ou spinner/loading visível; após resposta, loading some e resultado ou erro é exibido. |
| F-UI-02 | Mensagem de erro exibida após 4xx/5xx (Painel) | Erro retornado ao Painel | 1. Causar 402, 404, 422 ou 500/503. 2. Verificar a tela. | Mensagem de erro visível, em linguagem compreensível (não técnica em excesso). | Texto da mensagem exibido; usuário entende que houve erro e qual o tipo (saldo, destinatário, valor, problema temporário). |
| F-UI-03 | UI não trava após erro (Painel) | Qualquer erro no envio | 1. Provocar erro. 2. Verificar se a tela responde (scroll, cliques, novo envio). | Interface permanece utilizável; não há loading infinito nem tela congelada. | É possível fechar mensagem de erro, alterar campos e reenviar ou navegar para outra tela. |
| F-UI-04 | Estado do botão após falha (Painel) | Erro 4xx ou 5xx no Painel | 1. Receber erro no envio. 2. Verificar estado do botão de envio. | Botão reabilitado para permitir correção e retry (ou ação clara de "Tentar novamente"). | Botão de envio não permanece desabilitado indefinidamente; usuário pode retentar. |

---

## 6. Recovery (Recuperação)

**Objetivo:** Garantir que o usuário possa se recuperar do erro com estado preservado e opções claras.

| ID | Cenário | Pré-condições | Passos | Resultado esperado | Critério de sucesso |
|----|---------|----------------|--------|---------------------|---------------------|
| F-REC-01 | Retentativa com mesma idempotency-key | Primeira requisição com key K retornou 201 | 1. Reenviar mesma requisição com mesma idempotency-key K. 2. Verificar resposta. | 201 com mesmo transaction_id; nenhuma segunda transferência. | Resposta 201; mesmo transaction_id; uma única transferência no banco. |
| F-REC-02 | 422 — mensagem permite corrigir e reenviar | Requisição com amount inválido | 1. Receber 422. 2. Verificar mensagem e comportamento no Painel. | Mensagem indica o campo e a regra; usuário pode ajustar e reenviar. | Detalhe do erro (ex.: valor mínimo) presente; formulário preservado; usuário consegue corrigir amount e reenviar. |
| F-REC-03 | 503/429 — Retry-After ou indicação de retry (se aplicável) | API retorna 503 ou 429 | 1. Receber 503 ou 429. 2. Verificar headers e corpo. | Se houver política de retry, header Retry-After ou mensagem sugere quando retentar. | Documentação ou resposta indicam que é temporário; Retry-After presente se implementado. |
| F-REC-04 | Opção de "Tentar novamente" após 500/503 (Painel) | Painel recebe 500 ou 503 | 1. Provocar 500 ou 503 no Painel. 2. Verificar se há ação para retentar. | Botão ou link "Tentar novamente" (ou reenviar) visível. | Usuário pode clicar e reenviar sem precisar recarregar a página manualmente. |

---

## 7. Emotions (Emoções)

**Objetivo:** Garantir que as mensagens de falha sejam neutras, empáticas e com próximo passo claro.

| ID | Cenário | Pré-condições | Passos | Resultado esperado | Critério de sucesso |
|----|---------|----------------|--------|---------------------|---------------------|
| F-EMO-01 | Mensagens não culpabilizam o usuário | Cenários 402, 404, 422 | 1. Provocar cada erro. 2. Ler a mensagem retornada. | Texto descreve o problema de forma neutra e sugere ação; evita "Você errou", "Sua solicitação é inválida" sem contexto. | Mensagens focadas em "saldo insuficiente", "destinatário não encontrado", "valor abaixo do mínimo" com próximo passo (ex.: "Verifique seu saldo e o valor informado."). |
| F-EMO-02 | 500/503 — tom de problema temporário | Resposta 500 ou 503 | 1. Receber 500 ou 503. 2. Verificar mensagem no corpo e, se aplicável, no Painel. | Comunicação transmite que o problema é temporário e que pode tentar novamente. | Mensagem do tipo "Problema temporário. Tente novamente em instantes." ou equivalente; sem tom de culpa ou de falha definitiva. |
| F-EMO-03 | Usuário informado e com opções (Painel) | Qualquer erro exibido no Painel | 1. Causar erro no envio. 2. Verificar mensagem e ações disponíveis. | Usuário entende o que aconteceu e vê opção clara para retentar ou corrigir. | Mensagem exibida + botão/link para retentar ou corrigir; usuário não fica sem ação. |

---

## Resumo por Dimensão

| Dimensão | Quantidade de casos | Foco principal |
|----------|---------------------|----------------|
| Functional | 4 | Rollback, nenhum débito em 4xx, banco indisponível, isolamento |
| Appropriate | 6 | Códigos 402/404/422/400/401/500/503 e mensagens adequadas |
| Impact | 4 | Integridade de saldo, idempotência, preservação de estado, orientação de retry |
| Log | 3 | Contexto para diagnóstico, sem dados sensíveis, request_id |
| UI | 4 | Loading, mensagem de erro, UI não trava, estado do botão |
| Recovery | 4 | Idempotency-key, 422 acionável, Retry-After, opção de retry no Painel |
| Emotions | 3 | Tom neutro, 500/503 temporário, usuário com opções |

---

## Checklist FAILURE Aplicado ao Requisito

- [ ] **Functional**: Rollback garantido (F-FUN-01); nenhum débito em 402/404/422 (F-FUN-02); resposta em indisponibilidade (F-FUN-03); isolamento (F-FUN-04).
- [ ] **Appropriate**: Códigos e mensagens por cenário (F-APP-01 a F-APP-06).
- [ ] **Impact**: Saldos íntegros; idempotência; estado preservado no Painel; orientação de retry (F-IMP-01 a F-IMP-04).
- [ ] **Log**: Logs com contexto; sem dados sensíveis; request_id para 500 (F-LOG-01 a F-LOG-03).
- [ ] **UI**: Loading, mensagem clara, UI utilizável, botão após falha (F-UI-01 a F-UI-04).
- [ ] **Recovery**: Idempotency-key; 422 acionável; Retry-After quando aplicável; opção de retry (F-REC-01 a F-REC-04).
- [ ] **Emotions**: Mensagens neutras e com próximo passo; 500/503 com tom temporário (F-EMO-01 a F-EMO-03).

---

## Conclusão

Os casos de teste acima cobrem o fluxo de **Envio de QualiPoints** sob as sete dimensões da heurística FAILURE. Eles validam que, em cenários de falha, o sistema mantém integridade dos dados (Functional, Impact), responde de forma apropriada (Appropriate), registra falhas com segurança (Log), reflete o estado na interface (UI), permite recuperação (Recovery) e comunica-se de forma empática (Emotions). A execução desses casos contribui para que o sistema falhe de forma **controlada, transparente e recuperável**.

*Heurística FAILURE — Ben Simo. Uma lente abrangente para o teste de cenários de falha.*
