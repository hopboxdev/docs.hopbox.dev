# hopboxd

> The commands, flags, logs, and exit status of hopboxd, with the start and stop order of hopboxd run, the leader jobs, install, upgrade and its rollback, enroll, preflight, and the admin commands for tokens, servers, admin API keys, orgs, machines, and images.

`hopboxd` is the server binary of hopbox. It runs on every server of an install.

## Usage

```sh
hopboxd [--log-level=LEVEL] <command> [flags]
```

When you run `hopboxd` without a command, it prints its help and an error, and exits with status 80.

## Global flags

A global flag goes before or after the command.

| Flag | Type | Default | Description |
| --- | --- | --- | --- |
| `--log-level` | level | `info` | The lowest level of the log records that `hopboxd` writes. The values are `debug`, `info`, `warn`, and `error`. Case does not matter. A value can also carry an offset, for example `info+2`, as in Go's `log/slog`. |
| `-h`, `--help` | bool | `false` | Prints the help of `hopboxd`, or of the command that it follows, and exits. |

Example:

```sh
hopboxd --log-level=debug version
```

## Commands

| Command | Description |
| --- | --- |
| `run` | Runs the roles of this server until it gets `SIGINT` or `SIGTERM`. See [run](#run). |
| `install` | Installs the release of this `hopboxd` on this server, or brings the install up to date. See [install](#install). |
| `upgrade` | Upgrades this server to the release of a bundle file, or rolls back to the previous release. See [upgrade](#upgrade). |
| `enroll` | Makes this server a part of an install with an enrollment token. See [enroll](#enroll). |
| `preflight` | Checks that this server can run machines. See [preflight](#preflight). |
| `admin kek rotate` | Adds a new key-encryption key through the admin socket, for an install with one control server. See [admin](#admin). |
| `admin kek stage` | Adds a staged key-encryption key, which seals nothing yet. |
| `admin kek promote` | Makes a staged key-encryption key the current one on every control server. |
| `admin kek list` | Lists the key-encryption keys and the sealed values of each. |
| `admin kek rewrap` | Moves every sealed value to the current key-encryption key. |
| `admin config get` | Prints the settings of the install through the admin socket. |
| `admin config set` | Changes settings of the install through the admin socket. |
| `admin config smtp get` | Prints the SMTP config, without its password. |
| `admin config smtp set` | Replaces the SMTP config. |
| `admin config smtp clear` | Removes the SMTP config. |
| `admin config oauth get` | Prints the GitHub and Google clients for sign-in, without their secrets, with the redirect URIs. `admin config oauth` alone does the same. |
| `admin config oauth set` | Sets the client of one sign-in provider. |
| `admin config oauth clear` | Removes the client of one sign-in provider. |
| `admin certificate get` | Prints the certificate of the install, without its key, and the state of the ACME job. |
| `admin login-link` | Prints a one-time sign-in link for the first instance admin. |
| `admin instance-admin list` | Lists the instance admins. |
| `admin instance-admin grant` | Makes an account an instance admin. |
| `admin instance-admin revoke` | Takes the instance admin role from an account. |
| `admin account sign-in-method list` | Lists the sign-in methods of an account. |
| `admin account sign-in-method remove` | Removes a sign-in method of an account, to recover its sign-in. |
| `admin enrollment-token create` | Makes an enrollment token for a new server through the admin socket. |
| `admin server list` | Lists the enrolled servers through the admin socket. |
| `admin server get` | Prints one server through the admin socket. |
| `admin server cordon` | Stops new machines from landing on a server, through the admin socket. |
| `admin server uncordon` | Lets new machines land on a cordoned server again, through the admin socket. |
| `admin server set-labels` | Replaces the labels of a server, through the admin socket. |
| `admin server drain` | Moves every machine of a server to other nodes, then decommissions the server, through the admin socket. |
| `admin server remove-role` | Starts the removal of the roles edge, dns, node, or more than one from a running server, through the admin socket. |
| `admin server cancel-remove-role` | Ends the pending removal of roles from a server that cannot finish, through the admin socket. |
| `admin server cancel-drain` | Ends the drain of a server, which turns cordoned or ready as it was before the drain, through the admin socket. |
| `admin server decommission` | Takes a server out of the install for good, through the admin socket. |
| `admin control list` | Lists the control processes of the install, live or not, through the admin socket. |
| `admin pool create` | Adds a pool of servers. |
| `admin pool get` | Prints one pool. |
| `admin pool list` | Lists the pools. |
| `admin pool delete` | Deletes a pool without servers and without live sizes. |
| `admin size create` | Adds a size of machine. |
| `admin size get` | Prints one size. |
| `admin size list` | Lists the sizes. |
| `admin size retire` | Stops new machines of a size. |
| `admin org list` | Lists every org of the install, with metadata only. |
| `admin org suspend` | Suspends an org. |
| `admin org unsuspend` | Ends the suspension of an org. |
| `admin org set-limits` | Replaces the usage limits of one org. |
| `admin org default-limits get` | Prints the default usage limits of every org. |
| `admin org default-limits set` | Changes the default usage limits of every org. |
| `admin machine list` | Lists the machines of every org, or of one org, with metadata only. |
| `admin machine stop` | Stops any machine, also in a suspended org, and waits for it. |
| `admin machine delete` | Deletes any machine, also in a suspended org, and waits for it. |
| `admin machine move` | Moves any machine cold to another node. It returns at once. With `--cancel`, it ends the move of the machine that runs. With `--give-back` too, it asks the target node to give back a machine that a failed move left there. |
| `admin machine moves` | Lists the cold moves of one machine, or of every machine, newest first. |
| `admin image register` | Adds an image from its `artifact.json`. |
| `admin image list` | Lists the images of the install. |
| `admin admin-key create` | Makes an admin API key and prints it once. |
| `admin admin-key register` | Makes an admin API key from the SHA-256 of a key that you made. |
| `admin admin-key list` | Lists the admin API keys. |
| `admin admin-key revoke` | Deletes an admin API key. |
| `admin mail list` | Lists the mail files that the file mail transport wrote on this server. |
| `admin mail show` | Prints one mail file, with its link whole. |
| `admin maintenance on` | Turns maintenance mode of the install on, and waits until the writes, the control channels, and the leader jobs that ran have ended in every live control process. |
| `admin maintenance off` | Turns maintenance mode of the install off. |
| `admin db export` | Copies the control store of this server from the bundled Postgres to an empty external Postgres, in maintenance mode, and prints the config lines for it. |
| `version` | Prints the version of `hopboxd`. See [the version command](/reference/version/index.html.md). |
| `stop-guest` | Shuts down the guest of a machine unit. systemd runs it as the `ExecStop` of the unit. The help does not list it. See [stop-guest](#stop-guest). |

## run

```sh
hopboxd run [--config=PATH]
```

`run` reads the bootstrap config, starts the roles that the config lists, and runs until it gets `SIGINT` or `SIGTERM`. Then it stops the roles and exits with status 0. [The bootstrap config of hopboxd](/reference/hopboxd-config/index.html.md) lists the keys of the bootstrap config.

| Flag | Type | Default | Description |
| --- | --- | --- | --- |
| `--config` | path | `/etc/hopbox/hopboxd.toml` | The bootstrap config file. |

First, `run` starts the metrics listener and the host firewall. Then it starts the units of the control role, the DNS role, the edge role, and the node role, in this order. Then it starts the unit of the control channel. A role that the config does not list has no units. Each unit starts after the unit before it is ready.

| Order | Unit | What it does | Stop timeout |
| --- | --- | --- | --- |
| 0 | `metrics` | Serves the Prometheus metrics on `metrics.listen` ([Metrics](/reference/metrics/index.html.md)). | 5 s |
| 1 | `host firewall` | Replaces the nftables table `inet hopbox_host` with the rules of the roles of the server, in one netlink batch ([the host firewall](/reference/host-firewall/index.html.md)). The table is in place before any role opens a listener. If the batch fails, the start fails, and the table before it stays. Until its stop, it watches the table. It installs the table again after a flush, and after a change of its sets or of the uplink of an edge server. Its stop keeps the table, so the firewall holds while `hopboxd` is down. It keeps its last peers, jump hosts, and known peers in `<data_dir>/hostfw` ([the host firewall](/reference/host-firewall/index.html.md#never-a-lockout)). After the install of the table, an edge server that enrolls turns on forwarding for the egress ([the host firewall](/reference/host-firewall/index.html.md#the-egress-of-the-machines)). | 0.1 s |
| 2 | `postgres` | Starts the bundled Postgres ([the bundled Postgres](/reference/postgres/index.html.md)). The cluster is in `<data_dir>/pg`, and the socket is in `<run_dir>/pg`. With `postgres.url`, it starts no Postgres. It reads the CA file and the password file of the external Postgres, and the next units connect to it over TLS ([the bundled Postgres](/reference/postgres/index.html.md#external-postgres)). | 30 s |
| 3 | `store` | Opens the connection pool of the control store and runs the migrations ([the bundled Postgres](/reference/postgres/index.html.md#schema-migrations)). Then it checks the key files. It first applies a promote of a key-encryption key that happened while this process was down, and then opens the install CA of the control store with the key files of this server. It makes the CA when none exists. If the key files cannot open it, the start fails with `control: the key files in <dir> do not open the install CA of the control store: <error>. Copy the key files of a running control server there`. A keyring without any key passes this check ([Sealing](/reference/sealing/index.html.md#several-control-servers)). Then it imports the bootstrap admin API key from the file that `hopboxd install` can leave ([Server layout](/reference/server-layout/index.html.md#the-bootstrap-admin-api-key)). Then it registers the default image of the release ([The default image of the release](#the-default-image-of-the-release)). Until its stop, it reads the schema of the store every 10 s. If a newer `hopboxd` applied a migration that does not run with this one, if the store lost its schema, or if 12 reads in a row fail, it stops `hopboxd`. 12 reads take about 2 minutes, which outlasts the failover of a managed PostgreSQL ([the bundled Postgres](/reference/postgres/index.html.md#two-releases-on-one-store)). The limit is safe only because of the fence below: a process that cannot reach the store fences itself within 15 s, long before the schema watch stops it, so it never writes while the other control processes no longer count it. The migration runner writes the first row of this control process in the table `control_processes` while it still holds the migration lock, right after the migrations and with the same check of the schema, so no newer `hopboxd` applies a migration that does not allow this one before the first row exists ([the bundled Postgres](/reference/postgres/index.html.md#schema-migrations)). At the end of the unit, it writes the full row and keeps it up to date until its stop ([Control store tables](/reference/control-store/index.html.md#control_processes)). The row holds a new random ID for each start, the ID of the enrolled server of the same `hopboxd` or none, the host name, the URL of the internal listener, the release, the schema version, the key files and the staged key files that it can open, the key with which it seals, and the maintenance and drained flags. While its key dir has a staged file, or while it misses the promoted key, each renewal first applies the promote of the install ([Sealing](/reference/sealing/index.html.md#several-control-servers)), as does each change event `keyring.kek_promoted`. The unit renews the row on a fixed ticker every 5 s. Each try takes at most 2 s, so a slow store never moves the next renewal. It also renews at once after a turn of maintenance mode, after the internal listener starts, at a change event of a promote of a KEK, and once at the end of a drain, when no call, control channel, or leader job of the process runs any more in maintenance mode. A row renewed in the last 20 s is live. Each renewal reads the schema of the store in the same transaction, and fails when the store no longer runs with this `hopboxd`, so a renewal that works means both ([the bundled Postgres](/reference/postgres/index.html.md#two-releases-on-one-store)). A renewal that fails this way fences the process and stops `hopboxd` at once. When no renewal worked for 15 s (the 20 s of the live window less a margin of 5 s, counted from the start of the last renewal that worked), the process fences itself before its row stops being live. While it is fenced, every listener of the API refuses every call, reads too, with `unavailable`, the header `Hopbox-Not-Run: 1`, the header `Retry-After: 30`, and the message `this control server lost the control store for a while, so it takes no call until it reaches the store again. Retry later, or through another control server`. Only gRPC health and reflection pass. The plain HTTP routes that maintenance mode refuses, such as the starts of the GitHub and Google sign-in, answer 503 with that message. No control channel and no leader job runs. A leader that stays fenced for 15 s gives up its lease, and a fenced process does not try for the leader lock ([Leader jobs](#leader-jobs)). The fence lifts at the next renewal that works. `admin control list` shows the rows. | 5 s |
| 4 | `events` | Reads the change events of the control store from the head ([Control store tables](/reference/control-store/index.html.md#subscribe)). It feeds them to the watches and the lifecycle waits of the machine API in this process ([the machine API](/reference/machine-api/index.html.md#change-events)). So they see each change, also one of another control process or of a node. If the reader stops by itself, `hopboxd` stops. | 2 s |
| 5 | `leader election` | Takes part in the leader election ([Control store tables](/reference/control-store/index.html.md#leader-election)). The leader runs the [leader jobs](#leader-jobs). | 15 s |
| 6 | `api` | Serves the API on `control.api_listen` ([the API server](/reference/api/index.html.md)). | 10 s |
| 7 | `internal listener` | Serves the cluster API over TLS on `control.internal_listen`: enrollment, the control channels of the servers, and `EdgeService`. It also serves the API and the console to the edge servers, for the clients that they forward ([the internal listener](/reference/internal-listener/index.html.md)). | 5 s |
| 8 | `admin socket` | Serves the admin API on `<run_dir>/admin.sock` ([the admin API](/reference/admin-api/index.html.md#the-admin-socket)). | 10 s |
| 9 | `dns` | The DNS role ([the DNS role](/reference/dns/index.html.md)). At its start, it reads the zones on its disk, and serves them if they pass every check ([the DNS role](/reference/dns/index.html.md#zones-on-the-disk)). Then it opens a UDP socket and a TCP socket at each address of `dns.listen`. So a port that another program holds fails the start. Without zones on its disk, it answers `SERVFAIL` to each query until the first zones arrive over the control channel. | 1 s |
| 10 | `edge` | The edge role ([the edge role](/reference/edge/index.html.md)). At its start, it opens the HTTPS listener at `edge.https_listen` and the HTTP listener at `edge.http_listen`. So a port that another program holds fails the start. It refuses each TLS handshake until the first certificate arrives over the control channel. At its stop, it closes the listeners, gives the open requests 2 s, and then ends the rest, also the streams and the WebSockets. | 5 s |
| 11 | `machines` | The node role. It first sets up the node network: it replaces the nftables table `inet hopbox_node` with one that holds the tap devices of the machines that run, and turns on IPv4 and IPv6 forwarding ([the node network](/reference/node-network/index.html.md#at-the-start-and-at-the-stop-of-hopboxd)). Until its stop, it watches the table, and installs it again after a flush. It adds the guest kernel of `vmm.kernel` to the kernel cache. It starts the executors of the machines with the Firecracker driver ([the node executor](/reference/executor/index.html.md), [the Firecracker driver](/reference/firecracker-driver/index.html.md)). It also starts the store of the machine logs ([Machine logs](/reference/machine-logs/index.html.md)). Once the server has a mesh address, it serves the disks of the cold moves on port 7444 of that address ([Cold move](/reference/cold-move/index.html.md#the-disk-listener)). At its start, the executors attach to the microVMs that run. At its stop, the executors end their actions at once. Every microVM keeps running, because it runs in a systemd unit of its own. The unit of each machine runs [stop-guest](#stop-guest) at its stop, and is ordered before `hopboxd.service`. | 10 s |
| 12 | `channel` | Keeps the control channel of this server to one of the live control servers that the control role sent, kept in `<data_dir>/server/control-urls`, or to one of `channel.control_urls`, over mutual TLS. After a lost or refused channel, it dials the next URL ([the control channel](/reference/control-channel/index.html.md#the-agent)). If the control role runs in the same `hopboxd`, the channel still goes over TLS ([the control channel](/reference/control-channel/index.html.md)). It hands the zones that the control role sends to the `dns` unit. It hands the certificate, the routes, and the access changes to the `edge` unit. It hands the specs of the machines to the executors of the `machines` unit, and the machine streams to its handlers. It also keeps the mesh device `hbm0` ([the mesh](/reference/mesh/index.html.md)). It checks the device at its start, before the first heartbeat, and then every 2 s. It hands the device the mesh that the control role sends. At its start, it removes what the IPv6 block of a release before N29 left on the server, and tries again every 2 s until nothing is left ([the egress of the machines](/reference/egress/index.html.md#what-the-ipv6-block-left)). It also keeps the egress tunnels `hbe*`, with a check at its start and then every 2 s ([the egress of the machines](/reference/egress/index.html.md#checks-a-restart-and-a-reboot)). It hands them each mesh. On an edge server, it also checks the uplink at its start and then every 2 s. An edge server says `egress_exit` in each heartbeat only while its tunnel is ready and its uplink works ([the egress of the machines](/reference/egress/index.html.md#the-uplink-of-an-egress-edge)). On an edge server with `edge.public_ipv4`, `edge.floating_addresses`, or `edge.public_ipv6`, it keeps the floating addresses on the uplink and the peer address as the source of the default route, with a check at its start, after each change of an address or a route, and every 2 s ([the edge role](/reference/edge/index.html.md#the-check-of-the-uplink)). Its start returns after these first checks. Before the enrollment of the server, it waits for the identity in `<data_dir>/server`. So `run` can start before [enroll](#enroll). | 5 s |

Unit 1 runs for every role. Units 2 to 8 belong to the control role and unit 9 to the DNS role. Unit 10 belongs to the edge role and unit 11 to the node role. Unit 12 runs on a server that enrolls: one with a role other than `control`. Unit 13, `systemd notify`, runs for every role:

| Order | Unit | What it does | Stop timeout |
| --- | --- | --- | --- |
| 13 | `systemd notify` | Sends `READY=1` to the socket of `NOTIFY_SOCKET` at its start, and `STOPPING=1` at its stop ([sd_notify(3)](https://www.freedesktop.org/software/systemd/man/latest/sd_notify.html)). Without `NOTIFY_SOCKET`, it does nothing. It starts last. So the unit `hopboxd.service` of `Type=notify` becomes active only after every other unit is ready ([Server layout](/reference/server-layout/index.html.md#hopboxdservice)). If it cannot send, the start fails. | 1 s |

On a server with only the node role, `run` starts the metrics listener and the `host firewall` unit. Then it starts the `machines` unit, the `channel` unit, and the `systemd notify` unit. On a server with only the DNS role, it starts the `dns` unit in place of the `machines` unit. On a server with the edge role, it starts the `edge` unit before the `channel` unit. So what the channel gets for these roles always has a server.

`run` stops the units in the reverse order. First it stops `systemd notify`, then `channel`, `machines`, `edge`, and `dns`. Then it stops the units of the control role, from the admin socket to Postgres, then the host firewall, and last the metrics listener. So no unit stops before a unit that needs it. A server can have the control role and a role that enrolls. Then the control role sees the channel of its own server close before its listeners stop. Each stop has the timeout in the table:

- The `channel` unit closes the control channel and waits for its loop and its checks to end. It keeps the device `hbm0` and the egress tunnels. So their traffic goes on while `hopboxd` is down.
- The `machines` unit ends the actions of the executors, closes the guest links, and closes the files of the machine logs. It stops no microVM, and keeps the node network.
- The `edge` unit closes its listeners and gives the open requests 2 s. Then it cancels the rest: the streams, the WebSockets, and the long calls. At the timeout, it closes every connection ([the edge role](/reference/edge/index.html.md)).
- The `dns` unit closes its sockets and waits for the queries in flight.
- The `api`, `internal listener`, and `admin socket` units stop taking new connections and wait for the running unary calls. They end every open stream at once, because a stream has no natural end. They cancel the stream and end every read of its request body. So a stream that waits for the client, such as a reflection stream, ends too. A unary call that still reads its request body gets 2 s more. So a stalled upload cannot hold the stop. At the timeout, they close the rest ([the API server](/reference/api/index.html.md#when-a-listener-stops)).
- The `leader election` unit ends the term of the leader and releases the leader lease.
- The `events` unit stops its reader of change events and closes its connection.
- The `store` unit stops the renewals of the row of this control process, and marks the row stopped, so no check waits for this process. The row stays until the sweep. Then it stops the schema watch and closes its connections.
- The `host firewall` unit ends its watch of the table, and keeps the table.
- The `postgres` unit asks for a fast shutdown. After 20 s, it asks for an immediate shutdown, and 5 s later it kills Postgres. With an external Postgres, it does nothing.

Only the units of the roles of the server run, so the sum of their timeouts depends on the roles. The unit `hopboxd.service` gives `hopboxd` 105 s to stop (`TimeoutStopSec=105s`, [Server layout](/reference/server-layout/index.html.md#hopboxdservice)).

| Roles | Sum of the stop timeouts |
| --- | --- |
| `control` | 83.1 s |
| `control`, `dns` | 89.1 s |
| `control`, `edge` | 93.1 s |
| `control`, `dns`, `edge` | 94.1 s |
| `control`, `node` | 98.1 s |
| `control`, `node`, `dns` | 99.1 s |
| `control`, `node`, `edge` | 103.1 s |
| `control`, `node`, `dns`, `edge` | 104.1 s |
| `dns` | 12.1 s |
| `edge` | 16.1 s |
| `dns`, `edge` | 17.1 s |
| `node` | 21.1 s |
| `node`, `dns` | 22.1 s |
| `node`, `edge` | 26.1 s |
| `node`, `dns`, `edge` | 27.1 s |

Each sum is under the 105 s of `hopboxd.service`.

If the install has no key-encryption key yet, the internal listener starts all the same. It then has no certificate, and logs `the internal listener has no certificate yet`. Until root makes the first key with [admin kek rotate](#admin-kek-rotate), it fails each TLS handshake ([the internal listener](/reference/internal-listener/index.html.md)).

If `run` gets `SIGINT` or `SIGTERM` while a unit starts, it stops the units that started, in the reverse order, and exits with status 0.

If a unit fails to start, `run` stops the units that started, in the reverse order, and exits with status 1. If a unit fails while it runs, for example when the API listener stops, `run` stops every unit and exits with status 1. A supervisor such as systemd then restarts `hopboxd`. A crash of Postgres alone does not stop `hopboxd`, because the supervisor of Postgres restarts it ([the bundled Postgres](/reference/postgres/index.html.md#start-restart-and-stop)).

### Leader jobs

The leader of the control role runs these jobs. In maintenance mode, the leader pauses every job, and keeps its lease ([the API server](/reference/api/index.html.md#maintenance-mode)). It logs `maintenance mode paused the leader jobs` at level info, with `component=leader`. A fence of the process also pauses every job ([Run](#run)). If the fence lasts longer than one lease TTL (15 s), the leader gives up its lease, so another control process can lead. It logs `this process stays fenced, so it gives up the leader lease to a process that can run the leader jobs`. While it is fenced, the process does not try for the leader lock. When maintenance mode ends, the jobs start again as at the start of a term, with the sweep first and 20 s without a check of silent servers. The sweep runs once at the start of each term of the leader, and then every 10 minutes. The check of silent servers runs every 5 s. The auto-delete of idle machines and the return of SSH host keys run every minute. The check of silent servers skips the first 20 s of each term. In that time, the servers can dial again after a stop of the control role, before the leader marks any of them.

| Job | Every | What it does |
| --- | --- | --- |
| Sweep of idempotency keys | 10 min | Deletes the expired idempotency keys ([Control store tables](/reference/control-store/index.html.md#sweep-the-expired-keys)). |
| Sweep of sign-ins | 10 min | Deletes the ended sessions, and the magic links, login links, and OAuth flows that are older than 24 hours ([Sessions, cookies, and sign-in links](/reference/sessions/index.html.md#sweep)). Deletes the expired CLI tokens, the ended CLI sign-ins, and the device sign-ins and failed device approvals that are older than 24 hours. Deletes the API keys and admin API keys that expired more than 30 days ago ([CLI tokens, API keys, admin API keys, and SSH keys](/reference/credentials/index.html.md#sweep)). |
| Sweep of control processes | 10 min | Deletes the rows of the control processes that were not renewed for 24 hours. They ended without their stop ([Control store tables](/reference/control-store/index.html.md#control_processes)). |
| Sweep of the fleet | 10 min | Deletes the enrollment tokens that expired more than 1 day ago, and the server certificates that expired more than 1 day ago. It keeps the newest certificate of each server ([Control store tables](/reference/control-store/index.html.md#enrollment_tokens)). |
| Check of silent servers | 5 s | Marks `unresponsive` each `ready`, `cordoned`, `draining`, or `shutting_down` server whose last heartbeat is 20 s old or older: 4 missed heartbeats. The write checks the leader lease, so a leader that lost its lease writes nothing. Each server that it marks gets an audit entry `server.mark_unresponsive` with the actor `system` ([Control store tables](/reference/control-store/index.html.md#servers)). The next heartbeat of the server makes it `ready` again, or `cordoned`, or `draining` when the server has a drain ([the control channel](/reference/control-channel/index.html.md)). |
| Step of the drains | 5 s | Takes the drain of each `draining` or `shutting_down` server one step on: it starts the cold moves off the server, at most 2 at once, names the machines that cannot move, and runs the empty checks ([Cold move](/reference/cold-move/index.html.md#drain-of-a-node)). The first empty check turns the server `shutting_down`, with the audit entry `server.mark_shutting_down`. A check that finds it not empty turns it `draining` again, with `server.mark_draining`. The third empty check in a row, one minute after the second, decommissions the server, with the audit entry `server.decommission`. Both entries have the actor `system`. Every write checks the leader lease. |
| Step of the removals of the node role | 5 s | Takes each pending removal of the role `node` one step on, after the step of the drains. It keeps the server cordoned, and starts the cold moves off the server, at most 2 at once. It names the machines that cannot move ([Cold move](/reference/cold-move/index.html.md#removal-of-the-node-role)). It runs no empty check and no decommission. Every write checks the leader lease. |
| Sweep of invitations | 10 min | Deletes the invitations to orgs that ended more than 24 hours ago, by a use, a revoke, or the expiry. It also deletes the records of their rate limits that are older than 24 hours ([the org API](/reference/org-api/index.html.md#invitations)). |
| Sweep of machines | 10 min | Frees the machine names and the SSH ports whose reservation of 7 days ended ([Domain rules](/reference/domain-rules/index.html.md#name-reservation)). Drops the row of each machine that its executor deleted, 7 days after its delete request. Then deletes the machine lease of each machine whose row is gone ([Control store tables](/reference/control-store/index.html.md#machines)). Last, it frees the subnet of a decommissioned node once no machine address of it is held ([Control store tables](/reference/control-store/index.html.md#node_subnets)). Each of these five steps runs in a transaction of its own that checks the leader lease. So a leader that lost its lease frees nothing more. |
| Prune of change events | 10 min | Deletes the change events that are older than 24 hours. Every reader of the events starts from the head, so an older event serves nobody ([Control store tables](/reference/control-store/index.html.md#events)). |
| Certificate of the install | when due | Gets the wildcard certificate of the install from the ACME CA, and renews it after two thirds of its lifetime. It starts at the start of each term, and plans again after each change of an input of the ACME job ([the certificate of the install](/reference/acme/index.html.md#the-acme-job)). Every write checks the leader lease. |
| Auto-delete of idle machines | 1 min | Deletes each machine whose last recorded use is older than its auto-delete timeout plus 1 minute. A use is recorded at most once a minute ([the machine API](/reference/machine-api/index.html.md#auto-delete)). It takes at most 100 machines per run, the oldest last use first. The next run takes the rest. It skips the machines of a suspended org. Each machine goes in a transaction of its own, which checks the leader lease, so a leader that lost its lease deletes nothing. The delete reserves the name and the SSH port, as `DeleteMachine` does. Each delete gets an audit entry `machine.delete` with the actor `system` and the detail `reason` (`auto-delete`), and the log record `deleted an idle machine`. A machine whose delete fails gets the log record `deleting an idle machine failed`, and the job goes on with the next one. Only a lost leader lease stops the run. |
| Return of SSH host keys | 1 min | Gives each SSH host key that a route record claims, and that no record lists, to the oldest record that claims it. A report that lets a key go gives it at once, but it skips a record that another transaction locks, and a `hopboxd` of v0.1.0 gives no key. It takes at most 100 keys per run, and each key goes in a transaction of its own. The write needs no leader lease, because a report makes the same write. Each key gets the log record `a machine got an SSH host key that no other machine holds` at level info, with `machine` ([Control store tables](/reference/control-store/index.html.md#route_host_keys)). |

### Example

```sh
sudo hopboxd run --config /etc/hopbox/hopboxd.toml
```

[Run the control role on a laptop](/guide/run-hopboxd/index.html.md) runs the control role on a laptop. [Enroll the node role on one server](/guide/enroll-node/index.html.md) enrolls the node role of a server with the control role of the same server. [Serve the domain of an install with the DNS role](/guide/dns-role/index.html.md) runs the DNS role.

## enroll

```sh
hopboxd enroll --token=TOKEN [--config=PATH] [--json]
```

`enroll` makes this server a part of an install ([Enrollment and the install PKI](/reference/enrollment/index.html.md)). It takes these steps:

1. It reads the bootstrap config. The config must list a role that enrolls: every role but `control`. Today that is `node`, `dns`, `edge`, or more than one.
2. It checks the form of the token.
3. If the config lists `node`, it runs the [preflight check](/reference/preflight/index.html.md) and prints each result to standard error. A failed check of severity `fail` stops the enrollment, before any call. A failed check of severity `warn` only prints.
4. It reads the hardware of the server: the CPU model, the CPU threads, the RAM, and the whole disks ([Enrollment and the install PKI](/reference/enrollment/index.html.md#hardware-of-a-pool)).
5. It makes its own ECDSA P-256 key and a certificate request, and saves them with the token and the roles in `<data_dir>/server/pending.pem` before the call. If that file holds a pending enrollment for the same token and roles, it sends that request again instead, so a server that lost the answer gets its certificate ([Enrollment and the install PKI](/reference/enrollment/index.html.md#retry-after-a-lost-answer)). The private key never leaves the server.
6. It dials the first URL of `channel.control_urls`. The TLS handshake checks that the control listener has a certificate of the CA whose hash the token holds, before the token leaves the server. Only if a dial fails does it go on to the next URL, in the order of the list. A call that reached a control server can have used the token, so the next URL never gets it.
7. It calls `EnrollmentService.Enroll` with the token, the request, the host name, the roles, and the hardware. The roles are the roles of the config without `control`.
8. It checks the answer, writes the identity to `<data_dir>/server`, and removes `pending.pem`.

The whole command has 60 s. A running `hopboxd run` with a role other than `control` finds the new identity within 2 s and opens the control channel.

| Flag | Type | Default | Description |
| --- | --- | --- | --- |
| `--token` | string | none, required | The enrollment token, `hbxe_<id>_<secret>_<cahash>`, from [admin enrollment-token create](#admin-enrollment-token-create). The environment variable `HOPBOX_ENROLLMENT_TOKEN` gives it too, so the token stays out of the process list. The flag wins over the variable. |
| `--config` | path | `/etc/hopbox/hopboxd.toml` | The bootstrap config file. |
| `--json` | bool | `false` | Prints one JSON object with the identity of the server instead of its ID. |

On success, `enroll` prints the server ID to standard output, and the line `hopboxd: enrolled as the server <id>. hopboxd run connects it to the control plane.` to standard error. With `--json`, it prints one object with these keys:

| Key | Description |
| --- | --- |
| `server_id` | The ID of the new server. |
| `roles` | The roles in its certificate, for example `["node"]` or `["node","dns"]`. |
| `ca_hash` | The SHA-256, in hex, of the certificate of the install CA. |
| `certificate_expire_time` | The end of the server certificate, 30 days after the enrollment, in UTC. |
| `dir` | The dir of the identity, `<data_dir>/server`. |

Example:

```text
$ sudo HOPBOX_ENROLLMENT_TOKEN="$TOKEN" hopboxd enroll
preflight kvm	ok
preflight cgroup2	ok
preflight tun	ok
preflight nftables	ok
preflight reflink	ok
preflight kernel	ok
hopboxd: enrolled as the server 01a14404-ce7b-72d9-b18e-5006c3e462db. hopboxd run connects it to the control plane.
01a14404-ce7b-72d9-b18e-5006c3e462db
```

A server that is enrolled already, and whose config lists a role beyond those of its identity, adds the new roles in place ([Enrollment and the install PKI](/reference/enrollment/index.html.md#add-roles-to-an-enrolled-server)). The token must allow each new role. A server whose config leaves out a role of its identity ends the removal of that role instead ([Enrollment and the install PKI](/reference/enrollment/index.html.md#remove-roles-from-an-enrolled-server)). The token must allow each role that stays. The server keeps its ID. `enroll` then prints `hopboxd: the server <id> has the roles <roles> now. Restart hopboxd to use them: systemctl restart hopboxd`. `hopboxd install` restarts it on its own. On the installed server, `enroll` takes the lock of the install, and writes the marker `/opt/hopbox/.restart-pending` before the call to the control role. It keeps the marker once the identity on disk changed. If an install, an upgrade, or another `enroll` holds the lock, `enroll` stops before the call with `another hopboxd install, upgrade, or enroll runs on this server. Wait for it to end`. So if you forget the restart, or `enroll` stops after the save, the next `hopboxd install` restarts `hopboxd`. A restart by hand leaves the marker, so that install restarts `hopboxd` once more. The installed server is the one whose config `/etc/hopbox/hopboxd.toml` names the same `data_dir` as `--config`.


| Error | Cause |
| --- | --- |
| `the config <path> lists no role that enrolls. Add "node", "dns", "edge", or more than one to roles` | The config lists only `control`. |
| `the token does not have the form hbxe_<id>_<secret>_<cahash>. Make one with hopboxd admin enrollment-token create` | The token is not an enrollment token. |
| `the preflight check failed, so this server cannot run machines:` and one line for each failed check | A check of severity `fail` failed ([the preflight check](/reference/preflight/index.html.md)). On a Mac, every check fails, as it must. |
| `this server is enrolled already as the server <id> with the roles <roles>. To enroll it again, remove <dir> first` | `<data_dir>/server` holds an identity with the roles of the config. |
| `enroll at <url>: invalid_argument: invalid argument: the roles drop <roles>, and the server <id> has no removal of them. Start one with hopboxd admin server remove-role` | The config lists fewer roles than the identity, and no removal of the missing roles is pending ([the admin API](/reference/admin-api/index.html.md#removeserverroles)). |
| `pki: the peer is not the control plane of this install: ...` inside `enroll at <url>: ...` | The control listener has no certificate of the CA that the token names. The token never left the server. |
| `enroll at <url>: unauthenticated: the enrollment token is not valid...` | The control role refused the token: it is unknown, it expired, it was used, or it was made for another CA ([Enrollment and the install PKI](/reference/enrollment/index.html.md)). |
| `enroll at <url>: unavailable: ...` | No control listener answers at this URL of `channel.control_urls`, or its TLS handshake failed, for example before the install has a key-encryption key. |
| several lines `enroll at <url>: ...`, one for each URL | Every dial failed. The error joins the failures, in the order of the list. |

## preflight

```sh
hopboxd preflight [--config=PATH] [--json]
```

`preflight` runs the [preflight check](/reference/preflight/index.html.md) of this server: KVM, cgroup v2, tun, nftables, reflinks on the data dir, and the tested OS and kernel. It reads the bootstrap config only for `data_dir`. It runs every check, also after a failure, and prints one line for each:

```text
preflight <name>	ok
preflight <name>	<severity>	<message>
```

`<severity>` is `fail` or `warn`. If a check of severity `fail` failed, the command exits with status 1 and the error `the preflight check failed, so this server cannot run machines`. A failed `warn` check alone exits with status 0.

| Flag | Type | Default | Description |
| --- | --- | --- | --- |
| `--config` | path | `/etc/hopbox/hopboxd.toml` | The bootstrap config file. |
| `--json` | bool | `false` | Prints one JSON object with `passed` and `checks`. `passed` is false after a failed check of severity `fail`. `checks` is a list of objects with `name`, `severity`, `passed`, and `message`. The `message` of a check that passed is empty. |

Example on a Mac:

```text
$ hopboxd preflight --config /tmp/hb/hopboxd.toml
preflight kvm	fail	hopbox runs machines only on Linux with KVM, and this host runs darwin
...
hopboxd: error: the preflight check failed, so this server cannot run machines
```

## install

```sh
sudo ./hopboxd install --roles=ROLE,... [--domain=DOMAIN] [--bootstrap-admin-key-hash=HEX]
  [--enroll-token=TOKEN] [--control-url=URL] [--label=KEY=VALUE ...] [--dns-listen=HOST:PORT ...]
  [--edge-public-ipv4=ADDRESS] [--edge-floating-address=ADDRESS,...] [--edge-no-floating] [--peer-address=ADDRESS] [--internal-listen=ADDRESS:PORT] [--json]
```

`install` installs the release of the `hopboxd` that runs it. That release is the dir of the binary, with its `manifest.json` ([Release bundle](/reference/release-bundle/index.html.md)). For a first install, run the `hopboxd` of an unpacked bundle. After that, `sudo hopboxd install` runs the current release. `install` must run as root on Linux. The Terraform provider runs it on each new server. [Install hopbox on a Debian 13 server by hand](/guide/install-debian/index.html.md) runs it by hand.

An install is idempotent: a second run with the same flags changes nothing and does not restart `hopboxd`. A run from a new release is the last step of an [upgrade](#upgrade). [Server layout](/reference/server-layout/index.html.md) lists every path, user, unit, and setting that it makes.

It takes these steps, in order:

1. It checks the flags. On an enrolled server whose `--roles` adds or drops an enrolling role, it checks the change of the roles ([Remove roles](#remove-roles)). It takes the lock `/opt/hopbox/.lock`, or the lock that an upgrade passes to it. If another install or upgrade holds the lock, the install stops. It checks the release dir against its manifest.
2. If the bootstrap config `/etc/hopbox/hopboxd.toml` exists, it reads the file with the owner and mode checks of `hopboxd run` ([the bootstrap config of hopboxd](/reference/hopboxd-config/index.html.md#file)). Then it applies the flags. The table `dns` goes with the role `dns`, and the table `edge` with the role `edge`, when the file listed the role and `--roles` does not. With `--internal-listen` on a server with a role that enrolls, the URL of the new listener takes the place of each loopback URL of `channel.control_urls` ([the internal listener](/reference/internal-listener/index.html.md#move-the-first-server-to-its-peer-address)).
3. With the control role and no Postgres cluster yet, the install makes the control store of this server, so it needs `--domain`. With `postgres.url`, the install connects to the external Postgres instead. If its database has no table `schema_migrations`, the install counts as the first one, and it needs `--domain`. An external Postgres that the install cannot reach, for example because of a wrong CA, stops the install here ([the bundled Postgres](/reference/postgres/index.html.md#external-postgres)).
4. With the node role, it checks that `data_dir` is on XFS with reflinks. The DNS role needs no XFS.
5. It checks that `/usr/local/bin/hopboxd` is a symlink or does not exist.
6. It makes the users: `hopbox-pg` for the control role without `postgres.url`, and `hopbox-jail` with the IDs of the jailer for the node role. It refuses IDs of the jailer that another entry of `/etc/subuid`, `/etc/subgid`, `/etc/passwd`, or `/etc/group` uses ([Server layout](/reference/server-layout/index.html.md#users-and-ids)).
7. It makes the dirs of the roles: `<data_dir>/server` for a role that enrolls, and the dirs of the machines for the node role. It refuses a dir that is a symlink or that another user owns, and it sets the mode of each dir ([Server layout](/reference/server-layout/index.html.md#paths)).
8. With the node role, it runs the [preflight check](/reference/preflight/index.html.md) and prints each result to standard error. A failed check of severity `fail` stops the install.
9. If `/opt/hopbox/releases/<version>` does not exist, it copies the release there, and checks that the copy has the manifest of the source.
10. With the control role, it checks that the Postgres of the release has the major version of the cluster in `PG_VERSION` ([the bundled Postgres](/reference/postgres/index.html.md#upgrades)). With `postgres.url`, it skips that check. It also makes the schema check of [upgrade](#upgrade) with the migrations that this `hopboxd` knows, on the external Postgres when the config names one. So a `hopboxd install` of an older release dir, run by hand, cannot switch into a schema that does not run with it.
11. With the node role, it puts the default image of the release into the image cache. It removes the disk of the image from the release dir. If the cache has a dir of that image without `artifact.json`, it adds the file of the release. A node that downloads the disk of an image leaves such a dir ([the Firecracker driver](/reference/firecracker-driver/index.html.md#the-image-cache)).
12. With the control role, it does three things. If the keyring has no key, it makes the first key-encryption key. On the install of step 3, it writes the hash of `--bootstrap-admin-key-hash` to a file. A later run ignores the flag and logs a warning. If no cluster exists and the config has no `postgres.url`, it runs `initdb`.
13. With the node role, it reserves the SSH ports 10000 to 30000.
14. On an enrolled server whose `--roles` adds or drops an enrolling role, it writes the marker `/opt/hopbox/.restart-pending`. Then it calls `Enroll` with the token and saves the new identity ([Remove roles](#remove-roles)). Without `--enroll-token`, it takes the token of a pending enrollment for the same enrolling roles ([Enrollment and the install PKI](/reference/enrollment/index.html.md#retry-after-a-lost-answer)). So a run that stops right after the save still restarts `hopboxd` at the next run. If the call fails, the install stops. It removes the marker first, but only when the identity on disk did not change and the marker was not there before this run. A call can fail after it saved the new identity, for example at the sync of the server dir. Then the marker stays, and the next run restarts `hopboxd` with the new identity. Only a change of the roles, and an install of a server that has no identity yet, read `<data_dir>/server/pending.pem`. So a damaged file stops no other install, such as an upgrade of an enrolled server. An install that reads a damaged file stops with `node: <path> must hold one PRIVATE KEY block and one CERTIFICATE REQUEST block with the token, the roles, and the end. Remove it to enroll with a new key`.
15. If the release, the config, a unit, or the roles are about to change, it writes the marker `/opt/hopbox/.restart-pending`. If the bootstrap config changed, it then writes the file.
16. It points `/opt/hopbox/previous` to the release that ran before, and then `/opt/hopbox/current` to the release. In this order, a crash between the two renames never loses the release of a rollback. It points `/usr/local/bin/hopboxd` to `/opt/hopbox/current/hopboxd`.
17. It writes each unit that changed. If a unit changed, or the marker was there before this run, it runs `systemctl daemon-reload`. Then it runs `systemctl enable hopboxd.service`.
18. If the release, the config, a unit, or the roles changed, or the marker was there before this run, it runs `systemctl restart hopboxd.service`. Otherwise it runs `systemctl start hopboxd.service`. If `hopboxd` runs, that start does nothing. Both wait until `hopboxd` is ready. After a good start, it removes the marker.
19. It removes the releases other than the current and the previous one.
20. With the control role, it reads the domain of the install through the admin socket. If `--domain` names another domain, it sets it. Without `--domain`, an install without a domain fails.
21. If the server has a role other than control and no identity yet, it enrolls with the enrollment token, as [enroll](#enroll) does. If the config names an enrolling role beyond the identity, it adds the role with the token and restarts `hopboxd`. It writes the marker `/opt/hopbox/.restart-pending` before that call, and removes it after the restart. So a run that stops after the save restarts `hopboxd` at the next run, as in step 14. Without `--enroll-token`, both take the token of a pending enrollment for the enrolling roles of the config ([Enrollment and the install PKI](/reference/enrollment/index.html.md#retry-after-a-lost-answer)).
22. With `--peer-address`, it reads the server of the identity through the admin socket. If the server has another peer address or none, it sets the address with `SetServerPeerAddress` ([the admin API](/reference/admin-api/index.html.md#setserverpeeraddress)).

Steps 1 to 13 make every check that can refuse. They change nothing that a running `hopboxd` reads: the identity, the config, the current release, and the units stay the same until step 14. So a refused install leaves the running `hopboxd` as it was. If a run stops after the marker of step 14 or step 15 and before the restart, the marker stays. The next run then restarts `hopboxd`, also without another change. The install never removes data.

| Flag | Type | Default | Description |
| --- | --- | --- | --- |
| `--roles` | list of strings, joined by commas | none, required | The roles of this server: one or more of `control`, `node`, `dns`, and `edge`. The order does not matter. The config lists them in the order `control`, `node`, `dns`, `edge`. |
| `--domain` | string | none | The domain of the install, such as `example.com`. It must follow the rules in [Domain rules](/reference/domain-rules/index.html.md#install-domain). It needs the control role. The install that makes the Postgres cluster needs it. A later run without it keeps the domain. |
| `--bootstrap-admin-key-hash` | string | none | The SHA-256 of the first admin API key of the install, as 64 lowercase hex digits. It needs the control role. Only the install that makes the Postgres cluster writes it, to `<data_dir>/bootstrap-admin-key.sha256` ([Server layout](/reference/server-layout/index.html.md#the-bootstrap-admin-api-key)). A later run ignores it. |
| `--enroll-token` | string | none | An enrollment token, `hbxe_<id>_<secret>_<cahash>`. After the start, a server with a role other than `control` enrolls with it, unless it is enrolled. If `--roles` adds an enrolling role to an enrolled server, the install adds that role with the token. That step comes before the install writes the config ([Remove roles](#remove-roles) has the steps). Without a token, such an install takes the token of a pending enrollment for the same roles, after a lost answer ([Enrollment and the install PKI](/reference/enrollment/index.html.md#retry-after-a-lost-answer)). Without either, it fails before any change. If the config does not load before the install, the install adds the role after it writes the config. It then restarts hopboxd, so the control channel uses the new certificate. That install fails without a token. The error starts with `the config names the roles <roles>, but the server <id> enrolled with <roles>`. Give it in the environment variable `HOPBOX_ENROLLMENT_TOKEN` instead, so the token stays out of the process list. The flag wins over the variable. |
| `--control-url` | URL, repeatable | the value of the config | The URL of the internal listener of a control server. Repeat the flag for each control server, or join the URLs with commas. With one URL, the install writes the old key `channel.control_url`, unless the config has `channel.control_urls` already. So a rollback to a `hopboxd` from before `control_urls`, which refuses that key, still reads the file. With more than one URL, it writes `channel.control_urls` and deletes `channel.control_url` ([the bootstrap config of hopboxd](/reference/hopboxd-config/index.html.md#keys)). With a token, the enrollment tries the URLs in order, as [enroll](#enroll) does. It needs a role that enrolls: `node`, `dns`, `edge`, or more than one. |
| `--label` | `KEY=VALUE` | the labels of the config | A label of the server, for `channel.labels`. Repeat the flag for each label. With one, it replaces every label of the config. It needs a role that enrolls: `node`, `dns`, `edge`, or more than one. |
| `--dns-listen` | `HOST:PORT` | the value of the config | An address of the DNS role, for `dns.listen`, such as `203.0.113.5:53`. Repeat the flag for each address. With one, it replaces the list of the config. The rules of `dns.listen` apply ([the bootstrap config of hopboxd](/reference/hopboxd-config/index.html.md#keys)). It needs the role `dns`. A change of the list restarts `hopboxd`, as a change of any key of the config does. The Terraform provider runs it after a change of the floating addresses of a server with the DNS role ([hopbox Terraform provider](/reference/terraform-provider/index.html.md#edge-servers-and-floating-addresses)). |
| `--edge-public-ipv4` | `ADDRESS` | the value of the config | The floating IPv4 address of this edge server, for `edge.public_ipv4`, such as `203.0.113.101`. The egress of the machines leaves from it ([the egress of the machines](/reference/egress/index.html.md#the-source-addresses)). The rules of `edge.public_ipv4` apply ([the bootstrap config of hopboxd](/reference/hopboxd-config/index.html.md#keys)). It needs the role `edge`. A change of the key restarts `hopboxd`, and the same value changes nothing. The Terraform provider runs it after a change of the floating addresses of an edge server ([hopbox Terraform provider](/reference/terraform-provider/index.html.md#edge-servers-and-floating-addresses)). |
| `--edge-floating-address` | `ADDRESS` | the value of the config | A floating IPv4 address of this edge server, for `edge.floating_addresses`, such as `203.0.113.102`. Repeat the flag for each address. With one, it replaces the list of the config. The rules of `edge.floating_addresses` apply ([the bootstrap config of hopboxd](/reference/hopboxd-config/index.html.md#keys)). It needs the role `edge`. A change of the key restarts `hopboxd`. The Terraform provider gives it every floating address of an edge server ([hopbox Terraform provider](/reference/terraform-provider/index.html.md#edge-servers-and-floating-addresses)). |
| `--edge-no-floating` | | false | Removes `edge.public_ipv4` and `edge.floating_addresses` from the config, for an edge server that holds no floating IPv4 address any more. It takes neither `--edge-public-ipv4` nor `--edge-floating-address`. A change of the config restarts `hopboxd`. |
| `--internal-listen` | `ADDRESS:PORT` | the value of the config | The address of the internal listener, for `control.internal_listen`: the peer address of this server and port 7443, such as `203.0.113.12:7443` ([the internal listener](/reference/internal-listener/index.html.md#move-the-first-server-to-its-peer-address)). The address must be a peer address that an interface of the host holds, and it must equal `--peer-address` when both flags come. Without `--peer-address`, on a server whose roles enrolled, the install reads the peer address of the server through the admin socket, before any change, and the address must equal it. The other servers allow the stored peer address, and the control URLs name the listener, so the two must be one address. On a server with a role that enrolls, the URL `https://<ADDRESS:PORT>` takes the place of each loopback URL of `channel.control_urls`, or of the default `https://127.0.0.1:7443`, so the roles of the server dial the listener at its new address. It needs the control role. A change restarts `hopboxd`, and the same value changes nothing. The machines keep running. |
| `--peer-address` | `ADDRESS` | none | The peer address of this server: one public unicast IPv4 address on its uplink, such as `203.0.113.12` ([Enrollment and the install PKI](/reference/enrollment/index.html.md#peer-addresses)). It is for the first server, whose roles enroll with a token from its own control role. After the enrollment, the install sets the address through the admin socket, and writes the audit entry `server.set_peer_address`. The same address again changes nothing. It needs the control role and a role that enrolls. A server without the control role gets its peer address from its enrollment token. The install does not keep the address in the config. |
| `--json` | bool | `false` | Prints one JSON object instead of the version. |

### Remove roles

`hopboxd install --roles <every role that stays>` with `--enroll-token` ends the removal of roles from an enrolled server ([Enrollment and the install PKI](/reference/enrollment/index.html.md#remove-roles-from-an-enrolled-server)). An add of a role that a removal took runs the same steps with every role, the old ones and the new ones ([Enrollment and the install PKI](/reference/enrollment/index.html.md#add-roles-to-an-enrolled-server)). `RemoveServerRoles` starts the removal, and the token names the server and the enrolling roles that stay ([the admin API](/reference/admin-api/index.html.md#removeserverroles)). On a control server, `--roles` keeps `control`, such as `--roles control,node`. First the install makes each check that can refuse, such as the lock, the release, and the preflight check of the node role. Then it calls `Enroll` with the token and the enrolling roles that stay, and saves the new identity. It calls the control role at `--control-url`, or at `channel.control_urls` of the config. Then the install writes the config without the roles, and restarts `hopboxd` with the new identity. A refusal of a check or of the control role changes nothing on the server: the identity, the config, and the roles stay. If a later step of the install fails, the install still restarts `hopboxd`, because the registry refuses the old certificate at once. The log of that restart is `the install failed after the change of the roles, so hopboxd restarts with the new identity`, with `component=install` and the field `err`. If the call of the control role fails and the install cannot remove the marker `/opt/hopbox/.restart-pending` of step 14, it logs `the restart marker stays, so the next install restarts hopboxd` at level warn, with the field `err`. If the answer of the control role gets lost, the server keeps its old identity, which the registry refuses. Run the same install again within the lifetime of the token, without `--enroll-token` or with the same token. Do not give a new token: a new token replaces `pending.pem` with a new key, so the key of the used token is gone, and no call can give the server its certificate. The server sends the request of `<data_dir>/server/pending.pem` again, and gets the same certificate ([Enrollment and the install PKI](/reference/enrollment/index.html.md#retry-after-a-lost-answer)). Without a token, and without a pending enrollment for the roles, the install refuses before any change:

- `--roles drops the roles <roles> of the server <id>. Start their removal with hopboxd admin server remove-role, and once it is ready, give a token for the server and the roles that stay with --enroll-token`, without a token
- `--roles names no role that enrolls, but the server <id> has the roles <roles>, and its certificate needs one. A drain takes a whole server out: hopboxd admin server drain <id>`
- `--roles adds roles to the server <id> and drops <roles>. Change the roles one change at a time`
- `--roles adds the roles <roles> to the server <id>, which enrolled with <roles>. Give an enrollment token for the server and the new roles with --enroll-token`, without a token

At its start, `hopboxd run` removes the data of each role that neither the config nor the identity names. For the DNS role, that is the dir `<data_dir>/dns` with the zones ([the DNS role](/reference/dns/index.html.md#zones-on-the-disk)). For the node role, that is the table `inet hopbox_node` and the tap devices `hbt*` ([the node network](/reference/node-network/index.html.md#subnet-of-a-server-without-the-node-role)). The edge role keeps the certificate of the install, its key, and the SSH host key of the install only in memory. So the restart drops them ([the edge role](/reference/edge/index.html.md)). The host firewall opens only the ports of the roles of the config. So the ports of a role that went close at the restart ([the host firewall](/reference/host-firewall/index.html.md)). The log records, with `component=roles`:

| Message | Level | Fields |
| --- | --- | --- |
| `removed the data of a role that this server no longer has` | info | `role`, `removed` |
| `cannot remove the zones of the DNS role that this server no longer has` | warn | `path`, `err` |
| `cannot remove the node network of the node role that this server no longer has` | warn | `removed`, `err` |
| `cannot read the identity of this server, so it keeps the data of every role` | warn | `err` |

A failure keeps `hopboxd` running, and the next start tries again.

### The config file of an install

The install keeps every key of an existing config that its flags do not set. When it changes the file, it drops the comments of the file, and it writes this header:

```toml
# The bootstrap config of hopboxd. hopboxd install keeps the keys that its
# flags do not set. docs/reference/hopboxd-config.md lists every key.
roles = ["control", "node"]
```

On success, `install` prints the version of the release to standard output. It prints a line such as `hopboxd: installed v0.1.0, and hopboxd runs. Changed: release, config.` to standard error. With `--json`, it prints one object with these keys:

| Key | Description |
| --- | --- |
| `version` | The version of the release that runs now. |
| `previous` | The version that ran before this install, or `""` on a first install. On a second run, it is the same as `version`. |
| `changes` | What the install changed, as a list of words. The words of files are `release files`, `config`, `keys`, and `bootstrap admin key`. The other words are `users`, `postgres`, `ports`, `release`, `units`, `roles`, and `peer address`. An install that changed nothing gives `[]`. |
| `restarted` | Whether the install restarted `hopboxd`. |
| `server_id` | The ID of the server, after an enrollment, or `""` before one. |

Example of a first install of a server with the control role and the node role:

```text
$ tar -xzf hopbox-v0.1.0-linux-amd64.tar.gz
$ sudo ./hopbox-v0.1.0/hopboxd install --roles control,node --domain example.com
preflight kvm	ok
preflight cgroup2	ok
preflight tun	ok
preflight nftables	ok
preflight reflink	ok
preflight kernel	ok
hopboxd: installed v0.1.0, and hopboxd runs. Changed: users, release files, keys, postgres, ports, config, release, units.
v0.1.0
```

The log records go to standard error between these lines, as JSON ([Logs](#logs)).

| Error | Cause |
| --- | --- |
| `hopboxd install runs only on a Linux server, and this host runs <os>` | The host is not Linux. |
| `hopboxd install must run as root. Run it with sudo` | The user is not root. |
| `--roles: unknown role "<role>". ...` or `--roles: list at least one role, such as control,node` | A bad `--roles`. |
| `--domain is required on the first install of the control role on a server` | The control role has no Postgres cluster yet, and `--domain` is missing. |
| `the install has no domain. Run the install again with --domain` | The control store has no domain, and `--domain` is missing. `hopboxd` runs. |
| `<dir> belongs to uid <n>, but root writes and runs what it holds, so it must belong to uid 0. ...` | A dir of the install, such as `/opt/hopbox/releases`, belongs to another user. |
| `<dir> is a symlink, but the install uses only real dirs, ...` | A dir of the install, such as `/var/lib/hopbox`, is a symlink. |
| `<dir> has mode <mode>, which lets group or others change it, ...` | A dir of the host that the install writes into, such as `/etc/systemd/system`, is writable by group or others. |
| `/usr/local/bin/hopboxd is not a symlink, ...` | A regular file is there, for example a `hopboxd` from a build from source. |
| `/etc/subuid gives <user> the IDs <a> to <b>, which overlap the IDs 1879048192 to 1879113727 of the jailer` | Another range takes IDs of the jailer. `/etc/passwd` and `/etc/group` give a similar error for a user or a group with such an ID. |
| `another hopboxd install, upgrade, or enroll runs on this server. Wait for it to end` | Another run holds `/opt/hopbox/.lock`. |
| `--control-url and --label need a role that enrolls: node, dns, edge, or more than one` | `--control-url` or `--label` on a server with only the control role. |
| `--dns-listen needs the role dns` | `--dns-listen` on a server without the role `dns`. |
| `--edge-public-ipv4 needs the role edge` | `--edge-public-ipv4` on a server without the role `edge`. |
| `--edge-floating-address needs the role edge` | `--edge-floating-address` on a server without the role `edge`. |
| `--edge-no-floating removes the floating addresses, so it takes neither --edge-public-ipv4 nor --edge-floating-address` | `--edge-no-floating` with one of those flags. |
| `--domain needs the control role, which holds the settings of the install` | `--domain` on a server without the control role. |
| `--peer-address: invalid peer address: ...` | `--peer-address` is not one public unicast IPv4 address. The rest of the message names the reason. |
| `--peer-address needs the control role. A server without it gets its peer address from its enrollment token: hopboxd admin enrollment-token create --peer-address` | `--peer-address` on a server without the control role. |
| `--peer-address needs a role that enrolls: node, dns, edge, or more than one` | `--peer-address` on a server with only the control role. |
| `--internal-listen needs the control role, whose internal listener it moves` | `--internal-listen` on a server without the control role. |
| `--internal-listen: "<value>" is not an address and a port, such as 203.0.113.11:7443` | The value has no port, or port 0. |
| `--internal-listen: invalid peer address: ...` | The address is not one public unicast IPv4 address. The rest of the message names the reason. |
| `--internal-listen: the listener goes on the peer address <address> of --peer-address, not on <address>` | The two flags name two addresses. |
| `--internal-listen: this host holds no address <address>. Give the peer address of this server, which sits on its uplink` | No interface of the host holds the address, so the listener cannot listen there. The install changes nothing. |
| `--internal-listen: the server <id> has the peer address <address>, not <address>. The listener goes on the peer address of the server` | The stored peer address of the server differs. Give its peer address, or change it first with `hopboxd admin server set-peer-address`. The install changes nothing. |
| `--internal-listen: the server <id> has no peer address. Give --peer-address <address> too` | The server enrolled without a peer address. The install changes nothing. |
| `--internal-listen: read the peer address of the server <id> through <socket>: ... Give --peer-address <address> too` | The admin socket did not answer, such as while `hopboxd` is down. With `--peer-address`, the install sets the address after its start instead. The install changes nothing. |
| `--peer-address sets the peer address of the enrolled server, but this server is not enrolled. Give an enrollment token with --enroll-token` | `--peer-address` on a server whose roles did not enroll. The rest of the install is done. |
| `read the peer address of the server <id> through <socket>: ...` or `set the peer address of the server <id>: ...` | The admin call of step 22 failed. Run the install again. |
| `--bootstrap-admin-key-hash is not a SHA-256 in 64 lowercase hex digits` | A bad hash. |
| `--enroll-token does not have the form hbxe_<id>_<secret>_<cahash>. ...` | The token is not an enrollment token. |
| `the release in <dir>: ...` | The dir of the binary is not a release, or it differs from its manifest ([Release bundle](/reference/release-bundle/index.html.md#errors)). |
| `<data_dir> is on <fs>, but the disks of machines need XFS with reflinks there. ...` | The node role, and `data_dir` is on another file system, such as ext4. |
| `<data_dir> is on XFS without reflinks, but the disks of machines need them. ...` | The node role, and the XFS file system has no reflinks. |
| `the preflight check failed, so this server cannot run machines:` and one line for each failed check | A check of severity `fail` failed. |
| `postgres data dir <dir> holds a Postgres <N> cluster, but the bundled binaries are Postgres <M>, so pgsup will not start it` | The release has another major version of Postgres. Nothing changed. |
| `the release dir <dir> holds another build of <version>. ...` | `/opt/hopbox/releases/<version>` holds a release with another manifest. |
| `config: ...` | The config has an error ([the bootstrap config of hopboxd](/reference/hopboxd-config/index.html.md#errors)). |
| `hopboxd <version> did not start: ... See journalctl -u hopboxd. To go back to <previous>, run hopboxd upgrade --rollback.` | `systemctl restart` failed: `hopboxd` exited before it was ready. The second sentence comes only after a switch of the release. |
| `read the domain of the install through <socket>: ...` or `set the domain of the install: ...` | The admin call failed. Run the install again. |

## upgrade

```sh
sudo hopboxd upgrade --bundle=FILE --sha256=HEX [--json]
sudo hopboxd upgrade --rollback [--json]
```

`upgrade` moves this server to the release of a bundle file. `upgrade --rollback` moves it back to the previous release. It must run as root on Linux, on a server with an install. If the `bundle` of a server changes, the Terraform provider runs it.

With `--bundle`, it takes these steps:

1. It reads the bootstrap config and `/opt/hopbox/current`, so it fails on a server without an install.
2. It takes the lock `/opt/hopbox/.lock`. It holds the lock until the install of step 6 ends. It passes the lock to that install as the file descriptor 3, which the environment variable `HOPBOX_INSTALL_LOCK_FD` names. So no other run can prune the new release in between.
3. It checks the sha256 of the bundle file against `--sha256`. The manifest proves only that the bundle is whole. The sha256 from a trusted place, such as the `.sha256` file of the build, proves that it is the bundle that you built.
4. It unpacks the bundle to a temp dir `/opt/hopbox/releases/.incoming-<random>`, and checks the release against its manifest and its owner ([Release bundle](/reference/release-bundle/index.html.md#checks-of-a-release)). It renames the temp dir to `/opt/hopbox/releases/<version>`. If that dir exists with the same manifest, it keeps it. If it exists with another manifest, the upgrade stops.
5. With the control role, it reads the newest schema migration of the control store and the oldest schema version that runs with it. It compares that version with `schema_version` of the new release ([Release bundle](/reference/release-bundle/index.html.md#manifest)). If the `schema_version` of the release is lower than that oldest schema version, the upgrade stops ([the bundled Postgres](/reference/postgres/index.html.md#two-releases-on-one-store)). It reads through the socket of the running Postgres. If no Postgres answers, it starts the Postgres of the current release for the read, and stops it again. If that Postgres does not start, it uses the Postgres of the new release. After the guard of the install, both have the same major version. Each read has 30 s, and each start has the 60 s of pgsup on top.
6. It runs `/opt/hopbox/releases/<version>/hopboxd install --roles=<roles>`, with the roles of the config. The [install](#install) of the new release does the rest: the Postgres guard, the image cache, the switch of `current` and `previous`, the units, and the restart.

With `--rollback`, it takes the same lock and checks the release of `/opt/hopbox/previous` and its owner. It makes the schema check of step 5, and runs the `hopboxd install --roles=<roles>` of that release. The install then points `current` to the previous release, and `previous` to the release that ran.

The old `hopboxd` only unpacks and checks. The new release runs its own install. So each release brings the steps that it needs. The CLI of `install` is the contract between two releases. Until the install of step 6 switches the release, the running `hopboxd` keeps running, and nothing that it reads changes.

The microVMs of the machines run in their own units in `hopbox-machines.slice`, not in `hopboxd.service`. So they keep running through the restart of an upgrade or a rollback ([Server layout](/reference/server-layout/index.html.md#hopbox-machinesslice)).

| Flag | Type | Default | Description |
| --- | --- | --- | --- |
| `--bundle` | path | none | The bundle file, such as `hopbox-v0.2.0-linux-amd64.tar.gz`. |
| `--sha256` | string | none, required with `--bundle` | The sha256 of the bundle file, in hex, from the `.sha256` file of the build. The upgrade refuses a file with another sha256. |
| `--rollback` | bool | `false` | Goes back to the previous release. Give `--bundle` or `--rollback`, not both. |
| `--json` | bool | `false` | Passes `--json` to the install, which then prints its JSON object. |

The output is the output of the install, after the line `hopboxd: <current> runs now. The install of <version> finishes the switch.` on standard error.

Example:

```text
$ sudo hopboxd upgrade --bundle hopbox-v0.2.0-linux-amd64.tar.gz --sha256 "$(cut -d' ' -f1 hopbox-v0.2.0-linux-amd64.tar.gz.sha256)"
hopboxd: v0.1.0 runs now. The install of v0.2.0 finishes the switch.
preflight kvm	ok
...
hopboxd: installed v0.2.0, and hopboxd runs. Changed: release.
v0.2.0
$ sudo hopboxd upgrade --rollback
hopboxd: v0.2.0 runs now. The install of v0.1.0 finishes the switch.
...
hopboxd: installed v0.1.0, and hopboxd runs. Changed: release.
v0.1.0
```

A rollback runs an older `hopboxd` on the control store of the newer one. A schema migration of the newer release can allow the older release, with its line `-- oldest_schema_version`. Then the rollback goes on, and the older `hopboxd` runs on the newer store ([the bundled Postgres](/reference/postgres/index.html.md#two-releases-on-one-store)). A migration that does not allow it makes the older `hopboxd` refuse the database at its start, with `store: the database has a migration that does not run with this hopboxd`. So the schema check of step 5 refuses such a rollback before the switch, and the current release keeps running. Upgrade to a release that knows the migrations instead. v0.1.0 refuses every store with a migration that it does not know, so no rollback from a later release to v0.1.0 passes the check.

After a rollback to a release without `stop-guest`, some units run the older `hopboxd` at their stop: the units of the machines that the later release booted. That `hopboxd` exits with status 80, `unexpected argument stop-guest`. The journal of the unit then shows the usage text of the older `hopboxd` at each stop. You can ignore it. The unit ignores the status, so a shutdown or a reboot in the guest still counts as a shutdown, not as a crash. Until the next boot of the machine, a stop of the server or of the unit ends Firecracker with `SIGTERM`, as before `stop-guest` ([the Firecracker driver](/reference/firecracker-driver/index.html.md#a-stop-through-systemd)).

A build from before the rename of `[node]` to `[channel]` refuses a bootstrap config with `[channel]`. Its error `unknown keys` names the keys of `[channel]`. So before a rollback to such a build, rename `[channel]` back to `[node]` in `/etc/hopbox/hopboxd.toml` ([the bootstrap config of hopboxd](/reference/hopboxd-config/index.html.md#the-old-name-node)).

A release from before the egress tunnels refuses the key `edge.public_ipv4`. A rollback by hand on an edge server with the key fails at the install of that release, with `unknown keys` and the name of the key. So before such a rollback, delete the line `public_ipv4` from the section `[edge]` of `/etc/hopbox/hopboxd.toml`. The older release keeps no rule that guards the forwarding of the egress. So after the rollback, turn off the forwarding that `hopboxd` turned on, and only that, as the Terraform provider does on its own ([hopbox Terraform provider](/reference/terraform-provider/index.html.md#releases-with-and-without-the-key)). The files are in `<data_dir>/hostfw`, `/var/lib/hopbox/hostfw` by default:

- If `forward-guard-ipv6` exists, run `sudo sysctl -w net.ipv6.conf.all.forwarding=0`, then remove the file.
- On a server without the role `node`, if `forward-guard` exists, run `sudo sysctl -w net.ipv4.ip_forward=0`, then remove the file.

```sh
d=/var/lib/hopbox/hostfw
if [ -e "$d/forward-guard-ipv6" ]; then sudo sysctl -w net.ipv6.conf.all.forwarding=0 && sudo rm "$d/forward-guard-ipv6"; fi
if [ -e "$d/forward-guard" ]; then sudo sysctl -w net.ipv4.ip_forward=0 && sudo rm "$d/forward-guard"; fi
```

Run the second line only on a server without the role `node`. An edge server that ran a build of `main` with the egress from before `forward-guard-ipv6` has no such file. After the rollback of such a server, run `sudo sysctl -w net.ipv6.conf.all.forwarding=0` by hand, unless another tool on the server needs IPv6 forwarding.

A node turns on IPv6 forwarding for its machines too, and its table in a release from before the egress tunnels drops IPv6 only to and from the tap devices. So after a rollback of a server with the role `node` to such a release, turn off the IPv6 forwarding that the node network recorded in `<data_dir>/network/forward-guard-ipv6`:

```sh
f=/var/lib/hopbox/network/forward-guard-ipv6
if [ -e "$f" ]; then sudo sysctl -w net.ipv6.conf.all.forwarding=0 && sudo rm "$f"; fi
```

Keep the IPv4 forwarding of a node, because its machines need it. A node that ran a build of `main` with the egress from before this record has no such file. After the rollback of such a node, run `sudo sysctl -w net.ipv6.conf.all.forwarding=0` by hand, unless another tool on the server needs IPv6 forwarding.

A release from before N29 refuses `edge.public_ipv6` without a prefix length, the floating IPv6 address of this release. So before a rollback to such a release, delete the line `public_ipv6` from the section `[edge]` of `/etc/hopbox/hopboxd.toml` ([the bootstrap config of hopboxd](/reference/hopboxd-config/index.html.md#keys)).

A release without the feature `floating-addresses` refuses the key `edge.floating_addresses`. So before a rollback to such a release, delete the line `floating_addresses` from the section `[edge]` of `/etc/hopbox/hopboxd.toml`.

| Error | Cause |
| --- | --- |
| `give --bundle or --rollback` | Neither flag, or both. |
| `--bundle needs --sha256: ...` | `--bundle` without `--sha256`. |
| `another hopboxd install, upgrade, or enroll runs on this server. Wait for it to end` | Another run holds `/opt/hopbox/.lock`. |
| `the control store has the schema migration <n>, which runs only with a hopboxd that knows migration <o> or later, and hopboxd <version> knows the migrations up to <m>. ...` | The release cannot run on the database. Nothing changed. |
| `read the schema version of the control store: ...` | No Postgres answers, and neither the Postgres of the current release nor the one of the new release starts. Nothing changed. |
| `--sha256 checks a bundle file, so it needs --bundle` | `--sha256` with `--rollback`. |
| `hopboxd upgrade runs only on a Linux server, and this host runs <os>` or `hopboxd upgrade must run as root. Run it with sudo` | Not Linux, or not root. |
| `hopbox is not installed on this server: .... Run hopboxd install first` | The config or `/opt/hopbox/current` is missing. |
| `the bundle <file> has the sha256 <got>, not <want>` | `--sha256` does not match. Nothing changed. |
| `bundle: ...` or `the bundle <file>: ...` | The bundle file is broken, unsafe, or differs from its manifest ([Release bundle](/reference/release-bundle/index.html.md#errors)). Nothing changed. |
| `the release dir <dir> holds another build of <version>. Make the bundle with a new version` | Another build of the same version is in place. |
| `this server has no previous release to roll back to` | `--rollback` before the first upgrade. |
| `the install of <version> failed, and <current> keeps running unless its start failed: ...` | The install of the new release failed. Its own error comes before this line. If it failed at the start of `hopboxd`, run `hopboxd upgrade --rollback`. |

### Tests of install and upgrade

| Test | Where it runs | What it checks |
| --- | --- | --- |
| `TestInstallUpgradeRollback`, `TestInstallCompletesADownloadedImage`, `TestInstallRefuses`, `TestFirstInstallNeedsADomain`, `TestInstallReportsAFailedStart`, `TestUpgradeRefuses`, `TestCheckSchema`, `TestInstallExternalPostgres`, `TestPlanConfig`, `TestReservePorts`, `TestLockRefusesASecondRun`, `TestLockAdoptsTheLockOfAnUpgrade`, `TestDirChecks`, `TestCheckIDs` | `make test` | The steps of `internal/install` in temp dirs, with the bundled Postgres and a fake systemd. The tests cover a first install and a second install that changes nothing. They cover an upgrade that keeps the keys and the cluster, a rollback, and the removal of old releases. They also cover the Postgres guard, the schema check (`TestCheckSchema`), also with a newer migration that allows an older release, and each refusal with a config that stays the same. `TestInstallExternalPostgres` installs the control role on a TLS Postgres: a first install without a cluster and without users, the schema check on that Postgres, and a refusal of a wrong CA. Other tests cover the lock and its hand-off, and the checks of dirs and jailer IDs. They also cover a later run with another bootstrap key hash, and a restart that a stopped run left due. |
| `TestSystemdServer` | `make test-install`, and the CI job `install` | A Debian 13 container with systemd as PID 1. It refuses the node role on the overlay root, then installs from an unpacked bundle on a loop XFS. It checks the paths, modes, users, IDs, reserved ports, and `Type=notify`. A second install must not restart `hopboxd`. An install refuses a config of mode 0666, and a releases dir of another user. The upgrade refuses a bundle without `--sha256`, and a release that does not run with the newest schema migration of the store. An install with an enrollment token makes the server `ready`. It runs an upgrade with `--sha256` that keeps the domain and the key, refuses a bundle with Postgres 19, and rolls back. A transient unit in `hopbox-machines.slice` stands in for a microVM, and it keeps its process through every restart. |

`make test-install` needs Docker that runs privileged containers, loop devices, and the kernel module `xfs`. It reads `PG_LINUX`, a Postgres install for `linux/amd64`. Without it, it runs `tools/pgdist -target linux/amd64`. The test is `internal/install/systemd_test.go`. It runs only with `HOPBOX_TEST_SYSTEMD=1`, and it reads the Postgres install from `HOPBOX_TEST_PG_LINUX`, or from `HOPBOX_PG_DIR` on a `linux/amd64` host.

## admin

```sh
hopboxd admin [--config=PATH] [--socket=PATH] <command> [flags]
```

`admin` calls the admin API of the `hopboxd` that runs on this server, through its admin socket ([the admin API](/reference/admin-api/index.html.md#the-admin-socket)). The socket accepts only root and the user that runs `hopboxd`. On a server, `hopboxd` runs as root, so run `admin` with `sudo`. The audit log records each call, with the actor `local_root` ([Control store tables](/reference/control-store/index.html.md#audit_log)).

| Flag | Type | Default | Description |
| --- | --- | --- | --- |
| `--config` | path | `/etc/hopbox/hopboxd.toml` | The bootstrap config. `admin` calls the socket `<run_dir>/admin.sock` of this config ([the bootstrap config of hopboxd](/reference/hopboxd-config/index.html.md#keys)). If the config does not load, `admin` prints `hopboxd: warning: <error>. Using the admin socket /run/hopbox/admin.sock` to standard error and calls that socket, so a bad config never locks you out. A missing default file gets no warning. |
| `--socket` | path | none | The admin socket. It overrides `--config`, and `admin` then reads no config. |

`admin` finds the admin socket in this order:

1. The path of `--socket`.
2. `<run_dir>/admin.sock` of the bootstrap config of `--config`, or of `/etc/hopbox/hopboxd.toml`.
3. If the config does not load, `/run/hopbox/admin.sock`. Then `admin` prints a warning with the error of the config. If the default file does not exist, it prints no warning. Thus a broken config never locks the operator out of the socket.

Each `admin` command waits at most 40 s for its call.

### admin kek rotate

```sh
hopboxd admin kek rotate [--json]
```

Adds a new key-encryption key and makes it the current one. It calls `RotateKEK` ([the admin API](/reference/admin-api/index.html.md#rotatekek)) with a new idempotency key. It prints the ID of the new key.

Use `rotate` only with one control server. A rotation makes the new key current only on this server. So the command fails while the install has a control process on another host, live or not, with `failed_precondition: failed precondition: the install has a control server on <host> too, ...`. With more than one control server, run [admin kek stage](#admin-kek-stage) and then [admin kek promote](#admin-kek-promote).

| Flag | Type | Default | Description |
| --- | --- | --- | --- |
| `--json` | bool | `false` | Prints one JSON object, `{"key_id":"<id>"}`, instead of the ID. |

Example:

```text
$ sudo hopboxd admin kek rotate
file:2
```

### admin kek stage

```sh
hopboxd admin kek stage [--json]
```

Adds a new key-encryption key as a staged key file `<n>.staged` in the key dir of this server. A staged key opens values but seals none, so the current key stays the same. It calls `StageKEK` ([the admin API](/reference/admin-api/index.html.md#stagekek)). It prints the ID of the key, a tab, and the path of the file. Copy that file to the same path on every other control server, then run [admin kek promote](#admin-kek-promote) ([Sealing](/reference/sealing/index.html.md#several-control-servers)). hopbox does not copy the file for you yet. Use `scp` or `rsync` over SSH.

| Flag | Type | Default | Description |
| --- | --- | --- | --- |
| `--json` | bool | `false` | Prints one JSON object, `{"key_id":"file:2","file":"/var/lib/hopbox/kek/2.staged"}`, instead of the line. |

Example:

```text
$ sudo hopboxd admin kek stage
file:2	/var/lib/hopbox/kek/2.staged
```

### admin kek promote

```sh
hopboxd admin kek promote <id> [--json]
```

Makes the staged key `<id>`, such as `file:2`, the current key on every control server. It calls `PromoteKEK` ([the admin API](/reference/admin-api/index.html.md#promotekek)). It prints the ID when every other live control process seals with the key. The command can take longer than a few seconds, because it waits for the next renewal of each control process. It waits at most 40 s, like each `admin` command.

| Argument or flag | Type | Default | Description |
| --- | --- | --- | --- |
| `<id>` | string | none, required | The ID of the staged key, such as `file:2`. |
| `--json` | bool | `false` | Prints one JSON object, `{"key_id":"file:2"}`, instead of the ID. |

Example:

```text
$ sudo hopboxd admin kek promote file:2
file:2
```

The command refuses while a live control process has no copy of the file. Copy the file and run the command again. If the 40 s end first, the command fails with `unavailable`, but the promote stays recorded and the control processes go on ([Sealing](/reference/sealing/index.html.md#several-control-servers)). See [Errors of admin](#errors-of-admin) for the errors.

### admin kek list

```sh
hopboxd admin kek list [--json]
```

Lists the key-encryption keys of the keyring, and the keys that sealed values name. It calls `ListKEKs` ([the admin API](/reference/admin-api/index.html.md#listkeks)). Each line has the key ID, a tab, the number of sealed values that the key seals, a tab, and notes: `current` for the current key, `staged` for a staged key, and `missing from the keyring` for a key whose file is gone. The keys of the `.key` files come first, then the staged keys.

| Flag | Type | Default | Description |
| --- | --- | --- | --- |
| `--json` | bool | `false` | Prints one JSON object, `{"keys":[{"id":"file:1","current":true,"staged":false,"in_keyring":true,"sealed_values":1}]}`. |

Example:

```text
$ sudo hopboxd admin kek list
file:1	1	
file:2	0	current
```

A key with 0 sealed values can go. First back up the keyring dir ([Sealing](/reference/sealing/index.html.md#key-files)).

### admin kek rewrap

```sh
hopboxd admin kek rewrap [--json]
```

Moves every sealed value to the current key-encryption key. It calls `RewrapSealedValues` ([the admin API](/reference/admin-api/index.html.md#rewrapsealedvalues)). It prints `rewrapped <n> to <key ID>`, then the keys as `admin kek list` prints them.

| Flag | Type | Default | Description |
| --- | --- | --- | --- |
| `--json` | bool | `false` | Prints one JSON object with the keys `current_key_id`, `rewrapped`, and `keys`. |

Example:

```text
$ sudo hopboxd admin kek rewrap
rewrapped 1 to file:2
file:1	0	
file:2	1	current
```

### admin config get

```sh
hopboxd admin config get [--json]
```

Prints the settings of the install. It calls `GetInstallConfig` ([the admin API](/reference/admin-api/index.html.md#getinstallconfig)).

| Flag | Type | Default | Description |
| --- | --- | --- | --- |
| `--json` | bool | `false` | Prints one JSON object with the keys `domain`, `public_addresses`, `acme_directory_url`, `acme_email`, `acme_terms_agreed`, `mail_transport`, `maintenance`, `version`, and `update_time`. `public_addresses` is an array. It is `[]` for an install without public addresses. `acme_terms_agreed` and `maintenance` are JSON booleans. |

The text output has one line for each setting: the name, a tab, and the value. An empty domain prints as an empty value. `public_addresses` joins the addresses by commas, in their order. An install without public addresses prints `-`. An empty `acme_email` prints `-`. `acme_terms_agreed` and `maintenance` print `true` or `false`. `mail_transport` prints `smtp` or `file` ([Mail](/reference/mail/index.html.md#mail-transports)). `maintenance` is `true` in maintenance mode ([the API server](/reference/api/index.html.md#maintenance-mode)).

```text
$ sudo hopboxd admin config get
domain	example.com
public_addresses	203.0.113.10,2001:db8::10
acme_directory_url	https://acme-v02.api.letsencrypt.org/directory
acme_email	ops@example.org
acme_terms_agreed	true
mail_transport	smtp
maintenance	false
version	4
update_time	2026-10-16T09:00:00Z
```

### admin config set

```sh
hopboxd admin config set [--domain=DOMAIN] [--public-address=IP,... | --no-public-addresses]
  [--acme-directory-url=URL] [--acme-email=EMAIL] [--[no-]acme-agree-terms]
  [--mail-transport=smtp|file] [--expected-version=N] [--json]
```

Changes the settings that the flags give, and keeps the others. It calls `UpdateInstallConfig` ([the admin API](/reference/admin-api/index.html.md#updateinstallconfig)). Then it prints the settings, as `admin config get` does. Without a setting flag, it fails. A change of the domain or of the public addresses changes the zone of the DNS role ([the DNS role](/reference/dns/index.html.md#the-zone-of-the-install)). The `--acme-` flags change the ACME settings ([the certificate of the install](/reference/acme/index.html.md#settings)).

| Flag | Type | Default | Description |
| --- | --- | --- | --- |
| `--domain` | string | none | The new domain of the install. It must follow the rules in [Domain rules](/reference/domain-rules/index.html.md#install-domain). |
| `--public-address` | IP address, repeatable | none | A public address of the install, IPv4 or IPv6, at which the edge servers take traffic. Repeat the flag, or join the addresses by commas. The list replaces every public address of the install. Each address must follow the rules in [Domain rules](/reference/domain-rules/index.html.md#public-addresses). |
| `--no-public-addresses` | bool | `false` | Removes every public address of the install. It does not go with `--public-address`. |
| `--acme-directory-url` | URL | none | The ACME directory of the CA that issues the certificate of the install ([the certificate of the install](/reference/acme/index.html.md#settings)). A new directory clears the agreement to the terms, unless `--acme-agree-terms` comes with it. |
| `--acme-email` | string | none | The contact email of the ACME account. An empty value, `--acme-email ""`, removes it from the install, but the CA keeps the old contact of the account. |
| `--acme-agree-terms` | bool | none | Agrees to the terms of the CA of the ACME directory, so the install gets its certificate. `--no-acme-agree-terms` withdraws the agreement and stops new certificates. Without either flag, the agreement stays as it is. |
| `--mail-transport` | `smtp` or `file` | none | How the install delivers its mail ([Mail](/reference/mail/index.html.md#mail-transports)). `smtp` sends it through the SMTP server. `file` writes each mail to a file on the control server, for a dev or test install only. The SMTP config stays in both cases. |
| `--expected-version` | int | none | The version `N` of the settings, as `admin config get` printed it. If the version is no longer `N`, the command fails with `aborted` and changes nothing. |
| `--json` | bool | `false` | Prints one JSON object, as `admin config get --json` does. |

Example:

```text
$ sudo hopboxd admin config set --domain example.com --json
{"domain":"example.com","public_addresses":[],"acme_directory_url":"https://acme-v02.api.letsencrypt.org/directory","acme_email":"","acme_terms_agreed":false,"mail_transport":"smtp","version":2,"update_time":"2026-10-16T09:00:00Z"}
$ sudo hopboxd admin config set --public-address 203.0.113.10 --public-address 2001:db8::10
domain	example.com
public_addresses	203.0.113.10,2001:db8::10
acme_directory_url	https://acme-v02.api.letsencrypt.org/directory
acme_email	-
acme_terms_agreed	false
mail_transport	smtp
version	3
update_time	2026-10-16T09:05:00Z
$ sudo hopboxd admin config set --acme-email ops@example.org --acme-agree-terms
domain	example.com
public_addresses	203.0.113.10,2001:db8::10
acme_directory_url	https://acme-v02.api.letsencrypt.org/directory
acme_email	ops@example.org
acme_terms_agreed	true
mail_transport	smtp
version	4
update_time	2026-10-16T09:06:00Z
```

| Error | Cause |
| --- | --- |
| `give at least one setting to change, such as --domain or --public-address` | The command has no setting flag. |
| `give --public-address or --no-public-addresses, not both` | The command has both flags. |
| `invalid_argument: invalid argument: invalid public address: ...` | An address of `--public-address` breaks a rule of [Domain rules](/reference/domain-rules/index.html.md#public-addresses), such as an address twice. The message names the address and the rule. |
| `invalid_argument: invalid argument: invalid ACME setting: ...` | `--acme-directory-url` is not an `https` URL of a host, or `--acme-email` is not one plain email address. |
| `--mail-transport "<value>" is not smtp or file` | `--mail-transport` has another value. |
| `aborted: ...` | `--expected-version` is not the version of the settings now. |

### admin config smtp get

```sh
hopboxd admin config smtp get [--json]
```

Prints the SMTP config, without its password ([SMTP config](/reference/smtp/index.html.md)). It calls `GetSMTPConfig` ([the admin API](/reference/admin-api/index.html.md#getsmtpconfig)). The text output has one line for each field: the name, a tab, and the value. The fields are `host`, `port`, `tls`, `username`, `password_set`, `from`, `mail_transport`, `version`, and `update_time`. An install without an SMTP config prints an empty host. `mail_transport` is the transport of the install, `smtp` or `file`. With `file`, the install does not use the SMTP server ([Mail](/reference/mail/index.html.md#mail-transports)).

| Flag | Type | Default | Description |
| --- | --- | --- | --- |
| `--json` | bool | `false` | Prints one JSON object with the same fields. |

### admin config smtp set

```sh
hopboxd admin config smtp set --host=HOST --from=ADDRESS [--port=PORT] [--tls=MODE]
  [--username=USER] [--password-stdin | --clear-password] [--expected-version=N] [--json]
```

Replaces the SMTP config, and prints it as `admin config smtp get` does. It calls `SetSMTPConfig` ([the admin API](/reference/admin-api/index.html.md#setsmtpconfig)). Without `--password-stdin` or `--clear-password`, the password that is set now stays.

| Flag | Type | Default | Description |
| --- | --- | --- | --- |
| `--host` | string | none, required | The host name or the IP address of the SMTP server. |
| `--port` | int | the usual port of `--tls`: 587, 465, or 25 | The TCP port. |
| `--tls` | `starttls`, `tls`, or `none` | `starttls` | How the connection is encrypted ([SMTP config](/reference/smtp/index.html.md#tls-modes)). |
| `--username` | string | empty | The SMTP login. It needs a password. |
| `--password-stdin` | bool | `false` | Reads the password from standard input, at most 4096 bytes. A trailing newline is dropped. No flag takes the password, because the process list shows the command line. |
| `--clear-password` | bool | `false` | Removes the password. |
| `--from` | string | none, required | The sender, such as `"hopbox <no-reply@example.org>"`. |
| `--expected-version` | int | none | The version `N` of the install config, as `admin config get` printed it. With another version, the command fails with `aborted` and changes nothing. |
| `--json` | bool | `false` | Prints one JSON object. |

A password needs a key-encryption key. [install](#install) makes the first one. On a server that `hopboxd install` did not set up, such as a dev run, run `admin kek rotate` first.

Example:

```text
$ printf '%s\n' "$SMTP_PASSWORD" | sudo hopboxd admin config smtp set \
    --host smtp.example.org --username hopbox@example.org --password-stdin \
    --from 'hopbox <no-reply@example.org>'
host	smtp.example.org
port	587
tls	starttls
username	hopbox@example.org
password_set	true
from	hopbox <no-reply@example.org>
mail_transport	smtp
version	3
update_time	2026-10-16T09:09:37Z
```

### admin config smtp clear

```sh
hopboxd admin config smtp clear [--json]
```

Removes the SMTP config and its password, so the install sends no more sign-in links. It calls `ClearSMTPConfig` ([the admin API](/reference/admin-api/index.html.md#clearsmtpconfig)), and prints the empty config.

| Flag | Type | Default | Description |
| --- | --- | --- | --- |
| `--json` | bool | `false` | Prints one JSON object. |

### admin config oauth get

```sh
hopboxd admin config oauth [get] [--json]
```

Prints the client of each sign-in provider, without its secret, and the redirect URI to register at the provider ([Sign-in with GitHub and Google](/reference/oauth/index.html.md#redirect-uris)). It calls `GetOAuthConfig` ([the admin API](/reference/admin-api/index.html.md#getoauthconfig)). `get` is the default, so `hopboxd admin config oauth` alone prints the same. The text output has one line for each provider, GitHub first. Each line has four fields with a tab between them: the provider, `on` or `off`, the client ID or `-`, and the redirect URI. Without a domain, the URI field is `- (set the domain first)`.

| Flag | Type | Default | Description |
| --- | --- | --- | --- |
| `--json` | bool | `false` | Prints one JSON object: `clients`, each with `provider`, `enabled`, `client_id`, `client_secret_set`, and `redirect_uri`, then `version` and `update_time`. |

Example:

```text
$ sudo hopboxd admin config oauth
github	on	Ov23liExampleClient	https://example.com/auth/github/callback
google	off	-	https://example.com/auth/google/callback
```

### admin config oauth set

```sh
hopboxd admin config oauth set PROVIDER --client-id=ID [--client-secret-stdin] [--expected-version=N] [--json]
```

Sets the client of one provider, and prints it as `admin config oauth get` does. It calls `SetOAuthClient` ([the admin API](/reference/admin-api/index.html.md#setoauthclient)). `PROVIDER` is `github` or `google`.

| Flag | Type | Default | Description |
| --- | --- | --- | --- |
| `--client-id` | string | none, required | The client ID that the provider gave. |
| `--client-secret-stdin` | bool | `false` | Reads the client secret from standard input, at most 4096 bytes. A trailing newline is dropped. No flag takes the secret, because the process list shows the command line. Without this flag, the secret that is set now stays. That works only for the same client ID. |
| `--expected-version` | int | none | The version `N` of the install config, as `admin config get` printed it. With another version, the command fails with `aborted` and changes nothing. |
| `--json` | bool | `false` | Prints one JSON object. |

A secret needs a key-encryption key, as an SMTP password does. On a server that `hopboxd install` did not set up, run `admin kek rotate` first.

Example:

```text
$ printf '%s\n' "$GITHUB_CLIENT_SECRET" | sudo hopboxd admin config oauth set github \
    --client-id Ov23liExampleClient --client-secret-stdin
github	on	Ov23liExampleClient	https://example.com/auth/github/callback
```

| Message | Cause |
| --- | --- |
| `standard input holds no client secret` | `--client-secret-stdin` read nothing. |
| `invalid_argument: invalid argument: a new client ID needs its client secret` | The provider has no client with this client ID, and the command has no `--client-secret-stdin`. |
| `invalid_argument: invalid argument: ...` | The client ID or the secret breaks a rule of [Sign-in with GitHub and Google](/reference/oauth/index.html.md#config). |
| `failed_precondition: ...` | The keyring has no key-encryption key. |
| `aborted: ...` | `--expected-version` is not the version of the install config now. |

### admin config oauth clear

```sh
hopboxd admin config oauth clear PROVIDER [--expected-version=N] [--json]
```

Removes the client of one provider and its secret, so nobody can start a sign-in with it. It calls `ClearOAuthClient` ([the admin API](/reference/admin-api/index.html.md#clearoauthclient)), and prints the empty client with its redirect URI. The sign-in methods of the provider stay on their accounts.

| Flag | Type | Default | Description |
| --- | --- | --- | --- |
| `--expected-version` | int | none | As for `admin config oauth set`. |
| `--json` | bool | `false` | Prints one JSON object. |

### admin certificate get

```sh
hopboxd admin certificate get [--json]
```

Prints the wildcard certificate of the install, without its key, and the state of the ACME job ([the certificate of the install](/reference/acme/index.html.md)). It calls `GetCertificate` ([the admin API](/reference/admin-api/index.html.md#getcertificate)).

| Flag | Type | Default | Description |
| --- | --- | --- | --- |
| `--json` | bool | `false` | Prints one JSON object with the keys `certificate` and `status`. `certificate` is `null` before the first certificate. Else it holds `domain`, `names` (an array), `serial`, `issuer`, `not_before`, `not_after`, `renew_time`, `directory_url`, and `update_time`. `status` holds `failures`, `last_attempt_time`, `last_error`, `next_attempt_time`, `ca_wait`, `terms_url`, and `renewals_stopped`. A time that is absent is `null`. |

The text output has one line for each field: the name, a tab, and the value. `names` joins the names by commas. An empty value prints `-`. Before the first certificate, the fields of the certificate are one line `certificate	-`.

```text
$ sudo hopboxd admin certificate get
domain	example.com
names	example.com,*.example.com
serial	4553a65221ea631b
issuer	CN=Pebble Intermediate CA 512ad8
not_before	2026-10-20T22:58:39Z
not_after	2027-01-18T22:58:38Z
renew_time	2026-12-19T22:58:38Z
directory_url	https://localhost:14000/dir
update_time	2026-10-20T22:58:44Z
failures	0
last_attempt_time	2026-10-20T22:58:38Z
last_error	-
next_attempt_time	-
ca_wait	false
terms_url	data:text/plain,Do%20what%20thou%20wilt
renewals_stopped	-
```

After a failed attempt, `last_error` names the cause:

```text
failures	1
last_attempt_time	2026-10-20T23:30:29Z
last_error	read the ACME directory: provisioning client: performing request: Get "https://localhost:1/dir": dial tcp [::1]:1: connect: connection refused
next_attempt_time	2026-10-20T23:31:30Z
ca_wait	false
terms_url	data:text/plain,Do%20what%20thou%20wilt
renewals_stopped	-
```

### admin login-link

```sh
hopboxd admin login-link --email=EMAIL [--json]
```

Prints a one-time sign-in link for the first instance admin. It calls `CreateLoginLink` ([the admin API](/reference/admin-api/index.html.md#createloginlink)). The link works once, for 15 minutes. It signs in to the account of earlier login links for the email, or to a new account, and makes that account an instance admin. The email stays unverified, so the account never links to another sign-in method with that email ([Sessions, cookies, and sign-in links](/reference/sessions/index.html.md#what-a-sign-in-does)).

The command prints the link on standard output, and the email and the end of the link on standard error. The install must have a domain. The audit log records the link with the actor `local_root`, but never its token. Treat the link as a password until its use or its end.

| Flag | Type | Default | Description |
| --- | --- | --- | --- |
| `--email` | string | none, required | The email of the account. |
| `--json` | bool | `false` | Prints one JSON object with the keys `url`, `email`, and `expire_time`. |

Example:

```text
$ sudo hopboxd admin login-link --email root@example.org
https://example.com/auth/login-link#token=tOzJ8YQyibHutrU34tIWLHnGd5XlzYs4WoIBXgSDh5I
The link signs in root@example.org once, until 2026-10-16T09:24:15Z.
```

[First sign-in on a new install](/guide/first-sign-in/index.html.md) uses the link on a new install.

### admin instance-admin list

```sh
hopboxd admin instance-admin list [--json]
```

Lists the instance admins, in the order of their grant. It calls `ListInstanceAdmins` ([the admin API](/reference/admin-api/index.html.md#listinstanceadmins)). Each line has the account ID, a tab, the emails, a tab, and the time of the grant. An unverified email ends with ` (unverified)`.

| Flag | Type | Default | Description |
| --- | --- | --- | --- |
| `--json` | bool | `false` | Prints one JSON object, `{"admins":[...]}`. Each admin has the keys `account_id`, `display_name`, `emails` (each with `email` and `verified`), and `grant_time`. |

```text
$ sudo hopboxd admin instance-admin list
01a143f9-1cee-7ec0-9238-7976abaacfd6	root@example.org (unverified)	2026-10-16T09:09:21Z
```

### admin instance-admin grant

```sh
hopboxd admin instance-admin grant (--account-id=ID | --email=EMAIL)
```

Makes an account an instance admin, and prints its ID. It calls `GrantInstanceAdmin` ([the admin API](/reference/admin-api/index.html.md#grantinstanceadmin)). Root on a control server can always make an account an instance admin, as a way to recover.

| Flag | Type | Default | Description |
| --- | --- | --- | --- |
| `--account-id` | string | none | The UUID of the account. |
| `--email` | string | none | A verified email of the account. An unverified email names no account. |

Give exactly one of the two flags.

### admin instance-admin revoke

```sh
hopboxd admin instance-admin revoke --account-id=ID
```

Takes the instance admin role from an account. It calls `RevokeInstanceAdmin` ([the admin API](/reference/admin-api/index.html.md#revokeinstanceadmin)). An install keeps at least one instance admin, so the command fails for the last one.

| Flag | Type | Default | Description |
| --- | --- | --- | --- |
| `--account-id` | string | none, required | The UUID of the account. |

### admin account sign-in-method list

```sh
hopboxd admin account sign-in-method list (--account-id=ID | --email=EMAIL) [--json]
```

Lists the sign-in methods of an account, oldest first. It calls `ListAccountSignInMethods` ([the admin API](/reference/admin-api/index.html.md#listaccountsigninmethods)). The text output has one line per method, with a tab between the fields: the ID, the kind, the subject, the email, and the time of the last use.

| Flag | Type | Default | Description |
| --- | --- | --- | --- |
| `--account-id` | string | none | The UUID of the account. Give it or `--email`. |
| `--email` | string | none | A verified email of the account. |
| `--json` | bool | `false` | Prints one JSON object: `account_id` and `methods`, each with `id`, `kind`, `subject`, `email`, `last_use_time`, and `create_time`. |

Example:

```text
$ sudo hopboxd admin account sign-in-method list --email erin@example.org
01a15c02-7d1e-7a4c-9b51-2b1c5e8f0a7d	github	4242	erin@example.org	2026-10-21T09:12:04Z
```

### admin account sign-in-method remove

```sh
hopboxd admin account sign-in-method remove --account-id=ID --method-id=ID [--allow-last]
```

Removes one sign-in method of an account. It calls `RemoveAccountSignInMethod` ([the admin API](/reference/admin-api/index.html.md#removeaccountsigninmethod)). Use it for a person who lost the only method of the account, for example the GitHub user ([Sign-in with GitHub and Google](/reference/oauth/index.html.md#recover-a-lost-identity)). The command prints nothing. It does not end the sessions or the CLI sign-ins that the method started ([the admin API](/reference/admin-api/index.html.md#removeaccountsigninmethod)).

| Flag | Type | Default | Description |
| --- | --- | --- | --- |
| `--account-id` | string | none, required | The UUID of the account, as `list` prints it. |
| `--method-id` | string | none, required | The UUID of the method. |
| `--allow-last` | bool | `false` | Removes also the last method. The account then has no identity of a provider, so it signs in through a new method that proves one of its verified emails, such as a magic link. An account without a verified email keeps its last method, and the command fails with `failed_precondition`. |

### admin enrollment-token create

```sh
hopboxd admin enrollment-token create [--pool=NAME] [--roles=ROLE,...] [--server=ID] [--peer-address=ADDRESS]
  [--ttl=DURATION] [--json]
```

Makes an enrollment token with which one new server joins a pool, with some of the roles of the token. The token works once and expires after 1 hour, or after `--ttl`. A retry of its enrollment with the same key gets the same certificate again until then ([Enrollment and the install PKI](/reference/enrollment/index.html.md#retry-after-a-lost-answer)). It calls `CreateEnrollmentToken` ([the admin API](/reference/admin-api/index.html.md#createenrollmenttoken)) with a new idempotency key. It prints the token alone on one line, so a script can keep it in a variable. The token is a secret. The install keeps only a hash of its secret, so this output is its only copy on the control servers. The new server keeps a copy in `<data_dir>/server/pending.pem` until its enrollment ends ([Enrollment and the install PKI](/reference/enrollment/index.html.md#retry-after-a-lost-answer)). The first token of an install makes the install CA, so the install needs a key-encryption key first. [install](#install) makes it, and [admin kek rotate](#admin-kek-rotate) makes it on a server that `hopboxd install` did not set up.

| Flag | Type | Default | Description |
| --- | --- | --- | --- |
| `--pool` | string | `default` | The pool that the new server joins. Every install has the pool `default`. |
| `--roles` | list of strings, joined by commas | `node` | The roles that the new server can enroll with: `node`, `dns`, `edge`, or more than one. |
| `--server` | string | none | The ID of an enrolled server. The token then adds the roles to that server alone, in its pool, and enrolls no new server ([Enrollment and the install PKI](/reference/enrollment/index.html.md#add-roles-to-an-enrolled-server)). It ignores `--pool`. |
| `--peer-address` | string | none | The peer address of the server that enrolls, such as `203.0.113.12`: one public unicast IPv4 address on its uplink. The enrollment must come from it, and the new server keeps it ([Enrollment and the install PKI](/reference/enrollment/index.html.md#peer-addresses)). |
| `--ttl` | duration | `1h` | How long the token works, such as `2h`. More than 0 and at most `24h`. |
| `--json` | bool | `false` | Prints one JSON object with the keys `token`, `token_id`, `pool`, `roles`, `expire_time` (UTC), and `ca_hash`. A token with `--server` also has `server_id`, and a token with `--peer-address` has `peer_address`. |

Example:

```text
$ TOKEN="$(sudo hopboxd admin enrollment-token create)"
$ sudo hopboxd admin enrollment-token create --json
{"token":"hbxe_<id>_<secret>_<cahash>","token_id":"01a14404-...","pool":"default","roles":["node"],"expire_time":"2026-10-16T10:00:00Z","ca_hash":"<cahash>"}
```

### admin server list

```sh
hopboxd admin server list [--json]
```

Lists the enrolled servers, oldest enrollment first. It calls `ListServers` ([the admin API](/reference/admin-api/index.html.md#listservers)). The text output is a table. Its columns are `ID`, `POOL`, `STATE`, `STATUS`, `ROLES`, `HOSTNAME`, `VERSION`, `LAST HEARTBEAT`, and `PEER ADDRESS`. `VERSION` is the version of `hopboxd` on the server. `LAST HEARTBEAT` is an RFC 3339 time. `PEER ADDRESS` is the peer address of the server ([Enrollment and the install PKI](/reference/enrollment/index.html.md#peer-addresses)). An empty value prints as `-`. With no server, the table has only its header.

| Flag | Type | Default | Description |
| --- | --- | --- | --- |
| `--json` | bool | `false` | Prints one JSON object, `{"servers":[...]}`, with one object for each server, as `admin server get --json` prints it. |

Example:

```text
$ sudo hopboxd admin server list
ID                                    POOL     STATE  STATUS  ROLES  HOSTNAME  VERSION  LAST HEARTBEAT        PEER ADDRESS
01a14404-ce7b-72d9-b18e-5006c3e462db  default  ready  ok      node   server1   v0.1.0   2026-10-16T09:00:05Z  -
```

### admin server get

```sh
hopboxd admin server get <id> [--json]
```

Prints the server with the ID `<id>`. It calls `GetServer` ([the admin API](/reference/admin-api/index.html.md#getserver)). The text output has one line for each field: the name, a tab, and the value. The lines are these:

- `id`, `pool`, `hostname`, and `roles` (joined by commas)
- `state`, `cordoned` (`true` or `false`), `status`, and `problems` (joined by `; `)
- `labels` (`key=value` pairs sorted by key, joined by commas), and `labels_set_by_admin` (`true` or `false`)
- `hopboxd_version`, `kernel_release`, `cpu` (the arch, the vendor, and the model name), `cpu_template`, and `compatibility_class`
- `cpu_threads`, `memory_bytes`, `disk_bytes`, `disk_free_bytes`, `machine_count`, and `allocations` and `actions` (their counts)
- `last_heartbeat_time` (empty before the first heartbeat), `enroll_time`, `certificate_expire_time`, `version`, and `update_time`
- `peer_address`: the public IPv4 address at which the other servers reach the server, empty for a server without one ([Enrollment and the install PKI](/reference/enrollment/index.html.md#peer-addresses))
- `mesh_address` and `mesh_public_key`: the part of the server in the mesh, empty for a server outside the mesh ([the mesh](/reference/mesh/index.html.md))
- `dns_zone_serial`: the serial of the zones that the DNS role of the server served at its last heartbeat, empty on a server without the DNS role or without zones ([the DNS role](/reference/dns/index.html.md#the-serial-of-each-dns-server))
- Only while the server drains: `drain_start_time`, `drain_moving`, `drain_empty_checks`, `drain_last_check_time` (empty before the first check), `drain_error`, `drain_stuck` (`true` or `false`), and `drain_cordoned_before` (`true` or `false`). Then one line `drain_stranded` for each stranded machine, with the value `<name> <machine id>: <reason>`.
- `removed_roles`: the roles that a removal took from the server, joined by commas ([the admin API](/reference/admin-api/index.html.md#removeserverroles))
- Only while a removal of roles is pending: `removal_roles` (joined by commas), `removal_start_time`, `removal_cordoned_before`, `removal_moving`, `removal_error`, `removal_stuck`, and `removal_ready` (`true` or `false`). Then one line `removal_stranded` for each stranded machine, with the value `<name> <machine id>: <reason>`.

Times are RFC 3339 in UTC. `state` is `ready`, `cordoned`, `unresponsive`, `draining`, `shutting_down`, or `decommissioned`. `status` is `ok`, `degraded`, or empty before the first heartbeat.

| Flag | Type | Default | Description |
| --- | --- | --- | --- |
| `--json` | bool | `false` | Prints one JSON object. |

The JSON object has these keys:

- `id`, `pool`, `hostname`, `roles`, `state`, `cordoned`, `status`, and `problems`
- `labels`, `labels_set_by_admin`, `machine_count`, `hopboxd_version`, and `kernel_release`
- `cpu`: an object with `arch`, `vendor`, `family`, `model`, `stepping`, `model_name`, `microcode`, `flags_hash`, and `cpu_template`
- `compatibility_class`
- `capacity`: an object with `cpu_threads`, `memory_bytes`, `disk_bytes`, and `disk_free_bytes`
- `allocations`: a list of objects with `machine_id`, `vcpus`, `memory_bytes`, and `disk_bytes`
- `actions`: a list of objects with `machine_id`, `action`, and `start_time`
- `last_heartbeat_time` (`null` before the first heartbeat), `enroll_time`, `certificate_expire_time`, `version`, and `update_time`
- `mesh`: an object with `address` and `public_key`, or `null` for a server outside the mesh
- `dns_zone_serial`: a number, or `null` on a server without the DNS role or without zones
- `peer_address`: a string, empty for a server without one
- `drain`: `null` for a server that does not drain, or an object with `start_time`, `moving`, `stranded` (a list of objects with `machine_id`, `name`, `reason`, and `on_server`), `empty_checks`, `last_check_time` (`null` before the first check), `error`, `stuck`, and `cordoned_before`
- `removal`: `null` without a pending removal of roles, or an object with `roles`, `start_time`, `cordoned_before`, `moving`, `stranded` (as in `drain`), `error`, `stuck`, and `ready`
- `removed_roles`: the roles that a removal took from the server

Lists and the `labels` object are empty, never `null`. [The admin API](/reference/admin-api/index.html.md#server) describes each field.

### admin machine list

```sh
hopboxd admin machine list [--org-id=ID] [--json]
```

Lists the machines of every org, or of one org, in the order of their IDs, with metadata only. It calls `ListMachines` of the admin API, page by page, until the last page ([the admin API](/reference/admin-api/index.html.md#listmachines)). A machine whose current state is `deleted` is not in the list. Each line has six parts, with a tab between them. They are the machine ID, the name, and `org=` with the org ID. Then come the current state, `target=` with the target state, and `server=` with the server ID of its node. A state is printed as the domain names it, for example `running` ([Domain rules](/reference/domain-rules/index.html.md#machine-states)).

| Flag | Type | Default | Description |
| --- | --- | --- | --- |
| `--org-id` | string | none | Lists only the machines of this org. |
| `--json` | bool | `false` | Prints one JSON object, `{"machines":[...]}`. Each machine has the keys `id`, `org_id`, `name`, `state`, `target_state`, `size` (the size name), and `image_digest`. It also has `creator_account_id`, `creator_key_id`, `shared`, `server_id`, `ssh_port`, `url`, `delete_request_time`, `version`, and `create_time`. An unset ID is the empty string. An unset `delete_request_time` is `null`. The times are in RFC 3339 form, in UTC. |

```text
$ sudo hopboxd admin machine list
01a1460c-1f6e-7b8a-9c42-3d5e7f8a9b10	box1	org=01a14565-8448-7753-9c1f-5ec0a86b8d92	pending	target=running	server=01a14403-6bd1-7522-9ab0-b3ab4517e0e4
```

### admin machine stop

```sh
hopboxd admin machine stop (--name=NAME | --id=ID) [--json]
```

Sets the target of any machine to `stopped`, also in a suspended org, and waits for it. It calls `StopMachine` of the admin API ([the admin API](/reference/admin-api/index.html.md#stopmachine)). It prints one line per fact: a name, a tab, and a value. The facts are `id`, `name`, `state`, `target_state`, `duration`, and `done`.

The API waits for up to 60 s. But each `admin` command waits at most 40 s for its call. So the wait ends after about 39 s, 1 s before the deadline of the command. A wait that ends before the machine is stopped prints `done` `false`, and the command still succeeds. The machine goes on toward its target.

| Flag | Type | Default | Description |
| --- | --- | --- | --- |
| `--name` | string | none | The name of the machine. Give `--name` or `--id`, not both. |
| `--id` | string | none | The UUID of the machine. |
| `--json` | bool | `false` | Prints one JSON object with the keys `machine` (with the keys of `admin machine list --json`), `duration` (for example `39.001s`), and `done`. |

```text
$ sudo hopboxd admin machine stop --name box1
id	01a1460c-1f6e-7b8a-9c42-3d5e7f8a9b10
name	box1
state	stopped
target_state	stopped
duration	512ms
done	true
```

The executor of the node stops the microVM of the machine ([the node executor](/reference/executor/index.html.md)). The guest of the default image shuts down in less than a second ([the Firecracker driver](/reference/firecracker-driver/index.html.md#stop)).

### admin machine delete

```sh
hopboxd admin machine delete (--name=NAME | --id=ID) [--json]
```

Sets the target of any machine to `deleted`, also in a suspended org, and waits for it, as `admin machine stop` does. It calls `DeleteMachine` of the admin API ([the admin API](/reference/admin-api/index.html.md#deletemachine)). The name and the SSH port of the machine stay reserved for its org for 7 days. The output is that of `admin machine stop`.

| Flag | Type | Default | Description |
| --- | --- | --- | --- |
| `--name` | string | none | The name of the machine. Give `--name` or `--id`, not both. |
| `--id` | string | none | The UUID of the machine. |
| `--json` | bool | `false` | Prints one JSON object with the keys `machine`, `duration`, and `done`, as `admin machine stop --json` does. |

```sh
sudo hopboxd admin machine delete --id 01a1460c-1f6e-7b8a-9c42-3d5e7f8a9b10
```

### admin machine move

```sh
hopboxd admin machine move (--name=NAME | --id=ID) [--to=SERVER_ID | --cancel [--give-back] | --withdraw-give-back] [--json]
```

Moves any machine cold to another node, also in a suspended org ([Cold move](/reference/cold-move/index.html.md)). The node of the machine stops it. The other node then pulls its disk, and boots it if its target is `running`. It calls `MoveMachine` of the admin API ([the admin API](/reference/admin-api/index.html.md#movemachine)), with a new idempotency key. The command returns at once, with the move in the state `stopping`. Follow the move with `admin machine moves`.

| Flag | Type | Default | Description |
| --- | --- | --- | --- |
| `--name` | string | none | The name of the machine. Give `--name` or `--id`, not both. |
| `--id` | string | none | The UUID of the machine. |
| `--to` | string | none | The UUID of the node to move the machine to. Without it, placement picks a node. |
| `--cancel` | bool | `false` | Ends the move of the machine that runs, with `CancelMove` of the admin API ([the admin API](/reference/admin-api/index.html.md#cancelmove)). The machine stays on its source node with its disk, or goes back there, stopped. Give `--to` or `--cancel`, not both. |
| `--give-back` | bool | `false` | With `--cancel`, also asks the target node to give back a machine that a failed move left there while its source keeps a copy of its disk, also after the move failed ([Cold move](/reference/cold-move/index.html.md#give-back)). The target node gives it back only when it never proved the pull of the move. Otherwise it keeps the machine, and the source removes its copy. Without `--cancel`, the command fails with `--give-back needs --cancel`. |
| `--withdraw-give-back` | bool | `false` | Takes back a give-back that waits for the answer of the target node, such as one that the node withholds ([Cold move](/reference/cold-move/index.html.md#give-back)). The move gets back its state from before the request. Give only one of `--to`, `--cancel`, and `--withdraw-give-back`. |
| `--json` | bool | `false` | Prints one JSON object with the key `move`, which has the keys of `admin machine moves --json`. |

```console
$ sudo hopboxd admin machine move --name box1
id	01a1460c-2a7b-7c11-8d23-4e5f6a7b8c9d
machine_id	01a1460c-1f6e-7b8a-9c42-3d5e7f8a9b10
source_server_id	01a14600-0a1b-7c2d-8e3f-405162738495
target_server_id	01a14600-1b2c-7d3e-8f40-516273849506
state	stopping
```

If a move is too slow or no longer needed, end it with `--cancel`. The command prints the move in the state `failed`:

```console
$ sudo hopboxd admin machine move --name box1 --cancel
id	01a1460c-2a7b-7c11-8d23-4e5f6a7b8c9d
machine_id	01a1460c-1f6e-7b8a-9c42-3d5e7f8a9b10
source_server_id	01a14600-0a1b-7c2d-8e3f-405162738495
target_server_id	01a14600-1b2c-7d3e-8f40-516273849506
state	failed
```

A failed move can leave the machine on its target while the source keeps a copy of its disk, so the machine does not move again. Ask for the give-back. The command returns at once, with the line `give_back` `requested`. Follow the answer of the target node with `admin machine moves`: `done` gives the machine back to its source node, stopped, and `kept` leaves it on the target, whose disk is whole:

```console
$ sudo hopboxd admin machine move --name box1 --cancel --give-back
id	01a1460c-2a7b-7c11-8d23-4e5f6a7b8c9d
machine_id	01a1460c-1f6e-7b8a-9c42-3d5e7f8a9b10
source_server_id	01a14600-0a1b-7c2d-8e3f-405162738495
target_server_id	01a14600-1b2c-7d3e-8f40-516273849506
state	failed
give_back	requested
```

### admin machine moves

```sh
hopboxd admin machine moves [--id=ID] [--json]
```

Lists the cold moves of one machine, or of every machine, newest first. It calls `ListMoves` of the admin API ([the admin API](/reference/admin-api/index.html.md#listmoves)) for each page. Each line holds one move, with tabs between its parts. The parts are the ID of the move, the machine, the state, the source node, and the target node. Then come the node that still holds an old copy, and the error of a failed move. A move with a give-back ends with `give_back=` and its state: `requested`, `done`, or `kept`. A give-back that the target node withholds adds `give_back_note=` and the note of its reason.

| Flag | Type | Default | Description |
| --- | --- | --- | --- |
| `--id` | string | none | Lists only the moves of the machine with this UUID. |
| `--json` | bool | `false` | Prints one JSON object with the key `moves`. Each move has the keys `id`, `machine_id`, `source_server_id`, `target_server_id`, and `state`. It also has `target_state`, `error`, `cleanup_server_id`, `create_time`, `update_time`, `give_back` (`requested`, `done`, `kept`, or empty without a give-back), and `give_back_note`, why the target node withholds its answer, or empty. |

The states are `stopping`, `pulling`, `done`, and `failed` ([Cold move](/reference/cold-move/index.html.md#the-steps-of-a-move)).

```console
$ sudo hopboxd admin machine moves --id 01a1460c-1f6e-7b8a-9c42-3d5e7f8a9b10
01a1460c-2a7b-7c11-8d23-4e5f6a7b8c9d	machine=01a1460c-1f6e-7b8a-9c42-3d5e7f8a9b10	done	from=01a14600-0a1b-7c2d-8e3f-405162738495	to=01a14600-1b2c-7d3e-8f40-516273849506	cleanup=
```

### The default image of the release

Each release ships its own default image, because the image holds `hopbox-guest` of the same commit ([Release bundle](/reference/release-bundle/index.html.md#the-default-image-in-the-bundle)). So at each start, the control role registers the image of `control.release_image`, by default `/opt/hopbox/current/image/artifact.json` ([the bootstrap config of hopboxd](/reference/hopboxd-config/index.html.md#keys)). New machines of the name `default` then take the guest agent of the release. Machines that exist keep their image.

The rules are simple:

- The start registers the image by its digest. If the install has the digest already, the start changes nothing and writes no audit entry. So a restart of the same release changes nothing. When two control servers start at once, one of them adds the image, and the other one finds it.
- A new image gets the audit entry `image.register` with the actor `local_root` and the detail `already` `false`, as `admin image register` as root does.
- The start registers only the image of the release, under its own name. It never changes or removes an image of another name.
- New machines take the image of a name that the install registered last. So an image that an operator registers after the start wins, also an older build. It wins until a release brings an image that the install does not have yet. To keep an image of your own across releases, give it another name, and create machines with that image name.
- A rollback does not bring its image back. `hopboxd upgrade --rollback` starts the older release, whose image the install has already. So new machines keep the image of the newer release, with its guest agent. A register of the older image changes nothing either, because the install has its digest. Each guest agent takes the versions of the guest link that its node speaks ([Guest link](/reference/guest-link/index.html.md#versions)).
- In phase 2, an install can have control servers on two releases, for example during an upgrade. At its start, the control role skips the register when the install has an image of the same name that a newer release registered. It compares the releases as semver versions. So a control server on the older release never makes its image the newest. The start logs `the install has the default image of a newer release, so the control role registers none`, and changes nothing. The control role records its release on the images that it registers. An operator image has no release, and so has an image that a release before migration `0045` registered. Neither makes the control role skip. A version that is not semver, such as a build for development, skips nothing. The start does not compare the build times, because a reproducible build of the image has a fixed build time. The check and the register run in one transaction under the lock `LockKeyReleaseImages`, so two control servers of two releases that start at once never both register ([the bundled Postgres](/reference/postgres/index.html.md#advisory-lock-keys)). After a full rollback of every control server to the older release, the image of the newer release stays the newest of its name, because the older release skips its own. Register the older image by hand with `admin image register` if you want it back.

Only a server with the control role registers the image. The nodes still need the disk of the image: the install of a release puts it into the image cache of a node ([the Firecracker driver](/reference/firecracker-driver/index.html.md#the-image-cache)).

### admin image register

```sh
hopboxd admin image register ARTIFACT_JSON [--url=URL] [--json]
```

Adds an image from its `artifact.json`, for example the one of `make image` in `bin/image/` ([Default image](/reference/default-image/index.html.md)). It reads the file and calls `RegisterImage` with its bytes ([the admin API](/reference/admin-api/index.html.md#registerimage)). New machines take the newest image of a name. A second call with the same file changes nothing, and prints `registered` `false`. Without `--json`, it prints the lines `id`, `name`, `digest`, and `registered`, each with a tab before its value.

| Argument or flag | Type | Default | Description |
| --- | --- | --- | --- |
| `ARTIFACT_JSON` | path | none, required | The `artifact.json` of the image. A file that does not exist exits with status 80. |
| `--url` | string | empty | An https URL from which nodes download the compressed disk. |
| `--json` | bool | `false` | Prints one JSON object with the keys `image` and `registered`. `image` has the keys `id`, `name`, `digest`, `arch`, `kernel_sha256`, `disk_size`, `disk_sha256`, `url`, `build_time`, and `register_time`. |

```text
$ sudo hopboxd admin image register bin/image/artifact.json --url https://dl.example.org/images/default/disk.raw.zst
id	01a1460a-7c21-7d4e-8f90-1a2b3c4d5e6f
name	default
digest	sha256:422d31d7a4fd5e6958162b22e3d772f71b5a1465cbd6255d253ae8c087873dc8
registered	true
```

### admin image list

```sh
hopboxd admin image list [--json]
```

Lists the images of the install, by name, newest first. It calls `ListImages` ([the admin API](/reference/admin-api/index.html.md#listimages)). Each line has the image ID, the name, the digest, and the time of the registration in RFC 3339 form, with a tab between them.

| Flag | Type | Default | Description |
| --- | --- | --- | --- |
| `--json` | bool | `false` | Prints one JSON object, `{"images":[...]}`, with the keys of `admin image register --json` for each image. |

```text
$ sudo hopboxd admin image list
01a1460a-7c21-7d4e-8f90-1a2b3c4d5e6f	default	sha256:422d31d7a4fd5e6958162b22e3d772f71b5a1465cbd6255d253ae8c087873dc8	2026-10-16T16:01:40Z
```

Example:

```text
$ sudo hopboxd admin server get 01a14404-ce7b-72d9-b18e-5006c3e462db
id	01a14404-ce7b-72d9-b18e-5006c3e462db
pool	default
hostname	server1
roles	node
state	ready
cordoned	false
status	ok
problems	
labels	zone=gra
labels_set_by_admin	false
hopboxd_version	v0.1.0
kernel_release	6.18.57-hopbox1
cpu	amd64 GenuineIntel Intel(R) Xeon(R) CPU E5-1650 v4 @ 3.60GHz
cpu_template	none
compatibility_class	cc-9c6a6f2eb2ae6047
cpu_threads	12
memory_bytes	135061946368
disk_bytes	3998614224896
disk_free_bytes	3998000000000
machine_count	0
allocations	0
actions	0
last_heartbeat_time	2026-10-16T09:00:05Z
enroll_time	2026-10-16T08:59:00Z
certificate_expire_time	2026-11-15T08:59:00Z
version	14
update_time	2026-10-16T09:00:05Z
mesh_address	10.239.0.2
mesh_public_key	gN65BkIKy1eCE9pP1wdc8ROUtkHLF2PfAqYdO0JbmjI=
dns_zone_serial	
```

### admin server cordon

```sh
hopboxd admin server cordon <id> [--json]
```

Stops new machines from landing on the server `<id>`. Its machines keep running. It calls `CordonServer` ([the admin API](/reference/admin-api/index.html.md#cordonserver)) and prints the server after the change, as `admin server get` does. A cordon of a cordoned server changes nothing. A server that is unresponsive stays unresponsive, and comes back `cordoned` with its next heartbeat.

| Flag | Type | Default | Description |
| --- | --- | --- | --- |
| `--json` | bool | `false` | Prints one JSON object, as `admin server get --json` does. |

### admin server uncordon

```sh
hopboxd admin server uncordon <id> [--json]
```

Lets new machines land on the server `<id>` again. It calls `UncordonServer` ([the admin API](/reference/admin-api/index.html.md#uncordonserver)) and prints the server after the change, as `admin server get` does. An uncordon of a server that is not cordoned changes nothing.

| Flag | Type | Default | Description |
| --- | --- | --- | --- |
| `--json` | bool | `false` | Prints one JSON object, as `admin server get --json` does. |

### admin server set-labels

```sh
hopboxd admin server set-labels <id> [--label=KEY=VALUE ...] [--json]
```

Replaces the labels of the server `<id>`. It calls `SetServerLabels` ([the admin API](/reference/admin-api/index.html.md#setserverlabels)) and prints the server after the change, as `admin server get` does. The labels replace every label of the server. Without `--label`, the server gets no labels. From then on, the install keeps these labels, and `channel.labels` of the bootstrap config of the server has no effect. The hopbox Terraform provider runs this command over SSH on the first server of an install ([hopbox Terraform provider](/reference/terraform-provider/index.html.md)).

| Flag | Type | Default | Description |
| --- | --- | --- | --- |
| `--label` | `KEY=VALUE` | none | A label of the server. Repeat the flag for each label. The rules of `channel.labels` apply ([the bootstrap config of hopboxd](/reference/hopboxd-config/index.html.md#keys)). |
| `--json` | bool | `false` | Prints one JSON object, as `admin server get --json` does. |

Example:

```sh
sudo hopboxd admin server set-labels 01a14404-ce7b-72d9-b18e-5006c3e462db --label zone=gra --label disk=nvme
```

### admin server set-peer-address

```sh
hopboxd admin server set-peer-address <id> <address> [--json]
```

Sets the peer address of the server with the ID `<id>`: one public unicast IPv4 address on its uplink, at which the other servers reach it, such as `203.0.113.12` ([Enrollment and the install PKI](/reference/enrollment/index.html.md#peer-addresses)). It is for a server that enrolled without one, such as a server from before peer addresses. It calls `SetServerPeerAddress` ([the admin API](/reference/admin-api/index.html.md#setserverpeeraddress)) and prints the server as [admin server get](#admin-server-get) does. The call writes the audit entry `server.set_peer_address`. A heartbeat never changes the address.

| Flag | Type | Default | Description |
| --- | --- | --- | --- |
| `--json` | bool | `false` | Prints the server as one JSON object. |

Example:

```text
$ sudo hopboxd admin server set-peer-address 01a14404-ce7b-72d9-b18e-5006c3e462db 203.0.113.12 | grep peer_address
peer_address	203.0.113.12
```

### admin server drain

```sh
hopboxd admin server drain <id> [--json]
```

Moves every machine of the server `<id>` cold to other nodes, then decommissions the server ([Cold move](/reference/cold-move/index.html.md#drain-of-a-node), [Drain a node](/guide/drain-a-node/index.html.md)). It calls `DrainServer` ([the admin API](/reference/admin-api/index.html.md#drainserver)) and prints the server after the change, as `admin server get` does. The command returns at once. A leader job does the moves. Follow the drain with `admin server get`. A second run starts the drain again: it tries each stranded machine again, and counts the empty checks from zero. A server that drains stays cordoned, so `admin server uncordon` refuses it. To keep the server, run `admin server cancel-drain`. A server without the role `node` holds no machine, so the command refuses it: run `admin server decommission` for it. Do not drain a server that Terraform manages. Run `terraform destroy` instead, which drains it first. After a drain outside Terraform, the next refresh drops a decommissioned node from the Terraform state, and fails on the first server.

| Flag | Type | Default | Description |
| --- | --- | --- | --- |
| `--json` | bool | `false` | Prints one JSON object, as `admin server get --json` does. |

Example:

```text
$ sudo hopboxd admin server drain 01a14404-ce7b-72d9-b18e-5006c3e462db
id	01a14404-ce7b-72d9-b18e-5006c3e462db
pool	default
hostname	node1
roles	node
state	draining
cordoned	true
...
drain_start_time	2026-10-16T09:10:00Z
drain_moving	0
drain_empty_checks	0
drain_last_check_time	
drain_error	
drain_stuck	false
drain_cordoned_before	false
removed_roles	
```

### admin server remove-role

```sh
hopboxd admin server remove-role <id> <role>... [--json]
```

Starts the removal of the roles `<role>` from the running server `<id>`: `edge`, `dns`, `node`, or more than one. It calls `RemoveServerRoles` ([the admin API](/reference/admin-api/index.html.md#removeserverroles)) and prints the server with its removal, as `admin server get` does. The command returns at once. The role `control` cannot go in this release. The role `dns` runs only on edge servers, so the removal of `edge` from a server with `dns` names `dns` too.

The call checks that another ready server keeps each role, and that an edge that stays holds a floating address. It also checks that the public addresses of the install name no address of the server. Set the public addresses first with `admin config set --public-address`. For `node`, the call cordons the server, and a leader job moves its machines cold to other nodes, as a drain does ([Cold move](/reference/cold-move/index.html.md#removal-of-the-node-role)). Follow the removal with `admin server get`. Once `removal_ready` is `true`, an enrollment of the server with the roles that stay ends the removal. Run `hopboxd install --roles <every role that stays>` on the server, with a token for the server and those roles ([Enrollment and the install PKI](/reference/enrollment/index.html.md#remove-roles-from-an-enrolled-server)).

For a server that Terraform manages, drop the roles from its `roles` in the configuration and apply instead. The provider runs this command on the first server, and ends the removal ([hopbox Terraform provider](/reference/terraform-provider/index.html.md#roles-of-a-running-server)). A second run with the same roles prints the pending removal. A run with other roles fails while a removal is pending.

| Argument or flag | Type | Default | Description |
| --- | --- | --- | --- |
| `<id>` | string | none, required | The ID of the server. |
| `<role>` | string | none, required | A role that goes: `edge`, `dns`, or `node`. Repeat it for each role. |
| `--json` | bool | `false` | Prints one JSON object, as `admin server get --json` does. |

Example:

```text
$ sudo hopboxd admin server remove-role 01a14404-ce7b-72d9-b18e-5006c3e462db edge dns | grep removal
removal_roles	dns,edge
removal_start_time	2026-10-28T09:10:00Z
removal_cordoned_before	false
removal_moving	0
removal_error	
removal_stuck	false
removal_ready	true
```

### admin server cancel-remove-role

```sh
hopboxd admin server cancel-remove-role <id> [--json]
```

Ends the pending removal of roles from the server `<id>` that cannot finish, such as a removal of `node` with a machine that has no room elsewhere. It calls `CancelRoleRemoval` ([the admin API](/reference/admin-api/index.html.md#cancelroleremoval)) and prints the server after the cancel, as `admin server get` does. No new move starts, and each move of the removal that still runs ends. A machine that moved stays on its new node. The server keeps every role, and the cordon of the removal goes. A cordon that the server had before the removal stays. For a server that Terraform manages, put the role back in the configuration after the cancel, so the next plan shows no change.

| Argument or flag | Type | Default | Description |
| --- | --- | --- | --- |
| `<id>` | string | none, required | The ID of the server. |
| `--json` | bool | `false` | Prints one JSON object, as `admin server get --json` does. |

### admin server cancel-drain

```sh
hopboxd admin server cancel-drain <id> [--json]
```

Ends the drain of the server `<id>`, such as a drain that started by mistake. It calls `CancelDrain` ([the admin API](/reference/admin-api/index.html.md#canceldrain)) and prints the server after the cancel, as `admin server get` does. No new move starts. A move whose source still stops its machine ends, and the machine keeps its own target on the server. A move whose target pulls the disk goes on to its end, so that machine runs on its new node. The stranded machines stay on the server. The server turns `cordoned` when `drain_cordoned_before` was `true`, and `ready` otherwise. For a server that Terraform manages, a cancel during `terraform destroy` fails the destroy, and the server stays in the Terraform state.

| Argument or flag | Type | Default | Description |
| --- | --- | --- | --- |
| `<id>` | string | none, required | The ID of the server. |
| `--json` | bool | `false` | Prints one JSON object, as `admin server get --json` does. |

Example:

```text
$ sudo hopboxd admin server cancel-drain 01a14404-ce7b-72d9-b18e-5006c3e462db | grep -E '^(state|cordoned|drain)'
state	ready
cordoned	false
```

### admin server decommission

```sh
hopboxd admin server decommission <id> [--json]
```

Takes the server `<id>` out of the install for good. Do not run it on a server that Terraform manages. Run `terraform destroy` instead. After a decommission outside Terraform, the provider keeps the first server in its state. A new create erases the install. It calls `DecommissionServer` ([the admin API](/reference/admin-api/index.html.md#decommissionserver)) and prints the server after the change, as `admin server get` does. The install refuses the certificates of the server from then on, and `hopboxd` on that server retries once a minute without success. The command refuses a server that still holds machines. Drain the server first. It also refuses a server that a move runs from or to, and a server that keeps the copy of a disk after a failed move ([the admin API](/reference/admin-api/index.html.md#decommissionserver)). It also refuses a server that hosts a live control process, while no live control process runs on another host and other servers belong to the install. A decommission of a decommissioned server changes nothing.

| Flag | Type | Default | Description |
| --- | --- | --- | --- |
| `--json` | bool | `false` | Prints one JSON object, as `admin server get --json` does. |

### admin control list

```sh
hopboxd admin control list [--json]
```

Lists the control processes that recorded themselves in the control store, oldest start first. A control process is one run of the control role, on this server or on another one. The control role never enrolls, so `admin server list` does not show it. The command calls `ListControlProcesses` ([the admin API](/reference/admin-api/index.html.md#listcontrolprocesses)). A process that ended without its stop, for example after a crash, stays in the list for 24 hours with `LIVE` set to `false`. The text output is a table. Its columns are `ID`, `LIVE`, `HOSTNAME`, `INTERNAL URL`, `RELEASE`, `SCHEMA`, `KEKS`, `MAINTENANCE`, and `LAST RENEWAL`. `SCHEMA` is the schema version of the `hopboxd` of the process. `KEKS` joins the IDs of the key files and the staged key files of the process by commas. `MAINTENANCE` is `off`, `draining` while the process applies maintenance mode and still runs work that it stops, or `drained`. `LAST RENEWAL` is an RFC 3339 time in UTC. An empty value prints as `-`.

| Flag | Type | Default | Description |
| --- | --- | --- | --- |
| `--json` | bool | `false` | Prints one JSON object, `{"processes":[...]}`, with one object for each process. |

Each object has the keys `id`, `server_id`, `hostname`, `internal_url`, `release`, `schema_version`, `kek_ids`, `kek_current`, `kek_fingerprints`, `maintenance`, `drained`, `live`, `start_time`, `renew_time`, `stop_time`, and `version`. `server_id` is empty when no role of the same `hopboxd` enrolled. `stop_time` is `null` unless the process stopped cleanly. `kek_ids` and `kek_fingerprints` are empty lists, never `null`. [The admin API](/reference/admin-api/index.html.md#controlprocess) describes each field.

Example:

```text
$ sudo hopboxd admin control list
ID                                    LIVE  HOSTNAME  INTERNAL URL            RELEASE  SCHEMA  KEKS     MAINTENANCE  LAST RENEWAL
01a14404-ce7b-72d9-b18e-5006c3e462db  true  control1  https://172.31.0.11:7443  v0.2.0   45      default  off          2026-10-26T09:00:05Z
```

### admin pool create

```sh
hopboxd admin pool create <name> [--json]
```

Adds the pool `<name>` ([the admin API](/reference/admin-api/index.html.md#createpool)). It sends a new idempotency key with the call. It prints the pool, one field on each line: the name of the field, a tab, and the value. The fields are `id`, `name`, `hardware`, `server_count`, `size_count`, `version`, `create_time`, and `update_time`. `hardware` is `-` until the first server enrolls into the pool.

| Flag | Type | Default | Description |
| --- | --- | --- | --- |
| `--json` | bool | `false` | Prints one JSON object with the fields of the message `Pool` as keys, apart from the deprecated `ovh_plan`. `hardware` is `null` before the first enrollment, else an object with `cpu_model`, `cpu_threads`, `memory_bytes`, and `disk_bytes`. The times are in RFC 3339 form. |

Example:

```sh
sudo hopboxd admin pool create big
```

### admin pool get

```sh
hopboxd admin pool get <name> [--json]
```

Prints the pool `<name>`, as `admin pool create` does ([the admin API](/reference/admin-api/index.html.md#getpool)).

| Flag | Type | Default | Description |
| --- | --- | --- | --- |
| `--json` | bool | `false` | Prints one JSON object, as `admin pool create --json` does. |

### admin pool list

```sh
hopboxd admin pool list [--json]
```

Lists the pools, by name, in a table ([the admin API](/reference/admin-api/index.html.md#listpools)). The columns are `NAME`, `SERVERS`, `SIZES`, and `HARDWARE`. `SERVERS` counts the servers that are not decommissioned, and `SIZES` the sizes that name the pool and are not retired. `-` stands for a pool without hardware.

| Flag | Type | Default | Description |
| --- | --- | --- | --- |
| `--json` | bool | `false` | Prints one JSON object, `{"pools":[...]}`, with the keys of `admin pool create --json` for each pool. |

```text
$ sudo hopboxd admin pool list
NAME     SERVERS  SIZES  HARDWARE
big      2        1      AMD EPYC 4244P 6-Core Processor, 12 CPU threads, 62.5 GiB of RAM, disks 960 GB, 960 GB
default  1        0      -
```

### admin pool delete

```sh
hopboxd admin pool delete <name>
```

Deletes the pool `<name>` and frees its name ([the admin API](/reference/admin-api/index.html.md#deletepool)). It prints nothing on success. It refuses the pool `default`, a pool with a server that is not decommissioned, and a pool that a live size names.

### admin size create

```sh
hopboxd admin size create <name> --vcpus=N --memory-gib=N --disk-gb=N [--pool=NAME] [--label=KEY=VALUE ...] [--json]
```

Adds the size `<name>` ([the admin API](/reference/admin-api/index.html.md#createsize)). It sends a new idempotency key with the call. A size never changes. To change one, add a size with a new name, and retire the old one. The command prints the size, one field on each line: the name of the field, a tab, and the value. The fields are `name`, `vcpus`, `memory_bytes`, `disk_bytes`, `pool`, `labels`, `retire_time`, `version`, `create_time`, and `update_time`. `labels` holds the `KEY=VALUE` pairs, sorted by key and joined by commas. `retire_time` is empty while the size takes new machines.

| Flag | Type | Default | Description |
| --- | --- | --- | --- |
| `--vcpus` | int | none, required | The count of vCPUs, 1 to 32. |
| `--memory-gib` | int | none, required | The RAM, in GiB. It must be 1 or more. |
| `--disk-gb` | int | none, required | The size of the sparse disk, in GB. It must be 1 or more. |
| `--pool` | string | none | The pool of the machines of the size. Leave it out for any pool. |
| `--label` | `KEY=VALUE` | none | A label that a node must have to take a machine of the size. Repeat the flag for each label. The rules of the labels of a server apply ([the bootstrap config of hopboxd](/reference/hopboxd-config/index.html.md#keys)). |
| `--json` | bool | `false` | Prints one JSON object with the fields of the message `Size` as keys. `retire_time` is `null` while the size takes new machines. The times are in RFC 3339 form. |

Example:

```sh
sudo hopboxd admin size create large --vcpus 8 --memory-gib 32 --disk-gb 200 --pool big --label disk=nvme
```

### admin size get

```sh
hopboxd admin size get <name> [--json]
```

Prints the size `<name>`, also a retired one, as `admin size create` does ([the admin API](/reference/admin-api/index.html.md#getsize)).

| Flag | Type | Default | Description |
| --- | --- | --- | --- |
| `--json` | bool | `false` | Prints one JSON object, as `admin size create --json` does. |

### admin size list

```sh
hopboxd admin size list [--all] [--json]
```

Lists the sizes, smallest first, in a table ([the admin API](/reference/admin-api/index.html.md#listsizes)). The columns are `NAME`, `VCPUS`, `RAM`, `DISK`, `POOL`, `LABELS`, and `RETIRED`. `RAM` is in GiB, or in MiB for a size that is not a whole number of GiB. `DISK` is in GB, or in bytes for a size that is not a whole number of GB. `RETIRED` is the time of the retirement, or `-`.

| Flag | Type | Default | Description |
| --- | --- | --- | --- |
| `--all` | bool | `false` | Also lists the retired sizes. |
| `--json` | bool | `false` | Prints one JSON object, `{"sizes":[...]}`, with the keys of `admin size create --json` for each size. |

```text
$ sudo hopboxd admin size list
NAME     VCPUS  RAM     DISK    POOL  LABELS     RETIRED
default  2      8 GiB   100 GB  -     -          -
large    8      32 GiB  200 GB  big   disk=nvme  -
```

### admin size retire

```sh
hopboxd admin size retire <name> [--json]
```

Stops new machines of the size `<name>` ([the admin API](/reference/admin-api/index.html.md#retiresize)). Its machines keep running and keep their size. Its name stays taken. It prints the size after the change, as `admin size create` does. A retirement of a retired size changes nothing. The command refuses the size `default`.

| Flag | Type | Default | Description |
| --- | --- | --- | --- |
| `--json` | bool | `false` | Prints one JSON object, as `admin size create --json` does. |

### admin org list

```sh
hopboxd admin org list [--json]
```

Lists every org of the install, oldest first, with metadata only. It calls `ListOrgs` of the admin API, page by page, until the last page ([the admin API](/reference/admin-api/index.html.md#listorgs)). Each line has six parts, with a tab between them. They are the org ID, the name, and `personal` or `org`. Then come `members=` and the member count, `active` or `suspended`, and `machines=` with the machines in use, a slash, and their limit.

| Flag | Type | Default | Description |
| --- | --- | --- | --- |
| `--json` | bool | `false` | Prints one JSON object, `{"orgs":[...]}`. Each org has the fields of the message `Org` of the admin API as keys ([the admin API](/reference/admin-api/index.html.md#orgservice)). The times are in RFC 3339 form. An override without a value is `null`. |

```text
$ sudo hopboxd admin org list
01a14565-8448-7753-9c1f-5ec0a86b8d92	Personal	personal	members=1	active	machines=0/3
01a14565-a2df-75a1-9401-ed28dfa37fc4	Team	org	members=2	active	machines=0/3
```

### admin org suspend

```sh
hopboxd admin org suspend --org-id=ID [--reason=TEXT] [--json]
```

Suspends an org, and prints it. It calls `SuspendOrg` ([the admin API](/reference/admin-api/index.html.md#suspendorg)). The members of the org see the reason. Its owners and admins see the audit entry. A suspended org stays as it is, with its first reason.

| Flag | Type | Default | Description |
| --- | --- | --- | --- |
| `--org-id` | string | none, required | The UUID of the org. |
| `--reason` | string | empty | The reason, which the members of the org see. At most 256 characters. |
| `--json` | bool | `false` | Prints the org as one JSON object, with the keys of `admin org list`. |

Without `--json`, the command prints one line per fact: a name, a tab, and a value. Each limit line holds the usage, a slash, and the limit, in bytes for RAM and disk.

```text
$ sudo hopboxd admin org suspend --org-id 01a14565-a2df-75a1-9401-ed28dfa37fc4 --reason 'spam reports'
id	01a14565-a2df-75a1-9401-ed28dfa37fc4
name	Team
members	2
state	suspended
suspend_time	2026-10-16T15:47:41Z
suspend_reason	spam reports
machines	0/10
vcpus	0/6
memory_bytes	0/68719476736
disk_bytes	0/300000000000
version	3
```

A personal org also prints the line `personal_account_id`.

### admin org unsuspend

```sh
hopboxd admin org unsuspend --org-id=ID [--json]
```

Ends the suspension of an org, and prints it as `admin org suspend` does. It calls `UnsuspendOrg` ([the admin API](/reference/admin-api/index.html.md#unsuspendorg)).

| Flag | Type | Default | Description |
| --- | --- | --- | --- |
| `--org-id` | string | none, required | The UUID of the org. |
| `--json` | bool | `false` | Prints the org as one JSON object. |

### admin org set-limits

```sh
hopboxd admin org set-limits --org-id=ID [--machines=N] [--vcpus=N] [--memory-gib=N] [--disk-gb=N] [--expected-version=N] [--json]
```

Replaces the usage limits of one org, and prints the org as `admin org suspend` does. It calls `SetOrgLimits` ([the admin API](/reference/admin-api/index.html.md#setorglimits)). A limit that you leave out takes the default of the install. So `set-limits` with only `--org-id` puts the org back on the defaults.

| Flag | Type | Default | Description |
| --- | --- | --- | --- |
| `--org-id` | string | none, required | The UUID of the org. |
| `--machines` | int32 | the default | The most machines. |
| `--vcpus` | int32 | the default | The most vCPUs of all machines together. |
| `--memory-gib` | int64 | the default | The most RAM of all machines together, in GiB (2^30 bytes). |
| `--disk-gb` | int64 | the default | The most disk of all machines together, in GB (10^9 bytes). |
| `--expected-version` | int64 | none | The version that the org must have now. With another version, the command changes nothing and fails with `aborted`. |
| `--json` | bool | `false` | Prints the org as one JSON object. |

Each limit is 0 or more. The command refuses a negative `--memory-gib` or `--disk-gb`, and the API refuses a negative `--machines` or `--vcpus` with `invalid_argument`.

```sh
sudo hopboxd admin org set-limits --org-id 01a14565-a2df-75a1-9401-ed28dfa37fc4 --machines 10 --memory-gib 64
```

### admin org default-limits get

```sh
hopboxd admin org default-limits get [--json]
```

Prints the limits of every org without its own limits. It calls `GetDefaultOrgLimits` ([the admin API](/reference/admin-api/index.html.md#getdefaultorglimits)).

| Flag | Type | Default | Description |
| --- | --- | --- | --- |
| `--json` | bool | `false` | Prints one JSON object with the keys `limits`, `max_orgs_per_account`, `version`, and `update_time`. |

```text
$ sudo hopboxd admin org default-limits get
machines	3
vcpus	6
memory_bytes	25769803776
disk_bytes	300000000000
max_orgs_per_account	10
version	3
update_time	2026-10-16T15:47:06Z
```

The version is that of the whole install config.

### admin org default-limits set

```sh
hopboxd admin org default-limits set [--machines=N] [--vcpus=N] [--memory-gib=N] [--disk-gb=N] [--expected-version=N] [--json]
```

Changes the default limits of every org, and prints them as `default-limits get` does. It calls `SetDefaultOrgLimits` ([the admin API](/reference/admin-api/index.html.md#setdefaultorglimits)). A limit that you leave out keeps its value. The command needs at least one limit.

| Flag | Type | Default | Description |
| --- | --- | --- | --- |
| `--machines`, `--vcpus`, `--memory-gib`, `--disk-gb` | as for `set-limits` | the value now | The new defaults. |
| `--max-orgs-per-account` | int32 | the value now | The most orgs that one account owns besides its personal org. |
| `--expected-version` | int64 | none | The version that the install config must have now. With another version, the command changes nothing and fails with `aborted`. |
| `--json` | bool | `false` | Prints one JSON object. |

### admin admin-key create

```sh
hopboxd admin admin-key create --name=NAME [--lifetime=DAYS | --expire-time=TIME] [--json]
```

Makes an admin API key, for the hopbox Terraform provider and scripts that call the admin API on the API listener ([CLI tokens, API keys, admin API keys, and SSH keys](/reference/credentials/index.html.md#admin-api-keys)). It calls `CreateAdminKey` ([the admin API](/reference/admin-api/index.html.md#createadminkey)) with a new idempotency key. It prints the key alone on one line, so a script can keep it in a variable. The install keeps only a hash of the key, so this output is its only copy. Standard error gets the ID, the name, and the end of the key.

| Flag | Type | Default | Description |
| --- | --- | --- | --- |
| `--name` | string | none, required | A name of 1 to 64 characters that tells the keys apart, such as `terraform`. |
| `--lifetime` | `30d`, `90d`, or `365d` | `90d` | The lifetime of the key. |
| `--expire-time` | RFC 3339 time | none | A custom end of the key, such as `2027-01-31T00:00:00Z`, at most 365 days ahead. Give `--lifetime` or `--expire-time`, not both. |
| `--json` | bool | `false` | Prints one JSON object with the keys `id`, `name`, `expire_time`, `last_use_time` (`null`), `create_time`, and `secret`. |

```text
$ ADMIN_KEY="$(sudo hopboxd admin admin-key create --name terraform)"
The admin key 01a145b6-3fc1-7ba1-a54d-d1dceac38930 (terraform) works until 2027-01-14T17:15:34Z. This is its only copy.
$ curl -s -X POST -H 'Content-Type: application/json' -H "Authorization: Bearer $ADMIN_KEY" -d '{}' \
    http://127.0.0.1:7080/hopbox.admin.v1.InstallConfigService/GetInstallConfig
{"config":{"domain":"example.com","version":"2","updateTime":"2026-10-16T17:14:47.254670Z"}}
```

### admin admin-key register

```sh
hopboxd admin admin-key register --name=NAME --sha256=HASH [--lifetime=DAYS | --expire-time=TIME] [--json]
```

Makes an admin API key from the SHA-256 of a key that you made, so the key itself never reaches the server ([CLI tokens, API keys, admin API keys, and SSH keys](/reference/credentials/index.html.md#admin-api-keys)). It calls `RegisterAdminKey` ([the admin API](/reference/admin-api/index.html.md#registeradminkey)) and prints the key as `admin-key list` does. A second registration of the same hash prints the first key, unchanged, and writes `An earlier call registered this hash already. The key stays as it was.` to standard error. It still exits with status 0.

| Flag | Type | Default | Description |
| --- | --- | --- | --- |
| `--name` | string | none, required | A name of 1 to 64 characters, such as `bootstrap`. |
| `--sha256` | string | none, required | The SHA-256 of the whole key text, `hbxadm_` included, in 64 lowercase hex characters. |
| `--lifetime` | `30d`, `90d`, or `365d` | `90d` | The lifetime of the key. |
| `--expire-time` | RFC 3339 time | none | A custom end of the key, at most 365 days ahead. |
| `--json` | bool | `false` | Prints one JSON object with the keys of `admin-key list --json`, plus `registered` (`false` for a hash that an earlier call registered). |

```sh
KEY="hbxadm_$(head -c 32 /dev/urandom | base64 | tr '+/' '-_' | tr -d '=')"
sudo hopboxd admin admin-key register --name bootstrap --sha256 "$(printf '%s' "$KEY" | shasum -a 256 | cut -d' ' -f1)"
```

```text
01a145b6-400d-7c41-979e-14666fc2fe21	bootstrap	2027-01-14T17:15:34Z	never
```

### admin admin-key list

```sh
hopboxd admin admin-key list [--json]
```

Lists the admin API keys, oldest first. It calls `ListAdminKeys` ([the admin API](/reference/admin-api/index.html.md#listadminkeys)). Each line has four fields, split by tabs: the ID, the name, the end of the key, and the time of its last use, or `never`.

| Flag | Type | Default | Description |
| --- | --- | --- | --- |
| `--json` | bool | `false` | Prints one JSON object, `{"keys":[...]}`. Each key has the keys `id`, `name`, `expire_time`, `last_use_time` (`null` for a key without a call), and `create_time`. |

```text
$ sudo hopboxd admin admin-key list
01a145b6-3fc1-7ba1-a54d-d1dceac38930	terraform	2027-01-14T17:15:34Z	2026-10-16T17:15:34Z
01a145b6-400d-7c41-979e-14666fc2fe21	bootstrap	2027-01-14T17:15:34Z	never
```

### admin admin-key revoke

```sh
hopboxd admin admin-key revoke --id=ID
```

Deletes an admin API key at once. It calls `RevokeAdminKey` ([the admin API](/reference/admin-api/index.html.md#revokeadminkey)). The next call with the key gets `unauthenticated`.

| Flag | Type | Default | Description |
| --- | --- | --- | --- |
| `--id` | string | none, required | The UUID of the key, as `admin-key list` prints it. |

### admin mail list

```sh
hopboxd admin mail list [--json]
```

Lists the mail files that the file mail transport wrote on this server, newest first ([Mail](/reference/mail/index.html.md#the-file-transport)). With several control servers, each one writes the mail of the calls that it takes into its own folder, so run the command on each control server. It calls `ListMailFiles` ([the admin API](/reference/admin-api/index.html.md#listmailfiles)), which prunes the folder first. Each line has four fields, split by tabs: the name of the file, the time of the write in RFC 3339, the `To` header, and the `Subject` header. Without files, it prints `hopboxd: no mail files in <dir>` to standard error and nothing to standard output. The files stay after a switch back to the `smtp` transport, until a prune removes them.

| Flag | Type | Default | Description |
| --- | --- | --- | --- |
| `--json` | bool | `false` | Prints one JSON object, `{"dir":"/var/lib/hopbox/mail","files":[{"name":"...","create_time":"...","to":"<ann@example.org>","subject":"...","size_bytes":712}]}`. |

Example:

```text
$ sudo hopboxd admin mail list
20261023T094602.118205000Z-0c8e1f2a9b3d4e5f.eml	2026-10-23T09:46:02Z	<bob@example.org>	Join Team on dev.example.org
20261023T094531.606059000Z-5cb912a8d0bb45d7.eml	2026-10-23T09:45:31Z	<ann@example.org>	Sign in to dev.example.org
```

### admin mail show

```sh
hopboxd admin mail show [NAME] [--raw | --json]
```

Prints one mail file of this server ([Mail](/reference/mail/index.html.md#read-the-mail)). It calls `GetMailFile` ([the admin API](/reference/admin-api/index.html.md#getmailfile)). Without `NAME`, it calls `ListMailFiles` first and prints the newest file. The audit log records each read.

The output starts with the lines `name`, `create_time`, `to`, and `subject`, each with a tab and the value. An empty line and the text of the mail follow. The text is the body, decoded from quoted-printable, so a link is whole on one line.

| Argument or flag | Type | Default | Description |
| --- | --- | --- | --- |
| `NAME` | string | the newest file | The name of the file, as `admin mail list` prints it. |
| `--raw` | bool | `false` | Prints the file as it is on disk: an RFC 5322 message with CRLF line ends, in quoted-printable. A mail client opens it. |
| `--json` | bool | `false` | Prints one JSON object with the keys `name`, `create_time`, `to`, `subject`, `size_bytes`, `text`, and `content`. `content` is the raw file. |

Example:

```text
$ sudo hopboxd admin mail show
name	20261023T094531.606059000Z-5cb912a8d0bb45d7.eml
create_time	2026-10-23T09:45:31Z
to	<ann@example.org>
subject	Sign in to dev.example.org

Open this link to sign in to dev.example.org:

https://dev.example.org/auth/email#token=Zm9vYmFyYmF6cXV4cXV1eGNvcmdlZ3JhdWx0Z2FycGx5

The link works once, for 15 minutes. If you did not ask for it, you can ignore this email.
```

| Error | Cause |
| --- | --- |
| `no mail files in <dir>` | `admin mail show` without `NAME`, and the folder holds no file. |
| `invalid_argument: invalid argument: the name is not the name of a mail file, ...` | `NAME` is not the name of a mail file, for example a path. |
| `not_found: not found: the mail file <name> is not in <dir>. ...` | No file has the name. A prune removes a file after 7 days. It also removes the oldest files beyond 1000. |

### admin maintenance on

```sh
hopboxd admin maintenance on [--json]
```

Turns maintenance mode of the install on ([the API server](/reference/api/index.html.md#maintenance-mode)). It calls `SetMaintenance` with `maintenance` set to `true` ([the admin API](/reference/admin-api/index.html.md#setmaintenance)). In maintenance mode, the API takes only the reads that write nothing, the control role ends every control channel and takes no new one, and the leader pauses its jobs. Every `hopboxd admin` command fails with `unavailable`, but `admin maintenance on` and `off`, because each other admin call writes an audit entry ([the API server](/reference/api/index.html.md#maintenance-mode)). The machines keep running. Their URLs and the SSH front door answer no client until maintenance mode ends, because the edge servers lose their channels too. An open SSH session to a machine ends in the window, because its edge server cannot check its access again.

The command succeeds once the writes, the control channels, and the leader jobs that ran have ended while maintenance mode stayed on. This holds for every live control process of the install, not only for the one that takes the call. Each other live control process must report maintenance mode on and drained in a renewal of its row after the turn, and the call polls the rows every 250 ms. A control process that stops renewing its row counts as ended. The wait compares versions, not times. Each control process reports in its row the version of `install_config` at which it read maintenance mode (`maintenance_version`), and the call waits until every other live process reports maintenance mode on, drained, and a `maintenance_version` at least the version that the turn wrote. A control process of a release before migration `0045` records no row, so the command does not see it, and neither does the migration guard, the KEK promote, the decommission check, or the rolling upgrade order. Upgrade every control server before you rely on these checks. So after a success, no change of the API or of a server reaches the control store. `hopboxd admin db export` runs it first. On an install that is in maintenance mode already, it changes nothing, and waits the same way. Since every admin read is refused in maintenance mode, a second `admin maintenance on` is the harmless way to print the settings, `maintenance` among them, in that window.

It prints the settings of the install as [admin config get](#admin-config-get) does, with `maintenance` set to `true`.

| Flag | Type | Default | Description |
| --- | --- | --- | --- |
| `--json` | bool | `false` | Prints the settings as one JSON object, as `admin config get --json` does. |

```text
$ sudo hopboxd admin maintenance on
domain	example.com
...
maintenance	true
version	5
update_time	2026-10-25T10:00:00Z
```

| Error | Cause |
| --- | --- |
| `unavailable: unavailable: <n> calls, control channels, or leader jobs still run: context deadline exceeded` | Some work did not end within the deadline of the call. Maintenance mode stays on. Run the command again. |
| `failed_precondition: failed precondition: the control process <id> on <host> (hopboxd <rel>) knows the migrations up to <n>, and only a hopboxd that knows migration 44 honors maintenance mode. Upgrade it first` | A live control process runs a `hopboxd` of schema version 43 or less. Maintenance mode stays off. Only releases from migration `0045` on write rows, so this guards the rows of a future release. |
| `unavailable: unavailable: maintenance mode is on, but the control process <id> on <host> (hopboxd <rel>) did not report that its calls, control channels, and leader jobs ended: ...` | Another control process did not report that it is drained before the deadline of the call. Maintenance mode stays on. Run the command again. |
| `unavailable: unavailable: maintenance mode turned off while it waited for the calls, control channels, and leader jobs to end` | Maintenance mode turned off during the wait, for example by `admin maintenance off` in another shell. The work runs again. Run the command again. |

### admin maintenance off

```sh
hopboxd admin maintenance off [--json]
```

Turns maintenance mode of the install off. It calls `SetMaintenance` with `maintenance` set to `false`. The admin socket takes this call also in maintenance mode, because no other write could turn it off. The API takes writes again, the servers dial their control channels again within 30 s, and their heartbeats bring new grants of the machine leases. The leader runs its jobs again, and gives the servers 20 s to dial before it marks a silent server `unresponsive`.

It prints the settings of the install, with `maintenance` set to `false`.

| Flag | Type | Default | Description |
| --- | --- | --- | --- |
| `--json` | bool | `false` | Prints the settings as one JSON object, as `admin config get --json` does. |

```text
$ sudo hopboxd admin maintenance off
domain	example.com
...
maintenance	false
version	6
update_time	2026-10-25T10:05:00Z
```

### admin db export

```sh
sudo hopboxd admin db export --to=URL --ca-file=PATH --password-file=PATH [--json]
```

Copies the control store of this server from the bundled Postgres to an external Postgres, such as managed PostgreSQL. An install with one server uses it once, to move to the external Postgres that several control servers share. [Run the control store on an external Postgres](/guide/external-postgres/index.html.md#move-a-one-server-install-to-the-external-postgres) has the whole move. The code is in `internal/dbexport`.

| Flag | Type | Default | Description |
| --- | --- | --- | --- |
| `--to` | URL | none, required | The external Postgres, in the form of `postgres.url`: `postgres://<user>@<host>[:<port>]/<database>?sslmode=verify-full`, without a password ([the bootstrap config of hopboxd](/reference/hopboxd-config/index.html.md#keys)). Its database must be empty. |
| `--ca-file` | path | none, required | The PEM file of the CA of the certificate of the external Postgres, as `postgres.ca_file` takes it. A relative path becomes absolute. |
| `--password-file` | path | none, required | The file that holds the password of the user of `--to`, as `postgres.password_file` takes it. As root, only root may read it. A relative path becomes absolute. |
| `--json` | bool | `false` | Prints one JSON object with the keys `target`, `target_version`, `migration`, `tables`, `rows`, `duration` (in nanoseconds), and `config` (the config lines). |

Run it as the user of `hopboxd`, root on a server. It reads the bootstrap config of `--config` for the paths. The server must have the control role, and no `postgres.url`. The export takes these steps:

1. It connects to the bundled Postgres over its socket, and to the target with TLS and `sslmode=verify-full`, with the CA file and the password file. It checks the files as `hopboxd run` checks them.
2. It checks the target. The target must run the major version of the bundled Postgres or a newer one (plan question Q68), so Postgres 18 or later. Its user must be able to create tables in the schema `public`. Its database must hold no table, view, or sequence outside the catalogs, except those of an extension. A refusal of steps 1 and 2 changes nothing.
3. It turns maintenance mode on through the admin socket, and waits until the writes and the control channels of the install ended ([admin maintenance on](#admin-maintenance-on)).
4. It opens a read-only transaction on the bundled Postgres, and exports its snapshot. In that snapshot, it checks that the flag of maintenance mode is on, reads the last migration, and counts the rows of each table of the schema `public`. A store out of maintenance mode, for example after `admin maintenance off` during the export, is refused, because it can still change. Then `pg_dump` dumps the bundled Postgres in that snapshot, in the custom format, to `<data_dir>/export/hopbox-<time>.dump`. The export makes the dir `<data_dir>/export` with mode `0700`, and sets that mode also on a dir that exists. It makes the dump file first, empty, with mode `0600`, and `pg_dump` writes into it and keeps that mode, whatever the umask. The dump holds the sealed secrets of the install, so no other user may read it. So the counts describe the dump exactly.
5. `pg_restore` restores the dump into the target in one transaction, and stops at the first error. Every object gets the user of `--to` as its owner, and keeps no grant of the bundled Postgres. A failed restore leaves the target empty.
6. It checks the copy: the same last migration in `schema_migrations`, the same tables, and the same row count in each table.
7. It deletes the rows of `control_processes` on the copy, and clears `servers.channel_process` there. They name the processes of the source, which never run on the copy, and a row that still looks live would hold back the checks of the new control servers ([Control store tables](/reference/control-store/index.html.md#control_processes)).
8. It removes the dump, prints the config lines, and tells the operator to stop `hopboxd` on this server.

Steps 4 and 5 run these two programs of `<postgres.install>/bin`:

```text
pg_dump --format=custom --snapshot=<snapshot> --file=<data_dir>/export/hopbox-<time>.dump --dbname=<the socket of the bundled Postgres>
pg_restore --no-owner --no-privileges --single-transaction --exit-on-error --dbname=<--to, with sslrootcert=<--ca-file>> <dump>
```

They run with the env `LC_ALL=C`, `TZ=UTC`, `PGAPPNAME=hopboxd-export`, and `OPENSSL_CONF=/dev/null`, and no other env of the caller. So the bundled OpenSSL reads no `openssl.cnf` of the host. The password reaches `pg_restore` in a password file in the form of `.pgpass`, `<data_dir>/export/pgpass-<random>`, with mode `0600`, which the env var `PGPASSFILE` names. It never shows in the arguments or the env of `pg_restore`, and the export removes the file after the restore.

On success, it prints the config lines for the new control server to standard output. Add them to the table `[postgres]` of its bootstrap config:

```text
$ sudo hopboxd admin db export --to 'postgres://avnadmin@db.example.org:20184/hopbox?sslmode=verify-full' \
    --ca-file /etc/hopbox/postgres-ca.pem --password-file /etc/hopbox/postgres-password
hopboxd: copied the control store to postgres://avnadmin@db.example.org:20184/hopbox (Postgres 18.1): migration 44, 61 tables, 2317 rows, in 4s.
hopboxd: the install stays in maintenance mode. Stop hopboxd on this server now, so that two control planes never run: sudo systemctl stop hopboxd. Then give the new control server these lines in its bootstrap config and the key files of /var/lib/hopbox/kek, start it, and run hopboxd admin maintenance off there.
[postgres]
url = "postgres://avnadmin@db.example.org:20184/hopbox?sslmode=verify-full"
ca_file = "/etc/hopbox/postgres-ca.pem"
password_file = "/etc/hopbox/postgres-password"
```

The lines that start with `hopboxd:` go to standard error. The export also writes log records to standard error, with `component=dbexport`:

| Message | Level | Fields |
| --- | --- | --- |
| `maintenance mode is on, so the store no longer changes` | info | |
| `dumped the control store` | info | `tables`, `migration` |
| `restored the control store` | info | `target`, the URL without its query |
| `the copy holds every row of the control store` | info | `tables`, `rows`, `duration` |

The install stays in maintenance mode, and so does the copy, because the flag lives in the store. So neither control plane serves. In maintenance mode, no call writes to the store, also no read that would write its audit entry ([the API server](/reference/api/index.html.md#maintenance-mode)). Only the internal writes of the control process go on, such as the renewals of the leader lease. So the copy misses no change of the API or of a server. Stop `hopboxd` on this server right after the export, so it never serves again. Its bundled cluster stays on disk. A new control server then starts on the copy with these lines, the key-encryption keys of `<data_dir>/kek` ([Sealing](/reference/sealing/index.html.md#key-files)), and `control` in `roles`. Then `hopboxd admin maintenance off` on that server lets the servers dial it, and their heartbeats bring new grants of the machine leases, with new epochs.

A failed step 3 can leave maintenance mode off or on: a turn to on can commit, and then fail in its wait for the calls that run. So after a failed step 3, the export reads the flag back from the bundled Postgres. After a failed step 3 with the flag on, or a failed step 4 or later, the install stays in maintenance mode, except after the refusal of a store out of maintenance mode in step 4. Fix the cause and run the export again, or run `hopboxd admin maintenance off` on this server to go back to the bundled Postgres. The export takes at most 1 hour.

| Error | Cause |
| --- | --- |
| `the config <path> names no control role, so this server holds no control store` | The server has no control role. |
| `the control store of this server is the external Postgres of postgres.url in <path> already. db export copies only the bundled Postgres` | The config has `postgres.url`. |
| `dbexport: refused: --to: "<url>" needs sslmode=verify-full, so hopboxd checks the certificate of the server and its host name. The export changed nothing` | `--to` breaks a rule of `postgres.url`. Each error of `postgres.url` can come here ([the bootstrap config of hopboxd](/reference/hopboxd-config/index.html.md#errors)). |
| `dbexport: refused: postgres.password_file: ...` or `dbexport: refused: postgres.ca_file: ...` | A file is missing, or fails a check of the files of the bootstrap config. |
| `dbexport: refused: connect to the target <url>: ... x509: certificate signed by unknown authority ...` | The CA of `--ca-file` did not sign the certificate of the target. |
| `dbexport: refused: the target runs Postgres <version>, older than the Postgres <n> of this server. Give a Postgres <n> or newer` | The target runs an older major version. |
| `dbexport: refused: the user <user> cannot create tables in the schema public of the database <db>. Make the user the owner of the database` | The user of `--to` lacks the right to create tables. |
| `dbexport: refused: the database <db> of the target holds <n> tables, such as <schema>.<name>. Give an empty database, because the copy goes into one transaction and replaces nothing` | The database is not empty, for example after an earlier export. Drop and make the database again. |
| `dbexport: refused: the bundled Postgres of this server holds no control store` | The bundled Postgres has no schema. |
| `connect to the bundled Postgres of this server: ...` | `hopboxd` does not run, or the export runs as another user than `hopboxd`. |
| `turn maintenance mode on: ...` | The call to the admin socket failed ([admin maintenance on](#admin-maintenance-on)). |
| `dbexport: maintenance mode is off in the snapshot of the dump, so the store can still change. Turn maintenance mode on, and run the export again` | Maintenance mode turned off after step 3. The export dumped nothing, and the install runs as before. |
| `pg_dump: ...` or `pg_restore: ... The restore ran in one transaction, so the target stays empty` | The program failed. The error ends with the last 4 KiB of its output. |
| `dbexport: check the copy: ... The copy stays in the target, so drop the database of <url> and make it again, empty, before the next export` | The copy differs from the dump: another migration, other tables, or another row count. The copy stays in the target, so the next export refuses the database until you drop it and make it again. |

Each error that leaves the install in maintenance mode ends with `The install stays in maintenance mode, and this server keeps its bundled Postgres. Fix the cause and run the export again, or run hopboxd admin maintenance off to go back`: each error of step 4 or later but that refusal, and an error of step 3 after which the flag of the bundled Postgres is on. An error of step 3 with the flag off ends without it, and the install runs as before.

If the source must come back instead, run `hopboxd admin maintenance off` on it. Do not roll it back to a release older than schema version 44 while it is in maintenance mode: such a release does not know the flag and serves writes again.

`make test` runs `TestExport` and `TestExportRefusals` of `internal/dbexport` against a TLS Postgres of the bundled install, whose user is no superuser ([the bundled Postgres](/reference/postgres/index.html.md#tests)). `TestExport` then runs the migrations, a write, a change event, and the leader election on the copy. `TestExportRefusals` covers a target with a table, a wrong CA, a URL without `verify-full`, a user that cannot create tables, and a failed dump, which keeps maintenance mode on and the target empty. `TestCompareMajor` covers the refusal of an older major version. The fleet test runs the export with live nodes.

### Errors of admin

| Error | Cause |
| --- | --- |
| `no hopboxd serves the admin socket <path>. Is hopboxd running? ...` | The socket file does not exist, or no process listens on it. |
| `cannot open the admin socket <path>. Run the command as root, for example with sudo: ...` | The mode of the socket lets only root in. |
| an error that ends with `broken pipe` or `EOF` | The socket closed the connection, because the caller is neither root nor the user of `hopboxd`. The log of `hopboxd` has the record `admin socket refused a peer`. |
| `<code>: <message>` | The admin API refused the call. [The API server](/reference/api/index.html.md#errors) lists the codes. |
| `give at least one setting to change, such as --domain` | `admin config set` without a setting flag. |
| `standard input holds no password. To remove the password, use --clear-password` | `admin config smtp set --password-stdin` with an empty standard input. |
| `missing flags: --account-id=ID or --email=EMAIL` | `admin instance-admin grant` without an account. The exit status is 80. |
| `failed_precondition: failed precondition: the keyring has no key-encryption key to seal or open the value. Run hopboxd admin kek rotate to make the first one` | `admin enrollment-token create` before the first key-encryption key. |
| `not_found: not found: no pool is named "<pool>"` | `admin enrollment-token create --pool` with a pool that does not exist. |
| `invalid_argument: invalid argument: the server ID "<id>" is not a UUID` | `admin server get`, `cordon`, `uncordon`, `set-labels`, `drain`, or `decommission` with a bad ID. |
| `not_found: not found: no server has the ID <id>` | No server has the ID. |
| `--lifetime must be 30d, 90d, or 365d, not "<value>"` | `admin admin-key create` or `register` with another lifetime. |
| `--lifetime and --expire-time can't be used together` | Both flags at once. The exit status is 80. |
| `not_found: not found: no admin key <id>` | `admin admin-key revoke` of a key that does not exist. |
| `failed_precondition: failed precondition: the server <id> is decommissioned` | `admin server cordon`, `uncordon`, `set-labels`, or `drain` of a decommissioned server. |
| `failed_precondition: failed precondition: the install has a control server on <host> too, and a rotation makes the new key current only on this server. Run hopboxd admin kek stage, copy the staged file to every control server, then run hopboxd admin kek promote` | `admin kek rotate` while the install has a control process on another host, live or not. |
| `failed_precondition: failed precondition: a key file holds another key than the install recorded for its ID: ...` | `admin kek stage`, `admin kek promote`, or a start of the control role, with a key file whose fingerprint differs from the one that the install recorded for its ID ([Sealing](/reference/sealing/index.html.md#key-fingerprints)). |
| `failed_precondition: failed precondition: the control process <id> on <host> (hopboxd <rel>) does not hold the current key <key> with this fingerprint, ...` | `admin kek rewrap` while a live control process lacks the current key. |
| `invalid_argument: invalid argument: "<id>" is not the ID of a file key, such as file:2` | `admin kek promote` with an ID that is no file key ID. |
| `failed_precondition: failed precondition: this control server has no key file of <id>. Stage it with hopboxd admin kek stage first` | `admin kek promote` of a key that this server has neither as a staged file nor as a key file. |
| `failed_precondition: failed precondition: the key file <k> is newer than <id>, so <id> would never seal. Stage a new key instead` | `admin kek promote` of a key when a key file with a higher number exists. |
| `failed_precondition: failed precondition: the control process <id> on <host> (hopboxd <release>) reports no key file of <key>. Copy the staged file to the key dir of that server with mode 0400, and promote again after its next renewal, within 5 s` | `admin kek promote` while a live control process does not list the key in `kek_ids`. |
| `unavailable: unavailable: <key> is promoted, but the control process <id> still seals with <old key>: ...` | `admin kek promote` ran out of time while a process still seals with the old key. The promote stays recorded. |
| `failed_precondition: failed precondition: the server <id> holds <n> machines, such as <names>. Drain it first, so that its machines move to other nodes, or delete them` | `admin server decommission` of a server with machines. |
| `failed_precondition: failed precondition: the move <move id> of the machine <machine id> runs from or to the server <id>. Wait for its end, or cancel it` | `admin server decommission` of a server that a move runs from or to. |
| `failed_precondition: failed precondition: the server <id> keeps a copy of the disk of the machine <name>, which a failed move left <state> on another node. Give the machine back with hopboxd admin machine move --cancel --give-back, delete it, or keep the server` | `admin server decommission` of a server that keeps the only copy of a disk. |
| `failed_precondition: failed precondition: the server <id> drains, and stays cordoned until the drain decommissions it. To keep the server, cancel the drain with hopboxd admin server cancel-drain <id>` | `admin server uncordon` of a server that drains. |
| `failed_precondition: failed precondition: the server <id> does not drain` | `admin server cancel-drain` of a server without a drain. |
| `failed_precondition: failed precondition: the control process <process> knows the migrations up to <n>, and only a hopboxd that knows migration 51 drains servers. Upgrade it first` | `admin server drain` while a live control process runs a release of the schema version below 51. |
| `failed_precondition: failed precondition: the server <id> has no role node, so it holds no machine to move. Decommission it instead` | `admin server drain` of a server without the role `node`. |
| `failed_precondition: failed precondition: the server <id> has a pending removal of the roles <roles>. Wait for its end, or cancel it with hopboxd admin server cancel-remove-role <id>` | `admin server drain` of a server with a pending removal of roles, or `admin server remove-role` with other roles than the pending removal. |
| `failed_precondition: failed precondition: the removal of the node role keeps the server <id> cordoned, so that its machines move off. Cancel the removal first with hopboxd admin server cancel-remove-role <id>` | `admin server uncordon` of a server with a pending removal of `node`. |
| `unimplemented: unimplemented: the role control cannot leave a running server in this release. ...` | `admin server remove-role` that names `control`. |
| `failed_precondition: failed precondition: the roles <roles> cannot leave the server <id>: <reason>` | `admin server remove-role` that breaks a check of `RemoveServerRoles` ([the admin API](/reference/admin-api/index.html.md#errors-of-serverservice)). |
| `failed_precondition: failed precondition: the server <id> has no pending removal of a role` | `admin server cancel-remove-role` of a server without a removal. |
| `failed_precondition: failed precondition: the server <id> shares its host with the last live control process of the install, ...` | `admin server decommission` of the server on the host of the last live control process, before the other servers. Decommission them first, or start the control role on another server. |
| `invalid_argument: invalid argument: invalid label: ...` | `admin server set-labels` with a label that breaks a rule. |
| `--memory-gib <n> is too large`, `--disk-gb <n> is too large` | `admin org set-limits` or `admin org default-limits set` with a size of more than 2^63 - 1 bytes. |
| `--memory-gib must be from 1 to <max>, not <n>`, `--disk-gb must be from 1 to <max>, not <n>` | `admin size create` with a size of 0, a negative size, or a size of more than 2^63 - 1 bytes. |
| `already_exists: already exists: a pool is named "<name>" already` | `admin pool create` with the name of a live pool. |
| `already_exists: already exists: a size is named "<name>" already. ...` | `admin size create` with the name of a size, also of a retired one. |
| `not_found: not found: no pool is named "<name>"` | `admin pool get` or `delete` of an unknown pool, or `admin size create --pool` with one. |
| `not_found: not found: no size is named "<name>"` | `admin size get` or `retire` of an unknown size. |
| `failed_precondition: failed precondition: the pool <name> holds <n> servers. Decommission them first` | `admin pool delete` of a pool with servers. |
| `failed_precondition: failed precondition: <n> sizes name the pool <name>. Retire them first` | `admin pool delete` of a pool that live sizes name. |
| `failed_precondition: failed precondition: the pool default stays, ...`, `failed_precondition: failed precondition: the size default stays, ...` | `admin pool delete default` or `admin size retire default`. |
| `--memory-gib must be 0 or more, not <n>`, `--disk-gb must be 0 or more, not <n>` | A negative size. |
| `missing flags: --name=NAME or --id=ID` | `admin machine stop` or `delete` without a machine. The exit status is 80. |
| `--name and --id can't be used together` | `admin machine stop` or `delete` with both flags. The exit status is 80. |
| `<metadata>: stat <path>: no such file or directory` | `admin image register` with a file that does not exist. The exit status is 80. |
| `not_found: not found: no machine has the name "<name>"` | `admin machine stop` or `delete` of an unknown machine. |
| `failed_precondition: failed precondition: machine action not allowed: ...` | `admin machine stop` of a hibernated machine, or a second `admin machine delete`. |
| `invalid_argument: invalid argument: invalid artifact metadata: ...` | `admin image register` with a file that breaks the artifact format ([Artifact format](/reference/artifact-format/index.html.md)). |

## stop-guest

```sh
hopboxd stop-guest --api-socket PATH [--timeout DURATION]
```

The unit of each machine runs `stop-guest` as its `ExecStop` ([the Firecracker driver](/reference/firecracker-driver/index.html.md#a-stop-through-systemd)). So a reboot of the server, or a `systemctl stop` of the unit, shuts the guest down cleanly. You do not run it yourself. The help of `hopboxd` does not list it.

| Flag | Type | Default | Description |
| --- | --- | --- | --- |
| `--api-socket` | path | none, required | The API socket of the Firecracker of the machine. |
| `--timeout` | duration | `30s` | How long to wait for the end of Firecracker. |

`stop-guest` reads the process ID of Firecracker from the env var `MAINPID`, which systemd sets for `ExecStop`. Without `MAINPID`, Firecracker ended already, and `stop-guest` exits with status 0 at once. Otherwise it sends `SendCtrlAltDel` through the API socket, and waits for the end of the process. If Firecracker ends within the timeout, it exits with status 0. If Firecracker does not end in time, it exits with status 1. If the call fails, and Firecracker does not end within 2 s, it exits with status 1 too. The unit ignores the status, and systemd then ends Firecracker with `SIGTERM`. It logs to the journal of the unit, at level info. Without `MAINPID`, it logs `Firecracker ended before the stop, so the guest needs no shutdown`. Otherwise it logs `asking the guest to shut down`, with `pid` and `timeout`, and after the end of Firecracker `the guest shut down`, with `duration`.

The flags of `stop-guest` stay the same in each release. A unit keeps the command line of the release that booted the machine, and runs the `hopboxd` of the current release.

## Logs

`hopboxd` writes its logs to standard error. Each record is one JSON object on one line, with the fields `time`, `level`, and `msg`, plus the fields of the record. Under systemd, standard error goes to journald.

This record shows the format only. `hopboxd` does not write this message:

```json
{"time":"2026-10-16T09:00:00.000000000Z","level":"INFO","msg":"example","machine":"box1"}
```

A field of type duration is a number of nanoseconds.

`run` writes these records. The records of the daemon have `component=daemon`.

| Message | Level | Fields |
| --- | --- | --- |
| `hopboxd starting` | info | `version`, `roles`, `config` |
| `the bootstrap config uses an old name` | warn | `config`, `warning`. Once for each old name in the bootstrap config, such as `[node]` ([the bootstrap config of hopboxd](/reference/hopboxd-config/index.html.md#the-old-name-node)). |
| `starting` | info | `unit` |
| `started` | info | `unit`, `duration` |
| `start failed` | error | `unit`, `err` |
| `hopboxd is ready` | info | |
| `stopping hopboxd during the start` | info | `unit`, `reason` |
| `stopping hopboxd` | info | `reason`, for example `interrupt signal received` |
| `a unit failed, so hopboxd stops` | error | `err` |
| `stopped` | info | `unit`, `duration` |
| `stop failed` | error | `unit`, `err`, `duration` |
| `metrics listening` | info | `addr`, without `component` |
| `api listening` | info | `addr`, without `component` |
| `admin socket listening` | info | `path`, without `component` |
| `internal listener listening` | info | `addr`, without `component` |
| `the internal listener has no certificate yet` | warn | `err`, without `component`. The install has no key-encryption key yet, so it has no CA. |
| `swept expired idempotency keys` | info | `component=leader`, `count` |
| `sweep of idempotency keys failed` | warn | `component=leader`, `err` |
| `swept ended sessions and sign-in links` | info | `component=leader`, `count`: the sessions, links, CLI tokens, CLI sign-ins, device sign-ins, failed device approvals, and expired keys |
| `sweep of sign-ins failed` | warn | `component=leader`, `err` |
| `registered the bootstrap admin API key` | info | `component=control`, `id`, `expire_time` |
| `registered the default image of the release` | info | `component=control`, `name`, `digest`, `id` |
| `the install has the default image of the release already` | info | `component=control`, `digest` |
| `the install has the default image of a newer release, so the control role registers none` | info | `component=control`, `digest`, `newer_release` |
| `the release has no default image, so the control role registers none` | info | `component=control`, `path` |
| `cannot read the default image of the release, so the control role registers none` | error | `component=control`, `path`, `err` |
| `the default image of the release is not valid, so the control role registers none` | error | `component=control`, `path`, `err` |
| `the install has an admin API key already, so the bootstrap admin key file is removed unused` | warn | `component=control`, `path` |
| `the bootstrap admin key file is refused, so no admin API key is registered` | error | `component=control`, `path`, `err`. The file stays. |
| `swept expired enrollment tokens and server certificates` | info | `component=leader`, `tokens`, `certificates` |
| `sweep of enrollment tokens and server certificates failed` | warn | `component=leader`, `err` |
| `swept ended invitations` | info | `component=leader`, `count` |
| `sweep of invitations failed` | warn | `component=leader`, `err` |
| `freed reserved machine names and ports` | info | `component=leader`, `names`, `ports`, `machines`, `subnets` (the counts) |
| `sweep of machine names and ports failed` | warn | `component=leader`, `err` |
| `pruned old change events` | info | `component=leader`, `count` |
| `swept the rows of control processes that ended without their stop` | info | `component=leader`, `count` |
| `sweep of control processes failed` | warn | `component=leader`, `err` |
| `prune of change events failed` | warn | `component=leader`, `err` |
| `deleting idle machines failed` | warn | `component=leader`, `err`. The auto-delete run stopped, for example after the loss of the leader lease. |
| `giving back SSH host keys failed` | warn | `component=leader`, `err`. The return of SSH host keys stopped. The next run takes the rest. |
| `marking silent servers unresponsive failed` | warn | `component=leader`, `err` |
| `maintenance mode paused the leader jobs` | info | `component=leader`. Maintenance mode turned on ([Leader jobs](#leader-jobs)). |
| `this process stays fenced, so it gives up the leader lease to a process that can run the leader jobs` | warn | `component=leader`, `after`. The leader stayed fenced for one lease TTL ([Leader jobs](#leader-jobs)). |
| `the install is in maintenance mode: the API takes only reads, and the control role serves no control channel. Turn it off with hopboxd admin maintenance off` | warn | At the start of the `store` unit, when the control store is in maintenance mode, such as a copy of `admin db export`. |
| `renewing the row of this control process failed` | warn | `component=control`, `err`. Once, until a renewal works. The row stays live for 20 s after its last renewal. |
| `renewed the row of this control process again` | info | `component=control`. After a failed renewal. |
| `this control process could not renew its row for too long, so it fences itself: it takes no call, serves no control channel, and runs no leader job until a renewal works` | warn | `component=control`, `since`. No renewal worked for 15 s. |
| `this control process renewed its row again, so it lifts its fence` | info | `component=control`. The first renewal that works after the fence. |
| `marking the row of this control process stopped failed. It stops being live within 20 s` | warn | `component=control`, `err`. At the stop of the `store` unit. The row stays until the sweep of the leader. |
| `cannot read the keyring for the row of this control process` | warn | `component=control`, `err`. Once. The row then lists no key file. |
| `the install promoted a key-encryption key that this control server has no file of. Copy its key file from another control server` | error | `component=control`, `key_id`, `dir`. Once for each missing key, from the start of the `store` unit and from each renewal. |
| `a staged key file of this control server holds another key than the install recorded for its ID, so its row leaves it out. Copy the staged file of the server that staged it here again` | error | `component=control` from a renewal, once for each file until it changes, or `component=service` from the start of the `store` unit, with `key_id` and `dir` ([Sealing](/reference/sealing/index.html.md#key-fingerprints)). |
| `cannot apply the promoted key-encryption key` | warn | `component=control`, `err`. The start or a renewal could not promote the staged file, or could not read the promoted key from the store. |
| `reading maintenance mode after its change failed` | warn | `err`. The control process keeps its old maintenance mode until the next change event. |


The service layer writes these records about servers, with `component=service`:

| Message | Level | Fields |
| --- | --- | --- |
| `created the install CA` | info | `ca_hash` |
| `enrolled a server` | info | `server`, `pool`, `hostname`, `roles` |
| `recorded the hardware of a pool` | info | `pool`, `hardware` (the CPU model, the CPU threads, the RAM, and the disks, as text). The first server of a pool enrolled ([Enrollment and the install PKI](/reference/enrollment/index.html.md#hardware-of-a-pool)). |
| `added roles to a server` | info | `server`, `added` (the new roles), `roles` (every role) ([Enrollment and the install PKI](/reference/enrollment/index.html.md#add-roles-to-an-enrolled-server)) |
| `renewed a server certificate` | info | `server`, `serial` |
| `retired other server certificates` | info | `server`, `serial`, `retired` (a count) |
| `sent a pending server certificate again` | info | `server`, `serial` |
| `a retired server certificate called. Another party can hold a copy of it` | error | `server`, `serial`, `peer`, `suppressed`. At most once an hour for each certificate. In maintenance mode, each call logs it without `suppressed`, and writes no audit entry. |
| `the alarm of a retired certificate failed` | error | `server`, `serial`, `err`. The control store did not record the alarm of a retired certificate. The call then writes no alarm record and no audit entry. |
| `the audit entry of a retired certificate failed` | error | `err` |
| `a server is unresponsive` | warn | `server`, `after` (a duration) |

The service layer writes these records about key-encryption keys, with `component=service`. The calls `RotateKEK` and `RewrapSealedValues` write theirs too ([the admin API](/reference/admin-api/index.html.md#rotatekek), [the admin API](/reference/admin-api/index.html.md#rewrapsealedvalues)).

| Message | Level | Fields |
| --- | --- | --- |
| `staged a key-encryption key` | info | `key_id`, `caller` |
| `staged a key-encryption key, but the audit entry failed` | error | `key_id`, `err`. The staged file exists. |
| `promoted a key-encryption key` | info | `key_id`, `caller`. After the promote of the file on this server. |
| `cannot remove a staged key-encryption key that the install refused` | error | `component=service`, `key_id`, `err`. A stage made a key whose ID the install had with another fingerprint, and the removal of its staged file failed. Remove `<data_dir>/kek/<n>.staged` by hand. |
| `made the promoted key-encryption key current` | info | `key_id`. This control process promoted its staged copy of the key that an `admin kek promote` named, at its start, at a change event, or at a renewal. |

`install` writes these records with `component=install`, and `upgrade` with `component=upgrade`. The `initdb` of a first install writes the records of Postgres ([the bundled Postgres](/reference/postgres/index.html.md#logs)).

| Message | Level | Fields |
| --- | --- | --- |
| `created the user` | info | `user` |
| `reserved the IDs of the jailer` | info | `user`, `first`, `count` |
| `copied the release` | info | `version`, `dir` |
| `added the default image to the image cache` | info | `digest`, `dir` |
| `wrote the bootstrap config` | info | `path`, `roles` |
| `made the first key-encryption key` | info | `key_id` |
| `wrote the hash of the bootstrap admin API key` | info | `path` |
| `reserved the SSH ports` | info | `ports` |
| `switched the current release` | info | `version`, `previous` |
| `starting hopboxd` | info | `verb` (`start` or `restart`), `version` |
| `removed an old release` | info | `version` |
| `the server has a control store already, so the install ignores --bootstrap-admin-key-hash` | warn | |
| `unpacked the release` | info | `component=upgrade`, `version`, `dir` |
| `the release is in place already` | info | `component=upgrade`, `version`, `dir` |

The service layer writes these records about machines, with `component=service`:

| Message | Level | Fields |
| --- | --- | --- |
| `no node has room for a machine` | warn | `org`, `err`, `request_id` ([Placement](/reference/placement/index.html.md#logs)) |
| `deleted an idle machine` | info | `machine` |
| `deleting an idle machine failed` | warn | `machine`, `err` |

The parts of the control role write their own records:

- Postgres ([the bundled Postgres](/reference/postgres/index.html.md#logs)) and the migrations ([the bundled Postgres](/reference/postgres/index.html.md#schema-migrations))
- the leader election ([Control store tables](/reference/control-store/index.html.md#leader-election))
- the API ([the API server](/reference/api/index.html.md#logs)) and the admin socket ([the admin API](/reference/admin-api/index.html.md#logs-of-the-socket))
- the control channels ([the control channel](/reference/control-channel/index.html.md))
- the service layer ([the admin API](/reference/admin-api/index.html.md#rotatekek), [the admin API](/reference/admin-api/index.html.md#rewrapsealedvalues), [the sign-in, account, and audit API](/reference/auth-api/index.html.md#requestmagiclink))

The `channel` unit writes the records of its side of the control channel with `component=node` ([the control channel](/reference/control-channel/index.html.md)). It writes the records of the mesh device with `component=mesh` ([the mesh](/reference/mesh/index.html.md#logs)). The DNS role writes its records with `component=dns` ([the DNS role](/reference/dns/index.html.md#logs)). The edge role writes its records with `component=edge` ([the edge role](/reference/edge/index.html.md)).

## Exit status

| Status | Meaning |
| --- | --- |
| 0 | The command succeeded. For `run`, the roles stopped after `SIGINT` or `SIGTERM`, and every unit stopped well. |
| 1 | The command failed. For `run`, the bootstrap config is not valid, or a unit failed to start, to run, or to stop. For `enroll`, the preflight check or the enrollment failed. For `preflight`, a check of severity `fail` failed. For `install` and `upgrade`, a check refused, a step failed, or `hopboxd` did not start. For `stop-guest`, the guest did not shut down in time. `hopboxd` prints `hopboxd: error: <message>` to standard error. |
| 80 | The command line is not valid, for example an unknown command or flag, or a bad flag value. |
