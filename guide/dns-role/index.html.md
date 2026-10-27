# Serve the domain of an install with the DNS role

> How to run the DNS role, set the domain and the public addresses, and query the zone. Also how to delegate the domain of an install.

This guide runs the DNS role next to the control role, first on a Mac and then on a server. The DNS role answers DNS queries for the domain of the install, such as `example.com`. The domain then points at the public addresses of the install, and each machine name, such as `box1.example.com`, points at them too. At the end, the guide delegates the domain from its parent zone, so every resolver on the internet finds the DNS role.

[The DNS role](/reference/dns/index.html.md) covers the zone, the answers, and the push in full.

The release files of hopbox are not public yet. The hopbox team gives them on request.

## How the zone reaches the DNS role

The DNS role holds no zone of its own. The control role builds the zone from the install config: the domain, the public addresses, and the TXT records of ACME. Then it pushes the zone over the control channel to each server with the DNS role ([the control channel](/reference/control-channel/index.html.md)). So a server with the DNS role enrolls with the control role, as a node does.

Each change of the domain or the addresses raises the serial of the zone. Within a second, the DNS role answers with the new zone. The DNS role keeps the last zone on its disk, so after a restart it answers while no control server answers ([the DNS role](/reference/dns/index.html.md#zones-on-the-disk)). It bounds the UDP answers of each client network. So forged queries send a victim at most 20 full answers each second for each answer of the zone ([the DNS role](/reference/dns/index.html.md#rate-limit-on-udp)).

## What you need

- For the steps on a server: a server installed from the release bundle, as in [Install hopbox on a Debian 13 server by hand](/guide/install-debian/index.html.md) or [Install hopbox on an OVHcloud server with Terraform](/guide/install-on-ovhcloud/index.html.md).
- For the steps on a Mac: a checkout of the source of hopbox, built with `make build`. The source is not public yet, so these steps are for the people who work on hopbox.
- For the steps on a Mac: the bundled Postgres. `make pg` builds it on the first run and prints its dir.
- `dig`. macOS has it.

## Run the DNS role on a Mac

Port 53 needs root, so the DNS role on a Mac listens on a high port. Do not use port 5353, because macOS uses it for multicast DNS.

### Write the bootstrap config

A unix socket path has at most 103 bytes on macOS, so put the sockets in a short dir under `/tmp`:

```sh
mkdir -p /tmp/hbd && chmod 700 /tmp/hbd
cat > /tmp/hbd/hopboxd.toml <<EOF
roles = ["control", "dns"]
data_dir = "$HOME/hopbox-dns-dev"
run_dir = "/tmp/hbd"

[dns]
listen = ["127.0.0.1:15353"]

[postgres]
install = "$(make pg)"
EOF
```

The server enrolls at the default `channel.control_urls`, `["https://127.0.0.1:7443"]`, which is the internal listener of the same `hopboxd` ([the bootstrap config of hopboxd](/reference/hopboxd-config/index.html.md#keys)). The `[channel]` keys apply to each server that enrolls, also to a server with the DNS role alone.

### Start hopboxd

```sh
./bin/darwin_arm64/hopboxd run --config /tmp/hbd/hopboxd.toml
```

The units of the control role start first. Then the unit `dns` opens its sockets, and the unit `channel` waits for the enrollment of the server:

```json
{"level":"INFO","msg":"dns listening","component":"dns","network":"udp","addr":"127.0.0.1:15353"}
{"level":"INFO","msg":"dns listening","component":"dns","network":"tcp","addr":"127.0.0.1:15353"}
{"level":"INFO","msg":"waiting for the enrollment of this server","component":"node","dir":"/Users/user1/hopbox-dns-dev/server"}
```

Leave it running, and use a second terminal for the next steps. Set two variables, so the commands stay short:

```sh
H=./bin/darwin_arm64/hopboxd
C=/tmp/hbd/hopboxd.toml
```

The DNS role has no zone yet, on the control channel or on its disk, so it answers `SERVFAIL`. A resolver then asks another name server:

```text
$ dig @127.0.0.1 -p 15353 example.com A +norec | grep status
;; ->>HEADER<<- opcode: QUERY, status: SERVFAIL, id: 39721
```

### Enroll the server with the DNS role

The install needs a key-encryption key before it has a CA. A token for the DNS role names the role `dns`:

```text
$ $H admin --config $C kek rotate
file:1
$ TOKEN="$($H admin --config $C enrollment-token create --roles dns)"
$ HOPBOX_ENROLLMENT_TOKEN="$TOKEN" $H enroll --config $C
hopboxd: enrolled as the server 01a14a5e-b88d-7c9f-81cc-5f5b8e5b1e48. hopboxd run connects it to the control plane.
01a14a5e-b88d-7c9f-81cc-5f5b8e5b1e48
```

A server without the node role runs no machine. So `enroll` runs no preflight check, and the enrollment works on a Mac. Within 2 seconds, the channel connects, and the first heartbeat brings the zones:

```text
$ $H admin --config $C server list
ID                                    POOL     STATE  STATUS  ROLES  HOSTNAME  VERSION  LAST HEARTBEAT
01a14a5e-b88d-7c9f-81cc-5f5b8e5b1e48  default  ready  ok      dns    macbook   v0.1.0   2026-10-17T14:58:05Z
```

The install has no domain yet, so the set of zones is empty. The DNS role now refuses every name:

```text
$ dig @127.0.0.1 -p 15353 example.com A +norec | grep status
;; ->>HEADER<<- opcode: QUERY, status: REFUSED, id: 57040
```

### Set the domain and the public addresses

The public addresses are the addresses at which the edge servers take traffic. On a laptop, use any address. The examples use the documentation ranges `203.0.113.0/24` and `2001:db8::/32`:

```text
$ $H admin --config $C config set --domain example.com --public-address 203.0.113.10 --public-address 2001:db8::10
domain	example.com
public_addresses	203.0.113.10,2001:db8::10
acme_directory_url	https://acme-v02.api.letsencrypt.org/directory
acme_email	-
acme_terms_agreed	false
mail_transport	smtp
version	2
update_time	2026-10-17T14:58:13Z
```

The log of `hopboxd run` shows the new zone:

```json
{"level":"INFO","msg":"serving dns zones","component":"dns","serial":1792249094,"zones":["example.com."],"records":12}
```

### Query the zone

A machine name gets the address through the wildcard. The flag `aa` says that the answer is authoritative:

```text
$ dig @127.0.0.1 -p 15353 box1.example.com A +norec
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 9368
;; flags: qr aa; QUERY: 1, ANSWER: 1, AUTHORITY: 0, ADDITIONAL: 1
...
;; ANSWER SECTION:
box1.example.com.	60	IN	A	203.0.113.10
```

The NS query names `ns1` and `ns2`, with their addresses:

```text
$ dig @127.0.0.1 -p 15353 example.com NS +norec +noall +answer +additional
example.com.		60	IN	NS	ns1.example.com.
example.com.		60	IN	NS	ns2.example.com.
ns1.example.com.	60	IN	A	203.0.113.10
ns1.example.com.	60	IN	AAAA	2001:db8::10
ns2.example.com.	60	IN	A	203.0.113.10
ns2.example.com.	60	IN	AAAA	2001:db8::10
```

The CAA record lets only Let's Encrypt, the default ACME CA, issue certificates for the domain. It follows the ACME CA of the install ([the DNS role](/reference/dns/index.html.md#caa-records)):

```text
$ dig @127.0.0.1 -p 15353 example.com CAA +short
0 issue "letsencrypt.org"
```

A name below `ns1` does not exist, and a name outside the zone gets no answer:

```text
$ dig @127.0.0.1 -p 15353 x.ns1.example.com A +norec | grep -E 'status|SOA'
;; ->>HEADER<<- opcode: QUERY, status: NXDOMAIN, id: 23694
example.com.		60	IN	SOA	ns1.example.com. hostmaster.example.com. 1792249094 3600 600 604800 60
$ dig @127.0.0.1 -p 15353 example.org A +norec | grep status
;; ->>HEADER<<- opcode: QUERY, status: REFUSED, id: 47432
```

TCP works too:

```text
$ dig @127.0.0.1 -p 15353 example.com SOA +tcp +short
ns1.example.com. hostmaster.example.com. 1792249094 3600 600 604800 60
```

### Change the addresses

Set two new IPv4 addresses. The list replaces the old one, so this install now has no IPv6 address:

```sh
$H admin --config $C config set --public-address 203.0.113.11,203.0.113.12
```

Within a second, the domain has both addresses, and `ns2` takes the second one:

```text
$ dig @127.0.0.1 -p 15353 example.com A +short
203.0.113.11
203.0.113.12
$ dig @127.0.0.1 -p 15353 ns2.example.com A +short
203.0.113.12
```

The metrics count the queries, and show the serial that the DNS role serves ([the DNS role](/reference/dns/index.html.md#metrics)):

```sh
curl -s http://127.0.0.1:9464/metrics | grep '^hopbox_dns'
```

### Stop hopboxd

Press Ctrl-C in the first terminal. `hopboxd` stops the unit `channel` first, then the unit `dns`, then the units of the control role. To start from nothing, remove `~/hopbox-dns-dev` and `/tmp/hbd`.

## Delegate the domain of the install

In this example, the install serves `example.com` from one server, with the public addresses `203.0.113.10` and `2001:db8::10`. A delegated zone needs two name servers. The install has one server, so both name servers point at it. A bigger install runs the DNS role on two edge servers, one for each name server ([Run edge servers apart from the nodes](/guide/edge-servers/index.html.md)).

### Give the server the DNS role

A server with every role lists all four in its bootstrap config:

```toml
roles = ["control", "node", "dns", "edge"]
```

`hopboxd install --roles control,node,dns,edge` writes this list ([hopboxd](/reference/hopboxd/index.html.md#install)). The roles of a server are in its certificate, so a server that enrolled without `dns` must enroll again to get it ([Enrollment and the install PKI](/reference/enrollment/index.html.md)). The token for this server names the three roles that enroll:

```sh
TOKEN="$(sudo hopboxd admin enrollment-token create --roles node,dns,edge)"
```

The tested OS runs `systemd-resolved` on `127.0.0.53:53`, so the DNS role cannot listen on port 53 of every address. The Terraform provider writes `dns.listen` with port 53 of each public address before the install ([hopbox Terraform provider](/reference/terraform-provider/index.html.md#create)). Open UDP and TCP port 53 in each firewall in front of the server.

### Set the public addresses

The provider sets the public addresses of the install at the create of the server. To set them by hand:

```sh
sudo hopboxd admin config set --public-address 203.0.113.10 --public-address 2001:db8::10
```

### Check the server before the delegation

From your laptop, ask the server directly. The answer must have the flag `aa` and the address of the server:

```sh
dig @203.0.113.10 example.com SOA +norec
dig @203.0.113.10 box1.example.com A +norec
```

### Make the records in the parent zone

The parent zone holds the delegation: the NS records and the glue records. For a domain such as `example.com`, the records are in the zone `com`, and the registrar of the domain makes them. Give the registrar the two name servers and their addresses. For a subdomain such as `hop.example.net`, add the records in the zone `example.net`. For `example.com`, the records are:

| Name | Type | Target |
| --- | --- | --- |
| `example.com.` | `NS` | `ns1.example.com.` |
| `example.com.` | `NS` | `ns2.example.com.` |
| `ns1.example.com.` | `A` | `203.0.113.10` |
| `ns2.example.com.` | `A` | `203.0.113.10` |
| `ns1.example.com.` | `AAAA` | `2001:db8::10` |
| `ns2.example.com.` | `AAAA` | `2001:db8::10` |

The `A` and `AAAA` records are the glue records. They must hold the same addresses as `ns1` and `ns2` in the zone that the DNS role serves. Without IPv6 on the server, leave out the two `AAAA` records. For a subdomain, the parent zone must have no other record at the subdomain or below it, because the delegation hands those names to the DNS role.

### Check the delegation

The parent zone can take some minutes to publish the change. Then follow the delegation from the root:

```sh
dig +trace box1.example.com A
```

The last step names `ns1.example.com` or `ns2.example.com`, and gives `203.0.113.10`. A public resolver gives the same answer:

```sh
dig @1.1.1.1 box1.example.com A +short
```

The TTL of each record is 60 seconds. So a later change of the public addresses reaches every resolver within a minute.
