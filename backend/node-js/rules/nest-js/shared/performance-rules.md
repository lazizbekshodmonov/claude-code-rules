# Performance Rules (NestJS)

> NestJS-specific performance rules covering middleware, database optimization, pagination, and async patterns.

## Async Operations

- Run independent operations in **parallel** with `Promise.all`:

```ts
// ❌ Incorrect — sequential
async getDashboard(userId: number) {
  const user = await this.userService.findOne(userId);
  const orders = await this.orderService.findByUser(userId);
  const notifications = await this.notificationService.findByUser(userId);
  return { user, orders, notifications };
}

// ✅ Correct — parallel
async getDashboard(userId: number) {
  const [user, orders, notifications] = await Promise.all([
    this.userService.findOne(userId),
    this.orderService.findByUser(userId),
    this.notificationService.findByUser(userId),
  ]);
  return { user, orders, notifications };
}
```

## Logger Middleware

- Use a logging middleware that captures request/response timing:

```ts
@Injectable()
export class LoggerMiddleware implements NestMiddleware {
  private readonly logger = new Logger(LoggerMiddleware.name);

  use(req: Request, res: Response, next: NextFunction) {
    const { method, originalUrl } = req;
    const start = Date.now();
    const ip = getClientIp(req);

    res.on("finish", () => {
      const { statusCode } = res;
      const delay = Date.now() - start;
      const message = `${method} - ${statusCode} ${originalUrl} - ${delay}ms, IP: ${ip}`;

      if (statusCode >= 500) {
        this.logger.error(message);
      } else if (statusCode >= 400) {
        this.logger.warn(message);
      } else {
        this.logger.log(message);
      }
    });

    next();
  }
}
```

- Register in `AppModule`:

```ts
export class AppModule {
  configure(consumer: MiddlewareConsumer) {
    consumer.apply(LoggerMiddleware).forRoutes("*");
  }
}
```

## Database Performance

- Select only required fields in QueryBuilder:

```ts
// ❌ Incorrect — selects all columns
const companies = await this.companyRepository.find();

// ✅ Correct — select specific fields
const qb = this.createQueryBuilder("company")
  .select(["company.id", "company.name", "company.status", "company.createdAt"])
  .leftJoin("company.logo", "logo")
  .addSelect(["logo.publicUrl"]);
```

- Avoid N+1 — use `relations` or QueryBuilder joins:

```ts
// ❌ Incorrect — N+1
const users = await this.userRepository.find();
for (const user of users) {
  user.orders = await this.orderRepository.findBy({ userId: user.id });
}

// ✅ Correct — join in QueryBuilder
const users = await this.userRepository
  .createQueryBuilder("user")
  .leftJoinAndSelect("user.orders", "order")
  .where("user.status = :status", { status: UserStatus.ACTIVE })
  .getMany();
```

## Pagination

- **All** list endpoints must be paginated via `Pagination.of()`:

```ts
async findOfPagination(query: UserRequestQueryDto): Promise<[UserEntity[], number]> {
  const { page = 1, size = 10, search, status } = query;

  const qb = this.createQueryBuilder("user")
    .orderBy("user.id", "ASC")
    .skip((page - 1) * size)
    .take(size);

  if (status) {
    qb.andWhere("user.status = :status", { status });
  }

  if (search) {
    qb.andWhere("(user.name ILIKE :search OR user.email ILIKE :search)", {
      search: `%${search.trim()}%`,
    });
  }

  return qb.getManyAndCount();
}
```

- Use `.skip()` and `.take()` — never load unbounded result sets.

## Query Logging

- Enable TypeORM slow query logging:

```ts
TypeOrmModule.forRoot({
  logging: ["error", "warn"],
  maxQueryExecutionTime: 500, // log queries slower than 500ms
});
```

## Compression

```ts
// main.ts
import compression from "compression";
app.use(compression());
```

## Payload Size

```ts
app.use(json({ limit: "1mb" }));
app.use(urlencoded({ extended: true, limit: "1mb" }));
```

## Background Processing

- Use `@nestjs/bull` or `@nestjs/bullmq` for heavy tasks:

```ts
@Processor("reports")
export class ReportsProcessor {
  @Process("generate")
  async handleGenerate(job: Job<GenerateReportDto>) {
    return this.reportsService.generate(job.data);
  }
}

@Injectable()
export class ReportsService {
  constructor(@InjectQueue("reports") private readonly reportsQueue: Queue) {}

  async requestReport(dto: GenerateReportDto): Promise<{ jobId: string }> {
    const job = await this.reportsQueue.add("generate", dto);
    return { jobId: job.id as string };
  }
}
```

- Use queues for: email sending, report generation, image processing, data imports, webhooks.

## Streaming Large Data

```ts
@Get("export")
async exportUsers(@Res() res: Response) {
  res.setHeader("Content-Type", "text/csv");
  res.setHeader("Content-Disposition", "attachment; filename=users.csv");

  const queryStream = await this.userRepository
    .createQueryBuilder("user")
    .stream();

  queryStream.pipe(new CsvTransform()).pipe(res);
}
```

## Timeouts

- Set timeouts on all external HTTP calls:

```ts
@Injectable()
export class PaymentService {
  constructor(private readonly httpService: HttpService) {}

  async charge(data: ChargeDto) {
    const { data: result } = await firstValueFrom(
      this.httpService.post("/charge", data, { timeout: 5000 }),
    );
    return result;
  }
}
```

## Mapper Performance

- Use `UserMapper.toDto` as a reference to `Array.map()` for bulk conversions:

```ts
// ✅ Efficient — pass static method reference
return Pagination.of(query, total, entities.map(UserMapper.toDto));

// ❌ Inefficient — creates closure per call
return Pagination.of(query, total, entities.map(e => UserMapper.toDto(e)));
```

## Rules

- Never block the event loop — use async I/O everywhere.
- Use `Promise.all` for independent parallel operations.
- Paginate all list endpoints — never return unbounded collections.
- Select only needed fields — avoid `SELECT *`.
- Use Logger middleware to track request timing and slow requests.
- Use queues for operations taking more than 5 seconds.
- Log slow queries — set `maxQueryExecutionTime` in TypeORM config.
- Enable compression in production.
- Set timeouts on all external HTTP calls.
- Limit request body size to prevent abuse.
