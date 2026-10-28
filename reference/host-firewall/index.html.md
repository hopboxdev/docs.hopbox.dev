# The host firewall

> The host firewall of every server, the nftables table inet hopbox_host. It covers the zones and ports, the jump hosts, and the ways in that it keeps. It also covers the egress chains of an edge server, the uplink and the NAT66 of the IPv6 egress, the repair, metrics, logs, and tests.

The network redesign of 2026-10-27 changes this page. hopbox leaves the vRack and the OVHcloud API, and the servers talk over their public addresses (Q90). This page describes the code after milestone N29, which sends the IPv6 egress of the machines out of the uplink of the edge server. The rules of the vRack zone stay until N31. Milestones N30 to N32 change the code and this page again. Before you set up a fleet on the vRack, read Q90.

The host firewall is the nftables table `inet hopbox_host` of every server of an install. It makes the edge servers the bastion of the install (rules 2 and 5). Only an edge server takes traffic from the whole internet. On its public address, a control server or a node takes only the traffic of the install, from the peer addresses of the servers ([Sets](#sets)). A peer address is the public IPv4 address at which the other servers reach a server ([Enrollment and the install PKI](/reference/enrollment/index.html.md)). On the vRack and on the mesh, each server takes only the ports of its roles. Operators reach a server that is not an edge server by SSH only through an edge server as a jump host. On an edge server, the table also forwards and translates the egress of the machines ([The egress of the machines](#the-egress-of-the-machines)).

The firewall is in `internal/hostfw`. `hopboxd run` starts it on every server, right after the metrics listener and before the units of the roles ([hopboxd](/reference/hopboxd/index.html.md#run)). The provider block of the Terraform provider names the jump host in `ssh_jump_host` ([hopbox Terraform provider](/reference/terraform-provider/index.html.md#jump-host)).

In the examples, the vRack interface is `enp3s0f1`. The edge servers have the peer addresses `203.0.113.21` and `203.0.113.22`, and the vRack addresses `172.31.0.21` and `172.31.0.22`.

## Zones

The table sorts each new connection by the interface that takes it. A zone is the set of interfaces that share one list of open ports.

| Zone | Interfaces | Sources that a port takes |
| --- | --- | --- |
| Loopback | `lo` | Every source. The table takes every packet of loopback. |
| vRack | The interface that holds the one address of the server in the vRack range | The vRack range `172.31.0.0/16`, or the vRack jump hosts for port 22. |
| Mesh | `hbm0` | The mesh range `10.239.0.0/16`. |
| Tap devices | The tap devices of the machines on a node, `hbt*` | The table takes them, and leaves them to the table `inet hopbox_node` ([the node network](/reference/node-network/index.html.md#the-table-inet-hopbox_node)), which owns all traffic of the machines. |
| Public | Each other interface, such as the uplink. The egress tunnels `hbe*` are in it too, so a machine reaches the public ports of an edge server through its tunnel. | Every source for the ports of the roles `edge` and `dns`. Each port of the traffic of the install takes only its set ([Sets](#sets)). |

A machine behind an egress tunnel reaches the public ports of its edge server at the floating addresses, as the internet does. It reaches no port at a peer address of the install. Its own edge server is no exception ([The egress of the machines](#the-egress-of-the-machines)). Each zone ends with a drop, so a port that no role opens takes nothing. The replies of the connections of the server pass in every zone, and so do the ICMP messages of [ICMP and DHCPv6](#icmp-and-dhcpv6).

## Ports

Each role opens its ports. A server with more than one role opens the ports of each role.

| Role | Zone | Port | Source | Use |
| --- | --- | --- | --- | --- |
| A server without the role `edge` | Public | TCP 22 | The set `jump_hosts`. While the server never knew an edge server and never had a vRack address, every source ([Never a lockout](#never-a-lockout)). | The SSH of the operators and of the provider through an edge server ([The jump hosts](#the-jump-hosts)). |
| Every server | vRack | TCP 22 | The set `vrack_jump_hosts`. While it is empty, the vRack range. | The SSH through an edge server on the vRack. |
| A server that enrolls: `node`, `dns`, or `edge` | Public | UDP 51820 | The set `peers` | The WireGuard packets of the mesh, which runs on the peer addresses ([the mesh](/reference/mesh/index.html.md#values)). |
| A server that enrolls | vRack | UDP 51820 | The vRack range | Nothing since N28. N31 removes the vRack zone. |
| `edge`, on a server that enrolls | public | UDP 51821 | The set `peers` | The egress tunnel `hbe0`, which takes the egress of the nodes ([the egress of the machines](/reference/egress/index.html.md#the-tunnel-of-an-edge-server)). |
| `node` without `edge`, on a server that enrolls | public | UDP 51821 to 51836 | The set `peers` | The egress tunnels `hbe0` to `hbe15` of the node ([the egress of the machines](/reference/egress/index.html.md#the-tunnels-of-a-node)). |
| `control` | Public | TCP 7443, the port of `control.internal_listen` | The sets `peers` and `registered` | The internal listener on the peer address of the control server: the enrollment, the control channels, and the API and the console that the edge servers forward ([the internal listener](/reference/internal-listener/index.html.md)). |
| `control` | vRack | TCP 7443, the port of `control.internal_listen` | The vRack range | The same, for a listener on the vRack address of a release before N28. N31 removes the vRack zone. |
| `node` | Mesh | TCP 7444 | The mesh range | The disk listener of the cold moves ([Cold move](/reference/cold-move/index.html.md#the-disk-listener)). |
| `edge` | Public | TCP 22 | Every source | The `sshd` of the operators. The edge server is the jump host. |
| `edge` | Public | TCP 80 and 443 | Every source | The HTTP and HTTPS listeners ([the edge role](/reference/edge/index.html.md#listeners)). |
| `edge` | Public | TCP 2222, the port of `edge.ssh_listen`, only for connections that a redirect sent there | Every source | The SSH ports of the machines, TCP 10000 to 30000. The table `inet hopbox_edge` sends them to `edge.ssh_listen` ([the SSH front door](/reference/ssh-front-door/index.html.md)), so the input hook sees port 2222. When conntrack records the redirect (`ct status dnat`), the rule takes the connection there. Any redirect or destination NAT to that port passes, also one that an operator with `edge.ssh_redirect = false` writes. A connection straight to port 2222 takes nothing. |
| `dns` | Public | TCP and UDP 53 | Every source | The DNS role ([the DNS role](/reference/dns/index.html.md)). |

An edge server takes no new connection on the vRack other than its ports of the vRack zone. The edge servers of an install run the DNS role. Port 53 follows the role `dns`, not the role `edge`, as the owner decided on 2026-10-27 (plan question Q74). So a server with the DNS role alone, as in [Serve the domain of an install with the DNS role](/guide/dns-role/index.html.md), opens port 53 and no edge port. An edge server without the DNS role keeps port 53 closed. The table opens the same ports whatever the listen addresses of the config say, as the owner decided on 2026-10-27 (plan question Q80). A listener on another port, such as `edge.https_listen = ":8443"`, takes no connection from outside the server. So at its start, `hopboxd` logs `the host firewall opens no public port for this listener, so it takes no connection from outside the server` at level warn, for each such listener of `edge.https_listen`, `edge.http_listen`, and `dns.listen`. A listener on loopback gets no warning.

The other listeners of `hopboxd` take nothing from outside the server: the API listener on `127.0.0.1:7080`, the metrics listener on `127.0.0.1:9464`, and the listener of the SSH front door on port 2222.

## Sets

The table holds up to five sets of IPv4 addresses. A port of the traffic of the install takes only the addresses of its set.

| Set | Holds | Ports that take it |
| --- | --- | --- |
| `peers` | The peer addresses of the servers of the install that are not decommissioned ([Control store tables](/reference/control-store/index.html.md#servers)). A server without a peer address is not in it. It also holds the address of the internal listener of each live control process on a peer address ([Control store tables](/reference/control-store/index.html.md#control_processes)). A server with the control role alone has no row in `servers`, so this address lets the control servers reach the internal listeners of each other. This is a build default of N28, not an owner decision. | Public UDP 51820, the UDP ports of the egress tunnels, and TCP 7443 of a control server. |
| `jump_hosts` | The peer addresses of the edge servers among them. | Public TCP 22 of a server without the role `edge`. |
| `registered` | The peer addresses of the enrollment tokens that are not used and not expired ([Enrollment and the install PKI](/reference/enrollment/index.html.md)). Only a server with the control role has this set. | Public TCP 7443 of a control server. |
| `known_peers` | Every peer address that the install knows, and each address of `peers`. These are the peer addresses of the servers that are not decommissioned, and the address of the internal listener of each row of `control_processes`, live or not. An address leaves only with the decommission of its server or with the sweep of its row, 24 hours after the last renewal, never with a missed heartbeat. Only a server with the egress of the machines, an edge server, has this set. A control role of an older release sends no known peers, and the set then holds `peers` alone. This is a build default of N28, not an owner decision. | None. The egress of the machines reaches none of its addresses ([The egress of the machines](#the-egress-of-the-machines)). |
| `vrack_jump_hosts` | The vRack addresses of the edge servers, as the table before the start of `hopboxd` held them. Nothing sets it since N28. N31 removes it with the vRack zone. | TCP 22 of the vRack interface. |

A registered address reaches the internal listener only while its token works. So a new server enrolls from its peer address, and its later control channel comes from the same address, which `peers` then holds. A peer address stays in `peers` until its server is decommissioned. Providers reuse addresses, so a stranger can get the address of a decommissioned server.

The sets come from two places:

| Server | Where the sets come from |
| --- | --- |
| A server with the control role | The control role reads `peers`, `jump_hosts`, `known_peers`, and `registered` from the store every second ([Control store tables](/reference/control-store/index.html.md#servers)). So a new enrollment token reaches `registered` within 2 s, and the Terraform provider waits for that before it enrolls a server. The control role hands the sets to the firewall only after a change. |
| A server without the control role | Each mesh that the control role sends holds `peers`, `jump_hosts`, and `known_peers` in its field `firewall` ([the control channel](/reference/control-channel/index.html.md#mesh-pushes)). A change reaches the server within one heartbeat, 5 s. The agent hands the sets to the firewall before the mesh device takes the mesh. A mesh of a control role of an older release has no `firewall`, and the server keeps the sets that it has. |

The set `vrack_jump_hosts` comes from the table before the start ([The jump hosts](#the-jump-hosts)).

Each server keeps its last `peers`, `jump_hosts`, and known peers in the file `<data_dir>/hostfw/peers` ([Server layout](/reference/server-layout/index.html.md#paths)), and writes it after each change, through a temp file and a rename. The start of `hopboxd` loads the file before the control channel comes back. So a restart without a control server keeps the mesh port and the way in through the jump hosts. Without the file, the start takes the sets of the table before it. It never takes `registered` from the file or the table: the control role reads it within a second, and each registration ends with its token.

The file has one address on each line, after its kind. The line `knew_edge` records that the server knew an edge server ([Never a lockout](#never-a-lockout)). The line `known_peers_whole` records that the known peers came from the control role ([The egress of the machines](#the-egress-of-the-machines)). A line that starts with `#` is a comment:

```text
# The last peers, jump hosts, and known peers of the host firewall of this server. hopboxd writes this file.
knew_edge
known_peers_whole
peer 203.0.113.21
peer 203.0.113.22
peer 203.0.113.31
jump_host 203.0.113.21
jump_host 203.0.113.22
known_peer 203.0.113.11
known_peer 203.0.113.21
known_peer 203.0.113.22
known_peer 203.0.113.31
```

Each check compares the elements of each set with the sets of the firewall. So a flush of a set alone comes back within 2 s, and so does a delete of an element. Each new set from the control role or from a mesh checks the table again, also the same sets.

## The jump hosts

Port 22 of the public interface of a server without the role `edge` takes the peer addresses of the edge servers alone, the set `jump_hosts`. An edge server reaches a peer address straight over the internet. So the SSH of an operator through an edge server comes from the peer address of the edge server. The Terraform provider dials the peer address of a server through the jump host too.

An edge list that turns empty keeps the last `jump_hosts`, as the owner decided on 2026-10-27 (plan question Q93). So the last edge server that leaves stays a jump host of each server, and `hopboxd` logs `the install has no edge server now, so port 22 of the public interface keeps the last jump hosts` at level warn. A new edge server replaces the list. The kept address has the risk of reuse of the peers ([Sets](#sets)): a stranger who gets the address of the last edge server that left reaches port 22 of each server, but still needs an SSH key.

Port 22 of the vRack zone takes the vRack addresses of the edge servers alone, the set `vrack_jump_hosts`. Since N28, the mesh runs on the peer addresses, so neither the mesh nor the control role knows a vRack address. The start of `hopboxd` takes the set of the table before it, and nothing changes the set after that. A table of a release before N27 holds the vRack addresses in `jump_hosts`, and the start takes them as the vRack jump hosts. While the set is empty, port 22 of the vRack zone takes the whole vRack range. N31 removes the vRack zone.

## Never a lockout

A change of the table never closes the way of an operator into a server in a way that this section does not name. The owner decided the ways of SSH on 2026-10-27 (plan question Q93, which replaces Q75):

- A server that never knew an edge server and never had a vRack address keeps port 22 open on its public interface, to every source. No jump host reaches it. This covers a new server before its first control channel, and a first server without the edge role, such as ks. A server knew an edge server once its set `jump_hosts` or its set `vrack_jump_hosts` had an address. So a server whose table of a release before N28 held vRack jump hosts keeps port 22 of its public interface closed.
- A server that has a vRack address, or had one, keeps port 22 of its public interface closed to every address outside `jump_hosts`, also before it knew an edge server. A jump host reaches it over the vRack, where port 22 takes the vRack jump hosts that the table held at the start, or the whole vRack range while that set is empty, until N31 removes the vRack zone. The owner decided this on 2026-10-27 (plan question Q75, and question Q10 of the review of N16): a short outage of the vRack never opens public SSH. While port 22 of its public interface takes every source, `hopboxd` logs `this server never knew an edge server, so no jump host reaches it, and its public interface keeps port 22 open` at level warn, at each install of the table. An edge server opens port 22 to every source anyway.
- A server that knew an edge server keeps port 22 of its public interface closed to every address outside `jump_hosts`, also while that set is empty. This holds after a restart, through the file `<data_dir>/hostfw/peers`, and after the last edge server left ([The jump hosts](#the-jump-hosts)). The way back after a lockout is the rescue mode or the console of the provider. A computer that the operator owns has no such provider tools, so its way back is its own console.
- While the set `vrack_jump_hosts` is empty, port 22 of the vRack zone takes the whole vRack range ([The jump hosts](#the-jump-hosts)).
- One netlink batch replaces the whole table. If the kernel refuses the batch, the table before it stays as it was.
- A host with two vRack addresses keeps the vRack interface of the last check, so a mistake on the host never closes the vRack zone.
- A server that had a vRack address and lost it keeps the rules of its vRack interface. The Terraform provider sets up no vRack address since N30. So the address comes back only with a restart of the network service of the host, when the network configuration of the host still holds it. The file `<data_dir>/hostfw/vrack-interface` names the interface, so a restart or a reboot without the address keeps it too. `hopboxd` logs `this server lost its vRack address, so the host firewall keeps the rules of its vRack interface` at level error. Its public interface keeps its own rules, so the jump hosts reach it there. To let a server that left the vRack for good drop the rules of the vRack interface, remove the file and restart `hopboxd`.
- The table outlives `hopboxd`. A stop, a crash, or an upgrade of `hopboxd` keeps the table, so SSH through the jump host goes on.
- A destroy of the Terraform provider stops `hopboxd`, and then removes the table. Its public address then takes SSH again ([The Terraform provider](#the-terraform-provider)).

To let a server that left its install for good take SSH from every source again, run these commands on it:

```sh
sudo rm /var/lib/hopbox/hostfw/peers
sudo nft delete table inet hopbox_host
sudo systemctl restart hopboxd
```

While the server is still in an install, the next sets of the control role close port 22 again.

A release from before the host firewall keeps the table of a later release, and never changes its jump hosts. So the Terraform provider removes the table after it moves a server to such a release ([hopbox Terraform provider](/reference/terraform-provider/index.html.md#update)). The owner decided it on 2026-10-27 (plan question Q79). It reads the release from `hopboxd version --json`: from N16 on, the field `features` names `host-firewall` ([the version command](/reference/version/index.html.md#json-output)). After a move by hand, run `sudo nft delete table inet hopbox_host` on the server through the jump host.

If a server stays closed, the rescue mode of its provider boots it into another system, which runs no table.

## The Terraform provider

The provider reaches a server without the role `edge` through the jump host of `ssh_jump_host`, which dials the peer address of the server ([hopbox Terraform provider](/reference/terraform-provider/index.html.md#jump-host)). If the provider block has no `ssh_jump_host`, a create or an update of such a server fails before any step. The owner decided two more rules of the provider on 2026-10-27:

- An import takes the peer address of the server alone, such as `203.0.113.21`. The import of a server without the role `edge` dials that address through the jump host. The install must know a server with that peer address, or the import fails (plan question Q96, [hopbox Terraform provider](/reference/terraform-provider/index.html.md#import)).
- A destroy removes the table once `hopboxd` stopped. It also removes the files `vrack-interface` and `peers` of `<data_dir>/hostfw`. So a later `hopboxd` on the server never starts with the jump hosts of the old install. So its public address takes SSH again (plan question Q77, [hopbox Terraform provider](/reference/terraform-provider/index.html.md#delete)). A server without `nft` gets the package `nftables` first. The login, the stop of `hopboxd`, or the removal of the table can fail. On a server without the role `edge`, the table then keeps port 22 closed to every address but the edge servers. The jump host still reaches the server. So the destroy fails with an error that names the commands to run, and keeps the resource. The next destroy goes on.

## The egress of the machines

An edge server that enrolls takes the egress of the machines of the nodes from its egress tunnel `hbe0` ([the egress of the machines](/reference/egress/index.html.md)). Its table then has six more chains:

| Chain | Hook and priority | What it does |
| --- | --- | --- |
| `egress_mark` | prerouting, mangle | Puts the conntrack mark bit `0x20000000` on each new connection from a tunnel. Puts the packet mark `0x68620100` on each reply of such a connection. The rule of that mark routes the reply back into the tunnel, and not over the mesh ([the egress of the machines](/reference/egress/index.html.md#the-tunnel-of-an-edge-server)). |
| `egress_output` | output, mangle, of the type `route` | Puts the same mark on each reply of the server itself. A machine reaches a public port of its own edge server, and the reply goes back through the tunnel (plan question Q52). |
| `forward` | forward, filter | Sends each packet from a tunnel to the chain `egress`. Lets the replies into a tunnel, and drops each other packet into a tunnel. On a server with the node role, it takes the local exit too. It drops each packet from a tap device that conntrack marks invalid, and each packet from a tap device to an address of the set `known_peers`. It refuses each IPv6 packet from a tap device to `2000::/3` that leaves through another interface than the uplink. Drops the forwarded traffic of the other interfaces, as [The forward guard](#the-forward-guard) says. |
| `egress` | none, the chain `forward` jumps to it | Drops each packet that conntrack marks invalid. Drops each destination in the set `known_peers`. Drops the private ranges of the node table, which hold the mesh range and the vRack range. Drops each IPv6 destination outside `2000::/3`. Accepts IPv4 to the public zone. Accepts IPv6 from `fd65:1c74:e769::/48` only to the uplink, and only while the uplink has a global IPv6 address. Refuses the rest: TCP with a reset, and other packets with ICMP administratively prohibited. |
| `egress_nat` | postrouting, srcnat, of the type `nat` | Translates the source of the egress. IPv4 from a tunnel to the public zone gets `edge.public_ipv4`, or the masquerade without that key. IPv6 of every machine to the uplink gets the global IPv6 address of the uplink (NAT66). This holds for the tunnel and for the local exit. |
| `egress_guard` | postrouting, `srcnat + 10` | Drops each packet with a source in `10.240.0.0/12` or `fd65:1c74:e769::/48` that leaves through an interface other than `lo`, `hbm0`, a tunnel, or a tap device. It comes after the source NAT, so it sees the translated source. |

The table of the node network translates the IPv4 egress of the machines of a node with the edge role. So the rule of IPv4 in `egress_nat` takes the egress of a tunnel alone ([the node network](/reference/node-network/index.html.md#the-table-inet-hopbox_node)). The rule of IPv6 takes the machines of the tunnel and of the local exit alike.

The egress of a machine reaches no peer address of the install, through a tunnel or through the local exit. An edge server without `edge.public_ipv4` translates the IPv4 egress to the address of its uplink, which is its peer address. Without the drop, a machine reaches the ports that the other servers open to the set `peers`, such as the internal listener of a control server. The drop holds also when the edge server is a control server. It also holds for a server with the control role alone that stopped: such a server has no row in `servers`, so it leaves `peers` 20 s after its stop, but its own table keeps its sets and still takes this edge. It stays in `known_peers` until the sweep of its row. This is a known limit: a control server that stays down for more than 24 hours leaves `known_peers`. At its return, its start writes its row again, and the edges drop its address again within seconds.

The drop needs the whole set `known_peers`. The start of `hopboxd` knows the whole set when the file `<data_dir>/hostfw/peers` has the line `known_peers_whole` and no line that the start skips. The firewall writes that line once the sets come from the control role or from a mesh. So an empty set counts as whole too, such as on a server without a peer address. The table before the start never counts, because a start with a closed egress installs a partial set too. Until then, the chain `egress` drops each packet, and the chain `input` drops each packet from a tunnel. On a server with the node role, the chain `forward` also drops each packet from a tap device to the public zone. So no machine reaches a public address before the first sets of the control role or of a mesh, and no machine reaches a peer that the set lacks. This covers these starts:

- A start with a file of a later release that has a line that this `hopboxd` skips.
- A start with a file that it cannot read.
- A start with a file of a release before N28, without the line `known_peers_whole`.
- A start after a reboot, without the file.
- A start without the file that adopts the table before it.
- A second start before the first sets, after a start with a closed egress. The first start writes the file again without `known_peers_whole`.

At such a start, `hopboxd` logs `the host firewall does not know every peer address of the install yet, so the egress of the machines reaches no public address until the first sets of the control plane` at level warn. The first sets open the egress again. This is a build default of N28, not an owner decision.

A connection of a machine to the peer address of its own edge server never passes the forward hook. The input hook takes it, with the address of the machine as its source. The chain `input` drops each packet from a tunnel to an address of the set `known_peers`, before the jump to the zones. So a machine behind the tunnel reaches no port at the peer address of its edge server. It still reaches the public ports at the floating addresses, as the internet does. On the local exit, the node table drops a connection of a machine to its own server ([the node network](/reference/node-network/index.html.md#what-a-machine-can-reach)). Peer addresses are IPv4 only (plan question Q91), so these drops have no rule of IPv6.

NAT never runs for a packet that conntrack cannot place, such as a TCP segment with SYN and FIN, or one without flags. Without a rule, such a packet leaves the uplink with the address of the machine. Two rules stop it. The chains `egress` and `forward` drop each such packet from a tunnel or a tap device, before any accept. The chain `egress_guard` drops each packet that still has the source of a machine on the way out. Each rule holds alone.

### The uplink and the NAT66

The IPv4 egress leaves through the public zone. The IPv6 egress leaves only through the uplink. The uplink is the device of the IPv6 default route of the main table, with the lowest metric. Without `edge.public_ipv4`, `hopboxd` logs `this edge server has no edge.public_ipv4, so the egress of the machines leaves from the address of its output interface` at level warn, at each start.

The NAT66 gives the IPv6 egress one global IPv6 address of the uplink. The firewall picks it in this order:

1. If the uplink holds the preferred source (`src`) of the IPv6 default route, that address.
2. Else the lowest permanent address, such as a static address of the network config of the host.
3. Else the lowest other address, such as an address of a router advertisement.

The pick skips an address in duplicate address detection, an address that failed it, a deprecated address, and a temporary address. It also skips unique local addresses, link-local addresses, and the floating address of `edge.public_ipv6` ([the bootstrap config of hopboxd](/reference/hopboxd-config/index.html.md#keys)). The AAAA records name that floating address, and the egress never leaves from it.

With a multipath IPv6 default route, the uplink is the device of its first next hop. The kernel can send a flow to another next hop, and the table refuses the IPv6 egress there. So such a flow gets a reset or ICMPv6 administratively prohibited. Give an edge server one IPv6 default route.

Each check of the table reads the uplink again, every 2 s ([The table must stay](#the-table-must-stay)). A new uplink or a new address installs the table again. A failed read keeps the uplink of the last check.

Without an IPv6 default route, or without a global IPv6 address on the uplink, the edge server has no NAT66. The chain `egress` then refuses the IPv6 egress of the tunnel. The chain `forward` refuses the IPv6 egress of the local exit to `2000::/3`. TCP gets a reset, and other packets get ICMPv6 administratively prohibited (plan question Q85). `hopboxd` logs a warning at the start and at each change ([Logs](#logs)).

To see the uplink and the address of the NAT66 on an edge server, run these commands:

```sh
ip -6 route show default
sudo nft list chain inet hopbox_host egress_nat
```

### The forward guard

At the start of an edge server that enrolls, the host firewall reads `net.ipv4.ip_forward`. When it is `0`, the firewall turns it on for the egress, after the install of the table. Then the chain `forward` drops each forwarded IPv4 packet that touches neither a tunnel nor a tap device:

```text
		meta nfproto ipv4 iifname != "hbe*" oifname != "hbe*" drop
```

On a server with the node role, the rule also names the tap devices: `iifname != "hbt*" oifname != "hbt*"`. The file `<data_dir>/hostfw/forward-guard` records that `hopboxd` turned forwarding on ([Server layout](/reference/server-layout/index.html.md#paths)). Each later start finds forwarding on, reads the file, and keeps the guard. Something else can turn forwarding on before the first start of `hopboxd`, such as Docker. The table then has no guard for IPv4, and the forwarding of that host stays as it was. To remove the guard, remove the file and restart `hopboxd`:

```sh
sudo rm /var/lib/hopbox/hostfw/forward-guard
sudo systemctl restart hopboxd
```

IPv6 forwarding is one switch for the whole host. The firewall turns it on at each start of an edge server that enrolls, also after the install of the table ([the egress of the machines](/reference/egress/index.html.md#ipv6-forwarding)). So the chain `forward` always drops each forwarded IPv6 packet that touches neither a tunnel nor a tap device.

## ICMP and DHCPv6

Each interface takes these packets, also without a connection, as the owner decided on 2026-10-27 (plan question Q78 rule 2):

| Packets | Why |
| --- | --- |
| ICMP echo request, and ICMPv6 echo request | The monitoring of OVHcloud pings each server. A server that does not answer can get an intervention. |
| ICMP destination unreachable, time exceeded, and parameter problem. ICMPv6 destination unreachable, packet too big, time exceeded, and parameter problem. | Path MTU discovery and the errors of the routes. |
| ICMPv6 router advertisement, neighbor solicitation, neighbor advertisement, and multicast listener query | IPv6 on the link stops without them. |
| UDP to port 546 from a link-local address | The answers of a DHCPv6 server. The request goes to a multicast address, so the answer does not count as a reply. |

## The table

On a server with the control role and the node role, the table looks like this. The set `jump_hosts` holds the edge servers, and so does `vrack_jump_hosts`:

```text
table inet hopbox_host {
	set jump_hosts {
		type ipv4_addr
	}

	set peers {
		type ipv4_addr
	}

	set registered {
		type ipv4_addr
	}

	set vrack_jump_hosts {
		type ipv4_addr
	}

	chain input {
		type filter hook input priority filter; policy accept;
		iifname "lo" accept
		ct state established,related accept
		iifname "hbt*" accept
		meta l4proto icmp icmp type echo-request accept
		meta l4proto icmp icmp type destination-unreachable accept
		meta l4proto icmp icmp type time-exceeded accept
		meta l4proto icmp icmp type parameter-problem accept
		meta l4proto ipv6-icmp icmpv6 type echo-request accept
		meta l4proto ipv6-icmp icmpv6 type destination-unreachable accept
		meta l4proto ipv6-icmp icmpv6 type packet-too-big accept
		meta l4proto ipv6-icmp icmpv6 type time-exceeded accept
		meta l4proto ipv6-icmp icmpv6 type parameter-problem accept
		meta l4proto ipv6-icmp icmpv6 type mld-listener-query accept
		meta l4proto ipv6-icmp icmpv6 type nd-router-advert accept
		meta l4proto ipv6-icmp icmpv6 type nd-neighbor-solicit accept
		meta l4proto ipv6-icmp icmpv6 type nd-neighbor-advert accept
		meta nfproto ipv6 ip6 saddr fe80::/10 meta l4proto udp th dport 546 accept
		iifname "enp3s0f1" jump vrack
		iifname "hbm0" jump mesh
		jump public
	}

	chain vrack {
		meta nfproto ipv4 ip saddr @vrack_jump_hosts meta l4proto tcp th dport 22 accept
		meta nfproto ipv4 ip saddr 172.31.0.0/16 meta l4proto udp th dport 51820 accept
		meta nfproto ipv4 ip saddr 172.31.0.0/16 meta l4proto tcp th dport 7443 accept
		drop
	}

	chain mesh {
		meta nfproto ipv4 ip saddr 10.239.0.0/16 meta l4proto tcp th dport 7444 accept
		drop
	}

	chain public {
		meta nfproto ipv4 ip saddr @jump_hosts meta l4proto tcp th dport 22 accept
		meta nfproto ipv4 ip saddr @peers meta l4proto udp th dport 51820 accept
		meta nfproto ipv4 ip saddr @peers meta l4proto tcp th dport 7443 accept
		meta nfproto ipv4 ip saddr @registered meta l4proto tcp th dport 7443 accept
		meta nfproto ipv4 ip saddr @peers meta l4proto udp th dport 51821-51836 accept
		drop
	}
}
```

While the server never knew an edge server and never had a vRack address, the first rule of the chain `public` is `meta l4proto tcp th dport 22 accept`. While the set `vrack_jump_hosts` is empty, its rule is `meta nfproto ipv4 ip saddr 172.31.0.0/16 meta l4proto tcp th dport 22 accept`. On a server without a vRack address, the table has no chain `vrack`. A server without the control role has no set `registered`.

Take an edge server with the DNS role, `edge.public_ipv6 = "2001:db8:1::99"`, and `edge.public_ipv4 = "203.0.113.101"`. Its uplink is `eno1`, with the global IPv6 address `2001:db8:1::11`. On it, these chains differ, and the table has no set `registered`. The edge server enrolls, and `hopboxd` turned IPv4 forwarding on, so the table has the chains of the egress too:

```text
	chain input {
		type filter hook input priority filter; policy accept;
		iifname "lo" accept
		ct state established,related accept
		meta l4proto icmp icmp type echo-request accept
		meta l4proto icmp icmp type destination-unreachable accept
		meta l4proto icmp icmp type time-exceeded accept
		meta l4proto icmp icmp type parameter-problem accept
		meta l4proto ipv6-icmp icmpv6 type echo-request accept
		meta l4proto ipv6-icmp icmpv6 type destination-unreachable accept
		meta l4proto ipv6-icmp icmpv6 type packet-too-big accept
		meta l4proto ipv6-icmp icmpv6 type time-exceeded accept
		meta l4proto ipv6-icmp icmpv6 type parameter-problem accept
		meta l4proto ipv6-icmp icmpv6 type mld-listener-query accept
		meta l4proto ipv6-icmp icmpv6 type nd-router-advert accept
		meta l4proto ipv6-icmp icmpv6 type nd-neighbor-solicit accept
		meta l4proto ipv6-icmp icmpv6 type nd-neighbor-advert accept
		meta nfproto ipv6 ip6 saddr fe80::/10 meta l4proto udp th dport 546 accept
		iifname "hbe*" meta nfproto ipv4 ip daddr @known_peers drop
		iifname "enp3s0f1" jump vrack
		iifname "hbm0" jump mesh
		jump public
	}

	chain vrack {
		meta nfproto ipv4 ip saddr @vrack_jump_hosts meta l4proto tcp th dport 22 accept
		meta nfproto ipv4 ip saddr 172.31.0.0/16 meta l4proto udp th dport 51820 accept
		drop
	}

	chain mesh {
		drop
	}

	chain public {
		meta nfproto ipv4 ip saddr @peers meta l4proto udp th dport 51820 accept
		meta nfproto ipv4 ip saddr @peers meta l4proto udp th dport 51821 accept
		meta l4proto tcp th dport 22 accept
		meta l4proto tcp th dport 80 accept
		meta l4proto tcp th dport 443 accept
		meta l4proto tcp th dport 2222 ct status dnat accept
		meta l4proto tcp th dport 53 accept
		meta l4proto udp th dport 53 accept
		drop
	}

	chain egress_mark {
		type filter hook prerouting priority mangle; policy accept;
		iifname "hbe*" ct state new ct mark set ct mark | 0x20000000
		ct direction reply ct mark & 0x20000000 == 0x20000000 meta mark set 0x68620100
	}

	chain egress_output {
		type route hook output priority mangle; policy accept;
		ct direction reply ct mark & 0x20000000 == 0x20000000 meta mark set 0x68620100
	}

	chain forward {
		type filter hook forward priority filter; policy accept;
		iifname "hbe*" jump egress
		oifname "hbe*" ct state established,related accept
		oifname "hbe*" drop
		meta nfproto ipv4 iifname != "hbe*" oifname != "hbe*" drop
		meta nfproto ipv6 iifname != "hbe*" oifname != "hbe*" drop
	}

	chain egress {
		ct state invalid drop
		meta nfproto ipv4 ip daddr @known_peers drop
		meta nfproto ipv4 ip daddr 0.0.0.0/8 drop
		meta nfproto ipv4 ip daddr 10.0.0.0/8 drop
		meta nfproto ipv4 ip daddr 100.64.0.0/10 drop
		meta nfproto ipv4 ip daddr 127.0.0.0/8 drop
		meta nfproto ipv4 ip daddr 169.254.0.0/16 drop
		meta nfproto ipv4 ip daddr 172.16.0.0/12 drop
		meta nfproto ipv4 ip daddr 192.0.0.0/24 drop
		meta nfproto ipv4 ip daddr 192.168.0.0/16 drop
		meta nfproto ipv4 ip daddr 198.18.0.0/15 drop
		meta nfproto ipv4 ip daddr 224.0.0.0/3 drop
		meta nfproto ipv6 ip6 daddr != 2000::/3 drop
		meta nfproto ipv4 oifname != "lo" oifname != "hbm0" oifname != "hbe*" oifname != "enp3s0f1" accept
		meta nfproto ipv6 ip6 saddr fd65:1c74:e769::/48 oifname "eno1" accept
		meta l4proto tcp reject with tcp reset
		reject with icmpx admin-prohibited
	}

	chain egress_nat {
		type nat hook postrouting priority srcnat; policy accept;
		iifname "hbe*" meta nfproto ipv4 ip saddr 10.240.0.0/12 oifname != "lo" oifname != "hbm0" oifname != "hbe*" oifname != "enp3s0f1" snat ip to 203.0.113.101
		meta nfproto ipv6 ip6 saddr fd65:1c74:e769::/48 oifname "eno1" snat ip6 to 2001:db8:1::11
	}

	chain egress_guard {
		type filter hook postrouting priority srcnat + 10; policy accept;
		meta nfproto ipv4 ip saddr 10.240.0.0/12 oifname != "lo" oifname != "hbm0" oifname != "hbe*" drop
		meta nfproto ipv6 ip6 saddr fd65:1c74:e769::/48 oifname != "lo" oifname != "hbm0" oifname != "hbe*" drop
	}
```

The floating address `2001:db8:1::99` is in no rule. Without `edge.public_ipv4`, the rule of IPv4 in `egress_nat` ends with `masquerade`. An edge server can have no IPv6 default route, or no global IPv6 address on the uplink. Then the chain `egress` has no rule of IPv6 to the uplink, and `egress_nat` has no rule of IPv6 ([The uplink and the NAT66](#the-uplink-and-the-nat66)).

On a server with the node role, more rules name the tap devices `hbt*`. These are the rules of the guard, of `egress_guard`, and of the public zone in `egress` and `egress_nat`. The chain `forward` also gets these rules of the local exit:

```text
		iifname "hbt*" ct state invalid drop
		iifname "hbt*" meta nfproto ipv4 ip daddr @known_peers drop
		iifname "hbt*" meta nfproto ipv6 ip6 daddr 2000::/3 oifname != "eno1" meta l4proto tcp reject with tcp reset
		iifname "hbt*" meta nfproto ipv6 ip6 daddr 2000::/3 oifname != "eno1" reject with icmpx admin-prohibited
```

Without the NAT66, the two rules of IPv6 have no `oifname`, so they refuse each IPv6 packet of a tap device to `2000::/3`.

Until the start knows the whole set `known_peers`, the chain `egress` holds one `drop`. The rule of the tunnel in the chain `input` is `iifname "hbe*" drop`, without the set. On a local exit, the chain `forward` keeps the drop of the invalid packets, and then drops each packet from a tap device to the public zone, in place of the rules of the peers and of IPv6.

`internal/hostfw/testdata` holds both tables, and a test compares them with the rules of the code. `nft list table inet hopbox_host` prints the same rules in its own words.

### The other tables

A server can have three tables of `hopboxd`. Each table sees each packet of its hooks. An accept of one table does not stop the next table, and a drop of any table drops the packet.

| Table | Server | Its part |
| --- | --- | --- |
| `inet hopbox_host` | Every server | What reaches the server. |
| `inet hopbox_node` | A node | What the machines reach, and what reaches a machine ([the node network](/reference/node-network/index.html.md#the-table-inet-hopbox_node)). |
| `inet hopbox_edge` | An edge server | The redirect of the SSH ports to the SSH front door ([the SSH front door](/reference/ssh-front-door/index.html.md)). |

On a server that is not an edge server, the table `inet hopbox_host` filters only the input hook. So it does not touch the traffic that a node forwards between the mesh, its machines, and its egress tunnels. On an edge server that enrolls, it also has the chains of [The egress of the machines](#the-egress-of-the-machines).

## At the start and at the stop of hopboxd

The unit `host firewall` of `hopboxd run` takes these steps at its start:

1. It reads the sets of the table that an earlier `hopboxd` installed. Without such a table, the sets start empty.
2. It reads `<data_dir>/hostfw/peers`, which gives `peers`, `jump_hosts`, and `known_peers` ([Sets](#sets)). On an edge server, the egress stays closed until the start knows the whole set `known_peers`. Only a file with `known_peers_whole` and no skipped line gives the whole set, never the table of step 1 ([The egress of the machines](#the-egress-of-the-machines)). Without the file, it keeps those of step 1. A file or a line of it that the start cannot read never stops the start, because after a reboot no table is left. The start logs the error, skips each such line, and counts the server as one that knew an edge server, so port 22 of the public interface stays closed. Without a readable file, it keeps the sets of step 1. The next sets write the file again.
3. It finds the vRack interface.
4. On an edge server that enrolls, it reads `net.ipv4.ip_forward` and the file of [the forward guard](#the-forward-guard). It finds the uplink and the address of the NAT66 ([The uplink and the NAT66](#the-uplink-and-the-nat66)).
5. It replaces the table with the rules of the roles and the sets, in one netlink batch. No packet sees a table that is half made. If the batch fails, the start of `hopboxd` fails, and the table before it stays.
6. On an edge server that enrolls, it turns on the IPv4 forwarding that step 4 found off. Then it turns on IPv6 forwarding ([the egress of the machines](/reference/egress/index.html.md#ipv6-forwarding)).
7. It starts to watch the table.

The stop of `hopboxd` ends the watch, and keeps the table.

### The table must stay

Do not flush the nftables ruleset of a server: no `nft flush ruleset`, and no restart of `nftables.service` of Debian, whose start flushes the ruleset.

The unit watches nftables. After each change in nftables, and every 2 s in case the watch fails, it checks the table: its sets, its chains, the expressions of each rule, and the elements of each set. So a flush, a delete of a part, and a replace of a rule each show. When a part differs, it installs the whole table again. It logs this error at most once a minute, with the count of repairs in the field `repairs`:

```text
the nftables table inet hopbox_host lost a part outside hopboxd, so the host firewall installs it again. Do not flush the nftables ruleset of a server
```

The bound of the repair is 2 s, the time between two checks. In the test fleet, the table was back 22 to 48 ms after a flush. A change in nftables within 2 s of a repair waits for the next check. So two writers of the table take turns every 2 s, not at the speed of the events. Two `hopboxd` processes or another tool can be such writers. In the gap, the server has no host firewall. A connection that opened in the gap stays open after the repair, because the table takes the packets of each connection that conntrack knows. While `hopboxd` does not run, nothing repairs the table, and the next start installs it.

Each check also finds the vRack interface again, and on an edge server the uplink of the NAT66. A check installs the table again after a new vRack interface, a new uplink or uplink address, and new sets. A lost vRack address keeps the interface ([Never a lockout](#never-a-lockout)).

## Check the firewall

On a server, list the table, the jump hosts, and the peers:

```sh
sudo nft list table inet hopbox_host
sudo nft list set inet hopbox_host jump_hosts
sudo nft list set inet hopbox_host peers
sudo cat /var/lib/hopbox/hostfw/peers
```

In these commands, the node has the peer address `203.0.113.31` and the vRack address `172.31.0.31`, and the edge server has the floating address `203.0.113.101`. From a host on the internet that is not a server of the install, probe the public address of the node. A connection must time out, also on port 22:

```sh
nc -vz -w 3 203.0.113.31 22
```

Log in to the node through the edge server as the jump host, at the peer address of the node:

```sh
ssh -J debian@203.0.113.101 debian@203.0.113.31
```

## Metrics

| Metric | Type | Labels | Description |
| --- | --- | --- | --- |
| `hopbox_host_firewall_repairs_total` | counter | none | The installs of the table after it lost a part outside `hopboxd`. On a healthy server, it stays flat. It can grow every 2 s while nothing else writes nftables. Then the kernel reads a rule back in another form than the one that `hopboxd` wrote. |
| `hopbox_host_firewall_up` | gauge | none | 1 when the last check found the table whole or installed it. 0 when its install failed. |

A host without nftables, such as a Mac, serves neither metric.

## Logs

The firewall logs with `component=hostfw`. A check logs a state only after a change of the state, so a check every 2 s does not repeat a line.

| Message | Level | Fields | When |
| --- | --- | --- | --- |
| `the host firewall installed its table` | info | `table`, `vrack_interface`, `jump_hosts`, `peers`, `registered`, `vrack_jump_hosts` | An install at the start, or after a change of the vRack interface or of a set. `peers` and `registered` are the counts of the addresses of these sets. A repair logs the next row instead. |
| `the nftables table inet hopbox_host lost a part outside hopboxd, so the host firewall installs it again. Do not flush the nftables ruleset of a server` | error | `err`, `repairs` | A check found the table broken, at most once a minute ([The table must stay](#the-table-must-stay)). |
| `this server lost its vRack address, so the host firewall keeps the rules of its vRack interface` | error | `vrack_interface` | A server that had a vRack address has none now ([Never a lockout](#never-a-lockout)). |
| `cannot keep the vRack interface of this server on its disk, so a restart without its vRack address takes its vRack interface as a public interface` | error | `err`, `dir` | The write of `<data_dir>/hostfw/vrack-interface` failed. |
| `cannot keep the sets of the host firewall on its disk, so a restart takes them from the table, and a restart after a loss of the table opens port 22 of the public interface` | error | `err`, `dir` | The write of `<data_dir>/hostfw/peers` failed. The next set of the control role tries again. It logs once until a write works. |
| `cannot read the last sets of the host firewall, so it takes the sets of the table, and keeps port 22 of the public interface closed` | error | `err`, `file` | The start cannot read `<data_dir>/hostfw/peers`. It goes on with the sets of the table before it ([At the start and at the stop of hopboxd](#at-the-start-and-at-the-stop-of-hopboxd)). |
| `the file of the last sets of the host firewall has a line that this hopboxd cannot read, so it skips the line, and keeps port 22 of the public interface closed` | error | `file`, `line`, `text` | A line of `<data_dir>/hostfw/peers` is not `peer <address>`, `jump_host <address>`, or `knew_edge`, or its address is no peer address, such as a line of a later release after a rollback. The start goes on with the other lines. |
| `the install has no edge server now, so port 22 of the public interface keeps the last jump hosts` | warn | `jump_hosts` | The last edge server left the install ([The jump hosts](#the-jump-hosts)). It logs once until an edge server comes back. |
| `cannot install the host firewall. The next check tries again` | error | `err`, `retry_in` | An install after the start failed. The table before it stays. |
| `this server never knew an edge server, so no jump host reaches it, and its public interface keeps port 22 open` | warn | | An install on a server without the edge role that never knew an edge server and never had a vRack address ([Never a lockout](#never-a-lockout)). |
| `cannot find the vRack interface, so the host firewall keeps the vRack interface of its last check` | error | `err`, `vrack_interface` | The host has two or more vRack addresses. |
| `cannot read the sets of the table before this start, so port 22 of the vRack interface takes the whole vRack range` | warn | `err` | The start did not read the sets of the table before it. `peers` and `jump_hosts` still come from `<data_dir>/hostfw/peers`. |
| `this host has no nftables, so hopboxd runs without the host firewall` | warn | `err` | `hopboxd` runs on a host other than Linux, such as a Mac. |
| `cannot watch nftables, so the host firewall checks its table only every <interval>` | warn | `err` | The watch of nftables did not start. The interval is 2 s. |
| `the watch of nftables ended, so the host firewall checks its table only every <interval>` | warn | | The watch of nftables stopped. The interval is 2 s. |
| `the host firewall opens no public port for this listener, so it takes no connection from outside the server` | warn | `key`, `listen`, `proto` | The start found a public listener of the config on a port that the table does not open ([Ports](#ports)). |
| `turned on IPv4 forwarding for the egress of the machines, and drops the forwarded traffic of other interfaces` | info | `guard_file` | At the start of an edge server, IPv4 forwarding was off. The firewall turned it on, with the forward guard in place ([The forward guard](#the-forward-guard)). |
| `turned on IPv6 forwarding for the egress of the machines, and keeps the router advertisements of the interfaces` | info | `accept_ra_2` | At the start of an edge server, IPv6 forwarding was off, or an interface had `accept_ra` 1 ([the egress of the machines](/reference/egress/index.html.md#ipv6-forwarding)). |
| `this edge server has no edge.public_ipv4, so the egress of the machines leaves from the address of its output interface` | warn | | The start of an edge server without `edge.public_ipv4` ([The uplink and the NAT66](#the-uplink-and-the-nat66)). |
| `the IPv6 egress of the machines leaves from the uplink address of this edge server` | info | `device`, `address` | The start of an edge server, or a new uplink or uplink address. `device` is the uplink, and `address` the source of the NAT66 ([The uplink and the NAT66](#the-uplink-and-the-nat66)). |
| `this edge server has no IPv6 default route, so the IPv6 egress of the machines gets ICMPv6 administratively prohibited, and TCP a reset` | warn | | The start of an edge server without an IPv6 default route, or the loss of that route. |
| `the uplink of this edge server has no global IPv6 address, so the IPv6 egress of the machines gets ICMPv6 administratively prohibited, and TCP a reset` | warn | `device` | The start of an edge server whose uplink has no global IPv6 address that the NAT66 can take, or the loss of that address. |
| `cannot find the IPv6 uplink of this edge server, so the NAT66 of the machines keeps the uplink of the last check` | error | `err`, `device`, `address` | A read of the IPv6 routes or addresses failed. `device` and `address` are those of the last check. It logs once until the error changes. |
| `the host firewall does not know every peer address of the install yet, so the egress of the machines reaches no public address until the first sets of the control plane` | warn | `file` | The start of an edge server that does not know the whole set `known_peers`. The egress stays closed until the first sets ([The egress of the machines](#the-egress-of-the-machines)). |
| `cannot read the sets of the host firewall, so it keeps the last ones. The next read tries again` | warn | `err`, `retry_in` | A read of the sets from the store failed, on a server with the control role. The control role logs it with `component=control`, once until a read works. |

## Errors

| Error | Cause |
| --- | --- |
| `hostfw: install the nftables table inet hopbox_host: ... hopboxd needs CAP_NET_ADMIN and the kernel needs nf_tables` | The start cannot make the table. `hopboxd` stops, and systemd starts it again. |
| `hostfw: read the vRack interface of the server: ...` | The start cannot read `<data_dir>/hostfw/vrack-interface`. `hopboxd` stops. |
| `hostfw: read the IPv4 forwarding of the host: ...`, `hostfw: read the IPv6 forwarding of the host: ...`, `hostfw: record the forward guard: ...`, or `hostfw: record the IPv6 forwarding: ...` | The start of an edge server cannot read `net.ipv4.ip_forward` or `net.ipv6.conf.all.forwarding`, or cannot write `<data_dir>/hostfw/forward-guard` or `<data_dir>/hostfw/forward-guard-ipv6`. `hopboxd` stops. |
| `hostfw: turn on IPv4 forwarding: ...` or `hostfw: turn on IPv6 forwarding: ...` | The start of an edge server cannot write the sysctl. The table is in place. `hopboxd` stops. |

## Tests

| Test | Where it runs | What it checks |
| --- | --- | --- |
| The tests of `internal/hostfw` without root | `make test`, also on a Mac | The ports of each role, the sets of each role, the tables of `testdata`, the drop at the end of each zone, the mesh range, and the tap prefix. The egress of a tunnel and of a local exit drops the set `known_peers`, which keeps a control server that left `peers`, also through a restart from the file `peers`. Until the start knows the whole set, the egress of a tunnel and of a local exit reaches no public address: after a file with a skipped line, a file that the start cannot read, a file without the line `known_peers_whole`, a start without a file, also with a table that holds known peers, and a file with `known_peers_whole` and a skipped line. Two more starts without sets after a skipped line keep the egress closed, and write no `known_peers_whole`. A start from a whole file, also one without a known peer, keeps the egress open. So does a restart after empty sets of a server without a peer address. The first sets open the egress in each case. The ways in of [Never a lockout](#never-a-lockout): a server that never knew an edge, a server with a vRack address before an edge and after its loss, an edge known on the vRack alone, the last jump hosts after the last edge left, and the file `peers` through a restart. A lost vRack address. The NAT66 to the uplink, with and without its address, and the drop of `known_peers` in the chain `input` for a tunnel, or of the whole tunnel until the start knows the whole set. A check that follows a new uplink, keeps the last one after a failed read, and warns without an address. An install after each change, a repair, the wait after a repair, a failed install, and the sets of a restart, from the file or from the table, also a table of a release before N27. A bad file `peers` and a file that the start cannot read, each of which still starts with port 22 closed, sets before the start that never replace the file, a failed write of the file, and the sets of a mesh that the firewall refuses. The warning about a closed listener, and the stop of the unit. The chains of the egress with and without the keys, on a server with the node role, and the family of each match. The order of the install and of the switches of forwarding, and the guard file after a restart. |
| `TestHostFirewall` of `internal/hostfw` | `make test-linux`, as root in a privileged container | A control server with the node role in network namespaces, with the real kernel. The public interface of the server with a vRack address takes port 22 from the jump hosts alone, also before the server knew an edge. It takes the mesh port and the egress ports from the peers alone, and the internal listener from the peers and the registered addresses. The vRack interface takes the ports of the roles from the vRack range, and port 22 from the vRack jump hosts alone. The mesh device takes the disk listener from the mesh range alone. A flush of the ruleset, a flush of each set, and a replace of a rule each come back within 2 s. A batch that the kernel refuses keeps the table. The last edge leaves, and port 22 keeps it. A restart without the table keeps the sets of the file `peers`. A lost vRack address keeps the rules of the vRack interface, also after a restart. A server that never knew an edge and has no vRack address keeps port 22 on its public interface. Once it knew an edge on the vRack alone, port 22 closes, also after a restart without the table. |
| `TestEdgeHostFirewall` of `internal/hostfw` | `make test-linux` | An edge server with the DNS role. The table reads back whole. The public ports and the SSH ports through the redirect are open at three addresses. These are the IPv4 and the IPv6 address of the uplink, and the floating IPv6 address. Port 2222 and a port of no role are closed. The vRack address takes no public port. The egress port takes a peer, and not the test server, nor a packet to the floating IPv6 address. |
| `TestEdgeEgress` of `internal/hostfw` | `make test-linux` | An edge server in network namespaces, with the internet on its uplink. A machine is behind a veth pair named `hbe0`. The machine reaches the internet with the floating address over IPv4. Over IPv6, it reaches the internet with the address of the uplink, never with the floating IPv6 address. It reaches the public ports of its edge server at the floating addresses, and the reply comes back through the tunnel, not over the mesh. It reaches no peer address of the install, also not that of its own edge server. It reaches no private range and no vRack address. Without a global IPv6 address on the uplink, a connect over IPv6 gets a refusal at once, and the IPv4 egress goes on. The forward guard drops the traffic from the internet to the vRack. |
| `TestNoMachineSourceOnTheUplink` of `internal/hostfw` | `make test-linux` | A machine behind a veth pair named `hbe0`, and a machine on a tap device of a local exit. Each sends TCP segments with SYN and FIN, and without flags, over IPv4 and IPv6. The internet sees none of them with the address of the machine, and sees a valid SYN with the address of the edge. |
| `TestLocalExitReachesNoPeer` of `internal/hostfw` | `make test-linux` | An edge server with the node role, a machine on a tap device, and another server on the internet. The machine reaches that server with the address of the edge until the edge knows it as a peer, and not after. |
| `TestLocalExitIPv6` of `internal/hostfw` | `make test-linux` | An edge server with the node role, and a machine on a tap device. The machine leaves through the uplink with the address of the uplink over IPv6, as a machine behind the tunnel does. Without that address, or toward another device, a connect over IPv6 gets a refusal at once. |
| `TestBastionFirewall` of `test/fleet` | `make test-fleet` | A control server with its internal listener on its peer address on `inet0`, two nodes, and two edge servers, each enrolled from its peer address. A scan from `inet0` finds only the public ports of the edges. It probes each public address and each floating address, IPv4 and IPv6. The set `peers` of each server holds the control server too. The counters show that the mesh port takes a peer and drops the test server. A peer reaches the internal listener, and a registered address reaches it within 2 s, until its token expires. Port 22 on `inet0` takes the edges alone. SSH through each edge reaches each other server after each change of the table, also after a restart without a control server and without the table. A decommissioned edge leaves the jump hosts, and the last edge stays one. |
| `TestImportByPeerAddress`, `TestProviderFleet`, `TestNoJumpHostRefusesAServerBehindTheFirewall`, and `TestDestroyKeepsAFirewalledServer` of `internal/tfprovider` | `make test` | The import of a server by its peer address, and of a node through the jump host. A server behind the firewall needs `ssh_jump_host`. A destroy removes the table. While the table stays, the destroy fails and keeps the resource. |
| `TestMoveToAReleaseWithoutTheFirewallDropsIt` of `internal/tfprovider` | `make test` | After a move to a release whose version names no host firewall, the provider removes the table. After a move to a release with it, the table stays. |
| `TestDropHostFirewallRunsNft` of `internal/tfprovider/hostsetup` | `make test` | The script of the destroy deletes the table with `nft`, and removes the files `vrack-interface` and `peers`. If `nft` is missing and `apt-get` cannot install it, the script fails. |
