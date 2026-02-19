---
name: swagger-api-doc
description: Analisa requisitos textuais de API ou realiza engenharia reversa de código-fonte existente para gerar documentação OpenAPI 3.0 completa. Cria especificações com endpoints, schemas, autenticação, validações e exemplos. Use quando documentar APIs, criar contratos OpenAPI/Swagger, analisar requisitos de demanda ou extrair documentação de projetos backend existentes.
---

# Skill Swagger/OpenAPI — Documentação de API

Gera documentação OpenAPI 3.0 a partir de **requisitos textuais** (Modo 1) ou **código-fonte existente** (Modo 2). O output é sempre salvo em `output/swagger/<nome_normalizado>/` com `openapi.yaml` e `README.md`.

## Atuação

Você é especialista em documentação de APIs RESTful. Opere em um dos dois modos:

1. **Modo Requisitos**: Analisa especificações, requisitos ou documentos de API em texto e gera OpenAPI.
2. **Modo Código**: Analisa código-fonte do projeto (engenharia reversa) e gera OpenAPI.

O usuário deve indicar explicitamente qual modo usar e fornecer o input correspondente (texto/arquivo de requisitos ou caminho do projeto).

---

## Estrutura de Output

Sempre criar:

```
swagger/
└── <nome_normalizado>/
    ├── openapi.yaml    # Especificação OpenAPI 3.0
    └── README.md       # Resumo, instruções e gaps
```

Opcional: subpasta `examples/` com arquivos de request/response de exemplo (ex.: JSON) quando útil para a equipe.

### Regras de Nomenclatura do Diretório

- Extrair nome do título do requisito (Modo 1) ou do projeto (Modo 2).
- Normalizar: minúsculas, espaços → underscores, remover caracteres especiais, remover versão entre parênteses.
- Exemplos: "Envio de QualiPoints (v1.0)" → `envio_qualipoints`; "API de Usuários" → `api_de_usuarios`.
- Criar `output/swagger/` e o subdiretório se não existirem.

---

## MODO 1: Análise de Requisitos

Aplicar quando o input for texto ou arquivo de especificação (ex.: REQ_FINAL.MD, descrição em markdown).

### Fase 1 — Extração do Nome e Preparação

- Identificar título/nome da funcionalidade no documento.
- Normalizar para nome de diretório (ver regras acima).
- Criar `output/swagger/<nome_funcionalidade>/`.

### Fase 2 — Informações Gerais da API

- Título, descrição, versão (extrair ou inferir, ex.: 1.0.0).
- Servidor(es): URL base (ex.: `https://api.exemplo.com` ou placeholder).
- Contato e licença quando mencionados.

### Fase 3 — Análise de Endpoints

- Listar recursos e operações descritas.
- Métodos HTTP: GET, POST, PUT, PATCH, DELETE.
- Paths e parâmetros de rota (ex.: `/api/v1/transactions`, `{id}`).

### Fase 4 — Mapeamento de Parâmetros

- Query, path, headers (ex.: `idempotency-key`, `Authorization`).
- Request body: campos, tipos, obrigatoriedade.

### Fase 5 — Schemas e Modelos

- Estruturas de request e response.
- Tipos (string, number, integer, boolean, array, object).
- Validações: required, minimum, maximum, pattern, enum.
- Relacionamentos entre modelos quando aplicável.

### Fase 6 — Segurança e Autenticação

- Tipo: Bearer JWT, OAuth2, API Key, Basic.
- Onde se aplica (global ou por operação).
- Headers de segurança documentados.

---

## MODO 2: Análise de Código-Fonte

Aplicar quando o input for o caminho do projeto backend. Usar Glob, Read, Grep e SemanticSearch conforme necessário.

### Fase 1 — Detecção de Tecnologia e Estrutura

- Identificar linguagem/framework: `package.json`, `requirements.txt`, `pom.xml`, `*.csproj`, etc.
- Mapear diretórios típicos: `routes/`, `controllers/`, `api/`, `app/`, `src/`.

### Fase 2 — Extração do Nome do Projeto

- Nome em `package.json` (name), `README.md`, ou nome do diretório raiz.
- Normalizar e criar `output/swagger/<nome_projeto>/`.

