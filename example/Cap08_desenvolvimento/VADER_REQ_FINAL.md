# Análise VADER: Envio de QualiPoints (REQ_FINAL)

## Contexto do Caso

Este documento apresenta a aplicação da heurística **VADER** (Stuart Ashman) sobre o requisito de **Envio de QualiPoints** ([setUp/REQ_FINAL.MD](../../setUp/REQ_FINAL.MD)). A análise valida a API sob as cinco dimensões VADER — Values, Authorization, Data Consistency, Error Handling e Rate Limiting & Resource Consumption — identificando gaps de especificação, riscos de segurança, consistência de dados, tratamento de erros e vulnerabilidades de escalabilidade.

## Entradas e Contrato em Análise

Com base no contrato de API e nas regras de negócio:

| Entrada | Tipo | Origem | Uso |
|--------|------|--------|-----|
| `recipient_id` | string | Body (JSON) | Identificador do destinatário |
| `amount` | number | Body (JSON) | Valor em QualiPoints a transferir |
| `user_id` | (implícito) | JWT token | Remetente — nunca do body |
| `idempotency-key` | string | Header | Evitar transferências duplicadas |

**Endpoint:** `POST /api/v1/transactions`

---

## 1. Values (Valores)

**Como a API lida com diferentes tipos de valores em seus parâmetros (válidos, inválidos, de limite, ausentes)?**

### Situação no Requisito

- **Valor mínimo**: 1 QualiPoint — definido.
- **Valor máximo**: saldo disponível do remetente — definido conceitualmente; teto absoluto não definido.
- **Tipos**: `recipient_id` como string e `amount` como number; formato de `recipient_id` (UUID, etc.) e tipo numérico de `amount` (inteiro vs decimal) não especificados.
- **Limites de tamanho**: `recipient_id` e `idempotency-key` sem limite de comprimento.
- **Valores ausentes**: Não há menção a tratamento de campos ausentes, null ou string vazia.

### Questões Aplicadas

- Quais são os tipos e formatos esperados para cada parâmetro?
- Quais são os limites mínimos e máximos para valores numéricos e comprimento de strings?
- Como a API trata valores ausentes (null, campos não enviados) e valores fora do range?
- Campos opcionais têm valores default definidos?

### Gaps Identificados

| Gap | Risco | Recomendação |
|-----|--------|---------------|
| Formato de `recipient_id` não definido | UUID vs email vs outro; comportamento inconsistente | Definir no contrato (ex.: UUID v4). Rejeitar formato inválido com 422. |
| Tipo e precisão de `amount` não especificados | Inteiro vs decimal; 1.5 QP? Precisão em cálculos | Definir: inteiro não negativo ou decimal com N casas. Documentar no contrato. |
| Teto máximo absoluto para `amount` não definido | Overflow ou abuso com valores extremos | Definir teto global (ex.: 1.000.000 QP). Rejeitar acima com 422 ou 402. |
| `amount` zero ou negativo não citado | Tratamento inconsistente | Documentar: 0 e negativos inválidos → 422. |
| Tamanho máximo de `recipient_id` e `idempotency-key` não definido | Payload/DoS ou comportamento indefinido | Definir limite (ex.: 36 para UUID, 128 para key). Rejeitar com 413 ou 422. |
| Comportamento para campos ausentes ou null | 500 ou resposta indefinida | Definir obrigatoriedade; ausência/null → 400 ou 422 com mensagem clara. |
| Body vazio ou `{}` | Resposta indefinida | Definir: 400 Bad Request com mensagem indicando campos obrigatórios. |

### Regras de Validação Sugeridas para Documentação

- **recipient_id**: string, obrigatório, formato UUID v4 (ou conforme definição), comprimento 1–L caracteres.
- **amount**: number, obrigatório, inteiro (ou decimal com N casas), mínimo 1, máximo min(saldo_remetente, TETO_GLOBAL); zero e negativos → 422.
- **idempotency-key**: string, opcional, 1–128 caracteres.
- Body deve ser JSON válido com ambos os campos; caso contrário → 400.

---

## 2. Authorization (Autorização)

**A API garante que apenas usuários ou sistemas autorizados acessem recursos e executem ações?**

### Situação no Requisito

- Autenticação via JWT e `user_id` extraído do token (nunca do body) — definido.
- Não há menção a códigos HTTP para falhas de autenticação/autorização nem a verificação de propriedade (ex.: usuário só transfere da própria carteira).
- Não há definição de comportamento para token ausente, inválido ou expirado.

