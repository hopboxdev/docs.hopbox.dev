# The control channel

> The control channel of a server: its messages, heartbeats, certificate renewal, the reverse-attach data streams and their windows, the machine streams of exec, file copy, and logs, server states, and logs.

The control channel is the stream that each enrolled server keeps open to the control role. It carries the heartbeats of the server and the commands of the control role. This page covers the protocol `hopbox.cluster.v1.ControlService`, the heartbeat, the server states, the certificate renewal, the DNS zones, the mesh, the data streams, and the forwarding of data streams between control processes. The code is in `internal/control/channel` (the control side) and `internal/node` (the server side).

A server enrolls first ([Enrollment and the install PKI](/reference/enrollment/index.html.md)). Both calls of the channel run on the internal listener, over TLS 1.3 and HTTP/2, with the certificate of the server ([the internal listener](/reference/internal-listener/index.html.md)).

## The server always dials

The server opens every connection. It opens `Channel` and keeps it open. When the control role needs a data stream to the server, it sends `OpenStream` on the channel, and the server opens `AttachStream` for it. This is the reverse-attach pattern. HTTP/2 carries the channel and every data stream on one connection, so hopbox needs no other multiplexer.

On a server with the control role and a role that enrolls, the control role runs in the same `hopboxd` as the `channel` unit. The `channel` unit still connects over mutual TLS.

