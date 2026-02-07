# Database Rules (NestJS)

> NestJS-specific database rules using TypeORM with custom repository pattern, BaseEntity, and mappers.

## BaseEntity

- All entities extend the abstract `BaseEntity` — never define `id`, `createdAt`, `updatedAt`, `deletedAt` manually:

```ts
// common/entities/base.entity.ts
export abstract class BaseEntity {
  @PrimaryGeneratedColumn()
  id: number;

  @CreateDateColumn({ name: "created_at", type: "timestamptz" })
  readonly createdAt: Date;

  @UpdateDateColumn({ name: "updated_at", type: "timestamptz" })
  readonly updatedAt: Date;

  @DeleteDateColumn({ name: "deleted_at", type: "timestamptz", nullable: true })
  deletedAt: Date | null = null;
}
```

## Entity Definition

- Use `@Column({ name: "snake_case" })` for all columns — DB uses `snake_case`, TypeScript uses `camelCase`:

```ts
@Entity("users")
@Unique(["email"])
export class UserEntity extends BaseEntity {
  @Column({ name: "name", type: "varchar", length: 150 })
  name: string;

  @Column({ name: "email", type: "varchar", nullable: false })
  email: string;

  @Column({ name: "password_hash", type: "varchar", nullable: true })
  passwordHash: string | null;

  @Column({
    name: "role",
    type: "enum",
    enum: UserRole,
    enumName: "user_role_enum",
    default: UserRole.USER,
  })
  role: UserRole;

  @Column({
    name: "status",
    type: "enum",
    enum: UserStatus,
    enumName: "user_status_enum",
    default: UserStatus.ACTIVE,
  })
  status: UserStatus;
}
```

- Always provide `enumName` for enum columns — prevents duplicate type errors in migrations.
- Use `@Unique()` at entity level — not `unique: true` on individual columns.
- Entity class naming: `<Name>Entity` (e.g., `UserEntity`, `CompanyEntity`).

## TypeORM Module Registration

```ts
// app.module.ts
TypeOrmModule.forRootAsync({
  inject: [ConfigService],
  useFactory: (configService: ConfigService) => ({
    type: "postgres",
    host: configService.get<string>("database.postgres.host"),
    port: configService.get<number>("database.postgres.port"),
    username: configService.get<string>("database.postgres.username"),
    password: configService.get<string>("database.postgres.password"),
    database: configService.get<string>("database.postgres.name"),
    entities: [__dirname + "/**/*.entity{.ts,.js}"],
    migrations: [__dirname + "/database/migrations/*{.ts,.js}"],
    synchronize: false, // NEVER true in production
  }),
}),
```

- **NEVER** set `synchronize: true` in production — always use migrations.

## Custom Repository Pattern

- **Never** use `@InjectRepository()`. Create custom repository classes:

```ts
// ❌ Incorrect — @InjectRepository
@Injectable()
export class UserService {
  constructor(
    @InjectRepository(UserEntity)
    private readonly userRepository: Repository<UserEntity>,
  ) {}
}

// ✅ Correct — custom repository
@Injectable()
export class UserRepository extends Repository<UserEntity> {
  constructor(private dataSource: DataSource) {
    super(UserEntity, dataSource.createEntityManager());
  }

  async findByUsername(email: string): Promise<UserEntity | null> {
    return this.findOne({ where: { email } });
  }

  async findOfPagination(query: UserRequestQueryDto): Promise<[UserEntity[], number]> {
    const { page = 1, size = 10, search, status, role } = query;

    const qb = this.createQueryBuilder("user")
      .orderBy("user.id", "ASC")
      .skip((page - 1) * size)
      .take(size);

    if (status) {
      qb.andWhere("user.status = :status", { status });
    }

    if (role) {
      qb.andWhere("user.role = :role", { role });
    }

    if (search) {
      qb.andWhere("(user.name ILIKE :search OR user.email ILIKE :search)", {
        search: `%${search.trim()}%`,
      });
    }

    return qb.getManyAndCount();
  }
}
```

- Register repositories as **providers** in the module (not via `TypeOrmModule.forFeature`):

```ts
@Module({
  providers: [UserService, UserRepository],
  controllers: [UserController],
  exports: [UserService, UserRepository],
})
export class UserModule {}
```

## Mapper Pattern

- Use static mapper classes for Entity ↔ DTO conversion:

