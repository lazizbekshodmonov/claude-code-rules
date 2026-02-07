# Configuration Rules

> Backend configuration rules for Claude Code. Apply to all Node.js backend projects.

## Environment Variables

- All configuration must come from **environment variables** — never hardcode values:

```ts
// ❌ Incorrect — hardcoded
const PORT = 3000;
const DB_HOST = "localhost";
const API_KEY = "sk-1234567890";

// ✅ Correct — from environment
const PORT = process.env.PORT;
const DB_HOST = process.env.DB_HOST;
const API_KEY = process.env.API_KEY;
```

## Config Validation

- **Validate all environment variables** at application startup — fail fast if required values are missing:

```ts
// ✅ Correct — validate with Zod
import { z } from "zod";

const envSchema = z.object({
  NODE_ENV: z.enum(["development", "production", "test"]),
  PORT: z.coerce.number().default(3000),
  DATABASE_URL: z.string().url(),
  JWT_SECRET: z.string().min(32),
  JWT_EXPIRES_IN: z.string().default("1h"),
  REDIS_URL: z.string().url().optional(),
});

export const env = envSchema.parse(process.env);
```

```ts
// ✅ Correct — NestJS ConfigModule with Joi
@Module({
  imports: [
    ConfigModule.forRoot({
      validationSchema: Joi.object({
        NODE_ENV: Joi.string().valid("development", "production", "test").required(),
        PORT: Joi.number().default(3000),
        DATABASE_URL: Joi.string().required(),
        JWT_SECRET: Joi.string().min(32).required(),
      }),
    }),
  ],
})
export class AppModule {}
```

- If a required variable is missing, the app should **crash immediately** with a clear error message — do not start in a broken state.

## Config Module

- Centralize all configuration in a dedicated module — never read `process.env` directly throughout the codebase:

```ts
// ❌ Incorrect — scattered env reads
class UserService {
  async sendEmail() {
    const apiKey = process.env.SENDGRID_API_KEY;
  }
}

// ✅ Correct — centralized config
// config/app.config.ts
export const appConfig = {
  port: env.PORT,
  nodeEnv: env.NODE_ENV,
  isDev: env.NODE_ENV === "development",
  isProd: env.NODE_ENV === "production",
};

// config/database.config.ts
export const databaseConfig = {
  url: env.DATABASE_URL,
  maxConnections: env.DB_MAX_CONNECTIONS,
};

// config/auth.config.ts
export const authConfig = {
  jwtSecret: env.JWT_SECRET,
  jwtExpiresIn: env.JWT_EXPIRES_IN,
  refreshTokenExpiresIn: env.REFRESH_TOKEN_EXPIRES_IN,
};
```

## Config File Structure

```
src/
├── config/
│   ├── index.ts              ← Re-exports all configs
│   ├── env.validation.ts     ← Schema validation
│   ├── app.config.ts         ← Server, general settings
│   ├── database.config.ts    ← Database connection
│   ├── auth.config.ts        ← JWT, session config
│   ├── cache.config.ts       ← Redis, caching config
│   └── external.config.ts    ← Third-party services
```

## Environment Files

- Maintain environment files for different environments:

```
.env                ← Local development (DO NOT commit)
.env.example        ← Template with placeholder values (commit this)
.env.test           ← Test environment (commit if no secrets)
```

- `.env.example` must include every required variable with descriptive comments:

```bash
# .env.example

# Server
PORT=3000
NODE_ENV=development

# Database
DATABASE_URL=postgresql://user:password@localhost:5432/myapp

# Auth
JWT_SECRET=your-secret-key-min-256-bits
JWT_EXPIRES_IN=1h
REFRESH_TOKEN_EXPIRES_IN=30d

# Redis (optional)
REDIS_URL=redis://localhost:6379

# External Services
SENDGRID_API_KEY=your-sendgrid-key
S3_BUCKET=your-bucket-name
S3_REGION=us-east-1
```

## Secrets Management

- In production, use a **secret manager** — do not rely on `.env` files:
  - AWS Secrets Manager
  - HashiCorp Vault
  - Google Secret Manager
  - Azure Key Vault

- Rotate secrets regularly — especially database passwords and API keys.
- Use different secrets for each environment — never share between dev/staging/production.

## Feature Flags

- Use environment variables or a feature flag service for toggling features:

```ts
// ✅ Correct — simple feature flag
const features = {
  enableNewPaymentFlow: env.FEATURE_NEW_PAYMENT === "true",
  enableEmailNotifications: env.FEATURE_EMAIL_NOTIFICATIONS === "true",
};

if (features.enableNewPaymentFlow) {
  await processPaymentV2(order);
} else {
  await processPayment(order);
}
```

## Rules

- Never commit `.env` files with real secrets — only commit `.env.example`.
- Validate all config at startup — fail fast, not at runtime when a config is first accessed.
- Use typed config objects — never pass raw `process.env` values around the codebase.
- Different environments must have different configurations — never share secrets across environments.
- Document every environment variable in `.env.example` with comments.
- Review config changes carefully in PRs — a wrong value can break production.
