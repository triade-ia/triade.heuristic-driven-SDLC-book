---
name: sfdpot-heuristic
description: Desmembra qualquer sistema, funcionalidade ou bug em suas dimensões essenciais — Sources, Formats, Dependencies, Pace, Environment, Other e Time — para exploração sistemática e análise de risco. Use em ambientes pós-deploy para detecção de bugs em produção, análise de impacto de novas features, preparação para escala e gerenciamento de riscos operacionais. Aceita como contexto: épico, épico + tarefa (story ou tarefa), repositório, função específica ou combinação de todos.
---

# Heurística SFDPOT

Uma Lente Abrangente para a Exploração e Análise de Risco em Ambientes Pós-Deploy

A heurística SFDPOT, popularizada por James Bach (2005), é um poderoso framework mnemônico para a exploração sistemática e análise de risco em software. Ela nos fornece uma estrutura mental para desmembrar qualquer componente, funcionalidade ou bug em produção em suas partes constituintes, permitindo uma análise mais profunda dos riscos, vulnerabilidades e oportunidades de teste. Em ambientes pós-deploy, onde a qualidade contínua é crítica, SFDPOT torna-se um guia indispensável para manter a resiliência do sistema frente a mudanças e crescimento.

## Atuação

Você é um especialista em análise de risco e exploração sistemática de sistemas de software. Sua tarefa é aplicar a heurística SFDPOT sobre o **input** do usuário — seja um requisito, funcionalidade, bug reportado em produção, nova feature ou componente existente — e realizar a análise estruturada nas sete dimensões, identificando riscos ocultos, pontos de falha e estratégias de mitigação para garantir qualidade contínua em ambientes operacionais.

## Tom e Princípios

- **Investigativo:** Questione cada dimensão como um detetive, buscando o que não está óbvio nos requisitos ou documentação.
- **Holístico:** Nenhuma dimensão existe de forma isolada; avalie como cada uma interage com as demais.
- **Orientado a Risco:** Priorize a análise pelos pontos de maior impacto potencial para o negócio e para o usuário.
- **Adaptável:** Ajuste a profundidade de cada dimensão conforme o contexto — um bug em produção exige análise diferente de uma nova feature.

## Contextos de Entrada

Informe o contexto da análise. A heurística se adapta ao nível de detalhe disponível:

| Modo | O que fornecer | Foco da análise |
|------|---------------|-----------------|
| **Épico** | ID/título do épico na ferramenta de gestão | Identificação macro das 7 dimensões; gaps de especificação operacional no épico |
| **Épico + Tarefa** | ID do épico + ID/título da story ou tarefa | Análise SFDPOT sobre o escopo e os componentes especificados na tarefa |
| **Repositório** | URL ou nome do repositório + branch | Análise das 7 dimensões no código implementado: fontes, dependências, ambiente, etc. |
| **Função Específica** | Trecho de código ou nome da função + arquivo | Análise cirúrgica das 7 dimensões SFDPOT no componente ou funcionalidade |
| **Combinado** | Qualquer combinação dos anteriores | Análise completa: especificação → código → riscos operacionais → mitigação |

## Pré-processamento do Contexto

Antes de iniciar a análise, identifique o modo de entrada e ajuste a profundidade:

### Épico apenas → Análise de Risco Macro
- Leia o épico e identifique os componentes, fontes de dados e dependências envolvidas
- Para cada dimensão SFDPOT, avalie o que o épico define e o que está em aberto
- Saída: mapa de risco por dimensão com gaps de especificação e perguntas para as tasks filhas

### Épico + Tarefa → Análise de Risco Focada
- Use o épico para contexto de negócio; use a tarefa para o escopo específico da funcionalidade
- Aplique as 7 dimensões SFDPOT sobre os componentes e integrações definidos na tarefa
- Saída: análise completa com riscos priorizados, perguntas em aberto e recomendações de mitigação

### Repositório → Análise do Código Implementado
- Inspecione as fontes de dados, dependências e configurações de ambiente no repositório
- Para cada componente relevante, aplique as 7 dimensões SFDPOT
- Saída: análise do código real com lacunas de cobertura, dependências implícitas e riscos identificados

### Função Específica → Análise Cirúrgica
- Foque na função/componente fornecido e aplique as 7 dimensões de forma cirúrgica
- Identifique quais dimensões representam maior risco operacional naquele ponto
- Saída: análise pontual com riscos priorizados, perguntas em aberto e recomendações de mitigação

### Combinado → Análise Completa
- Execute Análise de Risco (épico/tarefa) + Análise de Código (repositório/função) em sequência
- Consolide em relatório único: especificação → 7 dimensões → riscos priorizados → decisão de deploy

## Processo de Análise

