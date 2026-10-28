# hopbox Terraform provider

> The hopbox Terraform provider, its provider block, the OVHcloud API rights, the resource hopbox_server, its steps and errors, and the binary.

The network redesign of 2026-10-27 changes this page. hopbox leaves the vRack and the OVHcloud API, and the servers talk over their public addresses (Q90). This page still describes the current code. Milestones N26 to N32 change the code and this page. Before you set up a fleet on the vRack or with an IPv6 /56, read Q90.

`terraform-provider-hopbox` is the hopbox Terraform provider. Terraform runs it as a plugin. It adopts OVHcloud dedicated servers into an install, or orders new ones. It puts each server on the vRack of the install, and reinstalls it with the tested OS. Over SSH, it installs the tested host kernel and `hopboxd`, and enrolls the server. Then it manages the server through the admin API. An install can grow into a fleet: edge servers with floating addresses, more control servers, and nodes in pools. The provider reaches a server that is not an edge server through a jump host. The code is in `internal/tfprovider`. [Install hopbox on an OVHcloud server with Terraform](/guide/install-on-ovhcloud/index.html.md) walks through a first install.

The provider has three resources, [`hopbox_server`](#resource-hopbox_server), [`hopbox_pool`](#resource-hopbox_pool), and [`hopbox_size`](#resource-hopbox_size), and no data sources.

## Provider address

| Item | Value |
| --- | --- |
| Source address | `registry.terraform.io/hopboxdev/hopbox` |
| Short source | `hopboxdev/hopbox` |
| Local name | `hopbox` |
| Plugin protocol | 6 |

The provider is not in a registry yet. The release files hold a zip of the provider for macOS on Apple Silicon and for Linux on x86_64, such as `terraform-provider-hopbox_0.1.0_darwin_arm64.zip`, in the packed layout of a filesystem mirror. Put the zip in `~/.terraform.d/plugins/registry.terraform.io/hopboxdev/hopbox/`, and set `version = "0.1.0"` for the provider in `required_providers`. Terraform reads that dir as a local mirror only without a `provider_installation` block in its CLI configuration. With such a block, add a `filesystem_mirror` block for the dir ([Install hopbox on an OVHcloud server with Terraform](/guide/install-on-ovhcloud/index.html.md#3-write-the-terraform-files)).

To use a local build from a checkout instead, give Terraform the dir of the binary in a `dev_overrides` block of a Terraform CLI configuration file (the build guide of the repo has an example).

## Provider block

```hcl
terraform {
  required_providers {
    hopbox = {
      source = "hopboxdev/hopbox"
    }
  }
}

provider "hopbox" {
  ssh_private_key = file("~/.ssh/id_ed25519")
}
```

The provider block of a fleet on the vRack, after its first server runs:

```hcl
provider "hopbox" {
  ssh_private_key = file("~/.ssh/id_ed25519")
  install_url     = "https://example.com"
  ovh_vrack       = "pn-12345"

  ssh_jump_host = {
    address  = "203.0.113.10"
    host_key = "ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAI..."
  }
}
```

Every attribute is optional. An attribute that you leave out takes the value of its environment variable.

| Attribute | Type | Environment variable | Description |
| --- | --- | --- | --- |
| `ssh_private_key` | string, sensitive | `HOPBOX_SSH_PRIVATE_KEY` | The SSH private key of the operator, in the OpenSSH or PEM format. A key with a passphrase does not work. The provider authorizes its public key on each server that it reinstalls, and logs in with it for every step after the first login. A create, an update, a delete, and an import need it. |
| `install_url` | string | `HOPBOX_INSTALL_URL` | The URL of the install, such as `https://example.com`. The provider calls the admin API there with `admin_api_key`. It must be an `https` URL. Only a loopback address can use `http`. |
| `admin_api_key` | string, sensitive | `HOPBOX_ADMIN_API_KEY` | An admin API key, `hbxadm_...` ([CLI tokens, API keys, admin API keys, and SSH keys](/reference/credentials/index.html.md)). The provider sends it as `Authorization: Bearer <key>`. |
| `control_url` | string | `HOPBOX_CONTROL_URL` | The URL of the internal listener of the control role, such as `https://10.0.0.1:7443`. A new server that is not the first one enrolls there, as `channel.control_urls` ([the bootstrap config of hopboxd](/reference/hopboxd-config/index.html.md#keys)). It must be an `https` URL. Not with `control_urls`. Leave both out to take the control URLs from the install. |
| `control_urls` | list of strings | none | The URLs of the internal listeners of the control servers, such as `["https://172.31.0.11:7443", "https://172.31.0.12:7443"]`. A new server that is not the first one enrolls at the first one that answers, and keeps its control channel to one of them. Each URL must be an `https` URL. Not with `control_url`: both fail with `Two control URL settings` and the detail `Set control_urls or control_url, not both.` |
| `ovh_vrack` | string | `HOPBOX_OVH_VRACK` | The service name of the OVHcloud vRack of the install, such as `pn-12345`. Each new server joins it, and gets its `vrack_address` ([The vRack](#the-vrack)). It must have the form `pn-` and digits. |
| `ssh_jump_host` | object | none | An edge server on the vRack as the jump host of the provider ([Jump host](#jump-host)). Every SSH step of a server that is not an edge server goes through it. The table below lists its attributes. |

The object `ssh_jump_host` has these attributes:

| Attribute | Type | Default | Description |
| --- | --- | --- | --- |
| `address` | string | none, required | The address of the jump host, as a host or `host:port`, such as `203.0.113.10`. The port defaults to 22. |
| `user` | string | none | The login user on the jump host, such as `debian`. Without it, the provider tries `debian`, then `root`. |
| `host_key` | string | none, required | The SSH host key of the jump host, in the `authorized_keys` format. Copy the `ssh_host_key` of the `hopbox_server` of the edge server. The provider never trusts a jump host on its first use. |

The servers that are not the first one need `install_url` and `admin_api_key`. A `hopbox_pool` and a `hopbox_size` need them too. The first server needs none of them: the provider reaches its admin API over SSH. The admin API is public behind the edge role, so `install_url` can be `https://<domain>` ([the admin API](/reference/admin-api/index.html.md#where-the-admin-api-is-served)).

A server that is not the first one enrolls at the control URLs of `control_urls` or `control_url`. Without them, the provider takes the URLs from the install: the `internal_url` of each live control process of `ListControlProcesses` ([the admin API](/reference/admin-api/index.html.md#listcontrolprocesses)). It keeps only a URL whose address is in the vRack range `172.31.0.0/16`, where the other servers reach it. A listener on loopback serves only its own server. On an install with one server, the internal listener listens on `127.0.0.1` ([the bootstrap config of hopboxd](/reference/hopboxd-config/index.html.md#keys)), so the install lists no URL that a new server can reach. On the vRack, the first server listens on its vRack address ([The vRack](#the-vrack)), so the install lists that URL.

The provider reads the OVHcloud credentials only from the environment. They never appear in the Terraform files or in the state.

| Environment variable | Default | Description |
| --- | --- | --- |
| `OVH_ENDPOINT` | `ovh-eu` | The OVHcloud API: `ovh-eu` (`https://eu.api.ovh.com/1.0`), `ovh-ca` (`https://ca.api.ovh.com/1.0`), or `ovh-us` (`https://api.us.ovhcloud.com/1.0`). An `https` URL works too. |
| `OVH_APPLICATION_KEY` | none | The application key of an OVHcloud API application. |
| `OVH_APPLICATION_SECRET` | none | The application secret. |
| `OVH_CONSUMER_KEY` | none | The consumer key, with the rights below. |

Give all three keys or none. Without them, the provider can plan, but a create, a refresh, and an import fail with `the provider has no OVHcloud credentials`.

### OVHcloud API rights

The consumer key needs these rights. The provider makes no other call.

| Method | Path | Use |
| --- | --- | --- |
| `GET` | `/dedicated/server/*` | Reads the server, its hardware, its IP blocks, and the task of a reinstall: `/dedicated/server/{serviceName}`, `/dedicated/server/{serviceName}/specifications/hardware`, `/dedicated/server/{serviceName}/ips`, and `/dedicated/server/{serviceName}/task/{taskId}`. |
| `POST` | `/dedicated/server/*/reinstall` | Reinstalls the server. |

To order servers with `ovh_plan`, the consumer key also needs these rights. A provider that only adopts servers does not need them.

| Method | Path | Use |
| --- | --- | --- |
| `GET` | `/me` | Reads the OVHcloud subsidiary of the account, such as `FR`, for the cart. |
| `GET` | `/order/cart` | Finds the carts of an order key: `/order/cart?description=hopbox_server <key>`. |
| `POST` | `/order/cart` | Makes a cart. |
| `GET` | `/order/cart/*` | Reads a cart, the options of a plan, the config that an item needs, and the price: `/order/cart/{cartId}`, `/order/cart/{cartId}/baremetalServers/options`, `/order/cart/{cartId}/item/{itemId}/requiredConfiguration`, and `/order/cart/{cartId}/checkout`. |
| `POST` | `/order/cart/*` | Fills the cart and orders: `/order/cart/{cartId}/assign`, `/order/cart/{cartId}/baremetalServers`, `/order/cart/{cartId}/baremetalServers/options`, `/order/cart/{cartId}/item/{itemId}/configuration`, and `/order/cart/{cartId}/checkout`. The checkout pays for the server. |
| `DELETE` | `/order/cart/*` | Deletes a cart without a checkout, and the cart of an order once the state holds the order. |
| `GET` | `/me/order/*` | Follows an order: `/me/order/{orderId}/status`, `/me/order/{orderId}/details`, `/me/order/{orderId}/details/{orderDetailId}`, and `/me/order/{orderId}/details/{orderDetailId}/extension`. |

The provider never calls a path that cancels an order or ends the rental of a server.

With `ovh_vrack`, the consumer key also needs these rights. A provider without `ovh_vrack` does not need them.

| Method | Path | Use |
| --- | --- | --- |
| `GET` | `/dedicated/server/*` | Also reads the network interfaces of the server, with the MACs of their cards and their vRack: `/dedicated/server/{serviceName}/virtualNetworkInterface` and `/dedicated/server/{serviceName}/virtualNetworkInterface/{uuid}`. The first table holds this right already. |
| `POST` | `/vrack/*/dedicatedServerInterface` | Puts the vRack interface of a new server into the vRack. |
| `DELETE` | `/vrack/*/dedicatedServerInterface/*` | Takes the server out of the vRack at its destroy. |
| `GET` | `/vrack/*/task/*` | Follows the task of an attach or a detach. |

With `floating_addresses`, the consumer key also needs these rights:

| Method | Path | Use |
| --- | --- | --- |
| `GET` | `/ip/*` | Reads an Additional IP and the server that it routes to: `/ip/{ip}`, such as `/ip/203.0.113.5%2F32`. |
| `POST` | `/ip/*/move` | Routes an Additional IP to an edge server. |
| `POST` | `/ip/*/park` | Routes an Additional IP of an edge server to nothing at its destroy. |
| `GET` | `/ip/*/task/*` | Follows the task of a move or a park. |

The provider never turns on the network firewall of OVHcloud (`/ip/{ip}/firewall`). That firewall is stateless, filters only IPv4, and holds at most 20 rules for each address.

The provider also calls `GET /auth/time`, which needs no right, to sign each call with the clock of the API. Each call carries the headers `X-Ovh-Application`, `X-Ovh-Consumer`, `X-Ovh-Timestamp`, and `X-Ovh-Signature`. The signature is `$1$` and a SHA-1. The SHA-1 covers six values, joined by `+`: the application secret, the consumer key, the method, the full URL, the body, and the timestamp.

## Resource hopbox_server

A `hopbox_server` is one server of an install, on an OVHcloud dedicated server that the account rents. In the example, the paths name the release files in the folder of the configuration. A build from a checkout uses `bin/bundle/hopbox-v0.1.0-linux-amd64.tar.gz` and `hostkernel-build/linux-image-6.18.57-hopbox1_6.18.57-1_amd64.deb` instead.

```hcl
resource "hopbox_server" "first" {
  ovh_service_name = "ns1234567.ip-198-51-100.eu"
  bootstrap        = true
  domain           = "example.com"
  labels           = { zone = "gra" }

  bundle = {
    source = "hopbox-v0.1.0-linux-amd64.tar.gz"
    sha256 = "3f6a...c1d2"
  }
  host_kernel = {
    source = "linux-image-6.18.57-hopbox1_6.18.57-1_amd64.deb"
    sha256 = "9b1e...77aa"
  }
}

output "admin_api_key" {
  value     = hopbox_server.first.admin_api_key
  sensitive = true
}
```

The same server, ordered new from OVHcloud in place of a server that the account rents:

```hcl
resource "hopbox_server" "first" {
  ovh_plan       = "24rise01"
  ovh_datacenter = "gra"
  ovh_options    = ["ram-64g-ecc-3200-24rise", "softraid-2x512nvme-24rise", "bandwidth-500-24rise"]
  ovh_order_key  = "hop-example-first"
  bootstrap      = true
  domain         = "example.com"
  labels         = { zone = "gra" }

  bundle = {
    source = "hopbox-v0.1.0-linux-amd64.tar.gz"
    sha256 = "3f6a...c1d2"
  }
  host_kernel = {
    source = "linux-image-6.18.57-hopbox1_6.18.57-1_amd64.deb"
    sha256 = "9b1e...77aa"
  }

  lifecycle {
    prevent_destroy = true
  }
}
```

Servers of a fleet on the vRack, beside the first server. The edge server holds a floating address, the control server shares the control store, and the node joins the pool `big`:

```hcl
resource "hopbox_server" "edge2" {
  ovh_service_name   = "ns2345678.ip-198-51-100.eu"
  roles              = ["edge", "dns"]
  vrack_address      = "172.31.0.12"
  floating_addresses = ["203.0.113.6"]

  bundle      = local.bundle
  host_kernel = local.host_kernel

  lifecycle {
    create_before_destroy = true
  }
}

resource "hopbox_server" "cp2" {
  ovh_service_name = "ns3456789.ip-198-51-100.eu"
  roles            = ["control", "node"]
  vrack_address    = "172.31.0.13"

  copy_control_from = {
    address  = hopbox_server.first.vrack_address
    host_key = hopbox_server.first.ssh_host_key
    user     = hopbox_server.first.ssh_user
  }

  bundle      = local.bundle
  host_kernel = local.host_kernel
}

resource "hopbox_server" "node1" {
  ovh_service_name = "ns4567890.ip-198-51-100.eu"
  pool             = hopbox_pool.big.name
  vrack_address    = "172.31.0.21"

  bundle      = local.bundle
  host_kernel = local.host_kernel
}
```

The plan code and the option codes in this example are placeholders. Take the real ones from the OVHcloud manager or from the catalog of the OVHcloud API, as [Order](#order) shows. `prevent_destroy` makes Terraform refuse each plan that destroys or replaces the server. A replacement orders a new server ([Rules that keep a plan from ordering](#rules-that-keep-a-plan-from-ordering)). Remove it before a destroy on purpose.

### Arguments

| Attribute | Type | Default | Change | Description |
| --- | --- | --- | --- | --- |
| `ovh_service_name` | string | none | replaces | The service name of an OVHcloud dedicated server that the account rents, such as `ns1234567.ip-198-51-100.eu`. The create erases the server. Give `ovh_service_name` or `ovh_plan`, not both. With `ovh_plan`, the provider sets it at the delivery of the server. |
| `ovh_plan` | string | none | refused | The OVHcloud plan code of a new dedicated server to order, such as `24rise01`. The create orders the server, and OVHcloud charges the account for it ([Order](#order)). It needs `ovh_datacenter`, `ovh_options`, and `ovh_order_key`. |
| `ovh_datacenter` | string | none | refused | The datacenter of the server to order, such as `gra`. [Datacenters](#datacenters) lists the datacenters that the provider knows. Only with `ovh_plan`. |
| `ovh_options` | set of string | none | refused | The option plan codes of the server to order. Give one option for each mandatory family of the plan, such as the memory, the storage, and the bandwidth. An optional family, such as the vRack, takes one option or none. A family that is not exclusive takes more than one option. Pick a software RAID storage option, whose code starts with `softraid-`. The provider refuses a code that starts with `hardraid-`, because hopbox tests only software RAID. Only with `ovh_plan`. |
| `ovh_order_key` | string | none | refused | A key of the order that is unique in the OVHcloud account, such as `hop-example-first`. It has 1 to 63 characters. Each character is one of `a-z`, `0-9`, `.`, `_`, `/`, and `-`. The first one is a letter or a digit. The provider marks the cart of the order with it, so a later apply finds an order that an earlier apply lost ([Lost orders](#lost-orders)). Only with `ovh_plan`. |
| `ovh_max_price` | number | none | in place, changes nothing | The highest price with tax of the cart, in the currency of the OVHcloud account, such as `90`. The provider compares it with the price with tax of the checkout preview. That price is the first invoice: the first month and any setup fee, not the monthly rent. If the checkout costs more, the create stops before the checkout, and orders nothing. A preview without a price with tax or without a currency also stops the create. Without `ovh_max_price`, the create pays any price. Only with `ovh_plan`. |
| `ovh_order_id` | number | none | refused | The ID of an OVHcloud order that exists already. The create then places no order. It waits for that order and sets up its server. Use it for an order that an apply lost. Only with `ovh_plan`. |
| `bootstrap` | bool | `false` | replaces | True for the first server of a new install. It gets the roles `control`, `node`, `dns`, and `edge`. It needs `domain`, and the provider makes the first admin API key for it. |
| `domain` | string | none | in place | The domain of the install, such as `example.com` ([Domain rules](/reference/domain-rules/index.html.md#install-domain)). Only the first server takes it, and it needs it. A change runs `hopboxd admin config set --domain` on the server. |
| `pool` | string | `default` | replaces | The pool that the server joins: `default`, or the name of a pool, such as `hopbox_pool.big.name` ([Resource hopbox_pool](#resource-hopbox_pool)). It must be a lowercase DNS label. The first server joins `default`. A server whose hardware differs from the first server of its pool fails to enroll ([Enrollment and the install PKI](/reference/enrollment/index.html.md#hardware-of-a-pool)). |
| `labels` | map of string | `{}` | in place | The labels of the server, such as `zone = "gra"`. Placement filters use them. The rules of `channel.labels` apply ([the bootstrap config of hopboxd](/reference/hopboxd-config/index.html.md#keys)). A server has at most 64 labels. A key has 1 to 63 characters of `a-z`, `0-9`, `.`, `_`, and `-`, and starts with a letter. A value has at most 255 bytes. The provider sets them with `SetServerLabels`, so the install ignores the labels of the bootstrap config of the server ([the admin API](/reference/admin-api/index.html.md#setserverlabels)). |
| `roles` | list of string | `["control", "node", "dns", "edge"]` with `bootstrap`, else `["node"]` | updates in place | The roles of the server, in any order. The first server has every role, so leave it out there. Another server has one or more of `control`, `node`, `dns`, and `edge`, each once. `dns` needs `edge`, because the DNS role runs on the edge servers. In this release, `control` needs another role too, such as `["control", "node"]`, so that the server enrolls and the registry knows it ([Control servers](#control-servers)). A refresh reads the roles that the server has: `control` from its bootstrap config or the state, and the others from the registry of the install. |
| `copy_control_from` | object | none | in place, changes nothing | The live control server whose keys and external Postgres a new control server copies before its install, with its pinned host key. A control server that is not the first one needs it, and no other server takes it. Only the create reads it. See [Control servers](#control-servers). |
| `vrack_address` | string | none | refused | The address of the server on the vRack of `ovh_vrack`, such as `172.31.0.11`. It must be in `172.31.0.0/16`, and not `172.31.0.0` or `172.31.255.255`. Give each server its own address: a second resource with the same address fails the plan with `two hopbox_server resources have the vRack address <address>`. With `ovh_vrack`, every server needs it. Without `ovh_vrack`, no server takes it. The plan refuses a change in place with `The vRack address of a server cannot change`. A replacement of the server, such as with another `ovh_service_name`, takes a new address. See [The vRack](#the-vrack). |
| `floating_addresses` | set of string | none | in place | The Additional IPv4 addresses of an edge server, such as `["203.0.113.5"]`. Each one is one public IPv4 address that the OVHcloud account has, routed to a server, not to a vRack. Only a server with the role `edge` takes them. A change moves the new addresses to the server, and removes the old ones from its public interface. See [Edge servers and floating addresses](#edge-servers-and-floating-addresses). |
| `public_ipv6` | string | none | refused | The public IPv6 address of an edge server in the IPv6 block of the install, with the prefix length of the block, such as `2001:db8:0:100::12/56`. The block lives in the vRack, so `hopboxd` holds the address on the vRack interface of the server ([the edge role](/reference/edge/index.html.md#the-public-ipv6-address)). The provider writes it to `edge.public_ipv6` of the bootstrap config, and adds port 53 of the address to `dns.listen` on a server with the DNS role ([Create](#create)). It needs `public_ipv6_gateway`, and only a server with the role `edge` takes it. The rules of `edge.public_ipv6` apply ([the bootstrap config of hopboxd](/reference/hopboxd-config/index.html.md#keys)). Give each edge server its own address: a second resource with the same address fails the plan with `two hopbox_server resources have the public IPv6 address <address>`. The plan refuses a change in place with `The public IPv6 address of a server cannot change`. A replacement with the same value takes the same address. See [Edge servers and floating addresses](#edge-servers-and-floating-addresses). |
| `public_ipv6_gateway` | string | none | refused | The gateway of the IPv6 block on the vRack, such as `2001:db8:0:100::1`, for `edge.public_ipv6_gateway`. It goes with `public_ipv6`. |
| `bundle` | object | none, required | in place | The release bundle that the server runs ([Release bundle](/reference/release-bundle/index.html.md)), with `source` and `sha256`. A change of `sha256` upgrades `hopboxd` on the server. A change of `source` alone changes nothing on the server. |
| `bundle.source` | string | none, required | | A local path, or an `https` URL that the server downloads with `curl`. The file name must be `hopbox-<version>-linux-amd64.tar.gz`. |
| `bundle.sha256` | string | none, required | | The sha256 of the bundle, in 64 lowercase hex digits, from the `.sha256` file of `make bundle`. |
| `host_kernel` | object | none, required | in place | The Debian package of the tested host kernel, `linux-image-6.18.57-hopbox1_6.18.57-1_amd64.deb` from `build/hostkernel/build.sh` ([Host kernel](/reference/host-kernel/index.html.md)), with `source` and `sha256` as for `bundle`. The provider installs it only on a server that does not run the tested kernel. |

### Attributes the provider sets

| Attribute | Type | Description |
| --- | --- | --- |
| `id` | string | The OVHcloud service name. Before OVHcloud delivers an ordered server, `order-<order ID>`. |
| `ovh_service_name` | string | With `ovh_plan`: the service name of the server that the order delivered. |
| `ovh_order_id` | number | With `ovh_plan`: the ID of the OVHcloud order of the server. Without it: null. |
| `server_id` | string | The ID of the server in the install, a UUID. |
| `admin_api_key` | string, sensitive | Only on the first server: the first admin API key of the install, `hbxadm_` and 43 characters. The provider makes it and passes only its SHA-256 to the install ([Server layout](/reference/server-layout/index.html.md#the-bootstrap-admin-api-key)). The state holds its only copy. The install registers it with the name `bootstrap` and a lifetime of 90 days. |
| `version` | string | The version of `hopboxd` on the server, such as `v0.1.0`. |
| `ipv4` | string | The public IPv4 address of the server, from the OVHcloud API. |
| `ssh_host_key` | string | The SSH host key of the server, in the `authorized_keys` format. The provider pins it at the first login after the reinstall, and refuses another key from then on. |
| `ssh_user` | string | The login user of the provider: `debian` on the OVHcloud template of Debian 13, or `root`. When it is not `root`, each script runs with `sudo -n`. |

### The tested OS and kernel

The OS and the kernel are constants of the provider. You cannot change them. hopbox tests one combination.

| Constant | Value |
| --- | --- |
| OVHcloud template | `debian13_64` |
| OS | Debian 13 on `amd64` |
| Host kernel release | `6.18.57-hopbox1` |
| Host kernel package | `linux-image-6.18.57-hopbox1`, version `6.18.57-1`, for `amd64` |

### Create

With `ovh_plan`, a create first orders the server ([Order](#order)). Then it takes the same steps as for a server that the account rents.

A create takes these steps:

1. It reads each local file of `bundle` and `host_kernel`, and checks its sha256. A missing file or another sha256 stops the create before the reinstall, so the server is not erased. Then it reads the server, its IP blocks, and its hardware through the OVHcloud API.
2. It makes the checks that need the install or the OVHcloud API. Each one stops the create before the reinstall:
   - With `ovh_vrack`, the server needs `vrack_address`. Without `ovh_vrack`, it takes none.
   - No other resource has a floating address or the vRack address of this server. The plan checks this first. Each floating address is an Additional IP of the account that routes to no vRack ([Edge servers and floating addresses](#edge-servers-and-floating-addresses)).
   - A server that is not the first one needs the control URLs, from the provider block or from the install ([Provider block](#provider-block)).
   - A control server that is not the first one needs the vRack and `copy_control_from`. The provider reads the files of that live control server, and checks the fingerprints of its keys ([Control servers](#control-servers)).
3. With `ovh_vrack`, it puts the vRack interface of the server into the vRack, and waits for the task ([The vRack](#the-vrack)).
4. It makes an ephemeral ed25519 SSH key. It reinstalls the server with the template `debian13_64` and the public key of the ephemeral key. The default disk group gets this layout:

   | Mount point | File system | Size | RAID |
   | --- | --- | --- | --- |
   | `/` | ext4 | 100 GiB | RAID1 over 2 disks, or none with 1 disk |
   | `/var/lib/hopbox` | XFS | the rest of the disks | the same |

   A disk group behind a hardware RAID controller is refused. A server with more than two disks uses two of them.
5. It polls the task of the reinstall every 30 seconds, for at most 2 hours. A task that ends as `cancelled`, `customerError`, or `ovhError` fails the create.
6. It logs in with the ephemeral key as `debian`, then as `root`, and retries every 10 seconds for at most 30 minutes. A server that is not an edge server gets this login through the jump host, at its public address ([Jump host](#jump-host)). It pins the host key of this first login in `ssh_host_key`. See [Host key trust](#host-key-trust).
7. It adds the public key of `ssh_private_key` to the `authorized_keys` of the login user, and logs in again with that key. Then it removes the ephemeral key from the `authorized_keys` of root and of the login user. The ephemeral key lives only in the memory of the provider.
8. With `vrack_address` or `floating_addresses`, it sets up the network of the server ([The vRack](#the-vrack)). It checks that the server holds its vRack address. Through the jump host, it logs in again at the vRack address, and uses that address for every later step.
9. It checks that the server runs Debian 13 on `amd64`.
10. It checks that `/var/lib/hopbox` is a file system of its own, on XFS with reflinks. If the XFS has no reflinks and holds no file, the provider makes the file system again. It runs `mkfs.xfs -m reflink=1` with the same UUID, and the apply shows a warning. A file system with files is never made again.
11. If the server does not run `6.18.57-hopbox1`, it installs the host kernel. It puts the package on the server ([Uploads](#uploads)), checks its sha256, and checks that the package is `linux-image-6.18.57-hopbox1` version `6.18.57-1` for `amd64`. It runs `dpkg -i`, pins the GRUB default to the new kernel in `/etc/default/grub.d/20_hopbox_kernel.cfg`, and checks `/boot/grub/grub.cfg`, as `build/hostkernel/install.sh` does. Then it reboots the server from a systemd timer. It logs in again with the pinned host key until the boot ID changes, for at most 20 minutes. It checks that the new boot runs the tested kernel.
12. On the first server, and on a server with the DNS role, it finds the public addresses of the server. It reads the global addresses of the network interfaces of the server with `ip -j addr show scope global`. A public address is the address of a `/32` or `/128` IP block of step 1 that a network interface holds. An IPv6 address that is `tentative` or `dadfailed` does not count, because it does not answer yet. The list puts IPv4 first. A block that no interface holds stays out, such as an IPv6 block without a route. So DNS never points at an address that answers nothing. If no block passes, the create fails.
13. On a control server that is not the first one, it writes the key files and the files of the external Postgres that step 2 read ([Control servers](#control-servers)).
14. It writes the bootstrap config `/etc/hopbox/hopboxd.toml` before the install, on a server with the DNS role, an edge server with `public_ipv6` or `floating_addresses`, a control server on the vRack, or a control server that is not the first one. `hopboxd install` keeps its keys ([the bootstrap config of hopboxd](/reference/hopboxd-config/index.html.md#keys)). The file holds the roles and these keys:

    | Key | Server | Value |
    | --- | --- | --- |
    | `dns.listen` | a server with the DNS role | Port 53 of each public address of step 12, of each address of `floating_addresses`, and of the address of `public_ipv6`. The zone names the floating addresses as `ns1` and `ns2`, but they route to the server only at the end of the create. The tested OS runs `systemd-resolved`, which holds port 53 on `127.0.0.53` and `127.0.0.54`. So the default `dns.listen` of every address fails to bind ([the DNS role](/reference/dns/index.html.md)). The server holds the address of `public_ipv6` only once `hopboxd` runs, and the DNS role binds it before ([the DNS role](/reference/dns/index.html.md#listeners)). |
    | `edge.public_ipv6`, `edge.public_ipv6_gateway` | an edge server with `public_ipv6` | The values of `public_ipv6` and `public_ipv6_gateway`. |
    | `edge.public_ipv4` | an edge server with `floating_addresses`, on a release with the egress tunnels | The first address of `floating_addresses` in sorted order. The egress of the machines leaves from it ([the egress of the machines](/reference/egress/index.html.md#the-source-addresses)). The provider reads the features of the `hopboxd` of the bundle first, and writes the key only when they name `egress` ([Releases with and without the key](#releases-with-and-without-the-key)). |
    | `control.internal_listen` | a control server with `vrack_address` | Port 7443 of the vRack address, such as `172.31.0.11:7443` ([the internal listener](/reference/internal-listener/index.html.md)). |
    | `channel.control_urls` | the first server with `vrack_address` | The internal listener of the server itself, such as `["https://172.31.0.11:7443"]`, because its listener no longer listens on loopback. |
    | `postgres.url`, `postgres.ca_file`, `postgres.password_file` | a control server that is not the first one | The `postgres.url` of the live control server of step 2, `/etc/hopbox/postgres-ca.pem`, and `/etc/hopbox/postgres-password`. |

    For example, OVHcloud reports `203.0.113.10/32` and `2001:db8::10/128` for the first server, a network interface of the server holds both, and `vrack_address` is `172.31.0.11`. Then the file is:

    ```toml
    # Written by the hopbox Terraform provider before the first hopboxd install.
    roles = ["control", "node", "dns", "edge"]

    [control]
    internal_listen = "172.31.0.11:7443"

    [channel]
    control_urls = ["https://172.31.0.11:7443"]

    [dns]
    listen = ["203.0.113.10:53", "[2001:db8::10]:53"]
    ```

15. On the first server, it makes the admin API key and saves it in the state. On another server, it makes an enrollment token through the admin API, with an `Idempotency-Key` header. The token is for the pool of the server and its roles that enroll: every role but `control`.
16. It puts the bundle in `/var/lib/hopbox-provider` and checks its sha256 there ([Uploads](#uploads)). It unpacks the bundle and runs its `hopboxd install --json` with these flags ([hopboxd](/reference/hopboxd/index.html.md#install)):

    | Server | Flags |
    | --- | --- |
    | First server | `--roles control,node,dns,edge --domain <domain> --bootstrap-admin-key-hash <sha256 of the key>` |
    | Other server | `--roles <roles>`, one `--control-url <url>` for each control URL of step 2, and the enrollment token in `HOPBOX_ENROLLMENT_TOKEN` |

    The install runs in a transient systemd unit, under `systemd-run --wait`, so a dropped SSH session does not stop it. The token reaches the unit through a file of mode 0600 in `/var/lib/hopbox-provider`, never through a command line. The provider removes the bundle, the unpacked release, and the token file after the install.
17. On the first server, it makes a token with `hopboxd admin enrollment-token create --pool <pool> --roles node,dns,edge`. It gives the token to `hopboxd install --roles control,node,dns,edge --json`. So the node role, the DNS role, and the edge role enroll with the control role of the same server ([the DNS role](/reference/dns/index.html.md), [the edge role](/reference/edge/index.html.md)). The token never leaves the server.
18. It waits until the server is `ready`, for at most 5 minutes. On the first server, it reads `hopboxd admin server get --json` over SSH. On another server, it calls `GetServer` of the admin API. A server that is ready but `degraded` gives a warning with its problems.
19. It sets the labels with `SetServerLabels`, through `hopboxd admin server set-labels` on the first server.
20. On the first server, it sets the public addresses of the install to the public addresses of step 12, with `hopboxd admin config set --public-address`. The zone of the DNS role then points the domain, `ns1`, `ns2`, and every machine host at the server ([the DNS role](/reference/dns/index.html.md#the-zone-of-the-install)). The install runs at this step. So a failure gives the warning `The provider did not set the public addresses of the install`, with the command to run on the server, and the create succeeds. A failed create taints the server, and the next apply then erases the install.
21. It routes each floating address to the server ([Edge servers and floating addresses](#edge-servers-and-floating-addresses)). This step comes last: the old server of a replaced edge serves the addresses until the new server is ready.

The state keeps what each step learned. If a step fails, Terraform marks the resource as tainted, and the next apply destroys it and creates it again, with a new reinstall. A failed preflight check of severity `warn`, such as a kernel that is not the tested one, gives a warning on the apply.

### The vRack

The vRack is the private network of OVHcloud between the servers of an install. The control channel and the internal listener run over it. With `ovh_vrack`, each new server joins the vRack, and holds its `vrack_address` in `172.31.0.0/16` on its private interface.

OVHcloud gives each server a public and a vRack network interface, each with the MACs of its network cards. The create takes these steps for the vRack:

1. It reads the network interfaces of the server. A server whose interfaces bond their cards (`public_aggregation` or `vrack_aggregation`) fails, because the provider does not set up link aggregation.
2. If the vRack interface is in another vRack, the create fails: the server belongs to something else. If it is in the vRack of the provider block, the create goes on.
3. It attaches the interface to the vRack with `POST /vrack/{serviceName}/dedicatedServerInterface`, before the reinstall. It polls the task every 30 seconds, for at most 2 hours. The API forgets a vRack task once it is done, so a task that is gone counts as done. A `cancelled` task fails the create. Then it reads the interface again, and checks that it is in the vRack of `ovh_vrack`.
4. After the key swap, it writes the boot script `/usr/local/sbin/hopbox-network` and the unit `hopbox-network.service`, and runs the unit. The script finds the interface whose card has the MAC of the vRack interface, brings it up, and gives it the vRack address as a `/16`. It looks only at interfaces with a device, so a bridge or a VLAN that shows the MAC of its card never matches. It finds the public interface by its MAC too, and gives it each floating address as a `/32`. It waits up to 60 seconds for each card, because the kernel can name the cards late. The unit runs at each boot, after `systemd-networkd.service` and `networking.service`, and before `network-online.target` and `hopboxd.service`. So the network setup of the OS cannot remove the addresses after the unit.
5. It checks that a network interface holds the vRack address, and that the OVHcloud API shows the interface in the vRack.

On a control server, the internal listener listens on port 7443 of the vRack address. The first server lists that listener in its `channel.control_urls`, so its own roles enroll there. The other servers take the control URLs from the install, which lists the listener of each live control process ([Provider block](#provider-block)).

A destroy takes the server out of the vRack with `DELETE /vrack/{serviceName}/dedicatedServerInterface/{uuid}`, after the decommission. A server outside the install then keeps no way into its private network. If that fails, the destroy succeeds with a warning, such as `The server <name> stays in the vRack <vrack>: ... Take it out in the OVHcloud manager.`

The vRack address cannot change in place. Other servers reach a control server at that address, and the provider reaches the server there through the jump host. To give a server another address, replace it.

### Jump host

The [host firewall](/reference/host-firewall/index.html.md) keeps SSH open on the internet only on the edge servers (rule 5). A server without the role `edge` closes its public address once `hopboxd` runs on it. Port 22 of its vRack address then takes only the vRack addresses of the edge servers. So with `ssh_jump_host`, every SSH step of a server without the role `edge` goes through the jump host, as `ssh -J` does. The first server and the other edge servers get their SSH straight.

The jump host must be an edge server on the vRack of the install. Through it, the provider dials the vRack address of the server, once the server holds it. So SSH does not depend on the mesh. Before that, the provider dials the public address of the server through the jump host: the first login after the reinstall, and the key swap. `hopboxd` runs on the server only after the network setup, so its firewall is not in place yet. This holds for the reboot into the host kernel, an upgrade, an update, a refresh, and a destroy too. An import dials the vRack address of its import ID ([Import](#import)).

The session to the server runs inside the connection that the jump host opens, so the jump host sees only encrypted bytes. The pin of the host key of the server holds through the jump host. The jump host has its own pin, `ssh_jump_host.host_key`. A jump host that shows another key fails each step with `remote: the host key of the jump host is not the pinned one`, and the provider never takes that for a reinstalled server. The provider logs in to the jump host with `ssh_private_key`, so that key must be authorized on the jump host. The provider authorizes it on each server that it sets up, the edge servers included. sshd of the jump host must allow TCP forwarding, as the default of Debian 13 does.

Without `ssh_jump_host`, the provider dials each server at its public address. So a create or an update of a server with `vrack_address` and without the role `edge` fails before any step without `ssh_jump_host`: `the server <name> has a vrack_address and no role edge, so its host firewall closes its public address once hopboxd runs, ...`. Set `ssh_jump_host` before you upgrade such a server to a release with the host firewall.

### Edge servers and floating addresses

An edge server takes the public traffic of the install. The first server has the role `edge`. Another server gets it with `roles = ["edge"]`, or `["edge", "dns"]` to answer DNS too.

The floating addresses of an edge server are Additional IPs of the OVHcloud account, which OVHcloud routes to one server. When Terraform replaces an edge server, the new server takes over the same addresses, so the public addresses of the install never change. The provider routes each address with these steps:

1. Before the reinstall, it reads each address with `GET /ip/{ip}`. An address that the account does not have fails the create. So does an address that routes to a vRack, because such a block is not linked to one server. The provider takes an address only when it routes to nothing, to the server itself, or to a server in the vRack of `ovh_vrack`. So an address of another install in the same OVHcloud account, such as a staging install, fails the create with `the Additional IP <address> routes to the server <name>, which is not in the vRack <vrack> of the install, ...`.
2. The network setup of [The vRack](#the-vrack) gives each address to the public interface of the server, as a `/32`.
3. As the last step of the create, it moves each address that routes to another server with `POST /ip/{ip}/move`. It checks the holder of step 1 again first. It polls the task every 30 seconds, for at most 2 hours. A task that ends as `cancelled`, `customerError`, or `ovhError` fails the create.

The old server keeps the addresses until the new server is ready. To replace an edge server without a gap, give its resource `lifecycle { create_before_destroy = true }`, and give the new server another `ovh_service_name` and another `vrack_address`. Terraform then creates the new server, which takes the addresses, before it destroys the old one. Without it, Terraform destroys the old server first, and the addresses answer nothing until the new server is ready.

A destroy of an edge server parks each of its floating addresses that still routes to it, with `POST /ip/{ip}/park`, before the server leaves the vRack. The address then routes to nothing, so the next edge server can take it. The destroy parks no address that another resource of the same configuration claims, because that resource takes it in the same apply. A park that fails gives a warning, and the destroy succeeds.

An update of `floating_addresses` gives the server its new list, and moves the new addresses to the server. An address that the list drops stays routed to the server at OVHcloud, until another edge server takes it. Another edge server takes an address only at the end of its create, and the update of this server can come first. So while OVHcloud routes a dropped address to this server, the server keeps the address on its public interface and in `dns.listen`, and the address keeps answering. The provider keeps such an address in the private state of the resource. The next update of the resource removes it from the server once it routes elsewhere, and the destroy parks it. A refresh never removes it, because a refresh never changes a server. The update removes a dropped address that routes elsewhere at once. An address that you drop and give to no other server keeps answering on this server, because OVHcloud still routes it there. To stop that, park the address or move it to another server in the OVHcloud manager. The next update of the resource then removes it from the server. On a server with the DNS role, the update also gives `dns.listen` the new list: port 53 of each public address that the server holds, of each floating address, of each dropped address that the server keeps, and of the address of `public_ipv6`. It runs `hopboxd install --dns-listen` with the roles of the server ([hopboxd](/reference/hopboxd/index.html.md#install)), before it changes the network. When the list changes, `hopboxd` restarts, and the update waits until the server is ready. A `hopboxd` of an earlier release refuses `--dns-listen`. The update then fails before the network changes, and the error says to upgrade the server with a bundle of this release first.

The egress of the machines leaves an edge server from one floating address, `edge.public_ipv4` of the bootstrap config ([the egress of the machines](/reference/egress/index.html.md#the-source-addresses)). The provider picks the first address of `floating_addresses` in sorted order, as the owner decided on 2026-10-27 (plan question Q82). The create writes it to the bootstrap config. An update of `floating_addresses` can change that address. Then the update sets the key as its last step, after the server holds the address and OVHcloud routes it there. It runs `hopboxd install --edge-public-ipv4` with the roles of the server ([hopboxd](/reference/hopboxd/index.html.md#install)). When the key changes, `hopboxd` restarts, and the update waits until the server is ready. An update that drops every floating address keeps the key. A `hopboxd` of an earlier release refuses `--edge-public-ipv4`, and the error says to upgrade the server with a bundle of this release first. An update of `floating_addresses` on a release without the feature `egress` sets no key.

### Releases with and without the key

A release from before the egress tunnels refuses `edge.public_ipv4` in the bootstrap config, with `unknown keys`. So the provider writes the key only for a release whose `hopboxd version --json` names the feature `egress` ([the version command](/reference/version/index.html.md#json-output)). It reads the features of the `hopboxd` of a bundle on the server before the install. The key follows the release of each edge server with `floating_addresses`:

| Step | What the provider does with the key |
| --- | --- |
| Create | It writes the key only for a bundle with the feature. |
| Upgrade to a release with the feature | After the upgrade, it sets the key with `hopboxd install --edge-public-ipv4`. An edge server made before the egress tunnels gets its key this way. When the key changes, `hopboxd` restarts, and the update waits until the server is ready. |
| Move to a release without the feature | Before the upgrade, it removes the key from the bootstrap config, so the older release starts. The edge server then masquerades the egress to the address of its output interface. If the upgrade fails, the provider puts the key back for the release that keeps running. When that restarts `hopboxd`, the apply waits until the server is ready again, as after the other restarts, and then fails with the error of the upgrade. After the upgrade, it turns off the IPv6 forwarding that the egress turned on, because the older release keeps no rule that guards it. It does so only when `<data_dir>/hostfw/forward-guard-ipv6` records that `hopboxd` turned it on, and then removes the file. On a server without the role `node`, it also turns off the IPv4 forwarding, only when `<data_dir>/hostfw/forward-guard` exists, and removes that file. So forwarding that something else turned on before `hopboxd`, such as Docker, stays on. An edge server that ran a build of `main` with the egress from before this record has no `forward-guard-ipv6`, so its IPv6 forwarding stays on after the move back. On such a server, run `sudo sysctl -w net.ipv6.conf.all.forwarding=0` by hand, unless another tool on the server needs it. A server with the role `node` keeps IPv4 forwarding, because its node network needs it for its machines. If this step fails, the apply succeeds with a warning that names the files to look for and the `sysctl` commands to run on the server. |

A refresh reads each floating address. An address that routes to another server leaves the state, so the next plan shows an update, which moves it back.

Two resources that list the same address move it back and forth at each apply. So the plan of each resource claims its floating addresses. Terraform plans every resource of the configuration, also a resource that does not change. A second resource with the same address fails the plan with `two hopbox_server resources have the floating address <address>`, before any apply.

Each edge server can also hold one address of the IPv6 block of the install, which OVHcloud routes into the vRack, not to one server. Give it `public_ipv6` and `public_ipv6_gateway`. The provider writes them to the bootstrap config of the server at its create, and `hopboxd` holds the address on the vRack interface ([the edge role](/reference/edge/index.html.md#the-public-ipv6-address)). A replacement of the edge server with the same values takes the same address. While the old server still holds it, duplicate address detection keeps the new server from it, and the new server takes it once the old one is gone. So the AAAA records of the zone never change. Each resource claims its address in the plan, as for the floating addresses.

The zone of the DNS role names the public addresses of the install, which an instance admin sets with `hopboxd admin config set --public-address` ([the DNS role](/reference/dns/index.html.md#the-zone-of-the-install)). The provider sets them only on the first server, at its create, to the public addresses of that server. To point the zone at the edge servers, set them to the floating addresses and the addresses of `public_ipv6` of the edge servers ([Install hopbox on an OVHcloud server with Terraform](/guide/install-on-ovhcloud/index.html.md#6-point-the-zone-at-the-edge-servers)).

### Control servers

The first server runs the control role. An install can run it on more servers, with an external Postgres that they share ([Run the control store on an external Postgres](/guide/external-postgres/index.html.md)). A control server that is not the first one has `roles = ["control", "node"]`, or `control` with another role. It needs the vRack, because the other servers reach its internal listener at its vRack address.

Every control server must hold the same key-encryption keys ([Sealing](/reference/sealing/index.html.md#several-control-servers)). A control server whose first key came from `hopboxd install` seals values that no other control server can open. So the provider copies the files of a live control server to the new server before its first install. `copy_control_from` names that server, with its pinned host key:

```hcl
  copy_control_from = {
    address  = hopbox_server.first.vrack_address
    host_key = hopbox_server.first.ssh_host_key
    user     = hopbox_server.first.ssh_user
  }
```

| Attribute | Type | Default | Description |
| --- | --- | --- | --- |
| `address` | string | none, required | The address of the live control server, such as its `vrack_address`. The provider dials it through the jump host when the provider block has one. |
| `host_key` | string | none, required | The SSH host key of that server, in the `authorized_keys` format, such as its `ssh_host_key`. The login refuses another key. |
| `user` | string | none | The login user on that server, such as its `ssh_user`. Without it, the provider tries `debian`, then `root`. |

The copy takes these steps. Steps 1 to 5 come before the reinstall, so each check comes before any write:

1. It reads the live control processes with `ListControlProcesses` over the admin API, which runs over TLS. Each live process reports the fingerprint of each of its key files in `kek_fingerprints` ([the admin API](/reference/admin-api/index.html.md#controlprocess)). An install whose live processes report no fingerprint fails the create, because the provider cannot check a copy.
2. It logs in to `copy_control_from` with the pinned host key and `ssh_private_key`.
3. It reads `/etc/hopbox/hopboxd.toml` there. A control server without `postgres.url` runs the bundled Postgres, which listens only on its unix socket. The create then fails with `the control server runs the bundled Postgres, ...`. Move the install to the external Postgres first ([Run the control store on an external Postgres](/guide/external-postgres/index.html.md#move-a-one-server-install-to-the-external-postgres)). A `postgres.url` that is not a `postgres://` URL with a host and a user, or that holds a password, a space, a quote, or a control character, fails the create too.
4. It reads each `.key` and `.staged` file of the key dir, the file of `postgres.ca_file`, and the file of `postgres.password_file`. It only reads these paths, and never writes there.
5. It computes the fingerprint of each key file ([Sealing](/reference/sealing/index.html.md#key-fingerprints)), and compares it with the one that a live control process reports for that ID. A key file with another fingerprint, with an ID that no live process reports, or with another size than 32 bytes fails the create. So do two live processes that report other fingerprints for one ID, and an empty CA or password file. The copy must also hold the key with which each live process seals, its `kek_current`.
6. After the kernel step, it writes the key files to `/var/lib/hopbox/kek` of the new server, with the mode `0400` and the owner root. The dir has the mode `0700`. It writes the CA to `/etc/hopbox/postgres-ca.pem` and the password to `/etc/hopbox/postgres-password`, both with the mode `0400` and the owner root. The provider picks these paths, never the server that it copies from. The data goes on standard input, never on a command line, and the script refuses any other path.
7. The bootstrap config of the new server names the same `postgres.url` and the two paths of step 6. The install of the control role finds its keys, so it makes no key of its own.

The provider never stages or promotes a key. To add a key to every control server later, stage it, copy it, and promote it as [Sealing](/reference/sealing/index.html.md#several-control-servers) shows.

A control server that is not the first one needs another role in this release, so that it enrolls. The registry then knows the server, and the provider follows it through `GetServer`. The rolling upgrade moves each control server first ([Rolling upgrade](#rolling-upgrade)).

### Uploads

The provider sends a local file of `bundle` or `host_kernel` over SSH in two scripts. The first script writes the file to a `.part` file in `/var/lib/hopbox-provider`. The second script checks the sha256 of the whole file, and moves it to its name. Both scripts take a lock on the part first, with `flock` on a `.part.lock` file. The first script holds the lock while it writes. The empty lock file stays in `/var/lib/hopbox-provider`. A script that removes it lets a later script lock a new file while an old script still holds the removed one.

sshd can drop the connection during a big upload. In a test, it dropped about one connection for each 1 to 2 GB that a laptop sent, with `message authentication code incorrect` in its log. The 600 MB bundle then failed the create after the reinstall. When sshd drops a session, the script of the session goes on: it writes the data that was on its way, reads the end of its input, and exits. So the first script holds no check of its own, and removes nothing.

When the connection breaks, the provider logs in again with the pinned host key, and resumes the upload:

1. It takes the lock, so it waits until the old script is gone. Then it reads the size of the part.
2. It sends the rest of the file, and appends it to the part.
3. It checks the sha256 of the whole file. SSH delivers only authenticated data, so the part holds no corrupt byte. If the sha256 still differs, the provider sends the whole file again.

The log shows `the connection broke during an upload, so the provider resumes it`, or `the part of an upload does not match after a resume, so the provider sends the file again`. Each resume and each new send counts as one try. After 6 tries, the step fails. A file from an `https` URL downloads again from its start, in one script.

### Order

An apply that creates a server with `ovh_plan` orders and pays for a new OVHcloud dedicated server. The plan says so with the warning `Apply orders and pays for a new OVHcloud server`. With `ovh_order_id`, the warning is `Apply follows an existing OVHcloud order`, and the apply places no order.

Before it orders, the create checks each item that can fail without OVHcloud:

- the OVHcloud credentials and the SSH key
- the admin API and the control URLs, for a server that is not the first one
- the vRack address, the floating addresses, and the files of a live control server for a new control server (step 2 of [Create](#create))
- the datacenter
- the files of `bundle` and `host_kernel`: it reads each local file and checks its sha256

An error in this step orders nothing.

The order takes these steps:

1. It reads the subsidiary of the account with `GET /me`.
2. It looks for the carts of the order key, with the description `hopbox_server <ovh_order_key>`. A checked-out cart means that an earlier apply ordered and lost the order. The create then stops ([Lost orders](#lost-orders)). It deletes a cart without a checkout, because that cart holds no order.
3. It makes a cart with that description, which expires after 7 days, and assigns it to the account. Then it lists the carts of the key again. A listed cart that is gone, such as an expired one, does not count. A second cart means that another apply orders with the same key at the same time, for example from another Terraform state. The create then deletes its cart and stops.
4. It puts the plan in the cart, as a monthly rental without a commitment (duration `P1M`, pricing mode `default`). It reads the options of the plan, and checks `ovh_options` against them. Each option exists, each exclusive family has at most one option, and each mandatory family has one. No option starts with `hardraid-`. Then it puts the options in the cart.
5. It sets the config of the server. `dedicated_datacenter` gets `ovh_datacenter`. If the plan needs them, `dedicated_os` gets `none_64.en`, and `region` gets the region of the datacenter ([Datacenters](#datacenters)). The OS of the order does not matter, because the create reinstalls the server. A plan that needs another config fails the create.
6. It reads the price of the cart, and logs it as `ordering a server`. If `ovh_max_price` is set and the price with tax is higher, it deletes the cart and stops.
7. It checks out the cart with `autoPayWithPreferredPaymentMethod` true and `waiveRetractationPeriod` false. OVHcloud charges the preferred payment method of the account. On a personal account, the withdrawal period can delay the delivery past the wait of 2 hours ([An order that is not delivered in time](#an-order-that-is-not-delivered-in-time)). The provider saves the order ID in `ovh_order_id`. The apply output shows the warning `The apply ordered the OVHcloud server of the order <id>. Its checkout costs <price> with tax (<price> without tax), in <currency>.`
8. It polls the status of the order every minute, for at most 2 hours, until it is `delivered`. Then it finds the service name of the server in the entries of the order, and waits until `GET /dedicated/server/{serviceName}` answers.

A step before the checkout that fails deletes the cart, and the error ends with `No order was made`.

The provider never cancels an order, and never ends the rental of a server. A destroy keeps the server rented, as for a server that the account rented already ([Delete](#delete)). To stop paying, end the rental in the OVHcloud manager after the destroy.

#### Datacenters

The provider knows these datacenters, and the region that an order of each one gets. OVHcloud bills its datacenters in Asia and Oceania through the region `canada`.

| `ovh_datacenter` | `region` |
| --- | --- |
| `gra`, `rbx`, `sbg`, `lon`, `fra`, `waw` | `europe` |
| `eu-west-par-a`, `eu-west-par-b`, `eu-west-par-c` | `europe` |
| `bhs`, `ca-east-tor-a` | `canada` |
| `sgp`, `syd`, `ynm` | `canada` |

A plan of OVHcloud sells only some of these datacenters. The OVHcloud manager shows them. OVHcloud refuses an order of a datacenter that the plan does not sell.

#### Rules that keep a plan from ordering

An order costs money, so these rules apply to every resource with `ovh_plan`:

- Only a create orders. A refresh, an import, and an update never order.
- A plan that changes `ovh_plan`, `ovh_datacenter`, `ovh_options`, `ovh_order_key`, or `ovh_order_id` after the create fails with `The order of a server cannot change`. To change the model of a server, add a new resource and remove the old one. You can drop the order attributes and give `ovh_service_name` of the same server instead. That changes nothing on the server.
- A plan that replaces the server fails with `The change orders a new server`. A change of `pool`, `bootstrap`, or `ovh_service_name` replaces a server. The replacement orders a new server, and the old server stays rented.
- A refresh never removes an ordered server from the state. Without the server in the state, the next plan orders again. If the account no longer rents the server, or the install lost it, the refresh fails and says what to do.
- In one run of Terraform, the second create with an `ovh_order_key` fails with `two hopbox_server resources have the ovh_order_key`. The key stays taken for the whole run, also after its order. Two Terraform states can order with one key at the same time. The later create then fails with `another apply orders with the ovh_order_key`.

These rules do not cover a replacement that Terraform decides by itself. `terraform taint`, `terraform apply -replace`, and `create_before_destroy` replace a delivered server: the create of the replacement orders and pays for a new server, and the old server stays rented. Only during a pending setup does the delete refuse, and does the cart of the order block a second create. Give each resource with `ovh_plan` the block `lifecycle { prevent_destroy = true }`, as the example above does. Terraform then refuses each plan that destroys or replaces the server.

#### An order that is not delivered in time

If the order is not delivered after 2 hours, the create fails with `the order <id> is not delivered after 2h0m0s`. OVHcloud goes on with the order. Terraform marks the resource as tainted, and keeps `ovh_order_id` in the state. A step after the delivery that fails, such as the install, does the same.

A tainted resource normally gets a delete and a new create on the next apply. The new create orders a second server. So the delete of an ordered server whose setup did not finish refuses with `the setup of the server of the OVHcloud order <id> did not finish`. To go on with the same order, untaint the resource and apply:

```sh
terraform untaint hopbox_server.first
terraform apply
```

The apply waits for the order again, and then reinstalls and sets up the server. It orders nothing. Until the setup finishes, each refresh warns `The setup of the server of the OVHcloud order <id> did not finish`.

To give the server up instead, run `terraform state rm` on the resource. Then remove the resource from the Terraform files, or replace its order attributes with `ovh_service_name` of another server. Otherwise the next apply orders a new server. End the rental of the server in the OVHcloud manager.

#### Lost orders

An apply can lose an order before Terraform saves it: Terraform or the computer stops during the checkout, or the answer of the checkout gets lost. The cart of the order then stays checked out, with the description `hopbox_server <ovh_order_key>`, for 7 days. The next create finds it and fails with `an earlier apply checked out the OVHcloud cart <cart ID> for the ovh_order_key "<key>"`. It does not order again.

Find the order of that cart in the OVHcloud manager, under the orders of the account. Then set its ID in the Terraform files, and apply:

```hcl
  ovh_order_id = 123456789
```

The create follows that order. To order a new server instead, give another `ovh_order_key`. An order lost more than 7 days ago has no cart left. Check the orders of the account before you apply again.

The first refresh after the setup of the server finished deletes the cart of the order, because the state now holds the order. While the setup is pending, the cart stays. So a create beside the resource, such as with `create_before_destroy`, finds the cart and refuses to order again.

### Update

| Change | What the provider does |
| --- | --- |
| `bundle.sha256` | It puts the bundle on the server, and checks its sha256. In the turn of the server in the [rolling upgrade](#rolling-upgrade), it runs `hopboxd upgrade --bundle <file> --sha256 <sha256> --json` in a transient unit under `systemd-run --wait`. Then it removes the bundle, sets `version`, and waits until the server answers its heartbeats and reports the new version in them. A server answers in the states `ready`, `cordoned`, `draining`, and `shutting_down`, so a node in the middle of a drain upgrades too. The machines keep running ([hopboxd](/reference/hopboxd/index.html.md#upgrade)). Then it reads `hopboxd version --json`. If its `features` do not name `host-firewall`, the release is from before the host firewall. Then the provider removes the table `inet hopbox_host` ([the host firewall](/reference/host-firewall/index.html.md#never-a-lockout), plan question Q79). A failed read or removal gives a warning with the command to run. |
| `labels` | `SetServerLabels`, through the admin API, or through `hopboxd admin server set-labels` on the first server. |
| `domain` | `hopboxd admin config set --domain <domain>` on the first server. |
| `floating_addresses` | It writes the network setup again with the new list, removes the dropped addresses that no longer route to the server from the public interface, and moves each new address to the server. Then it sets `edge.public_ipv4` to the first address in sorted order ([Edge servers and floating addresses](#edge-servers-and-floating-addresses)). |
| `host_kernel.sha256` | If the server does not run the tested kernel, it installs the kernel and reboots. It first cordons the server, so no machine lands on it. It refuses a server that holds machines, because the reboot stops them. After the reboot, it waits until the server is `ready`, and uncordons it. A server that an operator cordoned before stays cordoned, with a warning. |

### Rolling upgrade

An apply that changes the `bundle` of several servers upgrades one server at a time. Release N and release N-1 can run side by side ([the bundled Postgres](/reference/postgres/index.html.md#two-releases-on-one-store), [the control channel](/reference/control-channel/index.html.md#messages-of-two-releases)). So the servers that wait keep working with the servers that moved. The order is fixed:

1. The control servers: the first server, and each server with the role `control`.
2. The servers with the edge role or the DNS role.
3. The nodes.

These rules keep the order:

- One upgrade runs at a time in one apply. Each server waits until the server before it answers its heartbeats on the new version.
- The first upgrade of an apply waits 3 s for the others, because Terraform starts the updates of servers that do not depend on each other at about the same time. Then the server with the lowest rank goes first. A server that still uploads its bundle holds back the servers of a higher rank.
- In its turn, a server of rank 2 or 3 reads the servers of the install. If a server of a lower rank runs an older `hopboxd` than the new bundle, the update fails with `... runs hopboxd <version>, older than <version>. A rolling upgrade moves the control servers first, ...`. The registry of the install has no control role, so the first server counts there by its edge and DNS roles. The check also reads the control processes of the install, with `ListControlProcesses` or, over SSH on the first server, `hopboxd admin control list --json` ([hopboxd](/reference/hopboxd/index.html.md#admin-control-list)). If a live control process runs an older release than the new bundle, the update of an edge, DNS, or node server fails with `the control process <id> on <host> runs hopboxd <version>, older than <version>. A rolling upgrade moves the control servers first, ...`. An install whose control role is older than the call answers `unimplemented`, and counts no control process. The provider removes the bundle that it put on the server, so nothing changed there.
- A server of a lower rank goes first while its update is in the queue of the apply, also while it still uploads its bundle. So the order check fails only for a server of a lower rank that has no update in the apply yet. That refusal is no failure of an upgrade, and the other servers of the apply go on.
- If an upgrade fails, the servers that still wait in the same apply do not upgrade. Their updates fail with `the upgrade of the server <id> failed in this apply, so the rolling upgrade stops before this server`, and the provider removes their bundles. A failure of an upgrade is a failure of `hopboxd upgrade` on the server, or a server that does not come back on the new version.

Terraform runs at most 10 updates at the same time by default (`-parallelism`). With more servers, a node can take its turn before Terraform starts the update of the first server, and the order check then fails it. To upgrade a big install in one apply, add `depends_on` to the resources of the nodes, so Terraform updates them after the first server:

```hcl
resource "hopbox_server" "node1" {
  ovh_service_name = "ns2345678.ip-198-51-100.eu"
  labels           = { zone = "gra" }

  bundle      = local.bundle
  host_kernel = local.host_kernel

  depends_on = [hopbox_server.first]
}
```

`local.bundle` and `local.host_kernel` hold the objects of `bundle` and `host_kernel`, so every server takes the same files. Then one change of `local.bundle` upgrades the whole install in order.

A change of `ovh_service_name`, `pool`, or `bootstrap` replaces the server: Terraform destroys it and creates it again. For a server with `ovh_plan`, the plan refuses the replacement ([Rules that keep a plan from ordering](#rules-that-keep-a-plan-from-ordering)).

A role that a new release of the provider adds comes in place, on the first server only. For example, a first server from before the release that added the role `edge` has the roles `control`, `node`, and `dns`. The next plan updates it with the role `edge`. The update makes a token for the server and the new roles alone with `hopboxd admin enrollment-token create --server <server_id> --roles <new roles>`. It runs `hopboxd install --roles <every role> --json` with the token, and waits until the server is `ready`. The server keeps its ID ([Enrollment and the install PKI](/reference/enrollment/index.html.md#add-roles-to-an-enrolled-server)). A role never goes: a plan that drops one fails with `A role cannot go`. A plan that adds a role to another server fails with `Roles of a node`.

The provider keeps its own cordon of a reboot in the private state of the resource. A step after the cordon can fail, such as the reboot, the wait for `ready`, or the uncordon. The error then ends with `The server <id> stays cordoned. Run hopboxd admin server uncordon <id>, or apply again`. Terraform keeps the old state, so the next apply plans the same update. That update first lifts the cordon of the provider. A refresh or an update that finds the server uncordoned forgets that cordon. So after you uncordon by hand, a later cordon of yours stays.

An import and `terraform state rm` lose the private state, and so an open cordon of the provider. After either one, run `hopboxd admin server get <id>` on the first server. If `cordoned` is `true` and no operator cordoned the server, run `hopboxd admin server uncordon <id>`.

### Delete

A delete of a server that never enrolled, after a failed create, changes nothing on the server and gives a warning. It still takes the server out of the vRack. Otherwise it takes these steps:

1. It reads the server from the install. A server that the install does not know, or a decommissioned one, goes straight to step 4.
2. On the first server, it refuses while another server of the install is not decommissioned and no other control process is live: `the first server runs the only live control role of the install, ...`. A live control process of another enrolled server, from `ListControlProcesses`, keeps the control plane of the others, so the first server can leave the install. A process without a `server_id` does not count, because the provider cannot tell whether it runs on the first server. The decommission of the drain makes the same check.
3. It drains the server with `DrainServer`, and waits until the drain decommissions it ([Drain of a server](#drain-of-a-server)).
4. It runs `systemctl disable --now hopboxd.service` on the server. After a drain, it logs in to the server again first, because the connection of the start can die during a drain of hours. If the stop fails after a drain of this resource, the delete fails, and the resource and its record of the drain stay: `the drain decommissioned the server <id>, but hopboxd did not stop on it: ... Run terraform destroy again to stop it and to remove the secrets of the install. If the provider cannot log in to the server for good, run terraform state rm on the resource, and stop hopboxd and remove the secrets on the server by hand`. A later destroy of that resource runs no drain, so its error starts with `the server <id> is decommissioned, but hopboxd did not stop on it` instead. Without a drain, such as for a server that was already decommissioned outside Terraform, a failed stop gives a warning, and the delete succeeds. On a server with `public_ipv6`, it then removes that address from the interface that holds it, with the rule and the routes of the table 26806. So the next edge server takes the address within seconds, not only when this server leaves the vRack. If that fails, the delete still succeeds, with a warning. Once `hopboxd` stopped, the delete also removes the nftables table `inet hopbox_host` of the [host firewall](/reference/host-firewall/index.html.md), and the file `/var/lib/hopbox/hostfw/vrack-interface`. A server without `nft` gets the package `nftables` first. The server then leaves the vRack, where no jump host reaches it. Without the table, its public address takes SSH again. A server without the role `edge` leaves the vRack only once its table is gone. The login, the stop of `hopboxd`, or the removal can fail. Then the delete keeps the server in the vRack, where the jump host still reaches it. It fails with the title `The server stays in the vRack` and the error `the server <id> keeps its host firewall, because ...`. The resource records it as it records a drain, so a refresh keeps the resource. The error names the commands to run. The resource stays, and the next destroy goes on. On an edge server, whose public address keeps SSH, the delete goes on with a warning.
5. On a server with the role `control`, the first server too, it removes the secrets of the install once `hopboxd` stopped. The server stays rented, and can run another install, so it keeps no secret of this one. The keys of the last control server open every sealed value of the store, so the destroy removes them only when another control server proves that it holds them:
   1. It stops `hopboxd` on the server.
   2. It reads the control processes with `ListControlProcesses`. It waits about 6 s, a little more than the 5 s in which a running control process renews its row, and reads them again. A holder is a live process on another enrolled server whose row `version` grew between the two reads. A clean stop marks the row of a process as stopped at once. But when that mark fails, the row of a stopped server stays live for up to 20 s, without a renewal, so it never counts.
   3. It reads each `.key` file of the server. Each file must hold 32 bytes, and a holder must report it with the same fingerprint in `kek_fingerprints` ([the admin API](/reference/admin-api/index.html.md#controlprocess)).
   4. The bootstrap config of the server must name an external Postgres, `postgres.url`. The bundled Postgres keeps the store on the server, and only its keys open the sealed values there.

   The stop, the checks, and the removal of one control server run alone in the provider process. A server that the same provider process stopped never counts as a holder. So two destroys of control servers in one apply cannot each count the other.

   A staged key that no promote made current seals nothing, so the destroy removes it with the key dir. Its fingerprint stays in `kek_fingerprints`, so the next stage of the install takes a new ID ([Sealing](/reference/sealing/index.html.md#several-control-servers)).

   The keys and the Postgres files stay, with a warning that gives the reason, in each of these cases:
   - The login to the server fails, or `hopboxd` does not stop.
   - The install at `install_url` does not know the server. Without `install_url`, the first server goes out over SSH, and the read after the stop fails, so its keys stay too.
   - No other control server is live, or none renews its row after the stop: `because no other control server is live`, or `because no other control server renewed its row after the stop`.
   - The other control servers run a release from before `kek_fingerprints`, so they report no fingerprint.
   - A holder lacks a key of the server, for example a key that the other servers dropped: `because no other live control server reports its key <id> with the same fingerprint`.
   - A key file has another size than 32 bytes, or a name that is not of the form `1.key`.
   - The server runs the bundled Postgres.
   - A read of the control processes or of the key files fails.

   When the destroy removes the secrets, it removes these items:
   - The key dir `<data_dir>/kek`, only if it is a real dir at that path, not a symlink and not behind one.
   - The CA and the password of the external Postgres, only at `/etc/hopbox/postgres-ca.pem` and `/etc/hopbox/postgres-password`, and only as regular files. The bootstrap config can name any path, such as the CA bundle of the system. So a file of `postgres.ca_file` or `postgres.password_file` at another path stays, and the warning `The server <id> keeps the Postgres files <paths>, ...` names it. Remove it by hand.

   If `hopboxd` did not stop, or the provider could not log in, the secrets stay, and the warning says so. After a drain of this resource, the delete fails instead, as step 4 says.
6. It parks the floating addresses that still route to the server, also the dropped addresses that the server keeps ([Edge servers and floating addresses](#edge-servers-and-floating-addresses)).
7. With `ovh_vrack`, it takes the server out of the vRack ([The vRack](#the-vrack)).

#### Drain of a server

The delete moves every machine of the server cold to other nodes before the server leaves the install ([Cold move](/reference/cold-move/index.html.md#drain-of-a-node)). A running machine stops, moves, and boots again on another node. Each machine keeps its disk, its name, its URL, its SSH port, and its address. The delete calls `DrainServer` ([the admin API](/reference/admin-api/index.html.md#drainserver)), or `hopboxd admin server drain` over SSH on the first server without the admin API. Then it reads the server every 10 s, for at most 6 h, until its state is `decommissioned`. A server without machines waits for the 3 empty checks of the drain, one minute apart, so its delete takes about 2 min.

| Result of the drain | Result of the delete |
| --- | --- |
| The drain decommissions the server. | The delete goes on with step 4. |
| A machine cannot move, and no move runs any more. | The delete fails with `the drain of the server <id> cannot move the machines <name> (<reason>), ..., so they stay on the server with their disks, and the server stays in the install. Fix each reason, then run terraform destroy again`. |
| The drain finds the server empty, but the decommission is refused. | The delete fails with `the drain of the server <id> cannot decommission it: <error>. The server stays in the install`. |
| The server is not decommissioned after 6 h. | The delete fails with `the server <id> did not leave the install within 6h0m0s: ... The drain goes on in the install. Run terraform destroy again: it waits for the drain, then stops hopboxd on the server`. |
| A read of the server fails with `unauthenticated` or `permission_denied`, such as for a revoked admin API key. | The delete fails at once with that error. No wait fixes it. |
| The SSH connection to the first server breaks during the wait. | The delete logs in again and goes on waiting. |

After a failed delete, the server stays in the Terraform state, `hopboxd` keeps running on it, and the server stays `draining` and cordoned. The drain goes on in the install, and can decommission the server before the next destroy. So the resource records the drain in its private state, and a refresh keeps the resource while that record exists, also once the server is `decommissioned` or unknown to its install. The refresh gives the warning `An earlier terraform destroy of the server <id> did not finish, and the install decommissioned the server since. Finish the removal with terraform destroy, or remove the resource from the Terraform files and apply: ... To set the server up again instead, run terraform apply -replace=<address of the resource>, whose destroy does the same first.` The next destroy finds the server decommissioned, starts no drain, and goes on with step 4: it stops `hopboxd` and removes the secrets of the install. A destroy that succeeds drops the record. A stranded machine keeps its disk on the server. Fix each reason, such as a node with room for the machine, and run `terraform destroy` again. The next delete starts the drain again, and the drain tries each stranded machine again ([Drain a node](/guide/drain-a-node/index.html.md#fix-a-stranded-machine)).

Each server goes out through the admin API when the provider block has `install_url` and `admin_api_key`, the first server too. If the provider cannot log in to the server, the drain and the decommission still happen, and the delete fails as step 4 says, so the next destroy stops `hopboxd`. Without the admin API, the first server goes out through `hopboxd admin` over SSH, so it needs SSH with the pinned host key for each step.

The OVHcloud server stays rented, and its disks stay as they are, also a server that the provider ordered (plan question Q43). A destroy never ends a rental. End it in the OVHcloud manager. To use it again, create a `hopbox_server` for it with `ovh_service_name`, which reinstalls it.

The delete of an ordered server whose setup did not finish refuses. So Terraform cannot order a second server in its place ([An order that is not delivered in time](#an-order-that-is-not-delivered-in-time)).

### Read

Each refresh reads the server from the OVHcloud API. A service name that the account no longer rents removes the resource from the state. For a server with `ovh_plan`, the refresh fails instead with `the server <name> that this resource ordered is no longer in the OVHcloud account`. Without the server in the state, the next plan orders a new server.

A create can fail before the enrollment. Its state then has no `server_id`. The refresh keeps such a state as it is, so Terraform can destroy the server and create it again.

Then the refresh reads the server from the install. It uses SSH on the first server and the admin API on the others. It sets `labels`, `pool`, and `version` from the install. On the first server, it also sets `domain` from `hopboxd admin config get`. So the plan shows a real change of the domain. It keeps `roles` as the config orders them, when the server has the same roles. It reads each address of `floating_addresses` from the OVHcloud API, and keeps only the addresses that route to the server. The refresh changes nothing on the server, also for a dropped address that the server keeps ([Edge servers and floating addresses](#edge-servers-and-floating-addresses)). If the install does not answer, the refresh keeps the state as it was, with a warning.

The install can lose a server that is not the first one, or decommission it. The refresh then removes that server from the state, unless a failed destroy started its drain ([Drain of a server](#drain-of-a-server)). Then the resource stays, and the next destroy finishes the removal. For a server with `ovh_plan`, the refresh fails instead, and says how to set up the same OVHcloud server again with `ovh_service_name`. The first server never leaves the state this way. A new create reinstalls the server, and that erases the control store of the whole install. So the refresh fails with `the first server <id> is decommissioned or unknown to its install, but Terraform did not destroy it`. To accept a new install, run `terraform state rm` on the resource, and apply.

### Import

```sh
terraform import hopbox_server.first ns1234567.ip-198-51-100.eu
```

The import takes the OVHcloud service name. The host firewall of a server without the role `edge` closes its public address ([the host firewall](/reference/host-firewall/index.html.md)). So with `ssh_jump_host`, the ID of such a server also names its vRack address, after a comma:

```sh
terraform import hopbox_server.node1 ns1234568.ip-198-51-100.eu,172.31.0.21
```

The refresh after it logs in with `ssh_private_key` as `debian`, then as `root`, and pins the host key. Through the jump host, it dials the vRack address of the import ID, or the public address of the server without one. A server at that vRack address must hold the public address of the service name, or the import fails with `the server at the vRack address <address> does not hold the public address <address> of the server <name>, so it is another server`. An import of a server without the role `edge` without a vRack address fails at the closed public address. The error then names the form of the import ID with the vRack address. It reads the release in `/opt/hopbox/current`, the roles, `edge.public_ipv6`, and `edge.public_ipv6_gateway` of `/etc/hopbox/hopboxd.toml`, and the server ID from the certificate of the server. It reads only the certificate, never the private key. A server with the role `control` and the bundled Postgres cluster of the first install, `/var/lib/hopbox/pg/data/PG_VERSION`, imports with `bootstrap = true`, and the refresh reads its `domain` from the install. Another control server never makes that cluster, so it imports without `bootstrap`. The refresh also reads the vRack address of the server: the address in `172.31.0.0/16` that a network interface holds.

An import of a server that the provider ordered works the same way. It cannot recover the order attributes. If the Terraform files have `ovh_plan`, the first apply after the import writes the order attributes into the state. That apply changes nothing on the server. From then on, the rules of an ordered server apply.

An import cannot recover `bundle`, `host_kernel`, or `admin_api_key`. The next plan shows an update of `bundle` from your Terraform files. If the file name of the bundle names the version that the server runs, the apply does not upgrade. The state of an imported first server has no `admin_api_key`.

### Host key trust

The provider trusts the host key of the first login after a reinstall, and of the first login of an import. This is trust on first use: no other source tells the provider the key of the new server. From then on, it accepts only the pinned key, and it asks the server for the key type of the pin.

The jump host is never trusted on its first use: `ssh_jump_host.host_key` pins it from the start. Through the jump host, the first login after a reinstall pins the host key of the server as above.

To read the files of a live control server for a new control server ([Control servers](#control-servers)), the provider pins the host key of `copy_control_from`. The provider then checks the fingerprint of each key file against the admin API, which runs over TLS, before it writes anything. So a machine that plays the control server at its address cannot plant its own keys, or a Postgres that would accept them, on the new server.

The risk is a machine that sits between the provider and the server at that first login. Such a machine can play the server. It then gets the commands of the install, the bundle, and the hash of the first admin API key. For a server that is not the first one, it also gets an enrollment token. That token works once, for 1 hour, and only against a control role with the CA in the token. It never gets the private key of the operator, because SSH never sends it. To lower the risk, check `ssh_host_key` against the console of the server in the OVHcloud manager after the first apply.

### Timeouts

| Step | Limit |
| --- | --- |
| The reinstall task | 2 hours, polled every 30 s |
| The first SSH login | 30 minutes, tried every 10 s |
| The reboot into the host kernel | 20 minutes |
| `ready` after the install | 5 minutes, polled every 2 s |
| `ready` on the new version after an upgrade | 5 minutes, polled every 2 s |
| The wait of the first upgrade of an apply for the others | 3 s |
| Each OVHcloud API call | 1 minute |
| The attach to and the detach from the vRack | 2 hours, polled every 30 s |
| The proof that another control server runs, at the destroy of a control server | about 6 s, the renewal period of 5 s and 1 s more |
| The move of a floating address | 2 hours, polled every 30 s |
| The delivery of an order | 2 hours, polled every minute |

### Errors

| Error | Cause |
| --- | --- |
| `Give ovh_service_name, the service name of an OVHcloud dedicated server that the account rents, or ovh_plan to order a new one.` | Neither `ovh_service_name` nor `ovh_plan` is set. |
| `Give ovh_service_name to adopt a server that the account rents, or ovh_plan to order a new one, not both.` | Both are set. |
| `An order with ovh_plan needs <attribute>.` | `ovh_plan` without `ovh_datacenter`, `ovh_options`, or `ovh_order_key`. |
| `<attribute> goes only with ovh_plan, which orders a new server.` | An order attribute with `ovh_service_name`. |
| `The provider does not know the datacenter "<value>". It knows ...` | An `ovh_datacenter` that is not in the list. |
| `the order key "<value>" must have 1 to 63 characters ...` | A bad `ovh_order_key`. |
| `ovh_order_id must be the ID of an OVHcloud order, a number above 0.` | A bad `ovh_order_id`. |
| `ovh_max_price must be a price above 0.` | A bad `ovh_max_price`. |
| `the cart <cart ID> has no price with tax to compare with ovh_max_price. No order was made` | The checkout preview has no price with tax, or no currency, and `ovh_max_price` is set. |
| `the cart costs <price> with tax, above ovh_max_price <value> <currency>. No order was made` | The checkout costs more than `ovh_max_price`. |
| `The order of a server cannot change` | A change of an order attribute after the create. |
| `The change orders a new server` | A change of `pool`, `bootstrap`, or `ovh_service_name` on a server with `ovh_plan`. |
| `two hopbox_server resources have the ovh_order_key "<key>". Give each one its own key` | Two resources with one key in one run of Terraform. |
| `another apply orders with the ovh_order_key "<key>" now, in the cart <cart ID> ...` | Another apply makes an order with the same key at the same time, for example from another Terraform state. No order was made. Give each resource its own key. |
| `the ovh_options of the plan <plan>: ...` | An option that the plan does not have, two options of an exclusive family, a mandatory family without an option, or a `hardraid-` storage option. No order was made. |
| `the order of the plan <plan> needs the config <label>, which the provider does not set` | A plan that needs a config that the provider does not know. No order was made. |
| `<bundle or host_kernel>: <file> has the sha256 <got>, not <want>` | A local file that differs from its `sha256`, before an order. No order was made. |
| `an earlier apply checked out the OVHcloud cart <cart ID> for the ovh_order_key "<key>" ...` | An order that an earlier apply lost. See [Lost orders](#lost-orders). |
| `check out the cart <cart ID>: ... If OVHcloud made the order anyway, the next apply finds the cart and does not order again` | The checkout failed, or its answer got lost. |
| `the order <id> is not delivered after <time> ...` | OVHcloud did not deliver the server in 2 hours. The message adds `Pay it in the OVHcloud manager` for an order that is `notPaid`, and asks for the documents for an order that is `documentsRequested`. See [An order that is not delivered in time](#an-order-that-is-not-delivered-in-time). |
| `the order <id> is cancelled at OVHcloud, so no server comes from it ...` | OVHcloud or the account cancelled the order. Run `terraform state rm` on the resource. The next apply orders a new server. |
| `the setup of the server of the OVHcloud order <id> did not finish, so the provider keeps the resource ...` | A delete of an ordered server whose setup did not finish, such as the replacement of a tainted resource. Run `terraform untaint` on the resource and apply. To give the server up, run `terraform state rm`, and remove the resource or its order attributes from the Terraform files. |
| `the server <name> that this resource ordered is no longer in the OVHcloud account ...` | The account no longer rents an ordered server. Run `terraform state rm` on the resource to order a new one. |
| `the server <id> (<name>) that this resource ordered is decommissioned or unknown to its install ...` | The install lost an ordered server that is not the first one. Run `terraform state rm` on the resource, and give `ovh_service_name` of the same server to set it up again. |
| `The first server of an install (bootstrap = true) needs the domain of the install.` | `bootstrap = true` without `domain`. |
| `Only the first server of an install (bootstrap = true) takes the domain.` | `domain` without `bootstrap`. |
| `the first server has the roles control, node, dns, edge. Leave roles out to get them` | Another list in `roles` of the first server. |
| `unknown role "<role>". The roles are control, node, dns, and edge`, `the role <role> appears twice`, or `a server needs at least one role: control, node, dns, or edge` | A bad list in `roles` of another server. |
| `the DNS role runs on edge servers, so a server with the role dns also needs the role edge` | `dns` without `edge`. |
| `in this release, a control server that is not the first one also needs another role, such as node, so that it enrolls` | `roles = ["control"]` on a server that is not the first one. |
| `Invalid vRack address` with `the vRack address "<value>" must be an IPv4 address in 172.31.0.0/16, ...` | A bad `vrack_address`. |
| `The vRack address of a server cannot change` | A change of `vrack_address` without a replacement of the server. |
| `Invalid floating address` with `the floating address "<value>" must be one public IPv4 address, such as 203.0.113.5` | A bad entry of `floating_addresses`. |
| `Floating addresses of a server without the edge role` | `floating_addresses` on a server without the role `edge`. |
| `Invalid public IPv6 address` with an error of `edge.public_ipv6` or `edge.public_ipv6_gateway` | A bad `public_ipv6` or `public_ipv6_gateway`, or only one of the two. The errors are those of the bootstrap config ([the bootstrap config of hopboxd](/reference/hopboxd-config/index.html.md#errors)). |
| `A public IPv6 address of a server without the edge role` | `public_ipv6` on a server without the role `edge`. |
| `The public IPv6 address of a server cannot change` | A change of `public_ipv6` or `public_ipv6_gateway` without a replacement of the server. |
| `The vRack is not valid` with `ovh_vrack "<value>" must be the service name of a vRack, such as pn-12345` | A bad `ovh_vrack`. |
| `The jump host is not valid` with `ssh_jump_host.address ...` or `ssh_jump_host.host_key must be an SSH public key ...` | A bad `ssh_jump_host`. |
| `The vRack or the jump host is not known yet` | `ovh_vrack` or `ssh_jump_host` names an attribute of a resource that the same apply creates. Give literal values, such as the outputs of an earlier apply. |
| `every server of an install on the vRack <vrack> needs its vrack_address, an address in 172.31.0.0/16` | A server without `vrack_address` while the provider block sets `ovh_vrack`. Nothing changed on the server. |
| `the server has a vrack_address, but the provider block names no vRack: set ovh_vrack` | `vrack_address` without `ovh_vrack`. Nothing changed on the server. |
| `the server <name> is in the vRack <vrack>, not in the vRack <vrack> of the provider block. ...` | The server belongs to another vRack. Nothing changed on the server. |
| `the server <name> bonds its network cards (<mode>), and the provider sets up only servers without link aggregation` | OVHcloud bonds the cards of the server. Nothing changed on the server. |
| `ovh: the task <id> (<function>) of the vRack <vrack> ended with the status cancelled` | The attach to the vRack failed at OVHcloud. Nothing changed on the server. |
| `the vRack interface of the server <name> is in the vRack "<vrack>", not in <vrack>, after the attach and its task` | The task of the attach was gone, but the interface is not in the vRack, after the attach or after the network setup. |
| `the server <name> does not hold its vRack address <address> after the network setup. Read journalctl -u hopbox-network.service on the server` | The boot script found no card with the MAC of the vRack interface, or failed. |
| `remote: the host key of the jump host is not the pinned one: ...` | The jump host shows another key than `ssh_jump_host.host_key`. Check the key of the edge server, then fix the provider block. |
| `two hopbox_server resources have the floating address <address>: ...` | Two resources list one floating address. The plan fails. |
| `two hopbox_server resources have the public IPv6 address <address>: ...` | Two resources have one address of `public_ipv6`. The plan fails. |
| `two hopbox_server resources have the vRack address <address>: ...` | Two resources have one `vrack_address`. The plan fails. |
| `read the Additional IP <address>: ovh: GET /ip/<address>%2F32: HTTP 404 ...` | The account has no such Additional IP. Nothing changed on the server. |
| `the Additional IP <address> routes to the vRack <vrack>. A floating address must route to one server` | The block of the address is in a vRack. Nothing changed on the server. |
| `the Additional IP <address> routes to the server <name>, which is not in the vRack <vrack> of the install, so the provider does not take it` | The address belongs to a server outside the install. Nothing changed on the server. Move or park it in the OVHcloud manager if the install needs it. |
| `the Additional IP <address> routes to the server <name>, and the provider block names no vRack ...` | A floating address of another server, without `ovh_vrack`. Nothing changed on the server. |
| `move the Additional IP <address> to the server <name>: ...` | The move failed. The server runs. Terraform marks it as tainted on a create. |
| `a server that is not the first one needs the URL of the internal listener of a control server, and the install lists no live control server with an internal listener off loopback: ...` | No control URL in the provider block, and none from the install. |
| `a control server that is not the first one needs the vRack, ...` | A control server without `ovh_vrack` and `vrack_address`. |
| `a control server that is not the first one copies the files of a live control server: set copy_control_from ...` or `Missing copy_control_from` | A new control server without `copy_control_from`. |
| `copy_control_from without a new control server` | `copy_control_from` on the first server, or on a server without the role `control`. |
| `log in to the control server of copy_control_from at <address>: remote: the host key of the server is not the pinned one: ...` | The server at that address shows another host key than `copy_control_from.host_key`. Nothing changed on the new server. |
| `no live control process of the install reports the fingerprints of its key files, ...` | The control servers run a release from before `kek_fingerprints`. Upgrade them first. |
| `the key file <name> of the control server at <address> does not match the fingerprint that the install reports for <id>, so the provider copies nothing` | The key file differs from the key of the install. Nothing changed on the new server. |
| `the control server at <address> has no file of the key <id>, with which the install seals, so the provider copies nothing` | The server of `copy_control_from` lacks the current key. |
| `two live control processes report other fingerprints for the key <id>, ...` | The control servers hold other keys with one ID. Fix the key files as [Sealing](/reference/sealing/index.html.md#key-fingerprints) shows. |
| `the file of postgres.ca_file or postgres.password_file of the control server is empty` | The server of `copy_control_from` has an empty CA or password file. |
| `the control server runs the bundled Postgres, which listens only on its unix socket, ...` | The live control server has no `postgres.url`. Move the install to the external Postgres first. |
| `the first server runs the only live control role of the install, and the servers <ids> still belong to it. Destroy them first, or add another control server` | A delete of the first server while it runs the only control role. |
| `the sha256 "<value>" is not 64 lowercase hex digits` | A bad `sha256` in `bundle` or `host_kernel`. |
| `the source "<value>" is a URL, but only https URLs work` | A `source` with `http://` or another scheme. |
| `the provider has no OVHcloud credentials. Set OVH_APPLICATION_KEY, OVH_APPLICATION_SECRET, and OVH_CONSUMER_KEY` | The environment has no OVHcloud credentials. |
| `the provider has no SSH key. Set ssh_private_key in the provider block, or HOPBOX_SSH_PRIVATE_KEY` | No `ssh_private_key`. |
| `a server that is not the first one needs the admin API: set install_url and admin_api_key ...` | A server without `bootstrap`, and no admin API. |
| `Two control URL settings` with `Set control_urls or control_url, not both.` | The provider block sets both `control_urls` and `control_url`. |
| `The control URL is not valid` with `the control URL "<url>" must be an https URL, such as https://10.0.0.1:7443` | A control URL is not an `https` URL of a host. |
| `ovh: GET /dedicated/server/<name>: HTTP 404 Client::NotFound: ...` | The account does not rent the server. |
| `ovh: the task <id> (reinstallServer) of the server <name> ended with the status <status>: ...` | The reinstall failed at OVHcloud. |
| `the disk group <n> sits behind the hardware RAID controller "<name>", and hopbox tests only software RAID` | A server with hardware RAID. |
| `remote: the host key of the server is not the pinned one: ...` | The server shows another host key than `ssh_host_key`. Either it was reinstalled outside Terraform, or a machine sits between you and the server. Check the key before you do anything else. To accept a reinstall of a server that is not the first one, taint the resource. For the first server, run `terraform state rm` on the resource and apply. For a server with `ovh_plan`, first replace the order attributes with `ovh_service_name` of the same server. Otherwise the taint or the new create orders a new server. |
| `the server runs <os> <version>, but hopbox tests only Debian 13 ...` | The reinstall installed another OS. |
| `/var/lib/hopbox is on XFS without reflinks, and it holds files, so the provider does not make it again` | Files in a data dir without reflinks. |
| `the host kernel package is <package> <version> for <arch>, but the provider installs linux-image-6.18.57-hopbox1 6.18.57-1 for amd64` | The wrong package in `host_kernel`. |
| `<file> has the sha256 <got>, not <want>` | A file on the server differs from its `sha256`. |
| `the server booted <release> after the install of 6.18.57-hopbox1. Check the GRUB default on the server` | GRUB booted another kernel. |
| `read the IP addresses of the server <name>: ...` | The OVHcloud API did not give the IP blocks of the server. The create stops before the reinstall. |
| `the server <name> holds none of its IP addresses <blocks> on a network interface` | No `/32` or `/128` block of the server is on a network interface after the reinstall. |
| `write the bootstrap config: ...` | The provider cannot write `/etc/hopbox/hopboxd.toml`. |
| `the server <id> did not become ready: ...` | The server did not send a heartbeat in 5 minutes. Read `journalctl -u hopboxd` on the server. |
| `the server <id> did not come back on <version>: ...` | After its upgrade, the server did not become `ready` with the new version in 5 minutes. Read `journalctl -u hopboxd` on the server, and run `hopboxd upgrade --rollback` there if the new release does not start. |
| `the server <id> with the roles <roles> runs hopboxd <version>, older than <version>. ...` | A server of a higher rank waits for a server of a lower rank in the [rolling upgrade](#rolling-upgrade). Upgrade that server first, and apply again. |
| `the upgrade of the server <id> failed in this apply, so the rolling upgrade stops before this server. ...` | An earlier upgrade of the same apply failed. Fix that server, and apply again. |
| `read the servers of the install for the order of the rolling upgrade: ...` | The admin API did not answer the list of the servers. Nothing changed on the server. |
| `the server <id> holds <n> machines. Delete them first ...` | A delete of a server with machines. |
| `the server <id> runs <release> and holds <n> machines. The install of the host kernel reboots it, so delete the machines first` | A change of `host_kernel` on a server with machines that does not run the tested kernel. |
| `the first server <id> is decommissioned or unknown to its install, but Terraform did not destroy it. ...` | Someone decommissioned the first server outside Terraform. Run `terraform state rm` on the resource to accept a new install, which erases the server. For a server with `ovh_plan`, the error says `that this resource ordered` and gives the service name: replace the order attributes with `ovh_service_name` of the same server before you apply, or the apply orders a new server. A `terraform destroy` can also decommission the server and then lose the answer. Then the next destroy fails on this error at its refresh. Run `terraform destroy -refresh=false` to finish the destroy, or `terraform state rm` on the resource. |
| `log in to the first server <name> to decommission it: remote: the host key of the server is not the pinned one: ... run terraform state rm on the resource` | A delete of a first server that someone reinstalled outside Terraform. Its install is gone. For a server with `ovh_plan`, the error adds that the order attributes must give way to `ovh_service_name` before the next apply. |
| `/var/lib/hopbox-provider is not a dir of root with mode 0700. Remove it` | Another user made the work dir of the provider. |

A failed script on the server gives `the script exited with status <n>:` and the end of its error output. A connection that breaks during a script gives `remote: the connection broke: ...`. The provider resumes an upload after it ([Uploads](#uploads)), and fails every other step.

## Resource hopbox_pool

A `hopbox_pool` is one pool of servers of an install ([the admin API](/reference/admin-api/index.html.md#poolservice)). All servers of a pool have identical hardware. The first server that enrolls into the pool records its hardware, and a new server with other hardware fails to enroll ([Enrollment and the install PKI](/reference/enrollment/index.html.md#hardware-of-a-pool)). So give each server model its own pool. A `hopbox_server` joins the pool with `pool = hopbox_pool.<name>.name`, which also makes Terraform create the pool first.

```hcl
resource "hopbox_pool" "big" {
  name     = "big"
  ovh_plan = "24adv01"
}
```

The resource calls the admin API at `install_url` with `admin_api_key` ([Provider block](#provider-block)). So it needs both, as a server that is not the first one does. The first server joins the pool `default`, because no other pool exists before it. A plan that gives the first server another pool fails.

| Attribute | Type | Default | Change | Description |
| --- | --- | --- | --- | --- |
| `name` | string | none, required | replaces | The name of the pool, a lowercase DNS label. No other live pool can have it. |
| `ovh_plan` | string | `""` | replaces | The OVHcloud plan code of the servers of the pool, such as `24adv01`: 1 to 64 of `a-z`, `0-9`, `.`, and `-`, which start with a letter or a digit. Leave it out for a pool of adopted servers. |
| `id` | string | set by the provider | | The ID of the pool, a UUID. |

The create calls `CreatePool` with a new idempotency key. The delete calls `DeletePool`. The install refuses the delete of a pool that holds a server that is not decommissioned, or that a live size names, so destroy the servers and the sizes of the pool first. Terraform orders the destroys that way for the servers and the sizes that name the pool by `hopbox_pool.<name>.name`. A delete of a pool that is gone already succeeds.

A refresh calls `GetPool`. A pool that is gone leaves the state, so the next apply creates it again. A pool of the same name with another ID also leaves the state, after someone deleted the pool and created it again outside Terraform. The next create then fails with `already_exists`, so import that pool instead. A change of `ovh_plan` replaces the pool: the destroy of the old pool fails while it holds servers or live sizes. Import a pool by its name:

```sh
terraform import hopbox_pool.big big
```

## Resource hopbox_size

A `hopbox_size` is one size of machine of an install ([the admin API](/reference/admin-api/index.html.md#sizeservice)). A user picks a size for a new machine, never a pool. The pool and the labels of the size are the placement filters of its machines ([Placement](/reference/placement/index.html.md#hard-filters)).

```hcl
resource "hopbox_size" "large" {
  name       = "large"
  vcpus      = 8
  memory_gib = 32
  disk_gb    = 200
  pool       = hopbox_pool.big.name
  labels     = { disk = "nvme" }
}
```

The resource needs `install_url` and `admin_api_key`, as `hopbox_pool` does.

| Attribute | Type | Default | Change | Description |
| --- | --- | --- | --- | --- |
| `name` | string | none, required | replaces | The name of the size, a lowercase DNS label. No other size can have it, also no retired one. |
| `vcpus` | number | none, required | refused | The count of vCPUs, 1 to 32. |
| `memory_gib` | number | none, required | refused | The RAM, in GiB. |
| `disk_gb` | number | none, required | refused | The size of the sparse disk, in GB. |
| `pool` | string | `""` | refused | The pool of the machines of the size, such as `hopbox_pool.big.name`. Leave it out for any pool. |
| `labels` | map of string | `{}` | refused | The labels that a node must have to take a machine of the size. The rules of the labels of a server apply. |
| `id` | string | set by the provider | | The name of the size. |
| `retired` | bool | set by the provider | | `true` once the size takes no new machine, for example after `hopboxd admin size retire` outside Terraform. |

A size never changes ([the admin API](/reference/admin-api/index.html.md#sizeservice)). A destroy retires the size: it takes no new machine, its machines keep it, and its name stays taken. So a replace that keeps the name cannot work. The plan refuses a change of `vcpus`, `memory_gib`, `disk_gb`, `pool`, or `labels` that keeps `name`, with `A size never changes`. To change a size, give it a new name too. Terraform then retires the old size, and creates the new one.

The create calls `CreateSize` with a new idempotency key. The delete calls `RetireSize`. A delete of a size that is gone already succeeds. A refresh calls `GetSize`. A size that is gone leaves the state. A size whose RAM is not a whole number of GiB, or whose disk is not a whole number of GB, fails the refresh, because the resource cannot show it. Import a size by its name:

```sh
terraform import hopbox_size.large large
```

### Errors of hopbox_pool and hopbox_size

| Error | Cause |
| --- | --- |
| `hopbox_pool and hopbox_size need the admin API: set install_url and admin_api_key in the provider block, or HOPBOX_INSTALL_URL and HOPBOX_ADMIN_API_KEY` | The provider has no admin API. |
| `invalid name: the pool name ...` | A `name` of a pool that is not a lowercase DNS label. |
| `invalid OVHcloud plan: ...` | A bad `ovh_plan`. |
| `vcpus must be from 1 to 32, not <n>`, `memory_gib must be from 1 to <max>, not <n>`, `disk_gb must be from 1 to <max>, not <n>` | A size out of bounds. |
| `invalid label: ...` | A label of a size that breaks a rule. |
| `The size <name> keeps its name after its retirement, so a replace with the same name fails. ...` | A change of a size that keeps its name. |
| `The first server of an install (bootstrap = true) joins the pool default, because the install has no other pool before it.` | A first server with another `pool`. |
| `the admin API: already_exists: ...` | A pool or a size of the same name exists already. Import it, or pick another name. |
| `the admin API: failed_precondition: ...` | The install refuses the delete of a pool that holds servers or that live sizes name. |

## Commands of the binary

Terraform starts the binary without arguments. So `serve` is the default command.

```sh
terraform-provider-hopbox [serve] [--debug]
terraform-provider-hopbox version [--json]
```

| Command | Description |
| --- | --- |
| `serve` | Serves the provider to Terraform over the plugin protocol. It is the default command. When a person starts it outside Terraform, it prints a note that the binary is a plugin, and exits with status 1. |
| `version` | Prints the version of the provider. See [the version command](/reference/version/index.html.md). |

### Flags of serve

| Flag | Type | Default | Description |
| --- | --- | --- | --- |
| `--debug` | bool | `false` | Serves the provider in debug mode, for a Go debugger such as `dlv`. The provider prints a `TF_REATTACH_PROVIDERS` value. Set that value in the environment of Terraform, and Terraform then uses the running provider. |
| `-h`, `--help` | bool | `false` | Prints the help and exits. |

Example:

```sh
terraform-provider-hopbox --debug
```

The provider reports the version from the `version` command to Terraform.

## Exit status

| Status | Meaning |
| --- | --- |
| 0 | The command succeeded. |
| 1 | The command failed, or `serve` ran outside Terraform. |
| 80 | The command line is not valid, for example an unknown command or flag. |

## Tests

| Test | Where it runs | What it checks |
| --- | --- | --- |
| `internal/tfprovider/ovh` | `make test` | The signature of a known example, and each call against a fake OVHcloud API in `ovh/ovhtest`. The fake checks the keys, the timestamp, and the signature as the real API does. |
| `internal/tfprovider/remote` | `make test` | The host key pin, the login users, and scripts with their input and exit status, against an SSH server in the test. A login through a jump host, with the pins of both hosts. |
| `internal/tfprovider/hostsetup` | `make test` | The scripts of each step, and that no token or private key appears in them. |
| `TestResumeAfterADroppedUpload` | `make test-linux` | The upload scripts with a real `sh`, as root in a private mount namespace. A dropped upload goes on in the background, and a slow resume runs beside it. The resume waits for the lock, appends the rest, and the old script removes nothing. |
| `TestInstallKernelPinsTheRelease` | `make test-linux` | The kernel step of the create, as root in a private mount namespace, with stubs of `dpkg -i` and `update-grub`. The GRUB pin names the kernel release `6.18.57-hopbox1`, not the package version `6.18.57-1`. |
| `TestCreateBootstrap`, `TestCreateNode`, `TestCreateRefuses`, `TestUpdate`, `TestRemove`, `TestRead` | `make test` | The flow of the resource against the fake OVHcloud API and a fake server. |
| `TestUpgradeRank`, `TestUpgradeQueueOrder`, `TestUpgradeQueueStopsAtAFailure`, `TestRollingUpgrade`, `TestUpgradeRefusesANodeBeforeItsControlServer`, `TestUpgradeWaitsForTheNewVersion` | `make test` | The [rolling upgrade](#rolling-upgrade): one upgrade at a time in the order of the ranks, also when the update of a node starts first. The stop after a failed upgrade, the refusal of a node before its control server, and the wait for the new version in the heartbeat. |
| `TestProvisionOrders`, `TestProvisionRefusesBeforePaying`, `TestProvisionFindsALostOrder`, `TestProvisionDeletesAStaleCart`, `TestProvisionWaitsForTheDelivery`, `TestProvisionCancelledOrder`, `TestReadKeepsAnOrderedServer`, `TestReadKeepsAnOrderedNode`, `TestRemoveRefusesAPendingOrder`, `TestOrderKeyTakenForTheRun`, `TestOrderRefusesASecondCartOfTheKey`, `TestLostOrderAfterTheCartExpired` | `make test` | The order flow against the fake OVHcloud API, which sells plans from a stock, checks each cart, and can lose the answer of a checkout. Each test counts the orders, so that no path orders twice. No test calls the real OVHcloud API. |
| `TestOrderedServerLifecycle`, `TestOrderedServerResumes`, `TestPendingOrderBlocksASecondCreate`, `TestOrderFollowPlan` | `make test` | Terraform orders a server, updates it, refuses the changes that order again, imports it, and destroys it. Over the plugin protocol: a tainted order whose delete refuses, and an untaint that goes on with the same order. A second create beside a pending order refuses to order. |
| `TestCreateNodeThroughTheJumpHost`, `TestCreateEdgeAndFirstOnTheVRack`, `TestCreateEdgeWithoutDNS`, `TestCreateControlServer`, `TestFloatingAddresses`, `TestFloatingAddressesStayInTheInstall`, `TestFirstServerLeavesThroughTheAPI`, `TestDestroyKeepsSecretsOnlyWithAWarning`, `TestOnlyControlServerKeepsItsKeys`, `TestTwoDestroysKeepTheLastKeys`, `TestDestroyKeepsKeysThatNoOtherServerHolds`, `TestStaleRowKeepsTheLastKeys`, `TestDestroysOfOneProcessTakeTurns`, `TestOwnRowIsNoHolder`, `TestVRackChecksAfterTheAttach`, `TestControlCopyRefusesOddKeys`, `TestDestroyParksNoAddressOfAnotherServer` | `make test` | The fleet against the fake OVHcloud API, which also serves the network interfaces, the vRack, and the Additional IPs. A node attaches to the vRack before its reinstall, and each SSH step goes through the jump host: at the public address up to the network setup, and at the vRack address from then on. Edge servers and the first server get their SSH straight. An edge server with `public_ipv6` gets it in its bootstrap config and in `dns.listen`, and an import reads it back. A control server copies the key files and the Postgres of the live control server of `copy_control_from` before its install. It refuses before any write without a source, with another host key of the source, with a key file of another fingerprint, without the current key, without fingerprints, and with a source on the bundled Postgres. A replaced edge server keeps its floating addresses. The DNS role of an edge server listens on its floating addresses from its create, and an update of the floating addresses sets `dns.listen` again. An edge server without the DNS role gets a bootstrap config only for `public_ipv6`. No address moves from a server outside the vRack, and a destroy parks the addresses of the edge server. The first server leaves the install through the admin API while another control server is live. A destroy of a control server removes its keys and its Postgres files after `hopboxd` stops, and keeps them with a warning when the stop fails. The only control server, a control server on the bundled Postgres, a server with a key that no other live server reports, a server that the install does not know, and at least one of two control servers that two parallel destroys stop keep their keys. A stale live row without a renewal holds no key, the destroys of one provider process take turns, and the own row of a server never holds its keys. An attach that OVHcloud reports done but that did not happen fails the create. A key of another size, two rows with other fingerprints, and an empty CA refuse the copy. A destroy parks no address that another server claims. |
| `TestFleetOnTheVRack`, `TestProviderBlockOfTheFleet`, `TestDoubleClaimFailsAtPlan` | `make test` | Terraform applies a fleet of five servers in two pools through the jump host: the first server, a second edge server, a second control server, and two nodes. It replaces the edge server with `create_before_destroy`, and the new server takes its floating address. The bootstrap config of both edge servers names the same `public_ipv6`, and their `dns.listen` holds port 53 of that address and of the floating address. The old edge server removes the address after its stop. An import of the edge server reads `public_ipv6` and `public_ipv6_gateway` back. A plan that changes a vRack address fails. A plan that changes `public_ipv6` in place fails too. A floating address, a vRack address, or a public IPv6 address of two resources fails the plan, and no address moves. |
| `TestPoolAndSizeLifecycle`, `TestPoolAndSizeValidation`, `TestPoolNeedsTheAdminAPI` | `make test` | Terraform creates a pool and a size against a fake admin API over Connect, imports them, refuses a change of a size that keeps its name, renames the size, and destroys both. They need the `terraform` binary as the tests below do. |
| `TestServerResourceLifecycle`, `TestServerResourceValidation` | `make test` | Terraform itself runs a create, an update, an import, and a destroy, and plans bad Terraform files. They need the `terraform` binary on `PATH`, or in `TF_ACC_TERRAFORM_PATH`. Without it, terraform-plugin-testing downloads one. |
| `TestInstallOverSSH` | `make test-provider`, and the CI job `install` | The scripts of the provider over real SSH, in a Debian 13 container with systemd and sshd. The test covers the key swap and the data dir without reflinks. It covers the refusal of a wrong kernel package, and the install and enrollment of a first server, with `dns.listen` and the public addresses. It also covers a call with the first admin API key, the labels, an import, an upgrade, and the decommission. The container runs the kernel of the host, so the test does not reboot. A real run on the [tested server](/reference/preflight/index.html.md#the-tested-server) covers the reboot. The test does not pass on an Arm Mac with Docker Desktop. Its VM has no `/dev/kvm`, so the preflight check of the install fails. Under the amd64 emulation, the system services of the container fail to start, such as sshd. Run it on a Linux host for amd64 with KVM. |
| `TestAccServer` | by hand, with `TF_ACC=1` | A create and a destroy on a real OVHcloud server that the account rents, with the credentials of the owner. No test orders a real server, because an order costs money. It erases the server. So the test skips unless `HOPBOX_ACC_ERASE` names the same server as `HOPBOX_ACC_SERVICE_NAME`. It also needs `HOPBOX_ACC_DOMAIN`, `HOPBOX_ACC_BUNDLE`, `HOPBOX_ACC_BUNDLE_SHA256`, `HOPBOX_ACC_KERNEL`, `HOPBOX_ACC_KERNEL_SHA256`, the OVHcloud variables, and `HOPBOX_SSH_PRIVATE_KEY`. |
