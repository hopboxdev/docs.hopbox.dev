# Install hopbox on an OVHcloud server with Terraform

> How to install hopbox on an OVHcloud server with Terraform, and how to upgrade, import, and destroy it.

The network redesign of 2026-10-27 changes this page. hopbox leaves the vRack and the OVHcloud API, and the servers talk over their public addresses (Q90). This page still describes the current code. Milestones N26 to N32 change the code and this page. Before you set up a fleet on the vRack or with an IPv6 /56, read Q90.

This guide makes a new install of hopbox on one OVHcloud dedicated server with the hopbox Terraform provider. The server gets every role. The provider adopts a server that your account rents, or orders a new one ([Order a new server](#order-a-new-server)) and pays for it. Then the guide upgrades the server, changes its labels, imports it into a new state, and destroys it.

hopbox supports only the servers that this provider sets up. [hopbox Terraform provider](/reference/terraform-provider/index.html.md) lists every attribute and each step that the provider takes. [Install hopbox on a Debian 13 server by hand](/guide/install-debian/index.html.md) shows the same install by hand.

The release files of hopbox are not public yet. The hopbox team gives them on request.

The apply erases the server. Back up anything on it first.

## What you need

- An OVHcloud dedicated server with KVM, rented in your account. The [tested server](/reference/preflight/index.html.md#the-tested-server) has an Intel Xeon E5-1650 v4 CPU. You need its service name, such as `ns1234567.ip-198-51-100.eu`. The OVHcloud manager shows it. To order a new server instead, you need a payment method in the account and the budget for a monthly rental. See [Order a new server](#order-a-new-server).
- An OVHcloud API application and a consumer key with the rights of [step 1](#1-make-the-ovhcloud-api-keys).
- A Mac with Apple Silicon or a Linux host on x86_64, with Terraform 1.5 or later. This example uses the Mac.
- The release files of v0.1.0, from the hopbox team ([2. Check the release files](#2-check-the-release-files)).
- A domain for the install, such as `example.com`.
- An SSH key pair for the operators, without a passphrase. This example uses `~/.ssh/id_ed25519`.

## 1. Make the OVHcloud API keys

Make an application and a consumer key on the page `https://eu.api.ovh.com/createToken/`. Give the consumer key these rights, and no other:

| Method | Path |
| --- | --- |
| `GET` | `/dedicated/server/*` |
| `POST` | `/dedicated/server/*/reinstall` |

To order new servers, also give it these rights ([hopbox Terraform provider](/reference/terraform-provider/index.html.md#ovhcloud-api-rights)):

| Method | Path |
| --- | --- |
| `GET` | `/me` |
| `GET` | `/order/cart` |
| `POST` | `/order/cart` |
| `GET` | `/order/cart/*` |
| `POST` | `/order/cart/*` |
| `DELETE` | `/order/cart/*` |
| `GET` | `/me/order/*` |

A key with `POST /order/cart/*` can pay for orders with the payment method of the account. Give it only to the people who are allowed to spend that money.

To grow the install into a fleet on the vRack ([Grow the install into a fleet](#grow-the-install-into-a-fleet)), also give it these rights:

| Method | Path |
| --- | --- |
| `POST` | `/vrack/*/dedicatedServerInterface` |
| `DELETE` | `/vrack/*/dedicatedServerInterface/*` |
| `GET` | `/vrack/*/task/*` |
| `GET` | `/ip/*` |
| `POST` | `/ip/*/move` |
| `POST` | `/ip/*/park` |
| `GET` | `/ip/*/task/*` |

Keep the values in your shell, not in a file:

```sh
export OVH_ENDPOINT=ovh-eu
export OVH_APPLICATION_KEY=...
export OVH_APPLICATION_SECRET=...
export OVH_CONSUMER_KEY=...
```

## 2. Check the release files

Put these release files in one folder, such as `~/hopbox-release`:

| File | What it holds |
| --- | --- |
| `terraform-provider-hopbox_0.1.0_darwin_arm64.zip` | The provider for a Mac with Apple Silicon. Linux hosts use `terraform-provider-hopbox_0.1.0_linux_amd64.zip`. |
| `hopbox-v0.1.0-linux-amd64.tar.gz` and its `.sha256` file | The release bundle of the servers ([Release bundle](/reference/release-bundle/index.html.md)). |
| `linux-image-6.18.57-hopbox1_6.18.57-1_amd64.deb` and its `.sha256` file | The tested host kernel, a Debian package ([Host kernel](/reference/host-kernel/index.html.md)). |
| `SHA256SUMS` | The sha256 of each other file. |

Check each file against its line in `SHA256SUMS`. On Linux, use `sha256sum -c` in place of `shasum -a 256 -c`:

```sh
cd ~/hopbox-release
grep -e terraform-provider-hopbox_0.1.0_darwin_arm64.zip -e hopbox-v0.1.0-linux-amd64.tar.gz -e linux-image-6.18.57 SHA256SUMS |
  shasum -a 256 -c
```

The command prints `OK` for each of the four files. `SHA256SUMS` has no line for the `.sha256` file of the host kernel package. If it prints `FAILED`, get that file again.

The people who work on hopbox can build the same files from a checkout instead.

## 3. Write the Terraform files

The provider is in no registry. Put its zip in the local mirror of Terraform, in the dir of its source address:

```sh
dir=~/.terraform.d/plugins/registry.terraform.io/hopboxdev/hopbox
mkdir -p "$dir"
cp ~/hopbox-release/terraform-provider-hopbox_0.1.0_darwin_arm64.zip "$dir/"
```

Terraform reads the dir `~/.terraform.d/plugins` as a local mirror only without a `provider_installation` block in its CLI config. If your CLI config has such a block, add a `filesystem_mirror` block for that dir to it.

Make a folder for the install:

```sh
mkdir -p ~/hopbox-install && cd ~/hopbox-install
```

Write `main.tf`. Put the paths of step 2 in it, and the sha256 values of the two `.sha256` files:

```hcl
terraform {
  required_providers {
    hopbox = { source = "hopboxdev/hopbox", version = "0.1.0" }
  }
}

provider "hopbox" {
  ssh_private_key = file(pathexpand("~/.ssh/id_ed25519"))
}

resource "hopbox_server" "first" {
  ovh_service_name = "ns1234567.ip-198-51-100.eu"
  bootstrap        = true
  domain           = "example.com"
  labels           = { zone = "gra" }

  bundle = {
    source = pathexpand("~/hopbox-release/hopbox-v0.1.0-linux-amd64.tar.gz")
    sha256 = "<the sha256 of the bundle>"
  }
  host_kernel = {
    source = pathexpand("~/hopbox-release/linux-image-6.18.57-hopbox1_6.18.57-1_amd64.deb")
    sha256 = "<the sha256 of the package>"
  }
}

output "ipv4" {
  value = hopbox_server.first.ipv4
}

output "admin_api_key" {
  value     = hopbox_server.first.admin_api_key
  sensitive = true
}
```

The state holds the first admin API key of the install, and the pinned host key of the server. Keep the state in a place that only the instance admins can read. A `terraform.tfstate` file on a laptop is fine to try the provider. For a real install, use a backend with encryption and access control.

## 4. Apply

```sh
terraform init
terraform apply
```

The apply takes 30 to 90 minutes. Most of it is the reinstall at OVHcloud, and the reboot into the host kernel. To follow the steps, set `TF_LOG_PROVIDER=INFO`. The log shows lines such as `reinstalling the server`, `pinned the host key of the server`, `installing the host kernel`, and `installing hopboxd`.

At the end, Terraform prints the output `ipv4`. A warning such as `The preflight check of the server warns: ...` means that the server runs, with a risk that the warning names.

## 5. Check the server

The first login of the provider trusted the host key that the server showed, and pinned it in the state. Compare it with the key on the console of the server in the OVHcloud manager ([hopbox Terraform provider](/reference/terraform-provider/index.html.md#host-key-trust)):

```sh
terraform state show hopbox_server.first | grep ssh_host_key
```

Log in with the key of the operators. The OVHcloud template of Debian 13 has the user `debian`, with `sudo`:

```sh
ssh -i ~/.ssh/id_ed25519 debian@"$(terraform output -raw ipv4)" sudo hopboxd admin server list
```

The output shows one server, `ready`, with the version of the bundle:

```text
ID                                    POOL     STATE  STATUS  ROLES          HOSTNAME   VERSION  LAST HEARTBEAT
01a14404-ce7b-72d9-b18e-5006c3e462db  default  ready  ok      node,dns,edge  ns1234567  v0.1.0   2026-10-16T09:00:05Z
```

The ROLES column lists the roles that enrolled. The control role of the first server runs in the same `hopboxd` from its bootstrap config, without a server record.

`uname -r` on the server prints `6.18.57-hopbox1`.

The server also has the DNS role, which answers DNS for the domain of the install. The provider set the public addresses of the install to the addresses of the server, so the domain points at the server. The DNS role listens on port 53 of these addresses, because `systemd-resolved` of the OS holds port 53 on `127.0.0.53` ([hopbox Terraform provider](/reference/terraform-provider/index.html.md#create)). Read them:

```sh
ssh -i ~/.ssh/id_ed25519 debian@"$(terraform output -raw ipv4)" sudo hopboxd admin config get
```

The line `public_addresses` holds the IPv4 address of the server. If OVHcloud reports a `/128` block that a network interface of the server holds, the line holds that IPv6 address too. Then delegate the domain from its parent zone, as [Serve the domain of an install with the DNS role](/guide/dns-role/index.html.md#delegate-the-domain-of-the-install) shows. Use the same addresses for the glue records.

The server also has the edge role. It serves the API, the console, and the machine URLs on ports 443 and 80 ([the edge role](/reference/edge/index.html.md)). It starts HTTPS once the ACME job got the certificate of the domain ([Get the wildcard certificate of an install](/guide/acme-certificate/index.html.md)). Open TCP ports 80 and 443 in each firewall in front of the server.

## 6. Keep the admin API key

```sh
terraform output -raw admin_api_key
```

The key starts with `hbxadm_`. It is the first admin API key of the install, with the name `bootstrap`. It expires after 90 days. Before it expires, make a new key on the server and use it in the provider block of your other configurations:

```sh
ssh -i ~/.ssh/id_ed25519 debian@"$(terraform output -raw ipv4)" sudo hopboxd admin admin-key create --name terraform --lifetime 365d
```

## 7. Sign in

Make a login link for the first instance admin, and open it ([First sign-in on a new install](/guide/first-sign-in/index.html.md)):

```sh
ssh -i ~/.ssh/id_ed25519 debian@"$(terraform output -raw ipv4)" sudo hopboxd admin login-link --email you@example.org
```

## Upgrade

Get the bundle of the new release and its `.sha256` file. Change `bundle` in `main.tf` to its path and sha256, and apply:

```sh
terraform apply
```

The plan shows an update in place of `bundle` and `version`. The provider uploads the bundle and runs `hopboxd upgrade` on the server. The running machines stay up ([hopboxd](/reference/hopboxd/index.html.md#upgrade)). If the new release cannot run on the data, the upgrade stops before the switch, and the old release keeps running.

The provider then waits until the server is `ready` on the new version. An install with more servers upgrades them one at a time, the first server before the nodes ([hopbox Terraform provider](/reference/terraform-provider/index.html.md#rolling-upgrade)).

## Change the labels

Change `labels` in `main.tf`, for example to `{ zone = "gra", disk = "nvme" }`, and apply. The provider calls `SetServerLabels`. `sudo hopboxd admin server get <id>` on the server then shows `labels	disk=nvme,zone=gra` and `labels_set_by_admin	true`.

## Add pools and sizes

After the first server runs, you can add pools and sizes in the same Terraform files. A pool groups the servers of one model, and a size can keep its machines in one pool. Both need the admin API, so set `install_url` and `admin_api_key` in the provider block first, with the key of step 6:

```hcl
provider "hopbox" {
  ssh_private_key = file("~/.ssh/id_ed25519")
  install_url     = "https://example.com"
}

resource "hopbox_pool" "big" {
  name     = "big"
  ovh_plan = "24adv01"
}

resource "hopbox_size" "large" {
  name       = "large"
  vcpus      = 8
  memory_gib = 32
  disk_gb    = 200
  pool       = hopbox_pool.big.name
}
```

Keep the key out of the files: export it as `HOPBOX_ADMIN_API_KEY` before the apply. The first server stays in the pool `default`. A server that joins `big` sets `pool = hopbox_pool.big.name`. [Add pools and sizes](/guide/pools-and-sizes/index.html.md) explains the rules, and [hopbox Terraform provider](/reference/terraform-provider/index.html.md#resource-hopbox_pool) lists every attribute.

## Grow the install into a fleet

A fleet runs the roles on several servers: edge servers that take the public traffic, control servers that share one external Postgres, and nodes that run machines. The servers talk over the vRack, the private network of OVHcloud. This section builds a fleet of five servers in two pools:

| Resource | Roles | Pool | vRack address | Floating address | IPv6 address |
| --- | --- | --- | --- | --- | --- |
| `first` | `control`, `node`, `dns`, `edge` | `default` | `172.31.0.11` | `203.0.113.5` | none |
| `edge2` | `edge`, `dns` | `default` | `172.31.0.12` | `203.0.113.6` | `2001:db8:0:100::12` |
| `cp2` | `control`, `node` | `default` | `172.31.0.13` | none | none |
| `node1` | `node` | `default` | `172.31.0.21` | none | none |
| `node2` | `node` | `big` | `172.31.0.22` | none | none |

The first server joins the vRack at its create, so a fleet starts with a new install. The provider never moves a server that runs onto the vRack. A plan that gives such a server a `vrack_address` fails with `The vRack address of a server cannot change`.

### What a fleet needs

- The rights of the fleet in [step 1](#1-make-the-ovhcloud-api-keys).
- A vRack in the OVHcloud account, such as `pn-12345`, and servers of a range that offers the vRack. The OVHcloud manager shows both.
- One Additional IPv4 address for each edge server, routed to a server of the account and not to the vRack. This example uses `203.0.113.5` and `203.0.113.6`.
- For IPv6, an Additional IPv6 block of the account, routed into the vRack in the OVHcloud manager, and the gateway of the block. This example uses the block `2001:db8:0:100::/56` with the gateway `2001:db8:0:100::1`. Each edge server takes one address of the block ([Run edge servers apart from the nodes](/guide/edge-servers/index.html.md)).
- For the second control server, an external Postgres, such as managed PostgreSQL of OVHcloud on the vRack ([Run the control store on an external Postgres](/guide/external-postgres/index.html.md)).

### 1. Make the first server on the vRack

Start from `main.tf` of step 3. Name the vRack in the provider block, and give the first server its vRack address and its floating address:

```hcl
provider "hopbox" {
  ssh_private_key = file(pathexpand("~/.ssh/id_ed25519"))
  ovh_vrack       = "pn-12345"
}

locals {
  bundle = {
    source = pathexpand("~/hopbox-release/hopbox-v0.1.0-linux-amd64.tar.gz")
    sha256 = "<the sha256 of the bundle>"
  }
  host_kernel = {
    source = pathexpand("~/hopbox-release/linux-image-6.18.57-hopbox1_6.18.57-1_amd64.deb")
    sha256 = "<the sha256 of the package>"
  }
}

resource "hopbox_server" "first" {
  ovh_service_name   = "ns1234567.ip-198-51-100.eu"
  bootstrap          = true
  domain             = "example.com"
  vrack_address      = "172.31.0.11"
  floating_addresses = ["203.0.113.5"]
  bundle             = local.bundle
  host_kernel        = local.host_kernel
}

output "first_host_key" {
  value = hopbox_server.first.ssh_host_key
}
```

Apply, as in step 4:

```sh
terraform apply
```

The provider attaches the server to the vRack before the reinstall. After the reinstall, it gives the server its vRack address with the unit `hopbox-network.service`, which sets the address again at each boot. The internal listener of the control role listens on `172.31.0.11:7443`, so the other servers can enroll there. At the end, the floating address routes to the server. The zone of the DNS role names the public addresses of the server for now. [Step 6](#6-point-the-zone-at-the-edge-servers) points it at the edge servers.

Check the network of the server:

```sh
ssh -i ~/.ssh/id_ed25519 debian@"$(terraform output -raw ipv4)" ip -br addr
```

The output shows `172.31.0.11/16` on the vRack interface, and `203.0.113.5/32` on the public interface.

### 2. Set the jump host and the admin API

The other servers enroll through the admin API, so the provider needs `install_url` and an admin API key ([6. Keep the admin API key](#6-keep-the-admin-api-key)). Export the key, and keep it out of the files:

```sh
export HOPBOX_ADMIN_API_KEY="$(terraform output -raw admin_api_key)"
terraform output -raw first_host_key
```

The first server is an edge server on the vRack, so it is the jump host of the provider. The jump host must be on the vRack, because the provider dials the vRack address of each other server through it. Put its address and the host key of the output in the provider block:

```hcl
provider "hopbox" {
  ssh_private_key = file(pathexpand("~/.ssh/id_ed25519"))
  install_url     = "https://example.com"
  ovh_vrack       = "pn-12345"

  ssh_jump_host = {
    address  = "<the ipv4 output>"
    host_key = "<the first_host_key output>"
  }
}
```

From then on, every SSH step of a server without the role `edge` goes through the first server. It dials the vRack address of the server ([hopbox Terraform provider](/reference/terraform-provider/index.html.md#jump-host)). The host firewall of each node and each control server closes its public address once `hopboxd` runs on it ([the host firewall](/reference/host-firewall/index.html.md)). Port 22 of its vRack address then takes only the vRack addresses of the edge servers. So an edge server as the jump host is then the only way in.

### 3. Add an edge server and the nodes

Add a pool for the bigger nodes, as in [Add pools and sizes](#add-pools-and-sizes), and the servers:

```hcl
resource "hopbox_pool" "big" {
  name     = "big"
  ovh_plan = "24adv01"
}

resource "hopbox_server" "edge2" {
  ovh_service_name    = "ns2345678.ip-198-51-100.eu"
  roles               = ["edge", "dns"]
  vrack_address       = "172.31.0.12"
  floating_addresses  = ["203.0.113.6"]
  public_ipv6         = "2001:db8:0:100::12/56"
  public_ipv6_gateway = "2001:db8:0:100::1"
  bundle              = local.bundle
  host_kernel         = local.host_kernel
  depends_on          = [hopbox_server.first]

  lifecycle {
    create_before_destroy = true
  }
}

resource "hopbox_server" "node1" {
  ovh_service_name = "ns3456789.ip-198-51-100.eu"
  vrack_address    = "172.31.0.21"
  bundle           = local.bundle
  host_kernel      = local.host_kernel
  depends_on       = [hopbox_server.first]
}

resource "hopbox_server" "node2" {
  ovh_service_name = "ns4567890.ip-198-51-100.eu"
  pool             = hopbox_pool.big.name
  vrack_address    = "172.31.0.22"
  bundle           = local.bundle
  host_kernel      = local.host_kernel
  depends_on       = [hopbox_server.first]
}
```

Give each server its own vRack address. `public_ipv6` gives `edge2` its address of the IPv6 block, with the prefix length of the block. The provider writes it to the bootstrap config of the server, and `hopboxd` holds it on the vRack interface ([the edge role](/reference/edge/index.html.md#the-public-ipv6-address)). `depends_on` makes Terraform update the first server before the others in a [rolling upgrade](/reference/terraform-provider/index.html.md#rolling-upgrade). Apply:

```sh
terraform apply
```

The servers enroll at the internal listener of the first server, which the install lists. List the servers on the first server:

```sh
ssh -i ~/.ssh/id_ed25519 debian@"$(terraform output -raw ipv4)" sudo hopboxd admin server list
```

The output shows four servers, each `ready`: the first server, `edge2` with the roles `dns,edge`, and the two nodes, `node2` in the pool `big`.

Check the host firewall of `node1`. From your laptop, a connection to its public address must time out, also on port 22:

```sh
nc -vz -w 3 "$(terraform state show -no-color hopbox_server.node1 | awk '$1 == "ipv4" {gsub(/"/, "", $3); print $3}')" 22
```

Through the jump host, list the jump hosts that `node1` takes on port 22:

```sh
ssh -i ~/.ssh/id_ed25519 -J debian@203.0.113.5 debian@172.31.0.21 sudo nft list set inet hopbox_host jump_hosts
```

The set holds `172.31.0.11` and `172.31.0.12`, the vRack addresses of the two edge servers. If it is empty, wait 10 s and list it again: until the first mesh, port 22 takes the whole vRack range ([the host firewall](/reference/host-firewall/index.html.md#the-jump-hosts)).

### 4. Add a second control server

A second control server shares the control store with the first one, so the install must run on the external Postgres first. Move the store as [Run the control store on an external Postgres](/guide/external-postgres/index.html.md#move-a-one-server-install-to-the-external-postgres) shows. Then add the server:

```hcl
resource "hopbox_server" "cp2" {
  ovh_service_name = "ns5678901.ip-198-51-100.eu"
  roles            = ["control", "node"]
  vrack_address    = "172.31.0.13"
  bundle           = local.bundle
  host_kernel      = local.host_kernel
  depends_on       = [hopbox_server.first]

  copy_control_from = {
    address  = hopbox_server.first.vrack_address
    host_key = hopbox_server.first.ssh_host_key
    user     = hopbox_server.first.ssh_user
  }
}
```

`copy_control_from` names the live control server that `cp2` copies from, with its pinned host key. Apply. Before the reinstall, the provider logs in to the first server through the jump host, and reads its key-encryption keys and the settings of its external Postgres. It checks the fingerprint of each key against the admin API, and writes the files on `cp2` before the install. So both control servers seal with the same keys, and use the same store ([hopbox Terraform provider](/reference/terraform-provider/index.html.md#control-servers)). If the first server still runs the bundled Postgres, the apply stops before the reinstall with `the control server runs the bundled Postgres`.

Check that both control processes are live:

```sh
ssh -i ~/.ssh/id_ed25519 debian@"$(terraform output -raw ipv4)" sudo hopboxd admin control list
```

The output shows two live processes, with the internal URLs `https://172.31.0.11:7443` and `https://172.31.0.13:7443`.

### 5. Replace an edge server

To replace `edge2` with the server `ns6789012.ip-198-51-100.eu`, change its service name and its vRack address in `main.tf`:

```hcl
resource "hopbox_server" "edge2" {
  ovh_service_name    = "ns6789012.ip-198-51-100.eu"
  roles               = ["edge", "dns"]
  vrack_address       = "172.31.0.14"
  floating_addresses  = ["203.0.113.6"]
  public_ipv6         = "2001:db8:0:100::12/56"
  public_ipv6_gateway = "2001:db8:0:100::1"
  bundle              = local.bundle
  host_kernel         = local.host_kernel
  depends_on          = [hopbox_server.first]

  lifecycle {
    create_before_destroy = true
  }
}
```

Apply. With `create_before_destroy`, Terraform creates the new server first. The new server takes `203.0.113.6` once it is ready, so the address keeps answering. Then Terraform destroys the old server: the provider decommissions it, and takes it out of the vRack. The old server and the new one run side by side for a while, so they need two vRack addresses.

Keep `public_ipv6` the same, so the new server takes the same IPv6 address. While the old server holds it, the new server logs that another host on the vRack holds the address. Once the old server leaves, the new server takes the address within a few seconds. So the public addresses of the install, and the records of the zone, stay the same.

### 6. Point the zone at the edge servers

The zone of the DNS role names the public addresses of the install. Until now, they are the addresses of the first server. Set them to the floating addresses and the IPv6 addresses of the edge servers. `ns1` takes the first address of each family, and `ns2` the second ([the DNS role](/reference/dns/index.html.md#the-zone-of-the-install)):

```sh
ssh -i ~/.ssh/id_ed25519 debian@"$(terraform output -raw ipv4)" sudo hopboxd admin config set --public-address 203.0.113.5,203.0.113.6,2001:db8:0:100::12
```

Ask each edge server for the zone. Each answer names both floating addresses:

```sh
dig @203.0.113.5 example.com A +norec
dig @203.0.113.6 example.com A +norec
dig @2001:db8:0:100::12 example.com AAAA +norec
```

Then change the glue records of the domain at your registrar ([Serve the domain of an install with the DNS role](/guide/dns-role/index.html.md#make-the-records-in-the-parent-zone)): `ns1.example.com` at `203.0.113.5`, `ns2.example.com` at `203.0.113.6`, and both at `2001:db8:0:100::12` for IPv6. A resolver keeps the old records for up to 60 seconds after the parent zone publishes the change.

### 7. Let the first server leave the install

Once the zone names the edge servers, the first server can leave the install. The install keeps running on the other servers. Before you start, check these conditions:

- A second control server is live, on the external Postgres, as in [step 4](#4-add-a-second-control-server).
- The first server holds no machine. Move each of its machines to another node with `hopboxd admin machine move` ([hopboxd](/reference/hopboxd/index.html.md#admin-machine-move)), or delete it.
- Each other server knows `cp2`. The `channel.control_urls` of a server made before `cp2` names only the first server. The control role sends the URLs of the live control servers on the control channel, and each server keeps them in `/var/lib/hopbox/server/control-urls` ([the control channel](/reference/control-channel/index.html.md#control-urls)). Read that file on each server, through the jump host. For example, on `node1`:

  ```sh
  ssh -i ~/.ssh/id_ed25519 -J debian@203.0.113.6 debian@172.31.0.21 sudo cat /var/lib/hopbox/server/control-urls
  ```

  The output must list `https://172.31.0.13:7443`. If it does not, wait 5 s and read it again. If the file still does not list it, look for `cannot keep the control URLs` in the log of `hopboxd` on that server, with `journalctl -u hopboxd`. If the cause is a full disk, free space on it. The next heartbeat writes the file within 5 s.

Then take these steps:

1. Make `edge2` the jump host. Read its address and host key with `terraform state show hopbox_server.edge2`, and put its `ipv4` and `ssh_host_key` in `ssh_jump_host` of the provider block. Apply.
2. Give the floating address of the first server to a new edge server. In the resource `first`, set `floating_addresses = []`. Add the new server, with its own vRack address and its own address of the IPv6 block:

   ```hcl
   resource "hopbox_server" "edge3" {
     ovh_service_name    = "ns7890123.ip-198-51-100.eu"
     roles               = ["edge", "dns"]
     vrack_address       = "172.31.0.15"
     floating_addresses  = ["203.0.113.5"]
     public_ipv6         = "2001:db8:0:100::13/56"
     public_ipv6_gateway = "2001:db8:0:100::1"
     bundle              = local.bundle
     host_kernel         = local.host_kernel
     depends_on          = [hopbox_server.cp2]
   }
   ```

   Apply. The new server takes `203.0.113.5` at the end of its create. Terraform can update the first server before that. The first server then keeps `203.0.113.5` while OVHcloud routes it there, so `ns1` and the URLs of the install keep answering. In step 5, the first server leaves the install. OVHcloud routes the address to `edge3` already.
3. Add the IPv6 address of `edge3` to the public addresses. List it first of its family, so `ns1` names it:

   ```sh
   ssh -i ~/.ssh/id_ed25519 debian@"$(terraform output -raw ipv4)" sudo hopboxd admin config set --public-address 203.0.113.5,203.0.113.6,2001:db8:0:100::13,2001:db8:0:100::12
   ```

   Then change the AAAA glue record of `ns1.example.com` at your registrar to `2001:db8:0:100::13`.
4. Edit `main.tf`:
   - Remove the resource `first`, and the outputs `ipv4`, `admin_api_key`, and `first_host_key`, which name it. The provider keeps the admin API key of `HOPBOX_ADMIN_API_KEY` from step 2, so keep that key in a safe place first.
   - In `cp2`, remove `depends_on`. In each other resource, change `depends_on = [hopbox_server.first]` to `depends_on = [hopbox_server.cp2]`, so a rolling upgrade updates the control server first.
   - In `copy_control_from` of `cp2`, replace the references to `hopbox_server.first` with their values from `terraform state show hopbox_server.first`. Only the create of `cp2` reads them.
5. Apply. Terraform destroys the first server. The provider drains it through the admin API at `install_url`, because another control server is live. The first server holds no machine, so the drain decommissions it after its empty checks, in about 2 minutes ([hopbox Terraform provider](/reference/terraform-provider/index.html.md#delete)). It stops `hopboxd`, and removes the key files and the Postgres files of the server once `cp2` proves that it holds the same keys. Then it takes the server out of the vRack.

Check the servers through the jump host:

```sh
ssh -i ~/.ssh/id_ed25519 -J debian@203.0.113.6 debian@172.31.0.13 sudo hopboxd admin server list
```

The output shows the first server as `decommissioned`, and every other server as `ready`.

## Import a server into a new state

A server that the provider set up can move to a new state, for example after the loss of the state file:

```sh
terraform import hopbox_server.first ns1234567.ip-198-51-100.eu
```

The import logs in with the key of the operators. It reads the server ID, the version, and the domain from the server. The new state has no `admin_api_key`. Make a new admin API key as in step 6. The new state also forgets a cordon that a failed kernel reboot left. Check the server with `sudo hopboxd admin server get <id>`. If it shows `cordoned	true` and nobody cordoned it on purpose, run `sudo hopboxd admin server uncordon <id>`. The first plan after the import shows an update of `bundle`. If the bundle of `main.tf` is the release that the server runs, the apply changes nothing on the server.

The host firewall of a node or a control server closes its public address. So with `ssh_jump_host`, give its vRack address after the service name, with a comma:

```sh
terraform import hopbox_server.node1 ns2345678.ip-198-51-100.eu,172.31.0.21
```

The import then logs in at the vRack address through the jump host. An edge server keeps SSH open on its public address, so it takes the service name alone.

## Order a new server

The provider can order the server in place of a server that your account rents. The apply pays for it with the preferred payment method of the account. It rents the server by the month, without a commitment.

Find the plan code of a server model, and the options of its families, in the public catalog of OVHcloud. This call needs no key:

```sh
curl -s 'https://eu.api.ovh.com/1.0/order/catalog/public/baremetalServers?ovhSubsidiary=FR' -o catalog.json
jq -r '.plans[] | [.planCode, .invoiceName] | @tsv' catalog.json
jq '.plans[] | select(.planCode == "<plan code>") | {configurations, families: [.addonFamilies[] | {name, mandatory, addons}]}' catalog.json
```

The `configurations` show the datacenters of the plan, in `dedicated_datacenter`. Pick one option of each family with `"mandatory": true`, such as the memory, the storage, and the bandwidth. For the storage, pick a `softraid-` option: the provider refuses a `hardraid-` option, because hopbox tests only software RAID. All the servers of a pool have the same model, so use the same plan and options for each server of a pool.

In `main.tf` of step 3, replace `ovh_service_name` with the order:

```hcl
resource "hopbox_server" "first" {
  ovh_plan       = "<plan code>"
  ovh_datacenter = "gra"
  ovh_options    = ["<memory option>", "<storage option>", "<bandwidth option>"]
  ovh_order_key  = "hop-example-first"
  ovh_max_price  = 120
  bootstrap      = true
  domain         = "example.com"
  labels         = { zone = "gra" }

  bundle = {
    source = pathexpand("~/hopbox-release/hopbox-v0.1.0-linux-amd64.tar.gz")
    sha256 = "<the sha256 of the bundle>"
  }
  host_kernel = {
    source = pathexpand("~/hopbox-release/linux-image-6.18.57-hopbox1_6.18.57-1_amd64.deb")
    sha256 = "<the sha256 of the package>"
  }

  lifecycle {
    prevent_destroy = true
  }
}
```

`ovh_max_price` is the highest price with tax that the checkout pays, in the currency of your account. The checkout pays the first invoice: the first month and any setup fee, not only the monthly rent. If the cart costs more, the apply stops before the checkout and orders nothing. The error then shows the price of the cart. `ovh_order_key` names the order in your OVHcloud account. Give each server its own key. The provider marks the cart of the order with it. So a later apply finds an order that an apply lost, and does not pay twice.

Plan first, and read the warning:

```sh
terraform plan
```

The plan says `Apply orders and pays for a new OVHcloud server`, with the plan, the datacenter, and the options. Only a create orders. Then apply:

```sh
terraform apply
```

The apply output shows the price of the checkout in the warning `The apply ordered the OVHcloud server of the order <id>`. The log of the provider shows `ordering a server` with the price, `ordered a server` with the order ID, and `waiting for the order` with its status. OVHcloud delivers some models in minutes, and others in days. The apply waits up to 2 hours. Then it reinstalls the server and goes on as in step 4. `terraform state show hopbox_server.first` shows the service name of the new server in `ovh_service_name`, and the order in `ovh_order_id`.

If OVHcloud did not deliver the server in 2 hours, the apply stops with `the order <id> is not delivered after 2h0m0s`, and Terraform marks the resource as tainted. Do not remove the resource. Untaint it and apply again later:

```sh
terraform untaint hopbox_server.first
terraform apply
```

The apply waits for the same order, and orders nothing new. Do the same after a failure of a later step, such as the install.

After the create, the plan refuses each change that orders a new server: another `ovh_plan`, `ovh_datacenter`, `ovh_options`, or `ovh_order_key`, and a change of `pool` or `bootstrap`. To change the model of a server, add a new resource and remove the old one.

`terraform taint`, `terraform apply -replace`, and `create_before_destroy` also replace the server, and the replacement orders a new one. The provider cannot see these replacements. The block `lifecycle { prevent_destroy = true }` makes Terraform refuse them. Keep it on each ordered server, and remove it only before a destroy on purpose.

## Destroy

```sh
terraform destroy
```

The destroy decommissions the server in the install, and stops `hopboxd` on it. Then it removes the table of the host firewall, so the public address of the server takes SSH again. A node or a control server leaves the vRack only after that. If the table stays, the destroy keeps the server in the vRack, and fails with the commands to run. It refuses a server that holds machines. Delete the machines first. The OVHcloud server stays rented, with its disks as they are. With `ovh_service_name`, another apply of `main.tf` erases the server again and makes a new install.

The destroy never cancels an order and never ends a rental, also for a server that the provider ordered. For an ordered server, remove `prevent_destroy` from `main.tf` before the destroy. After the destroy, another apply of `main.tf` with the order attributes orders and pays for a new server. To make a new install on the same server, replace the order attributes with `ovh_service_name` of that server first. To stop paying for the server, end its rental in the OVHcloud manager, and remove the resource from `main.tf`.

## If the apply stops

| Message | Fix |
| --- | --- |
| `the provider has no OVHcloud credentials` | Export the variables of step 1 in the shell that runs Terraform. |
| `ovh: ... HTTP 403 ... This call has not been granted` | The consumer key lacks a right of step 1. Make a new consumer key. |
| `ovh: GET /dedicated/server/<name>: HTTP 404` | The service name is wrong, or the server is in another account. |
| `the disk group <n> sits behind the hardware RAID controller` | Pick a server model without hardware RAID. |
| `log in to the reinstalled server ...: context deadline exceeded` | The server did not answer SSH for 30 minutes after the reinstall. Check the server in the OVHcloud manager, then run `terraform apply` again. Terraform reinstalls the server. |
| `remote: the host key of the server is not the pinned one` | Someone reinstalled the server outside Terraform, or a machine sits between you and the server. Check the key first. A reinstall of the first server erased its install. To accept it, run `terraform state rm hopbox_server.first` and apply, which makes a new install. For an ordered server, first replace the order attributes in `main.tf` with `ovh_service_name` of the same server. Otherwise the apply orders a new server. |
| `/boot/grub/grub.cfg has no entry for <release>. The provider removed the pin, and the old default stays` | GRUB made no menu entry for the kernel, and the server still boots its old kernel. Read `/boot/grub/grub.cfg` on the server. If `<release>` is `6.18.57-1`, the provider is from before the fix of this error: it pinned the package version, not the kernel release. Build the provider again, and adopt the server again. |
| `the server booted <release> after the install of 6.18.57-hopbox1` | GRUB booted another kernel. Read `/etc/default/grub.d/20_hopbox_kernel.cfg` and `/boot/grub/grub.cfg` on the server ([Host kernel](/reference/host-kernel/index.html.md#fallback)). |
| `the bundle: upload ...: remote: the connection broke: ...` | sshd dropped the SSH connection 6 times during the upload, and the provider gave up ([hopbox Terraform provider](/reference/terraform-provider/index.html.md#uploads)). Read `journalctl -u ssh` on the server. `message authentication code incorrect` means that data changed on the network path. Terraform tainted the server, so recover as below, from a better network. |
| `read the IP addresses of the server <name>: ...` | The OVHcloud API did not give the IP blocks. The apply stopped before the reinstall, so the server is as it was. Check the rights of step 1, and apply again. |
| `the server <name> holds none of its IP addresses <blocks> on a network interface` | The reinstall erased the server, but the OS set up none of the addresses that OVHcloud reports. Read `ip addr` on the server and the IP blocks in the OVHcloud manager. Terraform tainted the server, so the next apply reinstalls it. |
| `write the bootstrap config: ...` | The provider cannot write `/etc/hopbox/hopboxd.toml`. Read the error, which names the cause. Terraform tainted the server, so the next apply reinstalls it. |
| Warning `The provider did not set the public addresses of the install: ...` | The install runs, and Terraform did not taint the server. Run the command of the warning on the server, such as `sudo /usr/local/bin/hopboxd admin config set --public-address '203.0.113.10'`. Then `sudo hopboxd admin config get` shows the addresses. |
| `the server <id> did not become ready` | Read `sudo journalctl -u hopboxd` on the server. |
| `the first server <id> is decommissioned or unknown to its install` | Someone ran `hopboxd admin server decommission` on the first server. Its install cannot take machines again. To make a new install on it, run `terraform state rm hopbox_server.first` and apply. That erases the server. For an ordered server, first replace the order attributes in `main.tf` with `ovh_service_name` of the same server. Otherwise the apply orders a new server. A destroy that lost its answer after the decommission gives the same error. Then run `terraform destroy -refresh=false`. |
| `... stays cordoned. Run hopboxd admin server uncordon <id>, or apply again` | A step of a kernel reboot failed. Fix the cause, and apply again. The apply lifts the cordon. |
| `the ovh_options of the plan <plan>: ...` | An option is wrong, or a mandatory family has no option. Read the families of the plan in the catalog. No order was made. |
| `the order <id> is not delivered after ...` | OVHcloud did not deliver the server in time. For an order that is `notPaid`, pay it in the OVHcloud manager. Then run `terraform untaint hopbox_server.first` and `terraform apply`. |
| `the setup of the server of the OVHcloud order <id> did not finish, so the provider keeps the resource` | Terraform tried to replace a tainted ordered server. Run `terraform untaint hopbox_server.first` and apply. To give the server up, run `terraform state rm hopbox_server.first`. Remove the resource or its order attributes from `main.tf`, and end the rental in the OVHcloud manager. |
| `an earlier apply checked out the OVHcloud cart <cart ID> for the ovh_order_key "<key>"` | An apply ordered a server and lost the order, for example because Terraform stopped during the checkout. Find the order of that cart in the OVHcloud manager. Set `ovh_order_id = <order ID>` in the resource, and apply. The apply follows that order. |
| `the order <id> is cancelled at OVHcloud` | No server comes from the order. Run `terraform state rm hopbox_server.first`, and apply to order again. |
| `The order of a server cannot change` or `The change orders a new server` | The change needs a new server. Add a new resource with its own `ovh_order_key`, and remove the old one. |
| `the server <name> has a vrack_address and no role edge, so its host firewall closes its public address once hopboxd runs, ...` | Set `ssh_jump_host` in the provider block, as in [2. Set the jump host and the admin API](#2-set-the-jump-host-and-the-admin-api), and apply again. |
| `the server <id> keeps its host firewall, because ...` | The destroy kept the server in the vRack. Fix the cause that the error names, or run its commands on the server through the jump host. Then run `terraform destroy` again. |
| `log in to the server <name> to import it at its public address through the jump host: ...` | The host firewall of the server closes its public address. Import it with its vRack address, as in [Import a server into a new state](#import-a-server-into-a-new-state). |
| `the server at the vRack address <address> does not hold the public address <address> of the server <name>, so it is another server` | The import ID names the vRack address of another server. Read the vRack address of the server in your Terraform files, and import again. |

After a failed create of a server that your account rents, Terraform marks the server as tainted. The next apply starts again with a new reinstall. With `lifecycle { prevent_destroy = true }`, Terraform refuses that apply with `Instance cannot be destroyed`. Then run `terraform state rm hopbox_server.first`, and apply again. `terraform untaint` does not help, because the state then holds a server without `hopboxd`, and the next apply installs nothing. A failed create of an ordered server works another way: see [Order a new server](#order-a-new-server).
