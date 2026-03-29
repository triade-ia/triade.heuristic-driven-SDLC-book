# Relatório BAICA — Envio de QualiPoints entre Usuários

**Requisito analisado:** `input/qualitpoints.md` (Requisito Revisado v2)
**Heurística:** BAICA — Básico, Automação, Interrupção, Criação de Novos Dados, Anônimo
**Modo de entrada:** Épico (Análise de Requisitos)
**Data da análise:** 2026-03-29

---

## Funcionalidades identificadas

| # | Funcionalidade | Canal | Fluxo principal |
|---|---------------|-------|-----------------|
| 1 | Envio de QualiPoints | App mobile | Remetente autenticado envia pontos para destinatário via `POST /api/v1/transfers` |
| 2 | Lista de transações recentes | App mobile / Painel web | Usuário consulta últimas 10 transações próprias |
| 3 | Consulta de saldo | App mobile / Painel web | Usuário visualiza saldo atual da carteira |
| 4 | Retry com idempotência | App mobile | Reenvio da mesma requisição com mesma `Idempotency-Key` após falha/timeout |

---

## 1. Básico (B)

**Realize as ações mais simples e fundamentais da funcionalidade.**

### 1.1 Envio de QualiPoints — Happy Path

| # | Cenário básico | Coberto pelo requisito? | Resultado esperado |
|---|---------------|------------------------|-------------------|
| B-01 | Enviar QualiPoints com dados válidos (amount=100, recipientId válido, token válido, Idempotency-Key única) | Sim (seção 10) | 200 OK com `transactionId`, `amount`, `recipientId`, `status: completed`, `completedAt` |
| B-02 | Saldo do remetente é debitado após envio | Sim (seção 8.1, regra 5) | Saldo reduzido em `amount` |
| B-03 | Saldo do destinatário é creditado após envio | Sim (seção 8.1, regra 5) | Saldo aumentado em `amount` |
| B-04 | Transação aparece na lista de recentes do remetente | Sim (seção 6.3, regra 6) | Transação visível ordenada por data mais recente |
| B-05 | Transação aparece na lista de recentes do destinatário | Sim (seção 6.3, regra 6) | Transação visível ordenada por data mais recente |

### 1.2 Operações CRUD

| # | Operação | Coberto? | Observação |
|---|----------|----------|------------|
| B-06 | **Create** — Criar transação (envio) | Sim | `POST /api/v1/transfers` |
| B-07 | **Read** — Consultar transações recentes | Sim (seção 6.3) | Últimas 10, sem filtros no MVP |
| B-08 | **Read** — Consultar saldo | Implícito | Requisito menciona saldo no App e painel, mas não detalha endpoint |
| B-09 | **Update** — Editar transação | N/A | Transações são imutáveis (correto para este domínio) |
| B-10 | **Delete** — Cancelar/reverter transação | N/A | Não previsto no MVP (correto) |

### 1.3 Validações essenciais e mensagens de erro

| # | Validação | Coberto? | Resposta esperada |
|---|-----------|----------|-------------------|
| B-11 | Valor zero ou negativo | Sim | 400 `INVALID_AMOUNT` |
| B-12 | Valor acima de 10.000 | Sim | 400 `INVALID_AMOUNT` |
| B-13 | Destinatário inexistente ou inativo | Sim | 404 `RECIPIENT_NOT_FOUND` |
| B-14 | Envio para si mesmo | Sim | 409 `SELF_TRANSFER_NOT_ALLOWED` |
| B-15 | Saldo insuficiente | Sim | 422 `INSUFFICIENT_BALANCE` |
| B-16 | Token ausente ou inválido | Sim | 401 `UNAUTHORIZED` |
| B-17 | Conta bloqueada / não ativa | Sim | 403 `ACCOUNT_NOT_ACTIVE` |
| B-18 | Timeout | Sim | 408/504 `REQUEST_TIMEOUT` |

### 1.4 Gaps identificados no Básico

