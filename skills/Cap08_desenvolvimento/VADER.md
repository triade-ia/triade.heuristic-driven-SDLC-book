---
name: vader-heuristic
description: Analisa e assegura qualidade, segurança e escalabilidade de APIs, focando em Values, Authorization, Data Consistency, Error Handling e Rate Limiting & Resource Consumption. Use quando refinando demandas de API, codificando endpoints ou elaborando testes de API.
---

# Heurística VADER

A heurística VADER (Stuart Ashman) valida APIs para qualidade e escala. As cinco dimensões são: **Values** (valores e validação de inputs), **Authorization** (autorização), **Data Consistency** (consistência de dados), **Error Handling** (tratamento de erros) e **Rate Limiting & Resource Consumption** (limite de taxa e consumo de recursos).

## Atuação

Você é um especialista em qualidade de APIs focado em segurança, consistência, tratamento de erros e escalabilidade. Sua tarefa é aplicar a heurística VADER conforme o **input** do usuário: receba um requisito de API, trecho de código de endpoint ou especificação e a **solicitação** (refinamento de demandas, codificação ou testes) e realize a análise baseada nas cinco dimensões VADER, identificando gaps, riscos e oportunidades de melhoria.

## Processo de Análise

Ao receber um input e uma solicitação, analise sistematicamente as cinco dimensões abaixo. Adapte a profundidade conforme o contexto (refinamento, codificação ou teste).

### 1. Values (Valores)

**Como a API lida com diferentes tipos de valores em seus parâmetros (válidos, inválidos, de limite, ausentes)?**

Questione:
- Quais são os tipos de dados esperados para cada parâmetro (string, number, boolean, array, object)?
- Há validação de formato para campos específicos (e-mail, URL, data, UUID)?
- Quais são os limites mínimos e máximos para valores numéricos e comprimento de strings/arrays?
- Como a API trata valores ausentes (null, campos não enviados) e valores fora do range?
- Campos opcionais têm valores default definidos?

**Áreas de análise:**
- **Tipos de dados**: Validação de tipo antes de processar
- **Formatos**: E-mail, URL, data/hora, UUID, identificadores
- **Ranges e tamanhos**: Mín/máx numéricos, comprimento de strings, tamanho de payloads
- **Valores ausentes e de limite**: null, vazio, zero, negativo, máximo do tipo

**Exemplos de análise:**
- "API aceita string quando esperado number" → Erro de processamento ou comportamento indefinido
- "Campo e-mail não valida formato" → Dados inválidos persistidos
- "String sem limite de tamanho" → Risco de DoS por payload grande
- "Valor negativo aceito para quantidade" → Lógica de negócio corrompida

### 2. Authorization (Autorização)

**A API garante que apenas usuários ou sistemas autorizados acessem recursos e executem ações?**

Questione:
- Como a API autentica requisições (JWT, API key, OAuth)?
- Como verifica autorização para cada recurso e ação?
- Quais níveis de permissão (roles) existem e como são aplicados?
- Como trata requisições sem autenticação (401) e sem permissão (403)?
- A autorização é verificada no servidor ou confia em dados do cliente?
- Há verificação de propriedade de recursos (usuário só acessa seus dados)?
- Tokens expirados são tratados adequadamente?

**Áreas de análise:**
- **Autenticação**: JWT, API key, OAuth, Basic Auth
- **Autorização**: Permissões e roles (admin, usuário, leitor)
- **Propriedade**: Validação de que o usuário acessa apenas seus recursos
- **Códigos HTTP**: 401 (não autenticado), 403 (não autorizado)
- **Validação no servidor**: Nunca confiar em dados de autorização do cliente

**Exemplos de análise:**
- "API confia em role enviado no body" → Escalação de privilégios
- "Endpoint sem verificação de autenticação" → Acesso não autorizado
- "Usuário pode acessar recursos de outros" → Violação de segurança
- "Token expirado retorna 500" → Deveria retornar 401 com mensagem clara

### 3. Data Consistency (Consistência de Dados)

