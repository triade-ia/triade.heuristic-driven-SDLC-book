
# TEMPLATE: Caso de Teste (CT)

> Modelo e orientações para escrever um Caso de Teste (CT) claro e padronizado.

## Título

- Sempre iniciar o título com **Validar**, seguido da ação e do comportamento esperado.
	- Exemplo: **Validar registro de transação com condição de pagamento diferido**

## Pré-condições

[Detalhar todas as permissões, cadastros, place features, org features necessárias para a execução do caso de teste descrito no step by step de forma minuciosamente detalhada]

## Step by step

[Seguir o padrão de escrita de casos de teste do time de forma detalhada. Usar passos numerados, verbos no infinitivo e colchetes para elementos clicáveis.]

Exemplo (apagar depois):
1. Abrir o aplicativo
2. Navegar até [Menu Principal] >> [Submenu de operação]
3. Selecionar o item desejado
4. Clicar em [Próximo] >> Informar no campo 'Identificador' o valor "001"
5. Clicar em [Salvar] >> [Confirmar]
6. Selecionar a condição de pagamento desejada

> O resultado deve ser descrito na seção **Resultado esperado**; aqui informamos apenas os passos até o estado final antes da validação.

## Resultado esperado

[Detalhar todos os lugares que têm impacto com a execução desse caso de teste de forma minuciosamente detalhada]

---

## Regras para escrever um Caso de Teste (CT)

### 1. Estrutura e formatação

- Título: iniciar com **Validar**.
- Seções padrão: **Pré-condições**, **Step by step**, **Resultado esperado**.
- Usar Markdown ou identação clara para facilitar leitura em documentos, planilhas ou ferramenta de gestão.

### 2. Escrita do step by step

- Usar colchetes `[]` ao mencionar botões, opções ou menus clicáveis. Ex.: `Clicar em [Funcionalidade] >> [Ação]`.
- Usar aspas simples ou duplas para identificar nomes de tela, campos e labels. Ex.: no campo 'Identificador' informar "001".
- Evitar gerúndio — usar verbos no infinitivo: `Abrir`, `Clicar`, `Selecionar`, `Confirmar`.
- Evitar termos técnicos de código (ex.: funções, variáveis, JSON) — focar na ação visível do usuário.
- Descrever valores concretos usados no teste. Ex.: `Definir valor "R$ 50,00"`.
- Não deixar ambiguidade: cada passo deve ser reproduzível por outro membro do time.

### 3. Escrita do Resultado Esperado

- Iniciar com o estado final esperado (não o processo). Ex.: `O sistema deve exibir a mensagem 'Operação realizada com sucesso'`.
- Indicar onde validar o resultado (ex.: `Sistema de gestão → Seção de relatórios → Registros operacionais → Aba de transações`).
- Ser objetivo e mensurável para permitir avaliação clara de passou/falhou.

### 4. Clareza e consistência

- Usar sempre o mesmo padrão de verbos de ação: `Abrir`, `Clicar`, `Selecionar`, `Confirmar`, `Verificar`, `Aguardar`, `Validar`.
- Manter nomes de telas e menus idênticos aos exibidos no produto.
- Indicar contexto quando necessário (ex.: nível de validação, tipo de usuário, ambiente).

### 5. Complementos recomendados

- Título curto alternativo: versão curta para uso em boards ou planilhas.
- Alinhamento com automação: verificar se o CT reflete o comportamento validado pelos testes automatizados.
- Tags: incluir criticidade e tipo de teste quando aplicável (blocker, major, regression, smoke).
- Linguagem neutra e profissional; evitar abreviações informais.
- Observações ou recomendações extras devem ir em nota ao final, fora dos passos principais.

---

## Exemplo completo

[ID-001] - **Validar registro de transação com condição de pagamento diferido**

### Pré-condições
- No sistema administrativo: permissão "Registrar transações" ativada.
- Contexto/ambiente operacional configurado e ativo.
- Usuário operador com credenciais válidas e acesso ao sistema.
- Tipo de transação "Serviço antecipado" cadastrado no valor de R$ 150,00.
- Identificador de acesso (dispositivo ou cartão) vinculado a um usuário cliente válido.

### Step by step
1. Abrir o aplicativo e autenticar-se com as credenciais do operador.
2. Navegar até [Menu Operacional] >> [Funcionalidade de registro].
3. No campo 'Nome do cliente', registrar um novo cliente.
4. Retornar à tela inicial.
5. Acessar [Módulo de operações] >> [Registrar transação].
6. Selecionar o tipo de transação "Serviço antecipado" e marcar a opção [Condição de pagamento diferido].
7. Confirmar a operação e aguardar mensagem de sucesso.

### Resultado esperado
- O sistema exibe a mensagem "Transação registrada com sucesso".
- O cliente possui um saldo em aberto de R$ 150,00.
- Nenhum pagamento foi processado e as transações não foram contabilizadas como recebidas.


