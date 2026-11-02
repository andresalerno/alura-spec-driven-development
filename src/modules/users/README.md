# Módulo: users

## Objetivo

Gerenciar perfis de usuários: leitura, atualização e exclusão, com controle de acesso baseado em role.

## Responsabilidade Principal

Expor operações sobre o recurso `User`, garantindo que usuários comuns só manipulem o próprio dado e que ADMINs tenham acesso irrestrito.

## Funcionalidades Existentes

| Rota | Método | Proteção | Descrição |
|---|---|---|---|
| `/api/v1/users/:id` | GET | JWT | Retorna perfil (USER: só o próprio; ADMIN: qualquer) |
| `/api/v1/users/:id` | PUT | JWT | Atualiza usuário (USER: só o próprio; ADMIN: qualquer) |
| `/api/v1/users/:id` | DELETE | JWT | Remove usuário (USER: só o próprio; ADMIN: qualquer) |

### Regras de Acesso

- **GET /:id** — verificação no controller: se não for ADMIN e `id !== req.user.sub`, retorna 403.
- **PUT /:id** — verificação no service: se não for ADMIN e `sub !== userId`, lança `AppError(403)`. Além disso, usuários não-ADMIN não podem alterar o campo `role`.
- **DELETE /:id** — verificação no service: mesma regra do PUT.

### Proteção de senha
`password` é removido do objeto retornado em `getProfile` e `updateUser` com destructuring `{ password, ...userWithoutPassword }`.

## Dependências Internas

| Dependência | Uso |
|---|---|
| `src/config/database` | Queries Prisma (via userRepository) |
| `src/middlewares/authMiddleware` | Proteção de todas as rotas |
| `src/utils/AppError` | Erros de negócio (`FORBIDDEN`, `RESOURCE_NOT_FOUND`) |

## Dependências Externas

- `@prisma/client`

## Módulos Relacionados

- **auth** — `authService` usa `userRepository` diretamente para criar e buscar usuários. O módulo `users` não é chamado pelo `auth`.

## Pontos de Entrada

- `userRoutes.js` → registrado no `server.js` em `/api/v1/users`
- Todas as rotas exigem `authMiddleware`

## Fluxos Importantes

```
PUT /api/v1/users/:id
  authMiddleware (req.user = { sub, role })
    → userController.update(req.params.id, req.body, req.user)
      → userService.updateUser(userId, data, requestUser)
        → verifica se requestUser.role === ADMIN || requestUser.sub === userId
        → userRepository.findById (verifica existência)
        → remove data.role se não for ADMIN
        → userRepository.update(userId, data)
        → retorna { ...user, sem password }
```

## Arquivos Críticos

| Arquivo | Descrição |
|---|---|
| `userService.js` | Regras de autorização e limpeza de dados sensíveis |
| `userRepository.js` | Contém `findAll` que não tem rota exposta |

## Observações Técnicas e Débitos

1. **`userRepository.findAll` sem rota exposta:** existe `findAll(skip, take)` no repository mas não há `GET /` na API. É uma funcionalidade implementada sem consumidor visível. **[Hipótese: rota planejada mas não implementada]**.
2. **Sem rota `/me`:** o padrão REST comum seria `GET /api/v1/users/me` para o usuário obter o próprio perfil. A implementação atual exige que o cliente saiba o próprio `id`.
3. **Verificação de autorização duplicada entre controller e service:** em `getProfile`, a verificação de acesso está no controller; em `updateUser` e `deleteUser`, está no service. Falta padronização.
4. **Sem hash de senha no `updateUser`:** o comentário no código (`// Se houver senha, deveria ser hasheada aqui`) indica que enviar `password` no PUT persiste a senha em texto plano. Débito de segurança grave.
5. **`authService` não usa `userService`:** importa `userRepository` diretamente, criando um acoplamento implícito entre os módulos que contorna as regras de negócio do `userService`.
