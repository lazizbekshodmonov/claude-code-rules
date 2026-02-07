# Security Rules (NestJS)

> NestJS-specific security rules using guards, pipes, decorators, and middleware.

## Authentication Guard

- Implement JWT authentication with Passport and guards:

```ts
// auth/jwt.strategy.ts
@Injectable()
export class JwtStrategy extends PassportStrategy(Strategy) {
  constructor(private readonly configService: ConfigService) {
    super({
      jwtFromRequest: ExtractJwt.fromAuthHeaderAsBearerToken(),
      secretOrKey: configService.get("JWT_SECRET"),
    });
  }

  async validate(payload: JwtPayload): Promise<UserPayload> {
    return { id: payload.sub, email: payload.email, role: payload.role };
  }
}

// auth/jwt-auth.guard.ts
@Injectable()
export class JwtAuthGuard extends AuthGuard("jwt") {}
```

- Apply globally or per-route:

```ts
// Global (main.ts or AppModule)
app.useGlobalGuards(new JwtAuthGuard());

// Per-route
@UseGuards(JwtAuthGuard)
@Get("profile")
async getProfile(@CurrentUser() user: UserPayload) { ... }
```

## Role-Based Access Control (RBAC)

- Create a `Roles` decorator and `RolesGuard`:

```ts
// decorators/roles.decorator.ts
export const Roles = (...roles: UserRole[]) => SetMetadata("roles", roles);

// guards/roles.guard.ts
@Injectable()
export class RolesGuard implements CanActivate {
  constructor(private readonly reflector: Reflector) {}

  canActivate(context: ExecutionContext): boolean {
    const requiredRoles = this.reflector.getAllAndOverride<UserRole[]>("roles", [
      context.getHandler(),
      context.getClass(),
    ]);
    if (!requiredRoles) return true;

    const { user } = context.switchToHttp().getRequest();
    return requiredRoles.includes(user.role);
  }
}
```

- Usage:

```ts
@UseGuards(JwtAuthGuard, RolesGuard)
@Roles(UserRole.ADMIN)
@Delete(":id")
async deleteUser(@Param("id", ParseUUIDPipe) id: string) { ... }
```

## Public Routes

- Use a `@Public()` decorator to skip auth on specific routes:

```ts
export const IS_PUBLIC_KEY = "isPublic";
export const Public = () => SetMetadata(IS_PUBLIC_KEY, true);

// Update JwtAuthGuard
@Injectable()
export class JwtAuthGuard extends AuthGuard("jwt") {
  constructor(private readonly reflector: Reflector) { super(); }

  canActivate(context: ExecutionContext) {
    const isPublic = this.reflector.getAllAndOverride<boolean>(IS_PUBLIC_KEY, [
      context.getHandler(),
      context.getClass(),
    ]);
    if (isPublic) return true;
    return super.canActivate(context);
  }
}

// Usage
@Public()
@Post("auth/login")
async login(@Body() dto: LoginDto) { ... }
```

## Input Validation

- Use `ValidationPipe` + class-validator for all incoming data:

```ts
export class CreateUserDto {
  @IsEmail()
  email: string;

  @IsString()
  @MinLength(8)
  @Matches(/^(?=.*[A-Z])(?=.*\d)/, { message: "Password too weak" })
  password: string;

  @IsString()
  @MinLength(2)
  @MaxLength(100)
  name: string;
}
```

- Validate params and queries with pipes:

```ts
@Get(":id")
async findById(@Param("id", ParseUUIDPipe) id: string) { ... }
```

## Resource Ownership

- Always verify the user owns the resource they're accessing:

```ts
async findOrderForUser(userId: string, orderId: string): Promise<Order> {
  const order = await this.orderRepository.findOneBy({ id: orderId });
  if (!order) throw new NotFoundException(`Order ${orderId} not found`);
  if (order.userId !== userId) throw new ForbiddenException("Access denied");
  return order;
}
```

## Rate Limiting

- Use `@nestjs/throttler` for rate limiting:

```ts
// app.module.ts
@Module({
  imports: [
    ThrottlerModule.forRoot([{
      ttl: 60000,  // 1 minute window
      limit: 100,  // max 100 requests
    }]),
  ],
  providers: [{ provide: APP_GUARD, useClass: ThrottlerGuard }],
})
export class AppModule {}

// Stricter limits on auth endpoints
@Throttle({ default: { ttl: 60000, limit: 5 } })
@Post("auth/login")
async login(@Body() dto: LoginDto) { ... }

// Skip throttling for specific routes
@SkipThrottle()
@Get("health")
async healthCheck() { ... }
```

## Helmet

- Enable helmet in `main.ts`:

```ts
import helmet from "helmet";
app.use(helmet());
```

## CORS

```ts
// main.ts
app.enableCors({
  origin: ["https://app.example.com", "https://admin.example.com"],
  credentials: true,
  methods: ["GET", "POST", "PUT", "PATCH", "DELETE"],
});
```

## Password Hashing

- Hash in the service layer, never store plain text:

```ts
@Injectable()
export class AuthService {
  async register(dto: RegisterDto): Promise<User> {
    const hash = await bcrypt.hash(dto.password, 12);
    return this.userRepository.save({ ...dto, password: hash });
  }

  async validateUser(email: string, password: string): Promise<User> {
    const user = await this.userRepository.findOneBy({ email });
    if (!user || !(await bcrypt.compare(password, user.password))) {
      throw new UnauthorizedException("Invalid credentials");
    }
    return user;
  }
}
```

## Cookie-Based Tokens

```ts
res.cookie("refreshToken", token, {
  httpOnly: true,
  secure: true,
  sameSite: "strict",
  maxAge: 7 * 24 * 60 * 60 * 1000,
  path: "/api/auth/refresh",
});
```

## Rules

- Use guards for auth — never check tokens manually in controllers.
- Apply `ValidationPipe` globally — validate all inputs automatically.
- Use `@Roles()` + `RolesGuard` for authorization — never check roles in service logic.
- Always verify resource ownership — users must not access others' data.
- Enable rate limiting on auth endpoints (login, register, password-reset).
- Use helmet and CORS in production — configure strict origins.
- Never log or return passwords — hash with bcrypt (cost 12+).
- Run `npm audit` regularly.
