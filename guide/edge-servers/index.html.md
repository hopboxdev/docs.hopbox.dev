# Run edge servers apart from the nodes

> How to add two edge servers apart from the nodes, with floating IPv4 addresses and IPv6 addresses of the vRack block, point the zone at both, and reach a machine through each one.

The network redesign of 2026-10-27 changes this page. hopbox leaves the vRack and the OVHcloud API, and the servers talk over their public addresses (Q90). This page still describes the current code. Milestones N26 to N32 change the code and this page. Before you set up a fleet on the vRack or with an IPv6 /56, read Q90.

An edge server takes the public traffic of an install: the URL of each machine, SSH, the API, and the console. A small install runs every role on one server. A bigger install runs the edge role and the DNS role on two servers of their own, `edge1` and `edge2`. The install then has two name servers. When one edge server stops, the other one takes the traffic. This guide adds two edge servers to an install on the vRack, and checks a machine through each one.

An edge server without the node role reaches each machine over the mesh, the WireGuard device `hbm0` between the servers ([the mesh](/reference/mesh/index.html.md)). [The edge role](/reference/edge/index.html.md#the-way-to-the-machines) covers the way to the machines in full.

The release files of hopbox are not public yet. The hopbox team gives them on request.

## How a client reaches a machine

1. The client asks DNS for `box1.example.com`. Each edge server answers with the DNS role. The zone names the floating IPv4 address of each edge server, and its IPv6 address. `ns1` is `edge1`, and `ns2` is `edge2`.
2. The client connects to one of the addresses. It uses HTTPS for a machine URL or the API, and SSH for the SSH port of the machine.
3. The edge server finds the node of the machine in its routes. It connects to the address of the machine over `hbm0`, from its mesh address.
4. The node lets the connection reach the machine, because the mesh address belongs to an edge server.

A floating address is an Additional IP of OVHcloud: a public IPv4 address that the account routes to one server. A replacement of the edge server takes the same address, so the zone never changes. The IPv6 addresses come from the IPv6 block of the install, a `/56` that OVHcloud routes into the vRack, not to one server. Each edge server holds one address of the block on its vRack interface.

## What you need

- An install with a control server on the vRack, and nodes in the mesh ([the mesh](/reference/mesh/index.html.md#which-servers-join)).
- Two servers for `edge1` and `edge2`, each with an address on the vRack in `172.31.0.0/16`. The Terraform provider sets it ([hopbox Terraform provider](/reference/terraform-provider/index.html.md)).
- One floating IPv4 address for each edge server, routed to it and held on its public interface. The Terraform provider routes it with the attribute `floating_addresses`. This guide uses `203.0.113.101` for `edge1` and `203.0.113.102` for `edge2`.
- The IPv6 block of the install, routed into the vRack. This guide uses the block `2001:db8:0:100::/56` with the gateway `2001:db8:0:100::1`. `edge1` takes `2001:db8:0:100::11`, and `edge2` takes `2001:db8:0:100::12`.
- The internal URL of the control server, such as `https://172.31.0.11:7443`.
- `dig`, `curl`, and `ssh` on your laptop.

## Add the edge servers

### Write the bootstrap config of each edge server

On `edge1`, write `/etc/hopbox/hopboxd.toml` before the install. The install keeps the keys of the file ([hopboxd](/reference/hopboxd/index.html.md#install)):

```toml
roles = ["edge", "dns"]

[dns]
listen = ["203.0.113.101:53", "[2001:db8:0:100::11]:53"]

[edge]
public_ipv6 = "2001:db8:0:100::11/56"
public_ipv6_gateway = "2001:db8:0:100::1"
```

On `edge2`, write the same file with `203.0.113.102` and `2001:db8:0:100::12`.

The tested OS runs `systemd-resolved` on port 53 of a loopback address. So `dns.listen` names the public addresses of the server, not every address ([the DNS role](/reference/dns/index.html.md#listeners)). The DNS role takes queries on the IPv6 address once `hopboxd` sets it. `edge.public_ipv6` names the address of the server with the prefix length of the block ([the bootstrap config of hopboxd](/reference/hopboxd-config/index.html.md#keys)).

### Install and enroll each edge server

On the control server, make a token for each edge server:

```sh
sudo hopboxd admin enrollment-token create --roles edge,dns
```

On `edge1`, install with the URL of the control server, and enroll with the token:

```sh
sudo hopboxd install --roles edge,dns --control-url https://172.31.0.11:7443
sudo hopboxd enroll --token <token>
```

Do the same on `edge2` with its own token.

### Check the mesh and the IPv6 address

On the control server, list the servers. Each edge server is `ready`:

```sh
sudo hopboxd admin server list
```

`hopboxd admin server get <id>` shows the mesh address of each edge server in `mesh_address` ([hopboxd](/reference/hopboxd/index.html.md#admin-server-get)).

On `edge1`, check the IPv6 address, the rule, and the route of the table `hopbox_egress6`, table number 26806:

```sh
ip -6 addr show to 2001:db8:0:100::/56
ip -6 rule show table 26806
ip -6 route show table 26806
```

The vRack interface holds `2001:db8:0:100::11/56`. The rule sends each packet from the block to the table, and the table holds one default route via `2001:db8:0:100::1`. So the replies of the IPv6 address leave through the vRack, not through the uplink. The log of `hopboxd` shows the line `this edge server holds its public IPv6 address on the vRack` ([the edge role](/reference/edge/index.html.md#the-public-ipv6-address)).

## Point the zone at both edge servers

### Set the public addresses

The zone names the public addresses of the install. `ns1` takes the first address of each family, and `ns2` the second ([the DNS role](/reference/dns/index.html.md#the-zone-of-the-install)). So list `edge1` first:

```sh
sudo hopboxd admin config set --public-address 203.0.113.101,203.0.113.102,2001:db8:0:100::11,2001:db8:0:100::12
```

### Check each name server

Ask each edge server at each of its addresses. Each answer has the flag `aa`, and names both edge servers:

```sh
dig @203.0.113.101 box1.example.com A +norec
dig @2001:db8:0:100::12 box1.example.com AAAA +norec
dig @203.0.113.102 example.com NS +norec
```

The `NS` answer names `ns1.example.com` and `ns2.example.com`. Its additional section gives `ns1` the addresses of `edge1`, and `ns2` the addresses of `edge2`.

Each heartbeat of an edge server reports the serial of the zone that it serves. Both edge servers report the same serial:

```console
$ sudo hopboxd admin server get <id of edge1> | grep dns_zone_serial
dns_zone_serial	1792249094
```

### Delegate the domain

Give the parent zone the two name servers and their glue records: `ns1` at the addresses of `edge1`, and `ns2` at the addresses of `edge2` ([Serve the domain of an install with the DNS role](/guide/dns-role/index.html.md#make-the-records-in-the-parent-zone)). If the domain points at another server now, change the glue records last. A resolver keeps an answer for 60 seconds.

## Reach a machine through each edge server

The certificate of the install reaches both edge servers over the control channel ([Get the wildcard certificate of an install](/guide/acme-certificate/index.html.md)). Open the URL of `box1` through each edge server. `--resolve` sends the request to one address:

```sh
curl --resolve box1.example.com:443:203.0.113.101 https://box1.example.com/
curl --resolve 'box1.example.com:443:[2001:db8:0:100::12]' https://box1.example.com/
```

Open SSH through each edge server, at the SSH port of `box1`, such as 10022 ([Reach a machine over SSH](/guide/ssh/index.html.md)):

```sh
ssh -p 10022 user@203.0.113.101
ssh -p 10022 user@2001:db8:0:100::12
```

The code of the machine sees the mesh address of the edge server as the client, such as `10.239.0.3`. The header `X-Forwarded-For` of a request holds the address of the real client ([the edge role](/reference/edge/index.html.md#what-a-machine-gets)).

## When an edge server stops

The other edge server goes on: it answers DNS, the machine URLs, SSH, and the API. A resolver that gets no answer from one name server asks the other one. The address of the stopped edge server gives no answer to a client. It answers again when the edge server is back, or when its replacement takes the address.

A DNS server keeps the last zones of the control plane on its disk ([the DNS role](/reference/dns/index.html.md#zones-on-the-disk)). So an edge server that restarts while no control server runs answers with those zones. Its log shows `serving the dns zones on the disk until the control plane sends its zones`.

## Replace an edge server

The new server takes the same addresses as the old one:

1. The Terraform provider moves the floating IPv4 address to the new server.
2. Write the bootstrap config of the new server with the same `edge.public_ipv6`, then install and enroll it as above.
3. While the old server still holds the IPv6 address, duplicate address detection fails on the new server. The new server logs `another host on the vRack holds the public IPv6 address of this edge server, so it takes no traffic`, and tries the address again at each check, every 2 seconds. Once the old server is gone, the new server takes the address.

The public addresses of the install stay the same, so the zone and the glue records need no change.
