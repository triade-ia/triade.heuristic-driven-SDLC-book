---
title: Análise FLOOD — REQ_INICIAL_V2
based_on: output/requisito-revisado/REQ_INICIAL_V2.md
heuristic: Skill/Cap09_teste/FLOOD.md
date: 2026-02-21
author: Análise automática (heurística FLOOD)
---

# Relatório de Análise FLOOD — Requisito `REQ_INICIAL_V2`

Resumo: análise do requisito de envio de QualiPoints aplicando as cinco dimensões da heurística FLOOD (Picos de Requisições, Transações Concorrentes, Entrada de Dados Massiva, Degradação da Performance, Estabilidade e Recuperação). Identifica gaps, riscos e recomendações práticas para requisitos, implementação e testes.

## Achados por dimensão

1) Picos de Requisições
- Observações:
  - O requisito define processamento síncrono com timeout de 10s e idempotency key; não há limites de taxa nem políticas de rejeição/graceful degradation.
  - Não há definição de limites por período (RPS, RPM) — explicitamente adiado para backlog.
- Riscos:
  - Sem rate limiting, um pico de requisições (ou retentativas em massa) pode esgotar threads, conexões ao banco e causar latência elevada ou erros 5xx.
  - Idempotência evita débito duplicado, mas não mitiga sobrecarga (thundering herd) quando muitos clientes repetem tentativas simultâneas.
- Recomendações:
  - Adicionar requisitos de rate limiting (por usuário e global) e comportamento quando excedido (429/503).
  - Definir política de debounce/disable no cliente (desabilitar botão enviar até resposta) — já sugerido no requisito UI, garantir implementação.

2) Transações Concorrentes
- Observações:
  - Requisito exige atomicidade (débito + crédito + registro em mesma transação) e uso de idempotency key.
  - Não especifica detalhamento de estratégia de bloqueio/isolation (SELECT FOR UPDATE, optimistic locking, nível de isolamento serializable) nem tratamento de deadlocks.
- Riscos:
  - Alterações concorrentes ao mesmo saldo podem provocar overspending se a estratégia de locking não for adequada (ex.: leitura-antes-de-escrever sem lock).
  - Possibilidade de deadlocks ou contenção no banco em alta concorrência se atualizações de saldo travarem muitas linhas.
- Recomendações:
  - Especificar técnica: uso de operação atômica no banco (ex.: UPDATE com verificação de saldo e constraint CHECK, ou SELECT FOR UPDATE + verificação) e persistência de idempotency key de forma atômica.
  - Adicionar constraints no DB (saldo >= 0) e usar transações curtas; considerar serializable ou row-level locks conforme custo/throughput.

3) Entrada de Dados Massiva
- Observações:
  - O fluxo primário não contempla uploads nem grandes payloads; portanto impacto direto é menor.
  - Porém, possibilidade de envio em massa por scripts/abusos (bulk transfers) não está limitada.
- Riscos:
  - Envio massivo de várias transferências síncronas pode monopolizar recursos e levar a timeouts generalizados.
- Recomendações:
  - Definir limites por requisição (já há máximo por transação) e limites por período/usuário para prevenir abuso.
  - Considerar endpoint batch assíncrono para cargas legítimas de volume (fora do MVP), com processamento em fila.

4) Degradação da Performance
- Observações:
  - SLA operacional de tempo de resposta: API deve responder em até 10s; não há metas para p50/p95/p99 nem métricas/alertas descritos.
- Riscos:
  - Sem métricas e alertas não há visibilidade de degradação progressiva; falhas podem ocorrer sem aviso até crash.
- Recomendações:
  - Definir objetivos de desempenho (ex.: p95 < 300ms até X RPS) e instrumentar telemetria: latência (p50/p95/p99), taxa de erro, utilização CPU/memória, pool de conexões DB, tempos de lock e filas.
  - Configurar alertas que disparem antes do ponto de ruptura (ex.: aumento de p95, saturação de conexões DB, crescimento de filas de backpressure).

5) Estabilidade e Recuperação
- Observações:
  - Requisito descreve comportamento de timeout e retry com idempotency-key, e que não há estado "pendente" no servidor. Não há menção a circuit breaker, rate limiter no gateway, backpressure, ou políticas de rejeição graciosa.
