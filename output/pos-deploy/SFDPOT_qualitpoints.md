# Relatório SFDPOT — Envio de QualiPoints entre Usuários

**Requisito analisado:** Envio de QualiPoints entre usuários (v2)
**Modo de entrada:** Épico + Tarefa (requisito detalhado com contrato de API)
**Data da análise:** 2026-03-29
**Heurística aplicada:** SFDPOT (Bach, 2005)

---

## 1. Mapa de Dimensões

### S — Sources (Fontes)

**Fontes identificadas:**

| Fonte | Tipo | Risco associado |
|-------|------|-----------------|
| App mobile (usuário remetente) | Entrada direta do usuário | Manipulação de payload, injeção de dados, automação maliciosa (bots simulando App) |
| Token de autenticação (Bearer) | Identidade do remetente | Token expirado, roubado ou forjado; sessão comprometida |
| Header `Idempotency-Key` | Controle de duplicidade | Chave reutilizada indevidamente, chave ausente, formato inesperado |
| Banco de dados relacional | Fonte de verdade (saldo, contas, transações) | Dados inconsistentes, latência de leitura, locks em concorrência |
| Painel web | Consulta (somente leitura) | Dados desatualizados por cache ou polling; usuário toma decisão com saldo defasado |

**Riscos críticos identificados:**
- **Fonte não documentada:** O requisito não menciona se há API Gateway, WAF ou middleware entre o App e a API. Componentes intermediários podem alterar headers, rejeitar requisições ou introduzir latência não prevista.
- **Bot como fonte:** Não há menção a rate limiting ou detecção de automação. Um script poderia enviar milhares de requisições válidas com Idempotency-Keys distintas, drenando saldo rapidamente.
- **Confiabilidade do token:** O requisito define que o remetente vem do token, mas não especifica tempo de vida do token nem mecanismo de revogação imediata (ex.: conta bloqueada após emissão do token).

---

### F — Formats (Formatos)

**Formatos de entrada:**

| Campo | Formato esperado | Risco |
|-------|-----------------|-------|
| `recipientId` | String (ID de usuário) | Formato do ID não especificado (UUID? numérico? alfanumérico?). Sem definição, a validação pode ser fraca |
| `amount` | Integer (1–10.000) | Valores decimais (ex.: 99.5) — como a API rejeita? Float que trunca silenciosamente para int? |
| `Idempotency-Key` | String única (ex.: UUID) | Comprimento máximo não definido. Strings extremamente longas podem causar problemas de armazenamento |
| `Content-Type` | `application/json` | Requisições com charset diferente (ex.: `application/json; charset=utf-16`) podem causar parsing incorreto |
| Body JSON | Objeto com 2 campos | Campos extras no body — a API os ignora ou rejeita? (strict vs. lenient parsing) |

**Riscos críticos identificados:**
- **Formato do `recipientId` indefinido:** O requisito diz "string (ID do usuário)" mas não define o padrão. Isso pode levar a injeção (SQL injection se o ID for usado em query sem sanitização) ou a confusão entre IDs de diferentes formatos.
- **Tipo do `amount` ambíguo na prática:** Embora o requisito diga "integer", clientes HTTP enviam JSON onde `100` e `100.0` são ambos válidos. A API precisa rejeitar explicitamente `100.0` ou truncar? Truncar silenciosamente `99.9` para `99` é um bug de negócio.
- **Encoding de caracteres:** Não há menção a encoding. Se o `recipientId` contiver caracteres especiais (acentos em nomes de usuário usados como ID), pode haver corrupção silenciosa.

---

### D — Dependencies (Dependências)

**Dependências diretas:**

| Dependência | Tipo | Criticidade | Fallback previsto |
|-------------|------|-------------|-------------------|
| Banco de dados relacional | Persistência | **Crítica** — sem BD, nada funciona | Nenhum mencionado |
| Serviço de autenticação | Validação de token | **Crítica** — sem auth, nenhum envio é processado | Nenhum mencionado |
| Rede (HTTPS/TLS) | Transporte | **Crítica** — sem rede, sem comunicação | Retry pelo cliente |

**Dependências implícitas (não documentadas):**

