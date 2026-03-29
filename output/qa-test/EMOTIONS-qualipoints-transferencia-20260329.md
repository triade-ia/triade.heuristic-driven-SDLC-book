# Relatório EMOTIONS: Envio de QualiPoints entre Usuários

**Data**: 2026-03-29
**Requisito**: Envio de QualiPoints entre usuários (v2)
**Heurística**: EMOTIONS (Alegria, Tristeza, Raiva, Medo, Nojinho, Surpresa, Confiança, Antecipação, Desconfiança)
**Contexto**: Carteira digital QualiPoints — transferência síncrona via App mobile + consulta painel web

---

## Resumo da Funcionalidade

Usuário autenticado envia QualiPoints (1–10.000, inteiro) para outro usuário ativo. Operação síncrona com idempotência, atômica (débito + crédito + registro), feedback em 3 estados (Processando / Concluído / Falhou). Lista de últimas 10 transações. Painel web somente consulta.

---

## Análise por Emoção

### 1. Alegria (Joy)

**Onde o sistema proporciona satisfação ao usuário?**

| Aspecto | Análise | Oportunidade/Gap |
|---------|---------|------------------|
| Eficiência | Envio direto: destinatário + valor + confirmar. Fluxo curto e objetivo | Reforçar: poucos passos = satisfação |
| Feedback positivo | Requisito prevê estado "Concluído" com confirmação de sucesso | **Gap**: não detalha mensagem de sucesso. Sugestão: "Pronto! 500 QualiPoints enviados para user123" com resumo visual |
| Atualização imediata | Saldo e lista de recentes atualizam após envio | Gera sensação de controle e conclusão |
| Simplicidade | Body da API tem apenas `recipientId` e `amount` | Boa escolha para MVP — menos campos = menos atrito |

**Cenários de teste (Alegria):**
- EMOT-ALG-01: Após envio bem-sucedido, usuário vê mensagem clara de sucesso com valor e destinatário — **P0**
- EMOT-ALG-02: Saldo atualiza imediatamente na tela após confirmação — **P0**
- EMOT-ALG-03: Transação aparece no topo da lista de recentes — **P1**
- EMOT-ALG-04: Fluxo completo (abrir tela → enviar → ver confirmação) requer no máximo 3 interações — **P1**

---

### 2. Tristeza (Sadness)

**Onde o sistema pode levar à decepção ou perda?**

| Aspecto | Análise | Oportunidade/Gap |
|---------|---------|------------------|
| Perda de dados no formulário | **Gap**: requisito não menciona preservação de campos em caso de erro. Se a API retorna 422 (saldo insuficiente), o formulário mantém destinatário e valor preenchidos? | Crítico: evitar que o usuário redigite tudo |
| Tom das mensagens de erro | Mensagens mapeadas (seção 11.2) são informativas, mas algumas são frias: "Destinatário não encontrado. Verifique o ID ou o status da conta." | Sugestão: tom mais empático — "Não encontramos esse destinatário. Verifique o ID e tente novamente." |
| Saldo insuficiente | Mensagem prevista: "Saldo insuficiente. Seu saldo atual não permite este envio." | **Gap**: não mostra o saldo atual ao usuário na mensagem do App (API opcionalmente retorna `currentBalance`) — decisão de exibir ou não impacta a tristeza |
| Timeout / falha de rede | Usuário não sabe se o envio foi processado ou não | **Gap**: cenário de incerteza máxima. Mensagem "A operação demorou mais que o esperado. Tente novamente." não tranquiliza sobre o débito |

**Cenários de teste (Tristeza):**
- EMOT-TRI-01: Após erro 422 (saldo insuficiente), campos do formulário permanecem preenchidos — **P0**
- EMOT-TRI-02: Após erro de rede, mensagem informa que o saldo NÃO foi debitado (ou orienta verificar) — **P0**
- EMOT-TRI-03: Mensagem de saldo insuficiente inclui saldo atual para o usuário entender a diferença — **P1**
- EMOT-TRI-04: Tom das mensagens de erro é empático e orientado a ação, não técnico — **P1**

