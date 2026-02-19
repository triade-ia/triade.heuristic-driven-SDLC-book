# Relatório: User Scenarios — REQ_INICIAL (QualiPoints)

**Requisito analisado:** Implementar funcionalidade de envio de "QualiPoints" entre usuários.  
**Skill aplicada:** Cap04_concepcao — User Scenarios  
**Data:** 2025-02-18

---

## 1. Resumo do requisito original

- **Descrição:** O usuário quer poder enviar pontos para outro usuário (ex.: amigo). O sistema deve ter tela para a operação, API que processa o envio e funcionar no App. Requisitos de rapidez e segurança.
- **Contexto:** Carteira digital; operação via app mobile; confirmação via painel web; banco relacional; comunicação REST API.
- **Regras citadas (lacunares):** usuário logado; informar ID do destinatário e valor; saldo atualizado após envio; lista de transações recentes.

---

## 2. Cenário 1: A Persona no Fluxo Padrão

### A Persona (nome e breve perfil)

- **Nome:** Ana.
- **Quem é:** Usuária que já utiliza o app da carteira, conhece o saldo e costuma enviar pontos para amigos (pagamento de café, presente, divisão de conta).
- **Por que está fazendo isso agora:** Ação de rotina; quer enviar os pontos combinados para uma amiga após um almoço em conjunto. Não é a primeira vez que transfere.

### A Situação em que a Persona Está

- **Onde está:** Em casa, no sofá, com Wi-Fi estável; usa apenas o app no celular e às vezes confere o extrato no painel web no notebook.
- **Contexto imediato:** Acabou de combinar com a amiga Julia o valor a repassar; abriu o app para fazer a transferência na hora e não esquecer.
- **Estado emocional ou prático:** Calma, confiante; sabe onde clicar e o que preencher.
- **Riscos ou incertezas já presentes:** Nenhum crítico; só a dúvida ocasional se o ID da Julia está correto (ela costuma enviar por nome ou telefone quando o sistema permitir).

Essa situação favorece o fluxo feliz: Ana consegue concluir o envio e ver a confirmação sem sustos.

### O Ambiente e o Contexto Técnico

Conexão estável (Wi-Fi ou 4G); um dispositivo por vez (app ou painel); sem interrupções relevantes.

### O Fluxo de Valor (Happy Path)

1. Ana abre o app e está logada.
2. Acessa “Enviar QualiPoints” (ou equivalente).
3. Informa destinatário (ID da Julia ou busca por nome/telefone, se o requisito evoluir) e valor.
4. Confirma; o sistema valida saldo, processa e atualiza os saldos.
5. Ana vê confirmação de sucesso e a transação na lista de recentes.
6. Opcionalmente confere no painel web para ter certeza.

### O "E se?" (Edge Cases de Negócio)

- **Destinatário inexistente ou ID errado:** Ana digita um ID que não existe ou troca um dígito — o requisito não detalha mensagem nem bloqueio; ela pode achar que enviou quando não enviou.
- **Valor maior que o saldo:** O requisito não diz como o sistema se comporta (mensagem, bloqueio, sugestão de valor máximo); Ana pode tentar enviar mais do que tem e ficar sem feedback claro.
- **App fechado logo após confirmar:** Se Ana fechar o app pensando que já terminou, não sabe se a transação foi concluída ou ficou pendente — falta definição de estados e de como informar o usuário.

---

## 3. Cenário 2: A Persona sob Pressão ou Estresse

### A Persona (nome e breve perfil)

- **Nome:** Roberto.
- **Quem é:** Mesmo perfil de usuário do app, mas neste momento está em situação de urgência ou distração (pagar alguém na fila, resolver uma dívida rápida, multitarefa).
- **Por que está fazendo isso agora:** Necessidade imediata; pouco tempo; pode estar dividindo atenção com outra tarefa ou com o ambiente (fila, transporte).

### A Situação em que a Persona Está

