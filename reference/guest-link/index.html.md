# Guest link

> The transport, link key, handshake, versions, messages, exec, and file copy of the guest link, and its Go API on the node.

The guest link is the channel between a node and the guest agent `hopbox-guest` in one machine. Each link has a version and uses authentication. The link carries the configuration of the machine, the status of the guest, the output of the start command, exec, and file copy.

The definitions are in `proto/hopbox/guest/v1/link.proto`, package `hopbox.guest.v1`. The node side is the Go package `internal/node/guestlink`. The guest side is the Go package `internal/guest`. The package `internal/guest/protocol` holds the constants and the proofs that both sides share.

## Transport

| Item | Value |
| --- | --- |
| Socket | vsock, a socket type between a virtual machine and its host |
| Direction | The guest agent dials the host. The node never dials the guest. |
| Host context ID | `2` |
| Port | `10240` |
| Host side | The unix socket `<uds>_10240`. `<uds>` is the path of the vsock device socket of the microVM. Firecracker gives each guest connection to port 10240 to this socket, and the node listens on it. `protocol.SocketPath(uds)` returns the path. |
| Protocol | Connect over HTTP/2 without TLS, with prior knowledge (h2c). The guest agent is the HTTP/2 client. The node is the HTTP/2 server. |
| Compression | none. The node never compresses its messages with gzip. So the receive buffer of a call holds at most 256 KiB of the input of an exec, not a compressed copy of many MiB. A call of the guest agent in gzip fails with `unimplemented` and the message `unknown compression "gzip"`. |
| Keepalive | After 15 s without frames, each side sends an HTTP/2 PING. If the answer takes more than 5 s, the side drops the connection. |
| Largest data in one message | 32 KiB |
| Largest message to the node | 96 KiB. The node reads no larger message, also before the handshake. It checks the length in the header of each message, ends the call with `resource_exhausted`, and does not read the body. |
| Largest message to the guest agent | 4 MiB. The node refuses a configuration that does not fit. |
| Calls at once on one connection | 64 |
| HTTP/2 receive buffers | 256 KiB for each call, 16 MiB for each connection, on both sides. All 64 calls fit in the buffer of the connection at once. So a call whose reader stalls holds only its own buffer, and the other calls on the connection go on. |
| Connections at once | 8. Any process in the guest can connect, also a user without root. So a connection that carries no authenticated `Link` and no accepted attach within `HandshakeTimeout` (10 s) closes. When 8 connections are open, a new connection closes the oldest connection that did not prove the key. If all 8 proved the key, the node closes the new connection. |
| Idle connections | A connection without calls closes after 2 min. The `Link` call keeps the connection of the guest agent busy. |

Each microVM has its own vsock device socket, so the socket tells the node which machine connects. The guest agent opens a new link at once after a drop. A Firecracker snapshot resets every vsock connection, also in the running machine. After a failed attempt, the guest agent waits 100 ms, then twice as long after each further failure, up to 5 s. A link that ends within 1 s counts as a failed attempt, so a link that drops at once does not loop without a pause.

