# Casos de Teste EMOTIONS: Envio de QualiPoints (REQ_FINAL)

## Contexto do Caso

Este documento apresenta os **casos de teste** baseados na heurística **EMOTIONS** (Priscila Caimi e Jonatas Martins) para o requisito de **Envio de QualiPoints** ([setUp/REQ_FINAL.MD](../../setUp/REQ_FINAL.MD)). O objetivo é validar o impacto emocional do fluxo de transferência de pontos sobre o usuário, cobrindo as nove emoções: Alegria, Tristeza, Raiva, Medo, Nojinho, Surpresa, Confiança, Antecipação e Desconfiança.

## Entradas e Contrato em Análise

| Entrada | Tipo | Origem | Uso |
|--------|------|--------|-----|
| `recipient_id` | string | Body (JSON) | Identificador do destinatário |
| `amount` | number | Body (JSON) | Valor em QualiPoints a transferir |
| `user_id` | (implícito) | JWT token | Remetente |
| `idempotency-key` | string | Header | Evitar transferências duplicadas |

**Endpoint:** `POST /api/v1/transactions`  
**Fluxo:** Transferência síncrona; resposta em tempo real (201 Sucesso ou erro 402/404/422).  
**Cliente:** API e Painel Web (visualização de histórico; envio com formulário).

---

## 1. Alegria (Joy)

**Objetivo:** Verificar que o sistema proporciona satisfação, eficiência e feedback positivo nos fluxos de sucesso.

| ID | Cenário | Pré-condições | Passos | Resultado esperado (impacto emocional) | Critério de sucesso |
|----|---------|----------------|--------|----------------------------------------|---------------------|
| E-JOY-01 | Confirmação clara de sucesso | Usuário autenticado, saldo suficiente, destinatário ativo | 1. Enviar transferência válida (recipient_id, amount). 2. Receber resposta 201. | Usuário recebe confirmação explícita de que a operação foi concluída; corpo da resposta inclui `transaction_id` e `new_balance`. | Resposta 201 com corpo contendo `transaction_id` e `new_balance`; no Painel, mensagem de sucesso clara (ex.: "Transferência realizada com sucesso") e exibição do novo saldo. |
| E-JOY-02 | Eficiência do fluxo (um envio, uma confirmação) | Idem | 1. Preencher destinatário e valor. 2. Clicar em enviar uma vez. 3. Aguardar resposta. | Usuário conclui a tarefa com o mínimo de passos e sem necessidade de repetir ação. | Uma única requisição gera uma única confirmação; não é necessário reenviar ou confirmar em segunda tela (no escopo atual). |
| E-JOY-03 | Feedback positivo no Painel após sucesso | Idem | 1. Realizar transferência com sucesso pelo Painel. 2. Observar a tela após 201. | Usuário vê feedback positivo (mensagem e/ou atualização de saldo) que reforça a conclusão da meta. | Mensagem de sucesso visível; saldo atualizado (ou indicador de que foi atualizado); opção clara de nova transferência ou retorno ao histórico. |

---

## 2. Tristeza (Sadness)

**Objetivo:** Garantir que falhas e cenários de "não encontrado" ou "indisponível" não gerem perda de trabalho nem comunicação fria ou culpabilizadora.

