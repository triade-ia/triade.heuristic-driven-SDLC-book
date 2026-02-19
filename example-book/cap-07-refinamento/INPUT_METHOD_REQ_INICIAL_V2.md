# Relatório: Input Method (Método de Entrada) — REQ_INICIAL_V2 (QualiPoints)

**Requisito analisado:** Envio de QualiPoints entre usuários (REQ_INICIAL_V2).
**Skill aplicada:** Cap07_refinamento — Input Method (Método de Entrada)
**Data:** 2026-02-18
**Status:** Análise de refinamento concluída

---

## 1. Métodos de Entrada Identificados no Requisito

Antes de aplicar os aspectos Input Method, mapeamos todos os pontos de entrada de dados presentes ou implícitos no requisito:

| Ponto de Entrada | Canal | Método de Entrada Provável | Especificado no Req |
|------------------|-------|---------------------------|---------------------|
| **Campo `recipientId`** | App mobile | Digitação, cópia de ID de contato | Não especificado |
| **Campo `amount`** | App mobile | Digitação numérica, cópia | Não especificado |
| **Header `Idempotency-Key`** | API | Gerado automaticamente pelo App | Implícito (UUID gerado no App) |
| **Header `Authorization`** | API / App | Token obtido via autenticação | Definido (Bearer token) |
| **Header `Content-Type`** | API | Definido automaticamente pelo App | Definido (application/json) |
| **Validações de entrada** | API (server-side) | N/A — validadas na API | Bem definidas (seção 6.1 e 10.4) |
| **Lista de transações** | App / Painel web | Consulta (sem entrada de dados) | Definida |

---

## 2. Análise por Método de Entrada

### 2.1 Typing (Digitação)

#### O que está definido

- O campo `amount` deve ser inteiro (seção 4.1), com valores entre 1 e 10.000 (seção 4.2).
- O campo `recipientId` deve ser um ID de usuário válido (seção 3.2).
- A API valida todas as entradas no servidor (seção 6.1).
- A seção 11.2 mapeia mensagens de erro derivadas de entradas inválidas.

#### Lacunas identificadas

| Questão Input Method | Análise |
|----------------------|---------|
| O campo `amount` exibe teclado numérico em mobile? | **Lacuna:** o tipo de campo não está especificado; sem `inputmode="numeric"` ou campo `type="number"`, usuários mobile verão teclado alfanumérico para um campo que só aceita inteiros |
| Há validação em tempo real durante a digitação do `amount`? | **Lacuna:** o requisito não define se a validação de range ocorre ao digitar, ao perder foco (blur) ou apenas no submit |
| O campo `recipientId` tem auto-completar ou sugestões? | **Não especificado** — sem histórico de destinatários ou busca por nome, o usuário precisa conhecer o ID exato do destinatário |
| Há máscara ou formatação automática para `amount`? | **Lacuna:** o requisito não define se "1000" é exibido como "1.000" (com separador de milhar) durante a digitação |
| Campos numéricos têm atalhos de teclado (Tab entre campos)? | **Não especificado** — navegação por teclado entre `recipientId` e `amount` não está definida |
| O campo `recipientId` tem limite de caracteres visível? | **Lacuna:** o tipo e tamanho do ID (UUID de 36 chars? ID numérico? username?) não estão definidos no requisito |
| Há suporte a múltiplos idiomas no campo `recipientId`? | **Risco baixo:** IDs provavelmente são alfanuméricos ASCII, mas a especificação não define charset aceito |

**Risco principal:** Um campo `amount` sem teclado numérico nativo em mobile aumenta o atrito de entrada e o risco de digitação de letras acidentais — que a API rejeita com 400, sem que o usuário compreenda o motivo.