| # | Gap | Risco | Recomendação |
|---|-----|-------|--------------|
| B-G01 | Requisito não define endpoint de consulta de saldo (apenas menciona que App e painel exibem saldo) | Sem endpoint explícito, App e painel não sabem como buscar o saldo | Definir `GET /api/v1/wallet/balance` ou similar no contrato de API |
| B-G02 | Requisito não define mensagem de sucesso no App após envio — apenas os estados (seção 11.1) | UX pode ficar inconsistente entre implementações | Definir mensagem de sucesso: ex. "Envio de {amount} QualiPoints para {destinatário} realizado com sucesso" |
| B-G03 | Lista de transações: requisito não define os campos retornados por transação | Cada implementação pode retornar campos diferentes | Definir contrato: `GET /api/v1/transactions` retornando array com `transactionId`, `amount`, `senderId`, `recipientId`, `status`, `completedAt` |
| B-G04 | Requisito não define comportamento quando `amount` é enviado como decimal (ex.: 10.5) ou string | Pode ser aceito silenciosamente ou gerar erro genérico 500 | Definir: rejeitar com 400 `INVALID_AMOUNT` — "Valor deve ser um número inteiro entre 1 e 10.000" |
| B-G05 | Requisito não define comportamento para body JSON vazio `{}` ou campos ausentes | Erro genérico 500 ao tentar processar campos inexistentes | Validar presença obrigatória de `recipientId` e `amount`; retornar 400 com mensagem clara |

---

## 2. Automação (A)

**Verifique a testabilidade da funcionalidade para futura automação.**

### 2.1 API — Testabilidade para automação

| # | Aspecto | Coberto? | Análise |
|---|---------|----------|---------|
| A-01 | Contrato de API documentado | Sim (seção 10) | Endpoint, headers, body e respostas definidos — facilita automação de testes de API |
| A-02 | Códigos de erro específicos e distintos | Sim (seção 10.4) | Cada cenário tem código HTTP e código de erro próprio (`INVALID_AMOUNT`, `RECIPIENT_NOT_FOUND`, etc.) — excelente para assertions |
| A-03 | Idempotência via header | Sim (seção 8.2) | Permite retry seguro em testes automatizados; facilita setup/teardown |
| A-04 | Estados de transação documentados | Sim (seção 11.1) | Processando, Concluído, Falhou — estados verificáveis |

### 2.2 App mobile — Testabilidade para automação

| # | Aspecto | Coberto? | Análise |
|---|---------|----------|---------|
| A-05 | Identificadores únicos em elementos de UI (IDs, data-testid) | **Não definido** | Requisito não menciona atributos de testabilidade na interface do App |
| A-06 | Indicador de carregamento identificável | **Não definido** | Seção 11.1 define "exibir indicador de carregamento" mas não define como identificá-lo programaticamente |
| A-07 | Mensagens de erro identificáveis por código | Parcial (seção 11.2) | Mensagens mapeadas por código da API — bom para automação se renderizadas com atributo identificável |
| A-08 | Botão de retry identificável | **Não definido** | Requisito menciona "permitir retry" mas não detalha o elemento de UI |

### 2.3 Painel web — Testabilidade para automação

| # | Aspecto | Coberto? | Análise |
|---|---------|----------|---------|
| A-09 | Tabela de transações com elementos identificáveis | **Não definido** | Requisito não menciona atributos de testabilidade no painel |
| A-10 | Saldo exibido com identificador | **Não definido** | Requisito não menciona como o saldo é renderizado |

### 2.4 Gaps identificados na Automação

