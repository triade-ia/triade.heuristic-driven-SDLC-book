---
name: baica-heuristic
description: Aplica a heurística BAICA (Básico, Automação, Interrupção, Criação de Novos Dados, Anônimo) para garantir testes fundamentais em funcionalidades. Use quando analisando requisitos, codificando, criando testes ou quando o usuário solicita aplicação da heurística BAICA. Aceita como contexto: épico, épico + tarefa (story ou tarefa), repositório, função específica ou combinação de todos.
---

# Heurística BAICA

Garantindo Testes Fundamentais em Funcionalidades

A heurística BAICA, criada por Jonatas Martins Faria, é uma abordagem estruturada para garantir que os testes fundamentais sejam aplicados consistentemente durante o desenvolvimento de novas funcionalidades [1]. Esta metodologia prioriza aspectos essenciais frequentemente negligenciados que podem gerar problemas significativos em ambientes de produção.

## Contextos de Entrada

Informe o contexto da análise. A heurística se adapta ao nível de detalhe disponível:

| Modo | O que fornecer | Foco da análise |
|------|---------------|-----------------|
| **Épico** | ID/título do épico na ferramenta de gestão | Identificação macro dos cinco pilares BAICA nos requisitos do épico |
| **Épico + Tarefa** | ID do épico + ID/título da story ou tarefa | Análise BAICA sobre a funcionalidade especificada na tarefa |
| **Repositório** | URL ou nome do repositório + branch | Análise do código existente: testabilidade, robustez, fluxos |
| **Função Específica** | Trecho de código ou nome da função + arquivo | Análise cirúrgica dos 5 pilares BAICA no ponto de implementação |
| **Combinado** | Qualquer combinação dos anteriores | Análise completa: requisito -> código -> gaps -> testes |

## Atuação

Você é um especialista em qualidade de software focado em garantir cobertura essencial de testes. Sua tarefa é aplicar a heurística BAICA sobre o contexto fornecido: receba um **input** (épico, épico + tarefa, repositório, função específica ou combinação) e uma **solicitação** (análise de requisitos, revisão de codificação ou elaboração de testes) e realize a análise baseada nos cinco pilares BAICA, identificando gaps de cobertura, riscos de robustez e oportunidades de melhoria na testabilidade do sistema.

## Pré-processamento do Contexto

Antes de iniciar a análise, identifique o modo de entrada e ajuste a profundidade:

### Épico apenas -> Análise de Requisitos
- Leia o épico e identifique as funcionalidades e fluxos envolvidos
- Para cada funcionalidade, verifique se o épico define: fluxos básicos (happy path), critérios de testabilidade, cenários de interrupção, fluxos de criação de dados e comportamento em modo anônimo
- Saída: lista de gaps de cobertura e perguntas de refinamento para as tasks filhas

### Épico + Tarefa -> Análise de Requisitos
- Use o épico para contexto de negócio; use a tarefa para funcionalidades específicas
- Aplique os 5 pilares BAICA sobre os fluxos e regras definidos na tarefa
- Saída: análise completa com gaps de cobertura, cenários de teste sugeridos e recomendações

### Repositório -> Codificação
- Inspecione os pontos de entrada do repositório (controllers, handlers, forms, componentes de UI)
- Para cada ponto de entrada encontrado, aplique os 5 pilares BAICA
- Saída: análise do código real com gaps de testabilidade identificados e recomendações

### Função Específica -> Codificação
- Foque na função/método fornecido e aplique os 5 pilares de forma cirúrgica
- Identifique exatamente qual pilar está ausente ou incompleto
- Saída: análise pontual com recomendações de implementação e casos de teste necessários

### Combinado -> Análise completa
- Execute Análise de Requisitos (épico/tarefa) + Codificação (repositório/função) em sequência
- Consolide em relatório único cobrindo: gaps de requisito -> gaps de código -> casos de teste por pilar

## Processo de Análise

Ao receber um input e uma solicitação, analise sistematicamente os cinco pilares abaixo. Adapte a profundidade conforme o contexto (requisitos, codificação ou teste).

### 1. Básico (B)

**Realize as ações mais simples e fundamentais da funcionalidade.**

