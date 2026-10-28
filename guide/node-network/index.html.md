# Check the network of a machine

> How to check the network of a machine: its address, its egress, what it cannot reach, and its traffic counters.

The network redesign of 2026-10-27 changes this page. hopbox leaves the vRack and the OVHcloud API, and the servers talk over their public addresses (Q90). This page still describes the current code. Milestones N26 to N32 change the code and this page. Before you set up a fleet on the vRack or with an IPv6 /56, read Q90.

This guide follows the network of one machine on a server with the node role. It finds the address of the machine, checks its egress and the destinations that it cannot reach, and reads its traffic counters. On an install with several servers on the vRack, it also checks the mesh between the servers. [The node network](/reference/node-network/index.html.md) and [the mesh](/reference/mesh/index.html.md) describe each part.

## What you need

- A Linux server with the control role, the node role, and the edge role, installed as in [Install hopbox on a Debian 13 server by hand](/guide/install-debian/index.html.md). `sudo hopboxd admin server list` must show the server `ready`.
- A running machine `box1`, made as in [Run machines with Firecracker](/guide/firecracker-machines/index.html.md).
- The hopbox CLI, signed in to the install, as in [Use the hopbox CLI](/guide/cli/index.html.md).
- For the last steps, the `nft` tool on the server: `sudo apt install nftables`. `hopboxd` does not need it.

## Set the resolvers

The machines use the resolvers of the server by default. To give them other resolvers, set `network.nameservers` in `/etc/hopbox/hopboxd.toml`, and restart `hopboxd`. A resolver on the server itself does not work, because a machine reaches no service of its node:

```toml
[network]
nameservers = ["213.186.33.99", "9.9.9.9"]
```

```sh
sudo systemctl restart hopboxd
```

The running machines keep their network during the restart. A machine gets the new resolvers at its next boot.

## Find the address of the machine

Each machine has an IPv4 address of the subnet of its node, as a `/32`:

```sh
hopbox exec box1 ip -4 addr show eth0
hopbox exec box1 ip route
```

```text
2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1420 qdisc pfifo_fast state UP group default qlen 1000
    inet 10.240.0.1/32 scope global eth0
default via 169.254.0.1 dev eth0 onlink
169.254.169.254 dev eth0 scope link
```

