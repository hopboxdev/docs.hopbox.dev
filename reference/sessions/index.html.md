# Sessions, cookies, and sign-in links

> The session cookie, the lifetimes, the check against cross-site requests, the magic links and login links, what a sign-in does, the sign-up modes, the limits of new accounts, the pending accounts, and the rate limits.

A person signs in to the console of an install with a magic link, with GitHub, or with Google. On the first day, the first instance admin signs in with a first-admin login link. A sign-in starts a session, and a cookie holds the session in the browser. This page gives the cookie, the lifetimes, the check against cross-site requests, and the recent sign-in. It also gives the form of the links, the sign-up modes, the pending accounts, the link of a new sign-in method, and the rate limits. [The sign-in, account, and audit API](/reference/auth-api/index.html.md) lists the calls. [Sign-in with GitHub and Google](/reference/oauth/index.html.md) covers the sign-in with GitHub and Google. [CLI tokens, API keys, admin API keys, and SSH keys](/reference/credentials/index.html.md) covers the credentials of the hopbox CLI, scripts, and SSH clients. The code is in `internal/auth`, `internal/service`, and `internal/api`.

## Lifetimes

| Credential | Lifetime | Use | Source |
| --- | --- | --- | --- |
| Session | 30 days after the sign-in. After 7 days without a call, it ends before that. | many calls | A phase 1 default |
| Recent sign-in of a session | 10 minutes after the sign-in | the calls that need it, such as the Admin area of the console ([Recent sign-in](#recent-sign-in)) | A phase 2 default |
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
| `RequestMagicLink` with `link` ([the sign-in, account, and audit API](/reference/auth-api/index.html.md#requestmagiclink)) | The link adds a sign-in method to the account. The method outlives the session. |
| `GET /auth/{provider}/start?link=1` ([Sign-in with GitHub and Google](/reference/oauth/index.html.md#link-to-a-signed-in-account)) | The same. This endpoint is not a call of the API. It sends an older session to the sign-in page of the console instead of an error. |
| Each call of the admin API that a session can make ([the admin API](/reference/admin-api/index.html.md#admin-sessions)), the reads too | An instance admin operates the whole install. A session lives for days in a browser, so the Admin area of the console works only for 10 minutes after each sign-in. |
| `ListAuditEntries` with `whole_install`, or for an org in which the instance admin is not an owner or an admin ([the sign-in, account, and audit API](/reference/auth-api/index.html.md#listauditentries)) | An instance admin reads these audit logs as in the Admin area. A CLI token of an instance admin reads them without this rule. An org owner or org admin reads the log of its own org without it. |

The rule applies only to a console session, because only a session has a sign-in in a browser. Such a call gives these results:

| Caller | Result |
| --- | --- |
| A session whose sign-in is at most 10 minutes old | The call goes on. |
| A session whose sign-in is more than 10 minutes old | `permission_denied` with the message `permission denied: this action needs a sign-in within the last 10 minutes. Sign in again, then retry`, and the detail `hopbox.v1.RecentSignInRequired` with `max_age` 10 minutes ([the sign-in, account, and audit API](/reference/auth-api/index.html.md#recentsigninrequired)). |
| Any other caller, such as a CLI token, an API key, or local root | The check of the sign-in passes it, and the policy of the call decides. For example, `ApproveDevice` takes only a session. So a CLI token or a key gets the policy refusal `permission denied: a caller of kind <kind> may not do auth.approve_device`, without the detail. An anonymous caller gets `unauthenticated`. |

Only the refusal of an old session has the detail. So a client knows from the detail that a new sign-in fixes the error. The other calls of an old session keep working. On the admin API, the session of an account that is not an instance admin gets the refusal of the policy without the detail. A new sign-in does not change that refusal.

To get a recent sign-in, the person signs in again in the same browser. The new sign-in ends the old session of that browser and starts a new one ([What a sign-in does](#what-a-sign-in-does)). The sessions of the account in other browsers stay. The console shows the sign-in page for this with `/signin?again=1` ([Console](/reference/console/index.html.md#sign-in)).

The policy check enforces the rule for the admin API and for `ListAuditEntries` ([the API server](/reference/api/index.html.md#callers-and-policy)). Their rules allow the session of an instance admin only with a recent sign-in. A rule can refuse a session with an older sign-in, and allow the same session with a recent one. In that case, `policy.Authorize` returns the error of this section. `ApproveDevice`, `RequestMagicLink` with `link`, and the start of a link with GitHub or Google call `service.RequireRecentSignIn` after their policy check. Both checks refuse only a session with an older sign-in, so an admin key and local root keep the calls that their policy allows.

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

A link that adds an email to an account has another subject and text ([Link a new sign-in method](#link-a-new-sign-in-method)):

```text
From: hopbox <no-reply@example.org>
To: ann.work@example.org
Subject: Add your email to your account on example.com

Open this link to add ann.work@example.org as a sign-in method of your account on example.com:

https://example.com/auth/email#token=<token>

Open it in the browser where you asked for it. The link works once, for 15 minutes. If you did not ask for it, you can ignore this email.
```

### Rate limits

`RequestMagicLink` counts the links of the last hour in the control store, so the limits hold across every control server.

| Limit | Value | Error |
| --- | --- | --- |
| Links for one email | 5 per hour | `resource_exhausted`: `too many requests: this email got the most sign-in links of an hour. Use one of them, or wait` |
| Links from one client | 20 per hour | `resource_exhausted`: `too many requests: this client asked for the most sign-in links of an hour. Wait, then retry` |

A client is one IPv4 address, or one IPv6 /64, because one IPv6 host often holds a whole /64. An IPv4 address in IPv6 form counts as its IPv4 address. The address is the peer of the request. Behind the edge, the peer is the address of the client that the edge forwards. Only a server with the edge role can name it ([the internal listener](/reference/internal-listener/index.html.md#forwarded-requests)). The limit of a client skips a loopback peer, such as `127.0.0.1` or `::1`. A loopback peer is a process or an SSH tunnel on the same server, on the API listener. It stands for every client, so a limit on it blocks everyone. Each request takes a transaction lock (`LockKeyMagicLinks`, [the bundled Postgres](/reference/postgres/index.html.md#advisory-lock-keys)), so requests at the same time cannot pass a limit together. A request whose email fails to send still counts. A link of an email to an account counts as a magic link ([Link a new sign-in method](#link-a-new-sign-in-method)). The messages never say how many links an email got, so they tell nothing about the activity of others.

## First-admin login links

`sudo hopboxd admin login-link --email <email>` prints a login link ([hopboxd](/reference/hopboxd/index.html.md#admin-login-link)). The link is for the first instance admin of a new install, which has no SMTP config and no OAuth app yet.

```text
https://example.com/auth/login-link#token=<token>
```

The console page `/auth/login-link` reads the token and calls `VerifyLoginLink`. The link works once, for 15 minutes. It needs the domain of the install.

## What a sign-in does

Each sign-in proves an identity: a method and an email. It then reaches one account:

1. If the account of an earlier sign-in holds the method, the sign-in reaches that account. A magic link names its method by the email. A login link names its method by the email too, apart from magic links. GitHub and Google name their method by the subject at the provider, so a new email at the provider reaches the same account. If a delete of that account runs at the same time, the sign-in waits for the delete and then fails with `not_found`: `not found: the account of this sign-in method was deleted during the sign-in. Sign in again to make a new account`. A magic link or a login link stays unused, so the person opens it again, and it creates a new account. GitHub and Google use their flow before this check, so the callback answers `404` ([Sign-in with GitHub and Google](/reference/oauth/index.html.md#errors)), and the person starts the sign-in again ([Control store tables](/reference/control-store/index.html.md#orgs)).
2. Else, the method can prove the email, and an account can hold that email as verified. If that account has no identity of a provider, the sign-in links the method to it. An identity of a provider is a method of GitHub or Google. A magic link proves its email. GitHub and Google prove the email that the provider verified, and nothing else passes ([Sign-in with GitHub and Google](/reference/oauth/index.html.md#providers)). A login link proves no email.

   If the account has an identity of a provider, the sign-in fails and changes nothing. Then only a link from a session adds a method to the account ([Link a new sign-in method](#link-a-new-sign-in-method)). The error is `failed_precondition` from `VerifyMagicLink`, or `409` at the callback of [Sign-in with GitHub and Google](/reference/oauth/index.html.md#errors), with this message:

   ```text
   the sign-in conflicts with a sign-in method of the account: the account of dana@example.org signs in with GitHub or Google, so a new sign-in method joins it only through a link. Sign in with a method of the account, then link Google on the account page. If you lost every method of the account, the operator removes the lost one with hopboxd admin account sign-in-method remove
   ```

   The rule protects an account against a lapsed domain. The new owner of the domain can prove an old address of the account, with a magic link or at a provider. The email of a method follows the provider, so a rule by email misses an old address. So once the account has an identity at a provider, the email alone links nothing. A person who lost every method of the account gets help from the operator ([Sign-in with GitHub and Google](/reference/oauth/index.html.md#recover-a-lost-identity)).
3. Else, the sign-in creates an account with the email, the method, and a personal org named `Personal`, with the account as its owner. The sign-up mode gives the state and the tier of the new account ([Sign-up modes](#sign-up-modes)). The limits of new accounts apply first ([Limits of new accounts](#limits-of-new-accounts)). A magic link, GitHub, and Google mark the email verified. A login link leaves it unverified. GitHub and Google also give the account the name of the person at the provider as its display name. The display name stays empty for a name that breaks the rules of [Domain rules](/reference/domain-rules/index.html.md#display-names). At GitHub, an empty name falls back to the login.

So the email of a login link never links to another account. A magic link, GitHub, or Google with the same email reaches another account. A second login link for the same email reaches the account of the first one. A verified email belongs to at most one account.

A sign-in also does these steps:

- A login link makes the account an instance admin. An account that is one already stays one.
- A sign-in from a browser that has a session ends that session. The new one replaces it.
- It starts a session and records the sign-in in the audit log ([the sign-in, account, and audit API](/reference/auth-api/index.html.md#audit-entries)). A link of a method to an account by its verified email gets its own entry.

A sign-in that reaches an account that exists reads the state of the account first. An instance admin can suspend an account ([the admin API](/reference/admin-api/index.html.md#suspendaccount)). The sign-in of a suspended account fails, and no session starts. A login link of a suspended account fails too. The error is `permission_denied` from `VerifyMagicLink` and `VerifyLoginLink`, or `403` at the callback of [Sign-in with GitHub and Google](/reference/oauth/index.html.md#errors), with this message:

```text
permission denied: an instance admin suspended this account, so it cannot sign in. The reason: abuse report 1
```

The transaction of the sign-in rolls back, so a magic link or a login link stays unused. The console page of the link shows the message, with its first letter in upper case and the request ID, and a link back to the sign-in page ([Console](/reference/console/index.html.md#pages)). It opens no console. The log record is `sign-in refused` at level info, with `component=service`, `kind`, `account`, and `reason` (`the account is suspended`). The suspend deletes every session of the account. While the account is suspended, its session cookie names no live session, so the request gate treats the cookie as no credential ([the API server](/reference/api/index.html.md#the-contract-of-a-resolver)).

A refused link writes the log record `sign-in link refused` at level info, with `component=service` and `kind`, never the token. A refusal for a reason other than the token has the field `reason` too: `another session asked for the link`, or `the account needs a link for a new sign-in method`.

Two first sign-ins with one email at the same time make one account: each sign-in takes a transaction lock (`LockKeySignIn`).

### Sign-up modes

The install setting `sign_up_mode` gives the state and the tier of the account that a first sign-in of a new person creates. An instance admin changes it with `UpdateInstallConfig` ([the admin API](/reference/admin-api/index.html.md#updateinstallconfig)) or with `hopboxd admin config set --sign-up-mode` ([hopboxd](/reference/hopboxd/index.html.md#admin-config-set)).

| Mode | The new account |
| --- | --- |
| `approval` | The state `pending` and the tier `limited`, with its personal org. An instance admin approves or rejects it ([Pending accounts](#pending-accounts)). This is the default of a new install. Migration `0062` also sets it on an install that exists. |
| `open` | The state `active` and the tier `limited`. No limit depends on the tier yet. |

These sign-ins ignore the mode and the limits of new accounts:

- A first-admin login link. Its new account is active, with the tier `full`, and it becomes an instance admin.
- A sign-in that reaches an account that exists, and the link of a new sign-in method to an account ([Link a new sign-in method](#link-a-new-sign-in-method)).

Each other first sign-in writes a row in `account_sign_ups` ([Control store tables](/reference/control-store/index.html.md#account_sign_ups)). The row keeps the method and the client address. For GitHub, it also keeps the login and the creation time of the GitHub user ([Sign-in with GitHub and Google](/reference/oauth/index.html.md#providers)). The client address is the address that asked for the magic link, or that started the flow of GitHub or Google. It is not the address that opens the link. The audit entry `account.create` has the details `state` and `client_ip` too, and `login` for GitHub ([the sign-in, account, and audit API](/reference/auth-api/index.html.md#audit-entries)).

A `hopboxd` of the schema version 57 to 61 does not know the mode. In each mode, its first sign-in makes an active account of the tier `full`, and writes no row in `account_sign_ups`.

### Limits of new accounts

A first sign-in of a new person runs these checks in this order, in the transaction of the sign-in, under `LockKeySignIn`. A refusal rolls the transaction back and makes no account. So a magic link stays unused. A flow of GitHub or Google is used, and the person starts it again.

| Check | Message | Code |
| --- | --- | --- |
| An entry of the block list matches the email, the domain of the email or one of its parent domains, or the identity at GitHub or Google ([Domain rules](/reference/domain-rules/index.html.md#block-list-entries)). A blocked identity at GitHub or Google also joins no account that exists through a shared verified email. That sign-in fails with the same error. | `permission denied: this install takes no new account for this sign-in` | `permission_denied`, or `403` at the callback of [Sign-in with GitHub and Google](/reference/oauth/index.html.md#errors) |
| The client address made 3 accounts in the last 24 hours. | `too many requests: this client address made the most new accounts of a day. Retry tomorrow` | `resource_exhausted`, or `429` at the callback |
| In the mode `approval` only: the install holds 50 pending accounts. | `too many requests: this install takes no new accounts now. Retry later` | `resource_exhausted`, or `429` at the callback |
| In the mode `approval` only: a live control process has a schema version below 62 (`service.SignUpSchema`). Such a release lets a pending account act. | `unavailable: the install upgrades now, so it takes no new accounts. Retry later` | `unavailable`, or `503` with `Retry-After: 30` at the callback |

The limit of a client counts the rows of `account_sign_ups` of the last 24 hours. A client is one IPv4 address, or one IPv6 /64, as for the rate limits of links ([Rate limits](#rate-limits)). A loopback address and an empty address have no limit. The rows of a rejected or a deleted account still count. The leader deletes such a row one day after the sign-up ([hopboxd](/reference/hopboxd/index.html.md#leader-jobs)). A login link writes no row, so it does not count.

Each refusal writes the log record `sign-up refused` with `component=service`, `kind`, and `reason`. The first three checks log at level info, with `client_ip`. Their reasons are `an entry of the block list matches`, `the client made the most new accounts of a day`, and `the install holds the most pending accounts`. These three also count in `hopbox_account_creations_total` with `state="refused"` after the transaction of the sign-in ends ([Metrics](/reference/metrics/index.html.md#metrics-of-the-accounts)). The refusals by the limit of a client and by the cap of pending accounts also count in `hopbox_rate_limited_total`, with the limits `account_creations` and `pending_cap`. A refusal by the cap of pending accounts also gives the instance admins a notice, at most one every hour ([the admin API](/reference/admin-api/index.html.md#notices-of-new-accounts)). The check of the schema logs at level warn, with the reason `a control process of an older release is live`, `control_process`, and `schema_version`. It does not count in the metric.

### Pending accounts

A pending account waits for approval by an instance admin. It signs in, and its session lives as any other session. The request gate reads the state of the account with the session on each call, as it reads the instance admin role. The policy check allows the session of a pending account only these calls:

- `GetAccount`, which returns `pending` and the note ([the sign-in, account, and audit API](/reference/auth-api/index.html.md#getaccount))
- `SetSignUpNote`, which writes one note of at most 500 characters to the instance admins ([the sign-in, account, and audit API](/reference/auth-api/index.html.md#setsignupnote))
- `DeleteAccount`, so the person deletes the account ([the sign-in, account, and audit API](/reference/auth-api/index.html.md#deleteaccount))
- `SignOut`
- the calls that a caller without a credential can make, such as `GetInstallInfo`

Each other call fails with `permission_denied`, the message `permission denied: the account waits for approval by an instance admin`, and the detail `hopbox.v1.AccountPending` ([the sign-in, account, and audit API](/reference/auth-api/index.html.md#accountpending)). The console reads the detail and shows the page of a pending account.

The other ways of a pending account in are closed too:

| What | Result |
| --- | --- |
| The poll of a device sign-in that the person approved | It fails with the same error, and uses up the device code. |
| A CLI access token or a member API key | It works only for an active account, so it counts as revoked ([CLI tokens, API keys, admin API keys, and SSH keys](/reference/credentials/index.html.md#how-the-request-gate-reads-them)). |
| A role in an org | Only an active account acts in an org (`GetActingRole`), so a pending account acts in none. |
| A change that locks the account of its caller, such as `CreateOrg` | It fails with the same error. |
| An SSH key at the SSH front door | It opens nothing. The reason is `the account of the key waits for approval` ([the SSH front door](/reference/ssh-front-door/index.html.md#who-opens-a-machine)). |
| `GrantInstanceAdmin` of a pending account | `failed_precondition` with the message `failed precondition: the account waits for approval. Approve it first` ([the admin API](/reference/admin-api/index.html.md#grantinstanceadmin)). |

An instance admin approves the account with `ApproveAccount`, or rejects it with `RejectAccount` ([the admin API](/reference/admin-api/index.html.md#approveaccount)). The leader deletes a pending account that nobody approves after 30 days ([hopboxd](/reference/hopboxd/index.html.md#leader-jobs)).

## Link a new sign-in method

A person who is signed in adds a new sign-in method to the account. Once the account has an identity of GitHub or Google, this is the only way in for a new method. The console offers it on `/account` ([Console](/reference/console/index.html.md#pages)). Each link needs a session with a recent sign-in ([Recent sign-in](#recent-sign-in)).

| Method | Start | End |
| --- | --- | --- |
| Email link | `RequestMagicLink` with `link` and the email ([the sign-in, account, and audit API](/reference/auth-api/index.html.md#requestmagiclink)). The email gets a link. | The person opens the link in the same browser. `VerifyMagicLink` with the session of the request adds the email method. |
| GitHub or Google | The browser opens `/auth/<provider>/start?link=1` ([Sign-in with GitHub and Google](/reference/oauth/index.html.md#link-to-a-signed-in-account)). | The callback of the provider comes back with the same session, and adds the identity. |

A link needs a recent sign-in when it starts. From then on, it binds to its session for its lifetime, so it ends also after those 10 minutes. A link binds to the session that started it. The link works only with the session cookie of that session, so the same browser must open it. Another browser, another session, or a browser without a session gets `permission_denied` from `VerifyMagicLink`, and the link stays unused. At the callback of a provider, another caller gets `400`. A sign-out ends the session, so its links never work after it. A link that ends after its session ended fails with `unauthenticated`: `this call needs a credential: the session that started the link ended. Sign in again, then link the method again`, or `403` at the callback. A magic link then stays unused. So a person cannot add a method to the account of another person, also with the mailbox of that person.

A link starts no session and sets no cookie. The session of the browser stays. A link does these steps in one transaction:

1. It takes the transaction lock `LockKeySignIn`, then `LockKeyAdminNotices` ([the bundled Postgres](/reference/postgres/index.html.md#order-of-the-advisory-locks)), and checks that the session is still live. A share lock on the session row keeps a sign-out from ending it before the link commits.
2. It refuses a method that signs in to another account: `failed_precondition`, or `409` at the callback. The message names the method, for example `this Google identity signs in to another account. Remove it from that account first`.
3. It refuses a method that the account has: `already_exists`, or `409` at the callback. `RequestMagicLink` refuses an email method of the account before it sends the mail.
4. It refuses an email that another account holds as verified: `failed_precondition` with `the email kim@example.org is a verified email of another account, so it signs in only to that account`. The method of an email decides where a magic link of that email signs in. So without this step, the holder of the mailbox could move the magic links of another account to the own account.
5. It refuses a second identity of the same provider: `the account already signs in with another GitHub identity. Remove that method first, then link the new one`.
6. It adds the method. The email of the method becomes a verified email of the account. For an identity of a provider, this step skips an email that another account holds as verified. An unverified email of the account, such as the email of a login link, becomes verified.
7. It writes the audit entry `account.link_sign_in_method` ([the sign-in, account, and audit API](/reference/auth-api/index.html.md#audit-entries)). If the account is an instance admin, the entry is a notice to the instance admins, and hopbox sends its email after the commit ([the admin API](/reference/admin-api/index.html.md#notices-to-instance-admins)).

A link of GitHub or Google needs no match between its email and the emails of the account. The session proves the account, and the provider proves the identity. So a login-link account, whose email is unverified, can link GitHub or Google (plan question Q23).

## Sweep

Every 10 minutes, the leader deletes the ended sessions. It also deletes the magic links, the login links, and the flows of GitHub and Google that are older than 24 hours ([hopboxd](/reference/hopboxd/index.html.md#leader-jobs)). It deletes the codes of machine URLs that expired before their trade too. The delete of a session also deletes the tokens of the machine URLs that it made ([Control store tables](/reference/control-store/index.html.md#machine_url_sessions)). A link stays after its use for the rate limits. The same sweep deletes the ended CLI sign-ins and the old device sign-ins ([CLI tokens, API keys, admin API keys, and SSH keys](/reference/credentials/index.html.md#sweep)).