**Recomendação:** Especificar que:
- Campo `amount` usa `inputmode="numeric"` ou `type="number"` (teclado numérico em mobile).
- Validação de range 1–10.000 ocorre ao perder foco (blur), com mensagem inline.
- O campo exibe hint de range "Entre 1 e 10.000 QualiPoints" abaixo do input.
- `recipientId` tem tamanho máximo definido correspondente ao formato do ID do sistema (ex.: 36 chars para UUID).

---

### 2.2 Copy/Paste (Copiar/Colar)

#### O que está definido

- O requisito não aborda copy/paste explicitamente.
- O `recipientId` é definido como "ID do usuário destinatário" — em sistemas reais, usuários frequentemente copiam este ID de perfis, e-mails ou chats.

#### Lacunas identificadas

| Questão Input Method | Análise |
|----------------------|---------|
| O campo `amount` aceita valores colados com separador de milhar? | **Lacuna:** "1.000" ou "1,000" colados resultariam em `NaN` ou zero em conversão para inteiro sem normalização |
| Espaços extras no `recipientId` colado são removidos automaticamente? | **Lacuna:** " user-uuid-123 " (com espaços) passaria na validação do App mas poderia falhar na API por ID inválido |
| O sistema sanitiza dados colados contra XSS/injection? | **Lacuna:** o requisito menciona segurança (HTTPS, autenticação), mas não especifica sanitização de inputs colados |
| Dados colados são validados da mesma forma que dados digitados? | **Parcialmente implícito** — a validação server-side cobre o caso, mas validação client-side para dados colados não está definida |
| Há feedback visual ao colar um valor? | **Não especificado** |

**Risco de usabilidade:** Um usuário que copia "R$ 1.000" de um e-mail e cola no campo `amount` verá um comportamento inesperado (valor inválido) sem entender por quê. Similarmente, copiar um ID com espaços ou quebra de linha gerará erro 404 (`RECIPIENT_NOT_FOUND`) — confundindo o usuário.

**Recomendação:** Especificar que:
- Ao colar no campo `amount`, o App normaliza o valor: remove caracteres não numéricos (`.`, `,`, `R$`, espaços) e converte para inteiro.
- Ao colar no campo `recipientId`, o App realiza trim automático (remove espaços no início e no fim) antes de submeter.
- Dados colados disparam a mesma validação de blur que dados digitados.

---

### 2.3 Import (Importação de Dados)

#### O que está definido

- O requisito não define funcionalidade de importação.
- O MVP é de transferência individual: 1 remetente → 1 destinatário por operação (seção 2.5 — Destinatários por Transferência).

#### Análise

A importação de dados em massa (ex.: transferências em lote via CSV) está **fora do escopo do MVP** e alinhada com o princípio "Eliminate" da seção 7.2. Não há lacunas críticas para o MVP neste aspecto.

**Risco de backlog:** À medida que o produto cresce (ex.: departamento RH distribuindo pontos para N colaboradores), a ausência de importação em lote forçará operações manuais repetitivas. Este ponto deve constar no backlog com formato de arquivo, template e comportamento de importação parcial definidos quando houver demanda.

**Recomendação para backlog:** Quando a funcionalidade de importação for implementada, definir:
- Formato suportado: CSV com colunas `recipientId,amount`.
- Preview dos dados antes de confirmar.
- Relatório de erros por linha com código e mensagem.
- Comportamento em falha parcial (importar válidos, reportar inválidos).

---

### 2.4 Drag/Drop (Arrastar e Soltar)

#### O que está definido

- Não há referência a drag/drop no requisito.

#### Análise

Drag/drop **não é aplicável** ao fluxo de envio de QualiPoints (formulário com dois campos de texto/número). Nenhuma lacuna ou risco identificado para o MVP.

**Contexto de relevância futura:** Drag/drop poderia ser relevante em uma funcionalidade de upload de comprovante ou organização de transações favoritas — fora do escopo atual.

---

### 2.5 Various Interfaces (GUI vs. API)

#### O que está definido

