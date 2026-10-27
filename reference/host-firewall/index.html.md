# The host firewall

> The host firewall of every server, the nftables table inet hopbox_host: its zones and ports, the jump hosts, the ways in that it keeps, the address of the /56, its repair, metrics, logs, and tests.

The host firewall is the nftables table `inet hopbox_host` of every server of an install. It makes the edge servers the bastion of the install (rules 2 and 5). Only an edge server takes traffic from the internet. A control server or a node takes no new connection on its public address. On the vRack and on the mesh, each server takes only the ports of its roles. Operators reach a server that is not an edge server by SSH only through an edge server as a jump host.

The firewall is in `internal/hostfw`. `hopboxd run` starts it on every server, right after the metrics listener and before the units of the roles ([hopboxd](/reference/hopboxd/index.html.md#run)). The provider block of the Terraform provider names the jump host in `ssh_jump_host` ([hopbox Terraform provider](/reference/terraform-provider/index.html.md#jump-host)).

In the examples, the vRack interface is `enp3s0f1`, and the edge servers have the vRack addresses `172.31.0.21` and `172.31.0.22`.

## Zones

The table sorts each new connection by the interface that takes it. A zone is the set of interfaces that share one list of open ports.

| Zone | Interfaces | Sources that a port takes |
| --- | --- | --- |
| Loopback | `lo` | Every source. The table takes every packet of loopback. |
| vRack | The interface that holds the one vRack address of the server, as the mesh finds it ([the mesh](/reference/mesh/index.html.md#which-servers-join)) | The vRack range `172.31.0.0/16`, or the jump hosts for port 22. The `/56` of the install is not part of the vRack range. |
| Mesh | `hbm0` | The mesh range `10.239.0.0/16`. |
| Tap devices | The tap devices of the machines on a node, `hbt*` | The table takes them, and leaves them to the table `inet hopbox_node` ([the node network](/reference/node-network/index.html.md#the-table-inet-hopbox_node)), which owns all traffic of the machines. |
| Public | Each other interface, such as the uplink | Every source. |

On an edge server, the address of the `/56` on the vRack interface is in the public zone too ([The address of the /56](#the-address-of-the-56)). Each zone ends with a drop, so a port that no role opens takes nothing. The replies of the connections of the server pass in every zone, and so do the ICMP messages of [ICMP and DHCPv6](#icmp-and-dhcpv6).

## Ports

Each role opens its ports. A server with more than one role opens the ports of each role.

| Role | Zone | Port | Source | Use |
| --- | --- | --- | --- | --- |
| Every server | vRack | TCP 22 | The jump hosts | The SSH of the operators and of the provider through an edge server ([The jump hosts](#the-jump-hosts)). |
| A server that enrolls: `node`, `dns`, or `edge` | vRack | UDP 51820 | The vRack range | The WireGuard packets of the mesh ([the mesh](/reference/mesh/index.html.md#values)). |
| `control` | vRack | TCP 7443, the port of `control.internal_listen` | The vRack range | The internal listener: the enrollment, the control channels, and the API and the console that the edge servers forward ([the internal listener](/reference/internal-listener/index.html.md)). |
| `node` | Mesh | TCP 7444 | The mesh range | The disk listener of the cold moves ([Cold move](/reference/cold-move/index.html.md#the-disk-listener)). |
| `edge` | Public | TCP 22 | Every source | The `sshd` of the operators. The edge server is the jump host. |
| `edge` | Public | TCP 80 and 443 | Every source | The HTTP and HTTPS listeners ([the edge role](/reference/edge/index.html.md#listeners)). |
| `edge` | Public | TCP 2222, the port of `edge.ssh_listen`, only for connections that a redirect sent there | Every source | The SSH ports of the machines, TCP 10000 to 30000. The table `inet hopbox_edge` sends them to `edge.ssh_listen` ([the SSH front door](/reference/ssh-front-door/index.html.md)), so the input hook sees port 2222. When conntrack records the redirect (`ct status dnat`), the rule takes the connection there. Any redirect or destination NAT to that port passes, also one that an operator with `edge.ssh_redirect = false` writes. A connection straight to port 2222 takes nothing. |
| `dns` | Public | TCP and UDP 53 | Every source | The DNS role ([the DNS role](/reference/dns/index.html.md)). |

An edge server takes no new connection on the vRack other than its ports of the vRack zone. The edge servers of an install run the DNS role. Port 53 follows the role `dns`, not the role `edge`, as the owner decided on 2026-10-27 (plan question Q74). So a server with the DNS role alone, as in [Serve the domain of an install with the DNS role](/guide/dns-role/index.html.md), opens port 53 and no edge port. An edge server without the DNS role keeps port 53 closed. The table opens the same ports whatever the listen addresses of the config say, as the owner decided on 2026-10-27 (plan question Q80). A listener on another port, such as `edge.https_listen = ":8443"`, takes no connection from outside the server. So at its start, `hopboxd` logs `the host firewall opens no public port for this listener, so it takes no connection from outside the server` at level warn, for each such listener of `edge.https_listen`, `edge.http_listen`, and `dns.listen`. A listener on loopback gets no warning.

The other listeners of `hopboxd` take nothing from outside the server: the API listener on `127.0.0.1:7080`, the metrics listener on `127.0.0.1:9464`, and the listener of the SSH front door on port 2222.

## The jump hosts

Port 22 of the vRack zone takes the vRack addresses of the edge servers alone. An edge server reaches a vRack address straight over its vRack interface. So the SSH of the provider comes from the vRack address of the edge server, not from its mesh address. The set `jump_hosts` of the table holds these addresses.

| Server | Where the jump hosts come from |
| --- | --- |
| A server that enrolls | Each mesh that the control role sends. The jump hosts are the vRack addresses of the peers with the role `edge` ([the mesh](/reference/mesh/index.html.md#the-peers)). A new edge server reaches the firewall of each server within about two heartbeats. |
| A server with the control role alone | The control role reads the vRack addresses of the edge servers from the table `mesh_members` every 5 s ([Control store tables](/reference/control-store/index.html.md#mesh_members)). |

While the server knows no edge server, port 22 takes the whole vRack range. This holds before the first mesh after a reboot, and in an install without an edge server on the vRack. A restart of `hopboxd` keeps the set of the table before it, so a restart while no control server answers keeps the jump hosts.

Each check compares the elements of the set with the jump hosts. So a flush of the set alone comes back within 2 s, and so does a delete of an element. Each new mesh checks the set again, also a mesh with the same edge servers.

## Never a lockout

A change of the table never closes the way of an operator into a server. The owner decided the first two ways in on 2026-10-27 (plan question Q75):

- A server that never had a vRack address keeps port 22 open on its public interface, because no jump host reaches it. ks is such a server. `hopboxd` logs `this server has no address on the vRack, so no jump host reaches it, and its public interface keeps port 22 open` at level warn.
- While the server knows no edge server, port 22 of the vRack zone takes the whole vRack range ([The jump hosts](#the-jump-hosts)).
- One netlink batch replaces the whole table. If the kernel refuses the batch, the table before it stays as it was.
- A host with two vRack addresses keeps the vRack interface of the last check, so a mistake on the host never closes the vRack zone.
- A server that had a vRack address and lost it keeps its vRack interface, and its public interface stays closed (design rule 5). The owner decided this on 2026-10-27 (plan question Q75): a short outage of the vRack never opens public SSH. The cost: if the address is gone for good, only the rescue mode of OVHcloud reaches the server. The address comes back with the network setup of the provider, or a restart of the network service. The file `<data_dir>/hostfw/vrack-interface` names the interface, so a restart or a reboot without the address keeps it too. `hopboxd` logs `this server lost its vRack address, so the host firewall keeps its vRack interface, and its public interface stays closed` at level error. To let a server that left the vRack for good open port 22 again, remove the file and restart `hopboxd`.
- The table outlives `hopboxd`. A stop, a crash, or an upgrade of `hopboxd` keeps the table, so SSH through the jump host goes on.
- A destroy of the Terraform provider stops `hopboxd`, and then removes the table, before the server leaves the vRack. Its public address then takes SSH again ([The Terraform provider](#the-terraform-provider)).

A release from before the host firewall keeps the table of a later release, and never changes its jump hosts. So the Terraform provider removes the table after it moves a server to such a release ([hopbox Terraform provider](/reference/terraform-provider/index.html.md#update)). The owner decided it on 2026-10-27 (plan question Q79). It reads the release from `hopboxd version --json`: from N16 on, the field `features` names `host-firewall` ([the version command](/reference/version/index.html.md#json-output)). After a move by hand, run `sudo nft delete table inet hopbox_host` on the server through the jump host.

If a server stays closed, the rescue mode of OVHcloud boots it into another system, which runs no table.

## The Terraform provider

The provider reaches a server without the role `edge` through the jump host of `ssh_jump_host`, at its vRack address ([hopbox Terraform provider](/reference/terraform-provider/index.html.md#jump-host)). If the provider block has no `ssh_jump_host`, a create or an update of such a server with `vrack_address` fails before any step. The owner decided two more rules of the provider on 2026-10-27:

- An import of such a server names its vRack address after a comma in the import ID: `<service name>,<vRack address>`, such as `ns1234568.ip-198-51-100.eu,172.31.0.21`. The import dials that address through the jump host. The server there must hold the public address of the service name, or the import fails. An edge server imports with the service name alone (plan question Q76, [hopbox Terraform provider](/reference/terraform-provider/index.html.md#import)).
- A destroy removes the table once `hopboxd` stopped, before the server leaves the vRack. So its public address takes SSH again (plan question Q77, [hopbox Terraform provider](/reference/terraform-provider/index.html.md#delete)). A server without `nft` gets the package `nftables` first. The login, the stop of `hopboxd`, or the removal of the table can fail. Then the destroy keeps the server in the vRack, where the jump host reaches it. It fails with an error that names the commands to run, and the next destroy goes on.

## The address of the /56

An edge server holds one address of the `/56` of the install on its vRack interface ([the edge role](/reference/edge/index.html.md#the-public-ipv6-address)). The vRack carries the public IPv6 traffic of that address. So the address takes the public ports of the server from every source, as the uplink does. The other addresses of the vRack interface keep the rules of the vRack zone.

The provider of the uplink does not route the `/56`. So on an edge server with `edge.public_ipv6`, the chain `postrouting` drops some packets. It drops each packet with a source in the `/56` that leaves through an interface other than the vRack interface and loopback. If the rule or the route of the table `hopbox_egress6` is gone, a reply of the `/56` still never leaves through the uplink.

The chain runs at the priority `srcnat + 10`, after the source NAT at `srcnat`. So it also drops a packet that a source NAT gives an address of the `/56`. The NAT66 of the egress of N17 is such a source NAT. A rule of N17 that changes the source of a packet must run at `srcnat` or before, never after the guard.

## ICMP and DHCPv6

Each interface takes these packets, also without a connection, as the owner decided on 2026-10-27 (plan question Q78 rule 2):

| Packets | Why |
| --- | --- |
| ICMP echo request, and ICMPv6 echo request | The monitoring of OVHcloud pings each server. A server that does not answer can get an intervention. |
| ICMP destination unreachable, time exceeded, and parameter problem. ICMPv6 destination unreachable, packet too big, time exceeded, and parameter problem. | Path MTU discovery and the errors of the routes. |
| ICMPv6 router advertisement, neighbor solicitation, neighbor advertisement, and multicast listener query | IPv6 on the link stops without them. |
| UDP to port 546 from a link-local address | The answers of a DHCPv6 server. The request goes to a multicast address, so the answer does not count as a reply. |

## The table

On a server with the control role and the node role, the table looks like this. The set `jump_hosts` holds the edge servers:

```text
table inet hopbox_host {
	set jump_hosts {
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
		meta nfproto ipv4 ip saddr @jump_hosts meta l4proto tcp th dport 22 accept
		meta nfproto ipv4 ip saddr 172.31.0.0/16 meta l4proto udp th dport 51820 accept
		meta nfproto ipv4 ip saddr 172.31.0.0/16 meta l4proto tcp th dport 7443 accept
		drop
	}

	chain mesh {
		meta nfproto ipv4 ip saddr 10.239.0.0/16 meta l4proto tcp th dport 7444 accept
		drop
	}

	chain public {
		drop
	}
}
```

While the set `jump_hosts` is empty, its rule is `meta nfproto ipv4 ip saddr 172.31.0.0/16 meta l4proto tcp th dport 22 accept`. On a server without a vRack address, the table has no chain `vrack`, and the chain `public` opens port 22 first.

On an edge server with the DNS role and `edge.public_ipv6 = "2001:db8:0:100::11/56"`, these chains differ:

```text
	chain vrack {
		meta nfproto ipv6 ip6 daddr 2001:db8:0:100::11 jump public
		meta nfproto ipv4 ip saddr @jump_hosts meta l4proto tcp th dport 22 accept
		meta nfproto ipv4 ip saddr 172.31.0.0/16 meta l4proto udp th dport 51820 accept
		drop
	}

	chain mesh {
		drop
	}

	chain public {
		meta l4proto tcp th dport 22 accept
		meta l4proto tcp th dport 80 accept
		meta l4proto tcp th dport 443 accept
		meta l4proto tcp th dport 2222 ct status dnat accept
		meta l4proto tcp th dport 53 accept
		meta l4proto udp th dport 53 accept
		drop
	}

	chain postrouting {
		type filter hook postrouting priority srcnat + 10; policy accept;
		meta nfproto ipv6 ip6 saddr 2001:db8:0:100::/56 oifname != "lo" oifname != "enp3s0f1" drop
	}
```

`internal/hostfw/testdata` holds both tables, and a test compares them with the rules of the code. `nft list table inet hopbox_host` prints the same rules in its own words.

### The other tables

A server can have three tables of `hopboxd`. Each table sees each packet of its hooks. An accept of one table does not stop the next table, and a drop of any table drops the packet.

| Table | Server | Its part |
| --- | --- | --- |
| `inet hopbox_host` | Every server | What reaches the server. |
| `inet hopbox_node` | A node | What the machines reach, and what reaches a machine ([the node network](/reference/node-network/index.html.md#the-table-inet-hopbox_node)). |
| `inet hopbox_edge` | An edge server | The redirect of the SSH ports to the SSH front door ([the SSH front door](/reference/ssh-front-door/index.html.md)). |

The table `inet hopbox_host` filters only the input hook, and the postrouting hook on an edge server with a `/56` address. So it does not touch the traffic that a node forwards between the mesh and its machines.

## At the start and at the stop of hopboxd

The unit `host firewall` of `hopboxd run` takes these steps at its start:

1. It reads the set `jump_hosts` of the table that an earlier `hopboxd` installed. Without such a table, the set starts empty.
2. It finds the vRack interface.
3. It replaces the table with the rules of the roles, in one netlink batch. No packet sees a table that is half made. If the batch fails, the start of `hopboxd` fails, and the table before it stays.
4. It starts to watch the table.

The stop of `hopboxd` ends the watch, and keeps the table.

### The table must stay

Do not flush the nftables ruleset of a server: no `nft flush ruleset`, and no restart of `nftables.service` of Debian, whose start flushes the ruleset.

The unit watches nftables. After each change in nftables, and every 2 s in case the watch fails, it checks the table: its chains, the expressions of each rule, and the elements of its set. So a flush, a delete of a part, and a replace of a rule each show. When a part differs, it installs the whole table again. It logs this error at most once a minute, with the count of repairs in the field `repairs`:

```text
the nftables table inet hopbox_host lost a part outside hopboxd, so the host firewall installs it again. Do not flush the nftables ruleset of a server
```

The bound of the repair is 2 s, the time between two checks. In the test fleet, the table was back 22 to 48 ms after a flush. A change in nftables within 2 s of a repair waits for the next check. So two writers of the table take turns every 2 s, not at the speed of the events. Two `hopboxd` processes or another tool can be such writers. In the gap, the server has no host firewall. A connection that opened in the gap stays open after the repair, because the table takes the packets of each connection that conntrack knows. While `hopboxd` does not run, nothing repairs the table, and the next start installs it.

Each check also finds the vRack interface again. A check installs the table again after a new vRack interface and after new jump hosts. A lost vRack address keeps the interface ([Never a lockout](#never-a-lockout)).

## Check the firewall

On a server, list the table and the jump hosts:

```sh
sudo nft list table inet hopbox_host
sudo nft list set inet hopbox_host jump_hosts
```

In these commands, the node has the public address `203.0.113.31` and the vRack address `172.31.0.31`. From a host on the internet, probe the public address of the node. A connection must time out, also on port 22:

```sh
nc -vz -w 3 203.0.113.31 22
```

Log in to the node through an edge server as the jump host, at the vRack address of the node:

```sh
ssh -J debian@203.0.113.5 debian@172.31.0.31
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
| `the host firewall installed its table` | info | `table`, `vrack_interface`, `jump_hosts` | An install at the start, or after a change of the vRack interface or of the jump hosts. A repair logs the next row instead. |
| `the nftables table inet hopbox_host lost a part outside hopboxd, so the host firewall installs it again. Do not flush the nftables ruleset of a server` | error | `err`, `repairs` | A check found the table broken, at most once a minute ([The table must stay](#the-table-must-stay)). |
| `this server lost its vRack address, so the host firewall keeps its vRack interface, and its public interface stays closed` | error | `vrack_interface` | A server that had a vRack address has none now ([Never a lockout](#never-a-lockout)). |
| `cannot keep the vRack interface of this server on its disk, so a restart without its vRack address opens port 22 of its public interface` | error | `err`, `dir` | The write of `<data_dir>/hostfw/vrack-interface` failed. |
| `cannot install the host firewall. The next check tries again` | error | `err`, `retry_in` | An install after the start failed. The table before it stays. |
| `this server has no address on the vRack, so no jump host reaches it, and its public interface keeps port 22 open` | warn | `vrack_range` | An install on a server that never had a vRack address, and has no edge role ([Never a lockout](#never-a-lockout)). |
| `cannot find the vRack interface, so the host firewall keeps the vRack interface of its last check` | error | `err`, `vrack_interface` | The host has two or more vRack addresses. |
| `cannot read the jump hosts of the table before this start, so port 22 of the vRack interface takes the whole vRack range until the first mesh` | warn | `err` | The start did not read the set of the table before it. |
| `this host has no nftables, so hopboxd runs without the host firewall` | warn | `err` | `hopboxd` runs on a host other than Linux, such as a Mac. |
| `cannot watch nftables, so the host firewall checks its table only every <interval>` | warn | `err` | The watch of nftables did not start. The interval is 2 s. |
| `the watch of nftables ended, so the host firewall checks its table only every <interval>` | warn | | The watch of nftables stopped. The interval is 2 s. |
| `the host firewall opens no public port for this listener, so it takes no connection from outside the server` | warn | `key`, `listen`, `proto` | The start found a public listener of the config on a port that the table does not open ([Ports](#ports)). |
| `cannot read the edge servers for the host firewall, so it keeps the last ones. The next read tries again` | warn | `err`, `retry_in` | A read of the jump hosts from the store failed, on a server without the mesh. The control role logs it with `component=control`. |

## Errors

| Error | Cause |
| --- | --- |
| `hostfw: install the nftables table inet hopbox_host: ... hopboxd needs CAP_NET_ADMIN and the kernel needs nf_tables` | The start cannot make the table. `hopboxd` stops, and systemd starts it again. |
| `hostfw: read the vRack interface of the server: ...` | The start cannot read `<data_dir>/hostfw/vrack-interface`. `hopboxd` stops. |

## Tests

| Test | Where it runs | What it checks |
| --- | --- | --- |
| The tests of `internal/hostfw` without root | `make test`, also on a Mac | The ports of each role, the tables of `testdata`, the drop at the end of each zone, the mesh range, and the tap prefix. The ways in of [Never a lockout](#never-a-lockout), a lost vRack address, and the rules of the `/56`. An install after each change, a repair, the wait after a repair, a failed install, and the jump hosts of a restart. The warning about a closed listener, and the stop of the unit. |
| `TestHostFirewall` of `internal/hostfw` | `make test-linux`, as root in a privileged container | A control server with the node role in network namespaces, with the real kernel. The public interface takes nothing. The vRack interface takes the ports of the roles from the vRack range, and port 22 from the jump hosts alone. The mesh device takes the disk listener from the mesh range alone. A flush of the ruleset, a flush of the set, and a replace of a rule each come back within 2 s. A batch that the kernel refuses keeps the table. A restart keeps the jump hosts. A lost vRack address keeps the public interface closed, also after a restart. A server that never had a vRack address keeps port 22 on its public interface. |
| `TestEdgeHostFirewall` of `internal/hostfw` | `make test-linux` | An edge server with the DNS role. The table reads back whole. The public ports and the SSH ports through the redirect are open, and port 2222 is closed. The address of the `/56` on the vRack interface takes the public ports. No packet of the `/56` leaves through the uplink, also one that a source NAT gives an address of the `/56`. The same packets arrive before the setup. |
| `TestBastionFirewall` of `test/fleet` | `make test-fleet` | A control server, two nodes, and two edge servers. A scan from `inet0` finds only the public ports of the edges. It probes each public address, each floating address, and each address of the `/56` through `gw6`. A scan from `gw6` of the vRack addresses finds no port. Port 22 of the vRack takes the edges alone. SSH through each edge reaches each other server after each change of the table. A node that loses its vRack address keeps its public address closed. A decommissioned edge leaves the jump hosts. |
| `TestImportBehindTheJumpHost`, `TestFleetOnTheVRack`, `TestNoJumpHostRefusesAServerBehindTheFirewall`, and `TestDestroyKeepsTheFirewalledServerInTheVRack` of `internal/tfprovider` | `make test` | The import of a node at its vRack address through the jump host. A server behind the firewall needs `ssh_jump_host`. A destroy removes the table, and keeps the server in the vRack while the table stays. |
| `TestMoveToAReleaseWithoutTheFirewallDropsIt` of `internal/tfprovider` | `make test` | After a move to a release whose version names no host firewall, the provider removes the table. After a move to a release with it, the table stays. |
| `TestDropHostFirewallRunsNft` of `internal/tfprovider/hostsetup` | `make test` | The script of the destroy deletes the table with `nft`. If `nft` is missing and `apt-get` cannot install it, the script fails. |

## Later milestones

Later milestones of phase 2 add to the table (the build plan of phase 2):

- N17 adds the UDP ports of the egress tunnels to the vRack zone, and the forward and NAT rules of the egress of the machines.
- N21 adds the ports of the one-time move of ks, whose control channel and disk copy cross the public internet.
