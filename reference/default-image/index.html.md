# Default image

> The programs, users, services, ports, files, disk, pins, build, and boot test of the default image.

The default image is the disk that a new machine starts from. It is Ubuntu 26.04 LTS for x86_64, with the programs below, the guest agent `hopbox-guest`, and systemd. The recipe lives in `images/default/`. Each build is an artifact in the format of [Artifact format](/reference/artifact-format/index.html.md). `make image` builds it, and `make check-image` checks it (the build guide of the repo has the steps and a boot test).

## Programs

The design asks for the latest stable release of each program, and the latest LTS release of Ubuntu and Node.js. `pins.env` fixes these versions for a build, and `make image-pins` moves them to the latest releases ([Pins](#pins)). Each program with an official build of its own comes from that build:

| Program | Source | Installed as |
| --- | --- | --- |
| Ubuntu 26.04 LTS | the Docker image `ubuntu:26.04`, by digest, and a snapshot of the Ubuntu archive | the base system |
| Docker CE with Compose and Buildx | the apt repository of Docker, `download.docker.com`, signed by its pinned key | `docker`, `docker compose`, `docker buildx`, `containerd` |
| nginx, latest stable release | the stable apt repository of `nginx.org`, signed by its pinned key | the HTTP server on port 8000, off by default ([nginx](#nginx)) |
| Node.js, latest LTS | the release archive from `nodejs.org`, with its sha256. The build checks the signature of `SHASUMS256.txt` against the release keys of Node.js. | `/usr/local/bin/node`, `npm`, `npx`, `corepack` |
| Python, latest stable release | the CPython build of `python-build-standalone`, installed by uv, which checks its sha256 | `python` and `python3` of `user` in login shells, from `~/.local/bin` ([Python](#python)) |
| uv | the release archive from GitHub, with its sha256 | `/usr/local/bin/uv`, `uvx` |
| Go | the release archive from `go.dev`, with its sha256 | `/usr/local/go`, with `go` and `gofmt` in `/usr/local/bin` |
| GitHub CLI | the release archive from GitHub, with its sha256 | `/usr/local/bin/gh` |
| ripgrep | the static release archive from GitHub, with its sha256 | `/usr/local/bin/rg` |
| Neovim | the release archive from GitHub, with its sha256 | `/usr/local/bin/nvim` |

The other programs have no official build for Ubuntu. They come from the Ubuntu archive, at the version of the snapshot. This version is the latest version that Ubuntu 26.04 LTS ships, which can be older than the latest release of the project. This is an exception to the rule of the design:

| Program | Installed as |
| --- | --- |
| systemd | the init system, PID 1 |
| OpenSSH server and client | `sshd`, started by `ssh.socket` on port 22 |
| Python 3 of Ubuntu | `/usr/bin/python3`, with `venv` and `pip`, for system tools. `/usr/bin/python` points at it (`python-is-python3`). |
| FFmpeg | `ffmpeg`, `ffprobe` |
| Vim | `vim` |
| git, curl, and the packages below | |

The image also has more Ubuntu packages, for work on a machine. apt installs them without recommended packages.

| Group | Packages |
| --- | --- |
| System | `sudo`, `dbus`, `libpam-systemd`, `udev`, `tzdata`, `ca-certificates` |
| Network | `iproute2`, `iptables`, `iputils-ping`, `netcat-openbsd`, `bind9-dnsutils`, `curl`, `wget` |
| Files and text | `git`, `gnupg`, `rsync`, `unzip`, `zip`, `xz-utils`, `zstd`, `jq`, `less`, `file` |
| Shell and processes | `tmux`, `htop`, `procps`, `psmisc`, `lsof`, `bash-completion` |
| Builds | `build-essential`, `pkg-config` |

Phase 1 leaves these parts out of the image:

- coding agents, and their instructions and skill
- the desktop and the browser
- the in-machine CLI
- the CA certificate of an egress proxy

Phase 5 adds the coding agents and the in-machine CLI.

The Ubuntu container image is minimized: it leaves out manual pages and most of `/usr/share/doc`. The default image keeps it so.

### Program manifest

Each build records the version of each program in a manifest. The manifest is in the image at `/usr/share/hopbox/programs.json`, and in the `programs` field of `artifact.json` ([Artifact format](/reference/artifact-format/index.html.md#artifactjson)). The build reads each version from the installed program or package. For an Ubuntu or Docker package, the version is the version of the package.

| Name | Version from |
| --- | --- |
| `ubuntu` | `VERSION` in `/etc/os-release`, for example `26.04.1`. The source is the pinned Docker image. |
| `systemd`, `openssh-server`, `sudo`, `git`, `ffmpeg`, `vim` | the Ubuntu package. The source is the snapshot of the archive, for example `snapshot.ubuntu.com/ubuntu/20261016T080000Z`. |
| `python3` | `/usr/bin/python3 --version`, the interpreter of Ubuntu. Its version can differ from the version of the package `python3`. |
| `docker-ce`, `containerd.io`, `docker-buildx-plugin`, `docker-compose-plugin` | the Docker package |
| `nginx` | the nginx.org package |
| `python` | `~/.local/bin/python3 --version` of `user`, the CPython of uv |
| `ripgrep` | `rg --version` |
| `node`, `npm` | `node --version`, `npm --version` |
| `go` | `go version` |
| `uv` | `uv --version` |
| `gh` | `gh --version` |
| `neovim` | `nvim --version` |
| `hopbox-guest` | `hopbox-guest version` |

Example from the build of 2026-10-16, with the snapshot `20261016T080000Z`:

```sh
$ jq -r '.[] | "\(.name) \(.version)"' /usr/share/hopbox/programs.json
containerd.io 2.4.2-2~ubuntu.26.04~resolute
docker-buildx-plugin 0.38.0-1~ubuntu.26.04~resolute
docker-ce 5:29.9.1-1~ubuntu.26.04~resolute
docker-compose-plugin 5.6.0-1~ubuntu.26.04~resolute
ffmpeg 7:8.0.1-3ubuntu2
gh 2.102.0
git 1:2.53.0-1ubuntu1
go 1.27.2
hopbox-guest v0.0.0-m18dev
neovim 0.12.6
nginx 1.30.6-1~resolute
node 24.22.0
npm 11.19.1
openssh-server 1:10.2p1-2ubuntu3.8
python 3.15.0
python3 3.14.4
ripgrep 15.2.0
sudo 1.9.17p2-1ubuntu3.2
systemd 259.5-0ubuntu3.4
ubuntu 26.04.1
uv 0.13.1
vim 2:9.1.2141-1ubuntu4.9
```

## Python

The image has two Pythons:

| Python | Path | For |
| --- | --- | --- |
| The latest stable CPython, for example 3.15.0 | `~/.local/bin/python` and `python3` of `user`, links into `~/.local/share/uv/python` | `user` in login shells: SSH sessions, `hopbox connect`, and `bash -l`. The `~/.profile` of Ubuntu puts `~/.local/bin` first in the `PATH`. |
| The Python 3 of Ubuntu | `/usr/bin/python3` and `/usr/bin/python` | root, system tools, and commands without a login shell, for example exec without `bash -l` |

The build runs `uv python install --default` as `user` with the pinned version. uv downloads the build of `python-build-standalone` and checks it against the sha256 that the pinned uv ships. To get another Python, run `uv python install 3.14`, or use `uv venv` and `uv run` in a project.

## Users

| User | ID | Groups | Shell | Password | sudo |
| --- | --- | --- | --- | --- | --- |
| `user` | 1000, group `user` 1000 | `user`, `sudo`, `docker` | `/bin/bash` | none (`*`): no password works | every command, without a password |
| `root` | 0 | `root` | `/bin/bash` | none | |

`user` is the main user of a machine. The guest agent runs the start command, exec, and file copy as `user` by default ([hopbox-guest](/reference/hopbox-guest/index.html.md)). Its home is `/home/user`. The Ubuntu container image has a user `ubuntu` with the ID 1000. The build deletes it.

`/etc/sudoers.d/90-hopbox-user` (mode 0440) gives `user` sudo without a password. The group `docker` lets `user` run `docker` without sudo.

SSH logins use keys only. The guest agent writes the key of the SSH front door to `/etc/hopbox/authorized_keys`, which sshd reads for every user ([hopbox-guest](/reference/hopbox-guest/index.html.md#files)). A user can add keys to `~/.ssh/authorized_keys`.

## Services

| Unit | State | What it does |
| --- | --- | --- |
| `hopbox-guest.service` | enabled | The guest agent. It sets up the network, the host name, and the SSH host key, and keeps the guest link open ([hopbox-guest](/reference/hopbox-guest/index.html.md#systemd-unit)). |
| `ssh.socket` | enabled | Listens on port 22 and starts `sshd` at the first connection. |
| `docker.service`, `docker.socket`, `containerd.service` | enabled | Docker. |
| `nginx.service` | disabled | The HTTP server on port 8000 ([nginx](#nginx)). |
| `systemd-growfs-root.service` | runs at each boot | Grows the root file system to the size of the disk ([Disk](#disk)). |
| `systemd-firstboot.service` | masked | Without the mask, it asks for a locale and a time zone on the serial console at the first boot, and the boot waits. |
| `systemd-ssh-generator` | masked | Without the mask, it opens sshd on an `AF_VSOCK` socket and on a local socket. Only the guest agent uses vsock. |

The image has no network manager. The guest agent sets up the network from the configuration that the node pushes ([Guest link](/reference/guest-link/index.html.md#configuration)). The image has no `cloud-init`.

The default target is `graphical.target`, as in Ubuntu. It has no graphical programs, so it is the same as `multi-user.target`.

## Ports

| Port | Service | State |
| --- | --- | --- |
| 22/tcp | sshd, through `ssh.socket` | on |
| 8000/tcp | nginx | off. `sudo systemctl enable --now nginx` turns it on. |

The guest has no other listening port. The guest link is a connection that the guest agent dials over vsock, so the guest listens on no vsock port.

## nginx

nginx serves `/var/www/html` on port 8000, the default port of the machine URL. It is off by default. To turn it on now and at each boot:

```sh
sudo systemctl enable --now nginx
curl -s http://localhost:8000/
```

nginx comes from the stable apt repository of nginx.org, so its layout is the layout of nginx.org: the main file is `/etc/nginx/nginx.conf`, and the workers run as the user `nginx`. The site is `/etc/nginx/conf.d/default.conf`. The page `/var/www/html/index.html` says that nginx serves it. Replace the page, or change the site to proxy to an app.

## No vsock for services

Only the guest agent talks to the node over vsock. The drop-in `/usr/lib/systemd/system/service.d/50-hopbox-no-vsock.conf` sets two values for every systemd service:

| Value | Effect |
| --- | --- |
| `RestrictAddressFamilies=~AF_VSOCK` | A process of the service cannot open an `AF_VSOCK` socket. `socket()` fails with `EAFNOSUPPORT`, "Address family not supported by protocol". |
| `SystemCallArchitectures=native` | The process can make only x86_64 system calls. The filter of `RestrictAddressFamilies` checks only these calls. Without this value, a 64-bit process can open the socket through the i386 call `socket` of `int 0x80`. A test on a server proved it: with `RestrictAddressFamilies` alone, the probe of the boot test opened the socket, and with both values the kernel ended it with `SIGSYS`. |

The guest kernel also has no 32-bit system calls ([Guest kernel](/reference/guest-kernel/index.html.md#config-fragment)), so this path is closed twice. The drop-in `/usr/lib/systemd/system/hopbox-guest.service.d/60-hopbox-allow-vsock.conf` sets empty values for the guest agent. systemd applies drop-ins in the order of their file names, so the empty values undo the restriction.

systemd applies the restriction with a seccomp filter. A child process keeps the filter of its parent, also through `sudo`. So these processes cannot open `AF_VSOCK` either:

| Process | Why |
| --- | --- |
| SSH sessions, and `sudo` in them | `ssh.service` starts them. |
| User services of `systemctl --user` | `user@.service` starts them. |
| Docker containers | `containerd.service` starts them. |
| `systemd-run` commands and timers | Each one is a service. |
| nginx and its workers | `nginx.service` starts them. |

Processes that the guest agent starts are not restricted: the start command, and the commands of exec and `hopbox connect`. They are children of the guest agent. Also, root can remove the drop-in. The [threat model of the guest link](/reference/guest-link/index.html.md#threat-model) does not depend on this restriction.

The filter does not bring `NoNewPrivileges` to `user@.service`, the user manager of `systemctl --user`. On the [tested server](/reference/preflight/index.html.md#the-tested-server), the user manager of `user` runs with the filter and with `NoNewPrivs: 0`, and the boot test checks it. So `sudo` works in SSH sessions, in `hopbox connect`, in exec, and in user services. The systemd docs say that systemd can set `NoNewPrivileges=yes` for another service with `User=` and a filter. The image has no such service that runs `sudo`.

## Files

The image adds these files to Ubuntu. The recipe keeps them in `images/default/rootfs/`, at the same paths.

| Path | Content |
| --- | --- |
| `/usr/local/bin/hopbox-guest` | The guest agent, built from the same checkout for `linux/amd64` (mode 0755). |
| `/usr/lib/systemd/system/hopbox-guest.service` | The unit of the guest agent ([hopbox-guest](/reference/hopbox-guest/index.html.md#systemd-unit)). |
| `/usr/lib/systemd/system/service.d/50-hopbox-no-vsock.conf` | `RestrictAddressFamilies=~AF_VSOCK` and `SystemCallArchitectures=native` for every service. |
| `/usr/lib/systemd/system/hopbox-guest.service.d/60-hopbox-allow-vsock.conf` | `RestrictAddressFamilies=` and `SystemCallArchitectures=` for the guest agent. |
| `/usr/lib/systemd/system-shutdown/hopbox-poweroff` | A hook of `systemd-shutdown` (mode 0755). At a poweroff or a halt, it writes `b` to `/proc/sysrq-trigger`, a reboot. Firecracker has no power-off device, so without it a poweroff leaves the microVM running with a stopped kernel. The hook runs once the root file system is read-only, so the reboot is clean, and the reboot ends Firecracker ([the Firecracker driver](/reference/firecracker-driver/index.html.md#exit-reasons)). `poweroff -ff`, and a direct `reboot(2)` call that powers off, skip `systemd-shutdown` and the hook. They leave the microVM running with a stopped kernel until a stop of the machine. A machine keeps the files of the image that it was made from. So a machine from an image without the hook keeps the old poweroff until it is made again. |
| `/etc/ssh/sshd_config.d/10-hopbox-image.conf` | `HostKey /etc/ssh/ssh_host_ed25519_key`, `PasswordAuthentication no`, and `KbdInteractiveAuthentication no`. |
| `/etc/sudoers.d/90-hopbox-user` | `user ALL=(ALL:ALL) NOPASSWD: ALL` (mode 0440). |
| `/etc/sysctl.d/90-hopbox.conf` | `kernel.modules_disabled = 1` ([Kernel](#kernel)). |
| `/etc/fstab` | `/dev/vda / ext4 defaults,x-systemd.growfs 0 0` |
| `/etc/nginx/conf.d/default.conf` | The nginx site on port 8000. It replaces the site of the nginx.org package on port 80. |
| `/var/www/html/index.html` | The page of nginx. |
| `/etc/profile.d/go-path.sh` | Adds `~/go/bin` to the `PATH` of login shells, where `go install` puts programs. |
| `/etc/default/locale` | `LANG=C.UTF-8` |
| `/etc/hostname`, `/etc/hosts` | The host name `hopbox`. The guest agent replaces it with the name of the machine. |
| `/etc/localtime`, `/etc/timezone` | `Etc/UTC` |
| `/etc/systemd/system-generators/systemd-ssh-generator` | A link to `/dev/null`, which masks the generator. |
| `/usr/share/hopbox/programs.json` | The [program manifest](#program-manifest). |
| `/etc/apt/keyrings/docker.asc`, `/etc/apt/keyrings/nginx.asc`, and the two files in `/etc/apt/sources.list.d/` | The apt repositories of Docker and nginx.org, so `apt upgrade` in a machine updates them. |
| `/home/user/.local/bin/python`, `python3`, and `/home/user/.local/share/uv/python/` | The CPython of uv ([Python](#python)). |

The image leaves out what each machine must make on its own, or what changes from build to build:

| Left out | Made by |
| --- | --- |
| SSH host keys, also the RSA and ECDSA keys of the Ubuntu package | The guest agent makes the ed25519 key at the first boot of each machine ([hopbox-guest](/reference/hopbox-guest/index.html.md#run)). |
| The machine ID: `/etc/machine-id` is empty | systemd, at the first boot of each machine. |
| `/var/lib/systemd/random-seed` | systemd, at the first boot. |
| apt package lists and caches, logs, `/tmp`, and backup files such as `/etc/shadow-` | apt and the services, at their first run. |
| The apt snapshot line of the build | apt in a machine reads the current Ubuntu archive. |
| The files under `/root` that `go` and `gh` write at the version check of the build, such as the device ID of `gh` | `go` and `gh`, at their first run in the machine. |
| The cache of uv in `/home/user/.cache` | uv, at its first run. |

In `/etc/shadow`, the build sets the date of the last password change to the day of the snapshot. So the file is the same on each build.

## Kernel

The image boots with the guest kernel of [Guest kernel](/reference/guest-kernel/index.html.md), with the kernel command line of [Guest kernel](/reference/guest-kernel/index.html.md#boot-requirements). The kernel has module support only for its BPF compiler, and the image has no modules. The boot argument `sysctl.kernel.modules_disabled=1` turns module loading off before the first program of the guest runs. `systemd-sysctl` sets the same value from `/etc/sysctl.d/90-hopbox.conf`, so a boot without the argument also ends with module loading off. The guest agent starts after `systemd-sysctl.service`.

The `modprobe@` services of systemd report that they find no `modprobe` program. The image has no `kmod` package, because it has no modules to load. These services end without an error.

## Disk

| Item | Value |
| --- | --- |
| Layout | One ext4 file system on the whole disk, without a partition table |
| Device in the guest | `/dev/vda`, the root device of Firecracker |
| Label | `hopbox-root` |
| UUID | `2f6e1d4a-7c3b-4e58-9a0d-6b1f3c5e8a72`, the same in every build |
| Block size | 4096 bytes |
| Size | The size of the files, plus a tenth, plus 1 GiB, rounded up to 256 MiB. The build of 2026-10-16 is 3,758,096,384 bytes. |
| Features | The ext4 defaults of `mke2fs` 1.47.2. The inode tables and the journal are written at build time, so a machine does not write them at its first boot. |

The node makes the disk of a machine as a reflink copy of the image disk. Then it grows the copy, for example to 100 GB. At each boot, `systemd-growfs-root.service` grows the file system to the size of the disk, while it is mounted. The `x-systemd.growfs` option in `/etc/fstab` turns this on. A machine sees all of its disk without a step of its own.

## Pins

`images/default/pins.env` holds every version and digest that a build uses. A build with the same pins and the same `hopbox-guest` gives the same disk ([Reproducible build](#reproducible-build)).

| Pin | Example | Description |
| --- | --- | --- |
| `UBUNTU_IMAGE` | `ubuntu:26.04@sha256:f144425f...` | The Ubuntu container image, by the digest of its multi-platform index. |
| `UBUNTU_SNAPSHOT` | `20261016T080000Z` | The time of the snapshot of the Ubuntu archive on `snapshot.ubuntu.com`. It fixes the version of every Ubuntu package. It is also the build time: `SOURCE_DATE_EPOCH`. |
| `DOCKER_GPG_SHA256` | `1500c1f5...` | The sha256 of the signing key of the Docker apt repository. If the key differs, the build fails. |
| `DOCKER_VERSION` | `5:29.9.1-1~ubuntu.26.04~resolute` | The version of `docker-ce` and `docker-ce-cli`. Its last part names the Ubuntu suite. |
| `CONTAINERD_VERSION`, `DOCKER_BUILDX_VERSION`, `DOCKER_COMPOSE_VERSION` | `2.4.2-2~ubuntu.26.04~resolute` | The versions of `containerd.io`, `docker-buildx-plugin`, and `docker-compose-plugin`. |
| `NGINX_GPG_SHA256` | `55385da3...` | The sha256 of the signing key of the nginx.org apt repository. If the key differs, the build fails. |
| `NGINX_VERSION` | `1.30.6-1~resolute` | The version of the package `nginx` of nginx.org. Its last part names the Ubuntu suite. |
| `NODE_VERSION`, `NODE_SHA256` | `24.22.0` | The Node.js release and the sha256 of `node-v<version>-linux-x64.tar.xz`. |
| `GO_VERSION`, `GO_SHA256` | `1.27.2` | The Go release and the sha256 of `go<version>.linux-amd64.tar.gz`. |
| `PYTHON_VERSION` | `3.15.0` | The CPython release that uv installs for `user`. uv checks it against the sha256 that it ships. |
| `UV_VERSION`, `UV_SHA256` | `0.13.1` | The uv release and the sha256 of `uv-x86_64-unknown-linux-gnu.tar.gz`. |
| `GH_VERSION`, `GH_SHA256` | `2.102.0` | The GitHub CLI release and the sha256 of `gh_<version>_linux_amd64.tar.gz`. |
| `NEOVIM_VERSION`, `NEOVIM_SHA256` | `0.12.6` | The Neovim release and the sha256 of `nvim-linux-x86_64.tar.gz`. |
| `RIPGREP_VERSION`, `RIPGREP_SHA256` | `15.2.0` | The ripgrep release and the sha256 of `ripgrep-<version>-x86_64-unknown-linux-musl.tar.gz`. |

Each line of `pins.env` is `NAME=value`. `build.sh` reads the file as data, without a shell. It refuses an unknown name, a missing pin, and a value with a character other than letters, digits, and `.+~:@/_-`. `tools/imagepins` checks each value against a stricter pattern of its own pin, for example `1.2.3` for a release version, before it writes the file. So a value from a release page or a hand edit cannot run as shell code in the build.

`images/default/node-release-keys` lists the fingerprints of the primary keys of the active Node.js releasers, from the Node.js README. The build downloads the keyring of `nodejs/release-keys` and `SHASUMS256.txt.asc` of the release. `gpgv` checks the signature. The signer must be in the list, and the signed file must list `NODE_SHA256`. A change of the releasers is a change of this file by hand.

`make image-pins` runs `tools/imagepins`, which sets each pin to the latest release:

| Pins | Latest release from |
| --- | --- |
| `UBUNTU_IMAGE` | The digest of the same tag, for example `26.04`, from Docker Hub. If Docker Hub has a tag of a later LTS release, for example `28.04`, the tool prints a warning. A move to it is a change of the tag by hand, together with the suites of Docker and nginx. |
| `UBUNTU_SNAPSHOT` | The current hour in UTC, rounded down, for example `20261016T080000Z` at 08:41. The snapshot service serves the archive as it was at that time. |
| Docker | The newest version of each package in the Docker apt repository for the suite of `DOCKER_VERSION`. |
| `NGINX_VERSION` | The newest `nginx` in the stable nginx.org repository for the suite of `NGINX_VERSION`. |
| `NODE_*` | The newest LTS release in `https://nodejs.org/dist/index.json`, with the sha256 from its `SHASUMS256.txt`. |
| `GO_*` | The newest stable release in `https://go.dev/dl/?mode=json`. |
| `PYTHON_VERSION` | The newest stable release of Python 3 on `python.org`. |
| `UV_*`, `GH_*`, `NEOVIM_*`, `RIPGREP_*` | The latest GitHub release of `astral-sh/uv`, `cli/cli`, `neovim/neovim`, and `BurntSushi/ripgrep`, with the sha256 that GitHub records for the archive. uv, the GitHub CLI, and ripgrep also publish a checksum file. The file must list the same sha256. Neovim publishes none. |

The tool refuses a release whose tag is not a version, for example `v2.0;$(id)`. It does not change `DOCKER_GPG_SHA256` and `NGINX_GPG_SHA256`. If Docker or nginx.org changes its signing key, the tool fails, and a person checks the new key.

Run `make image-pins` before the build of an image that machines will use, so the image has the latest releases. Then commit `pins.env`.

## Recipe

| File | What it holds |
| --- | --- |
| `images/default/pins.env` | The [pins](#pins). |
| `images/default/Dockerfile` | Four stages. `base` is the pinned Ubuntu image with apt on the snapshot. `unpack` downloads the release archives, checks each one against its pin, and unpacks them with `bsdtar`. `rootfs` installs the programs and the files of the image, and copies the files of `unpack`. `builder` holds `e2fsprogs`, `bsdtar`, and `zstd`. The stages that unpack use `bsdtar`. GNU tar of Ubuntu 26.04 fails under the amd64 emulation of Docker Desktop on an Arm Mac, with `Cannot open: Function not implemented` for each file in a dir. The first line pins the Dockerfile frontend `docker/dockerfile:1.28.0` by digest. So the meaning of the file does not change with the Docker of the host. |
| `images/default/node-release-keys` | The fingerprints of the Node.js release keys ([Pins](#pins)). |
| `images/default/manifest.sh` | Writes the program manifest inside the root file system. |
| `images/default/rootfs/` | The files that the image adds, at their paths in the image. |
| `images/default/build.sh` | Runs the build and the check in Docker. `make image` and `make check-image` call it. |
| `images/default/finalize.sh` | Runs in the `builder` stage. Turns the root file system into the disk and its zstd copy. |
| `images/default/check-disk.sh` | Runs in the `builder` stage. Reads the disk with `debugfs` and checks the rules of this page. |
| `images/default/boottest/` | The boot test of `make test-image`. |
| `tools/imageartifact` | Writes and checks `artifact.json` ([Artifact format](/reference/artifact-format/index.html.md#imageartifact)). |
| `tools/imagepins` | Moves the pins to the latest releases. |

The build does these steps:

1. It builds `hopbox-guest` for `linux/amd64` with the version of the checkout, as `make build` does. It uses the Go release of the `toolchain` line in `go.mod`, through `GOTOOLCHAIN`, so every host builds the same binary.
2. It builds the `rootfs` stage with `docker buildx` for `linux/amd64` and exports the root file system as a tar file.
3. `finalize.sh` runs in the `builder` stage. It unpacks the tar file inside the container, not in the mounted output dir. A mount from macOS keeps no owners and no extended attributes, such as the file capabilities of `ping`. Then it makes the ext4 disk from the files with `mke2fs -d`. Then it checks the disk with `e2fsck` and compresses it with `zstd -19`.
4. `tools/imageartifact` writes `artifact.json` and prints the digest.

apt checks the signatures of the Ubuntu snapshot. The first `apt-get update` of the build runs before the image has CA certificates, so it does not check the TLS certificate of `snapshot.ubuntu.com`. The signatures still protect every package. Every later step checks TLS.

## Reproducible build

Two builds of the same checkout with the same pins give the same disk, the same compressed copy, and the same digest. These rules make it so:

- The Ubuntu image is pinned by digest, and the Ubuntu packages come from one snapshot of the archive.
- The Docker and nginx packages are pinned by version, and every release archive by its sha256. uv checks the CPython download against the sha256 that the pinned uv ships.
- The Dockerfile frontend is pinned by digest, and `hopbox-guest` is built by the Go release of `go.mod`.
- `mke2fs` reads `SOURCE_DATE_EPOCH`, the time of the snapshot. No timestamp in the file system is later than that time. `mke2fs` adds the files in sorted order.
- The file system has a fixed UUID and a fixed hash seed.
- The image leaves out the files that change on each build ([Files](#files)).
- `zstd` gives the same output for the same input and level.

The image holds `hopbox-guest`, which carries the version and the commit of the checkout. So each commit gives a new digest.

On the tested server, on 2026-10-16, two builds of the recipe of this page gave the same files and the same digest `sha256:354fb76713c333b4d22c0341cab5f9b04b288eb44bd97110f81ca13bfc260e7c`. One build ran with `IMAGE_NO_CACHE=1`, which builds every Docker step again. The two builds ran at the same time. The disk has the sha256 `465579a3e137d8222d1bfde177cca61fb344791812203d628aa09777a1e26162`. These builds held a `hopbox-guest` with the version `v0.0.0-m18dev`. An earlier recipe gave the same result for two builds one after the other.

## Outputs

`make image` writes the artifact to `bin/image/`, or to `IMAGE_OUT`:

| File | What it is |
| --- | --- |
| `disk.raw` | The disk, a sparse file of 3,758,096,384 bytes, with about 2.37 GB of data. |
| `disk.raw.zst` | The zstd copy, about 559 MB. |
| `artifact.json` | The metadata ([Artifact format](/reference/artifact-format/index.html.md#artifactjson)). |

It also writes `bin/linux_amd64/hopbox-guest`. `make bundle` puts `artifact.json` and `disk.raw.zst` into the release bundle, and the install of a node puts them into the image cache of the node ([Release bundle](/reference/release-bundle/index.html.md#the-default-image-in-the-bundle)).

## Make targets

| Target | What it does | Needs |
| --- | --- | --- |
| `image` | Builds `hopbox-guest` for `linux/amd64` and the image, and prints the digest. | Docker with buildx and `linux/amd64` containers, Go |
| `check-image` | Checks the artifact in `IMAGE_OUT`: `artifact.json`, the size and sha256 of each file, and the rules below. | Docker, Go |
| `test-image` | Boots the image under Firecracker and checks it ([Boot test](#boot-test)). | Linux, KVM, sudo, Firecracker, a built kernel and image |
| `image-pins` | Sets `images/default/pins.env` to the latest releases ([Pins](#pins)). | Go, internet access |

| Variable | Default | Description |
| --- | --- | --- |
| `IMAGE_OUT` | `bin/image` | The artifact folder. |
| `IMAGE_NO_CACHE` | empty | Set to `1` in the environment to build every Docker step again, for example to check that the build is reproducible. |
| `KERNEL_OUT` | `bin/kernel` | The folder of the kernel for `test-image`. |
| `FIRECRACKER` | `firecracker` | The Firecracker binary for `test-image`. It must be v1.17. |
| `TEST_DOCKER_IMAGE` | empty | A `docker save` tar file of `hello-world` for `test-image`. Without it, the test skips the Docker check. |

`check-image` reads the disk with `debugfs`, without a mount. It checks these rules. It reports each broken rule, and then it fails:

- The disk has no SSH host key, and `/etc/machine-id` is empty.
- `/usr/local/bin/hopbox-guest` exists with mode 0755, and `hopbox-guest.service` is enabled.
- The drop-in that removes `AF_VSOCK` from services exists.
- `nginx.service` is not enabled, and `systemd-ssh-generator` is masked.
- `/etc/sudoers.d/90-hopbox-user` has mode 0440.
- `/etc/passwd` has `user` with the ID 1000, and no user `ubuntu`.
- The apt snapshot line and the apt package lists of the build are gone.
- The disk holds no device ID of `gh` from the build.
- `user` has the `python3` of uv in `~/.local/bin`.
- `/usr/local/bin` has no `claude`, `codex`, or `opencode`.

## Boot test

`make test-image` runs the Go test in `images/default/boottest` as root. It boots the image the way a node will, with these parts:

- A copy of the disk, grown to 16 GiB. Where the file system has reflinks, the copy is a reflink copy. Elsewhere, it is a sparse copy.
- A tap device with `169.254.0.1/32` and a route to the machine.
- MMDS version 2 with a new link key, and a vsock device.
- A guest link server of `internal/node/guestlink`.

Then it checks the machine through the guest link, SSH, and HTTP:

| Check | What passes |
| --- | --- |
| Guest link | The guest agent authenticates and applies its configuration. |
| Kernel | `uname -r` is `6.1.188-hopbox1`. The kernel has the i8042 keyboard, so `SendCtrlAltDel` works. SELinux denies nothing. Module loading is off. The BPF compiler is on. |
| Boot | `systemctl is-system-running --wait` prints `running`, and `/etc/machine-id` is new. |
| Network | The host name, the address, the default route, the MTU, and `/etc/resolv.conf` match the configuration. |
| Configuration | A login shell has the env of the configuration. `/etc/hopbox/authorized_keys` holds its key. The start command prints its line. |
| Main user | `user` has the ID 1000, the groups `sudo` and `docker`, and sudo without a password. |
| Disk | The root file system fills the 16 GiB disk. |
| Programs | The manifest in the image matches `artifact.json`, and each program runs. In a login shell of `user`, `python` and `python3` are the CPython of uv in the manifest. `/usr/bin/python3` is the Python of Ubuntu. No coding agent is in the `PATH`. |
| Docker | `docker load` of `hello-world` and `docker run --network none hello-world`, without internet access. |
| SSH | The guest agent reports one ed25519 host key. An SSH client that trusts only that key logs in as `user` with the key of the configuration. |
| vsock | An SSH session, `sudo` in it, and a `systemd-run` service cannot open `AF_VSOCK`. A static x86_64 probe that calls the i386 `socket` through `int 0x80` cannot open it either, in each of these places and in exec. nginx runs with the filter and `SystemCallArchitectures=native`, and the guest agent without a filter. The user manager has the filter and `NoNewPrivs: 0`. The guest listens on no vsock port. |
| nginx | Port 8000 is closed. After `sudo systemctl enable --now nginx`, it serves the page. |
| Second machine | A second machine from the same image has another SSH host key and another machine ID. |
| Stop | `SendCtrlAltDel` stops each machine, and Firecracker exits. |

On the tested server, the work folder is on the ext4 root file system, which has no reflinks, so the test makes sparse copies. The XFS file system of `/var/lib/hopbox` has reflinks.

The test writes the serial console of each machine to `console-<n>.log` in its work folder. It reads these variables, which `make test-image` sets:

| Variable | Description |
| --- | --- |
| `HOPBOX_TEST_KVM` | Must be `1`. Otherwise the test skips. |
| `HOPBOX_TEST_FIRECRACKER` | The Firecracker binary. |
| `HOPBOX_TEST_KERNEL` | The guest kernel `vmlinux`. Its sha256 must match `kernel.sha256` of the image. |
| `HOPBOX_TEST_IMAGE` | The artifact folder. The test runs `Verify` on it first. |
| `HOPBOX_TEST_DOCKER_IMAGE` | Optional. A `docker save` tar file of `hello-world`. |
| `HOPBOX_TEST_WORKDIR` | Optional. The folder for the disk copies and the console logs. A reflink copy needs this folder on the file system of the image. The default is a new temporary folder. |
