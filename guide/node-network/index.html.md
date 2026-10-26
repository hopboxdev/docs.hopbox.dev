# Check the network of a machine

> How to check the network of a machine: its address, its egress, what it cannot reach, and its traffic counters.

This guide follows the network of one machine on a server with the node role. It finds the address of the machine, checks its egress and the destinations that it cannot reach, and reads its traffic counters. [The node network](/reference/node-network/index.html.md) describes each part.

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

The guest agent wrote the resolvers of the node to `/etc/resolv.conf` of the machine:

```sh
hopbox exec box1 cat /etc/resolv.conf
```

The node routes the address to the tap device of the machine. The tap device has the alias `hopbox machine <id>`:

```sh
ip -d link show | grep -B1 'alias hopbox machine'
ip route show 10.240.0.1
```

```text
10.240.0.1 dev hbt3f2c9a0b1d4e proto static scope link
```

## Check the egress

A machine reaches the internet. The node gives its packets the address of the server:

```sh
hopbox exec box1 curl -sS https://ifconfig.me
```

The command prints the public address of the server, not the address of the machine.

## Check what the machine cannot reach

A machine reaches no service of its node, no private range, and no other machine. Each of these commands fails after the timeout of 3 seconds:

```sh
hopbox exec box1 curl -m 3 http://169.254.0.1:9464/metrics
hopbox exec box1 curl -m 3 http://192.168.1.1/
hopbox exec box1 curl -m 3 http://10.240.0.2:8000/
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

The sets `machines` and `inbound` hold the tap device of each machine, each with its counter:

```text
	set machines {
		type ifname . ipv4_addr
		counter
		elements = { "hbt3f2c9a0b1d4e" . 10.240.0.1 counter packets 1520 bytes 194311 }
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