A paused guest answers no PING. So before a pause, the node closes the link and its socket ([the node executor](/reference/executor/index.html.md#standby)). A resume gives the guest a new link key and a new socket. The guest agent sees the old connection end when its vCPUs run again, and opens a new link at once. The first exec after a resume thus waits for that handshake.

## Link key

Each boot of a machine has its own link key: 32 random bytes. The node puts the key in the metadata service of the microVM, and keeps the same key for the handshake. The key never goes on the kernel command line.

| Item | Value |
| --- | --- |
| Metadata service | Firecracker MMDS version 2, at `http://169.254.169.254` by default |
| Path | `/hopbox/guest_link_key` |
| Value | the key as 64 lowercase hex characters, in a JSON string |
| MMDS data that the node writes | `{"hopbox": {"guest_link_key": "<64 hex characters>"}}` |

The guest agent reads the key before each handshake, because a clone gets a new key. MMDS version 2 needs a session token:

1. `PUT /latest/api/token` with the header `X-metadata-token-ttl-seconds: 60`. The body of the answer is the token.
2. `GET /hopbox/guest_link_key` with the headers `X-metadata-token: <token>` and `Accept: application/json`.

The guest agent also accepts the key as plain text, without the JSON quotes.

Before the network of the machine has its address, the guest cannot reach the metadata service. So when the metadata service has an IPv4 link-local address, the guest agent first does these steps on its network interface:

1. It brings the interface up.
2. When the interface has no IPv4 address, it adds `169.254.0.2/32` with link scope.
3. It adds the route `169.254.169.254/32` on the interface, with link scope.

The network setup of the first configuration removes the address `169.254.0.2/32` again. The route stays.

## Handshake

Every link starts with a handshake on the `Link` call. The guest agent speaks first.

1. The guest agent sends `GuestHello`: the lowest and the highest version that it speaks, its build version, and a new nonce of 32 random bytes. It also sends the boot ID of the guest kernel and a random ID of its own run. A new boot ID means that the machine booted again. A new run ID with the same boot ID means that the agent restarted, which restarts the start command.
2. The node picks the highest version that both sides speak. When no version fits, the node ends the call with the code `failed_precondition`.
3. The node sends `NodeHello`: the version, a new nonce of 32 random bytes, its proof, and its build version.
4. The guest agent checks the version and the proof. When one of them is wrong, it ends the call and sends nothing more. So a process that poses as the node never gets the proof of the guest.
5. The guest agent sends `GuestAuth` with its proof.
6. The node checks the proof. When it is wrong, the node ends the call with the code `unauthenticated`.
7. On a link of version 4 or later, the node sends `Sync` ([Clock and randomness](#clock-and-randomness)). A link of version 5 or later can name the feature `clone` in its hello. On such a link, the node then sends `CloneIdentity` while a clone identity step waits ([Freeze and clone identity step](#freeze-and-clone-identity-step)). Then it sends the configuration in a `Config` message. On such a link, the node then sends `Thaw`, unless a clone identity step or a freeze waits. The first message after the proof tells the guest agent that the node accepted it.

The node waits at most 10 s for the hello and the proof together. `HandshakeTimeout` in `guestlink.Options` sets this limit.

| Message | Field | Type | Description |
| --- | --- | --- | --- |
| `GuestHello` | `min_version` | uint32 | The lowest guest link version that the guest agent speaks. |
| `GuestHello` | `max_version` | uint32 | The highest guest link version that the guest agent speaks. |
| `GuestHello` | `agent_version` | string | The build version of `hopbox-guest`, for example `v0.1.0`. |
| `GuestHello` | `nonce` | bytes | 32 random bytes, new for each link. |
| `GuestHello` | `boot_id` | string | The boot ID of the guest kernel. It changes at each boot of the machine. |
| `GuestHello` | `agent_instance_id` | string | A random ID of this run of the guest agent. Each restart of the agent gives a new ID. |
| `GuestHello` | `features` | list of strings | The features of the guest agent and its image. With `clone`, the agent runs the start command and exec in `hopbox-user.slice`. It freezes the user processes for a save, and runs the clone identity step ([hopbox-guest](/reference/hopbox-guest/index.html.md#freeze-of-the-user-processes)). An older agent sends no features. |
| `NodeHello` | `version` | uint32 | The version that the node picked. |
| `NodeHello` | `nonce` | bytes | 32 random bytes, new for each link. |
| `NodeHello` | `proof` | bytes | The node proof. |
| `NodeHello` | `node_version` | string | The build version of `hopboxd`. |
| `GuestAuth` | `proof` | bytes | The guest proof. |

### Proofs

Each proof is HMAC-SHA256 with the link key over these bytes, in this order:

| Part | Bytes |
| --- | --- |
| Label | `hopbox guest link` and one zero byte |
| Role | `node` for the node proof, `guest` for the guest proof, then one zero byte |
| Offered range | the lowest and the highest version of `GuestHello`, each a 32-bit big-endian number |
| Version | the version of `NodeHello`, a 32-bit big-endian number |
| Guest nonce | the 32 bytes of `GuestHello` |
| Node nonce | the 32 bytes of `NodeHello` |

Both nonces are new on each link, so an old proof never works again. The role differs between the two proofs, so a guest agent cannot send the proof of the node back. The proofs cover the offered range and the version, so nothing between the two sides can force a lower version.

## Versions

A guest link version is a whole number from 1. This build speaks versions 1 to 5. Version 2 adds `ExecControl` on `Link`. On version 1, the node sends the resizes and the signals of an exec on `Exec`. From version 2 on, it sends them as `ExecControl`. Version 3 adds the credit of `Exec` ([Credit of an exec](#credit-of-an-exec)). On a link of version 3, the guest agent acks the messages of the node on `Exec`, and the node sends them on credit. Version 4 adds `Sync` and `SyncResult` ([Clock and randomness](#clock-and-randomness)). Version 5 adds `Freeze`, `FreezeResult`, `Thaw`, `ThawResult`, `CloneIdentity`, and `CloneResult` ([Freeze and clone identity step](#freeze-and-clone-identity-step)). The node sends the three messages of version 5 only to a guest agent whose hello names the feature `clone`. The negotiated version of the link decides, not the build of the guest agent or of `hopboxd`.

| Rule | Detail |
| --- | --- |
| Range | Each side speaks a range of versions, both ends included. |
| Choice | The node picks the highest version in both ranges. |
| Old guest agent | A node with a newer `hopboxd` keeps the old versions in its range, so it still talks to a guest agent from an older image. |
| No common version | The node refuses the link with `failed_precondition`, and its state records the error. The guest agent tries again with its backoff. |
| New fields | A new field in a message needs no new version. An older side ignores fields that it does not know. |
| New version | A change that an older side reads wrongly needs a new version, for example a new meaning of a field. |

## Link messages

After the handshake, both sides keep the `Link` call open. Each `LinkRequest` and each `LinkResponse` holds one message in its `oneof` field `message`.

| Message | Field of the oneof | From | When |
| --- | --- | --- | --- |
| `GuestHello` | `LinkRequest.hello` | guest | the first message of each link |
| `GuestAuth` | `LinkRequest.auth` | guest | the second message, after a `NodeHello` with a valid proof |
| `NodeHello` | `LinkResponse.hello` | node | the answer to `GuestHello` |
| `Sync` | `LinkResponse.sync` | node | from version 4 on, the first message after a valid `GuestAuth`, ahead of the first `Config` of the link |
| `SyncResult` | `LinkRequest.sync_result` | guest | from version 4 on, the answer to `Sync` |
| `CloneIdentity` | `LinkResponse.clone_identity` | node | from version 5 on, to a guest agent with the feature `clone`: the clone identity step. It comes after `Sync` and ahead of the first `Config` of the link. |
| `CloneResult` | `LinkRequest.clone_result` | guest | from version 5 on, the answer to `CloneIdentity` |
| `Freeze` | `LinkResponse.freeze` | node | from version 5 on, to a guest agent with the feature `clone`: freeze the user processes before a save. It holds `seq`, a number that grows with each `Freeze` of the server. |
| `FreezeResult` | `LinkRequest.freeze_result` | guest | from version 5 on, the answer to `Freeze` |
| `Thaw` | `LinkResponse.thaw` | node | from version 5 on, to a guest agent with the feature `clone`: thaw the user processes. It holds `seq`. A non-zero `seq` asks for a `ThawResult`, also with nothing frozen. |
| `ThawResult` | `LinkRequest.thaw_result` | guest | from version 5 on, the answer to a `Thaw` that ended a freeze. It also reports a thaw that the guest agent did by itself. |
| `Config` | `LinkResponse.config` | node | after the handshake, and after each change |
| `ConfigApplied` | `LinkRequest.config_applied` | guest | after each `Config`. It holds the generation, the errors, and the count of env values that the agent left out. |
| `Status` | `LinkRequest.status` | guest | after the handshake, and after each change of the start command. It is the first message of the guest agent after its proof, ahead of the first `ConfigApplied` of the link. A status and a `ConfigApplied` that wait together go in this order. So the node has the SSH host keys of the guest when the link becomes ready. A guest agent of an older image can send `ConfigApplied` first, if its sender starts late. |
| `LogEntry` | `LinkRequest.log` | guest | for each line of output of the start command |
| `Attach` | `LinkResponse.attach` | node | to start an exec, an upload, or a download. It holds the attach ID `id`, and the operation `exec` (an `ExecStart`), `upload` (an `UploadStart`), or `download` (a `DownloadStart`). |
| `RunStartCommand` | `LinkResponse.run_start_command` | node | to run the start command again, for the restart policy "always". It has no fields. If the start command runs, the guest agent does nothing. If there is no start command, it sends the status `START_COMMAND_STATE_NONE` again. |
| `CancelAttach` | `LinkResponse.cancel_attach` | node | to stop an exec, an upload, or a download early. It holds the attach ID `id`. |
| `ExecControl` | `LinkResponse.exec_control` | node | from version 2 on, for a resize or a signal of the command of an exec. It holds the attach ID `id`, and `resize` (a `WindowSize`) or `signal` (a name). `seq` numbers the controls of each exec from 1. |

### Clock and randomness

A guest in standby does not run, and a load gives the guest the clock of its save. The default image has no time service. So from version 4 on, the node sends its wall clock at the start of each link. Each resume and each load opens a new link, so each of them gets the clock.

The same message carries the randomness step of a later load. Firecracker calls it insecure to load one saved state more than once without a step that makes each copy unique. The node asks for the step with the option `Reseed` of the server ([Node API](#node-api)). The executor sets it on each load of a memory file that loaded before ([the node executor](/reference/executor/index.html.md#the-randomness-step)).

| Field of `Sync` | Type | What the guest agent does |
| --- | --- | --- |
| `time` | timestamp | The wall clock of the node at the send. If the clock of the guest differs from it by more than 1 s (`protocol.ClockTolerance`), the guest agent sets the clock of the guest kernel to it. |
| `entropy` | bytes | 64 fresh random bytes of the node (`protocol.EntropySize`), only for the randomness step. The guest agent gives them to the kernel with the ioctl `RNDADDENTROPY` on `/dev/urandom`, and credits 8 bits for each byte. Then it forces a new seed with `RNDRESEEDCRNG`. Empty: no step. |

The guest agent runs the step first, then sets the clock. It answers with `SyncResult`:

| Field of `SyncResult` | Type | Meaning |
| --- | --- | --- |
| `offset` | duration | The clock of the node minus the clock of the guest, at the receipt of `Sync`. A positive value means that the guest clock was behind. |
| `clock_set` | bool | The guest agent set the clock. |
| `reseeded` | bool | The guest agent ran the randomness step. |
| `error` | string | Why the step or the set of the clock failed, one error per line. Empty after a success. The node keeps and logs at most 1024 bytes of it. |

The guest agent handles the messages of `Link` in order, so its `SyncResult` comes before the `ConfigApplied` of the link. So when `Server.Ready` returns, `Server.State` holds the result. The node takes one `SyncResult` for each `Sync`, and only on a link of version 4 or later. It ignores any other `SyncResult`, and logs `ignored a SyncResult that answers no Sync` at level debug.

A failed step or a failed set does not fail the link, and the machine runs. The guest kernel already reseeds itself at each load through VMGenID. The owner decided this on 2026-11-10, as Q215. The node makes the failure visible and tries again:

- It logs `the guest agent failed the randomness step, so the next link asks for it again` or `the guest agent did not set its clock` at level error, with `err`.
- It counts each step in `hopbox_guest_randomness_steps_total{result}` ([Metrics](/reference/metrics/index.html.md#metrics-of-the-executors)).
- While the guest agent did not report the step, the node sends new entropy on each new link. The executor keeps that request across a restart of `hopboxd` ([the node executor](/reference/executor/index.html.md#the-randomness-step)).

The VMGenID reseed has a race window between the load and the new seed.

A guest agent whose guest link version is 3 or lower gets no `Sync`, and never sets the clock. After a load without `clock_realtime`, its clock stays at the time of the save until a program in the guest sets it. A pause stops the guest, but kvm-clock reads the clock of the host after the resume, so the guest clock lags little: 0.65 s after a pause of 60 s, in the KVM test on the CI runner on 2026-11-10. A later load of such a machine has only the reseed of VMGenID, and the node logs `the guest agent speaks no randomness step, so this later load of a saved state has only the kernel reseed of VMGenID`.

### Freeze and clone identity step

Stock Firecracker resumes each process at a load. So the guest agent freezes the user processes before a save for a fork, a snapshot, or a checkpoint. It thaws them only after the steps of the load. The owner decided this on 2026-11-06, as Q178, and on 2026-11-14 that the freeze covers systemd itself, as Q256. [hopbox-guest](/reference/hopbox-guest/index.html.md#freeze-of-the-user-processes) lists the cgroups that a freeze covers, and says what the freeze guarantees and what it does not.

A freeze runs in this order:

1. The node sends `Freeze` with a new `seq`, and waits for the `FreezeResult` with the same `seq`. A late answer to an earlier `Freeze` answers no later one. From then on, the node holds the freeze. A `FreezeResult` with an `error` fails the save.
2. The node saves the microVM. A save resets the vsock connections, so the guest agent opens a new link to the same server. While the node holds the freeze, a new link gets no `Thaw`.
3. At the resume of the microVM, also after a failed save, the node ends its hold. It sends `Thaw` with a new `seq` on the current link. Without a link, the next link carries it. The node waits for the `ThawResult` with that `seq`. A result with `broken` or `by_itself` fails the save, because each load of the save fails its clone identity step.

The thaw has one rule: the node sends `Thaw` on each new link once no step of a load waits and no freeze holds. A restart of `hopboxd` ends the hold, so the first link of the new `hopboxd` thaws the guest. A source can freeze and then get no `Thaw`. It then thaws by itself after 60 sleeps of 1 s from the freeze or from its last new link. It counts sleeps, not the time between two reads of the clock. So a jump of the guest clock at a load costs one sleep at most, and a lost node leaves no frozen source. A loaded copy never thaws by itself, and only the `Thaw` of the node ends its freeze (Q257). The guest agent tells the two apart with the VMClock generation counter.

| Field of `FreezeResult` | Type | Meaning |
| --- | --- | --- |
| `frozen` | list of strings | The cgroups that froze, as paths below `/sys/fs/cgroup`, for example `hopbox.slice/hopbox-user.slice`. |
| `not_frozen` | list of strings | The cgroups that did not report the freeze within 2 s (`protocol.FreezeWait`). Each keeps its freeze. Its tasks are inside a kernel call, and stop before they return to user code. So the save goes on, and only waits no longer. The executor counts such a freeze in `hopbox_guest_freezes_total{result="slow"}`. |
| `error` | string | Why a part of the freeze failed, one error per line. Empty after a success. |
| `seq` | uint64 | The `seq` of the `Freeze` that the result answers. |

| Field of `ThawResult` | Type | Meaning |
| --- | --- | --- |
| `thawed` | uint32 | The count of cgroups that the guest agent thawed. |
| `by_itself` | bool | The guest agent thawed by itself, or ended, during the freeze. |
| `error` | string | Why a part of the thaw failed, one error per line. |
| `broken` | string | Why the freeze check failed, such as a restart of the guest agent, a record of an early thaw, or a cgroup that appeared. Empty for a freeze that held ([hopbox-guest](/reference/hopbox-guest/index.html.md#freeze-of-the-user-processes)). |
| `seq` | uint64 | The `seq` of the `Thaw` that the result answers, or 0. |

A fork, and a new machine from a snapshot, load a save of another machine. The node then asks for the full clone identity step on each link of the load, until the guest agent reports it:

1. `Sync` carries fresh entropy, and the guest agent runs the randomness step ([Clock and randomness](#clock-and-randomness)).
2. `CloneIdentity` carries the MAC address of the new machine in `mac_address`, for example `06:00:3f:a2:91:0c`. The driver gives the microVM its new tap device with `network_overrides`, which changes no MAC address. So the guest agent sets the MAC address of its interface, and drops its neighbor entries. It writes a new `/etc/machine-id`, removes `/var/lib/systemd/random-seed`, shows a new `boot_id`, and replaces the SSH host keys ([hopbox-guest](/reference/hopbox-guest/index.html.md#clone-identity-step)). It answers with `CloneResult`.
3. `Config` carries the new host name and the new address of the machine.
4. The node sends `Thaw` with a `seq` after three answers of the link: a `CloneResult` without an error, a `SyncResult` that ran the randomness step, and a `ConfigApplied` without an error. So no user process runs with the host name or the address of the source.
5. `Server.Ready` of the clone returns only after the `ThawResult` of that `seq` reports a freeze that held. A result with `broken` or `by_itself` fails `Ready` with `ErrCloneStep`.

| Field of `CloneResult` | Type | Meaning |
| --- | --- | --- |
| `error` | string | Why the step failed, one error per line. Empty after a success. |
| `boot_id`, `machine_id` | string | The new `boot_id` and `machine-id` of the guest. |

A failed part of the step, a failed randomness step, or a failed `Config` fails the clone. The node then sends no `Thaw`, so no user process of the clone runs. `Server.Ready` returns the error, which wraps `ErrCloneStep`, and the executor ends the microVM of the clone with a kill ([the node executor](/reference/executor/index.html.md#the-clone-identity-step)). The source stays as it was. After the step, the guest agent sends a status with the new SSH host keys. That status comes ahead of the `ConfigApplied` of the link. So the node reports nothing as `running` before the step, and the SSH front door learns only the new host keys ([the executor protocol](/reference/executor-protocol/index.html.md#what-the-control-role-does)).

A guest agent without the feature `clone` gets no `Freeze` and no clone. `Server.Freeze` fails with `ErrNoClone`, and `Server.Ready` of a clone fails with it too. The error says `the guest agent of the machine cannot freeze its user processes, because its image is older than hopbox-user.slice, so the machine gets no fork, snapshot, or checkpoint. Make the machine again from a new image`, with the build and the link version of the agent. The node then logs `the guest agent of a clone has no clone identity step, so the clone must not run` at level error, and counts it once in `hopbox_guest_clone_steps_total{result="unsupported"}`.

| Message of the node | Level | Fields |
| --- | --- | --- |
| `the guest agent froze its user processes` | info | `cgroups`, a count, and `took` |
| `cgroups of the guest did not freeze in time, so their processes run during the save` | warn | `cgroups` |
| `a part of the freeze of the user processes failed` | error | `err` |
| `the guest agent thawed its user processes` | info | `cgroups` |
| `the guest agent thawed its user processes by itself, because no thaw came in time` | warn | `cgroups` |
| `a part of the thaw of the user processes failed` | error | `err` |
| `the guest agent ran the clone identity step` | info | `boot_id`, `machine_id` |
| `the clone identity step failed, so the clone must not run` | error | `err`, at most 1024 bytes |
| `cannot thaw the user processes of the clone` | warn | `err` |
| `the freeze of the save did not hold until the thaw of the clone, so the clone must not run` | error | `err` |
| `ignored a FreezeResult that answers no Freeze` | debug | |
| `ignored the answer to an earlier freeze` | debug | `seq`, `want` |
| `the freeze of the user processes did not hold until the thaw` | error | `err`, the `broken` of the guest agent |
| `ignored a CloneResult that answers no CloneIdentity` | debug | `version` |

### Configuration

| Field | Type | What the guest agent does |
| --- | --- | --- |
| `generation` | uint64 | Returns it in `ConfigApplied`. The node sets it: 1 for the first configuration, then one more for each change. |
| `hostname` | string | Writes `/etc/hostname`, points `127.0.1.1` at the name in `/etc/hosts`, and sets the kernel host name. An empty value leaves all three as they are. The name must be DNS labels of letters, digits, and hyphens, at most 64 characters in all. |
| `network.address` | string | Makes this address with its prefix length, for example `10.240.0.5/32`, the only IPv4 address of the interface. |
| `network.gateway` | string | Adds the default route through this address, on-link, for example `169.254.0.1`. |
| `network.mtu` | uint32 | Sets the MTU of the interface, for example `1420`. Zero leaves it as it is. |
| `network.address6` | string | Makes this IPv6 address with its prefix length, for example `fd65:1c74:e769::af0:5/128`, the only global IPv6 address of the interface. It skips duplicate address detection, because the machine is alone on its link with the node. The link-local address stays. Empty leaves the IPv6 of the interface as it is. A node whose kernel has IPv6 sends it ([the egress of the machines](/reference/egress/index.html.md#the-ipv6-addresses-of-the-machines)). A guest agent before this field ignores it. |
| `network.gateway6` | string | Adds the IPv6 default route `::/0` through this address, for example `fe80::1`. The node sends it with `address6`. The guest agent refuses a gateway that is not link-local, and the error of the network names it. |
| `nameservers` | list of strings | Writes one `nameserver` line for each IP address to `/etc/resolv.conf`. The new file replaces a symlink at that path. An empty list leaves the file as it is. In phase 1, the node sends the upstream resolvers. |
| `env` | list of `EnvVar`, each with `name` and `value` | Writes `/etc/hopbox/env`. Exec and the start command also get these values. The control role builds the list from the env vars and secrets of the org ([the env var API](/reference/vars-api/index.html.md#env-of-a-machine)). The guest agent leaves out a value whose name it refuses, and logs it ([hopbox-guest](/reference/hopbox-guest/index.html.md#files)). |
| `authorized_keys` | list of strings | Writes the keys, in `authorized_keys` format, to `/etc/hopbox/authorized_keys`. sshd reads this file for every Linux user, next to the `~/.ssh/authorized_keys` of the user. So the SSH front door can log in as any user, and only root can remove its key. |
| `start_command` | string | Runs the command once per boot, after the first configuration that it applies without an error, and after systemd reached `multi-user.target`. [hopbox-guest](/reference/hopbox-guest/index.html.md#start-command) gives every case. |

The guest agent applies each part, even after another part failed. `ConfigApplied.error` then holds the errors of all failed parts, one per line. `ConfigApplied.skipped_env` counts the env values that the agent left out, because their names break its rules. Such a value is no error, so the configuration still applies. An older agent sends 0. A configuration without `network` leaves the network as it is. Network setup needs Linux.

On a new link, the node sends the current configuration again, and the guest agent applies it again. A second apply of a part gives the same result as the first. A new link never starts the start command a second time.

[hopbox-guest](/reference/hopbox-guest/index.html.md#files) lists the files that the guest agent writes.

### Status

| Field | Meaning |
| --- | --- |
| `start_command.state` | `start_command` is a `StartCommandStatus`. Its `state` is a `StartCommandState`. `START_COMMAND_STATE_NONE`: no start command, or no configuration yet. `START_COMMAND_STATE_WAITING`: the command waits for the end of the boot. `START_COMMAND_STATE_RUNNING`: the command runs. `START_COMMAND_STATE_EXITED`: the command ended. `START_COMMAND_STATE_FAILED`: the command did not start. |
| `start_command.runs` | How many times the start command started in this boot. |
| `start_command.exit_code` | The exit code of the last run. A run that a signal ended has 128 plus the number of the signal. |
| `start_command.signal` | The name of the signal that ended the last run, for example `SIGKILL`. |
| `start_command.started_at`, `exited_at` | The times of the last run. |
| `start_command.error` | Why the start command did not start. In the state `WAITING`, it holds a warning after 2 min of waiting for the boot. |
| `ssh_host_keys` | Each public SSH host key of the guest, from `/etc/ssh/ssh_host_*_key.pub`, as an `SshHostKey`: `public_key` in `authorized_keys` format, and `fingerprint`, the SHA-256 fingerprint, for example `SHA256:...`. The SSH front door checks the host key of the guest against this list. |

The guest agent reports the exit of the start command and does not start it again on its own. The node applies the restart policy, and sends `RunStartCommand` for the policy "always".

### Logs

Each `LogEntry` holds one line of output of the start command in `data`, without its newline. It also holds its `time`, and its `stream`, a `LogStream`: `LOG_STREAM_STDOUT` or `LOG_STREAM_STDERR`. A line longer than 16 KiB arrives in parts of 16 KiB.

While the link is down, the guest agent keeps the lines in memory: at most 10,000 lines and 4 MiB. When the buffer is full, it drops the oldest line. The next line that it sends has the number of dropped lines in `dropped`.

## Exec and file copy

The node starts an exec, an upload, or a download with an `Attach` message on `Link`. The message holds a new attach ID of 16 random bytes and the operation. The guest agent then opens the matching call, `Exec`, `Upload`, or `Download`, and sends the attach ID as its first message. The node accepts each attach ID once, and only for the matching call. An unknown attach ID ends the call with the code `not_found`. The attach ID travels only on the authenticated link, so it proves that the call comes from the guest agent.

The node waits at most 10 s for the guest agent to open the call. If the link drops before that, the call fails at once.

In these three calls, the guest agent is the client. So a request message goes from the guest to the node, and a response message goes from the node to the guest.

### Exec

`ExecStart` describes the command:

| Field | Default | Description |
| --- | --- | --- |
| `argv` | empty | The program and its arguments. The guest agent looks the program up in the `PATH` of the command. An empty list runs the login shell of the user as a login shell, which `hopbox connect` uses. |
| `env` | empty | Env vars on top of the env of the machine. |
| `user` | the main user | The Linux user. The guest agent sets the user ID, the group ID, and the groups of the user from `/etc/passwd` and `/etc/group`. |
| `dir` | the home directory | The working directory. A relative path starts at the home directory. When the home directory does not exist, the default is `/`. |
| `tty` | `false` | Runs the command in a pseudo-terminal. All output then arrives as stdout. As with sshd, the terminal belongs to the user and the group `tty`, with mode 0620. |
| `size` | 24 rows, 80 columns | The first size of the terminal, a `WindowSize` with `rows` and `cols`. |
| `term` | `xterm-256color` | The `TERM` env var, with `tty` only. |

The guest agent builds the env of a command in four steps. A later value replaces an earlier one.

1. `HOME`, `USER`, `LOGNAME`, `SHELL`, `PATH`, and `LANG`. `LANG` comes from `/etc/default/locale`, the file that a login on Ubuntu reads. Without a value there, it is `C.UTF-8`.
2. `TERM`, with a terminal only.
3. The env of the machine.
4. The `env` field of `ExecStart`. The default `PATH` is the one of Ubuntu: `/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin:/usr/games:/usr/local/games:/snap/bin`.

| Message | From | Content |
| --- | --- | --- |
| `ExecRequest.attach_id` | guest | the attach ID, always the first message |
| `ExecRequest.stdout`, `stderr` | guest | output |
| `ExecRequest.exit` | guest | the end of the command, always the last message |
| `ExecRequest.read` | guest | from version 3 on, an ack: what the messages of the node cost that the guest agent took off the call since its last ack |
| `ExecResponse.stdin` | node | input |
| `ExecResponse.stdin_eof` | node | the end of the input. Without a terminal, the guest agent closes the input of the command. With a terminal, it sends the end-of-file character, Ctrl-D. |
| `ExecResponse.resize` | node | a new size of the terminal |
| `ExecResponse.signal` | node | a signal for the process group of the command, by name, for example `SIGINT`. The guest agent ignores a name that it does not know. |
| `ExecResponse.control` | node | from version 2 on, the copy of an `ExecControl` that the node sent on `Link` |

From version 2 on, the node sends `resize` and `signal` as `ExecControl` on `Link` instead. On `Exec`, they wait behind the input that the command did not read yet. On `Link`, they reach a command that reads no input at once. The node queues each `ExecControl` for `Link`, at most 64 for each link. A guest agent that reads `Link` slowly never blocks the node: once the queue is full, the node refuses the next resize or signal, and `Process.Resize` or `Process.Signal` returns an error. The client of the exec does not learn of the refusal. The node logs `a resize or a signal of an exec did not reach the guest agent` at level warn, with `component=access`, `machine`, `err`, and `missed`, at most once a minute for each exec. A guest agent of version 2 or later takes them on both paths. An `ExecControl` for an exec whose command did not start yet waits for the start, and the guest agent applies it then, in order. At most 16 wait for each exec, and the guest agent drops the next one.

`Link` and `Exec` are two calls. So the guest agent can read a control on `Link` after the input that came after it on `Exec`. A shell can then print the old size of the terminal. The node prevents this with a copy of each `ExecControl` that it queued. It sends the copies as `ExecResponse.control` at once, and always before the next `stdin` or `stdin_eof`. A resize right after a resize replaces the copy of the older one while that copy waits.

The guest agent keeps the highest `seq` that it applied for each exec. It ignores a control whose `seq` is not higher. So the first copy to arrive takes effect, and the other does nothing. A `seq` of 0 comes from an older node, and the guest agent always applies that control. The result: a resize or a signal takes effect before any input that the node sends after `Process.Resize` or `Process.Signal` returns. A guest agent from an older image ignores the copies and keeps the old order.

The copies wait only while the guest agent reads no messages of the exec. From version 3 on, they also wait for the credit of the call ([Credit of an exec](#credit-of-an-exec)). At most 64 copies wait for each exec. Once 64 wait, the node refuses the next resize or signal, and `Process.Resize` or `Process.Signal` returns an error. A resize right after a waiting resize replaces it, so the node still takes it. The node logs the refusal as a control that did not reach the guest agent.

#### Credit of an exec

The HTTP/2 server of Go queues the WINDOW_UPDATE of a call behind the DATA of the same call ([golang/go#82615](https://github.com/golang/go/issues/82615)). The node is the HTTP/2 server. While its input for a command waits for the window of the guest agent, the node sends no WINDOW_UPDATE on the call, although it reads all output. Then the guest agent can send no more than 256 KiB of output. A command that blocks on its output before it reads more input, such as `gzip` or `cat`, then stops for good. This happens when the input fills the queue of the guest agent (64 chunks, 2 MiB), the pipe of the command, and the window of the call.

From version 3 on, the node sends on credit:

- The node keeps at most 128 KiB in flight on the call (`protocol.ExecCredit`), half of the window of the guest agent. A message costs its size as `proto.Size` counts it, plus 32 (`protocol.ExecMessageCost`). One message costs at most 96 KiB, so it always fits next to the part that the guest agent did not ack yet. A send that needs more waits until an ack comes or the command ends.
- The guest agent counts the cost of each message that it takes off the call. Once the count reaches 32 KiB (`protocol.ExecAckAfter`), it sends the count in `ExecRequest.read`. It sends from its own goroutine, so a wait of the output never holds back the input. It sends no ack after the exit.
- The node clamps each ack to 128 KiB and never lets the count in flight go below 0. A guest agent that acks more than it took, or before it took it, can only make the node wait for HTTP/2 flow control on its own call, as before version 3. The calls of other machines never see its acks.
- `Process.Signal` refuses a name of more than 32 bytes (`protocol.MaxSignalName`), and each input message holds at most 32 KiB. So every message costs at least its bytes on the wire.
- The node takes the acks in its reader of the call, behind the output. So a consumer of the output that stops reading stops the input too, but no command can stop its own output.

On version 2 and before, the node sends without a bound, and such a command can still stop. `TestExecWritesBeforeItReads` checks a command that writes 1 MiB before it reads 8 MiB of input. `TestExecCopiesBothWays` checks 4 MiB through `cat` with and without the credit. `TestExecWritesBeforeItReadsThroughEveryHop` runs the same command through the control role, the control channel, and the guest link. It needs the credit of both hops ([the control channel](/reference/control-channel/index.html.md#credit-of-a-data-stream)).

`ExecExit` reports the end:

| Case | `code` | `signal` | `error` |
| --- | --- | --- | --- |
| The command exited | its exit code | empty | empty |
| A signal ended the command | 128 plus the number of the signal | the name, for example `SIGTERM` | empty |
| The program is not in the `PATH` | `127` | empty | the reason |
| The command did not start for another reason, for example an unknown user, a missing working directory, or a file without execute permission | `126` | empty | the reason |

When the node ends the call before the exit, it also sends `CancelAttach` with the attach ID on `Link`. The guest agent then sends `SIGTERM` to the process group of the command. After 2 s, it sends `SIGKILL` to a command that still runs. This works also for a command that reads no input and writes no output. Each command leads its own process group, so the signals also reach its children. The guest agent sends no signal after it reaps the command, because the kernel can then give its group ID to another process. A background child can keep the output of a command open. So after the exit, the guest agent waits at most 2 s for the rest of the output.

### Upload

`UploadStart` describes one upload into the machine:

| Field | Default | Description |
| --- | --- | --- |
| `path` | required | The target. A relative path starts at the home directory of the user. |
| `archive` | `false` | The data is a tar stream of the contents of a directory. If the directory at `path` is missing, the guest agent creates it. Otherwise the data is the contents of one file. |
| `mode` | `0644` | The permission bits of a file. An archive carries the bits of each entry. |
| `user` | the main user | The user that owns the new files and directories. |

The guest agent sends the attach ID in `UploadRequest.attach_id` first. The node sends the data in `UploadResponse.data` messages, then `UploadResponse.eof`. The guest agent answers with `UploadRequest.result`, an `UploadResult`. Its field `error` is empty after an upload that succeeded.

A file upload writes a temporary file in the target directory, then renames it to the target in one step. So a reader never sees a part of the file, and a failed upload leaves no file. An upload to the path of a directory fails.

An archive upload writes regular files, directories, symlinks, and hard links. It skips other entry types. Every entry stays inside the target directory, also through a symlink in the archive. An entry that leaves it fails the upload. The guest agent keeps only the permission bits of each entry, never set-user-ID or set-group-ID bits, and keeps the modification time of files.

### Download

`DownloadStart` describes one download from the machine:

| Field | Default | Description |
| --- | --- | --- |
| `path` | required | The source. A relative path starts at the home directory of the user. |
| `user` | the main user | The user whose home directory a relative path starts at. |

The guest agent sends the attach ID in `DownloadRequest.attach_id` first. Then it sends `DownloadRequest.info`, a `DownloadInfo`, then the data in `DownloadRequest.data` messages, then ends its side of the call. The node answers with an empty `DownloadResponse`. `DownloadInfo` holds:

| Field | Description |
| --- | --- |
| `archive` | The source is a directory, and the data is a tar stream of its contents. Otherwise it is the contents of one file. |
| `mode` | The permission bits of the file or the directory. |
| `size` | The size of a file in bytes. The guest agent sends no more than this, also for a file that grows. The node fails the download at the first byte over this size, or at an end before this size. Zero for an archive. |

An archive holds regular files, directories, and symlinks, with names relative to the directory. It skips devices, sockets, and pipes. When the guest agent fails, it sends `DownloadRequest.error` and nothing more. The node limits the size of a download, not the guest agent: see `MaxBytes` below.

### Rights

The guest agent runs as root, but a file copy runs with the file system rights of the user of the copy: its user ID, its group ID, and its groups. The agent sets them on one thread with `setfsuid`, `setfsgid`, and `setgroups`. So a copy cannot read or write a file that its user cannot read or write. This also holds for a path through a symlink that the user made, for example a link from the home directory to `/etc`. New files belong to the user. Exec runs as its user.

## Threat model

The link key keeps a process that poses as the node from getting the proof of the guest, or from pushing a configuration. It also keeps a process outside the machine from posing as the guest, because each microVM has its own vsock device socket. It does not keep a process inside the machine away from the key: every process in the guest can read the metadata service, and the main user has sudo. A process in the machine with the key can pose as the guest agent of its own machine only.

## Node API

The executor of each machine uses the Go package `internal/node/guestlink` ([the node executor](/reference/executor/index.html.md)). One `Server` serves one boot of one machine.

| Function | Description |
| --- | --- |
| `Listen(path, options...)` | Listens on the unix socket `path`, and first removes a socket file that a stopped node left. The option `SocketMode(mode)` sets the permission bits of the socket file, and `SocketOwner(uid, gid)` sets its owner. Firecracker connects to the socket from its jail as the user of the jailer, so it needs write permission. |
| `NewServer(Options)` | Returns a server for one link key and one first configuration. |
| `Server.Serve(listener)` | Accepts guest agent connections until `Close`. |
| `Server.Close()` | Stops the server and ends every call. It waits for the handlers of the server and for `OnState`, so nothing logs or calls `OnState` after it returns. |
| `Server.Ready(ctx)` | Waits until the guest agent applied the current configuration on an authenticated link. If the configuration failed, it returns the error of the guest agent. The node uses it for the "running" state of a machine. |
| `Server.State()` | Returns `State`. It holds whether a link is up, the version, and the build version of the guest agent. It holds the boot ID and the run ID of the agent from the last hello. It also holds the current and the applied generation, the configuration error, and the count of skipped env values. It holds the last status of the guest and the last link error. It holds the last `SyncResult`: `ClockOffset`, `ClockSet`, and `SyncError`, and `Reseeded`, which stays true once a link of the server ran the randomness step. Last, it holds `Features` from the last hello, and `Frozen` from a `FreezeResult` until a `ThawResult`. `Cloned` stays true once a link ran the clone identity step, and `CloneThawed` once the thaw of the clone reported a freeze that held. `CloneError` holds the error of the step or of that thaw on the current link. `State.CanClone()` reports a link of version 5 or later whose hello names `clone`. |
| `Server.UpdateConfig(ctx, config)` | Makes `config` the configuration, pushes it, and waits until the guest agent applied it. |
| `Server.RunStartCommand(ctx)` | Asks the guest agent to run the start command again. |
| `Server.Freeze(ctx)` | Freezes the user processes before a save, and holds the freeze until `Thaw`. It returns a `FreezeResult` with `Frozen`, `NotFrozen`, `Error`, and `Took`. A guest agent without the feature `clone` fails with `ErrNoClone`. The caller calls `Thaw` once the save ended, also after an error. |
| `Server.Thaw(ctx)` | Ends the hold of `Freeze`, and sends `Thaw` on the current link. Without a link, the next link sends it. It waits for the answer, and returns a `ThawResult` with `Thawed`, `ByItself`, `Broken`, and `Error`. `ThawResult.Held()` reports a freeze that held until the thaw. A guest agent without the feature `clone` gets no `Thaw`, and the call returns at once. |
| `Server.Exec(ctx, ExecOptions)` | Starts a command. `Process.Wait` returns its `ExecExit`. `Process.Resize` sets the terminal size. `Process.Signal` sends a signal by name. Both use `ExecControl` from version 2 on. When `ctx` ends first, the guest agent stops the command with `SIGTERM`, then `SIGKILL`. |
| `Server.Upload(ctx, UploadStart, reader)` | Uploads the data of the reader. |
| `Server.Download(ctx, DownloadStart, options...)` | Returns a `Download`, which holds the `DownloadInfo` and reads the data. The option `MaxBytes(n)` caps the data, for example at 4 MB for a laptop. A file over the cap fails before any data, and an archive fails at the cap, both with `ErrTooLarge`. |

| Option | Default | Description |
| --- | --- | --- |
| `Key` | required | The link key, 32 bytes. |
| `Config` | empty | The first configuration. |
| `NodeVersion` | empty | The build version of `hopboxd` for `NodeHello`. |
| `Versions` | the versions of this build | The versions that the node speaks. |
| `OnReseed` | none | Gets the result of each randomness step that the server asked for: `ok`, `failed`, or `unsupported`, once for each server with a guest agent whose guest link version is 3 or lower. It must not block. |
| `Reseed` | `false` | Asks for the randomness step: the microVM runs from a load of a saved state that a guest already ran from. Each link of version 4 or later then carries fresh `entropy`, until the guest agent reports the step. |
| `Clone` | none | Asks for the full clone identity step, with the `MAC` of the new machine: the microVM is a fork, or a new machine from a snapshot. Each link then carries the randomness step and `CloneIdentity`, until the guest agent reports both. `Ready` waits for them. It fails after a failure of one of them, and for a guest agent without the feature `clone`. |
| `OnClone` | none | Gets the result of each clone identity step: `ok`, `failed`, or `unsupported`, once for each server with a guest agent without the feature `clone`. It must not block. |
| `OnState` | none | Gets each new `State`, in order, on a goroutine of the server. It can call the methods of the server. It gets no state after `Close` returns. When `OnState` falls behind, a new state replaces a queued state with the same `Connected` value, so every connect and disconnect stays in the queue. |
| `OnLog` | none | Gets each `LogEntry`, in order. |
| `HandshakeTimeout` | 10 s | The longest wait for the hello, for the proof, and for the attach ID of each call. |
| `AttachTimeout` | 10 s | The longest wait for the guest agent to open a call after an `Attach`. |

An error that the guest agent reports for a file copy has the type `GuestError`. A call that needs the link while no guest agent is connected fails with `ErrNotConnected`. A freeze or a clone of a guest agent without the feature `clone` fails with `ErrNoClone`. The `Ready` of a clone whose step, randomness step, or first `Config` failed fails with `ErrCloneStep`.
