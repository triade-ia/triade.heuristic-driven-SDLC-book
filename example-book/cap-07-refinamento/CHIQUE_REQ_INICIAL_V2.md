# Relatório: Chique — REQ_INICIAL_V2 (QualiPoints)

**Requisito analisado:** Envio de QualiPoints entre usuários (REQ_INICIAL_V2).
**Skill aplicada:** Cap07_refinamento — Chique (Elegância e Polidez na Interação Essencial)
**Data:** 2026-02-18
**Status:** Análise de refinamento concluída

---

## 1. Elementos de Interação Identificados no Requisito

Antes de aplicar os aspectos Chique, mapeamos todos os elementos de formulário, fluxos e interações presentes ou implícitos no requisito:

| Elemento / Fluxo | Definido no Req | Observação |
|-------------------|-----------------|------------|
| **Campo `recipientId`** | Obrigatório (implícito) | Tipo de dado, limite de caracteres e feedback visual não especificados |
| **Campo `amount`** | Obrigatório, 1–10.000, inteiro | Indicador de obrigatoriedade e feedback de limite não especificados |
| **Botão "Enviar"** | Citado em estado Processando | Lógica de habilitar/desabilitar antes do envio não especificada |
| **Indicador de carregamento** | Mencionado (estado Processando) | Tipo de indicador (spinner, barra, skeleton) não especificado |
| **Mensagens de erro** | Mapeadas no contrato (seção 11.2) | Forma de exibição (inline, toast, modal) não especificada |
| **Mensagem de sucesso** | Mencionada (estado Concluído) | Conteúdo e duração não especificados |
| **Lista de transações recentes** | Especificada (últimas 10) | Comportamento de empty state não especificado |
| **Fluxo de expiração de sessão** | Erro 401 mapeado | Preservação do contexto do formulário não definida |

---

## 2. Análise por Aspecto Chique

### 2.1 Campos Obrigatórios

O requisito exige `recipientId` e `amount` no corpo da requisição (seção 10.3), mas não especifica como esses campos são apresentados no App.

#### O que está definido

- A API rejeita requisições sem `recipientId` ou `amount` inválido com HTTP 400 (seção 10.4).
- A seção 6.1 afirma que "o App pode exibir feedback de UX (ex.: campo obrigatório)", delegando a responsabilidade para a camada de UI sem especificá-la.
- Mensagens de erro por código estão mapeadas em linguagem de usuário (seção 11.2).

#### Lacunas identificadas

| Questão Chique | Análise |
|----------------|---------|
| Como os campos obrigatórios são visualmente diferenciados? | **Lacuna:** nenhum padrão de indicação está especificado (asterisco, rótulo destacado, cor diferente) |
| A indicação de obrigatoriedade é consistente em todo o sistema? | **Lacuna:** o painel web não tem sua UX de formulário especificada; consistência entre App e painel indefinida |
| A mensagem indica qual campo está faltando? | Parcialmente — os códigos de erro são específicos por tipo, mas a mensagem de `INVALID_AMOUNT` cobre tanto "ausência de valor" quanto "valor fora do range", sem distinção |
| A validação ocorre no momento certo? | **Lacuna crítica:** o requisito não define o timing de validação no App (em tempo real, ao perder foco ou apenas no submit) |
| Há feedback visual imediato quando o campo é preenchido? | **Não especificado** — ausência de indicação visual de "campo válido" (ex.: borda verde, ícone de check) |

**Risco:** O usuário pode preencher ambos os campos, tocar em "Enviar", e só descobrir que o `recipientId` está inválido após a chamada à API (latência de rede), quando uma validação em tempo real poderia antecipar o feedback e reduzir o tempo de correção.

