# Performance Rules

> Backend performance rules for Claude Code. Apply to all Node.js backend projects.

## Async Operations

- Always use **async/await** for I/O operations — never block the event loop:

```ts
// ❌ Incorrect — synchronous file read blocks event loop
const data = fs.readFileSync("/path/to/file");

// ✅ Correct — async file read
const data = await fs.promises.readFile("/path/to/file");
```

- Run independent async operations in **parallel** with `Promise.all`:

```ts
// ❌ Incorrect — sequential (slow)
const user = await userService.findById(userId);
const orders = await orderService.findByUserId(userId);
const notifications = await notificationService.findByUserId(userId);

// ✅ Correct — parallel (fast)
const [user, orders, notifications] = await Promise.all([
  userService.findById(userId),
  orderService.findByUserId(userId),
  notificationService.findByUserId(userId),
]);
```

## Caching

- Cache frequently read, rarely changed data to reduce database load:

```ts
// ✅ Correct — Redis cache with TTL
async findById(id: string): Promise<User> {
  const cacheKey = `user:${id}`;
  const cached = await redis.get(cacheKey);
  if (cached) return JSON.parse(cached);

  const user = await prisma.user.findUnique({ where: { id } });
  if (user) await redis.set(cacheKey, JSON.stringify(user), "EX", 300);
  return user;
}
```

- **Invalidate cache** when data changes:

```ts
async update(id: string, data: UpdateUserDto): Promise<User> {
  const user = await prisma.user.update({ where: { id }, data });
  await redis.del(`user:${id}`);
  return user;
}
```

- Cache candidates: configuration, user profiles, static lookups, permission lists.
- Do NOT cache: rapidly changing data, user-specific session data, sensitive data.

## Database Performance

- Use **indexes** on columns used in WHERE, JOIN, and ORDER BY.
- Select only required fields — avoid `SELECT *`.
- Use pagination for list endpoints — never return unbounded collections.
- Avoid N+1 queries — use eager loading or batch queries.
- Monitor slow queries — set a threshold (e.g., 500ms) and log them.

## Connection Pooling

- Reuse database and Redis connections — never create a new connection per request:

```ts
// ❌ Incorrect — new connection per request
app.get("/users", async (req, res) => {
  const client = new Client();
  await client.connect();
  const result = await client.query("SELECT * FROM users");
  await client.end();
  res.json(result.rows);
});

// ✅ Correct — shared pool
const pool = new Pool({ max: 20 });
app.get("/users", async (req, res) => {
  const result = await pool.query("SELECT * FROM users");
  res.json(result.rows);
});
```

## Payload Size

- Compress responses with **gzip/brotli**:

```ts
import compression from "compression";
app.use(compression());
```

- Limit request body size to prevent abuse:

```ts
app.use(express.json({ limit: "1mb" }));
```

- Return only the data the client needs — avoid over-fetching.

## Background Jobs

- Move heavy processing to **background jobs** (Bull, Agenda, BullMQ):

```ts
// ❌ Incorrect — heavy processing in request handler
app.post("/reports", async (req, res) => {
  const report = await generateLargeReport(req.body); // takes 30 seconds
  res.json(report);
});

// ✅ Correct — queue for background processing
app.post("/reports", async (req, res) => {
  const job = await reportQueue.add("generate", req.body);
  res.status(202).json({ jobId: job.id, status: "processing" });
});
```

- Use queues for: email sending, report generation, image processing, data imports.

## Memory Management

- Avoid loading large datasets into memory — use streams for large files:

```ts
// ❌ Incorrect — loads entire file into memory
const data = await fs.promises.readFile("large-file.csv", "utf-8");

// ✅ Correct — stream processing
const stream = fs.createReadStream("large-file.csv");
stream.pipe(csvParser()).on("data", (row) => processRow(row));
```

- Monitor memory usage in production — set up alerts for unusual growth.
- Avoid closures that hold references to large objects longer than needed.

## Timeouts

- Set timeouts on all external calls — never wait indefinitely:

```ts
// HTTP client timeout
const response = await axios.get(url, { timeout: 5000 });

// Database query timeout
await prisma.$queryRaw`SELECT * FROM users`; // configure in connection string
```

## Rules

- Performance is a feature — consider it during design, not after deployment.
- Measure before optimizing — use profiling tools (clinic.js, 0x) to find actual bottlenecks.
- Never block the Node.js event loop — use async I/O for everything.
- Cache aggressively but invalidate correctly — stale data causes bugs.
- Set up monitoring and alerting for response times, memory, and CPU usage.
- Load test critical endpoints before release — know your system's limits.
