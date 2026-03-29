---
name: test-e2e-guide
description: Guia para escrever Casos de Teste E2E no formato CT (Caso de Teste). Use quando tiver relatório de estratégia (output/test-strategy-*.md) com casos E2E-XXX e precisar gerar casos de teste manuais de fluxo completo (UI → API → banco).
---

# Guia de Testes E2E (End-to-End)

Escreva **Casos de Teste (CT)** E2E com base no **relatório de estratégia** gerado por [TEST_STRATEGY.md](TEST_STRATEGY.md). Use os casos com prefixo **E2E-XXX** para gerar CTs de fluxo completo seguindo o padrão do [TEMPLATE_CT.MD](../templates/TEMPLATE_CT.MD).

## Quando Usar Este Guide

- Relatório contém casos E2E (fluxos críticos de negócio)
- Precisa testar: login → ação → resultado, mensagens de sucesso/erro, recuperação, duplo envio
- Heurísticas: FAILURE (Emotions, Recovery), EMOTIONS

## Definição e Escopo

Testes E2E validam **fluxos completos** em ambiente real (navegador, backend, banco).

**Características**: Fluxo ponta a ponta, focar em jornadas críticas (poucos testes).

**Quando usar**: Login, compra, pagamento, transferência — fluxos que precisam estar sempre estáveis.

**Quando NÃO usar**: Casos de borda (unitários/integração); validações de input (componentes).

## Como Usar o Relatório

1. Receba o relatório: `@TEST_E2E_GUIDE.md @output/test-strategy-xxx.md`
2. Localize "Testes E2E" e casos E2E-XXX
3. Para cada caso E2E-XXX, escreva um CT seguindo o [TEMPLATE_CT.MD](../templates/TEMPLATE_CT.MD)
4. Cada CT deve conter: **Título** (iniciar com "Validar"), **Pré-condições**, **Step by step** e **Resultado esperado**

## Formato Obrigatório

Todos os casos de teste E2E devem seguir o padrão do [TEMPLATE_CT.MD](../templates/TEMPLATE_CT.MD):

- **Título**: Sempre iniciar com "Validar" + ação + comportamento esperado
- **Pré-condições**: Detalhar permissões, cadastros, configurações e estado necessário
- **Step by step**: Passos numerados, verbos no infinitivo, colchetes `[]` para elementos clicáveis
- **Resultado esperado**: Estado final esperado, onde validar, mensurável (passou/falhou)

## Regras de Escrita

- Usar colchetes `[]` para botões, menus e opções clicáveis. Ex.: `Clicar em [Transferir] >> [Confirmar]`
- Usar aspas para nomes de campos e labels. Ex.: no campo 'Destinatário' informar "recipient@test.com"
- Verbos no infinitivo: `Abrir`, `Clicar`, `Selecionar`, `Confirmar`, `Aguardar`, `Verificar`
- Evitar termos técnicos de código — focar na ação visível do usuário
- Descrever valores concretos usados no teste
- Cada passo deve ser reproduzível por outro membro do time

## Exemplos: Fluxo de Transferência

### [E2E-001] - Validar transferência com sucesso no fluxo completo

**Pré-condições**
- Usuário remetente com credenciais válidas (email: sender@test.com, senha válida)
- Usuário destinatário cadastrado no sistema (email: recipient@test.com)
- Saldo disponível do remetente maior que R$ 500,00
- Ambiente acessível via navegador

**Step by step**
1. Abrir o aplicativo e acessar a tela de login
2. No campo 'Email' informar "sender@test.com" >> No campo 'Senha' informar a senha válida
3. Clicar em [Entrar] >> Aguardar redirecionamento para o painel principal
4. Clicar em [Transferir] no menu de navegação
5. No campo 'Destinatário' informar "recipient@test.com"
6. No campo 'Valor' informar "500"
7. Verificar que o botão [Enviar] está habilitado
8. Clicar em [Enviar]
9. Aguardar a mensagem de processamento ser exibida
10. Aguardar a mensagem de confirmação ser exibida

**Resultado esperado**
- O sistema exibe a mensagem indicando "Processando" durante o envio
- O sistema exibe a mensagem de sucesso após conclusão
- O sistema redireciona para a tela de transações
- Na lista de transações, o primeiro registro exibe: destinatário "recipient@test.com", valor "500" e status "Concluída"

