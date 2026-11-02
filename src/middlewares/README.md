# Camada: middlewares

## Objetivo

Comportamentos transversais aplicados ao pipeline do Express: autenticação, validação de payload, logging HTTP e tratamento centralizado de erros.

## Arquivos

| Arquivo | Função | Registrado em |
|---|---|---|
| `authMiddleware.js` | Extrai e verifica o JWT do header `Authorization: Bearer`. Popula `req.user = { sub, role }`. | `server.js` (cart) e `*Routes.js` (orders, payments, users) |
| `validateRequest.js` | Lê erros acumulados pelo `express-validator` e lança `AppError(400)` com mapa de campos inválidos. | Imediatamente após validators em `authRoutes.js` |
| `errorHandler.js` | Middleware de erro global. Formata resposta `{ error: { code, message, details } }` e loga. | Último middleware em `server.js` |
| `httpLogger.js` | Morgan formatado para Winston. Só ativo em `development`. | `server.js` antes das rotas |
| `requestLogger.js` | Middleware simples com `console.log`. **Não registrado no `server.js`. Arquivo órfão.** | — |

## Observações

- **`authMiddleware`** lança `AppError` com `throw` (sem `next()`). Isso funciona no Express 5 (que captura erros síncronos em middlewares), mas é incompatível com Express 4.
- **`requestLogger.js`** está presente no código mas não importado em nenhum lugar. Pode ser removido com segurança ou substituído pelo `httpLogger.js`.
- **`validateRequest.js`** deve sempre ser usado imediatamente após os arrays de validators do `express-validator`, nunca sozinho.
