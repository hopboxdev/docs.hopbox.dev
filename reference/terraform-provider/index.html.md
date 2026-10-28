# hopbox Terraform provider

> The hopbox Terraform provider for any Debian 13 host over SSH, its provider block, the host that the operator sets up, the resources hopbox_server, hopbox_pool, and hopbox_size, their steps and errors, and the binary.

`terraform-provider-hopbox` is the hopbox Terraform provider. Terraform runs it as a plugin. It installs hopbox over SSH on hosts that run Debian 13, from any provider of servers, or computers that the operator owns. The operator orders each host and installs Debian 13 on it, and the provider takes it from there. It installs the tested host kernel and `hopboxd`, and enrolls the server. Then it manages the server through the admin API. An install can grow into a fleet: edge servers with floating addresses, more control servers, and nodes in pools. The provider reaches a server that is not an edge server through a jump host. The provider calls no API of a provider of servers (plan question Q90). The code is in `internal/tfprovider`. [Install hopbox on your servers with Terraform](/guide/install-with-terraform/index.html.md) walks through a first install.

The provider has three resources, [`hopbox_server`](#resource-hopbox_server), [`hopbox_pool`](#resource-hopbox_pool), and [`hopbox_size`](#resource-hopbox_size), and no data sources.

## Provider address

| Item | Value |
| --- | --- |
| Source address | `registry.terraform.io/hopboxdev/hopbox` |
| Short source | `hopboxdev/hopbox` |
| Local name | `hopbox` |
| Plugin protocol | 6 |

The provider is not in a registry yet. v0.2.0 is the first release with this provider. Until the hopbox team tags it, build the release files from a checkout. The release files hold a zip of the provider for macOS on Apple Silicon and for Linux on x86_64, such as `terraform-provider-hopbox_0.2.0_darwin_arm64.zip`, in the packed layout of a filesystem mirror. Put the zip in `~/.terraform.d/plugins/registry.terraform.io/hopboxdev/hopbox/`, and set `version = "0.2.0"` for the provider in `required_providers`. Terraform reads that dir as a local mirror only without a `provider_installation` block in its CLI configuration. With such a block, add a `filesystem_mirror` block for the dir ([Install hopbox on your servers with Terraform](/guide/install-with-terraform/index.html.md#3-write-the-terraform-files)).

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

The provider block of a fleet, after its first server runs:

```hcl
provider "hopbox" {
  ssh_private_key = file("~/.ssh/id_ed25519")
  install_url     = "https://example.com"

  ssh_jump_host = {
    address  = "203.0.113.11"
    host_key = "ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAI..."
  }
}
```

Every attribute is optional. An attribute that you leave out takes the value of its environment variable.

| Attribute | Type | Environment variable | Description |
| --- | --- | --- | --- |
| `ssh_private_key` | string, sensitive | `HOPBOX_SSH_PRIVATE_KEY` | The SSH private key of the operator, in the OpenSSH or PEM format. A key with a passphrase does not work. The operator puts its public key in the `authorized_keys` of the login user of each host before the create ([The host](#the-host)). The provider logs in with it for every step. A create, an update, a delete, and an import need it. |
| `install_url` | string | `HOPBOX_INSTALL_URL` | The URL of the install, such as `https://example.com`. The provider calls the admin API there with `admin_api_key`. It must be an `https` URL. Only a loopback address can use `http`. |
| `admin_api_key` | string, sensitive | `HOPBOX_ADMIN_API_KEY` | An admin API key, `hbxadm_...` ([CLI tokens, API keys, admin API keys, and SSH keys](/reference/credentials/index.html.md)). The provider sends it as `Authorization: Bearer <key>`. |
| `control_url` | string | `HOPBOX_CONTROL_URL` | The URL of the internal listener of the control role on its peer address, such as `https://203.0.113.10:7443`. A new server that is not the first one enrolls there, as `channel.control_urls` ([the bootstrap config of hopboxd](/reference/hopboxd-config/index.html.md#keys)). It must be an `https` URL. Not with `control_urls`. Leave both out to take the control URLs from the install. |
| `control_urls` | list of strings | none | The URLs of the internal listeners of the control servers, such as `["https://203.0.113.10:7443", "https://203.0.113.13:7443"]`. A new server that is not the first one enrolls at the first one that answers, and keeps its control channel to one of them. Each URL must be an `https` URL. Not with `control_url`: both fail with `Two control URL settings` and the detail `Set control_urls or control_url, not both.` |
| `ssh_jump_host` | object | none | An edge server as the jump host of the provider ([Jump host](#jump-host)). Every SSH step of a server that is not an edge server goes through it. The table below lists its attributes. |

The object `ssh_jump_host` has these attributes:

| Attribute | Type | Default | Description |
| --- | --- | --- | --- |
| `address` | string | none, required | The address of the jump host, as a host or `host:port`, such as `203.0.113.11`. The port defaults to 22. |
| `user` | string | none | The login user on the jump host, such as `debian`. Without it, the provider tries `root`, then `debian`. |
| `host_key` | string | none, required | The SSH host key of the jump host, in the `authorized_keys` format. Copy the `ssh_host_key` of the `hopbox_server` of the edge server. The provider never trusts a jump host on its first use. |

The servers that are not the first one need `install_url` and `admin_api_key`. A `hopbox_pool` and a `hopbox_size` need them too. The first server needs none of them: the provider reaches its admin API over SSH. The admin API is public behind the edge role, so `install_url` can be `https://<domain>` ([the admin API](/reference/admin-api/index.html.md#where-the-admin-api-is-served)).

A server that is not the first one enrolls at the control URLs of `control_urls` or `control_url`. Without them, the provider takes the URLs from the install: the `internal_url` of each live control process of `ListControlProcesses` ([the admin API](/reference/admin-api/index.html.md#listcontrolprocesses)). It keeps only a URL whose address is a peer address, where the other servers reach it ([Enrollment and the install PKI](/reference/enrollment/index.html.md#peer-addresses)). A listener on loopback serves only its own server. A first server from before peer addresses listens on loopback until its update moves the listener ([The internal listener](#the-internal-listener)). That update can run in the same apply, so the create of a new server waits up to 5 minutes for a listener on a peer address. Then it fails with `a server that is not the first one enrolls at the internal listener of a control server on its peer address, and the install lists none`.

## Resource hopbox_server

A `hopbox_server` is one server of an install, on a host that the operator set up ([The host](#the-host)). In the example, the paths name the release files in the folder of the configuration. A build from a checkout uses `bin/bundle/hopbox-v0.2.0-linux-amd64.tar.gz` and `hostkernel-build/linux-image-6.18.57-hopbox1_6.18.57-1_amd64.deb` instead.

```hcl
resource "hopbox_server" "first" {
  address      = "203.0.113.10"
  ssh_host_key = "ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIFirst..."
  bootstrap    = true
  domain       = "example.com"
  labels       = { zone = "zone1" }

  bundle = {
    source = "hopbox-v0.2.0-linux-amd64.tar.gz"
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

output "admin_api_key" {
  value     = hopbox_server.first.admin_api_key
  sensitive = true
}
```

Servers of a fleet, beside the first server. The edge server holds a floating address, the control server shares the control store, and the node joins the pool `big`:

```hcl
resource "hopbox_server" "edge2" {
  address            = "203.0.113.11"
  ssh_host_key       = "ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIEdge2..."
  roles              = ["edge", "dns"]
  floating_addresses = ["198.51.100.6"]

  bundle      = local.bundle
  host_kernel = local.host_kernel

  lifecycle {
    create_before_destroy = true
  }
}

resource "hopbox_server" "cp2" {
  address      = "203.0.113.13"
  ssh_host_key = "ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAICp2..."
  roles        = ["control", "node"]

  copy_control_from = {
    address  = hopbox_server.first.address
    host_key = hopbox_server.first.ssh_host_key
    user     = hopbox_server.first.ssh_user
  }

  bundle      = local.bundle
  host_kernel = local.host_kernel
}

resource "hopbox_server" "node1" {
  address      = "203.0.113.21"
  ssh_host_key = "ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAINode1..."
  pool         = hopbox_pool.big.name

  bundle      = local.bundle
  host_kernel = local.host_kernel
}
```

`prevent_destroy` makes Terraform refuse each plan that destroys the first server, whose destroy ends the install. The provider also refuses a plan that replaces the first server ([Update](#update)).

### Arguments

| Attribute | Type | Default | Change | Description |
| --- | --- | --- | --- | --- |
| `address` | string | none, required | replaces | The peer address of the server: one public IPv4 address on its uplink, with no NAT, such as `203.0.113.12` ([Enrollment and the install PKI](/reference/enrollment/index.html.md#peer-addresses)). The other servers reach the server there, and the provider logs in there, port 22. A private, loopback, link-local, shared, or multicast address fails with `Invalid address`, as does an address in the mesh range or the machine ranges. A change replaces the server. The plan refuses a change of the first server. |
| `ssh_host_key` | string | none, required | refused | The SSH host key of the server, in the `authorized_keys` format, such as `ssh-ed25519 AAAA...`. Copy it from the console of the provider of the server (plan question Q95). The provider pins it before its first login, and refuses any other key ([Host key trust](#host-key-trust)). A comment after the key changes nothing. A plan with another key fails with `The SSH host key of a server cannot change`, unless it replaces the server. |
| `ssh_user` | string | `root` | in place, changes nothing on the server | The login user of the provider: `root`, or a user with `sudo` without a password, such as `debian` on ks. When it is not `root`, each script runs with `sudo -n`. |
| `bootstrap` | bool | `false` | replaces | True for the first server of a new install. It starts with the roles `control`, `node`, `dns`, and `edge`. It needs `domain`, and the provider makes the first admin API key for it. |
| `domain` | string | none | in place | The domain of the install, such as `example.com` ([Domain rules](/reference/domain-rules/index.html.md#install-domain)). Only the first server takes it, and it needs it. A change runs `hopboxd admin config set --domain` on the server. |
| `pool` | string | `default` | replaces | The pool that the server joins: `default`, or the name of a pool, such as `hopbox_pool.big.name` ([Resource hopbox_pool](#resource-hopbox_pool)). It must be a lowercase DNS label. The first server joins `default`. A server whose hardware differs from the first server of its pool fails to enroll ([Enrollment and the install PKI](/reference/enrollment/index.html.md#hardware-of-a-pool)). |
| `labels` | map of string | `{}` | in place | The labels of the server, such as `zone = "zone1"`. Placement filters use them. The rules of `channel.labels` apply ([the bootstrap config of hopboxd](/reference/hopboxd-config/index.html.md#keys)). A server has at most 64 labels. A key has 1 to 63 characters of `a-z`, `0-9`, `.`, `_`, and `-`, and starts with a letter. A value has at most 255 bytes. The provider sets them with `SetServerLabels`, so the install ignores the labels of the bootstrap config of the server ([the admin API](/reference/admin-api/index.html.md#setserverlabels)). |
| `roles` | list of string | `["control", "node", "dns", "edge"]` with `bootstrap`, else `["node"]` | updates in place | The roles of the server, in any order. The first server starts with every role, so leave it out at its create. Another server has one or more of `control`, `node`, `dns`, and `edge`, each once. `dns` needs `edge`, because the DNS role runs on the edge servers. `control` needs another role too, such as `["control", "node"]`, so that the server enrolls and the registry knows it ([Control servers](#control-servers)). A plan that drops `edge`, `dns`, or `node` from a running server removes the role in place, also on the first server. A plan that adds `node` back to a server whose node role a removal took adds it in place ([Roles of a running server](#roles-of-a-running-server)). The role `control` never goes. A refresh reads the roles that the server has: `control` from its bootstrap config or the state, and the others from the registry of the install. |
| `copy_control_from` | object | none | in place, changes nothing | The live control server whose keys and external Postgres a new control server copies before its install, with its pinned host key. A control server that is not the first one needs it, and no other server takes it. Only the create reads it. See [Control servers](#control-servers). |
| `floating_addresses` | set of string | none | in place | The floating IPv4 addresses of an edge server, such as `["198.51.100.6"]` (rule 4). Each one is one public IPv4 address that the operator routes to the server at its provider. Only a server with the role `edge` takes them, and the peer address of the server is none of them. A change puts the new addresses on the uplink of the server, and removes the old ones. See [Edge servers and floating addresses](#edge-servers-and-floating-addresses). |
| `public_ipv6` | string | none | refused | The floating IPv6 address of an edge server, without a prefix length, such as `2001:db8::12` ([the edge role](/reference/edge/index.html.md#the-public-ipv6-address)). The operator routes it to the server at its provider. The network setup of the provider puts it on the uplink of the server as a `/128` ([The network setup of an edge server](#the-network-setup-of-an-edge-server)). The provider writes it to `edge.public_ipv6` of the bootstrap config. On a server with the DNS role, it adds port 53 of the address to `dns.listen` ([Create](#create)). The zone names it in its AAAA records. The IPv6 egress of the machines never leaves from it. Only a server with the role `edge` takes it. The rules of `edge.public_ipv6` apply ([the bootstrap config of hopboxd](/reference/hopboxd-config/index.html.md#keys)). Give each edge server its own address: a second resource with the same address fails the plan with `two hopbox_server resources have the public IPv6 address <address>`. The plan refuses a change in place with `The public IPv6 address of a server cannot change`. A replacement with the same value takes the same address. See [Edge servers and floating addresses](#edge-servers-and-floating-addresses). |
| `bundle` | object | none, required | in place | The release bundle that the server runs ([Release bundle](/reference/release-bundle/index.html.md)), with `source` and `sha256`. A change of `sha256` upgrades `hopboxd` on the server. A change of `source` alone changes nothing on the server. |
| `bundle.source` | string | none, required | | A local path, or an `https` URL that the server downloads with `curl`. The file name must be `hopbox-<version>-linux-amd64.tar.gz`. |
| `bundle.sha256` | string | none, required | | The sha256 of the bundle, in 64 lowercase hex digits, from the `.sha256` file of `make bundle`. |
| `host_kernel` | object | none, required | in place | The Debian package of the tested host kernel, `linux-image-6.18.57-hopbox1_6.18.57-1_amd64.deb` from `build/hostkernel/build.sh` ([Host kernel](/reference/host-kernel/index.html.md)), with `source` and `sha256` as for `bundle`. The provider installs it only on a server that does not run the tested kernel. |

### Attributes the provider sets

| Attribute | Type | Description |
| --- | --- | --- |
| `id` | string | The peer address of the server, as `address`. |
| `server_id` | string | The ID of the server in the install, a UUID. |
| `admin_api_key` | string, sensitive | Only on the first server: the first admin API key of the install, `hbxadm_` and 43 characters. The provider makes it and passes only its SHA-256 to the install ([Server layout](/reference/server-layout/index.html.md#the-bootstrap-admin-api-key)). The state holds its only copy. The install registers it with the name `bootstrap` and a lifetime of 90 days. |
| `version` | string | The version of `hopboxd` on the server, such as `v0.1.0`. |
| `internal_url` | string | On a control server, the URL of its internal listener on its peer address, such as `https://203.0.113.10:7443` ([the internal listener](/reference/internal-listener/index.html.md)). The other servers enroll there. A refresh reads it from the live control process of the server. Null on a server without the role `control`. See [The internal listener](#the-internal-listener). |

### The host

The operator orders each host at its provider and sets it up, before the create of its `hopbox_server` (plan questions Q95 and Q98):

- Debian 13 on `amd64`, with KVM.
- The peer address on its own uplink, with no NAT in front of it. An edge server also holds its floating addresses at the provider.
- SSH for `root`, or for a user with `sudo` without a password, with the public key of `ssh_private_key` in its `authorized_keys`.
- An `sshd` that takes keys alone: `sshd -T` prints `passwordauthentication no`, and `kbdinteractiveauthentication no`, for a login of `root` and of `ssh_user`. The SSH port of a new host faces the internet before `hopboxd` and its host firewall run.
- For a server with the role `node`, a resolver that the machines reach, in `/run/systemd/resolve/resolv.conf` or `/etc/resolv.conf`. A resolver on loopback alone, such as a local `unbound`, does not count.
- `/var/lib/hopbox` as an XFS file system with reflinks, on a partition of its own ([Install hopbox on a Debian 13 server by hand](/guide/install-debian/index.html.md#2-make-the-xfs-file-system)).
- The SSH host key of the host, from the console of the provider, for `ssh_host_key`.

[Install hopbox on your servers with Terraform](/guide/install-with-terraform/index.html.md#1-set-up-the-hosts) shows each step.

### The tested OS and kernel

The OS and the kernel are constants of the provider. You cannot change them. hopbox tests one combination.

| Constant | Value |
| --- | --- |
| OS | Debian 13 on `amd64` |
| Host kernel release | `6.18.57-hopbox1` |
| Host kernel package | `linux-image-6.18.57-hopbox1`, version `6.18.57-1`, for `amd64` |

### Create

A create takes these steps. Steps 1 to 8 change nothing on the host but the upload of the bundle, which a refusal of step 8 removes. So a refusal there leaves the host as it was, and Terraform keeps no state of the server:

1. It reads each local file of `bundle` and `host_kernel`, and checks its sha256. A missing file or another sha256 stops the create before its first login.
2. It makes the checks that need the install. A server without the role `edge` needs `ssh_jump_host` ([Jump host](#jump-host)). No other resource has a floating address of this server. The plan checks this first. A server that is not the first one needs the control URLs, from the provider block or from the install ([Provider block](#provider-block)). A control server that is not the first one needs `copy_control_from`. The provider reads the files of that live control server, and checks the fingerprints of its keys ([Control servers](#control-servers)).
3. It logs in at `address` as `ssh_user`, with `ssh_private_key`, and accepts only the host key of `ssh_host_key`. A server that is not an edge server gets this login through the jump host. Another host key fails with `log in to the server at <address>: remote: the host key of the server is not the pinned one: ... Compare ssh_host_key with the host key in the console of the provider of the server`.
4. It runs `sshd -T -C user=<user>,host=192.0.2.1,addr=192.0.2.1,laddr=<address>,lport=22` for `root`, and for `ssh_user` when it is another user. With `-C`, `sshd` applies the `Match` blocks of a login of that user from an internet address. `192.0.2.1` stands for any such address. A `Match` block for a range of addresses, such as an office network, does not count. An `sshd` that prints `passwordauthentication` or `kbdinteractiveauthentication` with a value other than `no` fails the create with the fix: `sshd of the server takes password logins: ... Put PasswordAuthentication no and KbdInteractiveAuthentication no in /etc/ssh/sshd_config.d/10-hopbox.conf on the server, run systemctl reload ssh, and apply again`.
5. It refuses a host that belongs to an install, so a wrong address never erases a live server:
   - A host where `hopboxd.service` is active, or a `hopboxd` process runs, fails with `the host holds an install: hopboxd runs on the host at <address>, ...`.
   - A host with the identity of an enrolled server, `<data_dir>/server/identity.pem`, fails with `the host holds an install: the host at <address> holds the identity of an enrolled server in <data_dir>/server, ...`. A destroy of its `hopbox_server` removes the identity ([Delete](#delete)).
   - For the first server, a host with the control store of an earlier install, `<data_dir>/pg/data/PG_VERSION`, fails with `the host holds an install: the host at <address> holds the control store of an earlier install in <data_dir>/pg, ...`. Move or remove `<data_dir>/pg` and `<data_dir>/kek` first.

   `<data_dir>` is `data_dir` of `/etc/hopbox/hopboxd.toml` of an earlier install, or `/var/lib/hopbox`. The provider reads the file with a TOML parser. A `data_dir` that is not a clean absolute path without spaces or quotes fails with `the bootstrap config of the server: the path ... is not a clean absolute path without spaces or quotes`.
6. It checks that the host runs Debian 13 on `amd64`, and that a network interface holds `address`. A host without the address fails with `the server at <address> holds no address <address> on a network interface, so a NAT sits in front of it, or address names another server`. For a server with the role `node`, it reads `/run/systemd/resolve/resolv.conf` and `/etc/resolv.conf`, the files from which the node role takes the resolvers of the machines ([the node network](/reference/node-network/index.html.md#the-resolvers-of-the-machines)). A host whose files name no IPv4 resolver outside loopback, link-local, and the machine range fails with `the host at <address> names no resolver that a machine can reach in ... Put a nameserver line with a resolver that the host reaches over its uplink in /etc/resolv.conf, and apply again`.
7. It checks that `/var/lib/hopbox` is a file system of its own, on XFS with reflinks, or an empty XFS without them. A data dir that is not a file system of its own, or that is not XFS, fails with the fix: `Make it an XFS file system with reflinks on a partition of its own before the create: mkfs.xfs -m reflink=1 <partition>, a line for /var/lib/hopbox in /etc/fstab, and mount /var/lib/hopbox. mkfs.xfs erases the partition`. An XFS without reflinks that holds files fails too.
8. It puts the bundle in `/var/lib/hopbox-provider` and checks its sha256 there ([Uploads](#uploads)). On an edge server with `floating_addresses` or `public_ipv6`, it reads the features of the `hopboxd` of the bundle with `hopboxd version --json`. A server with `public_ipv6` and a bundle without the feature `floating-ipv6` fails with `public_ipv6 needs a release of hopboxd whose version names the feature floating-ipv6, ...`. The provider then removes the bundle.
9. From this step on, the create changes the host, so a failure keeps the resource in the state, and Terraform marks it as tainted. If the XFS has no reflinks and holds no file, the provider makes the file system again. It runs `mkfs.xfs -m reflink=1` with the same UUID, and the apply shows a warning. A file system with files is never made again.
10. On an edge server with `floating_addresses` or `public_ipv6`, it writes the network setup of the provider, and runs it ([The network setup of an edge server](#the-network-setup-of-an-edge-server)).
11. If the server does not run `6.18.57-hopbox1`, it installs the host kernel. It puts the package on the server ([Uploads](#uploads)), checks its sha256, and checks that the package is `linux-image-6.18.57-hopbox1` version `6.18.57-1` for `amd64`. It runs `dpkg -i`, pins the GRUB default to the new kernel in `/etc/default/grub.d/20_hopbox_kernel.cfg`, and checks `/boot/grub/grub.cfg`, as `build/hostkernel/install.sh` does. Then it reboots the server from a systemd timer. It logs in again with the pinned host key until the boot ID changes, for at most 20 minutes. It checks that the new boot runs the tested kernel.
12. On a control server that is not the first one, it writes the key files and the files of the external Postgres that step 2 read ([Control servers](#control-servers)).
13. It writes the bootstrap config `/etc/hopbox/hopboxd.toml`, also on a host where an earlier install left one. `hopboxd install` keeps its keys ([the bootstrap config of hopboxd](/reference/hopboxd-config/index.html.md#keys)). The file holds the roles and these keys:

    | Key | Server | Value |
    | --- | --- | --- |
    | `dns.listen` | a server with the DNS role | Port 53 of `address`, of each address of `floating_addresses`, and of the address of `public_ipv6`, IPv4 first. The tested OS runs `systemd-resolved`, which holds port 53 on `127.0.0.53` and `127.0.0.54`. So the default `dns.listen` of every address fails to bind ([the DNS role](/reference/dns/index.html.md)). The network setup of step 10 puts the address of `public_ipv6` on the uplink before the install. The DNS role binds each address with `IP_FREEBIND`, also one that the server does not hold yet ([the DNS role](/reference/dns/index.html.md#listeners)). |
    | `edge.public_ipv6` | an edge server with `public_ipv6` | The value of `public_ipv6`. |
    | `edge.public_ipv4` | an edge server with `floating_addresses`, on a release with the egress tunnels | The first address of `floating_addresses` in sorted order. The egress of the machines leaves from it ([the egress of the machines](/reference/egress/index.html.md#the-source-addresses)). The provider reads the features of the `hopboxd` of the bundle first, and writes the key only when they name `egress` ([Releases with and without the key](#releases-with-and-without-the-key)). |
    | `edge.floating_addresses` | an edge server with `floating_addresses`, on a release with the feature `floating-addresses` | Every address of `floating_addresses` in sorted order. `hopboxd` puts each one back on the uplink when it goes missing ([the edge role](/reference/edge/index.html.md#the-check-of-the-uplink)). The provider writes the key only when the features of the bundle name `floating-addresses`. |
    | `control.internal_listen` | a control server, the first server too | Port 7443 of `address`, such as `203.0.113.10:7443` ([the internal listener](/reference/internal-listener/index.html.md)). |
    | `channel.control_urls` | the first server | The internal listener of the server itself, such as `["https://203.0.113.10:7443"]`, because its listener does not listen on loopback. |
    | `postgres.url`, `postgres.ca_file`, `postgres.password_file` | a control server that is not the first one | The `postgres.url` of the live control server of step 2, `/etc/hopbox/postgres-ca.pem`, and `/etc/hopbox/postgres-password`. |

    For example, the first server at `203.0.113.10` without floating addresses gets this file:

    ```toml
    # Written by the hopbox Terraform provider before the first hopboxd install.
    roles = ["control", "node", "dns", "edge"]

    [control]
    internal_listen = "203.0.113.10:7443"

    [channel]
    control_urls = ["https://203.0.113.10:7443"]

    [dns]
    listen = ["203.0.113.10:53"]
    ```

14. On the first server, it makes the admin API key and saves it in the state. On another server, it makes an enrollment token through the admin API, with an `Idempotency-Key` header. The token is for the pool of the server and its roles that enroll: every role but `control`. It names `address` as its `peer_address`, so the enrollment must come from that address, and the server gets it as its peer address ([the admin API](/reference/admin-api/index.html.md#createenrollmenttoken), plan question Q92). The token works for 1 hour. While it works, its address is in the set `registered` of each control server, whose host firewall then lets the enrollment in. The control role reads the set every second, so the provider waits 3 s before the install ([the host firewall](/reference/host-firewall/index.html.md#sets)).
15. It unpacks the bundle and runs its `hopboxd install --json` with these flags ([hopboxd](/reference/hopboxd/index.html.md#install)):

    | Server | Flags |
    | --- | --- |
    | First server | `--roles control,node,dns,edge --domain <domain> --bootstrap-admin-key-hash <sha256 of the key>` |
    | Other server | `--roles <roles>`, one `--control-url <url>` for each control URL of step 2, and the enrollment token in `HOPBOX_ENROLLMENT_TOKEN` |

    The install runs in a transient systemd unit, under `systemd-run --wait`, so a dropped SSH session does not stop it. The token reaches the unit through a file of mode 0600 in `/var/lib/hopbox-provider`, never through a command line. The provider removes the bundle, the unpacked release, and the token file after the install.
16. On the first server, it makes a token with `hopboxd admin enrollment-token create --pool <pool> --roles node,dns,edge`. It gives the token to `hopboxd install --roles control,node,dns,edge --json --peer-address <address>`. So the node role, the DNS role, and the edge role enroll with the control role of the same server ([the DNS role](/reference/dns/index.html.md), [the edge role](/reference/edge/index.html.md)), and the server gets its peer address. The token never leaves the server.
17. It waits until the server is `ready`, for at most 5 minutes. On the first server, it reads `hopboxd admin server get --json` over SSH. On another server, it calls `GetServer` of the admin API. A server that is ready but `degraded` gives a warning with its problems.
18. It sets the labels with `SetServerLabels`, through `hopboxd admin server set-labels` on the first server.
19. On the first server, it sets the public addresses of the install with `hopboxd admin config set --public-address`: the floating IPv4 addresses of the first server, or its `address` without one, and the address of `public_ipv6`. The zone names no other IPv6 address, such as an address of the uplink, because an AAAA record names only a floating address (plan question Q94). The zone of the DNS role then points the domain, `ns1`, `ns2`, and every machine host at the server ([the DNS role](/reference/dns/index.html.md#the-zone-of-the-install)). The install runs at this step. So a failure gives the warning `The provider did not set the public addresses of the install`, with the command to run on the server, and the create succeeds. A failed create taints the server, and the next apply then destroys the install.

The state keeps what each step learned. If step 9 or a later step fails, Terraform marks the resource as tainted. The next apply destroys it, which undoes the steps that ran ([Delete](#delete)), and creates it again. A failed preflight check of severity `warn`, such as a kernel that is not the tested one, gives a warning on the apply.

### The network setup of an edge server

An edge server with `floating_addresses` or `public_ipv6` gets the network setup of the provider (rule 4). The operator routes each floating address to the server at its provider first. The provider never moves or parks a floating address, and calls no API of a provider. The setup takes these steps:

1. It writes the boot script `/usr/local/sbin/hopbox-network` and the unit `hopbox-network.service`, and runs the unit.
2. The script finds the uplink, the interface of the IPv4 default route. It waits up to 60 seconds for the route, because the network setup of the OS can come after the unit.
3. It gives the uplink each floating address as a `/32`, and the address of `public_ipv6` as a `/128`. `hopboxd` never takes that IPv6 address as the source of the NAT66 of the machines.
4. It sets `src <address>` on the default route of the uplink, with the gateway, the protocol, the metric, and the flags of the route as they were. So the own connections of the edge server leave from its peer address, whatever the order of the addresses on the uplink. These include its connections to port 7443 of the control servers, its SSH as the jump host to port 22 of the other servers, and its mesh. The host firewall of each other server takes them only from peer addresses ([the host firewall](/reference/host-firewall/index.html.md)).

The unit runs at each boot, after `systemd-networkd.service` and `networking.service`, and before `network-online.target` and `hopboxd.service`. A server without floating addresses and without `public_ipv6` gets no network setup, and the provider leaves its network as the operator set it.

The script takes two commands. `hopbox-network drop <address>...` removes addresses from the uplink. `hopbox-network undo` removes the floating addresses of the script, its IPv6 address, and the source of the default route. A delete runs `undo`, and then removes the script and the unit ([Delete](#delete)).

### Jump host

The [host firewall](/reference/host-firewall/index.html.md) keeps SSH open on the internet only on the edge servers (rule 5). Port 22 of the public address of a server without the role `edge` takes only the peer addresses of the edge servers, the set `jump_hosts`. Until the server knew an edge server, that port takes every source ([the host firewall](/reference/host-firewall/index.html.md#never-a-lockout)). So every SSH step of a server without the role `edge` goes through the jump host of `ssh_jump_host`, as `ssh -J` does, to the peer address of the server. Each edge server gets its SSH straight, the first server too while it has the role `edge`.

A create or an update of a server without the role `edge` fails without `ssh_jump_host`, before any step: `the server <address> has no role edge, so its host firewall takes SSH only from the edge servers once hopboxd runs, ...`.

The jump host must be an edge server of the install. It dials the server from its peer address, the source of its default route ([The network setup of an edge server](#the-network-setup-of-an-edge-server)). The session to the server runs inside the connection that the jump host opens, so the jump host sees only encrypted bytes. The pin of the host key of the server holds through the jump host. The jump host has its own pin, `ssh_jump_host.host_key`. A jump host that shows another key fails each step with `remote: the host key of the jump host is not the pinned one`. The provider logs in to the jump host with `ssh_private_key`, so that key must be authorized on the jump host too. sshd of the jump host must allow TCP forwarding, as the default of Debian 13 does.

### Edge servers and floating addresses

An edge server takes the public traffic of the install. The first server has the role `edge`. Another server gets it with `roles = ["edge"]`, or `["edge", "dns"]` to answer DNS too.

The floating addresses of an edge server are public IPv4 addresses that its provider can move between servers, such as the Additional IPs of OVHcloud or the floating IPs of other providers. The operator routes each one to the edge server at the provider, and moves it to another edge server there. When an edge server goes, its replacement takes over the same addresses, so the public addresses of the install never change. The provider takes them as configuration alone:

- The create puts them on the uplink of the server ([The network setup of an edge server](#the-network-setup-of-an-edge-server)), and in `dns.listen` of a server with the DNS role.
- An update of `floating_addresses` gives `dns.listen` the new list first, then writes the network setup again with the new list. Then it writes the new keys to the bootstrap config, and waits for the restart of `hopboxd`. Only then does it remove each dropped address from the uplink. `hopboxd` puts back each floating address of its config that the uplink lost, so a removal before the new keys does not hold.
- A refresh reads the floating addresses that the server holds on a network interface, over SSH. An address that the server lost leaves the state, so the next plan shows an update, which puts it back on the uplink. A refresh never changes the server.
- A delete removes the floating addresses from the uplink.

To replace an edge server without a gap, give its resource `lifecycle { create_before_destroy = true }`, and give the new server another `address`. Terraform then creates the new server before it destroys the old one. Route each floating address to the new server at the provider once the new server is ready. Both servers hold the addresses on their uplinks in between. The provider of the servers routes each address to one of them, so one server answers on it.

On a server with the DNS role, the update of `floating_addresses` runs `hopboxd install --dns-listen` with the roles of the server ([hopboxd](/reference/hopboxd/index.html.md#install)). The list holds port 53 of `address`, of each floating address, and of the address of `public_ipv6`. When the list changes, `hopboxd` restarts, and the update waits until the server is ready. A `hopboxd` of an earlier release refuses `--dns-listen`. The update then fails before the network changes, and the error says to upgrade the server with a bundle of this release first.

The egress of the machines leaves an edge server from one floating address, `edge.public_ipv4` of the bootstrap config ([the egress of the machines](/reference/egress/index.html.md#the-source-addresses)). The provider picks the first address of `floating_addresses` in sorted order, as the owner decided on 2026-10-27 (plan question Q82). The create writes it to the bootstrap config. An update of `floating_addresses` can change that address. Then the update sets the key after the server holds the new addresses, and before it removes the dropped ones. It runs `hopboxd install --edge-public-ipv4` with the roles of the server ([hopboxd](/reference/hopboxd/index.html.md#install)). On a release with the feature `floating-addresses`, the same install gives `edge.floating_addresses` the whole list with `--edge-floating-address`. An update that drops every floating address removes both keys with `--edge-no-floating` on such a release. On an older release, it removes `edge.public_ipv4` from the file, and the running `hopboxd` keeps the old value until its next start. When a key changes, `hopboxd` restarts, and the update waits until the server is ready. A `hopboxd` of an earlier release refuses `--edge-public-ipv4`, and the error says to upgrade the server with a bundle of this release first. An update of `floating_addresses` on a release without the feature `egress` sets no key.

Two resources that list the same address would each put it on their uplink. So the plan of each resource claims its floating addresses. Terraform plans every resource of the configuration, also a resource that does not change. A second resource with the same address fails the plan with `two hopbox_server resources have the floating address <address>`, before any apply.

Each edge server can also hold one floating IPv6 address (plan question Q94). Give it `public_ipv6`, without a prefix length. The operator routes the address to the server at its provider. The provider writes it to the bootstrap config of the server at its create ([the edge role](/reference/edge/index.html.md#the-public-ipv6-address)), and the network setup puts it on the uplink as a `/128`. `hopboxd` puts it back when no interface holds it ([the edge role](/reference/edge/index.html.md#the-check-of-the-uplink)). The provider never puts an IPv6 address of the uplink in `edge.public_ipv6` or in the public addresses of the install. A replacement of the edge server with the same value takes the same address, so the AAAA records of the zone never change. Each resource claims its address in the plan, as for the floating addresses. The IPv6 egress of the machines never leaves from this address. It leaves from a global address of the uplink ([the host firewall](/reference/host-firewall/index.html.md#the-uplink-and-the-nat66)).

The zone of the DNS role names the public addresses of the install, which an instance admin sets with `hopboxd admin config set --public-address` ([the DNS role](/reference/dns/index.html.md#the-zone-of-the-install)). The provider sets them only on the first server, at its create. To point the zone at the edge servers, set them to the floating addresses of the edge servers ([Install hopbox on your servers with Terraform](/guide/install-with-terraform/index.html.md#5-point-the-zone-at-the-edge-servers)).

### Releases with and without the key

A release from before the egress tunnels refuses `edge.public_ipv4` in the bootstrap config, with `unknown keys`. So the provider writes the key only for a release whose `hopboxd version --json` names the feature `egress` ([the version command](/reference/version/index.html.md#json-output)). It reads the features of the `hopboxd` of a bundle on the server before the install. The key follows the release of each edge server with `floating_addresses`:

| Step | What the provider does with the key |
| --- | --- |
| Create | It writes the key only for a bundle with the feature. |
| Upgrade to a release with the feature | After the upgrade, it sets the key with `hopboxd install --edge-public-ipv4`, and `edge.floating_addresses` on a release with the feature `floating-addresses`. An edge server made before the egress tunnels gets its key this way. When the key changes, `hopboxd` restarts, and the update waits until the server is ready. |
| Move to a release without the feature | Before the upgrade, it removes the key and `edge.floating_addresses` from the bootstrap config, so the older release starts. The edge server then masquerades the egress to the address of its output interface. If the upgrade fails, the provider puts the key back for the release that keeps running. When that restarts `hopboxd`, the apply waits until the server is ready again, as after the other restarts, and then fails with the error of the upgrade. After the upgrade, it turns off the IPv6 forwarding that the egress turned on, because the older release keeps no rule that guards it. It does so only when `<data_dir>/hostfw/forward-guard-ipv6` records that `hopboxd` turned it on, and then removes the file. On a server without the role `node`, it also turns off the IPv4 forwarding, only when `<data_dir>/hostfw/forward-guard` exists, and removes that file. So forwarding that something else turned on before `hopboxd`, such as Docker, stays on. An edge server that ran a build of `main` with the egress from before this record has no `forward-guard-ipv6`, so its IPv6 forwarding stays on after the move back. On such a server, run `sudo sysctl -w net.ipv6.conf.all.forwarding=0` by hand, unless another tool on the server needs it. A server with the role `node` keeps IPv4 forwarding, because its node network needs it for its machines. If this step fails, the apply succeeds with a warning that names the files to look for and the `sysctl` commands to run on the server. |

A release without the feature `floating-addresses` refuses `edge.floating_addresses` in the same way. Before a move to such a release, the provider removes that key alone. If the upgrade fails, it puts the key back for the release that keeps running. After a move to a release with the feature, it sets the key. Before any upgrade of an edge server without floating addresses, the provider removes `edge.public_ipv4` and `edge.floating_addresses`, such as a key that an older provider left. Otherwise a release with the feature takes the old address back on the uplink at its start.

### The internal listener

The internal listener of each control server listens on port 7443 of its peer address ([the internal listener](/reference/internal-listener/index.html.md)). The other servers enroll and keep their control channel there. The create writes `control.internal_listen` for each control server, the first server too, so `internal_url` is `https://<address>:7443` from the start.

A first server from before peer addresses, such as ks, has no peer address in its install, and its listener listens on loopback. The plan of such a server shows `internal_url` going from `https://127.0.0.1:7443` to `https://<address>:7443`. Its update takes these steps, after an upgrade of the same apply:

1. If the install records no peer address for the server, it runs `hopboxd admin server set-peer-address <server_id> <address>` on the server ([hopboxd](/reference/hopboxd/index.html.md#admin-server-set-peer-address)). If the install records another address, the update fails with `the install records the peer address <peer address> for the first server <id>, not its address <address>`.
2. It reads the live control process of the server. If its `internal_url` is the URL of the peer address, the update stops here.
3. It checks that `hopboxd version --json` names the feature `peer-address`. A release without it fails the update with `the hopboxd of the server <id> runs no internal listener on a peer address: ...`. Upgrade the server with a bundle of this release first, in the same apply or before.
4. It runs `hopboxd install --roles <every role> --internal-listen <address>:7443 --json` in a transient unit. This is the move of the first server (plan milestone N28). It rewrites `control.internal_listen`, and the loopback URL of `channel.control_urls`, and restarts `hopboxd` after a change. The machines keep running.
5. It waits until the server is `ready`.

Run the move before a second server enrolls. A create of a server that is not the first one waits up to 5 minutes for a listener on a peer address ([Provider block](#provider-block)).

### Control servers

The first server runs the control role. An install can run it on more servers, with an external Postgres that they share ([Run the control store on an external Postgres](/guide/external-postgres/index.html.md)). A control server that is not the first one has `roles = ["control", "node"]`, or `control` with another role. Its internal listener listens on its peer address ([The internal listener](#the-internal-listener)).

Every control server must hold the same key-encryption keys ([Sealing](/reference/sealing/index.html.md#several-control-servers)). A control server whose first key came from `hopboxd install` seals values that no other control server can open. So the provider copies the files of a live control server to the new server before its first install. `copy_control_from` names that server, with its pinned host key:

```hcl
  copy_control_from = {
    address  = hopbox_server.first.address
    host_key = hopbox_server.first.ssh_host_key
    user     = hopbox_server.first.ssh_user
  }
```

| Attribute | Type | Default | Description |
| --- | --- | --- | --- |
| `address` | string | none, required | The address of the live control server, such as its `address`. The provider dials it through the jump host when the provider block has one. |
| `host_key` | string | none, required | The SSH host key of that server, in the `authorized_keys` format, such as its `ssh_host_key`. The login refuses another key. An empty value fails with `Invalid host key`, so the provider never trusts that server on its first use. |
| `user` | string | none | The login user on that server, such as its `ssh_user`. Without it, the provider tries `root`, then `debian`. |

The copy takes these steps. Steps 1 to 5 come before the login to the new server, so each check comes before any write:

1. It reads the live control processes with `ListControlProcesses` over the admin API, which runs over TLS. Each live process reports the fingerprint of each of its key files in `kek_fingerprints` ([the admin API](/reference/admin-api/index.html.md#controlprocess)). An install whose live processes report no fingerprint fails the create, because the provider cannot check a copy.
2. It logs in to `copy_control_from` with the pinned host key and `ssh_private_key`.
3. It reads `/etc/hopbox/hopboxd.toml` there. A control server without `postgres.url` runs the bundled Postgres, which listens only on its unix socket. The create then fails with `the control server runs the bundled Postgres, ...`. Move the install to the external Postgres first ([Run the control store on an external Postgres](/guide/external-postgres/index.html.md#move-a-one-server-install-to-the-external-postgres)). A `postgres.url` that is not a `postgres://` URL with a host and a user, or that holds a password, a space, a quote, or a control character, fails the create too.
4. It reads each `.key` and `.staged` file of the key dir, the file of `postgres.ca_file`, and the file of `postgres.password_file`. It only reads these paths, and never writes there.
5. It computes the fingerprint of each key file ([Sealing](/reference/sealing/index.html.md#key-fingerprints)), and compares it with the one that a live control process reports for that ID. A key file with another fingerprint, with an ID that no live process reports, or with another size than 32 bytes fails the create. So do two live processes that report other fingerprints for one ID, and an empty CA or password file. The copy must also hold the key with which each live process seals, its `kek_current`.
6. After the kernel step and the upload of the bundle, it writes the key files to `/var/lib/hopbox/kek` of the new server, with the mode `0400` and the owner root. The dir has the mode `0700`. It writes the CA to `/etc/hopbox/postgres-ca.pem` and the password to `/etc/hopbox/postgres-password`, both with the mode `0400` and the owner root. The provider picks these paths, never the server that it copies from. The data goes on standard input, never on a command line, and the script refuses any other path.
7. The bootstrap config of the new server names the same `postgres.url` and the two paths of step 6. The install of the control role finds its keys, so it makes no key of its own.

The provider never stages or promotes a key. To add a key to every control server later, stage it, copy it, and promote it as [Sealing](/reference/sealing/index.html.md#several-control-servers) shows.

A control server that is not the first one needs another role in this release, so that it enrolls. The registry then knows the server, and the provider follows it through `GetServer`. The rolling upgrade moves each control server first ([Rolling upgrade](#rolling-upgrade)).

### Uploads

The provider sends a local file of `bundle` or `host_kernel` over SSH in two scripts. The first script writes the file to a `.part` file in `/var/lib/hopbox-provider`. The second script checks the sha256 of the whole file, and moves it to its name. Both scripts take a lock on the part first, with `flock` on a `.part.lock` file. The first script holds the lock while it writes. The empty lock file stays in `/var/lib/hopbox-provider`. A script that removes it lets a later script lock a new file while an old script still holds the removed one.

sshd can drop the connection during a big upload. In a test, it dropped about one connection for each 1 to 2 GB that a laptop sent, with `message authentication code incorrect` in its log. The 600 MB bundle then failed the create. When sshd drops a session, the script of the session goes on: it writes the data that was on its way, reads the end of its input, and exits. So the first script holds no check of its own, and removes nothing.

When the connection breaks, the provider logs in again with the pinned host key, and resumes the upload:

1. It takes the lock, so it waits until the old script is gone. Then it reads the size of the part.
2. It sends the rest of the file, and appends it to the part.
3. It checks the sha256 of the whole file. SSH delivers only authenticated data, so the part holds no corrupt byte. If the sha256 still differs, the provider sends the whole file again.

The log shows `the connection broke during an upload, so the provider resumes it`, or `the part of an upload does not match after a resume, so the provider sends the file again`. Each resume and each new send counts as one try. After 6 tries, the step fails. A file from an `https` URL downloads again from its start, in one script.

### Update

| Change | What the provider does |
| --- | --- |
| `bundle.sha256` | It puts the bundle on the server, and checks its sha256. In the turn of the server in the [rolling upgrade](#rolling-upgrade), it runs `hopboxd upgrade --bundle <file> --sha256 <sha256> --json` in a transient unit under `systemd-run --wait`. Then it removes the bundle, sets `version`, and waits until the server answers its heartbeats and reports the new version in them. A server answers in the states `ready`, `cordoned`, `draining`, and `shutting_down`, so a node in the middle of a drain upgrades too. The machines keep running ([hopboxd](/reference/hopboxd/index.html.md#upgrade)). Then it reads `hopboxd version --json`. If its `features` do not name `host-firewall`, the release is from before the host firewall. Then the provider removes the table `inet hopbox_host` ([the host firewall](/reference/host-firewall/index.html.md#never-a-lockout), plan question Q79). A failed read or removal gives a warning with the command to run. |
| `labels` | `SetServerLabels`, through the admin API, or through `hopboxd admin server set-labels` on the first server. |
| `domain` | `hopboxd admin config set --domain <domain>` on the first server. |
| `floating_addresses` | It gives `dns.listen` the new list on a server with the DNS role, and writes the network setup again with the new list. Then it sets `edge.public_ipv4` to the first address in sorted order, and `edge.floating_addresses` to the list, and waits for the restart of `hopboxd`. Then it removes the dropped addresses from the uplink ([Edge servers and floating addresses](#edge-servers-and-floating-addresses)). |
| `host_kernel.sha256` | If the server does not run the tested kernel, it installs the kernel and reboots. It first cordons the server, so no machine lands on it. It refuses a server that holds machines, because the reboot stops them. After the reboot, it waits until the server is `ready`, and uncordons it. A server that an operator cordoned before stays cordoned, with a warning. |
| `internal_url` | It sets the peer address of a first server without one, and moves the internal listener of a control server to its peer address ([The internal listener](#the-internal-listener)). |
| `ssh_user` | Nothing on the server. The next steps log in as the new user. |
| `roles` | It removes `edge`, `dns`, or `node` from the server, or adds `node` back, in place ([Roles of a running server](#roles-of-a-running-server)). It runs after the upgrade and the move of the internal listener, so the server runs a release with the removal. |

Every update of the first server also checks its peer address in the install, and sets it when the install has none, as step 1 of [The internal listener](#the-internal-listener) does.

A change of `address`, `pool`, or `bootstrap` replaces the server: Terraform destroys it and creates it again. The plan refuses a replacement of the first server with `The change replaces the first server`, because its destroy ends the install. To end the install on purpose, run `terraform destroy` on the resource first.

The plan refuses another `ssh_host_key` for the same server with `The SSH host key of a server cannot change`. A server that shows another host key was reinstalled outside Terraform, or a machine sits between you and the server. Make sure that the key in the console of the provider is the key of `ssh_host_key`. For a reinstalled server that lost its install, run `terraform state rm` on the resource, make sure that nothing of the old install stays on the host, and apply again.

An update logs in only with the host key of the state. So the plan also refuses an `ssh_host_key` that only the apply knows, such as the output of a resource that the apply makes, with `Unknown SSH host key`, unless the plan replaces the server.

### Rolling upgrade

An apply that changes the `bundle` of several servers upgrades one server at a time. Release N and release N-1 can run side by side ([the bundled Postgres](/reference/postgres/index.html.md#two-releases-on-one-store), [the control channel](/reference/control-channel/index.html.md#messages-of-two-releases)). So the servers that wait keep working with the servers that moved. The order is fixed:

1. The control servers: the first server, and each server with the role `control`.
2. The servers with the edge role or the DNS role.
3. The nodes.

These rules keep the order:

- One upgrade runs at a time in one apply. Each server waits until the server before it answers its heartbeats on the new version.
- The first upgrade of an apply waits 3 s for the others, because Terraform starts the updates of servers that do not depend on each other at about the same time. Then the server with the lowest rank goes first. A server that still uploads its bundle holds back the servers of a higher rank.
- In its turn, each server reads `hopboxd version --json` of the new bundle. If its `features` do not name `peer-address`, the release runs no mesh and no control channel on the peer addresses. Then the update fails while a server of the install that is not decommissioned has a peer address, with `hopboxd <version> runs no mesh and no control channel on the peer addresses, and the server <id> of the install has the peer address <address>. ...` (plan question Q79, N28). The provider removes the bundle, so nothing changed on the server.
- In its turn, a server of rank 2 or 3 reads the servers of the install. If a server of a lower rank runs an older `hopboxd` than the new bundle, the update fails with `... runs hopboxd <version>, older than <version>. A rolling upgrade moves the control servers first, ...`. The registry of the install has no control role, so the first server counts there by its edge and DNS roles. The check also reads the control processes of the install, with `ListControlProcesses` or, over SSH on the first server, `hopboxd admin control list --json` ([hopboxd](/reference/hopboxd/index.html.md#admin-control-list)). If a live control process runs an older release than the new bundle, the update of an edge, DNS, or node server fails with `the control process <id> on <host> runs hopboxd <version>, older than <version>. A rolling upgrade moves the control servers first, ...`. An install whose control role is older than the call answers `unimplemented`, and counts no control process. The provider removes the bundle that it put on the server, so nothing changed there.
- A server of a lower rank goes first while its update is in the queue of the apply, also while it still uploads its bundle. So the order check fails only for a server of a lower rank that has no update in the apply yet. That refusal is no failure of an upgrade, and the other servers of the apply go on.
- If an upgrade fails, the servers that still wait in the same apply do not upgrade. Their updates fail with `the upgrade of the server <id> failed in this apply, so the rolling upgrade stops before this server`, and the provider removes their bundles. A failure of an upgrade is a failure of `hopboxd upgrade` on the server, or a server that does not come back on the new version.

Terraform runs at most 10 updates at the same time by default (`-parallelism`). With more servers, a node can take its turn before Terraform starts the update of the first server, and the order check then fails it. To upgrade a big install in one apply, add `depends_on` to the resources of the nodes, so Terraform updates them after the first server:

```hcl
resource "hopbox_server" "node1" {
  address      = "203.0.113.21"
  ssh_host_key = "ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAINode1..."
  labels       = { zone = "zone1" }

  bundle      = local.bundle
  host_kernel = local.host_kernel

  depends_on = [hopbox_server.first]
}
```

`local.bundle` and `local.host_kernel` hold the objects of `bundle` and `host_kernel`, so every server takes the same files. Then one change of `local.bundle` upgrades the whole install in order.

Until N32, a role that a new release brought came in place to a first server that lacked it. An example is the role `edge` on a first server from before that role. That path ends with N32. Every install has the roles `edge` and `dns`, so no current release brings a role. The next release that adds a role defines its own rule for it. A token adds a role to a running server only when a removal took that role from the server ([Enrollment and the install PKI](/reference/enrollment/index.html.md#add-roles-to-an-enrolled-server)). So a plan that adds a role other than `node` to a running server fails with `Roles of a node`, and waits for the owner. [Roles of a running server](#roles-of-a-running-server) describes the drop of a role and the add of `node` back.

The provider keeps its own cordon of a reboot in the private state of the resource. A step after the cordon can fail, such as the reboot, the wait for `ready`, or the uncordon. The error then ends with `The server <id> stays cordoned. Run hopboxd admin server uncordon <id>, or apply again`. Terraform keeps the old state, so the next apply plans the same update. That update first lifts the cordon of the provider. A refresh or an update that finds the server uncordoned forgets that cordon. So after you uncordon by hand, a later cordon of yours stays.

An import and `terraform state rm` lose the private state, and so an open cordon of the provider. After either one, run `hopboxd admin server get <id>` on the first server. If `cordoned` is `true` and no operator cordoned the server, run `hopboxd admin server uncordon <id>`.

### Roles of a running server

A plan that drops `edge`, `dns`, or `node` from a running server, the first server too, updates it in place (plan question Q100). `dns` needs `edge`, so a plan that drops `edge` drops `dns` too. It also drops `floating_addresses` and `public_ipv6` in the same change, because only an edge server holds them. A server without `edge` takes SSH only through the jump host, so the provider block needs `ssh_jump_host`, and the jump host must be another edge server.

The update takes these steps:

1. It calls `RemoveServerRoles` with the roles that go, through the admin API, or with `hopboxd admin server remove-role` over SSH on the first server ([the admin API](/reference/admin-api/index.html.md#removeserverroles)). A pending removal of the same roles from an earlier apply goes on.
2. For `node`, it waits for the moves of the machines with the wait of a drain, at most 6 hours ([Drain of a server](#drain-of-a-server)). A machine that cannot move fails the update with `the removal of the node role from the server <id> cannot move the machines <name> (<reason>), ...`. The machine stays on the server with its disk, and the removal waits.
3. It ends the removal: a token for the server, the enrolling roles that stay, and its peer address, from the admin API, or with `hopboxd admin enrollment-token create --server <server_id> --roles <roles> --peer-address <address>` on the first server. It runs `hopboxd install --roles <every role that stays> --json` with the token in a transient unit, `control` included on a control server ([hopboxd](/reference/hopboxd/index.html.md#remove-roles)). The install enrolls with the token, and restarts `hopboxd` without the roles. Then the update waits until the server is ready.
4. For `edge`, it then takes the floating addresses off the uplink, with the undo of the network setup of the provider ([The network setup of an edge server](#the-network-setup-of-an-edge-server)). The config and the restart come first, and the drop of the addresses comes after, so `hopboxd` never runs the edge role without them. The operator moves the floating addresses to another edge at the provider before the apply, so the checks of the removal pass.

The restart stops the services of the server for some seconds. On the only edge, the API, the URLs, SSH, and DNS stop for that time.

After a failed update, the state keeps the old `roles`, because the update writes no new roles, and a refresh reads the same roles from the registry. The removal stays pending. Then take one of two ways:

- To go on, fix the cause, such as a machine without room, and run `terraform apply` again. It returns the pending removal and waits again.
- To give up, run `sudo hopboxd admin server cancel-remove-role <server_id>` on a control server, and put the role back in the configuration, so the next plan shows no change ([the admin API](/reference/admin-api/index.html.md#cancelroleremoval)).

A plan that adds `node` back to a running server updates it in place, when the server row records that a removal took `node` from it (`removed_roles`). The update makes a token for `node` alone that names the server and its peer address. It runs `hopboxd install` with every role of the server, and waits until the server is ready. The node takes its old subnet again. A server that never lost `node` fails with `the server <id> never lost the role node, so the provider cannot add it`, before any token.

The plan refuses these changes:

| Change | Error |
| --- | --- |
| A drop of `control` | `A role cannot go` |
| An add of a role other than `node` | `Roles of a node` |
| A drop and an add in one plan | `Roles change one way at a time`. Apply the drop and the add one after the other. |
| Roles of the first server without `control` | `Unsupported roles` |
| A first server that does not start with every role | `Unsupported roles`, at its create only |
| A drop of `edge` with `ssh_jump_host` on the same server | `ssh_jump_host of the provider block names the server <address>, whose role edge the plan drops`, at the apply |

### Delete

A delete drains the server, stops `hopboxd`, and removes what the provider put on the host for the install. The host stays with its provider (plan question Q43). It takes these steps:

1. It reads the server from the install. A server that the install does not know, or a decommissioned one, goes straight to step 4.
2. On the first server, it refuses while another server of the install is not decommissioned and no other control process is live: `the first server runs the only live control role of the install, ...`. A live control process of another enrolled server, from `ListControlProcesses`, keeps the control plane of the others, so the first server can leave the install. A process without a `server_id` does not count, because the provider cannot tell whether it runs on the first server. The decommission of the drain makes the same check.
3. It drains the server with `DrainServer`, and waits until the drain decommissions it ([Drain of a server](#drain-of-a-server)). A server without the role `node` holds no machine, so it goes out with `DecommissionServer` at once.
4. It runs `systemctl disable --now hopboxd.service` on the server. After a drain, it logs in to the server again first, because the connection of the start can die during a drain of hours. If the stop fails after a drain of this resource, the delete fails, and the resource and its record of the drain stay: `the drain decommissioned the server <id>, but hopboxd did not stop on it: ... Run terraform destroy again to stop it and to remove the secrets of the install. If the provider cannot log in to the server for good, run terraform state rm on the resource, and stop hopboxd and remove the secrets on the server by hand`. A later destroy of that resource runs no drain, so its error starts with `the server <id> is decommissioned, but hopboxd did not stop on it` instead. Without a drain, such as for a server that was already decommissioned outside Terraform, a failed stop gives a warning, and the delete succeeds.
5. On a server with the role `control`, the first server too, it removes the secrets of the install once `hopboxd` stopped, as the next list says.
6. Once `hopboxd` stopped, it removes the rest of the install from the host:
   - The identity of the server, `<data_dir>/server`, with its private key. The install decommissioned the server, so the identity opens nothing, and a later create on the same host enrolls a new server. A failure gives a warning, and a later create on the host refuses it until you remove the dir.
   - The nftables table `inet hopbox_host` of the [host firewall](/reference/host-firewall/index.html.md), and the files `/var/lib/hopbox/hostfw/peers` and `/var/lib/hopbox/hostfw/vrack-interface`, as the owner decided on Q77. Without the table, the public address of the server takes SSH again. A server without `nft` gets the package `nftables` first.
   - The network setup of the provider: it runs `hopbox-network undo`, which removes the floating addresses, the address of `public_ipv6`, and the source of the default route, and removes the script and the unit. A failure gives a warning with the commands to run.

The login, the stop of `hopboxd`, or the removal of the table can fail. On a server without the role `edge`, the table then keeps port 22 closed to every address but the edge servers, and the jump host still reaches the server. So the delete fails with the title `The server keeps its host firewall` and the error `the server <id> keeps its host firewall, because ...`. The resource records it as it records a drain, so a refresh keeps the resource, and the next destroy goes on. The error names the commands to run. On an edge server, whose public address keeps SSH, the delete goes on with a warning.

On a server with the role `control`, the keys of the last control server open every sealed value of the store. So the delete removes them only when another control server proves that it holds them:

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

A delete of a server that never enrolled, after a failed create, decommissions nothing, and gives the warning `The server never enrolled, so the provider decommissions nothing.` It still stops a `hopboxd` that the create started, and removes the table of the host firewall and the network setup, so a new create takes the host. On the first server, the control store that the failed create made stays in `/var/lib/hopbox/pg`, with its keys in `/var/lib/hopbox/kek`, and a warning says so. A new create of the first server on that host refuses them, so remove both first.

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

The host stays with its provider, and its disks stay as they are (plan question Q43). A destroy never ends a rental. End it at the provider. To use the host again, create a `hopbox_server` for it: the destroy removed the identity of the server, so the host takes a new create ([Create](#create)).

### Read

A create can fail before the enrollment. Its state then has no `server_id`. The refresh keeps such a state as it is, so Terraform can destroy the server and create it again.

Otherwise the refresh reads the server from the install. It uses SSH on the first server and the admin API on the others. It sets `labels`, `pool`, and `version` from the install. On the first server, it also sets `domain` from `hopboxd admin config get`. So the plan shows a real change of the domain. It keeps `roles` as the configuration orders them, when the server has the same roles. On a control server, it sets `internal_url` from the live control process of the server. On an edge server with `floating_addresses`, it logs in to the server and keeps only the addresses that the server holds on a network interface. The refresh changes nothing on the server. If the install does not answer, the refresh keeps the state as it was, with a warning.

The install can lose a server that is not the first one, or decommission it. The refresh then removes that server from the state, unless a failed destroy started its drain ([Drain of a server](#drain-of-a-server)). Then the resource stays, and the next destroy finishes the removal. The first server never leaves the state this way, because a new create would make a new install on it. So the refresh fails with `the first server <id> is decommissioned or unknown to its install, but Terraform did not destroy it`. To make a new install on it, run `terraform state rm` on the resource, remove the install from the host, and apply.

### Import

```sh
terraform import hopbox_server.node1 203.0.113.21
```

The import takes the peer address of the server alone (plan question Q96). The refresh after it logs in at that address with `ssh_private_key` as `root`, then as `debian`, through the jump host when the provider block has one. It reads the release in `/opt/hopbox/current`, the roles, and `edge.public_ipv6` of `/etc/hopbox/hopboxd.toml`, and the server ID from the certificate of the server. It reads `edge.public_ipv6` only when the value has no prefix length. The address of the IPv6 block of a release before N29 reads as none, as `hopboxd` ignores it. It reads only the certificate, never the private key. A server with the role `control` and the bundled Postgres cluster of the first install, `/var/lib/hopbox/pg/data/PG_VERSION`, imports with `bootstrap = true`, and the refresh reads its `domain` from the install. Another control server never makes that cluster, so it imports without `bootstrap`.

The import checks the server before it keeps it:

- The server holds the address on a network interface. Otherwise the import fails with `the server at <address> holds no address <address> on a network interface, ...`.
- The install knows the server, through the admin API at `install_url`, or over SSH on a first server without it. A server of another install fails with `the install knows no server <id>, the server at <address>, so it belongs to another install`.
- The install records the address as the peer address of the server. Another address fails with `the install knows the server <id> with the peer address <peer address>, not <address>. Import it by its peer address`. A first server without a peer address imports, and its next update sets it. Another server without a peer address fails with `the install knows the server <id> at <address> without a peer address, so the import cannot tell it by its address`. Such a server enrolled before peer addresses (plan milestone N26). Set its peer address with `hopboxd admin server set-peer-address` on a control server ([hopboxd](/reference/hopboxd/index.html.md#admin-server-set-peer-address)), and import it again.

The import has no pinned host key yet, so it keeps the key that the server shows at this login, with the login user. It asks the server for its ed25519 key first. That is the key of `/etc/ssh/ssh_host_ed25519_key.pub`, which the guide copies from the console ([Install hopbox on your servers with Terraform](/guide/install-with-terraform/index.html.md#1-set-up-the-hosts)). The next plan compares that key with `ssh_host_key`, and fails with `The SSH host key of a server cannot change` when they differ, before any change of the server. Make sure that the key in the console of the provider of the server is the key of `ssh_host_key`.

An import cannot recover `bundle`, `host_kernel`, or `admin_api_key`. The next plan shows an update of `bundle` from your Terraform files. If the file name of the bundle names the version that the server runs, the apply does not upgrade. The state of an imported first server has no `admin_api_key`.

### Host key trust

The provider never trusts a server on its first use (plan question Q95). The operator copies the SSH host key of each host from the console of its provider into `ssh_host_key`, and the provider pins that key before its first login. A server that shows another key fails each step with `remote: the host key of the server is not the pinned one`, and the create changes nothing on it. It asks the server for the key type of the pin.

The jump host has its own pin, `ssh_jump_host.host_key`, from the start. To read the files of a live control server for a new control server ([Control servers](#control-servers)), the provider pins the host key of `copy_control_from`. The provider then checks the fingerprint of each key file against the admin API, which runs over TLS, before it writes anything. So a machine that plays the control server at its address cannot plant its own keys, or a Postgres that would accept them, on the new server.

An import is the one login without a pin. The plan after it refuses a key other than `ssh_host_key` ([Import](#import)).

### A state of an earlier release

The provider of phase 1 named each server by the OVHcloud API, with a state of schema version 0. This release names a host by its peer address, with schema version 1. The first refresh with this release moves a state of version 0 to version 1, with no change on the server, as for ks:

| Attribute of version 1 | Value from the state of version 0 |
| --- | --- |
| `id`, `address` | The `ipv4` of the old state, the public IPv4 address of the server. A state without `ipv4`, or with an address that cannot be a peer address, fails the upgrade, and says to import the server by its peer address. |
| `ssh_host_key` | The pinned host key of the old state. A state without one fails the upgrade. |
| `ssh_user` | The login user of the old state, such as `debian` with `sudo` on ks. |
| `server_id`, `admin_api_key`, `version`, `pool`, `labels`, `roles`, `bootstrap`, `domain`, `bundle`, `host_kernel`, `floating_addresses`, `copy_control_from` | The same values. |
| `public_ipv6` | The same floating IPv6 address. The address of the IPv6 block of a release before N29, with its prefix length, becomes null, as `hopboxd` ignores it. |
| `internal_url` | Null. The refresh reads it from the server. |

The attributes of OVHcloud and of the vRack go, with the order of a server, and so does `public_ipv6_gateway`. The private state keeps the cordon of the provider and the record of a drain. Before the first plan, give the resource `address`, `ssh_host_key`, and `ssh_user` with the values of the state, so the plan changes nothing that the server keeps. On a first server from before peer addresses, the plan then shows the move of its internal listener ([The internal listener](#the-internal-listener)). The guide for ks shows the steps for ks.

### Timeouts

| Step | Limit |
| --- | --- |
| The login after the reboot into the host kernel | 30 minutes, tried every 10 s |
| The reboot into the host kernel | 20 minutes |
| The wait after the make of an enrollment token, for the set `registered` | 3 s |
| A listener on a peer address, for the create of a server that is not the first one | 5 minutes, polled every 2 s |
| `ready` after the install | 5 minutes, polled every 2 s |
| `ready` on the new version after an upgrade | 5 minutes, polled every 2 s |
| The wait of the first upgrade of an apply for the others | 3 s |
| The proof that another control server runs, at the destroy of a control server | about 6 s, the renewal period of 5 s and 1 s more |
| The decommission by a drain | 6 hours, polled every 10 s |

### Errors

| Error | Cause |
| --- | --- |
| `The argument "address" is required` or `The argument "ssh_host_key" is required` | A `hopbox_server` without its address or its host key. |
| `Invalid address` with `invalid peer address: <address> is a private address. A peer address is one public IPv4 address on the uplink of the server, with no NAT`, or another reason | An `address` that is not one public IPv4 address. |
| `Invalid host key` | An `ssh_host_key` or a `copy_control_from.host_key` that is not an SSH public key in the `authorized_keys` format. |
| `Unknown SSH host key` | An `ssh_host_key` of a server in the state that only the apply knows ([Update](#update)). |
| `Invalid SSH user` | An `ssh_user` that is empty or holds a space, a quote, or another character that no user name has. |
| `Invalid import ID` with `Import a server by its peer address: ...` | An import ID that is not a peer address. |
| `The first server of an install (bootstrap = true) needs the domain of the install.` | `bootstrap = true` without `domain`. |
| `Only the first server of an install (bootstrap = true) takes the domain.` | `domain` without `bootstrap`. |
| `the first server has the roles control, node, dns, edge. Leave roles out to get them` | Another list in `roles` of the first server. |
| `unknown role "<role>". The roles are control, node, dns, and edge`, `the role <role> appears twice`, or `a server needs at least one role: control, node, dns, or edge` | A bad list in `roles` of another server. |
| `the DNS role runs on edge servers, so a server with the role dns also needs the role edge` | `dns` without `edge`. |
| `in this release, a control server that is not the first one also needs another role, such as node, so that it enrolls` | `roles = ["control"]` on a server that is not the first one. |
| `Invalid floating address` with `the floating address "<value>" must be one public IPv4 address, such as 203.0.113.5` | A bad entry of `floating_addresses`. |
| `The peer address as a floating address` | `floating_addresses` holds `address`. |
| `Floating addresses of a server without the edge role` | `floating_addresses` on a server without the role `edge`. |
| `Invalid public IPv6 address` with an error of `edge.public_ipv6` | A bad `public_ipv6`, such as an address with a prefix length or a unique local address. The errors are those of the bootstrap config ([the bootstrap config of hopboxd](/reference/hopboxd-config/index.html.md#errors)). |
| `Unsupported argument` for `public_ipv6_gateway` | A configuration from before N29 sets `public_ipv6_gateway`, which the resource no longer has. Remove the argument, and give `public_ipv6` the floating IPv6 address alone. The upgrade of an old state drops the attribute, and drops an address of the IPv6 block in `public_ipv6` ([A state of an earlier release](#a-state-of-an-earlier-release)). So a plan without `public_ipv6` changes nothing on the server. |
| `A public IPv6 address of a server without the edge role` | `public_ipv6` on a server without the role `edge`. |
| `The public IPv6 address of a server cannot change` | A change of `public_ipv6` without a replacement of the server. |
| `The SSH host key of a server cannot change` | Another `ssh_host_key` for the same server, or an import that read another key than `ssh_host_key`. See [Update](#update). |
| `The change replaces the first server` | A change of `address`, `pool`, or `bootstrap` of the first server. |
| `public_ipv6 needs a release of hopboxd whose version names the feature floating-ipv6, and the bundle <name> does not: ...` | A create, or a move to another release, of a server with `public_ipv6`, with a bundle from before N29. That release refuses `edge.public_ipv6` without the gateway of the IPv6 block, so `hopboxd` would not start. At a create, the provider stops before the first change of the host and removes the bundle, so Terraform keeps no state of the server. At a move, it stops before the upgrade. At a create, give the server a bundle of the release of the provider, or leave out `public_ipv6`. At a move, keep a bundle of a release with the feature. `public_ipv6` cannot change in place, so to run the older release, replace the server with a new one without `public_ipv6`. |
| `The jump host is not valid` with `ssh_jump_host.address ...` or `ssh_jump_host.host_key must be an SSH public key ...` | A bad `ssh_jump_host`. |
| `The jump host is not known yet` | `ssh_jump_host` names an attribute of a resource that the same apply creates. Give literal values, such as the outputs of an earlier apply. |
| `the server <address> has no role edge, so its host firewall takes SSH only from the edge servers once hopboxd runs, ...` | A server without the role `edge`, and no `ssh_jump_host`. Nothing changed on the server. |
| `remote: the host key of the jump host is not the pinned one: ...` | The jump host shows another key than `ssh_jump_host.host_key`. Make sure that the key of the edge server is the key of `ssh_host_key` of its `hopbox_server`, then fix the provider block. |
| `two hopbox_server resources have the floating address <address>: ...` | Two resources list one floating address. The plan fails. |
| `two hopbox_server resources have the public IPv6 address <address>: ...` | Two resources have one address of `public_ipv6`. The plan fails. |
| `a server that is not the first one enrolls at the internal listener of a control server on its peer address, and the install lists none. ...` | No control URL in the provider block, and no live listener on a peer address in the install within 5 minutes. Apply the first server first, so its update moves its listener. |
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
| `<bundle or host_kernel>: <file> has the sha256 <got>, not <want>` | A local file that differs from its `sha256`. The create stops before its first login. |
| `the provider has no SSH key. Set ssh_private_key in the provider block, or HOPBOX_SSH_PRIVATE_KEY` | No `ssh_private_key`. |
| `a server that is not the first one needs the admin API: set install_url and admin_api_key ...` | A server without `bootstrap`, and no admin API. |
| `Two control URL settings` with `Set control_urls or control_url, not both.` | The provider block sets both `control_urls` and `control_url`. |
| `The control URL is not valid` with `the control URL "<url>" must be an https URL, such as https://203.0.113.10:7443` | A control URL is not an `https` URL of a host. |
| `log in to the server at <address>: remote: the host key of the server is not the pinned one: ... Compare ssh_host_key with the host key in the console of the provider of the server. ...` | The server shows another host key than `ssh_host_key`. Either the address names another server, the server was reinstalled, or a machine sits between you and the server. Check the key before you do anything else. |
| `log in to the server at <address> as <user>: ...` | The login failed, such as for a key that the server does not authorize for `ssh_user`. |
| `sshd of the server takes password logins: sshd -T prints passwordauthentication yes for a login of root from 192.0.2.1. ...` | The `sshd` of the host takes passwords. Turn them off as the error says. Nothing changed on the server. |
| `the host holds an install: hopboxd runs on the host at <address>, ...` | The create found a running `hopboxd`. Make sure that `address` names the new host. Nothing changed on the server. |
| `the host holds an install: the host at <address> holds the identity of an enrolled server in <data_dir>/server, ...` | An earlier install left its identity on the host. Nothing changed on the server. |
| `the host holds an install: the host at <address> holds the control store of an earlier install in <data_dir>/pg, ...` | The create of a first server on a host with an earlier control store. Nothing changed on the server. |
| `the server runs <os> <version>, but hopbox tests only Debian 13. Install Debian 13 on the server at its provider first` | The host runs another OS. |
| `the server at <address> holds no address <address> on a network interface, ...` | A NAT sits in front of the host, or `address` names another server. Nothing changed on the server. |
| `the host at <address> names no resolver that a machine can reach in ...` | A node on a host with a resolver on loopback alone. Nothing changed on the server. |
| `/var/lib/hopbox is not a file system of its own. Make it an XFS file system with reflinks on a partition of its own before the create: ...` | The host has no data dir of its own. |
| `/var/lib/hopbox is on <fs>, but the disks of machines need XFS with reflinks there. Make it ...` | The data dir is not XFS. |
| `/var/lib/hopbox is on XFS without reflinks, and it holds files, so the provider does not make it again` | Files in a data dir without reflinks. |
| `set up the network of the server: ... Read journalctl -u hopbox-network.service on the server` | The network setup of an edge server failed, such as a server without an IPv4 default route. |
| `the host kernel package is <package> <version> for <arch>, but the provider installs linux-image-6.18.57-hopbox1 6.18.57-1 for amd64` | The wrong package in `host_kernel`. |
| `<file> has the sha256 <got>, not <want>` | A file on the server differs from its `sha256`. |
| `the server booted <release> after the install of 6.18.57-hopbox1. Check the GRUB default on the server` | GRUB booted another kernel. |
| `write the bootstrap config: ...` | The provider cannot write `/etc/hopbox/hopboxd.toml`. |
| `the server <id> did not become ready: ...` | The server did not send a heartbeat in 5 minutes. Read `journalctl -u hopboxd` on the server. |
| `the server <id> did not come back on <version>: ...` | After its upgrade, the server did not become `ready` with the new version in 5 minutes. Read `journalctl -u hopboxd` on the server, and run `hopboxd upgrade --rollback` there if the new release does not start. |
| `the server <id> with the roles <roles> runs hopboxd <version>, older than <version>. ...` | A server of a higher rank waits for a server of a lower rank in the [rolling upgrade](#rolling-upgrade). Upgrade that server first, and apply again. |
| `the upgrade of the server <id> failed in this apply, so the rolling upgrade stops before this server. ...` | An earlier upgrade of the same apply failed. Fix that server, and apply again. |
| `read the servers of the install for the order of the rolling upgrade: ...` | The admin API did not answer the list of the servers. Nothing changed on the server. |
| `hopboxd <version> runs no mesh and no control channel on the peer addresses, and the server <id> of the install has the peer address <address>. Move the server to a release whose hopboxd version --json names the feature peer-address` | The new bundle is a release from before N28, and the install runs on the peer addresses ([rolling upgrade](#rolling-upgrade)). Use a newer bundle. Nothing changed on the server. |
| `read the servers of the install for their peer addresses: ...` | The admin API did not answer the list of the servers. Nothing changed on the server. |
| `the install records the peer address <peer address> for the first server <id>, not its address <address>. ...` | The `address` of the first server differs from the peer address in its install. |
| `the hopboxd of the server <id> runs no internal listener on a peer address: its version names no feature peer-address. ...` | The move of the listener on a release from before N28. Upgrade the server first. |
| `move the internal listener to <address>:7443: ...` | `hopboxd install --internal-listen` failed on the server. |
| `the server <id> runs <release> and holds <n> machines. The install of the host kernel reboots it, so delete the machines first` | A change of `host_kernel` on a server with machines that does not run the tested kernel. |
| `the first server <id> is decommissioned or unknown to its install, but Terraform did not destroy it. ...` | Someone decommissioned the first server outside Terraform. A `terraform destroy` can also decommission the server and then lose the answer. Then the next destroy fails on this error at its refresh. Run `terraform destroy -refresh=false` to finish the destroy, or `terraform state rm` on the resource. |
| `log in to the first server to decommission it: remote: the host key of the server is not the pinned one: ... run terraform state rm on the resource` | A delete of a first server that someone reinstalled outside Terraform. Its install is gone. |
| `The server keeps its host firewall` with `the server <id> keeps its host firewall, because ...` | A delete of a server without the role `edge` whose stop of `hopboxd` or removal of the table failed. Fix the cause, and run `terraform destroy` again. |
| `the install knows no server <id>, the server at <address>, so it belongs to another install` | An import of a server of another install. |
| `the install knows the server <id> with the peer address <peer address>, not <address>. Import it by its peer address` | An import by an address that is not the peer address of the server. |
| `The state of the server cannot move to this release` | A state of version 0 without `ipv4` or without a pinned host key. Run `terraform state rm` on the resource, and import the server by its peer address. |
| `/var/lib/hopbox-provider is not a dir of root with mode 0700. Remove it` | Another user made the work dir of the provider. |

A failed script on the server gives `the script exited with status <n>:` and the end of its error output. A connection that breaks during a script gives `remote: the connection broke: ...`. The provider resumes an upload after it ([Uploads](#uploads)), and fails every other step.

## Resource hopbox_pool

A `hopbox_pool` is one pool of servers of an install ([the admin API](/reference/admin-api/index.html.md#poolservice)). All servers of a pool have identical hardware. The first server that enrolls into the pool records its hardware, and a new server with other hardware fails to enroll ([Enrollment and the install PKI](/reference/enrollment/index.html.md#hardware-of-a-pool)). So give each server model its own pool. A `hopbox_server` joins the pool with `pool = hopbox_pool.<name>.name`, which also makes Terraform create the pool first.

```hcl
resource "hopbox_pool" "big" {
  name = "big"
}
```

The resource calls the admin API at `install_url` with `admin_api_key` ([Provider block](#provider-block)). So it needs both, as a server that is not the first one does. The first server joins the pool `default`, because no other pool exists before it. A plan that gives the first server another pool fails.

| Attribute | Type | Default | Change | Description |
| --- | --- | --- | --- | --- |
| `name` | string | none, required | replaces | The name of the pool, a lowercase DNS label. No other live pool can have it. |
| `id` | string | set by the provider | | The ID of the pool, a UUID. |

The create calls `CreatePool` with a new idempotency key. The delete calls `DeletePool`. The install refuses the delete of a pool that holds a server that is not decommissioned, or that a live size names, so destroy the servers and the sizes of the pool first. Terraform orders the destroys that way for the servers and the sizes that name the pool by `hopbox_pool.<name>.name`. A delete of a pool that is gone already succeeds.

A refresh calls `GetPool`. A pool that is gone leaves the state, so the next apply creates it again. A pool of the same name with another ID also leaves the state, after someone deleted the pool and created it again outside Terraform. The next create then fails with `already_exists`, so import that pool instead. A state of an earlier release with `ovh_plan` loses the attribute at the next refresh, and nothing changes in the install. Import a pool by its name:

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
| `internal/tfprovider/remote` | `make test` | The host key pin, the login users, and scripts with their input and exit status, against an SSH server in the test. A login through a jump host, with the pins of both hosts. |
| `internal/tfprovider/hostsetup` | `make test` | The scripts of each step, and that no token or private key appears in them. The boot script of the network setup runs with a stub of `ip`: it finds the uplink by its default route, sets each floating address, and puts the peer address as the source of the route, with its flags. `drop` and `undo` remove what the setup set. The checks of `sshd -T`, of a host that holds an install, and of the data dir give their fixes. |
| `TestResumeAfterADroppedUpload` | `make test-linux` | The upload scripts with a real `sh`, as root in a private mount namespace. A dropped upload goes on in the background, and a slow resume runs beside it. The resume waits for the lock, appends the rest, and the old script removes nothing. |
| `TestInstallKernelPinsTheRelease` | `make test-linux` | The kernel step of the create, as root in a private mount namespace, with stubs of `dpkg -i` and `update-grub`. The GRUB pin names the kernel release `6.18.57-hopbox1`, not the package version `6.18.57-1`. |
| `TestEdgeSourceIsItsPeerAddress` of `test/fleet` | `make test-fleet` | The boot script of the network setup with the real kernel, in a network namespace whose uplink holds a floating address first. Before the script, the default route takes the floating address as its source. After it, the route takes the peer address, also when the script runs again at a boot. The floating IPv6 address sits on the uplink as a `/128`. `drop` and `undo` remove the addresses and the source. |
| `TestCreateBootstrap`, `TestPublicAddressesNameOnlyFloatingAddresses`, `TestCreateNode`, `TestCreateRefuses`, `TestCreateChecksFilesBeforeAnyChange`, `TestCreateAfterADestroyOfTheSameHost`, `TestPartialCreateRefreshesAndDestroys`, `TestUpdate`, `TestFirstServerMovesItsListener`, `TestNodeWaitsForTheMoveOfTheFirstServer`, `TestRemove`, `TestRead`, `TestImportByPeerAddress` | `make test` | The flow of the resource against fake hosts and a fake admin API, with no API of a provider. A wrong host key, a host with password logins, a host whose `hopboxd` runs, and a host with the identity of a server fail before any change. The public addresses of a new install name its floating addresses, and never an IPv6 address of the uplink. A first server from before peer addresses gets its peer address and moves its listener at its update. A destroy removes the identity, the table, and the network setup, and a new create takes the same host. An import by peer address refuses a server of another install. |
| `TestUpgradeRank`, `TestUpgradeQueueOrder`, `TestUpgradeQueueStopsAtAFailure`, `TestRollingUpgrade`, `TestUpgradeRefusesANodeBeforeItsControlServer`, `TestUpgradeRefusesAReleaseWithoutPeerAddresses`, `TestUpgradeWaitsForTheNewVersion` | `make test` | The [rolling upgrade](#rolling-upgrade): one upgrade at a time in the order of the ranks, also when the update of a node starts first. The stop after a failed upgrade, the refusal of a node before its control server, the refusal of a release without the feature `peer-address` in an install with a peer address, and the wait for the new version in the heartbeat. |
| `TestCreateNodeThroughTheJumpHost`, `TestCreateEdge`, `TestCreateEdgeWithoutDNS`, `TestCreateControlServer`, `TestFloatingAddresses`, `TestFirstServerLeavesThroughTheAPI`, `TestDestroyKeepsSecretsOnlyWithAWarning`, `TestOnlyControlServerKeepsItsKeys`, `TestTwoDestroysKeepTheLastKeys`, `TestDestroyKeepsKeysThatNoOtherServerHolds`, `TestStaleRowKeepsTheLastKeys`, `TestDestroysOfOneProcessTakeTurns`, `TestOwnRowIsNoHolder`, `TestControlCopyRefusesOddKeys`, `TestNoJumpHostRefusesAServerBehindTheFirewall`, `TestDestroyKeepsAFirewalledServer` | `make test` | The fleet against fake hosts. A node gets each SSH step through the jump host at its peer address. Edge servers and the first server get their SSH straight. An edge server gets its floating IPv4 addresses and its floating IPv6 address on its uplink, with its peer address as the source, and in `dns.listen`. An update of the floating addresses sets the new list, and a refresh reads the addresses from the server. A control server copies the key files and the Postgres of the live control server of `copy_control_from` before its install, and refuses before any write on each bad source. The destroy of a control server keeps the keys that no other live server proves it holds. A destroy of a server behind the jump host whose table stays keeps the resource. |
| `TestProviderFleet`, `TestProviderBlockOfTheFleet`, `TestDoubleClaimFailsAtPlan`, `TestDestroyAfterADrainThatOutlivedIt` | `make test` | Terraform applies a fleet of five servers in two pools through the jump host: the first server, a second edge server, a second control server, and two nodes. It imports an edge server and a node by peer address. It replaces the edge server with `create_before_destroy`, and the new server holds its floating address. A plan that changes `public_ipv6`, the address of the first server, or a host key fails. A floating address or a public IPv6 address of two resources fails the plan. |
| `TestOldStateOfKsUpgrades` | `make test` | A state of version 0 of ks, with the user `debian` and an OVHcloud record, moves to version 1. The upgrade and the refresh run no script that changes the server. The plan replaces nothing, and shows the move of the listener, which the apply runs with the peer address of the old state. |
| `TestPoolAndSizeLifecycle`, `TestPoolAndSizeValidation`, `TestPoolNeedsTheAdminAPI` | `make test` | Terraform creates a pool and a size against a fake admin API over Connect, imports them, refuses a change of a size that keeps its name, renames the size, and destroys both. They need the `terraform` binary as the tests below do. |
| `TestServerResourceLifecycle`, `TestServerResourceValidation` | `make test` | Terraform itself runs a create, an update, an import, and a destroy, and plans bad Terraform files. They need the `terraform` binary on `PATH`, or in `TF_ACC_TERRAFORM_PATH`. Without it, terraform-plugin-testing downloads one. |
| `TestRolesChangeInPlace`, `TestRemovalOfNodeFailsOnAStrandedMachine`, `TestRemovalThatACancelEnded`, `TestRemovalGoesOnAfterAFailedEnd`, `TestRemovalOfEdgeKeepsTheJumpHost`, `TestNodeComesBackOnlyAfterItsRemoval` | `make test` | Terraform drops `edge` and `dns` from the first server with a token of its admin socket. It drops `node` from a second edge and adds it back with tokens of the admin API. Each change is an update in place. The floating address goes after the restart. The plan refuses an add of `edge` and a first server without `control`. A stranded machine and a cancel fail the update, and the error names the way out. A failed end goes on at the next apply. The jump host keeps its edge, and `node` comes back only after its removal ([Roles of a running server](#roles-of-a-running-server)). |
| `TestInstallOverSSH` | `make test-provider`, and the CI job `install` | The provider over real SSH against three Debian 13 containers with systemd and sshd, on a Docker network of public documentation addresses, with no API of a provider. The first server logs in as `debian` with `sudo`, an edge server holds a floating address before its peer address on its uplink, and a node sits behind the jump host. A wrong host key, a host with password logins, and a host without its data dir fail before any change, with their fixes. The edge enrolls from its peer address, and its connections to port 7443 and port 22 come from that address. The test covers an import by peer address, the refusal of a host whose `hopboxd` runs, an upgrade, a delete that leaves the node open to SSH, and a new create on the same host. The containers run the kernel of the host, so the test does not reboot. It needs `/dev/kvm` for the preflight check, so it does not pass on an Arm Mac with Docker Desktop. Run it on a Linux host for amd64 with KVM. |
| `TestAccServer` | by hand, with `TF_ACC=1` | A create and a destroy on a real host that runs Debian 13, as the first server of a new install. It ends the install on the host. So the test skips unless `HOPBOX_ACC_TAKE` names the same address as `HOPBOX_ACC_ADDRESS`. It also needs `HOPBOX_ACC_HOST_KEY`, `HOPBOX_ACC_DOMAIN`, `HOPBOX_ACC_BUNDLE`, `HOPBOX_ACC_BUNDLE_SHA256`, `HOPBOX_ACC_KERNEL`, `HOPBOX_ACC_KERNEL_SHA256`, and `HOPBOX_SSH_PRIVATE_KEY`. `HOPBOX_ACC_SSH_USER` sets the login user, `root` by default. |
