# Relatório SFDPOT — Envio de QualiPoints entre usuários

**Heurística:** SFDPOT (Bach, 2005)
**Input analisado:** REQ_INICIAL_V2.md — Envio de QualiPoints entre usuários
**Data da análise:** 2026-02-22
**Fase:** Cap10 — Pós-Deploy
**Status:** Análise de risco para ambiente de produção

---

## Mapa de Dimensões

### S — Sources (Fontes)

**O que alimenta a funcionalidade de transferência:**

| Fonte | Tipo | Risco |
|-------|------|-------|
| App mobile (iOS/Android) | Usuário | Inputs maliciosos, replay attacks, bypass de validação client-side |
| Token de autenticação (Bearer) | Autenticação | Token roubado, expirado sem revalidação, token de ambiente errado |
| `recipientId` (string no body) | Usuário | ID manipulado, enumeração de IDs de usuário válidos |
| `amount` (integer no body) | Usuário | Overflow de inteiro, valores não numéricos, ponto flutuante disfarçado |
| `Idempotency-Key` (UUID gerado no App) | Cliente | UUID colidente, reutilização indevida de chaves entre usuários distintos |
| Banco de dados relacional | Interno | Saldo desatualizado por leitura suja, concorrência em alta carga |
| Painel web | Externo (leitura) | Dados desatualizados se cache não for invalidado após transação |

**Riscos identificados nesta dimensão:**

- O requisito não especifica quem pode gerar a `Idempotency-Key`. Se dois usuários diferentes enviarem a mesma chave, a API pode retornar o resultado de outro usuário (vazamento de dado ou bloqueio indevido de operação).
- O `recipientId` é uma string sem formato definido no requisito. Em produção, IDs sequenciais permitem enumeração de usuários válidos para direcionamento de fraude.
- Fontes automatizadas (bots, scripts) não são mencionadas — não há rate limiting no MVP, tornando a API vulnerável a abuso por automação.

---

### F — Formats (Formatos)

**Formatos processados pela API de transferência:**

| Dado | Formato esperado | Risco de variação |
|------|-----------------|-------------------|
| `recipientId` | String (formato não definido) | UUID, integer, e-mail, handle? Sem schema claro. |
| `amount` | Integer (1 a 10.000) | Float como `100.0`, string `"100"`, notação científica `1e2`, overflow `2147483648` |
| `Authorization` | `Bearer <token>` | Token com espaço duplo, case-sensitivity do prefixo, token truncado |
| `Idempotency-Key` | UUID (sugestão do req.) | Chave vazia `""`, chave com espaços, chave com 500+ caracteres |
| `completedAt` | ISO8601 | Fuso horário implícito — UTC ou local do servidor? |
| Corpo da resposta 422 | `{ ..., "currentBalance": 50 }` | Campo opcional expõe saldo — risco de vazamento de informação |

**Riscos identificados nesta dimensão:**

- O requisito descreve `amount` como integer, mas não especifica como a API rejeita `100.0` ou `"100"`. Em linguagens com coerção de tipos (JavaScript, PHP), esses valores podem passar como válidos silenciosamente.
- O campo `currentBalance` no erro 422 é marcado como "opcional" no requisito. Em produção, retornar o saldo atual de um usuário em uma resposta de erro é uma exposição desnecessária de dado sensível — um atacante com acesso ao token pode sondar o saldo sem realizar transações bem-sucedidas.
- `completedAt` em ISO8601 sem fuso horário definido pode gerar inconsistência entre App (exibição local) e painel web (exibição em outro fuso).
- Não há schema de validação (`JSON Schema` ou equivalente) documentado — cada implementação pode aceitar variações distintas.

---

### D — Dependencies (Dependências)

**Dependências diretas e indiretas:**

| Dependência | Criticidade | Comportamento documentado em falha |
|-------------|-------------|-------------------------------------|
| Banco de dados relacional | Crítica | Rollback documentado; mas comportamento em lentidão (não falha total) não especificado |
| Sistema de autenticação (validação de token) | Crítica | Retorna 401 — mas e se o serviço de auth estiver lento? |
| App mobile (cliente) | Alta | Retry com mesma Idempotency-Key documentado |
| Painel web | Baixa | Consulta somente; polling/refresh manual aceitável |
| Serviço de notificação (e-mail, push) | Fora do MVP | — |
| CDN / proxy / load balancer | Não mencionado | Pode terminar conexões antes do timeout de 10s |

