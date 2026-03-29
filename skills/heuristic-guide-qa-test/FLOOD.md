---
name: flood-heuristic
description: Testa resiliência do sistema sob pressão extrema com alto volume de requisições, transações concorrentes e entrada massiva de dados, focando em performance, estabilidade, escalabilidade e recuperação. Use quando receber um épico, épico + tarefa (story ou tarefa), requisito, trecho de código ou especificação para refinamento de demandas de alta carga, codificação de mecanismos de escala ou elaboração de testes de carga e estresse.
---

# Heurística FLOOD

Testando a Resiliência sob Pressão Extrema

A heurística Flood (Inundação) foca em testar o comportamento do sistema sob condições de alto volume, onde ele é deliberadamente "inundado" com inúmeras requisições simultâneas, transações ou entradas de dados massivas [1, 2]. O objetivo é avaliar a performance, a estabilidade e a capacidade de o sistema lidar com estresse sem travar, degradar significativamente ou apresentar inconsistências. Diferentemente de heurísticas atribuídas a um criador singular, Flood reflete práticas consolidadas na comunidade de testes e engenharia de software para lidar com cenários de carga e estresse de forma sistemática.

## Atuação

Você é um especialista em resiliência e performance sob carga. Sua tarefa é aplicar a heurística Flood conforme o **input** do usuário e a **solicitação** (refinamento de demandas, codificação ou elaboração de testes), realizando a análise baseada nas cinco dimensões Flood, identificando gaps de comportamento sob carga, limites de capacidade, gargalos de performance e oportunidades de melhoria na escalabilidade e na recuperação sob estresse.

### Formatos de Input Aceitos

| Formato | Quando usar | Profundidade da análise |
|---|---|---|
| **Épico** | Análise estratégica da funcionalidade completa | Abrangente: mapear fluxos críticos e riscos de carga em nível de funcionalidade |
| **Épico + Tarefa** (story ou tarefa) | Análise focada em uma entrega específica | Detalhada: usar o épico como contexto de volume e escala esperados; focar a análise nas cinco dimensões da tarefa |
| Requisito / código / especificação / cenário de teste | Análise técnica ou funcional direta | Como descrito no Processo de Análise abaixo |

## Adaptação por Contexto de Input

Antes de iniciar a análise, identifique o formato do input e ajuste a abordagem:

**Somente Épico**
- Identifique os fluxos de maior volume e os recursos compartilhados cobertos pelo épico
- Aplique as cinco dimensões Flood em nível de funcionalidade, mapeando onde picos de carga ou concorrência podem causar degradação ou falha
- Priorize as dimensões mais críticas para o domínio do épico (ex.: Transações Concorrentes em épicos de estoque; Picos de Requisições em épicos de checkout; Entrada de Dados Massiva em épicos de importação)
- Resultado esperado: visão estratégica com pontos de atenção por fluxo, limites não especificados e gaps de capacidade

**Épico + Tarefa (story ou tarefa)**
- Use o épico para entender o volume esperado, os SLAs do sistema e as dependências que a tarefa compartilha com outros fluxos
- Concentre a análise das cinco dimensões na tarefa/story específica
- Considere como a carga na tarefa pode impactar outros fluxos do épico (ex.: uma tarefa de processamento em massa pode saturar recursos compartilhados pelo épico inteiro)
- Resultado esperado: análise aprofundada da entrega, contextualizada no épico, com requisitos de carga e testes de estresse direcionados

## Processo de Análise

Ao receber um input e uma solicitação, analise sistematicamente as cinco dimensões abaixo. Adapte a profundidade conforme o contexto de input (épico, épico + tarefa, requisito) e o contexto de solicitação (refinamento, codificação ou teste).

### 1. Picos de Requisições

**O que acontece se muitos usuários acessarem a mesma funcionalidade crítica ao mesmo tempo?**

Questione:
- O que acontece se muitos usuários acessarem a mesma funcionalidade crítica simultaneamente?
- O sistema suporta múltiplos logins, muitas buscas, adições massivas ao carrinho ou múltiplas transações simultâneas?
- O que ocorre quando a fila é inundada (ex.: múltiplos cliques no botão de envio)?
- O sistema degrada, trava ou mantém resposta aceitável? Em que ponto?

**Áreas de análise:**
- **Throughput**: Quantas requisições por segundo o sistema suporta antes de degradar
- **Pontos de contenção**: Endpoints ou operações que se tornam gargalo sob pico
- **Fila e backpressure**: Comportamento quando requisições excedem a capacidade de processamento
- **UI sob pico**: Duplo clique, múltiplos envios, botões que não desabilitam durante o processamento

**Exemplos de análise:**
- "Múltiplos logins simultâneos causam timeout no serviço de autenticação" → Gargalo sem escalabilidade
- "Botão de envio aceita vários cliques e gera transações duplicadas" → Ausência de proteção contra inundação na UI
- "Buscas massivas derrubam o banco" → Falta de rate limiting ou cache

### 2. Transações Concorrentes

