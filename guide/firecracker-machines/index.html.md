# Run machines with Firecracker

> How to follow a machine on its node: its systemd unit, its disk, a restart of hopboxd with the machine running, and the causes of a failed boot.

This guide follows one machine on a server with the node role. It makes the machine, finds its systemd unit and its files, restarts `hopboxd` with the machine running, and deletes the machine. [The Firecracker driver](/reference/firecracker-driver/index.html.md) describes each step of the driver.

The node also connects each machine to its node network. [Check the network of a machine](/guide/node-network/index.html.md) follows the network of a machine.

## What you need

- A Linux server with the control role and the node role, installed as in [Install hopbox on a Debian 13 server by hand](/guide/install-debian/index.html.md). The server needs KVM, systemd, and `/var/lib/hopbox` on XFS with reflinks. `sudo hopboxd admin server list` must show the server `ready`.
- The default image of the release, registered as in [Create and manage machines](/guide/machines/index.html.md#register-the-default-image). The install put it in the image cache.
- The hopbox CLI, signed in to the install, as in [Use the hopbox CLI](/guide/cli/index.html.md).

## Make a machine

```sh
hopbox new box1
```

The CLI waits until the machine runs. The first machine of an image takes longer, because the node unpacks the image first. On the [tested server](/reference/preflight/index.html.md#the-tested-server), it takes about 25 s. The log of `hopboxd` shows each step:

```sh
sudo journalctl -u hopboxd -o cat | grep -E 'unpacked|booted|state changed'
```

```text
{"level":"INFO","msg":"unpacked the disk of an image","component":"firecracker","digest":"sha256:e591c258...","duration":22400000000}
{"level":"INFO","msg":"booted a microVM","component":"firecracker","machine":"01a1610f-...","unit":"hopbox-machine-01a1610f-....service","jail_id":1879048192,"tap":"hbt3f2c9a0b1d4e","duration":76000000}
{"level":"INFO","msg":"machine state changed","component":"executor","machine":"01a1610f-...","from":"starting","to":"running","boot_generation":0}
```

Run a command in the machine:

```sh
hopbox exec box1 uname -r
```

```text
6.1.188-hopbox1
```

## Find the unit of the machine

Each machine runs in a systemd unit of its own, in the slice `hopbox-machines.slice`:

```sh
systemctl list-units 'hopbox-machine-*'
```

```text
UNIT                                                         LOAD   ACTIVE SUB     DESCRIPTION
hopbox-machine-01a1610f-26b5-71c8-abbd-93be11b94c53.service loaded active running hopbox machine 01a1610f-26b5-71c8-abbd-93be11b94c53
```

The main process of the unit is Firecracker. It runs as the jail user of the machine, in the jail of the machine. The journal of the unit holds the serial console of the guest and the log of Firecracker:

```sh
sudo journalctl -u hopbox-machine-01a1610f-26b5-71c8-abbd-93be11b94c53.service
```

## Look at the disk

The disk of the machine is a reflink copy of the image, as large as the disk size of the machine. Only the blocks that the machine writes take space of their own:

```sh
sudo du -h --apparent-size /var/lib/hopbox/machines/01a1610f-26b5-71c8-abbd-93be11b94c53/disk.raw
sudo du -h /var/lib/hopbox/machines/01a1610f-26b5-71c8-abbd-93be11b94c53/disk.raw
```

The first command prints the size of the disk, 94G for 100 GB. The second prints the space of the file system that the file uses, at most the size of the image. The blocks that the disk shares with the image count in it too.

## Restart hopboxd

A restart or an upgrade of `hopboxd` stops only the processes of `hopboxd.service`. The unit of the machine is not one of them:

```sh
systemctl show -P MainPID hopbox-machine-01a1610f-26b5-71c8-abbd-93be11b94c53.service
sudo systemctl restart hopboxd
systemctl show -P MainPID hopbox-machine-01a1610f-26b5-71c8-abbd-93be11b94c53.service
hopbox exec box1 uptime
```

Both commands print the same PID, and `uptime` shows that the machine did not boot again. The new `hopboxd` logs `attached to a microVM` for the machine. Then it gives the machine a new link key, and the guest agent connects again.

## Reboot the server

A reboot or a shutdown of the server shuts each guest down cleanly. You do not need to stop the machines first. The unit of each machine runs `hopboxd stop-guest` at its stop, which asks the guest to shut down. Look at the stop command of the unit:

```sh
systemctl show -P ExecStop hopbox-machine-01a1610f-26b5-71c8-abbd-93be11b94c53.service
```

The output names `/opt/hopbox/current/hopboxd stop-guest`, with the API socket of the machine. At a shutdown, systemd stops `hopboxd` first and then the machines. Each guest writes its data to the disk, and unmounts its file systems. If a guest does not end within 30 s, systemd kills its Firecracker.

After the boot of the server, `hopboxd` boots each machine with the target `running` again. A machine that ran before the reboot thus runs again, with its disk:

```sh
sudo systemctl reboot
```

After the reboot, `hopbox exec box1 uptime` shows a new boot of the machine. A microVM that an older release booted can have no `ExecStop`. Stop and start such a machine once, so that its unit gets one.

Do not stop a machine with `systemctl stop` on the server. `hopboxd` did not ask for that stop, so the restart policy applies ([the Firecracker driver](/reference/firecracker-driver/index.html.md#a-stop-through-systemd)). Use `hopbox machine stop` instead.

## Stop, start, and delete

```sh
hopbox machine stop box1
hopbox machine start box1
hopbox machine delete box1
```

A stop asks the guest to shut down, and kills the microVM after 30 s. A start boots the machine from its disk, so the files of the machine stay. A delete stops the microVM, and removes its unit, its disk, its jail, and its tap device.

## When a boot fails

A machine whose boot fails gets the state `failed` ([the node executor](/reference/executor/index.html.md#actions)). The log of `hopboxd` holds the error of the boot, in the record `a machine action failed` of the executor:

```sh
sudo journalctl -u hopboxd -o cat | grep 'a machine action failed'
```

These are the common causes:

| Error | What to do |
| --- | --- |
| `the node has no guest kernel with the sha256 <sha256>, which the image needs` | Install the release that shipped the image. Its install adds its kernel to the kernel cache. |
| `the image cache has no image <digest>, and the image has no URL` | Install the release of the image, or register the image with `--url`. |
| `the file system of <dir> has no reflinks` | Put `/var/lib/hopbox` on XFS made with `mkfs.xfs -m reflink=1`. |
| `the jailer or Firecracker ended at its start` | Read the journal of the unit. The error names it. |

After a failed boot, no microVM of the machine runs. The restart policy `always` boots the machine again after a backoff.

## Run the KVM tests

`make test-kvm` runs the same steps on a test host, with names of its own for the units and the tap devices.