**Riscos identificados nesta dimensão:**

- **Banco de dados lento (não indisponível):** O requisito documenta rollback em falha total, mas não especifica o comportamento quando o banco responde em 9s — a operação pode ser iniciada e o timeout de 10s encerrar a conexão do cliente antes da resposta, deixando o App sem saber se a transação foi concluída. A Idempotency-Key resolve o retry, mas o estado final pode ser incerto para o usuário.
- **Dependência implícita de clock:** A transação atômica pressupõe que o banco tem um clock confiável para `completedAt`. Se o banco estiver em um nó diferente com clock drift, timestamps podem ser inconsistentes na lista de recentes.
- **Validação de token:** O requisito diz "autenticação via token (Bearer)", mas não especifica se a API valida o token localmente (JWT) ou consulta um serviço externo. Se for consulta externa, a indisponibilidade desse serviço causa falha de 100% das transferências.
- **Sem circuit breaker documentado:** Não há especificação de comportamento quando dependências ficam lentas mas não totalmente indisponíveis.

---

### P — Pace (Ritmo)

**Volume e frequência de operações:**

| Cenário | Especificado no requisito | Risco |
|---------|--------------------------|-------|
| Volume normal de transferências | Não especificado | Sem baseline de capacidade definida |
| Pico de uso (campanhas, gamificação) | Não mencionado | Sem estratégia de spike |
| Rate limit por usuário | Backlog | MVP sem proteção contra abuso |
| Rate limit global (API) | Não mencionado | Risco de DoS por volume |
| Concorrência do mesmo usuário | Documentado (idempotência) | Mitigado por Idempotency-Key |
| Timeout de 10s por requisição | Documentado | Síncrono bloqueia thread/conexão sob carga |

**Riscos identificados nesta dimensão:**

- **Sem rate limiting no MVP:** Um único usuário ou bot pode enviar centenas de transferências por segundo. Sem proteção, isso pode esgotar conexões do banco, causar contenção em locks de transação e afetar todos os usuários.
- **Processamento síncrono sob carga:** O modelo síncrono (cliente espera resposta da API) significa que cada requisição mantém uma conexão aberta por até 10s. Em pico com 1.000 requisições simultâneas, 10.000 segundos de conexão acumulada podem saturar o pool de conexões.
- **Ausência de teste de carga documentado:** O requisito não menciona SLO de throughput (ex.: suportar X transações/segundo). Sem esse dado, não há como validar se o sistema está dentro da capacidade esperada em produção.
- **A lista das últimas 10 transações** é gerada sob demanda. Em tabelas com muitas transações, sem índice adequado por usuário + data, essa query pode degradar com o crescimento dos dados.

---

### E — Environment (Ambiente)

**Contextos de execução:**

| Ambiente | Mencionado | Risco |
|----------|-----------|-------|
| App mobile iOS | Implícito | Versões antigas do OS podem não suportar TLS 1.3; comportamento de retry varia |
| App mobile Android | Implícito | Fragmentação de versões; comportamento de timeout de rede varia por fabricante |
| Painel web (browser) | Explícito | CORS não mencionado — a API permite requisições do domínio do painel? |
| HTTPS | Obrigatório | Certificate pinning não mencionado no App |
| Banco relacional | Implícito | Cloud? On-premise? Multi-region? |
| Ambiente de staging | Não mencionado | Paridade com produção não documentada |
| Variáveis de ambiente / secrets | Não mencionado | Configuração de token secrets, connection strings, timeout — risco de config drift |

**Riscos identificados nesta dimensão:**

- **CORS não especificado:** O painel web faz requisições à API. Se a API não estiver configurada com CORS correto, o painel não consegue consultar o extrato. Em produção, esse erro pode só aparecer após o deploy quando o painel tenta acessar.
- **Certificate pinning:** O App usa HTTPS, mas o requisito não menciona certificate pinning. Sem pinning, ataques man-in-the-middle em redes públicas podem interceptar tokens.
- **Timeout de rede no mobile:** O timeout de 10s é da API, mas redes móveis podem ter timeouts diferentes configurados no SO ou no cliente HTTP. Um timeout de 8s no cliente e 10s na API pode causar o cliente cancelar a requisição enquanto a API ainda processa — a transação pode ser concluída no servidor sem o cliente saber.
- **Diferença de timezone por região:** Se o produto for usado em múltiplos fusos, `completedAt` sem UTC explícito pode exibir horários incorretos na lista de transações.

