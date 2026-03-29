---
name: test-strategy
description: Analisa requisitos e código para sugerir estratégia de testes. Identifica heurísticas aplicáveis, define distribuição por camada, gera casos de teste e SALVA RELATÓRIO em output/test. Use quando precisar decidir QUAIS testes criar e ONDE. Sempre gere relatório estruturado ao finalizar análise.
---

# Estratégia de Testes

Skill orquestradora que analisa requisitos e/ou código, identifica heurísticas aplicáveis, define distribuição por camada e **gera relatório estruturado** para uso pelos guides de implementação.

## Atuação

Você é um especialista em estratégia de testes. Ao receber **requisito funcional** e/ou **código** (função, endpoint, componente):

1. **Identifique heurísticas aplicáveis** por fase do SDLC (ver tabela abaixo)
2. **Defina distribuição por camada** (pirâmide: 70% unitários, 20% integração, 5% componentes, 3% serviço, 2% E2E)
3. **Gere casos de teste detalhados** com IDs, inputs, outputs esperados e prioridade
4. **Crie relatório estruturado** e salve em `output/test/strategy/test-strategy-[nome-funcao]-[YYYYMMDD].md`
5. **Recomende próximos passos** (qual guide consultar e como passar o relatório)

**IMPORTANTE**: Ao finalizar a análise, SEMPRE gerar o arquivo de relatório em `output/test/strategy/` para que o desenvolvedor possa usá-lo como contexto ao chamar os guides.

## Processo de Análise

### 1. Identificação de Heurísticas

Para cada requisito/código, avalie quais heurísticas aplicar. As heurísticas estão organizadas por fase do SDLC:

#### Desenvolvimento (`/heuristic-guide-development`)

| Se o requisito/código envolve... | Aplique heurística |
|----------------------------------|---------------------|
| Entrada de dados (inputs, payloads) | **Baica** (Boundaries, Nulls, Special Chars, Formats, Failure Patterns) |
| API/Endpoint HTTP | **VADER** (Values, Authorization, Data Consistency, Error Handling, Rate Limiting) |

#### Arquitetura de Código (`/heuristic-guide-architeture-code`)

| Se o requisito/código envolve... | Aplique heurística |
|----------------------------------|---------------------|
| Operações Create, Read, Update, Delete | **CRUD** (segurança, performance, consistência por operação) |
| Quantidades variáveis (0, 1, N itens) | **Count** (comportamento nos 3 extremos de volume) |

#### Design de Código (`/heuristic-guide-desing-code`)

| Se o requisito/código envolve... | Aplique heurística |
|----------------------------------|---------------------|
| Múltiplos usuários simultâneos, recursos compartilhados | **Multi-User** (Colisão, Sessão Dupla, Inventário Crítico, UX de Conflito) |

#### QA & Testes (`/heuristic-guide-qa-test`)

| Se o requisito/código envolve... | Aplique heurística |
|----------------------------------|---------------------|
| Cenários de falha, rollback, recuperação | **FAILURE** (Functional, Appropriate, Impact, Log, UI, Recovery, Emotions) |
| Diferentes perfis e estados emocionais do usuário | **EMOTIONS** (9 emoções: Alegria, Tristeza, Raiva, Medo, Nojinho, Surpresa, Confiança, Antecipação, Desconfiança) |
| Carga massiva, stress, picos de requisições | **FLOOD** (picos de requisições, transações concorrentes, entrada massiva, degradação, estabilidade) |

#### Pós-Deploy (`/heuristic-guide-pos-deploy`)

| Se o requisito/código envolve... | Aplique heurística |
|----------------------------------|---------------------|
| Priorização de testes de regressão pós-deploy | **RCRCRC** (Recent, Core, Risky, Configuration-sensitive, Conformance, Complex) |
| Análise de risco em produção | **SFDPOT** (Sources, Formats, Dependencies, Pace, Environment, Other, Time) |

