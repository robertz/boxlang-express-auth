# Writing a provider

A provider is any object with these members. `Auth` checks for them when
you call `auth.provider( name, instance )`.

| Member | |
|---|---|
| `name` | Short identifier; `identity.provider` is forced to the name it's registered under. |
| `callbackUrl` | Absolute URL of `GET <mount>/<name>/callback`, exactly as registered with the provider. |
| `authorizationUrl( state, codeVerifier, nonce, callbackUrl )` | The URL to send the browser to. |
| `exchangeCode( code, codeVerifier, callbackUrl )` | Returns `{ accessToken, tokenType, scope }`, plus `idToken` when the provider sent one (more keys are fine). |
| `fetchProfile( tokens, context )` | Returns the normalized identity (below). `context` is `{ nonce }`. |

`exchangeCode()` and `fetchProfile()` signal failure by throwing — `Auth`
turns any throw into `token_exchange_failed` / `profile_failed` and logs
`e.type` and `e.message`. **Messages must never include the code, a token, or
the client secret.**

The identity:

```js
{
	provider,        // set by Auth
	id,              // the provider's stable, immutable user id, as a string — never a username or email
	email,           // "" when unknown
	emailVerified,   // true only when the provider asserts it
	name, username, avatarUrl,   // "" when unknown
	raw              // the provider's own profile struct
}
```

`nonce` is minted for every login and stored with the pending state. It goes
to `authorizationUrl()` and back to `fetchProfile()` as `context.nonce`, so an
OIDC provider can check the ID token against it. Plain OAuth2 providers
ignore both.

## Extending OAuth2Provider

`models/providers/OAuth2Provider.bx` already does option validation, the
authorize URL (always with `code_challenge` / `code_challenge_method=S256`),
the token exchange (form POST, `Accept: application/json`, error-in-200
handling), and `getJSON( url, tokens, headers )` for authenticated API reads.
A subclass passes its endpoints as `init()` defaults and implements
`fetchProfile()`:

```js
class extends="bxModules.boxexpressauth.models.providers.OAuth2Provider" {

	function init( struct options = {} ) {
		return super.init( options, {
			name: "gitea",
			authorizeUrl: options.baseUrl & "/login/oauth/authorize",
			tokenUrl: options.baseUrl & "/login/oauth/access_token",
			scopes: [ "read:user" ]
		} )
	}

	function fetchProfile( required struct tokens ) {
		var user = getJSON( "https://gitea.example.com/api/v1/user", tokens )
		return {
			provider: this.name,
			id: toString( user.id ),
			email: user.email ?: "",
			emailVerified: false,
			name: user.full_name ?: "",
			username: user.login ?: "",
			avatarUrl: user.avatar_url ?: "",
			raw: user
		}
	}

}
```

Add a BIF in `bifs/` that returns `new bxModules.boxexpressauth.models.providers.Gitea( options )`,
and specs that drive it through `FakeHttpClient`.

## Outbound HTTP

Every request goes through `variables.httpClient.request( { method, url, headers, form } )`,
which returns `{ status, headers, body }` and only throws when there was no
response at all. The default is `models/HttpClient.bx` (`bx:http`, no
redirects followed, 15s timeout). Pass `httpClient` in the provider options
to replace it — tests use `tests/helpers/FakeHttpClient.bx`; an app might
add a proxy or tracing.

## OIDC providers

`models/providers/OIDCProvider.bx` is the generic OpenID Connect provider
(see the README for its options). It extends `OAuth2Provider` with
`discoverEndpoints: true` and overrides:

- `getAuthorizeUrl()` / `getTokenUrl()` — read from the cached discovery
  document instead of fixed options.
- `authorizationParams( nonce )` — adds `nonce`.
- `fetchProfile( tokens, context )` — validates `tokens.idToken` (signature
  against the JWKS, `iss`, `aud`/`azp`, `exp`/`iat`, `nonce` =
  `context.nonce`), optionally merges userinfo, then calls
  `identityFromClaims( claims )`.

A preset for a specific issuer is a subclass that passes defaults and, if
needed, overrides `identityFromClaims()` to enforce something extra — see
`Google.bx`, which pins the issuer, accepts Google's two `iss` spellings, and
enforces `hostedDomain` against the `hd` claim:

```js
class extends="bxModules.boxexpressauth.models.providers.OIDCProvider" {

	function init( struct options = {} ) {
		return super.init( options, {
			name: "acme",
			issuer: "https://login.acme.test",
			scopes: [ "openid", "email" ]
		} )
	}

	function identityFromClaims( required struct claims ) {
		if ( !( claims.acme_employee ?: false ) ) {
			throw( type = "BoxExpressAuth.ProfileFailed", message = "acme account isn't an employee" )
		}
		return super.identityFromClaims( claims )
	}

}
```

ID tokens are verified by `models/Jwt.bx`: RS256/RS384/RS512 only, keys of
2048 bits or more. Adding ES256 (some IdPs default to it) means teaching
`Jwt.bx` to build an EC key from a JWK's `crv`/`x`/`y` and to convert the
JWS signature from its raw `r||s` form to the DER form Java's `Signature`
expects.
