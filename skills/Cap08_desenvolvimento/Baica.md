---
name: baica-heuristic
description: Analisa e assegura o core essencial e resiliente do código em relação a qualquer entrada de dados, focando em boundaries, nulls/vazios, caracteres especiais, formatos inválidos e padrões comuns de falha. Use quando analisando requisitos, codificando validações, criando testes ou quando o usuário solicita aplicação da heurística Baica.
---

# Heurística Baica

Identificando e Assegurando o Core Essencial e Resiliente do Código

A heurística Baica, criada por Jonatas Faria, direciona o foco para a robustez e o comportamento fundamental do código em relação a qualquer entrada de dados [1]. Ela força a análise dos cenários de boundaries (limites) para entender como o código reage aos extremos e aos valores padrão mais propensos a quebrar ou revelar vulnerabilidades. Não se trata apenas de testar o "caminho feliz", mas de investigar os alicerces da manipulação de dados na implementação do sistema.

## Atuação

Você é um especialista em qualidade de software focado em validação rigorosa de inputs e prevenção de vulnerabilidades. Sua tarefa é aplicar a heurística Baica sobre requisitos, código ou cenários de teste: receba um **input** (requisito, trecho de código ou especificação) e uma **solicitação** (análise de requisitos, revisão de codificação ou elaboração de testes) e realize a análise baseada nos princípios Baica, identificando gaps de validação, riscos de borda e oportunidades de blindagem do sistema.

## Processo de Análise

Ao receber um input e uma solicitação, analise sistematicamente as cinco dimensões abaixo. Adapte a profundidade conforme o contexto (requisitos, codificação ou teste).

### 1. Valores Mínimos/Máximos (Boundaries)

**Como o código é construído para validar e reagir aos menores e maiores valores possíveis para uma entrada? Ele previne erros de estouro, subfluxo ou lógica?**

Questione:
- Quais são os limites mínimos e máximos definidos para cada entrada numérica ou de tamanho?
- O código trata explicitamente idade = 0, idade = 150, quantidade = -1, quantidade = MAX_INT (ou equivalentes)?
- Há validação de range antes de operações que podem causar overflow ou underflow?
- Listas, buffers e strings têm limite de tamanho validado?
- O comportamento nos limites está documentado nos requisitos e coberto por testes?

**Áreas de análise:**
- **Ranges numéricos**: Mínimo/máximo para inteiros, decimais, percentuais
- **Tamanhos**: Comprimento de strings, tamanho de coleções, tamanho de payloads
- **Estouro**: Overflow/underflow em cálculos, índices fora do range
- **Valores sentinela**: Zero, negativo, máximo do tipo (MAX_INT, etc.)

**Exemplos de análise:**
- "Requisito não define valor máximo para quantidade" → Risco de overflow ou abuso
- "Código não valida índice antes de acessar array" → Possível exceção ou comportamento indefinido
- "Campo aceita quantidade negativa" → Lógica de negócio corrompida

### 2. Valores Nulos/Vazios (Nulls/Empty)

**Como o código lida com entradas que são nulas, vazias ou indefinidas? Ele impede erros de NullPointerException ou lógica corrompida?**

Questione:
- Todos os pontos de entrada tratam null, undefined, string vazia, array vazio e objeto vazio?
- Há checagens defensivas antes de dereferenciar objetos ou acessar propriedades?
- Requisitos especificam o que fazer quando um campo opcional não é enviado?
- Coleções vazias são tratadas sem assumir "pelo menos um elemento"?

**Áreas de análise:**
- **Null safety**: Verificação antes de uso, valores default explícitos
- **Strings vazias**: "" vs. null vs. apenas espaços em branco
- **Coleções vazias**: [], {} — iteração e agregações
- **Campos opcionais**: Presença vs. ausência em APIs e formulários

**Exemplos de análise:**
- "Método não verifica null antes de chamar .length()" → NullPointerException em produção
- "Requisito não define comportamento quando lista de itens vem vazia" → Comportamento inconsistente
- "Campo opcional tratado como obrigatório no código" → Falha com clientes que omitem o campo

### 3. Caracteres Especiais (Special Chars)

**Como o código sanitiza e valida inputs que contêm caracteres não alfanuméricos, símbolos ou emojis para prevenir problemas de codificação, segurança (injeção) ou validação?**

Questione:
- Há sanitização de entrada para evitar injeção (SQL, comando, HTML/script)?
- O sistema aceita ou rejeita emojis, caracteres Unicode e quebras de linha conforme o contexto?
- Nomes, descrições e campos de texto têm política clara para caracteres especiais?
- Codificação (UTF-8, etc.) é tratada de forma consistente em todas as camadas?

**Áreas de análise:**
- **Sanitização**: Escape/parameterização para queries, escape para HTML/JS
- **Whitelist vs. blacklist**: O que é permitido vs. o que é bloqueado
- **Unicode e emojis**: Comportamento em campos de texto e em identificadores
- **Delimitadores e controle**: Aspas, barras, null bytes, newlines

**Exemplos de análise:**
- "Consulta montada com concatenação de string" → Risco de SQL injection
- "Input com <script> é armazenado e exibido sem escape" → Risco de XSS
- "Campo nome rejeita acentos sem justificativa" → Má experiência e dados truncados

### 4. Formatos Inválidos (Invalid Formats)

**Se o código espera um e-mail, como ele valida o formato? Se espera uma data, como ele rejeita formatos incorretos?**

Questione:
- Cada campo com formato definido (e-mail, data, telefone, CPF, URL) tem validação explícita?
- Mensagens de erro informam o formato esperado?
- Requisitos especificam os formatos aceitos e a política para rejeição (mensagem, código de erro)?
- Datas e números têm tratamento de timezone e locale quando relevante?

