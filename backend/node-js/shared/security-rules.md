# Security Rules

> Backend security rules for Claude Code. Apply to all Node.js backend projects.

## Secrets and Credentials

- **NEVER** hardcode API keys, passwords, tokens, or secrets in source code:

```ts
// ❌ Incorrect
const JWT_SECRET = "my-super-secret-key";
const DB_PASSWORD = "password123";

// ✅ Correct
const JWT_SECRET = process.env.JWT_SECRET;
const DB_PASSWORD = process.env.DATABASE_PASSWORD;
```

- Ensure `.env` files are in `.gitignore` — never commit them.
- Use `.env.example` with placeholder values for documentation.
- In production, use secret managers (AWS Secrets Manager, Vault, etc.) instead of `.env` files.

## Input Validation

- **Validate all incoming data** at the controller/route level before processing:

```ts
// ✅ Correct — DTO validation with class-validator (NestJS)
export class CreateUserDto {
  @IsEmail()
  email: string;

  @IsString()
  @MinLength(2)
  @MaxLength(100)
  name: string;

  @IsInt()
  @Min(0)
  @Max(150)
  age: number;
}

// ✅ Correct — Zod validation (Express)
const createUserSchema = z.object({
  email: z.string().email(),
  name: z.string().min(2).max(100),
  age: z.number().int().min(0).max(150),
});
```

- Validate path params, query params, headers, and body — not just body.
- Reject unknown fields — do not silently ignore extra properties.

## SQL Injection Prevention

- Always use parameterized queries or ORM methods:

```ts
// ❌ Incorrect — SQL injection risk
const query = `SELECT * FROM users WHERE email = '${email}'`;

// ✅ Correct
const user = await prisma.user.findUnique({ where: { email } });
```

## Authentication

- Use **JWT** or **session-based** auth — never roll your own auth protocol.
- Store JWT secrets in environment variables — minimum 256 bits.
- Set reasonable token expiration: access token (15min–1h), refresh token (7–30 days).
- Use `httpOnly`, `secure`, `sameSite` cookies for token storage when possible:

```ts
res.cookie("refreshToken", token, {
  httpOnly: true,
  secure: true,
  sameSite: "strict",
  maxAge: 7 * 24 * 60 * 60 * 1000,
});
```

## Authorization

- Check permissions on **every** protected endpoint — never rely on client-side checks alone.
- Implement role-based (RBAC) or attribute-based (ABAC) access control:

```ts
// ❌ Incorrect — no authorization check
@Get("admin/dashboard")
async getDashboard() {
  return this.dashboardService.getData();
}

// ✅ Correct — guard/middleware checks role
@UseGuards(AuthGuard, RolesGuard)
@Roles("admin")
@Get("admin/dashboard")
async getDashboard() {
  return this.dashboardService.getData();
}
```

- Always verify resource ownership — a user should not access another user's data:

```ts
// ✅ Correct — verify ownership
async getOrder(userId: string, orderId: string) {
  const order = await prisma.order.findUnique({ where: { id: orderId } });
  if (order.userId !== userId) throw new ForbiddenError();
  return order;
}
```

## Rate Limiting

- Apply rate limiting to all public endpoints, especially auth routes:

```ts
// Express
import rateLimit from "express-rate-limit";

const authLimiter = rateLimit({
  windowMs: 15 * 60 * 1000,
  max: 10,
  message: { message: "Too many attempts, try again later" },
});

app.use("/api/auth", authLimiter);
```

- Stricter limits for login/register/password-reset endpoints.

## Helmet and Security Headers

- Use **helmet** to set security headers:

```ts
import helmet from "helmet";
app.use(helmet());
```

- This sets `Content-Security-Policy`, `X-Frame-Options`, `X-Content-Type-Options`, and more.

## CORS

- Configure CORS to allow only trusted origins:

```ts
// ❌ Incorrect — allows everything
app.use(cors({ origin: "*" }));

// ✅ Correct — whitelist specific origins
app.use(cors({
  origin: ["https://app.example.com", "https://admin.example.com"],
  credentials: true,
}));
```

## Password Handling

- Hash passwords with **bcrypt** (cost factor 10+) or **argon2** — never store plain text:

```ts
import bcrypt from "bcrypt";

const hash = await bcrypt.hash(password, 12);
const isMatch = await bcrypt.compare(password, hash);
```

- Never log or return passwords in API responses.

## File Uploads

- Validate file type, size, and content:

```ts
const upload = multer({
  limits: { fileSize: 5 * 1024 * 1024 }, // 5MB max
  fileFilter: (req, file, cb) => {
    const allowed = ["image/jpeg", "image/png", "image/webp"];
    if (!allowed.includes(file.mimetype)) {
      return cb(new ValidationError([{ field: "file", message: "Invalid file type" }]));
    }
    cb(null, true);
  },
});
```

- Generate random filenames — never use the original filename directly.
- Store uploads outside the application directory.

## Rules

- Security is not optional — every feature must consider security implications.
- Apply the principle of least privilege everywhere.
- Never disable security features "temporarily" — they tend to stay disabled.
- Run `npm audit` regularly and fix critical vulnerabilities.
- Log security events (failed logins, permission denials, suspicious requests).
- Never trust client-side data — always validate on the server.