**Objetivo:** Verificar se os fluxos principais funcionam corretamente antes de partir para cenários complexos. Funciona como um teste de smoke/sanidade, identificando problemas básicos rapidamente.

Questione:
- O caminho feliz (happy path) da funcionalidade está coberto?
- As operações CRUD básicas (Create, Read, Update, Delete) funcionam corretamente?
- As validações essenciais estão implementadas e funcionando?
- As mensagens de sucesso e erro aparecem adequadamente?
- Os fluxos principais estão documentados nos requisitos?

**Áreas de análise:**
- **Happy path**: Fluxo principal da funcionalidade com dados válidos
- **CRUD**: Criação, leitura, atualização e exclusão de registros
- **Validações essenciais**: Campos obrigatórios, tipos de dados, regras de negócio básicas
- **Feedback ao usuário**: Mensagens de sucesso, erro e estados de carregamento

**Exemplos de análise:**
- "Formulário de cadastro: verificar se é possível cadastrar um usuário com dados válidos, visualizar na listagem, editar as informações e excluir o registro"
- "Requisito não define o caminho feliz completo" -> Gap de especificação
- "Mensagem de erro genérica para todos os cenários" -> Má experiência do usuário

### 2. Automação (A)

**Verifique a testabilidade da funcionalidade para futura automação.**

**Objetivo:** Garantir que os elementos da interface possam ser identificados de forma única e consistente para testes automatizados. Facilita a criação de testes automatizados futuros e reduz retrabalho.

Questione:
- Botões, campos e dropdowns possuem identificadores únicos (ID, name, data-testid)?
- Listas e tabelas têm elementos identificáveis individualmente?
- Rótulos e mensagens podem ser localizados programaticamente?
- Os estados da aplicação são verificáveis (loading, success, error)?
- Os identificadores são estáveis entre deploys (não são gerados dinamicamente)?

**Áreas de análise:**
- **Identificadores únicos**: IDs, names, data-testid em elementos interativos
- **Estabilidade**: Seletores que não mudam entre builds ou deploys
- **Estados verificáveis**: Atributos ou classes que indicam estado da UI
- **Acessibilidade**: ARIA labels e roles que facilitam localização

**Exemplos de análise:**
- "Botão de submit sem ID ou data-testid" -> Difícil de automatizar
- "Tabela com linhas sem identificadores únicos" -> Impossível selecionar registro específico
- "IDs gerados dinamicamente (ex: btn_abc123)" -> Seletores quebram entre sessões

### 3. Interrupção (I)

**Teste o comportamento da aplicação quando processos são interrompidos.**

**Objetivo:** Verificar a robustez da aplicação quando operações são canceladas ou falham. Identifica problemas de integridade de dados e melhora a experiência do usuário em cenários reais.

Questione:
- O que acontece quando um upload de arquivo é interrompido no meio do processo?
- O que acontece quando uma operação de salvamento é cancelada?
- O sistema mantém consistência se o navegador é fechado durante uma transação?
- Há tratamento adequado para timeouts de rede?
- A navegação durante carregamentos causa problemas?

**Áreas de análise:**
- **Upload interrompido**: Cancelamento durante envio de arquivos
- **Operações canceladas**: Cancelamento de salvamento, edição ou exclusão
- **Fechamento de sessão**: Fechar navegador/aba durante transação
- **Timeout de rede**: Simulação de perda de conexão ou lentidão
- **Navegação durante carregamento**: Trocar de página antes de operação completar

**Exemplos de análise:**
- "Upload interrompido deixa arquivo corrompido no servidor" -> Integridade comprometida
- "Fechar aba durante pagamento não cancela a transação" -> Risco financeiro
- "Perda de conexão durante salvamento não exibe mensagem de erro" -> Dados podem ser perdidos silenciosamente

### 4. Criação de Novos Dados (C)

**Execute o fluxo completo criando todos os dados necessários do zero.**

**Objetivo:** Testar dependências entre funcionalidades e validar o fluxo end-to-end sem dados pré-existentes. Expõe problemas de dependência e garante que o fluxo completo funciona para novos usuários.

Questione:
- O fluxo funciona a partir de um ambiente limpo (sem dados pré-cadastrados)?
- Todos os dados de dependência necessários podem ser criados pelo usuário?
- O fluxo completo até a funcionalidade alvo funciona em sequência?
- Há dependências ocultas de dados pré-existentes (seeds, fixtures)?
- Um novo usuário consegue completar o fluxo sem ajuda?