- **Onde está:** No metrô ou no ônibus, ou na fila de um estabelecimento; rede 3G ou instável; tela do celular pequena; pode receber ligação ou notificação a qualquer momento.
- **Contexto imediato:** Precisou pagar um colega na hora (dividir conta, reembolso) ou está tentando enviar os pontos antes de perder o sinal ou de ser chamado.
- **Estado emocional ou prático:** Com pressa, um pouco ansioso; quer que “funcione logo”; sensível a travamentos ou telas que não respondem.
- **Riscos ou incertezas já presentes:** Rede pode cair no meio; não sabe se deve esperar ou sair da tela; medo de enviar duas vezes se clicar de novo.

Para Roberto, o impacto de timeout, falha de rede ou falta de feedback é alto: ele pode achar que falhou quando na verdade processou, ou o contrário.

### O Ambiente e o Contexto Técnico

Rede instável ou 3G; possível alternância entre app e painel em outro dispositivo; interrupções (ligação, notificação); dispositivo móvel com tela pequena.

### O Fluxo de Valor (Happy Path)

Mesmo fluxo do Cenário 1, mas com foco em: resposta rápida, feedback claro de “processando” e “concluído”, e possibilidade de retry em caso de falha de rede, sem duplicar o envio.

### O "E se?" (Edge Cases de Negócio)

- **Timeout:** O requisito não define tempo máximo de espera nem o que fazer (retry, mensagem, estado da transação). Se Roberto estiver com pressa e a API demorar, ele não sabe se deve esperar ou tentar de novo.
- **Rede cai no meio:** Não está definido se a transação fica “pendente”, “em processamento” ou se Roberto deve refazer — e se refizer, pode haver débito duplo sem idempotência.
- **App em segundo plano ou fechado:** Não há regra para notificação de conclusão ou falha; Roberto pode perder a confirmação e ficar na dúvida.
- **Múltiplos envios rápidos (duplo clique / retry):** Risco de duplicidade ou race condition se não houver idempotência ou fila adequada.

---

## 4. Cenário 3: A Persona Mal Intencionada ou Confusa

### A Persona (nome e breve perfil)

- **Nome:** Carla (confusa) e Bruno (mal intencionado).
- **Carla — Quem é:** Usuária que às vezes se confunde com telas e termos; acha que “enviar” pode ser “solicitar” ou não entende bem o que é “ID do destinatário”.
- **Bruno — Quem é:** Usuário que tenta explorar falhas (múltiplas contas, envios em loop, replay de requisições).
- **Por que estão fazendo isso agora:** Carla por engano ou dúvida; Bruno para testar limites ou obter vantagem.

### A Situação em que a Persona Está

- **Carla:** Em casa ou no trabalho; preenche o formulário sem ter certeza do destinatário ou do valor (ex.: digita 1000 em vez de 10); pode enviar para si mesma pensando que é “trocar de carteira”.
- **Bruno:** Em qualquer ambiente; repete a mesma operação várias vezes em pouco tempo ou reutiliza o mesmo pedido para ver se debita duas vezes; pode tentar acessar listagem de transações de outros usuários.

O impacto aqui é de segurança, integridade dos dados e experiência: Carla precisa de validações e mensagens que a protejam do próprio engano; Bruno precisa de barreiras (validação, idempotência, autorização).

### O Ambiente e o Contexto Técnico

Qualquer; pode ser uso repetido em pouco tempo ou testes manuais sem critério; múltiplas abas ou sessões no caso de Bruno.

### O Fluxo de Valor (Happy Path)

Não se aplica como “sucesso”; o foco é em regras que impeçam ou tratem desvios (validação, mensagens claras, idempotência, controle de acesso).

### O "E se?" (Edge Cases de Negócio)

