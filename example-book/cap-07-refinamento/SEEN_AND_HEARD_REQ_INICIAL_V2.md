# Relatório: Seen and Heard (Visto e Ouvido) — REQ_INICIAL_V2 (QualiPoints)

**Requisito analisado:** Envio de QualiPoints entre usuários (REQ_INICIAL_V2).
**Skill aplicada:** Cap07_refinamento — Seen and Heard (Comunicação Efetiva do Sistema com o Usuário)
**Data:** 2026-02-18
**Status:** Análise de refinamento concluída

---

## 1. Pontos de Comunicação Sistema→Usuário Identificados no Requisito

Antes de aplicar os aspectos Seen and Heard, mapeamos todos os momentos em que o sistema precisa comunicar algo ao usuário:

| Momento de Comunicação | Canal | Definido no Req | Observação |
|------------------------|-------|-----------------|------------|
| **Estado "Processando"** | App mobile | Sim (seção 11.1) | Tipo de indicador não especificado |
| **Estado "Concluído"** | App mobile | Sim (seção 11.1) | Conteúdo e duração da confirmação não especificados |
| **Estado "Falhou"** | App mobile | Sim (seção 11.1) | Mecanismo de exibição não especificado |
| **Mensagens de erro por código** | App mobile | Sim (seção 11.2) | 8 mensagens mapeadas em linguagem de usuário |
| **Atualização do saldo** | App mobile | Sim (seção 11.1) | Após sucesso; timing não especificado |
| **Atualização da lista de recentes** | App / Painel web | Sim (seção 13) | Polling ou refresh manual aceitos no MVP |
| **Consistência App/Painel pós-envio** | Painel web | Sim (seção 13) | Sem WebSocket; polling aceitável |
| **Feedback de retry** | App mobile | Sim (seção 12) | Usuário informado sobre retry com mesma Idempotency-Key |
| **Campos obrigatórios** | App mobile | Implícito (seção 6.1) | Forma de indicação não especificada |
| **Acessibilidade (leitores de tela)** | App mobile / Painel web | Não definido | Ausência total de spec de acessibilidade |

---

## 2. Análise por Aspecto Seen and Heard

### 2.1 Feedback Visual

#### O que está definido

- Seção 11.1 define três estados visuais: **Processando**, **Concluído** e **Falhou**.
- No estado **Processando**: "exibir indicador de carregamento; desabilitar novo envio até resposta."
- No estado **Concluído**: "exibir confirmação de sucesso; atualizar saldo e lista de recentes."
- No estado **Falhou**: "exibir mensagem conforme código (ver mensagens de erro); permitir retry."
- Seção 11.2: 8 códigos de erro mapeados para mensagens em linguagem de usuário.

#### Lacunas identificadas

| Questão Seen and Heard | Análise |
|------------------------|---------|
| Qual indicador de carregamento é exibido no estado Processando? | **Lacuna:** o requisito menciona "indicador de carregamento" mas não especifica o tipo (spinner, barra de progresso, skeleton, shimmer) |
| A confirmação de sucesso é visualmente distinta do estado neutro? | **Lacuna:** "exibir confirmação de sucesso" não detalha componente (toast verde, banner, tela de resultado, checkmark animado) |
| Os erros são diferenciados visualmente por severidade? | **Lacuna:** erros de validação (400), autenticação (401) e timeout (408/504) recebem a mesma especificação de "mensagem conforme código", sem distinção visual de severidade |
| O saldo atualizado é destacado visualmente após sucesso? | **Lacuna:** seção 11.1 menciona "atualizar saldo", mas não define se há animação ou destaque transitório indicando a mudança (ex.: fade ou counter animado) |
| A nova transação na lista de recentes é destacada após o envio? | **Lacuna:** a transação recém-criada poderia ser destacada (ex.: fundo diferente por 3 segundos) para confirmar que o envio apareceu na lista |
| Há diferença visual entre o botão em estado normal, hover e loading? | **Lacuna:** estados do botão "Enviar" (normal, desabilitado, loading) não estão especificados visualmente |
| Há redundância visual (ícone + cor + texto) para mensagens de erro? | **Não especificado** — mensagens de erro definidas apenas como texto; ícones e cores não mencionados |

**Risco:** Um spinner genérico sem texto ("Processando transferência...") pode ser confundido com carregamento de outra parte da tela. Um toast de sucesso sem cor verde ou ícone de check pode passar despercebido, especialmente em ambientes com muito ruído visual.