| # | Gap | Risco | Recomendação |
|---|-----|-------|--------------|
| A-G01 | Requisito não define atributos de testabilidade para elementos do App mobile (botões, campos, listas) | Testes automatizados E2E e de componente terão dificuldade para localizar elementos; seletores frágeis baseados em texto ou posição | Adicionar requisito não-funcional: todo elemento interativo deve ter `data-testid` único e estável (ex.: `data-testid="input-amount"`, `data-testid="input-recipient"`, `data-testid="btn-send"`, `data-testid="loading-indicator"`) |
| A-G02 | Requisito não define atributos de testabilidade para o painel web | Mesmos riscos de A-G01 para automação do painel | Adicionar `data-testid` para tabela de transações, linhas, colunas e exibição de saldo |
| A-G03 | Requisito não define se mensagens de erro no App são renderizadas com identificadores associados ao código de erro | Automação precisará buscar por texto exato da mensagem, que pode mudar | Renderizar mensagens com atributo `data-error-code` correspondente ao código da API (ex.: `data-error-code="INVALID_AMOUNT"`) |
| A-G04 | Requisito não define estados verificáveis no App além de texto (Processando/Concluído/Falhou) | Automação terá dificuldade para verificar transição de estados | Definir atributos de estado no container principal (ex.: `data-state="processing"`, `data-state="completed"`, `data-state="failed"`) |
| A-G05 | Não há endpoint de saúde (health check) definido para o serviço | Testes automatizados não conseguem verificar se o serviço está pronto antes de executar | Definir `GET /api/v1/health` para uso em pipelines de CI/CD e setup de testes |
| A-G06 | Não há mecanismo para reset de estado em ambiente de teste (ex.: limpar transações, redefinir saldo) | Setup e teardown de testes automatizados será manual ou via acesso direto ao banco | Considerar endpoints administrativos ou seeds para ambientes de teste |

---

## 3. Interrupção (I)

**Teste o comportamento da aplicação quando processos são interrompidos.**

### 3.1 Cenários de interrupção cobertos

| # | Cenário | Coberto? | Mecanismo de proteção |
|---|---------|----------|----------------------|
| I-01 | Timeout da API (>10s) | Sim (seção 5.2) | Retorna 504/408; cliente pode retry com mesma `Idempotency-Key` |
| I-02 | Falha de rede durante envio | Sim (seção 12) | Retry com mesma `Idempotency-Key`; API não duplica débito |
| I-03 | Duplo clique / múltiplos envios rápidos | Sim (seção 14) | `Idempotency-Key` evita processamento duplicado |
| I-04 | Falha parcial na transação de banco (débito OK, crédito falha) | Sim (seção 8.1, 6.2) | Operação atômica — rollback completo se qualquer passo falhar |

### 3.2 Gaps identificados na Interrupção

