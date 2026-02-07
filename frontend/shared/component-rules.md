# Component Rules

> Universal component architecture rules for frontend projects. Apply to all frameworks (Vue, React, Svelte, Angular, etc.).

## Single Responsibility

- **Keep components small and single-purpose.** Each component should handle one clear responsibility.
- If a component grows too large or handles multiple concerns, break it down into smaller, focused child components:

```
❌ Incorrect — one large component doing everything
UserDashboard (profile + stats + activity + settings all in one)

✅ Correct — split into focused components
UserDashboard
├── UserProfileCard
├── UserStatsPanel
├── UserActivityFeed
└── UserSettingsForm
```

## Page / View Components

- **Page files must be thin wrappers only.** Do NOT write logic, markup, or styles directly inside page/view files.
- All logic, template markup, and composables/hooks must live in a `components/` folder next to the page file.
- The page file should only import and render the main component:

```
pages/
└── auth/
    ├── login.vue (or login.tsx)     ← Thin wrapper ONLY
    └── components/
        ├── LoginPage.vue (or .tsx)  ← All markup here
        ├── useLoginPage.ts          ← All logic here
        ├── LoginForm.vue
        ├── LoginSocialButtons.vue
        └── index.ts
```

- This applies to **every** page/view in the project — no exceptions.
- Benefits: keeps routing files clean, makes page logic testable, enables component reuse, follows single-responsibility principle.

## Use UI Library Components

- If the project uses a UI library (e.g., Vuetify, Element Plus, PrimeVue, Ant Design, Chakra UI, Shadcn), you **must** use its components exclusively.
- Do NOT create custom HTML tags or wrapper elements that duplicate existing library components:

```
❌ Incorrect — custom element when UI library has the component
<custom-button @click="submit">Save</custom-button>
<div class="custom-input-wrapper">
  <input type="text" />
</div>

✅ Correct — use the UI library component
<UButton @click="submit">Save</UButton>
<UInput v-model="name" />
```

- Before creating a new shared component, check if the UI library already provides one.
- If the UI library lacks a component, check with the team before creating a custom one.

## Global Component Registration

- Register frequently used elements (button, modal, input, icon) as global components to avoid repetitive imports:

```ts
// ❌ Incorrect — importing the same component in dozens of files
import BaseButton from "@/shared/ui/BaseButton";
import BaseInput from "@/shared/ui/BaseInput";

// ✅ Correct — register once globally
app.component("BaseButton", BaseButton);
app.component("BaseInput", BaseInput);
app.component("BaseModal", BaseModal);
```

- Only promote components that are truly reused project-wide — keep the global registry clean.

## Component Naming

- Component names must be **multi-word** and descriptive.
- Specify what the component does and which module it belongs to:

```
❌ Incorrect
Modal
Card
Button
List

✅ Correct
CashbackOpenModal
UserProfileCard
BaseButton (shared/global)
CandidateList
```

- Icon components start with `Icon` prefix: `IconClose`, `IconArrowLeft`.
- Page wrapper components include the page context: `LoginPage`, `DashboardPage`.

## Remove Unnecessary Wrappers

- Remove root wrapper elements (`<div>`, `<section>`) if they are unnecessary.
- Remove the `<style>` block if the component has no custom styles — do not leave empty style blocks.

## Rules

- Keep components small and single-purpose — split when complexity grows.
- Page files are thin wrappers only — all logic in `components/` folder.
- Use UI library components exclusively — never duplicate library functionality.
- Register frequently used components globally.
- Component names must be multi-word and descriptive.
- Remove unnecessary wrappers and empty style blocks.