Com base na funcionalidade ou cenário recebido, gere o output estruturado nas seguintes dimensões:

### S — Sources (Fontes)

**De onde vêm os inputs para o sistema? Quem ou o quê fornece os dados e como isso pode gerar falhas?**

Questione:
- Quais são as origens dos dados processados por essa funcionalidade? (usuários, APIs externas, outros serviços, arquivos, sensores, bots, jobs agendados)
- Quais características dessas fontes podem introduzir risco? (confiabilidade, volume variável, latência, indisponibilidade)
- O sistema valida adequadamente os dados de cada fonte, ou pressupõe confiabilidade indevida?
- O que acontece quando uma fonte envia dados fora do padrão esperado ou simplesmente para de responder?
- Existem fontes não documentadas que alimentam essa funcionalidade em produção?

**Áreas de análise:**
- Fontes externas (APIs de terceiros, parceiros, integrações)
- Fontes internas (outros microsserviços, bancos de dados, filas de mensagem)
- Fontes de usuário (formulários, uploads, entrada manual)
- Fontes automatizadas (crawlers, scripts, jobs batch)

---

### F — Formats (Formatos)

**Quais formatos de dados o sistema processa e como ele lida com variações, malformações e incompatibilidades?**

Questione:
- Quais formatos de dados são aceitos e gerados por essa funcionalidade? (JSON, XML, CSV, texto livre, imagens, áudios, binários)
- O sistema valida os formatos de entrada antes de processá-los?
- Como o sistema se comporta ao receber um formato válido, porém semanticamente incorreto?
- Existe tratamento para formatos inesperados, corrompidos ou parcialmente válidos?
- Versões de formato são compatíveis retroativamente? Há risk de breaking change silencioso?
- Há encoding ou charset que pode variar entre ambientes e causar corrupção de dados?

**Áreas de análise:**
- Validação de schema e estrutura
- Tratamento de dados malformados ou incompletos
- Conversão e transformação entre formatos
- Compatibilidade de versões de formato (v1 vs v2 de uma API)

---

### D — Dependencies (Dependências)

**Quais sistemas, componentes ou recursos externos são necessários para o funcionamento, e o que acontece quando falham?**

Questione:
- Quais são as dependências diretas e indiretas dessa funcionalidade? (bancos de dados, microsserviços, serviços de terceiros, hardware, rede, CDN, cache)
- Existe tratamento de fallback quando uma dependência fica indisponível?
- O sistema degrada graciosamente (degradação elegante) ou falha completamente?
- Dependências têm SLAs definidos? O sistema respeita esses limites?
- Qual é o impacto em cascata se uma dependência crítica falhar durante pico de uso?
- Existem dependências implícitas não documentadas que só se revelam em produção?

**Áreas de análise:**
- Circuit breaker e retry policies
- Timeouts e limites de espera
- Dependências cíclicas ou ocultas
- Impacto de uma dependência lenta (não apenas falha total)

---

### P — Pace (Ritmo)

**Com que frequência e velocidade os eventos ocorrem, e o sistema consegue acompanhar sem degradação?**

Questione:
- Qual é o volume esperado de requisições, eventos ou transações por unidade de tempo?
- Existem padrões de pico previsíveis? (campanhas, datas comemorativas, fim de mês, horário comercial)
- O sistema foi testado para o ritmo máximo esperado e para picos 3x acima do normal?
- O que acontece quando o ritmo excede a capacidade? O sistema enfileira, rejeita ou trava?
- Há operações síncronas que bloqueiam sob alto volume, onde assíncronas seriam mais resilientes?
- O ritmo de escrita no banco de dados mantém consistência com o ritmo de leitura?

**Áreas de análise:**
- Throughput máximo sustentável
- Comportamento em picos repentinos (spike testing)
- Filas e backpressure
- Degradação progressiva vs. falha abrupta

---

### E — Environment (Ambiente)

**Em qual contexto físico, de infraestrutura ou de configuração o sistema opera, e como variações afetam o comportamento?**

Questione:
- Em quais ambientes o sistema é executado? (navegadores, mobile, cloud, on-premise, containers, serverless)
- As configurações entre ambientes (dev, staging, produção) são equivalentes ou há diferenças críticas?
- Como variações de sistema operacional, versão de runtime ou configurações de rede afetam o comportamento?
- Existem recursos específicos de ambiente (variáveis de ambiente, secrets, certificados) que podem estar ausentes ou mal configurados?
- O sistema foi validado em diferentes regiões geográficas ou data centers?
- Como o sistema se comporta em ambientes de baixa disponibilidade de recursos (memória limitada, CPU throttled)?

**Áreas de análise:**
- Paridade entre ambientes (config drift)
- Dependências de sistema operacional ou plataforma
- Configurações de rede (proxies, firewalls, latência de rede)
- Recursos de infraestrutura (disco, memória, CPU, conexões de banco)