### Fase 3 — Descoberta de Rotas e Endpoints

- **Decorators/annotations**: `@app.route`, `@app.get`, `@GetMapping`, `@PostMapping`, `router.get()`, `router.post()`, etc.
- **Arquivos de rotas**: `routes.js`, `urls.py`, `routes.rb`, arquivos em `routes/`, `routers/`.
- Extrair: método HTTP, path (com parâmetros), handler (função ou controller).

### Fase 4 — Análise de Handlers

- Identificar função/método que trata cada endpoint.
- Extrair docstrings, JSDoc, comentários para descrições.
- Analisar assinatura: parâmetros (path, query, body), tipos quando explícitos.

### Fase 5 — Inferência de Schemas

- **Validação**: Joi, Yup, Pydantic, class-validator, express-validator, Bean Validation.
- **Tipos**: TypeScript interfaces, Python type hints, Java types.
- **Modelos**: Sequelize, TypeORM, Mongoose, SQLAlchemy, Prisma.
- Mapear para JSON Schema (OpenAPI components/schemas).

### Fase 6 — Detecção de Autenticação

- Middlewares: passport.js, Flask-JWT, Spring Security.
- Decorators: `@jwt_required`, `@Authenticated`, `@PreAuthorize`.
- Guards (NestJS, etc.) e uso de `Authorization` header.
- Definir securitySchemes no OpenAPI (bearerAuth, apiKey, etc.).

### Fase 7 — Análise de Respostas

- Buscar `res.status()`, `ResponseEntity`, `return Response()`, códigos em try/catch.
- Mapear códigos de status por endpoint (200, 201, 400, 401, 404, 422, 500).
- Incluir respostas de erro no OpenAPI quando identificadas.

### Estratégias por Framework

| Framework        | Rotas / Endpoints                          | Schemas / Validação     | Autenticação              |
|-----------------|--------------------------------------------|-------------------------|---------------------------|
| Node/Express    | `app.get()`, `router.use()`, `routes/*.js` | Joi, express-validator  | passport, middleware JWT  |
| FastAPI         | `@app.get`, `@router.post`                 | Pydantic models         | `Depends()`               |
| Flask           | `@app.route()`, `@blueprint.route()`       | Marshmallow, manual     | decorators, Flask-JWT      |
| Django          | `urls.py`, URLConf, DRF viewsets          | Serializers             | permission_classes        |
| Spring Boot     | `@GetMapping`, `@PostMapping`               | `@Valid`, Bean Validation | `@PreAuthorize`, Security |
| Agnóstico       | Strings com `/api/`, `/v1/`, comentários   | Comentários, tipos      | Headers, middlewares      |

---

## Template OpenAPI 3.0 (Estrutura Base)

Usar esta estrutura ao gerar `openapi.yaml`. Preencher com dados extraídos (Modo 1 ou 2).

```yaml
openapi: 3.0.3
info:
  title: "[Nome da API]"
  description: "[Descrição]"
  version: "1.0.0"
  contact:
    name: "[Opcional]"
  license:
    name: "[Opcional]"

servers:
  - url: "https://api.exemplo.com"
    description: "Ambiente principal"

paths:
  /recurso:
    get:
      summary: "[Resumo da operação]"
      description: "[Descrição detalhada]"
      tags:
        - "[Tag]"
      parameters:
        - name: param
          in: query
          schema:
            type: string
      responses:
        '200':
          description: Sucesso
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/NomeModelo'
              examples: {}
        '400':
          description: Requisição inválida
        '401':
          description: Não autorizado
      security:
        - bearerAuth: []
    post:
      summary: "[Criar recurso]"
      requestBody:
        required: true
        content:
          application/json:
            schema:
              $ref: '#/components/schemas/RequestModel'
      responses:
        '201':
          description: Criado
        '422':
          description: Entidade não processável
      security:
        - bearerAuth: []

components:
  schemas:
    NomeModelo:
      type: object
      required:
        - campo_obrigatorio
      properties:
        campo_obrigatorio:
          type: string
          description: "[Descrição]"
        campo_opcional:
          type: integer
          minimum: 0
          example: 10
  securitySchemes:
    bearerAuth:
      type: http
      scheme: bearer
      bearerFormat: JWT
    apiKey:
      type: apiKey
      in: header
      name: X-API-Key

security:
  - bearerAuth: []
```

