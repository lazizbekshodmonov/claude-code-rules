# Naming Rules

> Backend naming rules for Claude Code. Apply to all Node.js backend projects.

## General Principles

- Names must be **descriptive and self-explanatory** — a reader should understand the purpose without looking at the implementation.
- Avoid abbreviations unless universally known (`id`, `url`, `api`, `db`, `config`, `dto`).
- Do NOT use single-letter variables except in short loops:

```ts
// ❌ Incorrect
const u = await getUser();
const r = await processRequest();

// ✅ Correct
const currentUser = await getUser();
const processedResult = await processRequest();
```

## Casing Conventions

| Entity | Casing | Example |
|--------|--------|---------|
| Variables | camelCase | `userName`, `isActive` |
| Functions / Methods | camelCase | `getUserById`, `createOrder` |
| Classes | PascalCase | `UserService`, `OrderController` |
| Interfaces / Types | PascalCase | `UserEntity`, `CreateUserDto` |
| Enums | PascalCase | `UserRole`, `OrderStatus` |
| Enum keys | UPPER_SNAKE_CASE | `UserRole.ADMIN`, `OrderStatus.PENDING` |
| Constants (primitive) | UPPER_SNAKE_CASE | `MAX_RETRY_COUNT`, `JWT_EXPIRES_IN` |
| Files (classes) | kebab-case | `user.service.ts`, `order.controller.ts` |
| Files (types/DTOs) | kebab-case | `create-user.dto.ts`, `user.entity.ts` |
| Folders | kebab-case | `user-management/`, `order-processing/` |
| Env variables | UPPER_SNAKE_CASE | `DATABASE_URL`, `JWT_SECRET` |
| Database tables | snake_case | `user_profiles`, `order_items` |
| Database columns | snake_case | `created_at`, `first_name` |
| API endpoints | kebab-case | `/user-profiles`, `/order-items` |

## File Naming

- Use the NestJS suffix convention for all files:

```
// ❌ Incorrect
users.ts
userHelper.ts
UserModel.ts

// ✅ Correct
users.controller.ts
users.service.ts
users.module.ts
users.guard.ts
users.interceptor.ts
users.middleware.ts
create-user.dto.ts
user.entity.ts
user.interface.ts
user.enum.ts
user.constant.ts
```

## Class Naming

- Class names include the suffix describing their role:

```ts
// ❌ Incorrect
class Users { }
class UserHandler { }

// ✅ Correct
class UsersController { }
class UsersService { }
class UsersModule { }
class UsersRepository { }
class AuthGuard { }
class LoggingInterceptor { }
class TransformPipe { }
```

## Method Naming

- Methods start with a verb describing the action:

| Prefix | Usage | Example |
|--------|-------|---------|
| `find` | Query data | `findById`, `findAll`, `findByEmail` |
| `create` | Insert new record | `createUser`, `createOrder` |
| `update` | Modify existing | `updateProfile`, `updateStatus` |
| `delete` / `remove` | Remove record | `deleteUser`, `removeExpiredTokens` |
| `validate` | Check validity | `validateToken`, `validateInput` |
| `check` / `verify` | Boolean check | `checkPermission`, `verifyOwnership` |
| `process` | Business logic | `processPayment`, `processRefund` |
| `send` | Dispatch | `sendEmail`, `sendNotification` |
| `generate` | Create output | `generateReport`, `generateToken` |
| `parse` | Transform input | `parseCSV`, `parseRequestBody` |
| `handle` | Event/error handler | `handleWebhook`, `handleError` |

```ts
// ❌ Incorrect
async user(id: string) { }
async doStuff() { }
async run() { }

// ✅ Correct
async findUserById(id: string) { }
async processPayment(orderId: string) { }
async generateMonthlyReport(month: number) { }
```

## Boolean Naming

Booleans must start with a verb prefix that implies true/false:

| Prefix | Usage | Example |
|--------|-------|---------|
| `is` | State | `isActive`, `isVerified`, `isExpired` |
| `has` | Possession | `hasPermission`, `hasSubscription` |
| `can` | Capability | `canAccess`, `canDelete` |
| `should` | Recommendation | `shouldRetry`, `shouldNotify` |

## DTO Naming

- Prefix with action, suffix with `Dto`:

```ts
// ❌ Incorrect
class UserData { }
class UserInput { }

// ✅ Correct
class CreateUserDto { }
class UpdateUserDto { }
class UserResponseDto { }
class UserQueryDto { }
class LoginDto { }
```

## Rules

- Naming is documentation — invest time in choosing clear, descriptive names.
- Be consistent — use the same name for the same concept everywhere.
- If a name requires a comment to explain, the name is not good enough.
- Longer, descriptive names are better than short, cryptic ones.
- When renaming, search the entire codebase for all references to ensure consistency.
