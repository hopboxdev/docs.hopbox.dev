# The SSH front door

> The SSH front door of the edge role: its listener, the redirect of the SSH ports, the keys, and the sign-in. Also the connection to a machine, the channels, the checks, the limits, AuthorizeSSH, metrics, and logs.

The SSH front door is the SSH endpoint of the edge role. A client reaches a machine at the SSH port of that machine on an edge server, with an SSH key of its account. The user name is the Linux user in the machine. The front door checks the key with the control role. Then it opens its own SSH connection to the OpenSSH server in the machine, and it forwards the channels of both sides. It never runs a shell on the edge server itself.

This page covers the listener, the redirect of the SSH ports, the keys, and the sign-in. It also covers the connection to the machine, the channels and the requests that pass, and the checks of an open connection. Then it covers the limits, `EdgeService.AuthorizeSSH`, the audit entries, the metrics, and the logs.

The front door is in `internal/edge/sshfd`. The keys, the check of a key, and the host keys of the guests are in `internal/service/sshfrontdoor.go`. The control role pushes the keys in `internal/control/channel/edge.go`.

In the examples, the domain of the install is `example.com`, and the machine `box1` has the SSH port 10022.

## Listener

| Item | Value |
| --- | --- |
| Address | `edge.ssh_listen` of the bootstrap config. Default: `:2222`, port 2222 on every address ([the bootstrap config of hopboxd](/reference/hopboxd-config/index.html.md#keys)). The port must be outside 10000 to 30000. With the redirect, the listener takes every address, because the redirect sends each connection to the address of the interface that took it. |
| Redirect | `edge.ssh_redirect`, true by default. `hopboxd` then sends every TCP connection to a port from 10000 to 30000 of the server to the listener ([Redirect of the SSH ports](#redirect-of-the-ssh-ports)). |
| Protocol | SSH 2. The version string is `SSH-2.0-hopbox`. The key exchanges, the ciphers, and the MACs are the defaults of `golang.org/x/crypto/ssh`. |
| Open connections | At most 16,384. A connection past the bound waits in the backlog of the kernel. |
| Open connections of one client | At most 64. A client is an IPv4 address, or the /64 of an IPv6 address. The front door closes a connection past this bound at once. |
| Handshake timeout | 30 s from the TCP connection to the end of the sign-in. |
| Sign-in tries of one connection | At most 6, as `MaxAuthTries` of OpenSSH. |

The unit `edge` of `hopboxd run` opens the listener with the HTTPS and HTTP listeners ([the edge role](/reference/edge/index.html.md#listeners)). If another program holds the port, the start fails, and `hopboxd` exits with status 1. The error names the address, such as `sshfd: listen on :2222: bind: address already in use`.

At a stop, the unit ends every SSH connection first. An SSH connection cannot move to another process, so the client sees the connection close. Then the unit removes the redirect.

## Redirect of the SSH ports

Each machine has its own SSH port on the edge servers, from 10000 to 30000 ([Domain rules](/reference/domain-rules/index.html.md#ssh-ports)). The port names the machine before the SSH handshake. With `edge.ssh_redirect`, `hopboxd` replaces the nftables table `inet hopbox_edge` at the start of the edge role with this table:

```
table inet hopbox_edge {
  chain prerouting {
    type nat hook prerouting priority dstnat;
    fib daddr type local tcp dport 10000-30000 redirect to :2222
  }
  chain output {
    type nat hook output priority dstnat;
    fib daddr type local tcp dport 10000-30000 redirect to :2222
  }
}
```

The chain `prerouting` takes the connections of other hosts. The chain `output` takes the connections of the server to its own addresses, such as `ssh -p 10022 127.0.0.1` on the server. In both chains, the match `fib daddr type local` takes only a connection to an address of the server itself. So the traffic that the server forwards keeps its port, such as the egress of machines and of containers. The outgoing connections of the server to other hosts keep their ports too. One netlink batch replaces the table, so no connection sees a part of it. A stop of the edge role deletes the table first, before it ends the connections.

The front door reads the port that the client dialed from the connection tracking of the kernel (`SO_ORIGINAL_DST`, or `IP6T_SO_ORIGINAL_DST` for IPv6). A connection that no rule moved, such as one to port 2222 itself, has the port of the listener. The front door closes a connection whose port is outside 10000 to 30000 at once. When conntrack records the redirect, the host firewall takes the connection on the port of the listener ([the host firewall](/reference/host-firewall/index.html.md#ports)). So a connection from outside the server to port 2222 itself gets no answer.

The redirect needs Linux, the kernel module `nf_tables`, and `CAP_NET_ADMIN`. `hopboxd.service` runs `hopboxd` as root ([Server layout](/reference/server-layout/index.html.md#hopboxdservice)). On a system without nftables, such as macOS, the edge role logs a warning and serves only its own port. A connection to that port is outside the SSH ports, so it reaches no machine there. With `edge.ssh_redirect = false`, the operator makes the redirect. A connection then reaches the front door with the port that the client dialed, as the kernel tracks it.

A flush of the nftables ruleset also drops the table `inet hopbox_edge`, and with it every SSH port. So every 2 s, the front door checks its table: the table, its chains `prerouting` and `output`, and one rule in each. If a part is gone, it installs the table again, and logs `the nftables table inet hopbox_edge lost a part outside hopboxd, so the SSH front door installed it again. Do not flush the nftables ruleset of an edge server` at level error. Each nftables request ends after 10 s, so a kernel answer that never comes does not block the front door. A check that gets no answer within 10 s counts as a broken table, so the front door installs the table again and logs the same error. A stop of the edge role ends the checks before it removes the table.

`hopboxd install` keeps the SSH ports out of the ports of outgoing connections on a server with the node role ([Server layout](/reference/server-layout/index.html.md#reserved-ports)). The redirect matches only new connections, so the answers to an outgoing connection of the edge server never reach the front door.

## Keys

The install has two Ed25519 keys for the front door. Both are the same on every edge server and never change.

| Key | Use |
| --- | --- |
| Host key | The front door shows it to every client on every SSH port. `GetInstallInfo` returns its public key in `ssh_host_key` ([the API server](/reference/api/index.html.md#installservice)). The hopbox CLI writes it to `~/.ssh/known_hosts` under the name `hopbox-<domain>`, and the managed SSH block sets `HostKeyAlias hopbox-<domain>` ([hopbox CLI](/reference/cli/index.html.md#ssh-files)). So the first connection never asks the user about the host key. |
| Client key | The front door signs in to the OpenSSH server of each machine with it. Each machine spec carries its public key in `authorized_keys`. The guest agent writes it to `/etc/hopbox/authorized_keys`, and sshd reads that file for every Linux user ([hopbox-guest](/reference/hopbox-guest/index.html.md#files)). |

The first control process that starts with a key-encryption key makes both keys. It seals the private keys with the places `InstallSecret{Name: "ssh_host.key"}` and `InstallSecret{Name: "ssh_client.key"}`, and stores them in the table `ssh_front_door_keys` ([Control store tables](/reference/control-store/index.html.md#ssh_front_door_keys)). The rewrap of `hopboxd admin kek rewrap` moves both ([Sealing](/reference/sealing/index.html.md)). Without a key-encryption key, the control role logs `the install has no keys of the SSH front door yet` at its start, and `ssh_host_key` stays empty.

The control role pushes both private keys to each server with the edge role after the first heartbeat of each control channel, in `EdgeSSHKeys` ([the control channel](/reference/control-channel/index.html.md#edge-pushes)). Before the first key-encryption key, each heartbeat tries again, so the keys reach the edge soon after the first `hopboxd admin kek rotate`. The edge role keeps them only in memory. Before they arrive, the front door closes each connection before the handshake.

## Sign-in

The client signs in with a public key. Every other method fails.

1. The client offers a public key without a signature. The front door says yes to every key, and asks the control role nothing. So the public part of a key tells nobody whether it opens a machine.
2. The client signs with the key. The front door checks the signature.
3. The front door refuses a certificate, and a user name that is not a Linux user name ([User name](#user-name)).
4. The front door takes one try from each bucket: the bucket of the client, of its network, and of every client ([Limits](#limits)).
5. The front door refuses a key that holds 64 open connections already ([Limits](#limits)). So such a key never reaches the control role, and writes no audit entry.
6. The front door calls `EdgeService.AuthorizeSSH` with the SSH port, the public key, the user name, and the IP address of the client.
7. The front door checks that the routes of the edge name the same machine at the port. It takes one slot of the key. Another sign-in of the same key can take the last slot between steps 5 and 7, so the front door checks the bound here again.

Every refusal is the same for the client: `Permission denied (publickey)`. If the client has another key, it then tries that key.

The refusals take different paths in the control role. An unknown key needs one read of the control store. A known key also needs the read of the machine of the port. So the time of an answer can tell the holder of a valid key which ports hold a machine. It tells nothing about the machines of other accounts beyond that, and a key of nobody learns nothing.

### User name

The user name of the connection is the Linux user in the machine. It has 1 to 32 characters: lowercase letters, digits, `_`, and `-`. Its first character is a letter or `_`. Examples: `user`, `root`, `_apt`. The main user of the default image is `user` ([Default image](/reference/default-image/index.html.md)). The OpenSSH server of the machine decides whether the user exists and can sign in.

The control role refuses a user name that looks like a credential, such as `hbx_dev` or `ghp_` and 20 letters. The audit entry of a sign-in names the user, and the audit log refuses such a value ([Control store tables](/reference/control-store/index.html.md#record-an-entry)). The control role logs the reason `the Linux user name looks like a credential, which the audit log never stores`.

### Who opens a machine

The control role finds the account of the key by its fingerprint ([CLI tokens, API keys, admin API keys, and SSH keys](/reference/credentials/index.html.md#ssh-keys)), and the machine by the port. If all of these are true, the key opens the machine:

- The account uses the machine, as for an exec: it created the machine, or the machine is shared and the account is a member of its org. An org admin or an org owner cannot open a private machine of another member. The rule is the policy action `machine.ssh`, with the caller kind `ssh_key` ([the API server](/reference/api/index.html.md#callers-and-policy)).
- An instance admin did not suspend the org of the machine.
- The machine holds the port, and its target is not `deleted`.

Each allowed check records a use of the machine for its auto-delete timeout ([the machine API](/reference/machine-api/index.html.md#auto-delete)). Each allowed sign-in also writes an audit entry ([Audit](#audit)). SSH never creates, starts, or changes a machine.

## Connection to the machine

After the sign-in, the front door reads the route of the port ([the edge role](/reference/edge/index.html.md#machine-urls)). Then it opens an SSH connection to port 22 of the address of the machine.

| Item | Value |
| --- | --- |
| Address | The address of the route record of the machine, port 22. An edge server without the node role reaches it only over the mesh ([The mesh](#the-mesh)). |
| User | The user name of the client. |
| Key | The client key of the install. |
| Host key | Only a host key that the guest agent of the machine reported. The executor sends the host keys of the guest with each report of running ([the executor protocol](/reference/executor-protocol/index.html.md#machinereport)), and the route record keeps them. So the front door does not sign in to another honest guest at an old address. Every guest accepts the client key, so the host key is what tells two machines apart. A guest can show a host key that the route of another machine holds, such as after a copy of a disk. The guest that reports the key second then does not get that key in its route. If the guest has no other key, it loses SSH. An honest guest loses it too. When the first route lets the key go, because its guest changed its keys or its machine stopped, the second route gets the key in the same transaction. If another transaction locks the second route, or a `hopboxd` of v0.1.0 let the key go, a leader job gives the key within a minute ([Control store tables](/reference/control-store/index.html.md#route_host_keys)). A hostile guest can show a stolen host key. The guard against it is the hold of a released address: no new machine gets the address of a deleted machine for 5 minutes, much longer than a machine lease ([the node network](/reference/node-network/index.html.md#the-hold-of-a-released-address)). |
| Timeouts | 10 s for the TCP connection, and 10 s for the SSH handshake. |
| Retries | A machine reports `running` once its guest agent applied its config. OpenSSH can start listening about a second later. So when the machine refuses the TCP connection, the front door dials again. It waits 100 ms first, then twice as long each time, up to 1 s. It stops at the end of the 10 s of the TCP connection. Another failure, such as no route to the address, ends the dial at once. |

When the front door cannot reach the machine, the client stays signed in, and each channel that it opens fails with a message. The connection closes after the first refused channel, or after 30 s. With `ssh`, the message reads `channel 0: open failed: connect failed: hopbox: the machine box1 cannot take SSH connections now: <reason>`.

| Reason | When |
| --- | --- |
| `it is stopped. Start it with: hopbox machine start box1` | The machine does not run, or its lease ended at the edge. The state in the message is the state of the machine. |
| `it has no network address on its node. Use hopbox connect instead` | The route record has no address. Each report of running sets the address of the machine there ([the node network](/reference/node-network/index.html.md#where-the-address-shows)), so only a machine without an address gets it. |
| `its guest agent has not reported the SSH host keys of the machine yet. Retry in a few seconds` | The route has no host key. |
| `the front door cannot reach the OpenSSH server of the machine` | The TCP connection failed, or the machine refused it for 10 s. The front door logs `the front door cannot reach the OpenSSH server of a machine` at level info, with `machine`, `addr`, and `err`. |
| `the OpenSSH server of the machine refused the front door` | The SSH handshake failed: another host key, or no sign-in with the client key. |
| `the machine is gone` | The routes name no machine at the port any more. |

A machine in standby or hibernated does not wake at a connection before phase 3.

## The mesh

The front door reaches the OpenSSH server of a machine in the same way as the edge role reaches a machine URL ([the edge role](/reference/edge/index.html.md#the-way-to-the-machines)):

- On an edge server with the node role, the routes of the host decide: the tap device of a machine of this server, or the mesh device `hbm0` for a machine of another node.
- On an edge server without the node role, each connection to a machine is bound to `hbm0`, and leaves from the mesh address of the edge server. An address outside the machine range `10.240.0.0/12` fails at once.

So the OpenSSH server of the machine sees the mesh address of the edge server as the client. The audit entry `machine.ssh` keeps the address of the real client ([Audit](#audit)).

When the edge server has no mesh device yet, each channel fails with the reason `the front door cannot reach the OpenSSH server of the machine`, and the log record of that reason has an `err` such as `This edge server reaches the machines only over the mesh, and has no mesh device hbm0 yet`.

## Channels and requests

The front door forwards each channel and each request that the table names, both ways, with its answer. It refuses the others. When one side ends while the front door opens a channel on it, the front door refuses the channel with `the connection to the machine ended` or `the connection of the client ended`, and the connection ends. In `golang.org/x/crypto/ssh`, such an open can get no answer at all. It then waits until `hopboxd` restarts, and the gauge `hopbox_edge_ssh_lost_opens` counts it. A refused channel fails with `the SSH front door of hopbox does not forward <type> channels`. A refused request that wants an answer gets a failure.

| From | Channels | Requests |
| --- | --- | --- |
| The client | `session` (shell, exec, scp, SFTP, rsync), `direct-tcpip` (`ssh -L`, `ssh -D`, VS Code Remote-SSH) | Global: `tcpip-forward` and `cancel-tcpip-forward` (`ssh -R`), `keepalive@openssh.com`, `no-more-sessions@openssh.com`. On a session: `pty-req`, `env`, `shell`, `exec`, `subsystem`, `window-change`, `signal`, `break`, `auth-agent-req@openssh.com` (`ssh -A`), `keepalive@openssh.com`. |
| The machine | `forwarded-tcpip` (`ssh -R`), `auth-agent@openssh.com` (`ssh -A`) | Global: `keepalive@openssh.com`. On a session: `exit-status`, `exit-signal`, `eow@openssh.com`, `xon-xoff`, `keepalive@openssh.com`. |

So these stay off:

- X11 forwarding: the channel `x11` and the request `x11-req`.
- The forwarding of Unix sockets: the channels `direct-streamlocal@openssh.com` and `forwarded-streamlocal@openssh.com`, and `streamlocal-forward@openssh.com`.
- `hostkeys-00@openssh.com` of the OpenSSH server of the machine. The client knows only the host key of the install, so the host keys of a guest never reach its `known_hosts`.

The OpenSSH server of the machine applies its own `sshd_config` too, such as `AllowTcpForwarding` and `AcceptEnv`.

The front door sends the exit status of a session only after the last output of the session. It closes each end of a channel only after the answers to the requests of that end.

## Checks of an open connection

An open connection keeps its access only while the control role allows it. The front door calls `AuthorizeSSH` again:

- every `valid_for` of the last answer, 30 s today. The period is at least 5 s and at most 1 minute,
- at each access change that the control role pushes for the machine, or for every machine ([the edge role](/reference/edge/index.html.md#private-machine-urls)). For a change of every machine, such as a sign-out, each connection waits a random time of up to 10 s first. So the connections of an edge server do not all call the control role at once.

An answer of `permission_denied`, `unauthenticated`, `failed_precondition`, `not_found`, or `invalid_argument`, or an answer for another machine, ends the connection. Any other error, such as a control role that does not answer, keeps the connection until the next check. Each allowed check also records a use of the machine for its auto-delete timeout. These checks set `recheck`, so they write no audit entry.

Each change of a machine pushes an access change, such as a share, an unshare, or a delete. So do a removed SSH key, a member that leaves an org or that an admin removes, a deleted account, and a suspended org ([Control store tables](/reference/control-store/index.html.md#the-event-edgeaccess_changed)). A sign-out does too, but it does not end SSH connections, because an SSH key is not a sign-in.

New routes end an open connection at once in four cases. The port has no machine. Another machine holds the port. The org of the machine is suspended. The machine runs in another execution, with another lease.

## Limits

Each edge server counts its own limits. An install with more edge servers allows more in all.

Users behind one IPv4 address, or in one IPv6 /64, are one client, so they share its 30 tries a minute. A known key does not skip this bucket.

Other clients in a network can spend the bucket of that network with new keys. The new users of that /24 or /48 then wait for the bucket to fill again. The users that this edge server allowed in the last 15 minutes still sign in.

The bucket of every client is the last guard of the control role. Clients in many networks can still spend it, with new keys that open nothing. While it is spent, the users that this edge server allowed in the last 15 minutes still sign in. A user whose key is new to this edge server waits until the bucket fills again. That is the trade-off. A flood of new keys can keep new users out for a while. It never keeps out the users of the last 15 minutes, and it never floods the control role.

| Limit | Value | Past it |
| --- | --- | --- |
| Open connections of the listener | 16,384 | The connection waits in the backlog of the kernel. |
| Open connections of one client | 64 | The front door closes the connection at once. |
| Sign-in tries of one client | 30 a minute. A try is a key that the client signed with. The bucket fills again over a minute. | The try fails without a call to the control role. |
| Sign-in tries of one network | 300 a minute. A network is the /24 of an IPv4 address, or the /48 of an IPv6 address. So a client with many addresses of one network, such as the many /64 of one IPv6 /48, counts once. A key that this edge server allowed in the last 15 minutes skips this bucket. | The try fails without a call to the control role. |
| Sign-in tries of every client together | 60,000 a minute, in one bucket. A key that this edge server allowed in the last 15 minutes skips this bucket. | The try fails without a call to the control role. |
| Open connections of one SSH key | 64, over every client | The front door refuses the key before it asks the control role. |
| Sign-in tries of one connection | 6 | The connection ends. |
| Handshake | 30 s | The connection ends. |

## EdgeService.AuthorizeSSH

`hopbox.cluster.v1.EdgeService.AuthorizeSSH` serves the edge servers on the internal listener ([the edge role](/reference/edge/index.html.md#edgeservice)). It needs the certificate of a server with the edge role. Another caller gets `permission_denied`. The front door calls it with a timeout of 10 s.

| Request field | Type | Meaning |
| --- | --- | --- |
| `ssh_port` | uint32 | The port that the client dialed, 10000 to 30000. |
| `public_key` | bytes | The public key of the client in the SSH wire format, at most 16 KiB. |
| `linux_user` | string | The user name of the client. |
| `client_ip` | string | The IP address of the client, such as `192.0.2.10`. An IPv4-mapped IPv6 address counts as its IPv4 address. A sign-in needs it. The audit entry records it. |
| `recheck` | bool | `true` for a check of an open connection, which writes no audit entry. `false` for a sign-in. |

| Response field | Type | Meaning |
| --- | --- | --- |
| `machine_id` | string | The ID of the machine, a UUID. |
| `machine_name` | string | The name of the machine, for the messages to the client. |
| `account_id` | string | The account of the key, a UUID. |
| `valid_for` | duration | How long the connection runs before the next check: 30 s. |

Every refusal is `permission_denied` with the same message: `permission denied: the SSH key opens no machine at this port for this user`. The control role logs the reason ([Logs](#logs)). Other errors, such as `unavailable`, mean that the control role did not check the key.

| Error | Code | Message |
| --- | --- | --- |
| A sign-in without `client_ip` | `invalid_argument` | `invalid argument: a sign-in needs the IP address of the SSH client` |
| A `client_ip` that is not an IP address | `invalid_argument` | `invalid argument: the client IP is not an IP address` |

An edge server of an older release sends no `client_ip`, so the control role refuses each of its sign-ins. Until milestone N09 of phase 2 adds two releases side by side, the edge servers and the control role must run the same release.

## Audit

Each sign-in that `AuthorizeSSH` allows writes one audit entry `machine.ssh` in the transaction of the check ([Control store tables](/reference/control-store/index.html.md#audit_log)). The owner decided this on 2026-10-24.

| Field | Value |
| --- | --- |
| Org | The org of the machine. The org owners and the org admins read the entry with `ListAuditEntries` ([the sign-in, account, and audit API](/reference/auth-api/index.html.md#listauditentries)). An org member reads no entry of the org, as for every audit entry. Instance admins read every entry. |
| Actor | The kind `ssh_key`, the account of the key, and the ID of the SSH key as the credential ([CLI tokens, API keys, admin API keys, and SSH keys](/reference/credentials/index.html.md#ssh-keys)). |
| Target | The kind `machine`, and the machine ID. |
| `name` | The name of the machine. The entry goes without it when the audit log refuses the name, as for a name from before a new credential pattern ([Domain rules](/reference/domain-rules/index.html.md#machine-names)). The target ID still names the machine. |
| `linux_user` | The user name of the connection. |
| `fingerprint` | The SHA256 fingerprint of the key, such as `SHA256:FXuRc2JyZfKx3tXHxoblBpr+f1jfrmFaGfkF4S5o0Ho`. |
| `client_ip` | The IP address of the client, as the edge server saw it. |

Each sign-in that the control role allows writes one entry. The checks of an open connection write none. A refusal writes none, and the control role logs it ([Logs](#logs)).

The entry records each sign-in that the control role allowed, also one that the edge server refuses after it. The edge server refuses the key in two cases: its routes name another machine at the port, or another sign-in of the key took its last slot in the meantime ([Sign-in](#sign-in)). The client then tries its next key, and a key that opens the machine writes a second entry. A key that holds 64 open connections before the call never reaches the control role, so it writes no entry. A machine that cannot take SSH now gets an entry too, because the client signed in ([Connection to the machine](#connection-to-the-machine)).

An entry of the API looks like this:

```json
{"id":"01a173a5-aa40-7cd0-82d6-384d41fcc7fe", "orgId":"01a173a5-aa01-7045-a1cb-415460b1ee81", "actor":{"kind":"ssh_key", "accountId":"01a173a5-aa08-7da5-a57a-88b0a1766cdc", "credentialId":"01a173a5-aa3b-7649-8377-8034b2728136"}, "action":"machine.ssh", "targetKind":"machine", "targetId":"01a173a5-aa2d-7824-a289-9a0fa1342bdb", "details":{"client_ip":"192.0.2.10", "fingerprint":"SHA256:FXuRc2JyZfKx3tXHxoblBpr+f1jfrmFaGfkF4S5o0Ho", "linux_user":"user", "name":"box1"}, "requestId":"01a173a5-aa3f-7d61-9b0e-36c1f2a7d4e5", "createTime":"2026-10-25T15:19:59.295802Z"}
```

The `requestId` is the ID of the call of `AuthorizeSSH` on the internal listener.

## Metrics

The edge role serves these metrics on the metrics listener of `hopboxd` ([Metrics](/reference/metrics/index.html.md#metrics-of-the-edge-role)).

| Metric | Type | Labels | Meaning |
| --- | --- | --- | --- |
| `hopbox_edge_ssh_connections_total` | counter | `result` | The TCP connections of the front door, by result. See the table of results below. |
| `hopbox_edge_ssh_auth_total` | counter | `result` | The keys that clients signed with: `allowed`, `refused`, `limited`, or `error` (the control role did not answer). |
| `hopbox_edge_ssh_unreachable_total` | counter | `reason` | The signed-in connections that reached no OpenSSH server: `not_running`, `no_address`, `no_host_key`, `dial`, or `handshake`. |
| `hopbox_edge_ssh_channels_total` | counter | `type`, `result` | The channels, by type, and by result: `forwarded`, `refused` (by the other side, or because its connection ended during the open), or `prohibited` (by the front door). An unknown type counts as `other`. |
| `hopbox_edge_ssh_ended_total` | counter | `reason` | The open connections that the front door ended: `access_lost`, `route_changed`, or `shutdown`. |
| `hopbox_edge_ssh_open_connections` | gauge | none | The signed-in connections that the front door holds now. |
| `hopbox_edge_ssh_lost_opens` | gauge | none | The channel opens that got no answer and still wait, after their connection ended. Each one holds a little memory until `hopboxd` restarts. |

| `result` of `hopbox_edge_ssh_connections_total` | Meaning |
| --- | --- |
| `forwarded` | The connection reached its machine. |
| `unreachable` | The client signed in, but the machine cannot take SSH now. |
| `refused` | No key of the client opened the machine. |
| `limited` | The client held 64 open connections. |
| `no_machine` | The port is outside the SSH ports. |
| `no_keys` | The keys of the front door have not arrived yet. |
| `handshake_failed` | The SSH handshake did not finish. |

## Logs

The front door logs with `component=sshfd`.

| Message | Level | When |
| --- | --- | --- |
| `SSH front door listening` | info | The listener is open, with `addr`. |
| `the SSH ports reach the front door` | info | The redirect is in place, with `ports` and `listen_port`. |
| `this system has no nftables, so no SSH port reaches the front door. Only Linux runs the redirect` | warn | `edge.ssh_redirect` on a system without nftables, with `os`. |
| `the nftables table inet hopbox_edge lost a part outside hopboxd, so the SSH front door installed it again. ...` | error | The check of the redirect found its table broken, or got no answer within 10 s, and installed it again. |
| `cannot repair the redirect of the SSH ports` | error | `err`. The check of the redirect failed. It tries again 2 s later. |
| `serving the SSH host key of the install` | info | The first keys, with the `fingerprint` of the host key. |
| `closed a connection that reached no SSH port` | debug | A connection came to a port outside the SSH ports, with `client`, `port`, and `err`. |
| `closed an SSH connection: <error>` | warn | The keys of the front door have not arrived yet, with `client`. The error is `sshfd: the keys of the SSH front door have not arrived yet`. |
| `an SSH connection did not sign in` | debug | The SSH handshake with the client failed, or no key of the client opened the machine, with `client`, `port`, and `err`. |
| `SSH connection forwarded` | info | A connection reached its machine, with `client`, `machine`, `port`, `user`, and `fingerprint`. |
| `SSH connection closed` | info | A forwarded connection ended. |
| `a signed-in SSH connection cannot reach its machine` | info | With `reason` and `err` ([Connection to the machine](#connection-to-the-machine)). |
| `the front door cannot reach the OpenSSH server of a machine` | info | The TCP connection to the machine failed, with `machine`, `addr`, and `err`. A client that leaves during the dial gets no record. |
| `the SSH connection to a machine failed` | warn | The SSH handshake with the machine failed, with `addr`, `user`, and `err`. |
| `cannot check an SSH key with the control role` | warn | `AuthorizeSSH` failed without an answer. |
| `refused an SSH connection past the bound of open connections of its key` | info | With `fingerprint` and `max`. |
| `ended an SSH connection that lost its access` | info | A check refused the access of an open connection. |
| `checking an SSH connection failed. The next check tries again` | warn | A check got no answer. |
| `ended the SSH connections of machines whose route changed` | info | New routes ended open connections, with `connections`. |
| `cannot remove the redirect of the SSH ports` | warn | The stop did not delete the table. |

The control role logs each refusal of `AuthorizeSSH` as `refused an SSH connection` at info level. Its fields are `ssh_port`, `linux_user`, `client_ip`, `fingerprint`, and `reason`, such as `the account does not use the machine`. A refusal writes no audit entry, so this record is the only trace of it. It logs each access that it allows as `allowed an SSH connection` at debug level, with `ssh_port`, `linux_user`, `client_ip`, `fingerprint`, `machine`, and `account`. It logs `created the keys of the SSH front door` once, with `host_key`.

## Not in phase 1

- A machine in standby or hibernated wakes at the first packet to its SSH port (phase 3).
