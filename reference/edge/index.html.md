# The edge role

> The edge role: its listeners, TLS, routing, and forwarded requests. Also the machine URLs, their sign-in, EdgeService, metrics, and logs.

The network redesign of 2026-10-27 changes this page. hopbox leaves the vRack and the OVHcloud API, and the servers talk over their public addresses (Q90). This page still describes the current code. Milestones N26 to N32 change the code and this page. Before you set up a fleet on the vRack or with an IPv6 /56, read Q90.

The edge role is the HTTPS front of an install. It takes the HTTPS traffic of every client, and it routes each request by its host name. The domain of the install goes to the control role, which serves the API and the console. A machine URL, such as `box1.example.com`, goes to its machine. A private machine URL needs a sign-in first. This page covers the listeners, TLS, the routing, and the forwarded requests. It also covers the machine URLs, the sign-in of a private machine URL, the public IPv6 address of an edge server, `EdgeService`, the metrics, and the logs. The edge role also runs the SSH front door of the machines, which [the SSH front door](/reference/ssh-front-door/index.html.md) covers.

The edge role is in `internal/edge`. The pushes of the control role are in `internal/control/channel/edge.go`. The routes and the sign-in are in `internal/service/edge.go`. `EdgeService`, `/auth/machine`, and the forwarded requests are in `internal/api/edge.go`.

In the examples, the domain of the install is `example.com`.

## Listeners