| Dependência provável | Risco |
|----------------------|-------|
| API Gateway / Load Balancer | Pode introduzir timeouts próprios menores que 10s, rejeitando requisições antes da API processar |
| Serviço de logging/auditoria | Se síncrono, falha no log pode bloquear a transação; se assíncrono, pode perder registros de auditoria |
| DNS | Resolução de nomes entre serviços internos — falha de DNS causa indisponibilidade total |
| Certificado TLS | Expiração do certificado causa indisponibilidade total com erro opaco para o usuário |

**Riscos críticos identificados:**
- **Sem fallback para o banco de dados:** A operação é atômica e síncrona. Se o banco ficar lento (não indisponível, apenas lento), todas as requisições ficam presas até o timeout de 10s, acumulando conexões e potencialmente causando cascata de falhas.
- **Sem circuit breaker mencionado:** O requisito não prevê mecanismo de circuit breaker. Em cenário de degradação do banco, a API continuará aceitando requisições e consumindo recursos até esgotar o pool de conexões.
- **Dependência do serviço de autenticação:** Se o serviço de auth estiver lento mas não indisponível, cada requisição de envio terá latência adicional não contabilizada no timeout de 10s.

---

### P — Pace (Ritmo)

**Cenários de volume:**

| Cenário | Volume estimado | Risco |
|---------|----------------|-------|
| Uso normal | Dezenas a centenas de transações/hora | Baixo |
| Campanha promocional | Pico de milhares de transações/minuto | **Alto** — sem rate limiting definido |
| Ataque/abuso | Milhões de requisições/minuto | **Crítico** — sem proteção |
| Concorrência no mesmo saldo | Múltiplas requisições simultâneas do mesmo remetente | **Alto** — lock contention no banco |

**Riscos críticos identificados:**
- **Sem rate limiting no MVP:** O requisito explicitamente diz que limites por período ficam para backlog. Isso significa que um usuário pode fazer 10.000 transações de 1 QualiPoint em sequência, ou um atacante pode gerar carga massiva com Idempotency-Keys distintas.
- **Lock contention em concorrência:** A operação atômica (débito + crédito + registro) implica lock no registro de saldo. Se um destinatário popular recebe muitos envios simultâneos, o lock na linha de saldo dele serializa todas as transações, criando gargalo.
- **Processamento síncrono sob pico:** Como o processamento é síncrono, picos de volume traduzem-se diretamente em consumo de threads/conexões do servidor. Sem backpressure, o servidor pode esgotar recursos e rejeitar requisições legítimas.
- **Idempotency-Key storage:** Cada transação grava uma chave de idempotência. Sob alto volume, a tabela de idempotência cresce rapidamente. Não há menção a TTL ou limpeza dessas chaves.

---

### E — Environment (Ambiente)

**Ambientes identificados:**

| Ambiente | Característica | Risco |
|----------|---------------|-------|
| App mobile (iOS/Android) | Diferentes versões de OS, rede instável (3G/4G/5G/WiFi) | Timeouts parciais, requisições duplicadas por instabilidade de rede |
| Painel web (navegador) | Diferentes navegadores, versões, extensões | Exibição incorreta de saldo, polling inconsistente |
| API (servidor) | Cloud ou on-premise — não especificado | Config drift entre ambientes |
| Banco relacional | Instância única ou cluster — não especificado | Replicação lag pode mostrar saldo desatualizado |

**Riscos críticos identificados:**
- **Infraestrutura não especificada:** O requisito não define se a API roda em container, serverless, VM. Isso impacta diretamente o comportamento de timeout, cold start e escalabilidade.
- **Paridade staging/produção:** Sem menção a como garantir que staging reproduz o comportamento de produção. Bugs de concorrência e lock contention frequentemente só se manifestam em produção com volume real.
- **Rede mobile instável:** O cenário mais comum de retry é rede mobile. O requisito define retry com mesma Idempotency-Key, mas não define o comportamento do App se a resposta for recebida parcialmente (ex.: conexão cai após o servidor processar mas antes de enviar a resposta completa).
- **Variáveis de ambiente e secrets:** Não há menção a como tokens, chaves de criptografia e configurações de banco são gerenciados entre ambientes.

---

### O — Other (Outros)

