# Camada: config

## Objetivo

Instanciar e exportar os clientes de infraestrutura compartilhados por toda a aplicação.

## Arquivos

| Arquivo | Exporta | Descrição |
|---|---|---|
| `database.js` | `prisma` (PrismaClient) | Cliente Prisma configurado com o adapter `PrismaPg` via `DATABASE_URL`. Singleton por módulo Node.js. |
| `redis.js` | `redisClient` | Cliente Redis conectado automaticamente na inicialização. Loga erros e conexão via Winston. |
| `logger.js` | `logger` (Winston) | Logger com níveis `error/warn/info/http/debug`. Grava em `logs/error.log` e `logs/all.log`. Em produção, nível mínimo `warn`. |

## Observações

- **`database.js`** usa `@prisma/adapter-pg` — requer `npx prisma generate` após qualquer alteração no schema.
- **`redis.js`** conecta imediatamente ao ser importado (IIFE assíncrona). Se o Redis não estiver disponível, a aplicação inicia mas todas as operações de Redis lançarão erro.
- **`logger.js`** assume que a pasta `logs/` existe na raiz do projeto. Deve estar no `.gitignore` para não commitar logs.
