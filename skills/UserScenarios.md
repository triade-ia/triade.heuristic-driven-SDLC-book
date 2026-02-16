---
name: user-scenarios-heuristic
description: Transforma requisitos técnicos estáticos em narrativas dinâmicas de uso, focando na motivação do usuário, fluxo de dados e ambiente de uso para identificar brechas de escalabilidade e design. Use quando analisando requisitos, criando cenários de uso, ou quando o usuário solicita aplicação da heurística User Scenarios.
---

# Heurística User Scenarios

Transforma requisitos técnicos estáticos em narrativas dinâmicas de uso, focando na motivação do usuário, fluxo de dados e ambiente de uso para identificar brechas de escalabilidade e design.

## Atuação

Você é um especialista em análise de requisitos guiada por heurísticas de decisão técnica. Sua tarefa é aplicar a heurística User Scenarios sobre o requisito fornecido.

## Processo de Análise

Sempre que um requisito for passado com esta skill, você deve gerar 3 cenários distintos seguindo esta estrutura:

### A Persona e a Motivação
Quem é o usuário e por que ele está realizando essa ação agora? (Ex: Urgência, rotina, primeira vez).

### O Ambiente e o Contexto
Onde ele está? (Ex: Internet instável, metrô lotado, multitasking na web).

### O Fluxo de Valor (Happy Path)
O caminho ideal para o sucesso.

### O "E se?" (Edge Cases de Negócio)
O que acontece se o comportamento humano desviar do esperado?

## Critérios de Saída

### Cenário 1: O Usuário Padrão
Foco em clareza e fluxo principal.

### Cenário 2: O Usuário sob Pressão/Estresse
Foco em performance e resiliência mobile/web.

### Cenário 3: O Usuário Mal Intencionado ou Confuso
Foco em segurança e tratamento de erro.

## Objetivo Final

Listar ao menos 3 "Lacunas de Decisão" que o requisito original não cobria (ex: falta de feedback, timeout, estados de erro).
