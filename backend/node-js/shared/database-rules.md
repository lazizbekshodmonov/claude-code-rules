# Database Rules

> Backend database rules for Claude Code. Apply to all Node.js backend projects.

## ORM Usage

- Use an ORM (TypeORM, Prisma, Sequelize) or query builder (Knex) — avoid raw SQL unless absolutely necessary:

```ts
// ❌ Incorrect — raw SQL with string interpolation
const user = await db.query(`SELECT * FROM users WHERE id = ${id}`);

// ✅ Correct — TypeORM
const user = await userRepository.findOneBy({ id });

// ✅ Correct — Prisma
const user = await prisma.user.findUnique({ where: { id } });

// ✅ Correct — raw SQL with parameterized query (when ORM is insufficient)
const user = await dataSource.query("SELECT * FROM users WHERE id = $1", [id]);
```

- Define all models/entities in a dedicated folder:

```
src/
├── database/
│   ├── entities/             ← TypeORM entities
│   │   ├── user.entity.ts
│   │   └── order.entity.ts
│   ├── migrations/
│   ├── seeds/
│   └── data-source.ts       ← TypeORM DataSource configuration
```

## Entity Definition (TypeORM)

- Use decorators to define entities with proper column types, relations, and indexes:

```ts
@Entity("users")
export class User {
  @PrimaryGeneratedColumn("uuid")
  id: string;

  @Column({ type: "varchar", length: 100 })
  name: string;

  @Column({ type: "varchar", unique: true })
  email: string;

  @Column({ type: "enum", enum: UserRole, default: UserRole.USER })
  role: UserRole;

  @Column({ type: "boolean", default: true })
  isActive: boolean;

  @DeleteDateColumn()
  deletedAt: Date | null;

  @CreateDateColumn()
  createdAt: Date;

  @UpdateDateColumn()
  updatedAt: Date;

  @OneToMany(() => Order, (order) => order.user)
  orders: Order[];

  @Index()
  @Column({ type: "varchar", nullable: true })
  phone: string;
}
```

- Always use `@CreateDateColumn()`, `@UpdateDateColumn()` — do not manage timestamps manually.
- Use `@DeleteDateColumn()` for soft deletes — TypeORM automatically filters soft-deleted records.
- Add `@Index()` on columns used in WHERE, JOIN, and ORDER BY clauses.

## Repository Pattern (TypeORM)

- Use the **Repository pattern** — never call `dataSource.manager` directly from services:

```ts
// ❌ Incorrect — direct manager usage in service
async findById(id: string): Promise<User> {
  return this.dataSource.manager.findOneBy(User, { id });
}

// ✅ Correct — inject repository (NestJS)
@Injectable()
export class UsersService {
  constructor(
    @InjectRepository(User)
    private readonly userRepository: Repository<User>,
  ) {}

  async findById(id: string): Promise<User> {
    return this.userRepository.findOneBy({ id });
  }
}
```

- For complex queries, use **custom repositories**:

```ts
@Injectable()
export class UsersRepository extends Repository<User> {
  constructor(private readonly dataSource: DataSource) {
    super(User, dataSource.createEntityManager());
  }

  async findActiveByRole(role: UserRole): Promise<User[]> {
    return this.find({
      where: { role, isActive: true },
      order: { createdAt: "DESC" },
    });
  }
}
```

## Migrations

- **Every database change** must go through a migration — never modify the schema manually:

```bash
# ✅ Correct — TypeORM
npx typeorm migration:generate src/database/migrations/AddUserRole -d src/database/data-source.ts
npx typeorm migration:run -d src/database/data-source.ts

# ✅ Correct — Prisma
npx prisma migrate dev --name add-user-role
npx prisma migrate deploy

# ❌ Incorrect — manual ALTER TABLE in production
```

- Migration naming: use descriptive names with the action:

```
// ❌ Incorrect
Migration001
UpdateTable

// ✅ Correct
AddRoleColumnToUsers
CreateOrdersTable
RemoveDeprecatedStatusField
```

- Migrations must be **reversible** — always implement both `up()` and `down()`:

```ts
export class AddRoleColumnToUsers1700000000000 implements MigrationInterface {
  async up(queryRunner: QueryRunner): Promise<void> {
    await queryRunner.addColumn("users", new TableColumn({
      name: "role",
      type: "enum",
      enum: ["user", "admin", "moderator"],
      default: "'user'",
    }));
  }

  async down(queryRunner: QueryRunner): Promise<void> {
    await queryRunner.dropColumn("users", "role");
  }
}
```

- Never modify an existing migration that has been deployed — create a new one.

## Query Optimization

- **Select only the fields you need** — avoid `SELECT *`:

```ts
// ❌ Incorrect — fetches all columns
const users = await userRepository.find();

// ✅ Correct — TypeORM: select specific fields
const users = await userRepository.find({
  select: { id: true, name: true, email: true },
});

// ✅ Correct — Prisma: select specific fields
const users = await prisma.user.findMany({
  select: { id: true, name: true, email: true },
});
```

