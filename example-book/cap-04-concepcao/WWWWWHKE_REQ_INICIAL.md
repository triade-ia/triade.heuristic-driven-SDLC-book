# Relatório: WWWWWHKE — REQ_INICIAL (QualiPoints)

**Requisito analisado:** Implementar funcionalidade de envio de "QualiPoints" entre usuários.  
**Skill aplicada:** Cap04_concepcao — WWWWWHKE  
**Data:** 2025-02-18

---

## 1. Resumo do requisito original

- **Descrição:** Usuário pode enviar pontos para outro usuário; sistema com tela, API de processamento e App; requisitos de rapidez e segurança.
- **Contexto:** Carteira digital, operação via app mobile, confirmação via painel web, banco relacional, REST API.
- **Regras citadas (lacunares):** usuário logado; informar ID do destinatário e valor; saldo atualizado após envio; lista de transações recentes.

---

## 2. Questionamentos Críticos (O Filtro)

Para cada dimensão, a pergunta mais difícil que o requisito **não** responde:

### Who (Quem) — [Autorização e Identidade]

- Quem exatamente tem permissão para enviar QualiPoints? Apenas contas "ativas"? Existe verificação de idade, KYC ou status (bloqueada, pendente, em análise)?
- Quem autoriza ou é notificado em caso de falha de sistema (transação travada, saldo inconsistente)? Existe um papel de "suporte" ou "admin" para desbloqueio?
- Quais perfis são afetados: apenas "usuário final" ou também admin, auditor, sistema de conciliação? O destinatário precisa estar em algum estado específico (ex.: conta ativa, aceitando recebimentos)?

### What (O quê) — [Objeto e Limites]

- O que exatamente está sendo movido? É um valor inteiro em "pontos", com casas decimais, ou há conversão para outra unidade?
- Qual o valor **mínimo** e **máximo** por transação e por período (dia/mês) para evitar fraude massiva em segundos ou lavagem?
- Existe limite de quantas transações por minuto/hora por usuário (rate limit de negócio)?

### When (Quando) — [Temporalidade e Concorrência]

- A operação é **síncrona** (usuário espera a confirmação na hora) ou **assíncrona** (processamento em fila, notificação depois)?
- Existe **expiração** de uma transação "em processamento"? Após quantos segundos o sistema desiste ou tenta novamente?
- O que acontece se duas ações (ex.: dois envios do mesmo usuário ou débito concorrente) ocorrerem no mesmo milissegundo? A regra de saldo é atômica?

### Where (Onde) — [Arquitetura e Localização]

- **Onde** a regra de saldo é validada: apenas no App (confiável?) ou a **API garante** a verdade única (validação server-side obrigatória)?
- Onde o dado é persistido (qual tabela/serviço) e onde ele pode ficar **inconsistente** (ex.: débito ok, crédito falhou)?
- A "lista de transações recentes" é gerada no mesmo banco da carteira ou em outro sistema (cache, read replica)? Onde está a fonte da verdade?

### Why (Por que) — [Valor vs. Complexidade]

- Esta funcionalidade resolve o problema central (transferência confiável entre usuários) ou há acessórios que adicionam latência e risco (ex.: comprovante em PDF, e-mail por transação)?
- Precisamos de "confirmação via painel web" no **mesmo** fluxo do envio no App, ou um log de transação no painel basta para o MVP?

### How (Como) — [Transição de Estado e Segurança]

- **Como** o sistema sai do estado "saldo X" para "saldo X - valor" de forma atômica? Há transação distribuída, saga ou compensação?
- Como garantimos que o saldo **não** seja descontado se a confirmação de crédito no destinatário falhar (atomicidade débito-crédito)?
- Qual protocolo protege a transição (HTTPS, idempotency key, auditoria de tentativas)?

### Keep (Manter) — [Essencialidade]

- Validação de saldo **no servidor** antes de debitar.
- Identificação do remetente (usuário logado) e do destinatário (ID válido e elegível).
- Atualização consistente do saldo após envio (regra de negócio atômica ou claramente definida).
- Registro de transação (quem, para quem, valor, momento) para auditoria e "transações recentes".
- Comunicação segura (HTTPS) e autenticação do usuário.

### Eliminate (Eliminar) — [Redução de Ruído]

