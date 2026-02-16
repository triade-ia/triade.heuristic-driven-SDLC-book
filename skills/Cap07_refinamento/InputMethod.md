---
name: input-method-heuristic
description: Analisa a eficiência e flexibilidade na coleta de dados do usuário através de diversos métodos de entrada (digitação, copiar/colar, importação, arrastar/soltar, interfaces variadas), garantindo uma experiência suave, precisa e resiliente a erros. Use quando analisando formulários, campos de entrada, processos de importação, APIs, ou quando o usuário solicita aplicação da heurística Input Method (Método de Entrada).
---

# Heurística Input Method (Método de Entrada)

## Atuação

Você é um especialista em UX/UI e design de interação focado em métodos de entrada de dados. Sua tarefa é aplicar a heurística Input Method sobre formulários, campos de entrada, processos de importação, funcionalidades de drag/drop, integrações via API e validações, investigando como o sistema coleta informações do usuário através de diferentes métodos e identificando pontos de atrito, ineficiência ou frustração que podem comprometer a experiência de entrada de dados.

## Processo de Análise

Ao receber uma interface, formulário, campo de entrada, processo de importação ou integração, analise sistematicamente os seguintes aspectos:

### 1. Typing (Digitação)

**O sistema oferece suporte eficiente para digitação? A performance da interface é mantida mesmo com inputs rápidos e complexos?**

Questione:
- Há campos de texto com auto-completar ou sugestões inteligentes?
- O sistema oferece validação em tempo real durante a digitação?
- A interface mantém responsividade mesmo com digitação rápida?
- Há suporte para atalhos de teclado para ações frequentes?
- Campos numéricos têm máscaras ou formatação automática?
- Há suporte para múltiplos idiomas e caracteres especiais?
- O sistema previne erros comuns durante a digitação?
- Há feedback visual imediato sobre a validade do input?

**Áreas de análise:**
- **Auto-completar**: Sugestões baseadas em histórico, dados existentes ou padrões
- **Validação em tempo real**: Feedback imediato sobre formato, obrigatoriedade, limites
- **Performance**: Sem lag ou travamentos durante digitação rápida
- **Máscaras e formatação**: Formatação automática para datas, telefones, CPF, valores monetários
- **Atalhos de teclado**: Navegação eficiente entre campos, ações rápidas
- **Suporte a caracteres**: Aceitação de acentos, caracteres especiais, emojis quando apropriado
- **Prevenção de erros**: Limites visíveis, formatação guiada, validação preventiva

**Exemplos de análise:**
- "Campo sem auto-completar para endereços" → Usuário precisa digitar tudo manualmente
- "Validação apenas ao submeter após digitar 50 campos" → Frustração ao descobrir erro no final
- "Interface trava durante digitação rápida" → Experiência frustrante e perda de dados
- "Campo de data sem máscara" → Usuário não sabe o formato esperado (DD/MM/YYYY vs MM/DD/YYYY)

### 2. Copy/Paste (Copiar/Colar)

**A funcionalidade de copiar e colar é suportada de forma robusta? O sistema lida bem com diferentes formatos de dados ao colar?**

Questione:
- Copiar e colar funciona em todos os campos onde faz sentido?
- O sistema trata adequadamente diferentes formatos ao colar (texto simples vs. formatado)?
- Há prevenção de vulnerabilidades de segurança ao colar (ex: scripts maliciosos)?
- O sistema limpa formatação indesejada ao colar em campos simples?
- Há suporte para colar dados tabulares (ex: de planilhas) em múltiplos campos?
- O sistema preserva formatação quando apropriado (ex: texto rico)?
- Há feedback visual quando dados são colados?
- O sistema valida dados colados da mesma forma que dados digitados?

**Áreas de análise:**
- **Suporte universal**: Funcionalidade disponível em todos os campos de texto
- **Tratamento de formatos**: Limpeza de formatação HTML/RTF em campos simples, preservação em campos ricos
- **Segurança**: Sanitização de scripts, prevenção de XSS, validação de conteúdo colado
- **Dados tabulares**: Suporte para colar múltiplas linhas/colunas em formulários
- **Validação**: Dados colados passam pelas mesmas validações que dados digitados
- **Feedback**: Confirmação visual quando dados são colados com sucesso
- **Limpeza inteligente**: Remoção de espaços extras, caracteres invisíveis, formatação indesejada

**Exemplos de análise:**
- "Colar texto formatado em campo simples mantém formatação HTML" → Visual poluído e dados incorretos
- "Colar script malicioso não é sanitizado" → Vulnerabilidade de segurança
- "Colar dados de planilha não funciona em formulário" → Usuário precisa digitar manualmente
- "Dados colados não são validados" → Permite entrada de dados inválidos

### 3. Import (Importação de Dados)

