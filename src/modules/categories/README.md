# Módulo: categories

## Objetivo

Gerenciar a taxonomia de produtos: criação, consulta, atualização e remoção de categorias, com suporte a hierarquia pai/filho de um nível.

## Responsabilidade Principal

Manter a estrutura de classificação dos produtos, com controle de acesso (somente admins podem escrever).

## Funcionalidades Existentes

| Rota | Método | Proteção | Descrição |
|---|---|---|---|
| `/api/v1/categories` | GET | Pública | Lista todas as categorias com filhos de 1 nível |
| `/api/v1/categories/:id` | GET | Pública | Retorna categoria por ID com filhos |
| `/api/v1/categories` | POST | JWT + ADMIN | Cria nova categoria |
| `/api/v1/categories/:id` | PUT | JWT + ADMIN | Atualiza categoria |
| `/api/v1/categories/:id` | DELETE | JWT + ADMIN | Remove categoria (impede se houver produtos) |

### Hierarquia

O model `Category` tem auto-relação `parent/children` via `parentId`. A listagem retorna filhos de 1 nível via `include: { children: true }`. Recursividade profunda não é suportada.

### Exclusão

Se a categoria tiver produtos associados, o Prisma lança erro `P2003` (FK constraint) que é capturado e convertido em `AppError(409, 'CONFLICT')`.

## Dependências Internas

| Dependência | Uso |
|---|---|
| `src/config/database` | Queries Prisma (via categoryRepository) |
| `src/middlewares/authMiddleware` | Proteção das rotas de escrita |
| `src/utils/AppError` | Erros de negócio |

## Dependências Externas

- `@prisma/client`

## Módulos Relacionados

- **products** — `Product` tem FK `categoryId → Category`. Exclusão de categoria com produtos é bloqueada.

## Pontos de Entrada

- `categoryRoutes.js` → registrado no `server.js` em `/api/v1/categories`

## Fluxos Importantes

```
GET /api/v1/categories
  → categoryController.list
    → categoryService.listCategories
      → categoryRepository.findAll
        → prisma.category.findMany({ include: { children: true } })

DELETE /api/v1/categories/:id (ADMIN)
  authMiddleware → adminOnly
    → categoryController.delete
      → categoryService.deleteCategory(id)
        → categoryRepository.delete
          → prisma.category.delete
            → erro P2003? → AppError(409, CONFLICT)
```

## Arquivos Críticos

| Arquivo | Descrição |
|---|---|
| `categoryService.js` | Lógica e tratamento de FK constraint na exclusão |
| `categoryRepository.js` | Queries com hierarquia (`include: children`) |
| `categoryRoutes.js` | Contém `adminOnly` inline (duplicado de `productRoutes.js`) |

## Observações Técnicas e Débitos

1. **`adminOnly` duplicado:** mesma função definida em `categoryRoutes.js` e `productRoutes.js`. Deve ser extraída para `src/middlewares/adminMiddleware.js`.
2. **Hierarquia limitada a 1 nível:** `findAll` e `findById` usam `include: { children: true }`, que retorna apenas filhos diretos. Não há suporte a subcategorias de múltiplos níveis.
3. **Sem cache:** diferente de produtos, categorias não usam Redis. Sendo dados raramente alterados, cache seria adequado.
4. **Sem validação de payload nas rotas de escrita:** `create` e `update` não têm validators — qualquer body é aceito.
5. **`updateCategory` chama `getCategoryById` para verificar existência** antes do update. Se o update lançar `P2025` (not found), o erro não é tratado explicitamente no Service.