**Recomendação:** Especificar que:
- Ambos os campos exibem asterisco (`*`) ou rótulo "obrigatório".
- `amount` valida em tempo real (range 1–10.000) ao sair do campo (blur).
- `recipientId` exibe estado "verificando" ao perder foco (opcional: lookup assíncrono do destinatário).
- Atributos `aria-required="true"` e `aria-describedby` para mensagens de erro.

---

### 2.2 Habilitar/Desabilitar Formulários

#### O que está definido

- A seção 11.1 especifica: "desabilitar novo envio até resposta" enquanto a requisição está em estado **Processando**.
- O botão é reabilitado para retry após falha.

#### Lacunas identificadas

| Questão Chique | Análise |
|----------------|---------|
| O botão "Enviar" está desabilitado quando os campos estão vazios? | **Lacuna:** o requisito não define se o botão está habilitado antes do preenchimento completo |
| A razão da desabilitação é clara para o usuário? | **Lacuna:** se o botão estiver desabilitado (durante Processando), não há especificação de tooltip ou indicador explicando o motivo |
| Há feedback visual adequado para o estado desabilitado? | **Lacuna:** aparência visual do botão nos estados habilitado, desabilitado (vazio) e desabilitado (processando) não diferenciada no requisito |
| O que acontece com a Idempotency-Key quando o usuário edita os campos após uma falha? | **Lacuna crítica:** se o usuário altera o `recipientId` ou `amount` após um erro e tenta novamente com a mesma key, ocorre conflito semântico — o requisito só menciona "mesma intenção de envio" sem definir quando uma nova key deve ser gerada |

**Risco:** Um botão "Enviar" habilitado com campos vazios induz cliques prematuros e submissões inválidas, sobrecarregando a API com requisições 400.

**Recomendação:** Definir que o botão "Enviar" fica desabilitado enquanto qualquer campo obrigatório estiver vazio. Especificar que uma nova `Idempotency-Key` é gerada sempre que o usuário altera os dados do formulário após uma tentativa anterior (novo UUID ao modificar `recipientId` ou `amount`).

---

### 2.3 Interrupção da Ação

#### O que está definido

- Seção 11.1 define os três estados visíveis: **Processando**, **Concluído** e **Falhou**.
- Seção 11.2 mapeia 8 códigos de erro para mensagens em linguagem de usuário.
- Em **Processando**: exibir indicador de carregamento e desabilitar novo envio.

#### Lacunas identificadas

| Questão Chique | Análise |
|----------------|---------|
| Como os erros são exibidos? (inline, modal, toast, banner) | **Lacuna:** o requisito define o conteúdo das mensagens de erro, mas não o mecanismo de exibição |
| O contexto do formulário é preservado após um erro? | **Lacuna:** não está especificado se os campos `recipientId` e `amount` permanecem preenchidos após uma resposta de erro da API |
| Após corrigir o erro, a mensagem desaparece automaticamente? | **Não especificado** |
| Pop-ups/modais são usados ou há alternativas menos intrusivas? | **Não especificado** — um modal bloqueante para `INSUFFICIENT_BALANCE` seria desnecessariamente intrusivo |
| Erros de timeout (408/504) são diferenciados visualmente dos erros de validação (400)? | **Lacuna:** a seção 11.2 lista as mensagens, mas não diferencia severidade visual |

**Risco:** Se os campos forem limpos após um erro de `INSUFFICIENT_BALANCE`, o usuário precisará redigitar todos os dados para tentar novamente, aumentando o atrito e o risco de abandono.

**Recomendação:** Especificar que:
- Erros de validação (400, 409, 422) são exibidos como mensagens inline, próximas ao campo relacionado, sem modal.
- Erros de autenticação (401) e timeout (408/504) são exibidos como banner no topo da tela ou toast.
- O formulário preserva os valores preenchidos após qualquer resposta de erro.
- A mensagem de erro desaparece automaticamente ao iniciar nova tentativa.

---

### 2.4 Quebra de Fluxos

#### O que está definido