- A seção 6.1 define explicitamente: "Toda validação de negócio é feita no servidor (API)."
- A seção 6.1 complementa: "O App pode exibir feedback de UX, mas a fonte da verdade é a API."
- O contrato da API está bem documentado (seção 10): endpoint, headers, body, respostas.
- A validação é centralizada na API — tanto App quanto consumidores diretos da API passam pelas mesmas regras.

#### Lacunas identificadas

| Questão Input Method | Análise |
|----------------------|---------|
| A validação é consistente entre GUI e API? | **Bem definido** — toda validação de negócio está na API; o App é apenas um canal de entrada |
| As mensagens de erro da API são claras para desenvolvedores? | **Bem definido** — seção 10.4 mapeia códigos HTTP e bodies de erro específicos |
| Há rate limiting para a API? | **Lacuna:** o requisito menciona rate limit como backlog (seção 4.3), mas sem definição de quando e como será implementado |
| Há versionamento da API? | **Parcialmente definido** — o endpoint usa `/api/v1/transfers`, indicando versionamento, mas a política de versionamento (compatibilidade retroativa, deprecação) não está especificada |
| Autenticação é consistente entre GUI e API? | **Bem definido** — Bearer token usado em ambos os casos |
| A API valida o `Content-Type`? | **Lacuna menor:** o header `Content-Type: application/json` é listado como obrigatório, mas o comportamento quando omitido ou incorreto não está definido (HTTP 415 Unsupported Media Type?) |

**Ponto positivo:** O design de ter toda a validação server-side é um padrão correto e robusto. A consistência GUI/API está garantida por arquitetura.

**Risco do versionamento:** Sem política de versionamento definida, uma mudança futura no contrato da API pode quebrar clientes que consomem `/api/v1/transfers` diretamente (integrações externas, parceiros).

**Recomendação:** Especificar que:
- A API retorna HTTP 415 (Unsupported Media Type) quando `Content-Type` está ausente ou incorreto.
- Implementar rate limiting na camada de API Gateway (mesmo que seja backlog, definir a política agora).
- Documentar a política de versionamento: `/api/v1/` permanece estável por N meses antes de deprecação anunciada.

---

### 2.6 Tolerância a Erros e Formatos Flexíveis

#### O que está definido

- O campo `amount` deve ser inteiro (seção 4.1) — sem variações de formato aceitas.
- A API rejeita valores zero, negativos e acima de 10.000 com HTTP 400 (seção 4.2 e 10.4).
- Valores fora do range retornam `INVALID_AMOUNT` sem distinção entre "zero", "negativo" e "acima do máximo".

#### Lacunas identificadas

| Questão Input Method | Análise |
|----------------------|---------|
| "1.000" (com ponto) é aceito como 1000? | **Lacuna:** o requisito não define se separadores de milhar são aceitos ou normalizados |
| "1,000" (com vírgula — formato brasileiro) é aceito? | **Lacuna:** o campo inteiro não especifica como lidar com formatação localizada |
| Espaços no início/fim do `recipientId` são tolerados? | **Lacuna:** "  user-uuid-123  " levaria a busca por ID inválido na base |
| `recipientId` com letras maiúsculas/minúsculas é normalizado? | **Lacuna:** se o ID for case-sensitive (UUID padrão é case-insensitive mas depende da implementação), "USER-UUID-123" pode falhar |
| `amount` negativo retorna mensagem diferente de `amount` acima do máximo? | **Lacuna leve:** ambos retornam `INVALID_AMOUNT`, sem distinção que permitiria feedback mais preciso ao usuário |
| O sistema sugere correção para formatos próximos? | **Não especificado** — sem autocorreção ou sugestão de formato |
| Quebra de linha no `recipientId` colado é tratada? | **Lacuna:** `\n` ao final de um ID copiado pode resultar em falha de lookup |

