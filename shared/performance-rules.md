# Performance Rules

> Universal performance best practices for Claude Code. Apply to all projects regardless of tech stack.

## Rendering Optimization

- Avoid unnecessary re-renders. Only update components when their data actually changes.
- Use memoization for expensive computations:

```ts
// Vue — use computed instead of methods for derived state
const filteredUsers = computed(() => users.value.filter((u) => u.isActive));

// React — use useMemo for expensive calculations
const filteredUsers = useMemo(() => users.filter((u) => u.isActive), [users]);
```

- Never perform heavy computation inside render/template — move it to computed properties or memoized values.
- Avoid creating new objects/arrays/functions inside templates on every render:

```ts
// ❌ Incorrect — new function created every render
<button @click="() => handleClick(item.id)">Click</button>

// ✅ Correct — stable reference
<button @click="handleClick(item.id)">Click</button>
```

## List Rendering

- Always use a unique, stable `key` for list items — never use array index:

```vue
<!-- ❌ Incorrect -->
<div v-for="(item, index) in items" :key="index">

<!-- ✅ Correct -->
<div v-for="item in items" :key="item.id">
```

- For large lists (100+ items), implement **virtual scrolling** to render only visible items.
- Paginate API responses — never fetch all records at once.

## Lazy Loading

- Lazy load routes/pages — do NOT load the entire app upfront:

```ts
// Vue Router
const UserProfile = () => import("@/pages/UserProfile.vue");

// React Router
const UserProfile = lazy(() => import("@/pages/UserProfile"));
```

- Lazy load heavy components that are not immediately visible (modals, tabs, below-the-fold content).
- Use `loading="lazy"` for images below the fold.

## Image Optimization

- Compress all images through [squoosh.app](https://squoosh.app/) or TinyIMG before adding to the project.
- Use modern formats: **WebP** or **AVIF** with JPEG/PNG fallbacks.
- Always specify `width` and `height` attributes to prevent layout shifts.
- Implement progressive/placeholder loading for large images.

## Bundle Size

- Avoid importing entire libraries when you only need one function:

```ts
// ❌ Incorrect — imports entire lodash (~70KB)
import _ from "lodash";
const result = _.debounce(fn, 300);

// ✅ Correct — imports only debounce (~1KB)
import debounce from "lodash/debounce";
const result = debounce(fn, 300);
```

- Regularly analyze bundle size: `npm run build -- --analyze`
- Remove unused dependencies and dead code.
- Use dynamic imports for heavy libraries that are not needed on initial load.

## Network Requests

- Implement request **debouncing** for search inputs and filters.
- Cache API responses where appropriate (SWR, React Query, or manual caching).
- Avoid duplicate requests — cancel previous pending requests when a new one is triggered.
- Use **pagination** or **infinite scroll** instead of loading all data at once.

## Memory Management

- Clean up event listeners, intervals, and subscriptions when components unmount:

```ts
// Vue
onMounted(() => {
  window.addEventListener("resize", handleResize);
});
onUnmounted(() => {
  window.removeEventListener("resize", handleResize);
});

// React
useEffect(() => {
  window.addEventListener("resize", handleResize);
  return () => window.removeEventListener("resize", handleResize);
}, []);
```

- Avoid storing large datasets in memory — paginate and fetch on demand.
- Watch for memory leaks in long-lived components (dashboards, real-time feeds).

## CSS Performance

- Avoid deep CSS nesting (more than 3 levels).
- Prefer Tailwind utility classes over complex custom CSS.
- Use `will-change` sparingly and only on elements that actually animate.
- Avoid layout thrashing — batch DOM reads and writes.

## Rules

- Performance is a feature, not an afterthought — consider it from the start.
- Measure before optimizing — use browser DevTools, Lighthouse, and profiling tools.
- Set performance budgets: initial load under 3 seconds, bundle size under target.
- Test on low-end devices and slow network connections.
- Never sacrifice code readability for micro-optimizations unless profiling proves necessity.
