# 📋 Análise de User Scenarios - Requisito: Envio de QualiPoints

## Requisito Analisado
**Implementar funcionalidade de envio de "QualiPoints" entre usuários.**

O usuário quer poder mandar pontos para um amigo. O sistema deve ter uma tela para isso, uma API que processa e funcionar no App. Tem que ser rápido e seguro.

---

## 🎭 Cenário 1: O Usuário Padrão (Foco em clareza e fluxo principal)

### A Persona e a Motivação
**Maria**, 28 anos, desenvolvedora de software, está em casa após o trabalho. Ela quer agradecer ao colega **João** por ter ajudado em um projeto compartilhando 50 QualiPoints como reconhecimento. É uma ação rotineira e planejada, sem urgência.

### O Ambiente e o Contexto
- **Local**: Casa, conectada via Wi-Fi estável
- **Dispositivo**: Smartphone Android/iOS com bateria suficiente
- **Horário**: 20h30, ambiente tranquilo
- **Contexto**: Usuário já conhece o app, já realizou transferências anteriormente

### O Fluxo de Valor (Happy Path)
1. Maria abre o app e faz login (já está autenticada)
2. Navega até a seção "Enviar QualiPoints"
3. Visualiza seu saldo atual (ex: 500 QualiPoints)
4. Digita o ID do destinatário: "joao.silva" ou seleciona de uma lista de contatos recentes
5. Informa o valor: 50 QualiPoints
6. Visualiza um resumo: "Enviar 50 QualiPoints para João Silva?"
7. Confirma a operação
8. Recebe feedback imediato: "✅ Transferência realizada com sucesso!"
9. Visualiza o novo saldo atualizado: 450 QualiPoints
10. A transação aparece na lista de transações recentes

### O "E se?" (Edge Cases de Negócio)
- **E se** Maria digitar um ID que não existe? → Sistema deve validar e mostrar: "Usuário não encontrado. Verifique o ID."
- **E se** Maria tentar enviar mais pontos do que possui? → Sistema deve bloquear e mostrar: "Saldo insuficiente. Você possui 500 QualiPoints."
- **E se** Maria digitar um valor negativo ou zero? → Sistema deve validar e mostrar: "Valor inválido. Informe um valor maior que zero."
- **E se** Maria tentar enviar para ela mesma? → Sistema deve detectar e mostrar: "Você não pode enviar pontos para si mesmo."

---

## ⚡ Cenário 2: O Usuário sob Pressão/Estresse (Foco em performance e resiliência mobile/web)

### A Persona e a Motivação
**Carlos**, 35 anos, gerente de projetos, está em uma reunião importante e precisa rapidamente enviar 100 QualiPoints para um membro da equipe como incentivo imediato. Ele está com pressa e precisa que a operação seja concluída em segundos.

### O Ambiente e o Contexto
- **Local**: Escritório, conectado via rede corporativa instável
- **Dispositivo**: Smartphone com bateria baixa (15%), tela pequena
- **Horário**: 14h00, durante reunião, multitasking
- **Contexto**: Usuário está distraído, precisa fazer rápido, conexão pode cair a qualquer momento

### O Fluxo de Valor (Happy Path)
1. Carlos abre o app rapidamente (app já está em background)
2. Acessa "Enviar QualiPoints" (atalho ou widget na tela inicial)
3. Visualiza saldo rapidamente (cache local mostra saldo aproximado)
4. Digita parcialmente o ID: "mar" → Sistema sugere "maria.santos" via autocomplete
5. Seleciona o destinatário da lista
6. Usa um botão rápido "Enviar 100" (valor pré-definido)
7. Confirma com um toque rápido
8. Operação processa mesmo com conexão instável (retry automático)
9. Recebe notificação push: "Transferência concluída"
10. Pode continuar a reunião sem precisar verificar

### O "E se?" (Edge Cases de Negócio)
- **E se** a conexão cair durante o envio? → Sistema deve salvar localmente e retentar automaticamente quando a conexão voltar, mostrando: "Processando... Sua transferência será concluída quando a conexão for restaurada."
- **E se** o app travar ou fechar acidentalmente? → Sistema deve manter o estado da transação e permitir retomar de onde parou
- **E se** Carlos enviar para o usuário errado por pressa? → Sistema deve ter um período de cancelamento (ex: 30 segundos) ou confirmação dupla para valores acima de um limite
- **E se** o servidor estiver lento? → Sistema deve mostrar feedback de progresso: "Processando... Isso pode levar alguns segundos"
- **E se** Carlos estiver offline? → Sistema deve permitir criar a transação offline e sincronizar quando voltar online, com indicação clara: "Modo offline - será enviado quando conectado"

---

## 🔒 Cenário 3: O Usuário Mal Intencionado ou Confuso (Foco em segurança e tratamento de erro)

### A Persona e a Motivação
**Ana**, 22 anos, estudante, recebeu uma mensagem suspeita pedindo para enviar QualiPoints para um "usuário oficial" para "verificar sua conta". Ela está confusa e pode estar sendo vítima de uma tentativa de golpe. Alternativamente, pode ser um usuário tentando explorar vulnerabilidades do sistema.

### O Ambiente e o Contexto
- **Local**: Qualquer lugar, conexão variável
- **Dispositivo**: Smartphone compartilhado ou dispositivo público
- **Horário**: Qualquer horário
- **Contexto**: Usuário pode estar sendo enganado ou tentando explorar o sistema