- "Confirmada via painel web" como **obrigação no fluxo** do envio — pode ser apenas visualização no painel, sem aprovação manual, para o MVP.
- Exigência de "lista de transações recentes" **rica** no primeiro release — um limite pequeno (ex.: últimas 10) e sem filtros complexos reduz superfície de erro.
- Suposições não escritas (ex.: "tem que ter comprovante", "tem que mandar e-mail") — remover tudo que não estiver explícito no requisito mínimo.

---

## 3. Gaps de Implementação (O Risco)

| # | Gap | Impacto |
|---|-----|--------|
| 1 | **Contrato da API** (endpoint, payload, códigos HTTP, formato de erro) | Desenvolvedor não sabe como chamar nem como tratar falhas. |
| 2 | **Regras de valor** (mínimo, máximo, decimal ou inteiro) | Risco de aceitar 0, negativo ou valores absurdos. |
| 3 | **Comportamento em timeout/falha** (retry, cancelar, estado "pendente") | Inconsistência de saldo e má experiência (usuário não sabe se enviou). |
| 4 | **Idempotência** (chave de idempotência para evitar débito duplo) | Risco de duplicar débito em retentativas. |
| 5 | **Mensagens de erro mapeadas** (destinatário inválido, saldo insuficiente, conta bloqueada) | UI e integrações não sabem o que exibir. |
| 6 | **Definição de "transações recentes"** (quantas, período, ordenação, por usuário) | Backend e front divergem ou implementam ad hoc. |
| 7 | **Onde valida saldo** (só API ou também App) | Dupla fonte de verdade ou validação só no cliente (inseguro). |

---

## 4. Keep (Manter) — Resumo

- Autenticação do usuário (logado) para enviar.
- Validação server-side de saldo, destinatário e valor.
- Operação atômica ou bem definida (débito + crédito + registro).
- Registro de transação para auditoria e lista recente.
- Uso de HTTPS e boas práticas de segurança na API.

---

## 5. Eliminate (Eliminar) — Resumo

- Aprovação manual no painel web como parte do fluxo de envio (manter só consulta no painel).
- Funcionalidades não citadas (comprovante PDF, e-mail automático, filtros avançados de extrato) até que sejam requisitadas.
- Suposições implícitas que não estejam no escopo mínimo acordado.

---

## 6. Lista de Priorização (O Mínimo Inegociável)

### Resolver agora (para o desenvolvimento começar)

1. **Contrato da API de envio** — endpoint, método, corpo (remetente implícito pelo token, destinatário, valor), respostas 200/4xx/5xx e formato de erro.
2. **Regras de valor** — mínimo (ex.: 1), máximo por transação e se valor é inteiro ou decimal.
3. **Decisão síncrono vs. assíncrono** — usuário espera confirmação na hora ou recebe notificação depois; e o que fazer em timeout (retry, cancelar, estado pendente).
4. **Garantia de consistência** — validação de saldo e destinatário na API; definição de atomicidade (débito + crédito + registro).
5. **Idempotency key** — como o cliente envia e como a API evita processar duas vezes o mesmo envio.

### Backlog (refinamento posterior)

- Limite de transações por período (rate limit de negócio).
- Definição detalhada da "lista de transações recentes" (quantidade, período, filtros).
- Papéis de suporte/admin para desbloqueio e conciliação.
- Comprovante, e-mail ou notificação push (se forem requisitados).

---

## 7. Parecer Técnico de Viabilidade

**Requisito está pronto para desenvolvimento?** **Não.**

O requisito deixa em aberto pontos críticos para codificação e operação segura: contrato da API, limites de valor, comportamento em falha/timeout, idempotência e validação server-side. Recomenda-se **refinamento** antes do início da implementação, priorizando os itens da lista "Resolver agora". Após fechamento desses tópicos, o requisito tende a estar em condições de viabilidade imediata para desenvolvimento.

---

## 8. Conclusão

A aplicação da heurística **WWWWWHKE** ao REQ_INICIAL evidencia que a ideia (envio de QualiPoints entre usuários, rápido e seguro) é clara, mas o nível de detalhe é insuficiente para evitar ambiguidade e feature creep. Os questionamentos Who, What, When, Where, Why e How expõem lacunas de autorização, limites, temporalidade, arquitetura e segurança. **Keep** e **Eliminate** ajudam a fixar o mínimo inegociável e a cortar suposições. O parecer é que o requisito **precisa de refinamento** (contrato de API, regras de valor, timeout/idempotência e consistência) antes de ser considerado pronto para desenvolvimento.
