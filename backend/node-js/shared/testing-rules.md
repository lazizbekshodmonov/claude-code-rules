# Testing Rules

> Backend testing rules for Claude Code. Apply to all Node.js backend projects.

## Mandatory Testing

- **Every** new or modified service, controller, and utility must have accompanying tests.
- All tests **must pass** before a task can be marked as "Completed".
- If a test fails, fix the code or the test — never skip or delete a failing test.

## Test Types

### Unit Tests

Must cover:
- Service methods — business logic, validation, edge cases
- Utility functions — input/output correctness, error handling
- Guards and interceptors — access control logic
- DTOs and validators — schema validation rules

### Integration Tests

Must cover:
- API endpoints — full request/response cycle with database
- Authentication/authorization flows
- Database operations — CRUD with real or test database
- Third-party service interactions (mocked)

### E2E Tests

Must cover:
- Critical user flows (register, login, main features)
- Multi-step workflows (create order -> pay -> confirm)
- Error scenarios (invalid input, unauthorized access)

## Test File Location

Place test files next to source files or in a dedicated `__tests__` directory:

```
src/
├── users/
│   ├── users.service.ts
│   ├── users.controller.ts
│   ├── __tests__/
│   │   ├── users.service.spec.ts
│   │   ├── users.controller.spec.ts
│   │   └── users.e2e-spec.ts
│   └── index.ts
```

## Naming Conventions

- Unit test files: `{name}.spec.ts`
- Integration test files: `{name}.integration-spec.ts`
- E2E test files: `{name}.e2e-spec.ts`
- Test descriptions: clear, human-readable sentences

```ts
// ❌ Incorrect
describe("UserService", () => {
  it("works", () => {});
  it("error", () => {});
});

// ✅ Correct
describe("UserService", () => {
  describe("findById", () => {
    it("should return the user when a valid ID is provided", () => {});
    it("should throw NotFoundError when user does not exist", () => {});
    it("should throw ValidationError when ID format is invalid", () => {});
  });
});
```

## Test Database

- Use a **separate test database** — never run tests against development or production:

```
# .env.test
DATABASE_URL="postgresql://user:pass@localhost:5432/myapp_test"
```

- Reset the database before each test suite:

```ts
beforeAll(async () => {
  await prisma.$executeRaw`TRUNCATE TABLE users, orders CASCADE`;
});
```

- Use factories or fixtures for test data — never hardcode IDs or timestamps.

## Mocking

- Mock **external dependencies** (HTTP calls, email services, file storage):

```ts
// ✅ Correct — mock external HTTP call
jest.spyOn(httpService, "get").mockResolvedValue({ data: mockResponse });

// ✅ Correct — mock email service
jest.spyOn(emailService, "send").mockResolvedValue(undefined);
```

- Do **NOT** mock the module you are testing — mock only its dependencies:

```ts
// ❌ Incorrect — mocking the service we're testing
jest.spyOn(userService, "findById").mockResolvedValue(mockUser);

// ✅ Correct — mock the repository/ORM the service depends on
jest.spyOn(prisma.user, "findUnique").mockResolvedValue(mockUser);
```

## API Testing

- Test full request/response cycle with supertest:

```ts
import request from "supertest";

describe("GET /api/users/:id", () => {
  it("should return 200 and user data for valid ID", async () => {
    const res = await request(app)
      .get(`/api/users/${testUser.id}`)
      .set("Authorization", `Bearer ${token}`)
      .expect(200);

    expect(res.body.data).toMatchObject({
      id: testUser.id,
      name: testUser.name,
    });
  });

  it("should return 404 for non-existent user", async () => {
    await request(app)
      .get("/api/users/non-existent-id")
      .set("Authorization", `Bearer ${token}`)
      .expect(404);
  });

  it("should return 401 without auth token", async () => {
    await request(app)
      .get(`/api/users/${testUser.id}`)
      .expect(401);
  });
});
```

## Coverage Requirements

- Minimum expected coverage per new file: **80%** (statements and branches).
- Critical business logic (payments, auth, data processing): **90%+** coverage.

## Running Tests

```bash
# Run all tests
npm run test

# Run specific test file
npm run test -- --filter UserService

# Run with coverage
npm run test -- --coverage

# Run e2e tests
npm run test:e2e
```

## Rules

- Never commit code without corresponding tests.
- Test the behavior, not the implementation — tests should not break when internals are refactored.
- Each test must be independent — no shared mutable state between tests.
- Keep tests fast — mock expensive operations (DB, HTTP, file I/O).
- Use descriptive assertion messages so failures are easy to diagnose.
- Test both happy paths and error paths — edge cases cause production bugs.
