# TypeScript SDK

> The TypeScript SDK @hopbox/sdk for Node.js and browsers: its options, retries, idempotency keys, typed errors, credentials, exec, file copy, and tests.

`@hopbox/sdk` in `web/packages/sdk` is the TypeScript SDK of the public API `hopbox.v1`. It runs in Node.js and in a browser. It is the generated code of protoc-gen-es plus a small hand-written facade. The facade follows the rules of the Go client ([Go client](/reference/go-client/index.html.md)): retries with backoff, idempotency keys on create calls, typed errors, and the bearer credential of the caller. The console uses it. [Call the API from TypeScript](/guide/typescript-sdk/index.html.md) shows a script that uses it.

The SDK never holds the admin API. `buf.gen.yaml` keeps `hopbox.admin.v1` out of its generated code ([API definitions](/reference/api-definitions/index.html.md#generators-and-outputs)).

## Package

| Item | Value |
| --- | --- |
| Name | `@hopbox/sdk` |
| Folder | `web/packages/sdk` |
| Published | No. The package is `private`. The owner decides on the npm scope `@hopbox` before a release to npm (plan item 13). The console takes the package from the pnpm workspace. |
| Module format | ES modules only (`"type": "module"`) |
| Node.js | 24 or later, the version that CI tests |
| Runtime dependencies | `@bufbuild/protobuf` 2.16.0, `@connectrpc/connect` 2.2.0, `@connectrpc/connect-node` 2.2.0, `@connectrpc/connect-web` 2.2.0, all pinned to an exact version |
| Build output | `dist/`, which git ignores. Build it with `corepack pnpm --filter @hopbox/sdk build`. |

### Entry points

| Import | Condition | File | What it holds |
| --- | --- | --- | --- |
| `@hopbox/sdk` | `browser` | `dist/browser.js` | `createClient` for a browser, and every export of the facade |
| `@hopbox/sdk` | `node` | `dist/node.js` | `createClient` for Node.js, and every export of the facade |
| `@hopbox/sdk` | `default` | `dist/browser.js` | the same as `browser` |
| `@hopbox/sdk/v1` | none | `dist/v1.js` | every generated message, enum, service, and schema of `hopbox.v1`, such as `MachineState` and `MachineService` |

Node.js picks the `node` condition, and a bundler for the browser, such as Vite, picks `browser`. Both entry points export the same names with the same types: the names on this page. A test checks it.

| Runtime | Unary calls and server streams | `Exec` | `Upload` |
| --- | --- | --- | --- |
| Node.js | Connect protocol, binary protobuf, over HTTP/2. Plain `http://` to a loopback install starts HTTP/2 with prior knowledge. | the bidi stream `Exec` over HTTP/2 by default, or the WebSocket | yes |
| Browser | Connect protocol, binary protobuf, `POST` requests over `fetch` | the WebSocket `/api/ws/exec` only | no. A browser cannot stream a request body. |

The browser client passes the check of the session cookie ([Sessions, cookies, and sign-in links](/reference/sessions/index.html.md#calls-with-the-cookie)). Each call is a Connect `POST` request with `Connect-Protocol-Version: 1`, and the browser adds the `Origin` header itself. The client never uses a Connect `GET` request.

## Make a client

```ts
import { bearerToken, createClient, NotFoundError } from "@hopbox/sdk";

const hb = createClient({
  url: "https://example.com",
  credentials: bearerToken(process.env.HOPBOX_API_KEY ?? ""),
});
try {
  const { machine } = await hb.machines.getMachine({ machine: { case: "name", value: "box1" } });
  console.log(machine?.state);
} catch (err) {
  if (err instanceof NotFoundError) {
    console.log("no machine box1");
  }
} finally {
  hb.close();
}
```

`createClient(options)` checks `options.url` with `parseURL` and returns a `HopboxClient`. A bad URL throws a `BadURLError`. A browser client with `execTransport: "stream"` throws an `Error`.

| `ClientOptions` field | Type | Default | Description |
| --- | --- | --- | --- |
| `url` | string | required | The URL of the install, such as `https://example.com` ([Install URLs](#install-urls)). |
| `credentials` | `Credentials` | none | Gives the bearer token of each call ([Credentials](#credentials)). Without it, the calls carry no `Authorization` header. The console leaves it out, so the session cookie of the browser authenticates each call. The device flow leaves it out too. |
| `userAgent` | string | `hopbox-ts-client (<platform>/<arch>)` in Node.js, such as `hopbox-ts-client (darwin/arm64)` | The `User-Agent` header of each call. A browser sends its own and ignores this field. |
| `attemptTimeoutMs` | number | `DEFAULT_ATTEMPT_TIMEOUT_MS`, 90000 | The deadline of one try of a unary call, in milliseconds. The server gives a call 30 s, and a lifecycle call of a machine 60 s. |
| `maxRetryAfterMs` | number | none, no cap | The longest `Retry-After` that the client waits for, in milliseconds. A longer one ends the call with its error, and `retryAfterMs` of the error holds the wanted wait. Without it, the client waits as long as the server asks, up to 24 hours. |
| `logger` | `Logger` | no logs | Gets a record for each retry ([Retries](#retries)). `console` works. |
| `execTransport` | `"stream"` or `"websocket"` | `"stream"` in Node.js, `"websocket"` in a browser | How `exec` reaches the machine ([Exec](#exec)). |
| `fetch` | `typeof fetch` | `globalThis.fetch` | The `fetch` function of a browser client. Node.js ignores it. |

`Logger` has two methods, `debug(message, fields)` and `warn(message, fields)`.

### `HopboxClient`

| Field or method | Type | Description |
| --- | --- | --- |
| `baseUrl` | string | The URL of the install, as `parseURL` returned it. |
| `transport` | `Transport` | The Connect transport of the client, with steps 2 to 5 of [The steps of a call](#the-steps-of-a-call). A tool such as connect-query takes it, so its calls get the same retries, keys, and credential ([The transport](#the-transport)). |
| `install` | `Client<typeof InstallService>` | `GetInstallInfo` ([the API server](/reference/api/index.html.md#installservice)) |
| `auth` | `Client<typeof AuthService>` | sign-in links, the device flow, and CLI tokens ([the sign-in, account, and audit API](/reference/auth-api/index.html.md)) |
| `account` | `Client<typeof AccountService>` | the account of the caller ([the sign-in, account, and audit API](/reference/auth-api/index.html.md)) |
| `audit` | `Client<typeof AuditService>` | the audit log ([the sign-in, account, and audit API](/reference/auth-api/index.html.md)) |
| `orgs` | `Client<typeof OrgService>` | orgs, members, and invitations ([the org API](/reference/org-api/index.html.md)) |
| `apiKeys` | `Client<typeof APIKeyService>` | API keys ([the sign-in, account, and audit API](/reference/auth-api/index.html.md)) |
| `sshKeys` | `Client<typeof SSHKeyService>` | SSH keys ([the sign-in, account, and audit API](/reference/auth-api/index.html.md)) |
| `machines` | `Client<typeof MachineService>` | machines, their lifecycle, and the access calls ([the machine API](/reference/machine-api/index.html.md)) |
| `vars` | `Client<typeof VarService>` | env vars and secrets ([the env var API](/reference/vars-api/index.html.md)) |
| `exec(start, options?)` | `ExecSession` | Runs a command in a running machine ([Exec](#exec)). |
| `upload(start, data, options?)` | `Promise<UploadResponse>` | Writes a file, or a tar stream into a directory, in a running machine ([Upload](#upload)). Node.js only. |
| `download(request, options?)` | `Promise<Download>` | Reads a file, or a directory as a tar stream, from a running machine ([Download](#download)). |
| `close()` | `void` | Ends the HTTP/2 connection of a Node.js client. A later call opens a new one. In a browser, it does nothing. |

Each service field is the generated Connect client of that service, `Client<S>` of `@connectrpc/connect`. A method takes the request message as a plain object, and an optional `CallOptions` of Connect: `timeoutMs`, `headers`, `signal`, `onHeader`, `onTrailer`, and `contextValues`. A unary method returns a `Promise` of the response message. A server stream, such as `watchMachines` or `streamMachineLogs`, returns an `AsyncIterable` of messages. Each method name is the name of the RPC with a small first letter, such as `listMachines` for `ListMachines`.

`timeoutMs` and an `AbortSignal.timeout()` in `signal` both end a call in time. Only `timeoutMs` also sends the deadline to the server in the header `Connect-Timeout-Ms`, so the server stops its work too. A unary call without `timeoutMs` has no deadline of its own. Then each try has the deadline `attemptTimeoutMs`. The helpers `exec`, `upload`, and `download` set no deadline, because a stream can run for hours.

### The transport

`transport` is the Connect transport under the service fields. Its calls get the user agent, the idempotency keys, the retries, and the credential. They do not get the typed errors, because that step wraps the service fields. So a call through the transport throws the `ConnectError` of `@connectrpc/connect`. The console gives it to connect-query:

```tsx
import { TransportProvider } from "@connectrpc/connect-query";

<TransportProvider transport={hb.transport}>{children}</TransportProvider>
```

The transport already retries a call ([Retries](#retries)). So set `retry: false` in the options of TanStack Query, or the query tries each call up to 3 times more on top:

```ts
const queryClient = new QueryClient({ defaultOptions: { queries: { retry: false }, mutations: { retry: false } } });
```

## Install URLs

`parseURL(raw)` returns the base URL of an install, such as `https://example.com`. It follows `ParseURL` of the Go client:

- A bare domain gets `https://`.
- The host is in lower case, and a `/` at the end goes. A default port goes too.
- `http://` works only for a loopback host: `localhost`, `127.0.0.0/8`, or `[::1]`.
- A path, a query, a fragment, a user name, or a password throws a `BadURLError`.

The message of a `BadURLError` hides the user info of the URL. It shows `xxxxx` in place of all text between `://` and the last `@`, such as `https://xxxxx@example.com`. A password with a `/` or a `?` in it stays hidden too. A URL without an `@` shows as it is.

## The steps of a call

Each unary call passes these steps in order:

1. **Typed errors.** The error of the call becomes a `HopboxError` ([Errors](#errors)).
2. **User agent.** The step sets `User-Agent`, in Node.js only.
3. **Idempotency key.** See [Idempotency keys](#idempotency-keys).
4. **Retries.** See [Retries](#retries). The steps after this one run again on each try.
5. **Credential.** See [Credentials](#credentials).

A stream gets the steps 1, 2, and 5, without retries. The typed errors also cover each message of a stream.

## Idempotency keys

A method with the option `(hopbox.v1.idempotent)` needs an `Idempotency-Key` header ([the API server](/reference/api/index.html.md#idempotency-keys)). Today these are `CreateMachine`, `CreateOrg`, `CreateAPIKey`, and `AddSSHKey`. The client puts a new UUIDv7 in the header of each such call that has no key yet. The step runs before the retries, so each try of one call sends the same key. A key that the caller sets in `CallOptions.headers` stays:

```ts
await hb.machines.createMachine({ name: "box1" }, { headers: { "Idempotency-Key": "deploy-42" } });
```

A replay of a known key returns the first result. Its response has the header `Idempotent-Replayed: true`, which `onHeader` can read.

## Retries

The client tries a unary call at most 3 times: the first try and `MAX_RETRIES`, 2, retries.

| Code of the failed try | Retried |
| --- | --- |
| `unavailable` | Only for a call that is safe to run again. |
| `resource_exhausted` | Only for a call that is safe to run again. |
| `deadline_exceeded` | Only for a call that is safe to run again. |
| Any other code | No. An auth error or a bad request never gets a retry. |

A call is safe to run again in these cases, as for the Go client ([Go client](/reference/go-client/index.html.md#retries)):

| Case | How the client knows it |
| --- | --- |
| The call only reads. | Its method has the option `(hopbox.v1.read_only)`, or `idempotency_level` `NO_SIDE_EFFECTS` or `IDEMPOTENT` ([API definitions](/reference/api-definitions/index.html.md#hopboxv1read_only)). Examples are `listOrgs` and `getMachine`. |
| The call carries an idempotency key. | Its method has the option `(hopbox.v1.idempotent)`, such as `createMachine` ([Idempotency keys](#idempotency-keys)). |
| The try surely did not reach the server. | The answer of the try has the header `NOT_RUN_HEADER`, `Hopbox-Not-Run`, with the value `1` ([the API server](/reference/api/index.html.md#answers-of-calls-that-changed-nothing)). In Node.js, the try also counts when it got no connection: the cause of its error has the code `ECONNREFUSED`, `ENOTFOUND`, or `EAI_AGAIN`. |

Each other call changes state and has no key, such as `createInvitation`. The client does not retry it after a lost answer, because the server can have run it. A second run can then act twice, for example with a second invitation. The check holds for each try. A refusal with `Hopbox-Not-Run` on the first try, then a lost answer on the second try, ends the call with the error of the second try.

A network error fails a call with `unavailable`. In a browser, `fetch` then rejects with a `TypeError`, both for a connection that never opened and for a lost answer. So a browser client retries a change without a key only after an answer with `Hopbox-Not-Run`. A read call and a call with a key get their retries in both runtimes. A timeout of the connection, `ETIMEDOUT` in Node.js, can come after the request went out, so it does not count as a try that did not reach the server.

The wait before a retry has full jitter: a random time from 0 up to 500 ms before the first retry, and up to 1 s before the second (`BACKOFF_MS`). If the error has a `Retry-After` header, the client waits that time instead. The client reads the header as the Go client does. It takes a count of seconds that fits in a signed 64-bit number, such as `5` or `+5`. It also takes an HTTP date in one of the three forms of RFC 9110: `Wed, 21 Oct 2026 12:00:10 GMT`, `Wednesday, 21-Oct-26 12:00:10 GMT`, or `Wed Oct 21 12:00:10 2026`. Any other value counts as no header. The client reads at most `MAX_RETRY_AFTER_HEADER_MS`, 24 hours, from either form. The client calls `logger.debug("retrying a call", fields)` before each retry. A wait of `LONG_WAIT_MS`, 10 s, or more calls `logger.warn` instead. `fields` has `procedure`, `code`, `retry`, and `waitMs`. `code` is the name of the code, such as `unavailable`.

The client does not retry in these cases:

- `maxRetryAfterMs` is set, and `Retry-After` is longer.
- The wait ends after the deadline of the call, from `timeoutMs`.
- The `signal` of the call aborts.

Each try has its own deadline. It is `attemptTimeoutMs`, or the rest of the deadline of the call, whichever ends first. The client sends it in the header `Connect-Timeout-Ms`. A try that runs out of time fails with `deadline_exceeded`, and the table above applies.

## Errors

Each error of a call is a `HopboxError`, or one of its subclasses. The errors of the SDK itself, `SignedOutError` and `BadURLError`, stay as they are.

| `HopboxError` field | Type | Description |
| --- | --- | --- |
| `code` | `Code` | The Connect code, such as `Code.NotFound`. The SDK exports `Code`. |
| `codeName` | string | The name of the code, such as `not_found`. |
| `message` | string | `<code name>: <message of the server>`, such as `not_found: not found: no machine has the name "box9"`. |
| `rawMessage` | string | The message of the server, without the code. |
| `procedure` | string | The call, such as `/hopbox.v1.MachineService/GetMachine`. For the WebSocket of exec, it is `/api/ws/exec`. |
| `requestId` | string | The `Hopbox-Request-Id` of the response ([the API server](/reference/api/index.html.md#the-chain-of-a-call)). For a call that did not reach the server, it is empty. For an error of the WebSocket, it is empty too, because a browser cannot read the headers of a WebSocket. |
| `retryAfterMs` | number | The `Retry-After` of the response in milliseconds, or 0. |
| `details` | `Message[]` | The typed details of the error, decoded with the messages of `hopbox.v1`, such as a `MachineNameTaken` with `$typeName` `hopbox.v1.MachineNameTaken`. A detail of an unknown type is left out. |
| `cause` | unknown | The `ConnectError` of `@connectrpc/connect`, or undefined. |

`instanceof` tells the kinds apart. The codes are those of [the API server](/reference/api/index.html.md#errors).

| Class | Code |
| --- | --- |
| `CanceledError` | `canceled` |
| `UnknownError` | `unknown` |
| `InvalidArgumentError` | `invalid_argument` |
| `DeadlineExceededError` | `deadline_exceeded` |
| `NotFoundError` | `not_found` |
| `AlreadyExistsError` | `already_exists` |
| `PermissionDeniedError` | `permission_denied` |
| `ResourceExhaustedError` | `resource_exhausted` |
| `FailedPreconditionError` | `failed_precondition` |
| `AbortedError` | `aborted` |
| `UnimplementedError` | `unimplemented` |
| `InternalError` | `internal` |
| `UnavailableError` | `unavailable` |
| `UnauthenticatedError` | `unauthenticated` |

Another code, such as `out_of_range` or `data_loss`, gives a plain `HopboxError`. hopboxd sends neither.

| Other error | Meaning |
| --- | --- |
| `SignedOutError` | The credential has no token, for example `bearerToken("")`. No request goes out. |
| `BadURLError` | `parseURL` or `createClient` refused the URL. |

The SDK makes some errors itself:

| Error | When |
| --- | --- |
| `UnknownError`, `the exec ended without its exit chunk` | An exec ended with no `exit` chunk. |
| `UnknownError`, `the download did not start with its info` | The first message of a download held no `info`. |
| `UnknownError`, `the WebSocket of exec sent a text message` | The WebSocket sent a text message. The client then closes the socket. |
| `FailedPreconditionError`, `the exec ended, so it takes no more input` | A write to an exec after its end. |
| `CanceledError`, `the exec was canceled` | `cancel()`, a `break` out of the `for await` loop, or an abort of the `signal` of `ExecOptions`. This holds also before the WebSocket opens. |
| `ResourceExhaustedError`, `the reader of the exec fell more than 16777216 bytes behind the WebSocket` | More than 16 MiB of output of the WebSocket waits for the reader. The client then closes the socket. |
| `UnimplementedError`, `a browser cannot stream an upload. ...` | `upload` in a browser. |

## Credentials

```ts
interface Credentials {
  token(signal: AbortSignal): string | Promise<string>;
  rejected?(token: string): boolean | Promise<boolean>;
}
```

The client puts the token of `credentials` in the header `Authorization: Bearer <token>` of each try, and of each stream. If `token` throws, the call fails with that error, and no request goes out.

Sometimes the server refuses the token of a unary call with `unauthenticated`, and the credential has `rejected`. Then the client calls `rejected` with the token. If `rejected` returns true, the client gets a token again. If the new token differs, the client sends the call once more with it. If getting the token throws a `SignedOutError`, the call fails with that error. A credential with `rejected` fits a token that a program renews, such as a CLI access token. A stream gets no second try.

`bearerToken(token)` is a token that never changes, such as an API key in the form `hbx_<key>` ([CLI tokens, API keys, admin API keys, and SSH keys](/reference/credentials/index.html.md)). An empty token throws a `SignedOutError`.

The SDK does not refresh CLI tokens. The hopbox CLI keeps them in its own files ([hopbox CLI](/reference/cli/index.html.md#files)).

## Machine names

A request names a machine in the oneof field `machine`, with `id` or `name` ([the machine API](/reference/machine-api/index.html.md#naming-a-machine)). Two helpers make the value of the field:

| Function | Returns |
| --- | --- |
| `byName(name)` | `{ case: "name", value: name }` |
| `byId(id)` | `{ case: "id", value: id }` |

```ts
await hb.machines.stopMachine({ machine: byName("box1") });
```

## Exec

`exec(start, options?)` runs a command in a running machine, with an optional terminal. `start` is an `ExecStart` as a plain object ([the machine API](/reference/machine-api/index.html.md#exec)): `machine`, `argv`, `env`, `user`, `dir`, `tty`, `size`, `term`, and `timeout`. The session starts at once and returns an `ExecSession`.

| Transport | Used by | Credential | Wire |
| --- | --- | --- | --- |
| `"stream"` | Node.js, by default | the `Authorization` header | the bidi stream `/hopbox.v1.MachineService/Exec` over HTTP/2 |
| `"websocket"` | a browser, always, and Node.js with `execTransport: "websocket"` | the session cookie in a browser. In Node.js, the `Authorization` header of `credentials`. | `GET /api/ws/exec`, one `ExecChunk` in each binary message ([the machine API](/reference/machine-api/index.html.md#the-websocket-endpoint)) |

A browser cannot add a header to a WebSocket. So in a browser, the WebSocket carries only the session cookie, also with `credentials` set. The page must have the origin of the install.

| `ExecOptions` field | Type | Description |
| --- | --- | --- |
| `stdin` | `Uint8Array`, string, or `AsyncIterable<Uint8Array>` | Input for the command. The session sends it, then the end of the input. Without it, send input with `write` and `closeStdin`. |
| `signal` | `AbortSignal` | An abort of this signal ends the exec and the command, as `cancel()` does. |

| `ExecSession` member | Description |
| --- | --- |
| `write(data)` | Sends input, a `Uint8Array` or a string in UTF-8. The session splits it into `stdin` chunks of at most 256 KiB (`MAX_STDIN_CHUNK`). The promise resolves after the transport took the input. |
| `closeStdin()` | Sends `stdin_eof`. With a terminal, the command reads Ctrl-D. |
| `resize(rows, cols)` | Sends a new terminal size. |
| `sendSignal(name)` | Sends a signal, such as `SIGINT`, to the process group of the command. |
| `cancel()` | Ends the exec and the command. A read then throws a `CanceledError`. |
| `output()` | Reads the whole output, and returns `{ stdout, stderr, exit }`. `stdout` and `stderr` are `Uint8Array`. `exit` is the `ExecExit`: `code`, `signal`, `error`, and `timedOut`. |
| `[Symbol.asyncIterator]()` | Reads the output one `ExecEvent` at a time, with `for await`. |

`ExecEvent` is one of these:

| `kind` | Other field | Description |
| --- | --- | --- |
| `"stdout"` | `data: Uint8Array` | Output of the command. With a terminal, all output arrives as `stdout`. |
| `"stderr"` | `data: Uint8Array` | Error output of the command. |
| `"exit"` | `exit: ExecExit` | How the command ended. It is the last event. |

Read the output of a session once, with `for await` or with `output()`. A second read throws an `Error`. A `break` out of the `for await` loop cancels the exec, as `cancel()` does.

Only the stream transport has backpressure. Over the stream, the session reads the next message only after the reader took the last event. So a slow reader slows the command down ([the machine API](/reference/machine-api/index.html.md#how-a-stream-ends)). If nobody reads the output, the command stops once the window is full. A WebSocket has no backpressure, so the session keeps the output that waits for the reader in memory. At most `MAX_WEBSOCKET_BACKLOG`, 16 MiB, can wait. Past that, the exec fails with a `ResourceExhaustedError`, and the session closes the socket. The reader gets the event that it waits for, then the error. The rest of the output that waited is lost.

A write waits until the transport takes the input. If the exec ends first, each waiting write rejects with a `FailedPreconditionError`.

A failed exec throws its `HopboxError` from the read, such as a `FailedPreconditionError` for a machine that does not run. The close of the WebSocket maps to an error like this:

| Close code | Result |
| --- | --- |
| 1000 after the `exit` chunk | The read ends. |
| 1000 without the `exit` chunk | `UnknownError`, `the exec ended without its exit chunk` |
| 1001 | `UnavailableError` with the reason of the server |
| 4000 plus a Connect code, such as 4009 | The error class of that code, such as `FailedPreconditionError`, with the reason as `rawMessage` |
| any other code, such as 1006 | `UnavailableError`, `the WebSocket of exec closed with the code <code>` |

A browser reports a refused upgrade, such as HTTP status 403 for another origin, as the close code 1006.

After the `exit` chunk, both transports ignore an end that loses nothing. Over the stream, that is a reset of the stream or a cancel. Over the WebSocket, it is the close code 1000, 1005, or 1006. Any other error after the `exit` chunk still throws, such as the close code 4013 for `internal`.

## Upload

`upload(start, data, options?)` sends `start`, an `UploadStart` as a plain object ([the machine API](/reference/machine-api/index.html.md#upload)), then `data` in messages of at most 256 KiB (`MAX_UPLOAD_CHUNK`). It returns the `UploadResponse`, whose `bytes` is a `bigint`.

`UploadData` is one of these: a `Uint8Array`, a string in UTF-8, a `Blob`, an `Iterable<Uint8Array>`, an `AsyncIterable<Uint8Array>`, or a `ReadableStream<Uint8Array>`. In Node.js, a file read stream of `node:fs` is an `AsyncIterable<Uint8Array>`.

```ts
import { createReadStream } from "node:fs";

const { bytes } = await hb.upload({ machine: byName("box1"), path: "app.tar", mode: 0o644 }, createReadStream("app.tar"));
```

## Download

`download(request, options?)` reads the first message, the `DownloadInfo`, and returns a `Download` ([the machine API](/reference/machine-api/index.html.md#download)).

| `Download` member | Description |
| --- | --- |
| `info` | The `DownloadInfo`: `archive`, `mode`, and `size` as a `bigint`. |
| `bytes()` | Reads the rest of the data into one `Uint8Array`. |
| `[Symbol.asyncIterator]()` | Reads the data in parts of at most 32 KiB, with `for await`. |

Read the data once. A download to a laptop holds at most 4 MiB ([the machine API](/reference/machine-api/index.html.md#limits)).

## Logs and other streams

The log of a machine and the watch of machines have no helper. Call the generated methods, which carry the credential and the typed errors:

```ts
const ctl = new AbortController();
for await (const res of hb.machines.streamMachineLogs({ machine: byName("box1"), tail: 20, follow: true }, { signal: ctl.signal })) {
  if (res.event.case === "entry") {
    process.stdout.write(new TextDecoder().decode(res.event.value.data) + "\n");
  }
}
```

To stop a stream, abort its `signal`, or leave the `for await` loop.

## Constants

| Name | Value |
| --- | --- |
| `MAX_RETRIES` | 2 |
| `BACKOFF_MS` | `[500, 1000]` |
| `DEFAULT_ATTEMPT_TIMEOUT_MS` | 90000 |
| `LONG_WAIT_MS` | 10000 |
| `MAX_RETRY_AFTER_HEADER_MS` | 86400000, 24 hours |
| `IDEMPOTENCY_KEY_HEADER` | `Idempotency-Key` |
| `REQUEST_ID_HEADER` | `Hopbox-Request-Id` |
| `RETRY_AFTER_HEADER` | `Retry-After` |
| `NOT_RUN_HEADER` | `Hopbox-Not-Run` |
| `WEBSOCKET_EXEC_PATH` | `/api/ws/exec` |
| `MAX_WEBSOCKET_BACKLOG` | 16777216, 16 MiB |
| `MAX_STDIN_CHUNK` | 262144, 256 KiB |
| `MAX_UPLOAD_CHUNK` | 262144, 256 KiB |

## Build and tests

The scripts of `web/packages/sdk/package.json`:

| Script | What it does |
| --- | --- |
| `build` | `tsc -p tsconfig.build.json`, which writes `dist/` with the `.d.ts` files |
| `typecheck` | `tsc -p tsconfig.json`, over the code and the tests |
| `test` | `vitest run --project unit` |
| `test:e2e` | `vitest run --project e2e`. It needs the env of `TestTypeScriptSDK`, so run that Go test instead. |

`make test-sdk` runs the type check, the build, and the unit tests. Then it runs `TestTypeScriptSDK` of `internal/control` with `HOPBOX_TEST_SDK=1`. It needs `corepack pnpm install` first.

- The unit tests in `web/packages/sdk/test/*.test.ts` run the client against a fake Connect server over HTTP/2 and HTTP/1.1. They check the retries, the waits, `Retry-After`, and the idempotency keys. The table of retries covers a read call, a call with a key, and `createInvitation`, after a lost answer, an error code, an answer with `Hopbox-Not-Run`, and a refused connection, in Node.js and in the browser client. They also check the typed errors, the credentials, and the user agent. Then they check exec over the stream and over a fake WebSocket, upload, download, and the browser client.
- `TestTypeScriptSDK` serves the real public API of a control role. Its node runs a machine `box1` with the real guest agent and the test VMM driver. The test makes an API key, then runs `test/e2e/sdk.test.ts` with vitest. The suite checks exec over the stream and over the WebSocket, and a terminal with a resize. It also checks upload, download, and the log. Then it checks an idempotent replay, the details of a taken name, and a call with the session cookie of the console. It runs exec over the WebSocket with the session cookie and the `Origin` of the console, and checks that another `Origin` is refused.

The test reads each file of the SDK and of its tests first. So `go test` runs it again after a change of the SDK, instead of a result from its cache. Without `web/packages/sdk/node_modules`, `TestTypeScriptSDK` skips, so `make test` passes without Node.js. With `HOPBOX_TEST_SDK` set, it fails instead.

| Env var | Description |
| --- | --- |
| `HOPBOX_TEST_SDK` | Without the node_modules of the SDK, `TestTypeScriptSDK` fails instead of a skip. |
| `HOPBOX_E2E_URL`, `HOPBOX_E2E_KEY`, `HOPBOX_E2E_COOKIE`, `HOPBOX_E2E_ORIGIN`, `HOPBOX_E2E_MACHINE` | The install, API key, session cookie, origin, and machine of the end-to-end suite. `TestTypeScriptSDK` sets them. Without them, the suite fails at once. |