**Recomendação:** Especificar que:
- Estado **Processando**: spinner circular + texto "Processando transferência..." no botão ou overlay da tela.
- Estado **Concluído**: toast verde com ícone de check + texto "Transferência realizada com sucesso!" (auto-dismiss em 4 segundos).
- Estado **Falhou** (validação 400/409/422): mensagem inline abaixo do campo relacionado + ícone de alerta vermelho.
- Estado **Falhou** (401/408/504): banner ou toast laranja/vermelho no topo da tela.
- Saldo atualizado: valor exibido com animação de contagem decrescente (ou destaque transitório em amarelo por 2 segundos).

---

### 2.2 Feedback Auditivo

#### O que está definido

- O requisito não menciona feedback auditivo em nenhuma seção.

#### Lacunas identificadas

| Questão Seen and Heard | Análise |
|------------------------|---------|
| Há som de confirmação para transferência bem-sucedida? | **Não especificado** — em apps financeiros (ex.: Nubank, PicPay), um som de confirmação reforça a conclusão da operação |
| Há alerta sonoro para erros críticos? | **Não especificado** |
| Usuários com deficiência visual dependem de feedback auditivo do leitor de tela? | **Lacuna crítica:** sem especificação de ARIA, os leitores de tela não têm como anunciar os estados de Processando, Concluído e Falhou |
| Há opção de desabilitar sons? | **Não especificado** |

**Risco de acessibilidade:** Usuários cegos que utilizam TalkBack (Android) ou VoiceOver (iOS) dependem de regiões ARIA live (`aria-live`, `role="alert"`, `role="status"`) para serem informados sobre mudanças de estado. A ausência de especificação de ARIA equivale a tornar a funcionalidade de transferência inacessível para este grupo.

**Recomendação:** Especificar que:
- Estado **Processando**: botão tem `aria-busy="true"` e `aria-label="Processando transferência, aguarde"`.
- Estado **Concluído**: região `role="status"` com `aria-live="polite"` anuncia "Transferência concluída com sucesso".
- Estado **Falhou**: região `role="alert"` com `aria-live="assertive"` anuncia a mensagem de erro.
- Som de confirmação de sucesso é opcional e controlável via configurações do dispositivo/App.

---

### 2.3 Feedback Tátil/Haptic (Mobile)

#### O que está definido

- O requisito define o App como canal mobile (seção 2), mas não menciona feedback tátil.

#### Lacunas identificadas

| Questão Seen and Heard | Análise |
|------------------------|---------|
| Há vibração de confirmação ao completar a transferência? | **Não especificado** |
| Há vibração de alerta para erros de saldo insuficiente? | **Não especificado** |
| O padrão de vibração diferencia sucesso de erro? | **Não especificado** |
| Há opção de desabilitar vibrações? | **Não especificado** |

**Contexto de relevância:** O feedback háptico é especialmente valioso em apps financeiros mobile — uma vibração distinta ao concluir uma transferência de QualiPoints reforça a percepção de que a ação foi concluída, complementando o feedback visual (principalmente em ambientes com brilho de tela reduzido ou quando o usuário não está olhando diretamente para o App).

**Recomendação para MVP:** Especificar feedback háptico básico:
- **Sucesso:** vibração curta e suave (padrão "light impact" do iOS / `VIRTUAL_KEY` no Android).
- **Erro:** vibração dupla curta (padrão "error" do iOS / `REJECT` no Android).
- Ambos respeitam as configurações de vibração do dispositivo.

---

### 2.4 Acessibilidade

#### O que está definido

- O requisito não faz nenhuma menção a acessibilidade, WCAG, ARIA, leitores de tela ou navegação por teclado.

#### Lacunas identificadas

| Questão Seen and Heard | Análise |
|------------------------|---------|
| O formulário de envio é navegável apenas por teclado? | **Lacuna:** sem spec, o campo `amount` pode não ter foco correto após tab do `recipientId` |
| Há suporte para leitores de tela (VoiceOver, TalkBack)? | **Lacuna crítica:** campos sem `aria-label` ou `aria-describedby` são inacessíveis para leitores de tela |
| Os elementos interativos têm labels descritivos para leitores de tela? | **Lacuna:** o requisito menciona "indicador de carregamento" sem definir que o botão deve ter `aria-busy` |
| O contraste de cores atende WCAG 2.1 AA (4.5:1 para texto normal)? | **Lacuna:** nenhum requisito de contraste especificado |
| Mensagens de erro são associadas aos campos por aria-describedby? | **Lacuna:** sem associação ARIA, leitores de tela não sabem que a mensagem de erro pertence ao campo |
| O formulário tem estrutura semântica adequada (label→input)? | **Não especificado** |
| Atalhos de teclado para ações frequentes estão definidos? | **Não especificado** |