### Questões Aplicadas

- Como a API autentica e como verifica autorização para este endpoint?
- Como trata requisições sem autenticação (401) e sem permissão (403)?
- Há verificação de propriedade (remetente = dono da carteira debitada)?
- Tokens expirados são tratados adequadamente?
- A autorização é validada no servidor (nunca confiar em dados do cliente)?

### Gaps Identificados

| Gap | Risco | Recomendação |
|-----|--------|---------------|
| Códigos 401/403 não citados no contrato | Cliente não sabe o que esperar ao não autenticar ou sem permissão | Documentar: ausência ou token inválido/expirado → 401 Unauthorized; sem permissão para operação → 403 Forbidden. |
| Comportamento para token expirado não especificado | Pode retornar 500 ou mensagem confusa | Definir: token expirado ou inválido → 401 com mensagem clara (ex.: "Token expirado ou inválido"). |
| Verificação de propriedade não explícita | Garantir que débito seja sempre da carteira do usuário do token | Documentar: remetente é sempre o usuário do JWT; qualquer tentativa de debitar de outro → 403. |
| Idempotency-key não atada ao usuário | Reutilização de key entre usuários pode causar confusão ou bypass | Documentar: idempotency-key é validada no escopo do usuário autenticado (mesmo usuário + mesma key = mesma resposta). |
| Transferência para si mesmo | Não definido se é permitido ou rejeitado | Definir regra: permitir (201) ou rejeitar (422 com mensagem). Documentar no contrato. |

### Regras de Autorização Sugeridas para Documentação

- **401 Unauthorized**: Ausência de token, token inválido ou token expirado. Corpo com `code` e `message` (ex.: `TOKEN_EXPIRED`, `TOKEN_INVALID`).
- **403 Forbidden**: Usuário autenticado sem permissão para executar transferência (se houver roles no futuro) ou tentativa de operação fora do escopo do usuário.
- Remetente sempre identificado pelo JWT; ignorar qualquer campo no body que indique remetente.
- Idempotency-key com escopo por usuário autenticado.

---

## 3. Data Consistency (Consistência de Dados)

**A API assegura que os dados permaneçam consistentes e íntegros após as operações?**

### Situação no Requisito

- **Atomicidade**: Operação em transação de banco; rollback se crédito no destinatário falhar — definido.
- **Idempotência**: Header `idempotency-key` para evitar duplicatas em retentativas — definido.
- Não há menção a isolamento de transação, tratamento de falhas parciais fora do DB, nem a concorrência (duas transferências simultâneas do mesmo usuário).
- Destinatário deve existir e estar "Ativo" — validação antes de processar está implícita.

### Questões Aplicadas

- A API é idempotente quando apropriado? Como garante consistência em operações que afetam dois saldos?
- Há uso de transações para operações atômicas?
- Como trata falhas parciais (ex.: débito ok, crédito falha)?
- Há validação de regras de negócio antes de persistir (saldo suficiente, destinatário ativo)?
- Como lida com concorrência (duas transferências do mesmo usuário ao mesmo tempo)?
- Idempotency key evita duplicatas no escopo correto (por usuário)?

### Gaps Identificados

| Gap | Risco | Recomendação |
|-----|--------|---------------|
| Isolamento de transação não especificado | Leitura suja ou não repetível pode gerar saldo incorreto sob concorrência | Documentar: uso de transação com nível de isolamento adequado (ex.: Read Committed ou Serializable conforme necessidade). |
| Ordem de operações na transação (débito/crédito) não definida | Deadlock ou inconsistência em cenários de falha | Documentar: ordem determinística (ex.: sempre débito primeiro, depois crédito) e rollback em qualquer falha. |
| Validação de saldo e de destinatário antes da transação | Evitar transação desnecessária quando regras falham | Documentar: validar saldo suficiente e destinatário existente/ativo antes de abrir transação; falhas → 402 ou 404 sem alterar dados. |
| Concorrência: duas transferências simultâneas do mesmo usuário | Possível saldo negativo se ambas passarem na validação | Documentar: validação de saldo dentro da transação com lock (ex.: SELECT FOR UPDATE na linha do saldo) ou uso de constraint/check. |
| Idempotency: duplicata com mesma key e mesmo usuário | Já coberto pelo requisito | Garantir na especificação: mesma key + mesmo usuário → retornar mesma resposta (201 + mesmo transaction_id), sem nova débito/crédito. |
| Destinatário inexistente ou inativo após início da transação | Raro mas possível (conta desativada entre validação e commit) | Tratar dentro da transação; se destinatário ficar inativo antes do commit, rollback e 404 ou 422. |

