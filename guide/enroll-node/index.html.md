# Enroll the node role on one server

> How to enroll the node role of a server with the control role of the same server, and check its heartbeats.

This guide gives one server both the control role and the node role. The node role enrolls with the control role of the same `hopboxd`. Then it keeps its control channel to the control role, over mutual TLS on loopback. At the end, `hopboxd admin server list` shows the server as `ready`, with the facts of its heartbeats.

[Enrollment and the install PKI](/reference/enrollment/index.html.md) covers the enrollment in full, and [the control channel](/reference/control-channel/index.html.md) covers the channel.

## What you need

- A Linux server with KVM. hopbox tests Debian 13 with the kernel `6.18.57-hopbox1` ([Host kernel](/reference/host-kernel/index.html.md)).
- A data dir on a file system with reflinks, such as XFS made with `reflink=1`. The default data dir is `/var/lib/hopbox`.
- `hopboxd` and the bundled Postgres on the server.
- Root on the server. Every command below runs with `sudo`.

A Mac cannot run the node role, because it has no KVM. The preflight check of `hopboxd enroll` refuses it:

```text
$ ./bin/darwin_arm64/hopboxd enroll --config /tmp/hb/hopboxd.toml --token "$TOKEN"
preflight kvm	fail	hopbox runs machines only on Linux with KVM, and this host runs darwin
preflight cgroup2	fail	hopbox runs machines only on Linux with KVM, and this host runs darwin
preflight tun	fail	hopbox runs machines only on Linux with KVM, and this host runs darwin
preflight nftables	fail	hopbox runs machines only on Linux with KVM, and this host runs darwin
preflight reflink	fail	hopbox runs machines only on Linux with KVM, and this host runs darwin
preflight kernel	warn	this host runs darwin with the kernel 27.0.0, and hopbox tests only Debian 13 with the kernel 6.18.57-hopbox1. hopbox runs, with a higher risk
hopboxd: error: the preflight check failed, so this server cannot run machines:
  kvm: hopbox runs machines only on Linux with KVM, and this host runs darwin
  cgroup2: hopbox runs machines only on Linux with KVM, and this host runs darwin
  tun: hopbox runs machines only on Linux with KVM, and this host runs darwin
  nftables: hopbox runs machines only on Linux with KVM, and this host runs darwin
  reflink: hopbox runs machines only on Linux with KVM, and this host runs darwin
```

That refusal is correct. Use a Linux server for this guide.

## Check the server

Run the preflight check first. It tests KVM, cgroup v2, tun, nftables, and reflinks on the data dir ([the preflight check](/reference/preflight/index.html.md)):

```text
$ sudo hopboxd preflight
preflight kvm	ok
preflight cgroup2	ok
preflight tun	ok
preflight nftables	ok
preflight reflink	ok
preflight kernel	ok
```

A line with `fail` names a problem that you must fix before the enrollment. A line with `warn` only warns.

## Write the bootstrap config