**Risco de usabilidade:** A falta de normalização de formatos de entrada coloca a carga de formatação correta no usuário. Em um sistema financeiro com entrada de valores monetários, usuários brasileiros frequentemente usam "1.000" ou "1,000" para representar mil unidades — que a API interpretaria como inválido sem normalização.

**Recomendação:** Especificar que:
- O App normaliza o valor de `amount` antes de submeter: remove pontos e vírgulas de milhar, converte para inteiro. Se o valor resultante for inválido, exibe erro.
- O App realiza trim no `recipientId` (remove espaços, quebras de linha, caracteres de controle) antes de submeter.
- A mensagem `INVALID_AMOUNT` é diferenciada no App por faixas: "Valor deve ser ao menos 1" (para zero/negativo) vs. "Valor não pode exceder 10.000" (para valores acima do máximo).

---

## 3. Bugs de Entrada Identificados

| # | Bug Potencial | Método de Entrada | Severidade | Origem |
|---|---------------|-------------------|------------|--------|
| B-1 | **Campo `amount` sem teclado numérico** em mobile permite digitação de letras, gerando erro 400 | Typing | Alta | Tipo de campo não especificado |
| B-2 | **"1.000" colado no `amount`** é interpretado como inválido por ter separador de milhar | Copy/Paste | Alta | Normalização de formato não especificada |
| B-3 | **Espaços colados no `recipientId`** geram erro 404 por ID inválido, confundindo o usuário | Copy/Paste | Alta | Trim automático não especificado |
| B-4 | **Quebra de linha no `recipientId`** (ID copiado com `\n`) falha silenciosamente na busca | Copy/Paste | Média | Sanitização de caracteres de controle não especificada |
| B-5 | **`INVALID_AMOUNT` para zero e para 10001** retornam a mesma mensagem, sem orientação de correção específica | Typing | Média | Código de erro não diferencia tipos de violação |
| B-6 | **Sem rate limiting** na API permite envio de múltiplas requisições em loop (ex.: por bug no App) | API (Various Interfaces) | Média (backlog) | Rate limiting não definido |
| B-7 | **`Content-Type` ausente** na requisição à API não tem comportamento definido (415? 400?) | API (Various Interfaces) | Baixa | Resposta para header ausente não especificada |

---

## 4. Cenários de Teste Derivados da Heurística Input Method

### 4.1 Typing (Digitação)

```typescript
// React Testing Library
it('campo amount deve exibir teclado numérico (inputmode) (Input Method - Typing)', () => {
  render(<TransferForm />);

  const amountInput = screen.getByLabelText(/Valor/);
  expect(amountInput).toHaveAttribute('inputmode', 'numeric');
});

it('deve validar range ao perder foco do campo amount (Input Method - Typing)', async () => {
  const user = userEvent.setup();
  render(<TransferForm />);

  const amountInput = screen.getByLabelText(/Valor/);
  await user.type(amountInput, '0');
  await user.tab(); // blur

  expect(screen.getByText(/Valor deve ser ao menos 1 QualiPoint/)).toBeInTheDocument();
});

it('deve aceitar digitação de valor máximo 10000 sem erro (Input Method - Typing)', async () => {
  const user = userEvent.setup();
  render(<TransferForm />);

  await user.type(screen.getByLabelText(/Valor/), '10000');
  await user.tab();

  expect(screen.queryByText(/inválido|erro|obrigatório/i)).not.toBeInTheDocument();
});
```

### 4.2 Copy/Paste

