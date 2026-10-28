# The egress of the machines

> The egress of the machines through the edge servers. It covers the egress tunnels hbe*, the egress edges, the routing, and the local exit. It also covers the IPv6 addresses of the machines, IPv6 forwarding, and what the IPv6 block of an older release left. Then logs and tests.

The egress of a machine is its traffic to the internet. It leaves the install through the edge servers. So the internet sees the floating IPv4 address of an edge server, and the IPv6 address of its uplink. A node without the edge role sends the egress of its machines through egress tunnels. These are WireGuard devices on the peer addresses of the servers, beside the mesh device `hbm0` ([the mesh](/reference/mesh/index.html.md)). The owner decided the rules of this page on 2026-10-27, in the plan questions Q81 to Q87.

The tunnels are in `internal/egress`. The node table `inet hopbox_node` picks the tunnel of each packet ([the node network](/reference/node-network/index.html.md#the-table-inet-hopbox_node)). The host firewall of an edge server forwards and translates the egress ([the host firewall](/reference/host-firewall/index.html.md#the-egress-of-the-machines)).

In the examples, the vRack interface is `enp3s0f1`, and the uplink is `eno1`. `edge1` has the peer address `203.0.113.21`, the floating address `203.0.113.101`, and the uplink address `2001:db8:1::21`. `edge2` has `203.0.113.22`, `203.0.113.102`, and `2001:db8:2::22`.

## The way out of each server

| Server | Its egress tunnels | The way out of its machines |
| --- | --- | --- |
| A node without the edge role | One device `hbe<n>` for each egress edge, at most 16 ([The tunnels of a node](#the-tunnels-of-a-node)). | Through the tunnel that the hash of the machine address picks. Without a tunnel, the egress fails closed ([Fail closed](#fail-closed)). |
| An edge server without the node role | One device `hbe0`, with each node as a peer ([The tunnel of an edge server](#the-tunnel-of-an-edge-server)). | It has no machines. Its host firewall forwards the egress of the nodes. |
| A server with the node role and the edge role | One device `hbe0`, as an edge server. | The server is the local exit of its machines. They leave through its own uplink, without a tunnel ([The local exit](#the-local-exit)). |
| A server that enrolls with neither role, such as a server with the DNS role alone | None. A check removes the tunnels of an older config. | It has no machines. |

A server needs the mesh device for a tunnel, because the tunnels use its private key. While the device has no key, the log has `the mesh device has no key yet, so this server has no egress tunnel yet`. Each enrolled server makes the device, also a server without a peer address ([the mesh](/reference/mesh/index.html.md#which-servers-join)). Such a server gets no mesh, so a node without the edge role then knows no egress edge, and its egress fails closed. A server with the node role and the edge role, such as the first server of an install, keeps the egress of its machines as the local exit.

## Values

| Item | Value |
| --- | --- |
| Device | `hbe0`, `hbe1`, and so on: kernel WireGuard devices in the network namespace of `hopboxd`. The prefix of the names is `hbe`. |
| Private key | The private key of the mesh device `hbm0`. So the heartbeat carries no new key, and each peer knows the public key of the tunnel from the mesh. |
| MTU | `1420`, as for `hbm0`. A packet of a tap device fits ([the node network](/reference/node-network/index.html.md#addresses)). |
| UDP port | `51821` on an edge server. `51821` + n for the device `hbe<n>` of a node, so `51821` to `51836`. |
| Address of the device | None. The routes of the tables below name the device. |
| Most tunnels of a node | 16 |
| Routes and rules | The protocol `107`, so `ip route show table all proto 107` lists the routes of the tunnels. |
| Mark of tunnel n on a node | `0x68620000` + n, with the table `26820` + n and the rule priority `26820` + n. |
| Return mark on an edge server | `0x68620100`, with the table `26819` and the rule priority `26800`. |
| Conntrack mark on an edge server | The bit `0x20000000`, on each connection that comes out of a tunnel. It is for hopbox alone, as the mark of the node is ([the node network](/reference/node-network/index.html.md#the-mark-of-the-node-is-reserved)). |

The [host firewall](/reference/host-firewall/index.html.md#ports) opens UDP `51821` from the set `peers` on an edge server. It opens UDP `51821` to `51836` from the set `peers` on a node without the edge role. The vRack zone opens none of these ports.

## Egress edges

An egress edge is an edge server that takes the egress of the nodes now. The control role tells each node its egress edges in each mesh, with the field `egress` of each peer ([the control channel](/reference/control-channel/index.html.md#meshmember-meshpeers-and-meshpeer)). These conditions give a peer `egress` true, all of them together:

- It has the edge role.
- Its last heartbeat said `egress_exit`, and the control role recorded the version of that heartbeat in `servers.egress_exit_version` ([Control store tables](/reference/control-store/index.html.md#servers)).
- That version equals its `hopboxd_version`. So an edge server that went back to a release without the tunnels takes no egress.
- Its state is `ready`, `cordoned`, or `draining`.

An edge server says `egress_exit` in each heartbeat while its tunnel `hbe0` is ready, with the reverse path filter off ([the control channel](/reference/control-channel/index.html.md#heartbeat), [The reverse path filter](#the-reverse-path-filter)). After a restart, it keeps the tunnel that it finds, so it says `egress_exit` also before its first mesh.

The leader marks a silent edge server `unresponsive` after 20 s. The next heartbeat of each node then gets a mesh without that edge server. So the egress moves to the other egress edges within 20 s, one sweep of the leader, and one heartbeat of the node. `TestEgress` of the test fleet checks this bound with 40 s.

### A control role of an older release

A control role of a release before the egress tunnels never sets `egress`. A rollback can put a node of a newer release behind such a control role. So each mesh of a newer control role also has `egress_marked` ([the control channel](/reference/control-channel/index.html.md#meshmember-meshpeers-and-meshpeer)). A mesh without it says nothing about the egress edges. The node then keeps the egress tunnels that it has, and its machines keep their egress. It also keeps the tunnel of an edge server that stops, because no mesh of the older control role takes that edge away. So the machines whose hash picks that tunnel have no egress until the control role runs a release with the egress tunnels. An edge server still takes the nodes of such a mesh as the peers of its tunnel.

## The tunnels of a node

A node without the edge role has one tunnel for each egress edge, in the order of the mesh addresses of the edge servers. It takes the first 16 of them. The tunnel `hbe<n>` has these parts:

| Part | Value |
| --- | --- |
| Port | UDP `51821` + n. |
| Peer | The egress edge n, with the endpoint `<peer address of the edge>:51821`, and the allowed IPs `0.0.0.0/0` and `::/0`. The endpoint comes from the mesh ([the mesh](/reference/mesh/index.html.md#the-peers)). |
| Routing table | `26820` + n, with the default routes `0.0.0.0/0` and `::/0` on the device, both with the protocol 107. |
| Rule | `fwmark 0x68620000+n lookup 26820+n`, with the priority `26820` + n, for IPv4 and for IPv6. |
| Reverse path filter | `net.ipv4.conf.hbe<n>.rp_filter` is `0` ([The reverse path filter](#the-reverse-path-filter)). |

```console
$ ip rule show | grep 'proto 107'
26820:	from all fwmark 0x68620000 lookup 26820 proto 107
26821:	from all fwmark 0x68620001 lookup 26821 proto 107
$ ip route show table 26820
default dev hbe0 proto 107 scope link
$ sudo wg show hbe0
interface: hbe0
  public key: <public key of this server>
  private key: (hidden)
  listening port: 51821

peer: <public key of edge1>
  endpoint: 203.0.113.21:51821
  allowed ips: 0.0.0.0/0, ::/0
```

### The hash of a machine

The chain `egress` of the node table marks each packet of a machine with the mark of one tunnel ([the node network](/reference/node-network/index.html.md#the-table-inet-hopbox_node)). The mark is a Jenkins hash of the IPv4 address of the machine, modulo the count of tunnels, plus `0x68620000`. The seed of the hash is `0x6862`, so each install of the table picks the same tunnel. For IPv6, the hash reads the low 32 bits of the source address, which hold the IPv4 address of the machine. So both families of a machine take the same tunnel, and so the same edge server.

The chain leaves two kinds of packets without a mark, so they take the main table:

- The packets to the mesh range. They are the replies to the connections of the edge servers, which go back over `hbm0`.
- The packets to a resolver of the machines in a private range ([The resolvers of the machines](#the-resolvers-of-the-machines)).

### A change of the egress edges

The tunnels hand their count to the node network after each change. A change of the count replaces the rules of the chain `egress` alone, in one netlink batch. So the sets of the table and their counters stay. The order of the steps keeps each packet on a tunnel that exists:

- A new tunnel comes first, with its routes and its rules. Then the machines use it.
- The machines leave a tunnel first. Then the tunnel goes, after its rules and its routes.

A new count can move a machine to another edge server, and so to another source address.

## The tunnel of an edge server

An edge server has one tunnel, `hbe0`, on UDP `51821`. Each node of the mesh is a peer, without an endpoint. WireGuard learns the endpoint of a node from its first packet. The allowed IPs of a node are its prefixes outside the mesh range: its node subnet, the `/64` of that subnet, and the `/32` and the `/128` of each machine that moved to it ([the mesh](/reference/mesh/index.html.md#allowed-ips)). A node without a subnet yet is no peer. So a packet from a tunnel always has the source of a machine of that node.

The edge server routes the replies back into the tunnel, and not over the mesh, by a mark:

| Part | Value |
| --- | --- |
| Routing table | `26819`, with `10.240.0.0/12` and `fd65:1c74:e769::/48` on `hbe0`, both with the protocol 107. |
| Rule | `fwmark 0x68620100 lookup 26819`, with the priority `26800`, for IPv4 and for IPv6. It comes before the main table at `32766`. |
| Marks | The host firewall puts the conntrack mark bit `0x20000000` on each new connection from a tunnel. It puts the packet mark `0x68620100` on each reply of such a connection ([the host firewall](/reference/host-firewall/index.html.md#the-egress-of-the-machines)). |

The main table of the edge server routes the machines over `hbm0`. Without the mark, a reply to a machine takes the mesh. The node takes only the mesh address of an edge server as a source on `hbm0`, so it drops that reply.

```console
$ ip rule show table 26819
26800:	from all fwmark 0x68620100 lookup 26819 proto 107
$ ip route show table 26819
10.240.0.0/12 dev hbe0 proto 107 scope link
$ ip -6 route show table 26819
fd65:1c74:e769::/48 dev hbe0 proto 107 metric 1024 pref medium
```

### The source addresses

| Family | Source of the egress | Without the key |
| --- | --- | --- |
| IPv4 | `edge.public_ipv4`, the floating address of the edge server ([the bootstrap config of hopboxd](/reference/hopboxd-config/index.html.md#keys)). The Terraform provider writes it only for a release whose version names the feature `egress` ([hopbox Terraform provider](/reference/terraform-provider/index.html.md#releases-with-and-without-the-key)). | The address of the output interface (masquerade). `hopboxd` logs `this edge server has no edge.public_ipv4, so the egress of the machines leaves from the address of its output interface` at its start. |
| IPv6 | A global IPv6 address of the uplink, by NAT66 ([the host firewall](/reference/host-firewall/index.html.md#the-uplink-and-the-nat66)). Never the floating address of `edge.public_ipv6`. | The edge server refuses the IPv6 egress. This holds without an IPv6 default route, and without a global IPv6 address on the uplink ([IPv6 without an uplink address](#ipv6-without-an-uplink-address)). |

The IPv4 egress leaves through the uplink. The IPv6 egress leaves only through the uplink, the device of the IPv6 default route of the main table. When the edge server is replaced, the IPv6 source of its egress changes.

A machine also reaches the public addresses of its own install through its edge server, such as the URL of another machine (plan question Q52). A reply of the edge server itself takes the return mark too, so it goes back into the tunnel.

## The local exit

A server with the node role and the edge role makes no egress tunnel for its own machines (plan question Q81). They leave through its own uplink:

| Family | Source |
| --- | --- |
| IPv4 | `edge.public_ipv4`, or the address of the output interface without the key. The node table translates it. |
| IPv6 | A global IPv6 address of the uplink, by NAT66, as for the machines behind the tunnel. The host firewall translates it ([the host firewall](/reference/host-firewall/index.html.md#the-uplink-and-the-nat66)). Without an uplink address, the host firewall refuses the IPv6 egress. It also refuses an IPv6 packet to `2000::/3` that leaves through another device than the uplink. |

The edge role of the server still takes the egress tunnels of the other nodes. So an install of one server keeps its egress, and a bigger install can run the node role on its edge servers.

## Fail closed

A node without the edge role sends the egress of its machines only through a tunnel (plan question Q87). Without a tunnel, a packet of a machine takes the main table, and its output interface is not a tunnel. The node table then answers it at once: a TCP packet with a reset, and another packet with ICMP administratively prohibited. So a connect of the machine fails at once, and does not wait for a timeout. A node without an egress edge fails closed in this way. So does a node after a reboot, before its first mesh, and a node whose reverse path filter is on ([The reverse path filter](#the-reverse-path-filter)).

## IPv6 without an uplink address

An exit without an IPv6 default route, or without a global IPv6 address on its uplink, refuses the IPv6 egress of the machines. It answers with ICMPv6 administratively prohibited (plan question Q85). A TCP connect gets a reset, so it fails at once. The host firewall refuses it, for the tunnel and for the local exit. The machines still get their IPv6 address. `hopboxd` logs a warning at the start and at each change ([the host firewall](/reference/host-firewall/index.html.md#logs)).

## The resolvers of the machines

A resolver in a private range is one of the blocked ranges of the node table, such as `192.168.1.53`. A node sends the queries of its machines to such a resolver straight out, with the address of the node (masquerade). It sends the queries to a public resolver, such as `9.9.9.9`, through the tunnel, with the rest of the egress (plan question Q86). An edge server drops each private destination, so a private resolver works only from the node.

## The IPv6 addresses of the machines

Every install gives its machines addresses of one unique local prefix, `fd65:1c74:e769::/48` (plan question Q84). Its global ID was picked once at random, as RFC 4193 says. The address follows from the IPv4 address of the machine alone, so the control store keeps no IPv6 address.

| Item | Rule | Example |
| --- | --- | --- |
| Node subnet | The node subnet with the index i in `10.240.0.0/12` has the `/64` with the subnet ID i. | `10.240.0.0/20` has `fd65:1c74:e769::/64`, and `10.240.16.0/20` has `fd65:1c74:e769:1::/64`. |
| Machine | The `/128` of the `/64` of its subnet, with its IPv4 address in the low 32 bits. | `10.240.0.5` has `fd65:1c74:e769::af0:5`. |
| Gateway | `fe80::1`, on-link. The node holds it on each tap device. | |

A moved machine keeps its IPv6 address, as it keeps its IPv4 address. The mesh allows the `/64` of each node and the `/128` of each moved machine ([the mesh](/reference/mesh/index.html.md#allowed-ips)). The edge servers translate the address to the IPv6 address of their uplink, so it never leaves the install.

The node gives the guest its IPv6 address and its gateway in the fields `address6` and `gateway6` of the network of the guest link ([Guest link](/reference/guest-link/index.html.md)). A node whose kernel has no IPv6 gives none.

## IPv6 forwarding

The IPv6 egress needs IPv6 forwarding. In Linux, it is one switch for the whole host. Each node and each edge server turns on `net.ipv6.conf.all.forwarding` at its start. It does so only after its table drops the forwarded IPv6 traffic of the other interfaces ([the node network](/reference/node-network/index.html.md#the-forward-guard), [the host firewall](/reference/host-firewall/index.html.md#the-egress-of-the-machines)).

With forwarding on, the kernel takes no router advertisement on an interface whose `accept_ra` is `1`. It then removes the default routes that such advertisements gave. So before it turns forwarding on, `hopboxd` sets `accept_ra` to `2` on each such interface, and on `default` for new interfaces. The main table keeps the IPv6 default route of the uplink, and the own traffic of the server keeps using it. Loopback and `all` stay as they are.

The node network logs `turned on IPv6 forwarding for the machines, and keeps the router advertisements of the interfaces`, and the host firewall of an edge server logs `turned on IPv6 forwarding for the egress of the machines, and keeps the router advertisements of the interfaces`. The field `accept_ra_2` lists the interfaces that changed. A host without IPv6 turns nothing on. When the host firewall of an edge server finds IPv6 forwarding off, it writes the file `<data_dir>/hostfw/forward-guard-ipv6`, then turns forwarding on. A move of the edge server to a release from before the egress turns IPv6 forwarding off only when that file exists ([hopbox Terraform provider](/reference/terraform-provider/index.html.md#releases-with-and-without-the-key)).

## The reverse path filter

A tunnel has no address. The kernel drops each packet of a device without an address under any reverse path filter, also the loose one. So `hopboxd` sets `net.ipv4.conf.hbe<n>.rp_filter` to `0` when it sets up a tunnel. Each check reads the value of each tunnel again, and sets it to `0` again after a write of another tool, such as a rule of `systemd-sysctl` for `net.ipv4.conf.*`. That write leaves the WireGuard sessions of the tunnel as they are.

The kernel takes the larger value of the device and of `all`. So `net.ipv4.conf.all.rp_filter` must be `0` too. While the filter of `all` or of a tunnel is on, an edge server takes no egress: its heartbeats say no `egress_exit`, so the nodes leave it at their next mesh ([Egress edges](#egress-edges)). A node with the filter on hands its node table no tunnel, so its machines fail closed with a refusal, and do not wait for a timeout ([Fail closed](#fail-closed)). `hopboxd` logs this error once for each new state, with the values in the field `rp_filter`, such as `all=2`:

```text
the reverse path filter is on, so the kernel drops the packets of the egress tunnels, and this server takes no egress. Set the sysctl net.ipv4.conf.all.rp_filter to 0
```

To fix it, set the sysctl, and keep it across a reboot:

```sh
echo 'net.ipv4.conf.all.rp_filter = 0' | sudo tee /etc/sysctl.d/90-rp-filter.conf
sudo sysctl --system
```

## Checks, a restart, and a reboot

The unit `channel` of `hopboxd run` checks the tunnels at its start, and then every 2 s ([hopboxd](/reference/hopboxd/index.html.md#run)). A check makes each device that is missing, and sets its key, port, MTU, and peers. It sets the routes and the rules, and removes the routes and the rules with the protocol 107 that it does not want. It removes the WireGuard devices `hbe*` that it does not want. Each new mesh checks the tunnels too.

| Event | What happens to the tunnels |
| --- | --- |
| A restart or an upgrade of `hopboxd` | The tunnels stay, with their routes and their rules, and the kernel goes on forwarding. Before the first mesh of the new process, a check keeps the tunnels that it finds. The node network takes the count of the tunnels `hbe0`, `hbe1`, and so on that are up, until the first gap. While the reverse path filter is on, of `all` or of one of these tunnels, the count is 0, because the kernel drops the replies. So the egress goes on also while no control server runs. |
| A stop of `hopboxd` | The tunnels stay, and the egress goes on. |
| A reboot | The tunnels are gone. The mesh device gets a new key, so the tunnels wait for the first mesh. Until then, the egress of a node fails closed, and an edge server says no `egress_exit`. |
| `ip link del hbe0` | The next check, within 2 s, makes the device again, with its routes. |
| A change of a device, a route, or a rule outside `hopboxd` | The next check sets it again, and logs `the egress tunnels lost a part outside hopboxd, so hopboxd set it again` with the parts: `tunnels`, `routes`, or `rules`. |

Do not manage the devices `hbe*` with `wg-quick`, `systemd-networkd`, or another tool. `hopboxd` sets them back at each check.

## Check the egress

On a node, list the tunnels and their rules:

```sh
sudo wg show all endpoints
ip rule show | grep 'proto 107'
sudo nft list chain inet hopbox_node egress
```

On an edge server, list the nodes of its tunnel:

```sh
sudo wg show hbe0 allowed-ips
```

[Check the network of a machine](/guide/node-network/index.html.md#check-the-egress) checks the egress from a machine.

## Upgrades

A rolling upgrade moves the edge servers before the nodes ([hopbox Terraform provider](/reference/terraform-provider/index.html.md#rolling-upgrade)). The egress of the machines depends on that order:

- An edge server of the new release runs its tunnel, and says `egress_exit`, before any node uses it. A node of the old release masquerades its machines to its own address until its own upgrade.
- A node that moves to the release with the egress tunnels has no tunnel at its first start. Its node table fails closed until its first mesh, about one heartbeat later, when it makes its tunnels. So for a few seconds, each new connection of its machines gets a refusal. The connections that run before the upgrade stop too, because their source changes from the address of the node to the address of an edge server.
- A node of the new release behind a control role of the old release keeps its tunnels ([A control role of an older release](#a-control-role-of-an-older-release)).

### What the IPv6 block left

A release before N29 gave the IPv6 egress an address of an IPv6 block in the vRack. On an edge server with the block, it kept these parts: an address of the block on the vRack interface, and the routing table `hopbox_egress6`, number `26806`. That table had two rules, `from <block>` at the priority `26806` and `from fd65:1c74:e769::/48` at `26807`. The current release sends the IPv6 egress out of the uplink, so these parts must go.

At its start, each server that enrolls removes them, in this order:

1. The addresses on any device that are in a block that a rule of table `26806` names, with the prefix length of that block. The old release set its address in that form. A floating address of the same block, such as a `/128` for the `AAAA` records, stays.
2. Each route of table `26806`, also its `unreachable` default route.
3. Each rule of table `26806`. The rules go last, because they name the blocks, so a failed step keeps them for the next check.

The host firewall replaces the table `inet hopbox_host` as a whole at its first install. So its old chain `postrouting` goes too ([the host firewall](/reference/host-firewall/index.html.md#the-table)). After a failure, the server tries again every 2 s, until a check finds nothing left. A failure never stops `hopboxd`.

To check that nothing is left on a server, run these commands. Each one prints nothing:

```sh
ip -6 rule show table 26806
ip -6 route show table 26806
```

## Logs

The tunnels log with `component=egress`. A check logs a state only after a change of the state, so a check every 2 s does not repeat a line.

| Message | Level | Fields | When |
| --- | --- | --- | --- |
| `cannot read the egress tunnels. The next check tries again` | error | `err`, `retry_in` | The read of the devices failed. |
| `cannot read the key of the mesh device, so the egress tunnels stay as they are. The next check tries again` | error | `err` | The read of the key of `hbm0` failed. |
| `the mesh device has no key yet, so this server has no egress tunnel yet` | info | `device` | `hbm0` does not exist yet, such as on a server outside the mesh. |
| `cannot set up the egress tunnels. The next check tries again` | error | `mode`, `err`, `retry_in` | A change of the kernel failed. `mode` is `node`, `edge`, or `none`. |
| `the egress tunnels are ready` | info | `mode`, `tunnels` | A check set every tunnel, for the first time or after a change. `tunnels` lists the devices. |
| `the egress tunnels lost a part outside hopboxd, so hopboxd set it again` | warn | `mode`, `parts` | A check set again what changed outside `hopboxd`. |
| `the reverse path filter is on, so the kernel drops the packets of the egress tunnels, and this server takes no egress. Set the sysctl net.ipv4.conf.all.rp_filter to 0` | error | `rp_filter` | See [The reverse path filter](#the-reverse-path-filter). |
| `the machines of this node send their egress through the egress tunnels` | info | `tunnels` | The count of tunnels of a node changed, and the tunnels hand it to the node network. |
| `removed what the IPv6 block of the vRack left on this server, because this release sends the IPv6 egress of the machines out of the uplink` | info | `removed` | A check removed parts of the IPv6 block of a release before N29 ([What the IPv6 block left](#what-the-ipv6-block-left)). `removed` lists the addresses, the count of routes, and the rules. |
| `cannot remove what the IPv6 block of the vRack left on this server. The next check tries again` | error | `err`, `retry_in` | A read or a removal failed. It logs once until the error changes. |

The other parts log with their own components:

| Message | Level | Component | When |
| --- | --- | --- | --- |
| `the machines send their egress through the egress tunnels` | info | `netcfg` | The node table took a new count of tunnels, in the field `tunnels`. |
| `the node network cannot take the egress tunnels. The next change of the tunnels tries again` | error | `netcfg` | The change of the chain `egress` failed, with `err`. |
| `this edge server has no edge.public_ipv4, so the egress of the machines leaves from the address of its output interface` | warn | `hostfw` | The start of an edge server without `edge.public_ipv4`. |
| `the IPv6 egress of the machines leaves from the uplink address of this edge server` | info | `hostfw` | The start of an edge server, or a new uplink or uplink address, with `device` and `address`. |
| `this edge server has no IPv6 default route, so the IPv6 egress of the machines gets ICMPv6 administratively prohibited, and TCP a reset` | warn | `hostfw` | The start of an edge server without an IPv6 default route, or the loss of that route. |
| `the uplink of this edge server has no global IPv6 address, so the IPv6 egress of the machines gets ICMPv6 administratively prohibited, and TCP a reset` | warn | `hostfw` | The start of an edge server whose uplink has no global IPv6 address, or the loss of that address, with `device`. |

## Errors

These errors come in the field `err` of `cannot set up the egress tunnels. The next check tries again`:

| Error | Cause |
| --- | --- |
| `set up the egress tunnel <name>: make the device: ... hopboxd needs CAP_NET_ADMIN and the kernel module wireguard` | The kernel has no WireGuard, or `hopboxd` runs without `CAP_NET_ADMIN`. |
| `set up the egress tunnel <name>: the device <name> is a device of type <type>, not wireguard. Remove it, and hopboxd makes the egress tunnel` | Another device took the name. |
| `set up the egress tunnel <name>: set the key, the port, and the peers: ...` | WireGuard refused the change, such as a port that another program holds. |

## Tests

| Test | Where it runs | What it checks |
| --- | --- | --- |
| The tests of `internal/egress` without root | `make test`, also on a Mac | One tunnel for each egress edge, at most 16, and the order of a new tunnel and of its use. A restart keeps the tunnels until the first mesh, and counts none while the filter of a tunnel is on. No mesh key, no tunnel. One tunnel with each node on an edge server. A change of mode removes the old tunnels. A repair and its log, a failure, the reverse path filter, and `egress_exit`. The router advertisements of IPv6 forwarding. The pick of the uplink address of the NAT66. The removal of what the IPv6 block left, with the rules last, and nothing to do on a new server. |
| `TestFindUplink` of `internal/egress` | `make test-linux` | An uplink with a static address, an address of a router advertisement, the floating address, and a unique local address. The pick takes the static address, or the preferred source of the default route, and never the floating address. |
| `TestLeftoversRemoveTheIPv6Block` of `internal/egress` | `make test-linux` | What a release before N29 left on an edge server: the address of the block, the table `hopbox_egress6` with its two default routes, and its two rules. A check removes them, and keeps the other addresses, routes, and rules. |
| `TestTheTunnelsCarryTheEgressBothWays` of `internal/egress` | `make test-linux`, as root in a privileged container | A node and an edge server on one vRack link, each with its mesh device and the real kernel WireGuard. A packet of a machine with the mark of its tunnel reaches the edge server with the address of the machine. The reply with the return mark comes back through the tunnel. It checks IPv4 and IPv6. |
| `TestIPv6ForwardingKeepsTheDefaultRouteOfTheUplink` of `internal/egress` | `make test-linux` | A host whose uplink takes its IPv6 default route from a real router advertisement. After IPv6 forwarding turns on, the route stays, and a new advertisement renews it. |
| `TestEdgeEgress` and `TestLocalExitIPv6` of `internal/hostfw` | `make test-linux` | The host firewall of an edge server, and the NAT66 of a local exit ([the host firewall](/reference/host-firewall/index.html.md#tests)). |
| `TestNodeNetwork` and `TestLocalExit` of `internal/node/netcfg` | `make test-linux` | The node table with a stand-in tunnel, and a local exit ([the node network](/reference/node-network/index.html.md#tests)). |
| `TestEgressEdges` of `internal/service` | `make test` | The field `egress` of a peer: the heartbeat, the edge role, an unresponsive edge server, and another release. |
| `TestEgress` of `test/fleet` | `make test-fleet` | Two edge servers and a node. |
| `TestEgressWithoutAFloatingAddress` of `test/fleet` | `make test-fleet` | An edge server with the node role and no `edge.public_ipv4`, and a node. |
