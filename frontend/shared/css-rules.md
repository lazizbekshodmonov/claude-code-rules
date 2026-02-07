# CSS Rules

> Universal CSS rules for frontend projects. Apply to all frameworks (Vue, React, Svelte, Angular, etc.).

## Semicolons

- Always use semicolons after every CSS declaration:

```css
/* ❌ Incorrect */
.example {
  color: red
  padding: 10px
}

/* ✅ Correct */
.example {
  color: red;
  padding: 10px;
}
```

## Tailwind CSS

- If the project uses **Tailwind CSS**, minimize custom CSS as much as possible.
- Use Tailwind utility classes instead of writing custom styles:

```html
<!-- ❌ Incorrect — custom CSS when Tailwind can handle it -->
<div class="custom-card">...</div>
<style>
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

- Only fall back to custom CSS when Tailwind cannot achieve the desired result.

## Colors and Design Tokens

- Define all colors as global CSS variables or Tailwind theme tokens in a single location.
- Do NOT hardcode color values anywhere in the codebase:

```css
/* ✅ Define colors globally (e.g., in tailwind.config or :root) */
:root {
  --color-primary: #3b82f6;
  --color-secondary: #64748b;
  --color-danger: #ef4444;
}
```

```html
<!-- ❌ Incorrect — hardcoded color -->
<div class="text-[#3b82f6]">...</div>
<style>
.title { color: #3b82f6; }
</style>

<!-- ✅ Correct — using variable / theme token -->
<div class="text-primary">...</div>
```

- All colors must come from the Design System. If a color is missing, ask the designers to update the mockup.

## Nesting

- Avoid deep CSS nesting — maximum 3 levels:

```css
/* ❌ Incorrect — too deep */
.page .sidebar .nav .item .link .icon {
  color: red;
}

/* ✅ Correct — flat or shallow nesting */
.nav-item-icon {
  color: red;
}
```

## Scoped Styles

- Always use scoped styles (Vue `scoped`, CSS Modules, Styled Components) unless global styles are explicitly needed.
- Prefer Tailwind classes in markup over writing custom CSS.
- Remove the `<style>` block entirely if the component has no custom styles — do not leave empty style blocks.

## Animation Performance

- Use `will-change` sparingly and only on elements that actually animate.
- Avoid layout thrashing — batch DOM reads and writes.
- Use `prefers-reduced-motion` for users sensitive to animation:

```css
@media (prefers-reduced-motion: reduce) {
  * {
    animation: none !important;
    transition: none !important;
  }
}
```

## Rules

- Always use semicolons in CSS declarations.
- Prefer Tailwind utility classes over custom CSS.
- Never hardcode color values — use CSS variables or theme tokens.
- Avoid deep CSS nesting (max 3 levels).
- Always use scoped styles unless global styles are needed.
- Remove empty style blocks.
- Use `prefers-reduced-motion` for accessibility.
