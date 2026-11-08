# Standby and hibernate

> How to put a machine in standby and bring it back, and what happens when a machine loses its standby.

This guide shows you how to put a machine in standby and how to bring it back. Standby keeps the memory of a machine in the RAM of its node. A machine in standby does not run, and it goes on from the same place at the resume. Processes keep their PIDs, and data in RAM stays. Hibernate comes in a later milestone of phase 3.

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

If the guest agent does not connect again, the machine stays in the state `standby`, although its guest runs. It leaves that state when its link comes back.

A machine in standby refuses a new name, because a rename reboots a running machine. Resume the machine first. The refusal also covers a machine on its way in or out of standby.

## A machine in standby on a drained node

A machine in standby does not move, because its memory lives in the RAM of its node. A drain strands such a machine until you resume it or stop it. A later milestone of phase 3 moves such a machine with its memory. You cannot pause a machine while it moves to another node.

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

## Errors

| Error | Cause | Fix |
| --- | --- | --- |
| `failed_precondition: failed precondition: machine action not allowed: cannot pause a machine whose target state is stopped` | Only a running machine goes to standby. | Start the machine first. |
| `failed_precondition: failed precondition: machine action not allowed: cannot resume a machine whose target state is running` | The machine is not in standby, or it lost its standby. | Read its state with `hopbox machine show box1`. |
| `failed_precondition: failed precondition: the machine box1 is in standby. Resume it first` | An exec, a file copy, or `hopbox connect` found the machine in standby. | Run `hopbox machine resume box1`. |
| `failed_precondition: failed precondition: machine action not allowed: cannot rename a machine in standby: resume it first` | A rename found the machine in standby, or on its way in or out of it. | Run `hopbox machine resume box1`, then rename the machine. |
| `failed_precondition: failed precondition: the machine box1 moves to another node. Pause it after the move` | An instance admin or a drain moves the machine. | Wait for the end of the move, and pause again. |
| `failed_precondition: failed precondition: the node of the machine box1 runs hopboxd <version>, whose heartbeats name no feature standby. An instance admin must upgrade the node first` | The node runs an older release. | Ask an instance admin to upgrade the node. |

## See also

- [The machine API](/reference/machine-api/index.html.md#pausemachine) for `PauseMachine` and `ResumeMachine`.
- [hopbox CLI](/reference/cli/index.html.md#machine-pause) for the commands.
- [The node executor](/reference/executor/index.html.md#standby) for the steps on the node.
