# Next.js 04 — Advanced Routing (Parallel Routes, Intercepting Routes & Static Params)

Advanced routing in Next.js lets you render multiple independent sections of a page at once and build shareable, URL-aware modals — patterns that used to require heavy client-side state management before the App Router existed natively.

## Table of Contents

- [Parallel Routes — The Core Idea](#parallel-routes--the-core-idea)
- [Slots Are Not URL Segments](#slots-are-not-url-segments)
- [`default.tsx` — Handling Unmatched Slots](#defaulttsx--handling-unmatched-slots)
- [Soft Navigation vs Hard Navigation Behavior](#soft-navigation-vs-hard-navigation-behavior)
- [Conditional Routes (Role-Based UI)](#conditional-routes-role-based-ui)
- [Tab Groups Inside a Slot](#tab-groups-inside-a-slot)
- [Reading the Active Segment with `useSelectedLayoutSegment`](#reading-the-active-segment-with-useselectedlayoutsegment)
- [Intercepting Routes — The Core Idea](#intercepting-routes--the-core-idea)
- [The `(.)` Convention Explained](#the-.-convention-explained)
- [Full Walkthrough: A Real Shareable Login Modal](#full-walkthrough-a-real-shareable-login-modal)
- [Closing the Modal Correctly](#closing-the-modal-correctly)
- [Independent Loading & Error States](#independent-loading--error-states)
- [`generateStaticParams` — Pre-Building Dynamic Routes](#generatestaticparams--pre-building-dynamic-routes)
- [Common Mistakes](#common-mistakes)
- [Quick Revision](#quick-revision)

---

## Parallel Routes — The Core Idea

**Parallel Routes** let a single layout render more than one independent page at the same time, each navigable on its own. Defined using **named slots** with the `@folder` convention:

```text
app/
└── dashboard/
    ├── layout.tsx
    ├── page.tsx
    ├── @team/
    │   └── page.tsx
    └── @analytics/
        └── page.tsx
```

Each slot is passed to the shared layout as a **prop**, alongside the implicit `children` prop (which represents `page.tsx`):

```tsx
// app/dashboard/layout.tsx
export default function DashboardLayout({
  children,
  team,
  analytics,
}: {
  children: React.ReactNode
  team: React.ReactNode
  analytics: React.ReactNode
}) {
  return (
    <>
      {children}
      {team}
      {analytics}
    </>
  )
}
```

- **`children`** is an **implicit slot** — `app/dashboard/page.tsx` behaves exactly like `app/dashboard/@children/page.tsx`, you just never have to name it.
- Each slot streams and renders independently, which is why dashboards with multiple widgets are the textbook use case for this feature.

## Slots Are Not URL Segments

> ⚠️ **Warning:** `@team` and `@analytics` never appear in the URL. A page at `app/dashboard/@analytics/views/page.tsx` is reachable at `/dashboard/views` — **not** `/dashboard/@analytics/views`. Slots are a rendering/organizational concept only.

One important consequence: **you cannot have one slot statically rendered and a sibling slot dynamically rendered at the same route level** — if any slot at that level needs dynamic rendering, all slots at that level become dynamic together.

## `default.tsx` — Handling Unmatched Slots

Say `@team` has a `/settings` page but `@analytics` does not:

```text
app/dashboard/@team/settings/page.tsx      -> exists
app/dashboard/@analytics/settings/page.tsx -> does NOT exist
```

- **On client-side navigation** to `/dashboard/settings`, Next.js keeps `@analytics` showing whatever it was already showing — no error.
- **On a hard reload** at `/dashboard/settings`, Next.js has no memory of `@analytics`'s previous state, so it looks for `app/dashboard/@analytics/default.tsx`. If that file doesn't exist, it renders a **404** for the whole route.

```tsx
// app/dashboard/@analytics/default.tsx
export default function Default() {
  return null
}
```

> 💡 **Tip:** Because `children` is an implicit slot too, you often need a `default.tsx` at the top level of a parallel-routes setup as well — not just inside the named `@slot` folders.

## Soft Navigation vs Hard Navigation Behavior

| Navigation type | What happens to unmatched slots |
|---|---|
| **Soft** (client-side, via `<Link>` or `router.push`) | Next.js performs a **partial render** — the slot that matches the new URL updates, every other slot keeps showing its last active subpage |
| **Hard** (full page load / browser refresh) | Next.js cannot recover unmatched slot state — it renders `default.tsx`, or `404` if that file doesn't exist |

This is the mechanism that lets a dashboard's `@team` tab stay exactly where the user left it while they click around in `@analytics`.

## Conditional Routes (Role-Based UI)

A powerful, senior-level pattern: render an entirely different slot based on application logic like a user's role.

```tsx
// app/dashboard/layout.tsx
import { checkUserRole } from '@/lib/auth'

export default function Layout({
  user,
  admin,
}: {
  user: React.ReactNode
  admin: React.ReactNode
}) {
  const role = checkUserRole()
  return role === 'admin' ? admin : user
}
```

> ⚠️ **Warning:** Both slots render **on the server** regardless of which one the layout ultimately returns — `@admin/page.tsx` still executes its data fetches for every visitor, admin or not. The `if` statement only decides what the *browser receives*, not what *runs on the server*. Never rely on this pattern alone for authorization — check permissions inside each slot's own data-fetching code (a Data Access Layer), not just at the layout level.

## Tab Groups Inside a Slot

A slot can have its own nested `layout.tsx`, letting users navigate within that slot independently of the rest of the page — perfect for building tabs:

```tsx
// app/dashboard/@analytics/layout.tsx
import Link from 'next/link'

export default function AnalyticsLayout({ children }: { children: React.ReactNode }) {
  return (
    <>
      <nav>
        <Link href="/page-views">Page Views</Link>
        <Link href="/visitors">Visitors</Link>
      </nav>
      <div>{children}</div>
    </>
  )
}
```

## Reading the Active Segment with `useSelectedLayoutSegment`

To know which subpage is currently active *inside a specific slot* (from the parent layout's perspective), pass the slot's name to `useSelectedLayoutSegment`:

```tsx
'use client'
import { useSelectedLayoutSegment } from 'next/navigation'

export default function DashboardLayout({ auth }: { auth: React.ReactNode }) {
  const loginSegment = useSelectedLayoutSegment('auth')
  // loginSegment === "login" when the user is at app/@auth/login
}
```

---

## Intercepting Routes — The Core Idea

**Intercepting Routes** let you load a route **within the current layout**, while masking the URL bar — the classic example being an Instagram-style photo modal: clicking a photo in a feed opens it as an overlay, but the URL still updates to `/photo/123`, and refreshing that URL loads the **full, standalone** photo page instead of the modal.

```text
Click from feed (soft navigation)  -> modal opens over the feed, URL becomes /photo/123
Direct visit / refresh (hard nav)  -> full /photo/123 page renders, no modal, no feed behind it
```

This solves a problem that used to require manual client-side routing logic: **the same URL renders differently depending on how the user arrived at it** — and Next.js gives you this for free through file-system convention alone.

## The `(.)` Convention Explained

Intercepting folders are named relative to **route segments**, not literal file-system nesting:

| Convention | Matches |
|---|---|
| `(.)folder` | A segment at the **same level** |
| `(..)folder` | A segment **one level above** |
| `(..)(..)folder` | A segment **two levels above** |
| `(...)folder` | A segment from the **root** `app` directory |

```text
app/
├── feed/
│   ├── page.tsx           -> /feed
│   └── (..)photo/
│       └── [id]/
│           └── page.tsx    -> intercepts /photo/[id] while at /feed
└── photo/
    └── [id]/
        └── page.tsx         -> /photo/[id] (the real, standalone page)
```

> ⚠️ **Warning:** These dots are based on the **route hierarchy**, not folder depth on disk. If a slot (`@folder`) sits between two segments, it does not count as a "level" for `(..)` purposes, since slots aren't route segments at all.

## Full Walkthrough: A Real Shareable Login Modal

This is the canonical pattern combining both features — worth studying closely, since it comes up constantly in real products (auth modals, image lightboxes, quick-view product cards).

**Step 1 — Create the real, standalone page** (what renders on direct visit/refresh):

```tsx
// app/login/page.tsx
import { Login } from '@/app/ui/login'

export default function Page() {
  return <Login />
}
```

**Step 2 — Add a `default.tsx` that returns `null`** inside the `@auth` slot, so nothing renders there when the modal isn't active:

```tsx
// app/@auth/default.tsx
export default function Default() {
  return null
}
```

**Step 3 — Intercept `/login` inside the `@auth` slot**, wrapping it in a `<Modal>`:

```tsx
// app/@auth/(.)login/page.tsx
import { Modal } from '@/app/ui/modal'
import { Login } from '@/app/ui/login'

export default function Page() {
  return (
    <Modal>
      <Login />
    </Modal>
  )
}
```

**Step 4 — Render the `@auth` slot in the root layout**, alongside `children`:

```tsx
// app/layout.tsx
import Link from 'next/link'

export default function Layout({
  auth,
  children,
}: {
  auth: React.ReactNode
  children: React.ReactNode
}) {
  return (
    <>
      <nav>
        <Link href="/login">Open modal</Link>
      </nav>
      <div>{auth}</div>
      <div>{children}</div>
    </>
  )
}
```

Now clicking the "Open modal" link performs a **soft navigation** — the `@auth` slot intercepts it and shows the modal, URL becomes `/login`, feed stays visible behind it. Refreshing that same URL performs a **hard navigation** — the interception is bypassed, and the real standalone `app/login/page.tsx` renders full-screen instead.

> 💡 **Tip:** Keep the `<Modal>` wrapper itself as a Client Component (it needs `useRouter` to close), but let its children (like `<Login>`) stay Server Components when possible — this keeps most of the modal's actual content server-rendered, which is better for bundle size and data fetching.

## Closing the Modal Correctly

```tsx
// app/ui/modal.tsx
'use client'
import { useRouter } from 'next/navigation'

export function Modal({ children }: { children: React.ReactNode }) {
  const router = useRouter()
  return (
    <>
      <button onClick={() => router.back()}>Close modal</button>
      <div>{children}</div>
    </>
  )
}
```

`router.back()` is preferred because it returns the user to exactly where they were, without the modal's history entry, and it naturally supports the "close on back-navigation, reopen on forward-navigation" behavior users expect from real modals.

> ⚠️ **Warning:** If the user might navigate to the modal from a link **elsewhere** (not just back), you also need a catch-all fallback so the slot doesn't keep showing a stale modal:

```tsx
// app/@auth/[...catchAll]/page.tsx
export default function CatchAll() {
  return null
}
```

## Independent Loading & Error States

Because each parallel route streams independently, you can give each slot its **own** `loading.tsx` and `error.tsx` — one slow widget on a dashboard won't block the others from rendering:

```text
app/dashboard/@analytics/loading.tsx   -> only covers the analytics widget
app/dashboard/@team/error.tsx          -> only covers the team widget
```

## `generateStaticParams` — Pre-Building Dynamic Routes

For a dynamic route like `app/blog/[slug]/page.tsx`, `generateStaticParams` tells Next.js **which exact URLs to pre-render at build time**, instead of generating them on-demand for every request.

```tsx
// app/blog/[slug]/page.tsx
export async function generateStaticParams() {
  const posts = await fetch('https://api.example.com/posts').then((res) => res.json())

  return posts.map((post) => ({
    slug: post.slug,
  }))
}

export default async function Page({
  params,
}: {
  params: Promise<{ slug: string }>
}) {
  const { slug } = await params
  const post = await getPost(slug)
  return <h1>{post.title}</h1>
}
```

**What this buys you:** instead of every visitor triggering a server render, Next.js builds `/blog/hello-world`, `/blog/my-second-post`, etc. as static HTML during `next build` — as fast as serving a plain file.

- **`generateStaticParams` replaces `getStaticPaths`** from the old Pages Router — same job, new API.
- Runs **before** the matching layouts/pages during `next build`, and on-demand during `next dev` as you navigate.
- **`dynamicParams`** (a route segment config option) controls what happens if someone visits a slug that *wasn't* in the returned list — by default, Next.js will still render it on-demand; set `export const dynamicParams = false` to make Next.js return a 404 instead for anything not pre-generated.
- For **nested dynamic segments**, a child segment's `generateStaticParams` runs once per param set the parent generated, and receives the parent's resolved `params` as an argument — letting you build combinations like `/products/[category]/[product]` correctly.

> ⚠️ **Warning:** You must always return an array from `generateStaticParams`, even an empty one — returning `undefined` or nothing is a common source of confusing build errors.

## Common Mistakes

> ⚠️ **Warning:** Trying to use `@slot` names as actual URL path segments — remember slots are invisible to the URL and exist purely to let a layout render multiple independent trees.

> ⚠️ **Warning:** Forgetting `default.tsx` for parallel route slots — without it, a hard refresh on a route that doesn't match a slot's current state renders a full 404 instead of gracefully falling back.

> ⚠️ **Warning:** Treating conditional parallel routes as an authorization mechanism — both branches still execute server-side; you must still gate the actual data access, not just the rendered branch.

> ⚠️ **Warning:** Counting `(..)` levels by file-system folders instead of route segments — a `@slot` folder in the path does not count as a level to climb past.

---

## Quick Revision

- Parallel Routes use `@folder` to define named **slots** that a shared layout receives as props (alongside the implicit `children` slot) and renders simultaneously — great for dashboards where independent widgets load and error independently.
- Slots never appear in the URL, and all slots at one route level must share the same rendering mode (all static or all dynamic).
- `default.tsx` provides the fallback for a slot on hard navigation/refresh when its state can't be recovered — without it, Next.js renders a 404.
- Soft (client-side) navigation preserves each slot's last active state; hard navigation (full reload) resets unmatched slots to their `default.tsx`.
- Conditional parallel routes (e.g., admin vs. user dashboards) render **both** branches on the server regardless of which one is shown — real authorization must happen in each slot's own data access code.
- Intercepting Routes use `(.)`, `(..)`, `(..)(..)`, and `(...)` — based on route-segment distance, not file-system depth — to render one route's content inside another route's layout while masking the URL.
- The canonical modal pattern combines Parallel + Intercepting Routes: a real standalone page for direct visits, a `default.tsx` returning `null`, an intercepted page wrapped in `<Modal>`, and `router.back()` (plus a catch-all fallback) to close it correctly.
- `generateStaticParams` pre-builds specific dynamic route URLs at build time (replacing the old `getStaticPaths`), must always return an array (even empty), and pairs with `dynamicParams` to control what happens for URLs it didn't generate.