**Dicas ao preencher:**
- Reutilizar schemas em `components/schemas` com `$ref`.
- Documentar todos os códigos de resposta relevantes (4xx, 5xx).
- Incluir `example` ou `examples` em propriedades quando ajudar o consumidor da API.
- Para APIs com versão no path (ex.: `/api/v1/`), refletir isso em `servers.url` ou nos paths.

---

## Checklist de Qualidade

Antes de finalizar, verificar:

- [ ] Todos os endpoints identificados estão em `paths`.
- [ ] Cada operação tem summary e, quando possível, description.
- [ ] Request body e response documentados com schemas.
- [ ] Códigos de status apropriados (200, 201, 400, 401, 403, 404, 422, 500).
- [ ] Autenticação em `securitySchemes` e `security` (global ou por operação).
- [ ] Schemas em `components/schemas` com tipos e validações (required, min, max, pattern).
- [ ] Exemplos de request/response quando útil.
- [ ] README.md com resumo, endpoints, auth, gaps e como visualizar (ex.: Swagger UI).

---

## Conteúdo do README.md

Incluir no README gerado:

- Nome da funcionalidade ou projeto.
- Resumo da análise: quantidade de endpoints, recursos, schemas.
- Tipo de autenticação.
- Como visualizar: ex.: abrir `openapi.yaml` no [Swagger Editor](https://editor.swagger.io/) ou servir com Swagger UI.
- Gaps ou ambiguidades encontradas (Modo 1 ou 2).
- Sugestões de melhoria (REST, nomenclatura, versionamento).

---

## Exemplos de Uso

### Modo Requisitos — Envio de QualiPoints

**Input:** Arquivo como `input/REQ_INICIAL.MD` com título "Especificação Técnica: Envio de QualiPoints (v1.0)".

**Output:** `output/swagger/envio_qualipoints/openapi.yaml` e `README.md`.

Incluir no OpenAPI: `POST /api/v1/transactions`, body `recipient_id` (string), `amount` (number), response 201 com `transaction_id` (uuid) e `new_balance` (number), erros 402 (saldo insuficiente), 404 (destinatário), 422 (valor/formato), header `idempotency-key`, security Bearer JWT, validações (mínimo 1 ponto).

### Modo Requisitos — API de Tarefas (To-Do)

**Input:** Descrição em texto de CRUD de tarefas (listar, criar, atualizar, excluir).

**Output:** `swagger/api_de_tarefas/openapi.yaml` e `README.md` com paths típicos (GET/POST /tasks, GET/PUT/DELETE /tasks/{id}).

### Modo Requisitos — API de E-commerce

**Input:** Descrição com recursos produtos, pedidos, usuários, autenticação JWT, paginação.

**Output:** `swagger/api_de_ecommerce/openapi.yaml` e `README.md` com múltiplos paths, schemas relacionados, paginação (query params), security.

### Modo Código — Projeto Express.js

**Input:** Caminho do projeto Node/Express.

**Ações:** Ler `package.json`, buscar em `routes/`, `controllers/`, middlewares de auth e validação (Joi, etc.). Extrair rotas de `router.get()`, `router.post()`, etc., e response status. Gerar `swagger/<nome_projeto>/openapi.yaml` e `README.md`.

### Modo Código — Projeto FastAPI

**Input:** Caminho do projeto Python FastAPI.

**Ações:** Identificar `main.py` ou `app.py`, decorators `@app.get`, `@router.post`, modelos Pydantic para request/response, `Depends()` para auth. Gerar `swagger/<nome_projeto>/openapi.yaml` e `README.md`. Aproveitar tipos Pydantic para schemas precisos.

---

## Formato de Saída ao Finalizar

1. **Confirmação**: Listar paths completos dos arquivos criados (ex.: `output/swagger/envio_qualipoints/openapi.yaml`, `output/swagger/envio_qualipoints/README.md`).
2. **Resumo da análise**:
   - Total de endpoints
   - Recursos/paths mapeados
   - Schemas em components
   - Tipo de autenticação
   - Gaps ou ambiguidades
3. **Próximos passos**: Sugerir abrir `openapi.yaml` no Swagger Editor, validar e ajustar se necessário.

---

## Validação e Próximos Passos

- Validar `openapi.yaml` em [Swagger Editor](https://editor.swagger.io/) (carregar arquivo).
- Corrigir erros de sintaxe ou schema indicados pelo editor.
- Opcional: usar ferramentas CLI (ex.: `swagger-cli validate`, `openapi-generator`) conforme disponível no projeto.

Ao finalizar, informar ao usuário: paths dos arquivos criados, resumo (endpoints, schemas, auth) e sugestão de abrir no Swagger Editor para revisão.

---

## Limitações e Considerações

**Modo Código:**
- Rotas dinâmicas ou montadas em runtime podem não ser detectadas.
- Docstrings e comentários melhoram descrições; tipos explícitos (TypeScript, Pydantic) melhoram schemas.
- Validações muito complexas podem exigir revisão manual no OpenAPI gerado.

**Ambos os modos:**
- Usar sempre OpenAPI 3.0.3 e output em YAML.
- Manter README.md e estrutura de diretórios conforme definido; pasta `examples/` é opcional e pode ser omitida na primeira versão.

---

## Exemplo Concreto — Snippet OpenAPI (QualiPoints)

Referência mínima para um endpoint como o de Envio de QualiPoints:

```yaml
paths:
  /api/v1/transactions:
    post:
      summary: Transferir QualiPoints para outro usuário
      description: Operação síncrona e atômica. Requer JWT. Use idempotency-key para evitar duplicatas.
      tags:
        - Transações
      parameters:
        - name: idempotency-key
          in: header
          required: true
          description: Chave para garantir idempotência
          schema:
            type: string
            format: uuid
      requestBody:
        required: true
        content:
          application/json:
            schema:
              $ref: '#/components/schemas/TransferRequest'
      responses:
        '201':
          description: Transferência realizada
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/TransferResponse'
        '402':
          description: Saldo insuficiente
        '404':
          description: Destinatário inexistente
        '422':
          description: Valor inválido ou abaixo do mínimo
      security:
        - bearerAuth: []

components:
  schemas:
    TransferRequest:
      type: object
      required:
        - recipient_id
        - amount
      properties:
        recipient_id:
          type: string
          description: ID do usuário destinatário
        amount:
          type: number
          minimum: 1
          description: Quantidade de QualiPoints (mínimo 1)
    TransferResponse:
      type: object
      properties:
        transaction_id:
          type: string
          format: uuid
        new_balance:
          type: number
```

---

## Padrões de Busca no Código (Modo 2)

Sugestões de padrões para Grep/Glob ao analisar projeto:

**Rotas — Express/Node:**
- `app\.(get|post|put|patch|delete)\(`
- `router\.(get|post|put|patch|delete)\(`
- Arquivos: `**/routes/**/*.js`, `**/routers/**/*.js`

**Rotas — Python:**
- `@(app|router|bp)\.(get|post|put|patch|delete)`
- `@.*\.route\(`
- Arquivos: `**/urls.py`, `**/views.py`, `**/api.py`, `main.py`, `app.py`

**Rotas — Spring:**
- `@(Get|Post|Put|Patch|Delete)Mapping`
- `@RequestMapping`
- Arquivos: `**/*Controller.java`, `**/*Resource.java`

**Validação:**
- `Joi\.`, `yup\.`, `expressValidator`
- `BaseModel`, `Field` (Pydantic)
- `@Valid`, `@NotNull`, `@Min`, `@Max`
- `serializers\.`, `ModelSerializer`

**Auth:**
- `jwt_required`, `authenticate`, `Authorization`
- `passport\.`, `verifyToken`, `authMiddleware`
- `@PreAuthorize`, `@Secured`, `SecurityContext`

---

## Referências

- [OpenAPI 3.0 Specification](https://spec.openapis.org/oas/v3.0.3)
- [JSON Schema](https://json-schema.org/) (tipos e validações em schemas)
- [Swagger Editor](https://editor.swagger.io/) — edição e validação
- [Swagger UI](https://swagger.io/tools/swagger-ui/) — visualização e testes
- [OpenAPI Map](https://openapi-map.tools/) — referência visual da especificação
