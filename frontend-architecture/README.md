# Frontend Architecture

Where a page gets rendered (server, build time, client) and how the browser's security model
constrains what a frontend can and can't do across origins. Frontend architecture decisions are
system design decisions — rendering strategy affects backend load shape, CDN cacheability, and
even which team owns which failure mode.

## In this folder

- `rendering-strategies.md` — SSR, CSR, SSG, ISR: what actually happens on first load vs.
  subsequent navigation, and the SEO/perf/infra trade-offs between them.
- `cross-site-security.md` — the same-origin policy, CORS, CSRF, XSS, and cookie attributes —
  the model that makes "why is this API call blocked" and "how do I stop this attack" the same
  underlying question.

## Staff-engineer notes

- Rendering strategy is rarely all-or-nothing across a real product — a marketing/landing page
  (SEO-critical, mostly static) and a logged-in dashboard (highly interactive, no SEO need)
  inside the same app often warrant different strategies, and modern meta-frameworks (Next.js,
  Nuxt, Remix) support mixing them per-route.
- Frontend security bugs (XSS, CSRF, leaking tokens to third-party scripts) are disproportionately
  caused by convenience shortcuts under deadline pressure (dangerouslySetInnerHTML, disabling
  CSRF checks "temporarily," storing a JWT in localStorage instead of an httpOnly cookie) — treat
  these as the default things to check in a frontend-touching design review, not edge cases.
