# The VMM driver interface

> The VMM driver interface of the executor, its capabilities, its drivers, and the test driver.

The VMM driver is the layer between the executor of a machine and the VMM that runs its microVM. The executor is the only code that calls a driver ([the node executor](/reference/executor/index.html.md)). Placement and the API never call a driver. Each driver declares its capabilities. For a feature that the driver does not declare, the executor does nothing. The code is in `internal/node/vmm`.

This release has two drivers. The Firecracker driver of `internal/node/vmm/firecracker` runs the machines of a node ([the Firecracker driver](/reference/firecracker-driver/index.html.md)). The test driver of `internal/node/vmm/vmmtest` lives in a test package and never ships.

## Capabilities

`Driver.Capabilities()` returns the capabilities of a driver. A driver can always boot, stop, attach to, and delete a machine.

| Capability | Go value | What it allows | Interface |
| --- | --- | --- | --- |
| pause | `vmm.CapPause` | standby: the microVM pauses in RAM | none yet |
| snapshot | `vmm.CapSnapshot` | hibernate: the memory goes to a memory file | `vmm.Snapshotter` |
| restore | `vmm.CapRestore` | wake: a boot from the memory file | `vmm.Restorer` |
| fork | `vmm.CapFork` | a new microVM from the state of another | none yet |
| migrate | `vmm.CapMigrate` | a move of a microVM to another node | none yet |

The Firecracker driver of phase 1 declares none.

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
| `Attach(ctx, id)` | Returns the microVM of a machine that runs, for example after a restart of `hopboxd`. It fails with `vmm.ErrNotRunning` for a machine that does not run. |
| `Delete(ctx, id)` | Removes the disk, the memory file, and every other file of a machine that does not run. A machine without files succeeds. |

| `Status` field | Description |
| --- | --- |
| `Disk` | The disk of the machine exists. |
| `Running` | The microVM runs. |
| `Memory` | A memory file of a hibernate is kept. Only `Delete` drops it. |

`Boot` holds the `Machine` and the link key of the boot:

| Field | Description |
| --- | --- |
| `ID` | The ID of the machine. |
| `VCPUs`, `MemoryBytes`, `DiskBytes` | The size of the machine. |
| `Image` | The image: its digest, its URL, and the SHA-256 of its kernel and of its disk ([the executor protocol](/reference/executor-protocol/index.html.md#machinespec)). |
| `Address` | The IPv4 address of the machine on its node, from the network of the spec, or the zero `netip.Addr` for a machine without one. A driver with a network device for each machine connects the device to the node network with it ([the node network](/reference/node-network/index.html.md)). |
| `LinkKey` | The link key of this boot, 32 random bytes. The driver puts it in the metadata service of the microVM, never on the kernel command line. |

## Instance

`Boot`, `Attach`, and `Restore` return an `Instance`: one running microVM.

| Method | Description |
| --- | --- |
| `Done()` | A channel that closes once the microVM stopped, for any reason. |
| `Exit()` | Why the microVM stopped, once `Done` closed. |
| `Stop(ctx)` | Stops the microVM, and returns once `Done` closed. The driver first asks the guest to shut down. At the end of `ctx`, it ends the microVM. A stopped microVM succeeds. |
| `SetLinkKey(ctx, key)` | Puts a new link key in the metadata service. The executor calls it after an attach, because the key of a boot lives only in memory. |

| Exit reason | Go value | Meaning |
| --- | --- | --- |
| `stopped` | `vmm.ExitStopped` | The end of `Stop`. |
| `shutdown` | `vmm.ExitShutdown` | A shutdown or a reboot from inside the machine. |
| `crash` | `vmm.ExitCrash` | An end that nobody asked for, such as a crash of the VMM. `Exit.Err` says what went wrong. |
| `snapshot` | `vmm.ExitSnapshot` | The end of `Snapshot`. The memory is in the memory file. |

## Snapshotter and Restorer

| Interface | Method | Description |
| --- | --- | --- |
| `vmm.Snapshotter` | `Snapshot(ctx, inst)` | Saves the memory and the state of a running microVM to the memory file of the machine, and stops the microVM. |
| `vmm.Restorer` | `Restore(ctx, Boot)` | Boots a machine from its memory file, with a new link key. A failure keeps the memory file, and leaves no microVM of the machine running. A success drops the memory file. |

## Mover

A driver that keeps the disk of each machine in one file is a `vmm.Mover`. A cold move copies that file to another node ([Cold move](/reference/cold-move/index.html.md)). A node whose driver is not a `Mover` takes part in no move.

| Interface | Method | Description |
| --- | --- | --- |
| `vmm.Mover` | `DiskPath(id)` | The path of the disk file of a machine. The file exists from the first boot of the machine until `Delete`. A boot keeps a disk that it finds there, such as a pulled one. `Delete` also removes the partial copy of a pull beside the file. |

## The test driver

`vmmtest.New(dir, logger, capabilities...)` returns a driver for tests. Its microVM is the real guest agent of `internal/guest`, run in the test process. It connects to the guest link socket over a unix socket. So an executor goes through the same steps as on a real node, without KVM. Its microVMs live as long as the driver, so they outlive the `Manager` that booted them. This is how a microVM outlives `hopboxd`.

| Method | Description |
| --- | --- |
| `Crash(id)`, `Shutdown(id)` | Ends the running microVM of a machine as a crash, or as a shutdown from inside. |
| `FailBoots(err)`, `FailWakes(err)` | Makes each boot or wake fail with `err`, until the next call with nil. |
| `SetDelay(d)` | Makes each call wait `d` before it acts. A call whose context ends during the wait fails with the cause of the end. A stop then kills the microVM. |
| `Ops()` | The calls so far, in order. |
| `Overlaps()` | Each call that ran while another call of the same machine ran. An executor must never cause one. |
| `Boots(id)`, `Status(id)`, `Root(id)` | How many times a machine booted, what the driver holds of it, and the dir of the `/etc` of its guest. |
| `DiskPath(id)` | The disk file of a machine, `<dir>/disks/<id>/disk.raw`: a sparse file of the disk size that the first boot makes, at least 1 MiB. The guest agent of the driver never writes to it, so a test of a cold move writes its data. `Delete` removes its dir. |
| `Close()` | Ends every microVM, at the end of a test. |

The guest agent starts each command, exec or start command, in a process group of its own. So a Ctrl-C in the terminal does not reach the command. A killed test binary also skips `Close()`.

To keep such commands from running on, the driver starts a watchdog for each command (`internal/watchdog`). A watchdog is a `/bin/sh` that waits on a pipe whose write end only the test binary holds. When the test binary dies, for any reason, the kernel closes that end, and the watchdog sends `SIGKILL` to the process group of the command. A `SIGKILL` to the `go` command alone does not end the test binary. So the watchdog fires at the end of the test binary. At the latest, `-test.timeout` ends it.

After a command exits, its background children stay in its process group. So the driver keeps the watchdog until the process group is empty, or until the microVM stops. A real stop ends each process of the microVM. So when the microVM stops, the driver sends `SIGKILL` to each process group that it still keeps, and stops its watchdog.

The guest agent reports each command to the driver through `guest.Options.Started`. A real guest leaves that option empty. The test `TestGuestCommandsEndWithTheTestProcess` in `internal/node/executor` kills a test binary with `SIGKILL`. It checks that the start command of its machine and the children of that command exit within 5 s. It also checks this for a child that outlives its start command, and for an exec of `yes`.

The dir of the driver must have a short path. A unix socket path has at most 104 bytes on macOS.
