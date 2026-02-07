# Documentation Rules

> Universal code documentation requirements for Claude Code. Apply to all projects regardless of tech stack.

## JSDoc for Functions

**Every function must have a JSDoc comment** above it describing its purpose, parameters, and return value.

```ts
// ❌ Incorrect — no documentation
const calculateDiscount = (price: number, percent: number): number => {
  return price - (price * percent) / 100;
};

// ✅ Correct — clear JSDoc
/**
 * Calculates the discounted price based on the given percentage.
 *
 * @param price - Original price of the item
 * @param percent - Discount percentage (0–100)
 * @returns The final price after applying the discount
 */
const calculateDiscount = (price: number, percent: number): number => {
  return price - (price * percent) / 100;
};
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
- Standard lifecycle hooks (e.g., `onMounted`, `useEffect`)
- Type-only declarations (interfaces, types, enums — use inline comments instead)

## Block Comments for Components

**Every component must have a detailed block comment** at the very top of the script section describing the component's purpose, usage context, and key behavior.

### Required Fields

| Field | Description |
|-------|------------|
| `@description` | What the component does and where it is used |
| `@props` | Brief summary of key props |
| `@emits` | List of emitted events with descriptions |
| `@usage` | A short usage example |

### Example (Vue)

```vue
<script setup lang="ts">
  /**
   * UserProfileCard
   *
   * @description Displays user avatar, name, and role badge.
   *   Used in the dashboard sidebar and team members list.
   *
   * @props
   *   - user: User object containing name, avatar URL, and role
   *   - compact: If true, renders a smaller card without the role badge
   *
   * @emits
   *   - click: Fired when the card is clicked
   *   - edit: Fired when the edit button is pressed
   *
   * @usage
   *   <UserProfileCard :user="currentUser" @click="openProfile" />
   */
</script>
```

### Example (React)

```tsx
/**
 * UserProfileCard
 *
 * @description Displays user avatar, name, and role badge.
 *   Used in the dashboard sidebar and team members list.
 *
 * @props
 *   - user: User object containing name, avatar URL, and role
 *   - compact: If true, renders a smaller card without the role badge
 *
 * @events
 *   - onClick: Fired when the card is clicked
 *   - onEdit: Fired when the edit button is pressed
 *
 * @usage
 *   <UserProfileCard user={currentUser} onClick={openProfile} />
 */
const UserProfileCard: React.FC<UserProfileCardProps> = ({ user, compact }) => {
  // ...
};
```

## Inline Comments

- Use inline comments sparingly — only for complex or non-obvious logic.
- Do NOT comment obvious code:

```ts
// ❌ Incorrect — obvious
const count = 0; // Initialize count to zero

// ✅ Correct — explains WHY, not WHAT
// Offset by 1 because the API returns 0-indexed page numbers
const displayPage = apiPage + 1;
```

## File-Level Comments

For utility files, services, and config files, add a brief file-level comment at the top:

```ts
/**
 * @file api.service.ts
 * @description Centralized HTTP client with interceptors for auth token
 *   refresh, error handling, and request/response logging.
 */
```

## README and Module Documentation

- Each major module or feature folder should have a brief `README.md` if the module has:
  - Non-obvious architecture decisions
  - Complex data flow
  - External service integrations
  - Setup or configuration requirements

## Rules

- Documentation must be written **at the time of code creation** — never "I'll document later."
- Keep documentation in sync with code — update comments when logic changes.
- Write for the **next developer** who will read this code, not for yourself.
- Prefer self-documenting code (clear naming, small functions) over heavy comments.
- Never leave TODO comments without a linked issue or task number.
