# Go client

> The Go client of the public API: its options, idempotency keys, retries, typed errors, and credentials.

`internal/client` is the Go client of the public API `hopbox.v1`. The hopbox CLI uses it ([hopbox CLI](/reference/cli/index.html.md)). It wraps the generated Connect clients of `gen/hopbox/v1/hopboxv1connect` with these rules: retries with backoff, idempotency keys on create calls, typed errors, and the bearer credential of the caller. The TypeScript SDK follows the same rules ([TypeScript SDK](/reference/ts-sdk/index.html.md)). The package links no control code. The test `TestCLINeverLinksControlCode` of `tools/rolesplit` checks this for `internal/client`, `internal/cli`, and `cmd/hopbox`.

## Make a client

```go
c, err := client.New("https://example.com", client.Options{
	Credentials: client.BearerToken(os.Getenv("HOPBOX_API_KEY")),
})
if err != nil {
	return err
}
resp, err := c.Orgs.ListOrgs(ctx, connect.NewRequest(&hopboxv1.ListOrgsRequest{}))
if errors.Is(err, client.ErrPermissionDenied) {
	// The key cannot list orgs.
}
```

`New(rawURL, opts)` checks the URL with `ParseURL` and returns a `*Client`. A bad URL fails with an error that wraps `ErrBadURL`.

| `Options` field | Type | Default | Description |
| --- | --- | --- | --- |
| `HTTPClient` | `*http.Client` | `DefaultHTTPClient()` | Sends the calls. The default speaks HTTP/2 over TLS, with HTTP/1.1 for a server without HTTP/2. To a loopback install without TLS, it speaks HTTP/2 with prior knowledge (h2c), as `hopboxd` does, because `Exec` needs a stream in both directions. A loopback server that speaks only HTTP/1.1 then fails the call. It follows no redirect, so the bearer token never goes to another host: a redirect fails the call. A client of your own must do the same. |
| `Credentials` | `Credentials` | nil | Gives the bearer token of each call. With nil, the calls carry no credential, as the device flow needs. |
| `UserAgent` | string | `hopbox-go-client (<os>/<arch>)` | The `User-Agent` header of each call. The CLI sends `hopbox-cli/<version> (<os>/<arch>)`. |
| `AttemptTimeout` | `time.Duration` | `DefaultAttemptTimeout`, 90 s | The deadline of one try of a unary call. The server gives a call 30 s, and a lifecycle call of a machine 60 s. |
| `MaxRetryAfter` | `time.Duration` | 0, no cap | The longest `Retry-After` that the client waits for. A longer one ends the call with its error, and `Error.RetryAfter` holds the wanted wait. With 0, the client waits as long as the server asks. The CLI sets 5 minutes. |
| `Logger` | `*slog.Logger` | no logs | Gets a debug record for each retry, and a record for each refresh of CLI tokens. |

`Client` has the field `BaseURL`, the URL of `ParseURL`. It has one generated client for each service of `hopbox.v1`: `Install`, `Auth`, `Accounts`, `Orgs`, `APIKeys`, `SSHKeys`, `Machines`, `Vars`, and `Audit`. Each call of each one goes through the steps below.

## Install URLs

`ParseURL(raw)` returns the base URL of an install, such as `https://example.com`.

- A bare domain gets `https://`.
- The host is in lower case, and a `/` at the end goes.
- `http://` works only for a loopback host: `localhost`, `127.0.0.0/8`, or `::1`.
- A path, a query, a fragment, a user name, or a password fails with `ErrBadURL`.

The error of a refused URL hides its user info. It shows `xxxxx` in place of all text between `://` and the last `@`, such as `https://xxxxx@example.com`. A password with a `/` or a `?` in it stays hidden too. A URL without an `@` shows as it is.

## The steps of a call

Each unary call passes these steps in order:

