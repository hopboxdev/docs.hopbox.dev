# hopbox-guest

> The commands, flags, files, systemd unit, logs, and exit status of the guest agent hopbox-guest.

`hopbox-guest` is the guest agent of hopbox. It runs inside every machine. Images ship it. Users and operators do not install it.

The guest agent keeps the guest link to the node open ([Guest link](/reference/guest-link/index.html.md)). It applies the configuration that the node pushes, runs the start command, and runs exec and file copy for the node.

## Usage

```sh
hopbox-guest [--log-level=LEVEL] <command> [flags]
```

When you run `hopbox-guest` without a command, it prints its help and an error, and exits with status 80.

## Global flags

A global flag goes before or after the command.

| Flag | Type | Default | Description |
| --- | --- | --- | --- |
| `--log-level` | level | `info` | The lowest level of the log records that `hopbox-guest` writes. The values are `debug`, `info`, `warn`, and `error`. Case does not matter. A value can also carry an offset, for example `info+2`, as in Go's `log/slog`. |
| `-h`, `--help` | bool | `false` | Prints the help of `hopbox-guest`, or of the command that it follows, and exits. |

## Logs

`hopbox-guest` writes its logs to standard error, in the same JSON format as `hopboxd` ([hopboxd](/reference/hopboxd/index.html.md#logs)). Under systemd, standard error goes to journald. The output of the start command does not go to these logs. It goes over the guest link to the log of the machine.

| Message | Level | Fields | When |
| --- | --- | --- | --- |
| `prepare the SSH host keys` | error | `err` | At the start, the agent did not write the sshd drop-in, make the host key, or read the host keys. The agent goes on with the steps that worked. |
| `created the SSH host key` | info | | The image had no ed25519 SSH host key, so the agent made one. |
| `tell systemd that the agent is ready` | warn | `err` | The ready notice to systemd failed. |
| `guest agent started` | info | `link`, `metadata_url`, `interface`, `user` | The agent starts its guest link. |
| `the commands of the agent run in their own slice, so the agent can freeze them for a save` | info | `slice` | The guest runs systemd with cgroup v2. The image has `hopbox.slice` and `hopbox-user.slice`, and runs each kept unit in `hopbox.slice`. The agent names the feature `clone` ([Freeze of the user processes](#freeze-of-the-user-processes)). |
| `the guest has no cgroup v2, so the agent cannot freeze the user processes for a save` | warn | `err` | `/sys/fs/cgroup/cgroup.controllers` is missing. The agent names no feature `clone`. |
| `cannot find the binary of the agent for the gate of its commands` | warn | `err` | The agent cannot read the path of its own binary. It names no feature `clone`. |
| `cannot read the units of the freeze, so the agent cannot freeze the user processes for a save` | error | `slice`, `err` | systemd did not answer, also on a second try 1 s later. The agent names no feature `clone`. |
| `the image cannot freeze the user processes for a save` | warn | `reason` | The image has no unit `hopbox.slice` or `hopbox-user.slice`, or a kept unit runs in another slice. The reason names it. The agent names no feature `clone`. |
| `cannot stop the commands of an earlier run of the agent` | error | `slice`, `err` | The stop of `hopbox-user.slice` at the start of the agent failed. |
| `cannot read the cgroups of an earlier freeze` | error | `err` | The list of the cgroups at the start of the agent failed. |
| `thawed the user processes that an earlier run of the agent froze, and recorded the broken freeze` | warn | `cgroups` | An earlier run of the agent froze the user processes and ended before a thaw. The new run never holds such a freeze, and the thaw of the node reports it as broken. |
| `froze the user processes for a save` | info | `cgroups`, `took` | The node sent `Freeze`. |
| `a part of the freeze of the user processes failed` | error | `err` | A write of `cgroup.freeze` failed. The agent sends the error to the node. |
| `cgroups did not freeze in time, so their processes go on during the save` | warn | `cgroups`, `wait` | A cgroup did not report the freeze within 2 s. Its tasks still stop before they return to user code. |
| `thawed the user processes` | info | `cgroups` | The node sent `Thaw` after a freeze. |
| `thawed the user processes by itself, because no thaw came` | warn | `ticks`, `cgroups` | A source ran 60 sleeps of 1 s after the freeze or the last new link, and no `Thaw` came. The agent records it in `/run/hopbox/thawed-early`, so the freeze check fails. A loaded copy never thaws by itself. |
| `the freeze of the user processes did not hold until the thaw` | error | `err` | The freeze check at a `Thaw` failed. The agent sends the reason to the node in `broken`. |
| `cannot record the restart during a freeze` | error | `err` | A new run of the agent found a freeze, and did not write `/run/hopbox/thawed-early`. |
| `cannot thaw the user processes that an earlier run of the agent froze` | error | `err` | A write of `cgroup.freeze` at the start of the agent failed. |
| `the guest has no VMClock generation counter, so the agent cannot freeze the user processes for a save` | warn | `err` | `/dev/vmclock0` is missing, or it has no generation counter. The agent names no feature `clone`. |
| `a part of the thaw of the user processes failed` | error | `err` | A write of `cgroup.freeze` failed. |
| `ran the clone identity step` | info | `machine_id`, `boot_id`, `took` | The node sent `CloneIdentity` ([Clone identity step](#clone-identity-step)). |
| `the clone identity step failed` | error | `err` | A part of the step failed. The agent sends the error to the node, and the node ends the clone. |
| `guest link up` | info | `version`, `node_version` | The node accepted the handshake. |
| `guest link down, reconnecting` | warn | `err` | A link that was up for at least 1 s dropped. The agent opens a new link at once. |
| `guest link failed` | warn | `err`, `retry_in` | A link did not come up, or dropped within 1 s. The agent waits `retry_in` before the next try. The wait starts at 100 ms and doubles after each failure, up to 5 s. |
| `ignored an older configuration` | warn | `generation`, `applied` | The node pushed a configuration with a lower generation than the one that the agent applied. |
| `ran the randomness step` | info | `bytes` | The node sent entropy for a later load of a saved state. The agent gave it to the kernel with `RNDADDENTROPY`, and forced a new seed with `RNDRESEEDCRNG` ([Guest link](/reference/guest-link/index.html.md#clock-and-randomness)). |
| `the randomness step failed` | error | `err` | One of the two ioctls failed. The agent sends the error to the node, and goes on. |
| `set the clock from the node` | info | `jump` | The guest clock was more than 1 s off the clock of the node, so the agent set it. `jump` is the clock of the node minus the old guest clock, in nanoseconds. A positive value moved the clock forward. |
| `cannot set the clock from the node` | error | `jump`, `err` | The set of the clock failed. The agent sends the error to the node. |
| `configuration applied` | info | `generation` | The agent applied a configuration. |
| `configuration failed` | error | `generation`, `err` | The agent did not apply a configuration. It sends the error to the node. |
| `left an env var out of the machine` | warn | `err` | An env var of the configuration breaks the rules of [Files](#files). |
| `ignored a guest link message that this agent does not know` | warn | | The node sent a message of a newer protocol. |
| `ignored an attach with a bad ID` | warn | `size` | An `Attach` message has an ID that is not 16 bytes long. |
| `ignored an attach that this agent does not know` | warn | | An `Attach` message names an operation of a newer protocol. |
| `exec: attach failed` | warn | `err` | The agent did not send the attach ID on a new `Exec` call. |
| `exec: send the exit` | warn | `err` | The agent did not send the exit of a command to the node. |
| `upload: attach failed` | warn | `err` | The agent did not send the attach ID on a new `Upload` call. |
| `upload: the node ended the call` | warn | `path`, `err` | The node ended an upload before its end. |
| `upload failed` | warn | `path`, `err` | The agent did not write an upload. It sends the error to the node. |
| `upload: send the result` | warn | `err` | The agent did not send the result of an upload to the node. |
| `download: attach failed` | warn | `err` | The agent did not send the attach ID on a new `Download` call. |
| `download failed` | warn | `path`, `err` | The agent did not read or send a download. |
| `the boot of the guest did not finish within <duration>, so the start command waits` | warn | | The boot did not reach `multi-user.target` within 2 min. The status of the start command holds the same text. |
| `wait for the end of the boot` | warn | `err` | The check of `multi-user.target` failed. Unless the agent stops, the start command then starts. |
| `start command started` | info | `pid`, `run` | The start command started. `run` counts the runs of this boot. |
| `start command failed to start` | error | `err` | The agent did not start the start command, for example for an unknown main user. The status of the start command is `FAILED`. |
| `start command wait` | warn | `err` | The wait for the end of the start command failed with an error other than its exit status. |
| `start command exited` | info | `code`, `signal` | The start command ended. |

## Commands

| Command | Description |
| --- | --- |
| `run` | Runs the guest agent until it gets `SIGTERM` or `SIGINT`. |
| `version` | Prints the version of `hopbox-guest`. See [the version command](/reference/version/index.html.md). |
| `exec-gate` | Hidden. Only the agent runs it, as the first step of each new command ([exec-gate](#exec-gate)). |
| `freeze-watch` | Hidden. Only the agent runs it, at each freeze ([freeze-watch](#freeze-watch)). |

### run

```sh
hopbox-guest run [--link=ADDR] [--metadata-url=URL] [--interface=NAME] [--user=NAME]
```

| Flag | Type | Default | Description |
| --- | --- | --- | --- |
| `--link` | address | `vsock://2:10240` | The address of the node. `vsock://CID:PORT` dials vsock. `unix:///path` dials a unix socket, for tests and local runs. |
| `--metadata-url` | URL | `http://169.254.169.254` | The base URL of the metadata service that holds the link key. If the host of the URL is an IPv4 link-local address, the agent first routes it on `--interface`. |
| `--interface` | name | `eth0` | The network interface that the network configuration sets up. The agent also reaches the metadata service through it. |
| `--user` | name | `user` | The main Linux user. The start command runs as this user. If the node names no user, exec and file copy use this user. The group of this user can read the env file. |

The command does these steps:

1. It checks for systemd, cgroup v2, the VMClock generation counter, the units `hopbox.slice` and `hopbox-user.slice`, and the kept units in `hopbox.slice`. With all of them, the agent runs each command in `hopbox-user.slice` and names the feature `clone` ([Freeze of the user processes](#freeze-of-the-user-processes)).
2. It writes the sshd drop-in. If the image has no ed25519 SSH host key, it creates one. Each machine then has its own host key.
3. Under systemd, it tells systemd that it is ready. sshd starts after this step, so sshd always has a host key and the drop-in.
4. With the slice, it stops `hopbox-user.slice`, so the commands of an earlier run end. An earlier run can end during a freeze. The agent then first records the broken freeze and thaws its cgroups ([Freeze of the user processes](#freeze-of-the-user-processes)).
5. It reads the link key from the metadata service, dials the node, and does the handshake.
6. On a link of version 4 or later, it sets the guest clock from the node when the two differ by more than 1 s. When the node sends entropy, it first runs the randomness step ([Guest link](/reference/guest-link/index.html.md#clock-and-randomness)).
7. It applies each configuration that the node pushes, and starts the start command as [Start command](#start-command) says.
8. After a drop of the link, it reads the key again and opens a new link at once.

On `SIGTERM` or `SIGINT`, it kills the start command and exits with status 0.

The agent must run as root. Root can set the host name, the network, and the clock, reseed the kernel, write the files below, and switch users for exec. Outside Linux, the host name, the network setup, the set of the clock, and the randomness step fail with an error, and the rest works.

Example, on a local unix socket with a local metadata service:

```sh
hopbox-guest run --link=unix:///tmp/box1.sock_10240 --metadata-url=http://127.0.0.1:8169
```

### exec-gate

```sh
hopbox-guest exec-gate -- <path> <argv0> [<arg>...]
```

The agent runs this command as the first step of the start command and of each exec, in place of the command itself. The gate has the user, the working directory, the env, the process group, and the terminal of the command. It waits on file descriptor 3 for one byte from the agent. In that time, the agent moves the gate into a new scope of `hopbox-user.slice`. Then the gate runs `<path>` in its own place, with the argv `<argv0> <arg>...`. So the command keeps the PID of the gate, and no process of the command runs outside its scope. If the pipe ends without the byte, the gate runs nothing, and exits with status 126. A run of `exec-gate` by hand waits for that byte, and exits with status 126 too.

The gate runs as the user of the command, and the kernel can dump it while it waits. So with `kernel.yama.ptrace_scope` 0, a process of the same user can trace it. That gives the user nothing new, because the main user has sudo.

### freeze-watch

```sh
hopbox-guest freeze-watch [--generation=N]
```

| Flag | Type | Default | Description |
| --- | --- | --- | --- |
| `--generation` | integer | `-1` | The VMClock generation counter at the freeze. `-1` means a guest without the counter. |

The agent runs this command at each freeze, in its own cgroup, with the read end of a pipe as file descriptor 3. At the thaw, the agent writes one byte, and the watcher exits. An end of the pipe without the byte means that the agent died. In a source, the watcher then writes `/run/hopbox/thawed-early`, and thaws each frozen cgroup, with `init.scope` last, so systemd restarts the agent. In a loaded copy, a guest whose counter now reads another value, it does nothing. A counter that it cannot read counts as a copy.

## Files

The agent writes these files in the guest. Each write replaces the file in one rename, so a reader never sees a part of a file.

| Path | Mode | Owner | Content |
| --- | --- | --- | --- |
| `/etc/hostname` | 0644 | root | The host name, from the configuration. |
| `/etc/hosts` | 0644 | root | The agent points the line of `127.0.1.1` at the host name, as Debian and Ubuntu do. It keeps every other line. |
| `/etc/resolv.conf` | 0644 | root | One `nameserver` line for each resolver of the configuration. The file replaces a symlink at this path, for example the link of systemd-resolved. |
| `/etc/hopbox/env` | 0640 | root, group of the main user | The env vars and secrets of the machine, one `export NAME='value'` line each, in POSIX shell syntax. Only root and the group of the main user can read it, because it holds secrets. The agent checks each name with the rules of `domain.ValidateEnvVarName` ([the env var API](/reference/vars-api/index.html.md#names-and-values)), and each value for a NUL byte. A value that fails stays out of the file, and out of the env of exec and the start command. The agent logs a warning for it, and writes the other values. |
| `/etc/profile.d/hopbox.sh` | 0644 | root | A script. If the user can read `/etc/hopbox/env`, the script reads it. Every login shell runs the script, so the env vars and secrets reach every login shell. It holds no values. |
| `/etc/hopbox/authorized_keys` | 0644 | root | The keys of the configuration, for example the key of the SSH front door. Only root can change it, so no user can remove the key of the front door. |
| `/etc/ssh/sshd_config.d/hopbox.conf` | 0644 | root | `AuthorizedKeysFile .ssh/authorized_keys .ssh/authorized_keys2 /etc/hopbox/authorized_keys`. The first two are the default of Ubuntu. sshd then reads the keys of the configuration for every Linux user, next to the keys of each user. Ubuntu reads this folder at the top of `sshd_config`, and sshd keeps the first value of a setting, so this value wins. At each start, the agent restores this content. |
| `/etc/ssh/ssh_host_ed25519_key` | 0600 | root | The SSH host key. If the image has no such key, the agent writes it. Otherwise it keeps the key. |
| `/etc/ssh/ssh_host_ed25519_key.pub` | 0644 | root | The public part of that key. |
| `/etc/machine-id` | 0444 | root | A new machine ID, from the clone identity step. A file `/var/lib/dbus/machine-id` gets it too. A link there stays as it is. |
| `/run/hopbox/boot_id` | 0644 | root | A new boot ID, from the clone identity step. A bind mount shows it at `/proc/sys/kernel/random/boot_id`. |
| `/run/hopbox/thawed-early` | 0644 | root | The record of a thaw by itself, or of an end of the agent, since the last freeze. The freeze check fails while it exists, and the next freeze removes it. |

The agent reads `/etc/passwd` and `/etc/group` for the user ID, the groups, the home directory, and the login shell of each user. It reports every `/etc/ssh/ssh_host_*_key.pub` key to the node.

Example of `/etc/hopbox/env`:

```text
# Written by hopbox-guest from the env vars and secrets of the machine.
# hopbox-guest rewrites this file at each configuration.
export DATABASE_URL='postgres://db1/app'
export GREETING='it'\''s here'
```

## Kernel changes

The agent makes these changes through netlink. They need Linux.

| Change | When |
| --- | --- |
| The kernel host name. | The configuration has a host name. |
| The interface comes up and gets its MTU. The configured address becomes its only IPv4 address. The default route goes through the gateway, on-link. | The configuration has a network. |
| The configured IPv6 address becomes the only global IPv6 address of the interface. The IPv6 default route goes through the IPv6 gateway, such as `fe80::1`. | The network has `address6` ([Guest link](/reference/guest-link/index.html.md)). |
| The interface comes up and gets a route to the metadata service. If it has no IPv4 address, it gets `169.254.0.2/32`. | The metadata service has an IPv4 link-local address. The change comes before each read of the link key. |
| `1` or `0` in `cgroup.freeze` of each cgroup of a freeze. | A `Freeze` or a `Thaw` of the node, a thaw by itself, and the start of the agent ([Freeze of the user processes](#freeze-of-the-user-processes)). |
| The interface gets the MAC address of the new machine, and its neighbor entries go. The bind mount of the new boot ID. | The clone identity step ([Clone identity step](#clone-identity-step)). |

## Start command

The start command runs with `/bin/sh -c` as the main user, in the home directory of that user. If the home directory does not exist, it runs in `/`. Its env holds two groups of values. A value of the second group replaces a value of the first.

1. The login basics: `HOME`, `USER`, `LOGNAME`, `SHELL`, `PATH`, and `LANG`. `LANG` comes from `/etc/default/locale`, with the default `C.UTF-8`.
2. The env vars and secrets of the machine.

Two conditions must hold before the start command starts:

- The agent applied its first configuration without an error, so the env file is in place.
- systemd reports `multi-user.target` as active, so the services of the image, such as Docker, run. Until then, the state of the start command is `WAITING`. Without systemd, the agent does not wait. After 2 min of waiting, the agent logs a warning once and puts it in the `error` field of the status.

The agent sends each line of its output to the node, and reports its exit. The agent starts the start command in these cases only:

| Case | What happens |
| --- | --- |
| The first configuration of a boot | The start command starts, after the wait above. |
| A new configuration, or a new link | Nothing. The start command keeps its state. |
| `RunStartCommand` from the node | The start command starts again, unless it runs. The node sends it for the restart policy "always". |
| A restart of the agent, for example after a crash | The start command stops: systemd stops `hopbox-user.slice` with the agent, and the new agent stops it at its start. The new agent starts the command again, as for a new boot. |

When the agent stops, it kills the start command and its children.

## systemd unit

The default image installs the unit `images/default/rootfs/usr/lib/systemd/system/hopbox-guest.service` at `/usr/lib/systemd/system/hopbox-guest.service`, with the binary at `/usr/local/bin/hopbox-guest`. It enables the unit. [Default image](/reference/default-image/index.html.md) lists the other files of the image.

| Setting | Value | Reason |
| --- | --- | --- |
| `Type` | `notify` | The agent sends `READY=1` after the SSH host key exists. |
| `ExecStart` | `/usr/local/bin/hopbox-guest run` | The default flags fit a Firecracker machine. |
| `Restart` | `always`, after 1 s | The node cannot reach the machine without the agent. |
| `KillMode` | `control-group` | A restart of the agent ends each helper process of the agent. The start command and exec run in `hopbox-user.slice`, outside the cgroup of the agent. |
| `DefaultDependencies` | `no` | The agent sets up the network itself, so it starts before `network-pre.target`. |
| Order | after `local-fs.target` and `systemd-sysctl.service`. Before `network-pre.target`, `ssh.service`, `sshd.service`, and `ssh.socket`. | The values of `/etc/sysctl.d`, such as `kernel.modules_disabled`, hold before the agent runs a command. sshd starts with a host key. |
| `WantedBy` | `multi-user.target` | The agent runs in every normal boot. |
| `Slice` | `hopbox.slice` | A freeze of the user processes spares the agent ([Freeze of the user processes](#freeze-of-the-user-processes)). |
| `hopbox-user.slice` | the unit `images/default/rootfs/usr/lib/systemd/system/hopbox-user.slice`, with `PartOf=hopbox-guest.service` | A stop or a restart of the agent stops the slice, and with it the start command and each exec. The agent also stops the slice at its start, for a restart after a crash. |
| `RestrictAddressFamilies`, `SystemCallArchitectures` | empty, from the drop-in `hopbox-guest.service.d/60-hopbox-allow-vsock.conf` | The image removes `AF_VSOCK` from every other service. The agent dials the node over vsock, so its drop-in undoes that ([Default image](/reference/default-image/index.html.md#no-vsock-for-services)). The commands that the agent starts keep this freedom. |

## Freeze of the user processes

Stock Firecracker resumes each process at a load. So before a save for a fork, a snapshot, or a checkpoint, the node sends `Freeze`. The agent then freezes the user processes with `cgroup.freeze` of cgroup v2, and systemd itself too ([Guest link](/reference/guest-link/index.html.md#freeze-and-clone-identity-step)). The owner decided the freeze on 2026-11-06, as Q178, and the freeze of systemd on 2026-11-14, as Q256.

The agent runs each command in a scope of its own, in `hopbox-user.slice`. The scope has a name of the form `hopbox-cmd-<pid>-<random>.scope`. The gate `exec-gate` holds each new command until the agent placed it in its scope through the private socket of systemd ([exec-gate](#exec-gate)). So a freeze of the slice never freezes the agent.

The image runs the units that a save and a later login need in `hopbox.slice`, through a drop-in `70-hopbox-keep.conf` with `Slice=hopbox.slice` for each: `systemd-journald.service`, `dbus.service`, `dbus-broker.service`, `systemd-logind.service`, `systemd-udevd.service`, and `ssh.service`. The unit of the agent sets `Slice=hopbox.slice` too. `hopbox.slice` has the dependencies of `system.slice`, so journald and udevd start in it before `sysinit.target`, and it stops last at a shutdown.

At a `Freeze`, the agent does these steps in this order:

1. It removes `/run/hopbox/thawed-early`, and reads the generation counter of the VMClock from `/dev/vmclock0`.
2. It starts its watcher, `hopbox-guest freeze-watch` ([freeze-watch](#freeze-watch)).
3. It freezes `init.scope`, the cgroup of systemd, and waits for `frozen 1`. From then on, no unit starts or stops, no timer fires, and no login gets a session. Each of them waits for the thaw.
4. It freezes each top-level cgroup but `hopbox.slice`, and each child cgroup of `hopbox.slice` but the kept units, by their exact names. So `system.slice`, `user.slice`, `machine.slice`, each slice that a user adds, `hopbox-user.slice`, a unit with `Slice=hopbox.slice`, and each `hopbox-*.slice` freeze too.
5. It lists the cgroups again, and freezes each new one, until a pass finds none. More than 5 passes is an error.
6. It waits for `frozen 1` of each cgroup. One budget of 2 s covers the steps 3 to 6.
7. It checks the root cgroup. A process there that is no kernel thread is an error that names its PID and its name.

A cgroup that does not report `frozen 1` in 2 s keeps `1` in its `cgroup.freeze`. Its tasks are inside a kernel call, and the kernel stops each of them before it returns to user code. So such a cgroup runs no user code, and the save only waits no longer for it. An error of a step fails the save. If the freeze of `init.scope` fails, nothing else freezes, because a running systemd must never meet a frozen slice. While the freeze holds, the agent starts no command: the start command and each exec wait for the thaw, apart from the loop of the link, which goes on to take the `Thaw`.

At a `Thaw`, the agent first runs the freeze check. The check passes only with all of these conditions:

- A `Freeze` of this run of the agent started the freeze.
- `/run/hopbox/thawed-early` does not exist.
- Each frozen cgroup still reads `1`, `init.scope` included.
- No new cgroup exists outside the freeze.
- The root cgroup holds only kernel threads.
 The check freezes a new cgroup that it finds, and names it. Its failure goes to the node in `broken`, and the save fails. Then the agent thaws each user cgroup, and `init.scope` last. It thaws `init.scope` only after a pass in which each other write worked and no other cgroup still reads frozen. It tries 3 passes. After that, systemd stays frozen, and the error goes to the node, so a later `Thaw` can try again.

A source can get no `Thaw`. It then thaws by itself after 60 sleeps of 1 s from the freeze or from its last new link. Each sleep counts once, also one across a jump of the guest clock. The agent records such a thaw in `/run/hopbox/thawed-early`. A loaded copy, a guest whose VMClock generation counter changed since the freeze, never thaws by itself. Only the `Thaw` of the node ends its freeze (Q257). A guest without `/dev/vmclock0` names no feature `clone`.

A frozen systemd restarts no agent that died. So at an end of the agent during a freeze, its watcher acts. In a source, it records the end in `/run/hopbox/thawed-early`, and ends the whole freeze: it thaws each frozen cgroup, with `init.scope` last. systemd then restarts the agent, and the restart stops `hopbox-user.slice` through `PartOf=`, as at each restart of the agent. In a loaded copy, it does nothing, and the copy stays frozen until the node ends it.

The rule: a running systemd never meets a frozen slice. systemd 259 starts a unit with `clone3` in the mode of `vfork`, right into the cgroup of the unit. In a frozen slice, the new process stops before its `exec`, and PID 1 waits for it in the kernel. So a thaw of `init.scope` alone holds systemd at its next start of a unit in a frozen slice, such as a timer. For the same reason, a new run of the agent never holds a freeze. A new run starts only with systemd thawed, so only in a source. If it finds frozen cgroups, it writes the record, so the thaw of the node reports a broken freeze, and thaws them, with `init.scope` last. Then it stops `hopbox-user.slice` through systemd, as without a freeze.

To see the scope of a command, run in the machine:

```sh
cat /proc/self/cgroup
```

The output has the form `0::/hopbox.slice/hopbox-user.slice/hopbox-cmd-2817-9f3a1c22.scope`.

A guest agent from an image without `hopbox.slice` and `hopbox-user.slice` names no feature `clone`. Its commands run in the cgroup of the agent, so it cannot freeze them, and the machine gets no fork, snapshot, or checkpoint.

### What the freeze guarantees

From the answer to `Freeze` until the `Thaw` of the node, no process of the guest runs user code. systemd stands still too. The exceptions are the guest agent and five services: journald, D-Bus, logind, udev, and sshd, each with the helpers that it starts in its own cgroup. In a clone, the node sends the thaw only after the clone identity step, the randomness step, and the first configuration. So no program of the user runs in a clone before the clone has its new identity and fresh kernel randomness. A clone whose step fails ends with a kill, and its programs never run.

### What the freeze does not guarantee

- A program keeps what it holds in memory: keys, tokens, a `machine-id` or a host name that it read before, and the state of its own random number generator. OpenSSL, Go, Python, and Java each keep such a state. After the thaw, each clone goes on from the same state. Only randomness that a program takes from the kernel after the step is fresh. A program can read the VMClock generation counter to learn that it is a copy.
- systemd and the kept services are copies too. They keep the old `machine-id`.
- The kernel works on for a frozen process. It answers on open TCP connections, and a kernel WireGuard device sends on. The node drops each packet with the address of the source until the step.
- A helper of a kept service runs during the freeze. Examples are a udev rule with `RUN`, a PAM module of sshd, and a D-Bus service without a systemd unit. A helper of the kernel, such as `core_pattern`, runs too.
- The user is root in the machine. A user who moves a process into a kept cgroup, or who changes the agent, defeats the freeze. The freeze protects against accidents, not against the owner of the machine.
- A source whose node is lost thaws by itself after 60 s. A source whose agent dies during a freeze thaws at once. A save of such a freeze fails.
- A hibernate and a move do not freeze (Q178).

## Clone identity step

A fork, and a new machine from a snapshot, start from the memory of another machine. On each link of such a load, until the step is done, the node sends entropy in `Sync`, then `CloneIdentity` ([Guest link](/reference/guest-link/index.html.md#freeze-and-clone-identity-step)). The user processes and systemd are still frozen from the save. The step first runs the freeze check of [Freeze of the user processes](#freeze-of-the-user-processes). A failed check fails the step, with `the freeze of the save did not hold, so user processes can have run before the step`. The agent runs each part of the step, also after another part failed:

1. It sets the MAC address of `--interface` to the address of the new machine, and drops the neighbor entries of the interface. The old entries name the tap device of the source.
2. It writes a new random `/etc/machine-id`.
3. It removes `/var/lib/systemd/random-seed`.
4. It writes a new random boot ID to `/run/hopbox/boot_id`, and bind-mounts it over `/proc/sys/kernel/random/boot_id`. A later step writes the file in place. The next hello of the agent names the new boot ID.
5. It removes each SSH host key, makes a new ed25519 key, and sends the new keys in its status. Then it sends `SIGHUP` to the sshd listener, the process of `hopbox.slice/ssh.service` whose parent is PID 1, so sshd loads the new key. The step calls no part of systemd, because systemd is frozen. With no listener, sshd reads the new key at its start through `ssh.socket`.

The `Config` that follows sets the new host name and the new address. After the step and that `Config`, the node sends `Thaw`, and the user processes run with the new identity. A failed part ends the step with an error, and the node ends the clone with its processes still frozen.

Two parts reach only some processes:

- The bind mount of the boot ID holds in the mount namespace of the agent, the root one. A Docker container reads `boot_id` from its own `/proc`, so it reads the boot ID of the source.
- A daemon that read `machine-id` before the save keeps the old value, such as systemd itself, journald, and D-Bus. So journald writes on to the dir of the old ID in `/var/log/journal`. A program that reads the file after the step gets the new ID. Every clone holds a copy of the keys and the tokens that the programs of its source hold in RAM.

## Exit status

| Status | Meaning |
| --- | --- |
| 0 | The command succeeded, or `run` stopped on a signal. |
| 1 | The command failed, for example on a bad `--link` address. |
| 80 | The command line is not valid, for example an unknown command or flag, or a bad flag value. |
