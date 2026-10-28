# The bootstrap config of hopboxd

> Every key and default of the bootstrap config /etc/hopbox/hopboxd.toml, and its errors.

The bootstrap config holds the few settings that `hopboxd` needs before it can reach the control store: its roles, its dirs, its listeners, and the address of the control role. `hopboxd run` reads it ([hopboxd](/reference/hopboxd/index.html.md#run)). The settings of the whole install, such as its domain, live in the control store and change through the admin API ([the admin API](/reference/admin-api/index.html.md)). The code is in `internal/config`.

## File

| Item | Value |
| --- | --- |
| Path | `/etc/hopbox/hopboxd.toml`. `hopboxd run --config` gives another path. |
| Format | [TOML](https://toml.io) |
| Encoding | UTF-8 |

`hopboxd install` writes the file ([hopboxd](/reference/hopboxd/index.html.md#install)). It sets `roles` from `--roles`, and `channel.control_urls` (or the old key `channel.control_url`) and `channel.labels` from `--control-url` and `--label`. With one `--control-url`, it writes the old key `channel.control_url`, unless the file has `channel.control_urls` already. So a rollback to a `hopboxd` from before `control_urls`, which refuses that key, still reads the file. With more than one `--control-url`, it writes `channel.control_urls` and deletes `channel.control_url`. It keeps every other key of a file that exists, and writes a file with the mode `0644`. Before it reads a file that exists, it makes the owner and mode checks below. So root never takes values from a file that another user can change.

`hopboxd run` reads the file once, at start. A change takes effect at the next start. An install that changes the file restarts `hopboxd`. `hopboxd admin` reads it to find the admin socket ([hopboxd](/reference/hopboxd/index.html.md#admin)). `hopboxd enroll` and `hopboxd preflight` read it for the roles, the data dir, and the control URLs ([hopboxd](/reference/hopboxd/index.html.md#enroll)).

The file names the Postgres binaries that `hopboxd` runs and the external Postgres that it trusts, so a `hopboxd` that runs as root refuses a file that another user can change:

- The file must be a regular file of at most 64 KiB.
- As root, the path must not be a symlink. `hopboxd` opens it with `O_NOFOLLOW`.
- As root, the file must belong to root.
- As root, group and others must not be able to write the file. Mode `0644` or `0600` is fine.
- As root, the dir of the file, such as `/etc/hopbox`, must be a real dir, not a symlink. It must belong to root, and group and others must not be able to write it. Otherwise another user can swap the file. `hopboxd` checks only this dir, not the dirs above it, such as `/etc`. The install owns `/etc/hopbox`, and the OS owns the dirs above it.

## Keys

| Key | Type | Default | Description |
| --- | --- | --- | --- |
| `roles` | array of strings | none, required | The roles of this server: `control`, `node`, `dns`, `edge`, or more than one. Each role appears at most once. The control role runs the control store, the API, the internal listener, and the admin socket. The node role runs machines. The DNS role answers DNS queries for the zones of the install ([the DNS role](/reference/dns/index.html.md)). The edge role serves HTTPS for the domain of the install and for every machine URL ([the edge role](/reference/edge/index.html.md)). It also runs the SSH front door of the machines ([the SSH front door](/reference/ssh-front-door/index.html.md)). A server with a role other than `control` enrolls with the control role. It keeps a control channel to it ([the control channel](/reference/control-channel/index.html.md)). |
| `data_dir` | absolute path | `/var/lib/hopbox` | The state of the server. The bundled Postgres keeps its cluster in `<data_dir>/pg` ([the bundled Postgres](/reference/postgres/index.html.md#paths-on-a-server)), the file keyring keeps its key-encryption keys in `<data_dir>/kek` ([Sealing](/reference/sealing/index.html.md#key-files)), and an enrolled server keeps its identity in `<data_dir>/server` ([Enrollment and the install PKI](/reference/enrollment/index.html.md)). The control role keeps the mail of the file mail transport in `<data_dir>/mail` ([Mail](/reference/mail/index.html.md#the-mail-folder)). The reflink check of the preflight check runs in this dir ([the preflight check](/reference/preflight/index.html.md)). The node role keeps its images, kernels, disks, and jails here ([the Firecracker driver](/reference/firecracker-driver/index.html.md#files-of-a-machine)). On a server with the node role, the path has at most 25 characters, because each jail holds unix sockets. `hopboxd` and `hopboxd install` refuse a longer path with `data_dir: "<path>" has <n> characters. With the node role, it has at most 25, because each jail in it holds unix sockets`. |
| `run_dir` | absolute path | `/run/hopbox` | The sockets of the server. The bundled Postgres has its socket in `<run_dir>/pg`, and the admin socket is `<run_dir>/admin.sock` ([the admin API](/reference/admin-api/index.html.md#the-admin-socket)). |
| `control.api_listen` | `host:port` | `127.0.0.1:7080` | The TCP address of the API listener of the control role ([the API server](/reference/api/index.html.md#listener)). The host must be a loopback IP address, such as `127.0.0.1` or `[::1]`. Port `0` picks a free port. |
| `control.internal_listen` | `host:port` | `127.0.0.1:7443` | The TCP address of the internal listener of the control role, which serves the cluster API over TLS to the servers of the install ([the internal listener](/reference/internal-listener/index.html.md)). The host must be an IP address, of any kind, such as `127.0.0.1` or `10.0.0.5`. `0.0.0.0` listens on every address. Port `0` picks a free port. It must differ from `control.api_listen`, except when both use port `0`. On an install with more than one server, use the peer address of the server and port 7443, because the other servers and control processes reach this listener at the address that it names. `hopboxd install --internal-listen` sets it ([the internal listener](/reference/internal-listener/index.html.md#move-the-first-server-to-its-peer-address)). A listener on every address gets no forwarded data streams ([the internal listener](/reference/internal-listener/index.html.md)). |
| `control.release_image` | absolute path | `/opt/hopbox/current/image/artifact.json` | The `artifact.json` of the default image of the release. At each start, the control role registers this image, unless the install has it already or has an image of the same name from a newer release ([hopboxd](/reference/hopboxd/index.html.md#the-default-image-of-the-release)). A missing file gives a log record at level info, and a file that is not valid gives one at level error. In both cases, the start goes on. |
| `channel.control_urls` | array of URLs | `["https://127.0.0.1:7443"]` | The URLs of the internal listeners of the control servers. List 1 to 8 URLs, each one once. A server with a role other than `control` enrolls at the first URL that answers ([Enrollment and the install PKI](/reference/enrollment/index.html.md)). It keeps one control channel, to one URL at a time. The first channel goes to the URL that a CRC-32 of the server ID picks, so the channels of a fleet spread over the control servers. After a lost or refused channel, the next dial goes to the next URL of the list ([the control channel](/reference/control-channel/index.html.md#the-agent)). Once the control role sends the URLs of the live control servers on the channel, the server keeps them in `<data_dir>/server/control-urls`, and dials them before the URLs of this key ([the control channel](/reference/control-channel/index.html.md#control-urls)). So this list needs to name only one control server that is up at the first start. The edge role also sends the requests of the API and the console to these URLs, in turn ([the edge role](/reference/edge/index.html.md#forwarded-requests)). Each URL must be an `https` URL of a host, with an optional port, and without user info, a path other than `/`, a query, or a fragment. Without a port, the port is 443. On a server that also has the control role, a URL names the internal listener of the same `hopboxd`. |
| `channel.control_url` | URL | none | The old key of `channel.control_urls` with one URL. `hopboxd` moves its value into `channel.control_urls` when it reads the file. A file with both keys fails ([Errors](#errors)). |
| `channel.labels` | table of strings | empty | The labels of the server, for example `zone = "gra"`. Each heartbeat reports them, and the registry keeps them ([the control channel](/reference/control-channel/index.html.md)), until an instance admin sets the labels through the admin API. From then on, the registry keeps the labels of the admin ([the admin API](/reference/admin-api/index.html.md#setserverlabels)). At most 64 labels. A key has 1 to 63 characters of `a-z`, `0-9`, `.`, `_`, and `-`, and starts with a lowercase letter. A value is valid UTF-8 of at most 255 bytes, without control characters. |
| `dns.listen` | array of `host:port` | `[":53"]` | The addresses of the DNS role ([the DNS role](/reference/dns/index.html.md)). It listens on UDP and on TCP at each address. List 1 to 16 addresses, each one once. The host is empty, which listens on every address, or an IP address, such as `203.0.113.10`. A host name is refused. The port is a number. Port `0` picks a free port, and UDP and TCP share that port. The DNS role opens its sockets at the start of `hopboxd run`, so a port that another program holds fails the start ([hopboxd](/reference/hopboxd/index.html.md#run)). On Linux, an address that the server does not hold yet, such as the address of `edge.public_ipv6`, passes, and takes queries once it exists ([the DNS role](/reference/dns/index.html.md#listeners)). |
| `edge.https_listen` | `host:port` | `:443` | The address of the HTTPS listener of the edge role ([the edge role](/reference/edge/index.html.md#tls)). The host is empty, which listens on every address, or an IP address, such as `203.0.113.10`. A host name is refused. Port `0` picks a free port. The edge role opens the listener at the start of `hopboxd run`, so a port that another program holds fails the start ([hopboxd](/reference/hopboxd/index.html.md#run)). |
| `edge.http_listen` | `host:port` | `:80` | The address of the plain HTTP listener of the edge role. It answers each request for a host of the install with a redirect to HTTPS ([the edge role](/reference/edge/index.html.md#routing)). The rules of `edge.https_listen` apply. The two addresses must differ. Only port `0` can appear in both. |
| `edge.ssh_listen` | `host:port` | `:2222` | The address of the listener of the SSH front door ([the SSH front door](/reference/ssh-front-door/index.html.md#listener)). The rules of `edge.https_listen` apply. Its port must be outside the SSH ports 10000 to 30000, and it must differ from the two HTTP listeners. Only port `0` can appear twice. With `edge.ssh_redirect`, the host must be empty or `::`. The redirect sends each connection to the address of the network interface that took it. So a listener on one address, such as a floating IP, gets no client, and `0.0.0.0` gets no IPv6 client. |
| `edge.ssh_redirect` | bool | `true` | Makes `hopboxd` replace the nftables table `inet hopbox_edge` at the start of the edge role. The table sends every TCP connection to a port from 10000 to 30000 of the server to `edge.ssh_listen` ([the SSH front door](/reference/ssh-front-door/index.html.md#redirect-of-the-ssh-ports)). A stop deletes it. On a system without nftables, such as macOS, the edge role logs a warning and serves only `edge.ssh_listen`, which reaches no machine. If the operator makes the redirect, set it to `false`. |
| `edge.public_ipv6` | IPv6 address | none | The floating IPv6 address of this edge server on its uplink, without a prefix length, such as `2001:db8::11` ([the edge role](/reference/edge/index.html.md#the-public-ipv6-address)). The zone names it in its `AAAA` records. Without it, the zone has no `AAAA` record. The network setup of the Terraform provider puts it on the uplink ([hopbox Terraform provider](/reference/terraform-provider/index.html.md#the-network-setup-of-an-edge-server)). `hopboxd` puts it on the uplink whenever no interface of the server holds it, also at its first start when the address was never there ([the edge role](/reference/edge/index.html.md#the-check-of-the-uplink)). The NAT66 of the machines never takes it ([the host firewall](/reference/host-firewall/index.html.md#the-uplink-and-the-nat66)). The address is a public IPv6 address: not a unique local address, and not a link-local address. Only a server with the role `edge` takes the key. A `hopboxd` from before N29 refuses an address without a prefix length, so add the key only after the upgrade of the server. |
| `edge.public_ipv6_gateway` | IPv6 address | none | The old key of the gateway of the IPv6 block of the vRack, which this release removed. `hopboxd` ignores it, and logs a warning ([The IPv6 block of an older release](#the-ipv6-block-of-an-older-release)). |
| `edge.public_ipv4` | IPv4 address | none | The floating IPv4 address of this edge server, such as `203.0.113.101`. The IPv4 egress of the machines leaves from it ([the egress of the machines](/reference/egress/index.html.md#the-source-addresses)). The Terraform provider writes the first address of `floating_addresses` in sorted order ([hopbox Terraform provider](/reference/terraform-provider/index.html.md#edge-servers-and-floating-addresses)). The Terraform provider puts it on the uplink, and `hopboxd` puts it on the uplink whenever no interface of the server holds it, also at its first start when the address was never there ([the edge role](/reference/edge/index.html.md#the-check-of-the-uplink)). Without it, the egress leaves from the address of the output interface, and `hopboxd` logs a warning at its start. The address is a public IPv4 address, outside the private ranges, the vRack range, and the machine range. Only a server with the role `edge` takes the key. `hopboxd install --edge-public-ipv4` sets it ([hopboxd](/reference/hopboxd/index.html.md#install)). A `hopboxd` before this key refuses the file, so add it only after the upgrade of the server. |
| `edge.floating_addresses` | array of IPv4 addresses | none | The other floating IPv4 addresses of this edge server beside `edge.public_ipv4`, such as `["203.0.113.102"]`. The Terraform provider puts them on the uplink, and `hopboxd` puts each one on the uplink whenever no interface of the server holds it, also at its first start when the address was never there ([the edge role](/reference/edge/index.html.md#the-check-of-the-uplink)). An address of `edge.public_ipv4` can be in the list too. The Terraform provider writes every floating address of the server, `edge.public_ipv4` included. `hopboxd install --edge-floating-address` sets the key, and `--edge-no-floating` removes it with `edge.public_ipv4` ([hopboxd](/reference/hopboxd/index.html.md#install)). List at most 64 addresses, each one once. Each address is a public IPv4 address, as for `edge.public_ipv4`. Only a server with the role `edge` takes the key. A `hopboxd` before this key refuses the file, so add it only after the upgrade of the server. The version of a release that takes the key names the feature `floating-addresses` ([the version command](/reference/version/index.html.md#json-output)). |
| `acme.name_servers` | array of `host:port` | empty | The addresses at which the ACME job of the control role checks that each name server serves a change of the zone ([the certificate of the install](/reference/acme/index.html.md#the-name-server-check)). Empty means port 53 at the addresses of `ns1` and `ns2` of the zone. At most 16. Each host is an IP address. A host name is refused. When the DNS role listens on another port, set it, such as `["127.0.0.1:15353"]` on a laptop. |
| `acme.ca_file` | absolute path | empty | A PEM file of more root certificates for the HTTPS of the ACME directory. The ACME job trusts them, for example the root of the test CA Pebble ([the certificate of the install](/reference/acme/index.html.md#test-with-pebble)). Empty means the roots of the system only. A file that does not load, or holds no certificate, fails the start of `hopboxd run`. |
| `postgres.install` | absolute path | `/opt/hopbox/current/postgres` | The bundled Postgres: a dir with `bin/`, `lib/`, and `share/` ([the bundled Postgres](/reference/postgres/index.html.md#the-install)). `hopboxd admin db export` runs `pg_dump` and `pg_restore` from it ([hopboxd](/reference/hopboxd/index.html.md#admin-db-export)). |
| `postgres.user` | string | `hopbox-pg` | The OS user that runs Postgres when `hopboxd` runs as root ([the bundled Postgres](/reference/postgres/index.html.md#os-users-and-database-roles)). |
| `postgres.url` | URL | empty | An external Postgres for the control store, such as managed PostgreSQL ([the bundled Postgres](/reference/postgres/index.html.md#external-postgres)). With it, the control role starts no bundled Postgres and connects over TLS. The form is `postgres://<user>@<host>[:<port>]/<database>?sslmode=verify-full`. The scheme can also be `postgresql`. It must name one host, a user, and a database, and have `sslmode=verify-full`. It must not hold a password, or the parameters `password`, `passfile`, `sslrootcert`, `sslcert`, `sslkey`, `sslpassword`, `sslcertmode`, `service`, `servicefile`, `host`, `hostaddr`, `port`, `dbname`, or `user`. Other parameters of libpq and pgx, such as `connect_timeout`, pass on. It needs `postgres.ca_file` and `postgres.password_file`. |
| `postgres.ca_file` | absolute path | empty | The PEM file of the CA that signed the certificate of the external Postgres. `hopboxd` checks the certificate against it, and the host name of `postgres.url` against the certificate. For managed PostgreSQL, it is the CA certificate that OVHcloud gives for the service. It needs `postgres.url`. As root, `hopboxd` makes the owner and mode checks of the bootstrap config on it. |
| `postgres.password_file` | absolute path | empty | A file that holds the password of the user of `postgres.url`, on one line. A newline at the end is fine. It needs `postgres.url`. As root, the file must belong to root, must not be a symlink, and group and others must not read or write it: mode `0400` or `0600`. Its dir has the checks of the dir of the bootstrap config. At most 4 KiB. |
| `metrics.listen` | `host:port` | `127.0.0.1:9464` | The TCP address of the Prometheus metrics ([Metrics](/reference/metrics/index.html.md)). The host must be a loopback IP address. Every role uses it. |
| `vmm.firecracker` | absolute path | `/opt/hopbox/current/firecracker/firecracker` | The Firecracker binary of the node role ([the Firecracker driver](/reference/firecracker-driver/index.html.md)). Its file name must be `firecracker`, because the jailer names the jail of each machine after it. Each boot uses the file at this path, so an upgrade moves new boots to the Firecracker of the new release. A running microVM keeps its own copy. The compatibility class takes its snapshot format version ([the control channel](/reference/control-channel/index.html.md#compatibility-class)). |
| `vmm.jailer` | absolute path | `/opt/hopbox/current/firecracker/jailer` | The jailer binary of the node role. The unit of each machine runs it ([the Firecracker driver](/reference/firecracker-driver/index.html.md#the-unit-of-a-machine)). |
| `vmm.kernel` | absolute path | `/opt/hopbox/current/kernel/vmlinux` | The guest kernel of the release. At each start, `hopboxd run` adds it to the kernel cache ([the Firecracker driver](/reference/firecracker-driver/index.html.md#the-kernel-cache)). A missing file gives a warning, and the start goes on. |
| `network.nameservers` | array of IPv4 addresses | empty | The resolvers of the machines of the node role, at most 3, each once ([the node network](/reference/node-network/index.html.md#the-resolvers-of-the-machines)). The guest agent writes them to `/etc/resolv.conf` of each machine, and the machines reach them on port 53. Empty takes the IPv4 resolvers of the host, from `/run/systemd/resolve/resolv.conf` or `/etc/resolv.conf`. A loopback, unspecified, link-local, or multicast address, or an address of the install range `10.240.0.0/12`, is refused. A resolver on the node itself does not work, because a machine reaches no service of its node. |

The API listener has no TLS. So `hopboxd` refuses a `control.api_listen` that is not a loopback address, and the API never leaves the server in clear text. A host name, even `localhost`, is refused, because a name can resolve to an address that is not loopback. The internal listener has TLS, and every call on it except `Enroll` needs a server certificate, so `control.internal_listen` can be any IP address. On an install with one server, it stays on `127.0.0.1`. With more servers, it listens on the peer address of the server. `hopboxd` refuses a host name there too.

`hopboxd` checks the keys of `[postgres]` on every server, but only the control role uses them. `hopboxd run` reads `postgres.ca_file` and `postgres.password_file` at its start, so a change of either file takes effect at the next start. `hopboxd` checks the keys of `[control]` only on a server with the control role. It checks the keys of `[channel]` only on a server that enrolls: one with a role other than `control`. It checks the keys of `[dns]` only on a server with the DNS role. It checks the keys of `[edge]` only on a server with the edge role. It checks the keys of `[acme]` only on a server with the control role. It checks the keys of `[vmm]` and `[network]` only on a server with the node role.

### The old name `[node]`

Before the DNS role, `[channel]` had the name `[node]`. This release still reads `[node]` as `[channel]`, and the next release refuses it. With `[node]`, `hopboxd run` logs this record at level warn, with the path of the file in `config`:

```json
{"level":"WARN","msg":"the bootstrap config uses an old name","config":"/etc/hopbox/hopboxd.toml","warning":"the section [node] of the bootstrap config is now [channel]. Rename it. hopboxd install renames it, and the next release of hopboxd refuses [node]"}
```

`hopboxd enroll` prints the same warning on stderr, after `hopboxd: warning: `. When `hopboxd install` writes the file, it renames `[node]` to `[channel]`. An error about a key of `[node]` names the key under `channel`, such as `channel.labels`. A file with both sections is refused.

### The IPv6 block of an older release

A release before N29 gave each edge server an address of an IPv6 block of the vRack, in `edge.public_ipv6` with a prefix length, and the gateway of the block in `edge.public_ipv6_gateway`:

```toml
[edge]
public_ipv6 = "2001:db8:0:100::11/56"
public_ipv6_gateway = "2001:db8:0:100::1"
```

This release still reads such a file, so a server that upgrades still starts. With `edge.public_ipv6_gateway`, `hopboxd` ignores that key. It also ignores `edge.public_ipv6` when the value has a prefix length. It then logs this warning, in the same way as for `[node]`:

```text
edge.public_ipv6_gateway names the IPv6 block of the vRack, which this release removed, so hopboxd ignores it, and edge.public_ipv6 of the block with it. Remove both keys. The IPv6 egress of the machines now leaves from the uplink address of the edge server
```

Remove both keys from the file. To give the edge server a floating IPv6 address, set `edge.public_ipv6` to that address alone, without a prefix length. Then restart `hopboxd`. At its start, `hopboxd` removes the address of the block, its routes, and its rules from the server ([the egress of the machines](/reference/egress/index.html.md#what-the-ipv6-block-left)).

The paths of the sockets have a limit: a unix socket path has at most 103 bytes on macOS. So keep `run_dir` short. The admin socket needs 11 bytes more for its temp name ([the admin API](/reference/admin-api/index.html.md#the-admin-socket)).

## Examples

The smallest file. Every other key has its default:

```toml
roles = ["control"]
```

A server with the control role and the node role. The node role reaches the control role of the same `hopboxd` at the default `channel.control_urls`:

```toml
roles = ["control", "node"]
```

A control server on an external Postgres, such as managed PostgreSQL. The control role starts no bundled Postgres ([Run the control store on an external Postgres](/guide/external-postgres/index.html.md)):

```toml
roles = ["control", "node"]

[postgres]
url = "postgres://hopbox@db.example.org:5432/hopbox?sslmode=verify-full"
ca_file = "/etc/hopbox/postgres-ca.pem"
password_file = "/etc/hopbox/postgres-password"
```

The same server, with the resolvers of its machines:

```toml
roles = ["control", "node"]

[network]
nameservers = ["213.186.33.99", "9.9.9.9"]
```

A DNS server of the install. It enrolls with the control role at `channel.control_urls`, and answers DNS on port 53 of one address:

```toml
roles = ["dns"]

[channel]
control_urls = ["https://10.0.0.5:7443"]
labels = { zone = "gra" }

[dns]
listen = ["203.0.113.10:53"]
```

An edge server of the install. It enrolls with the control role at `channel.control_urls`, and sends the API and the console there:

```toml
roles = ["edge"]

[channel]
control_urls = ["https://10.0.0.5:7443", "https://10.0.0.6:7443"]

[edge]
https_listen = "203.0.113.10:443"
http_listen = "203.0.113.10:80"
ssh_listen = ":2222"
```

A file with every key:

```toml
roles = ["control", "node", "dns", "edge"]
data_dir = "/var/lib/hopbox"
run_dir = "/run/hopbox"

[control]
api_listen = "127.0.0.1:7080"
internal_listen = "127.0.0.1:7443"

[channel]
control_urls = ["https://127.0.0.1:7443"]
labels = { zone = "gra" }

[dns]
listen = [":53"]

[edge]
https_listen = ":443"
http_listen = ":80"
ssh_listen = ":2222"
ssh_redirect = true
public_ipv6 = "2001:db8::11"
public_ipv4 = "203.0.113.101"
floating_addresses = ["203.0.113.102"]

[acme]
name_servers = ["203.0.113.10:53"]
ca_file = "/etc/hopbox/acme-roots.pem"

[postgres]
install = "/opt/hopbox/current/postgres"
user = "hopbox-pg"
url = "postgres://hopbox@db.example.org:5432/hopbox?sslmode=verify-full"
ca_file = "/etc/hopbox/postgres-ca.pem"
password_file = "/etc/hopbox/postgres-password"

[metrics]
listen = "127.0.0.1:9464"
```

## Errors

`hopboxd run` refuses a file that breaks a rule, and exits with status 1. The error names the file, and then each broken rule on its own line.

| Error | Cause |
| --- | --- |
| `config: open <path>: no such file or directory` | The file does not exist. |
| `config: <path> belongs to uid <n>, and hopboxd runs as root, so the file must belong to root` | As root, the file belongs to another user. |
| `config: <path> has mode <mode>, which lets group or others change it, and hopboxd runs as root` | As root, group or others can write the file. |
| `config: <path> is not a regular file` | The path is a dir or a device. |
| `config: <path> is a symlink, and hopboxd runs as root, so it reads only a real file` | As root, the path is a symlink. |
| `config: the dir <dir> belongs to uid <n>, and hopboxd runs as root, so the dir must belong to root` | As root, the dir of the file belongs to another user. |
| `config: the dir <dir> has mode <mode>, which lets group or others change it, and hopboxd runs as root` | As root, group or others can write the dir of the file. |
| `config: the dir <dir> is a symlink, and hopboxd runs as root` | As root, the dir of the file is a symlink. |
| `config: <path> is larger than 65536 bytes` | The file is too large. |
| `config: <path>: toml: ...` | The file is not valid TOML, or a value has the wrong type, for example `roles = "control"`. |
| `config: <path>: unknown keys: <keys>` | The file has a key that is not in the table above, for example a typo such as `control.listen`. A typo never falls back to a default. |
| `roles: list at least one role. This hopboxd runs the roles: control, node, dns, edge` | `roles` is missing or empty. |
| `roles: unknown role "<role>". This hopboxd runs the roles: control, node, dns, edge` | A role that this `hopboxd` does not have, such as `proxy`. |
| `roles: the role "<role>" appears twice` | A role is in the list twice. |
| `<key>: "<value>" is not an absolute path` | `data_dir`, `run_dir`, `postgres.install`, `postgres.ca_file`, or `postgres.password_file` is a relative path. |
| `postgres.url: the value is not a URL` | `postgres.url` does not parse. |
| `postgres.url: "<url>" holds a password. Put the password in its own file, and name that file` | The URL has a password. Errors show the URL without its password and its query. |
| `postgres.url: "<url>" is not a postgres:// URL, such as postgres://hopbox@db.example.org:5432/hopbox?sslmode=verify-full` | The scheme is not `postgres` or `postgresql`. |
| `postgres.url: "<url>" names no host, such as ...` | The URL has no host. |
| `postgres.url: "<url>" names more than one host. Name the one host of the service, whose failover keeps its name` | The host part lists hosts with a comma. |
| `postgres.url: "<url>" names no user, such as ...` | The URL has no user. |
| `postgres.url: "<url>" names no database, such as ...` | The path is empty, or has more than one segment. |
| `postgres.url: "<url>" has a fragment` | The URL ends with `#...`. |
| `postgres.url: "<url>" has the parameter <name>. The CA comes from postgres.ca_file, the password from postgres.password_file, and the host, the user, and the database from the URL itself` | The query has a parameter of the refused list of `postgres.url`. |
| `postgres.url: "<url>" needs sslmode=verify-full, so hopboxd checks the certificate of the server and its host name` | `sslmode` is missing, appears twice, or has another value. |
| `postgres.ca_file: postgres.url needs it` | `postgres.url` is set, and `postgres.ca_file` is not. The same error exists for `postgres.password_file`. |
| `postgres.ca_file and postgres.password_file need postgres.url` | A file is set without `postgres.url`. |
| `postgres.ca_file: <path> holds no PEM certificate` | At the start of `hopboxd run`, the file holds no certificate. |
| `postgres.ca_file: config: ...` or `postgres.password_file: config: ...` | At the start of `hopboxd run`, the file is missing, or fails a check of the file of the bootstrap config, such as `belongs to uid <n>`. |
| `postgres.password_file: config: <path> has mode <mode>, which lets group or others read it, and it holds a secret. Give it mode 0400` | As root, group or others can read the password file. |
| `postgres.password_file: <path> must hold the password on one line` | The file is empty, or holds more than one line. |
| `control.api_listen: "<value>" is not host:port: ...` | The address has no port, or does not parse. |
| `control.api_listen: "<value>" is not a loopback IP address. ...` | The host is not a loopback IP address. |
| `control.internal_listen: "<value>" is not host:port: ...` | The address has no port, or does not parse. |
| `control.internal_listen: "<value>" has no port` | The address ends with `:`. |
| `control.internal_listen: "<value>" is not an IP address and a port, such as 127.0.0.1:7443` | The host is a name, such as `localhost`. |
| `control.internal_listen: "<value>" is also control.api_listen. Give each listener its own address` | Both listeners have the same address, with a port other than `0`. |
| `acme.name_servers: "<value>" is not an IP address and a port, such as 127.0.0.1:7443` | A name server is a host name. |
| `acme.name_servers: "<value>" is not host:port: ...` | A name server has no port, or does not parse. |
| `acme.name_servers: <n> addresses, more than 16` | The list is too long. |
| `acme.ca_file: "<value>" is not an absolute path` | The path is relative. |
| `control.release_image: "<value>" is not an absolute path` | The path is relative. |
| `acme.ca_file: ...` | At the start of `hopboxd run`, the file does not open, or holds no PEM certificate. |
| `[node] is the old name of [channel]. Keep only [channel]` | The file has both `[node]` and `[channel]`. |
| `network.nameservers: "<value>" is not an IPv4 address that a machine can reach, such as 9.9.9.9` | A resolver is not IPv4, or is a loopback, unspecified, link-local, multicast, or machine address. |
| `network.nameservers: <address> appears twice` | A resolver is in the list twice. |
| `network.nameservers: <n> resolvers, more than 3` | The list is too long. |
| `channel.control_url is the old key of channel.control_urls. Keep only channel.control_urls` | The file has both keys. |
| `channel.control_urls: "<value>" is not a URL: ...` | A value does not parse as a URL. |
| `channel.control_urls: "<value>" is not an https URL of a host, such as https://127.0.0.1:7443` | The scheme is not `https`, the host is missing, or the URL has user info, a path, a query, or a fragment. |
| `channel.control_urls: <n> URLs, more than 8` | The list is too long. |
| `channel.control_urls: "<value>" appears twice` | A URL is in the list twice. |
| `channel.labels: invalid label: ...` | A label breaks a rule of `channel.labels`. The rest of the message names the label and the rule. |
| `dns.listen: list 1 to 16 addresses, such as ":53"` | `dns.listen` is empty, or has more than 16 addresses. |
| `dns.listen: "<value>" is not host:port: ...` | The address has no `:`, or does not parse. |
| `dns.listen: "<value>" has no port number` | The port is empty, is not a number, or is larger than 65535. |
| `dns.listen: "<value>" is not an IP address and a port, such as 203.0.113.10:53, or a port alone, such as :53` | The host is a name, such as `localhost`. |
| `dns.listen: "<value>" appears twice` | An address is in the list twice. |
| `edge.https_listen: "<value>" is not host:port: ...` | The address has no `:`, or does not parse. The same errors exist for `edge.http_listen`. |
| `edge.https_listen: "<value>" has no port number` | The port is empty, is not a number, or is larger than 65535. |
| `edge.https_listen: "<value>" is not an IP address and a port, such as 203.0.113.10:443, or a port alone, such as :443` | The host is a name, such as `localhost`. |
| `edge.http_listen: "<value>" is also edge.https_listen. Give each listener its own address` | Both listeners of the edge role have the same address, with a port other than `0`. |
| `edge.ssh_listen: "<value>" is in the SSH ports 10000 to 30000, which the redirect sends to it. Pick a port outside them, such as :2222` | The port of the SSH listener is an SSH port of a machine. The errors of `edge.https_listen` also exist for `edge.ssh_listen`. |
| `edge.ssh_listen: "<value>" listens on one address, but the redirect of the SSH ports sends each connection to the address of the interface that took it. Give a port alone, such as :2222, or set edge.ssh_redirect = false` | `edge.ssh_redirect` is true, and the host of `edge.ssh_listen` is one address. |
| `edge.ssh_listen: "0.0.0.0:<port>" takes only IPv4, so the IPv6 clients that the redirect of the SSH ports sends to it get no answer. Give a port alone, such as :2222, or set edge.ssh_redirect = false` | `edge.ssh_redirect` is true, and the host of `edge.ssh_listen` is `0.0.0.0`. |
| `edge.ssh_listen: "<value>" is also an HTTP listener. Give each listener its own address` | The SSH listener has the address of `edge.https_listen` or `edge.http_listen`, with a port other than `0`. |
| `edge.public_ipv6: "<value>" is not an IPv6 address without a prefix length or a zone, such as 2001:db8::11` | The value does not parse as an IPv6 address. It is an IPv4 address, an IPv4 address in IPv6 form, or has a prefix length or a zone. A value with a prefix length and `edge.public_ipv6_gateway` gives a warning instead ([The IPv6 block of an older release](#the-ipv6-block-of-an-older-release)). |
| `edge.public_ipv6: <address> is not a public IPv6 address` | The address is not a global unicast address, or is a unique local address of `fc00::/7`. A link-local address is not a global unicast address. |
| `edge.public_ipv6: only a server with the role edge holds a floating address` | A server without the role `edge` sets the key. |
| `edge.public_ipv4: "<value>" is not an IPv4 address` | The value does not parse as an IPv4 address, such as `203.0.113.101/32` or an IPv6 address. |
| `edge.public_ipv4: <address> is not a public IPv4 address` | The address is not a global unicast address, or is in a private range, the vRack range, or the machine range. |
| `edge.public_ipv4: only a server with the role edge holds a floating address` | A server without the role `edge` sets the key. |
| `edge.floating_addresses: "<value>" is not an IPv4 address` | An entry does not parse as an IPv4 address, such as `203.0.113.102/32` or an IPv6 address. |
| `edge.floating_addresses: <address> is not a public IPv4 address` | An entry is not a global unicast address, or is in a private range, the vRack range, or the machine range. |
| `edge.floating_addresses: <address> appears twice` | The list holds an address two times. |
| `edge.floating_addresses: <n> addresses, more than 64` | The list is too long. |
| `edge.floating_addresses: only a server with the role edge holds a floating address` | A server without the role `edge` sets the key. |
| `metrics.listen: "<value>" is not a loopback IP address. ...` | The host of the metrics listener is not a loopback IP address. |
| `metrics.listen: "<value>" is not host:port: ...` | The address has no port, or does not parse. |
