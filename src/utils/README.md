# Camada: utils

## Objetivo

Utilitários reutilizáveis sem dependência de infraestrutura ou domínio.

## Arquivos

### `AppError.js`

Classe que estende `Error` com campos extras para a API:

| Campo | Tipo | Descrição |
|---|---|---|
| `message` | string | Mensagem legível |
| `statusCode` | number | HTTP status (padrão: 500) |
| `code` | string | Código semântico (ex: `CONFLICT`, `UNAUTHORIZED`) |
| `details` | object | Mapa de erros de campo (usado por `validateRequest`) |
| `isOperational` | boolean | Sempre `true` — indica erro esperado, não bug |

Usada em todos os Services e Middlewares para erros de negócio. O `errorHandler` formata qualquer `AppError` na resposta HTTP.

### `token.js`

Funções de geração e verificação de JWT, sem estado:

| Função | Descrição |
|---|---|
| `generateAccessToken(userId, role)` | JWT assinado com `JWT_ACCESS_SECRET`, expira em `JWT_ACCESS_EXPIRATION` |
| `generateRefreshToken(userId)` | JWT assinado com `JWT_REFRESH_SECRET`, expira em `JWT_REFRESH_EXPIRATION` |
| `verifyAccessToken(token)` | Verifica e decodifica access token. Lança `JsonWebTokenError` se inválido. |
| `verifyRefreshToken(token)` | Verifica e decodifica refresh token. **Não usada atualmente** (sem rota de refresh). |

## Observações

- `verifyRefreshToken` está implementada mas sem consumidor no código atual. Necessária quando a rota de refresh for implementada.
- `AppError` com `isOperational: true` pode ser usada futuramente para distinguir erros tratáveis de bugs inesperados no `errorHandler`.
