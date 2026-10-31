# The machine API

> The machine calls of the public API, MachineService: who can do what, the messages, the lifecycle calls, the access calls, sharing, settings, the suspension, the audit entries, and the change events.

The service `hopbox.v1.MachineService` of the public API creates, reads, and changes the machines of an org. [Placement](/reference/placement/index.html.md) covers how a create picks the node of a machine. [The admin API](/reference/admin-api/index.html.md#machineservice) covers the calls of instance admins on machines, and [the admin API](/reference/admin-api/index.html.md#imageservice) the images that new machines start from. [The API server](/reference/api/index.html.md) covers the parts that every call shares: the request gate, the policy check, the idempotency keys, and the error codes. The generated reference is [Public API](/api/hopbox.v1.openapi.yaml). The code is in `internal/service` (`machines.go`, `machine_lifecycle.go`, `machine_admin.go`, `machine_events.go`) and `internal/api` (`handlers_machine.go`). The access calls are in `machine_access.go` and `handlers_machine_access.go`.

| Calls | What they do |
| --- | --- |
| `CreateMachine` | Makes a machine and places it on a node. |
| `GetMachine`, `ListMachines`, `WatchMachines` | Read the machines that the caller sees, once or as a stream of changes. |
| `ListMachineSizes` | Reads the sizes that a machine can have. |
| `StartMachine`, `StopMachine`, `RebootMachine`, `RenameMachine`, `DeleteMachine` | Set the target state of a machine, then wait for it ([Lifecycle calls](#lifecycle-calls)). |
| `ShareMachine`, `UnshareMachine` | Open a private machine to the whole org, or make it private again. |
| `UpdateMachine` | Changes the restart policy, the start command, or the auto-delete timeout. |
| `SetMachineURLPort`, `SetMachineURLVisibility` | Change the port that the URL of a machine reaches, and who can open the URL. |
| `Exec`, `Upload`, `Download`, `StreamMachineLogs` | Run a command in a machine, copy files in and out, and read its log ([Access calls](#access-calls)). The WebSocket `/api/ws/exec` carries the same exec for the console. |

The examples on this page use the shell function `browser` of [the sign-in, account, and audit API](/reference/auth-api/index.html.md), with the session cookie of the caller. [Create and manage machines](/guide/machines/index.html.md) runs the whole flow on a server.

## What runs the machines

The machine API sets the target state of a machine and waits for it. The executor of the machine on its node moves the machine there, and reports each step back ([the node executor](/reference/executor/index.html.md), [the executor protocol](/reference/executor-protocol/index.html.md)). The waits of the lifecycle calls end once the machine got there ([Lifecycle calls](#lifecycle-calls)).

On a server with the node role, `hopboxd run` runs the executors with the Firecracker driver ([the Firecracker driver](/reference/firecracker-driver/index.html.md)). A new machine reaches `running` once its guest agent applied the configuration of the machine. The first boot of an image on a node also unpacks the image. For the default image on the [tested server](/reference/preflight/index.html.md#the-tested-server), the whole create takes about 25 s.

Each machine gets an IPv4 address of the subnet of its node at its create ([the node network](/reference/node-network/index.html.md#addresses)). Its URL and the SSH front door reach it at this address ([the edge role](/reference/edge/index.html.md), [the SSH front door](/reference/ssh-front-door/index.html.md)). When the subnet of the node has no free address, the create fails with `no capacity`. Exec, file copy, and the log run over the guest link, without the network ([the node executor](/reference/executor/index.html.md#exec-file-copy-and-logs)). Env vars and secrets reach a machine through its spec ([the env var API](/reference/vars-api/index.html.md)).

## The org of a request

`CreateMachine`, `ListMachines`, and `WatchMachines` act in one org. They have the field `org_id`, an optional string. Leave it out to act in the default org of the caller: the org of an API key, or else the personal org of the account. A caller without either acts in no org, and the policy refuses it.

The service layer reads the role of the caller in the org before the policy check. A caller that is not a member has no role. So an org that does not exist gets the same `permission_denied` as an org of others. An `org_id` that is not a UUID gets `invalid_argument`.

## Naming a machine

A call on one machine names it in the oneof field `machine`: either `id`, the UUID of the machine, or `name`, its name. Machine names are unique in the whole install ([Domain rules](/reference/domain-rules/index.html.md#machine-names)), so a name alone finds the machine.

| Case | Code | Message |
| --- | --- | --- |
| Neither field is set | `invalid_argument` | `invalid argument: the request names no machine` |
| `id` is not a UUID | `invalid_argument` | `invalid argument: the machine ID "<id>" is not a UUID` |
| `name` breaks the name rules | `invalid_argument` | `invalid argument: invalid name: ...`, or `invalid argument: reserved machine name: "ns1"` |
| No machine has the ID | `not_found` | `not found: no machine has the ID <id>` |
| No machine has the name | `not_found` | `not found: no machine has the name "<name>"` |

A machine that is being deleted keeps its name in its row until its executor deleted it. Meanwhile, a new machine of the same org can take the name back. A name then names two machines for a while. The lookup by name returns the one that holds the name live, else the newest. A lookup by name never finds a machine whose current state is `deleted`. A lookup by ID still finds it, until the leader drops its row ([hopboxd](/reference/hopboxd/index.html.md#leader-jobs)).

## Who can do what

The creator of a machine is the account that made it, or the org key that made it. A machine starts private to its creator. Its creator can share it with the whole org.

| Call | The creator | A member, on a shared machine | A member, on a private machine of another | An org admin or owner, on a private machine of another |
| --- | --- | --- | --- | --- |
| `GetMachine`, and the machine in `ListMachines` and `WatchMachines` | yes | yes | no, `not_found` | yes |
| `StopMachine`, `DeleteMachine` | yes | yes | no, `not_found` | yes |
| `StartMachine`, `RebootMachine`, `RenameMachine`, `UpdateMachine` | yes | yes | no, `not_found` | no, `permission_denied` |
| `ShareMachine`, `UnshareMachine`, `SetMachineURLPort`, `SetMachineURLVisibility` | yes | no, `permission_denied` | no, `not_found` | no, `permission_denied` |
| `Exec`, `Upload`, `Download`, `StreamMachineLogs`, and the WebSocket of exec (action `machine.exec`) | yes | yes | no, `not_found` | no, `permission_denied` |

These rules hold for each caller that acts in the org of the machine:

- A `session` or a `cli_token` acts in each org where its account has a role.
- A `member_key` acts only in its own org, as its account, while the account has a role there.
- An `org_key` acts only in its own org. It has the rights of a member, without private machines. It is never an org admin. A machine that an org key makes is shared from the start, and its creator is that key.
- An instance admin has only the rights of its own roles here. It acts on the machines of others through the admin API ([the admin API](/reference/admin-api/index.html.md#machineservice)).

A caller that cannot read a machine gets `not_found`, with the same message as for a machine that does not exist. So the call tells nothing about the private machines of others.

`RenameMachine`, `UpdateMachine`, `SetMachineURLPort`, and `SetMachineURLVisibility` check their fields before they look up the machine. So a bad value gets `invalid_argument`, also for a machine that does not exist, or that the caller cannot read. The check reads only the request, so it tells nothing about the machine. One check of `RenameMachine` comes after the lookup: a new name that looks like a credential ([Domain rules](/reference/domain-rules/index.html.md#machine-names)). So a rename of a machine to its own name never fails on it. An anonymous caller still gets `unauthenticated` first. A caller that can read the machine, but not do the action, gets `permission_denied` with the message `permission denied: a caller of kind <kind> may not do <action>`.

The API listener resolves session cookies, CLI tokens, and API keys ([the API server](/reference/api/index.html.md#request-gate)). A CLI token acts as its account. A member key acts as its account, in the org of the key only. An org key acts as a member of its org without private machines.

## Messages

### Machine

| Field | Type | Description |
| --- | --- | --- |
| `id` | string | The UUID of the machine. |
| `org_id` | string | The UUID of the org of the machine. |
| `name` | string | The name of the machine, unique in the install. It is the first label of its URL. |
| `state` | `MachineState` | The current state ([Domain rules](/reference/domain-rules/index.html.md#machine-states)). |
| `target_state` | `MachineState` | The state that the machine must reach. |
| `size` | `MachineSize` | The size of the machine. The machine keeps the numbers of its size from its create. |
| `image_name` | string | The name of the image that the machine started from, for example `default`. |
| `image_digest` | string | The digest of that image: `sha256:` and 64 hex digits. |
| `creator_account_id` | string | The UUID of the account that made the machine. Empty for a machine that an org key made, and after the delete of the account. |
| `creator_key_id` | string | The UUID of the org key that made the machine, or empty. |
| `shared` | bool | `true` for a machine that is open to the whole org. |
| `restart_policy` | `RestartPolicy` | The action of the executor after the machine stops by itself. |
| `start_command` | string | The command that the guest agent runs after each boot, or empty. |
| `auto_delete_timeout` | `google.protobuf.Duration` | The time without use after which hopbox deletes the machine. Unset means off. The delete comes 1 to 2 minutes after this time from the last recorded use ([Auto-delete](#auto-delete)). |
| `url` | string | `https://<name>.<domain>`, for example `https://box1.example.com`. Empty while the install has no domain. |
| `url_port` | int32 | The port in the machine that the URL reaches. The default is 8000. |
| `url_visibility` | `URLVisibility` | Who can open the URL. The default is `URL_VISIBILITY_PRIVATE`. |
| `ssh_port` | int32 | The SSH port of the machine on the edge servers, 10000 to 30000. It never changes. |
| `reboot_pending` | bool | `true` while a boot that a request asked for did not run yet, and the target is `running`. A machine with another target has no pending boot. For example, a stop that comes before the boot of a start ends it. |
| `skipped_env_vars` | int32 | The count of env vars and secrets that the guest agent of the running machine left out, because their names break its rules ([the env var API](/reference/vars-api/index.html.md#names-and-values)). 0 is the normal case. |
| `last_activity_time` | `google.protobuf.Timestamp` | The last use of the machine, for the auto-delete timeout ([Auto-delete](#auto-delete)). |
| `state_change_time` | `google.protobuf.Timestamp` | The time of the last change of the current state. |
| `delete_request_time` | `google.protobuf.Timestamp` | The time of the delete request. Unset for a machine whose target is not `deleted`. |
| `version` | int64 | The version of the machine. It starts at 1 and grows by 1 with each change of the machine. A request makes such a change. A report of its executor makes one too. So does a new count of skipped env values. So does a change of an env var or a secret that reaches the machine. |
| `create_time` | `google.protobuf.Timestamp` | The creation time of the machine. |
| `update_time` | `google.protobuf.Timestamp` | The time of the last change that a request made. |

In the JSON codec, a field with its default value is left out, for example `shared` false or an empty `start_command`.

### MachineSize

| Field | Type | Description |
| --- | --- | --- |
| `name` | string | The name of the size, for example `default`. |
| `vcpus` | int32 | The count of vCPUs. |
| `memory_bytes` | int64 | The RAM, in bytes. |
| `disk_bytes` | int64 | The size of the sparse disk, in bytes. |

A fresh install has one size, `default`: 2 vCPUs, 8 GiB of RAM, and a sparse disk of 100 GB ([Domain rules](/reference/domain-rules/index.html.md#sizes)). An instance admin can add other sizes ([the admin API](/reference/admin-api/index.html.md#sizeservice)).

### MachineState

| Value | State |
| --- | --- |
| `MACHINE_STATE_UNSPECIFIED` | Never in a response. |
| `MACHINE_STATE_PENDING` | `pending`: the machine exists, but no node acted on it yet. |
| `MACHINE_STATE_STARTING` | `starting`: the microVM boots. |
| `MACHINE_STATE_RUNNING` | `running`: the microVM runs. |
| `MACHINE_STATE_STANDBY` | `standby`: the microVM is paused, and its memory stays in the RAM of the node. |
| `MACHINE_STATE_HIBERNATED` | `hibernated`: the memory is in a file on disk, and the microVM does not run. |
| `MACHINE_STATE_STOPPED` | `stopped`: the microVM does not run. The disk stays. |
| `MACHINE_STATE_FAILED` | `failed`: an action failed. The saved state of the machine stays. |
| `MACHINE_STATE_DELETED` | `deleted`: the machine and its saved state are gone. |
| `MACHINE_STATE_MOVING` | `moving`: a drain moves the machine to another node. |

### RestartPolicy

| Value | Policy |
| --- | --- |
| `RESTART_POLICY_UNSPECIFIED` | Never in a response. In `CreateMachineRequest`, it takes the default `always`. In `UpdateMachineRequest`, it keeps the policy. |
| `RESTART_POLICY_ALWAYS` | `always`: the executor starts the machine again after a crash, a shutdown from inside, or a node reboot ([Domain rules](/reference/domain-rules/index.html.md#restart-policy)). |
| `RESTART_POLICY_NEVER` | `never`: the machine and its start command stay stopped. |

A value that is not in the enum gets `invalid_argument` with the message `invalid argument: the restart policy <n> is not known`.

### URLVisibility

| Value | Visibility |
| --- | --- |
| `URL_VISIBILITY_UNSPECIFIED` | Never in a response. A request cannot use it. |
| `URL_VISIBILITY_PRIVATE` | `private`: only an account that uses the machine can open its URL, in a browser with a console session ([the edge role](/reference/edge/index.html.md#private-machine-urls)). |
| `URL_VISIBILITY_PUBLIC` | `public`: anyone can open the URL, without an account. |

The edge servers serve the URL of a machine ([the edge role](/reference/edge/index.html.md#machine-urls)). A change of the URL port or of the visibility reaches them with the next routes, within a second or two ([the control channel](/reference/control-channel/index.html.md#edge-pushes)).

## Lifecycle calls

`StartMachine`, `StopMachine`, `RebootMachine`, `RenameMachine`, and `DeleteMachine` are the lifecycle calls. Each one takes these steps:

1. It checks the policy, then its input.
2. It locks the rows in the order of [Control store tables](/reference/control-store/index.html.md#orgs): the key lock of the creator account, the org row, then the machine row. It checks the policy again under the locks, because the role or the machine can change in between.
3. It refuses a call that the suspension of the org blocks ([Suspension](#suspension)).
4. It sets the new target state, as `Apply(target, current)` of the domain gives it ([Domain rules](/reference/domain-rules/index.html.md#lifecycle-actions)). An action that the states do not allow gets `failed_precondition`, for example `failed precondition: machine action not allowed: cannot reboot a machine that is pending`.
5. It writes the audit entry and the change event in the same transaction, and commits.
6. It waits for the machine, then returns it.

A request that asks for the target that the machine has already changes nothing. It writes no audit entry and no event, and the version stays. The call still waits. For example, a stop of a machine with the target `stopped` waits for the state `stopped`. The rule has three exceptions. A reboot always asks for a new boot. A start of a `failed` machine asks for a new boot too, even with the target `running` already ([StartMachine](#startmachine)). A delete of a machine whose target is `deleted` fails with `failed_precondition` and the message `failed precondition: machine action not allowed: cannot delete a machine that is being deleted`.

### The wait

The executor of the node moves the machine toward its target. The call waits for that, and ends its wait at the first of these events:

| Event | `done` |
| --- | --- |
| The machine reached what the call asked for. | `true` |
| The current state turned `failed` after the change of the call. A machine that was `failed` before the call is on its way out of that state, so only a later failure counts. | `false` |
| Another request set another target, for example a stop during the wait of a start. | `false` |
| 60 s passed, or the deadline of the call is 1 s away. | `false` |
| The client canceled the call. | `false` |
| The listener of the call stops, for example at a stop or an upgrade of `hopboxd`. | `false` |

The wait wakes on each change event of the machine ([Change events](#change-events)), and reads the machine again every 2 s in any case. The machine in the response is the machine at the end of the wait. A wait that ends with `done` false is not an error: the call succeeds, and the machine goes on toward its target.

| Call | `done` is `true` when |
| --- | --- |
| `StartMachine` | the current state is `running`, and every boot that a request asked for ran: `reboot_pending` is `false` |
| `StopMachine` | the current state is `stopped` |
| `RebootMachine` | the current state is `running`, and the new boot runs: `reboot_pending` is `false` |
| `RenameMachine` | the new boot runs, for a machine whose current and target state are both `running`. Any other rename has no wait, and returns `done` `true` at once. |
| `DeleteMachine` | the current state is `deleted` |

A unary call whose client sends no deadline gets 30 s ([the API server](/reference/api/index.html.md#the-chain-of-a-call)). A lifecycle call gets 61 s instead (`api.LifecycleTimeout`), so its wait of 60 s fits. A deadline from the client stays as it is, and the wait ends 1 s before it. For example, with the header `Connect-Timeout-Ms: 10000`, the wait ends after about 9 s. A deadline of 1 s or less gives no wait: the call returns the machine after its change.

Each lifecycle response has these fields:

| Field | Type | Description |
| --- | --- | --- |
| `machine` | `Machine` | The machine at the end of the wait. |
| `duration` | `google.protobuf.Duration` | How long the call took, the wait included. |
| `done` | bool | `true`: the machine reached what the call asked for before the wait ended. In the JSON codec, `false` is left out. |

## Machines

### CreateMachine

Makes a machine and places it on a node ([Placement](/reference/placement/index.html.md)). The machine starts with the state `pending` and the target `running`. The call never waits for the machine.

| Item | Value |
| --- | --- |
| Procedure | `/hopbox.v1.MachineService/CreateMachine` |
| Policy action | `machine.create`: a caller that acts in the org ([Who can do what](#who-can-do-what)) |
| Idempotent option | yes. A call must have an `Idempotency-Key` header ([the API server](/reference/api/index.html.md#idempotency-keys)). A retry with the same key returns the same machine. A create that failed committed nothing, so a retry runs again. |
| Audit entry | action `machine.create`, org the org, target kind `machine`, target ID the machine ID, details `name`, `size`, `image` (the digest), and `shared` |

The request `CreateMachineRequest` has these fields:

| Field | Type | Default | Description |
| --- | --- | --- | --- |
| `org_id` | optional string | the default org of the caller | The org of the machine ([The org of a request](#the-org-of-a-request)). |
| `name` | string | a free name: `box1`, `box2`, and so on | The name of the machine ([Domain rules](/reference/domain-rules/index.html.md#machine-names)). |
| `size` | string | `default` | The name of a size. |
| `image` | string | `default` | The name of an image. The machine takes the newest image of that name ([the admin API](/reference/admin-api/index.html.md#registerimage)). |
| `restart_policy` | `RestartPolicy` | `RESTART_POLICY_ALWAYS` | The restart policy. |
| `start_command` | string | empty | A command that the guest agent runs after each boot ([Domain rules](/reference/domain-rules/index.html.md#start-command)). |
| `auto_delete_timeout` | `google.protobuf.Duration` | unset, off | Deletes the machine after this time without use: 0 for off, or 1 minute to 365 days in whole seconds ([Domain rules](/reference/domain-rules/index.html.md#auto-delete-timeout)). The delete comes 1 to 2 minutes after this time from the last recorded use. A use is recorded at most once a minute, and the job runs once a minute ([Auto-delete](#auto-delete)). |

The response `CreateMachineResponse` has the field `machine`, the new `Machine`. A new machine has the URL port 8000, the visibility `private`, and `shared` `false`. A machine that an org key makes has `shared` `true`.

The call takes these steps in one transaction:

1. It takes the key lock of the account of the caller, then locks the org row with `FOR NO KEY UPDATE` ([Control store tables](/reference/control-store/index.html.md#orgs)). So the creates and starts of one org run one after the other, and each limit check counts the one before it.
2. It checks the policy again under the lock, then the suspension of the org.
3. It reads the size with a share lock on its row, and the newest image of the name. A retired size fails. The lock makes a retirement of the size wait for the create, and the create wait for a retirement in progress ([the admin API](/reference/admin-api/index.html.md#retiresize)).
4. It checks the usage limits of the org with the new machine: `limits.Check(usage.AfterCreate(size))` ([Domain rules](/reference/domain-rules/index.html.md#usage-limits)).
5. It claims the name. A request without a name gets the first free name of a list: `box1` to `box9`, then 3 names with a random suffix, such as `box-k3x9q` ([Domain rules](/reference/domain-rules/index.html.md#machine-name-suggestions)). A free name has no machine and no reservation of another org.
6. It claims a random free SSH port from 10000 to 30000. A free port has no machine and no reservation ([Domain rules](/reference/domain-rules/index.html.md#ssh-ports)).
7. It places the machine on a node, with the pool and the labels of the size as filters ([Placement](/reference/placement/index.html.md)).
8. It grants the machine lease `machine/<machine ID>` to the node, with the server ID as the holder and a TTL of 30 s ([Control store tables](/reference/control-store/index.html.md#leases)).
9. It inserts the machine and a row of `placements` with the epoch and the token of the lease grant.
10. It writes the audit entry and publishes the event `machine.created` ([Change events](#change-events)).

A create that fails at any step leaves nothing behind: no machine, no name, no port, and no lease. A create that finds no node with room also logs `no node has room for a machine` ([Placement](/reference/placement/index.html.md#logs)).

| Error | Code |
| --- | --- |
| A bad name, start command, or auto-delete timeout, for example `invalid argument: invalid machine config: the start command holds a NUL byte` | `invalid_argument` |
| A name that looks like a credential, such as `xoxb-` with 10 more digits: `invalid argument: invalid name: the machine name looks like a credential. A machine name is public: it is in the machine URL, in DNS, and in the audit log. Pick another name` ([Domain rules](/reference/domain-rules/index.html.md#machine-names)) | `invalid_argument` |
| An unknown size: `invalid argument: the install has no size "<size>"` | `invalid_argument` |
| A retired size: `invalid argument: the size "<size>" is retired and takes no new machine. Pick another size` | `invalid_argument` |
| The org was deleted after the policy check: `not found: the org <id> was deleted` | `not_found` |
| A suspended org: `failed precondition: org suspended: an instance admin suspended the org, so it cannot create a machine` | `failed_precondition` |
| No image has the name: `failed precondition: the install has no image "<image>". An instance admin registers one with hopboxd admin image register` | `failed_precondition` |
| The org is at a limit, for example `usage limit exceeded: the org would have 4 machines, and its limit is 3` | `resource_exhausted` |
| The name is taken, by a live machine or by a reservation of another org: `already exists: the machine name "box1" is taken. A free name is "box2"`. Without a free suggestion, the message ends with `Pick another name`. | `already_exists` |
| No name of the list is free for a create without a name: `already exists: hopbox found no free name for the machine. Give it a name` | `already_exists` |
| No node has room ([Placement](/reference/placement/index.html.md#when-no-node-has-room)), for example `no capacity: no node has room for a machine of size default (2 vCPUs, 8 GiB of RAM). The install has no node yet` | `unavailable` |
| Every SSH port is taken: `no capacity: the install has no free SSH port in 10000-30000` | `unavailable` |
| Other creates took each free port that this one picked, 5 times: `unavailable: other creates took each free SSH port that this one picked. Retry` | `unavailable` |

The SDKs retry `unavailable`. A retry with the same idempotency key runs the create again, because the failed try committed nothing.

```sh
browser -H 'Idempotency-Key: 9b2f4c1e-box1' -d '{"name":"box1"}' $API/hopbox.v1.MachineService/CreateMachine
```

```json
{"machine":{"id":"01a1460c-1f6e-7b8a-9c42-3d5e7f8a9b10","orgId":"01a14565-8448-7753-9c1f-5ec0a86b8d92","name":"box1","state":"MACHINE_STATE_PENDING","targetState":"MACHINE_STATE_RUNNING","size":{"name":"default","vcpus":2,"memoryBytes":"8589934592","diskBytes":"100000000000"},"imageName":"default","imageDigest":"sha256:422d31d7a4fd5e6958162b22e3d772f71b5a1465cbd6255d253ae8c087873dc8","creatorAccountId":"01a14565-8448-774f-9317-ea6348e9a930","restartPolicy":"RESTART_POLICY_ALWAYS","url":"https://box1.example.com","urlPort":8000,"urlVisibility":"URL_VISIBILITY_PRIVATE","sshPort":21467,"lastActivityTime":"2026-10-16T16:02:11.402518Z","stateChangeTime":"2026-10-16T16:02:11.402518Z","version":"1","createTime":"2026-10-16T16:02:11.402518Z","updateTime":"2026-10-16T16:02:11.402518Z"}}
```

A second create with the name `box1`, from any org:

```json
{"code":"already_exists","message":"already exists: the machine name \"box1\" is taken. A free name is \"box2\"","details":[{"type":"hopbox.v1.MachineNameTaken","value":"CgRib3gxEgRib3gy","debug":{"name":"box1","suggestedName":"box2"}}]}
```

The error of a taken name carries the detail `hopbox.v1.MachineNameTaken`, so a client reads the free name without parsing the message:

| Field | Type | Description |
| --- | --- | --- |
| `name` | string | The name that the request asked for. |
| `suggested_name` | string | A free name close to it. If hopbox found none, it is empty. |

### GetMachine

Returns one machine that the caller sees.

| Item | Value |
| --- | --- |
| Procedure | `/hopbox.v1.MachineService/GetMachine` |
| Policy action | `machine.get`: those who use the machine, and the org admins and owners ([Who can do what](#who-can-do-what)) |
| Idempotent option | no |
| Audit entry | none |

The request `GetMachineRequest` names the machine with `id` or `name` ([Naming a machine](#naming-a-machine)). The response `GetMachineResponse` has the field `machine`.

```sh
browser -d '{"name":"box1"}' $API/hopbox.v1.MachineService/GetMachine
```

### ListMachines

Lists the machines of an org that the caller sees, by name. A machine whose current state is `deleted` is not in the list. A machine whose target is `deleted` is, until its executor deleted it.

| Item | Value |
| --- | --- |
| Procedure | `/hopbox.v1.MachineService/ListMachines` |
| Policy action | `machine.list`: a caller that acts in the org. The list holds only the machines that the caller can read with `machine.get`. |
| Idempotent option | no |
| Audit entry | none |

The request `ListMachinesRequest` has the field `org_id`. The response `ListMachinesResponse` has the field `machines`, a repeated `Machine`. The list has no pages.

```sh
browser -d '{}' $API/hopbox.v1.MachineService/ListMachines
```

### WatchMachines

A server stream. First, it sends each machine of an org that the caller sees. Then it sends a message with `synced` `true`. Then it sends each change of those machines as it happens.

| Item | Value |
| --- | --- |
| Procedure | `/hopbox.v1.MachineService/WatchMachines` |
| Policy action | `machine.watch`: a caller that acts in the org. The stream holds only the machines that the caller can read with `machine.get`. |
| Idempotent option | no |
| Audit entry | none |

The request `WatchMachinesRequest` has the field `org_id`. Each message `WatchMachinesResponse` holds one field of the oneof `event`:

| Field | Type | Description |
| --- | --- | --- |
| `machine` | `Machine` | A machine as it is now: in the first list, new, or changed. |
| `gone_machine_id` | string | The UUID of a machine that the caller no longer sees, for example after an unshare, or after the leader dropped its row. The stream sends it only for a machine that it sent before. |
| `synced` | bool | `true`. It ends the first list. Every later message is a change. |

The stream follows the change events of the machines of the org ([Change events](#change-events)). For each change, it reads the machine again and sends it as it is then. So several quick changes of one machine can arrive as one message. A machine that its executor deleted comes once more, with the state `deleted`, and then no more. A machine that becomes visible arrives as a `machine` message, for example after its creator shares it.

Every 30 s, the stream checks two things again:

- The credential of the caller. The end of the credential ends the stream with `unauthenticated` and the message `the credential of this stream ended. Sign in again: ...`. Examples are a sign-out of a session, the end of a CLI sign-in, the revoke of an API key, and an expiry. A suspend of the account ends its credentials too ([the admin API](/reference/admin-api/index.html.md#suspendaccount)). The stream also ends at the expiry of its credential, without a wait for the next check ([the API server](/reference/api/index.html.md#streams-and-their-credential)).
- The role of the caller in the org. A caller that left the org, or lost its access in another way, gets `permission_denied`, and the stream ends. After a change of role, the caller can lose the sight of a machine. An example is a private machine of another member, after an admin became a member. Such a machine arrives as `gone_machine_id`.

The stream has no default deadline, so it runs until the client ends it, until its credential ends, or until the listener stops ([the API server](/reference/api/index.html.md#when-a-listener-stops)).

A server stream needs a streaming protocol: gRPC, gRPC-Web, or the Connect streaming protocol with the content type `application/connect+json`. A call with the session cookie must use the Connect protocol ([Sessions, cookies, and sign-in links](/reference/sessions/index.html.md#calls-with-the-cookie)). [Create and manage machines](/guide/machines/index.html.md#watch-the-machines) shows such a call with curl.

| Error | Code |
| --- | --- |
| The control role has no feed of change events: `unavailable: this server streams no machine changes`. The control role of `hopboxd run` always has one ([hopboxd](/reference/hopboxd/index.html.md#run)). | `unavailable` |
| The caller lost its access to the org during the stream | `permission_denied` |
| The credential of the caller ended during the stream, or before it: a sign-out, the end of a CLI sign-in, the revoke of an API key, or an expiry | `unauthenticated` |

### ListMachineSizes

Lists the sizes that a machine can have, smallest first: by vCPUs, then RAM, then name. The list leaves out the retired sizes.

| Item | Value |
| --- | --- |
| Procedure | `/hopbox.v1.MachineService/ListMachineSizes` |
| Policy action | `machine.list_sizes`: `session`, `cli_token`, `member_key`, and `org_key` |
| Idempotent option | no |
| Audit entry | none |

The request `ListMachineSizesRequest` has no fields. The response `ListMachineSizesResponse` has the field `sizes`, a repeated `MachineSize`. An instance admin adds and retires sizes with the admin API ([the admin API](/reference/admin-api/index.html.md#sizeservice)). A user picks a size, never a pool: the pool and the labels of an operator size stay with the instance admin.

```sh
browser -d '{}' $API/hopbox.v1.MachineService/ListMachineSizes
```

```json
{"sizes":[{"name":"default","vcpus":2,"memoryBytes":"8589934592","diskBytes":"100000000000"}]}
```

## Lifecycle

### StartMachine

Sets the target of a machine to `running`, and waits until it runs ([Lifecycle calls](#lifecycle-calls)).

| Item | Value |
| --- | --- |
| Procedure | `/hopbox.v1.MachineService/StartMachine` |
| Policy action | `machine.start`: those who use the machine |
| Idempotent option | no. A second call finds the target `running`, and changes nothing, unless the machine is `failed`. |
| Audit entry | action `machine.start`, target kind `machine`, details `name`, `old_target`, `retry`, and `boot_again`. `retry` is `true` for the start of a `failed` machine. `boot_again` is `true` for the start of a stopped machine with the restart policy `never`. Each of these two starts asks for a new boot. |

The request `StartMachineRequest` names the machine. The response `StartMachineResponse` has the fields `machine`, `duration`, and `done`.

A start is allowed for a machine whose target is `stopped` or `running`. A start locks the org row with `FOR NO KEY UPDATE`, as a create does. A machine whose target is `stopped` holds no vCPUs and no RAM yet. So its start checks these:

1. The usage limits of the org: `limits.Check(usage.AfterStart(size, counted))` ([Domain rules](/reference/domain-rules/index.html.md#usage-limits)).
2. The room on the node of the machine, with the rules of placement: the vCPUs with an overcommit of 2, and the RAM without overcommit ([Placement](/reference/placement/index.html.md#start-of-a-stopped-machine)). The machine and its disk live on that node, so a start cannot go elsewhere in phase 1.

A start of a machine whose current state is `failed` also asks for a new boot, as a reboot does. So the executor tries again, even with the target `running` already. For example, it retries a wake from the kept memory file ([Domain rules](/reference/domain-rules/index.html.md#executor-contract-after-a-failed-wake)). The call then waits for that new boot.

A start of a machine with the restart policy `never` whose current state and target are both `stopped` also asks for a new boot. After a shutdown from inside, the executor keeps such a machine down until a new request ([the node executor](/reference/executor/index.html.md#restart-policy)). The control role sends a spec with the state of the machine at the time of the send. So when a start comes soon after the shutdown, the executor can get the target `running` again, and never the target `stopped`. The new boot request tells the executor to boot the machine. `reboot_pending` is `true` until that boot runs. A second start finds the target `running`, and asks for no other boot.

A start that changes the machine also counts as a use of the machine. It sets `last_activity_time` to now.

| Error | Code |
| --- | --- |
| The target is `standby`, `hibernated`, or `deleted`, for example `failed precondition: machine action not allowed: cannot start a machine whose target state is hibernated` | `failed_precondition` |
| A suspended org: `failed precondition: org suspended: an instance admin suspended the org, so it cannot start a machine` | `failed_precondition` |
| The org is at a limit, for example `usage limit exceeded: the org would use 8 vCPUs, and its limit is 6` | `resource_exhausted` |
| The node of the machine has no room: `no capacity: the node of the machine has too few free vCPUs to start it. Stop another machine, or retry later`, or the same with `too little free RAM` | `unavailable` |

### StopMachine

Sets the target of a machine to `stopped`, and waits until it is stopped.

| Item | Value |
| --- | --- |
| Procedure | `/hopbox.v1.MachineService/StopMachine` |
| Policy action | `machine.stop`: those who use the machine, and the org admins and owners |
| Idempotent option | no. A second call changes nothing. |
| Audit entry | action `machine.stop`, target kind `machine`, details `name` and `old_target` |

The request `StopMachineRequest` names the machine. The response `StopMachineResponse` has the fields `machine`, `duration`, and `done`. A stopped machine keeps its disk and its place in the machine count of the org, and frees its vCPUs and RAM ([the org API](/reference/org-api/index.html.md#usage-limits)).

| Error | Code |
| --- | --- |
| The target is `hibernated`: `failed precondition: machine action not allowed: cannot stop a hibernated machine: wake it first` ([Domain rules](/reference/domain-rules/index.html.md#hibernated-machines-keep-their-memory-file)) | `failed_precondition` |
| The target is `deleted` | `failed_precondition` |

```sh
browser -d '{"name":"box1"}' $API/hopbox.v1.MachineService/StopMachine
```

On a node with an executor, the call returns once the machine stopped:

```json
{"machine":{"id":"01a1460c-1f6e-7b8a-9c42-3d5e7f8a9b10","orgId":"01a14565-8448-7753-9c1f-5ec0a86b8d92","name":"box1","state":"MACHINE_STATE_STOPPED","targetState":"MACHINE_STATE_STOPPED",...,"version":"7",...},"duration":"0.412s","done":true}
```

On an install without an executor, the wait ends after 60 s. The state stays `MACHINE_STATE_PENDING`, and `done` is left out because it is `false`.

The output here is cut short.

### RebootMachine

Boots a running machine again, and waits for the new boot. The call adds 1 to the reboot number of the machine, so `reboot_pending` is `true` until the executor runs the new boot. The target stays `running`.

| Item | Value |
| --- | --- |
| Procedure | `/hopbox.v1.MachineService/RebootMachine` |
| Policy action | `machine.reboot`: those who use the machine |
| Idempotent option | no. Each call asks for one more boot. |
| Audit entry | action `machine.reboot`, target kind `machine`, details `name` |

The request `RebootMachineRequest` names the machine. The response `RebootMachineResponse` has the fields `machine`, `duration`, and `done`.

| Error | Code |
| --- | --- |
| The target is not `running`, or the current state is not `running`, for example `failed precondition: machine action not allowed: cannot reboot a machine that is pending` | `failed_precondition` |
| A suspended org: `failed precondition: org suspended: an instance admin suspended the org, so it cannot reboot a machine` | `failed_precondition` |

### RenameMachine

Gives a machine a new name. The old name stays reserved for the org for 7 days, as the name of a deleted machine does. So its URL never leads to the machine of a stranger. The org can take the old name back during those 7 days. A rename reboots a machine whose current and target state are both `running`, so the guest learns its new name. The call then waits for the new boot.

| Item | Value |
| --- | --- |
| Procedure | `/hopbox.v1.MachineService/RenameMachine` |
| Policy action | `machine.rename`: those who use the machine |
| Idempotent option | no. A second call finds the new name, and changes nothing. |
| Audit entry | action `machine.rename`, target kind `machine`, details `old_name`, `new_name`, and `name` (the old name). Each entry of a machine goes without the refused name details, each of these three that looks like a credential ([Domain rules](/reference/domain-rules/index.html.md#machine-names)). |

| Field | Type | Description |
| --- | --- | --- |
| `id` or `name` | string | The machine ([Naming a machine](#naming-a-machine)). |
| `new_name` | string | The new name ([Domain rules](/reference/domain-rules/index.html.md#machine-names)). |

The response `RenameMachineResponse` has the fields `machine`, `duration`, and `done`. A rename to the name that the machine has already changes nothing, and returns `done` `true` at once. A rename is allowed in a suspended org.

An org holds at most 100 names in reservation (`domain.MaxNameReservations`): the old names of renames and the names of deleted machines, for 7 days each. A rename while the org holds 100 fails, so renames cannot hold back names from other orgs without end. A delete always reserves its name, also beyond 100.

Two renames that swap the names of two machines at the same time lock both name rows in the order of the names ([Control store tables](/reference/control-store/index.html.md#orgs)). So they never deadlock: the first one wins, and the second one finds its new name taken.

| Error | Code |
| --- | --- |
| A bad new name, also a name that looks like a credential, with the message of `CreateMachine` | `invalid_argument` |
| The new name is taken, with a free name in the message and in the detail `MachineNameTaken`, as for `CreateMachine` | `already_exists` |
| The org holds 100 reserved names: `usage limit exceeded: the org holds 100 reserved machine names, the most that it can hold. A name stays reserved for 7 days after its rename or delete` | `resource_exhausted` |
| The target is `deleted` | `failed_precondition` |

### DeleteMachine

Sets the target of a machine to `deleted`, and waits until its executor deleted it. Its name and its SSH port stay reserved for 7 days ([Domain rules](/reference/domain-rules/index.html.md#name-reservation)). Only the same org can take the name back in that time. The machine stops counting in the usage of the org at once.

| Item | Value |
| --- | --- |
| Procedure | `/hopbox.v1.MachineService/DeleteMachine` |
| Policy action | `machine.delete`: those who use the machine, and the org admins and owners |
| Idempotent option | no. A second call fails with `failed_precondition`. |
| Audit entry | action `machine.delete`, target kind `machine`, details `name` |

The request `DeleteMachineRequest` names the machine. The response `DeleteMachineResponse` has the fields `machine`, `duration`, and `done`. The machine has `delete_request_time`.

A delete is allowed from any target but `deleted`. It is the only action that drops the memory file of a hibernated machine ([Domain rules](/reference/domain-rules/index.html.md#hibernated-machines-keep-their-memory-file)). The leader drops the row of the machine 7 days after the delete request, once its executor deleted it ([hopboxd](/reference/hopboxd/index.html.md#leader-jobs)).

| Error | Code |
| --- | --- |
| The target is `deleted` already: `failed precondition: machine action not allowed: cannot delete a machine that is being deleted` | `failed_precondition` |
| A suspended org: `failed precondition: org suspended: an instance admin suspended the org, so it cannot delete a machine` | `failed_precondition` |

```sh
browser -d '{"name":"box1"}' $API/hopbox.v1.MachineService/DeleteMachine
```

## Access calls

The access calls run a command in a machine, copy files in and out of it, and read its log. They are streams. `Exec` streams both ways, `Upload` streams from the client, and `Download` and `StreamMachineLogs` stream from the server. The control role never reaches the guest agent itself. It opens a data stream to the node of the machine over the control channel ([the control channel](/reference/control-channel/index.html.md#machine-streams)). The node runs the call on the guest link of the machine. The code is in `internal/service/machine_access.go`, `internal/api/handlers_machine_access.go`, and `internal/api/websocket.go`.

| Call | Kind | What the machine must be |
| --- | --- | --- |
| `Exec` | bidi stream | running |
| `Upload` | client stream | running |
| `Download` | server stream | running |
| `StreamMachineLogs` | server stream | any state but deleted. A stopped machine keeps its log. |

### Who can call them

Every access call has the policy action `machine.exec`. Only a caller who uses the machine passes: its creator, or for a shared machine, any caller that acts in its org ([Who can do what](#who-can-do-what)). An org admin or an org owner cannot call them on a private machine of another member. An instance admin cannot call them on a machine of an org of which it is not a member.

| Case | Code |
| --- | --- |
| A caller without a credential | `unauthenticated` |
| A member on a private machine of another member | `not_found` |
| An org admin or owner on a private machine of another member | `permission_denied` |
| A machine that is not running, for `Exec`, `Upload`, or `Download` | `failed_precondition`, with the message `failed precondition: the machine <name> is <state>, not running. Start it first` |
| A machine of a suspended org, for `Exec`, `Upload`, or `Download` | `failed_precondition`, with the message `failed precondition: org suspended: an instance admin suspended the org of the machine` |
| A machine whose target or current state is `deleted` | `not_found` |
| The node of the machine has no channel to the control process. Or it has 255 data streams open. Or it did not attach the stream within 10 s. | `unavailable`. A retry can work. |
| The caller has 32 open access calls to the node of the machine (`service.MaxCallerStreams`) | `resource_exhausted`, with the message `too many requests: you have 32 open streams to the node of this machine, the most for one caller. End one, then retry` |
| The org of the machine has 128 open access calls to its node (`service.MaxOrgStreams`) | `resource_exhausted`, with the message `too many requests: the org has 128 open streams to the node of this machine, the most for one org. End one, then retry` |
| The guest agent of the machine is not connected, or the machine stopped during the call | `unavailable` |

### How a stream ends

Each access call runs under the guard of its credential ([the API server](/reference/api/index.html.md#streams-and-their-credential)). It also checks the access of the caller again at each change event of the machine ([Change events](#change-events)), and every 30 s (`service.DefaultCredentialRecheck`):

- The end of the credential ends the call: a sign-out, the end of a CLI sign-in, the revoke of an API key, or an expiry. The code is `unauthenticated`, and the message is `the credential of this stream ended. Sign in again: ...`.
- A loss of the access ends the call: an unshare, a delete, the leave of the caller from the org, or a change of its role. The code is that of the new check, such as `not_found` or `permission_denied`. The message starts with `the access to the machine ended:`. An unshare and a delete end the call at once, because they are change events of the machine. A change of role ends it within 30 s.
- A suspension of the org ends an `Exec`, an `Upload`, or a `Download` with `failed_precondition`. The suspension also stops the machine.
- The stop of the API listener ends each access call with `unavailable` and the message `unavailable: the server shuts down. Call again`. So a read of the log with `follow` never ends with `ok` at a stop.

When a call ends early, the node ends the command or the copy. The guest agent stops the command of an exec: first with SIGTERM, then after 2 s with SIGKILL.

Each check records a use of the machine for its auto-delete timeout ([Auto-delete](#auto-delete)). A check runs at the start of the call, and then every 30 s.

The control role logs these records of the checks, with the field `machine`. The call goes on after each of them:

| Message | Level | Fields |
| --- | --- | --- |
| `cannot record the use of a machine` | warn | `machine`, `err`. The write of the last use failed, at the start of the call or at a check. |
| `the check of the access to a machine failed` | warn | `machine`, `err`. The check itself failed, for example on a control store that does not answer. The next check tries again. |

A client that reads nothing slows the command down, and holds back no other call. Each hop reads the next message only after it handed on the last one: the client, the edge, the control role, the node, and the guest agent. So no hop keeps more than its HTTP/2 window of a stream. The same holds for input: a command that reads no input stalls the client after about 5 MiB, the windows and the input queues of the hops together. The control channel and the guest link send no gzip, so their windows count the bytes of the input, not of a compressed copy. A call that ended early, and whose client still reads nothing 5 s later, ends without its last message.

### Limits

| Limit | Value |
| --- | --- |
| Data of a download | 4 MiB (4,194,304 bytes, `service.DownloadLimit`), for every caller of this release. A download from inside a machine will get 1 GB with the machine tokens of phase 5. |
| Data of an upload | no limit |
| Time without data in an upload | 1 min (`service.UploadIdleTimeout`), counted only while the call waits for the client. A node that takes the data slowly does not count. After the time, the upload ends with `deadline_exceeded`. |
| Open access calls of one caller to one node | 32 (`service.MaxCallerStreams`). The caller is an account. All the org keys of an org count as one caller. |
| Open access calls of one org to one node | 128 (`service.MaxOrgStreams`), for all its callers together. A node has at most 255 data streams for every org together ([the control channel](/reference/control-channel/index.html.md#limits-and-windows-of-the-data-streams)). |
| One `stdin` chunk, or one `data` message of an upload | 1 MiB |
| `argv` of an exec | at most 4096 items and 256 KiB in all. Each item is valid UTF-8 without a NUL byte. The first item is not empty. |
| `env` of an exec | at most 256 env vars and 256 KiB of names and values in all. A name has 1 to 128 characters: letters, digits, and `_`. It does not start with a digit. A value is valid UTF-8 without a NUL byte. Unlike a stored env var, any such name works, also a reserved name or one that looks like a credential ([the env var API](/reference/vars-api/index.html.md)). The caller has a shell in the machine anyway, and the value lasts only for the command. |
| `user`, `term` | at most 256 bytes each, valid UTF-8 without a NUL byte |
| `dir`, `path` | at most 4096 bytes, valid UTF-8 without a NUL byte. `path` is not empty. |
| `timeout` of an exec | 1 s to 24 h, or unset for none |
| Terminal size | 1 to 1000 rows, and 1 to 1000 columns |
| `tail` of `StreamMachineLogs` | 0 to 10000 |

A request that breaks a limit gets `invalid_argument`.

### Exec

Runs a command in a running machine, as the Linux user `user` by default ([Default image](/reference/default-image/index.html.md)). The command gets the env of the machine, then the env vars of the request ([the env var API](/reference/vars-api/index.html.md)).

| Item | Value |
| --- | --- |
| Procedure | `/hopbox.v1.MachineService/Exec` |
| Policy action | `machine.exec` |
| Idempotent option | no |
| Audit entry | none |

Each message `ExecRequest` and `ExecResponse` holds one `ExecChunk` in the field `chunk`. The first chunk from the client holds `start`, and only the first chunk does. The next chunks from the client hold `stdin`, `stdin_eof`, `resize`, or `signal`. The server sends `stdout` and `stderr`, and at the end `exit`, its last chunk. Then the stream ends with the code `ok`. An exec needs a stream in both directions, so it needs HTTP/2, or the WebSocket endpoint.

`ExecChunk` holds one field of the oneof `chunk`:

| Field | Type | From | Description |
| --- | --- | --- | --- |
| `start` | `ExecStart` | the client | The machine and the command. |
| `stdin` | bytes | the client | Input for the command. |
| `stdout` | bytes | the server | Output of the command. With a terminal, all output arrives as `stdout`. |
| `stderr` | bytes | the server | Error output of the command. |
| `resize` | `TerminalSize` | the client | A new size of the terminal. With a guest agent that knows `ExecResponse.control` ([Guest link](/reference/guest-link/index.html.md#exec)), it takes effect before any `stdin` that the client sends after it. An older guest agent can apply it later. |
| `stdin_eof` | bool | the client | No more input. With a terminal, the command reads the end-of-file character, Ctrl-D. |
| `exit` | `ExecExit` | the server | How the command ended. |
| `signal` | string | the client | A signal for the process group of the command, by name, such as `SIGINT` or `SIGTERM`. The guest agent ignores a name that it does not know. With a guest agent that knows `ExecResponse.control` ([Guest link](/reference/guest-link/index.html.md#exec)), it takes effect before any `stdin` that the client sends after it. An older guest agent can apply it later. |

`ExecStart`:

| Field | Type | Description |
| --- | --- | --- |
| `id` or `name` | string | The machine, in the oneof `machine` ([Naming a machine](#naming-a-machine)). |
| `argv` | repeated string | The program and its arguments. The guest agent looks the program up in the `PATH` of the command. Empty runs the login shell of the user as a login shell, as `hopbox connect` does. |
| `env` | map of string to string | Env vars on top of the env of the machine. |
| `user` | string | The Linux user. Empty is `user`. |
| `dir` | string | The working directory. Empty is the home directory of the user. |
| `tty` | bool | Runs the command in a pseudo-terminal. |
| `size` | `TerminalSize` | The first size of the terminal, with `tty`. |
| `term` | string | The `TERM` env var, with `tty`. Empty is `xterm-256color`. |
| `timeout` | `google.protobuf.Duration` | Ends the command after this time. Unset is no timeout. |

`TerminalSize` has `rows` and `cols`, both uint32.

`ExecExit`:

| Field | Type | Description |
| --- | --- | --- |
| `code` | int32 | The exit code. A command that a signal ended has 128 plus the signal number. A program that does not exist has 127. A command that did not start for another reason has 126. A command that its timeout ended has 124. |
| `signal` | string | The name of the signal that ended the command, such as `SIGKILL`, or empty. |
| `error` | string | Why the command did not start, or empty. |
| `timed_out` | bool | The timeout ended the command. |

A client that ends its side of the stream sends no more input, and the command runs on. A client that cancels the call ends the command.

`hopbox connect` opens a shell with `Exec`: `tty` true and no `argv`. Phase 1 has no standby and no hibernate, so a connect to a machine that does not run gets `failed_precondition`. The CLI gives exec its JSON output, its exit status, and its terminal handling ([hopbox CLI](/reference/cli/index.html.md#exec)).

### The WebSocket endpoint

A browser cannot open a stream in both directions with an RPC, so the console uses a WebSocket for exec. The endpoint runs the same exec as `Exec`.

| Item | Value |
| --- | --- |
| Path | `GET /api/ws/exec` on the API listener, over HTTP/1.1 (`api.WebSocketExecPath`) |
| Messages | One `hopbox.v1.ExecChunk` in each binary message, encoded as protobuf. The first message from the client holds `start`. A text message ends the socket with `invalid_argument`. |
| Credentials | The session cookie, or the bearer credentials of the API: a CLI token or an API key in `Authorization: Bearer ...` ([the API server](/reference/api/index.html.md#request-gate)) |
| Origin | A request with an `Origin` header must come from the origin of the install, such as `https://example.com`. Another origin gets HTTP status 403 before the upgrade. The session cookie needs the `Origin` header ([Sessions, cookies, and sign-in links](/reference/sessions/index.html.md#calls-with-the-cookie)). |
| Largest message from the client | 1 MiB plus 64 KiB |
| Wait for the start chunk | 10 s. Then the socket closes with 4003. A socket without a credential closes at once with 4016. |
| Ping | every 30 s. A client that answers no ping within 15 s is gone, and the server ends the socket. The server reads a pong only while it reads the socket. So the pings pause while the command takes its input slowly and a message of the client waits for it. A pause lasts at most 2 min. Then the pings count again, so a client that vanished meanwhile loses its socket. |
| Compression | none |

The server ends the socket with a close frame:

| Close code | When |
| --- | --- |
| 1000 | The exec ended, after the `exit` chunk. |
| 1001 | The API listener stops. The reason is `the server shuts down. Connect again`. |
| 4000 plus a Connect code | The exec failed, for example 4016 for `unauthenticated`, 4007 for `permission_denied`, 4005 for `not_found`, or 4009 for `failed_precondition`. The reason is the message of the error, at most 123 bytes. |

A close of the socket by the client ends the command, because a WebSocket has no half close. The `api call` log record of the socket has the procedure `/api/ws/exec` and the protocol `websocket` ([the API server](/reference/api/index.html.md#logs)).

### Upload

Writes a file into a running machine, or extracts a tar stream into a directory.

| Item | Value |
| --- | --- |
| Procedure | `/hopbox.v1.MachineService/Upload` |
| Policy action | `machine.exec` |
| Idempotent option | no |
| Audit entry | none |

The first message `UploadRequest` holds `start`. The next ones hold `data`. The end of the client stream is the end of the data. A client stream also works over HTTP/1.1 with the Connect protocol.

`UploadStart`:

| Field | Type | Description |
| --- | --- | --- |
| `id` or `name` | string | The machine, in the oneof `machine`. |
| `path` | string | The target. A relative path starts at the home directory of the user. |
| `archive` | bool | The data is a tar stream of the contents of a directory. When the directory `path` does not exist, the guest agent makes it. Then it extracts the stream into the directory. Otherwise the data is the contents of one file. |
| `mode` | uint32 | The permission bits of a file, such as `0755`. 0 is `0644`. An archive carries its own. Larger than `07777` gets `invalid_argument`. |
| `user` | string | The Linux user that owns the new files, and whose home directory a relative path starts at. Empty is `user`. |

`UploadResponse` has `bytes`, the count of bytes that the machine got. An error of the guest agent, such as a directory that the user cannot write, ends the call with `failed_precondition` and the message `failed precondition: the machine: <error>`.

### Download

Reads a file or a directory from a running machine. A directory arrives as a tar stream of its contents.

| Item | Value |
| --- | --- |
| Procedure | `/hopbox.v1.MachineService/Download` |
| Policy action | `machine.exec` |
| Idempotent option | no |
| Audit entry | none |

`DownloadRequest`:

| Field | Type | Description |
| --- | --- | --- |
| `id` or `name` | string | The machine, in the oneof `machine`. |
| `path` | string | The source. A relative path starts at the home directory of the user. |
| `user` | string | The Linux user whose home directory a relative path starts at. Empty is `user`. |

The first message `DownloadResponse` holds `info`. The next ones hold `data`, at most 32 KiB each.

`DownloadInfo`:

| Field | Type | Description |
| --- | --- | --- |
| `archive` | bool | The data is a tar stream of a directory. Otherwise it is the contents of one file. |
| `mode` | uint32 | The permission bits of the file or the directory. |
| `size` | int64 | The size of a file in bytes. 0 for an archive. |

| Error | Code and message |
| --- | --- |
| A file larger than 4 MiB | `failed_precondition`, before any data: `failed precondition: the download has more than 4194304 bytes, the most that a download to a laptop can have. ...` |
| A directory whose tar stream grows past 4 MiB | the same error, after the data up to the limit. Drop the data that came. |
| A missing file, or one that the user cannot read | `failed_precondition`, with the message `failed precondition: the machine: <error>` |

### StreamMachineLogs

Sends the log of a machine: the output of its start command, line by line ([Machine logs](/reference/machine-logs/index.html.md)). The node of the machine keeps the log for 7 days or 100 MiB, whichever ends first.

| Item | Value |
| --- | --- |
| Procedure | `/hopbox.v1.MachineService/StreamMachineLogs` |
| Policy action | `machine.exec` |
| Idempotent option | no |
| Audit entry | none |

`StreamMachineLogsRequest`:

| Field | Type | Description |
| --- | --- | --- |
| `id` or `name` | string | The machine, in the oneof `machine`. |
| `tail` | uint32 | Sends only the last `tail` entries. 0 sends all of them. |
| `since` | `google.protobuf.Timestamp` | Sends only the entries from this time on. Unset sends all of them. With `tail`, the stream sends the last `tail` entries from `since` on. |
| `follow` | bool | Keeps the stream open after the old entries, and sends each new entry as it arrives. |

Each message `StreamMachineLogsResponse` holds one field of the oneof `event`: `entry`, a `MachineLogEntry`, or `synced`, which is `true`. Only a stream with `follow` sends `synced`, once, after the old entries. A stream without `follow` ends after the old entries.

`MachineLogEntry`:

| Field | Type | Description |
| --- | --- | --- |
| `time` | `google.protobuf.Timestamp` | When the guest agent read the line. |
| `stream` | `MachineLogStream` | `MACHINE_LOG_STREAM_STDOUT` or `MACHINE_LOG_STREAM_STDERR` |
| `data` | bytes | The line without its newline. A line longer than 16 KiB arrives in parts. |
| `dropped` | uint64 | The count of lines that were lost before this one, because a buffer of the guest agent or of the node was full. |

A machine that runs again adds to the same log. The delete of the machine drops its log.

## Sharing and settings

The calls of this section change the machine at once and do not wait. Each one returns the machine after the change, in the field `machine` of its response. A request that asks for what the machine has already changes nothing, writes no audit entry, and keeps the version. Each call fails with `failed_precondition` on a machine whose target is `deleted`.

### ShareMachine

Opens a private machine to the whole org. Every member and every org key of the org can then use it. A shared machine stays in the org after its creator leaves.

| Item | Value |
| --- | --- |
| Procedure | `/hopbox.v1.MachineService/ShareMachine` |
| Policy action | `machine.share`: the creator only |
| Idempotent option | no. A second call changes nothing. |
| Audit entry | action `machine.share`, target kind `machine`, details `name` and `shared` (`true`) |

The request `ShareMachineRequest` names the machine. The response `ShareMachineResponse` has the field `machine`. On a machine whose target is `deleted`, the error is `failed precondition: machine action not allowed: cannot machine.share a machine that is being deleted`.

A share fails closed. In its transaction, the call resolves the env that the machine gets as a shared machine: the shared values and the all values of its org, with each secret open ([the env var API](/reference/vars-api/index.html.md#env-of-a-machine)). If that env does not resolve, the call fails with `failed_precondition`. The machine then stays private, at the same version, with its old env. The call writes no audit entry and publishes no event. The message names the cause:

| Cause | Message |
| --- | --- |
| The keyring has no key-encryption key. | `failed precondition: the env of the machine as a shared machine does not resolve, so the machine stays private: the keyring has no key-encryption key to seal or open the value. Run hopboxd admin kek rotate to make the first one` |
| The keyring dir or a key file is not safe. | `failed precondition: the env of the machine as a shared machine does not resolve, so the machine stays private: seal: <path> is not safe to hold key-encryption keys: ...`. The rest says which check failed ([Sealing](/reference/sealing/index.html.md#errors)). |
| The keyring does not open, for example after a read error of its dir. | `failed precondition: the env of the machine as a shared machine does not resolve, so the machine stays private: service: open the keyring: seal: ...`. The rest says why, for example an error of the operating system or a key file of the wrong size. |
| A shared or an all secret does not open. | `failed precondition: the env of the machine as a shared machine does not resolve, so the machine stays private: service: open the secret <name> of org <org ID>: seal: ...`. The rest is an error of [Sealing](/reference/sealing/index.html.md#errors), for example `the sealed value does not open: it was changed, or it belongs to another place`. |

A private value never reaches a shared machine. So a private secret of the creator that does not open does not stop the share. A failed read of the control store is not a refusal of this kind: the call fails with `internal`, and a retry can work. After a refused share, fix the cause, and share again. For example, put back the key file of the key-encryption key, or set the secret again ([the executor protocol](/reference/executor-protocol/index.html.md#a-machine-whose-env-does-not-resolve)).

### UnshareMachine

Makes a shared machine private to its creator again. The other members stop seeing it: a watch of theirs gets its `gone_machine_id`.

| Item | Value |
| --- | --- |
| Procedure | `/hopbox.v1.MachineService/UnshareMachine` |
| Policy action | `machine.unshare`: the creator only |
| Idempotent option | no. A second call changes nothing. |
| Audit entry | action `machine.unshare`, target kind `machine`, details `name` and `shared` (`false`) |

The request `UnshareMachineRequest` names the machine. The response `UnshareMachineResponse` has the field `machine`.

An unshare never checks the env. If the env of the machine as a private machine does not resolve, the unshare still works. The machine then keeps its old env, which holds only the shared values and the all values of its org. Every member of the org already uses those values in the shared machines, so the old env shows nothing new to anyone.

A machine that an org key made has no creator account, so no account can own it alone. It stays shared, and its unshare fails with `failed_precondition` and the message `failed precondition: an org key made this machine, and no account owns it, so it stays shared`. A shared machine whose creator account was deleted, or whose creator org key was revoked, stays shared too. Its unshare fails with `failed precondition: the account or the org key that made this machine is gone, so it stays shared`. A revoke of an org key never stops or deletes the machines that it made.

### UpdateMachine

Changes the restart policy, the start command, or the auto-delete timeout of a machine. A field that the request leaves out keeps its value. The executor applies a new restart policy at once, and a new start command at the next boot ([the node executor](/reference/executor/index.html.md#the-guest-link)).

| Item | Value |
| --- | --- |
| Procedure | `/hopbox.v1.MachineService/UpdateMachine` |
| Policy action | `machine.update`: those who use the machine |
| Idempotent option | no. A second call with the same request changes nothing. |
| Audit entry | action `machine.update`, target kind `machine`, details `name` and one detail for each field of the request. `restart_policy` is the new policy. `start_command_changed` is `true` or `false`. `auto_delete_timeout` is the new timeout as Go writes a duration, for example `1h0m0s`, or `0s` for off. The entry never holds the start command. |

| Field | Type | Description |
| --- | --- | --- |
| `id` or `name` | string | The machine ([Naming a machine](#naming-a-machine)). |
| `restart_policy` | `RestartPolicy` | The new restart policy. `RESTART_POLICY_UNSPECIFIED` keeps it. |
| `start_command` | optional string | The new start command. Leave it out to keep it. An empty string removes it. |
| `auto_delete_timeout` | `google.protobuf.Duration` | The new auto-delete timeout. Leave it out to keep it. A zero duration turns it off. |
| `expected_version` | optional int64 | The `version` of the machine that the caller read. If the machine has another version now, the call fails with `aborted` and changes nothing. The version also grows without a request, for example at each report of the executor or after an env change. So a call can fail with `aborted` although no other request came in between. Read the machine again and retry. |

The response `UpdateMachineResponse` has the field `machine`. A new auto-delete timeout counts from the time of the update: the call sets `last_activity_time` to now. So a new timeout never deletes a machine at once for its idle time before the update. An update is allowed in a suspended org.

| Error | Code |
| --- | --- |
| A request that sets no field: `invalid argument: the update sets no field` | `invalid_argument` |
| A start command or a timeout that breaks its rule ([Domain rules](/reference/domain-rules/index.html.md#start-command)) | `invalid_argument` |
| A stale `expected_version`: `conflict: the machine is at version <n>, not <m>. Read it again` | `aborted` |
| The target is `deleted`: `failed precondition: machine action not allowed: cannot update a machine that is being deleted` | `failed_precondition` |

```sh
browser -d '{"name":"box1","startCommand":"python3 -m http.server 8000","autoDeleteTimeout":"86400s","expectedVersion":"2"}' \
  $API/hopbox.v1.MachineService/UpdateMachine
```

### SetMachineURLPort

Sets the port in the machine that its URL reaches. The default is 8000.

| Item | Value |
| --- | --- |
| Procedure | `/hopbox.v1.MachineService/SetMachineURLPort` |
| Policy action | `machine.set_url_port`: the creator only |
| Idempotent option | no. A second call changes nothing. |
| Audit entry | action `machine.set_url_port`, target kind `machine`, details `name`, `url_port`, `old_url_port`, `url_visibility`, and `old_url_visibility` |

| Field | Type | Description |
| --- | --- | --- |
| `id` or `name` | string | The machine. |
| `port` | int32 | The port in the machine, 1 to 65535. |

The edge sends the requests of the URL `https://<name>.<domain>` to this port of the machine ([the edge role](/reference/edge/index.html.md#routing)). The change event of the change pushes the routes to the edge servers again, also for a running machine ([the control channel](/reference/control-channel/index.html.md#edge-pushes)).

The response `SetMachineURLPortResponse` has the field `machine`. A port out of range gets `invalid_argument` with the message `invalid argument: invalid machine config: the URL port <port> is not 1 to 65535`. On a machine whose target is `deleted`, the error is `failed precondition: machine action not allowed: cannot change the URL of a machine that is being deleted`.

### SetMachineURLVisibility

Makes the URL of a machine public or private. A new machine is private.

| Item | Value |
| --- | --- |
| Procedure | `/hopbox.v1.MachineService/SetMachineURLVisibility` |
| Policy action | `machine.set_url_visibility`: the creator only |
| Idempotent option | no. A second call changes nothing. |
| Audit entry | action `machine.set_url_visibility`, target kind `machine`, details `name`, `url_port`, `old_url_port`, `url_visibility`, and `old_url_visibility` |

| Field | Type | Description |
| --- | --- | --- |
| `id` or `name` | string | The machine. |
| `visibility` | `URLVisibility` | `URL_VISIBILITY_PRIVATE` or `URL_VISIBILITY_PUBLIC`. |

A public URL needs no sign-in at the edge. A private URL sends a browser without a valid machine cookie to the sign-in ([the edge role](/reference/edge/index.html.md#private-machine-urls)). The change reaches the edge servers with the next push of the routes. A change to private also tells the edges to check each private URL again.

The response `SetMachineURLVisibilityResponse` has the field `machine`. `URL_VISIBILITY_UNSPECIFIED` or an unknown value gets `invalid_argument` with the message `invalid argument: the URL visibility must be private or public`. On a machine whose target is `deleted`, the error is the one of `SetMachineURLPort`.

## Suspension

An instance admin can suspend an org ([the org API](/reference/org-api/index.html.md#suspension)). The suspension stops the machines of the org, and leaves a hibernated machine hibernated. The rule of each lifecycle action comes from `Action.CheckSuspended()` ([Domain rules](/reference/domain-rules/index.html.md#org-suspension)):

| Call | In a suspended org |
| --- | --- |
| `CreateMachine`, `StartMachine`, `RebootMachine` | blocked, with `failed_precondition` and the message `failed precondition: org suspended: an instance admin suspended the org, so it cannot <action> a machine` |
| `DeleteMachine` | blocked, because a suspension keeps the data of the org |
| `StopMachine`, `RenameMachine` | allowed |
| `GetMachine`, `ListMachines`, `WatchMachines`, `ListMachineSizes`, `ShareMachine`, `UnshareMachine`, `UpdateMachine`, `SetMachineURLPort`, `SetMachineURLVisibility`, `StreamMachineLogs` | allowed |
| `Exec`, `Upload`, `Download` | blocked, with `failed_precondition` and the message `failed precondition: org suspended: an instance admin suspended the org of the machine`. A suspension also ends the open ones ([How a stream ends](#how-a-stream-ends)). |

The machine URLs of a suspended org are blocked too: the edge answers `403`, and `GET /auth/machine` gives no code ([the edge role](/reference/edge/index.html.md#routing)).

The suspend of an account suspends its personal org and each org that it owns alone ([the admin API](/reference/admin-api/index.html.md#suspendaccount)). In each other org, it sets the target `stopped` on the private machines of the account, and leaves a hibernated one hibernated. Such an org is not suspended, so the rules of the table do not apply there. A private machine stays stopped until its creator starts it again.

Instance admins can stop and delete any machine of a suspended org through the admin API ([the admin API](/reference/admin-api/index.html.md#machineservice)). The auto-delete job of the leader skips the machines of a suspended org.

## Usage and limits

A machine counts in the usage of its org ([the org API](/reference/org-api/index.html.md#usage-limits)):

| Usage | The machines that count |
| --- | --- |
| Machines and disk | every machine whose target is not `deleted` |
| vCPUs and RAM | the machines whose target is `running` or `standby` |

`CreateMachine` and the start of a stopped machine check the limits under the lock of the org row. The other calls do not add to the usage, so they take only a shared lock of the org row ([Control store tables](/reference/control-store/index.html.md#orgs)).

## Auto-delete

A machine with an auto-delete timeout goes after that time without use. The leader checks every minute ([hopboxd](/reference/hopboxd/index.html.md#leader-jobs)). The use is `last_activity_time`. Today these set it: the create, each start that changes the machine, and each update with a new auto-delete timeout. The access calls set it too: each `Exec`, `Upload`, `Download`, and `StreamMachineLogs` at its start, and then every 30 s while it runs ([How a stream ends](#how-a-stream-ends)). SSH sessions set it, and so does inbound traffic: each heartbeat of the node names the machines with inbound traffic ([the node network](/reference/node-network/index.html.md#inbound-traffic-as-a-use)). An SSH session calls `Service.TouchMachine` of `internal/service` after its own policy check. `TouchMachine` writes at most once a minute for each machine, and does not change the `version` of the machine. So `last_activity_time` can be up to a minute older than the last use. The job deletes a machine only once `last_activity_time` is older than its timeout plus 1 minute (`service.TouchInterval`). So a machine that someone used within its timeout always stays. The job reads the idle machines in pages of 100, in the order of their last use. It goes on after a machine that fails, so a few bad machines never hold back the others. The job deletes the machine as `DeleteMachine` does, so its name and its SSH port stay reserved for 7 days. It skips the machines of a suspended org. Its audit entry has the actor `system`.

## Audit entries

The machine calls write these entries in the transaction of the change. Their org is the org of the machine, so its owners and admins read them with `ListAuditEntries` ([the sign-in, account, and audit API](/reference/auth-api/index.html.md#listauditentries)). The target kind is `machine`, and the target ID is the machine ID. Each entry has the detail `name`: the name of the machine before the change. The actor is the caller, except for the auto-delete, whose actor is `system`.

| Action | Written by | Details besides `name` |
| --- | --- | --- |
| `machine.create` | `CreateMachine` | `size`, `image` (the digest), `shared` |
| `machine.start` | `StartMachine` | `old_target`, `retry`, `boot_again` |
| `machine.stop` | `StopMachine`, and `SuspendOrg` and `SuspendAccount` of the admin API for each machine that they stop | `old_target`, and `reason` for a stop of the admin API. The reason is `org suspended` in an org that the call suspends, and `account suspended` for a private machine of the account in another org. |
| `machine.reboot` | `RebootMachine` | none |
| `machine.rename` | `RenameMachine` | `old_name`, `new_name` |
| `machine.delete` | `DeleteMachine`, the calls of [What a member leaves behind](/reference/org-api/index.html.md#what-a-member-leaves-behind), and the auto-delete | `reason` for a delete by hopbox or by another call. It is `member left` (`LeaveOrg`, `RemoveOrgMember`, `DeleteAccount`), `org deleted` (`DeleteOrg`, `DeleteAccount` for the personal org), or `auto-delete`. The auto-delete adds `auto_delete_timeout` and `last_activity_at` (RFC 3339, UTC). |
| `machine.share` | `ShareMachine` | `shared` (`true`) |
| `machine.unshare` | `UnshareMachine` | `shared` (`false`) |
| `machine.update` | `UpdateMachine` | `restart_policy`, `start_command_changed`, and `auto_delete_timeout`, each one only for a field that the request sets |
| `machine.set_url_port` | `SetMachineURLPort` | `url_port`, `old_url_port`, `url_visibility`, `old_url_visibility` |
| `machine.set_url_visibility` | `SetMachineURLVisibility` | the same as `machine.set_url_port` |

A call that changes nothing writes no entry. The read calls write none. The admin calls on machines write `machine.list_all`, `machine.admin_stop`, `machine.admin_delete`, `machine.move`, `machine.cancel_move`, and `machine.list_moves` ([the admin API](/reference/admin-api/index.html.md#machineservice)). Each sign-in through the SSH front door writes `machine.ssh`, with the actor `ssh_key` ([the SSH front door](/reference/ssh-front-door/index.html.md#audit)). `Exec`, `Upload`, and `Download` write no entry.

## Change events

Each change of a machine publishes a change event in its transaction ([Control store tables](/reference/control-store/index.html.md#events)). The control role feeds the events to the watches and the waits of its process. So a watch or a wait sees each change, also one that another control process or a node made.

| Kind | Published by | `org_id` | `subject_id` | `payload` |
| --- | --- | --- | --- | --- |
| `machine.created` | `CreateMachine` | the org | the machine | `{"action":"machine.create"}` |
| `machine.updated` | each other change that writes an audit entry. The calls of this page make such changes. So do [What a member leaves behind](/reference/org-api/index.html.md#what-a-member-leaves-behind), `SuspendOrg`, and the admin calls. The auto-delete and a machine that stopped by itself make them too. `SetVar` and `DeleteVar` publish it for each machine that the value reaches. | the org | the machine | `{"action":"<the audit action>"}`, for example `{"action":"machine.stop"}` |
| `machine.state_changed` | each report of the executor that changes the current state ([the executor protocol](/reference/executor-protocol/index.html.md#machinereport)) | the org | the machine | `{"action":"server.report_machine"}` |

On each of these events, each control process also pushes the routes of the machine URLs to its edge servers. It tells them to check each private URL again too ([the control channel](/reference/control-channel/index.html.md#edge-pushes)).

The payload carries only the action. A reader reads the machine again for its state. A report of a machine with the restart policy `never` that stopped by itself also sets the target `stopped`. That change publishes `machine.updated` with `{"action":"machine.stop"}`.
