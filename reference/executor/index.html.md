# The node executor

> The executor of each machine on its node: its actions, the guest link for exec and file copy, the restart policy, saved state, node-local state, logs, and metrics.

The executor of a machine runs on the node of the machine. It is the only code that changes the microVM of the machine. It compares the target state that the control role sends with what the node holds, and it takes one action at a time. It reports each step of the current state back. [The executor protocol](/reference/executor-protocol/index.html.md) describes the messages. The code is in `internal/node/executor`, and it calls the VMM through the driver interface of `internal/node/vmm` ([the VMM driver interface](/reference/vmm/index.html.md)).

On a server with the node role, `hopboxd run` starts the executors with the Firecracker driver ([the Firecracker driver](/reference/firecracker-driver/index.html.md)), in the unit `machines` ([hopboxd](/reference/hopboxd/index.html.md#run)). Their state dir is `<data_dir>/executor`. The tests run the executors with the test driver too ([the VMM driver interface](/reference/vmm/index.html.md#the-test-driver)).

## One executor per machine

The `Manager` of a node runs one goroutine for each machine that the control role names, and for each machine of the node-local state. The goroutine does one action at a time, so a stop and a delete that come at once run one after the other. The executor reads the newest spec before each action.

The executor acts only with a valid lease. A lease is valid while all of these hold:

- The control role named the grant in a `ReconcileMachine`, or a heartbeat renewed it.
- The control role did not refuse it. A refusal is a heartbeat answer that did not renew it, or a report with the outcome stale.
- Its time did not run out. The time is the arrival of the spec plus `lease_remaining`, or the send time of the last heartbeat that renewed it plus 30 s. In both cases, the executor stops 3 s early (`LeaseMargin`).

Without a valid lease, the executor takes no action and sends no report. An action that runs at the loss of the lease ends at once. A boot that ends this way leaves no microVM. A stop that ends this way kills the microVM at once, as the end of the context of `Instance.Stop` asks the driver ([the VMM driver interface](/reference/vmm/index.html.md#instance)). A microVM that runs keeps running. A new grant lets the executor act again. It then starts from the current state that the control role sends with the grant.

| Log message | Level | Fields |
| --- | --- | --- |
| `acting with a new lease` | info | `machine`, `epoch` |
| `lost the machine lease, so the executor stops acting` | warn | `machine`, `epoch`, `reason` |
| `the machine lease expired, so the executor stops acting` | warn | `machine`, `epoch` |
| `the control plane sent a machine spec without a valid lease or spec` | warn | `err` |

Every log record of the executor has `component=executor`, apart from the records of the guest link server (see [The guest link](#the-guest-link)). Each record of the executor of one machine also has the field `machine`. The records of the `Manager`, such as a spec without a valid lease, have no such field. No record holds the spec, because the spec holds the secrets of the machine.

## Actions

| Target | What the node holds | Action | Reports |
| --- | --- | --- | --- |
| `running` | no microVM, no memory file | boot | `starting`, then `running` |
| `running` | no microVM, a memory file | wake: a boot from the memory file | `starting`, then `running` |
| `running` | a microVM, and a reboot that a request asked for | reboot: a stop, then a boot | `starting`, then `running` |
| `running` | a microVM without a served guest link, after a restart of `hopboxd` | attach the guest link with a new key | `running`. The report goes only for another state in the control store. |
| `stopped` | a microVM | stop | `stopped` |
| `stopped` | no microVM, and the state `pending`, `starting`, or `failed` | none | `stopped`. A memory file stays. |
| `stopped` | the state `hibernated` | none | none. A hibernated machine never steps to `stopped`, because that step drops its memory file. |
| `deleted` | anything | delete: a stop of the microVM, then the removal of the disk and the memory file | `deleted` |
| `hibernated` | a microVM, and a driver with the capability snapshot | hibernate | `hibernated` |
| `hibernated` or `standby` | a driver without the capability | none | none. The executor logs `the executor cannot reach the target` at level warn, with `target` and `reason`. |
| any but `deleted` | a move in `pending_move` whose disk is not here yet, and a spec with `move` | pull: a copy of the disk from the source node of the move ([Cold move](/reference/cold-move/index.html.md#the-pull)) | none, or `failed` after 3 pulls that fail for good |
| `running` | the disk of the move in `pending_move` is here, without the proof that the control role accepted | report the proof of the pull, then boot once the control role accepted it | `stopped` with `pulled_move_id`, then `starting` and `running` |
| `stopped`, `hibernated`, or `standby` | the same | report the proof of the pull | `stopped` with `pulled_move_id`. A machine that the control store has `stopped` already gets a report of `stopped` again. |
| `deleted` | the state `moving` | delete: the removal of what arrived | `stopped`, then `deleted` |
| any but `deleted` | a move in `pending_move` whose disk is not here yet, and a spec without `move` | none | none. The executor logs `the executor cannot reach the target` at level warn, with `target` and `reason`. |

The executor never boots a machine of a move before its disk is whole. A boot without the disk makes a new disk from the image. The move stays in `pending_move` of the node-local state until the control role accepted a report with the proof of its pull, `pulled_move_id`. The executor then writes the move in `acked_move`. Until then, the machine never boots on the node, whatever its current state. This rule holds after a restart of `hopboxd`, and on a spec of a control role that does not know the move. It also holds after such a control role accepted a step out of `moving`. If the control role ended the move before it accepted the proof, it refuses that report, and the machine never boots on the node. Once the disk is here, each report of the executor names the move in `pulled_move_id`. A pull can write bytes and end early. The executor then reports its current state again with `pull_received`, which holds off the pull deadline of the move. After a pull that failed for good, the executor reports `failed` and acts no more with its grant: the control role gives the machine back to its source node.

A boot or a wake starts only after the control role answered each report that waits in the queue. Such an answer can find the grant stale, for example the answer to the report of a crash. A boot with a stale grant can run the machine beside the executor of the live grant. So a restart after a crash waits for the answer to `failed`. If the control channel is down at the crash, the machine does not restart on the node until the channel is back. When the answer arrives, the machine restarts with the same grant. If the lease ended in the meantime, the machine waits for the next grant, and restarts with it. The wait for that grant can add up to the rest of the lease, about 30 s (`domain.MachineLeaseTTL`). A boot or a wake succeeds once the guest agent applied the configuration of the machine on an authenticated guest link ([Guest link](/reference/guest-link/index.html.md#node-api)). Only then does the executor report `running`. A boot that fails reports `failed` with the error. If the microVM runs but its guest agent does not come up, the executor stops the microVM first.

When a report needs a step that the lifecycle does not allow, the executor reports through one state between. For example, a delete of a machine in `starting` reports `stopped`, then `deleted` ([Domain rules](/reference/domain-rules/index.html.md#steps-between-current-states)).

Each action has a timeout of 5 min (`DefaultActionTimeout`). An action that fails without a state to report, such as a delete that the driver refuses, runs again 5 s later.

| Log message | Level | Fields |
| --- | --- | --- |
| `machine state changed` | info | `from`, `to`, `boot_generation` |
| `a machine action failed` | warn | `boot_generation`, `err` |
| `stopping the machine failed`, `deleting the machine failed`, `hibernating the machine failed` | warn | `err` |
| `the guest agent of the running machine did not come back` | warn | `err` |
| `cannot read the machine from the VMM driver` | warn | `err`. Before its first action, the executor asks the driver for the microVM. It tries again 5 s later. |
| `cannot stop a machine whose guest agent did not come up` | error | `err` |
| `stopping the machine for its delete failed` | warn | `err`. The delete runs again 5 s later. |
| `the VMM driver declares the capability snapshot but cannot snapshot` | error | `driver`. The hibernate runs again 5 s later. |
| `the lifecycle has no steps between two states` | error | `from`, `to`. The executor sends no report. |

## The guest link

Each boot gets its own link key of 32 random bytes. The executor serves the guest link on the socket that the driver names. The driver puts the key in the metadata service of the microVM ([Guest link](/reference/guest-link/index.html.md#link-key)). The configuration of the guest link comes from the spec:

| Configuration | From |
| --- | --- |
| `hostname` | `name` |
| `env` | `env` |
| `authorized_keys` | `authorized_keys` |
| `start_command` | `start_command` |
| `network`, `nameservers` | `network`. Without an address, the guest agent leaves the network as it is. A spec with an address and without resolvers gets the resolvers of the node, from `Config.Nameservers` ([the node network](/reference/node-network/index.html.md#the-resolvers-of-the-machines)). |

The executor also reports the SSH host keys of the guest with `running`. After a change of the keys, it reports `running` again. The SSH front door accepts only these keys ([the SSH front door](/reference/ssh-front-door/index.html.md#connection-to-the-machine)). The guest agent leaves out an env value whose name breaks its rules, and counts it in its answer. The executor logs `the guest agent left env values out of the machine` at level warn, with `count`, and reports the count with `running` ([the executor protocol](/reference/executor-protocol/index.html.md#machinereport)). The machine API shows it as `skipped_env_vars`. A running machine gets new env vars and new authorized keys at once: the executor pushes them to the guest agent, and logs `pushed a new configuration to the guest agent`. A share of the machine can change its env vars, for example. A new start command and a new host name apply at the next boot. A new restart policy applies at once. After a restart of `hopboxd`, the executor serves the guest link again with the current spec and a new key. It gives the new key to the metadata service first. The guest agent reads the key before each handshake, so it connects again.

| Log message | Level | Fields |
| --- | --- | --- |
| `cannot push the new configuration to the guest agent` | warn | `err`. The push runs again 5 s later. |
| `cannot give the machine a new link key` | warn | `err`. After a restart of `hopboxd`, the attach runs again 5 s later. |
| `cannot serve the guest link` | warn | `err`. After a restart of `hopboxd`, the attach runs again 5 s later. |
| `the guest link stopped` | warn | `err`. The server of the guest link ended with an error. |

The server of the guest link, `internal/node/guestlink`, writes these records with `component=guestlink` and `machine`. They have no `component=executor`. Each record has one `component` key, so every JSON reader gets the same value.

| Log message | Level | Fields |
| --- | --- | --- |
| `guest link up` | info | `version`, `agent_version` |
| `guest link handshake failed` | warn | `err` |
| `guest link down` | warn | `err` |
| `push the configuration` | warn | `generation`, `err` |
| `the guest agent failed to apply the configuration` | error | `generation`, `err` |
| `ignored a guest link message that this node does not know` | debug | |

## Exec, file copy, and logs

The executor serves the exec and the file copy of the machine API on the guest link ([the machine API](/reference/machine-api/index.html.md#access-calls)). `Manager.Guest(id)` returns the guest link of a machine. The handlers of the machine streams call it ([the control channel](/reference/control-channel/index.html.md#machine-streams)). It returns the link only in these cases:

| Case | Result |
| --- | --- |
| The executor has no machine with the ID. | `executor.ErrNoMachine`: `this node does not run the machine` |
| The executor holds no valid lease of the machine. | an error that wraps `ErrNoMachine` |
| The machine does not run, or its guest agent did not come up yet. | `executor.ErrNotRunning`: `the machine is not running` |
| The machine runs, and its guest agent came up. | the guest link of the boot |

The executor withdraws the link before it closes it, at each stop, crash, reboot, hibernate, and delete. A call on a link that closes ends with an error.

The guest agent sends each line of output of the start command over the guest link. The executor hands each line to `Config.Logs`, such as a `machinelog.Store`, and never waits for it ([Machine logs](/reference/machine-logs/index.html.md)). The executor drops the log of a machine once the control role accepted its delete.

On a server with the node role, `hopboxd run` sets `node.Agent.Handlers` to `access.Handlers{Guests: manager, Logs: store}.Map()`, with the logs in `<data_dir>/logs`. `machines.Machines` of `internal/node/machines` builds both, and its unit runs the executors and the logs. The unit writes these records, without a component:

| Log message | Level | Fields |
| --- | --- | --- |
| `the kernel cache has the guest kernel of the release` | info | `kernel`, `sha256` |
| `cannot add the guest kernel of the release to the kernel cache` | warn | `kernel`, `err`. The node still boots the machines whose kernel is in the cache. |
| `the executors stopped` | error | `err` |
| `the machine logs stopped` | error | `err` |

## Restart policy

A machine can stop by itself: a crash, a shutdown or a reboot from inside, or a boot that failed. With the target `running`, the restart policy then decides.

| Event | `always` | `never` |
| --- | --- | --- |
| A crash | The executor reports `failed`, waits for the backoff, and boots again. | The executor reports `failed`. The machine stays down. A start asks for a new boot ([the machine API](/reference/machine-api/index.html.md#startmachine)). |
| A shutdown from inside | The executor reports `stopped`, waits for the backoff, and boots again. | The executor reports `stopped` with `stopped_by_itself`. The control role then sets the target to `stopped`. The machine stays down until a request. A later start asks for a new boot ([the machine API](/reference/machine-api/index.html.md#startmachine)). |
| A failed boot or wake | The executor reports `failed`, waits for the backoff, and tries again. | The executor reports `failed`. The machine stays down until a start. |
| A node reboot | The node shuts the guest down cleanly after the stop of `hopboxd`, so the executor sees nothing and reports nothing ([the Firecracker driver](/reference/firecracker-driver/index.html.md#a-stop-through-systemd)). The microVM does not run after the restart of `hopboxd`, so the executor boots the machine. | The same. The target `running` asks for it. |
| A `systemctl stop` of the unit of the machine | If the guest shut down in time, the same as a shutdown from inside. If systemd had to kill Firecracker, the same as a crash. | The same. |
| An exit of the start command | The executor waits for the backoff, and asks the guest agent to run the command again. | The command stays stopped. |

The backoff starts at 1 s and doubles after each restart, up to 5 min (`domain.RestartBackoff`). A run of at least 5 min starts the count again (`DefaultStableRun`). The start command has a count of its own. A request that asks for a boot skips the backoff. Such a request is a reboot, or a start of a failed machine. A change of the target also ends the backoff.

| Log message | Level | Fields |
| --- | --- | --- |
| `the machine crashed` | warn | `err` |
| `the machine shut down from inside` | info | |
| `the machine starts again after a backoff` | info | `restart`, `wait` |
| `the start command exited, and runs again after a backoff` | info | `exit_code`, `restart`, `wait` |
| `ran the start command again` | info | |
| `cannot run the start command again` | warn | `err` |

## Saved state

A failure never deletes saved state. The executor follows the contract of [Domain rules](/reference/domain-rules/index.html.md#executor-contract-after-a-failed-wake):

1. A wake that fails reports `failed`. The memory file stays.
2. A stop then steps from `failed` to `stopped`. The memory file stays.
3. A later start wakes the machine from the kept memory file.
4. Only a delete drops the memory file.

## Node-local state

The executor keeps one file per machine in its state dir, `<dir>/<machine ID>.json`, with mode `0600`. The file exists only so that a restarted `hopboxd` can attach to its microVMs again and clean up after them. The control store stays the source of truth. The file holds no secret.

| Field | Description |
| --- | --- |
| `machine_id` | The ID of the machine. |
| `epoch`, `token` | The grant that the executor acted with last. After a restart, the next heartbeat lists it. It counts as valid only after a heartbeat renews it, or after a new spec names it. |
| `boot_generation` | The reboot generation of the last boot, so a restart never reboots a running machine again. |
| `pulled_move` | The move whose disk the node pulled whole, so a restart before the report of the move pulls it no more. Absent for a machine that never moved here. |
| `pending_move` | The move whose disk the node waits for, or `unknown` for a moving machine whose spec names no move. Absent for a machine that never moved here. |
| `acked_move` | The move whose proof of the pull the control role accepted. Until it names the move of `pending_move`, the machine never boots on the node. Absent for a machine that never moved here. |

The executor writes the file at each new grant and each boot, through a temporary file, a rename, and a sync of the dir. At its start, it removes each temporary file `.tmp-*` that a crash left. It removes the file once the control role accepted the report `deleted`. A stale answer to that report ends the executor too: the driver deleted the machine already, and the control store dropped its row. A file that does not parse is logged as `cannot read the node-local state of a machine` at level error, and the other machines still load.

After a restart, the executor asks the driver for each machine of the state dir. It attaches to each microVM that runs, and logs `attached to the running microVM of the machine`. It acts only once the control role sent the spec again.

| Log message | Level | Fields |
| --- | --- | --- |
| `attaching to the machines of this node` | info | `machines` (a count), `driver`. The state dir names at least one machine. |
| `cannot save the node-local state of the machine` | error | `err` |
| `the control plane has the machine deleted already` | info | `reason`. The answer to the report `deleted` was stale. |
| `cannot remove the node-local state of a deleted machine` | error | `err` |
| `cannot remove the log of a deleted machine` | error | `err` |
| `the executor of the deleted machine ends` | info | |

## Metrics

| Metric | Type | Labels | Description |
| --- | --- | --- | --- |
| `hopbox_executor_actions_total` | counter | `action`, `result` | The actions of the executors of the node. |
| `hopbox_executor_action_duration_seconds` | histogram | `action` | The duration of the actions. |

| Label | Values |
| --- | --- |
| `action` | `boot`, `wake`, `reboot`, `stop`, `delete`, `hibernate`, `link`, `report`, `config`, `start_command`, `pull` |
| `result` | `ok`, `failed`, or `canceled` for an action that ended with the loss of its lease |

The histogram has the buckets 0.01, 0.05, 0.1, 0.5, 1, 2.5, 5, 10, 30, 60, 120, and 300 seconds.

## Go API

`executor.New(Config)` returns a `Manager`. `Manager.Run(ctx)` reads the node-local state and runs the executors until `ctx` ends. The microVMs keep running after it returns.

| Field of `Config` | Default | Description |
| --- | --- | --- |
| `Driver` | required | The VMM driver. |
| `Dir` | required | The state dir. |
| `NodeVersion` | empty | The version of `hopboxd`, for the guest link. |
| `ActionTimeout` | 5 min | The timeout of each action. |
| `StableRun` | 5 min | The run time that starts the count of restarts again. |
| `Backoff` | `domain.RestartBackoff` | The wait before restart number n. |
| `Logger` | `slog.Default()` | The logger. |
| `Metrics` | none | The metrics of `executor.NewMetrics(registerer)`. |
| `Logs` | none | The logs of the start commands, an `executor.LogSink`: `Append(id, entry)` takes a line and must not block, and `Remove(id)` drops the log of a deleted machine. Nil drops the lines. |
| `Nameservers` | none | A func that returns the resolvers of the machines of the node, such as `Nameservers` of a `netcfg.Network`. Nil gives a machine no resolver. |
| `Puller` | none | Pulls the disk of a machine that moves to the node, such as a `transfer.Client` ([Cold move](/reference/cold-move/index.html.md)). Nil, or a driver that is no `vmm.Mover`, pulls none, and a moving machine waits. |

The `Manager` is the `Machines` of the agent of the node, `node.Agent.Machines`. The agent calls these methods:

| Method | When |
| --- | --- |
| `Reconcile(ReconcileMachine)` | For each `ReconcileMachine` of the control role. |
| `Leases()`, `Allocations()`, `Actions()` | For each heartbeat. |
| `LeasesRenewed(sentAt, sent, renewed)` | For each `HeartbeatAck`, with the leases of its heartbeat. |
| `ReportResult(MachineReportResult)` | For each result of a report. |
| `Connected(send)`, `Disconnected()` | At the start and the end of each channel. |

For the cold moves, the node role calls these methods ([Cold move](/reference/cold-move/index.html.md#old-copies)):

| Method | What it does |
| --- | --- |
| `Busy(id)` | Reports whether the executor of the machine holds a valid lease, runs an action, or runs the microVM. A microVM that the executor attached to after a restart, before a spec arrived, counts too. The disk listener serves no disk of such a machine. |
| `Forget(ids)` | Removes the files of the machines that left the node, once their executor holds no valid lease and their microVM does not run. A machine that a later call names no more keeps its files. |
| `Removed()` | Returns the machines whose files `Forget` removed, for the heartbeats. |