| # | Gap | Risco | Recomendação |
|---|-----|-------|--------------|
| I-G01 | Requisito não define comportamento quando o usuário **fecha o App** durante o estado "Processando" | Transação pode ter sido processada com sucesso no servidor, mas o usuário não viu a confirmação. Ao reabrir, pode tentar enviar novamente (com nova `Idempotency-Key`), causando **débito duplicado** real | Definir: ao reabrir o App, verificar última transação pendente. Se foi enviada mas sem resposta, consultar status no servidor antes de permitir novo envio. Opção: persistir `Idempotency-Key` localmente e fazer retry automático ao reabrir |
| I-G02 | Requisito não define comportamento quando o usuário **navega para outra tela** durante "Processando" | Requisição pode completar em background sem feedback; ou pode ser cancelada pelo App | Definir: requisição deve completar em background; ao retornar à tela, exibir resultado. Não cancelar a requisição HTTP ao sair da tela |
| I-G03 | Requisito não define comportamento para **perda de conexão no meio da resposta** (requisição enviada, resposta parcial recebida) | App não sabe se a transação foi processada ou não | Definir: tratar como timeout; permitir retry com mesma `Idempotency-Key`. Documentar que este cenário é coberto pelo mecanismo de idempotência |
| I-G04 | Requisito não define o que acontece se o **servidor reinicia** durante o processamento de uma transação | Transação pode ficar em estado inconsistente se o restart ocorrer entre débito e crédito | Definir: a atomicidade da transação de banco garante rollback automático em caso de crash. Documentar que o banco é a proteção — não há estado "em progresso" fora da transação de banco |
| I-G05 | Requisito não define comportamento para **cancelamento explícito** pelo usuário (botão cancelar durante "Processando") | Se o usuário cancelar no App, a requisição pode já ter sido processada no servidor | Definir: "Cancelar" no App apenas interrompe a espera da resposta no cliente. A transação no servidor continua. Exibir mensagem: "A operação pode já ter sido processada. Verifique seu saldo antes de tentar novamente." |
| I-G06 | Requisito não define comportamento do **painel web** se atualizado durante um envio em andamento no App | Painel pode exibir saldo desatualizado ou transação ainda não registrada | Definir: comportamento esperado — o painel reflete o estado do banco no momento da consulta. Se a transação ainda não completou, não aparece. Documentar latência esperada |
| I-G07 | Requisito não define limite de **retentativas** com a mesma `Idempotency-Key` | Usuário pode ficar em loop infinito de retry se o erro for persistente | Definir: App limita a N retentativas (ex.: 3) com mesma chave. Após N falhas, exibir mensagem orientando o usuário a tentar mais tarde ou contatar suporte |
| I-G08 | Requisito menciona "desabilitar novo envio até resposta" (seção 11.1) mas não define timeout do lado do cliente | Botão pode ficar desabilitado indefinidamente se a resposta nunca chegar | Definir timeout no cliente (ex.: 15s) após o qual o botão é reabilitado e a mensagem de timeout é exibida |

---

## 4. Criação de Novos Dados (C)

**Execute o fluxo completo criando todos os dados necessários do zero.**

### 4.1 Cadeia de dependências para envio de QualiPoints

Para um usuário completamente novo realizar um envio, a seguinte cadeia é necessária:

```
1. Criar conta de usuário remetente (cadastro)
2. Ativar conta do remetente
3. Remetente fazer login (obter token)
4. Remetente ter saldo na carteira (como?)
5. Criar conta de usuário destinatário
6. Ativar conta do destinatário
7. Remetente realizar envio de QualiPoints
8. Verificar saldo atualizado de ambos
9. Verificar transação na lista de recentes de ambos
```

### 4.2 Gaps identificados na Criação de Novos Dados

| # | Gap | Risco | Recomendação |
|---|-----|-------|--------------|
| C-G01 | **Requisito não define como um usuário obtém QualiPoints inicialmente** — não há fluxo de carga de saldo, compra de pontos ou saldo inicial | Fluxo end-to-end impossível para usuário novo: não há como ter saldo para enviar. Testes de fluxo completo dependem de dados pré-existentes (seed de saldo) | Definir no requisito: qual é a origem dos QualiPoints? Opções: saldo inicial no cadastro, compra via pagamento, crédito administrativo. Sem isso, o fluxo de envio é testável apenas com saldo pré-carregado |
| C-G02 | **Requisito não define o fluxo de cadastro/ativação de conta** que precede o envio | Testes end-to-end do envio dependem de uma funcionalidade não especificada neste escopo | Referenciar o requisito de cadastro de usuário. Se não existir, documentar como dependência bloqueante |
| C-G03 | **Requisito não define como obter o `recipientId`** de outro usuário no App | Para enviar, o remetente precisa saber o ID do destinatário. Não há busca de usuários, lista de contatos ou compartilhamento de ID definido | Definir mecanismo: busca por nome/email/telefone, QR code, lista de contatos, ou digitação manual do ID. Sem isso, o fluxo real é incompleto |
| C-G04 | Requisito não define o estado inicial da **carteira** ao criar uma conta | Novo usuário pode não ter registro de carteira; envio para ele pode falhar com erro inesperado | Definir: ao criar conta, carteira é criada automaticamente com saldo 0. Ou: carteira é criada no primeiro recebimento |
| C-G05 | Requisito não define comportamento do **primeiro acesso** ao painel web | Painel exibe últimas 10 transações — mas para um usuário novo, a lista está vazia | Definir: exibir estado vazio amigável (ex.: "Nenhuma transação realizada ainda") em vez de lista vazia ou erro |
| C-G06 | Requisito não define se o destinatário **precisa ter feito login ao menos uma vez** para poder receber | Se a verificação é apenas "conta ativa", um usuário cadastrado que nunca logou pode receber pontos sem saber | Definir: é válido enviar para quem nunca logou? Se sim, como o destinatário é notificado? (notificações estão no backlog) |
| C-G07 | Requisito não define como o **App persiste o token** entre sessões para um novo usuário | Se o token expira e o usuário precisa re-logar, o fluxo de envio é interrompido | Referenciar requisito de autenticação: refresh token, duração da sessão, comportamento de re-login |

