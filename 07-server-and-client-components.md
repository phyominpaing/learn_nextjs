# Next.js 07 — Server & Client Components

**Server Components** and **Client Components** are the two kinds of React components in the App Router: one runs only on the server and ships no JavaScript to the browser, the other runs in the browser (after being pre-rendered on the server) and is where all interactivity lives.

## Table of Contents

- [The Core Idea](#the-core-idea)
- [When to Use Which](#when-to-use-which)
- [How They Work Under the Hood](#how-they-work-under-the-hood)
- [The `'use client'` Boundary](#the-use-client-boundary)
- [Keep Client Components Small (Push the Boundary Down)](#keep-client-components-small-push-the-boundary-down)
- [Passing Data from Server to Client](#passing-data-from-server-to-client)
- [Interleaving: Server Components Inside Client Components](#interleaving-server-components-inside-client-components)
- [Context Providers](#context-providers)
- [Third-Party Components](#third-party-components)
- [Preventing Environment Poisoning](#preventing-environment-poisoning)
- [Hydration Errors (Senior-Level Debugging)](#hydration-errors-senior-level-debugging)
- [Client-Only Libraries with `next/dynamic`](#client-only-libraries-with-nextdynamic)
- [A Real-World Composition Example](#a-real-world-composition-example)
- [Common Mistakes](#common-mistakes)
- [Quick Revision](#quick-revision)

---

## The Core Idea

In the App Router, **every component is a Server Component by default** — layouts, pages, and anything you import into them. You only opt into the browser by adding the `'use client'` directive.

```tsx
// Server Component (default) — can be async, can query a database
export default async function Page() {
  const post = await getPost('hello-world')
  return <h1>{post.title}</h1>
}
```

```tsx
// Client Component — opted in with the directive on the first line
'use client'
import { useState } from 'react'

export default function Counter() {
  const [count, setCount] = useState(0)
  return <button onClick={() => setCount(count + 1)}>{count} likes</button>
}
```

- **Server Component** — rendered on the server only. Its code never reaches the browser.
- **Client Component** — its JavaScript is sent to the browser so it can respond to clicks, hold state, and use browser APIs.

> 💡 **Tip:** Because Server Component code never reaches the browser, a `console.log` inside one prints to your **terminal**, not the browser DevTools console. This trips up almost every beginner.

## When to Use Which

| You need to... | Use |
|---|---|
| Fetch data from a database or API, close to the source | **Server** |
| Use API keys, tokens, or secrets safely | **Server** |
| Reduce the JavaScript sent to the browser | **Server** |
| Improve First Contentful Paint and stream content | **Server** |
| Hold state (`useState`, `useReducer`) | **Client** |
| Handle events (`onClick`, `onChange`) | **Client** |
| Run effects/lifecycle logic (`useEffect`) | **Client** |
| Use browser-only APIs (`window`, `localStorage`, `navigator.geolocation`) | **Client** |
| Use custom hooks that rely on state/effects | **Client** |

> 💡 **Tip:** Start with a Server Component. Reach for `'use client'` only when one of the bottom five rows applies — and apply it to the **smallest** component that needs it.

## How They Work Under the Hood

Understanding this flow is what separates "I use `'use client'`" from "I understand the architecture."

**On the server:**

1. Server Components are rendered into a compact, serialized format called the **RSC Payload** (React Server Component Payload).
2. Client Components and the RSC Payload are then used to **prerender HTML**.

The RSC Payload contains three things:

- The **rendered result** of Server Components
- **Placeholders** marking where Client Components go, plus references to their JavaScript files
- Any **props** passed from Server Components to Client Components

**On the client (first page load):**

```text
1. HTML        -> shows a fast, non-interactive preview immediately
2. RSC Payload -> reconciles the Server and Client component trees
3. JavaScript  -> hydrates Client Components, making them interactive
```

**On subsequent navigations:**

- The RSC Payload is **prefetched and cached** for instant navigation.
- Client Components render **entirely on the client**, without server-rendered HTML.

> ⚠️ **Warning:** A very common misconception is "Client Components don't render on the server." They **do** — on the initial page load their HTML is pre-rendered on the server, then hydrated in the browser. `'use client'` means "this component's JavaScript is shipped to the browser," not "this only runs in the browser."

**Hydration** is React's process of attaching event handlers to server-rendered HTML so the static page becomes interactive.

## The `'use client'` Boundary

`'use client'` doesn't mark one component — it declares a **boundary** between the server and client module graphs.

```tsx
// app/ui/dashboard-shell.tsx
'use client'

import Chart from './chart'       // becomes part of the client bundle
import Filters from './filters'   // becomes part of the client bundle

export default function DashboardShell() {
  // ...
}
```

- Once a file has `'use client'`, **everything it imports and directly renders joins the client bundle.** You do **not** need to add the directive to `Chart` or `Filters` individually.
- This applies to the **module graph** — the files a module imports and the components it renders directly.
- It does **not** apply to Server Components passed in as `children` or other props. Those are rendered on the server and handed over as already-rendered output (more on this in the interleaving section).

> ⚠️ **Warning:** If you `import` a Server Component directly into a Client Component file, it stops being a Server Component — it joins the client bundle. An `async` component, or one that touches the database, will then break. Pass it as `children` instead.

## Keep Client Components Small (Push the Boundary Down)

Every line of code inside a client boundary adds to your JavaScript bundle. Mark only the **interactive leaf**, not the whole page.

```tsx
// app/layout.tsx — stays a Server Component
import Search from './ui/search'  // Client Component
import Logo from './ui/logo'      // Server Component

export default function Layout({ children }: { children: React.ReactNode }) {
  return (
    <>
      <nav>
        <Logo />
        <Search />
      </nav>
      <main>{children}</main>
    </>
  )
}
```

```tsx
// app/ui/search.tsx — only this small piece is a Client Component
'use client'

export default function Search() {
  // input state, onChange handlers, etc.
}
```

The layout is mostly static (a logo and links) with one interactive search bar. Only `<Search />` needs the directive — the rest ships **zero** JavaScript.

```text
❌ Anti-pattern:  'use client' at the top of layout.tsx   -> the whole app becomes client JS
✅ Pattern:       'use client' only inside search.tsx     -> only the search box ships JS
```

## Passing Data from Server to Client

Use ordinary props:

```tsx
// app/[id]/page.tsx — Server Component
import LikeButton from '@/app/ui/like-button'
import { getPost } from '@/lib/data'

export default async function Page({ params }: { params: Promise<{ id: string }> }) {
  const { id } = await params
  const post = await getPost(id)

  return (
    <main>
      <h1>{post.title}</h1>
      <LikeButton likes={post.likes} />
    </main>
  )
}
```

```tsx
// app/ui/like-button.tsx — Client Component
'use client'

export default function LikeButton({ likes }: { likes: number }) {
  // ...
}
```

The server does the data fetching; the client component only receives the number it needs.

**Props must be serializable.** They travel inside the RSC Payload, so React has to be able to turn them into data and rebuild them in the browser.

| Generally OK to pass | Not OK to pass |
|---|---|
| Strings, numbers, booleans, `null` | Regular functions (event handlers defined on the server) |
| Plain objects and arrays | Class instances (e.g. an ORM model instance with methods) |
| Other React-supported types such as `Date`, `Map`, `Set`, and Promises | Anything React can't serialize |
| Server Actions (a special kind of function — covered in the Server Actions note) | |

> ⚠️ **Warning:** Remember from nextjs-06: **everything you pass as a prop to a Client Component is visible to the user.** Pass a minimal DTO, never a raw database row.

> 💡 **Tip:** You can also *stream* a Server Component's promise to a Client Component and unwrap it with React's `use()` — see nextjs-06.

## Interleaving: Server Components Inside Client Components

This is the most important composition pattern. **You can pass Server Components as props — usually `children` — into a Client Component**, which visually nests server-rendered UI inside client-managed UI.

```tsx
// app/ui/modal.tsx — Client Component (owns open/close state)
'use client'

export default function Modal({ children }: { children: React.ReactNode }) {
  return <div className="modal">{children}</div>
}
```

```tsx
// app/page.tsx — Server Component
import Modal from './ui/modal'
import Cart from './ui/cart'   // a Server Component that fetches on the server

export default function Page() {
  return (
    <Modal>
      <Cart />
    </Modal>
  )
}
```

Why this works: `Page` (on the server) renders `<Cart />` ahead of time. The RSC Payload carries `Cart`'s **rendered output** plus a placeholder for `Modal`. The browser never needs `Cart`'s code — it just slots the finished result into `Modal`'s `children`.

```text
✅ Pass as children/props:   <Modal><Cart /></Modal>         -> Cart stays a Server Component
❌ Import inside client file: import Cart from './cart'      -> Cart is pulled into the client bundle
```

> 💡 **Tip:** Think of `children` as a **slot**. The Client Component controls *when and how* the slot appears (open, closed, animated); the Server Component controls *what* fills it. This is exactly how the modal pattern from nextjs-04 stays mostly server-rendered.

## Context Providers

React context (`createContext`) is **not supported in Server Components**. The fix is to create a Client Component provider that accepts `children`:

```tsx
// app/theme-provider.tsx
'use client'

import { createContext } from 'react'

export const ThemeContext = createContext({})

export default function ThemeProvider({ children }: { children: React.ReactNode }) {
  return <ThemeContext.Provider value="dark">{children}</ThemeContext.Provider>
}
```

```tsx
// app/layout.tsx — still a Server Component
import ThemeProvider from './theme-provider'

export default function RootLayout({ children }: { children: React.ReactNode }) {
  return (
    <html>
      <body>
        <ThemeProvider>{children}</ThemeProvider>
      </body>
    </html>
  )
}
```

Even though `ThemeProvider` is a Client Component, the pages passed through `children` can remain Server Components — that's the interleaving pattern again.

> 💡 **Tip:** Render providers **as deep as possible**. Notice `ThemeProvider` wraps only `{children}`, not the whole `<html>` document — this lets Next.js optimize the static parts of your Server Components.

**Real-world version:** combine all your providers (theme, TanStack Query, auth state) in one file:

```tsx
// app/providers.tsx
'use client'

import { useState } from 'react'
import { QueryClient, QueryClientProvider } from '@tanstack/react-query'
import ThemeProvider from './theme-provider'

export default function Providers({ children }: { children: React.ReactNode }) {
  // Create the client inside useState so each browser session (and each server render)
  // gets its own instance instead of sharing one module-level singleton
  const [queryClient] = useState(() => new QueryClient())

  return (
    <QueryClientProvider client={queryClient}>
      <ThemeProvider>{children}</ThemeProvider>
    </QueryClientProvider>
  )
}
```

Then wrap `{children}` with `<Providers>` in the root layout.

> ⚠️ **Warning:** Don't create a `QueryClient` at module scope on the server — a shared instance can leak cached data between different users' requests.

## Third-Party Components

Many npm packages use `useState` or `useEffect` but don't (yet) have `'use client'` in their source. Using them **inside** a Client Component works fine. Using them **directly in a Server Component** throws an error.

```tsx
// app/carousel.tsx — a thin wrapper that fixes it
'use client'

import { Carousel } from 'acme-carousel'

export default Carousel
```

```tsx
// app/page.tsx — Server Component can now use the wrapper
import Carousel from './carousel'

export default function Page() {
  return (
    <div>
      <p>View pictures</p>
      <Carousel />
    </div>
  )
}
```

A two-line wrapper file is the standard fix.

> 💡 **Tip (for library authors):** If you publish a component library, add `'use client'` to the entry points that need client features so your users don't need wrappers. Be aware that some bundlers strip the directive during the build — verify your output still includes it.

## Preventing Environment Poisoning

JavaScript modules can be imported from both server and client code, so server-only code can be imported into the browser by mistake.

```ts
// lib/data.ts
export async function getData() {
  const res = await fetch('https://external-service.com/data', {
    headers: { authorization: process.env.API_KEY! },
  })
  return res.json()
}
```

If a Client Component imported this, `API_KEY` wouldn't leak — Next.js only inlines env vars prefixed `NEXT_PUBLIC_` into the client bundle, and replaces unprefixed ones with an **empty string**. But the function would silently fail in the browser — a confusing bug.

Guard against it with the `server-only` marker:

```ts
// lib/data.ts
import 'server-only'

export async function getData() { /* ... */ }
```

Now importing this module into a Client Component causes a **build-time error** with a clear message.

| Package | Marks a module as | Typical use |
|---|---|---|
| `server-only` | Must never be bundled for the client | DB access, secrets, DAL files |
| `client-only` | Must never run on the server | Code that touches `window` or `document` |

> 💡 **Tip:** Installing these packages is **optional** in Next.js — it handles the imports internally to give clearer errors. Install them anyway if your linter flags undeclared dependencies.

## Hydration Errors (Senior-Level Debugging)

A **hydration mismatch** happens when the HTML the server produced differs from what React renders on the client's first pass. React then warns (or in some cases re-renders) because it can't safely attach to mismatched DOM.

**Common causes:**

| Cause | Example |
|---|---|
| Time/random values during render | `new Date()`, `Math.random()`, `Date.now()` in JSX |
| Browser-only checks during render | `typeof window !== 'undefined' ? A : B` |
| Invalid HTML nesting | `<p>` containing a `<div>` |
| Reading `localStorage` during the first render | Server has none, browser does |
| Browser extensions modifying the DOM | Not your code, but shows up as a mismatch |

```tsx
// ❌ Server and client render different times -> mismatch
'use client'
export function Clock() {
  return <p>{new Date().toLocaleTimeString()}</p>
}
```

```tsx
// ✅ Render a stable placeholder first, fill in after hydration
'use client'
import { useEffect, useState } from 'react'

export function Clock() {
  const [time, setTime] = useState<string | null>(null)

  useEffect(() => {
    setTime(new Date().toLocaleTimeString())
  }, [])

  return <p>{time ?? '--:--:--'}</p>
}
```

The first client render matches the server's (`--:--:--`); `useEffect` only runs **after** hydration, so the real value appears without a mismatch.

> 💡 **Tip:** For a single unavoidable difference (like a timestamp), React allows `suppressHydrationWarning` on that one element. Use it sparingly — it hides the warning, it doesn't fix the cause.

## Client-Only Libraries with `next/dynamic`

Some libraries (maps, rich-text editors, charting libs) break when rendered on the server because they touch `window` immediately. Load them client-side only:

```tsx
'use client'
import dynamic from 'next/dynamic'

const Map = dynamic(() => import('./map'), {
  ssr: false,
  loading: () => <p>Loading map...</p>,
})

export default function Location() {
  return <Map />
}
```

- `ssr: false` skips server pre-rendering for that component entirely.
- `dynamic()` also code-splits — the library's JavaScript is only downloaded when the component is needed.
- Use `ssr: false` from inside a **Client Component**; it isn't allowed in Server Components in current Next.js versions.

(We'll revisit lazy loading as a performance tool in the optimization note.)

## A Real-World Composition Example

A product page combining everything from this note and nextjs-06:

```text
app/products/[id]/
├── page.tsx                # Server: fetches product, composes the page
├── loading.tsx             # Server: skeleton
└── _components/
    ├── product-info.tsx    # Server: title, price, description (zero client JS)
    ├── reviews.tsx         # Server: async, fetches reviews (wrapped in <Suspense>)
    ├── add-to-cart.tsx     # Client: quantity state + onClick
    └── image-gallery.tsx   # Client: swipe/zoom interactions
```

```tsx
// app/products/[id]/page.tsx
import { Suspense } from 'react'
import { getProduct } from '@/lib/data'
import ProductInfo from './_components/product-info'
import Reviews from './_components/reviews'
import AddToCart from './_components/add-to-cart'

export default async function Page({ params }: { params: Promise<{ id: string }> }) {
  const { id } = await params
  const product = await getProduct(id)

  return (
    <>
      <ProductInfo product={product} />
      <AddToCart productId={product.id} price={product.price} />
      <Suspense fallback={<p>Loading reviews...</p>}>
        <Reviews productId={product.id} />
      </Suspense>
    </>
  )
}
```

- The page, info, and reviews are **Server Components** — no JS shipped for them, and data fetching happens next to the database.
- Only `AddToCart` and `ImageGallery` are **Client Components** — small, focused, receiving just the props they need.
- `reviews.tsx` streams in via `<Suspense>` without blocking the rest.

## Common Mistakes

> ⚠️ **Warning:** Putting `'use client'` at the top of a page or layout "to make an error go away." It silently converts that whole subtree into client JavaScript. Find the one component that actually needs state/effects and mark only that.

> ⚠️ **Warning:** Calling `useState`, `useEffect`, or `onClick` in a Server Component. This produces an error telling you the hook only works in a Client Component — the fix is to extract the interactive piece into its own `'use client'` file.

> ⚠️ **Warning:** Importing a Server Component into a Client Component file instead of passing it as `children`. It gets pulled into the client bundle and loses its server-only abilities.

> ⚠️ **Warning:** Passing a function (like an `onClick` defined in a Server Component) as a prop to a Client Component. Plain functions aren't serializable — define the handler inside the Client Component, or use a Server Action.

> ⚠️ **Warning:** Wrapping the entire `<html>` in a provider. Put providers as deep as possible so more of the tree can stay static.

> ⚠️ **Warning:** Forgetting that Client Components are still pre-rendered on the server. Code that reads `window` at the top level of a Client Component will crash during that pre-render — move it into `useEffect` or load the component with `dynamic(..., { ssr: false })`.

---

## Quick Revision

- In the App Router, every component is a **Server Component by default**; adding `'use client'` at the top of a file opts into a Client Component whose JavaScript is shipped to the browser.
- Use Server Components for data fetching, secrets, and reducing bundle size; use Client Components for state, event handlers, effects, custom hooks, and browser-only APIs.
- On the server, Server Components render into the **RSC Payload** (rendered output, placeholders for Client Components, and their props), and Client Components plus that payload are used to prerender HTML; on the client, the HTML shows instantly, the payload reconciles the trees, and JavaScript hydrates Client Components.
- Client Components are **still pre-rendered on the server** during the initial load — `'use client'` controls what JavaScript ships, not whether the server renders it.
- `'use client'` declares a **boundary**: everything the file imports and directly renders joins the client bundle, so push the directive down to small interactive leaves like a search box or like button instead of marking whole pages or layouts.
- Props passed from server to client must be **serializable** and are visible to the user, so pass minimal data — never raw database rows, and never plain functions (Server Actions are the exception).
- To nest server-rendered UI inside client-managed UI, pass Server Components as **`children`** (or other props) rather than importing them into the Client Component file — this keeps them as Server Components.
- React context isn't supported in Server Components: create a Client Component provider that accepts `children`, render it as deep in the tree as possible, and wrap third-party components that use client-only features in a small `'use client'` re-export file.
- Use `import 'server-only'` (and `client-only` for browser-only code) to turn accidental wrong-environment imports into build-time errors, and remember unprefixed environment variables become empty strings in the client bundle.
- Hydration mismatches come from time/random values, `window` checks, or invalid HTML nesting during render; fix them by rendering a stable first pass and updating in `useEffect`, and load browser-dependent libraries with `next/dynamic` and `ssr: false` from a Client Component.