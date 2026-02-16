# 🔍 Análise Heurística WWWWWHKE - Requisito: Envio de QualiPoints

## Requisito Analisado
**Implementar funcionalidade de envio de "QualiPoints" entre usuários.**

O usuário quer poder mandar pontos para um amigo. O sistema deve ter uma tela para isso, uma API que processa e funcionar no App. Tem que ser rápido e seguro.

### Contexto do Projeto: Sistema de Transferência "QualiPoints"

O sistema deve permitir que usuários enviem pontos entre si através de uma carteira digital. A operação deve ser feita via aplicativo mobile e confirmada via painel web. Os dados residem em um banco de dados relacional e a comunicação é via REST API.

### Regras de Negócio Iniciais (Lacunares - propositalmente incompletas)

- O usuário deve estar logado para enviar.
- É necessário informar o ID do destinatário e o valor.
- O saldo deve ser atualizado após o envio.
- Deve haver uma lista de transações recentes.

---

## 🔍 Análise Heurística WWWWWHKE - Parecer Técnico de Viabilidade Imediata

### 1. Questionamentos Críticos (O Filtro)

#### Who (Quem) - [Contexto: Autorização e Identidade]

**Pergunta crítica não respondida**: 
- Quais perfis de usuário podem enviar QualiPoints? Todos os usuários logados ou apenas contas verificadas/premium?
- Existe limite de idade ou status de conta (ativa, suspensa, em análise) que impede o envio?
- Quem autoriza transações em caso de falha de sistema ou disputa? Existe um processo de reversão manual?
- O destinatário precisa estar ativo/logado para receber, ou pode receber pontos mesmo offline?
- Existe diferenciação de permissões entre usuários comuns e administradores para visualizar/auditar transações?

#### What (O quê) - [Contexto: Objeto e Limites]

**Pergunta crítica não respondida**:
- Qual o valor mínimo e máximo de QualiPoints que pode ser enviado em uma única transação?
- Existe limite diário/semanal/mensal por usuário para prevenir fraude ou abuso?
- Qual o formato e precisão do valor (inteiro, decimal com quantas casas)?
- O que acontece se o usuário tentar enviar mais pontos do que possui? A transação é rejeitada ou entra em débito?
- Qual a estrutura de dados completa de uma transação? (ID, timestamp, remetente, destinatário, valor, status, motivo, etc.)

#### When (Quando) - [Contexto: Temporalidade e Concorrência]

**Pergunta crítica não respondida**:
- A operação é síncrona (aguarda confirmação imediata) ou assíncrona (processa em background)?
- O que acontece se duas transações simultâneas tentarem debitar o mesmo saldo? Como garantir consistência?
- Existe expiração ou janela de tempo para confirmar uma transação pendente?
- Se a API de destino demorar mais de X segundos, o sistema cancela automaticamente ou tenta novamente?
- Qual o comportamento em caso de timeout de rede durante o envio?
- A confirmação via painel web é obrigatória ou apenas visualização? Se obrigatória, qual o prazo?

#### Where (Onde) - [Contexto: Arquitetura e Localização]

**Pergunta crítica não respondida**:
- Onde a validação de saldo ocorre: apenas no servidor (API) ou também no cliente (App)? A API garante a verdade única?
- Onde os dados são persistidos? Qual tabela/coleção do banco de dados?
- Onde pode ocorrer inconsistência? Como garantir atomicidade da operação (debitar remetente + creditar destinatário)?
- A lista de transações recentes é consultada do mesmo banco ou há cache/read replica?
- Onde fica a lógica de negócio: na API, em stored procedures, ou em serviços separados?

#### Why (Por que) - [Contexto: Valor vs. Complexidade]

**Pergunta crítica não respondida**:
- A confirmação via painel web é essencial para o MVP ou pode ser apenas visualização de histórico?
- Precisamos de comprovantes em PDF/email agora ou um log de transação basta para o MVP?
- A lista de transações recentes precisa ser em tempo real ou pode ter delay de alguns minutos?
- O "rápido" significa <1s de resposta ou <3s? Isso impacta a arquitetura (cache, async, etc.)?
- Por que "seguro" - há risco de fraude financeira real ou é apenas pontos virtuais sem valor monetário?

#### How (Como) - [Contexto: Transição de Estado e Segurança]

**Pergunta crítica não respondida**:
- Como garantimos atomicidade: se o crédito no destinatário falhar após debitar o remetente, há rollback automático?
- Qual protocolo de autenticação/autorização protege a API? (JWT, OAuth, API Key?)
- Como validamos que o destinatário existe e está ativo antes de processar?
- Como prevenimos duplicação de transações (idempotência)? Token único por requisição?
- Como o sistema sai do estado "pendente" para "confirmado" ou "falhou"? Máquina de estados?
- Há criptografia de dados sensíveis em trânsito (HTTPS) e em repouso?

#### Keep (Manter) - [Contexto: Essencialidade]

