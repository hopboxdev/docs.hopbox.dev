# Standby and hibernate

> How to put a machine in standby and bring it back, and what happens when a machine loses its standby. Also how to hibernate and wake a machine, why a hibernate fails, what a failed wake leaves, how to drop a memory file, and how an operator protects the memory files and the free disk of a node.

This guide shows you how to put a machine in standby or hibernate it, and how to bring it back. Standby keeps the memory of a machine in the RAM of its node. Hibernate saves that memory to a memory file on the disk of its node, and frees the RAM. In both cases, the machine does not run. At its return, it goes on from the same place. Processes keep their PIDs, and data in RAM stays.

| | Standby | Hibernate |
| --- | --- | --- |
| Command | `hopbox machine pause` | `hopbox machine hibernate` |
| Back with | `hopbox machine resume` | `hopbox machine wake` |
| The memory | stays in the RAM of the node | goes to a memory file on the disk of the node |
| vCPUs and RAM in the limits of the org | count | do not count |
| The saved state of the org | nothing | the memory file counts, and the save counts in the daily budget of saves |
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
- The next `hopbox exec`, `hopbox connect`, file copy, request of its URL, or SSH session resumes the machine ([A connection brings the machine back](#a-connection-brings-the-machine-back)).
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

A guest in standby does not run. The default image has no time service. So at each resume, the guest agent compares the clock of the guest with the clock of the node. If the two differ by more than 1 s, the guest agent sets the clock of the guest, and its log names the jump ([hopbox-guest](/reference/hopbox-guest/index.html.md#logs)). Read that log in the machine:

```sh
journalctl -u hopbox-guest | grep 'set the clock from the node'
```

A guest agent of v0.2.0-rc.1 or older never sets the clock. A machine from an image with such a guest agent keeps the clock that the guest kernel reads after a resume, until a program in the machine sets it. After a pause of 60 s, that clock was 0.65 s behind in a test ([Guest link](/reference/guest-link/index.html.md#clock-and-randomness)).

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
- The next `hopbox exec`, `hopbox connect`, file copy, request of its URL, or SSH session wakes the machine ([A connection brings the machine back](#a-connection-brings-the-machine-back)).
- The machine frees its vCPUs and its RAM, so they do not count in the limits of its org. Its disk and its memory file stay on its node.
- The memory file counts in the stored state of the org, and the save counts in the saves of the last 24 hours. `hopbox org usage` shows both ([hopbox CLI](/reference/cli/index.html.md#org-usage)).
- A reboot of the node keeps the memory file. The machine stays hibernated.

### When a hibernate fails

Two checks can refuse a hibernate. In both cases, the machine runs on, or stays in standby, and the command fails with status 6:

- The daily budget of saves of the org. The save writes up to the whole RAM of the machine, so the check counts the RAM. A machine with 8 GiB of RAM needs 8 GiB of the budget. The stored state of the org never refuses a hibernate.
- The free disk of the node. A save needs a floor of free disk on the node after it, 10% of the disk and 50 GiB by default. The node refuses a save below that floor.

Check the budget of the org:

```sh
hopbox org usage
```

The row `saves in 24 hours` shows the saves of the last 24 hours and the budget. A save leaves the budget 24 hours after it. Try the hibernate again later, or ask an instance admin for a higher budget. For a refused save of the node, ask an instance admin to free disk on the node.

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

On a host with another clock source, such as the CI runner, the save goes without the wall clock ([the Firecracker driver](/reference/firecracker-driver/index.html.md#the-guest-clock-after-a-load)). On every host, the guest agent sets the clock of the guest from the node when its guest link comes up after the wake, as after a resume ([What the wall clock does](#what-the-wall-clock-does)).

### Randomness after a second load

A wake that fails can leave a guest that ran from the memory file. The next wake loads the same file again, so two runs start from one state of the random number generator. On each such load, the node sends fresh random bytes, and the guest agent reseeds the kernel with them before the machine reports `running` ([the node executor](/reference/executor/index.html.md#the-randomness-step)). A hibernate does not freeze the programs of the machine. So on such a load, the programs of the machine run again before the step, from the moment of the load until the guest agent links. The guest kernel reseeds itself at the load through VMGenID, so only that short time runs without the step.

If the step fails in the guest, the machine runs anyway. The node logs `the guest agent failed the randomness step, so the next link asks for it again` at level error, and counts the failure in `hopbox_guest_randomness_steps_total{result="failed"}`. Each new guest link of the machine asks for the step again, also after a restart of `hopboxd`, until the guest agent reports it. A guest agent of v0.2.0-rc.1 or older cannot do this step, and the guest kernel then reseeds only through VMGenID.

## Idle timers put a machine to sleep by themselves

Each machine has two idle timers. After its auto-standby timeout without inbound traffic, a running machine goes to standby. After its auto-hibernate timeout without inbound traffic, a machine hibernates, from running or from standby. A new machine hibernates after 4 hours without inbound traffic, and has no auto-standby, as on boxd. A machine from before the upgrade to the release with the timers has both timers off.

Inbound traffic is a request of the URL, an SSH session, an exec, a file copy, or a terminal. An open connection counts for as long as it is open. CPU use does not count. So a machine that only runs a job, such as a build or a crawler, with no inbound traffic, goes to sleep after its timeout, and the job stops with it.

If a machine runs such a job, turn its auto-hibernate timer off. Run this command on your computer:

```sh
hopbox machine update box1 --auto-hibernate=off
```

To put a machine in standby after 30 minutes without inbound traffic, and hibernate it after 2 hours, run this command:

```sh
hopbox machine update box1 --auto-standby=30m --auto-hibernate=2h
```

A new timer counts from the time of the change. `hopbox machine show box1` shows both timers on the lines `auto-standby` and `auto-hibernate`. To set the timers at the create, give the same flags to `hopbox new`. In the console, the section Idle timers of a machine sets both, in minutes, with 0 for off.

The timers act 1 to 2 minutes after their time, because hopbox records a use at most once a minute, and checks the timers once a minute. The audit log of the org holds an entry `machine.pause` or `machine.hibernate` with the actor `system` and the reason `auto-standby` or `auto-hibernate`. A connection brings the machine back, as after a pause or a hibernate by hand ([A connection brings the machine back](#a-connection-brings-the-machine-back)).

A timer does not act in these cases:

- The org is suspended.
- The machine moves to another node.
- The node of the machine cannot pause or save. On a node that can pause but not save, only the auto-standby timer acts. The auto-hibernate timer does nothing there.
- A control process of the install runs a release before hibernate. Only the auto-standby timer acts until that process is upgraded.
- An edge server of the install runs an older release. The timers wait until each edge server is upgraded.

A hibernate of a timer takes the checks of `hopbox machine hibernate` ([When a hibernate fails](#when-a-hibernate-fails)). If the daily budget of saves or the free disk of the node refuses it, the machine runs on, and the timer counts again from the refusal.

## A connection brings the machine back

You do not need to resume or wake a machine by hand. A connection that passes its check of access brings the machine back:

| Connection | Check of access |
| --- | --- |
| `hopbox exec`, `hopbox connect`, `hopbox cp`, and the web terminal | Your rights on the machine, as for `hopbox machine resume` |
| An SSH session through the SSH port of the machine | An SSH key of an account with access to the machine |
| A request of a private URL | A sign-in of an account with access to the machine |
| A request of a public URL | None. Any request passes. |

The connection waits until the machine runs, so you see only a slower start. For SSH, the login takes longer. `hopbox list`, `hopbox machine show`, and `hopbox logs` do not bring a machine back. A scan of the SSH ports does not either, because a key that opens nothing passes no check.

Run this command on your computer to try it:

```sh
hopbox machine hibernate box1
curl -s -o /dev/null -w '%{http_code} %{time_total}\n' https://box1.example.com/
```

For a public URL with a server on the URL port of the machine, curl prints `200`, and the time includes the wake. A private URL sends a client without a sign-in to the sign-in, and wakes nothing.

A wake on a connection takes the checks of `hopbox machine wake`: the limits of the org, and the room on the node. A public URL never wakes a private machine whose creator an instance admin suspended. A refused wake leaves the machine as it was:

- A request of the URL gets `503 Service Unavailable` with a page that starts with `The machine box1 did not wake:` and names the reason. On a public URL, only a visitor with a sign-in of an account with access sees the reason. Other visitors see `this machine is not available right now. Retry later`.
- An SSH session gets `channel 0: open failed: connect failed: hopbox: the machine box1 cannot take SSH connections now: it did not wake: <reason>`.
- `hopbox exec`, `hopbox connect`, and `hopbox cp` fail with the error of `hopbox machine wake`.

The URL and the SSH session wait for at most 30 s. A machine that does not run by then gets the same answers, with the reason `it did not wake within 30s. Retry`.

An open connection keeps its machine awake: an SSH session, a terminal, an exec, or an open request of the URL, such as a WebSocket. A session that you forget holds the RAM of the machine on its node. Close the sessions that you do not use.

If an idle timer puts a machine in standby or hibernates it while it has an open connection at an edge server, hopbox brings it back within about 5 s. This covers a timer that acts in the seconds between a new connection and the first report of that connection. The audit entry `machine.resume` or `machine.wake` then has the actor `system` and the reason `open_connection`.

A pause or a hibernate by you, by an API key, or by an admin wins over an open connection. The machine stays in standby or hibernated, and the open session waits. This is also true when an idle timer put the machine to sleep first: a pause of a machine in standby, or a hibernate of a hibernated machine, keeps it asleep. A new connection still resumes or wakes the machine, as after any pause.

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

A save can fill the disk that the running machines write to. So each node refuses a save that leaves less free disk than its floor: `vmm.save_free_percent`, 10% by default, and `vmm.save_free_bytes`, 50 GiB by default ([the bootstrap config of hopboxd](/reference/hopboxd-config/index.html.md#keys)). A node with a small data dir sets lower values in `/etc/hopbox/hopboxd.toml`, for example:

```toml
[vmm]
save_free_percent = 5
save_free_bytes = 536870912
```

Restart `hopboxd` on the node after the change. Each refused save counts in `hopbox_saved_state_refusals_total`, and the alert `HopboxSavedStateDiskLow` fires ([Monitor an install with Prometheus and Grafana](/guide/monitor-an-install/index.html.md#hopboxsavedstatedisklow)). A node also runs at most 4 saves at once, and 2 for each org. A save above that waits for its turn.

## Errors

| Error | Cause | Fix |
| --- | --- | --- |
| `failed_precondition: failed precondition: machine action not allowed: cannot pause a machine whose target state is stopped` | Only a running machine goes to standby. | Start the machine first. |
| `failed_precondition: failed precondition: machine action not allowed: cannot resume a machine whose target state is running` | The machine is not in standby, or it lost its standby. | Read its state with `hopbox machine show box1`. |
| `unavailable: the machine did not resume or wake in time. Retry: the machine box1 is <state>` | An exec, a file copy, or `hopbox connect` resumed or woke the machine, and it did not run within 60 s. | Retry. Read the state with `hopbox machine show box1`. |
| `503` `The machine box1 did not wake: <reason>.` | A request of the URL brought the machine back, and the wake did not work. | Fix the reason, such as a limit of the org. After a failed wake, read [A failed wake](#a-failed-wake). |
| `failed_precondition: failed precondition: machine action not allowed: cannot rename a machine in standby: resume it first` | A rename found the machine in standby, or on its way in or out of it. | Run `hopbox machine resume box1`, then rename the machine. |
| `failed_precondition: failed precondition: machine action not allowed: cannot rename a hibernated machine: wake it first` | A rename found the machine hibernated, or on its way in or out of it. | Run `hopbox machine wake box1`, then rename the machine. |
| `failed_precondition: failed precondition: the machine box1 moves to another node. Pause it after the move` | An instance admin or a drain moves the machine. | Wait for the end of the move, and pause again. |
| `failed_precondition: failed precondition: the node of the machine box1 runs hopboxd <version>, whose heartbeats name no feature standby. An instance admin must upgrade the node first` | The node runs an older release. | Ask an instance admin to upgrade the node. |
| `failed_precondition: failed precondition: machine action not allowed: cannot hibernate a machine whose target state is stopped` | Only a running machine or a machine in standby hibernates. | Start the machine first. |
| `failed_precondition: failed precondition: machine action not allowed: cannot wake a machine whose target state is running` | The machine is not hibernated, or a wake of it failed. | Read its state with `hopbox machine show box1`. After a failed wake, run `hopbox machine start box1` ([A failed wake](#a-failed-wake)). |
| `failed_precondition: failed precondition: machine action not allowed: cannot stop a hibernated machine: wake it first` | A stop does not keep the memory of a hibernated machine. | Run `hopbox machine wake box1`, or drop the memory file with `hopbox machine discard-memory box1`. |
| `failed_precondition: failed precondition: machine action not allowed: cannot stop a machine that is hibernated: wait for the end of its wake. After a failed wake, drop the file with DiscardMemory` | A wake of the machine is pending, and its load did not start yet. | Wait for the wake, then stop the machine. If the wake fails, run `hopbox machine discard-memory box1`. |
| `failed_precondition: failed precondition: the machine box1 is failed, not running. Start it first` | An exec, a file copy, or `hopbox connect` woke the machine, and the wake failed. | Read [A failed wake](#a-failed-wake). |
| `resource_exhausted: usage limit exceeded: ...` | The wake takes the org over a limit. | Stop another machine of the org, or ask an owner of the org for a higher limit. |
| `resource_exhausted: usage limit exceeded: the saves of the org would write <bytes> bytes in 24 hours, and its daily budget is <limit>` | A hibernate passes the daily budget of saves of the org. The machine runs on. | Try again later, or ask an instance admin for a higher budget ([When a hibernate fails](#when-a-hibernate-fails)). |
| `resource_exhausted: resource exhausted: the node of the machine box1 refused the save, so the machine runs on: the free disk of the node is below the floor of a save: ...` | The save leaves less free disk on the node than its floor. | Ask an instance admin to free disk on the node ([The memory file on the node](#the-memory-file-on-the-node)). |
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
