# Módulo: cart

## Objetivo

Gerenciar o carrinho de compras efêmero de cada usuário, armazenado inteiramente no Redis.

## Responsabilidade Principal

Ler, adicionar e remover itens do carrinho, validando disponibilidade e estoque do produto no banco antes de cada adição. O carrinho não tem representação no PostgreSQL.

## Funcionalidades Existentes

| Rota | Método | Proteção | Descrição |
|---|---|---|---|
| `/api/v1/cart` | GET | JWT (`authMiddleware` no server.js) | Retorna o carrinho atual |
| `/api/v1/cart/items` | POST | JWT | Adiciona ou incrementa item |
| `/api/v1/cart/items/:itemId` | DELETE | JWT | Remove item pelo `productId` |

### Comportamentos

- **getCart:** lê a chave `cart:default:{userId}` no Redis. Se não existir, retorna `{ items: [], total: 0 }`.
- **addItem:** valida produto (existência + estoque) no banco → atualiza item existente (acumula quantidade) ou adiciona novo → recalcula total → salva no Redis com TTL de 30 dias.
- **removeItem:** o parâmetro de rota é `:itemId`, mas internamente o `cartService.removeItem` filtra por `productId`, não por um ID único de item. **O `:itemId` na rota é tratado como `productId`.**

## Estrutura do Carrinho no Redis

```json
{
  "items": [
    {
      "productId": "uuid",
      "name": "Nome do Produto",
      "price": 99.90,
      "quantity": 2
    }
  ],
  "total": 199.80
}
```

**Chave Redis:** `cart:{tenantId}:{userId}` — o `tenantId` é fixo em `"default"` no código atual.  
**TTL:** 30 dias (`CART_TTL = 30 * 24 * 60 * 60`).

## Dependências Internas

| Dependência | Uso |
|---|---|
| `src/config/redis` | Leitura e escrita do estado do carrinho |
| `src/config/database` | Validação de produto (findUnique) e estoque |
| `src/utils/AppError` | Erros de negócio (`RESOURCE_NOT_FOUND`, `OUT_OF_STOCK`) |

## Dependências Externas

- `redis` (cliente Node.js)
- `@prisma/client` (via `src/config/database`)

## Módulos Relacionados

- **products** — `cartService` consulta `prisma.product` diretamente (sem passar por `productRepository`)
- **orders** — `orderService` consome `cartService.getCart` no checkout e deleta a chave Redis diretamente

## Pontos de Entrada

- `cartRoutes.js` → registrado no `server.js` em `/api/v1/cart` com `authMiddleware` aplicado antes

## Fluxos Importantes

```
POST /api/v1/cart/items
  authMiddleware (verifica JWT, popula req.user)
    → cartController.addItem (extrai userId de req.user.sub)
      → cartService.addItem
        → prisma.product.findUnique (valida produto)
        → cartService.getCart (estado atual)
        → atualiza/adiciona item + recalcula total
        → redisClient.set(cart:default:{userId}, json, TTL)
        → retorna cart atualizado
```

## Arquivos Críticos

| Arquivo | Descrição |
|---|---|
| `cartService.js` | Toda a lógica; único arquivo com código funcional no módulo |
| `cartController.js` | Tem fallback `req.query.userId` para rotas sem auth — código legado/temporário |

## Observações Técnicas e Débitos

1. **Sem `clearCart`:** não existe método de limpeza do carrinho no `cartService`. O `orderService` deleta a chave Redis diretamente (`redisClient.del`) em vez de delegar ao módulo `cart`, violando o encapsulamento.
2. **`cartController` tem fallback `req.query.userId`** para quando `req.user` não estiver presente — comentário no código indica ser temporário. Com `authMiddleware` aplicado no server.js, esse fallback está inacessível nas rotas configuradas, mas é um código morto que pode causar confusão.
3. **`cartRoutes.js` tem comentário sobre importar `authMiddleware`** — o middleware está sendo aplicado no nível do server.js, não no router. A coerência entre os dois lugares deve ser mantida.
4. **Acesso direto ao Prisma no Service:** `cartService` acessa `prisma.product` diretamente em vez de usar `productRepository`. Acoplamento direto à implementação do Prisma.
5. **`tenantId` fixo em `"default"`:** sem isolamento real por tenant no carrinho.
