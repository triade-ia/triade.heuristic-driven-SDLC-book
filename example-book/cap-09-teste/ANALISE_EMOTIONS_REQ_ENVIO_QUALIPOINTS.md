---
name: analise-emotions-qualipoints
description: Análise do requisito "Envio de QualiPoints entre usuários" sob a lente das 9 emoções. Identifica oportunidades de satisfação e pontos de atrito emocional no fluxo de transferência de pontos.
---

# Análise EMOTIONS: Requisito Envio de QualiPoints entre Usuários

**Requisito analisado:** REQ_INICIAL_V2.md — Envio de QualiPoints entre usuários  
**Heurística aplicada:** EMOTIONS (Alegria, Tristeza, Raiva, Medo, Nojinho, Surpresa, Confiança, Antecipação, Desconfiança)  
**Data da análise:** 21 de fevereiro de 2026  
**Tipo de análise:** Refinamento de demandas + Teste  

---

## 1. ALEGRIA (Joy) — Satisfação, eficiência e momentos de "uau"

### Análise do requisito

**Pontos positivos identificados:**
- ✅ **Operação síncrona com feedback imediato**: o usuário recebe confirmação na hora (200 OK); não aguarda processamento assíncrono longo. Isso gera alívio e certeza.
- ✅ **Fluxo direto no App**: envio em poucos passos (inserir ID + valor + enviar); reduz fricção.
- ✅ **Idempotência implementada**: usuário pode retry sem medo de débito duplo; reduz ansiedade em caso de falha de rede.
- ✅ **Atualização automática**: após sucesso, saldo e lista de transações são atualizados no App; feedback visual de conclusão.

**Gaps identificados (oportunidades):**
- ❌ **Falta de celebração/momentos de "uau"**: mensagens genéricas ("Envio concluído"). Oportunidade: mensagens positivas, ícones de sucesso, animação suave (confete sutil?), ou feedback como "João recebeu seus QualiPoints! 🎉".
- ❌ **Sem atalhos ou sugestões inteligentes**: não há menção a favoritos, contatos frequentes ou sugestões de quem receber. Oportunidade: "Enviar para João novamente?" ou "Contatos favoritos".
- ❌ **Sem recompensa ou "bonus"**: não há menção a incentivos para envios (ex.: cashback, bônus por referência). Escopo MVP, mas relevante para engajamento.
- ❌ **Falta de confirmação visual de atualização**: o painel web requer "refresh manual"; não há indicador de que os dados mudaram. Oportunidade: WebSocket ou polling com notificação visual.

### Recomendações para Alegria

1. **Feedback positivo destacado**: mensagem de sucesso com nome do destinatário e valor, não genérica. Ex.: "✓ 100 QualiPoints enviados para João com sucesso!"
2. **Animação/celebração sutil**: transição suave ou ícone animado ao confirmar (não excessivo; manter profissionalismo).
3. **Atualização visível de saldo**: mostrar novo saldo em tempo real; barra de mudança de saldo (antes → depois).
4. **Histórico resumido**: após envio, exibir a transação logo na lista (topo), com destaque visual temporário.
5. **(Backlog) Contatos favoritos**: permitir marcar contatos para envios rápidos.

---

## 2. TRISTEZA (Sadness) — Cenários de perda, desilusão, tom empático

### Análise do requisito

**Pontos positivos identificados:**
- ✅ **Atomicidade garantida**: se algo falhar, débito é revertido; não há perda de dados ou QualiPoints "pendurados".
- ✅ **Validações antes do envio**: API valida saldo, destinatário e valor; erros são retornados com código específico.
- ✅ **Mensagens de erro mapeadas** (seção 11.2): há sugestões de mensagens empáticas (ex.: "Saldo insuficiente. Seu saldo atual não permite este envio.").
- ✅ **Idempotência evita perda por retry**: se falha de rede, o usuário reboa sem medo de perda dupla.

