# Error Handling Rules (Express)

> Express-specific error handling using middleware, custom error classes, and async wrappers.

## Custom Error Classes

```ts
// src/errors/app.error.ts
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

## Async Handler Wrapper

- Express does not catch async errors automatically — use a wrapper:

```ts
// src/utils/async-handler.ts
type AsyncRequestHandler = (req: Request, res: Response, next: NextFunction) => Promise<any>;

export const asyncHandler = (fn: AsyncRequestHandler) =>
  (req: Request, res: Response, next: NextFunction) =>
    Promise.resolve(fn(req, res, next)).catch(next);
```

- Usage in routes:

```ts
// ❌ Incorrect — unhandled promise rejection
router.get("/users/:id", async (req, res) => {
  const user = await userService.findById(req.params.id);
  res.json({ data: user });
});

// ✅ Correct — errors forwarded to error handler
router.get("/users/:id", asyncHandler(async (req, res) => {
  const user = await userService.findById(req.params.id);
  res.json({ data: user });
}));
```

## Global Error Handler Middleware

- Register as the **last middleware** in the app:

```ts
// src/middleware/error-handler.ts
const errorHandler = (err: Error, req: Request, res: Response, next: NextFunction) => {
  if (err instanceof AppError) {
    return res.status(err.statusCode).json({
      message: err.message,
      code: err.code,
      errors: err.errors,
    });
  }

  // TypeORM unique violation
  if ((err as any).code === "23505") {
    return res.status(409).json({
      message: "Resource already exists",
      code: "CONFLICT",
    });
  }

  // Unexpected error
  logger.error("Unhandled error", {
    error: err.message,
    stack: err.stack,
    path: req.path,
    method: req.method,
  });

  return res.status(500).json({
    message: "Internal server error",
    code: "INTERNAL_ERROR",
  });
};

// Register last
app.use(errorHandler);
```

## Input Validation with Zod

- Validate request body, params, and query with a middleware:

```ts
import { z, ZodSchema } from "zod";

const validate = (schema: { body?: ZodSchema; params?: ZodSchema; query?: ZodSchema }) =>
  (req: Request, res: Response, next: NextFunction) => {
    try {
      if (schema.body) req.body = schema.body.parse(req.body);
      if (schema.params) req.params = schema.params.parse(req.params) as any;
      if (schema.query) req.query = schema.query.parse(req.query) as any;
      next();
    } catch (err) {
      if (err instanceof z.ZodError) {
        throw new ValidationError(
          err.errors.map((e) => ({ field: e.path.join("."), message: e.message })),
        );
      }
      throw err;
    }
  };

// Usage
const createUserSchema = {
  body: z.object({
    email: z.string().email(),
    name: z.string().min(2).max(100),
    password: z.string().min(8),
  }),
};

router.post("/users", validate(createUserSchema), asyncHandler(async (req, res) => {
  const user = await userService.create(req.body);
  res.status(201).json({ data: user });
}));
```

## Service Layer Pattern

- Services throw errors — routes stay thin:

```ts
// ❌ Incorrect — error logic in route
router.get("/users/:id", asyncHandler(async (req, res) => {
  const user = await userRepository.findOneBy({ id: req.params.id });
  if (!user) return res.status(404).json({ message: "Not found" });
  res.json({ data: user });
}));

// ✅ Correct — service throws, route delegates
router.get("/users/:id", asyncHandler(async (req, res) => {
  const user = await userService.findById(req.params.id);
  res.json({ data: user });
}));
```

## 404 Handler

- Add a catch-all for undefined routes:

```ts
// After all routes, before error handler
app.use((req, res) => {
  res.status(404).json({
    message: `Route ${req.method} ${req.path} not found`,
    code: "ROUTE_NOT_FOUND",
  });
});
```

## Process-Level Errors

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

- Always use `asyncHandler` wrapper — Express does not catch async errors.
- Use custom `AppError` classes — never throw raw strings or generic `Error`.
- Register error handler as the last middleware.
- Services throw errors — routes never contain error logic.
- Never expose stack traces or internal details in responses.
- Validate all inputs with Zod middleware — reject bad data at the edge.
- Log all unexpected errors with full context (path, method, body).