**Como o sistema lida com múltiplas tentativas de modificar o mesmo recurso?**

Questione:
- O que acontece quando muitos usuários tentam comprar o último item em estoque?
- Como o sistema trata muitos usuários editando o mesmo documento ao mesmo tempo?
- Há race conditions? Dados inconsistentes? Sobrecarga de locks?
- Existe versionamento otimista, pessimista ou mecanismo de fila para operações conflitantes?

**Áreas de análise:**
- **Controle de concorrência**: Locks, versionamento, transações isoladas
- **Integridade sob concorrência**: Evitar overselling, duplicação de operações, corrupção de dados
- **Serialização**: Filas, ordenação de operações críticas
- **Retry e idempotência**: Comportamento quando várias requisições tentam a mesma ação

**Exemplos de análise:**
- "Último item vendido várias vezes sob concorrência" → Falta de lock ou validação atômica de estoque
- "Edição simultânea sobrescreve alterações sem aviso" → Ausência de conflito detectável (optimistic locking)
- "Fila de transações cresce sem limite e estoura memória" → Ausência de backpressure

### 3. Entrada de Dados Massiva

**O que acontece se grandes volumes de dados forem inseridos rapidamente?**

Questione:
- O sistema suporta upload de vários arquivos simultaneamente? Há limite de tamanho e quantidade?
- Como trata importação de grandes planilhas ou envio de mensagens em massa?
- Há limites definidos? O sistema rejeita graciosamente ou trava ao exceder?
- Há processamento em lote, streaming ou filas para não bloquear a requisição?

**Áreas de análise:**
- **Limites de volume**: Tamanho máximo de payload, número de itens por requisição, tamanho de upload
- **Processamento assíncrono**: Uso de filas para operações pesadas
- **Streaming e chunking**: Evitar carregar tudo em memória
- **Feedback ao usuário**: Progresso, timeout, mensagem quando o limite é excedido

**Exemplos de análise:**
- "Upload de muitos arquivos simultâneos esgota conexões ou memória" → Sem limite ou processamento em lote
- "Importação de planilha grande trava o servidor" → Processamento síncrono sem limite
- "Mensagens em massa enviadas em uma requisição causam timeout" → Falta de batch ou fila

### 4. Degradação da Performance

**Em que ponto o sistema começa a ficar lento? Ele mostra sinais de alerta antes de falhar?**

Questione:
- A partir de quantas requisições simultâneas a latência aumenta de forma perceptível?
- Há métricas de monitoramento (latência, throughput, uso de CPU/memória)?
- O sistema exibe sinais de alerta (logs, dashboards) antes de falhar?
- Há degradação gradual (respostas mais lentas) ou falha abrupta?

**Áreas de análise:**
- **Limite de capacidade**: Ponto de ruptura ou degradação significativa
- **Métricas e observabilidade**: Latência (p50, p95, p99), taxa de erro, uso de recursos
- **Alertas**: Configuração de alertas antes do ponto de falha
- **Graceful degradation**: Respostas mais lentas mas corretas vs. erros ou timeouts

**Exemplos de análise:**
- "Sistema fica lento sem aviso e depois cai" → Ausência de alertas e limite conhecido
- "Latência sobe linearmente com carga sem limite aparente" → Gargalo não identificado
- "Sem métricas de performance sob carga" → Impossível prever falha em produção

### 5. Estabilidade e Recuperação

**O sistema trava, reinicia ou apresenta erros internos sob estresse? Como se comporta sob excesso de requests? Se recupera graciosamente ou precisa de intervenção manual?**

Questione:
- O sistema trava, reinicia ou apresenta erros internos sob estresse?
- Como se comporta sob excesso de requisições (carga, performance, segurança)?
- Após pico de carga, o sistema se recupera sozinho ou precisa de intervenção manual?
- Há rate limiting, circuit breaker ou rejeição graciosa (ex.: 429, 503) em vez de crash?

**Áreas de análise:**
- **Estabilidade**: Ausência de crash, OOM, deadlock sob carga
- **Recuperação automática**: Retorno ao normal após redução de carga; health checks
- **Rejeição graciosa**: 429 Too Many Requests, 503 Service Unavailable, mensagem clara
- **Proteção contra DoS/DDoS**: Limites, throttling, priorização de usuários legítimos

**Exemplos de análise:**
- "Sob carga alta o serviço reinicia em loop" → Falta de proteção e recuperação
- "Excesso de requests causa erro 500 em cascata" → Deveria retornar 429/503 e rejeitar graciosamente
- "Após pico, sistema não volta ao normal sem restart manual" → Ausência de auto-recovery

## Aplicação em Três Contextos

### Refinamento de Demandas

Ao analisar um **épico**, mapeie todos os fluxos de maior volume e recursos compartilhados, aplicando as cinco dimensões Flood em cada um, identificando limites não especificados e gaps de capacidade. Ao analisar um **épico + tarefa**, use o épico para entender o volume esperado e os SLAs do sistema e concentre a análise nos gaps de comportamento sob carga da tarefa específica.