```typescript
it('deve normalizar valor colado com separador de milhar (Input Method - Copy/Paste)', async () => {
  const user = userEvent.setup();
  render(<TransferForm />);

  const amountInput = screen.getByLabelText(/Valor/);
  await user.click(amountInput);
  await user.paste('1.000');

  expect(amountInput).toHaveValue(1000); // Normalizado para inteiro
});

it('deve fazer trim de espaços no recipientId colado (Input Method - Copy/Paste)', async () => {
  const user = userEvent.setup();
  render(<TransferForm />);

  const recipientInput = screen.getByLabelText(/Destinatário/);
  await user.click(recipientInput);
  await user.paste('  user-uuid-123  ');

  expect(recipientInput).toHaveValue('user-uuid-123'); // Sem espaços
});

it('deve remover quebra de linha do recipientId colado (Input Method - Copy/Paste)', async () => {
  const user = userEvent.setup();
  render(<TransferForm />);

  const recipientInput = screen.getByLabelText(/Destinatário/);
  await user.click(recipientInput);
  await user.paste('user-uuid-123\n');

  expect(recipientInput).toHaveValue('user-uuid-123');
});

it('dados colados devem disparar validação (Input Method - Copy/Paste)', async () => {
  const user = userEvent.setup();
  render(<TransferForm />);

  const amountInput = screen.getByLabelText(/Valor/);
  await user.click(amountInput);
  await user.paste('99999');
  await user.tab();

  expect(screen.getByText(/não pode exceder 10\.000/i)).toBeInTheDocument();
});
```

### 4.3 API (Various Interfaces)

```typescript
// Serviço — testes de API direta
it('deve retornar 400 para amount negativo (Input Method - API)', async () => {
  const response = await post('/api/v1/transfers', {
    recipientId: 'user-2',
    amount: -1,
  }, { headers: { Authorization: 'Bearer token', 'Idempotency-Key': uuid() } });

  expect(response.status).toBe(400);
  expect(response.body.code).toBe('INVALID_AMOUNT');
});

it('deve retornar 400 para amount string (Input Method - API)', async () => {
  const response = await post('/api/v1/transfers', {
    recipientId: 'user-2',
    amount: 'mil',
  }, { headers: { Authorization: 'Bearer token', 'Idempotency-Key': uuid() } });

  expect(response.status).toBe(400);
});

it('deve retornar 415 quando Content-Type ausente (Input Method - API)', async () => {
  const response = await postRaw('/api/v1/transfers',
    JSON.stringify({ recipientId: 'user-2', amount: 100 }),
    { headers: { Authorization: 'Bearer token', 'Idempotency-Key': uuid() } }
    // Content-Type omitido
  );

  expect(response.status).toBe(415);
});

it('API e GUI devem rejeitar amount=0 com o mesmo código (Input Method - Consistência GUI/API)', async () => {
  // Via API direta
  const apiResponse = await post('/api/v1/transfers', { recipientId: 'user-2', amount: 0 });
  expect(apiResponse.body.code).toBe('INVALID_AMOUNT');

  // Via GUI (validação client-side)
  const user = userEvent.setup();
  render(<TransferForm />);
  await user.type(screen.getByLabelText(/Valor/), '0');
  await user.tab();
  expect(screen.getByText(/Valor deve ser ao menos 1/)).toBeInTheDocument();
});
```

### 4.4 Tolerância a Erros e Formatos

```typescript
it('deve normalizar "1,000" colado para 1000 (Input Method - Tolerância)', async () => {
  const user = userEvent.setup();
  render(<TransferForm />);

  const amountInput = screen.getByLabelText(/Valor/);
  await user.click(amountInput);
  await user.paste('1,000'); // Formato com vírgula de milhar

  expect(amountInput).toHaveValue(1000);
});

it('mensagem de erro deve diferenciar zero de valor acima do máximo (Input Method - Tolerância)', async () => {
  const user = userEvent.setup();
  const { rerender } = render(<TransferForm />);

  // Zero
  await user.type(screen.getByLabelText(/Valor/), '0');
  await user.tab();
  expect(screen.getByText(/ao menos 1 QualiPoint/i)).toBeInTheDocument();

  // Acima do máximo
  rerender(<TransferForm />);
  await user.type(screen.getByLabelText(/Valor/), '10001');
  await user.tab();
  expect(screen.getByText(/não pode exceder 10\.000/i)).toBeInTheDocument();
});
```

