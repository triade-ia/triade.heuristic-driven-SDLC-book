# Análise Seen and Heard: Sistema de Transferência de QualiPoints

## Contexto do Caso

Este documento apresenta a aplicação da heurística **Seen and Heard (Visto e Ouvido)** sobre o requisito de transferência de QualiPoints entre usuários. A análise investiga como o sistema comunica seu estado, ações e consequências através de múltiplos canais sensoriais (visual, auditivo, tátil), garantindo feedback claro, oportuno e acessível, identificando gaps de comunicação, barreiras de acessibilidade e oportunidades de inclusão para garantir que todos os usuários recebam informações de forma efetiva.

## Elementos de Interface Identificados

O sistema de transferência de QualiPoints envolve os seguintes elementos de interface principais:
- **Formulário de Transferência**: Campos para destinatário e valor
- **Feedback de Validação**: Mensagens de erro e sucesso
- **Indicadores de Estado**: Status do sistema, progresso de processamento
- **Confirmações**: Modais, toasts, mensagens de sucesso/erro
- **Navegação e Acessibilidade**: Suporte a teclado, leitores de tela, múltiplos canais sensoriais

## Análise Seen and Heard por Aspecto

### 1. Feedback Visual

**O sistema fornece indicações visuais claras sobre o que está acontecendo? As informações importantes são visíveis e distinguíveis?**

#### Elementos Visuais Identificados no Requisito

**Estados do Sistema:**
- ⚠️ Não especifica indicadores visuais de estado (conectado, offline, processando)
- ⚠️ Não define diferenciação visual entre estados de sucesso/erro/processamento
- ⚠️ Não menciona indicadores de progresso para ações síncronas

**Feedback de Ações:**
- ✅ Resposta 201 com `transaction_id` e `new_balance` (sucesso)
- ✅ Erros específicos: 402, 404, 422 (mas não especifica como são exibidos visualmente)
- ⚠️ Não especifica confirmação visual após transferência
- ⚠️ Não define indicadores visuais durante processamento

**Validação Visual:**
- ⚠️ Não especifica como campos inválidos são destacados visualmente
- ⚠️ Não define mensagens de erro visíveis e distinguíveis
- ⚠️ Não menciona diferenciação visual entre tipos de erro

#### Questões Levantadas

- ❓ Há indicadores visuais claros sobre o estado atual do sistema?
- ❓ O feedback visual é imediato após ações do usuário?
- ❓ As informações importantes são destacadas visualmente?
- ❓ Há diferenciação visual clara entre estados (ex: sucesso, erro, processando)?
- ❓ O contraste de cores é adequado para leitura?
- ❓ Há redundância visual (ex: ícone + cor + texto) para garantir compreensão?
- ❓ Os elementos visuais são consistentes em todo o sistema?

#### Gaps Identificados

- ⚠️ **Falta de indicadores de estado**: Não especifica como comunicar estado do sistema (conectado, offline, processando)
- ⚠️ **Falta de feedback visual imediato**: Não define confirmação visual após ações do usuário
- ⚠️ **Falta de diferenciação visual**: Não especifica cores, ícones ou estilos distintos para diferentes estados
- ⚠️ **Falta de indicadores de progresso**: Não menciona barras de progresso, spinners ou skeletons para ações síncronas
- ⚠️ **Falta de validação visual**: Não especifica como campos inválidos são destacados
- ⚠️ **Falta de hierarquia visual**: Não define como informações importantes são destacadas
- ⚠️ **Falta de consistência**: Não garante padrões visuais consistentes

#### Recomendações

**Indicadores de Estado do Sistema:**
- Badge de status no topo da interface:
  - Verde "Conectado" quando online
  - Vermelho "Offline" quando sem conexão
  - Amarelo "Sincronizando..." durante atualizações
- Ícone de status sempre visível com tooltip explicativo
- Animação sutil para estados ativos (pulsação leve)

**Feedback Visual Imediato:**
- Botão "Transferir" com feedback visual ao clicar:
  - Mudança de cor/opacidade ao pressionar
  - Estado de loading com spinner e texto "Transferindo..."
  - Desabilitar botão durante processamento (evitar duplo clique)