**Aspectos adicionais identificados:**

| Aspecto | Situação | Risco |
|---------|----------|-------|
| **LGPD / Privacidade** | Transações envolvem dados pessoais (IDs de usuários, saldos) | Exposição de `currentBalance` na resposta 422 pode ser considerada vazamento de dado sensível |
| **Segurança** | HTTPS + Bearer token | Sem menção a CORS, CSP, proteção contra replay attack além de idempotência |
| **Auditoria** | Toda tentativa deve ser registrada | Sem definição de retenção de logs, formato ou acessibilidade para equipe de suporte |
| **Acessibilidade** | App mobile | Sem menção a acessibilidade (leitores de tela, contraste, tamanho de fonte) |
| **Fraude** | Transferência P2P de pontos | Sem mecanismo anti-fraude (detecção de lavagem de pontos, transferências suspeitas) |

**Riscos críticos identificados:**
- **Exposição de saldo na resposta de erro:** A resposta 422 inclui `currentBalance` como campo opcional. Isso pode ser explorado: um atacante que conheça o `recipientId` de outro usuário pode tentar envios com valores crescentes para deduzir o saldo da vítima (embora o saldo exposto seja do remetente, não do destinatário — risco menor, mas ainda relevante se o token for comprometido).
- **Ausência de detecção de fraude:** QualiPoints são transferíveis entre usuários. Sem limites por período e sem detecção de padrões (muitas transferências pequenas, transferências circulares), o sistema é vulnerável a esquemas de lavagem de pontos ou abuso de promoções.
- **Compliance e auditoria:** O requisito menciona auditoria, mas não define retenção, formato, busca ou alerta. Em caso de incidente, a equipe pode não conseguir investigar adequadamente.

---

### T — Time (Tempo)

**Aspectos temporais identificados:**

| Aspecto | Definição no requisito | Risco |
|---------|----------------------|-------|
| Timeout da API | 10 segundos | Adequado para operação síncrona, mas sem definição de timeout por camada (DB, auth, rede) |
| Timestamp da transação (`completedAt`) | ISO8601 | Sem menção explícita a fuso horário (UTC?) |
| Validade do token | Não especificada | Token de longa duração + conta bloqueada = janela de abuso |
| TTL da Idempotency-Key | Não especificada | Chaves nunca expiram? Crescimento infinito da tabela |
| Ordenação de transações | "data/hora mais recente primeiro" | Sem definição de qual timestamp (criação? conclusão? recebimento?) |

**Riscos críticos identificados:**
- **TTL da Idempotency-Key não definido:** Se as chaves nunca expiram, a tabela cresce indefinidamente. Se expiram, um retry após a expiração pode causar débito duplo — exatamente o cenário que a idempotência deveria evitar.
- **Fuso horário ambíguo:** O campo `completedAt` é ISO8601, mas o requisito não define se o servidor armazena em UTC e o App converte para local. Usuários em fusos diferentes podem ver horários confusos na lista de transações.
- **Timeout distribuído:** O timeout de 10s é da API, mas não contabiliza: tempo de validação do token no serviço de auth + tempo de query no banco + tempo de commit da transação. Se o auth levar 3s (degradado) e o banco levar 8s (sob carga), o total excede 10s, mas o timeout pode cortar no meio do commit — potencialmente deixando uma transação parcialmente persistida se o rollback não for executado corretamente.
- **Validade do token vs. bloqueio de conta:** Se um token válido por 24h é emitido, e a conta é bloqueada 1h depois, o usuário ainda pode enviar QualiPoints por mais 23h com o token válido (a menos que haja validação de status da conta a cada requisição, o que não está explícito).
- **Sessão expirada durante operação:** O requisito define que sessão expirada retorna 401, mas não define o que acontece se o token expirar entre o envio da requisição e o processamento no servidor (race condition temporal).

---

## 2. Riscos Priorizados

