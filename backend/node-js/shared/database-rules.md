# Database Rules

> Backend database rules for Claude Code. Apply to all Node.js backend projects.

## ORM Usage

- Use an ORM (Prisma, TypeORM, Sequelize) or query builder (Knex) — avoid raw SQL unless absolutely necessary:

```ts
// ❌ Incorrect — raw SQL with string interpolation
const user = await db.query(`SELECT * FROM users WHERE id = ${id}`);

// ✅ Correct — ORM
const user = await prisma.user.findUnique({ where: { id } });

// ✅ Correct — raw SQL with parameterized query (when ORM is insufficient)
const user = await prisma.$queryRaw`SELECT * FROM users WHERE id = ${id}`;
```

- Define all models/entities in a dedicated folder:

```
src/
├── database/
│   ├── prisma/
│   │   └── schema.prisma
│   ├── migrations/
│   └── seeds/
```

## Migrations

- **Every database change** must go through a migration — never modify the schema manually:

```bash
# ✅ Correct
npx prisma migrate dev --name add-user-role
npx prisma migrate deploy

# ❌ Incorrect — manual ALTER TABLE in production
```

- Migration naming: use descriptive names with the action:

```
// ❌ Incorrect
migration_001
update_table

// ✅ Correct
add-role-column-to-users
create-orders-table
remove-deprecated-status-field
```

- Migrations must be **reversible** — always consider rollback strategy.
- Never modify an existing migration that has been deployed — create a new one.

## Query Optimization

- **Select only the fields you need** — avoid `SELECT *`:

```ts
// ❌ Incorrect — fetches all columns
const users = await prisma.user.findMany();

// ✅ Correct — select specific fields
const users = await prisma.user.findMany({
  select: { id: true, name: true, email: true },
});
```

- Use **eager loading** to avoid N+1 queries:

```ts
// ❌ Incorrect — N+1 problem
const users = await prisma.user.findMany();
for (const user of users) {
  const orders = await prisma.order.findMany({ where: { userId: user.id } });
}

// ✅ Correct — eager loading with include
const users = await prisma.user.findMany({
  include: { orders: true },
});
```

- Add **indexes** on columns used in WHERE, JOIN, and ORDER BY clauses:

```prisma
model User {
  id    Int    @id @default(autoincrement())
  email String @unique
  name  String

  @@index([name])
  @@index([createdAt])
}
```

## Transactions

- Use transactions for operations that modify multiple tables:

```ts
// ✅ Correct — atomic operation
await prisma.$transaction(async (tx) => {
  const order = await tx.order.create({ data: orderData });
  await tx.orderItem.createMany({ data: items });
  await tx.inventory.update({
    where: { productId },
    data: { quantity: { decrement: 1 } },
  });
});
```

- Never perform partial updates without transactions — either everything succeeds or nothing does.

## Connection Pooling

- Configure connection pool size based on expected load:

```
// .env
DATABASE_URL="postgresql://user:pass@host:5432/db?connection_limit=10"
```

- Always close database connections gracefully on shutdown:

```ts
process.on("SIGTERM", async () => {
  await prisma.$disconnect();
  process.exit(0);
});
```

## Soft Deletes

- Prefer **soft deletes** for user-facing data — add `deletedAt` column instead of removing rows:

```ts
// ✅ Correct — soft delete
await prisma.user.update({
  where: { id },
  data: { deletedAt: new Date() },
});

// Query active records only
const activeUsers = await prisma.user.findMany({
  where: { deletedAt: null },
});
```

## Seeding

- Maintain seed files for development and testing environments:

```ts
// prisma/seed.ts
async function main() {
  await prisma.user.createMany({
    data: [
      { name: "Admin", email: "admin@example.com", role: "ADMIN" },
      { name: "User", email: "user@example.com", role: "USER" },
    ],
  });
}
```

- Never seed production databases with test data.

## Rules

- Never store passwords in plain text — always hash with bcrypt or argon2.
- Use UUIDs instead of auto-increment IDs for public-facing resources.
- Always validate data before writing to the database — the DB is the last line of defense, not the first.
- Keep migration files in version control — they are part of the codebase.
- Monitor slow queries in production — set up query logging with execution time thresholds.
- Back up production databases regularly and test restore procedures.
