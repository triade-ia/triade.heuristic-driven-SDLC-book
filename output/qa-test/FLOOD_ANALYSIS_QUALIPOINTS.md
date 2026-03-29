# Análise FLOOD — Envio de QualiPoints entre Usuários

**Heurística:** FLOOD (Testando a Resiliência sob Pressão Extrema)
**Requisito analisado:** Envio de QualiPoints entre usuários (v2)
**Contexto:** Refinamento de demandas + Elaboração de testes
**Data:** 2026-03-29

---

## Resumo Executivo

O requisito de envio de QualiPoints apresenta um fluxo **síncrono e atômico** (débito + crédito + registro) exposto via API REST, consumido por App mobile e painel web. A análise FLOOD identificou **gaps críticos** nas cinco dimensões, especialmente na ausência de rate limiting, limites por período, métricas de performance e mecanismos de recuperação sob carga — todos classificados como backlog no MVP mas que representam riscos reais de indisponibilidade e inconsistência em cenários de alta demanda.

---

## 1. Picos de Requisições

**O que acontece se muitos usuários acessarem o endpoint `POST /api/v1/transfers` ao mesmo tempo?**

### Gaps Identificados

| # | Gap | Severidade | Detalhe |
|---|-----|------------|---------|
| 1.1 | **Ausência de rate limiting no MVP** | **Crítico** | O requisito (seção 4.3) declara explicitamente que rate limit fica para refinamento posterior. Sem limite de transações por minuto/hora por usuário, um único usuário (ou atacante) pode inundar o endpoint com milhares de requisições simultâneas. |
| 1.2 | **Sem limite diário/mensal de transações** | **Alto** | Apenas o limite por transação (1–10.000 pontos) está definido. Não há limite de volume agregado, permitindo esvaziamento de saldo em rajadas rápidas. |
| 1.3 | **Proteção contra duplo clique depende exclusivamente da Idempotency-Key** | **Médio** | O requisito menciona desabilitar o botão de envio durante processamento (seção 11.1), o que é positivo. Porém, se o App gerar uma nova Idempotency-Key a cada clique (implementação incorreta), o mecanismo de proteção falha. O requisito não especifica que a key deve ser gerada **uma vez por intenção**, antes de habilitar o botão. |
| 1.4 | **Throughput máximo não especificado** | **Alto** | Não há SLA de requisições por segundo que o endpoint deve suportar. Sem essa meta, não há como dimensionar infraestrutura nem definir critérios de aceite para testes de carga. |
| 1.5 | **Comportamento sob fila de requisições não definido** | **Alto** | Se o volume de requisições exceder a capacidade do servidor, o requisito não define se deve enfileirar, rejeitar com 429/503 ou simplesmente degradar. |

### Cenários de Risco

- **Campanha promocional:** Empresa distribui QualiPoints como incentivo → milhares de envios simultâneos no mesmo minuto → possível queda do serviço de autenticação ou do banco.
- **Bot automatizado:** Sem rate limiting, um script pode enviar centenas de transferências por segundo, saturando o banco relacional com transações atômicas pesadas.
- **Pico de login + envio:** Múltiplos logins simultâneos + envios imediatos podem criar contenção no serviço de autenticação e no endpoint de transferência ao mesmo tempo.

---

## 2. Transações Concorrentes

**Como o sistema lida com múltiplas transferências simultâneas que afetam o mesmo saldo?**

### Pontos Positivos do Requisito

- **Atomicidade explícita** (seção 5.3, 6.2, 8.1): débito + crédito + registro em transação única de banco com rollback em caso de falha.
- **Idempotency-Key** (seção 8.2): evita reprocessamento da mesma intenção de envio.

### Gaps Identificados