Each server keeps one channel, whatever its roles. The node role, the DNS role, and the edge role use the same channel of their server. The DNS role gets its zones on it ([DNS zones](#dns-zones)). The edge role gets its certificate, its routes, and the access changes on it ([Edge pushes](#edge-pushes)). The mesh device of the server gets its peers on it ([Mesh pushes](#mesh-pushes)). Every server gets the URLs of the live control servers on it ([Control URLs](#control-urls)). Today, `node`, `dns`, and `edge` are the roles that enroll.

## ControlService

| Procedure | Kind | Description |
| --- | --- | --- |
| `/hopbox.cluster.v1.ControlService/Channel` | bidi stream | The control channel. |
| `/hopbox.cluster.v1.ControlService/AttachStream` | bidi stream | One data stream that the control role asked for. |

Both calls need the certificate of a server. The policy lets a server act only for itself ([the API server](/reference/api/index.html.md#callers-and-policy)):

| Action | Operation | Allowed caller |
| --- | --- | --- |
| `server.channel` | open `Channel` | the server that the certificate names |
| `server.heartbeat` | record a heartbeat | the same server |
| `server.renew_certificate` | sign a new certificate | the same server |
| `server.attach_stream` | open `AttachStream` | the server of which the control role asked the stream |
| `server.read_dns_zones` | read the zones that the server serves, before a push of `dns_zones` | the same server, with a certificate that carries the role `dns` |
| `server.read_mesh_peers` | read the mesh as the server sees it, before a push of `mesh_peers` | the same server |
| `server.read_control_urls` | read the internal listeners of the live control processes on the vRack, before a push of `control_urls` | the same server |
| `server.read_disk_transfers` | read the part of the server in the cold moves, before a push of `disk_transfers` | the same server, with a certificate that carries the role `node` |
| `server.read_edge_certificate` | read the certificate of the install with its key, before a push of `edge_certificate` | the same server, with a certificate that carries the role `edge` |
| `server.read_edge_routes` | read the routes of the machine URLs, before a push of `edge_routes` | the same server, with a certificate that carries the role `edge` |

### ChannelRequest

Each message from the server holds one of these fields:

| Field | Type | When |
| --- | --- | --- |
| `heartbeat` | `Heartbeat` | The first message of each channel, then one every 5 seconds. |
| `certificate_renewal` | `CertificateRenewal` | Once the certificate is 20 days old. |
| `stream_refused` | `StreamRefused` | The answer to an `OpenStream` that the server does not serve. |
| `machine_report` | `MachineReport` | A step of the current state of a machine, from its executor ([the executor protocol](/reference/executor-protocol/index.html.md#machinereport)). |

### ChannelResponse

Each message from the control role holds one of these fields:

| Field | Type | When |
| --- | --- | --- |
| `heartbeat_ack` | `HeartbeatAck` | The answer to each heartbeat. |
| `certificate_issued` | `CertificateIssued` | The answer to a `CertificateRenewal` that the control role grants. |
| `open_stream` | `OpenStream` | A request for a data stream. |
| `certificate_refused` | `CertificateRefused` | The answer to a `CertificateRenewal` that the control role does not grant now. |
| `dns_zones` | `DNSZones` | Every zone that the DNS role of the server serves. Only a server with the DNS role gets it ([DNS zones](#dns-zones)). |
| `reconcile_machine` | `ReconcileMachine` | What a machine of the server must become, with the lease of its executor ([the executor protocol](/reference/executor-protocol/index.html.md#reconcilemachine)). |
| `machine_report_result` | `MachineReportResult` | The answer to each `MachineReport` ([the executor protocol](/reference/executor-protocol/index.html.md#machinereportresult)). |
| `edge_certificate` | `EdgeCertificate` | The certificate of the install with its key. Only a server with the edge role gets it ([Edge pushes](#edge-pushes)). |
| `edge_routes` | `EdgeRoutes` | Every machine that a machine URL can name, with the target of each running one. Only a server with the edge role gets it. |
| `edge_lease_renewals` | `EdgeLeaseRenewals` | The rest of the lease of each target, in place of routes that stay the same. Only a server with the edge role gets it. |
| `edge_access_changed` | `EdgeAccessChanged` | A change of the access to machines. Only a server with the edge role gets it. |
| `edge_ssh_keys` | `EdgeSSHKeys` | The keys of the SSH front door. Only a server with the edge role gets it. |
| `mesh_peers` | `MeshPeers` | The mesh address of the server and the other servers of the mesh. Only a server whose heartbeat holds `mesh` gets it ([Mesh pushes](#mesh-pushes)). |
| `disk_transfers` | `DiskTransfers` | The disks that the node serves to other nodes, and the machines whose old files it removes. Only a server whose heartbeat says `disk_transfer` gets it ([Disk transfers](#disk-transfers)). |
| `control_urls` | `ControlURLs` | The internal listeners of the live control processes on the vRack. Every server gets it ([Control URLs](#control-urls)). |

A message without a known field ends the channel with `invalid_argument`. A server that gets a message that it does not know logs `the control plane sent a message that this hopboxd does not know. Upgrade hopboxd` and goes on.

### Messages of two releases

An install can run release N and release N-1 at the same time ([the bundled Postgres](/reference/postgres/index.html.md#two-releases-on-one-store)). A rolling upgrade moves the control servers first, then the edge and DNS servers, then the nodes. So during the upgrade, a control role of N serves the channels of servers of N-1. A rollback of one server can also put a server of N on a control role of N-1. These rules keep the channel working in both cases:

- A message keeps the number and the type of each field. A new field gets a new number. `make breaking` checks this against `main` ([API definitions](/reference/api-definitions/index.html.md)).
- Each release reads every message of the release before it. A field that N-1 does not send has its zero value, and N reads that value as N-1 meant it.
- A new field of a message that N-1 reads must not change what N-1 does with the other fields, because N-1 drops the new field.
- A new field of `ChannelRequest` comes from servers only in the release after the one whose control role reads it. A control role of N-1 ends each channel that sends a message without a field that it knows.
- A new field of `ChannelResponse` reaches the servers of N-1, which drop it with a warning. So the control role sends it only when the drop does no harm, or only to a server whose `hopboxd_version` of the [heartbeat](#heartbeat) knows the field.
- A new value of an enum is the same: a release that does not know it reads it as the zero value. The control role sends a new value only to a server of a release that knows it.

### Heartbeat

The heartbeat holds these values.

| Field | Type | Description | Limit |
| --- | --- | --- | --- |
| `status` | `ServerStatus` | `SERVER_STATUS_OK` or `SERVER_STATUS_DEGRADED`. | `SERVER_STATUS_UNSPECIFIED` is refused. |
| `problems` | repeated string | The failed checks of the server, one line each, in the form `<check>: <error>`. | at most 32, each at most 1024 bytes of UTF-8 |
| `cpu` | `CPUFingerprint` | The CPU fingerprint. | each string at most 256 bytes |
| `compatibility_class` | string | The compatibility class. | at most 256 bytes |
| `labels` | map of string to string | The labels of the server. | the label rules below |
| `hopboxd_version` | string | The version of `hopboxd`, such as `v0.1.0`. | at most 256 bytes |
| `kernel_release` | string | The release of the host kernel, as `uname -r` prints it. | at most 256 bytes |
| `capacity` | `Capacity` | What the server has for machines. | each value fits a signed 64-bit number. `cpu_threads` fits a signed 32-bit number. |
| `allocations` | repeated `Allocation` | The resources that the running machines of the server hold, from its executors. | at most 4096 |
| `actions` | repeated `ActionInProgress` | The actions that the executors of the server run now, such as `boot` ([the node executor](/reference/executor/index.html.md#metrics)). | at most 4096, each name at most 256 bytes |
| `sequence` | uint64 | Numbers the heartbeats of one channel, from 1. | |
| `executor` | bool | Tells that the server runs machine executors. | |
| `leases` | repeated `MachineLease` | The machine leases that the executors hold. The control role renews them ([the executor protocol](/reference/executor-protocol/index.html.md#renewals)). | at most 4096 |
| `mesh` | `MeshMember` | The part of the server in the mesh: the public key of its WireGuard device and its vRack address. It is absent on a server outside the mesh ([the mesh](/reference/mesh/index.html.md#which-servers-join)). | the key has 32 bytes. The address is an IPv4 address in `172.31.0.0/16`. A part that breaks this limit ends no channel: the control role records the heartbeat without it. |
| `dns_zone_serial` | optional uint32 | The serial of the zones that the DNS role of the server serves, also the zones on its disk after a restart ([the DNS role](/reference/dns/index.html.md#zones-on-the-disk)). It is absent on a server without the DNS role, and before its first zones. The control role records it in `servers.dns_zone_serial`. A heartbeat without it clears the column. | none |
| `inbound_machines` | repeated string | The IDs of the machines of the server with inbound traffic since the last heartbeat. The control role sets `last_activity_at` of each machine of the server in this list, at most once a minute for each machine ([the node network](/reference/node-network/index.html.md#inbound-traffic-as-a-use)). The IDs of other servers change nothing. | at most 4096, each a UUID |
| `disk_transfer` | bool | Tells that the server serves and pulls the disks of the cold moves ([Cold move](/reference/cold-move/index.html.md#two-releases)). The control role records the `hopboxd_version` of the heartbeat with it. | |
| `removed_machines` | repeated string | The IDs of the machines of the last `remove` of `DiskTransfers` whose old files the server removed ([Disk transfers](#disk-transfers)). | at most 4096, each a UUID |
| `egress_exit` | bool | Tells that the server takes the egress of the machines of the other nodes. It is true on an edge server whose egress tunnel `hbe0` is ready ([the egress of the machines](/reference/egress/index.html.md#egress-edges)). The control role records the `hopboxd_version` of the heartbeat with it, in `servers.egress_exit_version`. A server of a release before the egress tunnels never sends it. | |

`CPUFingerprint`:

| Field | Type | Description |
| --- | --- | --- |
| `arch` | string | The CPU architecture, as Go names it, such as `amd64`. |
| `vendor` | string | `vendor_id` of `/proc/cpuinfo`, such as `GenuineIntel`. |
| `family` | uint32 | `cpu family` of `/proc/cpuinfo`. |
| `model` | uint32 | `model` of `/proc/cpuinfo`. |
| `stepping` | uint32 | `stepping` of `/proc/cpuinfo`. |
| `model_name` | string | `model name` of `/proc/cpuinfo`. |
| `microcode` | string | `microcode` of `/proc/cpuinfo`, such as `0xb000040`. |
| `flags_hash` | string | The SHA-256, in lowercase hex, of the CPU flags, sorted and joined by spaces. |
| `cpu_template` | string | The Firecracker CPU template. Always `none` in phase 1. |

`Capacity`:

| Field | Type | Description |
| --- | --- | --- |
| `cpu_threads` | uint32 | The CPU threads that the kernel runs, from `/sys/devices/system/cpu/online`. A CPU affinity of `hopboxd` does not change the count. |
| `memory_bytes` | uint64 | `MemTotal` of `/proc/meminfo`. |
| `disk_bytes` | uint64 | The size of the file system of `data_dir`. |
| `disk_free_bytes` | uint64 | The space on the file system of `data_dir` that a user without root can use. |

`Allocation`:

| Field | Type | Description |
| --- | --- | --- |
| `machine_id` | string | The ID of the machine. It must be a UUID. |
| `vcpus` | uint32 | Its vCPUs. |
| `memory_bytes` | uint64 | Its memory. |
| `disk_bytes` | uint64 | Its disk. |

`ActionInProgress`:

| Field | Type | Description |
| --- | --- | --- |
| `machine_id` | string | The ID of the machine. It must be a UUID. |
| `action` | string | The name of the action, such as `start`. |
| `start_time` | timestamp | The start of the action. |

A label key has 1 to 63 characters of `a-z`, `0-9`, `.`, `_`, and `-`, and starts with a letter. A label value is UTF-8 of at most 255 bytes without control characters. A server has at most 64 labels.

A heartbeat that breaks a limit ends the channel with `invalid_argument` and a message such as `invalid argument: the heartbeat: 33 problems, more than 32`. The registry keeps the last valid heartbeat.

### HeartbeatAck

| Field | Type | Description |
| --- | --- | --- |
| `state` | `ServerState` | The state of the server after the heartbeat: `SERVER_STATE_READY`, `SERVER_STATE_CORDONED`, `SERVER_STATE_UNRESPONSIVE`, `SERVER_STATE_DRAINING`, `SERVER_STATE_SHUTTING_DOWN`, or `SERVER_STATE_DECOMMISSIONED`. |
| `sequence` | uint64 | The heartbeat that the control role recorded last on the channel. |
| `renewed` | repeated `MachineLease` | The leases of that heartbeat that the control role renewed. The other leases of that heartbeat are stale ([the executor protocol](/reference/executor-protocol/index.html.md#renewals)). |

### CertificateRenewal and CertificateIssued

| Message | Field | Type | Description |
| --- | --- | --- | --- |
| `CertificateRenewal` | `csr` | bytes | A DER certificate request for the ECDSA P-256 key of the renewal. |
| `CertificateIssued` | `certificate` | bytes | The new DER certificate. It names the same server and roles. |
| `CertificateRefused` | `reason` | string | Why the control role grants no certificate now. |
| `CertificateRefused` | `retry_time` | `google.protobuf.Timestamp` | The earliest time of the next renewal. |

### OpenStream and StreamRefused

| Message | Field | Type | Description |
| --- | --- | --- | --- |
| `OpenStream` | `stream_id` | string | A random ID of the stream, 26 characters. The server sends it back in its first `AttachStreamRequest`. |
| `OpenStream` | `kind` | string | What the stream carries, and so which part of the server serves it. |
| `StreamRefused` | `stream_id` | string | The ID of the refused stream. |
| `StreamRefused` | `reason` | string | Why the server refuses, such as `this server serves no stream of kind "exec"`. |

### DNSZones, DNSZone, and DNSRecord

`DNSZones` is the whole set of zones that the DNS role serves. Each set replaces the set before it.

| Message | Field | Type | Description |
| --- | --- | --- | --- |
| `DNSZones` | `serial` | uint32 | The SOA serial of every zone. It grows with each change ([Control store tables](/reference/control-store/index.html.md#dns_zone)). |
| `DNSZones` | `zones` | repeated `DNSZone` | The zones. Empty until an instance admin sets the domain of the install. |
| `DNSZone` | `origin` | string | The name of the zone, in lowercase, without the final dot, such as `example.com`. |
| `DNSZone` | `records` | repeated `DNSRecord` | Every record of the zone: exactly one `SOA` record and at least one `NS` record at the origin. |
| `DNSRecord` | `name` | string | The owner of the record, relative to the origin, in lowercase: empty for the origin, `*` for the wildcard, or labels such as `ns1` or `_acme-challenge`. |
| `DNSRecord` | `type` | string | The record type: `SOA`, `NS`, `A`, `AAAA`, `TXT`, or `CAA`. |
| `DNSRecord` | `ttl` | uint32 | The time to live of the record, in seconds. |
| `DNSRecord` | `data` | string | The data of the record in the text form of RFC 1035, such as `203.0.113.10` for an `A` record, or `"a value"` for a `TXT` record. A name in the data ends with a dot. |

[The DNS role](/reference/dns/index.html.md#the-zone-of-the-install) lists the records of the zone of the install.

### EdgeCertificate, EdgeRoutes, EdgeRoute, EdgeTarget, EdgeLeaseRenewals, EdgeAccessChanged, and EdgeSSHKeys

These messages feed the edge role ([the edge role](/reference/edge/index.html.md)). Each `EdgeCertificate` replaces the one before it, and each `EdgeRoutes` replaces the set before it. `EdgeService`, the service of the edge role on the internal listener, is in [the internal listener](/reference/internal-listener/index.html.md#edgeservice).

| Message | Field | Type | Description |
| --- | --- | --- | --- |
| `EdgeCertificate` | `domain` | string | The domain of the install, in lowercase, such as `example.com`. The certificate covers it and `*.<domain>`. |
| `EdgeCertificate` | `chain` | repeated bytes | The DER certificates, the leaf first. |
| `EdgeCertificate` | `private_key` | bytes | The PKCS #8 DER key of the leaf. Only the mutual TLS of the channel protects it. |
| `EdgeRoutes` | `domain` | string | The domain of the install, in lowercase. A machine URL is `https://<name>.<domain>`. It is empty until an instance admin sets the domain, and then the set has no route. |
| `EdgeRoutes` | `routes` | repeated `EdgeRoute` | One route for each machine that holds its name and whose target and current state are not `deleted`. |
| `EdgeRoute` | `name` | string | The name of the machine. |
| `EdgeRoute` | `machine_id` | string | The ID of the machine, a UUID. |
| `EdgeRoute` | `public` | bool | The URL visibility of the machine. True means `public`: the URL needs no sign-in. |
| `EdgeRoute` | `current_state` | string | The current state of the machine, such as `running` or `stopped`. |
| `EdgeRoute` | `org_suspended` | bool | The suspension of the org of the machine. True means that an instance admin suspended it. |
| `EdgeRoute` | `ssh_port` | uint32 | The SSH port of the machine on the edge servers, 10000 to 30000 ([the SSH front door](/reference/ssh-front-door/index.html.md)). |
| `EdgeRoute` | `target` | `EdgeTarget` | Set only for a running machine whose route record names the live grant of its lease ([Control store tables](/reference/control-store/index.html.md#route_records)). |
| `EdgeTarget` | `address` | string | The IP address of the machine on its node, from its route record ([the node network](/reference/node-network/index.html.md#where-the-address-shows)). It is empty for a machine without an address. |
| `EdgeTarget` | `url_port` | uint32 | The port in the machine that the URL reaches, 1 to 65535. |
| `EdgeTarget` | `lease` | `MachineLease` | The grant of the machine lease that the route record names ([the executor protocol](/reference/executor-protocol/index.html.md)). |
| `EdgeTarget` | `lease_remaining` | `google.protobuf.Duration` | How long the edge can send traffic to the target, counted from the arrival of the message. See [Edge pushes](#edge-pushes). |
| `EdgeTarget` | `ssh_host_keys` | repeated string | The public SSH host keys of the guest, in the `authorized_keys` format, from the route record. The SSH front door accepts only these keys from the machine. |
| `EdgeLeaseRenewals` | `renewals` | repeated `EdgeLeaseRenewal` | The rest of the lease of each target of the last routes. |
| `EdgeLeaseRenewal` | `lease` | `MachineLease` | The grant of the lease. It extends only the target of the same grant. |
| `EdgeLeaseRenewal` | `lease_remaining` | `google.protobuf.Duration` | As in `EdgeTarget`. |

`EdgeAccessChanged` has one field, `machine_id`. It names the one machine whose access changed, a UUID. Empty means every machine.

`EdgeSSHKeys` holds the keys of the SSH front door ([the SSH front door](/reference/ssh-front-door/index.html.md#keys)). `host_key` is the PKCS #8 DER Ed25519 host key of the install. `client_key` is the PKCS #8 DER Ed25519 key with which the front door signs in to the machines. Only the mutual TLS of the channel protects them.

### MeshMember, MeshPeers, and MeshPeer

These messages carry the mesh ([the mesh](/reference/mesh/index.html.md)), and `HostFirewallSets` the sets of the host firewall that come with it. Each `MeshPeers` replaces the one before it.

| Message | Field | Type | Description |
| --- | --- | --- | --- |
| `MeshMember` | `public_key` | bytes | The public key of the WireGuard device `hbm0` of the server: 32 bytes. |
| `MeshMember` | `vrack_address` | string | The IPv4 address of the server on the vRack, in `172.31.0.0/16`, such as `172.31.0.12`. |
| `MeshPeers` | `address` | string | The mesh address of the server, in `10.239.0.0/16`, such as `10.239.0.3`. |
| `MeshPeers` | `peers` | repeated `MeshPeer` | The other servers of the mesh that are not decommissioned, in the order of their mesh addresses. At most 4096. |
| `MeshPeers` | `egress_marked` | bool | True from a control role that sets `MeshPeer.egress`, from N17 on. A control role of an older release never sets it. A node then keeps the egress tunnels that it has, and does not read the missing `egress` as no egress edge ([the egress of the machines](/reference/egress/index.html.md#a-control-role-of-an-older-release)). |
| `MeshPeers` | `firewall` | `HostFirewallSets` | The sets of the host firewall of the server, from N27 on ([Mesh pushes](#mesh-pushes)). A control role of an older release sends none. The server then keeps the sets that it has. |
| `HostFirewallSets` | `peers` | repeated string | The peer addresses of the servers of the install that are not decommissioned, sorted, such as `203.0.113.12`. At most 4096. |
| `HostFirewallSets` | `jump_hosts` | repeated string | The peer addresses of the edge servers among them, sorted. At most 4096. |
| `MeshPeer` | `server_id` | string | The ID of the server, a UUID. |
| `MeshPeer` | `public_key` | bytes | The public key of its WireGuard device: 32 bytes. |
| `MeshPeer` | `endpoint` | string | Its vRack address and the UDP port `51820`, such as `172.31.0.12:51820`. |
| `MeshPeer` | `allowed_ips` | repeated string | The prefixes that WireGuard sends to this peer, and takes from it ([the mesh](/reference/mesh/index.html.md#allowed-ips)). |
| `MeshPeer` | `roles` | repeated string | The roles of the peer, sorted, such as `node` or `edge`. |
| `MeshPeer` | `egress` | bool | True for an egress edge: a peer with the edge role and the state `ready`, `cordoned`, or `draining`, whose `egress_exit_version` equals its `hopboxd_version` ([the egress of the machines](/reference/egress/index.html.md#egress-edges)). A node without the edge role makes one egress tunnel to each such peer, in the order of the peers. |

### DiskTransfers and DiskExport

These messages carry the part of a node in the cold moves ([Cold move](/reference/cold-move/index.html.md)). Each `DiskTransfers` replaces the one before it.

| Message | Field | Type | Description |
| --- | --- | --- | --- |
| `DiskTransfers` | `serve` | repeated `DiskExport` | The disks that the node lets another node pull: those of the moves in `pulling` whose source is the node. At most 4096. |
| `DiskTransfers` | `remove` | repeated string | The IDs of the machines that left the node, whose old files it removes. At most 4096. |
| `DiskExport` | `machine_id` | string | The ID of the machine, a UUID. |
| `DiskExport` | `target_server_id` | string | The ID of the one node that can pull the disk, a UUID. |
| `DiskExport` | `target_serials` | repeated string | The serials of the certificates of that node that are not retired and not expired. The disk listener serves the disk only to a client certificate with one of these serials. |

### ControlURLs

This message carries the control servers that a server dials ([Control URLs](#control-urls)). Each `ControlURLs` replaces the one before it.

| Message | Field | Type | Description |
| --- | --- | --- | --- |
| `ControlURLs` | `urls` | repeated string | The internal listeners of the live control processes, each `https://<address>:<port>` with an address in the vRack range `172.31.0.0/16`, such as `https://172.31.0.11:7443`. Sorted, none twice, 1 to 8 of them. |

### AttachStreamRequest and AttachStreamResponse

| Message | Field | Type | Description |
| --- | --- | --- | --- |
| `AttachStreamRequest` | `stream_id` | string | The ID from `OpenStream` in the first message. Empty after it. |
| `AttachStreamRequest` | `data` | bytes | The next bytes from the server, at most 64 KiB. The first message can carry bytes too. |
| `AttachStreamRequest` | `acks` | bool | True in the first message of a server that reports `read`. The control role then keeps at most 128 KiB of the stream in flight ([Credit of a data stream](#credit-of-a-data-stream)). |
| `AttachStreamRequest` | `read` | uint64 | The cost of the messages of the control role that the server took off the stream since its last report: the bytes of their data, and 64 for each message. A message with only `read` carries no data. |
| `AttachStreamResponse` | `data` | bytes | The next bytes from the control role, at most 64 KiB. |

## The heartbeat

At the start of the channel, the server sends a heartbeat. Then it sends one every 5 seconds. The control role writes the facts to the server registry in the control store ([Control store tables](/reference/control-store/index.html.md#servers)) and answers with `HeartbeatAck`. The admin calls `ListServers` and `GetServer` show the facts ([the admin API](/reference/admin-api/index.html.md#serverservice)).

The control role writes at most one heartbeat of a channel every 2.5 seconds, half the interval. A heartbeat that comes sooner gets the answer of the last one, with its state, and no write. So a server that sends too fast cannot load the control store.

Sometimes the write of a heartbeat fails, for example because the control store does not answer for a moment. Then the control role logs `recording a heartbeat failed` at level warn, with the fields `server` and `err`. It sends no `HeartbeatAck`, and keeps the channel open. The next heartbeat tries again. So a short outage of the control store does not drop the channels of the whole fleet. Only a refused heartbeat ends the channel: one with a bad field (`invalid_argument`), or one of a server that the registry no longer serves (`unauthenticated`).

On a server, the `Collector` of `internal/node` gathers the facts. It reads some facts once, at the first heartbeat of the process, and others at each heartbeat:

| Fact | Source | When |
| --- | --- | --- |
| `problems` from the checks | the preflight check ([the preflight check](/reference/preflight/index.html.md)): one line `<check>: <error>` for each failed check, of either severity | once |
| `status` | A failed check of severity `fail` gives `SERVER_STATUS_DEGRADED`. Else the status is `SERVER_STATUS_OK`. A warning alone, such as a kernel that is not the tested one, leaves the status `ok`. | once |
| `kernel_release` | `uname`. A failure adds the problem `host: <error>`. | once |
| `cpu` | the first processor of `/proc/cpuinfo`. A failure adds the problem `cpu: <error>`, and the fingerprint then holds only the architecture and the template `none`. | once |
| `compatibility_class` | the formula below, with the snapshot format version that `firecracker --snapshot-version` prints. The VMM driver of the node runs the Firecracker of `vmm.firecracker` ([the bootstrap config of hopboxd](/reference/hopboxd-config/index.html.md#keys)), for at most 5 seconds ([the Firecracker driver](/reference/firecracker-driver/index.html.md#capabilities)). It never runs a `firecracker` of `PATH`. If it fails, the version is `unknown`, and the problem `firecracker: <error>` appears. | once |
| `capacity` | the CPU threads of the process, `/proc/meminfo`, and `statfs` of `data_dir`. A failure on Linux adds the problem `capacity: <error>`. | each heartbeat |
| `labels` | `channel.labels` of the bootstrap config ([the bootstrap config of hopboxd](/reference/hopboxd-config/index.html.md#keys)). Once `SetServerLabels` of the admin API set the labels of the server, the registry keeps those and ignores this field ([the admin API](/reference/admin-api/index.html.md#setserverlabels)). | each heartbeat |
| `hopboxd_version` | the version of `hopboxd` | each heartbeat |
| `allocations`, `actions`, `leases`, `executor` | the executors of the node ([the node executor](/reference/executor/index.html.md)). A server with the node role runs the executors. A server without it sends empty lists and `executor` false. | each heartbeat |
| `inbound_machines` | the inbound counters of the table `inet hopbox_node` ([the node network](/reference/node-network/index.html.md#counters)). A server without the node role sends an empty list. | each heartbeat |
| `disk_transfer`, `removed_machines` | the disk listener and the executors of the node ([Cold move](/reference/cold-move/index.html.md)). A node whose driver keeps no disk in files, and a server without the node role, send false and an empty list. | each heartbeat |
| `dns_zone_serial` | the serial of the zones that the DNS role serves, the value of `hopbox_dns_zone_serial` ([the DNS role](/reference/dns/index.html.md#metrics)). A server without the DNS role, or without zones, leaves it out. | each heartbeat |
| `egress_exit` | the last check of the egress tunnel `hbe0` of an edge server ([the egress of the machines](/reference/egress/index.html.md#egress-edges)). A server without the edge role sends false. | each heartbeat |

A server without the node role runs no machine. So it skips the preflight checks and the snapshot version. Its `status` is always `SERVER_STATUS_OK`, its `problems` come only from the host, the CPU, and the capacity, and its `compatibility_class` is empty.

Off Linux, the CPU, memory, and disk facts are not available. The capacity then stays 0 without a problem line, and the preflight problems already say why.

### Compatibility class

The compatibility class names the servers that can restore the memory state of each other. It is `cc-` and 16 lowercase hex digits:

```text
cc-<first 8 bytes of SHA-256(arch NUL model NUL kernel_series NUL snapshot_version), in hex>
```

| Part | Value |
| --- | --- |
| `arch` | The CPU architecture, such as `amd64`. |
| `model` | The CPU template. For the template `none`, it is `<vendor>/<family>/<model>/<stepping>`, such as `GenuineIntel/6/79/1`. The model name and the microcode do not count. |
| `kernel_series` | The first two numbers of the kernel release, such as `6.18` for `6.18.57-hopbox1`. |
| `snapshot_version` | The output of `firecracker --snapshot-version`, such as `v12.0.0`, or `unknown`. |

## Silence

Both sides watch the channel:

| Side | Rule | Then |
| --- | --- | --- |
| Control role | No message for 20 seconds, which is 4 heartbeat intervals. | It ends the channel with `deadline_exceeded`, and logs `control channel closed` with the error `the server sent nothing for 4 heartbeat intervals`. |
| Server | No `HeartbeatAck` for 4 heartbeat intervals. | It drops the channel with the error `the control plane answered no heartbeat for 4 intervals`, and dials again. |

A new channel of a server replaces the open one of the same server. The control role ends the old channel with `aborted`, and the cause `a newer channel of the same server replaced this one` goes to its log.

When the control side ends a channel, the server gets the code and the cause: `deadline_exceeded` with `the server sent nothing for 4 heartbeat intervals`, `aborted` with `a newer channel of the same server replaced this one`, `permission_denied` with `the control plane ended the channel of this server` after `Hub.Disconnect`, `unavailable` with `the install went into maintenance mode, which ended this call. Retry later` ([Maintenance mode](#maintenance-mode)), or `invalid_argument` with `the message has no known content. Upgrade hopboxd on the control servers`. An error of the service layer, such as a refused heartbeat, keeps its code and message as in [the API server](/reference/api/index.html.md#errors).

## Maintenance mode

In maintenance mode, the control role serves no control channel ([the API server](/reference/api/index.html.md#maintenance-mode)):

- When maintenance mode turns on, each control process ends every open channel and every data stream with `unavailable` and the message `the install went into maintenance mode, which ended this call. Retry later`. `SetMaintenance` returns only after each channel of every live control process ended, so no heartbeat, report, or lease renewal comes after it. The other control processes report this in their rows of `control_processes` ([the admin API](/reference/admin-api/index.html.md#setmaintenance)).
- A new `Channel` or `AttachStream` gets `unavailable` with the message `the install is in maintenance mode, so it takes only reads. Retry later`, and the headers `Hopbox-Not-Run: 1` and `Retry-After: 30`. The server dials again with its backoff of up to 30 s ([The agent](#the-agent)).
- The calls of `EdgeService` and `EnrollmentService` get the same refusal. So the edge role cannot sign anyone in to a machine URL or to the SSH front door.

The machines keep running on their nodes. Their leases are not renewed, so 30 s later each executor holds no valid lease and takes no action, but its microVM keeps running ([the node executor](/reference/executor/index.html.md)). The edge role counts the lease of each target from its last routes, so within 30 s each machine URL answers `503 Service Unavailable` ([the edge role](/reference/edge/index.html.md#routing)). The leader does not mark the silent servers `unresponsive` in maintenance mode ([hopboxd](/reference/hopboxd/index.html.md#leader-jobs)). When maintenance mode turns off, the servers dial again. Their heartbeats bring new grants of the leases, with new epochs, and the routes of the edge servers come back. The leader gives the servers 20 s to dial before its first check of silent servers.

## Server states

A server is in one of these states. The drain of a node sets `draining` and `shutting_down` ([Cold move](/reference/cold-move/index.html.md#drain-of-a-node)). Both states exist in `hopbox.admin.v1.ServerState` and in `hopbox.cluster.v1.ServerState` as `SERVER_STATE_DRAINING = 5` and `SERVER_STATE_SHUTTING_DOWN = 6`.

| State | Meaning |
| --- | --- |
| `ready` | The server sends heartbeats and takes new machines. |
| `cordoned` | An instance admin cordoned it. It keeps its machines but takes no new one. |
| `unresponsive` | It sent no heartbeat for 20 seconds, or none since its enrollment. Its machines stay on it. |
| `draining` | A drain cordoned it, and moves its machines to other nodes. It takes no new machine. |
| `shutting_down` | A drain found it empty. After 3 empty checks in a row, the drain decommissions it. |
| `decommissioned` | It left the install. The control role refuses its certificates, its heartbeats, and its renewals. |

The registry also keeps the flag `cordoned`, which is the choice of the instance admin. The flag stays while the server is unresponsive, so the server comes back cordoned. A server with a drain comes back `draining`, before the cordon rule.

| From | Event | To | Audit entry |
| --- | --- | --- | --- |
| none | `Enroll` | `unresponsive` | `server.enroll` ([Enrollment and the install PKI](/reference/enrollment/index.html.md#enroll)) |
| `unresponsive` | a heartbeat, flag not set | `ready` | `server.heartbeat`, actor `server` |
| `unresponsive` | a heartbeat, flag set | `cordoned` | `server.heartbeat`, actor `server` |
| `ready` | `CordonServer` | `cordoned` | `server.cordon` |
| `cordoned` | `UncordonServer` | `ready` | `server.uncordon` |
| `unresponsive` | `CordonServer` or `UncordonServer` | `unresponsive`, with the flag set or cleared | `server.cordon` or `server.uncordon` |
| `unresponsive` | a heartbeat, the server has a drain | `draining` | `server.heartbeat`, actor `server` |
| `ready` or `cordoned` | `DrainServer` | `draining` | `server.drain` |
| `unresponsive` | `DrainServer` | `unresponsive`, with the flag set. It turns `draining` at its next heartbeat. | `server.drain` |
| `draining` | the first empty check of the drain | `shutting_down` | `server.mark_shutting_down`, actor `system`, details `old_state`, `state`, and `checks` |
| `shutting_down` | a check finds the node not empty | `draining` | `server.mark_draining`, actor `system`, details `old_state`, `state`, and `reason` |
| `shutting_down` | `DrainServer` again | `draining` | `server.drain` |
| `ready`, `cordoned`, `draining`, or `shutting_down` | no heartbeat for 20 seconds, found by the leader job | `unresponsive` | `server.mark_unresponsive`, actor `system` |
| `shutting_down` | the third empty check of the drain | `decommissioned` | `server.decommission`, actor `system`, details `old_state` `shutting_down`, `state` `decommissioned`, and `drain` `true` |
| `ready`, `cordoned`, `unresponsive`, `draining`, or `shutting_down` | `DecommissionServer` | `decommissioned` | `server.decommission` |

Only a heartbeat that changes the state writes the audit entry `server.heartbeat`. Its details are `old_state` and `state`. The entries of `CordonServer` and `UncordonServer` have the same details ([the admin API](/reference/admin-api/index.html.md#serverservice)). `DecommissionServer` of the admin API sets `decommissioned` ([the admin API](/reference/admin-api/index.html.md#decommissionserver)). When Terraform destroys a server, the hopbox Terraform provider calls `DrainServer`, and the drain decommissions the server.

### The leader job

The leader of the control role checks the heartbeats every 5 seconds ([hopboxd](/reference/hopboxd/index.html.md#leader-jobs)). The checks of the first 20 seconds of each term do nothing, and so do those of the first 20 seconds after maintenance mode. In that time, the servers can dial again after a stop of the control role. In maintenance mode, the job does not run ([Maintenance mode](#maintenance-mode)). Each check marks unresponsive every `ready`, `cordoned`, `draining`, or `shutting_down` server whose last heartbeat is 20 seconds old or older.

The write is fenced with the leader lease ([Control store tables](/reference/control-store/index.html.md#fenced-writes)). So a leader that lost its lease changes nothing, and its check fails with `lease: the lease epoch is stale or expired`.

For each server that it marks, the job writes the audit entry `server.mark_unresponsive`. The entry has the actor `system`, the target `server`, and the detail `last_heartbeat_at` in RFC 3339. The service logs `a server is unresponsive` at level warn, with `component=service` and the fields `server` and `after`.

All times come from the clock of Postgres. A server that has never sent a heartbeat stays `unresponsive` and is never marked again.

## Certificate renewal

A server certificate lives 30 days ([Enrollment and the install PKI](/reference/enrollment/index.html.md#server-certificate)). The certificate becomes due for renewal at the age of 20 days. At the next heartbeat tick, the server makes a key for the renewal and sends `CertificateRenewal` with a request for it. It sends at most one renewal at a time. It keeps that key, also across channels and drops, until it installs a certificate for it, so each new try asks for the same key.

The control role locks the row of the server, so two renewals of one server run one after the other. A renewal from a channel whose certificate a renewal retired fails with `unauthenticated`, which ends the channel. Then the control role takes the first step that fits:

1. A certificate that an earlier renewal issued for the same key, and that the server never used, comes back as it is. The answer to that renewal got lost. The service logs `sent a pending server certificate again` with `server` and `serial`. This needs no new certificate, so the limit below does not apply.
2. Within 1 hour after the newest certificate of the server, the control role grants no new one. It answers `CertificateRefused` with `retry_time` at the end of that hour, and keeps the channel open.
3. Otherwise it signs a new certificate with the same server ID and the roles of the registry.

For a new certificate, the control role records the certificate and writes the audit entry `server.renew_certificate` with the details `serial` and `not_after`. Then it sends `CertificateIssued`. The service logs `renewed a server certificate` with `component=service` and the fields `server` and `serial`.

The server checks that the CA signed the certificate, that it holds the new key, and that it names the same server. Then it saves the new identity in `<data_dir>/server` ([Enrollment and the install PKI](/reference/enrollment/index.html.md#identity-files)) and logs `renewed the server certificate` with the field `not_after`. The open HTTP/2 connection still holds the old certificate, and its data streams fail once that certificate expires. So the server closes the channel at once, logs `dialing the control channel again with the renewed certificate`, and dials again without a wait, with the new certificate.

The redial is the first call with the new certificate, which retires every other certificate of the server ([Enrollment and the install PKI](/reference/enrollment/index.html.md#client-certificates-on-each-call)). So a stolen copy of the old certificate stops working as soon as the server renewed. A channel that an old certificate still holds open in this control process ends at once with `unauthenticated` and the message `a renewal retired the certificate of this channel`. In another control process, it ends at its next heartbeat, which the registry refuses. The redial is the only gap that a renewal makes.

When the control role grants no certificate, it logs `a certificate renewal failed` at level warn, with `server` and `err`, and answers `CertificateRefused`. For the limit of one certificate an hour, `retry_time` is the end of that hour. For any other failure, such as a lost connection to the control store, it is 1 minute later. The channel stays open in both cases. The server keeps its certificate and the key of its renewal. It logs `the control plane refused the certificate renewal` with `reason` and `retry_at`. It asks again at `retry_time`, but not before its next heartbeat. A renewal that gets no answer for 4 heartbeat intervals counts as lost: the server logs `the control plane did not answer the certificate renewal` with `retry_at`, and asks again 1 minute later.

A renewal for a decommissioned server fails with `unauthenticated` and the message `the server is decommissioned or unknown`, which ends the channel. A bad request ends it with `invalid_argument`.

A `CertificateIssued` that the server did not ask for is ignored, with the warning `the control plane sent a certificate that this server did not ask for`.

## DNS zones

The control role sends the zones of the DNS role on the channel of each server whose certificate carries the role `dns` ([the DNS role](/reference/dns/index.html.md#how-the-zones-reach-the-dns-role)). A server without the DNS role gets no zones. The control role sends a `DNSZones` message at these times:

- after the first heartbeat of each `Channel` that the control role records,
- after each change event `dns.zone_changed` ([Control store tables](/reference/control-store/index.html.md#dns_zone)). Each control process reads the events and pushes to the channels that it holds, so a change in any control process reaches every channel.
- at the next recorded heartbeat after a failed read of the zones.

For each push, the control role reads the zones from the control store with the caller of the channel, under the policy action `server.read_dns_zones`. If the read fails, it logs `reading the dns zones of a server failed. The next heartbeat tries again` and marks the zones of the channel as stale. The channel stays open.

A slow channel gets only the newest set. A set that waits to be sent is replaced by a newer one. The control role never sends a set whose serial is not higher than the serial of the last set of the channel. So a server never gets an older set after a newer one.

The agent hands each set to the DNS role of its server. The DNS role checks the whole set before it serves it. If the DNS role refuses a set, the agent logs `the dns role refused the zones of the control plane, and keeps the zones that it has` at level error. A set with a lower serial than the set that the DNS role serves is no fault, because a newer push came first. For it, the agent logs `the dns role keeps newer zones than the zones of the control plane` at level info. The DNS role keeps the zones that it serves, and the channel stays open. The next change sends a new set. A server without the DNS role that gets a set logs `the control plane sent dns zones to a server without the dns role` at level warn, and drops the set.

## Edge pushes

The control role sends five kinds of message on the channel of each server whose certificate carries the role `edge`. A server without the edge role gets none of them. Each channel of an edge server has one worker. A push sets a flag for its kind and wakes the worker. For each flag that is set, the worker then reads and queues one message. The order is the certificate, the keys of the SSH front door, the routes, and the access change. So a burst of changes costs one read of each kind. The messages go out in the order of the queue, after the messages before them.

| Message | The control role sends it |
| --- | --- |
| `edge_certificate` | after the first recorded heartbeat of each `Channel`, and after each change event `certificate.changed` ([the certificate of the install](/reference/acme/index.html.md#where-the-certificate-lives)) |
| `edge_routes` | after the first recorded heartbeat of each `Channel`. If the routes changed, also after each later recorded heartbeat, and after each change event `machine.*` or `dns.zone_changed`. The zone holds the domain. |
| `edge_ssh_keys` | after the first recorded heartbeat of each `Channel`, or after the first heartbeat at which the keys exist. The keys never change. |
| `edge_lease_renewals` | in place of `edge_routes`, for routes that stay the same since the last `edge_routes` of the channel |
| `edge_access_changed` | after the first recorded heartbeat of each `Channel`, for every machine. After each change event of a machine, for that machine. After each change event `edge.access_changed`, for every machine ([Control store tables](/reference/control-store/index.html.md#events)). The worker sends one message for each machine that changed since its last send. Past 256 machines, it sends one message for every machine. |

Each control process reads the change events and pushes to its own channels (`Hub.Notify`). So a change in any control process reaches every channel. An edge can miss changes while it has no channel. So the first heartbeat of a channel sends the certificate and the keys of the SSH front door. It also tells the edge to check its private machine URLs and its SSH connections again.

For the certificate, the control role reads the certificate with `Service.EdgeCertificate`, under the policy action `server.read_edge_certificate`. The read fails before the first certificate, and for a stored certificate of an old domain. Then the control role sends nothing. It logs `the edge server gets no certificate yet` at level info, and waits for the next `certificate.changed`. So an edge never gets the certificate of an old domain. Another failed read logs `reading the certificate for an edge server failed. The next heartbeat tries again` at level warn, and the next recorded heartbeat reads it again.

For the keys of the SSH front door, the control role reads them with `Service.EdgeSSHKeys`, under the policy action `server.read_edge_ssh_keys`. If the install has no keys yet, the read makes them. Without a key-encryption key, it logs `the edge server gets no keys of the SSH front door yet. Each heartbeat tries again` at level warn, once for each channel. Each recorded heartbeat then reads the keys again, until they exist. Another failed read logs `reading the keys of the SSH front door for an edge server failed. The next heartbeat tries again` at level warn.

For the routes, the control role reads them with `Service.EdgeRoutes`, under the policy action `server.read_edge_routes`. A failed read logs `reading the routes for an edge server failed. The next heartbeat tries again` at level warn. The channel stays open in each case.

Each heartbeat reads the routes again. The worker keeps the SHA-256 of the last routes that it sent on the channel, without the rest of each lease. If the routes did not change, it sends only `edge_lease_renewals`: the grant and the rest of the lease of each target. So a heartbeat costs a few bytes for each running machine, not the routes of the install. The edge acknowledges no routes, and can refuse a set. So after 11 sends of renewals, the worker sends the whole routes again: about once a minute. Either way, the lease of each target stays fresh at the edge. The machine leases last 30 s, and the heartbeats of the nodes renew them ([the executor protocol](/reference/executor-protocol/index.html.md)). At the send of either message, the control role takes two times from the lease of each target: the time since the read, and a tenth of the machine TTL, 3 s. So `lease_remaining` is the rest of the lease at the send, less 3 s, and at least 0. The edge counts it from the arrival on its own clock, and sends no traffic to the target after it. The clocks of the servers need not agree. A node can stop its renewals of the lease of a machine. Then the traffic of the machine stops at the edge within the lease, also without a change event.

The agent hands each message to the edge role of its server (`node.Agent.Edge`). The edge role checks each one before it uses it ([the edge role](/reference/edge/index.html.md)). If the edge role refuses a message, the agent logs `the edge role refused a push of the control plane, and keeps what it has` at level error. The fields are `push` and `err`. The push is `certificate`, `routes`, `lease renewals`, `access change`, or `SSH keys`. The channel stays open. A server without the edge role that gets one of these messages logs `the control plane sent edge data to a server without the edge role` at level warn, with `push`, and drops it.

## Mesh pushes

A server joins the mesh with the field `mesh` of its heartbeats ([the mesh](/reference/mesh/index.html.md#how-a-server-joins)). The control role records that field in the transaction of the heartbeat. After each heartbeat with `mesh` that it records, the control role reads the mesh as that server sees it, under the policy action `server.read_mesh_peers`. The read gives the mesh address of the server, and every other server of the mesh that is not decommissioned. It also gives the sets of the host firewall of the server in `firewall`: the peer addresses of the servers that are not decommissioned, and those of the edge servers among them ([Control store tables](/reference/control-store/index.html.md#servers)). A change of a peer address or of the edge servers changes the mesh too, so it reaches each server within 5 s. The control role keeps the SHA-256 of the last mesh that it sent on the channel. If the mesh differs, it sends `mesh_peers`. Otherwise, it sends nothing. So the first heartbeat of each channel sends the mesh, and a later heartbeat sends it only after a change.

So a change of one server reaches each other server at the next recorded heartbeat of that server, within 5 s. Each control process reads the store, so this holds with several control processes too. One read runs at a time for each channel. A heartbeat that finds a read running skips its own read, and the next heartbeat reads again. A failed read logs `reading the mesh for a server failed. The next heartbeat tries again` at level warn. The channel stays open.

The agent first hands the sets of `firewall` to the host firewall of its server (`node.Agent.FirewallSets`), so a device that refuses the mesh never holds the sets back. If the firewall refuses the sets, the agent logs `the host firewall refused the sets of the control plane, and keeps the sets that it has` at level error, with the fields `peers`, `jump_hosts`, and `err`. A mesh without `firewall` changes no set. Then the agent hands the mesh to the mesh device of its server (`node.Agent.Mesh`). If the device refuses a mesh, the agent logs `the mesh device refused the mesh of the control plane, and keeps the peers that it has` at level error, with the fields `address`, `peers`, and `err` ([the mesh](/reference/mesh/index.html.md#refused-mesh)). The channel stays open. A server without the device that gets a mesh logs `the control plane sent the mesh to a server outside the mesh` at level warn, and drops it.

A heartbeat whose `mesh` holds the key of another server of the mesh is recorded without its part of the mesh. A full mesh range has the same effect on the heartbeat of a new server. The control role then logs `the server stays out of the mesh, or keeps its old key and vRack address there` at level error, with `component=service` and the fields `server` and `err`.

A heartbeat whose `mesh` breaks its limit is recorded without its part of the mesh too, and the channel stays open. So a server of another release with another rule keeps its channel. The control role logs `the heartbeat has a bad part of the mesh, so the control plane records it without that part` at level warn, with `component=service` and the fields `server` and `err`.

## Disk transfers

A node takes part in the cold moves with the field `disk_transfer` of its heartbeats ([Cold move](/reference/cold-move/index.html.md)). The control role records such a heartbeat. Then it reads the part of the node in the moves, under the policy action `server.read_disk_transfers`. It keeps the SHA-256 of the last `DiskTransfers` that it sent on the channel, and sends `disk_transfers` only after a change, as for the mesh. So the first heartbeat of each channel sends it, and a new step of a move reaches the node within 5 s. A failed read logs `reading the moves of a node failed. The next heartbeat tries again` at level warn. The channel stays open.

The agent hands each `DiskTransfers` to the node role (`node.Agent.Moves`). The disk listener takes `serve`. It ends the streams of each disk that the new message no longer lists, or lists with another node or other serials. The executors take `remove`. The heartbeats then name each machine of `remove` whose files the node removed in `removed_machines`, until a later `DiskTransfers` names it no more. The control role records the removal in the transaction of the heartbeat. A server that takes part in no move logs `the control plane sent the disks of cold moves to a server that takes part in no move` at level warn, and drops the message.

## Control URLs

A server dials the control servers of `channel.control_urls` ([the bootstrap config of hopboxd](/reference/hopboxd-config/index.html.md#keys)). That list is fixed at the install of the server. The control servers of the install change later: a new one starts, an old one leaves. So the control role also sends the URLs of the live control servers on the channel.

After each heartbeat that it records, the control role reads the live rows of `control_processes` ([Control store tables](/reference/control-store/index.html.md)) under the policy action `server.read_control_urls`. It keeps each internal listener whose URL is `https` and whose address is in the vRack range `172.31.0.0/16`. A URL without a port gets port 443, such as `https://172.31.0.22:443` for `https://172.31.0.22`. A listener on loopback serves only its own server, so the list leaves it out. The list is sorted, has no URL twice, and holds at most 8 URLs. The control role keeps the SHA-256 of the last list that it sent on the channel, and sends `control_urls` only after a change, as for the mesh. It never sends an empty list. So the first heartbeat of each channel sends the list, and a control server that starts or stops reaches each server within 5 s. A failed read logs `reading the control URLs for a server failed. The next heartbeat tries again` at level warn. The channel stays open.

The agent takes a list only when each URL is `https://<address>:<port>` with an address in `172.31.0.0/16`, the list has 1 to 8 URLs, and no URL comes twice. Otherwise, it logs `the control plane sent control URLs that this server refuses, and it keeps the URLs that it has` at level error, and keeps the last list that it took. A new list replaces the old one, so a control server that left does not stay in it. The agent writes the list to `<data_dir>/server/control-urls` ([Server layout](/reference/server-layout/index.html.md)), one URL per line, with mode `0600`, through a temp file and a rename. Then it logs `the control plane sent new control URLs` at level info, with the field `urls`. If the write fails, the old file stays. The next start then dials the older list first, because some of its URLs can still be live. The agent logs `cannot keep the control URLs of the control plane. This server dials them until hopboxd stops, and tries the write again at each heartbeat` at level error, once until a write works. It dials the new list until it stops. It writes the file again at each heartbeat, and at the next push, also when that push sends the same list. After a good write, it logs `kept the control URLs of the control plane on the disk after a failed write` at level info, with the field `urls`.

The dial order is the kept list first, because it is the live set, and then each URL of `channel.control_urls` that is not in it. Without a kept list, the dial order is `channel.control_urls`. At its start, the agent reads the kept file. It refuses a file that is not a regular file of the user of `hopboxd`, that group or others can write, that has more than 4096 bytes, or whose list fails the checks of a list from the channel. Then it logs `ignoring the control URLs that this server kept. It dials channel.control_urls until the control plane sends new ones` at level warn, with `err`. The edge role forwards to the control servers in the same order ([the edge role](/reference/edge/index.html.md#forwarded-requests)), and gets each new order from the agent. It spreads its requests over the kept list, or over `channel.control_urls` without one, and tries the other URLs only after them.

A server of the release before this one does not know `control_urls`. It drops the message with the warning `the control plane sent a message that this hopboxd does not know. Upgrade hopboxd`, once for each channel and each change of the list ([Messages of two releases](#messages-of-two-releases)). It keeps dialing `channel.control_urls`.

## Data streams

The control role opens a data stream to a server in these steps:

1. A part of the control role calls `Hub.OpenStream(ctx, server, kind)`.
2. The hub sends `OpenStream` with a new random ID and the kind on the channel of the server.
3. The server looks up its handler of the kind. Without one, it sends `StreamRefused`, and `OpenStream` fails with `channel: the server refused the stream: <reason>`.
4. The server opens `AttachStream` and sends the ID in the first message.
5. The hub checks that a stream waits for the ID, and that the caller is the server of which it asked the stream. Then `OpenStream` returns the stream.

The ID works once. An `AttachStream` with an unknown or spent ID fails with `not_found` and the message `no stream waits for this ID. It expired, or it was never asked for`. An `AttachStream` of another server fails with `permission_denied`, and then `OpenStream` fails at once, because the ID leaked. Without an attach within 10 seconds, `OpenStream` fails with `channel: no attach of the <kind> stream: context deadline exceeded`.

Bytes travel in messages of at most 64 KiB in each direction.

| Side | Read | Write | Close |
| --- | --- | --- | --- |
| Control (`channel.Stream`) | Reads the bytes of the server. Returns `io.EOF` once the server closed its side. | Sends bytes, in chunks of 64 KiB. | Ends the whole stream. The server then reads `io.EOF`. |
| Server (`node.Stream`) | Reads the bytes of the control role. Returns `io.EOF` once the control role closed the stream. | Sends bytes, in chunks of 64 KiB. | `CloseWrite` closes only the side of the server, and reads go on. `Close` ends both sides. |

On each side, one goroutine can read while another writes. After the handler of the server returns, the server closes the stream.

Each control process knows only the channels that it holds. The server dials `AttachStream` at the URL of its channel, so the attach reaches the process that holds the channel. A control process that needs a stream to a server of another process forwards the call to that process ([Streams through another control process](#streams-through-another-control-process)).

### Limits and windows of the data streams

| Item | Value |
| --- | --- |
| Data streams of one server to one control process | at most 255 (`channel.MaxStreams`), open or waiting for their attach. One more fails with `ErrTooManyStreams`: `channel: the server has 255 data streams open, the most that it can have`. |
| HTTP/2 streams of one connection to the internal listener | at most 256 (`machinestream.MaxStreams`): the channel and 255 data streams |
| HTTP/2 receive window of one stream | 256 KiB (`machinestream.StreamWindow`), on the internal listener and in the client of the server |
| HTTP/2 receive window of one connection | 64 MiB (`machinestream.ConnectionWindow`), which is 256 times the window of one stream |
| Compression | none. `ControlService` never compresses a response with gzip, so a window holds at most 256 KiB of the data of an exec. With gzip, the window held a compressed copy, and 256 KiB held the copy of many MiB of zeros. A request in gzip fails with `unimplemented` and the message `unknown compression "gzip"`. The other services of the internal listener still take gzip. |

The window of a connection holds every stream at once. So a data stream whose reader stalls, such as an exec whose client reads no output, fills only its own window. The heartbeats and the other data streams of the server go on. With the default HTTP/2 windows of Go, the internal listener gives each stream 1 MiB and the connection 1 MiB too. Then a few stalled streams can fill the window of the connection. With these windows, `TestSlowClientsStallNoOtherStream` shows that the later stalled streams cannot take all of their data. No test shows a stall of the other streams. `TestInternalListenerWindows` and `TestClientWindows` check the windows that each side announces, and `TestSlowClientsStallNoOtherStream` checks the effect.

A close on one side ends a write that waits for the other side:

- `channel.Stream.Close` on the control side ends the call of `AttachStream`. A `Write` that still waits for flow control fails at once, because the handler sets a write deadline in the past (`api.AbortWrites`).
- `node.Stream.Close` on the server side closes the request side first, so the last bytes go out. While a `Write` waits for the control role, it cancels the call instead, and the `Write` fails at once.

### Credit of a data stream

The HTTP/2 server of Go queues the WINDOW_UPDATE of a stream behind the DATA of the same stream ([golang/go#82615](https://github.com/golang/go/issues/82615)). While the DATA of the control role waits for the window of the server, the control role sends no WINDOW_UPDATE on that stream, although it reads every byte. Then the server can send no more than one window. Sometimes the server stops reading because it waits for its own write. An example is an exec whose command blocks on its output before it reads more input, such as `gzip`. Then the stream stops for good. The connection-level WINDOW_UPDATE frames still go out, so the other streams go on. The credit below fixes only the hop from the control role to the node. On the next hop of an exec, the node is the HTTP/2 server of the guest link, and the guest link has a credit of its own from version 3 on ([Guest link](/reference/guest-link/index.html.md#credit-of-an-exec)). With both, such an exec runs to its end. `TestExecWritesBeforeItReadsThroughEveryHop` checks both hops at once.

So the control role sends on credit to a server that sets `acks` in its first message:

- The control role keeps at most 128 KiB in flight on the stream (`machinestream.AttachCredit`), half of the window of the server. A message costs its bytes of data and 64 more (`machinestream.MessageCost`). A `Write` that needs more waits, and fails with `io.ErrClosedPipe` when the stream ends first.
- The server counts the cost of each message that it takes off the stream. Once the count reaches 32 KiB (`machinestream.AttachAckAfter`), it sends the count in `read`, in a message of its own. A `Read` never waits for this send.
- The control role takes the messages of the server off the stream also while no `Read` waits, so an ack reaches it without a `Read`. It holds one message with data until a `Read` takes it, and the messages behind that one wait. So once the server sent bytes, a `Write` on the control side goes on only while its user keeps reading. For example, a user that writes 1 MiB to an echo and reads only after the `Write` returns stops once the server echoed more than one message. The server has the same rule: its ack waits for a `Write` of its own that waits for the control role. Every user of `channel.Stream` reads while it writes.
- The control role reads the end of the server's side after every byte before it. From then on, it sends without a bound, because no ack comes any more. It also sends no more WINDOW_UPDATE on the stream, so nothing waits for one.

The DATA of the control role never waits for the window of the server, so its WINDOW_UPDATE frames always go out. A server of the release before, which does not set `acks`, gets the bytes without a bound, as before, and its streams can still stop. A control role of the release before reads a message with only `read` as a message without data.

`TestStreamCarriesBothWaysWhileTheServerWrites` checks a server that writes 1 MiB before it reads, while the control role writes 1 MiB. `TestStreamWithoutAcks` checks a server without `acks` and a server that closed its side. `TestStreamManySmallWrites` checks that 4096 one-byte writes cost the same on both sides. `TestStreamCloseEndsAWriteThatWaitsForCredit` and `TestStreamCloseWithUnreadBytes` check that a `Close` ends the call while a `Write` waits for credit, or while a message of the server waits for a `Read`.

## Streams through another control process

An install can run several control processes. A server keeps its channel to one of them, and only that process can open a data stream to the server. The control process that takes an API call for the machine of such a server can be another one. It forwards the data stream (exec, file copy, or logs of machines) to the holder of the channel. The code is in `internal/control/channel/peer.go`.

### The holder

Each recorded heartbeat sets the column `servers.channel_process` to the ID of the control process that recorded it, while that process still holds the channel of the heartbeat ([Control store tables](/reference/control-store/index.html.md#servers)). A heartbeat of a channel that a newer channel of the same server replaced leaves the column as it is, so it never erases the holder that the newer channel recorded. When a channel ends and it was the last one of its server in that process, the hub clears the column, if it still names this process. A stale holder only costs a forward that fails, and the next recorded heartbeat names the new holder.

When `OpenStream` fails with `ErrNotConnected`, the control process reads the holder: the column `channel_process` joined with the live row of `control_processes` ([Control store tables](/reference/control-store/index.html.md#control_processes)). It forwards the stream when the holder is another live process with an `internal_url`. Without a live holder, or when the holder is the process itself, the call fails as before with `ErrNotConnected`. A forward that fails with `ErrNotConnected` shows the user the same message as a server without a channel: `the node of the machine is not connected to the control plane. Retry soon`.

### ControlPeerService

| Procedure | Kind | Description |
| --- | --- | --- |
| `/hopbox.cluster.v1.ControlPeerService/ForwardStream` | bidi stream | One data stream that another control process asked for. |

The service runs on the internal listener ([the internal listener](/reference/internal-listener/index.html.md#controlpeerservice)). It takes no gzip, as `ControlService` does not. Only another control process can call it, under the policy action `control.forward_stream` ([the API server](/reference/api/index.html.md#callers-and-policy)). A server certificate gets `permission_denied`.

The steps of a forwarded stream:

1. The forwarding process sends the first request with `server_id` and `kind`. It waits at most 15 s for the first response (`channel.ForwardOpenTimeout`: the 10 s attach timeout and 5 s more).
2. The holder opens the stream on its own channel, as in [Data streams](#data-streams): `OpenStream`, then the attach of the server. It never forwards again.
3. Once the server attached, the holder answers with a first response without bytes.
4. Then both sides carry the bytes, in messages of at most 64 KiB.

A close on either side ends the whole stream. The holder counts the stream in the limit of 255 data streams of the server ([Limits and windows](#limits-and-windows-of-the-data-streams)).

### ForwardStreamRequest and ForwardStreamResponse

| Message | Field | Type | Description |
| --- | --- | --- | --- |
| `ForwardStreamRequest` | `server_id` | string | The ID of the server of the stream, a UUID. In the first message only. |
| `ForwardStreamRequest` | `kind` | string | The kind of the stream, such as `exec`. In the first message only. |
| `ForwardStreamRequest` | `data` | bytes | The next bytes for the server, at most 64 KiB. |
| `ForwardStreamResponse` | `data` | bytes | The next bytes from the server, at most 64 KiB. The first response has none. |

### Errors of a forward

| Holder error | Code of the call | Forwarding process error |
| --- | --- | --- |
| `ErrNotConnected` | `unavailable` | `ErrNotConnected`, with `through the control process at <url>` in the message |
| `ErrTooManyStreams` | `resource_exhausted` | `ErrTooManyStreams`, with `through the control process at <url>` |
| `ErrStreamRefused` | `failed_precondition` | `ErrStreamRefused`, with `through the control process at <url>` |
| any other failure, such as no attach in time | `aborted` | `channel: forward the <kind> stream to the control process at <url>: <err>` |

A first response that does not come within 15 s fails with the same error as `aborted`, with `context deadline exceeded`.

### The client

A control process dials another one over HTTP/2 with TLS 1.3. It sends the control certificate of its own internal listener as its client certificate ([Enrollment and the install PKI](/reference/enrollment/index.html.md#control-certificate)), and checks that the peer is a control listener of the install CA (`pki.VerifyControl`). The client has the HTTP/2 windows of the client of a server (256 KiB for a stream, 64 MiB for a connection), and a dial timeout and a TLS handshake timeout of 10 s. The listener of the holder resolves the certificate to the caller kind `control` ([the internal listener](/reference/internal-listener/index.html.md#callers)).

## Machine streams

Exec, file copy, and the logs of machines run on data streams ([the machine API](/reference/machine-api/index.html.md#access-calls)). The code is in `internal/machinestream` (the parts that both sides share), `internal/service/machine_access.go` (the control side), and `internal/node/access` (the server side).

| Kind | Call of the machine API | What the server does |
| --- | --- | --- |
| `exec` | `Exec`, and the WebSocket of exec | Runs the command on the guest link of the machine. |
| `upload` | `Upload` | Writes the data into the machine on the guest link. |
| `download` | `Download` | Reads a file or a directory on the guest link. |
| `logs` | `StreamMachineLogs` | Reads the node-local log of the machine ([Machine logs](/reference/machine-logs/index.html.md)). |

### Frames

The bytes of a machine stream are frames. A frame is a varint length and then one protobuf message. The message is a `MachineStreamInput` from the control role, or a `MachineStreamOutput` from the server, both in `proto/hopbox/cluster/v1/machine_stream.proto`. A frame has at most 1 MiB (`machinestream.MaxFrame`). A larger frame ends the stream.

`MachineStreamInput` holds one field of the oneof `message`:

| Field | Type | Description |
| --- | --- | --- |
| `open` | `MachineStreamOpen` | The first frame of each stream. |
| `stdin` | bytes | Input of the command of an exec, at most 32 KiB. |
| `stdin_eof` | bool | No more input for the command. |
| `resize` | `TerminalSize` | A new size of the terminal of an exec. |
| `signal` | string | A signal for the command of an exec, by name. |
| `data` | bytes | The next bytes of an upload, at most 32 KiB. |
| `eof` | bool | The end of the data of an upload. |

`MachineStreamOpen` has `machine_id`, the UUID of the machine, and one field of the oneof `operation`. The operation must match the kind of the stream.

| Operation | Fields |
| --- | --- |
| `exec`, a `MachineExec` | The fields of `ExecStart` of the machine API without the machine. `env` is a repeated `MachineEnvVar`. `timeout` is a `google.protobuf.Duration`. |
| `upload`, a `MachineUpload` | `path`, `archive`, `mode`, `user` |
| `download`, a `MachineDownload` | `path`, `user`, and `max_bytes`, the cap of the data. 0 is no cap. The control role sends 4 MiB. |
| `logs`, a `MachineLogs` | `tail`, `since` (a `google.protobuf.Timestamp`), and `follow` |

`MachineStreamOutput` holds one field of the oneof `message`:

| Field | Type | Description |
| --- | --- | --- |
| `stdout`, `stderr` | bytes | Output of the command of an exec. |
| `exit` | `MachineExecExit` | `code`, `signal`, `error`, and `timed_out`. The last frame of an exec. |
| `error` | `MachineStreamError` | `code`, a `MachineStreamErrorCode`, and `message`, at most 1024 bytes. The last frame of a stream that failed. |
| `info` | `MachineFileInfo` | `archive`, `mode`, and `size`. The first frame of a download. |
| `data` | bytes | The next bytes of a download, at most 32 KiB. |
| `done` | bool | The last frame of an upload, a download, or a logs stream that ended well. |
| `log` | `MachineLogEntry` | `time`, `stream` (a `MachineLogStream`), `data`, and `dropped`. One entry of the log. |
| `synced` | bool | The end of the old entries of a logs stream with `follow`. |

| `MachineStreamErrorCode` | When | Code of the machine API |
| --- | --- | --- |
| `MACHINE_STREAM_ERROR_CODE_UNAVAILABLE` | The guest agent is not connected, or the machine stopped during the call. | `unavailable` |
| `MACHINE_STREAM_ERROR_CODE_FAILED_PRECONDITION` | The server does not run the machine, holds no valid lease of it, or the guest agent did not come up yet. | `failed_precondition` |
| `MACHINE_STREAM_ERROR_CODE_INVALID_ARGUMENT` | The first frame does not open the stream, names no machine, or opens another kind. | `internal`, because the control role sent it |
| `MACHINE_STREAM_ERROR_CODE_GUEST` | The guest agent reported an error, such as a file that does not exist. | `failed_precondition` |
| `MACHINE_STREAM_ERROR_CODE_TOO_LARGE` | A download has more bytes than `max_bytes`. | `failed_precondition` |
| `MACHINE_STREAM_ERROR_CODE_INTERNAL` | Any other failure of the server. | `internal` |

### The steps of a machine stream

1. The control role checks the access of the caller to the machine. It opens the data stream on the node of the machine, and writes `open`.
2. The server waits at most 10 s for `open`. It finds the guest link of the machine through `executor.Manager.Guest`. Each frame of the control role goes to the guest link, and each message of the guest link becomes a frame.
3. The server writes the last frame: `exit`, `done`, or `error`. Then it closes its side, and waits at most 10 s for the control role to close the stream. So the last frame arrives before the end.
4. The control role reads the last frame and closes the stream.

When the control role closes the stream before the last frame, the server reads the end of the stream. It ends the operation at once: the guest agent ends the command, or stops the copy. The timeout of an exec ends the command too, and the server then sends `exit` with the code 124 and `timed_out` true.

Each side reads the next frame only after it handed on the one before. So each hop holds at most the window of its stream, and a slow client slows the command down. The one exception is the input of an exec on the server. It waits in a queue of 64 chunks, 2 MiB, so the server reads a resize or a signal behind it at once. The server then sends the resize or the signal on `Link` of the guest link, outside the call of the exec ([Guest link](/reference/guest-link/index.html.md#versions)). Only a full queue holds a signal back. The server also sends a copy on the call of the exec, at once and always before the next input. The guest agent of a current image knows this copy ([Guest link](/reference/guest-link/index.html.md#exec)). With it, the resize or the signal takes effect before any input that the client sent after it.

### Go API of the hub

The service layer and the units of the control role reach the servers through the hub of the control role, `Role.Hub()` of `internal/control`:

| Function | Description |
| --- | --- |
| `Hub.Connected(server)` | Reports whether the server has an open channel to this process. |
| `Hub.OpenStream(ctx, server, kind)` | Opens a data stream. It fails with `ErrNotConnected` (`channel: the server has no open control channel to this control process`), with `ErrStreamRefused`, with `ErrTooManyStreams`, or after the attach timeout. The caller must close the stream. The control role gives it to the service layer as `Service.MachineStreams`. |
| `Hub.ForwardStream(ctx, bidi)` | Serves `ControlPeerService.ForwardStream` for a stream that another control process forwards ([Streams through another control process](#streams-through-another-control-process)). |
| `channel.Forward(ctx, client, url, server, kind)` | Opens a data stream to a server through the control process at `url`. It fails with `ErrNotConnected`, `ErrTooManyStreams`, or `ErrStreamRefused`, as `Hub.OpenStream` does, or after `ForwardOpenTimeout`. The caller must close the stream. |
| `Hub.DisconnectRetired(server, serial)` | If a certificate other than `serial` opened the channel of the server in this process, ends that channel. The service layer calls it through `Service.CertificatesRetired` once the first use of `serial` retired the other certificates. The server gets `unauthenticated`. |
| `Hub.Dispatch(ctx, machineHub)` | Sends `ReconcileMachine` to the executor of each machine that a change event names, on the channels of this process, until `ctx` ends ([the executor protocol](/reference/executor-protocol/index.html.md#reconcilemachine)). The internal listener of the control role runs it. |
| `Hub.Notify(event)` | Pushes again what a change event changes, to the servers on the channels of this process: the zones after `dns.zone_changed`, the certificate after `certificate.changed`, and the routes and the access of the edge role as in [Edge pushes](#edge-pushes). The `events` unit of the control role calls it for each change event. |
| `Hub.PushDNSZones()` | Sends the zones again to every server with the DNS role on a channel of this process. |
| `Hub.PushEdgeCertificate()`, `Hub.PushEdgeRoutes()`, `Hub.PushEdgeAccessChanged(machine)` | Ask the worker of each channel of an edge server in this process for one push of their kind. |
| `Hub.Disconnect(server, reason)` | Ends the open channel of the server in this process, and so its data streams, and reports whether there was one. The server gets `permission_denied`. `DecommissionServer` calls it on the hub of its own process after the commit of the new state. A channel in another control process ends at its next heartbeat, at most 5 seconds later. The registry refuses the heartbeat of a decommissioned server. A new dial fails too, because the internal listener refuses the certificates of a decommissioned server. |

| Hub field | Default | Description |
| --- | --- | --- |
| `Silence` | 20 s | How long a channel can stay without a message. |
| `AttachTimeout` | 10 s | How long `OpenStream` waits for the attach. |
| `MinHeartbeatGap` | 2.5 s | The shortest time between two heartbeats of a channel that the hub writes. |

On the server, `node.Agent.Handlers` maps each kind to a `StreamHandler`, `func(ctx context.Context, s *node.Stream)`. `access.Handlers.Map()` of `internal/node/access` returns the handlers of the machine streams. A server that runs executors serves them ([the node executor](/reference/executor/index.html.md#exec-file-copy-and-logs)).

## The agent

The `channel` unit of `hopboxd run` runs the agent of `internal/node` ([hopboxd](/reference/hopboxd/index.html.md#run)). It runs on each server with a role other than `control`. The agent takes these steps:

1. It reads the identity from `<data_dir>/server`. Without one, it logs `waiting for the enrollment of this server` once and looks again every 2 seconds. So `hopboxd run` can start before `hopboxd enroll`.
2. It dials one URL of the dial order with its certificate, and checks that the peer is the control role of its install CA. The dial order is the kept list of the control role, then the URLs of `channel.control_urls` that are not in it ([Control URLs](#control-urls)). The first channel goes to the URL that a CRC-32 of the server ID picks among the kept URLs, or among `channel.control_urls` without a kept list. So the channels of a fleet spread over the control servers.
3. It opens `Channel`, sends a heartbeat at once, and then one every 5 seconds.
4. After any drop, it waits and dials again. The wait starts at 1 second and doubles up to 10 seconds. Each wait is a random time between half the wait and the whole wait. A channel that lasted more than 1 minute starts the waits from 1 second again. After a lost or refused channel, the next dial goes to the next URL of the dial order, round the order. If the URL of the lost channel left the order, the next dial goes to the URL that the CRC-32 picks. A renewal of the certificate dials the same URL again. After 20 seconds of silence, the leader marks a server `unresponsive`. The longest wait stays well under that. So a short stop of the control role never shows as one.
5. The control plane can refuse the server with `unauthenticated` or `permission_denied`, for example after a decommission or a retired certificate. Then the agent logs `the control plane refuses this server. hopboxd retries every minute` once at level error, with `err`. After that, it dials only once a minute. The first channel that works ends the slow retries.

When the unit stops, the agent closes the channel. The control role then ends it at once.

## Logs

The control side writes these records with `component=channel` and the field `server`:

| Message | Level | Fields |
| --- | --- | --- |
| `control channel opened` | info | `peer` |
| `control channel closed by the server` | info | |
| `control channel lost` | info | `err` |
| `control channel closed` | warn | `err`. A refused heartbeat, renewal, or message, or the silence. |
| `control channel closed` | info | `err`. The server went away, the listener stopped, a newer channel replaced this one, or `Hub.Disconnect` ended it. |
| `recording a heartbeat failed` | warn | `err`. The channel stays open. |
| `reading the dns zones of a server failed. The next heartbeat tries again` | warn | `err`. The channel stays open. |
| `reading the mesh for a server failed. The next heartbeat tries again` | warn | `err`. The channel stays open. |
| `reading the control URLs for a server failed. The next heartbeat tries again` | warn | `err`. The channel stays open. |
| `the edge server gets no certificate yet` | info | `err`. The install has no certificate, or one of an old domain. |
| `reading the certificate for an edge server failed. The next heartbeat tries again` | warn | `err`. The channel stays open. |
| `reading the routes for an edge server failed. The next heartbeat tries again` | warn | `err`. The channel stays open. |
| `the edge server gets no keys of the SSH front door yet. Each heartbeat tries again` | warn | `err`. The install has no key-encryption key. Once for each channel. |
| `reading the keys of the SSH front door for an edge server failed. The next heartbeat tries again` | warn | `err`. The channel stays open. |
| `the keys of the SSH front door do not encode` | error | `err`. The stored keys do not convert to the message `EdgeSSHKeys`. |
| `the certificate of the install does not encode` | error | `err`. The stored certificate or key does not parse. |
| `refused a machine report` | warn | `machine`, `epoch`, `state`, `outcome`, `err` ([the executor protocol](/reference/executor-protocol/index.html.md#machinereportresult)) |
| `recording a machine report failed` | warn | `machine`, `err`. The executor sends the report again. |
| `sending machine specs failed` | warn | `err`, `retry_in`, and `machines` from the dispatcher or `server` after a heartbeat. The channel stays open. |
| `the channel of a node is busy, so its next heartbeat gets every machine spec` | info | `server` |
| `the env of machines does not resolve, so they get no spec until it does` | warn | `machines`, `err` |
| `held machines need their hold no more` | info | `machines` |
| `retrying held machines failed` | warn | `err` |
| `forgetting the holder of a control channel failed` | warn | `server`, `err`. The last channel of the server in this process ended, and the write that clears `servers.channel_process` failed. A forward to this process fails until the next recorded heartbeat names the new holder. |
| `a certificate renewal failed` | warn | `err`. The server gets `CertificateRefused`, and the channel stays open. |

The server side writes these records with `component=node`:

| Message | Level | Fields |
| --- | --- | --- |
| `waiting for the enrollment of this server` | info | `dir` |
| `cannot read the identity of this server` | error | `err` |
| `control channel connected` | info | `server`, `control`, `state` |
| `control channel lost` | warn | `control` (the URL of the lost channel), `err`, `retry_in`, `next_control` (the URL of the next dial) |
| `the control plane refuses this server. hopboxd retries every minute` | error | `control`, `err`. Once, until a channel works again. |
| `dialing the control channel again with the renewed certificate` | info | |
| `the control plane refused the certificate renewal` | info | `reason`, `retry_at` |
| `the control plane did not answer the certificate renewal` | warn | `retry_at` |
| `renewed the server certificate` | info | `server`, `not_after` |
| `the control plane sent a certificate that this server did not ask for` | warn | `server` |
| `the control plane sent a message that this hopboxd does not know. Upgrade hopboxd` | warn | `server` |
| `the dns role refused the zones of the control plane, and keeps the zones that it has` | error | `server`, `serial`, `err` |
| `the dns role keeps newer zones than the zones of the control plane` | info | `server`, `serial`, `err` |
| `the control plane sent dns zones to a server without the dns role` | warn | `server` |
| `the edge role refused a push of the control plane, and keeps what it has` | error | `server`, `push`, `err` |
| `the control plane sent edge data to a server without the edge role` | warn | `server`, `push` |
| `the mesh device refused the mesh of the control plane, and keeps the peers that it has` | error | `server`, `address`, `peers`, `err` |
| `the control plane sent the mesh to a server outside the mesh` | warn | `server` |
| `the control plane sent new control URLs` | info | `server`, `urls` |
| `the control plane sent control URLs that this server refuses, and it keeps the URLs that it has` | error | `server`, `err` |
| `cannot keep the control URLs of the control plane. This server dials them until hopboxd stops, and tries the write again at each heartbeat` | error | `server`, `err` |
| `kept the control URLs of the control plane on the disk after a failed write` | info | `server`, `urls` |
| `ignoring the control URLs that this server kept. It dials channel.control_urls until the control plane sends new ones` | warn | `err` |
| `cannot refuse a stream` | warn | `stream` (the kind), `err` |
| `cannot attach a stream` | warn | `stream` (the kind), `err` |
| `the control plane sent a machine spec, but this server runs no machine executor` | warn | `server`. Once per channel. |
| `machine stream ended` | info | `component=access`, `stream` (the kind), `machine`, `outcome`, `duration`. The outcome is `ok`, or the name of the code of the error frame. A last frame that did not go out has the outcome `lost`. |
| `a machine stream ended before its open frame` | warn | `component=access`, `stream`, `err` |
| `a resize or a signal of an exec did not reach the guest agent` | warn | `component=access`, `machine`, `err`, `missed`. At most once a minute for each exec ([Guest link](/reference/guest-link/index.html.md#versions)). |

At its end, each call of the channel also gets the access log record `api call` of the internal listener ([the API server](/reference/api/index.html.md#logs)).