### O Fluxo de Valor (Happy Path)
1. Ana tenta acessar a funcionalidade de envio
2. Sistema verifica autenticação (se não estiver logada, redireciona para login)
3. Ana tenta enviar para um ID suspeito ou inválido
4. Sistema valida o destinatário e mostra aviso se necessário
5. Ana tenta enviar um valor muito alto ou suspeito
6. Sistema aplica limites de segurança e solicita confirmação adicional
7. Operação é registrada com logs de auditoria
8. Sistema monitora padrões suspeitos

### O "E se?" (Edge Cases de Negócio)
- **E se** Ana tentar enviar para um ID que não existe repetidamente? → Sistema deve detectar tentativas suspeitas e limitar: "Muitas tentativas inválidas. Tente novamente em 5 minutos."
- **E se** Ana tentar enviar valores muito altos repetidamente? → Sistema deve aplicar rate limiting e exigir verificação adicional (2FA) para valores acima de um limite
- **E se** Ana tentar explorar uma vulnerabilidade de SQL injection no campo de ID? → Sistema deve sanitizar inputs e validar formato: "ID inválido. Use apenas letras, números e pontos."
- **E se** Ana tentar enviar pontos usando uma sessão expirada ou token inválido? → Sistema deve invalidar a sessão e exigir novo login: "Sessão expirada. Faça login novamente."
- **E se** Ana tentar enviar pontos para uma conta bloqueada ou inativa? → Sistema deve verificar status do destinatário e bloquear: "Conta do destinatário está inativa ou bloqueada."
- **E se** Ana tentar fazer múltiplas transferências simultâneas para o mesmo destinatário? → Sistema deve detectar e aplicar cooldown: "Aguarde alguns segundos antes de enviar novamente para o mesmo destinatário."
- **E se** Ana tentar enviar pontos usando automação/bot? → Sistema deve implementar CAPTCHA ou verificação humana após N tentativas
- **E se** Ana tentar enviar pontos para uma conta que ela criou falsamente? → Sistema deve validar identidade do destinatário e aplicar políticas anti-fraude

---

## 🎯 Lacunas de Decisão Identificadas

### 1. **Feedback e Confirmação Visual**
- **Lacuna**: O requisito não especifica como o usuário será informado sobre o sucesso/falha da operação
- **Decisão Necessária**: Definir mecanismos de feedback (notificações push, mensagens na tela, emails, etc.)
- **Impacto**: Usuário pode não saber se a transferência foi concluída, especialmente em cenários de conexão instável

### 2. **Tratamento de Timeout e Reconexão**
- **Lacuna**: O requisito não menciona o que acontece se a conexão cair durante o envio
- **Decisão Necessária**: Implementar retry automático, salvamento local de transações pendentes, e sincronização quando a conexão for restaurada
- **Impacto**: Transações podem ser perdidas ou ficar em estado inconsistente

### 3. **Validação e Limites de Segurança**
- **Lacuna**: O requisito não define limites de valor, rate limiting, ou validações de segurança
- **Decisão Necessária**: Estabelecer limites diários/mensais por usuário, validação de formato de ID, detecção de padrões suspeitos, e autenticação adicional para valores altos
- **Impacto**: Sistema vulnerável a fraudes, golpes e exploração de vulnerabilidades

### 4. **Estados de Erro e Mensagens ao Usuário**
- **Lacuna**: O requisito não especifica como tratar erros (saldo insuficiente, usuário inexistente, conta bloqueada, etc.)
- **Decisão Necessária**: Mapear todos os possíveis estados de erro e definir mensagens claras e acionáveis para cada um
- **Impacto**: Usuário pode ficar confuso ou não saber como proceder em caso de erro

### 5. **Cancelamento e Reversão de Transações**
- **Lacuna**: O requisito não menciona se é possível cancelar uma transação após o envio
- **Decisão Necessária**: Definir política de cancelamento (janela de tempo, aprovação do destinatário, etc.) e mecanismo de reversão
- **Impacto**: Erros de digitação ou envios acidentais não podem ser corrigidos

### 6. **Histórico e Auditoria**
- **Lacuna**: O requisito menciona "lista de transações recentes" mas não define escopo (quantas? por quanto tempo? filtros?)
- **Decisão Necessária**: Definir quantas transações mostrar, período de retenção, filtros (por data, destinatário, valor), e logs de auditoria para segurança
- **Impacto**: Usuário pode não conseguir rastrear suas transações adequadamente

### 7. **Performance e Otimização**
- **Lacuna**: O requisito diz "tem que ser rápido" mas não define métricas ou estratégias de otimização
- **Decisão Necessária**: Definir tempo máximo de resposta esperado, cache de saldo, otimização de queries, e estratégias de lazy loading
- **Impacto**: Sistema pode ser lento em cenários de alta carga ou conexão instável

### 8. **Autenticação e Autorização**
- **Lacuna**: O requisito diz "usuário deve estar logado" mas não especifica mecanismos de autenticação ou expiração de sessão
- **Decisão Necessária**: Definir política de sessão, refresh tokens, 2FA para operações sensíveis, e validação de permissões
- **Impacto**: Sistema vulnerável a acesso não autorizado ou sessões comprometidas

---

## 📊 Resumo Executivo

Esta análise revelou **8 lacunas críticas** no requisito original que precisam ser endereçadas antes da implementação:

1. ✅ Feedback e confirmação visual
2. ✅ Tratamento de timeout e reconexão  
3. ✅ Validação e limites de segurança
4. ✅ Estados de erro e mensagens
5. ✅ Cancelamento e reversão
6. ✅ Histórico e auditoria
7. ✅ Performance e otimização
8. ✅ Autenticação e autorização

Cada lacuna representa um risco potencial para a experiência do usuário, segurança do sistema ou escalabilidade da solução.
