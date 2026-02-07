# Testing Rules (NestJS)

> NestJS-specific testing rules using `@nestjs/testing`, Jest, and supertest.

## Test Module Setup

- Use `Test.createTestingModule` to create isolated test modules:

```ts
describe("UsersService", () => {
  let service: UsersService;
  let repository: Repository<User>;

  beforeEach(async () => {
    const module = await Test.createTestingModule({
      providers: [
        UsersService,
        {
          provide: getRepositoryToken(User),
          useValue: {
            findOneBy: jest.fn(),
            find: jest.fn(),
            save: jest.fn(),
            softDelete: jest.fn(),
            createQueryBuilder: jest.fn(),
          },
        },
      ],
    }).compile();

    service = module.get(UsersService);
    repository = module.get(getRepositoryToken(User));
  });
});
```

## Unit Testing Services

- Mock repository methods — test business logic in isolation:

```ts
describe("findById", () => {
  it("should return user when found", async () => {
    const mockUser = { id: "uuid-1", name: "John", email: "john@test.com" };
    jest.spyOn(repository, "findOneBy").mockResolvedValue(mockUser as User);

    const result = await service.findById("uuid-1");

    expect(result).toEqual(mockUser);
    expect(repository.findOneBy).toHaveBeenCalledWith({ id: "uuid-1" });
  });

  it("should throw NotFoundException when user not found", async () => {
    jest.spyOn(repository, "findOneBy").mockResolvedValue(null);

    await expect(service.findById("uuid-1")).rejects.toThrow(NotFoundException);
  });
});

describe("create", () => {
  it("should create and return a new user", async () => {
    const dto: CreateUserDto = { name: "John", email: "john@test.com" };
    const savedUser = { id: "uuid-1", ...dto };
    jest.spyOn(repository, "findOneBy").mockResolvedValue(null);
    jest.spyOn(repository, "save").mockResolvedValue(savedUser as User);

    const result = await service.create(dto);

    expect(result).toEqual(savedUser);
    expect(repository.save).toHaveBeenCalledWith(dto);
  });

  it("should throw ConflictException when email exists", async () => {
    const dto: CreateUserDto = { name: "John", email: "john@test.com" };
    jest.spyOn(repository, "findOneBy").mockResolvedValue({ id: "existing" } as User);

    await expect(service.create(dto)).rejects.toThrow(ConflictException);
  });
});
```

## Unit Testing Controllers

```ts
describe("UsersController", () => {
  let controller: UsersController;
  let service: UsersService;

  beforeEach(async () => {
    const module = await Test.createTestingModule({
      controllers: [UsersController],
      providers: [
        {
          provide: UsersService,
          useValue: {
            findById: jest.fn(),
            create: jest.fn(),
            update: jest.fn(),
            delete: jest.fn(),
          },
        },
      ],
    }).compile();

    controller = module.get(UsersController);
    service = module.get(UsersService);
  });

  it("should return user for valid ID", async () => {
    const mockUser = { id: "uuid-1", name: "John" };
    jest.spyOn(service, "findById").mockResolvedValue(mockUser as User);

    const result = await controller.findById("uuid-1");

    expect(result).toEqual(mockUser);
  });
});
```

## E2E Testing

- Use `@nestjs/testing` with supertest for full request/response cycle:

```ts
describe("UsersController (e2e)", () => {
  let app: INestApplication;
  let dataSource: DataSource;

  beforeAll(async () => {
    const module = await Test.createTestingModule({
      imports: [AppModule],
    }).compile();

    app = module.createNestApplication();
    app.useGlobalPipes(new ValidationPipe({ whitelist: true }));
    await app.init();

    dataSource = module.get(DataSource);
  });

  afterAll(async () => {
    await app.close();
  });

  beforeEach(async () => {
    await dataSource.query("TRUNCATE TABLE users CASCADE");
  });

  describe("GET /users/:id", () => {
    it("should return 200 and user data", async () => {
      const user = await createTestUser(dataSource);
      const token = generateTestToken(user);

      const res = await request(app.getHttpServer())
        .get(`/users/${user.id}`)
        .set("Authorization", `Bearer ${token}`)
        .expect(200);

      expect(res.body).toMatchObject({
        id: user.id,
        name: user.name,
      });
    });

    it("should return 404 for non-existent user", async () => {
      const token = generateTestToken({ id: "uuid-1", role: "admin" });

      await request(app.getHttpServer())
        .get("/users/non-existent-uuid")
        .set("Authorization", `Bearer ${token}`)
        .expect(404);
    });

    it("should return 401 without auth token", async () => {
      await request(app.getHttpServer())
        .get("/users/uuid-1")
        .expect(401);
    });
  });

  describe("POST /users", () => {
    it("should return 201 for valid input", async () => {
      const token = generateTestToken({ id: "uuid-1", role: "admin" });

      const res = await request(app.getHttpServer())
        .post("/users")
        .set("Authorization", `Bearer ${token}`)
        .send({ name: "Jane", email: "jane@test.com" })
        .expect(201);

      expect(res.body.email).toBe("jane@test.com");
    });

    it("should return 422 for invalid email", async () => {
      const token = generateTestToken({ id: "uuid-1", role: "admin" });

      await request(app.getHttpServer())
        .post("/users")
        .set("Authorization", `Bearer ${token}`)
        .send({ name: "Jane", email: "not-an-email" })
        .expect(422);
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

  it("should deny access when user lacks required role", () => {
    jest.spyOn(reflector, "getAllAndOverride").mockReturnValue(["admin"]);
    const context = createMockExecutionContext({ user: { role: "user" } });

    expect(guard.canActivate(context)).toBe(false);
  });
});
```

## Test Database

- Use a separate test database — configure in `.env.test`:

```
DATABASE_URL="postgresql://user:pass@localhost:5432/myapp_test"
```

- Use test helpers for creating/cleaning data:

```ts
export async function createTestUser(dataSource: DataSource, overrides = {}): Promise<User> {
  const repo = dataSource.getRepository(User);
  return repo.save({
    name: "Test User",
    email: `test-${Date.now()}@test.com`,
    ...overrides,
  });
}
```

## Coverage

- Minimum: **80%** per file. Critical logic (auth, payments): **90%+**.

## Rules

- Use `Test.createTestingModule` — never instantiate services directly.
- Mock repositories with `getRepositoryToken()` — never use real DB in unit tests.
- E2E tests use a real test database — reset between test suites.
- Test guards, pipes, and interceptors separately.
- Test both success and error paths for every service method.
- Each test must be independent — no shared mutable state.
- Keep tests fast — mock everything except what you're testing.