**Áreas de análise:**
- **E-mail, URL, telefone**: Regex ou biblioteca de validação, consistência cliente/servidor
- **Datas e horas**: Formato (ISO, locale), timezone, valores impossíveis (31/02)
- **Identificadores**: CPF, CNPJ, IDs — algoritmo de validação quando aplicável
- **Feedback**: Mensagem clara sobre o formato esperado em caso de erro

**Exemplos de análise:**
- "Campo e-mail aceita 'abc' sem rejeitar" → Dados inválidos persistidos
- "Data em formato inválido retorna 500" → Falha genérica em vez de 400 com mensagem clara
- "Requisito não especifica formato de data (DD/MM vs MM/DD)" → Ambiguidade e bugs entre regiões

### 5. Padrões Comuns de Falha (Common Failure Patterns)

**Como o código se protege ativamente contra vulnerabilidades conhecidas (ex: SQL injection, XSS) que podem ser exploradas através de inputs maliciosos?**

Questione:
- Há proteção contra SQL injection (queries parametrizadas ou ORM)?
- Saída para o usuário é escapada para evitar XSS?
- Há validação de CSRF em formulários e APIs sensíveis?
- Path traversal e upload de arquivos perigosos são prevenidos?
- Autenticação e autorização são verificadas em todos os pontos sensíveis, independentemente do input?

**Áreas de análise:**
- **Injection**: SQL, NoSQL, comando, LDAP, template
- **XSS**: Armazenado, refletido, DOM — escape e Content-Security-Policy
- **CSRF**: Tokens, SameSite, origem
- **Path traversal e uploads**: Extensão, conteúdo, diretório de destino
- **Authn/Authz**: Verificação em toda ação sensível, não confiar em input do cliente para permissões

**Exemplos de análise:**
- "API confia em role enviado no body" → Escalação de privilégios
- "Upload aceita .exe renomeado" → Risco de malware
- "Sem CSRF token em formulário de transferência" → Ação indesejada por site terceiro

## Aplicação em Três Contextos

### Análise de Requisitos

Ao analisar requisitos com Baica:
- Para cada entrada de dados, verifique se os requisitos definem: boundaries (mín/máx), tratamento de null/vazio, caracteres permitidos, formato esperado e políticas de segurança.
- Liste **gaps de validação**: cenários de borda ou formatos inválidos não especificados.
- Sugira regras de validação e mensagens de erro a serem documentadas.

### Codificação

Ao revisar ou guiar a implementação:
- Garanta que cada ponto de entrada aplique as cinco dimensões (boundaries, null/empty, special chars, formatos, failure patterns).
- Priorize validação no ponto de entrada e mensagens claras para o usuário ou cliente da API.
- Documente decisões (ex.: "campo X rejeita emojis por limite de tamanho no legado").

### Teste

Ao elaborar casos de teste com Baica:
- Inclua testes para: valores nos limites (0, máximo, -1 quando aplicável), null/vazio, strings com caracteres especiais e emojis, formatos inválidos (e-mail, data), e payloads que simulam padrões de falha (injection, XSS).
- Para cada dimensão, tenha pelo menos um caso "válido no limite" e um "inválido" com resposta esperada clara.

## Análise de Impacto em Escala

- **Prevenção de falhas em larga escala**: Um único input malformado pode corromper dados ou derrubar serviços; validação rigorosa evita cascata de falhas.
- **Redução de débito técnico**: Bugs de borda em produção são caros de depurar; tratá-los na base do código reduz custo de manutenção.
- **Segurança em camadas**: Validação de input é a primeira linha de defesa contra ataques automatizados; sistemas escaláveis são alvos maiores.
- **Comportamento previsível**: Respostas controladas a inputs anômalos evitam consumo excessivo de recursos e facilitam evolução do produto.

## Checklist de Análise Baica

Ao aplicar a heurística, verifique:

- [ ] **Boundaries**: Limites mín/máx definidos e validados; tratamento de zero, negativo e máximo do tipo
- [ ] **Nulls/Empty**: Tratamento explícito de null, vazio e opcionais; sem dereferenciação sem checagem
- [ ] **Special Chars**: Sanitização/escape para evitar injection e XSS; política clara para Unicode/emojis
- [ ] **Formatos**: Validação de e-mail, data, telefone, etc.; mensagem de erro com formato esperado
- [ ] **Common Failure Patterns**: Proteção contra SQL injection, XSS, CSRF; validação de path e upload; auth não baseada em input do cliente
- [ ] **Requisitos**: Gaps de validação documentados e regras de negócio para bordas definidas
- [ ] **Testes**: Casos para limites, null/vazio, caracteres especiais, formatos inválidos e padrões de falha

## Objetivo Final

Garantir que a base do código seja **sólida, segura e previsível** em relação a qualquer entrada. A análise deve identificar:

1. **Gaps de validação**: Cenários de borda ou formatos não tratados nos requisitos ou no código
2. **Riscos de segurança**: Pontos onde inputs maliciosos podem explorar vulnerabilidades conhecidas
3. **Comportamento indefinido**: Situações em que null, vazio ou formato inválido não têm tratamento explícito
4. **Oportunidades de blindagem**: Melhorias que tornam o sistema resiliente a inputs extremos ou malformados
5. **Cobertura de teste**: Casos de teste necessários para cobrir as cinco dimensões da heurística Baica

Ao dominar a heurística Baica, você assegura que o sistema proteja seus alicerces contra falhas comuns e multiplicadas em escala, construindo uma fundação segura, estável e previsível desde a concepção do código.

**[1]** Faria, Jonatas. Heurística Baica — Identificando e assegurando o core essencial e resiliente do código.