---

### O — Other (Outros)

**Aspectos regulatórios, segurança e histórico:**

**Segurança:**
- O `recipientId` é exposto no body da requisição — se o ID de usuário for sequencial ou previsível, um atacante autenticado pode tentar transferências para IDs sequenciais para mapear usuários ativos no sistema.
- A resposta de erro 404 (`RECIPIENT_NOT_FOUND`) confirma existência/inexistência de IDs — isso é um oracle de enumeração de usuários. Considerar resposta genérica ou rate limiting antes de mapear isso como aceitável.
- O campo `currentBalance` no erro 422 pode vazar informação de saldo para um atacante que controla o `amount` da requisição (tentativa binária com valores progressivos para inferir o saldo exato).
- Auditoria de falhas documentada ("toda tentativa de envio, sucesso ou falha"), mas sem menção ao armazenamento seguro desses logs (logs com dados de usuário, valor e destinatário podem violar LGPD se não protegidos).

**Conformidade (LGPD):**
- Transações contêm dados pessoais (ID do remetente, ID do destinatário, valor, timestamp). O requisito não menciona prazo de retenção desses dados, direito ao esquecimento, ou como dados são anonimizados em logs.
- O campo `currentBalance` na resposta 422 retorna dado financeiro sensível — deve ter base legal clara para o processamento.

**Dívida técnica identificada:**
- Lista de 10 transações sem paginação: ao crescer para 10 transações, o limite é fixo. A migração futura para paginação pode ser um breaking change na API (novo campo `page`, `cursor`).
- `recipientId` como string sem formato definido: migração futura para um tipo específico (UUID) pode quebrar clientes que esperam outro formato.

---

### T — Time (Tempo)

**Dimensões temporais da funcionalidade:**

| Aspecto | Especificado | Risco |
|---------|-------------|-------|
| Timeout da API (10s) | Sim | Cliente mobile pode ter timeout menor; estado da transação incerto |
| `completedAt` em ISO8601 | Sim | Fuso horário não definido explicitamente |
| Validade do token de autenticação | Não especificado | Token pode expirar durante processamento de 10s |
| Validade da Idempotency-Key | Não especificado | Por quanto tempo a chave é armazenada? |
| Jobs agendados | Não mencionados | Sem risco direto no MVP |
| Expiração de sessão durante operação | Não tratado | Token expira enquanto API processa — 401 ou sucesso? |

**Riscos identificados nesta dimensão:**

- **Validade da Idempotency-Key não definida:** O requisito diz que a API retorna o resultado persistido para a mesma chave, mas não define por quanto tempo essa chave é mantida. Se o armazenamento de chaves expirar em 1 hora e o usuário tentar retry depois de 2 horas, a API pode reprocessar e gerar débito duplo.
- **Token expirado durante processamento:** Se o token de autenticação expirar durante os 10s de processamento, a API pode retornar 401 mesmo que a transação tenha sido iniciada. O estado da operação fica ambíguo.
- **Fuso horário em `completedAt`:** O campo está em ISO8601, mas sem definição explícita de UTC. Se o servidor estiver em UTC-3 e o App exibir no fuso local do usuário sem conversão adequada, o histórico de transações pode mostrar horários incorretos.
- **Ordering da lista de recentes:** "Ordenadas por data/hora mais recente primeiro" pressupõe que os timestamps no banco são confiáveis. Em ambiente distribuído ou com clock skew, duas transações quase simultâneas podem aparecer em ordem inversa.
- **Horário de verão (DST):** Se o banco armazena em horário local e há transição de DST, transações no momento da transição podem ter timestamps ambíguos ou duplicados.

---

## Riscos Priorizados

