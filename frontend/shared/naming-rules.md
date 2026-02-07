# Naming Rules

> Universal naming conventions for Claude Code. Apply to all projects regardless of tech stack.

## General Principles

- Names must be **descriptive and self-explanatory** — a reader should understand the purpose without looking at the implementation.
- Avoid abbreviations unless they are universally known (`id`, `url`, `api`, `db`, `config`).
- Do NOT use single-letter variables except in short loops (`i`, `j`, `k`):

```ts
// ❌ Incorrect
const d = new Date();
const u = await getUser();
const r = items.filter((x) => x.active);

// ✅ Correct
const currentDate = new Date();
const activeUser = await getUser();
const activeItems = items.filter((item) => item.active);
```

## Casing Conventions

| Entity | Casing | Example |
|--------|--------|---------|
| Variables | camelCase | `userName`, `isLoading` |
| Functions | camelCase | `getUserById`, `handleSubmit` |
| Classes | PascalCase | `UserService`, `HttpClient` |
| Interfaces / Types | PascalCase | `UserProps`, `ApiResponse` |
| Enums | PascalCase | `UserRole`, `OrderStatus` |
| Enum keys | PascalCase | `UserRole.Admin`, `OrderStatus.Pending` |
| Constants (primitive) | UPPER_SNAKE_CASE | `MAX_RETRY_COUNT`, `API_TIMEOUT` |
| Components | PascalCase | `UserProfileCard.vue`, `LoginForm.tsx` |
| Files (components) | PascalCase | `UserProfileCard.vue` |
| Files (utilities) | camelCase or kebab-case | `formatDate.ts`, `string-utils.ts` |
| Files (types) | kebab-case | `user.types.ts`, `api-response.types.ts` |
| CSS classes | kebab-case | `user-card`, `nav-item` |
| Folders | kebab-case | `user-profile/`, `auth-module/` |
| Env variables | UPPER_SNAKE_CASE | `VITE_API_BASE_URL` |
| Database tables | snake_case | `user_profiles`, `order_items` |
| API endpoints | kebab-case | `/user-profiles`, `/order-items` |

## Boolean Naming

Booleans must start with a verb prefix that implies true/false:

| Prefix | Usage | Example |
|--------|-------|---------|
| `is` | State | `isLoading`, `isActive`, `isVisible` |
| `has` | Possession | `hasPermission`, `hasError`, `hasChildren` |
| `can` | Capability | `canEdit`, `canDelete`, `canSubmit` |
| `should` | Recommendation | `shouldRefresh`, `shouldValidate` |
| `was` | Past state | `wasSubmitted`, `wasDeleted` |

```ts
// ❌ Incorrect
const loading = ref(false);
const admin = ref(true);
const edit = ref(false);

// ✅ Correct
const isLoading = ref(false);
const isAdmin = ref(true);
const canEdit = ref(false);
```

## Function Naming

Functions must start with a verb describing the action:

| Prefix | Usage | Example |
|--------|-------|---------|
| `get` | Retrieve data | `getUserById`, `getActiveItems` |
| `set` | Assign value | `setCurrentUser`, `setTheme` |
| `fetch` | API request | `fetchUserList`, `fetchOrderDetails` |
| `create` | Instantiate | `createOrder`, `createNotification` |
| `update` | Modify existing | `updateProfile`, `updateSettings` |
| `delete` / `remove` | Remove | `deleteUser`, `removeItem` |
| `handle` | Event handler | `handleSubmit`, `handleClick` |
| `on` | Event callback | `onSuccess`, `onError`, `onChange` |
| `validate` | Validation | `validateEmail`, `validateForm` |
| `format` | Transform display | `formatDate`, `formatCurrency` |
| `parse` | Transform data | `parseResponse`, `parseCSV` |
| `convert` | Type conversion | `convertToDTO`, `convertCurrency` |
| `check` / `verify` | Boolean check | `checkPermission`, `verifyToken` |
| `init` / `setup` | Initialize | `initApp`, `setupRouter` |
| `open` / `close` | Toggle UI | `openModal`, `closeDrawer` |
| `show` / `hide` | Visibility | `showNotification`, `hideTooltip` |
| `enable` / `disable` | Toggle state | `enableFeature`, `disableAutoSave` |

- Avoid generic or single-word function names:

```ts
// ❌ Incorrect
close()
open()
delete()
process()
run()

// ✅ Correct
closeModal()
openSettingsDrawer()
deleteDocument()
processPayment()
runValidation()
```

## Component Naming

- Component names must be **multi-word** and descriptive.
- Include the module/feature name as a prefix:

```
// ❌ Incorrect
Modal.vue
Card.vue
Button.vue
List.vue

// ✅ Correct
CashbackOpenModal.vue
UserProfileCard.vue
BaseButton.vue (shared/global components)
CandidateList.vue
```

- Icon components start with `Icon` prefix: `IconClose.vue`, `IconArrowLeft.vue`
- Page wrapper components include the page context: `LoginPage.vue`, `DashboardPage.vue`

## Composable / Hook Naming

- Always start with `use` prefix:

```ts
// ❌ Incorrect
authLogic()
formHandler()
apiClient()

// ✅ Correct
useAuth()
useForm()
useApi()
useDebounce()
```

## Event Naming

- Emitted events use camelCase:

```ts
// ❌ Incorrect
emit("do-something");
emit("CLOSE_MODAL");

// ✅ Correct
emit("doSomething");
emit("closeModal");
emit("update:modelValue");
```

## Avoid Unnecessary CamelCase Splits

Do not add camelCase breaks to compound words:

```ts
// ❌ Incorrect
withOut
withIn
reTry

// ✅ Correct
without
within
retry
```

## Rules

- Naming is documentation — invest time in choosing clear, descriptive names.
- Be consistent — use the same name for the same concept everywhere.
- If a name requires a comment to explain, the name is not good enough.
- Longer, descriptive names are better than short, cryptic ones.
- When renaming, search the entire codebase for all references to ensure consistency.
