# Módulo: orders

## Objetivo

Processar o checkout do carrinho, criando um pedido com itens, total calculado e pagamento stub associado. Também expõe a listagem de pedidos do usuário autenticado.

## Responsabilidade Principal

Orquestrar a transação de checkout: ler o carrinho, validar produtos, decrementar estoque, persistir pedido + itens + pagamento numa única transação Prisma e limpar o carrinho.

## Funcionalidades Existentes

| Rota | Método | Proteção | Descrição |
|---|---|---|---|
| `/api/v1/orders` | GET | JWT | Lista pedidos do usuário logado |
| `/api/v1/orders` | POST | JWT | Realiza checkout (cria pedido) |

### Checkout (`POST /`)

1. Recupera carrinho do Redis via `cartService.getCart`.
2. Para cada item do carrinho, valida no banco: produto existe, `status === ACTIVE`, estoque suficiente.
3. Captura snapshot do preço atual do produto.
4. Executa `prisma.$transaction`:
   - Decrementa estoque de cada produto.
   - Verifica se estoque ficou negativo após decrement (verificação pós-update — ver débitos).
   - Cria `Order` com status `PENDING`.
   - Cria `OrderItem[]` com preço snapshot.
   - Cria `Payment` stub com status `AWAITING_CONFIRMATION`.
5. Deleta chave do carrinho no Redis diretamente (`cart:default:{userId}`).
6. Loga evento simulado `order.created`.

### Listagem (`GET /`)

Retorna todos os pedidos do usuário com `items` e `payment` incluídos, ordenados por `createdAt` desc.

## Dependências Internas

| Dependência | Uso |
|---|---|
| `src/modules/cart/cartService` | Leitura do carrinho no checkout |
| `src/modules/products/productRepository` | Validação de produto e preço snapshot |
| `src/config/database` | Importado diretamente no `orderService` para deletar carrinho no Redis |
| `src/config/redis` | Deletado diretamente no `orderService` (ver débitos) |
| `src/config/logger` | Log de erros e eventos simulados |
| `src/utils/AppError` | Erros de negócio |

## Dependências Externas

- `@prisma/client`
- `redis`

## Módulos Relacionados

- **cart** — leitura e limpeza do carrinho
- **products** — validação de estoque e snapshot de preço
- **payments** — `Payment` stub criado dentro da transação do checkout; `paymentService` gerencia o ciclo de vida do pagamento

## Pontos de Entrada

- `orderRoutes.js` → registrado no `server.js` em `/api/v1/orders`  
- Todas as rotas exigem `authMiddleware` (`router.use(authMiddleware)`)

## Fluxos Importantes

```
POST /api/v1/orders
  authMiddleware
    → orderController.checkout (req.user.sub)
      → orderService.checkout(userId)
        → cartService.getCart(userId)            [Redis]
        → productRepository.findById × N         [PostgreSQL]
        → orderRepository.createOrderTransaction [PostgreSQL - $transaction]
            ├── product.update × N (decrement stock)
            ├── product.findUnique × N (verifica stock >= 0)
            ├── order.create + orderItems.create
            └── payment.create (AWAITING_CONFIRMATION)
        → redisClient.del(cart:default:{userId}) [Redis]
        → logger.info("order.created")
        → retorna Order
```

## Arquivos Críticos

| Arquivo | Descrição |
|---|---|
| `orderService.js` | Orquestração do checkout; importa `redisClient` diretamente |
| `orderRepository.js` | Transação Prisma com lógica de decrement e criação de entidades |

## Observações Técnicas e Débitos

1. **Race condition no estoque:** `orderRepository` decrementa o estoque e depois verifica se ficou negativo. Em alta concorrência, dois checkouts simultâneos do mesmo item podem ambos passar e levar o estoque a negativo. A solução correta é `WHERE stock >= quantity` na cláusula `update` ou uma constraint `CHECK (stock >= 0)` no banco.
2. **Limpeza do carrinho fora do módulo cart:** `orderService.js` importa `redisClient` e deleta `cart:default:{userId}` diretamente — bypass do encapsulamento. Deveria existir `cartService.clearCart(userId)`.
3. **`tenantId` hardcoded:** a chave Redis para limpeza do carrinho usa `cart:default:{userId}` diretamente, ignorando o parâmetro `tenantId` do `cartService`.
4. **`Payment` criado na transação de Order:** o `orderRepository` cria o `Payment` stub internamente. Isso cria acoplamento entre os modelos Order e Payment dentro da mesma transação, mas sem passar pelo `paymentService`.
5. **Sem paginação na listagem:** `GET /` retorna todos os pedidos do usuário sem limite ou paginação.
6. **Sem rota de detalhe do pedido:** não há `GET /:id` para consultar um pedido específico.