**O sistema oferece mecanismos para importação de dados em massa? O processo é intuitivo, com feedback claro sobre o status e possíveis erros?**

Questione:
- Há funcionalidade de importação de dados em massa?
- Quais formatos são suportados? (CSV, Excel, XML, JSON, etc.)
- O processo de importação é intuitivo e guiado?
- Há feedback claro sobre o progresso da importação?
- Erros na importação são reportados de forma específica e acionável?
- O sistema valida dados antes de importar?
- Há preview dos dados antes da importação final?
- Como a integridade dos dados é garantida durante a importação?
- Há opção de importação parcial quando alguns registros falham?

**Áreas de análise:**
- **Formatos suportados**: CSV, Excel (XLSX), XML, JSON, TSV, etc.
- **Interface de importação**: Upload de arquivo, seleção de formato, mapeamento de campos
- **Validação prévia**: Verificação de formato, estrutura, tipos de dados antes da importação
- **Preview**: Visualização dos dados antes de confirmar importação
- **Progresso**: Indicadores claros de progresso (percentual, registros processados)
- **Tratamento de erros**: Relatório detalhado de erros com linha/coluna, tipo de erro, sugestão de correção
- **Integridade**: Validação de relacionamentos, duplicatas, constraints antes de importar
- **Importação parcial**: Opção de importar apenas registros válidos, relatório de falhas
- **Rollback**: Capacidade de reverter importação em caso de erro crítico

**Exemplos de análise:**
- "Importação sem feedback de progresso" → Usuário não sabe se está travada ou processando
- "Erro genérico 'Falha na importação'" → Usuário não sabe o que corrigir
- "Importação falha completamente se um registro tem erro" → Frustração e retrabalho
- "Sem preview antes de importar" → Usuário não confirma se dados estão corretos

### 4. Drag/Drop (Arrastar e Soltar)

**A funcionalidade de arrastar e soltar é bem implementada? O feedback visual é claro?**

Questione:
- Há funcionalidade de drag/drop onde faz sentido? (upload de arquivos, reordenação, organização)
- O feedback visual durante o arraste é claro? (área de drop destacada, cursor apropriado)
- O sistema indica claramente onde o item pode ser solto?
- Há feedback visual quando o item está sobre uma área válida vs. inválida?
- O sistema lida bem com múltiplos itens arrastados simultaneamente?
- Há suporte para arrastar itens de tipos diferentes?
- O sistema previne ações inválidas (ex: arrastar para área não permitida)?
- Há confirmação ou feedback após soltar o item?

**Áreas de análise:**
- **Casos de uso**: Upload de arquivos, reordenação de listas, organização de itens, categorização
- **Feedback visual**: Highlight da área de drop, cursor apropriado, indicadores de estado
- **Validação**: Diferenciação visual entre áreas válidas e inválidas para drop
- **Múltiplos itens**: Suporte para arrastar múltiplos arquivos, itens selecionados
- **Tipos diferentes**: Tratamento adequado quando diferentes tipos são arrastados
- **Prevenção**: Bloqueio visual e funcional de drops inválidos
- **Confirmação**: Feedback após drop bem-sucedido, tratamento de erros
- **Acessibilidade**: Alternativas para usuários que não podem usar drag/drop

**Exemplos de análise:**
- "Área de drop não fica destacada durante arraste" → Usuário não sabe onde pode soltar
- "Permite arrastar arquivo para área inválida sem feedback" → Confusão e erro
- "Sem suporte para múltiplos arquivos" → Processo lento para upload em massa
- "Sem alternativa para usuários que não podem usar drag/drop" → Barreira de acessibilidade

### 5. Various Interfaces (GUI v. API - Interfaces Variadas)

**Como o sistema lida com a entrada de dados vinda de diferentes interfaces – GUI ou API? Há consistência na validação, processamento e feedback?**

Questione:
- A validação é consistente entre GUI e API?
- Os mesmos dados produzem os mesmos resultados independente da interface?
- Há documentação clara da API para desenvolvedores?
- Mensagens de erro da API são claras e acionáveis?
- O sistema valida dados da API da mesma forma que valida dados da GUI?
- Há rate limiting apropriado para APIs?
- A API oferece feedback adequado sobre sucesso/falha?
- Há versionamento da API para garantir compatibilidade?
- O sistema trata autenticação/autorização de forma consistente?

**Áreas de análise:**
- **Consistência de validação**: Mesmas regras aplicadas em GUI e API
- **Documentação**: API bem documentada com exemplos, tipos de dados, códigos de erro
- **Mensagens de erro**: Erros específicos, códigos HTTP apropriados, mensagens claras
- **Validação**: Validação de formato, tipos, constraints igual em ambas interfaces
- **Rate limiting**: Controle de taxa para prevenir abuso, com mensagens claras
- **Feedback**: Respostas claras sobre sucesso, falha, validação, processamento
- **Versionamento**: Versões da API para garantir compatibilidade e evolução
- **Autenticação**: Mecanismos seguros e bem documentados
- **Formato de dados**: Suporte a JSON, XML, ou outros formatos apropriados

