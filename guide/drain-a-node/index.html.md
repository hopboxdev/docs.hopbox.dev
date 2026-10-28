# Drain a node

> How to drain a node so that its machines move to other nodes and the node leaves the install, what the states and the fields of a drain mean, and what to do with a stranded machine.

A drain takes every machine off a node, so that the node can leave the install. The drain moves each machine cold to another node. Every machine keeps its disk, its name, its URL, its SSH port, and its address. When the node holds nothing, the drain decommissions it.

This guide drains the server `node1`, which holds the machines `box1`, `box2`, and `box3`. [Cold move](/reference/cold-move/index.html.md#drain-of-a-node) covers the mechanics, and [the admin API](/reference/admin-api/index.html.md#drainserver) covers the call.

## Why drain

A decommission refuses a server that holds machines. A delete of the machines loses their disks. A drain is the way to keep them. It uses the cold move, so a running machine stops, moves, and boots again. It loses its memory, and keeps its disk ([Cold move](/reference/cold-move/index.html.md)). A drain never deletes a machine.

A machine that cannot move stays on the node with its disk: a running machine with no room keeps running, a machine whose move failed stays stopped. The drain names it. The node stays in the install until you fix the reason.

## What you need

If Terraform manages the server, run `terraform destroy` instead of the commands of this guide. The provider drains the server first ([Drain with Terraform](#drain-with-terraform)). After a drain outside Terraform, the next refresh drops a decommissioned node from the Terraform state, and fails on the first server.

- A server with the role `node`. A server without it holds no machine, and the drain refuses it. Decommission such a server with `hopboxd admin server decommission`.
- At least one other `ready` server of the same pool, with room for the machines of `node1`. Placement picks the target of each machine ([Placement](/reference/placement/index.html.md)).
- Root on a control server. Every command below runs with `sudo`.
- Every control server on a release that knows migration 51. While an older control process is live, the drain refuses to start.

## Run a drain

1. List the servers, and copy the ID of `node1`:

   ```text
   $ sudo hopboxd admin server list
   ID                                    POOL     STATE  STATUS  ROLES  HOSTNAME  VERSION  LAST HEARTBEAT        PEER ADDRESS
   01a14404-ce7b-72d9-b18e-5006c3e462db  default  ready  ok      node   node1     v0.1.0   2026-10-16T09:00:05Z  -
   01a14405-0c11-7a3e-90d2-2b1f6d6a77aa  default  ready  ok      node   node2     v0.1.0   2026-10-16T09:00:06Z  -
   ```

2. Start the drain. The command returns at once:

   ```text
   $ sudo hopboxd admin server drain 01a14404-ce7b-72d9-b18e-5006c3e462db
   id	01a14404-ce7b-72d9-b18e-5006c3e462db
   ...
   state	draining
   cordoned	true
   ...
   machine_count	3
   ...
   drain_start_time	2026-10-16T09:10:00Z
   drain_moving	0
   drain_empty_checks	0
   drain_last_check_time	
   drain_error	
   drain_stuck	false
   ```

3. Follow the drain with `server get`. A leader job takes the drain one step on every 5 s. It starts at most 2 moves at once:

   ```text
   $ sudo hopboxd admin server get 01a14404-ce7b-72d9-b18e-5006c3e462db | grep -E '^(state|machine_count|drain)'
   state	draining
   machine_count	1
   drain_start_time	2026-10-16T09:10:00Z
   drain_moving	1
   drain_empty_checks	0
   drain_last_check_time	
   drain_error	
   drain_stuck	false
   ```

4. When `machine_count` is 0 and no move runs, the first empty check turns the state to `shutting_down`:

   ```text
   state	shutting_down
   machine_count	0
   drain_moving	0
   drain_empty_checks	1
   drain_last_check_time	2026-10-16T09:12:30Z
   ```

5. The third empty check, one minute after the second, decommissions the server. The `state` reads `decommissioned`, and the drain lines are gone:

   ```text
   $ sudo hopboxd admin server get 01a14404-ce7b-72d9-b18e-5006c3e462db | grep -E '^(state|drain)'
   state	decommissioned
   ```

The machines `box1`, `box2`, and `box3` are now on `node2`. `hopboxd admin machine list` shows their new node.

## The states

| State | Meaning |
| --- | --- |
| `draining` | The drain cordoned the server and moves its machines. The server takes no new machine. |
| `shutting_down` | The drain found the server empty. It waits for 3 empty checks in a row, one minute apart. |
| `unresponsive` | The server sent no heartbeat for 20 s. The drain waits. It goes on at the next heartbeat, and the state turns `draining` again. |
| `decommissioned` | The drain decommissioned the server. |

A server in `shutting_down` turns `draining` again when a check finds it not empty, or when you run the drain again. A server that drains stays cordoned until its decommission, so `hopboxd admin server uncordon` refuses it:

```text
$ sudo hopboxd admin server uncordon 01a14404-ce7b-72d9-b18e-5006c3e462db
hopboxd: error: failed_precondition: failed precondition: the server 01a14404-ce7b-72d9-b18e-5006c3e462db drains, and stays cordoned until the drain decommissions it. A drain cannot stop. Follow it with hopboxd admin server get, and fix the reasons of its stranded machines. For a server that Terraform manages, run terraform destroy, which waits for the drain
```

## The fields of a drain

`server get` prints these lines while the server drains:

| Line | Meaning |
| --- | --- |
| `drain_start_time` | The start of the drain, or of its last retry. |
| `drain_moving` | The moves from the server that run now. |
| `drain_empty_checks` | The empty checks in a row: 0 to 3. |
| `drain_last_check_time` | The time of the last empty check. Empty before the first one. |
| `drain_error` | Why the drain waits now, such as `the machine box2 has not booted yet, and moves once it runs`. Empty while it goes on. |
| `drain_stuck` | `true` when the drain cannot go on by itself. |
| `drain_stranded` | One line for each machine that the drain cannot move: `<name> <machine id>: <reason>`. |

With `--json`, the key `drain` holds the same facts. It is `null` for a server that does not drain.

A server is empty when it holds no machine, no allocation in its last heartbeat, no move, no old copy that it must still remove, and no copy of a disk that a failed move left. A check that finds anything else restarts the count from zero.

## Fix a stranded machine

A stranded machine stays on the server, with its disk. The drain does not try it again until you run the drain again. Here `box3` is stranded:

```text
$ sudo hopboxd admin server get 01a14404-ce7b-72d9-b18e-5006c3e462db | grep -E '^(state|drain)'
state	draining
drain_start_time	2026-10-16T09:10:00Z
drain_moving	0
drain_empty_checks	0
drain_last_check_time	
drain_error	
drain_stuck	true
drain_stranded	box3 01a14406-1f2e-7b44-8c10-5d3a9e0b2c11: no capacity: no node has room for a machine of size ...
```

1. Read the reason.
2. Make room, or fix the reason. For `no capacity`, enroll another server or delete a machine on another server.
3. Run the drain again:

   ```sh
   sudo hopboxd admin server drain 01a14404-ce7b-72d9-b18e-5006c3e462db
   ```

   The retry clears the stranded list, restarts `drain_start_time`, and counts the empty checks from zero.

These are the reasons:

| Reason | What to do |
| --- | --- |
| `no capacity: ...` | Make room on another server, then run the drain again. |
| `a cold move needs a machine that booted, with the target running or stopped. This one is <state>, with the target <target>` | The machine never booted, is hibernated, or is in standby. Start it, stop it, or delete it. Then run the drain again. |
| `its move failed: <error of the move>` | The machine is stopped, with its disk on the node. Fix the error of the move, such as a node that cannot take the disk. Then run the drain again. The machine boots again on a start. |

A drain never deletes a machine. A machine that cannot move stays on the node with its disk: a running machine with no room keeps running, a machine whose move failed stays stopped.

## A kept copy of a disk

A failed move can leave the machine on another node while this server keeps a copy of its disk that nothing removes ([Cold move](/reference/cold-move/index.html.md#a-step-that-an-older-control-process-accepted)). The drain strands that machine too, with the reason `a failed move left the machine <state> on the node <id>, and this server keeps a copy of its disk that nothing removes (reference/cold-move.md)`. Its `drain_stranded` line looks like the others. With `--json`, its entry of `drain.stranded` has `on_server` set to `false`, because the machine is on another node. A decommission refuses the server while it keeps that copy:

```text
failed_precondition: failed precondition: the server 01a14404-ce7b-72d9-b18e-5006c3e462db keeps a copy of the disk of the machine box3, which a failed move left stopped on another node. Delete the machine, or keep the server
```

Delete the machine to end the reason. Or keep the server in the install.

## A refused decommission

The third empty check runs the decommission with all its checks. A refusal shows in `drain_error`, and the drain tries again one minute later. For example, the last control server cannot leave while other servers belong to the install:

```text
drain_empty_checks	3
drain_error	failed precondition: the server 01a14404-ce7b-72d9-b18e-5006c3e462db shares its host with the last live control process of the install, and 2 other servers still belong to the install. Decommission them first, or start the control role on another server
drain_stuck	true
```

Fix the cause, and the next check decommissions the server.

## Limits

The drain of this release has these limits:

- The drain does not try a stranded machine again by itself, also when room appears later. Run the drain again, or run `terraform destroy` again, to try it.
- You cannot stop or cancel a drain. A server that drains stays cordoned until its decommission, and `hopboxd admin server uncordon` refuses it.
- The empty checks need a server that sends heartbeats. If an empty server stops, the drain waits with `the server sends no heartbeat, so the drain waits for it`. To take out such a server, run `hopboxd admin server decommission`, which takes an unresponsive server that holds nothing.
- A server that keeps a copy of a disk after a failed move stays in the install until the machine of that copy is deleted ([A kept copy of a disk](#a-kept-copy-of-a-disk)). No call gives such a machine back to this server yet.

## Take only the node role

A drain takes the whole server out. To take only the machines off a server and keep its other roles, remove its role `node` instead. An example is an edge server. Run `sudo hopboxd admin server remove-role <id> node`. The removal moves the machines cold as a drain does, at most 2 at once. It names each machine that cannot move. No decommission comes at the end. Unlike a drain, a removal can stop: `sudo hopboxd admin server cancel-remove-role <id>` ends it, and the machines that moved stay on their new nodes ([Cold move](/reference/cold-move/index.html.md#removal-of-the-node-role)). A drain and a removal exclude each other on one server.

## Drain with Terraform

When Terraform destroys a `hopbox_server`, the hopbox Terraform provider drains the server first, and waits until the drain decommissions it ([hopbox Terraform provider](/reference/terraform-provider/index.html.md#drain-of-a-server)). If a machine cannot move, `terraform destroy` fails and names the machine with its reason. The server then stays in the install and in the Terraform state. Fix the reason, then run `terraform destroy` again. The drain goes on meanwhile, and can decommission the server by itself. The resource then stays in the Terraform state, and the next `terraform destroy` stops `hopboxd` on the server and removes the secrets of the install from it. The OVHcloud server stays rented after the destroy.

## Use the Admin area

The Admin area has a button for the drain ([Use the Admin area](/guide/admin-area/index.html.md#drain-a-server)).

1. Open the page "Servers", and click the name of `node1`.
2. Click "Drain". The dialog "Drain node1" explains the drain. It says that a running machine stops, moves, and boots again, so it loses its memory. It says that you cannot stop a drain.
3. Click "Drain". The state changes to `draining`. The page reads again every 2 s, and its section "Drain" shows Started, Moves, Empty checks, and Waits. A table lists each stranded machine with its reason.
4. After you fix a reason, click "Drain again".

There is no way to stop a drain. A server that drains stays cordoned until its decommission. The dialog also says to destroy a server that Terraform manages with Terraform. The page has no button "Drain" for a server without the role `node`. A session cannot drain a server that hosts a control process: the call fails with `permission denied: the server <id> hosts a control process, ...`. Drain such a server as root on a control server, or with `terraform destroy`.