---

### 3. Raiva (Anger)

**Onde o sistema causa frustração ou sensação de desamparo?**

| Aspecto | Análise | Oportunidade/Gap |
|---------|---------|------------------|
| Envio para si mesmo (409) | Mensagem prevista: "Não é permitido enviar QualiPoints para a própria conta." | Adequada, mas **Gap**: o App poderia impedir antes de enviar (validação client-side) para evitar frustração de esperar a resposta da API |
| Conta bloqueada (403) | Mensagem: "Sua conta não está ativa para envio. Entre em contato com o suporte." | **Gap**: não há link direto para suporte. Usuário fica sem saber o que fazer |
| Sessão expirada (401) | Mensagem: "Sessão expirada. Faça login novamente." | **Gap**: se o usuário preencheu o formulário e a sessão expirou, perde os dados? Fluxo de re-login deveria preservar estado |
| Timeout (10s) | 10 segundos sem feedback é muito tempo. Requisito prevê loading, mas **Gap**: não menciona possibilidade de cancelar | Usuário preso esperando sem opção de cancelar → raiva |
| Erro interno (5xx) | Mensagem genérica: "Ocorreu um erro. Tente novamente em instantes." | Frustrante se repetido. **Gap**: não há orientação após múltiplas falhas |

**Cenários de teste (Raiva):**
- EMOT-RAI-01: Tentar enviar para si mesmo mostra erro ANTES de chamar a API (validação local) — **P1**
- EMOT-RAI-02: Erro 403 (conta bloqueada) inclui canal de contato com suporte (link, telefone) — **P1**
- EMOT-RAI-03: Se sessão expira durante preenchimento, após re-login os dados do formulário são restaurados — **P1**
- EMOT-RAI-04: Durante loading (processando), há opção de cancelar ou indicador de tempo restante — **P2**
- EMOT-RAI-05: Após 3 erros 5xx consecutivos, mensagem orienta canal alternativo de suporte — **P2**

---

### 4. Medo (Fear)

**Onde o sistema gera insegurança ou ansiedade?**

| Aspecto | Análise | Oportunidade/Gap |
|---------|---------|------------------|
| Ação irreversível | Enviar QualiPoints é irreversível (requisito não menciona estorno) | **Gap crítico**: não há tela de confirmação/resumo antes do envio. Usuário pode enviar valor errado ou para destinatário errado sem chance de revisão |
| Valor alto | Até 10.000 QualiPoints por transação. Erro de digitação pode custar caro | **Gap**: não há confirmação adicional para valores altos |
| Identificação do destinatário | Apenas ID do usuário. Usuário pode digitar errado | **Gap**: não há exibição do nome do destinatário para confirmação antes do envio |
| Segurança da transação | HTTPS + token + idempotência — bem coberto tecnicamente | Positivo, mas **Gap**: o App transmite essa segurança visualmente ao usuário? (ex.: ícone de cadeado, indicação de conexão segura) |
| Duplo débito | Idempotency-Key protege contra duplo débito | Requisito bem desenhado, mas **Gap**: o usuário não sabe disso. Mensagem de retry deveria tranquilizar: "Pode tentar novamente com segurança, seu saldo não será debitado duas vezes." |

**Cenários de teste (Medo):**
- EMOT-MED-01: Antes de confirmar envio, tela de resumo exibe: destinatário (nome + ID), valor, saldo após envio — **P0**
- EMOT-MED-02: Para valores acima de 5.000, confirmação adicional (ex.: "Você está enviando 8.000 QualiPoints. Confirma?") — **P1**
- EMOT-MED-03: Ao digitar ID do destinatário, sistema exibe nome para validação visual — **P0**
- EMOT-MED-04: Mensagem de retry após timeout inclui garantia de que não haverá débito duplo — **P1**
- EMOT-MED-05: Indicador visual de conexão segura visível durante o fluxo de envio — **P2**

---

### 5. Nojinho (Disgust)

**Onde o sistema apresenta elementos desagradáveis ou desorganizados?**

