# Heurística CRUD — Análise do Requisito QualiPoints

**Fonte analisada:** `input/qualitpoints.md` — Envio de QualiPoints entre usuários (v2)
**Heurística aplicada:** CRUD
**Data:** 2026-03-29

---

## Create (Criação)

### Operação identificada: **Criar transação de envio de QualiPoints**
- **Endpoint:** `POST /api/v1/transfers`
- **Dados inseridos:** registro de transação (transactionId, senderId, recipientId, amount, status, completedAt)

### Análise

| Aspecto | Status | Observação |
|---------|--------|------------|
| **Validações antes da inserção** | Coberto | Valor 1–10.000, inteiro, destinatário existe e ativo, remetente ≠ destinatário, saldo suficiente, conta ativa |
| **Duplicatas / Conflitos** | Coberto | Idempotency-Key no header evita processamento duplo |
| **Atomicidade** | Coberto | Débito + crédito + registro em transação única com rollback |
| **Rate limiting** | Não coberto (backlog) | Sem limite de transações por minuto/hora no MVP — risco de abuso |
| **Auditoria** | Coberto | Toda tentativa (sucesso/falha) deve ser registrada |

### Gaps e riscos identificados

1. **Rate limiting ausente no MVP** — Sem limite por período, um usuário mal-intencionado pode disparar milhares de transferências válidas em sequência (ex: 1 QualiPoint por vez) para sobrecarregar o sistema ou manipular saldos em cadeia. Recomendação: mesmo no MVP, implementar um rate limit básico (ex: 10 transações/minuto por usuário).

2. **TTL da Idempotency-Key não definido** — O requisito não especifica por quanto tempo a chave de idempotência é armazenada. Se for permanente, a tabela cresce indefinidamente. Se expirar cedo demais, um retry tardio pode gerar débito duplo. **Recomendação:** definir TTL explícito (ex: 24h) e documentar.

3. **Formato da Idempotency-Key** — Não há validação definida para o formato (tamanho máximo, charset). Chaves malformadas ou excessivamente longas podem causar problemas de storage.

4. **Criação de carteira/saldo inicial** — O requisito não menciona como o saldo do usuário é inicializado. Se a carteira (wallet) não existir no momento da transferência, o sistema deve criar uma? Isso afeta a operação de crédito ao destinatário.

---

## Read (Leitura)

### Operações identificadas:
1. **Listar transações recentes** — últimas 10 do usuário autenticado
2. **Consultar saldo** — implícito (App e painel web)
3. **Validação de destinatário** — leitura interna durante o Create

### Análise

| Aspecto | Status | Observação |
|---------|--------|------------|
| **Filtros e ordenação** | Coberto | Filtro por userId (remetente ou destinatário), ordenação por data desc |
| **Paginação** | Parcial | Fixo em 10 registros no MVP — sem paginação real |
| **Controle de acesso** | Coberto | Usuário vê apenas as próprias transações |
| **Performance / Indexação** | Não especificado | Necessário índice em (userId, createdAt) |
| **Caching** | Não especificado | Saldo consultado frequentemente, candidato a cache |

### Gaps e riscos identificados

5. **Sem paginação real** — Fixar em 10 registros funciona no MVP, mas não há endpoint definido para buscar transações anteriores. Quando o backlog evoluir, a API precisará suporte a cursor/offset. **Recomendação:** já desenhar o response com metadata de paginação (`hasMore`, `cursor`) mesmo que no MVP retorne apenas 10.

6. **Leitura de saldo não tem endpoint explícito** — O requisito menciona que App e painel exibem saldo, mas não define um endpoint `GET /api/v1/balance` ou equivalente. A lista de transações recentes também não inclui o saldo no response. **Recomendação:** definir endpoint de saldo ou incluir `currentBalance` no response da listagem.

7. **Indexação não especificada** — Para queries de transações filtradas por usuário e ordenadas por data, é essencial um índice composto `(userId, createdAt DESC)` ou equivalente. Sem isso, a performance degrada rapidamente com volume.

8. **Consistência App/Painel** — O requisito aceita "polling ou refresh manual", mas não define intervalo mínimo. Leituras concorrentes de App e painel podem ver dados diferentes durante uma janela. Isso é aceitável no MVP, mas deve ser documentado.

---

## Update (Atualização)

### Operações identificadas:
1. **Atualização de saldo do remetente** (débito) — durante o Create
2. **Atualização de saldo do destinatário** (crédito) — durante o Create

### Análise

| Aspecto | Status | Observação |
|---------|--------|------------|
| **Concorrência** | Parcial | Atomicidade garantida, mas mecanismo de lock não especificado |
| **Campos atualizáveis** | N/A | Saldo é atualizado apenas pelo sistema, nunca pelo usuário |
| **Conflitos de concorrência** | Risco | Dois envios simultâneos do mesmo remetente podem competir pelo mesmo saldo |
| **Versionamento** | Não especificado | Sem menção a optimistic locking ou versão no saldo |

### Gaps e riscos identificados

9. **Mecanismo de lock do saldo não definido** — O requisito diz "atômica" mas não especifica se usa `SELECT FOR UPDATE`, optimistic locking com versão, ou outro mecanismo. Em cenários concorrentes (dois envios simultâneos do mesmo usuário), sem lock adequado pode ocorrer race condition no saldo. **Recomendação:** especificar pessimistic locking (`SELECT FOR UPDATE` na linha do saldo do remetente) ou optimistic locking com campo `version`.

