---
name: dependencies-heuristic
description: Analisa relacionamentos "tem um" (has-a) e conexões sistêmicas. Além de mapear riscos de acoplamento, atua como um guia indicando quais heurísticas complementares (CRUD, Count, Position, Selection) devem ser aplicadas em cada relacionamento identificado para uma análise completa. Use quando analisando arquitetura, mapeando dependências entre serviços, ou quando o usuário solicita aplicação da heurística Dependencies.
---

# Heurística Dependencies

Analisa relacionamentos "tem um" (has-a) e conexões sistêmicas. Além de mapear riscos de acoplamento, atua como um guia, indicando quais heurísticas complementares (CRUD, Count, etc.) devem ser aplicadas em cada relacionamento identificado para uma análise completa.

## Processo de Análise

Com base na funcionalidade, gere o output estruturado nas seguintes seções:

### 1. Mapeamento de Relacionamentos (O "Tem um")

Identifique as entidades e serviços envolvidos:

#### Relacionamento de Dados
Como as entidades se conectam? (Referência por ID, Duplicação/Denormalização).

#### Dependência de Fluxo
Quais serviços são vitais para o sucesso da operação?

### 2. Matriz de Propagação de Impacto (O "Efeito Dominó")

Analise a resiliência da corrente:

#### Mudança na Origem
Riscos de inconsistência se o dado principal mudar.

#### Falha de Vizinho
Morte da funcionalidade vs. Degradação Graciosa.

#### Concorrência
Gargalos em recursos compartilhados (ex: contenção de tabela).

### 3. Gatilhos de Ação Manual (Investigação Complementar) ⚡

Esta é a seção mais importante para a continuidade da análise. Para cada relacionamento ou dependência identificada acima, você deve emitir uma recomendação de ação manual para o usuário:

- **Se houver persistência de dados**: "Para complementar a análise do relacionamento [X], aplique a heurística CRUD."

- **Se houver coleções, listas ou volumes de dados**: "Para validar os limites e a escalabilidade de [X], aplique a heurística Count (0, 1, Muitos)."

- **Se houver ordenação ou listas encadeadas**: "Para investigar o comportamento de [X], aplique a heurística Position (Primeiro, Último, Meio)."

- **Se houver filtros ou estados variáveis**: "Para garantir a integridade da busca em [X], aplique a heurística Selection (Alguns, Nenhum, Todos)."

### 4. Decisões de Desacoplamento e Resiliência

Proponha estratégias de escalabilidade:

#### Isolamento
Como impedir que a falha de um elo bloqueie o usuário.

#### Contrato
Comunicação Síncrona vs. Assíncrona.

## Objetivo Final

Identificar acoplamentos ocultos e fornecer um mapa de roteiro. O relatório deve indicar não apenas os riscos atuais, mas orientar o próximo passo do processo investigativo, direcionando o usuário para as heurísticas complementares necessárias para blindar o requisito.
