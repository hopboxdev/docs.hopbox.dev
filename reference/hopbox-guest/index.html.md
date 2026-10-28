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
| `guest link up` | info | `version`, `node_version` | The node accepted the handshake. |
| `guest link down, reconnecting` | warn | `err` | A link that was up for at least 1 s dropped. The agent opens a new link at once. |
| `guest link failed` | warn | `err`, `retry_in` | A link did not come up, or dropped within 1 s. The agent waits `retry_in` before the next try. The wait starts at 100 ms and doubles after each failure, up to 5 s. |
| `ignored an older configuration` | warn | `generation`, `applied` | The node pushed a configuration with a lower generation than the one that the agent applied. |
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

1. It writes the sshd drop-in. If the image has no ed25519 SSH host key, it creates one. Each machine then has its own host key.
2. Under systemd, it tells systemd that it is ready. sshd starts after this step, so sshd always has a host key and the drop-in.
3. It reads the link key from the metadata service, dials the node, and does the handshake.
4. It applies each configuration that the node pushes, and starts the start command as [Start command](#start-command) says.
5. After a drop of the link, it reads the key again and opens a new link at once.

On `SIGTERM` or `SIGINT`, it kills the start command and exits with status 0.

The agent must run as root. Root can set the host name and the network, write the files below, and switch users for exec. Outside Linux, the host name and the network setup fail with an error, and the rest works.

Example, on a local unix socket with a local metadata service:

```sh
hopbox-guest run --link=unix:///tmp/box1.sock_10240 --metadata-url=http://127.0.0.1:8169
```

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

The agent reads `/etc/passwd` and `/etc/group` for the user ID, the groups, the home directory, and the login shell of each user. It reports every `/etc/ssh/ssh_host_*_key.pub` key to the node.

Example of `/etc/hopbox/env`:

```sh
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
| A restart of the agent, for example after a crash | systemd stops the start command with the agent. The new agent starts it again, as for a new boot. |

When the agent stops, it kills the start command and its children.

## systemd unit

The default image installs the unit `images/default/rootfs/usr/lib/systemd/system/hopbox-guest.service` at `/usr/lib/systemd/system/hopbox-guest.service`, with the binary at `/usr/local/bin/hopbox-guest`. It enables the unit. [Default image](/reference/default-image/index.html.md) lists the other files of the image.

| Setting | Value | Reason |
| --- | --- | --- |
| `Type` | `notify` | The agent sends `READY=1` after the SSH host key exists. |
| `ExecStart` | `/usr/local/bin/hopbox-guest run` | The default flags fit a Firecracker machine. |
| `Restart` | `always`, after 1 s | The node cannot reach the machine without the agent. |
| `KillMode` | `control-group` | A restart of the agent stops the start command and exec. The new agent runs the start command again. |
| `DefaultDependencies` | `no` | The agent sets up the network itself, so it starts before `network-pre.target`. |
| Order | after `local-fs.target` and `systemd-sysctl.service`. Before `network-pre.target`, `ssh.service`, `sshd.service`, and `ssh.socket`. | The values of `/etc/sysctl.d`, such as `kernel.modules_disabled`, hold before the agent runs a command. sshd starts with a host key. |
| `WantedBy` | `multi-user.target` | The agent runs in every normal boot. |
| `RestrictAddressFamilies`, `SystemCallArchitectures` | empty, from the drop-in `hopbox-guest.service.d/60-hopbox-allow-vsock.conf` | The image removes `AF_VSOCK` from every other service. The agent dials the node over vsock, so its drop-in undoes that ([Default image](/reference/default-image/index.html.md#no-vsock-for-services)). The commands that the agent starts keep this freedom. |

## Exit status

| Status | Meaning |
| --- | --- |
| 0 | The command succeeded, or `run` stopped on a signal. |
| 1 | The command failed, for example on a bad `--link` address. |
| 80 | The command line is not valid, for example an unknown command or flag, or a bad flag value. |
