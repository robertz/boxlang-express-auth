# Changelog

**0.2.0**
- **Google sign-in** (`models/providers/Google.bx`, `boxExpressAuthGoogle()`),
  built as a preset of a new **generic OpenID Connect provider**
  (`models/providers/OIDCProvider.bx`, `boxExpressAuthOIDC()`) rather than a
  Google-only class: Google needs nearly everything OIDC does (discovery, a
  nonce, ID-token validation), and the tracker's self-hosted Keycloak and
  Authelia need exactly the same code.
  - The ID token's signature is checked against the issuer's JWKS even
    though it arrives straight from the token endpoint, where OIDC Core and
    Google's docs both allow skipping it. It's cheap, and it means a
    misrouted or intercepted token endpoint can't mint a login.
  - `iss`, `aud`/`azp`, `exp`/`iat` and the `nonce` are all checked; `none`
    and HMAC algorithms and RSA keys under 2048 bits are refused. The
    discovery document must name the configured issuer, or it could point
    validation at someone else's keys. Plain http is allowed only for
    loopback issuers, for local Keycloak/Authelia.
  - The JWKS is cached and refetched once for an unknown `kid` (how a key
    rotation shows up), throttled by `jwksMinRefreshSeconds`.
  - Userinfo is read only when the ID token has no email (Authelia's
    default), must describe the same `sub`, and can't override signed
    claims.
  - Google's `hostedDomain` is enforced against the signed `hd` claim, not
    just sent as the `hd` parameter: Google documents the parameter as a
    UI hint a user can strip.
  - Only RS256/RS384/RS512 ID tokens for now (`models/Jwt.bx`), which
    covers Google, Keycloak and Authelia defaults.
- **Pending logins and identities are keyed by the name a provider is
  registered under** (`models/Auth.bx`), not the provider's own `name`. Two
  generic OIDC providers share a class; keyed by `name`, they'd have
  overwritten each other's pending state.
- **`GET /auth/:provider` builds the authorize URL before storing anything,
  and a failure there is a `provider_error` redirect** instead of a 500 —
  an OIDC provider may fetch its discovery document at that point, and an
  IdP outage shouldn't look like a crash.
- **`authorizeParams`** on every provider passes extra authorize-URL
  parameters (`prompt`, `login_hint`, …). The parameters the flow's security
  rests on (`state`, `nonce`, PKCE, `redirect_uri`, `scope`) are refused, so
  configuration can't quietly weaken it.
- **`exchangeCode()` also returns `idToken`** when the provider sends one,
  so `verify()` can keep it (for RP-initiated logout) — it's still never
  stored by this module.

**0.1.0**
- First release: third-party login for BoxExpress apps, built for a
  developer-facing issue tracker that needs "Sign in with GitHub" now and
  self-hosted OIDC (Keycloak/Authelia) and Google next.
- **Core** (`models/Auth.bx`, `boxExpressAuth()`): `routes()` for
  `GET /:provider`, `GET /:provider/callback` and `POST /logout`;
  `required()`/`optional()` guards. The app supplies `verify()` to map an
  identity to its own user, because only the app knows how its users are
  stored — the module never creates or stores users itself.
- **Security choices**, each because the naive version is exploitable:
  - `state` is 32 CSPRNG bytes, single-use (deleted before it's checked,
    so a failed attempt spends it too), expires after 10 minutes, and is
    compared in constant time.
  - PKCE `S256` on every login, so an intercepted code is useless without
    the verifier held in the session.
  - The session is regenerated before the user id is written, so a session
    id planted before login never becomes an authenticated one.
  - `returnTo` accepts only relative paths and rejects `//`, `/\`, any
    backslash and any control character. Browsers strip tab/CR/LF from URLs,
    so `/<TAB>/evil.com` would otherwise pass a prefix check and resolve as
    `//evil.com`.
  - Provider tokens go to `verify()` and nowhere else: not the session,
    not the logs. Failure redirects carry only a fixed set of codes, never
    provider-supplied text.
  - Logout is POST-only, so another site can't sign users out with a link,
    and it works under `boxExpressCsrf()`.
  - A `verify()`/`loadUser()` that throws reaches the app's error handler as
    a 500 instead of being reported as a failed login, so a database outage
    isn't mistaken for bad credentials.
- **Providers** (`models/providers/OAuth2Provider.bx`, `GitHub.bx`,
  `boxExpressAuthGitHub()`): the base class owns the authorize URL, code
  exchange, and option checks, so adding a provider is mostly
  `fetchProfile()`. GitHub identities use the numeric id (logins can be
  renamed and re-registered) and take the email from `/user/emails`, since
  `/user` often has none and never says whether it's verified. The provider
  interface already takes a `nonce` so OIDC can be added without changing it.
- **Outbound HTTP is injectable** (`models/HttpClient.bx`, `httpClient`
  option): specs run the whole flow against a fake that fails on any
  unexpected URL, so none of them touch GitHub. The default client treats
  `bx:http`'s synthetic "502 Connection Failure" as an error rather than a
  response, since it means nothing answered.
