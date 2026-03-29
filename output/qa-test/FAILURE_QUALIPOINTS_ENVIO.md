# Relatório FAILURE — Envio de QualiPoints entre Usuários

**Requisito analisado:** Requisito Revisado v2 — Envio de QualiPoints entre usuários
**Heurística aplicada:** FAILURE (Ben Simo)
**Contexto:** Análise de requisito para refinamento de demandas e elaboração de testes
**Data:** 2026-03-29

---

## Resumo Executivo

O requisito v2 de Envio de QualiPoints é bem estruturado e já endereça vários cenários de falha (idempotência, atomicidade, timeout, edge cases). A análise FAILURE identifica **gaps residuais** em cada uma das sete dimensões, com foco em cenários não especificados, riscos de impacto e oportunidades de melhoria na resiliência do sistema.

---

## 1. Functional (Funcional)

**Pergunta central:** O sistema continua funcionando parcial ou totalmente após uma falha? A falha em um componente afeta outros?

### Pontos fortes do requisito
- Operação atômica (débito + crédito + registro) com rollback em caso de falha (seções 6.2, 8.1)
- Idempotência previne débito duplo em retentativas (seção 8.2)
- Validação centralizada no servidor — App não toma decisão de negócio (seção 6.1)

### Gaps identificados

| # | Gap | Risco | Recomendação |
|---|-----|-------|--------------|
| F1 | **Sem fallback ou modo degradado definido.** O requisito não especifica o que acontece se o banco de dados estiver indisponível ou lento. A API simplesmente retorna 5xx? O App exibe apenas mensagem genérica? | Sistema totalmente indisponível para transferências sem alternativa | Especificar comportamento esperado quando o banco está indisponível: retornar 503 com mensagem clara ("Serviço temporariamente indisponível") e orientar retry |
| F2 | **Sem isolamento entre funcionalidades.** Se a API de transferência falhar, a consulta de saldo e lista de transações recentes também ficam afetadas? Compartilham mesma conexão de banco? | Falha em transferência pode derrubar consultas | Definir se endpoints de consulta (saldo, transações recentes) devem funcionar independentemente do endpoint de envio |
| F3 | **Sem especificação de circuit breaker ou health check.** Para o MVP pode ser aceitável, mas não há menção a qualquer mecanismo de detecção de falha sistêmica | Falhas repetidas sem detecção proativa | Considerar ao menos um health check básico da API e do banco para monitoramento |
| F4 | **Comportamento de consulta de transações recentes após falha parcial.** Se uma transação falhou e fez rollback, ela aparece na lista de recentes? O requisito diz "últimas 10 transações" mas não especifica se transações com falha são listadas | Confusão do usuário ao ver ou não ver tentativas com falha | Especificar se a lista inclui apenas transações com status "completed" ou também tentativas com falha |
| F5 | **Sem rate limit no MVP.** O requisito reconhece isso (seção 4.3), mas sem nenhum rate limit, um usuário ou bot pode sobrecarregar a API com requisições, afetando a funcionalidade para todos | Risco de indisponibilidade por abuso | Considerar ao menos um rate limit básico (ex.: 10 transações/minuto por usuário) mesmo no MVP |

---

## 2. Appropriate (Apropriado)

**Pergunta central:** A resposta do sistema à falha é apropriada? Mensagens, códigos e ações fazem sentido?

### Pontos fortes do requisito
- Tabela de respostas HTTP bem definida com códigos e mensagens claras (seção 10.4)
- Mapeamento de mensagens de erro para exibição no App (seção 11.2)
- Diferenciação de cenários com códigos específicos (409 para self-transfer, 422 para saldo insuficiente)

### Gaps identificados