| ID | Cenário | Pré-condições | Passos | Resultado esperado (impacto emocional) | Critério de sucesso |
|----|---------|----------------|--------|----------------------------------------|---------------------|
| E-SAD-01 | Erro 402 (saldo insuficiente) — tom empático | Usuário com saldo menor que o valor a transferir | 1. Enviar amount maior que o saldo. 2. Receber 402. | Usuário entende o motivo da falha sem se sentir culpado; mensagem sugere próximo passo (ex.: verificar saldo, reduzir valor). | Resposta 402 com mensagem em linguagem clara (ex.: "Saldo insuficiente. Verifique seu saldo e o valor informado."); sem códigos técnicos ou tom acusatório. |
| E-SAD-02 | Erro 404 (destinatário inexistente) — reconhecimento da frustração | Destinatário inexistente ou inativo | 1. Enviar com recipient_id inexistente ou inativo. 2. Receber 404. | Usuário sabe que o destinatário não foi encontrado e pode corrigir (ex.: verificar ID ou lista de destinatários). | Resposta 404 com mensagem clara (ex.: "Destinatário não encontrado ou não está ativo."); no Painel, formulário preservado para correção. |
| E-SAD-03 | Preservação do formulário após erro (Painel) | Qualquer erro 4xx retornado ao Painel | 1. Preencher destinatário e valor. 2. Enviar e receber 402 ou 404 ou 422. 3. Verificar a tela. | Usuário não perde o que digitou; pode ajustar e retentar sem refazer tudo. | Campos `recipient_id` e `amount` permanecem preenchidos; mensagem de erro exibida; botão de reenviar disponível. |
| E-SAD-04 | Erro 422 (validação) — mensagem acionável | Valor &lt; 1 ou formato inválido | 1. Enviar amount = 0 ou negativo ou formato inválido. 2. Receber 422. | Usuário entende qual regra foi violada e como corrigir. | Resposta 422 com mensagem ou detalhes por campo (ex.: "Valor mínimo é 1 QualiPoint"); formulário preservado no Painel. |

---

## 3. Raiva (Anger)

**Objetivo:** Evitar frustração extrema por mensagens confusas, bloqueios sem saída ou falta de opção de recuperação.

| ID | Cenário | Pré-condições | Passos | Resultado esperado (impacto emocional) | Critério de sucesso |
|----|---------|----------------|--------|----------------------------------------|---------------------|
| E-ANG-01 | Mensagem de erro não culpabiliza o usuário | Cenário que gera 402 ou 404 ou 422 | 1. Provocar erro de negócio. 2. Ler a mensagem retornada. | Texto não culpa o usuário (evitar "Você errou", "Sua solicitação é inválida" sem contexto); descreve o problema e sugere ação. | Mensagens sem pronomes culpabilizadores; foco em "saldo insuficiente", "destinatário não encontrado", "valor abaixo do mínimo" com próximo passo. |
| E-ANG-02 | Sem bloqueio sem saída — opção de retry (Painel) | Erro 4xx ou 5xx no Painel | 1. Causar erro ao enviar. 2. Verificar se há como continuar. | Usuário não fica preso; pode corrigir e reenviar ou tentar novamente. | Após erro: mensagem exibida, formulário preservado quando aplicável, botão ou link para "Tentar novamente" ou reenviar visível. |
| E-ANG-03 | Loading/espera com controle (Painel) | Envio em andamento | 1. Clicar em enviar. 2. Observar durante o processamento. | Usuário vê que a ação está em andamento; não há múltiplos cliques gerando dúvida. | Botão de envio desabilitado e/ou indicador de loading durante o POST; após resposta, botão reabilitado ou resultado exibido. |
| E-ANG-04 | Erro 500/503 — mensagem com próximo passo | Servidor retorna 500 ou 503 | 1. Simular ou provocar 500/503. 2. Ver resposta (API e/ou Painel). | Usuário entende que o problema é temporário e que pode tentar de novo; não fica sem direção. | Corpo de erro com mensagem tipo "Problema temporário. Tente novamente em instantes."; no Painel, mesma ideia e opção de retry. |

---

## 4. Medo (Fear)

**Objetivo:** Reduzir insegurança em ações irreversíveis e transmitir segurança na operação.

