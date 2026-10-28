# The internal listener

> The TLS internal listener of the control role, its certificate, its client certificates, and the server caller.

The internal listener is the TLS listener of the control role for the servers of the install. New servers enroll on it ([Enrollment and the install PKI](/reference/enrollment/index.html.md)), and enrolled servers keep their control channel on it ([the control channel](/reference/control-channel/index.html.md)). The edge servers call `EdgeService` on it, and send it the requests of the clients of the API and the console ([the edge role](/reference/edge/index.html.md#forwarded-requests)). The code is in `internal/control` and `internal/api`.

## Listener

| Item | Value |
| --- | --- |
| Address | `control.internal_listen` of the bootstrap config, by default `127.0.0.1:7443` ([the bootstrap config of hopboxd](/reference/hopboxd-config/index.html.md#keys)) |
| Address rules | An IP address and a port. A host name, even `localhost`, is refused. Any IP address works, also one that is not loopback. Port `0` picks a free port. |
| Transport | TLS 1.3 only |
| HTTP versions | HTTP/2 and HTTP/1.1, picked with ALPN |
| HTTP/2 frames | At most 64 KiB for each frame that a peer sends. A client of HTTP/2 holds a buffer of that size for each request body that it streams. So the edge holds 64 KiB for each request that it forwards, not 512 KiB. |
| Protocols | Connect, gRPC, and gRPC-Web, with the binary protobuf and JSON codecs. `ControlService` takes no gzip: a request in gzip fails with `unimplemented` and the message `unknown compression "gzip"`, and no response uses gzip ([the control channel](/reference/control-channel/index.html.md#limits-and-windows-of-the-data-streams)). The other services of the listener take gzip and answer in it. |
| Services | `hopbox.cluster.v1.EnrollmentService`, `hopbox.cluster.v1.ControlService`, `hopbox.cluster.v1.ControlPeerService`, `hopbox.cluster.v1.EdgeService`, gRPC reflection, and gRPC health. A forwarded request reaches the public API instead. See [Forwarded requests](#forwarded-requests). |
| Server certificate | the control certificate of the install CA, with the URI SAN `hopbox://control` ([Enrollment and the install PKI](/reference/enrollment/index.html.md#control-certificate)) |
| Client certificate | asked for in every handshake. A client can send none. A certificate that it sends must chain to the install CA and allow client auth, or the handshake fails. A server certificate and the control certificate of another control process both pass ([Callers](#callers)). |
| Unit | `internal listener`, after `api` and before `admin socket` ([hopboxd](/reference/hopboxd/index.html.md#run)) |
| Stop timeout | 5 s. A stop ends every open stream at once, so every control channel ends ([the API server](/reference/api/index.html.md#when-a-listener-stops)). |
| Metrics | the label `listener="internal"` for the cluster API, and `listener="edge"` for the forwarded requests ([Metrics](/reference/metrics/index.html.md#metrics-of-the-api)) |

On an install with one server, the listener stays on `127.0.0.1`, and the `channel` unit of the same server dials it on loopback. With more servers, it listens on the peer address of the control server, such as `203.0.113.11:7443`, so the other servers can reach it. The host firewall takes its port on the public interface only from the peer addresses of the servers, the internal listeners of the live control processes, and the registered addresses of the enrollment tokens that work ([the host firewall](/reference/host-firewall/index.html.md#sets)). Each other server then lists the listener of one or more control servers in `channel.control_urls` ([the bootstrap config of hopboxd](/reference/hopboxd-config/index.html.md#keys)). The control role sends each server the listeners on the peer addresses of the live control processes, and the server dials them first ([the control channel](/reference/control-channel/index.html.md#control-urls)). A listener on loopback or on a private address is not in that list.

Each control process records the URL of its listener in its row of `control_processes` (`internal_url`, [Control store tables](/reference/control-store/index.html.md#control_processes)), as `https://` and the address of the listener. The other control processes forward data streams to that URL ([the control channel](/reference/control-channel/index.html.md#streams-through-another-control-process)). A listener on every address, such as `0.0.0.0:7443`, names no address, so its row has no URL, and no data stream reaches its channels from another control process. With several control servers, set `control.internal_listen` to the peer address of the server and port 7443. `hopboxd install --internal-listen` sets it ([hopboxd](/reference/hopboxd/index.html.md#install)).

## Move the first server to its peer address

The first server of an install has every role, and its listener stays on loopback while it is the only server. Before a second server enrolls, the listener moves to the peer address of the first server. The Terraform provider runs this step at an update of the first server ([hopbox Terraform provider](/reference/terraform-provider/index.html.md#the-internal-listener)). To run it by hand, take these steps on the first server.

1. Make sure that the server has its peer address. The line `peer_address` of `sudo hopboxd admin server get <id>` names it. If it is empty, set it with `sudo hopboxd admin server set-peer-address <id> <address>` ([hopboxd](/reference/hopboxd/index.html.md#admin-server-set-peer-address)). The install refuses a listener on another address than the stored peer address.
2. Run the install with the peer address and port 7443:

   ```sh
   sudo hopboxd install --roles control,node,dns,edge --internal-listen 203.0.113.11:7443
   ```

The install writes `control.internal_listen = "203.0.113.11:7443"`. It writes `https://203.0.113.11:7443` in the place of each loopback URL of `channel.control_urls`, or of the default `https://127.0.0.1:7443`. Then it restarts `hopboxd`, and the roles of the server dial the listener at the new URL. No loopback URL stays: the listener has one address. The machines keep running through the restart.

After the restart, these things hold:

- The control role publishes `https://203.0.113.11:7443`, so each server keeps that URL in `<data_dir>/server/control-urls` ([the control channel](/reference/control-channel/index.html.md#control-urls)).
- The roles of the first server dial the listener from the peer address, not from loopback. The enrollment takes a call from the address of the control server itself as a call from loopback, so a token without a peer address still adds a role to the first server ([Enrollment and the install PKI](/reference/enrollment/index.html.md#peer-addresses)).
- The host firewall takes port 7443 of the public interface from the set `peers` and the set `registered` alone. A connection of the server to its own address goes over loopback, so the firewall takes it.

Then make the token of the second server with its peer address, and enroll the server at `https://203.0.113.11:7443` ([Enrollment and the install PKI](/reference/enrollment/index.html.md#peer-addresses)).

The HTTP server has the timeouts and limits of the API listener ([the API server](/reference/api/index.html.md#listener)): the header read timeout, the body read timeout of a unary call, the idle timeout, and the largest request message of 4 MiB.

In maintenance mode, the listener refuses every call of the cluster API with `unavailable`, and takes only health and reflection. No method of `hopbox.cluster.v1` has the option `(hopbox.v1.read_only)`. The forwarded requests of the edge servers get the refusals of the public API ([the API server](/reference/api/index.html.md#maintenance-mode)).

## Before the first key-encryption key

The certificate of the listener comes from the install CA, and the CA needs a key-encryption key to seal its key ([Enrollment and the install PKI](/reference/enrollment/index.html.md#install-ca)). A fresh install has no such key. So the listener starts without a certificate, and logs this warning:

```json
{"time":"2026-10-16T09:00:00Z","level":"WARN","msg":"the internal listener has no certificate yet","err":"failed precondition: the keyring has no key-encryption key to seal or open the value. Run hopboxd admin kek rotate to make the first one"}
```

Until then, each TLS handshake fails, and the HTTP server logs `http: TLS handshake error from <peer>: the internal listener has no certificate: ...` with `component=http`. `hopboxd install` makes the first key-encryption key, so a server that it set up never logs this. On another server, run `sudo hopboxd admin kek rotate` ([hopboxd](/reference/hopboxd/index.html.md#admin-kek-rotate)). The next handshake then makes the CA and the certificate, without a restart.

Any other failure to read the CA stops the start of the unit.

## Callers

The request gate of the listener has one resolver, the server-certificate resolver ([the API server](/reference/api/index.html.md#request-gate)):

| Case | Caller |
| --- | --- |
| The connection has no client certificate. | `anonymous` |
| The certificate names `hopbox://control`, and so is the control certificate of another control process. | `control`, with no account and no credential ID |
| The certificate passes the checks of [Enrollment and the install PKI](/reference/enrollment/index.html.md#client-certificates-on-each-call). | `server`, with the ID of the server |
| The certificate fails those checks, for example its serial is unknown or its server is decommissioned. | none. The call ends with `unauthenticated` and the message `the credential is not valid`. |

The TLS handshake already checked that the install CA signed a certificate that names `hopbox://control`. The `control` caller has no audit credential and no idempotency scope. The policy lets it do only `control.forward_stream`, besides the actions that every caller may do (`GetInstallInfo`, health, and reflection).

Every procedure of the listener needs a server certificate or a control certificate, except `EnrollmentService.Enroll`, whose enrollment token is its credential. An anonymous call of another procedure ends with `unauthenticated` and the message `this call needs a server certificate`. This rule runs after the gate and before the policy check. So it covers health and reflection too, which the policy allows for every caller on the other listeners.

| Procedure | Without a certificate | With a server certificate |
| --- | --- | --- |
| `EnrollmentService.Enroll` | allowed | allowed |
| `ControlService.Channel`, `ControlService.AttachStream` | `unauthenticated` | allowed for the server itself ([the control channel](/reference/control-channel/index.html.md#controlservice)) |
| `ControlPeerService.ForwardStream` | `unauthenticated` | `permission_denied`. Only the control certificate of another control process is allowed. |
| `EdgeService.ExchangeMachineCode`, `EdgeService.AuthorizeMachineRequest`, `EdgeService.AuthorizeSSH` | `unauthenticated` | allowed for a server whose certificate carries the role `edge`. Another server gets `permission_denied`. |
| `grpc.health.v1.Health`, reflection | `unauthenticated` | allowed |

## ControlPeerService

`hopbox.cluster.v1.ControlPeerService` lets a control process forward a data stream to the control process that holds the channel of a server ([the control channel](/reference/control-channel/index.html.md#streams-through-another-control-process)). It takes no gzip, as `ControlService` does not. Each call needs the control certificate of another control process, which is also a client certificate: the control certificate has the extended key usage client auth ([Enrollment and the install PKI](/reference/enrollment/index.html.md#control-certificate)).

| Procedure | Policy action | Description |
| --- | --- | --- |
| `/hopbox.cluster.v1.ControlPeerService/ForwardStream` | `control.forward_stream` | Opens a data stream of a kind to a server whose channel this process holds, and carries its bytes both ways. The first request names the server and the kind. The first response has no bytes and tells that the server attached the stream. |

The messages and the errors are in [the control channel](/reference/control-channel/index.html.md#controlpeerservice).

## EdgeService

`hopbox.cluster.v1.EdgeService` serves the sign-in of private machine URLs ([the edge role](/reference/edge/index.html.md#private-machine-urls)) and of the SSH front door ([the SSH front door](/reference/ssh-front-door/index.html.md)) to the edge servers. Each call needs the certificate of a server with the edge role.

| Procedure | Policy action | Description |
| --- | --- | --- |
| `/hopbox.cluster.v1.EdgeService/ExchangeMachineCode` | `server.exchange_machine_url_code` | Trades the one-time code of a private machine URL for the token of the cookie `__Host-hopbox_machine` of that URL. The code works once, within 60 s, and only for its machine. The new token replaces any older token of the same console session and machine. |
| `/hopbox.cluster.v1.EdgeService/AuthorizeSSH` | `server.authorize_ssh` | Checks that the account of an SSH key can open a shell, as a Linux user, in the machine of an SSH port now. Each answer that allows also records a use of the machine, and each allowed sign-in writes the audit entry `machine.ssh`. The request, the response, and the rules are in [the SSH front door](/reference/ssh-front-door/index.html.md#edgeserviceauthorizessh). |
| `/hopbox.cluster.v1.EdgeService/AuthorizeMachineRequest` | `server.authorize_machine_url` | Checks that the account of a token can open the private URL of the machine now. Each answer that allows also records a use of the console session, as a call of the console does ([Sessions, cookies, and sign-in links](/reference/sessions/index.html.md)). |

| Message | Field | Type | Description |
| --- | --- | --- | --- |
| `ExchangeMachineCodeRequest` | `machine` | string | The name of the machine of the host of the request, such as `box1` for `box1.example.com`. |
| `ExchangeMachineCodeRequest` | `code` | string | The code of the query of the request. |
| `ExchangeMachineCodeResponse` | `token` | string | The value of the cookie `__Host-hopbox_machine`, valid only for that machine. |
| `AuthorizeMachineRequestRequest` | `machine` | string | The name of the machine of the host of the request. |
| `AuthorizeMachineRequestRequest` | `token` | string | The value of the cookie `__Host-hopbox_machine`. |
| `AuthorizeMachineRequestResponse` | `machine_id` | string | The ID of the machine, a UUID. |
| `AuthorizeMachineRequestResponse` | `account_id` | string | The account of the token, a UUID. |
| `AuthorizeMachineRequestResponse` | `valid_for` | `google.protobuf.Duration` | How long the edge can keep the answer: at most 1 minute, and never past the end of the console session. A change event of the access ends it sooner ([the control channel](/reference/control-channel/index.html.md#edge-pushes)). |

`AuthorizeMachineRequest` checks four rules. Each must hold:

- The token names a row of `machine_url_sessions`, and the console session of that row lives: its absolute end and its idle end are ahead ([Control store tables](/reference/control-store/index.html.md#machine_url_sessions), [Sessions, cookies, and sign-in links](/reference/sessions/index.html.md)).
- The machine of the request holds its name, its target is not `deleted`, and it is the machine of the token.
- The account uses the machine, under the policy action `machine.open_url`: it is the creator of a private machine, or a member of the org of a shared one ([the API server](/reference/api/index.html.md#callers-and-policy)). An org admin does not open the private machine of another member.
- An instance admin did not suspend the org of the machine.

| Code | Message | Cause |
| --- | --- | --- |
| `unauthenticated` | `the sign-in of the machine URL is not valid: ...` | The code or the token is malformed, unknown, used, expired, replaced, or for another machine, or its console session ended. The machine can also be gone. The edge starts the sign-in again. |
| `permission_denied` | `permission denied: ...` | The account has no access to the machine now. Another server than an edge server gets it too. |
| `failed_precondition` | `failed precondition: org suspended: an instance admin suspended the org of the machine` | The org of the machine is suspended. |
| `invalid_argument` | `invalid argument: ...` | The machine name is not a valid name ([Domain rules](/reference/domain-rules/index.html.md)). |

## Forwarded requests

The edge servers forward the requests of the domain of the install to this listener, over the mutual TLS of their server certificates ([the edge role](/reference/edge/index.html.md#forwarded-requests)). Each forwarded request carries the header `Hopbox-Client-Addr` with the address of the client, `IP:port`. The edge sets the header on each request, over any value that the client sent.

The listener splits the requests by this header:

- A request without the header reaches the cluster API, as in [Callers](#callers).
- A request with the header reaches the public API and the admin API, as on the API listener ([the API server](/reference/api/index.html.md)). The plain HTTP routes under `/auth/` and the WebSocket of exec are there too. So a forwarded request never reaches the cluster API with the certificate of the edge server.

Before it serves a forwarded request, the listener checks the certificate of its connection. The certificate must pass the checks of [Enrollment and the install PKI](/reference/enrollment/index.html.md#client-certificates-on-each-call). It must also carry the role `edge`, under the policy action `server.forward_requests`. The listener keeps a passed check for 5 s, by the serial of the certificate. So each request does not cost a read of the control store. It keeps at most 1024 checks. So a decommission or a retired certificate stops the forwards within 5 s. Then the address of the header becomes the peer of the request, and the header goes. The request gate, the rate limits, and the logs see the address of the client. The credentials of the request, such as the session cookie or an API key, prove its caller. The certificate of the edge server proves no caller. Only this check trusts a client address: no listener reads `X-Forwarded-For`.

| Status | Body | Cause |
| --- | --- | --- |
| 403 | `only a server with the edge role forwards the requests of clients` | The connection has no TLS client certificate, or the certificate is refused, or its server has no edge role. The listener logs `forwarded request refused` at level warn, with `peer` and `err`. |
| 503 | `the server failed to check the credential. Retry later` | The check of the certificate failed, for example on a failure of the control store. The listener logs `forwarded request check failed` at level error, with `peer` and `err`. |
| 400 | `the header Hopbox-Client-Addr must hold one IP:port` | The header has more than one value, or a value that is not an IP address and a port. |

The forwarded requests get the label `listener="edge"` in the metrics of the API. Each forwarded call also gets the access log record `api call` ([the API server](/reference/api/index.html.md#logs)).

## Examples

Root on an enrolled server can call the listener with the identity of the server. `identity.pem` holds both the certificate and the key, and `ca.pem` holds the install CA ([Enrollment and the install PKI](/reference/enrollment/index.html.md#identity-files)). The control certificate names the IP address of the listener, so the usual host check of curl and grpcurl passes for that address.

A health check with curl and the Connect protocol:

```sh
cd /var/lib/hopbox/server
sudo curl -s --cacert ca.pem --cert identity.pem --key identity.pem \
  -H 'Content-Type: application/json' -d '{}' \
  https://127.0.0.1:7443/grpc.health.v1.Health/Check
```

```json
{"status":"SERVING"}
```

The same call without the certificate:

```sh
sudo curl -s --cacert /var/lib/hopbox/server/ca.pem \
  -H 'Content-Type: application/json' -d '{}' \
  https://127.0.0.1:7443/grpc.health.v1.Health/Check
```

```json
{"code":"unauthenticated","message":"this call needs a server certificate"}
```

The services of the listener, with grpcurl:

```sh
cd /var/lib/hopbox/server
sudo grpcurl -cacert ca.pem -cert identity.pem -key identity.pem 127.0.0.1:7443 list
```

```text
hopbox.cluster.v1.ControlService
hopbox.cluster.v1.EdgeService
hopbox.cluster.v1.EnrollmentService
```

Reflection lists only the services of the cluster API, as on the other listeners.

A control server that is not enrolled has no identity files. Use `ca.pem` of an enrolled server, or `--insecure` with curl only to see the error of a call without a certificate.
