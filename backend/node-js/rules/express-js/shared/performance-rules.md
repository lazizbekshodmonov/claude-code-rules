# Performance Rules (Express)

> Express-specific performance rules covering middleware, caching, database, and async patterns.

## Async Operations

- Run independent operations in **parallel**:

```ts
// ❌ Incorrect — sequential
router.get("/dashboard", authenticate, asyncHandler(async (req, res) => {
  const user = await userService.findById(req.user.id);
  const orders = await orderService.findByUser(req.user.id);
  const stats = await statsService.getForUser(req.user.id);
  res.json({ data: { user, orders, stats } });
}));

// ✅ Correct — parallel
router.get("/dashboard", authenticate, asyncHandler(async (req, res) => {
  const [user, orders, stats] = await Promise.all([
    userService.findById(req.user.id),
    orderService.findByUser(req.user.id),
    statsService.getForUser(req.user.id),
  ]);
  res.json({ data: { user, orders, stats } });
}));
```

## Response Timing Middleware

```ts
app.use((req, res, next) => {
  const start = Date.now();
  res.on("finish", () => {
    const duration = Date.now() - start;
    if (duration > 500) {
      logger.warn("Slow request", {
        method: req.method,
        path: req.originalUrl,
        statusCode: res.statusCode,
        duration,
      });
    }
  });
  next();
});
```

## Compression

```ts
import compression from "compression";
app.use(compression());
```

## Payload Size Limits

```ts
app.use(express.json({ limit: "1mb" }));
app.use(express.urlencoded({ extended: true, limit: "1mb" }));
```

## Database Performance

- Select only required fields:

```ts
const users = await userRepository.find({
  select: { id: true, name: true, email: true },
});
```

- Avoid N+1 — use relations or QueryBuilder:

```ts
const users = await userRepository.find({
  relations: { orders: true },
});
```

- Paginate all list endpoints:

```ts
async function findAll(page: number, limit: number): Promise<[User[], number]> {
  return userRepository.findAndCount({
    skip: (page - 1) * limit,
    take: limit,
    order: { createdAt: "DESC" },
  });
}
```

## Background Jobs

- Use Bull/BullMQ for heavy processing:

```ts
import Queue from "bull";

const reportQueue = new Queue("reports", process.env.REDIS_URL!);

// Producer — add job
router.post("/reports", authenticate, asyncHandler(async (req, res) => {
  const job = await reportQueue.add("generate", req.body);
  res.status(202).json({ jobId: job.id, status: "processing" });
}));

// Consumer — process job
reportQueue.process("generate", async (job) => {
  const report = await reportService.generate(job.data);
  return report;
});
```

## Streaming Large Data

```ts
router.get("/export/users", authenticate, authorize("admin"), asyncHandler(async (req, res) => {
  res.setHeader("Content-Type", "text/csv");
  res.setHeader("Content-Disposition", "attachment; filename=users.csv");

  const stream = await userRepository
    .createQueryBuilder("user")
    .stream();

  stream.pipe(new CsvTransform()).pipe(res);
}));
```

## Timeouts

```ts
import axios from "axios";

const httpClient = axios.create({
  timeout: 5000,
  baseURL: process.env.PAYMENT_API_URL,
});
```

## Static File Caching

```ts
// Cache static files for 1 year
app.use("/static", express.static("public", {
  maxAge: "1y",
  immutable: true,
}));
```

## Rules

- Never block the event loop — use async I/O everywhere.
- Use `Promise.all` for independent parallel operations.
- Paginate all list endpoints.
- Enable compression in production.
- Use queues for operations taking more than 5 seconds.
- Set timeouts on all external HTTP calls.
- Log slow requests — monitor response times.
- Limit request body size to prevent abuse.
