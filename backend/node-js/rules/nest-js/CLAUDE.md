# Code Writing Rules (NestJS + TypeScript)

> Production-ready NestJS rules based on real project patterns.
>
> Want to contribute? See the [Contributing guide](../../../../README.md#contributing).

## Shared Rules

Include the following shared rule files in your project's `CLAUDE.md`:

### Universal (All Stacks)

- [Agent Rules](../../../../shared/agent-rules.md)
- [Git Rules](../../../../shared/git-rules.md)
- [Planning Rules](../../../../shared/planning-rules.md)

### Node.js Universal

- [API Design Rules](../../shared/api-design-rules.md)
- [Naming Rules](../../shared/naming-rules.md)
- [Code Review Rules](../../shared/code-review-rules.md)
- [Documentation Rules](../../shared/documentation-rules.md)
- [Logging Rules](../../shared/logging-rules.md)
- [Configuration Rules](../../shared/configuration-rules.md)

### NestJS-Specific

- [Database Rules](./shared/database-rules.md)
- [Error Handling Rules](./shared/error-handling-rules.md)
- [Security Rules](./shared/security-rules.md)
- [Testing Rules](./shared/testing-rules.md)
- [Performance Rules](./shared/performance-rules.md)
- [Caching Rules](./shared/caching-rules.md)

---

## Project Structure

### Module Structure

Each module follows a standard folder layout:

```
modules/<module-name>/
├── controllers/          # Route handlers (thin — delegates to services)
│   ├── <name>.controller.ts
│   └── <name>-admin.controller.ts
├── services/             # Business logic
│   └── <name>.service.ts
├── repositories/         # Custom TypeORM repositories
│   └── <name>.repository.ts
├── entities/             # TypeORM entity definitions
│   └── <name>.entity.ts
├── mappers/              # Entity ↔ DTO static converters
│   └── <name>.mapper.ts
├── dto/                  # Request/response DTOs + index.ts barrel
│   ├── index.ts
│   ├── <name>-create.dto.ts
│   ├── <name>-update.dto.ts
│   ├── <name>-response.dto.ts
│   └── <name>-request-query.dto.ts
├── enums/                # Module-specific enums (including error enums)
│   ├── <name>-error.enum.ts
│   └── <name>-role.enum.ts
├── swagger/              # Swagger decorator factories
│   └── <name>.swagger.ts
└── <module-name>.module.ts
```

### Common / Shared Directory

```
common/
├── dto/                  # ErrorResponseDto, shared DTOs
├── entities/             # BaseEntity (abstract)
├── enums/                # GlobalError, UserStatus, etc.
├── exceptions/           # AppException
├── filters/              # HttpExceptionFilter
├── localization/         # error-messages.ts, type.ts
├── modules/              # RedisModule, etc.
├── pagination/           # Pagination helper, DTOs, swagger decorator
├── services/             # CookieService, etc.
└── utils/                # Utility functions
```

---

## Core Patterns

### BaseEntity

All entities extend `BaseEntity` — never define `id`, `createdAt`, `updatedAt`, `deletedAt` manually:

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

// Usage
@Entity("users")
@Unique(["email"])
export class UserEntity extends BaseEntity {
  @Column({ name: "name", type: "varchar", length: 150 })
  name: string;
  // ...
}
```

### AppException + Error Enums + i18n

Never use NestJS built-in exceptions (`NotFoundException`, `BadRequestException`, etc.) directly. Use `AppException` with module-specific error enums:

```ts
// common/exceptions/app-exception.ts
export class AppException extends HttpException {
  constructor(
    public readonly code: AppExceptionCode,
    status: HttpStatus = HttpStatus.BAD_REQUEST,
  ) {
    super({ code }, status);
  }
}

// Module error enum — prefix with module name
export enum UserError {
  ALREADY_EXISTS = "USER_ALREADY_EXISTS",
  NOT_FOUND = "USER_NOT_FOUND",
  INACTIVE = "USER_INACTIVE",
  SELF_UPDATE_FORBIDDEN = "USER_SELF_UPDATE_FORBIDDEN",
  SELF_DELETE_FORBIDDEN = "USER_SELF_DELETE_FORBIDDEN",
}

// Usage in services
throw new AppException(UserError.NOT_FOUND, HttpStatus.NOT_FOUND);
throw new AppException(AuthError.INVALID_CREDENTIALS, HttpStatus.UNAUTHORIZED);
throw new AppException(AuthError.PERMISSION_DENIED, HttpStatus.FORBIDDEN);
```

Each error code has i18n translations (uz, ru, en, cyr) in `common/localization/error-messages.ts`:

```ts
export const ERROR_MESSAGES: Record<AppExceptionCode, LocalizedString> = {
  USER_NOT_FOUND: {
    uz: "Foydalanuvchi topilmadi yoki mavjud emas.",
    ru: "Пользователь не найден или не существует.",
    en: "User not found or does not exist.",
    cyr: "Фойдаланувчи топилмади ёки мавжуд эмас.",
  },
  // ...
};
```

The `AppExceptionCode` type is a union of all module error enums:

```ts
export type AppExceptionCode = GlobalError | AuthError | UserError | CompanyError | ...;
```

### Custom Repository Pattern

Never use `@InjectRepository()`. Create custom repository classes that extend `Repository<Entity>`:

```ts
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

    if (search) {
      qb.andWhere("(user.name ILIKE :search OR user.email ILIKE :search)", {
        search: `%${search.trim()}%`,
      });
    }

    return qb.getManyAndCount();
  }
}
```

Register repositories as providers (not via `TypeOrmModule.forFeature`):

```ts
@Module({
  providers: [UserService, UserRepository],
  controllers: [UserController],
  exports: [UserService, UserRepository],
})
export class UserModule {}
```

### Mapper Pattern

Use static mapper classes for Entity ↔ DTO conversion. Never convert in controllers or services directly:

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

// Usage in services
const entity = UserMapper.toCreateEntity(dto, UserRole.USER);
const saved = await this.userRepository.save(entity);
return UserMapper.toDto(saved);
```