| # | Risco | Dimensão | Severidade | Probabilidade | Prioridade |
|---|-------|----------|------------|---------------|------------|
| 1 | **Ausência de rate limiting** permite abuso massivo e esgotamento de recursos | Pace | Alta | Alta | **Crítica** |
| 2 | **TTL da Idempotency-Key indefinido** — crescimento infinito da tabela ou débito duplo após expiração | Time | Alta | Alta | **Crítica** |
| 3 | **Lock contention no saldo** sob concorrência alta (destinatário popular) causa serialização e timeouts | Pace + Dependencies | Alta | Média | **Alta** |
| 4 | **Timeout de 10s não contabiliza latência distribuída** (auth + DB); pode cortar mid-transaction | Time + Dependencies | Alta | Média | **Alta** |
| 5 | **Token válido após bloqueio de conta** — janela de abuso se validação de status não for feita a cada request | Time + Sources | Alta | Média | **Alta** |
| 6 | **Sem circuit breaker** — degradação do banco causa cascata de falhas em toda a API | Dependencies | Alta | Média | **Alta** |
| 7 | **Sem detecção de fraude** — transferências circulares e lavagem de pontos não são detectadas | Other | Média | Alta | **Alta** |
| 8 | **Formato do `recipientId` indefinido** — risco de injeção ou validação insuficiente | Formats | Média | Média | **Média** |
| 9 | **Fuso horário ambíguo em `completedAt`** — confusão na exibição de transações | Time | Baixa | Alta | **Média** |
| 10 | **Resposta parcial em rede instável** — App não sabe se transação foi ou não processada | Environment | Média | Média | **Média** |

---

## 3. Perguntas em Aberto

### Para Produto/Negócio
1. Qual é o volume esperado de transações no lançamento? E em 6 meses? (dimensiona rate limiting e infraestrutura)
2. Existe plano de promoções que gerem picos de transferência? Em caso positivo, qual o fator multiplicador esperado?
3. Expor `currentBalance` na resposta 422 é aceitável do ponto de vista de privacidade e compliance?
4. Há requisitos de compliance (LGPD, regulatório de pontos/fidelidade) que exijam retenção mínima de logs de auditoria?

### Para Desenvolvimento/Arquitetura
5. Qual é o formato exato do `recipientId`? (UUID v4, numérico sequencial, outro?)
6. Qual é o TTL planejado para as Idempotency-Keys? Como o sistema se comporta após expiração?
7. A validação de status da conta (ativa/bloqueada) é feita a cada requisição ou apenas no login?
8. O timeout de 10s é end-to-end (incluindo auth e DB) ou apenas do processamento da transação?
9. Qual é a estratégia de locking no banco para operações concorrentes no mesmo saldo? (pessimistic lock, optimistic lock com retry?)

### Para Operações/Infraestrutura
10. Qual é a infraestrutura planejada? (containers, serverless, VMs?) Há auto-scaling?
11. O banco será instância única ou cluster com réplicas? Em caso de réplica, como evitar leitura de saldo desatualizado?
12. Há API Gateway com seus próprios timeouts e rate limits que possam interferir no timeout de 10s da API?

---

## 4. Lacunas de Cobertura

| Área | Lacuna | Impacto |
|------|--------|---------|
| **Teste de carga** | Sem definição de volume alvo, não há baseline para teste de performance | Impossível validar se o sistema aguenta o ritmo esperado |
| **Teste de concorrência** | Lock contention e race conditions em saldo não mencionados nos edge cases | Bugs de saldo inconsistente só aparecerão em produção |
| **Teste de resiliência** | Sem cenários de falha de dependência (DB lento, auth indisponível) | Comportamento em degradação é desconhecido |
| **Teste de segurança** | Sem menção a pen testing, OWASP, proteção contra enumeration | Vulnerabilidades podem ser exploradas antes de serem descobertas |
| **Teste de fuso horário** | Sem definição de como timestamps são armazenados e exibidos | Bugs de ordenação e exibição em diferentes regiões |
| **Teste de idempotência sob falha** | Cenário: servidor processa mas crash antes de gravar idempotency key | Transação processada mas retry causará débito duplo |
| **Monitoramento** | Sem definição de métricas, alertas ou dashboards | Problemas em produção não serão detectados proativamente |

---

## 5. Recomendações de Mitigação

### Prioridade Crítica (antes do deploy)

1. **Implementar rate limiting básico** mesmo no MVP — limitar por IP e por usuário (ex.: 10 transações/minuto/usuário). O custo é baixo e previne abuso catastrófico.

