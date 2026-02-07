# Security Rules (NestJS)

> NestJS-specific security rules using JWT cookies, custom guards, role-based access, and multi-tenancy.

## JWT Cookie-Based Authentication

- Tokens are stored in **httpOnly cookies** — not Bearer headers:

```ts
// common/services/cookie.service.ts
@Injectable()
export class CookieService {
  private readonly secureCookie: boolean;

  constructor(private readonly configService: ConfigService) {
    const mode = this.configService.get<string>("security.mode", "development");
    this.secureCookie = mode === "production" || mode === "staging";
  }

  accessTokenSetCookie(response: Response, token: string, expiresIn: Date) {
    response.cookie("access_token", token, {
      httpOnly: true,
      secure: this.secureCookie,
      sameSite: "lax",
      expires: expiresIn,
    });
  }

  refreshTokenSetCookie(response: Response, token: string, expiresIn: Date) {
    response.cookie("refresh_token", token, {
      httpOnly: true,
      secure: this.secureCookie,
      sameSite: "lax",
      expires: expiresIn,
    });
  }

  clearCookie(response: Response) {
    response.clearCookie("access_token", {
      httpOnly: true,
      secure: this.secureCookie,
      sameSite: "lax",
    });
    response.clearCookie("refresh_token", {
      httpOnly: true,
      secure: this.secureCookie,
      sameSite: "lax",
    });
  }
}
```

## JwtWebGuard

- Extracts JWT from `access_token` cookie, verifies it, and sets `request.user`:

```ts
@Injectable()
export class JwtWebGuard implements CanActivate {
  constructor(
    private readonly jwtService: AuthJwtService,
    private readonly cookieService: CookieService,
  ) {}

  async canActivate(context: ExecutionContext): Promise<boolean> {
    const request = context.switchToHttp().getRequest<Request>();
    const response = context.switchToHttp().getResponse<Response>();

    const accessToken = request.cookies?.["access_token"] as string | undefined;
    if (!accessToken) {
      throw new AppException(AuthError.ACCESS_TOKEN_INVALID_OR_EXPIRED, HttpStatus.UNAUTHORIZED);
    }

    const payload = await this.jwtService.verifyAccessToken(accessToken);

    // Verify company key header matches JWT (for non-admin users)
    if (payload.companyKey) {
      const headerCompanyKey = request.headers["x-company-key"] as string | undefined;
      if (payload.companyKey !== headerCompanyKey) {
        this.cookieService.clearCookie(response);
        throw new AppException(AuthError.COMPANY_KEY_MISMATCH, HttpStatus.UNAUTHORIZED);
      }
    }

    request.user = payload;
    return true;
  }
}
```

- **Do NOT** use Passport `AuthGuard("jwt")` — use the custom `JwtWebGuard`.

## AuthJwtService

```ts
@Injectable()
export class AuthJwtService {
  private readonly jwtSecret: string;
  private readonly accessExpiresIn: string;
  private readonly refreshExpiresIn: string;

  constructor(
    private readonly jwtService: JwtService,
    private readonly configService: ConfigService,
  ) {
    this.jwtSecret = this.configService.get<string>("security.jwt.jwt_secret", "");
    this.accessExpiresIn = this.configService.get<string>("security.jwt.access_expires_in", "1d");
    this.refreshExpiresIn = this.configService.get<string>("security.jwt.refresh_expires_in", "7d");
  }

  async verifyAccessToken(token: string): Promise<JwtPayload> {
    try {
      return await this.jwtService.verifyAsync<JwtPayload>(token, {
        secret: this.jwtSecret,
      });
    } catch {
      throw new AppException(AuthError.ACCESS_TOKEN_INVALID_OR_EXPIRED, HttpStatus.UNAUTHORIZED);
    }
  }

  accessTokenGenerate(payload: JwtPayload): string {
    return this.jwtService.sign(payload, {
      secret: this.jwtSecret,
      expiresIn: this.accessExpiresIn,
    });
  }

  refreshTokenGenerate(payload: JwtPayload): string {
    return this.jwtService.sign(payload, {
      secret: this.jwtSecret,
      expiresIn: this.refreshExpiresIn,
    });
  }
}
```

