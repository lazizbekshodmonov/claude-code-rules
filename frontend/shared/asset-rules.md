# Asset Rules

> Universal rules for managing images, icons, and static files in frontend projects.

## Image Optimization

- Compress all images through [squoosh.app](https://squoosh.app/) or TinyIMG before adding to the project.
- Use modern formats: **WebP** or **AVIF** with JPEG/PNG fallbacks.
- Always specify `width` and `height` attributes to prevent layout shifts.
- Use `loading="lazy"` for images below the fold.
- Store images in `public/images`.

## SVG Icons as Components

- **SVG icons must be wrapped in framework components.** Do NOT use inline SVG code or `<img src="icon.svg">` directly in templates:

```
components/
└── icons/
    ├── IconArrowLeft.vue (or .tsx)
    ├── IconCheck.vue
    ├── IconClose.vue
    ├── IconUser.vue
    └── index.ts
```

```html
<!-- ❌ Incorrect — inline SVG in template -->
<template>
  <svg xmlns="http://www.w3.org/2000/svg" width="24" height="24" viewBox="0 0 24 24">
    <path d="M19 6.41L17.59 5 12 ..." />
  </svg>
</template>

<!-- ❌ Incorrect — img tag with SVG -->
<img src="@/assets/icons/close.svg" />

<!-- ✅ Correct — framework component -->
<IconClose />
```

## SVG Icon Standards

- SVG icons must **NOT** contain hardcoded colors or sizes. Replace all hardcoded values:

| Attribute | Hardcoded | Correct |
|-----------|-----------|---------|
| `width` / `height` | `"24"`, `"16"`, `"32"` | `"1em"` |
| `fill` | `"#333"`, `"black"`, `"#FF0000"` | `"currentColor"` |
| `stroke` | `"#333"`, `"black"` | `"currentColor"` |

```html
<!-- ❌ Incorrect — hardcoded color and size -->
<svg width="24" height="24" viewBox="0 0 24 24">
  <path fill="#333333" d="M12 2L2 7l10 5 10-5-10-5z" />
  <path stroke="#000000" d="M2 17l10 5 10-5" />
</svg>

<!-- ✅ Correct — inherits color and scales with font-size -->
<svg width="1em" height="1em" viewBox="0 0 24 24">
  <path fill="currentColor" d="M12 2L2 7l10 5 10-5-10-5z" />
  <path stroke="currentColor" d="M2 17l10 5 10-5" />
</svg>
```

- `1em` makes the icon scale with the parent's `font-size` — control size via CSS/Tailwind: `text-sm`, `text-lg`, `text-2xl`.
- `currentColor` makes the icon inherit the parent's `color` — control color via CSS/Tailwind: `text-red-500`, `text-primary`.

```html
<!-- Usage — size and color controlled by parent -->
<span class="text-2xl text-red-500">
  <IconClose />
</span>

<span class="text-sm text-gray-400">
  <IconCheck />
</span>
```

## Icon Naming

- Icon component names must start with the `Icon` prefix: `IconClose`, `IconUser`, `IconArrowLeft`.
- Export all icons from `components/icons/index.ts`:

```ts
export { default as IconArrowLeft } from "./IconArrowLeft.vue";
export { default as IconCheck } from "./IconCheck.vue";
export { default as IconClose } from "./IconClose.vue";
export { default as IconUser } from "./IconUser.vue";
```

- If icons are used frequently across the project, register them as global components.

## Rules

- Compress all images before adding to the project.
- Use modern image formats (WebP/AVIF) with fallbacks.
- Always specify `width` and `height` on images to prevent layout shifts.
- Wrap SVG icons in framework components — never use inline SVG or `<img>`.
- Use `currentColor` and `1em` in all SVG icons.
- Icon components start with the `Icon` prefix.
