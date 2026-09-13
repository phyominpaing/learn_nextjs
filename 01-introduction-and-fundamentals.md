# Next.js 01 — Introduction & Fundamentals

Next.js is a **React framework** that adds routing, rendering, data fetching, and build tooling on top of React so you can build production-grade web applications without stitching those pieces together yourself.

## Table of Contents

- [Why Frontend Frameworks Exist](#why-frontend-frameworks-exist)
- [Why React Alone Isn't Enough](#why-react-alone-isnt-enough)
- [What Is Next.js?](#what-is-nextjs)
- [SPA vs MPA — The Starting Point](#spa-vs-mpa--the-starting-point)
- [Rendering Strategies Overview](#rendering-strategies-overview)
- [SPA vs SSR — The Core Trade-off](#spa-vs-ssr--the-core-trade-off)
- [Why Use Next.js Instead of Plain React?](#why-use-nextjs-instead-of-plain-react)
- [Next.js vs Remix](#nextjs-vs-remix)
- [How a Next.js App "Thinks"](#how-a-nextjs-app-thinks)
- [Common Beginner Misunderstandings](#common-beginner-misunderstandings)
- [Quick Revision](#quick-revision)

---

## Why Frontend Frameworks Exist

Before frameworks, developers wrote raw HTML/CSS/JavaScript and manually manipulated the DOM (`document.getElementById`, `innerHTML`, etc.) for every UI change. This worked for small sites but broke down as apps grew:

- **State management became messy** — no clear place to store "what the UI currently looks like."
- **DOM updates were manual and error-prone** — you had to remember every place the UI needed to change.
- **Code reuse was hard** — no clean way to build a "Button" once and use it everywhere.

Frameworks like React solved this by introducing:

- **Component-based architecture** — the UI is broken into small, reusable, self-contained pieces.
- **Declarative rendering** — you describe *what* the UI should look like for a given state, and the framework figures out *how* to update the DOM.
- **A virtual DOM** — an in-memory representation of the UI that React diffs against the real DOM to make efficient updates.

> 💡 **Tip:** Think of React as "the UI layer." It solves *component* and *state* problems, but it does **not** ship an opinion on routing, data fetching, or how your app is structured on disk. That gap is exactly what Next.js fills.

## Why React Alone Isn't Enough

Plain React (created with something like Vite or Create React App) gives you components and state, but as soon as you build a real product you need to answer questions React doesn't answer for you:

- How do I navigate between pages/URLs? → **Routing**
- How do I fetch data before the page renders? → **Data fetching**
- How do I make the site show up fast on Google and social previews? → **SEO / rendering strategy**
- How do I split my JavaScript so users don't download the whole app at once? → **Code splitting / bundling**
- How do I organize folders as the app grows to 100+ components? → **Project structure conventions**

You *can* solve all of this yourself with libraries like React Router, a custom Express server, Webpack config, and SEO libraries — but that's a lot of manual wiring and decisions. Next.js bundles opinionated, production-tested answers to all of these.

## What Is Next.js?

**Next.js** is an open-source React framework, built by **Vercel**, that adds:

- **File-based routing** — the folder/file structure under `app/` (or `pages/`) automatically becomes your URL routes. No router library to configure.
- **Multiple rendering strategies** — render pages on the server, at build time, or in the browser, and mix all three in the same app.
- **Built-in data fetching APIs** — fetch data directly inside components that run on the server, with automatic caching.
- **Automatic code splitting & bundling** — each page only ships the JavaScript it actually needs.
- **API routes** — write backend endpoints inside the same project, no separate server required.
- **Image, font, and script optimization** — built-in components (`next/image`, `next/font`, `next/script`) that optimize automatically.
- **Zero-config build tooling** — TypeScript, ESLint, and bundling work out of the box.

```bash
npx create-next-app@latest my-app
```

Running this one command scaffolds a complete, production-ready React project — routing, TypeScript, ESLint, and a build pipeline all pre-wired. (We'll walk through every option this command asks in the next note.)

> 💡 **Tip:** Next.js is not a replacement for React — it's a **superset**. Every React skill you already have (components, hooks, props, state) still applies. Next.js adds structure *around* React.

## SPA vs MPA — The Starting Point

Before comparing rendering strategies, it helps to know the two oldest models of building websites:

| Model | How it works | Example feel |
|---|---|---|
| **MPA** (Multi-Page Application) | Every link click sends a request to the server, which returns a brand-new HTML page. Full page reload every time. | Classic PHP/WordPress sites |
| **SPA** (Single-Page Application) | The browser loads one HTML shell + a JavaScript bundle. JavaScript then swaps content in and out without reloading the page. | Plain React/Vite apps |

- **MPA** — **Multi-Page Application**: simple mental model, great SEO (server sends full content), but slower navigation (full reloads).
- **SPA** — **Single-Page Application**: fast, app-like navigation after the first load, but the *first* load can be slow and empty until JavaScript runs, which historically hurt SEO.

Next.js exists partly to **get the best of both worlds** — the app-like feel of an SPA with the fast-first-paint and SEO benefits of an MPA.

## Rendering Strategies Overview

This is the single most important concept to internalize before going further. Next.js supports four rendering strategies. We'll go deep into each one in a dedicated note (nextjs-05), but you need the vocabulary now:

- **CSR (Client-Side Rendering)** — The browser downloads a mostly-empty HTML page, then JavaScript builds the UI in the browser. This is how plain React (SPA) apps work by default.
- **SSR (Server-Side Rendering)** — The server generates the full HTML **for every request**, sends it to the browser already populated, then React "hydrates" it (attaches interactivity).
- **SSG (Static Site Generation)** — The full HTML is generated **once, at build time**, and reused for every visitor. Fastest possible response since it's just a static file.
- **ISR (Incremental Static Regeneration)** — Like SSG, but Next.js can regenerate a static page in the background after a set time interval, without rebuilding the whole site.

```text
CSR:  Browser -> (empty HTML + JS) -> JS builds UI in browser
SSR:  Browser -> Server builds HTML per request -> Browser -> React hydrates
SSG:  Build time -> HTML file generated once -> Server just serves that file
ISR:  Like SSG, but regenerated on a timer/on-demand after deployment
```

> ⚠️ **Warning:** A very common beginner mistake is assuming Next.js = "always SSR." In reality, Next.js lets you **choose per-page, even per-component**, which strategy to use. This flexibility is Next.js's biggest advantage over plain React.

## SPA vs SSR — The Core Trade-off

Since this is the trade-off that motivates Next.js's existence, here it is side by side:

| Aspect | SPA (CSR) | SSR |
|---|---|---|
| **First paint** | Slower — blank until JS loads and runs | Faster — HTML arrives already filled in |
| **SEO** | Weaker by default (crawlers may not run JS) | Strong — crawlers see full HTML immediately |
| **Server load** | Low — server just serves static files | Higher — server does work on every request |
| **Navigation after load** | Very fast (no full reloads) | Fast, but each new page may hit the server again |
| **Data freshness** | Always fresh (fetched client-side) | Fresh per request |
| **Best for** | Dashboards, admin panels, logged-in apps | Marketing pages, blogs, e-commerce, anything SEO-sensitive |

**Definitions to lock in:**

- **Hydration**: the process where React "attaches" event listeners and interactivity to server-rendered HTML that's already sitting in the browser. The HTML shows up first (fast), then becomes interactive shortly after.
- **SEO (Search Engine Optimization)**: making your pages easy for search engines to read and rank. Full HTML content on first load is a major SEO advantage.
- **TTFB (Time To First Byte)**: how long it takes the browser to receive the first byte of the response — a key performance metric affected by rendering strategy.

## Why Use Next.js Instead of Plain React?

- **You don't build a router from scratch** — folders define routes automatically.
- **You choose rendering strategy per page** — a blog post can be static, a dashboard can be client-rendered, a product page can be server-rendered — all in one app.
- **SEO works out of the box** — because SSR/SSG send full HTML, not an empty shell.
- **Performance optimizations are automatic** — image optimization, font optimization, and code splitting happen without extra config.
- **Full-stack capability** — you can write backend API endpoints in the same codebase (no separate Express server needed for simple APIs).
- **Huge ecosystem + backed by Vercel** — frequent updates, strong documentation, wide industry adoption (used by Netflix, TikTok, Twitch, Nike, and many more).

## Next.js vs Remix

Remix is the other major "full-stack React framework" you'll hear about. Knowing the difference matters for interviews and architecture decisions at senior level.

| Feature | Next.js | Remix |
|---|---|---|
| **Backed by** | Vercel | Shopify |
| **Routing** | File-based (`app/` directory) | File-based (nested routes) |
| **Rendering default** | Mix of SSR/SSG/ISR, flexible per route | Primarily SSR, embraces web standards (fetch/Request/Response) |
| **Data fetching model** | `fetch()` in Server Components, Server Actions | Loaders and Actions per route |
| **Static generation (SSG)** | First-class, strong support | Limited / less emphasized |
| **Deployment** | Optimized for Vercel, but deployable anywhere | Deploy-target agnostic by design |
| **Learning curve** | Slightly gentler for React devs due to bigger community | Steeper if unfamiliar with web platform APIs |
| **Popularity** | Larger ecosystem and community | Smaller but growing, popular for "close to the metal" web fans |

> 💡 **Tip:** You don't need to master Remix — just know it exists and *why* someone might pick it (tighter web-standards philosophy, strong nested routing model). This is common senior-level interview knowledge.

## How a Next.js App "Thinks"

At a high level, mentally model a Next.js app like this:

1. A **request** comes in for a URL (e.g., `/products/42`).
2. Next.js matches that URL to a **file in your folder structure**.
3. Next.js decides **how to render** that file's component — statically, on the server per-request, or leaves it to the client — based on how you wrote the code.
4. The **HTML (and only the needed JavaScript)** is sent to the browser.
5. React **hydrates** the page, making it interactive.

This request-to-response pipeline is the mental model we'll build on for every future topic — routing, data fetching, caching, and rendering all plug into this same flow.

## Common Beginner Misunderstandings

> ⚠️ **Warning:** Next.js is **not** a separate language or a totally different framework from React — you're still writing React components. Next.js adds conventions and tooling around React, not a replacement for it.

> ⚠️ **Warning:** "Server" in Next.js doesn't always mean "a server you manage." On platforms like Vercel, server-side code often runs as serverless functions — you write server code, but you don't necessarily manage a traditional always-on server.

> ⚠️ **Warning:** Choosing SSR everywhere "to be safe" is a common junior mistake. It adds server load and latency where it isn't needed — e.g., a static About page never needs to be re-rendered on every request.

---

## Quick Revision

- Next.js is a React framework (by Vercel) that adds routing, rendering strategies, data fetching, and build tooling on top of plain React — it does not replace React knowledge, it builds on it.
- Frontend frameworks exist to solve state management, DOM updates, and code reuse problems that raw JavaScript makes painful at scale.
- MPA reloads the full page per navigation; SPA loads once and swaps content via JavaScript — Next.js aims to combine SPA-like navigation with MPA-like fast first loads and SEO.
- The four rendering strategies to know by name are CSR, SSR, SSG, and ISR — each trades off first-paint speed, SEO, server load, and data freshness differently; Next.js lets you pick per page.
- Hydration is the step where React attaches interactivity to already-rendered HTML sent from the server.
- Plain React alone doesn't give you routing, data fetching conventions, or SEO-friendly rendering — you'd have to wire all of that up manually; Next.js provides it out of the box.
- Remix is Next.js's closest competitor (Shopify-backed, web-standards-first, strong nested routing) — worth knowing conceptually, not urgent to learn deeply right now.
- Avoid two junior mistakes: assuming Next.js always means SSR, and using SSR everywhere "just to be safe" when a static or client-rendered page would be more efficient.