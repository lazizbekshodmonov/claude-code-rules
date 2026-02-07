# Code Organization Rules

> Universal code organization rules for frontend projects. Apply to all frameworks (Vue, React, Svelte, Angular, etc.).

## Index Files

- `index.ts` files must contain **only re-exports** — no logic, no state, no side effects:

```ts
// ❌ Incorrect — logic inside index.ts
export const formatDate = (date: Date) => { ... };

// ✅ Correct — re-exports only
export { UserProfileCard } from "./UserProfileCard";
export { useUserProfile } from "./useUserProfile";
export type { UserProfileProps } from "./user-profile.types";
```

## Prettier and ESLint

> **CRITICAL:** If Prettier and/or ESLint are NOT installed in the project, they must be installed and fully configured before writing any code.

### Required Setup Steps

1. Check if `prettier` and `eslint` exist in `package.json` (`devDependencies`).
2. If missing, install all required packages.
3. Create `.prettierrc` with the project's Prettier config.
4. Create ESLint config with framework-specific plugins.
5. Create `.prettierignore` and `.eslintignore`:

```
node_modules
dist
.nuxt
.next
.output
coverage
```

6. Add lint scripts to `package.json` if missing:

```json
{
  "scripts": {
    "lint": "eslint . --fix",
    "format": "prettier --write \"**/*.{vue,tsx,ts,js,json,css,md}\""
  }
}
```

7. Run `npm run lint` and `npm run format` to verify the setup works.

### Prettier Base Config

The project must have a consistent Prettier configuration. Key rules:

- Always use **semicolons** (`;`)
- Use consistent quotes (double or single — pick one for the project)
- Indent with **2 spaces** — never tabs
- Use **trailing commas** in ES5-compatible positions
- Always use **parentheses** around arrow function parameters: `(x) => x`
- Line endings: **LF** (`\n`) — not CRLF

## Code Formatting Compliance

- Code must fully comply with the project's Prettier and linter configuration.
- Code must not contain unnecessary line breaks, extra indentation, or trailing spaces.
- Run `npm run lint` and `npm run format` before committing.

## Import Order

Organize imports in a consistent order:

1. Framework/library imports (`vue`, `react`, `@nestjs/*`)
2. Third-party library imports (`axios`, `lodash`, `dayjs`)
3. Absolute path imports (`@/services`, `@/components`)
4. Relative imports (`./`, `../`)
5. Type imports (`import type { ... }`)
6. Style imports (`import "./styles.css"`)

```ts
// ✅ Correct order
import { ref, computed } from "vue";
import axios from "axios";
import { userService } from "@/services";
import { UserCard } from "../components";
import type { User } from "./user.types";
import "./styles.css";
```

## One Class Per File

- Only one class per file.
- The class name must match the file/module name.

## Arrow Functions vs Function Declarations

- Prefer arrow functions over function declarations unless hoisting is needed.
- Declare arrow functions before function declarations within the same scope.

## Exceptions / Non-Compliance

Any deviation from project rules is allowed **only** when:

1. There is no other viable option.
2. The decision is clearly justified with at least one example.
3. Pros and cons of both approaches are listed.
4. The decision is approved by the Team Lead.

## Rules

- `index.ts` files contain only re-exports — no logic.
- Prettier and ESLint must be configured before writing any code.
- Follow consistent import ordering.
- One class per file.
- Prefer arrow functions over function declarations.
- Any rule deviation must be justified and approved.
