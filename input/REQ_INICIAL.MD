# **Requisito**: Implementar funcionalidade de envio de "QualiPoints" entre usuários.

**Descrição**: O usuário quer poder mandar pontos para um amigo. O sistema deve ter uma tela para isso, uma API que processa e funcionar no App. Tem que ser rápido e seguro.

## 📄 Contexto do Projeto: Sistema de Transferência "QualiPoints"

### 1. Visão Geral do Requisito

O sistema deve permitir que usuários enviem pontos entre si através de uma carteira digital. A operação deve ser feita via aplicativo mobile e confirmada via painel web. Os dados residem em um banco de dados relacional e a comunicação é via REST API.

### 2. Regras de Negócio (Lacunares - propositalmente incompletas)

- O usuário deve estar logado para enviar.
- É necessário informar o ID do destinatário e o valor.
- O saldo deve ser atualizado após o envio.
- Deve haver uma lista de transações recentes.