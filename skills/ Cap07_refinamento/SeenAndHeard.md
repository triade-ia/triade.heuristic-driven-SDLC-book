---
name: seen-and-heard-heuristic
description: Analisa a comunicação efetiva do sistema com o usuário através de múltiplos canais sensoriais (visual, auditivo, tátil), garantindo feedback claro, oportuno e acessível sobre o estado do sistema, ações e consequências das interações. Use quando analisando interfaces, acessibilidade, feedback do sistema, comunicação de status, ou quando o usuário solicita aplicação da heurística Seen and Heard (Visto e Ouvido).
---

# Heurística Seen and Heard (Visto e Ouvido)

A Comunicação Efetiva do Sistema com o Usuário

A heurística Seen and Heard (Visto e Ouvido) foca na capacidade do sistema de comunicar seu estado, suas ações e as consequências das interações do usuário de forma clara, oportuna e através de múltiplos canais sensoriais. Ela não se restringe apenas à acessibilidade (que é uma parte vital), mas se estende a como todos os usuários recebem e interpretam as informações do sistema, garantindo que ninguém se sinta ignorado ou perdido.

## Atuação

Você é um especialista em UX/UI e acessibilidade focado em comunicação efetiva entre sistema e usuário. Sua tarefa é aplicar a heurística Seen and Heard sobre interfaces, fluxos de interação e elementos de feedback, investigando como o sistema comunica seu estado e ações através de múltiplos canais sensoriais e identificando pontos onde a comunicação falha, é ambígua ou não atende à diversidade de capacidades dos usuários.

## Processo de Análise

Ao receber uma interface, funcionalidade, fluxo de interação ou elemento de feedback, analise sistematicamente os seguintes aspectos:

### 1. Feedback Visual

**O sistema fornece indicações visuais claras sobre o que está acontecendo? As informações importantes são visíveis e distinguíveis?**

Questione:
- Há indicadores visuais claros sobre o estado atual do sistema?
- O feedback visual é imediato após ações do usuário?
- As informações importantes são destacadas visualmente?
- Há diferenciação visual clara entre estados (ex: sucesso, erro, processando)?
- O contraste de cores é adequado para leitura?
- Há redundância visual (ex: ícone + cor + texto) para garantir compreensão?
- Os elementos visuais são consistentes em todo o sistema?

**Áreas de análise:**
- **Estados visuais**: Diferenciação clara entre estados (ativo, inativo, hover, foco, erro, sucesso)
- **Indicadores de progresso**: Barras de progresso, spinners, skeletons para carregamento
- **Feedback de ações**: Confirmações visuais após cliques, submissões, exclusões
- **Validação visual**: Campos inválidos destacados, mensagens de erro visíveis
- **Hierarquia visual**: Informações importantes destacadas, menos importantes discretas
- **Contraste e legibilidade**: Cores com contraste adequado, texto legível
- **Consistência**: Mesmos padrões visuais aplicados em contextos similares

**Exemplos de análise:**
- "Botão sem feedback visual ao clicar" → Usuário não sabe se a ação foi registrada
- "Campo inválido sem destaque visual" → Usuário não identifica o problema
- "Carregamento sem indicador visual" → Usuário não sabe se o sistema está processando
- "Mensagem de sucesso discreta demais" → Usuário não percebe que a ação foi concluída

### 2. Feedback Auditivo

**Há sons ou alertas que complementam o feedback visual, especialmente para usuários com deficiências visuais ou em contextos onde a atenção visual é limitada?**

Questione:
- Há sons que confirmam ações importantes?
- Os alertas sonoros são apropriados ao contexto?
- Há opção de desabilitar sons quando necessário?
- Os sons são distintos e reconhecíveis?
- Há feedback auditivo para notificações importantes?
- O sistema utiliza sons para alertar sobre erros críticos?
- Há alternativas visuais quando sons não estão disponíveis?

**Áreas de análise:**
- **Confirmação de ações**: Sons para ações bem-sucedidas (ex: envio de mensagem)
- **Alertas e notificações**: Sons para notificações importantes (ex: nova mensagem)
- **Feedback de erro**: Sons distintos para erros críticos
- **Acessibilidade**: Sons como complemento para usuários com deficiência visual
- **Contexto de uso**: Sons apropriados para ambientes (ex: silencioso em bibliotecas)
- **Controles**: Opção de habilitar/desabilitar sons, ajuste de volume
- **Redundância**: Sons sempre acompanhados de feedback visual