**Risco de exclusão:** A ausência total de especificação de acessibilidade significa que o sistema pode ser lançado sem suporte a usuários com deficiências visuais, motoras ou cognitivas. Em muitos países, acessibilidade em produtos financeiros é obrigação legal (ex.: LGPD + normativas do Banco Central do Brasil, ADA nos EUA, EN 301 549 na Europa).

**Recomendação:** Especificar requisitos mínimos de acessibilidade para MVP:

| Requisito | Padrão |
|-----------|--------|
| Labels associados a campos | `<label for="amount">` ou `aria-label` |
| Campos obrigatórios | `aria-required="true"` |
| Mensagens de erro | `aria-describedby` associando campo à mensagem |
| Estado de loading do botão | `aria-busy="true"` durante Processando |
| Anúncio de sucesso/erro | `role="status"` (sucesso) e `role="alert"` (erro) |
| Contraste mínimo | 4.5:1 para texto (WCAG 2.1 AA) |
| Navegação por teclado | Tab order lógico: recipientId → amount → botão |

---

### 2.5 Status e Progresso

#### O que está definido

- Seção 11.1 define três estados do ponto de vista do usuário: **Processando**, **Concluído**, **Falhou**.
- Seção 5.2: API deve responder em até 10 segundos; após timeout retorna 408/504.
- Seção 5.2: O comportamento de retry é definido (mesma Idempotency-Key).
- Seção 13: App e painel web são consistentes após atualização (polling/refresh manual aceito).

#### Lacunas identificadas

| Questão Seen and Heard | Análise |
|------------------------|---------|
| O usuário sabe quanto tempo falta para concluir? | **Lacuna:** sem estimativa de tempo ou barra de progresso determinística — apenas "indicador de carregamento" (indeterminado) durante até 10 segundos |
| O que o usuário vê nos 10 segundos de timeout? | **Lacuna:** o requisito define o comportamento de timeout na API, mas não o que o usuário vê no App enquanto aguarda — apenas "indicador de carregamento" por até 10 segundos pode gerar ansiedade |
| Há aviso de que o sistema está offline? | **Lacuna:** se o App perder conectividade antes de receber resposta, o usuário vê timeout genérico sem saber se é problema de rede ou da API |
| O estado de sincronização do painel web é comunicado? | **Lacuna:** a seção 13 menciona que o painel web precisa de refresh para atualizar, mas não define indicador de "dados possivelmente desatualizados" |
| Após sucesso, a confirmação persiste ou desaparece rapidamente? | **Lacuna:** duração do estado "Concluído" não está definida |

**Risco de ansiedade do usuário:** Uma transferência financeira com 10 segundos de processamento sem feedback de progresso (apenas spinner) pode levar o usuário a:
1. Tocar o botão múltiplas vezes (mitigado pela Idempotency-Key, mas ainda assim confuso).
2. Fechar o App achando que travou.
3. Ficar inseguro sobre se a transferência foi realizada.

**Recomendação:** Especificar que:
- Durante **Processando**: exibir mensagem textual "Transferência em andamento..." além do spinner.
- A partir de 5 segundos sem resposta: exibir mensagem secundária "Isso está demorando mais que o esperado. Aguarde..." para reduzir ansiedade.
- Após 10 segundos (timeout): exibir mensagem específica de timeout com opção clara de "Tentar novamente" (que reutiliza a Idempotency-Key).
- Estado **Concluído**: confirmação visível por pelo menos 4 segundos antes de navegar para a próxima tela ou ser dispensada.
- Detectar ausência de conectividade antes do envio e exibir mensagem "Sem conexão com a internet. Verifique sua rede e tente novamente." antes de gerar erro de timeout.

---

### 2.6 Clareza das Mensagens

#### O que está definido

- Seção 11.2 mapeia 8 códigos de erro para mensagens sugeridas em português, em linguagem de usuário:

| Código | Mensagem definida |
|--------|-------------------|
| `INVALID_AMOUNT` | "Valor deve ser entre 1 e 10.000 QualiPoints." |
| `RECIPIENT_NOT_FOUND` | "Destinatário não encontrado. Verifique o ID ou o status da conta." |
| `SELF_TRANSFER_NOT_ALLOWED` | "Não é permitido enviar QualiPoints para a própria conta." |
| `INSUFFICIENT_BALANCE` | "Saldo insuficiente. Seu saldo atual não permite este envio." |
| `ACCOUNT_NOT_ACTIVE` | "Sua conta não está ativa para envio. Entre em contato com o suporte." |
| `REQUEST_TIMEOUT` | "A operação demorou mais que o esperado. Tente novamente." |
| `INTERNAL_ERROR` | "Ocorreu um erro. Tente novamente em instantes." |
| `UNAUTHORIZED` | "Sessão expirada. Faça login novamente." |