| ID | Cenário | Pré-condições | Passos | Resultado esperado (impacto emocional) | Critério de sucesso |
|----|---------|----------------|--------|----------------------------------------|---------------------|
| E-FEA-01 | Transferência é irreversível — usuário informado | Painel ou documentação | 1. Acessar tela/fluxo de envio. 2. Verificar se há aviso sobre irreversibilidade (se aplicável ao produto). | Se o negócio considerar a transferência irreversível, o usuário deve ser informado antes de confirmar (texto ou confirmação explícita). | Documentação ou UI indicam que a transferência não pode ser desfeita; ou confirmação em duas etapas no Painel antes do envio. |
| E-FEA-02 | Confirmação antes de enviar (Painel) | Painel com formulário de envio | 1. Preencher destinatário e valor. 2. Clicar em enviar. 3. Verificar se há confirmação. | Usuário tem chance de revisar valor e destinatário antes da operação definitiva (evita envio por clique acidental). | Botão de envio abre confirmação (modal ou segunda tela) com resumo (ex.: "Enviar X pontos para [destinatário]?") e opções "Confirmar" e "Cancelar"; ou requisito explicita que não há confirmação (decisão consciente). |
| E-FEA-03 | Transação atômica — sem débito sem crédito | Qualquer falha durante processamento | 1. Garantir que em falha (ex.: 404, 402, erro interno) nenhum débito é aplicado sem crédito. | Usuário não teme perder pontos por falha do sistema. | Em 402, 404, 422 e 5xx: saldo do remetente inalterado; apenas em 201 há débito e crédito aplicados. |

---

## 5. Nojinho (Disgust)

**Objetivo:** Garantir que a interface e as mensagens não sejam desorganizadas, poluídas ou inconsistentes.

| ID | Cenário | Pré-condições | Passos | Resultado esperado (impacto emocional) | Critério de sucesso |
|----|---------|----------------|--------|----------------------------------------|---------------------|
| E-DIS-01 | Mensagens de erro em linguagem clara | API e Painel | 1. Provocar 402, 404, 422. 2. Ler mensagens no corpo da resposta e no Painel. | Textos sem jargão técnico excessivo (ex.: evitar "0x4A", "NullPointerException"); português correto e profissional. | Mensagens em português claro; sem stack trace ou códigos internos expostos ao usuário final. |
| E-DIS-02 | Consistência entre API e Painel | Resposta da API exibida no Painel | 1. Causar erro na API. 2. Ver como o Painel exibe a mensagem. | O que a API retorna é refletido de forma consistente na tela (tradução ou mapeamento coerente). | Mensagem exibida no Painel é derivada da resposta da API e mantém o mesmo significado (ex.: saldo insuficiente → "Saldo insuficiente..."). |
| E-DIS-03 | Tela de envio organizada (Painel) | Painel de envio de QualiPoints | 1. Abrir a tela de transferência. 2. Avaliar hierarquia e clareza. | Campos e ações principais são visíveis; não há poluição por pop-ups ou banners antes do conteúdo essencial. | Campos destinatário e valor claramente identificados; botão de envio evidente; sem modais ou anúncios bloqueando o fluxo principal no primeiro carregamento (conforme desenho do produto). |

---

## 6. Surpresa (Surprise)

**Objetivo:** Evitar surpresas negativas (mudança de comportamento sem aviso); favorecer previsibilidade.

| ID | Cenário | Pré-condições | Passos | Resultado esperado (impacto emocional) | Critério de sucesso |
|----|---------|----------------|--------|----------------------------------------|---------------------|
| E-SUR-01 | Resposta 201 contém o prometido | Requisito define transaction_id e new_balance | 1. Realizar transferência com sucesso. 2. Verificar corpo da resposta. | Usuário recebe exatamente o que o contrato promete: identificador da transação e novo saldo. | Corpo da resposta 201 contém `transaction_id` (UUID) e `new_balance` (number); nenhum campo essencial faltando. |
| E-SUR-02 | Códigos de erro alinhados ao contrato | Requisito cita 402, 404, 422 | 1. Provocar saldo insuficiente, destinatário inexistente, valor inválido. 2. Verificar status HTTP. | Usuário (ou cliente da API) recebe o código esperado para cada situação, sem mudança súbita de contrato. | 402 para saldo insuficiente; 404 para destinatário inexistente/inativo; 422 para valor abaixo do mínimo ou formato inválido. |
| E-SUR-03 | Idempotency — mesma chave, mesma resposta | Dois envios com mesma idempotency-key | 1. Enviar transferência válida com idempotency-key K. 2. Receber 201. 3. Reenviar mesma requisição com mesma key K. | Segunda requisição não gera segunda transferência; usuário não é surpreendido por débito duplicado. | Segunda resposta 201 com mesmo `transaction_id`; apenas uma transferência no banco. |