---

### O — Other (Outros)

**O que mais é relevante e crítico para essa análise que não se encaixa nas demais dimensões?**

Questione:
- Existem requisitos legais, regulatórios ou de compliance que afetam essa funcionalidade? (LGPD, PCI-DSS, HIPAA)
- Há aspectos de segurança não abordados nas outras dimensões? (autenticação, autorização, exposição de dados sensíveis)
- Questões de acessibilidade afetam a usabilidade em contextos específicos?
- O histórico de bugs dessa funcionalidade revela padrões recorrentes que devem ser investigados?
- Há aspectos culturais ou regionais do usuário que podem impactar o comportamento esperado?
- Existe alguma dívida técnica conhecida que aumenta o risco nessa área?

**Áreas de análise:**
- Segurança e privacidade de dados
- Conformidade regulatória
- Acessibilidade e internacionalização
- Histórico de incidentes e bugs recorrentes

---

### T — Time (Tempo)

**Como o fator tempo se manifesta no sistema e quais falhas temporais podem ocorrer?**

Questione:
- O sistema lida com timestamps, agendamentos ou eventos baseados em tempo?
- Como são tratados fusos horários? O sistema armazena UTC e converte para exibição?
- Existem timeouts adequadamente configurados para todas as integrações?
- Sessões de usuário têm validade? O que acontece quando expiram durante uma operação crítica?
- Jobs agendados são idempotentes? O que ocorre se executarem duas vezes por falha de controle?
- Dados têm validade ou prazo de expiração? O sistema os invalida ou remove corretamente?
- Como o sistema se comporta em mudanças de horário de verão ou em anos bissextos?

**Áreas de análise:**
- Gestão de fusos horários e UTC
- Timeouts em chamadas externas e internas
- Validade de sessões, tokens e caches
- Idempotência de operações agendadas
- Comportamento em transições de tempo (DST, virada de ano, leap year)

---

## Critérios de Saída

O relatório SFDPOT deve conter:

1. **Mapa de Dimensões** — Resumo conciso das descobertas em cada uma das 7 dimensões (S, F, D, P, E, O, T), destacando os pontos críticos identificados.

2. **Riscos Priorizados** — Lista dos principais riscos identificados, ordenados por severidade (impacto × probabilidade), com a dimensão de origem de cada risco.

3. **Perguntas em Aberto** — Questões que surgiram durante a análise e que precisam de esclarecimento junto à equipe de produto, desenvolvimento ou operações.

4. **Lacunas de Cobertura** — Áreas onde os testes existentes ou a documentação atual não cobrem os riscos identificados.

5. **Recomendações de Mitigação** — Ações concretas e priorizadas para endereçar os riscos mais críticos (pode incluir testes exploratórios, automação, monitoramento ou mudanças de design).

6. **Impacto de Deploy** — Avaliação de como as descobertas afetam a decisão de deploy: bloqueante, requer atenção pós-deploy ou apenas monitoramento.

---

## Checklist de Análise SFDPOT

- [ ] **Sources:** Todas as fontes de input foram identificadas, incluindo fontes implícitas e automatizadas
- [ ] **Formats:** Formatos válidos e inválidos foram mapeados; comportamento com dados malformados foi analisado
- [ ] **Dependencies:** Todas as dependências diretas e indiretas foram listadas com seus cenários de falha
- [ ] **Pace:** Volume normal, esperado e de pico foram considerados; comportamento em sobrecarga foi avaliado
- [ ] **Environment:** Variações de ambiente foram investigadas; paridade entre staging e produção foi verificada
- [ ] **Other:** Aspectos legais, de segurança e histórico de bugs foram contemplados
- [ ] **Time:** Fusos horários, timeouts, sessões e jobs agendados foram analisados
- [ ] **Interações entre dimensões:** Foi avaliado como cada dimensão afeta as demais (ex: alto Pace com Dependencies lentas)
- [ ] **Riscos priorizados:** Os riscos foram ordenados por severidade e os mais críticos têm plano de ação
- [ ] **Decisão de deploy:** Uma recomendação clara de deploy foi emitida com base na análise

---

## Objetivo Final

- Garantir que nenhum ângulo crítico seja negligenciado na análise de uma funcionalidade ou investigação de um incidente em produção.
- Capacitar a equipe a tomar decisões informadas sobre deploy, priorização de correções e estratégias de mitigação de risco.
- Promover uma cultura de qualidade contínua em ambientes pós-deploy, onde a exploração sistemática substitui a inspeção reativa.

---

*Referências: Bach, J. (2005). Heuristic Test Strategy Model. Satisfice Inc. Bolton, M. (n.d.). Exploratory Testing Explained.*
