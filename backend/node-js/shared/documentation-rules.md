# Documentation Rules

> Backend documentation rules for Claude Code. Apply to all Node.js backend projects.

## Swagger / OpenAPI

- **Every API endpoint** must be documented with Swagger/OpenAPI decorators or annotations:

```ts
// NestJS example
@ApiOperation({ summary: "Get user by ID" })
@ApiParam({ name: "id", description: "User UUID", example: "550e8400-e29b-41d4-a716-446655440000" })
@ApiResponse({ status: 200, description: "User found", type: UserResponseDto })
@ApiResponse({ status: 404, description: "User not found" })
@Get(":id")
async findById(@Param("id") id: string): Promise<UserResponseDto> {
  return this.userService.findById(id);
}
```

- Swagger docs must include:
  - Endpoint description and summary
  - Request parameters (path, query, body) with types and examples
  - Response schemas for all status codes
  - Authentication requirements

## JSDoc for Functions

- **Every service method** must have a JSDoc comment:

```ts
// ❌ Incorrect — no documentation
async createUser(data: CreateUserDto): Promise<User> { ... }

// ✅ Correct
/**
 * Creates a new user account and sends a welcome email.
 *
 * @param data - User registration data
 * @returns The created user entity
 * @throws ConflictError if email is already registered
 * @throws ValidationError if data is invalid
 */
async createUser(data: CreateUserDto): Promise<User> { ... }
```

### Required JSDoc Tags

| Tag | When to Use |
|-----|------------|
| `@param` | Every parameter with a brief description |
| `@returns` | Describing the return value |
| `@throws` | If the function can throw errors |
| `@example` | For complex or non-obvious functions |
| `@deprecated` | If the function is deprecated with migration path |

### When JSDoc is Optional

- Simple getter/setter one-liners with self-explanatory names
- Standard lifecycle hooks
- Private helper functions with obvious logic

## DTO Documentation

- Document DTO fields with descriptions and examples:

```ts
export class CreateOrderDto {
  /** Product UUID to order */
  @ApiProperty({ example: "550e8400-e29b-41d4-a716-446655440000" })
  @IsUUID()
  productId: string;

  /** Number of items to order (1-100) */
  @ApiProperty({ example: 2, minimum: 1, maximum: 100 })
  @IsInt()
  @Min(1)
  @Max(100)
  quantity: number;
}
```

## Inline Comments

- Use inline comments sparingly — only for complex or non-obvious logic:

```ts
// ❌ Incorrect — obvious
const count = 0; // Initialize count

// ✅ Correct — explains WHY
// Retry with exponential backoff because the payment gateway
// occasionally returns 503 during peak hours
const result = await retryWithBackoff(() => processPayment(order), { maxRetries: 3 });
```

## File-Level Comments

- Add file-level comments for services, modules, and config files:

```ts
/**
 * @file order.service.ts
 * @description Handles order lifecycle: creation, payment processing,
 *   fulfillment tracking, and cancellation. Integrates with payment
 *   gateway and inventory service.
 */
```

## Environment Variables

- Document all required environment variables in `.env.example`:

```bash
# .env.example

# Server
PORT=3000
NODE_ENV=development

# Database
DATABASE_URL=postgresql://user:password@localhost:5432/myapp

# JWT
JWT_SECRET=your-secret-key-min-256-bits
JWT_EXPIRES_IN=1h

# Redis
REDIS_URL=redis://localhost:6379

# External Services
PAYMENT_GATEWAY_API_KEY=your-api-key
PAYMENT_GATEWAY_URL=https://api.payment.example.com
```

## Rules

- Documentation must be written **at the time of code creation** — never "I'll document later."
- Keep documentation in sync with code — update when logic changes.
- Write for the **next developer** who will read this code.
- Every public API endpoint must be in Swagger — no undocumented endpoints.
- Prefer self-documenting code (clear naming, typed parameters) over excessive comments.
- Never leave TODO comments without a linked issue or task number.