### Pagination

Use the `Pagination.of()` helper with standardized DTOs:

```ts
// Service
async findAllUsers(query: UserRequestQueryDto): Promise<PaginationResponseDto<UserResponseDto>> {
  const [entities, total] = await this.userRepository.findOfPagination(query);
  return Pagination.of(query, total, entities.map(UserMapper.toDto));
}

// Controller
@Get()
@Roles(UserRole.ADMIN)
findAll(@Query() query: UserRequestQueryDto): Promise<PaginationResponseDto<UserResponseDto>> {
  return this.userService.findAllUsers(query);
}
```

Query DTOs extend `PaginationRequestDto`:

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

### Swagger Documentation

Place swagger decorators in separate `swagger/` folders per module. Create decorator factory functions:

```ts
// modules/user/swagger/user.swagger.ts
export function UserFindAllSwaggerDoc() {
  return applyDecorators(
    ApiOperation({
      summary: "Get list of users",
      description: "Returns a paginated list of users.",
    }),
    ApiPaginatedResponse(UserResponseDto, "Paginated list of users"),
  );
}

// Usage in controller
@Get()
@Roles(UserRole.ADMIN)
@UserFindAllSwaggerDoc()
findAll(@Query() query: UserRequestQueryDto) { ... }
```

### Guards & Decorators

Use `JwtWebGuard` + `RolesGuard` at controller level. Use custom parameter decorators:

```ts
@ApiTags("Users")
@ApiBearerAuth()
@UseGuards(JwtWebGuard, RolesGuard)
@Controller("user")
export class UserController {
  @Get("profile")
  @HttpCode(HttpStatus.OK)
  profile(@AuthenticatedUser() jwtPayload: JwtPayload) {
    return this.userService.getProfile(jwtPayload);
  }

  @Post()
  @HttpCode(HttpStatus.CREATED)
  @Roles(UserRole.ADMIN)
  create(@Body() dto: UserCreateDto) {
    return this.userService.createUser(dto);
  }

  @Get(":id")
  @Roles(UserRole.ADMIN)
  findOne(@Param("id", ParseIntPipe) id: number) {
    return this.userService.findOne(id);
  }
}
```

### JWT Cookie-Based Auth

Tokens are stored in httpOnly cookies, not Bearer headers. Use `CookieService` for setting/clearing:

```ts
// Login — set cookies
const tokens = await this.authService.login(dto, companyKey);
this.cookieService.accessTokenSetCookie(response, tokens.accessToken, tokens.accessExpiresAt);
this.cookieService.refreshTokenSetCookie(response, tokens.refreshToken, tokens.refreshExpiresAt);

// JwtWebGuard — extract from cookies
const accessToken = request.cookies?.["access_token"] as string | undefined;
if (!accessToken) {
  throw new AppException(AuthError.ACCESS_TOKEN_INVALID_OR_EXPIRED, HttpStatus.UNAUTHORIZED);
}
const payload = await this.jwtService.verifyAccessToken(accessToken);
request.user = payload;
```

### YAML Configuration