| Aspecto | Análise | Oportunidade/Gap |
|---------|---------|------------------|
| Consistência visual | Requisito não detalha UI, mas menciona App + painel web | **Gap**: garantir consistência visual entre App e painel (mesmos termos, mesmos status, mesma ordem de informações) |
| Lista de transações | Últimas 10, sem filtros | Minimalista para MVP — OK. Mas **Gap**: se o usuário tem muitas transações, ver apenas 10 sem opção de busca pode ser frustrante |
| Mensagens de erro | Algumas misturam termos técnicos: "Idempotency-Key", "408/504", "INVALID_AMOUNT" | **Gap**: códigos internos da API (INVALID_AMOUNT, SELF_TRANSFER_NOT_ALLOWED) nunca devem aparecer no App — apenas a mensagem traduzida |
| Campos do formulário | Apenas recipientId + amount | **Gap**: "recipientId" como label é técnico. Na UI deve ser "Destinatário" ou "ID do destinatário" |

**Cenários de teste (Nojinho):**
- EMOT-NOJ-01: Nenhum código técnico da API (ex.: INVALID_AMOUNT) é exibido ao usuário final — **P0**
- EMOT-NOJ-02: Terminologia consistente entre App e painel web (mesmo nome para status, campos, valores) — **P1**
- EMOT-NOJ-03: Labels dos campos usam linguagem do usuário, não termos técnicos da API — **P1**
- EMOT-NOJ-04: Formatação de valores com separador de milhar (ex.: 10.000 e não 10000) — **P2**

---

### 6. Surpresa (Surprise)

**O que pode pegar o usuário de surpresa?**

| Aspecto | Análise | Oportunidade/Gap |
|---------|---------|------------------|
| Surpresa positiva | **Gap**: requisito não prevê nenhum "delighter". Sugestão: animação sutil de sucesso, ou mostrar "Você enviou X QualiPoints este mês" como insight | Oportunidade de encantar |
| Surpresa negativa — destinatário inativo | Usuário digita ID correto mas destinatário foi bloqueado → 404 | Pode surpreender negativamente. Mensagem deveria contextualizar sem expor status da conta do outro |
| Surpresa negativa — limite por transação | Usuário não sabe que o máximo é 10.000 até tentar | **Gap**: limite deveria ser visível no formulário ANTES do envio |
| Comportamento do painel web | Painel é somente consulta, mas usuário pode esperar enviar por lá | **Gap**: requisito não menciona comunicação clara de que o envio é apenas pelo App |

**Cenários de teste (Surpresa):**
- EMOT-SUR-01: Limite máximo (10.000) é visível no campo de valor antes do usuário preencher — **P1**
- EMOT-SUR-02: Painel web exibe mensagem clara de que envio é feito apenas pelo App — **P1**
- EMOT-SUR-03: Após envio bem-sucedido, feedback visual celebratório (animação, cor, ícone) — **P2**
- EMOT-SUR-04: Destinatário inativo retorna mensagem genérica que não expõe status da conta alheia — **P1**

---

### 7. Confiança (Trust)

**O usuário sente segurança e previsibilidade?**

| Aspecto | Análise | Oportunidade/Gap |
|---------|---------|------------------|
| Consistência de dados | Saldo e lista vêm da mesma fonte de verdade (banco relacional) | Ponto forte do requisito |
| Atomicidade | Débito + crédito + registro atômicos. Promessa cumprida | Gera confiança — transação não fica "pela metade" |
| Consistência App × Painel | Requisito menciona que painel reflete dados após refresh | **Gap**: pode haver delay se o usuário abre o painel logo após enviar no App. Expectativa de tempo real vs. polling |
| Histórico de transações | Lista de recentes mostra o que aconteceu | Gera confiança. Mas **Gap**: sem detalhes da transação (apenas últimas 10, sem clique para expandir) |
| Idempotência | Retry seguro com mesma chave | Excelente para confiança, mas invisível ao usuário. O App precisa comunicar isso |

