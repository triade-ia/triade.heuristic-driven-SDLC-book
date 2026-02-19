---
name: test-component-guide
description: Guia para implementar testes de componentes UI. Use quando tiver relatório de estratégia (output/test-strategy-*.md) com casos de componente (COMP-XXX) e precisar gerar testes para formulários, estados visuais e acessibilidade em TypeScript (React Testing Library) ou Java.
---

# Guia de Testes de Componentes (UI)

Implemente testes de componentes com base no **relatório de estratégia** gerado por [TEST_STRATEGY.md](TEST_STRATEGY.md). Use os casos relacionados a UI (prefixo COMP-XXX ou seção "Testes de Componentes") para validar renderização, interações e acessibilidade.

## Quando Usar Este Guide

- Relatório inclui testes de componentes (formulários, estados, feedback)
- Precisa testar: renderização, campos obrigatórios, habilitar/desabilitar, loading, erro, sucesso, ARIA
- Heurísticas: Chique, SeenAndHeard, InputMethod

## Definição e Escopo

Testes de componentes validam **componentes de UI isoladamente**, sem aplicação completa no navegador.

**Características**: Renderização isolada, dependências mockadas, 100–500 ms por teste, interações simuladas (userEvent).

**Quando usar**: Formulários, botões, estados (loading/erro/sucesso), acessibilidade (ARIA, teclado).

**Quando NÃO usar**: Lógica pura (unitários); fluxo completo no browser (use [TEST_E2E_GUIDE.md](TEST_E2E_GUIDE.md)).

## Como Usar o Relatório

1. Receba o relatório: `@TEST_COMPONENT_GUIDE.md @output/test-strategy-xxx.md`
2. Localize casos de componentes (formulário, página)
3. Para cada caso: renderizar componente, simular interação (type, click), assert (texto, atributos ARIA, disabled/enabled)
4. Priorize: Chique (obrigatório, habilitar/desabilitar), SeenAndHeard (feedback, acessibilidade), InputMethod (typing)

## Frameworks

**TypeScript**: React Testing Library, Jest, @testing-library/user-event.  
**Java**: TestFX (JavaFX), AssertJ Swing (Swing). Testes de componente web são mais comuns em TS.

## Exemplo: Formulário de Transferência (Chique + SeenAndHeard)

```typescript
import { render, screen, waitFor } from '@testing-library/react';
import userEvent from '@testing-library/user-event';
import { TransferForm } from '../../src/components/TransferForm';

describe('TransferForm - Componente (Chique + SeenAndHeard)', () => {
  describe('Renderização inicial (Chique)', () => {
    it('deve renderizar campos com indicação de obrigatório', () => {
      render(<TransferForm onSubmit={jest.fn()} />);
      expect(screen.getByLabelText(/Destinatário/)).toBeInTheDocument();
      expect(screen.getByLabelText(/Valor/)).toHaveAttribute('aria-required', 'true');
    });
    it('botão deve estar desabilitado quando campos vazios', () => {
      render(<TransferForm onSubmit={jest.fn()} />);
      expect(screen.getByRole('button', { name: /Transferir/ })).toBeDisabled();
    });
  });

  describe('Interação (InputMethod - Typing)', () => {
    it('deve habilitar botão quando campos preenchidos', async () => {
      const user = userEvent.setup();
      render(<TransferForm onSubmit={jest.fn()} />);
      await user.type(screen.getByLabelText(/Destinatário/), 'user123');
      await user.type(screen.getByLabelText(/Valor/), '500');
      expect(screen.getByRole('button', { name: /Transferir/ })).toBeEnabled();
    });
  });

  describe('Loading (SeenAndHeard)', () => {
    it('deve mostrar estado de loading durante submissão', async () => {
      const user = userEvent.setup();
      const slowSubmit = jest.fn(() => new Promise(r => setTimeout(r, 100)));
      render(<TransferForm onSubmit={slowSubmit} />);
      await user.type(screen.getByLabelText(/Destinatário/), 'user123');
      await user.type(screen.getByLabelText(/Valor/), '500');
      await user.click(screen.getByRole('button', { name: /Transferir/ }));
      expect(screen.getByRole('button', { name: /Transferindo/ })).toBeInTheDocument();
      expect(screen.getByRole('button')).toHaveAttribute('aria-busy', 'true');
    });
  });

  describe('Mensagens de erro (SeenAndHeard + FAILURE)', () => {
    it('deve exibir mensagem de erro clara', async () => {
      const user = userEvent.setup();
      const failSubmit = jest.fn(() => Promise.reject(new Error('Saldo insuficiente')));
      render(<TransferForm onSubmit={failSubmit} />);
      await user.type(screen.getByLabelText(/Destinatário/), 'user123');
      await user.type(screen.getByLabelText(/Valor/), '500');
      await user.click(screen.getByRole('button', { name: /Transferir/ }));
      await waitFor(() => {
        expect(screen.getByRole('alert')).toHaveTextContent('Saldo insuficiente');
        expect(screen.getByRole('alert')).toHaveAttribute('aria-live', 'polite');
      });
      expect(screen.getByRole('button', { name: /Transferir/ })).toBeEnabled();
    });
  });

  describe('Sucesso (SeenAndHeard)', () => {
    it('deve exibir mensagem de sucesso e limpar campos', async () => {
      const user = userEvent.setup();
      render(<TransferForm onSubmit={jest.fn().mockResolvedValue(undefined)} />);
      await user.type(screen.getByLabelText(/Destinatário/), 'user123');
      await user.type(screen.getByLabelText(/Valor/), '500');
      await user.click(screen.getByRole('button', { name: /Transferir/ }));
      await waitFor(() => {
        expect(screen.getByRole('status')).toHaveTextContent(/sucesso/i);
      });
      expect(screen.getByLabelText(/Destinatário/)).toHaveValue('');
    });
  });

  describe('Acessibilidade (SeenAndHeard)', () => {
    it('formulário deve ter label apropriado', () => {
      render(<TransferForm onSubmit={jest.fn()} />);
      expect(screen.getByRole('form', { name: /Formulário de transferência/ })).toBeInTheDocument();
    });
  });
});
```

## Integração com Heurísticas

- **Chique**: Campos obrigatórios (*, aria-required), botão habilitado/desabilitado conforme estado, estouro de campos
- **SeenAndHeard**: role="alert" para erro, role="status" para loading/sucesso, aria-live="polite", aria-busy, labels
- **InputMethod**: userEvent.type, userEvent.click, userEvent.tab (navegação por teclado)

## Para QAs

Casos de componente no relatório: renderização, estados (vazio, loading, erro, sucesso), interações, acessibilidade (ARIA, foco).

## Checklist

- [ ] Renderização e labels corretos?
- [ ] Campos obrigatórios indicados (Chique)?
- [ ] Botão habilitado/desabilitado conforme regra?
- [ ] Loading visível durante submissão?
- [ ] Mensagens de erro e sucesso claras e acessíveis (role="alert", role="status")?
- [ ] ARIA (aria-required, aria-live, aria-busy) quando aplicável?
- [ ] Interações (type, click) cobertas?

## Referências

- Estratégia: [TEST_STRATEGY.md](TEST_STRATEGY.md)
- Unitários: [TEST_UNIT_GUIDE.md](TEST_UNIT_GUIDE.md)
- E2E: [TEST_E2E_GUIDE.md](TEST_E2E_GUIDE.md)
- Chique: [../../Cap07_refinamento/Chique.md](../../Cap07_refinamento/Chique.md)
- SeenAndHeard: [../../Cap07_refinamento/SeenAndHeard.md](../../Cap07_refinamento/SeenAndHeard.md)
