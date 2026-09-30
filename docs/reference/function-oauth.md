# OAuth login for functions

The [of-watchdog](/architecture/watchdog/) can protect a function with browser login through an OAuth 2.0 or OpenID Connect (OIDC) provider. It handles sign-in and checks the browser's session before forwarding requests to the function.

## Example: Deploy a function with OIDC login

This walkthrough deploys a function that requires visitors to sign in through an OIDC provider. It shows how to configure the watchdog to enable OAuth without changing the function handler.

1. Create a function with the `golang-middleware` template:

    ```bash
    faas-cli template store pull golang-middleware
    faas-cli new profile --lang golang-middleware
    ```

    The watchdog handles login, so the function does not need to implement any login or OAuth logic in its handler.

2. Register a client with your OIDC provider.

    Set its redirect (callback) URI to `https://gateway.example.com/function/profile/auth/callback`.
    Note its client ID and client secret. The client secret is optional; public clients using PKCE are also supported.

3. Create a signing key as an [OpenFaaS secret](/reference/secrets/):

    ```bash
    faas-cli secret generate | faas-cli secret create profile-signing-key \
      --from-file=/dev/stdin --gateway=https://gateway.example.com
    ```

    The watchdog uses this key to sign its own JWT session cookie, which it sets as an HttpOnly cookie after successful authentication.

    Save the client secret from your provider in a local file named `client-secret`, then create an OpenFaaS secret from it:

    ```bash
    faas-cli secret create profile-client-secret --from-file=client-secret \
      --gateway=https://gateway.example.com
    ```

4. Configure `stack.yaml` to enable login and mount both secrets:

    ```yaml
    version: 1.0
    provider:
      name: openfaas
      gateway: https://gateway.example.com
    functions:
      profile:
        lang: golang-middleware
        handler: ./profile
        image: ttl.sh/openfaas-examples/profile:latest
        environment:
          oauth_enabled: 'true'
          oauth_base_url: https://gateway.example.com/function/profile
          oauth_client_id: profile
          oauth_issuer_url: https://keycloak.example.com/realms/openfaas
          oauth_signing_key: profile-signing-key
          oauth_client_secret: profile-client-secret
        secrets:
          - profile-signing-key
          - profile-client-secret
    ```

    Replace the gateway URL, image, client ID, and issuer URL with your own values.

    - `oauth_base_url` is the public URL where visitors reach the function, normally `<gateway-url>/function/<function-name>` (here, `https://gateway.example.com/function/profile`). The watchdog needs it to redirect visitors back after login and to build the correct OAuth callback URL.
    - `oauth_client_id` is the client ID assigned by the provider when the application is registered.
    - `oauth_issuer_url` is the provider's issuer URL, which the watchdog uses to discover its authorization, token, and public-key endpoints.
    - `oauth_signing_key` is the name of the secret with the signing key.
    - `oauth_client_secret` is the name of the secret holding the provider-issued client secret.

    For a public client using PKCE without a client secret, omit `oauth_client_secret` and its entry in `secrets`.

5. Build and deploy the function:

    ```bash
    faas-cli up --tag=sha
    ```

    Visit `https://gateway.example.com/function/profile`. Without a valid cookie, the watchdog shows its sign-in page. When *Sign in* is clicked, the watchdog redirects to the configured OAuth provider. After login, the browser sends the session cookie with each request to the function. The watchdog validates it before forwarding the request to the handler, enforcing authentication.

    To sign out, have the visitor's browser send a `POST` to `{oauth_base_url}/auth/logout`, for example with a sign-out form. The watchdog clears the browser's session cookie and returns the visitor to the sign-in page.

