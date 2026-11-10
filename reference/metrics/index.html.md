# Metrics

> The Prometheus metrics listener of hopboxd and every metric that it serves.

`hopboxd` serves Prometheus metrics on a loopback listener. The code is in `internal/obs`, `internal/daemon`, `internal/api`, `internal/audit`, `internal/service`, `internal/control`, `internal/control/channel`, `internal/acme`, `internal/store`, `internal/placement`, `internal/node`, `internal/node/executor`, `internal/node/machines`, `internal/node/vmm/firecracker`, `internal/node/netcfg`, `internal/mesh`, and `internal/dns`.

The counter `hopbox_account_creations_total` came with milestone A2, the approval of new accounts (metrics catalog).

## Listener

| Item | Value |
| --- | --- |
| Address | `metrics.listen` of the bootstrap config, by default `127.0.0.1:9464` ([the bootstrap config of hopboxd](/reference/hopboxd-config/index.html.md#keys)). It must be a loopback address. |
| Path | `/metrics`. On a server with the control role, also `/alerts`, the [alert webhook](#alert-webhook). Every other path gets HTTP status 404. |
| Method | `GET` for `/metrics`, and `POST` for `/alerts` |
| Format | the Prometheus text format. When the scraper asks for the OpenMetrics format, the listener uses it. |
| Auth | none for `/metrics`. Only a process on the server can reach a loopback address. `/alerts` needs the alert token of the install. |

The metrics listener is the first unit that `hopboxd run` starts, and the last one that it stops ([hopboxd](/reference/hopboxd/index.html.md#run)). So the metrics cover the start and the stop of every other unit.

Example:

```sh
curl -s http://127.0.0.1:9464/metrics | grep '^hopbox_api_calls_total'
```

```text
hopbox_api_calls_total{code="ok",listener="api",procedure="/hopbox.v1.InstallService/GetInstallInfo"} 3
hopbox_api_calls_total{code="unauthenticated",listener="api",procedure="/hopbox.admin.v1.InstallConfigService/GetInstallConfig"} 1
```

## Alert webhook

Alertmanager of the monitoring bundle sends each alert to `POST /alerts` on the metrics listener of the control server ([Monitor an install with Prometheus and Grafana](/guide/monitor-an-install/index.html.md)). `hopboxd` turns each alert into a notice to the instance admins, and emails them through the mail of the install ([the admin API](/reference/admin-api/index.html.md#notices-of-alerts)). The owner decided this path (Q143 to Q145, Q148, and Q149). The path listens on loopback only, as the rest of the listener does, so nothing opens to the internet.

| Item | Value |
| --- | --- |
| Address | the metrics listener of a server with the control role, by default `http://127.0.0.1:9464/alerts`. A server without the control role answers 404. |
| Method | `POST` |
| Auth | the header `Authorization: Bearer hbxam_...` with the alert token of the install. `hopboxd` compares the SHA-256 of the token with the hash in the control store, in constant time. The token never goes to a log. |
| Body | the JSON of the Alertmanager webhook, version `4`, at most 1 MiB, with at most 1,000 alerts. `hopboxd` reads `version`, and the fields `status`, `labels`, `annotations`, `startsAt`, `endsAt`, and `fingerprint` of each alert. It keeps at most 64 labels and 64 annotations of an alert, without the names longer than 128 bytes, and cuts each value to 2,000 bytes. |
| Maintenance mode | each call gets 503, and Alertmanager tries again later. |

`sudo hopboxd admin config alert-token set` makes the token, keeps its SHA-256 in the control store, and writes the token to the file that Alertmanager reads ([hopboxd](/reference/hopboxd/index.html.md#admin-config-alert-token-set)). Each control server of the install takes the same token.

A call that the webhook refuses changes nothing. A call with a bad alert changes nothing for the other alerts of the call either. The answers:

| HTTP status | Body | Cause |
| --- | --- | --- |
| 200 | empty | The webhook stored each alert of the call. The email of its new alerts goes out after the answer, in the background. |
| 400 | `the body is not the JSON of the Alertmanager webhook: <error>` | The body is no valid JSON. |
| 400 | `the body has the version <version>, and hopboxd reads version 4 of the Alertmanager webhook` | Another version of the webhook body. |
| 400 | `invalid argument: ...`, such as `invalid argument: the fingerprint of an alert must be 16 lowercase hex characters, not "x"` | An alert has a status other than `firing` or `resolved`, a bad fingerprint, no label `alertname`, or no start time, or the call holds more than 1,000 alerts. |
| 400 | `read the body: <error>` | The client closed the connection during the body. |
| 401 | `the call needs the alert token of the install` | The header has no token, another token, or a malformed one, or the install has no token. The answer has the header `WWW-Authenticate: Bearer realm="hopbox alerts"`. |
| 413 | `the body is larger than 1 MiB` | The body is over the limit. |
| 500 | `hopboxd failed to store the alerts. Retry later` | A write to the control store failed. |
| 503 | `the control store is not open yet. Retry later` | The control role starts or stops. The metrics listener starts before the control store, and stops after it. |
| 503 | `the check of the token failed. Retry later` | The read of the hash of the token from the control store failed. |
| 503 | `the install is in maintenance mode: this call writes to the control store, so it waits for the end of maintenance mode` | The install is in maintenance mode. |
| 503 | `unavailable: hopboxd stops, so it takes no more alerts. Retry later` | The control role stops. It waits for the calls that run and their emails, and refuses the new ones. |

Alertmanager tries a call again after a status of 500 and above, and after a network error. A status of 400 to 499 ends its tries for that call. The alert `HopboxAlertWebhookFailing` fires when the webhook answers with an error ([Monitor an install with Prometheus and Grafana](/guide/monitor-an-install/index.html.md#hopboxalertwebhookfailing)).

The webhook counts each call, also each refused one:

| Metric | Type | Labels | Description |
| --- | --- | --- | --- |
| `hopbox_alert_webhook_requests_total` | counter | `code` | The calls of the alert webhook by HTTP status: `200`, `400`, `401`, `413`, `500`, or `503`. Each code starts at 0 on a server with the control role, so the first error shows as an increase. The alert `HopboxAlertWebhookFailing` reads the codes other than `2xx` (Q149). |

The webhook writes these log records, with `component=alerts`, apart from the last one:

| Message | Level | Fields |
| --- | --- | --- |
| `refused a call of the alert webhook` | warn | `remote`, the address of the client, and `reason`, such as `the alert webhook needs the token of the install: the token is missing or malformed` |
| `the check of the token of the alert webhook failed` | error | `err` |
| `the alert webhook failed` | error | `alerts`, the count of the alerts of the call, and `err` |
| `received alerts from Alertmanager` | info | `component=service`, `alerts`, `fired`, `repeated`, and `resolved`, the counts of the call. The alert `Watchdog` counts in `alerts` only. |

The email of the new alerts of a call writes the records of the emails of a notice ([the admin API](/reference/admin-api/index.html.md#notices-of-alerts)).

## Metrics of the API

At its end, each API call adds to these metrics ([the API server](/reference/api/index.html.md#the-chain-of-a-call)). A call that the gate or the policy refuses counts too.

| Metric | Type | Labels | Description |
| --- | --- | --- | --- |
| `hopbox_api_calls_total` | counter | `listener`, `procedure`, `code` | The API calls that ended. |
| `hopbox_api_call_duration_seconds` | histogram | `listener`, `procedure`, `code` | The duration of the API calls that ended. A stream counts from its start to its end. |
| `hopbox_api_in_flight` | gauge | `listener` | The API calls that run now. A stream counts from its start to its end, such as each control channel of a server on the `internal` listener. Each listener starts at 0. |

| Label | Values |
| --- | --- |
| `listener` | `api`: the API listener. `internal`: the cluster API of the internal listener ([the internal listener](/reference/internal-listener/index.html.md)). `edge`: the requests that the edge servers forward to the internal listener ([the internal listener](/reference/internal-listener/index.html.md#forwarded-requests)). `admin_socket`: the admin socket. |
| `procedure` | The procedure of a mounted service, for example `/hopbox.v1.InstallService/GetInstallInfo`. A path that names no service gets 404 and does not count, so the set of values stays bounded. |
| `code` | `ok`, or the Connect error code of the call, for example `unauthenticated` ([the API server](/reference/api/index.html.md#errors)) |

Some requests end before they reach a procedure, so these two metrics never see them. One is a path that names no service (HTTP 404). Another is a request that Connect refuses during its decode, such as a message larger than 4 MiB. The HTTP metric below counts every request, these ones too:

| Metric | Type | Labels | Description |
| --- | --- | --- | --- |
| `hopbox_http_requests_total` | counter | `listener`, `status` | The HTTP requests that ended, by listener and HTTP status, for example `200`, `404`, or `429`. |

The gRPC and gRPC-Web protocols send the code of a call in a trailer, with HTTP status 200. So a gRPC call that Connect refuses while it decodes it counts as `status="200"` in `hopbox_http_requests_total`, and not in `hopbox_api_calls_total`. The Connect protocol sends the code as an HTTP status, so its refusals count with their own status. A WebSocket of exec counts once, at its end, as `status="101"`. It is not in `hopbox_api_calls_total`.

The Grafana host counts in `hopbox_http_requests_total` with `listener="edge"`, together with the other forwarded requests ([the edge role](/reference/edge/index.html.md#the-grafana-host)). It is not in `hopbox_api_calls_total`, so `HopboxAPIErrors` does not read it. The panels of D1 and D5 that read `hopbox_http_requests_total` show its answers, also a `502` of a Grafana that does not answer.

The histogram has the default buckets of the Prometheus Go client: 0.005, 0.01, 0.025, 0.05, 0.1, 0.25, 0.5, 1, 2.5, 5, and 10 seconds.

## Metrics of placement

The control role counts the placements of new machines, so an instance admin sees a shortage of nodes ([Placement](/reference/placement/index.html.md#metrics)).

| Metric | Type | Labels | Description |
| --- | --- | --- | --- |
| `hopbox_placements_total` | counter | `result`, `reason` | The placements of `CreateMachine` ([the machine API](/reference/machine-api/index.html.md#createmachine)). |

| Label | Values |
| --- | --- |
| `result` | `placed` for a create that committed. `no_capacity` for a create that found no node with room ([Placement](/reference/placement/index.html.md#when-no-node-has-room)), or no free SSH port. |
| `reason` | Empty for `placed`. For `no_capacity`, the reason that skipped the most servers, from the order of the filters: `not_node`, `not_ready`, `degraded`, `pool`, `labels`, `class`, `version`, `cpu`, `ram`, or `refused`. A tie goes to the earlier filter. `none` for an install without a server. `ssh_ports` for a full range of SSH ports. |

A create that fails for another cause does not count. Examples are a usage limit and a taken name. A start of a stopped machine does not count either.

```sh
curl -s http://127.0.0.1:9464/metrics | grep '^hopbox_placements_total'
```

```text
hopbox_placements_total{reason="",result="placed"} 4
hopbox_placements_total{reason="ram",result="no_capacity"} 1
```

## Metrics of the accounts

The control role counts the first sign-ins of new people, so an instance admin sees a flood of sign-ups ([Sessions, cookies, and sign-in links](/reference/sessions/index.html.md#sign-up-modes)).

| Metric | Type | Labels | Description |
| --- | --- | --- | --- |
| `hopbox_account_creations_total` | counter | `method`, `state` | The first sign-ins of new people, by sign-in method and by the result. Each control process counts the sign-ins that it serves. |

| Label | Values |
| --- | --- |
| `method` | `email_link`, `github`, `google`, or `login_link` |
| `state` | `pending` for a new pending account, `active` for a new active account, and `refused` for a sign-in that a limit of new accounts or the sign-up mode `invite_only` refused ([Sessions, cookies, and sign-in links](/reference/sessions/index.html.md#limits-of-new-accounts)). The refusal while an older release runs does not count. A new account through an instance invitation or an org invitation counts as `active`. |

A sign-in counts after its transaction ends, so a refusal counts once for each try. A sign-in that reaches an account that exists does not count. A first-admin login link counts as `active`.

```sh
curl -s http://127.0.0.1:9464/metrics | grep '^hopbox_account_creations_total'
```

```text
hopbox_account_creations_total{method="github",state="pending"} 2
hopbox_account_creations_total{method="email_link",state="refused"} 1
```

## Metrics of the control role

Each control process counts these events itself, so the sum over the control processes is the count of the install. A count starts from zero at each start of `hopboxd`.

| Metric | Type | Labels | Description |
| --- | --- | --- | --- |
| `hopbox_audit_entries_total` | counter | `action`, `actor_kind` | The entries of the audit log that this control process wrote. An entry counts when its change commits. An entry of a change that rolled back does not count. |
| `hopbox_rate_limited_total` | counter | `limit` | The requests that a rate limit refused ([Sessions, cookies, and sign-in links](/reference/sessions/index.html.md#rate-limits), [Sign-in with GitHub and Google](/reference/oauth/index.html.md#rate-limits), [the org API](/reference/org-api/index.html.md)). |
| `hopbox_limit_refusals_total` | counter | `limit` | The calls that an org limit refused ([Domain rules](/reference/domain-rules/index.html.md#usage-limits)). |
| `hopbox_api_key_creations_total` | counter | `kind`, `lifetime` | The new API keys and admin keys ([CLI tokens, API keys, admin API keys, and SSH keys](/reference/credentials/index.html.md)). A second registration of the hash of an admin key does not count. |
| `hopbox_mail_sends_total` | counter | `kind`, `transport`, `result` | The emails that this control process sent ([SMTP config](/reference/smtp/index.html.md)). An email to each instance admin counts once for each admin. |
| `hopbox_sign_ins_total` | counter | `method`, `result` | The sign-ins that ended, also the refused ones ([Sessions, cookies, and sign-in links](/reference/sessions/index.html.md#what-a-sign-in-does)). A link of a new sign-in method to an account does not count. An OAuth callback that names an unknown provider does not count. |
| `hopbox_enrollments_total` | counter | `kind`, `result` | The calls of `Enroll` that ended ([Enrollment and the install PKI](/reference/enrollment/index.html.md#enroll)). A call that fails before `hopboxd` knows its kind, such as one with a token of a bad form, counts as `first`, or as `add_roles` or `remove_roles` for a server that calls with its certificate. |
| `hopbox_moves_total` | counter | `result`, `cause` | The cold moves that ended ([Cold move](/reference/cold-move/index.html.md)). A move counts once, when the change that ends it commits, in the control process that ends it. A later give-back of a move that ended does not count again. Each series starts at 0. |
| `hopbox_invitations_total` | counter | `kind`, `result` | The invitations to orgs ([the org API](/reference/org-api/index.html.md#invitations)) and to the install ([the admin API](/reference/admin-api/index.html.md#invitationservice)). An invitation counts when its change commits. An expired invitation counts when the sweep deletes it, 24 hours after its expiry. |

| Label | Values |
| --- | --- |
| `action` | The action of the audit entry, such as `org.create`, `api_key.create`, `machine.create`, or `machine.set_url_port` |
| `actor_kind` | `anonymous`, `session`, `cli_token`, `member_key`, `org_key`, `admin_key`, `local_root`, `server`, `ssh_key`, `control`, or `system` |
| `limit` of `hopbox_rate_limited_total` | `sign_in_links`: the sign-in links of an email or of a client. `device_sign_ins`: the device sign-ins of a client, and the device sign-ins that wait for an approval. `device_codes`: the wrong codes of the device approvals of an account. `oauth_starts`: the sign-ins with GitHub and Google of a client and of the install. `oauth_callbacks`: the callbacks of a client in one control process. `invitations`: the invitations of an org, of the install, of an inviter, and to an email ([the admin API](/reference/admin-api/index.html.md#invitationservice)). `machine_url_codes`: the machine URL codes of a browser. `grafana_codes`: the codes of the Grafana host of a console session ([the edge role](/reference/edge/index.html.md#the-grafana-host)). `account_creations`: the new accounts of a client address in a day ([Sessions, cookies, and sign-in links](/reference/sessions/index.html.md#limits-of-new-accounts)). `pending_cap`: the first sign-ins that the cap of 50 pending accounts refused. `sign_up_notes`: the changes of the note of a pending account in an hour ([the sign-in, account, and audit API](/reference/auth-api/index.html.md#setsignupnote)). |
| `limit` of `hopbox_limit_refusals_total` | `machines`, `vcpus`, `memory`, and `disk`: the usage limits of the org. `public_urls`: a change of a machine URL to public in an org without public URLs ([the machine API](/reference/machine-api/index.html.md#setmachineurlvisibility)). `orgs`: the orgs that one account owns (`max_orgs_per_account`), and each `CreateOrg` of a limited account. `stored_state`, `snapshots`, `backups`, and `saved_bytes_per_day`: the limits of saved state ([Domain rules](/reference/domain-rules/index.html.md#limits-of-saved-state)). Today only `saved_bytes_per_day` refuses a call, a hibernate. |
| `kind` of `hopbox_api_key_creations_total` | `member`, `org`, or `admin` |
| `lifetime` | `30d`, `90d`, `365d`, or `custom` for a key with a custom expiry |
| `kind` of `hopbox_mail_sends_total` | `magic_link`: a sign-in link. `email_link`: the link that adds an email to an account. `invitation`: the link of an org invitation. `admin_notice`: an admin notice. `digest`: the digest of new pending accounts. `approval`: the email to a person whose account an instance admin approved. |
| `transport` | `smtp`: through the SMTP server of the install. `file`: to a file in the mail folder of the control server ([SMTP config](/reference/smtp/index.html.md)). |
| `result` of `hopbox_mail_sends_total` | `sent`, or `failed` for an email that the SMTP server refused, that timed out, or whose file write failed |
| `method` | `email_link`, `login_link`, `github`, or `google` |
| `result` of `hopbox_sign_ins_total` | `ok`. `refused`: a link or an OAuth flow that is unknown, used, or expired, a refusal by the provider, or an email that the provider did not verify. `conflict`: a conflict with a sign-in method of another account ([Sessions, cookies, and sign-in links](/reference/sessions/index.html.md#link-a-new-sign-in-method)). `suspended`: an account that an instance admin suspended. `sign_up_refused`: a new account that the block list, the sign-up mode `invite_only`, or a limit of new accounts refused, or whose invitation ended during the sign-in ([Sessions, cookies, and sign-in links](/reference/sessions/index.html.md#limits-of-new-accounts)). `failed`: another error, such as a provider that did not answer. |
| `kind` of `hopbox_enrollments_total` | `first`: a new server ([Enrollment and the install PKI](/reference/enrollment/index.html.md#hopboxd-enroll)). `add_roles` and `remove_roles`: a change of the roles of an enrolled server. `retry`: a retry after a lost answer. `resync`: a resync after the end of the token. |
| `result` of `hopbox_enrollments_total` | `ok`. `refused`: an error of the request, such as a token that is unknown, used, expired, or for another CA, a bad field, or a server that the registry refuses. `failed`: another error, such as an error of the control store. |
| `result` of `hopbox_moves_total` | `done`: the target has the whole disk. `canceled`: an instance admin canceled the move, its drain, or the removal of the node role, also when the machine went back to its source. `given_back`: the move failed while the target pulled, and the machine went back to its source node, stopped. `failed`: the move failed, and the machine stayed where it was. |
| `cause` | `admin`: `MoveMachine`. `drain`: the drain of the source node. `removal`: the removal of the node role of the source node. A move that a `hopboxd` of schema version 57 to 66 started has no cause in the store ([Control store tables](/reference/control-store/index.html.md#machine_moves)). It counts as `drain` while its source node has a drain, as `removal` while the source node loses its node role, and as `admin` otherwise. |
| `kind` of `hopbox_invitations_total` | `org`: an invitation to an org. `instance`: an instance invitation, which lets a new person in ([Sessions, cookies, and sign-in links](/reference/sessions/index.html.md#sign-up-modes)). |
| `result` of `hopbox_invitations_total` | `created`, `accepted`, `revoked`, or `expired`. A new invitation to an email that has an open one revokes the old one, and counts as `revoked` too. An instance invitation counts as `accepted` with the first sign-in of its invitee, which makes the account. |

Each value of `limit`, `kind`, `lifetime`, `transport`, `method`, `result`, and `cause` starts at 0, so the first refusal, key, email, sign-in, invitation, or move shows as an increase. A series of `hopbox_audit_entries_total` shows after its first entry.

```sh
curl -s http://127.0.0.1:9464/metrics | grep -E '^hopbox_(audit_entries|rate_limited|limit_refusals|api_key_creations|mail_sends)_total'
```

```text
hopbox_api_key_creations_total{kind="member",lifetime="90d"} 2
hopbox_audit_entries_total{action="org.create",actor_kind="session"} 1
hopbox_limit_refusals_total{limit="machines"} 1
hopbox_mail_sends_total{kind="magic_link",result="sent",transport="smtp"} 4
hopbox_rate_limited_total{limit="sign_in_links"} 3
```

## Metrics of the leader

The leader of the control role reads these gauges from the control store every minute, and at the start of its jobs ([hopboxd](/reference/hopboxd/index.html.md#leader-jobs)). It serves them until its jobs end, at the end of its term or in maintenance mode. A read takes at most 30 s. A read that fails keeps the values of the read before. After 3 failed reads in a row, the leader serves none of the gauges until a read works, so a dashboard shows a gap and not old values. Another control process serves none of them, so no value counts twice. The leader serves each value of the fixed label sets, so a count that falls to 0 shows as 0.

| Metric | Type | Labels | Description |
| --- | --- | --- | --- |
| `hopbox_accounts` | gauge | `state`, `tier` | The accounts by state (`pending`, `active`, or `suspended`) and tier (`limited` or `full`). |
| `hopbox_pending_account_oldest_seconds` | gauge | none | The age of the oldest `pending` account, or 0 without one. |
| `hopbox_orgs` | gauge | `kind`, `suspended` | The orgs by kind (`personal` or `team`) and suspension (`true` or `false`). |
| `hopbox_api_keys` | gauge | `kind` | The API keys that did not expire, by kind (`member`, `org`, or `admin`). |
| `hopbox_public_machine_urls` | gauge | none | The machines with a public URL, apart from the deleted machines ([Run the edge role and open machine URLs](/guide/machine-urls/index.html.md)). |
| `hopbox_machines` | gauge | `state` | The machines by current state ([Domain rules](/reference/domain-rules/index.html.md#machine-states)). A deleted machine counts as `deleted` until the sweep drops its row. |
| `hopbox_machines_off_target` | gauge | none | The machines whose current state differs from their target state in each read of the last 5 minutes or more, such as a machine that stays `failed` with the target `running`. A machine with an open cold move, in `stopping` or `pulling`, does not count (Q147). The moves have their own gauge, deadlines, and alert. A machine that stays in `moving` without an open move counts, because nothing moves it any more. The leader counts from its own reads while its jobs run. So the 5 minutes start again after a change of the leader, and after a pause of the jobs by maintenance mode or by the fence. |
| `hopbox_moves` | gauge | `state` | The cold moves by state: `stopping`, `pulling`, `done`, or `failed` ([Cold move](/reference/cold-move/index.html.md)). A move that ended stays as `done` or `failed` until the sweep drops its machine. |
| `hopbox_drain_stranded_machines` | gauge | `server` | The machines that the drain or the removal of the node role of the server cannot move ([Cold move](/reference/cold-move/index.html.md#drain-of-a-node)). Only a server whose drain or removal runs has a series, 0 when every machine can move. |
| `hopbox_admin_notices_open` | gauge | `kind` | The open admin notices by kind, `security`, `sign_up`, `alert`, or `abuse` ([the admin API](/reference/admin-api/index.html.md#notices-to-instance-admins)). The kind `alert` counts the open notices of the alerts of the monitoring bundle ([the admin API](/reference/admin-api/index.html.md#notices-of-alerts)). The kind `abuse` counts the open notices of the detection of abuse ([Abuse detection](/reference/abuse-detection/index.html.md#notices)). A notice that several instance admins have counts once, until the last of them dismisses it. |
| `hopbox_store_database_bytes` | gauge | none | The size of the database of the control store, from `pg_database_size`. |

The registry gets the capacity of each server in its heartbeats ([the control channel](/reference/control-channel/index.html.md)). The leader serves the capacity and the allocations of each server with the node role that is not decommissioned, with the label `server`, the ID of the server. So one Prometheus on the control server sees the capacity of the whole fleet without a scrape of the nodes.

| Metric | Type | Labels | Description |
| --- | --- | --- | --- |
| `hopbox_server_vcpus` | gauge | `server` | The vCPU capacity of the node: its CPU threads times the vCPU overcommit of 2 ([Placement](/reference/placement/index.html.md)). |
| `hopbox_server_vcpus_allocated` | gauge | `server` | The vCPUs of the machines that use compute on the node, as placement counts them ([Placement](/reference/placement/index.html.md#what-a-node-uses)). |
| `hopbox_server_memory_bytes` | gauge | `server` | The RAM of the node. |
| `hopbox_server_memory_allocated_bytes` | gauge | `server` | The RAM of the machines that use compute on the node, as placement counts them. |
| `hopbox_server_disk_bytes` | gauge | `server` | The size of the file system of the data dir of the node. |
| `hopbox_server_disk_free_bytes` | gauge | `server` | The free space of that file system. |
| `hopbox_server_disk_allocated_bytes` | gauge | `server` | The disk sizes of the machines on the node that are not deleted. A disk is sparse, so this sum can be larger than the disk of the node. |
| `hopbox_server_machines` | gauge | `server`, `state` | The machines on the node by current state. |

The leader also serves the registry of every server, whatever its roles ([Enrollment and the install PKI](/reference/enrollment/index.html.md), [the control channel](/reference/control-channel/index.html.md)).

| Metric | Type | Labels | Description |
| --- | --- | --- | --- |
| `hopbox_servers` | gauge | `state` | The servers by state: `ready`, `cordoned`, `unresponsive`, `draining`, `shutting_down`, or `decommissioned`. |
| `hopbox_server_info` | gauge | `server`, `name`, `version`, `status` | 1 for each server that is not decommissioned. `name` is its host name, `version` the release of its `hopboxd`, and `status` is `ok`, `degraded`, or empty before its first heartbeat. |
| `hopbox_server_heartbeat_age_seconds` | gauge | `server` | The age of the last heartbeat of each server that is not decommissioned, at the time of the scrape. A server without a heartbeat has no series. The leader reads the times of the heartbeats every 5 s, so the age is exact to about 5 s. |
| `hopbox_server_dns_zone_serial` | gauge | `server` | The serial of the zones that each server with the DNS role reports in its heartbeats ([the DNS role](/reference/dns/index.html.md)). A DNS server without zones has no series. |

Each node has 16 series: one for each of the 7 capacity gauges, and 9 of `hopbox_server_machines`, one for each state. Each server that is not decommissioned has 1 series of `hopbox_server_info`, and up to 1 of the heartbeat age and 1 of the zone serial. The leader also serves 6 series of `hopbox_accounts`, 4 of `hopbox_orgs`, 3 of `hopbox_api_keys`, 9 of `hopbox_machines`, 4 of `hopbox_moves`, 6 of `hopbox_servers`, and 3 of `hopbox_admin_notices_open`.

```sh
curl -s http://127.0.0.1:9464/metrics | grep -E '^hopbox_(accounts|server_vcpus)'
```

```text
hopbox_accounts{state="active",tier="full"} 3
hopbox_accounts{state="pending",tier="limited"} 1
hopbox_server_vcpus{server="0193a5b2-7c1e-7d4a-9f3b-2a1c4e5f6a7b"} 24
hopbox_server_vcpus_allocated{server="0193a5b2-7c1e-7d4a-9f3b-2a1c4e5f6a7b"} 4
```

```sh
curl -s http://127.0.0.1:9464/metrics | grep -E '^hopbox_(servers|server_info|server_heartbeat)'
```

```text
hopbox_server_heartbeat_age_seconds{server="0193a5b2-7c1e-7d4a-9f3b-2a1c4e5f6a7b"} 2.1
hopbox_server_info{name="ks",server="0193a5b2-7c1e-7d4a-9f3b-2a1c4e5f6a7b",status="ok",version="v0.9.0"} 1
hopbox_servers{state="ready"} 1
```

## Metrics of the leader jobs

These metrics show the leader lease, the jobs of the leader, and the state of each control process ([hopboxd](/reference/hopboxd/index.html.md#leader-jobs)).

| Metric | Type | Labels | Description |
| --- | --- | --- | --- |
| `hopbox_leader` | gauge | none | 1 on the control process that holds the leader lease, and 0 on each other control process. The sum over the control processes is 1 while the install has a leader. |
| `hopbox_control_fenced` | gauge | none | 1 while this control process is fenced from the control store, and 0 otherwise. A fenced process takes no call and runs no leader job ([hopboxd](/reference/hopboxd/index.html.md#run)). |
| `hopbox_maintenance` | gauge | none | 1 while the install is in maintenance mode, and 0 otherwise. Only the leader serves it, also while maintenance mode pauses its jobs ([the API server](/reference/api/index.html.md#maintenance-mode)). |
| `hopbox_leader_job_runs_total` | counter | `job`, `result` | The runs of each leader job that ended, by `result`: `ok`, or `failed` for a run that logged a failure. A run that the end of the jobs cut does not count. Each series starts at 0. |
| `hopbox_leader_job_last_success_seconds` | gauge | `job` | The Unix time of the last run of the job that worked. The leader serves it while its jobs run, after the first run of the job that worked. At the end of the term, and while maintenance mode or the fence pauses the jobs, the series go away. |

| `job` | Runs |
| --- | --- |
| `sweep` | At the start of the jobs, then every 10 minutes: the sweep of idempotency keys, sign-ins, sign-ups, invitations, machine names and ports, change events, control processes, enrollment tokens and server certificates, and samples of the history. It fails when one of its parts fails. |
| `pending_expiry` | With each sweep: the delete of the pending accounts older than 30 days ([Sessions, cookies, and sign-in links](/reference/sessions/index.html.md#pending-accounts)). |
| `unresponsive` | Every 5 s, 20 s after the start of the jobs: the mark of silent servers as unresponsive. |
| `auto_delete` | Every minute: the delete of the machines that nobody used for their auto-delete timeout. |
| `moves` | Every 2 s: the steps of the cold moves ([Cold move](/reference/cold-move/index.html.md)). |
| `drains`, `removals` | Every 5 s: the steps of the drains and of the removals of the node role. |
| `sign_up_digest` | At the start of the jobs, then every minute: the digest of new pending accounts. |
| `host_keys` | Every minute: the return of SSH host keys that no route record holds. |
| `abuse_detection` | Every minute: the detection of abuse for each account ([Abuse detection](/reference/abuse-detection/index.html.md#the-leader-job)). A run that its timeout of 30 s cuts counts as `failed`. |
| `acme` | Each turn of the ACME job: a read of the state of the certificate, and the attempt when one is due. A turn follows each attempt, at most 1 hour after the turn before, and 1 minute after a failed read ([the certificate of the install](/reference/acme/index.html.md#the-acme-job)). |

The ACME job also serves the state of the certificate of the install. Only the leader runs the job, so only the leader serves these gauges, from the last read of its state.

| Metric | Type | Labels | Description |
| --- | --- | --- | --- |
| `hopbox_acme_attempts_total` | counter | `result` | The attempts for the certificate of the install, by `result`: `issued`, `failed`, or `rate_limited` for a rate limit of the CA. Each series starts at 0. An attempt that the end of the term cut does not count. |
| `hopbox_acme_failures_in_row` | gauge | none | The attempts that failed since the last certificate. |
| `hopbox_acme_certificate_not_after_seconds` | gauge | none | The end of the certificate of the install in the control store, in seconds since the Unix epoch. The edge servers load this certificate. An install without a certificate does not serve it. |

```sh
curl -s http://127.0.0.1:9464/metrics | grep -E '^hopbox_(leader|maintenance|control_fenced|acme_)'
```

```text
hopbox_acme_attempts_total{result="issued"} 1
hopbox_acme_certificate_not_after_seconds 1.7998e+09
hopbox_acme_failures_in_row 0
hopbox_control_fenced 0
hopbox_leader 1
hopbox_leader_job_last_success_seconds{job="sweep"} 1.7922494e+09
hopbox_leader_job_runs_total{job="sweep",result="ok"} 12
hopbox_maintenance 0
```

## Metrics of the control store

Each control process serves these metrics of the control store as it sees it ([Control store tables](/reference/control-store/index.html.md)). The pool metrics cover the connection pool of the process, which runs the queries of the API, the leader jobs, and the control channels. The change events and the leader election use their own connections, outside the pool.

| Metric | Type | Labels | Description |
| --- | --- | --- | --- |
| `hopbox_store_connections` | gauge | `state` | The connections of the pool: `in_use` by a call, `idle`, `total`, the open connections, and `max`, the most connections that the pool opens. `in_use` equal to `max` means a full pool, and calls then wait for a connection. |
| `hopbox_store_acquire_wait_seconds_total` | counter | none | The time that calls waited for a connection of the pool. |
| `hopbox_store_errors_total` | counter | `kind` | The queries of the pool and the waits for a connection that failed. Each value of `kind` starts at 0. |
| `hopbox_store_renewals_total` | counter | `result` | The renewals of the row of this control process every 5 s, by `result`: `ok` or `failed` ([hopboxd](/reference/hopboxd/index.html.md#run)). A renewal that the stop of the process cut does not count. |
| `hopbox_store_schema_version` | gauge | none | The schema version of the control store, the number of its newest migration. The process reads it after its migrations, then every 10 s ([the bundled Postgres](/reference/postgres/index.html.md#schema-migrations)). |
| `hopbox_events_lag_seconds` | gauge | none | The age of the last change event that this control process read, at the time of its read ([Control store tables](/reference/control-store/index.html.md#events)). It is 0 before the first event. A clock of the server ahead of the clock of Postgres gives 0. The process reads up to the end of the events after each new event, and at least every 30 s. After 1 minute without such a read, for example after a lost connection, the gauge gives the time since the last one, so a stalled subscription shows as a growing lag. It shows once the events unit starts. |
| `hopbox_postgres_up` | gauge | none | 1 while the bundled Postgres runs and takes connections, and 0 while it restarts or after its stop ([the bundled Postgres](/reference/postgres/index.html.md#start-restart-and-stop)). A control process with an external Postgres does not serve it. |
| `hopbox_postgres_restarts_total` | counter | none | The starts of the bundled Postgres after an exit that `hopboxd` did not ask for, such as a crash. A control process with an external Postgres does not serve it. |
| `hopbox_postgres_disk_free_bytes` | gauge | none | The free bytes of the file system that holds the data dir of the bundled Postgres, for a user that is not root, read at each scrape (Q155). A control process with an external Postgres does not serve it. A read that fails leaves it out of that scrape. |

| `kind` | Errors |
| --- | --- |
| `connection` | A lost or refused connection: a network error, a closed connection, an error of class 08, a shutdown of the server (57P01 to 57P03), or too many connections (53300). A failed wait for a connection of the pool with another error counts here too. |
| `timeout` | A deadline of the call, a statement timeout or cancel (57014), a lock timeout (55P03), or an idle transaction timeout (25P03). |
| `serialization` | A serialization failure (40001) or a deadlock (40P01). |
| `other` | Every other error of a query, such as a missing table. |

A query that finds no row, a call that its caller canceled, and a refusal of an integrity constraint (class 23) do not count. The service reads such a refusal as an answer, for example a name that another machine took.

```sh
curl -s http://127.0.0.1:9464/metrics | grep -E '^hopbox_(store|events|postgres)_'
```

```text
hopbox_events_lag_seconds 0.004
hopbox_postgres_disk_free_bytes 3.1e+11
hopbox_postgres_restarts_total 0
hopbox_postgres_up 1
hopbox_store_acquire_wait_seconds_total 0.021
hopbox_store_connections{state="idle"} 3
hopbox_store_connections{state="in_use"} 1
hopbox_store_connections{state="max"} 4
hopbox_store_connections{state="total"} 4
hopbox_store_errors_total{kind="timeout"} 0
hopbox_store_renewals_total{result="ok"} 720
hopbox_store_schema_version 68
```

## Metrics of the control channels

Each control process serves the control channels that it holds, and the install CA ([the control channel](/reference/control-channel/index.html.md)).

| Metric | Type | Labels | Description |
| --- | --- | --- | --- |
| `hopbox_channels` | gauge | none | The control channels of servers that this control process holds now. It shows once the internal listener runs. |
| `hopbox_heartbeat_writes_total` | counter | `result` | The heartbeats of the channels of this control process, by `result`: `written` to the registry, `skipped` because it came less than 2.5 s after the last write of its channel, or `failed` ([the control channel](/reference/control-channel/index.html.md#the-heartbeat)). Each value starts at 0. |
| `hopbox_install_ca_not_after_seconds` | gauge | none | The end of the install CA, in seconds since the Unix epoch ([Enrollment and the install PKI](/reference/enrollment/index.html.md#install-ca)). It shows once the process opened the CA, at its start when its keyring has a key, or at the first call that needs the CA. |

Each server that enrolls serves the metrics of its own channel and of its certificate, apart from a server with only the control role ([hopboxd](/reference/hopboxd/index.html.md#run)).

| Metric | Type | Labels | Description |
| --- | --- | --- | --- |
| `hopbox_channel_connected` | gauge | none | 1 while this server holds its control channel, from the first `HeartbeatAck` of the channel until the channel ends, and 0 otherwise. |
| `hopbox_channel_reconnects_total` | counter | `reason` | The control channels of this server that ended, after which it dials again. A channel that the stop of `hopboxd` ends does not count. Each value starts at 0. |
| `hopbox_server_certificate_not_after_seconds` | gauge | none | The end of the certificate of this server, in seconds since the Unix epoch ([Enrollment and the install PKI](/reference/enrollment/index.html.md#server-certificate)). A server that is not enrolled does not serve it. |
| `hopbox_certificate_renewals_total` | counter | `result` | The renewals of the certificate of this server over the control channel ([the control channel](/reference/control-channel/index.html.md#certificate-renewal)). Each value starts at 0. |

| Label | Values |
| --- | --- |
| `reason` | `renewed`: the server dials again with its renewed certificate. `refused`: the control plane refused the server (`unauthenticated` or `permission_denied`). `no_ack`: the control plane answered no heartbeat for 4 intervals. `closed`: the control role closed the channel without an error. `lost`: another error, such as a lost connection, a newer channel of the server, or a stop of the control role. |
| `result` of `hopbox_certificate_renewals_total` | `issued`: the server got and saved a new certificate. `refused`: the control plane refused the renewal, for example by its rate limit. `unanswered`: no answer within 4 heartbeats. `failed`: the save of the certificate that the server got failed. |

```sh
curl -s http://127.0.0.1:9464/metrics | grep -E '^hopbox_(channel|channels|heartbeat_writes|server_certificate|certificate_renewals|install_ca)'
```

```text
hopbox_certificate_renewals_total{result="issued"} 1
hopbox_channel_connected 1
hopbox_channel_reconnects_total{reason="renewed"} 1
hopbox_channels 3
hopbox_heartbeat_writes_total{result="written"} 4310
hopbox_install_ca_not_after_seconds 2.1078e+09
hopbox_server_certificate_not_after_seconds 1.8003e+09
```

## Metrics of the DNS role

A server with the DNS role adds these metrics to the listener of its `hopboxd run` ([the DNS role](/reference/dns/index.html.md#metrics)). A server without the DNS role does not have them.

| Metric | Type | Labels | Description |
| --- | --- | --- | --- |
| `hopbox_dns_queries_total` | counter | `transport`, `rcode` | The DNS queries that the DNS role answered. |
| `hopbox_dns_rate_limited_total` | counter | `action` | The UDP answers over the rate limit of their client network. `action` is `slip`, a short answer with the TC flag, or `drop`, no answer ([the DNS role](/reference/dns/index.html.md#rate-limit-on-udp)). |
| `hopbox_dns_zone_serial` | gauge | none | The serial of the zones that the DNS role serves. It is 0 before the first zones arrive. |

| Label | Values |
| --- | --- |
| `transport` | `udp` or `tcp` |
| `rcode` | The response code of the answer, such as `NOERROR`, `NXDOMAIN`, `REFUSED`, `SERVFAIL`, `FORMERR`, `NOTIMP`, or `BADVERS` |

```sh
curl -s http://127.0.0.1:9464/metrics | grep '^hopbox_dns_'
```

```text
hopbox_dns_queries_total{rcode="NOERROR",transport="udp"} 12
hopbox_dns_queries_total{rcode="NXDOMAIN",transport="udp"} 2
hopbox_dns_zone_serial 1.7922494e+09
```

The Prometheus text format writes a large gauge in exponent form, so the serial 1792249400 shows as `1.7922494e+09`.

## Metrics of the edge role

A server with the edge role adds these metrics to the listener of its `hopboxd run` ([the edge role](/reference/edge/index.html.md#metrics)). A server without the edge role does not have them.

| Metric | Type | Labels | Description |
| --- | --- | --- | --- |
| `hopbox_edge_requests_total` | counter | `to` | The HTTP requests that the edge role took, by where they went. |
| `hopbox_edge_tls_refusals_total` | counter | `reason` | The TLS handshakes that the edge role refused. |
| `hopbox_edge_client_connection_refusals_total` | counter | `listener`, `bound` | The connections that the edge role closed at once, because their client or their IPv6 /48 held the most open connections ([the edge role](/reference/edge/index.html.md#listeners)). |
| `hopbox_edge_open_connections` | gauge | `listener` | The connections that a listener of the edge role holds now. |
| `hopbox_edge_connection_limit` | gauge | `listener` | The most connections that a listener of the edge role holds at once. |
| `hopbox_edge_routes` | gauge | none | The machines that the machine URLs of the edge role name. |
| `hopbox_edge_certificate_not_after_seconds` | gauge | none | The end of the certificate that the edge role serves, in seconds since the Unix epoch. It is 0 before the first certificate. |
| `hopbox_edge_machine_requests_total` | counter | `visibility`, `status_class` | The HTTP requests that the edge role forwarded to a machine, by the visibility of the machine URL (`public` or `private`) and the class of the status of the answer (`1xx` to `5xx`). A forward that failed counts as `5xx`. Each of the 10 series starts at 0 ([the edge role](/reference/edge/index.html.md#metrics)). |
| `hopbox_edge_upstream_errors_total` | counter | `reason` | The 502 and 503 answers to the requests of a machine URL, by cause: `not_running`, `no_address`, `no_answer`, or `expired_lease` ([the edge role](/reference/edge/index.html.md#metrics)). Each of the 4 series starts at 0. |
| `hopbox_edge_expired_targets` | gauge | none | The targets of the machine URLs whose lease ended at the edge. |
| `hopbox_edge_ssh_connections_total` | counter | `result` | The TCP connections of the SSH front door, by result ([the SSH front door](/reference/ssh-front-door/index.html.md#metrics)). |
| `hopbox_edge_ssh_auth_total` | counter | `result` | The keys that SSH clients signed with: `allowed`, `refused`, `limited`, or `error`. |
| `hopbox_edge_ssh_unreachable_total` | counter | `reason` | The signed-in SSH connections that reached no OpenSSH server of a machine. |
| `hopbox_edge_ssh_channels_total` | counter | `type`, `result` | The SSH channels that the front door forwarded or refused. |
| `hopbox_edge_ssh_ended_total` | counter | `reason` | The open SSH connections that the front door ended. |
| `hopbox_edge_ssh_open_connections` | gauge | none | The signed-in SSH connections that the front door holds now. |
| `hopbox_edge_ssh_lost_opens` | gauge | none | The channel opens of the front door that got no answer and still wait, after their connection ended ([the SSH front door](/reference/ssh-front-door/index.html.md#channels-and-requests)). |

| Label | Values |
| --- | --- |
| `to` | `api`: to the control role. `machine`: to a machine. `redirect`: a redirect to HTTPS or to a sign-in, or the cookie of a sign-in. `refused`: a page of the edge, such as 404, 421, or 503. |
| `reason` | `no_sni`: the client sent no server name. `unknown_host`: the name is not under the domain. `no_certificate`: the edge has no certificate for the domain yet. |

```sh
curl -s http://127.0.0.1:9464/metrics | grep '^hopbox_edge_'
```

```text
hopbox_edge_certificate_not_after_seconds 1.7998e+09
hopbox_edge_requests_total{to="api"} 42
hopbox_edge_requests_total{to="machine"} 7
hopbox_edge_requests_total{to="redirect"} 3
hopbox_edge_routes 2
hopbox_edge_tls_refusals_total{reason="no_sni"} 1
```

## Metrics of the executors

The executors of a node count their actions, and what happens to their machines and guest links ([the node executor](/reference/executor/index.html.md#metrics)). `hopboxd run` registers them on a server with the node role.

| Metric | Type | Labels | Description |
| --- | --- | --- | --- |
| `hopbox_executor_actions_total` | counter | `action`, `result` | The actions of the executors of the node, such as `boot`, `pause`, `resume`, `hibernate`, and `wake` ([the node executor](/reference/executor/index.html.md#metrics)). `drop_memory` drops the memory file of a woken machine in a later step, after a drop that failed or a restart of `hopboxd`. `discard_memory` drops it for `DiscardMemory`. |
| `hopbox_executor_action_duration_seconds` | histogram | `action` | The duration of the actions. |
| `hopbox_executor_lease_losses_total` | counter | none | The machine leases that the node lost: the control role did not renew one or refused a report with it, or it expired. Each grant counts once, 10 s after its end. The end of a grant by a cold move, or after a delete, does not count ([the node executor](/reference/executor/index.html.md#metrics)). |
| `hopbox_executor_boot_phase_seconds` | histogram | `phase` | The duration of each phase of a boot or a reboot: `unit_start`, `vmm_ready`, `guest_link`, and `start_command`. |
| `hopbox_machine_restarts_total` | counter | `machine`, `reason` | The restarts of each machine by the restart policy: `crash`, `guest_shutdown`, or `start_command_exit`. The series of a machine go away when it leaves the node. |
| `hopbox_guest_link_handshakes_total` | counter | `result` | The handshakes of the guest links: `ok`, `bad_proof`, `no_version`, or `error`. Each of the 4 series starts at 0. |
| `hopbox_guest_links_down` | gauge | none | The machines whose microVM runs on the node without an authenticated guest link. A boot counts until its guest agent connects. A machine that runs before its first spec after a restart of `hopboxd` does not count. A machine in standby does not count, because the node closed its link on purpose. |
| `hopbox_vmm_save_seconds` | histogram | `kind` | The duration of each save of the memory of a machine by the VMM driver that worked, by kind: `hibernate`. The series starts at 0. |
| `hopbox_vmm_load_seconds` | histogram | `kind` | The duration of each load of saved memory by the VMM driver that worked, until the microVM runs, by kind: `wake`. The wait for the guest agent does not count. The series starts at 0. |
| `hopbox_saved_state_refusals_total` | counter | `reason` | The saves of memory that the node refused, by reason: `disk_low`, for a save that leaves less free disk than the floor of `vmm.save_free_percent` and `vmm.save_free_bytes` ([the node executor](/reference/executor/index.html.md#the-gate-of-a-save)). The series starts at 0. `HopboxSavedStateDiskLow` fires on it. |

The catalog names the phases and the results in words, so these label values stand for them:

| Label | Values |
| --- | --- |
| `phase` | `unit_start`: the unit start of the catalog, from the start of the boot of the driver until systemd started the unit. `vmm_ready`: the "Firecracker ready" of the catalog, until Firecracker answers and started the microVM. The name holds no driver, because the VMM layer takes other drivers. `guest_link`: until the guest agent applied its configuration on an authenticated link. `start_command`: until the guest agent reports the first run of the start command. |
| `reason` of the restarts | `crash`, `guest_shutdown`, and `start_command_exit`, as the catalog names them. A boot that failed is no restart, and counts only in `hopbox_executor_actions_total`. |
| `result` of the handshakes | `ok`. `bad_proof`: the failed proofs of the catalog, a guest proof that does not match the link key. `no_version`: no common version of the guest link. `error`: any other failure, such as a broken message, a timeout, or a guest agent that ends the call before its proof. |

```sh
curl -s http://127.0.0.1:9464/metrics | grep -E '^hopbox_(executor_lease|machine_restarts|guest_link)'
```

```text
hopbox_executor_lease_losses_total 0
hopbox_guest_link_handshakes_total{result="bad_proof"} 0
hopbox_guest_link_handshakes_total{result="error"} 0
hopbox_guest_link_handshakes_total{result="no_version"} 0
hopbox_guest_link_handshakes_total{result="ok"} 12
hopbox_guest_links_down 0
hopbox_machine_restarts_total{machine="0193a5b2-7c1e-7d4a-9f3b-2a1c4e5f6a7b",reason="crash"} 1
```

## Metrics of the machines

A server with the node role serves what its running machines use. At each scrape, the node reads the cgroup of the unit `hopbox-machine-<id>.service` of each machine that runs ([the Firecracker driver](/reference/firecracker-driver/index.html.md)). The cgroup holds Firecracker and its threads, so the values include the VMM. The node totals come from the cgroup of the slice `hopbox-machines.slice`.

| Metric | Type | Labels | Description |
| --- | --- | --- | --- |
| `hopbox_machine_info` | gauge | `machine`, `name`, `org`, `size`, `vcpus`, `memory_bytes`, `disk_bytes`, `state` | 1 for each machine that runs on the node, with `state` `running`. Also 1 with `state` `saved` for each machine whose saved state the node keeps without a microVM, such as a hibernated machine. It is the only series that links a machine to its org, by org ID. A query joins it to the other series on `machine`. A count of the running machines leaves out `state="saved"`. |
| `hopbox_machine_vcpus` | gauge | `machine` | The vCPUs of the size of the machine, the value of the label `vcpus` of `hopbox_machine_info`. A query divides the CPU time by it, because PromQL cannot read a label as a number. |
| `hopbox_machine_cpu_seconds_total` | counter | `machine` | The CPU time of the machine since its boot, from `usage_usec` of `cpu.stat`. It starts from zero at each boot. |
| `hopbox_machine_memory_bytes` | gauge | `machine` | The host RAM that the machine holds, from `memory.current`. |
| `hopbox_machine_disk_bytes` | gauge | `machine` | The bytes of the blocks that the sparse disk file of the machine holds on the node. |
| `hopbox_machine_cpu_pressure_seconds_total` | counter | `machine` | The time that at least one process of the machine waited for a CPU since its boot, from the line `some` of `cpu.pressure`. |
| `hopbox_machine_oom_kills_total` | counter | `machine` | The processes of the unit of the machine that the kernel killed for a lack of memory since its boot, from `oom_kill` of `memory.events`. A kill inside the guest does not count: the guest kernel ends its own processes. |
| `hopbox_machine_disk_read_bytes_total` | counter | `machine` | The bytes that the machine read from the block devices of the node since its boot, from `rbytes` of `io.stat`. Only the device of the file system of `<data_dir>/machines` counts. |
| `hopbox_machine_disk_written_bytes_total` | counter | `machine` | The bytes that the machine wrote to the block devices of the node since its boot, from `wbytes` of `io.stat`. Only the device of the file system of `<data_dir>/machines` counts. |
| `hopbox_machines_cpu_seconds_total` | counter | none | The CPU time of every machine of the node, from `cpu.stat` of the slice. |
| `hopbox_machines_memory_bytes` | gauge | none | The host RAM of every machine of the node, from `memory.current` of the slice. |
| `hopbox_machine_saved_state_bytes` | gauge | `machine` | The bytes of the disk of the node that the saved state of the machine holds: its memory file and its VMM state ([the Firecracker driver](/reference/firecracker-driver/index.html.md#the-bytes-of-a-saved-state)). A hibernated machine has it, and so does a machine during its wake, until the node drops the file. |
| `hopbox_saved_state_bytes` | gauge | `kind` | The bytes of the disk of the node that the saved state of all its machines holds, by kind: `memory`, the memory files of hibernated machines. The series is there also at 0. |

| Label | Values |
| --- | --- |
| `machine` | The ID of the machine |
| `name` | The name of the machine |
| `org` | The ID of the org of the machine, or empty for a machine whose org is gone |
| `size` | The name of the size of the machine, such as `default` |
| `vcpus`, `memory_bytes`, `disk_bytes` | The vCPUs, the RAM in bytes, and the disk in bytes of the size |
| `state` of `hopbox_machine_info` | `running`: the microVM of the machine runs on the node. `saved`: the node keeps the saved state of the machine, and runs no microVM of it. |
| `kind` of `hopbox_saved_state_bytes` | `memory` |

The four counters of pressure, OOM kills, and disk IO start from zero at each boot, because each boot gets a new cgroup. A host can lack their files, and then the node serves none of these series. `cpu.pressure` needs a kernel with PSI on, and a kernel that boots with `psi=0` has none. `io.stat` needs the io controller, which the unit of each machine turns on with `IOAccounting` ([the Firecracker driver](/reference/firecracker-driver/index.html.md#the-unit-of-a-machine)). A machine that booted before this release has no `io.stat` until its next boot. `io.stat` lists each device that the IO of the cgroup reached. A RAID or a device mapper passes each IO on to its member disks, which list it again. So the node reads only the line of the device of the file system of `<data_dir>/machines`, such as `9:3` for `/dev/md3`. A file system without a device of its own, such as btrfs, has no such line, and its disk IO stays at 0. A file that the node cannot read leaves its series out of that scrape.

Some drivers read no use of the machines, such as the test driver ([the VMM driver interface](/reference/vmm/index.html.md#usagereader)). Their nodes still serve `hopbox_machine_info`, `hopbox_machine_vcpus`, and the two series of saved state. A machine with saved state and no microVM has only two series in this section: `hopbox_machine_info` with `state` `saved`, and `hopbox_machine_saved_state_bytes`. A machine that runs before its first spec after a restart of `hopboxd` has no `hopbox_machine_info` and no `hopbox_machine_vcpus` until the spec arrives. A read that fails leaves its series out of that scrape, and the other series stay. For example, a machine can stop between the list of the machines and the read of its cgroup. The slice has no cgroup before the first machine runs in it, so the node totals are absent then.

Each running machine has up to 9 series in this section. A running machine with network also has 8 series in [Metrics of the node network](#metrics-of-the-node-network), and a machine without network has none of them. A machine that restarted also has a series of `hopbox_machine_restarts_total` for each reason ([Metrics of the executors](#metrics-of-the-executors)). A hibernated machine has 2 series. So a node with N running machines, each with network, and H hibernated machines serves about 17 × N + 2 × H + 3 series of the machines. The owner accepted about 17 series for each running machine on 2026-11-01: 1,000 machines give about 17,000 series, which take about 57 MB of RAM in Prometheus (Q129).

```sh
curl -s http://127.0.0.1:9464/metrics | grep -E '^hopbox_(machines?_(info|vcpus|cpu|memory|disk_bytes|saved_state)|saved_state_bytes)'
```

```text
hopbox_machine_cpu_seconds_total{machine="0193a5b2-7c1e-7d4a-9f3b-2a1c4e5f6a7b"} 812.4
hopbox_machine_disk_bytes{machine="0193a5b2-7c1e-7d4a-9f3b-2a1c4e5f6a7b"} 3.81e+09
hopbox_machine_info{disk_bytes="100000000000",machine="0193a5b2-7c1e-7d4a-9f3b-2a1c4e5f6a7b",memory_bytes="8589934592",name="box1",org="0193a5b2-0000-7000-8000-000000000001",size="default",state="running",vcpus="2"} 1
hopbox_machine_info{disk_bytes="100000000000",machine="0193a5b2-7c1e-7d4a-9f3b-2a1c4e5f6a7e",memory_bytes="8589934592",name="box2",org="0193a5b2-0000-7000-8000-000000000001",size="default",state="saved",vcpus="2"} 1
hopbox_machine_memory_bytes{machine="0193a5b2-7c1e-7d4a-9f3b-2a1c4e5f6a7b"} 2.15e+09
hopbox_machine_saved_state_bytes{machine="0193a5b2-7c1e-7d4a-9f3b-2a1c4e5f6a7e"} 2.147483648e+09
hopbox_machine_vcpus{machine="0193a5b2-7c1e-7d4a-9f3b-2a1c4e5f6a7b"} 2
hopbox_machines_cpu_seconds_total 812.4
hopbox_machines_memory_bytes 2.15e+09
hopbox_saved_state_bytes{kind="memory"} 2.147483648e+09
```

## Metrics of the image cache

A server with the node role serves these metrics of its image cache `<data_dir>/images` ([the Firecracker driver](/reference/firecracker-driver/index.html.md#the-image-cache)). The Firecracker driver adds them, so a node with another driver has none.

| Metric | Type | Labels | Description |
| --- | --- | --- | --- |
| `hopbox_image_cache_bytes` | gauge | none | The bytes of the blocks that the files of the image cache hold, read at each scrape. An unpacked disk is sparse, so it takes less than its size. 0 before the first image. |
| `hopbox_image_fetches_total` | counter | `source`, `result` | The images that the node prepared for a boot: `bundle` for an unpack of the compressed disk of the release bundle, and `url` for a download from the URL of the image. `result` is `ok` or `failed`. Each of the 4 series starts at 0. |

A check of a disk that the cache holds already is no fetch. A boot that waits for the fetch of another boot counts no fetch of its own.

```sh
curl -s http://127.0.0.1:9464/metrics | grep '^hopbox_image_'
```

```text
hopbox_image_cache_bytes 2.43e+09
hopbox_image_fetches_total{result="failed",source="bundle"} 0
hopbox_image_fetches_total{result="failed",source="url"} 0
hopbox_image_fetches_total{result="ok",source="bundle"} 1
hopbox_image_fetches_total{result="ok",source="url"} 0
```

## Metrics of the node network

A server with the node role serves the traffic counters of each machine with network ([the node network](/reference/node-network/index.html.md#metrics)). The node reads them from the table `inet hopbox_node` at each scrape. They start from zero at each start of `hopboxd`.

| Metric | Type | Labels | Description |
| --- | --- | --- | --- |
| `hopbox_machine_inbound_bytes_total` | counter | `machine` | Bytes to the machine of connections that the node opened. |
| `hopbox_machine_inbound_packets_total` | counter | `machine` | Packets to the machine of connections that the node opened. |
| `hopbox_machine_egress_bytes_total` | counter | `machine` | Bytes from the machine that the node let out. |
| `hopbox_machine_egress_packets_total` | counter | `machine` | Packets from the machine that the node let out. |
| `hopbox_machine_egress_connections_total` | counter | `machine` | Packets that opened an outbound connection of the machine that the node let out, retries included ([the node network](/reference/node-network/index.html.md#counters)). |
| `hopbox_machine_dropped_packets_total` | counter | `machine`, `reason` | Packets of the machine that the node dropped, by reason: `forged_source` for a source that is not an address of the machine, `blocked_destination` for a blocked range or an IPv6 address outside `2000::/3`, and `port_25` for TCP port 25 while it is closed ([the node network](/reference/node-network/index.html.md#counters)). Each machine with network has the 3 series. |
| `hopbox_node_network_repairs_total` | counter | none | The installs of the table `inet hopbox_node` after it lost a part outside `hopboxd`, or after a check that got no answer within 10 s ([the node network](/reference/node-network/index.html.md#the-table-must-stay)). |

```sh
curl -s http://127.0.0.1:9464/metrics | grep -E '^hopbox_(machine|node_network)_'
```

## Metrics of the egress

Each enrolled server serves the state of its egress ([the egress of the machines](/reference/egress/index.html.md#metrics)). A node without the edge role serves `hopbox_egress_exits`. An edge server serves `hopbox_egress_exit` and `hopbox_egress_uplink_up`.

| Metric | Type | Labels | Description |
| --- | --- | --- | --- |
| `hopbox_egress_exits` | gauge | none | The egress edges whose tunnels the machines of the node use now. At 0, the egress of the machines fails closed. |
| `hopbox_egress_exit` | gauge | none | 1 while the edge server is an egress exit: its tunnel is ready and its uplink works. |
| `hopbox_egress_uplink_up` | gauge | none | 1 while the check of the uplink of the edge server finds that it works. |

```sh
curl -s http://127.0.0.1:9464/metrics | grep '^hopbox_egress_'
```

```text
hopbox_egress_exit 1
hopbox_egress_uplink_up 1
```

## Metrics of the mesh

A server with the mesh device `hbm0` serves these metrics ([the mesh](/reference/mesh/index.html.md#metrics)). The device reads the counters of WireGuard at each scrape. A server outside the mesh serves none of them. The label `server` is the ID of the peer.

| Metric | Type | Labels | Description |
| --- | --- | --- | --- |
| `hopbox_mesh_peers` | gauge | none | The peers of the mesh device. |
| `hopbox_mesh_peer_last_handshake_seconds` | gauge | `server` | The Unix time of the last WireGuard handshake with the peer, or 0 before the first one. |
| `hopbox_mesh_peer_receive_bytes_total` | counter | `server` | The bytes that the mesh device got from the peer. |
| `hopbox_mesh_peer_transmit_bytes_total` | counter | `server` | The bytes that the mesh device sent to the peer. |
| `hopbox_mesh_repairs_total` | counter | none | The changes to the mesh device after it lost a part outside `hopboxd` ([the mesh](/reference/mesh/index.html.md#a-restart-a-stop-and-a-reboot)). |
| `hopbox_mesh_expected_peers` | gauge | none | The peers that the last push of the control role named. It shows after the first push of the process. Compare it with `hopbox_mesh_peers` to find a missing peer. |

```sh
curl -s http://127.0.0.1:9464/metrics | grep '^hopbox_mesh_'
```

## Metrics of the host firewall

Every server on Linux serves these metrics of its table `inet hopbox_host` ([the host firewall](/reference/host-firewall/index.html.md#metrics)).

| Metric | Type | Labels | Description |
| --- | --- | --- | --- |
| `hopbox_host_firewall_repairs_total` | counter | none | The installs of the table `inet hopbox_host` after it lost a part outside `hopboxd`, or after a check that got no answer within 10 s ([the host firewall](/reference/host-firewall/index.html.md#the-table-must-stay)). |
| `hopbox_host_firewall_up` | gauge | none | 1 when the last check found the table whole or installed it. 0 when its install failed. A host without nftables serves neither metric. |

```sh
curl -s http://127.0.0.1:9464/metrics | grep '^hopbox_host_firewall_'
```

## Metrics of the cold moves

A node whose driver keeps its disks in files counts the bytes of the disks that it serves and pulls for the cold moves ([Cold move](/reference/cold-move/index.html.md#metrics)). The pulls also count in `hopbox_executor_actions_total` with the action `pull`.

| Metric | Type | Labels | Description |
| --- | --- | --- | --- |
| `hopbox_disk_transfer_bytes_total` | counter | `direction` | The bytes of disk data of the pulls: `sent` to other nodes, and `received` from another node. |
| `hopbox_disk_transfers` | gauge | `direction` | The pulls that run now: those that the node serves (`sent`), and those that it makes (`received`). |
| `hopbox_disk_transfer_restarts_total` | counter | `reason` | The pulls that dropped their partial copy and start again from zero: `digest_mismatch`, `source_changed`, or `bad_progress` ([Cold move](/reference/cold-move/index.html.md#metrics)). |

Each of the 2 series of `hopbox_disk_transfers` and the 3 series of `hopbox_disk_transfer_restarts_total` starts at 0.

The catalog names the restarts "by digest change or error", so these values of `reason` stand for them:

| Value | Meaning |
| --- | --- |
| `digest_mismatch` | The copy had another digest than the disk of the source. The pull drops it, and the next pull starts from zero. |
| `source_changed` | The disk of the source changed since the pull that left the partial copy. |
| `bad_progress` | The node cannot read the saved state of the digest of the partial copy. |

## node_exporter

node_exporter serves the metrics of the host: CPU, RAM, file systems, disks, network, and pressure. hopbox does not run it. The operator installs the Debian 13 package `prometheus-node-exporter` on each server. The release bundle carries the arguments of node_exporter in `monitoring/node-exporter/prometheus-node-exporter`. They make node_exporter listen on `127.0.0.1:9100` only, as the metrics listener of `hopboxd` does. The default collectors of node_exporter give every host metric that the alerts read: `node_cpu_seconds_total`, `node_memory_MemAvailable_bytes`, `node_filesystem_avail_bytes`, `node_disk_*`, `node_network_*`, and `node_pressure_*`.

The arguments also turn on the systemd collector for the unit `hopboxd.service` alone. It serves the state of the unit in `node_systemd_unit_state{name="hopboxd.service"}`, one series for each state with the value 1 for the current state. It serves the restarts of the unit by systemd in `node_systemd_service_restart_total{name="hopboxd.service"}`. A restart by `systemctl restart` does not count there, because systemd counts only its own restarts after a failure. The collector reads systemd over D-Bus.

```sh
curl -s http://127.0.0.1:9100/metrics | grep -E '^node_systemd_(service_restart_total\{name="hopboxd.service"\}|unit_state\{name="hopboxd.service",state="active")'
```

```text
node_systemd_service_restart_total{name="hopboxd.service"} 0
node_systemd_unit_state{name="hopboxd.service",state="active",type="notify"} 1
```

To set up node_exporter on a server, run these commands as root:

1. Install the package.

   ```sh
   apt-get install prometheus-node-exporter
   ```

2. Copy the arguments of the release over the file of the package.

   ```sh
   install -m 0644 /opt/hopbox/current/monitoring/node-exporter/prometheus-node-exporter /etc/default/prometheus-node-exporter
   ```

3. Restart node_exporter.

   ```sh
   systemctl restart prometheus-node-exporter
   ```

4. Make sure that node_exporter answers on loopback.

   ```sh
   curl -s http://127.0.0.1:9100/metrics | grep '^node_memory_MemAvailable_bytes'
   ```

The host firewall of hopbox blocks inbound traffic on port 9100 ([the host firewall](/reference/host-firewall/index.html.md)). An upgrade of `hopboxd` does not copy the file again, so run step 2 after an upgrade whose file is new.

[Monitor an install with Prometheus and Grafana](/guide/monitor-an-install/index.html.md) sets up node_exporter together with Prometheus, Alertmanager, and Grafana.

## postgres_exporter

postgres_exporter serves the statistics of Postgres (Q132). hopbox does not run it. It is an optional part of the monitoring bundle, from the Debian 13 package `prometheus-postgres-exporter`. The bundle carries its arguments in `monitoring/postgres-exporter/prometheus-postgres-exporter`, and its scrape job in `monitoring/scrape-configs/postgres.yml`. It listens on `127.0.0.1:9187`, and connects with the role `hopbox_monitor` ([the bundled Postgres](/reference/postgres/index.html.md#who-can-reach-the-socket)). D5 reads these families:

| Metric | Source of the value |
| --- | --- |
| `pg_stat_database_numbackends`, `pg_settings_max_connections` | The connections of each database, and the limit of Postgres |
| `pg_stat_database_deadlocks` | The deadlocks of each database |
| `pg_stat_database_blks_hit`, `pg_stat_database_blks_read` | The block reads of each database that found the block in the cache of Postgres, and those that did not |
| `pg_wal_size_bytes` | The size of the write-ahead log |
| `pg_stat_activity_max_tx_duration` | The age of the longest open transaction, by database, state, and role |

The names of `pg_stat_database_deadlocks` and `pg_stat_database_blks_hit` have no `_total`, so Prometheus answers a rate of them with the info `metric might not be a counter`, and Grafana shows a warning icon on the panel. So the row "Postgres" of D5 reads recording rules for these rates ([Recording rules](#recording-rules)).

[Monitor an install with Prometheus and Grafana](/guide/monitor-an-install/index.html.md#add-postgres_exporter) sets it up.

## Recording rules

The monitoring bundle computes these series in Prometheus, in `monitoring/rules/hopbox-recording.yml`. The alerts and the Grafana dashboards read them ([Monitor an install with Prometheus and Grafana](/guide/monitor-an-install/index.html.md)). A name has the form `level:metric:operation`. The level names the labels that stay: `machine`, `server`, `instance`, or `install` for one value of the whole install.

| Recording rule | Value |
| --- | --- |
| `machine:hopbox_machine_cpu_share:rate30m`, `machine:hopbox_machine_cpu_share:rate5m` | The CPU time of each machine per second, divided by `hopbox_machine_vcpus`. 1 is all vCPUs of the machine. |
| `machine:hopbox_machine_egress_bytes:rate5m`, `machine:hopbox_machine_inbound_bytes:rate5m`, `machine:hopbox_machine_egress_connections:rate5m` | The bytes and the new outbound connections of each machine per second |
| `machine:hopbox_machine_cpu_pressure:rate5m`, `machine:hopbox_machine_disk_written_bytes:rate5m` | The share of the time that a process of each machine waited for a CPU, and the bytes that each machine wrote to the disks of the node per second |
| `machine:hopbox_machine_restarts:increase15m` | The restarts of each machine by the restart policy in the last 15 minutes, all reasons together |
| `server:hopbox_server_vcpus:free` | The vCPUs of each node that placement can still give |
| `install:hopbox_nodes:count`, `install:hopbox_server_vcpus:free` | The nodes of the install, and the vCPUs that placement can still give on all of them. They read 0 on an install without a node. While the leader serves no stats, as in maintenance mode, they record no sample, because the count is unknown. |
| `server:hopbox_server_vcpus_allocated:ratio`, `server:hopbox_server_memory_allocated:ratio` | The allocated share of the vCPU capacity and of the RAM of each node |
| `server:hopbox_server_disk_free:ratio`, `server:hopbox_server_disk_allocated:ratio` | The free share of the data dir of each node, and the disk sizes of its machines divided by the size of the data dir |
| `instance:node_cpu_busy:ratio_rate5m`, `instance:node_memory_available:ratio` | The busy share of the CPUs and the available share of the RAM of each host, from node_exporter |
| `install:hopbox_account_creations:increase1h` | The new accounts of the last hour, without the refused sign-ins |
| `install:hopbox_org_creations:increase1h`, `install:hopbox_machine_creations:increase1h` | The audit entries `org.create` and `machine.create` of the last hour |
| `install:hopbox_url_changes:increase1h` | The audit entries `machine.set_url_port` and `machine.set_url_visibility` of the last hour |
| `install:hopbox_api_key_creations:increase1h`, `install:hopbox_long_api_key_creations:increase1h` | The new API keys of the last hour, and those with the lifetime `365d` |
| `install:hopbox_rate_limited:increase1h`, `install:hopbox_limit_refusals:increase1h` | The refusals of the rate limits and of the org limits in the last hour |
| `install:hopbox_placements_no_capacity:increase15m` | The placements with the result `no_capacity` in the last 15 minutes |
| `install:hopbox_edge_public_machine_requests:rate1h` | The requests to public machine URLs per second, over the last hour |
| `install:hopbox_api_failed_calls:increase10m`, `install:hopbox_api_calls:increase10m` | The API calls of the last 10 minutes with the code `internal`, `unavailable`, `unknown`, or `data_loss`, and all API calls. The alert `HopboxAPIErrors` reads them. |
| `install:hopbox_maintenance:max` | 1 while the install is in maintenance mode. The alert `HopboxMaintenanceLong` reads it over 2 hours. |
| `install:pg_stat_database_deadlocks:rate5m`, `install:pg_stat_database_blks_hit:ratio_rate5m` | The deadlocks per second of the database `hopbox`, and the share of its block reads that found the block in the cache of Postgres. They record nothing without postgres_exporter. |
| `install:hopbox_server_versions:count` | The releases of `hopboxd` that the servers run. It counts each value of the label `version` of `hopbox_build_info` and `hopbox_server_info` once. Only a scraped `hopboxd` serves `hopbox_build_info`. An older server and a server that Prometheus does not scrape show through `hopbox_server_info` of the leader. While no leader serves that metric, one scraped release says nothing of the other servers, so the rule records no sample. The alert `HopboxVersionSkew` reads this rule over 24 hours, and skips such a gap. |

The leader serves the gauges of the servers. After a change of the leader, two control processes can serve a series of the same server for a short time. So the `server:` rules keep the largest value of each server.

The counts of the group `hopbox-accounts`, from `install:hopbox_account_creations:increase1h` to `install:hopbox_limit_refusals:increase1h`, read 0 before the first event while `hopbox_leader` shows that a control process answers. Without that process, or with a rule that fails, they record no sample. So a panel reads "No data" when the count is unknown. An alert on these counts fires at the same values as before.

A counter with labels shows only after its first event, so `increase()` misses that event. Each `install:` count adds the value of each series that is new in the window. The count needs a scrape of that `hopboxd` at the start of the window. A new series has no sample in the 5 minutes before the window. So a failed scrape does not make an old series new. After the first install of Prometheus, no series counts as new. So the totals of `hopboxd` from before the first scrape count as no burst. A control process that Prometheus did not scrape at the start of the window loses its first event in that window.

## Metrics of the logs

Every `hopboxd run` counts the log records at level warn and error that it writes (Q133). So the count covers each part of hopbox, also a part without a metric of its own. The code is in `internal/obs`.

| Metric | Type | Labels | Description |
| --- | --- | --- | --- |
| `hopbox_log_messages_total` | counter | `component`, `level` | The log records at level warn and error that this `hopboxd` wrote. |

| Label | Values |
| --- | --- |
| `component` | The value of the attribute `component` of the record, such as `executor`, `netcfg`, or `control`. A record with two values counts with the last one, the one that the JSON record shows last. A `component` inside a group with an empty key counts too, because the JSON record shows it at the top level. `none` for a record without a component. |
| `level` | `warn` or `error` |

A record below the level of `--log-level` is not written, and does not count. A series shows after its first record.

```sh
curl -s http://127.0.0.1:9464/metrics | grep '^hopbox_log_messages_total'
```

```text
hopbox_log_messages_total{component="executor",level="warn"} 2
hopbox_log_messages_total{component="netcfg",level="error"} 1
```

## Metrics of the units

`hopboxd run` does its work in units, and starts them in the order of [hopboxd](/reference/hopboxd/index.html.md#run). A unit that fails stops every unit, so `hopboxd` exits, and systemd starts it again. So the value 0 shows a start that waits or fails, or a stop.

| Metric | Type | Labels | Description |
| --- | --- | --- | --- |
| `hopbox_unit_up` | gauge | `unit` | 1 after the start of the unit returned, until the unit fails or its stop begins. 0 during its start and its stop, and after a failure. A unit that `hopboxd` did not begin to start has no series, so a start that hangs names one unit. |

The value 1 says that the unit started and reported no failure. A part of the work of the unit can still wait, as this table shows. Another metric reports that part.

| `unit` | The value 1 means | What it leaves out |
| --- | --- | --- |
| `metrics` | The metrics listener serves. | |
| `host firewall` | `hopboxd` read and set up the table `inet hopbox_host`, and watches it. | `hopbox_host_firewall_up` says whether the table is whole. |
| `role-data` | `hopboxd` removed the data of the roles that left this server. This unit does its work at its start, and holds nothing after it. | A removal that fails is a warning in the log, and the next start tries again. |
| `postgres` | The bundled Postgres started once, and its supervisor runs. With an external Postgres, `hopboxd` read `postgres.url`. | `hopbox_postgres_up` says whether the bundled Postgres takes connections now. The supervisor restarts Postgres after a crash, and the unit stays at 1. |
| `store` | The pool of the control store is open, and the migrations ended. | `hopbox_store_errors_total` counts the queries that fail. |
| `events` | The reader of the change events runs. | `hopbox_events_lag_seconds` says how far it lags. |
| `leader election` | This control process takes part in the election. | `hopbox_leader` says whether it leads. |
| `api`, `internal listener`, `admin socket` | The listener serves. | |
| `dns`, `edge` | The listeners of the role serve. | |
| `machines` | The node network and the executors run. | |
| `channel` | The loop that keeps the control channel runs. | `hopbox_channel_connected` says whether this server holds its channel. The loop dials again after a loss, and the unit stays at 1. |
| `systemd notify` | `hopboxd` told systemd that it is ready. | |

Maintenance mode stops no unit, so it does not change these values.

```sh
curl -s http://127.0.0.1:9464/metrics | grep '^hopbox_unit_up'
```

```text
hopbox_unit_up{unit="host firewall"} 1
hopbox_unit_up{unit="machines"} 0
hopbox_unit_up{unit="metrics"} 1
hopbox_unit_up{unit="role-data"} 1
```

In this example, the `machines` unit did not end its start, and the later units did not begin theirs.

## Metrics of the process

| Prefix | Source |
| --- | --- |
| `go_` | The Go runtime: goroutines, memory, and garbage collection. |
| `process_` | The process: CPU time, memory, open files, and start time. Linux only. On macOS, these metrics are absent. |

| Metric | Type | Labels | Description |
| --- | --- | --- | --- |
| `hopbox_build_info` | gauge | `version`, `go_version` | Always 1. `version` is the release of this `hopboxd`, the same string that its heartbeats give to `hopbox_server_info`. `go_version` is the Go version of its build. |

```sh
curl -s http://127.0.0.1:9464/metrics | grep '^hopbox_build_info'
```

```text
hopbox_build_info{go_version="go1.27.1",version="v0.9.0"} 1
```

## History of samples

Each control process keeps a history of samples: once a minute, it reads the families below from its own registry, the same one that `/metrics` serves, and writes them to the table `metric_samples` of the control store ([Control store tables](/reference/control-store/index.html.md#metric_samples)). The history keeps 24 hours at 1-minute steps (Q138). The console pages Overview, Capacity, and Machines read it ([Console](/reference/console/index.html.md#admin-area)), so they show trends without Prometheus. The code is in `internal/history`.

| Families | Source | What the history keeps |
| --- | --- | --- |
| `hopbox_accounts`, `hopbox_pending_account_oldest_seconds`, `hopbox_orgs`, `hopbox_api_keys`, `hopbox_public_machine_urls`, `hopbox_machines`, and the 8 `hopbox_server_*` gauges | leader | Every series. |
| `hopbox_account_creations_total`, `hopbox_api_key_creations_total`, `hopbox_rate_limited_total`, `hopbox_limit_refusals_total` | each control process | Every series. |
| `hopbox_audit_entries_total` | each control process | One series for each `action`. The values of `actor_kind` add up. |
| `hopbox_api_calls_total` | each control process | One series for each `listener` and `code`. The values of `procedure` add up. |
| `hopbox_executor_actions_total` | each node | One series for each `result`. The values of `action` add up. |
| `hopbox_node_network_repairs_total`, `hopbox_machine_info`, `hopbox_machine_cpu_seconds_total`, `hopbox_machine_memory_bytes`, `hopbox_machine_disk_bytes`, `hopbox_machine_egress_bytes_total`, `hopbox_machine_egress_connections_total`, `hopbox_machines_cpu_seconds_total`, `hopbox_machines_memory_bytes` | each node | Every series. |
| `hopbox_edge_certificate_not_after_seconds` | each edge | Every series. |
| `hopbox_host_firewall_up` | every `hopboxd` | Every series. |

A reader adds the counters of every instance, as `sum` does in Prometheus. It takes a leader family from one process for each minute, the newest sample with values in that minute, so a change of the leader counts no value twice. An increase over a range adds the increases between the samples of each series within the range. A drop of a counter is a restart, so the value after the drop counts whole. A reader reads the 15 minutes before the range too (`history.ReadMargin`), and a reader at steps of 30 minutes reads one step before the range. A series with a sample before the range began before it. Of the increase from its last sample before the range to its first sample in the range, the reader counts the part after the start of the range, at a steady rate.

A series without a sample before the range is new in it, such as a counter with labels after its first event. Its first value counts whole when its instance took samples before the range, as in the recording rules of the monitoring bundle, so the console and Grafana agree. The instance is the server of the process that took the sample, or the process for a control process without the node role, as a scrape target is in Prometheus. The series of one server go on across a restart of its `hopboxd`, and the drop of their counters counts as a restart. When the history of an instance began within the range, no first value of it counts, so an old total does not count as one burst. A control process without the node role is a new instance after its restart, so it loses the events between its start and its first sample.

A gap in the samples at the start of the range, such as maintenance mode, changes nothing while the gap is shorter than 15 minutes. After a longer gap, the reader counts from the end of the gap, and loses the events of the gap. The reader does not extrapolate to the end of the range, as `increase` in Prometheus does.

The series of a machine merge across the processes of a restart of `hopboxd` and across the nodes of a move, by the label `machine`, and so do the node totals of a server. A series cuts the samples of the series that began before it from its first sample to its last, so the old and the new node of a move never alternate. A move back to an earlier node keeps the samples after the return. The CPU time of a machine comes from its cgroup, which outlives a restart of `hopboxd`, so its series goes on. A share of CPU over 30 minutes or an hour shows only when the samples of the machine cover the whole window. A machine that started 10 minutes ago has a share of 5 minutes only.

The history keeps the series of `hopbox_machine_info` with `state` `saved` too. The machines that run, for the console and for the detection of abuse, leave them out. The history keeps no other family. It drops the labels that multiply the rows of a family, such as `procedure`, so its size grows with the running machines and not with the API calls. Only the families of a process with the control role reach the history. A server with the node role alone writes no sample, so its machines are missing from the Machines page, as from the Prometheus of the monitoring bundle until Q121.