**O que é inegociável para segurança e funcionamento básico**:
- ✅ Autenticação obrigatória antes de qualquer transação
- ✅ Validação de saldo suficiente no servidor (nunca confiar no cliente)
- ✅ Transação atômica (debitar + creditar ou nada)
- ✅ Log/auditoria de todas as transações para rastreabilidade
- ✅ Validação de destinatário existente e ativo
- ✅ Tratamento de erros básico (saldo insuficiente, destinatário inválido, falha de rede)

#### Eliminate (Eliminar) - [Contexto: Redução de Ruído]

**O que é "perfumaria" ou suposição que pode ser removida para acelerar a entrega**:
- ❌ Confirmação obrigatória via painel web (pode ser apenas visualização para MVP)
- ❌ Geração de comprovantes em PDF ou envio de emails
- ❌ Notificações push em tempo real (pode ser polling ou webhook simples)
- ❌ Histórico completo com filtros avançados (apenas lista recente básica)
- ❌ Validações complexas de fraude (se não há valor monetário real)
- ❌ Interface de busca avançada de destinatários (apenas busca por ID ou nome simples)

### 2. Gaps de Implementação (O Risco)

**O que falta tecnicamente para que o código seja escrito sem impedimentos**:

1. **Contrato de API não definido**:
   - Endpoints exatos (POST /transactions, GET /transactions, etc.)
   - Estrutura de request/response (JSON schema)
   - Códigos de erro padronizados e mensagens de erro mapeadas

2. **Modelo de dados incompleto**:
   - Schema da tabela de transações não especificado
   - Relacionamentos entre tabelas (users, transactions, wallets)
   - Índices necessários para performance

3. **Regras de negócio não especificadas**:
   - Limites de valor (mínimo, máximo, diário)
   - Comportamento em caso de saldo insuficiente
   - Política de retry em caso de falha

4. **Arquitetura não definida**:
   - Estratégia de transações (síncrona vs assíncrona)
   - Estratégia de cache (se houver)
   - Estratégia de idempotência

5. **Segurança não especificada**:
   - Método de autenticação/autorização
   - Rate limiting necessário
   - Validação de entrada (sanitização, tipos)

6. **Testes não definidos**:
   - Casos de teste críticos (saldo insuficiente, destinatário inválido, concorrência)
   - Dados de teste/mock necessários

### 3. Lista de Priorização (O "Mínimo Inegociável")

#### 🔴 Deve ser resolvido HOJE para desenvolvimento começar:

1. **Definir contrato de API**:
   - Endpoint: `POST /api/v1/transactions`
   - Request: `{ recipient_id: string, amount: number }`
   - Response: `{ transaction_id: string, status: string, new_balance: number }`
   - Códigos de erro: 400 (validação), 401 (não autenticado), 402 (saldo insuficiente), 404 (destinatário não encontrado), 500 (erro servidor)

2. **Definir modelo de dados mínimo**:
   - Tabela `transactions`: id, sender_id, recipient_id, amount, status, created_at
   - Tabela `users`: id, balance (ou tabela separada `wallets`)

3. **Definir regras críticas**:
   - Valor mínimo: 1 QualiPoint
   - Valor máximo: saldo disponível do remetente
   - Operação: síncrona com transação atômica no banco
   - Validação: saldo e destinatário validados no servidor

4. **Definir autenticação**:
   - Método: JWT token no header Authorization
   - Extração de user_id do token para identificar remetente

#### 🟡 Pode ficar para o backlog (mas documentar decisão):

1. Confirmação via painel web (apenas visualização por enquanto)
2. Lista de transações com paginação/filtros avançados
3. Notificações push
4. Comprovantes em PDF
5. Limites diários/semanais (implementar após MVP validado)
6. Histórico completo de transações (apenas últimas 10-20 por enquanto)

#### 🟢 Decisões arquiteturais para alinhar com time:

1. Estratégia de cache (se necessário para performance)
2. Estratégia de monitoramento/logging
3. Estratégia de testes (unitários, integração, E2E)

---

## ✅ Requisito Refinado - Mínimo Inegociável

### Funcionalidade Core (MVP)

**Como usuário autenticado, quero enviar QualiPoints para outro usuário através do aplicativo mobile, de forma que:**

1. **Autenticação**: Devo estar logado e autenticado via JWT
2. **Envio**: Informo o ID do destinatário e o valor desejado
3. **Validação**: O sistema valida no servidor:
   - Meu saldo é suficiente
   - O destinatário existe e está ativo
   - O valor está dentro dos limites (mínimo: 1, máximo: meu saldo)
4. **Processamento**: A transação é processada atomicamente:
   - Meu saldo é debitado
   - O saldo do destinatário é creditado
   - Uma transação é registrada no banco
5. **Resposta**: Recebo confirmação imediata com status e novo saldo
6. **Visualização**: Posso ver minhas transações recentes (últimas 20) no app e no painel web

### Não está no MVP (Backlog)

- Confirmação obrigatória via painel web
- Comprovantes em PDF
- Notificações push
- Limites diários/semanais
- Histórico completo com filtros avançados