## Role-Based Access Control (RBAC)

- Use `@Roles()` decorator + `RolesGuard`:

```ts
// decorators/roles.decorator.ts
export const ROLES_KEY = "roles";

type AllowedRole = UserRole | EmployeeRole;

export const Roles = (...roles: AllowedRole[]) => {
  const rolesText = `\n\n**Allowed roles:** ${roles.join(", ")}`;

  return applyDecorators(
    SetMetadata(ROLES_KEY, roles),
    // Auto-append roles to Swagger description
    (target, propertyKey, descriptor) => {
      const existing = Reflect.getMetadata("swagger/apiOperation", descriptor.value) || {};
      ApiOperation({
        ...existing,
        description: `${existing.description || ""}${rolesText}`,
      })(target, propertyKey, descriptor);
    },
  );
};
```

```ts
// guards/role.guard.ts
@Injectable()
export class RolesGuard implements CanActivate {
  constructor(private readonly reflector: Reflector) {}

  canActivate(context: ExecutionContext): boolean {
    const requiredRoles = this.reflector.getAllAndOverride<string[]>(ROLES_KEY, [
      context.getHandler(),
      context.getClass(),
    ]);
    if (!requiredRoles || requiredRoles.length === 0) return true;

    const { user } = context.switchToHttp().getRequest<{ user: JwtPayload }>();
    if (!user) throw new AppException(AuthError.PERMISSION_DENIED, HttpStatus.FORBIDDEN);

    const hasUserRole = requiredRoles.includes(user.role);
    const hasEmployeeRole = user.employeeRole && requiredRoles.includes(user.employeeRole);

    if (!hasUserRole && !hasEmployeeRole) {
      throw new AppException(AuthError.PERMISSION_DENIED, HttpStatus.FORBIDDEN);
    }

    return true;
  }
}
```

## CompanyKeyGuard (Multi-Tenancy)

- Validates `X-Company-Key` header and sets `request.companyId`:

```ts
@Injectable()
export class CompanyKeyGuard implements CanActivate {
  constructor(private readonly companyRepository: CompanyRepository) {}

  async canActivate(context: ExecutionContext): Promise<boolean> {
    const request = context.switchToHttp().getRequest<Request & { user: JwtPayload; companyId: number }>();
    const user = request.user;
    const headerCompanyKey = request.headers["x-company-key"] as string | undefined;

    if (!headerCompanyKey) {
      throw new AppException(AuthError.COMPANY_KEY_REQUIRED, HttpStatus.UNAUTHORIZED);
    }

    // Admin can use any company key
    if (user.role === UserRole.ADMIN) {
      const company = await this.companyRepository.findByKey(headerCompanyKey);
      if (!company) throw new AppException(AuthError.COMPANY_NOT_FOUND, HttpStatus.NOT_FOUND);
      request.companyId = company.id;
      return true;
    }

    // Non-admin: company key must match JWT
    if (user.companyKey !== headerCompanyKey) {
      throw new AppException(AuthError.COMPANY_KEY_MISMATCH, HttpStatus.UNAUTHORIZED);
    }

    request.companyId = user.companyId!;
    return true;
  }
}
```

## Custom Parameter Decorators

```ts
// @AuthenticatedUser() — extracts JWT payload from request
export const AuthenticatedUser = createParamDecorator(
  (_: unknown, ctx: ExecutionContext): JwtPayload => {
    const request = ctx.switchToHttp().getRequest<{ user: JwtPayload }>();
    if (!request.user) {
      throw new AppException(AuthError.ACCESS_TOKEN_INVALID_OR_EXPIRED, HttpStatus.UNAUTHORIZED);
    }
    return request.user;
  },
);

// @CompanyId() — extracts company ID set by CompanyKeyGuard
export const CompanyId = createParamDecorator(
  (_: unknown, ctx: ExecutionContext): number => {
    const request = ctx.switchToHttp().getRequest<{ companyId: number }>();
    if (!request.companyId) {
      throw new AppException(AuthError.COMPANY_KEY_REQUIRED, HttpStatus.UNAUTHORIZED);
    }
    return request.companyId;
  },
);
```

