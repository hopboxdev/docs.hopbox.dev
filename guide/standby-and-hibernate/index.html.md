# Standby and hibernate

> How to put a machine in standby and bring it back, and what happens when a machine loses its standby. Also how to hibernate and wake a machine, what a failed wake leaves, how to drop a memory file, and how an operator protects the memory files on a node.

This guide shows you how to put a machine in standby or hibernate it, and how to bring it back. Standby keeps the memory of a machine in the RAM of its node. Hibernate saves that memory to a memory file on the disk of its node, and frees the RAM. In both cases, the machine does not run. At its return, it goes on from the same place. Processes keep their PIDs, and data in RAM stays.

| | Standby | Hibernate |
| --- | --- | --- |
| Command | `hopbox machine pause` | `hopbox machine hibernate` |
| Back with | `hopbox machine resume` | `hopbox machine wake` |
| The memory | stays in the RAM of the node | goes to a memory file on the disk of the node |
| vCPUs and RAM in the limits of the org | count | do not count |
| A reboot of the node | ends the memory | keeps the memory file |

You need:

- The `hopbox` CLI, signed in ([Use the hopbox CLI](/guide/cli/index.html.md)).
- A running machine, here `box1`.

## Put a machine in standby

Run this command on your computer:

```sh
hopbox machine pause box1
```

The command waits until the machine is in standby, and prints:

```text
Paused the machine box1 in <time>.
```

`hopbox list` now shows `box1` with the state `standby`. In the console, each running machine has the button Pause.

A pause has these effects:

- Each exec and each terminal of the machine ends with an error.
- `hopbox exec`, `hopbox connect`, and a file copy fail with `the machine box1 is in standby. Resume it first`.
- The URL and the SSH port of the machine refuse connections, as for a machine that does not run. A later milestone of phase 3 makes a connection resume the machine.
- The machine still holds its RAM on its node. So it counts in the vCPU and RAM limits of its org, as a running machine does ([Create and manage machines](/guide/machines/index.html.md)).

## Bring a machine back

Run this command on your computer:

```sh
hopbox machine resume box1
```

The command waits until the guest agent of the machine connected, and prints:

```text
Resumed the machine box1 in <time>.
```

A resume boots nothing, so the boot ID of the machine stays, and the start command does not run again. In the console, a machine in standby has the button Resume.

You can also stop or delete a machine in standby. The guest then resumes for a moment, and shuts down cleanly.

If the guest agent does not connect again, the machine stays in the state `standby`, although its guest runs. Once its link comes back, it leaves that state.

A machine in standby refuses a new name, because a rename reboots a running machine. Resume the machine first. The refusal also covers a machine on its way in or out of standby.

## A machine in standby on a drained node

A machine in standby does not move, because its memory lives in the RAM of its node. A drain strands such a machine until you resume it or stop it. A later milestone of phase 3 moves such a machine with its memory. You cannot pause a machine while it moves to another node. A hibernated machine does not move either, because its memory file lives on its node. A drain strands it until you wake it, stop it after a wake, or drop its memory file.

## What the wall clock does

A guest in standby does not run, so its wall clock stops. After a resume, the clock of the guest stays behind until the guest syncs it. The default image has no time service. A later milestone of phase 3 sets the clock at each resume.

## When a machine loses its standby

The memory of a machine in standby lives only in the RAM of its node. A reboot or a crash of the node ends that RAM, and so does a kill of the unit of the machine. The machine then loses what its memory held. Its disk stays. The restart policy of the machine decides the next step:

| Restart policy | What happens |
| --- | --- |
| `always` | The machine boots from its disk. |
| `never` | The machine stays down until you start it. |

So a machine with the restart policy `never` stays down after a reboot of its node. The audit log of the org holds an entry with the actor `system` and the reason `the RAM of the machine in standby is gone`. A planned reboot of the node shuts each guest down cleanly first, so data that the guest wrote to its disk stays.

Keep data that must survive a reboot on the disk.

## Hibernate a machine

Run this command on your computer. The machine can run or be in standby:

```sh
hopbox machine hibernate box1
```

The command waits until the node saved the memory of the machine to its memory file, and prints:

```text
Hibernated the machine box1 in <time>.
```

The save writes the whole RAM of the machine, so a machine with more RAM takes longer. `hopbox list` now shows `box1` with the state `hibernated`, and `hopbox machine show box1` shows the line `memory file  yes`. In the console, a running machine and a machine in standby have the button Hibernate.

A hibernate has these effects:

- Each exec and each terminal of the machine ends with an error.
- `hopbox exec`, `hopbox connect`, and a file copy fail, because the machine does not run.
- The URL and the SSH port of the machine refuse connections. A later milestone of phase 3 makes a connection wake the machine.
- The machine frees its vCPUs and its RAM, so they do not count in the limits of its org. Its disk and its memory file stay on its node.
- A reboot of the node keeps the memory file. The machine stays hibernated.

