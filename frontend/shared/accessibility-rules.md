# Accessibility (a11y) Rules

> Universal accessibility rules for Claude Code. Apply to all web projects regardless of tech stack.

## Semantic HTML

- Use semantic HTML elements instead of generic `<div>` and `<span>`:

| Instead of | Use |
|-----------|-----|
| `<div>` for navigation | `<nav>` |
| `<div>` for main content | `<main>` |
| `<div>` for sidebar | `<aside>` |
| `<div>` for footer | `<footer>` |
| `<div>` for article | `<article>` |
| `<div>` for section | `<section>` |
| `<div>` as button | `<button>` |
| `<div>` with click handler | `<button>` or `<a>` |

```html
<!-- ❌ Incorrect -->
<div class="nav">
  <div class="nav-item" @click="navigate">Home</div>
</div>

<!-- ✅ Correct -->
<nav>
  <a href="/" @click.prevent="navigate">Home</a>
</nav>
```

## Images

- **Every** `<img>` must have a meaningful `alt` attribute:

```html
<!-- ❌ Incorrect -->
<img src="user-avatar.jpg" />
<img src="user-avatar.jpg" alt="image" />

<!-- ✅ Correct -->
<img src="user-avatar.jpg" alt="John Doe's profile photo" />

<!-- Decorative images — empty alt -->
<img src="divider.svg" alt="" role="presentation" />
```

## Forms

- Every form input must have an associated `<label>`:

```html
<!-- ❌ Incorrect — no label -->
<input type="email" placeholder="Enter email" />

<!-- ✅ Correct — explicit label -->
<label for="email">Email</label>
<input id="email" type="email" placeholder="Enter email" />
```

- Use `aria-required` or `required` attribute for mandatory fields.
- Provide clear error messages linked to the input:

```html
<label for="email">Email</label>
<input id="email" type="email" aria-describedby="email-error" />
<span id="email-error" role="alert">Please enter a valid email address</span>
```

## Keyboard Navigation

- All interactive elements must be keyboard accessible (focusable and operable with Enter/Space).
- Use logical tab order — do NOT use positive `tabindex` values:

```html
<!-- ❌ Incorrect — breaks natural tab order -->
<button tabindex="5">First</button>
<button tabindex="1">Second</button>

<!-- ✅ Correct — natural DOM order -->
<button>First</button>
<button>Second</button>
```

- Provide visible focus indicators — never remove outline without a replacement:

```css
/* ❌ Incorrect */
*:focus { outline: none; }

/* ✅ Correct */
*:focus-visible {
  outline: 2px solid var(--color-primary);
  outline-offset: 2px;
}
```

- Implement keyboard shortcuts for modals: Escape to close, trap focus inside.

## ARIA Attributes

- Use ARIA attributes only when semantic HTML is insufficient.
- Common ARIA patterns:

```html
<!-- Loading state -->
<div aria-busy="true" aria-live="polite">Loading...</div>

<!-- Modal dialog -->
<div role="dialog" aria-modal="true" aria-labelledby="modal-title">
  <h2 id="modal-title">Confirm Action</h2>
</div>

<!-- Expandable section -->
<button aria-expanded="false" aria-controls="panel-1">Show Details</button>
<div id="panel-1" hidden>Details content here</div>
```

- Do NOT use `role="button"` on `<div>` — use a `<button>` element instead.

## Color and Contrast

- Maintain minimum color contrast ratio:
  - Normal text: **4.5:1**
  - Large text (18px+ or 14px+ bold): **3:1**
- Never rely on color alone to convey information:

```html
<!-- ❌ Incorrect — color is the only indicator -->
<span class="text-red-500">Error</span>

<!-- ✅ Correct — icon + color + text -->
<span class="text-red-500">⚠️ Error: Invalid email address</span>
```

## Dynamic Content

- Use `aria-live` for dynamically updated content:

```html
<!-- Notifications, toasts, status updates -->
<div aria-live="polite">3 new messages</div>

<!-- Urgent alerts -->
<div aria-live="assertive" role="alert">Session expired. Please log in again.</div>
```

## Rules

- Accessibility is not optional — it is a requirement for every feature.
- Test with keyboard navigation — every flow must work without a mouse.
- Test with screen readers (VoiceOver, NVDA) for critical user flows.
- Run automated accessibility checks: axe DevTools, Lighthouse accessibility audit.
- Use `prefers-reduced-motion` media query for users who are sensitive to animation:

```css
@media (prefers-reduced-motion: reduce) {
  * { animation: none !important; transition: none !important; }
}
```
