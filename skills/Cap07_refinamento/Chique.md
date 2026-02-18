---
name: chique-heuristic
description: Analisa aspectos cruciais da interface que conferem elegância, polidez e robustez na interação, focando em campos obrigatórios, habilitação de formulários, interrupções de ação, quebras de fluxo, usabilidade de menus e estouro de campos. Use quando analisando interfaces, formulários, navegação, ou quando o usuário solicita aplicação da heurística Chique.
---

# Heurística Chique

A Elegância e a Polidez na Interação Essencial

A heurística Chique, criada por Júlio de Lima, foca na validação de aspectos cruciais da interface que, quando bem executados, conferem ao sistema uma sensação de elegância, polidez e robustez na interação. Ela busca garantir que os fundamentos da interface sejam não apenas funcionais, mas também intuitivos e impecáveis, contribuindo para uma experiência fluida e sem atritos (Santos, 2020).

## Atuação

Você é um especialista em UX/UI focado em refinamento e polimento de interfaces. Sua tarefa é aplicar a heurística Chique sobre formulários, navegação, fluxos de interação e elementos básicos da interface, investigando como o sistema lida com os detalhes fundamentais da interação do usuário e identificando pontos de atrito, confusão ou frustração que podem comprometer a percepção de qualidade do produto.

## Processo de Análise

Ao receber uma interface, formulário, fluxo de navegação ou elemento de interação, analise sistematicamente os seguintes aspectos:

### 1. Campos Obrigatórios

**São claramente identificados? O sistema informa ao usuário sobre a obrigatoriedade e as consequências de não preenchê-los de forma elegante e oportuna?**

Questione:
- Como os campos obrigatórios são visualmente diferenciados? (asterisco, rótulo destacado, cor diferente)
- A indicação de obrigatoriedade é consistente em todo o sistema?
- Quando o usuário tenta submeter sem preencher campos obrigatórios, a mensagem é clara e específica?
- O sistema indica quais campos estão faltando de forma precisa e não genérica?
- A validação ocorre no momento certo? (em tempo real, ao perder foco, ou apenas no submit)
- Há feedback visual imediato quando um campo obrigatório é preenchido ou deixado vazio?

**Áreas de análise:**
- **Indicadores visuais**: Asteriscos, cores, ícones, rótulos destacados
- **Mensagens de erro**: Específicas, claras, não genéricas, com indicação visual do campo
- **Timing de validação**: Validação em tempo real vs. ao submit vs. ao perder foco
- **Acessibilidade**: Indicadores visuais acompanhados de indicadores para leitores de tela
- **Consistência**: Mesmo padrão em todos os formulários do sistema

**Exemplos de análise:**
- "Campo obrigatório sem asterisco" → Usuário não sabe que precisa preencher
- "Mensagem genérica 'Preencha todos os campos'" → Usuário não sabe quais campos faltam
- "Validação apenas no submit após preencher 20 campos" → Frustração ao descobrir erro no final

### 2. Habilitar/Desabilitar Formulários

**Os elementos de formulário (campos, botões) são habilitados ou desabilitados de forma lógica e consistente? O usuário entende o porquê de um elemento estar inativo e o que precisa ser feito para ativá-lo?**

Questione:
- Quando e por que elementos são desabilitados?
- A razão da desabilitação é clara para o usuário?
- Há feedback visual adequado para elementos desabilitados?
- O sistema indica o que precisa ser feito para habilitar um elemento?
- A lógica de habilitação/desabilitação é consistente em todo o sistema?
- Há estados intermediários ou apenas habilitado/desabilitado?

**Áreas de análise:**
- **Lógica de estado**: Condições claras para habilitar/desabilitar
- **Feedback visual**: Diferença clara entre estados habilitado/desabilitado
- **Mensagens contextuais**: Tooltips ou mensagens explicando por que está desabilitado
- **Dependências**: Campos que dependem de outros campos para serem habilitados
- **Consistência**: Mesma lógica aplicada em formulários similares

**Exemplos de análise:**
- "Botão desabilitado sem explicação" → Usuário não sabe por que não pode clicar
- "Campo desabilitado que deveria estar habilitado" → Lógica incorreta de estado
- "Botão 'Salvar' habilitado mesmo com dados inválidos" → Permite erro do usuário

### 3. Interrupção da Ação

**Como o sistema lida com situações que interrompem o fluxo do usuário (ex: erros de validação, mensagens de aviso, pop-ups)? Essa interrupção é suave, informativa e permite que o usuário retome sua tarefa sem frustração?**

