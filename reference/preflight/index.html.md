# The preflight check

> The preflight checks of a server, their severities, their errors, the tested OS and kernel, and the tested server.

The preflight check tests that a server can run machines. `hopboxd install` runs it on a server with the node role, before it switches the release ([hopboxd](/reference/hopboxd/index.html.md#install)). `hopboxd enroll` runs it on a server with the node role, before it contacts the control role ([Enrollment and the install PKI](/reference/enrollment/index.html.md#hopboxd-enroll)). On a server with the node role, the `channel` unit of `hopboxd run` runs it again at its first heartbeat. The heartbeats report the result ([the control channel](/reference/control-channel/index.html.md#the-heartbeat)). A server with only the DNS role runs no machine. So its enrollment and its heartbeats skip the preflight check. `hopboxd preflight` runs it on demand. The code is in `internal/node/preflight`.

## Severities

| Severity | A failure means |
| --- | --- |
| `fail` | The node cannot run a microVM. The check stops an enrollment, and the heartbeats report the status `degraded`. |
| `warn` | hopbox runs, with a higher risk. An enrollment goes on, and the status stays `ok`. The problem still appears in the heartbeats. |

Every check runs, also after a failure, so one run lists every problem. Each check has at most 10 seconds.

## The tested host

hopbox tests one operating system and one host kernel ([Host kernel](/reference/host-kernel/index.html.md)):

| Item | Value |
| --- | --- |
| Operating system | Debian 13: `ID=debian` and `VERSION_ID=13` in `/etc/os-release` |
| Kernel release | `6.18.57-hopbox1`, as `uname -r` prints it |

### The tested server

The tested server is the server on which hopbox measures its times and runs its acceptance suite. It is an OVHcloud dedicated server:

| Item | Value |
| --- | --- |
| CPU | Intel Xeon E5-1650 v4, with 12 threads |
| Operating system | Debian 13 |
| Host kernel | `6.18.57-hopbox1` ([Host kernel](/reference/host-kernel/index.html.md)) |

A page that gives a time or a result "on the tested server" gives a number of this server. Another server can give other numbers.

## Checks

The checks run in the order of this table.

| Check | Severity | What it does |
| --- | --- | --- |
| `kvm` | `fail` | Opens `/dev/kvm` for reading and writing, and asks for the KVM API version with the ioctl `KVM_GET_API_VERSION`. The version must be 12. |
| `cgroup2` | `fail` | Checks with `statfs` that `/sys/fs/cgroup` is a cgroup v2 file system, the unified hierarchy that the jailer needs. |
| `tun` | `fail` | Opens `/dev/net/tun` for reading and writing. |
| `nftables` | `fail` | Opens a netlink socket for nftables and lists the tables, with a bound of 10 s. This needs the kernel module `nf_tables` and the capability `CAP_NET_ADMIN`. |
| `reflink` | `fail` | Makes a missing `data_dir`. Then it writes a test file of 64 KiB in `data_dir` and clones it with the ioctl `FICLONE`. It removes both files after the test. The test files are `.preflight-reflink-<random>` and `.preflight-reflink-<random>.clone`. |
| `kernel` | `warn` | Compares the operating system and the kernel release with the tested host. |

The checks need root, or rights that most users do not have, such as the group `kvm` and `CAP_NET_ADMIN`. Run them as root.

### Errors of the checks

| Check | Error | Cause |
| --- | --- | --- |
| `kvm` | `/dev/kvm does not exist. Turn on virtualization (VT-x or AMD-V) in the firmware, and load the kvm_intel or kvm_amd module` | The CPU or the kernel gives no KVM. |
| `kvm` | `open /dev/kvm: <error>. hopboxd must run as root or in the group kvm` | The user cannot open the device. |
| `kvm` | `KVM_GET_API_VERSION on /dev/kvm: <error>` | The ioctl failed. |
| `kvm` | `/dev/kvm has the KVM API version <n>, not 12` | A KVM that hopbox does not know. |
| `cgroup2` | `statfs /sys/fs/cgroup: <error>` | `/sys/fs/cgroup` does not exist. |
| `cgroup2` | `/sys/fs/cgroup is not cgroup v2. The jailer needs the unified cgroup hierarchy: boot without systemd.unified_cgroup_hierarchy=0` | The host runs cgroup v1 or a hybrid hierarchy. |
| `tun` | `/dev/net/tun does not exist. Load the tun module` | The kernel has no tun device. |
| `tun` | `open /dev/net/tun: <error>` | The user cannot open the device. |
| `nftables` | `open a netlink socket for nftables: <error>` | The socket failed. |
| `nftables` | `list the nftables tables: <error>. The kernel needs nf_tables, and hopboxd needs CAP_NET_ADMIN` | The kernel has no `nf_tables`, or the user lacks the capability. |
| `reflink` | `create the data dir: <error>` | `data_dir` is missing, and `hopboxd` cannot make it. |
| `reflink` | `create a test file in <dir>: <error>`, `write a test file in <dir>: <error>`, or `sync a test file in <dir>: <error>` | `data_dir` is not writable, or the disk is full. |
| `reflink` | `FICLONE in <dir>: <error>. The data dir needs a file system with reflinks, such as XFS made with reflink=1` | The file system of `data_dir` has no reflinks, for example ext4 or tmpfs. |
| `kernel` | `this host runs <name> with the kernel <release>, and hopbox tests only Debian 13 with the kernel 6.18.57-hopbox1. hopbox runs, with a higher risk` | Another operating system or kernel. `<name>` is `PRETTY_NAME` of `/etc/os-release`, or its `ID` and `VERSION_ID`. |
| `kernel` | `read os-release: <error>` or `uname: <error>` | `hopboxd` cannot read the host. It reads `/etc/os-release`. Without that file, it reads `/usr/lib/os-release`. |

### Off Linux

hopbox runs machines only on Linux with KVM. On another operating system, such as macOS, every check of severity `fail` fails with `hopbox runs machines only on Linux with KVM, and this host runs <os>`. The `kernel` check reports the name of the operating system and its kernel release. So a real `hopboxd enroll` on a Mac fails, as it must.

## hopboxd preflight

```sh
hopboxd preflight [--config=PATH] [--json]
```

`preflight` runs every check on this server and prints the result. It changes nothing, except for the test files of the `reflink` check, which it removes, and a missing `data_dir`, which it makes.

| Flag | Type | Default | Description |
| --- | --- | --- | --- |
| `--config` | path | `/etc/hopbox/hopboxd.toml` | The bootstrap config. `preflight` reads its `data_dir` for the `reflink` check ([the bootstrap config of hopboxd](/reference/hopboxd-config/index.html.md#keys)). |
| `--json` | bool | `false` | Prints one JSON object instead of the lines. |

The text output has one line for each check. A line has the word `preflight`, a space, and the name of the check. Then come a tab and `ok`, or a tab, the severity, a tab, and the error:

```text
preflight <check>	ok
preflight <check>	<warn|fail>	<error>
```

The JSON object has these keys:

| Key | Type | Description |
| --- | --- | --- |
| `passed` | bool | Whether every check of severity `fail` passed. |
| `checks` | array | One object for each check, in order. |
| `checks[].name` | string | The name of the check, such as `kvm`. |
| `checks[].severity` | string | `fail` or `warn`. |
| `checks[].passed` | bool | Whether the check passed. |
| `checks[].message` | string | The error of a failed check, or an empty string. |

| Exit status | Meaning |
| --- | --- |
| 0 | No check of severity `fail` failed. Warnings can still appear. |
| 1 | A check of severity `fail` failed. `hopboxd` prints `hopboxd: error: the preflight check failed, so this server cannot run machines` to standard error. A config that does not load also gives status 1. |

Example on the tested server:

```text
$ sudo hopboxd preflight
preflight kvm	ok
preflight cgroup2	ok
preflight tun	ok
preflight nftables	ok
preflight reflink	ok
preflight kernel	ok
```

Example on a Mac:

```text
$ ./bin/darwin_arm64/hopboxd preflight --config /tmp/hb/hopboxd.toml
preflight kvm	fail	hopbox runs machines only on Linux with KVM, and this host runs darwin
preflight cgroup2	fail	hopbox runs machines only on Linux with KVM, and this host runs darwin
preflight tun	fail	hopbox runs machines only on Linux with KVM, and this host runs darwin
preflight nftables	fail	hopbox runs machines only on Linux with KVM, and this host runs darwin
preflight reflink	fail	hopbox runs machines only on Linux with KVM, and this host runs darwin
preflight kernel	warn	this host runs darwin with the kernel 27.0.0, and hopbox tests only Debian 13 with the kernel 6.18.57-hopbox1. hopbox runs, with a higher risk
hopboxd: error: the preflight check failed, so this server cannot run machines
```

## In hopboxd enroll

With the node role in the config, `hopboxd enroll` runs the checks. It prints the same lines to standard error. A failed check of severity `fail` stops it before it contacts the control role, with this error:

```text
hopboxd: error: the preflight check failed, so this server cannot run machines:
  kvm: /dev/kvm does not exist. Turn on virtualization (VT-x or AMD-V) in the firmware, and load the kvm_intel or kvm_amd module
```

The error has one indented line for each failed check of severity `fail`. Warnings do not stop the enrollment.

## In hopboxd install

With the node role in `--roles`, `hopboxd install` runs the checks after it makes the dirs of the data dir. It prints the same lines to standard error. A failed check of severity `fail` stops the install with the error of `hopboxd enroll`. The install then has not changed the config, the current release, or the units. Before the checks, the install also refuses a data dir that is not on XFS with reflinks ([Server layout](/reference/server-layout/index.html.md#the-file-system-of-the-data-dir)).

## In the heartbeats

On a server with the node role, the `channel` unit runs the checks once, at the first heartbeat after `hopboxd run` starts. Each failed check, of either severity, becomes one problem line `<check>: <error>` in the heartbeats. A failed check of severity `fail` makes the status `degraded` ([the control channel](/reference/control-channel/index.html.md#the-heartbeat)). `sudo hopboxd admin server get <id>` shows both ([hopboxd](/reference/hopboxd/index.html.md#admin-server-get)). A fix of the host shows after the next restart of `hopboxd`.

## Tests

| Test | Where it runs | What it checks |
| --- | --- | --- |
| `TestReport`, `TestChecksOrder`, `TestParseOSRelease`, `TestHostTested` | `make test` | The report, the order of the checks, and the tested host, with injected checks. |
| `TestChecksFailOffLinux` | `make test` on a Mac | Every check of severity `fail` fails off Linux. |
| `TestLinuxChecks` | `make test-linux`, as root in a privileged container | `cgroup2`, `tun`, and `nftables` pass. On a host with `/dev/kvm`, such as the CI runner, `kvm` passes. On a host without it, `kvm` reports the missing device. |
| `TestReflinkRefusesAFileSystemWithoutReflinks` | `make test-linux` | The `reflink` check fails on a tmpfs and leaves no file. |
| `TestReflinkPasses` | by hand, with `HOPBOX_TEST_REFLINK_DIR` | The `reflink` check passes in the dir that the variable names, such as a dir on XFS with `reflink=1` on the tested server. Without the variable, the test skips. |

The tests of `hopboxd enroll` swap the checks, so they run on a Mac too.
