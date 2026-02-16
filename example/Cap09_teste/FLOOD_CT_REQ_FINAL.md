# Casos de Teste FLOOD: Envio de QualiPoints (REQ_FINAL)

## Contexto do Caso

Este documento apresenta os **casos de teste** baseados na heurística **FLOOD** (Inundação) para o requisito de **Envio de QualiPoints** ([setUp/REQ_FINAL.MD](../../setUp/REQ_FINAL.MD)). O objetivo é validar a resiliência do sistema sob pressão extrema: picos de requisições, transações concorrentes, limites de volume, degradação de performance e estabilidade/recuperação, garantindo que o fluxo de transferência suporte alta demanda sem travar, degradar de forma inaceitável ou corromper dados.

## Entradas e Contrato em Análise

| Entrada | Tipo | Origem | Uso |
|--------|------|--------|-----|
| `recipient_id` | string | Body (JSON) | Identificador do destinatário |
| `amount` | number | Body (JSON) | Valor em QualiPoints a transferir |
| `user_id` | (implícito) | JWT token | Remetente — nunca do body |
| `idempotency-key` | string | Header | Evitar transferências duplicadas |

**Endpoint:** `POST /api/v1/transactions`  
**Fluxo:** Transferência síncrona com transação atômica (débito + crédito); rollback se crédito no destinatário falhar. Resposta em tempo real (Sucesso ou Erro).  
**Cliente:** API e Painel Web (visualização de histórico; envio com formulário).

---

## 1. Picos de Requisições

**Objetivo:** Verificar o comportamento quando muitos usuários ou muitas requisições acessam o endpoint de transferência ao mesmo tempo, e se a UI protege contra múltiplos envios.

| ID | Cenário | Pré-condições | Passos | Resultado esperado | Critério de sucesso |
|----|---------|----------------|--------|---------------------|---------------------|
| FL-PIC-01 | Múltiplos usuários transferindo simultaneamente | N usuários autenticados com saldo e destinatários válidos; ferramenta de carga ou scripts | 1. Disparar transferências válidas de N usuários distintos em paralelo (ex.: 10, 50, 100 conforme meta). 2. Medir taxa de sucesso (201), latência e taxa de erro. | Sistema processa as requisições sem travar; respostas 201 ou 4xx conforme regras de negócio; sem 500 em cascata. | Todas as requisições recebem resposta (201, 402, 404 ou 422); nenhum timeout sem resposta; saldos finais consistentes (uma transferência por 201). |
| FL-PIC-02 | Idempotency-key evita duplicata por múltiplos cliques | Um usuário, transferência válida | 1. Enviar a mesma transferência duas ou mais vezes em sequência rápida com a **mesma** idempotency-key. 2. Verificar respostas e saldos. | Apenas a primeira requisição resulta em transferência efetiva; demais retornam 201 com mesmo transaction_id. | Uma única transferência no banco; todas as respostas 201 com mesmo transaction_id; saldo debitado uma vez. |
| FL-PIC-03 | Botão desabilitado durante envio (Painel) | Painel de envio disponível | 1. Preencher destinatário e valor. 2. Clicar em enviar. 3. Durante o processamento, tentar clicar novamente no botão. | Botão permanece desabilitado e/ou loading visível até a resposta; segundo clique não dispara nova requisição (ou segunda requisição usa mesma idempotency-key). | Botão desabilitado ou loading ativo durante o POST; não há duas requisições com payloads diferentes (ou ambas com mesma idempotency-key). |
| FL-PIC-04 | Rate limit (429) quando aplicável | Sistema com rate limiting configurado | 1. Enviar número de requisições acima do limite (por user_id ou IP) em janela de tempo definida. 2. Verificar status da resposta. | Após exceder o limite, API retorna 429 Too Many Requests; corpo ou headers indicam retry (ex.: Retry-After). | Resposta 429 quando limite é excedido; nenhuma transferência processada para as requisições rejeitadas; documentação descreve o limite. |

---

## 2. Transações Concorrentes

