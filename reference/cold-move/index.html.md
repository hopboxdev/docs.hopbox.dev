# Cold move

> The cold move of a machine to another node. Its steps, and the disk listener on port 7444 of the mesh address. The stream and its digest, a resumed pull, a failed move, old copies, two releases, metrics, and logs.

A cold move takes one machine from its node to another node (step 4 of a drain). The source node stops the machine. Then the target node pulls the disk of the machine from the source node over the mesh. If the target of the machine is `running`, the target node boots it. The machine loses its memory, and keeps its disk, its name, its URL, its SSH port, and its address. The disk travels in one copy format for every moving state: a stream of the blocks that hold data, with a digest of the whole disk.

An instance admin starts a move with `MoveMachine` of the admin API ([the admin API](/reference/admin-api/index.html.md#movemachine)) or `hopboxd admin machine move` ([hopboxd](/reference/hopboxd/index.html.md#admin-machine-move)). `CancelMove` or `hopboxd admin machine move --cancel` ends a move that runs. The drain of a node moves each machine of the node the same way ([Drain of a node](#drain-of-a-node)).

The control side is in `internal/service/moves.go` and the leader job of `internal/control`. The node side is in `internal/node/transfer` (the disk listener and the pull), `internal/node/executor` (`move.go`), and `internal/node/machines`.

## Values

| Item | Value |
| --- | --- |
| Disk listener | Port `7444` (`transfer.Port`) of the mesh address of the node, with TLS 1.3 ([the mesh](/reference/mesh/index.html.md)) |
| Block | 4096 bytes (`transfer.BlockSize`). A block of zeros is a hole. |
| Frame | At most 1 MiB of data |
| Idle timeout of a pull | 30 s without a byte (`transfer.DefaultIdleTimeout`), counted only while the target waits for the source. The timer counts each wait for the next bytes, so one frame can take longer than 30 s. A save of the progress and the wait of a rate limit do not count. A source that keeps sending bytes holds a pull until the action timeout. If no frame completes, the pull reports no `pull_received`, so the pull deadline ends the move. |
| Write timeout of the listener | 30 s for each frame (`transfer.DefaultWriteTimeout`) |
| Save of the progress of a pull | Every 64 MiB, and at each stop of the pull |
| Length of one pull | At most 5 min, the timeout of an action of the executor (`executor.DefaultActionTimeout`). A larger disk arrives over several pulls. |
| Wait between two pulls | 1 s after a pull that got bytes. After a pull without a byte, 2 s, then twice the wait each time, up to 30 s. |
| Pulls that fail for good | 3 in a row (`maxPullFailures`) |
| Stop timeout | 10 min (`domain.MoveStopTimeout`) |
| Pull deadline | 30 min without progress (`domain.MovePullTimeout`): from the hand-off, or from the last pull that wrote bytes |
| Step of the leader job | Every 2 s (`control.MoveEvery`) |

## The steps of a move

| State of the move | What happens | Current state of the machine |
| --- | --- | --- |
| `stopping` | `MoveMachine` checks the machine, picks the target, and records the move. The spec of the machine on its node then has the target `stopped`, so the executor stops it. The machine row keeps its own target. | `running`, then `stopped` |
| `pulling` | Once the machine is `stopped`, the leader hands it to the target node in one transaction. The source loses its lease. The target gets a new grant and a new row of `placements`. The machine and its address go to the target node with the current state `moving`. The target node pulls the disk from the source node. | `moving` |
| `done` | The disk on the target is whole, and its digest matches. The executor of the target reports `stopped` with the proof of the pull, the ID of the move. If the target of the machine is `running`, the executor boots it after the control role accepts that report. The source node then removes its old copy. | `stopped`, then `starting` and `running` |
| `failed` | The pull failed, the pull deadline passed, or an instance admin canceled the move. The machine stays on the source node, or goes back there, with its disk. A machine that is no longer in `moving` on the target stays there ([A step that an older control process accepted](#a-step-that-an-older-control-process-accepted)). | `stopped` |

The control role writes the current state `moving` itself, and also `stopped` after a failed pull. In both cases, no executor holds the lease of the machine at the time of the write ([the executor protocol](/reference/executor-protocol/index.html.md#cold-moves)).

### Start of a move

`MoveMachine` locks the machine as a lifecycle call does, and refuses these cases with `failed_precondition`:

| Case | Message |
| --- | --- |
| The machine never booted, has the current state `hibernated`, `standby`, `moving`, or `deleted`, or a target other than `running` and `stopped` | `a cold move needs a machine that booted, with the target running or stopped. This one is <state>, with the target <target>` |
| The machine moves already, or a node still holds an old copy of its files | `the machine moves already, or a node still removes the files of its last move. Retry later` |
| A failed move left the machine on its target, and its source keeps a copy of its disk ([A step that an older control process accepted](#a-step-that-an-older-control-process-accepted)) | `a failed move left the machine on its node, and the node <id> keeps a copy of its disk. The node of the machine can lack a whole disk of it, so it does not move again. Delete the machine to end it (reference/cold-move.md)` |
| The node of the machine serves no disk | `the node <id> of the machine serves no disk for a cold move. It needs a hopboxd that knows cold moves, and an address in the mesh` |
| The named target fails a filter of placement | `the node <id> cannot take the machine: it <reason>` |
| A live control process runs a release of the schema version 44 to 49 ([Two releases](#two-releases)) | `the control process <id> on <host> (hopboxd <release>) knows the migrations up to <version>, and only a hopboxd that knows migration 50 moves machines. Upgrade it first` |

A named target that is the node of the machine fails with `invalid_argument`. An unknown named target fails with `not_found`. Without a named target, placement picks one ([Placement](/reference/placement/index.html.md#hard-filters)). It takes the size, the pool, and the labels of the size of the machine, and leaves out the node of the machine. A move needs no compatibility class, because it carries no memory. When no node takes the machine, the call fails with `unavailable` and the message of [Placement](/reference/placement/index.html.md#when-no-node-has-room).

The start of a move writes the audit entry `machine.move` with `move`, `source`, `target`, and `old_target`.

### Hand-off to the target

The leader job runs every 2 s. For each move in `stopping`, it locks the machine row, then the move row, and does one of these things:

| Machine | What the job does |
| --- | --- |
| `stopped`, and its node is `ready`, `cordoned`, `draining`, or `shutting_down` | It hands the machine to the target node. It locks the row of the target node as a placement does ([Placement](/reference/placement/index.html.md#the-steps-of-a-placement)). A target that has no room any more gives way to another node that placement picks, unless the instance admin named it. |
| `stopped`, and its node is `unresponsive` | It waits, because the source node must serve the disk. |
| `stopped`, and a live control process runs a release of the schema version 44 to 49 | It waits. 10 min after the start of the move, the move fails with `a control process runs a hopboxd that does not know the moves`. |
| deleted, or its target is `deleted` | The move fails with `the machine was deleted during the move`. A delete wins over the stop of a move. |
| not `stopped` after 10 min, or its node is unresponsive after 10 min | The move fails. The spec gives the machine its own target again. |

### The pull

The spec of a moving machine names the move, the source node, and the disk listener of the source ([the executor protocol](/reference/executor-protocol/index.html.md#machinemove)). The executor of the target never boots such a machine before its disk is whole. A boot without the disk makes a new disk from the image. It pulls the disk first, whatever the target. A target `deleted` only removes what arrived.

The executor writes the move in its node-local state, as `pending_move`, before the first pull ([the node executor](/reference/executor/index.html.md#node-local-state)). The machine never boots on the node until the control role accepted the proof of the pull of that move. The executor keeps that move as `acked_move`. This rule holds after a restart of `hopboxd`. It also holds for a spec that names another state or no move. A control role of the schema version 44 to 49 sends such a spec, because it does not know the moves.

The source node serves the disk once its next heartbeat brings the `DiskTransfers` of the control role ([the control channel](/reference/control-channel/index.html.md#disk-transfers)). Until then, the source answers `403`. While its executor still holds the lease or the microVM runs, it answers `409`. The target tries again in both cases.

A pull runs in slices. One pull lasts at most 5 min, the timeout of an action of the executor. A pull that gets no byte for 30 s ends, and so does a cut link. The bytes that arrived stay in the partial copy, with the state of their digest, and the next pull resumes from the last whole frame. A restart of `hopboxd` on the target resumes too, from the last saved progress. So a large disk arrives over several pulls, and no step reads the whole copy again. After a pull that wrote bytes and did not end, the executor reports its state again with `pull_received`, the bytes of that pull. That report holds off the pull deadline. These causes count toward the 3 failures in a row that end the move:

- The source has no disk of the machine (`404`).
- The copy has another digest than the disk of the source.
- The target cannot write the copy, such as a full disk.

Any other failure, such as a cut link, a refusal, or a timeout, makes the executor try again until the pull ends. After the third failure, the executor reports `failed` from `moving`, with the error `pull the disk from node <id>: ...`, and stops acting with its lease.

### Pull deadline and cancel

If the target pulls no part of the disk for 30 min (`domain.MovePullTimeout`), a move in `pulling` fails with the error `the target node pulled no part of the disk for 30m0s`. The 30 min count from the hand-off, and again from each report with `pull_received`. So a large disk on a busy link keeps its move while it makes progress. The leader job checks the deadline at each step. A resumed pull can wait for the digest job of the source, and that wait writes no bytes. So the digest of a disk must take less than 30 min. An instance admin can also end a move that runs, with `CancelMove` ([the admin API](/reference/admin-api/index.html.md#cancelmove)) or `hopboxd admin machine move --cancel` ([hopboxd](/reference/hopboxd/index.html.md#admin-machine-move)). The move then fails with the error `an instance admin canceled the move`, and the service logs `an instance admin canceled the move of a machine`.

| State of the move | Result |
| --- | --- |
| `stopping` | Only a cancel ends it. The spec gives the machine its own target again on its node. |
| `pulling` | The result of a failed pull ([End of a move](#end-of-a-move)). The machine goes back to the source node with its disk, stopped. A machine with the target `deleted` goes back with that target. A machine that is no longer in `moving` on the target stays there. |

The next heartbeat of the source node withdraws the disk from its listener, and the listener ends the streams of that disk at once. The target node then removes what it pulled.

### End of a move

`ReportMachine` reads the move of the machine in the transaction of each report of the target node while the move is `pulling`. Only a report that names the move in `pulled_move_id` proves that the target has the whole disk ([the executor protocol](/reference/executor-protocol/index.html.md#machinereport)). Once the disk is here, the executor names the move in each report:

| Report | Result |
| --- | --- |
| Any state but `failed`, with the ID of the move in `pulled_move_id` | The move is `done`. The source node must remove its old copy. The service logs `a machine moved to another node`. A report of the same state ends the move too. |
| The same state, without the proof | Nothing ends. A report with `pull_received` above 0 holds off the pull deadline. |
| `failed` | The move is `failed`, with the error of the report. The control role ends the lease of the target. It gives the machine and its address back to the source node, with the current state and the target `stopped`. The target node must remove what it pulled. The service logs `the target node could not pull the disk, so the machine stays stopped on its source node`. |
| Any other step | The same result as `failed`, with the error `the target node left the state <state> with <new state>, without the whole disk of the move`. An executor of an older release gives such a report, because it boots a new disk from the image. |

The executor of the target boots the machine only after the control role accepted the report with the proof. If the move ended before, such as through a cancel, the control role refuses that report, because the lease of the target ended. So the machine never boots on the target.

An executor of an older release on the target can boot a new disk from the image before its report. The move fails, and the machine goes back to the source. That executor loses its lease, but its microVM of the new disk keeps running on the node until the node upgrades. After the upgrade, the node gets the removal of the files of the machine. The files stay while that microVM runs, and the node logs `the control plane asks this node to remove the files of a machine that it still acts on, so they stay for now`. An operator stops the systemd unit of that machine on the node.

The next heartbeat of the source node grants it the machine again. The machine stays stopped there with its disk. A start boots it from that disk.

The end of a move writes the audit entry `machine.move` with the actor `system` and `move`, `result`, and for a failure `error`. A failure that leaves the machine on its target adds `stayed`, the state of the machine.

### A step that an older control process accepted

A control process of the schema version 44 to 49 can take the channel of the target node during a pull. It accepts a step of the target out of `moving`, such as `failed`, and does not end the move. The move then stays `pulling`, and the machine is on the target in another state. The executor still waits for the proof, and pulls the disk again once a spec names the move.

A cancel, the pull deadline, or a step of the target without the proof then fail the move, and leave the machine where it is. The target can already run the machine with the pulled disk. So the control role never takes the machine back, and no two microVMs run it. The move gets no `cleanup_server_id`, and the source keeps its copy of the disk. The service logs `a cold move failed while its machine was not in the state moving on the target node, so the machine stays there. The source node keeps its copy of the disk` at level warn, with `state` and `err`. An operator decides about that copy. If the target has no whole disk, the machine never boots there, and a delete removes it. The machine does not move again while the source keeps the copy, because a move from the target could make a partial or a blank disk the machine. `MoveMachine` and a drain refuse it. The source never counts as empty for a drain, and `DecommissionServer` refuses it, also after a later move of the machine from another node. Only a delete of the machine ends the copy.

### Old copies

`cleanup_server_id` of a move names the node that still holds an old copy of the files of the machine: the source after `done`, the target after `failed`. The `DiskTransfers` of that node lists the machine in `remove`, once the machine is on another node or its row is gone. The executor of the node removes the files once it holds no valid lease of the machine and the microVM does not run: the disk, the partial copy, the node-local state, and the log. Then the next heartbeats name the machine in `removed_machines`, and the control role clears `cleanup_server_id`. A new move of the machine waits until then.

## Drain of a node

A drain takes every machine off a node, so that the node can leave the install. `DrainServer` of the admin API ([the admin API](/reference/admin-api/index.html.md#drainserver)) or `hopboxd admin server drain` ([hopboxd](/reference/hopboxd/index.html.md#admin-server-drain)) starts it. The guide [Drain a node](/guide/drain-a-node/index.html.md) shows it step by step. The drain cordons the node and sets its state to `draining` ([the control channel](/reference/control-channel/index.html.md#server-states)). Then a leader job moves each machine of the node cold to another node. A drain never deletes a machine. Every machine keeps its disk.

The code is in `internal/service/drains.go`. The leader job calls `service.AdvanceDrains` every 5 s (`control.DrainEvery`).

| Item | Value |
| --- | --- |
| Moves from the node at once | At most 2 (`service.DrainMoves`). The count includes the moves that an admin started. |
| Empty checks in a row | 3 (`service.DrainEmptyChecks`) |
| Time between two empty checks | 1 min (`service.DrainCheckEvery`) |
| Step of the leader job | Every 5 s (`control.DrainEvery`) |

### Start

`DrainServer` locks the server, and does these things in one transaction:

1. It writes a row of `server_drains` for the node ([Control store tables](/reference/control-store/index.html.md#server_drains)). A second call finds the row. It clears the stranded machines, restarts `started_at`, and counts the empty checks from zero. This is a retry.
2. It sets the cordon flag. A node in `ready` or `cordoned` turns `draining`.
3. It writes the audit entry `server.drain` with the target kind `server` and the details `old_state`, `state`, and `retry` (`true` or `false`).

A node in `unresponsive` stays `unresponsive` with the cordon flag set. It turns `draining` at its next heartbeat, because a node with a row in `server_drains` comes back as `draining`, before the cordon rule. `UncordonServer` refuses a node that drains, with `failed precondition: the server <id> drains, and stays cordoned until the drain decommissions it. A drain cannot stop. Follow it with hopboxd admin server get, and fix the reasons of its stranded machines. For a server that Terraform manages, run terraform destroy, which waits for the drain`. Placement takes only `ready` nodes, so a draining node takes no new machine.

`DrainServer` refuses these cases:

| Case | Error |
| --- | --- |
| The ID is not valid | `invalid_argument` |
| No server has the ID | `not_found` |
| The server left the install | `failed precondition: the server <id> is decommissioned` |
| The server has no role `node` | `failed precondition: the server <id> has no role node, so it holds no machine to move. Decommission it instead` |
| An admin session drains a server that shares its host with a live control process, or with the control process that takes the call | `permission denied: the server <id> hosts a control process, so only root on a control server or an admin API key drains it`. A drain ends in a decommission, and a control server goes only by root or by the Terraform provider. |
| A live control process runs a release of the schema version below 51 ([Two releases](#two-releases)) | `failed precondition: the control process <process> knows the migrations up to <n>, and only a hopboxd that knows migration 51 drains servers. Upgrade it first` |

### The machines of the node

Each run, the job reads the machines of the node in the order of their IDs. A machine that its executor has deleted does not count. For each machine, it does the first case that fits:

| Machine | What the job does |
| --- | --- |
| A move of the machine runs | It waits. |
| The target is `deleted` | It waits, with the reason `the machine <name> waits for its delete`. The delete goes by itself. |
| The current state is `pending`, and the target is `running` | It waits until the machine boots, with the reason `the machine <name> has not booted yet, and moves once it runs`. Then it can move. |
| The machine cannot move cold: it never booted with the target `stopped`, is `hibernated` or `standby`, or has another target | The machine is stranded, with the reason `a cold move needs a machine that booted, with the target running or stopped. This one is <state>, with the target <target>`. |
| The newest move of the machine from the node since the start of the drain failed | The machine is stranded, with the reason `its move failed: <error of the move>`. It stays stopped with its disk on the node. |
| Otherwise | A move starts, if fewer than 2 moves run from the node. |

A move starts as `MoveMachine` starts it ([Start of a move](#start-of-a-move)), with a target that placement picks. The audit entry is `machine.move` with the actor `system`, the same details, and the detail `drain`, the ID of the server. A machine whose last move still waits for the cleanup of an old copy waits too, with the reason `the machine <name> waits until the node <id> (<state>) removes the old copy of its last move`. The state shows a node that cannot do the cleanup now, such as an `unresponsive` one. A start that fails with `failed_precondition` or with no capacity strands the machine. Its reason is the message of that error, such as `no capacity: no node has room for a machine of size ...`.

A machine that cannot move stays on the node with its disk: a running machine with no room keeps running, a machine whose move failed stays stopped. The stop is part of the cold move, and an early stop of a machine with no room elsewhere only adds downtime.

A stranded machine is not tried again until `DrainServer` runs again. The drain also strands a machine whose disk copy a failed move left on this node ([A step that an older control process accepted](#a-step-that-an-older-control-process-accepted)). The machine is on another node. Its reason is `a failed move left the machine <state> on the node <id>, and this server keeps a copy of its disk that nothing removes (reference/cold-move.md)`. A delete of that machine ends the reason.

The drain is stuck when no move runs, every machine left on the server is stranded, and a machine is stranded or the decommission was refused. A stuck drain waits for an operator. The operator makes room or fixes the reason, then runs `DrainServer` again.

### Empty checks and the decommission

The `error` of the drain says why the drain waits now: an older control process, a node without heartbeats, the first wait of the table above, an old copy on the node, or a refused decommission. So a drain that starts no move always shows its reason. The control store keeps the time of the empty checks, so a new leader with another clock keeps the gap of 1 min.

After the moves, the job checks the node. The node counts as empty when all these things hold:

- The state is `draining` or `shutting_down`, so the node answers its heartbeats.
- No machine is on the node in the control store, except machines that their executor has deleted.
- The last heartbeat of the node lists no allocation.
- No move runs from or to the node.
- No old copy waits for the node to remove it (`cleanup_server_id`). While one waits, the reason is `the server still removes the old copy of the machine <name>`.
- The node keeps no copy of a disk after a failed move.
- No live control process runs a release of the schema version below 51.

The first empty check turns the state to `shutting_down`. It writes the audit entry `server.mark_shutting_down` with the actor `system` and the details `old_state` `draining`, `state` `shutting_down`, and `checks` `1`. The later checks change no state and write no such entry. The checks are 1 min apart, and the third one decommissions the node with all the checks of `DecommissionServer` ([the admin API](/reference/admin-api/index.html.md#decommissionserver)). The audit entry is `server.decommission` with the actor `system` and the details `old_state` `shutting_down`, `state` `decommissioned`, and `drain` `true`. The row of `server_drains` goes with the decommission.

| Event | Result |
| --- | --- |
| A check finds the node not empty | The state goes back to `draining`, and the empty checks restart from zero. The audit entry is `server.mark_draining` with the actor `system` and the details `old_state` `shutting_down`, `state` `draining`, and `reason`: `the node is not empty`, or why the drain waits. |
| `DrainServer` runs again | The state goes back to `draining`, and the empty checks restart from zero. |
| The node sends no heartbeat for 20 s | The state turns `unresponsive`, and the audit entry is `server.mark_unresponsive`. The leader job marks `draining` and `shutting_down` nodes too. The error of the drain is `the server sends no heartbeat, so the drain waits for it`. The drain goes on at the next heartbeat. |
| The third check meets a refusal of the decommission, such as `... shares its host with the last live control process ...` | The error of the drain holds the message. The empty checks stay at 3, and the next check one minute later tries again. |
| A live control process runs a release of the schema version below 51 | The error holds the refusal message of `DrainServer`. The drain starts no move and runs no check. |

`DecommissionServer` still works on a `draining` or `shutting_down` node that is empty. Its refusals now name the cause: the machines, the move, or the kept copy ([the admin API](/reference/admin-api/index.html.md#decommissionserver)). A decommission also drops the old copies that the server still had to remove. It clears `cleanup_server_id` of those moves, so that a new move of those machines does not wait.

### Logs of the drain

The control role logs with `component=service`:

| Message | Level | Fields |
| --- | --- | --- |
| `the drain of a server starts` | info | `server`, `retry` |
| `the drain of a server moves a machine` | info | `server`, `machine`, `move`, `target` |
| `the drain of a server cannot move a machine, so the machine stays with its disk` | warn | `server`, `machine`, `reason` |
| `a step of the drain of a server failed. The next run tries again` | warn | `server`, `err` |
| `the drain found a server empty` | info | `server`, `checks` |
| `the drain decommissioned a server` | info | `server` |
| `the drain cannot decommission a server that it found empty` | warn | `server`, `err` |

The leader logs `taking the drains on failed` at level warn with `err`, with `component=leader`.

### The subnet of a decommissioned node

The leader sweep frees the subnet of a decommissioned node, with the other sweeps ([Control store tables](/reference/control-store/index.html.md#node_subnets)). A subnet is free once no machine address of it is held. An address is held when `held_until` is null or in the future. A moved machine keeps the address of its first node, so the subnet stays taken while such a machine lives. The next node that gets its first machine takes the first free subnet, which can be the freed one ([the node network](/reference/node-network/index.html.md)). The sweep log `freed reserved machine names and ports` has the field `subnets` for the count.

## Removal of the node role

The removal of the role `node` from a running server moves its machines off as a drain does. But the server stays in the install with its other roles. An example is an edge server that keeps `edge`. `RemoveServerRoles` of the admin API ([the admin API](/reference/admin-api/index.html.md#removeserverroles)) or `hopboxd admin server remove-role <server> node` ([hopboxd](/reference/hopboxd/index.html.md#admin-server-remove-role)) starts it. The call cordons the server, and the state turns `cordoned`, not `draining`.

The code is in `internal/service/removals.go`. The leader job calls `service.AdvanceRemovals` every 5 s, after `service.AdvanceDrains`. Each run does what a drain does with the machines of the node ([The machines of the node](#the-machines-of-the-node)): at most 2 moves at once, the same waits, and the same reasons of a stranded machine. The rows of `removal_strands` name the stranded machines, and `removal_error` says why the removal waits ([Control store tables](/reference/control-store/index.html.md#removal_strands)). The audit entry of each move is `machine.move` with the actor `system` and the detail `removal`, the ID of the server. Each run also sets the cordon again, when a control process of an older release lifted it.

These things differ from a drain:

- No empty check and no decommission come at the end. The removal is ready once the server holds no machine, no allocation, and no move from or to it. It must also hold no old copy of a moved machine, and no copy of a disk that a failed move left. An enrollment of the server with the roles that stay then ends the removal ([Enrollment and the install PKI](/reference/enrollment/index.html.md#remove-roles-from-an-enrolled-server)).
- A stranded machine stays stranded until the cancel of the removal. A second `RemoveServerRoles` with the same roles returns the pending removal, and changes nothing.
- `CancelRoleRemoval` ends a removal that cannot finish. Each move from the server that started with the removal and still runs ends as `CancelMove` ends it, with the error `an instance admin canceled the removal of the node role`. A machine that moved stays on its new node ([the admin API](/reference/admin-api/index.html.md#cancelroleremoval)).
- The server keeps its row in `node_subnets`, because each moved machine keeps the address of its first node ([the node network](/reference/node-network/index.html.md)).

### Logs of the removal

The control role logs with `component=service`:

| Message | Level | Fields |
| --- | --- | --- |
| `the removal of roles from a server starts` | info | `server`, `roles`, `cordoned_before` |
| `the removal of the node role moves a machine` | info | `server`, `machine`, `move`, `target` |
| `the removal of the node role cannot move a machine, so the machine stays with its disk` | warn | `server`, `machine`, `reason` |
| `a step of the removal of the node role failed. The next run tries again` | warn | `server`, `err` |
| `an instance admin canceled the removal of roles from a server` | info | `server`, `roles`, `moved`, `canceled_moves` |

The leader logs `taking the removals of the node role on failed` at level warn with `err`, with `component=leader`.

## What the machine keeps

| Item | After the move |
| --- | --- |
| Disk | The same bytes, with the same holes. |
| Memory | Lost. A running machine boots again on the target. |
| Name, URL, SSH port | The same. |
| Address | The same address of the subnet of its first node, and the same IPv6 address. The target node routes the `/32` and the `/128` to the tap device of the machine. The mesh peer of the target allows both ([the node network](/reference/node-network/index.html.md#moved-machines), [the mesh](/reference/mesh/index.html.md#allowed-ips)). |
| Route record | Dropped at the stop, and written again with the lease of the target at the report of `running`. So no traffic goes to the source after the move. |
| Target state | The same. During the stop, the spec of the source has the target `stopped`, and the machine row keeps its own target. So the usage of its org and of its node stay as they are. |
| Lifecycle calls | They change the machine row as usual. During the stop, the spec stays `stopped`, apart from a delete. On the target, the machine reaches the target of the row once its disk is whole. |

## The disk listener

Three things let a node run the disk listener on port `7444` of its mesh address. Its driver keeps its disks in files, and the node has an identity and a mesh address. Once the node knows its mesh address, the listener listens on it. A check every 2 s finds a new address, and the listener then listens on it.

### Access to a disk

Both sides of a pull use their server certificates of the install CA ([Enrollment and the install PKI](/reference/enrollment/index.html.md)):

1. The listener asks each client for its certificate. The certificate must chain to the install CA, have the usage client auth, and name a server with the role `node`. Any other client fails the TLS handshake.
2. The target checks the certificate of the listener the same way, and checks that it names the source node of the spec. The mesh address of a node is in no certificate, so this check replaces the check of the host name.
3. The listener serves the disk of a machine only to one node, and only with one of its certificates. That node is the target in `serve` of the last `DiskTransfers` of the control role. The serial of the certificate must be in `target_serials` of that entry. The control role lists a disk there only while its move is `pulling`, with the target of that move. `target_serials` holds the serials of the certificates of the target that are not retired and not expired.

So a node can pull only the disk of a machine whose lease the control role granted to it for a move. A pull of another machine, a pull by another node, or a pull with a retired certificate gets `403`. When the next `DiskTransfers` no longer lists a disk, the listener ends each stream of that disk at once. The listener never serves the disk of a machine whose executor holds a valid lease or whose microVM runs.

### Requests

```text
GET https://<mesh address>:7444/v1/disks/<machine ID>?offset=<bytes>
```

| Status | When |
| --- | --- |
| `200` | The stream of the disk from `offset` follows. |
| `400` | The path names no machine, or `offset` is not a byte offset. |
| `401` | The request has no certificate of a node. |
| `403` | The control role names another node, or none, for the disk, or the certificate of the request is not in `target_serials`. |
| `404` | The node has no disk of the machine. |
| `409` | The machine still runs on the node, or its executor holds its lease. |
| `416` | `offset` is past the end of the disk. |
| `500` | The node cannot read the disk. |
| `503` | The node has no identity yet. |

| Header of `200` | Value |
| --- | --- |
| `Content-Type` | `application/vnd.hopbox.disk` |
| `Hopbox-Disk-Size` | The size of the disk in bytes. |
| `Hopbox-Disk-Version` | The version of the disk file: `<inode>-<time of change in ns>-<size>`. |
| `Trailer` | `Hopbox-Disk-Digest` |

| Trailer of `200` | Value |
| --- | --- |
| `Hopbox-Disk-Digest` | The digest of the whole disk, `sha256:<hex>`. It comes after the end frame. A stream that ends early has no digest. |

The listener sends the headers at once, before it reads the disk. So no digest delays the start of a stream.

### The stream

The body is a list of frames. Each frame is a header of 12 bytes, then its data:

| Bytes | Value |
| --- | --- |
| 0 to 7 | The offset of the data in the disk, big-endian. |
| 8 to 11 | The length of the data, at most 1 MiB, big-endian. |

The frames come in the order of their offsets, from `offset` rounded down to a block. Each frame holds blocks that are not all zeros, so the holes of the disk and its blocks of zeros never cross the mesh. The listener skips the holes that the file system reports with `SEEK_DATA` and `SEEK_HOLE`, and reads the rest. A frame with the length 0 and the offset of the size of the disk ends the stream. A stream without it was cut.

The target writes each frame at its offset in the partial copy `<disk>.pull`, which it first makes the size of the disk. So the copy keeps the holes. The file `<disk>.pull.json` holds the progress. It has the move, the version of the disk file of the source, and the size. It also has the offset up to which the copy is on disk, and the digest state of the copy at that offset. A pull of another move, or a source whose size or version changed, starts again from zero.

### Digest

The digest of a disk is one sha256. It takes each block that holds a byte other than zero, as its offset in 8 bytes big-endian and then its bytes. Then it takes the size of the disk in 8 bytes big-endian. A hole and a block of zeros give the same digest, so both nodes agree whatever their file systems allocate.

A stream from offset 0 computes the digest of the source while it sends the frames. A stream from another offset needs the digest of the whole disk. For such a stream, the source starts a digest job apart from the request. When the job ends, the source sends the trailer. The job outlives the request, so a pull that stops early finds the digest later. The source keeps each digest for the version of its disk file while the control role lists the disk.

The target adds each frame to the digest of its copy as it writes it, and keeps the state of that digest with the progress. After the end frame, the target waits for the trailer without an idle timeout. When both digests match, it renames the copy to the disk file of the machine, and syncs the dir. Otherwise, it drops the copy.

## Two releases

If the heartbeats of a node say `disk_transfer`, the node takes part in moves ([the control channel](/reference/control-channel/index.html.md#heartbeat)). The control role keeps the version of `hopboxd` of the last such heartbeat in `servers.disk_transfer_version` ([Control store tables](/reference/control-store/index.html.md#servers)). A node is a source or a target only while two things hold. That version is the current version of the node, and the node is in the mesh. So a node that went back to a release without moves takes no move.

A release of the schema version 44 to 49 runs beside migration 0050 ([Control store tables](/reference/control-store/index.html.md#machine_moves)). Such a control role sends the spec of a moving machine without `move`. Its lifecycle calls do not know the moves. Two checks keep the disk of a machine safe in that case:

1. A live row of `control_processes` can have a `schema_version` below 50 (`service.MovesSchema`, [Control store tables](/reference/control-store/index.html.md#control_processes)). While it does, the control role refuses `MoveMachine` and holds the hand-off of each move. A control process of a release before migration 0045 writes no row, so upgrade every control server before the first move.
2. The target node keeps the move in its node-local state. It never boots the machine until it has the whole disk of that move, whatever the spec says ([The pull](#the-pull)).

An executor of an older release on the target can boot a new disk from the image, and report `stopped` or `running`. That report has no `pulled_move_id`, so the move fails and the source keeps the disk ([End of a move](#end-of-a-move)).

The drain needs migration 0051 (`service.DrainsSchema`, [Control store tables](/reference/control-store/index.html.md#server_drains)). A release of the schema version 44 to 50 runs beside it. It never reads the new tables, runs no drain job, and shows `draining` and `shutting_down` as unknown. It places nothing on such a node. It keeps the state in its heartbeats and never marks such a node unresponsive. Its uncordon can clear the cordon flag, which the next step of the drain sets again. Its hand-off of a move from a draining source waits and fails after 10 min. So while such a control process is live, `DrainServer` refuses, and the drain job starts no move, runs no check, and decommissions nothing ([Start](#start)).

The removal of a role needs migration 0056 (`service.RemovalsSchema`, [Control store tables](/reference/control-store/index.html.md#removal_strands)). A release of the schema version 44 to 55 runs beside it. It never reads the removal, and its enrollment refuses a list that drops a role. Its uncordon can clear the cordon of a removal of `node`, which the next step of the removal sets again. So while such a control process is live, `RemoveServerRoles` refuses, and the removal job starts no move.

## Metrics

| Metric | Type | Labels | Description |
| --- | --- | --- | --- |
| `hopbox_disk_transfer_bytes_total` | counter | `direction` | The bytes of disk data of the pulls. `sent` counts what the node sent to other nodes. `received` counts what it pulled from another node. |

The pulls count in `hopbox_executor_actions_total` with the action `pull` ([the node executor](/reference/executor/index.html.md#metrics)).

## Logs

The control role logs with `component=service`:

| Message | Level | Fields |
| --- | --- | --- |
| `a cold move of a machine starts` | info | `machine`, `move`, `source`, `target` |
| `handed a stopped machine to the target node of its move` | info | `move`, `machine`, `source`, `target` |
| `a cold move failed, so the machine stays on its node` | warn | `move`, `machine`, `source`, `target`, `err` |
| `a step of a cold move failed. The next run tries again` | warn | `move`, `machine`, `err` |
| `a machine moved to another node` | info | `machine`, `move`, `source`, `target` |
| `a cold move failed while its machine was not in the state moving on the target node, so the machine stays there. The source node keeps its copy of the disk` | warn | `machine`, `move`, `source`, `target`, `state`, `err` |
| `an instance admin canceled the move of a machine` | info | `machine`, `move`, `source`, `target` |
| `the target node could not pull the disk, so the machine stays stopped on its source node` | warn | `machine`, `move`, `source`, `target`, `err` |
| `a node removed the old files of a machine that left it` | info | `machine`, `move`, `server` |

| Message | Level | Component | Fields |
| --- | --- | --- | --- |
| `taking the cold moves on failed` | warn | `leader` | `err` |
| `reading the moves of a node failed. The next heartbeat tries again` | warn | `channel` | `server`, `err` |
| `the control plane sent the disks of cold moves to a server that takes part in no move` | warn | `node` | |

The executor logs with `component=executor` and `machine`:

| Message | Level | Fields |
| --- | --- | --- |
| `the disk of the machine is on this node` | info | `move`, `source`, `size` |
| `the pull of the disk stopped, and the executor tries again` | warn | `move`, `source`, `received`, `err`, `retry_in` |
| `the pull of the disk failed for good, so the machine goes back to its source node` | error | `move`, `source`, `err` |
| `the spec names no valid source node of the disk` | error | `source` |
| `removed the files of a machine that left this node` | info | |
| `removing the files of a machine that left this node failed` | warn | `err`. The next step tries again. |
| `the control plane asks this node to remove the files of a machine that it still acts on, so they stay for now` | warn | |
| `cannot remove the node-local state of a machine that left this node` | error | `err` |
| `cannot remove the log of a machine that left this node` | error | `err` |

A moving machine whose spec names no source logs `the executor cannot reach the target` at level warn, with the reason `the machine moves to this node, but the spec names no source node of its disk, or this node pulls no disk`.

The disk listener and the pulls log with `component=transfer` and `machine`. The listener adds `peer`, the pulling node, and a pull adds `source`:

| Message | Level | Fields |
| --- | --- | --- |
| `the disk listener serves the pulls of disks` | info | `address`. No `machine`. |
| `cannot listen for the pulls of disks. The next check tries again` | error | `address`, `port`, `err`, `retry_in`. No `machine`. |
| `a node pulls the disk of a machine` | info | `offset`, `size` |
| `a node pulled the disk of a machine` | info | `sent`, `size`, `duration` |
| `the pull of a disk ended early` | warn | `sent`, `err` |
| `refused the pull of a disk by a node that the control plane did not name for it` | warn | |
| `a node pulls the disk of a machine that has no disk on this node` | error | |
| `cannot open the disk of a machine for a pull` | error | `err` |
| `cannot compute the digest of a disk` | error | `err`. The error `transfer: the disk changed during the pull` means that the disk file changed during the digest job. The next pull then starts again from zero. |
| `pulling the disk of a machine` | info | `size` |
| `resumed the pull of a disk` | info | `offset`, `size` |
| `pulled the disk of a machine` | info | `size`, `received`, `digest`, `duration` |
| `the disk of the source changed since the last pull, so the pull starts again from zero` | warn | |
| `the control plane named a disk to serve without valid IDs` | warn | `machine`, `target` |

## Errors

| Error | Cause |
| --- | --- |
| `transfer: the source node does not serve the disk now: ...` | `403` or `409`. The executor tries again. |
| `transfer: the source node has no disk of the machine: ...` | `404`. It counts toward the end of the move. |
| `transfer: the copy of the disk has another digest than the disk of the source: ...` | The copy is dropped. It counts toward the end of the move. |
| `transfer: cannot write the copy of the disk on this node: ...` | A local failure, such as a full disk. It counts toward the end of the move. |
| `transfer: the peer is not a node of this install: ...` | A certificate that does not chain to the install CA, names no node, or names another node than the source. |
| `transfer: the source sent nothing for 30s` | The idle timeout of a pull. |
| `transfer: the source sent no valid size or version of the disk` | The headers of the source have no size or no version. The executor tries again. |
| `transfer: the source sent no digest of the disk` | The stream ended without the trailer. The executor tries again from the saved progress. |

## Tests

| Test | Where it runs | What it checks |
| --- | --- | --- |
| The tests of `internal/node/transfer` | `make test`, also on a Mac | The digest of holes and zeros, and a pull that keeps the holes. The refusals of the listener, a node of another install, a server without the node role, and a retired certificate. A cut pull that resumes, a changed disk and another move that start from zero, and the idle timeout. A slow frame and a rate limit that the idle timeout does not end, and a stall in the middle of a frame that it ends. A digest that takes longer than the timeout of a pull, a digest that does not match, and a withdrawn disk that ends its stream. |
| The tests of `internal/node/executor` | `make test`, also on a Mac | A moving machine that never boots without its disk, also after a restart and with a spec of an older control role. A machine that boots only after the control role accepted the proof of its pull, also after an older control process accepted `failed`. `Busy`, and a removal that waits until the node stops acting on the machine. |
| `TestColdMove`, `TestColdMoveFails`, `TestMoveRefusals`, `TestDeleteDuringAMove`, `TestColdMoveNeedsTheProofOfThePull`, `TestCancelMove`, `TestPullDeadline`, `TestMovesWaitForEveryControlProcess`, `TestMoveAfterAnOldControlAcceptedTheTarget` of `internal/service` | `make test` | The steps of a move in the control store. The spec of the stop, the hand-off, the lease, the address, the mesh, `DiskTransfers`, the end, and the cleanup. A failed stop, a failed pull, the refusals, and a delete during a move. A target that leaves `moving` without the proof of the pull, a cancel, and the pull deadline with its progress. A control process of an older release, also one that accepted a step of the target. |
| `TestColdMoveEndToEnd` of `internal/control` | `make test` | Two nodes over mutual TLS on loopback, with their executors, the test driver, and the disk listener. A running machine moves with the data of its disk, its name, its URL, its SSH port, and its address. The source removes its old copy. A move back to a node that cannot write the disk fails, and the machine stays stopped with its disk. |
| `TestColdMove` of `test/fleet` | `make test-fleet` | Two nodes of the test driver on `inet0` with the real mesh on their peer addresses. A cut of the `uplink` of the target in the middle of the pull, and the pull resumes. The data, the name, the URL, the SSH port, and the address stay. A failed target keeps the disk on the source. |
| `TestColdMoveKVM` of `test/fleet` | `make test-fleet` with `FLEET_KVM_BUNDLE`, on a host with `/dev/kvm` | Two KVM nodes with the real Firecracker. A file that the guest wrote is there after the move, and the address in the guest stays. It skips without KVM. |
| `TestDrain`, `TestDrainWithoutRoom`, `TestDrainStrandsAFailedMove`, `TestDrainKeepsACopyOfADisk`, `TestDecommissionWaitsForAMove`, `TestDrainWaits`, `TestDrainOfTheLastControlServer`, `TestSubnetOfADecommissionedNode` of `internal/service` | `make test` | The drain in the control store: the states, the moves at most 2 at once, the stranded machines and their reasons, the empty checks, the decommission, the refusals, and the subnet of a decommissioned node. |
| `TestDrainEndToEnd` of `internal/control` | `make test` | Two nodes over mutual TLS on loopback. A running and a stopped machine move with their data and their address, and the drain decommissions the node. |
| `TestDrain` of `test/fleet` | `make test-fleet` | With no room, the drain strands all three machines. Then a drain with a failed pull of box3 strands box3, while box1 and box2 move, at most 2 at once. A retry moves box3, and 3 empty checks a minute apart decommission node1. |
| `TestDrainKVM` of `test/fleet` | `make test-fleet` with `FLEET_KVM_BUNDLE`, on a host with `/dev/kvm` | KVM nodes with the real Firecracker. It skips without `/dev/kvm` and `FLEET_KVM_BUNDLE`. |
| `TestRemoveEdgeAndDNS`, `TestTwoRemovalsSeeEachOther`, `TestRemoveServerRolesRefusals`, `TestRemoveNode`, `TestRemoveNodeWithoutRoom`, `TestCancelEndsTheMovesOfTheRemoval`, `TestRemovalStartsNoMoveBesideAnOldControlProcess`, `TestEnrollEndsTheRemovalOfEdgeAndDNS`, `TestEnrollEndsTheRemovalOfNode`, `TestTokenAddsOnlyARoleThatTheServerLost`, `TestEnrollRefusesAnAddAndADropAtOnce`, `TestAnAddBesideTheEndOfARemovalAddsNothing`, `TestRemovalOfNodeWaitsForTheOldCopy`, `TestRemovalOfNodeWaitsForAKeptCopy`, `TestAddWaitsForAPendingRemoval`, `TestEndNeedsATokenForTheRolesThatStay`, `TestDecommissionEndsTheRemoval`, `TestRemoveServerRolesRefusesADrainingServer`, `TestLeaderCordonsTheRemovalAgain`, `TestCancelKeepsTheMovesOfAnAdmin` of `internal/service` | `make test` | The removal of a role in the control store. The tests cover each refusal of the start, the moves of node at most 2 at once, and the stranded machines. They cover the cancel with its cordon and the moves of an admin, and the end through `Enroll` with its refusals and an unused token. They also cover the leftovers of a node, the old certificate, `removed_roles`, the gate of an add, and the mesh peer without the subnet. |
| `TestRemoveNodeFromAnEdge` of `test/fleet` | `make test-fleet` | The removal of node from an edge with three machines. The test covers the moves, a stranded machine, a cancel, and the end. It covers the add of node back with the same subnet, and a token for a role that the server never lost. |