Ao analisar requisitos com Flood:
- Para cada funcionalidade crítica, verifique se os requisitos definem: comportamento sob pico de acessos, tratamento de transações concorrentes no mesmo recurso, limites de volume de entrada (upload, importação, envio em massa), expectativas de performance (latência, throughput) e comportamento esperado quando o limite é excedido (rejeição, fila, degradação).
- Liste **gaps de especificação sob carga**: cenários de alto volume não especificados, limites não documentados, recuperação não definida.
- Sugira requisitos para as cinco dimensões: limites de requisições, políticas de concorrência, limites de dados por operação, SLAs de performance e mecanismos de rejeição/recuperação.

### Codificação

Ao revisar ou guiar a implementação:
- Garanta que cada fluxo crítico considere as cinco dimensões: proteção contra picos (rate limiting, debounce no cliente), controle de concorrência (locks, versionamento, idempotency), limites de volume (tamanho de payload, batch size), métricas e alertas de performance, e rejeição/recuperação graciosa (429, 503, circuit breaker, backpressure).
- Priorize rate limiting, circuit breakers, backpressure, caching e processamento assíncrono para operações pesadas.
- Documente decisões (ex.: "limite de N requisições por segundo por usuário"; "estoque validado em transação com lock"; "upload assíncrono com fila").

### Teste

Ao elaborar casos de teste com Flood:
- Inclua testes de carga para: picos de requisições em funcionalidades críticas, transações concorrentes no mesmo recurso, entrada massiva (upload, importação, envio em massa), degradação de performance (identificar ponto de ruptura) e estabilidade/recuperação após pico.
- Para cada funcionalidade crítica, tenha cenários de teste de carga (volume esperado e acima do esperado), testes de estresse (até falha), spike tests (pico súbito) e, quando aplicável, soak tests (carga sustentada).
- Defina resultado esperado claro: latência máxima aceitável, taxa de erro aceitável, comportamento ao exceder limite (429/503), integridade dos dados sob concorrência.

## Análise de Impacto em Escala

- **Continuidade do serviço**: Em escala, indisponibilidade significa perda de receita e confiança. Flood ajuda a garantir que o sistema suporte picos (ex.: Black Friday, lançamento viral) sem cair.
- **Prevenção de inconsistências sob concorrência**: Sistemas mal projetados podem corromper dados sob Flood de transações simultâneas. Testar ativamente valida mecanismos de controle de concorrência.
- **Otimização de custos**: Entender o limite do sistema permite provisionar recursos de forma inteligente, evitando over ou under-provisioning.
- **Validação de arquiteturas assíncronas**: Filas e processamento assíncrono são comuns em escala. Flood é crucial para testar se a fila não estoura e se o processamento se recupera.
- **Experiência do usuário sob carga**: Mesmo sem queda, lentidão excessiva frustra usuários. Flood identifica pontos de degradação da UX.

## Checklist de Análise FLOOD

- [ ] **Picos de Requisições**: Sistema suporta múltiplos acessos simultâneos à funcionalidade crítica? Há proteção contra duplo envio na UI?
- [ ] **Transações Concorrentes**: Race conditions tratadas? Estoque, documentos e recursos compartilhados protegidos?
- [ ] **Entrada de Dados Massiva**: Limites de volume definidos? Processamento em lote ou assíncrono quando necessário?
- [ ] **Degradação de Performance**: Métricas e alertas configurados? Ponto de ruptura conhecido?
- [ ] **Estabilidade e Recuperação**: Sistema rejeita graciosamente (429/503) ou recupera após pico? Sem necessidade de restart manual?
- [ ] **Requisitos**: Gaps de comportamento sob carga documentados? Limites e SLAs especificados?
- [ ] **Testes**: Casos de teste de carga, estresse e recuperação para cenários críticos?

## Objetivo Final

Garantir que o sistema **prospere sob qualquer volume de interações**, identificando limites de capacidade, revelando gargalos de performance, validando mecanismos de escala (auto-scaling, balanceamento, filas) e mitigando vulnerabilidades a ataques DoS/DDoS. A análise deve identificar: cenários de pico não especificados; ausência de controle de concorrência; limites de volume indefinidos; falta de métricas e alertas; instabilidade ou recuperação manual sob estresse; e cobertura de teste de carga necessária para as cinco dimensões Flood em fluxos críticos.

Ao dominar a heurística Flood, você contribui para construir sistemas que não apenas funcionam em condições normais, mas permanecem estáveis e performáticos sob alta demanda, garantindo continuidade do serviço, integridade dos dados e confiança do usuário — pilares da verdadeira escalabilidade.

*Heurística Flood — Uma heurística da comunidade para ambientes de alta demanda. Testando a resiliência sob pressão extrema.*

**[1]** Práticas consolidadas na comunidade de testes e engenharia de software para cenários de carga e estresse.
**[2]** Referências à engenharia de resiliência e teste exploratório em sistemas distribuídos e alto volume.