**Gaps identificados:**
- ❌ **Mensagens de erro podem ser muito técnicas em alguns casos**: códigos como `ACCOUNT_NOT_ACTIVE` não explicam bem por que a conta foi bloqueada. Oportunidade: "Sua conta está sob análise. Entre em contato com o suporte para mais informações."
- ❌ **Falta de opção de recuperação ou suporte integrado**: quando há erro (ex.: conta bloqueada), não há link direto para suporte ou FAQ. Usuário fica desamparado.
- ❌ **Sem notificação de falha ou retenção de tentativa**: se timeout ocorre e o usuário fecha o App, pode não saber que a transação falhou. Oportunidade: notificação ou badge com "1 envio pendente de retry".
- ❌ **Falta de contexto em caso de rejeição**: se envio para si mesmo é rejeitado, mensagem pode soar como culpa ("Envio para a própria conta não é permitido"). Oportunidade: tom mais leve ("Parece que você selecionou a própria conta. Verifique e tente novamente.").

### Recomendações para Tristeza

1. **Mensagens empáticas e acionáveis**: não culpabilizar; oferecer próximos passos. Ex.: "Saldo insuficiente. Você tem 50 QualiPoints. Para enviar 100, ganhe mais 50." (com link para "Como ganhar pontos?").
2. **Suporte integrado**: em caso de erro crítico (conta bloqueada, erro interno), oferecer botão "Contatar suporte" ou link para FAQ.
3. **Recuperação de estado**: se a tela fecha ou App bate, recuperar contexto do envio (ID do destinatário, valor) para não perder trabalho.
4. **Notificação de falha persistente**: se timeout ocorre, exibir badge/notificação lembrando que há um envio aguardando retry.
5. **Histórico de tentativas falhadas**: em "transações recentes", destacar tentativas falhadas com opção de retry rápido.

---

## 3. RAIVA (Anger) — Frustração, bloqueios, falta de saída

### Análise do requisito

**Pontos positivos identificados:**
- ✅ **Sem loops infinitos ou bloqueios**: validações são rápidas (timeout 10s máximo); não há captchas ou passos desnecessários.
- ✅ **Claro caminho de recuperação**: erros retornam códigos específicos (não genéricos); usuário sabe exatamente o que fazer.
- ✅ **Possibilidade de cancelamento**: usuário pode sair do fluxo ou fechar a tela sem ficar "preso".
- ✅ **Sem aprovação manual**: envio é imediato; não há espera indefinida por aprovação de suporte.

**Gaps identificados:**
- ❌ **Validação de valor pode ser frustrante sem feedback prévia**: se usuário digita valor fora do intervalo (ex.: 11.000), só descobre após enviar. Oportunidade: validação em tempo real (conforme digita).
- ❌ **Erro de timeout (504) sem clara explicação**: mensagem genérica pode frustrar. Usuário não sabe se deve retry imediatamente ou aguardar. Oportunidade: "Servidores ocupados. Tente novamente em 10 segundos."
- ❌ **Falta de feedback durante carregamento**: requisição pode levar até 10s; sem indicador de progresso claro, usuário pode pensar que travou.
- ❌ **Validação de ID de destinatário**: se usuário digita ID errado (ex.: typo), só descobre após enviar. Oportunidade: autocomplete, busca de usuário, ou validação em tempo real.
- ❌ **Sem desfazer (undo) após concluído**: se usuário envia por engano, não há opção de reverter (no escopo MVP). Apenas suporte manual.

### Recomendações para Raiva

1. **Validação em tempo real**: 
   - Campo de valor: indicar intervalo aceitável; blocar caracteres inválidos.
   - Campo de destinatário: autocomplete/busca com validação de existência antes de enviar.
2. **Indicador de progresso claro**: barra de carregamento ou spinner + mensagem ("Processando seu envio..." ou "Validando dados...").
3. **Mensagens de timeout melhores**: "Servidores sobrecarregados. Tente novamente em alguns segundos" + opção de retry automático.
4. **Botão de cancelamento visível**: em toda tela de processamento, permitir cancelar sem culpa.
5. **(Backlog) Desfazer transação**: após alguns minutos (ex.: 5 min), permitir reverter envio com confirmação (para casos de engano).

---

## 4. MEDO (Fear) — Insegurança, ações irreversíveis, privacidade

### Análise do requisito