- **Envio para si mesmo:** O requisito não proíbe; pode ser útil em alguns contextos ou indesejado; precisa de regra explícita. Carla pode fazer por engano.
- **Valor zero ou negativo:** Não há validação citada; a API deve rejeitar e retornar erro claro para Carla e Bruno.
- **ID de destinatário inválido ou de outra base:** Sem regra de validação e mensagem de erro; tanto Carla quanto Bruno podem gerar requisições inválidas.
- **Tentativa de replay:** Bruno reutiliza o mesmo pedido para debitar duas vezes; necessidade de idempotência (ex.: idempotency key).
- **Listagem de transações:** Se “recentes” for acessível sem restrição, pode vazar informação entre usuários; necessidade de autorização por usuário/sessão.

---

## 5. Lacunas de Decisão

Gaps que o requisito original **não cobria** e que impactam concepção/design, com relação às situações das personas quando aplicável:

| # | Lacuna | Impacto (e link com as personas) |
|---|--------|----------------------------------|
| 1 | **Feedback e estados da transação** | Não há definição de estados (processando, concluído, falhou, pendente) nem de mensagens/UI para cada um. **Ana** precisa ver que deu certo; **Roberto** precisa saber se deve esperar ou tentar de novo em caso de demora ou falha. Falta especificar timeout e o que mostrar ao usuário. |
| 2 | **Comportamento em falha de rede/timeout** | Não está definido se a transação é retentada, marcada como pendente ou cancelada; nem como o usuário é informado (push, e-mail, ao reabrir o app). **Roberto**, sob pressão, é especialmente afetado: pode duplicar envio ou achar que falhou quando não falhou. |
| 3 | **Validações e segurança de entrada** | Não há regras para: valor mínimo/máximo, valor zero ou negativo, envio para si mesmo, destinatário inexistente ou inativo. **Carla** precisa de proteção contra engano; **Bruno**, de barreiras. Também não há menção a idempotência para evitar débito duplo. |
| 4 | **Escopo de “transações recentes”** | Não está definido: quantas transações, período, filtros, se é por usuário e se há controle de acesso (só minhas transações). Impacta **Ana** (clareza) e **Bruno** (risco de vazamento entre usuários). |
| 5 | **Consistência entre app e painel web** | Não está claro como se garante que o saldo e a lista no app e no painel web permaneçam consistentes após o envio (tempo real, refresh, polling). **Ana** que confere nos dois lugares precisa de coerência. |

---

## 6. Edge Cases de Negócio (síntese)

- **Cenário 1 (Ana — fluxo padrão):** Destinatário inválido ou ID errado; valor maior que o saldo; app fechado logo após confirmar; falta de feedback e estados claros.
- **Cenário 2 (Roberto — pressão/estresse):** Timeout da API; rede cai no meio; app em background; vários envios rápidos (risco de duplicidade); necessidade de retry e idempotência.
- **Cenário 3 (Carla confusa / Bruno mal intencionado):** Envio para si mesmo; valor zero/negativo; ID inválido; replay de requisição; listagem de transações sem controle de acesso.

---

## 7. Conclusão

A aplicação da heurística **User Scenarios** ao REQ_INICIAL, com **personas nomeadas** (Ana, Roberto, Carla, Bruno) e **situações descritas**, evidencia que o requisito cobre bem o fluxo feliz (usuário logado, envia com ID e valor, saldo atualizado e lista de recentes), mas deixa em aberto:

- Estados e feedback da transação (incluindo timeout e falha), críticos para **Ana** e **Roberto**.
- Resiliência (rede instável, retry, idempotência), essencial para **Roberto** sob pressão.
- Validações e regras de negócio (valor, destinatário, envio para si mesmo), importantes para **Carla** e **Bruno**.
- Definição e segurança da lista de “transações recentes”, com impacto para **Ana** (clareza) e **Bruno** (acesso indevido).
- Alinhamento de comportamento entre app e painel web.

Recomenda-se tratar essas lacunas na fase de concepção/design (Cap 04/05) antes de detalhar API e telas.
