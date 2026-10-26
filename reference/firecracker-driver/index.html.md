# The Firecracker driver

> The Firecracker driver of the node role. It covers the unit, the jail, and the disk of each machine, the caches of images and kernels, and the KVM tests.

The Firecracker driver is the VMM driver of the node role ([the VMM driver interface](/reference/vmm/index.html.md)). It runs the microVM of each machine with Firecracker v1.17.0 under the jailer, the Firecracker tool that isolates each microVM process. Each microVM runs in a transient systemd unit of its own. A transient unit is a unit that a program starts through systemd, without a unit file. So when `hopboxd` restarts or upgrades, each microVM keeps running. The code is in `internal/node/vmm/firecracker`.

`hopboxd run` makes the driver on a server with the node role ([hopboxd](/reference/hopboxd/index.html.md#run)). The executor of each machine is the only code that calls it ([the node executor](/reference/executor/index.html.md)).

## Capabilities

The driver declares no capability in phase 1. It boots, stops, attaches to, and deletes machines. Pause, snapshot, restore, and fork come in phase 3.

`SnapshotFormat` runs `firecracker --snapshot-version` once, with at most 5 s, and returns the version that it prints, such as `v12.0.0`. The compatibility class of the node holds it ([the control channel](/reference/control-channel/index.html.md#compatibility-class)).

## Files of a machine

`<data_dir>` is `data_dir` of the bootstrap config, `/var/lib/hopbox` by default ([Server layout](/reference/server-layout/index.html.md#paths)). Each dir is on one file system with reflinks, XFS or btrfs, so a disk and a kernel can enter a jail as hard links.

| Path | Owner | Mode | What it holds |
| --- | --- | --- | --- |
| `<data_dir>/machines/<id>/disk.raw` | the jail user of the machine | 0600 | The disk of the machine. See [Disks](#disks). |
| `<data_dir>/machines/<id>/machine.json` | root | 0600 | The jail ID of the machine, as `{"jail_id":1879048192}`. |
| `<data_dir>/jail/firecracker/<id>/root/` | the jail user | 0700 | The root of the jail. The jailed Firecracker sees it as `/`. |
| `.../root/firecracker` | the jail user | 0755 | The copy of Firecracker that the jailer makes at each boot. A running microVM keeps its copy, so an upgrade does not change it. |
| `.../root/vmlinux` | root | 0644 | The guest kernel, as a hard link to the kernel cache. |
| `.../root/rootfs` | the jail user | 0600 | The disk, as a hard link to `disk.raw`. |
| `.../root/run/firecracker.socket` | the jail user | socket | The API socket of Firecracker. |
| `.../root/v.sock` | the jail user | socket | The vsock device socket. Firecracker listens on it. |
| `.../root/v.sock_10240` | root | 0666 socket | The guest link socket. The executor listens on it, and Firecracker gives it each connection of the guest agent to vsock port 10240 ([Guest link](/reference/guest-link/index.html.md#transport)). Only root and the jail user can enter the jail dir, so the socket is open to every user. |
| `.../root/dev/` | the jail user | | The device nodes that the jailer makes: `kvm`, `net/tun`, and others. |
| `<data_dir>/kernels/<sha256>` | root | 0644 | One guest kernel. See [The kernel cache](#the-kernel-cache). |
| `<data_dir>/images/<hex>/` | root | 0700 | One image. See [The image cache](#the-image-cache). |

A unix socket path has at most 107 bytes on Linux. The API socket of a machine has the path `<data_dir>/jail/firecracker/<id>/root/run/firecracker.socket`, so `data_dir` has at most 25 characters. The bootstrap config refuses a longer path on a server with the node role ([the bootstrap config of hopboxd](/reference/hopboxd-config/index.html.md#keys)), so an install stops before it changes anything.

## The unit of a machine

Each microVM runs in the unit `hopbox-machine-<id>.service`, in the slice `hopbox-machines.slice` ([Server layout](/reference/server-layout/index.html.md#hopbox-machinesslice)). The driver starts it through the D-Bus API of systemd. It first tries the private socket of systemd, which only root can use, and then the system bus.

| Property | Value |
| --- | --- |
| `Description` | `hopbox machine <id>` |
| `Slice` | `hopbox-machines.slice` |
| `Type` | `exec`. systemd reports the start once the jailer runs. |
| `ExecStart` | the jailer, with the arguments below |
| `MemoryMax` | the RAM of the machine plus 256 MiB. The extra memory is for Firecracker and for the page cache of the disk. |
| `TasksMax` | 256. Firecracker runs one thread for each vCPU and a few of its own. |
| `LogRateLimitIntervalUSec`, `LogRateLimitBurst` | 30 s and 1000. The journal takes at most 1000 lines of the unit in 30 s, a tenth of the default of journald. A guest can write to its serial console without end, and journald drops the lines above the limit. |
| `ExecStop` | `hopboxd stop-guest --api-socket <API socket> --timeout <StopTimeout>`, with the path of the `hopboxd` that booted the machine, such as `/opt/hopbox/current/hopboxd`. systemd ignores a failure of the command, as the prefix `-` of a unit file says. See [A stop through systemd](#a-stop-through-systemd). A driver without `StopHelper` sets no `ExecStop`. |
| `TimeoutStopUSec` | `StopTimeout` plus 10 s, 40 s by default. systemd waits this long for `ExecStop`, and then this long again for the end of Firecracker after `SIGTERM`. If Firecracker still runs, systemd sends `SIGKILL` and can wait a third time. So a stop takes at most 2 min. |
| `Before` | `hopboxd.service`, the unit of `DaemonUnit`. At a shutdown, systemd stops both units, and this order makes it stop `hopboxd` first. A restart of `hopboxd` alone does not stop the unit. |
| `RequiresMountsFor` | `<data_dir>/machines` and `<data_dir>/jail`. At a shutdown, systemd unmounts the file system of the disks only after the unit stopped. |

The jailer gets these arguments:

| Argument | Value |
| --- | --- |
| `--id` | the machine ID |
| `--exec-file` | `vmm.firecracker` of the bootstrap config ([the bootstrap config of hopboxd](/reference/hopboxd-config/index.html.md#keys)) |
| `--uid`, `--gid` | the jail ID of the machine. See [Jail users](#jail-users). |
| `--chroot-base-dir` | `<data_dir>/jail` |
| `--cgroup-version` | `2` |
| `--parent-cgroup` | the cgroup of the unit, such as `hopbox.slice/hopbox-machines.slice/hopbox-machine-<id>.service`. Firecracker runs in it already. When a cgroup `/firecracker` exists and the argument is missing, the jailer moves Firecracker into that cgroup. |
| `--netns` | `NetNS` of the config: by default `/proc/<pid of hopboxd>/ns/net`, the network namespace of `hopboxd`. The driver makes the tap device there, so Firecracker must open it there. A unit of systemd starts in the namespace of the host. So for a `hopboxd` in a namespace of its own, as in the KVM tests, only this argument puts Firecracker next to its device. |
| after `--` | `--api-sock /run/firecracker.socket`, for Firecracker |

The jailer makes no cgroup of its own. The limits of the unit apply to Firecracker. The jailer then runs Firecracker in the same process, so the main process of the unit is Firecracker. It uses the default seccomp filter of Firecracker.

The serial console of the guest and the log of Firecracker go to the journal of the unit:

```sh
journalctl -u hopbox-machine-01a1460c-1f6e-7b8a-9c42-3d5e7f8a9b10.service
```

A stop of `hopboxd.service` stops only the processes of `hopboxd.service`. The unit of a machine is not part of it, so the microVM keeps running.

A reboot or a shutdown of the server shuts each guest down cleanly. See [A stop through systemd](#a-stop-through-systemd).

## Jail users

Each machine runs as its own user and group ID, its jail ID. The IDs come from the range of the user `hopbox-jail`, 1879048192 to 1879113727 ([Server layout](/reference/server-layout/index.html.md#users-and-ids)). At the first boot of a machine, the driver gives it the lowest free ID of the node. It writes the ID to `machine.json`. The machine keeps the ID until its delete. When all 65536 IDs are in use, a boot fails with `firecracker: all 65536 jail IDs are in use`.

## Boot

A boot runs these steps. The executor then waits for the guest agent ([the node executor](/reference/executor/index.html.md#actions)).

1. The driver checks the size of the machine. Firecracker takes 1 vCPU or an even number of vCPUs, up to 32. The RAM is a whole number of MiB, at least 128 MiB.
2. It refuses a machine whose unit runs, and resets the failed state of an old unit.
3. It takes the kernel that the image names from the kernel cache.
4. It takes the disk of the image from the image cache. The first boot of an image can unpack or download it.
5. It makes the disk of the machine, at its first boot. See [Disks](#disks).
6. It fills the jail: it removes what an earlier boot left there, apart from the guest link socket, and makes hard links to the kernel and the disk.
7. It makes the tap device of the machine, and connects it to the node network. See [The tap device](#the-tap-device).
8. It starts the unit, and waits at most 10 s for the API socket of Firecracker.
9. It configures the microVM through the API, and starts it.

| API call | Body |
| --- | --- |
| `PUT /machine-config` | `vcpu_count`, `mem_size_mib`, and `smt` false. No CPU template. |
| `PUT /boot-source` | `kernel_image_path` `/vmlinux`, and the boot arguments of [Guest kernel](/reference/guest-kernel/index.html.md#boot-requirements). The kernel command line holds no secret. |
| `PUT /drives/rootfs` | The root disk `/rootfs`, read-write, with `cache_type` `Writeback` and `discard` true. See [Disks](#disks). |
| `PUT /network-interfaces/eth0` | The tap device, and a MAC address from a hash of the machine ID, `06:00:xx:xx:xx:xx`. |
| `PUT /mmds/config` | MMDS version 2 on `eth0`, at `169.254.169.254`. |
| `PUT /mmds` | `{"hopbox": {"guest_link_key": "<64 hex characters>"}}`, the link key of the boot ([Guest link](/reference/guest-link/index.html.md#link-key)). |
| `PUT /vsock` | `guest_cid` 3, `uds_path` `/v.sock`. The vsock device of Firecracker carries the connections over unix sockets, so every guest can have the address 3. |
| `PUT /actions` | `InstanceStart` |

`Boot` returns once Firecracker started the microVM. If a step fails after the start of the unit, the driver kills the unit and waits until it stopped. Then it resets the failed state of the unit and removes the tap device. So no microVM of the machine runs after a failed boot. Each step has its own error, such as `firecracker PUT /actions: 400 Bad Request: <message of Firecracker>`.

## Disks

The disk of a machine is a reflink copy of the disk of its image, made with the `FICLONE` call of Linux. Then the driver makes the file as large as the disk size of the machine, 100 GB for the default size. The file stays sparse: only the blocks that the machine writes take space of their own. At boot, `systemd-growfs` in the guest grows the file system to the end of the disk ([Default image](/reference/default-image/index.html.md#disk)).

| Item | Value |
| --- | --- |
| When | At the first boot of the machine. Each later boot uses the disk that exists. A larger disk size grows it at the next boot. |
| File system | XFS with reflinks or btrfs. On another file system, the boot fails with `the file system of <dir> has no reflinks`. |
| Cache mode | `Writeback`: a flush in the guest reaches the disk of the node. |
| Discard | On: the guest can give freed blocks back, so the disk stays sparse. |
| Delete | `Delete` removes the disk with the machine dir. |

## The image cache

`<data_dir>/images/<hex>` holds the image with the digest `sha256:<hex>` ([Server layout](/reference/server-layout/index.html.md#the-image-cache)). The driver takes the disk of an image in this order:

1. When the dir has `artifact.json`, it must hold the image of the digest. Its disk must have the sha256 and the size that the control role sends, and it must name the kernel of the spec.
2. When the disk is there, the driver checks its size and its sha256 at the first use in each process of `hopboxd`. For the default image of 3.5 GiB, this takes about 9 s on the [tested server](/reference/preflight/index.html.md#the-tested-server).
3. An install leaves only the compressed copy. In that case, the driver checks the copy and unpacks it with zstd. Then it checks the disk and removes the copy. The unpack of the default image takes about 22 s on the tested server. The disk keeps a hole for each block of 64 KiB of zeros.
4. If the dir has no disk and the image has a URL, the driver downloads the compressed disk from the URL. It unpacks it and checks the disk against the sha256 of the image. Only an `https` URL works. If the download gets larger than the disk plus an eighth, the driver stops it. The dir then holds only `disk.raw`. A later install of a release with the same image adds its `artifact.json`.
5. Otherwise the boot fails with `firecracker: the image cache has no image <digest>, and the image has no URL`.

One job of the process prepares each image. It runs apart from the boot that started it, with a deadline of 2 hours. So a download can take longer than the timeout of one boot. The next boot of the machine then waits for the same job. Two boots of one image at the same time wait for one job. A boot whose wait ends first fails with `firecracker: the image <digest> is not ready yet, and the node goes on with it`, and the restart policy boots the machine again later.

Each unpack and download writes a temp file `.<name>.tmp-<random>` in the dir of the image, and renames it only after the checks. At the first use of an image in a process, the job removes each such file that an earlier process left. It also removes a compressed copy that stayed beside a checked disk. The cache never removes an image.

A disk that does not match fails the boot with an error such as `artifact file disk.raw: sha256 is <got>, the metadata says <want>`. Remove the dir of the image. The next boot unpacks or downloads it again.

## The kernel cache

`<data_dir>/kernels/<sha256>` holds each guest kernel that the node got with a release. The image names the kernel that it was built and tested with, by its sha256 ([Artifact format](/reference/artifact-format/index.html.md#artifactjson)). With the cache, a machine still boots after an upgrade brings another kernel.

At each start of `hopboxd run`, the node adds the kernel of `vmm.kernel` to the cache, unless the cache has it. When the kernel is missing, `hopboxd` logs a warning and starts anyway. Each boot checks the sha256 of a kernel at its first use in the process. A boot whose kernel is not in the cache fails with `firecracker: the node has no guest kernel with the sha256 <sha256>, which the image needs`. The cache never removes a kernel.

## The tap device

Each machine has a tap device, a virtual network device of Linux, in the network namespace of `hopboxd`. The guest reaches its node and the internet through it ([the node network](/reference/node-network/index.html.md)). The metadata service of Firecracker, which holds the link key, also needs a network interface in the guest.

| Item | Value |
| --- | --- |
| Name | `hbt` and the first 12 hex digits of the sha256 of the machine ID, such as `hbt3f2c9a0b1d4e` |
| Owner and group | the jail ID of the machine, so the jailed Firecracker can open it |
| State | For a machine with an address, the driver calls `Network.Connect` of its config before it starts the unit. The device is then up, with the gateway address and the route of the machine ([the node network](/reference/node-network/index.html.md#the-tap-device-of-a-machine)). A failed connect fails the boot. A machine without an address, or a driver without `Network`, keeps the device down, and the driver logs `the machine boots without network`. Firecracker answers the requests of the guest to the metadata service itself, so they never reach the device. |
| Removal | when the microVM ends while `hopboxd` watches it, after a failed boot, and at a delete. The driver first calls `Network.Disconnect`, so the table of the node network drops the device. |

## Stop

`Instance.Stop` sends `SendCtrlAltDel` through the API. The guest reboots, and with `reboot=k` a reboot ends Firecracker. The stop waits for the end of the process. After 30 s, or at the end of the context of the call, the driver kills the unit with `SIGKILL`. On the tested server, a stop of the default image takes about 0.4 s.

## A stop through systemd

A reboot or a shutdown of the server stops each unit through systemd. So does `sudo systemctl stop hopbox-machine-<id>.service`. Without `ExecStop`, systemd sends `SIGTERM`, which ends Firecracker at once, as a power loss does. So the unit of each machine gets an `ExecStop`, and a stop runs these steps:

1. At a shutdown, systemd stops `hopboxd.service` first, because each unit is ordered before it. So no executor sees its guests go down, and none applies the restart policy or reports a state.
2. systemd runs `hopboxd stop-guest`. It sends `SendCtrlAltDel` through the API socket, as `Instance.Stop` does.
3. The guest shuts down. It writes its page cache to the disk, and unmounts its file systems. Then it reboots, and with `reboot=k` Firecracker ends with status 0.
4. `stop-guest` sees the end of the process, and exits with status 0. The unit is then inactive.

If the guest does not end within `StopTimeout`, 30 s by default, `stop-guest` exits with status 1. If the call to Firecracker fails, and Firecracker does not end within 2 s, `stop-guest` exits with status 1 too. systemd ignores the status, and sends `SIGTERM` to Firecracker. Firecracker ends at once, as at a power loss. The unit then ends inactive, not failed, because `SIGTERM` is a clean end for systemd. The root file system of the guest is still in use, so the driver reports `crash`.

systemd runs `ExecStop` also after Firecracker ended by itself, as after a reboot from inside. It then leaves `MAINPID` unset, and `stop-guest` exits with status 0 at once.

The next boot of the server brings each machine back. The target of each machine stays as it was, so the executor boots each machine with the target `running`, whatever its restart policy ([the node executor](/reference/executor/index.html.md#restart-policy)).

A stop of one unit while `hopboxd` runs is a stop that `hopboxd` did not ask for. When the guest shuts down cleanly, the exit reason is `shutdown`, as for a shutdown from inside. When systemd had to kill Firecracker, the exit reason is `crash`. The executor then applies the restart policy. With `always`, the machine boots again after the backoff. With `never`, a clean stop sets the target to `stopped`. To stop a machine, use the API or `hopbox machine stop` instead.

A unit keeps the properties of the boot that started it. So a microVM that a release without `ExecStop` booted still gets `SIGTERM` at a shutdown, until its next boot. After a rollback to a release without `stop-guest`, the units of the later release run the older `hopboxd`. It exits with status 80, `unexpected argument stop-guest`. systemd ignores that status. A shutdown or a reboot from inside thus still counts as `shutdown`. A stop through systemd sends `SIGTERM` at once, as before `stop-guest`, until the next boot of the machine. Each later release keeps the flags of `stop-guest`. A unit runs the `hopboxd` of the current release, with the flags of the release that booted it.

## Exit reasons

The driver watches the main process of the unit with a pidfd, a file descriptor of a process. When the process ends, it asks systemd why.

| What ended the process | Unit after the end | Root file system | Exit reason |
| --- | --- | --- | --- |
| A stop of the driver | any | any | `stopped` |
| A shutdown or a reboot in the guest, or a stop through systemd that the guest finished in time | inactive | clean | `shutdown` |
| A kernel panic or a reset in the guest | inactive | in use | `crash`, with the error `the guest ended without a clean unmount of its root file system, as after a kernel panic or a reset` |
| A crash of Firecracker or a signal | failed | any | `crash`, with an error such as `the Firecracker process ended: unit failed/failed, result signal, killed by SIGKILL` |
| A stop through systemd that the guest did not finish in time | inactive | in use | `crash`, with the error of a kernel panic above |

A kernel panic and a reboot both end Firecracker with status 0, because of `panic=1` and `reboot=k`. So the driver reads the ext4 superblock of the disk of the machine after the end. A shutdown or a reboot makes the root file system read-only first, which clears the journal flag `needs_recovery`. After a panic or a reset, the flag stays set. The driver reads only this flag. The error state of ext4 tells nothing about the last boot, because it stays across clean unmounts until `e2fsck` clears it. A disk without ext4 or without a journal tells nothing, so its end counts as `shutdown`, and the driver logs a warning. A panic before the guest mounts its root file system read-write also counts as `shutdown`, because the flag is not set yet.

A poweroff or a halt in the guest also ends Firecracker. Firecracker has no power-off device, so the default image turns both into a clean reboot at their last step ([Default image](/reference/default-image/index.html.md#files)).

The driver resets the failed state of the unit, so the name is free for the next boot.

## Attach after a restart or an upgrade

After a restart of `hopboxd`, the executor asks the driver for each machine of its node-local state ([the node executor](/reference/executor/index.html.md#node-local-state)). `Inspect` reports a machine as running while its unit is `active`, `activating`, `deactivating`, `reloading`, or `refreshing`. `Attach` then asks the API of Firecracker for the state of the microVM. For a microVM that runs, it watches the main process of the unit again. The executor then gives the microVM a new link key through `PUT /mmds`, and the guest agent connects again.

A microVM in the state `Not started` comes from a boot that `hopboxd` left before `InstanceStart`. Its link key went with that `hopboxd`. So `Attach` kills its unit and returns `vmm.ErrNotRunning`, and the executor boots the machine again.

An upgrade changes the files of `vmm.firecracker` and `vmm.kernel`. A running microVM keeps its copy of Firecracker in its jail and the kernel that it booted with. The next boot of a machine uses the new Firecracker.

## Delete

`Delete` refuses a machine whose unit runs. It resets a failed unit, removes the tap device, the jail dir, and the machine dir with the disk. The jail ID of the machine is then free.

## Logs

Every record has `component=firecracker`.

| Log message | Level | Fields |
| --- | --- | --- |
| `booted a microVM` | info | `machine`, `unit`, `jail_id`, `tap`, `duration` |
| `attached to a microVM` | info | `machine`, `unit`, `state` |
| `stopping a microVM that never started` | warn | `machine`, `unit` |
| `made the disk of a machine as a reflink copy of its image` | info | `machine`, `image` |
| `unpacked the disk of an image` | info | `digest`, `duration` |
| `downloaded the disk of an image` | info | `digest`, `url`, `duration` |
| `checked the disk of an image` | info | `digest`, `duration` |
| `cannot remove the compressed disk of an unpacked image` | warn | `digest`, `err` |
| `removed a temp file of the image cache` | info | `file` |
| `cannot remove a temp file of the image cache` | warn | `file`, `err` |
| `cannot ask the guest to shut down, so the microVM gets killed` | warn | `machine`, `err` |
| `the guest did not shut down in time, so the microVM gets killed` | warn | `machine`, `timeout` |
| `cannot stop the unit of a failed boot` | error | `machine`, `unit`, `err` |
| `the machine boots without network` | warn | `machine`, `address`, `network` |
| `cannot remove the tap device of a machine` | warn | `machine`, `err` |
| `cannot watch a microVM` | error | `machine`, `err` |
| `cannot reset a failed unit` | warn | `unit`, `err` |
| `cannot read the root file system of a stopped guest, so its end counts as a shutdown` | warn | `machine`, `err` |
| `the root file system of the guest is not ext4 with a journal, so its end counts as a shutdown` | warn | `machine` |

`hopboxd stop-guest` logs to the journal of the unit of the machine, at level info ([hopboxd](/reference/hopboxd/index.html.md#stop-guest)). For a stop of a running guest, it logs `asking the guest to shut down` and then `the guest shut down`. On a failure, it prints the error and exits with status 1, for example `firecracker: the guest did not shut down within 30s`.

The driver has no metrics of its own. The executor counts each boot, stop, and delete ([the node executor](/reference/executor/index.html.md#metrics)).

## Go API

`firecracker.New(Config)` checks the config and returns a `Driver`. It touches no file and no system service, so a node without KVM still starts, and each boot names what is missing.

| Field of `Config` | Default | Description |
| --- | --- | --- |
| `Firecracker` | required | The path of Firecracker. Its file name must be `firecracker`, because the jailer names the jail dir after it. |
| `Jailer` | required | The path of the jailer. |
| `ImagesDir`, `KernelsDir`, `MachinesDir`, `JailDir` | required | The dirs of [Files of a machine](#files-of-a-machine). |
| `JailIDFirst`, `JailIDCount` | required | The range of the jail IDs. |
| `Slice` | `hopbox-machines.slice` | The slice of the units. |
| `UnitPrefix` | `hopbox-machine-` | The start of the unit names. |
| `TapPrefix` | `hbt` | The start of the tap device names, at most 4 characters. |
| `Network` | nil | The node network, such as a `netcfg.Network`. Its `Connect` and `Disconnect` take the tap device of each machine with an address. Nil keeps each device down. |
| `NetNS` | `/proc/<pid>/ns/net` of the process | The network namespace of the microVMs, for `--netns` of the jailer. It must be an absolute path. |
| `StopTimeout` | 30 s | The wait for the guest at a stop, also in `stop-guest`. |
| `StopHelper` | empty | The absolute path of `hopboxd`, for the `ExecStop` of each unit. `hopboxd run` sets the path that started it, such as `/opt/hopbox/current/hopboxd`. Empty sets no `ExecStop`, so a stop through systemd ends Firecracker with `SIGTERM`. |
| `DaemonUnit` | empty | The unit of `hopboxd`. Each unit of a machine is ordered before it. `hopboxd run` sets `hopboxd.service`. Empty adds no order. |
| `StartTimeout` | 10 s | The wait for the API socket of a new Firecracker. |
| `MemoryOverhead` | 256 MiB | The memory of a unit beyond the RAM of the guest. |
| `HTTPClient` | a client with the proxy of the environment | The client of image downloads. |
| `Logger` | `slog.Default()` | The logger. |

| Method | Description |
| --- | --- |
| `AddKernel(ctx, path)` | Adds the kernel at `path` to the kernel cache, and returns its sha256. |
| `SnapshotFormat(ctx)` | The snapshot format version of Firecracker. |
| `Close()` | Closes the connection to systemd. A later call makes a new one. `hopboxd run` calls it after the stop of the unit `machines`. |

The function `firecracker.StopGuest(ctx, sock, pid, timeout)` is the body of `hopboxd stop-guest`. It sends `SendCtrlAltDel` to the API socket `sock`, and waits for the end of the process `pid`, at most for `timeout`. A `pid` of 0 returns at once.

The methods of `vmm.Driver` follow [the VMM driver interface](/reference/vmm/index.html.md#driver). The driver needs Linux. On another system, `Inspect`, `Boot`, `Attach`, and `Delete` fail with `firecracker: the driver runs only on Linux`.

## Tests

`make test` runs the unit tests of the driver on any system: the config checks, the names, the API client against a fake Firecracker, the image cache, the kernel cache, and the jail IDs. `make test-linux` runs them on Linux too.

`make test-kvm` runs the KVM tests. `TestKVMLifecycle` boots the default image with the real guest agent and checks the unit, the jail user, the cgroup, the tap device, and the disk. Then it restarts the driver and attaches, stops, and boots from the kept disk. It reboots and powers off from inside, which must give `shutdown`. A kernel panic and a kill of Firecracker must give `crash`. It checks `ExecStop`, `Before`, and `TimeoutStopUSec` of the unit. Then the guest turns its writeback off and writes a file without a sync, and `systemctl stop` stops the unit. That must give `shutdown`, and the file must be there at the next boot. Then the stop helper is `/bin/false`, as after a rollback. A reboot from inside must still give `shutdown`, and the unit must not end failed. A stop of the unit must then give `crash`, with a root file system in use. It also attaches to a microVM that never started, and deletes the machine. `TestKVMFailedBoot` checks that a failed boot leaves no unit and no tap device. `TestKVMSystemdConnection` checks that the connection to systemd outlives the call that made it. `TestKVMEndToEnd` of `internal/control` runs a single-host install. It makes a machine through the API, runs exec, stops and starts it, upgrades `hopboxd` with the microVM running, and deletes it. It also checks the node network with a second machine ([the node network](/reference/node-network/index.html.md#tests)). The KVM tests run in a network namespace of their own.

| Variable of `make test-kvm` | Default | Description |
| --- | --- | --- |
| `KVM_WORKDIR` | none, required | A dir on XFS with reflinks, of at most 22 characters. |
| `FIRECRACKER`, `JAILER` | `firecracker`, `jailer` | Firecracker v1.17.0 and its jailer, on the host of the tests. |
| `KVM_HOST` | empty | An SSH host that runs the tests. Empty runs them on this host. |
| `KVM_DIR` | `/var/tmp/hopbox-kvm` | The dir on `KVM_HOST` for the test binaries, the kernel, the image, and Postgres. |
| `KVM_TEST_ENV` | empty | More env vars for the tests. |
| `KERNEL_OUT`, `IMAGE_OUT`, `PG_LINUX` | as in the `Makefile` | The kernel, the image, and the Linux build of Postgres. |

| Env var of the tests | Default | Description |
| --- | --- | --- |
| `HOPBOX_TEST_KVM` | none | `1` runs the KVM tests. Without it, they skip. |
| `HOPBOX_TEST_FIRECRACKER`, `HOPBOX_TEST_JAILER` | none, required | The Firecracker binary and its jailer. `make test-kvm` sets them from `FIRECRACKER` and `JAILER`. |
| `HOPBOX_TEST_KERNEL` | none, required | The guest kernel `vmlinux`. `make test-kvm` sets it from `KERNEL_OUT`. |
| `HOPBOX_TEST_IMAGE` | none, required | The artifact folder of the default image. `make test-kvm` sets it from `IMAGE_OUT`. |
| `HOPBOX_TEST_WORKDIR` | none, required | The work folder of the tests, `KVM_WORKDIR` of `make test-kvm`. |
| `HOPBOX_PG_DIR` | none, required by `TestKVMEndToEnd` | The bundled Postgres for Linux. `make test-kvm` sets it to `PG_LINUX`, or to a Linux build of `tools/pgdist`. |
| `HOPBOX_TEST_HOPBOXD` | none, required | A `hopboxd` for Linux, for the `ExecStop` of the units of the tests. `make test-kvm` builds it. |
| `HOPBOX_TEST_UNIT_PREFIX` | `hopbox-test-machine-` | The start of the unit names of the tests. |
| `HOPBOX_TEST_SLICE` | `hopbox_test.slice` | The slice of the tests. |
| `HOPBOX_TEST_TAP_PREFIX` | `hbx` | The start of the tap device names of the tests. |
| `HOPBOX_TEST_JAIL_ID_FIRST` | `1879110000` | The first jail ID of `TestKVMLifecycle`. |
| `HOPBOX_TEST_PG_USER` | `hopbox-pg` | The user of Postgres in the tests of `internal/control`, which run as root. Set it on a host without the user `hopbox-pg`, such as `nobody`. |
