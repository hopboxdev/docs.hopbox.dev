# The node network

> The network of the machines of a node: the addresses and their hold, the tap devices, the nftables table inet hopbox_node, the resolvers, and the traffic counters.

The node network connects each machine to its node, and through its node to the internet. It gives each machine an address, and routes the address to the tap device of the machine. It keeps the machines apart from each other and from the node, and it counts their traffic. A tap device is a virtual network device: Firecracker writes the packets of the guest to it, and reads the packets for the guest from it.

The control role gives out the addresses (`internal/service/network.go`). The node role sets up the network (`internal/node/netcfg`). A node reaches the other servers of the install over the mesh, a WireGuard device on the vRack ([the mesh](/reference/mesh/index.html.md)). The edge servers reach the machines of the node over the mesh.

## Addresses

| Item | Value |
| --- | --- |
| Install range | `10.240.0.0/12`. Every machine address of the install is in it. |
| Node subnet | A `/20` of the install range for each node, such as `10.240.0.0/20`. When the control role places the first machine of a node, the node gets the first free subnet. The install range holds 256 node subnets. The subnet of a decommissioned node goes free again ([Subnet of a decommissioned node](#subnet-of-a-decommissioned-node)). |
| Machine address | One IPv4 address of the subnet of the first node of the machine. A create takes the first free address, from the second address of the subnet to the one before the last. A subnet holds 4094 machines. A machine that moves to another node keeps its address ([Moved machines](#moved-machines)). |
| Prefix length in the guest | `/32`. The guest has no neighbor on its link apart from the gateway. |
| Gateway | `169.254.0.1`, on-link. The node has this address on each tap device. |
| MTU | `1420`, on the tap device and in the guest. A packet of this size fits the mesh device `hbm0`, whose MTU is `1420` too ([the mesh](/reference/mesh/index.html.md#values)). |
| Hold | 5 minutes after the delete of a machine (`domain.AddressHold`). |
| IPv6 | None in phase 1. IPv6 egress comes in phase 2, with the egress through the edge servers. |

A machine keeps its address for its whole life. A stop, a start, a reboot, a restart of `hopboxd`, and a cold move keep it.

A create claims the address in its own transaction, under the row lock of the node. So two creates never take the same address. When the subnet of the node has no free address, the create fails with `no capacity` and does nothing ([the machine API](/reference/machine-api/index.html.md)).

### The hold of a released address

The executor reports `deleted` once it removed the machine from its node. The report starts the hold of the address of the machine. During the hold, no new machine gets the address.

An edge server gets a machine lease with each route. It sends traffic to the address of the route for at most the rest of that lease, 30 seconds ([the edge role](/reference/edge/index.html.md)). The hold of 5 minutes is much longer. So no new machine at an old address gets the requests or the SSH sessions of the machine that had it before ([the SSH front door](/reference/ssh-front-door/index.html.md)). After the hold, the next create on the node can take the address again.

### Moved machines

A cold move takes a machine to another node ([Cold move](/reference/cold-move/index.html.md)). The machine keeps its address, which stays in the subnet of its first node. The row of the address in `machine_addresses` names the new node from the hand-off of the move. At the boot of the machine, the new node routes the `/32` of the address to its tap device ([The tap device of a machine](#the-tap-device-of-a-machine)). The table `inet hopbox_node` takes the device and the address in the set `machines`. So the rules work for an address outside the subnet of the node too.

The mesh peer of the new node allows the `/32` of the address ([the mesh](/reference/mesh/index.html.md#allowed-ips)). WireGuard sends a packet to the peer with the longest prefix that holds its destination. So the edge servers and the other nodes reach the machine at its new node, and the rest of the subnet at the first node. The first node no longer has a tap device for the address, so its mesh routes the `/32` over `hbm0` like the other servers do.

The subnet of the first node keeps the address while the machine lives. So a create on the first node never takes it.

### Subnet of a decommissioned node

A drain moves every machine off a node, and then decommissions the node ([Cold move](/reference/cold-move/index.html.md#drain-of-a-node)). The leader sweep, which runs every 10 min with the other sweeps, frees the subnet of a decommissioned node once no machine address of that subnet is held. An address is held when `held_until` is null or in the future. A moved machine keeps the address of its first node, so the subnet stays taken while such a machine lives. After a delete, the hold of 5 min ends, and the next sweep frees the subnet. The next node that gets its first machine takes the first free subnet, which can be the freed one.

### Where the address shows

| Place | What it holds |
| --- | --- |
| `machine_addresses` | The address of each machine, with its node and its hold ([Control store tables](/reference/control-store/index.html.md#machine_addresses)). |
| `node_subnets` | The subnet of each node ([Control store tables](/reference/control-store/index.html.md#node_subnets)). |
| The spec of the machine | `MachineNetwork`: the address as a `/32`, the gateway, and the MTU ([the executor protocol](/reference/executor-protocol/index.html.md#machinespec)). |
| The route record | `route_records.address`, from each report of running. The edge role reaches the machine at this address ([Control store tables](/reference/control-store/index.html.md#route_records)). |
| The guest | The address of `eth0`, and an on-link default route through the gateway ([Guest link](/reference/guest-link/index.html.md)). |

Migration 0039 gives addresses to the machines that exist before it. Each node gets a subnet, in the order of the IDs of the nodes. The machines of a node get the addresses of its subnet, in the order of the IDs of the machines.

## The resolvers of the machines

The guest agent writes the resolvers of the machine to `/etc/resolv.conf`. The node adds them to the network of the spec: `network.nameservers` of the bootstrap config ([the bootstrap config of hopboxd](/reference/hopboxd-config/index.html.md#keys)). When that key is empty, the node takes the resolvers of its host at the start. It reads the first file of this list that names a usable resolver:

1. `/run/systemd/resolve/resolv.conf`. With systemd-resolved, `/etc/resolv.conf` names only the stub resolver at `127.0.0.53`, which no machine can reach.
2. `/etc/resolv.conf`.

A usable resolver is an IPv4 address outside the loopback range, the link-local range, multicast, and the install range. The bootstrap config refuses another address. The node takes no other address from the resolv.conf of the host either. The node takes at most 3 resolvers, because glibc reads at most 3. A machine reaches its resolvers on UDP and TCP port 53. This also holds for a resolver in a private range. When the host names no usable resolver, the machine unit of `hopboxd` does not start, and `hopboxd` stops with this error:

```text
machines: netcfg: the host names no resolver that a machine can reach in /run/systemd/resolve/resolv.conf or /etc/resolv.conf. Set network.nameservers in the bootstrap config
```

A resolver on the node itself does not work, at any address of the node. The chain `input` drops each new connection from a tap device. So a machine never reaches a resolver that listens on the node, such as dnsmasq or systemd-resolved. Give the machines a resolver on another host.

Internal names such as `box1.internal` come in phase 4, with the resolver of the node.

## The tap device of a machine

The Firecracker driver makes the tap device of each machine at each boot ([the Firecracker driver](/reference/firecracker-driver/index.html.md#the-tap-device)). Its name is `hbt` and 12 hex digits of a hash of the machine ID. When the machine has an address, the driver then connects the device to the node network, before it starts the unit of the machine. The connect makes these changes:

| Change | Value |
| --- | --- |
| Alias | `hopbox machine <id>`. A new `hopboxd` finds the machine of each device from it. |
| MTU | `1420` |
| `net.ipv6.conf.<tap>.disable_ipv6` | `1`. The device has no IPv6 address, so no IPv6 packet of the guest reaches the node. |
| `net.ipv4.conf.<tap>.arp_ignore` | `1`. The node answers ARP on the device only for the gateway address. |
| Address | `169.254.0.1/32`, scope link. The node never uses it as the source address toward another device. |
| State | up |
| Route | `<machine address>/32 dev <tap> scope link proto static` |
| Sets of the table | The device and the address in the set `machines`, and the device in the set `inbound`. See [The table](#the-table-inet-hopbox_node). |

A second connect of one device replaces the first one. Its elements in the sets start again with counters at zero. So a boot that reuses a tap device that a crash left behind reports no traffic of the boot before.

The driver takes the device out of the sets and then removes the device, wherever it removes the device: after the end of the microVM, after a failed boot, and at the delete of the machine. The removal of the device also removes its address and its route.

A machine without an address, or a node without the node network, boots with its tap device down. The driver then logs `the machine boots without network` as a warning. The metadata service of Firecracker still answers the guest, because Firecracker answers it on the device itself.

Firecracker runs in the network namespace of `hopboxd`, because the jailer gets `--netns /proc/<pid of hopboxd>/ns/net`. So Firecracker opens the tap device in the namespace where the driver made it. This also holds for a test that runs `hopboxd` in a namespace of its own.

## The table `inet hopbox_node`

The node network owns the nftables table `inet hopbox_node`. The family `inet` holds IPv4 and IPv6. The set `edges` holds the mesh address of each edge server of the mesh, from the last mesh of the control role ([the mesh](/reference/mesh/index.html.md#traffic-of-the-edge-servers-to-machines)). Each new install of the table keeps the set, also the install of a new `hopboxd`. This is the table on a node whose tap devices start with `hbt`, with the resolver `213.186.33.99`. The golden file `internal/node/netcfg/testdata/ruleset.nft` holds the same text. When the rules change and the file does not, a test fails.

```text
table inet hopbox_node {
	set machines {
		type ifname . ipv4_addr
		counter
	}

	set inbound {
		type ifname
		counter
	}

	set edges {
		type ipv4_addr
	}

	chain forward {
		type filter hook forward priority filter; policy accept;
		iifname "hbt*" tcp flags syn / syn,rst tcp option maxseg size set rt mtu
		oifname "hbt*" tcp flags syn / syn,rst tcp option maxseg size set rt mtu
		iifname "hbm0" meta nfproto ipv4 ip saddr @edges oifname "hbt*" ct state new tcp flags syn / syn,ack ct mark set ct mark | 0x40000000
		iifname "hbt*" jump from_machine
		oifname "hbt*" jump to_machine
	}

	chain from_machine {
		meta nfproto ipv6 drop
		meta nfproto ipv4 iifname . ip saddr != @machines drop
		meta nfproto ipv4 ip daddr 213.186.33.99 meta l4proto udp th dport 53 iifname . ip saddr @machines accept
		meta nfproto ipv4 ip daddr 213.186.33.99 meta l4proto tcp th dport 53 iifname . ip saddr @machines accept
		oifname "hbm0" ct mark & 0x40000000 == 0x40000000 ct state established,related meta nfproto ipv4 iifname . ip saddr @machines accept
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
		meta nfproto ipv4 iifname . ip saddr @machines accept
		drop
	}

	chain to_machine {
		meta nfproto ipv6 drop
		ct state established,related accept
		iifname "hbm0" ct mark & 0x40000000 == 0x40000000 accept
		drop
	}

	chain output {
		type filter hook output priority filter; policy accept;
		oifname "hbt*" ct state new tcp flags syn / syn,ack ct mark set ct mark | 0x40000000
	}

	chain input {
		type filter hook input priority filter; policy accept;
		iifname "hbt*" ct mark & 0x40000000 == 0x40000000 ct state established,related meta nfproto ipv4 iifname . ip saddr @machines accept
		iifname "hbt*" drop
	}

	chain postrouting {
		type filter hook postrouting priority filter; policy accept;
		oifname "hbt*" ct mark & 0x40000000 == 0x40000000 oifname @inbound
	}

	chain nat {
		type nat hook postrouting priority srcnat; policy accept;
		meta nfproto ipv4 ip saddr 10.240.0.0/12 oifname != "hbt*" masquerade
	}
}
```

Each base chain has the policy `accept`. Each rule of the chains above matches only the traffic of tap devices, or of the mesh device `hbm0` to a tap device. So these rules change no other traffic of the host, such as the traffic of containers. The one exception is the chain `forward_guard` below.

### The forward guard

When `net.ipv4.ip_forward` is `0` at the start of `hopboxd`, the node turns it on for the machines alone. The table then gets one more chain:

```text
	chain forward_guard {
		type filter hook forward priority filter; policy accept;
		meta nfproto ipv4 iifname != "hbt*" oifname != "hbt*" drop
	}
```

It drops each forwarded IPv4 packet that neither comes from nor goes to a tap device. Without it, IPv4 forwarding makes the host a router between all its interfaces. The node installs the guarded table first, and only then turns forwarding on.

The dir `<data_dir>/network` has the mode 0700, which each start sets again. The file `<data_dir>/network/forward-guard` records that `hopboxd` turned forwarding on ([Server layout](/reference/server-layout/index.html.md#paths)). Each later start finds forwarding on, reads the file, and keeps the guard. So a flush of the ruleset followed by a restart of `hopboxd` keeps it too. Something else can turn forwarding on before the first start of `hopboxd`, such as Docker. The table then has no guard, and the forwarding of that host stays as it was.

To remove the guard, for example once Docker runs on the host, remove the file and restart `hopboxd`. Forwarding is on already, so the new table has no guard:

```sh
sudo rm /var/lib/hopbox/network/forward-guard
sudo systemctl restart hopboxd
```

### What a machine can reach

| Destination | Result |
| --- | --- |
| A public IPv4 address | Passes. The node gives the packets the address of its output interface (masquerade). On one server, this is the egress through the edge. |
| A resolver of the machines, on UDP or TCP port 53 | Passes, also in a private range. |
| A private, shared, loopback, link-local, or reserved range, multicast, or broadcast | Dropped. The list is `BlockedRanges` of `internal/node/netcfg`, as in the table above. |
| Another machine | Dropped, because every machine address is in `10.0.0.0/8`. Org networks and tag networks come in phase 4. |
| A server of the mesh, at its mesh address | Dropped, because the mesh range `10.239.0.0/16` is in `10.0.0.0/8`. Only the replies to the connections of an edge server pass. |
| The node, at any of its addresses | Dropped by the chain `input`. Only the replies to the connections that the node opened pass. So a machine reaches no service of the node, such as the API, the metrics, or the control role. |
| Any IPv6 address | Dropped. |
| Any destination, from a source address that is not the address of the machine | Dropped. |

No egress allowlist exists in phase 1, so a machine reaches every public host. Allowlists and the egress proxy come in phase 4.

### What can reach a machine

The processes of the node can open TCP connections to a machine: the edge role for the machine URLs, and the SSH front door ([the edge role](/reference/edge/index.html.md), [the SSH front door](/reference/ssh-front-door/index.html.md)). They do it from the output hook, so the chain `to_machine` does not see their packets.

The chain `output` puts the conntrack mark `0x40000000` (`NodeMark`) on each TCP connection that the node opens to a tap device. It knows such a connection by its first packet: a SYN without ACK, in a new conntrack entry. The chain `input` lets a machine answer only connections with this mark. Conntrack can take a TCP connection up again in the middle. The side that sends next is then its origin. Such a take-up has no SYN, so it never gets the mark. So no connection of a machine to the node passes as a connection of the node. The node opens no UDP or ICMP flow to a machine with an answer: a ping from the node to a machine gets no reply. The chain `to_machine` lets the replies to the egress of the machine pass. It drops each new connection from another host or from a container, apart from the connections of the edge servers over the mesh.

An edge server on another server opens its connections to a machine over the mesh, from its mesh address ([the mesh](/reference/mesh/index.html.md#traffic-of-the-edge-servers-to-machines)). Such a connection reaches the node on `hbm0`, through the forward hook. The chain `forward` puts the mark of the node on its first packet, a SYN without ACK in a new conntrack entry. It does this only for a source in the set `edges`. The chain `to_machine` then lets the packets of `hbm0` with the mark pass. The chain `from_machine` lets the replies of the machine back to `hbm0`, before its drop of the private ranges. WireGuard takes a packet from a peer only with a source in the allowed IPs of that peer. So a server that is not an edge cannot send from the mesh address of an edge. Another server of the mesh, or a machine of another node, opens no connection to a machine.

### The mark of the node is reserved

The bit `0x40000000` of the conntrack mark is for hopbox alone. Other rules on a node must not clear it or set it. This holds for the connmark rules of an operator too.

- If a rule clears the bit, the chain `input` drops the replies of the machines. The machine URLs of the edge and the shells of the SSH front door then break. The inbound counter also stops, so the machines look idle.
- If a rule sets the bit on an egress connection of a machine, the chain `postrouting` counts the replies as inbound traffic. A machine that only polls the internet then looks used, and its auto-delete timeout never ends. The chain `input` still drops each new connection of a machine to the node, also with the bit.

A rule that writes the whole mark also clears the bit. These rules write the whole mark:

| Rule | Form that leaves the bit alone |
| --- | --- |
| `ct mark set 0x1` (nftables) | `ct mark set ct mark \| 0x1` |
| `ct mark set meta mark` (nftables) | `ct mark set ct mark & 0x40000000 \| meta mark & 0xbfffffff`. This form needs a bitwise OR of two registers. A test proved it on the hopbox kernel 6.18.57-hopbox1 with nft 1.1.3. An older kernel or nft can refuse the rule. |
| `-j CONNMARK --set-mark 0x1` (iptables) | `-j CONNMARK --set-mark 0x1/0x1` |
| `-j CONNMARK --save-mark` (iptables) | `-j CONNMARK --save-mark --nfmask 0xbfffffff --ctmask 0xbfffffff` |

The mask `0xbfffffff` holds every bit apart from `0x40000000`. A rule that copies the conntrack mark to the packet mark, such as `-j CONNMARK --restore-mark`, does not change the conntrack mark.

### MSS clamping

The chain `forward` clamps the TCP MSS of each SYN to or from a tap device. It lowers the MSS to fit the MTU of the route of the packet. The MSS is the largest TCP payload in one packet. The kernel only lowers the MSS here, and never raises it.

## At the start and at the stop of hopboxd

The machine unit of `hopboxd run` sets up the network before it starts the executors ([hopboxd](/reference/hopboxd/index.html.md#run)):

1. It finds the resolvers of the machines. See [The resolvers of the machines](#the-resolvers-of-the-machines).
2. It reads `net.ipv4.ip_forward`. When forwarding is off, or the file of the [forward guard](#the-forward-guard) exists, the table gets the guard.
3. It finds the tap devices that an earlier `hopboxd` connected: up, with the alias `hopbox machine <id>`, and with a `/32` route to an address of the install range.
4. It reads the set `edges` of the table that an earlier `hopboxd` installed. Without such a table, the set starts empty. A failed read starts the set empty too, and logs `cannot read the edge servers of the table before this start, so the edge servers reach no machine until the first mesh` at level warn.
5. It replaces the table `inet hopbox_node` with the rules above, with the elements of these devices, and with these edge servers, in one netlink batch. No packet sees a table that is half made.
6. It ends the connections that machines opened to the node while the table was gone. See [The table must stay](#the-table-must-stay).
7. When forwarding was off, it sets `net.ipv4.ip_forward` to `1`.
8. It starts to watch the table, apart from the control channel.

So a restart or an upgrade of `hopboxd` keeps the network of the machines that run. The edge servers reach the machines also before the first mesh of the control role, which can come late while no control server runs. The first mesh of the new process then sets the set `edges`. The counters start again from zero. The stop of `hopboxd` keeps the table, the devices, and the routes.

### The table must stay

The isolation of the machines lives in the table `inet hopbox_node` alone. Do not flush the nftables ruleset of a node: no `nft flush ruleset`, and no restart of `nftables.service` of Debian, whose start flushes the ruleset.

The machine unit of `hopboxd` watches nftables, apart from the control channel. After each change in nftables, and every 2 s in case the watch fails, it checks the table: its chains, the count of rules of each chain, and its sets. Each heartbeat checks it too. When a part is gone, the node installs the whole table again, with the tap devices of the moment, and logs this error:

```text
the nftables table inet hopbox_node lost a part outside hopboxd, so the node network installs it again. Do not flush the nftables ruleset of a node
```

Run also checks the table once at its start, so a flush between the setup and the start of the watch waits for no event. The bound of the repair is 2 s, the time between two checks. The watch is usually much faster: on the [tested server](/reference/preflight/index.html.md#the-tested-server), the table was back 8 to 48 ms after its delete. One repair of 20 took 1.73 s, so a change in nftables can also miss the watch. While `hopboxd` does not run, nothing repairs the table, and the next start installs it.

In the gap, a machine can open a connection to a service of the node. Such a connection has no mark of the node, so the chain `input` drops its packets once the table is back. This holds also after conntrack takes the connection up again, for example after a keepalive of the node. To end such connections at once, the node also does this after each repair, and at each start:

- It deletes the conntrack flows from the install range to the addresses of the node.
- It destroys each TCP socket that a service of the node accepted from an address of the install range (`SOCK_DESTROY` of `inet_diag`). The kernel sends a reset to the machine. This needs `CONFIG_INET_DIAG_DESTROY`, which the tested host kernel has. Without it, the node logs `cannot end every connection of a machine to the node`. The socket then stays open until its own timeout, but no data of the machine reaches it, because of the mark.

The node logs `ended the connections that machines opened to the node without the table`, at level warn, with the counts of `sockets` and `flows`. A connect and a disconnect also repair the table before they try again, so a boot and a delete go on. The metric `hopbox_node_network_repairs_total` counts the repairs. The counters of the machines start again from zero. The count of their inbound traffic starts from zero too, so a request right after a repair still counts as a use.

The SSH front door keeps its own table `inet hopbox_edge` in the same way. Every 2 s, it checks the table. If the table lost a part, the front door installs it again ([the SSH front door](/reference/ssh-front-door/index.html.md)).

### A host with Docker

A host with Docker has the chain `FORWARD` of iptables with the policy `drop`. Each forwarded packet must pass every base chain of its hook, so this policy also drops the egress of the machines. On such a host, accept the traffic of the tap devices in the chain `DOCKER-USER`, or run the node without Docker:

```sh
sudo iptables -I DOCKER-USER -i hbt+ -j ACCEPT
sudo iptables -I DOCKER-USER -o hbt+ -m conntrack --ctstate RELATED,ESTABLISHED -j ACCEPT
```

The first rule lets out the packets of the machines. The second one lets in the replies to them. The table `inet hopbox_node` still drops what a machine must not reach. Docker makes the chain `DOCKER-USER` again at each start, without these rules, so add them to the boot of the host.

## Counters

Each element of the sets `machines` and `inbound` has a counter. A rule that finds a packet in the set adds the packet to the counter of its element.

| Counter | What it counts |
| --- | --- |
| Inbound | The packets to the machine of connections that the node opened, such as a request through the machine URL or an SSH session: the connections with the mark of the node. The replies to the egress of the machine do not count. The connections of the edge servers over the mesh count too, because they carry the mark. This is the inbound traffic that counts as a use of the machine ([Inbound traffic as a use](#inbound-traffic-as-a-use)). |
| Egress | The packets of the machine that the node lets out: its egress to the internet and to its resolvers, and its replies to the connections that the node opened. The replies to the connections of the edge servers over the mesh count too. These replies stay in the install, so the counter is more than the meter "egress out of the install" (plan question Q53). Traffic between machines is dropped in phase 1, so its meter is zero. |

### Metrics

The metrics listener serves the counters of each connected machine ([Metrics](/reference/metrics/index.html.md)). Each metric has the label `machine`, the machine ID. The node reads the counters from the table at each scrape. They start from zero at each start of `hopboxd`, and at each new connect of the tap device.

| Metric | Type | Meaning |
| --- | --- | --- |
| `hopbox_machine_inbound_bytes_total` | counter | Bytes to the machine of connections that the node opened. |
| `hopbox_machine_inbound_packets_total` | counter | Packets to the machine of connections that the node opened. |
| `hopbox_machine_egress_bytes_total` | counter | Bytes from the machine that the node let out. |
| `hopbox_machine_egress_packets_total` | counter | Packets from the machine that the node let out. |
| `hopbox_node_network_repairs_total` | counter | The installs of the table after it lost a part outside `hopboxd`. It has no label. See [The table must stay](#the-table-must-stay). |

### Inbound traffic as a use

Each heartbeat of the node carries the machines whose inbound packet count grew since the last heartbeat, in `inbound_machines` ([the control channel](/reference/control-channel/index.html.md#heartbeat)). The control role then sets `last_activity_at` of each of them. It writes at most once a minute for each machine, and only for the machines of that node. So inbound traffic keeps a machine from its auto-delete timeout ([the machine API](/reference/machine-api/index.html.md#auto-delete)).

A start of `hopboxd`, a connect, and a repair begin the count of a tap device at zero, as its new counter does. So the first heartbeat after them reports each packet since then. A heartbeat that the control role never gets loses its machines, and the next inbound packet counts again.

## Logs

The node network logs with `component=netcfg`.

| Message | Level | Fields | When |
| --- | --- | --- | --- |
| `turned on IPv4 forwarding for the machines, and drops the forwarded traffic of other interfaces` | info | `guard_file` | At the start, IPv4 forwarding was off. The node turned it on, with the forward guard in place ([The forward guard](#the-forward-guard)). |
| `the node network is ready` | info | `table`, `machines`, `nameservers`, `edges` | The start installed the table. `machines` is the count of the tap devices that the node took over. `edges` is the count of the edge servers of the table before it. |
| `connected a machine` | info | `machine`, `tap`, `address` | A boot connected the tap device of a machine. |
| `disconnected a machine` | info | `machine`, `tap` | The node removed the tap device of a machine. |
| `the nftables table inet hopbox_node lost a part outside hopboxd, so the node network installs it again. Do not flush the nftables ruleset of a node` | error | `err`, `machines` | A check found the table broken ([The table must stay](#the-table-must-stay)). |
| `cannot repair the node network` | error | `err` | A check found the table broken, and the install of the table failed. The next check tries again. |
| `ended the connections that machines opened to the node without the table` | warn | `sockets`, `flows` | See [The table must stay](#the-table-must-stay). |
| `cannot end every connection of a machine to the node` | error | `err`, `sockets`, `flows` | See [The table must stay](#the-table-must-stay). |
| `cannot watch nftables, so the node network checks its table only every <interval>` | warn | `err` | The watch of nftables did not start. The interval is 2 s. |
| `the watch of nftables ended, so the node network checks its table only every <interval>` | warn | | The watch of nftables stopped. The interval is 2 s. |
| `cannot read the inbound traffic of the machines` | warn | `err` | A heartbeat did not read the counters. It reports no inbound traffic ([Inbound traffic as a use](#inbound-traffic-as-a-use)). |
| `cannot read the edge servers of the table before this start, so the edge servers reach no machine until the first mesh` | warn | `err` | The read of the set `edges` of the old table failed at the start. The set starts empty. |
| `the machines take the connections of the edge servers over the mesh` | info | `edges` | The set `edges` changed. `edges` is the count of edge servers. |
| `the node network cannot take the edge servers of the mesh. The next change of the mesh tries again` | error | `err` | The change of the set `edges` failed. |

## Errors

| Error or log | Cause |
| --- | --- |
| `netcfg: the host names no resolver ...` | See [The resolvers of the machines](#the-resolvers-of-the-machines). |
| `netcfg: install the nftables table inet hopbox_node: ... hopboxd needs CAP_NET_ADMIN and the kernel needs nf_tables` | The start cannot make the table. The machine unit does not start. |
| `firecracker: netcfg: connect the tap device <tap> of machine <id>: ...` | The boot fails, and the driver removes the device. The executor reports the machine `failed`, and the restart policy applies. |
| `the nftables table inet hopbox_node lost a part outside hopboxd, ...` (error) | A flush of the ruleset or a change of the table. The node installs the table again. See [The table must stay](#the-table-must-stay). |
| `cannot watch nftables, so the node network checks its table only every 2s` (warning) | The watch of nftables did not start. The check every 2 s still repairs the table. |
| `the machine boots without network` (warning) | The spec of the machine has no address, or the node has no node network. |
| `no capacity: the subnet <subnet> of node <id> has no free machine address ...` | The create fails. A delete frees an address after its hold. |
| `no capacity: every /20 of the machine range 10.240.0.0/12 belongs to another node` | The install has 256 nodes with machines. |

## Tests

| Test | Where it runs | What it checks |
| --- | --- | --- |
| `TestRulesetGolden` of `internal/node/netcfg` | `make test`, also on a Mac | The rules, against `testdata/ruleset.nft`. Run `go test ./internal/node/netcfg -run TestRulesetGolden -update` to write the file after a change. |
| `TestNodeNetwork` of `internal/node/netcfg` | `make test-linux`, as root in a privileged container | A node, two machines, another interface, and the internet in network namespaces, with veth pairs in place of tap devices. It checks the masquerade, each blocked destination, the resolver, a spoofed source, the MSS, and the inbound counter. It checks that a second connect of a device starts its counters again from zero. It checks the forward guard, the repair after a flush of the ruleset, and a new `hopboxd` that takes over the devices. |
| `TestEdgeOverTheMesh` of `internal/node/netcfg` | `make test-linux`, as root in a privileged container | A node, a machine, and an edge server in network namespaces, with a veth pair named `hbm0` in place of the mesh device. An edge in the set `edges` reaches the machine, and its connection counts as inbound. Another mesh address and the machine itself open nothing. A repair and a new `hopboxd` keep the set. |
| `TestKVMEndToEnd` of `internal/control` | `make test-kvm`, in the network namespace `KVM_NETNS` | Two Firecracker machines on one install. It checks their addresses, the egress with masquerade, and the blocked destinations. It opens a machine URL through the edge role, and a shell through the SSH front door. A probe in a guest sends UDP from forged sources, ARP requests and claims, and IPv6. Only the real packet must leave the node. The test deletes the table, and the next heartbeat must bring the isolation back. It checks inbound traffic as a use, the metrics, the network after a restart of `hopboxd`, and the release at a delete. |

`TestKVMEndToEnd` refuses to run in the network namespace of the host, because it changes nftables tables, routes, and sysctls. `make test-kvm` adds the namespace `KVM_NETNS`, `hopbox-kvm` by default, runs the KVM tests in it, and deletes it.

## Not in phase 1

- IPv6 egress, and egress through separate edge servers: phase 2.
- Internal names such as `box1.internal`, org networks, tag networks, egress allowlists, and the egress proxy: phase 4.
- The store of the meters and their export to Stripe: phase 5.