| # | Gap | Risco | Recomendação |
|---|-----|-------|--------------|
| A1 | **Uso de 200 para idempotência em vez de 200/304.** Quando a API retorna o mesmo resultado de uma Idempotency-Key já processada, o código é sempre 200. Não há distinção entre "processado agora" e "já processado antes" | Cliente não sabe se a transação foi processada nesta chamada ou antes. Pode causar confusão em logs e depuração | Considerar header de resposta indicando se é resultado original ou replay (ex.: `Idempotency-Replayed: true`) |
| A2 | **Conflito potencial no código 408 vs 504 para timeout.** O requisito diz "504 Gateway Timeout (ou 408 Request Timeout, conforme convenção)" — sem definir qual usar | Inconsistência entre ambientes; clientes implementam retry diferente para 408 vs 504 | Definir um único código de timeout para o projeto e documentar |
| A3 | **Sem especificação de resposta para Idempotency-Key ausente ou inválida.** O header é obrigatório, mas não há código de erro definido para quando está ausente ou malformado | App sem Idempotency-Key recebe erro genérico 400? Ou 5xx? | Adicionar resposta específica: 400 com código `MISSING_IDEMPOTENCY_KEY` ou `INVALID_IDEMPOTENCY_KEY` |
| A4 | **Sem especificação de resposta para Content-Type incorreto.** Header obrigatório mas sem erro mapeado | Erro genérico dificulta debug | Definir resposta 415 Unsupported Media Type |
| A5 | **Sem definição de resposta para valor decimal (ex.: 10.5).** O requisito diz "inteiro" mas não define o erro específico para valor decimal | Mensagem genérica de INVALID_AMOUNT pode não ser clara o suficiente | Considerar mensagem específica: "O valor deve ser um número inteiro, sem casas decimais" |
| A6 | **Sem definição de comportamento para campos extras no body.** Se o cliente enviar campos além de `recipientId` e `amount`, a API ignora ou rejeita? | Inconsistência de contrato; risco de dados sensíveis passados acidentalmente | Definir política: ignorar campos extras (loose) ou rejeitar (strict) |

---

## 3. Impact (Impacto)

**Pergunta central:** Qual o impacto da falha nos dados, no usuário e no negócio?

### Pontos fortes do requisito
- Atomicidade garante consistência de dados (sem débito sem crédito)
- Idempotência evita débito duplo — proteção financeira crítica
- Lista de transações usa mesma fonte de verdade (seção 6.3)

### Gaps identificados

| # | Gap | Risco | Recomendação |
|---|-----|-------|--------------|
| I1 | **Sem análise de impacto de falha no crédito após débito bem-sucedido dentro da mesma transação de banco.** Embora a atomicidade esteja definida, o requisito não detalha o cenário onde o commit do banco falha após débito + crédito terem sido escritos (falha de infraestrutura no momento do commit) | Em cenário extremo de crash do banco durante commit, pode haver inconsistência | Documentar expectativa: transação de banco garante tudo-ou-nada; em caso de crash, o banco faz rollback automático. Monitorar com alertas |
| I2 | **Sem especificação de impacto de expiração de Idempotency-Key.** Por quanto tempo a chave é armazenada? Se expirar e o cliente reenviar, processa novamente? | Risco de débito duplo se a chave expirar antes do retry | Definir TTL da Idempotency-Key (ex.: 24h) e documentar comportamento após expiração |
| I3 | **Sem especificação de impacto em concorrência de saldo.** Dois envios simultâneos do mesmo remetente com Idempotency-Keys diferentes, onde a soma excede o saldo. O requisito menciona atomicidade mas não detalha lock de saldo | Possível race condition: ambas passam na validação de saldo antes do débito | Especificar: lock pessimista ou otimista no saldo do remetente; em caso de conflito, segundo envio recebe 422 INSUFFICIENT_BALANCE |
| I4 | **Sem limite de tentativas de retry.** O cliente pode retentar indefinidamente com mesma Idempotency-Key | Carga desnecessária na API; experiência ruim se a falha é persistente | Definir limite de retries no App (ex.: 3 tentativas) com backoff exponencial |
| I5 | **Sem impacto documentado de conta do remetente ser bloqueada entre envio e retry.** Remetente inicia envio (timeout), conta é bloqueada por admin, remetente faz retry com mesma key | Comportamento indefinido: a API retorna o resultado original (200) ou verifica status da conta no retry? | Definir: retry com mesma key sempre retorna resultado original, independente de mudança de status |