The machine also has one IPv6 address, which follows from its IPv4 address, and an IPv6 default route through `fe80::1` ([the egress of the machines](/reference/egress/index.html.md#the-ipv6-addresses-of-the-machines)):

```sh
hopbox exec box1 ip -6 addr show eth0 scope global
hopbox exec box1 ip -6 route show default
```

```text
2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1420 state UP qlen 1000
    inet6 fd65:1c74:e769::af0:1/128 scope global nodad
default via fe80::1 dev eth0 metric 1024 pref medium
```

The guest agent wrote the resolvers of the node to `/etc/resolv.conf` of the machine:

```sh
hopbox exec box1 cat /etc/resolv.conf
```

The node routes the address to the tap device of the machine. The tap device has the alias `hopbox machine <id>`:

```sh
ip -d link show | grep -B1 'alias hopbox machine'
ip route show 10.240.0.1
ip -6 route show fd65:1c74:e769::af0:1
```

```text
10.240.0.1 dev hbt3f2c9a0b1d4e proto static scope link
fd65:1c74:e769::af0:1 dev hbt3f2c9a0b1d4e proto static metric 1024 pref medium
```

## Check the egress

The server of this guide has the node role and the edge role, so it is the local exit of its machines ([the egress of the machines](/reference/egress/index.html.md#the-local-exit)). A machine reaches the internet. The node gives its packets the address of the server:

```sh
hopbox exec box1 curl -sS https://ifconfig.me
```

The command prints the public address of the server, not the address of the machine. With `edge.public_ipv4` in the bootstrap config, it prints that address.

Over IPv6, the egress needs `edge.public_ipv6`, an address of the IPv6 block of the install on the vRack ([the edge role](/reference/edge/index.html.md#the-public-ipv6-address)). Without it, the connect fails at once:

```sh
hopbox exec box1 curl -6 -m 3 -sS https://ifconfig.me
```

With `edge.public_ipv6`, the command prints that address.

### With edge servers apart

Edge servers can run apart from the nodes. A node then sends the egress of its machines through an egress tunnel to one edge server ([the egress of the machines](/reference/egress/index.html.md)). The hash of the machine address picks the edge server. So `curl https://ifconfig.me` in a machine prints the floating address of that edge server, and `curl -6` its address of the IPv6 block.

On the node, list the tunnels, one for each edge server, and their rules:

```sh
sudo wg show all endpoints
ip rule show | grep 'proto 107'
```

```text
hbe0	<public key of edge1>	172.31.0.21:51821
hbe1	<public key of edge2>	172.31.0.22:51821
26820:	from all fwmark 0x68620000 lookup 26820 proto 107
26821:	from all fwmark 0x68620001 lookup 26821 proto 107
```

When an edge server stops, the next mesh drops it, and its machines move to the other tunnel. The move waits for the unresponsive time of 20 s, a sweep of the leader, and a heartbeat of the node. The test fleet checks that it takes at most 40 s.

## Check what the machine cannot reach

A machine reaches no service of its node, no private range, and no other machine. Each of these commands fails after the timeout of 3 seconds:

```sh
hopbox exec box1 curl -m 3 http://169.254.0.1:9464/metrics
hopbox exec box1 curl -m 3 http://192.168.1.1/
hopbox exec box1 curl -m 3 http://10.240.0.2:8000/
hopbox exec box1 curl -m 3 'http://[fd65:1c74:e769::af0:2]:8000/'
```

```text
curl: (28) Connection timed out after 3002 milliseconds
```

The edge role on the node still reaches the machine. A machine URL and an SSH session work as in [Run the edge role and open machine URLs](/guide/machine-urls/index.html.md) and [Reach a machine over SSH](/guide/ssh/index.html.md).

## Read the table and the counters

The node keeps its rules in the nftables table `inet hopbox_node`:

```sh
sudo nft list table inet hopbox_node
```

The sets `machines`, `machines6`, and `inbound` hold the tap device of each machine, each with its counter:

```text
	set machines {
		type ifname . ipv4_addr
		counter
		elements = { "hbt3f2c9a0b1d4e" . 10.240.0.1 counter packets 1520 bytes 194311 }
	}

	set machines6 {
		type ifname . ipv6_addr
		counter
		elements = { "hbt3f2c9a0b1d4e" . fd65:1c74:e769::af0:1 counter packets 0 bytes 0 }
	}

	set inbound {
		type ifname
		counter
		elements = { "hbt3f2c9a0b1d4e" counter packets 61 bytes 8840 }
	}
```

Do not flush the nftables ruleset of the server, and do not restart `nftables.service`: the machines lose their isolation for a moment. Within milliseconds, `hopboxd` installs the table again, ends each connection that a machine opened to the server in the gap, and logs `the nftables table inet hopbox_node lost a part outside hopboxd`. `hopbox_node_network_repairs_total` counts these repairs.

The metrics listener serves the same counters for each machine:

```sh
curl -s http://127.0.0.1:9464/metrics | grep '^hopbox_machine_'
```

```text
hopbox_machine_egress_bytes_total{machine="01a1610f-26b5-71c8-abbd-93be11b94c53"} 194311
hopbox_machine_egress_packets_total{machine="01a1610f-26b5-71c8-abbd-93be11b94c53"} 1520
hopbox_machine_inbound_bytes_total{machine="01a1610f-26b5-71c8-abbd-93be11b94c53"} 8840
hopbox_machine_inbound_packets_total{machine="01a1610f-26b5-71c8-abbd-93be11b94c53"} 61
```

The inbound counter grows only with the connections that the node opened, such as a request through the machine URL. The answers to the egress of the machine do not count. Each heartbeat tells the control role which machines got inbound traffic, so these machines count as used for their auto-delete timeout.

## Check the mesh

This part needs an install with several servers on the vRack: each server has one address in `172.31.0.0/16`. A server without such an address, such as a server alone on ks, stays out of the mesh. Its log then has `this server has no address on the vRack, so it stays out of the mesh`.

Each enrolled server on the vRack has the WireGuard device `hbm0`, with a mesh address in `10.239.0.0/16`. The admin API shows the mesh part of each server. Run this command on a control server, with the ID of a server from `sudo hopboxd admin server list`:

```sh
sudo hopboxd admin server get <id> | grep '^mesh_'
```

```text
mesh_address	10.239.0.2
mesh_vrack_address	172.31.0.12
mesh_public_key	gN65BkIKy1eCE9pP1wdc8ROUtkHLF2PfAqYdO0JbmjI=
```

On a server of the mesh, `wg show` lists one peer for each other server, at its vRack address. Install the tool first with `sudo apt install wireguard-tools`. `hopboxd` does not need it.

```sh
sudo wg show hbm0
ip route show proto 104
```

A node has the subnets of the other nodes as routes over `hbm0`. An edge server has the subnet of each node. From an edge server, a TCP connection to a machine on another node goes over the mesh. Run this command on an edge server, with the address of a machine that listens on port 8000:

```sh
curl -m 3 http://10.240.0.1:8000/
```

The machine sees the mesh address of the edge server as the source. A server without the edge role cannot open a connection to a machine, and the command times out there.

On the vRack itself, only WireGuard packets go between the servers of the mesh, apart from the control channel. The mesh uses UDP port 51820, and the egress tunnels the ports from 51821. The metrics of the mesh show the handshake and the bytes of each peer:

```sh
curl -s http://127.0.0.1:9464/metrics | grep '^hopbox_mesh_'
```

A restart or an upgrade of `hopboxd` keeps the device, so the mesh traffic goes on. Do not manage `hbm0` with another tool, such as `wg-quick`. `hopboxd` sets the device back within 2 s.

On the public interface, the host firewall takes UDP port 51820 and the ports of the egress tunnels only from the peer addresses of the servers of the install ([the host firewall](/reference/host-firewall/index.html.md#sets)). A server without a peer address is not in that set. List the set, and the last copy that the server keeps on its disk:

```sh
sudo nft list set inet hopbox_host peers
sudo cat /var/lib/hopbox/hostfw/peers
```

## Docker on the server

A server with Docker drops each forwarded packet that Docker does not know, so the egress of the machines times out. Accept the tap devices in the chain `DOCKER-USER`:

```sh
sudo iptables -I DOCKER-USER -i hbt+ -j ACCEPT
sudo iptables -I DOCKER-USER -o hbt+ -m conntrack --ctstate RELATED,ESTABLISHED -j ACCEPT
```

The first rule lets out the packets of the machines. The second one lets in the replies to them. The table `inet hopbox_node` still drops what a machine must not reach. Docker makes the chain again at each start, without these rules, so add them to the boot of the server.

If Docker came after the first start of `hopboxd`, the table has the forward guard, which drops the forwarded traffic of the containers. Remove the guard, and restart `hopboxd`:

```sh
sudo rm /var/lib/hopbox/network/forward-guard
sudo systemctl restart hopboxd
```

## When something fails

| What you see | Cause and fix |
| --- | --- |
| `hopboxd` stops at its start with `the host names no resolver that a machine can reach` | The resolv.conf of the server names only a loopback resolver. Set `network.nameservers`. |
| The log of `hopboxd` has `the machine boots without network` | The machine has no address, for example on a node without the node network. Its tap device stays down. |
| The egress of a machine times out, but the edge reaches the machine | A host with Docker drops forwarded packets in the chain `FORWARD` of iptables. Accept the tap devices in the chain `DOCKER-USER`, as in [Docker on the server](#docker-on-the-server), or run the node without Docker. |
| The log of `hopboxd` has `the nftables table inet hopbox_node lost a part outside hopboxd` | Something flushed the ruleset, such as a restart of `nftables.service`. `hopboxd` installed the table again. Find what flushed it, and stop it. |
| `no capacity: the subnet ... has no free machine address` | The node holds 4094 machines and held addresses. Delete a machine, and wait 5 minutes for the hold of its address. |
| The log of `hopboxd` has `this server has more than one address on the vRack, so it stays out of the mesh` | Two addresses of the server are in `172.31.0.0/16`. Remove the address that the provider did not set. |
| The log of `hopboxd` has `cannot set up the mesh device. The next check tries again` | The kernel has no module `wireguard`, or another device has the name `hbm0`. Load the module with `sudo modprobe wireguard`, or remove the other device. |
| On a node without the edge role, each connect of a machine to the internet fails at once | The node has no egress edge, so its egress fails closed. Check that `sudo wg show all endpoints` lists a tunnel `hbe0`, and that an edge server is `ready` in `hopboxd admin server list` ([the egress of the machines](/reference/egress/index.html.md#fail-closed)). |
| The log of `hopboxd` has `the reverse path filter is on` | The kernel drops the packets of the egress tunnels, and an edge server takes no egress. Set the sysctl `net.ipv4.conf.all.rp_filter` to 0 ([the egress of the machines](/reference/egress/index.html.md#the-reverse-path-filter)). |
| `wg show hbm0` lists no peer | The control role sends the mesh only to a server whose heartbeat carries its part of the mesh. Check that `hopboxd admin server get` shows `mesh_address` for the server. |