Give the server the control role and the node role. Every other key keeps its default ([the bootstrap config of hopboxd](/reference/hopboxd-config/index.html.md#keys)):

```sh
sudo mkdir -p /etc/hopbox
sudo tee /etc/hopbox/hopboxd.toml >/dev/null <<'EOF'
roles = ["control", "node"]

[channel]
labels = { zone = "gra" }
EOF
```

With the defaults, the internal listener of the control role listens on `127.0.0.1:7443`, and the node dials `https://127.0.0.1:7443`. The labels are optional. Each heartbeat reports them.

## Start hopboxd

```sh
sudo hopboxd run
```

`hopboxd` starts the units of the control role, then the `channel` unit, which keeps the control channel of the server. The server has no identity yet, so the unit waits, and logs this record once. Its records have `component=node`:

```json
{"time":"2026-10-16T09:00:01.000000000Z","level":"INFO","msg":"waiting for the enrollment of this server","component":"node","dir":"/var/lib/hopbox/server"}
```

On a fresh install, the internal listener also warns `the internal listener has no certificate yet`, because the install CA needs a key-encryption key. Leave `hopboxd` running, and use a second terminal for the next steps.

## Make the first key-encryption key

`hopboxd install` makes the first key-encryption key, so a server that it set up can skip this step ([Install hopbox on a Debian 13 server by hand](/guide/install-debian/index.html.md)). So can any install with a key-encryption key already.

```text
$ sudo hopboxd admin kek rotate
file:1
```

The internal listener now makes the install CA at its next handshake, without a restart.

## Make an enrollment token

```sh
TOKEN="$(sudo hopboxd admin enrollment-token create)"
```

The token works once, for 1 hour, in the pool `default`. It is a secret, so keep it in a variable and out of your shell history.

All servers of a pool have identical hardware. The first server of a pool records its CPU model, CPU threads, RAM, and disks, and the enrollment of a server with other hardware fails ([Enrollment and the install PKI](/reference/enrollment/index.html.md#hardware-of-a-pool)). If the new server is another model, make the token for another pool ([Add pools and sizes](/guide/pools-and-sizes/index.html.md)).

## Enroll the server

```text
$ sudo HOPBOX_ENROLLMENT_TOKEN="$TOKEN" hopboxd enroll
preflight kvm	ok
preflight cgroup2	ok
preflight tun	ok
preflight nftables	ok
preflight reflink	ok
preflight kernel	ok
hopboxd: enrolled as the server 01a14403-6bd1-7522-9ab0-b3ab4517e0e4. hopboxd run connects it to the control plane.
01a14403-6bd1-7522-9ab0-b3ab4517e0e4
```

`enroll` made a key for the server. Before it sent the token, it checked the CA of the control role against the hash in the token. The identity of the server is now in `/var/lib/hopbox/server`.

Within 2 seconds, the waiting `channel` unit finds the identity and opens its channel. The log of `hopboxd run` shows:

```json
{"time":"2026-10-16T09:02:11.000000000Z","level":"INFO","msg":"control channel connected","component":"node","server":"01a14403-6bd1-7522-9ab0-b3ab4517e0e4","control":"https://127.0.0.1:7443","state":"SERVER_STATE_READY"}
```

## See the server

```text
$ sudo hopboxd admin server list
ID                                    POOL     STATE  STATUS  ROLES  HOSTNAME  VERSION  LAST HEARTBEAT        PEER ADDRESS
01a14403-6bd1-7522-9ab0-b3ab4517e0e4  default  ready  ok      node   server1   v0.1.0   2026-10-16T09:02:16Z  -
```

`get` prints every fact of the last heartbeat, one line for each field:

```text
$ sudo hopboxd admin server get 01a14403-6bd1-7522-9ab0-b3ab4517e0e4
id	01a14403-6bd1-7522-9ab0-b3ab4517e0e4
pool	default
hostname	server1
roles	node
state	ready
cordoned	false
status	ok
problems	
labels	zone=gra
hopboxd_version	v0.1.0
kernel_release	6.18.57-hopbox1
cpu	amd64 GenuineIntel Intel(R) Xeon(R) CPU E5-1650 v4 @ 3.60GHz
cpu_template	none
compatibility_class	cc-3f1c2a9d8e7b6a50
cpu_threads	12
memory_bytes	135061946368
disk_bytes	3999688294400
disk_free_bytes	3998170157056
allocations	0
actions	0
last_heartbeat_time	2026-10-16T09:02:21Z
enroll_time	2026-10-16T09:02:10Z
certificate_expire_time	2026-11-15T09:02:10Z
version	4
update_time	2026-10-16T09:02:21Z
```

The heartbeat arrives every 5 seconds, so `last_heartbeat_time` and `version` grow. A server with a failed check shows `status degraded`, and `problems` lists the checks.

## Set the peer address

The peer address of a server is the public IPv4 address at which the other servers of the install reach it ([Enrollment and the install PKI](/reference/enrollment/index.html.md#peer-addresses)). It is one public address on the uplink of the server, with no NAT. The server of this guide enrolled with a token without a peer address, so it has none yet. Set it before you add a second server. Replace `203.0.113.12` with the public IPv4 address of the server:

```text
$ sudo hopboxd admin server set-peer-address 01a14403-6bd1-7522-9ab0-b3ab4517e0e4 203.0.113.12 | grep peer_address
peer_address	203.0.113.12
```

From then on, a token without a peer address enrolls only from the control server itself: from loopback, or from the address of its internal listener. Make the token of each other server with its own peer address, such as `sudo hopboxd admin enrollment-token create --peer-address 203.0.113.13`. That server must then enroll from `203.0.113.13`.

The internal listener still listens on loopback, so no other server reaches it. Before you add a second server, move the listener to the peer address. In `/etc/hopbox/hopboxd.toml`, add these keys, and restart `hopboxd`:

```toml
[control]
internal_listen = "203.0.113.12:7443"
```

Under `[channel]`, add `control_urls = ["https://203.0.113.12:7443"]` too, so the node role dials the listener at its new address. A server that `hopboxd install` set up takes both changes and the restart from one command: `sudo hopboxd install --roles control,node --internal-listen 203.0.113.12:7443` ([the internal listener](/reference/internal-listener/index.html.md#move-the-first-server-to-its-peer-address)).

## Cordon and uncordon

A cordoned server keeps its machines and takes no new one:

```text
$ sudo hopboxd admin server cordon 01a14403-6bd1-7522-9ab0-b3ab4517e0e4 --json
{"id":"01a14403-6bd1-7522-9ab0-b3ab4517e0e4","pool":"default","hostname":"server1","roles":["node"],"state":"cordoned","cordoned":true,...}
$ sudo hopboxd admin server uncordon 01a14403-6bd1-7522-9ab0-b3ab4517e0e4
```

The output of `--json` here is cut short. The next heartbeat answer tells the node its new state. The audit log records both calls, with the actor `local_root`.

## Stop and start hopboxd

Press Ctrl-C in the first terminal. `hopboxd` stops the `channel` unit first, which closes the channel. Then it stops the units of the control role.

In an install with other control servers, the leader marks a server unresponsive after 20 seconds without a heartbeat. The state is then `unresponsive`, and the audit log has the entry `server.mark_unresponsive` with the actor `system`. On one server, the leader stops with the node, so no one marks the server while `hopboxd` is down.

Start `hopboxd` again with `sudo hopboxd run`. The node reads its identity, opens its channel, and sends a heartbeat at once. The first heartbeat makes an unresponsive server `ready` again. A server that you cordoned becomes `cordoned` again. There is no need to enroll again: the identity stays in `/var/lib/hopbox/server`, and the node renews its certificate over the channel every 20 days.

## Start again from nothing

To enroll the server again, stop `hopboxd`, remove `/var/lib/hopbox/server`, and repeat the steps from [Make an enrollment token](#make-an-enrollment-token). The server gets a new ID. The old ID stays in the registry as an unresponsive server.