---

## 4. Log (Logs)

**Pergunta central:** O sistema registra informações úteis sobre a falha? Suficientes para diagnóstico? Sem dados sensíveis?

### Pontos fortes do requisito
- Requisito menciona auditoria de toda tentativa de envio (seção 8.3)
- Menciona registro com idempotency key, usuário, valor, destinatário e resultado

### Gaps identificados

| # | Gap | Risco | Recomendação |
|---|-----|-------|--------------|
| L1 | **Sem definição de estrutura de log.** O requisito menciona "log" mas não especifica formato, campos obrigatórios ou nível (INFO, WARN, ERROR) | Logs inconsistentes dificultam busca e alertas em produção | Definir estrutura mínima: `{ timestamp, level, request_id, idempotency_key, sender_id, recipient_id, amount, status_code, error_code, duration_ms }` |
| L2 | **Sem definição de correlation/request ID.** Não há menção a request ID ou trace ID para rastrear uma requisição do App até o banco | Impossível rastrear uma requisição end-to-end em caso de falha | Adicionar header `X-Request-ID` gerado pelo App ou pela API, registrado em todos os logs da requisição |
| L3 | **Sem especificação de quais dados NÃO devem ser logados.** Token de autenticação, dados pessoais do usuário? | Risco de expor dados sensíveis em logs de produção | Definir política: não logar tokens, senhas, PII. Logar apenas IDs de usuário, não nomes ou e-mails |
| L4 | **Sem definição de nível de log por tipo de falha.** 4xx é WARN ou INFO? 5xx é ERROR? Timeout é ERROR ou WARN? | Alertas mal calibrados — muitos falsos positivos ou falhas ignoradas | Definir: 4xx → WARN (erro do cliente), 5xx → ERROR (erro do sistema), timeout → ERROR |
| L5 | **Sem menção a métricas ou alertas.** Quantas transações por minuto? Taxa de erro? Latência p99? | Sem visibilidade operacional; falhas detectadas apenas por reclamação de usuário | Definir métricas mínimas: taxa de sucesso/erro, latência média e p99, volume por minuto |

---

## 5. UI (Interface do Usuário)

**Pergunta central:** A interface reflete o estado da falha de forma clara? A UI trava ou fica inconsistente?

### Pontos fortes do requisito
- Tabela de estados da transação (Processando/Concluído/Falhou) com ação na UI (seção 11.1)
- Mensagens mapeadas por código de erro (seção 11.2)
- Desabilitação de novo envio durante processamento

### Gaps identificados

| # | Gap | Risco | Recomendação |
|---|-----|-------|--------------|
| U1 | **Sem especificação de timeout da UI.** A API tem timeout de 10s, mas quanto tempo o App espera antes de mostrar erro? O mesmo 10s? Mais? | Usuário fica com loading infinito se a conexão cair sem resposta | Definir timeout do App (ex.: 12s, ligeiramente acima da API) e comportamento ao atingir |
| U2 | **Sem especificação de estado da UI após erro.** O formulário de envio é limpo? Os campos (destinatário, valor) são preservados? | Usuário perde dados preenchidos e precisa redigitar | Especificar: após erro 4xx, preservar campos preenchidos; após sucesso, limpar formulário |
| U3 | **Sem especificação de feedback durante o processamento.** "Indicador de carregamento" — qual tipo? Spinner? Barra de progresso? Mensagem textual? | UI inconsistente entre plataformas; feedback insuficiente | Definir ao menos: spinner com texto "Processando envio..." e desabilitação do botão |
| U4 | **Sem especificação de duplo clique / múltiplos taps.** O requisito menciona que idempotência resolve no backend, mas não especifica prevenção na UI | Experiência ruim: múltiplas requisições desnecessárias antes do backend rejeitar | Especificar: desabilitar botão "Enviar" imediatamente após primeiro toque; reabilitar após resposta |
| U5 | **Sem definição de exibição de saldo atualizado pós-erro.** Se o erro é 422 (saldo insuficiente), a UI atualiza o saldo exibido? | Usuário vê saldo desatualizado e tenta novamente com mesmo resultado | Especificar: em erro 422, atualizar saldo exibido (se a API retorna `currentBalance`) |
| U6 | **Sem especificação de acessibilidade das mensagens de erro.** Screen readers, contraste, tamanho de fonte | Usuários com deficiência visual não percebem o erro | Definir requisitos mínimos de acessibilidade para mensagens de erro (ARIA roles, contraste WCAG) |

