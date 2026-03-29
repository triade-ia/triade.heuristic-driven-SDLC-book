# Casos de Teste E2E: Envio de QualiPoints

**Data**: 2026-03-29
**Relatório base**: [test-strategy-qualipoints-envio-20260329.md](../strategy/test-strategy-qualipoints-envio-20260329.md)
**Guide**: [TEST_E2E_GUIDE.md](../../../docs/guides/TEST_E2E_GUIDE.md)
**Template**: [TEMPLATE_CT.MD](../../../docs/templates/TEMPLATE_CT.MD)

---

## [E2E-TRF-01] - Validar envio de QualiPoints com sucesso no fluxo completo (login, envio, confirmação, saldo e lista)

### Pré-condições
- Usuário remetente cadastrado com credenciais válidas (email: sender@test.com, senha válida)
- Usuário destinatário cadastrado e com conta ativa no sistema (email: recipient@test.com)
- Saldo disponível do remetente maior que 500 QualiPoints
- Ambiente completo acessível via navegador (App ou Painel Web)
- Serviço de autenticação e API `POST /api/v1/transfers` disponíveis

### Step by step
1. Abrir o aplicativo e acessar a tela de login
2. No campo 'Email' informar "sender@test.com" >> No campo 'Senha' informar a senha válida
3. Clicar em [Entrar] >> Aguardar redirecionamento para o painel principal
4. Verificar o saldo exibido no painel principal e anotar o valor atual (ex: "1000 QualiPoints")
5. Clicar em [Transferir] no menu de navegação
6. No campo 'Destinatário' informar "recipient@test.com"
7. No campo 'Valor' informar "500"
8. Verificar que o botão [Enviar] está habilitado
9. Clicar em [Enviar]
10. Aguardar a mensagem de processamento ser exibida (loading/spinner)
11. Aguardar a mensagem de confirmação ser exibida
12. Verificar o saldo atualizado no painel principal
13. Acessar a tela de transações via [Menu] >> [Transações]
14. Verificar o primeiro registro na lista de transações recentes

### Resultado esperado
- O sistema exibe indicador de carregamento (spinner) durante o processamento da transferência
- O sistema exibe mensagem de sucesso após a conclusão (ex: "Transferência realizada com sucesso")
- O saldo do remetente é atualizado e exibe o valor anterior menos 500 QualiPoints (ex: "500 QualiPoints")
- Na tela de transações, o primeiro registro exibe: destinatário "recipient@test.com", valor "500", status "Concluída" e data/hora da transação
- A lista de transações recentes está ordenada por data decrescente (mais recente primeiro)

**Tags**: blocker, smoke
**Heurísticas**: RCRCRC - Core, EMOTIONS - Alegria, EMOTIONS - Confiança
**Prioridade**: P0

---

## [E2E-TRF-02] - Validar retry com mesma Idempotency-Key após timeout sem gerar débito duplo

### Pré-condições
- Usuário remetente autenticado no sistema (email: sender@test.com)
- Usuário destinatário cadastrado e com conta ativa no sistema (email: recipient@test.com)
- Saldo disponível do remetente maior que 500 QualiPoints
- Ambiente configurado para simular timeout na primeira tentativa de envio (ex: interceptar resposta da API para forçar timeout de 10s)
- Ambiente completo acessível via navegador

### Step by step
1. Acessar o aplicativo já autenticado como "sender@test.com"
2. Verificar e anotar o saldo atual exibido no painel principal (ex: "1000 QualiPoints")
3. Clicar em [Transferir] no menu de navegação
4. No campo 'Destinatário' informar "recipient@test.com"
5. No campo 'Valor' informar "500"
6. Clicar em [Enviar]
7. Aguardar o timeout da primeira tentativa (sistema exibe mensagem de erro de timeout ou falha de conexão)
8. Verificar que o sistema exibe mensagem orientativa de retry (ex: "Falha na conexão. Tente novamente.")
9. Verificar que os campos 'Destinatário' e 'Valor' preservam os valores previamente informados
10. Clicar em [Enviar] novamente (retry com mesma Idempotency-Key)
11. Aguardar a mensagem de confirmação ser exibida
12. Verificar o saldo atualizado no painel principal
13. Acessar a tela de transações via [Menu] >> [Transações]
14. Verificar a quantidade de registros de transferência para "recipient@test.com" com valor "500"

