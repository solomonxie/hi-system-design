# Rendering Strategies: SSR, CSR, SSG, ISR

## CSR — Client-Side Rendering

Server sends a near-empty HTML shell plus a JS bundle; the browser downloads, parses, and
executes the JS to render the page, then fetches data via API calls. Standard SPA model
(React/Vue/Angular without a meta-framework).

- **Pro**: rich, app-like interactivity once loaded; navigation between routes is fast (no full
  page reload); backend only needs to serve a JSON API, not render HTML.
- **Con**: slow first paint (blank/spinner until JS downloads + executes + fetches data — this
  is Time to Interactive dominating the experience); bad default SEO (crawlers historically
  didn't execute JS, though major ones do now with caveats and delay); every user pays the full
  JS bundle cost even for content that's the same for everyone.

## SSR — Server-Side Rendering

Server renders the full HTML for the requested page on each request (running the same
component tree server-side), sends complete HTML immediately, then the client "hydrates" it
(attaches event listeners, JS takes over) to become interactive.

- **Pro**: fast first paint (real content immediately, no blank shell); good SEO by default
  (crawlers get real HTML); works without JS for the initial view.
- **Con**: every request costs server render time and compute (can't just serve a static file);
  Time to Interactive still waits on hydration, and a page that's visible-but-not-yet-hydrated is
  a real UX gap (clicking before hydration completes does nothing) — this is what React's
  concurrent/streaming SSR and selective/progressive hydration exist to shrink.

## SSG — Static Site Generation

HTML is rendered once, at *build* time, not per-request — the output is a static file served
from a CDN. Fastest possible serving (no render cost per request, maximal cacheability), but the
content is only as fresh as the last build/deploy.

- **Best fit**: content that changes rarely relative to traffic volume (marketing pages, docs,
  blog posts) — pay the render cost once, serve millions of times for free from the edge.
- **Bad fit**: content that's per-user or changes frequently — you'd need to rebuild
  constantly, or per-user pages you can't statically generate at all.

## ISR — Incremental Static Regeneration

SSG's staleness problem, fixed: pages are generated statically but can be regenerated on a
schedule or on-demand after deploy, without a full site rebuild — serve the (possibly slightly
stale) static version immediately, revalidate/regenerate in the background, next request gets
the fresh version. Popularized by Next.js; the general idea (stale-while-revalidate) predates any
one framework and shows up in HTTP caching semantics too.

## Picking one

| Need | Fit |
|---|---|
| SEO-critical, content rarely changes | SSG (+ ISR if it changes occasionally) |
| SEO-critical, content changes per-request/per-user | SSR |
| Rich interactivity, SEO doesn't matter (logged-in app) | CSR |
| High-traffic content that changes moderately | ISR |

## Staff-engineer notes

- These aren't mutually exclusive within one product — route-level mixing is the modern default
  (marketing pages SSG, dashboard CSR, a product page SSR or ISR) via meta-frameworks that support
  per-route rendering mode.
- SSR shifts real, ongoing compute cost onto your infrastructure that CSR/SSG don't have — this
  belongs in the capacity-planning conversation (render cost per request × expected traffic), not
  just the frontend team's decision in isolation.
- Hydration mismatches (server-rendered HTML doesn't match what the client would render — often
  from using non-deterministic values like `Date.now()` or `Math.random()` during render, or
  reading browser-only APIs during SSR) are a recurring, hard-to-debug class of bug specific to
  SSR/hydration architectures — worth calling out explicitly in a frontend design review.
