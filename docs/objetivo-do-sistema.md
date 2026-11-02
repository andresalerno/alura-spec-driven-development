# Objetivo do Sistema

> Documento gerado com base em análise do código-fonte, schema Prisma e estrutura de módulos.  
> Inferências marcadas explicitamente como **[Hipótese]**.

---

## Propósito Principal

API backend para uma plataforma de **e-commerce**, responsável por gerenciar o ciclo completo de compra: cadastro e autenticação de usuários, catálogo de produtos por categoria, carrinho de compras efêmero, processamento de pedidos e pagamentos.

---

## Problemas que o Sistema Resolve

| Problema | Solução no sistema |
|---|---|
| Autenticação segura com sessões curtas | JWT de curta duração (15 min) + refresh token (7 dias) em Redis |
| Consistência de estoque em compras concorrentes | Transação Prisma com decrement atomico no checkout |
| Carrinho temporário sem necessidade de persistência relacional | Armazenamento no Redis com TTL de 30 dias |
| Preço capturado no momento da compra | Snapshot de `price` no `OrderItem`, independente de alterações futuras no `Product` |
| Dados sensíveis (senha) nunca expostos nas respostas | `password` removido explicitamente antes de retornar `User` |
| Operações críticas (confirmar/cancelar pagamento) sem duplicação | Header `Idempotency-Key` obrigatório na confirmação |

---

## Principais Fluxos de Negócio

### 1. Registro e Login

```
POST /api/v1/auth/register
  → valida payload (express-validator)
  → verifica email único
  → hash da senha (bcrypt, 10 rounds)
  → cria usuário no banco
  → gera accessToken (JWT 15min) + refreshToken (JWT 7d)
  → salva refreshToken no Redis (key: refresh:{userId}, TTL 7d)
  → retorna { user, tokens }

POST /api/v1/auth/login
  → valida credenciais
  → compara senha com hash
  → gera e armazena novos tokens (mesmo fluxo acima)
  → retorna { user, tokens }
```

### 2. Gestão do Carrinho

```
GET  /api/v1/cart          → lê chave cart:default:{userId} no Redis
POST /api/v1/cart/items    → valida produto e estoque no banco → atualiza cart no Redis
DELETE /api/v1/cart/items/:itemId  → remove item pelo productId → salva cart atualizado
```

Todas as rotas do carrinho exigem JWT válido (`authMiddleware` aplicado no `server.js`).

### 3. Checkout (Pedido)

```
POST /api/v1/orders
  1. Recupera carrinho do Redis
  2. Para cada item: valida existência, estoque e status ACTIVE do produto (snapshot)
  3. Prisma.$transaction:
     a. Decrementa estoque de cada produto
     b. Cria Order (status: PENDING)
     c. Cria OrderItems com preço snapshot
     d. Cria Payment stub (status: AWAITING_CONFIRMATION)
  4. Deleta chave do carrinho no Redis
  5. Loga evento simulado "order.created"
  → retorna Order criada
```

### 4. Pagamento

```
POST /api/v1/payments/:id/confirm  (header: Idempotency-Key obrigatório)
  → Prisma.$transaction:
     a. Payment.status → PAID
     b. Order.status → PAID
  → loga evento simulado "order.paid"

POST /api/v1/payments/:id/cancel
  → Prisma.$transaction:
     a. Payment.status → CANCELED
     b. Order.status → CANCELED
  ⚠ Não reverte o estoque decrementado no checkout (débito técnico identificado)
```

---

## Atores Envolvidos

| Ator | Role no sistema | Permissões |
|---|---|---|
| **Usuário comum** | `USER` | Gerenciar próprio perfil, carrinho, pedidos e pagamentos próprios |
| **Administrador** | `ADMIN` | Tudo acima + criar/editar/excluir produtos e categorias + visualizar qualquer usuário |
| **Sistema** (interno) | — | Geração de IDs, timestamps, TTLs, logs |

---

## Funcionalidades Centrais

### Implementadas

- [x] Registro e login de usuários com JWT (access + refresh)
- [x] Perfil de usuário (leitura, edição, exclusão) com controle de acesso
- [x] Catálogo de categorias hierárquico (pai/filhos, 1 nível)
- [x] Catálogo de produtos com busca por texto, filtro por categoria e paginação
- [x] Cache de listagem de produtos via Redis
- [x] Carrinho de compras efêmero em Redis por usuário
- [x] Checkout com validação de estoque e snapshot de preço
- [x] Criação automática de `Payment` stub no checkout
- [x] Confirmação e cancelamento de pagamento com transação atômica
- [x] Healthcheck em `GET /health`
- [x] Controle de acesso RBAC (USER / ADMIN)

### Não Implementadas / Débitos

- [ ] Rota de logout (invalidação do refresh token no Redis)
- [ ] Refresh de access token via refresh token
- [ ] Revogação de estoque ao cancelar pagamento
- [ ] `clearCart` no cartService (o clear hoje é feito diretamente no orderService)
- [ ] Documentação Swagger (swagger.yaml não existe; código comentado no server.js)
- [ ] Testes automatizados
- [ ] Invalidação de cache ao criar/editar/excluir produto

---

## Visão de Produto

**[Hipótese]** O sistema está posicionado como backend de um e-commerce B2C com arquitetura preparada para multi-tenancy (campo `tenantId` em Users, Products e Orders). A base de código sugere uma plataforma que poderá suportar múltiplos lojistas ou contextos de negócio isolados no futuro, compartilhando a mesma instância da API.

---

## Contexto Operacional

| Componente | Tecnologia | Função |
|---|---|---|
| Runtime | Node.js 20+ | Execução da API |
| Framework HTTP | Express 5 | Roteamento e middlewares |
| ORM | Prisma 7 + adapter-pg | Acesso ao PostgreSQL |
| Banco de dados | PostgreSQL 15 | Persistência relacional |
| Cache / Sessão | Redis 7 | Carrinho, refresh tokens, cache de produtos |
| Containerização | Docker Compose | Orquestra PostgreSQL e Redis para desenvolvimento |
| Logging | Winston + Morgan | Logs em arquivo e console |
| Autenticação | JWT (jsonwebtoken) | Tokens de acesso e refresh |
| Validação | express-validator | Validação de payload nas rotas |
| Segurança | Helmet + CORS | Headers HTTP de segurança |

**Ponto de entrada:** `src/server.js`  
**Porta padrão:** `3000` (configurável via `PORT` no `.env`)  
**Base das rotas:** `/api/v1/`