---

## 6. Recovery (Recuperação)

**Pergunta central:** O sistema se recupera automaticamente? É fácil para o usuário se recuperar?

### Pontos fortes do requisito
- Retry com mesma Idempotency-Key é seguro (seção 8.2, 12)
- Mensagem de timeout orienta retry (seção 11.2)
- Fluxo síncrono simplifica recuperação (sem estados pendentes)

### Gaps identificados

| # | Gap | Risco | Recomendação |
|---|-----|-------|--------------|
| R1 | **Sem retry automático pelo App.** O requisito diz "pode tentar novamente" mas não especifica se o App faz retry automático em timeout/5xx ou se depende de ação manual do usuário | Usuário precisa clicar "Tentar novamente" manualmente para cada falha temporária | Especificar: retry automático (1-2 tentativas com backoff) em timeout e 5xx antes de exibir erro ao usuário |
| R2 | **Sem botão explícito "Tentar novamente" na especificação da UI.** As mensagens de erro são definidas, mas não há menção a um botão de retry na tela | Usuário vê a mensagem de erro mas não sabe como retentar (precisa sair e voltar?) | Adicionar à especificação da UI: botão "Tentar novamente" visível após erro de timeout ou 5xx |
| R3 | **Sem orientação de recovery para erro persistente.** Se após N tentativas o erro persiste, o que o usuário faz? | Usuário fica preso tentando infinitamente | Definir: após 3 tentativas, exibir "Se o problema persistir, entre em contato com o suporte" com link ou canal |
| R4 | **Sem recovery para sessão expirada durante envio.** Se o token expira entre o preenchimento e o envio, o usuário recebe 401 e perde o contexto? | Usuário é redirecionado para login e perde dados do formulário | Especificar: após re-login, restaurar formulário preenchido; ou usar refresh token para renovar sessão transparentemente |
| R5 | **Sem especificação de recuperação do painel web.** Se o painel web mostra saldo desatualizado após uma transferência no App, como o usuário "se recupera"? | Confusão: saldo diferente entre App e painel | Especificar: botão de "Atualizar" visível no painel; ou auto-refresh a cada N segundos |

---

## 7. Emotions (Emoções)

**Pergunta central:** Como a falha afeta o estado emocional do usuário? Ele se sente no controle ou desamparado?

### Pontos fortes do requisito
- Mensagens não técnicas e claras (seção 11.2)
- Mensagem de timeout indica que pode tentar novamente (não culpa o usuário)
- Diferenciação de mensagens por tipo de erro

### Gaps identificados

