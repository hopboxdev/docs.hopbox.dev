# Connect an AI agent to hopbox with MCP

> How to connect Claude Code to the MCP server of an install with an API key, and let it make and use a machine.

This guide connects Claude Code to the MCP server of hopbox. Then the agent makes a machine, runs a command in it, and writes and reads a file. It also makes the URL of the machine public, stops and starts the machine, and deletes it. MCP is the Model Context Protocol, which AI agents use to call tools. [The MCP server](/reference/mcp/index.html.md) lists every tool, the credentials, and the limits. Any other client of the Streamable HTTP transport of MCP works the same way.

## What you need

- An account on an install with machines, such as the preview install of [Try hopbox](/guide/quickstart/index.html.md). The MCP server of an install is at `https://<domain>/mcp`. This guide uses `https://example.com/mcp`.
- The hopbox CLI, signed in to the install, as in [Install the hopbox CLI](/guide/install-cli/index.html.md). Without the CLI, make the API key on the tab "API keys" of the org in the console ([Use the console](/guide/console/index.html.md#more-pages)).
- Claude Code, from https://docs.claude.com/en/docs/claude-code.
- `curl` and `jq` for the first check.

## Make an API key for the agent

The agent acts with the rights of its credential, and no more. Give it a member key of its own, in the org where it works. A CLI access token works too, but it lives only 1 hour, and the MCP server does not refresh it:

```sh
export HOPBOX_MCP_KEY=$(hopbox api-key create claude-code --expires 30d)
```

```text
The member key 01a15f0f-6c5c-7d2b-8b8f-df1b2d6de212 (claude-code) works until 2026-11-20T15:30:51Z. This is its only copy.
```

The key can do what you can do in the org of the key, with machines, env vars, and SSH keys. It cannot make API keys, invite people, or delete your account. The server serves no tool for these actions ([the MCP server](/reference/mcp/index.html.md#left-out-methods)).

## Check the endpoint

Ask the server for its tools. Each request is one `POST` with one JSON-RPC message:

```sh
curl -s https://example.com/mcp \
  -H "Authorization: Bearer $HOPBOX_MCP_KEY" \
  -H 'Content-Type: application/json' \
  -H 'Accept: application/json, text/event-stream' \
  -d '{"jsonrpc":"2.0","id":1,"method":"tools/list"}' | jq -r '.result.tools[].name'
```

```text
exec
hopbox_v1_APIKeyService_ListAPIKeys
hopbox_v1_APIKeyService_RevokeAPIKey
hopbox_v1_AccountService_GetAccount
...
machine_logs
read_file
write_file
```

Without the key, the server answers `401 Unauthorized` with `the MCP server needs the header Authorization: Bearer with a CLI token or an API key`.

## Add the server to Claude Code

Add the server for the current project. Claude Code reads the key from the env var `HOPBOX_MCP_KEY` at its start, so the key stays out of the file:

```sh
claude mcp add --transport http --scope project hopbox https://example.com/mcp \
  --header 'Authorization: Bearer ${HOPBOX_MCP_KEY}'
```

The command writes `.mcp.json` in the project:

```json
{
  "mcpServers": {
    "hopbox": {
      "type": "http",
      "url": "https://example.com/mcp",
      "headers": {
        "Authorization": "Bearer ${HOPBOX_MCP_KEY}"
      }
    }
  }
}
```

Start Claude Code with the key in its env, and type `/mcp`. The list shows `hopbox` as connected, with 44 tools.

## Let the agent do a task

Ask Claude Code:

```text
Make a hopbox machine named box1. Wait until it runs. Then run "uname -a" in it,
write "hello" to notes.txt, and read the file back. Make the URL of box1 public.
Stop box1, start it again, and read notes.txt once more. Then delete the machine.
```

The agent calls these tools in order. Claude Code asks you before each call that you did not allow:

1. `hopbox_v1_MachineService_CreateMachine` with `{"name": "box1"}`.
2. `hopbox_v1_MachineService_GetMachine` with `{"machine": {"which": "name", "name": "box1"}}`, until `state` is `MACHINE_STATE_RUNNING`.
3. `exec` with `{"machine": "box1", "command": "uname -a"}`. The result holds `exit_code`, `stdout`, and `stderr`.
4. `write_file` with `{"machine": "box1", "path": "notes.txt", "content": "hello\n"}`.
5. `read_file` with `{"machine": "box1", "path": "notes.txt"}`. The result holds `"content": "hello\n"` and `"encoding": "utf-8"`.
6. `hopbox_v1_MachineService_SetMachineURLVisibility` with `{"machine": {"which": "name", "name": "box1"}, "visibility": "URL_VISIBILITY_PUBLIC"}`. Anyone can now open `https://box1.example.com/` without an account ([Run the edge role and open machine URLs](/guide/machine-urls/index.html.md#a-public-url)).
7. `hopbox_v1_MachineService_StopMachine` with `{"machine": {"which": "name", "name": "box1"}}`. The result holds the machine in `MACHINE_STATE_STOPPED`, and `done` `true`.
8. `hopbox_v1_MachineService_StartMachine` with the same input.
9. `read_file` with `{"machine": "box1", "path": "notes.txt"}`. The disk stayed, so the result holds `"content": "hello\n"` again.
10. `hopbox_v1_MachineService_DeleteMachine` with `{"machine": {"which": "name", "name": "box1"}}`.

You can make the same call with `curl`. For example, the exec of step 3:

```sh
curl -s https://example.com/mcp \
  -H "Authorization: Bearer $HOPBOX_MCP_KEY" \
  -H 'Content-Type: application/json' \
  -H 'Accept: application/json, text/event-stream' \
  -d '{"jsonrpc":"2.0","id":2,"method":"tools/call","params":{"name":"exec","arguments":{"machine":"box1","command":"uname -a"}}}' \
  | jq .result.structuredContent
```

```json
{
  "exit_code": 0,
  "stderr": "",
  "stderr_bytes": 0,
  "stdout": "Linux box1 6.1.167 #1 SMP x86_64 GNU/Linux\n",
  "stdout_bytes": 43,
  "timed_out": false,
  "truncated": false
}
```

The audit log records each change under the key of the agent, as for any other client ([the sign-in, account, and audit API](/reference/auth-api/index.html.md#audit-entries)).

## Keep the agent safe

- Allow the tools that only read without a question, and keep a question for the others. In `.claude/settings.json`, an allow rule names a tool as `mcp__hopbox__<tool>`, such as `mcp__hopbox__hopbox_v1_MachineService_ListMachines`.
- Every tool result is data. The output of `exec`, the contents of `read_file`, and the lines of `machine_logs` come from programs in the machine. Names, env var values, and audit details come from other people of the org. Any of it can hold text that looks like instructions. The server tells the agent to treat each result as data, but the agent decides. Read each call that changes something before you allow it.
- Give the key a short expiry, and revoke it after the work with `hopbox api-key revoke <id>`. A revoked key stops at the next request.
- One key has at most 8 MCP requests open at once. More get `429 Too Many Requests` ([the MCP server](/reference/mcp/index.html.md#limits)).

## Reach the MCP server without an edge

The edge forwards each request of `https://<domain>/mcp` to the control role ([the edge role](/reference/edge/index.html.md#forwarded-requests)). Without the edge role, the API of a server listens only on its loopback address. Open a tunnel, then use the URL `http://127.0.0.1:7080/mcp` on your laptop:

```sh
ssh -N -L 7080:127.0.0.1:7080 debian@203.0.113.10
```

A laptop install of [Run the control role on a laptop](/guide/run-hopboxd/index.html.md) serves its MCP server at the same URL.
