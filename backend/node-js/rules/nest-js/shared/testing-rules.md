# Testing Rules (NestJS)

> NestJS-specific testing rules using `@nestjs/testing`, Jest, and supertest — aligned with custom repository, mapper, and AppException patterns.

## Test Module Setup

- Use `Test.createTestingModule` to create isolated test modules:

```ts
describe("UserService", () => {
  let service: UserService;
  let repository: UserRepository;

  beforeEach(async () => {
    const module = await Test.createTestingModule({
      providers: [
        UserService,
        {
          provide: UserRepository,
          useValue: {
            findOneBy: jest.fn(),
            findOne: jest.fn(),
            find: jest.fn(),
            save: jest.fn(),
            softDelete: jest.fn(),
            findByUsername: jest.fn(),
            findOfPagination: jest.fn(),
          },
        },
      ],
    }).compile();

    service = module.get(UserService);
    repository = module.get(UserRepository);
  });
});
```

- Mock the **custom repository class** directly (not via `getRepositoryToken()` since we use custom repositories).

## Unit Testing Services

- Mock repository methods — test business logic and AppException throwing:

```ts
describe("findOne", () => {
  it("should return user DTO when found", async () => {
    const mockEntity = { id: 1, name: "John", email: "john@test.com", role: UserRole.USER, status: UserStatus.ACTIVE };
    jest.spyOn(repository, "findOneBy").mockResolvedValue(mockEntity as UserEntity);

    const result = await service.findOne(1);

    expect(result).toEqual(expect.objectContaining({
      id: 1,
      name: "John",
      email: "john@test.com",
    }));
    expect(repository.findOneBy).toHaveBeenCalledWith({ id: 1 });
  });

  it("should throw AppException with USER_NOT_FOUND when user does not exist", async () => {
    jest.spyOn(repository, "findOneBy").mockResolvedValue(null);

    try {
      await service.findOne(1);
      fail("Expected AppException");
    } catch (error) {
      expect(error).toBeInstanceOf(AppException);
      expect(error.code).toBe(UserError.NOT_FOUND);
      expect(error.getStatus()).toBe(HttpStatus.NOT_FOUND);
    }
  });
});

describe("createUser", () => {
  it("should create and return user DTO", async () => {
    const dto: UserCreateDto = { name: "John", email: "john@test.com", password: "Test1234" };
    const savedEntity = { id: 1, name: "John", email: "john@test.com", role: UserRole.USER, status: UserStatus.ACTIVE };
    jest.spyOn(repository, "findByUsername").mockResolvedValue(null);
    jest.spyOn(repository, "save").mockResolvedValue(savedEntity as UserEntity);

    const result = await service.createUser(dto);

    expect(result.email).toBe("john@test.com");
  });

  it("should throw AppException with USER_ALREADY_EXISTS when email exists", async () => {
    jest.spyOn(repository, "findByUsername").mockResolvedValue({ id: 1 } as UserEntity);

    try {
      await service.createUser({ name: "John", email: "john@test.com", password: "Test1234" });
      fail("Expected AppException");
    } catch (error) {
      expect(error).toBeInstanceOf(AppException);
      expect(error.code).toBe(UserError.ALREADY_EXISTS);
    }
  });
});
```

## Testing AppException

- Always verify both `code` and `status`:

```ts
try {
  await service.someMethod();
  fail("Expected AppException");
} catch (error) {
  expect(error).toBeInstanceOf(AppException);
  expect(error.code).toBe(ModuleError.ERROR_CODE);
  expect(error.getStatus()).toBe(HttpStatus.EXPECTED_STATUS);
}
```

## Unit Testing Controllers

```ts
describe("UserController", () => {
  let controller: UserController;
  let service: UserService;

  beforeEach(async () => {
    const module = await Test.createTestingModule({
      controllers: [UserController],
      providers: [
        {
          provide: UserService,
          useValue: {
            findOne: jest.fn(),
            findAllUsers: jest.fn(),
            createUser: jest.fn(),
            updateUser: jest.fn(),
            deleteUser: jest.fn(),
            getProfile: jest.fn(),
          },
        },
      ],
    }).compile();

    controller = module.get(UserController);
    service = module.get(UserService);
  });

  it("should return user for valid ID", async () => {
    const mockDto = { id: 1, name: "John", email: "john@test.com" };
    jest.spyOn(service, "findOne").mockResolvedValue(mockDto as UserResponseDto);

    const result = await controller.findOne(1);

    expect(result).toEqual(mockDto);
  });
});
```

## E2E Testing

