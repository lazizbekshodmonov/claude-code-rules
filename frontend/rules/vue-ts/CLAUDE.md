# Code Writing Rules (Vue + TypeScript)

> Vue-specific rules for projects using Vue 3 + TypeScript + Composition API.
>
> Want to contribute? See the [Contributing guide](../../../README.md#contributing).

## Shared Rules

Include the following shared rule files in your project's `CLAUDE.md`:

### Universal (All Stacks)

- [Agent Rules](../../../shared/agent-rules.md)
- [Git Rules](../../../shared/git-rules.md)
- [Planning Rules](../../../shared/planning-rules.md)

### Frontend Shared

- [Naming Rules](../../shared/naming-rules.md)
- [TypeScript Rules](../../shared/typescript-rules.md)
- [Component Rules](../../shared/component-rules.md)
- [API Rules](../../shared/api-rules.md)
- [Testing Rules](../../shared/testing-rules.md)
- [Documentation Rules](../../shared/documentation-rules.md)
- [Code Review Rules](../../shared/code-review-rules.md)
- [Performance Rules](../../shared/performance-rules.md)
- [Security Rules](../../shared/security-rules.md)
- [Accessibility Rules](../../shared/accessibility-rules.md)
- [CSS Rules](../../shared/css-rules.md)
- [Asset Rules](../../shared/asset-rules.md)
- [Code Organization Rules](../../shared/code-organization-rules.md)

---

## 1. General Vue Rules

- Do NOT use native DOM methods to access elements, check classes, or other attributes. Vue provides enough functionality to handle various behaviors without direct DOM access. Use Vue refs instead.

## 2. Emit Naming

- Event names in interfaces with the `Emits` suffix must use camelCase (per [Vue docs](https://vuejs.org/guide/components/events)):

```ts
// ❌ Incorrect
interface Emits {
  (event: "do-something"): void;
}

// ✅ Correct
interface Emits {
  (event: "doSomething"): void;
}
```

## 3. Avoid Watch

- Avoid using `watch` whenever possible. Reasons:
  - `watch` reacts to any state change. If a side effect should only trigger under specific conditions, you end up writing workarounds inside the watcher.
  - `watch` creates implicit links between state and side effects, making application logic harder to trace.
  - Multiple watchers act as independent entry points, complicating data flow understanding.
- **Preferred approach:** Use explicit state update methods as single entry points for all changes. Favor the imperative approach ("call a specific method → controlled state change") over the reactive one ("something changed → automatic reaction").

## 4. Component Structure

- Props/Emits interfaces and the main composable must be named consistently with the file names:

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
<!-- some-component/SomeComponent.vue -->
<script lang="ts" setup>
import { SomeComponentProps, SomeComponentEmits } from "./some-component.types";
import useSomeComponent from "./useSomeComponent";

const props = defineProps<SomeComponentProps>();
const emit = defineEmits<SomeComponentEmits>();

const {} = useSomeComponent(props, emit);
</script>
```

```ts
// some-component/index.ts — always use a named alias
export { default as SomeComponent } from "./SomeComponent.vue";
```

- Extract **ALL** component logic into the main composable. The `<script>` tag should contain only macro usage (`defineProps`, `defineEmits`) and the composable call — no other logic.

## 5. Component Block Comment

- **Every component must have a detailed block comment** at the very top of the `<script>` tag:

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
import { SomeComponentProps, SomeComponentEmits } from "./some-component.types";
import useSomeComponent from "./useSomeComponent";

const props = defineProps<SomeComponentProps>();
const emit = defineEmits<SomeComponentEmits>();

const {} = useSomeComponent(props, emit);
</script>
```

- Must include: `@description`, `@props`, `@emits`, `@usage`.

## 6. Composable Code Order

Code order inside a composable file:

1. Use-function calls and primitive constants (`use*`, `const`)
2. `ref` / `reactive`
3. `computed`
4. Arrow functions, declarative functions
5. Vue lifecycle hooks (`onMounted`, `onUnmounted`, etc.)

## 7. Page / View Files

- Page files must be **thin wrappers only**. All logic, template, and composables live in `components/` next to the page:

```
pages/
└── auth/
    ├── login.vue                  ← Thin wrapper ONLY
    └── components/
        ├── LoginPage.vue          ← All HTML / template
        ├── login-page.types.ts
        ├── useLoginPage.ts        ← All logic
        ├── __tests__/
        │   └── LoginPage.spec.ts
        ├── LoginForm.vue
        ├── LoginSocialButtons.vue
        └── index.ts
```

```vue
<!-- pages/auth/login.vue — thin wrapper -->
<script lang="ts" setup>
import { LoginPage } from "./components";
</script>

<template>
  <LoginPage />
</template>
```

## 8. UI Library Components

- If the project uses a UI library (e.g., `uzum-ui`, Vuetify, Element Plus, PrimeVue, Naive UI), use its components exclusively.
- Use lightweight `UFlex` instead of `USpace`, but if you just need a wrapper, use a plain `<div>`.
- Avoid creating new shared components in `shared/components` or `shared/UI`. Look for existing local solutions or components in the UI library first.

## 9. Vue Formatting Rules

### 9.1. Prettier Config

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

Key rules:
- Always use **double quotes** (`"`)
- Max line width for `<script>` and `<style>`: **200 characters**
- `vueIndentScriptAndStyle: true` — all code inside `<script>` and `<style>` indented 2 spaces from the tag

### 9.2. ESLint Template Rules (printWidth: 100)

Since Prettier cannot set a separate `printWidth` for `<template>`, the following `eslint-plugin-vue` rules enforce a **max line width of ~100 characters** in template blocks:

```js
{
  "rules": {
    "vue/max-attributes-per-line": ["error", {
      "singleline": { "max": 3 },
      "multiline": { "max": 1 }
    }],
    "vue/html-closing-bracket-newline": ["error", {
      "singleline": "never",
      "multiline": "always"
    }],
    "vue/first-attribute-linebreak": ["error", {
      "singleline": "ignore",
      "multiline": "below"
    }],
    "vue/max-len": ["error", {
      "code": 200,
      "template": 100,
      "ignoreUrls": true,
      "ignoreStrings": false,
      "ignoreTemplateLiterals": true,
      "ignoreHTMLAttributeValues": false,
      "ignoreHTMLTextContents": true
    }],
    "vue/html-self-closing": ["error", {
      "html": { "void": "always", "normal": "never", "component": "always" },
      "svg": "always",
      "math": "always"
    }],
    "vue/html-indent": ["error", 2, {
      "attribute": 1,
      "baseIndent": 1,
      "closeBracket": 0,
      "alignAttributesVertically": true
    }]
  }
}
```

**Line width per block:**

| Block | Max Width | Enforced By |
|-------|-----------|-------------|
| `<script>` | 200 chars | Prettier |
| `<template>` | 100 chars | ESLint (`vue/max-len`) |
| `<style>` | 200 chars | Prettier |

### 9.3. `<script>` Block Rules

- Always use `<script setup lang="ts">`.
- `vueIndentScriptAndStyle: true` — indent all code 2 spaces from the `<script>` tag:

```vue
<!-- ❌ Incorrect — no indentation from script tag -->
<script setup lang="ts">
import { ref } from "vue";
const count = ref(0);
</script>

<!-- ✅ Correct — indented 2 spaces -->
<script setup lang="ts">
  import { ref } from "vue";
  const count = ref(0);
</script>
```

- Code order inside `<script>`:
  1. Component JSDoc comment
  2. Type imports (`import type { ... }`)
  3. Module imports (`import { ... }`)
  4. `defineProps` / `defineEmits` macros
  5. Composable destructuring
  6. No other logic — everything belongs in the composable

### 9.4. `<template>` Block Rules

- **Max line width: 100 characters** (ESLint enforced).
- Template content starts at zero indentation from the `<template>` tag.
- Use **Tailwind CSS classes** instead of custom CSS wherever possible.
- Use **UI library components** instead of native HTML tags.
- Attribute ordering on elements:
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
<n-button
  v-if="canSubmit"
  type="primary"
  size="large"
  :loading="isLoading"
  @click="submit"
>
  Submit
</n-button>
```

- Multi-attribute elements: if an element has **3+ attributes**, place each on its own line.

### 9.5. `<style>` Block Rules

- `vueIndentScriptAndStyle: true` — indent all CSS 2 spaces from the `<style>` tag:

```vue
<!-- ✅ Correct -->
<style scoped>
  .container {
    display: flex;
  }
</style>
```

- Always use `scoped` unless global styles are explicitly needed.
- Prefer Tailwind classes in `<template>` over custom CSS.
- Remove `<style>` tag entirely if the component has no custom styles.

## 10. Testing

- Test files must be co-located with the source file:

```
some-component/
├── SomeComponent.vue
├── useSomeComponent.ts
├── some-component.types.ts
├── __tests__/
│   ├── SomeComponent.spec.ts
│   ├── useSomeComponent.spec.ts
│   └── SomeComponent.integration.spec.ts
└── index.ts
```

- **Unit tests** must cover:
  - Component rendering with different props
  - Emit events and user interactions
  - Composable return values and state changes

- **Integration tests** must cover:
  - Component interaction with child components
  - API call flows (mocked)
  - Store / state management integration

- After writing tests, always run them:

```bash
npm run test
npm run test -- --filter SomeComponent
npm run test -- --coverage
```
