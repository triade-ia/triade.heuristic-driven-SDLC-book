
# TEMPLATE: Caso de Teste (CT)

> Modelo e orientações para escrever um Caso de Teste (CT) claro e padronizado.

## Título

- Sempre iniciar o título com **Validar**, seguido da ação e do comportamento esperado.
	- Exemplo: **Validar lançamento de entrada de consumação antecipada [Pagar depois]**

## Pré-condições

[Detalhar todas as permissões, cadastros, place features, org features necessárias para a execução do caso de teste descrito no step by step de forma minuciosamente detalhada]

## Step by step

[Seguir o padrão de escrita de casos de teste do time de forma detalhada. Usar passos numerados, verbos no infinitivo e colchetes para elementos clicáveis.]

Exemplo (apagar depois):
1. Abrir o app
2. Clicar no menu [Venda] >> [Venda ficha]
3. Selecionar o produto
4. Clicar em [Próximo] >> Informar no campo 'Número da mesa' o valor "10"
5. Clicar em [Salvar] >> [Confirmar]
6. Selecionar a forma de pagamento desejada

> O resultado deve ser descrito na seção **Resultado esperado**; aqui informamos apenas os passos até o estado final antes da validação.

## Resultado esperado

[Detalhar todos os lugares que têm impacto com a execução desse caso de teste de forma minuciosamente detalhada]

---

## Regras para escrever um Caso de Teste (CT)

### 1. Estrutura e formatação

- Título: iniciar com **Validar**.
- Seções padrão: **Pré-condições**, **Step by step**, **Resultado esperado**.
- Usar Markdown ou identação clara para facilitar leitura em documentos, planilhas ou ClickUp.

### 2. Escrita do step by step

- Usar colchetes `[]` ao mencionar botões, opções ou menus clicáveis. Ex.: `Clicar em [Check-in/Check-out] >> [Cadastrar]`.
- Usar aspas simples ou duplas para identificar nomes de tela, campos e labels. Ex.: no campo 'Nome' informar "João".
- Evitar gerúndio — usar verbos no infinitivo: `Abrir`, `Clicar`, `Selecionar`, `Confirmar`.
- Evitar termos técnicos de código (ex.: funções, variáveis, JSON) — focar na ação visível do usuário.
- Descrever valores concretos usados no teste. Ex.: `Definir valor "R$ 22,33"`.
- Não deixar ambiguidade: cada passo deve ser reproduzível por outro QA.

### 3. Escrita do Resultado Esperado

- Iniciar com o estado final esperado (não o processo). Ex.: `O sistema deve exibir a mensagem 'Entrada lançada com sucesso'`.
- Indicar onde validar o resultado (ex.: `Dashboard → Relatórios → Contas abertas → Aba Entradas`).
- Ser objetivo e mensurável para permitir avaliação clara de passou/falhou.

### 4. Clareza e consistência

- Usar sempre o mesmo padrão de verbos de ação: `Abrir`, `Clicar`, `Selecionar`, `Confirmar`, `Verificar`, `Aguardar`, `Validar`.
- Manter nomes de telas e menus idênticos aos exibidos no produto.
- Indicar contexto quando necessário (ex.: nível de validação: evento, place, cliente).

### 5. Complementos recomendados

- Título curto alternativo: versão curta para uso em boards ou planilhas.
- Alinhamento com automação: verificar se o CT reflete o comportamento validado pelos testes automatizados.
- Tags: incluir criticidade e tipo de teste quando aplicável (blocker, major, regression, smoke).
- Linguagem neutra e profissional; evitar abreviações informais.
- Observações ou recomendações extras devem ir em nota ao final, fora dos passos principais.

---

## Exemplo completo

[ID-455] - **Validar lançamento de entrada**

### Pré-condições
- No Admin (Dashboard): permissão “Cobrar entradas” ativada.
- Evento aberto.
- Operador com acesso ao PDV.
- Entrada “Consumação antecipada” cadastrada no valor de R$ 22,33.
- ZigCard físico vinculado a um baladeiro válido.

### Step by step
1. Abrir o app e logar com as credenciais do operador.
2. Clicar em [Check-in/Check-out] >> [Cadastrar].
3. No campo 'Nome', cadastrar um baladeiro anônimo.
4. Retornar à Home.
5. Clicar em [Caixa e Recepção] >> [Lançar entrada].
6. Selecionar a entrada “Consumação antecipada” e marcar [Pagar depois].
7. Confirmar a operação e aguardar mensagem de sucesso.

### Resultado esperado
- O sistema exibe a mensagem “Entrada lançada com sucesso”.
- O baladeiro possui uma dívida pendente de R$ 22,33.
- Nenhum pagamento foi realizado e o saldo de consumo permanece zerado.


