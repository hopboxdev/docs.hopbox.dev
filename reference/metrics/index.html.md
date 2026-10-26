# Metrics

> The Prometheus metrics listener of hopboxd and every metric that it serves.

`hopboxd` serves Prometheus metrics on a loopback listener. The code is in `internal/obs`, `internal/api`, `internal/placement`, `internal/node/executor`, `internal/node/netcfg`, and `internal/dns`.

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

## Metrics of the DNS role

A server with the DNS role adds these metrics to the listener of its `hopboxd run` ([the DNS role](/reference/dns/index.html.md#metrics)). A server without the DNS role does not have them.

| Metric | Type | Labels | Description |
| --- | --- | --- | --- |
| `hopbox_dns_queries_total` | counter | `transport`, `rcode` | The DNS queries that the DNS role answered. |
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
| `hopbox_edge_ssh_connections_total` | counter | `result` | The TCP connections of the SSH front door, by result ([the SSH front door](/reference/ssh-front-door/index.html.md#metrics)). |
| `hopbox_edge_ssh_auth_total` | counter | `result` | The keys that SSH clients signed with: `allowed`, `refused`, `limited`, or `error`. |
| `hopbox_edge_ssh_unreachable_total` | counter | `reason` | The signed-in SSH connections that reached no OpenSSH server of a machine. |
| `hopbox_edge_ssh_channels_total` | counter | `type`, `result` | The SSH channels that the front door forwarded or refused. |
| `hopbox_edge_ssh_ended_total` | counter | `reason` | The open SSH connections that the front door ended. |
| `hopbox_edge_ssh_open_connections` | gauge | none | The signed-in SSH connections that the front door holds now. |

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

## Metrics of the node network

A server with the node role serves the traffic counters of each machine with network ([the node network](/reference/node-network/index.html.md#metrics)). The node reads them from the table `inet hopbox_node` at each scrape. They start from zero at each start of `hopboxd`.

| Metric | Type | Labels | Description |
| --- | --- | --- | --- |
| `hopbox_machine_inbound_bytes_total` | counter | `machine` | Bytes to the machine of connections that the node opened. |
| `hopbox_machine_inbound_packets_total` | counter | `machine` | Packets to the machine of connections that the node opened. |
| `hopbox_machine_egress_bytes_total` | counter | `machine` | Bytes from the machine that the node let out. |
| `hopbox_machine_egress_packets_total` | counter | `machine` | Packets from the machine that the node let out. |
| `hopbox_node_network_repairs_total` | counter | none | The installs of the table `inet hopbox_node` after it lost a part outside `hopboxd` ([the node network](/reference/node-network/index.html.md#the-table-must-stay)). |

```sh
curl -s http://127.0.0.1:9464/metrics | grep -E '^hopbox_(machine|node_network)_'
```

## Metrics of the process

| Prefix | Source |
| --- | --- |
| `go_` | The Go runtime: goroutines, memory, and garbage collection. |
| `process_` | The process: CPU time, memory, open files, and start time. Linux only. On macOS, these metrics are absent. |
