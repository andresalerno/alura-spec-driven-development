# Módulo: auth

## Objetivo

Gerenciar autenticação de usuários: registro de novas contas e login com emissão de tokens JWT (access + refresh).

## Responsabilidade Principal

Verificar identidade, criar sessões via tokens e armazenar o refresh token no Redis para controle de validade.

## Funcionalidades Existentes

| Rota | Método | Proteção | Descrição |
|---|---|---|---|
| `/api/v1/auth/register` | POST | Pública | Registra novo usuário e retorna tokens |
| `/api/v1/auth/login` | POST | Pública | Autentica usuário e retorna tokens |

### Lógica do Register
1. Valida payload via `registerValidator` (nome, e-mail, senha ≥ 6 chars).
2. Verifica unicidade do e-mail no banco.
3. Faz hash da senha com bcrypt (10 rounds).
4. Cria o usuário via `userRepository.create`.
5. Gera `accessToken` (JWT, expira em `JWT_ACCESS_EXPIRATION`) e `refreshToken` (JWT, expira em `JWT_REFRESH_EXPIRATION`).
6. Salva `refreshToken` no Redis na chave `refresh:{userId}` com TTL de 7 dias.
7. Retorna `{ user: { id, name, email, role }, tokens: { accessToken, refreshToken } }`.

### Lógica do Login
Mesmo fluxo a partir do passo 5, após verificar credenciais com `bcrypt.compare`.

## Dependências Internas

| Dependência | Uso |
|---|---|
| `src/modules/users/userRepository` | Busca e criação de usuário no banco |
| `src/utils/token` | Geração e verificação de JWT |
| `src/utils/AppError` | Erros de negócio (`CONFLICT`, `UNAUTHORIZED`) |
| `src/config/redis` | Armazenamento do refresh token |
| `src/middlewares/validateRequest` | Validação de payload |

## Dependências Externas

- `bcrypt` — hash e comparação de senhas
- `jsonwebtoken` — geração e verificação de tokens
- `express-validator` — validação de campos do body

## Módulos Relacionados

- **users** — `auth` consome `userRepository` diretamente, sem passar pelo `userService`
- **middlewares/authMiddleware** — usa `verifyAccessToken` do mesmo `utils/token` para proteger outras rotas

## Pontos de Entrada

- `authRoutes.js` → registrado no `server.js` em `/api/v1/auth`

## Fluxos Importantes

```
POST /register
  authValidators.registerValidator
    → validateRequest
      → authController.register
        → authService.register
          → userRepository.findByEmail (verifica duplicidade)
          → bcrypt.hash
          → userRepository.create
          → token.generateAccessToken + generateRefreshToken
          → redisClient.set(refresh:{id})
          → { user, tokens }
```

## Arquivos Críticos

| Arquivo | Descrição |
|---|---|
| `authService.js` | Toda a lógica de negócio; ponto central do módulo |
| `authValidators.js` | Regras de validação de entrada |
| `authRoutes.js` | Declaração das rotas públicas |

## Observações Técnicas e Débitos

1. **Sem rota de logout:** o refresh token é criado mas nunca invalidado explicitamente. Para logout, é necessário implementar `redisClient.del('refresh:{userId}')`.
2. **Sem rota de refresh:** não há endpoint para obter um novo access token via refresh token. A lógica de `verifyRefreshToken` existe em `utils/token.js` mas não é usada.
3. **`role` aceito no registro:** o body de register aceita um campo `role` sem validação. Um usuário pode se auto-promover para `ADMIN` diretamente. Débito de segurança importante.
4. **`authService` importa `userRepository` diretamente**, acoplando auth à implementação interna de users. O ideal seria passar pelo `userService` ou expor uma interface de domínio.
