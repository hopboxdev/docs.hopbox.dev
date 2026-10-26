# hopbox v0.1.0

> The release notes of v0.1.0, the pre-release at the end of phase 1: what it delivers by area, its known limits, the upgrade, and the acceptance on the dev install.

v0.1.0 is the first pre-release of hopbox. It ends phase 1, the foundation on one server. Each phase ends with a tagged pre-release, and version 1.0 is the end of phase 5.

The release has one server bundle for `linux/amd64`, the hopbox CLI, the hopbox Terraform provider, and the tested host kernel package `linux-image-6.18.57-hopbox1_6.18.57-1_amd64.deb`. At the adoption of a server, the provider installs that package ([Host kernel](/reference/host-kernel/index.html.md)). The release files are not public yet. The hopbox team gives them on request.

## What phase 1 delivers

Phase 1 runs every role of hopbox on one server. The dev install, which is the preview install at https://dev.hopbox.dev, ran this release at its acceptance ([Try hopbox](/guide/quickstart/index.html.md)). The sections below follow the parts of phase 1.

### Identity and sign-in

- A person signs in with GitHub, with Google, or with a magic link by email ([Sign in with GitHub and Google](/guide/github-google-sign-in/index.html.md)).
- A new account gets a personal org at its first sign-in.
- The first instance admin of a new install signs in with a one-time login link ([First sign-in on a new install](/guide/first-sign-in/index.html.md)).
- The audit log records each sign-in and each security action. Org owners and org admins read the entries of their org.
- [Sessions, cookies, and sign-in links](/reference/sessions/index.html.md), [Sign-in with GitHub and Google](/reference/oauth/index.html.md), and [the sign-in, account, and audit API](/reference/auth-api/index.html.md) give the rules and the calls.

### Orgs and keys

- An org has members with the roles owner, admin, and member. Invitations bring new members in ([Work in a team org](/guide/orgs/index.html.md)).
- Each org has usage limits, because sign-up is open. The defaults are 3 machines, 6 vCPUs, 24 GiB of RAM, and 300 GB of disk.
- An instance admin can change the limits of an org, and suspend an org.
- The CLI signs in with the device flow, and gets CLI tokens. People also make API keys and add SSH keys ([Use the hopbox CLI](/guide/cli/index.html.md), [CLI tokens, API keys, admin API keys, and SSH keys](/reference/credentials/index.html.md)).
- An org holds env vars and secrets with scopes. hopbox seals each secret with envelope encryption ([Set env vars and secrets](/guide/env-vars/index.html.md), [Sealing](/reference/sealing/index.html.md)).
- [The org API](/reference/org-api/index.html.md), [CLI tokens, API keys, admin API keys, and SSH keys](/reference/credentials/index.html.md), and [the env var API](/reference/vars-api/index.html.md) give the details.

### API and admin API

- The public API `hopbox.v1` uses Connect, with idempotency keys for changes ([the API server](/reference/api/index.html.md)).
- The admin API `hopbox.admin.v1` manages the install, its servers, its orgs, and admin API keys ([the admin API](/reference/admin-api/index.html.md)).
- `hopboxd admin` calls the admin API on the local admin socket of a server.
- The API definitions under `proto/` generate the Go code, the TypeScript code, and the API reference ([API definitions](/reference/api-definitions/index.html.md)).
- The Go client wraps the API with retries and typed errors ([Go client](/reference/go-client/index.html.md)). [Run the control role on a laptop](/guide/run-hopboxd/index.html.md) runs the API on a laptop.

### MCP server

- `hopboxd` serves an MCP server at `/mcp`. An AI agent uses it to manage machines ([Connect an AI agent to hopbox with MCP](/guide/mcp/index.html.md)).
- The agent sends an API key or a CLI token, and gets the rights of that credential and no other rights.
- Each unary call of `hopbox.v1` gets a tool, apart from the calls that take or return a credential ([the MCP server](/reference/mcp/index.html.md)).

### TypeScript SDK

- `@hopbox/sdk` calls the API from Node.js and from browsers ([Call the API from TypeScript](/guide/typescript-sdk/index.html.md)).
- It has retries, idempotency keys, typed errors, exec, and file copy ([TypeScript SDK](/reference/ts-sdk/index.html.md)).
- The console uses the same package. The package is not on npm yet.