- Use **eager loading** to avoid N+1 queries:

```ts
// ❌ Incorrect — N+1 problem
const users = await userRepository.find();
for (const user of users) {
  const orders = await orderRepository.findBy({ userId: user.id });
}

// ✅ Correct — TypeORM: eager loading with relations
const users = await userRepository.find({
  relations: { orders: true },
});

// ✅ Correct — TypeORM: QueryBuilder for complex joins
const users = await userRepository
  .createQueryBuilder("user")
  .leftJoinAndSelect("user.orders", "order")
  .where("user.isActive = :isActive", { isActive: true })
  .orderBy("user.createdAt", "DESC")
  .getMany();

// ✅ Correct — Prisma: eager loading with include
const users = await prisma.user.findMany({
  include: { orders: true },
});
```

## Transactions

- Use transactions for operations that modify multiple tables:

```ts
// ✅ Correct — TypeORM: transaction with QueryRunner
const queryRunner = dataSource.createQueryRunner();
await queryRunner.connect();
await queryRunner.startTransaction();
try {
  const order = await queryRunner.manager.save(Order, orderData);
  await queryRunner.manager.save(OrderItem, items);
  await queryRunner.manager.decrement(Inventory, { productId }, "quantity", 1);
  await queryRunner.commitTransaction();
} catch (error) {
  await queryRunner.rollbackTransaction();
  throw error;
} finally {
  await queryRunner.release();
}

// ✅ Correct — TypeORM: transaction with dataSource.transaction
await dataSource.transaction(async (manager) => {
  const order = await manager.save(Order, orderData);
  await manager.save(OrderItem, items);
  await manager.decrement(Inventory, { productId }, "quantity", 1);
});

// ✅ Correct — Prisma: transaction
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

```ts
// TypeORM DataSource
export const dataSource = new DataSource({
  type: "postgres",
  url: process.env.DATABASE_URL,
  extra: {
    max: 20,           // max connections in pool
    idleTimeoutMillis: 30000,
  },
  entities: [User, Order],
  migrations: ["src/database/migrations/*.ts"],
});
```

- Always close database connections gracefully on shutdown:

```ts
process.on("SIGTERM", async () => {
  await dataSource.destroy();   // TypeORM
  // await prisma.$disconnect(); // Prisma
  process.exit(0);
});
```

## Soft Deletes

- Prefer **soft deletes** for user-facing data:

```ts
// ✅ Correct — TypeORM: use @DeleteDateColumn (automatic filtering)
// Entity has @DeleteDateColumn() deletedAt

await userRepository.softDelete(id);        // sets deletedAt
await userRepository.restore(id);           // clears deletedAt

// Active records returned automatically (soft-deleted excluded)
const activeUsers = await userRepository.find();

// Include soft-deleted records
const allUsers = await userRepository.find({ withDeleted: true });

// ✅ Correct — Prisma: manual soft delete
await prisma.user.update({
  where: { id },
  data: { deletedAt: new Date() },
});
const activeUsers = await prisma.user.findMany({
  where: { deletedAt: null },
});
```

## Seeding

- Maintain seed files for development and testing environments:

```ts
// src/database/seeds/user.seed.ts
export async function seedUsers(dataSource: DataSource) {
  const userRepository = dataSource.getRepository(User);
  await userRepository.save([
    { name: "Admin", email: "admin@example.com", role: UserRole.ADMIN },
    { name: "User", email: "user@example.com", role: UserRole.USER },
  ]);
}
```

- Never seed production databases with test data.

## QueryBuilder Best Practices (TypeORM)

- Use QueryBuilder for complex queries with dynamic conditions:

```ts
async findAll(query: UserQueryDto): Promise<[User[], number]> {
  const qb = this.userRepository.createQueryBuilder("user");

  if (query.search) {
    qb.andWhere("(user.name ILIKE :search OR user.email ILIKE :search)", {
      search: `%${query.search}%`,
    });
  }

  if (query.role) {
    qb.andWhere("user.role = :role", { role: query.role });
  }

  return qb
    .orderBy(`user.${query.sortBy || "createdAt"}`, query.order || "DESC")
    .skip((query.page - 1) * query.limit)
    .take(query.limit)
    .getManyAndCount();
}
```

- Always use **parameterized queries** in QueryBuilder — never interpolate values:

```ts
// ❌ Incorrect — SQL injection risk
qb.where(`user.email = '${email}'`);

// ✅ Correct — parameterized
qb.where("user.email = :email", { email });
```

## Rules

- Never store passwords in plain text — always hash with bcrypt or argon2.
- Use UUIDs instead of auto-increment IDs for public-facing resources.
- Always validate data before writing to the database — the DB is the last line of defense, not the first.
- Keep migration files in version control — they are part of the codebase.
- Always implement both `up()` and `down()` in migrations for safe rollbacks.
- Monitor slow queries in production — set up query logging with execution time thresholds.
- Back up production databases regularly and test restore procedures.
