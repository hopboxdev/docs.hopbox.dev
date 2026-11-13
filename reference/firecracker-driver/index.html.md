# The Firecracker driver

> The Firecracker driver of the node role. It covers the unit, the jail, and the disk of each machine, and the caches of images and kernels. It also covers the pause, the save and the load of a hibernate, the files of a saved state and their bytes, and the KVM tests.

The Firecracker driver is the VMM driver of the node role ([the VMM driver interface](/reference/vmm/index.html.md)). It runs the microVM of each machine with Firecracker v1.17.0 under the jailer, the Firecracker tool that isolates each microVM process. Each microVM runs in a transient systemd unit of its own. A transient unit is a unit that a program starts through systemd, without a unit file. So when `hopboxd` restarts or upgrades, each microVM keeps running. The code is in `internal/node/vmm/firecracker`.

`hopboxd run` makes the driver on a server with the node role ([hopboxd](/reference/hopboxd/index.html.md#run)). The executor of each machine is the only code that calls it ([the node executor](/reference/executor/index.html.md)).

## Capabilities

The driver declares the capabilities `pause`, `snapshot`, and `restore`. It boots, stops, attaches to, pauses, resumes, hibernates, wakes, and deletes machines ([Hibernate and wake](#hibernate-and-wake)). It also saves a running machine without a stop, as a `vmm.Saver` ([Save without a stop](#save-without-a-stop)). Fork and migrate come in later milestones of phase 3. Standby needs Firecracker 1.17.0, or 1.16.2 and later, because 1.16.0 and 1.16.1 can block vsock traffic after a pause. The release bundle holds 1.17.0, and the driver does not read the version.

`SnapshotFormat` runs `firecracker --snapshot-version` once, with at most 5 s, and returns the version that it prints, such as `v12.0.0`. The compatibility class of the node holds it ([the control channel](/reference/control-channel/index.html.md#compatibility-class)).

## Files of a machine

`<data_dir>` is `data_dir` of the bootstrap config, `/var/lib/hopbox` by default ([Server layout](/reference/server-layout/index.html.md#paths)). Each dir is on one file system with reflinks, XFS or btrfs, so a disk and a kernel can enter a jail as hard links.

| Path | Owner | Mode | What it holds |
| --- | --- | --- | --- |
| `<data_dir>/machines/<id>/disk.raw` | the jail user of the machine | 0600 | The disk of the machine. See [Disks](#disks). |
| `<data_dir>/machines/<id>/machine.json` | root | 0600 | The jail ID of the machine, as `{"jail_id":1879048192}`. |
| `<data_dir>/machines/<id>/state/` | root | 0700 | The saved state of a hibernate. Only a hibernated machine, and a machine whose wake did not end, has it. See [Hibernate and wake](#hibernate-and-wake). |
| `.../state/memory` | root | 0600 | The memory file: the RAM of the guest at the save. The name stands only for a whole file. |
| `.../state/vmstate` | root | 0600 | The state of the microVM at the save: its vCPUs and its devices. |
| `.../state/state.json` | root | 0600 | The record of the saved state. See [The record of a saved state](#the-record-of-a-saved-state). |
| `.../state/disk.raw` | root | 0600 | The reflink copy of the disk that the first load of the saved state makes. |
| `.../state/disk-<disk id>.raw` | root | 0600 | The reflink copy of each read-write extra disk that the first load makes. See [Extra disks](#extra-disks). |
| `<data_dir>/disks/<disk id>/disk.raw` | the jail user of the machine that holds the disk last | 0600, or 0400 for a read-only attach | An extra disk ([Extra disks](/reference/disks/index.html.md)). The disk keeper of the node makes, pulls, and removes it. The delete of a machine keeps it. |
| `<data_dir>/machines/<id>/save.partial/` | root | 0700 | The files of a save without a stop, before its dir gets its name. Each such save removes what an earlier one left, and `Delete` removes it with the machine dir. See [Save without a stop](#save-without-a-stop). |
| `<data_dir>/jail/firecracker/<id>/root/` | the jail user | 0700 | The root of the jail. The jailed Firecracker sees it as `/`. |
| `.../root/firecracker` | the jail user | 0755 | The copy of Firecracker that the jailer makes at each boot. A running microVM keeps its copy, so an upgrade does not change it. |
| `.../root/vmlinux` | root | 0644 | The guest kernel, as a hard link to the kernel cache. |
| `.../root/rootfs` | the jail user | 0600 | The disk, as a hard link to `disk.raw`. |
| `.../root/disk-<disk id>` | the jail user | 0600 | Each extra disk of the machine, as a hard link to its file. |
| `.../root/memory`, `.../root/vmstate` | the jail user | 0600 | The reflink copies of the memory file and of `vmstate` that a load reads. Firecracker maps the memory copy. A boot removes them. |
| `.../root/memory.new`, `.../root/vmstate.new` | the jail user, then root | 0600 | The files that a save writes, before it moves them into `state/`. A failed save removes them. |
| `.../root/rootfs.repair` | the jail user | 0600 | The work file of a repair: a reflink copy of `disk.raw`, on which `e2fsck` runs. After a repair, it becomes `disk.raw`. Else the driver removes it. A boot removes one that an earlier boot left. See [Repair of the root file system](#repair-of-the-root-file-system). |
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
| `IOAccounting` | `yes`. The cgroup of the unit gets the file `io.stat`, from which the node reads the disk IO of the machine ([Metrics](/reference/metrics/index.html.md#metrics-of-the-machines)). |
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
6. It repairs the root file system when its ext4 superblock has the error state. The boot goes on also after a failed repair. See [Repair of the root file system](#repair-of-the-root-file-system).
7. It fills the jail. It removes what an earlier boot left there, apart from the guest link socket. Then it makes hard links to the kernel, the disk, and each extra disk. See [Extra disks](#extra-disks).
8. It makes the tap device of the machine, and connects it to the node network. See [The tap device](#the-tap-device).
9. It starts the unit, and waits at most 10 s for the API socket of Firecracker.
10. It configures the microVM through the API, and starts it.

| API call | Body |
| --- | --- |
| `PUT /machine-config` | `vcpu_count`, `mem_size_mib`, and `smt` false. No CPU template. |
| `PUT /boot-source` | `kernel_image_path` `/vmlinux`, and the boot arguments of [Guest kernel](/reference/guest-kernel/index.html.md#boot-requirements). The kernel command line holds no secret. |
| `PUT /drives/rootfs` | The root disk `/rootfs`, read-write, with `cache_type` `Writeback` and `discard` true. See [Disks](#disks). |
| `PUT /drives/disk_<32 hex>` | Each extra disk, in the order of the spec, at `/disk-<disk id>`, with `is_root_device` false, `is_read_only` from the attach, and `cache_type` `Writeback`. A read-write disk has `discard` true. The drive ID holds only letters, digits, and `_`, as Firecracker asks. |
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
| Repair | Before a boot, the driver repairs a root file system with the ext4 error state, on a reflink copy. See [Repair of the root file system](#repair-of-the-root-file-system). |
| Delete | `Delete` removes the disk with the machine dir. |
| Cold move | `DiskPath(id)` names `disk.raw`, so the driver is a `vmm.Mover` ([the VMM driver interface](/reference/vmm/index.html.md#mover)). The target node of a move pulls the disk into `disk.raw.pull` of the machine dir, and renames it to `disk.raw` once its digest matches ([Cold move](/reference/cold-move/index.html.md#the-stream)). The next boot keeps that disk, and gives it to the jail user. |

## Extra disks

The extra disks of a machine come in `Disks` of `vmm.Machine`, in the order of their devices in the guest ([Extra disks](/reference/disks/index.html.md)). The driver is a `vmm.DiskHolder`: `DisksDir()` returns `<data_dir>/disks`, the dir of the disk keeper. The file of each disk is `vmm.DiskFile(<data_dir>/disks, <disk id>)`. The dir is on the file system of the jail dir, so a disk enters a jail as a hard link. The unit of a machine lists it in `RequiresMountsFor`.

| Step | What the driver does |
| --- | --- |
| Boot and load | It opens the file of each disk with `O_NOFOLLOW`, and refuses a file that is not a regular file. Through the descriptor, it gives the file to the jail user of the machine with mode 0600, or 0400 for a read-only disk. A disk without its file fails with `vmm.ErrDiskMissing`, and the driver never makes a blank file. It links each file into the jail root as `disk-<disk id>`, and adds it with `PUT /drives` before the network interface. |
| First load of a saved state | It makes the reflink copy `state/disk-<disk id>.raw` of each read-write disk, beside `state/disk.raw`. A read-only disk needs none. |
| Later load | It puts each copy back over its disk, so every disk matches the memory again. A disk whose file is gone fails with `vmm.ErrDiskMissing`, and the copy never makes the file again. The rule is A second load of one state of the phase 3 plan. |
| `DiscardMemory` | It puts each copy back too, with the same check of the file, logs `put the extra disks of the save back before the discard of its memory file` with `disks`, and drops the saved state. |
| A new save | It drops the copies of the older state. |
| `ReleaseDisk` | It removes the link `disk-<disk id>` from the jail root of each machine whose unit does not run. The disk keeper calls it before it removes a file, so the removal frees the blocks. A unit that runs fails the call, and keeps its link. |

A snapshot of Firecracker names each drive with its path in the jail. Attach and detach need a machine without a memory file ([Extra disks](/reference/disks/index.html.md#rules-of-a-disk)), so a load always finds the drives of its save.

## Repair of the root file system

ext4 sets an error state in its superblock at an error, and only `e2fsck` clears it. Linux skips orphan recovery on a file system with that state, in 6.1 and in 7.0. An orphan is a file that a process held open after its delete. After a crash that leaves an orphan, each later shutdown fails to make the root file system read-only, with `Orphan file not empty on read-only fs`. The journal flag `needs_recovery` then stays set, so each clean stop counts as `crash` ([Exit reasons](#exit-reasons)), and the orphans keep their space. So the driver repairs such a file system before a boot, as Q235 of the phase 3 plan decided.

| Item | Value |
| --- | --- |
| When | Only in `Boot`, after the refusal of a unit that runs and after the disk exists, and before the jail. Never in `Restore`, `Save`, `Snapshot`, `Attach`, or a load. |
| What | Only the root disk `disk.raw`. Never an extra disk, on which hopbox puts no file system, a copy in `state/` or `save.partial/`, a save dir, a snapshot, or a checkpoint. |
| Trigger | The bit `0x2` of `s_state` at offset `0x3A` of the ext4 superblock, which starts at byte 1024 of the disk. A disk without ext4 gets no repair. |
| Skip | While `state/` of the machine holds a file, the driver logs a warning and skips the repair. A load of that state needs the disk of its save. |
| Program | `e2fsck -p` of `e2fsprogs` 1.47.0 or later, at `E2fsck` of the config, `/usr/sbin/e2fsck` by default. The ext4 feature `orphan_file` of the default image needs 1.47.0. Debian 13 has 1.47.2. The preflight check `e2fsck` tests it ([the preflight check](/reference/preflight/index.html.md#checks)). |
| Work file | A reflink copy of `disk.raw` at `.../root/rootfs.repair` in the jail of the machine, for the jail user with mode 0600. |
| After exit status 0, 1, or 2 | The driver opens the work file with `O_NOFOLLOW`, and refuses a file that is not a regular file of the jail user with one link, as for the files of a save ([Save](#save)), and a file whose size differs from that of `disk.raw`. It syncs the file, renames it over `disk.raw`, and syncs the machine dir. |
| After another status, a timeout, or a kill | The driver removes the work file and logs an error, and the boot goes on with the old disk. |
| No `e2fsck` or `setpriv`, an `e2fsck` older than 1.47.0, or exit status 8 | A warning, and the boot goes on with the old disk. `orphan_file` is a compatible feature, so an `e2fsck` older than 1.47.0 may run on such a file system and exit with status 4, not 8. So the driver reads the version from `e2fsck -V` before the first repair after its start, with the parse of the preflight check, and runs no older `e2fsck`. It keeps the first version that it reads, so an upgrade of `e2fsprogs` needs a restart of `hopboxd`. An `e2fsck -V` that fails or times out without a version counts `unavailable` for that boot, and the next boot reads the version again. |
| A work file that an earlier boot left | The next boot removes it, and stops a unit of a repair that still runs. |

`e2fsck` parses a disk that the guest controls, so it never runs as root. It runs in the transient unit `hopbox-machine-<id>-fsck.service`, in the slice of the machines. systemd cannot run a process as a jail ID, because the ID has no entry in `/etc/passwd`. So the unit runs `/usr/bin/setpriv --reuid=<jail ID> --regid=<jail ID> --clear-groups --no-new-privs -- <E2fsck> -p <work file>`, and `e2fsck` runs as the jail user of the machine, with no capability.

| Property | Value |
| --- | --- |
| `Type` | `exec` |
| `RemainAfterExit` | `yes`, so the unit keeps the exit status after a status 0 until the driver stops it |
| `RuntimeMaxSec` | `RepairTimeout` of the config, 3 min by default. Then systemd stops the unit, and the result is `timeout`. |
| `TimeoutStopSec` | 10 s |
| `MemoryMax`, `MemorySwapMax` | 1 GiB and 0 |
| `CPUQuota` | 100%, one CPU |
| `TasksMax` | 16 |
| `NoNewPrivileges` | `yes` |
| `CapabilityBoundingSet` | `CAP_SETUID CAP_SETGID`, only for `setpriv`. The change of the user drops them, so `e2fsck` has none. |
| `PrivateNetwork`, `RestrictAddressFamilies` | `yes` and `none`: no network and no socket, also no unix socket |
| `PrivateDevices` | `yes` |
| `ProtectSystem`, `ProtectHome` | `strict` and `yes`: the file system of the host is read-only, and the home dirs are hidden |
| `TemporaryFileSystem` | `<data_dir>/jail:ro`, an empty tmpfs that hides every jail. `/tmp` and `/var/tmp` with `mode=1777,nosuid,noexec`, in place of `PrivateTmp`, whose dirs are on the disk of the host. The pages of a tmpfs count in `MemoryMax`. |
| `RestrictNamespaces` | `yes`: no namespace of any type |
| `SystemCallFilter` | `@system-service`. It holds `@setuid`, with the `setresuid`, `setresgid`, and `setgroups` of `setpriv`, and `@process`, with its `prctl`. `@system-service` lists the `capset` of `setpriv` directly, not in `@setuid` or `@process`. |
| `ProtectKernelTunables`, `ProtectKernelModules`, `ProtectKernelLogs`, `ProtectClock` | `yes` |
| `ProtectProc`, `ProcSubset` | `invisible` and `pid` |
| `SystemCallArchitectures` | `native`: the unit makes only the system calls of the ABI of the host, not those of another ABI such as 32-bit x86. |
| `LockPersonality`, `MemoryDenyWriteExecute`, `RestrictSUIDSGID` | `yes` |
| `LimitFSIZE` | the size of `disk.raw`. `e2fsck` writes in place, so no file of the unit grows past the disk. |
| `LimitCORE` | 0: no core dump |
| `BindPaths` | the work file, at its own path. `<data_dir>/jail` and `<data_dir>/machines` have mode 0700 for root, so the jail user reaches the work file only through this tmpfs. |
| `RequiresMountsFor` | `<data_dir>/jail` |

The driver asks systemd every 250 ms whether `e2fsck` ended, at most for the repair timeout plus 40 s. Then it stops the unit, checks that the unit is inactive, and resets its failed state. A unit that does not stop may still run `e2fsck` on the work file. So the repair then counts as `failed`, also after an exit status 0, and the driver removes the work file and keeps the old disk.

A repair that fails, times out, or is killed keeps the error state on the disk. So it runs again at each later boot of the machine, with no skip, until one repairs. Each such run costs the boot up to `RepairTimeout` plus 10 s for the stop, 3 min 10 s by default, one CPU, and at most 1 GiB of RAM in the slice of the machines. The work file is a reflink copy, so it takes space only for the blocks that `e2fsck` writes, until the driver removes it. Each run counts once more in `hopbox_root_repairs_total`, so a result other than `repaired` that rises at each boot names a disk that `e2fsck` cannot repair. The repair counts in the metric `hopbox_root_repairs_total` ([Metrics](/reference/metrics/index.html.md#metrics-of-the-root-repairs)). The phase `unit_start` of the boot holds the time of the repair.

On 2026-11-12, `e2fsck -p` 1.47.2 repaired a copy of such a disk with exit status 1, and gave its 100 MB back.

## The image cache

`<data_dir>/images/<hex>` holds the image with the digest `sha256:<hex>` ([Server layout](/reference/server-layout/index.html.md#the-image-cache)). The driver takes the disk of an image in this order:

1. When the dir has `artifact.json`, it must hold the image of the digest. Its disk must have the sha256 and the size that the control role sends, and it must name the kernel of the spec.
2. When the disk is there, the driver checks its size and its sha256 at the first use in each process of `hopboxd`. For the default image of 3.5 GiB, this takes about 9 s on the [tested server](/reference/preflight/index.html.md#the-tested-server).
3. An install leaves only the compressed copy. In that case, the driver checks the copy and unpacks it with zstd. Then it checks the disk and removes the copy. The unpack of the default image takes about 22 s on the tested server. The disk keeps a hole for each block of 64 KiB of zeros.
4. If the dir has no disk and the image has a URL, the driver downloads the compressed disk from the URL. It unpacks it and checks the disk against the sha256 of the image. Only an `https` URL works. If the download gets larger than the disk plus an eighth, the driver stops it. The dir then holds only `disk.raw`. A later install of a release with the same image adds its `artifact.json`.
5. Otherwise the boot fails with `firecracker: the image cache has no image <digest>, and the image has no URL`.

One job of the process prepares each image. It runs apart from the boot that started it, with a deadline of 2 hours. So a download can take longer than the timeout of one boot. The next boot of the machine then waits for the same job. Two boots of one image at the same time wait for one job. A boot whose wait ends first fails with `firecracker: the image <digest> is not ready yet, and the node goes on with it`, and the restart policy boots the machine again later.

Step 3 counts as a fetch with the source `bundle`, and step 4 as one with the source `url`, in `hopbox_image_fetches_total` ([Metrics](/reference/metrics/index.html.md#metrics-of-the-image-cache)). The checks of steps 1 and 2 are no fetch.

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

## Pause and resume

`Pause` sends `PATCH /vm` with `{"state": "Paused"}`, and `Resume` sends it with `{"state": "Resumed"}`. The vCPUs stop, and the memory stays in the RAM of the node. The process of Firecracker goes on, and so does its API. The unit keeps its memory limit, so the RAM of the guest stays in the cgroup of the unit. Phase 0 measured about 1 ms for each call on ks.

`PUT /mmds` works on a paused microVM. So the executor gives the microVM its new link key before the resume, and the guest never runs without it.

Firecracker can apply a call whose answer gets lost. So after a failed `Pause` or `Resume`, the driver reads the state with `GET /`, and `Instance.Paused` gives that state. A state that the driver cannot read counts as paused. A resume of a running microVM succeeds, but a guest link to a paused guest never comes up. The error of the call then wraps `vmm.ErrStateUnknown`, and the executor pauses again before it reports `standby` ([the node executor](/reference/executor/index.html.md#standby)).

## Hibernate and wake

A hibernate saves the RAM of the guest to the memory file of the machine, and ends its microVM. A wake loads that file into a new microVM. Both run in the jail of the machine, with its jail user. The memory file holds the secrets of the machine in plain bytes. Only root and the jail user can read it, through the dirs of mode 0700 and the files of mode 0600. An operator puts `<data_dir>` on an encrypted block device for more ([Standby and hibernate](/guide/standby-and-hibernate/index.html.md#the-memory-file-on-the-node)).

### Save

`Snapshot` runs these steps:

1. It reads the state of the microVM with `GET /`. It waits for the writeback of the last save without `Sync` of the microVM ([The limit of the unit](#the-limit-of-the-unit)), so the guest never waits for it in a pause. The wait ends at the end of `ctx`, and then the save fails. A running microVM then pauses. A paused one, as in standby, stays paused.
2. It reads the guest RAM with `GET /machine-config`, and builds two limits from the size of the machine, as the unit of a boot does ([The unit of a machine](#the-unit-of-a-machine)). The limit of a boot is the RAM plus 256 MiB. The limit of a save is the limit of a boot plus the RAM. The driver never reads the limit of the cgroup, so a limit that a killed save left raised does not add up at the next save. If the call fails, the save fails.
3. It reads the snapshot format of Firecracker, and removes the files of an earlier failed save from the jail root.
4. It sets the `MemoryMax` of the unit to the limit of a save until the unit ends, through systemd. The page cache of the memory file that Firecracker writes counts in the unit. A guest that touched all its RAM leaves no room for it below the limit of a boot, and the kernel then ends Firecracker. On a failure, the driver logs `cannot give the unit room for the page cache of a save`, and the save goes on. The owner decided this rule in Q202. Only the page cache of the save uses the extra room, and the anon memory of the unit does not grow. So placement does not count the raise ([Placement](/reference/placement/index.html.md#what-a-node-uses)). The host reclaims that page cache once the write ends. The KVM test `TestKVMSaveRoom` measures it ([Tests](#tests)). For that test, the driver can also raise the limit by a set size instead of the RAM. It can also ask the kernel to reclaim the clean page cache of the unit before the save, through `memory.reclaim` of its cgroup, in steps of 64 MiB. A save runs no reclaim by default. The reclaim asks for no more than the clean page cache. The kernel answers `EAGAIN` to a step that frees less than it asks, and that ends the reclaim. The driver writes `memory.reclaim` with plain system calls, because on `EAGAIN` the Go `os.File` waits for the file to become writable, and a cgroup file never signals that. In CI, that wait held a save for more than 20 minutes. If the driver cannot read `memory.stat` of the unit, it logs `cannot read the memory of the unit before a save`, and the save goes on. If a step of the reclaim fails, the driver logs `the kernel reclaimed less than the clean page cache of the unit before a save`, and the save goes on.
5. It sends `PUT /snapshot/create` with `snapshot_type` `Full`, `snapshot_path` `/vmstate.new`, and `mem_file_path` `/memory.new`. Firecracker writes both files into the jail root, and syncs them to the disk before it answers. The call has no timeout of its own, because the write of a large RAM takes long. The end of `ctx` does not end the call at once either. Firecracker writes the files on its one VMM thread until the end, so a resume sent before its answer waits behind it, and the guest stays paused. So at the end of `ctx`, the driver logs `the context of a save ended during the create; waiting for Firecracker` and waits for the answer, up to a grace: 10 s plus 1 s for each whole 64 MiB of guest RAM. That is 26 s at 1 GiB and 138 s at 8 GiB. The slowest create in CI wrote about 150 MiB/s ([The pause in CI](#the-pause-in-ci)). An answer within the grace fails the save as usual. Without an answer, the save fails with `no answer of Firecracker to the create within <grace> after the end of the save, so the guest can stay paused`. Firecracker can then still write, so the driver removes the files of the save, but neither resumes the microVM nor sets the limit of a boot back: a resume waits behind the create, and a lower limit over the dirty page cache of the write can make the kernel end Firecracker. It logs `the microVM stays paused, because Firecracker did not answer the create of its save`. The microVM stays paused, and `Instance.Paused` says so, until a later call. The executor frees the slot of the save and its reserved bytes then ([the node executor](/reference/executor/index.html.md#the-gate-of-a-save)), while Firecracker can still write up to the RAM of the guest. It writes into the removed files, or into a `memory.new` that appears after the removal, and the next save of the machine removes that file. After a resume to `running`, the `MemoryMax` of the unit stays at the limit of a save until a later save of the machine or the next `Attach`. The end of `ctx` comes from a loss of the lease, or from a stop of `hopboxd`. A loss of the lease ends the save within the grace. A stop of `hopboxd` ends at the stop timeout of 10 s of the unit `machines` ([hopboxd](/reference/hopboxd/index.html.md#run)), which a grace can outlast. Then the stop of `machines` fails with `context deadline exceeded`, and `hopboxd` exits with an error while the guest stays paused. The next `Attach` handles the microVM as after a kill of `hopboxd` ([Save without a stop](#save-without-a-stop)). A Firecracker that ends closes the socket, which ends the call.
6. It reads the CPU microcode of the node. On a failure, it logs a warning and saves an empty value.
7. It reads the clock source of the host, `/sys/devices/system/clocksource/clocksource0/current_clocksource`. See [The guest clock after a load](#the-guest-clock-after-a-load).
8. It takes both files from the jail. The jail user can still write the jail root, so the driver opens each file with `O_NOFOLLOW`, and works only through that descriptor. It refuses a symbolic link and any other file that is not a regular file. It also refuses a file of another owner than the jail user, and a file with more than one link. Through the descriptor, it gives the file to root with mode 0600.
9. It moves both files into `state/`. It removes the memory file and the disk copy of an older saved state first. Then it moves `vmstate`, writes `state.json`, and moves `memory` last. After each rename, it checks that the new name holds the file that it took in step 8. If the name holds another file, the driver removes that name, and the save fails. It syncs the dir after each step. So the name `memory` always stands for one whole saved state or for none, also after a kill of `hopboxd`.
10. It ends the unit with `SIGKILL`, and waits at most 10 s for the end of Firecracker. The exit reason is `snapshot` ([the VMM driver interface](/reference/vmm/index.html.md#instance)).

A failure in steps 2 to 9 before `memory` has its name removes the files of the save from the jail root. It also sets the `MemoryMax` of the unit to the limit of a boot, from the size of the machine, and not to the value before the save. It resumes a microVM that ran before step 1, and a paused microVM stays paused. A create with no answer within its grace keeps the raised limit and the pause instead (step 5). A memory file of an earlier save stays, unless the failure came during step 9, after the move began. The error starts with `firecracker: save machine <id>:`.

Once `memory` has its name, the file is the newest state of the machine, so the save never resumes the microVM. The microVM stays paused until its unit ends.

| Failure after `memory` has its name | What the save does |
| --- | --- |
| The last sync of `state/` fails | It logs `the memory file of a save is in place, but the last sync of its dir failed`, and goes on with step 10. |
| The stop of the unit fails | It returns the `MemoryState` of the file, with the error `firecracker: end machine <id> after its save: vmm: the memory file of the save is whole: <cause>`. |
| Firecracker does not end within 10 s | It returns the `MemoryState` of the file, with the error `firecracker: machine <id> still runs after its save: vmm: the memory file of the save is whole`. |

Both errors wrap `vmm.ErrSaved` ([the VMM driver interface](/reference/vmm/index.html.md#snapshotter-and-restorer)). The executor then keeps the memory file, and takes the later end of the microVM for no crash ([the node executor](/reference/executor/index.html.md#saved-state)).

A microVM whose save is whole saves nothing again. A new `Snapshot` of it only ends its unit, as in step 10, and returns the `MemoryState` that `state.json` records. It logs `ended the microVM of a saved machine` with `saved_at`. A `Stop` of it sends no shutdown to the guest, and kills the unit at once, with the exit reason `snapshot`. A `Resume` of it lets the guest run past its save, so the driver logs `resumed a microVM after its save, so its memory file is older than the guest` as a warning. The exit reason is then no longer `snapshot`, and the executor drops the memory file once the guest agent links.

A `hopboxd` can end after step 9 and before the unit ends. The next `hopboxd` finds the unit running and the memory file in place. `Attach` then compares `saved_at` of `state.json` with the start of the main process of the unit, which systemd gives as `ExecMainStartTimestamp`. A load starts its unit after the save that it loads, so only a save of this unit is newer than the start. For such a save, the instance counts as saved, and `Attach` logs `attached to a microVM whose save is whole` with `unit` and `saved_at`. So its end has the reason `snapshot`, and the executor reports `hibernated`, not `stopped` or `failed`.

### The record of a saved state

`state.json` describes the saved state. The driver writes it through a temp file, a sync, and a rename, so a crash leaves the old content or the new one.

| Field | Type | Description |
| --- | --- | --- |
| `vmm` | string | The driver that saved the state: `firecracker`. |
| `vmm_version` | string | The version of Firecracker, such as `v1.17.0`. |
| `snapshot_format` | string | The snapshot format version of Firecracker at the save, such as `v12.0.0` ([Capabilities](#capabilities)). |
| `bytes` | integer | The bytes of the disk that the memory file and the `vmstate` file hold at the save ([The bytes of a saved state](#the-bytes-of-a-saved-state)). |
| `saved_at` | string | The time of the save, in RFC 3339, UTC. |
| `loads` | integer | The loads of this saved state so far, also those that failed. A new save starts at 0. |
| `microcode` | string | The CPU microcode of the node at the save, or absent. The compatibility class does not hold it. It explains a load that fails after an update of the microcode. |
| `clock_realtime` | bool | Whether the clock source of the host at the save was `tsc`. |

Example:

```json
{"vmm":"firecracker","vmm_version":"v1.17.0","snapshot_format":"v12.0.0","bytes":1073741824,"saved_at":"2026-11-08T10:12:03.52Z","loads":0,"microcode":"0xb4","clock_realtime":true}
```

`Inspect` reports `Memory` true while `state/memory` exists, with the fields of `state.json` in `MemoryState` ([the VMM driver interface](/reference/vmm/index.html.md#driver)). The bytes come from `state/memory` and `state/vmstate` themselves, not from `state.json`. The driver can lack a good `state.json`: the file is missing or damaged, or it names no snapshot format. Then `Inspect` still reports the memory file, with its bytes and its time. The driver logs `the memory file of a machine has no readable state file`.

### The bytes of a saved state

The driver counts the disk that a saved state holds with `savedstate.Bytes` of `internal/node/savedstate`. It counts each file of the state, `memory` and `vmstate`, and adds the two:

1. It reads the extents of the file with the `FS_IOC_FIEMAP` call of Linux. Each extent is a range of the disk that the file holds. The count is the sum of their lengths, so a hole of a sparse file counts nothing. An extent whose place the kernel does not report yet, such as a delayed write, counts whole.
2. A file system without `FIEMAP`, such as tmpfs, gives the blocks of the file instead: `st_blocks` times 512.
3. A file that the driver cannot read in this way counts its blocks too.

At a save, the driver counts the two files in the jail root, after Firecracker wrote them. `Inspect` counts them in `state/`. If neither file opens, it counts the blocks of `state/memory`.

The package can also leave out the extents that a base file holds, so a reflink copy counts only its new blocks. A save of a hibernate has no base, and no caller passes one yet. Example: a machine has 8 GiB of RAM, and its guest touched 1 GiB. Its memory file has a size of 8 GiB, but holds about 1 GiB of disk. So its `bytes` are about 1 GiB, plus the small `vmstate` file.

### Load

`Restore` runs these steps:

1. It checks the size of the machine as a boot does. It refuses a machine whose unit runs, and resets the failed state of an old unit.
2. It refuses a machine without `state/memory` or `state/vmstate`, with `firecracker: machine <id> has no saved state to load: no <file>`.
3. It refuses a saved state of another snapshot format than this Firecracker, with `firecracker: the saved state of machine <id> has the snapshot format <format>, and this Firecracker loads <format>`.
4. It adds 1 to `loads` in `state.json`, before the load.
5. At the first load of the saved state, it makes `state/disk.raw`, a reflink copy of the disk. At each later load, it puts that copy back over the disk, so the disk matches the memory again. A saved state gets its copy only after its count, and a load runs only after the copy. So a saved state without a copy never ran.
6. It fills the jail with the disk. A load needs no kernel. Then it puts reflink copies of `memory` and `vmstate` into the jail root. It opens each copy with `O_NOFOLLOW`, and refuses a file that is not a regular file. Through that descriptor, it gives the copy to the jail user with mode 0600. So a link in the place of a copy changes no other file.
7. It makes and connects the tap device, and starts a new unit in the same jail, with the same `MemoryMax` as a boot.
8. It waits for the API socket, and sends these calls:

| API call | Body |
| --- | --- |
| `PUT /snapshot/load` | `snapshot_path` `/vmstate`. `mem_backend` with `backend_type` `File` and `backend_path` `/memory`. `track_dirty_pages` false, and `resume_vm` false. `network_overrides` with the tap device of the machine for `eth0`. `clock_realtime` as `state.json` says. |
| `PUT /mmds` | `{"hopbox": {"guest_link_key": "<64 hex characters>"}}`, the new link key of the wake |
| `PATCH /vm` | `{"state": "Resumed"}` |

The `File` backend loads the memory lazily: Firecracker maps the memory copy, and the guest reads each page from it at its first use. So a load reads no memory, and damaged bytes in the file do not fail it. The pages of the file sit in the page cache of the unit. Its `MemoryMax` is the RAM plus 256 MiB, and the kernel reclaims the clean pages. The guest gets its new link key before it runs.

`Restore` keeps every file of `state/`. A failure after the start of the unit stops the unit and removes the tap device. The error starts with `firecracker: load the saved state of machine <id>:`. The executor drops the files only after the guest agent of the wake linked ([the node executor](/reference/executor/index.html.md#saved-state)).

### The guest clock after a load

With `clock_realtime` true, Firecracker sets the wall clock of the guest at the load. KVM puts the wall clock into the saved state only on a host with the clock source `tsc`. Firecracker refuses `clock_realtime` for a state without it. So a save on another clock source records `clock_realtime` false, and logs `the clock source of the host is not tsc, so a load of this save keeps the guest clock of the save`. A load of such a state keeps the guest clock of the save until the guest agent links. The guest agent then sets the clock from the node ([Guest link](/reference/guest-link/index.html.md#clock-and-randomness)). The owner decided this rule on 2026-11-08, as Q201. ks has the clock source `tsc`, so a save on ks records `clock_realtime` true. The CI runner has another clock source, so a save there records `clock_realtime` false.

### Drop of the memory file

`DropMemory` removes `state/memory` first, then the rest of `state/` with the disk copy, then the copies in the jail root, and syncs the machine dir. A microVM that maps its copy runs on, because the blocks stay until Firecracker ends. A machine without a memory file succeeds. A drop is an unlink, so the bytes can stay on the disk of the node until it reuses the blocks.

`DiscardMemory` serves the call `DiscardMemory` ([the machine API](/reference/machine-api/index.html.md#discardmemory)). It refuses a machine whose unit runs, with `firecracker: machine <id> runs, so its memory file stays`. When `state/disk.raw` exists, it puts that copy back over the disk as a reflink, syncs the machine dir, and logs `put the disk of the save back before the discard of its memory file`. So the next boot starts from the disk of the save, not from a disk that a failed wake changed. A saved state without a disk copy never ran, so its disk is the disk of the save already. Then `DiscardMemory` drops the saved state as `DropMemory` does. A failed copy fails with `firecracker: put the disk of the save of machine <id> back: <cause>`, and keeps the saved state.

## Save without a stop

`Save` saves the memory, the state, and the disk of a machine that goes on running. A fork, a snapshot, and a checkpoint start from such a save. The caller names a new dir on the file system of the machine, and gets the files `memory`, `vmstate`, and `disk.raw` in it ([the VMM driver interface](/reference/vmm/index.html.md#saver)). Milestone T10 builds the call. No executor action calls it yet.

`Save` runs these steps:

1. It refuses a microVM whose save of a hibernate is whole. It reads the state of the microVM with `GET /`, and refuses a microVM that neither runs nor is paused. It reads the guest RAM with `GET /machine-config` for the limits of the unit, and the snapshot format of Firecracker.
2. It refuses a dir that is not a clean absolute path, a dir that exists, and a dir whose parent is not on the file system of the machine dir. A reflink copy and a rename reach only that file system.
3. It removes `save.partial/` of the machine dir, and makes it again, with mode 0700, for root.
4. It waits for the writeback of the last save without `Sync` of the microVM ([The limit of the unit](#the-limit-of-the-unit)). The wait ends at the end of `ctx`, and then the save fails.
5. It removes the files of an earlier failed save from the jail root, and sets the `MemoryMax` of the unit to the limit of a save, as a hibernate does ([Save](#save)).
6. A running microVM pauses. A paused one, as in standby, stays paused.
7. It sends `PUT /snapshot/create` with `snapshot_type` `Full`, or `Diff` for `SaveOptions.Diff`, `snapshot_path` `/vmstate.new`, `mem_file_path` `/memory.new`, and `sync_snapshot_files` false. So Firecracker writes the files into the page cache and answers without a sync. It flushes the disk file of the machine during the call in any case. As for a hibernate, the end of `ctx` does not end the call at once. The driver waits for the answer up to the same grace, and an ended `ctx` then fails the save ([Save](#save)).
8. It makes `save.partial/disk.raw`, a reflink copy of `disk.raw` of the machine, with no sync. The guest is still paused, so the disk copy and the memory match one moment.
9. It takes `memory.new` and `vmstate.new` from the jail, with the checks of a hibernate: `O_NOFOLLOW`, a regular file, the jail user as the owner, one link, and then root and mode 0600 through the descriptor.
10. A microVM that ran before step 6 resumes. The pause thus holds steps 7 to 9.
11. It moves both files into `save.partial/`, as `memory` and `vmstate`, and checks after each rename that the name holds the file that it took.
12. It reads the CPU microcode and the clock source of the host, as a hibernate does.
13. With `SaveOptions.Sync`, it syncs the memory file, the state file, the disk copy, and `save.partial/`. The guest runs during this sync.
14. It renames `save.partial/` to the dir of the caller. With `Sync`, it then syncs the parent of that dir. If that sync fails, it removes the dir, and the save fails, because a crash of the host can then lose the name of the dir. That removal gets no sync either, so after a crash of the host the dir can come back, and the caller must not take it for a save that succeeded.
15. It sets the `MemoryMax` of the unit back to the limit of a boot ([The limit of the unit](#the-limit-of-the-unit)).

The sync runs after the resume, not in the pause. The owner decided this on 2026-11-11, as Q218. Phase 0 measured a pause of about 1.8 s without the sync in place of 7.8 s with it, at 8 GiB. A crash of the host before step 14 leaves no dir, as a failed save does. So `Sync` says whether `Save` waits for the disk, not how long the guest pauses. A save without `Sync` gets its name before its files reach the disk. So after a crash of the host, its dir can look whole and still hold damaged files, and such a dir must not load. T12 and T13 add this rule to their loads.

A failure in steps 5 to 14 removes the files of the save from the jail root and `save.partial/`. Then it sets the `MemoryMax` of the unit to the limit of a boot, and resumes a microVM that ran before step 6. A create that gets no answer within the grace is the exception, as for a hibernate: the limit stays raised, and the microVM stays paused. A paused microVM stays paused. The error starts with `firecracker: save machine <id> without a stop:`. The dir of the caller never exists after a failure. A save without a stop never touches `state/`, so the memory file of a hibernate and an earlier dir of a save stay.

A `hopboxd` can end in the middle of a save. The next `hopboxd` then finds the microVM running, or paused if the end came in the pause. Such a microVM does not count as saved, because `state/` holds no newer save. So the executor resumes it at a target of `running`, as for any paused microVM ([the node executor](/reference/executor/index.html.md#standby)). The dir of the caller does not exist. `Attach` removes `save.partial/` and the files of the save in the jail root. If the `MemoryMax` of the unit is the limit of a save or `max`, `Attach` then sets the limit of a boot back, as after a save without `Sync`. Any other limit stays. It can come from a unit that a boot under another `MemoryOverhead` started, and a lower limit can make the kernel end Firecracker. A goroutine first syncs the file system of the machine dir with `syncfs`, because the files of a whole save can still hold dirty page cache in the unit. The next save and the next hibernate wait for that goroutine. The same steps cover a hibernate that a kill of `hopboxd` cut before its memory file had its name.

The save resets the vsock connections of the guest, so the guest link ends. The guest agent links again with the same link key, because the key in the metadata service stays ([Guest link](/reference/guest-link/index.html.md#transport)). A stream that runs at the save ends with an error.

### The limit of the unit

The page cache of the files that Firecracker writes counts in the unit, as for a hibernate. So the save raises the `MemoryMax` of the unit by the RAM of the guest. With `Sync`, the files are clean once `Save` syncs them, so `Save` sets the limit of a boot back before it returns. Without `Sync`, the page cache stays dirty for a while. A limit below the memory of the unit makes the kernel reclaim at once, and a dirty page needs its write first, so the kernel can end Firecracker. So a goroutine syncs the memory file and the state file after `Save` returns, and only then sets the limit of a boot. It sets it only while the unit has the main process of the save. A stop, a resize, and a start of the machine before the end of the sync give it a new unit with a limit of its own, which stays. The next save and the next hibernate of the machine wait for that goroutine. The driver keeps the wait by machine, not by instance, so a new `Attach` to the same microVM in the same `hopboxd` keeps it, and leaves the limit to the goroutine. After a restart of `hopboxd` before its end, `Attach` sets the limit back ([Save without a stop](#save-without-a-stop)).

A `FICLONE` on XFS writes the dirty pages of its source back first. So a reflink copy of a memory file of a save without `Sync`, as for the jail of a fork, pays that write at the copy.

### The pause in CI

The KVM test `TestKVMSaveWithoutAStop` measured the pause of a guest of 1 GiB under its real `MemoryMax` ([Tests](#tests)). A loop in the guest wrote 64 MiB at a time to its disk with fsync, and a tmpfs held 256 MiB of random data. Two runs on the shared CI runner saved three times in each way:

| Way | Pauses |
| --- | --- |
| Sync in the pause | 2.58 s, 1.96 s, 3.26 s, then 2.86 s, 5.75 s, 1.21 s |
| Sync after the resume | 4.81 s, 1.11 s, 1.00 s, then 0.96 s, 3.00 s, 3.03 s |
| No sync | 5.08 s, 6.83 s, 1.60 s, then 2.68 s, 0.78 s, 2.82 s |

So the pause took 0.78 s to 6.83 s. Phase 0 scaled to 1 GiB gives about 0.27 s without a sync and 1.0 s with it. The pause in CI was thus 3 to 25 times the phase 0 figure without a sync, and the way of the sync did not order the pauses. `PUT /snapshot/create` held the whole pause each time. The disk copy and the take of the files cost 15 to 40 ms. Phase 0 measured an idle guest with no `MemoryMax` on a quiet disk. The test does not show the cause of the gap. A busy disk that Firecracker flushes in the create can explain it, and so can the limits of the unit on its dirty pages. The fsync default of a save does not change for now. T16 measures the pause on ks, and the owner decides the default then. The owner decided this on 2026-11-11, as Q234.

## Stop

`Instance.Stop` first reads the state of the microVM with `GET /`. `SendCtrlAltDel` needs a running guest, so a paused guest resumes first. If a call fails, the driver kills the microVM. Then `Instance.Stop` sends `SendCtrlAltDel` through the API. The guest reboots, and with `reboot=k` a reboot ends Firecracker. The stop waits for the end of the process. After 30 s, or at the end of the context of the call, the driver kills the unit with `SIGKILL`. On the tested server, a stop of the default image takes about 0.4 s.

## A stop through systemd

A reboot or a shutdown of the server stops each unit through systemd. So does `sudo systemctl stop hopbox-machine-<id>.service`. Without `ExecStop`, systemd sends `SIGTERM`, which ends Firecracker at once, as a power loss does. So the unit of each machine gets an `ExecStop`, and a stop runs these steps:

1. At a shutdown, systemd stops `hopboxd.service` first, because each unit is ordered before it. So no executor sees its guests go down, and none applies the restart policy or reports a state.
2. systemd runs `hopboxd stop-guest`. It resumes a paused guest, and sends `SendCtrlAltDel` through the API socket, as `Instance.Stop` does.
3. The guest shuts down. It writes its page cache to the disk, and unmounts its file systems. Then it reboots, and with `reboot=k` Firecracker ends with status 0.
4. `stop-guest` sees the end of the process, and exits with status 0. The unit is then inactive.

If the guest does not end within `StopTimeout`, 30 s by default, `stop-guest` exits with status 1. If the call to Firecracker fails, and Firecracker does not end within 2 s, `stop-guest` exits with status 1 too. systemd ignores the status, and sends `SIGTERM` to Firecracker. Firecracker ends at once, as at a power loss. The unit then ends inactive, not failed, because `SIGTERM` is a clean end for systemd. The root file system of the guest is still in use, so the driver reports `crash`.

systemd runs `ExecStop` also after Firecracker ended by itself, as after a reboot from inside. It then leaves `MAINPID` unset, and `stop-guest` exits with status 0 at once.

The next boot of the server brings each machine back. The target of each machine stays as it was, so the executor boots each machine with the target `running`, whatever its restart policy ([the node executor](/reference/executor/index.html.md#restart-policy)).

A stop of one unit while `hopboxd` runs is a stop that `hopboxd` did not ask for. When the guest shuts down cleanly, the exit reason is `shutdown`, as for a shutdown from inside. When systemd had to kill Firecracker, the exit reason is `crash`. The executor then applies the restart policy. With `always`, the machine boots again after the backoff. With `never`, a clean stop sets the target to `stopped`. To stop a machine, use the API or `hopbox machine stop` instead.

A unit keeps the properties of the boot that started it. So a microVM that a release without `ExecStop` booted still gets `SIGTERM` at a shutdown, until its next boot. After a rollback to a release without `stop-guest`, the units of the later release run the older `hopboxd`. It exits with status 80, `unexpected argument stop-guest`. systemd ignores that status. A shutdown or a reboot from inside thus still counts as `shutdown`. A stop through systemd sends `SIGTERM` at once, as before `stop-guest`, until the next boot of the machine. Each later release keeps the flags of `stop-guest`. A unit runs the `hopboxd` of the current release, with the flags of the release that booted it.

## Exit reasons

The driver watches the main process of the unit with a pidfd, a file descriptor of a process. When the process ends, it asks systemd why. A stop that starts during that wait still counts as a stop of the driver. A poll of the pidfd that fails with `EINTR` runs again at once. A poll that fails with `ENOMEM` runs again after 10 ms, then after twice as long each time, up to 1 s. Any other error of the poll ends the watch, and systemd then says why the unit ended.

| What ended the process | Unit after the end | Root file system | Exit reason |
| --- | --- | --- | --- |
| A stop of the driver | any | any | `stopped` |
| A shutdown or a reboot in the guest, or a stop through systemd that the guest finished in time | inactive | clean | `shutdown` |
| A kernel panic or a reset in the guest | inactive | in use | `crash`, with the error `the guest ended without a clean unmount of its root file system, as after a kernel panic or a reset` |
| A crash of Firecracker or a signal | failed | any | `crash`, with an error such as `the Firecracker process ended: unit failed/failed, result signal, killed by SIGKILL` |
| A stop through systemd that the guest did not finish in time | inactive | in use | `crash`, with the error of a kernel panic above |

A kernel panic and a reboot both end Firecracker with status 0, because of `panic=1` and `reboot=k`. So the driver reads the ext4 superblock of the disk of the machine after the end. A shutdown or a reboot makes the root file system read-only first, which clears the journal flag `needs_recovery`. After a panic or a reset, the flag stays set. The driver reads only this flag. The error state of ext4 tells nothing about the last boot, because it stays across clean unmounts until `e2fsck` clears it. With the error state, Linux skips orphan recovery, so after a crash that leaves an orphan, a shutdown cannot make the root file system read-only, and its end counts as `crash`. The next boot repairs such a file system first ([Repair of the root file system](#repair-of-the-root-file-system)). A disk without ext4 or without a journal tells nothing, so its end counts as `shutdown`, and the driver logs a warning. A panic before the guest mounts its root file system read-write also counts as `shutdown`, because the flag is not set yet.

A poweroff or a halt in the guest also ends Firecracker. Firecracker has no power-off device, so the default image turns both into a clean reboot at their last step ([Default image](/reference/default-image/index.html.md#files)).

The driver resets the failed state of the unit, so the name is free for the next boot.

## Attach after a restart or an upgrade

After a restart of `hopboxd`, the executor asks the driver for each machine of its node-local state ([the node executor](/reference/executor/index.html.md#node-local-state)). `Inspect` reports a machine as running while its unit is `active`, `activating`, `deactivating`, `reloading`, or `refreshing`. `Inspect` and `Attach` then ask the API of Firecracker for the state of the microVM. For a microVM that runs, `Attach` watches the main process of the unit again. The executor then gives the microVM a new link key through `PUT /mmds`, and the guest agent connects again. For a microVM in the state `Paused`, `Instance.Paused` returns true. The executor then leaves the guest paused while the target of the machine is `standby` ([the node executor](/reference/executor/index.html.md#standby)). For a microVM whose save is not whole, `Attach` also removes what a save that `hopboxd` left behind holds, and sets the limit of a boot back ([Save without a stop](#save-without-a-stop)).

A microVM in the state `Not started` comes from a boot that `hopboxd` left before `InstanceStart`. Its link key went with that `hopboxd`. So `Attach` kills its unit and returns `vmm.ErrNotRunning`, and the executor boots the machine again.

An upgrade changes the files of `vmm.firecracker` and `vmm.kernel`. A running microVM keeps its copy of Firecracker in its jail and the kernel that it booted with. The next boot of a machine uses the new Firecracker.

## Delete

`Delete` refuses a machine whose unit runs. It resets a failed unit, stops the unit `hopbox-machine-<id>-fsck.service` of a repair that an earlier boot left and resets its failed state ([Repair of the root file system](#repair-of-the-root-file-system)), removes the tap device, the jail dir, and the machine dir with the disk. The machine dir also holds the partial copy of a pull and the saved state in `state/`, which go with it. The jail ID of the machine is then free.

## Logs

Every record has `component=firecracker`.

| Log message | Level | Fields |
| --- | --- | --- |
| `booted a microVM` | info | `machine`, `unit`, `jail_id`, `tap`, `duration` |
| `attached to a microVM` | info | `machine`, `unit`, `state` |
| `attached to a microVM whose save is whole` | info | `machine`, `unit`, `saved_at` |
| `stopping a microVM that never started` | warn | `machine`, `unit` |
| `made the disk of a machine as a reflink copy of its image` | info | `machine`, `image` |
| `put the extra disks of the save back before the discard of its memory file` | info | `machine`, `disks` |
| `unpacked the disk of an image` | info | `digest`, `duration` |
| `downloaded the disk of an image` | info | `digest`, `url`, `duration` |
| `checked the disk of an image` | info | `digest`, `duration` |
| `cannot remove the compressed disk of an unpacked image` | warn | `digest`, `err` |
| `removed a temp file of the image cache` | info | `file` |
| `cannot remove a temp file of the image cache` | warn | `file`, `err` |
| `cannot ask the guest to shut down, so the microVM gets killed` | warn | `machine`, `err` |
| `the guest did not shut down in time, so the microVM gets killed` | warn | `machine`, `timeout` |
| `paused a microVM` | info | `machine`, `duration` |
| `resumed a microVM` | info | `machine`, `duration` |
| `resumed a microVM after its save, so its memory file is older than the guest` | warn | `machine` |
| `cannot read the state of a microVM after a failed call, so it counts as paused` | warn | `machine`, `err` |
| `saved a microVM to its memory file` | info | `machine`, `duration`, `save_duration` (the call `PUT /snapshot/create`), `bytes`, `snapshot_format`, `vmm_version`, `microcode`, `paused_before` |
| `cannot read the CPU microcode for a save` | warn | `machine`, `err` |
| `the clock source of the host is not tsc, so a load of this save keeps the guest clock of the save` | warn | `machine`, `clock_source` |
| `cannot resume a microVM after a failed save` | error | `machine`, `err` |
| `the context of a save ended during the create; waiting for Firecracker` | warn | `machine`, `grace` |
| `the microVM stays paused, because Firecracker did not answer the create of its save` | error | `machine`, `err` |
| `ended the microVM of a saved machine` | info | `machine`, `saved_at` |
| `cannot give the unit room for the page cache of a save` | warn | `machine`, `err` |
| `cannot read the memory of the unit before a save` | warn | `machine`, `err` |
| `the kernel reclaimed less than the clean page cache of the unit before a save` | info | `machine`, `left`, `err` |
| `the memory file of a save is in place, but the last sync of its dir failed` | warn | `machine`, `err` |
| `cannot set the MemoryMax of a unit back after a failed save` | warn | `machine`, `err` |
| `cannot remove a file of a save` | warn | `machine`, `file`, `err` |
| `saved a microVM without a stop` | info | `machine`, `dir`, `duration`, `paused` (the pause of the guest), `create_duration` (the pause until the end of `PUT /snapshot/create`), `sync_duration`, `sync`, `diff`, `bytes`, `snapshot_format`, `vmm_version`, `paused_before` |
| `cannot remove the dir of a failed save` | warn | `machine`, `dir`, `err` |
| `cannot set the MemoryMax of a unit back after a save` | warn | `machine`, `err` |
| `cannot write the files of a save to the disk, so the unit keeps the room of the save` | warn | `machine`, `err` |
| `cannot read the unit after a save, so it keeps the room of the save` | warn | `machine`, `err` |
| `cannot read the unit after the writeback of a save, so it keeps the room of the save` | warn | `machine`, `err` |
| `the main process of the unit changed during the writeback of a save, so its limit stays` | info | `machine`, `main_pid`, `now_main_pid` |
| `setting the MemoryMax of a unit back after a save that hopboxd left` | info | `machine`, `memory_max`, `boot_limit` |
| `cannot read the RAM of a microVM, so its MemoryMax stays as it is` | warn | `machine`, `err` |
| `cannot read the MemoryMax of a unit, so it stays as it is` | warn | `machine`, `err` |
| `loaded a microVM from its memory file` | info | `machine`, `unit`, `jail_id`, `tap`, `load` (the count of loads), `load_duration` (the call `PUT /snapshot/load`), `duration`, `snapshot_format`, `saved_at` |
| `dropped the memory file of a machine` | info | `machine` |
| `put the disk of the save back before the discard of its memory file` | info | `machine` |
| `the memory file of a machine has no readable state file` | warn | `machine`, `err` |
| `cannot stop the unit of a failed boot` | error | `machine`, `unit`, `err` |
| `the machine boots without network` | warn | `machine`, `address`, `network` |
| `cannot remove the tap device of a machine` | warn | `machine`, `err` |
| `cannot watch a microVM` | error | `machine`, `err` |
| `cannot reset a failed unit` | warn | `unit`, `err` |
| `cannot read the root file system of a stopped guest, so its end counts as a shutdown` | warn | `machine`, `err` |
| `the root file system of the guest is not ext4 with a journal, so its end counts as a shutdown` | warn | `machine` |
| `the root file system of a machine has ext4 errors, so e2fsck repairs a copy of its disk` | info | `machine`, `unit` |
| `repaired the root file system of a machine` | info | `machine`, `unit`, `status` (the exit status of `e2fsck`), `duration` |
| `e2fsck did not repair the root file system of a machine, so it boots with the old disk` | error | `machine`, `unit`, `result` (`failed`, `timeout`, or `killed`), `end` (how the unit ended, such as `unit failed/failed, result exit-code, exit status 4`) |
| `e2fsck cannot check the root file system of a machine, so it boots with the old disk` | warn | `machine`, `unit`, `end` |
| `cannot repair the root file system of a machine, so it boots with the old disk` | error | `machine`, `err`, and `unit` or `file` when the cleanup of an earlier repair failed |
| `cannot clean up what an earlier repair of the root file system of a machine left` | warn | `machine`, `unit` or `file`, `err`. The disk has no error state, so the boot needs no repair. |
| `the node lacks a program of the repair, so the root file system of a machine with ext4 errors boots unrepaired` | warn | `machine`, `program`, `err` |
| `the e2fsck of the node cannot repair the default image, so the root file system of a machine with ext4 errors boots unrepaired` | warn | `machine`, `program`, `err` (an `e2fsck` older than 1.47.0, or a version that `e2fsck -V` did not print) |
| `the root file system of a machine has ext4 errors, but its saved state holds a file, so the boot skips the repair` | warn | `machine`, `dir`, `err` |
| `cannot read the root file system of a machine before its boot, so the boot skips the repair` | warn | `machine`, `err` |
| `removed the work file of a repair that an earlier boot left` | info | `machine`, `file` |
| `cannot remove the work file of a repair` | warn | `machine`, `err` |
| `the disk of a machine is the repaired copy, but the sync of its dir failed` | warn | `machine`, `err` |
| `cannot stop the unit of a repair` | warn | `unit`, `err` |
| `cannot reset the failed state of the unit of a repair` | warn | `unit`, `err`. The unit has stopped, so the repair keeps its result. The next boot of the machine resets the unit again before a repair. |

`hopboxd stop-guest` logs to the journal of the unit of the machine, at level info ([hopboxd](/reference/hopboxd/index.html.md#stop-guest)). For a stop of a running guest, it logs `asking the guest to shut down` and then `the guest shut down`. On a failure, it prints the error and exits with status 1, for example `firecracker: the guest did not shut down within 30s`.

The driver serves the metrics of its image cache ([Metrics](/reference/metrics/index.html.md#metrics-of-the-image-cache)) and of the repairs of root file systems ([Metrics](/reference/metrics/index.html.md#metrics-of-the-root-repairs)). The executor counts each boot, stop, and delete ([the node executor](/reference/executor/index.html.md#metrics)). The driver times the phases `unit_start` and `vmm_ready` of each boot, and the executor serves them ([the VMM driver interface](/reference/vmm/index.html.md#driver)). The driver is a `vmm.UsageReader`: it reads what each microVM uses from the cgroup of its unit, and the node serves these values as the [metrics of the machines](/reference/metrics/index.html.md#metrics-of-the-machines). It reads `cpu.stat` and `memory.current`. When the cgroup has them, it also reads `cpu.pressure`, `memory.events`, and `io.stat`. From `io.stat`, it reads only the device of the file system of `MachinesDir`.

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
| `CgroupRoot` | `/sys/fs/cgroup` | The mount point of the cgroup v2 hierarchy, from which the driver reads what the machines use. |
| `E2fsck` | `/usr/sbin/e2fsck` | The absolute path of the `e2fsck` of [Repair of the root file system](#repair-of-the-root-file-system). |
| `RepairTimeout` | 3 min | The `RuntimeMaxSec` of the unit of a repair. `New` refuses a value below 0 or above 3 min, so the repair, its stop of 10 s, and the margin of 30 s stay inside the 5 min of a boot action ([the node executor](/reference/executor/index.html.md#actions)). |
| `Registerer` | none | Gets the metrics of the image cache and of the repairs. Nil registers none. |
| `Logger` | `slog.Default()` | The logger. |

| Method | Description |
| --- | --- |
| `AddKernel(ctx, path)` | Adds the kernel at `path` to the kernel cache, and returns its sha256. |
| `DiskPath(id)` | The path of `disk.raw` of a machine, for a cold move. |
| `Usage(id)` | The CPU time and the RAM of the unit of a running machine: `usage_usec` of `cpu.stat` and `memory.current` of its cgroup, such as `/sys/fs/cgroup/hopbox.slice/hopbox-machines.slice/hopbox-machine-<id>.service`. It fails for a machine without a cgroup. |
| `TotalUsage()` | The same values from the cgroup of the slice, for every machine together. |
| `DiskUsage(id)` | The bytes of the blocks of `disk.raw` of a machine: `st_blocks` times 512. |
| `SnapshotFormat(ctx)` | The snapshot format version of Firecracker. |
| `Close()` | Closes the connection to systemd. A later call makes a new one. `hopboxd run` calls it after the stop of the unit `machines`. |

The function `firecracker.StopGuest(ctx, sock, pid, timeout)` is the body of `hopboxd stop-guest`. It resumes a paused guest, and sends `SendCtrlAltDel` to the API socket `sock`. Then it waits for the end of the process `pid`, at most for `timeout`. A `pid` of 0 returns at once.

The methods of `vmm.Driver` follow [the VMM driver interface](/reference/vmm/index.html.md#driver), and so do `Snapshot`, `Restore`, `DropMemory`, and `DiscardMemory` ([the VMM driver interface](/reference/vmm/index.html.md#snapshotter-and-restorer)), and `Save` ([the VMM driver interface](/reference/vmm/index.html.md#saver)). The driver needs Linux. On another system, each of these methods fails with `firecracker: the driver runs only on Linux`: `Inspect`, `Boot`, `Attach`, `Pause`, `Resume`, `Snapshot`, `Save`, `Restore`, `DropMemory`, `DiscardMemory`, and `Delete`.

## Tests

`make test` runs the unit tests of the driver on any system: the config checks, the names, the API client against a fake Firecracker, the image cache, the kernel cache, and the jail IDs. They also cover the files of a saved state. The tests read and write `state.json`, and stop a save between two of its steps. They check the count of loads and the disk copy, and the refusal of another snapshot format. They check the API calls of a load, the drop, `Inspect` with a damaged `state.json`, and the clock source. A save must refuse a link and a file of another owner in the jail root. A swap of a name after its rename must fail the save. `DiscardMemory` must put the disk copy back. `make test-linux` runs them on Linux too, with a save that fails and puts the microVM back as it was. On Linux, the tests of `Save` check the API calls and the body of `PUT /snapshot/create` for a running and a paused microVM and for a diff. They check the files, their modes, and the removal of what a killed save left. A save fails at the create, at a create that writes no file, at the disk copy, at the resume, and after its files are whole. Each failure must give the microVM its state, leave no dir, and keep an earlier save. A save also refuses a dir that exists, a relative dir, and a microVM whose hibernate is whole. A save waits for the writeback of a save without `Sync` before it. The writeback sets the limit of a boot only while the unit has the main process of the save.

The unit tests of [Repair of the root file system](#repair-of-the-root-file-system) read the error state from a superblock, and check that a boot repairs only such a file system. While `state/` holds a file, a boot skips the repair and counts `skipped`, and a node without `e2fsck` or with an `e2fsck` older than 1.47.0 counts `unavailable`. The driver keeps the first version of `e2fsck` that it reads, and reads again after an `e2fsck -V` without a version. Exit status 0, 1, and 2 make the work file the disk. Status 4, status 8, a timeout, a kill, and a core dump keep the old disk and remove the work file. So do a link, a file with a second link, a dir, a file of another owner, and a file of another size in the place of the work file. A boot removes a work file that an earlier boot left. On Linux, `TestRepairUnit` checks the command and the properties of the unit, and the D-Bus signatures that systemd takes for them. A status 0 counts only when the unit stops after it. A failed reset of the failed state of a stopped unit keeps the repair, with a warning. A read of a failed unit with the result `success` and no exit status is not the end of `e2fsck`, so the driver reads again. A failed cleanup of an earlier repair logs a warning on a disk without the error state, and an error and `failed` on a disk with it. `Delete` stops a unit of a repair that an earlier boot left.

`TestKVMHibernate` hibernates and wakes a guest that holds a process and data in RAM. The process keeps its PID, and the data stays. `state.json` must record `clock_realtime` as the clock source of the host says, and the test logs that clock source. A truncated memory file fails the load and keeps every file. Damaged bytes in the memory file do not fail the load. A later load puts the disk copy back. A guest that writes its whole RAM after a wake is not killed. A save then starts with a raised limit that a killed save left, and the test fails it during `PUT /snapshot/create`. The limit during the save must be the RAM plus 256 MiB plus the RAM, and the limit after it must be the limit of a boot. Then the test kills `hopboxd` at five points of a save: before each of its three renames, during `PUT /snapshot/create`, and after the rename of `memory`, before the end of the unit. A hook of the driver parks the save at the point for good, so the killed save runs no step of a failure. A new driver must find a paused microVM or a whole memory file. Its save must succeed with the limit of a save. After the rename of `memory`, the new driver must save nothing again: it only ends the unit, and the microVM ends with the reason `snapshot`. A wake must keep the process and the data, with the limit of a boot. Then the test runs `hopbox-guest` of an older release in the guest, from `HOPBOX_TEST_OLD_GUEST`. That guest agent must link with its older version, also after a wake whose link asks for the randomness step, and the guest must keep its process and its data. A hibernate and a wake of a machine with the ext4 error state run no repair, and `disk.raw` keeps its inode. A hibernate from standby, then a delete, removes the saved state. `TestKVMLifecycle` and `TestKVMHibernate` log the clocksource of the guest kernel after a boot and after a load ([Guest kernel](/reference/guest-kernel/index.html.md#clocksource)).

`TestKVMSaveWithoutAStop` saves a guest of 1 GiB without a stop, for T10. During the whole test, a loop in the guest writes a counter to its disk with `O_DIRECT`, and then to RAM. Before each write, the loop checks that the disk holds the count of its last write. A second loop writes 64 MiB at a time to the disk, and a tmpfs holds 256 MiB of random data. After the save, the guest must link again, the counter must go on, and the limit must be the limit of a boot. The disk copy must share most of the blocks of the image disk, by `savedstate.SharedRanges`, and a shared range must hold the bytes of the image. Then the test loads the copy as a second machine, as a fork does. Its guest must link, keep the held process and the data in RAM, and find its disk at the count of its RAM. A disk and a memory from two moments break that check. The file system of the copy must log no ext4 error, and a write in the copy must not reach the source. Then the test logs the pause three times each with the sync in the pause, with the sync after the resume, and without a sync. The PASS line of that subtest proves that the measure ran. Phase 0 measured an idle guest without a `MemoryMax`. A save fails at its create, because a dir is in the place of `memory.new`, and a hook fails a save after its files are whole. Each failure must leave the guest running, no dir, and the limit of a boot. Then the test kills `hopboxd` during `PUT /snapshot/create` and after the files are whole. A new driver must find the microVM paused or running, no dir of the save, and an earlier save that did not change. After a resume, the guest must go on, and a new save must succeed.

`TestKVMSaveRoom` measures the room of a save for Q202. A guest writes all but 96 MiB of its free memory into a tmpfs, and then the driver saves it. The test logs the memory of the unit before and during the save, the highest `memory.current` and `memory.stat` that it saw, and the lines of the kernel log about memory. It measures a guest that never woke and a woken guest. For each, it saves with a raise by the RAM, with the limit of a boot, after a reclaim of the clean page cache, and with a raise by only the excess that the save with the raise by the RAM used. The saves with the raise by the RAM must pass. The others only log: a subtest whose name starts with `measure` passes also when its save fails, so its PASS line in CI proves that the measure ran, not that the save worked. A save that runs for 3 minutes fails the test, and the test kills its unit. If the save still runs one minute later, the test ends the run.

`make test-kvm` runs the KVM tests. Each machine of these tests has at most 1 GiB of RAM, because the host of the CI runner also runs other services. `TestKVMLifecycle` boots the default image with the real guest agent and checks the unit, the jail user, the cgroup, the tap device, and the disk. Then it restarts the driver and attaches, stops, and boots from the kept disk. It reboots and powers off from inside, which must give `shutdown`. A kernel panic and a kill of Firecracker must give `crash`. It checks `ExecStop`, `Before`, and `TimeoutStopUSec` of the unit. Then the guest turns its writeback off and writes a file without a sync, and `systemctl stop` stops the unit. That must give `shutdown`, and the file must be there at the next boot. Then the stop helper is `/bin/false`, as after a rollback. A reboot from inside must still give `shutdown`, and the unit must not end failed. A stop of the unit must then give `crash`, with a root file system in use. Then it tests [Repair of the root file system](#repair-of-the-root-file-system). With the repair off by a hook of the test, it sets the ext4 error state through `/sys/fs/ext4/vda/trigger_fs_error`, syncs, and reboots. It boots, holds an unlinked open file of 64 MiB, syncs, and kills the unit with `SIGKILL`. After the next boot, a reboot from inside must give `crash`. With the repair on, the next boot must log the repair, and give the 64 MiB back. A reboot from inside must then give `shutdown`, and the superblock must have no error state. A fake `e2fsck` with exit status 4 and with 8 must keep the inode of `disk.raw` and its error state, and the boot goes on. A fake `e2fsck` writes to the work file and sleeps. The test kills its unit with `SIGKILL`, and `disk.raw` must not change, while the boot goes on. A fake `e2fsck` that writes and sleeps again stands for a stop of `hopboxd` during a repair: the test ends the context of `Boot` and kills nothing. The boot must fail, the unit of the repair must be inactive, `disk.raw` must keep its inode and its error state, and no work file may stay. A boot with a new driver and the real `e2fsck` must then repair. For a kill of `hopboxd` during a repair, the test starts a unit that sleeps under the name of the unit of the repair, and writes a work file. A boot with a new driver must then log the removal of the work file and the repair, and leave the unit inactive. Then the same sequence without the hook must repair at the first boot after the error, and a reboot after the orphan must give `shutdown`. It also attaches to a microVM that never started, and deletes the machine. It pauses a guest that holds a process and data in RAM, restarts the driver, attaches, and resumes. The process, the data, and the boot ID must stay. It stops a paused guest, also through `systemctl stop`, and the root file system must end clean, with no lost file. A kill of a paused guest must give `crash`. `TestKVMFailedBoot` checks that a failed boot leaves no unit and no tap device. `TestKVMSystemdConnection` checks that the connection to systemd outlives the call that made it. `TestKVMEndToEnd` of `internal/control` runs a single-host install. It adds the size `kvm`, with 1 GiB of RAM, and makes a machine of that size through the API. It runs exec, stops and starts the machine, upgrades `hopboxd` with the microVM running, and deletes the machine. It also pauses the machine through the API, reads `hopbox_guest_links_down` for 30 s, resumes, and logs the time to the first exec. It restarts `hopboxd` in standby and stops a machine in standby. A kill of the unit in standby, and a restart of `hopboxd` with no microVM, must each end at the target of the restart policy. It hibernates and wakes the machine through the API, also across a restart of `hopboxd`, and logs the time to the first exec. After a standby of 60 s and after a hibernate of 60 s, the clock of the guest must be within 2 s of the host. Before the standby, the test sets the guest clock 10 min back, so the guest agent must set it after the resume. The test logs the jump that the guest agent made, and the clocksources of the guest and the host. A first wake must run no randomness step. A stop of `hopboxd` during a save must leave a machine that runs, or a whole memory file. A truncated memory file, and damaged bytes in it, must fail the wake and keep the file. The damaged bytes fill the kernel code, which `/proc/iomem` of the guest names, with an endless jump. Firecracker reads none of it at the load, so the load passes, the guest hangs, and the wake fails by the timeout of its link. `state.json` must count both loads, and the disk copy must stay. An admin move and a cold drain must refuse the machine after that. A later wake must run from the disk copy and run the randomness step, and `DiscardMemory` must drop the file, so the next start boots from the disk. It also checks the node network with a second machine ([the node network](/reference/node-network/index.html.md#tests)). The KVM tests run in a network namespace of their own.

| Variable of `make test-kvm` | Default | Description |
| --- | --- | --- |
| `KVM_WORKDIR` | none, required | A dir on XFS with reflinks, of at most 22 characters. |
| `FIRECRACKER`, `JAILER` | `firecracker`, `jailer` | Firecracker v1.17.0 and its jailer, on the host of the tests. |
| `KVM_HOST` | empty | An SSH host that runs the tests. Empty runs them on this host. |
| `KVM_DIR` | `/var/tmp/hopbox-kvm` | The dir on `KVM_HOST` for the test binaries, the kernel, the image, and Postgres. |
| `KVM_SCOPE` | empty | With a name, the test binaries run in the systemd scopes `<name>-driver` and `<name>-control`. |
| `KVM_TEST_ENV` | empty | More env vars for the tests. |
| `KVM_OLD_GUEST` | `v0.1.0` | The tag of the older release whose `hopbox-guest` the tests run. make builds it from the tag with `git archive`, so the checkout needs the tag. |
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
| `HOPBOX_TEST_JAIL_ID_FIRST` | `1879110000` for `TestKVMLifecycle`, `1879108192` for `TestKVMEndToEnd` | The first of the 100 jail IDs of each test. |
| `HOPBOX_TEST_GUEST_VERSION` | none | The version that `hopbox-guest version` must print in the machine of `TestKVMLifecycle`. Without it, that subtest skips. The CI job `fleet` sets it to the version of the checkout. |
| `HOPBOX_TEST_OLD_GUEST` | none | `hopbox-guest` of an older release, for Linux. `TestKVMHibernate` runs it in the guest. Without it, that subtest skips. `make test-kvm` builds it from `KVM_OLD_GUEST`. |
| `HOPBOX_TEST_PG_USER` | `hopbox-pg` | The user of Postgres in the tests of `internal/control`, which run as root. Set it on a host without the user `hopbox-pg`, such as `nobody`. |