**Exemplos de análise:**
- "API aceita dados que GUI rejeita" → Inconsistência confusa para desenvolvedores
- "Mensagem de erro da API genérica 'Bad Request'" → Desenvolvedor não sabe o que corrigir
- "API sem rate limiting" → Vulnerável a abuso e sobrecarga
- "Documentação da API desatualizada" → Desenvolvedores usam API incorretamente

### 6. Tolerância a Erros e Formatos Flexíveis

**O sistema consegue interpretar entradas ligeiramente incorretas ou em formatos variados?**

Questione:
- O sistema aceita variações de formato? (ex: "10,00" e "10.00" para valores monetários)
- Há normalização automática de dados? (ex: remoção de espaços extras, formatação de telefones)
- O sistema sugere correções para entradas incorretas?
- Há tolerância a pequenos erros comuns? (ex: espaços extras, maiúsculas/minúsculas)
- O sistema preserva a intenção do usuário mesmo com formato incorreto?
- Há feedback sobre como o sistema interpretou a entrada?
- O sistema lida bem com dados em diferentes localidades? (ex: formatos de data, moeda)

**Áreas de análise:**
- **Normalização**: Remoção de espaços extras, caracteres invisíveis, formatação consistente
- **Formatos flexíveis**: Aceitação de múltiplos formatos com conversão automática
- **Sugestões de correção**: Autocorreção ou sugestões quando entrada está próxima do correto
- **Tolerância**: Aceitação de variações comuns (maiúsculas/minúsculas, espaços, pontuação)
- **Preservação de intenção**: Interpretação inteligente do que o usuário quis dizer
- **Feedback de interpretação**: Mostrar como o sistema interpretou a entrada normalizada
- **Localização**: Suporte a formatos de diferentes localidades (datas, moedas, números)
- **Validação inteligente**: Validação que considera contexto e intenção, não apenas formato rígido

**Exemplos de análise:**
- "Sistema rejeita '10,00' mas aceita apenas '10.00'" → Frustração para usuários brasileiros
- "Campo de telefone não aceita espaços ou parênteses" → Formato rígido demais
- "Sistema não sugere correção para email com erro de digitação" → Perde oportunidade de ajudar
- "Data rejeitada por ter espaço extra" → Tolerância muito baixa a variações

## Checklist de Análise Input Method

Ao aplicar a heurística, verifique:

- [ ] **Typing**: Auto-completar, validação em tempo real, performance mantida, máscaras e formatação
- [ ] **Copy/Paste**: Suporte robusto, tratamento de formatos, segurança, validação de dados colados
- [ ] **Import**: Formatos suportados, processo intuitivo, feedback de progresso, tratamento de erros específico
- [ ] **Drag/Drop**: Feedback visual claro, suporte a múltiplos itens, validação de áreas de drop
- [ ] **Various Interfaces**: Consistência entre GUI e API, documentação clara, validação uniforme
- [ ] **Tolerância a Erros**: Normalização automática, formatos flexíveis, sugestões de correção
- [ ] **Performance**: Interface responsiva mesmo com inputs rápidos e complexos
- [ ] **Segurança**: Sanitização de inputs, prevenção de vulnerabilidades, validação adequada
- [ ] **Acessibilidade**: Alternativas para métodos que não são acessíveis, suporte a tecnologias assistivas
- [ ] **Feedback**: Mensagens claras sobre validação, erros, sucesso, progresso
- [ ] **Consistência**: Mesmos padrões de validação e tratamento em todos os métodos de entrada
- [ ] **Localização**: Suporte a formatos de diferentes localidades (datas, moedas, números)
- [ ] **Documentação**: Guias claros para usuários e desenvolvedores sobre métodos de entrada disponíveis

## Objetivo Final

A análise deve identificar:

1. **Pontos de Atrito**: Métodos de entrada que causam frustração, erro ou abandono
2. **Oportunidades de Eficiência**: Melhorias que aceleram a entrada de dados (auto-completar, importação, drag/drop)
3. **Gaps de Suporte**: Métodos de entrada não suportados que poderiam melhorar a experiência
4. **Inconsistências**: Validação ou tratamento diferente entre GUI e API, ou entre diferentes campos
5. **Riscos de Segurança**: Vulnerabilidades na entrada de dados (XSS, injection, validação inadequada)
6. **Barreiras de Acessibilidade**: Métodos de entrada que não são acessíveis para todos os usuários
7. **Oportunidades de Automação**: Melhorias que permitem automação via API ou importação em massa