### Regras de Consistência Sugeridas para Documentação

- Transação de banco com rollback em qualquer falha; débito e crédito na mesma transação.
- Validar saldo suficiente e destinatário existente/ativo antes ou no início da transação; rejeitar com 402/404 sem alterar dados.
- Evitar race condition: checagem de saldo dentro da transação com lock na linha do remetente (ou equivalente).
- Idempotency: mesma combinação (usuário + idempotency-key) retorna mesma resposta e não executa nova transferência.

---

## 4. Error Handling (Tratamento de Erros)

**Como a API responde a diferentes tipos de erros? As mensagens são claras e seguras? Os códigos HTTP são apropriados?**

### Situação no Requisito

- **402 Payment Required**: Saldo insuficiente — definido.
- **404 Not Found**: Destinatário inexistente — definido.
- **422 Unprocessable Entity**: Valor abaixo do mínimo ou formato inválido — definido.
- Não há menção a 400 (requisição malformada), 401 (não autenticado), 403 (não autorizado), 429 (rate limit), 500 (erro interno), 503 (indisponível).
- Não há estrutura padrão de corpo de erro nem mensagens por tipo de falha.
- Não há Retry-After para erros temporários.

### Questões Aplicadas

- Quais códigos HTTP são retornados para cada tipo de erro?
- As mensagens de erro são claras e acionáveis? Expõem informações sensíveis?
- Há estrutura consistente para respostas de erro?
- Erros de validação retornam detalhes sobre quais campos falharam?
- Há Retry-After para 429 ou 503?

### Gaps Identificados

| Gap | Risco | Recomendação |
|-----|--------|---------------|
| 400 e 401/403 não citados | Cliente não sabe o que esperar para body inválido ou não autenticado | Incluir no contrato: 400 Bad Request (body inválido, JSON malformado, campos obrigatórios faltando); 401 Unauthorized; 403 Forbidden. |
| 429 e 503 não citados | Cliente não sabe como reagir a rate limit ou serviço indisponível | Documentar: 429 Too Many Requests com Retry-After quando rate limit excedido; 503 Service Unavailable em manutenção ou sobrecarga. |
| 500 não documentado | Expectativa de que erros inesperados tenham tratamento padrão | Documentar: 500 Internal Server Error com mensagem genérica; nunca expor stack trace ou detalhes internos ao cliente. |
| Estrutura do corpo de erro não definida | Parsing inconsistente pelos clientes | Definir formato padrão (ex.: `{ "error", "code", "message" }` ou array de erros para 422 com múltiplos campos). |
| Mensagens por tipo de falha não especificadas | Cliente recebe mensagem genérica e não sabe corrigir | Documentar códigos e mensagens por caso (ex.: AMOUNT_BELOW_MINIMUM, RECIPIENT_ID_INVALID_FORMAT, INSUFFICIENT_BALANCE). |
| Erro de validação retornando 500 | Prática ruim; deveria ser 400/422 | Garantir na especificação: erros de validação sempre 400 ou 422 com detalhes; 500 apenas para falhas inesperadas do servidor. |
| Exposição de dados sensíveis em erro | Ex.: revelar se destinatário existe ou não em 404 | Definir política: mensagens seguras (ex.: "Recurso não encontrado" sem revelar dados de outros usuários). |

### Regras de Error Handling Sugeridas para Documentação

- **Estrutura de erro**: Ex.: `{ "error": "Unprocessable Entity", "code": "AMOUNT_BELOW_MINIMUM", "message": "O valor deve ser no mínimo 1 QualiPoint." }`.
- **Códigos por cenário**: 400 (body inválido, JSON malformado); 401 (token ausente/inválido/expirado); 403 (sem permissão); 402 (saldo insuficiente); 404 (destinatário inexistente); 422 (validação: valor, formato, campos); 429 (rate limit excedido, incluir Retry-After); 500 (erro interno, mensagem genérica); 503 (indisponível, opcional Retry-After).
- **Segurança**: Nunca expor stack trace, caminhos ou dados de outros usuários na resposta.
- **422 com múltiplos campos**: Listar erros por campo quando mais de um campo falhar na validação.

### Exemplo de corpo de erro (422)

```json
{
  "error": "Unprocessable Entity",
  "code": "AMOUNT_BELOW_MINIMUM",
  "message": "O valor deve ser no mínimo 1 QualiPoint."
}
```