**Objetivo:** Garantir que múltiplas transferências simultâneas do mesmo remetente ou para o mesmo destinatário não gerem race condition, saldo negativo ou corrupção de dados.

| ID | Cenário | Pré-condições | Passos | Resultado esperado | Critério de sucesso |
|----|---------|----------------|--------|---------------------|---------------------|
| FL-CON-01 | Dois envios simultâneos do mesmo remetente (saldo suficiente para apenas um) | Remetente com saldo S; duas transferências de valor S cada (ou soma > S) com idempotency-keys **diferentes** | 1. Disparar em paralelo duas requisições do mesmo usuário: transferência 1 (valor S) e transferência 2 (valor S). 2. Verificar respostas e saldo final. | Uma transferência retorna 201 e a outra 402 (saldo insuficiente); saldo final do remetente zero ou positivo; nenhum débito total superior ao saldo inicial. | Validação de saldo e débito atômicos: no máximo uma transferência completa; a outra recebe 402; saldo do remetente nunca negativo. |
| FL-CON-02 | Múltiplos créditos simultâneos no mesmo destinatário | Vários remetentes, mesmo destinatário ativo | 1. Disparar em paralelo várias transferências para o mesmo recipient_id (de usuários diferentes). 2. Verificar respostas 201 e saldo do destinatário. | Todas as transferências válidas são processadas; saldo do destinatário = soma dos créditos; nenhuma perda ou duplicação. | Saldo do destinatário igual ao anterior + soma dos amounts transferidos; número de transações de crédito igual ao número de 201. |
| FL-CON-03 | Concorrência: mesma idempotency-key em duas requisições paralelas | Uma transferência válida, mesma idempotency-key K | 1. Enviar duas requisições idênticas (mesmo body e mesma idempotency-key K) em paralelo. 2. Verificar respostas e banco. | Ambas podem retornar 201 com o mesmo transaction_id; apenas uma transferência persistida. | Uma única linha de transação; saldo debitado uma vez; ambas as respostas 201 com mesmo transaction_id (ou uma 201 e outra 201/409 conforme implementação). |
| FL-CON-04 | Múltiplas transferências do mesmo remetente em sequência rápida (saldo suficiente para todas) | Remetente com saldo suficiente para N transferências válidas | 1. Enviar N transferências válidas em sequência rápida (keys diferentes). 2. Verificar todas as respostas 201 e saldo final. | Todas retornam 201; saldo final = saldo inicial - soma dos amounts; integridade mantida. | N respostas 201; saldo do remetente = saldo inicial - soma(amount); nenhum 402 indevido; dados consistentes. |

---

## 3. Entrada de Dados Massiva

**Objetivo:** Verificar limites de payload e comportamento quando o cliente envia volume alto de requisições ou body fora do esperado.

| ID | Cenário | Pré-condições | Passos | Resultado esperado | Critério de sucesso |
|----|---------|----------------|--------|---------------------|---------------------|
| FL-MAS-01 | Body maior que o limite (se definido) | Limite de tamanho de body documentado (ex.: 4 KB) | 1. Enviar POST com body muito grande (ex.: payload de vários KB). 2. Verificar status da resposta. | API rejeita com 400 ou 413 (Request Entity Too Large); nenhuma transferência processada. | Status 400 ou 413; corpo com mensagem de rejeição; saldos inalterados. |
| FL-MAS-02 | Volume alto de requisições em sequência (um cliente) | Um usuário, muitos destinatários válidos, saldo suficiente | 1. Enviar muitas transferências em sequência rápida (ex.: 50 ou 100) do mesmo usuário. 2. Verificar se todas são processadas ou se há 429/503. | Sistema aceita até o limite definido ou rejeita com 429/503 de forma controlada; nenhum crash. | Comportamento conforme documentação: ou todas 201 (dentro do limite) ou 429/503 ao exceder; sem 500 em cascata. |
| FL-MAS-03 | Campos extras no body (payload mínimo) | Contrato define apenas recipient_id e amount | 1. Enviar body com recipient_id, amount e campos extras (ex.: descrição, metadados). 2. Verificar se a API ignora extras e processa ou rejeita. | API processa (ignorando extras) ou rejeita com 400; comportamento documentado. | Resposta 201 ou 400 conforme especificação; se 201, apenas recipient_id e amount usados; transferência correta. |
| FL-MAS-04 | Ausência de operação em lote no escopo | Requisito não prevê endpoint de transferência em lote | 1. Verificar contrato e documentação. | Não existe endpoint que aceite múltiplas transferências em uma única requisição; alto volume = muitas requisições. | Documentação deixa claro que cada POST é uma transferência; testes de volume usam múltiplas requisições. |

