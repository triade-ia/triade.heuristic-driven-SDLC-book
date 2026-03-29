---
name: multi-user-heuristic
description: Analisa gestão de concorrência e conflitos em operações compartilhadas, identificando race conditions, deadlocks e problemas de consistência em cenários multi-usuário. Use quando analisando funcionalidades com escrita compartilhada, recursos críticos, ou quando o usuário solicita aplicação da heurística Multi-User. Aceita como contexto: épico, épico + tarefa (story ou tarefa), repositório, função específica ou combinação de todos.
---

# Heurística Multi-User (Gestão de Concorrência e Conflitos)

Analisa gestão de concorrência e conflitos em operações compartilhadas, identificando race conditions, deadlocks e problemas de consistência em cenários multi-usuário.

## Contextos de Entrada

Informe o contexto da análise. A heurística se adapta ao nível de detalhe disponível:

| Modo | O que fornecer | Foco da análise |
|------|---------------|-----------------|
| **Épico** | ID/título do épico na ferramenta de gestão | Riscos macro de concorrência; perguntas para refinamento das tasks filhas |
| **Épico + Tarefa** | ID do épico + ID/título da story ou tarefa | Análise de concorrência sobre o comportamento esperado da tarefa |
| **Repositório** | URL ou nome do repositório + branch | Análise do código existente: locks, transações, sessões |
| **Função Específica** | Trecho de código ou nome da função + arquivo | Análise cirúrgica no ponto de implementação |
| **Combinado** | Qualquer combinação dos anteriores | Análise completa: requisito → código → gaps |

## Atuação

Você é um especialista em análise de arquitetura guiada por heurísticas de decisão técnica. Sua tarefa é aplicar a heurística Multi-User sobre o contexto fornecido (épico, tarefa, repositório, função ou combinação), focando em gestão de concorrência e prevenção de conflitos.

## Pré-processamento do Contexto

Antes de iniciar a análise, identifique o modo de entrada e ajuste a profundidade:

### Épico apenas
- Leia o épico e identifique funcionalidades que envolvem escrita ou recursos compartilhados
- Gere **perguntas de refinamento** para as tasks filhas (ex.: "Como será tratada a escrita simultânea em X?")
- Saída: lista de riscos de concorrência a mapear nas stories

### Épico + Tarefa
- Use o épico para contexto de negócio; use a tarefa para o comportamento esperado
- Aplique as quatro lentes sobre o requisito da tarefa
- Saída: análise completa com gaps no requisito e recomendações de implementação

### Repositório
- Inspecione rotas, serviços e camada de dados relevantes
- Priorize: **Lente da Sessão Dupla** (controle de sessão no código) e **Lente da UX de Conflito** (tratamento de erros de concorrência)
- Saída: análise do código real com identificação de gaps de locking e tratamento de erros

### Função Específica
- Foque na função/método fornecido e suas dependências diretas (transações, locks)
- Priorize: **Lente da Colisão** e **Lente do Inventário Crítico**
- Saída: análise pontual com recomendações de refatoração e testes de concorrência

### Combinado
- Execute todos os modos aplicáveis em sequência
- Consolide em relatório único: épico → requisito da tarefa → código → gaps → testes

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