**Pontos positivos identificados:**
- ✅ **HTTPS obrigatório**: comunicação segura; dados sensíveis não trafegam em plain text.
- ✅ **Autenticação via token**: remetente identificado com segurança; ninguém consegue enviar em nome de outro sem acesso.
- ✅ **Validação server-side**: decisão de débito é feita no servidor, não confiável ao cliente.
- ✅ **Atomicidade**: não há risco de "débito sem crédito" ou estado inconsistente.
- ✅ **Auditoria**: toda tentativa é registrada para análise.
- ✅ **Dados do destinatário não expõem privacidade**: apenas ID é enviado, não e-mail ou telefone na API.

**Gaps identificados:**
- ❌ **Falta de confirmação em duas etapas para envios altos**: envio de 10.000 QualiPoints é feito de uma vez; não há confirmação adicional ("Você tem certeza?"). Oportunidade: confirmação para valores acima de X.
- ❌ **Falta de menção a sessão/timeout**: quanto tempo a sessão do usuário é válida? Se deixar App aberto, sessão expira? Oportunidade: comunicar timeout de sessão (ex.: "Você será desconectado em 15 min de inatividade").
- ❌ **Sem controle de acesso granular**: requisito não menciona perguntas de segurança ou biometria para envios sensíveis. Backlog, mas relevante para confiança.
- ❌ **Falta de aviso sobre o que acontece com os dados**: não há menção a política de privacidade ou retenção de transações.
- ❌ **Sem verificação de identidade do destinatário**: usuário pode enviar para ID errado sem saber se é a pessoa certa. Oportunidade: confirmar nome do destinatário antes de enviar.

### Recomendações para Medo

1. **Confirmação em duas etapas para valores altos**: 
   - Se valor > 5.000 (configurável), exigir: revisar dados (ID + nome do destinatário + valor) + confirmar com "Enviar agora".
   - Adicionar checkbox "Tenho certeza" se valor é máximo (10.000).
2. **Exibir nome do destinatário antes de confirmar**: evita envio para ID errado. API pode retornar nome do destinatário para validação.
3. **Avisos de segurança sutis**: "Este é um envio permanente. Verifique o destinatário com cuidado."
4. **Menção a timeout de sessão**: "Sua sessão expirará em 15 minutos de inatividade" (em tela ou help).
5. **(Backlog) Autenticação adicional**: PIN, biometria ou pergunta de segurança para valores > X.
6. **Link a política de privacidade**: "Como seus dados são protegidos?" (em tela de envio ou help).

---

## 5. NOJINHO (Disgust) — Desordem, poluição visual, conteúdo inadequado

### Análise do requisito

**Pontos positivos identificados:**
- ✅ **Requisito descreve UI limpa**: não há menção a excesso de anúncios ou pop-ups.
- ✅ **Mensagens de erro claras e concisas**: códigos são mapeados a mensagens legíveis.
- ✅ **Lista de transações simples**: apenas últimas 10, sem filtros complexos; mantém simplicidade.
- ✅ **Sem dados irrelevantes no fluxo**: envio pedindo apenas ID e valor; nada supérfluo.

**Gaps identificados:**
- ❌ **Falta de especificação de UI/UX**: requisito é funcional, mas não detalha layout, cores, tipografia ou consistência visual. Oportunidade: design system estar documentado.
- ❌ **Falta de mensagens contextuais sobre prazos ou fees**: não há comunicação sobre se há taxa ou demora. Usuário não sabe se será imediato ou se há custo oculto.
- ❌ **Lista de transações sem formatação visual**: "últimas 10" pode ser monótona. Oportunidade: agrupar por data (Hoje, Ontem, Esta semana), destacar visual e detalhes.
- ❌ **Sem ícones ou status visuais**: transação de envio vs. recebimento pode não ser claro em lista. Ícones (seta ↗ ou ↙) ajudam.
- ❌ **Falta de feedback sobre contas bloqueadas ou inativos**: se destinatário está inativo, não há menção a por quê (suspeito? em análise?). Usuário pode ficar desconfortável.

### Recomendações para Nojinho

1. **Design system consistente**: garantir que tela de envio, confirmação e lista usem mesmas cores, fontes, espaçamento.
2. **Formatação visual da lista**:
   - Agrupar transações por data (Hoje, Ontem, Esta semana, Mais antigos).
   - Ícones para envio (↗) vs. recebimento (↙).
   - Destaque de transações recentes (fundo sutilmente mais claro).
