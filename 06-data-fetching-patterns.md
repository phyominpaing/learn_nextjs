# Next.js 06 — Data Fetching Patterns

Data fetching in the App Router means getting data from an API, database, or file into your components — and the big shift from classic React is that most of it now happens **on the server, inside the component itself**, using plain `async`/`await`.

## Table of Contents

- [Primer: Server vs Client Components](#primer-server-vs-client-components)
- [Where Can You Fetch Data?](#where-can-you-fetch-data)
- [Fetching in Server Components with `fetch`](#fetching-in-server-components-with-fetch)
- [Fetching with an ORM or Database](#fetching-with-an-orm-or-database)
- [Handling Errors Correctly](#handling-errors-correctly)
- [Fetch Where It's Needed: Request Memoization](#fetch-where-its-needed-request-memoization)
- [`React.cache` for Non-`fetch` Data](#reactcache-for-non-fetch-data)
- [Is `fetch` Cached by Default?](#is-fetch-cached-by-default)
- [Streaming: `loading.tsx` vs `<Suspense>`](#streaming-loadingtsx-vs-suspense)
- [Sequential vs Parallel Fetching (Waterfalls)](#sequential-vs-parallel-fetching-waterfalls)
- [Preloading Data](#preloading-data)
- [Fetching Data in Client Components](#fetching-data-in-client-components)
- [Handling Sensitive Data](#handling-sensitive-data)
- [A Real-World Data Layer Structure](#a-real-world-data-layer-structure)
- [Common Mistakes](#common-mistakes)
- [Quick Revision](#quick-revision)

---

## Primer: Server vs Client Components

You need two definitions before data fetching makes sense (we'll do a full deep dive on them in the next note):

- **Server Component** — the **default** in the App Router. Runs only on the server; its code is never sent to the browser. It can be `async`, can talk to databases directly, and can read secrets.
- **Client Component** — a file that starts with `'use client'`. Its code is sent to the browser and hydrated, so it can use `useState`, `useEffect`, and event handlers like `onClick`.

```tsx
// Server Component (default) — no directive needed
export default async function Page() {
  const posts = await getPosts() // runs on the server
  return <PostList posts={posts} />
}
```

```tsx
// Client Component — opt in with the directive
'use client'
import { useState } from 'react'

export function LikeButton() {
  const [liked, setLiked] = useState(false)
  return <button onClick={() => setLiked(!liked)}>{liked ? '♥' : '♡'}</button>
}
```

> 💡 **Tip:** Rule of thumb — start every component as a Server Component and only add `'use client'` to the smallest piece that truly needs interactivity or browser APIs.

## Where Can You Fetch Data?

| Location | Best for | Notes |
|---|---|---|
| **Server Component** | Initial page data, SEO content, anything touching a DB or secrets | The recommended default — fewer round trips, smaller bundles |
| **Client Component** (`use` API, SWR, React Query) | Live/polling data, user-driven filters, data after interaction | Runs in the browser, so no secrets, and the user sees a loading state |
| **Route Handler** (`route.ts`) | Your own HTTP API for the browser or external callers | Covered in the API note |
| **Server Action** | **Mutations** (create/update/delete) triggered from forms or buttons | Covered in the Server Actions note — not meant for reads |

## Fetching in Server Components with `fetch`

```tsx
// app/blog/page.tsx
export default async function Page() {
  const res = await fetch('https://api.vercel.app/blog')
  const posts = await res.json()

  return (
    <ul>
      {posts.map((post: { id: string; title: string }) => (
        <li key={post.id}>{post.title}</li>
      ))}
    </ul>
  )
}
```

To fetch on the server you only need to do two things: make the component an `async` function, and `await` the call. There is no `useEffect`, no `useState` for loading flags, and no `getServerSideProps`.

> 💡 **Tip:** During development you can log every server-side `fetch` in your terminal by configuring the `logging` option in `next.config.ts` — very useful for spotting duplicate or slow requests.

## Fetching with an ORM or Database

```tsx
// app/blog/page.tsx
import { db, posts } from '@/lib/db'

export default async function Page() {
  const allPosts = await db.select().from(posts)

  return (
    <ul>
      {allPosts.map((post) => (
        <li key={post.id}>{post.title}</li>
      ))}
    </ul>
  )
}
```

Because this code never reaches the browser, **credentials and query logic stay private**. (The `db` import above is illustrative — it could be Prisma, Drizzle, or any database client.)

> ⚠️ **Warning:** "It runs on the server" does **not** mean "it's automatically authorized." You must still check that the current user is allowed to see the data. See [Handling Sensitive Data](#handling-sensitive-data).

## Handling Errors Correctly

```tsx
async function getPosts() {
  const res = await fetch('https://api.example.com/posts')

  if (!res.ok) {
    // fetch does NOT throw on 404/500 — you must check yourself
    throw new Error(`Failed to fetch posts: ${res.status}`)
  }

  return res.json()
}
```

- **`fetch` only rejects on network failure**, not on HTTP error statuses — a `500` response resolves normally with `res.ok === false`.
- Throwing inside a Server Component bubbles up to the nearest **`error.tsx`** boundary (see nextjs-03), which renders your fallback UI.
- For "not found" cases, call `notFound()` instead of throwing, so the user sees a 404 rather than a generic error.

## Fetch Where It's Needed: Request Memoization

```tsx
// Both components call getUser(id) — the network request only happens ONCE per render pass
async function Header({ id }: { id: string }) {
  const user = await getUser(id)
  return <h1>{user.name}</h1>
}

async function Sidebar({ id }: { id: string }) {
  const user = await getUser(id) // reuses the result above
  return <aside>{user.email}</aside>
}
```

**Request memoization** means identical `fetch` calls (same URL and options) made while rendering a single request are **deduplicated automatically** — React reuses the first result.

- This lets each component fetch **exactly the data it needs**, instead of fetching everything at the top and "prop-drilling" it down through ten components.
- Memoization only lasts for the **duration of one request's render** — it is not a long-term cache.

## `React.cache` for Non-`fetch` Data

Automatic memoization only works for `fetch`. For database/ORM calls, wrap the function in React's `cache`:

```ts
// app/lib/user.ts
import { cache } from 'react'
import { db, eq, users } from '@/lib/db'

export const getUser = cache(async (id: string) => {
  return db.query.users.findFirst({ where: eq(users.id, id) })
})
```

Now `getUser('1')` can be called from a layout, a page, and three components in the same request, and the database is queried once.

> 💡 **Tip:** `React.cache` is scoped to **one request only** — each new request starts with an empty memo, so there's no risk of leaking one user's data into another's response.

## Is `fetch` Cached by Default?

> ⚠️ **Warning:** This changed between versions and is a classic source of confusion with old tutorials.

| Next.js version | Default `fetch` behavior |
|---|---|
| 13 and 14 | Cached by default (like `force-cache`) |
| 15 and later (including current docs) | **Not cached by default** — each request fetches fresh data |

In current Next.js, an uncached `fetch` in a Server Component also **blocks the page from rendering until it completes**. You have two ways to deal with that:

1. **Cache** the result — via `{ next: { revalidate: N } }` (classic model) or the `'use cache'` directive (Cache Components model), both covered in nextjs-05 and the upcoming caching note.
2. **Stream** it — wrap the fetching component in `<Suspense>` so the rest of the page renders immediately.

## Streaming: `loading.tsx` vs `<Suspense>`

Server-side fetching is **blocking by default**: one slow request delays the whole route. **Streaming** fixes this by sending the page in chunks, so fast parts appear immediately and slow parts fill in later.

**Option 1 — `loading.tsx`** streams the **entire page** while it loads:

```tsx
// app/blog/loading.tsx
export default function Loading() {
  return <div>Loading...</div>
}
```

Behind the scenes, Next.js wraps your `page.tsx` in a `<Suspense>` boundary using this file as the fallback. On navigation, the user instantly sees the layout plus this loading UI.

**Option 2 — `<Suspense>`** streams **individual components**, giving you granular control:

```tsx
// app/blog/page.tsx
import { Suspense } from 'react'
import BlogList from '@/components/BlogList'
import BlogListSkeleton from '@/components/BlogListSkeleton'

export default function BlogPage() {
  return (
    <div>
      {/* Sent to the client immediately */}
      <header>
        <h1>Welcome to the Blog</h1>
      </header>

      {/* Streams in when its data is ready */}
      <Suspense fallback={<BlogListSkeleton />}>
        <BlogList />
      </Suspense>
    </div>
  )
}
```

| Approach | Granularity | Best for |
|---|---|---|
| `loading.tsx` | Whole route segment | Quick, simple page-level loading state |
| `<Suspense>` | Single component | Mixed pages where only some parts are slow |

> ⚠️ **Warning (senior-level gotcha):** A **layout** that reads `cookies()`, `headers()`, or does an uncached fetch will **not** fall back to a same-segment `loading.tsx` — instead it blocks navigation until it finishes. Fix it by wrapping that specific access in its own `<Suspense>` boundary, or by moving the fetch into `page.tsx`. This is why the docs recommend placing `<Suspense>` as close to the uncached data as possible.

> 💡 **Tip (SEO):** Bots and crawlers are served differently from browsers — Next.js waits for data to finish and sends the **fully rendered page** instead of streaming it, so streaming doesn't hurt your SEO.

**Designing good loading states:** use skeletons that mimic the final layout, or show a meaningful piece of the future screen (a title or cover image), not just a generic spinner.

## Sequential vs Parallel Fetching (Waterfalls)

A **request waterfall** happens when requests run one after another even though they could have run together — it's one of the most common performance bugs in data-heavy pages.

```text
Sequential:  [getArtist 300ms] -> [getAlbums 300ms]           = 600ms total
Parallel:    [getArtist 300ms]
             [getAlbums 300ms]                                = ~300ms total
```

**Sequential (sometimes unavoidable)** — the second request genuinely needs the first one's result:

```tsx
export default async function Page({ params }: { params: Promise<{ username: string }> }) {
  const { username } = await params
  const artist = await getArtist(username)       // must finish first...

  return (
    <>
      <h1>{artist.name}</h1>
      <Suspense fallback={<div>Loading...</div>}>
        <Playlists artistID={artist.id} />        {/* ...because this needs artist.id */}
      </Suspense>
    </>
  )
}
```

**Accidentally sequential (the bug)** — two independent `await`s in a row:

```tsx
// ❌ getAlbums waits for getArtist even though it doesn't need it
const artist = await getArtist(username)
const albums = await getAlbums(username)
```

**Parallel (the fix)** — start both requests first, then await them together:

```tsx
export default async function Page({ params }: { params: Promise<{ username: string }> }) {
  const { username } = await params

  // Initiate both requests immediately
  const artistData = getArtist(username)
  const albumsData = getAlbums(username)

  const [artist, albums] = await Promise.all([artistData, albumsData])

  return (
    <>
      <h1>{artist.name}</h1>
      <Albums list={albums} />
    </>
  )
}
```

- By default, **layouts and pages already render in parallel**, so each route segment starts fetching as early as possible. Waterfalls mostly come from `await`s **inside one component**.
- Requests start the moment the function is **called**, not when it's `await`ed — that's why creating the promises first works.
- If you can't avoid a sequential dependency, put the dependent part inside `<Suspense>` so the rest of the page isn't blocked, and make sure the first request is as fast as possible (or cache it).

> ⚠️ **Warning:** With `Promise.all`, if **one** request fails, the **whole** thing fails. Use `Promise.allSettled` when you want partial results — for example, showing the artist even if the albums request errors.

## Preloading Data

**Preloading** starts a request *early*, before some other blocking work finishes, so it runs in parallel with that work.

```tsx
// app/item/[id]/item.tsx
async function getItem(id: string) {
  const res = await fetch(`https://api.example.com/items/${id}`)
  return res.json()
}

export const preload = (id: string) => {
  void getItem(id) // start the request, deliberately don't await it
}

export default async function Item({ id }: { id: string }) {
  const item = await getItem(id) // reuses the in-flight request
  return <div>{item.name}</div>
}
```

```tsx
// app/item/[id]/page.tsx
import Item, { preload } from './item'
import { checkIsAvailable } from '@/app/lib/data'

export default async function Page({ params }: { params: Promise<{ id: string }> }) {
  const { id } = await params

  preload(id)                                  // item request starts now...
  const isAvailable = await checkIsAvailable(id) // ...while this runs

  return isAvailable ? <Item id={id} /> : null
}
```

How it works:

- `void` simply tells JavaScript "I'm intentionally ignoring this promise's result."
- When `<Item>` renders and calls `getItem(id)` again, the **memoized** `fetch` (or a `React.cache`-wrapped function) returns the already-running request — no duplicate network call.
- The data function **must deduplicate** matching calls for this to work: plain `fetch` does it automatically, ORM calls need `React.cache`, and with Cache Components you can use `'use cache'`.

> 💡 **Tip:** Keep the `preload` function in the same file as the component that consumes the data, so the dependency is obvious if someone later moves or deletes the component.

## Fetching Data in Client Components

There are two main approaches.

**1. React's `use` API — stream a promise from the server**

```tsx
// app/blog/page.tsx (Server Component)
import Posts from '@/app/ui/posts'
import { Suspense } from 'react'

export default function Page() {
  // Don't await — pass the promise itself
  const posts = getPosts()

  return (
    <Suspense fallback={<div>Loading...</div>}>
      <Posts posts={posts} />
    </Suspense>
  )
}
```

```tsx
// app/ui/posts.tsx (Client Component)
'use client'
import { use } from 'react'

export default function Posts({
  posts,
}: {
  posts: Promise<{ id: string; title: string }[]>
}) {
  const allPosts = use(posts) // suspends until resolved

  return (
    <ul>
      {allPosts.map((post) => (
        <li key={post.id}>{post.title}</li>
      ))}
    </ul>
  )
}
```

The server **starts** the request immediately, the page streams the shell, and the Client Component unwraps the promise with `use()` once it resolves.

**2. A community library — SWR or TanStack (React Query)**

```tsx
'use client'
import useSWR from 'swr'

const fetcher = (url: string) => fetch(url).then((r) => r.json())

export default function BlogPage() {
  const { data, error, isLoading } = useSWR('https://api.vercel.app/blog', fetcher)

  if (isLoading) return <div>Loading...</div>
  if (error) return <div>Error: {error.message}</div>

  return (
    <ul>
      {data.map((post: { id: string; title: string }) => (
        <li key={post.id}>{post.title}</li>
      ))}
    </ul>
  )
}
```

These libraries give you client-side caching, background revalidation, deduplication, retries, and refetch-on-focus — things you'd otherwise hand-write with `useEffect`.

| Use this | When |
|---|---|
| Server Component `await` | Initial content, SEO, DB access (the default) |
| `use()` + Suspense | Server starts the fetch early, Client Component renders it |
| SWR / React Query | Polling, live data, user-triggered refetches, mutations with optimistic UI |

## Handling Sensitive Data

Server Components blur the line between server and client, so it's easy to **accidentally leak data**. Two principles keep you safe:

1. **A Server Component should only ever see data the current user is authorized to see.**
2. **Anything passed from a Server Component to a Client Component as props is sent to the browser.**

**The Data Access Layer (DAL)** — the officially recommended approach for new projects: one dedicated server-only module that handles all data access and authorization.

```ts
// app/lib/dal.ts
import 'server-only'
import { cache } from 'react'
import { cookies } from 'next/headers'
import { db } from '@/lib/db'

export const getCurrentUser = cache(async () => {
  const sessionId = (await cookies()).get('session')?.value
  if (!sessionId) return null

  const user = await db.user.findBySession(sessionId) // illustrative query

  if (!user) return null

  // Return a minimal DTO — only what the UI needs, never the whole DB row
  return { id: user.id, name: user.name }
})
```

- **`import 'server-only'`** makes the build **fail** if this file is ever imported into a Client Component — protecting against "environment poisoning." Install it with `npm install server-only`.
- **DTO (Data Transfer Object)** — a small, purpose-built object containing only the fields the caller needs. Returning `{ id, name }` instead of the full user row means `passwordHash` or `email` can't leak by accident.
- **Only the DAL reads `process.env`** — secrets stay in one auditable place.

```tsx
// ❌ Leaks: the entire user object (including sensitive fields) is serialized to the browser
const user = await db.user.findUnique({ where: { id } })
return <ProfileCard user={user} />   // ProfileCard is a 'use client' component

// ✅ Safe: pass only what the client needs
const user = await getCurrentUser()  // returns a minimal DTO
return <ProfileCard user={user} />
```

**Environment variables:**

| Variable | Available where |
|---|---|
| `DATABASE_URL` (no prefix) | Server only |
| `NEXT_PUBLIC_API_URL` | Server **and** inlined into the browser bundle at build time |

> ⚠️ **Warning:** Never put a secret in a `NEXT_PUBLIC_` variable — everything with that prefix is visible to anyone who opens the browser's DevTools.

> ⚠️ **Warning:** Hiding UI is not authorization. A Client Component check like `if (user.isAdmin)` only controls what's *displayed*; always enforce permissions in the server-side code that actually reads the data.

**React Taint APIs** (experimental, enabled via `experimental.taint` in `next.config.ts`) can mark specific objects or values as "never send to the client" — but the docs describe this as **an extra layer of defense**, not a replacement for filtering data in your DAL.

## A Real-World Data Layer Structure

A clean convention many production teams use:

```text
app/
├── lib/
│   ├── dal.ts            # server-only: auth-aware data access, returns DTOs
│   ├── data.ts           # server-only: shared read queries (wrapped in cache())
│   └── definitions.ts    # shared TypeScript types
├── blog/
│   ├── page.tsx          # Server Component: calls data functions
│   ├── loading.tsx
│   └── _components/
│       ├── post-list.tsx       # Server Component
│       └── like-button.tsx     # 'use client' — tiny interactive leaf
```

Coding standards worth adopting:

- **Colocate fetching with the component that needs it** (memoization makes this free).
- **Keep data functions in `lib/`**, never inline raw queries in many pages.
- **Wrap non-`fetch` reads in `React.cache`**, and mark server data modules with `'server-only'`.
- **Return DTOs**, not raw database rows.
- **Add `<Suspense>` + a skeleton** around anything slow, close to where the slow data is read.
- **Check `res.ok`** and handle errors via `error.tsx` / `notFound()`.

## Common Mistakes

> ⚠️ **Warning:** Using `useEffect` + `fetch` in a Client Component for initial page data that could have been fetched on the server — it adds a client-side waterfall (download JS → hydrate → *then* fetch) and hurts SEO and load time.

> ⚠️ **Warning:** Writing `await a(); await b();` for independent requests — the single most common cause of slow Server Component pages. Start both, then `Promise.all`.

> ⚠️ **Warning:** Assuming `fetch` is cached by default because an older tutorial said so. In current Next.js it isn't — cache explicitly where you want it.

> ⚠️ **Warning:** Passing a full database row to a Client Component as props — everything in that object ends up in the page's HTML/payload.

> ⚠️ **Warning:** Forgetting that a layout reading `cookies()` can block navigation instead of showing `loading.tsx` — wrap the access in its own `<Suspense>`.

---

## Quick Revision

- In the App Router, the default and recommended place to fetch data is inside **async Server Components** using plain `await` — with `fetch`, an ORM, or a database client — with no `useEffect` or `getServerSideProps` needed.
- `fetch` doesn't throw on 404/500, so always check `res.ok`; thrown errors bubble to the nearest `error.tsx`, and missing resources should call `notFound()`.
- Identical `fetch` calls in one request are **memoized** automatically, so each component can fetch its own data; for ORM/database calls, wrap the function in `React.cache`, which is also scoped to a single request.
- In current Next.js (15+), `fetch` is **not cached by default** and blocks rendering until it finishes — cache it explicitly or stream it; Next.js 13/14 tutorials that say otherwise are outdated.
- **Streaming** breaks a page into chunks: `loading.tsx` streams a whole route segment, while `<Suspense>` streams individual components — and `<Suspense>` placed close to the slow/runtime data is preferred, especially because a layout reading `cookies()` won't fall back to `loading.tsx`.
- **Waterfalls** come from sequential `await`s of independent requests; fix them by starting all requests first and awaiting with `Promise.all` (or `Promise.allSettled` for partial failure tolerance), and use `<Suspense>` when a sequential dependency is unavoidable.
- **Preloading** calls a data function with `void` early so its request runs in parallel with other blocking work; it only works because matching calls are deduplicated by memoization or `React.cache`.
- In Client Components, fetch with React's `use()` on a promise passed from the server, or with SWR / React Query for polling, live data, and user-driven refetching.
- Protect sensitive data with a server-only **Data Access Layer** (`import 'server-only'`) that authorizes access and returns minimal DTOs; remember that props passed to Client Components reach the browser, `NEXT_PUBLIC_` variables are public, and hiding UI is never a substitute for server-side authorization.