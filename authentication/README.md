# Authentication

Proving who's making a request. (Not authorization — *what they're allowed to do* — which is a
related but separate concern, usually layered on top of whatever identity auth establishes.)

## Session-based vs. token-based

- **Session cookie**: on login, server creates a session record (server-side state, typically in
  Redis/a DB) and gives the client an opaque session ID in an httpOnly cookie. Every request, the
  server looks up the session ID to get the user. Easy to revoke instantly (delete the session
  record) — the server is the source of truth. Cost: every request needs a lookup (or a cache),
  and it's inherently stateful, which complicates horizontal scaling (any instance needs access
  to the shared session store) — see also `distributed-systems/caching.md`.
- **Token-based (JWT)**: the token itself carries the claims (user ID, roles, expiry), signed by
  the server so it can be verified without a database lookup — genuinely stateless, which scales
  trivially across instances/services with no shared session store. Cost: **you cannot revoke a
  JWT before it expires** without reintroducing state (a blocklist, or short expiry + refresh
  tokens) — this is the recurring gotcha people miss, treating JWTs as if they were revocable
  sessions.
- The common real pattern: short-lived JWT access token (minutes, stateless, used on every
  request) + longer-lived opaque refresh token (stateful, checked against a DB, used only to mint
  a new access token) — gets JWT's low-latency stateless verification for the hot path while
  keeping a real revocation point for the rare "kill this session now" case.

## OAuth2 and OIDC

- **OAuth2** is an *authorization* delegation protocol — "let app X access resource Y on my
  behalf" — not itself an authentication protocol, despite being used as the plumbing under most
  "Sign in with X" flows. The **authorization code flow** (with PKCE for public clients — SPAs,
  mobile apps) is the standard flow: redirect to the provider, user authenticates there, provider
  redirects back with a code, the app exchanges the code (server-side, with a client secret or
  PKCE verifier) for tokens — the access token never transits the browser's URL bar or JS
  directly in this flow, which is the point.
- **OIDC (OpenID Connect)** is a thin identity layer built on top of OAuth2 specifically to add
  authentication: a standardized **ID token** (a JWT with standardized claims — `sub`, `email`,
  `name`) alongside OAuth2's access token, so "sign in with Google" can hand your app a verified
  identity, not just an access grant to some API.

## SSO (Single Sign-On)

One login, trusted across multiple applications — typically OIDC/SAML under the hood, with a
central Identity Provider (IdP) issuing tokens that each application (Service Provider, in SAML
terms) validates. The org-scale version of the token pattern above: apps trust a signed assertion
from the IdP rather than each maintaining its own user database and login flow.

## MFA (Multi-Factor Authentication)

Something you know (password) + something you have (a TOTP app, an SMS/push, a hardware key
like a YubiKey/WebAuthn) + sometimes something you are (biometric). TOTP and WebAuthn/passkeys
are meaningfully stronger than SMS (SIM-swap and SS7-interception are real, demonstrated attack
paths against SMS specifically) — worth knowing which factor a design actually recommends and
why, not just "MFA enabled: yes/no."

## Staff-engineer notes

- Never store passwords in any recoverable form — bcrypt/scrypt/Argon2 (slow-by-design, salted)
  hashing, never a fast general-purpose hash (MD5/SHA-256 alone) and never reversible encryption.
  This is table stakes, not a design choice.
- Token storage location on the client is itself a security decision, not an implementation
  detail — see the CSRF/XSS trade-off already covered in
  `frontend-architecture/cross-site-security.md` (httpOnly cookie vs. JS-accessible storage).
- Revocation is the question every "let's just use JWTs everywhere" proposal needs answered
  explicitly: what's the actual maximum window between "we decided to kill this session/user" and
  it taking effect, and is that acceptable for this specific system (a stolen-laptop logout vs. a
  compromised-credential incident have very different acceptable windows).
