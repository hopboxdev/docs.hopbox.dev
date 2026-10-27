# The sign-in, account, and audit API

> The sign-in, account, audit, API key, and SSH key calls of the public API: AuthService with the device flow, AccountService, AuditService, APIKeyService, and SSHKeyService, with their audit entries.

These services of the public API `hopbox.v1` sign people in to the console and the hopbox CLI. They also manage the account of the caller with its API keys and SSH keys, and read the audit log. [The API server](/reference/api/index.html.md) covers the parts that every call shares: the request gate, the policy check, and the error codes. [Sessions, cookies, and sign-in links](/reference/sessions/index.html.md) covers the session cookie, the check against cross-site requests, the links, and the rate limits. [Sign-in with GitHub and Google](/reference/oauth/index.html.md) covers the sign-in with GitHub and Google, which runs on browser endpoints, not on calls of this API. [CLI tokens, API keys, admin API keys, and SSH keys](/reference/credentials/index.html.md) covers the device flow, CLI tokens, API keys, and SSH keys. The generated reference is [Public API](/api/hopbox.v1.openapi.yaml).

| Service | Calls |
| --- | --- |
| `hopbox.v1.AuthService` | `RequestMagicLink`, `VerifyMagicLink`, `VerifyLoginLink`, `SignOut`, `StartDeviceAuthorization`, `GetDeviceAuthorization`, `ApproveDevice`, `PollDeviceAuthorization`, `RefreshCLIToken` |
| `hopbox.v1.AccountService` | `GetAccount`, `UpdateAccount`, `DeleteAccount`, `ListSignInMethods`, `RemoveSignInMethod`, `ListCLISignIns`, `EndCLISignIn` |
| `hopbox.v1.AuditService` | `ListAuditEntries` |
| `hopbox.v1.APIKeyService` | `CreateAPIKey`, `ListAPIKeys`, `RevokeAPIKey` |
| `hopbox.v1.SSHKeyService` | `AddSSHKey`, `ListSSHKeys`, `RemoveSSHKey` |