| # | Gap | Risco | Recomendação |
|---|-----|-------|--------------|
| E1 | **Mensagem "Ocorreu um erro" para INTERNAL_ERROR é vaga.** Não transmite transparência nem indica temporalidade | Usuário não sabe se é algo que ele fez, se é temporário, ou se perdeu dinheiro | Melhorar para: "Tivemos um problema técnico. Seu saldo não foi alterado. Tente novamente em instantes." |
| E2 | **Sem confirmação de que o saldo não foi afetado em caso de erro.** Mensagens de erro não tranquilizam sobre integridade financeira | Ansiedade: "Meu dinheiro foi debitado mesmo com erro?" — é a maior preocupação em sistemas financeiros | Adicionar a mensagens de 5xx e timeout: "Seu saldo não foi alterado" (quando aplicável, ou seja, quando rollback ocorreu) |
| E3 | **Mensagem de ACCOUNT_NOT_ACTIVE pode ser assustadora.** "Sua conta não está ativa para envio. Entre em contato com o suporte." — sem contexto sobre por que ou como resolver | Medo de perda de acesso ou de fundos | Melhorar: "Sua conta está temporariamente restrita para envios. Para mais informações, entre em contato com o suporte em [canal]." |
| E4 | **Sem indicação de tempo estimado para resolução.** Em timeout e 5xx, "tente novamente" mas quando? Agora? Em 5 minutos? | Incerteza aumenta frustração | Considerar: "Tente novamente em alguns segundos" para timeout; "Tente novamente em alguns minutos" para 5xx recorrente |
| E5 | **Sem tom empático em mensagem de saldo insuficiente.** "Saldo insuficiente. Seu saldo atual não permite este envio." — funcional mas frio | Frustração, especialmente se o usuário tem expectativa de ter saldo | Considerar: "O saldo disponível é menor que o valor informado. Você pode ajustar o valor ou verificar seu saldo." — oferece opção em vez de apenas bloquear |

---

## Checklist de Análise FAILURE — Envio de QualiPoints

- [x] **Functional**: Atomicidade e idempotência bem definidas; faltam fallback, isolamento de falhas e rate limit
- [x] **Appropriate**: Tabela de respostas HTTP sólida; faltam respostas para Idempotency-Key ausente, Content-Type inválido e definição de timeout único
- [x] **Impact**: Proteção financeira via atomicidade e idempotência; faltam TTL de idempotency key, lock de saldo e limite de retries
- [x] **Log**: Auditoria mencionada; faltam estrutura de log, correlation ID, política de dados sensíveis e métricas
- [x] **UI**: Estados e mensagens definidos; faltam timeout da UI, preservação de campos, feedback durante processamento e acessibilidade
- [x] **Recovery**: Retry via idempotência; faltam retry automático, botão explícito de retry, orientação para falha persistente e recovery de sessão
- [x] **Emotions**: Mensagens não técnicas; faltam confirmação de integridade de saldo em erro, tom empático e indicação de temporalidade

---

## Priorização de Gaps

### Críticos (resolver antes do desenvolvimento)

| # | Dimensão | Gap | Justificativa |
|---|----------|-----|---------------|
| I2 | Impact | TTL da Idempotency-Key não definido | Risco de débito duplo |
| I3 | Impact | Lock de saldo em concorrência | Risco de inconsistência financeira |
| A3 | Appropriate | Resposta para Idempotency-Key ausente | Contrato de API incompleto |
| L2 | Log | Correlation/Request ID | Essencial para diagnóstico em produção |

### Importantes (resolver durante o desenvolvimento)

| # | Dimensão | Gap | Justificativa |
|---|----------|-----|---------------|
| F1 | Functional | Comportamento com banco indisponível | Resiliência básica |
| A2 | Appropriate | Definição única de código de timeout | Consistência de contrato |
| L1 | Log | Estrutura de log | Operação em produção |
| U2 | UI | Preservação de campos após erro | Experiência do usuário |
| R1 | Recovery | Retry automático em timeout/5xx | Recuperação transparente |
| E1-E2 | Emotions | Mensagens que confirmam integridade do saldo | Confiança do usuário |

### Desejáveis (backlog ou refinamento posterior)

| # | Dimensão | Gap | Justificativa |
|---|----------|-----|---------------|
| F5 | Functional | Rate limit básico | Proteção contra abuso |
| U6 | UI | Acessibilidade | Inclusão |
| L5 | Log | Métricas e alertas | Observabilidade |
| R4 | Recovery | Recovery de sessão expirada | Fluxo edge case |

---

## Cenários de Teste FAILURE Recomendados

### Functional
| Cenário | Resultado esperado |
|---------|-------------------|
| Banco indisponível durante envio | 503 com mensagem "Serviço temporariamente indisponível" |
| Falha no crédito após débito (dentro da transação) | Rollback completo; saldo do remetente inalterado |
| Dois envios simultâneos que somados excedem saldo | Primeiro processa; segundo retorna 422 |
| API de transferência falha; endpoint de consulta de saldo | Consulta de saldo continua funcionando |

