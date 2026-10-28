# Install hopbox on your servers with Terraform

> How to set up Debian 13 hosts from any provider and install hopbox on them with Terraform. Also how to upgrade, grow a fleet with floating addresses, import, and destroy.

This guide makes a new install of hopbox on one server with the hopbox Terraform provider. The server gets every role. Then the guide upgrades the server and changes its labels. It grows the install into a fleet, imports a server into a new state, and destroys a server.

The provider installs hopbox over SSH on hosts that run Debian 13. A host can come from any provider of servers, or be a computer that you own. You order each host and install Debian 13 on it. The provider takes it from there. It calls no API of a provider of servers (Q90).

hopbox supports only the servers that this provider sets up. [hopbox Terraform provider](/reference/terraform-provider/index.html.md) lists every attribute and each step that the provider takes. [Install hopbox on a Debian 13 server by hand](/guide/install-debian/index.html.md) shows the same install by hand.

The release files of hopbox are not public yet. The hopbox team gives them on request.

Use a host that holds nothing else of value. The apply installs another kernel on the host, and reboots it.

## What you need

- One host for the first server, with these items. [1. Set up the hosts](#1-set-up-the-hosts) shows each one.
  - A CPU for `amd64` with KVM. The [tested server](/reference/preflight/index.html.md#the-tested-server) has an Intel Xeon E5-1650 v4 CPU.
  - Debian 13.
  - A public IPv4 address on its own uplink, with no NAT in front of it. This is the peer address of the server ([Enrollment and the install PKI](/reference/enrollment/index.html.md#peer-addresses)). This guide uses `203.0.113.10`.
  - A free partition or disk for `/var/lib/hopbox`.
  - A console at its provider, such as a KVM or serial console.
- A Mac with Apple Silicon or a Linux host on x86_64, with Terraform 1.5 or later. This example uses the Mac.
- The release files of v0.2.0, the first release with this provider, from the hopbox team ([2. Get the release files](#2-get-the-release-files)). Until the hopbox team tags v0.2.0, build the files from a checkout.
- A domain for the install, such as `example.com`.
- An SSH key pair for the operators, without a passphrase. This example uses `~/.ssh/id_ed25519`.

## 1. Set up the hosts

Do these steps on each host before the apply that creates its server ([hopbox Terraform provider](/reference/terraform-provider/index.html.md#the-host)). The SSH port of a new host faces the internet before `hopboxd` and its host firewall run. So the host must take SSH keys alone from the start.

1. Order the host at its provider with Debian 13 for `amd64`. Pick a server model with KVM. Keep a partition or a disk free for `/var/lib/hopbox`.
2. Put the public key of the operators in the `authorized_keys` of `root`. Most providers take an SSH key at the install of the OS. Instead of `root`, you can use a user with `sudo` without a password, such as `debian`. Then set `ssh_user = "debian"` in the resource of the server.
3. Open the console of the host at its provider. Log in there, and print the SSH host key of the host and its fingerprint:

   ```sh
   cat /etc/ssh/ssh_host_ed25519_key.pub
   ssh-keygen -lf /etc/ssh/ssh_host_ed25519_key.pub
   ```

   Copy the first line, such as `ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAI... root@host1`, for `ssh_host_key`. A comment after the key changes nothing.
4. Log in to the host over SSH as `root`, or log in as the `sudo` user and run `sudo -i`. At the first login, `ssh` shows the fingerprint of the host key. Type `yes` only if it is the fingerprint that step 3 printed.
5. Make sure that the host holds its peer address on its uplink:

   ```sh
   ip -br -4 addr show
   ```

   The line of the uplink names the public address, such as `203.0.113.10/24`. If the host holds only a private address, such as `10.0.0.5`, a NAT sits in front of it. Such a host cannot join an install.
   For a node, also make sure that the host names a resolver that machines reach:

   ```sh
   grep nameserver /run/systemd/resolve/resolv.conf /etc/resolv.conf
   ```

   One line names an address outside `127.0.0.0/8`, such as `192.0.2.53`. A resolver on loopback alone, such as a local `unbound`, reaches no machine.
6. Make sure that the host has KVM. `ls -l /dev/kvm` shows the device.
7. Turn off the password logins of `sshd`:

   ```sh
   printf 'PasswordAuthentication no\nKbdInteractiveAuthentication no\n' > /etc/ssh/sshd_config.d/10-hopbox.conf
   systemctl reload ssh
   sshd -T -C user=root,host=192.0.2.1,addr=192.0.2.1 | grep -e passwordauthentication -e kbdinteractiveauthentication
   ```

   The last command prints `passwordauthentication no` and `kbdinteractiveauthentication no`. If you log in as another user, such as `debian`, run it again with that user in `user=`. The provider makes the same check before the create, so a `Match` block that turns passwords on for one of these logins fails it. `sshd` takes the first value of a key, and reads the files of `/etc/ssh/sshd_config.d` in the order of their names. So `10-hopbox.conf` comes before a file of the provider, such as `50-cloud-init.conf`. If the command prints `yes`, look for another file that sets the key first.
8. Make `/var/lib/hopbox` an XFS file system with reflinks, on its free partition or disk, as [Install hopbox on a Debian 13 server by hand](/guide/install-debian/index.html.md#2-make-the-xfs-file-system) shows. `mkfs.xfs` erases the partition, so make sure that you name the free one.
9. Make sure that the file system is ready:

   ```sh
   findmnt -no FSTYPE /var/lib/hopbox
   xfs_info /var/lib/hopbox | grep -o 'reflink=[01]'
   ```

   The commands print `xfs` and `reflink=1`.

Take the host key from the console alone. Do not take it from `ssh-keyscan`, or from the first SSH login over the internet without the check of step 4. A machine between you and the host can show its own key there. The provider pins the key of `ssh_host_key` before its first login, and refuses any other key ([hopbox Terraform provider](/reference/terraform-provider/index.html.md#host-key-trust)).

An edge server of a fleet also needs its floating addresses. Route each one to the host at its provider before the apply ([Grow the install into a fleet](#grow-the-install-into-a-fleet)). The first server of this guide has none until the fleet.

## 2. Get the release files

Put these release files in one folder, such as `~/hopbox-release`:

| File | What it holds |
| --- | --- |
| `terraform-provider-hopbox_0.2.0_darwin_arm64.zip` | The provider for a Mac with Apple Silicon. Linux hosts use `terraform-provider-hopbox_0.2.0_linux_amd64.zip`. |
| `hopbox-v0.2.0-linux-amd64.tar.gz` and its `.sha256` file | The release bundle of the servers ([Release bundle](/reference/release-bundle/index.html.md)). |
| `linux-image-6.18.57-hopbox1_6.18.57-1_amd64.deb` and its `.sha256` file | The tested host kernel, a Debian package ([Host kernel](/reference/host-kernel/index.html.md)). |
| `SHA256SUMS` | The sha256 of each other file. |

Compare each file with its line in `SHA256SUMS`. On Linux, use `sha256sum -c` in place of `shasum -a 256 -c`:

```sh
cd ~/hopbox-release
grep -e terraform-provider-hopbox_0.2.0_darwin_arm64.zip -e hopbox-v0.2.0-linux-amd64.tar.gz -e linux-image-6.18.57 SHA256SUMS |
  shasum -a 256 -c
```

The command prints `OK` for each of the four files. `SHA256SUMS` has no line for the `.sha256` file of the host kernel package. If it prints `FAILED`, get that file again.

The people who work on hopbox can build the same files from a checkout instead.

## 3. Write the Terraform files

The provider is in no registry. Put its zip in the local mirror of Terraform, in the dir of its source address:

```sh
dir=~/.terraform.d/plugins/registry.terraform.io/hopboxdev/hopbox
mkdir -p "$dir"
cp ~/hopbox-release/terraform-provider-hopbox_0.2.0_darwin_arm64.zip "$dir/"
```

Terraform reads the dir `~/.terraform.d/plugins` as a local mirror only without a `provider_installation` block in its CLI configuration. If your CLI configuration has such a block, add a `filesystem_mirror` block for that dir to it.

Make a folder for the install:

```sh
mkdir -p ~/hopbox-install && cd ~/hopbox-install
```

Write `main.tf`. Put the peer address and the host key of step 1 in it. Also put the paths of step 2 in it, with the sha256 values of the two `.sha256` files:

```hcl
terraform {
  required_providers {
    hopbox = { source = "hopboxdev/hopbox", version = "0.2.0" }
  }
}

provider "hopbox" {
  ssh_private_key = file(pathexpand("~/.ssh/id_ed25519"))
}

locals {
  bundle = {
    source = pathexpand("~/hopbox-release/hopbox-v0.2.0-linux-amd64.tar.gz")
    sha256 = "<the sha256 of the bundle>"
  }
  host_kernel = {
    source = pathexpand("~/hopbox-release/linux-image-6.18.57-hopbox1_6.18.57-1_amd64.deb")
    sha256 = "<the sha256 of the package>"
  }
}

resource "hopbox_server" "first" {
  address      = "203.0.113.10"
  ssh_host_key = "ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAI..."
  bootstrap    = true
  domain       = "example.com"
  labels       = { zone = "zone1" }

  bundle      = local.bundle
  host_kernel = local.host_kernel

  lifecycle {
    prevent_destroy = true
  }
}

output "first_host_key" {
  value = hopbox_server.first.ssh_host_key
}

output "admin_api_key" {
  value     = hopbox_server.first.admin_api_key
  sensitive = true
}
```

`bootstrap = true` makes this server the first server of a new install. The destroy of the first server ends the install. So `prevent_destroy` makes Terraform refuse each plan that destroys it. Without `ssh_user`, the provider logs in as `root`.

The state holds the first admin API key of the install, and the pinned host key of the server. Keep the state in a place that only the instance admins can read. A `terraform.tfstate` file on a laptop is good enough to try the provider. For a real install, use a backend with encryption and access control.

## 4. Apply

```sh
terraform init
terraform apply
```

Most of the apply is the upload of the bundle, and the reboot into the host kernel. To follow the steps, set `TF_LOG_PROVIDER=INFO`. The log shows lines such as `logged in to the server with its pinned host key`, `installing the host kernel`, and `installing hopboxd`.

Before it changes the host, the provider makes sure that the host is ready. It refuses another host key, `sshd` with password logins, and a host that holds an install. It also refuses another OS, a host without its peer address, and `/var/lib/hopbox` without XFS. Each error names the fix ([If the apply stops](#if-the-apply-stops)). A warning such as `The preflight check of the server warns: ...` means that the server runs, with a risk that the warning names.

## 5. Make sure that the server runs

The examples log in as `root`. If you set `ssh_user = "debian"`, log in as `debian`, and put `sudo` before each command on the server. List the servers of the install:

```sh
ssh -i ~/.ssh/id_ed25519 root@203.0.113.10 hopboxd admin server list
```

The output shows one server, `ready`, with the version of the bundle and its peer address:

```text
ID                                    POOL     STATE  STATUS  ROLES          HOSTNAME  VERSION  LAST HEARTBEAT        PEER ADDRESS
01a14404-ce7b-72d9-b18e-5006c3e462db  default  ready  ok      node,dns,edge  host1     v0.2.0   2026-10-16T09:00:05Z  203.0.113.10
```

The ROLES column lists the roles that enrolled. The control role of the first server runs in the same `hopboxd` from its bootstrap configuration, without a server record.

`uname -r` on the server prints `6.18.57-hopbox1`.

The server also has the DNS role, which answers DNS for the domain of the install. The provider set the public addresses of the install to the peer address of the server, so the domain points at the server. The DNS role listens on port 53 of that address, because `systemd-resolved` of the OS holds port 53 on `127.0.0.53` ([hopbox Terraform provider](/reference/terraform-provider/index.html.md#create)). Read the public addresses:

```sh
ssh -i ~/.ssh/id_ed25519 root@203.0.113.10 hopboxd admin config get
```

The line `public_addresses` holds `203.0.113.10`. Delegate the domain from its parent zone, as [Serve the domain of an install with the DNS role](/guide/dns-role/index.html.md#delegate-the-domain-of-the-install) shows. Use the same address for the glue records.

The server also has the edge role. It serves the API, the console, and the machine URLs on ports 443 and 80 ([the edge role](/reference/edge/index.html.md)). It starts HTTPS once the ACME job got the certificate of the domain ([Get the wildcard certificate of an install](/guide/acme-certificate/index.html.md)). Open TCP ports 80 and 443, and port 53 for TCP and UDP, in each firewall of the provider in front of the server.

## 6. Keep the admin API key

```sh
terraform output -raw admin_api_key
```

The key starts with `hbxadm_`. It is the first admin API key of the install, with the name `bootstrap`. It expires after 90 days. Before it expires, make a new key on the server, and use it in the provider block of your other configurations:

```sh
ssh -i ~/.ssh/id_ed25519 root@203.0.113.10 hopboxd admin admin-key create --name terraform --lifetime 365d
```

## 7. Sign in

Make a login link for the first instance admin, and open it ([First sign-in on a new install](/guide/first-sign-in/index.html.md)):

```sh
ssh -i ~/.ssh/id_ed25519 root@203.0.113.10 hopboxd admin login-link --email you@example.org
```

## Upgrade

Get the bundle of the new release and its `.sha256` file. Change `local.bundle` in `main.tf` to its path and sha256, and apply:

```sh
terraform apply
```

The plan shows an update in place of `bundle` and `version`. The provider uploads the bundle and runs `hopboxd upgrade` on the server. The running machines stay up ([hopboxd](/reference/hopboxd/index.html.md#upgrade)). If the new release cannot run on the data, the upgrade stops before the switch, and the old release keeps running.

The provider then waits until the server is `ready` on the new version. An install with more servers upgrades them one at a time, the control servers before the nodes ([hopbox Terraform provider](/reference/terraform-provider/index.html.md#rolling-upgrade)).

An install from an earlier release of the provider, which ordered and reinstalled its servers at OVHcloud, keeps its servers. Before the first plan with this release, remove the `ovh_` attributes of each server. Give it `address`, `ssh_host_key`, and `ssh_user`, with the values of its state. The first refresh moves the state, with no change on the server ([hopbox Terraform provider](/reference/terraform-provider/index.html.md#a-state-of-an-earlier-release)). The guide for ks shows these steps for ks.

## Change the labels

Change `labels` in `main.tf`, for example to `{ zone = "zone1", disk = "nvme" }`, and apply. The provider calls `SetServerLabels`. `hopboxd admin server get <id>` on the server then shows `labels	disk=nvme,zone=zone1` and `labels_set_by_admin	true`.

## Add pools and sizes

After the first server runs, you can add pools and sizes in the same Terraform files. A pool groups the servers of one model, and a size can keep its machines in one pool. Both need the admin API. So set `install_url` in the provider block first, and export the key of step 6 as `HOPBOX_ADMIN_API_KEY`:

```hcl
provider "hopbox" {
  ssh_private_key = file(pathexpand("~/.ssh/id_ed25519"))
  install_url     = "https://example.com"
}

resource "hopbox_pool" "big" {
  name = "big"
}

resource "hopbox_size" "large" {
  name       = "large"
  vcpus      = 8
  memory_gib = 32
  disk_gb    = 200
  pool       = hopbox_pool.big.name
}
```

Keep the key out of the files. The first server stays in the pool `default`. A server that joins `big` sets `pool = hopbox_pool.big.name`. [Add pools and sizes](/guide/pools-and-sizes/index.html.md) explains the rules, and [hopbox Terraform provider](/reference/terraform-provider/index.html.md#resource-hopbox_pool) lists every attribute.

## Grow the install into a fleet

A fleet runs the roles on several servers: edge servers that take the public traffic, control servers that share one external Postgres, and nodes that run machines. The servers talk over their peer addresses on the public internet. The host firewall of each server takes the traffic of the install only from the peer addresses of the other servers ([the host firewall](/reference/host-firewall/index.html.md)). This section builds a fleet of five servers in two pools:

| Resource | Roles | Pool | Peer address | Floating address |
| --- | --- | --- | --- | --- |
| `first` | `control`, `node`, `dns`, `edge` | `default` | `203.0.113.10` | `198.51.100.5` |
| `edge2` | `edge`, `dns` | `default` | `203.0.113.11` | `198.51.100.6` |
| `cp2` | `control`, `node` | `default` | `203.0.113.13` | none |
| `node1` | `node` | `default` | `203.0.113.21` | none |
| `node2` | `node` | `big` | `203.0.113.22` | none |

### What a fleet needs

- A host for each new server, set up as in [1. Set up the hosts](#1-set-up-the-hosts). The hosts can come from other providers. All the servers of a pool have the same hardware ([Enrollment and the install PKI](/reference/enrollment/index.html.md#hardware-of-a-pool)).
- One floating IPv4 address for each edge server. A floating address is a public IPv4 address that the provider of the servers can route to one server and then to another. Examples are an Additional IP of OVHcloud, and a floating IP of another provider. This example uses `198.51.100.5` and `198.51.100.6`.
- For the second control server, an external Postgres that the control servers reach ([Run the control store on an external Postgres](/guide/external-postgres/index.html.md)).

You route each floating address at the provider of the servers. The Terraform provider never moves an address. It puts each address of `floating_addresses` on the uplink of the server as a `/32`. It also makes the peer address the source of the default route ([hopbox Terraform provider](/reference/terraform-provider/index.html.md#the-network-setup-of-an-edge-server)). So the connections of an edge server to the other servers leave from its peer address, which their host firewalls take.

This guide uses IPv4 alone, so the zone has no AAAA record. An edge server can also take one floating IPv6 address for the AAAA records, with `public_ipv6` ([hopbox Terraform provider](/reference/terraform-provider/index.html.md#edge-servers-and-floating-addresses)).

### 1. Give the first server its floating address

At the provider of the first server, route `198.51.100.5` to it. Then add the address to the resource `first` in `main.tf`:

```hcl
resource "hopbox_server" "first" {
  address            = "203.0.113.10"
  ssh_host_key       = "ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAI..."
  bootstrap          = true
  domain             = "example.com"
  labels             = { zone = "zone1" }
  floating_addresses = ["198.51.100.5"]

  bundle      = local.bundle
  host_kernel = local.host_kernel

  lifecycle {
    prevent_destroy = true
  }
}
```

Apply, as in step 4:

```sh
terraform apply
```

The plan shows an update in place. The provider adds port 53 of the address to the DNS role. It sets up the network with the unit `hopbox-network.service`, which runs again at each boot. It also makes the address the source of the egress of the machines ([the egress of the machines](/reference/egress/index.html.md#the-source-addresses)). The zone of the DNS role names the peer address of the server for now. [Step 5](#5-point-the-zone-at-the-edge-servers) points it at the edge servers.

Look at the network of the server:

```sh
ssh -i ~/.ssh/id_ed25519 root@203.0.113.10 ip -br -4 addr show
ssh -i ~/.ssh/id_ed25519 root@203.0.113.10 ip -4 route show default
```

The uplink holds `203.0.113.10` and `198.51.100.5/32`. The default route ends with `src 203.0.113.10`.

The internal listener of the control role listens on port 7443 of the peer address, `https://203.0.113.10:7443`. The other servers enroll there. `terraform state show hopbox_server.first` shows it in `internal_url`. A first server from an earlier release listens on loopback. Its next update moves the listener to its peer address ([hopbox Terraform provider](/reference/terraform-provider/index.html.md#the-internal-listener)).

### 2. Set the jump host and the admin API

The other servers enroll through the admin API, so the provider needs `install_url` and an admin API key ([6. Keep the admin API key](#6-keep-the-admin-api-key)). Export the key, and keep it out of the files:

```sh
export HOPBOX_ADMIN_API_KEY="$(terraform output -raw admin_api_key)"
terraform output -raw first_host_key
```

The first server is an edge server, so it is the jump host of the provider. Put its peer address and the host key of the output in the provider block:

```hcl
provider "hopbox" {
  ssh_private_key = file(pathexpand("~/.ssh/id_ed25519"))
  install_url     = "https://example.com"

  ssh_jump_host = {
    address  = "203.0.113.10"
    host_key = "ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAI..."
  }
}
```

From then on, every SSH step of a server without the role `edge` goes through the first server. The jump host dials the peer address of the server ([hopbox Terraform provider](/reference/terraform-provider/index.html.md#jump-host)). Once `hopboxd` runs on a node or a control server, its host firewall takes SSH only from the peer addresses of the edge servers ([the host firewall](/reference/host-firewall/index.html.md#the-jump-hosts)). So an edge server as the jump host is then the only way in.

### 3. Add an edge server and the nodes

Set up the hosts of `edge2`, `node1`, and `node2` as in [1. Set up the hosts](#1-set-up-the-hosts). At the provider of `edge2`, route `198.51.100.6` to it. Add a pool for the bigger nodes, as in [Add pools and sizes](#add-pools-and-sizes), and the servers:

```hcl
resource "hopbox_pool" "big" {
  name = "big"
}

resource "hopbox_server" "edge2" {
  address            = "203.0.113.11"
  ssh_host_key       = "ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAI..."
  roles              = ["edge", "dns"]
  floating_addresses = ["198.51.100.6"]
  bundle             = local.bundle
  host_kernel        = local.host_kernel
  depends_on         = [hopbox_server.first]

  lifecycle {
    create_before_destroy = true
  }
}

resource "hopbox_server" "node1" {
  address      = "203.0.113.21"
  ssh_host_key = "ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAI..."
  bundle       = local.bundle
  host_kernel  = local.host_kernel
  depends_on   = [hopbox_server.first]
}

resource "hopbox_server" "node2" {
  address      = "203.0.113.22"
  ssh_host_key = "ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAI..."
  pool         = hopbox_pool.big.name
  bundle       = local.bundle
  host_kernel  = local.host_kernel
  depends_on   = [hopbox_server.first]
}
```

Give each server the host key from its own console. `depends_on` makes Terraform update the first server before the others in a [rolling upgrade](/reference/terraform-provider/index.html.md#rolling-upgrade). Apply:

```sh
terraform apply
```

For each server, the provider makes an enrollment token that names its peer address. The host firewall of the first server then takes port 7443 from that address while the token works ([the host firewall](/reference/host-firewall/index.html.md#sets)). The servers enroll at the internal listener of the first server. List the servers on the first server:

```sh
ssh -i ~/.ssh/id_ed25519 root@203.0.113.10 hopboxd admin server list
```

The output shows four servers, each `ready`, with their peer addresses: the first server, `edge2` with the roles `dns,edge`, and the two nodes, `node2` in the pool `big`.

Through the jump host, list the jump hosts that `node1` takes on port 22:

```sh
ssh -i ~/.ssh/id_ed25519 -J root@203.0.113.10 root@203.0.113.21 nft list set inet hopbox_host jump_hosts
```

The set holds `203.0.113.10` and `203.0.113.11`, the peer addresses of the two edge servers. If it is empty, wait 10 s and list it again. Until the server knew an edge server, port 22 takes every source ([the host firewall](/reference/host-firewall/index.html.md#never-a-lockout)).

Then make sure that the public address of `node1` is closed. From your laptop, a connection to its port 22 must time out:

```sh
nc -vz -w 3 203.0.113.21 22
```

### 4. Add a second control server

A second control server shares the control store with the first one, so the install must run on the external Postgres first. Move the store as [Run the control store on an external Postgres](/guide/external-postgres/index.html.md#move-a-one-server-install-to-the-external-postgres) shows. Set up the host of `cp2` as in [1. Set up the hosts](#1-set-up-the-hosts). Then add the server:

```hcl
resource "hopbox_server" "cp2" {
  address      = "203.0.113.13"
  ssh_host_key = "ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAI..."
  roles        = ["control", "node"]
  bundle       = local.bundle
  host_kernel  = local.host_kernel
  depends_on   = [hopbox_server.first]

  copy_control_from = {
    address  = hopbox_server.first.address
    host_key = hopbox_server.first.ssh_host_key
    user     = hopbox_server.first.ssh_user
  }
}
```

`copy_control_from` names the live control server that `cp2` copies from, with its pinned host key. Apply. Before it changes `cp2`, the provider logs in to the first server, and reads its key-encryption keys and the files of its external Postgres. It compares the fingerprint of each key with the admin API, and writes the files on `cp2` before the install. So both control servers seal with the same keys, and use the same store ([hopbox Terraform provider](/reference/terraform-provider/index.html.md#control-servers)). If the first server still runs the bundled Postgres, the apply stops before any change of `cp2`, with `the control server runs the bundled Postgres`.

Make sure that both control processes are live:

```sh
ssh -i ~/.ssh/id_ed25519 root@203.0.113.10 hopboxd admin control list
```

The output shows two live processes, with the internal URLs `https://203.0.113.10:7443` and `https://203.0.113.13:7443`.

### 5. Point the zone at the edge servers

The zone of the DNS role names the public addresses of the install. Until now, it names the peer address of the first server. Set the public addresses to the floating addresses of the edge servers. `ns1` takes the first address, and `ns2` the second ([the DNS role](/reference/dns/index.html.md#the-zone-of-the-install)):

```sh
ssh -i ~/.ssh/id_ed25519 root@203.0.113.10 hopboxd admin config set --public-address 198.51.100.5,198.51.100.6
```

Ask each edge server for the zone. Each answer names both floating addresses:

```sh
dig @198.51.100.5 example.com A +norec
dig @198.51.100.6 example.com A +norec
```

Then change the glue records of the domain at your registrar ([Serve the domain of an install with the DNS role](/guide/dns-role/index.html.md#make-the-records-in-the-parent-zone)): `ns1.example.com` at `198.51.100.5`, and `ns2.example.com` at `198.51.100.6`. A resolver keeps the old records for up to 60 seconds after the parent zone publishes the change.

### 6. Replace an edge server

To move `edge2` to a new host at `203.0.113.14`, set up that host as in [1. Set up the hosts](#1-set-up-the-hosts). Do not route the floating address to it yet. Change the address and the host key of `edge2` in `main.tf`:

```hcl
resource "hopbox_server" "edge2" {
  address            = "203.0.113.14"
  ssh_host_key       = "ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAI..."
  roles              = ["edge", "dns"]
  floating_addresses = ["198.51.100.6"]
  bundle             = local.bundle
  host_kernel        = local.host_kernel
  depends_on         = [hopbox_server.first]

  lifecycle {
    create_before_destroy = true
  }
}
```

Apply. A new `address` replaces the server. With `create_before_destroy`, Terraform creates the new server first, and the new server holds `198.51.100.6` on its uplink. Then Terraform destroys the old server: the provider decommissions it, stops `hopboxd`, and removes the address from its uplink.

When the apply prints `hopbox_server.edge2: Creation complete`, route `198.51.100.6` to the new host at the provider. The address gives no answer from the destroy of the old server until the provider routes it to the new one. In that time, clients reach the install at `198.51.100.5`, because the zone names both addresses since step 5. The public addresses of the install, and the records of the zone, stay the same.

The old host stays with its provider. If you do not use it again, end its rental there.

### 7. Let the first server leave the install

Once the zone names the edge servers, the first server can leave the install. The install keeps running on the other servers. Before you start, make sure that these conditions are true:

- A second control server is live, on the external Postgres, as in [step 4](#4-add-a-second-control-server).
- The first server holds no machine. Move each of its machines to another node with `hopboxd admin machine move` ([hopboxd](/reference/hopboxd/index.html.md#admin-machine-move)), or delete it.
- Each other server knows `cp2`. The `channel.control_urls` of a server made before `cp2` names only the first server. The control role sends the URLs of the live control servers on the control channel, and each server keeps them in `/var/lib/hopbox/server/control-urls` ([the control channel](/reference/control-channel/index.html.md#control-urls)). Read that file on each server, through the jump host. For example, on `node1`:

  ```sh
  ssh -i ~/.ssh/id_ed25519 -J root@203.0.113.10 root@203.0.113.21 cat /var/lib/hopbox/server/control-urls
  ```

  The output must list `https://203.0.113.13:7443`. If it does not, wait 5 s and read it again. If the file still does not list it, look for `cannot keep the control URLs` in the log of `hopboxd` on that server, with `journalctl -u hopboxd`. If the cause is a full disk, free space on it. The next heartbeat writes the file within 5 s.

Then take these steps:

1. Make `edge2` the jump host. Read its address and host key with `terraform state show hopbox_server.edge2`. Put its `address` and `ssh_host_key` in `ssh_jump_host` of the provider block. Apply.
2. Set up a host for a new edge server, `edge3`, at `203.0.113.15`, as in [1. Set up the hosts](#1-set-up-the-hosts).
3. In the resource `first`, set `floating_addresses = []`. Add `edge3` with the floating address of the first server:

   ```hcl
   resource "hopbox_server" "edge3" {
     address            = "203.0.113.15"
     ssh_host_key       = "ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAI..."
     roles              = ["edge", "dns"]
     floating_addresses = ["198.51.100.5"]
     bundle             = local.bundle
     host_kernel        = local.host_kernel
     depends_on         = [hopbox_server.cp2]
   }
   ```

4. Apply. The update of the first server removes `198.51.100.5` from its uplink. The create of `edge3` puts the address on its own uplink.
5. When the apply prints `hopbox_server.edge3: Creation complete`, route `198.51.100.5` to the host of `edge3` at the provider. Until then, `ns1` and the URLs of the install give no answer at `198.51.100.5`. Clients reach the install at `198.51.100.6` in that time.
6. Edit `main.tf`:
   - Remove the resource `first`, and the outputs `admin_api_key` and `first_host_key`, which name it. The provider keeps the admin API key of `HOPBOX_ADMIN_API_KEY` from step 2. Keep that key in a safe place first.
   - In `cp2`, remove `depends_on`. In each other resource, change `depends_on = [hopbox_server.first]` to `depends_on = [hopbox_server.cp2]`, so a rolling upgrade updates the control server first.
   - In `copy_control_from` of `cp2`, replace the references to `hopbox_server.first` with their values from `terraform state show hopbox_server.first`. Only the create of `cp2` reads them.
7. Apply. Terraform destroys the first server. The provider drains it through the admin API at `install_url`, because another control server is live. The first server holds no machine, so the drain decommissions it in about 2 minutes ([hopbox Terraform provider](/reference/terraform-provider/index.html.md#delete)). The provider stops `hopboxd`. Then `cp2` proves that it holds the same keys. The provider then removes the key files and the Postgres files of the server. Then it removes the table of the host firewall and the network setup.

List the servers through the jump host:

```sh
ssh -i ~/.ssh/id_ed25519 -J root@203.0.113.14 root@203.0.113.13 hopboxd admin server list
```

The output shows the first server as `decommissioned`, and every other server as `ready`. The host of the first server stays with its provider.

## Import a server into a new state

A server that the provider set up can move to a new state, for example after the loss of the state file. Write its resource in `main.tf` first, with its `address` and the `ssh_host_key` from the console of its provider. Then import it by its peer address:

```sh
terraform import hopbox_server.first 203.0.113.10
```

The import logs in at the address with the key of the operators, as `root`, then as `debian`. A server without the role `edge` gets the login through the jump host of the provider block. The import reads the server ID, the version, and the domain from the server. The install must know a server with that peer address, so an import never takes a server of another install ([hopbox Terraform provider](/reference/terraform-provider/index.html.md#import)).

The import has no pinned host key yet, so it keeps the key that the server shows. The next plan compares that key with `ssh_host_key`. If they differ, the plan fails with `The SSH host key of a server cannot change`, before any change of the server. Then compare both keys with the console of the provider.

The new state has no `admin_api_key`. Make a new admin API key as in step 6. The new state also forgets a cordon that a failed kernel reboot left. Run `hopboxd admin server get <id>` on a control server. If it shows `cordoned	true` and nobody cordoned the server on purpose, run `hopboxd admin server uncordon <id>`. The first plan after the import shows an update of `bundle`. If the bundle of `main.tf` is the release that the server runs, the apply changes nothing on the server.

## Destroy

For the first server, remove the block `lifecycle { prevent_destroy = true }` from `main.tf` first. Then run the destroy:

```sh
terraform destroy
```

The destroy drains the server, so its machines move to other nodes. On an install with one server, the drain has no other node, so delete the machines first. Then the destroy decommissions the server in the install, stops `hopboxd`, and removes the secrets of the install from the host. It removes the table of the host firewall, so the public address of the server takes SSH again. It also removes the network setup of an edge server, with its floating addresses ([hopbox Terraform provider](/reference/terraform-provider/index.html.md#delete)).

If the table stays on a server without the role `edge`, the destroy fails with the commands to run, and keeps the resource. The jump host still reaches the server. Fix the cause, and run `terraform destroy` again.

The host stays with its provider, with its disks as they are. A destroy never ends a rental. End it at the provider. To use the host again, apply `main.tf` again: the destroy removed the identity of the server, so the host takes a new create.

## If the apply stops

| Message | Fix |
| --- | --- |
| `the provider has no SSH key` | Set `ssh_private_key` in the provider block, or export `HOPBOX_SSH_PRIVATE_KEY`. |
| `remote: the host key of the server is not the pinned one` | The host shows another key than `ssh_host_key`. Either `address` names another host, the host was reinstalled, or a machine sits between you and the host. Compare `ssh_host_key` with the key in the console of the provider before you do anything else. |
| `log in to the server at <address> as <user>: ...` | The host does not take the key of the operators for `ssh_user`. Put the public key in the `authorized_keys` of that user, as in [step 1](#1-set-up-the-hosts). |
| `sshd of the server takes password logins: ...` | Turn off the password logins, as in [step 1](#1-set-up-the-hosts), and apply again. Nothing changed on the host. |
| `the host holds an install: hopboxd runs on the host at <address>, ...` | `address` names a host with a live install. Make sure that `address` is correct. Nothing changed on the host. |
| `the host holds an install: the host at <address> holds the identity of an enrolled server in <data_dir>/server, ...` | An earlier install left its identity on the host. Destroy its `hopbox_server`, or remove the dir by hand. |
| `the host holds an install: the host at <address> holds the control store of an earlier install in <data_dir>/pg, ...` | Move or remove `<data_dir>/pg` and `<data_dir>/kek` on the host, and apply again. |
| `the server runs <os> <version>, but hopbox tests only Debian 13` | Install Debian 13 on the host at its provider. |
| `the server at <address> holds no address <address> on a network interface, ...` | A NAT sits in front of the host, or `address` names another host. Use the public address that the host holds on its uplink. |
| `the host at <address> names no resolver that a machine can reach in ...` | The host of a node names only a resolver on loopback, such as a local `unbound`. Put a resolver that the host reaches over its uplink in `/etc/resolv.conf`, and apply again. Nothing changed on the host. |
| `/var/lib/hopbox is not a file system of its own. ...` or `/var/lib/hopbox is on <fs>, ...` | Make `/var/lib/hopbox` an XFS file system with reflinks, as in [step 1](#1-set-up-the-hosts). |
| `/var/lib/hopbox is on XFS without reflinks, and it holds files, so the provider does not make it again` | Move the files away, or make the file system again with `mkfs.xfs -m reflink=1`. |
| `the server <address> has no role edge, so its host firewall takes SSH only from the edge servers once hopboxd runs, ...` | Set `ssh_jump_host` in the provider block, as in [2. Set the jump host and the admin API](#2-set-the-jump-host-and-the-admin-api), and apply again. |
| `a server that is not the first one enrolls at the internal listener of a control server on its peer address, and the install lists none. ...` | The first server still listens on loopback. Apply the first server alone first, with `terraform apply -target=hopbox_server.first`, so its update moves its listener. |
| `set up the network of the server: ...` | The network setup of an edge server failed. Read `journalctl -u hopbox-network.service` on the server. A server without an IPv4 default route fails this step. |
| `/boot/grub/grub.cfg has no entry for <release>. The provider removed the pin, and the old default stays` | GRUB made no menu entry for the kernel, and the server still boots its old kernel. Read `/boot/grub/grub.cfg` on the server. If `<release>` is `6.18.57-1`, the provider is from before the fix of this error: it pinned the package version, not the kernel release. Build the provider again, and apply again. |
| `the server booted <release> after the install of 6.18.57-hopbox1` | GRUB booted another kernel. Read `/etc/default/grub.d/20_hopbox_kernel.cfg` and `/boot/grub/grub.cfg` on the server ([Host kernel](/reference/host-kernel/index.html.md#fallback)). |
| `the bundle: upload ...: remote: the connection broke: ...` | `sshd` dropped the SSH connection 6 times during the upload, and the provider gave up ([hopbox Terraform provider](/reference/terraform-provider/index.html.md#uploads)). Read `journalctl -u ssh` on the server. `message authentication code incorrect` means that data changed on the network path. Terraform tainted the server, so recover as below, from a better network. |
| `write the bootstrap config: ...` | The provider cannot write `/etc/hopbox/hopboxd.toml`. Read the error, which names the cause. Terraform tainted the server, so recover as below. |
| Warning `The provider did not set the public addresses of the install: ...` | The install runs, and Terraform did not taint the server. Run the command of the warning on the server, such as `sudo /usr/local/bin/hopboxd admin config set --public-address '203.0.113.10'`. Then `hopboxd admin config get` shows the addresses. |
| `the server <id> did not become ready` | Read `journalctl -u hopboxd` on the server. |
| `the first server <id> is decommissioned or unknown to its install` | Someone ran `hopboxd admin server decommission` on the first server. Its install cannot take machines again. To make a new install on the host, run `terraform state rm hopbox_server.first`, remove the install from the host, and apply. A destroy that lost its answer after the decommission gives the same error. Then run `terraform destroy -refresh=false`. |
| `... stays cordoned. Run hopboxd admin server uncordon <id>, or apply again` | A step of a kernel reboot failed. Fix the cause, and apply again. The apply lifts the cordon. |
| `The server keeps its host firewall` | The destroy of a server without the role `edge` kept the table. Fix the cause that the error names, or run its commands on the server through the jump host. Then run `terraform destroy` again. |
| `the install knows the server <id> with the peer address <peer address>, not <address>. Import it by its peer address` | The import ID is not the peer address of the server. Import it again with its peer address. |
| `The state of the server cannot move to this release` | A state of an earlier release has no `ipv4` or no pinned host key. Run `terraform state rm` on the resource, and import the server by its peer address. |

After a failed create, Terraform marks the server as tainted. The next apply destroys it, which undoes the steps that ran, and creates it again on the same host ([hopbox Terraform provider](/reference/terraform-provider/index.html.md#create)). With `lifecycle { prevent_destroy = true }`, Terraform refuses that apply with `Instance cannot be destroyed`. Then remove the block for that apply, and put it back after. `terraform untaint` does not help, because the state then holds a server without `hopboxd`, and the next apply installs nothing.

The destroy of a first server that never enrolled keeps its control store in `/var/lib/hopbox/pg`, and its keys in `/var/lib/hopbox/kek`, with a warning. The new create then refuses the host. Remove both dirs on the host first.
