# Extra disks

> How to make an extra disk and attach it to a machine. How to format and mount it in the guest, and delete it.

This guide shows you how to give a machine an extra disk. An extra disk is a block device apart from the root disk of the machine. A detach keeps its data, and so does a delete of its machine. So you can keep data, such as a database or a build cache, apart from the machine that uses it.

| Rule | Value |
| --- | --- |
| Size | 1 to 2000 GB, in whole GB. The disk counts in the disk limit of your org by its size. |
| File system | None. You format the disk once, in the guest. |
| Machines | One machine at a time. A machine holds at most 8 disks. |
| When | Attach and detach need a stopped machine. |
| Where | A disk lives on one node, and attaches only to a machine on that node. |

[Extra disks](/reference/disks/index.html.md) gives every rule and every error.

You need:

- The `hopbox` CLI, signed in ([hopbox CLI](/reference/cli/index.html.md)).
- A machine, here `box1`.

## Make a disk

Make the disk with the name of the machine, so that the disk lives on the node of `box1`:

```sh
hopbox disk create disk1 --size 10 --machine box1
```

The command waits until the node of the disk made its file, and prints:

```text
Made the disk disk1 of 10 GB.
```

A new disk is private to you, as a new machine is. To open it to your org, run `hopbox disk share disk1`. A shared disk attaches only to a shared machine, and a private disk only to a private machine of its creator.

## Attach the disk

Stop the machine, then attach the disk:

```sh
hopbox machine stop box1
hopbox disk attach disk1 box1
```

The command prints the device of the disk in the guest:

```text
Attached the disk disk1 to the machine box1. The guest sees it as /dev/vdb at its next boot.
```

Add `--read-only` to the attach to give the guest a disk that it cannot write. Then start the machine:

```sh
hopbox machine start box1
```

## Format and mount the disk

The disk is empty, with no file system. Format it once, then mount it. Run these commands in the machine:

```sh
hopbox exec box1 -- sudo mkfs.ext4 -L disk1 /dev/vdb
hopbox exec box1 -- sudo mkdir -p /mnt/disk1
hopbox exec box1 -- sudo mount LABEL=disk1 /mnt/disk1
```

A format erases the disk. Run `mkfs.ext4` only on a new disk.

The guest sees the disks in the order of their attach: `/dev/vdb`, then `/dev/vdc`, up to `/dev/vdi`. If you detach an earlier disk, the names of the later disks move at the next boot. So mount a disk by its label, such as `LABEL=disk1`, and not by its device. hopbox mounts nothing at boot. To mount the disk at each boot, add a line to `/etc/fstab` in the machine, for example `LABEL=disk1 /mnt/disk1 ext4 defaults,nofail 0 2`.

`hopbox disk list` shows each disk, its machine, its device, and its mode:

```text
NAME   SIZE   MACHINE  DEVICE    MODE  SHARED  READY
disk1  10 GB  box1     /dev/vdb  rw    no      yes
```

## Detach or delete the disk

Stop the machine, then detach the disk. The data stays on the disk:

```sh
hopbox machine stop box1
hopbox disk detach disk1
```

A detached disk attaches to another stopped machine on its node. To delete the disk and its data, run:

```sh
hopbox disk delete disk1
```

The command asks first. Add `-y` to skip the question. A disk that is attached cannot be deleted: detach it first.

## What happens to a disk

| Event | The disk |
| --- | --- |
| A reboot, a stop, or a start of its machine | Keeps its data. |
| A hibernate of its machine | Stays attached. You cannot attach or detach a disk while the node of the machine keeps its memory file. Wake the machine, or drop the file with `hopbox machine discard-memory`, first. |
| A delete of its machine | Detaches, and keeps its data. The disk goes on counting in the limits of the org until you delete it. |
| A move of its machine to another node | Moves with the machine, with its data. |
| The drain of its node | A detached disk moves to another node by itself. If no node takes it, an instance admin sees it in the drain. |
| A suspension of the org | Keeps its data. You cannot make, attach, share, or delete a disk until the suspension ends. |
| You leave the org | A private disk of yours goes with its data. A shared disk stays in the org. |
| A backup of its machine | Is not in the backup. A backup copies the root disk only. |

A delete removes the file of the disk from its node. The bytes can stay on the disk of the node until the node reuses the blocks.