The edge role has two HTTP listeners, and the listener of the SSH front door ([the SSH front door](/reference/ssh-front-door/index.html.md#listener)). Their addresses are in the section `[edge]` of the bootstrap config ([the bootstrap config of hopboxd](/reference/hopboxd-config/index.html.md#keys)).

| Item | Value |
| --- | --- |
| HTTPS listener | `edge.https_listen`. Default: `:443`, port 443 on every address. |
| HTTP listener | `edge.http_listen`. Default: `:80`. It only sends each request to HTTPS ([HTTP to HTTPS](#http-to-https)). |
| Protocols | HTTPS: HTTP/2 and HTTP/1.1, by ALPN. HTTP: HTTP/1.1. |
| Open connections | At most 16,384 on the HTTPS listener, and 65,536 on the HTTP listener, of the 1,048,576 file descriptors that `hopboxd.service` allows. The bound of the HTTPS listener keeps the memory of its connections under 4 GiB ([Memory of stalled requests](#memory-of-stalled-requests)). A connection past the bound of the listener gets no refusal. It waits in the backlog of the kernel until another one closes. So a full HTTPS listener stops new console sessions and new CLI connections, with no error page. The open connections go on. `hopbox_edge_open_connections` and `hopbox_edge_connection_limit` show how full each listener is ([Metrics](#metrics)). |
| Open connections of one client | At most 256 on each listener. A client is an IPv4 address, or the /64 of an IPv6 address. The edge role closes a connection past this bound at once, so one client cannot hold every slot of a listener. Each one counts in `hopbox_edge_client_connection_refusals_total{bound="client"}`. |
| Open connections of one IPv6 network | At most 1,024 on each listener, for all /64s of one IPv6 /48 together. The holder of a /48 has 65,536 /64s, enough to fill a listener alone. The edge role closes a connection past this bound at once. The count of its /64 does not matter. Each one counts in `hopbox_edge_client_connection_refusals_total{bound="network"}`. An IPv4 address has no such bound ([Clients behind one network](#clients-behind-one-network)). |
| Request header | At most 64 KiB, also on HTTP/2. A larger header gets `431 Request Header Fields Too Large`. |
| Header timeout | 10 s for the header of a request. A slow client then loses its connection. |
| Idle timeout | 2 min without a request closes a connection. |
| HTTP/2 streams | At most 250 at once on one connection. |
| HTTP/2 receive buffer | 256 KiB for each stream, and 250 times that for the connection. So a stream whose target reads nothing, such as an exec with unread input, stalls alone. The other streams of the connection go on. |
| Open requests | At most 16,384 at once on the HTTPS listener, streams and WebSockets included. A request past the bound gets `503 Service Unavailable` with `Retry-After: 1` and `Hopbox-Not-Run: 1`. |
| Open requests of one client | At most 1,024 at once. A request past the bound gets `429 Too Many Requests` with `Retry-After: 1` and `Hopbox-Not-Run: 1`, so one client cannot take every slot of the 16,384. A client is an IPv4 address or the /64 of an IPv6 address. |
| Open requests of one IPv6 network | At most 4,096 at once, for all /64s of one IPv6 /48 together. A request past the bound gets `429 Too Many Requests` with `Retry-After: 1` and `Hopbox-Not-Run: 1`. The count of its /64 does not matter. So one /48 can take at most a quarter of the 16,384. Watch `hopbox_edge_requests_total{to="refused"}` for the refusals of both bounds. |
| HTTP/2 frames | At most 16 KiB for each frame that a client sends. |
| Request body on HTTP/1.1 | A body that sends nothing for 1 minute ends. The edge role then answers, and closes the connection. A body that keeps sending has no time limit. |
| Answer, HTTP/2 streams, and WebSockets | No timeout, so streams, downloads, and WebSockets can last. |

### Clients behind one network

The bounds of a client and of an IPv6 network count by address. So all people behind one address share its slots, also with their long streams, such as an office or a carrier with one NAT.

| Network | What it shares |
| --- | --- |
| One IPv4 address | 256 connections on each listener, and 1,024 open requests. A carrier NAT can put thousands of people behind one address. IPv4 has no bound of a wider network, because the next addresses often belong to other people. |
| One IPv6 /64 | 256 connections on each listener, and 1,024 open requests. One host usually holds a whole /64. |
| One IPv6 /48 | 1,024 connections on each listener, and 4,096 open requests, for all its /64s together. One site, such as an office or a server of a hosting provider, usually holds a /48. A carrier or a hosting provider can also give many customers a /64 each in one /48. These customers then share the bounds of the /48. |

Without the bound of a /48, 64 /64s of one /48 fill the 16,384 connections of the HTTPS listener. With it, a client needs 16 /48s. A lower bound for each /48 keeps the listener open for more networks, but it stops more users of a shared /48. The bound is 1,024 connections for each /48. If the users of a large network reach the bound, `hopbox_edge_client_connection_refusals_total{bound="network"}` and `hopbox_edge_requests_total{to="refused"}` grow.

### Memory of stalled requests

A request can stall in two ways. Its target reads none of its body, or its client reads none of the answer. A stalled request holds memory in the edge until it ends, because no timeout ends an HTTP/2 stream. The bounds of open requests cap that memory. At the end of each request, the edge closes its body. So an HTTP/2 stream whose answer waits for the window of the client holds no body:

| Buffer of one request | Size |
| --- | --- |
| The request header | At most 64 KiB. |
| The body on HTTP/2 | At most 256 KiB, the receive buffer of the stream. |
| The answer of the control role | At most 64 KiB, the receive buffer of the stream to the control role. |
| The frame buffer of the body to the control role | 64 KiB, the largest frame that the internal listener reads ([the internal listener](/reference/internal-listener/index.html.md)). |
| The copy buffers of the proxy | 32 KiB in each direction. |

So one request holds at most 512 KiB. A connection also holds memory without a request:

| Buffer of one HTTPS connection | Size |
| --- | --- |
| The read buffer of the HTTP/2 framer | At most 16 KiB, the largest frame. |
| The write buffer of the HTTP/2 framer | About 16 KiB. |
| The TLS records | About 17 KiB in each direction. |
| The HPACK tables | 4 KiB in each direction. |
| The save buffer of the HPACK decoder | Up to about 64 KiB, for a header in CONTINUATION frames. |
| The stacks of its goroutines | 16 KiB to 32 KiB each after the TLS handshake. |
| The structs of the connection | Some KiB. |

The edge counts 256 KiB for one HTTPS connection. A connection of the HTTP listener holds a read buffer and a write buffer of 4 KiB. It also holds a stack, its structs, and the wrappers of the listeners. An idle one takes about 20 KiB. The edge counts 32 KiB for it. So the bounds are these:

| Who | Most memory |
| --- | --- |
| One client | 584 MiB: 256 HTTPS connections of 256 KiB, 256 HTTP connections of 32 KiB, and 1,024 requests of 512 KiB. |
| One IPv6 /48 | 2.3 GiB: 1,024 HTTPS connections of 256 KiB, 1,024 HTTP connections of 32 KiB, and 4,096 requests of 512 KiB. |
| The HTTPS listener | 12 GiB: 16,384 connections of 256 KiB, and 16,384 requests of 512 KiB. |
| The HTTP listener | 2 GiB: 65,536 connections of 32 KiB. |

A test of `internal/edge` checks these bounds against the defaults.

The answer of a machine and the body of a request on HTTP/1.1 wait in the socket buffers of the kernel, not in `hopboxd`. The kernel parameters `net.ipv4.tcp_rmem`, `net.ipv4.tcp_wmem`, and `net.ipv4.tcp_mem` bound them. Each HTTP/1.1 connection to a machine also holds a read buffer and a write buffer of 4 KiB in the edge.

The unit `edge` of `hopboxd run` opens the listeners at its start ([hopboxd](/reference/hopboxd/index.html.md#run)). If another program holds a port, the start fails, and `hopboxd` exits with status 1. The error names the address, such as `edge: listen on :443: bind: address already in use`. Ports 443 and 80 need root, and `hopboxd.service` runs `hopboxd` as root ([Server layout](/reference/server-layout/index.html.md#hopboxdservice)).

At a stop, the unit ends the SSH connections, closes the listeners, gives the open requests 2 s, and then ends the rest: streams, WebSockets, and long calls. The whole stop takes at most 5 s.

## TLS

The edge role serves one certificate: the wildcard certificate of the install, for the domain and `*.<domain>` ([the certificate of the install](/reference/acme/index.html.md)). The control role pushes the certificate with its key over the control channel ([the control channel](/reference/control-channel/index.html.md#edge-pushes)). The edge role keeps it only in memory.

| Rule | Value |
| --- | --- |
| TLS versions | 1.2 and 1.3. The cipher suites are the defaults of Go. |
| Server name (SNI) | Required. A handshake without a server name fails. |
| Host names | The domain, and each name under it. A handshake for another name fails. |
| Before the first certificate | Every handshake fails. |
| Certificate of another domain | When the domain of the install changes, the edge role refuses every handshake until the certificate of the new domain arrives. It never serves the certificate of the old domain. |

The edge role checks each certificate that the control role pushes before it serves it. It refuses a certificate in each of these cases, and keeps the certificate that it has:

- The domain of the message is not a domain in lowercase without a final dot.
- The leaf does not cover the domain, or a name under it.
- The leaf is not valid now.
- The key is not a PKCS #8 key, or it does not match the leaf.

A handshake that fails counts in `hopbox_edge_tls_refusals_total` ([Metrics](#metrics)).

## Routing

The edge role routes each HTTPS request by its host. The host is the `Host` header, or the `:authority` of HTTP/2, in lowercase, without a port and without a final dot.

| Request | Answer |
| --- | --- |
| The host is not the server name of the TLS connection, and one of the two is not a name of the certificate: the domain, or one label under it | `421 Misdirected Request`. A browser sends the requests of every name of a certificate on one HTTP/2 connection, so those pass. A host outside the certificate does not, also in a request in the absolute form, such as `GET https://other.example/`. |
| Method `CONNECT` | `405 Method Not Allowed`. The edge role is not a proxy. |
| Host `example.com`, path under `/hopbox.cluster.v1.` | `404 Not Found`. The cluster API is only for the servers of the install, on the internal listener. |
| Host `example.com` | Forwarded to the control role ([Forwarded requests](#forwarded-requests)). |
| Host `<name>.example.com` | The machine `<name>` ([Machine URLs](#machine-urls)). |
| Any other host under the domain, such as `sub.box1.example.com` | `404 Not Found`. Subdomain routes come in phase 4. |

Each HTTPS answer carries the header `Strict-Transport-Security: max-age=31536000; includeSubDomains` (HSTS). A browser then reaches the domain and every machine URL only over HTTPS for a year.

An HTTP/1.1 request with the body encoding `chunked` gets `Connection: close` on its answer. Such a request can also have had a `Content-Length` header, which the server drops. Two such headers can hide a second request behind the first one, so the connection ends after the answer (RFC 9112, section 6.1). The server of Go refuses the other ambiguous requests. Two different `Content-Length` values, and a space before the colon of a header name, get `400 Bad Request`. A `Transfer-Encoding` other than `chunked` gets `501 Not Implemented`. A header line that continues on the next line reaches a machine as one line. The edge role builds a new request for each request that it forwards, so a forwarded request never carries the framing of the client.

The pages of the edge role are plain text, with `Cache-Control: no-store` and `X-Content-Type-Options: nosniff`. They also carry `Content-Security-Policy: default-src 'none'; frame-ancestors 'none'` and `X-Frame-Options: DENY`, so no page shows them in a frame. So do the redirects of the sign-in, and the pages of `/auth/machine`.

### HTTP to HTTPS

The HTTP listener answers each request for the domain or a name under it with `301 Moved Permanently` to the same host and path over HTTPS. For example, `http://box1.example.com/a?b=1` goes to `https://box1.example.com/a?b=1`. A request for another host, or before the edge role knows the domain, gets `404 Not Found`.

## Forwarded requests

The edge role forwards each request of the domain to the internal listeners of the control servers ([the internal listener](/reference/internal-listener/index.html.md#forwarded-requests)). It uses the dial order of the control channel: the URLs of the live control servers that the control role sent and the server keeps, then the URLs of `channel.control_urls` of the bootstrap config that are not among them ([the control channel](/reference/control-channel/index.html.md#control-urls)). Each new list of the control role replaces the order at once. A control server that stays in the order keeps its skip. It sends the requests to the live control servers in turn, one request after the other. It tries the other URLs of `channel.control_urls` in order after them, as fallbacks. Without a list from the control role, it sends the requests to the URLs of `channel.control_urls` in turn. Its `EdgeService` calls, such as the machine URL sign-in and the checks of the SSH front door, go the same way. A request that got no connection, because a dial or the TLS handshake failed, goes to the next control server with its body unread. The edge role then skips that control server for 10 s, so the next requests try the others first. A request that reached a control server and failed before its answer came does not go again, but the edge skips that control server for 10 s too. A request whose client went away, or whose deadline ended, marks no control server and goes to no other one. A short deadline of a caller says nothing of the control server. A dead control server still gets skipped: its connection fails the ping within 10 s, and a new dial to it fails within 3 s. A new connection to a control server has 3 s for its dial and 3 s for its TLS handshake: a dial to the peer address of a control server takes a few round trips of the internet, so a slower one is gone, and the request goes on to the next control server well within the deadline of its call. A control server that died hard leaves its HTTP/2 connection half open. So the edge pings a connection to a control server that sent nothing for 5 s, and closes it when the answer does not come within 5 s. The requests on it then fail, and the next requests go to the others. This covers Connect, gRPC, gRPC-Web, streams in both directions, trailers, the sign-in pages under `/auth/`, and the WebSocket of exec (`/api/ws/exec`). The admin API is public behind the edge too, for admin API keys.

| Item | Value |
| --- | --- |
| Transport | Mutual TLS 1.3 with the server certificate of the edge server, against the install CA. HTTP/2 for every request but a WebSocket upgrade, which uses HTTP/1.1, because HTTP/2 cannot carry it. |
| Host | The host of the client request, such as `example.com`. |
| `Hopbox-Client-Addr` | The address of the client, `IP:port`. The edge role sets it over any value that the client sent. |
| `X-Forwarded-For` | The IP address of the client alone. The edge role drops the value that the client sent. |
| `X-Forwarded-Host`, `X-Forwarded-Proto` | The host of the request, and `https`. |
| `Forwarded` | Dropped. |
| Flush | Each write of the control role goes to the client at once, so streams do not wait. |
| HTTP/2 receive buffer | 64 KiB for each stream, and 1 GiB for the connection, which holds the buffers of all 16,384 open requests. So an answer that one client does not read never stalls the answers of other clients. |
| Failure | `502 Bad Gateway` with a page that says that the control plane did not answer. If the request got no connection to the control role, because of a dial or TLS error to every control server, the answer also carries `Hopbox-Not-Run: 1`. After a connection, the control role can have run the call, so the answer has no such header. |

The header `Hopbox-Not-Run: 1` tells a client that the call changed nothing, so the client can retry it ([the API server](/reference/api/index.html.md#answers-of-calls-that-changed-nothing)). The refusals of the bounds of open requests carry it too ([Listeners](#listeners)).

The control role trusts `Hopbox-Client-Addr` only on a connection whose certificate proves a server with the edge role. The rate limits of sign-in then count each client by its own address ([Sessions, cookies, and sign-in links](/reference/sessions/index.html.md#rate-limits)).

## Machine URLs

The control role pushes the routes of the machine URLs over the control channel ([the control channel](/reference/control-channel/index.html.md#edge-pushes)). A route names a machine that holds its name and is not on its way out. A running machine has a target: the address of the machine on its node, its URL port, and its lease ([Control store tables](/reference/control-store/index.html.md#route_records)).

| Case | Answer |
| --- | --- |
| No machine has the name | The same answers as a private URL without a sign-in ([Answers of a private URL](#answers-of-a-private-url)). So a client cannot tell which names have a machine. After the sign-in, `/auth/machine` answers `404 Not Found`, also for a machine that the account does not see. |
| A path under `/__hopbox/` other than `/__hopbox/auth` | `404 Not Found`. The edge role keeps these paths for itself and never forwards one to a machine. |
| A private URL without a valid sign-in | The sign-in ([Private machine URLs](#private-machine-urls)). A private URL tells nothing else about its machine before the sign-in. |
| An instance admin suspended the org of the machine | `403 Forbidden` |
| The machine does not run | `503 Service Unavailable`, with the state of the machine, such as `The machine box1 is stopped. Start it, then retry.` |
| The lease of the target ended at the edge | `503 Service Unavailable`. The node of the machine did not renew its lease lately, so the machine can run on another node now. |
| The target has no address | `503 Service Unavailable`, with the page `The machine box1 runs, but it has no network address on its node, so its URL cannot reach it.` Each machine gets an address at its create, so only a machine without one gets it ([the node network](/reference/node-network/index.html.md#addresses)). |
| The machine does not answer on its URL port | `502 Bad Gateway`, with a page that says to check that a server listens on that port in the machine |
| Otherwise | Forwarded to the machine |

The edge role counts the lease of each target from the arrival of the routes. The control role sends the rest of the lease, less a tenth of the machine TTL. It also takes off the time that the routes waited in its queue. After each heartbeat of the edge server, it sends the changed routes, or only the rest of the lease of each target. So the lease of a live target never runs out. If the control channel of the edge server ends, the targets run out within the machine TTL of 30 s. The machine URLs then answer 503 until the channel is back.

The edge role keeps each request that it forwards to a machine until it ends, such as a WebSocket or a long download. New routes end an open request at once in three cases: the machine has no route any more, its org is suspended, or its URL turned private. A request of a public URL has no sign-in, so it cannot stay open on a private URL.

### What a machine gets

The edge role forwards a request to `http://<address>:<url port>` over HTTP/1.1, without TLS. The address is the address of the machine on its node ([the node network](/reference/node-network/index.html.md)). How the request reaches that address depends on the roles of the edge server ([The way to the machines](#the-way-to-the-machines)).

| Item | Value |
| --- | --- |
| Host | The machine host, such as `box1.example.com`. |
| `X-Forwarded-For` | The IP address of the client alone. |
| `X-Forwarded-Host`, `X-Forwarded-Proto` | The machine host, and `https`. |
| `Forwarded`, `Hopbox-Client-Addr` | Dropped. Only the control role reads `Hopbox-Client-Addr`. |
| `Cookie` | Each cookie of the client, except the cookies of hopbox: every name that starts with `__Host-hopbox_`, in any case. The other cookies stay as the client sent them. |
| WebSocket | An upgrade over HTTP/1.1 passes. |
| Answer header | At most 64 KiB. |

So the code of a machine never gets the machine cookie, nor a session cookie of the console. The cookies of the console are host-only `__Host-` cookies of the domain, so a browser never sends them to a machine host anyway ([Sessions, cookies, and sign-in links](/reference/sessions/index.html.md)).

The edge role also drops these `Set-Cookie` headers of an answer of a machine:

- A cookie whose name starts with `__Host-hopbox_`, in any case.
- A cookie with a `Domain` attribute other than the machine host. Such a cookie can reach the console and the other machines of the install, which are on the same site as the machine.

A cookie without a `Domain` attribute, or with the machine host as its domain, passes.

### The way to the machines

An edge server reaches each machine at the address of its route record. The way to that address depends on the roles of the edge server:

| Edge server | The way to a machine |
| --- | --- |
| With the node role, such as the first server of an install | The routes of the host. A machine of this server is behind its tap device. A machine of another node is behind the mesh device `hbm0`, with the route of the node subnet of that node ([the mesh](/reference/mesh/index.html.md#routes)). |
| Without the node role, an edge server apart from the nodes | Only the mesh device `hbm0`. Each connection to a machine is bound to the device. So it leaves from the mesh address of the edge server, and never through another interface, such as the uplink, also while the mesh has no route to the address. The edge server refuses an address outside the machine range `10.240.0.0/12`. |

The node of the machine lets a connection from `hbm0` reach the machine only from the mesh address of an edge server. The control role names the edge servers in each mesh that it pushes, and the table `inet hopbox_node` of the node keeps them in its set `edges` ([the node network](/reference/node-network/index.html.md#what-can-reach-a-machine)). The machine sees the mesh address of the edge server as the source of each connection. The header `X-Forwarded-For` holds the address of the client.

The same rules hold for the SSH front door ([the SSH front door](/reference/ssh-front-door/index.html.md#the-mesh)).

An edge server without the node role and without a mesh device reaches no machine. Its machine URLs answer `502 Bad Gateway`, and it logs `forwarding a request to a machine failed` with an error such as `This edge server reaches the machines only over the mesh, and has no mesh device hbm0 yet`. [The mesh](/reference/mesh/index.html.md#which-servers-join) lists why a server stays out of the mesh.

## Private machine URLs

The URL of each machine is private by default. Only an account that uses the machine opens it: its creator, and every member of its org once the creator shares it. An org admin cannot open the private URL of a private machine of another member, as with a shell. The rule is the policy action `machine.open_url` ([the API server](/reference/api/index.html.md)). The creator makes the URL public with `SetMachineURLVisibility` ([the machine API](/reference/machine-api/index.html.md)). A public URL needs no sign-in.

The sign-in of a private URL takes these steps. The machine is `box1`.

1. A browser asks for `https://box1.example.com/page?x=1` without a machine cookie. The edge role answers `303 See Other` to `https://example.com/auth/machine?machine=box1&path=%2Fpage%3Fx%3D1`. A path longer than 512 characters becomes `/`.
2. The control role reads the session cookie of the console at `/auth/machine`. With a live session of an account that uses the machine, it makes a one-time code, and answers `303 See Other` to `https://box1.example.com/__hopbox/auth?code=<code>&path=%2Fpage%3Fx%3D1`. Without a session, it answers `401 Unauthorized` with a page that says to sign in to the console first.
3. The edge role takes the code at `/__hopbox/auth`, and trades it for a token with `EdgeService.ExchangeMachineCode`. It sets the token in the cookie `__Host-hopbox_machine`, and answers `303 See Other` to the path. A path that is not a plain path on the host, such as `//other.example`, becomes `/`.
4. The browser asks for `https://box1.example.com/page?x=1` with the machine cookie. The edge role checks the token with `EdgeService.AuthorizeMachineRequest`, takes the cookie out of the request, and forwards the request to the machine.

The code never reaches the machine. The answers of steps 2 and 3 carry `Cache-Control: no-store` and `Referrer-Policy: no-referrer`.

| Item | Value |
| --- | --- |
| Code | 32 random bytes in base64url, 43 characters. The control store keeps only its SHA-256. |
| Code lifetime | 60 s, one use, and only for the machine that it names. |
| Codes of one session | At most 30 in a minute. The next one gets `429 Too Many Requests`. |
| Token | 32 random bytes in base64url. The control store keeps only its SHA-256. A new token of the same browser and machine replaces the older one. |
| Token lifetime | The life of the console session that made it: 7 days without a use, and 30 days at most. A sign-out ends it at once. Each check of a token keeps its session alive, as a call of the console does. |
| Cookie | `__Host-hopbox_machine=<token>; Path=/; Secure; HttpOnly; SameSite=Lax`. The cookie is host-only, and it has no `Max-Age`. When the browser closes, it drops the cookie. |
| Kept answers | The edge role keeps each answer that allows a token on a machine. It keeps an answer for at most 1 minute, and never past the end of the console session. It keeps no refusal, so a new grant works at once. It keeps at most 10,000 answers. When it is full, it starts again empty. |

The control role tells the edge servers about each change that can end an access. A change event of a machine, such as an unshare or a delete, is about that machine. An event `edge.access_changed` is about every machine ([Control store tables](/reference/control-store/index.html.md#the-event-edgeaccess_changed)). A sign-out, a member that leaves an org or that an admin removes, a deleted account, and a suspension of an org publish it. A change can arrive while an edge server has no channel. So each new control channel also brings a change of every machine.

At a change, the edge role drops the answers that it keeps for the machines of the change. It also checks each open request of their private URLs again in the background, such as a WebSocket or a long download. A token that lost its access ends its open requests at once. The edge role also checks every open request again each minute, because a console session can end by time, without a change event. So an open request never keeps an access for more than about a minute after its end.

### Answers of a private URL

| Case | Answer |
| --- | --- |
| A top-level navigation by `GET` or `HEAD` without a valid machine cookie | `303 See Other` to `/auth/machine` (step 1) |
| Any other request without a valid machine cookie | `401 Unauthorized`. A browser does not follow a redirect for these, so the page says to open the URL in a browser first. |
| A request from a page of another origin, other than a top-level navigation by `GET` or `HEAD` | `401 Unauthorized`. It counts as a request without a machine cookie ([Requests of other sites](#requests-of-other-sites)). |
| A WebSocket upgrade whose `Origin` is not the machine host | `403 Forbidden` |
| Two different machine cookies | It counts as a request without a machine cookie. The edge role sets one machine cookie on a host, so it cannot tell which one it set. |
| A token that names no live sign-in, such as after a sign-out | The edge role clears the machine cookie with `Max-Age=0`, and starts the sign-in again. |
| The account has no access to the machine now | `403 Forbidden` |
| The org of the machine is suspended | `403 Forbidden` |
| A code that is unknown, used, expired, or for another machine | `403 Forbidden` with a page that says to open the URL again |
| The control role does not answer | `502 Bad Gateway` |

### Requests of other sites

The machine hosts of an install share one site, such as `example.com`. A browser sends a `SameSite=Lax` cookie with each request inside a site. Without a check, a page of `box1` can send a fetch, a form `POST`, or a WebSocket to `box2`. The browser then adds the machine cookie of `box2`. So the code of one machine can act in the private URL of another account. The edge role reads the fetch metadata of the browser:

- A request with `Sec-Fetch-Site: same-site` or `cross-site` counts as a request without a machine cookie. Only a top-level navigation passes: `GET` or `HEAD` with `Sec-Fetch-Mode: navigate`, and `Sec-Fetch-Dest: document` or no `Sec-Fetch-Dest`. A link from another page then still opens the URL. A frame in a page of another machine has `Sec-Fetch-Dest: iframe`, so it shows the URL without the cookie.
- A WebSocket upgrade needs no `Origin`, or the origin `https://<machine host>`. A browser always sends the origin of its page with a WebSocket.
- `/__hopbox/auth` takes only a top-level navigation by `GET`. A request with another `Sec-Fetch-Mode` gets `405 Method Not Allowed`, so a page cannot plant a cookie with a code of its own.

A request from the page of the machine itself has `Sec-Fetch-Site: same-origin`, and uses the cookie. A client without fetch metadata, such as curl, also uses it.

### Answers of `/auth/machine`

`GET /auth/machine` is a plain HTTP endpoint of the public API on the domain of the install, outside of the Connect chain. Its parameters are `machine`, the name of the machine, and `path`, the path to return to. A path that is not a plain path on the host becomes `/`.

| Case | Status |
| --- | --- |
| A code | `303 See Other` to the machine host (step 2) |
| No live console session | `401 Unauthorized` |
| An account that uses no such machine, or no such machine | `404 Not Found` |
| An account that sees the machine, but does not use it, such as an org admin on a private machine of another member | `403 Forbidden` |
| More than 30 codes of the session in a minute | `429 Too Many Requests` |
| The org of the machine is suspended, or the install has no domain | `409 Conflict` |

The answer is a plain text page with the reason and the request ID.

## The public IPv6 address

The IPv6 block of the install, a `/56` at OVHcloud, lives in the vRack, not on a server (plan question Q63). Each edge server holds one address of the block on its vRack interface. A new edge server takes the same address, so the `AAAA` records of the zone never change. The public addresses of the install list these addresses beside the floating IPv4 addresses ([the DNS role](/reference/dns/index.html.md#the-zone-of-the-install)).

The bootstrap config names the address and the gateway of the block ([the bootstrap config of hopboxd](/reference/hopboxd-config/index.html.md#keys)):

```toml
[edge]
public_ipv6 = "2001:db8:0:100::11/56"
public_ipv6_gateway = "2001:db8:0:100::1"
```

The vRack interface is the interface that holds the one address of the server in the vRack range `172.31.0.0/16`. The search skips a bridge, a port of a bridge or of a VRF, an Open vSwitch device, and a WireGuard device, so a Docker network never counts. The uplink of the server does not route the block. So the replies from the address must leave through the vRack too. `hopboxd` keeps these parts:

| Part | Value |
| --- | --- |
| Address | The address of `edge.public_ipv6`, with the prefix length of the block, on the vRack interface. The kernel runs duplicate address detection on it, so it takes traffic up to about 2 s after `hopboxd` adds it. Duplicate address detection must be on: the sysctl `net.ipv6.conf.<device>.accept_dad` is 1, the default of Linux. Without it, the new server of a replaced edge takes the address while the old one still holds it, and a check logs an error. |
| Routing table | `hopbox_egress6`, the table number 26806. It holds two default routes, both with the protocol 106. The first one goes via `edge.public_ipv6_gateway` on the vRack interface, with the flag `onlink` and the metric 1024. The second one is `unreachable`, with the metric 4294967295. So while the route of the gateway is gone, a packet of the table gets an error, and never takes the uplink through the main table. |
| Rule of the block | `from <block> lookup 26806`, with the priority 26806, before the main table at 32766. Each packet with a source in the block takes the table. The other packets of the server keep the main table and the uplink. |
| Rule of the machines | `from fd65:1c74:e769::/48 lookup 26806`, with the priority 26807 and the protocol 106. The IPv6 egress of the machines takes the table. It comes from the egress tunnel `hbe0`, or from the tap devices of an edge server with the node role ([the egress of the machines](/reference/egress/index.html.md#the-source-addresses)). The host firewall then gives it the address of `edge.public_ipv6`. |

```console
$ ip -6 rule show table 26806
26806:	from 2001:db8:0:100::/56 lookup 26806 proto 106
26807:	from fd65:1c74:e769::/48 lookup 26806 proto 106
$ ip -6 route show table 26806
default via 2001:db8:0:100::1 dev enp3s0f1 proto 106 metric 1024 onlink pref medium
unreachable default dev lo proto 106 metric 4294967295 pref medium
```

The unit `channel` of `hopboxd run` checks the parts at its start and then every 2 s ([hopboxd](/reference/hopboxd/index.html.md#run)). Every server that enrolls runs this check, also a server without the edge role. A check sets again each part that changed outside `hopboxd`, such as after `ip -6 rule flush`. A stop of `hopboxd` leaves the parts, so the traffic of the address goes on while `hopboxd` restarts.

| Case | What a check does |
| --- | --- |
| The server has no vRack address, or more than one | It sets nothing, and logs it at level error. The next check tries again. |
| Another host on the vRack holds the address | Duplicate address detection fails, and the address takes no traffic. The check removes the address and adds it again. So the address works again once the other host lets it go, such as the old server after the replacement of an edge server. The check logs the conflict once. |
| The vRack interface holds another address of the block, or an address of the block of an older rule of the table | It removes that address. |
| The table holds another route, or another rule with a source looks up the table | It removes that route or that rule. A rule without a source stays. A route or a rule of the parts above that is gone comes back. |
| The bootstrap config names no address | It removes the addresses of the blocks of the rules of the table from the vRack interface. Then it removes each route of the table, and then each rule with a source, also the rule of the machines. The rules go last, because they name the blocks. So a check that fails on the way keeps them, and the next check tries again. A server that stops being an edge server, or whose config drops `edge.public_ipv6`, lets the address go. |

The [host firewall](/reference/host-firewall/index.html.md#the-address-of-the-56) opens the public ports of the edge server on this address, from every source, and no other port. It also drops each packet of the block that leaves through an interface other than the vRack interface, the egress tunnels, and the tap devices. The IPv6 egress of the machines leaves through the same table `hopbox_egress6`, with the address of `edge.public_ipv6` as its source. Without that key, the edge server refuses the IPv6 egress of the machines ([the egress of the machines](/reference/egress/index.html.md#ipv6-without-an-address-of-the-56)).

## The egress of the machines

Each edge server that enrolls takes the egress of the machines of the nodes through its egress tunnel `hbe0` ([the egress of the machines](/reference/egress/index.html.md)). Its host firewall forwards that egress to the uplink for IPv4, and to the vRack interface for IPv6 ([the host firewall](/reference/host-firewall/index.html.md#the-egress-of-the-machines)). The IPv4 egress gets the source `edge.public_ipv4`, the floating address of the edge server ([the bootstrap config of hopboxd](/reference/hopboxd-config/index.html.md#keys)). The IPv6 egress gets the address of `edge.public_ipv6`. An edge server with the node role is the local exit of its own machines ([the egress of the machines](/reference/egress/index.html.md#the-local-exit)).

## EdgeService

`hopbox.cluster.v1.EdgeService` serves the edge servers on the internal listener ([the internal listener](/reference/internal-listener/index.html.md)). Every call needs the certificate of a server with the edge role. Another server gets `permission_denied`.

| Call | Request | Response | Errors |
| --- | --- | --- | --- |
| `ExchangeMachineCode` | `machine`: the name of the machine of the host. `code`: the code of the query. | `token`: the value of the machine cookie. | `unauthenticated` for a code that is unknown, used, expired, malformed, or for another machine. `invalid_argument` for a name that breaks the name rules. |
| `AuthorizeMachineRequest` | `machine`: the name of the machine of the host. `token`: the value of the machine cookie. | `machine_id` and `account_id`, UUIDs. `valid_for`: how long the edge can keep the answer, at most 1 minute. | `unauthenticated` for a token that is unknown, malformed, for another machine, or of an ended console session. `permission_denied` for an account without access. `failed_precondition` for a suspended org. |
| `AuthorizeSSH` | `ssh_port`, `public_key`, `linux_user`, and `client_ip` of an SSH connection, and `recheck` for a check of an open connection ([the SSH front door](/reference/ssh-front-door/index.html.md#edgeserviceauthorizessh)). Each allowed sign-in writes the audit entry `machine.ssh`. | `machine_id`, `machine_name`, `account_id`, and `valid_for`. | `permission_denied` for every key, port, or user that opens nothing, with one message. `invalid_argument` for a sign-in without a valid `client_ip`, with the message `invalid argument: a sign-in needs the IP address of the SSH client` or `invalid argument: the client IP is not an IP address`. |

The edge role calls each one with a timeout of 10 s.

## Metrics

The edge role serves these metrics on the metrics listener of `hopboxd` ([Metrics](/reference/metrics/index.html.md)).

| Metric | Type | Labels | Meaning |
| --- | --- | --- | --- |
| `hopbox_edge_requests_total` | counter | `to`: `api`, `machine`, `redirect`, or `refused` | The HTTP requests of the edge role, by where they went. A redirect goes to HTTPS, to a sign-in, or back from a sign-in. A refusal got a page of the edge role, also a refusal past the bounds of open requests. |
| `hopbox_edge_tls_refusals_total` | counter | `reason`: `no_sni`, `unknown_host`, or `no_certificate` | The TLS handshakes that the edge role refused. |
| `hopbox_edge_client_connection_refusals_total` | counter | `listener`: `https` or `http`. `bound`: `client` or `network` | The connections that the edge role closed at once. `client`: their IPv4 address or IPv6 /64 held 256 open connections on the listener. `network`: their IPv6 /48 held 1,024. |
| `hopbox_edge_open_connections` | gauge | `listener`: `https` or `http` | The connections that the listener holds now. |
| `hopbox_edge_connection_limit` | gauge | `listener`: `https` or `http` | The most connections that the listener holds at once: 16,384 for `https`, and 65,536 for `http`. A connection past it waits in the backlog of the kernel. |
| `hopbox_edge_routes` | gauge | none | The machines that the routes name. |
| `hopbox_edge_certificate_not_after_seconds` | gauge | none | The end of the certificate that the edge role serves, in seconds since the Unix epoch, or 0 before the first certificate. |

The metrics of the SSH front door start with `hopbox_edge_ssh_` ([the SSH front door](/reference/ssh-front-door/index.html.md#metrics)).

## Logs

The edge role logs with `component=edge`.

| Message | Level | When |
| --- | --- | --- |
| `edge listening` | info | The listeners are open, with `https` and `http`. |
| `serving a certificate` | info | A new certificate, with `domain`, `not_after`, and `serial`. |
| `serving the machine URLs of a domain` | info | The first routes, or the routes of a new domain, with `domain` and `machines`. |
| `forwarding a request to the control plane failed` | warn | A forwarded request failed, with `path` and `err`. |
| `forwarding a request to a machine failed` | info | A machine did not answer, with `host` and `err`. |
| `checking a machine URL sign-in failed` | warn | `AuthorizeMachineRequest` failed without an answer, with `machine` and `err`. |
| `trading a machine URL code failed` | warn | `ExchangeMachineCode` failed without an answer, with `machine` and `err`. |
| `ended the open requests of a machine URL sign-in that lost its access` | info | A check refused a token with open requests. The fields are `machine`, `open_requests`, and `code`. |
| `ended the open requests of machine URLs that changed` | info | New routes ended open requests, with `open_requests` ([Machine URLs](#machine-urls)). |

The public IPv6 address logs with `component=egress`:

| Message | Level | When |
| --- | --- | --- |
| `this edge server holds its public IPv6 address on the vRack` | info | A check found every part in place, and the address passed duplicate address detection, for the first time or after a change. The fields are `address`, `gateway`, `device`, and `table`. |
| `the vRack interface runs no duplicate address detection, so this edge server can take the public IPv6 address that another one holds. Set the sysctl net.ipv6.conf.<device>.accept_dad to 1` | error | `accept_dad` is 0 for the vRack interface and for `all`, with `device`. A check logs it again only after a change. |
| `the public IPv6 address of this edge server lost a part outside hopboxd, so hopboxd set it again` | warn | A check set again what changed outside `hopboxd`, with `device` and `parts`: `address`, `route`, or `rule`. |
| `another host on the vRack holds the public IPv6 address of this edge server, so it takes no traffic. hopboxd tries the address again at each check` | error | Duplicate address detection failed, with `address` and `device`. |
| `this edge server has no address on the vRack, so it holds no public IPv6 address. The next check tries again` | error | The server has no address in `172.31.0.0/16`, with `address`. |
| `cannot find the vRack interface, so this edge server holds no public IPv6 address. The next check tries again` | error | The server has more than one address in `172.31.0.0/16`, or the read of its links failed, with `err`. |
| `cannot set the public IPv6 address of this edge server. The next check tries again` | error | A change of the kernel failed, with `address`, `device`, and `err`. |
| `removed the public IPv6 address of this edge server, because the bootstrap config names none` | info | A check removed the parts of an older config, with `removed`. |
| `cannot remove the old public IPv6 address of this edge server. The next check tries again` | error | That removal failed, with `err`. |

A check logs a state only after a change of the state, so a check every 2 s does not repeat a line.

The edge role can refuse a certificate or routes that the control role pushed. The agent of the control channel then logs `the edge role refused a push of the control plane, and keeps what it has` at error level, with `push` and `err`.

## Not in phase 1

These parts of the edge role come later:

- A machine in standby or hibernated wakes at the first request (phase 3). Until then, its URL answers 503.
- Subdomain routes, such as `sub.box1.example.com` (phase 4).
- Custom domains and org domains (phase 4).
- Raw TCP and UDP ports (phase 4).
