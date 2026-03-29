# Heuristic Guide — Design de Codigo

Guia de heuristica para a fase de **design de codigo**: analisa gestao de concorrencia e conflitos em operacoes compartilhadas, identificando race conditions, deadlocks e problemas de consistencia em cenarios multi-usuario (Multi-User).

## Quando usar

- Ao revisar o design de uma feature antes da implementacao
- Quando a funcionalidade envolve multiplos usuarios simultaneos (race conditions, deadlocks)
- Quando ha escrita compartilhada ou recursos criticos com quantidade limitada
- Para identificar problemas de consistencia em operacoes concorrentes

## Como invocar

```
/heuristic-guide-desing-code
```

Ou especifique diretamente:

```
Aplique Multi-User nessa feature: [descricao]
```

## Inputs aceitos

| Tipo | Formato | Exemplo |
|------|---------|---------|
| Descricao textual | Texto livre | `"Modulo de reserva de ingressos"` |
| Arquivo Markdown | Caminho de arquivo `.md` | `docs/features/orders.md` |
| Link da ferramenta de gestão | URL de epico, story ou ambos | URL do épico/story na ferramenta (ex: ClickUp, Jira, Azure DevOps) |
| Ferramenta de gestão + codigo | Combinacao de URL da ferramenta e caminho/bloco de codigo | URL + `src/modules/orders/` |

## O que a skill produz

**Multi-User:**
- Analise em 4 lentes: Colisao, Sessao Dupla, Inventario Critico, UX de Conflito
- Identificacao de race conditions e deadlocks potenciais
- Estrategias de locking recomendadas (pessimista, otimista ou hibrida)
- Mecanismos de controle de concorrencia propostos
- Tratamento de conflitos e experiencia do usuario
- Recomendacoes de heuristicas complementares (State Analysis, Count)

## Exemplos de uso

**Texto livre:**
```
Aplique Multi-User na feature:
"Modulo de reserva de ingressos com pagamento e estoque"
```

**Arquivo Markdown:**
```
Aplique Multi-User nesse arquivo: docs/features/orders.md
```

**Link da ferramenta de gestão (story):**
```
Aplique Multi-User nessa story: <URL da story na ferramenta de gestão>
```

**Link da ferramenta de gestão (epico + story):**
```
Aplique Multi-User nesses itens:
- Epico: <URL do épico na ferramenta de gestão>
- Story: <URL da story na ferramenta de gestão>
```

**Caminho de repositorio:**
```
Aplique Multi-User nesse modulo: src/modules/orders/
```

**Bloco de codigo:**
```
Aplique Multi-User nesse codigo:
[cole o bloco de codigo aqui]
```

**Ferramenta de gestão + codigo:**
```
Aplique Multi-User combinando:
- Story: <URL da story na ferramenta de gestão>
- Modulo: src/modules/orders/
```

## Skills relacionadas

- [`/heuristic-guide-architeture-code`](../heuristic-guide-architeture-code/) — heuristicas de arquitetura (Count, CRUD)
- [`/heuristic-guide-development`](../heuristic-guide-development/) — heuristicas para fase de desenvolvimento (Baica, VADER)
- [`/heuristic-guide-brainstorm`](../heuristic-guide-brainstorm/) — heuristicas de brainstorm inicial