---

## 5. Anônimo (A)

**Valide o comportamento da aplicação em navegação privada/anônima.**

### 5.1 Painel web em modo anônimo

| # | Cenário | Coberto? | Análise |
|---|---------|----------|---------|
| AN-01 | Login no painel web em modo anônimo do navegador | **Não definido** | Requisito não menciona comportamento do painel em modo privado |
| AN-02 | Consulta de saldo e transações em modo anônimo | **Não definido** | Depende de como o token de autenticação é armazenado no painel |
| AN-03 | Sessão do painel web sem cookies persistentes | **Não definido** | Se o painel usa cookies para sessão, modo anônimo pode afetar |

### 5.2 App mobile — Contexto de privacidade

| # | Cenário | Coberto? | Análise |
|---|---------|----------|---------|
| AN-04 | App em dispositivo com configurações de privacidade restritivas | **Não definido** | Bloqueio de tracking, permissões reduzidas podem afetar funcionalidade |
| AN-05 | App após limpar dados/cache do aplicativo | **Não definido** | Similar a modo anônimo — token e dados locais são removidos |

### 5.3 Gaps identificados no Anônimo

| # | Gap | Risco | Recomendação |
|---|-----|-------|--------------|
| AN-G01 | **Requisito não define como o painel web armazena a sessão/token** (cookie, localStorage, sessionStorage) | Em modo anônimo, `localStorage` é limpo ao fechar o navegador. Se o painel depende de `localStorage` para o token, o usuário perde a sessão ao fechar a aba | Definir mecanismo de armazenamento: se `sessionStorage`, funciona em modo anônimo durante a aba. Se `localStorage`, perdido ao fechar modo anônimo. Se cookie de sessão, funciona normalmente em modo anônimo (dentro da aba) |
| AN-G02 | **Requisito não define se o painel web funciona corretamente sem cookies de terceiros** | Navegadores modernos bloqueiam cookies de terceiros. Se o painel ou a API de autenticação depende deles, pode falhar em modo anônimo | Garantir que autenticação funcione com cookies first-party ou token via header (já indicado como Bearer no header) |
| AN-G03 | **Requisito não define comportamento após fechar e reabrir sessão anônima no painel** | Usuário perde todo o contexto. Se estava consultando transações, precisa fazer login novamente | Definir: comportamento esperado — redirecionar para tela de login. Não exibir dados residuais de sessão anterior |
| AN-G04 | **Requisito não menciona se dados sensíveis (saldo, transações, IDs) ficam em cache do navegador** | Em modo normal, cache pode persistir dados sensíveis no disco. Em modo anônimo, é limpo ao fechar — mas durante a sessão, outro script ou extensão pode acessar | Definir headers de cache: `Cache-Control: no-store` para respostas com dados financeiros. `Pragma: no-cache` para compatibilidade |
| AN-G05 | **Requisito não define comportamento do App mobile após "limpar dados" do aplicativo** | Equivalente a modo anônimo em mobile — token, preferências e cache são removidos | Definir: App deve redirecionar para tela de login. Não exibir erros de "dados corrompidos" |
| AN-G06 | **Requisito não define se a `Idempotency-Key` persiste localmente no App** | Se o App armazena a chave localmente para retry e o usuário limpa os dados, a chave é perdida. Um novo envio com nova chave pode causar débito duplicado | Definir: o mecanismo de idempotência deve ser resiliente à perda de dados locais. Se a chave foi perdida, o usuário deve verificar saldo/transações antes de reenviar |
| AN-G07 | Requisito não define se o painel web depende de JavaScript habilitado | Alguns modos de privacidade ou navegadores desabilitam JS. Se o painel é SPA, não funciona | Definir: JS é obrigatório. Exibir mensagem amigável se desabilitado: "Este painel requer JavaScript habilitado" |