#### Lacunas identificadas

| Questão Seen and Heard | Análise |
|------------------------|---------|
| As mensagens sugerem o próximo passo de forma acionável? | **Parcialmente:** `ACCOUNT_NOT_ACTIVE` orienta contato com suporte, mas `INTERNAL_ERROR` apenas diz "Tente novamente" sem definir após quanto tempo ou quantas vezes |
| O tom das mensagens é adequado (não culpabiliza o usuário)? | **Bem definido:** as mensagens são neutras e não culpabilizam (ex.: "Saldo insuficiente" e não "Você não tem saldo suficiente") |
| A mensagem de sucesso tem conteúdo específico? | **Lacuna:** a seção 11.1 menciona "exibir confirmação de sucesso", mas não define o texto da mensagem (ex.: "Você enviou 500 QualiPoints para [Nome/ID do destinatário]") |
| Há confirmação visual do valor e destinatário enviados? | **Lacuna:** o body da resposta 200 contém `transactionId`, `amount`, `recipientId`, `status`, `completedAt` — mas o requisito não define quais desses dados devem ser exibidos na tela de sucesso |
| A mensagem de timeout orienta sobre retry com mesma key? | **Parcialmente:** a seção 12 explica o comportamento, mas a mensagem sugerida ("A operação demorou mais que o esperado. Tente novamente.") não informa o usuário que retentar é seguro (sem débito duplo) |
| As mensagens são contextualizadas (próximas ao elemento)? | **Não especificado** — o requisito define o conteúdo mas não a localização das mensagens na tela |
| Há diferenciação visual entre tipos de mensagem? | **Não especificado** — sucesso vs. aviso vs. erro não têm padrão visual definido |

**Risco de abandono:** A mensagem de `INTERNAL_ERROR` — "Ocorreu um erro. Tente novamente em instantes." — é genérica demais. Em um contexto financeiro, o usuário pode ficar ansioso sobre se a transferência foi debitada ou não. A falta de uma mensagem de sucesso com detalhes (valor, destinatário) também reduz a confiança no sistema.

**Recomendação:** Especificar que:
- Mensagem de sucesso exibe: "Você enviou **[amount] QualiPoints** para **[recipientId]**." com `transactionId` disponível para consulta.
- Mensagem de `REQUEST_TIMEOUT` é complementada com: "A operação demorou mais que o esperado. Se tentar novamente, a transferência não será duplicada."
- Mensagem de `INTERNAL_ERROR` inclui: "Aguarde alguns minutos antes de tentar novamente. Se o problema persistir, entre em contato com o suporte." com link/botão de contato.
- Mensagens de erro de validação (400/409/422) são exibidas inline, próximas ao campo.
- Mensagens de sistema (401/408/504/5xx) são exibidas como banner no topo ou modal leve.

---

## 3. Bugs de Comunicação Identificados

| # | Bug Potencial | Aspecto Seen and Heard | Severidade | Origem |
|---|---------------|------------------------|------------|--------|
| B-1 | **Estados de Processando/Concluído/Falhou sem ARIA** — leitores de tela não anunciam mudança de estado | Acessibilidade | Alta | Ausência total de spec de ARIA |
| B-2 | **10 segundos de spinner sem feedback textual** — usuário não sabe se o sistema travou | Status e Progresso | Alta | Falta de mensagem de progresso para operações longas |
| B-3 | **Mensagem de sucesso sem detalhes** — usuário não sabe se o valor e destinatário corretos foram enviados | Clareza das Mensagens | Alta | Conteúdo da tela de sucesso não especificado |
| B-4 | **Mensagem de timeout sem orientação de retry seguro** — usuário teme débito duplo | Clareza das Mensagens | Alta | Contexto de idempotência não comunicado ao usuário |
| B-5 | **Campos sem `aria-label` nem `aria-describedby`** — leitores de tela não associam erros aos campos | Acessibilidade | Alta | Ausência de spec de ARIA |
| B-6 | **Erro de conectividade exibido como timeout** — usuário não distingue problema de rede do problema na API | Status e Progresso | Média | Detecção de offline antes do envio não especificada |
| B-7 | **Contraste de cores não especificado** — mensagens de erro podem ser ilegíveis em modo de alto contraste | Feedback Visual | Média | WCAG não referenciado no requisito |
| B-8 | **Sem feedback háptico em mobile** — transferência concluída sem reforço tátil | Feedback Tátil | Baixa | Feedback háptico não especificado |
| B-9 | **Lista de recentes no painel web sem indicador de desatualização** — usuário pode ver dados antigos sem saber | Status e Progresso | Baixa | Estado de sincronização do painel web não definido |

