# Karpathy behavioral guidelines

Behavioral guidelines to reduce common LLM coding mistakes. Merge with project-specific instructions as needed.

**Tradeoff:** These guidelines bias toward caution over speed. For trivial tasks, use judgment.

## 1. Think Before Coding

**Don't assume. Don't hide confusion. Surface tradeoffs.**

Before implementing:
- State your assumptions explicitly. If uncertain, ask.
- If multiple interpretations exist, present them - don't pick silently.
- If a simpler approach exists, say so. Push back when warranted.
- If something is unclear, stop. Name what's confusing. Ask.

## 2. Simplicity First

**Minimum code that solves the problem. Nothing speculative.**

- No features beyond what was asked.
- No abstractions for single-use code.
- No "flexibility" or "configurability" that wasn't requested.
- No error handling for impossible scenarios.
- If you write 200 lines and it could be 50, rewrite it.

Ask yourself: "Would a senior engineer say this is overcomplicated?" If yes, simplify.

## 3. Surgical Changes

**Touch only what you must. Clean up only your own mess.**

When editing existing code:
- Don't "improve" adjacent code, comments, or formatting.
- Don't refactor things that aren't broken.
- Match existing style, even if you'd do it differently.
- If you notice unrelated dead code, mention it - don't delete it.

When your changes create orphans:
- Remove imports/variables/functions that YOUR changes made unused.
- Don't remove pre-existing dead code unless asked.

The test: Every changed line should trace directly to the user's request.

## 4. Goal-Driven Execution

**Define success criteria. Loop until verified.**

Transform tasks into verifiable goals:
- "Add validation" → "Write tests for invalid inputs, then make them pass"
- "Fix the bug" → "Write a test that reproduces it, then make it pass"
- "Refactor X" → "Ensure tests pass before and after"

For multi-step tasks, state a brief plan:
```
1. [Step] → verify: [check]
2. [Step] → verify: [check]
3. [Step] → verify: [check]
```

Strong success criteria let you loop independently. Weak criteria ("make it work") require constant clarification.

---

**These guidelines are working if:** fewer unnecessary changes in diffs, fewer rewrites due to overcomplication, and clarifying questions come before implementation rather than after mistakes.

---

# Next.js Rules

## Always read docs before coding

Before any Next.js work, find and read the relevant doc in `node_modules/next/dist/docs/`. Your training data is outdated — the docs are the source of truth.

## Project Rules

- Always use bun, not npm

## Next.js Best Practices

Apply these rules when writing or reviewing Next.js code.

> **Cache Components patterns**: When the project has `cacheComponents: true` in `next.config.ts`, use `'use cache'`, `cacheLife()`, `cacheTag()`, `updateTag()`, and `revalidateTag()` for caching.

### File Conventions

- Special files: `page.tsx`, `layout.tsx`, `loading.tsx`, `error.tsx`, `not-found.tsx`, `global-error.tsx`, `route.ts`, `template.tsx`, `default.tsx`
- Route segments: `[slug]` dynamic, `[...slug]` catch-all, `[[...slug]]` optional catch-all, `(group)` route group (no URL impact)
- Parallel routes: `@slot` folders; intercepting routes: `(.)`, `(..)`, `(...)` conventions
- In Next.js 16+, `middleware.ts` → `proxy.ts` (export `proxy()` instead of `middleware()`)

### RSC Boundaries

| Pattern | Valid? | Fix |
|---------|--------|-----|
| `'use client'` + `async function` | No | Fetch in server parent, pass data |
| Pass `() => {}` to client | No | Define in client or use server action |
| Pass `new Date()` to client | No | Use `.toISOString()` |
| Pass `new Map()`/`Set`/class instance to client | No | Convert to plain object/array |
| Pass server action to client | Yes | — |
| Pass `string/number/boolean/plain object/array` | Yes | — |

### Async Patterns (Next.js 15+)

Always type `params` and `searchParams` as `Promise<...>` and `await` them:

```tsx
export default async function Page({
  params,
  searchParams,
}: {
  params: Promise<{ slug: string }>
  searchParams: Promise<{ query?: string }>
}) {
  const { slug } = await params
  const { query } = await searchParams
}
```

