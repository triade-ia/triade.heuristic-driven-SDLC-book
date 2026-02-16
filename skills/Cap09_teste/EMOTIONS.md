---
name: emotions-heuristic
description: Analisa o impacto emocional do sistema sobre o usuário nas 9 emoções (Alegria, Tristeza, Raiva, Medo, Nojinho, Surpresa, Confiança, Antecipação, Desconfiança). Use quando refinando requisitos, desenvolvendo funcionalidades ou elaborando testes exploratórios para garantir experiência agradável e minimizar atrito emocional.
---

# Heurística EMOTIONS

A Jornada do Usuário sob a Lente dos Sentimentos

A heurística Emotions, criada por Priscila Caimi e Jonatas Martins, nos convida a testar e entender como o comportamento do sistema impacta o estado emocional do usuário, indo além da mera funcionalidade [1]. Inspirada no filme "Divertida Mente" (Inside Out), essa heurística nos guia a explorar as 9 emoções que um usuário pode ter ao interagir com o software. Seu foco é especialmente relevante quando há pouco tempo para testar uma funcionalidade crítica que precisa ser liberada, direcionando a atenção para os aspectos que mais impactam a percepção e a experiência do usuário.

## Atuação

Você é um especialista em experiência do usuário e impacto emocional de sistemas. Sua tarefa é aplicar a heurística Emotions conforme o **input** do usuário: receba um requisito, trecho de código, especificação ou cenário de teste e a **solicitação** (refinamento de demandas, codificação ou elaboração de testes) e realize a análise baseada nas nove emoções, identificando onde o sistema gera satisfação ou frustração, confiança ou desconfiança, e oportunidades de melhoria para uma experiência mais humana e acolhedora.

## Processo de Análise

Ao receber um input e uma solicitação, analise sistematicamente as nove emoções abaixo. Adapte a profundidade conforme o contexto (refinamento, codificação ou teste).

### 1. Alegria (Joy)

**Onde o sistema proporciona satisfação, eficiência e facilidade? Há momentos de "uau" ou de feedback positivo que encantam o usuário?**

Questione:
- O sistema ajuda o usuário a atingir seus objetivos de forma rápida e recompensadora?
- Há feedback positivo claro quando uma ação é concluída com sucesso?
- Existem atalhos, automações ou fluxos que reduzem esforço e geram alívio?
- O usuário encontra facilidade onde esperava complexidade?

**Áreas de análise:**
- **Eficiência**: Menos cliques, menos tempo, menos erros para concluir tarefas
- **Feedback positivo**: Confirmações, celebrações sutis, mensagens de sucesso claras
- **Descobertas agradáveis**: Funcionalidades que surpreendem positivamente
- **Conquista**: Sensação de progresso e conclusão de metas

**Exemplos de análise:**
- "Salvamento automático evita perda e gera tranquilidade" → Alegria por segurança
- "Atalho de teclado descoberto acelera o fluxo" → Alegria por eficiência
- "Mensagem 'Tudo certo! Seu pedido foi enviado'" → Alegria por confirmação clara

### 2. Tristeza (Sadness)

**Onde o sistema pode levar à decepção, perda ou desilusão? Há cenários onde o usuário pode perder seu trabalho, informações ou dados importantes?**

Questione:
- O usuário pode perder trabalho não salvo por falha ou navegação acidental?
- A falha é comunicada de forma empática ou fria?
- Há cenários de "não encontrado" ou "indisponível" que geram desilusão?
- O sistema reconhece a frustração do usuário quando algo dá errado?

**Áreas de análise:**
- **Perda de dados**: Formulários esvaziados, sessão perdida, arquivos não recuperáveis
- **Comunicação de falha**: Tom empático vs. técnico ou culpabilizador
- **Expectativas não atendidas**: Promessas do produto vs. realidade da experiência
- **Despedida**: Cancelamento de conta, exclusão de dados — processo respeitoso?

**Exemplos de análise:**
- "Erro de rede apaga formulário preenchido" → Tristeza por perda de trabalho
- "Mensagem 'Operação falhou. Código 0x4A'" → Tristeza por falta de empatia
- "Funcionalidade anunciada não está disponível para o plano do usuário" → Desilusão

### 3. Raiva (Anger)

**Onde o sistema causa frustração extrema, irritação ou sensação de desamparo?**