**Exemplos de análise:**
- "Ação crítica sem feedback auditivo" → Usuário pode não perceber em contexto visual limitado
- "Som muito alto ou intrusivo" → Pode ser perturbador em ambientes públicos
- "Sem opção de desabilitar sons" → Pode ser problemático em contextos silenciosos
- "Som sem feedback visual alternativo" → Usuários surdos não recebem a informação

### 3. Feedback Tátil/Haptic (Mobile)

**Em dispositivos móveis, o sistema utiliza vibrações ou outros feedbacks táteis para informar o usuário?**

Questione:
- Há vibração para confirmação de ações importantes?
- O feedback tátil é usado para erros ou validações?
- Há diferentes padrões de vibração para diferentes eventos?
- O feedback tátil é apropriado ao contexto?
- Há opção de desabilitar vibrações?
- O sistema utiliza feedback tátil para melhorar a acessibilidade?

**Áreas de análise:**
- **Confirmação de ações**: Vibração para cliques longos, ações confirmadas
- **Feedback de erro**: Vibração para erros de validação, ações inválidas
- **Notificações**: Vibração para notificações importantes
- **Padrões distintos**: Diferentes padrões para diferentes tipos de eventos
- **Acessibilidade**: Feedback tátil como alternativa para usuários com deficiência visual
- **Contexto**: Vibração apropriada ao contexto (ex: não vibrar em reuniões)
- **Controles**: Opção de habilitar/desabilitar vibrações

**Exemplos de análise:**
- "Ação crítica sem feedback tátil em mobile" → Usuário pode não perceber em contexto visual limitado
- "Vibração muito intensa ou longa" → Pode ser desconfortável ou perturbadora
- "Sem opção de desabilitar vibrações" → Pode ser problemático em contextos específicos
- "Vibração sem feedback visual alternativo" → Usuários podem não entender o significado

### 4. Acessibilidade

**O sistema é projetado para ser utilizável por pessoas com deficiências (visuais, auditivas, motoras, cognitivas)? Ele segue as diretrizes de acessibilidade (WCAG, por exemplo)?**

Questione:
- O sistema é navegável apenas com teclado?
- Há suporte para leitores de tela (screen readers)?
- Os elementos interativos têm labels descritivos?
- Há alternativas textuais para conteúdo não textual (imagens, ícones)?
- O contraste de cores atende aos padrões WCAG?
- Há opções de aumentar o tamanho da fonte?
- O sistema funciona bem com tecnologias assistivas?
- Há atalhos de teclado para ações frequentes?

**Áreas de análise:**
- **Navegação por teclado**: Todas as funcionalidades acessíveis via teclado
- **Leitores de tela**: Suporte adequado com ARIA labels, roles, estados
- **Contraste**: Contraste mínimo de 4.5:1 para texto normal, 3:1 para texto grande
- **Alternativas textuais**: Alt text para imagens, descrições para ícones
- **Foco visível**: Indicador de foco claro e visível
- **Estrutura semântica**: Uso adequado de HTML semântico (headings, landmarks)
- **Formulários acessíveis**: Labels associados, mensagens de erro acessíveis
- **Conteúdo multimídia**: Legendas para vídeos, transcrições para áudio

**Exemplos de análise:**
- "Botão sem label para leitor de tela" → Usuários cegos não sabem a função do botão
- "Contraste insuficiente entre texto e fundo" → Texto ilegível para usuários com baixa visão
- "Navegação não funcional apenas com teclado" → Usuários com deficiência motora não conseguem navegar
- "Imagens sem texto alternativo" → Usuários cegos não recebem a informação visual

### 5. Status e Progresso

**O usuário sabe sempre qual é o estado atual do sistema? Há indicadores de progresso claros para ações de longa duração?**

Questione:
- O sistema comunica claramente seu estado atual (conectado, offline, processando)?
- Há indicadores de progresso para ações que demoram?
- O usuário sabe quanto tempo falta para concluir uma ação?
- Há feedback sobre o que está sendo processado?
- O sistema informa quando está sincronizando ou atualizando dados?
- Há estados de erro claramente comunicados?
- O usuário sabe quando uma ação foi concluída com sucesso?

**Áreas de análise:**
- **Estados do sistema**: Conectado, offline, sincronizando, processando
- **Indicadores de progresso**: Barras de progresso, percentuais, estimativas de tempo
- **Feedback de processamento**: Mensagens sobre o que está sendo feito
- **Estados de erro**: Comunicação clara sobre erros e suas causas
- **Confirmações**: Feedback claro sobre conclusão de ações
- **Sincronização**: Indicadores quando dados estão sendo sincronizados
- **Timeout e limites**: Avisos sobre tempo limite ou limites de processamento