---

## 4. Cenários de Teste Derivados da Heurística Seen and Heard

### 4.1 Feedback Visual

```typescript
// React Testing Library
it('deve exibir spinner e texto durante Processando (Seen and Heard - Visual)', async () => {
  const user = userEvent.setup();
  const slowSubmit = jest.fn(() => new Promise(resolve => setTimeout(resolve, 200)));
  render(<TransferForm onSubmit={slowSubmit} />);

  await user.type(screen.getByLabelText(/Destinatário/), 'user-uuid-123');
  await user.type(screen.getByLabelText(/Valor/), '500');
  await user.click(screen.getByRole('button', { name: /Transferir/ }));

  expect(screen.getByRole('status')).toHaveTextContent(/Processando transferência/i);
  expect(screen.getByRole('button')).toHaveAttribute('aria-busy', 'true');
});

it('deve exibir confirmação com detalhes após sucesso (Seen and Heard - Visual)', async () => {
  const user = userEvent.setup();
  const successSubmit = jest.fn(() => Promise.resolve({
    transactionId: 'tx-123',
    amount: 500,
    recipientId: 'user-uuid-123',
    status: 'completed',
  }));

  render(<TransferForm onSubmit={successSubmit} />);

  await user.type(screen.getByLabelText(/Destinatário/), 'user-uuid-123');
  await user.type(screen.getByLabelText(/Valor/), '500');
  await user.click(screen.getByRole('button', { name: /Transferir/ }));

  await waitFor(() => {
    const successAlert = screen.getByRole('status');
    expect(successAlert).toHaveTextContent(/500 QualiPoints/);
    expect(successAlert).toHaveTextContent(/user-uuid-123/);
    expect(successAlert).toBeVisible();
  });
});

it('deve exibir erro inline para saldo insuficiente (Seen and Heard - Visual)', async () => {
  const user = userEvent.setup();
  const failSubmit = jest.fn(() => Promise.reject({ code: 'INSUFFICIENT_BALANCE' }));

  render(<TransferForm onSubmit={failSubmit} />);

  await user.type(screen.getByLabelText(/Destinatário/), 'user-uuid-123');
  await user.type(screen.getByLabelText(/Valor/), '9999');
  await user.click(screen.getByRole('button', { name: /Transferir/ }));

  await waitFor(() => {
    const errorMessage = screen.getByRole('alert');
    expect(errorMessage).toHaveTextContent(/Saldo insuficiente/i);
    expect(errorMessage).toBeVisible();
  });
});
```

### 4.2 Acessibilidade (ARIA)

```typescript
// React Testing Library + jest-axe
import { axe } from 'jest-axe';

it('deve não ter violações de acessibilidade no formulário (Seen and Heard - Acessibilidade)', async () => {
  const { container } = render(<TransferForm />);

  const results = await axe(container);
  expect(results).toHaveNoViolations();
});

it('campos obrigatórios devem ter aria-required (Seen and Heard - Acessibilidade)', () => {
  render(<TransferForm />);

  expect(screen.getByLabelText(/Destinatário/)).toHaveAttribute('aria-required', 'true');
  expect(screen.getByLabelText(/Valor/)).toHaveAttribute('aria-required', 'true');
});

it('mensagem de erro deve ser associada ao campo por aria-describedby (Seen and Heard - Acessibilidade)', async () => {
  const user = userEvent.setup();
  render(<TransferForm />);

  const amountInput = screen.getByLabelText(/Valor/);
  await user.type(amountInput, '0');
  await user.tab();

  const errorMessage = screen.getByText(/Valor deve ser ao menos 1/);
  const errorId = errorMessage.getAttribute('id');
  expect(amountInput).toHaveAttribute('aria-describedby', errorId);
});

it('erro deve ser anunciado para leitores de tela via role alert (Seen and Heard - Acessibilidade)', async () => {
  const user = userEvent.setup();
  const failSubmit = jest.fn(() => Promise.reject({ code: 'INSUFFICIENT_BALANCE' }));

  render(<TransferForm onSubmit={failSubmit} />);

  await user.type(screen.getByLabelText(/Destinatário/), 'user-uuid-123');
  await user.type(screen.getByLabelText(/Valor/), '500');
  await user.click(screen.getByRole('button'));

  await waitFor(() => {
    const errorRegion = screen.getByRole('alert');
    expect(errorRegion).toHaveAttribute('aria-live', 'assertive');
    expect(errorRegion).toHaveTextContent(/Saldo insuficiente/);
  });
});

it('deve ter navegação por teclado funcional (Seen and Heard - Acessibilidade)', async () => {
  const user = userEvent.setup();
  render(<TransferForm />);

  await user.tab();
  expect(screen.getByLabelText(/Destinatário/)).toHaveFocus();

  await user.tab();
  expect(screen.getByLabelText(/Valor/)).toHaveFocus();

  await user.tab();
  expect(screen.getByRole('button', { name: /Transferir/ })).toHaveFocus();
});
```

