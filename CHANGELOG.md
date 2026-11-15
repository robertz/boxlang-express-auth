# Changelog

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
