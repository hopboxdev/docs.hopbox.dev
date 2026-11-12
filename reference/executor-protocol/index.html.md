# The executor protocol

> The executor protocol on the control channel: the machine leases and their renewals, ReconcileMachine, the current-state reports, and their results.

The executor protocol is the part of the control channel that runs the machines of a node ([the control channel](/reference/control-channel/index.html.md)). Each machine has one executor on its node. The executor is the only code that changes the microVM of the machine. A request only sets the target state in the control store. The control role then tells the executor what the machine must become, and the executor reports each step of the current state back.

The protocol has three parts:

1. The machine lease. The executor acts only with a live grant of the lease of its machine. The heartbeats of the node renew the leases.
2. `ReconcileMachine`. The control role sends the spec of a machine and the lease to act with.
3. `MachineReport`. The executor reports each step of the current state. The control role refuses a report of a grant that is not live.

The messages are in `proto/hopbox/cluster/v1/machine.proto`. The control side is in `internal/control/channel` (`executor.go`) and `internal/service` (`executor.go`). The node side is in `internal/node` (`agent.go`) and `internal/node/executor` ([the node executor](/reference/executor/index.html.md)).

## The machine lease

Each machine has the lease `machine/<machine ID>` in the table `leases` ([Control store tables](/reference/control-store/index.html.md#leases)). The holder is the server ID of the node of the machine. One grant of the lease is one execution of the machine on its node. The epoch of the grant is the execution ID, and the epoch and the random token together are its fencing token. Every message of this protocol and every route record carries both.

| Value | Default | Description |
| --- | --- | --- |
| TTL | 30 s (`domain.MachineLeaseTTL`, `lease.MachineTTL`) | The life of a grant after its grant or its last renewal. |
| Margin | 3 s (`executor.LeaseMargin`) | The executor stops acting this long before the lease expires, so the clocks of the node and of the control store need not agree. |

### Grants

| When | Who | What it writes |
| --- | --- | --- |
| `CreateMachine` places a machine | the machine API ([Placement](/reference/placement/index.html.md#the-machine-lease)) | the lease and the first row of `placements` |
| A heartbeat finds a machine of its node without a live lease | `RecordHeartbeat` | a new grant with a new epoch and token, and a new row of `placements` |

Three rules decide whether a heartbeat grants a lease again. All of them must hold:

- The heartbeat has `executor` set, so the node runs executors. A node without executors gets no new grant, so no grant stays alive for a node that cannot act.
- The machine is on the node, and its current state is not `deleted`.
- The lease of the machine expired, or has no row. A live grant that the node does not list waits for its expiry. So two executions never act at once.
- The machine is not held with the target `running` ([A machine whose env does not resolve](#a-machine-whose-env-does-not-resolve)). A held machine with another target gets its grant, so its stop or its delete goes on.

The same transaction moves the route record of the machine to the new grant ([Control store tables](/reference/control-store/index.html.md#route_records)). After the commit, the control role sends `ReconcileMachine` with the new grant. The service logs `granted a machine lease again` at level info, with `machine` and `server`.

### Renewals

Each heartbeat lists in `leases` the grants that the executors of the node hold. `RecordHeartbeat` renews them in the same transaction as the facts of the heartbeat, with one statement (`lease.RenewAll`). For each grant, the machine must be on the node, and the grant must be its current one, with the same epoch and token. Then the grant is renewed. A renewal moves the expiry to 30 s after the time of the write. It never changes the epoch or the token.

The transaction takes its locks in this order ([Control store tables](/reference/control-store/index.html.md#leases)):

1. The rows of the machines of the server, in the order of their IDs, `FOR NO KEY UPDATE`.
2. The lease rows to renew, in the order of their names, `FOR UPDATE`.
3. The row of the server, `FOR UPDATE`.

`sequence` of `HeartbeatAck` names the heartbeat that the control role recorded last on the channel. `renewed` holds the grants of that heartbeat that the control role renewed. The node matches the answer to the heartbeat with the same `sequence`:

| Lease of that heartbeat | What the executor does |
| --- | --- |
| in `renewed` | It counts the lease as live until the send time of the heartbeat plus 30 s, less the margin. |
| not in `renewed` | The lease is stale. The executor stops acting with it. |

The control role records at most one heartbeat of a channel every 2.5 s ([the control channel](/reference/control-channel/index.html.md#the-heartbeat)). A heartbeat that comes sooner gets the answer of the last recorded one, with its `sequence` and its `renewed`. The node takes each answer once, so a repeated answer changes nothing. A heartbeat that the control role fails to record gets no answer, and its leases are not renewed.

### Heartbeat fields

These fields of `Heartbeat` and `HeartbeatAck` carry the protocol. [The control channel](/reference/control-channel/index.html.md#heartbeat) lists the others.

| Message | Field | Type | Description | Limit |
| --- | --- | --- | --- | --- |
| `Heartbeat` | `sequence` | uint64 | Numbers the heartbeats of one channel, from 1. | |
| `Heartbeat` | `executor` | bool | Tells that the server runs machine executors. | |
| `Heartbeat` | `leases` | repeated `MachineLease` | The grants that the executors hold. | at most 4096 |
| `Heartbeat` | `inbound_machines` | repeated string | The machines of the node with inbound traffic since the last heartbeat ([the node network](/reference/node-network/index.html.md#inbound-traffic-as-a-use)). | at most 4096 |
| `HeartbeatAck` | `sequence` | uint64 | The heartbeat that the control role recorded last. | |
| `HeartbeatAck` | `renewed` | repeated `MachineLease` | The grants of that heartbeat that the control role renewed. | |

### MachineLease

| Field | Type | Description |
| --- | --- | --- |
| `machine_id` | string | The ID of the machine, a UUID. |
| `epoch` | int64 | The number of the grant, and the execution ID. |
| `token` | string | The random token of the grant, a UUID. |

A lease whose `machine_id` or `token` is not a UUID ends the channel with `invalid_argument`.

## ReconcileMachine

`ReconcileMachine` (field 5 of `ChannelResponse`) tells the executor of a machine what the machine must become. The executor acts on the newest one. The control process that holds the channel of the node sends it in these cases:

| Case | Machines |
| --- | --- |
| The first recorded heartbeat of a channel with `executor` set | every machine of the server |
| A change event `machine.created` or `machine.updated`, from any control process ([the machine API](/reference/machine-api/index.html.md#change-events)) | the machine of the event |
| A heartbeat that granted leases again | the machines of the new grants |

Three rules decide which machines get a spec. The machine runs on the server. Its current state is not `deleted`. The server holds its lease live. A machine whose lease expired gets its spec after the heartbeat that grants its lease again. A machine whose current state is `deleted` gets no spec, because its executor is done.

The dispatcher of each control process (`Hub.Dispatch`) follows the change events of every machine. It reads the node of each machine, and sends the specs on the channels of this process. A read that fails is logged as `sending machine specs failed` at level warn, with `machines`, `err`, and `retry_in`. It runs again 1 s later, then after twice the wait each time, up to 1 min (`DispatchRetry`, `MaxDispatchRetry`). The dispatcher never waits for a slow node. If the send queue of a channel is full, the dispatcher skips the node and logs `the channel of a node is busy, so its next heartbeat gets every machine spec` at level info, with `server`. The next heartbeat of the node then sends every spec of the node. In the same way, a failed read of the specs after a heartbeat keeps the channel open: the service logs `sending machine specs failed`, and the next heartbeat tries again. The reports of the executors publish `machine.state_changed`, which changes no spec, so that kind sends nothing.

| Field | Type | Description |
| --- | --- | --- |
| `lease` | `MachineLease` | The grant that the executor acts with. |
| `lease_remaining` | Duration | The rest of the life of the grant at the send of the message. The control role counts it from the time before its read of the lease. The executor counts it from the arrival of the message, less the margin. The executor also ends the lease at most 30 s after its last heartbeat, less the margin. So a spec that waited in a slow channel cannot stretch it. |
| `spec` | `MachineSpec` | What the machine must become. |
| `current_state` | `MachineState` | The current state in the control store: the last report that the control role accepted. |
| `current_boot_generation` | uint64 | The boot generation of that report. |
| `version` | int64 | The version of the machine row at the read. Two messages can arrive out of order, so the executor keeps the spec with the highest version of its grant. |

### MachineSpec

The spec holds the env vars and the secrets of the machine. No part of hopbox logs it, and the node never writes it to disk.

| Field | Type | Description |
| --- | --- | --- |
| `name` | string | The name of the machine, and its host name. |
| `target_state` | `MachineState` | The target state ([Domain rules](/reference/domain-rules/index.html.md#machine-states)). |
| `reboot_generation` | uint64 | Grows with each reboot that a request asks for, and with each start of a failed machine. The executor boots the machine again until it booted this generation. |
| `vcpus` | uint32 | The vCPUs of the size of the machine. |
| `memory_bytes` | uint64 | The RAM of the size, in bytes. |
| `disk_bytes` | uint64 | The disk of the size, in bytes. |
| `image` | `MachineImage` | The image that the disk starts from. |
| `env` | repeated `MachineEnvVar` | The env vars and the open secrets of the machine, from `Service.ResolveEnv` ([the env var API](/reference/vars-api/index.html.md#env-of-a-machine)). A machine without an org, or a private machine whose creator is gone, gets none. |
| `start_command` | string | The start command. Empty means none ([Domain rules](/reference/domain-rules/index.html.md#start-command)). |
| `restart_policy` | `RestartPolicy` | `RESTART_POLICY_ALWAYS` or `RESTART_POLICY_NEVER`. |
| `network` | `MachineNetwork` | The address setup of the machine ([the node network](/reference/node-network/index.html.md#addresses)). Not set for a machine without an address, which then boots without network. |
| `authorized_keys` | repeated string | The public keys that sshd in the machine accepts for every Linux user: the client key of the SSH front door ([the SSH front door](/reference/ssh-front-door/index.html.md#keys)). Empty while the install has no keys of the front door, which needs a key-encryption key. The control role then logs `the machine specs go out without the key of the SSH front door` at level warn, with `component=service` and `err`. |
| `env_unresolved` | bool | The env of the machine did not resolve. The spec then holds no env, and the executor pushes nothing to the guest with it. |
| `move` | `MachineMove` | Set while the machine moves to this node: where the node pulls its disk ([Cold moves](#cold-moves)). Not set for any other machine. |
| `give_back_move_id` | string | Set while an instance admin asks this node to give the machine back to the source node of a failed move ([Give-back](#give-back)). Empty otherwise. |
| `org_id` | string | The ID of the org of the machine, or empty for a machine whose org is gone. The node puts it in the label `org` of `hopbox_machine_info` ([Metrics](/reference/metrics/index.html.md#metrics-of-the-machines)). |
| `size` | string | The name of the size of the machine, such as `default`. The node puts it in the label `size` of `hopbox_machine_info`. |
| `disks` | repeated `MachineDisk` | The extra disks attached to the machine, in the order of their devices in the guest: `/dev/vdb` first ([Extra disks](/reference/disks/index.html.md)). `MachineDisk` has `disk_id`, a UUID, and `read_only`. The driver attaches each one at each boot and each load. A boot whose disk has no file on the node fails, and the node never makes a blank file in its place. A node of an older release ignores the field, and boots the machine without its disks. |
| `discard_memory_generation` | uint64 | The reboot generation that the last `DiscardMemory` set, or 0 ([the machine API](/reference/machine-api/index.html.md#discardmemory)). The executor drops a kept memory file whose save came from a boot of a lower generation, once no microVM of the machine runs ([the node executor](/reference/executor/index.html.md#saved-state)). So an old request never drops the file of a later hibernate. A node of an older release ignores the field. |

### A machine whose env does not resolve

The env of a machine can fail to resolve for three causes. The keyring has no key-encryption key, the keyring does not open, or a secret does not open. A retry cannot help until something changes. So the control role does this:

- A machine with the target `running` gets no spec. It does not boot without its env.
- A machine with another target gets a spec with `env_unresolved` and no env. So its stop or its delete goes on.
- The other machines of the node get their specs.
- A machine with the target `running` that got no spec is held. The hub logs `the env of machines does not resolve, so they get no spec until it does` at level warn, with `server`, `machines`, and `err`. A held machine keeps the spec that its executor has, with the old env.
- The hub tries each held machine again at the recorded heartbeats of its node, at most once each 1 min. It also tries 1 min after the hold, then after twice the wait each time, up to 5 min (`HeldRetry`, `MaxHeldRetry`). Nothing else tells the control role that a key-encryption key came back or that a secret opens again.
- A try ends the hold once the env resolves, or once the target is not `running` any more. The hub then logs `held machines need their hold no more` at level info, with `server` and `machines`, and sends the spec. A spec with another target, such as the spec of a stop, ends the hold too.
- A heartbeat lists the held machines of its node in `Hold`. A held machine with the target `running` gets no new grant of its expired lease. So a grant that no spec carries never makes a row of `placements`. After the hold, the next heartbeat grants the lease again.
- Any other failure of the env is not a hold, for example a failed read of the control store. Then the whole read of the specs fails. The hub tries again as after any failed read ([ReconcileMachine](#reconcilemachine)). The dispatcher tries after 1 s, then after longer waits. A channel tries at its next heartbeat.
- The hub forgets the held machines of a node in two cases. The channel of the node to this process ends, or a timed try finds no channel. The next channel sends every spec again.

A held machine keeps its old env. The old env of a private machine holds the private values of its creator. So a share fails closed: `ShareMachine` resolves the env of the machine as a shared machine in its transaction. If that env does not resolve, the share fails with `failed_precondition`, and the machine stays private ([the machine API](/reference/machine-api/index.html.md#sharemachine)). This check covers the moment of the share only. The key can go away after the commit of the share, and before the hub resolves the new spec. Then the hub holds the machine with its old env, as above. `UnshareMachine` does not check the env. If the env of the machine as a private machine does not resolve, the hub holds the machine, and the machine keeps its old env. That env holds only the shared values and the all values of its org, which every member already uses in the shared machines.

The spec carries no link key of the guest link. The node makes a new link key for each boot of the machine, and for each attach after a restart of `hopboxd` ([Guest link](/reference/guest-link/index.html.md#link-key)).

| `MachineImage` field | Type | Description |
| --- | --- | --- |
| `digest` | string | The digest of the artifact, `sha256:<hex>` ([Artifact format](/reference/artifact-format/index.html.md)). |
| `url` | string | Where the node downloads the image. Empty means that the node has it in its image cache. |
| `kernel_sha256` | string | The SHA-256, in hex, of the guest kernel. |
| `disk_sha256` | string | The SHA-256, in hex, of the disk file. |
| `disk_size` | uint64 | The size of the disk file, in bytes. |

| `MachineEnvVar` field | Type | Description |
| --- | --- | --- |
| `name` | string | The name of the env var. |
| `value` | string | The value. |

| `MachineNetwork` field | Type | Description |
| --- | --- | --- |
| `address` | string | The address of the machine with its prefix length, such as `10.240.0.5/32`. The node routes it to the tap device of the machine. |
| `gateway` | string | The on-link gateway of the default route: `169.254.0.1`. |
| `mtu` | uint32 | The MTU of the interface: `1420`. |
| `nameservers` | repeated string | The DNS resolvers of the machine. The control role leaves it empty, and the executor then gives the guest the resolvers of its node ([the node network](/reference/node-network/index.html.md#the-resolvers-of-the-machines)). |
| `outbound_smtp` | bool | `true` for a machine whose org has the override `outbound_smtp`. At each boot, the node then lets the machine reach TCP port 25, which it blocks for every other machine ([the node network](/reference/node-network/index.html.md#outbound-tcp-port-25)). A node of an older release blocks nothing and ignores the field. |

Each value of `MachineState` is a state of [Domain rules](/reference/domain-rules/index.html.md#machine-states) with the prefix `MACHINE_STATE_`, such as `MACHINE_STATE_RUNNING`. `MACHINE_STATE_UNSPECIFIED` is no state.

## Cold moves

A cold move takes a stopped machine to another node with its disk ([Cold move](/reference/cold-move/index.html.md)). The protocol carries it in these steps:

1. The move waits for the stop of the machine. Meanwhile, the spec of the machine on its node has the target `stopped`, apart from a target `deleted`. The executor stops the machine and reports `stopped`, as for any stop.
2. The leader hands the machine to the target node. In one transaction, it ends the grant of the source, grants the lease to the target, and writes a row of `placements`. It sets the current state `moving` and moves the machine and its address to the target node. No executor holds the lease at that time, so the control role writes this state itself.
3. The next heartbeat of the source finds its grant stale, and its executor stops acting. The spec goes to the target node with the current state `moving` and the field `move`.
4. The executor of the target writes the move in its node-local state, and pulls the disk with the action `pull` ([the node executor](/reference/executor/index.html.md#actions)). It never boots the machine before the control role accepted the proof of the pull of that move. With the target `deleted`, it deletes what arrived instead. A pull that wrote bytes and ended early gets a report of the same state with `pull_received`, which holds off the pull deadline.
5. Once the disk is whole, the executor leaves `moving` through `stopped`, with the ID of the move in `pulled_move_id`. That report ends the move. If the target of the machine is `running`, the executor boots the machine after the control role accepted the report.
6. After 3 pulls in a row that fail for good, the executor reports `failed` from `moving`, and acts no more with its grant. The report gives the machine back to the source node. The control role ends the grant of the target in the transaction of the report, and writes the current state `stopped`.

The control role reads the move in the transaction of each report of the target node while the move pulls:

| Report of the target | What the control role does in the transaction of the report |
| --- | --- |
| Any state but `failed`, with the ID of the move in `pulled_move_id` | The move is done. The source node must remove its old copy. A report of the same state ends the move too. |
| The same state, without the proof | Nothing ends. `pull_received` above 0 holds off the pull deadline. |
| `failed` from `moving` | The move failed. The control role ends the lease of the target. It gives the machine and its address back to the source node with the current state and the target `stopped`. The target node must remove what it pulled. |
| Any other step | The same as `failed`, with the error `the target node left the state <state> with <new state>, without the whole disk of the move`. |

The control role gives the machine back only while it is in `moving` on the target. A control process of a release without moves can accept a step of the target out of `moving` during a pull. After that, a failed move leaves the machine on the target ([Cold move](/reference/cold-move/index.html.md#a-step-that-an-older-control-process-accepted)).

A spec with the current state `moving` and without `move` makes the executor wait. A control process of a release without moves sends such a spec. The executor then logs `the executor cannot reach the target` with its reason. The node-local state of the executor keeps the move. While the disk of that move is not whole, a spec with another current state boots nothing either.

### Give-back

A failed move can leave a machine on its target while the source keeps a copy of its disk ([Cold move](/reference/cold-move/index.html.md#give-back)). An instance admin can ask the target node to give the machine back. The spec of the machine then names the move in `give_back_move_id`. For each new version of the spec, the executor answers once, with a report of its current state, once no other report waits:

| Node | Field of the report |
| --- | --- |
| It never proved the pull of the move, holds no disk file of the machine, and runs no microVM of it. A partial copy of a pull is no disk file. | `released_move_id`. The report has no `pulled_move_id`. |
| It proved the pull, or holds the whole disk of the move, and its disk file is here | `kept_move_id`, with `pulled_move_id` while the disk is here |
| One of three cases. It holds a disk file without the proof of the move. Or it proved the pull and lost the disk file. Or it cannot read the disk file. | `withheld_move_id`, with `withheld_reason` |
| A microVM of the machine runs | none |

A release gives the machine back to the source in the transaction of the report, and ends the grant of the target. So no later report of the target is accepted, and the executor never boots the machine. A node reports the feature `give-back` in its heartbeats ([the control channel](/reference/control-channel/index.html.md#heartbeat)), and the control role asks only a node that reports it.

### MachineMove

| Field | Type | Description |
| --- | --- | --- |
| `move_id` | string | The ID of the move, a UUID. A partial copy of another move starts again from zero. |
| `source_server_id` | string | The ID of the source node, a UUID. Its certificate must name it. |
| `source_address` | string | The disk listener of the source node: its mesh address and port `7444`, such as `10.239.0.2:7444`. |
| `disk_ids` | repeated string | The extra disks attached to the machine ([Cold move](/reference/cold-move/index.html.md#attached-disks)). The executor pulls each one after the disk of the machine, into the file of the disk on the node. It keeps the disks that it pulled whole in `pulled_parts` of its node-local state, so a restart pulls none of them again. It names the move in `pulled_move_id` only once every disk is here. |

## MachineReport

`MachineReport` (field 4 of `ChannelRequest`) is one step of the current state of a machine. The executor sends its reports one at a time: it sends the next one after the result of the one before. So the reports of a machine arrive in order. After a new channel opens, the executor sends its waiting report again. A report of the state that the control store holds already changes nothing, so a report that arrives twice is harmless.

| Field | Type | Description | Limit |
| --- | --- | --- | --- |
| `lease` | `MachineLease` | The grant that the executor acts with. | |
| `sequence` | uint64 | Numbers the reports of one grant, from 1. The result names it. | |
| `state` | `MachineState` | The new current state. `MACHINE_STATE_PENDING` and `MACHINE_STATE_UNSPECIFIED` are refused. | |
| `boot_generation` | uint64 | The reboot generation of the boot that the report is about: the boot that starts, runs, or failed. | |
| `stopped_by_itself` | bool | Tells that the machine shut down from inside, with the restart policy `never`. Only a report of `stopped` can set it. | |
| `error` | string | Why the machine failed, for a report of `failed`. With `save_refused`, why the node refused the save. | at most 1024 bytes, valid UTF-8 |
| `standby_lost` | bool | Tells that a machine with the target `standby` has no microVM, so its RAM is gone ([the node executor](/reference/executor/index.html.md#a-lost-standby)). Only a report of `stopped` can set it. A report of the state that the machine has already can carry it. | |
| `skipped_env` | uint32 | For a report of `running`, the count of env values that the guest agent left out ([Guest link](/reference/guest-link/index.html.md#configuration)). After a change of the count, the executor reports `running` again. | at most 2147483647 |
| `ssh_host_keys` | repeated string | For a report of `running`, the public SSH host keys of the guest, as the guest agent reported them in its status. Each one is in the `authorized_keys` format without a comment. The executor leaves out a key that does not parse, and a key that it has already. After a change of the keys, the executor reports `running` again. The list can be empty. The guest agent can fail to make or read a host key, or no key of its status parses. Or a guest agent of an older image sent `ConfigApplied` before its status ([Guest link](/reference/guest-link/index.html.md#link-messages)). A report of another state carries none. | at most 8, each one key without options or a comment |
| `pulled_move_id` | string | The ID of the move whose whole disk the executor pulled, with a digest that matches. The executor sets it on each report once the disk is here, until a later move. A step of the target of a move without it fails the move ([Cold moves](#cold-moves)). A value that is not a UUID counts as empty. | |
| `pull_received` | uint64 | For a report of the same state during a pull, the bytes that the last pull wrote. Above 0, it holds off the pull deadline of the move. 0 on every other report. | |
| `pull_restarted` | bool | For a report of the same state during a pull, after a pull whose copy had another digest than the disk of the source: the next pull starts from zero. The first such report of a move gives the move a new time limit, from that report ([Cold move](/reference/cold-move/index.html.md#pull-deadline-and-cancel)). A control role of an older release ignores it. | |
| `released_move_id` | string | The answer to a give-back: the node never proved the pull of that move, and gives the machine back ([Give-back](#give-back)). A value that is not a UUID counts as empty. | not with `kept_move_id` or `pulled_move_id` |
| `kept_move_id` | string | The answer to a give-back: the node holds the whole disk of that move, and keeps the machine. A value that is not a UUID counts as empty. | |
| `withheld_move_id` | string | The node gives neither answer to the give-back of that move, because either one can drop the newest disk ([Give-back](#give-back)). The control role writes the note of `withheld_reason` to the move, and the give-back keeps waiting. A value that is not a UUID counts as empty. | not with `released_move_id` or `kept_move_id` |
| `withheld_reason` | `GiveBackWithheld` | Why the node withholds its answer: `GIVE_BACK_WITHHELD_DISK_WITHOUT_PROOF`, `GIVE_BACK_WITHHELD_PROOF_WITHOUT_DISK`, or `GIVE_BACK_WITHHELD_DISK_UNREADABLE`. | required with `withheld_move_id` |
| `memory_state` | `MemoryState` | For a report of `hibernated`, the memory file that the node keeps for the machine. If the record of the file on the node is not complete, as after a damaged `state.json`, the executor leaves it out ([the node executor](/reference/executor/index.html.md#saved-state)). A report of another state with it is refused. | only on a report of `hibernated` |
| `save_refused` | `SaveRefusal` | Tells that the node refused the save of a machine with the target `hibernated` ([the node executor](/reference/executor/index.html.md#the-gate-of-a-save)). The report names the state that the machine has already, `running` or `standby`, and `error` says why. `SAVE_REFUSAL_DISK_LOW`: the save leaves less free disk on the node than the floor of `vmm.save_free_percent` and `vmm.save_free_bytes`. A control role that does not know a value takes the report as one without a refusal. | |

| `MemoryState` field | Type | Description | Limit |
| --- | --- | --- | --- |
| `vmm` | string | The VMM driver that saved the memory, such as `firecracker`. | 1 to 64 printable ASCII characters, no space |
| `vmm_version` | string | The version of the VMM, such as `v1.17.0`. | the same |
| `snapshot_format` | string | The format version of the saved files, such as `v12.0.0` ([the VMM driver interface](/reference/vmm/index.html.md#driver)). | the same |
| `bytes` | uint64 | The bytes of the disk of the node that the memory file and the VMM state file hold, from their blocks ([the Firecracker driver](/reference/firecracker-driver/index.html.md#the-bytes-of-a-saved-state)). A sparse file holds fewer bytes than the RAM of the machine. | at most 9223372036854775807 |
| `saved_at` | `google.protobuf.Timestamp` | The time of the save. | required, a valid time |

A `memory_state` with more bytes than the limit, or with a time that is not valid, breaks the protocol and ends the channel. A field that breaks another limit refuses the report.

### What the control role does

`ReportMachine` of the service layer writes each report in one transaction:

1. It checks that the certificate of the channel was not retired ([the control channel](/reference/control-channel/index.html.md#certificate-renewal)).
2. It locks the machine row. A machine that is gone, or that runs on another node, refuses the report as stale.
3. It fences the report with its lease (`lease.Fenced`, [Control store tables](/reference/control-store/index.html.md#fenced-writes)). A report of another grant, or of an expired one, is stale. One case differs. A report of `deleted` for a machine that is `deleted` already, with the grant of the delete, is accepted and changes nothing. That is the case of a lost answer, because the delete ended the lease.
4. A report that answers a give-back of the machine ends the give-back ([Give-back](#give-back)). After a release, the machine is on its source node, and the report changes nothing else. A report of the state that the machine has already, with no higher boot generation, changes no state. A repeated report of `running` still writes the route record with its grant, and the count of step 6.
5. It checks the step from the current state to the new state with `domain.CheckStep` ([Domain rules](/reference/domain-rules/index.html.md#steps-between-current-states)). A step from `hibernated` to `stopped` needs one more check: the control store must have no row of the memory file, because only `DiscardMemory` ends that row. Otherwise, the report fails with `failed precondition: machine state step not allowed: from hibernated to stopped while the memory file of the machine stays. Only DiscardMemory allows it`.
6. A report of `running` writes `skipped_env` to the machine. The service then logs `the guest agent left env values out of a machine` at level warn, with `machine` and `count`, for a count above 0. A change of the count alone publishes `machine.state_changed`.
7. It writes `current_state`, and moves `current_reboot_generation` up to `boot_generation`. For a new state, it also sets `state_changed_at`.
8. A report with `stopped_by_itself` of a machine with the target `running` and the restart policy `never` sets the target to `stopped`. So a later start boots the machine again. The `boot_generation` of the report must not be lower than the `reboot_generation` of the machine. A start after the shutdown adds 1 to `reboot_generation` ([the machine API](/reference/machine-api/index.html.md#startmachine)). So when the executor sends the report again on a new channel after that start, the report leaves the target `running`. The audit entry is `machine.stop` with the actor `system` and the details `old_target` and `reason`.
9. A report with `standby_lost` of a machine with the target `standby` sets the target by the restart policy of the machine: `running` for `always`, and `stopped` for `never`. The `boot_generation` of the report does not count. A reboot request that a pause crossed leaves the `reboot_generation` above the boot that the executor paused. No call raises it while the target is `standby`. A report that the executor sends again after a resume or a stop finds another target, and changes no target. The audit entry is `machine.start` or `machine.stop`, with the actor `system` and the details `old_target` and `reason`. The service logs `a machine lost its RAM in standby, so its target follows its restart policy` at level warn, with `machine`, `server`, and `target`. Both targets need no check of the limits, because `standby` counts as `running` does.
10. A report with `save_refused` of a machine with the target `hibernated` sets the target back to the state of the report. The control store must already hold that state, `running` or `standby`. A report that crossed another change of the target changes no target. The control role writes `save_refused_at` and `save_refused` of the machine, with the first 512 bytes of `error` as the reason ([Control store tables](/reference/control-store/index.html.md#machines)). The audit entry is `machine.save_refuse`, with the actor `system`. Its details are `old_target` `hibernated`, `target` `running` or `standby`, `refused` `disk_low`, and `reason`, the words of the node. The step puts the compute of the machine back in the usage of its org with no limit check, because the machine never stopped. The change publishes `machine.updated`, so the executor gets the new target. The service logs `the node refused the save of a hibernate, so the machine runs on` at level warn, with `machine`, `server`, `reason`, `err`, and `target`. `HibernateMachine` then fails ([the machine API](/reference/machine-api/index.html.md#hibernatemachine)).
11. A report of `running` writes the route record of the machine, with the address of the machine and `ssh_host_keys`. Each key goes to at most one route record, through the table `route_host_keys` ([Control store tables](/reference/control-store/index.html.md#route_host_keys)). Only reports that share a key wait on each other. If the record of another machine holds a key of the report, that key stays out of the record, and the other keys go in. The service then logs `a guest reported an SSH host key of another machine, so the SSH front door does not reach it` at level error. The fields are `machine` and `other_machine`. A key that the record lets go goes to the oldest other route record, by `created_at`, whose last report claimed it. A report of another state drops the record, and lets its keys go the same way ([Control store tables](/reference/control-store/index.html.md#route_records)).
12. A step to `hibernated` with `memory_state` writes the row of the machine in `machine_memory_states`, with the node of the report ([Control store tables](/reference/control-store/index.html.md#machine_memory_states)). It also adds the `bytes` of the save to the daily budget of the org, in `saved_state_writes` ([Control store tables](/reference/control-store/index.html.md#saved_state_writes)). A second report of the same save names the same machine and `saved_at`, so it adds nothing. A step to `running` or `deleted` ends the row. The executor reports `running` after a wake only after the handshake of the guest agent. A step to `failed` or `stopped` keeps the row, because a failed wake keeps the memory file. A report of the state that the machine has already, with no higher boot generation, writes no row. So a second report of `hibernated` never brings back a row that `DiscardMemory` ended. A step to `hibernated` without `memory_state` still writes the row, with what the control role knows. Its `vmm`, `vmm_version`, and `snapshot_format` are empty, its `bytes` are 0, and its `saved_at` is the time of the report. So a move still refuses the machine ([Control store tables](/reference/control-store/index.html.md#machine_memory_states)). The service then logs `a report of hibernated names no memory state, so the control store records no memory file of the machine` at level warn, with `machine` and `server`. One case writes a row without a step: a report of `hibernated` with `memory_state`, when the machine has no row and the `boot_generation` of the report is at least `discard_memory_generation`. A control process of schema version 57 to 72 can serve the channel of the node, take the step to `hibernated`, and drop `memory_state`, so the next report brings the row back. A report of a boot below the discard generation never does.
13. A report of `deleted` ends the lease of the machine, and starts the hold of its address ([the node network](/reference/node-network/index.html.md#the-hold-of-a-released-address)).
14. A report that leaves `moving` ends the move of the machine ([Cold moves](#cold-moves)).
15. It publishes the event `machine.state_changed` with the payload `{"action": "server.report_machine"}`. The lifecycle waits and `WatchMachines` follow it ([the machine API](/reference/machine-api/index.html.md#change-events)).

The service logs each change with `component=service` and the fields `machine`, `server`, `epoch`, and `boot_generation`:

| Message | Level | More fields |
| --- | --- | --- |
| `machine state changed` | info | `from`, `to` |
| `a machine failed` | warn | `from`, `err` (the `error` of the report) |

### MachineReportResult

The control role answers each report with `MachineReportResult` (field 6 of `ChannelResponse`).

| Field | Type | Description |
| --- | --- | --- |
| `lease` | `MachineLease` | The lease of the report. |
| `sequence` | uint64 | The sequence of the report. |
| `outcome` | `ReportOutcome` | What the control role did. |
| `reason` | string | Why, for an outcome other than accepted. |

| Outcome | When | What the executor does |
| --- | --- | --- |
| `REPORT_OUTCOME_ACCEPTED` | The control role wrote the report, or it had it already. | It sends the next report. |
| `REPORT_OUTCOME_STALE` | The lease is not the live grant of the machine, or the machine is gone or on another node. | It stops acting with the lease, and drops its waiting reports. |
| `REPORT_OUTCOME_REFUSED` | The report breaks a rule, such as a step that the lifecycle does not allow. | It logs `the control plane refused a report of the machine` at level error, drops its waiting reports, and goes on from the last state that the control role accepted. |
| `REPORT_OUTCOME_RETRY` | The control role failed to write the report, for example during an outage of the control store. | It sends the report again 1 s later. |

The control side logs with `component=channel` and the fields `server` and `machine`. A stale or refused report logs `refused a machine report` at level warn. A failed write logs `recording a machine report failed` at level warn. Some reports break the protocol, such as a report without a state. Such a report ends the channel with `invalid_argument`. A lease that is not a UUID does the same.

## Two releases

The nodes upgrade last in a rolling upgrade. So the executors of a node of release N-1 run machines for a control role of release N. The messages of this protocol follow the rules of [the control channel](/reference/control-channel/index.html.md#messages-of-two-releases): a new field of `MachineSpec` or `MachineReport` must not change what the other release does with the fields that it knows.

Placement puts a new machine only on a node whose `hopboxd` is at least `version.MinNodeVersion` ([Placement](/reference/placement/index.html.md#hard-filters)). Only a release whose control role sends a spec that older nodes cannot run raises it. That release raises it to the oldest release that runs the new spec. A machine that runs on an older node stays there. So a release that changes the spec keeps sending the old form to the nodes of the release before, until they upgrade.

A release before cold moves ignores the field `move`, and can boot a moving machine from its image. So a node is the source or the target of a move only while its heartbeats say `disk_transfer` with its current `hopboxd` ([Cold move](/reference/cold-move/index.html.md#two-releases)). If such a node rolls back during a pull, its report out of `moving` has no `pulled_move_id`. The move then fails, and the source keeps the disk.

A control role of an older release ignores `save_refused`. The target then stays `hibernated`, and the node checks its floor again at each try of the save. A node of an older release checks no floor, and sends no refusal.

## Policy

The policy lets a server act only for itself ([the API server](/reference/api/index.html.md#callers-and-policy)):

| Action | Operation | Allowed caller |
| --- | --- | --- |
| `server.read_machine_specs` | read the specs of the machines of the server, with their env vars and secrets | the server that the certificate names |
| `server.report_machine` | record a report of an executor of the server | the same server |
| `server.read_disk_transfers` | read the disks that the server serves, and the machines whose old files it removes ([Cold move](/reference/cold-move/index.html.md#old-copies)) | the same server, with the role `node` |

## Go API

| Function | Description |
| --- | --- |
| `Service.RecordHeartbeat(ctx, caller, Heartbeat)` | Records the facts, renews the leases, and grants expired leases again. Returns `HeartbeatResult` with `State`, `Renewed`, and `Granted`. |
| `Service.ExecutorSpecs(ctx, caller, ids)` | Returns the specs of the machines of the server of `caller` whose live lease it holds, or of every such machine for nil `ids`. |
| `Service.ReportMachine(ctx, caller, MachineReport)` | Writes one report. Fails with `lease.ErrStale` for a stale lease, and with `ErrFailedPrecondition` for a step that the lifecycle does not allow. |
| `Service.MachineNodes(ctx, ids)` | Returns the node of each machine, so the dispatcher finds the channel of its spec. |
| `Service.DiskTransfers(ctx, caller)` | Returns the disks that the node of `caller` serves to the targets of moves, and the machines whose old files it removes. |
| `Hub.Dispatch(ctx, MachineHub)` | Sends the specs of changed machines until `ctx` ends. The internal listener of the control role runs it. |
| `MachineHub.SubscribeKinds(kinds...)` | Follows the changes of every machine in events of the given kinds. |
