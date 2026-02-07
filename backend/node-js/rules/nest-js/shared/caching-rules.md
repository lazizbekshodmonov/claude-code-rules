# Caching Rules (NestJS)

> NestJS-specific caching rules using `@nestjs/cache-manager`, Redis, and custom decorators.

## Cache Module Setup

- Register the cache module with Redis store:

```ts
// app.module.ts
import { CacheModule } from "@nestjs/cache-manager";
import { redisStore } from "cache-manager-redis-yet";

@Module({
  imports: [
    CacheModule.registerAsync({
      isGlobal: true,
      useFactory: async () => ({
        store: await redisStore({
          socket: { host: process.env.REDIS_HOST, port: +process.env.REDIS_PORT },
        }),
        ttl: 300_000, // default 5 min (ms)
      }),
    }),
  ],
})
export class AppModule {}
```

## Cache Interceptor (Auto-Caching)

- Use `@CacheKey()` and `@CacheTTL()` decorators for automatic GET caching:

```ts
@Controller("products")
@UseInterceptors(CacheInterceptor)
export class ProductsController {
  @Get()
  @CacheKey("products_list")
  @CacheTTL(60_000) // 1 minute
  async findAll() {
    return this.productsService.findAll();
  }

  @Get(":id")
  @CacheKey("product_detail")
  @CacheTTL(300_000) // 5 minutes
  async findById(@Param("id") id: string) {
    return this.productsService.findById(id);
  }
}
```

- **Do NOT** apply `CacheInterceptor` on POST/PUT/PATCH/DELETE endpoints.

## Manual Caching in Services

- For fine-grained control, inject `CACHE_MANAGER` directly:

```ts
@Injectable()
export class UsersService {
  constructor(
    @InjectRepository(User)
    private readonly userRepository: Repository<User>,
    @Inject(CACHE_MANAGER)
    private readonly cacheManager: Cache,
  ) {}

  async findById(id: string): Promise<User> {
    const cacheKey = `user:${id}`;
    const cached = await this.cacheManager.get<User>(cacheKey);
    if (cached) return cached;

    const user = await this.userRepository.findOneBy({ id });
    if (!user) throw new NotFoundException(`User ${id} not found`);

    await this.cacheManager.set(cacheKey, user, 300_000); // 5 min
    return user;
  }
}
```

## Cache Invalidation

- **Always invalidate** when data changes — stale cache causes bugs:

```ts
async update(id: string, dto: UpdateUserDto): Promise<User> {
  const user = await this.findById(id);
  Object.assign(user, dto);
  const updated = await this.userRepository.save(user);

  // Invalidate related caches
  await this.cacheManager.del(`user:${id}`);
  await this.cacheManager.del("users_list");

  return updated;
}

async delete(id: string): Promise<void> {
  await this.userRepository.softDelete(id);

  await this.cacheManager.del(`user:${id}`);
  await this.cacheManager.del("users_list");
}
```

## Cache Key Patterns

- Use consistent, descriptive cache key patterns:

```ts
// ✅ Good patterns
`user:${id}`                    // single entity
`users:list:page:${page}`       // paginated list
`users:role:${role}`            // filtered list
`config:app_settings`           // app-wide config
`permissions:user:${userId}`    // user permissions

// ❌ Bad patterns
`u_${id}`                       // cryptic
`data`                          // too generic
`cache_1`                       // meaningless
```

## What to Cache

| Cache | TTL | Invalidation |
|-------|-----|--------------|
| User profile | 5 min | On update/delete |
| Permission list | 10 min | On role change |
| Configuration | 30 min | On config update |
| Product catalog | 5 min | On product CRUD |
| Static lookups (countries, categories) | 1 hour | On change |

## What NOT to Cache

- Rapidly changing data (real-time counters, live feeds)
- User-specific session data
- Sensitive data (tokens, passwords, personal info)
- Data that must always be fresh (payment status, inventory count)

## Cache Warming

- Pre-populate cache on app startup for frequently accessed data:

```ts
@Injectable()
export class CacheWarmupService implements OnModuleInit {
  constructor(
    @Inject(CACHE_MANAGER) private readonly cacheManager: Cache,
    private readonly configService: AppConfigService,
  ) {}

  async onModuleInit() {
    const config = await this.configService.loadAll();
    await this.cacheManager.set("config:app_settings", config, 1_800_000);
  }
}
```

## Redis Direct Access

- For advanced patterns (counters, sets, pub/sub), use Redis client directly:

```ts
@Injectable()
export class RateLimitService {
  constructor(@Inject("REDIS_CLIENT") private readonly redis: Redis) {}

  async checkLimit(userId: string, limit: number): Promise<boolean> {
    const key = `rate:${userId}`;
    const current = await this.redis.incr(key);
    if (current === 1) await this.redis.expire(key, 60);
    return current <= limit;
  }
}
```

## Rules

- Cache frequently read, rarely written data — not everything.
- Always invalidate cache on create/update/delete — stale data is worse than no cache.
- Use descriptive cache keys with consistent naming patterns.
- Set appropriate TTLs — shorter for volatile data, longer for static.
- Never cache sensitive data (tokens, passwords, personal info).
- Monitor cache hit/miss ratio — low hit rate means caching is not helping.
- Use `CacheInterceptor` for simple GET caching, manual caching for complex logic.
- Implement cache warming for critical startup data.
