# Install hopbox on a Debian 13 server by hand

> How to install hopbox on a Debian 13 server by hand, the path that the Terraform provider automates, and how to upgrade it and roll it back.

This guide installs hopbox on one Debian 13 server with the control role and the node role, then upgrades it and rolls it back. [Serve the domain of an install with the DNS role](/guide/dns-role/index.html.md) adds the DNS role.

The hopbox Terraform provider does these steps for each server of an install ([Install hopbox on an OVHcloud server with Terraform](/guide/install-on-ovhcloud/index.html.md)). Use this guide to try a release bundle on a test server, or to see what the provider does. An install of the hosted service gets its servers only from the provider.

The release files of hopbox are not public yet. The hopbox team gives them on request.

## What you need

- A Debian 13 server for `amd64`, with KVM and root access. The preflight check tells you what is missing ([the preflight check](/reference/preflight/index.html.md)).
- The package of the tested host kernel `6.18.57-hopbox1`, `linux-image-6.18.57-hopbox1_6.18.57-1_amd64.deb`, and its `.sha256` file, from the release files. Step 1 installs it. With another kernel, the install goes on with a warning.
- A free disk or partition for `/var/lib/hopbox`. The disks of machines need XFS with reflinks there.
- The release bundle `hopbox-v0.1.0-linux-amd64.tar.gz` and its `.sha256` file, from the release files. The people who work on hopbox can build them with `make bundle` instead.
- A domain for the install, such as `example.com`.

## 1. Install the host kernel

The host kernel `6.18.57-hopbox1` is the kernel that hopbox tests ([Host kernel](/reference/host-kernel/index.html.md)). The Debian kernels stay installed as fallbacks. Copy the package and its checksum file to the server, then check the package there:

```sh
scp linux-image-6.18.57-hopbox1_6.18.57-1_amd64.deb* root@server:/root/
ssh root@server
cd /root
sha256sum -c linux-image-6.18.57-hopbox1_6.18.57-1_amd64.deb.sha256
dpkg -i linux-image-6.18.57-hopbox1_6.18.57-1_amd64.deb
ls /boot/vmlinuz-6.18.57-hopbox1 /boot/initrd.img-6.18.57-hopbox1
```

`sha256sum` prints `linux-image-6.18.57-hopbox1_6.18.57-1_amd64.deb: OK`. If it prints `FAILED`, do not install the package. The pin below makes it the default of GRUB, also after a later upgrade of the Debian kernel.

GRUB names each menu entry after the UUID of the root file system, so the pin uses the ID of the entry. Read the UUID first:

```sh
uuid="$(findmnt -no UUID /)"
echo "$uuid"
```

If `echo` prints nothing, stop: the pin needs the UUID. Else write the pin, and make sure that `grub.cfg` has the entry:

```sh
mkdir -p /etc/default/grub.d
printf 'GRUB_DEFAULT="gnulinux-advanced-%s>gnulinux-6.18.57-hopbox1-advanced-%s"\n' "$uuid" "$uuid" \
  > /etc/default/grub.d/20_hopbox_kernel.cfg
update-grub
grep -F "'gnulinux-6.18.57-hopbox1-advanced-$uuid'" /boot/grub/grub.cfg
```

The `grep` prints the line of the entry. If it prints nothing, `grub.cfg` has no entry for the kernel. Then remove `/etc/default/grub.d/20_hopbox_kernel.cfg`, run `update-grub` again, and keep the Debian kernel.

Reboot, and check the kernel after the reboot:

```sh
reboot
```

```sh
uname -r
```

```text
6.18.57-hopbox1
```

On the tested server, SSH answered 187 to 319 seconds after the reboot.

## 2. Make the XFS file system

Install the XFS tools, and make the file system with reflinks on the free partition. This example uses `/dev/md3`. Use the name of your partition.

```sh
sudo apt-get install -y xfsprogs
sudo mkfs.xfs -m reflink=1 /dev/md3
sudo mkdir -p /var/lib/hopbox
echo '/dev/md3 /var/lib/hopbox xfs defaults 0 2' | sudo tee -a /etc/fstab
sudo mount /var/lib/hopbox
```

`mkfs.xfs` erases the partition. Check the name of the partition before you run it.

## 3. Copy and check the bundle

Copy the bundle and its checksum file to the server, then check it there:

```sh
scp hopbox-v0.1.0-linux-amd64.tar.gz* root@server:/root/
ssh root@server
cd /root
sha256sum -c hopbox-v0.1.0-linux-amd64.tar.gz.sha256
```

`sha256sum` prints `hopbox-v0.1.0-linux-amd64.tar.gz: OK`.

## 4. Install

Unpack the bundle, and run its `hopboxd install` as root:

```sh
tar -xzf hopbox-v0.1.0-linux-amd64.tar.gz
sudo ./hopbox-v0.1.0/hopboxd install --roles control,node --domain example.com
```

