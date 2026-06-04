# CLAUDE.md

Behavioral guidelines to reduce common LLM coding mistakes. Merge with project-specific instructions as needed.

**Tradeoff:** These guidelines bias toward caution over speed. For trivial tasks, use judgment.

## Styling: Tailwind CSS

**All styling must use [Tailwind CSS](https://tailwindcss.com) utility classes.** Do not write custom CSS files, inline `style` props, or CSS-in-JS.

- Apply classes directly on elements: `<div className="flex items-center gap-4 p-6">`
- Use Tailwind's responsive prefixes for breakpoints: `md:flex-row`, `lg:text-xl`
- Use `dark:` prefix for dark mode variants
- Prefer composing utilities over extracting components with `@apply` — only use `@apply` for truly repeated patterns that can't be componentized
- Do not install or use other styling solutions (styled-components, emotion, CSS modules, etc.)

---

## UI Components: shadcn/ui Only

**All UI components must use [shadcn/ui](https://ui.shadcn.com).** Do not install or use other component libraries (MUI, Chakra, Radix directly, Ant Design, etc.).

- Import from `@/components/ui/...` (e.g. `import { Button } from '@/components/ui/button'`)
- If a needed component isn't installed yet, add it with `npx shadcn@latest add <component>`
- Build composite UI from shadcn primitives — don't hand-roll what shadcn provides
- shadcn/ui components are pre-styled with Tailwind — extend them with `className` props, not overrides

---

## 0. Check Skills First

Before implementing anything, scan the available skills. If a skill matches the task invoke it via the Skill tool before writing code.

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

# Next.js Advanced Routing

## Overview

Provide comprehensive guidance for advanced Next.js App Router features including Route Handlers (API routes), Parallel Routes, Intercepting Routes, Server Actions, error handling, draft mode, and streaming with Suspense.

## TypeScript: NEVER Use `any` Type

**CRITICAL RULE:** This codebase has `@typescript-eslint/no-explicit-any` enabled. Using `any` will cause build failures.

**❌ WRONG:**
```typescript
function handleSubmit(e: any) { ... }
const data: any[] = [];
```

**✅ CORRECT:**
```typescript
function handleSubmit(e: React.FormEvent<HTMLFormElement>) { ... }
const data: string[] = [];
```

### Common Next.js Type Patterns

```typescript
// Page props
function Page({ params }: { params: { slug: string } }) { ... }
function Page({ searchParams }: { searchParams: { [key: string]: string | string[] | undefined } }) { ... }

// Form events
const handleSubmit = (e: React.FormEvent<HTMLFormElement>) => { ... }
const handleChange = (e: React.ChangeEvent<HTMLInputElement>) => { ... }

// Server actions
async function myAction(formData: FormData) { ... }
```

## ⚠️ CRITICAL: Server Action File Naming and Location

When work requirements mention a specific filename, follow that instruction exactly. If no name is given, pick the option that best matches the project conventions—`app/actions.ts` is a safe default for collections of actions, while `app/action.ts` works for a single form handler.

### Choosing between `action.ts` and `actions.ts`

- **Match existing patterns:** Check whether the project already has an actions file and extend it if appropriate.
- **Single vs multiple exports:** Prefer `action.ts` for a single action, and `actions.ts` for a group of related actions.
- **Explicit requirement:** If stakeholders call out a specific name, do not change it.

**Location guidelines**
- Server actions belong under the `app/` directory so they can participate in the App Router tree.
- Keep the file alongside the UI that invokes it unless shared across multiple routes.
- Avoid placing actions in `lib/` or `utils/` unless they are triggered from multiple distant routes and remain server-only utilities.

**Example placement**
```
app/
├── actions.ts       ← Shared actions that support multiple routes
└── dashboard/
    └── action.ts    ← Route-specific action colocated with a single page
```

## ⚠️ CRITICAL: Server Actions Return Types - Form Actions MUST Return Void

**This is a TypeScript requirement, not optional.**

When using form action attribute: `<form action={serverAction}>`
- The function **MUST have no return statement** (implicitly returns void)
- TypeScript will **REJECT any return value**, even `return undefined` or `return null`

❌ WRONG (causes build error):
```typescript
export async function saveForm(formData: FormData) {
  'use server';
  await db.save(name);
  return { success: true }; // ❌ BUILD ERROR
}
<form action={saveForm}>  {/* ❌ Expects void function */}
```

✅ CORRECT - Option 1 (Simple form action, no response):
```typescript
export async function saveForm(formData: FormData) {
  'use server';
  if (!name) throw new Error('Name required');
  await db.save(name);
  revalidatePath('/');
  // No return statement
}
```

✅ CORRECT - Option 2 (With useActionState for feedback):
```typescript
export async function saveForm(prevState: FormState, formData: FormData) {
  'use server';
  if (!name) return { error: 'Name required' };
  await db.save(name);
  return { success: true, message: 'Saved!' }; // ✅ OK with useActionState
}
```

**The key rule:** `<form action={...}>` expects `void`. If you need to return data, use `useActionState`.

## ⚠️ CRITICAL: Server Actions File Organization

**Two Patterns for 'use server' Directive:**

**Pattern 1: File-level (recommended for multiple actions):**
```typescript
// app/actions.ts
'use server';  // At the top - ALL exports are server actions

export async function createPost(formData: FormData) { ... }
export async function deletePost(postId: string) { ... }
```

**Pattern 2: Function-level (for single action or mixed file):**
```typescript
export async function createPost(formData: FormData) {
  'use server';  // Inside the function - ONLY this function is a server action
  await db.posts.create({ data: { title } });
}
```

**Client Component Calling Server Action — CORRECT Pattern:**
```typescript
// app/actions.ts - Server Actions file
'use server';
import { cookies } from 'next/headers';

export async function updateUserPreference(key: string, value: string) {
  const cookieStore = await cookies();
  cookieStore.set(key, value);
}

// app/InteractiveButton.tsx - Client Component
'use client';
import { updateUserPreference } from './actions';

export default function InteractiveButton() {
  return <button onClick={() => updateUserPreference('theme', 'dark')}>Update</button>;
}
```

**❌ WRONG - Mixing 'use server' and 'use client' in same file.**

## Route Handlers (API Routes)

Route Handlers replace API Routes from the Pages Router. Create them in `route.ts` files.

```typescript
// app/api/hello/route.ts
export async function GET(request: Request) {
  return Response.json({ message: 'Hello World' });
}

export async function POST(request: Request) {
  const body = await request.json();
  return Response.json({ message: 'Data received', data: body });
}
```

### Dynamic Route Handlers

```typescript
// app/api/posts/[id]/route.ts
export async function GET(
  request: Request,
  { params }: { params: { id: string } }
) {
  const post = await db.posts.findUnique({ where: { id: params.id } });
  return Response.json(post);
}
```

### Setting Cookies in Route Handlers

```typescript
// app/api/login/route.ts
import { cookies } from 'next/headers';

export async function POST(request: Request) {
  const { email, password } = await request.json();
  const token = await authenticate(email, password);
  if (!token) return Response.json({ error: 'Invalid credentials' }, { status: 401 });

  const cookieStore = await cookies();
  cookieStore.set('session-token', token, {
    httpOnly: true,
    secure: process.env.NODE_ENV === 'production',
    maxAge: 60 * 60 * 24 * 7,
    path: '/',
  });

  return Response.json({ success: true });
}
```

## Parallel Routes — Determine Scope First

Before implementing parallel routes, identify WHERE they should live.

- **Specific page/section** → Create under that route directory
- **Entire application** → Create at root level

❌ **WRONG - Parallel routes at root when feature-specific is needed:**
```
app/@x/   app/@y/   ← affects entire app
```

✅ **CORRECT:**
```
app/[feature]/@x/   app/[feature]/@y/   ← scoped to feature
```

### Layout with Parallel Routes

```typescript
// app/[feature]/layout.tsx
export default function FeatureLayout({
  children, slot1, slot2,
}: {
  children: React.ReactNode;
  slot1: React.ReactNode;
  slot2: React.ReactNode;
}) {
  return (
    <div>
      <div>{slot1}</div>
      <div>{slot2}</div>
    </div>
  );
}
```

Always create `default.tsx` for each slot:
```typescript
export default function Default() { return null; }
```

## Intercepting Routes

Conventions: `(.)` same level, `(..)` one level above, `(...)` from root.

Modal pattern:
```
app/
├── photos/[id]/page.tsx        # Full photo page
├── @modal/(.)photos/[id]/page.tsx  # Modal photo view
└── layout.tsx
```

## Error Boundaries

```typescript
// app/error.tsx
'use client';
export default function Error({ error, reset }: { error: Error & { digest?: string }; reset: () => void }) {
  return (
    <div>
      <h2>Something went wrong!</h2>
      <button onClick={reset}>Try again</button>
    </div>
  );
}
```

## Streaming and Suspense

```typescript
// app/dashboard/page.tsx
import { Suspense } from 'react';

export default function Dashboard() {
  return (
    <div>
      <Suspense fallback={<StatsSkeleton />}>
        <Stats />
      </Suspense>
      <Suspense fallback={<ActivitySkeleton />}>
        <RecentActivity />
      </Suspense>
    </div>
  );
}
```

## Revalidation and Redirection

```typescript
import { revalidatePath, revalidateTag } from 'next/cache';
import { redirect } from 'next/navigation';

export async function deletePost(postId: string) {
  await db.posts.delete({ where: { id: postId } });
  revalidatePath('/posts');
  redirect('/posts');
}
```

---

# Next.js Anti-Patterns

## Default to Server Components

Add `'use client'` **only** to the lowest-level component that needs it. Never put it on layouts, pages, or parent wrappers that contain static content.

## useEffect Anti-Patterns

### Browser detection — no hooks needed

❌ WRONG:
```typescript
'use client';
const [isSafari, setIsSafari] = useState(false);
useEffect(() => { setIsSafari(/Safari/.test(navigator.userAgent)); }, []);
```

✅ CORRECT:
```typescript
'use client';
const isSafari = typeof navigator !== 'undefined' &&
  /Safari/.test(navigator.userAgent) && !/Chrome/.test(navigator.userAgent);
if (isSafari) return <h1>Unsupported Browser</h1>;
```

- Use `typeof navigator !== 'undefined'` for SSR safety
- Direct detection in component body — no hooks, no state
- Place logic in the exported component consumers render (or compose a helper into it)

### Data fetching — use Server Components

❌ WRONG: `useEffect` + `useState` + `fetch('/api/...')`

✅ CORRECT:
```typescript
// No 'use client' — Server Component
export default async function BlogPosts() {
  const res = await fetch('https://api.example.com/posts', { next: { revalidate: 3600 } });
  const posts = await res.json();
  return <ul>{posts.map(p => <li key={p.id}>{p.title}</li>)}</ul>;
}
```

### URL access — read in event handler, not effect

❌ `useEffect(() => { setUrl(window.location.href); }, [])`
✅ `const handleShare = () => { const url = window.location.href; ... }`

## useState Anti-Patterns

### Derived values — calculate inline or useMemo

❌ `useEffect(() => { setTotal(products.reduce(...)); }, [products])`
✅ `const total = products.reduce((sum, p) => sum + p.price, 0);`

## Pages Router Patterns (don't use in App Router)

| ❌ Pages Router | ✅ App Router |
|---|---|
| `getServerSideProps` | `async` Server Component + `cache: 'no-store'` |
| `getStaticProps` + `revalidate` | `async` Server Component + `next: { revalidate: N }` |
| `import Head from 'next/head'` | `export const metadata: Metadata = { ... }` |

## Performance Anti-Patterns

### Serial awaits — use Promise.all

❌ `const user = await fetchUser(); const posts = await fetchPosts();`
✅ `const [user, posts] = await Promise.all([fetchUser(), fetchPosts()]);`

Even better: use Suspense so each section streams independently.

### Unnecessary API routes

If a Server Component can query the DB directly, skip the API route. API routes are for: external webhooks, client-side mutations, third-party integrations, public endpoints.

### Server Component inside Client Component

❌ Import ServerComponent directly into a `'use client'` file — it becomes a Client Component.
✅ Pass it as `children` from a Server parent into the Client wrapper.

## Navigation

- Use `<Link href="...">` for static links
- Use `useRouter().push()` in Client Component event handlers
- Use `redirect()` from `next/navigation` in Server Components
- Never `window.location.href = ...` (full page reload)
- Never `useRouter()` in a Server Component

## When Client Components ARE correct

- Event handlers / interactivity (`onClick`, `onChange`)
- `useState` / `useReducer` for UI state
- `useEffect` for subscriptions, scroll listeners, third-party DOM libs
- Browser APIs (`window`, `navigator`, `localStorage`)
- React Context consumers

## Quick Detection Checklist

- `useEffect` for data fetch → Server Component
- `useEffect` for browser detection → direct check in body
- `useState` for server data → Server Component
- `getServerSideProps` / `getStaticProps` → async Server Component
- `next/head` → `metadata` export
- Serial `await` chains → `Promise.all` or Suspense
- `'use client'` on static components → remove it
- Client Component importing Server Component → composition pattern
- `window.location.href` for nav → `Link` or `useRouter`

---

# Next.js App Router Fundamentals

## File Conventions

| File | Purpose |
|---|---|
| `layout.tsx` | Shared UI, preserves state, does NOT re-render on navigation |
| `page.tsx` | Makes a route publicly accessible |
| `loading.tsx` | Suspense fallback for the route segment |
| `error.tsx` | Error boundary for the route segment (must be `'use client'`) |
| `not-found.tsx` | 404 UI |
| `route.ts` | API endpoint (Route Handler) |
| `template.tsx` | Like layout but re-renders on every navigation |

Only `page.tsx` and `route.ts` create public routes. Other colocated files (components, utils) are not routable.

## Pages Router → App Router Mapping

| Pages Router | App Router |
|---|---|
| `pages/index.tsx` | `app/page.tsx` |
| `pages/about.tsx` | `app/about/page.tsx` |
| `pages/[id].tsx` | `app/[id]/page.tsx` |
| `pages/_app.tsx` | `app/layout.tsx` |
| `pages/_document.tsx` | `app/layout.tsx` (html/body tags) |
| `pages/api/hello.ts` | `app/api/hello/route.ts` |
| `getStaticPaths` | `generateStaticParams` |
| `getStaticProps` | async Server Component + `next: { revalidate: N }` |
| `getServerSideProps` | async Server Component + `cache: 'no-store'` |

## Root Layout (Required)

`app/layout.tsx` is **required** and must include `<html>` and `<body>`:

```typescript
export default function RootLayout({ children }: { children: React.ReactNode }) {
  return (
    <html lang="en">
      <body>{children}</body>
    </html>
  );
}
```

## Metadata

Static:
```typescript
import type { Metadata } from 'next';
export const metadata: Metadata = {
  title: 'My Page',
  description: 'Description',
  openGraph: { title: 'My Page', images: ['/og.jpg'] },
};
```

Dynamic:
```typescript
export async function generateMetadata({ params }: { params: { slug: string } }): Promise<Metadata> {
  const post = await getPost(params.slug);
  return { title: post.title, description: post.excerpt };
}
```

## Route Groups

Parenthesized folders group routes without affecting the URL:
```
app/
├── (marketing)/about/page.tsx   → /about
├── (marketing)/contact/page.tsx → /contact
└── (shop)/products/page.tsx     → /products
```

## generateStaticParams

App Router replacement for `getStaticPaths`. Server Components only — no `'use client'`.

```typescript
// app/blog/[slug]/page.tsx
export async function generateStaticParams() {
  const posts = await db.post.findMany();
  return posts.map(p => ({ slug: p.slug }));
}

// export const dynamicParams = false; // 404 for non-pre-rendered paths (default: true)
```

- Must be `export`ed
- Returns array of param objects, one per page to pre-render
- For multiple segments: return objects with all segment keys

## Common Pitfalls

- Root layout missing `<html>`/`<body>` → build error
- Route without `page.tsx` → not accessible (layout alone is not a route)
- `next/head` in App Router → use `metadata` export instead
- `'use client'` on `generateStaticParams` file → it won't work
- Conflicting routes in both `pages/` and `app/` → build failure, remove old files

---

# Next.js: Client-Triggered Cookie Pattern

Client components can't set cookies directly — only server code can. The pattern: client component handles the interaction, server action sets the cookie.

```typescript
// app/actions.ts
'use server';
import { cookies } from 'next/headers';

export async function setTheme(theme: 'light' | 'dark') {
  const cookieStore = await cookies(); // await required in Next.js 15+
  cookieStore.set('theme', theme, {
    httpOnly: true,
    secure: process.env.NODE_ENV === 'production',
    sameSite: 'lax',
    maxAge: 60 * 60 * 24 * 365,
    path: '/',
  });
}

// app/ThemeToggle.tsx
'use client';
import { setTheme } from './actions';

export default function ThemeToggle() {
  return <button onClick={() => setTheme('dark')}>Dark Mode</button>;
}
```

## Cookie options

| Option | Effect |
|---|---|
| `httpOnly: true` | Blocks JS access — use for sessions/auth |
| `httpOnly: false` | Client JS can read it — use for UI prefs like theme |
| `secure: true` | HTTPS only |
| `sameSite: 'lax'` | CSRF protection (default for most cases) |
| `maxAge` | Seconds until expiry |

## Reading cookies

**Server Component:**
```typescript
import { cookies } from 'next/headers';
const cookieStore = await cookies();
const theme = cookieStore.get('theme')?.value ?? 'light';
```

**Client Component** — `next/headers` is unavailable; read from `document.cookie`:
```typescript
const theme = document.cookie.split('; ')
  .find(r => r.startsWith('theme='))?.split('=')[1] ?? 'light';
```

## With redirect after setting cookie

```typescript
export async function login(email: string, password: string) {
  const session = await authenticate(email, password);
  const cookieStore = await cookies();
  cookieStore.set('session', session.token, {
    httpOnly: true, secure: true, sameSite: 'lax',
    maxAge: 60 * 60 * 24 * 7,
  });
  redirect('/dashboard');
}
```

---

# Next.js Dynamic Routes & Params

## Route structure — default to simplest

**Do NOT infer nesting from resource names.** Use the flattest structure unless the URL is explicitly specified.

| Requirement | ❌ Over-engineered | ✅ Correct |
|---|---|---|
| "Fetch a product by ID" | `app/products/[id]/page.tsx` | `app/[id]/page.tsx` |
| "Show user profile" | `app/users/[userId]/page.tsx` | `app/[userId]/page.tsx` |
| "Create route at /blog/[slug]" | — | `app/blog/[slug]/page.tsx` ✅ (explicit) |

Only add a prefix segment when the URL structure is explicitly required.

## Route syntax

```
app/[id]/page.tsx              → /123, /abc
app/blog/[slug]/page.tsx       → /blog/hello-world
app/[cat]/[id]/page.tsx        → /electronics/123
app/docs/[...slug]/page.tsx    → /docs/a, /docs/a/b/c  (slug is string[])
app/shop/[[...slug]]/page.tsx  → /shop, /shop/a, /shop/a/b  (optional)
```

## Accessing params — Next.js 15+: params is a Promise

```typescript
// ✅ CORRECT — Next.js 15+
export default async function Page({
  params,
}: {
  params: Promise<{ id: string }>;
}) {
  const { id } = await params;
  // ...
}

// ❌ WRONG — missing Promise wrapper, will fail at runtime
export default async function Page({ params }: { params: { id: string } }) {
  const item = await getItem(params.id); // params is a Promise!
}
```

Same applies to `generateMetadata` and Route Handlers:
```typescript
export async function generateMetadata({ params }: { params: Promise<{ id: string }> }) {
  const { id } = await params;
  return { title: (await getItem(id)).name };
}

// app/api/items/[id]/route.ts
export async function GET(_req: Request, { params }: { params: Promise<{ id: string }> }) {
  const { id } = await params;
  return Response.json(await db.items.findById(id));
}
```

## Client Components — use `useParams()`

`params` prop is not available in Client Components:
```typescript
'use client';
import { useParams } from 'next/navigation';

export default function ClientPage() {
  const { id } = useParams<{ id: string }>();
  // ...
}
```

Or pass params down as a prop from the Server Component parent.

## Always handle not-found

```typescript
import { notFound } from 'next/navigation';

const item = await getItem(id);
if (!item) notFound();
```

## Catch-all params

```typescript
// app/docs/[...slug]/page.tsx
const { slug } = await params; // string[]
const path = slug.join('/');

// Optional catch-all: app/shop/[[...slug]]/page.tsx
const { slug = [] } = await params;
```

## Fetch cache strategies

```typescript
// Always fresh (equivalent to getServerSideProps)
await fetch(url, { cache: 'no-store' });

// Cached, revalidate on interval (equivalent to getStaticProps + revalidate)
await fetch(url, { next: { revalidate: 60 } });

// Cached indefinitely until manually invalidated
await fetch(url, { next: { tags: ['product'] } });
// then: revalidateTag('product') in a server action
```

**Never add `'use client'` to a page that only needs to fetch data by URL param** — it's a Server Component by default and that's correct.

---

# Next.js: searchParams, useSearchParams & React `use()`

## searchParams in Server Components (Next.js 15+: it's a Promise)

`searchParams` is only available in `page.tsx`, never in `layout.tsx`.

```typescript
// app/search/page.tsx
export default async function SearchPage({
  searchParams,
}: {
  searchParams: Promise<{ q?: string; category?: string }>;
}) {
  // Inline access — keeps searchParams and the key on the same line
  const q = (await searchParams).q ?? '';
  const category = (await searchParams).category ?? 'all';

  const results = await searchProducts(q, category);
  return <ProductList products={results} />;
}
```

**Access pattern — always inline, never via intermediate variable:**
```typescript
// ✅ CORRECT
const q = (await searchParams).q ?? '';

// ❌ WRONG — don't use an intermediate variable
const params = await searchParams;
const q = params.q;
```

## useSearchParams — always needs both `'use client'` AND `<Suspense>`

```typescript
// app/page.tsx (Server Component parent)
import { Suspense } from 'react';
import SearchComponent from './SearchComponent';

export default function Page() {
  return (
    <Suspense fallback={<div>Loading...</div>}>
      <SearchComponent />
    </Suspense>
  );
}

// app/SearchComponent.tsx
'use client';
import { useSearchParams } from 'next/navigation';

export default function SearchComponent() {
  const searchParams = useSearchParams();
  const q = searchParams.get('q') ?? '';
  return <div>Query: {q}</div>;
}
```

Missing either `'use client'` or `<Suspense>` will cause a runtime error.

### Updating URL params from a Client Component

```typescript
'use client';
import { useSearchParams, useRouter } from 'next/navigation';

function Filters() {
  const searchParams = useSearchParams();
  const router = useRouter();

  const updateParam = (key: string, value: string) => {
    const params = new URLSearchParams(searchParams.toString()); // copy current params
    if (value) {
      params.set(key, value);
    } else {
      params.delete(key);
    }
    router.push(`?${params.toString()}`);
  };

  return <select onChange={(e) => updateParam('category', e.target.value)}>...</select>;
}
```

Reading multi-value params: `searchParams.getAll('tag')` → `string[]`

## React `use()` — pass a Server-fetched promise to a Client Component

Lets a Server Component start a fetch and hand the promise to a Client Component to unwrap (with Suspense streaming):

```typescript
// app/profile/page.tsx (Server Component)
import { Suspense } from 'react';
import UserProfile from './UserProfile';

export default function Page() {
  const userPromise = fetchUser(); // start fetch, don't await
  return (
    <Suspense fallback={<div>Loading...</div>}>
      <UserProfile userPromise={userPromise} />
    </Suspense>
  );
}

// app/UserProfile.tsx
'use client';
import { use } from 'react';

export default function UserProfile({ userPromise }: { userPromise: Promise<User> }) {
  const user = use(userPromise); // suspends until resolved
  return <div>{user.name}</div>;
}
```

## Server-only APIs — cannot be used in Client Components

| API | Available in |
|---|---|
| `cookies()` from `next/headers` | Server Components only |
| `headers()` from `next/headers` | Server Components only |
| `searchParams` prop | Server `page.tsx` only |
| `redirect()` from `next/navigation` | Server Components only |
| `useRouter()`, `usePathname()`, `useSearchParams()` | Client Components only |

## Button navigation without `'use client'` — use a form + Server Action

When a button needs to navigate (e.g. logout, submit-and-redirect), keep the page as a Server Component and redirect inside the action:

```typescript
// app/actions.ts
'use server';
import { redirect } from 'next/navigation';

export async function logout() {
  await clearSession();
  redirect('/login');
}

// app/page.tsx — no 'use client' needed
import { logout } from './actions';

export default function Page() {
  return (
    <form action={logout}>
      <button type="submit">Logout</button>
    </form>
  );
}
```

Only reach for `useRouter()` when you genuinely need client-side programmatic navigation (e.g. after an animation, or based on client state).
