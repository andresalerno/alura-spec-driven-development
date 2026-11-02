# Módulo: products

## Objetivo

Gerenciar o catálogo de produtos do e-commerce: criação, consulta, atualização e remoção, com paginação, filtros e cache Redis na listagem.

## Responsabilidade Principal

Expor operações CRUD sobre produtos, garantindo unicidade de SKU, vinculação com categorias e controle de acesso por role.

## Funcionalidades Existentes

| Rota | Método | Proteção | Descrição |
|---|---|---|---|
| `/api/v1/products` | GET | Pública | Lista produtos com paginação e filtros |
| `/api/v1/products/:id` | GET | Pública | Retorna produto por ID (inclui categoria) |
| `/api/v1/products` | POST | JWT + ADMIN | Cria novo produto |
| `/api/v1/products/:id` | PUT | JWT + ADMIN | Atualiza produto |
| `/api/v1/products/:id` | DELETE | JWT + ADMIN | Remove produto |

### Paginação e Filtros (GET /)

Query params aceitos:
- `page` (padrão: 1)
- `limit` (padrão: 20)
- `categoryId` — filtra por categoria
- `search` — busca por `name` ou `description` (case-insensitive, via Prisma `contains`)

Somente produtos com `status: ACTIVE` são retornados na listagem.

### Cache (listProducts)

A listagem usa **cache-aside** via Redis:
- **Chave:** `products:list:{json dos parâmetros}`
- **TTL:** 60 segundos
- **Erro de Redis** é silenciado com `console.error` (não interrompe o fluxo)
- ⚠ **Não há invalidação de cache nas operações de escrita** (create, update, delete)

## Dependências Internas

| Dependência | Uso |
|---|---|
| `src/config/database` | Todas as queries Prisma (via productRepository) |
| `src/config/redis` | Cache da listagem |
| `src/middlewares/authMiddleware` | Proteção das rotas de escrita |
| `src/utils/AppError` | Erros de negócio |

## Dependências Externas

- `@prisma/client`
- `redis`

## Módulos Relacionados

- **categories** — `Product` tem FK para `Category`; `findById` e `findAll` incluem `category`
- **cart** — `cartService` acessa `prisma.product` diretamente (sem passar pelo `productRepository`)
- **orders** — `orderService` usa `productRepository.findById` para validar produtos no checkout

## Pontos de Entrada

- `productRoutes.js` → registrado no `server.js` em `/api/v1/products`

## Fluxos Importantes

```
GET /api/v1/products?page=1&search=camiseta
  → productController.list
    → productService.listProducts({ page, limit, search })
      → redisClient.get(cacheKey) → HIT: retorna JSON; MISS: continua
      → productRepository.findAll({ skip, take, search })
        → prisma.product.findMany({ where: { status: ACTIVE, OR: [name, desc] }, include: category })
        → prisma.product.count (para paginação)
      → redisClient.set(cacheKey, json, TTL 60s)
      → retorna { data: [...], pagination: { page, limit, totalItems, totalPages } }
```

```
POST /api/v1/products  (ADMIN)
  authMiddleware → adminOnly
    → productController.create
      → productService.createProduct
        → productRepository.findBySku (unicidade)
        → productRepository.create
```

## Arquivos Críticos

| Arquivo | Descrição |
|---|---|
| `productService.js` | Cache-aside, paginação e regras de unicidade de SKU |
| `productRepository.js` | Queries Prisma com filtros compostos e include de categoria |
| `productRoutes.js` | Mix de rotas públicas e protegidas no mesmo router via `router.use(authMiddleware)` posicional |

## Observações Técnicas e Débitos

1. **Cache não é invalidado na escrita:** criar, atualizar ou deletar um produto não remove as chaves `products:list:*`. O catálogo exibido pode ficar desatualizado por até 60 segundos.
2. **`console.error` no Redis:** o erro de Redis em `listProducts` usa `console.error` em vez do logger Winston — inconsistência com o padrão do projeto.
3. **`adminOnly` duplicado:** a mesma função está declarada em `productRoutes.js` e `categoryRoutes.js`. Deveria ser centralizada em um middleware reutilizável.
4. **Sem validação de payload nas rotas de escrita:** `create` e `update` não têm validators (`express-validator`) — qualquer body é aceito e passado diretamente ao Prisma.
5. **`findAll` filtra apenas `status: ACTIVE`:** produtos inativos são invisíveis para qualquer usuário, incluindo ADMINs. Pode ser intencional, mas é um comportamento não óbvio.