### Resultado esperado
- Após o timeout, o sistema exibe mensagem clara e orientativa, sem culpabilizar o usuário (tom: "Tente novamente" e não "Você causou um erro")
- Os campos do formulário preservam os valores "recipient@test.com" e "500" após a falha (Recovery)
- Após o retry, o sistema exibe mensagem de sucesso
- O saldo do remetente é debitado apenas 1 vez (ex: saldo final = "500 QualiPoints", não "0")
- Na tela de transações, apenas 1 (um) registro de transferência é exibido para essa operação (idempotência garantida)
- Nenhum débito duplo foi realizado mesmo com a tentativa de reenvio

**Tags**: blocker, regression
**Heurísticas**: FAILURE - Recovery, EMOTIONS - Confiança, VADER - Data Consistency (idempotência)
**Prioridade**: P0

---

## [E2E-TRF-03] - Validar consistência de saldo e transações entre App e Painel Web após envio de QualiPoints

### Pré-condições
- Usuário remetente cadastrado com credenciais válidas (email: sender@test.com, senha válida)
- Usuário destinatário cadastrado e com conta ativa no sistema (email: recipient@test.com)
- Saldo disponível do remetente maior que 200 QualiPoints
- Acesso ao App (mobile ou desktop) e ao Painel Web simultaneamente
- Ambos os canais (App e Painel Web) consomem a mesma API `POST /api/v1/transfers`

### Step by step
1. Abrir o App e autenticar-se como "sender@test.com"
2. Verificar e anotar o saldo atual exibido no App (ex: "1000 QualiPoints")
3. Abrir o Painel Web em outra aba/dispositivo e autenticar-se como "sender@test.com"
4. Verificar que o saldo exibido no Painel Web é idêntico ao exibido no App
5. No App, clicar em [Transferir]
6. No campo 'Destinatário' informar "recipient@test.com"
7. No campo 'Valor' informar "200"
8. Clicar em [Enviar] >> Aguardar mensagem de confirmação no App
9. Verificar o saldo atualizado no App (ex: "800 QualiPoints")
10. No Painel Web, clicar em [Atualizar] ou realizar refresh da página (F5)
11. Verificar o saldo exibido no Painel Web após o refresh
12. No Painel Web, acessar [Transações] >> Verificar a lista de transações recentes

### Resultado esperado
- Após o envio no App, o saldo do remetente é atualizado imediatamente no App (ex: "800 QualiPoints")
- Após refresh no Painel Web, o saldo exibido é consistente com o saldo do App (ex: "800 QualiPoints")
- A lista de transações no Painel Web exibe o registro da transferência realizada no App: destinatário "recipient@test.com", valor "200", status "Concluída"
- Não há divergência de saldo ou transações entre os dois canais (App e Painel Web)
- A ordenação da lista de transações é por data decrescente em ambos os canais

**Tags**: major, regression
**Heurísticas**: SFDPOT - Sources (múltiplos canais), RCRCRC - Core
**Prioridade**: P1

---

## Checklist de Validação

- [x] Apenas fluxos críticos (3 CTs)?
- [x] Pré-condições detalham login, permissões e estado necessário?
- [x] Step by step usa verbos no infinitivo e colchetes para elementos clicáveis?
- [x] Resultado esperado é mensurável (passou/falhou)?
- [x] Feedback de loading e sucesso/erro descritos no resultado esperado?
- [x] Recuperação (corrigir e reenviar) coberta em CT específico (E2E-TRF-02 - FAILURE Recovery)?
- [x] Cada E2E-XXX do relatório tem um CT correspondente?
- [x] Tags de criticidade definidas (blocker, major, regression, smoke)?
- [x] Formato segue o [TEMPLATE_CT.MD](../../../docs/templates/TEMPLATE_CT.MD)?

## Referências

- Relatório de Estratégia: [test-strategy-qualipoints-envio-20260329.md](../strategy/test-strategy-qualipoints-envio-20260329.md)
- Template CT: [TEMPLATE_CT.MD](../../../docs/templates/TEMPLATE_CT.MD)
- Guide E2E: [TEST_E2E_GUIDE.md](../../../docs/guides/TEST_E2E_GUIDE.md)
- Guide Estratégia: [TEST_STRATEGY.md](../../../docs/guides/TEST_STRATEGY.md)
- FAILURE: [FAILURE.md](../../../skills/heuristic-guide-qa-test/FAILURE.md)
- EMOTIONS: [EMOTIONS.md](../../../skills/heuristic-guide-qa-test/EMOTIONS.md)
