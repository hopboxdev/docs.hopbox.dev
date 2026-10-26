# The DNS role

> The DNS role: its listeners, zone, answers, and push. Also its TXT records, the parent zone, metrics, and logs.

The DNS role is an authoritative DNS server for the domain of the install. An authoritative server answers from its own zones, and never asks another server. The control role builds the zone and pushes it to each server with the DNS role over the control channel ([the control channel](/reference/control-channel/index.html.md)). This page covers the listeners, the zone, the answers, the push, and the TXT records. It also covers the records of the parent zone, the metrics, and the logs.

The code is in `internal/dns` (the server), `internal/service/dns.go` (the zone), and `internal/control/channel` (the push). The server uses [miekg/dns](https://github.com/miekg/dns).

## Listeners

The DNS role listens on UDP and on TCP at each address of `dns.listen` in the bootstrap config ([the bootstrap config of hopboxd](/reference/hopboxd-config/index.html.md#keys)).

| Item | Value |
| --- | --- |
| Default address | `:53`: port 53 on every IPv4 and IPv6 address of the server |
| Addresses | 1 to 16. Each one is `host:port`. The host is an IP address, or empty for every address. |
| Port `0` | A free port. UDP and TCP of one address get the same port. |
| TCP limits | 2 s to read a query, 2 s to write an answer, and 8 s without a query before the server closes the connection |
| TCP connections | At most 512 open connections at each address. A connection past the bound waits in the backlog of the kernel until another connection closes. So a client that holds many connections cannot take every file descriptor of `hopboxd`. |
| TCP connections of one source | At most 16 open connections from each source at each address. A source is an IPv4 address, or the /64 of an IPv6 address. The server closes a connection past this bound at once. So one client cannot take all 512 connections. |
| Queries on one TCP connection | At most 16. After the 16th answer, the server closes the connection, and the client opens a new one. |

Each connection inside these bounds keeps the limits of the row TCP limits. So a held connection ends after 8 s without a query, and a slow query ends after 2 s.

The unit `dns` of `hopboxd run` opens every socket at its start ([hopboxd](/reference/hopboxd/index.html.md#run)). If another program holds a port, the start fails, and `hopboxd` exits with status 1. The error names the address, such as `dns: listen on udp :53: bind: address already in use`.

Port 53 needs root, or the capability `CAP_NET_BIND_SERVICE`. The unit `hopboxd.service` runs `hopboxd` as root ([Server layout](/reference/server-layout/index.html.md#hopboxdservice)). On a laptop, use a high port, such as `127.0.0.1:15353`.

A stub resolver on the server can hold port 53 on a loopback address. An example is `systemd-resolved` on `127.0.0.53:53`. Then set `dns.listen` to the public addresses of the server, such as `["203.0.113.10:53", "[2001:db8::10]:53"]`.

## The zone of the install

The control role builds one zone from the install config ([the admin API](/reference/admin-api/index.html.md#installconfigservice)): the domain, the public addresses, and the TXT records. The public addresses are the addresses at which the edge servers take traffic. The domain is `example.com` in the examples below.

| Name | Type | Data | Present when |
| --- | --- | --- | --- |
| `example.com` | `SOA` | `ns1.example.com. hostmaster.example.com. <serial> 3600 600 604800 60` | always |
| `example.com` | `NS` | `ns1.example.com.` and `ns2.example.com.` | always |
| `example.com` | `CAA` | `0 issue "<issuer>"` for each CAA issuer, such as `0 issue "letsencrypt.org"` | the zone has a CAA issuer ([CAA records](#caa-records)) |
| `example.com` | `A`, `AAAA` | each public address of the family | the install has an address of the family |
| `*.example.com` | `A`, `AAAA` | each public address of the family | the install has an address of the family |
| `ns1.example.com` | `A`, `AAAA` | the first public address of the family | the install has an address of the family |
| `ns2.example.com` | `A`, `AAAA` | the second public address of the family. With one address in the family, the first. | the install has an address of the family |
| each TXT name | `TXT` | each value of the name | a leader job added a TXT record ([TXT records](#txt-records)) |

Every record has a TTL of 60 seconds. The TTL is the time that a resolver keeps an answer. A change of the addresses reaches each client within a minute.

The domain serves the API and the console, and the wildcard serves the URL of each machine, such as `box1.example.com`. The names `ns1` and `ns2` are reserved, so no machine takes them ([Domain rules](/reference/domain-rules/index.html.md)).

The values of the SOA record:

| Field | Value | Meaning |
| --- | --- | --- |
| Primary name server | `ns1.<domain>.` | The first name server. |
| Mailbox | `hostmaster.<domain>.` | The mailbox of the person responsible for the zone, in DNS form. |
| Serial | the serial of the zones | Grows with each change ([How the zones reach the DNS role](#how-the-zones-reach-the-dns-role)). |
| Refresh, retry, expire | 3600, 600, 604800 | The usual values of RFC 1912. The DNS role has no secondary server, so they have no effect. |
| Minimum | 60 | The TTL of a negative answer (RFC 2308). |

A CAA record controls which certificate authority can issue a certificate for the domain and its names. The CAA records follow the ACME CA of the install ([CAA records](#caa-records)).

An install without a domain has no zone. An install with a domain and no public address has the SOA, NS, CAA, and TXT records only. The names then have no address, so set the public addresses before you delegate the domain:

```sh
sudo hopboxd admin config set --domain example.com --public-address 203.0.113.10 --public-address 2001:db8::10
```

[Domain rules](/reference/domain-rules/index.html.md) gives the rules of the public addresses. [hopboxd](/reference/hopboxd/index.html.md#admin-config-set) gives the flags.

## Answers

The DNS role answers each query from the zones that it has. It never asks another server, so it sets no `RA` flag. It copies the `RD` flag of the query. It has no DNSSEC.

| Query | Response code | `AA` flag | Answer |
| --- | --- | --- | --- |
| Before the first zones arrive | `SERVFAIL` | no | none |
| A name outside every zone, such as `example.org` or the parent `com` | `REFUSED` | no | none |
| A name of the zone with records of the type | `NOERROR` | yes | the records |
| A name of the zone without records of the type | `NOERROR` | yes | none. The authority section holds the SOA. |
| A name that does not exist and that the wildcard covers, such as `box1.example.com` | `NOERROR` | yes | the records of the wildcard, under the name of the query |
| A name that does not exist and that the wildcard does not cover, such as `x.ns1.example.com` | `NXDOMAIN` | yes | none. The authority section holds the SOA. |
| `NS` at the domain | `NOERROR` | yes | the 2 NS records. The additional section holds the addresses of `ns1` and `ns2`. |
| Type `ANY` | `NOERROR` | yes | one `HINFO` record with the text `RFC8482`, as RFC 8482 says |
| Type `AXFR` or `IXFR`, a zone transfer | `REFUSED` | no | none |
| A class other than `IN`, such as `CH` for `version.bind` | `REFUSED` | no | none |
| An opcode other than `QUERY`, such as `NOTIFY` | `NOTIMP` | no | none |
| No question, or more than one | `FORMERR` | no | none |
| EDNS version 1 or later | `BADVERS` | no | none, with an OPT record |

The wildcard follows RFC 4592. For each name that does not exist, the DNS role finds the nearest name above it that exists. If that name is the domain, the wildcard covers the name. So it covers `box1.example.com` and `sub1.box1.example.com`. It does not cover a name below `ns1`, `ns2`, or a TXT name, such as `x._acme-challenge.example.com`. A wildcard answer holds only `A` and `AAAA`. So a `CAA` query for a machine name gets an empty answer, and a certificate authority then uses the CAA record of the domain.

An answer keeps the case of the name in the query. A negative answer holds the SOA with a TTL of 60 seconds.

A query with EDNS gets an OPT record with a UDP size of 1232 bytes. Over UDP, an answer fits in 512 bytes, or in the UDP size of the query up to 1232 bytes. An answer that does not fit loses records and gets the `TC` flag, and the client asks again over TCP. A TCP answer is never cut.

## How the zones reach the DNS role

The control role pushes the whole set of zones in the message `DNSZones` of the control channel ([the control channel](/reference/control-channel/index.html.md#channelresponse)). Each set replaces the set before it. Only a server whose certificate carries the role `dns` gets the zones. The policy action `server.read_dns_zones` allows the read only for such a server ([the control channel](/reference/control-channel/index.html.md#controlservice)).

The control role sends the zones at these times:

- after the first heartbeat of each channel that it records,
- after each change event `dns.zone_changed` ([Control store tables](/reference/control-store/index.html.md#dns_zone)). Each control process pushes to the channels that it holds, so a change in one control process reaches every channel.
- at the next heartbeat after a push whose read of the zones failed.

A change of the domain or of the public addresses publishes the change event in the same transaction. So does each change of a TXT record. An update of the install config that changes neither value publishes nothing.

The serial grows with each change. It is the SOA serial of every zone. The control store keeps it ([Control store tables](/reference/control-store/index.html.md#dns_zone)). Each change raises it by 1. If the Unix time of the change is higher, the serial becomes that time. So the serial also grows after the data of an install was erased. On each channel, the control role sends only a set with a higher serial than the last set. A slow channel gets only the newest set.

The DNS role checks each set before it serves it. If one rule fails, it refuses the whole set and keeps the zones that it has:

| Rule | Bound |
| --- | --- |
| Zones | at most 1024, each origin once |
| Records | at most 65536 in all zones |
| Origin | a lowercase domain name, without the final dot |
| Name of a record | empty for the origin, `*` for the wildcard, or lowercase labels below the origin. A `*` is only the first label. |
| Type | `SOA`, `NS`, `A`, `AAAA`, `TXT`, or `CAA` |
| Data | the text form of RFC 1035, valid for the type, without a newline, `;`, or `$` |
| SOA | exactly 1, at the origin |
| NS | at least 1 at the origin, and none below it. The DNS role serves no delegation. |

The DNS role also refuses a set with a lower serial than the set that it serves, with the error `dns: old zones`. A set with the same serial replaces the set that it serves. A new channel sends such a set.

A zone answers for each name below its origin. When two zones cover a name, the zone with the longer origin answers.

## TXT records

A leader job of the control role adds a TXT record to the zone, and removes it again. The ACME job of DNS-01 uses it: it puts the challenge of each name at `_acme-challenge.<domain>` (RFC 8555 section 8.4). An order for the domain and its wildcard needs 2 values at that name at once. The API is in Go, in the service layer. No RPC and no command changes a TXT record.

| Function | Description |
| --- | --- |
| `Service.AddDNSTXTRecord(ctx, lease, name, value) (serial uint32, err error)` | Adds the value at the name. A record that exists already changes nothing. |
| `Service.RemoveDNSTXTRecord(ctx, lease, name, value) (serial uint32, err error)` | Removes the value at the name. A record that does not exist changes nothing. |
| `Service.RemoveDNSTXTRecords(ctx, lease, name) (serial uint32, err error)` | Removes every value at the name. A name without a record changes nothing. The ACME job calls it before each order, to remove the challenges of an order whose leader stopped before its cleanup. |

The functions take the lease of the leader job. The write checks the lease in its transaction, so a leader that lost its lease writes nothing, and gets `lease.ErrStale` ([Control store tables](/reference/control-store/index.html.md#leases)). Each one returns the serial of the zones after the change. If nothing changed, they return the serial now. A server with the DNS role serves the change once it serves that serial or a later one. `hopbox_dns_zone_serial` on the server shows the serial that it serves ([Metrics](#metrics)).

| Item | Rule |
| --- | --- |
| Name | Relative to the domain, such as `_acme-challenge`. 1 to 63 characters. Labels joined by dots. Each label starts with `_` and holds `a-z`, `0-9`, `-`, and `_`. A label without `_` can be a machine name. A TXT record below a machine name hides the wildcard address of that machine. |
| Value | 1 to 255 characters of printable ASCII, without `"`, `\`, `;`, or `$`. The DNS role accepts each value that passes, so a TXT record never makes it refuse a set of zones. |
| Count | At most 64 records in the zone. An add past the limit fails with `ErrFailedPrecondition`. An add of a record that exists passes also on a full zone, because it changes nothing. Each change locks the row of `dns_zone` first, so two adds at once never pass the limit. |

A name or a value that breaks a rule fails with `ErrInvalidArgument`. Each change writes an audit entry with the actor `system`: `dns.add_txt_record` or `dns.remove_txt_record`, with the target kind `dns_txt_record`, the name as the target ID, and the detail `serial`. The entry holds no value. Each change also writes the log record `changed a TXT record of the zone` with `component=service`, `action`, `name`, and `serial`. A remove of every value at a name writes the audit entry `dns.remove_txt_records` instead, with the details `serial` and `removed`, the number of values. Its log record is `removed the TXT records of a name`, with `name`, `removed`, and `serial`.

## CAA records

The CAA records of the zone name the CAs that can issue a certificate for the domain (RFC 8659). The zone has one record `0 issue "<issuer>"` for each CAA issuer. The control store keeps the issuers in `dns_zone.caa_issuers` ([Control store tables](/reference/control-store/index.html.md#dns_zone)). A new install has one issuer, `letsencrypt.org`, the CAA identity of Let's Encrypt, which is the default ACME CA ([the certificate of the install](/reference/acme/index.html.md)).

The ACME job of the leader sets the issuers. Before each attempt, it reads the directory of the CA. Then it sets the issuers to the CAA identities that the directory names (RFC 8555 section 7.1.1, `meta.caaIdentities`). So the CAA records follow the CA of the directory. A directory that names no CAA identity leaves the zone without a CAA record, so any CA can issue. RFC 8555 makes `meta.caaIdentities` optional, so a CA that checks CAA can leave it out. Let's Encrypt names `letsencrypt.org`.

| Function | Description |
| --- | --- |
| `Service.SetDNSCAAIssuers(ctx, lease, issuers) (serial uint32, err error)` | Replaces the issuers. An empty list removes every CAA record. The same issuers again change nothing. |

The function takes the lease of the leader job, and checks it in its transaction, as the TXT functions do. It returns the serial of the zones after the change. If nothing changed, it returns the serial now.

| Item | Rule |
| --- | --- |
| Issuer | A DNS name of 1 to 253 characters. Each label holds 1 to 63 of `a-z`, `0-9`, and `-`, and does not start or end with `-`. hopbox keeps it in lowercase, without a final dot. |
| Issuers | At most 8, each once. A second copy of an issuer is dropped. |

An issuer that breaks a rule fails with `ErrInvalidArgument`. Each change writes an audit entry with the actor `system` and the action `dns.set_caa_issuers`, without a target. Its details are `caa_issuers` and `old_caa_issuers`, joined by commas, and `serial`. It also writes the log record `changed the CAA issuers of the zone` with `component=service`, `caa_issuers`, `old_caa_issuers`, and `serial`.

## Delegate the domain

The parent zone sends resolvers to the DNS role with 2 NS records and their glue records. The parent zone is the zone that holds the domain of the install. For `example.com`, it is the zone `com`, and the registrar of the domain makes the records in it. For a subdomain such as `hop.example.net`, it is the zone `example.net`. A glue record is an address record of a name server in the parent zone. The parent zone needs it, because the name servers are below the domain that they serve.

Make these records in the parent zone, at your registrar or your DNS provider:

| Name | Type | Data |
| --- | --- | --- |
| `example.com.` | `NS` | `ns1.example.com.` |
| `example.com.` | `NS` | `ns2.example.com.` |
| `ns1.example.com.` | `A` | the first IPv4 public address |
| `ns2.example.com.` | `A` | the second IPv4 public address. With one IPv4 address, the first. |
| `ns1.example.com.` | `AAAA` | the first IPv6 public address. Leave it out for an install without IPv6. |
| `ns2.example.com.` | `AAAA` | the second IPv6 public address. With one IPv6 address, the first. Leave it out for an install without IPv6. |

The glue records must hold the same addresses as `ns1` and `ns2` in the zone of the install. A resolver that finds two different answers can use either.

A delegated zone needs at least 2 name servers. An install with one public address in a family points both glue records of that family at it. A bigger install runs the DNS role on two edge servers, each with its own address.

The DNS role of each name server must listen on port 53 of its public address, on UDP and TCP. Open both in every firewall in front of the server.

Check the DNS role directly before you change the parent zone:

```sh
dig @203.0.113.10 example.com SOA +norec
dig @203.0.113.10 box1.example.com A +norec
```

The answer has the flag `aa` and the status `NOERROR`. After the change of the parent zone, check the delegation from the root down:

```sh
dig +trace box1.example.com A
```

The last step of the trace names `ns1.example.com` or `ns2.example.com`, and answers with a public address. [Serve the domain of an install with the DNS role](/guide/dns-role/index.html.md) has the whole procedure.

## Metrics

The DNS role adds these metrics to the metrics listener of `hopboxd` ([Metrics](/reference/metrics/index.html.md)). A server without the DNS role has neither metric.

| Metric | Type | Labels | Description |
| --- | --- | --- | --- |
| `hopbox_dns_queries_total` | counter | `transport`, `rcode` | The queries that the DNS role answered. `transport` is `udp` or `tcp`. `rcode` is the response code, such as `NOERROR`, `NXDOMAIN`, `REFUSED`, `SERVFAIL`, `FORMERR`, `NOTIMP`, or `BADVERS`. |
| `hopbox_dns_zone_serial` | gauge | none | The serial of the zones that the DNS role serves, or 0 before the first zones arrive. |

```text
hopbox_dns_queries_total{rcode="NOERROR",transport="udp"} 3
hopbox_dns_queries_total{rcode="REFUSED",transport="udp"} 2
hopbox_dns_zone_serial 1.792249094e+09
```

## Logs

The DNS role writes these records with `component=dns`. It writes no record for each query. The metrics count the queries.

| Level | Message | Fields | When |
| --- | --- | --- | --- |
| info | `dns listening` | `network` (`udp` or `tcp`), `addr` | Once for each socket, at the start of the unit `dns`. |
| info | `serving dns zones` | `serial`, `zones` (the origins, with the final dot), `records` | The DNS role serves a set with a new serial. |
| debug | `dns: write a response failed` | `transport`, `remote`, `err` | The answer to a client failed, for example because the client closed its TCP connection. |

The channel writes these records about the zones ([the control channel](/reference/control-channel/index.html.md#logs)):

| Level | Component | Message | When |
| --- | --- | --- | --- |
| error | `node` | `the dns role refused the zones of the control plane, and keeps the zones that it has` | A set broke a rule of [How the zones reach the DNS role](#how-the-zones-reach-the-dns-role). The fields `serial` and `err` name the set and the rule. The channel stays open. |
| info | `node` | `the dns role keeps newer zones than the zones of the control plane` | A set had a lower serial than the set that the DNS role serves, with the error `dns: old zones`. A newer push came first, so the DNS role serves the newer set. The fields `serial` and `err` name the old set. |
| warn | `node` | `the control plane sent dns zones to a server without the dns role` | A server without the DNS role got a set. |
| warn | `channel` | `reading the dns zones of a server failed. The next heartbeat tries again` | A read of the zones for a push failed. |

Example:

```json
{"time":"2026-10-17T14:58:13.819135Z","level":"INFO","msg":"serving dns zones","component":"dns","serial":1792249094,"zones":["example.com."],"records":12}
```