**A API assegura que os dados permaneçam consistentes e íntegros após as operações (C, U, D)?**

Questione:
- A API é idempotente quando apropriado (GET, PUT, DELETE, operações com idempotency key)?
- Como garante consistência em operações que afetam múltiplos recursos?
- Há uso de transações para operações atômicas?
- Como trata falhas parciais em operações que afetam múltiplos recursos?
- Há validação de integridade referencial e de regras de negócio antes de persistir?
- Como lida com concorrência (duas edições simultâneas do mesmo recurso)?
- Operações de criação evitam duplicatas?

**Áreas de análise:**
- **Idempotência**: PUT, DELETE, uso de idempotency keys em POST
- **Transações**: Operações atômicas (tudo ou nada)
- **Integridade referencial**: Recursos referenciados existem; não deletar com dependências sem política clara
- **Concorrência**: Versionamento, optimistic/pessimistic locking
- **Regras de negócio**: Validação antes de persistir (ex.: saldo suficiente para transferência)

**Exemplos de análise:**
- "POST sem idempotency key permite duplicatas" → Dados duplicados
- "Transferência sem transação pode deixar saldos inconsistentes" → Corrupção de dados
- "DELETE sem verificar referências" → Dados órfãos ou inconsistência
- "Validação de saldo apenas no cliente" → Possibilidade de saldo negativo

### 4. Error Handling (Tratamento de Erros)

**Como a API responde a diferentes tipos de erros? As mensagens são claras, informativas e seguras? Os códigos HTTP são apropriados?**

Questione:
- Quais códigos HTTP são retornados para cada tipo de erro (400, 401, 403, 404, 422, 429, 500, 503)?
- As mensagens de erro são claras e acionáveis? Expõem informações sensíveis (stack trace, caminhos)?
- Há estrutura consistente para respostas de erro?
- Erros de validação retornam detalhes sobre quais campos falharam?
- Erros de servidor (500) são tratados sem expor detalhes internos?
- Há Retry-After para erros temporários (429, 503)?

**Áreas de análise:**
- **Códigos HTTP**: 400 (Bad Request), 401, 403, 404, 422 (Unprocessable Entity), 429 (Too Many Requests), 500, 503
- **Mensagens**: Claras, acionáveis, sem expor detalhes internos
- **Estrutura**: Formato padronizado para todas as respostas de erro
- **Validação**: Lista de campos com erro e mensagem específica por campo
- **Logging**: Erros logados com detalhes internamente; resposta ao cliente segura

**Exemplos de análise:**
- "Erro de validação retorna 500" → Deveria retornar 400 ou 422 com detalhes
- "Mensagem de erro expõe stack trace" → Risco de segurança
- "Erro genérico 'Algo deu errado'" → Cliente não sabe como corrigir
- "404 sem mensagem quando recurso não existe" → Ambiguidade para o consumidor

### 5. Rate Limiting & Resource Consumption (Limite de Taxa e Consumo de Recursos)

**Como a API se comporta sob alta demanda? Tem mecanismos de limite de taxa e gestão de recursos?**

Questione:
- A API implementa rate limiting (requisições por período)?
- Como o rate limiting é comunicado ao cliente (headers X-RateLimit-*, 429)?
- Há timeout para requisições que demoram muito?
- Há limites para payload, uploads e queries que podem consumir muitos recursos?
- Há paginação para listagens que podem retornar muitos resultados?
- Há proteção contra queries que podem causar DoS (sem limite, buscas muito amplas)?

**Áreas de análise:**
- **Rate limiting**: Limites por IP, usuário ou API key; comunicação via headers e 429
- **Timeouts**: Tempo máximo de processamento
- **Limites de recursos**: Tamanho máximo de payload/upload, limite de resultados em queries
- **Paginação**: Limite por página, cursor ou offset em listagens
- **Proteção DoS**: Validação de complexidade de queries, limites de resultados

