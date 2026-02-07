# Code Writing Rules (Vue + TypeScript)

## 1. Vue

### 1.1. General Vue Rules

- Do NOT use native DOM methods to access elements, check classes, or other attributes. Vue provides enough functionality to handle various behaviors without direct DOM access. Use Vue refs instead.

### 1.2. Naming

- Event names in interfaces with the `Emits` suffix must use camelCase (per [Vue docs](https://vuejs.org/guide/components/events)):

```ts
// ❌ Incorrect
interface Emits {
  (event: 'do-something')
}

// ✅ Correct
interface Emits {
  (event: 'doSomething')
}
```

### 1.3. Watch

- Avoid using `watch` whenever possible. Reasons:
  - `watch` reacts to any state change. If a side effect should only trigger under specific conditions (e.g., state changed by a child component but not by the parent), you end up writing workarounds inside the watcher.
  - `watch` creates implicit links between state and side effects, making application logic harder to trace across components.
  - Multiple watchers act as independent entry points, complicating data flow understanding.
- **Preferred approach:** Use explicit state update methods as single entry points for all changes. This makes data flow predictable and simplifies adding sequential logic. Favor the imperative approach ("call a specific method → controlled state change") over the reactive one ("something changed → automatic reaction").

### 1.4. Components

- Use lightweight `UFlex` instead of `USpace`, but if you just need a wrapper, use a plain `<div>` — do not abuse `UFlex`.
- Component names must be descriptive — specify what the component does and which module it belongs to:

```
CashbackModal.vue       // ❌ Incorrect
CashbackOpenModal.vue   // ✅ Correct
```

- Props/Emits interfaces and the main composable must be named consistently with the file names. Component structure example:

```
some-component/
├── some-component.types.ts    // SomeComponentProps, SomeComponentEmits
├── useSomeComponent.ts        // Main composable
├── SomeComponent.vue          // Component file
└── index.ts                   // Re-export with named alias
```

```ts
// some-component/some-component.types.ts
export interface SomeComponentProps {}
export interface SomeComponentEmits {}
```

```ts
// some-component/useSomeComponent.ts
export default function useSomeComponent(
  props: SomeComponentProps,
  emit: SomeComponentEmits
) {
  return {};
}
```

```vue
// some-component/SomeComponent.vue
<script lang="ts" setup>
import { SomeComponentProps, SomeComponentEmits } from './some-component.types';
import useSomeComponent from './useSomeComponent';

const props = defineProps<SomeComponentProps>();
const emit = defineEmits<SomeComponentEmits>();

const {} = useSomeComponent(props, emit);
</script>
```

```ts
// some-component/index.ts — always use a named alias
export { default as SomeComponent } from './SomeComponent.vue';
```

- **Every component must have a detailed block comment** at the very top of the `<script>` tag describing the component's purpose, usage context, and key behavior:

```vue
<script lang="ts" setup>
/**
 * SomeComponent
 *
 * @description Displays a modal dialog for opening cashback details.
 *   Used in the CashbackModule on the user dashboard page.
 *
 * @props
 *   - title: Modal header text
 *   - isVisible: Controls modal visibility
 *
 * @emits
 *   - close: Fired when the user closes the modal
 *   - confirm: Fired when the user confirms the action
 *
 * @usage
 *   <CashbackOpenModal title="Details" :is-visible="show" @close="hide" />
 */
import { SomeComponentProps, SomeComponentEmits } from './some-component.types';
import useSomeComponent from './useSomeComponent';

const props = defineProps<SomeComponentProps>();
const emit = defineEmits<SomeComponentEmits>();

const {} = useSomeComponent(props, emit);
</script>
```

- The comment must include:
  - `@description` — what the component does and where it is used
  - `@props` — brief summary of key props
  - `@emits` — list of emitted events with descriptions
  - `@usage` — a short usage example

- Extract ALL component logic into the main composable (as shown above). The `<script>` tag should contain only macro usage (`defineProps`, `defineEmits`) and the composable call — no other logic.
- Avoid creating new shared components in `shared/components` or `shared/UI`. Look for existing local solutions or components in `uzum-ui` first. If none exist, check with frontend colleagues and request development in `uzum-ui` if needed.

- If the project uses a UI library (e.g., `uzum-ui`, Vuetify, Element Plus, PrimeVue), you **must** use its components exclusively. Do NOT create custom HTML tags or wrapper elements that duplicate existing library components:

```vue
<!-- ❌ Incorrect — custom tag when UI library has the component -->
<custom-button @click="submit">Save</custom-button>
<div class="custom-input-wrapper">
  <input type="text" v-model="name" />
</div>

<!-- ✅ Correct — use the UI library component -->
<UButton @click="submit">Save</UButton>
<UInput v-model="name" />
```
- Remove the root `<div>` wrapper if it is unnecessary. Also remove the `<style>` tag if the component has no additional styles.

- **Keep components small and single-purpose.** Each component should handle one clear responsibility. If a component grows too large or handles multiple concerns, break it down into smaller, focused child components:

```
<!-- ❌ Incorrect — one large component doing everything -->
UserDashboard.vue  (profile + stats + activity + settings all in one)

<!-- ✅ Correct — split into focused components -->
UserDashboard.vue
├── UserProfileCard.vue
├── UserStatsPanel.vue
├── UserActivityFeed.vue
└── UserSettingsForm.vue
```

- **Register frequently used elements as global components.** If a component (e.g., button, modal, input, icon) is used across multiple modules, register it globally to avoid repetitive imports. Keep the global registry clean — only promote components that are truly reused project-wide:

```ts
// ❌ Incorrect — importing the same component everywhere
import BaseButton from '@/shared/ui/BaseButton.vue';
import BaseInput from '@/shared/ui/BaseInput.vue';
// ... repeated in dozens of files

// ✅ Correct — register once globally
// main.ts or global-components.ts
app.component('BaseButton', BaseButton);
app.component('BaseInput', BaseInput);
app.component('BaseModal', BaseModal);
```
- Code order inside a composable file:
  1. Use-function calls and primitive constants (`use*`, `const`)
  2. `ref` / `reactive`
  3. `computed`
  4. Arrow functions, declarative functions
  5. Vue lifecycle hooks

### 1.5. View / Page Components

- **View (page) files must be thin wrappers only.** Do NOT write any logic, HTML markup, or styles directly inside view/page files (e.g., files in `pages/` or `views/` directory).
- All logic, template markup, and composables must live in a `components/` folder next to the page file. The page file should only import and render the main component.

**Folder structure:**

```
pages/
└── auth/
    ├── login.vue                        // ← Thin wrapper ONLY
    └── components/
        ├── LoginPage.vue                // ← All HTML / template here
        ├── login-page.types.ts          // ← Types
        ├── useLoginPage.ts              // ← All logic here
        ├── __tests__/
        │   └── LoginPage.spec.ts
        ├── LoginForm.vue                // ← Child components
        ├── LoginSocialButtons.vue
        └── index.ts
```

**Page file — thin wrapper:**

```vue
<!-- pages/auth/login.vue -->
<!-- ❌ Incorrect — logic and HTML inside the page file -->
<script lang="ts" setup>
const email = ref('');
const password = ref('');
const handleLogin = async () => { /* ... */ };
</script>

<template>
  <div class="login-container">
    <form @submit.prevent="handleLogin">
      <input v-model="email" />
      <input v-model="password" type="password" />
      <button type="submit">Login</button>
    </form>
  </div>
</template>
```

```vue
<!-- pages/auth/login.vue -->
<!-- ✅ Correct — page is just a wrapper -->
<script lang="ts" setup>
import { LoginPage } from './components';
</script>

<template>
  <LoginPage />
</template>
```

**All logic goes into the component and composable:**

```ts
// pages/auth/components/useLoginPage.ts
/**
 * useLoginPage
 *
 * @description Handles login form state, validation, and API submission.
 */
export default function useLoginPage() {
  const email = ref('');
  const password = ref('');

  const handleLogin = async () => { /* ... */ };

  return { email, password, handleLogin };
}
```

```vue
<!-- pages/auth/components/LoginPage.vue -->
<script lang="ts" setup>
/**
 * LoginPage
 *
 * @description Main login page content with form and social login options.
 *   Rendered by pages/auth/login.vue wrapper.
 */
import useLoginPage from './useLoginPage';

const { email, password, handleLogin } = useLoginPage();
</script>

<template>
  <div class="flex flex-col items-center">
    <LoginForm
      v-model:email="email"
      v-model:password="password"
      @submit="handleLogin"
    />
    <LoginSocialButtons />
  </div>
</template>
```

- This rule applies to **every** page/view in the project — no exceptions.
- Benefits: keeps routing files clean, makes page logic testable, enables component reuse, and follows the single-responsibility principle.

## 2. JavaScript

### 2.1. General JavaScript Rules

- Always use semicolons (`;`) to indicate the end of a code block.
- Prefer arrow functions over declarative functions unless hoisting is needed.
- Do NOT access the DOM directly via JavaScript — use Vue refs instead.

### 2.2. Naming

- Use `camelCase` for everything, except:
  - Classes → `PascalCase`
  - Primitive constants → `UPPER_CASE`
- Do not add unnecessary camelCase splits. Write `without`, `within` — not `withOut`, `withIn`.

### 2.3. Functions

- Declare arrow functions before declarative functions.
- Avoid single-word function names like `close`, `open`, `delete`. Use descriptive names instead:

```ts
// ❌ Incorrect
close()
open()
delete()

// ✅ Correct
closeModal()
openModal()
deleteDocument()
```

- **Every function must have a JSDoc comment** above it describing its purpose, parameters, and return value:

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
 * @param percent - Discount percentage (0-100)
 * @returns The final price after applying the discount
 */
const calculateDiscount = (price: number, percent: number): number => {
  return price - (price * percent) / 100;
};
```

- JSDoc must include:
  - `@param` for every parameter with a brief description
  - `@returns` describing the return value
  - `@throws` if the function can throw errors
  - `@example` for complex or non-obvious functions

### 2.4. Classes

- Only one class per file.
- The class name must match the module name.

## 3. TypeScript

### 3.1. General TypeScript Rules

- Extract all types into a `{name}.types.ts` file.
- Type declaration order: `enum` → `interface` → `type`.
- Do NOT reference interface fields for typing if the interface is in a different file:

```ts
// users.types.ts
export interface User {
  name: string;
  age: number;
}

// users-utils.ts
export const getAgeByName = (name: User['name']): User['age'] => ''; // ❌ Incorrect
export const getAgeByName = (name: string): number => '';             // ✅ Correct
```

- If the Emits interface directly depends on the Props interface AND both are in the same types file, link emit fields to props fields directly:

```ts
// some-component.types.ts
export interface SomeComponentProps {
  name: string;
}

export interface SomeComponentEmits {
  (event: 'update:name', value: SomeComponentProps['name']): void;
}
```

- Do NOT write nested interfaces. Extract them separately:

```ts
// ❌ Incorrect
interface User {
  info: {
    genderId: number;
  };
}

// ✅ Correct
interface UserInfo {
  genderId: number;
}

interface User {
  info: UserInfo;
}
```

- Type everything, with the following **exceptions** (no explicit typing needed):

```ts
// Function already returns a typed value
const result = getResult();

// Value is a primitive
const primitive = 1;

// Composable / store return type
const useComposable = () => {};

// Object constants — use `as const`; primitive constants are already narrowed
const someConfig = {} as const;

// Boolean ref with a default value
const isActive = ref(false);

// Ref without default or with a union type — specify type explicitly
const isBlocked = ref<boolean | undefined>();
```

### 3.2. Naming

- All `enum` keys must use `PascalCase`.

### 3.3. Enums

- Always assign explicit values to numeric enum members to guarantee values even if key order changes:

```ts
// ❌ Incorrect
enum State {
  Active,
  Inactive,
}

// ✅ Correct
enum State {
  Active = 0,
  Inactive = 1,
}
```

- Always prefer `const enum` to prevent runtime code generation.

## 4. Module / Code Organization

### 4.1. General Rules

- `index.ts` files must contain only re-exports — no logic.

### 4.2. Formatting

- Code must fully comply with the project's Prettier and linter configuration.
- Code must not contain unnecessary line breaks, extra indentation, or trailing spaces.

> **CRITICAL: If Prettier and/or ESLint are NOT installed in the project, Claude Code must install and fully configure them before writing any code.**
>
> Steps to follow:
> 1. Check if `prettier` and `eslint` exist in `package.json` (`devDependencies`)
> 2. If missing, install all required packages:
>    ```bash
>    npm install -D prettier eslint eslint-plugin-vue @typescript-eslint/parser @typescript-eslint/eslint-plugin eslint-config-prettier
>    ```
> 3. Create `.prettierrc` with the project's Prettier config (see section 4.2.1)
> 4. Create `.eslintrc.json` with the project's ESLint config (see section 4.2.2)
> 5. Create `.prettierignore` and `.eslintignore` if they don't exist:
>    ```
>    node_modules
>    dist
>    .nuxt
>    .output
>    coverage
>    ```
> 6. Add lint scripts to `package.json` if missing:
>    ```json
>    {
>      "scripts": {
>        "lint": "eslint . --ext .vue,.ts,.js --fix",
>        "format": "prettier --write \"**/*.{vue,ts,js,json,css,md}\""
>      }
>    }
>    ```
> 7. Run `npm run lint` and `npm run format` to verify the setup works
>
> This must be done as the **very first step** before any other task.

#### 4.2.1. Prettier Base Config

The project uses the following Prettier configuration. All generated code **must** comply with these settings:

```json
{
  "semi": true,
  "singleQuote": false,
  "tabWidth": 2,
  "useTabs": false,
  "printWidth": 200,
  "trailingComma": "es5",
  "bracketSpacing": true,
  "arrowParens": "always",
  "endOfLine": "lf",
  "vueIndentScriptAndStyle": true
}
```

Key rules derived from this config:
- Always use **semicolons** (`;`)
- Always use **double quotes** (`"`) — not single quotes
- Indent with **2 spaces** — never tabs
- Max line width for `<script>` and `<style>` blocks: **200 characters**
- Trailing commas in **ES5-compatible** positions (objects, arrays, function params)
- Always use **parentheses** around arrow function parameters: `(x) => x`
- Line endings: **LF** (`\n`) — not CRLF

> **Important:** Prettier does not support per-block `printWidth` in Vue files. The `<template>` block line width is enforced via **ESLint** rules below.

#### 4.2.2. ESLint Template Rules (printWidth: 100)

Since Prettier cannot set a separate `printWidth` for `<template>`, the following `eslint-plugin-vue` rules enforce a **max line width of ~100 characters** in template blocks:

```js
// .eslintrc or eslint.config.js (vue rules)
{
  "rules": {
    // Force one attribute per line when element exceeds 100 chars or has 3+ attributes
    "vue/max-attributes-per-line": ["error", {
      "singleline": { "max": 3 },
      "multiline": { "max": 1 }
    }],

    // Force multiline elements to have closing bracket on a new line
    "vue/html-closing-bracket-newline": ["error", {
      "singleline": "never",
      "multiline": "always"
    }],

    // Force first attribute on a new line for multiline elements
    "vue/first-attribute-linebreak": ["error", {
      "singleline": "ignore",
      "multiline": "below"
    }],

    // Max line length in template: 100
    "vue/max-len": ["error", {
      "code": 200,
      "template": 100,
      "ignoreUrls": true,
      "ignoreStrings": false,
      "ignoreTemplateLiterals": true,
      "ignoreHTMLAttributeValues": false,
      "ignoreHTMLTextContents": true
    }],

    // Self-close components with no children
    "vue/html-self-closing": ["error", {
      "html": { "void": "always", "normal": "never", "component": "always" },
      "svg": "always",
      "math": "always"
    }],

    // Consistent indentation in template (2 spaces)
    "vue/html-indent": ["error", 2, {
      "attribute": 1,
      "baseIndent": 1,
      "closeBracket": 0,
      "alignAttributesVertically": true
    }]
  }
}
```

**Summary of line width per block:**

| Block | Max Width | Enforced By |
|-------|-----------|-------------|
| `<script>` | 200 chars | Prettier |
| `<template>` | 100 chars | ESLint (`vue/max-len`) |
| `<style>` | 200 chars | Prettier |

#### 4.2.3. `<script>` Block Rules

- **Max line width: 200 characters** (enforced by Prettier).
- `vueIndentScriptAndStyle: true` — all code inside `<script>` must be **indented by 2 spaces** from the tag:

```vue
<!-- ❌ Incorrect — no indentation from script tag -->
<script setup lang="ts">
import { ref } from "vue";

const count = ref(0);
</script>

<!-- ✅ Correct — indented 2 spaces from script tag -->
<script setup lang="ts">
  import { ref } from "vue";

  const count = ref(0);
</script>
```

- Always use `<script setup lang="ts">` with TypeScript.
- Block comment (JSDoc) for the component must come first, before any imports.
- Code order inside `<script>`:
  1. Component JSDoc comment
  2. Type imports (`import type { ... }`)
  3. Module imports (`import { ... }`)
  4. `defineProps` / `defineEmits` macros
  5. Composable destructuring
  6. No other logic — everything else belongs in the composable

```vue
<script setup lang="ts">
  /**
   * SomeComponent
   * @description ...
   */
  import type { SomeComponentProps, SomeComponentEmits } from "./some-component.types";
  import { useSomeComponent } from "./useSomeComponent";

  const props = defineProps<SomeComponentProps>();
  const emit = defineEmits<SomeComponentEmits>();

  const { data, handleClick } = useSomeComponent(props, emit);
</script>
```

#### 4.2.4. `<template>` Block Rules

- **Max line width: 100 characters** (enforced by ESLint `vue/max-len`, see section 4.2.2).
- Template content starts at **zero indentation** from the `<template>` tag (standard Vue behavior — `vueIndentScriptAndStyle` does not affect template):

```vue
<!-- ✅ Correct -->
<template>
  <div class="flex items-center gap-4">
    <n-button type="primary" size="large" @click="handleClick">
      Submit
    </n-button>
  </div>
</template>
```

- Use **Tailwind CSS classes** instead of custom CSS wherever possible.
- Use **UI library components** (e.g., `<n-form>`, `<n-button>`, `<n-input>`) instead of native HTML tags.
- Attribute ordering on elements (recommended):
  1. `v-if` / `v-else-if` / `v-else` / `v-show` / `v-for`
  2. `ref`
  3. `v-model` / `v-model:*`
  4. Static props (`:model`, `:rules`, `label`, `placeholder`)
  5. Dynamic props (`:loading`, `:disabled`)
  6. `class` / `:class`
  7. `@click` / `@submit` / other event handlers

```vue
<!-- ❌ Incorrect — messy attribute order -->
<n-button @click="submit" :loading="isLoading" type="primary" v-if="canSubmit" size="large">
  Submit
</n-button>

<!-- ✅ Correct — clean attribute order -->
<n-button v-if="canSubmit" type="primary" size="large" :loading="isLoading" @click="submit">
  Submit
</n-button>
```

- Multi-attribute elements: if an element has **3+ attributes**, place each attribute on its own line:

```vue
<!-- ❌ Incorrect — too many attributes on one line -->
<n-input v-model:value="formValue.email" size="large" placeholder="Enter email" :disabled="isLoading" @blur="validate" />

<!-- ✅ Correct — one attribute per line -->
<n-input
  v-model:value="formValue.email"
  size="large"
  placeholder="Enter email"
  :disabled="isLoading"
  @blur="validate"
/>
```

#### 4.2.5. `<style>` Block Rules

- **Max line width: 200 characters** (enforced by Prettier).
- `vueIndentScriptAndStyle: true` — all CSS inside `<style>` must be **indented by 2 spaces** from the tag:

```vue
<!-- ❌ Incorrect — no indentation from style tag -->
<style scoped>
.container {
  display: flex;
}
</style>

<!-- ✅ Correct — indented 2 spaces from style tag -->
<style scoped>
  .container {
    display: flex;
  }
</style>
```

- Always use `scoped` unless global styles are explicitly needed.
- Prefer Tailwind classes in `<template>` over writing custom CSS. Only add `<style>` when Tailwind cannot achieve the result.
- Remove the `<style>` tag entirely if the component has no custom styles — do NOT leave empty style blocks.
- Always use semicolons after every CSS declaration.
- Use CSS variables (or Tailwind theme tokens) for colors — never hardcode color values.

### 4.3. Design System

- All colors must come from the Design System. If a color is missing, ask the designers to update the mockup.

## 5. General Rules

### 5.1. Exceptions to These Rules

Any deviation from the rules above (anything marked as "avoid", "prefer not to", etc.) is allowed **only** when:

1. There is no other viable option.
2. The decision is clearly and thoroughly justified with at least one example.
3. Pros and cons of both approaches are listed.
4. The decision is approved by the Team Lead.

### 5.2. Non-Compliance

Any inconsistencies with these rules must be:

1. Clearly and thoroughly justified with examples for both approaches.
2. Accompanied by listed pros and cons of both approaches.
3. Approved by the Team Lead.

### 5.3. Code Review

- Write clear, detailed review comments.
- The reviewer must resolve their own comments — not the author.
- Do not approve a PR until the developer has reviewed or explained every comment.

### 5.4. Planning Mode (Claude Code)

- When given a task, **always start in planning mode** before writing any code. Think through the approach, affected files, and potential side effects first.
- Save every task plan as a Markdown file in the `planning-claude-tasks/` folder at the project root.
- File naming convention: `YYYY-MM-DD-short-task-description.md` (e.g., `2026-02-06-add-user-auth.md`).
- Each plan file must include:

```markdown
# Task: [Short description]

## Date
YYYY-MM-DD

## Objective
What needs to be done and why.

## Affected Files
- List of files that will be created / modified / deleted

## Implementation Plan
1. Step-by-step breakdown of the approach
2. ...
3. ...

## Potential Risks / Side Effects
- Any risks, breaking changes, or dependencies to consider

## Status
- [ ] Planned
- [ ] In Progress
- [ ] Completed
```

- Do NOT start writing or modifying code until the plan is saved and reviewed.
- Update the plan's status as the task progresses.
- This folder must be committed to Git so the team can track all AI-assisted task history.

### 5.5. Mandatory Testing (Claude Code)

- **Every** new or modified component and function must have accompanying unit and integration tests. No exceptions.
- Test files must be co-located with the source file using the `.test.ts` or `.spec.ts` suffix:

```
some-component/
├── SomeComponent.vue
├── useSomeComponent.ts
├── some-component.types.ts
├── __tests__/
│   ├── SomeComponent.spec.ts         // Unit test for the component
│   ├── useSomeComponent.spec.ts      // Unit test for the composable
│   └── SomeComponent.integration.spec.ts  // Integration test
└── index.ts
```

- **Unit tests** must cover:
  - Component rendering with different props
  - Emit events and user interactions
  - Composable return values and state changes
  - Utility / helper function edge cases

- **Integration tests** must cover:
  - Component interaction with child components
  - API call flows (mocked)
  - Store / state management integration
  - Route-dependent behavior (if applicable)

- **After writing tests, always run them** before considering the task complete:

```bash
# Run all tests
npm run test

# Run specific test file
npm run test -- --filter SomeComponent

# Run with coverage
npm run test -- --coverage
```

- All tests **must pass** before the task status can be marked as "Completed" in the planning file.
- If a test fails, fix the code or the test — never skip or delete a failing test.
- Minimum expected coverage per new file: **80%** (statements and branches).

### 5.6. Parallel Multi-Agent Execution (Claude Code)

- When a task is **large or complex** (e.g., involves multiple modules, 3+ files, or independent subtasks), Claude Code **must** break it into smaller subtasks and run multiple sub-agents in parallel using the `Task` tool.
- Do NOT execute large tasks sequentially when subtasks are independent of each other.

**When to parallelize:**
- Creating multiple components that don't depend on each other
- Writing tests for several files simultaneously
- Refactoring across multiple modules
- Generating types, composables, and components for a new feature at once

**How it should work:**

```
Large Task: "Build a user profile page with avatar upload, stats, and settings"

Sequential ❌ (slow):
  1. Build UserAvatar.vue → wait
  2. Build UserStats.vue → wait
  3. Build UserSettings.vue → wait
  4. Write tests → wait

Parallel ✅ (fast):
  Agent 1 → UserAvatar.vue + its composable + tests
  Agent 2 → UserStats.vue + its composable + tests
  Agent 3 → UserSettings.vue + its composable + tests
  Main    → UserProfilePage.vue (assembles after agents finish)
```

- Each sub-agent must follow **all project rules** (planning, testing, code style).
- After all parallel agents finish, the main agent must:
  1. Verify all subtasks are complete
  2. Run the full test suite to check for conflicts
  3. Ensure components integrate correctly together
- Document the parallelization strategy in the task's planning file under a `## Subtasks` section:

```markdown
## Subtasks
| # | Subtask | Agent | Status |
|---|---------|-------|--------|
| 1 | UserAvatar component + tests | Agent 1 | ✅ Done |
| 2 | UserStats component + tests | Agent 2 | ✅ Done |
| 3 | UserSettings component + tests | Agent 3 | ✅ Done |
| 4 | Integration + final test run | Main | ✅ Done |
```

#### 5.6.1. Context Limit Prevention

> **CRITICAL:** Each sub-agent has a limited context window. To prevent `Context limit reached` errors, follow these strict limits:

- Each sub-agent must handle **no more than 5–10 files** per task. If more files need changes, split into additional sequential batches.
- Do NOT read entire large files when only a small section needs editing. Use targeted reads (specific line ranges) instead of full file reads.
- Before starting parallel agents, **estimate the scope** of each subtask. If a subtask involves more than 10 files or files larger than 500 lines, split it further.
- If an agent hits the context limit mid-task:
  1. Use `/compact` to compress the context and continue
  2. If still insufficient, use `/clear` and restart only the remaining unfinished work
  3. Document which files were completed and which are still pending in the planning file

**Agent scoping example:**

```
Large Task: "Add JSDoc comments to all 40 components"

❌ Bad — 2 agents, 20 files each (will hit context limit):
  Agent 1 → 20 components
  Agent 2 → 20 components

✅ Good — 8 agents, 5 files each (stays within limits):
  Agent 1 → Components A-E (5 files)
  Agent 2 → Components F-J (5 files)
  Agent 3 → Components K-O (5 files)
  ...and so on
```

- Prefer **more agents with fewer files** over fewer agents with many files.
- After each batch completes, verify results before starting the next batch.
- If a task requires reading shared context (e.g., types file used by many components), include it in each agent's scope but count it toward the file limit.

## 6. CSS Rules

- Always use semicolons after every CSS declaration:

```css
.example {
  color: red;
  padding: 10px;
}
```

- If the project uses **Tailwind CSS**, minimize custom CSS as much as possible. Use Tailwind utility classes instead of writing custom styles. Only fall back to custom CSS when Tailwind cannot achieve the desired result.

```vue
<!-- ❌ Incorrect — custom CSS when Tailwind can handle it -->
<div class="custom-card">...</div>
<style scoped>
.custom-card {
  display: flex;
  padding: 16px;
  border-radius: 8px;
  background-color: white;
}
</style>

<!-- ✅ Correct — Tailwind utility classes -->
<div class="flex p-4 rounded-lg bg-white">...</div>
```

- Define all colors as global CSS variables (or Tailwind theme tokens) in a single location. Do NOT hardcode color values anywhere in the codebase — always reference variables instead:

```css
/* ✅ Define colors globally (e.g., in tailwind.config or :root) */
:root {
  --color-primary: #3b82f6;
  --color-secondary: #64748b;
  --color-danger: #ef4444;
}
```

```vue
<!-- ❌ Incorrect — hardcoded color -->
<div class="text-[#3b82f6]">...</div>
<style scoped>
.title { color: #3b82f6; }
</style>

<!-- ✅ Correct — using variable / theme token -->
<div class="text-primary">...</div>
```

## 7. File Storage Rules

- Compress images through [squoosh.app](https://squoosh.app/) or TinyIMG before adding to the project.
- Store images in `public/images`.
- **SVG icons must be wrapped in Vue components.** Do NOT use inline SVG code or `<img src="icon.svg">` directly in templates. Instead, create a `.vue` file in `components/icons/` and use it as a component:

**Folder structure:**

```
components/
└── icons/
    ├── IconArrowLeft.vue
    ├── IconCheck.vue
    ├── IconClose.vue
    ├── IconUser.vue
    └── index.ts
```

```vue
<!-- ❌ Incorrect — inline SVG in template -->
<template>
  <div>
    <svg xmlns="http://www.w3.org/2000/svg" width="24" height="24" viewBox="0 0 24 24">
      <path d="M19 6.41L17.59 5 12 10.59 6.41 5 5 6.41 10.59 12 5 17.59 6.41 19 12 13.41 17.59 19 19 17.59 13.41 12z" />
    </svg>
  </div>
</template>

<!-- ❌ Incorrect — img tag with svg -->
<template>
  <img src="@/assets/icons/close.svg" />
</template>

<!-- ✅ Correct — Vue component -->
<template>
  <div>
    <IconClose />
  </div>
</template>
```

```vue
<!-- components/icons/IconClose.vue -->
<template>
  <svg xmlns="http://www.w3.org/2000/svg" width="1em" height="1em" viewBox="0 0 24 24">
    <path fill="currentColor" d="M19 6.41L17.59 5 12 10.59 6.41 5 5 6.41 10.59 12 5 17.59 6.41 19 12 13.41 17.59 19 19 17.59 13.41 12z" />
  </svg>
</template>
```

- **SVG icons must NOT contain hardcoded colors or sizes.** Replace all hardcoded values:

| Attribute | ❌ Hardcoded | ✅ Correct |
|-----------|-------------|-----------|
| `width` / `height` | `"24"`, `"16"`, `"32"` | `"1em"` |
| `fill` | `"#333"`, `"black"`, `"#FF0000"` | `"currentColor"` |
| `stroke` | `"#333"`, `"black"` | `"currentColor"` |

```vue
<!-- ❌ Incorrect — hardcoded color and size -->
<template>
  <svg width="24" height="24" viewBox="0 0 24 24">
    <path fill="#333333" d="M12 2L2 7l10 5 10-5-10-5z" />
    <path stroke="#000000" d="M2 17l10 5 10-5" />
  </svg>
</template>

<!-- ✅ Correct — inherits color from parent, scales with font-size -->
<template>
  <svg width="1em" height="1em" viewBox="0 0 24 24">
    <path fill="currentColor" d="M12 2L2 7l10 5 10-5-10-5z" />
    <path stroke="currentColor" d="M2 17l10 5 10-5" />
  </svg>
</template>
```

- Using `1em` makes the icon scale with the parent's `font-size` — control size via Tailwind: `text-sm`, `text-lg`, `text-2xl`, etc.
- Using `currentColor` makes the icon inherit the parent's `color` — control color via Tailwind: `text-red-500`, `text-primary`, etc.

```vue
<!-- Usage — size and color controlled by parent -->
<span class="text-2xl text-red-500">
  <IconClose />
</span>

<span class="text-sm text-gray-400">
  <IconCheck />
</span>
```

```ts
// components/icons/index.ts
export { default as IconArrowLeft } from "./IconArrowLeft.vue";
export { default as IconCheck } from "./IconCheck.vue";
export { default as IconClose } from "./IconClose.vue";
export { default as IconUser } from "./IconUser.vue";
```

- Icon component names must start with the `Icon` prefix: `IconClose`, `IconUser`, `IconArrowLeft`.
- If icons are used frequently across the project, register them as global components.