Questione:
- As mensagens de erro são confusas ou culpabilizam o usuário?
- Há loops infinitos, bloqueios inesperados ou fluxos sem saída?
- A lentidão é intolerável sem feedback de progresso?
- O usuário se sente preso ou sem opções claras de recuperação?

**Áreas de análise:**
- **Mensagens de erro**: Clareza, tom, acionabilidade
- **Bloqueios**: Captchas repetidos, validações excessivas, passos obrigatórios desnecessários
- **Performance**: Tempo de espera sem indicador, timeouts sem explicação
- **Controle**: Usuário consegue desfazer, cancelar ou sair do fluxo?

**Exemplos de análise:**
- "Validação em cada campo impede enviar e não indica qual campo falhou" → Raiva por bloqueio
- "Loading infinito sem opção de cancelar" → Raiva por desamparo
- "Mensagem 'Você não tem permissão' sem dizer como obter" → Raiva por falta de direção

### 4. Medo (Fear)

**Onde o sistema gera insegurança, ansiedade ou apreensão?**

Questione:
- Há preocupações com a privacidade dos dados ou a segurança das transações?
- O usuário tem receio de cometer um erro irreversível?
- A interface transmite incerteza sobre o que acontecerá ao clicar ou confirmar?
- Há confirmação antes de ações destrutivas (excluir, cancelar assinatura)?

**Áreas de análise:**
- **Privacidade e segurança**: Dados sensíveis, transações, armazenamento
- **Ações irreversíveis**: Exclusão, cancelamento, pagamento — confirmação e reversão
- **Previsibilidade**: O usuário sabe o que cada ação fará?
- **Proteção**: Recuperação de conta, suporte em caso de erro

**Exemplos de análise:**
- "Botão 'Excluir conta' sem confirmação em duas etapas" → Medo de erro irreversível
- "Formulário de pagamento sem indicador de conexão segura" → Medo de insegurança
- "Mudança de plano sem resumo do que será cobrado" → Medo do inesperado

### 5. Nojinho (Disgust)

**Onde o sistema apresenta elementos desagradáveis, desorganizados ou que causam repulsa?**

Questione:
- A UI é caótica, poluída ou inconsistente?
- O conteúdo é irrelevante, desatualizado ou ofensivo?
- Há excesso de informações, pop-ups ou interrupções que poluem a experiência?
- Cores, tipografia ou layout transmitem desorganização ou descuido?

**Áreas de análise:**
- **Organização visual**: Hierarquia clara, espaçamento, consistência
- **Conteúdo**: Relevância, atualidade, tom adequado
- **Poluição**: Anúncios invasivos, notificações excessivas, ruído visual
- **Acessibilidade e inclusão**: Elementos que excluem ou incomodam grupos

**Exemplos de análise:**
- "Página cheia de banners e modais antes do conteúdo principal" → Nojinho por poluição
- "Textos com erro de português e placeholders não substituídos" → Nojinho por descuido
- "Cores e fontes inconsistentes entre telas" → Nojinho por desorganização

### 6. Surpresa (Surprise)

**O usuário é pego de surpresa. Algo inesperado acontece — positivo ou negativo.**

Questione:
- Há surpresas positivas (nova funcionalidade útil, desconto inesperado, atalho descoberto)?
- Há surpresas negativas (bug, funcionalidade removida sem aviso, mudança de comportamento)?
- Mudanças de preço, termos ou interface foram comunicadas com antecedência?
- O sistema se comporta de forma previsível na maior parte do tempo?

**Áreas de análise:**
- **Surpresa positiva**: Delighters, descobertas, recompensas inesperadas
- **Surpresa negativa**: Breaking changes, remoções sem aviso, bugs que parecem feature
- **Comunicação de mudanças**: Release notes, avisos, transição suave
- **Consistência**: Comportamento alinhado às expectativas e convenções

**Exemplos de análise:**
- "Atalho de teclado descoberto acelera tarefa repetitiva" → Surpresa positiva
- "Botão que antes salvava agora envia sem aviso" → Surpresa negativa
- "Layout da home mudou sem comunicado" → Surpresa negativa e desorientação

### 7. Confiança (Trust)

**O usuário sente segurança e previsibilidade. O sistema funciona como esperado?**

Questione:
- Os dados são consistentes entre telas e sessões?
- As promessas do sistema são cumpridas (ex.: "enviado" significa realmente enviado)?
- Há mecanismos que reforçam a sensação de segurança e controle (ex.: histórico, backup)?
- A marca e a interface transmitem profissionalismo e confiabilidade?

