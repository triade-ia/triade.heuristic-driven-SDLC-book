# Casos de Teste E2E: Envio de QualiPoints

**Data**: 2026-03-29
**Relatório base**: [test-strategy-qualipoints-envio-20260329.md](../strategy/test-strategy-qualipoints-envio-20260329.md)
**Guide**: [TEST_E2E_GUIDE.md](../../../docs/guides/TEST_E2E_GUIDE.md)
**Template**: [TEMPLATE_CT.MD](../../../docs/templates/TEMPLATE_CT.MD)

---

## [E2E-TRF-01] - Validar envio de QualiPoints com sucesso no fluxo completo (login, envio, confirmação, saldo e lista)

### Pre-condiciones
- Usuario remetente cadastrado com credenciais validas (email: sender@test.com, senha valida)
- Usuario destinatario cadastrado e com conta ativa no sistema (email: recipient@test.com)
- Saldo disponivel do remetente maior que 500 QualiPoints
- Ambiente completo acessivel via navegador (App ou Painel Web)
- Servico de autenticacao e API `POST /api/v1/transfers` disponiveis

### Step by step
1. Abrir o aplicativo e acessar a tela de login
2. No campo 'Email' informar "sender@test.com" >> No campo 'Senha' informar a senha valida
3. Clicar em [Entrar] >> Aguardar redirecionamento para o painel principal
4. Verificar o saldo exibido no painel principal e anotar o valor atual (ex: "1000 QualiPoints")
5. Clicar em [Transferir] no menu de navegacao
6. No campo 'Destinatario' informar "recipient@test.com"
7. No campo 'Valor' informar "500"
8. Verificar que o botao [Enviar] esta habilitado
9. Clicar em [Enviar]
10. Aguardar a mensagem de processamento ser exibida (loading/spinner)
11. Aguardar a mensagem de confirmacao ser exibida
12. Verificar o saldo atualizado no painel principal
13. Acessar a tela de transacoes via [Menu] >> [Transacoes]
14. Verificar o primeiro registro na lista de transacoes recentes

### Resultado esperado
- O sistema exibe indicador de carregamento (spinner) durante o processamento da transferencia
- O sistema exibe mensagem de sucesso apos a conclusao (ex: "Transferencia realizada com sucesso")
- O saldo do remetente e atualizado e exibe o valor anterior menos 500 QualiPoints (ex: "500 QualiPoints")
- Na tela de transacoes, o primeiro registro exibe: destinatario "recipient@test.com", valor "500", status "Concluida" e data/hora da transacao
- A lista de transacoes recentes esta ordenada por data decrescente (mais recente primeiro)

**Tags**: blocker, smoke
**Heuristicas**: RCRCRC - Core, EMOTIONS - Alegria, EMOTIONS - Confianca
**Prioridade**: P0

---

## [E2E-TRF-02] - Validar retry com mesma Idempotency-Key apos timeout sem gerar debito duplo

### Pre-condiciones
- Usuario remetente autenticado no sistema (email: sender@test.com)
- Usuario destinatario cadastrado e com conta ativa no sistema (email: recipient@test.com)
- Saldo disponivel do remetente maior que 500 QualiPoints
- Ambiente configurado para simular timeout na primeira tentativa de envio (ex: interceptar resposta da API para forcar timeout de 10s)
- Ambiente completo acessivel via navegador

### Step by step
1. Acessar o aplicativo ja autenticado como "sender@test.com"
2. Verificar e anotar o saldo atual exibido no painel principal (ex: "1000 QualiPoints")
3. Clicar em [Transferir] no menu de navegacao
4. No campo 'Destinatario' informar "recipient@test.com"
5. No campo 'Valor' informar "500"
6. Clicar em [Enviar]
7. Aguardar o timeout da primeira tentativa (sistema exibe mensagem de erro de timeout ou falha de conexao)
8. Verificar que o sistema exibe mensagem orientativa de retry (ex: "Falha na conexao. Tente novamente.")
9. Verificar que os campos 'Destinatario' e 'Valor' preservam os valores previamente informados
10. Clicar em [Enviar] novamente (retry com mesma Idempotency-Key)
11. Aguardar a mensagem de confirmacao ser exibida
12. Verificar o saldo atualizado no painel principal
13. Acessar a tela de transacoes via [Menu] >> [Transacoes]
14. Verificar a quantidade de registros de transferencia para "recipient@test.com" com valor "500"