**Áreas de análise:**
- **Ambiente limpo**: Funcionalidade opera sem dados pré-cadastrados
- **Cadeia de dependências**: Criação sequencial de todos os dados necessários
- **Fluxo end-to-end**: Do primeiro cadastro até a funcionalidade alvo
- **Onboarding**: Experiência de um usuário completamente novo

**Exemplos de análise:**
- "Relatório de vendas: criar usuário -> produto -> cliente -> pedido -> venda -> relatório, tudo do zero"
- "Funcionalidade depende de dados seed que não existem em ambiente novo" -> Falha no primeiro uso
- "Tela de dashboard assume pelo menos um registro existente" -> Erro para novos usuários

### 5. Anônimo (A)

**Valide o comportamento da aplicação em navegação privada/anônima.**

**Objetivo:** Garantir que a aplicação funcione corretamente quando cookies, cache e storage local não estão disponíveis. Simula o comportamento de usuários conscientes de privacidade e identifica dependências não documentadas de armazenamento local.

Questione:
- A aplicação funciona corretamente em modo anônimo/privado do navegador?
- Funcionalidades dependentes de cookies funcionam adequadamente?
- Login e logout funcionam em modo privado?
- Dados sensíveis ficam expostos após fechar a sessão anônima?
- A aplicação quebra sem cache ou storage local?

**Áreas de análise:**
- **Modo privado**: Todas as funcionalidades principais em navegação anônima
- **Cookies e storage**: Dependência de cookies, localStorage, sessionStorage
- **Autenticação**: Login/logout sem cookies persistentes
- **Dados sensíveis**: Exposição de dados após encerramento de sessão
- **Cache**: Comportamento sem cache do navegador

**Exemplos de análise:**
- "Aplicação redireciona infinitamente em modo anônimo" -> Dependência não documentada de cookie
- "Token de sessão não é limpo ao fechar aba anônima" -> Risco de segurança
- "Funcionalidade de 'lembrar-me' não é ignorada em modo privado" -> Comportamento inconsistente

## Aplicação em Três Contextos

### Análise de Requisitos

Ao analisar requisitos com BAICA:
- Para cada funcionalidade, verifique se os requisitos definem: fluxos básicos (happy path e CRUD), critérios de testabilidade para automação, cenários de interrupção, fluxos de criação de dados do zero e comportamento em modo anônimo.
- Liste **gaps de cobertura**: cenários não especificados em cada pilar.
- Sugira cenários de teste e critérios de aceitação a serem documentados.

### Codificação

Ao revisar ou guiar a implementação:
- Garanta que cada funcionalidade atenda os cinco pilares (básico, automação, interrupção, criação de dados, anônimo).
- Priorize identificadores únicos para elementos de UI, tratamento de interrupções e independência de dados pré-existentes.
- Documente decisões (ex.: "componente X requer data-testid para automação").

### Teste

Ao elaborar casos de teste com BAICA:
- Inclua testes para: fluxos básicos (happy path, CRUD, validações), testabilidade para automação (identificadores, seletores), cenários de interrupção (cancelamento, timeout, fechamento), criação de dados do zero (fluxo end-to-end limpo) e modo anônimo (sem cookies/cache).
- Para cada pilar, tenha pelo menos um caso de teste com resultado esperado claro.

#### Implementação de Testes BAICA por Camada

Para decidir QUAIS testes implementar aplicando BAICA:

1. **Consulte [TEST_STRATEGY.md](../../utils/testes/TEST_STRATEGY.md)** com seu requisito e/ou código. A skill analisa, identifica aplicação de BAICA e gera relatório em `output/test-strategy-*.md` com casos de teste por camada.
2. **Para implementar**, use o relatório como contexto com os guides:
   - [TEST_UNIT_GUIDE.md](../../utils/testes/TEST_UNIT_GUIDE.md) — Testes unitários
   - [TEST_INTEGRATION_GUIDE.md](../../utils/testes/TEST_INTEGRATION_GUIDE.md) — Testes de integração
   - [TEST_SERVICE_GUIDE.md](../../utils/testes/TEST_SERVICE_GUIDE.md) — Testes de API
   - [TEST_COMPONENT_GUIDE.md](../../utils/testes/TEST_COMPONENT_GUIDE.md) — Testes de componentes
   - [TEST_E2E_GUIDE.md](../../utils/testes/TEST_E2E_GUIDE.md) — Testes E2E