Same for `cookies()` and `headers()` — always `await` them. Migration codemod: `npx @next/codemod@latest next-async-request-api .`

### Data Patterns

| Pattern | Use Case | Notes |
|---------|----------|-------|
| Server Component fetch | Internal reads | Direct DB access, no API needed |
| Server Action | Mutations (POST/PUT/DELETE) | POST only, no HTTP caching |
| Route Handler | External APIs, webhooks | GET can be cached |
| Pass from Server → Client | Client reads | Preferred over client-side fetch |

**Avoid data waterfalls** — use `Promise.all` for parallel fetches, or Suspense streaming:

```tsx
// Good: parallel
const [user, posts] = await Promise.all([getUser(), getPosts()])

// Good: preload pattern
preloadUser(id)          // fire-and-forget
const user = await getUser(id)  // likely ready
```

### Error Handling

- `error.tsx` must be a Client Component (`'use client'`)
- `global-error.tsx` must include `<html>` and `<body>` tags
- **Do NOT wrap `redirect()` / `notFound()` / `forbidden()` / `unauthorized()` in try-catch** — they throw internally. Call them outside try-catch or use `unstable_rethrow(error)`:

```tsx
import { unstable_rethrow } from 'next/navigation'
async function action() {
  try {
    redirect('/success')
  } catch (error) {
    unstable_rethrow(error) // re-throws Next.js internal errors
    return { error: 'Something went wrong' }
  }
}
```

### Hydration Errors

Common causes and fixes:
- **Browser-only APIs** (`window`, `document`): use a `mounted` state with `useEffect`
- **Date/time**: render on client only via `useEffect`
- **Random values / IDs**: use `useId()` hook instead of `Math.random()`
- **Invalid HTML nesting**: no `<div>` inside `<p>`, no nested `<p>`
- **Third-party scripts**: use `next/script` with `strategy="afterInteractive"`

### Suspense Boundaries

Wrap `useSearchParams()` and `usePathname()` consumers in `<Suspense>` to avoid CSR bailout.

### Other Rules

- **Runtime:** Default to Node.js runtime. Use Edge runtime only when low latency at the edge is explicitly required.
- **Images:** Always `next/image` over `<img>`. Configure `remotePatterns` for external images. Set `priority` on LCP images.
- **Fonts:** Always `next/font`. Never `<link>` tags.
- **Directives:** Use `'use client'` and `'use server'` correctly.

---

# Next.js + shadcn/ui

Build distinctive, production-grade interfaces that avoid generic "AI slop" aesthetics.

## Core Principles

1. **Minimize noise** — Icons communicate; excessive labels don't
2. **No generic AI-UI** — Avoid purple gradients, excessive shadows, predictable layouts
3. **Context over decoration** — Every element serves a purpose
4. **Theme consistency** — Use CSS variables from `globals.css`, never hardcode colors

## Quick Start

```bash
bunx --bun shadcn@latest init -t next
```

## Component Rules

- `"use client"` only at leaf components (smallest boundary)
- Props must be serializable (data or Server Actions, no functions/classes)
- Pass server content via `children`
- Always use `@/` alias (e.g., `@/lib/utils`) instead of relative paths
- Use `cn()` from `@/lib/utils` for style merging

## File Organization

```
app/
├── (protected)/        # Auth required routes
│   ├── dashboard/
│   ├── settings/
│   ├── components/     # Route-specific components
│   └── lib/            # Route-specific utils/types
├── (public)/           # Public routes
├── actions/            # Server Actions (global)
├── api/                # API routes
├── layout.tsx
└── globals.css         # Theme tokens
components/
├── ui/                 # shadcn primitives
└── shared/             # Business components
hooks/                  # Custom React hooks
lib/                    # Shared utils
data/                   # Database queries
ai/                     # AI logic (tools, agents, prompts)
```

## Next.js 16 Key Rules

- **Async params:** Always `await params` and `searchParams` — they are Promises in Next.js 15+
- **Server Actions = mutations only** (create, update, delete) — never use them for data fetching
- **Data fetching** = Server Components or `'use cache'` functions
- **`proxy.ts`** replaces `middleware.ts` in Next.js 16 — place at project root
- **Caching:** Use `cacheTag()` + `cacheLife()` inside `"use cache"` functions; invalidate with `updateTag()` / `revalidateTag()`