- Erro 401 (`UNAUTHORIZED`) → mensagem "Sessão expirada. Faça login novamente." (seção 11.2).
- Timeout: usuário pode tentar novamente com a mesma `Idempotency-Key` (seção 12).
- Após sucesso: saldo e lista de recentes são atualizados (seção 11.1).

#### Lacunas identificadas

| Questão Chique | Análise |
|----------------|---------|
| O que acontece com o formulário preenchido quando a sessão expira? | **Lacuna:** a seção 11.2 informa a mensagem ("Sessão expirada"), mas não define se os dados do formulário são preservados para após o re-login |
| O usuário consegue retomar de onde parou após o re-login? | **Lacuna:** o fluxo App → login → retorno ao formulário pré-preenchido não está especificado |
| O usuário é avisado que o formulário tem dados não enviados antes de navegar para outra tela? | **Não especificado** |
| Links externos (painel web) abrem sem perder o contexto do App? | **Não aplicável no fluxo de envio** — o painel web é de consulta; o envio é exclusivo do App |
| O estado **Processando** bloqueia a navegação do usuário para outras telas? | **Lacuna:** o requisito diz para "desabilitar novo envio", mas não define se o usuário pode navegar para outra tela durante o processamento |

**Risco:** Se a sessão expirar enquanto o usuário preenche o formulário de envio, a perda dos dados digitados gera frustração e retrabalho, especialmente em transferências para destinatários menos conhecidos (cuja busca pelo ID pode ter sido trabalhosa).

**Recomendação:** Especificar que os dados do formulário são preservados localmente no App durante a sessão do dispositivo. Definir que, ao retornar após re-login, o App restaura os dados e solicita nova tentativa de envio.

---

### 2.5 Usabilidade dos Menus

#### O que está definido

- Canais definidos: App mobile (envio e consulta) e painel web (consulta de extrato e saldo) — seção 2.
- Escopo MVP não inclui navegação complexa: é uma tela de envio + lista de recentes.

#### Lacunas identificadas

| Questão Chique | Análise |
|----------------|---------|
| Como o usuário chega à tela de envio? | **Não especificado** — o requisito descreve o formulário de envio, mas não como o usuário acessa essa tela no App |
| A lista de transações recentes está no mesmo fluxo ou em tela separada? | **Lacuna:** a relação de navegação entre tela de envio e lista de recentes não está definida |
| Após envio bem-sucedido, o usuário é redirecionado para a lista de recentes ou permanece na tela de envio? | **Lacuna:** o requisito menciona "atualizar lista de recentes" (seção 11.1), mas não o comportamento de navegação pós-sucesso |

**Risco baixo para MVP:** A estrutura de navegação simples (tela de envio + lista) reduz o risco de menus complexos, mas a ausência de definição do fluxo pós-sucesso pode levar a experiências inconsistentes entre plataformas (iOS/Android).

**Recomendação:** Especificar que, após sucesso, o App exibe mensagem de confirmação na tela de envio e, após 2–3 segundos ou ao toque do usuário, navega para a lista de transações recentes atualizada.

---

### 2.6 Estouro de Campos

#### O que está definido

- Campo `amount`: inteiro, mínimo 1, máximo 10.000 (seção 4.2).
- Valores zero ou negativos rejeitados com HTTP 400 e código `INVALID_AMOUNT` (seção 4.2).
- Valor acima de 10.000 rejeitado com HTTP 400 e código `INVALID_AMOUNT` (seção 10.4).

#### Lacunas identificadas