| # | Gap | Severidade | Detalhe |
|---|-----|------------|---------|
| 2.1 | **Tipo de lock não especificado** | **Alto** | O requisito garante atomicidade mas não define se usa lock otimista (versionamento de saldo) ou pessimista (SELECT FOR UPDATE). Em cenários de alta concorrência, a escolha impacta diretamente throughput e risco de deadlock. |
| 2.2 | **Race condition no saldo do remetente** | **Alto** | Se o mesmo remetente faz 2 envios simultâneos com Idempotency-Keys diferentes, ambos leem saldo = 100, ambos tentam debitar 60. Sem lock adequado, ambas podem passar a validação de saldo. O requisito não especifica a ordem de lock (remetente → destinatário) para evitar deadlocks. |
| 2.3 | **Cenário de destinatário popular** | **Médio** | Se muitos remetentes enviam para o mesmo destinatário simultaneamente, o registro de saldo do destinatário se torna ponto de contenção (hot row). Não há menção de estratégia para esse cenário (ex.: saldo como soma de eventos, sharding, fila). |
| 2.4 | **Expiração da Idempotency-Key não definida** | **Médio** | O requisito não define por quanto tempo a Idempotency-Key é armazenada. Sem expiração, a tabela de chaves cresce indefinidamente. Com expiração curta demais, um retry tardio pode reprocessar a transação. |
| 2.5 | **Comportamento de retry após erro de negócio** | **Baixo** | Se a primeira requisição retorna 422 (saldo insuficiente), requisições subsequentes com a mesma key retornam o mesmo 422, mesmo que o saldo já tenha sido recarregado. O requisito menciona isso mas o impacto na UX não está claro. |

### Cenários de Risco

- **Esvaziamento por concorrência:** Usuário com saldo 100 envia 2 transferências de 60 pontos simultaneamente (keys diferentes) → sem lock, ambas passam → saldo fica -20.
- **Deadlock cruzado:** Usuário A envia para B, usuário B envia para A simultaneamente → se os locks são adquiridos em ordem diferente (A debita A, B debita B, A credita B, B credita A), deadlock.
- **Hot row no destinatário:** Premiação coletiva onde 500 usuários enviam para o mesmo destinatário → contenção severa no UPDATE do saldo desse destinatário.

---

## 3. Entrada de Dados Massiva

**O que acontece se grandes volumes de dados forem submetidos ao sistema?**

### Gaps Identificados

| # | Gap | Severidade | Detalhe |
|---|-----|------------|---------|
| 3.1 | **Ausência de envio em lote (batch)** | **Médio** | O requisito suporta apenas transferência individual (1 destinatário por requisição). Se um usuário precisa enviar para 500 destinatários (ex.: distribuição de pontos de incentivo), precisa fazer 500 requisições individuais. Sem batch, isso gera rajada de requisições. |
| 3.2 | **Payload não limitado além dos campos obrigatórios** | **Baixo** | O body aceita `recipientId` e `amount`. Não há menção de limite de tamanho do payload total ou proteção contra campos extras injetados no JSON. |
| 3.3 | **Lista de transações sem paginação para escala** | **Médio** | A lista retorna as últimas 10 transações (seção 6.3). Isso é adequado para o MVP, mas para um usuário com milhares de transações, a query precisa ser eficiente (índice por usuário + data). O requisito não menciona índices ou performance da consulta. |
| 3.4 | **Crescimento da tabela de transações** | **Médio** | Cada envio gera um registro. Sem política de retenção ou arquivamento, a tabela cresce indefinidamente, degradando consultas e operações de manutenção do banco. |

### Cenários de Risco

- **Distribuição massiva:** Empresa quer distribuir pontos para 10.000 colaboradores → 10.000 requisições individuais → rajada que pode derrubar o serviço.
- **Tabela de transações:** Após 1 ano com 100.000 transações/dia, a tabela terá ~36M de registros → queries sem índice adequado degradam.

---

## 4. Degradação da Performance

**Em que ponto o sistema começa a ficar lento? Há sinais de alerta?**

### Gaps Identificados