**Java (Selenium):**

```java
@Test
@DisplayName("Deve normalizar valor com ponto de milhar ao colar")
void shouldNormalizeThousandSeparatorOnPaste() {
    driver.get(BASE_URL + "/transfer");

    WebElement amountInput = driver.findElement(By.name("amount"));
    amountInput.click();

    // Simular colar via JavaScript
    ((JavascriptExecutor) driver).executeScript(
        "arguments[0].value = '1.000'; arguments[0].dispatchEvent(new Event('paste'));",
        amountInput
    );

    // Verificar normalização
    assertEquals("1000", amountInput.getAttribute("value"));
}

@Test
@DisplayName("Deve remover espaços do recipientId ao colar")
void shouldTrimRecipientIdOnPaste() {
    driver.get(BASE_URL + "/transfer");

    WebElement recipientInput = driver.findElement(By.name("recipient_id"));
    recipientInput.click();

    ((JavascriptExecutor) driver).executeScript(
        "arguments[0].value = '  user-uuid-123  '; " +
        "arguments[0].dispatchEvent(new Event('paste')); " +
        "arguments[0].dispatchEvent(new Event('blur'));",
        recipientInput
    );

    assertEquals("user-uuid-123", recipientInput.getAttribute("value"));
}

@Test
@DisplayName("API deve retornar 415 sem Content-Type")
void apiShouldReturn415WithoutContentType() throws Exception {
    HttpClient client = HttpClient.newHttpClient();
    HttpRequest request = HttpRequest.newBuilder()
        .uri(URI.create(BASE_URL + "/api/v1/transfers"))
        .POST(HttpRequest.BodyPublishers.ofString("{\"recipientId\":\"user-2\",\"amount\":100}"))
        .header("Authorization", "Bearer " + TOKEN)
        .header("Idempotency-Key", UUID.randomUUID().toString())
        // Content-Type omitido
        .build();

    HttpResponse<String> response = client.send(request, HttpResponse.BodyHandlers.ofString());
    assertEquals(415, response.statusCode());
}
```

---

## 5. Lacunas e Riscos Consolidados

| # | Lacuna / Risco | Método de Entrada | Severidade | Recomendação |
|---|----------------|-------------------|------------|--------------|
| L-1 | **Tipo do campo `amount` não especificado** — risco de teclado alfanumérico em mobile | Typing | Alta | Especificar `inputmode="numeric"` ou campo `type="number"` |
| L-2 | **Normalização de separadores de milhar** no `amount` não definida | Copy/Paste | Alta | Definir que App remove `.` e `,` de milhar antes de converter para inteiro |
| L-3 | **Trim automático do `recipientId`** não especificado | Copy/Paste | Alta | Especificar remoção de espaços, tabs e quebras de linha ao colar |
| L-4 | **Timing de validação client-side não definido** (blur vs. submit) | Typing | Alta | Especificar validação de range no blur para `amount` |
| L-5 | **`INVALID_AMOUNT` cobre zero, negativo e acima do máximo** sem distinção | Typing / Tolerância | Média | Diferenciar mensagem de UI: "ao menos 1" vs. "não pode exceder 10.000" |
| L-6 | **Comportamento do `Content-Type` ausente** não definido na API | API (Interfaces) | Média | Especificar HTTP 415 para Content-Type inválido/ausente |
| L-7 | **Rate limiting da API não definido** | API (Interfaces) | Média (backlog) | Definir política de rate limiting antes de produção |
| L-8 | **Auto-completar de destinatários não especificado** — usuário deve conhecer o ID exato | Typing | Média | Considerar histórico de destinatários recentes ou busca por nome para UX futura |
| L-9 | **Sanitização de XSS no `recipientId`** não explicitamente definida | Copy/Paste | Baixa | Especificar que o servidor sanitiza todos os inputs de string |
| L-10 | **Política de versionamento da API** não definida apesar do prefixo `/v1/` | API (Interfaces) | Baixa | Documentar política de compatibilidade retroativa e deprecação |