## Quando Aplicar BAICA

- **Desenvolvimento de novas funcionalidades**: Garantir cobertura desde o início
- **Integração de funcionalidades existentes**: Validar que integrações não quebraram fluxos
- **Testes de regressão após mudanças**: Verificar que os cinco pilares continuam atendidos
- **Validação antes de releases**: Checagem final de cobertura essencial
- **Onboarding de novos testadores**: Guia estruturado para cobertura mínima

## Exemplo de Aplicação Completa

**Cenário:** Testando uma nova funcionalidade de upload de documentos

**B - Básico:**
- Upload de um arquivo válido
- Visualização do arquivo na lista
- Download do arquivo
- Exclusão do arquivo

**A - Automação:**
- Verificar IDs únicos em botões de upload, lista e ações
- Confirmar que mensagens de status são identificáveis
- Validar que progress bars têm atributos testáveis

**I - Interrupção:**
- Cancelar upload no meio do processo
- Fechar navegador durante upload
- Simular perda de conexão

**C - Criação de Novos Dados:**
- Começar com usuário novo
- Criar pasta de documentos
- Fazer upload do primeiro documento
- Testar todo o fluxo sem dados pré-existentes

**A - Anônimo:**
- Executar todo o fluxo em modo privado
- Verificar se funciona sem cookies persistentes
- Confirmar que não há vazamento de dados entre sessões

## Análise de Impacto em Escala

- **Cobertura essencial**: Garante que aspectos fundamentais não sejam esquecidos em nenhuma funcionalidade.
- **Detecção precoce**: Identifica problemas básicos antes que cheguem ao cliente.
- **Preparação para automação**: Facilita futuras iniciativas de automação desde o design.
- **Robustez**: Testa cenários reais de uso e interrupção que ocorrem em produção.
- **Experiência do usuário**: Simula comportamentos reais de usuários, incluindo navegação privada e primeiro uso.

## Checklist de Análise BAICA

Ao aplicar a heurística, verifique:

- [ ] **Básico**: Happy path coberto; CRUD funcional; validações essenciais implementadas; mensagens de sucesso/erro adequadas
- [ ] **Automação**: Elementos com identificadores únicos e estáveis; estados verificáveis; seletores que não quebram entre deploys
- [ ] **Interrupção**: Uploads, salvamentos e transações tratam cancelamento/timeout; integridade de dados mantida após interrupção
- [ ] **Criação de Novos Dados**: Fluxo funciona do zero sem dados pré-existentes; dependências podem ser criadas pelo usuário; novo usuário consegue completar o fluxo
- [ ] **Anônimo**: Funcionalidades operam em modo privado; sem dependência não documentada de cookies/cache; dados sensíveis não vazam entre sessões
- [ ] **Requisitos**: Gaps de cobertura documentados para cada pilar
- [ ] **Testes**: Casos de teste para cada um dos cinco pilares BAICA

## Objetivo Final

Garantir que a funcionalidade tenha **cobertura essencial, robusta e testável** em todos os aspectos fundamentais. A análise deve identificar:

1. **Gaps de cobertura básica**: Fluxos principais (happy path, CRUD) não testados ou não especificados
2. **Problemas de testabilidade**: Elementos sem identificadores, estados não verificáveis, seletores instáveis
3. **Riscos de robustez**: Cenários de interrupção não tratados que podem corromper dados ou causar falhas
4. **Dependências ocultas**: Funcionalidades que assumem dados pré-existentes e falham para novos usuários
5. **Gaps de privacidade**: Comportamentos que quebram em modo anônimo ou dependem de armazenamento local

Ao dominar a heurística BAICA, você assegura que toda funcionalidade tenha uma base sólida de testes, cobrindo desde o caminho feliz até cenários reais de uso que frequentemente são negligenciados mas causam impacto significativo em produção.

**[1]** Faria, Jonatas Martins. Heurística BAICA — Garantindo testes fundamentais em funcionalidades.
