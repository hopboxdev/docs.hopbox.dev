# Extra disks

> The extra disks of the machines. The disk API, the rules of a disk, the rights, the files on the nodes, the moves of disks, two releases, and logs.

An extra disk is a block device that a user makes apart from a machine, and attaches to one machine at a time. The service `hopbox.v1.DiskService` of the public API manages the disks of an org. The guide [Extra disks](/guide/extra-disks/index.html.md) runs the calls from a laptop.

[The API server](/reference/api/index.html.md) covers the parts that every call shares: the request gate, the policy check, and the error codes. The generated reference is [Public API](/api/hopbox.v1.openapi.yaml). The control side is in `internal/service/disks.go`, `internal/service/disk_nodes.go`, `internal/service/disk_evacuation.go`, and `internal/api/handlers_disks.go`. The node side is in `internal/node/disks` and the VMM drivers.

| Call | What it does |
| --- | --- |
| [`CreateDisk`](#createdisk) | Makes an empty disk on one node. |
| [`GetDisk`](#getdisk) | Returns one disk. |
| [`ListDisks`](#listdisks) | Lists the disks of an org that the caller sees. |
| [`AttachDisk`](#attachdisk) | Attaches a disk to a stopped machine on the node of the disk. |
| [`DetachDisk`](#detachdisk) | Takes a disk off its stopped machine. The data stays. |
| [`DeleteDisk`](#deletedisk) | Deletes a detached disk with its data. |
| [`ShareDisk`](#sharedisk) | Opens a private disk to the whole org. |

The examples on this page use the shell function `browser` of [First sign-in on a new install](/guide/first-sign-in/index.html.md#sign-in-with-the-link), with the session cookie of the caller, and `API=http://127.0.0.1:7080`.

## Rules of a disk

The owner decided these rules on 2026-10-14 and 2026-11-06 in Q186 of the phase 3 plan. The owner decided the values that the design pages leave open on 2026-11-11, in Q220 to Q224 of the plan.

- A disk lives on one node. A disk that names a machine at its create lives on the node of that machine. Otherwise the disk lands on the ready node with the most free disk whose `hopboxd` names the feature `disks` ([the control channel](/reference/control-channel/index.html.md#heartbeat)). A degraded or cordoned node takes no new disk. The create locks the row of the node, so a decommission of the node waits for it.
- A disk has 1 GB to 2000 GB, in whole GB. Its file is sparse, so it holds only the blocks that the guest writes. The disk counts in the disk limit of its org by its size, as the disk of a machine does ([Domain rules](/reference/domain-rules/index.html.md#usage-limits)).
- A new disk has no file system. The guest sees an empty block device, and the user formats it, for example with `mkfs.ext4`. hopbox mounts nothing.
- A disk attaches to one machine at a time, read-only or read-write. A machine holds at most 8 disks.
- Attach and detach need a stopped machine: the current state `stopped`, the target `stopped`, no move, and no memory file on its node. The saved state of a hibernated machine names its drives. So a hibernated machine keeps its disks. A machine after a failed wake keeps them too, until a wake or `DiscardMemory` ([the machine API](/reference/machine-api/index.html.md#discardmemory)).
- The guest sees the attached disks as `/dev/vdb` to `/dev/vdi`, in the order of their attach. The root disk is `/dev/vda`. A detach of an earlier disk moves the names of the later ones at the next boot. The field `device` of each attached disk names its device.
- A disk name follows the rules of a machine name ([Domain rules](/reference/domain-rules/index.html.md#machine-names)), and is unique in its org. Another org can use the same name.
- A disk is ready once its node reports its file, at a heartbeat of the node a few seconds after the create. Only a ready disk attaches. `hopbox disk create` waits up to 30 s for that ([hopbox CLI](/reference/cli/index.html.md#disk-create)).
- The delete of a machine detaches each of its disks and keeps them. The disks go on counting in the limits of the org. The creator then deletes them.
- A backup leaves the extra disks out. A fork and a snapshot of a machine with an attached disk fail and name the disk. A checkpoint keeps a reflink copy of each attached disk. Milestones T12, T13, and T18 build these calls.

## Who can do what

A disk belongs to its creator, as a private machine does, until `ShareDisk` opens it to the org. The rights follow the rights of a machine with the same creator and the same share ([the machine API](/reference/machine-api/index.html.md#who-can-do-what)):

| Call | The creator | A member, on a shared disk | A member, on a private disk of another | An org admin or owner, on a private disk of another |
| --- | --- | --- | --- | --- |
| `GetDisk`, and the disk in `ListDisks` | yes | yes | no, `not_found` | yes |
| `DeleteDisk` | yes | yes | no, `not_found` | yes |
| `AttachDisk`, `DetachDisk` | yes | yes | no, `not_found` | no, `permission_denied` |
| `ShareDisk` | yes | no, `permission_denied` | no, `not_found` | no, `permission_denied` |

`CreateDisk` and `ListDisks` take a caller that acts in the org. An org key makes a shared disk, as it makes a shared machine. An instance admin acts on the disks of others through the admin API ([the admin API](/reference/admin-api/index.html.md#diskservice)).

`AttachDisk` and `DetachDisk` also need the action `machine.exec` on the machine, because the guest reads and writes the disk. A disk attaches only to a machine of its side, as the owner decided in Q222:

| Disk | Machine | Result |
| --- | --- | --- |
| private | a private machine of the creator of the disk | attaches |
| private | a shared machine, or a private machine of another | `failed precondition: the disk disk1 is private, and the machine box1 is shared. Share the disk first with ShareDisk, so that the org can use it`, or `... so it attaches only to a private machine of its creator` |
| shared | a shared machine | attaches |
| shared | a private machine | `failed precondition: the disk disk1 is shared, and the machine box1 is private. A shared disk attaches only to a shared machine` |

So no member reaches a private disk of another through a shared machine, and `ListDisks` never names a private machine of another member. For the same reason, `ShareMachine` refuses a machine with an attached private disk, and `UnshareMachine` refuses a machine with an attached shared disk ([the machine API](/reference/machine-api/index.html.md#sharemachine)). `ShareDisk` refuses a disk on a private machine.

## Messages

### Disk

| Field | Type | Description |
| --- | --- | --- |
| `id` | string | The UUID of the disk. |
| `org_id` | string | The UUID of the org of the disk. |
| `name` | string | The name of the disk, unique in its org. |
| `size_gb` | int64 | The size of the disk in GB. |
| `used_bytes` | int64 | The bytes that the blocks of the disk hold on its node, as its node last reported them. |
| `shared` | bool | If every member of the org can use the disk, `true`. |
| `creator_account_id` | optional string | The account that made the disk. |
| `creator_key_id` | optional string | The org key that made the disk. |
| `ready` | bool | `true` once the node of the disk made its file. |
| `attachment` | `DiskAttachment` | The machine that holds the disk, or not set. |
| `created_at` | timestamp | When the disk was made. |

### DiskAttachment

| Field | Type | Description |
| --- | --- | --- |
| `machine_id` | string | The UUID of the machine. |
| `machine_name` | string | The name of the machine. |
| `read_only` | bool | If the guest cannot write the disk, `true`. |
| `device` | string | The block device of the disk in the guest, such as `/dev/vdb`. |

## Calls

Each request that names a disk has the field `org_id` and the oneof `disk` with `id` or `name`. A name names a disk in the org of `org_id`. Leave `org_id` out for the org of an API key, or the personal org of the caller ([the org API](/reference/org-api/index.html.md#the-org-of-a-request)). A caller that cannot read the disk gets `not_found`, with the same message as for a disk that does not exist.

### CreateDisk

Makes an empty disk, and returns it at once with `ready` `false`. Its node makes the file at its next heartbeat, and the heartbeat after that makes the disk ready. A later `GetDisk` shows the change. The call does not wait, because an idempotent call commits only at its end.

| Item | Value |
| --- | --- |
| Procedure | `/hopbox.v1.DiskService/CreateDisk` |
| Policy action | `disk.create`: a caller that acts in the org |
| Idempotent option | yes. A retry with the same key returns the first disk. |
| Audit entry | action `disk.create`, target kind `disk`, details `name`, `size_gb`, and `server` |

The request `CreateDiskRequest` has the fields `org_id`, `name`, `size_gb`, and the oneof `machine` with `machine_id` or `machine_name`. The response `CreateDiskResponse` has the field `disk`.

| Error | Code |
| --- | --- |
| A name that breaks the rules, or a size outside 1 to 2000 GB | `invalid_argument` |
| The org has a disk with the name: `already exists: the org has a disk with the name "disk1" already` | `already_exists` |
| The size passes the disk limit of the org: `usage limit exceeded: the org would use <bytes> bytes of disk, and its limit is <bytes>` | `resource_exhausted` |
| A suspended org: `failed precondition: org suspended: an instance admin suspended the org, so it cannot create a disk` | `failed_precondition` |
| The machine is in another org, or has no node | `failed_precondition` |
| The node of the machine is not ready: `failed precondition: the node of the machine box1 is draining, so it takes no new disk. Make the disk after the machine moves to a ready node` | `failed_precondition` |
| The node of the machine runs an older release: `failed precondition: the node of the machine <name> runs hopboxd <version>, whose heartbeats name no feature disks. An instance admin must upgrade the node first` | `failed_precondition` |
| No ready node names the feature: `unavailable: no ready node takes a disk. A disk needs a node whose hopboxd names the feature disks` | `unavailable` |
| A control process of an older release is live: `failed precondition: the control process <process> knows the migrations up to <n>, and only a hopboxd that knows migration 77 has extra disks. Upgrade it first`. Such a process moves a machine without its disks. | `failed_precondition` |

```sh
browser -H 'Idempotency-Key: 0193a5b2-7c1e-7d4a-9f3b-2a1c4e5f6a7b' \
  -d '{"name":"disk1","size_gb":10,"machine_name":"box1"}' $API/hopbox.v1.DiskService/CreateDisk
```

### GetDisk

Returns one disk that the caller sees.

| Item | Value |
| --- | --- |
| Procedure | `/hopbox.v1.DiskService/GetDisk` |
| Policy action | `disk.get` |
| Idempotent option | no |
| Audit entry | none |

The request `GetDiskRequest` has the fields `org_id`, `id`, and `name`. The response `GetDiskResponse` has the field `disk`.

### ListDisks

Lists the disks of an org that the caller sees, by name. The list has no pages.

| Item | Value |
| --- | --- |
| Procedure | `/hopbox.v1.DiskService/ListDisks` |
| Policy action | `disk.list`: a caller that acts in the org. The list holds only the disks that the caller can read with `disk.get`. |
| Idempotent option | no |
| Audit entry | none |

The request `ListDisksRequest` has the field `org_id`. The response `ListDisksResponse` has the field `disks`, a repeated `Disk`.

```sh
browser -d '{}' $API/hopbox.v1.DiskService/ListDisks
```

### AttachDisk

Attaches a disk to a stopped machine on the node of the disk. The guest sees the disk at the next boot. A second call with the same machine and the same `read_only` changes nothing. The attach gives the machine a new version of its spec, so its node gets the list of its disks ([the executor protocol](/reference/executor-protocol/index.html.md#machinespec)).

| Item | Value |
| --- | --- |
| Procedure | `/hopbox.v1.DiskService/AttachDisk` |
| Policy action | `disk.attach` on the disk, and `machine.exec` on the machine |
| Idempotent option | no |
| Audit entry | action `disk.attach`, target kind `disk`, details `name`, `machine`, `machine_name`, `read_only`, and `device` |

The request `AttachDiskRequest` has the fields `org_id`, `id`, `name`, the oneof `machine` with `machine_id` or `machine_name`, and `read_only`. The response `AttachDiskResponse` has the field `disk`.

| Error | Code |
| --- | --- |
| The disk is attached to another machine: `failed precondition: the disk disk1 is attached to the machine box2 already. A disk attaches to one machine at a time, so detach it first` | `failed_precondition` |
| The disk is not ready: `failed precondition: the node of the disk disk1 has not made its file yet. Retry in a few seconds` | `failed_precondition` |
| The disk and the machine are on two nodes: `failed precondition: the disk disk1 lives on another node than the machine box1. Make the disk with the machine box1, so that it lives on its node` | `failed_precondition` |
| The machine is not stopped: `failed precondition: a disk attaches and detaches only while its machine is stopped. The machine box1 is running, with the target running. Stop it first` | `failed_precondition` |
| The node keeps the memory file of the machine: `failed precondition: the node of the machine box1 keeps its memory file, and the saved state names the disks of the machine. Wake the machine, or drop the file with DiscardMemory, before you attach a disk` | `failed_precondition` |
| The machine moves: `failed precondition: the machine box1 moves to another node. Attach or detach the disk after the move`, or the disk moves: `failed precondition: the disk disk1 moves to another node. Retry after the move` | `failed_precondition` |
| The machine holds 8 disks: `failed precondition: the machine box1 holds 8 disks, the most that a machine holds` | `failed_precondition` |
| The disk and the machine are on two sides ([Who can do what](#who-can-do-what)) | `failed_precondition` |
| The node runs an older release, which names no feature `disks` | `failed_precondition` |
| A suspended org: `failed precondition: org suspended: an instance admin suspended the org, so it cannot attach a disk` | `failed_precondition` |
| A control process of an older release is live | `failed_precondition` |

```sh
browser -d '{"name":"disk1","machine_name":"box1"}' $API/hopbox.v1.DiskService/AttachDisk
```

### DetachDisk

Takes a disk off its stopped machine. The disk and its data stay. A disk that is attached to no machine gives no change. The machine gets a new version of its spec.

| Item | Value |
| --- | --- |
| Procedure | `/hopbox.v1.DiskService/DetachDisk` |
| Policy action | `disk.detach` on the disk, and `machine.exec` on its machine |
| Idempotent option | no |
| Audit entry | action `disk.detach`, target kind `disk`, details `name`, `machine`, and `machine_name` |

The request `DetachDiskRequest` has the fields `org_id`, `id`, and `name`. The response `DetachDiskResponse` has the field `disk`. The errors are those of `AttachDisk` for the state of the machine. An attach or a detach by another call between the read of the disk and the change fails the call with `aborted: conflict: the disk disk1 changed its machine during the call. Retry`. `DeleteDisk` and `ShareDisk` fail the same way. A suspended org allows a detach, because a detach deletes no data and runs nothing (Q223).

### DeleteDisk

Deletes a detached disk. Its row goes at once, so the disk no longer counts in the limits. The node of the disk then removes the file ([The files of the disks](#the-files-of-the-disks)). The delete is an unlink, and the bytes can stay on the disk of the node until it reuses the blocks.

| Item | Value |
| --- | --- |
| Procedure | `/hopbox.v1.DiskService/DeleteDisk` |
| Policy action | `disk.delete` |
| Idempotent option | no. A second call gets `not_found`. |
| Audit entry | action `disk.delete`, target kind `disk`, details `name` |

The request `DeleteDiskRequest` has the fields `org_id`, `id`, and `name`. The response `DeleteDiskResponse` is empty.

| Error | Code |
| --- | --- |
| The disk is attached: `failed precondition: the disk disk1 is attached to the machine box1. Detach it first` | `failed_precondition` |
| A suspended org: `failed precondition: org suspended: an instance admin suspended the org, so it cannot delete a disk`. A suspension keeps the data of the org. | `failed_precondition` |

### ShareDisk

Opens a private disk to the whole org. Only its creator shares it. A shared disk stays shared.

| Item | Value |
| --- | --- |
| Procedure | `/hopbox.v1.DiskService/ShareDisk` |
| Policy action | `disk.share`: the creator of the disk only |
| Idempotent option | no |
| Audit entry | action `disk.share`, target kind `disk`, details `name` |

The request `ShareDiskRequest` has the fields `org_id`, `id`, and `name`. The response `ShareDiskResponse` has the field `disk`. A disk on a private machine fails with `failed precondition: the disk disk1 is attached to the private machine box1, and a shared disk attaches only to a shared machine. Share the machine first, or detach the disk`. A suspended org fails with `org suspended`.

## A member who leaves, and the delete of an org

The disks are a resource of the org ([the org API](/reference/org-api/index.html.md#what-a-member-leaves-behind)). `Service.OrgResources` holds them after the machines, so a leave first deletes the machines of the account, which detaches their disks:

| Method | What it does to the disks | Audit entries of each disk |
| --- | --- | --- |
| `MemberLeaving(org, account)` | Deletes each disk that the account made in the org and did not share. A shared disk stays. | `disk.delete`, with the detail `reason` (`member left`). A disk that a live machine still held gets `disk.detach` first. |
| `OrgDeleting(org)` | Deletes every disk of the org. | `disk.delete`, with the detail `reason` (`org deleted`) |
| `OrgSuspending(org)`, `MemberSuspending(org, account)` | Nothing. A suspension keeps the data of the org. | none |

The delete of a machine writes `disk.detach` for each of its disks, with the actor `system`. The details are `machine`, `machine_name`, and `reason` `machine deleted`.

## The files of the disks

The file of a disk is `<data_dir>/disks/<disk id>/disk.raw` on its node, `/var/lib/hopbox/disks/<disk id>/disk.raw` with the default `data_dir`. It lives apart from the dir of each machine, so the delete of a machine keeps it. The control role sends each node the message `NodeDisks` ([the control channel](/reference/control-channel/index.html.md#node-disks)). The disk keeper of the node acts on it:

| Item of `NodeDisks` | What the node does |
| --- | --- |
| A disk with `create` | It makes an empty sparse file of the size, once. |
| A disk without `create` | It keeps the file. A file that is not there stays missing: the node never makes a blank file in place of a lost one. A boot of a machine with that disk fails. |
| A disk with `pull` | It pulls the file from the source of a move of a detached disk ([Moves of disks](#moves-of-disks)). |
| `remove` | It removes the dir of the disk. It waits while an executor of the node holds a valid lease of a machine with the disk, or a microVM runs with it. |

Each heartbeat of the node names the files that it holds, with the bytes of their blocks, and the removals that it finished. The first such heartbeat makes a new disk ready. A removal row of the control store goes once the node reports it. A node that holds the disk again, after a move back, never removes it for an old row.

At each boot and each load, the VMM driver links the file of each attached disk into the jail of the machine. The jail user of the machine gets the file ([the Firecracker driver](/reference/firecracker-driver/index.html.md#extra-disks)).

The disk keeper logs with `component=disks` and the field `disk`:

| Message | Level | Fields |
| --- | --- | --- |
| `made the file of a disk` | info | `size` |
| `the file of a disk is not on this node, and the node makes no blank file in its place` | error | Once, until the file comes back. |
| `cannot read the file of a disk` | error | `err` |
| `cannot make the file of a disk` | error | `err` |
| `removed the file of a disk that left this node` | info | |
| `the control plane asks this node to remove the file of a disk that an executor still acts on, so it stays for now` | warn | Once. The next step tries again. |
| `removing the file of a disk failed` | warn | `err`. The next step tries again. |
| `a detached disk is on this node` | info | `move`, `source`, `size` |
| `the pull of a detached disk stopped, and the node tries again` | warn | `move`, `source`, `received`, `err`, `retry_in` |
| `the pull of a detached disk failed for good, so the disk stays on its source node` | error | `move`, `source`, `err` |

## Moves of disks

A cold move of a machine carries its attached disks ([Cold move](/reference/cold-move/index.html.md#attached-disks)). The target node pulls each of them after the disk of the machine, and the proof of the pull covers them all. A target must name the feature `disks`, and must hold no old copy of one of the disks that it still removes. At the end of the move, the disks live on the target, and the source removes its copies. A failed move keeps them on the source, and the target removes what it pulled. The end of the move acts only on the attached disks still on the source. A delete of the machine during its move detaches its disks on the source. Once the target pulls, it removes what it pulled of them. While the source still stops the machine, the target got no disk to pull, so it gets no removal.

A drain and the removal of the node role move each detached disk by a move of its own ([Cold move](/reference/cold-move/index.html.md#detached-disks)). A disk that cannot move strands, as a machine does. A node that holds a disk fails the empty checks of a drain, and `DecommissionServer` refuses it. The end of the removal of the node role waits too.

## Two releases

The disks need migration 0077 (`service.DisksSchema`, [Control store tables](/reference/control-store/index.html.md#disks)). A control process of the schema version 57 to 76 never reads the new tables, and moves a machine without its disks. So while such a process is live, `CreateDisk` and `AttachDisk` refuse. A node of an older release names no feature `disks`. So no disk lands on it, and no disk attaches to its machines. No machine with a disk moves to it either. A node that goes back to such a release after an attach boots the machine without its disks.

## Metrics

| Metric | Type | Labels | Description |
| --- | --- | --- | --- |
| `hopbox_disks_bytes` | gauge | `server` | The sizes of the extra disks on each node, from the leader. `HopboxDiskOvercommit` adds them to the disks of the machines. `hopbox_machine_disk_bytes` stays the disk of the machine alone. |

## Logs

The control role logs with `component=service`:

| Message | Level | Fields |
| --- | --- | --- |
| `a node removed the files of disks that left it` | info | `server`, `disks` |
| `a detached disk moved to another node` | info | `disk`, `move`, `source`, `target` |
| `the target node failed to pull a detached disk, so the disk stays on its source node` | warn | `disk`, `move`, `source`, `target`, `err` |
| `the drain of a server moves a disk` | info | `server`, `disk`, `move`, `target` |
| `the removal of the node role moves a disk` | info | `server`, `disk`, `move`, `target` |
| `the drain of a server cannot move a disk, so the disk stays with its data` | warn | `server`, `disk`, `reason` |
| `the removal of the node role cannot move a disk, so the disk stays with its data` | warn | `server`, `disk`, `reason` |
| `a step of the move of a disk failed. The next run tries again` | warn | `move`, `disk`, `err` |

The leader logs `taking the moves of disks on failed` at level warn with `err`, with `component=leader`.

## Tests

| Test | Where it runs | What it checks |
| --- | --- | --- |
| `TestDisks`, `TestAttachRefusals`, `TestDiskRights`, `TestDisksOfASuspendedOrg`, `TestDisksOfAMemberWhoLeaves` of `internal/service` | `make test` | The calls, the ready state, the order of the devices, and the spec of a machine. The delete of a machine, the removals, and the disk limit. The refusals of an attach, the rights, and the sides. A suspended org, a leave, and the delete of an org. |
| `TestNodeWithADiskIsNotEmpty`, `TestDiskHoldsTheRemovalOfTheNodeRole`, `TestDrainMovesADetachedDisk`, `TestDiskMoveTimeLimit`, `TestColdMoveCarriesAttachedDisks` of `internal/service` | `make test` | A disk keeps its node from a decommission, from the empty checks of a drain, and from the end of a removal. A drain moves a detached disk, and strands a disk that no node takes or whose move failed. A move that runs too long fails. A cold move carries an attached disk. |
| The tests of `internal/node/disks` | `make test` | The keeper makes the file of a new disk once, and never makes a blank file for a lost one. It waits to remove a busy disk, and pulls a detached disk. It fails a pull for good after 3 failures, but never for refusals. |
| `TestMoveCarriesTheExtraDisks`, `TestBootWithoutTheFileOfAnExtraDisk` of `internal/node/executor` | `make test` | A move pulls the disk of the machine, then each extra disk, also after a cut pull. It proves the pull only once every disk is here. A boot takes the disks of the spec, and fails without the file of a disk. |
| `TestPullOfExtraDisks` of `internal/node/transfer` | `make test` | The disk listener serves an attached and a detached disk at their path, and refuses the other kind of export. It waits for a busy machine or disk. |
| `TestBootStepsOfExtraDisks`, `TestTakeDisks`, `TestLoadsCopyTheReadWriteDisks` of `internal/node/vmm/firecracker` | `make test` | The drives of a boot, the hand-off of a file to the jail user, and the refusal of a link. The copies of the read-write disks at each load and at a discard. |
| `TestExtraDisksEndToEnd` of `internal/control` | `make test` | Two nodes over mutual TLS on loopback, with their executors, their disk keepers, and the test VMM driver. A new disk gets its file, a boot takes it, and a cold move carries it with its data. The delete of the machine keeps it, and a drain moves it back by a disk move. |
| `TestDisksKVM` of `test/fleet` | `make test-fleet` with `FLEET_KVM_BUNDLE`, on a host with `/dev/kvm`, in the shard `kvm` | Two KVM nodes with the real Firecracker. A file on an attached disk stays through a reboot and a cold move. A second attach and a delete of an attached disk fail. A failed wake that changed the disk leaves the bytes of the save after the retry. The delete of the machine keeps its disk. A drain moves the detached disk with its data, and a stranded disk blocks the decommission. |