### Regras de Decisão

- Função de validação de input → Baica (prioridade alta)
- Serviço que chama repositório → Baica + CRUD ou Count
- Controller/endpoint → VADER + FAILURE (error handling)
- Fluxo completo usuário → FAILURE (Emotions, Recovery) + EMOTIONS + E2E
- Recursos compartilhados entre usuários → Multi-User + Count
- Cenários de alta demanda → FLOOD + Count
- Deploy recente → RCRCRC (priorizar regressão) + SFDPOT (análise de risco)

### 2. Seleção de Camadas

Distribua os testes pela pirâmide:

- **Unitários (70%)**: Lógica pura, validações, funções isoladas, cálculos
- **Integração (20%)**: Persistência, repositórios, transações, queries
- **Componentes (5%)**: UI isolada, interações, estados visuais
- **Serviço (3%)**: APIs HTTP completas, autenticação, contratos
- **E2E (2%)**: Fluxos críticos de negócio, jornadas completas

Ajuste conforme o contexto: uma função de validação terá ~100% unitários; um endpoint completo terá unitários + integração + serviço.

### 3. Geração de Casos de Teste

Para cada camada recomendada, liste casos com:

- **ID único**: UT-XXX-01, IT-XXX-01, API-XXX-01, COMP-XXX-01, E2E-XXX-01
- **Nome do cenário**: "Rejeitar amount = 0"
- **Input**: valor ou ação
- **Output esperado**: resultado ou assertiva
- **Heurística**: qual dimensão cobre
- **Prioridade**: P0 (crítico), P1 (importante), P2 (desejável)

### 4. Geração do Relatório

**Obrigatório**: Ao concluir a análise, crie o arquivo de relatório.

**Caminho**: `output/test/strategy/test-strategy-[nome-funcao]-[YYYYMMDD].md`

**Exemplo de nome**: `output/test/strategy/test-strategy-validateAmount-20260217.md`

Use o **Template do Relatório** (seção abaixo) e preencha com os dados da análise. O relatório permite que o desenvolvedor:
- Passe o arquivo como contexto ao chamar TEST_UNIT_GUIDE, TEST_INTEGRATION_GUIDE, etc.
- Tenha lista exata de casos a implementar
- Mantenha rastreabilidade (IDs dos casos)

### 5. Recomendação de Implementação

No relatório e na resposta ao usuário, indique:

1. Qual guide consultar primeiro (ex: TEST_UNIT_GUIDE.md)
2. Que o relatório deve ser passado junto: `@TEST_UNIT_GUIDE.md @output/test/strategy/test-strategy-xxx.md`
3. Quais casos implementar (ex: "UT-VAL-01 a UT-VAL-07")
4. Arquivos sugeridos (ex: `output/test/unit/validators/validateAmount.test.ts`)

## Template do Relatório

Ao gerar o relatório, use a estrutura abaixo. Substitua os placeholders pelos dados reais da análise.