The install copies the release to `/opt/hopbox/releases/v0.1.0`, makes the users, the dirs, the config, the first key-encryption key, and the Postgres cluster. It reserves the SSH ports, writes the systemd units, runs the preflight check, and starts `hopboxd` ([hopboxd](/reference/hopboxd/index.html.md#install)). It prints the preflight check, log records, and these last lines:

```text
hopboxd: installed v0.1.0, and hopboxd runs. Changed: users, release files, keys, postgres, ports, config, release, units.
v0.1.0
```

The Terraform provider also passes `--bootstrap-admin-key-hash` with the hash of the first admin API key. At its first start, the control role registers that hash as the admin API key `bootstrap` and removes the file ([Server layout](/reference/server-layout/index.html.md#the-bootstrap-admin-api-key)). A server that you install by hand does not need it: root makes admin keys with `hopboxd admin admin-key create` ([hopboxd](/reference/hopboxd/index.html.md#admin-admin-key-create)).

Check that `hopboxd` runs, and that the install has its domain:

```sh
systemctl status hopboxd
sudo hopboxd admin config get
```

`systemctl status` shows `active (running)`. `admin config get` prints `domain	example.com`. The log of `hopboxd` is in the journal: `journalctl -u hopboxd`.

You can run the install again at any time. With the same flags, it changes nothing and does not restart `hopboxd`:

```text
$ sudo hopboxd install --roles control,node
hopboxd: installed v0.1.0, and hopboxd runs. Nothing changed.
v0.1.0
```

## 5. Enroll the node role

The node role of the server enrolls with the control role of the same server ([Enroll the node role on one server](/guide/enroll-node/index.html.md)). Make an enrollment token, and give it to the install:

```sh
TOKEN="$(sudo hopboxd admin enrollment-token create)"
sudo HOPBOX_ENROLLMENT_TOKEN="$TOKEN" hopboxd install --roles control,node
sudo hopboxd admin server list
```

The install prints `hopboxd: enrolled as the server <id>.` Within a few seconds, `admin server list` shows the server as `ready`.

## 6. Sign in

Make a login link for the first instance admin, and open it ([First sign-in on a new install](/guide/first-sign-in/index.html.md)):

```sh
sudo hopboxd admin login-link --email you@example.org
```

## Upgrade

Copy the new bundle and its checksum file to the server. Then run:

```sh
sudo hopboxd upgrade --bundle hopbox-v0.2.0-linux-amd64.tar.gz \
  --sha256 "$(cut -d' ' -f1 hopbox-v0.2.0-linux-amd64.tar.gz.sha256)"
```

The upgrade unpacks the bundle to `/opt/hopbox/releases/v0.2.0` and runs the install of the new release ([hopboxd](/reference/hopboxd/index.html.md#upgrade)). That install restarts `hopboxd`. The running machines stay up, because they run in `hopbox-machines.slice`, outside `hopboxd.service`. The data stays: the config, the keys, the cluster, and the identity of the server.

If the new release has another major version of Postgres, the upgrade stops before the switch, and the old release keeps running. The error starts with `postgres data dir /var/lib/hopbox/pg/data holds a Postgres 18 cluster`. The same holds for a release that does not run with the newest schema migration of the control store ([the bundled Postgres](/reference/postgres/index.html.md#two-releases-on-one-store)).

An install with more than one server upgrades one server at a time: first the control servers, then the edge and DNS servers, then the nodes. The hopbox Terraform provider keeps this order ([hopbox Terraform provider](/reference/terraform-provider/index.html.md#rolling-upgrade)). The release before can run beside the new one, so the servers that wait keep working.

## Roll back

The server keeps the previous release. To go back to it, run:

```sh
sudo hopboxd upgrade --rollback
```

A rollback cannot undo a schema migration of the newer release. Each migration names the oldest release that runs with it. If the newer release applied a migration that does not allow the previous release, the rollback stops before the switch, and the newer release keeps running. The error starts with `the control store has the schema migration`. Upgrade to a release that knows the migrations instead ([hopboxd](/reference/hopboxd/index.html.md#upgrade)).

v0.1.0 runs only on a store whose migrations it knows. So a rollback from v0.2.0 to v0.1.0 stops with that error. A rollback from a later release, such as v0.3.0, to v0.2.0 works when the migrations of v0.3.0 allow v0.2.0. Then `readlink /opt/hopbox/current` prints `releases/v0.2.0`, and `readlink /opt/hopbox/previous` prints `releases/v0.3.0`. A second rollback goes forward again.

## If the install stops

The install makes every check before it changes the config, the current release, or the units. So after a refusal, fix the cause and run the same command again.

| Message | Fix |
| --- | --- |
| `/var/lib/hopbox is on ext4, but the disks of machines need XFS with reflinks there` | Do step 2. |
| `the preflight check failed, so this server cannot run machines` | Fix each failed check that the lines above it list, such as KVM in the firmware ([the preflight check](/reference/preflight/index.html.md)). |
| `--domain is required on the first install of the control role on a server` | Add `--domain`. |
| `/usr/local/bin has mode 0775, which lets group or others change it` | A Debian host that was upgraded from an old release can have the mode 2775 with the group `staff` there. Run `sudo chmod 755 /usr/local/bin`. |
| `/opt/hopbox/releases belongs to uid 1000, ...` | Run `sudo chown root:root` on the dir that the message names. |
| `hopboxd v0.2.0 did not start` | Read `journalctl -u hopboxd`, then run `sudo hopboxd upgrade --rollback`. |

[Server layout](/reference/server-layout/index.html.md) lists every path, user, and unit of the install.