2. **Definir TTL para Idempotency-Keys** — sugestão: 24h. Documentar que retries após 24h podem resultar em nova transação. Implementar job de limpeza.

3. **Validar status da conta a cada requisição** — não confiar apenas no momento do login. Consultar status ativo no fluxo de envio antes de processar.

### Prioridade Alta (antes ou imediatamente após deploy)

4. **Implementar circuit breaker** na comunicação com o banco de dados e serviço de autenticação. Em caso de degradação, retornar 503 rapidamente em vez de acumular requisições até timeout.

5. **Definir estratégia de locking** — usar `SELECT ... FOR UPDATE` com timeout curto ou optimistic locking com retry (máximo 3 tentativas) para evitar deadlocks e serialização excessiva.

6. **Estabelecer timeout por camada** — ex.: auth max 2s, query DB max 5s, commit max 2s, total max 10s. Se qualquer camada exceder seu budget, abort e rollback.

7. **Definir e documentar o formato do `recipientId`** — validar com regex no servidor; rejeitar qualquer formato inesperado com 400.

### Prioridade Média (pós-deploy, monitorado)

8. **Padronizar timestamps em UTC** — armazenar `completedAt` em UTC; App e painel convertem para fuso local na exibição.

9. **Implementar métricas e alertas** — latência p50/p95/p99 do endpoint, taxa de erros 4xx/5xx, volume de transações/minuto, tamanho da tabela de idempotência.

10. **Planejar detecção de fraude** para próxima iteração — regras básicas: transferências circulares (A→B→A), volume anômalo por usuário, transferências para contas recém-criadas.

---

## 6. Impacto de Deploy

### Avaliação

| Categoria | Veredito |
|-----------|---------|
| **Funcionalidade core** | Bem especificada — atomicidade, idempotência e contrato de API estão claros |
| **Segurança** | Requer atenção — rate limiting ausente e validação de conta por requisição não confirmada |
| **Resiliência** | Requer atenção — sem circuit breaker, sem timeout por camada, lock strategy indefinida |
| **Operabilidade** | Requer atenção — sem métricas, alertas ou monitoramento definidos |

### Recomendação de Deploy

**Deploy com restrições (não bloqueante, mas com ações obrigatórias):**

O requisito está bem especificado para um MVP e cobre os edge cases mais relevantes. No entanto, a ausência de rate limiting e de validação de status da conta a cada requisição representam riscos exploráveis em produção. Recomenda-se:

1. **Antes do deploy:** Implementar rate limiting básico (risco #1) e definir TTL de idempotência (risco #2).
2. **No deploy:** Monitorar ativamente latência, erros e volume. Ter runbook para bloqueio manual de contas em caso de abuso.
3. **Até 2 semanas pós-deploy:** Implementar circuit breaker, timeout por camada e alertas automáticos.

---

## Checklist de Análise SFDPOT

- [x] **Sources:** Todas as fontes de input identificadas, incluindo fontes implícitas (API Gateway, DNS, certificados TLS)
- [x] **Formats:** Formatos válidos e inválidos mapeados; ambiguidades de tipo e encoding identificadas
- [x] **Dependencies:** Dependências diretas e implícitas listadas com cenários de falha e cascata
- [x] **Pace:** Volume normal e de pico considerados; ausência de rate limiting identificada como risco crítico
- [x] **Environment:** Variações de ambiente e rede mobile investigadas; infraestrutura indefinida flaggeada
- [x] **Other:** Aspectos de privacidade (LGPD), fraude e auditoria contemplados
- [x] **Time:** Fusos horários, TTL de idempotência, timeouts distribuídos e validade de token analisados
- [x] **Interações entre dimensões:** Avaliadas (Pace × Dependencies = lock contention; Time × Dependencies = timeout distribuído)
- [x] **Riscos priorizados:** 10 riscos ordenados por severidade com dimensão de origem
- [x] **Decisão de deploy:** Recomendação emitida — deploy com restrições e ações obrigatórias

---

*Análise gerada com a heurística SFDPOT (Bach, 2005) aplicada ao requisito "Envio de QualiPoints entre usuários v2".*
