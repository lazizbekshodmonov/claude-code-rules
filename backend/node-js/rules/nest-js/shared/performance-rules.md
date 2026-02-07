# Performance Rules (NestJS)

> NestJS-specific performance rules covering interceptors, caching, database, and async patterns.

## Async Operations

- Run independent operations in **parallel** with `Promise.all`:

```ts
// ❌ Incorrect — sequential
async getDashboard(userId: string) {
  const user = await this.usersService.findById(userId);
  const orders = await this.ordersService.findByUser(userId);
  const notifications = await this.notificationsService.findByUser(userId);
  return { user, orders, notifications };
}

// ✅ Correct — parallel
async getDashboard(userId: string) {
  const [user, orders, notifications] = await Promise.all([
    this.usersService.findById(userId),
    this.ordersService.findByUser(userId),
    this.notificationsService.findByUser(userId),
  ]);
  return { user, orders, notifications };
}
```

## Response Interceptor

- Use interceptors for response transformation and timing:

```ts
@Injectable()
export class ResponseTimeInterceptor implements NestInterceptor {
  private readonly logger = new Logger(ResponseTimeInterceptor.name);

  intercept(context: ExecutionContext, next: CallHandler): Observable<any> {
    const start = Date.now();
    return next.handle().pipe(
      tap(() => {
        const duration = Date.now() - start;
        const request = context.switchToHttp().getRequest();
        if (duration > 500) {
          this.logger.warn("Slow request", {
            method: request.method,
            path: request.url,
            duration,
          });
        }
      }),
    );
  }
}
```

## Database Performance

- Select only required fields:

```ts
// ❌ Incorrect
const users = await this.userRepository.find();

// ✅ Correct
const users = await this.userRepository.find({
  select: { id: true, name: true, email: true },
});
```

- Avoid N+1 — use `relations` or QueryBuilder:

```ts
// ✅ Relations
const users = await this.userRepository.find({
  relations: { orders: true },
  where: { isActive: true },
});

// ✅ QueryBuilder for complex joins
const users = await this.userRepository
  .createQueryBuilder("user")
  .leftJoinAndSelect("user.orders", "order", "order.status = :status", { status: "active" })
  .where("user.isActive = :isActive", { isActive: true })
  .getMany();
```

- Paginate all list endpoints:

```ts
async findAll(query: PaginationDto): Promise<[User[], number]> {
  return this.userRepository.findAndCount({
    skip: (query.page - 1) * query.limit,
    take: query.limit,
    order: { createdAt: "DESC" },
  });
}
```

## Query Logging

- Enable TypeORM slow query logging:

```ts
TypeOrmModule.forRoot({
  logging: ["error", "warn", "query"],
  maxQueryExecutionTime: 500, // log queries slower than 500ms
});
```

## Compression

- Enable response compression:

```ts
// main.ts
import compression from "compression";
app.use(compression());
```

## Payload Size

- Limit request body size:

```ts
app.use(json({ limit: "1mb" }));
app.use(urlencoded({ extended: true, limit: "1mb" }));
```

## Background Processing

- Use `@nestjs/bull` or `@nestjs/bullmq` for heavy tasks:

```ts
// queue processor
@Processor("reports")
export class ReportsProcessor {
  @Process("generate")
  async handleGenerate(job: Job<GenerateReportDto>) {
    const report = await this.reportsService.generate(job.data);
    return report;
  }
}

// service — add job to queue
@Injectable()
export class ReportsService {
  constructor(@InjectQueue("reports") private readonly reportsQueue: Queue) {}

  async requestReport(dto: GenerateReportDto): Promise<{ jobId: string }> {
    const job = await this.reportsQueue.add("generate", dto);
    return { jobId: job.id as string };
  }
}

// controller
@Post("reports")
async generateReport(@Body() dto: GenerateReportDto) {
  return this.reportsService.requestReport(dto);
}
```

- Use queues for: email sending, report generation, image processing, data imports, webhooks.

## Streaming Large Data

- Use streams for large file processing or data export:

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

- Set timeouts on external HTTP calls:

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

## Rules

- Never block the event loop — use async I/O everywhere.
- Use `Promise.all` for independent parallel operations.
- Paginate all list endpoints — never return unbounded collections.
- Select only needed fields — avoid `SELECT *`.
- Use queues for operations taking more than 5 seconds.
- Log slow queries — set `maxQueryExecutionTime` in TypeORM config.
- Enable compression in production.
- Set timeouts on all external HTTP calls.
- Monitor response times and memory usage — alert on anomalies.
