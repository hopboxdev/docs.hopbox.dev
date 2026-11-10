# The VMM driver interface

> The VMM driver interface of the executor, its capabilities, its drivers, and the test driver.

The VMM driver is the layer between the executor of a machine and the VMM that runs its microVM. The executor is the only code that calls a driver ([the node executor](/reference/executor/index.html.md)). Placement and the API never call a driver. Each driver declares its capabilities. For a feature that the driver does not declare, the executor does nothing. The code is in `internal/node/vmm`.

This release has two drivers. The Firecracker driver of `internal/node/vmm/firecracker` runs the machines of a node ([the Firecracker driver](/reference/firecracker-driver/index.html.md)). The test driver of `internal/node/vmm/vmmtest` lives in a test package and never ships.

## Capabilities

`Driver.Capabilities()` returns the capabilities of a driver. A driver can always boot, stop, attach to, and delete a machine.

| Capability | Go value | What it allows | Interface |
| --- | --- | --- | --- |
| pause | `vmm.CapPause` | standby: the microVM pauses in RAM | `vmm.Pauser` |
| snapshot | `vmm.CapSnapshot` | hibernate: the memory goes to a memory file | `vmm.Snapshotter` |
| restore | `vmm.CapRestore` | wake: a boot from the memory file | `vmm.Restorer` |
| fork | `vmm.CapFork` | a new microVM from the state of another | none yet |
| migrate | `vmm.CapMigrate` | a move of a microVM to another node | none yet |

