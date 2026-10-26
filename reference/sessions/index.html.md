# Sessions, cookies, and sign-in links

> The session cookie, the lifetimes, the check against cross-site requests, the magic links and login links, what a sign-in does, and the rate limits.

A person signs in to the console of an install with a magic link, with GitHub, or with Google. On the first day, the first instance admin signs in with a first-admin login link. A sign-in starts a session, and a cookie holds the session in the browser. This page gives the cookie, the lifetimes, the check against cross-site requests, the recent sign-in, the form of the links, and the rate limits. [The sign-in, account, and audit API](/reference/auth-api/index.html.md) lists the calls. [Sign-in with GitHub and Google](/reference/oauth/index.html.md) covers the sign-in with GitHub and Google. [CLI tokens, API keys, admin API keys, and SSH keys](/reference/credentials/index.html.md) covers the credentials of the hopbox CLI, scripts, and SSH clients. The code is in `internal/auth`, `internal/service`, and `internal/api`.

## Lifetimes

| Credential | Lifetime | Use | Source |
| --- | --- | --- | --- |
| Session | 30 days after the sign-in. After 7 days without a call, it ends before that. | many calls | A phase 1 default |
| Recent sign-in of a session | 10 minutes after the sign-in | the calls that need it ([Recent sign-in](#recent-sign-in)) | A phase 2 default |
| Magic link | 15 minutes | once | the same |
| First-admin login link | 15 minutes | once | the same |
| Flow of a sign-in with GitHub or Google | 10 minutes | once | [Sign-in with GitHub and Google](/reference/oauth/index.html.md#start) |
| Invitation to an org | 7 days | once | [The org API](/reference/org-api/index.html.md#invitations) |
| Code of a private machine URL | 60 seconds | once | [The edge role](/reference/edge/index.html.md#private-machine-urls) |
| Token of a private machine URL, in the cookie `__Host-hopbox_machine` | as long as the session that made it | many requests | [The edge role](/reference/edge/index.html.md#private-machine-urls) |

All times use the clock of Postgres. A session records its last use at most once per minute, so the idle time of 7 days is exact to one minute. In maintenance mode, a session records no use ([the API server](/reference/api/index.html.md#maintenance-mode)). Only a call that passes the check against cross-site requests records a use ([Calls with the cookie](#calls-with-the-cookie)).

## Tokens

Each session, magic link, login link, and invitation to an org carries a token. So does the state of a sign-in with GitHub or Google. A token is 32 random bytes, in base64url without padding, so 43 characters. The control store keeps only the SHA-256 of a token ([Control store tables](/reference/control-store/index.html.md#sessions)). A copy of the database thus signs nobody in. The audit log never holds a token, and the server never logs one.

A token of another form never matches a stored hash, so the server refuses it before any lookup.

## The session cookie

A sign-in sets this cookie:

```text
Set-Cookie: __Host-hopbox_session=<token>; Path=/; Expires=<end>; Max-Age=<seconds>; HttpOnly; Secure; SameSite=Lax
```

| Attribute | Value | Reason |
| --- | --- | --- |
| Name | `__Host-hopbox_session` | With the `__Host-` prefix, the browser keeps the cookie only from a secure origin, with `Path=/` and without `Domain`. So the cookie belongs to the host of the install only, and never reaches a machine URL such as `box1.example.com`. The edge also drops each cookie whose name starts with `__Host-hopbox_` from the requests to a machine and from its answers ([the edge role](/reference/edge/index.html.md#private-machine-urls)). |
| `Path` | `/` | The prefix needs it. |
| `Expires` and `Max-Age` | the end of the session, 30 days after the sign-in | At the end of the session, the browser drops the cookie. The server also ends an idle session. |
| `HttpOnly` | set | Scripts of the page cannot read the token. |
| `Secure` | set | The browser sends the cookie only over HTTPS. |
| `SameSite` | `Lax` | The browser sends the cookie with a link from another site, but not with a form post or a script call from another site. |

`SignOut` and `DeleteAccount` clear the cookie with `Max-Age=0`. They also end the private machine URLs of the session: the delete of the session deletes their tokens, and the edge servers check each URL again ([the edge role](/reference/edge/index.html.md#private-machine-urls)). A browser sends a cookie on its own, also after its session ended. So a cookie that names no live session counts as no cookie, and the response clears it. A stale cookie thus never blocks the next sign-in.

## Calls with the cookie

The session resolver of the request gate reads the cookie ([the API server](/reference/api/index.html.md#request-gate)). A call that carries the cookie must also pass this check against cross-site request forgery:

1. The `Origin` header is the origin of the install: `https://` and the domain of the install, for example `https://example.com`. The API and the console share this origin.
2. The call uses the Connect protocol, with the header `Connect-Protocol-Version: 1`.

A form or a link of another site can send neither. A script of another site can send the Connect header only after a CORS preflight, and hopbox grants none. A machine URL such as `https://box1.example.com` is another origin, so a page that a machine serves cannot act with the cookie either.

The gate resolves a call with the cookie in these steps:

| Case | Result |
| --- | --- |
| The call has no session cookie. | The session resolver passes the call to the next resolver. |
| The token names no live session: unknown, malformed, signed out, idle for 7 days, or past its 30 days. | The cookie counts as absent, and the call goes on as for a request without it. The response clears the cookie, unless the call sets a new one, as a sign-in does. A call that needs a credential then gets `unauthenticated` with the message `this call needs a credential`. |
| The session is live, but the call fails the check above. | `unauthenticated` with the message `the credential is not valid`. The response keeps the cookie, so a page of another site cannot sign anybody out. The log record `api credential refused` has the reason, for example `the Origin header "https://evil.example" is not "https://example.com"`. |
| The install has no domain. | `unauthenticated`, as above. Without a domain, the install has no origin to trust. |
| The control store does not answer. | `unavailable` with the message `the server failed to check the credential. Retry later`. |
| The session is live, and the call passes the check. | The caller is `session`, with the account, the session, whether the account is an instance admin, and whether the sign-in is recent. The gate reads the instance admin role on each call, so a revoke takes effect at once. The call records the use of the session. |

So a cookie call through gRPC, gRPC-Web, or a Connect GET request fails. The console uses Connect POST requests.

A WebSocket is the one exception to step 2. A browser cannot add a header to a WebSocket. So a request with the header `Upgrade: websocket` passes with the `Origin` header of step 1 alone (`auth.CheckWebSocketRequest`). This is safe, because only the WebSocket API of a browser sends that header, and it always sends the true origin of the page. Only the endpoint `/api/ws/exec` takes a WebSocket ([the machine API](/reference/machine-api/index.html.md#the-websocket-endpoint)).

The calls that set the cookie, `VerifyMagicLink` and `VerifyLoginLink`, must pass the same check, also without a cookie. Otherwise a page of another site can sign a browser in to an account of its choice. A call that fails it gets `permission_denied` with the message `the request does not come from the console of the install: ...`, and the link stays unused.

The callback of a sign-in with GitHub or Google also sets the cookie. The provider sends the browser to it from its own site, so it cannot pass the check. The state cookie of the browser takes the place of the check ([Sign-in with GitHub and Google](/reference/oauth/index.html.md#callback)).

## Recent sign-in

A session counts as recent for 10 minutes after its sign-in. The time of the sign-in is the column `created_at` of the session ([Control store tables](/reference/control-store/index.html.md#sessions)). The use of a session does not change it, so only a new sign-in makes a session recent again. The gate reads it with the clock of Postgres on each call.

A call that a session taken over must not reach needs a recent sign-in. These calls need one:

| Call | Reason |
| --- | --- |
| `ApproveDevice` ([the sign-in, account, and audit API](/reference/auth-api/index.html.md#approvedevice)) | The approval makes a CLI sign-in whose refresh token lives 30 days and renews with each use. So the CLI sign-in can outlive the session. |

The rule applies only to a console session, because only a session has a sign-in in a browser. Such a call first runs its policy check, then the check of the sign-in:

| Caller | Result |
| --- | --- |
| A session whose sign-in is at most 10 minutes old | The call goes on. |
| A session whose sign-in is more than 10 minutes old | `permission_denied` with the message `permission denied: this action needs a sign-in within the last 10 minutes. Sign in again, then retry`, and the detail `hopbox.v1.RecentSignInRequired` with `max_age` 10 minutes ([the sign-in, account, and audit API](/reference/auth-api/index.html.md#recentsigninrequired)). |
| Any other caller, such as a CLI token, an API key, or local root | The check of the sign-in passes it, and the policy of the call decides. For example, `ApproveDevice` takes only a session. So a CLI token or a key gets the policy refusal `permission denied: a caller of kind <kind> may not do auth.approve_device`, without the detail. An anonymous caller gets `unauthenticated`. |

Only the refusal of an old session has the detail. So a client knows from the detail that a new sign-in fixes the error. The other calls of an old session keep working.

To get a recent sign-in, the person signs in again in the same browser. The new sign-in ends the old session of that browser and starts a new one ([What a sign-in does](#what-a-sign-in-does)). The sessions of the account in other browsers stay. The console shows the sign-in page for this with `/signin?again=1` ([Console](/reference/console/index.html.md#sign-in)).

The code of the check is `service.RequireRecentSignIn`. An operation calls it after its policy check. The check refuses only a session with an older sign-in, so an admin key and local root keep the calls that their policy allows.

## Magic links

A magic link is a sign-in link that hopbox sends by email. It needs the domain of the install, and the SMTP config of the install ([SMTP config](/reference/smtp/index.html.md)). A dev or test install can use the file mail transport in place of the SMTP config ([Mail](/reference/mail/index.html.md#the-file-transport)).

```text
https://example.com/auth/email#token=<token>
```

The token is in the fragment, after `#`. A browser never sends the fragment to a server. So the token stays out of the access logs of servers and proxies, and out of `Referer` headers. The console page `/auth/email` reads the token and calls `VerifyMagicLink`.

The email has this form:

```text
From: hopbox <no-reply@example.org>
To: ann@example.org
Subject: Sign in to example.com

Open this link to sign in to example.com:

https://example.com/auth/email#token=<token>

The link works once, for 15 minutes. If you did not ask for it, you can ignore this email.
```

The server sends the email in the call, after the link commits, and waits at most 20 s for the SMTP server. With the file transport, the server writes the email to a file in `<data_dir>/mail` and sends nothing.

### Rate limits

`RequestMagicLink` counts the links of the last hour in the control store, so the limits hold across every control server.

| Limit | Value | Error |
| --- | --- | --- |
| Links for one email | 5 per hour | `resource_exhausted`: `too many requests: this email got the most sign-in links of an hour. Use one of them, or wait` |
| Links from one client | 20 per hour | `resource_exhausted`: `too many requests: this client asked for the most sign-in links of an hour. Wait, then retry` |

A client is one IPv4 address, or one IPv6 /64, because one IPv6 host often holds a whole /64. An IPv4 address in IPv6 form counts as its IPv4 address. The address is the peer of the request. Behind the edge, the peer is the address of the client that the edge forwards. Only a server with the edge role can name it ([the internal listener](/reference/internal-listener/index.html.md#forwarded-requests)). The limit of a client skips a loopback peer, such as `127.0.0.1` or `::1`. A loopback peer is a process or an SSH tunnel on the same server, on the API listener. It stands for every client, so a limit on it blocks everyone. Each request takes a transaction lock (`LockKeyMagicLinks`, [the bundled Postgres](/reference/postgres/index.html.md#advisory-lock-keys)), so requests at the same time cannot pass a limit together. A request whose email fails to send still counts. The messages never say how many links an email got, so they tell nothing about the activity of others.

## First-admin login links

`sudo hopboxd admin login-link --email <email>` prints a login link ([hopboxd](/reference/hopboxd/index.html.md#admin-login-link)). The link is for the first instance admin of a new install, which has no SMTP config and no OAuth app yet.

```text
https://example.com/auth/login-link#token=<token>
```

The console page `/auth/login-link` reads the token and calls `VerifyLoginLink`. The link works once, for 15 minutes. It needs the domain of the install.

## What a sign-in does

Each sign-in proves an identity: a method and an email. It then reaches one account:

1. If the account of an earlier sign-in holds the method, the sign-in reaches that account. A magic link names its method by the email. A login link names its method by the email too, apart from magic links. GitHub and Google name their method by the subject at the provider, so a new email at the provider reaches the same account.
2. Else, the method can prove the email, and an account can hold that email as verified. In that case, the sign-in links the method to that account. A magic link proves its email. GitHub and Google prove the email that the provider verified, and nothing else passes ([Sign-in with GitHub and Google](/reference/oauth/index.html.md#providers)). A login link proves no email.

   A link of GitHub or Google has one more rule. The account must not hold a method of the same provider with another subject, whatever its email. For example, an account with the GitHub user 1 never links the GitHub user 2. A domain can lapse, and its new owner can then prove an old address of the account at GitHub. The email of a method follows the provider, so a rule by email misses an old address. The rule stops only a second identity of the same provider. The holder of the mailbox of a lapsed domain still has other ways in. That person can sign in with a magic link, or with a provider that the account has not used yet. Such a sign-in fails with `failed_precondition`, or `409` at the callback of [Sign-in with GitHub and Google](/reference/oauth/index.html.md#errors). It changes nothing. A person who lost the old identity signs in another way, removes the old method, and signs in again. Without another method, the operator removes it with `hopboxd admin account sign-in-method remove` ([Sign-in with GitHub and Google](/reference/oauth/index.html.md#recover-a-lost-identity)).
3. Else, the sign-in creates an account with the email, the method, and a personal org named `Personal`, with the account as its owner. A magic link, GitHub, and Google mark the email verified. A login link leaves it unverified. GitHub and Google also give the account the name of the person at the provider as its display name. The display name stays empty for a name that breaks the rules of [Domain rules](/reference/domain-rules/index.html.md#display-names). At GitHub, an empty name falls back to the login.

So the email of a login link never links to another account. A magic link, GitHub, or Google with the same email reaches another account. A second login link for the same email reaches the account of the first one. A verified email belongs to at most one account.

A sign-in also does these steps:

- A login link makes the account an instance admin. An account that is one already stays one.
- A sign-in from a browser that has a session ends that session. The new one replaces it.
- It starts a session and records the sign-in in the audit log ([the sign-in, account, and audit API](/reference/auth-api/index.html.md#audit-entries)). A link of a method to an account by its verified email gets its own entry.

A refused link writes the log record `sign-in link refused` at level info, with `component=service` and `kind`, never the token.

Two first sign-ins with one email at the same time make one account: each sign-in takes a transaction lock (`LockKeySignIn`).

## Sweep

Every 10 minutes, the leader deletes the ended sessions. It also deletes the magic links, the login links, and the flows of GitHub and Google that are older than 24 hours ([hopboxd](/reference/hopboxd/index.html.md#leader-jobs)). It deletes the codes of machine URLs that expired before their trade too. The delete of a session also deletes the tokens of the machine URLs that it made ([Control store tables](/reference/control-store/index.html.md#machine_url_sessions)). A link stays after its use for the rate limits. The same sweep deletes the ended CLI sign-ins and the old device sign-ins ([CLI tokens, API keys, admin API keys, and SSH keys](/reference/credentials/index.html.md#sweep)).