---

## 6. Recomendações por Prioridade

### Para o MVP (inegociável)

| # | Recomendação | Justificativa |
|---|-------------|---------------|
| R-1 | Especificar campo `amount` com `inputmode="numeric"` (teclado numérico em mobile) | Previne digitação acidental de letras e melhora UX touch |
| R-2 | Definir trim automático do `recipientId` (espaços, `\n`, `\r`) antes do submit | Previne erros 404 por IDs com caracteres invisíveis |
| R-3 | Definir normalização do `amount` ao colar: remover separadores de milhar | Usuários brasileiros frequentemente copiam valores formatados |
| R-4 | Especificar timing de validação: range do `amount` validado ao perder foco (blur) | Antecipa feedback sem aguardar chamada à API |
| R-5 | Diferenciar mensagem de UI para `INVALID_AMOUNT` por faixa de violação | Feedback mais preciso reduz tentativas de erro e tempo de correção |

### Para o Backlog

| # | Recomendação |
|---|-------------|
| R-6 | Implementar rate limiting por usuário na API (N req/minuto) com resposta HTTP 429 |
| R-7 | Adicionar histórico de destinatários recentes no campo `recipientId` (auto-completar) |
| R-8 | Definir funcionalidade de importação em lote (CSV) para transferências múltiplas |
| R-9 | Documentar política de versionamento da API (`/api/v1/` → deprecação e `/api/v2/`) |
| R-10 | Especificar resposta HTTP 415 explicitamente no contrato da API para `Content-Type` ausente/incorreto |

---

## 7. Heurísticas Complementares Recomendadas

### Chique
**Conexão com Input Method:** A elegância na digitação (Chique) e a eficiência na entrada de dados (Input Method) são complementares. Campos obrigatórios claramente indicados (Chique) orientam o usuário sobre o que digitar, enquanto Input Method garante que o que o usuário digita ou cola seja interpretado corretamente.

**Recomendação:** Aplicar **Chique** para garantir que os campos de entrada tenham indicadores visuais adequados de obrigatoriedade, validação no momento certo e preservação de contexto após erros.

### Baica (Boundary/Invalid/Null/Special)
**Conexão com Input Method:** Os métodos de entrada — especialmente copy/paste — são vetores comuns para valores de fronteira, nulos e caracteres especiais. Um valor colado pode conter caracteres que a digitação normal não permitiria.

**Recomendação:** Aplicar **Baica** para testar sistematicamente entradas como: `null`, string vazia `""`, caracteres de controle (`\0`, `\t`), valores inteiros limítrofes (`0`, `1`, `10000`, `10001`) e strings com apenas espaços em `recipientId`.

### VADER (Validação)
**Conexão com Input Method:** A heurística VADER complementa Input Method com foco específico em validação de dados de entrada. A consistência entre validação GUI e API (Various Interfaces) é um princípio central de VADER.

**Recomendação:** Aplicar **VADER** para garantir que todas as validações definidas no contrato da API (seção 10.4) tenham cobertura equivalente na camada de UI.

---

## Referências

- **Requisito analisado:** [REQ_INICIAL_V2.md](../requisito-revisado/REQ_INICIAL_V2.md)
- **Skill aplicada:** [InputMethod.md](../../Skill/Cap07_refinamento/InputMethod.md)
- **Análises complementares já realizadas:** [COUNT_REQ_INICIAL_V2.md](./COUNT_REQ_INICIAL_V2.md), [CRUD_REQ_INICIAL_V2.md](./CRUD_REQ_INICIAL_V2.md), [CHIQUE_REQ_INICIAL_V2.md](./CHIQUE_REQ_INICIAL_V2.md)
- **Próximas heurísticas recomendadas:** Chique, Baica, VADER (ver seção 7)