3. **Clareza sobre taxas e prazos**: incluir nota na tela (ex.: "Transferência imediata. Sem taxas." ou "Taxa: 1 QualiPoint").
4. **Razão de rejeição de destinatário**: se `RECIPIENT_NOT_FOUND`, mensagem pode detalhar: "Usuário não encontrado ou conta inativa." (transmite que é legítimo, não suspicioso).
5. **Organização de erros**: mensagens de erro em box destacado (diferente do conteúdo normal); tom consistente.

---

## 6. SURPRESA (Surprise) — Surpresas positivas e negativas

### Análise do requisito

**Pontos positivos identificados:**
- ✅ **Comportamento previsível**: requisito detalha respostas esperadas para cada cenário; usuário não é pego de surpresa negativamente.
- ✅ **Mensagens de erro mapeadas**: códigos HTTP e respostas são documentados; nada inesperado.
- ✅ **Idempotência comunicada**: usuário (e desenvolvedor) sabem que retentar é seguro.

**Gaps identificados:**
- ❌ **Sem surpresas positivas planejadas**: nada mencionado sobre "bônus por referência", "milésimo envio", "parabéns, você enviou X vezes". Oportunidade para delight.
- ❌ **Mudanças futuras sem comunicação**: se limite de valor mudar de 10.000 para 5.000, como o App fica? Requisito não menciona breaking changes. Oportunidade: planejar deprecation com aviso prévio.
- ❌ **Sem nota sobre features futuras**: requisito não deixa claro se há roadmap (ex.: "em breve: comprovante em PDF"). Usuário pode ficar surpreso quando não entrega.
- ❌ **Comportamento de retry pode surpreender**: se usuário reenviar após 10 min com mesma Idempotency-Key e recebe o mesmo sucesso (sem reprocessar), pode pensar que foi debitado novamente. Oportunidade: comunicar que "você já enviou isso; aqui está o resultado anterior".

### Recomendações para Surpresa

1. **Surpresas positivas planejadas** *(backlog)*:
   - Notificação quando atinge milestones (100º envio, 100.000 pontos enviados, etc.).
   - Bônus raro por generosidade (ex.: "Você enviou 10x este mês! Ganhe 5 bônus.").
2. **Comunicação de breaking changes**:
   - Se limite mudar, avisar com antecedência (ex.: "Em 30 dias, limite máximo por envio será 5.000").
   - Documentar roadmap públi co ou em app ("Em breve: Comprovante em PDF").
3. **Clareza sobre idempotência**:
   - Se usuário reenviar com mesma chave e recebe sucesso antigo, exibir: "Este envio já foi concluído em [data]. Aqui está o recibo." (não deixar dúvida).
4. **Atualizações de painel**: comunicar quando dados foram atualizados por última vez (ex.: "Dados atualizados às 14h30. Atualizar agora?" no painel web).

---

## 7. CONFIANÇA (Trust) — Consistência, promessas cumpridas, transparência

### Análise do requisito

**Pontos positivos identificados:**
- ✅ **Atomicidade garantida**: promessa de que "débito + crédito + registro" ocorrem juntos; se um falhar, tudo reverte. Confiança máxima.
- ✅ **Fonte única de verdade (servidor)**: toda validação feita no servidor; App não toma decisão de débito.
- ✅ **Auditoria registrada**: toda tentativa é loggada; suporte pode rastrear.
- ✅ **Contrato da API claro**: seção 10 detalha headers, corpo, respostas; desenvolvedor confia no que esperar.
- ✅ **Idempotência implementada**: não há risco de débito duplo; confiança em retry.
- ✅ **HTTPS obrigatório**: dados em trânsito seguros.
- ✅ **Dados consistentes entre App e painel**: mesma fonte (banco); após envio, saldo reflete em ambos (após refresh).

**Gaps identificados:**
- ❌ **Falta de timestamp preciso**: transações não mencionam fuso horário. Usuário em São Paulo e outro em Brasília podem ficar confusos com horários. Oportunidade: ISO8601 com timezone.
- ❌ **Sem menção a SLA (Service Level Agreement)**: não há garantia de que API responde em 10s "99% das vezes". Usuário não sabe se é confiável. Requisito diz "até 10s", mas sem SLA.
- ❌ **Painel web requer refresh manual**: não há confirmação de que dados estão atualizados. Usuário pode confundir saldo desatualizado. Oportunidade: "Última atualização: 2 min atrás" ou polling automático.
- ❌ **Sem comprovante ou recibo persistente**: após envio, não há PDF ou e-mail de recibo (escopo MVP). Usuário pode questionar se foi realmente enviado. Oportunidade: ao menos um ID de transação visível que pode consultar depois.
- ❌ **Falta de menção a reversão de transação**: se enviou errado, não há forma de reverter (a não ser suporte manual). Confiança fica abalada.

