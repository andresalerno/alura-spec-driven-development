# Arquitetura do Sistema

> Documento gerado com base em análise estática do código em `src/`, `prisma/schema.prisma` e arquivos de configuração.  
> Inferências marcadas explicitamente como **[Hipótese]**.

---

## Visão Arquitetural

API REST em Node.js estruturada em **arquitetura modular por domínio** (feature-based), com separação clara entre camadas dentro de cada módulo. O sistema é stateless na camada HTTP e delega estado de sessão para Redis. Persistência relacional via PostgreSQL com Prisma ORM.

```
Cliente HTTP
     │
     ▼
 Express App (src/server.js)
     │
     ├── Middlewares Globais (Helmet, CORS, Compression, Morgan, ErrorHandler)
     │
     ├── /api/v1/{recurso}  ──►  Routes  ──►  Controller  ──►  Service  ──►  Repository  ──►  Prisma (PostgreSQL)
     │                                              │
     │                                              └──►  Redis (Cache / Carrinho / Tokens)
     │
     └── /health  (healthcheck)
```

---

## Padrões Utilizados

| Padrão | Onde é aplicado |
|---|---|
| **Controller → Service → Repository** | Todos os módulos de domínio |
| **Repository Pattern** | `*Repository.js` encapsula todas as queries Prisma |
| **Error centralizado** | `AppError` + `errorHandler` middleware |
| **JWT (access + refresh)** | `src/utils/token.js` + Redis para refresh tokens |
| **Cache-aside** | `productService.js` – busca Redis antes do banco |
| **Idempotência** | `paymentService.confirmPayment` – rejeita duplo processamento |
| **Prisma Transaction** | `orderRepository.createOrderTransaction` e `paymentService.confirmPayment/cancelPayment` |
| **Snapshot de preço** | `orderService.checkout` – preço do produto no momento da compra, não referência viva |
| **Paginação** | `productService.listProducts` – `page` e `limit` via query string |
| **RBAC simples** | Guard `adminOnly` em rotas de Products e Categories; verificação direta de `role` em Users |

---

## Regras Arquiteturais

1. **Controllers não acessam banco diretamente.** Toda lógica de persistência passa pelo Repository.
2. **Services são a única camada que orquestra lógica de negócio.** Controllers fazem apenas extração de parâmetros e formatação de resposta.
3. **Repositories encapsulam exclusivamente queries Prisma.** Não contêm regra de negócio.
4. **Erros de domínio são instâncias de `AppError`.** Lançados nos Services e capturados pelo `errorHandler`.
5. **Rotas públicas e protegidas são declaradas no mesmo router,** separadas por `router.use(authMiddleware)` posicional.
6. **Redis é o único estado compartilhado entre requisições** (carrinho, refresh tokens, cache de produtos). Não há variável de módulo com estado mutável.

---

## Convenções Técnicas

- **IDs:** UUID v4 gerado pelo PostgreSQL via `@default(uuid())`.
- **Timestamps:** `createdAt` e `updatedAt` gerenciados pelo Prisma em todos os modelos.
- **Respostas HTTP:** corpo sempre com `{ data: ... }` para sucesso; `{ error: { code, message, details } }` para erro.
- **Erros 4xx:** logados em `warn`. Erros 5xx: logados em `error` com stack trace.
- **Logging:** Winston (arquivo `logs/error.log`, `logs/all.log`) + Morgan para HTTP (só em `development`).
- **Variáveis de ambiente:** todas declaradas no `.env` e documentadas no README. Nunca commitadas.
- **Moeda:** `Decimal(10,2)` no banco; convertida para `Number` no JavaScript quando necessário.
- **Multi-tenancy:** campo `tenantId` presente em `User`, `Product` e `Order` com valor padrão `"default"`. Não há lógica de isolamento implementada ainda. **[Hipótese: campo reservado para evolução futura]**.

---

## Separação de Responsabilidades

```
src/
├── config/          # Instâncias únicas de infraestrutura (Prisma, Redis, Winston)
├── middlewares/     # Comportamentos transversais (auth, validação, logging, erro)
├── modules/
│   └── {domínio}/
│       ├── *Routes.js       # Declaração de rotas e guards inline
│       ├── *Controller.js   # Extração de parâmetros + chamada ao Service
│       ├── *Service.js      # Regras de negócio + orquestração
│       └── *Repository.js   # Queries Prisma (ausente em auth e cart)
├── utils/
│   ├── AppError.js  # Classe de erro operacional com statusCode, code e details
│   └── token.js     # Geração e verificação de JWT
└── server.js        # Bootstrap do Express, registro de middlewares e rotas
```

