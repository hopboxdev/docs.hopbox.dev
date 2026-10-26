# Placement

> How a create picks the node of a machine: the hard filters, the usage of a node, the score, the retries, the no-capacity error, the room check of a start, and the placement metric.

Placement picks the node of a new machine. `CreateMachine` runs it in its transaction ([the machine API](/reference/machine-api/index.html.md#createmachine)). The start of a stopped machine uses the same rules of room on the node of the machine. A cold move picks its target node with the same filters ([Cold move](/reference/cold-move/index.html.md#start-of-a-move)). The Go package `internal/placement` holds the rules, without I/O. The service layer reads the nodes and makes the placement (`internal/service/machines.go`).

## The steps of a placement

1. It reads every server that is not decommissioned, with what the machines of each one use ([What a node uses](#what-a-node-uses)). A decommissioned server is never a candidate, and the message of [When no node has room](#when-no-node-has-room) does not count it.
2. It runs the [hard filters](#hard-filters) on each server. A server that fails a filter is out, and the first filter that it fails is its reason.
3. It shuffles the servers that pass, and takes up to 3 of them: the candidates.
4. It scores each candidate ([Score](#score)). The lowest score wins.
5. It tries the winner. The try locks the row of the server in `servers` with `FOR NO KEY UPDATE`, and reads its state, its labels, and its usage again. Then it runs the filters again. A server that fails a filter now refuses the placement, for example a server that has no room any more, or whose labels changed.
6. After a refusal, it waits, reads the servers again, and goes back to step 2 without the servers that refused. It tries at most 3 servers in all.

The try runs in a savepoint of the transaction of the create. A server that refuses rolls back its savepoint, so its lock goes at once, and its heartbeats and other placements need not wait. The server that takes the machine stays locked until the create commits. The heartbeat of that server waits for the commit too.

| Rule | Value |
| --- | --- |
| Candidates | 3 random servers that pass the filters (`placement.Candidates`) |
| Tries | 3 servers at most (`placement.Tries`) |
| Wait before the second try | 20 ms (`placement.DefaultBackoff`) |
| Wait before the third try | 40 ms: each wait is twice the wait before it |
| Jitter | Each wait is a random time from half of its value to the whole value, so the tries of many creates spread out. |
| vCPU overcommit | 2 vCPUs for each CPU thread of a server (`placement.CPUOvercommit`) |
| RAM overcommit | none |

## Hard filters

A server must pass every filter, in this order:

| Order | Filter | The server passes when | Reason of a server that fails |
| --- | --- | --- | --- |
| 1 | Role | Its roles hold `node`. | `not_node` |
| 2 | State | Its state is `ready`. A `cordoned` or `unresponsive` server fails ([Control store tables](/reference/control-store/index.html.md#servers)). | `not_ready` |
| 3 | Health | Its last heartbeat had the status `ok`. A server whose last heartbeat had the status `degraded` fails: a check of severity `fail` failed, such as a missing `/dev/kvm` ([the preflight check](/reference/preflight/index.html.md)). | `degraded` |
| 4 | Pool | The request names no pool, or the pool of the server. | `pool` |
| 5 | Labels | The server has each label of the request, with the same value. | `labels` |
| 6 | Compatibility class | The request names no class, or the class of the server. | `class` |
| 7 | Version | Its `hopboxd` is at least the oldest version that the request allows. The version must be a valid semver version. | `version` |
| 8 | Cold move | Only for the target of a cold move: its heartbeats say that it pulls disks with its current `hopboxd`, and it is in the mesh ([Cold move](/reference/cold-move/index.html.md#two-releases)). A create skips this filter. | `move` |
| 9 | vCPUs | The vCPUs that it uses, plus the vCPUs of the size, are at most 2 times its CPU threads. | `cpu` |
| 10 | RAM | The RAM that it uses, plus the RAM of the size, is at most its RAM. | `ram` |

A create takes the pool and the labels of its size for filters 4 and 5. So does a cold move, with the size of the machine. The size `default` names no pool and no label, so its machines pass both filters on every server. A move with memory state names a compatibility class. A cold move carries no memory, so it names none, and filter 6 passes every server. The target of a cold move is never the node of the machine.

The oldest version that a create allows is `version.MinNodeVersion`. Today it is `v0.0.0-0.unknown`, which sorts below every build ([the version command](/reference/version/index.html.md#version-format)). So every node with a valid version passes. Only a release whose executor protocol needs it raises it, to the oldest release that follows the new protocol ([the executor protocol](/reference/executor-protocol/index.html.md#two-releases)). In a rolling upgrade, the nodes of the release before then take no new machine until they upgrade.

Disk is not a filter in phase 1. The disk of a machine is sparse, so its size says little about the space that it takes on the node.

## What a node uses

A machine uses its vCPUs and its RAM on its node in these cases:

- Its target is `running` or `standby` (`HoldsCompute`, [Domain rules](/reference/domain-rules/index.html.md#usage-limits)).
- Its current state is `starting`, `running`, `standby`, or `moving`, because its microVM can still run on the way to another target.

A stopped, hibernated, or failed machine whose target is `stopped` or `hibernated` uses nothing. A machine whose target is `deleted` uses its vCPUs and RAM until its microVM stops.

A placement that runs in another transaction counts too. The try of step 5 locks the server row and reads the usage again in a new statement. A second placement on the same server waits for that lock. When it gets the lock, the first create committed, and the new statement sees its machine. So two creates at the same time never pass the room check of one server together.

## Score

The score of a candidate is how full it is after it takes the machine:

```text
score = 0.7 × (used RAM + RAM of the size) / RAM of the server
      + 0.3 × (used vCPUs + vCPUs of the size) / (2 × CPU threads)
```

RAM counts most, because it has no overcommit. The lowest score wins. For example, take the size `default` (2 vCPUs, 8 GiB) and two servers with 12 threads and 128 GiB each. A server that uses 4 vCPUs and 16 GiB scores 0.7 × 24/128 + 0.3 × 6/24 = 0.21. An empty server scores 0.7 × 8/128 + 0.3 × 2/24 = 0.069, and wins.

## When no node has room

A placement fails in two cases: no server passes the filters, or 3 servers refused it. The create then fails with `unavailable`, and leaves nothing behind ([the machine API](/reference/machine-api/index.html.md#createmachine)). A retry can succeed once a node has room, and the SDKs retry `unavailable`.

The message names the size and the reason of each server:

```text
no capacity: no node has room for a machine of size <size> (<n> vCPUs, <RAM> of RAM): <reasons>
```

`<RAM>` is in GiB for a whole number of GiB, else in MiB. `<reasons>` counts the servers by the first filter that each one failed, in the order of the filters, joined by `, `. A server that refused the placement counts as `refused the placement`. After 3 refusals, the counts come from the last read of the servers.

| Reason | One server | More servers |
| --- | --- | --- |
| `not_node` | `1 node has no node role` | `<n> nodes have no node role` |
| `not_ready` | `1 node is cordoned, unresponsive, or decommissioned` | `<n> nodes are cordoned, unresponsive, or decommissioned` |
| `degraded` | `1 node failed a check in its last heartbeat` | `<n> nodes failed a check in their last heartbeat` |
| `pool` | `1 node is in another pool` | `<n> nodes are in another pool` |
| `labels` | `1 node lacks a label` | `<n> nodes lack a label` |
| `class` | `1 node is in another compatibility class` | `<n> nodes are in another compatibility class` |
| `version` | `1 node runs a hopboxd that is too old` | `<n> nodes run a hopboxd that is too old` |
| `move` | `1 node pulls no disk of a cold move` | `<n> nodes pull no disk of a cold move` |
| `cpu` | `1 node has too few free vCPUs` | `<n> nodes have too few free vCPUs` |
| `ram` | `1 node has too little free RAM` | `<n> nodes have too little free RAM` |
| `refused` | `1 node refused the placement` | `<n> nodes refused the placement` |

Examples:

```text
no capacity: no node has room for a machine of size default (2 vCPUs, 8 GiB of RAM): 1 node is cordoned, unresponsive, or decommissioned, 2 nodes have too little free RAM
```

An install without a server that is not decommissioned gets this message:

```text
no capacity: no node has room for a machine of size default (2 vCPUs, 8 GiB of RAM). The install has no node yet
```

The error wraps `placement.ErrNoCapacity`. The API maps it to `unavailable` ([the API server](/reference/api/index.html.md#errors)). The Go type `placement.NoCapacityError` holds the size, the count of servers, and the count of each reason.

## Start of a stopped machine

The disk of a machine lives on its node, so a start never moves the machine. Only a cold move takes a machine and its disk to another node ([Cold move](/reference/cold-move/index.html.md)), and the next start checks the room on the new node. The start of a machine whose target is `stopped` checks the room on its own node with the same rules: the vCPUs with an overcommit of 2, and the RAM without overcommit ([the machine API](/reference/machine-api/index.html.md#startmachine)). It locks the server row with `FOR NO KEY UPDATE`, and counts every other machine of the node as in [What a node uses](#what-a-node-uses). It checks no other filter: a cordoned or unresponsive node keeps its machines. For a machine on a decommissioned node, a start skips the check.

| Case | Code | Message |
| --- | --- | --- |
| Too few free vCPUs | `unavailable` | `no capacity: the node of the machine has too few free vCPUs to start it. Stop another machine, or retry later` |
| Too little free RAM | `unavailable` | `no capacity: the node of the machine has too little free RAM to start it. Stop another machine, or retry later` |

## The machine lease

The placement grants the machine lease `machine/<machine ID>` to the chosen node, in the transaction of the create, with a TTL of 30 s ([Control store tables](/reference/control-store/index.html.md#leases)). The holder is the server ID of the node. The row of `placements` keeps the epoch and the token of the grant, and the count of tries. The control role then sends the spec of the machine to the node with this grant. The executor of the machine acts with it ([the executor protocol](/reference/executor-protocol/index.html.md)). The heartbeats of the node renew the lease. The executor protocol keeps this contract ([Control store tables](/reference/control-store/index.html.md#leases)):

- The lease row always holds the epoch and the token of the current grant.
- Each new grant of the lease writes a new row of `placements` with its epoch and token, in the same transaction. An example is a new grant after an expiry.
- A heartbeat that renews or grants the leases of its machines locks the machine rows first, then the server row, as a placement does.

A machine stays on its node. A new grant after an expiry goes to the same node, and only after the node sends a heartbeat again. An unresponsive node keeps its machines, and no other node starts them.

## Logs

A create that finds no node with room writes this record with `component=service`:

| Message | Level | Fields |
| --- | --- | --- |
| `no node has room for a machine` | warn | `org`, `err` (the message of [When no node has room](#when-no-node-has-room)), `request_id` |

## Metrics

| Metric | Type | Labels | Description |
| --- | --- | --- | --- |
| `hopbox_placements_total` | counter | `result`, `reason` | The placements of creates ([Metrics](/reference/metrics/index.html.md#metrics-of-placement)). |

| Label | Values |
| --- | --- |
| `result` | `placed` for a create that committed, `no_capacity` for a create that found no node with room or no free SSH port |
| `reason` | Empty for `placed`. For `no_capacity`, the reason that skipped the most servers: `not_node`, `not_ready`, `degraded`, `pool`, `labels`, `class`, `version`, `cpu`, `ram`, or `refused`. A tie goes to the reason of the earlier filter. `none` for an install without a server, and `ssh_ports` for a full range of SSH ports. |

A create with an idempotency key runs in the transaction of its key, so it counts as `placed` only after that transaction commits.

A create that fails before its placement, or for another cause after it, does not count. A start of a stopped machine does not count either.

```text
hopbox_placements_total{reason="",result="placed"} 4
hopbox_placements_total{reason="ram",result="no_capacity"} 1
```