```ts
export class UserMapper {
  public static toDto(this: void, entity: UserEntity): UserResponseDto {
    const dto = new UserResponseDto();
    dto.id = entity.id;
    dto.name = entity.name;
    dto.email = entity.email;
    dto.role = entity.role;
    dto.status = entity.status;
    return dto;
  }

  public static toCreateEntity(this: void, dto: UserCreateDto, role: UserRole): UserEntity {
    const user = new UserEntity();
    user.name = dto.name;
    user.email = dto.email;
    user.role = role;
    user.passwordHash = dto.password;
    user.status = UserStatus.ACTIVE;
    return user;
  }

  public static toUpdateEntity(dto: UserUpdateDto, entity: UserEntity): UserEntity {
    const user = new UserEntity();
    user.name = dto.name ?? entity.name;
    user.email = dto.email ?? entity.email;
    return user;
  }
}
```

- Use `this: void` on static methods to prevent accidental `this` usage.
- Create a mapper per module in `mappers/<name>.mapper.ts`.
- Never convert entities to DTOs inline in services/controllers.

## QueryBuilder Best Practices

- Use QueryBuilder for dynamic filtering, pagination, and complex joins:

```ts
async findOfPagination(query: CompanyRequestDto): Promise<[CompanyEntity[], number]> {
  const { page = 1, size = 10, search, status } = query;

  const qb = this.createQueryBuilder("company")
    .select(["company.id", "company.name", "company.description", "company.status", "company.createdAt"])
    .leftJoin("company.logo", "logo")
    .addSelect(["logo.publicUrl"])
    .skip((page - 1) * size)
    .take(size);

  if (status) {
    qb.andWhere("company.status = :status", { status });
  }

  if (search) {
    qb.andWhere("(company.name ILIKE :search)", {
      search: `%${search.trim()}%`,
    });
  }

  return qb.getManyAndCount();
}
```

- Always use **parameterized values** — never interpolate:

```ts
// ❌ Incorrect
qb.where(`user.email = '${email}'`);

// ✅ Correct
qb.where("user.email = :email", { email });
```

- Select only needed fields with `.select()` when returning partial data.
- Use `.leftJoin()` + `.addSelect()` to load specific relation fields.

## Pagination

- All list endpoints use the `Pagination.of()` helper:

```ts
async findAllUsers(query: UserRequestQueryDto): Promise<PaginationResponseDto<UserResponseDto>> {
  const [entities, total] = await this.userRepository.findOfPagination(query);
  return Pagination.of(query, total, entities.map(UserMapper.toDto));
}
```

- Query DTOs extend `PaginationRequestDto`:

```ts
export class UserRequestQueryDto extends PaginationRequestDto {
  @IsOptional()
  @IsString()
  search?: string;

  @IsOptional()
  @IsEnum(UserStatus)
  status?: UserStatus;
}
```

## Migrations

```bash
# Generate migration from entity changes
npx typeorm migration:generate src/database/migrations/AddUserRole -d src/database/data-source.ts

# Run migrations
npx typeorm migration:run -d src/database/data-source.ts

# Revert last migration
npx typeorm migration:revert -d src/database/data-source.ts
```

- Always implement both `up()` and `down()`.

## Transactions

```ts
async createOrder(dto: CreateOrderDto): Promise<Order> {
  return this.dataSource.transaction(async (manager) => {
    const order = await manager.save(OrderEntity, dto);
    await manager.save(OrderItemEntity, dto.items);
    return order;
  });
}
```

## Soft Deletes

- `BaseEntity` includes `@DeleteDateColumn()` — all deletes are soft by default:

```ts
await this.userRepository.softDelete(id);
await this.userRepository.restore(id);

// Active records only (default)
const users = await this.userRepository.find();

// Include soft-deleted
const all = await this.userRepository.find({ withDeleted: true });
```

## Rules

- All entities extend `BaseEntity` — never define id/timestamp columns manually.
- Use custom repository classes — never `@InjectRepository()`.
- Use mapper classes for entity ↔ DTO conversion — never convert inline.
- Use `enumName` on all enum columns.
- Use `@Column({ name: "snake_case" })` — DB snake_case, TS camelCase.
- Use `@Unique()` at entity level for unique constraints.
- Never set `synchronize: true` in production — use migrations only.
- Always use parameterized QueryBuilder values — never interpolate.
- Paginate all list endpoints via `Pagination.of()`.
- Use transactions for multi-table writes.