**Tags**: blocker, smoke

---

### [E2E-002] - Validar exibição de erro e recuperação ao informar valor inválido na transferência

**Pré-condições**
- Usuário remetente autenticado no sistema (email: sender@test.com)
- Usuário destinatário cadastrado no sistema (email: recipient@test.com)

**Step by step**
1. Acessar a tela de transferência via [Menu] >> [Transferir]
2. No campo 'Destinatário' informar "recipient@test.com"
3. No campo 'Valor' informar "0"
4. Clicar em [Enviar]
5. Verificar a mensagem de erro exibida
6. Verificar que o campo 'Destinatário' mantém o valor previamente informado
7. Corrigir o campo 'Valor' informando "500"
8. Clicar em [Enviar]
9. Aguardar a mensagem de confirmação ser exibida

**Resultado esperado**
- O sistema exibe mensagem de erro indicando que o valor deve ser maior que 0
- A mensagem de erro não culpabiliza o usuário (tom claro e orientativo)
- O campo 'Destinatário' preserva o valor "recipient@test.com" após o erro (Recovery)
- Após correção e reenvio, o sistema exibe mensagem de sucesso

**Tags**: major, regression
**Heurísticas**: FAILURE - Recovery, FAILURE - Emotions

---

### [E2E-003] - Validar prevenção de duplo envio durante processamento de transferência

**Pré-condições**
- Usuário remetente autenticado no sistema (email: sender@test.com)
- Usuário destinatário cadastrado no sistema (email: recipient@test.com)
- Saldo disponível do remetente maior que R$ 500,00

**Step by step**
1. Acessar a tela de transferência via [Menu] >> [Transferir]
2. No campo 'Destinatário' informar "recipient@test.com"
3. No campo 'Valor' informar "500"
4. Clicar em [Enviar]
5. Imediatamente clicar em [Enviar] novamente (tentativa de duplo envio)
6. Verificar o estado do botão [Enviar] durante o processamento
7. Aguardar a mensagem de confirmação ser exibida
8. Acessar a tela de transações via [Menu] >> [Transações]

**Resultado esperado**
- O botão [Enviar] é desabilitado imediatamente após o primeiro clique (prevenção de duplo envio)
- O segundo clique não gera uma nova transação
- O sistema exibe mensagem de sucesso referente a uma única transação
- Na tela de transações, apenas 1 (um) registro de transferência é exibido

**Tags**: blocker, regression

## Integração com Heurísticas

- **FAILURE - Emotions**: Mensagens não culpabilizam usuário; tom claro e orientativo
- **FAILURE - Recovery**: Campos preservados após erro; usuário pode corrigir e reenviar
- **EMOTIONS**: Análise do impacto emocional das mensagens e feedbacks exibidos ao usuário

## Checklist

- [ ] Apenas fluxos críticos (poucos CTs)?
- [ ] Pré-condições detalham login, permissões e estado necessário?
- [ ] Step by step usa verbos no infinitivo e colchetes para elementos clicáveis?
- [ ] Resultado esperado é mensurável (passou/falhou)?
- [ ] Feedback de loading e sucesso/erro descritos no resultado esperado?
- [ ] Recuperação (corrigir e reenviar) coberta em CT específico (FAILURE - Recovery)?
- [ ] Cada E2E-XXX do relatório tem um CT correspondente?
- [ ] Tags de criticidade definidas (blocker, major, regression, smoke)?
- [ ] Formato segue o [TEMPLATE_CT.MD](../templates/TEMPLATE_CT.MD)?

## Referências

- Template CT: [TEMPLATE_CT.MD](../templates/TEMPLATE_CT.MD)
- Estratégia: [TEST_STRATEGY.md](TEST_STRATEGY.md)
- Componentes: [TEST_COMPONENT_GUIDE.md](TEST_COMPONENT_GUIDE.md)
- Serviço: [TEST_SERVICE_GUIDE.md](TEST_SERVICE_GUIDE.md)
- FAILURE: [FAILURE.md](../../skills/heuristic-guide-qa-test/FAILURE.md)
