# Caching Rules (Express)

> Express-specific caching rules using Redis and middleware patterns.

## Redis Client Setup

```ts
// src/lib/redis.ts
import Redis from "ioredis";

export const redis = new Redis(process.env.REDIS_URL || "redis://localhost:6379");

redis.on("error", (err) => logger.error("Redis error", { error: err.message }));
redis.on("connect", () => logger.info("Redis connected"));
```

## Cache Middleware

- Create reusable caching middleware for GET endpoints:

```ts
// src/middleware/cache.ts
export const cacheMiddleware = (ttlSeconds: number, keyPrefix?: string) =>
  asyncHandler(async (req: Request, res: Response, next: NextFunction) => {
    if (req.method !== "GET") return next();

    const key = keyPrefix
      ? `${keyPrefix}:${req.originalUrl}`
      : `cache:${req.originalUrl}`;

    const cached = await redis.get(key);
    if (cached) {
      return res.json(JSON.parse(cached));
    }

    // Override res.json to cache the response
    const originalJson = res.json.bind(res);
    res.json = (body: any) => {
      redis.set(key, JSON.stringify(body), "EX", ttlSeconds);
      return originalJson(body);
    };

    next();
  });

// Usage
router.get("/products",
  cacheMiddleware(300), // 5 minutes
  asyncHandler(async (req, res) => {
    const products = await productService.findAll(req.query);
    res.json({ data: products });
  }),
);

router.get("/products/:id",
  cacheMiddleware(600, "product"), // 10 minutes
  asyncHandler(async (req, res) => {
    const product = await productService.findById(req.params.id);
    res.json({ data: product });
  }),
);
```

## Manual Caching in Services

```ts
export const userService = {
  async findById(id: string): Promise<User> {
    const cacheKey = `user:${id}`;
    const cached = await redis.get(cacheKey);
    if (cached) return JSON.parse(cached);

    const user = await userRepository.findOneBy({ id });
    if (!user) throw new NotFoundError("User", id);

    await redis.set(cacheKey, JSON.stringify(user), "EX", 300);
    return user;
  },
};
```

## Cache Invalidation

- **Always invalidate** when data changes:

```ts
export const userService = {
  async update(id: string, data: UpdateUserDto): Promise<User> {
    const user = await userService.findById(id);
    Object.assign(user, data);
    const updated = await userRepository.save(user);

    // Invalidate caches
    await redis.del(`user:${id}`);
    await invalidatePattern("cache:/api/users*");

    return updated;
  },

  async delete(id: string): Promise<void> {
    await userRepository.softDelete(id);
    await redis.del(`user:${id}`);
    await invalidatePattern("cache:/api/users*");
  },
};

// Helper to delete keys by pattern
async function invalidatePattern(pattern: string): Promise<void> {
  const keys = await redis.keys(pattern);
  if (keys.length > 0) await redis.del(...keys);
}
```

## Cache Key Patterns

```ts
// ✅ Good patterns
`user:${id}`                    // single entity
`cache:/api/users?page=1`      // URL-based (middleware)
`config:app_settings`          // app-wide config
`permissions:user:${userId}`   // user permissions

// ❌ Bad patterns
`u_${id}`                      // cryptic
`data`                         // too generic
```

## What to Cache / What NOT to Cache

| Cache | TTL | Invalidation |
|-------|-----|--------------|
| User profile | 5 min | On update/delete |
| Product catalog | 5 min | On product CRUD |
| Configuration | 30 min | On config update |
| Static lookups | 1 hour | On change |

Do NOT cache:
- Rapidly changing data (counters, live feeds)
- Sensitive data (tokens, passwords)
- Data that must always be fresh (payment status, inventory)

## ETag / Conditional Requests

- Use ETags for client-side caching:

```ts
import etag from "etag";

app.set("etag", "strong");
// Express will automatically send 304 Not Modified when appropriate
```

## Cache Warming

```ts
// Warm cache on app startup
async function warmCache(): Promise<void> {
  const config = await configService.loadAll();
  await redis.set("config:app_settings", JSON.stringify(config), "EX", 1800);

  const categories = await categoryService.findAll();
  await redis.set("categories:all", JSON.stringify(categories), "EX", 3600);
}

// Call after app initialization
warmCache().catch((err) => logger.error("Cache warmup failed", { error: err }));
```

## Graceful Shutdown

```ts
process.on("SIGTERM", async () => {
  await redis.quit();
  await dataSource.destroy();
  process.exit(0);
});
```

## Rules

- Cache frequently read, rarely written data — not everything.
- Always invalidate on create/update/delete — stale data is worse than no cache.
- Use consistent cache key naming patterns.
- Set appropriate TTLs — shorter for volatile, longer for static data.
- Never cache sensitive data.
- Use the cache middleware for simple GET caching, manual caching for complex logic.
- Monitor cache hit/miss ratio.
- Warm critical caches on app startup.
