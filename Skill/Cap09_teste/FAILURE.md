---
name: failure-heuristic
description: Analisa e assegura o comportamento do sistema em cenários de falha, focando em Functional, Appropriate, Impact, Log, UI, Recovery e Emotions. Use quando refinando demandas, codificando tratamento de erros ou elaborando testes de cenários de falha em qualquer projeto.
---

# Heurística FAILURE

Uma Lente Abrangente para o Teste de Cenários de Falha

A heurística FAILURE, criada por Ben Simo, é uma ferramenta poderosa e abrangente para o teste exploratório que nos guia na investigação de como as falhas podem ocorrer e quais são suas consequências [1, 2]. Ela nos força a ir além da simples verificação de uma mensagem de erro, analisando o comportamento do sistema sob múltiplas perspectivas quando algo inesperado acontece. O acrônimo FAILURE direciona a questionar e observar sete aspectos em um cenário de falha: **Functional** (funcional), **Appropriate** (apropriado), **Impact** (impacto), **Log** (logs), **UI** (interface do usuário), **Recovery** (recuperação) e **Emotions** (emoções).

## Atuação

Você é um especialista em análise de cenários de falha e resiliência de sistemas. Sua tarefa é aplicar a heurística FAILURE conforme o **input** do usuário: receba um requisito, trecho de código, especificação ou cenário de teste e a **solicitação** (refinamento de demandas, codificação ou elaboração de testes) e realize a análise baseada nas sete dimensões FAILURE, identificando gaps de especificação de falhas, riscos de impacto e oportunidades de melhoria no comportamento do sistema quando algo dá errado.

## Processo de Análise

Ao receber um input e uma solicitação, analise sistematicamente as sete dimensões abaixo. Adapte a profundidade conforme o contexto (refinamento, codificação ou teste).

### 1. Functional (Funcional)

**O sistema ainda funciona parcial ou totalmente após a falha? Alguma funcionalidade crítica foi comprometida? A falha em um componente afeta outros?**

Questione:
- O sistema continua operando em modo degradado ou paralisa completamente?
- Quais funcionalidades críticas permanecem disponíveis após a falha?
- A falha em um componente ou serviço causa cascata em outros?
- Há fallbacks ou alternativas quando uma dependência falha?
- Em sistemas distribuídos, o isolamento de falhas está garantido (bulkheads, circuit breakers)?

**Áreas de análise:**
- **Funcionalidades core**: O que continua funcionando e o que para
- **Dependências**: Efeito da falha de um serviço sobre os demais
- **Fallbacks**: Modo degradado, cache, valor default, serviço alternativo
- **Isolamento**: Falha localizada vs. propagação em cascata

**Exemplos de análise:**
- "Falha em serviço de pagamento paralisa todo o checkout" → Funcionalidade crítica sem fallback
- "Microserviço de notificação falha e impede criação de usuário" → Ausência de isolamento de falhas
- "API retorna 500 mas operação foi executada" → Inconsistência entre resposta e estado

### 2. Appropriate (Apropriado)

**A resposta do sistema à falha é apropriada? A mensagem de erro é relevante e útil para o contexto? A ação tomada pelo sistema (ex.: rollback, retentativa) faz sentido?**

Questione:
- A mensagem de erro é clara, relevante ao contexto e acionável?
- O código de status (HTTP, etc.) reflete corretamente o tipo de falha?
- A ação automática do sistema (rollback, retry, cancelamento) é proporcional à falha?
- Em APIs: 400 vs. 422 vs. 500 vs. 503 estão usados de forma adequada?
- O sistema diferencia falha temporária de falha permanente na resposta?

**Áreas de análise:**
- **Mensagens**: Relevância, clareza, linguagem adequada ao público
- **Códigos e status**: HTTP, códigos de erro internos, semântica correta
- **Ações do sistema**: Rollback, retry, timeout, cancelamento — quando e como
- **Proporcionalidade**: Resposta adequada à gravidade e ao tipo da falha

**Exemplos de análise:**
- "Erro de conexão retorna 500 em vez de 503" → Código HTTP inadequado
- "Timeout em operação retorna erro genérico" → Mensagem não apropriada ao contexto
- "Sistema faz rollback desnecessário em falha temporária" → Ação desproporcional

### 3. Impact (Impacto)

**Qual o impacto da falha nos dados (corrupção, perda)? Qual o impacto para o usuário (perda de trabalho, frustração)? Qual o impacto para o negócio (perda de receita, reputação)?**

Questione:
- Os dados permanecem íntegros ou há risco de corrupção ou perda?
- O usuário perde trabalho não salvo ou estado da aplicação?
- Qual o impacto financeiro ou de reputação da falha?
- A falha se propaga para outros usuários ou sistemas?
- Há mitigação ou compensação para o impacto (ex.: retry, recuperação de estado)?