---

## 7. Confiança (Trust)

**Objetivo:** Reforçar que o sistema cumpre promessas e mantém consistência.

| ID | Cenário | Pré-condições | Passos | Resultado esperado (impacto emocional) | Critério de sucesso |
|----|---------|----------------|--------|----------------------------------------|---------------------|
| E-TRU-01 | new_balance reflete o débito real | Saldo conhecido, transferência bem-sucedida | 1. Anotar saldo antes. 2. Transferir valor V. 3. Receber 201 com new_balance. | O novo saldo retornado é exatamente saldo_anterior - V. | `new_balance` = saldo do remetente após débito; consistente com regra de negócio. |
| E-TRU-02 | Histórico ou log em tela (Painel) | Requisito menciona "log em tela" | 1. Realizar uma ou mais transferências. 2. Acessar histórico/log no Painel. | Usuário pode verificar que a operação foi registrada e rastrear. | Após transferência, há forma de visualizar a transação (ex.: lista ou log) com dados coerentes (valor, destinatário, data/hora). |
| E-TRU-03 | Nenhum débito em caso de erro de negócio | Cenários 402, 404, 422 | 1. Provocar cada erro. 2. Verificar saldo do remetente antes e depois. | Usuário confia que em erro de validação ou negócio nenhum ponto foi debitado. | Saldo do remetente inalterado após 402, 404 e 422. |

---

## 8. Antecipação (Anticipation)

**Objetivo:** Garantir que esperas longas tenham feedback e que prazos ou comportamentos sejam previsíveis.

| ID | Cenário | Pré-condições | Passos | Resultado esperado (impacto emocional) | Critério de sucesso |
|----|---------|----------------|--------|----------------------------------------|---------------------|
| E-ANT-01 | Resposta em tempo real (síncrona) | Requisito define operação síncrona | 1. Enviar transferência. 2. Medir tempo até resposta. | Usuário recebe confirmação ou erro na mesma requisição, sem precisar consultar outro endpoint para saber o resultado. | Resposta 201 ou 4xx/5xx retornada na mesma chamada POST; não é necessário polling para obter resultado. |
| E-ANT-02 | Indicador de carregamento durante envio (Painel) | Painel envia POST | 1. Clicar em enviar. 2. Observar até a resposta. | Usuário sabe que o sistema está processando; a expectativa é gerenciada. | Loading ou botão desabilitado visível durante o request; ao receber resposta, loading some e resultado ou erro é exibido. |
| E-ANT-03 | Tempo de resposta dentro do aceitável | Ambiente de teste estável | 1. Enviar transferência válida. 2. Medir latência. | Resposta não demora tanto a ponto de o usuário achar que travou (conforme SLA ou meta do projeto). | Latência da API dentro do aceitável definido (ex.: &lt; 3s em condições normais); ou documentar que sob carga o tempo pode aumentar e 503 pode ser retornado. |

---

## 9. Desconfiança (Distrust)

**Objetivo:** Evitar inconsistências e comportamentos que gerem ceticismo.