### CLI

- The hopbox CLI runs on macOS on Apple Silicon, and on Linux on x86_64 and arm64.
- It makes, lists, starts, stops, reboots, renames, shares, and deletes machines ([Use machines from the CLI](/guide/cli-machines/index.html.md)).
- It manages orgs, members, invitations, API keys, SSH keys, env vars, secrets, and profiles ([Use the hopbox CLI](/guide/cli/index.html.md)).
- Each command can print JSON for scripts.
- The prompt completes commands, flags, and live machine names with the Tab key ([Work in the prompt of the CLI](/guide/cli-prompt/index.html.md)).
- The CLI writes a managed block into the SSH config of the user. So `ssh box1.hop` reaches the machine `box1`. [hopbox CLI](/reference/cli/index.html.md) lists every command and flag.

### Console

- `hopboxd` holds the console, the web app of an install ([Use the console](/guide/console/index.html.md)).
- The console has the sign-in, the live list of machines, and the create, stop, start, and delete actions.
- It also has the switch that makes a machine URL public, and a web terminal ([Console](/reference/console/index.html.md)).

### Machines and Firecracker

- A machine has a lifecycle: create, start, stop, reboot, rename, and delete ([Create and manage machines](/guide/machines/index.html.md)).
- A machine can have a start command, a restart policy, and an auto-delete timeout.
- The creator of a machine can share it with the whole org.
- Each machine is a Firecracker 1.17 microVM in a jail, in its own systemd unit ([Run machines with Firecracker](/guide/firecracker-machines/index.html.md), [the Firecracker driver](/reference/firecracker-driver/index.html.md)).
- Machines keep running through a restart or an upgrade of `hopboxd`.
- Each machine gets an IPv4 address, a tap device, and nftables rules that limit what it can reach ([Check the network of a machine](/guide/node-network/index.html.md), [the node network](/reference/node-network/index.html.md)).
- Placement picks the node of each new machine ([Placement](/reference/placement/index.html.md)). The executor on the node runs it ([the node executor](/reference/executor/index.html.md)).

### Exec, file copy, and logs

- Exec runs a command in a machine, with or without a terminal ([Run commands and copy files in a machine](/guide/exec-and-files/index.html.md)).
- File copy moves files into and out of a machine.
- The log of the start command stays on the node for 7 days or 100 MiB ([Machine logs](/reference/machine-logs/index.html.md)).
- A WebSocket endpoint carries exec for browsers. The guest agent `hopbox-guest` runs each command in the machine ([Guest link](/reference/guest-link/index.html.md)).

### SSH front door and hopbox connect

- The SSH front door of the edge role takes `ssh`, `scp`, `sftp`, `rsync`, and port forwards ([Reach a machine over SSH](/guide/ssh/index.html.md)).
- The SSH key of the account is the identity. Each machine has its own SSH port.
- `hopbox connect` opens a shell through the API, without an SSH key.
- [The SSH front door](/reference/ssh-front-door/index.html.md) gives the checks, the limits, and the logs.

### Edge and machine URLs

- Each machine gets an HTTPS URL under the domain of the install ([Run the edge role and open machine URLs](/guide/machine-urls/index.html.md)).
- A URL is private at first. Only accounts with access to the machine open it.
- The creator of a machine can make its URL public, and choose the port that it reaches.
- The edge also serves the API and the console of the install ([the edge role](/reference/edge/index.html.md)).

### DNS role and ACME

- The DNS role answers for the domain of the install ([Serve the domain of an install with the DNS role](/guide/dns-role/index.html.md), [the DNS role](/reference/dns/index.html.md)).
- A leader job gets a wildcard certificate for the install from Let's Encrypt, through DNS-01 ([Get the wildcard certificate of an install](/guide/acme-certificate/index.html.md)).
- The same job renews the certificate ([the certificate of the install](/reference/acme/index.html.md)).

### Enrollment and control channel

- Each server enrolls with the control role through mutual TLS, with a one-time enrollment token ([Enroll the node role on one server](/guide/enroll-node/index.html.md)).
- The install CA signs the certificate of each server, and the control channel renews it ([Enrollment and the install PKI](/reference/enrollment/index.html.md)).
- The control channel carries heartbeats, machine leases, and the streams of exec, file copy, and logs ([the control channel](/reference/control-channel/index.html.md)).
- The preflight checks of a server run before it enrolls ([the preflight check](/reference/preflight/index.html.md)).