### Recomendações para Confiança

1. **Transação ID visível e copiável**: após envio bem-sucedido, exibir ID único (ex.: "TXN-202602211430-ABC123") que usuário pode anotar ou compartilhar. Reforça que é real.
2. **Timestamp com timezone**: ISO8601 com timezone (ex.: "2026-02-21T14:30:00-03:00 [Brasília]").
3. **Indicador de atualização no painel web**: "Última atualização: agora" ou "há 2 minutos" com botão de refresh.
4. **Recibo persistente** *(backlog)*: ao menos uma página de detalhes da transação que mostra: ID, data, hora, remetente, destinatário, valor, status. Pode ser compartilhada ou impressa.
5. **Política de reversão**: documentar claramente: "Transações são irreversíveis após [N] horas. Para desfazer, contate suporte em até [tempo]."
6. **Status de confiabilidade**: em vez de apenas "erro", retornar mais contexto. Ex.: "INSUFFICIENT_BALANCE" com "Saldo atual: 50 QP; requerido: 100 QP."

---

## 8. ANTECIPAÇÃO (Anticipation) — Expectativa, progresso, gestão de prazos

### Análise do requisito

**Pontos positivos identificados:**
- ✅ **Operação síncrona**: usuário **não espera** por aprovação manual; envio é imediato. Reduz ansiedade.
- ✅ **Timeout definido (10s)**: há um limite claro; usuário sabe que não aguardará infinitamente.
- ✅ **Feedback durante processamento**: requisito menciona estado "Processando" com indicador de carregamento.
- ✅ **Resposta rápida esperada**: SLA implícito de ~10s para transações normais.

**Gaps identificados:**
- ❌ **Sem indicador de progresso detalhado**: "Processando" é genérico. Não há menção a etapas ("Validando... Debitando... Creditando... Registrando...").
- ❌ **Sem comunicação sobre tempo estimado**: timeout é 10s, mas usuário não sabe se será 1s ou 9s. Oportunidade: "Isso deve levar alguns segundos."
- ❌ **Sem expectativa gerenciada sobre disponibilidade do destinatário**: envio é imediato, mas será que o destinatário recebe notificação? Quando fica disponível? Requisito não menciona.
- ❌ **Falta de opção de agendamento** *(backlog)*: não há menção a envios futuros ou recorrentes.
- ❌ **Painel web sem polling automático**: usuário vê saldo desatualizado e fica ansioso ("será que atualizou?").
- ❌ **Sem notificação de chegada** *(backlog)*: destinatário não recebe aviso de que recebeu pontos; fica aguardando indefinidamente.

### Recomendações para Antecipação

1. **Progresso visual em etapas**:
   - "Validando dados..." (1s)
   - "Processando transação..." (5s)
   - "Finalizando..." (1s)
   - Barra de progresso ou pontos animados (. → .. → ...).
2. **Mensagem sobre timing**: "Isso deve levar alguns segundos. Aguarde..." (reforça que é normal esperar um pouco).
3. **Expectativa sobre destinatário**: "João receberá seus QualiPoints imediatamente" ou "João será notificado sobre o envio" (no requisito ou mensagem pós-envio).
4. **Painel web com polling**: a cada 10-30s, buscar dados novamente (sem user notar); atualizar saldo em tempo real com animação suave.
5. **Confirmação de recebimento** *(backlog)*: após o destinatário acessar, marcar como "lido" na lista de transações do remetente.
6. **Notificação ao destinatário** *(backlog)*: push/SMS/e-mail quando recebe pontos (para criar antecipação positiva).

---

## 9. DESCONFIANÇA (Distrust) — Inconsistências, falta de transparência, ceticismo

### Análise do requisito