| # | Risco | Dimensão | Severidade | Probabilidade | Prioridade |
|---|-------|----------|------------|---------------|------------|
| 1 | Campo `currentBalance` no erro 422 permite inferência binária do saldo | F / O | Alta | Alta | **Crítico** |
| 2 | Sem rate limiting — vulnerável a abuso e DoS | P | Alta | Alta | **Crítico** |
| 3 | Validade da Idempotency-Key não definida — débito duplo após expiração | T | Alta | Média | **Alto** |
| 4 | Timeout assimétrico cliente/servidor — estado incerto da transação | T / D | Alta | Média | **Alto** |
| 5 | Oracle de enumeração via 404 `RECIPIENT_NOT_FOUND` | O / S | Média | Alta | **Alto** |
| 6 | `recipientId` formato não definido — enumeração de usuários | S / F | Média | Alta | **Alto** |
| 7 | Token expira durante processamento — estado ambíguo | T | Média | Média | **Médio** |
| 8 | CORS não especificado — painel web pode não funcionar em produção | E | Alta | Média | **Médio** |
| 9 | Fuso horário de `completedAt` não definido — histórico incorreto | T / F | Baixa | Alta | **Médio** |
| 10 | Logs de auditoria com dados pessoais sem política de retenção (LGPD) | O | Alta | Baixa | **Médio** |
| 11 | Banco lento (não indisponível) — comportamento não especificado | D | Média | Média | **Médio** |
| 12 | Certificate pinning ausente — risco de MITM em redes públicas | E | Alta | Baixa | **Baixo** |
| 13 | Clock drift em ambiente distribuído — ordering de transações | T | Baixa | Baixa | **Baixo** |

---

## Perguntas em Aberto

**Sobre Sources / Formats:**
1. Qual é o formato exato esperado para `recipientId`? UUID, integer, e-mail? Existe schema de validação documentado?
2. A `Idempotency-Key` é validada por usuário (remetente + chave) ou apenas pela chave? Dois usuários podem ter a mesma chave sem conflito?
3. Como a API rejeita `amount: 100.0` (float)? Erro de tipo ou coerção para 100?

**Sobre Dependencies / Pace:**
4. Qual é o pool de conexões configurado no banco? Qual o throughput máximo sustentável testado?
5. O serviço de validação de token é local (JWT) ou requer chamada externa? Qual o impacto na disponibilidade se esse serviço ficar lento?
6. Existe load balancer ou API gateway na frente da API? Como ele trata timeouts — encerra em menos de 10s?

**Sobre Environment:**
7. CORS está configurado? Quais origens são permitidas para o painel web?
8. A API é multi-region ou single-region? Como isso afeta latência e conformidade de dados?
9. Qual a paridade de configuração entre staging e produção? Existem variáveis de ambiente diferentes?

**Sobre Time:**
10. Por quanto tempo a `Idempotency-Key` é armazenada no servidor? Há TTL definido?
11. O campo `completedAt` é armazenado e retornado em UTC? Quem faz a conversão para exibição — App ou backend?
12. O que acontece se o token de autenticação expirar durante o processamento dos 10s?

**Sobre Other / LGPD:**
13. Qual é a política de retenção dos logs de auditoria de transações? Como dados pessoais nos logs são protegidos?
14. Existe base legal documentada para retornar `currentBalance` na resposta de erro 422?

---

## Lacunas de Cobertura

| Área | Lacuna identificada |
|------|---------------------|
| Testes de segurança | Sem menção a testes de enumeração de `recipientId`, teste de inferência de saldo via 422, ou testes de token inválido/expirado |
| Testes de carga | Sem baseline de throughput; sem cenário de spike testing documentado |
| Testes de timeout assimétrico | Comportamento quando cliente tem timeout menor que 10s não coberto |
| Testes de idempotência com chave expirada | Cenário de retry após expiração da chave não especificado |
| Testes de ambiente | Sem testes de paridade staging/produção documentados |
| Validação de formato | Sem testes de `amount` como float, string ou overflow |
| LGPD / compliance | Sem testes de controle de acesso nos logs de auditoria |
| Timezone | Sem testes de `completedAt` em diferentes fusos horários |

---

## Recomendações de Mitigação

### Imediatas (antes ou logo após deploy)

**R1 — Remover `currentBalance` da resposta 422**
O campo `currentBalance` no erro de saldo insuficiente é opcional no requisito e representa risco de vazamento de dado financeiro. Remover ou substituir por uma faixa genérica (ex.: `"balanceSufficient": false`).

**R2 — Implementar rate limiting mínimo**
Mesmo no MVP, adicionar rate limiting por usuário autenticado (ex.: 10 transferências por minuto) para evitar abuso. Configurar no API gateway ou middleware.

**R3 — Definir TTL da Idempotency-Key**
Documentar e implementar o prazo de validade da chave (ex.: 24 horas). Após expiração, chaves podem ser reutilizadas. Garantir que o cliente App saiba que não pode usar chaves antigas para retry tardio.