```markdown
# Relatório de Estratégia de Testes: [Nome da Funcionalidade]

**Data**: YYYY-MM-DD
**Requisito**: REQ-XXX (ou N/A)
**Funcionalidade**: [Descrição breve em 1 linha]

## Análise de Heurísticas

### Heurísticas Identificadas

#### Desenvolvimento
- [ ] Baica (Boundaries, Nulls, Special Chars, Formats, Failure Patterns)
- [ ] VADER (Values, Authorization, Data Consistency, Error Handling, Rate Limiting)

#### Arquitetura de Código
- [ ] CRUD (Create, Read, Update, Delete)
- [ ] Count (0, 1, Muitos)

#### Design de Código
- [ ] Multi-User (Colisão, Sessão Dupla, Inventário Crítico, UX de Conflito)

#### QA & Testes
- [ ] FAILURE (Functional, Appropriate, Impact, Log, UI, Recovery, Emotions)
- [ ] EMOTIONS (9 emoções: Alegria, Tristeza, Raiva, Medo, Nojinho, Surpresa, Confiança, Antecipação, Desconfiança)
- [ ] FLOOD (picos, concorrência, entrada massiva, degradação, estabilidade)

#### Pós-Deploy
- [ ] RCRCRC (Recent, Core, Risky, Configuration-sensitive, Conformance, Complex)
- [ ] SFDPOT (Sources, Formats, Dependencies, Pace, Environment, Other, Time)

### Justificativa por Heurística
[Para cada heurística marcada, 1-2 frases explicando por que se aplica]

## Estratégia por Camada

### Distribuição Recomendada
- Unitários: N testes (X%)
- Integração: N testes (X%)
- Componentes: N testes (X%)
- Serviço: N testes (X%)
- E2E: N testes (X%)
- **Total**: N testes

### Justificativa da Distribuição
[Por que essa distribuição para este requisito]

## Casos de Teste Detalhados

### Testes Unitários (N)

#### Módulo: [nome]
**Heurística: [ex: Baica - Boundaries]**

1. **UT-XXX-01**: [Nome do cenário]
   - Input: [valor ou descrição]
   - Output esperado: [resultado]
   - Heurística: [dimensão]
   - Prioridade: P0

[Repetir para cada caso...]

### Testes de Integração (N)
[Idem, com prefixo IT-XXX-01...]

### Testes de Serviço (N)
[Idem, com prefixo API-XXX-01...]

### Testes de Componentes (N)
[Idem, com prefixo COMP-XXX-01...]

### Testes E2E (N)
[Idem, com prefixo E2E-XXX-01...]

## Próximos Passos para Implementação

### 1. Implementar Testes Unitários (N testes)
- **Consulte**: [TEST_UNIT_GUIDE.md](TEST_UNIT_GUIDE.md)
- **Passe este relatório** como contexto
- **Arquivos a criar**: [lista]

### 2. [Próxima camada...]
[...]

## Comandos para Executar Testes
\`\`\`bash
npm test -- --testPathPattern=unit
# etc.
\`\`\`

## Referências
- Skills de heurísticas: [links para /heuristic-guide-development, /heuristic-guide-qa-test, etc.]
- Guides: [links para TEST_UNIT_GUIDE, etc.]

---
**IMPORTANTE**: Use este relatório como contexto ao chamar os guides!

Exemplo: `@TEST_UNIT_GUIDE.md @output/test/strategy/test-strategy-[este-arquivo].md`
"Implemente os testes unitários listados no relatório"
```

## Matriz de Decisão: Requisito → Heurísticas → Camadas

| Tipo de artefato | Heurísticas típicas | Camadas típicas |
|------------------|---------------------|-----------------|
| Função de validação (ex: validateAmount) | Baica, Count | Unitários (100%) |
| Repositório (ex: UserRepository) | CRUD, Count | Integração (80%), Unitários (20% mocks) |
| Serviço de negócio (ex: TransferService) | Baica, VADER, FAILURE | Unitários (mocks), Integração (transações) |
| Controller/Endpoint (ex: POST /transactions) | VADER, FAILURE, Baica | Serviço (60%), Unitários (40% validações) |
| Fluxo completo (ex: login → transferir) | FAILURE, EMOTIONS | E2E (70%), Serviço (30%) |
| Recurso compartilhado (ex: estoque de ingressos) | Multi-User, Count, CRUD | Integração (50%), Serviço (30%), E2E (20%) |
| Sistema sob alta demanda (ex: venda de ingressos) | FLOOD, Count, FAILURE | Serviço (40%), Integração (40%), E2E (20%) |
| Feature recém-deployada | RCRCRC, SFDPOT | E2E (50%), Serviço (30%), Integração (20%) |

## Exemplo de Análise Completa