**Cenários de teste (Confiança):**
- EMOT-CON-01: Saldo exibido no App é idêntico ao exibido no painel web após refresh — **P0**
- EMOT-CON-02: Transação aparece na lista de recentes tanto do remetente quanto do destinatário — **P0**
- EMOT-CON-03: Após retry com mesma Idempotency-Key, saldo é debitado apenas uma vez — **P0**
- EMOT-CON-04: Status "Concluído" no App corresponde a registro real no banco (transação persistida) — **P0**
- EMOT-CON-05: Lista de transações mostra dados consistentes (valor, data, destinatário) entre visualizações — **P1**

---

### 8. Antecipação (Anticipation)

**O usuário sente expectativa e sabe o que esperar?**

| Aspecto | Análise | Oportunidade/Gap |
|---------|---------|------------------|
| Estado "Processando" | Requisito prevê indicador de carregamento + desabilitar novo envio | Bom — gerencia a expectativa |
| Timeout de 10s | **Gap**: 10 segundos é longo. Usuário espera resposta em 2-3s. Se demorar mais, precisa de feedback progressivo (ex.: "Estamos processando, aguarde...") |
| Próximos passos | Após sucesso: saldo atualiza, lista atualiza. Claro | Após erro: mensagens orientam ação. Adequado |
| Feedback de progresso | **Gap**: não há indicação de progresso granular. Para operação síncrona de até 10s, um spinner simples pode não ser suficiente |
| Gestão de expectativa no retry | Mensagem de timeout sugere "Tente novamente" | **Gap**: não informa se é seguro tentar novamente (medo de débito duplo). Deve reforçar idempotência |

**Cenários de teste (Antecipação):**
- EMOT-ANT-01: Indicador de loading aparece imediatamente após clicar "Enviar" — **P0**
- EMOT-ANT-02: Botão de envio é desabilitado durante processamento (evita duplo clique) — **P0**
- EMOT-ANT-03: Se processamento excede 3 segundos, mensagem adicional aparece: "Estamos processando sua transferência..." — **P1**
- EMOT-ANT-04: Após timeout, mensagem inclui orientação de que é seguro tentar novamente — **P1**
- EMOT-ANT-05: Após sucesso, próximo passo é claro (voltar ao saldo, ver extrato, novo envio) — **P2**

---

### 9. Desconfiança (Distrust)

**O que pode gerar ceticismo ou dúvida?**

| Aspecto | Análise | Oportunidade/Gap |
|---------|---------|------------------|
| Saldo inconsistente | Se App mostra saldo diferente do painel web | **Gap**: polling/refresh no painel pode causar defasagem. Usuário verá valores diferentes e desconfiará |
| Timeout sem clareza | Requisito diz "Pode tentar novamente" mas não garante explicitamente que o dinheiro não saiu | **Gap crítico**: o maior gatilho de desconfiança é "meu dinheiro saiu e não chegou?" |
| Mensagens genéricas | "Ocorreu um erro. Tente novamente em instantes." para 5xx | Mensagem vaga gera desconfiança. Sem detalhes, usuário pensa "o sistema não funciona" |
| Falta de comprovante | MVP não tem comprovante (PDF, e-mail) | **Gap**: sem comprovante, o usuário só tem a lista de recentes como "prova". Se a lista mudar ou não carregar, não há evidência do envio |
| Destinatário sem nome | Usuário envia para um ID, não vê nome | Gera dúvida: "enviei para a pessoa certa?" |

**Cenários de teste (Desconfiança):**
- EMOT-DES-01: Após timeout, consultar lista de recentes mostra se a transação foi ou não processada — **P0**
- EMOT-DES-02: Saldo no App e saldo no painel convergem em até 5 segundos após atualização — **P1**
- EMOT-DES-03: Transação concluída permanece visível na lista de recentes (não desaparece) — **P0**
- EMOT-DES-04: Ao informar recipientId, sistema exibe nome/identificação do destinatário para confirmação — **P0**
- EMOT-DES-05: Mensagens de erro nunca são genéricas — sempre indicam o que aconteceu e o que fazer — **P1**

