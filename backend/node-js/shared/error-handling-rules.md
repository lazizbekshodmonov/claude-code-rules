# Error Handling Rules

> Backend error handling rules for Claude Code. Apply to all Node.js backend projects.

## Custom Error Classes

- Define a base application error and extend it for specific cases:

```ts
// src/common/errors/app.error.ts
export class AppError extends Error {
  constructor(
    public readonly message: string,
    public readonly statusCode: number,
    public readonly code: string,
    public readonly errors?: Record<string, string>[],
  ) {
    super(message);
    this.name = this.constructor.name;
  }
}

export class NotFoundError extends AppError {
  constructor(resource: string, id: string | number) {
    super(`${resource} with id ${id} not found`, 404, "NOT_FOUND");
  }
}

export class ValidationError extends AppError {
  constructor(errors: Record<string, string>[]) {
    super("Validation failed", 422, "VALIDATION_ERROR", errors);
  }
}

export class UnauthorizedError extends AppError {
  constructor(message = "Authentication required") {
    super(message, 401, "UNAUTHORIZED");
  }
}

export class ForbiddenError extends AppError {
  constructor(message = "Insufficient permissions") {
    super(message, 403, "FORBIDDEN");
  }
}

export class ConflictError extends AppError {
  constructor(message: string) {
    super(message, 409, "CONFLICT");
  }
}
```

## Throwing Errors

- Use custom error classes — never throw raw strings or generic `Error`:

```ts
// ❌ Incorrect
throw new Error("Not found");
throw "Something went wrong";

// ✅ Correct
throw new NotFoundError("User", userId);
throw new ValidationError([{ field: "email", message: "Invalid format" }]);
```

## Global Error Handler

- Implement a centralized error handler that catches all errors:

```ts
// Express middleware
const errorHandler = (err: Error, req: Request, res: Response, next: NextFunction) => {
  if (err instanceof AppError) {
    return res.status(err.statusCode).json({
      message: err.message,
      code: err.code,
      errors: err.errors,
    });
  }

  // Unexpected error — log and return generic message
  logger.error("Unhandled error", { error: err, path: req.path, method: req.method });

  return res.status(500).json({
    message: "Internal server error",
    code: "INTERNAL_ERROR",
  });
};

app.use(errorHandler);
```

- NestJS: Use exception filters:

```ts
@Catch()
export class AllExceptionsFilter implements ExceptionFilter {
  catch(exception: unknown, host: ArgumentsHost) {
    const ctx = host.switchToHttp();
    const response = ctx.getResponse<Response>();

    if (exception instanceof AppError) {
      return response.status(exception.statusCode).json({
        message: exception.message,
        code: exception.code,
        errors: exception.errors,
      });
    }

    return response.status(500).json({
      message: "Internal server error",
      code: "INTERNAL_ERROR",
    });
  }
}
```

## Error Response Format

- Use a consistent error response structure:

```json
{
  "message": "Validation failed",
  "code": "VALIDATION_ERROR",
  "errors": [
    { "field": "email", "message": "Invalid email format" },
    { "field": "age", "message": "Must be at least 18" }
  ]
}
```

- Never expose internal details (stack traces, SQL queries, file paths) in responses:

```ts
// ❌ Incorrect
res.status(500).json({ error: err.stack, query: err.query });

// ✅ Correct
res.status(500).json({ message: "Internal server error" });
```

## Async Error Handling

- Always wrap async route handlers to catch unhandled promise rejections:

```ts
// Express — use a wrapper
const asyncHandler = (fn: Function) => (req: Request, res: Response, next: NextFunction) =>
  Promise.resolve(fn(req, res, next)).catch(next);

router.get("/users/:id", asyncHandler(async (req, res) => {
  const user = await userService.findById(req.params.id);
  if (!user) throw new NotFoundError("User", req.params.id);
  res.json({ data: user });
}));
```

- NestJS handles this automatically with its built-in exception layer.

## Service Layer Errors

- Services should throw domain-specific errors — controllers should not contain error logic:

```ts
// ❌ Incorrect — error logic in controller
async getUser(req, res) {
  const user = await userRepository.findOneBy({ id: req.params.id });
  if (!user) return res.status(404).json({ message: "Not found" });
  res.json(user);
}

// ✅ Correct — service throws, controller delegates
// service
async findById(id: string): Promise<User> {
  const user = await this.userRepository.findOneBy({ id });
  if (!user) throw new NotFoundError("User", id);
  return user;
}

// controller
async getUser(req, res) {
  const user = await userService.findById(req.params.id);
  res.json({ data: user });
}
```

## Unhandled Rejections and Exceptions

- Catch unhandled process-level errors for graceful shutdown:

```ts
process.on("unhandledRejection", (reason: Error) => {
  logger.fatal("Unhandled rejection", { error: reason });
  process.exit(1);
});

process.on("uncaughtException", (error: Error) => {
  logger.fatal("Uncaught exception", { error });
  process.exit(1);
});
```

## Rules

- Never swallow errors silently — every `catch` block must either handle or re-throw.
- Log all errors with sufficient context (request path, user ID, input data) for debugging.
- Use appropriate HTTP status codes — a validation error is not a 500.
- Return user-friendly error messages — the client should understand what went wrong.
- Never use `try-catch` around code that cannot throw — it adds noise without value.
- Test error paths — happy path tests are not enough.
