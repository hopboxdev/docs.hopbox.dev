# Sign-in with GitHub and Google

> The sign-in with GitHub and Google: its config, redirect URIs, browser endpoints, checks, cookies, errors, limits, logs, and audit entries.

A person can sign in to an install with GitHub or Google. The operator registers an OAuth app at each provider and gives hopbox the client ID and the client secret. hopbox cannot register these apps itself, because each app has redirect URIs on the domain of one install. This page gives the config, the browser endpoints, and the checks of a sign-in. It also gives the link of an identity to a signed-in account, the cookies, the errors, the rate limits, the logs, and the audit entries. [Sessions, cookies, and sign-in links](/reference/sessions/index.html.md) covers the session that a sign-in starts. [Sign in with GitHub and Google](/guide/github-google-sign-in/index.html.md) sets up both providers step by step. The code is in `internal/auth/oauth`, `internal/service/oauth.go`, and `internal/api/oauth.go`.

## Providers

| Provider | Name in commands | Protocol | Scopes | Subject | Email |
| --- | --- | --- | --- | --- | --- |
| GitHub | `github` | OAuth 2.0 with PKCE, a GitHub OAuth app | `user:email` | the numeric user ID from `GET /user` | the primary email from `GET /user/emails`, which GitHub verified |
| Google | `google` | OpenID Connect with PKCE and a nonce, a Google OAuth client of the type "Web application" | `openid email profile` | the `sub` claim of the ID token | the `email` claim of the ID token. The claim `email_verified` must be `true`. |

hopbox takes only an email of ASCII characters from a provider. The provider verified the exact address, and a case mapping of other letters can lead to another address. For example, a case mapping makes `ceo@icorp.com` out of `ceo@İcorp.com`. An email with a character that is not ASCII thus never signs in.

The subject names the person at the provider and never changes. A later sign-in finds the account by the subject, also after the person changes the email at the provider. hopbox trusts only an email that the provider verified. At GitHub, only the primary email counts. A secondary email can be an old address that the person no longer holds, so it never links to an account.

hopbox keeps no token of a provider. The token of GitHub serves only the two reads of the sign-in. The ID token of Google serves only its checks.

