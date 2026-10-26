# hopbox Terraform provider

> The hopbox Terraform provider, its provider block, the OVHcloud API rights, the resource hopbox_server, its steps and errors, and the binary.

`terraform-provider-hopbox` is the hopbox Terraform provider. Terraform runs it as a plugin. It adopts OVHcloud dedicated servers into an install, or orders new ones. It reinstalls each server with the tested OS. Over SSH, it installs the tested host kernel and `hopboxd`, and enrolls the server. Then it manages the server through the admin API. The code is in `internal/tfprovider`. [Install hopbox on an OVHcloud server with Terraform](/guide/install-on-ovhcloud/index.html.md) walks through a first install.

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

Every attribute is optional. An attribute that you leave out takes the value of its environment variable.

| Attribute | Type | Environment variable | Description |
| --- | --- | --- | --- |
| `ssh_private_key` | string, sensitive | `HOPBOX_SSH_PRIVATE_KEY` | The SSH private key of the operator, in the OpenSSH or PEM format. A key with a passphrase does not work. The provider authorizes its public key on each server that it reinstalls, and logs in with it for every step after the first login. A create, an update, a delete, and an import need it. |
| `install_url` | string | `HOPBOX_INSTALL_URL` | The URL of the install, such as `https://example.com`. The provider calls the admin API there with `admin_api_key`. It must be an `https` URL. Only a loopback address can use `http`. |
| `admin_api_key` | string, sensitive | `HOPBOX_ADMIN_API_KEY` | An admin API key, `hbxadm_...` ([CLI tokens, API keys, admin API keys, and SSH keys](/reference/credentials/index.html.md)). The provider sends it as `Authorization: Bearer <key>`. |
| `control_url` | string | `HOPBOX_CONTROL_URL` | The URL of the internal listener of the control role, such as `https://10.0.0.1:7443`. A new server that is not the first one enrolls there, as `channel.control_urls` ([the bootstrap config of hopboxd](/reference/hopboxd-config/index.html.md#keys)). It must be an `https` URL. Not with `control_urls`. |
| `control_urls` | list of strings | none | The URLs of the internal listeners of the control servers, such as `["https://172.31.0.11:7443", "https://172.31.0.12:7443"]`. A new server that is not the first one enrolls at the first one that answers, and keeps its control channel to one of them. Each URL must be an `https` URL. Not with `control_url`: both fail with `Two control URL settings` and the detail `Set control_urls or control_url, not both.` |

The servers that are not the first one need `install_url`, `admin_api_key`, and `control_urls` or `control_url`. A `hopbox_pool` and a `hopbox_size` need `install_url` and `admin_api_key`. The first server needs none of them: the provider reaches its admin API over SSH. The admin API is public behind the edge role, so `install_url` can be `https://<domain>` ([the admin API](/reference/admin-api/index.html.md#where-the-admin-api-is-served)). But the internal listener of the first server listens only on `127.0.0.1` ([the bootstrap config of hopboxd](/reference/hopboxd-config/index.html.md#keys)), so a new server cannot reach a control URL. So in phase 1, an install has only its first server. Phase 2 puts the internal listener on the vRack.

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
| `roles` | list of string | `["control", "node", "dns", "edge"]` with `bootstrap`, else `["node"]` | updates in place | The roles of the server. In this release, only these two lists are valid, in any order. Leave it out. A refresh reads the roles that the server has: `control` from its bootstrap config, and the others from the registry of the install. |
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
2. It makes an ephemeral ed25519 SSH key. It reinstalls the server with the template `debian13_64` and the public key of the ephemeral key. The default disk group gets this layout:

   | Mount point | File system | Size | RAID |
   | --- | --- | --- | --- |
   | `/` | ext4 | 100 GiB | RAID1 over 2 disks, or none with 1 disk |
   | `/var/lib/hopbox` | XFS | the rest of the disks | the same |

   A disk group behind a hardware RAID controller is refused. A server with more than two disks uses two of them.
3. It polls the task of the reinstall every 30 seconds, for at most 2 hours. A task that ends as `cancelled`, `customerError`, or `ovhError` fails the create.
4. It logs in with the ephemeral key as `debian`, then as `root`, and retries every 10 seconds for at most 30 minutes. It pins the host key of this first login in `ssh_host_key`. See [Host key trust](#host-key-trust).
5. It adds the public key of `ssh_private_key` to the `authorized_keys` of the login user, and logs in again with that key. Then it removes the ephemeral key from the `authorized_keys` of root and of the login user. The ephemeral key lives only in the memory of the provider.
6. It checks that the server runs Debian 13 on `amd64`.
7. It checks that `/var/lib/hopbox` is a file system of its own, on XFS with reflinks. If the XFS has no reflinks and holds no file, the provider makes the file system again. It runs `mkfs.xfs -m reflink=1` with the same UUID, and the apply shows a warning. A file system with files is never made again.
8. If the server does not run `6.18.57-hopbox1`, it installs the host kernel. It puts the package on the server ([Uploads](#uploads)), checks its sha256, and checks that the package is `linux-image-6.18.57-hopbox1` version `6.18.57-1` for `amd64`. It runs `dpkg -i`, pins the GRUB default to the new kernel in `/etc/default/grub.d/20_hopbox_kernel.cfg`, and checks `/boot/grub/grub.cfg`, as `build/hostkernel/install.sh` does. Then it reboots the server from a systemd timer. It logs in again with the pinned host key until the boot ID changes, for at most 20 minutes. It checks that the new boot runs the tested kernel.
9. On the first server, and on a server with the DNS role, it finds the public addresses of the server. It reads the global addresses of the network interfaces of the server with `ip -j addr show scope global`. A public address is the address of a `/32` or `/128` IP block of step 1 that a network interface holds. An IPv6 address that is `tentative` or `dadfailed` does not count, because it does not answer yet. The list puts IPv4 first. A block that no interface holds stays out, such as an IPv6 block without a route. So DNS never points at an address that answers nothing. If no block passes, the create fails.
10. On a server with the DNS role, it writes the bootstrap config `/etc/hopbox/hopboxd.toml` before the install. The file holds the roles, and `dns.listen` on port 53 of each public address ([the bootstrap config of hopboxd](/reference/hopboxd-config/index.html.md#keys)). The tested OS runs `systemd-resolved`, which holds port 53 on `127.0.0.53` and `127.0.0.54`. So the default `dns.listen` of every address fails to bind ([the DNS role](/reference/dns/index.html.md)). `hopboxd install` keeps the key. For example, OVHcloud reports `203.0.113.10/32` and `2001:db8::10/128` for a server, and a network interface of the server holds both. Then the file is:

    ```toml
    # Written by the hopbox Terraform provider before the first hopboxd install.
    roles = ["control", "node", "dns", "edge"]

    [dns]
    listen = ["203.0.113.10:53", "[2001:db8::10]:53"]
    ```

11. On the first server, it makes the admin API key and saves it in the state. On another server, it makes an enrollment token through the admin API, with an `Idempotency-Key` header.
12. It puts the bundle in `/var/lib/hopbox-provider` and checks its sha256 there ([Uploads](#uploads)). It unpacks the bundle and runs its `hopboxd install --json` with these flags ([hopboxd](/reference/hopboxd/index.html.md#install)):

    | Server | Flags |
    | --- | --- |
    | First server | `--roles control,node,dns,edge --domain <domain> --bootstrap-admin-key-hash <sha256 of the key>` |
    | Other server | `--roles node`, one `--control-url <url>` for each URL of `control_urls` (or the one `control_url`), and the enrollment token in `HOPBOX_ENROLLMENT_TOKEN` |

    The install runs in a transient systemd unit, under `systemd-run --wait`, so a dropped SSH session does not stop it. The token reaches the unit through a file of mode 0600 in `/var/lib/hopbox-provider`, never through a command line. The provider removes the bundle, the unpacked release, and the token file after the install.
13. On the first server, it makes a token with `hopboxd admin enrollment-token create --pool <pool> --roles node,dns,edge`. It gives the token to `hopboxd install --roles control,node,dns,edge --json`. So the node role, the DNS role, and the edge role enroll with the control role of the same server ([the DNS role](/reference/dns/index.html.md), [the edge role](/reference/edge/index.html.md)). The token never leaves the server.
14. It waits until the server is `ready`, for at most 5 minutes. On the first server, it reads `hopboxd admin server get --json` over SSH. On another server, it calls `GetServer` of the admin API. A server that is ready but `degraded` gives a warning with its problems.
15. It sets the labels with `SetServerLabels`, through `hopboxd admin server set-labels` on the first server.
16. On the first server, it sets the public addresses of the install to the public addresses of step 9, with `hopboxd admin config set --public-address`. The zone of the DNS role then points the domain, `ns1`, `ns2`, and every machine host at the server ([the DNS role](/reference/dns/index.html.md#the-zone-of-the-install)). The install runs at this step. So a failure gives the warning `The provider did not set the public addresses of the install`, with the command to run on the server, and the create succeeds. A failed create taints the server, and the next apply then erases the install.

The state keeps what each step learned. If a step fails, Terraform marks the resource as tainted, and the next apply destroys it and creates it again, with a new reinstall. A failed preflight check of severity `warn`, such as a kernel that is not the tested one, gives a warning on the apply.

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
| `bundle.sha256` | It puts the bundle on the server, and checks its sha256. In the turn of the server in the [rolling upgrade](#rolling-upgrade), it runs `hopboxd upgrade --bundle <file> --sha256 <sha256> --json` in a transient unit under `systemd-run --wait`. Then it removes the bundle, sets `version`, and waits until the server is `ready` and reports the new version in its heartbeat. The machines keep running ([hopboxd](/reference/hopboxd/index.html.md#upgrade)). |
| `labels` | `SetServerLabels`, through the admin API, or through `hopboxd admin server set-labels` on the first server. |
| `domain` | `hopboxd admin config set --domain <domain>` on the first server. |
| `host_kernel.sha256` | If the server does not run the tested kernel, it installs the kernel and reboots. It first cordons the server, so no machine lands on it. It refuses a server that holds machines, because the reboot stops them. After the reboot, it waits until the server is `ready`, and uncordons it. A server that an operator cordoned before stays cordoned, with a warning. |

### Rolling upgrade

An apply that changes the `bundle` of several servers upgrades one server at a time. Release N and release N-1 can run side by side ([the bundled Postgres](/reference/postgres/index.html.md#two-releases-on-one-store), [the control channel](/reference/control-channel/index.html.md#messages-of-two-releases)). So the servers that wait keep working with the servers that moved. The order is fixed:

1. The control servers: the first server.
2. The servers with the edge role or the DNS role.
3. The nodes.

These rules keep the order:

- One upgrade runs at a time in one apply. Each server waits until the server before it is `ready` on the new version.
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

A delete of a server that never enrolled, after a failed create, changes nothing on the server and gives a warning. Otherwise it takes these steps:

1. It reads the server from the install. A server that the install does not know, or a decommissioned one, goes straight to step 4.
2. It refuses a server that holds machines, stopped ones included: `the server <id> holds <n> machines. Delete them first`. The drain that moves machines comes in phase 2.
3. On the first server, it refuses while another server of the install is not decommissioned, because the first server runs the control role. Then it calls `DecommissionServer` ([the admin API](/reference/admin-api/index.html.md#decommissionserver)).
4. It runs `systemctl disable --now hopboxd.service` on the server. If that fails, the delete still succeeds, with a warning.

The first server needs SSH with the pinned host key for each step, because the provider reaches its admin API over SSH. Another server goes out through the admin API. If the provider cannot log in to it, the decommission still happens, and a warning says that `hopboxd` did not stop.

The OVHcloud server stays rented, and its disks stay as they are, also a server that the provider ordered. To use it again, create a `hopbox_server` for it with `ovh_service_name`, which reinstalls it.

The delete of an ordered server whose setup did not finish refuses. So Terraform cannot order a second server in its place ([An order that is not delivered in time](#an-order-that-is-not-delivered-in-time)).

### Read

Each refresh reads the server from the OVHcloud API. A service name that the account no longer rents removes the resource from the state. For a server with `ovh_plan`, the refresh fails instead with `the server <name> that this resource ordered is no longer in the OVHcloud account`. Without the server in the state, the next plan orders a new server.

A create can fail before the enrollment. Its state then has no `server_id`. The refresh keeps such a state as it is, so Terraform can destroy the server and create it again.

Then the refresh reads the server from the install. It uses SSH on the first server and the admin API on the others. It sets `labels`, `pool`, and `version` from the install. On the first server, it also sets `domain` from `hopboxd admin config get`. So the plan shows a real change of the domain. If the install does not answer, the refresh keeps the state as it was, with a warning.

The install can lose a server that is not the first one, or decommission it. The refresh then removes that server from the state. For a server with `ovh_plan`, the refresh fails instead, and says how to set up the same OVHcloud server again with `ovh_service_name`. The first server never leaves the state this way. A new create reinstalls the server, and that erases the control store of the whole install. So the refresh fails with `the first server <id> is decommissioned or unknown to its install, but Terraform did not destroy it`. To accept a new install, run `terraform state rm` on the resource, and apply.

### Import

```sh
terraform import hopbox_server.first ns1234567.ip-198-51-100.eu
```

The import takes the OVHcloud service name. The refresh after it logs in with `ssh_private_key` as `debian`, then as `root`, and pins the host key. It reads the release in `/opt/hopbox/current`, the roles of `/etc/hopbox/hopboxd.toml`, and the server ID from the certificate of the server. It reads only the certificate, never the private key. A server with the role `control` imports with `bootstrap = true`, and the refresh reads its `domain` from the install.

An import of a server that the provider ordered works the same way. It cannot recover the order attributes. If the Terraform files have `ovh_plan`, the first apply after the import writes the order attributes into the state. That apply changes nothing on the server. From then on, the rules of an ordered server apply.

An import cannot recover `bundle`, `host_kernel`, or `admin_api_key`. The next plan shows an update of `bundle` from your Terraform files. If the file name of the bundle names the version that the server runs, the apply does not upgrade. The state of an imported first server has no `admin_api_key`.

### Host key trust

The provider trusts the host key of the first login after a reinstall, and of the first login of an import. This is trust on first use: no other source tells the provider the key of the new server. From then on, it accepts only the pinned key, and it asks the server for the key type of the pin.

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
| `In this release, the first server has the roles control, node, dns, edge, and every other server has the role node.` | Another list in `roles`. |
| `the sha256 "<value>" is not 64 lowercase hex digits` | A bad `sha256` in `bundle` or `host_kernel`. |
| `the source "<value>" is a URL, but only https URLs work` | A `source` with `http://` or another scheme. |
| `the provider has no OVHcloud credentials. Set OVH_APPLICATION_KEY, OVH_APPLICATION_SECRET, and OVH_CONSUMER_KEY` | The environment has no OVHcloud credentials. |
| `the provider has no SSH key. Set ssh_private_key in the provider block, or HOPBOX_SSH_PRIVATE_KEY` | No `ssh_private_key`. |
| `a server that is not the first one needs the admin API: set install_url and admin_api_key ...` | A server without `bootstrap`, and no admin API. |
| `a server that is not the first one needs the URL of the internal listener of the control role: set control_urls or control_url in the provider block, or HOPBOX_CONTROL_URL` | A server without `bootstrap`, and no control URL. |
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
| `the first server runs the control role of the install, and the servers <ids> still belong to it. Destroy them first` | A delete of the first server before the others. |

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
| `internal/tfprovider/remote` | `make test` | The host key pin, the login users, and scripts with their input and exit status, against an SSH server in the test. |
| `internal/tfprovider/hostsetup` | `make test` | The scripts of each step, and that no token or private key appears in them. |
| `TestResumeAfterADroppedUpload` | `make test-linux` | The upload scripts with a real `sh`, as root in a private mount namespace. A dropped upload goes on in the background, and a slow resume runs beside it. The resume waits for the lock, appends the rest, and the old script removes nothing. |
| `TestInstallKernelPinsTheRelease` | `make test-linux` | The kernel step of the create, as root in a private mount namespace, with stubs of `dpkg -i` and `update-grub`. The GRUB pin names the kernel release `6.18.57-hopbox1`, not the package version `6.18.57-1`. |
| `TestCreateBootstrap`, `TestCreateNode`, `TestCreateRefuses`, `TestUpdate`, `TestRemove`, `TestRead` | `make test` | The flow of the resource against the fake OVHcloud API and a fake server. |
| `TestUpgradeRank`, `TestUpgradeQueueOrder`, `TestUpgradeQueueStopsAtAFailure`, `TestRollingUpgrade`, `TestUpgradeRefusesANodeBeforeItsControlServer`, `TestUpgradeWaitsForTheNewVersion` | `make test` | The [rolling upgrade](#rolling-upgrade): one upgrade at a time in the order of the ranks, also when the update of a node starts first. The stop after a failed upgrade, the refusal of a node before its control server, and the wait for the new version in the heartbeat. |
| `TestProvisionOrders`, `TestProvisionRefusesBeforePaying`, `TestProvisionFindsALostOrder`, `TestProvisionDeletesAStaleCart`, `TestProvisionWaitsForTheDelivery`, `TestProvisionCancelledOrder`, `TestReadKeepsAnOrderedServer`, `TestReadKeepsAnOrderedNode`, `TestRemoveRefusesAPendingOrder`, `TestOrderKeyTakenForTheRun`, `TestOrderRefusesASecondCartOfTheKey`, `TestLostOrderAfterTheCartExpired` | `make test` | The order flow against the fake OVHcloud API, which sells plans from a stock, checks each cart, and can lose the answer of a checkout. Each test counts the orders, so that no path orders twice. No test calls the real OVHcloud API. |
| `TestOrderedServerLifecycle`, `TestOrderedServerResumes`, `TestPendingOrderBlocksASecondCreate`, `TestOrderFollowPlan` | `make test` | Terraform orders a server, updates it, refuses the changes that order again, imports it, and destroys it. Over the plugin protocol: a tainted order whose delete refuses, and an untaint that goes on with the same order. A second create beside a pending order refuses to order. |
| `TestPoolAndSizeLifecycle`, `TestPoolAndSizeValidation`, `TestPoolNeedsTheAdminAPI` | `make test` | Terraform creates a pool and a size against a fake admin API over Connect, imports them, refuses a change of a size that keeps its name, renames the size, and destroys both. They need the `terraform` binary as the tests below do. |
| `TestServerResourceLifecycle`, `TestServerResourceValidation` | `make test` | Terraform itself runs a create, an update, an import, and a destroy, and plans bad Terraform files. They need the `terraform` binary on `PATH`, or in `TF_ACC_TERRAFORM_PATH`. Without it, terraform-plugin-testing downloads one. |
| `TestInstallOverSSH` | `make test-provider`, and the CI job `install` | The scripts of the provider over real SSH, in a Debian 13 container with systemd and sshd. The test covers the key swap and the data dir without reflinks. It covers the refusal of a wrong kernel package, and the install and enrollment of a first server, with `dns.listen` and the public addresses. It also covers a call with the first admin API key, the labels, an import, an upgrade, and the decommission. The container runs the kernel of the host, so the test does not reboot. A real run on the [tested server](/reference/preflight/index.html.md#the-tested-server) covers the reboot. The test does not pass on an Arm Mac with Docker Desktop. Its VM has no `/dev/kvm`, so the preflight check of the install fails. Under the amd64 emulation, the system services of the container fail to start, such as sshd. Run it on a Linux host for amd64 with KVM. |
| `TestAccServer` | by hand, with `TF_ACC=1` | A create and a destroy on a real OVHcloud server that the account rents, with the credentials of the owner. No test orders a real server, because an order costs money. It erases the server. So the test skips unless `HOPBOX_ACC_ERASE` names the same server as `HOPBOX_ACC_SERVICE_NAME`. It also needs `HOPBOX_ACC_DOMAIN`, `HOPBOX_ACC_BUNDLE`, `HOPBOX_ACC_BUNDLE_SHA256`, `HOPBOX_ACC_KERNEL`, `HOPBOX_ACC_KERNEL_SHA256`, the OVHcloud variables, and `HOPBOX_SSH_PRIVATE_KEY`. |