---

## 5. Rate Limiting & Resource Consumption (Limite de Taxa e Consumo de Recursos)

**Como a API se comporta sob alta demanda? Tem mecanismos de limite de taxa e gestão de recursos?**

### Situação no Requisito

- Não há menção a rate limiting, timeout, limite de tamanho de payload ou política de retentativas.
- Endpoint é POST único (sem listagem); paginação não se aplica.
- Operação é síncrona; tempo de resposta depende de banco e regras de negócio.

### Questões Aplicadas

- A API implementa rate limiting?
- Como o rate limiting é comunicado ao cliente (headers, 429)?
- Há timeout para requisições que demoram muito?
- Há limites para payload?
- Há proteção contra abuso (muitas requisições por usuário/IP)?

### Gaps Identificados

| Gap | Risco | Recomendação |
|-----|--------|---------------|
| Rate limiting não especificado | Abuso, DoS, sobrecarga do serviço | Documentar: rate limiting por usuário autenticado (e opcionalmente por IP para não autenticados). Ex.: N requisições por minuto por usuário. |
| Resposta 429 não definida no contrato | Cliente não sabe como tratar limite excedido | Incluir: 429 Too Many Requests quando limite excedido; header Retry-After (segundos) indicando quando retentar. |
| Timeout de requisição não definido | Requisições travadas consomem recursos | Definir timeout máximo de processamento (ex.: 30s); após isso retornar 504 Gateway Timeout ou 503 e liberar recursos. |
| Tamanho máximo de payload não definido | Body muito grande pode consumir memória e CPU | Definir limite (ex.: 1 MB ou 64 KB para este endpoint); payload acima do limite → 413 Payload Too Large. |
| Headers de rate limit não documentados | Cliente não sabe quanto pode consumir | Opcional: documentar headers X-RateLimit-Limit, X-RateLimit-Remaining, X-RateLimit-Reset em respostas bem-sucedidas. |

### Regras de Rate Limiting e Recursos Sugeridas para Documentação

- **Rate limiting**: Limite por usuário (ex.: 60 requisições/minuto para POST /transactions). Resposta 429 com header Retry-After quando excedido.
- **Timeout**: Tempo máximo de processamento da requisição (ex.: 30s); acima disso → 504 ou 503.
- **Payload**: Tamanho máximo do body (ex.: 64 KB); acima → 413 Payload Too Large.
- **Headers opcionais**: X-RateLimit-Limit, X-RateLimit-Remaining, X-RateLimit-Reset para facilitar consumo responsável pela aplicação cliente.

---

## Resumo dos Gaps (VADER)

| Dimensão | Gaps principais |
|----------|------------------|
| **Values** | Formato e limites de recipient_id e idempotency-key; tipo e teto de amount; tratamento de ausentes, null e body vazio. |
| **Authorization** | 401/403 no contrato; token expirado; idempotency por usuário; transferência para si mesmo. |
| **Data Consistency** | Isolamento e ordem na transação; validação antes da transação; concorrência (lock de saldo); idempotência por usuário. |
| **Error Handling** | 400, 401, 403, 429, 500, 503 no contrato; estrutura de corpo de erro; mensagens e códigos por tipo; Retry-After; não expor dados sensíveis. |
| **Rate Limiting & Resource Consumption** | Rate limiting por usuário; 429 e Retry-After; timeout; tamanho máximo de payload; headers de rate limit opcionais. |

---

## Sugestões de Códigos de Erro e Contrato

### Códigos de erro sugeridos (422 e outros)

- `RECIPIENT_ID_REQUIRED`: recipient_id ausente ou vazio.
- `AMOUNT_REQUIRED`: amount ausente ou null.
- `AMOUNT_BELOW_MINIMUM`: valor < 1.
- `AMOUNT_INVALID_TYPE`: amount não é número.
- `AMOUNT_NEGATIVE`: valor negativo.
- `AMOUNT_ABOVE_MAX`: valor > saldo ou > teto global (ou usar 402 para saldo).
- `RECIPIENT_ID_INVALID_FORMAT`: formato de recipient_id inválido.
- `INSUFFICIENT_BALANCE`: saldo insuficiente (402).
- `RECIPIENT_NOT_FOUND`: destinatário inexistente (404).
- `TOKEN_EXPIRED` / `TOKEN_INVALID`: autenticação (401).
- `RATE_LIMIT_EXCEEDED`: limite de taxa excedido (429).

