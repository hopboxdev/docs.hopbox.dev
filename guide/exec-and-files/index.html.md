# Run commands and copy files in a machine

> How to run commands in a machine, copy files, and read its log through the API with grpcurl, and open a shell over the WebSocket.

This guide runs a command in a machine through the API, copies a file in and out, and reads the log of the start command. It uses the access calls of the API: `Exec`, `Upload`, `Download`, and `StreamMachineLogs` ([the machine API](/reference/machine-api/index.html.md#access-calls)). Use these calls from your own tools. The hopbox CLI does the same work with `exec`, `cp`, `logs`, and `connect` ([Use machines from the CLI](/guide/cli-machines/index.html.md)). The console uses the WebSocket of exec, and this guide shows it too.

Each access call needs a machine that runs, except the log. For a machine that does not run, the calls answer `failed_precondition`, with a message such as `the machine box1 is pending, not running. Start it first`.

## What you need

- A machine `box1` that runs, as in [Use machines from the CLI](/guide/cli-machines/index.html.md#make-a-machine). This guide uses the install `example.com`.
- An API key of your org in `KEY`, in the form `hbx_<key>`. Make one with `KEY=$(hopbox api-key create tools --expires 30d)`, as in [Use the hopbox CLI](/guide/cli/index.html.md#make-an-api-key-for-a-script). An exec streams in both directions, so it needs gRPC or HTTP/2. A tool such as grpcurl speaks gRPC, so it sends the key.
- grpcurl, from https://github.com/fullstorydev/grpcurl. The API serves gRPC reflection, so grpcurl needs no proto files.

The edge servers of the install serve the API over TLS on port 443:

```sh
API=example.com:443
hb() { grpcurl -H "Authorization: Bearer $KEY" "$@"; }
```

## Run a command

`Exec` takes the start chunk first, then the input. In JSON, a bytes field such as `stdin` or `stdout` is base64. This command prints a line on each output, and exits with the code 3:

```sh
hb -d @ $API hopbox.v1.MachineService/Exec <<'EOF'
{"chunk":{"start":{"name":"box1","argv":["sh","-c","echo out; echo err >&2; exit 3"]}}}
{"chunk":{"stdinEof":true}}
EOF
```

```text
{"chunk":{"stdout":"b3V0Cg=="}}
{"chunk":{"stderr":"ZXJyCg=="}}
{"chunk":{"exit":{"code":3}}}
```

`b3V0Cg==` is `out` and a newline. Add `"timeout":"10s"` to the start chunk to end a command after 10 seconds. Such a command ends with the code 124 and `"timedOut":true`.

A member of the org cannot run a command in your machine until you share it. An org admin can never run one in your private machine ([the machine API](/reference/machine-api/index.html.md#who-can-call-them)).

## Copy a file into the machine

`Upload` takes the start message first, then the data. grpcurl ends the data at the end of its input:

```sh
hb -d @ $API hopbox.v1.MachineService/Upload <<EOF
{"start":{"name":"box1","path":"notes.txt","mode":384}}
{"data":"$(printf 'hello from the laptop\n' | base64)"}
EOF
```

```text
{"bytes":"22"}
```

The mode 384 is 0600. A relative path starts at the home directory of the user `user`. To copy a directory, send a tar stream of its contents, with `"archive":true`.

## Copy the file back

```sh
hb -d '{"name":"box1","path":"notes.txt"}' $API hopbox.v1.MachineService/Download
```

```text
{"info":{"mode":384,"size":"22"}}
{"data":"aGVsbG8gZnJvbSB0aGUgbGFwdG9wCg=="}
```

A download to a laptop has at most 4 MiB. A larger file fails with `failed_precondition`. Pack the file in the machine and copy it in parts, or copy it over SSH.

## Read the log

The log holds the output of the start command of the machine. The node keeps it for 7 days or 100 MiB. This call sends the last 20 lines, then each new line, until you press Ctrl-C:

```sh
hb -d '{"name":"box1","tail":20,"follow":true}' $API hopbox.v1.MachineService/StreamMachineLogs
```

```text
{"entry":{"time":"2026-10-18T09:12:03.418Z","stream":"MACHINE_LOG_STREAM_STDOUT","data":"c2VydmluZyBvbiA6ODAwMA=="}}
{"synced":true}
```

The log stays readable while the machine is stopped.

## Open a shell from the console

The console opens a WebSocket to `/api/ws/exec` with the session cookie. Each binary message holds one `ExecChunk` encoded as protobuf. The generated messages of the TypeScript SDK, in `@hopbox/sdk/v1`, make and read it ([Call the API from TypeScript](/guide/typescript-sdk/index.html.md)). A start chunk without `argv` and with `tty` opens the login shell, as `hopbox connect` does:

```ts
import { create, toBinary, fromBinary } from "@bufbuild/protobuf";
import { ExecChunkSchema } from "@hopbox/sdk/v1";

const ws = new WebSocket("wss://example.com/api/ws/exec");
ws.binaryType = "arraybuffer";
ws.onopen = () => {
  const start = create(ExecChunkSchema, {
    chunk: { case: "start", value: { machine: { case: "name", value: "box1" }, tty: true, size: { rows: 24, cols: 80 } } },
  });
  ws.send(toBinary(ExecChunkSchema, start));
};
ws.onmessage = (ev) => {
  const chunk = fromBinary(ExecChunkSchema, new Uint8Array(ev.data));
  if (chunk.chunk.case === "stdout") console.log(new TextDecoder().decode(chunk.chunk.value));
};
ws.onclose = (ev) => console.log("closed", ev.code, ev.reason);
```

The TypeScript SDK wraps this socket in `exec`, with the same session for a browser and for Node.js ([Call the API from TypeScript](/guide/typescript-sdk/index.html.md#use-the-sdk-in-the-console)). The page must have the origin of the install, so this works only from the console. A page of another site gets HTTP status 403. At the end of the shell, the socket closes with the code 1000. An error closes it with 4000 plus the Connect code, such as 4007 for `permission_denied` ([the machine API](/reference/machine-api/index.html.md#the-websocket-endpoint)).
