# Host kernel

> The build, the kernel configuration, and the install of the tested host kernel, with the scripts in build/hostkernel/.

This page covers the host kernel of hopbox servers: its build from kernel.org sources, its config, and its install. The host kernel is the Linux kernel of the server that runs Firecracker. The tested kernel is `6.18.57-hopbox1` on Debian 13. Firecracker supports the host kernel 6.18 but not 6.12, the kernel of Debian 13, so hopbox builds 6.18 from kernel.org sources.

## Files

| File | What it holds |
| --- | --- |
| `build/hostkernel/build.sh` | The build script. It downloads the source, checks it, and builds Debian packages. |
| `build/hostkernel/install.sh` | The install script. It installs the kernel package and makes it the GRUB default. |
| `build/hostkernel/config-6.18.57-hopbox1` | The final kernel config. It is the same file as `/boot/config-6.18.57-hopbox1` on the [tested server](/reference/preflight/index.html.md#the-tested-server). |

## Pinned values

Each script pins its versions and checksums in variables at its top. To move to a new kernel release, change these values, `config-<release>`, and this page in one change.

| Variable | Script | Value | Meaning |
| --- | --- | --- | --- |
| `KERNEL_VERSION` | `build.sh` | `6.18.57` | The kernel.org release. It was the longterm 6.18 release on 2026-10-15. |
| `KERNEL_SHA256` | `build.sh` | `5bd9acdc82ec5efc3790ae840e5aaa40dfde7efff261cf6dde902dc570acf98c` | The sha256 of `linux-6.18.57.tar.xz`. |
| `LOCAL_VERSION` | `build.sh` | `-hopbox1` | The suffix of the kernel release. `uname -r` prints `6.18.57-hopbox1`. |
| `GREGKH_FPR` | `build.sh` | `647F28654894E3BD457199BE38DBBDC86092693E` | The key of Greg Kroah-Hartman. It signs the tarball. |
| `AUTOSIGNER_FPR` | `build.sh` | `B8868C80BA62A1FFFAF5FDA9632D3A06589DA6B1` | The key of the kernel.org checksum autosigner. It signs `sha256sums.asc`. |
| `DEBIAN_PACKAGE` | `build.sh` | `linux-image-6.18.15+deb13-amd64` | The Debian package that holds the base config. It comes from trixie-backports. |
| `DEBIAN_DEB_SHA256` | `build.sh` | `d38a7af6260782a47cd615ad717347a528b45f076245b4deabb3f1ecf8459186` | The sha256 of the Debian package file. |
| `KERNEL_RELEASE` | `install.sh` | `6.18.57-hopbox1` | The kernel release that the script installs. |
| `PACKAGE_VERSION` | `install.sh` | `6.18.57-1` | The version of the Debian package that `build.sh` writes. |

## Environment variables of build.sh

| Variable | Default | Meaning |
| --- | --- | --- |
| `WORK_DIR` | `$PWD/hostkernel-build` | The folder for the downloads, the source tree, and the packages. It needs about 40 GB of free space. |
| `JOBS` | the output of `nproc` | The number of parallel `make` jobs. |
| `ALLOW_CONFIG_DRIFT` | `0` | With `1`, the build goes on after `make olddefconfig` changes the config. |

## Requirements

- An amd64 host with Debian 13.
- A normal user for `build.sh`. The build needs no root.
- The build packages in the `BUILD_DEPS` list of `build.sh`. If one is missing, `build.sh` prints the `apt-get install` command and stops.
- Root for `install.sh`.
- UEFI Secure Boot off. The kernel has no Secure Boot signature.

These are the build packages:

```sh
sudo apt-get install bc bison build-essential cpio debhelper dwarves flex gawk gnupg \
  kmod libdw-dev libelf-dev libncurses-dev libssl-dev python3 rsync xz-utils zstd
```

## Build

```sh
build/hostkernel/build.sh
```

The script takes these steps:

1. It downloads `linux-6.18.57.tar.xz`, `linux-6.18.57.tar.sign`, and `sha256sums.asc` from `cdn.kernel.org`.
2. It checks the tarball against `KERNEL_SHA256`, and it checks that `sha256sums.asc` lists the same sha256.
3. It gets the kernel.org keys over WKD into a temporary GnuPG home. The autosigner key must sign `sha256sums.asc`, and the key of Greg Kroah-Hartman must sign the tarball.
4. It unpacks the source into `$WORK_DIR/linux-6.18.57`.
5. It copies `config-6.18.57-hopbox1` to `.config` and runs `make olddefconfig`. If the config changes, the script prints the change with `scripts/diffconfig` and stops.
6. It runs `make bindeb-pkg` and writes `SHA256SUMS.debs`.

The build writes these files into `$WORK_DIR`:

| File | What it holds |
| --- | --- |
| `linux-image-6.18.57-hopbox1_6.18.57-1_amd64.deb` | The kernel and its modules. `install.sh` installs only this package. |
| `linux-image-6.18.57-hopbox1-dbg_6.18.57-1_amd64.deb` | The debug symbols, about 1 GB. The config keeps `DEBUG_INFO`, as Debian does. |
| `linux-headers-6.18.57-hopbox1_6.18.57-1_amd64.deb` | The headers for modules outside the kernel tree. hopbox does not use them. |
| `linux-libc-dev_6.18.57-1_amd64.deb` | The kernel headers for user programs. hopbox does not use them. |
| `SHA256SUMS.debs` | The sha256 of each package. |

On the tested server, the build took 48 minutes with 12 jobs, Debian gcc 14.2.0-19, and pahole 1.30.

## Config

The config starts from the Debian 6.18.15 config. `build.sh config` makes `config-6.18.57-hopbox1` again from that base:

1. It downloads the Debian package with `apt-get download` and checks it against `DEBIAN_DEB_SHA256`. If the archive no longer has the package, put the file in `$WORK_DIR/dl/` by hand. The script then uses that file.
2. It copies `/boot/config-6.18.15+deb13-amd64` from the package to `.config` and runs `make olddefconfig`.
3. It applies the changes below with `scripts/config` and runs `make olddefconfig` again.
4. It writes the result to `build/hostkernel/config-6.18.57-hopbox1` and prints the difference to the Debian config.

These changes are on purpose:

| Option | Debian 6.18.15 | 6.18.57-hopbox1 | Reason |
| --- | --- | --- | --- |
| `LOCALVERSION` | `""` | `"-hopbox1"` | The release name shows a hopbox build. `LOCALVERSION_AUTO` stays off. |
| `MODULE_SIG` | `y` | `n` | hopbox does not sign modules. Unsigned modules on a kernel with `MODULE_SIG=y` mark the kernel as tainted at each load. |
| `MODULE_SIG_ALL` | `y` | not set | It depends on `MODULE_SIG`. |
| `SECURITY_LOCKDOWN_LSM` | `y` | `n` | It selects `MODULE_SIG`. `MODULE_SIG` cannot go off while this option is on. |
| `SECURITY_LOCKDOWN_LSM_EARLY` | `y` | `n` | It depends on `SECURITY_LOCKDOWN_LSM`. |
| `MODULE_SIG_KEY`, `SYSTEM_TRUSTED_KEYS`, `SYSTEM_REVOCATION_KEYS` | not in the file | `""` | The build makes no signing key and embeds no key. Kconfig then drops `MODULE_SIG_KEY` and `SYSTEM_REVOCATION_KEYS`. |

Kconfig also drops the options that depend on the options above, such as `CRYPTO_FIPS`, the other `MODULE_SIG_*` options, and `LOCK_DOWN_KERNEL_FORCE_*`.

`make olddefconfig` on the build host makes these changes:

- Rust is off, because the tested server has no `rustc` and no `bindgen`. The Rust drivers and `ANDROID_BINDER_IPC` drop out.
- The options that exist only in the Debian kernel patches drop out, such as `LOCK_DOWN_IN_EFI_SECURE_BOOT` and `SECURITY_PERF_EVENTS_RESTRICT`.
- The options that are new in 6.18.16 to 6.18.57 take their defaults, such as `CXL_RAS=y` and `UNWIND_USER=y`.
- `ACPI_PLATFORM_PROFILE` changes from `m` to `y`.

A different toolchain on the build host can change the config. For example, a host with `rustc` turns Rust on. Then `build.sh` stops at step 5 of the build.

The config has these features that hopbox needs:

| Feature | Options |
| --- | --- |
| KVM | `CONFIG_KVM=m`, `CONFIG_KVM_INTEL=m`, `CONFIG_KVM_X86=m` |
| vsock to the guest | `CONFIG_VHOST_VSOCK=m`, `CONFIG_VHOST=m`, `CONFIG_VSOCKETS=m` |
| tap devices | `CONFIG_TUN=m` |
| cgroup v2 | `CONFIG_CGROUPS=y`, `CONFIG_MEMCG=y`, `CONFIG_BLK_CGROUP=y`, `CONFIG_CGROUP_PIDS=y`, `CONFIG_CPUSETS=y`, `CONFIG_CGROUP_BPF=y` |
| nftables | `CONFIG_NF_TABLES=m`, `CONFIG_NF_TABLES_INET=y`, `CONFIG_NFT_NAT=m`, `CONFIG_NFT_MASQ=m` |
| The egress of the machines ([the egress of the machines](/reference/egress/index.html.md)) | `CONFIG_NFT_HASH=m`, `CONFIG_NFT_REJECT=m`, `CONFIG_NFT_REJECT_INET=m` |
| XFS with reflink | `CONFIG_XFS_FS=m`. Reflink has no option in 6.18. `xfs.ko` always has it. |
| btrfs | `CONFIG_BTRFS_FS=m` |
| WireGuard | `CONFIG_WIREGUARD=m` |

## Install

```sh
sudo build/hostkernel/install.sh /path/to/hostkernel-build
```

The argument is the `WORK_DIR` of the build. Its default is `$PWD/hostkernel-build`. The script takes these steps:

1. It checks that it runs as root on Debian 13 amd64.
2. It checks the image package against `SHA256SUMS.debs`.
3. It installs the image package with `dpkg -i`. The package makes `/boot/initrd.img-6.18.57-hopbox1` and runs `update-grub`.
4. It writes `/etc/default/grub.d/20_hopbox_kernel.cfg` and runs `update-grub` again.
5. It checks that `/boot/grub/grub.cfg` has the entry of the new kernel. If the entry is missing, the script removes the file of step 4, runs `update-grub`, and stops with an error.

The script does not reboot. Reboot by hand.

The file of step 4 pins the GRUB default by the ID of the menu entry. GRUB names each entry after the UUID of the root file system:

```sh
GRUB_DEFAULT="gnulinux-advanced-<root-uuid>>gnulinux-6.18.57-hopbox1-advanced-<root-uuid>"
```

A Debian kernel upgrade through `linux-image-amd64` does not change this default. The script does not edit `/etc/default/grub`.

After the reboot, these commands show a good host:

```sh
uname -r                      # 6.18.57-hopbox1
cat /proc/sys/kernel/tainted  # 0
ls /dev/kvm /dev/vhost-vsock
```

On the tested server, SSH answered 187 to 319 seconds after the reboot command.

## Fallback

The Debian kernels stay installed. They are in the "Advanced options" menu of GRUB.

To boot a Debian kernel once, run `sudo grub-reboot '<entry ID>'` and reboot. On the tested server, the GRUB environment block is on mdraid, and GRUB cannot clear `next_entry` at boot. `grub-reboot` prints a warning about this. After the reboot, clear it by hand:

```sh
sudo grub-editenv /boot/grub/grubenv unset next_entry
```

To go back to the Debian default for good, remove the pin and update GRUB:

```sh
sudo rm /etc/default/grub.d/20_hopbox_kernel.cfg
sudo update-grub
```
