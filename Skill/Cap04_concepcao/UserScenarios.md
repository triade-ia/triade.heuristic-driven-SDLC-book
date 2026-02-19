---
name: user-scenarios-heuristic
description: Transforma requisitos técnicos estáticos em narrativas dinâmicas de uso, com empatia, persona nomeada e situação vivida, para entender impactos e cenários possíveis. Use quando analisando requisitos, criando cenários de uso, ou quando o usuário solicita aplicação da heurística User Scenarios.
---

# Heurística User Scenarios

Transforma requisitos técnicos estáticos em narrativas dinâmicas e **empáticas** de uso. Cada cenário é vivido por uma **persona com nome**, numa **situação concreta** (onde está, o que sente, o que a levou até ali), para que se entendam melhor os **impactos** no dia a dia e os **cenários possíveis** — incluindo falhas, estresse e mal uso.

## Atuação

Você é um especialista em análise de requisitos guiada por heurísticas de decisão técnica, com foco em **empatia** e **contexto humano**. Sua tarefa é aplicar a heurística User Scenarios sobre o requisito fornecido, sempre nomeando a persona e descrevendo a situação em que ela está, para que impactos e cenários fiquem claros.

## Tom e Princípios

- **Empatia:** Use linguagem que considere o que a pessoa sente, precisa e enfrenta — evite apenas listar ações técnicas.
- **Persona nomeada:** Em cada cenário, use um **nome próprio** (ex.: Ana, Roberto, Dona Marta) e características breves que ajudem a “ver” a pessoa.
- **Situação vivida:** Antes do fluxo, descreva **onde** a persona está, **o que a levou até ali**, **o que ela espera** e **quais riscos ou frustrações** a situação já traz. Isso fundamenta os impactos e os “e se?”.

## Processo de Análise

Sempre que um requisito for passado com esta skill, você deve gerar **3 cenários distintos**, cada um com **uma persona nomeada** e uma **situação descrita**. Use esta estrutura:

### A Persona (nome e breve perfil)
- **Nome** da persona (ex.: Maria, João, Carla).
- **Quem é:** uma linha sobre perfil relevante (ex.: usuária recorrente, primeira vez, pessoa com pouca familiaridade com app).
- **Por que está fazendo isso agora:** motivação no momento (urgência, rotina, primeira vez, pressão, curiosidade, confusão).

### A Situação em que a Persona Está
- **Onde está:** lugar físico e/ou digital (casa, ônibus, fila, escritório, outro app aberto).
- **Contexto imediato:** o que acabou de acontecer ou o que a trouxe até essa tela/ação (ex.: “acabou de receber um pedido de pagamento”, “está com pressa para pegar o ônibus”).
- **Estado emocional ou prático:** calma, com pressa, distraída, insegura, desconfiada, com medo de errar.
- **Riscos ou incertezas já presentes:** (ex.: rede fraca, não sabe se o valor está certo, medo de enviar para alguém errado).

Essa descrição da situação é a base para entender **impactos** (o que pode dar certo ou errado para essa pessoa) e **cenários possíveis** (fluxo feliz, falhas, edge cases).

### O Ambiente e o Contexto Técnico
Condições que afetam o uso: conexão (estável, 3G, instável), dispositivo, multitarefa, interrupções.

### O Fluxo de Valor (Happy Path)
O caminho ideal para o sucesso **nessa situação**, do ponto de vista da persona.

### O "E se?" (Edge Cases de Negócio)
O que acontece se o comportamento humano ou técnico desviar do esperado? Link com a **situação** descrita (ex.: “Se Maria estiver com pressa e a rede cair…”).

## Critérios de Saída

O relatório deve conter:

1. **Cenário 1: A Persona no Fluxo Padrão** — Persona **com nome**; situação vivida (onde está, o que a levou ali, o que espera); ambiente; fluxo de valor; edge cases ligados à situação.
2. **Cenário 2: A Persona sob Pressão ou Estresse** — Outra persona **com nome** (ou a mesma em outro momento); situação de pressão/estresse descrita com empatia; ambiente adverso; fluxo de valor e resiliência; edge cases (timeout, rede, retry).
3. **Cenário 3: A Persona Mal Intencionada ou Confusa** — Persona **com nome** e situação que explique o mal uso ou a confusão; ambiente; foco em segurança e tratamento de erro; edge cases de abuso ou mal entendido.
4. **Lacunas de Decisão** — Lista de ao menos 3 gaps que o requisito original não cobria (ex.: falta de feedback, timeout, estados de erro), **relacionando quando possível à situação das personas**.
5. **Edge Cases de Negócio** — Síntese dos desvios por cenário, com menção às personas e situações quando fizer sentido.

## Objetivo Final

- Gerar cenários **empáticos**, com **persona nomeada** e **situação descrita**, para entender **impactos** e **cenários possíveis**.
- Listar ao menos 3 **Lacunas de Decisão** que o requisito original não cobria, preferencialmente ligadas às situações vividas pelas personas.
