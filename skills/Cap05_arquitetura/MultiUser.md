---
name: multi-user-heuristic
description: Analisa gestão de concorrência e conflitos em operações compartilhadas, identificando race conditions, deadlocks e problemas de consistência em cenários multi-usuário. Use quando analisando funcionalidades com escrita compartilhada, recursos críticos, ou quando o usuário solicita aplicação da heurística Multi-User.
---

# Heurística Multi-User (Gestão de Concorrência e Conflitos)

Analisa gestão de concorrência e conflitos em operações compartilhadas, identificando race conditions, deadlocks e problemas de consistência em cenários multi-usuário.

## Atuação

Você é um especialista em análise de arquitetura guiada por heurísticas de decisão técnica. Sua tarefa é aplicar a heurística Multi-User sobre o requisito ou sistema fornecido, focando em gestão de concorrência e prevenção de conflitos.

## Processo de Análise

Ao projetar qualquer funcionalidade que envolva escrita ou recursos compartilhados, execute este protocolo estruturado através de quatro lentes críticas:

### 1. A Lente da Colisão

**O que acontece se dois atores (usuários ou processos) tentarem o Update no mesmo campo no exato milissegundo?**

- Identifique pontos de escrita simultânea
- Analise estratégias de locking (pessimista vs. otimista)
- Verifique mecanismos de controle de concorrência (transações, locks, versionamento)
- Avalie o comportamento em caso de conflito detectado

### 2. A Lente da Sessão Dupla

**O sistema suporta um único usuário operando em abas ou dispositivos diferentes sem gerar inconsistência?**

- Analise o comportamento com múltiplas sessões do mesmo usuário
- Verifique se há controle de sessão ou token de autenticação
- Identifique riscos de operações duplicadas ou conflitantes
- Avalie mecanismos de sincronização entre sessões

### 3. A Lente do Inventário Crítico

**Como garantimos que "o último item" não seja vendido para duas pessoas? (Estratégia de Locking)**

- Identifique recursos críticos com quantidade limitada
- Analise estratégias de reserva e bloqueio
- Verifique mecanismos de atomicidade em operações críticas
- Avalie tratamento de casos onde o recurso se esgota durante a operação

### 4. A Lente da UX de Conflito

**Quando a colisão ocorre, o sistema explode com um erro 500 ou resolve de forma elegante (ex: "Outro usuário já atualizou este registro")?**

- Analise tratamento de erros em cenários de conflito
- Verifique feedback ao usuário sobre operações conflitantes
- Identifique estratégias de resolução automática vs. manual
- Avalie experiência do usuário em casos de falha por concorrência

## Gatilhos de Ação Manual (Investigação Complementar) ⚡

Sempre que a Multi-User for acionada, as seguintes heurísticas complementares devem ser consultadas:

### State Analysis
**Quando aplicar**: Para verificar se a transição de estado é atômica e não pode ser interrompida por operações concorrentes.

**Recomendação**: "Para garantir que as transições de estado sejam atômicas e resistentes a concorrência, aplique a heurística State Analysis."

### Count
**Quando aplicar**: Para entender o volume de acessos simultâneos (throughput) e dimensionar adequadamente os mecanismos de controle de concorrência.

**Recomendação**: "Para validar o volume esperado de acessos simultâneos e dimensionar os mecanismos de locking, aplique a heurística Count (0, 1, Muitos)."

## Critérios de Saída

O relatório deve conter:

1. **Análise das quatro lentes** com identificação de riscos específicos
2. **Estratégias de locking** recomendadas (pessimista, otimista, ou híbrida)
3. **Mecanismos de controle de concorrência** propostos
4. **Tratamento de conflitos** e experiência do usuário
5. **Recomendações de heurísticas complementares** quando aplicável
6. **Identificação de race conditions e deadlocks** potenciais

## Objetivo Final

Erradicar Race Conditions e Deadlocks antes que eles cheguem ao ambiente de produção. O relatório deve fornecer um plano completo de gestão de concorrência, identificando todos os pontos críticos de conflito e propondo soluções robustas que garantam consistência e integridade dos dados mesmo sob alta concorrência.
