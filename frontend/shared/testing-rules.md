# Testing Rules

> Universal testing requirements for Claude Code. Apply to all projects regardless of tech stack.

## Mandatory Testing

- **Every** new or modified component, module, and function must have accompanying tests. No exceptions.
- All tests **must pass** before a task status can be marked as "Completed" in the planning file.
- If a test fails, fix the code or the test — never skip or delete a failing test.

## Test Types

### Unit Tests

Must cover:
- Individual function input/output correctness
- Edge cases (null, undefined, empty, boundary values)
- Error handling and thrown exceptions
- Component rendering with different props/state
- Event emissions and user interaction handlers

### Integration Tests

Must cover:
- Component interaction with child components
- API call flows (mocked)
- Store / state management integration
- Route-dependent behavior (if applicable)
- Form submission and validation flows

## Test File Location

Place test files close to the source code they test:

```
some-module/
├── some-module.ts
├── some-module.types.ts
├── __tests__/
│   ├── some-module.spec.ts              # Unit tests
│   └── some-module.integration.spec.ts  # Integration tests
└── index.ts
```

## Naming Conventions

- Unit test files: `{name}.spec.ts` or `{name}.test.ts`
- Integration test files: `{name}.integration.spec.ts`
- Test descriptions: Use clear, human-readable sentences

```ts
// ❌ Incorrect
describe("fn", () => {
  it("works", () => {});
});

// ✅ Correct
describe("calculateDiscount", () => {
  it("should return the discounted price when given a valid percentage", () => {});
  it("should return the original price when discount is 0", () => {});
  it("should throw an error when percentage is negative", () => {});
});
```

## Coverage Requirements

- Minimum expected coverage per new file: **80%** (statements and branches).
- Critical business logic (payments, auth, data processing): **90%+** coverage.

## Running Tests

- **Always run tests** after writing them before considering the task complete:

```bash
# Run all tests
npm run test

# Run specific test file
npm run test -- --filter SomeModule

# Run with coverage
npm run test -- --coverage
```

## Rules

- Never commit code without corresponding tests.
- Test the behavior, not the implementation — tests should not break when internal code is refactored.
- Mock external dependencies (APIs, databases, third-party services) — never call real services in tests.
- Each test must be independent — tests should not depend on execution order or shared mutable state.
- Use descriptive assertion messages so failures are easy to diagnose.
- Keep tests fast — if a test takes more than 2 seconds, it likely needs optimization.
