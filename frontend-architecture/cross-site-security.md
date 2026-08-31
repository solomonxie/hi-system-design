# Cross-Site Security: Same-Origin, CORS, CSRF, XSS

## The same-origin policy (the foundation)

Two URLs are the "same origin" only if scheme, host, and port all match exactly. By default, a
script from origin A cannot read a response from origin B (though it can often still *trigger* a
request to B — that gap is what CSRF exploits, below). This one browser rule is the reason CORS,
CSRF tokens, and cookie `SameSite` all need to exist at all.

## CORS — Cross-Origin Resource Sharing

The mechanism that lets a server *opt in* to letting other origins read its responses via
JavaScript. Without it, same-origin policy blocks the read by default.

- Simple requests go out and the browser checks the response's `Access-Control-Allow-Origin`
  header before letting the calling JS read it — the request still happened server-side, only
  the *browser-side read* is blocked without the right header.
- "Non-simple" requests (custom headers, methods other than GET/POST/HEAD, certain content types)
  trigger a **preflight** — the browser sends an `OPTIONS` request first, asking the server what's
  allowed, before sending the real request at all.
- `Access-Control-Allow-Origin: *` is fine for public, unauthenticated APIs; never pair it with
  `Access-Control-Allow-Credentials: true` (browsers reject that combination, for good reason —
  it would let any site read authenticated responses on a user's behalf).

## CSRF — Cross-Site Request Forgery

An attacker's page makes the *victim's browser* send a request to your site, using the victim's
existing cookies (the browser attaches them automatically, same-origin policy or not — cookies
aren't blocked by CORS, only cross-origin *reads* of the response are). If your endpoint trusts
"a valid session cookie was present" as sufficient proof of intent, this is exploitable.

- **Defense: CSRF tokens** — a per-session (or per-form) unpredictable token the server embeds in
  the page and requires back on state-changing requests; an attacker's page can trigger the
  request but can't know the token to include.
- **Defense: `SameSite` cookie attribute** — `Strict`/`Lax` tells the browser not to send the
  cookie on cross-site requests at all (`Lax`, the modern default in most browsers, still allows
  it on top-level navigation like clicking a link, but not on cross-site form posts/fetches).
  This alone closes most CSRF vectors without needing a token, though defense-in-depth (both)
  is still common for sensitive actions.
- **Defense: check the `Origin`/`Referer` header** on state-changing requests as a supplementary
  check — not sufficient alone (headers can be stripped/spoofed in some contexts), but cheap and
  useful alongside the above.

## XSS — Cross-Site Scripting

An attacker gets their own script to execute in the context of your origin — the opposite
direction of CSRF (here the attacker's *code* runs as if it were yours, with access to your
cookies, localStorage, DOM). Three flavors: **stored** (malicious input saved server-side, served
to other users — a comment field rendered unescaped), **reflected** (malicious input in a
request, echoed back into the response unescaped — a search query rendered into the results
page), **DOM-based** (client-side JS itself writes untrusted data into the DOM unsafely, no
server round-trip needed).

- **Defense: escape output by context** — HTML-escape for HTML content, attribute-escape for HTML
  attributes, JS-string-escape for embedding in a `<script>` block; using the wrong escaping for
  the context is itself a common source of bypasses. Modern frameworks (React, Vue, Angular)
  escape by default on interpolation — the risk concentrates in the explicit escape hatches
  (`dangerouslySetInnerHTML`, `v-html`, `[innerHTML]`, manual `document.write`).
- **Defense: Content-Security-Policy (CSP)** header — restricts which script sources the browser
  will execute at all, so even a successful injection often can't run (no inline scripts allowed,
  no unlisted external script origins). The strongest practical mitigation, and worth adopting
  even with good output-escaping discipline as a second layer.
- **httpOnly cookies**: mark session cookies `httpOnly` so JavaScript (including an attacker's
  injected script) cannot read them via `document.cookie` at all — limits what a successful XSS
  can steal, even though it doesn't prevent the XSS itself.

## Staff-engineer notes

- CORS is not a security boundary by itself — it's an opt-in relaxation of a browser-only
  restriction, and it does nothing to stop a non-browser client (curl, a server-to-server call)
  from hitting your API. Authorization/authentication (see `authentication/`) is the actual
  security boundary; CORS is about which *browser-based* origins are allowed to read the
  response.
- Where a token lives changes its exposure: an httpOnly cookie is safe from XSS-driven theft
  but exposed to CSRF (mitigated by `SameSite`/CSRF tokens); a token in localStorage/JS memory is
  immune to CSRF (it's not auto-attached by the browser) but fully exposed to XSS (any injected
  script can just read it). There's no free option — pick based on which risk your app is better
  positioned to mitigate.
- In a design review, "how does this endpoint authenticate a state-changing request, and what
  stops a forged cross-origin request from succeeding" is the specific question that catches
  CSRF gaps before they ship — it's rarely caught by generic "is this endpoint authenticated"
  review, since a forged request *does* carry valid auth (the victim's real cookie).