---

## Checklist de Análise BAICA

- [x] **Básico**: Happy path bem definido; CRUD parcial (falta endpoint de saldo); validações e mensagens de erro completas; **gaps em**: endpoint de saldo, formato decimal do amount, body vazio, contrato de listagem de transações
- [x] **Automação**: API com contrato e códigos de erro distintos (boa base); **gaps em**: `data-testid` no App e painel, estados verificáveis na UI, health check, mecanismo de reset para testes
- [x] **Interrupção**: Timeout e retry via idempotência bem definidos; atomicidade garante integridade; **gaps em**: fechamento do App durante processamento, navegação entre telas, cancelamento explícito, limite de retentativas, timeout do cliente
- [x] **Criação de Novos Dados**: Fluxo de envio definido, mas dependências não cobertas; **gaps em**: origem dos QualiPoints (saldo inicial), mecanismo de descoberta do destinatário, estado inicial da carteira, primeiro acesso ao painel
- [x] **Anônimo**: Não abordado pelo requisito; **gaps em**: armazenamento de sessão no painel, cache de dados sensíveis, comportamento após limpar dados do App, resiliência da idempotência à perda de dados locais

---

## Resumo de Gaps por Criticidade

### Críticos (implementar antes do MVP)

| ID | Pilar | Gap |
|----|-------|-----|
| C-G01 | Criação de Novos Dados | Origem dos QualiPoints não definida — fluxo end-to-end impossível para novo usuário |
| I-G01 | Interrupção | Fechamento do App durante "Processando" pode causar débito duplicado real |
| C-G03 | Criação de Novos Dados | Mecanismo de descoberta do `recipientId` não definido — fluxo real incompleto |

### Altos (implementar no MVP)

| ID | Pilar | Gap |
|----|-------|-----|
| B-G01 | Básico | Endpoint de consulta de saldo não definido |
| B-G03 | Básico | Contrato da API de listagem de transações não definido |
| B-G04 | Básico | Comportamento para amount decimal/string não definido |
| I-G05 | Interrupção | Cancelamento explícito pelo usuário durante "Processando" sem definição |
| I-G07 | Interrupção | Sem limite de retentativas — risco de loop infinito |
| I-G08 | Interrupção | Sem timeout do lado do cliente — botão pode ficar desabilitado indefinidamente |
| C-G04 | Criação de Novos Dados | Estado inicial da carteira ao criar conta não definido |
| AN-G06 | Anônimo | Idempotency-Key perdida ao limpar dados do App pode causar débito duplicado |

### Médios (recomendado para MVP)

| ID | Pilar | Gap |
|----|-------|-----|
| B-G02 | Básico | Mensagem de sucesso no App não padronizada |
| B-G05 | Básico | Body vazio ou campos ausentes sem tratamento definido |
| A-G01 | Automação | Elementos do App sem `data-testid` |
| A-G02 | Automação | Elementos do painel web sem `data-testid` |
| A-G03 | Automação | Mensagens de erro sem identificador de código no DOM |
| A-G04 | Automação | Estados da UI sem atributos verificáveis |
| I-G02 | Interrupção | Navegação entre telas durante processamento sem definição |
| C-G05 | Criação de Novos Dados | Estado vazio no painel (primeiro acesso) sem definição |
| AN-G01 | Anônimo | Mecanismo de armazenamento de sessão no painel não definido |
| AN-G04 | Anônimo | Headers de cache para dados sensíveis não definidos |