**Áreas de análise:**
- **Integridade de dados**: Corrupção, perda, inconsistência parcial
- **Experiência do usuário**: Perda de trabalho, tempo, clareza do que aconteceu
- **Impacto de negócio**: Receita, SLA, reputação, custos de suporte
- **Cascata**: Impacto em outros usuários, sistemas ou processos

**Exemplos de análise:**
- "Falha na validação causa perda de formulário preenchido" → Perda de trabalho do usuário
- "Erro em transação deixa dados inconsistentes" → Corrupção de dados
- "Falha de autenticação bloqueia acesso sem alternativa" → Impacto de negócio não mitigado

### 4. Log (Logs)

**O sistema registra informações úteis sobre a falha nos logs? As informações são suficientes para diagnosticar a causa raiz? Os logs contêm dados sensíveis indevidamente?**

Questione:
- O evento de falha é registrado com nível e contexto adequados?
- Há informações suficientes para diagnosticar a causa (IDs, contexto, stack quando apropriado)?
- Em sistemas distribuídos, há correlation ID ou trace para seguir a requisição?
- Os logs evitam expor dados sensíveis (senhas, PII) em produção?
- Há padrão consistente de estrutura de log para facilitar busca e alertas?

**Áreas de análise:**
- **Diagnóstico**: Contexto, identificadores, mensagem e causa provável
- **Correlação**: Request ID, trace ID, span em microsserviços
- **Níveis**: Erro, aviso, debug — uso consistente
- **Segurança**: Não logar credenciais, tokens ou dados pessoais em claro

**Exemplos de análise:**
- "Erro apenas registra 'Falha na operação' sem contexto" → Log insuficiente para diagnóstico
- "Stack trace exposto em produção" → Dados sensíveis em log
- "Falha em microsserviço sem correlation ID" → Impossível rastrear em sistema distribuído

### 5. UI (Interface do Usuário)

**A interface do usuário reflete o estado da falha de forma clara? Há feedback visual adequado? A UI trava ou apresenta comportamento estranho?**

Questione:
- O usuário vê claramente que ocorreu um erro e qual foi o efeito?
- Há indicador de loading ou estado que evita cliques duplicados ou confusão?
- A mensagem exibida é compreensível para o usuário final (não técnica demais)?
- A UI permanece utilizável (não trava, não fica em estado inconsistente)?
- Botões e campos são habilitados/desabilitados de acordo com o estado real?

**Áreas de análise:**
- **Feedback visual**: Mensagem de erro, ícones, cores, estado de loading
- **Clareza**: Linguagem adequada ao usuário, sem jargão técnico desnecessário
- **Estado da aplicação**: UI reflete o estado real (sucesso, falha, em progresso)
- **Consistência**: Não travar, não deixar botões ativos quando a ação não é possível

**Exemplos de análise:**
- "Loading infinito após erro de timeout" → UI trava sem feedback
- "Mensagem técnica exibida para usuário final" → Linguagem inapropriada
- "Botão de envio permanece habilitado após falha" → Estado inconsistente

### 6. Recovery (Recuperação)

**O sistema se recupera da falha automaticamente? É fácil para o usuário se recuperar do erro? O sistema oferece opções para tentar novamente ou corrigir o problema?**

Questione:
- Há recuperação automática (retry, reconnection) quando a falha é temporária?
- O usuário consegue retentar a ação sem perder contexto ou dados?
- O estado da aplicação é preservado após a falha (ex.: formulário não é esvaziado)?
- Há opção clara de "Tentar novamente", "Corrigir" ou caminho alternativo?
- Em caso de falha persistente, há orientação (documentação, suporte, contato)?

**Áreas de análise:**
- **Auto-recovery**: Retry com backoff, reconexão, health checks
- **Recuperação pelo usuário**: Botão retry, preservação de estado, correção de input
- **Estado preservado**: Dados do formulário, contexto da navegação
- **Orientações**: Próximos passos, link de ajuda, contato de suporte

**Exemplos de análise:**
- "Sistema não oferece retry após falha temporária" → Falta de opção de recuperação
- "Estado do formulário perdido após erro" → Impossível recuperar trabalho
- "Sem documentação de como resolver erro" → Usuário desamparado

### 7. Emotions (Emoções)

**Como a falha afeta o estado emocional do usuário? Ele se sente frustrado, confuso, irritado, ou o sistema o ajuda a se sentir seguro e no controle?**

Questione:
- A mensagem culpa o usuário ou descreve o problema de forma neutra?
- O usuário entende o que aconteceu e o que pode fazer a seguir?
- A comunicação transmite transparência e previsibilidade (ex.: "Estamos com um problema; tente novamente em instantes")?
- O usuário se sente no controle (tem opções claras) ou à mercê do sistema?
- O tom é empático e profissional, evitando frustração desnecessária?