---

## 4. Degradação da Performance

**Objetivo:** Identificar a partir de qual carga a latência sobe de forma perceptível e se o sistema degrada de forma gradual ou falha de forma abrupta.

| ID | Cenário | Pré-condições | Passos | Resultado esperado | Critério de sucesso |
|----|---------|----------------|--------|---------------------|---------------------|
| FL-PER-01 | Latência sob carga esperada | Meta de latência definida (ex.: p95 &lt; 2s); ambiente estável | 1. Aplicar carga correspondente ao uso esperado (ex.: X req/s). 2. Medir latência (p50, p95, p99) das respostas 201. | Latência permanece dentro do SLA ou meta definida. | p95 (e p99 quando aplicável) dentro do aceitável; taxa de erro baixa (ex.: &lt; 1%). |
| FL-PER-02 | Aumento progressivo da carga até degradação | Ferramenta de carga configurável | 1. Aumentar gradualmente o número de requisições simultâneas (ex.: 5, 10, 20, 50, 100). 2. Registrar latência e taxa de erro em cada estágio. | À medida que a carga sobe, a latência aumenta ou a taxa de erro sobe de forma identificável; ponto de ruptura ou degradação pode ser registrado. | Relatório ou métricas indicando a partir de quantas req/s a latência excede o aceitável ou erros (5xx/503) começam; útil para capacidade. |
| FL-PER-03 | Spike test — pico súbito de requisições | Sistema em repouso ou carga baixa | 1. Submeter pico súbito de requisições (ex.: 10x a carga normal por 30s). 2. Verificar se o sistema retorna 201/4xx ou 503/500; se recupera após o pico. | Sistema responde com 201/4xx quando possível ou rejeita com 503/429; após redução da carga, volta a responder normalmente. | Não há crash nem reinício em loop; respostas 503 ou 429 preferíveis a 500 em cascata; recuperação após o pico. |
| FL-PER-04 | Soak test — carga sustentada (opcional) | Ambiente de teste disponível por tempo prolongado | 1. Manter carga constante (ex.: 70% do limite estimado) por período definido (ex.: 30 min ou 1 h). 2. Monitorar latência, taxa de erro e recursos (CPU, memória). | Sistema mantém estabilidade; sem degradação progressiva nem vazamento de recursos. | Latência e taxa de erro estáveis ao longo do tempo; memória/CPU estáveis (sem crescimento contínuo). |

---

## 5. Estabilidade e Recuperação

**Objetivo:** Garantir que sob estresse o sistema rejeite de forma graciosa (429/503) quando aplicável e se recupere após o pico sem intervenção manual.