---

## Resumo de Gaps Identificados

| # | Gap | Emoção principal | Prioridade | Recomendação |
|---|-----|-----------------|------------|--------------|
| 1 | Sem tela de confirmação/resumo antes do envio | Medo | **P0** | Adicionar tela: destinatário (nome + ID), valor, saldo após envio |
| 2 | Destinatário identificado apenas por ID (sem nome) | Medo, Desconfiança | **P0** | Exibir nome do destinatário ao digitar/selecionar ID |
| 3 | Formulário não preserva dados após erro | Tristeza | **P0** | Manter campos preenchidos após qualquer erro da API |
| 4 | Timeout sem clareza sobre débito | Desconfiança, Medo | **P0** | Mensagem de retry deve garantir: "Seu saldo não será debitado duas vezes" |
| 5 | Limite de 10.000 não visível antes do preenchimento | Surpresa | **P1** | Mostrar limite no campo ou como dica (placeholder/tooltip) |
| 6 | Conta bloqueada sem canal de suporte | Raiva | **P1** | Incluir link/telefone de suporte na mensagem 403 |
| 7 | Mensagens com tom técnico/frio | Tristeza, Nojinho | **P1** | Revisar tom para empático e orientado a ação |
| 8 | Sem feedback progressivo para espera > 3s | Antecipação | **P1** | Mensagem adicional após 3s de loading |
| 9 | Sem comprovante no MVP | Desconfiança | **P2** | Documentar como limitação conhecida; considerar "resumo compartilhável" como alternativa leve |
| 10 | Painel web não comunica que envio é só pelo App | Surpresa | **P1** | Adicionar aviso claro no painel |

---

## Consolidação de Cenários de Teste

### Prioridade P0 (Crítico) — 12 cenários

| ID | Cenário | Emoção |
|----|---------|--------|
| EMOT-ALG-01 | Mensagem de sucesso clara com valor e destinatário | Alegria |
| EMOT-ALG-02 | Saldo atualiza imediatamente após confirmação | Alegria |
| EMOT-TRI-01 | Campos preservados após erro 422 | Tristeza |
| EMOT-TRI-02 | Mensagem após erro de rede informa sobre estado do débito | Tristeza |
| EMOT-MED-01 | Tela de resumo antes de confirmar envio | Medo |
| EMOT-MED-03 | Sistema exibe nome do destinatário ao digitar ID | Medo |
| EMOT-NOJ-01 | Nenhum código técnico exposto ao usuário | Nojinho |
| EMOT-CON-01 | Saldo idêntico entre App e painel após refresh | Confiança |
| EMOT-CON-02 | Transação visível na lista de ambos (remetente e destinatário) | Confiança |
| EMOT-CON-03 | Retry com mesma Idempotency-Key debita apenas uma vez | Confiança |
| EMOT-CON-04 | Status "Concluído" corresponde a registro real no banco | Confiança |
| EMOT-ANT-01 | Loading aparece imediatamente ao clicar Enviar | Antecipação |
| EMOT-ANT-02 | Botão desabilitado durante processamento | Antecipação |
| EMOT-DES-01 | Após timeout, lista de recentes mostra estado real da transação | Desconfiança |
| EMOT-DES-03 | Transação concluída permanece visível na lista | Desconfiança |
| EMOT-DES-04 | Nome do destinatário exibido para confirmação | Desconfiança |

### Prioridade P1 (Importante) — 14 cenários