| Questão Chique | Análise |
|----------------|---------|
| Há limite de caracteres visível para `recipientId`? | **Lacuna:** o requisito não define tamanho máximo do ID do destinatário nem como isso é comunicado ao usuário |
| O campo `amount` impede digitação acima de 10.000 ou valida depois? | **Não especificado** — prevenção vs. validação posterior não definida |
| Há contador de caracteres ou indicador de range para `amount`? | **Lacuna:** nenhum contador ou indicador de limite (ex.: "1 a 10.000 QualiPoints") está especificado para o campo |
| O que acontece ao colar "10001" no campo `amount`? | **Lacuna:** comportamento de paste acima do limite não está definido (rejeitar silenciosamente, truncar, ou exibir erro?) |
| Campo numérico aceita letras? O que acontece? | **Lacuna:** o requisito define que `amount` é inteiro na API, mas não especifica validação de tipo no App (campo de texto livre ou campo numérico nativo?) |

**Risco:** Permitir que o usuário escreva "10001" no campo `amount` e só descobrir o erro após a chamada à API é um atrito desnecessário. Um campo com validação de range em tempo real (ou que impede digitação além de 5 dígitos) previne o erro antes da submissão.

**Recomendação:** Especificar que:
- O campo `amount` no App é do tipo numérico (exibe teclado numérico em mobile).
- O App valida o range 1–10.000 em tempo real (ao digitar ou ao perder foco), exibindo hint "Entre 1 e 10.000 QualiPoints" abaixo do campo.
- O campo `recipientId` tem limite máximo de caracteres definido (ex.: 36 para UUID) e exibe indicador de limite.
- Valores colados acima do máximo são sinalizados imediatamente com mensagem inline, sem aguardar o submit.

---

## 3. Bugs de Interação Identificados

| # | Bug Potencial | Aspecto Chique | Severidade | Origem |
|---|---------------|----------------|------------|--------|
| B-1 | **Botão "Enviar" habilitado com campos vazios** gera submissão prematura e erro desnecessário na API | Habilitar/Desabilitar | Alta | Ausência de validação de campos antes de habilitar o botão |
| B-2 | **Formulário limpo após erro** obriga o usuário a redigitar todos os dados | Interrupção da Ação | Alta | Comportamento pós-erro não especificado |
| B-3 | **Reutilização da Idempotency-Key com dados alterados** pode reprocessar uma intenção já registrada | Habilitar/Desabilitar | Alta | Ciclo de vida da chave não definido |
| B-4 | **Campo `amount` aceita letras** sem validação de tipo no App, gerando erro 400 genérico na API | Estouro de Campos | Média | Tipo de campo de entrada não especificado |
| B-5 | **Mensagem de erro desaparece** antes do usuário ler (auto-dismiss não controlado) | Interrupção da Ação | Média | Duração das mensagens não definida |
| B-6 | **Sessão expira durante preenchimento** e formulário é perdido sem aviso | Quebra de Fluxos | Média | Preservação de estado pré-autenticação não especificada |
| B-7 | **Tela em branco** na lista de transações para usuários novos, sem mensagem orientativa | Campos Obrigatórios | Baixa | Empty state não especificado |

---

## 4. Cenários de Teste Derivados da Heurística Chique

### 4.1 Campos Obrigatórios

```typescript
// React Testing Library
it('deve indicar campos obrigatórios com asterisco e aria-required (Chique - Campos Obrigatórios)', () => {
  render(<TransferForm />);

  expect(screen.getByText(/Destinatário \*/)).toBeInTheDocument();
  expect(screen.getByText(/Valor \*/)).toBeInTheDocument();
  expect(screen.getByLabelText(/Destinatário/)).toHaveAttribute('aria-required', 'true');
  expect(screen.getByLabelText(/Valor/)).toHaveAttribute('aria-required', 'true');
});

it('deve exibir erros específicos por campo ao submeter vazio (Chique - Campos Obrigatórios)', async () => {
  const user = userEvent.setup();
  render(<TransferForm />);

  await user.click(screen.getByRole('button', { name: /Transferir/ }));

  expect(screen.getByText(/Destinatário é obrigatório/)).toBeInTheDocument();
  expect(screen.getByText(/Valor é obrigatório/)).toBeInTheDocument();
});

it('deve exibir erro de range ao digitar valor fora do intervalo (Chique - Campos Obrigatórios)', async () => {
  const user = userEvent.setup();
  render(<TransferForm />);

  const amountInput = screen.getByLabelText(/Valor/);
  await user.type(amountInput, '10001');
  await user.tab(); // blur

  expect(screen.getByText(/Valor deve ser entre 1 e 10.000 QualiPoints/)).toBeInTheDocument();
});
```

