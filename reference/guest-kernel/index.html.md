# Guest kernel

> The source, config, fragment, builder, outputs, make targets, boot arguments, and security modules of the guest kernel.

The guest kernel boots every machine. It is the Amazon Linux 6.1 microVM kernel, with the Firecracker 6.1 config and a hopbox config fragment. The recipe lives in `kernel/`. Two builds of the same recipe give the same `vmlinux`, byte for byte. `make kernel` builds it in Docker (the build guide of the repo has the steps, and how to change the kernel).

## Files

| File | What it holds |
| --- | --- |
| `kernel/kernel.env` | The pins: the source, the base config, and the build stamp. |
| `kernel/hopbox.config` | The config fragment. It holds every option that hopbox needs. |
| `kernel/Dockerfile` | The builder image. |
| `kernel/build.sh` | Runs the build in the builder image. `make kernel` calls it. |
| `kernel/container.sh` | The build steps inside the builder image. |
| `kernel/vmlinux.config` | The final `.config`. `make kernel` writes it. Do not edit it by hand. |
| `kernel/vmlinux.sha256` | The sha256 of the last `vmlinux` and of its config. `make kernel` writes it. Do not edit it by hand. |
| `tools/kconfigcheck` | Checks that a final config keeps every option of the fragment. |

## Source

| Item | Value |
| --- | --- |
| Kernel version | 6.1.188 |
| Repository | `https://github.com/amazonlinux/linux` |
| Tag | `microvm-kernel-6.1.188-56.393.amzn2023` (tag object `a9ea02bac2b0d3906e5ca32dd91fd0d406eb4951`) |
| Commit | `8785eb28815fdac7e6743bb6fad34ef67dc24b26` |
| Tarball | `https://codeload.github.com/amazonlinux/linux/tar.gz/8785eb28815fdac7e6743bb6fad34ef67dc24b26`, 224,970,436 bytes |
| Tarball sha256 | `11cd54507d034b43f41690045ab26d90c8143f38210e9633432bc8a7c3eded2a` |

The build fetches the tarball by the commit, not by the tag, so a moved tag cannot change the source. If the tarball has another sha256, the build stops. If `make kernelversion` of the source is not the version in `kernel.env`, the build also stops.

GitHub makes the tarball on request. If GitHub changes how it makes archives, the sha256 changes and the build stops. In that case, check the new tarball against a git checkout of the commit before you change the pin.

Firecracker builds its 6.1 test kernels from the `microvm-kernel-*` tags. The tag above is the newest 6.1 microVM tag on 2026-10-15. It is later than 6.1.167, so it has the VMClock generation counter.

## Base config

| Item | Value |
| --- | --- |
| Firecracker version | v1.17.0 |
| File | `resources/guest_configs/microvm-kernel-ci-x86_64-6.1.config` |
| URL | `https://raw.githubusercontent.com/firecracker-microvm/firecracker/v1.17.0/resources/guest_configs/microvm-kernel-ci-x86_64-6.1.config` |
| sha256 | `153ca1b40f3312bfb40b7587b471ea548a915f98f480f0d0892a1537957a2147` |

Firecracker's CI also merges `ci.config` and `nvme.config` into this file. hopbox does not use them. The fragment carries the options of `ci.config` that hopbox needs: the i8042 keyboard controller and `/proc/config.gz`.

## Config fragment

`kernel/hopbox.config` lists every option that hopbox needs, also the options that the base config already sets. If a new base config drops one of them, the build fails. Each option is built in, because the guest has no kernel modules. If Kconfig picks `m` for any option, the build fails too.

