# Next.js 05 — Rendering Strategies Deep Dive

Rendering strategy is the decision of **when** a page's HTML gets built — at build time, at request time, or in the browser — and it's the single biggest lever you have over a Next.js app's speed, SEO, and server cost.

## Table of Contents

- [Rendering: The Big Picture](#rendering-the-big-picture)
- [Static Rendering](#static-rendering)
- [Dynamic Rendering](#dynamic-rendering)
- [What Triggers Dynamic Rendering](#what-triggers-dynamic-rendering)
- [Client-Side Rendering in the App Router](#client-side-rendering-in-the-app-router)
- [Mapping to the Classic Vocabulary: SSR / SSG / ISR / CSR](#mapping-to-the-classic-vocabulary-ssr--ssg--isr--csr)
- [Controlling Rendering: Route Segment Config](#controlling-rendering-route-segment-config)
- [Incremental Static Regeneration (ISR) in Practice](#incremental-static-regeneration-isr-in-practice)
- [On-Demand Revalidation](#on-demand-revalidation)
- [Partial Prerendering (PPR)](#partial-prerendering-ppr)
- [The Newer Model: Cache Components & `use cache`](#the-newer-model-cache-components--use-cache)
- [Choosing a Strategy for Real Routes](#choosing-a-strategy-for-real-routes)
- [Common Mistakes](#common-mistakes)
- [Quick Revision](#quick-revision)

---

## Rendering: The Big Picture

Every route in the App Router is rendered using one of two fundamental modes — everything else (SSG, ISR, PPR) is a refinement of these two:

```text
Static Rendering  -> HTML built ahead of time (build time, or on a schedule)
Dynamic Rendering -> HTML built fresh for every single incoming request
```

Next.js **automatically decides** which mode a route uses, based on the code you write inside it — you rarely set this with a single on/off switch; you influence it through what APIs and fetch options your route touches.

## Static Rendering

With **Static Rendering**, a route's HTML is generated **ahead of time** — either once at `next build`, or periodically through revalidation (covered below). The result is cached and reused for every visitor.

```tsx
// app/about/page.tsx — a fully static page
export default function AboutPage() {
  return <h1>About Us</h1>
}
```

This page has no dynamic data, no `cookies()`, no `searchParams` — Next.js detects that and pre-renders it once at build time. Every visitor gets the exact same cached HTML.

**Why this is the default-preferred mode:** serving a static file is about as fast as a server can respond — no database query, no computation, just bytes going out. It's also trivially cacheable on a CDN at the edge, close to the user.

## Dynamic Rendering

With **Dynamic Rendering**, HTML is generated **at request time**, specifically for that one request.

```tsx
// app/dashboard/page.tsx
import { cookies } from 'next/headers'

export default async function DashboardPage() {
  const cookieStore = await cookies()
  const session = cookieStore.get('session')
  const user = await getUserFromSession(session)

  return <h1>Welcome back, {user.name}</h1>
}
```

Reading `cookies()` here means the response legitimately depends on *who's asking* — there's no single static HTML file that could be correct for every visitor, so Next.js renders this route fresh, per request.

## What Triggers Dynamic Rendering

A route becomes dynamic the moment it uses a **Dynamic API** — these are APIs whose return value can only be known once an actual request comes in:

| Dynamic API | What it reads |
|---|---|
| `cookies()` | The incoming request's cookies |
| `headers()` | The incoming request's headers |
| `connection()` | Waits for an actual incoming connection before continuing |
| `searchParams` (as a page prop) | The URL's query string on this specific request |
| `fetch(url, { cache: 'no-store' })` | Explicitly opts a fetch out of caching |

> 💡 **Tip:** Dynamic rendering is **contagious upward within the request path, not automatically downward** — if any component actually *reads* one of these APIs during render, the whole route segment it's part of opts into dynamic rendering for that request.

> ⚠️ **Warning:** Just *importing* `cookies` or `headers` doesn't trigger dynamic rendering — only actually **calling and reading** the value during render does. Components only opt into dynamic rendering when the dynamic value is genuinely accessed.

## Client-Side Rendering in the App Router

CSR still exists and is still the right tool for data that's **personal to the browser session** and doesn't need SEO — think a live notification count, a client-only search filter, or a chart that updates as the user interacts.

```tsx
'use client'
import { useEffect, useState } from 'react'

export default function LiveNotifications() {
  const [count, setCount] = useState(0)

  useEffect(() => {
    const id = setInterval(() => {
      fetch('/api/notifications/count')
        .then((res) => res.json())
        .then((data) => setCount(data.count))
    }, 5000)
    return () => clearInterval(id)
  }, [])

  return <span>{count} new</span>
}
```

> 💡 **Tip:** In real Next.js apps, prefer a data-fetching library like **SWR** or **TanStack Query** over raw `useEffect` + `fetch` for client-side data — they handle caching, revalidation, and race conditions for you. We'll use SWR in the data fetching note.

The key architectural idea in the App Router: **a Server Component (static or dynamic) can render a Client Component as a child**, and only that child's JavaScript ships to the browser and hydrates — the rest of the page stays server-rendered. This mixing is normal and expected, not an exception.

## Mapping to the Classic Vocabulary: SSR / SSG / ISR / CSR

You'll hear SSR/SSG/ISR constantly in job interviews, blog posts, and older tutorials. Here's how they map onto the two fundamental modes above:

| Classic term | Modern Next.js equivalent | When HTML is built |
|---|---|---|
| **SSG** (Static Site Generation) | Static Rendering, no revalidation | Once, at `next build` |
| **SSR** (Server-Side Rendering) | Dynamic Rendering | Every request |
| **ISR** (Incremental Static Regeneration) | Static Rendering **with** a `revalidate` interval | Once, then re-generated in the background on a timer |
| **CSR** (Client-Side Rendering) | A Client Component fetching after hydration | In the browser, after JS loads |

> 💡 **Tip:** When a job description or interviewer says "SSG" or "SSR," they mean exactly the Static/Dynamic Rendering concepts above — Next.js's own docs have shifted vocabulary over the years, but the underlying behavior is consistent.

## Controlling Rendering: Route Segment Config

Beyond letting Next.js auto-detect, you can **explicitly force** a route's rendering mode by exporting special config variables from a `page.tsx`, `layout.tsx`, or `route.ts`:

```tsx
// app/products/page.tsx
export const dynamic = 'force-static'
// 'auto' | 'force-dynamic' | 'error' | 'force-static'
```

| Value | Meaning |
|---|---|
| `'auto'` (default) | Next.js decides automatically based on what APIs you use |
| `'force-dynamic'` | Always render on every request, no caching, equivalent to calling a Dynamic API everywhere |
| `'force-static'` | Force static rendering; `cookies()`, `headers()`, and `searchParams` will return empty values instead of making the route dynamic |
| `'error'` | Forces fully static rendering and throws a build **error** if any component tries to use a Dynamic API — useful to guarantee a route can never accidentally become dynamic |

Other useful segment config exports:

```tsx
export const revalidate = 3600   // re-generate this static page at most once per hour
export const fetchCache = 'auto' // 'auto' | 'force-cache' | 'force-no-store' | ...
export const runtime = 'nodejs'  // 'nodejs' | 'edge'
```

- **`revalidate`** — sets the maximum age (in seconds) for the static cache of this whole segment (this is how you do ISR at the segment level, not just per-fetch).
- **`fetchCache`** — overrides the default caching behavior for every `fetch()` call in this segment.
- **`runtime`** — choose between the full **Node.js runtime** (access to all Node APIs, slightly higher cold-start) or the lightweight **Edge runtime** (faster cold starts, runs geographically closer to users, but a restricted API surface — no native Node modules).

> ⚠️ **Warning:** `'force-*'` and `'only-*'` fetch-cache options exist precisely to **guarantee** a whole route is fully static or fully dynamic — mixing a `'force-dynamic'` segment with a `fetchCache: 'only-cache'` fetch inside it is a contradiction Next.js will flag.

## Incremental Static Regeneration (ISR) in Practice

ISR gives you the performance of static pages **without** a full rebuild every time content changes — Next.js serves the cached page instantly, then regenerates it in the background after the `revalidate` window passes.

```tsx
// app/blog/[slug]/page.tsx
export default async function Page({ params }: { params: Promise<{ slug: string }> }) {
  const { slug } = await params
  const post = await fetch(`https://api.example.com/posts/${slug}`, {
    next: { revalidate: 3600 }, // regenerate at most once per hour
  }).then((res) => res.json())

  return <article>{post.content}</article>
}
```

```text
Request 1 (t=0)     -> cache miss, Next.js renders and caches the page
Request 2 (t=30min) -> served instantly from cache (still "fresh")
Request 3 (t=61min) -> served instantly from the STALE cache, then regenerated in the background
Request 4 (t=61min+1s) -> served the newly regenerated, fresh page
```

> 💡 **Tip:** This is called **stale-while-revalidate** behavior — users (almost) never wait for a rebuild; they get the old version instantly while the new one generates behind the scenes.

## On-Demand Revalidation

Instead of waiting for a timer, you can invalidate a cached page the instant its underlying data actually changes — typically from inside a Server Action after a database write:

```tsx
// app/actions.ts
'use server'
import { revalidatePath, revalidateTag } from 'next/cache'

export async function publishPost(id: string) {
  await db.post.update({ where: { id }, data: { published: true } })

  revalidatePath('/blog')        // invalidate one specific path
  revalidateTag('posts')         // or invalidate every fetch tagged 'posts'
}
```

```tsx
// tagging a fetch so it can be targeted later
const posts = await fetch('https://api.example.com/posts', {
  next: { tags: ['posts'] },
})
```

- **`revalidatePath(path)`** — invalidates the cache for one specific route.
- **`revalidateTag(tag)`** — invalidates every cached `fetch()` across the whole app that was tagged with that string, no matter which route it lives in — very useful when the same data (e.g., "posts") is fetched in multiple places.

## Partial Prerendering (PPR)

**Partial Prerendering** lets a single route combine a **static shell** with **dynamic, per-request content**, delivered together in one response — solving the old all-or-nothing trade-off between Static and Dynamic Rendering.

```tsx
// app/product/[id]/page.tsx
import { Suspense } from 'react'
import { Reviews } from './reviews'

export default function ProductPage({ params }: { params: Promise<{ id: string }> }) {
  return (
    <div>
      {/* Static: prerendered into the shell, served instantly */}
      <ProductInfo params={params} />

      {/* Dynamic: streamed in after the shell, per request */}
      <Suspense fallback={<ReviewsSkeleton />}>
        <Reviews params={params} />
      </Suspense>
    </div>
  )
}
```

How it works:

1. At build time (or during revalidation), Next.js prerenders everything it can — including the `<Suspense>` boundary's **fallback** — into a static shell.
2. That shell is served **instantly** on every request.
3. The dynamic parts (anything inside `<Suspense>` that reads a Dynamic API) start **streaming in from the server in parallel**, filling in their slot in the shell as soon as they're ready — without the client having to make a second round trip.

> ⚠️ **Warning:** PPR is enabled per-route with `export const experimental_ppr = true` and requires the `ppr` flag in `next.config.ts`. Treat it as an advanced, evolving feature — check the current docs before shipping it, since its configuration has changed across recent Next.js versions.

## The Newer Model: Cache Components & `use cache`

Next.js is actively moving toward a more explicit caching model called **Cache Components**, built around a new `'use cache'` directive. This is worth knowing conceptually for a senior role, even if most production codebases you'll touch still use the classic `export const revalidate` model above.

```tsx
// app/components/bookings.tsx
import { cacheLife, cacheTag } from 'next/cache'

async function getBookings(userId: string) {
  'use cache'
  cacheLife('hours')       // how long this cache entry stays fresh
  cacheTag('bookings', userId) // a tag this entry can later be invalidated by

  return db.bookings.findMany({ where: { userId } })
}
```

- **`'use cache'`** — placed at the top of a file, component, or async function, marks its output as cacheable — similar in spirit to `'use client'`, but for caching instead of execution environment.
- **`cacheLife(profile)`** — sets how long a cache entry is considered fresh, using named profiles (`'seconds'`, `'minutes'`, `'hours'`, etc.) or a custom config.
- **`cacheTag(...tags)`** — tags a cache entry so it can be invalidated later via `revalidateTag`, same idea as fetch tags above but working for any cached function output, not just `fetch()` calls.
- Enabling the `cacheComponents` flag in `next.config.ts` makes **Partial Prerendering the default behavior** for the whole app, rather than something you opt into per-route.

> 💡 **Tip:** Think of the classic model (`export const revalidate`, `fetch(..., { next: { revalidate } })`) as **route/fetch-level** caching, and Cache Components (`'use cache'`) as **component/function-level** caching — a more granular, composable successor. Both are valid to know; check which one a given codebase has adopted before writing new code in it.

## Choosing a Strategy for Real Routes

A practical, senior-level decision guide:

| Route type | Recommended strategy |
|---|---|
| Marketing/About/Terms pages | Static Rendering (default, no config needed) |
| Blog posts, product pages with occasional updates | Static Rendering + `revalidate` (ISR) |
| Blog posts updated the instant an editor hits "publish" | Static Rendering + on-demand `revalidatePath`/`revalidateTag` |
| User dashboard, account settings | Dynamic Rendering (reads `cookies()` naturally) |
| Product page with static info + live stock/price | Partial Prerendering, or static shell + Client Component for the live part |
| Live chat, real-time notification badge | Client-Side Rendering (CSR) inside a small Client Component |

## Common Mistakes

> ⚠️ **Warning:** Adding `'force-dynamic'` to a route "just to be safe" when it has no real per-user data — this throws away free performance and CDN caching for no reason. Reach for it only when the route genuinely must be fresh per request.

> ⚠️ **Warning:** Forgetting that reading `searchParams` on a Server Component page automatically makes that route dynamic — if you only needed it for a minor UI tweak, consider moving that logic into a small Client Component with `useSearchParams()` instead, keeping the rest of the page static.

> ⚠️ **Warning:** Confusing `revalidate` (time-based, automatic) with `revalidatePath`/`revalidateTag` (on-demand, triggered by your code) — they solve different problems and are often used together: a long `revalidate` as a safety net, plus on-demand revalidation for instant updates after a known data change.

---

## Quick Revision

- Every route is fundamentally either **Static Rendering** (HTML built ahead of time) or **Dynamic Rendering** (HTML built per request) — Next.js decides automatically based on whether your code touches a Dynamic API (`cookies()`, `headers()`, `searchParams`, or an uncached `fetch`).
- The classic terms map cleanly onto this: SSG = static with no revalidation, SSR = dynamic, ISR = static with a `revalidate` interval, CSR = a Client Component fetching after hydration in the browser.
- You can explicitly force a mode with `export const dynamic = 'force-static' | 'force-dynamic' | 'error' | 'auto'`, and control cache lifetime with `export const revalidate = <seconds>`.
- ISR gives you stale-while-revalidate behavior: users get the cached page instantly, and Next.js regenerates it in the background once it goes stale.
- `revalidatePath()` and `revalidateTag()` let you invalidate cached content on-demand (e.g., right after a database write in a Server Action), rather than waiting for a timer.
- Partial Prerendering (PPR) combines a static shell with streamed-in dynamic content inside `<Suspense>` boundaries — the modern answer to "part of this page is static, part needs to be fresh."
- Cache Components (`'use cache'`, `cacheLife`, `cacheTag`) is the newer, more granular, component/function-level caching model Next.js is moving toward — know it exists, but check which model (classic or Cache Components) an existing codebase has adopted before writing new caching code.
- Decide rendering strategy per route based on real requirements — don't default to `force-dynamic` everywhere; most content benefits from static rendering plus revalidation.