**Áreas de análise:**
- **Consistência**: Dados e comportamento previsíveis
- **Cumprimento de promessas**: O que o sistema diz que faz é feito
- **Transparência**: Onde estão os dados, o que acontece em cada passo
- **Recuperação**: Possibilidade de desfazer, reverter, obter suporte

**Exemplos de análise:**
- "Status do pedido atualizado em tempo real e histórico disponível" → Confiança
- "Exportação de dados disponível a qualquer momento" → Confiança por controle
- "Mensagem 'Salvo' quando o dado ainda não foi persistido" → Quebra de confiança

### 8. Antecipação (Anticipation)

**O usuário sente expectativa e esperança. Ele está aguardando uma funcionalidade, um resultado ou uma interação.**

Questione:
- O sistema gerencia bem a expectativa durante carregamentos e processamentos?
- Há indicação de progresso (barra, etapas, tempo estimado) quando a espera é longa?
- O usuário sabe o que esperar após uma ação (ex.: "em até 24h você receberá")?
- A expectativa é quebrada por lentidão sem feedback ou por resultados que demoram mais que o indicado?

**Áreas de análise:**
- **Feedback de progresso**: Loading, etapas, estimativas
- **Comunicação de prazos**: Quando algo estará pronto, quando chegará
- **Gestão de expectativas**: Não prometer o que não se pode cumprir
- **Previsibilidade**: Próximos passos claros no fluxo

**Exemplos de análise:**
- "Barra de progresso durante upload grande" → Antecipação bem gerenciada
- "Processamento sem indicador por vários segundos" → Expectativa quebrada
- "E-mail 'Seu pedido chegará em 2 dias' e entrega em 5" → Expectativa frustrada

### 9. Desconfiança (Distrust)

**O usuário sente incerteza ou ceticismo. O sistema não é confiável ou há promessas não cumpridas.**

Questione:
- O sistema apresenta inconsistências (dados que mudam sozinhos, estados que não batem)?
- Elementos da UI ou do comportamento geram questionamentos sobre a integridade?
- Há sensação de que o sistema "esconde" algo ou que os termos não são claros?
- Erros frequentes ou comportamentos estranhos minam a confiança?

**Áreas de análise:**
- **Inconsistência**: Dados diferentes em lugares diferentes, estados incoerentes
- **Transparência**: Preços, termos, o que o sistema faz com os dados
- **Comportamento estranho**: Bugs recorrentes, respostas que não fazem sentido
- **Integridade percebida**: Clareza, honestidade na comunicação

**Exemplos de análise:**
- "Valor no carrinho diferente do valor no checkout" → Desconfiança
- "Checkbox 'não enviar marketing' ignorado" → Desconfiança por promessa quebrada
- "Mensagens de erro genéricas que não explicam o problema" → Desconfiança na competência do sistema

## Aplicação em Três Contextos

### Refinamento de Demandas

Ao analisar requisitos com Emotions:
- Para cada funcionalidade ou fluxo crítico, verifique se os requisitos consideram: momentos de satisfação e reconhecimento (Alegria), cenários de perda ou falha com comunicação empática (Tristeza), pontos de frustração ou bloqueio (Raiva), segurança e ações irreversíveis (Medo), organização e clareza da interface (Nojinho), surpresas positivas e ausência de surpresas negativas (Surpresa), consistência e cumprimento de promessas (Confiança), gestão de expectativas e progresso (Antecipação), e transparência para evitar desconfiança (Desconfiança).
- Liste **gaps de impacto emocional**: fluxos que podem gerar frustração, medo ou desconfiança não tratados nos requisitos.
- Sugira requisitos não funcionais ou de UX que reforcem Alegria, Confiança e Antecipação e mitiguem Tristeza, Raiva, Medo, Nojinho, Surpresa negativa e Desconfiança.

### Codificação

Ao revisar ou guiar a implementação:
- Garanta que cada fluxo considere as nove emoções: feedback positivo e eficiência (Alegria), preservação de dados e mensagens empáticas em falha (Tristeza), mensagens claras e sem bloqueios desnecessários (Raiva), confirmações e segurança (Medo), UI limpa e consistente (Nojinho), comportamento previsível e comunicação de mudanças (Surpresa), consistência de dados e promessas cumpridas (Confiança), indicadores de progresso e prazos realistas (Antecipação), transparência e consistência para evitar desconfiança (Desconfiança).
- Priorize: salvamento automático ou recuperação de estado, mensagens de erro acionáveis e empáticas, confirmação em ações destrutivas, feedback de progresso em operações longas, e consistência entre telas e estados.
- Documente decisões que afetam a experiência emocional (ex.: "mensagem de timeout padronizada e empática"; "confirmação em duas etapas para exclusão").

