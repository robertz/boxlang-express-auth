# boxlang-express-auth

Pluggable third-party login for [BoxExpress](https://github.com/robertz/boxlang-express)
apps: a generic OAuth2 authorization-code core (always PKCE `S256`) and a
GitHub provider. Your app decides who a provider identity *is*; this module
runs the flow, guards routes, and keeps the session honest.

- Requires BoxLang ≥ 1.15 and the `boxexpress` module (0.2.26+).
- Not in v0.1: OIDC providers (Google, Keycloak, Authelia — next), refresh
  tokens, token storage, account-linking UI, password login, API tokens.

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
| `POST /auth/logout` | Destroys the session. Redirects to `successRedirect` when `Accept` includes `text/html`, otherwise `204`. There is no `GET` logout — a link or an `<img>` on another site must not be able to sign people out. Use a form. |

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
| `provider_error` | The provider sent back any other error, or no code. |
| `invalid_state` | State missing, wrong, already used, or no login was pending in this session. |
| `expired_state` | Took longer than 10 minutes. |
| `token_exchange_failed` | The provider refused the code (GitHub: HTTP 200 with an `error` body). |
| `profile_failed` | The profile couldn't be read. |
| `login_denied` | `verify()` returned null. |

The reason is logged server-side (`[BoxExpressAuth] github login failed: …`),
never with codes, tokens or the client secret.

### Errors you'll see while wiring it up

| Type | When |
|---|---|
| `BoxExpressAuth.SessionRequired` | A request reached the auth routes or a guard without `boxExpressSession()` in front. |
| `BoxExpressAuth.InvalidConfig` | No `verify` function, or `loadUser` isn't one. |
| `BoxExpressAuth.InvalidProvider` | Missing `clientId`/`clientSecret`/`callbackUrl`, a relative `callbackUrl`, or a bad provider name. |
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

GitHub provider: `clientId`, `clientSecret`, `callbackUrl` (all required),
`scopes`, `userAgent` (default `boxlang-express-auth`), `httpClient`.

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
id, not the login (logins can be renamed, and then claimed by someone else).
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
account" step for a signed-in user.

## Tests

```bash
box install                # testbox + boxlang-express into boxlang_modules/
boxlang setup-tests.bxs    # once per checkout
boxlang run-tests.bxs
```

Specs drive the app with `app.inject()`; the GitHub API is a
`FakeHttpClient` that fails the test on any unscripted URL. The only socket
any spec opens is `HttpClientSpec`'s, to an in-process server on loopback.

## Writing a provider

See [docs/providers.md](docs/providers.md).

## License

MIT
