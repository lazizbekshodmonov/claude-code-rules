# Database Rules (NestJS)

> NestJS-specific database rules using TypeORM with the Repository pattern.

## Entity Definition

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

## TypeORM Module Registration

- Register TypeORM in the root module with `forRoot`, entities in feature modules with `forFeature`:

```ts
// app.module.ts
@Module({
  imports: [
    TypeOrmModule.forRoot({
      type: "postgres",
      url: process.env.DATABASE_URL,
      entities: [__dirname + "/**/*.entity{.ts,.js}"],
      migrations: [__dirname + "/database/migrations/*{.ts,.js}"],
      extra: { max: 20 },
      synchronize: false, // NEVER true in production
    }),
    UsersModule,
  ],
})
export class AppModule {}

// users.module.ts
@Module({
  imports: [TypeOrmModule.forFeature([User])],
  providers: [UsersService],
  controllers: [UsersController],
})
export class UsersModule {}
```

- **NEVER** set `synchronize: true` in production — always use migrations.

## Repository Pattern

- Inject repositories via `@InjectRepository` — never use `dataSource.manager` directly:

```ts
// ❌ Incorrect
@Injectable()
export class UsersService {
  constructor(private readonly dataSource: DataSource) {}

  async findById(id: string) {
    return this.dataSource.manager.findOneBy(User, { id });
  }
}

// ✅ Correct
@Injectable()
export class UsersService {
  constructor(
    @InjectRepository(User)
    private readonly userRepository: Repository<User>,
  ) {}

  async findById(id: string): Promise<User> {
    const user = await this.userRepository.findOneBy({ id });
    if (!user) throw new NotFoundException(`User ${id} not found`);
    return user;
  }
}
```

## Custom Repositories

- For complex queries, create custom repository classes:

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

  async findWithOrders(id: string): Promise<User> {
    return this.findOne({
      where: { id },
      relations: { orders: true },
    });
  }
}

// Register in module
@Module({
  imports: [TypeOrmModule.forFeature([User])],
  providers: [UsersService, UsersRepository],
})
export class UsersModule {}
```

## Query Optimization

- Select only required fields:

```ts
// ❌ Incorrect — fetches all columns
const users = await this.userRepository.find();

// ✅ Correct
const users = await this.userRepository.find({
  select: { id: true, name: true, email: true },
});
```

- Use `relations` or QueryBuilder for eager loading — avoid N+1:

```ts
// ❌ Incorrect — N+1
const users = await this.userRepository.find();
for (const user of users) {
  user.orders = await this.orderRepository.findBy({ userId: user.id });
}

// ✅ Correct — relations
const users = await this.userRepository.find({
  relations: { orders: true },
});

// ✅ Correct — QueryBuilder for complex joins
const users = await this.userRepository
  .createQueryBuilder("user")
  .leftJoinAndSelect("user.orders", "order")
  .where("user.isActive = :isActive", { isActive: true })
  .orderBy("user.createdAt", "DESC")
  .getMany();
```

## QueryBuilder Best Practices

- Use QueryBuilder for dynamic filtering and pagination:

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

- Always use **parameterized values** — never interpolate:

```ts
// ❌ Incorrect
qb.where(`user.email = '${email}'`);

// ✅ Correct
qb.where("user.email = :email", { email });
```

## Migrations

- Generate and run migrations via TypeORM CLI:

```bash
# Generate migration from entity changes
npx typeorm migration:generate src/database/migrations/AddUserRole -d src/database/data-source.ts

# Run migrations
npx typeorm migration:run -d src/database/data-source.ts

# Revert last migration
npx typeorm migration:revert -d src/database/data-source.ts
```

- Always implement both `up()` and `down()`:

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

## Transactions

- Use `QueryRunner` or `dataSource.transaction` for multi-table operations:

```ts
// ✅ QueryRunner — manual control
async createOrder(dto: CreateOrderDto): Promise<Order> {
  const queryRunner = this.dataSource.createQueryRunner();
  await queryRunner.connect();
  await queryRunner.startTransaction();
  try {
    const order = await queryRunner.manager.save(Order, dto);
    await queryRunner.manager.save(OrderItem, dto.items);
    await queryRunner.manager.decrement(Inventory, { productId: dto.productId }, "quantity", 1);
    await queryRunner.commitTransaction();
    return order;
  } catch (error) {
    await queryRunner.rollbackTransaction();
    throw error;
  } finally {
    await queryRunner.release();
  }
}

// ✅ dataSource.transaction — simpler
async createOrder(dto: CreateOrderDto): Promise<Order> {
  return this.dataSource.transaction(async (manager) => {
    const order = await manager.save(Order, dto);
    await manager.save(OrderItem, dto.items);
    return order;
  });
}
```

## Soft Deletes

```ts
// Entity uses @DeleteDateColumn()
await this.userRepository.softDelete(id);
await this.userRepository.restore(id);

// Active records only (default)
const users = await this.userRepository.find();

// Include soft-deleted
const all = await this.userRepository.find({ withDeleted: true });
```

## Graceful Shutdown

```ts
// main.ts
app.enableShutdownHooks();

// any.module.ts
@Injectable()
export class AppService implements OnModuleDestroy {
  async onModuleDestroy() {
    // cleanup connections
  }
}
```

## Rules

- Never set `synchronize: true` in production — use migrations only.
- Always implement `up()` and `down()` in migrations.
- Use the Repository pattern — inject via `@InjectRepository`.
- Use UUIDs for public-facing IDs, auto-increment for internal.
- Select only needed fields — avoid loading entire entities when unnecessary.
- Use transactions for multi-table writes — partial updates cause data corruption.
- Monitor slow queries — enable TypeORM query logging with duration threshold.