**Pontos positivos identificados:**
- ✅ **Regras de negócio claras**: seção 9 consolida todas as regras; sem ambiguidade.
- ✅ **Contrato da API detalhado**: código HTTP, corpo, headers — nada deixado ao acaso.
- ✅ **Validação robusta**: saldo, destinatário, valor, envio para si mesmo — tudo validado.
- ✅ **Edge cases tratados**: seção 14 lista cenários e como são tratados.
- ✅ **Atomicidade**: promessa de que débito e crédito ocorrem juntos ou não ocorrem; sem estado inconsistente.

**Gaps identificados:**
- ❌ **Sem menção a consistência eventual**: se há replicação de banco, há lag entre escrita e leitura em outro nó? Usuário pode ver saldo desatualizado no painel e desconfiar. Oportunidade: documentar.
- ❌ **Falta de transparência sobre "por trás dos panos"**: qual exatamente é o banco? Quem tem acesso? Oportunidade: documentação de infraestrutura (mesmo que resumida).
- ❌ **Sem menção a testes automatizados**: desenvolvedor confia que a atomicidade funciona, mas como foi testado? Oportunidade: mencionar testes (atomicidade, idempotência, timeout).
- ❌ **Falta de métricas de confiabilidade**: não há menção a taxa de sucesso, erros, ou SLA. Usuário não tem base para confiar.
- ❌ **Comportamento em caso de falha de banco**: se banco fica indisponível durante envio, o que acontece? Requisito não menciona. Oportunidade: "API retorna 503 em caso de indisponibilidade de banco."
- ❌ **Sem rastreabilidade de requisições**: usuário não consegue correlacionar o que enviou com o que está registrado. Oportunidade: exibir request ID ou transaction ID.
- ❌ **Falta de documentação de limites**: não há menção a rate limits futuros (ex.: "em breve, máximo 100 envios por hora"). Mudança futura pode parecer injusta.

### Recomendações para Desconfiança

1. **Transparência de infraestrutura**: documentar (mesmo que resumido) onde dados são armazenados, quem tem acesso, replicação, backup.
   - Ex.: "Dados armazenados em banco relacional com replicação em 3 datacenters. Suporte e segurança têm acesso apenas com permissão."
2. **Métricas de confiabilidade públicas** *(backlog)*:
   - Dashboard de status: "API: 99.9% disponível este mês" ou "Nenhuma falha crítica nos últimos 30 dias."
3. **Rastreabilidade de requisição**: cada transação tem ID único visível (ex.: `TXN-202602211430-ABC123` ou `Idempotency-Key`). Usuário pode compartilhar para suporte.
4. **Comportamento em falha de banco**: documentar: "Se banco indisponível, API retorna 503 (Service Unavailable). Tente novamente em alguns minutos."
5. **Taxa de sucesso por tipo de erro**: em relatório de confiabilidade, mencionar: "99% das transações completadas com sucesso; 0.5% tempo limite excedido; 0.3% saldo insuficiente; 0.2% erro interno."
6. **Notificação de incidente**: se há falha crítica (ex.: transações não estão sendo processadas), exibir banner no App: "⚠ Há um problema com envios no momento. Aguarde atualização."
7. **Documentação de limites futuros**: comunicar antecipadamente: "Em [data], novos limites entrarão em vigor: máximo 100 envios por hora por usuário." (90 dias antes).

---

## 10. Checklist de Análise EMOTIONS — Requisito Envio de QualiPoints