| # | Gap | Severidade | Detalhe |
|---|-----|------------|---------|
| 4.1 | **SLA de latência definido apenas como timeout** | **Alto** | O requisito define timeout de 10 segundos (seção 5.2), mas não define latência esperada para operação normal (ex.: p95 < 500ms). O timeout é o limite de falha, não o limite de qualidade. |
| 4.2 | **Nenhuma métrica de observabilidade especificada** | **Crítico** | Não há menção de métricas (latência p50/p95/p99, throughput, taxa de erro, uso de CPU/memória do banco). Sem métricas, é impossível detectar degradação antes da falha. |
| 4.3 | **Nenhum alerta definido** | **Crítico** | Sem métricas não há alertas. O requisito menciona auditoria (seção 8.3) para suporte, mas não menciona monitoramento operacional para detectar degradação de performance em tempo real. |
| 4.4 | **Ponto de ruptura desconhecido** | **Alto** | Sem testes de carga definidos e sem SLA de throughput, o ponto de ruptura do sistema é desconhecido até que ocorra em produção. |
| 4.5 | **Degradação gradual vs. falha abrupta não especificada** | **Alto** | O requisito não define se, sob carga crescente, o sistema deve degradar graciosamente (respostas mais lentas mas corretas) ou se simplesmente falhará. |

### Cenários de Risco

- **Black Friday de pontos:** Pico de 10x o volume normal → sistema fica lento sem aviso → timeouts em cascata → sem alerta, equipe descobre pelo volume de reclamações.
- **Degradação silenciosa:** Latência sobe de 200ms para 8s gradualmente → ninguém percebe até que os 10s de timeout sejam atingidos e transações comecem a falhar.

---

## 5. Estabilidade e Recuperação

**O sistema trava, reinicia ou apresenta erros internos sob estresse? Se recupera sozinho?**

### Gaps Identificados

| # | Gap | Severidade | Detalhe |
|---|-----|------------|---------|
| 5.1 | **Ausência de rate limiting e circuit breaker** | **Crítico** | Sem rate limiting (gap 1.1), não há primeira linha de defesa contra flood. Sem circuit breaker, uma falha no banco pode gerar retry storms que amplificam o problema. |
| 5.2 | **Comportamento sob excesso de requisições indefinido** | **Crítico** | O requisito não define se o sistema deve retornar 429 (Too Many Requests) ou 503 (Service Unavailable) sob carga excessiva. Sem rejeição graciosa, o sistema pode crashar em vez de rejeitar. |
| 5.3 | **Recuperação após pico não especificada** | **Alto** | Após um pico que sature conexões de banco ou memória, o requisito não define se o sistema se recupera automaticamente ou precisa de restart manual. |
| 5.4 | **Health check não mencionado** | **Médio** | Sem health check, orquestradores (Kubernetes, load balancer) não podem detectar instâncias degradadas e rotear tráfego para instâncias saudáveis. |
| 5.5 | **Proteção contra DoS/DDoS ausente** | **Alto** | Uma API de transferência financeira sem rate limiting é um alvo atrativo. O requisito não menciona WAF, throttling por IP, ou priorização de usuários legítimos. |
| 5.6 | **Pool de conexões do banco não dimensionado** | **Médio** | Transações atômicas pesadas (lock + 3 operações) mantêm conexão do banco por mais tempo. Sob carga, o pool pode se esgotar. O requisito não menciona limites de pool ou timeout de conexão. |

### Cenários de Risco

- **Retry storm:** Timeout de 10s causa retry no cliente → retries chegam junto com novas requisições → volume dobra → mais timeouts → cascata de falhas.
- **DoS via API:** Atacante envia milhares de requisições com Idempotency-Keys únicas → cada uma tenta adquirir lock no banco → saturação de conexões → serviço indisponível para todos.
- **OOM por acúmulo:** Sem rejeição, servidor aceita todas as requisições → filas internas crescem → out-of-memory → crash.

---

## Checklist de Análise FLOOD

| Dimensão | Status | Observação |
|----------|--------|------------|
| **Picos de Requisições** | :warning: Gaps críticos | Sem rate limiting, sem SLA de throughput, sem comportamento definido sob fila |
| **Transações Concorrentes** | :white_check_mark: Parcial | Atomicidade e idempotência bem definidas; falta especificar tipo de lock, ordem de lock e expiração de idempotency key |
| **Entrada de Dados Massiva** | :warning: Gaps médios | Sem batch, sem política de retenção, sem dimensionamento de tabela |
| **Degradação de Performance** | :x: Não especificado | Sem métricas, sem alertas, sem SLA de latência operacional, ponto de ruptura desconhecido |
| **Estabilidade e Recuperação** | :x: Não especificado | Sem rate limiting, sem circuit breaker, sem rejeição graciosa (429/503), sem health check |