| Group | Options | Why |
| --- | --- | --- |
| Release name | `LOCALVERSION="-hopbox1"`, `# LOCALVERSION_AUTO is not set`, `BUILD_SALT="6.1.188-hopbox1"` | `uname -r` prints `6.1.188-hopbox1`, so a hopbox build differs from a stock kernel. The salt goes into the build ID of `vmlinux`. |
| Module support | `MODULES`, `# MODULE_SIG is not set`, `MODPROBE_PATH=""` | `BPF_JIT` needs `MODULES` in 6.1. The build makes no modules, and the guest has no module tree. The kernel never starts a module loader, and the boot argument `sysctl.kernel.modules_disabled=1` turns module loading off before the first program runs ([Boot requirements](#boot-requirements)). Module signing stays off, because a signing key made at build time breaks the reproducible build. |
| No 32-bit system calls | `# IA32_EMULATION is not set`, `# X86_X32_ABI is not set` | Nothing in the default image is 32-bit. A 64-bit process can make i386 system calls through `int 0x80`. Seccomp filters, such as `RestrictAddressFamilies` of systemd, see them as calls of another architecture. Without 32-bit calls, no process can go around a filter that way. A 32-bit program does not run in the guest. |
| Firecracker devices | `VIRTIO_MMIO`, `VIRTIO_BLK`, `VIRTIO_NET`, `HW_RANDOM_VIRTIO`, `SERIAL_8250_CONSOLE`, `KVM_GUEST`, `PTP_1588_CLOCK_KVM` | The disks, the network, the entropy device, the serial console, and the clock of a microVM. |
| Guest link | `VSOCKETS`, `VIRTIO_VSOCKETS` | The guest agent talks to the node over vsock. |
| Copy detection | `ACPI`, `PTP_1588_CLOCK`, `PTP_1588_CLOCK_VMCLOCK`, `VMGENID` | VMClock and VMGenID tell the guest that it is a restored copy. VMClock is an ACPI device. |
| Memory | `MEMORY_HOTPLUG`, `MEMORY_HOTREMOVE`, `CONTIG_ALLOC`, `VIRTIO_MEM`, `MEMORY_BALLOON`, `VIRTIO_BALLOON`, `PAGE_REPORTING` | virtio-mem of Firecracker 1.17, and the balloon with free page reporting. |
| Clean stop | `SERIO`, `SERIO_I8042`, `SERIO_LIBPS2`, `INPUT_KEYBOARD`, `KEYBOARD_ATKBD` | Firecracker's `SendCtrlAltDel` action reaches the guest only through the i8042 keyboard controller. The base config has none of these. |
| Config in the guest | `IKCONFIG`, `IKCONFIG_PROC` | `/proc/config.gz` holds the config, so tools in the guest can check it. |
| systemd | `DEVTMPFS`, `FHANDLE`, `INOTIFY_USER`, `SIGNALFD`, `TIMERFD`, `EPOLL`, `AUTOFS_FS`, `TMPFS_POSIX_ACL`, `TMPFS_XATTR` | systemd in the default image. |
| Default image | `PSI`, `DMIID`, `BINFMT_MISC`, `EXT4_FS`, `EXT4_FS_POSIX_ACL`, `EXT4_FS_SECURITY`, `FUSE_FS`, `TUN` | systemd and the programs of the default image. |
| Isolation | `NAMESPACES`, `UTS_NS`, `IPC_NS`, `PID_NS`, `NET_NS`, `USER_NS`, `POSIX_MQUEUE`, `KEYS`, `SECCOMP`, `SECCOMP_FILTER`, `BPF_SYSCALL`, `CHECKPOINT_RESTORE` | Containers, and the browser sandbox later. |
| BPF compiler | `BPF_JIT`, `BPF_JIT_ALWAYS_ON` | BPF programs, for example the seccomp filters of systemd and Docker, run as native code. `BPF_JIT_ALWAYS_ON` removes the BPF interpreter, which speculative execution attacks use. |
| cgroup v2 controllers | `CGROUPS`, `MEMCG`, `BLK_CGROUP`, `BLK_DEV_THROTTLING`, `CGROUP_SCHED`, `FAIR_GROUP_SCHED`, `CFS_BANDWIDTH`, `CGROUP_PIDS`, `CPUSETS` | The controllers `cpu`, `cpuset`, `io`, `memory`, and `pids` for systemd and Docker. |
| cgroup v2 features | `CGROUP_BPF`, `CGROUP_FREEZER`, `CGROUP_HUGETLB`, `CGROUP_DEVICE`, `CGROUP_CPUACCT`, `CGROUP_PERF`, `CGROUP_NET_PRIO` | Device rules through BPF, freeze, the `hugetlb` controller, and the other cgroup features of Docker. |
| No RT group scheduling | `# CONFIG_RT_GROUP_SCHED is not set` | The base config turns it on. systemd asks for it off, because it blocks the cgroup v2 `cpu` controller while a real-time task runs outside the root cgroup. |
| Docker storage | `OVERLAY_FS` | The overlay storage of Docker. |
| Docker networks | `VETH`, `BRIDGE`, `BRIDGE_NETFILTER`, `VLAN_8021Q`, `BRIDGE_VLAN_FILTERING`, `VXLAN`, `MACVLAN`, `IPVLAN`, `DUMMY`, `IP_ADVANCED_ROUTER`, `IP_MULTIPLE_TABLES` | The bridge, overlay, macvlan, and ipvlan network drivers. |
| Netfilter | `NETFILTER`, `NETFILTER_ADVANCED`, `NETFILTER_NETLINK`, `NF_CONNTRACK`, `NF_CT_NETLINK`, `NF_CONNTRACK_FTP`, `NF_CONNTRACK_TFTP`, `NF_NAT`, `NF_NAT_FTP`, `NF_NAT_TFTP` | Connection tracking and NAT. Docker clears conntrack entries over netlink. |
| nf_tables families | `NF_TABLES`, `NF_TABLES_INET`, `NF_TABLES_IPV4`, `NF_TABLES_IPV6`, `NF_TABLES_BRIDGE`, `NFT_COMPAT` | Ubuntu's `iptables` is `iptables-nft`. It turns each rule into nf_tables expressions, and it runs the other matches through the xtables compat layer. |
| nf_tables expressions | `NFT_CT`, `NFT_NAT`, `NFT_MASQ`, `NFT_REDIR`, `NFT_REJECT`, `NFT_LIMIT`, `NFT_LOG`, `NFT_FIB_INET`, `NFT_FIB_IPV4`, `NFT_FIB_IPV6` | `iptables-nft` turns `-j MASQUERADE` into `NFT_MASQ`, for example. Docker's nftables firewall backend needs the same options. |
| xtables targets | `NETFILTER_XTABLES`, `NETFILTER_XT_MARK`, `NETFILTER_XT_NAT`, `NETFILTER_XT_TARGET_MASQUERADE`, `NETFILTER_XT_TARGET_REDIRECT`, `NETFILTER_XT_TARGET_LOG`, `NETFILTER_XT_TARGET_TCPMSS`, `NETFILTER_XT_TARGET_CHECKSUM` | The targets that `iptables-nft` runs through the xtables compat layer. `LOG`, `TCPMSS`, and `CHECKSUM` are common in the rules of firewalls, VPNs, and libvirt. |
| xtables matches | `NETFILTER_XT_MATCH_ADDRTYPE`, `NETFILTER_XT_MATCH_BPF`, `NETFILTER_XT_MATCH_COMMENT`, `NETFILTER_XT_MATCH_CONNTRACK`, `NETFILTER_XT_MATCH_IPVS`, `NETFILTER_XT_MATCH_MARK`, `NETFILTER_XT_MATCH_MULTIPORT`, `NETFILTER_XT_MATCH_STATE` | The matches that `iptables-nft` runs through the xtables compat layer for Docker. |
| More xtables matches | `NETFILTER_XT_MATCH_STATISTIC`, `NETFILTER_XT_MATCH_PHYSDEV`, `NETFILTER_XT_MATCH_PKTTYPE`, `NETFILTER_XT_MATCH_LIMIT`, `NETFILTER_XT_MATCH_RECENT`, `NETFILTER_XT_MATCH_OWNER` | kube-proxy uses `statistic`, bridge firewalls use `physdev`, and rate limits use `limit` and `recent`. |
| IP sets | `NETFILTER_XT_SET`, `IP_SET`, `IP_SET_HASH_IP`, `IP_SET_HASH_NET` | Rules that match a set of addresses. |
| iptables tables | `IP_NF_IPTABLES`, `IP_NF_FILTER`, `IP_NF_MANGLE`, `IP_NF_NAT`, `IP_NF_RAW`, `IP_NF_TARGET_MASQUERADE`, `IP_NF_TARGET_REDIRECT`, `IP6_NF_IPTABLES`, `IP6_NF_FILTER`, `IP6_NF_MANGLE`, `IP6_NF_NAT`, `IP6_NF_RAW`, `IP6_NF_TARGET_MASQUERADE` | The tables of Docker's rules. Docker 28 and later write rules in the `raw` table, which the base config does not have. |
| IPVS | `IP_VS`, `IP_VS_NFCT`, `IP_VS_PROTO_TCP`, `IP_VS_PROTO_UDP`, `IP_VS_RR` | The service load balancer of Docker swarm mode. |
| Traffic control | `NET_SCHED`, `NET_CLS_ACT`, `NET_SCH_FQ_CODEL`, `NET_SCH_INGRESS`, `NET_CLS_BPF` | systemd sets `fq_codel` as the default queue discipline. The ingress queue discipline and BPF classifiers carry `tc` programs. |
| vsock sockets in `ss` | `VSOCKETS_DIAG` | `ss --vsock` lists the guest link. |
| Encrypted overlay | `XFRM_USER`, `INET_ESP`, `CRYPTO_GCM`, `CRYPTO_SEQIV`, `CRYPTO_GHASH` | Encrypted overlay networks of Docker. |

These options are not in the kernel:

| Option | Why not |
| --- | --- |
| `SECURITY_APPARMOR` | The base config has SELinux and no AppArmor. If the kernel has no AppArmor, Docker runs without an AppArmor profile. |
| `NET_CLS_CGROUP` | Only cgroup v1 uses it. |
| `BTRFS_FS` | Docker uses overlay storage. |
| `IP_SCTP` | No part of hopbox uses SCTP. |
| `CRYPTO_FIPS` | With `MODULES`, it needs `MODULE_SIG`, which stays off. The base config has it. hopbox does not boot guests with `fips=1`. |

The moby script `contrib/check-config.sh` (moby commit `c5c09961ba9aa3db0aacae52b6a54fd7aee3dee4`) reports every "Generally Necessary" option of Docker as enabled in `kernel/vmlinux.config`. Of the optional options, it reports only `SECURITY_APPARMOR`, `NET_CLS_CGROUP`, `IP_SCTP`, and the Docker btrfs and zfs storage drivers as missing.

## Boot requirements

The kernel boots under Firecracker v1.17.0 as an uncompressed ELF `vmlinux`. The VMM driver gives it these kernel command line arguments:

| Argument | Why |
| --- | --- |
| `console=ttyS0` | The serial console. |
| `reboot=k` | A reboot in the guest ends the Firecracker process. With it, `SendCtrlAltDel` stops the microVM cleanly. |
| `panic=1` | A kernel panic reboots, so the Firecracker process ends. |
| `i8042.noaux i8042.nomux i8042.dumbkbd` | Firecracker emulates only a keyboard on its i8042 controller. With these arguments, the driver does not probe for a mouse port or a multiplexer, and does not test the keyboard. This saves tens of milliseconds at boot ([Firecracker actions](https://github.com/firecracker-microvm/firecracker/blob/v1.17.0/docs/api_requests/actions.md)). |
| `sysctl.kernel.modules_disabled=1` | The kernel turns module loading off at boot, before init runs. The default image sets the same value again with `systemd-sysctl`. |
| `memhp_default_state=online_movable` | The kernel puts memory that virtio-mem plugs online at once, in the movable zone, so the memory can be unplugged again. The 6.1 kernel has no Kconfig option for this. |

Firecracker adds `pci=off`, `root=/dev/vda rw` for the root disk, and the `virtio_mmio.device` arguments itself.

Do not pass `i8042.nopnp`, although the Firecracker docs list it with the three arguments above. Firecracker 1.17 describes the controller in its ACPI tables, as the device `PNP0303` with its interrupt. The driver finds the controller there. With `i8042.nopnp`, the driver uses the legacy interrupt 1, which a Firecracker guest with ACPI does not have. The probe then fails with `i8042: probe of i8042 failed with error -22`, and `SendCtrlAltDel` does nothing. Without the argument, the kernel logs `serio: i8042 KBD port at 0x60,0x64 irq 30`, and `SendCtrlAltDel` stops the default image in about one second. The boot test of the default image checks both ([Default image](/reference/default-image/index.html.md#boot-test)).

## Security modules

The base config builds in SELinux, Landlock, and the capability module. `/sys/kernel/security/lsm` in the guest lists `capability,landlock,selinux`. SELinux stays inactive:

- The default image has no SELinux policy and no tool that loads one.
- Without a policy, SELinux denies nothing. `/sys/fs/selinux/enforce` reads `0`.
- The kernel command line needs no `selinux=0`.

Docker and systemd work as on a kernel without SELinux. The kernel has no AppArmor ([Config fragment](#config-fragment)).

## Builder image

`kernel/Dockerfile` builds the image `hopbox-kernel-builder:<first 12 hex digits of the sha256 of kernel/Dockerfile>` for `linux/amd64`. A change to the Dockerfile gives a new tag. The image has the label `hopbox-build=kernel-builder`. `kernel/build.sh` builds the image only when it is missing or has no such label. On each run, it tags the image again with the same name, which gives the image a new tag time on the classic image store of Docker. The cleanup of the CI runner keeps an image with a recent tag time. Then the script runs the image by its ID.

| Item | Value |
| --- | --- |
| Base image | `debian:bookworm-slim@sha256:7c7b2c966bc9ee8cedfeef67e0e279108992c77681fa595db4a9d65c06ccc587` |
| Package source | `snapshot.debian.org` at `20261005T000000Z`, for `bookworm`, `bookworm-updates`, and `bookworm-security`. This is the snapshot that the base image comes from. |
| Packages | `bc`, `bison`, `ca-certificates`, `cpio`, `curl`, `flex`, `gcc`, `libc6-dev`, `libelf-dev`, `libssl-dev`, `make`, `perl` |
| Compiler | gcc 12.2.0 (`gcc-12 12.2.0-14+deb12u1`) |
| Binutils | 2.40 (`binutils 2.40-2`) |

The image digest and the snapshot fix every package version. apt checks the signatures of the snapshot, so the snapshot uses plain HTTP.

## Reproducible build

The build runs in a new container each time, from a fresh copy of the source in `/build/linux`. The build path ends up in the kernel image, so it is the same on every host. `container.sh` sets these values:

| Variable | Value | Effect |
| --- | --- | --- |
| `SOURCE_DATE_EPOCH` | `1791339570` | 2026-10-07T02:19:30Z, the committer date of the source commit. |
| `KBUILD_BUILD_TIMESTAMP` | `Wed Oct  7 02:19:30 UTC 2026` | The build date in `uname -v`. `container.sh` derives it from `SOURCE_DATE_EPOCH`. |
| `KBUILD_BUILD_USER` | `hopbox` | The user in `/proc/version`. |
| `KBUILD_BUILD_HOST` | `hopbox` | The host in `/proc/version`. |
| `KBUILD_BUILD_VERSION` | `1` | The build number `#1` in `uname -v`. |
| `LC_ALL`, `TZ` | `C`, `UTC` | Tools sort and print dates the same way on every host. |

The guest sees:

```text
$ uname -a
Linux localhost 6.1.188-hopbox1 #1 SMP PREEMPT_DYNAMIC Wed Oct  7 02:19:30 UTC 2026 x86_64 GNU/Linux
```

## Outputs

`make kernel` writes these files to `bin/kernel/`, or to `KERNEL_OUT`:

| File | What it is |
| --- | --- |
| `vmlinux` | The kernel, an uncompressed x86_64 ELF image of about 45 MB. |
| `vmlinux.sha256` | The sha256 of `vmlinux` and of `vmlinux.config`, in `sha256sum` format. |
| `vmlinux.config` | The final `.config`. |
| `toolchain.txt` | The versions of gcc, binutils, and the build packages. |
| `dl/` | The downloaded tarball and base config. Later builds reuse them after a sha256 check. |

It also copies `vmlinux.config` and `vmlinux.sha256` to `kernel/`, where git tracks them. `make bundle` puts `vmlinux` into the release bundle, and it refuses a `vmlinux` with another sha256 than the one in `kernel/vmlinux.sha256` ([Release bundle](/reference/release-bundle/index.html.md)). The current values are:

| File | sha256 |
| --- | --- |
| `vmlinux` | `dfb03703bab6d4b590964f2d9473fdc82994ae94384cc5ed1c1b4a0430118a91` |
| `vmlinux.config` | `99f0171775432d9239ed83f453811e09a496ffd88301feec3b9eb8924faf9ebc` |

## Make targets

| Target | What it does | Needs |
| --- | --- | --- |
| `kernel` | Runs `make kernel-config` first, so a dropped option fails before the compile. Then builds `vmlinux` and writes the outputs. | Docker |
| `kernel-config` | Writes only the final config, without a compile. Then runs `kconfigcheck`. | Docker |
| `check-kernel` | Runs `make kernel`. If `kernel/vmlinux.config` or `kernel/vmlinux.sha256` then differs from git, it fails. This proves that the build is reproducible on your host. | Docker |
| `check-kernel-config` | Runs `make kernel-config`. If `kernel/vmlinux.config` then differs from git, it fails. | Docker |

| Variable | Default | Description |
| --- | --- | --- |
| `KERNEL_OUT` | `bin/kernel` | The output folder. |
| `JOBS` | the number of CPUs in the container | The number of parallel compile jobs. Set it in the environment, for example `JOBS=4 make kernel`. It does not change the output. |

## Checks

- `kconfigcheck` reads the fragment and a final config. If the final config does not keep an option of the fragment, it fails and lists each such line. If a dependency is missing, Kconfig drops an option without an error, so this check is the only guard.
- If the final config has an option set to `m`, `container.sh` fails. With `MODULES`, Kconfig can pick `m`. The build makes no modules, so such an option is then missing from the guest.
- `make test` runs two tests in `tools/kconfigcheck`. One checks that `kernel/vmlinux.config` keeps the fragment. The other checks that `kernel/vmlinux.sha256` names the sha256 of `kernel/vmlinux.config`, so a config change without a new build fails.
- The CI job `kernel` runs `make check-kernel-config`. It catches a fragment or pin change without a new config. It keeps `bin/kernel/dl` in the Actions cache. The cache key is the hash of `kernel/kernel.env`. So a run downloads the source tarball only after a pin change. CI does not compile the kernel, because the compile takes many minutes. On two threads of the [tested server](/reference/preflight/index.html.md#the-tested-server), it takes about 15 minutes. The workflow `image.yml` compiles the kernel on request.