### 4.2 Habilitar/Desabilitar

```typescript
it('botão Transferir deve estar desabilitado quando campos estão vazios (Chique - Habilitar/Desabilitar)', () => {
  render(<TransferForm />);

  expect(screen.getByRole('button', { name: /Transferir/ })).toBeDisabled();
});

it('botão deve ser habilitado somente quando ambos os campos são válidos (Chique - Habilitar/Desabilitar)', async () => {
  const user = userEvent.setup();
  render(<TransferForm />);

  await user.type(screen.getByLabelText(/Destinatário/), 'user-uuid-123');
  expect(screen.getByRole('button', { name: /Transferir/ })).toBeDisabled(); // Apenas um campo preenchido

  await user.type(screen.getByLabelText(/Valor/), '500');
  expect(screen.getByRole('button', { name: /Transferir/ })).toBeEnabled();
});

it('botão deve estar desabilitado durante processamento (Chique - Habilitar/Desabilitar)', async () => {
  const user = userEvent.setup();
  const slowSubmit = jest.fn(() => new Promise(resolve => setTimeout(resolve, 500)));
  render(<TransferForm onSubmit={slowSubmit} />);

  await user.type(screen.getByLabelText(/Destinatário/), 'user-uuid-123');
  await user.type(screen.getByLabelText(/Valor/), '500');
  await user.click(screen.getByRole('button', { name: /Transferir/ }));

  expect(screen.getByRole('button', { name: /Transferir/ })).toBeDisabled();
});
```

### 4.3 Interrupção da Ação

```typescript
it('deve preservar campos preenchidos após erro da API (Chique - Interrupção)', async () => {
  const user = userEvent.setup();
  const failSubmit = jest.fn(() => Promise.reject({ code: 'INSUFFICIENT_BALANCE' }));
  render(<TransferForm onSubmit={failSubmit} />);

  await user.type(screen.getByLabelText(/Destinatário/), 'user-uuid-123');
  await user.type(screen.getByLabelText(/Valor/), '500');
  await user.click(screen.getByRole('button', { name: /Transferir/ }));

  await waitFor(() => {
    // Formulário preserva os valores
    expect(screen.getByLabelText(/Destinatário/)).toHaveValue('user-uuid-123');
    expect(screen.getByLabelText(/Valor/)).toHaveValue(500);
    // Mensagem de erro exibida inline
    expect(screen.getByRole('alert')).toHaveTextContent(/Saldo insuficiente/);
  });
});

it('deve exibir indicador de processamento visível (Chique - Interrupção)', async () => {
  const user = userEvent.setup();
  const slowSubmit = jest.fn(() => new Promise(resolve => setTimeout(resolve, 200)));
  render(<TransferForm onSubmit={slowSubmit} />);

  await user.type(screen.getByLabelText(/Destinatário/), 'user-uuid-123');
  await user.type(screen.getByLabelText(/Valor/), '500');
  await user.click(screen.getByRole('button', { name: /Transferir/ }));

  expect(screen.getByRole('status')).toHaveTextContent(/Processando/);
  expect(screen.getByRole('button')).toHaveAttribute('aria-busy', 'true');
});
```

### 4.4 Estouro de Campos