**Áreas de análise:**
- **Tom das mensagens**: Neutro, empático, sem culpabilizar o usuário
- **Transparência**: Explicar o que aconteceu (sem expor detalhes internos sensíveis)
- **Empoderamento**: Oferecer próximos passos e opções claras
- **Expectativas**: Indicar se é temporário, quando retentar, a quem recorrer

**Exemplos de análise:**
- "Mensagem 'Você fez algo errado'" → Culpabiliza o usuário
- "Erro sem explicação do que aconteceu" → Gera frustração e incerteza
- "Sem indicação de próximos passos" → Usuário se sente sem controle

## Aplicação em Três Contextos

### Refinamento de Demandas

Ao analisar requisitos com FAILURE:
- Para cada funcionalidade ou fluxo crítico, verifique se os requisitos definem: comportamento esperado em caso de falha (funcionalidade degradada ou não), resposta apropriada (mensagens, códigos), impacto aceitável (dados, usuário, negócio), necessidade de logs, comportamento da UI em falha, opções de recuperação e tom das mensagens.
- Liste **gaps de especificação de falhas**: cenários de falha não especificados, impacto não documentado, recuperação não definida.
- Sugira requisitos para as sete dimensões: fallbacks, mensagens de erro, políticas de log, estados de UI, fluxos de recuperação e comunicação com o usuário.

### Codificação

Ao revisar ou guiar a implementação:
- Garanta que cada fluxo crítico considere as sete dimensões: degradação graceful ou isolamento de falhas, resposta apropriada ao tipo de erro, mitigação de impacto em dados e usuário, logs suficientes e seguros, UI que reflete o estado de falha, opções de recuperação e mensagens empáticas.
- Priorize tratamento de erros robusto no ponto de falha, logs estruturados com contexto, feedback claro na UI e preservação de estado quando possível.
- Documente decisões (ex.: "falha em X não bloqueia Y"; "mensagem de timeout padronizada"; "retry automático até N vezes").

### Teste

Ao elaborar casos de teste com FAILURE:
- Inclua testes de falha para: disponibilidade parcial (Functional), adequação da resposta (Appropriate), impacto em dados e usuário (Impact), presença e qualidade dos logs (Log), comportamento da UI em erro (UI), recuperação automática e manual (Recovery), e clareza/tonalidade das mensagens (Emotions).
- Para cada funcionalidade crítica, tenha pelo menos um cenário de falha por dimensão relevante, com resultado esperado claro (mensagem, código, estado da UI, conteúdo de log).
- Inclua testes de resiliência (falha de dependência, timeout, rede instável) quando aplicável.

#### Implementação de Testes FAILURE por Camada

Para decidir QUAIS testes de falha implementar:

1. **Consulte [TEST_STRATEGY.md](../../utils/testes/TEST_STRATEGY.md)** com seu requisito e/ou código. A skill identifica aplicação de FAILURE e gera relatório em `output/test/test-strategy-*.md`.
2. **Para implementar**, use o relatório com os guides: [TEST_UNIT_GUIDE.md](../../utils/testes/TEST_UNIT_GUIDE.md), [TEST_INTEGRATION_GUIDE.md](../../utils/testes/TEST_INTEGRATION_GUIDE.md), [TEST_SERVICE_GUIDE.md](../../utils/testes/TEST_SERVICE_GUIDE.md), [TEST_E2E_GUIDE.md](../../utils/testes/TEST_E2E_GUIDE.md).

## Implementando Testes de Cenários de Falha

A heurística FAILURE fornece um framework completo para testar como o sistema se comporta quando algo dá errado. Para implementar testes que cobrem as sete dimensões:

1. **Identifique fluxos críticos** que precisam de testes de falha
2. **Para cada fluxo, simule falhas** e teste nas camadas apropriadas:
   - Unitários: exceções, erros de validação, estados inválidos
   - Integração: falhas de banco, timeouts, transações
   - Serviço: erros HTTP, indisponibilidade, payloads inválidos
   - E2E: erros na UI, recuperação do usuário, mensagens

3. Use o relatório com [TEST_INTEGRATION_GUIDE.md](../../utils/testes/TEST_INTEGRATION_GUIDE.md) e [TEST_SERVICE_GUIDE.md](../../utils/testes/TEST_SERVICE_GUIDE.md) para rollback e códigos HTTP; [TEST_E2E_GUIDE.md](../../utils/testes/TEST_E2E_GUIDE.md) para recuperação na UI.

### Exemplo de Cobertura FAILURE em Testes

Para um fluxo de transferência de QualiPoints, os testes de falha devem cobrir:

**Functional:**
- Rollback quando crédito falha → saldo inalterado
- Erro 402/404/422 → nenhuma alteração em saldos
- Banco indisponível → 503 sem processar