| ID | Cenário | Emoção |
|----|---------|--------|
| EMOT-ALG-03 | Transação no topo da lista de recentes | Alegria |
| EMOT-ALG-04 | Fluxo completo em no máximo 3 interações | Alegria |
| EMOT-TRI-03 | Mensagem de saldo insuficiente inclui saldo atual | Tristeza |
| EMOT-TRI-04 | Tom empático e orientado a ação nas mensagens | Tristeza |
| EMOT-RAI-01 | Validação local impede envio para si mesmo antes da API | Raiva |
| EMOT-RAI-02 | Erro 403 inclui canal de suporte | Raiva |
| EMOT-RAI-03 | Re-login preserva dados do formulário | Raiva |
| EMOT-MED-02 | Confirmação adicional para valores acima de 5.000 | Medo |
| EMOT-MED-04 | Retry garante ausência de débito duplo na mensagem | Medo |
| EMOT-NOJ-02 | Terminologia consistente App × painel | Nojinho |
| EMOT-NOJ-03 | Labels em linguagem do usuário | Nojinho |
| EMOT-SUR-01 | Limite máximo visível no campo de valor | Surpresa |
| EMOT-SUR-02 | Painel web avisa que envio é só pelo App | Surpresa |
| EMOT-SUR-04 | Destinatário inativo — mensagem genérica (não expõe status) | Surpresa |
| EMOT-CON-05 | Dados consistentes entre visualizações da lista | Confiança |
| EMOT-ANT-03 | Mensagem progressiva se loading > 3s | Antecipação |
| EMOT-ANT-04 | Orientação de retry seguro após timeout | Antecipação |
| EMOT-DES-02 | Saldo converge entre App e painel em até 5s | Desconfiança |
| EMOT-DES-05 | Mensagens de erro sempre indicam causa e próximo passo | Desconfiança |

### Prioridade P2 (Desejável) — 5 cenários

| ID | Cenário | Emoção |
|----|---------|--------|
| EMOT-RAI-04 | Opção de cancelar durante loading | Raiva |
| EMOT-RAI-05 | Após 3 erros 5xx, orienta canal alternativo | Raiva |
| EMOT-MED-05 | Indicador visual de conexão segura | Medo |
| EMOT-NOJ-04 | Formatação de valores com separador de milhar | Nojinho |
| EMOT-SUR-03 | Feedback visual celebratório após sucesso | Surpresa |
| EMOT-ANT-05 | Próximos passos claros após sucesso | Antecipação |

---

## Checklist EMOTIONS — QualiPoints

- [x] **Alegria**: Fluxo curto, feedback de sucesso previsto, saldo atualiza — gaps em mensagem detalhada de sucesso
- [x] **Tristeza**: Cenários de perda de dados e tom das mensagens analisados — gaps em preservação de formulário e empatia
- [x] **Raiva**: Bloqueios identificados (conta bloqueada, sessão expirada, timeout sem cancelar) — gaps em suporte e recuperação
- [x] **Medo**: Ação irreversível sem confirmação é o gap mais crítico — análise de segurança e identificação do destinatário
- [x] **Nojinho**: Códigos técnicos e terminologia inconsistente identificados — gaps em linguagem e formatação
- [x] **Surpresa**: Limites ocultos e painel sem comunicação clara — gaps em visibilidade de regras
- [x] **Confiança**: Atomicidade e idempotência são pontos fortes — gaps em consistência App/painel e comprovante
- [x] **Antecipação**: Loading e desabilitar botão previstos — gaps em feedback progressivo e orientação de retry
- [x] **Desconfiança**: Timeout sem clareza sobre débito é o maior risco — gaps em transparência e evidência de transação

---

## Próximos Passos

1. **Prioridade imediata**: Tratar gaps P0 no requisito — especialmente tela de confirmação, exibição do nome do destinatário e preservação de formulário
2. **Implementar testes de componente**: Usar este relatório com [TEST_COMPONENT_GUIDE.md](../../docs/guides/TEST_COMPONENT_GUIDE.md) para cobrir cenários de UI (formulário, loading, erro, sucesso, acessibilidade)
3. **Implementar testes E2E**: Usar com [TEST_E2E_GUIDE.md](../../docs/guides/TEST_E2E_GUIDE.md) para fluxos completos (envio → confirmação → lista de recentes)
4. **Revisar mensagens**: Aplicar tom empático e orientado a ação em todas as mensagens de erro mapeadas na seção 11.2 do requisito

---

*Análise realizada com a Heurística EMOTIONS — Priscila Caimi e Jonatas Martins.*