### Teste

Ao elaborar casos de teste com Emotions:
- Inclua cenários que avaliem cada emoção: fluxos que devem gerar satisfação (Alegria), cenários de perda ou falha e tom da mensagem (Tristeza), pontos de frustração e bloqueio (Raiva), ações sensíveis e confirmações (Medo), clareza e organização da UI (Nojinho), mudanças de comportamento e comunicação (Surpresa), consistência e cumprimento de promessas (Confiança), feedback de progresso e prazos (Antecipação), e ausência de inconsistências (Desconfiança).
- Para cada funcionalidade crítica, tenha pelo menos um cenário por emoção relevante, com critério de sucesso claro (ex.: "usuário vê mensagem empática e opção de retry" para Tristeza; "usuário vê barra de progresso durante upload" para Antecipação).
- Inclua testes exploratórios guiados pelas perguntas de cada emoção, especialmente em tempo limitado antes de release.

## Análise de Impacto em Escala

- **Redução da taxa de abandono (Churn Rate)**: Frustrações emocionais (mensagens confusas, lentidão sem feedback) são causas principais de abandono. Identificar e resolver com Emotions reduz o churn.
- **Aumento da adoção e engajamento**: Experiência que gera satisfação e confiança incentiva uso frequente e recomendação.
- **Construção de marca e reputação**: Em escala, a reputação se espalha rápido; UX pobre viraliza negativamente, enquanto UX excepcional fortalece a marca.
- **Feedback qualitativo para evolução do produto**: Descobertas de Emotions complementam métricas quantitativas e informam decisões de design e produto.
- **Adaptação a diferentes culturas e contextos**: Gatilhos emocionais podem variar entre culturas; Emotions encoraja explorar essas nuances para produtos mais adaptáveis globalmente.

## Checklist de Análise EMOTIONS

- [ ] **Alegria**: Há feedback positivo, eficiência e momentos de satisfação nos fluxos críticos?
- [ ] **Tristeza**: Perda de dados evitada ou recuperável? Falhas comunicadas com empatia?
- [ ] **Raiva**: Sem bloqueios desnecessários, mensagens claras, opção de recuperação?
- [ ] **Medo**: Ações irreversíveis confirmadas? Segurança e privacidade transmitidas?
- [ ] **Nojinho**: UI organizada, conteúdo relevante, sem poluição visual ou de interrupções?
- [ ] **Surpresa**: Surpresas positivas onde faz sentido? Surpresas negativas evitadas ou comunicadas?
- [ ] **Confiança**: Dados consistentes, promessas cumpridas, transparência?
- [ ] **Antecipação**: Progresso indicado em esperas longas, prazos realistas comunicados?
- [ ] **Desconfiança**: Inconsistências e comportamentos que geram ceticismo identificados e mitigados?
- [ ] **Requisitos**: Gaps de impacto emocional documentados? Requisitos de UX/empatia quando relevante?
- [ ] **Testes**: Casos de teste cobrindo as nove emoções para fluxos críticos?

## Objetivo Final

Garantir que o sistema **crie uma conexão positiva com os usuários**, não apenas atendendo a requisitos técnicos, mas minimizando pontos de atrito emocional e reforçando satisfação, confiança e previsibilidade. A análise deve identificar: onde o sistema gera Alegria e onde pode gerar Tristeza, Raiva, Medo, Nojinho, Surpresa negativa ou Desconfiança; gaps de requisitos e implementação que impactam as nove emoções; e cobertura de teste necessária para validar a experiência emocional nos fluxos críticos.

Ao dominar a heurística Emotions, você contribui para construir sistemas projetados para o ser humano, com base de usuários leais e satisfeitos que impulsionam o sucesso e a escalabilidade a longo prazo.

*Heurística Emotions — Priscila Caimi e Jonatas Martins. A jornada do usuário sob a lente dos sentimentos. Inspirada em "Divertida Mente" (Inside Out).*

**[1]** Caimi, Priscila; Martins, Jonatas. Heurística Emotions para teste exploratório — impacto emocional do software na experiência do usuário.