A call with the session cookie must carry the header `Origin` with the origin of the install, and the header `Connect-Protocol-Version: 1` ([Sessions, cookies, and sign-in links](/reference/sessions/index.html.md#calls-with-the-cookie)). The examples on this page use curl against a laptop install with the domain `example.com`, as in [First sign-in on a new install](/guide/first-sign-in/index.html.md). Each example of a call with the cookie uses these shell variables:

```sh
API=http://127.0.0.1:7080
COOKIE='__Host-hopbox_session=<token>'
browser() {
  curl -s -X POST -H 'Content-Type: application/json' -H 'Connect-Protocol-Version: 1' \
    -H 'Origin: https://example.com' -H "Cookie: $COOKIE" "$@"
}
```

## Messages

### Account

| Field | Type | Description |
| --- | --- | --- |
| `id` | string | The UUID of the account. |
| `display_name` | string | The name that the person chose. It can be empty. |
| `emails` | repeated `AccountEmail` | The emails of the account. |
| `instance_admin` | bool | `true` for an instance admin. |
| `personal_org_id` | string | The UUID of the personal org of the account. |
| `version` | int64 | The version of the account. It starts at 1 and grows by 1 with each change. |
| `create_time` | `google.protobuf.Timestamp` | The time of the first sign-in. |

### AccountEmail

| Field | Type | Description |
| --- | --- | --- |
| `email` | string | The address, in lower case. |
| `verified` | bool | `true` for an email that a sign-in method proved. Only a verified email links other sign-in methods to the account, and only while the account has no identity of GitHub or Google ([Sessions, cookies, and sign-in links](/reference/sessions/index.html.md#what-a-sign-in-does)). The email of a login link stays unverified, until a link from a session proves it ([Sessions, cookies, and sign-in links](/reference/sessions/index.html.md#link-a-new-sign-in-method)). |
| `verify_time` | `google.protobuf.Timestamp` | The time of the proof. It is unset for an email that is not verified. |

### SignInMethod

The enum `SignInMethod` names a kind of sign-in. `GetInstallInfo` lists the kinds that anyone can use ([the API server](/reference/api/index.html.md#getinstallinfo)).

| Value | Meaning | In `GetInstallInfo` |
| --- | --- | --- |
| `SIGN_IN_METHOD_UNSPECIFIED` | Never in a response. | never |
| `SIGN_IN_METHOD_EMAIL_LINK` | A magic link that hopbox sends by email. | while the install has an SMTP config or the file mail transport ([Mail](/reference/mail/index.html.md#mail-transports)) |
| `SIGN_IN_METHOD_LOGIN_LINK` | A first-admin login link that root prints. | never, because only root can make one |
| `SIGN_IN_METHOD_GITHUB` | A sign-in with the GitHub OAuth app of the install ([Sign-in with GitHub and Google](/reference/oauth/index.html.md)). A browser starts it at `/auth/github/start`. | while the install has a domain and a GitHub client |
| `SIGN_IN_METHOD_GOOGLE` | A sign-in with the Google OAuth client of the install, through OpenID Connect. A browser starts it at `/auth/google/start`. | while the install has a domain and a Google client |

### LinkedSignInMethod

| Field | Type | Description |
| --- | --- | --- |
| `id` | string | The UUID of the method on this account. |
| `method` | `SignInMethod` | The kind of sign-in. |
| `email` | string | The email that the method names. |
| `last_use_time` | `google.protobuf.Timestamp` | The time of the last sign-in with the method. |
| `create_time` | `google.protobuf.Timestamp` | The time of the link of the method to the account. |

### DeviceAuthorizationState

The enum `DeviceAuthorizationState` is the answer of `PollDeviceAuthorization`.

| Value | Meaning |
| --- | --- |
| `DEVICE_AUTHORIZATION_STATE_UNSPECIFIED` | Never in a response. |
| `DEVICE_AUTHORIZATION_STATE_PENDING` | Nobody approved the code yet. Poll again after `interval_seconds`. |
| `DEVICE_AUTHORIZATION_STATE_SLOW_DOWN` | The poll came less than the interval after the poll before. The interval grew by 5 s, and holds for every later poll. |
| `DEVICE_AUTHORIZATION_STATE_APPROVED` | A person approved the code. The response holds the CLI tokens. |

### CLITokens

The tokens of a CLI sign-in ([CLI tokens, API keys, admin API keys, and SSH keys](/reference/credentials/index.html.md#cli-tokens)). The install keeps only their hashes.

| Field | Type | Description |
| --- | --- | --- |
| `access_token` | string | Starts with `hbxat_`. It goes in the header `Authorization: Bearer <token>`. |
| `access_token_expire_time` | `google.protobuf.Timestamp` | The end of the access token, 1 hour after its issue. |
| `refresh_token` | string | Starts with `hbxrt_`. It works once, with `RefreshCLIToken`. |
| `refresh_token_expire_time` | `google.protobuf.Timestamp` | The end of the refresh token, 30 days after its issue. |
| `account_id` | string | The UUID of the account of the tokens. |
| `cli_sign_in_id` | string | The UUID of the CLI sign-in. Every refresh keeps it. |

### RecentSignInRequired

The detail of the `permission_denied` error of a call that needs a recent sign-in, such as `ApproveDevice` ([Sessions, cookies, and sign-in links](/reference/sessions/index.html.md#recent-sign-in)). A client reads it without parsing the message. The console then sends the person to sign in again.

| Field | Type | Description |
| --- | --- | --- |
| `max_age` | `google.protobuf.Duration` | How long a session counts as recent after its sign-in: 10 minutes. |

### APIKey

An API key without its secret ([CLI tokens, API keys, admin API keys, and SSH keys](/reference/credentials/index.html.md#api-keys)).

| Field | Type | Description |
| --- | --- | --- |
| `id` | string | The UUID of the key. |
| `org_id` | string | The UUID of the org of the key. |
| `kind` | `APIKeyKind` | `API_KEY_KIND_MEMBER` or `API_KEY_KIND_ORG`. |
| `name` | string | The name that its creator gave the key, such as `ci`. |
| `account_id` | string | The UUID of the account of a member key. Empty for an org key. |
| `creator_account_id` | string | The UUID of the account that made the key. Empty after the delete of that account. |
| `expire_time` | `google.protobuf.Timestamp` | The end of the key. |
| `last_use_time` | `google.protobuf.Timestamp` | The last call with the key, exact to a minute. Unset for a key without a call. |
| `create_time` | `google.protobuf.Timestamp` | The creation of the key. |

The enum `APIKeyKind` has the values `API_KEY_KIND_UNSPECIFIED`, which a request takes as a member key, `API_KEY_KIND_MEMBER`, and `API_KEY_KIND_ORG`. The enum `APIKeyLifetime` has the values `API_KEY_LIFETIME_UNSPECIFIED` (90 days), `API_KEY_LIFETIME_30_DAYS`, `API_KEY_LIFETIME_90_DAYS`, and `API_KEY_LIFETIME_365_DAYS`.

### SSHKey

A public key of an account ([CLI tokens, API keys, admin API keys, and SSH keys](/reference/credentials/index.html.md#ssh-keys)).

| Field | Type | Description |
| --- | --- | --- |
| `id` | string | The UUID of the key. |
| `name` | string | The name of the key, such as `ann@laptop`. It can be empty. |
| `type` | string | The key type, such as `ssh-ed25519`. |
| `bits` | int32 | The size of the key: 256 for Ed25519, the curve size for ECDSA, the modulus size for RSA. |
| `public_key` | string | The key in the `authorized_keys` form, without its comment. |
| `fingerprint` | string | The SHA-256 fingerprint, as `ssh-keygen -l` prints it. |
| `create_time` | `google.protobuf.Timestamp` | The addition of the key. |

## AuthService

### RequestMagicLink

Sends a sign-in link to an email ([Sessions, cookies, and sign-in links](/reference/sessions/index.html.md#magic-links)). It answers the same for an email with or without an account, so the call tells nothing about who has an account. With `link`, it sends a link that adds the email as a sign-in method of the account of the session instead ([Sessions, cookies, and sign-in links](/reference/sessions/index.html.md#link-a-new-sign-in-method)).

| Item | Value |
| --- | --- |
| Procedure | `/hopbox.v1.AuthService/RequestMagicLink` |
| Policy action | `auth.request_magic_link`: `anonymous` and `session`. With `link`: `account.link_sign_in_method`, the `session` of the account, with a sign-in within the last 10 minutes ([Sessions, cookies, and sign-in links](/reference/sessions/index.html.md#recent-sign-in)). |
| Idempotent option | no. Each call sends a new link. |
| Audit entry | action `auth.request_magic_link`, actor the caller, target kind `magic_link`, target ID the UUID of the link, details `email` and `client_ip`. With `link`, also `purpose` with the value `link`. |

The request `RequestMagicLinkRequest` has these fields. The response `RequestMagicLinkResponse` has no fields.

| Field | Type | Description |
| --- | --- | --- |
| `email` | string | The address that gets the link, such as `ann@example.org`. The server takes the email in lower case, without spaces around it ([Domain rules](/reference/domain-rules/index.html.md#emails)). |
| `link` | bool | `true` for a link that adds the email to the account of the session. Only that session can use the link. The email has the subject `Add your email to your account on <domain>`. |

| Error | Code |
| --- | --- |
| The email breaks a rule, such as `Ann <ann@example.org>` | `invalid_argument` |
| `link` without a session: `this call needs a credential` | `unauthenticated` |
| `link` with a caller that is not a session, such as a CLI token | `permission_denied` |
| `link` with an email that the account signs in with already. The call sends no mail. | `already_exists` |
| `link` with a session whose sign-in is more than 10 minutes old: `permission denied: this action needs a sign-in within the last 10 minutes. Sign in again, then retry`, with the detail `RecentSignInRequired` | `permission_denied` |
| The install has no SMTP config and the transport `smtp`: `failed precondition: this install has no SMTP config, so it sends no sign-in links` | `failed_precondition` |
| The install has no domain | `failed_precondition` |
| The SMTP password does not open, for example after the loss of its key-encryption key | `internal` |
| A rate limit ([Sessions, cookies, and sign-in links](/reference/sessions/index.html.md#rate-limits)) | `resource_exhausted` |
| The SMTP server failed, or the file transport failed to write the file: `unavailable: the server failed to send the email. Retry later` | `unavailable` |

The server logs the cause of a failed email as `the magic link email failed`, at level error, with `component=service` and `err`. With the file mail transport, the call writes the email to a file on the server and sends nothing ([Mail](/reference/mail/index.html.md#the-file-transport)). A rate limit applies first ([Sessions, cookies, and sign-in links](/reference/sessions/index.html.md#rate-limits)).

```sh
curl -s -X POST -H 'Content-Type: application/json' -d '{"email":"ann@example.org"}' \
  $API/hopbox.v1.AuthService/RequestMagicLink
```

```json
{}
```

A link of an email to the account of the session:

```sh
browser -d '{"email":"ann.work@example.org","link":true}' $API/hopbox.v1.AuthService/RequestMagicLink
```

### VerifyMagicLink

Signs in with the token of a magic link, and sets the session cookie ([Sessions, cookies, and sign-in links](/reference/sessions/index.html.md#what-a-sign-in-does)). The link works once, within 15 minutes. The token of a link from `RequestMagicLink` with `link` adds the email to the account of the session that asked for it, and sets no cookie ([Sessions, cookies, and sign-in links](/reference/sessions/index.html.md#link-a-new-sign-in-method)).

| Item | Value |
| --- | --- |
| Procedure | `/hopbox.v1.AuthService/VerifyMagicLink` |
| Policy action | `auth.verify_magic_link`: `anonymous` and `session` |
| Idempotent option | no |
| Audit entries | `account.create` for a new account, then `auth.sign_in` ([Audit entries](#audit-entries)). A link writes only `account.link_sign_in_method`. |

The request `VerifyMagicLinkRequest` has the field `token`, a string: the value after `#token=` in the link. The response `VerifyMagicLinkResponse` has these fields, and the header `Set-Cookie` with the session cookie. A link has no `Set-Cookie`.

| Field | Type | Description |
| --- | --- | --- |
| `account` | `Account` | The account of the new session, or the account that the link added the email to. |
| `new_account` | bool | `true` for an account that this sign-in created. |
| `session_expire_time` | `google.protobuf.Timestamp` | The end of the session, 30 days after the sign-in. After 7 days without a call, the session ends before that. Unset for a link. |
| `linked` | bool | `true` when the link added its email to the account of the session. No new session starts, and the session cookie stays. |

| Error | Code |
| --- | --- |
| The call does not carry the `Origin` of the install and the Connect protocol header. The link stays unused. | `permission_denied` |
| The token is unknown, used, or expired: `the sign-in link is not valid: it is unknown, used, or expired. Ask for a new link` | `unauthenticated` |
| The account of the email has GitHub or Google, and no email method for this email ([Sessions, cookies, and sign-in links](/reference/sessions/index.html.md#what-a-sign-in-does)): `the sign-in conflicts with a sign-in method of the account: the account of ann@example.org signs in with GitHub or Google, so a new sign-in method joins it only through a link. ...` | `failed_precondition` |
| A link, and the caller is not the session that asked for it. The link stays unused: `permission denied: this link adds an email to the account of the session that asked for it. Open it in the browser where you asked for it, while you are signed in there` | `permission_denied` |
| A link, and another account signs in with the email: `the sign-in conflicts with a sign-in method of the account: this email ann.work@example.org signs in to another account. Remove it from that account first` | `failed_precondition` |
| A link, and another account holds the email as verified: `the sign-in conflicts with a sign-in method of the account: the email kim@example.org is a verified email of another account, so it signs in only to that account` | `failed_precondition` |
| A link, and the account has the email method already | `already_exists` |
| A link, and the session that asked for it ended: `this call needs a credential: the session that started the link ended. Sign in again, then link the method again` | `unauthenticated` |

### VerifyLoginLink

Signs in with the token of a first-admin login link, and sets the session cookie. The link works once, within 15 minutes. It reaches the account of earlier login links for the same email, or a new account, and makes that account an instance admin. The email stays unverified ([Sessions, cookies, and sign-in links](/reference/sessions/index.html.md#what-a-sign-in-does)).

| Item | Value |
| --- | --- |
| Procedure | `/hopbox.v1.AuthService/VerifyLoginLink` |
| Policy action | `auth.verify_login_link`: `anonymous` and `session` |
| Idempotent option | no |
| Audit entries | `account.create` for a new account, `auth.sign_in`, and `instance_admin.grant` for an account that was not an instance admin yet |

The request `VerifyLoginLinkRequest` has the field `token`. The response `VerifyLoginLinkResponse` has the fields of `VerifyMagicLinkResponse`, and the header `Set-Cookie` with the session cookie. The errors are those of `VerifyMagicLink`.

```sh
curl -s -i -X POST -H 'Content-Type: application/json' -H 'Connect-Protocol-Version: 1' \
  -H 'Origin: https://example.com' -d '{"token":"<token>"}' \
  $API/hopbox.v1.AuthService/VerifyLoginLink
```

```text
HTTP/1.1 200 OK
Content-Type: application/json
Hopbox-Request-Id: 01a143f9-1cee-7170-bd72-5ceae51d8bc8
Set-Cookie: __Host-hopbox_session=6s6pHx8zxuTK_v6pj2KCmDxYvkEWEgE5hqOBhfvXs_c; Path=/; Expires=Sun, 15 Nov 2026 09:09:21 GMT; Max-Age=2591999; HttpOnly; Secure; SameSite=Lax

{"account":{"id":"01a143f9-1cee-7ec0-9238-7976abaacfd6", "emails":[{"email":"root@example.org"}], "instanceAdmin":true, "personalOrgId":"01a143f9-1cee-7ec1-a648-71fea47ae54f", "version":"1", "createTime":"2026-10-16T09:09:21.774252Z"}, "newAccount":true, "sessionExpireTime":"2026-11-15T09:09:21.774252Z"}
```

In the JSON codec, a field with its default value is left out, so the unverified email has no `verified` field.

### SignOut

Ends the session of the caller and clears the session cookie.

| Item | Value |
| --- | --- |
| Procedure | `/hopbox.v1.AuthService/SignOut` |
| Policy action | `auth.sign_out`: `session` and `cli_token` |
| Idempotent option | no |
| Audit entry | action `auth.sign_out`, org the personal org, target kind `session` and the UUID of the session. With a CLI token, target kind `cli_sign_in` and the UUID of the CLI sign-in. |

The request `SignOutRequest` and the response `SignOutResponse` have no fields. The response has the header `Set-Cookie` with `Max-Age=0`. The sign-out of a session also deletes the tokens of the private machine URLs that the session made, and publishes the change event `edge.access_changed`. So each edge server checks the machine URLs of the browser again ([the edge role](/reference/edge/index.html.md#private-machine-urls)).

```sh
browser -d '{}' $API/hopbox.v1.AuthService/SignOut
```

### StartDeviceAuthorization

Starts the device flow of the hopbox CLI ([CLI tokens, API keys, admin API keys, and SSH keys](/reference/credentials/index.html.md#the-device-flow)). It needs no credential. The CLI shows the user code and the verification URI, then polls with the device code.

| Item | Value |
| --- | --- |
| Procedure | `/hopbox.v1.AuthService/StartDeviceAuthorization` |
| Policy action | `auth.start_device_authorization`: every caller, also `anonymous`. The device code is the credential of the flow. |
| Idempotent option | no. Each call starts a new device sign-in. |
| Audit entry | action `auth.start_device_authorization`, actor the caller, no org, target kind `device_authorization`, details `client_name` and `client_ip` |

The request `StartDeviceAuthorizationRequest` has the field `client_name`, a string such as `hopbox CLI on ann-laptop`. It has at most 64 characters, and can be empty ([Domain rules](/reference/domain-rules/index.html.md#credential-names)). The console shows it at the approval. The response `StartDeviceAuthorizationResponse` has these fields:

| Field | Type | Description |
| --- | --- | --- |
| `device_code` | string | The secret with which the CLI polls: 43 characters of base64url. The CLI must not show it. |
| `user_code` | string | The code that the person types in the console, such as `QJTL-PHNV`. |
| `verification_uri` | string | The console page that approves the code: `https://<domain>/device`. |
| `verification_uri_complete` | string | The same page with the code in it: `https://<domain>/device?code=<user_code>`. |
| `expire_time` | `google.protobuf.Timestamp` | The end of the device sign-in, 10 minutes after its start. |
| `interval_seconds` | int32 | The least time between two polls: 5. |

| Error | Code |
| --- | --- |
| The client name breaks a rule | `invalid_argument` |
| The install has no domain: `failed precondition: the install has no domain, and the console page that approves the sign-in needs one` | `failed_precondition` |
| A limit of starts ([CLI tokens, API keys, admin API keys, and SSH keys](/reference/credentials/index.html.md#limits-of-the-device-flow)) | `resource_exhausted` |

```sh
curl -s -X POST -H 'Content-Type: application/json' -d '{"clientName":"hopbox CLI on ann-laptop"}' \
  $API/hopbox.v1.AuthService/StartDeviceAuthorization
```

```json
{"deviceCode":"<device code>","userCode":"QJTL-PHNV","verificationUri":"https://example.com/device","verificationUriComplete":"https://example.com/device?code=QJTL-PHNV","expireTime":"2026-10-16T17:24:56.701147Z","intervalSeconds":5}
```

### GetDeviceAuthorization

Returns the client that waits for a user code. The console page `/device` calls it first and shows the client name, the IP address, and the start time to the person. Only then does the person approve. Without this step, a person who got a code from someone else can sign that party in to their own account (RFC 8628 section 5.4).

| Item | Value |
| --- | --- |
| Procedure | `/hopbox.v1.AuthService/GetDeviceAuthorization` |
| Policy action | `auth.get_device_authorization`: `session` only |
| Idempotent option | no |
| Audit entry | none |

The request `GetDeviceAuthorizationRequest` has the field `user_code`, a string. Case, dashes, and spaces do not matter. The response `GetDeviceAuthorizationResponse` has these fields:

| Field | Type | Description |
| --- | --- | --- |
| `client_name` | string | The name that the CLI gave at its start. |
| `client_ip` | string | The IP address that started the device sign-in, or empty. |
| `create_time` | `google.protobuf.Timestamp` | The start of the device sign-in. `ApproveDevice` needs it back. |
| `expire_time` | `google.protobuf.Timestamp` | The end of the device sign-in. |

The errors are those of `ApproveDevice`, apart from the refusal of an old sign-in. The lookup takes a session of any age. A wrong code counts against the limit of wrong codes of the account.

```sh
browser -d '{"userCode":"QJTL-PHNV"}' $API/hopbox.v1.AuthService/GetDeviceAuthorization
```

```json
{"clientName":"hopbox CLI on ann-laptop","clientIp":"127.0.0.1","createTime":"2026-10-16T17:14:56.701147Z","expireTime":"2026-10-16T17:24:56.701147Z"}
```

### ApproveDevice

Approves a user code for the account of the session. The console page `/device` calls it after `GetDeviceAuthorization`, with the start time that the lookup showed. The next poll of the CLI then gets CLI tokens of that account.

The session needs a recent sign-in: at most 10 minutes old ([Sessions, cookies, and sign-in links](/reference/sessions/index.html.md#recent-sign-in)). The approval makes a CLI sign-in whose refresh token lives 30 days and renews with each use. So without this rule, someone who takes over an old session can make a CLI sign-in that outlives the session.

| Item | Value |
| --- | --- |
| Procedure | `/hopbox.v1.AuthService/ApproveDevice` |
| Policy action | `auth.approve_device`: `session` only. A CLI token or a key never approves a device. |
| Idempotent option | no |
| Audit entry | action `auth.approve_device`, org the personal org, target kind `device_authorization`, details `client_name` and `client_ip` |

The request `ApproveDeviceRequest` has these fields:

| Field | Type | Description |
| --- | --- | --- |
| `user_code` | string | The code that the CLI shows. Case, dashes, and spaces do not matter. |
| `create_time` | `google.protobuf.Timestamp` | The `create_time` of `GetDeviceAuthorizationResponse` for the code. The approval fails unless it matches, so the console approves only a client that it showed. The time has microsecond precision, so the console passes it back unchanged and never rounds it. |

The response `ApproveDeviceResponse` has these fields:

| Field | Type | Description |
| --- | --- | --- |
| `client_name` | string | The name that the CLI gave at its start. |
| `client_ip` | string | The IP address that started the device sign-in, or empty. |
| `create_time` | `google.protobuf.Timestamp` | The start of the device sign-in. |

| Error | Code |
| --- | --- |
| The call carries no session cookie, or the cookie call fails the check against cross-site requests ([Sessions, cookies, and sign-in links](/reference/sessions/index.html.md#calls-with-the-cookie)) | `unauthenticated` |
| A CLI token or a key | `permission_denied` |
| The sign-in of the session is more than 10 minutes old: `permission denied: this action needs a sign-in within the last 10 minutes. Sign in again, then retry`. The error has the detail [`RecentSignInRequired`](#recentsigninrequired). It does not count as a wrong code, and the code keeps waiting. | `permission_denied` |
| No device sign-in waits for the code: it is wrong, approved already, or expired, also between the lookup and the approval. `not found: no device sign-in waits for this code. Check the code in the terminal, or start the sign-in again`. The wrong code counts against the limit of the account. | `not_found` |
| The `create_time` does not match the device sign-in of the code: `failed precondition: the start time does not match the device sign-in of this code. Look the code up again, check the client, then approve it`. It counts as a wrong code. | `failed_precondition` |
| The account entered 10 wrong codes in the last hour ([CLI tokens, API keys, admin API keys, and SSH keys](/reference/credentials/index.html.md#limits-of-the-device-flow)) | `resource_exhausted` |

```sh
browser -d '{"userCode":"QJTL-PHNV","createTime":"2026-10-16T17:14:56.701147Z"}' $API/hopbox.v1.AuthService/ApproveDevice
```

```json
{"clientName":"hopbox CLI on ann-laptop","clientIp":"127.0.0.1","createTime":"2026-10-16T17:14:56.701147Z"}
```

The same call with a session whose sign-in is more than 10 minutes old gets HTTP status 403:

```json
{"code":"permission_denied","message":"permission denied: this action needs a sign-in within the last 10 minutes. Sign in again, then retry","details":[{"type":"hopbox.v1.RecentSignInRequired","value":"CgMI2AQ","debug":{"maxAge":"600s"}}]}
```

### PollDeviceAuthorization

Asks whether a person approved the device code. The first poll after the approval starts a CLI sign-in and returns its tokens, once.

| Item | Value |
| --- | --- |
| Procedure | `/hopbox.v1.AuthService/PollDeviceAuthorization` |
| Policy action | `auth.poll_device_authorization`: every caller, also `anonymous`. The device code is the credential. |
| Idempotent option | no |
| Audit entry | for the poll that takes the tokens: action `auth.sign_in` by the new CLI sign-in, actor kind `cli_token`. Org the personal org, target kind `cli_sign_in`. Details `method` (`device`), `client_name`, and `device_sign_in`. |

The request `PollDeviceAuthorizationRequest` has the field `device_code`, a string. The response `PollDeviceAuthorizationResponse` has these fields:

| Field | Type | Description |
| --- | --- | --- |
| `state` | `DeviceAuthorizationState` | The answer of the poll ([DeviceAuthorizationState](#deviceauthorizationstate)). |
| `interval_seconds` | int32 | The least time until the next poll. |
| `tokens` | `CLITokens` | The tokens of the new CLI sign-in, for `DEVICE_AUTHORIZATION_STATE_APPROVED` only ([CLITokens](#clitokens)). |

| Error | Code |
| --- | --- |
| The device code is unknown or malformed, or a poll took its tokens already: `the device code is not valid: it is unknown or used. Start the sign-in again` | `unauthenticated` |
| The device code expired: `failed precondition: the device code expired before its approval. Start the sign-in again` | `failed_precondition` |

Two polls at the same time run one after the other, so only one of them gets the tokens.

```sh
curl -s -X POST -H 'Content-Type: application/json' -d '{"deviceCode":"<device code>"}' \
  $API/hopbox.v1.AuthService/PollDeviceAuthorization
```

```json
{"state":"DEVICE_AUTHORIZATION_STATE_PENDING","intervalSeconds":5}
```

### RefreshCLIToken

Trades a refresh token for a new access token and a new refresh token of the same CLI sign-in. Each refresh token works once. A replaced refresh token that comes back ends the whole CLI sign-in ([CLI tokens, API keys, admin API keys, and SSH keys](/reference/credentials/index.html.md#cli-tokens)). The one exception is a race of two refreshes: the newest replaced token works once more within 10 s.

| Item | Value |
| --- | --- |
| Procedure | `/hopbox.v1.AuthService/RefreshCLIToken` |
| Policy action | `auth.refresh_cli_token`: every caller, also `anonymous`. The refresh token is the credential. |
| Idempotent option | no |
| Audit entry | none for a refresh. A reuse writes `auth.end_cli_sign_in` with the detail `reason` (`refresh token reuse`), in the personal org of the account. |

The request `RefreshCLITokenRequest` has the field `refresh_token`, the newest refresh token of the CLI sign-in. The response `RefreshCLITokenResponse` has the field `tokens`, a `CLITokens` with the same `cli_sign_in_id`. Send no `Authorization` header: an expired access token in it fails the call before the refresh.

| Error | Code |
| --- | --- |
| The refresh token is unknown, malformed, replaced, expired, or of a CLI sign-in that ended: `the refresh token is not valid: it is unknown, used, or expired. Sign in again` | `unauthenticated` |

```sh
curl -s -X POST -H 'Content-Type: application/json' -d '{"refreshToken":"hbxrt_<token>"}' \
  $API/hopbox.v1.AuthService/RefreshCLIToken
```

```json
{"tokens":{"accessToken":"hbxat_<token>","accessTokenExpireTime":"2026-10-16T18:15:15.785548Z","refreshToken":"hbxrt_<token>","refreshTokenExpireTime":"2026-11-15T17:15:15.785548Z","accountId":"01a145b5-ad29-70a3-9065-fc2498a5d006","cliSignInId":"01a145b5-d8d5-71a9-a7ea-4d5f0c7a43f1"}}
```

## AccountService

Each call of `AccountService` acts on the account of the session or the CLI token that calls it. Another caller, even an instance admin, cannot reach the account. A member key can read the account that it acts as, with `GetAccount`, but it cannot change it. An org key has no account.

### GetAccount

Returns the account of the caller.

| Item | Value |
| --- | --- |
| Procedure | `/hopbox.v1.AccountService/GetAccount` |
| Policy action | `account.get`: the `session`, `cli_token`, or `member_key` of the account |
| Idempotent option | no |
| Audit entry | none |

The request `GetAccountRequest` has no fields. The response `GetAccountResponse` has the field `account`, an `Account`.

```sh
browser -d '{}' $API/hopbox.v1.AccountService/GetAccount
```

```json
{"account":{"id":"01a143f9-1cee-7ec0-9238-7976abaacfd6", "emails":[{"email":"root@example.org"}], "instanceAdmin":true, "personalOrgId":"01a143f9-1cee-7ec1-a648-71fea47ae54f", "version":"1", "createTime":"2026-10-16T09:09:21.774252Z"}}
```

### UpdateAccount

Changes the fields that the request sets, and keeps the others. It returns the account after the change.

| Item | Value |
| --- | --- |
| Procedure | `/hopbox.v1.AccountService/UpdateAccount` |
| Policy action | `account.update`: the `session` or `cli_token` of the account |
| Idempotent option | no |
| Audit entry | action `account.update`, org the personal org, target kind `account`, details `display_name` and `old_display_name` |

| Field | Type | Description |
| --- | --- | --- |
| `display_name` | optional string | The new display name ([Domain rules](/reference/domain-rules/index.html.md#display-names)). Leave it out to keep the name. |
| `expected_version` | optional int64 | The `version` of the account that the caller read. If the account has another version now, the call fails with `aborted` and changes nothing. |

A request that sets no field gets `invalid_argument` with the message `invalid argument: the update sets no field`. The response `UpdateAccountResponse` has the field `account`.

```sh
browser -d '{"displayName":"Ann Root","expectedVersion":"1"}' $API/hopbox.v1.AccountService/UpdateAccount
```

### DeleteAccount

Deletes the account, its personal org, its sessions, its emails, and its sign-in methods, and clears the session cookie. It takes the account out of each other org. In each one, it also deletes what the account made and did not share ([the org API](/reference/org-api/index.html.md#what-a-member-leaves-behind)). The delete also removes the CLI sign-ins, member keys, and SSH keys of the account.

| Item | Value |
| --- | --- |
| Procedure | `/hopbox.v1.AccountService/DeleteAccount` |
| Policy action | `account.delete`: the `session` or `cli_token` of the account |
| Idempotent option | no |
| Audit entries | action `account.delete`, target kind `account`, details `personal_org`. Each other org of the account gets the action `org.member_remove`, with the account as its target. Its details are `role` and `reason` (`account deleted`). The owners of the org see it. |

The request `DeleteAccountRequest` and the response `DeleteAccountResponse` have no fields.

| Error | Code |
| --- | --- |
| The account is the only owner of an org other than its personal org: `failed precondition: the account is the only owner of the org "<name>" (<id>). Make another member an owner, or delete the org, first` | `failed_precondition` |
| The account is the last instance admin: `failed precondition: the account is the last instance admin, and an install keeps at least one. Make another account an instance admin first` | `failed_precondition` |
| The personal org of the account, or another org of the account, is suspended: `failed precondition: org suspended: an instance admin suspended an org of the account, and nothing deletes its data until the suspension ends. Ask for the end of the suspension first` ([the org API](/reference/org-api/index.html.md#suspension)) | `failed_precondition` |

An instance admin that is not the last one loses the role with the delete. The delete locks the account, then each org of the account, before the check ([Control store tables](/reference/control-store/index.html.md#orgs)). So two owners of one org that delete their accounts at the same time leave one owner.

A call that runs at the same time as the delete, with a session or a CLI token of the account, can pass the check of its credential before the delete ends. When such a call then reads the account or its personal org, it fails with `not_found`, because the account no longer exists. A `GetAccount` or a second `DeleteAccount` of the deleted account gets `not_found` too.

### ListSignInMethods

Lists the sign-in methods of the account, oldest first.

| Item | Value |
| --- | --- |
| Procedure | `/hopbox.v1.AccountService/ListSignInMethods` |
| Policy action | `account.list_sign_in_methods`: the `session` or `cli_token` of the account |
| Idempotent option | no |
| Audit entry | none |

The request `ListSignInMethodsRequest` has no fields. The response `ListSignInMethodsResponse` has the field `methods`, a repeated `LinkedSignInMethod`.

```sh
browser -d '{}' $API/hopbox.v1.AccountService/ListSignInMethods
```

```json
{"methods":[{"id":"01a143f9-1cee-7ec2-8d2e-1c5d4f0b8a11", "method":"SIGN_IN_METHOD_LOGIN_LINK", "email":"root@example.org", "lastUseTime":"2026-10-16T09:09:21.774252Z", "createTime":"2026-10-16T09:09:21.774252Z"}]}
```

### RemoveSignInMethod

Removes one sign-in method of the account. An account keeps at least one. The emails of the account stay. While the account has no identity of GitHub or Google, a later sign-in that proves a verified email links its method again ([Sessions, cookies, and sign-in links](/reference/sessions/index.html.md#what-a-sign-in-does)). The removal does not end the sessions or the CLI sign-ins that the method started. End them with `SignOut` and [EndCLISignIn](#endclisignin).

| Item | Value |
| --- | --- |
| Procedure | `/hopbox.v1.AccountService/RemoveSignInMethod` |
| Policy action | `account.remove_sign_in_method`: the `session` or `cli_token` of the account |
| Idempotent option | no |
| Audit entry | action `account.remove_sign_in_method`, org the personal org, target kind `sign_in_method`, details `kind` and `email` |

The request `RemoveSignInMethodRequest` has the field `id`, the UUID of the method. The response `RemoveSignInMethodResponse` has no fields.

| Error | Code |
| --- | --- |
| The ID is not a UUID | `invalid_argument` |
| The account has no method with the ID | `not_found` |
| It is the last method: `failed precondition: an account keeps at least one sign-in method. Add another one first` | `failed_precondition` |

### ListCLISignIns

Lists the CLI sign-ins of the account, oldest first: each device that holds CLI tokens ([CLI tokens, API keys, admin API keys, and SSH keys](/reference/credentials/index.html.md#cli-tokens)). The account finds a sign-in that is not its own here, for example one on a lost laptop.

| Item | Value |
| --- | --- |
| Procedure | `/hopbox.v1.AccountService/ListCLISignIns` |
| Policy action | `auth.list_cli_sign_ins`: the `session` or `cli_token` of the account |
| Idempotent option | no |
| Audit entry | none |

The request `ListCLISignInsRequest` has no fields. The response `ListCLISignInsResponse` has the field `sign_ins`, a repeated `CLISignIn`:

| Field | Type | Description |
| --- | --- | --- |
| `id` | string | The UUID of the CLI sign-in, the `cli_sign_in_id` of its tokens. |
| `client_name` | string | The name that the CLI gave at the start of the device flow. |
| `client_ip` | string | The IP address that started the device flow, or empty. |
| `last_use_time` | `google.protobuf.Timestamp` | The last call with an access token of the sign-in, exact to a minute. |
| `create_time` | `google.protobuf.Timestamp` | The start of the CLI sign-in. |
| `last_use_ip` | string | The IP address of the last call with an access token of the sign-in. It starts as `client_ip`. |

```sh
browser -d '{}' $API/hopbox.v1.AccountService/ListCLISignIns
```

### EndCLISignIn

Ends one CLI sign-in of the account at once, with all of its tokens. Only the console calls it, so a stolen CLI token cannot end the other sign-ins of its owner. A CLI ends its own sign-in with `SignOut`.

| Item | Value |
| --- | --- |
| Procedure | `/hopbox.v1.AccountService/EndCLISignIn` |
| Policy action | `auth.end_cli_sign_in`: the `session` of the account |
| Idempotent option | no |
| Audit entry | action `auth.end_cli_sign_in`, org the personal org, target kind `cli_sign_in`, details `reason` (`ended by the account`) |

The request `EndCLISignInRequest` has the field `id`, the UUID of the CLI sign-in. The response `EndCLISignInResponse` has no fields.

| Error | Code |
| --- | --- |
| The ID is not a UUID | `invalid_argument` |
| The account has no CLI sign-in with the ID | `not_found` |

## AuditService

### ListAuditEntries

Returns entries of the audit log ([Control store tables](/reference/control-store/index.html.md#audit_log)), newest first. Org owners and org admins read the entries of their org. Instance admins read the entries of every org and of the whole install.

| Item | Value |
| --- | --- |
| Procedure | `/hopbox.v1.AuditService/ListAuditEntries` |
| Policy action | `audit.list_org` for one org: a `session` or `cli_token` with the role `owner` or `admin` in the org. Also a `member_key` of such an account, in the org of the key, and a `cli_token` of an instance admin. Also the `session` of an instance admin with a sign-in within the last 10 minutes. `audit.list_install` for the whole install: a `cli_token` of an instance admin, or the `session` of an instance admin with a sign-in within the last 10 minutes ([Sessions, cookies, and sign-in links](/reference/sessions/index.html.md#recent-sign-in)). |
| Idempotent option | no |
| Audit entry | none |

The request `ListAuditEntriesRequest` has these fields:

| Field | Type | Default | Description |
| --- | --- | --- | --- |
| `org_id` | optional string | the org of a member key, or else the personal org of the caller | The UUID of the org whose entries to read. |
| `whole_install` | bool | `false` | Reads the entries of every org and of the install. `org_id` must then be left out. |
| `page_size` | int32 | 50 | The most entries in one response, 1 to 500. |
| `page_token` | string | empty | The `next_page_token` of the response before. Leave it empty for the newest entries. |

The response `ListAuditEntriesResponse` has the fields `entries`, a repeated `AuditEntry`, and `next_page_token`. The token is empty on the last page.

| `AuditEntry` field | Type | Description |
| --- | --- | --- |
| `id` | string | The UUID of the entry. |
| `org_id` | optional string | The UUID of the org of the action. It is unset for an action on the whole install. |
| `actor` | `AuditActor` | `kind`, such as `session` or `local_root`, then `account_id` and `credential_id`, or empty strings. |
| `action` | string | The action, such as `auth.sign_in`. |
| `target_kind` | string | The kind of object, such as `account`, or empty. |
| `target_id` | string | The ID or the name of the object, or empty. |
| `details` | map of string to string | More facts. They never hold a secret value. |
| `request_id` | string | The ID of the API request, or empty. |
| `create_time` | `google.protobuf.Timestamp` | The time of the action. |

| Error | Code |
| --- | --- |
| A caller without the role in the org, or a member. The org can also not exist. | `permission_denied` |
| `whole_install` from a caller that is not an instance admin | `permission_denied` |
| A session of an instance admin whose sign-in is more than 10 minutes old, with `whole_install` or an org where it is not an owner or an admin | `permission_denied` with the detail `hopbox.v1.RecentSignInRequired` ([Sessions, cookies, and sign-in links](/reference/sessions/index.html.md#recent-sign-in)) |
| `whole_install` with `org_id` | `invalid_argument` |
| An `org_id` that is not a UUID | `invalid_argument` |
| A page size out of range | `invalid_argument` |
| A page token that this call did not return | `invalid_argument` |

```sh
browser -d '{"wholeInstall":true,"pageSize":2}' $API/hopbox.v1.AuditService/ListAuditEntries
```

```json
{"entries":[{"id":"01a143f9-1cf1-73f3-a6ea-4a65c92b2116", "actor":{"kind":"session", "accountId":"01a143f9-1cee-7ec0-9238-7976abaacfd6", "credentialId":"01a143f9-1cf0-7418-a8f1-b79af67c1449"}, "action":"instance_admin.grant", "targetKind":"account", "targetId":"01a143f9-1cee-7ec0-9238-7976abaacfd6", "details":{"login_link":"01a143f9-0545-7bae-a957-716b4710a8c0"}, "requestId":"01a143f9-1cee-7170-bd72-5ceae51d8bc8", "createTime":"2026-10-16T09:09:21.774252Z"}, {"id":"01a143f9-1cf0-78f1-8413-dbe73566e206", "orgId":"01a143f9-1cee-7ec1-a648-71fea47ae54f", "actor":{"kind":"session", "accountId":"01a143f9-1cee-7ec0-9238-7976abaacfd6", "credentialId":"01a143f9-1cf0-7418-a8f1-b79af67c1449"}, "action":"auth.sign_in", "targetKind":"session", "targetId":"01a143f9-1cf0-7418-a8f1-b79af67c1449", "details":{"email":"root@example.org", "login_link":"01a143f9-0545-7bae-a957-716b4710a8c0", "method":"login_link", "new_account":"true"}, "requestId":"01a143f9-1cee-7170-bd72-5ceae51d8bc8", "createTime":"2026-10-16T09:09:21.774252Z"}], "nextPageToken":"AaFD-Rzwfviy4yXwHf2rvA"}
```

## APIKeyService

`hopbox.v1.APIKeyService` manages the API keys of an org ([CLI tokens, API keys, admin API keys, and SSH keys](/reference/credentials/index.html.md#api-keys)). Only a `session` or a `cli_token` calls it, so no API key makes, lists, or deletes another key. The examples use the access token of a CLI sign-in:

```sh
TOKEN=hbxat_<token>
cli() { curl -s -X POST -H 'Content-Type: application/json' -H "Authorization: Bearer $TOKEN" "$@"; }
```

### CreateAPIKey

Makes an API key. Any member makes a member key for itself. The org admins and owners make org keys. The response holds the only copy of the key.

| Item | Value |
| --- | --- |
| Procedure | `/hopbox.v1.APIKeyService/CreateAPIKey` |
| Policy action | `api_key.create`: a `session` or `cli_token` with a role in the org. A member key needs the role `member` or more. An org key needs `admin` or `owner`. |
| Idempotent option | yes. The install keeps no copy of the secret. So a retry with the same idempotency key fails with `already_exists` and the ID of the key ([the API server](/reference/api/index.html.md#idempotency-keys)). |
| Audit entry | action `api_key.create`, org the org of the key, target kind `api_key`, details `kind`, `name`, and `expire_time` |

The request `CreateAPIKeyRequest` has these fields:

| Field | Type | Default | Description |
| --- | --- | --- | --- |
| `org_id` | optional string | the personal org of the caller | The UUID of the org of the key. |
| `kind` | `APIKeyKind` | `API_KEY_KIND_MEMBER` | The kind of the key. |
| `name` | string | none, required | A name of 1 to 64 characters, such as `ci` ([Domain rules](/reference/domain-rules/index.html.md#credential-names)). |
| `lifetime` | `APIKeyLifetime` | 90 days | A preset lifetime. Give it or `expire_time`, not both. |
| `expire_time` | `google.protobuf.Timestamp` | none | A custom end, in the future and at most 365 days ahead. |

The response `CreateAPIKeyResponse` has the fields `key`, an `APIKey`, and `secret`, the key itself, which starts with `hbx_`.

| Error | Code |
| --- | --- |
| The caller is a key, or has no role or too low a role in the org. The org can also not exist. | `permission_denied` |
| No name, a bad name, an unknown kind or lifetime, or an expiry in the past or more than 365 days ahead: `invalid argument: the expiry date <time> must be in the future and at most 365 days ahead`. The policy check comes first, so a caller that cannot make a key gets `permission_denied` instead. | `invalid_argument` |
| An org key in a personal org: `invalid argument: a personal org has no shared machines, so it takes no org key. Make a member key, or an org key in a team org` | `invalid_argument` |
| The org holds 100 API keys, expired ones included: `usage limit exceeded: an org holds at most 100 API keys. Revoke one first` | `resource_exhausted` |
| A retry with an idempotency key that made a key already | `already_exists` |

```sh
cli -H 'Idempotency-Key: 0b2c5e1a-key-ci' -d '{"name":"ci","lifetime":"API_KEY_LIFETIME_30_DAYS"}' \
  $API/hopbox.v1.APIKeyService/CreateAPIKey
```

```json
{"key":{"id":"01a145b6-1aaf-7620-989d-f244cbd2aba6","orgId":"01a145b5-ad29-70a4-becc-5f6e654b8efa","kind":"API_KEY_KIND_MEMBER","name":"ci","accountId":"01a145b5-ad29-70a3-9065-fc2498a5d006","creatorAccountId":"01a145b5-ad29-70a3-9065-fc2498a5d006","expireTime":"2026-11-15T17:15:24.718683Z","createTime":"2026-10-16T17:15:24.718683Z"},"secret":"hbx_<key>"}
```

### ListAPIKeys

Lists the API keys of an org that the caller manages, oldest first: the own member keys for a member, and every key of the org for an org admin or owner.

| Item | Value |
| --- | --- |
| Procedure | `/hopbox.v1.APIKeyService/ListAPIKeys` |
| Policy action | `api_key.list`: a `session` or `cli_token` with a role in the org |
| Idempotent option | no |
| Audit entry | none |

The request `ListAPIKeysRequest` has the field `org_id`, an optional string. The default is the personal org of the caller. The response `ListAPIKeysResponse` has the field `keys`, a repeated `APIKey`. An expired key stays in the list for 30 days. Then the leader deletes it.

```sh
cli -d '{}' $API/hopbox.v1.APIKeyService/ListAPIKeys
```

### RevokeAPIKey

Deletes an API key at once. A member deletes its own member keys. An org admin or owner deletes any key of the org.

| Item | Value |
| --- | --- |
| Procedure | `/hopbox.v1.APIKeyService/RevokeAPIKey` |
| Policy action | `api_key.revoke`: a `session` or `cli_token`. Its own member key needs the role `member` or more. Any other key of the org needs `admin` or `owner`. |
| Idempotent option | no |
| Audit entry | action `api_key.revoke`, org the org of the key, target kind `api_key`, details `kind` and `name` |

The request `RevokeAPIKeyRequest` has the field `id`, the UUID of the key. The response `RevokeAPIKeyResponse` has no fields.

| Error | Code |
| --- | --- |
| The ID is not a UUID | `invalid_argument` |
| No key has the ID, or the key is in an org in which the caller has no role | `not_found` |
| A member and a key of another account, or an org key | `permission_denied` |

```sh
cli -d '{"id":"01a145b6-1aaf-7620-989d-f244cbd2aba6"}' $API/hopbox.v1.APIKeyService/RevokeAPIKey
```

## SSHKeyService

`hopbox.v1.SSHKeyService` manages the SSH keys of the account of the caller ([CLI tokens, API keys, admin API keys, and SSH keys](/reference/credentials/index.html.md#ssh-keys)). Each call acts on the account of the `session` or the `cli_token`. A member key or an org key cannot call it, because an SSH key works in every org of its account.

### AddSSHKey

Adds a public key to the account. A key that an account of the install holds already is refused, so each key works for one account only.

| Item | Value |
| --- | --- |
| Procedure | `/hopbox.v1.SSHKeyService/AddSSHKey` |
| Policy action | `ssh_key.add`: the `session` or `cli_token` of the account |
| Idempotent option | yes. A retry with the same idempotency key returns the first response. |
| Audit entry | action `ssh_key.add`, org the personal org, target kind `ssh_key`, details `fingerprint`, `type`, `bits`, and `name` |

The request `AddSSHKeyRequest` has these fields:

| Field | Type | Default | Description |
| --- | --- | --- | --- |
| `public_key` | string | none, required | One line of the `authorized_keys` form, as in `~/.ssh/id_ed25519.pub`. |
| `name` | string | the comment of the key | A name of at most 64 characters. A comment that is not a valid name gives an empty name. |

The response `AddSSHKeyResponse` has the field `key`, an `SSHKey`.

| Error | Code |
| --- | --- |
| The key is not one that hopbox takes, for example `invalid argument: invalid SSH key: the RSA key has 2048 bits, and hopbox takes at least 3072. Make an Ed25519 key with ssh-keygen -t ed25519` | `invalid_argument` |
| A bad name | `invalid_argument` |
| An account holds the key already, this one or another: `already exists: the install holds the key <fingerprint> already. Add another key` | `already_exists` |
| The account holds 50 SSH keys: `usage limit exceeded: an account holds at most 50 SSH keys. Remove one first` | `resource_exhausted` |

```sh
cli -H 'Idempotency-Key: ssh-1' -d "{\"publicKey\":\"$(cat ~/.ssh/id_ed25519.pub)\"}" \
  $API/hopbox.v1.SSHKeyService/AddSSHKey
```

```json
{"key":{"id":"01a145b6-6c4e-72c0-b0ee-895c30b08f74","name":"ann@laptop","type":"ssh-ed25519","bits":256,"publicKey":"ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIAwqTIQ7y4LoYxC/02UEz3yB/6qT2C6Oj4OEOIHfnVze","fingerprint":"SHA256:DRdf39Nvm+zAXRzjpCOtwde9E/UUaXIbgz0+g1NA+10","createTime":"2026-10-16T17:15:45.613938Z"}}
```

### ListSSHKeys

Lists the SSH keys of the account, oldest first.

| Item | Value |
| --- | --- |
| Procedure | `/hopbox.v1.SSHKeyService/ListSSHKeys` |
| Policy action | `ssh_key.list`: the `session` or `cli_token` of the account |
| Idempotent option | no |
| Audit entry | none |

The request `ListSSHKeysRequest` has no fields. The response `ListSSHKeysResponse` has the field `keys`, a repeated `SSHKey`.

### RemoveSSHKey

Removes an SSH key from the account. The removal publishes the event `edge.access_changed`, so the edge servers end the open SSH connections of the key ([the SSH front door](/reference/ssh-front-door/index.html.md#checks-of-an-open-connection)).

| Item | Value |
| --- | --- |
| Procedure | `/hopbox.v1.SSHKeyService/RemoveSSHKey` |
| Policy action | `ssh_key.remove`: the `session` or `cli_token` of the account |
| Idempotent option | no |
| Audit entry | action `ssh_key.remove`, org the personal org, target kind `ssh_key`, details `fingerprint`, `type`, `bits`, and `name` |

The request `RemoveSSHKeyRequest` has the field `id`, the UUID of the key. The response `RemoveSSHKeyResponse` has no fields.

| Error | Code |
| --- | --- |
| The ID is not a UUID | `invalid_argument` |
| The account has no key with the ID | `not_found` |

## Audit entries

The sign-in calls write these entries in the transaction of the sign-in. Their actor is the new session: actor kind `session`, the account, and the session as the credential. Their org is the personal org of the account, so the account reads them with `ListAuditEntries`. [CLI tokens, API keys, admin API keys, and SSH keys](/reference/credentials/index.html.md#audit-entries) lists the entries of the device flow, the CLI tokens, the API keys, and the SSH keys.

| Action | When | Target | Details |
| --- | --- | --- | --- |
| `account.create` | A sign-in created the account. | the account | `method`, `email`, `personal_org` |
| `account.link_sign_in_method` | A sign-in linked its method to the account through a verified email. | the new method | `kind`, `email` |
| `account.link_sign_in_method` | A link from a session added a method to the account ([Sessions, cookies, and sign-in links](/reference/sessions/index.html.md#link-a-new-sign-in-method)). The actor is that session, not a new one. | the new method | `kind`, `email`, `session`, and `magic_link` or the details of the provider: `oauth_flow`, `subject`, and for GitHub `login` |
| `auth.sign_in` | Every sign-in with a link. | the session | `method` (`email_link` or `login_link`), `email`, `new_account`, and `magic_link` or `login_link`: the UUID of the link |
| `auth.sign_in` | Every sign-in with GitHub or Google ([Sign-in with GitHub and Google](/reference/oauth/index.html.md#audit-entries)). | the session | `method` (`github` or `google`), `email`, `new_account`, `oauth_flow`, `subject`, and for GitHub `login` |
| `auth.sign_in` | A poll of a device flow took the CLI tokens. The actor is the new CLI sign-in, with actor kind `cli_token`. | the CLI sign-in, target kind `cli_sign_in` | `method` (`device`), `client_name`, `device_sign_in` |
| `instance_admin.grant` | A login link made the account an instance admin. This entry has no org. | the account | `login_link` |