10. **Não existe Update de transação pelo usuário** — Transações são imutáveis após criação (não há edição, cancelamento ou estorno no MVP). Isso é correto e bem definido, mas vale documentar explicitamente que transações são **append-only**.

11. **Atualização de status da conta** — O requisito menciona contas "bloqueadas" mas não define quem/como bloqueia. Se um update de status da conta ocorrer *durante* uma transferência em andamento, o comportamento não está definido.

---

## Delete (Exclusão)

### Operações identificadas:
- **Nenhuma operação de exclusão definida no MVP**

### Análise

| Aspecto | Status | Observação |
|---------|--------|------------|
| **Exclusão de transações** | N/A | Não previsto — transações são registro permanente |
| **Exclusão de conta** | Não especificado | Sem menção a encerramento de conta |
| **Expiração de Idempotency-Key** | Não especificado | Keys precisam ser limpas eventualmente |
| **Auditoria** | N/A | Sem exclusão = sem necessidade de auditoria de delete |

### Gaps e riscos identificados

12. **Limpeza de Idempotency-Keys** — As chaves de idempotência são, na prática, registros que precisam de exclusão periódica (TTL). Sem um mecanismo de purge (background job ou particionamento por data), a tabela cresce indefinidamente.

13. **Encerramento/exclusão de conta não previsto** — Se um usuário encerrar a conta, o que acontece com o saldo e o histórico de transações? Mesmo que fora do MVP, a modelagem de dados deve considerar soft delete na conta para preservar integridade referencial do histórico.

14. **LGPD / direito ao esquecimento** — Dependendo da jurisdição, pode haver necessidade de anonimizar dados de transações de usuários que solicitam exclusão. Isso impacta a modelagem (separar dados pessoais dos dados de transação).

---

## Análise de Segurança e Acesso

| Operação | Autenticação | Autorização | Auditoria |
|----------|-------------|-------------|-----------|
| Create (transferência) | Bearer token | Conta ativa, saldo suficiente | Log de toda tentativa |
| Read (transações) | Bearer token | Apenas próprias transações | Não especificado |
| Read (saldo) | Bearer token | Apenas próprio saldo | Não especificado |
| Update (saldo) | Apenas sistema | Automático via transferência | Via log da transação |
| Delete | N/A | N/A | N/A |

### Gaps de segurança

15. **Auditoria de leitura não definida** — Apenas o Create tem auditoria explícita. Consultas de saldo e transações não são auditadas. Em contextos de compliance financeiro, leituras também podem precisar de log.

16. **Enumeração de usuários** — O endpoint aceita `recipientId` e retorna 404 se não existe vs 409 se é o próprio. Isso permite a um atacante enumerar IDs de usuários válidos. **Recomendação:** considerar retornar a mesma mensagem genérica para destinatário inválido/inexistente/próprio, ou aceitar o risco documentando-o.

17. **Exposição de saldo no erro 422** — O campo `currentBalance` no response de `INSUFFICIENT_BALANCE` é marcado como opcional, mas se incluído, expõe informação sensível. Se a requisição for interceptada (mesmo com HTTPS), o saldo fica no log do cliente. **Recomendação:** não retornar `currentBalance` no erro.

---

## Análise de Consistência e Integridade

| Aspecto | Modelo | Observação |
|---------|--------|------------|
| Saldo do remetente | Consistência forte | Débito atômico, verificação no momento da transação |
| Saldo do destinatário | Consistência forte | Crédito na mesma transação |
| Lista de transações | Consistência eventual (aceitável) | App e painel podem ter delay |
| Idempotência | Consistência forte | Mesma key = mesmo resultado |

---

## Checklist Final

- [x] **Create**: Validações definidas, idempotência prevista, atomicidade especificada
- [ ] **Create**: Rate limiting ausente, TTL da idempotency key não definido
- [x] **Read**: Controle de acesso definido, ordenação especificada
- [ ] **Read**: Endpoint de saldo não definido, indexação não especificada, paginação limitada
- [x] **Update**: Atomicidade prevista, saldo atualizado apenas pelo sistema
- [ ] **Update**: Mecanismo de lock não especificado (race condition possível)
- [x] **Delete**: Corretamente ausente no MVP (transações imutáveis)
- [ ] **Delete**: Limpeza de idempotency keys não planejada, encerramento de conta não previsto
- [ ] **Segurança**: Risco de enumeração de usuários, exposição de saldo no erro 422
- [ ] **Escalabilidade**: Sem rate limit, sem cache, sem indexação definida

---

## Recomendações Priorizadas

### Críticas (resolver antes do desenvolvimento)
1. Definir mecanismo de lock do saldo (pessimistic ou optimistic) — **risco de race condition**
2. Definir TTL da Idempotency-Key e estratégia de limpeza
3. Adicionar rate limit básico mesmo no MVP (ex: 10 tx/min por usuário)

### Importantes (resolver durante o desenvolvimento)
4. Definir endpoint de consulta de saldo (`GET /api/v1/balance`)
5. Especificar índices necessários (`userId + createdAt`)
6. Remover `currentBalance` do response de erro 422
7. Documentar que transações são imutáveis (append-only)

### Backlog (planejar para próximas versões)
8. Paginação real com cursor na lista de transações
9. Auditoria de operações de leitura
10. Estratégia de soft delete para contas (LGPD)
11. Mitigação de enumeração de usuários
