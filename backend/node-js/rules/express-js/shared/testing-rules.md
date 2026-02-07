# Testing Rules (Express)

> Express-specific testing rules using Jest, supertest, and manual test setup.

## Test Setup

- Create a test app factory that initializes Express without starting the server:

```ts
// src/test/setup.ts
import { dataSource } from "../database/data-source";

export async function createTestApp() {
  if (!dataSource.isInitialized) {
    await dataSource.initialize();
  }
  // Import app after DB is ready
  const { app } = await import("../app");
  return app;
}

export async function cleanDatabase() {
  await dataSource.query("TRUNCATE TABLE users, orders CASCADE");
}

export async function closeDatabase() {
  if (dataSource.isInitialized) {
    await dataSource.destroy();
  }
}
```

## Unit Testing Services

- Mock repository methods — test business logic in isolation:

```ts
import { userService } from "../services/user.service";

// Mock the repository
jest.mock("../database/data-source", () => ({
  dataSource: {
    getRepository: jest.fn().mockReturnValue({
      findOneBy: jest.fn(),
      find: jest.fn(),
      save: jest.fn(),
      softDelete: jest.fn(),
    }),
  },
}));

describe("userService", () => {
  const mockRepo = dataSource.getRepository(User) as jest.Mocked<Repository<User>>;

  beforeEach(() => jest.clearAllMocks());

  describe("findById", () => {
    it("should return user when found", async () => {
      const mockUser = { id: "uuid-1", name: "John" };
      mockRepo.findOneBy.mockResolvedValue(mockUser as User);

      const result = await userService.findById("uuid-1");

      expect(result).toEqual(mockUser);
      expect(mockRepo.findOneBy).toHaveBeenCalledWith({ id: "uuid-1" });
    });

    it("should throw NotFoundError when user does not exist", async () => {
      mockRepo.findOneBy.mockResolvedValue(null);

      await expect(userService.findById("uuid-1")).rejects.toThrow(NotFoundError);
    });
  });

  describe("create", () => {
    it("should create and return user", async () => {
      const dto = { name: "John", email: "john@test.com" };
      const saved = { id: "uuid-1", ...dto };
      mockRepo.findOneBy.mockResolvedValue(null);
      mockRepo.save.mockResolvedValue(saved as User);

      const result = await userService.create(dto);

      expect(result).toEqual(saved);
    });

    it("should throw ConflictError when email exists", async () => {
      mockRepo.findOneBy.mockResolvedValue({ id: "existing" } as User);

      await expect(userService.create({ name: "John", email: "john@test.com" }))
        .rejects.toThrow(ConflictError);
    });
  });
});
```

## E2E / Integration Testing with Supertest

```ts
import request from "supertest";

describe("Users API", () => {
  let app: Express;

  beforeAll(async () => {
    app = await createTestApp();
  });

  afterAll(async () => {
    await closeDatabase();
  });

  beforeEach(async () => {
    await cleanDatabase();
  });

  describe("GET /api/users/:id", () => {
    it("should return 200 and user data", async () => {
      const user = await createTestUser();
      const token = generateTestToken(user);

      const res = await request(app)
        .get(`/api/users/${user.id}`)
        .set("Authorization", `Bearer ${token}`)
        .expect(200);

      expect(res.body.data).toMatchObject({
        id: user.id,
        name: user.name,
      });
    });

    it("should return 404 for non-existent user", async () => {
      const token = generateTestToken({ id: "uuid-1", role: "admin" });

      await request(app)
        .get("/api/users/non-existent-id")
        .set("Authorization", `Bearer ${token}`)
        .expect(404);
    });

    it("should return 401 without auth token", async () => {
      await request(app)
        .get("/api/users/uuid-1")
        .expect(401);
    });
  });

  describe("POST /api/users", () => {
    it("should return 201 for valid input", async () => {
      const token = generateTestToken({ id: "uuid-1", role: "admin" });

      const res = await request(app)
        .post("/api/users")
        .set("Authorization", `Bearer ${token}`)
        .send({ name: "Jane", email: "jane@test.com", password: "Test1234" })
        .expect(201);

      expect(res.body.data.email).toBe("jane@test.com");
    });

    it("should return 422 for invalid email", async () => {
      const token = generateTestToken({ id: "uuid-1", role: "admin" });

      await request(app)
        .post("/api/users")
        .set("Authorization", `Bearer ${token}`)
        .send({ name: "Jane", email: "not-an-email", password: "Test1234" })
        .expect(422);
    });
  });
});
```

## Testing Middleware

```ts
describe("authenticate middleware", () => {
  it("should set req.user for valid token", async () => {
    const token = jwt.sign({ sub: "uuid-1", role: "user" }, process.env.JWT_SECRET!);

    const res = await request(app)
      .get("/api/profile")
      .set("Authorization", `Bearer ${token}`)
      .expect(200);

    expect(res.body.data.id).toBe("uuid-1");
  });

  it("should return 401 for invalid token", async () => {
    await request(app)
      .get("/api/profile")
      .set("Authorization", "Bearer invalid-token")
      .expect(401);
  });

  it("should return 401 without token", async () => {
    await request(app)
      .get("/api/profile")
      .expect(401);
  });
});

describe("authorize middleware", () => {
  it("should return 403 when user lacks required role", async () => {
    const token = generateTestToken({ id: "uuid-1", role: "user" });

    await request(app)
      .delete("/api/users/uuid-2")
      .set("Authorization", `Bearer ${token}`)
      .expect(403);
  });
});
```

## Test Helpers

```ts
// src/test/helpers.ts
export async function createTestUser(overrides = {}): Promise<User> {
  const repo = dataSource.getRepository(User);
  return repo.save({
    name: "Test User",
    email: `test-${Date.now()}@test.com`,
    password: await bcrypt.hash("Test1234", 4), // low cost for speed
    ...overrides,
  });
}

export function generateTestToken(user: Partial<User>): string {
  return jwt.sign(
    { sub: user.id, email: user.email, role: user.role || "user" },
    process.env.JWT_SECRET!,
    { expiresIn: "1h" },
  );
}
```

## Test Database

```
# .env.test
DATABASE_URL="postgresql://user:pass@localhost:5432/myapp_test"
JWT_SECRET="test-secret-key-for-testing-only"
```

## Coverage

- Minimum: **80%** per file. Critical logic (auth, payments): **90%+**.

## Rules

- Use supertest for E2E tests — never start a real HTTP server in tests.
- Mock repositories in unit tests — never use real DB.
- E2E tests use a real test database — reset between test suites.
- Test auth middleware (valid token, invalid token, no token, wrong role).
- Test both happy and error paths.
- Each test must be independent — no shared mutable state.
- Use test helpers for creating users, generating tokens.
