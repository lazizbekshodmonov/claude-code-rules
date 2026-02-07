# Database Rules (Express)

> Express-specific database rules using TypeORM with manual dependency injection.

## DataSource Configuration

```ts
// src/database/data-source.ts
import { DataSource } from "typeorm";

export const dataSource = new DataSource({
  type: "postgres",
  url: process.env.DATABASE_URL,
  entities: [__dirname + "/../**/*.entity{.ts,.js}"],
  migrations: [__dirname + "/migrations/*{.ts,.js}"],
  extra: { max: 20, idleTimeoutMillis: 30000 },
  synchronize: false, // NEVER true in production
  logging: process.env.NODE_ENV === "development",
});

// Initialize on app start
export async function initDatabase(): Promise<void> {
  await dataSource.initialize();
  console.log("Database connected");
}
```

## Entity Definition

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

## Repository Access

- Get repositories from the DataSource — create a service layer:

```ts
// services/user.service.ts
import { dataSource } from "../database/data-source";

const userRepository = dataSource.getRepository(User);

export const userService = {
  async findById(id: string): Promise<User> {
    const user = await userRepository.findOneBy({ id });
    if (!user) throw new NotFoundError("User", id);
    return user;
  },

  async findAll(query: UserQueryDto): Promise<[User[], number]> {
    return userRepository.findAndCount({
      skip: (query.page - 1) * query.limit,
      take: query.limit,
      order: { createdAt: "DESC" },
    });
  },

  async create(data: CreateUserDto): Promise<User> {
    const exists = await userRepository.findOneBy({ email: data.email });
    if (exists) throw new ConflictError("Email already registered");
    return userRepository.save(data);
  },

  async update(id: string, data: UpdateUserDto): Promise<User> {
    const user = await userService.findById(id);
    Object.assign(user, data);
    return userRepository.save(user);
  },

  async delete(id: string): Promise<void> {
    await userRepository.softDelete(id);
  },
};
```

## Query Optimization

```ts
// Select only required fields
const users = await userRepository.find({
  select: { id: true, name: true, email: true },
});

// Eager loading — avoid N+1
const users = await userRepository.find({
  relations: { orders: true },
});

// QueryBuilder for complex queries
const users = await userRepository
  .createQueryBuilder("user")
  .leftJoinAndSelect("user.orders", "order")
  .where("user.isActive = :isActive", { isActive: true })
  .orderBy("user.createdAt", "DESC")
  .skip(offset)
  .take(limit)
  .getManyAndCount();
```

## Transactions

```ts
async function createOrder(dto: CreateOrderDto): Promise<Order> {
  return dataSource.transaction(async (manager) => {
    const order = await manager.save(Order, dto);
    await manager.save(OrderItem, dto.items);
    await manager.decrement(Inventory, { productId: dto.productId }, "quantity", 1);
    return order;
  });
}
```

## Migrations

```bash
npx typeorm migration:generate src/database/migrations/AddUserRole -d src/database/data-source.ts
npx typeorm migration:run -d src/database/data-source.ts
npx typeorm migration:revert -d src/database/data-source.ts
```

## Soft Deletes

```ts
await userRepository.softDelete(id);
await userRepository.restore(id);

// Active records only (default)
const users = await userRepository.find();

// Include soft-deleted
const all = await userRepository.find({ withDeleted: true });
```

## Graceful Shutdown

```ts
process.on("SIGTERM", async () => {
  await dataSource.destroy();
  process.exit(0);
});
```

## Rules

- Never set `synchronize: true` in production.
- Always use the service layer — never access repositories directly in routes.
- Use transactions for multi-table writes.
- Paginate all list queries — never return unbounded collections.
- Select only needed fields.
- Use parameterized queries in QueryBuilder — never interpolate values.