```ts
describe("UserController (e2e)", () => {
  let app: INestApplication;
  let dataSource: DataSource;

  beforeAll(async () => {
    const module = await Test.createTestingModule({
      imports: [AppModule],
    }).compile();

    app = module.createNestApplication();
    app.useGlobalPipes(new ValidationPipe({ whitelist: true, transform: true }));
    app.useGlobalFilters(new HttpExceptionFilter());
    await app.init();

    dataSource = module.get(DataSource);
  });

  afterAll(async () => {
    await app.close();
  });

  beforeEach(async () => {
    await dataSource.query("TRUNCATE TABLE users CASCADE");
  });

  describe("GET /user/:id", () => {
    it("should return 200 and user data", async () => {
      const user = await createTestUser(dataSource);
      const cookie = await loginAndGetCookies(app, user);

      const res = await request(app.getHttpServer())
        .get(`/user/${user.id}`)
        .set("Cookie", cookie)
        .set("X-Company-Key", testCompanyKey)
        .expect(200);

      expect(res.body).toMatchObject({ id: user.id, name: user.name });
    });

    it("should return 404 with USER_NOT_FOUND code", async () => {
      const cookie = await loginAsAdmin(app);

      const res = await request(app.getHttpServer())
        .get("/user/99999")
        .set("Cookie", cookie)
        .expect(404);

      expect(res.body.code).toBe("USER_NOT_FOUND");
      expect(res.body.message).toBeDefined();
      expect(res.body.locale).toBeDefined();
    });

    it("should return 401 without auth cookie", async () => {
      await request(app.getHttpServer())
        .get("/user/1")
        .expect(401);
    });
  });

  describe("POST /user", () => {
    it("should return 201 for valid input", async () => {
      const cookie = await loginAsAdmin(app);

      const res = await request(app.getHttpServer())
        .post("/user")
        .set("Cookie", cookie)
        .send({ name: "Jane", email: "jane@test.com", password: "Test1234!" })
        .expect(201);

      expect(res.body.email).toBe("jane@test.com");
    });

    it("should return 400 with VALIDATION_FAILED for invalid email", async () => {
      const cookie = await loginAsAdmin(app);

      const res = await request(app.getHttpServer())
        .post("/user")
        .set("Cookie", cookie)
        .send({ name: "Jane", email: "not-an-email", password: "Test1234!" })
        .expect(400);

      expect(res.body.code).toBe("VALIDATION_FAILED");
      expect(res.body.details).toBeDefined();
    });
  });
});
```

## Testing Guards

```ts
describe("RolesGuard", () => {
  let guard: RolesGuard;
  let reflector: Reflector;

  beforeEach(async () => {
    const module = await Test.createTestingModule({
      providers: [RolesGuard, Reflector],
    }).compile();

    guard = module.get(RolesGuard);
    reflector = module.get(Reflector);
  });

  it("should allow access when no roles are required", () => {
    jest.spyOn(reflector, "getAllAndOverride").mockReturnValue(undefined);
    const context = createMockExecutionContext({ user: { role: "user" } });

    expect(guard.canActivate(context)).toBe(true);
  });

  it("should throw AppException when user lacks required role", () => {
    jest.spyOn(reflector, "getAllAndOverride").mockReturnValue([UserRole.ADMIN]);
    const context = createMockExecutionContext({ user: { role: UserRole.USER } });

    expect(() => guard.canActivate(context)).toThrow(AppException);
  });
});
```

## Test Helpers

```ts
// test/helpers.ts
export async function createTestUser(dataSource: DataSource, overrides = {}): Promise<UserEntity> {
  const repo = dataSource.getRepository(UserEntity);
  return repo.save({
    name: "Test User",
    email: `test-${Date.now()}@test.com`,
    passwordHash: await bcrypt.hash("Test1234!", 4), // low cost for speed
    role: UserRole.USER,
    status: UserStatus.ACTIVE,
    ...overrides,
  });
}

export async function loginAndGetCookies(app: INestApplication, user: UserEntity): Promise<string[]> {
  const res = await request(app.getHttpServer())
    .post("/auth/login-web")
    .send({ email: user.email, password: "Test1234!" });
  return res.headers["set-cookie"];
}

export function createMockExecutionContext(data: { user?: Partial<JwtPayload> }): ExecutionContext {
  return {
    switchToHttp: () => ({
      getRequest: () => ({ user: data.user }),
    }),
    getHandler: () => jest.fn(),
    getClass: () => jest.fn(),
  } as unknown as ExecutionContext;
}
```

## Test Database

```yaml
# config/yml/application.test.yml
database:
  postgres:
    host: localhost
    port: 5432
    username: test_user
    password: test_pass
    name: myapp_test
```

## Coverage

- Minimum: **80%** per file. Critical logic (auth, payments): **90%+**.

## Rules

- Use `Test.createTestingModule` — never instantiate services directly.
- Mock custom repository classes directly (not `getRepositoryToken()`).
- Verify `AppException` with both `code` and `status` — not NestJS built-in exceptions.
- E2E tests use cookies for auth — not Bearer tokens.
- E2E tests verify error response shape (`code`, `message`, `locale`, `details`).
- Test guards, decorators, and mappers separately.
- Test both success and error paths for every service method.
- Each test must be independent — no shared mutable state.
- Use test helpers for creating users, logging in, mock contexts.
- Keep tests fast — mock everything except what you're testing.
