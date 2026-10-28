# Enrollment and the install PKI

> The install CA, the enrollment token, the steps of an enrollment, the identity files of a server, and EnrollmentService.Enroll.

Enrollment makes a new server a part of an install. This page covers the certificates of an install and the enrollment token. It also covers the command `hopboxd enroll`, the call `EnrollmentService.Enroll`, and the check of a server certificate on each call. The code is in `internal/pki`, `internal/node`, `internal/service`, and `cmd/hopboxd`.

After the enrollment, the server keeps a control channel to the control role ([the control channel](/reference/control-channel/index.html.md)). The server enrolls and talks to the control role on the internal listener ([the internal listener](/reference/internal-listener/index.html.md)).

## Overview

1. Root on a control server makes an enrollment token: `sudo hopboxd admin enrollment-token create` ([hopboxd](/reference/hopboxd/index.html.md#admin-enrollment-token-create)).
2. Root on the new server runs `sudo hopboxd enroll --token <token>`. The command runs the preflight check ([the preflight check](/reference/preflight/index.html.md)), makes a key, and calls `Enroll`.
3. The control role checks the token, records the server in the registry, and signs a certificate for the key of the server.
4. The new server saves its identity in `<data_dir>/server`. `hopboxd run` with a role other than `control` then opens the control channel with it.

The server never sends its private key. First it checks the CA of the control role against the CA hash in the token. Until that check passes, it sends nothing, not even the token.

## Certificates

Every key of an install is an ECDSA key on the curve P-256. Every certificate starts 5 minutes before the time of its issue, so a server with a clock that runs a little behind still accepts it.

### Install CA

| Item | Value |
| --- | --- |
| Key | ECDSA P-256 |
| Subject | `CN=hopbox install CA` |
| Lifetime | 10 years from its creation |
| Basic constraints | CA, path length 0 |
| Key usage | certificate sign, CRL sign, digital signature |
| Hash | the SHA-256 of the DER certificate, in lowercase hex (64 characters). An enrollment token holds it. |
| Storage | the table `install_ca` of the control store ([Control store tables](/reference/control-store/index.html.md#install_ca)). The certificate is plain. The key, in PKCS #8 DER, is sealed with the place `InstallSecret{Name: "install_ca.key"}` ([Sealing](/reference/sealing/index.html.md#places)). |

The install has one CA. The first control process that needs it makes it: the first `CreateEnrollmentToken`, or the start of the internal listener. To seal the key, the keyring must have a key-encryption key. `hopboxd install` makes the first one ([hopboxd](/reference/hopboxd/index.html.md#install)). On a server without it, root makes one with `sudo hopboxd admin kek rotate` ([hopboxd](/reference/hopboxd/index.html.md#admin-kek-rotate)). Until then, each call that needs the CA fails with `failed_precondition` and the message `failed precondition: the keyring has no key-encryption key to seal or open the value. Run hopboxd admin kek rotate to make the first one`.

Two control processes that make a CA at the same time both insert it. Only the first insert counts, and both read that one. Each control process keeps the CA in memory after its first read. A new CA makes the record `created the install CA` with the field `ca_hash` and `component=service`.

### Server certificate

The CA signs one certificate for each enrolled server, and a new one at each renewal ([the control channel](/reference/control-channel/index.html.md#certificate-renewal)).

| Item | Value |
| --- | --- |
| Key | ECDSA P-256. The server makes it. The request must prove that the server holds the private key. |
| Subject | `CN=<server ID>` |
| Lifetime | 30 days, and never past the end of the CA |
| Key usage | digital signature |
| Extended key usage | client auth only. A server certificate never works as the certificate of a listener. |
| URI SAN | `hopbox://server/<server ID>?roles=<roles>`, for example `hopbox://server/01a14403-6bd1-7522-9ab0-b3ab4517e0e4?roles=node`. The roles are lowercase words joined by plain commas, for example `?roles=node,dns`. |
| Serial number | 128 random bits. The control store keeps it in lowercase hex, without leading zeros. |

A certificate names exactly one server URI, with 1 to 8 roles. The roles of the certificate are the roles of the server in the registry.

### Control certificate

The internal listener presents a certificate of the same CA ([the internal listener](/reference/internal-listener/index.html.md)). A control process also sends it as its client certificate to the internal listener of another control process.

| Item | Value |
| --- | --- |
| Key | ECDSA P-256, new for each certificate. It stays in the memory of the control process. |
| Subject | `CN=hopbox control` |
| Lifetime | 30 days, and never past the end of the CA |
| Extended key usage | server auth and client auth. Client auth lets a control process send the certificate when it calls another one ([the control channel](/reference/control-channel/index.html.md#streams-through-another-control-process)). |
| URI SAN | `hopbox://control` |
| IP SAN | the IP address of `control.internal_listen`, such as `127.0.0.1`. For an address such as `0.0.0.0`, which listens on every address, the certificate has no IP SAN. |
| Chain | the certificate, then the CA certificate |

Each control process makes its own control certificate. It makes it at the start of its internal listener, or at the first handshake after the CA exists. At the first handshake after the certificate is 20 days old, it makes a new one.

A server checks three things about the control certificate. It must chain to the install CA. It must have the extended key usage server auth. It must name `hopbox://control`. If a check fails, the server refuses the control role. The server does not check a host name.

## Enrollment token

An enrollment token lets one server enroll into one pool.

| Item | Value |
| --- | --- |
| Form | `hbxe_<id>_<secret>_<cahash>`, 167 characters |
| `<id>` | the ID of the token, a UUIDv7 as 32 lowercase hex digits without hyphens |
| `<secret>` | 32 random bytes as 64 lowercase hex digits |
| `<cahash>` | the hash of the install CA, 64 lowercase hex digits |
| Lifetime | 1 hour from its creation by default. The call that makes it can set another `ttl`, at most 24 hours. |
| Use | once. The first enrollment that uses the token spends it. |
| Scope | one pool, by default the pool `default` |
| Peer address | none by default. A token can name the peer address of its server. The enrollment must then come from that address ([Peer addresses](#peer-addresses)). |
| Storage | the table `enrollment_tokens` ([Control store tables](/reference/control-store/index.html.md#enrollment_tokens)). It keeps the SHA-256 of the secret, never the secret. |

The text of a token is a secret. Keep it out of logs and shell history. `hopboxd enroll` also takes it from the environment variable `HOPBOX_ENROLLMENT_TOKEN`. The secret scan of the commits refuses the form of a token.

A token for another install CA never works: the new server refuses the control role in the TLS handshake, and the control role refuses the token too. A wrong secret does not spend a token. A failed check of the request, such as a bad key, does not spend it either.

The leader deletes the tokens that expired more than 1 day ago, every 10 minutes ([hopboxd](/reference/hopboxd/index.html.md#leader-jobs)).

### Make a token

The admin call `ServerService.CreateEnrollmentToken` makes a token ([the admin API](/reference/admin-api/index.html.md#createenrollmenttoken)). On a control server, run:

```sh
sudo hopboxd admin enrollment-token create [--pool NAME] [--roles ROLE,...] [--peer-address ADDRESS] [--ttl DURATION] [--json]
```

The command prints the token on one line, so a script can keep it in a variable. With `--peer-address`, the server must enroll from that address. `--ttl` sets the lifetime of the token, such as `2h`, at most `24h`. The token allows only its roles: `node`, `dns`, `edge`, or more than one, by default `node`. The row of the token keeps them. The response holds the only copy of the token. The call takes an idempotency key, but the control store keeps no copy of its response, so a retry with the same key fails with `already_exists` and makes no second token ([the admin API](/reference/admin-api/index.html.md#createenrollmenttoken)). A lost token costs only a new token.

### Peer addresses

The peer address of a server is the public address at which the other servers reach it (plan question Q91). It is one public unicast IPv4 address on the uplink of the server, with no NAT. The install refuses a private, loopback, link-local, multicast, or reserved address. It also refuses an address in the shared address space `100.64.0.0/10` of carrier NAT, the mesh range `10.239.0.0/16`, or the machine range `10.240.0.0/12`. The table `servers` keeps it ([Control store tables](/reference/control-store/index.html.md#servers)). A peer address belongs to one server until the server is decommissioned, because the Terraform provider imports a server by its peer address (plan question Q96).

A token can name the peer address of the server that enrolls with it (plan question Q92). The enrollment must then come from that address. The install makes no such token while a control process of a release before migration `0054` is live, because that release enrolls from any address ([the admin API](/reference/admin-api/index.html.md#createenrollmenttoken)). The new server keeps the address as its peer address. A heartbeat never changes a peer address. Only these three paths set it:

- The enrollment with a token that names one. For a server that adds roles to itself, such a token sets the peer address only when the server has none ([Add roles to an enrolled server](#add-roles-to-an-enrolled-server)).
- `hopboxd install --peer-address` on the first server, which enrolls with a token from its own control role ([hopboxd](/reference/hopboxd/index.html.md#install)).
- `hopboxd admin server set-peer-address`, such as on a server from before peer addresses ([the admin API](/reference/admin-api/index.html.md#setserverpeeraddress)).

A token without a peer address enrolls as before, and its server gets no peer address. Such a token enrolls from these addresses:

| The install | A token without a peer address enrolls from |
| --- | --- |
| No server of the install ever had a peer address. | Any address that reaches the internal listener. |
| A server of the install has or had a peer address, also a decommissioned one. | The control server itself only, such as the node role of a one-server install: loopback, or the address of the internal listener on which the call came. After the move of the internal listener to the peer address of the first server, its roles dial the listener from that address ([the internal listener](/reference/internal-listener/index.html.md#move-the-first-server-to-its-peer-address)). |

So once the servers of an install have peer addresses, a token that leaks is no use from another host. A connection that comes from the address of the control server itself comes from that host: TCP refuses a forged source, and a packet of a machine to an address of its host takes no source NAT. The host firewall also keeps the egress of the machines away from every peer address ([the host firewall](/reference/host-firewall/index.html.md#the-egress-of-the-machines)). The rule of the address of the control server itself is a build default of N28, not an owner decision. The rule stays for good: the decommission of the last server with a peer address does not end it. This is a build default that follows from Q91 and Q92, not an owner decision. The rule also holds when an enrolled server adds roles to itself ([Add roles to an enrolled server](#add-roles-to-an-enrolled-server)). The control role checks the address under the lock of the server registry, which `SetServerPeerAddress` also takes. So the first peer address of an install and an enrollment never overlap.

A control process of a release before migration `0054` ignores both rules. So while such a process is live, `CreateEnrollmentToken` refuses a `peer_address`, and `SetServerPeerAddress` refuses every call ([the admin API](/reference/admin-api/index.html.md#setserverpeeraddress)). A rollback of a control server to such a release still lets that process ignore both rules. It enrolls each live token from any address, until the token expires, at most 24 hours later. Wait for the live tokens to expire before a rollback across migration `0054`, or roll forward again.

While a token with a peer address is not used and not expired, its address is in the set `registered` ([Control store tables](/reference/control-store/index.html.md#enrollment_tokens)). The host firewall of each control server opens the public port of the internal listener to the addresses of that set, so that a new server can enroll ([the host firewall](/reference/host-firewall/index.html.md#sets)). A new token reaches the firewall of each control server within 2 s, so start the enrollment 2 s after the make of the token. After the enrollment, the address of the server is in the set `peers`, which keeps the listener open to its control channel. A long `ttl` keeps the listener open to that address for that time. So the default lifetime is 1 hour, and the longest is 24 hours.

## hopboxd enroll

```sh
hopboxd enroll --token=TOKEN [--config=PATH] [--json]
```

`enroll` makes this server a part of an install. Run it as root on the new server.

| Flag | Type | Default | Description |
| --- | --- | --- | --- |
| `--token` | string | the variable `HOPBOX_ENROLLMENT_TOKEN` | The enrollment token. Required. |
| `--config` | path | `/etc/hopbox/hopboxd.toml` | The bootstrap config ([the bootstrap config of hopboxd](/reference/hopboxd-config/index.html.md#keys)). `enroll` reads `roles`, `data_dir`, and `channel.control_urls`. |
| `--json` | bool | `false` | Prints the identity as one JSON object instead of the server ID. |

`enroll` takes these steps, and stops at the first that fails:

1. It reads the bootstrap config. The roles that enroll are every role of the config but `control`. Today these roles are `node`, `dns`, and `edge`.
2. It checks the form of the token.
3. Only with the node role, it runs the preflight check ([the preflight check](/reference/preflight/index.html.md)) and prints one line for each check to standard error. A failed check of severity `fail` stops the enrollment.
4. It checks that `<data_dir>/server` holds no identity yet.
5. It reads the hardware of the server ([Hardware of a pool](#hardware-of-a-pool)).
6. It makes a new ECDSA P-256 key and a certificate request for it. The request names nothing: the control role sets every name.
7. It dials the first URL of `channel.control_urls` over TLS 1.3 with HTTP/2. If the dial fails, it goes on to the next URL. A call that reached a control server is never sent again, because the token works once. In the handshake, it finds the CA certificate with the CA hash of the token in the chain of the control role. The certificate of the control role must chain to that CA and name `hopbox://control`. If a check fails, it ends the handshake and sends nothing.
8. It calls `EnrollmentService.Enroll` with the token, the request, the host name of the server, the roles, and the hardware.
9. It checks the answer. The CA certificate must have the hash of the token. The signature of the server certificate must be from the CA. The certificate must hold the new key and name the server ID and the roles of the answer.
10. It saves the identity in `<data_dir>/server`.

The whole command has at most 60 seconds.

On success, `enroll` prints the server ID on standard output, and this line on standard error:

```text
hopboxd: enrolled as the server <server ID>. hopboxd run connects it to the control plane.
```

With `--json`, it prints one object:

| Key | Value |
| --- | --- |
| `server_id` | The ID of the server. |
| `roles` | The roles of the server, such as `["node"]` or `["dns"]`. |
| `ca_hash` | The hash of the install CA. |
| `certificate_expire_time` | The end of the server certificate, in UTC. |
| `dir` | The dir of the identity, `<data_dir>/server`. |

Example:

```text
$ sudo hopboxd enroll --token "$TOKEN"
preflight kvm	ok
preflight cgroup2	ok
preflight tun	ok
preflight nftables	ok
preflight reflink	ok
preflight kernel	ok
hopboxd: enrolled as the server 01a14403-6bd1-7522-9ab0-b3ab4517e0e4. hopboxd run connects it to the control plane.
01a14403-6bd1-7522-9ab0-b3ab4517e0e4
```

A server that is enrolled already, and whose config lists a role beyond those of its identity, adds the new roles in place ([Enrollment and the install PKI](/reference/enrollment/index.html.md#add-roles-to-an-enrolled-server)). The token must allow each new role. The server keeps its ID. `enroll` then prints `hopboxd: the server <id> has the roles <roles> now. Restart hopboxd to use them: systemctl restart hopboxd`. `hopboxd install` restarts it on its own.

`enroll` does not start anything. The `channel` unit of `hopboxd run` waits for the identity and opens the control channel once it exists, so `hopboxd run` can run before `enroll` ([the control channel](/reference/control-channel/index.html.md#the-agent)).

### Add roles to an enrolled server

A role that a new release brings, such as `edge`, comes to a server that is enrolled already, in place. The server keeps its ID and its machines.

1. An operator makes a token for the new roles alone, that names the server, such as `hopboxd admin enrollment-token create --server <id> --roles edge`. Only that server takes the token, so a token that leaks adds no role to another server. No new server enrolls with it.
2. The server calls `EnrollmentService.Enroll` with the token, a new CSR, and every role, the old ones and the new ones. It calls with its current certificate as the client certificate, so the control role knows which server asks.
3. The control role checks that the token names the server, and allows each new role. It refuses a list that drops a role of the server with `invalid_argument`. It checks the address of the call as for a new server ([Peer addresses](#peer-addresses)). A token that names a peer address other than the peer address of the server fails with `unauthenticated`. A token that names a peer address gives that address to a server without one. It sets the roles of the server in the registry, issues a certificate with every role, and uses the token. The audit entry is `server.add_roles`, with `old_roles`, `roles`, `token_id`, and `serial`, and `peer_address` when the token gave the server its peer address.

A server keeps one peer address. So `CreateEnrollmentToken` refuses a token with `server_id` and a `peer_address` other than the peer address of that server. A token without `peer_address` works for any server. These rules are build defaults that follow from Q91 and Q92, not owner decisions.
4. The server saves the new identity. Its old certificate stops at once, because its roles no longer match the registry. So hopboxd must restart to use the new one. `hopboxd install` restarts it, and `hopboxd enroll` says to.

### Hardware of a pool

All servers of a pool have identical hardware. So the first server that enrolls into a pool records its hardware in the pool, and the enrollment of a new server with other hardware fails (plan question Q61). The pool shows the hardware in the field `hardware` ([the admin API](/reference/admin-api/index.html.md#poolservice)).

`hopboxd enroll` reads these facts on a new server. A server that adds roles to itself ([Add roles to an enrolled server](#add-roles-to-an-enrolled-server)) reads and sends none, because the control role checks the hardware only of a new server.

| Fact | Source |
| --- | --- |
| CPU model | `model name` of the first processor in `/proc/cpuinfo` |
| CPU threads | The CPUs in `/sys/devices/system/cpu/online`, the same count as in the heartbeats ([the control channel](/reference/control-channel/index.html.md)) |
| RAM | `MemTotal` of `/proc/meminfo` |
| Disks | The size of each whole disk under `/sys/block`. A whole disk has a `device` entry and `removable` 0, so loop devices, RAID and device-mapper volumes, zram, and optical drives do not count. A disk of size 0 does not count either. |

On Linux, `hopboxd enroll` refuses a new server whose CPU is not amd64, before it reads these facts. In v1, every server has an Intel or AMD CPU. Also, only the `/proc/cpuinfo` of amd64 names the vendor, the family, and the model of the CPU fingerprint ([the control channel](/reference/control-channel/index.html.md)).

On a Mac, which enrolls only the `dns` and the `edge` roles in development ([Serve the domain of an install with the DNS role](/guide/dns-role/index.html.md)), `hopboxd enroll` reads the CPU model, the CPU threads, and the RAM with `sysctl`: `machdep.cpu.brand_string`, `hw.logicalcpu`, and `hw.memsize`. It reports no disk.

The control role compares a new server with the hardware of its pool:

| Fact | Rule |
| --- | --- |
| CPU model | The same text |
| CPU threads | The same count |
| RAM | At most 2% apart, because the firmware of two servers of one model can keep a few MiB more or less (`domain.HardwareTolerance`) |
| Disks | The same count, and each disk at most 2% apart from the disk of the same rank, in the order of size |

The pool `default` and each pool from before this rule have no hardware yet. The next server that enrolls into such a pool records it, not the servers that the pool holds already. An install from before this rule can enroll a server of another model beside its old server in the pool `default`. The pool records no hardware of the old server, so it takes that new server.

A pool whose servers are all decommissioned takes the hardware of its next server. So an instance admin who wants another model in a pool decommissions every server of the pool first. Only an enrollment adds a server, and each enrollment holds the lock of the server registry and the lock of the pool, so no server joins between the count of the live servers and the record of the hardware. The same hardware again leaves the pool and its version as they are.

### Errors of enroll

| Error | Cause |
| --- | --- |
| `the config <path> lists no role that enrolls. Add "node", "dns", "edge", or more than one to roles` | The config lists only `control`. |
| `the token does not have the form hbxe_<id>_<secret>_<cahash>. Make one with hopboxd admin enrollment-token create` | The token is not a token. |
| `the preflight check failed, so this server cannot run machines:` and one line for each failed check | A check of severity `fail` failed ([the preflight check](/reference/preflight/index.html.md)). |
| `this server is enrolled already as the server <id> with the roles <roles>. To enroll it again, remove <dir> first` | `<data_dir>/server` holds an identity with every role of the config. |
| `enroll at <url>: invalid_argument: the server has the role <role>, and a role cannot go...` | The config lists fewer roles than the identity. A role never goes from a server. |
| `enroll at <url>: unavailable: ...pki: the peer is not the control plane of this install: no certificate of its chain has the CA hash of the enrollment token` | The control role at the URL has another CA than the token names, or the peer is not a control role. |
| `enroll at <url>: unavailable: ...` with a TLS or a connection error | No control role listens at this URL of `channel.control_urls`, or the install has no CA yet, so its handshakes fail. With several URLs, the error holds one such line for each URL. |
| `enroll at <url>: unauthenticated: the enrollment token is not valid...` | The control role refused the token ([Errors of Enroll](#errors-of-enroll)). |
| `enroll at <url>: invalid_argument: ...` | The control role refused the request, for example a role that it does not know. |
| `enroll at <url>: permission_denied: permission denied: the enrollment comes from <address>...` | The server enrolled from another address than the peer address of the token, or from an address other than loopback with a token without one ([Peer addresses](#peer-addresses)). |
| `read the hardware of the server: hostinfo: a server of hopbox needs an amd64 CPU from Intel or AMD, and this host runs <arch>` | A new server on Linux, with a CPU that is not amd64, such as `arm64` ([Hardware of a pool](#hardware-of-a-pool)). |
| `read the hardware of the server: ...` | `enroll` cannot read `/proc/cpuinfo`, `/proc/meminfo`, or `/sys/block`. Off Linux, the message ends with `the CPU and memory facts need Linux`. |
| `enroll at <url>: failed_precondition: the hardware of this server differs from the pool <pool>: ...` | The server has other hardware than the first server of its pool ([Hardware of a pool](#hardware-of-a-pool)). |
| `node: the control plane sent another CA than the token names` | The answer holds another CA certificate. |
| `node: the certificate does not hold the key of this server` | The answer holds a certificate for another key. |

## Identity files

| Path | Mode | Contents |
| --- | --- | --- |
| `<data_dir>/server` | `0700` | The dir of the identity. `enroll` makes it. |
| `<data_dir>/server/identity.pem` | `0600` | One `CERTIFICATE` block, the server certificate, and one `PRIVATE KEY` block, the PKCS #8 key of the server. |
| `<data_dir>/server/ca.pem` | `0644` | One `CERTIFICATE` block, the install CA. |

The default `data_dir` is `/var/lib/hopbox`, so the default dir is `/var/lib/hopbox/server`.

Each write goes to a temp file `.<name>.<random>.tmp` in the same dir, which `hopboxd` syncs and then renames. The CA file goes first and the identity file last. So a crash leaves the old identity or the new one, never a half. A renewal rewrites `identity.pem` in the same way.

`hopboxd` checks the identity each time that it reads it. It refuses an `identity.pem` that is not a regular file. It refuses one that group or others can read, with `node: <path> has mode <mode>, which lets group or others read the key of the server. Use mode 0600`. It refuses one that belongs to another user than the user of `hopboxd`. It also refuses a certificate that the CA of `ca.pem` did not sign, and a certificate that does not hold the key of the file.

### Enroll again

You can enroll a server again, for example after its install starts from nothing. Stop `hopboxd`, remove `<data_dir>/server`, and run `hopboxd enroll` with a new token. The server then gets a new server ID. The old server stays in the registry, unresponsive.

## EnrollmentService

`hopbox.cluster.v1.EnrollmentService` runs only on the internal listener ([the internal listener](/reference/internal-listener/index.html.md)). It is not part of the public API, and the SDKs do not include it.

### Enroll

| Item | Value |
| --- | --- |
| Procedure | `/hopbox.cluster.v1.EnrollmentService/Enroll` |
| Credential | the enrollment token in the request. The call needs no client certificate. |
| Policy action | `server.enroll`, every caller ([the API server](/reference/api/index.html.md#callers-and-policy)) |
| Idempotent option | no |
| Audit entry | `server.enroll` |

`EnrollRequest`:

| Field | Type | Description |
| --- | --- | --- |
| `token` | string | The enrollment token. |
| `csr` | bytes | A DER certificate request for an ECDSA P-256 key of the server. |
| `hostname` | string | The host name of the server, for the operator only. At most 253 bytes of printable ASCII without spaces. It can be empty. |
| `roles` | repeated string | The roles of the server: `node`, `dns`, `edge`, or more than one. |
| `hardware` | `Hardware` | The hardware of the server ([Hardware of a pool](#hardware-of-a-pool)). A new server must send it. |

`Hardware`:

| Field | Type | Description |
| --- | --- | --- |
| `cpu_model` | string | The model name of the CPU, such as `AMD EPYC 4244P 6-Core Processor`. Required. |
| `cpu_threads` | uint32 | The count of CPU threads. Required. |
| `memory_bytes` | uint64 | The RAM that the kernel sees, in bytes. Required. |
| `disk_bytes` | repeated uint64 | The sizes of the whole disks, in bytes, in any order. It can be empty, but a disk of 0 bytes fails. |

`EnrollResponse`:

| Field | Type | Description |
| --- | --- | --- |
| `server_id` | string | The ID of the new server, a UUIDv7. |
| `certificate` | bytes | The DER server certificate. |
| `ca_certificate` | bytes | The DER certificate of the install CA. |
| `pool` | string | The name of the pool that the server joined. |

The control role checks the request in this order:

1. The policy check, which allows every caller.
2. The form of the token.
3. The roles: at least one, each one a role that a server can enroll with, none twice.
4. The host name.
5. The install CA exists, or the control role makes it.
6. The CA hash of the token is the hash of the install CA.
7. For a new server, the hardware: a CPU model, CPU threads, RAM, and no disk of 0 bytes. A server that adds roles to itself skips this check.
8. In one transaction, it locks the token row. The token must exist, its secret must match, it must not be used, and it must not be expired. Each role of the request must be a role of the token. Otherwise the call fails with `unauthenticated` and the message `the enrollment token is not valid: it allows only the roles <roles>, not <role>. Make a token with --roles`, and the token stays unused.
9. In the same transaction, the certificate request must parse, its signature must be valid, and its key must be ECDSA P-256. Then the CA signs the certificate. So the CA signs only for a caller that holds a live token, and a bad request leaves the token unused.
10. In the same transaction, it takes the lock of the server registry. Then it checks the address from which the call came ([Peer addresses](#peer-addresses)). A token with a peer address needs that address. A token without one needs loopback once a server of the install has or had a peer address. Otherwise the call fails with `permission_denied`. The certificate of step 9 then goes nowhere, and the token stays unused.
11. In the same transaction, it locks the row of the pool of the token. A pool that an instance admin deleted after the make of the token fails with `failed_precondition`, and the token stays unused ([the admin API](/reference/admin-api/index.html.md#deletepool)).
12. A pool without hardware records the hardware of the request. A pool with hardware compares it with the request ([Hardware of a pool](#hardware-of-a-pool)). Other hardware fails with `failed_precondition`, and the token stays unused. The lock of the pool makes two first enrollments at the same time record the hardware once.
13. A token with a peer address that another server holds fails with `already_exists`, and the token stays unused. The transaction inserts the server with the peer address of the token, its certificate, and the use of the token, and writes the audit entry.

Two enrollments with one token wait for each other on the row lock, so only one of them succeeds.

The new server has the state `unresponsive` until its first heartbeat. The heartbeat makes it `ready` ([the control channel](/reference/control-channel/index.html.md#server-states)).

The audit entry `server.enroll` has the actor of the caller, most often `anonymous`. Its target is the new server: the target kind `server` and the server ID. It has these details:

| Detail | Value |
| --- | --- |
| `pool` | The name of the pool. |
| `token_id` | The ID of the token. |
| `hostname` | The host name from the request. |
| `roles` | The roles, joined by commas. |
| `serial` | The serial number of the certificate, in hex. |
| `hardware` | The hardware of the server, as text, such as `AMD EPYC 4244P 6-Core Processor, 12 CPU threads, 62.5 GiB of RAM, disks 960 GB, 960 GB`. |
| `peer_address` | The peer address of the token, such as `203.0.113.12`. Only an enrollment with a token that names one has it. |

The service also writes the log record `enrolled a server` with `component=service` and the fields `server`, `pool`, `hostname`, and `roles`. The first server of a pool also writes `recorded the hardware of a pool`, with the fields `pool` and `hardware`.

### Errors of Enroll

| Code | Message | Cause |
| --- | --- | --- |
| `unauthenticated` | `the enrollment token is not valid: it does not have the form hbxe_<id>_<secret>_<cahash>` | The token is not a token. |
| `unauthenticated` | `the enrollment token is not valid: it was made for another install CA` | The CA hash of the token is not the hash of the install CA. |
| `unauthenticated` | `the enrollment token is not valid` | No token has the ID, or the secret does not match. |
| `unauthenticated` | `the enrollment token is not valid: it was used already. Make a new token` | A server enrolled with the token already. |
| `unauthenticated` | `the enrollment token is not valid: it expired at <time>. Make a new token` | The lifetime of the token is over. The time is in RFC 3339, in UTC. |
| `permission_denied` | `permission denied: the enrollment comes from <address>, but the token allows only the peer address <peer address>` | The token names a peer address, and the call came from another address. `<address>` is `an unknown address` when the call has no network address. |
| `permission_denied` | `permission denied: the enrollment comes from <address>, and a token without a peer address enrolls only from this control server once a server of the install has a peer address. Make a token with --peer-address` | The token names no peer address, a server of the install has or had one, and the call came neither from loopback nor from the address of the control server on which it came. |
| `invalid_argument` | `invalid argument: unknown server role: the server names no role` | The request has no role. |
| `invalid_argument` | `invalid argument: unknown server role: "<role>". A server enrolls with the roles: node, dns, edge` | A role that a server cannot enroll with, such as `control`. |
| `invalid_argument` | `invalid argument: unknown server role: the role "<role>" appears twice` | A role appears twice. |
| `unauthenticated` | `the enrollment token is not valid: it names the peer address <address>, but the server has the peer address <server address>. Make a token with --peer-address <server address>` | A server that adds roles to itself with a token whose peer address is not its own, such as after `SetServerPeerAddress` changed it. |
| `already_exists` | `already exists: the server <id> has the peer address <address>. A peer address belongs to one server until its decommission` | Another server that is not decommissioned took the peer address of the token after its make. |
| `invalid_argument` | `invalid argument: the host name has <n> bytes, more than 253` | The host name is too long. |
| `invalid_argument` | `invalid argument: the host name "<name>" holds a character that is not printable ASCII` | The host name holds a space, a control character, or a character that is not ASCII. |
| `invalid_argument` | `invalid argument: pki: the certificate request is not valid: ...` | The request does not parse, its signature is wrong, or its key is not ECDSA P-256. |
| `failed_precondition` | `failed precondition: the keyring has no key-encryption key to seal or open the value. Run hopboxd admin kek rotate to make the first one` | The install has no CA and no key-encryption key. In practice, the TLS handshake fails before the call. |
| `failed_precondition` | `failed precondition: the pool <pool> was deleted. Make a token for a live pool` | An instance admin deleted the pool of the token after its make. |
| `failed_precondition` | `failed precondition: the hardware of this server differs from the pool <pool>: it has <differences>. All servers of a pool have identical hardware, so enroll it into a pool of its server model` | A new server whose hardware differs from the hardware of its pool. `<differences>` names each fact that differs, such as `16 CPU threads, not 12`. |
| `invalid_argument` | `invalid argument: the server reports no hardware. Update hopboxd on the server` | A new server from a release of `hopboxd` that does not send its hardware. |
| `invalid_argument` | `invalid argument: invalid hardware: ...` | The hardware has no CPU model, no CPU thread, no RAM, or a disk of 0 bytes. |

## Client certificates on each call

The internal listener verifies a client certificate against the install CA in the TLS handshake: its chain, its validity period, and the extended key usage client auth. A certificate of another CA fails the handshake. Then the request gate asks the server-certificate resolver for the caller ([the internal listener](/reference/internal-listener/index.html.md#callers)). The resolver ties the certificate to a server of the registry with these checks, in order:

| Check | Refusal |
| --- | --- |
| The certificate names exactly one `hopbox://server/<id>` URI with valid roles. | `the server certificate is not valid: pki: the certificate names no server: ...` |
| The table `server_certificates` has its serial. | `the server certificate is not valid: the CA never issued the serial <serial> to a server` |
| That row names the server of the URI. | `the server certificate is not valid: the serial <serial> belongs to the server <id>, not <id>` |
| The certificate has not expired. | `the server certificate is not valid: the certificate <serial> expired` |
| No renewal retired the certificate. | `the server certificate is not valid: a renewal retired the certificate <serial>` |
| The roles of the URI are the roles of the server in the registry. | `the server certificate is not valid: the certificate <serial> names the roles [<roles>], and the registry [<roles>]` |
| The server is not decommissioned. | `the server certificate is not valid: the server <id> is decommissioned` |

A certificate that passes proves the caller `server` with the ID of the server. A refused certificate ends the call with `unauthenticated` and the message `the credential is not valid`. The cause goes to the log record `api credential refused` ([the API server](/reference/api/index.html.md#logs)).

The first call with a certificate records its first use. That call also retires every other certificate of the server, older or never used, and logs `retired other server certificates` with `component=service` and the fields `server`, `serial`, and `retired`. Then the control role ends a channel that a retired certificate holds open ([the control channel](/reference/control-channel/index.html.md#certificate-renewal)). So after a renewal, the old certificate works only until the server uses the new one, which it does at once. A stolen copy of the old certificate then stops working. A server gets at most one new certificate an hour.

The caller `server` carries the serial of its certificate. A heartbeat or a renewal on a channel whose certificate a renewal retired fails with `unauthenticated` and the message `the server is decommissioned or unknown: a renewal retired the certificate of this channel`, which ends the channel.

A call with a retired certificate is a sign that another party holds a copy of it, because the server holds a newer one. So the refusal also logs `a retired server certificate called. Another party can hold a copy of it` at level error with `component=service` and the fields `server`, `serial`, `peer`, and `suppressed`, and writes the audit entry `server.retired_certificate_used`. The entry has the actor `server`, the server as target, and the details `serial`, `peer` (the address of the caller), and `suppressed`. A copy that keeps dialing gets this alarm at most once an hour for each certificate, so it cannot flood the audit log. The calls in between are only refused and counted, and `suppressed` in the next alarm gives their count. The count lives in the control store, so it holds across control processes. In maintenance mode, the alarm writes nothing: each call logs the error without `suppressed`, and gets the refusal of maintenance mode ([the API server](/reference/api/index.html.md#maintenance-mode)). The first use of a certificate writes too, so maintenance mode refuses it the same way.

The leader deletes a certificate more than 1 day after it expired, except the newest certificate of each server ([hopboxd](/reference/hopboxd/index.html.md#leader-jobs)).
