# Changelog

**0.3.0**
- **A test provider for apps' own specs** (`boxExpressAuthTestProvider()`,
  `models/providers/TestProvider.bx`). `login( app, { identity } )` runs the
  real routes — state, PKCE, `verify()`, session regeneration — through
  `app.inject()` and hands back the cookies; `deny` and `fail` simulate a
  refusal or a broken provider. Its authorize URL is under `.invalid` and
  codes only exist once test code calls `approve()`, so registering it by
  mistake doesn't open a login.
- **Logout at the issuer** for OIDC providers (RP-initiated logout). With
  `postLogoutRedirectUrl`, `POST /auth/logout` destroys the session and then
  sends the browser to the discovered `end_session_endpoint` (a `303`, or
  `200 { logoutUrl }` for a `fetch()`). The ID token is kept for
  `id_token_hint` only with `logoutIdTokenHint: true`, so by default the
  session still holds no provider token at all. `req.session.auth.provider`
  now records which provider a login came through.
- **Microsoft (Entra ID) preset** (`boxExpressAuthMicrosoft()`), with
  `tenant` (`common`, `organizations`, `consumers`, a tenant id or domain)
  and `allowedTenants`. The multi-tenant endpoints' templated issuer is
  handled by checking each token's `iss` against its own `tid`. The email
  is never reported as verified: Microsoft doesn't verify it.
- **GitLab preset** (`boxExpressAuthGitLab()`), for gitlab.com or a
  self-managed `baseUrl`.
- **Account linking**: with a `link( identity, user, tokens, req )` option,
  `POST /auth/:provider/link` lets a signed-in user attach another provider
  account. POST only (so it sits behind CSRF protection), same callback and
  checks as a login, refused with `link_denied` if the signed-in user changed
  in between. `linkRedirect` and `linkFailureRedirect` options.
- **ES256/ES384/ES512 ID tokens** (`models/Jwt.bx`), verified against EC JWKs
  on the curve the algorithm names, for issuers that sign with ECDSA.
- **Sign in with Apple** (`boxExpressAuthApple()`). Apple's ID tokens are
  ordinary RS256; what's different is everything around them:
  - The client secret is an ES256 JWT signed with the team's `.p8` key
    (`teamId`, `keyId`, `privateKey`), minted fresh with a five-minute
    lifetime for every code exchange.
  - Asking for name or email makes Apple return with a cross-site form
    POST, which a `SameSite=Lax` session cookie doesn't ride along on. The
    new `POST /auth/:provider/callback` answers it with a `303` to the
    usual GET callback — a top-level navigation, which does carry the
    cookie — so every state, PKCE and ID-token check runs unchanged. Only
    `code`, `state` and `error` go into that URL.
  - The user's name, which Apple posts once on first authorization and
    never in the ID token, is held in memory under a hash of the state
    until the GET callback collects it. The posted email and `id_token` are
    ignored in favour of the signed token from the token endpoint.
  - `auth.formPostRoutes()` mounts just that POST route ahead of
    `boxExpressCsrf()`, which would refuse Apple's tokenless POST.
- Providers can now override `clientSecretValue()` (what's sent as
  `client_secret`) and receive `state` in `fetchProfile()`'s context.
- **Guards on WebSocket upgrades**: `auth.required()` in
  `app.ws( path, handler, { use: [ sessions, auth.required() ] } )` (BoxExpress
  0.2.27+) answers an anonymous upgrade with a plain `401`, never a login
  redirect, whatever its `Accept` header says. Covered by a spec against a
  real listening server.
- OIDC subclasses can override `discoveryIssuerMatches()` and
  `tokenIssuerAllowed()` for issuers whose `iss` isn't a fixed string.

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