| ID | Cenário | Pré-condições | Passos | Resultado esperado (impacto emocional) | Critério de sucesso |
|----|---------|----------------|--------|----------------------------------------|---------------------|
| E-DST-01 | Saldo consistente entre resposta e estado real | Transferência 201 | 1. Fazer transferência. 2. Obter new_balance na resposta. 3. Consultar saldo (se houver endpoint ou tela). | O saldo exibido em outra consulta bate com o new_balance retornado. | new_balance da resposta 201 igual ao saldo do remetente em consulta subsequente (consistência de leitura). |
| E-DST-02 | Idempotency evita duplicidade visível | Dois envios com mesma idempotency-key | 1. Enviar com key K; receber 201. 2. Reenviar com mesma key K; receber 201. 3. Verificar histórico/saldos. | Uma única transferência; usuário não vê duas operações idênticas. | Uma única linha de transferência no histórico; saldo debitado uma vez. |
| E-DST-03 | user_id do token, não do body | Requisito: user_id do JWT | 1. Enviar requisição com body contendo outro user_id (se o contrato permitir algum campo). 2. Verificar quem foi debitado. | Remetente é sempre o dono do token; não há possibilidade de "enviar em nome de outro" por manipulação de body. | Débito sempre aplicado ao user_id extraído do JWT; body não pode sobrescrever o remetente. |

---

## Resumo por Emoção

| Emoção | Quantidade de casos | Foco principal |
|--------|---------------------|-----------------|
| Alegria | 3 | Confirmação de sucesso, eficiência, feedback positivo |
| Tristeza | 4 | Mensagens empáticas, preservação de formulário, sem perda de trabalho |
| Raiva | 4 | Mensagens não culpabilizadoras, opção de retry, loading, 500/503 com próximo passo |
| Medo | 3 | Irreversibilidade comunicada, confirmação antes de enviar, atomicidade |
| Nojinho | 3 | Linguagem clara, consistência API/Painel, UI organizada |
| Surpresa | 3 | Contrato 201 respeitado, códigos corretos, idempotência |
| Confiança | 3 | new_balance correto, histórico/log, sem débito em erro |
| Antecipação | 3 | Resposta síncrona, loading no Painel, latência aceitável |
| Desconfiança | 3 | Consistência de saldo (E-DST-01), idempotência visível (E-DST-02), remetente do token (E-DST-03) |

---

## Checklist EMOTIONS Aplicado ao Requisito

- [ ] **Alegria**: Feedback positivo e eficiência nos fluxos de sucesso (E-JOY-01 a E-JOY-03).
- [ ] **Tristeza**: Perda evitada, mensagens empáticas, formulário preservado (E-SAD-01 a E-SAD-04).
- [ ] **Raiva**: Mensagens claras, retry possível, loading, 500/503 com direção (E-ANG-01 a E-ANG-04).
- [ ] **Medo**: Confirmação e atomicidade (E-FEA-01 a E-FEA-03).
- [ ] **Nojinho**: Clareza e consistência (E-DIS-01 a E-DIS-03 na seção Nojinho).
- [ ] **Surpresa**: Contrato estável, códigos e idempotência (E-SUR-01 a E-SUR-03).
- [ ] **Confiança**: Dados consistentes e cumprimento de promessas (E-TRU-01 a E-TRU-03).
- [ ] **Antecipação**: Feedback de progresso e resposta previsível (E-ANT-01 a E-ANT-03).
- [ ] **Desconfiança**: Sem inconsistências (E-DST-01 a E-DST-03).

---

## Conclusão

Os casos de teste acima cobrem o fluxo de **Envio de QualiPoints** sob a lente das nove emoções da heurística EMOTIONS. Eles podem ser executados na API (contrato e corpo de resposta) e no Painel (formulário, mensagens, loading, preservação de estado e histórico). Aplicá-los antes da liberação da funcionalidade ajuda a garantir que a experiência do usuário reforce satisfação e confiança e minimize frustração, medo e desconfiança.

*Heurística Emotions — Priscila Caimi e Jonatas Martins. A jornada do usuário sob a lente dos sentimentos.*
