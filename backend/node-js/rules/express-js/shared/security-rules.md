# Security Rules (Express)

> Express-specific security rules using middleware, passport, and manual guards.

## JWT Authentication Middleware

```ts
// src/middleware/auth.ts
import jwt from "jsonwebtoken";

export const authenticate = (req: Request, res: Response, next: NextFunction) => {
  const token = req.headers.authorization?.replace("Bearer ", "");
  if (!token) throw new UnauthorizedError("Missing auth token");

  try {
    const payload = jwt.verify(token, process.env.JWT_SECRET!) as JwtPayload;
    req.user = { id: payload.sub, email: payload.email, role: payload.role };
    next();
  } catch {
    throw new UnauthorizedError("Invalid or expired token");
  }
};

// Usage
router.get("/profile", authenticate, asyncHandler(async (req, res) => {
  const user = await userService.findById(req.user.id);
  res.json({ data: user });
}));
```

## Role-Based Authorization Middleware

```ts
// src/middleware/authorize.ts
export const authorize = (...roles: string[]) =>
  (req: Request, res: Response, next: NextFunction) => {
    if (!req.user) throw new UnauthorizedError();
    if (!roles.includes(req.user.role)) throw new ForbiddenError();
    next();
  };

// Usage
router.delete("/users/:id",
  authenticate,
  authorize("admin"),
  asyncHandler(async (req, res) => {
    await userService.delete(req.params.id);
    res.status(204).send();
  }),
);
```

## Input Validation

- Use Zod for schema validation:

```ts
const createUserSchema = z.object({
  email: z.string().email(),
  name: z.string().min(2).max(100),
  password: z.string().min(8).regex(/^(?=.*[A-Z])(?=.*\d)/, "Password too weak"),
});

router.post("/users",
  validate({ body: createUserSchema }),
  asyncHandler(async (req, res) => {
    const user = await userService.create(req.body);
    res.status(201).json({ data: user });
  }),
);
```

## Resource Ownership

```ts
export const ownsResource = (getResourceUserId: (req: Request) => Promise<string>) =>
  asyncHandler(async (req: Request, res: Response, next: NextFunction) => {
    const resourceUserId = await getResourceUserId(req);
    if (resourceUserId !== req.user.id && req.user.role !== "admin") {
      throw new ForbiddenError("Access denied");
    }
    next();
  });

// Usage
router.get("/orders/:id",
  authenticate,
  ownsResource(async (req) => {
    const order = await orderService.findById(req.params.id);
    return order.userId;
  }),
  asyncHandler(async (req, res) => {
    const order = await orderService.findById(req.params.id);
    res.json({ data: order });
  }),
);
```

## Rate Limiting

```ts
import rateLimit from "express-rate-limit";

// General rate limit
const generalLimiter = rateLimit({
  windowMs: 15 * 60 * 1000,
  max: 100,
  message: { message: "Too many requests, try again later" },
});

// Strict auth rate limit
const authLimiter = rateLimit({
  windowMs: 15 * 60 * 1000,
  max: 5,
  message: { message: "Too many login attempts, try again later" },
});

app.use("/api", generalLimiter);
app.use("/api/auth/login", authLimiter);
app.use("/api/auth/register", authLimiter);
```

## Helmet

```ts
import helmet from "helmet";
app.use(helmet());
```

## CORS

```ts
import cors from "cors";

app.use(cors({
  origin: ["https://app.example.com", "https://admin.example.com"],
  credentials: true,
  methods: ["GET", "POST", "PUT", "PATCH", "DELETE"],
}));
```

## Password Hashing

```ts
import bcrypt from "bcrypt";

export const authService = {
  async register(data: RegisterDto): Promise<User> {
    const hash = await bcrypt.hash(data.password, 12);
    return userService.create({ ...data, password: hash });
  },

  async login(email: string, password: string): Promise<{ token: string }> {
    const user = await userRepository.findOneBy({ email });
    if (!user || !(await bcrypt.compare(password, user.password))) {
      throw new UnauthorizedError("Invalid credentials");
    }
    const token = jwt.sign(
      { sub: user.id, email: user.email, role: user.role },
      process.env.JWT_SECRET!,
      { expiresIn: "1h" },
    );
    return { token };
  },
};
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

## File Upload Security

```ts
import multer from "multer";

const upload = multer({
  limits: { fileSize: 5 * 1024 * 1024 },
  fileFilter: (req, file, cb) => {
    const allowed = ["image/jpeg", "image/png", "image/webp"];
    if (!allowed.includes(file.mimetype)) {
      return cb(new ValidationError([{ field: "file", message: "Invalid file type" }]));
    }
    cb(null, true);
  },
});

router.post("/upload", authenticate, upload.single("file"), asyncHandler(async (req, res) => {
  const url = await storageService.upload(req.file);
  res.json({ data: { url } });
}));
```

## Rules

- Use `authenticate` middleware on all protected routes.
- Use `authorize` middleware for role-based access — never check roles in services.
- Validate all inputs with Zod — reject bad data at the edge.
- Always verify resource ownership — users must not access others' data.
- Use rate limiting on auth endpoints and public APIs.
- Enable helmet and CORS with strict origin whitelist.
- Hash passwords with bcrypt (cost 12+) — never store plain text.
- Run `npm audit` regularly.
