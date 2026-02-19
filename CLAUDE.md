# heuristic-driven-SDLC — Instruções para o Claude

## Sobre este Repositório

Material prático do livro **"Heurísticas de Teste como Sistema de Decisão Técnica"**.
Contém skills de heurística organizadas por etapa do SDLC, exemplos de aplicação e guias de implementação.

---

## Estrutura de Skills

| Diretório | Propósito |
|-----------|-----------|
| `Skill/` | Skills de **análise** heurística — usadas para investigar requisitos e identificar lacunas |
| `docs/builSwagger/` | Skill para **geração** de documentação OpenAPI/Swagger |
| `docs/test/` | Guias de **implementação** de testes — usados após aplicar as heurísticas de Cap09 |

---

## Quando Aplicar as Skills

### Análise de Requisito (ordem sugerida por fase do SDLC)

1. **Cap04 — Concepção:** `Skill/Cap04_concepcao/UserScenarios.md` → `Skill/Cap04_concepcao/WWWWWHKE.MD`
2. **Cap05 — Design:** `Skill/Cap05_design/Dependencies.md` → `Skill/Cap05_design/MultiUser.md` → `Skill/Cap05_design/StateAnalysis.md`
3. **Cap06 — Arquitetura:** `Skill/Cap06_arquitetura/Count.md` → `Skill/Cap06_arquitetura/CRUD.md`
4. **Cap07 — Refinamento:** `Skill/Cap07_refinamento/Chique.md` → `Skill/Cap07_refinamento/InputMethod.md` → `Skill/Cap07_refinamento/SeenAndHeard.md`
5. **Cap08 — Desenvolvimento:** `Skill/Cap08_desenvolvimento/Baica.md` → `Skill/Cap08_desenvolvimento/VADER.md` → `Skill/Cap08_desenvolvimento/FAILURE.md`
6. **Cap09 — Testes:** `Skill/Cap09_teste/EMOTIONS.md` → `Skill/Cap09_teste/FAILURE.md` → `Skill/Cap09_teste/FLOOD.md`

### Geração de Documentação

- **Swagger/OpenAPI** → `docs/builSwagger/SWAGGER_API_DOC.md`
- **Estratégia de testes** → `docs/test/TEST_STRATEGY.md`
- **Implementação de testes** → guias específicos em `docs/test/` (unit, integration, component, service, e2e)

---

## Onde Salvar Outputs

| Tipo de Output | Caminho |
|----------------|---------|
| Análises de heurísticas | `output/` |
| Requisito revisado | `output/requisito-revisado/` |
| Swagger gerado | `output/swagger/<nome-funcionalidade>/` |
| Estratégia de testes | `output/test-strategy/` |
| Exemplos do livro | `example-book/cap-<XX>-<fase>/` |

---

## Exemplo de Uso

Para analisar um requisito com uma heurística:

```
Analise o @input/REQ_INICIAL.MD com a @Skill/Cap04_concepcao/UserScenarios.md
e gere o report na pasta output/
```

Para seguir o fluxo completo, aplique as skills na ordem das fases acima, usando os outputs de cada fase como input para a próxima.