---

## Requisitos Sugeridos para as Cinco Dimensões

### Para Picos de Requisições

1. **REQ-FLOOD-01:** Implementar rate limiting de N transações por minuto por usuário (definir N com base em uso esperado, ex.: 10/min).
2. **REQ-FLOOD-02:** Implementar rate limiting global de M requisições por segundo no endpoint `/api/v1/transfers` (definir M com base na capacidade do banco).
3. **REQ-FLOOD-03:** Definir SLA de throughput: o sistema deve suportar no mínimo X requisições por segundo com latência p95 < Y ms.

### Para Transações Concorrentes

4. **REQ-FLOOD-04:** Especificar estratégia de lock para validação de saldo (recomendado: `SELECT ... FOR UPDATE` no saldo do remetente, com ordem de lock consistente para evitar deadlocks).
5. **REQ-FLOOD-05:** Definir TTL da Idempotency-Key (ex.: 24 horas) e comportamento após expiração.
6. **REQ-FLOOD-06:** Documentar comportamento esperado quando o mesmo destinatário recebe centenas de transferências simultâneas (contenção aceitável vs. mecanismo de fila).

### Para Entrada de Dados Massiva

7. **REQ-FLOOD-07:** Avaliar endpoint de batch transfer para cenários de distribuição massiva (backlog, mas documentar a limitação no MVP).
8. **REQ-FLOOD-08:** Definir política de retenção/arquivamento para tabela de transações.
9. **REQ-FLOOD-09:** Garantir índices adequados: `(user_id, created_at DESC)` na tabela de transações para a query de últimas 10.

### Para Degradação de Performance

10. **REQ-FLOOD-10:** Definir SLA de latência operacional: p50 < 200ms, p95 < 500ms, p99 < 2s para o endpoint de transferência.
11. **REQ-FLOOD-11:** Implementar métricas de observabilidade: latência por percentil, throughput, taxa de erro, conexões ativas no banco, uso de CPU/memória.
12. **REQ-FLOOD-12:** Configurar alertas: latência p95 > 1s, taxa de erro > 1%, conexões de banco > 80% do pool.

### Para Estabilidade e Recuperação

13. **REQ-FLOOD-13:** Retornar **429 Too Many Requests** quando rate limit é excedido, com header `Retry-After`.
14. **REQ-FLOOD-14:** Retornar **503 Service Unavailable** quando o sistema está degradado (ex.: banco indisponível), com circuit breaker para evitar cascata.
15. **REQ-FLOOD-15:** Implementar health check endpoint (`GET /health`) para load balancer e orquestradores.
16. **REQ-FLOOD-16:** Garantir recuperação automática após pico: conexões de banco devem ser liberadas, memória estabilizada, sem necessidade de restart manual.

---

## Casos de Teste de Carga Sugeridos

### Testes de Carga (Load Tests)

| ID | Cenário | Volume | Resultado Esperado |
|----|---------|--------|--------------------|
| TC-FLOOD-01 | Transferências simultâneas — carga normal | 50 req/s por 5 min | Latência p95 < 500ms, taxa de erro < 0.1%, saldos consistentes |
| TC-FLOOD-02 | Transferências simultâneas — carga alta | 200 req/s por 5 min | Latência p95 < 2s, taxa de erro < 1%, saldos consistentes |
| TC-FLOOD-03 | Mesmo remetente, envios concorrentes | 20 req simultâneas do mesmo usuário, saldo = 100, amount = 10 cada | Exatamente 10 aprovadas, 10 rejeitadas (saldo insuficiente), saldo final = 0 |

### Testes de Estresse (Stress Tests)

| ID | Cenário | Volume | Resultado Esperado |
|----|---------|--------|--------------------|
| TC-FLOOD-04 | Ramp-up até falha | Incrementar 50 req/s a cada 30s até degradação | Identificar ponto de ruptura; sistema deve degradar (não crashar) |
| TC-FLOOD-05 | Destinatário popular (hot row) | 100 remetentes enviando para o mesmo destinatário simultaneamente | Todas as transações válidas são processadas; saldo do destinatário = soma correta |
| TC-FLOOD-06 | Pool de conexões esgotado | Requisições que excedam o pool de conexões do banco | Sistema retorna 503 ou enfileira; sem crash ou OOM |