The Firecracker driver declares pause, snapshot, and restore. The node names the capabilities of its driver in each heartbeat ([the control channel](/reference/control-channel/index.html.md#heartbeat)). So placement and the API can decide by them without a call to a driver.

## Driver

A driver runs the microVMs of one node. Its microVMs run apart from `hopboxd`, so they survive a restart and an upgrade of `hopboxd`. The executor of a machine calls the driver for that machine one call at a time. Calls for different machines can run at once.

| Method | Description |
| --- | --- |
| `Name()` | The name of the driver in logs, such as `firecracker`. |
| `Capabilities()` | The capabilities of the driver. |
| `SnapshotFormat(ctx)` | The format version of the snapshots that the VMM writes, such as `v12.0.0`. The compatibility class of the node holds it ([the control channel](/reference/control-channel/index.html.md#compatibility-class)). Each saved memory state records it. The test driver returns `vmmtest-1`. |
| `LinkSocket(id)` | The unix socket of the guest link of the machine: its path, its mode, and its owner. It is the same for each boot of the machine ([Guest link](/reference/guest-link/index.html.md#transport)). The Firecracker driver keeps the socket in the jail of the machine. So before it listens, the executor makes the dir of the socket. |
| `Inspect(ctx, id)` | What the node holds of the machine, as a `Status`. A machine that the node never saw has the zero `Status`. |
| `Boot(ctx, Boot)` | Boots a machine that does not run. For a machine without a disk, it makes the disk from the image first. It returns once the microVM runs, before the guest booted. On an error, no microVM of the machine runs. After a failed boot, the executor asks `Inspect` again before its next action. |
| `Attach(ctx, id)` | Returns the microVM of a machine that runs, for example after a restart of `hopboxd`, also a paused one. `Instance.Paused` then tells the executor that it is paused. It fails with `vmm.ErrNotRunning` for a machine that does not run. |
| `Delete(ctx, id)` | Removes the disk, the memory file, and every other file of a machine that does not run. A machine without files succeeds. |

| `Status` field | Description |
| --- | --- |
| `Disk` | The disk of the machine exists. |
| `Running` | The microVM runs, also while it is paused. |
| `Paused` | The microVM is paused in RAM. |
| `Memory` | A memory file of a hibernate is kept, also after a wake that failed. Only `Delete`, `Restorer.DropMemory`, and `Restorer.DiscardMemory` drop it. |
| `MemoryState` | What the kept memory file holds, as a `vmm.MemoryState`, while `Memory` is true. |

| `MemoryState` field | Description |
| --- | --- |
| `VMM`, `VMMVersion` | The driver that saved the memory, and the version of its VMM, such as `firecracker` and `v1.17.0`. |
| `SnapshotFormat` | The format version of the saved files, as `SnapshotFormat` returns it. |
| `Bytes` | The bytes of the disk of the node that the memory file and the VMM state hold, counted by their blocks ([the Firecracker driver](/reference/firecracker-driver/index.html.md#the-bytes-of-a-saved-state)). |
| `SavedAt` | The time of the save. |
| `Loads` | The loads of the saved state so far, also those that failed. |

The executor reports these fields with `hibernated`, apart from `Loads` ([the executor protocol](/reference/executor-protocol/index.html.md#machinereport)).

`Boot` holds the `Machine` and the link key of the boot:

| Field | Description |
| --- | --- |
| `ID` | The ID of the machine. |
| `VCPUs`, `MemoryBytes`, `DiskBytes` | The size of the machine. |
| `Image` | The image: its digest, its URL, and the SHA-256 of its kernel and of its disk ([the executor protocol](/reference/executor-protocol/index.html.md#machinespec)). |
| `Address` | The IPv4 address of the machine on its node, from the network of the spec, or the zero `netip.Addr` for a machine without one. A driver with a network device for each machine connects the device to the node network with it ([the node network](/reference/node-network/index.html.md)). |
| `LinkKey` | The link key of this boot, 32 random bytes. The driver puts it in the metadata service of the microVM, never on the kernel command line. |
| `OnPhase` | A func that gets the duration of each phase of the boot that the driver times, or nil. A driver calls `Boot.Phase(phase, took)` once for each phase. `vmm.PhaseUnitStart` (`unit_start`) runs from the start of the boot until the process of the VMM runs. Then `vmm.PhaseVMMReady` (`vmm_ready`) runs until the VMM answers and started the microVM. The executor serves them in `hopbox_executor_boot_phase_seconds` ([the node executor](/reference/executor/index.html.md#metrics)). `Restore` times no phase. |

## Instance

`Boot`, `Attach`, and `Restore` return an `Instance`: one running microVM.

| Method | Description |
| --- | --- |
| `Done()` | A channel that closes once the microVM stopped, for any reason. |
| `Exit()` | Why the microVM stopped, once `Done` closed. |
| `Stop(ctx)` | Stops the microVM, and returns once `Done` closed. The driver first asks the guest to shut down, and it resumes a paused guest for that. At the end of `ctx`, it ends the microVM. A stopped microVM succeeds. |
| `Kill(ctx)` | Ends the microVM at once, with no resume and no shutdown of the guest, and returns once `Done` closed. The exit reason is `stopped`. The executor ends a load that a restart of `hopboxd` cut before its resume with it, so the guest never runs. A stopped microVM succeeds. |
| `Paused()` | Whether the microVM is paused in RAM. |
| `Saved()` | Whether the save of the microVM is whole, also for a microVM that a new `hopboxd` attached to after the save. Such a microVM never runs again, and its end has the reason `snapshot`. |
| `SetLinkKey(ctx, key)` | Puts a new link key in the metadata service. The executor calls it after an attach, because the key of a boot lives only in memory. |

| Exit reason | Go value | Meaning |
| --- | --- | --- |
| `stopped` | `vmm.ExitStopped` | The end of `Stop`. |
| `shutdown` | `vmm.ExitShutdown` | A shutdown or a reboot from inside the machine. |
| `crash` | `vmm.ExitCrash` | An end that nobody asked for, such as a crash of the VMM. `Exit.Err` says what went wrong. |
| `snapshot` | `vmm.ExitSnapshot` | The end of `Snapshot`. The memory is in the memory file. |

## Pauser

A driver with the capability pause is a `vmm.Pauser`. Both calls are safe to run twice.

| Interface | Method | Description |
| --- | --- | --- |
| `vmm.Pauser` | `Pause(ctx, inst)` | Pauses the vCPUs of a running microVM. Its memory stays in the RAM of the node. The executor closes the guest link first, because a paused guest answers no keepalive ([Guest link](/reference/guest-link/index.html.md#transport)). A call can fail after the VMM applied it. So after a call that fails, `Instance.Paused` gives the state that the driver read from the VMM. A state that the driver cannot read counts as paused, and the error then wraps `vmm.ErrStateUnknown`. So the caller resumes such a guest, or pauses it again, and never takes it for paused. |
| `vmm.Pauser` | `Resume(ctx, inst)` | Runs a paused microVM again. The executor first gives it a new link key with `Instance.SetLinkKey`, and serves a new guest link. |

## Snapshotter and Restorer

| Interface | Method | Description |
| --- | --- | --- |
| `vmm.Snapshotter` | `Snapshot(ctx, inst)` | Saves the memory and the state of a running or paused microVM to the memory file of the machine. Then it stops the microVM with the exit reason `snapshot`. It returns the `MemoryState` of the new file. The memory file appears whole or not at all. A new file replaces the one before it, and the old file goes before the new one gets its name. An error that wraps `vmm.ErrSaved` comes with the `MemoryState` of a whole file. The microVM then ends later, with the exit reason `snapshot`. A new `Snapshot` or a `Stop` of that instance only ends the microVM, with the exit reason `snapshot`, and saves nothing again. On any other error, the microVM runs or stays paused as before, and no new memory file exists. |
| `vmm.Restorer` | `Restore(ctx, Boot)` | Boots a machine from its memory file, with a new link key. It counts the load first. The first load of a saved state makes a copy of each disk. Each later load puts that copy back, so the memory and the disks match. `Restore` keeps the memory file, because a load succeeds without a read of the memory. A failure keeps every file, and leaves no microVM of the machine running. |
| `vmm.Restorer` | `DropMemory(ctx, id)` | Drops the memory file of a machine and the disk copies of its saved state. The executor calls it after the guest agent of a wake linked ([the node executor](/reference/executor/index.html.md#saved-state)). A microVM that maps the file runs on. A machine without a memory file succeeds. |
| `vmm.Restorer` | `DiscardMemory(ctx, id)` | Drops the saved state of a machine that does not run, for the call `DiscardMemory` ([the machine API](/reference/machine-api/index.html.md#discardmemory)). A running machine refuses. It first puts the disk copy of the saved state back over the disk. So the next boot starts from the disk of the save, not from a disk that a failed wake changed. This is the default of Q205. A machine without a memory file succeeds. |

| Error | Go value | Meaning |
| --- | --- | --- |
| `vmm: the memory file of the save is whole` | `vmm.ErrSaved` | A save wrote a whole memory file, but its microVM did not end. The caller keeps the memory file, and never runs the microVM again. |

A wake runs only after a load. So only the guest link proves that the memory file holds the guest, and the file stays until the guest agent of the wake linked.

## Mover

A driver that keeps the disk of each machine in one file is a `vmm.Mover`. A cold move copies that file to another node ([Cold move](/reference/cold-move/index.html.md)). A node whose driver is not a `Mover` takes part in no move.

| Interface | Method | Description |
| --- | --- | --- |
| `vmm.Mover` | `DiskPath(id)` | The path of the disk file of a machine. The file exists from the first boot of the machine until `Delete`. A boot keeps a disk that it finds there, such as a pulled one. `Delete` also removes the partial copy of a pull beside the file. |

## UsageReader

A driver that reads what its microVMs use on the host is a `vmm.UsageReader`. The node serves these values as the [metrics of the machines](/reference/metrics/index.html.md#metrics-of-the-machines). A node whose driver is not a `UsageReader` serves none of these values. It still serves `hopbox_machine_info`, `hopbox_machine_vcpus`, and the series of saved state. The test driver is not one.

| Interface | Method | Description |
| --- | --- | --- |
| `vmm.UsageReader` | `Usage(id)` | The CPU time and the host RAM of the microVM of a running machine and of its VMM, as a `vmm.Usage`. It fails for a machine that does not run. A `vmm.Usage` also holds the CPU pressure, the OOM kills, and the bytes of disk IO. When the field `Has...` of a value is false, that value is not valid. |
| | `TotalUsage()` | The same values for every microVM of the node together. |
| | `DiskUsage(id)` | The bytes of the blocks that the disk of a machine holds on the node. |

## The test driver

`vmmtest.New(dir, logger, capabilities...)` returns a driver for tests. Its microVM is the real guest agent of `internal/guest`, run in the test process. It connects to the guest link socket over a unix socket. So an executor goes through the same steps as on a real node, without KVM. Its microVMs live as long as the driver, so they outlive the `Manager` that booted them. This is how a microVM outlives `hopboxd`.

| Method | Description |
| --- | --- |
| `Crash(id)`, `Shutdown(id)` | Ends the running microVM of a machine as a crash, or as a shutdown from inside. |
| `FailBoots(err)`, `FailWakes(err)` | Makes each boot or wake fail with `err`, until the next call with nil. A wake that fails still counts its load. |
| `FailSnapshots(err)`, `FailDrops(err)` | Makes each save, or each drop of a memory file, fail with `err`, until the next call with nil. After a failed save, the microVM runs on, and an earlier memory file stays. `FailDrops` also fails each `DiscardMemory`. |
| `SaveWithoutAnEnd(on)` | With true, each save keeps its memory file, as a save whose unit did not end in time. The save then fails with an error that wraps `vmm.ErrSaved`. The guest agent stops at once, and the microVM ends 200 ms later with the exit reason `snapshot`. With false, saves end their microVM again. |
| `SaveThatEndsAtAStop(on)` | With true, each save keeps its memory file, and fails with an error that wraps `vmm.ErrSaved`. The microVM ends only at a `Stop` or at a new `Snapshot`, each with the exit reason `snapshot`. |
| `FailStops(err)` | Makes each `Stop` fail with `err`, and the microVM run on, until the next call with nil. |
| `HangWakes(hang)` | With true, each wake runs a microVM whose guest agent never links, as a memory file with damaged bytes does. With false, wakes link again. |
| `LinkHungWake(id)` | Starts the guest agent of a running microVM whose wake `HangWakes` hung, as a guest that links late. |
| `Loads(id)` | How many times the memory file of a machine loaded, also in wakes that failed. A new save and a drop start it at 0. |
| `SetDelay(d)` | Makes each call wait `d` before it acts. A call whose context ends during the wait fails with the cause of the end. A stop then kills the microVM. |
| `MostSavesAtOnce()` | The most saves that ran at once, of any machines of the driver. |
| `Ops()` | The calls so far, in order. |
| `Overlaps()` | Each call that ran while another call of the same machine ran. An executor must never cause one. |
| `Guarded(id, pgid)` | Whether a watchdog of the running microVM of a machine guards the process group `pgid` (see below). |
| `Boots(id)`, `Status(id)`, `Root(id)` | How many times a machine booted, what the driver holds of it, and the dir of the `/etc` of its guest. |
| `DiskPath(id)` | The disk file of a machine, `<dir>/disks/<id>/disk.raw`: a sparse file of the disk size that the first boot makes, at least 1 MiB. The guest agent of the driver never writes to it, so a test of a cold move writes its data. `Delete` removes its dir. |
| `Close()` | Ends every microVM, at the end of a test. |

A `Stop` of a paused microVM records the call `resume_for_stop` in `Ops()`, because the Firecracker driver runs a paused guest for its shutdown. `Kill` records `kill`, with no resume.

The test driver keeps a disk copy as the Firecracker driver does. The first load of a saved state copies the disk to `disk-copy.raw` beside it, and keeps the holes of the sparse file. A later load puts the copy back over the disk. `DiscardMemory` puts the copy back and drops the memory file. `DropMemory` and a new save remove the copy.

The guest agent starts each command, exec or start command, in a process group of its own. So a Ctrl-C in the terminal does not reach the command. A killed test binary also skips `Close()`.

To keep such commands from running on, the driver starts a watchdog for each command (`internal/watchdog`). A watchdog is a `/bin/sh` that waits on a pipe whose write end only the test binary holds. When the test binary dies, for any reason, the kernel closes that end, and the watchdog sends `SIGKILL` to the process group of the command. On macOS, a child that a process of the group forks during the kill can miss the signal. So the watchdog sends `SIGKILL` to the group again every 0.1 s while the group exists, for 2 s at most. A `SIGKILL` to the `go` command alone does not end the test binary. So the watchdog fires at the end of the test binary. At the latest, `-test.timeout` ends it.

After a command exits, its background children stay in its process group. So the driver keeps the watchdog until the process group is empty, or until the microVM stops. A real stop ends each process of the microVM. So when the microVM stops, the driver sends `SIGKILL` to each process group that it still keeps, and stops its watchdog. As the watchdog does, the driver sends `SIGKILL` to a group again every 0.1 s while the group exists, for 2 s at most. Both take this rule from the same constants of `internal/watchdog`, and the driver calls `watchdog.KillGroups`.

The guest agent reports each command to the driver through `guest.Options.Started`. A real guest leaves that option empty. The agent reports a command after the command starts. So a test binary that dies before the driver starts the watchdog leaves the command running. The test `TestGuestCommandsEndWithTheTestProcess` in `internal/node/executor` kills a test binary with `SIGKILL` after `Guarded` reports the group of the command. It checks that the start command of its machine and the children of that command exit within 5 s. It also checks this for a child that outlives its start command, and for an exec of `yes`.

The dir of the driver must have a short path. A unix socket path has at most 104 bytes on macOS.