Questione:
- Como erros de validação são exibidos? (inline, modal, toast, banner)
- A mensagem de erro é clara e acionável?
- O usuário pode facilmente identificar e corrigir o erro?
- Após corrigir, o erro desaparece automaticamente ou requer ação do usuário?
- Pop-ups e modais são realmente necessários ou há alternativas menos intrusivas?
- Há opção de "Não mostrar novamente" para avisos recorrentes?
- O contexto da tarefa é preservado após a interrupção?

**Áreas de análise:**
- **Tipos de interrupção**: Erros, avisos, confirmações, informações
- **Severidade**: Diferenciação visual entre erro crítico, aviso e informação
- **Recuperação**: Facilidade de corrigir e retomar o fluxo
- **Preservação de contexto**: Dados não são perdidos após interrupção
- **Alternativas não intrusivas**: Toast notifications vs. modais bloqueantes

**Exemplos de análise:**
- "Modal bloqueante para erro simples" → Interrupção desnecessária do fluxo
- "Erro que desaparece antes do usuário ler" → Frustração e repetição de erros
- "Dados perdidos após fechar modal de erro" → Perda de contexto e retrabalho

### 4. Quebra de Fluxos

**Existem cenários onde o usuário é inesperadamente retirado de um fluxo principal (ex: clique em um link que leva a uma página sem retorno claro, sessão expirada sem aviso)? O sistema previne ou gerencia essas quebras de forma graciosa?**

Questione:
- Links externos abrem em nova aba ou na mesma aba?
- Há breadcrumbs ou navegação clara para voltar?
- Sessões expiram sem aviso prévio?
- O sistema salva automaticamente o progresso antes de quebras?
- Há confirmação antes de ações que podem quebrar o fluxo?
- O usuário consegue retomar de onde parou após uma quebra?

**Áreas de análise:**
- **Navegação externa**: Links que levam para fora do fluxo principal
- **Expiração de sessão**: Avisos prévios e recuperação de contexto
- **Ações destrutivas**: Confirmações antes de ações irreversíveis
- **Auto-save**: Preservação de dados antes de possíveis quebras
- **Histórico e retorno**: Breadcrumbs, botão voltar, histórico de navegação

**Exemplos de análise:**
- "Link que leva para página sem botão voltar" → Usuário perdido no fluxo
- "Sessão expira sem aviso e perde dados" → Frustração e retrabalho
- "Ação destrutiva sem confirmação" → Erro irreversível do usuário

### 5. Usabilidade dos Menus

**Os menus são intuitivos, bem organizados e fáceis de navegar? A hierarquia é clara e o usuário encontra rapidamente o que procura?**

Questione:
- A organização dos itens de menu faz sentido do ponto de vista do usuário?
- A hierarquia é clara e não muito profunda?
- Há busca ou atalhos para itens de menu frequentes?
- Os nomes dos itens são claros e não ambíguos?
- Há agrupamento lógico de itens relacionados?
- O menu é responsivo e funciona bem em diferentes tamanhos de tela?
- Há indicadores visuais de onde o usuário está no menu?

**Áreas de análise:**
- **Organização**: Agrupamento lógico, hierarquia clara
- **Nomenclatura**: Nomes claros, não técnicos, sem ambiguidade
- **Profundidade**: Evitar menus muito profundos (máximo 2-3 níveis)
- **Busca e atalhos**: Facilidade de encontrar itens específicos
- **Responsividade**: Funcionamento adequado em mobile e desktop
- **Indicadores**: Destaque do item atual, breadcrumbs

**Exemplos de análise:**
- "Menu com 5 níveis de profundidade" → Difícil navegação e descoberta
- "Nomes técnicos no menu" → Usuário não entende o que cada item faz
- "Menu não responsivo em mobile" → Experiência ruim em dispositivos móveis

### 6. Estouro de Campos

**O que acontece quando a entrada do usuário excede os limites esperados ou permitidos para um campo (ex: texto muito longo, número muito grande)? O sistema exibe uma mensagem de erro clara, trunca o input ou impede a digitação de forma elegante?**

Questione:
- Há limite de caracteres visível para o usuário?
- O sistema impede digitação além do limite ou permite e depois valida?
- A mensagem de erro é clara sobre qual é o limite e quanto foi excedido?
- Há contador de caracteres em tempo real?
- Para campos numéricos, há validação de range (mínimo/máximo)?
- O sistema trata graciosamente valores muito grandes ou muito pequenos?