### Spike Tests

| ID | Cenário | Volume | Resultado Esperado |
|----|---------|--------|--------------------|
| TC-FLOOD-07 | Pico súbito | 0 → 500 req/s instantâneo por 30s → 0 | Sistema absorve ou rejeita graciosamente (429); recupera em < 30s após pico |
| TC-FLOOD-08 | Duplo clique massivo | 1000 usuários clicam 5x cada com mesma Idempotency-Key | Exatamente 1000 transações (não 5000); idempotência garante deduplicação |

### Testes de Recuperação

| ID | Cenário | Volume | Resultado Esperado |
|----|---------|--------|--------------------|
| TC-FLOOD-09 | Recuperação pós-pico | Carga de 500 req/s por 2 min, depois 0 | Sistema volta a latência normal (p95 < 500ms) em < 60s sem intervenção |
| TC-FLOOD-10 | Recuperação após falha do banco | Simular indisponibilidade do banco por 30s durante carga | Circuit breaker ativado (503); após banco voltar, sistema retoma em < 30s |

### Testes de Concorrência e Integridade

| ID | Cenário | Volume | Resultado Esperado |
|----|---------|--------|--------------------|
| TC-FLOOD-11 | Race condition de saldo | 2 requisições simultâneas, mesmo remetente, saldo exato para apenas 1 | Apenas 1 aprovada, 1 rejeitada; saldo nunca fica negativo |
| TC-FLOOD-12 | Deadlock cruzado | Usuário A→B e B→A simultâneos, repetido 100x | Nenhum deadlock permanente; todas as transações eventualmente concluídas ou rejeitadas com retry |
| TC-FLOOD-13 | Idempotência sob concorrência | Mesma Idempotency-Key enviada 10x simultaneamente | Apenas 1 processamento; todas retornam o mesmo resultado |

---

## Análise de Impacto em Escala

| Dimensão | Impacto se não tratado | Prioridade |
|----------|----------------------|------------|
| **Rate limiting** | Indisponibilidade total do serviço; vulnerabilidade a DoS | **P0 — Antes do go-live** |
| **Lock de saldo** | Saldos negativos, inconsistência financeira | **P0 — Antes do go-live** |
| **Métricas e alertas** | Incapacidade de detectar degradação antes do impacto ao usuário | **P1 — Sprint seguinte** |
| **Rejeição graciosa (429/503)** | Crash em vez de degradação controlada | **P1 — Sprint seguinte** |
| **Circuit breaker** | Cascata de falhas amplificada por retries | **P1 — Sprint seguinte** |
| **Health check** | Load balancer não detecta instâncias degradadas | **P2 — Pré-produção** |
| **Batch transfer** | Rajada de requisições individuais em cenários de distribuição | **P3 — Backlog** |
| **Retenção de dados** | Degradação de performance de queries ao longo do tempo | **P3 — Backlog** |

---

## Conclusão

O requisito de Envio de QualiPoints apresenta **fundamentos sólidos de concorrência** (atomicidade e idempotência bem especificados), mas possui **gaps significativos nas dimensões de proteção contra picos, observabilidade e recuperação**. Os itens classificados como **P0** (rate limiting e especificação de lock) devem ser resolvidos **antes do go-live**, pois representam riscos de inconsistência financeira e indisponibilidade total. A ausência de métricas e alertas (P1) torna o sistema "cego" sob carga, impossibilitando detecção proativa de problemas.

**Recomendação:** Incorporar os requisitos REQ-FLOOD-01 a REQ-FLOOD-04 e REQ-FLOOD-13 no escopo do MVP, e executar os testes TC-FLOOD-03, TC-FLOOD-08, TC-FLOOD-11 e TC-FLOOD-13 como critério de aceite antes do deploy em produção.

---

*Análise gerada com a Heurística FLOOD — Testando a resiliência sob pressão extrema.*
*Requisito: Envio de QualiPoints entre usuários v2*