```typescript
it('deve exibir hint de range abaixo do campo amount (Chique - Estouro)', () => {
  render(<TransferForm />);

  expect(screen.getByText(/Entre 1 e 10\.000 QualiPoints/)).toBeInTheDocument();
});

it('deve rejeitar valor colado acima de 10.000 com mensagem inline (Chique - Estouro)', async () => {
  const user = userEvent.setup();
  render(<TransferForm />);

  const amountInput = screen.getByLabelText(/Valor/);
  await user.click(amountInput);
  await user.paste('999999');

  expect(screen.getByText(/Valor deve ser entre 1 e 10\.000 QualiPoints/)).toBeInTheDocument();
});

it('campo amount deve aceitar apenas dígitos numéricos (Chique - Estouro)', async () => {
  const user = userEvent.setup();
  render(<TransferForm />);

  const amountInput = screen.getByLabelText(/Valor/);
  await user.type(amountInput, 'abc');

  expect(amountInput).toHaveValue(null); // Campo numérico ignora letras
});
```

**Java (Selenium):**

```java
@Test
@DisplayName("Botão deve estar desabilitado com campos vazios")
void buttonDisabledWhenFieldsEmpty() {
    driver.get(BASE_URL + "/transfer");

    WebElement button = driver.findElement(By.cssSelector("button[type='submit']"));
    assertFalse(button.isEnabled(), "Botão deve estar desabilitado quando campos vazios");
}

@Test
@DisplayName("Deve preservar campos após erro de saldo insuficiente")
void shouldPreserveFieldsAfterInsufficientBalanceError() {
    driver.get(BASE_URL + "/transfer");

    driver.findElement(By.name("recipient_id")).sendKeys("user-uuid-123");
    driver.findElement(By.name("amount")).sendKeys("99999");
    driver.findElement(By.cssSelector("button[type='submit']")).click();

    WebElement error = wait.until(
        ExpectedConditions.presenceOfElementLocated(By.cssSelector("[role='alert']"))
    );
    assertTrue(error.getText().contains("Saldo insuficiente"));

    // Verificar que campos mantiveram os valores
    assertEquals("user-uuid-123", driver.findElement(By.name("recipient_id")).getAttribute("value"));
    assertEquals("99999", driver.findElement(By.name("amount")).getAttribute("value"));
}

@Test
@DisplayName("Deve exibir hint de range no campo amount")
void shouldShowRangeHintForAmountField() {
    driver.get(BASE_URL + "/transfer");

    WebElement hint = driver.findElement(By.cssSelector(".amount-hint, [data-testid='amount-hint']"));
    assertTrue(hint.getText().contains("1") && hint.getText().contains("10.000"));
}
```

---

## 5. Lacunas e Riscos Consolidados

| # | Lacuna / Risco | Aspecto Chique | Severidade | Recomendação |
|---|----------------|----------------|------------|--------------|
| L-1 | **Indicadores visuais de campos obrigatórios não especificados** | Campos Obrigatórios | Alta | Definir padrão de asterisco + aria-required em ambos os campos |
| L-2 | **Timing de validação no App não definido** | Campos Obrigatórios | Alta | Especificar validação de range do `amount` ao perder foco (blur) |
| L-3 | **Botão "Enviar" sem lógica de habilitar por preenchimento** | Habilitar/Desabilitar | Alta | Definir que botão fica desabilitado enquanto algum campo estiver vazio ou inválido |
| L-4 | **Ciclo de vida da Idempotency-Key após edição não definido** | Habilitar/Desabilitar | Alta | Especificar geração de nova key quando o usuário altera dados após tentativa anterior |
| L-5 | **Forma de exibição das mensagens de erro não especificada** | Interrupção da Ação | Alta | Definir inline para 400/422/409, banner/toast para 401/408/504 |
| L-6 | **Preservação do formulário após erro não especificada** | Interrupção da Ação | Alta | Especificar que campos permanecem preenchidos após qualquer resposta de erro |
| L-7 | **Comportamento pós-sucesso (navegação) não definido** | Quebra de Fluxos | Média | Especificar redirecionamento para lista de recentes após confirmação de sucesso |
| L-8 | **Preservação de formulário após expiração de sessão não definida** | Quebra de Fluxos | Média | Definir que App preserva dados localmente e restaura após re-login |
| L-9 | **Limite do campo `recipientId` não especificado** | Estouro de Campos | Média | Definir tamanho máximo e exibir indicador de limite |
| L-10 | **Tipo do campo `amount` no App não especificado** | Estouro de Campos | Média | Especificar campo numérico nativo (teclado numérico em mobile) |
| L-11 | **Empty state da lista de transações não definido** | Campos Obrigatórios | Baixa | Definir mensagem e call-to-action para lista vazia |