**Módulos sem Repository:** `auth` (usa `userRepository` de Users) e `cart` (acessa Redis e Prisma diretamente no Service).

---

## Fluxo de Comunicação entre Módulos

```
auth ──────────────────────────────────────────────────────► users (userRepository)
                                                                     │
                                                                     ▼
cart ──────────────────────────────────────────────────────► products (prisma.product)
  ▲                                                                   │
  │                                                                   │
orders ◄──────────────────────────────────────────────────── (cartService + productRepository)
  │
  ▼
payments ──────────────────────────────────────────────────► orders (prisma.order via tx)
```

Dependências diretas entre módulos de domínio:
- `auth` → `users`
- `orders` → `cart` + `products`
- `payments` → `orders` (via `prisma.$transaction` – acoplamento direto ao banco, não ao módulo)

---

## Dependências Críticas

| Dependência | Uso | Impacto se indisponível |
|---|---|---|
| **PostgreSQL** | Persistência de todos os modelos | API inoperante |
| **Redis** | Carrinho, refresh tokens, cache de produtos | Login bloqueia; carrinho inacessível; listagem de produtos sem cache |
| **`@prisma/client`** | ORM | Necessita `npx prisma generate` após qualquer pull |
| **`jsonwebtoken`** | Auth | API inoperante para rotas protegidas |
| **`bcrypt`** | Hash de senhas | Registro e login inoperantes |

---

## Riscos Técnicos e Acoplamentos Importantes

### Acoplamentos

1. **`orderService` importa `redisClient` diretamente** ([src/modules/orders/orderService.js:666](../src/modules/orders/orderService.js)) — bypassa o `cartService` para limpar o carrinho. Deveria existir um `cartService.clearCart()`.
2. **`paymentService` acessa `prisma` diretamente** para atualizar `order.status` — viola a camada de Repository do módulo Orders. Correto seria `orderRepository.updateStatus()` ou comunicação via evento.
3. **`adminOnly` duplicado** em `productRoutes.js` e `categoryRoutes.js` — deveria ser um middleware reutilizável.
4. **`requestLogger.js` (middleware)** usa `console.log` e não está registrado no `server.js`. É um arquivo órfão.

### Riscos

1. **Sem invalidação de cache na escrita de produtos:** criar, atualizar ou deletar um produto não invalida as chaves `products:list:*` no Redis.
2. **Estoque pode ficar negativo no checkout:** o decrement atomico em `orderRepository` não usa `WHERE stock >= quantity`, verificando o saldo após o update — janela de race condition em alta concorrência.
3. **Refresh token não é invalidado no logout:** não há rota de logout implementada. O refresh token no Redis expira em 7 dias, mas não há mecanismo de revogação explícita.
4. **`Payment.status` é `String` no schema Prisma,** mas os valores usados (`AWAITING_CONFIRMATION`, `PAID`, `CANCELED`) não estão em um `enum` — inconsistência com o restante do schema.
5. **`swagger-ui-express` e `yamljs` declarados como dependências de produção,** mas o swagger está comentado no `server.js`. A documentação da API não está implementada.
6. **Sem testes automatizados:** `jest` e `supertest` estão no `devDependencies`, mas o script `test` retorna erro imediatamente. Nenhum arquivo `*.test.js` ou `*.spec.js` existe na branch atual.

---

## Diretrizes para Futuras Implementações

1. **Ao criar ou modificar um módulo**, siga sempre a estrutura `Routes → Controller → Service → Repository`. Nunca pule camadas.
2. **Novos erros de negócio** devem usar `AppError` com código semântico (`CONFLICT`, `RESOURCE_NOT_FOUND`, `FORBIDDEN`, etc.).
3. **Operações que afetam múltiplos modelos** devem ser encapsuladas em `prisma.$transaction`.
4. **Cache Redis:** ao escrever um recurso cacheado, invalide todas as chaves relacionadas no mesmo Service.
5. **Ao implementar logout**, adicione `redisClient.del('refresh:{userId}')` no `authService`.
6. **`adminOnly`** deve ser extraído para `src/middlewares/adminMiddleware.js` e reutilizado.
7. **Testes:** estruture em `src/modules/{domínio}/*.test.js` usando Jest + Supertest. O `server.js` já exporta `app` sem iniciar o servidor quando importado como módulo.
8. **Swagger:** o arquivo `swagger.yaml` deve ser criado na raiz e o bloco comentado em `server.js` descomentado para habilitar `/api-docs`.
9. **Multi-tenancy:** qualquer isolamento real por `tenantId` deve ser implementado como middleware ou plugin do Prisma, nunca espalhado nos Services individualmente.
