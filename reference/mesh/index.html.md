# The mesh

> The WireGuard mesh of the servers over the vRack: the device hbm0, which servers join, the mesh addresses, the peers and their allowed IPs, the routes, a restart, metrics, logs, and errors.

The network redesign of 2026-10-27 changes this page. hopbox leaves the vRack and the OVHcloud API, and the servers talk over their public addresses (Q90). This page still describes the current code. Milestones N26 to N32 change the code and this page. Before you set up a fleet on the vRack or with an IPv6 /56, read Q90.

The mesh connects the servers of an install over the public internet with kernel WireGuard. Each server is a peer at its peer address: the public IPv4 address at which the other servers reach it ([Enrollment and the install PKI](/reference/enrollment/index.html.md#peer-addresses)). WireGuard encrypts the mesh traffic between servers. An edge server reaches the machines of each node over the mesh, and a cold move copies the disk of a machine over it ([Cold move](/reference/cold-move/index.html.md)). The control channel stays outside the mesh, on the peer addresses too.

The server side is in `internal/mesh`. The control role keeps the members of the mesh in `internal/service/mesh.go`, and pushes the mesh on the control channel from `internal/control/channel/mesh.go`.

## Values

| Item | Value |
| --- | --- |
| Device | `hbm0`, a kernel WireGuard device in the network namespace of `hopboxd`. `hopboxd` makes it with netlink, and sets its peers with wgctrl. |
| Mesh range | `10.239.0.0/16`. Each server of the mesh has one mesh address in it. The range is a constant. |
| Address of the device | The mesh address of the server, with the prefix length 16, such as `10.239.0.3/16`. |
| UDP port | `51820`. The kernel takes WireGuard packets on this port of every address of the server. The peers send them to its peer address. |
| MTU | `1420`. A packet of a tap device has at most 1420 bytes, so it fits ([the node network](/reference/node-network/index.html.md#addresses)). |
| Private key | `hopboxd` makes the key at the same time as the device, and keeps no copy. The key lives only in the kernel. |
| Public key | 32 bytes. Each heartbeat carries it. `wg show hbm0` prints it in base64. |
| Routes | Protocol `104`, so `ip route show proto 104` lists the routes of the mesh. |

## Which servers join

A server joins the mesh through its control channel. A server that joins the mesh has both of these:

- an enrolled server: a server with the role `node`, `dns`, or `edge`, which keeps a control channel ([the control channel](/reference/control-channel/index.html.md))
- a peer address in the table `servers` ([Control store tables](/reference/control-store/index.html.md#servers)). Its enrollment token names it, or `hopboxd install --peer-address` and `hopboxd admin server set-peer-address` set it ([Enrollment and the install PKI](/reference/enrollment/index.html.md#peer-addresses)).

The mesh looks for no address on the interfaces of the server. So an address of a Docker network, or of another private network, never becomes an endpoint. Only the control role names the endpoints, from the peer addresses in its store. A heartbeat never changes them.

These servers stay out of the mesh:

| Server | Why | Log |
| --- | --- | --- |
| A server with only the role `control` | It keeps no control channel. The other servers reach its internal listener at its peer address ([the internal listener](/reference/internal-listener/index.html.md)). | none |
| A server without a peer address, such as the first server of an install with one server | The other servers have no endpoint for it. Its device exists, and its heartbeats carry its key, but the control role adds it to no mesh. | none |
| A server without the kernel module `wireguard`, or without `CAP_NET_ADMIN` | `hopboxd` cannot make the device. | `cannot set up the mesh device. The next check tries again` at level error |
| A server outside Linux | The mesh needs the kernel WireGuard and netlink. | `the mesh needs Linux, so this server stays out of the mesh` at level info |

A server that stays out works as before. The control role sends it no mesh, so its host firewall gets no sets from the mesh either ([the host firewall](/reference/host-firewall/index.html.md#sets)).

A server that gets a peer address later joins at its next heartbeat. A change of the peer address of a member reaches the other servers at their next heartbeats, as their new endpoint of that member.

## How a server joins

1. The `channel` unit of `hopboxd run` checks the device at its start, before the first heartbeat, and then every 2 s ([hopboxd](/reference/hopboxd/index.html.md#run)). When the device does not exist, the check makes it with a new private key, sets the MTU and the port, and brings it up.
2. Each heartbeat carries `mesh`: the public key of the device ([the control channel](/reference/control-channel/index.html.md#meshmember-meshpeers-and-meshpeer)).
3. The control role records it in the table `mesh_members`, in the transaction of the heartbeat ([Control store tables](/reference/control-store/index.html.md#mesh_members)). The first such heartbeat of a server with a peer address gives it the first free address of the mesh range, from `10.239.0.1`. The address stays with the server for its whole life. A later heartbeat with another key updates the row, and the address stays. Each new row and each update writes the audit entry `server.mesh`.
4. After each heartbeat that it records, the control role reads the mesh as that server sees it: the mesh address of the server, and every other server of the mesh that is not decommissioned. If the mesh differs from the last one that it sent on that channel, it sends `mesh_peers` ([Mesh pushes](/reference/control-channel/index.html.md#mesh-pushes)).
5. The device takes the mesh: its address, one peer for each other server, and a route for each allowed IP of a peer outside the mesh range.

A change of one server reaches every other server within one heartbeat: the heartbeat of the server that changed records the change, and the next heartbeat of each other server takes it. A heartbeat comes every 5 s.

## The peers

Each peer of the device is another server of the mesh:

| Field of the peer | Value |
| --- | --- |
| Public key | The key of the last heartbeat of that server. |
| Endpoint | Its peer address and port `51820`, such as `203.0.113.12:51820`. The control role takes it from `servers.peer_address`, never from a heartbeat. |
| Allowed IPs | See [Allowed IPs](#allowed-ips). WireGuard sends a packet to the peer whose allowed IPs hold its destination. It drops a packet from a peer with a source outside the allowed IPs of that peer. |
| Egress | True for an edge server that takes the egress of the machines now, an egress edge ([the egress of the machines](/reference/egress/index.html.md#egress-edges)). The device does not use it. It hands the egress edges and the nodes of each mesh to the egress tunnels. |

An unresponsive server stays a peer. So when it comes back, the mesh works again at once. A decommissioned server leaves the mesh of the other servers at their next heartbeat.

### Allowed IPs

| Server | Allowed IPs |
| --- | --- |
| A server with the node role | Its mesh address as a `/32`. Its node subnet, such as `10.240.16.0/20`, and the IPv6 `/64` of that subnet, such as `fd65:1c74:e769:1::/64`, once it has one: the control role gives a node its subnet at its first machine ([the node network](/reference/node-network/index.html.md#addresses)). The `/32` and the `/128` of each machine on it whose address is outside its subnet, such as a machine that moved to it ([the node network](/reference/node-network/index.html.md#moved-machines)). |
| Another server | Its mesh address as a `/32`. |

The device refuses a mesh with an allowed IP outside these ranges: the mesh range, the machine range `10.240.0.0/12`, and the IPv6 range of the machines `fd65:1c74:e769::/48`. So no push can route a peer address, and so the control channel, over the device.

## Routes

The address of the device routes the mesh range over it: `10.239.0.0/16 dev hbm0 proto kernel scope link`. For each allowed IP of a peer outside the mesh range, the device adds a route over it with the mesh address as its source. On an edge server of a mesh with two nodes, the routes look like this:

```console
$ ip route show proto 104
10.240.0.0/20 dev hbm0 scope link src 10.239.0.3
10.240.16.0/20 dev hbm0 scope link src 10.239.0.3
```

So an edge server reaches the machines of each node over the mesh, from its mesh address. A node routes the subnets of the other nodes over the mesh in the same way. The mesh changes only the routes with the protocol `104` on `hbm0`.

Another route of the host can have the destination of a route of the mesh, such as the route of a tap device. Then the mesh leaves its route out, and logs `a route of the mesh has the destination of another route of the host, so the mesh leaves it out`. Each check tries that route again, and logs the clash only once. Once the other route goes away, the next check adds the route of the mesh.

`wg show hbm0` shows the device and its peers:

```console
$ sudo wg show hbm0
interface: hbm0
  public key: <public key of this server>
  private key: (hidden)
  listening port: 51820

peer: <public key of a node>
  endpoint: 203.0.113.12:51820
  allowed ips: 10.239.0.2/32, 10.240.16.0/20
  latest handshake: 12 seconds ago
  transfer: 18.42 KiB received, 9.10 KiB sent
```

## The host firewall

The [host firewall](/reference/host-firewall/index.html.md) of each server takes the port `51820` on its public interface only from the peer addresses of the servers, the set `peers` ([the host firewall](/reference/host-firewall/index.html.md#sets)). It takes port 22 of a server that is not an edge server only from the peer addresses of the edge servers, the set `jump_hosts`.

Each mesh holds these sets, in its field `firewall` ([the control channel](/reference/control-channel/index.html.md#mesh-pushes)). On a server without the control role, the agent hands them to the host firewall before the device takes the mesh. A server with the control role takes them from its store instead.

## The egress tunnels

The egress tunnels `hbe0`, `hbe1`, and so on run beside `hbm0` on the peer addresses, not inside the mesh ([the egress of the machines](/reference/egress/index.html.md)). They use the private key of `hbm0`, so they wait for the device. Their UDP ports start at `51821`. A node without the edge role has one tunnel to each egress edge of the mesh. An edge server has one tunnel with each node of the mesh as a peer. The allowed IPs of a node in that tunnel are its allowed IPs of the mesh outside the mesh range.

## Traffic of the edge servers to machines

An edge server opens its connections to a machine on another node from its mesh address, over the mesh. An edge server without the node role binds each such connection to `hbm0`, so it never leaves through another interface ([the edge role](/reference/edge/index.html.md#the-way-to-the-machines)). On the node, the table `inet hopbox_node` lets a new TCP connection from `hbm0` reach a tap device only from an edge server. It gives that connection the mark of the node ([the node network](/reference/node-network/index.html.md#what-can-reach-a-machine)). The device hands the mesh addresses of the edge servers to the node network with each new mesh. Each server of the mesh is a peer, but only an edge server reaches a machine.

## A restart, a stop, and a reboot

| Event | What happens to the device |
| --- | --- |
| A restart or an upgrade of `hopboxd` | The device stays, with its key, its address, its peers, and its routes. The kernel goes on forwarding. The new `hopboxd` keeps the device as it is until the first mesh of the control role, which is the same mesh, so nothing changes. On a node, the table `inet hopbox_node` keeps the edge servers of the table before the restart ([the node network](/reference/node-network/index.html.md#at-the-start-and-at-the-stop-of-hopboxd)). So the edge servers reach the machines also while no control server runs. |
| A stop of `hopboxd` | The device stays, and the mesh traffic goes on. |
| A reboot | The device is gone. At its start, `hopboxd` makes it again with a new key. The first heartbeat carries the new key, and the other servers take it within one heartbeat. |
| `ip link del hbm0` | The next check, within 2 s, makes the device again with a new key, and sets the mesh of the last push. The other servers take the new key within one heartbeat. |
| A change of the address, the peers, or the routes outside `hopboxd` | The next check sets them again, and logs `the mesh device lost a part outside hopboxd, so the mesh set it again` with the parts. Another tool can change the device again before each check. Then each check counts a repair, but logs only once, until a check finds nothing to repair. |

Do not manage `hbm0` with `wg-quick`, `systemd-networkd`, or another tool. `hopboxd` sets the device back at each check.

## Refused mesh

The device checks each mesh that the control role pushes. It refuses a mesh with one of these faults, and keeps the mesh that it has:

- an address outside the mesh range
- more than 4096 peers
- a peer whose server ID is not a UUID
- a key that is not 32 bytes, the key of this server, or the key of another peer
- an endpoint that is not a peer address, or without a port. A peer address is one public unicast IPv4 address ([Enrollment and the install PKI](/reference/enrollment/index.html.md#peer-addresses)).
- an allowed IP outside the ranges of [Allowed IPs](#allowed-ips), or with bits set after its prefix length

The node agent then logs `the mesh device refused the mesh of the control plane, and keeps the peers that it has` at level error, with the fields `address`, `peers`, and `err`. The next change of the mesh sends a new one.

## Where the mesh shows

| Place | What it holds |
| --- | --- |
| `mesh_members` | The mesh address and the public key of each server of the mesh ([Control store tables](/reference/control-store/index.html.md#mesh_members)). |
| `servers.peer_address` | The endpoint of each server of the mesh ([Control store tables](/reference/control-store/index.html.md#servers)). |
| The admin API | The field `mesh` of `Server`: the mesh address and the public key in base64 ([the admin API](/reference/admin-api/index.html.md#server)). The field `peer_address` of `Server` is the endpoint. |
| `hopboxd admin server get` | The lines `mesh_address` and `mesh_public_key`, and the key `mesh` of the JSON object ([hopboxd](/reference/hopboxd/index.html.md#admin-server-get)). |
| The audit log | `server.mesh`, for each server that joins the mesh, and for each new key. |

## Metrics

The metrics listener serves these metrics on each server with the device ([Metrics](/reference/metrics/index.html.md#metrics-of-the-mesh)). The device reads the counters of WireGuard at each scrape. The label `server` is the ID of the peer.

| Metric | Type | Labels | Meaning |
| --- | --- | --- | --- |
| `hopbox_mesh_peers` | gauge | none | The peers of the device. |
| `hopbox_mesh_peer_last_handshake_seconds` | gauge | `server` | The Unix time of the last WireGuard handshake with the peer, or 0 before the first one. WireGuard makes a handshake at least every 2 minutes while traffic flows. |
| `hopbox_mesh_peer_receive_bytes_total` | counter | `server` | The bytes that the device got from the peer. |
| `hopbox_mesh_peer_transmit_bytes_total` | counter | `server` | The bytes that the device sent to the peer. |
| `hopbox_mesh_repairs_total` | counter | none | The changes to the device after it lost a part outside `hopboxd`. |

## Logs

The device logs with `component=mesh`. A check logs a state only after a change of the state. So a check every 2 s does not repeat a line.

| Message | Level | Fields | When |
| --- | --- | --- | --- |
| `the mesh needs Linux, so this server stays out of the mesh` | info | none | `hopboxd` runs outside Linux, such as in a test on a Mac. |
| `made the mesh device` | info | `device`, `public_key` | A check made the device, with a new key. |
| `the mesh device is ready` | info | `device`, `public_key` | A check found the device ready, for the first time or after a change. |
| `cannot set up the mesh device. The next check tries again` | error | `device`, `err`, `retry_in` | A check failed. The heartbeats carry no part of the mesh while the device has no key. |
| `the mesh device took the mesh of the control plane` | info | `address`, `peers` | The device set a new mesh. |
| `the mesh device lost a part outside hopboxd, so the mesh set it again` | warn | `device`, `parts` | A check set again what changed outside `hopboxd`: `device`, `link`, `address`, `peers`, or `routes`. The checks after it log again only after a check without a repair, or with other parts. |
| `a route of the mesh has the destination of another route of the host, so the mesh leaves it out` | warn | `destination`, `device` | See [Routes](#routes). Once for each clash. |

The other parts of the mesh log with their own components:

| Message | Level | Component | When |
| --- | --- | --- | --- |
| `the mesh device refused the mesh of the control plane, and keeps the peers that it has` | error | `node` | See [Refused mesh](#refused-mesh). |
| `the control plane sent the mesh to a server outside the mesh` | warn | `node` | A server without the device got a mesh. |
| `reading the mesh for a server failed. The next heartbeat tries again` | warn | `channel` | The read of the mesh from the control store failed. |
| `the server stays out of the mesh, or keeps its old key there` | error | `service` | A heartbeat carried the key of another server of the mesh, or the mesh range has no free address. The heartbeat is recorded without its part of the mesh. |
| `the heartbeat has a bad part of the mesh, so the control plane records it without that part` | warn | `service` | A heartbeat carried a key that is not 32 bytes. The channel stays open. |
| `the machines take the connections of the edge servers over the mesh` | info | `netcfg` | The node network took the edge servers of a new mesh, with the field `edges`. |
| `the node network cannot take the edge servers of the mesh. The next change of the mesh tries again` | error | `netcfg` | The change of the set `edges` of the node network failed. |

## Errors

| Error | Cause |
| --- | --- |
| `make the device hbm0: ... hopboxd needs CAP_NET_ADMIN and the kernel module wireguard` | The kernel has no WireGuard, or `hopboxd` runs without `CAP_NET_ADMIN`. The server stays out of the mesh. |
| `the device hbm0 is a device of type <type>, not wireguard. Remove it, and hopboxd makes the mesh device` | Another device took the name `hbm0`. |
| `mesh: the address "<address>" is not in the mesh range 10.239.0.0/16` | See [Refused mesh](#refused-mesh). |
| `mesh: the endpoint "<endpoint>" of the peer <server> is not a peer address with a port` | See [Refused mesh](#refused-mesh). |
| `invalid argument: the heartbeat: the mesh key has <n> bytes, not 32` | A heartbeat with a bad key, in the field `err` of the log above. The control role records the heartbeat without its part of the mesh. |

## Tests

| Test | Where it runs | What it checks |
| --- | --- | --- |
| The tests of `internal/mesh` without root | `make test`, also on a Mac | The device of a fake kernel: a device without a vRack, the peers at their peer addresses, a repair and its log, a restart, a refused mesh, and a route that another route holds and then frees. A refused mesh includes an endpoint on the vRack, on a Docker bridge, on loopback, in the mesh range, or of IPv6. |
| `TestMeshDevice` of `internal/mesh` | `make test-linux`, as root in a privileged container | Two servers in network namespaces with the real kernel WireGuard, each at its peer address on a veth. It checks the device, and a connection between the mesh addresses and to the subnet of a node. It also checks a new `hopboxd`, a deleted device, and a route that another route holds. |
| `TestEdgeOverTheMesh` of `internal/node/netcfg` | `make test-linux` | An edge server reaches a machine over the mesh, and a server that is not an edge does not. |
| `TestMeshPeers` of `internal/service` | `make test` | The mesh addresses, the endpoints at the peer addresses, the allowed IPs, a new key and peer address, a key of another server, a server without a peer address that joins once it gets one, and a decommissioned server. |
| `TestEgressEdges` of `internal/service` | `make test` | The field `egress` of a peer ([the egress of the machines](/reference/egress/index.html.md#egress-edges)). |
| `TestMeshReachesEachServer` of `internal/control` | `make test` | The pushes on the control channel. A server without a peer address gets no mesh, and joins once it gets one. A new peer address reaches the other servers. |
| `TestMeshCarriesTheFirewallSets` of `internal/control` | `make test` | The sets of the host firewall in each mesh: the peer addresses of the servers, and those of the edges as the jump hosts. The agent hands them on also when the device refuses the mesh. |
| `TestMeshWithoutFirewallKeepsTheSets` of `internal/node` | `make test` | A mesh of a control role of an older release, without `firewall`, leaves the sets of the host firewall as they are. |
| `TestFirewallSets` of `internal/service` | `make test` | The sets of the host firewall in the store: a server without a peer address, a decommissioned edge, and a used or expired token stay out ([the host firewall](/reference/host-firewall/index.html.md)). |
| `TestControlServersJoinThePeers` of `internal/service` | `make test` | The internal listener of each live control process on a peer address joins the set `peers`. A listener on loopback or on the vRack, and a stopped process, stay out of it. A stopped or silent process stays a known peer until the sweep of its row. |
| `TestFleet` of `test/fleet` | `make test-fleet` | Three servers on the bridge `inet0`. A node and an edge reach each other only over the mesh, and the traffic between them on `inet0` is WireGuard only, apart from the control channel. The mesh and the control channel never use `vrack0`. A restart of `hopboxd` keeps the device and its traffic. A new key reaches the edge within a heartbeat. |
| `TestMeshOnPeerAddresses` of `test/fleet` | `make test-fleet` | Servers without a link to `vrack0` form the mesh over `inet0` at their peer addresses. A Docker bridge and an address of the old vRack range give a server no endpoint. Between the peer addresses, `inet0` carries only WireGuard, TLS on 7443, and SSH from the jump host. A machine behind an edge without `edge.public_ipv4` reaches no internal listener. |
| `TestFirstServerMovesToItsPeerAddress` of `test/fleet` | `make test-fleet` | The move of the internal listener of a first server to its peer address, and a second server that joins the mesh after it. |