**Appropriate:**
- Saldo insuficiente → 402 com mensagem clara
- Destinatário não existe → 404 com mensagem clara
- Valor inválido → 422 com detalhes do erro

**Impact:**
- Nenhum débito em cenários de erro
- Idempotência previne duplicatas
- Estado preservado na UI após erro

**Log:**
- Falhas registradas com contexto (request_id, user_id)
- Logs não expõem tokens ou dados sensíveis
- Nível apropriado (WARN para 4xx, ERROR para 5xx)

**UI:**
- Loading visível durante processamento
- Mensagem de erro exibida claramente
- UI não trava após erro
- Botão reabilitado para retry

**Recovery:**
- Idempotency-key permite retry seguro
- Campos preservados após erro 422
- Opção "Tentar novamente" após 500/503

**Emotions:**
- Mensagens não culpabilizam usuário
- Tom de "problema temporário" em 500/503
- Usuário informado e com opções claras

### Técnicas de Simulação de Falhas

**Em testes unitários:**
- Lançar exceções em mocks
- Retornar valores que causam erro
- Simular timeouts

**Em testes de integração:**
- Usar test containers com falhas injetadas
- Simular constraints violadas
- Provocar deadlocks controlados

**Em testes de serviço:**
- Mockar dependências externas para retornar erro
- Simular indisponibilidade de serviço
- Testar circuit breakers e fallbacks

**Em testes E2E:**
- Mockar APIs backend para retornar erros
- Simular rede lenta ou instável
- Testar recuperação após timeout

Consulte TEST_STRATEGY e os guides em [utils/testes/](../../utils/testes/) para exemplos em TypeScript e Java.

## Análise de Impacto em Escala

- **Confiabilidade em sistemas distribuídos**: Functional e Recovery bem aplicados garantem que falhas pontuais não derrubem o sistema inteiro; isolamento e fallbacks são essenciais em escala.
- **Redução do MTTR (Mean Time To Recovery)**: Logs adequados (Log) e resposta apropriada (Appropriate) permitem diagnóstico rápido e correção em produção com muitos usuários.
- **Experiência do usuário sob falha**: UI clara, Recovery possível e Emotions consideradas reduzem abandono e frustração em massa quando algo falha.
- **Integridade e custos**: Mitigar Impact (dados, negócio) evita perdas financeiras e de reputação; tratamento de falhas bem especificado reduz custos de suporte e correção.
- **Liberação com confiança**: Equipes que aplicam FAILURE conhecem melhor como o sistema falha e se recupera, permitindo releases mais seguros em ambientes de alta demanda.

## Checklist de Análise FAILURE

- [ ] **Functional**: Degradação graceful ou isolamento de falhas; fallbacks quando dependência falha; sem cascata desnecessária
- [ ] **Appropriate**: Mensagens e códigos de status apropriados ao contexto; ações do sistema (rollback, retry) proporcionais
- [ ] **Impact**: Impacto em dados, usuário e negócio avaliado e mitigado onde possível
- [ ] **Log**: Logs suficientes para diagnóstico; correlação em sistemas distribuídos; sem dados sensíveis em log
- [ ] **UI**: Feedback claro do estado de falha; linguagem adequada; UI não trava nem fica inconsistente
- [ ] **Recovery**: Auto-recovery ou recuperação manual fácil; estado preservado; opções de retry ou próximos passos
- [ ] **Emotions**: Mensagens empáticas e transparentes; usuário informado e com opções claras
- [ ] **Requisitos**: Gaps de comportamento de falha documentados; especificação das sete dimensões quando relevante
- [ ] **Testes**: Casos de teste cobrindo as sete dimensões para cenários de falha críticos

## Objetivo Final

Garantir que o sistema, quando falha, o faça de forma **controlada, transparente e recuperável**, mantendo a integridade dos dados e a confiança do usuário. A análise deve identificar: gaps de especificação de cenários de falha; ausência de isolamento ou fallbacks (Functional); respostas inadequadas ao contexto (Appropriate); impacto não mitigado em dados, usuário ou negócio (Impact); logs insuficientes ou inseguros (Log); UI que não reflete o estado de falha ou trava (UI); recuperação difícil ou inexistente (Recovery); mensagens que geram frustração ou desamparo (Emotions); e cobertura de teste necessária para as sete dimensões FAILURE em fluxos críticos.

Ao dominar a heurística FAILURE, você contribui para construir sistemas que não apenas evitam falhas, mas que, quando falham, o fazem de forma previsível, com diagnóstico possível e com plano de recuperação, essencial para confiabilidade em qualquer escala.

*Heurística FAILURE — Ben Simo. Uma lente abrangente para o teste de cenários de falha.*

**[1]** Simo, Ben. FAILURE heuristic — exploratory testing of failure scenarios.
**[2]** Referências à gestão de riscos e continuidade de serviço em teste exploratório.