**R4 — Definir fuso horário para `completedAt`**
Garantir que o backend armazena e retorna `completedAt` em UTC com sufixo `Z` explícito (ex.: `"2026-02-22T14:30:00Z"`). O App é responsável por converter para o fuso local do usuário.

**R5 — Verificar configuração de CORS**
Antes do primeiro acesso do painel web em produção, validar que os headers CORS (`Access-Control-Allow-Origin`) estão configurados para o domínio do painel.

### Pós-deploy (monitoramento e backlog)

**R6 — Monitorar latência do banco separadamente do timeout da API**
Implementar métricas de latência P95 e P99 das queries de transferência. Alertar quando P99 > 5s (metade do timeout).

**R7 — Avaliar responses de erro para mitigar enumeração**
Considerar resposta genérica para destinatário inválido/inativo em vez de `RECIPIENT_NOT_FOUND` específico, ou combinar com rate limiting de tentativas.

**R8 — Definir schema formal do `recipientId`**
Documentar e validar o formato (UUID v4, por exemplo) para prevenir enumeração e garantir consistência entre versões da API.

**R9 — Política de retenção de logs (LGPD)**
Definir prazo de retenção dos logs de auditoria, garantir que são armazenados em ambiente seguro e com acesso controlado, e documentar a base legal para o processamento desses dados.

**R10 — Teste de timeout assimétrico**
Criar teste que simule o cliente recebendo timeout antes dos 10s da API — verificar se a transação foi concluída e se o retry com a mesma chave retorna o estado correto.

---

## Impacto de Deploy

| Dimensão | Avaliação | Decisão de deploy |
|----------|-----------|-------------------|
| Sources | Sem validação de formato do `recipientId`; risco de enumeração | Requer atenção pós-deploy |
| Formats | `currentBalance` no 422 é risco ativo de vazamento | **Bloqueante** — remover antes do deploy |
| Dependencies | Comportamento em banco lento não especificado | Monitoramento pós-deploy |
| Pace | Sem rate limiting — risco de abuso e DoS | **Bloqueante** — implementar rate limit mínimo |
| Environment | CORS não verificado — painel pode falhar | Requer validação antes do deploy do painel |
| Other | LGPD: logs sem política de retenção definida | Requer atenção pós-deploy |
| Time | TTL da Idempotency-Key indefinido — risco de débito duplo tardio | Requer definição antes do deploy |

### Recomendação geral

**Deploy condicional** — os itens marcados como **Bloqueante** (remoção do `currentBalance` no 422 e implementação de rate limit mínimo) devem ser resolvidos antes do go-live. Os demais itens podem ser tratados como observabilidade e backlog priorizado nas primeiras sprints pós-deploy.

---

## Checklist de Análise SFDPOT

- [x] **Sources:** Fontes identificadas: App mobile, token de autenticação, `recipientId`, `amount`, `Idempotency-Key`, banco de dados
- [x] **Formats:** Formatos mapeados; riscos de float/string em `amount`, formato de `recipientId`, `currentBalance` no 422, timezone em `completedAt`
- [x] **Dependencies:** Dependências diretas (banco, autenticação) e implícitas (load balancer, clock) analisadas com cenários de falha
- [x] **Pace:** Volume e ritmo analisados; ausência de rate limiting e testes de carga identificados como risco crítico
- [x] **Environment:** Variações de ambiente mapeadas; CORS, certificate pinning e timezone identificados
- [x] **Other:** Aspectos de segurança (enumeração, saldo exposto), LGPD (logs, retenção) e dívida técnica contemplados
- [x] **Time:** Fusos horários, TTL da Idempotency-Key, expiração de token e timeout assimétrico analisados
- [x] **Interações entre dimensões:** Pace × Dependencies (banco lento em carga), Time × Sources (token expira durante 10s), Formats × Other (`currentBalance` como vazamento)
- [x] **Riscos priorizados:** 13 riscos ordenados por severidade com prioridade definida
- [x] **Decisão de deploy:** Recomendação de deploy condicional emitida com 2 itens bloqueantes e demais como pós-deploy

---

*Análise gerada com base em: REQ_INICIAL_V2.md (Envio de QualiPoints entre usuários) | Heurística SFDPOT — Bach, J. (2005). Heuristic Test Strategy Model. Satisfice Inc.*
