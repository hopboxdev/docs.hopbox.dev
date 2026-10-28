# Server layout

> Every path, user, ID range, systemd unit, and reserved port that hopboxd install makes on a server, the image cache, and the file of the bootstrap admin API key.

This page lists every path, user, unit, and kernel setting that `hopboxd install` makes on a server ([hopboxd](/reference/hopboxd/index.html.md#install)). The layout follows the defaults of phase 1. The code is in `internal/install`.

## Paths

The install makes the paths of the roles in the bootstrap config. `<data_dir>` is `data_dir` of the config, `/var/lib/hopbox` by default ([the bootstrap config of hopboxd](/reference/hopboxd-config/index.html.md#keys)).

| Path | Owner | Mode | Role | What it holds |
| --- | --- | --- | --- | --- |
| `/opt/hopbox` | root | 0755 | every role | The releases. |
| `/opt/hopbox/releases/<version>` | root | 0755 | every role | One release: the files of a bundle without the disk of the image ([Release bundle](/reference/release-bundle/index.html.md#contents)). |
| `/opt/hopbox/current` | root | symlink | every role | Points to `releases/<version>` of the release that runs. The unit runs `/opt/hopbox/current/hopboxd`, and the config names `/opt/hopbox/current/postgres`. |
| `/opt/hopbox/previous` | root | symlink | every role | Points to the release that ran before, for a rollback. A first install has none. |
| `/opt/hopbox/.lock` | root | 0600 | every role | The lock of `install` and `upgrade`. See [Releases](#releases). |
| `/opt/hopbox/.restart-pending` | root | 0644 | every role | Exists from the first change of the config, the release, or a unit until the restart of `hopboxd` that runs it. A run that stopped between the two leaves it, and the next install then restarts `hopboxd`. |
| `/usr/local/bin/hopboxd` | root | symlink | every role | Points to `/opt/hopbox/current/hopboxd`, so `sudo hopboxd admin ...` runs the current release. |
| `/etc/hopbox` | root | 0755 | every role | The config dir. |
| `/etc/hopbox/hopboxd.toml` | root | 0644 | every role | The bootstrap config ([the bootstrap config of hopboxd](/reference/hopboxd-config/index.html.md)). It holds no secret. |
| `/etc/hopbox/postgres-ca.pem` | root | 0644 | control, with `postgres.url` | The CA of the external Postgres, at the path of `postgres.ca_file`. The operator or the Terraform provider writes it, not the install ([the bundled Postgres](/reference/postgres/index.html.md#external-postgres)). |
| `/etc/hopbox/postgres-password` | root | 0400 | control, with `postgres.url` | The password of the external Postgres, at the path of `postgres.password_file`. The operator or the Terraform provider writes it, not the install. |
| `<data_dir>` | root | 0755 | every role | The state of the server. |
| `<data_dir>/pg` | `hopbox-pg` | 0700 | control, without `postgres.url` | The Postgres cluster in `data/` ([the bundled Postgres](/reference/postgres/index.html.md#paths-on-a-server)). The install runs `initdb` there. |
| `<data_dir>/kek` | root | 0700 | control | The key-encryption keys. The install makes the first one, `1.key`, with mode 0400 ([Sealing](/reference/sealing/index.html.md#key-files)). |
| `<data_dir>/export` | root | 0700 | control | The dump of `hopboxd admin db export`, mode `0600`, while the export runs. The export removes it at its end ([hopboxd](/reference/hopboxd/index.html.md#admin-db-export)). |
| `<data_dir>/bootstrap-admin-key.sha256` | root | 0600 | control | The hash of the bootstrap admin API key. See [The bootstrap admin API key](#the-bootstrap-admin-api-key). |
| `<data_dir>/images` | root | 0700 | node | The image cache. See [The image cache](#the-image-cache). |
| `<data_dir>/kernels` | root | 0700 | node | The kernel cache: each guest kernel that the node got with a release, named by its sha256 ([the Firecracker driver](/reference/firecracker-driver/index.html.md#the-kernel-cache)). |
| `<data_dir>/machines` | root | 0700 | node | One dir for each machine, with its disk and its jail ID ([the Firecracker driver](/reference/firecracker-driver/index.html.md#files-of-a-machine)). |
| `<data_dir>/jail` | root | 0700 | node | The chroot base of the jailer, with the jail of each machine in `jail/firecracker/<id>/root`. It is on the file system of `machines` and `kernels`, so a disk and a kernel can enter a jail as hard links. |
| `<data_dir>/executor` | root | 0700 | node | The node-local state of the executors: one file for each machine ([the node executor](/reference/executor/index.html.md#node-local-state)). |
| `<data_dir>/network` | root | 0700 | node | The state of the node network. The file `forward-guard`, mode 0600, exists once `hopboxd` turned IPv4 forwarding on, and keeps the forward guard of the table ([the node network](/reference/node-network/index.html.md#the-forward-guard)). |
| `<data_dir>/hostfw` | root | 0700 | every role | The state of the host firewall. The file `vrack-interface`, mode 0600, names the vRack interface once the server had a vRack address. So a server that loses the address keeps its public interface closed ([the host firewall](/reference/host-firewall/index.html.md#never-a-lockout)). On an edge server, the file `forward-guard`, mode 0600, keeps the forward guard of the table. It exists once `hopboxd` turned IPv4 forwarding on ([the host firewall](/reference/host-firewall/index.html.md#the-forward-guard)). The file `forward-guard-ipv6`, mode 0600, exists once `hopboxd` turned IPv6 forwarding on ([the egress of the machines](/reference/egress/index.html.md#ipv6-forwarding)). |
| `<data_dir>/logs` | root | 0700 | node | The logs of the machines: one directory per machine, with mode 0700, and segment files with mode 0600 ([Machine logs](/reference/machine-logs/index.html.md#files)). |
| `<data_dir>/dns` | root | 0700 | dns | The last zones that the DNS role got from the control plane, in the file `zones`, mode 0600. The DNS role answers from them after a restart while no control server answers ([the DNS role](/reference/dns/index.html.md#zones-on-the-disk)). |
| `<data_dir>/server` | root | 0700 | node, dns, edge | The identity of the enrolled server ([Enrollment and the install PKI](/reference/enrollment/index.html.md)), and the file `control-urls` with mode `0600`: the URLs of the live control servers that the control role sent last, one per line ([the control channel](/reference/control-channel/index.html.md#control-urls)). |
| `/etc/systemd/system/hopboxd.service` | root | 0644 | every role | The unit of `hopboxd`. See [systemd units](#systemd-units). |
| `/etc/systemd/system/hopbox-machines.slice` | root | 0644 | every role | The slice of the machines. |
| `/etc/sysctl.d/60-hopbox.conf` | root | 0644 | node | The reserved SSH ports at boot. See [Reserved ports](#reserved-ports). |
| `/run/hopbox` | root | 0755 | every role | The sockets: `admin.sock` and `pg/`. `hopboxd run` makes it at each start. |

The install never removes anything under `<data_dir>`.

Root writes and runs what these dirs hold. So the install refuses a dir that another user than root owns, and a dir that is a symlink. It sets the mode of the table on each dir that it makes or finds. A dir that group or others can write thus gets the safe mode back. The host owns three dirs that the install writes into: `/etc/systemd/system`, `/etc/sysctl.d`, and `/usr/local/bin`. The install keeps their mode. If group or others can write such a dir, the install refuses it. `postgres.md` and `sealing.md` give the checks of `<data_dir>/pg` and `<data_dir>/kek`. The data dir itself must be a real dir: mount the file system on it, and do not use a symlink.

### The file system of the data dir

On a server with the node role, `<data_dir>` must be on XFS with reflinks: the node makes the disk of a machine as a reflink copy of its image. When the Terraform provider installs a server, it makes this file system. By hand, make it with `mkfs.xfs -m reflink=1` and mount it on `/var/lib/hopbox` ([Install hopbox on a Debian 13 server by hand](/guide/install-debian/index.html.md)).

The install checks the file system before it changes anything. If `<data_dir>` does not exist yet, it checks the nearest dir above it. The install refuses another file system, and XFS without reflinks. The preflight check `reflink` tests reflinks again in `<data_dir>` ([the preflight check](/reference/preflight/index.html.md)). A server with only the control role has no such rule.

## Releases

Each install and each upgrade keeps two releases: the current one and the previous one. After a good start of `hopboxd`, the install removes every other dir in `/opt/hopbox/releases`. This includes the temp dirs `.incoming-*` of an upgrade that stopped.

A release dir is the release of one bundle. The install checks it before it uses it ([Release bundle](/reference/release-bundle/index.html.md#checks-of-a-release)). A release dir with the name of a version but another manifest stops the install: two builds of one version never share a name.

The switch of `current` is one rename of a new symlink over the old one. So a reader sees the old release or the new one, never a mix. The same holds for `previous`. The install moves `previous` first and `current` second. So after a crash between the two renames, `previous` still names the release that ran.

The file `/opt/hopbox/.lock` keeps two runs of `install` and `upgrade` apart, with `flock`. A run that finds the lock taken stops at once. `upgrade` holds the lock while it unpacks. It passes the lock to the install of the new release as the file descriptor 3, which the environment variable `HOPBOX_INSTALL_LOCK_FD` names. That install takes the lock on the same open file, so the lock covers the whole upgrade. It then marks the descriptor close-on-exec and removes the variable. So no program that it runs, such as `useradd`, `systemctl`, or Postgres, keeps the lock. The descriptor must be the lock file, and its `flock` must succeed. Otherwise the install refuses it. A descriptor of its own on the lock file is another open file, so it fails while the upgrade holds the lock.

## The image cache

`<data_dir>/images/<hex>` holds one image. `<hex>` is the digest of the image without `sha256:`, for example `images/354fb767...`. A dir holds `artifact.json` and the compressed disk `disk.raw.zst` ([Artifact format](/reference/artifact-format/index.html.md#digest)).

The install of a node copies the default image of its release into the cache, unless the cache has it already. It writes the copy in a temp dir `images/.incoming-<random>`, checks its digest and the sha256 of its disk, and renames it into place. Then it removes the disk from the release dir. A server without the node role also removes the disk and keeps no image. If such a server gets the node role later, run that install from an unpacked bundle, which has the disk. The install keeps the images of earlier releases, because a rollback needs them. It never removes an image, so the cache grows by about 560 MB with each upgrade that brings a new image. Remove a dir of an image that no release on the server names, and that no machine uses, by hand.

At the first boot of a machine from the image, the node role unpacks the disk to `disk.raw`. Then it checks the disk and removes the compressed copy ([the Firecracker driver](/reference/firecracker-driver/index.html.md#the-image-cache)). A node can also download the disk of an image that an admin registered with a URL. Its dir then holds only `disk.raw`, and a later install of a release with that image adds its `artifact.json`.

## Users and IDs

| User | Role | What it is for |
| --- | --- | --- |
| `hopbox-pg` | control, without `postgres.url` | Runs the bundled Postgres, which refuses to run as root ([the bundled Postgres](/reference/postgres/index.html.md#os-users-and-database-roles)). The name comes from `postgres.user` of the config. |
| `hopbox-jail` | node | Holds the user and group IDs of the jailer in `/etc/subuid` and `/etc/subgid`. No process runs as this user. |

The install makes each user with `useradd --system --user-group --no-create-home --home-dir /nonexistent --shell /usr/sbin/nologin`, unless the user exists.

The jailer runs the microVM of each machine with its own user ID and group ID. They come from this range:

| Item | Value |
| --- | --- |
| First ID | 1879048192 (`0x70000000`) |
| Count | 65536 |
| Last ID | 1879113727 |
| Entry in `/etc/subuid` and `/etc/subgid` | `hopbox-jail:1879048192:65536` |

The range is above the container ranges of systemd (524288 to 1879048191) and below 2^31, a gap that the UID map of systemd leaves unused. `useradd` gives normal users IDs below 60000 and subordinate IDs below 600100000. The entry in `/etc/subuid` and `/etc/subgid` tells every other tool of the host that the range is taken. The install adds the entry with `usermod --add-subuids` to the file that lacks it, and with `--add-subgids` to the other, so no file gets it twice. If another entry of `/etc/subuid` or `/etc/subgid` overlaps the range, the install refuses it. The same holds for a user of `/etc/passwd` or a group of `/etc/group` with an ID in the range.

## systemd units

### hopboxd.service

```ini
[Unit]
Description=hopbox server
After=network-online.target
Wants=network-online.target
RequiresMountsFor=/var/lib/hopbox

[Service]
Type=notify
ExecStart=/opt/hopbox/current/hopboxd run --config /etc/hopbox/hopboxd.toml
Restart=on-failure
RestartSec=5s
TimeoutStartSec=180s
TimeoutStopSec=105s
KillMode=mixed
LimitNOFILE=1048576
NoNewPrivileges=yes
ProtectSystem=full
ProtectHome=yes
PrivateTmp=yes
ProtectKernelModules=yes
ProtectKernelLogs=yes
ProtectControlGroups=yes
ProtectClock=yes
ProtectHostname=yes
RestrictSUIDSGID=yes
RestrictRealtime=yes
LockPersonality=yes

[Install]
WantedBy=multi-user.target
```

| Setting | Why |
| --- | --- |
| `Type=notify` | When every unit of the process is ready, `hopboxd run` sends `READY=1` ([hopboxd](/reference/hopboxd/index.html.md#run)). So `systemctl start` and `systemctl restart` return only at that time. If `hopboxd` exits first, they fail. An install and an upgrade depend on this. |
| `Restart=on-failure`, `RestartSec=5s` | If a unit fails, `hopboxd` exits with status 1, and systemd starts it again after 5 s. |
| `TimeoutStartSec=180s` | The first start migrates the control store. Postgres alone has 60 s to start. |
| `TimeoutStopSec=105s` | With every role, the stop timeouts of the units of `hopboxd` add up to 104.1 s. With the control role and the node role, they add up to 98.1 s ([hopboxd](/reference/hopboxd/index.html.md#run)). |
| `RequiresMountsFor=/var/lib/hopbox` | `hopboxd` starts only after the file system of the data dir is mounted. Without it, a slow mount at boot lets `hopboxd` write into the empty dir below the mount point. |
| `NoNewPrivileges=yes` | No process of the unit can gain privileges through a setuid program. `hopboxd` runs as root and only drops privileges, for example for Postgres. |
| `KillMode=mixed` | systemd sends `SIGTERM` to `hopboxd` alone, and `hopboxd` stops Postgres in order. A `SIGTERM` to every process of the unit stops Postgres out of order. |
| `ProtectSystem=full` | `/usr`, `/boot`, `/efi`, and `/etc` are read-only for the unit. `hopboxd run` writes only the data dir and `/run/hopbox`. `hopboxd install` and `hopboxd upgrade` run outside the unit, so they still write `/etc/hopbox` and `/opt/hopbox`. |
| `ProtectHome=yes` | `/home`, `/root`, and `/run/user` are empty for the unit. |
| `PrivateTmp=yes` | The unit has its own `/tmp` and `/var/tmp`. |
| `ProtectKernelModules=yes`, `ProtectKernelLogs=yes` | No process of the unit loads a kernel module or reads the kernel log. The kernel still loads the modules of nftables by itself. |
| `ProtectControlGroups=yes` | The cgroup tree is read-only for the unit. The unit of each machine gets its cgroup from systemd, over D-Bus. |
| `ProtectClock=yes`, `ProtectHostname=yes` | No process of the unit changes the clock or the host name of the server. |
| `RestrictSUIDSGID=yes`, `RestrictRealtime=yes`, `LockPersonality=yes` | No process of the unit makes a setuid or setgid file, takes a realtime priority, or changes its execution domain. |

The acceptance suite passes on the [tested server](/reference/preflight/index.html.md#the-tested-server) with these settings, through the boot, the restart policy, and the delete of real machines. Two settings stay off, because the node role needs what they block:

- `ProtectKernelTunables`: the node network writes sysctls at its start, such as `net.ipv4.ip_forward` ([the node network](/reference/node-network/index.html.md)).
- `PrivateDevices`: the driver makes the tap device of each machine through `/dev/net/tun`, and the node checks `/dev/kvm`.

The jailer and Firecracker run in the unit of each machine, not in this unit.

### hopbox-machines.slice

```ini
[Unit]
Description=hopbox machines

[Slice]
```

The microVM of each machine runs in its own transient unit in this slice, `hopbox-machine-<id>.service`, not in `hopboxd.service`. [The Firecracker driver](/reference/firecracker-driver/index.html.md#the-unit-of-a-machine) lists the properties of such a unit. A restart or an upgrade of `hopboxd` stops only the processes of `hopboxd.service`, so the microVMs keep running. The slice has no `[Install]` section: systemd starts it with the first unit in it. At a shutdown of the server, systemd stops `hopboxd.service` first, then each unit in the slice, and each guest shuts down cleanly ([the Firecracker driver](/reference/firecracker-driver/index.html.md#a-stop-through-systemd)).

The install writes each file whose content changed. Then it runs `systemctl daemon-reload`. It runs `systemctl enable hopboxd.service` on each run.

## Reserved ports

The SSH ports of the machines are 10000 to 30000. The kernel must not give them to an outgoing connection of the host. So the install of a server with the node role reserves them in two places:

| Place | What the install does |
| --- | --- |
| `/etc/sysctl.d/60-hopbox.conf` | Writes `net.ipv4.ip_local_reserved_ports = 10000-30000`, which `systemd-sysctl` applies at each boot. The setting covers IPv4 and IPv6. |
| `/proc/sys/net/ipv4/ip_local_reserved_ports` | Adds `10000-30000` to the list of the running kernel, and keeps the ports that the list holds already. If the list covers the range, it changes nothing. |

The drop-in sets the whole list. If another drop-in sets this key with a name that sorts after `60-hopbox.conf`, that drop-in wins at boot.

The disk listener of a node reserves no port either. It listens on port 7444 of the mesh address only ([Cold move](/reference/cold-move/index.html.md#the-disk-listener)).

The DNS role reserves no port. It listens on the ports of `dns.listen`, port 53 by default ([the bootstrap config of hopboxd](/reference/hopboxd-config/index.html.md#keys)). The edge role reserves no port either. It listens on `edge.https_listen`, `edge.http_listen`, and `edge.ssh_listen`, ports 443, 80, and 2222 by default. At its start, it also replaces the nftables table `inet hopbox_edge`, which sends the SSH ports 10000 to 30000 to `edge.ssh_listen`. A stop deletes the table ([the SSH front door](/reference/ssh-front-door/index.html.md#redirect-of-the-ssh-ports)). The redirect matches only new connections, so an edge server without the node role needs no reserved ports.

## The bootstrap admin API key

Before any control role runs, the Terraform provider makes a random admin API key. It passes only the hash of the key to the install of the first server. `hopboxd install --bootstrap-admin-key-hash` writes the hash to `<data_dir>/bootstrap-admin-key.sha256`. Only the install that makes the Postgres cluster of the server writes the file. A later run ignores the flag and logs a warning. So a later run can never bring back a key that an admin revoked.

| Item | Value |
| --- | --- |
| Content | The SHA-256 of the whole admin API key, as 64 lowercase hex digits, then a newline |
| Owner and mode | root, 0600 |
| Role | control only |

The file is the hand-off from the install to the control role. The install writes the file and needs no running control role and no admin call for the key. The `store` unit of `hopboxd run` reads the file after the migrations ([hopboxd](/reference/hopboxd/index.html.md#run)):

| Case | What the control role does |
| --- | --- |
| The install has no admin API key. | It registers the hash as the admin API key `bootstrap`, with a lifetime of 90 days. It calls `RegisterAdminKey` as the caller `local_root` of the `hopboxd` process. The audit log gets `admin_key.register`. Then it removes the file and logs `registered the bootstrap admin API key`. |
| The install has an admin API key already, expired or not. | It removes the file unused and logs a warning, so a stale file never adds a key. |
| The file is wrong: not a regular file, not of the user of `hopboxd` (root on a server), or not mode 0600. Or it does not hold 64 lowercase hex digits and an optional newline. | It registers nothing, keeps the file, and logs an error. The start goes on. |

The key text itself never reaches the server. [CLI tokens, API keys, admin API keys, and SSH keys](/reference/credentials/index.html.md#admin-api-keys) shows how to make a key and its hash.