## Package Manager

Always use `bun`, never `npm` or `npx`:
- `bun install`, `bun add`, `bunx --bun`

---

# React Best Practices (Vercel)

Comprehensive performance optimization guide — 40+ rules across 8 categories. Apply when writing, reviewing, or refactoring React/Next.js code.

> If the project has **React Compiler** enabled, manual `memo()`, `useMemo()`, and `useCallback()` optimizations are largely unnecessary. The compiler handles them automatically.

## 1. Eliminating Waterfalls — CRITICAL

- **Check cheap conditions before async flags**: evaluate cheap sync guards before expensive `await` calls to skip them entirely
- **Defer `await` until needed**: move `await` into the branch where it's actually used — don't block early-return paths
- **`Promise.all()` for independent operations**: never `await` sequential fetches when they're independent
  ```ts
  // Bad: 3 round trips
  const user = await fetchUser()
  const posts = await fetchPosts()
  const comments = await fetchComments()

  // Good: 1 round trip
  const [user, posts, comments] = await Promise.all([fetchUser(), fetchPosts(), fetchComments()])
  ```
- **Start promises early in API routes**: kick off independent fetches immediately, `await` later
  ```ts
  const sessionPromise = auth()
  const configPromise = fetchConfig()
  const session = await sessionPromise
  const [config, data] = await Promise.all([configPromise, fetchData(session.user.id)])
  ```
- **Chain nested fetches per-item**: `chatIds.map(id => getChat(id).then(chat => getUser(chat.author)))` — don't await the outer batch before starting inner fetches
- **Strategic Suspense boundaries**: wrap data-dependent children in `<Suspense>` so the outer shell renders immediately while data streams in

## 2. Bundle Size Optimization — CRITICAL

- **Avoid barrel file imports**: prefer `optimizePackageImports` in Next.js (already configured for common libraries). Direct deep imports for non-Next.js projects
- **`next/dynamic` for heavy components**: lazy-load components not needed on initial render (e.g., Monaco Editor, chart libraries)
  ```ts
  const MonacoEditor = dynamic(() => import('./monaco-editor').then(m => m.MonacoEditor), { ssr: false })
  ```
- **Defer non-critical third parties**: load analytics/logging after hydration with `dynamic(..., { ssr: false })`
- **Conditional module loading**: use `import()` inside `useEffect` gated on a feature flag, not at top level
- **Statically analyzable import paths**: use explicit maps of `() => import(...)` calls — avoid dynamic path construction that widens bundle traces
- **Preload on hover/focus**: trigger `void import('./heavy-module')` on `onMouseEnter`/`onFocus` for perceived speed

## 3. Server-Side Performance — HIGH

- **Authenticate every Server Action**: treat them like public API endpoints — always verify session inside the action, never rely on middleware alone
- **`React.cache()` for per-request deduplication**: wrap DB queries and auth checks so multiple component renderings share one result
  ```ts
  export const getCurrentUser = cache(async () => {
    const session = await auth()
    return session ? db.user.findUnique({ where: { id: session.user.id } }) : null
  })
  ```
- **No mutable module-level state for request data**: concurrent RSC renders share the same process — pass data via props, not shared variables
- **Hoist static I/O to module level**: load fonts, logos, and config files once at module init, not per request
- **Minimize RSC→Client serialization**: only pass fields the client actually uses — do `.filter()`/`.map()` on the client side to avoid duplicate serialization
- **Parallel component composition**: extract fetches into sibling Server Components so they run concurrently
- **`after()` for non-blocking work**: schedule analytics, logging, and cache invalidation after the response is sent

## 4. Client-Side Data Fetching — MEDIUM-HIGH

- **SWR for deduplication**: multiple component instances sharing the same `useSWR(key)` make only one request
- **Pass data from Server Components** to Client Components instead of fetching on mount when possible
- **Passive event listeners**: add `{ passive: true }` to `touchstart`/`wheel` listeners that don't call `preventDefault()`
- **Version localStorage keys**: prefix with `v2:`, store only needed fields, always wrap in try-catch (fails in Safari private mode)

## 5. Re-render Optimization — MEDIUM