## Controller Usage

```ts
@ApiTags("Users")
@ApiBearerAuth()
@UseGuards(JwtWebGuard, RolesGuard)
@Controller("user")
export class UserController {
  @Get("profile")
  @HttpCode(HttpStatus.OK)
  profile(@AuthenticatedUser() jwtPayload: JwtPayload) {
    return this.userService.getProfile(jwtPayload);
  }

  @Post()
  @HttpCode(HttpStatus.CREATED)
  @Roles(UserRole.ADMIN)
  create(@Body() dto: UserCreateDto) {
    return this.userService.createUser(dto);
  }

  @Delete(":id")
  @HttpCode(HttpStatus.OK)
  @Roles(UserRole.ADMIN)
  delete(@Param("id", ParseIntPipe) id: number) {
    return this.userService.deleteUser(id);
  }
}

// Multi-tenant controller
@UseGuards(JwtWebGuard, CompanyKeyGuard, RolesGuard)
@Controller("employee")
export class EmployeeController {
  @Get()
  @Roles(UserRole.ADMIN, EmployeeRole.HR)
  findAll(@CompanyId() companyId: number, @Query() query: EmployeeQueryDto) {
    return this.employeeService.findAll(companyId, query);
  }
}
```

## Login / Logout Flow

```ts
@Post("login-web")
@HttpCode(HttpStatus.OK)
async login(
  @Body() dto: AuthLoginDto,
  @Headers("x-company-key") companyKey: string | undefined,
  @Res({ passthrough: true }) response: Response,
) {
  const tokens = await this.authService.login(dto, companyKey || null);
  this.cookieService.accessTokenSetCookie(response, tokens.accessToken, tokens.accessExpiresAt);
  this.cookieService.refreshTokenSetCookie(response, tokens.refreshToken, tokens.refreshExpiresAt);
  return { message: "Logged in successfully!" };
}

@Post("logout")
@HttpCode(HttpStatus.OK)
logout(@Res({ passthrough: true }) res: Response) {
  this.cookieService.clearCookie(res);
  return { message: "Successfully logged out" };
}
```

## Input Validation

```ts
export class UserCreateDto {
  @IsEmail({}, { message: "Invalid email format" })
  email: string;

  @IsString()
  @MinLength(2)
  @MaxLength(150)
  name: string;

  @IsString()
  @MinLength(8)
  password: string;
}
```

- Use `ParseIntPipe` for numeric path params:

```ts
@Get(":id")
findOne(@Param("id", ParseIntPipe) id: number) { ... }
```

## Helmet & CORS

```ts
// main.ts
app.use(helmet());

app.enableCors({
  origin: configService.get<string[]>("cors.allowed-origins"),
  credentials: true,
  methods: ["GET", "POST", "PUT", "PATCH", "DELETE"],
});
```

## Password Hashing

```ts
// Use bcrypt with cost factor 12+
const hash = await bcrypt.hash(dto.password, 12);

async validateUser(email: string, password: string): Promise<UserEntity> {
  const user = await this.userRepository.findByUsername(email);
  if (!user || !(await bcrypt.compare(password, user.passwordHash))) {
    throw new AppException(AuthError.INVALID_CREDENTIALS, HttpStatus.UNAUTHORIZED);
  }
  return user;
}
```

## Rules

- Use `JwtWebGuard` — not Passport `AuthGuard("jwt")`.
- JWT tokens in httpOnly cookies — never expose tokens to JavaScript.
- Use `@Roles()` + `RolesGuard` for authorization — never check roles in service logic.
- Use `@AuthenticatedUser()` — never access `req.user` directly.
- Use `CompanyKeyGuard` + `@CompanyId()` for multi-tenant routes.
- Always verify resource ownership — users must not access others' data.
- Apply `ValidationPipe` globally — validate all inputs automatically.
- Use helmet and CORS — configure strict origins from `application.yml`.
- Never log or return passwords — hash with bcrypt (cost 12+).
- Guards throw `AppException` — never NestJS built-in exceptions.
