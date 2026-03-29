# Heuristic Guide — Arquitetura de Código

Guia de heurísticas para a fase de **arquitetura de código**: analisa escalabilidade por volume (Count — 0, 1, Muitos) e operações de dados sob aspectos de segurança, performance e consistência (CRUD).

## Quando usar

- Ao desenhar ou revisar a arquitetura de um módulo ou API
- Quando precisar identificar comportamentos extremos de volume (0, 1, muitos registros)
- Quando for analisar endpoints de criação, leitura, atualização e deleção
- Para garantir segurança e consistência nas operações de dados

## Como invocar

```
/heuristic-guide-architeture-code
```

Ou especifique a heurística:

```
Aplique Count nessa feature: [descrição]
Aplique CRUD nessa feature: [descrição]
```

## Inputs necessários

| Campo | Descrição | Exemplo |
|-------|-----------|---------|
| Épico | URL do épico na ferramenta de gestão | URL do épico (ex: ClickUp, Jira, Azure DevOps) |
| Épico + Story/Tarefa | URLs do épico e da story/tarefa | dois links da ferramenta de gestão |
| Repositório | Caminho de diretório no projeto | `src/modules/orders/` |
| Função específica | Caminho de arquivo + função ou bloco de código | `src/services/order.service.ts#fn` ou ` ``` ` |
| Combinação | Qualquer combinação dos anteriores | link da ferramenta + caminho |

## O que a skill produz

**Count (0, 1, Muitos):**
- Análise de comportamento nos 3 extremos de volume
- Casos de teste para cada extremo (incluindo exemplos em TypeScript ou Java)
- Identificação de limites e edge cases de escalabilidade

**CRUD:**
- Análise de cada operação (Create, Read, Update, Delete)
- Identificação de riscos de segurança (autorização, exposição de dados)
- Análise de performance e consistência de dados

## Exemplo de uso

```
Aplique Count na feature:
"Listagem de ingressos de um evento — o evento pode ter 0, 1 ou milhares de ingressos"
```

## Skills relacionadas

- [`/heuristic-guide-desing-code`](../heuristic-guide-desing-code/) — heurísticas de design (Dependencies, Multi-User, State Analysis)
- [`/heuristic-guide-development`](../heuristic-guide-development/) — heurísticas para desenvolvimento (Baica, VADER)
- [`/heuristic-guide-planning`](../heuristic-guide-planning/) — heurísticas de planejamento de testes