A hibernated machine refuses a stop, because a stop does not keep the memory. Wake it first, or drop the memory file ([Drop the memory file](#drop-the-memory-file)).

A hibernated machine also refuses a new name, because a wake runs no reboot, so the guest keeps its old host name. Wake the machine first, then rename it.

## Wake a machine

Run this command on your computer:

```sh
hopbox machine wake box1
```

The command waits until the guest agent of the machine connected, and prints:

```text
Woke the machine box1 in <time>.
```

A wake boots nothing, so the boot ID of the machine stays, and the start command does not run again. The node drops the memory file once the guest agent connected, so `hopbox machine show box1` no longer shows the line `memory file  yes`. In the console, a hibernated machine has the button Wake in place of Start.

Until the load runs, the machine stays `hibernated`, and `hopbox machine stop box1` refuses with `cannot stop a machine that is hibernated: wait for the end of its wake`. Wait for the wake, then stop the machine. If an instance admin suspends the org meanwhile, the wake ends, and the machine stays hibernated with its memory file. Wake it again once the org is active.

A hibernated machine holds no vCPUs and no RAM. So a wake takes the checks of a start: the limits of the org, and the room on the node of the machine. A wake refused for one of these reasons leaves the machine hibernated. Stop another machine of the org, or wait for room on the node, and wake again.

### The clock of a woken guest

At a wake, the guest gets the wall clock of the host from the save. KVM puts the wall clock into the save only on a host with the clock source `tsc`. Read the clock source of a node with this command:

```sh
cat /sys/devices/system/clocksource/clocksource0/current_clocksource
```

On a host with another clock source, such as the CI runner, the save goes without the wall clock. A guest that wakes from such a save keeps the clock of the save. Milestone T04 of phase 3 sets the clock after a load ([the Firecracker driver](/reference/firecracker-driver/index.html.md#the-guest-clock-after-a-load)). ks reads `tsc`, so a guest that wakes on ks gets the right time.

## A failed wake

A wake can fail. For example, the memory file can be cut short, or hold damaged bytes. The guest agent then does not connect within 60 s of the load, and the node ends the machine. A failed wake never drops the memory file. The command then fails with status 6:

```text
hopbox: the machine box1 is not running after <time>. It is failed. hopbox machine show box1 shows its state later
```

The command waits at most 60 s, as long as the node waits for the guest agent. So the message can also name the state `starting`. Then the node ends the machine a moment later. `hopbox machine show box1` then shows the state `failed` and the line `memory file  yes`. The machine keeps its disk and its memory file. With the restart policy `always`, the node tries the wake again after a backoff. With the restart policy `never`, the machine stays down.

You have three ways on:

1. Try the wake again. Run `hopbox machine start box1`. A start of a failed machine loads the same memory file again. Each later wake also puts back the disk as it was at the first wake, so the disk matches the memory.
2. Stop the machine, and keep the memory file for later. Run `hopbox machine stop box1`. A later `hopbox machine start box1` tries the wake again.
3. Drop the memory file, and boot from the disk ([Drop the memory file](#drop-the-memory-file)).

A node can keep the memory file of a failed machine. Until the file goes, an instance admin cannot move the machine to another node, and a drain strands it ([Drain a node](/guide/drain-a-node/index.html.md)).

## Drop the memory file

Only the creator of the machine can drop its memory file. The machine must not run: it is hibernated, or its wake failed. Run this command on your computer:

```sh
hopbox machine discard-memory box1
```

The command asks first:

```text
Drop the memory file of the machine box1? Its programs and their data in RAM end, and its next start boots from its disk. [y/N]
```

Answer `y`. The command waits until the machine is stopped, and prints:

```text
Dropped the memory file of the machine box1 in <time>.
```

The machine is now `stopped`. Before the drop, the node puts back the disk as it was at the save. So a failed wake leaves no change on the disk. Start the machine from its disk:

```sh
hopbox machine start box1
```

In a script, add `--yes` to skip the question. In the console, the creator sees the button "Discard memory" on the page of the machine. The audit log of the org holds the entry `machine.discard_memory`.

If the command refuses with `cannot discard the memory of a machine that is starting`, the node tries the wake again at that moment. Run `hopbox machine stop box1`, wait until the machine is stopped, and run `hopbox machine discard-memory box1` again.

## The memory file on the node

This section is for the operator of the install. A memory file holds the whole RAM of a machine in plain bytes: its secrets, its keys, and its open files. hopbox protects the file only with file modes. The dir `<data_dir>/machines/<id>/state/` belongs to root with mode 0700, and each file has mode 0600 ([the Firecracker driver](/reference/firecracker-driver/index.html.md#files-of-a-machine)). Firecracker reads the memory file as plain bytes, so hopbox cannot encrypt it file by file. The disk of a machine has the same exposure.

So put `/var/lib/hopbox` on an encrypted block device, such as a LUKS volume, on each node. Then a disk that leaves the server holds no plain memory files.

A drop of a memory file, by a wake, by `discard-memory`, or by a delete, unlinks the file. The bytes can stay on the disk of the node until the file system reuses the blocks. With the trim timer of Debian, `fstrim.timer`, a trim of the file system follows.

## Errors

| Error | Cause | Fix |
| --- | --- | --- |
| `failed_precondition: failed precondition: machine action not allowed: cannot pause a machine whose target state is stopped` | Only a running machine goes to standby. | Start the machine first. |
| `failed_precondition: failed precondition: machine action not allowed: cannot resume a machine whose target state is running` | The machine is not in standby, or it lost its standby. | Read its state with `hopbox machine show box1`. |
| `failed_precondition: failed precondition: the machine box1 is in standby. Resume it first` | An exec, a file copy, or `hopbox connect` found the machine in standby. | Run `hopbox machine resume box1`. |
| `failed_precondition: failed precondition: machine action not allowed: cannot rename a machine in standby: resume it first` | A rename found the machine in standby, or on its way in or out of it. | Run `hopbox machine resume box1`, then rename the machine. |
| `failed_precondition: failed precondition: machine action not allowed: cannot rename a hibernated machine: wake it first` | A rename found the machine hibernated, or on its way in or out of it. | Run `hopbox machine wake box1`, then rename the machine. |
| `failed_precondition: failed precondition: the machine box1 moves to another node. Pause it after the move` | An instance admin or a drain moves the machine. | Wait for the end of the move, and pause again. |
| `failed_precondition: failed precondition: the node of the machine box1 runs hopboxd <version>, whose heartbeats name no feature standby. An instance admin must upgrade the node first` | The node runs an older release. | Ask an instance admin to upgrade the node. |
| `failed_precondition: failed precondition: machine action not allowed: cannot hibernate a machine whose target state is stopped` | Only a running machine or a machine in standby hibernates. | Start the machine first. |
| `failed_precondition: failed precondition: machine action not allowed: cannot wake a machine whose target state is running` | The machine is not hibernated, or a wake of it failed. | Read its state with `hopbox machine show box1`. After a failed wake, run `hopbox machine start box1` ([A failed wake](#a-failed-wake)). |
| `failed_precondition: failed precondition: machine action not allowed: cannot stop a hibernated machine: wake it first` | A stop does not keep the memory of a hibernated machine. | Run `hopbox machine wake box1`, or drop the memory file with `hopbox machine discard-memory box1`. |
| `failed_precondition: failed precondition: machine action not allowed: cannot stop a machine that is hibernated: wait for the end of its wake. After a failed wake, drop the file with DiscardMemory` | A wake of the machine is pending, and its load did not start yet. | Wait for the wake, then stop the machine. If the wake fails, run `hopbox machine discard-memory box1`. |
| `failed_precondition: failed precondition: the machine box1 is hibernated, not running. Start it first` | An exec, a file copy, or `hopbox connect` found the machine hibernated. | Run `hopbox machine wake box1`. |
| `resource_exhausted: usage limit exceeded: ...` | The wake takes the org over a limit. | Stop another machine of the org, or ask an owner of the org for a higher limit. |
| `unavailable: no capacity: the node of the machine has too little free RAM to start it. Stop another machine, or retry later` | The node of the machine has no room for it now. | Retry later. |
| `failed_precondition: failed precondition: the machine has no memory file` | `discard-memory` found no memory file, as after a wake that worked. | None. The machine has no memory file to drop. |
| `failed_precondition: failed precondition: machine action not allowed: cannot discard the memory of a machine that is running, with the target running: only a machine with no microVM drops its memory file` | The machine runs. | Nothing to drop: a wake that worked dropped the file. |
| `permission_denied: permission denied: ...` on `discard-memory` | Only the creator of the machine drops its memory file. | Ask the creator. |
| `failed_precondition: failed precondition: the node of the machine box1 runs hopboxd <version>, whose heartbeats name no feature hibernate. An instance admin must upgrade the node first` | The node runs an older release. | Ask an instance admin to upgrade the node. |

## See also

- [The machine API](/reference/machine-api/index.html.md#pausemachine) for `PauseMachine` and `ResumeMachine`, and [HibernateMachine](/reference/machine-api/index.html.md#hibernatemachine), [WakeMachine](/reference/machine-api/index.html.md#wakemachine), and [DiscardMemory](/reference/machine-api/index.html.md#discardmemory).
- [hopbox CLI](/reference/cli/index.html.md#machine-pause) for the commands.
- [The node executor](/reference/executor/index.html.md#standby) for the steps on the node, and [Saved state](/reference/executor/index.html.md#saved-state) for a hibernate and a wake.
- [The Firecracker driver](/reference/firecracker-driver/index.html.md#hibernate-and-wake) for the files of a memory file on the node.