1. **Typed errors.** The error of the call becomes an `*Error`. See [Errors](#errors).
2. **User agent.** The step sets `User-Agent`.
3. **Idempotency key.** See [Idempotency keys](#idempotency-keys).
4. **Retries.** See [Retries](#retries). The steps after this one run again on each try.
5. **Credential.** See [Credentials](#credentials).

A stream gets the steps 1, 2, and 5, without retries. If the credential fails, the first `Send` or `Receive` of the stream returns the error, and the stream sends no request.

## Idempotency keys

A method with the option `(hopbox.v1.idempotent)` needs an `Idempotency-Key` header ([the API server](/reference/api/index.html.md#idempotency-keys)). Today these are `CreateMachine`, `CreateOrg`, `CreateAPIKey`, and `AddSSHKey`. The client puts a new UUIDv7 in the header of each such call that has no key yet. The step runs before the retries, so each try of one call sends the same key. The server then runs the call once. A key that the caller set stays.

## Retries

The client tries a unary call at most 3 times: the first try and `MaxRetries`, 2, retries.

| Code of the failed try | Retried |
| --- | --- |
| `unavailable` | Only for a call that is safe to run again. |
| `resource_exhausted` | Only for a call that is safe to run again. |
| `deadline_exceeded` | Only for a call that is safe to run again. |
| Any other code | No. An auth error or a bad request never gets a retry. |

A call is safe to run again in these cases:

| Case | How the client knows it |
| --- | --- |
| The call only reads. | Its method has the option `(hopbox.v1.read_only)`, or `idempotency_level` `NO_SIDE_EFFECTS` or `IDEMPOTENT` ([API definitions](/reference/api-definitions/index.html.md#hopboxv1read_only)). Examples are `ListOrgs` and `GetMachine`. |
| The call carries an idempotency key. | Its method has the option `(hopbox.v1.idempotent)`, such as `CreateMachine` ([Idempotency keys](#idempotency-keys)). |
| The try surely did not reach the server. | The try got no connection, because of a dial or TLS error, such as a refused connection. Or the answer of the try has the header `NotRunHeader`, `Hopbox-Not-Run`, with the value `1` ([the API server](/reference/api/index.html.md#answers-of-calls-that-changed-nothing)). |

Each other call changes state and has no key, such as `CreateInvitation`. The client does not retry it after a lost answer, because the server can have run it. A second run can then act twice, for example with a second invitation. The client retries such a call only after a try that surely did not reach the server. The check holds for each try. A refusal with `Hopbox-Not-Run` on the first try, then a lost answer on the second try, ends the call with the error of the second try.

A try that ends without an answer can have run. A reset of an HTTP/2 stream, such as a server that drops the call, fails with `internal`, which never gets a retry. A try that waits for an answer until `AttemptTimeout` fails with `deadline_exceeded`. Only a call that is safe to run again then goes again.

`RefreshCLIToken` from a `TokenSource` is the one exception, as the owner decided on 2026-10-25. The refresh changes state and has no key. But the install takes a replaced refresh token once more within its `RefreshGrace` of 10 s ([CLI tokens, API keys, admin API keys, and SSH keys](/reference/credentials/index.html.md#cli-tokens)). So after a lost answer, the client sends the refresh once more, if that retry starts within `RefreshReplayWindow`, 10 s, of the start of the first try. The wait before the retry counts. The same bound holds for every later try of the refresh after the lost answer, also a retry after a refusal with `Hopbox-Not-Run`, because each such try uses the one replay of the grace. The client gives up rather than start any of them later. Only `RefreshCLIToken` gets this retry, whatever the context of another call holds. The window starts before the install replaces the token, so a retry that reaches the install soon after its start falls within the grace. A retry that the network holds back past the grace counts as a reuse, and ends the CLI sign-in. A second lost answer fails the call. The other rules of the table still apply, so a refusal with `Hopbox-Not-Run` before the lost answer does not use up the one retry.

The client reads `Hopbox-Not-Run` from the HTTP answer, so it also sees the header on an answer that is not a Connect error, such as the `502` page of an edge server. For this, `New` wraps the transport of `Options.HTTPClient` in a copy of that client. The wrapper also gives each HTTP request a copy of the header of the call. So a try that the attempt timeout ended can still be in the transport while the next try sets its header. The wrapper looks at each HTTP request of a try, and only the last one counts. The credential step can send a call twice in one try: after `unauthenticated`, it sends the call again with a new token. So a refusal with `Hopbox-Not-Run` before a request that ran does not let the call go again.

The client knows that a try got no connection from two signs. A `ConnectDone` event of `net/http/httptrace` reports a failed dial, before any `GotConn` event. Or the error is one that only a dial or a TLS handshake returns: a dial `*net.OpError`, a `*net.DNSError`, a `tls.RecordHeaderError`, or a `*tls.CertificateVerificationError`. Any other error counts as a try that can have run. So a transport of your own that reports no such events never makes a lost answer look like a request that never left. The edge servers use the same check (`internal/notrun`).

The wait before a retry has full jitter: a random time from 0 up to 500 ms before the first retry, and up to 1 s before the second. If the error has a `Retry-After` header, the client waits that time instead. The header holds seconds, or an HTTP date. The client reads at most `MaxRetryAfterHeader`, 24 hours, from either form, so a date far ahead counts as 24 hours. A wait of `LongWait`, 10 s, or more logs `retrying a call` at level warn. A shorter one logs it at level debug. The client does not retry in these cases:

- `MaxRetryAfter` is set, and `Retry-After` is longer.
- The wait ends after the deadline of the context of the call.
- The context of the call ends.
- The credential step failed, also for a read call. An example is a refresh of CLI tokens that failed after its own retries. One more refresh with the same token would end the CLI sign-in.

Each try has its own deadline, `AttemptTimeout`. A try that runs out of time fails with `deadline_exceeded`, and the table above applies.

## Errors

Each error of a call is an `*Error`. Other errors, such as one of the credential, stay as they are.

| `Error` field | Type | Description |
| --- | --- | --- |
| `Code` | `connect.Code` | The code, such as `connect.CodeNotFound`. |
| `Message` | string | The message of the server, without the code. |
| `Procedure` | string | The call, such as `/hopbox.v1.OrgService/GetOrg`. |
| `RequestID` | string | The `Hopbox-Request-Id` of the response ([the API server](/reference/api/index.html.md#the-chain-of-a-call)). A call that did not reach the server has an empty ID. |
| `RetryAfter` | `time.Duration` | The `Retry-After` of the response, or 0. |
| `Details` | `[]proto.Message` | The typed details of the error, such as `*hopboxv1.MachineNameTaken`. |

`Error()` returns `<code>: <message>`, such as `not_found: not found: no org 01a1...`. `Unwrap` returns the `*connect.Error`, so `connect.CodeOf` works too.

`errors.Is(err, sentinel)` matches the code of an `*Error`. The codes are those of [the API server](/reference/api/index.html.md#errors).

| Sentinel | Code |
| --- | --- |
| `ErrCanceled` | `canceled` |
| `ErrUnknown` | `unknown` |
| `ErrInvalidArgument` | `invalid_argument` |
| `ErrDeadlineExceeded` | `deadline_exceeded` |
| `ErrNotFound` | `not_found` |
| `ErrAlreadyExists` | `already_exists` |
| `ErrPermissionDenied` | `permission_denied` |
| `ErrResourceExhausted` | `resource_exhausted` |
| `ErrFailedPrecondition` | `failed_precondition` |
| `ErrAborted` | `aborted` |
| `ErrUnimplemented` | `unimplemented` |
| `ErrInternal` | `internal` |
| `ErrUnavailable` | `unavailable` |
| `ErrUnauthenticated` | `unauthenticated` |

| Other error | Meaning |
| --- | --- |
| `ErrSignedOut` | The client has no live CLI sign-in: none in the store, the refresh token ended, the tokens belong to another install, or the install refused the refresh token. The way out is a new sign-in. |
| `ErrBadURL` | `ParseURL` or `New` refused the URL. |

## Credentials

```go
type Credentials interface {
	Token(ctx context.Context) (string, error)
}

type Renewer interface {
	Credentials
	Rejected(token string) bool
}
```

The client puts the token of `Credentials` in the header `Authorization: Bearer <token>` of each try. If the credential fails, the call fails with that error, and no request goes out.

Sometimes the server refuses the token with `unauthenticated`, and the credential is a `Renewer`. Then the client calls `Rejected` with the token. If `Rejected` returns true, the client gets a token again. If the new token differs, the client sends the call once more with it. If getting the token fails with `ErrSignedOut`, the call returns that error.

`BearerToken` is a token that never changes, such as an API key. An empty `BearerToken` fails with `ErrSignedOut`.

### CLI tokens

`TokenSource` is a `Renewer` for the tokens of a CLI sign-in ([CLI tokens, API keys, admin API keys, and SSH keys](/reference/credentials/index.html.md#cli-tokens)). `NewTokenSource(rawURL, store, opts)` makes one. It refreshes with its own client of `opts`, without a credential, because `RefreshCLIToken` must not carry an expired access token.

```go
type TokenStore interface {
	Lock(ctx context.Context) (unlock func(), err error)
	Load() (CLITokens, error)
	Save(CLITokens) error
	ClearIf(refreshToken string) error
}
```

The store keeps the tokens where every process of the CLI finds them. `Lock` holds off the `Lock` of every other process. With no tokens in the store, `Load` returns an error that wraps `ErrSignedOut`. The CLI keeps them in `credentials.toml` ([hopbox CLI](/reference/cli/index.html.md#files)).

`CLITokens` has the fields `URL`, `AccountID`, `SignInID`, `AccessToken`, `AccessExpireTime`, `RefreshToken`, and `RefreshExpireTime`. `URL` is the install that issued the tokens. `TokensFrom(url, tokens)` makes `CLITokens` from the `CLITokens` message of the API.

`Token` works in these steps:

1. If the token in memory lives for more than `RefreshBefore`, 1 minute, and the server did not refuse it, return it.
2. Take the lock of the store, and load the tokens.
3. If the tokens belong to another install, fail with `ErrSignedOut`.
4. If the stored access token lives for more than 1 minute, and the server did not refuse it, keep it and return it. Another process refreshed meanwhile.
5. If the refresh token ended, fail with `ErrSignedOut`.
6. Call `RefreshCLIToken` with the stored refresh token. After a lost answer, the call goes once more within `RefreshReplayWindow` ([Retries](#retries)). If that answer gets lost too, fail, and keep the store as it is. The call waits at most `RefreshMaxRetryAfter`, 30 s, for a `Retry-After`. A shorter `MaxRetryAfter` of the options wins. The refresh holds the lock of the store, and every other process of the CLI waits for that lock. If the server refuses it, call `ClearIf` with that refresh token, and fail with `ErrSignedOut`. If the store still holds that refresh token, `ClearIf` deletes the stored tokens. Else it keeps them.
7. Save the new tokens, and return the new access token. If the save fails, call `ClearIf` with the old refresh token, log `the refreshed CLI tokens could not be stored, so the next command must sign in again` at level warn, and still return the token.

The install replaces the refresh token before step 7 saves the new one. Two cases leave the old refresh token in the store: the save and the `ClearIf` both fail, or the process dies between steps 6 and 7. The next refresh sends it again, and the install ends the sign-in as a reuse. The steps make this unlikely, but they cannot rule it out.

`Rejected(token)` forgets the token in memory, so the next `Token` reads the store again. Only a store that holds the same refused token leads to a refresh.

A replaced refresh token ends the sign-in, so these steps never send one. The store must have a real lock. The grace of 10 s of the install covers only one second refresh with the same token. So three processes that refresh at once without a lock end the sign-in. The store of the CLI refuses a dir without file locks ([hopbox CLI](/reference/cli/index.html.md#files)). Two processes without a lock both get tokens through the grace, and the next refresh drops the second chain. Its process then reads the store again at the next refusal.

## Tests

- `internal/client/client_test.go` checks the retries, the waits, `Retry-After`, the idempotency keys, the typed errors, and the credentials against a fake API. Its table of retries covers a read call, a call with a key, and `CreateInvitation`, after a lost answer, an error code, an answer with `Hopbox-Not-Run`, and a refused connection. Other tests cover a reset stream, a hung answer, a resend with a new token after a refusal, a transport without connection events, and TLS with HTTP/2. `TestARefreshReplaysOnceWithinTheWindow` checks the one retry of a refresh, with a clock of the test, before and after the window.
- `internal/client/tokens_test.go` checks `TokenSource` with a store in memory.
- `internal/client/server_test.go` runs against a real install of `internal/api/apitest`. Four processes with one store make one refresh. Two processes without a lock both refresh through the grace, and the sign-in goes on after the next refresh. This shows the most that the grace covers. A sign-in that the console ends clears the store. One lost answer of a refresh recovers with no reuse, and the sign-in goes on. Two lost answers fail the call and keep the store as it is.