### Appropriate
| Cenário | Resultado esperado |
|---------|-------------------|
| Requisição sem header Idempotency-Key | 400 com código `MISSING_IDEMPOTENCY_KEY` |
| Requisição com Content-Type: text/plain | 415 Unsupported Media Type |
| Valor decimal (ex.: 10.5) | 400 com mensagem sobre valor inteiro |
| Retry com mesma Idempotency-Key após sucesso | 200 com mesmo corpo (indicação de replay) |

### Impact
| Cenário | Resultado esperado |
|---------|-------------------|
| Retry após expiração de Idempotency-Key (TTL excedido) | Comportamento definido (reprocessar ou rejeitar) |
| Race condition: dois envios simultâneos, saldo cobre apenas um | Exatamente um sucesso, um erro 422; saldo consistente |
| Conta bloqueada entre envio original e retry com mesma key | Retorna resultado original (200) sem revalidar status |

### Log
| Cenário | Resultado esperado |
|---------|-------------------|
| Falha de envio (4xx) | Log com nível WARN, request_id, sender_id, amount, error_code |
| Falha de envio (5xx) | Log com nível ERROR, request_id, stack trace (não exposto ao cliente) |
| Envio bem-sucedido | Log com nível INFO, todos os IDs, duration_ms |
| Qualquer log | Não contém token, senha ou PII |

### UI
| Cenário | Resultado esperado |
|---------|-------------------|
| Timeout do App (12s sem resposta) | Exibe mensagem de timeout; botão "Tentar novamente" habilitado |
| Erro 422 (saldo insuficiente) | Campos preservados; saldo atualizado na tela |
| Duplo clique no botão "Enviar" | Botão desabilitado após primeiro clique; apenas uma requisição enviada |
| Erro 5xx | Mensagem exibida; formulário preservado; botão retry visível |

### Recovery
| Cenário | Resultado esperado |
|---------|-------------------|
| Timeout → retry automático (até 2x) | App reenvia com mesma Idempotency-Key; sucesso no retry |
| 3 falhas consecutivas | Mensagem "Se o problema persistir, entre em contato com o suporte" |
| Token expira durante preenchimento → envio → 401 | Redireciona para login; após re-login, formulário preservado |
| Painel web após envio no App | Botão "Atualizar" ou auto-refresh mostra saldo atualizado |

### Emotions
| Cenário | Resultado esperado |
|---------|-------------------|
| Erro 500 | Mensagem: "Tivemos um problema técnico. Seu saldo não foi alterado. Tente novamente em instantes." |
| Erro de timeout | Mensagem inclui "Seu saldo não foi alterado" e "Tente novamente em alguns segundos" |
| Saldo insuficiente | Mensagem oferece opções (ajustar valor ou verificar saldo) |
| Conta não ativa | Mensagem não alarmista, com canal de suporte |

---

## Conclusão

O requisito v2 de Envio de QualiPoints demonstra maturidade significativa nos aspectos de **atomicidade, idempotência e mapeamento de erros**. Os principais gaps FAILURE estão em:

1. **Resiliência operacional** (Functional/Log): falta definir comportamento em indisponibilidade, estrutura de logs e métricas
2. **Proteção financeira em edge cases** (Impact): TTL de idempotência e lock de saldo em concorrência precisam ser especificados
3. **Experiência emocional em falha** (Emotions/UI/Recovery): mensagens podem ser mais transparentes sobre integridade do saldo, e a UI precisa de especificação mais detalhada de recovery

A resolução dos gaps **críticos** (I2, I3, A3, L2) antes do desenvolvimento reduzirá significativamente o risco de inconsistência financeira e dificuldade de diagnóstico em produção.

---

*Análise gerada com a heurística FAILURE — Ben Simo. Uma lente abrangente para o teste de cenários de falha.*