### Terraform provider

- The resource `hopbox_server` adopts an OVHcloud dedicated server, or orders a new one ([Install hopbox on an OVHcloud server with Terraform](/guide/install-on-ovhcloud/index.html.md)).
- The provider reinstalls the server with Debian 13 and the host kernel `6.18.57-hopbox1`. Then it installs `hopboxd` and enrolls the server.
- A new `bundle` in the config upgrades the server ([hopbox Terraform provider](/reference/terraform-provider/index.html.md)).

### Default image and guest kernel

- The default image is Ubuntu 26.04 LTS with Docker, Node.js, Python, Go, uv, the GitHub CLI, ripgrep, and Neovim ([Default image](/reference/default-image/index.html.md)).
- The guest kernel is a hopbox build of the Amazon Linux 6.1 kernel ([Guest kernel](/reference/guest-kernel/index.html.md)).
- The builds of the image and the kernel are reproducible.
- The release bundle holds `hopboxd`, Firecracker, the jailer, Postgres, the guest kernel, the default image, and the systemd units ([Release bundle](/reference/release-bundle/index.html.md)).
- `hopboxd install`, `hopboxd upgrade`, and `hopboxd upgrade --rollback` put a bundle on a server ([Install hopbox on a Debian 13 server by hand](/guide/install-debian/index.html.md)).

### Postgres and migrations

- `hopboxd` brings its own Postgres 18.6, and supervises it as a child process ([the bundled Postgres](/reference/postgres/index.html.md)).
- The control store gets its schema from numbered migrations that `hopboxd` embeds ([Control store tables](/reference/control-store/index.html.md)).
- An upgrade or a rollback refuses a release that knows fewer migrations than the control store has.

## Known limits

Later phases build these parts:

- Phase 2: more servers, the mesh, separate edge servers, a cold drain, an external Postgres, and the Admin area of the console.
- Phase 3: standby, hibernate, checkpoints, snapshots, forks, backups, extra disks, and a wake on connect.
- Phase 4: org private networks, internal DNS, egress allowlists, placeholder secrets, custom domains, raw TCP and UDP ports, and isolated machines.
- Phase 5: coding agents in the image, the in-machine CLI, OIDC single sign-on, the desktop, runs and cron runs, integrations, metering, and the Python SDK.

This release also has these limits:

- An install has one server, and that server has every role.
- Machines get no IPv6 address. IPv6 egress waits for the egress through the edge servers of phase 2.
- A download of a file to a laptop holds at most 4 MiB ([the machine API](/reference/machine-api/index.html.md#limits)).
- The release files are not public yet, and the CLI has no public download yet ([Install the hopbox CLI](/guide/install-cli/index.html.md)).
- The Terraform provider is in no registry. Terraform gets it from a local dir.
- The SDK is not on npm.
- Only tests against a fake OVHcloud API cover the order of a new server.

## Upgrade

The dev install is for development. Its data can be erased at any time before 1.0. Schema migrations work from the first commit, so an upgrade keeps the data of an install.

To move a server to v0.1.0, run `hopboxd upgrade` with the bundle of this release ([Install hopbox on a Debian 13 server by hand](/guide/install-debian/index.html.md#upgrade)). The machines keep running through the upgrade.

## Acceptance on a live install

On 2026-10-23, the acceptance tests passed against the dev install, which ran the code of this release. Every call went through the edge, as the calls of a user do. The same day, the sign-in with GitHub in a browser, the console, the hopbox CLI, the SSH front door, and the MCP server passed their acceptance tasks end to end.

## Fixes from the acceptance

The acceptance found these bugs, and this release has the fixes:

- One HTTP/2 stream whose target read nothing blocked every other request of the client on its connection to the edge. A second exec next to an exec with unread input then hung. The edge now gives each stream a window of its own. It also bounds the open requests of each client and of all clients, so stalled requests hold a known amount of memory ([the edge role](/reference/edge/index.html.md#memory-of-stalled-requests)).
- `hopbox new` and `hopbox machine show` printed the image digest as `sha256:sha256:...`.
- The delete dialog of the console showed its title on the right, and its text ran past the right edge of the dialog. The delete still worked.
