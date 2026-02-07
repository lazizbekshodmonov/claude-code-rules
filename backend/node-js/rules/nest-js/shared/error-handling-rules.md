# Error Handling Rules (NestJS)

> NestJS-specific error handling using exception filters, built-in exceptions, and the exception layer.

## Built-in HTTP Exceptions

- Use NestJS built-in exceptions for standard HTTP errors:

```ts
// ❌ Incorrect — generic Error
throw new Error("User not found");

// ✅ Correct — NestJS built-in exceptions
throw new NotFoundException(`User with id ${id} not found`);
throw new BadRequestException("Invalid email format");
throw new UnauthorizedException("Invalid credentials");
throw new ForbiddenException("Insufficient permissions");
throw new ConflictException("Email already registered");
```

| Exception | Status Code |
|-----------|-------------|
| `BadRequestException` | 400 |
| `UnauthorizedException` | 401 |
| `ForbiddenException` | 403 |
| `NotFoundException` | 404 |
| `ConflictException` | 409 |
| `UnprocessableEntityException` | 422 |
| `InternalServerErrorException` | 500 |

## Custom Exceptions

- Extend `HttpException` for domain-specific errors with custom response:

```ts
export class UserAlreadyExistsException extends ConflictException {
  constructor(email: string) {
    super({
      message: `User with email ${email} already exists`,
      code: "USER_ALREADY_EXISTS",
    });
  }
}

export class InsufficientBalanceException extends BadRequestException {
  constructor(required: number, available: number) {
    super({
      message: "Insufficient balance",
      code: "INSUFFICIENT_BALANCE",
      details: { required, available },
    });
  }
}
```

## Exception Filters

- Create a global exception filter for consistent error responses:

```ts
@Catch()
export class AllExceptionsFilter implements ExceptionFilter {
  private readonly logger = new Logger(AllExceptionsFilter.name);

  catch(exception: unknown, host: ArgumentsHost) {
    const ctx = host.switchToHttp();
    const response = ctx.getResponse<Response>();
    const request = ctx.getRequest<Request>();

    if (exception instanceof HttpException) {
      const status = exception.getStatus();
      const exceptionResponse = exception.getResponse();

      return response.status(status).json({
        statusCode: status,
        ...(typeof exceptionResponse === "object" ? exceptionResponse : { message: exceptionResponse }),
        timestamp: new Date().toISOString(),
        path: request.url,
      });
    }

    // Unexpected error
    this.logger.error("Unhandled exception", {
      error: exception,
      path: request.url,
      method: request.method,
    });

    return response.status(500).json({
      statusCode: 500,
      message: "Internal server error",
      timestamp: new Date().toISOString(),
      path: request.url,
    });
  }
}

// Register globally in main.ts
app.useGlobalFilters(new AllExceptionsFilter());
```

## Validation Errors

- Use `ValidationPipe` with class-validator for automatic DTO validation:

```ts
// main.ts
app.useGlobalPipes(
  new ValidationPipe({
    whitelist: true,           // strip unknown properties
    forbidNonWhitelisted: true, // throw on unknown properties
    transform: true,           // auto-transform types
    exceptionFactory: (errors) =>
      new UnprocessableEntityException({
        message: "Validation failed",
        code: "VALIDATION_ERROR",
        errors: errors.map((e) => ({
          field: e.property,
          messages: Object.values(e.constraints || {}),
        })),
      }),
  }),
);
```

- DTO with class-validator:

```ts
export class CreateUserDto {
  @IsEmail({}, { message: "Invalid email format" })
  email: string;

  @IsString()
  @MinLength(2, { message: "Name must be at least 2 characters" })
  @MaxLength(100)
  name: string;

  @IsEnum(UserRole)
  role: UserRole;
}
```

## Service Layer Error Pattern

- Services throw exceptions — controllers never contain error logic:

```ts
// ✅ Correct — service handles business logic and errors
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

  async create(dto: CreateUserDto): Promise<User> {
    const exists = await this.userRepository.findOneBy({ email: dto.email });
    if (exists) throw new UserAlreadyExistsException(dto.email);
    return this.userRepository.save(dto);
  }

  async update(id: string, dto: UpdateUserDto): Promise<User> {
    const user = await this.findById(id);
    Object.assign(user, dto);
    return this.userRepository.save(user);
  }
}

// ✅ Correct — controller is thin
@Controller("users")
export class UsersController {
  constructor(private readonly usersService: UsersService) {}

  @Get(":id")
  async findById(@Param("id", ParseUUIDPipe) id: string) {
    return this.usersService.findById(id);
  }

  @Post()
  async create(@Body() dto: CreateUserDto) {
    return this.usersService.create(dto);
  }
}
```

## TypeORM Error Handling

- Catch TypeORM-specific errors and convert to HTTP exceptions:

```ts
async create(dto: CreateUserDto): Promise<User> {
  try {
    return await this.userRepository.save(dto);
  } catch (error) {
    if (error.code === "23505") {
      // PostgreSQL unique violation
      throw new ConflictException("User with this email already exists");
    }
    throw error;
  }
}
```

## Pipes for Parameter Validation

- Use built-in pipes for parameter parsing and validation:

```ts
@Get(":id")
async findById(@Param("id", ParseUUIDPipe) id: string) { ... }

@Get()
async findAll(
  @Query("page", new DefaultValuePipe(1), ParseIntPipe) page: number,
  @Query("limit", new DefaultValuePipe(20), ParseIntPipe) limit: number,
) { ... }
```

## Rules

- Use NestJS built-in exceptions — never throw raw `Error` or generic `HttpException`.
- Create custom exceptions for domain-specific error cases.
- Register a global `ExceptionFilter` for consistent error response format.
- Use `ValidationPipe` globally — never validate DTOs manually in controllers.
- Services throw exceptions — controllers delegate to services.
- Log unexpected errors in exception filters with full context.
- Never expose stack traces or internal details in responses.
- Test both happy and error paths for every endpoint.