A first sign-in with GitHub also keeps the GitHub login and the field `created_at` of `GET /user`. That field is the creation time of the GitHub user. The instance admins see both with the new account, as `github_login` and `github_create_time` ([the admin API](/reference/admin-api/index.html.md#accountsignup)). A GitHub user made the same day is a hint of abuse.

## Config

The client of each provider lives in the table `install_config` ([Control store tables](/reference/control-store/index.html.md#install_config)). An instance admin sets it through the admin API ([the admin API](/reference/admin-api/index.html.md#getoauthconfig)) or with `hopboxd admin config oauth` ([hopboxd](/reference/hopboxd/index.html.md#admin-config-oauth-get)).

| Field | Type | Default | Rule |
| --- | --- | --- | --- |
| Client ID | string | empty, so the provider is off | 1 to 256 printable ASCII characters, without spaces. |
| Client secret | string | none | 1 to 512 printable ASCII characters, without spaces. A client ID needs a secret. The secret stays only with the client ID that it came with. A new client ID needs the secret again. |

The control store keeps the secret sealed with the current key-encryption key ([Sealing](/reference/sealing/index.html.md#places)). The place is the install secret `oauth.github.client_secret` or `oauth.google.client_secret`. `RewrapSealedValues` covers both secrets, and `ListKEKs` counts them. The API never returns a secret, only `client_secret_set`.

A provider with a client is on, on an install with a domain. Then `GetInstallInfo` lists `SIGN_IN_METHOD_GITHUB` or `SIGN_IN_METHOD_GOOGLE` ([the API server](/reference/api/index.html.md#getinstallinfo)). A clear of the client turns the provider off. The sign-in methods of that provider stay on their accounts. After a new set of the client, they work again.

## Redirect URIs

The redirect URI of a provider is the URI to which the provider sends the browser back. Register exactly this URI at the provider:

| Provider | Redirect URI |
| --- | --- |
| GitHub | `https://<domain>/auth/github/callback` |
| Google | `https://<domain>/auth/google/callback` |

For an install with the domain `example.com`, the URIs are `https://example.com/auth/github/callback` and `https://example.com/auth/google/callback`. `hopboxd admin config oauth` prints them. Each sign-in sends this exact URI to the provider twice: in the authorization URL and in the token request. The provider refuses a code for another URI only with exact matching. Google always matches exactly. A GitHub OAuth app with one callback URL can match by wildcard: it then also takes a URI on a subdomain or in a subdirectory of the callback URL. A machine URL is a subdomain of the install that serves the content of a user. So the operator must turn off wildcard matching for the callback URL of the GitHub app ([Sign in with GitHub and Google](/guide/github-google-sign-in/index.html.md#register-the-github-oauth-app)). A change of the domain changes the URIs, and the operator must register the new URIs at each provider.

## Endpoints

Each provider has two endpoints on the API listener. The edge servers forward them, with the rest of the domain of the install ([the edge role](/reference/edge/index.html.md#forwarded-requests)). They are plain HTTP `GET` requests, not Connect calls, because the browser goes to them through links and redirects. Another method gets `405 Method Not Allowed`.

| Endpoint | Query parameters | Answer |
| --- | --- | --- |
| `GET /auth/{provider}/start` | `return_to`: optional, the path on the install to open after the sign-in. The default is `/`. `link`: optional, `1` for a link of the identity to the account of the session ([Link to a signed-in account](#link-to-a-signed-in-account)). Another value of `link`, or two of them, gets `400`. | `303 See Other` to the authorization URL of the provider, with the state cookie |
| `GET /auth/{provider}/callback` | `code`, `state`, and `error`, which the provider sets | `303 See Other` to the return path, with the session cookie. A link sets no session cookie. |

`{provider}` is `github` or `google`. Another name gets `404 Not Found`.

Each answer has `Cache-Control: no-store`. The callback also has `Referrer-Policy: no-referrer`, because its URL holds the code.

### Start

A start makes a flow: a new state, a PKCE code verifier, and a nonce. The state is 32 random bytes in base64url, 43 characters. The browser gets the state in the URL of the provider and in the state cookie. The verifier never leaves the server before the token request. The URL of the provider has the client ID, the redirect URI, the scopes, and the state. It also has the PKCE challenge with the method `S256`, and for Google the nonce.

The control store keeps the flow in the table `oauth_flows` for 10 minutes ([Control store tables](/reference/control-store/index.html.md#oauth_flows)). It keeps only the SHA-256 of the state.

Only a page of the install, or the person, can start a sign-in. The start refuses a request whose header `Sec-Fetch-Site` is `cross-site` or `same-site`. A browser sets this header on its own, and a page cannot change it. So a page of another site cannot start a sign-in in the browser of a visitor. A machine URL such as `https://box1.example.com` is `same-site`, so a page that a machine serves cannot start one either. A request without the header passes, and so does a typed URL or a bookmark, which has the value `none`.

### Return path

The return path must be a plain path on the install, so a link to the start can never send the browser to another site. A return path must have all of these properties:

- It starts with exactly one `/`.
- It has no scheme and no host. `https://evil.example/` and `//evil.example` fail.
- It has no backslash, no space, and no control character.
- It has no `.` or `..` segment, also in the encoded form `%2e%2e`.
- It has at most 512 characters.

A query and a fragment can follow the path, for example `/device?code=ABCD-EFGH`. The flow keeps the return path, and the callback sends the browser to it as it is, without a decode. A client that builds the start link, such as the console ([Console](/reference/console/index.html.md#sign-in)), must pass the path as it is too. It must never decode `return_to` before it sends the browser there, because a decode can turn a safe path such as `/%2F%2Fevil.example` into `//evil.example`. Thus a change of the URL between the start and the callback has no effect on it.

### Callback

The callback ends a sign-in in these steps. Any failed step ends the sign-in with an error page.

1. The callback limit of the client IP address must allow the request ([Rate limits](#rate-limits)).
2. The response clears the state cookie, whatever the result.
3. The `state` parameter must have the form of a token and must equal the state cookie of the browser. The compare takes the same time for any value.
4. One statement marks the flow used. It must exist, have the same provider, be unused, and be within its 10 minutes. So a flow works once, also with two callbacks at the same time.
5. If the provider sent `error`, the sign-in ends. The flow stays used.
6. The server trades the code for a token at the provider, with the client secret, the redirect URI, and the PKCE verifier.
7. For GitHub, the server reads `GET /user` and `GET /user/emails`. For Google, it checks the ID token ([Checks of the ID token of Google](#checks-of-the-id-token-of-google)).
8. The provider must prove a verified email. The server then signs the person in, as [Sessions, cookies, and sign-in links](/reference/sessions/index.html.md#what-a-sign-in-does) gives. A flow of a link adds the identity to the account instead ([Link to a signed-in account](#link-to-a-signed-in-account)).
9. The response sets the session cookie and sends the browser to the return path. After a link, it only sends the browser to the return path.

The callback is the one request that sets the session cookie without the `Origin` check of [Sessions, cookies, and sign-in links](/reference/sessions/index.html.md#calls-with-the-cookie). The provider sends the browser back from its own site, so the request has no `Origin` of the install. Step 3 replaces the check. Another site can start a flow in its own browser and send the callback URL to a victim. But the browser of the victim does not have the state cookie of that flow, so the callback fails. A machine subdomain cannot set the cookie either, because of the `__Host-` prefix.

The calls to the provider run outside of any database transaction, with a limit of 20 s for the whole exchange.

If the browser has a live session, the new session replaces it, as for any sign-in.

### Link to a signed-in account

A person who is signed in links a GitHub user or a Google account to the account ([Sessions, cookies, and sign-in links](/reference/sessions/index.html.md#link-a-new-sign-in-method)). Once the account has an identity of a provider, this is the only way to add another one. The browser opens the start with `link=1`, and the console sends it there from `/account`:

```text
https://example.com/auth/google/start?link=1&return_to=%2Faccount
```

The start reads the session cookie of the browser. It checks the session in this order:

| Session | Answer |
| --- | --- |
| No live session | `303 See Other` to `/signin?return_to=<path>`. After the sign-in, the console opens the return path, where the person starts the link again. |
| A session whose sign-in is more than 10 minutes old ([Sessions, cookies, and sign-in links](/reference/sessions/index.html.md#recent-sign-in)) | `303 See Other` to `/signin?again=1&return_to=<path>`. |
| A session with a recent sign-in | The start of a sign-in, as above. The flow also records the session in `oauth_flows.link_session_id`. |

A return path of `/` leaves out `return_to`. A return path that breaks a rule gets the error page instead of the redirect. The limits of starts count a link as a start.

The callback of a link must come with the session cookie of the same session. Another session, or a browser without a session, gets `400` at step 4, and the flow is used. The exchange with the provider runs as for a sign-in. The callback then adds the identity to the account, with the checks of [Sessions, cookies, and sign-in links](/reference/sessions/index.html.md#link-a-new-sign-in-method). It sets no session cookie, and the session of the browser stays. The callback sends the browser to the return path, for example `/account`.

### Checks of the ID token of Google

The server reads the discovery document of the issuer `https://accounts.google.com` once per process and keeps it. A failed read is not kept, so the next sign-in reads it again. The ID token must pass these checks:

| Check | Error |
| --- | --- |
| The token response holds an `id_token`. | the provider failed |
| The signature matches a key of the `jwks_uri` of Google. | the provider refused the sign-in |
| The issuer is the issuer of Google. | the provider refused the sign-in |
| The audience holds the client ID of the install. | the provider refused the sign-in |
| The token is not expired. | the provider refused the sign-in |
| The `nonce` claim equals the nonce of the flow. | the provider refused the sign-in |
| The `sub` claim is not empty. | the provider refused the sign-in |
| The `email` claim is not empty, and `email_verified` is `true` or `"true"`. | no verified email |

## Cookies

The start sets the state cookie, and the callback clears it:

```text
Set-Cookie: __Host-hopbox_oauth_github=<state>; Path=/; Expires=<end>; Max-Age=600; HttpOnly; Secure; SameSite=Lax
```

| Attribute | Value | Reason |
| --- | --- | --- |
| Name | `__Host-hopbox_oauth_github` or `__Host-hopbox_oauth_google` | The `__Host-` prefix keeps the cookie on the host of the install. A machine subdomain cannot set it or read it. |
| `Max-Age` | 600, the 10 minutes of the flow | An old cookie cannot match a new flow. |
| `HttpOnly` and `Secure` | set | Scripts cannot read the state, and the browser sends it only over HTTPS. |
| `SameSite` | `Lax` | The provider sends the browser back with a top-level `GET` from its own site. A browser sends a `Lax` cookie with such a request, but not a `Strict` cookie. |

One browser has one state cookie per provider. A second start with the same provider replaces the cookie, so the first flow then fails at its callback with `400`.

The callback sets the session cookie of [Sessions, cookies, and sign-in links](/reference/sessions/index.html.md#the-session-cookie).

## Errors

Each error answers with a plain text page, `Content-Type: text/plain; charset=utf-8`, `X-Content-Type-Options: nosniff`, and `Content-Security-Policy: default-src 'none'`. A browser thus never runs markup from a URL parameter. The page has this form:

```text
The sign-in failed: <message>

Request ID: <request ID>
```

| Status | When | Message |
| --- | --- | --- |
| `400 Bad Request` | The state is missing, malformed, not the state of the browser, unknown, used, or expired. The callback has no code. | `the sign-in is not valid: it did not start in this browser, it was used, or it expired. Start the sign-in again` |
| `400 Bad Request` | Two tabs of one browser started the same provider. The second start replaced the state cookie of the first, so the first callback fails. | the same message as above |
| `400 Bad Request` | The return path breaks a rule. | `invalid argument: the return path must be a path on the install, such as /orgs: ...` |
| `403 Forbidden` | The start came from another site. | `start the sign-in from a page of this install, or open its link yourself` |
| `403 Forbidden` | The provider sent `error`. For example, the person clicked Cancel at the provider. | `the provider did not approve the sign-in: GitHub answered access_denied. Start the sign-in again` |
| `403 Forbidden` | The provider refused the code, the token, or the ID token. | `the provider did not approve the sign-in: GitHub refused the sign-in code. Start the sign-in again. If this repeats, the operator must check the GitHub client` |
| `403 Forbidden` | The email of the provider has a character that is not ASCII. | `the provider has not verified an email of this account: hopbox cannot use the email that GitHub gave: invalid email: ... holds a character that is not ASCII` |
| `403 Forbidden` | The provider proved no verified email. | `the provider has not verified an email of this account: verify an email at GitHub, make it the primary email there, and sign in again` |
| `403 Forbidden` | An instance admin suspended the account of the sign-in ([the admin API](/reference/admin-api/index.html.md#suspendaccount)). No session starts. The flow is used, so a new sign-in starts again at the provider ([Sessions, cookies, and sign-in links](/reference/sessions/index.html.md#what-a-sign-in-does)). | `permission denied: an instance admin suspended this account, so it cannot sign in. The reason: abuse report 1` |
| `403 Forbidden` | An entry of the block list matches a first sign-in of a new person ([Sessions, cookies, and sign-in links](/reference/sessions/index.html.md#limits-of-new-accounts)). No account starts. The flow is used. | `permission denied: this install takes no new account for this sign-in` |
| `429 Too Many Requests` | A first sign-in of a new person, and the client address that started the flow made 3 accounts in the last 24 hours ([Sessions, cookies, and sign-in links](/reference/sessions/index.html.md#limits-of-new-accounts)). | `too many requests: this client address made the most new accounts of a day. Retry tomorrow` |
| `429 Too Many Requests` | A first sign-in of a new person in the sign-up mode `approval`, and the install holds 50 pending accounts. | `too many requests: this install takes no new accounts now. Retry later` |
| `503 Service Unavailable` | A first sign-in of a new person in the sign-up mode `approval`, while a live control process has a schema version below 62, or in the mode `open`, below 63. The answer has `Retry-After: 30`. | `unavailable: the install upgrades now, so it takes no new accounts. Retry later` |
| `400 Bad Request` | The parameter `link` of the start is not `1`, or comes twice. | `invalid argument: the parameter link must be 1` |
| `409 Conflict` | The account of the verified email has an identity of a provider, so a new method joins it only through a link ([Sessions, cookies, and sign-in links](/reference/sessions/index.html.md#what-a-sign-in-does)). | `the sign-in conflicts with a sign-in method of the account: the account of dana@example.org signs in with GitHub or Google, so a new sign-in method joins it only through a link. Sign in with a method of the account, then link Google on the account page. If you lost every method of the account, the operator removes the lost one with hopboxd admin account sign-in-method remove` |
| `409 Conflict` | A link: the identity signs in to another account. | `the sign-in conflicts with a sign-in method of the account: this Google identity signs in to another account. Remove it from that account first` |
| `409 Conflict` | A link: the account has another identity of the same provider. | `the sign-in conflicts with a sign-in method of the account: the account already signs in with another GitHub identity. Remove that method first, then link the new one` |
| `403 Forbidden` | A link: the session ended between the start and the callback. | `this call needs a credential: the session that started the link ended. Sign in again, then link the method again` |
| `409 Conflict` | A link: the account has this identity already. | `already exists: the account already signs in with this GitHub identity` |
| `404 Not Found` | The provider name is unknown. | `no sign-in provider "gitlab". Use github or google` |
| `404 Not Found` | The account of the identity was deleted during the sign-in ([Sessions, cookies, and sign-in links](/reference/sessions/index.html.md#what-a-sign-in-does)). The flow is used, so start the sign-in again. It then makes a new account. | `not found: the account of this sign-in method was deleted during the sign-in. Sign in again to make a new account` |
| `404 Not Found` | The install has no client for the provider, or no domain. | `failed precondition: this install has no GitHub client, so it offers no sign-in with GitHub` |
| `429 Too Many Requests` | A rate limit refused the request. | the message of the limit, see below |
| `502 Bad Gateway` | The provider did not answer, or answered with an error of its own. | `unavailable: GitHub did not answer. Start the sign-in again later` |
| `500 Internal Server Error` | Any other failure, such as a control store that does not answer. | `internal error. The server log has the details under the request ID` |

The messages say Google instead of GitHub for a sign-in with Google. A value of `error` from the URL that is not 1 to 64 letters, digits, `_`, `.`, or `-` shows as `unknown`.

## Recover a lost identity

A person can lose the GitHub user or the Google account that the account signs in with. A sign-in with a new GitHub user and a verified email of the account then gets `409`. The account has an identity of a provider, so the email links no new method ([Sessions, cookies, and sign-in links](/reference/sessions/index.html.md#what-a-sign-in-does)). The rule stops the new owner of a lapsed domain, who can prove an old address of the account. The person recovers in one of two ways.

With another sign-in method of the account, such as its email link or Google:

1. Sign in with that method.
2. On `/account`, remove the old GitHub method. The API call is `RemoveSignInMethod` ([the sign-in, account, and audit API](/reference/auth-api/index.html.md#removesigninmethod)).
3. On `/account`, select "Link GitHub", and approve the new GitHub user at GitHub ([Link to a signed-in account](#link-to-a-signed-in-account)).

Without another method, for example on an install without SMTP, an instance admin removes the lost method on a control server:

```sh
sudo hopboxd admin account sign-in-method list --email dana@example.org
sudo hopboxd admin account sign-in-method remove --account-id <account> --method-id <method> --allow-last
```

The account then has no method, and so no identity of a provider. A sign-in that proves a verified email of the account links to it again. No other account holds an email link of that email, because a link refuses the verified email of another account ([Sessions, cookies, and sign-in links](/reference/sessions/index.html.md#link-a-new-sign-in-method)) ([hopboxd](/reference/hopboxd/index.html.md#admin-account-sign-in-method-remove)). The person signs in with a magic link to that email, or with the new GitHub user. Then the person links the other methods from `/account`.

The operator must first make sure that the person holds the email. Otherwise the removal hands the account to the holder of the address. The audit log of the personal org records the removal, with the actor.

The message of the `409` names both ways.

## Rate limits

| Limit | Value | Where it counts | Message |
| --- | --- | --- | --- |
| Starts from one client: one IPv4 address, or one IPv6 /64 | 30 per hour, for both providers together | the rows of `oauth_flows` in the control store, so it holds across every control server | `too many requests: this client started the most sign-ins of an hour. Wait, then retry` |
| Starts of the whole install | 5000 per hour (`service.OAuthStartsPerHour`), for both providers together, for every client, also a loopback peer | the rows of `oauth_flows` in the control store | `too many requests: the install started the most sign-ins with GitHub and Google of an hour. Wait, or sign in with an email link` |
| Callbacks from one client: one IPv4 address, or one IPv6 /64 | 60 per hour | the memory of each `hopboxd` process | `too many requests: this client sent the most sign-in callbacks of an hour. Wait, then retry` |

Each callback that reaches a provider needs a start, so the limit of starts also bounds the calls to the providers. The limit of callbacks keeps a flood of bad callbacks away from the control store. It tracks at most 100,000 clients per process. When it holds that many, it first drops the clients whose hour ended. If it is still full, it lets each new client pass without a count, and keeps the counts of the clients that it holds. So a flood of new addresses cannot reset the count of a client. It cannot lock every new client out for an hour either. Each callback still needs a start, and the limits of starts hold.

The limit of the install bounds the rows of `oauth_flows`. It holds also against many clients at once. Its cost: such a flood can block the sign-ins with GitHub and Google of everyone for up to an hour. The email links still work then. When the install reaches the limit, the service logs `the install reached its limit of OAuth sign-ins of an hour` at level warn, with `limit` and `request_id`.

As for magic links, the limits of a client skip a loopback peer, such as `127.0.0.1` or `::1` ([Sessions, cookies, and sign-in links](/reference/sessions/index.html.md#rate-limits)). A loopback peer is a process or an SSH tunnel on the same server, which stands for every client. Behind the edge, the peer is the address of the client that the edge forwards. Only a server with the edge role can name it ([the internal listener](/reference/internal-listener/index.html.md#forwarded-requests)). Each start takes a transaction lock (`LockKeyOAuthFlows`, [the bundled Postgres](/reference/postgres/index.html.md#advisory-lock-keys)), so starts at the same time cannot pass a limit together.

## Logs

Each record has `component=api` or `component=service`, and never holds a state, a code, a verifier, a nonce, a token, or a secret.

| Record | Level | Fields | When |
| --- | --- | --- | --- |
| `oauth sign-in started` | info | `provider`, `link`, `request_id` | A start sent the browser to the provider. `link` is `true` for a link to a signed-in account. |
| `oauth sign-in start refused` | info | `provider`, `sec_fetch_site`, `request_id` | A start came from another site. |
| `oauth sign-in start failed` | warn | `provider`, `err`, `request_id` | The discovery document of Google did not load. |
| `oauth sign-in` | info | `provider`, `account`, `new_account`, `request_id` | A callback signed a person in. |
| `oauth link` | info | `provider`, `account`, `request_id` | A callback linked an identity to the account of the session. |
| `oauth sign-in refused` | info | `provider`, `request_id`, `reason`, and `err` for an email | A callback failed a check of the state, had no code, or had no verified email. The callback of a link came without its session: `the link did not come back with the session that started it`. |
| `oauth sign-in refused` | warn | `provider`, `request_id`, `reason`, `subject` | The account of the email has an identity of a provider: `the account needs a link for a new sign-in method`. Or a link found the identity on another account, or another identity of the provider on the account: `the identity or the account holds another link`. |
| `oauth sign-in denied by the provider` | info | `provider`, `request_id`, `error` | The callback had the `error` parameter. |
| `oauth sign-in refused by the provider` | warn | `provider`, `request_id`, `err` | The provider refused the code or the token. A repeat can mean a wrong client secret or redirect URI. |
| `oauth sign-in failed` | warn or error | `provider`, `err`, `request_id` | The provider did not answer (warn), or another failure (error). |
| `the install reached its limit of OAuth sign-ins of an hour` | warn | `limit`, `request_id` | A start found 5000 starts in the last hour. |

## Audit entries

A sign-in writes the entries of [the sign-in, account, and audit API](/reference/auth-api/index.html.md#audit-entries) in its transaction. For GitHub and Google, `auth.sign_in` has the `method` `github` or `google`, and these details:

| Detail | Value |
| --- | --- |
| `email` | the verified email, in lower case |
| `new_account` | `true` for a first sign-in |
| `oauth_flow` | the UUID of the flow |
| `subject` | the subject at the provider |
| `login` | the GitHub login. Google has no login, so a sign-in with Google has no `login`. |

A first sign-in also writes `account.create`, and a link by a verified email writes `account.link_sign_in_method` with the `kind` `github` or `google`. A link to a signed-in account writes only `account.link_sign_in_method`, with the session as the actor. Its details are `kind`, `email`, `session`, `oauth_flow`, `subject`, and for GitHub `login`. Each `account.link_sign_in_method` of an instance admin also has `instance_admin` set to `true`, and tells the instance admins ([the admin API](/reference/admin-api/index.html.md#notices-to-instance-admins)). A start, a refused callback, and a failed callback write no entry, because no account acts. The log records above cover them.

The calls of the config write the entries of [the admin API](/reference/admin-api/index.html.md#getoauthconfig): `oauth_config.get`, `oauth_config.set`, and `oauth_config.clear`.

## Sweep

The leader deletes the flows that are older than 24 hours, every 10 minutes, with the sign-in links ([Sessions, cookies, and sign-in links](/reference/sessions/index.html.md#sweep)). A flow stays after its use or expiry for the limit of starts.
