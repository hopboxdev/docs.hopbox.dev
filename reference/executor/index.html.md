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
| `running` | no microVM, a memory file | wake: a load of the memory file, then the drop of the file once the guest agent linked ([Saved state](#saved-state)) | `starting`, then `running` after the handshake |
| any but `deleted` | a memory file that a guest agent linked from, also with no microVM. Or a microVM with its guest link and a memory file, as after a drop that failed. | drop the memory file, before any other action but a discard. So no later boot loads a file older than the guest. | none. The next step reports the target, such as `running`. |
| `running` | a microVM, and a reboot that a request asked for | reboot: a stop, then a boot | `starting`, then `running` |
| `running` | a microVM without a served guest link, after a restart of `hopboxd` | attach the guest link with a new key | `running`. The report goes only for another state in the control store. |
| `stopped` | a microVM | stop | `stopped` |
| `stopped` | no microVM, and the state `pending`, `starting`, or `failed` | none | `stopped`. A memory file stays. |
| `stopped` | the state `hibernated`, and a memory file | none | none. A hibernated machine steps to `stopped` only once `DiscardMemory` dropped its memory file. |
| `stopped` | the state `hibernated`, and no memory file | none | `stopped` |
| any but `deleted` | no microVM, and a spec whose `discard_memory_generation` is above the generation of the last boot on the node, with no report of it on the way | discard: the drop of the memory file, for `DiscardMemory`. Without a memory file, as when the file is gone or the last report of the discard got lost, nothing to drop. | `stopped`, with the discard generation as its boot generation, also when the control store has the machine stopped already |
| `deleted` | anything | delete: a stop of the microVM, then the removal of the disk and the memory file | `deleted` |
| `hibernated` | a microVM, and a driver with the capability snapshot | hibernate: the executor withdraws and closes the guest link, and saves the microVM | `hibernated`, with the memory file |
| `hibernated` | a microVM that runs, not paused, that a wake loaded before a restart of `hopboxd`, and whose guest agent did not link yet | link: the executor serves a new guest link, and waits for it at most `WakeLinkTimeout`, as after a wake. Only a guest that linked gets saved, so the save of a hung guest never replaces the memory file. Without a link, it ends the microVM and keeps the file. | `failed` without a link. With a link, the drop of the file, then `running`, then the save. |
| `hibernated` | a paused microVM that a wake loaded before a restart of `hopboxd`, whose save is not whole. The restart cut the load before its resume, so the guest never ran. | end load: the executor ends the microVM with `Instance.Kill`, with no resume, because `Stop` runs a paused guest for its shutdown. It keeps the memory file with the count of its loads, because a save sets that count back to 0. A stop that fails logs `cannot stop a microVM whose load a restart cut`, and the action runs again 5 s later. | `failed`, then `hibernated` |
| `hibernated` | a paused microVM whose save is whole, as after a restart of `hopboxd` between the save and the end of its unit | hibernate: the save ends the unit, and saves nothing again | `hibernated`, with the memory file |
| `hibernated` | a microVM that linked, and another state than `running` in the control store, such as `starting`, or `hibernated` when a restart lost the report of `starting` | none | `running`, so the save then steps from `running`. The report of `running` ends the record of the old memory file, and the save records the new one. |
| `hibernated` | no microVM, a memory file, and another state in the control store, as after a save whose report went with a restart of `hopboxd`, or after a canceled wake whose load failed (Q207) | none | `hibernated`, with the memory file. The step from `failed` needs the memory row in the control store. |
| `standby` | a microVM that runs, and a driver with the capability pause | pause: the executor withdraws and closes the guest link, and pauses the microVM | `standby` |
| `standby` | a paused microVM, and another state in the control store, after a restart of `hopboxd` | none | `standby` |
| `standby` | no microVM | none. The executor boots nothing ([Standby](#standby)). | `stopped` with `standby_lost`, once for each version of the spec |
| `running` | a paused microVM | resume: a new link key, a new guest link, and `PATCH /vm`. Then the executor waits for the guest agent. | `running`, after the handshake |
| `hibernated` or `standby` | a microVM, and a driver without the capability | none | none. The executor logs `the executor cannot reach the target` at level warn, with `target` and `reason`. |
| any but `deleted` | a move in `pending_move` whose disk is not here yet, and a spec with `move` | pull: a copy of the disk from the source node of the move ([Cold move](/reference/cold-move/index.html.md#the-pull)) | none, or `failed` after 3 pulls that fail for good |
| `running` | the disk of the move in `pending_move` is here, without the proof that the control role accepted | report the proof of the pull, then boot once the control role accepted it | `stopped` with `pulled_move_id`, then `starting` and `running` |
| `stopped`, `hibernated`, or `standby` | the same | report the proof of the pull | `stopped` with `pulled_move_id`. A machine that the control store has `stopped` already gets a report of `stopped` again. |
| `deleted` | the state `moving` | delete: the removal of what arrived | `stopped`, then `deleted` |
| any but `deleted` | a move in `pending_move` whose disk is not here yet, and a spec without `move` | none | none. The executor logs `the executor cannot reach the target` at level warn, with `target` and `reason`. |

The executor never boots a machine of a move before its disk is whole. A boot without the disk makes a new disk from the image. The move stays in `pending_move` of the node-local state until the control role accepted a report with the proof of its pull, `pulled_move_id`. The executor then writes the move in `acked_move`. Until then, the machine never boots on the node, whatever its current state. This rule holds after a restart of `hopboxd`, and on a spec of a control role that does not know the move. It also holds after such a control role accepted a step out of `moving`. If the control role ended the move before it accepted the proof, it refuses that report, and the machine never boots on the node. Once the disk is here, each report of the executor names the move in `pulled_move_id`. A pull can write bytes and end early. The executor then reports its current state again with `pull_received`, which holds off the pull deadline of the move. After a pull that failed for good, the executor reports `failed` and acts no more with its grant: the control role gives the machine back to its source node.

A boot or a wake starts only after the control role answered each report that waits in the queue. Such an answer can find the grant stale, for example the answer to the report of a crash. A boot with a stale grant can run the machine beside the executor of the live grant. So a restart after a crash waits for the answer to `failed`. If the control channel is down at the crash, the machine does not restart on the node until the channel is back. When the answer arrives, the machine restarts with the same grant. If the lease ended in the meantime, the machine waits for the next grant, and restarts with it. The wait for that grant can add up to the rest of the lease, about 30 s (`domain.MachineLeaseTTL`). A boot or a wake succeeds once the guest agent applied the configuration of the machine on an authenticated guest link ([Guest link](/reference/guest-link/index.html.md#node-api)). Only then does the executor report `running`. A boot that fails reports `failed` with the error. If the microVM runs but its guest agent does not come up, the executor stops the microVM first. A wake waits at most 60 s for the guest agent (`DefaultWakeLinkTimeout`). A boot waits up to the action timeout.

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
| `hibernated the machine` | info | `bytes`, `vmm_version`, `snapshot_format`, `duration` |
| `the memory file of the machine is whole, but its microVM did not end` | warn | `err`. The executor keeps the memory file, and the step runs again 5 s later. |
| `the microVM of a saved machine ended` | info | The microVM of a save that wrapped `vmm.ErrSaved` ended. The next step reads the memory file again, and reports `hibernated`. |
| `ended the microVM of the saved machine` | info | The try after a save that wrapped `vmm.ErrSaved` ended the microVM. The executor reports `hibernated`. |
| `the microVM of a saved machine did not end` | warn | `err`. The stop runs again 5 s later. |
| `the kept memory file has no complete record, so the report of hibernated names none` | warn | `vmm`, `snapshot_format`. The report of `hibernated` goes without `memory_state`. |
| `the machine woke from its memory file` | info | `loads` (the count of loads of the saved state with this one), `link` (the wait for the guest agent) |
| `dropped the memory file of the woken machine` | info | |
| `cannot drop the memory file of the woken machine` | warn | `err`. The machine runs. The drop runs again 5 s later. |
| `discarded the memory file of the machine at the request of DiscardMemory` | info | `saved_at`, `loads` |
| `cannot discard the memory file of the machine` | warn | `err`. The discard runs again 5 s later. |
| `the VMM driver declares the capability restore but cannot drop a memory file` | error | `driver`. The drop or the discard runs again 5 s later. |
| `the VMM driver declares the capability pause but cannot pause`, `the VMM driver declares the capability pause but cannot resume` | error | `driver`. The action runs again 5 s later. |
| `pausing the machine failed` | warn | `err`. If the guest still runs, or if the driver cannot read its state, the pause runs again 5 s later. Until then, the microVM runs on without a guest link. If the guest reads as paused, the executor reports `standby` at once ([Standby](#standby)). |
| `cannot run the machine again` | warn | `err`. A resume that fails keeps the microVM paused, and runs again 5 s later. |
| `the machine has the target standby without a microVM, so its RAM is gone` | warn | |
| `the microVM of a machine in standby ended` | warn | `reason`, `err` |
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

The executor also reports the SSH host keys of the guest with `running`. After a change of the keys, it reports `running` again. The SSH front door accepts only these keys ([the SSH front door](/reference/ssh-front-door/index.html.md#connection-to-the-machine)). A guest agent of an older image can send its status after `ConfigApplied` ([Guest link](/reference/guest-link/index.html.md#link-messages)). The first report of `running` then has no keys, and the next report of `running` has them. The guest agent leaves out an env value whose name breaks its rules, and counts it in its answer. The executor logs `the guest agent left env values out of the machine` at level warn, with `count`, and reports the count with `running` ([the executor protocol](/reference/executor-protocol/index.html.md#machinereport)). The machine API shows it as `skipped_env_vars`. A running machine gets new env vars and new authorized keys at once: the executor pushes them to the guest agent, and logs `pushed a new configuration to the guest agent`. A share of the machine can change its env vars, for example. A new start command and a new host name apply at the next boot. A new restart policy applies at once. After a restart of `hopboxd`, the executor serves the guest link again with the current spec and a new key. It gives the new key to the metadata service first. The guest agent reads the key before each handshake, so it connects again.

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

## Standby

A machine in standby keeps its microVM on the node, paused, with its memory in the RAM of the node. It keeps its allocation, so it counts on its node and in the limits of its org.

A paused guest answers no HTTP/2 PING, so its keepalive drops the guest link about 20 s into standby ([Guest link](/reference/guest-link/index.html.md#transport)). So the pause runs in this order:

1. The executor withdraws the guest link from `Manager.Guest`, and closes it. An exec or a file copy that runs ends with an error.
2. The executor pauses the microVM with `vmm.Pauser` ([the VMM driver interface](/reference/vmm/index.html.md#pauser)). A call can fail after Firecracker applied it, so the driver then reads the state of the microVM. If the guest still runs, it runs on without a guest link. With the target `standby`, the pause runs again 5 s later. With the target `running`, the executor attaches the guest link again. If the guest is paused, the executor goes on as for a pause that worked: with the target `standby`, it reports `standby`, and with the target `running`, it resumes the guest. If the driver cannot read the state, the error wraps `vmm.ErrStateUnknown`. The executor then reports no `standby`. With the target `standby`, the pause runs again 5 s later, and the report of `standby` waits for a pause that works. With the target `running`, the executor resumes the guest, which is safe in either state. Until then, the machine counts in `hopbox_guest_links_down`.
3. The executor reports `standby`.

The resume runs in this order:

1. The executor makes a new link key, and gives it to the microVM with `Instance.SetLinkKey`, as an attach does. So the guest never runs without its key.
2. The executor serves a new guest link, and resumes the microVM.
3. The guest agent sees the old connection end, reads the new key, and connects again ([Guest link](/reference/guest-link/index.html.md#transport)).
4. After the handshake, the executor publishes the guest link, and reports `running`. The guest agent keeps its boot ID and its run ID, so the start command does not run again.

If the guest agent does not come back within the action timeout, the machine stays in the state `standby` with its microVM running. The executor then attaches the guest link again, as after a restart of `hopboxd`, and `hopbox_guest_links_down` counts the machine.

A pause closes the backoff of a start command that exited, with its link. So the first status after the resume starts that backoff again.

After a restart of `hopboxd`, `Attach` tells the executor that the microVM is paused ([the VMM driver interface](/reference/vmm/index.html.md#driver)). With the target `standby`, the machine stays paused, and the executor names its allocation again. With the target `running`, it resumes. A reboot request that a pause crossed reboots the machine, and `Instance.Stop` resumes the guest for that. A stop and a delete also go through `Instance.Stop`, so the guest shuts down cleanly.

### A lost standby

A node reboot, a crash, or a kill of the unit ends the RAM of a machine in standby. With the target `standby` and no microVM, nothing else starts the machine. So the executor reports `stopped` with `standby_lost`, once for each version of the spec. A microVM that ends with the target `standby` needs no report of `failed` first.

In the transaction of that report, the control role sets the target by the restart policy of the machine, with an audit entry ([the executor protocol](/reference/executor-protocol/index.html.md#what-the-control-role-does)). The owner decided this on 2026-11-06, in Q182.

| Restart policy | New target | What happens next |
| --- | --- | --- |
| `always` | `running` | The executor boots the machine from its disk. |
| `never` | `stopped` | The machine stays down until a start. So a machine with `never` stays down after a node reboot, unlike a running one. |

A reboot request that a pause crossed leaves the boot of the report below the `reboot_generation` of the machine. The rule holds for that report too. With the target `running`, the executor then boots the new generation.

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
4. Only a delete, the end of a wake, and `DiscardMemory` drop the memory file. The owner decided on `DiscardMemory` on 2026-11-06, in Q192.

### Hibernate

The save ends the microVM, so the executor first withdraws the guest link from `Manager.Guest`, and closes it, as for a pause. An exec or a file copy that runs ends with an error. Then it saves the microVM with `vmm.Snapshotter` ([the VMM driver interface](/reference/vmm/index.html.md#snapshotter-and-restorer)), from `running` or from `standby`. The machine frees its allocation, and the executor reports `hibernated` with `memory_state`: the VMM, its version, the snapshot format, the bytes, and the time of the save ([the executor protocol](/reference/executor-protocol/index.html.md#machinereport)). The executor times the save in `hopbox_vmm_save_seconds`.

A save that fails logs `hibernating the machine failed`, and runs again 5 s later. The microVM runs on as it was, running or paused, and an earlier memory file stays. A running microVM then has no guest link until a step attaches it again.

A save can also write a whole memory file, but leave its microVM without an end. Its error then wraps `vmm.ErrSaved` ([the VMM driver interface](/reference/vmm/index.html.md#snapshotter-and-restorer)). The executor keeps the memory file as the newest state of the machine, and logs `the memory file of the machine is whole, but its microVM did not end`. The step runs again 5 s later, and only ends the microVM with `Instance.Stop`, because a second save replaces the memory file of the first. Once the microVM ended, the executor logs `ended the microVM of the saved machine`, and reports `hibernated` with the memory file. If the stop fails, it logs `the microVM of a saved machine did not end`, and tries again 5 s later. The microVM never runs again. Its later end has the exit reason `snapshot`, so it is no crash, and the restart policy does not apply. The executor logs `the microVM of a saved machine ended`, reads the memory file again, and reports `hibernated` with it.

After a restart of `hopboxd`, `Inspect` tells the executor that the node keeps a memory file ([the VMM driver interface](/reference/vmm/index.html.md#driver)). A save can end just before the restart, and its report can get lost. So with the target `hibernated`, no microVM, and a memory file, the executor reports `hibernated` with the file. The record of the file can lack the VMM, its version, the snapshot format, or the time of the save. The report of `hibernated` then goes without `memory_state`.

### The end of a wake

A load succeeds without a read of the memory, so only the guest link proves the memory file. The executor thus keeps the file until the guest agent of the wake linked:

1. The executor makes a new link key, and serves a new guest link. It loads the machine with `Restorer.Restore` and the new key. The driver counts the load first ([the Firecracker driver](/reference/firecracker-driver/index.html.md#load)). The executor times the load in `hopbox_vmm_load_seconds`.
2. It waits for the guest agent, at most `WakeLinkTimeout`, 60 s by default.
3. After the handshake, it logs `the machine woke from its memory file`. It writes the time of the save of the file in `woke_memory` of the node-local state ([Node-local state](#node-local-state)). From then on, the file is older than the guest.
4. It drops the memory file with `Restorer.DropMemory`, and clears `woke_memory`. Then it reports `running`, and the control role ends the row of the memory file ([the executor protocol](/reference/executor-protocol/index.html.md#what-the-control-role-does)).

A wake whose guest agent does not link in time ends the microVM, and reports `failed` with the error `wait for the guest agent: the guest agent did not link within 1m0s of the load, so the memory file can be damaged: <error>`. The memory file stays. A load that fails reports `failed` with `wake the machine: <error>`, and the memory file stays too. After a failed wake, the executor reads the record of the file from the driver again, because the load counted itself in that record. The restart policy then decides, as for a failed boot ([Restart policy](#restart-policy)). Each later wake loads the same saved state again, and puts the disk copy of its first load back.

A drop that fails keeps the file, and the machine runs. The executor reports `running` anyway, and drops the file at a later step, with the action `drop_memory`. That drop comes before any other action but a discard or a delete. It runs also after the end of the microVM. So a stop, a reboot, or a pause never leaves a file that a later start loads over the newer guest.

After a restart of `hopboxd`, the executor compares `woke_memory` with the time of the save of the kept memory file. When they match, a guest agent linked from that file already, so the executor drops it. Otherwise, a running microVM with a memory file can come from a wake that the restart cut before the link. Only its link proves the file. So the executor serves the guest link, and waits at most `WakeLinkTimeout` for it. A link records `woke_memory`, and the drop follows. Without a link in time, the executor ends the microVM, and reports `failed` with the error `the guest agent did not link within 1m0s of the load, so the memory file can be damaged`. The memory file stays.

### DiscardMemory

`DiscardMemory` sets `discard_memory_generation` in the spec to a new reboot generation ([the machine API](/reference/machine-api/index.html.md#discardmemory)). The executor compares it with the generation of the last boot of the machine on the node. A higher value asks for the drop of the memory file that the boot saved. The executor drops the file once no microVM of the machine runs, with the action `discard_memory`, before any other step. It calls `Restorer.DiscardMemory`, which first puts the disk copy of the saved state back over the disk ([the VMM driver interface](/reference/vmm/index.html.md#snapshotter-and-restorer)). Then the executor reports `stopped` with the discard generation: a hibernated machine steps to `stopped`, and so does a failed one. A machine that is stopped already reports `stopped` again with the new generation. That report tells the control role that the disk of the save is back, and a cold move waits for it ([Cold move](/reference/cold-move/index.html.md)). The executor takes the discard generation as the generation of its last boot only once the control role accepted that report, and saves its node-local state then. No other accepted report raises the generation, because a reboot reports `starting` before its stop, and that stop can fail. A refused report, or one that a new grant dropped, goes again, and so does the report of a discard whose memory file is gone. After a refusal, the next try waits 5 s. So a move never waits for the next boot. The next boot starts from the disk of the save, not from a disk that a failed wake changed. A later hibernate saves with a boot of a higher generation, so an old request never drops its file.

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
| `woke_memory` | The memory file from which a guest agent linked, named by the `saved_at` of that file, in RFC 3339 with nanoseconds, UTC. That file is older than the guest, so the executor drops it and never loads it again ([The end of a wake](#the-end-of-a-wake)). Absent while no guest agent linked from the kept file. |

Example of the fields of a woken machine whose drop did not run yet:

```json
{"machine_id":"0f6b1c9e-5d2a-4f7e-9a41-3c8e2b7d6a10","epoch":3,"token":"7c1e9a2f-0b4d-4e6a-8f3c-5d2b1a9e7c40","boot_generation":2,"woke_memory":"2026-11-08T10:12:03.521944107Z"}
```

The executor writes the file at each new grant, at each boot, and at each change of `woke_memory`. Each write goes through a temporary file, a rename, and a sync of the dir. At its start, it removes each temporary file `.tmp-*` that a crash left. It removes the file once the control role accepted the report `deleted`. A stale answer to that report ends the executor too: the driver deleted the machine already, and the control store dropped its row. A file that does not parse is logged as `cannot read the node-local state of a machine` at level error, and the other machines still load.

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
| `hopbox_executor_lease_losses_total` | counter | none | The machine leases that the node lost. |
| `hopbox_executor_boot_phase_seconds` | histogram | `phase` | The duration of each phase of a boot. |
| `hopbox_machine_restarts_total` | counter | `machine`, `reason` | The restarts of each machine by its restart policy. |
| `hopbox_guest_link_handshakes_total` | counter | `result` | The handshakes of the guest links of the node. |
| `hopbox_guest_links_down` | gauge | none | The machines whose microVM runs on the node without an authenticated guest link. A machine in standby does not count, because the executor closed its link on purpose. |
| `hopbox_vmm_save_seconds` | histogram | `kind` | The duration of each save of the memory of a machine by the VMM driver, from the start of the call to its end. Only a save that worked counts. |
| `hopbox_vmm_load_seconds` | histogram | `kind` | The duration of each load of saved memory by the VMM driver, until the microVM runs. The wait for the guest agent does not count. Only a load that worked counts. |

| Label | Values |
| --- | --- |
| `action` | `boot`, `wake`, `reboot`, `stop`, `delete`, `hibernate`, `pause`, and `resume`. Also `link`, `report`, `config`, `start_command`, `pull`, `drop_memory`, `discard_memory`, and `end_load`. |
| `kind` of the saves | `hibernate`. The series starts at 0. |
| `kind` of the loads | `wake`. The series starts at 0. |
| `result` of the actions | `ok`, `failed`, or `canceled` for an action that ended with the loss of its lease |
| `phase` | `unit_start`, `vmm_ready`, `guest_link`, or `start_command` |
| `machine` | The ID of the machine |
| `reason` | `crash`, `guest_shutdown`, or `start_command_exit` |
| `result` of the handshakes | `ok`. `bad_proof` for a guest agent whose proof does not match the link key. `no_version` for a guest agent without a common version of the guest link. `error` for any other failure, such as a broken message, a timeout, or a guest agent that ends the call before its proof. |

The histogram of the actions has the buckets 0.01, 0.05, 0.1, 0.5, 1, 2.5, 5, 10, 30, 60, 120, and 300 seconds. The histogram of the phases has the buckets 0.01, 0.05, 0.1, 0.25, 0.5, 1, 2.5, 5, 10, 30, 60, 120, and 300 seconds. The histogram of the saves has the buckets 0.1, 0.25, 0.5, 1, 2.5, 5, 10, 30, 60, 120, and 300 seconds. The histogram of the loads has the buckets 0.005, 0.01, 0.025, 0.05, 0.1, 0.25, 0.5, 1, 2.5, 5, 10, and 30 seconds.

A lease is lost in three cases. The control role does not renew it, or it refuses a report with the lease as stale, or the lease expires on the node. Each grant counts once, 10 s after its end (`executor.DefaultLossWait`). A cold move ends a grant on purpose. The hand-off ends the grant of the source node, and a failed move or a give-back ends the grant of the target ([Cold move](/reference/cold-move/index.html.md)). The disk transfers of the node can name the machine, to serve or to remove ([the control channel](/reference/control-channel/index.html.md#disktransfers-and-diskexport)). The window of such a name runs from 1 min before the end of the grant to 10 s after it (`executor.DefaultMoveWindow`). A name in that window stops the count of the end. For a grant, only the disk transfers after the start of that grant count. A failed move can give the machine back with a new grant. A loss of the new grant then counts, and the old grant keeps its move. The rule is the same for a new grant within the 10 s. At each heartbeat and at each change of the disk transfers, the executor forgets the old disk transfers of a machine. A disk transfer is old once it is older than the window and the wait. The executor of the machine can end within the 10 s, as after a delete. Then the end of the grant does not count either.

A boot and a reboot have four phases, one after the other. The driver times the first two ([the VMM driver interface](/reference/vmm/index.html.md#driver)).

1. `unit_start`: from the start of the boot of the driver until the process of the VMM runs. It holds the image, the disk, the network device, and the start of the unit. So a first boot that unpacks or downloads its image takes long.
2. `vmm_ready`: until the VMM, Firecracker, answers and started the microVM.
3. `guest_link`: until the guest agent applied its configuration on an authenticated link.
4. `start_command`: until the guest agent reports the first run of the start command. The guest agent starts it after the boot of the guest. A machine without a start command has no such phase.

A wake from a memory file times no phase. A boot that fails ends its phases where it failed.

A restart counts each time that the restart policy `always` starts the machine again. The reasons are a crash (`crash`) and a shutdown from inside the machine (`guest_shutdown`). A start command that exited and runs again counts too (`start_command_exit`). A boot that failed counts in `hopbox_executor_actions_total`, not here. The series of a machine go away at the end of its executor: after its delete, or after it left the node in a cold move.

`hopbox_guest_links_down` counts a machine from the start of its microVM until its guest agent connects, so a boot counts for a few seconds. After a restart of `hopboxd`, a running machine does not count before its first spec. Only the spec lets the executor give its guest agent a new link key. From its first spec, it counts until its guest agent connects again.

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
| `IPv6` | none | A func that reports whether the node gives the machines IPv6, such as `IPv6` of a `netcfg.Network`. Then the guest of a machine with an address gets `address6` and `gateway6` ([Guest link](/reference/guest-link/index.html.md)). Nil gives a machine no IPv6. |
| `Puller` | none | Pulls the disk of a machine that moves to the node, such as a `transfer.Client` ([Cold move](/reference/cold-move/index.html.md)). Nil, or a driver that is no `vmm.Mover`, pulls none, and a moving machine waits. |
| `LossWait` | 10 s | The wait before the end of a grant counts in `hopbox_executor_lease_losses_total`, for the disk transfers of a cold move ([Metrics](#metrics)). |
| `MoveWindow` | 1 min | How long before the end of a grant the disk transfers that named its machine still make the end a move, and no loss ([Metrics](#metrics)). |
| `WakeLinkTimeout` | 1 min | The wait for the guest agent after a wake. The executor then ends the microVM, reports `failed`, and keeps the memory file ([The end of a wake](#the-end-of-a-wake)). |

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
| `Moving(ids)` | Records that the disk transfers of the node name the machines, to serve or to remove. The end of the grant of such a machine does not count as a lost lease ([Metrics](#metrics)). |
