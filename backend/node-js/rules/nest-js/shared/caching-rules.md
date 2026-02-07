# Caching Rules (NestJS)

> NestJS-specific caching rules using a custom RedisModule/RedisService pattern — not `@nestjs/cache-manager`.

## Redis Module Setup

- Use a global `RedisModule` with `ioredis` — registered as a custom provider:

```ts
// common/modules/redis-module/redis.module.ts
@Global()
@Module({
  imports: [ConfigModule],
  providers: [
    {
      provide: "REDIS_CLIENT",
      inject: [ConfigService],
      useFactory: (configService: ConfigService) => {
        return new Redis({
          host: configService.get<string>("database.redis.host"),
          port: configService.get<number>("database.redis.port"),
        });
      },
    },
    RedisService,
  ],
  exports: ["REDIS_CLIENT", RedisService],
})
export class RedisModule {}
```

## RedisService

- Wraps the Redis client with typed methods:

```ts
// common/modules/redis-module/redis.service.ts
@Injectable()
export class RedisService {
  constructor(@Inject("REDIS_CLIENT") private readonly redisClient: Redis) {}

  async set(key: string, value: any, ttl?: number) {
    const data = typeof value === "string" ? value : JSON.stringify(value);
    if (ttl) {
      return this.redisClient.set(key, data, "EX", ttl);
    }
    return this.redisClient.set(key, data);
  }

  async get<T = any>(key: string): Promise<T | null> {
    const data = await this.redisClient.get(key);
    if (data === null) return null;
    try {
      return JSON.parse(data) as T;
    } catch {
      return data as T;
    }
  }

  async del(key: string) {
    return this.redisClient.del(key);
  }

  async incr(key: string) {
    return this.redisClient.incr(key);
  }

  async expire(key: string, seconds: number) {
    return this.redisClient.expire(key, seconds);
  }
}
```

- **Do NOT** use `@nestjs/cache-manager` or `CacheInterceptor` — use `RedisService` directly.

## Usage in Services

```ts
@Injectable()
export class AuthService {
  constructor(private readonly redisService: RedisService) {}

  // Store OTP with TTL
  async sendOtp(email: string): Promise<void> {
    const otp = generateOtp(6);
    await this.redisService.set(`otp:${email}`, otp, 300); // 5 minutes
    await this.mailService.sendOtp(email, otp);
  }

  // Verify OTP
  async verifyOtp(email: string, code: string): Promise<boolean> {
    const stored = await this.redisService.get<string>(`otp:${email}`);
    if (stored !== code) {
      throw new AppException(AuthError.INVALID_OTP_EXPIRATION, HttpStatus.BAD_REQUEST);
    }
    await this.redisService.del(`otp:${email}`);
    return true;
  }
}
```

## Rate Limiting with Redis

```ts
@Injectable()
export class OtpRateLimitService {
  constructor(private readonly redisService: RedisService) {}

  async checkOtpLimit(email: string, maxAttempts: number, windowMinutes: number): Promise<void> {
    const key = `otp:limit:${email}`;
    const count = await this.redisService.incr(key);

    if (count === 1) {
      await this.redisService.expire(key, windowMinutes * 60);
    }

    if (count > maxAttempts) {
      throw new AppException(AuthError.OTP_LIMIT_EXCEEDED, HttpStatus.TOO_MANY_REQUESTS);
    }
  }
}
```

## Session / Token Storage

```ts
// Store refresh token for invalidation
await this.redisService.set(`refresh:${userId}`, refreshToken, 86400); // 1 day

// Blacklist revoked tokens
await this.redisService.set(`blacklist:${tokenId}`, "1", tokenTtlSeconds);

// Check if token is blacklisted
const isBlacklisted = await this.redisService.get(`blacklist:${tokenId}`);
if (isBlacklisted) {
  throw new AppException(AuthError.TOKEN_INVALID_OR_EXPIRED, HttpStatus.UNAUTHORIZED);
}
```

## Caching Entities

```ts
@Injectable()
export class CompanyService {
  constructor(
    private readonly companyRepository: CompanyRepository,
    private readonly redisService: RedisService,
  ) {}

  async findByKey(key: string): Promise<CompanyResponseDto> {
    const cacheKey = `company:key:${key}`;
    const cached = await this.redisService.get<CompanyResponseDto>(cacheKey);
    if (cached) return cached;

    const company = await this.companyRepository.findByKey(key);
    if (!company) throw new AppException(CompanyError.NOT_FOUND, HttpStatus.NOT_FOUND);

    const dto = CompanyMapper.toDto(company);
    await this.redisService.set(cacheKey, dto, 600); // 10 minutes
    return dto;
  }
}
```

## Cache Invalidation

- **Always invalidate** when data changes:

```ts
async update(id: number, dto: CompanyUpdateDto): Promise<CompanyResponseDto> {
  const company = await this.companyRepository.findOneBy({ id });
  if (!company) throw new AppException(CompanyError.NOT_FOUND, HttpStatus.NOT_FOUND);

  Object.assign(company, dto);
  const updated = await this.companyRepository.save(company);

  // Invalidate related caches
  await this.redisService.del(`company:key:${company.key}`);
  await this.redisService.del(`company:${id}`);

  return CompanyMapper.toDto(updated);
}

async delete(id: number): Promise<void> {
  const company = await this.companyRepository.findOneBy({ id });
  if (!company) throw new AppException(CompanyError.NOT_FOUND, HttpStatus.NOT_FOUND);

  await this.companyRepository.softDelete(id);
  await this.redisService.del(`company:key:${company.key}`);
  await this.redisService.del(`company:${id}`);
}
```

## Cache Key Patterns

```ts
// ✅ Good patterns
`otp:${email}`                    // OTP storage
`otp:limit:${email}`              // OTP rate limit
`company:key:${companyKey}`       // Entity by key
`company:${id}`                   // Entity by ID
`refresh:${userId}`               // Refresh token
`blacklist:${tokenId}`            // Token blacklist
`config:app_settings`             // App-wide config
`rate:login:${email}`             // Login rate limit

// ❌ Bad patterns
`u_${id}`                         // Cryptic
`data`                            // Too generic
`cache_1`                         // Meaningless
```

## What to Cache

| Data | TTL | Invalidation |
|------|-----|--------------|
| OTP codes | 5 min | On verify/expire |
| Company by key | 10 min | On update/delete |
| Configuration | 30 min | On config update |
| Static lookups | 1 hour | On change |
| Rate limit counters | Window duration | Auto-expire |

## What NOT to Cache

- Rapidly changing data (real-time counters, live feeds)
- Sensitive data (passwords, full tokens)
- Data that must always be fresh (payment status, inventory count)
- User-specific session data (use cookies instead)

## Graceful Shutdown

```ts
// main.ts
app.enableShutdownHooks();

// In a module service
@Injectable()
export class AppService implements OnModuleDestroy {
  constructor(@Inject("REDIS_CLIENT") private readonly redis: Redis) {}

  async onModuleDestroy() {
    await this.redis.quit();
  }
}
```

## Rules

- Use `RedisService` — not `@nestjs/cache-manager` or `CacheInterceptor`.
- Cache frequently read, rarely written data — not everything.
- Always invalidate cache on create/update/delete — stale data is worse than no cache.
- Use descriptive cache keys with consistent `prefix:identifier` naming.
- Set appropriate TTLs — shorter for volatile data, longer for static.
- Never cache sensitive data (passwords, full tokens).
- Use Redis `incr` + `expire` for rate limiting.
- Use Redis for OTP storage with short TTLs.
- Implement graceful shutdown — quit Redis connection on SIGTERM.
