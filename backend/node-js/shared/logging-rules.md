# Logging Rules

> Backend logging rules for Claude Code. Apply to all Node.js backend projects.

## Logging Library

- Use a **structured logging** library (pino, winston) — never use `console.log` in production code:

```ts
// ❌ Incorrect
console.log("User created:", user);
console.error("Failed to fetch:", error);

// ✅ Correct
logger.info("User created", { userId: user.id, email: user.email });
logger.error("Failed to fetch user", { userId, error: error.message });
```

## Log Levels

Use appropriate log levels — each has a specific purpose:

| Level | When to Use | Example |
|-------|-------------|---------|
| `fatal` | App cannot continue, crashing | Unhandled exception, DB connection lost |
| `error` | Operation failed, needs attention | API call failed, payment error |
| `warn` | Something unusual but not broken | Deprecated endpoint used, slow query |
| `info` | Normal operations, audit trail | User login, order created, server started |
| `debug` | Detailed info for development | Request payload, query result, cache hit/miss |
| `trace` | Very verbose, step-by-step | Function entry/exit, variable values |

```ts
// ❌ Incorrect — wrong level
logger.error("Server started on port 3000"); // This is info, not error
logger.info("Database connection failed");     // This is error, not info

// ✅ Correct
logger.info("Server started", { port: 3000 });
logger.error("Database connection failed", { host: dbHost, error: err.message });
```

## Structured Logging

- Always log as **structured data** (key-value pairs), not raw strings:

```ts
// ❌ Incorrect — unstructured
logger.info(`User ${userId} created order ${orderId} for $${amount}`);

// ✅ Correct — structured
logger.info("Order created", { userId, orderId, amount, currency: "USD" });
```

- Structured logs are searchable, filterable, and parseable by log aggregation tools.

## Request Logging

- Log every incoming HTTP request with relevant metadata:

```ts
// Middleware
app.use((req, res, next) => {
  const start = Date.now();
  res.on("finish", () => {
    logger.info("HTTP request", {
      method: req.method,
      path: req.originalUrl,
      statusCode: res.statusCode,
      duration: Date.now() - start,
      userAgent: req.get("user-agent"),
      ip: req.ip,
    });
  });
  next();
});
```

## Correlation ID

- Assign a unique **correlation ID** to each request for tracing across services:

```ts
import { randomUUID } from "crypto";

app.use((req, res, next) => {
  req.correlationId = req.headers["x-correlation-id"] || randomUUID();
  res.setHeader("x-correlation-id", req.correlationId);
  next();
});

// Include in all log entries
logger.info("Processing order", { correlationId: req.correlationId, orderId });
```

## Sensitive Data

- **NEVER** log sensitive data:

```ts
// ❌ Incorrect — logs password, token, credit card
logger.info("Login attempt", { email, password });
logger.info("Auth token", { token: jwt });
logger.info("Payment", { cardNumber, cvv });

// ✅ Correct — redact sensitive fields
logger.info("Login attempt", { email });
logger.info("Auth token issued", { userId, expiresIn: "1h" });
logger.info("Payment processed", { orderId, amount, last4: "4242" });
```

- Redact or mask: passwords, tokens, API keys, credit card numbers, SSN, personal data.

## Error Logging

- Log errors with full context for debugging:

```ts
// ❌ Incorrect — no context
logger.error(error.message);

// ✅ Correct — full context
logger.error("Failed to create order", {
  userId,
  orderData,
  error: error.message,
  stack: error.stack,
});
```

- In production, send errors to an error tracking service (Sentry, Datadog, etc.).

## Log Configuration

- Use **JSON format** in production for log aggregation tools:
- Use **pretty print** in development for readability:

```ts
const logger = pino({
  level: process.env.LOG_LEVEL || "info",
  transport: process.env.NODE_ENV === "development"
    ? { target: "pino-pretty" }
    : undefined,
});
```

## Rules

- Never use `console.log` in production code — use the project's logger.
- Every log entry must have enough context to understand what happened without reading the code.
- Log at the appropriate level — do not abuse `error` for non-errors.
- Never log sensitive data — passwords, tokens, personal information.
- Use correlation IDs for request tracing across services.
- Configure log rotation in production to prevent disk space issues.
- Review logs regularly — they are useless if nobody reads them.