### Resultado esperado
- Apos o timeout, o sistema exibe mensagem clara e orientativa, sem culpabilizar o usuario (tom: "Tente novamente" e nao "Voce causou um erro")
- Os campos do formulario preservam os valores "recipient@test.com" e "500" apos a falha (Recovery)
- Apos o retry, o sistema exibe mensagem de sucesso
- O saldo do remetente e debitado apenas 1 vez (ex: saldo final = "500 QualiPoints", nao "0")
- Na tela de transacoes, apenas 1 (um) registro de transferencia e exibido para essa operacao (idempotencia garantida)
- Nenhum debito duplo foi realizado mesmo com a tentativa de reenvio

**Tags**: blocker, regression
**Heuristicas**: FAILURE - Recovery, EMOTIONS - Confianca, VADER - Data Consistency (idempotencia)
**Prioridade**: P0

---

## [E2E-TRF-03] - Validar consistencia de saldo e transacoes entre App e Painel Web apos envio de QualiPoints

### Pre-condiciones
- Usuario remetente cadastrado com credenciais validas (email: sender@test.com, senha valida)
- Usuario destinatario cadastrado e com conta ativa no sistema (email: recipient@test.com)
- Saldo disponivel do remetente maior que 200 QualiPoints
- Acesso ao App (mobile ou desktop) e ao Painel Web simultaneamente
- Ambos os canais (App e Painel Web) consomem a mesma API `POST /api/v1/transfers`

### Step by step
1. Abrir o App e autenticar-se como "sender@test.com"
2. Verificar e anotar o saldo atual exibido no App (ex: "1000 QualiPoints")
3. Abrir o Painel Web em outra aba/dispositivo e autenticar-se como "sender@test.com"
4. Verificar que o saldo exibido no Painel Web e identico ao exibido no App
5. No App, clicar em [Transferir]
6. No campo 'Destinatario' informar "recipient@test.com"
7. No campo 'Valor' informar "200"
8. Clicar em [Enviar] >> Aguardar mensagem de confirmacao no App
9. Verificar o saldo atualizado no App (ex: "800 QualiPoints")
10. No Painel Web, clicar em [Atualizar] ou realizar refresh da pagina (F5)
11. Verificar o saldo exibido no Painel Web apos o refresh
12. No Painel Web, acessar [Transacoes] >> Verificar a lista de transacoes recentes

### Resultado esperado
- Apos o envio no App, o saldo do remetente e atualizado imediatamente no App (ex: "800 QualiPoints")
- Apos refresh no Painel Web, o saldo exibido e consistente com o saldo do App (ex: "800 QualiPoints")
- A lista de transacoes no Painel Web exibe o registro da transferencia realizada no App: destinatario "recipient@test.com", valor "200", status "Concluida"
- Nao ha divergencia de saldo ou transacoes entre os dois canais (App e Painel Web)
- A ordenacao da lista de transacoes e por data decrescente em ambos os canais

**Tags**: major, regression
**Heuristicas**: SFDPOT - Sources (multiplos canais), RCRCRC - Core
**Prioridade**: P1

---

## Checklist de Validacao

- [x] Apenas fluxos criticos (3 CTs)?
- [x] Pre-condiciones detalham login, permissoes e estado necessario?
- [x] Step by step usa verbos no infinitivo e colchetes para elementos clicaveis?
- [x] Resultado esperado e mensuravel (passou/falhou)?
- [x] Feedback de loading e sucesso/erro descritos no resultado esperado?
- [x] Recuperacao (corrigir e reenviar) coberta em CT especifico (E2E-TRF-02 - FAILURE Recovery)?
- [x] Cada E2E-XXX do relatorio tem um CT correspondente?
- [x] Tags de criticidade definidas (blocker, major, regression, smoke)?
- [x] Formato segue o [TEMPLATE_CT.MD](../../../docs/templates/TEMPLATE_CT.MD)?

## Referencias

- Relatorio de Estrategia: [test-strategy-qualipoints-envio-20260329.md](../strategy/test-strategy-qualipoints-envio-20260329.md)
- Template CT: [TEMPLATE_CT.MD](../../../docs/templates/TEMPLATE_CT.MD)
- Guide E2E: [TEST_E2E_GUIDE.md](../../../docs/guides/TEST_E2E_GUIDE.md)
- Guide Estrategia: [TEST_STRATEGY.md](../../../docs/guides/TEST_STRATEGY.md)
- FAILURE: [FAILURE.md](../../../skills/heuristic-guide-qa-test/FAILURE.md)
- EMOTIONS: [EMOTIONS.md](../../../skills/heuristic-guide-qa-test/EMOTIONS.md)
