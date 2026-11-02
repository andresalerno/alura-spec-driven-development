# Módulo: payments

## Objetivo

Gerenciar o ciclo de vida de pagamentos: criação de intenção de pagamento, confirmação e cancelamento, com garantias de idempotência.

## Responsabilidade Principal

Atualizar o status de `Payment` e `Order` de forma atômica em resposta a ações de confirmação ou cancelamento.

## Funcionalidades Existentes

| Rota | Método | Proteção | Descrição |
|---|---|---|---|
| `/api/v1/payments` | POST | JWT | Cria intenção de pagamento (idempotente) |
| `/api/v1/payments/:id/confirm` | POST | JWT | Confirma pagamento (requer `Idempotency-Key`) |
| `/api/v1/payments/:id/cancel` | POST | JWT | Cancela pagamento e pedido |

### Criação (`POST /`)
- Verifica se já existe um `Payment` para o `orderId` informado.
- Se existir, retorna o existente (idempotência).
- Se não existir, cria com status `AWAITING_CONFIRMATION`.
- **Nota:** normalmente o `Payment` já é criado automaticamente no checkout. Esta rota existe para casos em que o stub não foi criado.

### Confirmação (`POST /:id/confirm`)
- Exige header `Idempotency-Key`.
- Se `Payment.status === PAID`, retorna o estado atual sem reprocessar (idempotência).
- Caso contrário, executa `prisma.$transaction`:
  - `Payment.status → PAID`
  - `Order.status → PAID`
- Loga evento simulado `order.paid`.

### Cancelamento (`POST /:id/cancel`)
- Executa `prisma.$transaction`:
  - `Payment.status → CANCELED`
  - `Order.status → CANCELED`
- ⚠ **Não reverte o estoque** decrementado no checkout.

## Dependências Internas

| Dependência | Uso |
|---|---|
| `src/config/database` | Importado diretamente para `prisma.$transaction` entre Payment e Order |
| `src/config/logger` | Log de evento simulado `order.paid` |
| `src/utils/AppError` | Erros de negócio (`RESOURCE_NOT_FOUND`, `INVALID_PAYLOAD`) |

## Dependências Externas

- `@prisma/client`

## Módulos Relacionados

- **orders** — `paymentService` atualiza `Order.status` diretamente via `prisma.$transaction`, sem passar pelo `orderRepository`. Acoplamento cross-module via banco.

## Pontos de Entrada

- `paymentRoutes.js` → registrado no `server.js` em `/api/v1/payments`
- Todas as rotas exigem `authMiddleware`

## Fluxos Importantes

```
POST /api/v1/payments/:id/confirm
  authMiddleware
    → paymentController.confirm (extrai idempotencyKey do header)
      → paymentService.confirmPayment(paymentId, idempotencyKey)
        → verifica idempotencyKey presente
        → paymentRepository.findById (busca payment + order)
        → se PAID: retorna (idempotência)
        → prisma.$transaction:
            ├── payment.update { status: PAID }
            └── order.update { status: PAID }
        → logger.info("order.paid")
        → retorna payment atualizado
```

## Arquivos Críticos

| Arquivo | Descrição |
|---|---|
| `paymentService.js` | Lógica de idempotência, transações e atualização cross-model |
| `paymentRepository.js` | Queries CRUD sobre `Payment` |

## Observações Técnicas e Débitos

1. **Estoque não revertido no cancelamento:** `cancelPayment` cancela `Payment` e `Order`, mas não devolve o estoque decrementado no checkout. Um cancelamento não libera os produtos para outros usuários.
2. **`paymentService` acessa `prisma` diretamente** para atualizar `Order` — viola a camada de Repository do módulo Orders. O correto seria `orderRepository.updateStatus(orderId, status, tx)`.
3. **`Payment.status` é `String` no schema Prisma,** não um `enum`. Os valores usados (`AWAITING_CONFIRMATION`, `PAID`, `CANCELED`) são strings literais sem garantia de tipo no banco. Divergência do restante do schema que usa `enum`.
4. **Idempotência simples por status:** a idempotência da confirmação verifica apenas se `status === PAID`. O `Idempotency-Key` do header é recebido mas não armazenado nem verificado contra duplicatas reais — um segundo request com chave diferente seria reprocessado se o status não for `PAID`.
5. **Sem controle de autorização por usuário:** qualquer usuário autenticado pode confirmar ou cancelar qualquer pagamento pelo `id`. Não há verificação se o pagamento pertence ao usuário que faz a requisição.