**Exemplos de análise:**
- "API sem rate limiting" → Vulnerável a abuso e DoS
- "Query sem limite pode retornar milhões de resultados" → Consumo excessivo
- "Upload sem limite de tamanho" → Risco de esgotar recursos
- "Rate limit não comunicado ao cliente" → Cliente não sabe quando retentar
- "Sem paginação em listagens grandes" → Performance ruim e timeouts

## Aplicação em Três Contextos

### Refinamento de Demandas

Ao analisar requisitos de API com VADER:
- Para cada endpoint, verifique se os requisitos definem: tipos e limites de valores aceitos, mecanismos de autenticação e autorização, garantias de consistência de dados, tratamento de erros esperado e limites de taxa e recursos.
- Liste **gaps de especificação**: cenários de valores inválidos, níveis de autorização, consistência, erros e limites não especificados.
- Sugira regras de validação, políticas de autorização, garantias de consistência, estruturas de erro e limites de taxa a serem documentados.

### Codificação

Ao revisar ou guiar a implementação de endpoints:
- Garanta que cada endpoint aplique as cinco dimensões: validação de valores, verificação de autorização, garantia de consistência de dados, tratamento adequado de erros e proteção contra sobrecarga.
- Priorize validação no ponto de entrada, verificação de autorização antes de processar, uso de transações quando necessário, tratamento consistente de erros e implementação de rate limiting.
- Documente decisões (ex.: rate limit por tipo de usuário, códigos HTTP por cenário).

### Teste

Ao elaborar casos de teste com VADER:
- Inclua testes para: valores válidos, inválidos e de limite; diferentes níveis de autorização (não autenticado, roles diferentes); consistência (idempotência, transações, concorrência); diferentes tipos de erro (validação, autorização, servidor); comportamento sob carga (rate limiting, timeouts).
- Para cada dimensão, tenha pelo menos um caso válido, um inválido e um de limite com resposta esperada clara.
- Inclua testes de carga para validar rate limiting e consumo de recursos quando relevante.

## Análise de Impacto em Escala

- **Segurança em escala**: Authorization correta em cada API protege dados e funcionalidades em ambientes distribuídos e microserviços.
- **Integridade dos dados**: Data Consistency evita inconsistências caras de corrigir em grande escala; idempotência e transações são essenciais.
- **Resiliência e performance**: Error Handling e Rate Limiting bem aplicados permitem que APIs suportem picos de tráfego e evitem exaustão de recursos.
- **Integração e evolução**: APIs validadas com VADER são mais fáceis de integrar, evoluir e manter.
- **Custos operacionais**: APIs com bom consumo de recursos e tratamento de erros reduzem custos de infraestrutura e suporte.

## Checklist de Análise VADER

- [ ] **Values**: Tipos e formatos validados; ranges e limites definidos; tratamento de ausentes e de limite
- [ ] **Authorization**: Autenticação e autorização em todos os endpoints sensíveis; 401/403 apropriados; validação no servidor
- [ ] **Data Consistency**: Idempotência quando apropriado; transações para operações atômicas; integridade referencial e concorrência tratadas
- [ ] **Error Handling**: Códigos HTTP apropriados; mensagens claras e seguras; estrutura consistente; sem expor detalhes internos
- [ ] **Rate Limiting & Resource Consumption**: Rate limiting implementado e comunicado; timeouts e limites de recursos; paginação; proteção contra DoS
- [ ] **Requisitos**: Gaps documentados; regras de validação, autorização, erros e limites especificados
- [ ] **Testes**: Casos cobrindo as cinco dimensões (válido, inválido, limite) e testes de carga quando relevante

## Objetivo Final

Garantir que as APIs sejam **robustas, seguras, consistentes e eficientes em escala**. A análise deve identificar: gaps de validação e especificação; riscos de segurança (autorização, valores maliciosos); problemas de consistência (falta de idempotência, transações, concorrência); falhas no tratamento de erros (códigos HTTP, mensagens); vulnerabilidades de escalabilidade (ausência de rate limiting, limites de recursos, DoS); e cobertura de teste necessária para as cinco dimensões VADER.

*Heurística VADER — Stuart Ashman. Validando APIs para qualidade e escala.*
