---
id: 6a0fc38ff59d5
name: JS/TS Programming
tags: []
updated_at: 2026-05-22T03:12:39.898563Z
---

## Type Safety (TS)

| Tip | Why it matters |
|:-----------|---------------:|
| Prefer `unknown` over `any` | Forces you to narrow the type before use |
| Use `strict: true` in `tsconfig.json` | Catches null/undefined bugs at compile time |
| Prefer `interface` for public APIs, `type` for unions/intersections | Interfaces are extendable; types are more composable |
| Use `as const` on literal objects/arrays | Narrows inferred types to their exact values |

## Async & Error Handling

```ts
// Prefer async/await over .then chains
const data = await fetchUser(id).catch((e) => { throw new AppError(e) });

// Never swallow errors silently
try { ... } catch (e) { logger.error(e); throw e; }
```

- Always `await` inside `try/catch` — unhandled promise rejections crash Node
- Use `Promise.all` for concurrent independent calls, not sequential `await`s

## JS Gotchas to Avoid

| Gotcha | Fix |
|--------|---------:|
| coercion surprises | Always use `===` |
| `typeof null === "object"` | Check `value === null` explicitly |
| Mutating function arguments | Clone with `structuredClone()` or spread |
| `var` hoisting bugs | Use `const` by default, `let` only when reassigning |

## Performance & Patterns

- Prefer `const` — signals intent and enables engine optimizations
- Use `Map`/`Set` over plain objects for frequent key lookups
- Avoid large synchronous loops on the main thread — break with `setTimeout` or `queueMicrotask`

## DevTools Tricks

**Inspect tooltip / hover-only CSS** — the tooltip disappears when you move to DevTools, so freeze it first:

```js
// In the browser console: hover over the element, then run:
setTimeout(() => { debugger; }, 3000)
// Switch back to page, trigger the tooltip within 3s — debugger pauses JS,
// keeping the tooltip visible. Now open DevTools Elements panel and inspect.
```

## Tooling Defaults

```bash
npx tsc --noEmit          # type-check without building
node --watch index.js     # built-in watch mode (Node 18+)
```

TAGS: javascript, typescript, tips, async, types