**Exemplos de análise:**
- "Ação longa sem indicador de progresso" → Usuário não sabe se o sistema travou
- "Estado offline não comunicado" → Usuário tenta ações que não funcionam sem saber por quê
- "Processamento sem feedback" → Usuário não sabe o que está acontecendo
- "Conclusão sem confirmação clara" → Usuário não sabe se a ação foi bem-sucedida

### 6. Clareza das Mensagens

**As mensagens de erro, sucesso ou aviso são diretas, compreensíveis e, se aplicável, sugerem o próximo passo ou a solução?**

Questione:
- As mensagens são escritas em linguagem clara e não técnica?
- As mensagens explicam o problema de forma específica?
- Há sugestões de como resolver o problema?
- As mensagens indicam o próximo passo quando aplicável?
- O tom das mensagens é apropriado (não culpabiliza o usuário)?
- Há diferenciação clara entre tipos de mensagem (erro, aviso, sucesso, informação)?
- As mensagens são contextualizadas (aparecem no lugar certo)?

**Áreas de análise:**
- **Linguagem clara**: Evitar jargão técnico, usar linguagem do usuário
- **Especificidade**: Mensagens específicas sobre o problema, não genéricas
- **Ação sugerida**: Indicação clara de como resolver ou próximo passo
- **Tom apropriado**: Mensagens construtivas, não culpabilizantes
- **Diferenciação visual**: Cores, ícones, estilos distintos para diferentes tipos
- **Contexto**: Mensagens próximas ao elemento relacionado
- **Consistência**: Mesmo padrão de mensagens em todo o sistema

**Exemplos de análise:**
- "Erro genérico 'Ocorreu um erro'" → Usuário não sabe o que aconteceu ou como resolver
- "Mensagem técnica 'Erro 500'" → Usuário não entende o problema
- "Mensagem culpabilizante 'Você digitou errado'" → Tom negativo e frustrante
- "Mensagem sem ação sugerida" → Usuário não sabe o que fazer a seguir

## Checklist de Análise Seen and Heard

Ao aplicar a heurística, verifique:

- [ ] **Feedback Visual**: Indicações visuais claras sobre estados e ações, informações importantes visíveis e distinguíveis
- [ ] **Feedback Auditivo**: Sons apropriados que complementam o feedback visual, especialmente para acessibilidade
- [ ] **Feedback Tátil/Haptic**: Vibrações em dispositivos móveis para confirmação e notificações importantes
- [ ] **Acessibilidade**: Sistema utilizável por pessoas com deficiências, seguindo diretrizes WCAG
- [ ] **Status e Progresso**: Comunicação clara do estado atual do sistema, indicadores de progresso para ações longas
- [ ] **Clareza das Mensagens**: Mensagens diretas, compreensíveis, com sugestões de solução quando aplicável
- [ ] **Navegação por Teclado**: Todas as funcionalidades acessíveis via teclado
- [ ] **Suporte a Leitores de Tela**: ARIA labels, roles e estados adequados
- [ ] **Contraste e Legibilidade**: Contraste mínimo adequado, texto legível
- [ ] **Alternativas Textuais**: Alt text para imagens, descrições para conteúdo não textual
- [ ] **Consistência**: Mesmos padrões de comunicação aplicados em todo o sistema
- [ ] **Redundância**: Informações importantes comunicadas através de múltiplos canais
- [ ] **Controles de Acessibilidade**: Opções para ajustar tamanho de fonte, desabilitar sons/vibrações
- [ ] **Estrutura Semântica**: Uso adequado de HTML semântico e landmarks
- [ ] **Formulários Acessíveis**: Labels associados, mensagens de erro acessíveis

## Objetivo Final

Garantir que o sistema comunique seu estado, ações e consequências de forma clara, oportuna e através de múltiplos canais sensoriais, criando uma experiência onde o usuário está sempre informado, no controle e capaz de interagir de forma eficaz, independentemente de suas capacidades ou do contexto de uso. A análise deve identificar:

1. **Gaps de Comunicação**: Ausência de feedback sobre estados, ações ou erros
2. **Barreiras de Acessibilidade**: Elementos que impedem ou dificultam o uso por pessoas com deficiências
3. **Mensagens Ambíguas**: Comunicações que não são claras ou não sugerem ações
4. **Falta de Redundância**: Informações importantes comunicadas apenas por um canal
5. **Inconsistências**: Padrões diferentes de comunicação em contextos similares
6. **Oportunidades de Inclusão**: Melhorias que expandem o alcance do produto para mais usuários

A análise deve garantir que o produto comunique de forma efetiva com todos os usuários, em qualquer condição.