### 4.3 Status e Progresso

```typescript
it('deve exibir mensagem adicional após 5s de processamento (Seen and Heard - Progresso)', async () => {
  jest.useFakeTimers();
  const user = userEvent.setup({ advanceTimers: jest.advanceTimersByTime });
  const slowSubmit = jest.fn(() => new Promise(resolve => setTimeout(resolve, 8000)));

  render(<TransferForm onSubmit={slowSubmit} />);

  await user.type(screen.getByLabelText(/Destinatário/), 'user-uuid-123');
  await user.type(screen.getByLabelText(/Valor/), '500');
  await user.click(screen.getByRole('button', { name: /Transferir/ }));

  // Após 5 segundos
  jest.advanceTimersByTime(5000);

  await waitFor(() => {
    expect(screen.getByText(/Isso está demorando mais que o esperado/i)).toBeInTheDocument();
  });

  jest.useRealTimers();
});

it('mensagem de timeout deve orientar sobre retry seguro (Seen and Heard - Clareza)', async () => {
  const user = userEvent.setup();
  const timeoutSubmit = jest.fn(() => Promise.reject({ code: 'REQUEST_TIMEOUT' }));

  render(<TransferForm onSubmit={timeoutSubmit} />);

  await user.type(screen.getByLabelText(/Destinatário/), 'user-uuid-123');
  await user.type(screen.getByLabelText(/Valor/), '500');
  await user.click(screen.getByRole('button'));

  await waitFor(() => {
    const errorMessage = screen.getByRole('alert');
    expect(errorMessage).toHaveTextContent(/não será duplicada/i);
    expect(screen.getByRole('button', { name: /Tentar novamente/ })).toBeInTheDocument();
  });
});
```

### 4.4 Clareza das Mensagens

```typescript
it('mensagem de sucesso deve conter valor e destinatário (Seen and Heard - Clareza)', async () => {
  const user = userEvent.setup();
  const successSubmit = jest.fn(() => Promise.resolve({
    transactionId: 'tx-123', amount: 500, recipientId: 'user-uuid-123', status: 'completed',
  }));

  render(<TransferForm onSubmit={successSubmit} />);

  await user.type(screen.getByLabelText(/Destinatário/), 'user-uuid-123');
  await user.type(screen.getByLabelText(/Valor/), '500');
  await user.click(screen.getByRole('button'));

  await waitFor(() => {
    expect(screen.getByText(/Você enviou 500 QualiPoints para user-uuid-123/i)).toBeInTheDocument();
  });
});

it('mensagem de erro interno deve orientar sobre suporte (Seen and Heard - Clareza)', async () => {
  const user = userEvent.setup();
  const internalError = jest.fn(() => Promise.reject({ code: 'INTERNAL_ERROR' }));

  render(<TransferForm onSubmit={internalError} />);

  await user.type(screen.getByLabelText(/Destinatário/), 'user-uuid-123');
  await user.type(screen.getByLabelText(/Valor/), '500');
  await user.click(screen.getByRole('button'));

  await waitFor(() => {
    const errorMessage = screen.getByRole('alert');
    expect(errorMessage).toHaveTextContent(/Tente novamente/i);
    expect(errorMessage).toHaveTextContent(/suporte/i);
  });
});
```

**Java (Selenium + Axe-core):**