---

## 6. Recomendações por Prioridade

### Para o MVP (inegociável)

| # | Recomendação | Justificativa |
|---|-------------|---------------|
| R-1 | Especificar indicadores visuais de campos obrigatórios (asterisco + aria-required) | Elegância fundamental: o usuário deve saber o que preencher antes de tentar submeter |
| R-2 | Definir que botão "Enviar" fica desabilitado com campos vazios/inválidos | Previne submissões prematuras e erros 400 desnecessários na API |
| R-3 | Especificar preservação dos dados do formulário após erro da API | Evitar retrabalho do usuário é essencial para percepção de qualidade |
| R-4 | Definir forma de exibição das mensagens de erro (inline vs. toast vs. banner) | Sem isso, cada plataforma (iOS/Android) pode implementar de forma diferente |
| R-5 | Especificar tipo do campo `amount` como numérico (teclado numérico em mobile) | Previne entrada de letras e melhora a UX em dispositivos touch |

### Para o Backlog

| # | Recomendação |
|---|-------------|
| R-6 | Definir ciclo de vida da Idempotency-Key: nova key gerada ao alterar dados do formulário após tentativa |
| R-7 | Especificar comportamento de navegação pós-sucesso (redirecionar para lista de recentes) |
| R-8 | Definir comportamento de preservação de estado do formulário após expiração de sessão |
| R-9 | Adicionar hint de range "Entre 1 e 10.000 QualiPoints" abaixo do campo `amount` |
| R-10 | Especificar empty state da lista de transações com mensagem orientativa e call-to-action |

---

## 7. Heurísticas Complementares Recomendadas

### Input Method
**Conexão com Chique:** A elegância dos campos obrigatórios depende do suporte a diferentes métodos de entrada — um campo `amount` que não aceita valor colado ou que rejeita "10,000" (com vírgula de milhar) quebra a experiência Chique mesmo que o indicador visual esteja correto.

**Recomendação:** Aplicar **Input Method** para garantir que digitação, copy/paste e interface de teclado numérico sejam tratados de forma consistente nos campos do formulário de envio.

### Seen and Heard
**Conexão com Chique:** Os estados de interrupção (Processando, Concluído, Falhou) precisam ser vistos e ouvidos — não apenas existir no requisito. A heurística Seen and Heard complementa Chique com foco em acessibilidade, feedback multicanal e clareza das mensagens.

**Recomendação:** Aplicar **Seen and Heard** para garantir que os indicadores de progresso e mensagens de erro atendam a usuários com deficiências visuais e motoras, e que o feedback seja perceptível em diferentes contextos de uso (em movimento, em ambiente barulhento, etc.).

---

## Referências

- **Requisito analisado:** [REQ_INICIAL_V2.md](../requisito-revisado/REQ_INICIAL_V2.md)
- **Skill aplicada:** [Chique.md](../../Skill/Cap07_refinamento/Chique.md)
- **Análises complementares já realizadas:** [COUNT_REQ_INICIAL_V2.md](./COUNT_REQ_INICIAL_V2.md), [CRUD_REQ_INICIAL_V2.md](./CRUD_REQ_INICIAL_V2.md)
- **Próximas heurísticas recomendadas:** Input Method, Seen and Heard (ver seção 7)
