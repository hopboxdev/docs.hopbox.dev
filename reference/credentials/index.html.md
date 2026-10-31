# CLI tokens, API keys, admin API keys, and SSH keys

> The CLI tokens, API keys, admin API keys, and SSH keys: their forms and prefixes, lifetimes, the device flow and its limits, refresh rotation, the rules of each key, and how the request gate reads them. Also curl examples of the device flow, a refresh, and an API key.

People sign in to the console with a session ([Sessions, cookies, and sign-in links](/reference/sessions/index.html.md)). Every other client proves who calls with one of the credentials on this page. The hopbox CLI uses CLI tokens from a device flow. Scripts use API keys. The hopbox Terraform provider uses admin API keys. SSH clients use SSH keys. This page gives the form of each credential, its lifetime, its rules, and how the request gate reads it. [The sign-in, account, and audit API](/reference/auth-api/index.html.md) lists the calls of the public API, and [the admin API](/reference/admin-api/index.html.md#adminkeyservice) the calls for admin API keys. The code is in `internal/auth`, `internal/service`, and `internal/api`.

## Forms and prefixes

Each credential that a client keeps starts with a prefix, then 32 random bytes in base64url without padding: 43 characters. The prefix tells the request gate which resolver knows the credential, and lets secret scanners find a leaked one.

| Credential | Prefix | Example form | Sent as |
| --- | --- | --- | --- |
| CLI access token | `hbxat_` | `hbxat_<43 characters>` | `Authorization: Bearer <token>` |
| CLI refresh token | `hbxrt_` | `hbxrt_<43 characters>` | the field `refresh_token` of `RefreshCLIToken`, never a header |
| API key, member or org | `hbx_` | `hbx_<43 characters>` | `Authorization: Bearer <key>` |
| Admin API key | `hbxadm_` | `hbxadm_<43 characters>` | `Authorization: Bearer <key>` |
| Device code | none | `<43 characters>` | the field `device_code` of `PollDeviceAuthorization` |
| User code | none | 8 letters, such as `WDJB-MJHT` | typed by a person in the console |

The control store keeps only the SHA-256 of each credential, of the whole text with its prefix ([Control store tables](/reference/control-store/index.html.md#cli_sign_ins)). A copy of the database thus signs nobody in. The audit log never holds a credential: it refuses a detail value that starts with one of the prefixes ([Control store tables](/reference/control-store/index.html.md#audit_log)). The pre-commit secret scan in `.pre-commit-config.yaml` refuses a commit with a credential of these forms. The server never logs a credential.

A credential of another form never matches a stored hash, so the server refuses it before any lookup.

## Lifetimes

| Credential | Lifetime | Use | Source |
| --- | --- | --- | --- |
| Device code and user code | 10 minutes | once | A phase 1 default |
| CLI access token | 1 hour | many calls | A product decision |
| CLI refresh token | 30 days. Each use gives a new refresh token with a new 30 days. | once | the same |
| API key | 30, 90, or 365 days, or a custom date at most 365 days ahead. The default is 90 days. | many calls | the same |
| Admin API key | as an API key | many calls | A product decision |
| SSH key | no expiry. It works until the account removes it. | many connections | A product decision |

All times use the clock of Postgres. Each key and each CLI sign-in records its last use at most once per minute. In maintenance mode, nothing records a use, so a read writes nothing to the control store ([the API server](/reference/api/index.html.md#maintenance-mode)).

## How the request gate reads them

The API listener asks its resolvers in this order: the session cookie, CLI access tokens, API keys, then admin API keys ([the API server](/reference/api/index.html.md#request-gate)). Each resolver of a bearer credential reads only the header `Authorization: Bearer <credential>` whose credential starts with its own prefix. Each one keeps the contract of a resolver ([the API server](/reference/api/index.html.md#the-contract-of-a-resolver)):

| Case | Result |
| --- | --- |
| The header is absent, has another scheme, or holds a credential with another prefix. | The resolver passes the call to the next one. |
| The credential has the prefix, but it is malformed, unknown, expired, or revoked. A member key whose account left the org of the key counts as revoked. A CLI access token or a member key of a suspended account counts as revoked too ([the admin API](/reference/admin-api/index.html.md#suspendaccount)). | `unauthenticated` with the message `the credential is not valid`. The log record `api credential refused` has the reason. |
| The control store does not answer. | `unavailable` with the message `the server failed to check the credential. Retry later`. |
| The credential is live. | The caller is `cli_token`, `member_key`, `org_key`, or `admin_key` ([the API server](/reference/api/index.html.md#callers-and-policy)). The call records the use of the credential. |

A bearer credential needs no `Origin` header and no Connect protocol header, because a browser never sends it on its own. The check against cross-site requests is for the cookie only ([Sessions, cookies, and sign-in links](/reference/sessions/index.html.md#calls-with-the-cookie)).

A call with an `Authorization` header that no resolver knows gets `unauthenticated`, so a credential never passes as anonymous.

The MCP server at `/mcp` takes a CLI access token, a member key, or an org key in the same header. It refuses an admin API key and ignores the session cookie ([the MCP server](/reference/mcp/index.html.md#credentials)).

A stream, such as `WatchMachines`, checks its credential again while it runs. The stream ends with its credential ([the API server](/reference/api/index.html.md#streams-and-their-credential)).

## The device flow

The hopbox CLI signs in with a device flow, in the shape of RFC 8628. A person approves the CLI in the console, and the CLI gets CLI tokens.

1. The CLI calls `StartDeviceAuthorization` without a credential ([the sign-in, account, and audit API](/reference/auth-api/index.html.md#startdeviceauthorization)). The response has a device code, a user code, the console page `https://<domain>/device`, and the poll interval of 5 s.
2. The CLI shows the user code and the page, and polls `PollDeviceAuthorization` with the device code every interval ([the sign-in, account, and audit API](/reference/auth-api/index.html.md#polldeviceauthorization)). Each poll before the approval gets the state `DEVICE_AUTHORIZATION_STATE_PENDING`.
3. The person signs in to the console, opens `/device`, and types the user code. The console page calls `GetDeviceAuthorization` with the session, and shows the client name, the IP address, and the start time of the device sign-in ([the sign-in, account, and audit API](/reference/auth-api/index.html.md#getdeviceauthorization)).
4. The person checks that this is their own CLI, and approves. The page calls `ApproveDevice` with the code and the start time that it showed ([the sign-in, account, and audit API](/reference/auth-api/index.html.md#approvedevice)). Without that start time, the approval fails. The approval also needs a session whose sign-in is at most 10 minutes old ([Sessions, cookies, and sign-in links](/reference/sessions/index.html.md#recent-sign-in)). After an older sign-in, the console asks the person to sign in again, then shows the code again.
5. The next poll gets the state `DEVICE_AUTHORIZATION_STATE_APPROVED` with the CLI tokens of the account of the person. The poll starts a CLI sign-in. A later poll with the same device code gets `unauthenticated`. If an instance admin suspended the account after the approval, the poll starts no CLI sign-in. It fails with `permission_denied` and the message `permission denied: an instance admin suspended this account, so it cannot sign in. The reason: <reason>`, and it uses up the device code.

A poll that comes less than the interval after the poll before gets the state `DEVICE_AUTHORIZATION_STATE_SLOW_DOWN`. The interval then grows by 5 s, and the larger interval holds for every later poll (RFC 8628 section 3.5). A device code that nobody approves within 10 minutes expires, and a poll gets `failed_precondition`. If no poll took the tokens of an approved code, the code also expires after its 10 minutes.

The user code has 8 letters from the alphabet `BCDFGHJKLMNPQRSTVWXZ`, shown in two groups of 4, such as `WDJB-MJHT`. It has no vowels, so it spells no words. `ApproveDevice` ignores case, dashes, and spaces, so `wdjb mjht` works too.

The device flow needs the domain of the install, for the console page. Without a domain, `StartDeviceAuthorization` gets `failed_precondition`.

### Limits of the device flow

The limits count rows in the control store, so they hold across every control server. Each start takes the transaction lock `LockKeyDeviceAuthorizations`, and each approval takes `LockKeyDeviceApprovals` ([the bundled Postgres](/reference/postgres/index.html.md#advisory-lock-keys)). So calls at the same time cannot pass a limit together.

| Limit | Value | Error |
| --- | --- | --- |
| Starts from one client: one IPv4 address, or one IPv6 /64 | 20 per hour | `resource_exhausted`: `too many requests: this client started the most device sign-ins of an hour. Wait, then retry` |
| Device sign-ins that wait for an approval, in the whole install | 1000 | `resource_exhausted`: `too many requests: the install has the most device sign-ins that wait for an approval. Retry in a few minutes` |
| Lookups and approvals with a wrong user code or a wrong start time, for one account | 10 per hour | `resource_exhausted`: `too many requests: this account entered the most wrong codes of an hour. Wait, then retry` |

The limit of a client works as the limit of magic links does ([Sessions, cookies, and sign-in links](/reference/sessions/index.html.md#rate-limits)). It skips a loopback peer, and reads the address that an edge server forwards. The limit of wrong codes keeps a person from guessing the code of another person. The step with the start time is a defense against phishing: someone can start a device flow and send its code to a victim. The lookup shows the victim a client that is not theirs before anything is approved (RFC 8628 section 5.4). The console page `/device` shows the client and asks the person to approve it ([Console](/reference/console/index.html.md#pages)). After 10 wrong codes in an hour, the right code fails too, until an hour after the oldest wrong one. A wrong code writes the log record `device approval refused` at level info, with `component=service` and `account`.

## CLI tokens

A CLI sign-in is a family of tokens that one device flow started. Each CLI sign-in has one ID, which every refresh keeps. It is the credential ID of the audit entries of the CLI ([Control store tables](/reference/control-store/index.html.md#audit_log)).

- The access token goes in the `Authorization` header of each call. The caller is `cli_token`, which acts as its account in every org of the account ([the API server](/reference/api/index.html.md#callers-and-policy)).
- The refresh token works once. `RefreshCLIToken` trades it for a new access token and a new refresh token of the same CLI sign-in ([the sign-in, account, and audit API](/reference/auth-api/index.html.md#refreshclitoken)). The old access token keeps working until its hour ends.
- A refresh token that a refresh replaced, and that comes back, means that someone copied it. The server then ends the whole CLI sign-in. It deletes every access token and every refresh token of the sign-in. It writes the audit entry `auth.end_cli_sign_in` with the reason `refresh token reuse`. It also writes the log record `a replaced CLI refresh token came back, so its sign-in ends` at level warn, with `account` and `cli_sign_in`. The call gets `unauthenticated`. The person signs in again with the device flow.
- A refresh token that is unknown, replaced, or expired gets `unauthenticated`. The server then logs `CLI refresh token refused` at level info, with `component=service`.
- Two refreshes with one refresh token, such as two CLI processes of one sign-in, must not end the sign-in. So the newest replaced refresh token works once more within 10 s of its replacement (`RefreshGrace`). This holds only while the token that replaced it is unused. That use gets a new pair of the same sign-in. A second use within the grace, or any use after it, counts as a reuse. The next normal refresh of either pair drops every other live refresh token of the sign-in. So only one chain goes on. The access token of the dropped pair stops working at once, and the next use of its refresh token ends the sign-in. A thief who redeems a copied token within the grace thus loses at the next refresh of the owner, and ends the sign-in with it.
- A refresh changes state and has no idempotency key. After a lost answer, the server replaced the refresh token, but the client still holds the old one. So the hopbox CLI sends the refresh once more with the old token, if that retry starts within 10 s of the first try ([Go client](/reference/go-client/index.html.md#retries)). The grace takes that one replay, so the command goes on, and the server records no reuse. The owner decided this exception to Q38 on 2026-10-25. If the second answer gets lost too, the command fails, and the CLI keeps its stored tokens. The grace is spent, so the next refresh with the old token counts as a reuse. The server then ends the CLI sign-in, and the person signs in again.
- `SignOut` with an access token ends the CLI sign-in of the token, with all of its tokens ([the sign-in, account, and audit API](/reference/auth-api/index.html.md#signout)).
- `ListCLISignIns` shows the account each of its CLI sign-ins. Each one has the name of its client, the IP address that started it, and the time and IP address of its last use. `EndCLISignIn` ends one of them from the console, for example the sign-in of a lost laptop ([the sign-in, account, and audit API](/reference/auth-api/index.html.md#endclisignin)). A CLI token cannot call it, so a stolen token cannot end the other sign-ins of its owner.
- A delete of the account ends each of its CLI sign-ins. So does a suspend of the account ([the admin API](/reference/admin-api/index.html.md#suspendaccount)). While the account is suspended, no access token of the account works.

The CLI sends no `Authorization` header to `RefreshCLIToken`. An expired access token in the header fails the call with `unauthenticated` before the refresh.

The hopbox CLI keeps its tokens in `credentials.toml`, mode 0600. It refreshes them under a lock of its config dir, so its processes make one refresh, and never send a replaced token ([hopbox CLI](/reference/cli/index.html.md#sign-in-and-tokens)).

## API keys

An API key belongs to one org and expires ([Domain rules](/reference/domain-rules/index.html.md#api-key-kinds)). There are two kinds:

| Kind | Caller | Rights |
| --- | --- | --- |
| `member` | `member_key` | It acts as the account that made it, only in the org of the key. It has the role of that account in the org, read on each call. It works only while the account is a member of the org and is not suspended. A suspend keeps the key in the store, so the key works again after the unsuspend ([the admin API](/reference/admin-api/index.html.md#suspendaccount)). It can read the account, but not change it. |
| `org` | `org_key` | A service credential of the org, without an account. It has the rights of an org member without personal resources. So it manages no members, no billing, and no keys. It reads its org, the usage of the org, the members ([the org API](/reference/org-api/index.html.md)), and the shared and all env vars and secrets of the org ([the env var API](/reference/vars-api/index.html.md#who-can-call)). It reads the names of the secrets, and the names and values of the env vars that are not secrets. It reads no account and no audit log, and it sets no env var. If its creator leaves the org, the key stays. In a later release, a machine that an org key creates is shared from the start. |

No API key can make, list, or delete a key. Only a `session` or a `cli_token` calls `APIKeyService` ([the sign-in, account, and audit API](/reference/auth-api/index.html.md#apikeyservice)):

| Who | Can |
| --- | --- |
| Any org member | make a member key for itself, list its own member keys, delete its own member keys |
| An org admin or org owner | also make org keys, list every key of the org, delete every key of the org |

The default org of a new key is the personal org of the caller. A personal org takes no org key, because it has no shared machines. An org holds at most 100 API keys (`MaxAPIKeysPerOrg`), expired keys included. The response of `CreateAPIKey` holds the only copy of the key. Each API key has its own scope of idempotency keys, so two scripts never replay each other ([the API server](/reference/api/index.html.md#idempotency-keys)).

## Admin API keys

An admin API key has install scope and calls the admin API on the API listener ([the admin API](/reference/admin-api/index.html.md#where-the-admin-api-is-served)). The hopbox Terraform provider uses one. It belongs to no account and no org, so it has its own table ([Control store tables](/reference/control-store/index.html.md#admin_keys)). It expires like an API key.

The caller `admin_key` can do every admin call except the calls of `KeyringService`, `MailService`, `LoginLinkService`, and `AdminKeyService` ([the API server](/reference/api/index.html.md#callers-and-policy)). So no key makes another key.

Root on a control server makes admin keys in two ways, through the admin socket:

- `hopboxd admin admin-key create` makes a key and prints it once ([hopboxd](/reference/hopboxd/index.html.md#admin-admin-key-create)).
- `hopboxd admin admin-key register --sha256` registers a key from the SHA-256 of a key that you made yourself, so the secret never reaches the server ([hopboxd](/reference/hopboxd/index.html.md#admin-admin-key-register)). The first server of an install gets its first key this way. `hopboxd install --bootstrap-admin-key-hash` writes the hash to a file, and the control role registers it at its first start ([Server layout](/reference/server-layout/index.html.md#the-bootstrap-admin-api-key)). A second registration of the same hash returns the first key, unchanged. If that key expired, the registration fails with `failed_precondition`.

A key for `register` is `hbxadm_` and 32 random bytes in base64url without padding. Its hash is the SHA-256 of the whole text, in lowercase hex:

```sh
KEY="hbxadm_$(head -c 32 /dev/urandom | base64 | tr '+/' '-_' | tr -d '=')"
HASH="$(printf '%s' "$KEY" | shasum -a 256 | cut -d' ' -f1)"
```

The key has 50 characters. Keep `$KEY` as a secret, and pass only `$HASH` to the server. In Go, `auth.NewPrefixedToken(auth.AdminKeyPrefix)` makes such a key, and the service operation `RegisterAdminKey` of `internal/service` registers its hash for `hopboxd install`. [Run the control role on a laptop](/guide/run-hopboxd/index.html.md#make-an-admin-api-key) makes and registers admin API keys on a laptop install.

An instance admin also makes, lists, and revokes admin keys with an admin session, the session of a sign-in within the last 10 minutes ([the admin API](/reference/admin-api/index.html.md#admin-sessions)). The response of `CreateAdminKey` holds the secret once, as for root. An admin session cannot register a hash.

## SSH keys

An SSH key is a credential of one account. It opens the machines that the account uses, through the SSH front door of the edge servers ([the SSH front door](/reference/ssh-front-door/index.html.md)). It never creates or changes a machine. The calls are in `SSHKeyService` ([the sign-in, account, and audit API](/reference/auth-api/index.html.md#sshkeyservice)).

| Rule | Value |
| --- | --- |
| Key types | `ssh-ed25519`, `ecdsa-sha2-nistp256`, `ecdsa-sha2-nistp384`, `ecdsa-sha2-nistp521`, and `ssh-rsa` of 3072 to 16384 bits |
| Refused | RSA keys under 3072 or over 16384 bits, DSA keys, and security keys (`sk-` types). Also certificates, key options such as `command=`, and a text with more than one line that is not empty. |
| Form | one line of the `authorized_keys` form, as in `~/.ssh/id_ed25519.pub`, at most 16 KiB |
| Fingerprint | the SHA-256 fingerprint, as `ssh-keygen -l` prints it, such as `SHA256:DRdf39Nvm+zAXRzjpCOtwde9E/UUaXIbgz0+g1NA+10` |
| Unique | Each fingerprint is unique in the whole install, so a key works for one account only. The SSH front door finds the account of a connection by its key alone. |
| Name | The name of the call. If the call gives none and the comment of the key is a valid name ([Domain rules](/reference/domain-rules/index.html.md#credential-names)), the name is the comment, such as `ann@laptop`. Else the name is empty. |
| Stored | The key type, the size in bits, the key in the `authorized_keys` form without its comment, and the fingerprint |

An RSA key under 3072 bits falls short of 128-bit security (NIST SP 800-57 part 1). An account holds at most 50 SSH keys (`MaxSSHKeysPerAccount`).

The front door checks a key only after the client signed with it, and it checks a key of any type. A key that no account holds opens nothing. A key of a suspended account opens nothing either, and it works again after the unsuspend ([the SSH front door](/reference/ssh-front-door/index.html.md#who-opens-a-machine)). A removal of a key ends its open SSH connections within seconds, through the event `edge.access_changed` ([Control store tables](/reference/control-store/index.html.md#the-event-edgeaccess_changed)).

Today an account adds a public key without a proof that it holds the private key. Such a key opens nothing for a party without the private key, but it keeps the real owner from adding the key to another account. The `already_exists` error does not say which account holds the key. No release adds the proof yet. The plan for it: the CLI signs a challenge of the server with the private key. Only a `session` or a `cli_token` of the account manages its SSH keys. A key of one org, a member key or an org key, cannot, because an SSH key works in every org of its account.

## Examples with curl

These examples call the API of a laptop install with curl, so they show each call that a client makes. They need the install of [Run the control role on a laptop](/guide/run-hopboxd/index.html.md) with the domain `example.com`, and the shell function `browser` of [First sign-in on a new install](/guide/first-sign-in/index.html.md#sign-in-with-the-link). That function calls the API as the console does, with the session cookie of a signed-in account.

```sh
API=http://127.0.0.1:7080
```

### A device sign-in

The CLI starts a device sign-in. The call needs no credential:

```sh
curl -s -X POST -H 'Content-Type: application/json' -d '{"clientName":"hopbox CLI on user1-laptop"}' \
  $API/hopbox.v1.AuthService/StartDeviceAuthorization | tee /tmp/hb/device.json
```

```json
{"deviceCode":"<device code>","userCode":"QJTL-PHNV","verificationUri":"https://example.com/device","verificationUriComplete":"https://example.com/device?code=QJTL-PHNV","expireTime":"2026-10-16T17:24:56.701147Z","intervalSeconds":5}
```

The CLI shows the user code and the page, and keeps the device code secret. It then polls every 5 seconds:

```sh
DEVICE=$(python3 -c 'import json; print(json.load(open("/tmp/hb/device.json"))["deviceCode"])')
poll() {
  curl -s -X POST -H 'Content-Type: application/json' -d "{\"deviceCode\":\"$DEVICE\"}" \
    $API/hopbox.v1.AuthService/PollDeviceAuthorization
}
poll
```

```json
{"state":"DEVICE_AUTHORIZATION_STATE_PENDING","intervalSeconds":5}
```

Poll again at once, and the server tells the CLI to slow down. The interval grows to 10 seconds, and stays there:

```json
{"state":"DEVICE_AUTHORIZATION_STATE_SLOW_DOWN","intervalSeconds":10}
```

The console page looks the code up with the session of the person. Case and the dash do not matter. Then it approves with the start time that it showed:

```sh
browser -d '{"userCode":"qjtl phnv"}' $API/hopbox.v1.AuthService/GetDeviceAuthorization | tee /tmp/hb/shown.json
CREATED=$(python3 -c 'import json; print(json.load(open("/tmp/hb/shown.json"))["createTime"])')
browser -d "{\"userCode\":\"qjtl phnv\",\"createTime\":\"$CREATED\"}" $API/hopbox.v1.AuthService/ApproveDevice
```

```json
{"clientName":"hopbox CLI on user1-laptop","clientIp":"127.0.0.1","createTime":"2026-10-16T17:14:56.701147Z","expireTime":"2026-10-16T17:24:56.701147Z"}
{"clientName":"hopbox CLI on user1-laptop","clientIp":"127.0.0.1","createTime":"2026-10-16T17:14:56.701147Z"}
```

The approval makes a CLI sign-in that can live longer than the session. So someone who takes over an old session cannot make one: an approval after a sign-in older than 10 minutes fails with `permission_denied`. [Run the control role on a laptop](/guide/run-hopboxd/index.html.md#sign-in-the-cli-with-curl) shows the new sign-in that the approval then needs.

Wait 10 seconds, and poll again. The poll gets the CLI tokens, once:

```sh
sleep 10; poll | tee /tmp/hb/tokens.json
```

```json
{"state":"DEVICE_AUTHORIZATION_STATE_APPROVED","intervalSeconds":10,"tokens":{"accessToken":"hbxat_<token>","accessTokenExpireTime":"2026-10-16T18:15:07.860480Z","refreshToken":"hbxrt_<token>","refreshTokenExpireTime":"2026-11-15T17:15:07.860480Z","accountId":"01a145b5-ad29-70a3-9065-fc2498a5d006","cliSignInId":"01a145b5-d8d5-71a9-a7ea-4d5f0c7a43f1"}}
```

A second poll fails with `unauthenticated`, because the device code works once.

### The access token and the refresh

The access token goes in the `Authorization` header. A bearer token needs no `Origin` header:

```sh
ACCESS=$(python3 -c 'import json; print(json.load(open("/tmp/hb/tokens.json"))["tokens"]["accessToken"])')
REFRESH=$(python3 -c 'import json; print(json.load(open("/tmp/hb/tokens.json"))["tokens"]["refreshToken"])')
cli() { curl -s -X POST -H 'Content-Type: application/json' -H "Authorization: Bearer $ACCESS" "$@"; }
cli -d '{}' $API/hopbox.v1.AccountService/GetAccount
```

```json
{"account":{"id":"01a145b5-ad29-70a3-9065-fc2498a5d006","emails":[{"email":"root@example.org"}],"instanceAdmin":true,"personalOrgId":"01a145b5-ad29-70a4-becc-5f6e654b8efa","version":"1","createTime":"2026-10-16T17:14:56.680331Z"}}
```

The access log of `hopboxd` names the caller `cli_token`. Trade the refresh token for new tokens:

```sh
curl -s -X POST -H 'Content-Type: application/json' -d "{\"refreshToken\":\"$REFRESH\"}" \
  $API/hopbox.v1.AuthService/RefreshCLIToken
```

```json
{"tokens":{"accessToken":"hbxat_<token>","accessTokenExpireTime":"2026-10-16T18:15:15.785548Z","refreshToken":"hbxrt_<token>","refreshTokenExpireTime":"2026-11-15T17:15:15.785548Z","accountId":"01a145b5-ad29-70a3-9065-fc2498a5d006","cliSignInId":"01a145b5-d8d5-71a9-a7ea-4d5f0c7a43f1"}}
```

Now send the old refresh token again, as a thief with a copy of it does. Within 10 seconds of the refresh, the old token still works once ([CLI tokens](#cli-tokens)). So wait first:

```sh
sleep 11
curl -s -X POST -H 'Content-Type: application/json' -d "{\"refreshToken\":\"$REFRESH\"}" \
  $API/hopbox.v1.AuthService/RefreshCLIToken
cli -d '{}' $API/hopbox.v1.AccountService/GetAccount
```

```json
{"code":"unauthenticated","message":"the refresh token is not valid: it is unknown, used, or expired. Sign in again"}
{"code":"unauthenticated","message":"the credential is not valid"}
```

The reuse ended the whole CLI sign-in, with every token of it. The audit log has `auth.end_cli_sign_in`.

### An API key

A member key acts as its account in one org. This call makes one in the personal org, for 30 days:

```sh
browser -H 'Idempotency-Key: key-ci-1' -d '{"name":"ci","lifetime":"API_KEY_LIFETIME_30_DAYS"}' \
  $API/hopbox.v1.APIKeyService/CreateAPIKey | tee /tmp/hb/key.json
```

```json
{"key":{"id":"01a145b6-1aaf-7620-989d-f244cbd2aba6","orgId":"01a145b5-ad29-70a4-becc-5f6e654b8efa","kind":"API_KEY_KIND_MEMBER","name":"ci","accountId":"01a145b5-ad29-70a3-9065-fc2498a5d006","creatorAccountId":"01a145b5-ad29-70a3-9065-fc2498a5d006","expireTime":"2026-11-15T17:15:24.718683Z","createTime":"2026-10-16T17:15:24.718683Z"},"secret":"hbx_<key>"}
```

The response holds the only copy of the key. A retry with the same `Idempotency-Key` gets `already_exists`, because the install keeps only a hash of the key. No key can make another key:

```sh
KEY=$(python3 -c 'import json; print(json.load(open("/tmp/hb/key.json"))["secret"])')
curl -s -X POST -H 'Content-Type: application/json' -H "Authorization: Bearer $KEY" \
  -H 'Idempotency-Key: key-child' -d '{"name":"child"}' $API/hopbox.v1.APIKeyService/CreateAPIKey
```

```json
{"code":"permission_denied","message":"permission denied: a caller of kind member_key may not do api_key.create"}
```

## Sweep

The leader deletes these rows every 10 minutes, with the sweep of sign-ins ([hopboxd](/reference/hopboxd/index.html.md#leader-jobs)):

- the expired CLI access tokens and CLI refresh tokens,
- the CLI sign-ins that have no refresh token left that can renew them,
- the device sign-ins and the failed approvals that are older than 24 hours,
- the API keys and admin API keys that expired more than 30 days ago.

A replaced refresh token stays until its 30 days end, so a copy of it that comes back is caught. An expired API key or admin API key stays in the lists for 30 days, so its owner sees why a script stopped. Then the sweep deletes it.

## Audit entries

These calls write audit entries in the transaction of the call ([the sign-in, account, and audit API](/reference/auth-api/index.html.md#audit-entries), [the admin API](/reference/admin-api/index.html.md#adminkeyservice)):

| Action | When | Actor | Org | Target | Details |
| --- | --- | --- | --- | --- | --- |
| `auth.start_device_authorization` | A device flow starts. | the caller, often `anonymous` | none | `device_authorization` | `client_name`, `client_ip` |
| `auth.approve_device` | A person approves a code. | the `session` | the personal org | `device_authorization` | `client_name`, `client_ip` |
| `auth.sign_in` | A poll takes the CLI tokens. | `cli_token`, with the new CLI sign-in | the personal org | `cli_sign_in` | `method` (`device`), `client_name`, `device_sign_in` (the ID of the device authorization) |
| `auth.end_cli_sign_in` | A replaced refresh token came back, or the account ended the sign-in. | the caller of the refresh, often `anonymous`, or the `session` of the account | the personal org | `cli_sign_in` | `reason` (`refresh token reuse` or `ended by the account`) |
| `auth.sign_out` | `SignOut` with a CLI token. | `cli_token` | the personal org | `cli_sign_in` | none |
| `api_key.create` | `CreateAPIKey` | the `session` or `cli_token` | the org of the key | `api_key` | `kind`, `name`, `expire_time` |
| `api_key.revoke` | `RevokeAPIKey` | the `session` or `cli_token` | the org of the key | `api_key` | `kind`, `name` |
| `ssh_key.add` | `AddSSHKey` | the `session` or `cli_token` | the personal org | `ssh_key` | `fingerprint`, `type`, `bits`, `name` |
| `ssh_key.remove` | `RemoveSSHKey` | the `session` or `cli_token` | the personal org | `ssh_key` | `fingerprint`, `type`, `bits`, `name` |
| `machine.ssh` | A sign-in with an SSH key through the SSH front door ([the SSH front door](/reference/ssh-front-door/index.html.md#audit)). | `ssh_key`, with the SSH key as the credential | the org of the machine | `machine` | `name`, `linux_user`, `fingerprint`, `client_ip` |
| `admin_key.create` | `CreateAdminKey` | `local_root` | none | `admin_key` | `name`, `expire_time` |
| `admin_key.register` | `RegisterAdminKey` | `local_root` | none | `admin_key` | `name`, `expire_time`, `new` (`true`, or `false` for a hash that a key had already) |
| `admin_key.list` | `ListAdminKeys` | `local_root` | none | none | none |
| `admin_key.revoke` | `RevokeAdminKey` | `local_root` | none | `admin_key` | `name` |

A refresh of CLI tokens writes no audit entry: it goes on a sign-in that the log records already. In the audit entries of the calls with a credential, the actor kind is the caller kind of the credential. The actor has the account of a person or a member key. Its credential ID is the CLI sign-in, the API key, the admin API key, or the SSH key.