**Input do desenvolvedor**: "Tenho a função validateAmount que valida amount entre 1 e 1.000.000, e o requisito REQ-002 de transferência de QualiPoints. Que testes implementar?"

**Passos da análise**:

1. **Heurísticas**: Baica (boundaries, nulls), Count (0, 1, máximo)
2. **Camadas**: 100% unitários para validateAmount; se houver endpoint, adicionar testes de serviço
3. **Casos**:
   - UT-VAL-01: amount = 0 → valid false
   - UT-VAL-02: amount = -1 → valid false
   - UT-VAL-03: amount = 1 → valid true
   - UT-VAL-04: amount = 1_000_000 → valid true
   - UT-VAL-05: amount = 1_000_001 → valid false
   - UT-VAL-06: amount = null → valid false
   - UT-VAL-07: amount = undefined → valid false
4. **Relatório**: Gerar `output/test/strategy/test-strategy-validateAmount-20260217.md` com os 7 casos
5. **Recomendação**: "Consulte TEST_UNIT_GUIDE.md passando o relatório. Implemente UT-VAL-01 a UT-VAL-07 em output/test/unit/validators/validateAmount.test.ts"

## Referências para Implementação

Após gerar o relatório, o desenvolvedor deve usar os **guides** para implementar o código dos testes:

| Camada | Guide | Quando usar |
|--------|--------|-------------|
| Unitários | [TEST_UNIT_GUIDE.md](TEST_UNIT_GUIDE.md) | Validações, lógica pura, serviços com mocks |
| Integração | [TEST_INTEGRATION_GUIDE.md](TEST_INTEGRATION_GUIDE.md) | Repositórios, banco de dados, transações |
| Componentes | [TEST_COMPONENT_GUIDE.md](TEST_COMPONENT_GUIDE.md) | Componentes UI, formulários, interações |
| Serviço/API | [TEST_SERVICE_GUIDE.md](TEST_SERVICE_GUIDE.md) | Endpoints HTTP, autenticação, contratos |
| E2E | [TEST_E2E_GUIDE.md](TEST_E2E_GUIDE.md) | Fluxos completos, jornadas do usuário |

### Skills de Heurísticas

| Fase | Skill | Heurísticas |
|------|-------|-------------|
| Desenvolvimento | [`/heuristic-guide-development`](../../skills/heuristic-guide-development/) | Baica, VADER |
| Arquitetura | [`/heuristic-guide-architeture-code`](../../skills/heuristic-guide-architeture-code/) | Count, CRUD |
| Design | [`/heuristic-guide-desing-code`](../../skills/heuristic-guide-desing-code/) | Multi-User |
| QA & Testes | [`/heuristic-guide-qa-test`](../../skills/heuristic-guide-qa-test/) | EMOTIONS, FAILURE, FLOOD |
| Pós-Deploy | [`/heuristic-guide-pos-deploy`](../../skills/heuristic-guide-pos-deploy/) | RCRCRC, SFDPOT |

**Como usar o relatório com um guide**:
1. Salve o relatório em `output/test/strategy` (a skill deve tê-lo gerado)
2. Chame o guide com o relatório: `@TEST_UNIT_GUIDE.md @output/test/strategy/test-strategy-validateAmount-20260217.md`
3. Solicite: "Implemente os testes unitários listados no relatório (UT-VAL-01 a UT-VAL-07)"
4. O guide usará os casos do relatório para gerar o código exato

## Checklist da Análise

Ao realizar a análise, verifique:

- [ ] Todas as heurísticas aplicáveis foram identificadas (por fase do SDLC)?
- [ ] A distribuição por camada segue a pirâmide (mais unitários, menos E2E)?
- [ ] Cada caso tem ID único, input, output e prioridade?
- [ ] O relatório foi gerado e salvo em `output/test/strategy/`?
- [ ] O relatório segue o template (heurísticas por fase, casos, próximos passos)?
- [ ] A recomendação indica qual guide usar e como passar o relatório?