- [x] **Alegria**: Feedback positivo, eficiência e momentos de satisfação? ⚠ Parcial — faltam celebração visual e atalhos.
- [x] **Tristeza**: Perda de dados evitada? Falhas comunicadas com empatia? ⚠ Parcial — falta suporte integrado e contexto em erros.
- [x] **Raiva**: Sem bloqueios desnecessários, mensagens claras, opção de recuperação? ⚠ Parcial — falta validação em tempo real.
- [x] **Medo**: Ações irreversíveis confirmadas? Segurança e privacidade transmitidas? ⚠ Parcial — falta confirmação em dois passos e verificação de identidade do destinatário.
- [x] **Nojinho**: UI organizada, conteúdo relevante, sem poluição visual? ⚠ Parcial — requisito funcional, mas sem detalhe de UI/UX.
- [x] **Surpresa**: Surpresas positivas onde apropriado? Surpresas negativas evitadas? ⚠ Parcial — sem delighters, sem comunicação de roadmap.
- [x] **Confiança**: Dados consistentes, promessas cumpridas, transparência? ✅ Forte — atomicidade, auditoria, contrato claro.
- [x] **Antecipação**: Progresso indicado em esperas longas, prazos realistas comunicados? ⚠ Parcial — falta progresso em etapas, polling automático.
- [x] **Desconfiança**: Inconsistências e comportamentos geradores de ceticismo identificados e mitigados? ⚠ Parcial — falta rastreabilidade, métricas de confiabilidade.
- [x] **Requisitos**: Gaps de impacto emocional documentados? ✅ Sim — documentado neste relatório.
- [x] **Testes**: Casos de teste cobrindo as nove emoções para fluxos críticos? ❌ Falta — não mencionado no requisito.

---

## 11. Sumário Executivo — Impacto Emocional

| Emoção | Nível | Status | Crítico? |
|--------|-------|--------|----------|
| **Alegria** | Médio | Funcional, mas sem delight. | Não |
| **Tristeza** | Médio | Atomicidade protege; falta empatia em erros. | Não |
| **Raiva** | Médio-Alto | Sem validação em tempo real gera frustração. | **Sim** |
| **Medo** | Médio | HTTPS + server-side seguro; falta confirmação adicional. | Não |
| **Nojinho** | Baixo | Requisito limpo, mas design não especificado. | Não |
| **Surpresa** | Médio | Previsível e bem-documentado. Sem surpresas boas planejadas. | Não |
| **Confiança** | Alto | Atomicidade, auditoria, contrato claro. Forte confian ça. | ✅ |
| **Antecipação** | Médio | Síncrono reduz espera; falta progresso em etapas. | Não |
| **Desconfiança** | Médio | Bem documentado; falta rastreabilidade e métricas. | Não |

**Análise geral:**
- ✅ **Pontos fortes**: Atomicidade, segurança (HTTPS), validação server-side, contrato de API claro, idempotência.
- ⚠️ **Pontos a melhorar** (MVP ou próximos releases):
  1. **Validação em tempo real** (Raiva) — Campo de valor e ID de destinatário.
  2. **Feedback visual mais rico** (Alegria) — Animação de sucesso, celebração sutil.
  3. **Empatia em mensagens de erro** (Tristeza) — Contexto, próximos passos, suporte integrado.
  4. **Confirmação adicional para valores altos** (Medo) — Duas etapas, verificação de nome do destinatário.
  5. **Rastreabilidade** (Desconfiança) — Transaction ID visível, timestamp preciso, último acesso ao saldo.
  6. **Progresso em etapas** (Antecipação) — Loading com descrição de etapa, barra de progresso.

---

## 12. Recomendações por Prioridade

### 🔴 **Críticas (MVP)**
1. **Validação em tempo real**: campo de valor com intervalo (1-10.000) validado conforme digita.
2. **Indicador de progresso claro**: "Processando..." com feedback visual (barra ou spinner).
3. **Mensagens de erro contextualizadas**: incluir saldo atual em erros de saldo, sugestão em caso de erro.

### 🟡 **Altas (Sprint próximo)**
1. **Confirmação adicional para valores altos**: revisão de dados antes de enviar (ID + nome do destinatário + valor).
2. **Feedback positivo destacado**: mensagem com nome do destinatário ("100 QualiPoints enviados para João!").
3. **Transaction ID visível**: exibir após sucesso para referência futura.
4. **Painel web com polling**: atualizar saldo sem refresh manual.

### 🟢 **Médias (Backlog)**
1. **Histórico de tentativas falhadas**: retry rápido de envios que falharam.
2. **Contatos favoritos**: marcar destinatários frequentes.
3. **Notificação de incidente**: avisar quando serviço está indisponível.
4. **Comprovante/recibo persistente**: página de detalhes da transação.

### 🔵 **Baixas (Roadmap futuro)**
1. **Surpresas positivas**: milestones, bônus por generosidade.
2. **Autenticação adicional**: PIN ou biometria para valores muito altos.
3. **Agendamento de envios**: transferências futuras.
4. **Desfazer transação**: reverter envio em até N horas.

---

