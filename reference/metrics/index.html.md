# Metrics

> The Prometheus metrics listener of hopboxd and every metric that it serves.

`hopboxd` serves Prometheus metrics on a loopback listener. The code is in `internal/obs`, `internal/api`, `internal/audit`, `internal/service`, `internal/placement`, `internal/node/executor`, `internal/node/machines`, `internal/node/netcfg`, `internal/mesh`, and `internal/dns`.

The counter `hopbox_account_creations_total` came with milestone A2, the approval of new accounts (metrics catalog).

## Listener

| Item | Value |
| --- | --- |
| Address | `metrics.listen` of the bootstrap config, by default `127.0.0.1:9464` ([the bootstrap config of hopboxd](/reference/hopboxd-config/index.html.md#keys)). It must be a loopback address. |
| Path | `/metrics`. Every other path gets HTTP status 404. |
| Method | `GET` |
| Format | the Prometheus text format. When the scraper asks for the OpenMetrics format, the listener uses it. |
| Auth | none. Only a process on the server can reach a loopback address. |

The metrics listener is the first unit that `hopboxd run` starts, and the last one that it stops ([hopboxd](/reference/hopboxd/index.html.md#run)). So the metrics cover the start and the stop of every other unit.

Example:

```sh
curl -s http://127.0.0.1:9464/metrics | grep '^hopbox_api_calls_total'
```

```text
hopbox_api_calls_total{code="ok",listener="api",procedure="/hopbox.v1.InstallService/GetInstallInfo"} 3
hopbox_api_calls_total{code="unauthenticated",listener="api",procedure="/hopbox.admin.v1.InstallConfigService/GetInstallConfig"} 1
```

## Metrics of the API

At its end, each API call adds to these metrics ([the API server](/reference/api/index.html.md#the-chain-of-a-call)). A call that the gate or the policy refuses counts too.

| Metric | Type | Labels | Description |
| --- | --- | --- | --- |
| `hopbox_api_calls_total` | counter | `listener`, `procedure`, `code` | The API calls that ended. |
| `hopbox_api_call_duration_seconds` | histogram | `listener`, `procedure`, `code` | The duration of the API calls that ended. A stream counts from its start to its end. |

| Label | Values |
| --- | --- |
| `listener` | `api`: the API listener. `internal`: the cluster API of the internal listener ([the internal listener](/reference/internal-listener/index.html.md)). `edge`: the requests that the edge servers forward to the internal listener ([the internal listener](/reference/internal-listener/index.html.md#forwarded-requests)). `admin_socket`: the admin socket. |
| `procedure` | The procedure of a mounted service, for example `/hopbox.v1.InstallService/GetInstallInfo`. A path that names no service gets 404 and does not count, so the set of values stays bounded. |
| `code` | `ok`, or the Connect error code of the call, for example `unauthenticated` ([the API server](/reference/api/index.html.md#errors)) |

Some requests end before they reach a procedure, so these two metrics never see them. One is a path that names no service (HTTP 404). Another is a request that Connect refuses during its decode, such as a message larger than 4 MiB. The HTTP metric below counts every request, these ones too:

| Metric | Type | Labels | Description |
| --- | --- | --- | --- |
| `hopbox_http_requests_total` | counter | `listener`, `status` | The HTTP requests that ended, by listener and HTTP status, for example `200`, `404`, or `429`. |

The gRPC and gRPC-Web protocols send the code of a call in a trailer, with HTTP status 200. So a gRPC call that Connect refuses while it decodes it counts as `status="200"` in `hopbox_http_requests_total`, and not in `hopbox_api_calls_total`. The Connect protocol sends the code as an HTTP status, so its refusals count with their own status. A WebSocket of exec counts once, at its end, as `status="101"`. It is not in `hopbox_api_calls_total`.

The histogram has the default buckets of the Prometheus Go client: 0.005, 0.01, 0.025, 0.05, 0.1, 0.25, 0.5, 1, 2.5, 5, and 10 seconds.

## Metrics of placement

The control role counts the placements of new machines, so an instance admin sees a shortage of nodes ([Placement](/reference/placement/index.html.md#metrics)).

| Metric | Type | Labels | Description |
| --- | --- | --- | --- |
| `hopbox_placements_total` | counter | `result`, `reason` | The placements of `CreateMachine` ([the machine API](/reference/machine-api/index.html.md#createmachine)). |

| Label | Values |
| --- | --- |
| `result` | `placed` for a create that committed. `no_capacity` for a create that found no node with room ([Placement](/reference/placement/index.html.md#when-no-node-has-room)), or no free SSH port. |
| `reason` | Empty for `placed`. For `no_capacity`, the reason that skipped the most servers, from the order of the filters: `not_node`, `not_ready`, `degraded`, `pool`, `labels`, `class`, `version`, `cpu`, `ram`, or `refused`. A tie goes to the earlier filter. `none` for an install without a server. `ssh_ports` for a full range of SSH ports. |

A create that fails for another cause does not count. Examples are a usage limit and a taken name. A start of a stopped machine does not count either.

```sh
curl -s http://127.0.0.1:9464/metrics | grep '^hopbox_placements_total'
```

```text
hopbox_placements_total{reason="",result="placed"} 4
hopbox_placements_total{reason="ram",result="no_capacity"} 1
```

## Metrics of the accounts

The control role counts the first sign-ins of new people, so an instance admin sees a flood of sign-ups ([Sessions, cookies, and sign-in links](/reference/sessions/index.html.md#sign-up-modes)).

| Metric | Type | Labels | Description |
| --- | --- | --- | --- |
| `hopbox_account_creations_total` | counter | `method`, `state` | The first sign-ins of new people, by sign-in method and by the result. Each control process counts the sign-ins that it serves. |

| Label | Values |
| --- | --- |
| `method` | `email_link`, `github`, `google`, or `login_link` |
| `state` | `pending` for a new pending account, `active` for a new active account, and `refused` for a sign-in that a limit of new accounts refused ([Sessions, cookies, and sign-in links](/reference/sessions/index.html.md#limits-of-new-accounts)). The refusal while an older release runs does not count. |

A sign-in counts after its transaction ends, so a refusal counts once for each try. A sign-in that reaches an account that exists does not count. A first-admin login link counts as `active`.

```sh
curl -s http://127.0.0.1:9464/metrics | grep '^hopbox_account_creations_total'
```

```text
hopbox_account_creations_total{method="github",state="pending"} 2
hopbox_account_creations_total{method="email_link",state="refused"} 1
```

## Metrics of the control role

Each control process counts these events itself, so the sum over the control processes is the count of the install. A count starts from zero at each start of `hopboxd`.

| Metric | Type | Labels | Description |
| --- | --- | --- | --- |
| `hopbox_audit_entries_total` | counter | `action`, `actor_kind` | The entries of the audit log that this control process wrote. An entry counts when its change commits. An entry of a change that rolled back does not count. |
| `hopbox_rate_limited_total` | counter | `limit` | The requests that a rate limit refused ([Sessions, cookies, and sign-in links](/reference/sessions/index.html.md#rate-limits), [Sign-in with GitHub and Google](/reference/oauth/index.html.md#rate-limits), [the org API](/reference/org-api/index.html.md)). |
| `hopbox_limit_refusals_total` | counter | `limit` | The calls that an org limit refused ([Domain rules](/reference/domain-rules/index.html.md#usage-limits)). |
| `hopbox_api_key_creations_total` | counter | `kind`, `lifetime` | The new API keys and admin keys ([CLI tokens, API keys, admin API keys, and SSH keys](/reference/credentials/index.html.md)). A second registration of the hash of an admin key does not count. |

| Label | Values |
| --- | --- |
| `action` | The action of the audit entry, such as `org.create`, `api_key.create`, `machine.create`, or `machine.set_url_port` |
| `actor_kind` | `anonymous`, `session`, `cli_token`, `member_key`, `org_key`, `admin_key`, `local_root`, `server`, `ssh_key`, `control`, or `system` |
| `limit` of `hopbox_rate_limited_total` | `sign_in_links`: the sign-in links of an email or of a client. `device_sign_ins`: the device sign-ins of a client, and the device sign-ins that wait for an approval. `device_codes`: the wrong codes of the device approvals of an account. `oauth_starts`: the sign-ins with GitHub and Google of a client and of the install. `oauth_callbacks`: the callbacks of a client in one control process. `invitations`: the invitations of an org, of an account, and to an email. `machine_url_codes`: the machine URL codes of a browser. `account_creations`: the new accounts of a client address in a day ([Sessions, cookies, and sign-in links](/reference/sessions/index.html.md#limits-of-new-accounts)). `pending_cap`: the first sign-ins that the cap of 50 pending accounts refused. `sign_up_notes`: the changes of the note of a pending account in an hour ([the sign-in, account, and audit API](/reference/auth-api/index.html.md#setsignupnote)). |
| `limit` of `hopbox_limit_refusals_total` | `machines`, `vcpus`, `memory`, and `disk`: the usage limits of the org. `public_urls`: a change of a machine URL to public in an org without public URLs ([the machine API](/reference/machine-api/index.html.md#setmachineurlvisibility)). `orgs`: the orgs that one account owns (`max_orgs_per_account`), and each `CreateOrg` of a limited account. |
| `kind` | `member`, `org`, or `admin` |
| `lifetime` | `30d`, `90d`, `365d`, or `custom` for a key with a custom expiry |

Each value of `limit`, `kind`, and `lifetime` starts at 0, so the first refusal or key shows as an increase. A series of `hopbox_audit_entries_total` shows after its first entry.

```sh
curl -s http://127.0.0.1:9464/metrics | grep -E '^hopbox_(audit_entries|rate_limited|limit_refusals|api_key_creations)_total'
```

```text
hopbox_api_key_creations_total{kind="member",lifetime="90d"} 2
hopbox_audit_entries_total{action="org.create",actor_kind="session"} 1
hopbox_limit_refusals_total{limit="machines"} 1
hopbox_rate_limited_total{limit="sign_in_links"} 3
```

## Metrics of the leader

The leader of the control role reads these gauges from the control store every minute, and at the start of its jobs ([hopboxd](/reference/hopboxd/index.html.md#leader-jobs)). It serves them until its jobs end, at the end of its term or in maintenance mode. A read takes at most 30 s. A read that fails keeps the values of the read before. After 3 failed reads in a row, the leader serves none of the gauges until a read works, so a dashboard shows a gap and not old values. Another control process serves none of them, so no value counts twice. The leader serves each value of the fixed label sets, so a count that falls to 0 shows as 0.

| Metric | Type | Labels | Description |
| --- | --- | --- | --- |
| `hopbox_accounts` | gauge | `state`, `tier` | The accounts by state (`pending`, `active`, or `suspended`) and tier (`limited` or `full`). |
| `hopbox_pending_account_oldest_seconds` | gauge | none | The age of the oldest `pending` account, or 0 without one. |
| `hopbox_orgs` | gauge | `kind`, `suspended` | The orgs by kind (`personal` or `team`) and suspension (`true` or `false`). |
| `hopbox_api_keys` | gauge | `kind` | The API keys that did not expire, by kind (`member`, `org`, or `admin`). |
| `hopbox_public_machine_urls` | gauge | none | The machines with a public URL, apart from the deleted machines ([Run the edge role and open machine URLs](/guide/machine-urls/index.html.md)). |
| `hopbox_machines` | gauge | `state` | The machines by current state ([Domain rules](/reference/domain-rules/index.html.md#machine-states)). A deleted machine counts as `deleted` until the sweep drops its row. |

The registry gets the capacity of each server in its heartbeats ([the control channel](/reference/control-channel/index.html.md)). The leader serves the capacity and the allocations of each server with the node role that is not decommissioned, with the label `server`, the ID of the server. So one Prometheus on the control server sees the capacity of the whole fleet without a scrape of the nodes.

| Metric | Type | Labels | Description |
| --- | --- | --- | --- |
| `hopbox_server_vcpus` | gauge | `server` | The vCPU capacity of the node: its CPU threads times the vCPU overcommit of 2 ([Placement](/reference/placement/index.html.md)). |
| `hopbox_server_vcpus_allocated` | gauge | `server` | The vCPUs of the machines that use compute on the node, as placement counts them ([Placement](/reference/placement/index.html.md#what-a-node-uses)). |
| `hopbox_server_memory_bytes` | gauge | `server` | The RAM of the node. |
| `hopbox_server_memory_allocated_bytes` | gauge | `server` | The RAM of the machines that use compute on the node, as placement counts them. |
| `hopbox_server_disk_bytes` | gauge | `server` | The size of the file system of the data dir of the node. |
| `hopbox_server_disk_free_bytes` | gauge | `server` | The free space of that file system. |
| `hopbox_server_disk_allocated_bytes` | gauge | `server` | The disk sizes of the machines on the node that are not deleted. A disk is sparse, so this sum can be larger than the disk of the node. |
| `hopbox_server_machines` | gauge | `server`, `state` | The machines on the node by current state. |

Each node has 16 series: one for each of the 7 capacity gauges, and 9 of `hopbox_server_machines`, one for each state. The leader also serves 6 series of `hopbox_accounts`, 4 of `hopbox_orgs`, 3 of `hopbox_api_keys`, and 9 of `hopbox_machines`.

```sh
curl -s http://127.0.0.1:9464/metrics | grep -E '^hopbox_(accounts|server_vcpus)'
```

```text
hopbox_accounts{state="active",tier="full"} 3
hopbox_accounts{state="pending",tier="limited"} 1
hopbox_server_vcpus{server="0193a5b2-7c1e-7d4a-9f3b-2a1c4e5f6a7b"} 24
hopbox_server_vcpus_allocated{server="0193a5b2-7c1e-7d4a-9f3b-2a1c4e5f6a7b"} 4
```

## Metrics of the DNS role

A server with the DNS role adds these metrics to the listener of its `hopboxd run` ([the DNS role](/reference/dns/index.html.md#metrics)). A server without the DNS role does not have them.

| Metric | Type | Labels | Description |
| --- | --- | --- | --- |
| `hopbox_dns_queries_total` | counter | `transport`, `rcode` | The DNS queries that the DNS role answered. |
| `hopbox_dns_rate_limited_total` | counter | `action` | The UDP answers over the rate limit of their client network. `action` is `slip`, a short answer with the TC flag, or `drop`, no answer ([the DNS role](/reference/dns/index.html.md#rate-limit-on-udp)). |
| `hopbox_dns_zone_serial` | gauge | none | The serial of the zones that the DNS role serves. It is 0 before the first zones arrive. |

| Label | Values |
| --- | --- |
| `transport` | `udp` or `tcp` |
| `rcode` | The response code of the answer, such as `NOERROR`, `NXDOMAIN`, `REFUSED`, `SERVFAIL`, `FORMERR`, `NOTIMP`, or `BADVERS` |

```sh
curl -s http://127.0.0.1:9464/metrics | grep '^hopbox_dns_'
```

```text
hopbox_dns_queries_total{rcode="NOERROR",transport="udp"} 12
hopbox_dns_queries_total{rcode="NXDOMAIN",transport="udp"} 2
hopbox_dns_zone_serial 1.7922494e+09
```

The Prometheus text format writes a large gauge in exponent form, so the serial 1792249400 shows as `1.7922494e+09`.

## Metrics of the edge role

A server with the edge role adds these metrics to the listener of its `hopboxd run` ([the edge role](/reference/edge/index.html.md#metrics)). A server without the edge role does not have them.

| Metric | Type | Labels | Description |
| --- | --- | --- | --- |
| `hopbox_edge_requests_total` | counter | `to` | The HTTP requests that the edge role took, by where they went. |
| `hopbox_edge_tls_refusals_total` | counter | `reason` | The TLS handshakes that the edge role refused. |
| `hopbox_edge_client_connection_refusals_total` | counter | `listener`, `bound` | The connections that the edge role closed at once, because their client or their IPv6 /48 held the most open connections ([the edge role](/reference/edge/index.html.md#listeners)). |
| `hopbox_edge_open_connections` | gauge | `listener` | The connections that a listener of the edge role holds now. |
| `hopbox_edge_connection_limit` | gauge | `listener` | The most connections that a listener of the edge role holds at once. |
| `hopbox_edge_routes` | gauge | none | The machines that the machine URLs of the edge role name. |
| `hopbox_edge_certificate_not_after_seconds` | gauge | none | The end of the certificate that the edge role serves, in seconds since the Unix epoch. It is 0 before the first certificate. |
| `hopbox_edge_machine_requests_total` | counter | `visibility`, `status_class` | The HTTP requests that the edge role forwarded to a machine, by the visibility of the machine URL (`public` or `private`) and the class of the status of the answer (`1xx` to `5xx`). A forward that failed counts as `5xx`. Each of the 10 series starts at 0 ([the edge role](/reference/edge/index.html.md#metrics)). |
| `hopbox_edge_ssh_connections_total` | counter | `result` | The TCP connections of the SSH front door, by result ([the SSH front door](/reference/ssh-front-door/index.html.md#metrics)). |
| `hopbox_edge_ssh_auth_total` | counter | `result` | The keys that SSH clients signed with: `allowed`, `refused`, `limited`, or `error`. |
| `hopbox_edge_ssh_unreachable_total` | counter | `reason` | The signed-in SSH connections that reached no OpenSSH server of a machine. |
| `hopbox_edge_ssh_channels_total` | counter | `type`, `result` | The SSH channels that the front door forwarded or refused. |
| `hopbox_edge_ssh_ended_total` | counter | `reason` | The open SSH connections that the front door ended. |
| `hopbox_edge_ssh_open_connections` | gauge | none | The signed-in SSH connections that the front door holds now. |
| `hopbox_edge_ssh_lost_opens` | gauge | none | The channel opens of the front door that got no answer and still wait, after their connection ended ([the SSH front door](/reference/ssh-front-door/index.html.md#channels-and-requests)). |

| Label | Values |
| --- | --- |
| `to` | `api`: to the control role. `machine`: to a machine. `redirect`: a redirect to HTTPS or to a sign-in, or the cookie of a sign-in. `refused`: a page of the edge, such as 404, 421, or 503. |
| `reason` | `no_sni`: the client sent no server name. `unknown_host`: the name is not under the domain. `no_certificate`: the edge has no certificate for the domain yet. |

```sh
curl -s http://127.0.0.1:9464/metrics | grep '^hopbox_edge_'
```

```text
hopbox_edge_certificate_not_after_seconds 1.7998e+09
hopbox_edge_requests_total{to="api"} 42
hopbox_edge_requests_total{to="machine"} 7
hopbox_edge_requests_total{to="redirect"} 3
hopbox_edge_routes 2
hopbox_edge_tls_refusals_total{reason="no_sni"} 1
```

## Metrics of the executors

The executors of a node count their actions ([the node executor](/reference/executor/index.html.md#metrics)). `hopboxd run` registers them on a server with the node role. The Firecracker driver adds no metric of its own ([the Firecracker driver](/reference/firecracker-driver/index.html.md#logs)).

| Metric | Type | Labels | Description |
| --- | --- | --- | --- |
| `hopbox_executor_actions_total` | counter | `action`, `result` | The actions of the executors of the node. |
| `hopbox_executor_action_duration_seconds` | histogram | `action` | The duration of the actions. |

## Metrics of the machines

A server with the node role serves what its running machines use. At each scrape, the node reads the cgroup of the unit `hopbox-machine-<id>.service` of each machine that runs ([the Firecracker driver](/reference/firecracker-driver/index.html.md)). The cgroup holds Firecracker and its threads, so the values include the VMM. The node totals come from the cgroup of the slice `hopbox-machines.slice`.

| Metric | Type | Labels | Description |
| --- | --- | --- | --- |
| `hopbox_machine_info` | gauge | `machine`, `name`, `org`, `size`, `vcpus`, `memory_bytes`, `disk_bytes` | 1 for each machine that runs on the node. It is the only series that links a machine to its org, by org ID. A query joins it to the other series on `machine`. |
| `hopbox_machine_cpu_seconds_total` | counter | `machine` | The CPU time of the machine since its boot, from `usage_usec` of `cpu.stat`. It starts from zero at each boot. |
| `hopbox_machine_memory_bytes` | gauge | `machine` | The host RAM that the machine holds, from `memory.current`. |
| `hopbox_machine_disk_bytes` | gauge | `machine` | The bytes of the blocks that the sparse disk file of the machine holds on the node. |
| `hopbox_machines_cpu_seconds_total` | counter | none | The CPU time of every machine of the node, from `cpu.stat` of the slice. |
| `hopbox_machines_memory_bytes` | gauge | none | The host RAM of every machine of the node, from `memory.current` of the slice. |

| Label | Values |
| --- | --- |
| `machine` | The ID of the machine |
| `name` | The name of the machine |
| `org` | The ID of the org of the machine, or empty for a machine whose org is gone |
| `size` | The name of the size of the machine, such as `default` |
| `vcpus`, `memory_bytes`, `disk_bytes` | The vCPUs, the RAM in bytes, and the disk in bytes of the size |

A machine that runs before its first spec after a restart of `hopboxd` has no `hopbox_machine_info` until the spec arrives. A read that fails leaves its series out of that scrape, and the other series stay. For example, a machine can stop between the list of the machines and the read of its cgroup. The slice has no cgroup before the first machine runs in it, so the node totals are absent then.

Each running machine has 4 series in this section. A running machine with network also has 5 series in [Metrics of the node network](#metrics-of-the-node-network), and a machine without network has none of them. So a node with N running machines, each with network, serves 9 × N + 2 series of the machines. The catalog plans up to 15 series for each running machine with the metrics of later milestones.

```sh
curl -s http://127.0.0.1:9464/metrics | grep -E '^hopbox_machines?_(info|cpu|memory|disk_bytes)'
```

```text
hopbox_machine_cpu_seconds_total{machine="0193a5b2-7c1e-7d4a-9f3b-2a1c4e5f6a7b"} 812.4
hopbox_machine_disk_bytes{machine="0193a5b2-7c1e-7d4a-9f3b-2a1c4e5f6a7b"} 3.81e+09
hopbox_machine_info{disk_bytes="100000000000",machine="0193a5b2-7c1e-7d4a-9f3b-2a1c4e5f6a7b",memory_bytes="8589934592",name="box1",org="0193a5b2-0000-7000-8000-000000000001",size="default",vcpus="2"} 1
hopbox_machine_memory_bytes{machine="0193a5b2-7c1e-7d4a-9f3b-2a1c4e5f6a7b"} 2.15e+09
hopbox_machines_cpu_seconds_total 812.4
hopbox_machines_memory_bytes 2.15e+09
```

## Metrics of the node network

A server with the node role serves the traffic counters of each machine with network ([the node network](/reference/node-network/index.html.md#metrics)). The node reads them from the table `inet hopbox_node` at each scrape. They start from zero at each start of `hopboxd`.

| Metric | Type | Labels | Description |
| --- | --- | --- | --- |
| `hopbox_machine_inbound_bytes_total` | counter | `machine` | Bytes to the machine of connections that the node opened. |
| `hopbox_machine_inbound_packets_total` | counter | `machine` | Packets to the machine of connections that the node opened. |
| `hopbox_machine_egress_bytes_total` | counter | `machine` | Bytes from the machine that the node let out. |
| `hopbox_machine_egress_packets_total` | counter | `machine` | Packets from the machine that the node let out. |
| `hopbox_machine_egress_connections_total` | counter | `machine` | Packets that opened an outbound connection of the machine that the node let out, retries included ([the node network](/reference/node-network/index.html.md#counters)). |
| `hopbox_node_network_repairs_total` | counter | none | The installs of the table `inet hopbox_node` after it lost a part outside `hopboxd`, or after a check that got no answer within 10 s ([the node network](/reference/node-network/index.html.md#the-table-must-stay)). |

```sh
curl -s http://127.0.0.1:9464/metrics | grep -E '^hopbox_(machine|node_network)_'
```

## Metrics of the mesh

A server with the mesh device `hbm0` serves these metrics ([the mesh](/reference/mesh/index.html.md#metrics)). The device reads the counters of WireGuard at each scrape. A server outside the mesh serves none of them. The label `server` is the ID of the peer.

| Metric | Type | Labels | Description |
| --- | --- | --- | --- |
| `hopbox_mesh_peers` | gauge | none | The peers of the mesh device. |
| `hopbox_mesh_peer_last_handshake_seconds` | gauge | `server` | The Unix time of the last WireGuard handshake with the peer, or 0 before the first one. |
| `hopbox_mesh_peer_receive_bytes_total` | counter | `server` | The bytes that the mesh device got from the peer. |
| `hopbox_mesh_peer_transmit_bytes_total` | counter | `server` | The bytes that the mesh device sent to the peer. |
| `hopbox_mesh_repairs_total` | counter | none | The changes to the mesh device after it lost a part outside `hopboxd` ([the mesh](/reference/mesh/index.html.md#a-restart-a-stop-and-a-reboot)). |

```sh
curl -s http://127.0.0.1:9464/metrics | grep '^hopbox_mesh_'
```

## Metrics of the host firewall

Every server on Linux serves these metrics of its table `inet hopbox_host` ([the host firewall](/reference/host-firewall/index.html.md#metrics)).

| Metric | Type | Labels | Description |
| --- | --- | --- | --- |
| `hopbox_host_firewall_repairs_total` | counter | none | The installs of the table `inet hopbox_host` after it lost a part outside `hopboxd`, or after a check that got no answer within 10 s ([the host firewall](/reference/host-firewall/index.html.md#the-table-must-stay)). |
| `hopbox_host_firewall_up` | gauge | none | 1 when the last check found the table whole or installed it. 0 when its install failed. A host without nftables serves neither metric. |

```sh
curl -s http://127.0.0.1:9464/metrics | grep '^hopbox_host_firewall_'
```

## Metrics of the cold moves

A node whose driver keeps its disks in files counts the bytes of the disks that it serves and pulls for the cold moves ([Cold move](/reference/cold-move/index.html.md#metrics)). The pulls also count in `hopbox_executor_actions_total` with the action `pull`.

| Metric | Type | Labels | Description |
| --- | --- | --- | --- |
| `hopbox_disk_transfer_bytes_total` | counter | `direction` | The bytes of disk data of the pulls: `sent` to other nodes, and `received` from another node. |

## node_exporter

node_exporter serves the metrics of the host: CPU, RAM, file systems, disks, network, and pressure. hopbox does not run it. The operator installs the Debian 13 package `prometheus-node-exporter` on each server. The release bundle carries the arguments of node_exporter in `monitoring/node-exporter/prometheus-node-exporter`. They make node_exporter listen on `127.0.0.1:9100` only, as the metrics listener of `hopboxd` does. The default collectors of node_exporter give every host metric that the alerts read: `node_cpu_seconds_total`, `node_memory_MemAvailable_bytes`, `node_filesystem_avail_bytes`, `node_disk_*`, `node_network_*`, and `node_pressure_*`.

To set up node_exporter on a server, run these commands as root:

1. Install the package.

   ```sh
   apt-get install prometheus-node-exporter
   ```

2. Copy the arguments of the release over the file of the package.

   ```sh
   install -m 0644 /opt/hopbox/current/monitoring/node-exporter/prometheus-node-exporter /etc/default/prometheus-node-exporter
   ```

3. Restart node_exporter.

   ```sh
   systemctl restart prometheus-node-exporter
   ```

4. Make sure that node_exporter answers on loopback.

   ```sh
   curl -s http://127.0.0.1:9100/metrics | grep '^node_memory_MemAvailable_bytes'
   ```

The host firewall of hopbox blocks inbound traffic on port 9100 ([the host firewall](/reference/host-firewall/index.html.md)). An upgrade of `hopboxd` does not copy the file again, so run step 2 after an upgrade whose file is new.

## Metrics of the process

| Prefix | Source |
| --- | --- |
| `go_` | The Go runtime: goroutines, memory, and garbage collection. |
| `process_` | The process: CPU time, memory, open files, and start time. Linux only. On macOS, these metrics are absent. |