### Contrato enriquecido (resumo)

- **Request**: Content-Type application/json obrigatório; body com recipient_id (string, UUID v4, 1–36 chars) e amount (number, inteiro, min 1, max min(saldo, TETO_GLOBAL)); header idempotency-key opcional (1–128 chars).
- **Resposta de sucesso**: 201 com transaction_id e new_balance.
- **Erros**: 400, 401, 403, 402, 404, 422, 429 (com Retry-After), 413, 500, 503; estrutura de erro com code e message; sem expor detalhes internos.

---

## Casos de Teste Sugeridos (VADER)

### Values

- amount = 0, -1, 1 (mínimo), saldo exato, saldo + 1 → respostas conforme regras (422, 201, 402).
- recipient_id ausente, null, "" → 400 ou 422.
- recipient_id formato inválido (não UUID) → 422.
- amount como string ou tipo errado → 422.
- Body vazio ou {} → 400.
- Payload acima do limite → 413.

### Authorization

- Request sem JWT → 401.
- Token expirado ou inválido → 401 com code/message.
- Token válido de usuário sem permissão (se aplicável) → 403.
- Idempotency-key: mesma key + mesmo usuário, duas requisições → mesma resposta (201, mesmo transaction_id).

### Data Consistency

- Transferência bem-sucedida: saldo remetente diminui e destinatário aumenta na mesma quantidade.
- Idempotência: repetir mesma request com mesma key → 201 e mesmo transaction_id, sem nova transferência.
- Saldo insuficiente: nenhuma alteração em saldos; 402.
- Destinatário inexistente: nenhuma alteração; 404.
- Concorrência: duas transferências simultâneas do mesmo usuário (saldo permite apenas uma) → uma 201 e uma 402; saldos consistentes.

### Error Handling

- Cada tipo de erro retorna código HTTP e corpo com code e message conforme contrato.
- Erro de validação (422): mensagem específica por campo.
- 500: mensagem genérica; sem stack trace.
- 429: presença de Retry-After.

### Rate Limiting & Resource Consumption

- Após exceder limite de requisições por usuário → 429 e Retry-After.
- Requisição que excede timeout (se simulável) → 504 ou 503.
- Payload acima do limite → 413.

---

## Checklist VADER Aplicado ao Requisito

- [ ] **Values**: Tipos e formatos (recipient_id, amount); ranges e limites; tratamento de ausentes e body vazio; tamanho máximo de payload/campos.
- [ ] **Authorization**: 401/403 no contrato; token expirado; remetente sempre do JWT; idempotency por usuário.
- [ ] **Data Consistency**: Transação atômica; validação antes/dentro da transação; concorrência (lock); idempotência com mesma key+usuário.
- [ ] **Error Handling**: Códigos 400, 401, 403, 402, 404, 422, 429, 413, 500, 503; estrutura de erro; mensagens por tipo; Retry-After para 429/503; não expor dados sensíveis.
- [ ] **Rate Limiting & Resource Consumption**: Rate limiting por usuário; 429 e Retry-After; timeout; tamanho máximo de payload.
- [ ] **Requisitos**: Gaps acima documentados no REQ_FINAL ou em anexo.
- [ ] **Testes**: Casos cobrindo as cinco dimensões (valores, autorização, consistência, erros, rate limit).

---

## Conclusão

A aplicação da heurística VADER ao requisito de Envio de QualiPoints permitiu identificar **gaps de especificação** nas cinco dimensões. O requisito já estabelece boas bases (JWT, atomicidade, idempotência, códigos 402/404/422), mas deve ser complementado com:

1. **Values**: Definições explícitas de formato (ex.: UUID para recipient_id), tipo e teto para amount, e tratamento de ausentes/null/body vazio.
2. **Authorization**: Inclusão de 401/403 no contrato, tratamento de token expirado e escopo de idempotency por usuário.
3. **Data Consistency**: Documentação de isolamento e ordem na transação, validação antes/dentro da transação e proteção contra concorrência (lock de saldo).
4. **Error Handling**: Contrato com 400, 401, 403, 429, 500, 503; estrutura padronizada de erro; códigos e mensagens por tipo; Retry-After; política de não expor dados sensíveis.
5. **Rate Limiting & Resource Consumption**: Rate limiting por usuário, 429 com Retry-After, timeout e tamanho máximo de payload.

Incorporar essas regras ao requisito e ao contrato da API alinha o sistema à heurística VADER, tornando a API mais robusta, segura, consistente e preparada para escala.