Use `application.yml` with `${ENV_VAR:default}` syntax. Access via `ConfigService` with dot notation:

```yaml
# config/yml/application.yml
server:
  port: ${PORT:8080}
  environment: ${NODE_ENV:development}

database:
  postgres:
    host: ${POSTGRES_HOST:localhost}
    port: ${POSTGRES_PORT:5432}

security:
  jwt:
    jwt_secret: ${JWT_SECRET:default_jwt_secret}
    access_expires_in: 30m
    refresh_expires_in: 1d
```

```ts
// Access config values
const port = this.configService.get<number>("server.port", 3000);
const jwtSecret = this.configService.get<string>("security.jwt.jwt_secret");
const redisHost = this.configService.get<string>("database.redis.host");
```

### Redis via Custom RedisService

Use the global `RedisModule` with `RedisService` — not `@nestjs/cache-manager`:

```ts
constructor(private readonly redisService: RedisService) {}

// Store data with TTL
await this.redisService.set("otp:user@email.com", otpCode, 300);

// Retrieve data
const otp = await this.redisService.get<string>("otp:user@email.com");

// Delete
await this.redisService.del("otp:user@email.com");

// Increment (rate limiting)
await this.redisService.incr("rate:login:user@email.com");
await this.redisService.expire("rate:login:user@email.com", 3600);
```

### Multi-Tenancy

Company-scoped access via `X-Company-Key` header. Use `CompanyKeyGuard` + `@CompanyId()` decorator:

```ts
@UseGuards(JwtWebGuard, CompanyKeyGuard, RolesGuard)
@Controller("employee")
export class EmployeeController {
  @Get()
  @Roles(UserRole.ADMIN, EmployeeRole.HR)
  findAll(@CompanyId() companyId: number, @Query() query: EmployeeQueryDto) {
    return this.employeeService.findAll(companyId, query);
  }
}
```

---

## Controller Rules

- Controllers are **thin** — only handle HTTP concerns (status codes, params, decorators).
- Always set `@HttpCode()` explicitly on every route.
- Use `ParseIntPipe` / `ParseUUIDPipe` for path params.
- Apply `@UseGuards(JwtWebGuard, RolesGuard)` at controller level.
- Use `@Roles()` on individual routes that need authorization.
- Use `@AuthenticatedUser()` to get JWT payload — never access `req.user` directly.
- Apply swagger decorator from `swagger/` folder on each route.
- Use `@Res({ passthrough: true })` only when setting cookies.

## Service Rules

- All business logic lives in services.
- Services throw `AppException` — never NestJS built-in exceptions.
- Use mappers for entity ↔ DTO conversion.
- Use `Pagination.of()` for paginated responses.
- Inject custom repositories (not `@InjectRepository`).

## DTO Rules

- Use `class-validator` + `class-transformer` for validation.
- Response DTOs have `@ApiProperty()` decorators for Swagger.
- Create query DTOs extending `PaginationRequestDto` for list endpoints.
- Export all DTOs from `dto/index.ts` barrel file.
- DTO file naming: `<name>-create.dto.ts`, `<name>-response.dto.ts`, etc.

## Entity Rules

- All entities extend `BaseEntity`.
- Use `@Column({ name: "snake_case" })` for column mapping.
- Use TypeORM enums with `enumName` to prevent migration conflicts.
- Use `@Unique()` at entity level for unique constraints.
- Column naming: DB uses `snake_case`, TypeScript uses `camelCase`.

## Error Handling Rules

- Define module-specific error enums with `MODULE_` prefix.
- Register all error codes in `ERROR_MESSAGES` with 4-language translations.
- Union all error enums into `AppExceptionCode` type.
- Always pass `HttpStatus` as second argument to `AppException`.

## File Naming

| Item | Pattern | Example |
|------|---------|---------|
| Entity | `<name>.entity.ts` | `user.entity.ts` |
| Repository | `<name>.repository.ts` | `user.repository.ts` |
| Service | `<name>.service.ts` | `user.service.ts` |
| Controller | `<name>.controller.ts` | `user.controller.ts` |
| Mapper | `<name>.mapper.ts` | `user.mapper.ts` |
| DTO | `<name>-<action>.dto.ts` | `user-create.dto.ts` |
| Error enum | `<name>-error.enum.ts` | `user-error.enum.ts` |
| Swagger | `<name>.swagger.ts` | `user.swagger.ts` |
| Module | `<name>.module.ts` | `user.module.ts` |
| Guard | `<name>.guard.ts` | `jwt-web.guard.ts` |
| Decorator | `<name>.decorator.ts` | `authenticated-user.decorator.ts` |
