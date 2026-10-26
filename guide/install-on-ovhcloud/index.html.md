# Install hopbox on an OVHcloud server with Terraform

> How to install hopbox on an OVHcloud server with Terraform, and how to upgrade, import, and destroy it.

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

## Import a server into a new state

A server that the provider set up can move to a new state, for example after the loss of the state file:

```sh
terraform import hopbox_server.first ns1234567.ip-198-51-100.eu
```

The import logs in with the key of the operators. It reads the server ID, the version, and the domain from the server. The new state has no `admin_api_key`. Make a new admin API key as in step 6. The new state also forgets a cordon that a failed kernel reboot left. Check the server with `sudo hopboxd admin server get <id>`. If it shows `cordoned	true` and nobody cordoned it on purpose, run `sudo hopboxd admin server uncordon <id>`. The first plan after the import shows an update of `bundle`. If the bundle of `main.tf` is the release that the server runs, the apply changes nothing on the server.

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

The destroy decommissions the server in the install, and stops `hopboxd` on it. It refuses a server that holds machines. Delete the machines first. The OVHcloud server stays rented, with its disks as they are. With `ovh_service_name`, another apply of `main.tf` erases the server again and makes a new install.

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

After a failed create of a server that your account rents, Terraform marks the server as tainted. The next apply starts again with a new reinstall. With `lifecycle { prevent_destroy = true }`, Terraform refuses that apply with `Instance cannot be destroyed`. Then run `terraform state rm hopbox_server.first`, and apply again. `terraform untaint` does not help, because the state then holds a server without `hopboxd`, and the next apply installs nothing. A failed create of an ordered server works another way: see [Order a new server](#order-a-new-server).