6. Optional: Add authorization rules to the handler.

    The watchdog checks for a valid session cookie from a successful login with the provider. It does not decide whether that visitor's identity may access the function or specific resources.

    The handler must enforce those rules by verifying the [session cookie](#session-cookie) and checking the identity claims it contains, such as `sub`.

## Reference

### Required configuration

Set `oauth_enabled=true` and configure the function's public URL, OIDC client ID, issuer URL, and signing key. For a provider without OIDC discovery (e.g. GitHub OAuth), use both endpoint options instead of `oauth_issuer_url`.

| Option | Usage |
| ------ | ----- |
| `oauth_enabled` | Set to `true` to enable OAuth/OIDC login and session validation. Default: `false`. |
| `oauth_base_url` | The function's public URL, including its path, such as `https://gateway.example.com/function/profile`. Used to build the callback URL, cookie path, and session issuer/audience. Register `{oauth_base_url}/auth/callback` with the provider. |
| `oauth_client_id` | The client ID registered with the provider. |
| `oauth_issuer_url` | The OIDC provider's issuer URL. The watchdog discovers its authorization, token, and keys endpoints and validates ID tokens. |
| `oauth_authorization_endpoint` | The provider's authorization URL. Use together with `oauth_token_endpoint` for providers without OIDC discovery, such as a GitHub OAuth App. |
| `oauth_token_endpoint` | The provider's token exchange URL. Use together with `oauth_authorization_endpoint` for providers without OIDC discovery. |
| `oauth_signing_key` | Name of an [OpenFaaS secret](/reference/secrets/) containing a base64-encoded random 32-byte key used to sign session cookies. |

### Optional configuration

| Option | Usage |
| ------ | ----- |
| `oauth_client_secret` | Name of an OpenFaaS secret containing the client secret. Omit for public clients without a secret that support PKCE. |
| `oauth_scopes` | Space- or comma-separated scopes. Default: `openid`. OIDC always includes `openid`. |
| `oauth_cookie_name` | Session cookie name. Default: `of_session`. Must be a valid cookie name and differ from the login cookie name. |
| `oauth_login_cookie_name` | Temporary login cookie name. Default: `of_login`. Must be a valid cookie name and differ from the session cookie name. |
| `oauth_login_redirect` | Destination after successful login. Defaults to `oauth_base_url`. |
| `oauth_session_default_ttl` | Session lifetime when the provider supplies no expiry. Default: `1h`. |
| `oauth_session_ttl` | Optional override for the session JWT and cookie lifetime, even beyond provider token expiry. The provider is not contacted again during the session. When unset, the ID token expiry, OAuth `expires_in`, or the default lifetime is used. |
| `oauth_allow_http` | Allow HTTP provider endpoints, discovery, and redirects for development. Default: `false` (HTTPS required). |
| `oauth_token_auth_method` | Client-secret authentication method: `client_secret_basic` (default) or `client_secret_post`. Unused without a client secret. |

Both session lifetime options accept positive Go durations in whole seconds, such as `30m` or `8h`.

### Login routes

The watchdog serves these routes under the function's public URL:

| Method and path | Purpose |
| --------------- | ------- |
| `GET /auth/login` | Show the built-in login page |
| `POST /auth/login` | Start login and redirect the browser to the provider |
| `GET /auth/callback` | Complete login and set the session cookie |
| `POST /auth/logout` | Clear the session cookie and return to the login page |

The login page has a button that starts the provider flow. After successful login, the watchdog sets a signed, HttpOnly session cookie (default name `of_session`) and sends the browser back to the function. Requests with no valid session cookie return to the login page, while login failures show a built-in error page. Logout clears the function's cookies but does not end the session at the provider.

### Session cookie

After a successful login, the watchdog forwards the session cookie with each authenticated request. The cookie contains a JWT issued by the watchdog and signed with HS256 using the key in `oauth_signing_key`.

The provider's access and ID tokens are discarded once login completes and are never stored in the cookie. When the provider returns an ID token, only a few identity claims are kept, following the same convention as [OpenFaaS IAM](/openfaas-pro/iam/overview/). The decoded payload looks like this:

```json
{
  "iss": "https://gateway.example.com/function/profile",
  "aud": ["https://gateway.example.com/function/profile"],
  "exp": 1800003600,
  "iat": 1800000000,
  "cookie_name": "of_session",
  "value": {
    "sub": "fed:8a2f6c1e-4d7b-4f0a-9c3e-2b5d7e9f1a4c",
    "fed:iss": "https://keycloak.example.com/realms/openfaas",
    "email": "alice@example.com",
    "name": "Alice"
  }
}
```

| Claim | Meaning |
| ----- | ------- |
| `iss`, `aud` | The function's public URL from `oauth_base_url`. These identify the watchdog-issued session, not the OIDC provider. |
| `iat`, `exp` | Unix timestamps for when the session was issued and when it expires. |
| `cookie_name` | The session cookie's name, `of_session` by default. |
| `value.sub` | The provider's subject, prefixed with `fed:`. Use this as the stable user identifier. |
| `value["fed:iss"]` | The provider's issuer URL. |
| `value.email`, `value.name` | Copied from the ID token when present. |

With plain OAuth, for example a GitHub OAuth App, there is no ID token, so `value` is empty (`{}`). The cookie then only proves that the visitor signed in with the provider, not who they are. If the function needs the visitor's identity, use an OIDC provider.

The function never receives the provider's access token, so it cannot call the provider's APIs on the visitor's behalf. If it needs to, the function must implement its own OAuth flow instead of using the watchdog.

!!! warning "Use of-watchdog 0.12.3 or later"
    Earlier releases stored the provider's tokens in the cookie in readable form. After upgrading, rotate `oauth_signing_key` to invalidate cookies issued by those releases.

#### Read the session in a handler

The watchdog validates the session cookie before forwarding each request, but the handler should still verify it before trusting its claims. Template servers listen on all interfaces inside the function's Pod, so anything that can reach the Pod directly can bypass the watchdog and send a forged cookie.

Verify the cookie with the same signing key the watchdog uses. Mount the secret, then check the following:

* the HS256 signature
* the issuer and audience, which must both equal `oauth_base_url`
* the expiry
* the `cookie_name` claim

The `oauth_base_url` environment variable is available to the handler as well as the watchdog.

For Go, using [golang-jwt](https://github.com/golang-jwt/jwt):

```go
// Session is the payload of the watchdog's of_session cookie.
type Session struct {
	jwt.RegisteredClaims
	CookieName string `json:"cookie_name"`
	Value      struct {
		Subject string `json:"sub"`
		Issuer  string `json:"fed:iss"`
		Email   string `json:"email"`
		Name    string `json:"name"`
	} `json:"value"`
}

var (
	baseURL    = os.Getenv("oauth_base_url")
	signingKey = mustReadKey("/var/openfaas/secrets/profile-signing-key")
)

func mustReadKey(path string) []byte {
	data, err := os.ReadFile(path)
	if err != nil {
		panic(err)
	}
	key, err := base64.StdEncoding.DecodeString(strings.TrimSpace(string(data)))
	if err != nil {
		panic(err)
	}
	return key
}

// readSession verifies the cookie with the function's signing key.
func readSession(r *http.Request) (*Session, error) {
	cookie, err := r.Cookie("of_session")
	if err != nil {
		return nil, err
	}
	session := &Session{}
	_, err = jwt.ParseWithClaims(cookie.Value, session,
		func(*jwt.Token) (any, error) { return signingKey, nil },
		jwt.WithValidMethods([]string{"HS256"}),
		jwt.WithIssuer(baseURL),
		jwt.WithAudience(baseURL),
		jwt.WithExpirationRequired())
	if err != nil || session.CookieName != "of_session" {
		return nil, errors.New("invalid session")
	}
	return session, nil
}
```

For Python, using [PyJWT](https://pyjwt.readthedocs.io/):

```python
import base64
import os
from http.cookies import SimpleCookie

import jwt

BASE_URL = os.environ["oauth_base_url"]

with open("/var/openfaas/secrets/profile-signing-key") as f:
    SIGNING_KEY = base64.b64decode(f.read().strip())


def read_session(cookie_header):
    """Verify the watchdog's of_session cookie and return its claims."""
    cookie = SimpleCookie(cookie_header).get("of_session")
    if cookie is None:
        return None
    try:
        claims = jwt.decode(
            cookie.value,
            SIGNING_KEY,
            algorithms=["HS256"],
            issuer=BASE_URL,
            audience=BASE_URL,
            options={"require": ["exp", "iss", "aud"]},
        )
    except jwt.InvalidTokenError:
        return None
    if claims.get("cookie_name") != "of_session":
        return None
    return claims["value"]
```

With the `python3-http` template, pass `event.headers.get("Cookie")` to `read_session`. If `oauth_cookie_name` is set, use that name instead of `of_session`.

Base authorization decisions on `value.sub`, not on `email` or `name`, which the provider may allow users to change. Do not log the raw cookie or return it to browser code.

## Related

* [Authentication for functions](/reference/authentication/)
* [Built-in function authentication with IAM](/openfaas-pro/iam/function-authentication/)