```java
@Test
@DisplayName("Deve não ter violações de acessibilidade na tela de transferência")
void shouldHaveNoAccessibilityViolations() {
    driver.get(BASE_URL + "/transfer");

    AxeBuilder axeBuilder = new AxeBuilder();
    Results results = axeBuilder.analyze(driver);

    List<Rule> violations = results.getViolations();
    assertEquals(0, violations.size(),
        "Violações encontradas: " + violations.stream()
            .map(Rule::getDescription)
            .collect(Collectors.joining(", ")));
}

@Test
@DisplayName("Deve exibir confirmação com detalhes após transferência bem-sucedida")
void shouldShowSuccessConfirmationWithDetails() {
    driver.get(BASE_URL + "/transfer");

    driver.findElement(By.name("recipient_id")).sendKeys("user-uuid-123");
    driver.findElement(By.name("amount")).sendKeys("500");
    driver.findElement(By.cssSelector("button[type='submit']")).click();

    WebElement successMessage = wait.until(
        ExpectedConditions.presenceOfElementLocated(By.cssSelector("[role='status']"))
    );

    assertTrue(successMessage.getText().contains("500 QualiPoints"));
    assertTrue(successMessage.getText().contains("user-uuid-123"));
    assertTrue(successMessage.isDisplayed());
}

@Test
@DisplayName("Mensagem de timeout deve mencionar que retry é seguro")
void timeoutMessageShouldMentionSafeRetry() {
    // Configurar mock de servidor com timeout...
    driver.get(BASE_URL + "/transfer");

    driver.findElement(By.name("recipient_id")).sendKeys("user-uuid-123");
    driver.findElement(By.name("amount")).sendKeys("500");
    driver.findElement(By.cssSelector("button[type='submit']")).click();

    WebElement errorMessage = wait.until(
        ExpectedConditions.presenceOfElementLocated(By.cssSelector("[role='alert']"))
    );

    String messageText = errorMessage.getText().toLowerCase();
    assertTrue(messageText.contains("duplicada") || messageText.contains("seguro"),
        "Mensagem de timeout deve informar que retry não gera débito duplo");
}

@Test
@DisplayName("Deve ter Tab order correto entre campos")
void shouldHaveCorrectTabOrder() {
    driver.get(BASE_URL + "/transfer");

    WebElement recipientInput = driver.findElement(By.name("recipient_id"));
    WebElement amountInput = driver.findElement(By.name("amount"));
    WebElement submitButton = driver.findElement(By.cssSelector("button[type='submit']"));

    // Foco inicial no recipientId
    recipientInput.click();
    assertEquals(recipientInput, driver.switchTo().activeElement());

    // Tab para amount
    recipientInput.sendKeys(Keys.TAB);
    assertEquals(amountInput, driver.switchTo().activeElement());

    // Tab para botão
    amountInput.sendKeys(Keys.TAB);
    assertEquals(submitButton, driver.switchTo().activeElement());
}
```

---

## 5. Lacunas e Riscos Consolidados

| # | Lacuna / Risco | Aspecto Seen and Heard | Severidade | Recomendação |
|---|----------------|------------------------|------------|--------------|
| L-1 | **ARIA não especificado** — estados Processando/Concluído/Falhou inacessíveis para leitores de tela | Acessibilidade | Alta | Definir `role="status"` (sucesso), `role="alert"` (erro), `aria-busy` (loading) |
| L-2 | **Conteúdo da tela de sucesso não especificado** — usuário não confirma valor e destinatário | Clareza das Mensagens | Alta | Especificar que tela de sucesso exibe valor enviado e ID/nome do destinatário |
| L-3 | **Mensagem de timeout não orienta sobre retry seguro** | Clareza das Mensagens | Alta | Adicionar "Se tentar novamente, a transferência não será duplicada." |
| L-4 | **10 segundos de spinner sem mensagem textual** — gera ansiedade e possível abandono | Status e Progresso | Alta | Definir mensagem "Transferência em andamento..." e feedback adicional após 5s |
| L-5 | **Campos sem labels ARIA** — inacessíveis para leitores de tela | Acessibilidade | Alta | Especificar `aria-label` / `<label for>` e `aria-describedby` para erros |
| L-6 | **Tipo de indicador de carregamento não definido** | Feedback Visual | Média | Especificar spinner + texto (não apenas spinner animado) |
| L-7 | **Diferenciação visual entre tipos de erro não definida** | Feedback Visual | Média | Definir inline (400/409/422) vs. banner/toast (401/408/504) |
| L-8 | **Contraste de cores WCAG não referenciado** | Acessibilidade | Média | Adicionar requisito mínimo 4.5:1 para texto de interface |
| L-9 | **Detecção de offline antes do envio não especificada** | Status e Progresso | Média | Verificar conectividade antes de submeter e exibir mensagem específica |
| L-10 | **Feedback háptico em mobile não especificado** | Feedback Tátil | Baixa | Especificar vibração de sucesso (curta, suave) e erro (dupla) |
| L-11 | **Indicador de dados desatualizados no painel web não definido** | Status e Progresso | Baixa | Exibir timestamp da última atualização ou botão de refresh explícito |

