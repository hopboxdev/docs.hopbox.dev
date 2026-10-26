# Create and manage machines

> How to register the default image, then create, list, watch, stop, and delete a machine through the API, and how the operator sees and stops the machines of every org.

This guide registers the default image, then creates, lists, watches, stops, and deletes a machine through the API. At the end, the operator sees and stops the machines of every org with `hopboxd admin`. [The machine API](/reference/machine-api/index.html.md) has every call and rule, and [Placement](/reference/placement/index.html.md) covers how a create picks a node.

The executor of a machine on its node boots and stops its microVM with Firecracker ([the node executor](/reference/executor/index.html.md), [the Firecracker driver](/reference/firecracker-driver/index.html.md)). Each machine gets an address on the network of its node, where its URL and its SSH port reach it ([Check the network of a machine](/guide/node-network/index.html.md)). [Run commands and copy files in a machine](/guide/exec-and-files/index.html.md) runs commands in a machine over its guest link.

This guide calls the API with curl, in the same way as the console does, so it shows each call. The console does the same work in a browser ([Use the console](/guide/console/index.html.md#create-a-machine-and-open-a-shell)). The hopbox CLI does the same with `hopbox new`, `hopbox list`, and `hopbox machine` ([Use machines from the CLI](/guide/cli-machines/index.html.md)).

The release files of hopbox are not public yet. The hopbox team gives them on request.

## What you need

- A Linux server with the control role and the node role, enrolled as in [Enroll the node role on one server](/guide/enroll-node/index.html.md). `sudo hopboxd admin server list` must show the server `ready`, with the status `ok`. A create needs a ready node with room for the size `default`: 2 vCPUs and 8 GiB of RAM ([Placement](/reference/placement/index.html.md#hard-filters)). Without one, it fails with `unavailable`.
- A domain and a session cookie of a signed-in account, as in [First sign-in on a new install](/guide/first-sign-in/index.html.md). Keep the cookie in `COOKIE`, in the form `__Host-hopbox_session=<token>`.
- The default image. A server installed from the release bundle registers the default image of its release at each start, so it needs no more step ([Install hopbox on a Debian 13 server by hand](/guide/install-debian/index.html.md)). Another build of the image needs its `artifact.json` on the server, and its compressed disk at a URL that the nodes reach. The people who work on hopbox build one with `make image` into `bin/image/`.

A Mac cannot be the node, because it has no KVM. On a Mac with only the control role, every create fails with `no capacity: ... The install has no node yet`.

Define a function that calls the API with the headers of the console. The API listens on `control.api_listen`, by default `127.0.0.1:7080`:

```sh
API=http://127.0.0.1:7080
browser() {
  curl -s -X POST -H 'Content-Type: application/json' -H 'Connect-Protocol-Version: 1' \
    -H 'Origin: https://example.com' -H "Cookie: $COOKIE" "$@"
  echo
}
```

## Register the default image

A create takes the newest image of a name, `default` by default. A control server that runs a release registers the default image of that release at each start, unless the install has a default image from a newer release ([hopboxd](/reference/hopboxd/index.html.md#the-default-image-of-the-release)). A control role without a release, such as `hopboxd` on a laptop, has no image, so a create fails:

```json
{"code":"failed_precondition","message":"failed precondition: the install has no image \"default\". An instance admin registers one with hopboxd admin image register"}
```

Register the image from its `artifact.json`. The install keeps only the metadata. `--url` tells the nodes where to download the compressed disk:

```text
$ sudo hopboxd admin image register artifact.json --url https://dl.example.org/images/default/disk.raw.zst
id	01a1460a-7c21-7d4e-8f90-1a2b3c4d5e6f
name	default
digest	sha256:422d31d7a4fd5e6958162b22e3d772f71b5a1465cbd6255d253ae8c087873dc8
registered	true
```

The digest is the sha256 of the bytes of `artifact.json`, so it names this one build. A second register of the same file changes nothing, and prints `registered	false`. A new build of the image has a new digest. Once you register it, new machines take it, and the machines that exist keep their image.

```text
$ sudo hopboxd admin image list
01a1460a-7c21-7d4e-8f90-1a2b3c4d5e6f	default	sha256:422d31d7a4fd5e6958162b22e3d772f71b5a1465cbd6255d253ae8c087873dc8	2026-10-16T16:01:40Z
```

## See the sizes

```sh
browser -d '{}' $API/hopbox.v1.MachineService/ListMachineSizes
```

```json
{"sizes":[{"name":"default","vcpus":2,"memoryBytes":"8589934592","diskBytes":"100000000000"}]}
```

A fresh install has one size. A create without a size takes `default`.

## Create a machine

A create takes an idempotency key, so a retry with the same key returns the same machine and makes no second one. Keep the key in a variable:

```sh
KEY=$(uuidgen)
browser -H "Idempotency-Key: $KEY" -d '{"name":"box1"}' $API/hopbox.v1.MachineService/CreateMachine
```

```json
{"machine":{"id":"01a1460c-1f6e-7b8a-9c42-3d5e7f8a9b10","orgId":"01a14565-8448-7753-9c1f-5ec0a86b8d92","name":"box1","state":"MACHINE_STATE_PENDING","targetState":"MACHINE_STATE_RUNNING","size":{"name":"default","vcpus":2,"memoryBytes":"8589934592","diskBytes":"100000000000"},"imageName":"default","imageDigest":"sha256:422d31d7a4fd5e6958162b22e3d772f71b5a1465cbd6255d253ae8c087873dc8","creatorAccountId":"01a14565-8448-774f-9317-ea6348e9a930","restartPolicy":"RESTART_POLICY_ALWAYS","url":"https://box1.example.com","urlPort":8000,"urlVisibility":"URL_VISIBILITY_PRIVATE","sshPort":21467,"lastActivityTime":"2026-10-16T16:02:11.402518Z","stateChangeTime":"2026-10-16T16:02:11.402518Z","version":"1","createTime":"2026-10-16T16:02:11.402518Z","updateTime":"2026-10-16T16:02:11.402518Z"}}
```

The request leaves out `orgId`, so the machine is in the personal org of the account. The create picked the node, granted it the lease of the machine, and claimed the name `box1` and a random SSH port. It did not wait: the machine is `pending`, with the target `running`. The node boots it at once. At the first boot of an image, the node also unpacks the image, so the machine runs about 25 s later on the [tested server](/reference/preflight/index.html.md#the-tested-server).

Send the same request with the same key again. The response is the same, with the header `Idempotent-Replayed: true`.

Machine names are unique in the whole install. A new create with the name `box1` fails, and suggests a free name:

```sh
browser -H "Idempotency-Key: $(uuidgen)" -d '{"name":"box1"}' $API/hopbox.v1.MachineService/CreateMachine
```

```json
{"code":"already_exists","message":"already exists: the machine name \"box1\" is taken. A free name is \"box2\""}
```

A create without a name gets the first free name of `box1`, `box2`, and so on. Each org has usage limits, by default 3 machines, 6 vCPUs, 24 GiB of RAM, and 300 GB of disk ([the org API](/reference/org-api/index.html.md#usage-limits)). A fourth machine of the size `default` fails with `resource_exhausted`.

If no node has room, the create fails with `unavailable` and names the reason of each node:

```json
{"code":"unavailable","message":"no capacity: no node has room for a machine of size default (2 vCPUs, 8 GiB of RAM): 1 node is cordoned, unresponsive, or decommissioned"}
```

The failed create leaves nothing behind. Uncordon the node, or enroll another one, then retry.

## List and read the machines

```sh
browser -d '{}' $API/hopbox.v1.MachineService/ListMachines
browser -d '{"name":"box1"}' $API/hopbox.v1.MachineService/GetMachine
```

`ListMachines` returns the machines of the org that you see, by name. A machine is private to its creator until the creator shares it with `ShareMachine`. Other members of the org do not see a private machine, and get `not_found` for it. An org admin or owner sees it, and can stop or delete it, but cannot start it or change it ([the machine API](/reference/machine-api/index.html.md#who-can-do-what)).

## Watch the machines

`WatchMachines` is a server stream. It sends each machine, then `{"synced":true}`, then each change as it happens. A call with the session cookie must use the Connect protocol, so use curl with the streaming form of Connect. Each message of the stream has a 5-byte prefix: one byte of flags and a 4-byte length. The request below sends the message `{}` with its prefix. The small Python program prints each message of the response on one line:

```sh
printf '\000\000\000\000\002{}' | curl -sN -X POST \
    -H 'Content-Type: application/connect+json' -H 'Connect-Protocol-Version: 1' \
    -H 'Origin: https://example.com' -H "Cookie: $COOKIE" \
    --data-binary @- $API/hopbox.v1.MachineService/WatchMachines |
  python3 -c '
import struct, sys
r = sys.stdin.buffer
while (h := r.read(5)):
    print(r.read(struct.unpack(">I", h[1:])[0]).decode(), flush=True)
'
```

```text
{"machine":{"id":"01a1460c-1f6e-7b8a-9c42-3d5e7f8a9b10",...,"name":"box1","state":"MACHINE_STATE_RUNNING","targetState":"MACHINE_STATE_RUNNING",...}}
{"synced":true}
```

The output here is cut short. Leave the watch running, and stop the machine in a second terminal. The watch prints the machine again, with its new target, and then with its new state:

```text
{"machine":{"id":"01a1460c-1f6e-7b8a-9c42-3d5e7f8a9b10",...,"name":"box1","state":"MACHINE_STATE_RUNNING","targetState":"MACHINE_STATE_STOPPED",...}}
{"machine":{"id":"01a1460c-1f6e-7b8a-9c42-3d5e7f8a9b10",...,"name":"box1","state":"MACHINE_STATE_STOPPED","targetState":"MACHINE_STATE_STOPPED",...}}
```

Press Ctrl-C to end the watch. The stream has no deadline. A watch of an org that you leave ends with `permission_denied` within 30 s.

## Stop the machine

A stop sets the target `stopped`, then waits for the machine, for up to 60 s:

```sh
browser -d '{"name":"box1"}' $API/hopbox.v1.MachineService/StopMachine
```

```json
{"machine":{"id":"01a1460c-1f6e-7b8a-9c42-3d5e7f8a9b10",...,"state":"MACHINE_STATE_STOPPED","targetState":"MACHINE_STATE_STOPPED",...},"duration":"0.512s","done":true}
```

The node asks the guest to shut down, and the call returns as soon as the machine is stopped. If the wait ends first, `done` is `false`, and the JSON codec leaves it out. The call still succeeded then: the target is `stopped`, and the machine goes on toward it. A machine with the restart policy `never` can also stop by itself, for example with `sudo reboot` inside it. Its target then turns `stopped`, so it stays stopped until a start.

To wait less, send a shorter deadline. The wait ends 1 s before it:

```sh
browser -H 'Connect-Timeout-Ms: 5000' -d '{"name":"box1"}' $API/hopbox.v1.MachineService/StartMachine
```

The start sets the target `running` again, and the node boots the machine from its disk. The call returns once the machine runs, or after about 4 s with `done` false. A stopped machine holds no vCPUs and no RAM. So the start first checks the limits of the org and the room on the node of the machine.

## Manage the machines as the operator

An instance admin sees the machines of every org, with metadata only. Root on a control server calls the admin API with `hopboxd admin`:

```text
$ sudo hopboxd admin machine list
01a1460c-1f6e-7b8a-9c42-3d5e7f8a9b10	box1	org=01a14565-8448-7753-9c1f-5ec0a86b8d92	running	target=running	server=01a14403-6bd1-7522-9ab0-b3ab4517e0e4
```

`--org-id` lists the machines of one org. To handle abuse, stop or delete any machine, also in a suspended org:

```text
$ sudo hopboxd admin machine stop --name box1
id	01a1460c-1f6e-7b8a-9c42-3d5e7f8a9b10
name	box1
state	stopped
target_state	stopped
duration	512ms
done	true
```

Each `admin` command waits at most 40 s for its call, so its wait ends after about 39 s at the latest. The audit log of the org records the entry `machine.admin_stop` of `local_root`, so the owners of the org see it.

A suspension of the org stops each of its machines too ([the org API](/reference/org-api/index.html.md#suspension)). The members of a suspended org cannot create, start, reboot, or delete a machine until the suspension ends.

## Delete the machine

```sh
browser -H 'Connect-Timeout-Ms: 5000' -d '{"name":"box1"}' $API/hopbox.v1.MachineService/DeleteMachine
```

The machine gets the target `deleted` and a `deleteRequestTime`. The node stops its microVM, and removes its disk and its jail. It stops counting in the usage of the org at once. Its name and its SSH port stay reserved for your org for 7 days, so another org cannot take `box1` in that time. Your org can: a new create with the name `box1` succeeds. A second delete of the same machine fails with `failed_precondition`.

## Read the audit log

Each change writes an audit entry in the org of the machine: `machine.create`, `machine.stop`, `machine.start`, `machine.admin_stop`, and `machine.delete` here ([the machine API](/reference/machine-api/index.html.md#audit-entries)). The owner of the personal org reads them:

```sh
browser -d '{"orgId":"01a14565-8448-7753-9c1f-5ec0a86b8d92","pageSize":5}' $API/hopbox.v1.AuditService/ListAuditEntries
```

To run commands in the machine and copy files, see [Run commands and copy files in a machine](/guide/exec-and-files/index.html.md).