- Confirmação visual após sucesso:
  - Toast notification verde com ícone de check
  - Animação de sucesso (fade-in suave)
  - Atualização inline do saldo com destaque visual

**Diferenciação Visual de Estados:**
- **Sucesso**: Verde (#10B981), ícone de check (✓), fundo verde claro
- **Erro**: Vermelho (#EF4444), ícone de alerta (⚠), fundo vermelho claro
- **Processando**: Azul (#3B82F6), spinner animado, texto "Processando..."
- **Aviso**: Amarelo (#F59E0B), ícone de atenção (⚠), fundo amarelo claro
- **Informação**: Azul claro (#60A5FA), ícone de informação (i), fundo azul claro

**Indicadores de Progresso:**
- Spinner animado durante processamento da transferência
- Mensagem de status: "Processando transferência..." com animação de pontos
- Barra de progresso indeterminada para ações que podem demorar
- Skeleton loader se houver carregamento de dados adicionais

**Validação Visual:**
- Campos inválidos:
  - Borda vermelha (2px sólida)
  - Fundo vermelho claro (#FEE2E2)
  - Ícone de erro ao lado do campo
  - Mensagem de erro abaixo do campo em vermelho
- Campos válidos:
  - Borda verde quando válido e preenchido
  - Ícone de check verde ao lado
  - Mensagem de sucesso discreta (opcional)
- Campos em foco:
  - Borda azul destacada (3px)
  - Sombra sutil para profundidade

**Hierarquia Visual:**
- Informações críticas: Tamanho maior, cor destacada, negrito
- Informações secundárias: Tamanho menor, cor neutra
- Saldo disponível: Destaque visual (caixa colorida, ícone, tamanho maior)
- Mensagens de erro: Posicionadas próximo ao elemento relacionado

**Consistência:**
- Documentar guia de estilo visual com cores, ícones e padrões
- Criar componentes reutilizáveis para feedback visual
- Garantir mesmo padrão em todas as telas do sistema

### 2. Feedback Auditivo

**Há sons ou alertas que complementam o feedback visual, especialmente para usuários com deficiências visuais ou em contextos onde a atenção visual é limitada?**

#### Elementos Auditivos Identificados no Requisito

**Confirmação de Ações:**
- ⚠️ Não especifica uso de sons para confirmação de transferência
- ⚠️ Não menciona feedback auditivo para ações bem-sucedidas
- ⚠️ Não define alertas sonoros para erros críticos

**Notificações:**
- ⚠️ Não especifica sons para notificações importantes
- ⚠️ Não menciona opção de habilitar/desabilitar sons

#### Questões Levantadas

- ❓ Há sons que confirmam ações importantes?
- ❓ Os alertas sonoros são apropriados ao contexto?
- ❓ Há opção de desabilitar sons quando necessário?
- ❓ Os sons são distintos e reconhecíveis?
- ❓ Há feedback auditivo para notificações importantes?
- ❓ O sistema utiliza sons para alertar sobre erros críticos?
- ❓ Há alternativas visuais quando sons não estão disponíveis?

#### Gaps Identificados

- ⚠️ **Falta de feedback auditivo**: Não especifica uso de sons para confirmação ou alertas
- ⚠️ **Falta de controle**: Não menciona opção de habilitar/desabilitar sons
- ⚠️ **Falta de acessibilidade auditiva**: Não considera usuários com deficiência visual
- ⚠️ **Falta de contexto**: Não especifica sons apropriados para diferentes situações
- ⚠️ **Falta de redundância**: Não garante que sons sempre tenham feedback visual alternativo

#### Recomendações

**Confirmação de Ações:**
- Som de sucesso (tom ascendente curto) quando transferência é concluída com sucesso
- Som discreto e agradável (não intrusivo)
- Volume ajustável nas configurações do usuário
- Opção de desabilitar completamente

**Alertas de Erro:**
- Som de alerta (tom descendente) para erros críticos:
  - Saldo insuficiente (402)
  - Destinatário inexistente (404)
  - Erro de validação crítico (422)
- Som distinto para cada tipo de erro (opcional)
- Volume moderado (não alarmante)

**Notificações:**
- Som de notificação discreto para:
  - Transferência recebida (se aplicável)
  - Atualizações de saldo importantes
- Som diferente para notificações informativas vs. críticas

**Controles de Acessibilidade:**
- Configurações de áudio:
  - Habilitar/desabilitar sons
  - Ajustar volume (0-100%)
  - Escolher sons personalizados (opcional)
  - Testar sons antes de salvar
- Opção "Modo silencioso" para ambientes públicos
- Respeitar configurações de sistema (modo silencioso do dispositivo)

**Redundância e Alternativas:**
- Sons sempre acompanhados de feedback visual equivalente
- Usuários surdos devem receber toda informação via visual
- Sons são complementares, nunca exclusivos
- Opção de vibrar dispositivo (mobile) como alternativa

**Contexto de Uso:**
- Detectar modo silencioso do dispositivo e não reproduzir sons
- Respeitar preferências de acessibilidade do sistema operacional
- Sons mais discretos em ambientes profissionais
- Opção de "Não perturbar" durante horários específicos

### 3. Feedback Tátil/Haptic (Mobile)

**Em dispositivos móveis, o sistema utiliza vibrações ou outros feedbacks táteis para informar o usuário?**

#### Elementos Táteis Identificados no Requisito

**Dispositivos Móveis:**
- ⚠️ Não especifica suporte a dispositivos móveis
- ⚠️ Não menciona uso de feedback tátil/vibração
- ⚠️ Não define padrões de vibração para diferentes eventos

#### Questões Levantadas

- ❓ Há vibração para confirmação de ações importantes?
- ❓ O feedback tátil é usado para erros ou validações?
- ❓ Há diferentes padrões de vibração para diferentes eventos?
- ❓ O feedback tátil é apropriado ao contexto?
- ❓ Há opção de desabilitar vibrações?
- ❓ O sistema utiliza feedback tátil para melhorar a acessibilidade?

#### Gaps Identificados

- ⚠️ **Falta de suporte móvel explícito**: Não especifica se sistema funciona em mobile
- ⚠️ **Falta de feedback tátil**: Não menciona uso de vibrações
- ⚠️ **Falta de padrões distintos**: Não define diferentes vibrações para diferentes eventos
- ⚠️ **Falta de controle**: Não menciona opção de desabilitar vibrações
- ⚠️ **Falta de acessibilidade tátil**: Não considera usuários com deficiência visual em mobile

#### Recomendações

**Confirmação de Ações:**
- Vibração curta e suave (100ms) quando transferência é concluída com sucesso
- Padrão: Uma vibração rápida e discreta
- Vibração apenas em ações críticas (não em todas as interações)

**Feedback de Erro:**
- Vibração dupla (200ms + 100ms pausa + 200ms) para erros críticos:
  - Saldo insuficiente
  - Destinatário inexistente
  - Erro de validação crítico
- Vibração distinta para alertar sobre problemas

**Validação:**
- Vibração leve (50ms) quando campo inválido é detectado (opcional)
- Não vibrar em cada erro de digitação (apenas em submit ou blur)

**Controles:**
- Configurações de vibração:
  - Habilitar/desabilitar vibrações
  - Escolher intensidade (baixa, média, alta)
  - Testar vibração antes de salvar
- Respeitar configurações de sistema (modo não perturbe)
- Opção "Não vibrar" para ambientes silenciosos

**Padrões Distintos:**
- **Sucesso**: Uma vibração curta (100ms)
- **Erro**: Duas vibrações rápidas (200ms + pausa + 200ms)
- **Aviso**: Uma vibração média (150ms)
- Padrões devem ser reconhecíveis e distintos

**Acessibilidade:**
- Feedback tátil como alternativa para usuários com deficiência visual
- Sempre acompanhado de feedback visual e auditivo
- Vibração não deve ser única forma de comunicação
- Opção de aumentar intensidade para usuários com deficiência sensorial

**Contexto:**
- Não vibrar em modo silencioso do dispositivo
- Respeitar configurações de "Não perturbe"
- Vibração apropriada ao contexto (não vibrar em reuniões)
- Opção de agendar horários sem vibração

### 4. Acessibilidade

**O sistema é projetado para ser utilizável por pessoas com deficiências (visuais, auditivas, motoras, cognitivas)? Ele segue as diretrizes de acessibilidade (WCAG, por exemplo)?**

#### Elementos de Acessibilidade Identificados no Requisito

**Navegação:**
- ⚠️ Não especifica navegação por teclado
- ⚠️ Não menciona suporte a leitores de tela
- ⚠️ Não define atalhos de teclado

**Estrutura:**
- ⚠️ Não especifica uso de HTML semântico
- ⚠️ Não menciona ARIA labels ou roles
- ⚠️ Não define estrutura acessível

**Contraste e Legibilidade:**
- ⚠️ Não especifica contraste de cores
- ⚠️ Não menciona tamanho de fonte ou opções de aumento
- ⚠️ Não define padrões de legibilidade

#### Questões Levantadas

- ❓ O sistema é navegável apenas com teclado?
- ❓ Há suporte para leitores de tela (screen readers)?
- ❓ Os elementos interativos têm labels descritivos?
- ❓ Há alternativas textuais para conteúdo não textual (imagens, ícones)?
- ❓ O contraste de cores atende aos padrões WCAG?
- ❓ Há opções de aumentar o tamanho da fonte?
- ❓ O sistema funciona bem com tecnologias assistivas?
- ❓ Há atalhos de teclado para ações frequentes?

#### Gaps Identificados

- ⚠️ **Falta de navegação por teclado**: Não especifica se todas funcionalidades são acessíveis via teclado
- ⚠️ **Falta de suporte a leitores de tela**: Não menciona ARIA labels, roles ou estados
- ⚠️ **Falta de contraste**: Não especifica se contraste atende padrões WCAG
- ⚠️ **Falta de alternativas textuais**: Não menciona alt text ou descrições para elementos não textuais
- ⚠️ **Falta de foco visível**: Não especifica indicador de foco claro
- ⚠️ **Falta de estrutura semântica**: Não menciona uso de HTML semântico
- ⚠️ **Falta de formulários acessíveis**: Não especifica labels associados ou mensagens acessíveis

#### Recomendações

**Navegação por Teclado:**
- Todas as funcionalidades acessíveis via teclado:
  - Tab para navegar entre campos
  - Enter para submeter formulário
  - Escape para fechar modais/toasts
  - Setas para navegar em listas/dropdowns
- Ordem de tabulação lógica e intuitiva
- Não usar apenas mouse/touch para ações críticas

**Suporte a Leitores de Tela:**
- ARIA labels descritivos:
  - `aria-label="Selecione o destinatário da transferência"` no campo recipient_id
  - `aria-label="Informe o valor em QualiPoints"` no campo amount
  - `aria-label="Transferir QualiPoints"` no botão de submit
- ARIA roles apropriados:
  - `role="alert"` para mensagens de erro críticas
  - `role="status"` para mensagens de sucesso
  - `role="button"` para elementos clicáveis não-button
- ARIA states:
  - `aria-required="true"` para campos obrigatórios
  - `aria-invalid="true"` para campos com erro
  - `aria-disabled="true"` para elementos desabilitados
  - `aria-live="polite"` para atualizações dinâmicas de saldo

**Contraste e Legibilidade:**
- Contraste mínimo WCAG AA:
  - Texto normal: 4.5:1
  - Texto grande (18pt+): 3:1
- Cores de erro/sucesso com contraste adequado
- Não usar apenas cor para comunicar informação (usar também ícone/texto)
- Opção de aumentar tamanho da fonte (até 200% sem quebrar layout)
- Modo de alto contraste disponível

**Alternativas Textuais:**
- Alt text descritivo para todos os ícones:
  - Ícone de check: `alt="Sucesso"`
  - Ícone de erro: `alt="Erro"`
  - Ícone de loading: `alt="Processando"`
- Descrições para elementos decorativos: `alt=""`
- Texto alternativo para gráficos ou visualizações de saldo
- Legendas para conteúdo multimídia (se aplicável)

**Foco Visível:**
- Indicador de foco claro e visível:
  - Borda azul sólida (3px) ao redor do elemento
  - Contraste alto (azul #3B82F6 em fundo branco)
  - Visível em todos os estados (hover, focus, active)
- Foco nunca removido ou escondido
- Estilo de foco consistente em todo o sistema

**Estrutura Semântica:**
- HTML semântico:
  - `<form>` para formulário de transferência
  - `<label>` associado a cada campo
  - `<fieldset>` e `<legend>` para agrupar campos relacionados
  - `<button>` para ações (não `<div>` ou `<span>`)
  - Headings hierárquicos (`<h1>`, `<h2>`, etc.)
- Landmarks ARIA:
  - `role="main"` para conteúdo principal
  - `role="form"` para formulário
  - `role="alert"` para mensagens críticas
  - `role="status"` para atualizações de status

**Formulários Acessíveis:**
- Labels associados a campos:
  - `<label for="recipient_id">Destinatário</label>`
  - `<label for="amount">Valor</label>`
- Mensagens de erro associadas a campos:
  - `aria-describedby` apontando para mensagem de erro
  - Mensagem de erro próxima ao campo (visualmente e semanticamente)
- Instruções claras antes de campos complexos
- Agrupamento lógico de campos relacionados

**Atalhos de Teclado:**
- Atalhos para ações frequentes:
  - Ctrl/Cmd + Enter: Submeter formulário
  - Ctrl/Cmd + K: Buscar destinatário (se aplicável)
  - Esc: Fechar modal/toast
- Atalhos documentados e não conflitantes com navegador
- Opção de desabilitar atalhos personalizados

**Tecnologias Assistivas:**
- Testar com leitores de tela populares:
  - NVDA (Windows)
  - JAWS (Windows)
  - VoiceOver (macOS/iOS)
  - TalkBack (Android)
- Garantir compatibilidade com tecnologias assistivas
- Documentar padrões de acessibilidade seguidos

### 5. Status e Progresso

**O usuário sabe sempre qual é o estado atual do sistema? Há indicadores de progresso claros para ações de longa duração?**

#### Elementos de Status Identificados no Requisito

**Estados do Sistema:**
- ⚠️ Não especifica como comunicar estado de conexão (online/offline)
- ⚠️ Não menciona indicadores de sincronização
- ⚠️ Não define comunicação de estado durante processamento

**Indicadores de Progresso:**
- ✅ Transação é síncrona (resposta em tempo real)
- ⚠️ Não especifica indicador visual durante processamento
- ⚠️ Não menciona estimativa de tempo ou feedback sobre o que está sendo processado

**Confirmações:**
- ✅ Resposta 201 com `transaction_id` e `new_balance` (sucesso)
- ⚠️ Não especifica como sucesso é comunicado visualmente
- ⚠️ Não menciona confirmação clara de conclusão

#### Questões Levantadas

- ❓ O sistema comunica claramente seu estado atual (conectado, offline, processando)?
- ❓ Há indicadores de progresso para ações que demoram?
- ❓ O usuário sabe quanto tempo falta para concluir uma ação?
- ❓ Há feedback sobre o que está sendo processado?
- ❓ O sistema informa quando está sincronizando ou atualizando dados?
- ❓ Há estados de erro claramente comunicados?
- ❓ O usuário sabe quando uma ação foi concluída com sucesso?

#### Gaps Identificados

- ⚠️ **Falta de comunicação de estado**: Não especifica como comunicar estado do sistema
- ⚠️ **Falta de indicadores de progresso**: Não menciona indicadores visuais durante processamento
- ⚠️ **Falta de feedback de processamento**: Não especifica mensagem sobre o que está sendo feito
- ⚠️ **Falta de confirmação clara**: Não define como comunicar conclusão de ações
- ⚠️ **Falta de estados de erro comunicados**: Não especifica como erros são comunicados claramente
- ⚠️ **Falta de sincronização**: Não menciona indicadores quando dados estão sendo sincronizados

#### Recomendações

**Estados do Sistema:**
- Badge de status sempre visível:
  - **Conectado**: Verde "Online" com ícone de conexão
  - **Offline**: Vermelho "Offline" com ícone de desconexão
  - **Sincronizando**: Amarelo "Sincronizando..." com spinner
- Tooltip explicativo ao passar mouse sobre status
- Notificação quando estado muda (ex: "Conexão perdida")

**Indicadores de Progresso:**
- Durante processamento da transferência:
  - Spinner animado no botão "Transferir"
  - Texto "Processando transferência..." com animação de pontos
  - Botão desabilitado para evitar duplo clique
  - Overlay sutil indicando que ação está em andamento
- Barra de progresso indeterminada se ação demorar mais de 2 segundos
- Mensagem de status: "Validando dados...", "Processando transferência...", "Atualizando saldo..."

**Feedback de Processamento:**
- Mensagens específicas sobre o que está sendo feito:
  - "Validando destinatário..."
  - "Verificando saldo disponível..."
  - "Processando transferência..."
  - "Atualizando saldos..."
- Mensagens aparecem em sequência lógica
- Usuário sempre sabe em qual etapa está

**Estados de Erro Comunicados:**
- Erros claramente comunicados com:
  - Ícone de erro visível
  - Mensagem específica e acionável
  - Destaque visual (cor vermelha, borda destacada)
  - Posicionamento próximo ao elemento relacionado
- Estados de erro:
  - **402**: "Saldo insuficiente. Seu saldo disponível é X QualiPoints"
  - **404**: "Destinatário não encontrado. Verifique o ID e tente novamente"
  - **422**: "Valor inválido. O valor mínimo é 1 QualiPoint"

**Confirmação de Conclusão:**
- Sucesso claramente comunicado:
  - Toast notification verde com ícone de check
  - Mensagem: "Transferência realizada com sucesso! Novo saldo: X QualiPoints"
  - Animação de sucesso (fade-in suave)
  - Atualização visual do saldo com destaque
- Confirmação permanece visível por tempo suficiente (5 segundos)
- Opção de fechar manualmente antes do tempo

**Sincronização:**
- Indicador quando dados estão sendo sincronizados:
  - Badge "Sincronizando..." no topo
  - Spinner discreto
  - Mensagem: "Sincronizando transações..."
- Notificação quando sincronização completa: "Dados atualizados"

**Timeout e Limites:**
- Aviso se processamento demorar muito:
  - "A transferência está demorando mais que o esperado. Aguarde..."
  - Opção de cancelar após 30 segundos (se aplicável)
- Mensagem clara sobre limites:
  - "Valor máximo: X QualiPoints (seu saldo disponível)"
  - "Valor mínimo: 1 QualiPoint"

**Consistência:**
- Mesmos padrões de comunicação de estado em todo o sistema
- Mesmos indicadores de progresso para ações similares
- Mesmas mensagens de confirmação para ações similares
- Documentar padrões de comunicação de status

### 6. Clareza das Mensagens

**As mensagens de erro, sucesso ou aviso são diretas, compreensíveis e, se aplicável, sugerem o próximo passo ou a solução?**

#### Mensagens Identificadas no Requisito

**Erros:**
- ✅ Erro 402 Payment Required: Saldo insuficiente
- ✅ Erro 404 Not Found: Destinatário inexistente
- ✅ Erro 422 Unprocessable Entity: Valor abaixo do mínimo ou formato inválido
- ⚠️ Códigos HTTP não são mensagens amigáveis ao usuário
- ⚠️ Não especifica mensagens de erro específicas e acionáveis

**Sucesso:**
- ✅ Resposta 201 com `transaction_id` e `new_balance`
- ⚠️ Não especifica mensagem de sucesso amigável ao usuário
- ⚠️ Não menciona próximo passo após sucesso

**Validações:**
- ⚠️ Não especifica mensagens de validação em tempo real
- ⚠️ Não menciona mensagens para campos obrigatórios vazios

#### Questões Levantadas

- ❓ As mensagens são escritas em linguagem clara e não técnica?
- ❓ As mensagens explicam o problema de forma específica?
- ❓ Há sugestões de como resolver o problema?
- ❓ As mensagens indicam o próximo passo quando aplicável?
- ❓ O tom das mensagens é apropriado (não culpabiliza o usuário)?
- ❓ Há diferenciação clara entre tipos de mensagem (erro, aviso, sucesso, informação)?
- ❓ As mensagens são contextualizadas (aparecem no lugar certo)?

#### Gaps Identificados

- ⚠️ **Falta de linguagem clara**: Códigos HTTP não são compreensíveis para usuários
- ⚠️ **Falta de especificidade**: Mensagens genéricas não explicam o problema específico
- ⚠️ **Falta de sugestões**: Mensagens não indicam como resolver o problema
- ⚠️ **Falta de próximo passo**: Mensagens não indicam o que fazer a seguir
- ⚠️ **Falta de tom apropriado**: Não especifica se mensagens são construtivas
- ⚠️ **Falta de diferenciação**: Não especifica diferenciação visual entre tipos de mensagem
- ⚠️ **Falta de contexto**: Não menciona onde mensagens devem aparecer

#### Recomendações

**Linguagem Clara:**
- Converter códigos HTTP em mensagens amigáveis:
  - **402**: "Saldo insuficiente" → "Você não tem saldo suficiente para esta transferência. Seu saldo disponível é X QualiPoints."
  - **404**: "Destinatário não encontrado" → "Não encontramos um usuário com este ID. Verifique o ID e tente novamente."
  - **422**: "Valor inválido" → "O valor informado não é válido. O valor mínimo é 1 QualiPoint."
- Evitar jargão técnico:
  - ❌ "Erro 402 Payment Required"
  - ✅ "Saldo insuficiente para realizar a transferência"
- Usar linguagem do usuário, não do desenvolvedor

**Especificidade:**
- Mensagens específicas sobre o problema:
  - ❌ "Erro ao processar"
  - ✅ "O valor informado (0.5) é menor que o mínimo permitido (1 QualiPoint)"
- Incluir valores específicos quando relevante:
  - "Você tentou transferir 100 QualiPoints, mas seu saldo disponível é apenas 50 QualiPoints"
- Identificar campo específico com problema:
  - "O campo 'Valor' está vazio. Informe o valor da transferência."

**Sugestões de Solução:**
- Mensagens com ação sugerida:
  - "Saldo insuficiente. Seu saldo disponível é X QualiPoints. Ajuste o valor e tente novamente."
  - "Destinatário não encontrado. Verifique o ID digitado ou busque por nome/email."
  - "O valor mínimo é 1 QualiPoint. Informe um valor maior ou igual a 1."
- Botões de ação quando aplicável:
  - "Ver meu saldo" (link para página de saldo)
  - "Buscar destinatário" (abrir busca)

**Próximo Passo:**
- Mensagens indicam o que fazer a seguir:
  - Sucesso: "Transferência realizada! Você pode ver os detalhes no histórico de transações."
  - Erro: "Corrija os campos destacados e tente novamente."
  - Validação: "Preencha todos os campos obrigatórios para continuar."
- Links ou botões para próximos passos quando relevante

**Tom Apropriado:**
- Mensagens construtivas, não culpabilizantes:
  - ❌ "Você digitou errado"
  - ✅ "O formato do valor não é válido. Use apenas números (ex: 100 ou 100.50)"
- Tom positivo quando possível:
  - ❌ "Não foi possível transferir"
  - ✅ "A transferência não pôde ser concluída. Verifique seu saldo e tente novamente."
- Evitar linguagem que culpa o usuário

**Diferenciação Visual:**
- Diferentes estilos para diferentes tipos:
  - **Erro**: Vermelho, ícone de alerta, borda vermelha
  - **Sucesso**: Verde, ícone de check, borda verde
  - **Aviso**: Amarelo, ícone de atenção, borda amarela
  - **Informação**: Azul, ícone de informação, borda azul
- Consistência visual em todo o sistema
- Ícones + cores + texto para redundância

**Contextualização:**
- Mensagens próximas ao elemento relacionado:
  - Erro de campo: Abaixo ou ao lado do campo
  - Erro de formulário: No topo do formulário + próximo aos campos
  - Sucesso: Toast notification + atualização inline
- Mensagens aparecem no momento certo:
  - Validação em tempo real: Durante digitação ou ao perder foco
  - Erro de submit: Após tentativa de envio
  - Sucesso: Imediatamente após conclusão

**Consistência:**
- Mesmo padrão de mensagens em todo o sistema
- Mesmo tom e estilo de escrita
- Mesma estrutura (problema + solução + próximo passo)
- Documentar guia de estilo de mensagens

## Checklist de Análise Seen and Heard

- [x] **Feedback Visual**: Indicações visuais claras sobre estados e ações, informações importantes visíveis e distinguíveis
- [x] **Feedback Auditivo**: Sons apropriados que complementam o feedback visual, especialmente para acessibilidade
- [x] **Feedback Tátil/Haptic**: Vibrações em dispositivos móveis para confirmação e notificações importantes
- [x] **Acessibilidade**: Sistema utilizável por pessoas com deficiências, seguindo diretrizes WCAG
- [x] **Status e Progresso**: Comunicação clara do estado atual do sistema, indicadores de progresso para ações longas
- [x] **Clareza das Mensagens**: Mensagens diretas, compreensíveis, com sugestões de solução quando aplicável
- [ ] **Navegação por Teclado**: Todas as funcionalidades acessíveis via teclado (requer implementação)
- [ ] **Suporte a Leitores de Tela**: ARIA labels, roles e estados adequados (requer implementação)
- [ ] **Contraste e Legibilidade**: Contraste mínimo adequado, texto legível (requer implementação)
- [ ] **Alternativas Textuais**: Alt text para imagens, descrições para conteúdo não textual (requer implementação)
- [ ] **Consistência**: Mesmos padrões de comunicação aplicados em todo o sistema (requer implementação)
- [ ] **Redundância**: Informações importantes comunicadas através de múltiplos canais (requer implementação)
- [ ] **Controles de Acessibilidade**: Opções para ajustar tamanho de fonte, desabilitar sons/vibrações (requer implementação)
- [ ] **Estrutura Semântica**: Uso adequado de HTML semântico e landmarks (requer implementação)
- [ ] **Formulários Acessíveis**: Labels associados, mensagens de erro acessíveis (requer implementação)

## Resumo de Gaps e Recomendações Prioritárias

### Críticos (Alto Impacto na Comunicação)

1. **Falta de Feedback Visual Imediato**: Implementar indicadores visuais claros para estados (sucesso, erro, processando), confirmações após ações e diferenciação visual entre tipos de feedback
2. **Mensagens Técnicas Não Amigáveis**: Converter códigos HTTP (402, 404, 422) em mensagens claras, específicas e acionáveis com sugestões de solução
3. **Falta de Acessibilidade Básica**: Implementar navegação por teclado, suporte a leitores de tela (ARIA), contraste adequado e estrutura semântica HTML

### Importantes (Médio Impacto na Comunicação)

4. **Falta de Indicadores de Status**: Implementar comunicação clara do estado do sistema (conectado, offline, processando) e indicadores de progresso durante ações
5. **Falta de Feedback Multimodal**: Implementar feedback auditivo e tátil como complemento ao visual, especialmente para acessibilidade
6. **Falta de Confirmação Clara**: Implementar confirmações visuais claras após ações bem-sucedidas com indicação de próximo passo

### Desejáveis (Baixo Impacto, Alto Refinamento)

7. **Feedback Auditivo Opcional**: Implementar sons discretos para confirmação e alertas com opção de desabilitar
8. **Feedback Tátil em Mobile**: Implementar vibrações para confirmação e erros críticos em dispositivos móveis
9. **Controles de Acessibilidade**: Implementar opções para ajustar tamanho de fonte, desabilitar sons/vibrações e modo de alto contraste
10. **Mensagens Contextualizadas**: Garantir que mensagens apareçam no lugar certo e no momento certo, próximas aos elementos relacionados

## Objetivo Final

Garantir que o sistema de transferência de QualiPoints comunique seu estado, ações e consequências de forma clara, oportuna e através de múltiplos canais sensoriais, criando uma experiência onde o usuário está sempre informado, no controle e capaz de interagir de forma eficaz, independentemente de suas capacidades ou do contexto de uso:

- **Feedback visual claro**: Usuário sempre sabe o que está acontecendo através de indicações visuais
- **Feedback multimodal**: Informações importantes comunicadas através de múltiplos canais (visual, auditivo, tátil)
- **Acessibilidade inclusiva**: Sistema utilizável por todos, independentemente de capacidades ou tecnologias assistivas
- **Status sempre visível**: Usuário sempre sabe o estado atual do sistema e o progresso de ações
- **Mensagens claras e acionáveis**: Mensagens compreensíveis que explicam problemas e sugerem soluções
- **Comunicação consistente**: Mesmos padrões de comunicação aplicados em todo o sistema

A análise Seen and Heard identificou gaps críticos de comunicação, barreiras de acessibilidade e oportunidades de inclusão que devem ser endereçados para garantir que o produto comunique de forma efetiva com todos os usuários, em qualquer condição, criando uma experiência verdadeiramente inclusiva e acessível.