- **Derive state during render**: if a value can be computed from props/state, don't store it in `useState` or sync it via `useEffect`
- **Don't define components inside components**: creates a new component type every render → React unmounts and remounts, losing all state
- **Use functional `setState` updates**: `setItems(curr => [...curr, newItem])` — eliminates stale closure bugs and stable `useCallback` dependencies
- **Narrow effect dependencies**: use `user.id` not `user` as a dependency when only the id is needed
- **Split combined `useMemo`/`useEffect`**: separate independent computations with different dependency sets
- **`useDeferredValue` for expensive derived renders**: keeps input responsive while heavy lists/charts catch up
- **`useRef` for transient values**: mouse position, scroll offset, interval IDs — values that change frequently but don't need to trigger re-renders
- **`startTransition` for non-urgent updates**: mark scroll tracking, search filtering, etc. as non-urgent
- **Lazy `useState` initialization**: `useState(() => JSON.parse(localStorage.getItem('x') || '{}'))` — avoids re-running expensive init on every render
- **Hoist default non-primitive props in `memo` components**: extract `const NOOP = () => {}` to module scope to prevent broken memoization
- **Put interaction logic in event handlers, not effects**: if a side effect is triggered by a click/submit, do it in the handler — not `useEffect([submitted])`
- **Subscribe to derived boolean state**: `useMediaQuery('(max-width: 767px)')` re-renders only on boolean changes, not every pixel

## 6. Rendering Performance — MEDIUM

- **Explicit conditional rendering**: use `count > 0 ? <Badge /> : null` not `count && <Badge />` — `0` renders as text
- **`useTransition` over manual loading state**: `const [isPending, startTransition] = useTransition()` — auto-resets on error, handles interruptions
- **`content-visibility: auto`** on long-list items: defers layout/paint for off-screen rows
- **Animate `<div>` wrapper, not `<svg>`**: browsers GPU-accelerate transforms on divs, not SVG elements
- **`suppressHydrationWarning`** on elements with intentionally different server/client values (e.g., timestamps, locale-formatted dates) — don't use to hide bugs
- **Inline script for localStorage-based themes**: avoids flash without SSR breakage — runs synchronously before React hydrates
- **`<Activity mode="hidden">`** for frequently toggled expensive components: preserves state/DOM instead of remounting
- **React DOM resource hints**: use `preconnect()`, `preload()`, `preinit()` in Server Components to start loading fonts/scripts before the client receives HTML
- **Hoist static JSX** to module scope: `const skeleton = <div className="animate-pulse" />` — avoids re-creation every render

## 7. JavaScript Performance — LOW-MEDIUM

- **Build index `Map` for repeated lookups**: `new Map(users.map(u => [u.id, u]))` — O(n) build, O(1) per lookup vs O(n) per `.find()`
- **Batch DOM reads then writes**: avoid layout thrashing by not interleaving `element.style.x = ...` with `element.offsetWidth` reads
- **`Set`/`Map` for membership checks**: `allowedIds.has(id)` is O(1) vs `allowedIds.includes(id)` O(n)
- **`.toSorted()` not `.sort()`**: immutable sort — never mutate React props/state arrays
- **`.flatMap()` over `.map().filter()`**: single pass, no intermediate array
- **Early return**: return as soon as the result is determined
- **Loop for min/max instead of sort**: single O(n) pass vs O(n log n) sort
- **Combine multiple `.filter()` passes** into one `for...of` loop
- **Cache `localStorage`/`sessionStorage` reads** in a module-level Map — storage access is synchronous and expensive
- **`requestIdleCallback` for non-critical work**: defer analytics tracking, prefetch, local state persistence to browser idle time
- **Hoist `RegExp` outside render**: use `useMemo(() => new RegExp(query, 'gi'), [query])` or module-level constants

## 8. Advanced Patterns — LOW

- **Never put `useEffectEvent` results in dependency arrays**: their identity changes every render by design — only use reactive values as deps
- **`useEffectEvent` for stable callbacks in effects**: wraps a callback so the effect always calls the latest version without needing it as a dependency
- **Initialize app-wide singletons with a module-level guard**: `let didInit = false` — `useEffect([])` runs twice in dev Strict Mode
- **Store event handlers in refs** for stable subscriptions: or use `useEffectEvent` (React 19)
