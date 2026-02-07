# Security Rules

> Universal security rules for Claude Code. Apply to all projects regardless of tech stack.

## Secrets and Credentials

- **NEVER** hardcode API keys, passwords, tokens, or secrets in source code.
- Store all secrets in environment variables (`.env` files) and access them through config:

```ts
// ❌ Incorrect — hardcoded secret
const API_KEY = "sk-1234567890abcdef";
const DB_PASSWORD = "mysecretpassword";

// ✅ Correct — environment variable
const API_KEY = process.env.VITE_API_KEY;
const DB_PASSWORD = process.env.DATABASE_PASSWORD;
```

- Ensure `.env` files are in `.gitignore` — never commit them.
- Use `.env.example` with placeholder values for documentation:

```
# .env.example
VITE_API_KEY=your_api_key_here
DATABASE_URL=postgresql://user:password@localhost:5432/dbname
```

## Input Validation

- **Never trust user input.** Validate and sanitize all inputs on both client and server side.
- Use schema validation libraries (Zod, Yup, Joi) for structured validation:

```ts
// ✅ Correct — schema validation
import { z } from "zod";

const UserSchema = z.object({
  email: z.string().email(),
  age: z.number().min(0).max(150),
  name: z.string().min(1).max(100),
});
```

- Prevent SQL injection — always use parameterized queries or ORM methods, never string concatenation:

```ts
// ❌ Incorrect — SQL injection risk
const query = `SELECT * FROM users WHERE id = ${userId}`;

// ✅ Correct — parameterized query
const user = await prisma.user.findUnique({ where: { id: userId } });
```

## Authentication and Authorization

- Store tokens securely — use `httpOnly` cookies instead of `localStorage` for auth tokens when possible.
- Implement token refresh logic — never expose long-lived tokens.
- Check permissions on every protected route and API endpoint.
- Use the principle of least privilege — grant minimum required access.

## XSS Prevention

- Never use `v-html` (Vue) or `dangerouslySetInnerHTML` (React) with user-provided content.
- Sanitize HTML content before rendering if absolutely necessary:

```ts
// ❌ Dangerous
<div v-html="userComment" />

// ✅ Safer — sanitize first
import DOMPurify from "dompurify";
const safeHTML = DOMPurify.sanitize(userComment);
```

## Dependency Security

- Regularly audit dependencies: `npm audit`
- Do not install packages with known critical vulnerabilities.
- Pin dependency versions in `package-lock.json` — always commit lock files.
- Prefer well-maintained packages with active communities.

## Error Handling

- Never expose internal error details (stack traces, database errors) to the client:

```ts
// ❌ Incorrect — leaks internal info
res.status(500).json({ error: error.stack });

// ✅ Correct — generic message
res.status(500).json({ error: "An unexpected error occurred" });
```

- Log detailed errors server-side for debugging.
- Use global error handlers to catch unhandled exceptions.

## CORS and Headers

- Configure CORS to allow only trusted origins — never use `*` in production.
- Set security headers: `Content-Security-Policy`, `X-Frame-Options`, `X-Content-Type-Options`.

## File Uploads

- Validate file type, size, and content before accepting uploads.
- Never store uploaded files in publicly accessible directories without validation.
- Generate random filenames — never use the original filename directly.

## Rules

- Security is not optional — every feature must consider security implications.
- When in doubt, prefer the more restrictive approach.
- Report potential security issues immediately — do not leave them as TODOs.
- Never disable security features (CSRF protection, CORS, etc.) "temporarily" — they tend to stay disabled.
- Run security audits (`npm audit`, dependency checks) before every release.
