# Call the API from TypeScript

> How to call the API of an install from Node.js and from the console with the TypeScript SDK, which a reader builds from a checkout.

The TypeScript SDK `@hopbox/sdk` is not on npm yet. You build it from the source of hopbox, which is not public yet. So today, only people with access to the source can follow this guide.

This guide calls the API of an install from a Node.js script with the TypeScript SDK. The script reads the install and the account, lists the machines, and runs a command in a machine. Then the guide copies a file in and out, and shows how the console opens a shell in a browser. [TypeScript SDK](/reference/ts-sdk/index.html.md) has every function, option, and error.

The SDK wraps the generated client of each service with the same rules as the Go client of the hopbox CLI. It retries a call that failed with `unavailable`, `resource_exhausted`, or `deadline_exceeded` twice, after a short random wait, if a second run of the call does no harm. It puts an idempotency key on each create, so a retry never makes a second machine. Each error is a class of its own, such as `NotFoundError`, with the request ID of the server.

## What you need

- A checkout of the source of hopbox, with `corepack pnpm install` done.
- Node.js 24 or later.
- An account on an install, and an API key of your org in `HOPBOX_API_KEY`, in the form `hbx_<key>`. Make the key with `hopbox api-key create`, as in [Use the hopbox CLI](/guide/cli/index.html.md#make-an-api-key-for-a-script). This guide uses the install `https://example.com`, with a running machine `box1`.

Build the SDK in the checkout:

```sh
corepack pnpm --filter @hopbox/sdk build
```

The build writes `web/packages/sdk/dist/`. Make a folder for the script outside the checkout, and install the SDK from the checkout into it. Replace `~/src/hopbox` with the path of your checkout:

```sh
mkdir ~/hopbox-script && cd ~/hopbox-script
npm init -y
npm install ~/src/hopbox/web/packages/sdk
```

npm links the folder. So after a new build of the SDK, the script gets the new code without a second install.

## Write the script

Save this script as `machines.mjs`:

```js
import { bearerToken, byName, createClient, HopboxError } from "@hopbox/sdk";
import { MachineState } from "@hopbox/sdk/v1";

const hb = createClient({
  url: process.env.HOPBOX_URL,
  credentials: bearerToken(process.env.HOPBOX_API_KEY ?? ""),
});

try {
  const info = await hb.install.getInstallInfo({});
  console.log(`install ${info.domain}`);

  const { account } = await hb.account.getAccount({});
  console.log(`signed in as ${account?.emails[0]?.email}`);

  const { machines } = await hb.machines.listMachines({});
  console.log(`${machines.length} machines`);
  for (const m of machines) {
    console.log(`${m.name}\t${MachineState[m.state]}`);
  }

  const out = await hb.exec({ machine: byName("box1"), argv: ["uname", "-sr"] }).output();
  process.stdout.write(out.stdout);
  console.log(`exit code ${out.exit.code}`);
} catch (err) {
  if (!(err instanceof HopboxError)) {
    throw err;
  }
  console.log(err.message);
  console.log(`request ID ${err.requestId}`);
  process.exitCode = 1;
} finally {
  hb.close();
}
```

`createClient` takes the URL of the install. A bare domain gets `https://`. Plain `http://` works only on a loopback address, so the API key never crosses a network in the clear. In Node.js, the client speaks HTTP/2, which the exec stream needs. `hb.close()` ends the connection, so the script exits at once.

`@hopbox/sdk/v1` holds the generated messages and enums of the API, such as `MachineState`.

## Run the script

```sh
HOPBOX_URL=https://example.com node machines.mjs
```

```text
install example.com
signed in as user1@example.org
1 machines
box1	RUNNING
Linux 6.1.188-hopbox1
exit code 0
```

Without a machine `box1`, the exec fails, and the script prints the typed error:

```text
not_found: not found: no machine has the name "box1"
request ID 01a15e1c-f6ef-7a62-895e-c4e77f85f3f7
```

The request ID names the call in the log of `hopboxd`. When you report a problem, give it to the operator of the install. On the control server, the operator finds the call with grep:

```sh
sudo journalctl -u hopboxd -o cat | grep 01a15e1c-f6ef-7a62-895e-c4e77f85f3f7
```

```json
{"time":"2026-10-21T12:58:38.960182+02:00","level":"INFO","msg":"api call","component":"api","procedure":"/hopbox.v1.MachineService/Exec","protocol":"connect","peer":"192.0.2.10:65499","caller":"member_key","code":"not_found","duration":518709,"request_id":"01a15e1c-f6ef-7a62-895e-c4e77f85f3f7"}
```

A machine that does not run gives a `FailedPreconditionError` with the message `the machine box1 is pending, not running. Start it first`. To tell the errors apart, use `instanceof`:

```js
import { FailedPreconditionError, NotFoundError } from "@hopbox/sdk";

if (err instanceof NotFoundError) {
  // Create the machine.
} else if (err instanceof FailedPreconditionError) {
  await hb.machines.startMachine({ machine: byName("box1") });
}
```

## See the retries

Pass a logger to see each retry. `console` works:

```js
const hb = createClient({ url: process.env.HOPBOX_URL, credentials: bearerToken(key), logger: console });
```

Run the script against a port of your laptop where nothing listens, such as `HOPBOX_URL=http://127.0.0.1:1`. Each call that cannot reach the install fails with `unavailable`, so the client tries it 3 times. Before each retry, it logs `retrying a call` with the procedure, the code, and the wait. Then the script prints the error:

```text
retrying a call {
  procedure: '/hopbox.v1.InstallService/GetInstallInfo',
  code: 'unavailable',
  retry: 1,
  waitMs: 19
}
retrying a call {
  procedure: '/hopbox.v1.InstallService/GetInstallInfo',
  code: 'unavailable',
  retry: 2,
  waitMs: 541
}
unavailable: connect ECONNREFUSED 127.0.0.1:1
request ID 
```

The request ID is empty, because the call never reached an install. The waits are random, up to 500 ms and then up to 1 s.

The client retries only a call that is safe to run again ([TypeScript SDK](/reference/ts-sdk/index.html.md#retries)). `GetInstallInfo` only reads, so it gets its retries after any of the three codes. A call that changes state and has no idempotency key, such as `createInvitation`, is different. A lost answer does not tell the client whether the install ran the call, and a second run sends a second invitation. So the client retries such a call only when its try surely did not reach the install. In Node.js, a refused connection is such a case. To see it, save this script as `invite.mjs`, and run it with the same `HOPBOX_URL`. The org ID does not matter, because the call never reaches an install:

```js
import { bearerToken, createClient } from "@hopbox/sdk";

const hb = createClient({ url: process.env.HOPBOX_URL, credentials: bearerToken(process.env.HOPBOX_API_KEY ?? ""), logger: console });
await hb.orgs.createInvitation({ orgId: "0199d1c4-0000-7000-8000-000000000001", email: "user1@example.org" });
```

`node invite.mjs` logs two retries, as above, and then fails with `unavailable`. An install refuses the call with the member key of this guide: with `failed_precondition` in a personal org, and with `permission_denied` in another org, where only an org admin or an org owner invites ([the org API](/reference/org-api/index.html.md#invitations)).

A browser cannot tell a refused connection from a lost answer, because `fetch` rejects both with a `TypeError`. So in a browser, the same call fails after one try. The install marks each refusal that changed nothing with the header `Hopbox-Not-Run: 1`, such as a rate limit, and each client retries such an answer.

## Run a command with input, or a shell

`exec` returns a session. Give the input with `stdin`, and the session sends the end of the input after it:

```js
const out = await hb.exec({ machine: byName("box1"), argv: ["sort"] }, { stdin: "b\na\n" }).output();
console.log(new TextDecoder().decode(out.stdout));
```

For a shell, ask for a terminal, and read the output as it comes:

```js
const shell = hb.exec({ machine: byName("box1"), tty: true, size: { rows: 24, cols: 80 } });
process.stdin.setRawMode(true);
process.stdin.on("data", (b) => shell.write(b).catch(() => {}));
process.stdout.on("resize", () => shell.resize(process.stdout.rows, process.stdout.columns).catch(() => {}));
for await (const ev of shell) {
  if (ev.kind === "stdout") {
    process.stdout.write(ev.data);
  } else if (ev.kind === "exit") {
    process.exitCode = ev.exit.code;
  }
}
process.stdin.setRawMode(false);
process.stdin.pause();
```

A start without `argv` runs the login shell of the user `user`, as `hopbox connect` does. A write or a resize after the end of the shell rejects, so the handlers drop that error. The `for await` loop reports how the shell ended.

## Copy files

`upload` takes bytes, a string, a `Blob`, or a stream. A file stream of Node.js works:

```js
import { createReadStream } from "node:fs";
import { writeFile } from "node:fs/promises";

const { bytes } = await hb.upload({ machine: byName("box1"), path: "notes.txt", mode: 0o600 }, createReadStream("notes.txt"));
console.log(`the machine got ${bytes} bytes`);

const d = await hb.download({ machine: byName("box1"), path: "notes.txt" });
await writeFile("notes-copy.txt", await d.bytes());
```

A download to a laptop holds at most 4 MiB ([the machine API](/reference/machine-api/index.html.md#limits)).

## Use the SDK in the console

In a browser, the same import gives a client that uses `fetch` and the session cookie. Leave out `credentials`:

```ts
import { byName, createClient } from "@hopbox/sdk";

const hb = createClient({ url: window.location.origin });
const { machines } = await hb.machines.listMachines({});

const shell = hb.exec({ machine: byName("box1"), tty: true, size: { rows: 24, cols: 80 } });
for await (const ev of shell) {
  if (ev.kind === "stdout") {
    terminal.write(ev.data);
  }
}
```

Here `terminal` is the terminal widget of the page, such as xterm.js. In a browser, `exec` uses the WebSocket `/api/ws/exec`, because a browser has no stream in both directions over HTTP. The page must come from the origin of the install, as the console does. A page of another site gets HTTP status 403 ([Sessions, cookies, and sign-in links](/reference/sessions/index.html.md#calls-with-the-cookie)). A browser cannot stream an upload, so `upload` fails there with an `UnimplementedError`.