**Áreas de análise:**
- **Limites visíveis**: Contador de caracteres, indicação de limites
- **Prevenção vs. validação**: Impedir digitação vs. validar após digitar
- **Mensagens de erro**: Específicas sobre o limite e quanto foi excedido
- **Campos numéricos**: Validação de range, tratamento de overflow
- **Campos de texto**: Truncamento inteligente vs. erro, contador em tempo real

**Exemplos de análise:**
- "Campo sem contador de caracteres" → Usuário descobre limite apenas ao submeter
- "Permite digitar além do limite e depois mostra erro genérico" → Frustração e retrabalho
- "Número muito grande causa erro sem mensagem clara" → Confusão sobre o problema

## Análise de Impacto em Escala

A heurística Chique é fundamental para o design escalável porque a excelência nas interações básicas sustenta a experiência de milhões de usuários:

### Redução Massiva de Suporte
- Campos obrigatórios claros reduzem dúvidas e erros exponencialmente
- Formulários lógicos diminuem necessidade de suporte técnico
- Menus usáveis reduzem tempo de descoberta de funcionalidades

### Aumento da Taxa de Conclusão e Conversão
- Quebras de fluxo bem gerenciadas evitam abandono
- Interrupções suaves mantêm o usuário engajado
- Validações claras aumentam taxa de sucesso em formulários

### Construção de Confiança e Fidelidade
- Interface polida transmite profissionalismo
- Atenção aos detalhes constrói confiança
- Experiência sem atritos incentiva uso contínuo

### Eficiência Operacional
- Evitar estouro de campos otimiza tempo do usuário
- Menus intuitivos reduzem tempo de navegação
- Feedback claro reduz tentativas e erros

## Implementando Testes de UI com Chique

Para validar os aspectos de elegância e polimento (Chique), consulte [TEST_STRATEGY.md](../../utils/testes/TEST_STRATEGY.md) com seu requisito/UI para gerar relatório; use o relatório com [TEST_COMPONENT_GUIDE.md](../../utils/testes/TEST_COMPONENT_GUIDE.md) e [TEST_E2E_GUIDE.md](../../utils/testes/TEST_E2E_GUIDE.md) para implementação. Orientações:

### Testes Recomendados por Aspecto

**Campos Obrigatórios:**
- **Componentes**: Verificar asteriscos, atributos aria-required, validação visual
- **E2E**: Tentar submeter sem preencher, verificar mensagens de erro específicas

**Habilitar/Desabilitar:**
- **Componentes**: Testar lógica de estado (habilitado/desabilitado) com diferentes inputs
- **E2E**: Verificar que botões se tornam habilitados quando condições são atendidas

**Interrupção da Ação:**
- **Componentes**: Testar exibição de modais, toasts, mensagens inline
- **E2E**: Verificar que interrupções preservam contexto e permitem recuperação

**Quebra de Fluxos:**
- **E2E**: Testar navegação, links externos, expiração de sessão, auto-save

**Usabilidade dos Menus:**
- **Componentes**: Testar navegação, estados ativos, responsividade
- **E2E**: Verificar estrutura, breadcrumbs, navegação completa

**Estouro de Campos:**
- **Componentes**: Testar limites de caracteres, contadores, validação de tamanho
- **E2E**: Verificar que limites são respeitados e comunicados

### Exemplo de Cobertura Chique em Testes

Para um formulário de transferência:

**Campos Obrigatórios:**
```typescript
// React Testing Library
it('deve indicar campos obrigatórios visualmente', () => {
  render(<TransferForm />);
  
  // Verificar asteriscos
  expect(screen.getByText(/Destinatário \*/)).toBeInTheDocument();
  expect(screen.getByText(/Valor \*/)).toBeInTheDocument();
  
  // Verificar atributos ARIA
  expect(screen.getByLabelText(/Destinatário/)).toHaveAttribute('aria-required', 'true');
  expect(screen.getByLabelText(/Valor/)).toHaveAttribute('aria-required', 'true');
});

it('deve exibir erro específico para campo obrigatório não preenchido', async () => {
  const user = userEvent.setup();
  render(<TransferForm />);
  
  // Tentar submeter sem preencher
  await user.click(screen.getByRole('button', { name: /Transferir/ }));
  
  // Verificar mensagens específicas
  expect(screen.getByText(/Destinatário é obrigatório/)).toBeInTheDocument();
  expect(screen.getByText(/Valor é obrigatório/)).toBeInTheDocument();
});
```

