# boxlang-express-auth

Pluggable third-party login for [BoxExpress](https://github.com/robertz/boxlang-express)
apps: a generic OAuth2 authorization-code core (always PKCE `S256`), a
generic OpenID Connect provider, presets for GitHub, Google, Microsoft,
GitLab and Apple, and a test provider for your own specs. Your app decides who a
provider identity *is*; this module runs the flow, guards routes (WebSocket
upgrades included), links extra accounts, and keeps the session honest.

- Requires BoxLang ≥ 1.15 and the `boxexpress` module (0.2.26+; 0.2.27+ to
  guard WebSockets).
- Not included: refresh tokens, token storage, an account-linking UI (the
  route is here, the settings page is yours), password login, API tokens,
  ID-token algorithms other than RS256/384/512 and ES256/384/512.

## Install

```bash
box install boxlang-express-auth
```

Install `boxlang-express` too, if the app doesn't already have it — this
module builds its routes with `boxExpressRouter()` but doesn't install
BoxExpress for you. Both land in the app's `boxlang_modules/`.

## Quickstart

```js
app = boxExpress()
app.set( "trust proxy", 1 )

auth = boxExpressAuth( {
	// Required. Map the provider identity to your own user.
	// Return a struct with at least { id }, or null to refuse the login.
	verify: ( identity, tokens, req ) => users.findOrCreateFromIdentity( identity ),
	// Optional. Turns the stored id back into req.user on guarded routes.
	// Without it, req.user is { id }.
	loadUser: ( id ) => users.get( id ),
	successRedirect: "/",          // after login (unless returnTo) and after browser logout
	failureRedirect: "/login",     // gets ?error=<code>
	loginPath: "/login"            // where required() sends anonymous browsers
} )

auth.provider( "github", boxExpressAuthGitHub( {
	clientId: getSystemSetting( "GITHUB_CLIENT_ID" ),
	clientSecret: getSystemSetting( "GITHUB_CLIENT_SECRET" ),
	callbackUrl: "https://tracker.example.com/auth/github/callback",
	scopes: [ "read:user", "user:email" ]     // the default
} ) )

app.use( boxExpressSession( { resave: false, saveUninitialized: false } ) )   // required, and first
app.use( boxExpressUrlencoded() )
app.use( boxExpressCsrf() )                       // optional; guards POST /auth/logout too
app.use( "/auth", auth.routes() )

app.get( "/dashboard", auth.required(), ( req, res ) => res.json( { user: req.user } ) )
app.get( "/", auth.optional(), ( req, res ) => res.render( "home", { user: req.user ?: "" } ) )
```

Link to `/auth/github` to sign in, or `/auth/github?returnTo=/projects/7` to
come back somewhere specific. `examples/server.bxs` is a complete runnable
demo (`GITHUB_CLIENT_ID=… GITHUB_CLIENT_SECRET=… boxlang examples/server.bxs`).

### Routes

Mounted wherever you `app.use()` the router — the callback URLs you register
with providers have to point at the same place.

| Route | |
|---|---|
| `GET /auth/:provider` | Starts a login. `?returnTo=` takes a relative path only. |
| `GET /auth/:provider/callback` | Where the provider sends the user back. |
| `POST /auth/:provider/callback` | Only for providers that come back with a form post (Apple). Answers `303` to the GET callback; see [Apple](#apple). Otherwise `404`. |
| `POST /auth/:provider/link` | A signed-in user attaching another provider account. Only with a `link` option; see [Linking accounts](#linking-accounts). |
| `POST /auth/logout` | Destroys the session. Redirects to `successRedirect` when `Accept` includes `text/html`, otherwise `204`. If the user signed in through an OIDC provider with `postLogoutRedirectUrl`, it sends them on to the issuer's logout instead ([below](#logging-out-at-the-issuer)). There is no `GET` logout — a link or an `<img>` on another site must not be able to sign people out. Use a form. |

An unknown provider name is a `404`.

### Guards

- `auth.required()` — passes logged-in requests with `req.user` set. An
  anonymous request whose `Accept` includes `text/html` is redirected to
  `loginPath?returnTo=<the original URL>`; anything else gets
  `401 { error: true, code: "unauthenticated" }`. Point `loginPath` at
  `/auth/github` to skip a login page when GitHub is your only provider.
- `auth.optional()` — sets `req.user` if logged in, never blocks. Fine to
  mount globally; `loadUser` still runs once per request.

If `loadUser` returns null (the account was deleted), the request is treated
as anonymous and the stale id is cleared from the session.

Both work as WebSocket upgrade middleware (BoxExpress 0.2.27+). Pass the
same session middleware instance you gave `app.use()`, then the guard:

```js
sessions = boxExpressSession()
app.use( sessions )

app.ws( "/live", ( connection, req ) => {
	connection.onMessage( ( text ) => feed.publish( req.user.id, text ) )
}, { use: [ sessions, auth.required() ] } )
```

A signed-in user's socket opens with `req.user` set. Anyone else gets a
plain `401` — never the login redirect, whatever the upgrade's `Accept`
says, since a socket can't follow one.

### What happens on the callback

1. The pending login is removed from the session — whatever happens next,
   that `state` can't be used again.
2. `state` is compared, in constant time, with the one minted at the start;
   it must also be under 10 minutes old (`stateTtlSeconds`).
3. The code is exchanged (with the PKCE verifier), then the profile fetched.
4. `verify( identity, tokens, req )` runs. Null, or no `id`, refuses the login.
5. The session is **regenerated** (new id, old one deleted — no session
   fixation), then `req.session.auth.userId` is set.
6. Redirect to `returnTo`, or `successRedirect`.

Provider tokens are handed to `verify()` and then dropped: they're never
written to the session or logged. If your app needs one later (say, to call
the GitHub API for repositories), store it yourself — encrypted — in
`verify()`. `tokens` is `{ accessToken, tokenType, scope }`.

If `verify()` or `loadUser()` **throws**, that goes to your error handler as
a 500 — it means your code or your database failed, not the login.

### Failure codes

`failureRedirect` gets `?error=` with one of these, and nothing else — a
provider's own error text is never echoed:

| Code | Meaning |
|---|---|
| `access_denied` | The user declined on the provider's consent screen. |
| `provider_error` | The provider sent back any other error or no code — or, on `GET /auth/:provider`, an OIDC issuer's discovery document couldn't be read. |
| `invalid_state` | State missing, wrong, already used, or no login was pending in this session. |
| `expired_state` | Took longer than 10 minutes. |
| `token_exchange_failed` | The provider refused the code (GitHub: HTTP 200 with an `error` body). |
| `profile_failed` | The profile couldn't be read, or an OIDC ID token failed validation (the log says which check). |
| `login_denied` | `verify()` returned null. |
| `link_denied` | A link was refused: `link()` returned false or null, or the user who started it is no longer the one signed in. Sent to `linkFailureRedirect`. |

The reason is logged server-side (`[BoxExpressAuth] github login failed: …`),
never with codes, tokens or the client secret.

### Errors you'll see while wiring it up

| Type | When |
|---|---|
| `BoxExpressAuth.SessionRequired` | A request reached the auth routes or a guard without `boxExpressSession()` in front. |
| `BoxExpressAuth.InvalidConfig` | No `verify` function, or `loadUser` or `link` isn't one. |
| `BoxExpressAuth.InvalidProvider` | Missing `clientId`/`clientSecret`/`callbackUrl`, a relative `callbackUrl`, a bad provider name, an `authorizeParams` key the flow owns, or (OIDC) a missing or plain-http issuer or no `openid` scope. |
| `BoxExpressAuth.DuplicateProvider` | Two providers registered under one name. |

### Options

| Option | Default | |
|---|---|---|
| `verify` | — | Required. `( identity, tokens, req ) => user struct or null` |
| `loadUser` | none | `( id ) => user struct or null` |
| `successRedirect` | `"/"` | |
| `failureRedirect` | `"/login"` | `?error=` (or `&error=`) is appended |
| `loginPath` | `"/login"` | |
| `sessionKey` | `"auth"` | Key under `req.session` this module owns |
| `stateTtlSeconds` | `600` | |
| `link` | none | `( identity, user, tokens, req ) => true/false`. Turns on `POST /auth/:provider/link` |
| `linkRedirect` | `successRedirect` | After a link (unless `returnTo`) |
| `linkFailureRedirect` | `failureRedirect` | After a failed link, with `?error=` |

Every provider takes `clientId`, `clientSecret`, `callbackUrl` (all
required), `scopes`, `authorizeParams` (extra authorize-URL parameters such
as `{ prompt: "select_account" }`; the flow's own — `state`, `nonce`,
`scope`, `redirect_uri`, PKCE — can't be overridden), `userAgent` (default
`boxlang-express-auth`) and `httpClient`. The OIDC providers (generic,
Google, Microsoft, GitLab) add the options in their sections below.

## Registering a GitHub OAuth app

1. GitHub → **Settings → Developer settings → OAuth Apps → New OAuth App**
   (or the same under an organization's settings).
2. **Homepage URL**: your app, e.g. `https://tracker.example.com`.
3. **Authorization callback URL**: exactly the provider's `callbackUrl` —
   the mount path plus `/github/callback`:
   `https://tracker.example.com/auth/github/callback`.
   For local development, register a second OAuth app with
   `http://localhost:3000/auth/github/callback` rather than widening the
   production one.
4. Leave **Enable Device Flow** off.
5. Copy the Client ID, generate a client secret, and put both in the
   environment (`GITHUB_CLIENT_ID`, `GITHUB_CLIENT_SECRET`). Never commit them.

GitHub supports PKCE for OAuth apps (`S256` only); this module always sends
it. The default scopes are `read:user` and `user:email` — the latter is what
lets the provider see private addresses and whether they're verified.

## Google

```js
auth.provider( "google", boxExpressAuthGoogle( {
	clientId: getSystemSetting( "GOOGLE_CLIENT_ID" ),
	clientSecret: getSystemSetting( "GOOGLE_CLIENT_SECRET" ),
	callbackUrl: "https://tracker.example.com/auth/google/callback",
	hostedDomain: "example.com"        // optional — see below
} ) )
```

Link to `/auth/google`. Scopes default to `openid email profile`. The
identity's `id` is Google's `sub` (a long numeric string); `username` is
always `""` because Google has none; `avatarUrl` is the `picture` claim.

**`hostedDomain`** limits sign-in to one Google Workspace domain. It's sent
as Google's `hd` parameter — which only pre-selects the account and can be
removed by the user — and then **enforced** against the `hd` claim inside
the signed ID token. `"*"` accepts any Workspace account but no consumer
`@gmail.com` one. Don't restrict by the email's domain instead: Google says
to use `hd` for that.

### Registering a Google OAuth client

1. [Google Cloud console](https://console.cloud.google.com/) → pick or create
   a project → **APIs & Services → OAuth consent screen**. Choose *Internal*
   (Workspace users only) or *External*, add the `openid`, `email` and
   `profile` scopes, and add test users while the app is in testing.
2. **APIs & Services → Credentials → Create credentials → OAuth client ID**,
   application type **Web application**.
3. **Authorized redirect URIs**: exactly the provider's `callbackUrl` —
   `https://tracker.example.com/auth/google/callback`. Add
   `http://localhost:3000/auth/google/callback` to a separate dev client
   rather than the production one.
4. Put the client ID and secret in the environment (`GOOGLE_CLIENT_ID`,
   `GOOGLE_CLIENT_SECRET`).

## Any OpenID Connect provider (Keycloak, Authelia, …)

```js
auth.provider( "keycloak", boxExpressAuthOIDC( {
	name: "keycloak",
	issuer: "https://sso.example.com/realms/tracker",
	clientId: getSystemSetting( "OIDC_CLIENT_ID" ),
	clientSecret: getSystemSetting( "OIDC_CLIENT_SECRET" ),
	callbackUrl: "https://tracker.example.com/auth/keycloak/callback"
} ) )
```

The provider reads `<issuer>/.well-known/openid-configuration` on the first
login (not at startup, so the app boots while the IdP is down), sends a
`nonce` with every login, and validates the ID token before `verify()` sees
anything:

- the signature, against the issuer's JWKS — RS256/384/512 with an RSA key
  of at least 2048 bits, or ES256/384/512 with a P-256/P-384/P-521 key;
  `none` and HMAC are refused. The key set is
  cached and refetched once when a token names an unknown `kid` (a key
  rotation), at most every `jwksMinRefreshSeconds` (default 60).
- `iss` is the configured issuer, and the discovery document names that
  same issuer.
- `aud` contains `clientId`; `azp`, when present (or when there are several
  audiences), is `clientId`.
- `exp` hasn't passed and `iat` isn't in the future, give or take
  `clockSkewSeconds` (default 60).
- `nonce` is the one minted for this login.

The identity's `id` is `sub`; `username` is `preferred_username`;
`emailVerified` is `email_verified` (the string `"true"` counts). When the
ID token has no email, the userinfo endpoint is read as well — its `sub`
must match, and it never overrides a signed ID-token claim.

| Option | Default | |
|---|---|---|
| `issuer` | — | Required. https, or http on `localhost`/`127.0.0.1`/`[::1]` only |
| `name` | `"oidc"` | Register under the same name you use here |
| `scopes` | `[ "openid", "email", "profile" ]` | Must include `openid` |
| `userinfo` | `"auto"` | `"auto"` (when the ID token lacks an email), `"always"`, `"never"` |
| `clockSkewSeconds` | `60` | |
| `jwksMinRefreshSeconds` | `60` | |
| `acceptedIssuers` | `[ issuer ]` | Extra `iss` spellings (Google uses this) |
| `algorithms` | RS256/384/512, ES256/384/512 | Narrow it, never widen it |
| `postLogoutRedirectUrl` | none | Turns on [logout at the issuer](#logging-out-at-the-issuer) |
| `logoutIdTokenHint` | `false` | Keep the ID token in the session to send as `id_token_hint` |
| `authorizeUrl`, `tokenUrl`, `jwksUri`, `userinfoUrl`, `endSessionUrl` | discovered | Give the first three to skip discovery |

`tokens` passed to `verify()` includes `idToken` for OIDC providers. It
isn't stored, unless `logoutIdTokenHint` asks for it.

Keycloak: create a confidential client (*Client authentication* on, *Standard
flow* on), set the redirect URI, and use
`https://<host>/realms/<realm>` as the issuer. Authelia: register the client
under `identity_providers.oidc.clients` with `authorization_policy`,
`redirect_uris`, `scopes: [ openid, email, profile ]` and
`require_pkce: true`, `pkce_challenge_method: S256`; the issuer is
Authelia's own URL. Authelia often leaves email out of the ID token, which
is what `userinfo: "auto"` covers.

### Logging out at the issuer

`POST /auth/logout` ends the session in your app; the user is still signed
in at Keycloak, Authelia or Microsoft, and the next login goes straight
through. For single sign-on that's rarely what "Sign out" should mean.
Give the provider a `postLogoutRedirectUrl` (registered with the issuer as
a post-logout redirect URI) and logout continues at the issuer's
`end_session_endpoint`:

```js
auth.provider( "keycloak", boxExpressAuthOIDC( {
	…,
	postLogoutRedirectUrl: "https://tracker.example.com/"
} ) )
```

- A form post (`Accept: text/html`) gets a `303` to
  `<end_session_endpoint>?client_id=…&post_logout_redirect_uri=…`.
- A `fetch()` gets `200 { logoutUrl }` — set `location.href` to it.
- The local session is destroyed first either way. If the issuer has no
  `end_session_endpoint` (GitLab, Google), or discovery fails, it's a
  normal local logout.

Without an `id_token_hint`, most issuers show a "do you want to sign out?"
page. `logoutIdTokenHint: true` keeps the login's ID token in the session
(`req.session.auth.logout.idToken`) so it can be sent, and the issuer skips
the question. It's off by default because it's the one token this module
would otherwise never store: an ID token isn't an API credential, but it
is a signed statement of who the user is, sitting in your session store.

## Microsoft

```js
auth.provider( "microsoft", boxExpressAuthMicrosoft( {
	clientId: getSystemSetting( "MICROSOFT_CLIENT_ID" ),
	clientSecret: getSystemSetting( "MICROSOFT_CLIENT_SECRET" ),
	callbackUrl: "https://tracker.example.com/auth/microsoft/callback",
	tenant: "organizations"
} ) )
```

Entra ID (the v2.0 endpoints), as an OIDC preset. `tenant` decides who can
sign in:

| `tenant` | |
|---|---|
| `"common"` (default) | Any work/school account, and personal Microsoft accounts |
| `"organizations"` | Work/school accounts only |
| `"consumers"` | Personal Microsoft accounts only |
| a tenant id or domain | That one tenant (`"72f988bf-…"` or `"contoso.onmicrosoft.com"`) |

The multi-tenant endpoints publish a templated issuer
(`https://login.microsoftonline.com/{tenantid}/v2.0`) and every token
carries its own tenant's `iss`. The preset checks `iss` against the token's
`tid` claim and `tid` against `tenant`; with `allowedTenants: [ "<tenant
id>", … ]`, only those tenants get in — how a multi-tenant SaaS app limits
sign-in to the customers it has onboarded.

**`identity.emailVerified` is always false.** Microsoft doesn't verify the
`email` claim (a tenant admin can set it to anything), so it must never
decide who someone is. `identity.id` is `sub`; `identity.tenantId` is the
`tid`; `identity.username` is `preferred_username` (usually the UPN, which
can change). Userinfo is off (`userinfo: "never"`) — the ID token already
has everything userinfo would add.

Register the app under *Microsoft Entra ID → App registrations*: add a
*Web* redirect URI of `<origin>/auth/microsoft/callback` (and, for logout
at Microsoft, your `postLogoutRedirectUrl` as a redirect URI too), and
create a client secret under *Certificates & secrets*. The *Supported account types* choice
has to agree with `tenant`.

## GitLab

```js
auth.provider( "gitlab", boxExpressAuthGitLab( {
	clientId: getSystemSetting( "GITLAB_CLIENT_ID" ),
	clientSecret: getSystemSetting( "GITLAB_CLIENT_SECRET" ),
	callbackUrl: "https://tracker.example.com/auth/gitlab/callback",
	baseUrl: "https://gitlab.com"      // or your self-managed instance
} ) )
```

GitLab is an OIDC issuer at its own base URL, so this is the generic
provider with that issuer. `identity.username` is the GitLab username;
`emailVerified` comes from GitLab's `email_verified`. Create the
application under *User settings → Applications* (or an instance-wide one
under *Admin → Applications*) with the `openid`, `email` and `profile`
scopes. GitLab has no `end_session_endpoint`, so logout is local only.

## Apple

```js
auth.provider( "apple", boxExpressAuthApple( {
	clientId: getSystemSetting( "APPLE_SERVICES_ID" ),        // the Services ID, e.g. com.example.tracker.web
	teamId: getSystemSetting( "APPLE_TEAM_ID" ),
	keyId: getSystemSetting( "APPLE_KEY_ID" ),
	privateKey: fileRead( getSystemSetting( "APPLE_PRIVATE_KEY_PATH" ) ),   // the .p8 file
	callbackUrl: "https://tracker.example.com/auth/apple/callback"
} ) )

app.use( boxExpressUrlencoded() )
app.use( boxExpressSession( { resave: false, saveUninitialized: false } ) )
app.use( "/auth", auth.formPostRoutes() )      // before CSRF — see below
app.use( boxExpressCsrf() )
app.use( "/auth", auth.routes() )
```

There's no `clientSecret`: Apple's is a JWT signed (ES256) with your key,
and the provider mints a fresh one, valid for five minutes, for each code
exchange. The key never leaves the process, and errors never quote it.

**The callback is a POST.** When you ask for `name` or `email` (the default
`scopes`), Apple sends the user back with a form post from
appleid.apple.com. Two things follow:

- A `SameSite=Lax` session cookie isn't sent on a cross-site POST, so the
  login couldn't be matched to its session there. `POST
  /auth/apple/callback` doesn't try: it answers `303` to `GET
  /auth/apple/callback?code=…&state=…`, a top-level navigation the cookie
  *is* sent with, and the usual checks run there. Keep `sameSite: "Lax"`;
  don't loosen it to `None` for Apple's sake.
- `boxExpressCsrf()` would refuse that POST (no token, no session). Mount
  `auth.formPostRoutes()` — only that one route — at the same path
  *before* the CSRF middleware, as above, and `boxExpressUrlencoded()`
  before both. Without the body parser the POST arrives empty and the
  login fails with `provider_error` (the log says why).

With `scopes: []` Apple uses a plain GET callback and none of that applies
— but then there's no email in the ID token.

**The name comes once.** Apple never puts the name in the ID token; it
posts it as a `user` field on the user's *first* authorization of your app
only. The provider holds it in memory, keyed by a hash of that login's
state, until the GET callback hands it to `verify()` as `identity.name`.
Save it then — later logins have `name: ""`. With several app instances and
no sticky sessions the GET can land elsewhere and the name is lost (the
login still works); the user can get Apple to send it again by removing
your app under *Sign in with Apple* in their Apple account settings. The
email in that `user` field is ignored; `identity.email` comes from the
signed ID token.

`identity.id` is Apple's `sub`, stable for your developer team.
`identity.email` may be a private relay address (`…@privaterelay.appleid.com`)
— `identity.isPrivateEmail` says so — and `emailVerified` is Apple's
`email_verified`. Apple has no userinfo endpoint and no
`end_session_endpoint`, so logout is local only.

To register: in the Apple developer account create an App ID with *Sign in
with Apple*, then a *Services ID* (its identifier is `clientId`) with your
domain and the return URL `<origin>/auth/apple/callback` — Apple requires
https and a real domain, so local development needs a tunnel. Under
*Keys*, create a key with *Sign in with Apple*; download the `.p8` once
(`privateKey`) and note its Key ID (`keyId`) and your Team ID (`teamId`).

### Behind a proxy, and cookies

The callback is a top-level `GET` from github.com back to your site, so the
session cookie must be sent on a cross-site navigation: the default
`sameSite: "Lax"` works; **`sameSite: "Strict"` makes every login fail with
`invalid_state`**. Set `secure: true` on the session cookie once TLS is in
front, and `app.set( "trust proxy", … )` as BoxExpress's docs describe.

With a database-backed session store, keep `boxExpressSession()`'s
`resave: false, saveUninitialized: false`.

Consider rate-limiting the auth routes:
`app.use( "/auth", boxExpressRateLimit( { max: 30, windowMs: 60000 } ) )`
before mounting them.

## Storing users: key on the provider id, never on email

The identity passed to `verify()`:

```js
{ provider: "github", id: "583231", email: "octocat@github.com", emailVerified: true,
  name: "The Octocat", username: "octocat", avatarUrl: "https://…", raw: { …the provider's profile… } }
```

`id` is the provider's stable user id, as a string — for GitHub, the numeric
id, not the login (logins can be renamed, and then claimed by someone else);
for Google, Microsoft, GitLab, Apple and OIDC, the `sub` claim.
Store identities in their own table, linked to your users:

```sql
CREATE TABLE users (
	id           INT IDENTITY PRIMARY KEY,
	display_name NVARCHAR(200) NOT NULL,
	email        NVARCHAR(320) NULL,
	created_at   DATETIME2 NOT NULL DEFAULT SYSUTCDATETIME()
);

CREATE TABLE user_identities (
	id               INT IDENTITY PRIMARY KEY,
	user_id          INT NOT NULL REFERENCES users ( id ),
	provider         VARCHAR(32) NOT NULL,
	provider_user_id VARCHAR(255) NOT NULL,
	email            NVARCHAR(320) NULL,
	email_verified   BIT NOT NULL DEFAULT 0,
	created_at       DATETIME2 NOT NULL DEFAULT SYSUTCDATETIME(),
	last_login_at    DATETIME2 NULL,
	CONSTRAINT uq_user_identities UNIQUE ( provider, provider_user_id )
);
```

In `verify()`: look up `( identity.provider, identity.id )`. Found → that's
the user. Not found → create a user and an identity row. Never look a user
up by email to decide who someone is: an address can be unverified,
reassigned, or typed into another provider by someone else. If you want to
attach a new provider to an existing account automatically by email, only do
it when `identity.emailVerified` is true — and prefer an explicit "link
account" step for a signed-in user — next.

## Linking accounts

Matching a new provider login to an existing user by email is the shortcut
the section above warns against. The safe version is explicit: a user who
is already signed in proves they also own the other account.

```js
auth = boxExpressAuth( {
	verify: …,
	loadUser: …,
	link: ( identity, user, tokens, req ) => {
		var owner = identities.find( identity.provider, identity.id )
		if ( !isNull( owner ) && owner.userId != user.id ) {
			return false                 // already someone else's
		}
		identities.attach( user.id, identity )
		return true
	},
	linkRedirect: "/settings/accounts",
	linkFailureRedirect: "/settings/accounts"
} )
```

```html
<form method="post" action="/auth/google/link">
	<input type="hidden" name="_csrf" value="…req.csrfToken()…">
	<input type="hidden" name="returnTo" value="/settings/accounts">
	<button>Connect Google</button>
</form>
```

`POST /auth/:provider/link` runs the same flow as a login — same callback
URL, same state, PKCE and ID-token checks — but the callback calls
`link( identity, user, tokens, req )` instead of `verify()`, with `user` the
signed-in user. It doesn't regenerate the session or change who is signed
in.

- It's POST so another site can't start it with a link or an image; keep
  `boxExpressCsrf()` in front.
- Anonymous: `401`, or a `303` to `loginPath` for a browser.
- If the user signed out, or someone else signed in, between the start and
  the callback, it's refused with `link_denied`.
- Return `false` (or null) from `link()` to refuse — say, an identity that
  already belongs to another user. Throws go to your error handler.
- Without a `link` option the route is a `404`.

## Testing your app

`boxExpressAuthTestProvider()` logs in through the real `/auth` routes —
state, PKCE, your `verify()`, session regeneration — as whatever identity
a spec names, with no provider account and no network:

```js
testLogin = boxExpressAuthTestProvider( { identity: { id: "u1", email: "ada@example.com" } } )
auth.provider( "test", testLogin )      // only when building the app for tests

var login = testLogin.login( app )                              // → { response, location, cookies, started }
expect( app.inject( { url: "/dashboard", cookies: login.cookies } ).statusCode ).toBe( 200 )

testLogin.login( app, { identity: { id: "u2", emailVerified: false }, returnTo: "/settings" } )
testLogin.login( app, { deny: "access_denied" } )               // the user clicked Cancel
testLogin.login( app, { fail: "profile_failed" } )              // or "token_exchange_failed"
testLogin.link( app, { cookies: login.cookies, form: { _csrf: token } } )
```

Its default identity is `{ id: "test-user", email: "test@example.com",
emailVerified: true, name: "Test User", username: "test" }`, merged with
the `identity` you pass. Its `callbackUrl` defaults to
`http://localhost/auth/<name>/callback`; set it if you mount the routes
elsewhere. For a step at a time, `approve( location, identity )` and
`deny( location, error )` turn the `Location` of `GET /auth/test` into the
callback path to inject.

It can't log anyone in from a browser — `GET /auth/test` redirects to
`auth.test.invalid`, which never resolves, and a code only exists once
test code calls `approve()` — but register it only when building the app
for tests all the same.

## Tests

```bash
box install                # testbox + boxlang-express into boxlang_modules/
boxlang setup-tests.bxs    # once per checkout
boxlang run-tests.bxs
```

Specs drive the app with `app.inject()`; provider APIs are a
`FakeHttpClient` that fails the test on any unscripted URL, and ID tokens
are signed by real RSA and EC keys generated per run. The only sockets any
spec opens are on loopback: `HttpClientSpec`'s in-process server, and
`WebSocketGuardSpec`'s app (port 38742) for real upgrade requests.

## Writing a provider

See [docs/providers.md](docs/providers.md).

## License

MIT
