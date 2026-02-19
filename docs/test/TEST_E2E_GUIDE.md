---
name: test-e2e-guide
description: Guia para implementar testes E2E. Use quando tiver relatório de estratégia (output/test-strategy-*.md) com casos E2E-XXX e precisar gerar testes de fluxo completo no navegador (Playwright ou Selenium) em TypeScript ou Java.
---

# Guia de Testes E2E (End-to-End)

Implemente testes E2E com base no **relatório de estratégia** gerado por [TEST_STRATEGY.md](TEST_STRATEGY.md). Use os casos com prefixo **E2E-XXX** para gerar testes de fluxo completo (UI → API → banco) no navegador.

## Quando Usar Este Guide

- Relatório contém casos E2E (fluxos críticos de negócio)
- Precisa testar: login → ação → resultado, mensagens de sucesso/erro, recuperação, duplo envio
- Heurísticas: Chique, SeenAndHeard, FAILURE (Emotions, Recovery)

## Definição e Escopo

Testes E2E validam **fluxos completos** em ambiente real (navegador, backend, banco).

**Características**: Browser real, 5–30 s por teste, focar em jornadas críticas (poucos testes).

**Quando usar**: Login, compra, pagamento, transferência — fluxos que precisam estar sempre estáveis.

**Quando NÃO usar**: Casos de borda (unitários/integração); validações de input (componentes).

## Como Usar o Relatório

1. Receba o relatório: `@TEST_E2E_GUIDE.md @output/test-strategy-xxx.md`
2. Localize "Testes E2E" e casos E2E-XXX
3. Para cada caso: navegar, preencher, clicar, aguardar feedback, assert URL e conteúdo
4. Inclua setup (ex.: login) em beforeEach

## Frameworks

**TypeScript**: Playwright (recomendado), Cypress.  
**Java**: Selenium WebDriver, Selenide, Playwright for Java.

## Exemplo: Fluxo de Transferência (Playwright)

```typescript
import { test, expect } from '@playwright/test';

test.describe('Fluxo de Transferência E2E', () => {
  test.beforeEach(async ({ page }) => {
    await page.goto('http://localhost:3000/login');
    await page.fill('input[name="email"]', 'sender@test.com');
    await page.fill('input[name="password"]', 'password123');
    await page.click('button[type="submit"]');
    await page.waitForURL('**/dashboard');
  });

  test('deve completar transferência com sucesso (fluxo crítico)', async ({ page }) => {
    await page.click('a[href="/transfer"]');
    await expect(page).toHaveURL(/.*transfer/);
    await page.fill('input[name="recipient_id"]', 'recipient@test.com');
    await page.fill('input[name="amount"]', '500');
    const submitButton = page.locator('button[type="submit"]');
    await expect(submitButton).toBeEnabled();
    await submitButton.click();
    await expect(page.locator('[role="status"]')).toContainText(/Processando/i);
    await expect(page.locator('[role="alert"]')).toContainText(/sucesso/i, { timeout: 10000 });
    await expect(page).toHaveURL(/.*transactions/);
    const firstRow = page.locator('table tr').first();
    await expect(firstRow).toContainText('recipient@test.com');
    await expect(firstRow).toContainText('500');
    await expect(firstRow).toContainText('Concluída');
  });

  test('deve exibir erro e permitir correção (FAILURE - Recovery + Emotions)', async ({ page }) => {
    await page.goto('http://localhost:3000/transfer');
    await page.fill('input[name="recipient_id"]', 'recipient@test.com');
    await page.fill('input[name="amount"]', '0');
    await page.click('button[type="submit"]');
    await expect(page.locator('[role="alert"]')).toContainText(/maior que 0/i);
    await expect(page.locator('input[name="recipient_id"]')).toHaveValue('recipient@test.com');
    await page.fill('input[name="amount"]', '500');
    await page.click('button[type="submit"]');
    await expect(page.locator('[role="alert"]')).toContainText(/sucesso/i);
  });

  test('deve prevenir duplo envio durante processamento (Chique)', async ({ page }) => {
    await page.goto('http://localhost:3000/transfer');
    await page.fill('input[name="recipient_id"]', 'recipient@test.com');
    await page.fill('input[name="amount"]', '500');
    const submitButton = page.locator('button[type="submit"]');
    await submitButton.click();
    await submitButton.click();
    await expect(submitButton).toBeDisabled();
    await expect(page.locator('[role="alert"]')).toContainText(/sucesso/i, { timeout: 10000 });
    await page.goto('http://localhost:3000/transactions');
    await expect(page.locator('table tbody tr')).toHaveCount(1);
  });
});
```

## Integração com Heurísticas

- **Chique**: Botão habilitado só quando válido; desabilitado durante envio; evitar duplo envio
- **SeenAndHeard**: Loading visível, mensagem de sucesso/erro clara, roles (status, alert)
- **FAILURE - Emotions**: Mensagens não culpabilizam usuário; tom claro
- **FAILURE - Recovery**: Campos preservados após erro; usuário pode corrigir e reenviar

## Para QAs

Casos E2E no relatório: pré-condições (login), passos (navegar, preencher, submeter), resultado esperado (URL, texto, quantidade de registros).

## Checklist

- [ ] Apenas fluxos críticos (poucos testes)?
- [ ] Setup de login/sessão em beforeEach?
- [ ] Feedback de loading e sucesso/erro verificados?
- [ ] Recuperação (corrigir e reenviar) testada?
- [ ] Duplo envio evitado (botão desabilitado)?
- [ ] Cada E2E-XXX do relatório implementado?
- [ ] Timeout adequado para operações assíncronas?

## Referências

- Estratégia: [TEST_STRATEGY.md](TEST_STRATEGY.md)
- Componentes: [TEST_COMPONENT_GUIDE.md](TEST_COMPONENT_GUIDE.md)
- Serviço: [TEST_SERVICE_GUIDE.md](TEST_SERVICE_GUIDE.md)
- Chique: [../../Cap07_refinamento/Chique.md](../../Cap07_refinamento/Chique.md)
- FAILURE: [../../Cap08_desenvolvimento/FAILURE.md](../../Cap08_desenvolvimento/FAILURE.md)