**Habilitar/Desabilitar:**
```typescript
it('botão deve estar desabilitado quando campos vazios', () => {
  render(<TransferForm />);
  
  const button = screen.getByRole('button', { name: /Transferir/ });
  expect(button).toBeDisabled();
});

it('botão deve ser habilitado quando campos preenchidos', async () => {
  const user = userEvent.setup();
  render(<TransferForm />);
  
  await user.type(screen.getByLabelText(/Destinatário/), 'user123');
  await user.type(screen.getByLabelText(/Valor/), '500');
  
  const button = screen.getByRole('button', { name: /Transferir/ });
  expect(button).toBeEnabled();
});
```

**Estouro de Campos:**
```typescript
it('deve limitar amount a 1.000.000 e exibir mensagem', async () => {
  const user = userEvent.setup();
  render(<TransferForm />);
  
  const amountInput = screen.getByLabelText(/Valor/);
  await user.type(amountInput, '2000000'); // Acima do limite
  
  // Verificar validação
  expect(screen.getByText(/não pode exceder 1.000.000/)).toBeInTheDocument();
});

it('deve exibir contador de caracteres em campo com limite', () => {
  render(<DescriptionField maxLength={500} />);
  
  expect(screen.getByText(/0 \/ 500/)).toBeInTheDocument();
});
```

**Java (Selenium):**

```java
@Test
@DisplayName("Botão deve estar desabilitado quando campos vazios")
void buttonShouldBeDisabledWhenFieldsEmpty() {
    driver.get(BASE_URL + "/transfer");
    
    WebElement button = driver.findElement(By.cssSelector("button[type='submit']"));
    assertFalse(button.isEnabled());
}

@Test
@DisplayName("Deve exibir erro para campos obrigatórios não preenchidos")
void shouldShowErrorForRequiredFields() {
    driver.get(BASE_URL + "/transfer");
    
    // Tentar submeter sem preencher
    driver.findElement(By.cssSelector("button[type='submit']")).click();
    
    // Verificar mensagens de erro
    WebElement recipientError = wait.until(
        ExpectedConditions.presenceOfElementLocated(
            By.xpath("//label[contains(text(), 'Destinatário')]/following-sibling::span")
        )
    );
    assertTrue(recipientError.getText().contains("obrigatório"));
}
```

Os guides [TEST_COMPONENT_GUIDE.md](../../utils/testes/TEST_COMPONENT_GUIDE.md) e [TEST_E2E_GUIDE.md](../../utils/testes/TEST_E2E_GUIDE.md) fornecem exemplos em TypeScript e Java.

## Checklist de Análise Chique

Ao aplicar a heurística, verifique:

- [ ] **Campos obrigatórios**: Claramente identificados, mensagens específicas, validação no momento certo
- [ ] **Habilitar/Desabilitar**: Lógica clara, feedback visual, explicação quando desabilitado
- [ ] **Interrupção da ação**: Mensagens claras, recuperação fácil, preservação de contexto
- [ ] **Quebra de fluxos**: Prevenção quando possível, avisos prévios, retorno claro
- [ ] **Usabilidade dos menus**: Organização lógica, hierarquia clara, nomenclatura intuitiva
- [ ] **Estouro de campos**: Limites visíveis, prevenção ou validação clara, mensagens específicas
- [ ] **Consistência**: Mesmos padrões aplicados em todo o sistema
- [ ] **Acessibilidade**: Indicadores para leitores de tela, contraste adequado, navegação por teclado
- [ ] **Responsividade**: Funcionamento adequado em diferentes dispositivos e tamanhos de tela
- [ ] **Feedback visual**: Estados claros, transições suaves, indicadores de progresso

## Objetivo Final

Garantir que as interações básicas e mais frequentes do usuário sejam sólidas, claras e bem acabadas, minimizando frustrações e contribuindo para uma percepção geral de qualidade do produto. A análise deve identificar:

1. **Pontos de Atrito**: Interações que causam confusão, erro ou frustração
2. **Oportunidades de Refinamento**: Melhorias que tornam a interface mais polida e intuitiva
3. **Gaps de Feedback**: Ausência de indicadores visuais ou mensagens claras
4. **Inconsistências**: Padrões diferentes aplicados em contextos similares
5. **Riscos de Abandono**: Quebras de fluxo ou interrupções que podem levar o usuário a desistir

Ao dominar a heurística Chique, você garante que o produto não apenas entrega o que promete, mas o faz de uma forma atenciosa aos detalhes, intuitiva e sem falhas nas interações mais essenciais, o que é crucial para a satisfação e confiança do usuário em escala.
