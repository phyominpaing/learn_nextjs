# Next.js 02 — Setup, `create-next-app` & Project Structure

Project structure in Next.js is not just a folder-naming convention — the file system itself **is** the router, so understanding it correctly from day one prevents confusing bugs later.

## Table of Contents

- [System Requirements](#system-requirements)
- [Creating a Project with `create-next-app`](#creating-a-project-with-create-next-app)
- [What Gets Generated](#what-gets-generated)
- [App Router vs Pages Router](#app-router-vs-pages-router)
- [Top-Level Folders & Files](#top-level-folders--files)
- [Routing File Conventions](#routing-file-conventions)
- [Component Rendering Hierarchy](#component-rendering-hierarchy)
- [Dynamic Routes (Bracket Syntax)](#dynamic-routes-bracket-syntax)
- [Route Groups & Private Folders](#route-groups--private-folders)
- [Parallel & Intercepting Routes (Preview)](#parallel--intercepting-routes-preview)
- [Colocation — Why Next.js Is "Unopinionated"](#colocation--why-nextjs-is-unopinionated)
- [Ways to Organize a Project](#ways-to-organize-a-project)
- [Common Beginner Misunderstandings](#common-beginner-misunderstandings)
- [Quick Revision](#quick-revision)

---

## System Requirements

As of the current Next.js major version:

- **Node.js 20.9 or later**
- **macOS, Windows (including WSL), or Linux**
- Modern browsers with zero extra config: Chrome 111+, Edge 111+, Firefox 111+, Safari 16.4+

> ⚠️ **Warning:** Older tutorials online often say "Node 18 is fine." Always check the [official installation docs](https://nextjs.org/docs/app/getting-started/installation) for the *current* minimum version before starting a new project — Next.js raises this requirement over time as it adopts newer JavaScript runtime features.

## Creating a Project with `create-next-app`

```bash
npx create-next-app@latest
```

This single command scaffolds a complete project. You'll be prompted with something like:

```txt
What is your project named? my-app
Would you like to use the recommended Next.js defaults?
    Yes, use recommended defaults - TypeScript, ESLint, Tailwind CSS, App Router, AGENTS.md
    No, reuse previous settings
    No, customize settings - Choose your own preferences
```

- **Recommended defaults** — picks TypeScript, ESLint, Tailwind CSS, the App Router, and generates an `AGENTS.md` file (guidance file for AI coding agents working in the repo). This is the fastest path and what most new projects should choose.
- **Customize settings** — walks you through each choice individually:

```txt
Would you like to use TypeScript? No / Yes
Which linter would you like to use? ESLint / Biome / None
Would you like to use Tailwind CSS? No / Yes
Would you like your code inside a `src/` directory? No / Yes
Would you like to use App Router? (recommended) No / Yes
Would you like to use Turbopack? (recommended) No / Yes
Would you like to customize the import alias (`@/*` by default)? No / Yes
```

**What each prompt means:**

- **TypeScript** — adds static typing to JavaScript, catching many bugs before you even run the code. Strongly recommended for anything beyond a toy project.
- **Linter (ESLint / Biome)** — a tool that scans your code for style issues and common mistakes automatically. Biome is a newer, faster alternative to ESLint.
- **Tailwind CSS** — a utility-first CSS framework (covered in depth in the styling note).
- **`src/` directory** — an optional folder that holds all your application code (`app/`, `components/`, etc.), keeping it separate from root-level config files like `next.config.js`.
- **App Router (recommended)** — the modern routing system built on React Server Components. This series focuses on the **App Router**, since it's the current and future direction of Next.js.
- **Turbopack (recommended)** — Next.js's new Rust-based bundler, used by default for `next dev` (and increasingly for `next build`). It replaces Webpack for much faster local development.
- **Import alias (`@/*`)** — lets you write `import Button from '@/components/Button'` instead of `import Button from '../../../components/Button'`.

> 💡 **Tip:** Always choose **App Router**, not Pages Router, for new projects in 2025+. Pages Router still exists for legacy support, but all new features (Server Components, Server Actions, streaming) are built for the App Router first.

## What Gets Generated

After answering the prompts:

```bash
cd my-app
npm run dev
```

- `npm run dev` — starts the local dev server (using Turbopack by default) at `http://localhost:3000`.
- `npm run build` — creates an optimized production build.
- `npm run start` — runs that production build locally (to test before deploying).
- `npm run lint` — runs ESLint/Biome across your codebase.

```json
{
  "scripts": {
    "dev": "next dev",
    "build": "next build",
    "start": "next start",
    "lint": "eslint"
  }
}
```

> 💡 **Tip:** `next dev` gives you **Fast Refresh** — save a file and see the change in the browser almost instantly, without losing component state. This is one of the biggest daily productivity wins of the framework.

## App Router vs Pages Router

Next.js currently ships with two routing systems living side by side. You will see both mentioned constantly in docs, blog posts, and job listings, so know the difference clearly.

| Aspect | App Router (`app/`) | Pages Router (`pages/`) |
|---|---|---|
| **Status** | Current, actively developed | Legacy, still supported |
| **Rendering model** | React Server Components by default | Client Components by default |
| **Routing folder** | `app/` | `pages/` |
| **Data fetching** | `fetch()` directly inside components, Server Actions | `getServerSideProps`, `getStaticProps`, `getInitialProps` |
| **Layouts** | Native, nested `layout.tsx` files | Manual, via a shared `_app.tsx` |
| **Loading/Error UI** | Built-in `loading.tsx` / `error.tsx` conventions | Manual implementation |
| **Streaming support** | Yes, built-in | No |
| **Best for** | All new projects | Maintaining older codebases |

- **React Server Components (RSC)**: components that render **on the server only** — their code never ships to the browser as JavaScript, which shrinks bundle size and lets you query databases or read secrets directly inside a component.
- **Client Components**: components marked with `"use client"` that run in the browser like traditional React — needed for interactivity (`onClick`, `useState`, etc.).

> ⚠️ **Warning:** A project can technically have *both* an `app/` and a `pages/` folder during migration, but never mix the two routing systems for the **same** URL path — that will cause routing conflicts.

## Top-Level Folders & Files

These live at the root of your project:

| Folder/File | Purpose |
|---|---|
| `app/` | App Router — where your routes, layouts, and pages live |
| `pages/` | Pages Router (legacy alternative to `app/`) |
| `public/` | Static assets served as-is (images, favicon, robots.txt) at the root URL |
| `src/` | Optional folder to hold `app/` and other app code, separated from config files |
| `next.config.js` | Main Next.js configuration file |
| `package.json` | Dependencies and npm scripts |
| `middleware.ts` | Code that runs **before** a request completes (auth checks, redirects, headers) |
| `instrumentation.ts` | Hook for OpenTelemetry / monitoring setup |
| `.env`, `.env.local`, `.env.production` | Environment variable files (never commit `.env.local` to git) |
| `tsconfig.json` | TypeScript configuration |
| `eslint.config.mjs` | ESLint configuration |

> ⚠️ **Warning:** `.env.local` should always be in `.gitignore`. It's the file most likely to hold real secrets (API keys, database URLs) for your local machine.

## Routing File Conventions

Inside `app/`, specific **reserved filenames** each have a special meaning. Next.js looks for these exact names:

| File | Purpose |
|---|---|
| `layout.tsx` | Shared UI that wraps a segment and its children (nav bars, sidebars) — persists across navigation |
| `page.tsx` | Makes a route segment **publicly accessible** at a URL; this is the actual page content |
| `loading.tsx` | Instant loading UI shown while a segment's content streams in (via React Suspense) |
| `error.tsx` | Error boundary UI for a segment when something throws |
| `global-error.tsx` | Root-level error boundary for the whole app |
| `not-found.tsx` | UI shown for 404s within a segment |
| `route.ts` | Defines an **API endpoint** for this segment (no UI, just server logic) |
| `template.tsx` | Like a layout, but re-mounts (resets state) on every navigation instead of persisting |
| `default.tsx` | Fallback UI for parallel route slots when no match exists |

**Key rule:** a folder inside `app/` is just an organizational URL segment — it does **not** become a visitable page until it contains a `page.tsx` or `route.ts` file.

```text
app/
├── page.tsx              -> /
├── blog/
│   ├── page.tsx          -> /blog
│   └── authors/
│       └── page.tsx      -> /blog/authors
```

> 💡 **Tip:** This means you can safely create a folder like `app/blog/_components/` to store helper components right next to the route that uses them — since it has no `page.tsx`, it is never turned into a URL. This is called **colocation**.

## Component Rendering Hierarchy

When multiple special files exist in the same segment, React renders them nested in this fixed order (outer to inner):

```text
layout.tsx
 └── template.tsx
      └── error.tsx (error boundary)
           └── loading.tsx (suspense boundary)
                └── not-found.tsx (error boundary)
                     └── page.tsx (or nested layout.tsx)
```

This nesting is **recursive** — a child segment's whole stack renders *inside* its parent segment's stack, all the way up to the root `layout.tsx`.

## Dynamic Routes (Bracket Syntax)

Square brackets turn a folder name into a URL parameter:

| Folder pattern | Example URL(s) matched | Meaning |
|---|---|---|
| `[slug]` | `/blog/my-first-post` | Single dynamic segment |
| `[...slug]` | `/shop/clothing`, `/shop/clothing/shirts` | **Catch-all** — matches one or more segments |
| `[[...slug]]` | `/docs`, `/docs/getting-started` | **Optional catch-all** — also matches the base path with zero segments |

```tsx
// app/blog/[slug]/page.tsx
export default async function Page({
  params,
}: {
  params: Promise<{ slug: string }>
}) {
  const { slug } = await params
  return <h1>Post: {slug}</h1>
}
```

The value is accessed through the `params` prop, which Next.js automatically fills in based on the URL. (We'll use this heavily in the routing note.)

## Route Groups & Private Folders

Two ways to organize `app/` without changing your URLs:

| Syntax | Name | Effect |
|---|---|---|
| `(folderName)` | **Route group** | Groups routes for organization or shared layouts — parentheses are **omitted** from the URL |
| `_folderName` | **Private folder** | Opts a folder (and everything inside it) **out of routing entirely** — never becomes a URL, even with a `page.tsx` inside |

```text
app/
├── (marketing)/
│   ├── layout.tsx        -> layout only for marketing pages
│   └── page.tsx          -> /
├── (shop)/
│   ├── layout.tsx        -> a different layout for shop pages
│   └── cart/
│       └── page.tsx      -> /cart
```

Both `(marketing)` and `(shop)` groups can even define their **own root layout** (their own `<html>`/`<body>` tags), letting one app host completely different UI experiences under different sections.

> 💡 **Tip:** Use route groups when different sections of your site (marketing pages vs. dashboard pages) need different layouts but shouldn't show up as part of the URL.

## Parallel & Intercepting Routes (Preview)

Two advanced patterns you'll meet in full in the next routing note — just recognize the syntax for now:

| Pattern | Meaning |
|---|---|
| `@folder` | **Parallel route** (named slot) — render independent sections simultaneously, e.g., a sidebar and main content updating independently |
| `(.)folder` | **Intercepting route** — intercept a route at the *same* level (e.g., open a photo as a modal without leaving the feed) |
| `(..)folder` | Intercept a route **one level up** |
| `(..)(..)folder` | Intercept a route **two levels up** |
| `(...)folder` | Intercept a route from the **root** |

> ⚠️ **Warning:** Don't try to memorize these deeply yet — they solve very specific UI problems (like Instagram-style modal overlays). We'll build real examples in nextjs-04.

## Colocation — Why Next.js Is "Unopinionated"

Officially, Next.js does **not** force a specific folder structure for your components, hooks, or utilities — only the routing files above are reserved. This means all of these are valid strategies:

- **Outside `app/`** — keep `components/`, `lib/`, `hooks/` at the project root; `app/` is used purely for routing.
- **Inside `app/`** — put shared `components/` and `lib/` folders directly inside `app/`.
- **Split by feature** — keep global shared code at the root of `app/`, but colocate feature-specific components inside their own route folder (e.g., `app/dashboard/_components/`).

> 💡 **Tip:** Pick **one** strategy and stay consistent across the whole project/team. There's no "official right answer" — consistency matters more than which one you pick.

## Ways to Organize a Project

- **`src/` directory** — wraps `app/`, `components/`, etc. inside `src/`, separating app code from root config files (`next.config.js`, `package.json`). Purely organizational, has zero effect on routing or URLs.
- **Multiple root layouts** — remove the single top-level `layout.tsx` and instead give each route group (e.g., `(marketing)` and `(shop)`) its own root layout, useful when sections of a site need entirely different HTML shells.
- **Scoped `loading.tsx`** — wrap a specific page in its own route group just so a loading skeleton applies only to that one page, not its siblings.

## Common Beginner Misunderstandings

> ⚠️ **Warning:** Creating a folder does **not** create a route. `app/dashboard/` with no `page.tsx` inside it is invisible to visitors — it's just an organizational segment.

> ⚠️ **Warning:** Route groups `(name)` and private folders `_name` look similar but do opposite jobs — a route group only hides the folder name *from the URL*; a private folder hides the folder *from the router entirely*.

> ⚠️ **Warning:** Don't confuse `layout.tsx` with `template.tsx`. A layout **persists** its state across navigations between its child routes; a template **remounts from scratch** on every navigation. Use `template.tsx` only when you specifically need fresh state (e.g., re-triggering an enter animation).

---

## Quick Revision

- `create-next-app` scaffolds a full project in one command; for new projects always choose the App Router and Turbopack, since that's where all current Next.js development is focused.
- The App Router (`app/`) is built on React Server Components by default and is the current standard; the Pages Router (`pages/`) is legacy but still supported for older codebases — never mix both routers for the same URL.
- Inside `app/`, folders are just URL organization — a segment only becomes a real, visitable page once it contains a `page.tsx` (or an API endpoint via `route.ts`).
- Reserved filenames (`layout`, `page`, `loading`, `error`, `not-found`, `route`, `template`, `default`) each have one specific job, and they render nested in a fixed hierarchy from `layout` down to `page`.
- Bracket syntax creates dynamic routes: `[slug]` for one segment, `[...slug]` for catch-all, `[[...slug]]` for optional catch-all.
- `(folderName)` route groups organize routes and can add per-section layouts without affecting the URL; `_folderName` private folders remove a folder from routing entirely — they solve different problems despite looking similar.
- `@folder` (parallel routes) and `(.)folder` (intercepting routes) are advanced patterns for things like simultaneous UI sections and modal-style overlays — recognize the syntax now, deep dive comes later.
- Next.js is intentionally unopinionated about where you put non-routing files (components, hooks, utils) thanks to colocation — pick one organizational strategy and stay consistent.