---

## 6. Recomendações por Prioridade

### Para o MVP (inegociável)

| # | Recomendação | Justificativa |
|---|-------------|---------------|
| R-1 | Especificar ARIA para estados da operação (`role="status"`, `role="alert"`, `aria-busy`) | Acessibilidade é requisito fundamental em apps financeiros |
| R-2 | Definir conteúdo da tela de sucesso com valor e destinatário | Confirma ao usuário que a operação correta foi concluída |
| R-3 | Complementar mensagem de timeout com orientação sobre retry seguro | Reduz medo de débito duplo e incentiva o retry |
| R-4 | Definir feedback visual de Processando com mensagem textual além do spinner | Spinner sem texto após 5s gera abandono |
| R-5 | Especificar `aria-label` e `aria-describedby` para campos e mensagens de erro | Formulário inacessível para VoiceOver/TalkBack sem esses atributos |

### Para o Backlog

| # | Recomendação |
|---|-------------|
| R-6 | Definir padrão de contraste WCAG 2.1 AA (mínimo 4.5:1) para toda a interface |
| R-7 | Especificar feedback háptico de sucesso e erro em mobile |
| R-8 | Implementar detecção de offline antes do submit com mensagem específica |
| R-9 | Adicionar mensagem de progresso adicional após 5s de processamento |
| R-10 | Definir indicador de última atualização ou botão de refresh explícito no painel web |
| R-11 | Realizar auditoria de acessibilidade com axe-core antes do lançamento |

---

## 7. Heurísticas Complementares Recomendadas

### Chique
**Conexão com Seen and Heard:** A comunicação visual (Seen and Heard) e a elegância dos elementos de interface (Chique) são complementares. Os indicadores visuais de estados (spinner, toast, inline error) que Seen and Heard especifica precisam da elegância Chique para serem exibidos no momento certo, com a duração correta e sem interromper desnecessariamente o fluxo.

**Recomendação:** Aplicar **Chique** para garantir que as mensagens de estado definidas por Seen and Heard sejam exibidas sem modais bloqueantes desnecessários (Interrupção da Ação), com botão habilitado somente quando adequado (Habilitar/Desabilitar) e com o formulário preservado após erros.

### Input Method
**Conexão com Seen and Heard:** O feedback de validação em tempo real (Seen and Heard - Feedback Visual) depende de como o usuário insere os dados (Input Method - Typing). Sem definir quando a validação ocorre (ao digitar, ao perder foco, ao submeter), não é possível definir quando o feedback visual deve aparecer.

**Recomendação:** Aplicar **Input Method** para alinhar o timing do feedback visual com os eventos de entrada de dados — especialmente para o campo `amount` onde a validação de range deve ser visível antes da chamada à API.

### Emotions (Empatia e Emoção)
**Conexão com Seen and Heard:** A clareza das mensagens (Seen and Heard) afeta diretamente as emoções do usuário. Mensagens de erro bem escritas reduzem ansiedade; confirmações de sucesso com detalhes geram confiança. A heurística Emotions complementa com foco na resposta emocional gerada pela comunicação do sistema.

**Recomendação:** Aplicar **Emotions** para avaliar se as mensagens definidas em Seen and Heard (especialmente `INTERNAL_ERROR`, `ACCOUNT_NOT_ACTIVE` e a tela de sucesso) geram as emoções desejadas (confiança, clareza, segurança) e não as indesejadas (ansiedade, culpa, confusão).

---

## Referências

- **Requisito analisado:** [REQ_INICIAL_V2.md](../requisito-revisado/REQ_INICIAL_V2.md)
- **Skill aplicada:** [SeenAndHeard.md](../../Skill/Cap07_refinamento/SeenAndHeard.md)
- **Análises complementares já realizadas:** [COUNT_REQ_INICIAL_V2.md](./COUNT_REQ_INICIAL_V2.md), [CRUD_REQ_INICIAL_V2.md](./CRUD_REQ_INICIAL_V2.md), [CHIQUE_REQ_INICIAL_V2.md](./CHIQUE_REQ_INICIAL_V2.md), [INPUT_METHOD_REQ_INICIAL_V2.md](./INPUT_METHOD_REQ_INICIAL_V2.md)
- **Próximas heurísticas recomendadas:** Chique, Input Method, Emotions (ver seção 7)