| ID | Cenário | Pré-condições | Passos | Resultado esperado | Critério de sucesso |
|----|---------|----------------|--------|---------------------|---------------------|
| FL-EST-01 | Excesso de requisições — 429 ou 503 em vez de 500 | Carga acima da capacidade ou rate limit excedido | 1. Submeter carga que exceda o limite (rate limit ou capacidade do serviço). 2. Verificar status das respostas de rejeição. | API retorna 429 (Too Many Requests) ou 503 (Service Unavailable) em vez de 500; mensagem ou header indicam retry. | Respostas 429 ou 503; corpo ou Retry-After indicam que é temporário; nenhuma transferência processada para as requisições rejeitadas. |
| FL-EST-02 | Recuperação após pico de carga | Pico aplicado; depois carga reduzida ou zero | 1. Aplicar pico de carga até 503/429 ou degradação. 2. Reduzir carga ao normal ou zerar. 3. Enviar novas transferências válidas. | Após alguns segundos/minutos, o sistema volta a aceitar requisições e responder 201 normalmente. | Novas requisições após redução de carga recebem 201 (ou 4xx por regra de negócio); não é necessário restart manual do serviço. |
| FL-EST-03 | Ausência de crash ou OOM sob carga alta | Ferramenta de carga; monitoramento de processo/serviço | 1. Aumentar carga até o limite ou até respostas 503/429. 2. Verificar se o processo do serviço permanece estável (não termina com OOM ou crash). | Serviço permanece em execução; rejeição graciosa (429/503) preferível a queda do processo. | Processo não é finalizado por OOM ou exceção fatal; logs indicam rejeição ou throttling em vez de crash. |
| FL-EST-04 | Integridade dos dados após pico e recuperação | Saldos conhecidos antes do pico | 1. Registrar saldos de remetentes e destinatários. 2. Aplicar pico com transferências válidas e rejeições (429/503). 3. Após recuperação, consultar saldos e histórico. | Apenas transferências que receberam 201 refletem em saldo; rejeitadas (429/503) não geram débito/crédito; saldos consistentes. | Soma de débitos = soma de créditos; saldos batem com transações efetivamente processadas (201); nenhuma operação "fantasma". |

---

## Resumo por Dimensão

| Dimensão | Quantidade de casos | Foco principal |
|----------|---------------------|----------------|
| Picos de Requisições | 4 | Múltiplos usuários, idempotency-key, botão desabilitado no Painel, rate limit (429) |
| Transações Concorrentes | 4 | Saldo atômico (sem overspend), múltiplos créditos no mesmo destinatário, idempotency em paralelo, sequência rápida |
| Entrada de Dados Massiva | 4 | Limite de body, volume de requisições em sequência, payload com extras, ausência de batch |
| Degradação da Performance | 4 | Latência sob carga esperada, aumento progressivo até degradação, spike test, soak test |
| Estabilidade e Recuperação | 4 | 429/503 em vez de 500, recuperação após pico, sem crash/OOM, integridade após pico |

---

## Checklist FLOOD Aplicado ao Requisito

- [ ] **Picos de Requisições**: Múltiplos acessos simultâneos (FL-PIC-01); idempotency evita duplicata (FL-PIC-02); Painel desabilita botão (FL-PIC-03); rate limit 429 quando aplicável (FL-PIC-04).
- [ ] **Transações Concorrentes**: Concorrência no saldo do remetente (FL-CON-01); múltiplos créditos no mesmo destinatário (FL-CON-02); idempotency em paralelo (FL-CON-03); sequência rápida (FL-CON-04).
- [ ] **Entrada de Dados Massiva**: Limite de body (FL-MAS-01); volume em sequência (FL-MAS-02); payload com campos extras (FL-MAS-03); sem batch no escopo (FL-MAS-04).
- [ ] **Degradação da Performance**: Latência sob carga (FL-PER-01); aumento até degradação (FL-PER-02); spike test (FL-PER-03); soak test quando aplicável (FL-PER-04).
- [ ] **Estabilidade e Recuperação**: 429/503 em excesso (FL-EST-01); recuperação após pico (FL-EST-02); sem crash/OOM (FL-EST-03); integridade após pico (FL-EST-04).

---

## Conclusão

Os casos de teste acima cobrem o fluxo de **Envio de QualiPoints** sob as cinco dimensões da heurística FLOOD. Eles validam que o sistema suporta picos de requisições (com proteção na UI e idempotency), mantém integridade sob transações concorrentes, respeita limites de volume quando definidos, permite observar degradação de performance e se mantém estável ou rejeita de forma graciosa (429/503) e se recupera após pico. A execução desses casos contribui para que o sistema **prospere sob alta demanda**, garantindo continuidade do serviço, integridade dos dados e previsibilidade sob carga.

*Heurística Flood — Testando a resiliência sob pressão extrema.*