## 13. Casos de Teste EMOTIONS — Sugestões

### Teste de Alegria
- ✅ **Cenário:** Usuário envía 100 QualiPoints para João com saldo suficiente.
- **Critério:** Mensagem de sucesso mostra nome do destinatário e valor; saldo é atualizado em tempo real; confirmação visual é clara e positiva.

### Teste de Tristeza
- ✅ **Cenário:** Usuário perde conexão de rede durante envio e faz retry.
- **Critério:** Retry com mesma Idempotency-Key não debita novamente; App mostra "Tentando novamente..." com opção de retentar manualmente.

- ✅ **Cenário:** Envio falha por saldo insuficiente.
- **Critério:** Mensagem mostra saldo atual, quanto falta, e link para "Como ganhar pontos?".

### Teste de Raiva
- ✅ **Cenário:** Usuário digita valor fora do intervalo (ex.: 11.000).
- **Critério:** Campo rejeira entrada ou mostra aviso em tempo real ("Máximo 10.000"); usuário não é pego de surpresa após enviar.

- ✅ **Cenário:** Requisição leva 10s (timeout).
- **Critério:** Loading é visível; mensagem clara ("Servidores sobrecarregados. Tente novamente."); botão para retry.

### Teste de Medo
- ✅ **Cenário:** Usuário tenta enviar para si mesmo.
- **Critério:** Validação em tempo real bloqueia; mensagem é clara e não culpabilizadora ("Verifique o destinatário.").

- ✅ **Cenário:** Envio de 10.000 QualiPoints (máximo).
- **Critério:** Confirmação adicional pedindo revisão; checkbox "Tenho certeza"; não há saída fácil sem confirmar.

### Teste de Nojinho
- ✅ **Cenário:** Lista de transações é exibida.
- **Critério:** Transações agrupadas por data; ícones para envio vs. recebimento; cores e fonts consistentes; sem poluição visual.

### Teste de Surpresa
- ✅ **Cenário:** Usuário reenvía com mesma Idempotency-Key 10 minutos depois.
- **Critério:** Recebe confirmação de que "Este envio já foi concluído em [data]. Aqui está o recibo" (não deixa dúvida de duplicação).

### Teste de Confiança
- ✅ **Cenário:** Após envio bem-sucedido, usuário acessa painel web.
- **Critério:** Saldo reflete nova transação (após refresh ou polling); Transaction ID visível e copiável.

### Teste de Antecipação
- ✅ **Cenário:** Envio normal leva 3-5 segundos.
- **Critério:** Loading mostra progresso em etapas ("Validando... Processando... Finalizando..."); mensagem "Isso deve levar alguns segundos."

### Teste de Desconfiança
- ✅ **Cenário:** Usuário quer rastrear transação enviada há 1 semana.
- **Critério:** Pode encontrar na lista ou buscar por Transaction ID; detalhes completos (data, hora, remetente, destinatário, valor, status) disponíveis.

---

## 14. Conclusão

O **Requisito REQ_INICIAL_V2** para Envio de QualiPoints demonstra **sólida base técnica** em termos de segurança, atomicidade e validação server-side. A confiança do usuário é **forte** neste aspecto.

No entanto, há **oportunidades claras** de melhoria emocional:

1. **Validação em tempo real** reduzirá Raiva (frustração com bloqueios).
2. **Feedback visual e animações** aumentarão Alegria (satisfação com sucesso).
3. **Mensagens empáticas e acionáveis** mitigarão Tristeza (desilusão com falhas).
4. **Confirmação adicional e verificação de identidade** reduzirão Medo (insegurança em ações irreversíveis).
5. **Rastreabilidade e métricas** eliminarão Desconfiança (ceticismo sobre confiabilidade).

Essas melhorias, priorizadas pela urgência e impacto, **transformarão a experiência de um fluxo funcional para um fluxo que encanta**, reduzindo churn, aumentando engajamento e fortalecendo a reputação da Carteira Digital QualiPoints.

---

## Referências

- **Arquivo analisado:** REQ_INICIAL_V2.md (Requisito Revisado v2)
- **Heurística aplicada:** EMOTIONS (Priscila Caimi, Jonatas Martins) — Análise de impacto emocional do software
- **Data da análise:** 21 de fevereiro de 2026