- Riscos:
  - Em sobrecarga, API pode começar a responder 500 ou reiniciar se recursos se esgotarem; sem circuit breaker/rejeição graciosa, falha em cascata é provável.
- Recomendações:
  - Implementar camada de proteção: rate limiting, circuit breaker para dependências críticas (DB), limites de concorrência por instância, e retornar 429/503 quando necessário.
  - Documentar estratégia de recuperação automática (health checks, auto-scaling, fallback) e exigir testes de recuperação.

## Gaps do requisito (prioritizados)
- Falta de limites de taxa e política de rejeição (alta prioridade).
- Ausência de métricas e alertas (alta prioridade).
- Estratégia de concurrency control DB não especificada (média-alta).
- Persistência e retenção das `Idempotency-Key` não definidos (média).
- Comportamento esperado sob picos (ex.: resposta 429 vs 504) não definido claramente (média).
- Testes de carga/estresse/soak/spike não especificados (média).

## Recomendação de requisitos adicionais (exemplos concisos)
- Rate limiting: `N` req/s por usuário e `M` req/s global; comportamento: retornar `429 Too Many Requests` com Retry-After.
- SLAs operacionais: medir p50/p95/p99; definir alvo (ex.: p95 < 500ms até 100 RPS por endpoint crítico).
- Idempotency: persistir chave por pelo menos 24h com resultado e erro correspondente; documentar formato e armazenamento.
- Concurrency DB: exigir que a equipe implemente atualização de saldo com lock de linha (SELECT FOR UPDATE) ou operação SQL atômica e constraint `saldo >= 0`.
- Observabilidade: métricas (latência p50/p95/p99, erros, DB connections, queue length) e alertas pré-definidos.

## Recomendações de implementação (práticas)
- Persistir `Idempotency-Key` e resultado atomically para evitar reprocessamento e para devolver o mesmo resultado às retries.
- Atualização de saldo:
  - Usar transação curta com row-level locking (ou operação UPDATE ... WHERE saldo >= amount RETURNING ...) para garantir atomicidade sem oversell.
  - Garantir índices e queries rápidas para reduzir tempo de lock.
- Evitar transações longas que travem conexões DB em picos; manter lógica simples no DB e delegar trabalhos pesados para filas (se necessário).
- Implementar rate limiter no API gateway (token bucket) e limites por usuário/por IP.
- Implementar circuit breaker e limites máximos de conexão para dependências.

## Plano de testes sugerido (mínimo)
- Testes de carga (ramp): aumentar RPS até degradar latência; medir throughput máximo antes de p95 ultrapassar SLAs.
- Testes de estresse: submeter picos até a falha para validar comportamento (espera-se 429/503, não 500/crash).
- Spike tests: súbito grande volume seguido de retorno ao normal; verificar recuperação automática.
- Concurrency tests: várias requisições simultâneas do mesmo remetente tentando gastar o mesmo saldo — verificar ausência de oversell e integridade de dados.
- Retry storm: simular clientes que reenviam com a mesma/diferente idempotency-key para validar idempotência e proteção contra thundering herd.
- Soak tests (opcional): carga sustentada para detectar leaks ou degradação lenta.

## Ações recomendadas imediatas (curto prazo)
1. Adicionar requisitos mínimos de rate limiting e comportamento (429 + Retry-After).  
2. Especificar a estratégia de locking/isolamento para updates de saldo e exigir constraints DB para saldo >= 0.  
3. Persistir `Idempotency-Key` e resultado por 24h (mínimo).  
4. Exigir instrumentação básica (latência p95/p99, taxa de erro, conexões DB) e definição de alertas.  
5. Planejar e executar testes: concurrency + spike + recovery antes do deploy em produção.

## Conclusão
O requisito `REQ_INICIAL_V2` cobre bem aspectos funcionais e traz medidas importantes (atomicidade e idempotência). Porém, para prosperar sob alta carga, ele precisa de requisitos complementares sobre limites de taxa, estratégia de concorrência detalhada, métricas/alertas e políticas de rejeição/graceful degradation. Implementar as recomendações reduzirá risco de falhas em picos, manterá a integridade financeira e facilitará operação/recuperação em produção.

---
Arquivo gerado automaticamente — revisão técnica recomendada.
