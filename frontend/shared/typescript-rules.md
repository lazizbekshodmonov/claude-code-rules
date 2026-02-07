# TypeScript Rules

> Universal TypeScript rules for frontend projects. Apply to all frameworks (Vue, React, Svelte, Angular, etc.).

## Type File Organization

- Extract all types into a `{name}.types.ts` file — do not define types inline in component or logic files:

```ts
// ❌ Incorrect — types mixed with logic
// user.service.ts
interface User {
  id: number;
  name: string;
}
const getUser = async (id: number): Promise<User> => { ... };

// ✅ Correct — types in a separate file
// user.types.ts
export interface User {
  id: number;
  name: string;
}

// user.service.ts
import type { User } from "./user.types";
const getUser = async (id: number): Promise<User> => { ... };
```

## Type Declaration Order

Within a types file, follow this order:

1. `enum`
2. `interface`
3. `type`

```ts
// ✅ Correct order
export enum UserRole {
  Admin = 0,
  User = 1,
}

export interface User {
  id: number;
  name: string;
  role: UserRole;
}

export type UserCreatePayload = Pick<User, "name" | "role">;
```

## No Nested Interfaces

- Do NOT write nested interfaces — extract them separately:

```ts
// ❌ Incorrect — nested interface
interface User {
  info: {
    genderId: number;
    address: {
      city: string;
    };
  };
}

// ✅ Correct — extracted
interface UserAddress {
  city: string;
}

interface UserInfo {
  genderId: number;
  address: UserAddress;
}

interface User {
  info: UserInfo;
}
```

## Cross-File Type References

- Do NOT reference interface fields for typing if the interface is in a different file:

```ts
// user.types.ts
export interface User {
  name: string;
  age: number;
}

// user-utils.ts
// ❌ Incorrect — cross-file field reference
export const getAgeByName = (name: User["name"]): User["age"] => { ... };

// ✅ Correct — use primitive types directly
export const getAgeByName = (name: string): number => { ... };
```

- Exception: If both the referencing type and the referenced type are in the **same file**, linking fields is acceptable:

```ts
// same-file.types.ts
export interface SomeComponentProps {
  name: string;
}

// ✅ OK — same file
export interface SomeComponentEmits {
  (event: "update:name", value: SomeComponentProps["name"]): void;
}
```

## When to Type Explicitly

Type everything, with these **exceptions** (no explicit typing needed):

```ts
// Function already returns a typed value
const result = getResult();

// Value is a primitive
const count = 1;

// Composable / hook return type
const { data, error } = useAuth();

// Object constants — use `as const`
const config = { MAX_RETRIES: 3, TIMEOUT: 5000 } as const;

// Boolean with a default value (type is inferred)
const isActive = ref(false);       // Vue
const [isActive] = useState(false); // React

// Ref/state without default or with a union type — specify explicitly
const isBlocked = ref<boolean | undefined>();
const [user, setUser] = useState<User | null>(null);
```

## Enum Rules

- Always assign **explicit values** to numeric enum members:

```ts
// ❌ Incorrect — implicit values shift when keys are reordered
enum State {
  Active,
  Inactive,
}

// ✅ Correct — explicit values
enum State {
  Active = 0,
  Inactive = 1,
}
```

- All enum keys must use **PascalCase**:

```ts
// ❌ Incorrect
enum UserRole {
  ADMIN = "admin",
  REGULAR_USER = "user",
}

// ✅ Correct
enum UserRole {
  Admin = "admin",
  RegularUser = "user",
}
```

- Prefer `const enum` to prevent runtime code generation (when compatible with your build tool).

## Avoid `any`

- Never use `any` — use `unknown` when the type is truly unknown, then narrow:

```ts
// ❌ Incorrect
const parseData = (raw: any): User => { ... };

// ✅ Correct
const parseData = (raw: unknown): User => {
  if (!isUser(raw)) throw new Error("Invalid user data");
  return raw;
};
```

## Rules

- Extract all types into `{name}.types.ts` files.
- Follow declaration order: `enum` → `interface` → `type`.
- Never nest interfaces — extract them separately.
- Do not cross-reference interface fields from different files.
- Type everything unless the type is clearly inferred.
- Always assign explicit values to numeric enums.
- Enum keys use PascalCase.
- Never use `any` — use `unknown` and type narrowing.
