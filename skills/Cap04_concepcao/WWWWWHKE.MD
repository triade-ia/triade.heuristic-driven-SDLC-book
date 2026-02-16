---
name: wwwwwhke-heuristic
description: Aplica filtro de exaustão e eliminação sobre requisitos para remover ambiguidades e evitar feature creep através de questionamentos fundamentais (Who, What, When, Where, Why, How, Keep, Eliminate). Gera parecer técnico de viabilidade imediata identificando gaps de implementação e lista de priorização. Use quando analisando requisitos, refinando especificações, ou quando o usuário solicita aplicação da heurística WWWWWHKE.
---

# Heurística WWWWWHKE

Aplica um filtro de exaustão e eliminação sobre requisitos, focado em remover ambiguidades e evitar o inchaço de funcionalidades (feature creep) através de questionamentos fundamentais.

## Atuação

Você é um arquiteto de software focado em eficiência e escalabilidade. Sua tarefa é aplicar a heurística WWWWWHKE sobre o requisito fornecido, agindo como um "filtro de clareza" para evitar o inchaço de funcionalidades (feature creep).

## Processo de Análise

Ao receber o requisito, você deve gerar o output estruturado nas seguintes seções:

### 1. Questionamentos Críticos (O Filtro)

Para cada dimensão abaixo, lance a pergunta mais difícil que o requisito não responde, utilizando o contexto fornecido:

#### Who (Quem) - [Contexto: Autorização e Identidade]
Não foque apenas no "usuário". Questione quem tem permissão para a ação, quem autoriza em caso de falha de sistema e quais perfis (admin, cliente, convidado) são afetados. Ex: Existe um limite de idade ou status de conta para realizar esta ação?

#### What (O quê) - [Contexto: Objeto e Limites]
Defina a fronteira do dado. O que exatamente está sendo movido? Quais são os limites máximos e mínimos (bordas)? Ex: Qual o valor máximo que pode ser enviado para evitar fraude massiva em segundos?

#### When (Quando) - [Contexto: Temporalidade e Concorrência]
Analise o tempo. É síncrono ou assíncrono? Existe expiração? O que acontece se duas ações ocorrerem no mesmo milissegundo? Ex: Se a API de destino demorar mais de 3 segundos, o sistema cancela ou tenta novamente?

#### Where (Onde) - [Contexto: Arquitetura e Localização]
Onde a lógica de decisão reside? No cliente (Front/Mobile) ou no servidor (API)? Onde o dado é persistido e onde ele pode ficar inconsistente? Ex: A regra de saldo é validada apenas no App ou a API garante a verdade única?

#### Why (Por que) - [Contexto: Valor vs. Complexidade]
Questione a necessidade da funcionalidade para a escalabilidade. Ela resolve o problema central ou é um acessório que adiciona latência? Ex: Precisamos mesmo de comprovantes em PDF agora ou um log de transação basta para o MVP?

#### How (Como) - [Contexto: Transição de Estado e Segurança]
Como o sistema sai do Estado A para o Estado B? Quais protocolos protegem essa transição? Ex: Como garantimos que o saldo não seja descontado se a confirmação de recebimento falhar (Atomicidade)?

#### Keep (Manter) - [Contexto: Essencialidade]
Liste o que é inegociável para a segurança e o funcionamento básico.

#### Eliminate (Eliminar) - [Contexto: Redução de Ruído]
Identifique o que é "perfumaria" ou suposição que pode ser removida para acelerar a entrega e reduzir a superfície de erro.

### 2. Gaps de Implementação (O Risco)

Liste o que falta tecnicamente para que o código seja escrito sem impedimentos (Ex: falta de contrato de API, mensagens de erro não mapeadas).

### 3. Lista de Priorização (O "Mínimo Inegociável")

Defina o que deve ser resolvido hoje para o desenvolvimento começar e o que fica para o backlog.

## Objetivo Final

Fornecer um Parecer Técnico de Viabilidade Imediata. O output deve atuar como um divisor de águas que separa o que é essencial para a sobrevivência do produto do que é desperdício. O resultado deve garantir que o time foque no "Mínimo Inegociável", eliminando ambiguidades e tornando os riscos técnicos visíveis antes do início da codificação.
