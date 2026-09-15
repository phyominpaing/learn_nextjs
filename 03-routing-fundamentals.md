# Next.js 03 — Routing Fundamentals (App Router)

Routing is how Next.js turns your `app/` folder structure into real, navigable URLs, and how users move between those URLs without full page reloads.

> 💡 This series uses the **App Router** exclusively, since it's Next.js's current and actively developed routing system (see nextjs-02 for the App Router vs Pages Router comparison). Every code sample below assumes an `app/` directory project.

## Table of Contents

- [Creating Your First Page](#creating-your-first-page)
- [The Root Layout — Required, Not Optional](#the-root-layout--required-not-optional)
- [Nested Routes](#nested-routes)
- [Nesting Layouts](#nesting-layouts)
- [Dynamic Segments in Practice](#dynamic-segments-in-practice)
- [Reading Search Params](#reading-search-params)
- [Navigating with `<Link>`](#navigating-with-link)
- [Programmatic Navigation with `useRouter`](#programmatic-navigation-with-userouter)
- [Reading the Current URL: `usePathname` & `useParams`](#reading-the-current-url-usepathname--useparams)
- [Building an Active Nav Link](#building-an-active-nav-link)
- [How Prefetching & Client-Side Navigation Actually Work](#how-prefetching--client-side-navigation-actually-work)
- [Redirecting](#redirecting)
- [Handling 404s with `notFound()`](#handling-404s-with-notfound)
- [Loading UI in Practice](#loading-ui-in-practice)
- [Error UI in Practice](#error-ui-in-practice)
- [Common Mistakes](#common-mistakes)
- [Quick Revision](#quick-revision)

---

## Creating Your First Page

A **page** is UI rendered at a specific URL. Add a `page.tsx` file inside `app/` and default-export a component:

```tsx
// app/page.tsx
export default function Page() {
  return <h1>Hello Next.js!</h1>
}
```

This single file becomes the `/` route. That's the entire mental model — no router config file, no route array to maintain by hand.

## The Root Layout — Required, Not Optional

Every App Router project **must** have a root `layout.tsx` directly inside `app/`. Unlike every other layout, it must contain the `<html>` and `<body>` tags, because Next.js doesn't inject them for you at this level.

```tsx
// app/layout.tsx
export default function RootLayout({
  children,
}: {
  children: React.ReactNode
}) {
  return (
    <html lang="en">
      <body>
        <main>{children}</main>
      </body>
    </html>
  )
}
```

- **`children`** is where Next.js injects whatever page or nested layout matches the current URL.
- A layout **preserves state and stays interactive across navigations** within it — it does not re-render just because you clicked a link to a sibling page. This is different from a traditional multi-page app, where every navigation re-runs everything.

> ⚠️ **Warning:** Forgetting the root layout, or forgetting `<html>`/`<body>` inside it, is one of the most common first-project errors — Next.js will throw a build error immediately.

## Nested Routes

A **nested route** is any URL made of multiple segments, like `/blog/my-first-post`. In Next.js:

- **Folders** define the URL segments.
- **Files** (`page.tsx`, `layout.tsx`, etc.) define what actually renders for that segment.

```text
app/
├── page.tsx               -> /
└── blog/
    ├── page.tsx            -> /blog
    └── [slug]/
        └── page.tsx         -> /blog/my-first-post
```

```tsx
// app/blog/page.tsx
import { getPosts } from '@/lib/posts'
import { Post } from '@/ui/post'

export default async function Page() {
  const posts = await getPosts()

  return (
    <ul>
      {posts.map((post) => (
        <Post key={post.id} post={post} />
      ))}
    </ul>
  )
}
```

Notice this is an `async` component that directly `await`s data — no `getStaticProps` or `getServerSideProps` needed. We'll go deep on this pattern in the data fetching note (nextjs-06); for now just recognize that Server Components can be `async` functions.

## Nesting Layouts

Layouts nest automatically based on folder hierarchy — a child segment's layout renders **inside** its parent's layout via `children`.

```tsx
// app/blog/layout.tsx
export default function BlogLayout({
  children,
}: {
  children: React.ReactNode
}) {
  return <section className="blog-shell">{children}</section>
}
```

With this file added, the render order for `/blog/my-first-post` becomes:

```text
app/layout.tsx (root)
 └── app/blog/layout.tsx
      └── app/blog/[slug]/page.tsx
```

> 💡 **Tip:** Put anything that should visually persist across an entire section — a sidebar, a sub-navigation bar, a "back to blog" link — in that section's `layout.tsx` rather than repeating it in every `page.tsx`.

## Dynamic Segments in Practice

Wrap a folder name in square brackets to make it dynamic: `app/blog/[slug]/page.tsx`.

```tsx
// app/blog/[slug]/page.tsx
export default async function BlogPostPage({
  params,
}: {
  params: Promise<{ slug: string }>
}) {
  const { slug } = await params
  const post = await getPost(slug)

  return (
    <div>
      <h1>{post.title}</h1>
      <p>{post.content}</p>
    </div>
  )
}
```

> ⚠️ **Warning:** In current Next.js, `params` (and `searchParams`, below) are delivered as a **Promise**, not a plain object — you must `await` them before use. This is a deliberate API design that lets Next.js start streaming a page's shell before the exact params have resolved. If you're following an older tutorial that destructures `params` directly without awaiting, it was written for an earlier Next.js version.

Layouts within a dynamic segment can access the same `params` prop, so a `layout.tsx` next to that `page.tsx` would also receive `params: Promise<{ slug: string }>`.

## Reading Search Params

Search params (`?filters=active`) are handled differently depending on **where** you need them:

```tsx
// app/page.tsx — Server Component
export default async function Page({
  searchParams,
}: {
  searchParams: Promise<{ [key: string]: string | string[] | undefined }>
}) {
  const filters = (await searchParams).filters
  // use filters to query a database, etc.
}
```

```tsx
// Client Component
'use client'
import { useSearchParams } from 'next/navigation'

export default function FilterBar() {
  const searchParams = useSearchParams()
  const filters = searchParams.get('filters')
  // ...
}
```

| Situation | What to use |
|---|---|
| You need search params to **load data for the page** (pagination, DB filtering) | `searchParams` prop on the Server Component page |
| Search params only affect **client-side behavior** on data you already have | `useSearchParams()` hook |
| You just need to read params inside an event handler, without triggering a re-render | `new URLSearchParams(window.location.search)` |

> ⚠️ **Warning:** Reading `searchParams` on a page opts that page into **dynamic rendering** — Next.js can no longer fully pre-render it at build time, since search params only exist once a real request comes in. Keep this in mind when a page seems slower than expected; we'll cover this trade-off fully in the rendering strategies note.

## Navigating with `<Link>`

`<Link>` is the primary, recommended way to navigate in Next.js. It extends the HTML `<a>` tag with automatic prefetching and client-side navigation.

```tsx
import Link from 'next/link'

export default function Page() {
  return <Link href="/dashboard">Dashboard</Link>
}
```

```tsx
// Dynamic href built from data
import Link from 'next/link'

export default async function Posts() {
  const posts = await getPosts()
  return (
    <ul>
      {posts.map((post) => (
        <li key={post.slug}>
          <Link href={`/blog/${post.slug}`}>{post.title}</Link>
        </li>
      ))}
    </ul>
  )
}
```

Useful props beyond `href`:

- **`replace`** — replaces the current history entry instead of pushing a new one (no "back" step created).
- **`scroll`** — whether to scroll to the top after navigating (defaults to `true`).
- **`prefetch`** — controls whether Next.js preloads the route in the background (defaults to `true` for routes in the viewport).

> 💡 **Tip:** Always reach for `<Link>` first. Only drop down to `useRouter` (next section) when you need to navigate as a *side effect* of some other logic — like redirecting after a form submits successfully.

## Programmatic Navigation with `useRouter`

For navigation triggered by code rather than a click on an anchor — after a successful login, a timeout, a form submission — use the `useRouter` hook from `next/navigation`.

```tsx
'use client'
import { useRouter } from 'next/navigation'

export default function Page() {
  const router = useRouter()

  return (
    <button type="button" onClick={() => router.push('/dashboard')}>
      Dashboard
    </button>
  )
}
```

> ⚠️ **Warning:** `useRouter` is imported from **`next/navigation`** in the App Router — not `next/router`, which is the old Pages Router import. Mixing these up is a very common copy-paste mistake when following mismatched tutorials.

Common `router` methods:

- **`router.push(href)`** — navigate to a new URL, adding a history entry.
- **`router.replace(href)`** — navigate without adding a history entry.
- **`router.back()` / `router.forward()`** — move through browser history.
- **`router.refresh()`** — re-fetch the current route's data from the server without losing client state or scroll position (very useful after a mutation).
- **`router.prefetch(href)`** — manually preload a route (rarely needed — `<Link>` already does this).

## Reading the Current URL: `usePathname` & `useParams`

Two more Client Component hooks, also from `next/navigation`:

```tsx
'use client'
import { usePathname, useParams } from 'next/navigation'

export default function Debug() {
  const pathname = usePathname()   // e.g. "/blog/my-first-post"
  const params = useParams()       // e.g. { slug: "my-first-post" }

  return <p>{pathname}</p>
}
```

- **`usePathname()`** — returns the current URL's path as a plain string. Great for highlighting the active nav item.
- **`useParams()`** — returns the dynamic route params as a plain object, read directly (not a Promise) since it's a Client Component hook reacting to already-resolved client-side state.

## Building an Active Nav Link

A classic real-world pattern combining `usePathname` with `<Link>`:

```tsx
'use client'
import Link from 'next/link'
import { usePathname } from 'next/navigation'

const links = [
  { href: '/', label: 'Home' },
  { href: '/blog', label: 'Blog' },
  { href: '/dashboard', label: 'Dashboard' },
]

export default function NavBar() {
  const pathname = usePathname()

  return (
    <nav>
      {links.map((link) => (
        <Link
          key={link.href}
          href={link.href}
          className={pathname === link.href ? 'active' : ''}
        >
          {link.label}
        </Link>
      ))}
    </nav>
  )
}
```

> 💡 **Tip:** This component must be a **Client Component** (`'use client'`) because `usePathname` relies on browser navigation state — hooks from `next/navigation` only work inside Client Components.

## How Prefetching & Client-Side Navigation Actually Work

This is the part that makes Next.js navigation feel instant, and it's worth understanding at a senior level:

1. **Code-splitting by route segment** — on the server, your app is automatically split so each route only ships the JavaScript it needs.
2. **Prefetching** — any `<Link>` currently in the viewport is automatically prefetched in the background, so its route segment's code (and, depending on the route, its data) is already cached client-side before the user even clicks.
3. **Client-side transition** — when the user clicks, the browser does **not** do a full reload. Next.js swaps only the route segments that changed, re-rendering just that part of the React tree.

```text
Full page reload (traditional): click -> new HTML request -> blank flash -> full re-render
Next.js client-side transition: click -> swap only changed segments -> no full reload, no blank flash
```

> 💡 **Tip:** This is *why* layouts don't re-render on navigation between sibling pages — the layout was never part of what changed, so Next.js just swaps the `children` underneath it.

## Redirecting

Two different tools exist depending on where the redirect happens:

```tsx
// Inside a Server Component, Route Handler, or Server Action
import { redirect } from 'next/navigation'

export default async function Page({ params }: { params: Promise<{ id: string }> }) {
  const { id } = await params
  const post = await getPost(id)

  if (!post) {
    redirect('/blog') // throws internally — stops execution immediately
  }

  return <h1>{post.title}</h1>
}
```

```tsx
// Inside a Client Component, in response to user interaction
'use client'
import { useRouter } from 'next/navigation'

export default function LogoutButton() {
  const router = useRouter()
  return <button onClick={() => router.push('/login')}>Log out</button>
}
```

| Tool | Where it works | Behavior |
|---|---|---|
| `redirect()` (`next/navigation`) | Server Components, Server Actions, Route Handlers | Immediately stops rendering and issues a redirect; works even before any HTML is sent |
| `permanentRedirect()` (`next/navigation`) | Same as above | Same as `redirect()`, but signals a **permanent** (308) redirect for SEO purposes |
| `router.push()` / `router.replace()` | Client Components only | Client-side navigation, not a true HTTP redirect |

> ⚠️ **Warning:** `redirect()` works by throwing a special internal error that Next.js catches — so code written *after* a `redirect()` call never runs. Don't wrap it in a `try/catch` that might accidentally swallow it.

## Handling 404s with `notFound()`

```tsx
import { notFound } from 'next/navigation'

export default async function Page({ params }: { params: Promise<{ slug: string }> }) {
  const { slug } = await params
  const post = await getPost(slug)

  if (!post) {
    notFound() // renders the nearest not-found.tsx
  }

  return <h1>{post.title}</h1>
}
```

```tsx
// app/blog/not-found.tsx
export default function NotFound() {
  return <h2>This post could not be found.</h2>
}
```

`notFound()` renders the closest `not-found.tsx` up the folder tree — if `app/blog/not-found.tsx` doesn't exist, Next.js walks up to `app/not-found.tsx`.

## Loading UI in Practice

Add a `loading.tsx` next to a `page.tsx` to get an instant loading state, automatically wired to React Suspense, while that segment's data is fetched:

```tsx
// app/blog/[slug]/loading.tsx
export default function Loading() {
  return <p>Loading post...</p>
}
```

Next.js shows this **immediately** on navigation, then swaps in the real page once its data-fetching `await`s resolve — no manual `isLoading` state needed.

## Error UI in Practice

```tsx
// app/blog/[slug]/error.tsx
'use client' // error.tsx must always be a Client Component

import { useEffect } from 'react'

export default function Error({
  error,
  reset,
}: {
  error: Error & { digest?: string }
  reset: () => void
}) {
  useEffect(() => {
    console.error(error)
  }, [error])

  return (
    <div>
      <h2>Something went wrong loading this post.</h2>
      <button onClick={() => reset()}>Try again</button>
    </div>
  )
}
```

- `error.tsx` **must** be a Client Component — error boundaries rely on React features only available on the client.
- The `reset` function attempts to re-render the segment that errored, without a full page reload.
- Any uncaught error thrown inside that segment's `page.tsx`, `layout.tsx`, or `loading.tsx` is automatically caught by the nearest `error.tsx`.

## Common Mistakes

> ⚠️ **Warning:** Importing `useRouter` from `next/router` instead of `next/navigation` — the two APIs look similar but belong to different routers (Pages vs App), and mixing them causes confusing runtime errors.

> ⚠️ **Warning:** Forgetting to `await params` / `await searchParams` — since these are Promises, accessing a property directly on them (`params.slug`) instead of awaiting first will not give you the expected string.

> ⚠️ **Warning:** Using `router.push()` inside a Server Component — `useRouter` is a Client Component hook and cannot run on the server. Use `redirect()` there instead.

---

## Quick Revision

- A page is created with `page.tsx`; a shared UI wrapper is created with `layout.tsx` — folders define URL segments, files define what renders for them.
- The root `layout.tsx` is mandatory and must include `<html>` and `<body>` — every other layout nests inside it based on folder hierarchy.
- Dynamic segments use `[slug]` folder names, and in current Next.js both `params` and `searchParams` arrive as **Promises** that must be awaited before use.
- Choose `searchParams` (the prop) when a Server Component needs the value to fetch data, and `useSearchParams()` (the hook) when only a Client Component needs it for client-side behavior — reading `searchParams` opts a page into dynamic rendering.
- `<Link>` is the default way to navigate — it prefetches automatically and enables client-side transitions that swap only the segments that changed, which is why layouts don't re-render across sibling navigations.
- `useRouter`, `usePathname`, and `useParams` all come from `next/navigation` (not `next/router`, which is the legacy Pages Router import) and only work inside Client Components.
- Use `redirect()` / `permanentRedirect()` from `next/navigation` for server-side redirects, and `router.push()` / `router.replace()` for client-side navigation — they are not interchangeable.
- `notFound()` renders the nearest `not-found.tsx`; `loading.tsx` gives instant Suspense-powered loading states; `error.tsx` (always a Client Component) catches uncaught errors in its segment and can `reset()` to retry.