### Baixos (backlog)

| ID | Pilar | Gap |
|----|-------|-----|
| A-G05 | Automação | Health check endpoint não definido |
| A-G06 | Automação | Mecanismo de reset de estado para testes não definido |
| I-G04 | Interrupção | Comportamento em restart do servidor (coberto pela atomicidade do banco) |
| I-G06 | Interrupção | Consistência painel/App durante envio em andamento |
| C-G02 | Criação de Novos Dados | Fluxo de cadastro/ativação como dependência |
| C-G06 | Criação de Novos Dados | Destinatário que nunca logou pode receber pontos |
| C-G07 | Criação de Novos Dados | Persistência de token entre sessões |
| AN-G02 | Anônimo | Cookies de terceiros no painel |
| AN-G03 | Anônimo | Reabrir sessão anônima no painel |
| AN-G05 | Anônimo | Limpar dados do App mobile |
| AN-G07 | Anônimo | Dependência de JavaScript no painel |

---

## Exemplo de Aplicação Completa — Envio de QualiPoints

### B - Básico
- Enviar 100 QualiPoints com dados válidos → 200 OK
- Verificar saldo debitado do remetente
- Verificar saldo creditado do destinatário
- Visualizar transação na lista de recentes (remetente e destinatário)
- Enviar com valor 0 → 400 `INVALID_AMOUNT`
- Enviar para si mesmo → 409 `SELF_TRANSFER_NOT_ALLOWED`
- Enviar com saldo insuficiente → 422 `INSUFFICIENT_BALANCE`

### A - Automação
- Campo de valor (`input-amount`) tem `data-testid` único
- Campo de destinatário (`input-recipient`) tem `data-testid` único
- Botão enviar (`btn-send`) tem `data-testid` único e estável
- Indicador de carregamento tem `data-state="processing"`
- Mensagens de erro têm `data-error-code` correspondente ao código da API
- Tabela de transações no painel tem linhas identificáveis

### I - Interrupção
- Fechar App durante "Processando" → ao reabrir, verificar status da última transação
- Perda de conexão durante envio → retry com mesma `Idempotency-Key`
- Duplo clique no botão enviar → apenas uma transação processada
- Cancelar durante processamento → mensagem informando que transação pode já ter sido processada
- Timeout do cliente (15s) → reabilitar botão e exibir mensagem

### C - Criação de Novos Dados
- Cadastrar novo usuário remetente → ativar conta → login
- Carregar saldo inicial na carteira (mecanismo a definir)
- Cadastrar novo usuário destinatário → ativar conta
- Remetente localiza destinatário (mecanismo a definir)
- Realizar envio de QualiPoints
- Verificar saldo e transações de ambos
- Acessar painel web e verificar consistência

### A - Anônimo
- Abrir painel web em modo anônimo → fazer login → consultar saldo e transações
- Fechar aba anônima → reabrir → verificar que sessão foi perdida (redirecionado ao login)
- Limpar dados do App → verificar redirecionamento para login
- Verificar que respostas da API possuem `Cache-Control: no-store`
- Verificar que dados sensíveis não persistem após encerrar sessão anônima

---

## Próximos Passos

1. **Refinamento do requisito**: Incorporar os gaps críticos e altos antes de iniciar a codificação — especialmente a origem dos QualiPoints, mecanismo de descoberta do destinatário e comportamento de interrupção no App
2. **Testabilidade**: Adicionar requisitos não-funcionais de `data-testid` e estados verificáveis para viabilizar automação
3. **Testes**: Usar este relatório como base para gerar casos de teste por camada — consultar `TEST_STRATEGY` com este relatório como contexto

---

**Referência:** Faria, Jonatas Martins. Heurística BAICA — Garantindo testes fundamentais em funcionalidades.
