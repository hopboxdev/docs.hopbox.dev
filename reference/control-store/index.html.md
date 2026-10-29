# Control store tables

> The tables of the control store, leases, leader election, change events, the audit log, idempotency keys, the install config, the tables of accounts, orgs, invitations, usage, sessions, sign-in links, and instance admins, the tables of pools, the install CA, the server registry, and enrollment tokens, the tables of sizes, images, machines, machine names, SSH ports, and placements, and the tables of the DNS zone.

This page lists the tables of the control store and the Go packages that write them. The control store is the Postgres database of the control role. [The bundled Postgres](/reference/postgres/index.html.md) covers the bundled Postgres, the migration runner, and the advisory lock keys.

`hopboxd run` opens the control store for the control role ([hopboxd](/reference/hopboxd/index.html.md#run)). The API writes `audit_log`, `idempotency_keys`, `install_config`, the tables of accounts and sign-in: `accounts`, `account_emails`, `sign_in_methods`, `orgs`, `memberships`, `org_invitations`, `org_invitation_sends`, `sessions`, `magic_links`, `login_links`, `oauth_flows`, `machine_url_sessions`, `instance_admins`, and `admin_notices`, the tables of the fleet: `install_ca`, `enrollment_tokens`, `servers`, and `server_certificates`, the keys of the SSH front door in `ssh_front_door_keys`, and the tables of credentials: `device_authorizations`, `device_approval_failures`, `cli_sign_ins`, `cli_refresh_tokens`, `cli_access_tokens`, `api_keys`, `admin_keys`, and `ssh_keys`. The machine API writes `machines`, `machine_names`, `ssh_ports`, `placements`, and the machine leases in `leases`. The control channels write the reports of the executors to `machines`, `route_records`, and `route_host_keys`. They also write the renewals and the new grants of the machine leases to `leases` and `placements` ([the executor protocol](/reference/executor-protocol/index.html.md)). A change of the zones of the DNS role writes `dns_zone` and publishes an event, and the TXT records go to `dns_txt_records`. It also publishes an event in `events` for each change of a machine ([the machine API](/reference/machine-api/index.html.md#change-events)). The admin API writes `images`. The control channels write the heartbeats to `servers`, the mesh part of the heartbeats to `mesh_members`, and the renewed certificates to `server_certificates`. The leader election writes `leases`. Each control process writes its own row in `control_processes`, and the leader sweeps the rows of processes that ended without their stop. The leader marks silent servers in `servers`. It sweeps `idempotency_keys`, `sessions`, `magic_links`, `login_links`, `oauth_flows`, and the expired codes of `machine_url_sessions`. It also sweeps `org_invitations`, `org_invitation_sends`, `enrollment_tokens`, and `server_certificates`, and the tables of the device flow and the CLI tokens. It frees the reservations of `machine_names` and `ssh_ports`, and drops the rows of deleted machines and their leases. It deletes idle machines, and prunes `events` after 24 hours ([hopboxd](/reference/hopboxd/index.html.md#leader-jobs)). Only the migrations write `pools` and `sizes` so far.

## Rules for every table

- Each migration in `internal/store/migrations/` makes one table or a small group of tables. Migrations only grow.
- A row that changes after its insert has a `version bigint` column. Each update adds 1 to it. A writer can then refuse a row that changed after its read.
- A row that never changes after its insert has no `version` column.
- The Go code makes each ID of type `uuid` with `uuid.NewV7`. The database never makes an ID. A row that a migration inserts, such as the pool `default`, has a fixed ID in the migration. The lease token is not an ID: it is a random UUIDv4 from `uuid.NewV4`.

## Migrations

The control store runs with two releases at the same time: release N and release N-1 ([the bundled Postgres](/reference/postgres/index.html.md#two-releases-on-one-store)). So each migration records the oldest release that runs with it, as the schema version of that release, in `schema_migrations.oldest_schema_version`. A `hopboxd` starts only on a store whose newest migration allows its schema version. A migration of N names the schema version of N-1 when the code of N-1 still works after it. These rules keep that true:

- A migration adds a table, a column with a default or `NULL`, an index, or a row. It does not drop or rename what N-1 reads or writes, and it does not add a constraint that a write of N-1 can break.
- The code of N fills a new column also for the rows that N-1 writes without it, or the column has a default that holds for them.
- A change that N-1 cannot follow takes two releases. The first release adds the new form beside the old one, and writes both. The second release drops the old form, in a migration that names the schema version of the first release.
- A migration without the line `-- oldest_schema_version` runs only with a `hopboxd` that knows it. A running control role of an older release then stops, and a rollback across it stops.

## Connections

`store.Open` opens the connection pool of `hopboxd`. The leader election and the event subscriber open connections of their own, with the same network settings.

| Setting | Value | Reason |
| --- | --- | --- |
| `idle_in_transaction_session_timeout` | 15 s, on pool connections | A client that stops inside a transaction loses its session, and its locks go. One such lock is the `FOR SHARE` lock of a fenced write, which holds back the renewals of a lease. |
| TCP keepalive | first probe after 5 s, then 3 probes 3 s apart | A dead TCP peer is found in about 14 s. |
| `TCP_USER_TIMEOUT` | 15 s, on Linux only | A connection whose sent data stays unacknowledged for 15 s ends. |

`store.DeadPeerTimeout` is 15 s, the same as the leader lease TTL. `store.SetDeadPeerTimeout(cfg, d)` applies the TCP settings for another time `d`: keepalive after `d/3`, 3 probes `d/6` apart, and a user timeout of `d`. Go rounds the keepalive times up to whole seconds, so for 15 s the probes are 3 s apart, not 2.5 s.

The bundled Postgres listens only on a unix socket, which has no keepalive. So the leader election and the event subscriber also put a timeout on each call ([Leader election](#leader-election), [Subscribe](#subscribe)). An external Postgres takes the same three kinds of connections over TCP with TLS, and the TCP settings of the table apply to them ([the bundled Postgres](/reference/postgres/index.html.md#external-postgres)).

Each connection names itself with `application_name`, so `pg_stat_activity` shows which part of `hopboxd` holds it:

| `application_name` | Connection |
| --- | --- |
| `hopboxd` | The connection pool. |
| `hopboxd-leader` | The connection of the leader election. |
| `hopboxd-events` | The connection of the event subscriber. |

## leases

Migration `0002_leases.sql`. One row per lease. The leader lease and the machine leases share this table.

| Column | Type | Description |
| --- | --- | --- |
| `name` | `text`, primary key | The name of the lease: `leader`, or `machine/<machine ID>`. |
| `holder` | `text` | Who holds the lease. For the leader, the host name and the process ID, for example `server1/4242`. For a machine, the server ID of its node. |
| `epoch` | `bigint` | The number of the current grant. It grows with each grant. |
| `token` | `uuid` | A random value of the current grant. |
| `expires_at` | `timestamptz` | The end of the lease. A lease is live while this time is in the future. |
| `version` | `bigint` | The row version. |
| `created_at` | `timestamptz` | The first grant. |
| `updated_at` | `timestamptz` | The last change. |

Each grant takes a new epoch from the sequence `lease_epochs`, while it holds the lock on the row. So epochs only grow, across all leases. Two grants that wait for the same row also get growing epochs. An epoch never comes back on the same database, even after a row is deleted. The epoch of a machine lease is also the execution ID of the machine on its node.

An epoch alone does not name one execution after a failover of Postgres, as the next paragraph explains. So route records, and every other record or message that names an execution, carry the epoch and the token ([the executor protocol](/reference/executor-protocol/index.html.md)).

The epoch and the token together are the fencing token of a grant. After a failover of Postgres to a replica that missed the last grants, the sequence can hand out an epoch again. The token of the new grant still differs, so the fence still tells the two grants apart.

All lease times use the clock of Postgres (`clock_timestamp()`), never the clock of a server.

`CreateMachine` grants the lease of a new machine to the node of its placement, with the TTL `MachineTTL` ([Placement](/reference/placement/index.html.md#the-machine-lease)). The grant is in the transaction of the create. The row of `placements` keeps the epoch and the token of the grant. The heartbeats of the node renew the lease, and grant it again after it expired ([the executor protocol](/reference/executor-protocol/index.html.md#the-machine-lease)). A report of `deleted` from the executor ends the lease. The leader sweep deletes the lease of each machine whose row is gone.

The executor protocol keeps this contract:

- The row of a machine lease always holds the epoch and the token of the current grant. A renewal changes only `expires_at`.
- Each new grant of a machine lease writes a new row of `placements` with its epoch and token, in the same transaction. An example is a new grant after the lease expired. So `placements` always holds the grant that runs the machine now, as its newest row.
- A heartbeat locks the rows of the machines of its server first, in the order of their IDs, `FOR NO KEY UPDATE`. Then it renews the lease rows, and then it locks the server row. That is the order of [orgs](#orgs), in which a placement locks the server row last.
- A report of the executor locks the machine row first, and then fences its lease row `FOR SHARE`. So a report and a heartbeat take their locks in the same order.

### Lease functions

The package `internal/lease` holds these functions. A `lease.Lease` value is one grant: its name, holder, epoch, token, and expiry.

| Function | What it does |
| --- | --- |
| `Grant(name, holder, ttl)` | Gives a lease with a new epoch and a new token. It fails with `ErrHeld` while an earlier grant is live, even for the same holder. |
| `Renew(lease, ttl)` | Moves the expiry to `ttl` from now. For another grant or an expired lease, it fails with `ErrStale`. |
| `RenewAll(leases, ttl)` | Renews many leases in one statement, and returns the leases that it renewed. A lease that is missing from the result is stale. A node heartbeat uses it for the machine leases of the node. |
| `Release(lease)` | Ends a live grant now, so the next grant needs no wait. For a grant that is not live, it fails with `ErrStale`. |
| `Get(name)` | Returns the current grant, live or expired. |
| `Fenced(tx, lease)` | Checks inside a transaction that the grant is the live grant of its lease: the same epoch and the same token. It fails with `ErrStale` otherwise. |

| Value | Default | Source |
| --- | --- | --- |
| `LeaderTTL` | 15 s | A phase 1 default |
| `LeaderRenewEvery` | 5 s | the same |
| `MachineTTL` | 30 s | the same |

### Fenced writes

Some writes are for the holder of a lease only. Such a write calls `Fenced` first, in the same transaction. If `Fenced` fails, the caller must roll back.

`Fenced` locks the lease row with `FOR SHARE` until the transaction ends. A new grant updates the row, so it waits for that transaction. Thus a write of an old grant can never commit after a new grant exists.

The lock also holds back the renewals of the lease. A fenced transaction must stay short and rare. Keep it well under the TTL of the lease, and never wait inside it. Otherwise renewals starve, and the holder loses the lease. The pool ends a transaction that stays idle for 15 s ([Connections](#connections)). A transaction that fences several leases calls `Fenced` in the order of their names. `RenewAll` locks its rows in that order too, so the two never deadlock.

```go
err := pgx.BeginFunc(ctx, pool, func(tx pgx.Tx) error {
	if err := lease.Fenced(ctx, tx, l); err != nil {
		return err
	}
	_, err := tx.Exec(ctx, "UPDATE ...")
	return err
})
```

### Leader election

`lease.Elector` picks one leader among the control-role processes. Each process runs `Run(ctx, lead)`:

1. It opens a connection of its own, outside the pool, with `application_name=hopboxd-leader`. The connection sets `idle_session_timeout` to the lease TTL, and gets the TCP settings of [Connections](#connections) for the TTL. The connect, each try for the lock, and the grant time out after the TTL.
2. It calls `pg_try_advisory_lock(LockKeyLeader)` on that connection every `RetryEvery`, until it gets the lock. Before each try, if `Ready` is set and its channel is not closed, it closes the connection and goes on at step 7.
3. With the lock, it gives the leader lease a new epoch and token. It does not wait for the lease of the previous leader to expire. The lock proves that the previous leader lost its connection, and `Fenced` refuses the writes of the previous leader from then on.
4. It calls `lead` with the lease and a context for the term.
5. Every `RenewEvery`, it renews the lease on the same connection. A renewal must finish a tenth of the TTL before the lease expires. So a renewal that hangs, for example on a silent connection or behind a fenced transaction, ends the term before the lease expires.
6. Three events end the term: the end of `ctx`, the return of `lead`, and a failed renewal. Then `Run` ends the context of `lead`, waits for `lead` to return, and closes the connection. Closing the connection frees the lock. After a clean end, `Run` also releases the lease.
7. Until `ctx` ends, `Run` goes back to step 1. If `Ready` is set, `Run` first waits until its channel is closed, without a connection.

When a renewal fails, the cause of the context of the term is `ErrLeadershipLost`.

A leader process can freeze, for example under a debugger or a long pause of the VM. Its connection then stays open, but it sends nothing. Postgres ends the session after `idle_session_timeout`, which frees the lock at about the time that the lease expires. Another process then takes over.

| Field | Default | Description |
| --- | --- | --- |
| `ConnString` | none | The connection string of the control store. |
| `Holder` | host name and process ID | The holder name in the lease row. |
| `Logger` | `slog.Default()` | The logger. |
| `TTL` | 15 s | The life of the lease after each renewal, and the `idle_session_timeout` of the connection. |
| `RenewEvery` | 5 s | The time between renewals. It must be shorter than `TTL`. |
| `RetryEvery` | 1 s | The time between two tries for the lock. It must be shorter than `TTL`. |
| `Ready` | none | A function that returns a channel that is closed while this process may lead. The control role passes the fence of the process, so a fenced process does not try for the lock. |

The elector writes these logs, with `component=lease` and `holder`:

| Message | Level | Fields |
| --- | --- | --- |
| `became leader` | info | `epoch` |
| `lost leadership` | warn | `epoch`, `err` |
| `stepped down as leader` | info | `epoch` |
| `release the leader lease` | warn | `epoch`, `err` |
| `leader election failed` | warn | `err` |

### Lease errors

| Error | Go value | Cause |
| --- | --- | --- |
| `lease: the lease is held and did not expire: <name>` | `ErrHeld` | `Grant` of a live lease. |
| `lease: the lease epoch is stale or expired: ...` | `ErrStale` | A renewal, a release, or a fenced write with an old epoch, or of an expired lease. |
| `lease: leadership lost: ...` | `ErrLeadershipLost` | The cause of the context of a term that ended after a failed renewal. |
| `lease: RenewEvery <d> and RetryEvery <d> must be shorter than TTL <d>` | | `Run` with bad times. |

## events

Migration `0003_events.sql`. One row per change event. A change event tells the control-role processes that data changed. The rows never change after their insert. Each change of a machine publishes an event, `machine.created` or `machine.updated` ([the machine API](/reference/machine-api/index.html.md#change-events)). Each change of the DNS zones publishes the event `dns.zone_changed`, without an org, a subject, or a payload ([dns_zone](#dns_zone)). Each control process then pushes the zones again on its control channels ([the control channel](/reference/control-channel/index.html.md#dns-zones)). Each change of an input of the ACME job publishes the event `acme.config_changed`, also without an org, a subject, or a payload ([install_config](#install_config)). Each new certificate of the install publishes the event `certificate.changed`, also without an org, a subject, or a payload ([install_certificate](#install_certificate)). Each control process then pushes the certificate to its edge servers ([the control channel](/reference/control-channel/index.html.md#edge-pushes)). Some changes of the access to machines show in no machine event. Each of them publishes the event `edge.access_changed`, also without an org, a subject, or a payload. See [The event edge.access_changed](#the-event-edgeaccess_changed). Each promote of a key-encryption key publishes the event `keyring.kek_promoted`, also without an org, a subject, or a payload ([install_config](#install_config)). Each control process then promotes its staged copy of the key ([Sealing](/reference/sealing/index.html.md#several-control-servers)). Each change of maintenance mode publishes the event `install.maintenance_changed`, also without an org, a subject, or a payload ([install_config](#install_config)). Each control process then reads the flag again, and turns its own maintenance mode on or off ([the API server](/reference/api/index.html.md#maintenance-mode)). The `events` unit of each control process reads them from the head, and feeds the watches and the waits of the machine API ([hopboxd](/reference/hopboxd/index.html.md#run)). Every reader starts from the head, so the leader deletes the events that are older than 24 hours, through `Prune` with `Head()`.

| Column | Type | Description |
| --- | --- | --- |
| `id` | `bigint`, primary key | A number from an identity column. It grows with each insert. |
| `xact_id` | `bigint` | The ID of the transaction that wrote the row, from `pg_current_xact_id()`. |
| `kind` | `text` | The change, as lowercase words joined by dots, for example `machine.created`. |
| `org_id` | `uuid`, can be null | The org of the change. |
| `subject_id` | `uuid`, can be null | The changed object, for example a machine. |
| `payload` | `jsonb` | More data about the change, as a JSON object. The default is `{}`. |
| `created_at` | `timestamptz` | When the transaction wrote the row. |

The ID is a number, not a UUID, because the readers walk the rows in order.

### The event edge.access_changed

These operations publish `edge.access_changed` in their transaction:

| Operation | Why |
| --- | --- |
| `SignOut` of a session ([the sign-in, account, and audit API](/reference/auth-api/index.html.md#signout)) | The delete of the session deletes the tokens of its machine URLs. |
| `DeleteAccount` | The account loses every machine. |
| `RemoveOrgMember` and `LeaveOrg` | The account loses the shared machines of the org. |
| `SuspendOrg` | The machine URLs of the org stop. |
| `RemoveSSHKey` | The open SSH connections of the key end ([the SSH front door](/reference/ssh-front-door/index.html.md#checks-of-an-open-connection)). |

Each control process then tells its edge servers to check each private machine URL and each SSH connection again ([the control channel](/reference/control-channel/index.html.md#edge-pushes)). Each machine event does the same, so a share, an unshare, a delete, and a change of the URL reach the edges too.

### Publish an event

`events.Publish(ctx, tx, event)` writes the row in the transaction of the change. It also calls `pg_notify('hopbox_events', <id>)`. Postgres sends the NOTIFY only at the commit of the transaction, so a rolled-back change sends no event.

```go
err := pgx.BeginFunc(ctx, pool, func(tx pgx.Tx) error {
	// ... the change ...
	_, err := events.Publish(ctx, tx, events.Event{Kind: "machine.created", SubjectID: &machineID})
	return err
})
```

### Read events in order

A reader keeps a cursor: the `xact_id` and the `id` of the last event that it handled. It reads the events after the cursor in the order of `(xact_id, id)`.

A reader reads only the rows of transactions that ended before the oldest open transaction. The ID order alone can skip an event. A transaction can take a small ID, then commit after a reader passed a larger ID. A transaction that ended before the oldest open one can never add a row later, so this rule skips no event. The price is a delay: a long open transaction holds back the events of later transactions until it ends. So keep each write transaction short. Never wait inside one, for example for a machine to reach a state.

| Function | What it does |
| --- | --- |
| `Read(after, limit)` | Returns up to `limit` events after the cursor. `limit` must be 1 or more. It also reports whether events after the cursor wait, most often behind an open transaction. |
| `Head()` | Returns a cursor after every event that a reader can read now. A reader that starts from it gets every later event. It can also get some events that committed just before the call, but it never misses one. |
| `Prune(retention, through)` | Deletes the events that are older than `retention` and not after the cursor `through`. It returns how many it deleted. Pass the oldest cursor of the readers that must not miss an event. If every reader starts from the head, pass `Head()`. The age uses the clock of Postgres. `retention` must be positive. |

### Subscribe

`events.Subscriber` delivers every event after a cursor, in order, and each event once. `Run(ctx, after, handle)` does these steps:

1. It opens a connection of its own, outside the pool, with `application_name=hopboxd-events`, and runs `LISTEN hopbox_events`.
2. It reads and handles every event after the cursor, in batches.
3. It waits for a NOTIFY, for at most `Idle`. If an open transaction holds back events, it waits for at most `Recheck`, because that transaction can end with no NOTIFY.
4. It goes back to step 2.

Each connect, `LISTEN`, and read has a timeout of `Idle`. So a connection that goes silent without a close fails within two `Idle` periods: one wait for a NOTIFY, then one read that times out. The connection also has the TCP settings of [Connections](#connections).

If the connection breaks, `Run` connects again and goes back to step 1. Step 2 then reads the events whose NOTIFY went to the broken connection. The wait before a new connection starts at 100 ms and doubles up to `MaxBackoff`. A read that reaches the end sets the wait back to 100 ms.

The cursor moves past an event only after `handle` returns nil. If `handle` fails, `Run` stops and returns a `*HandlerError` with the event and a cursor. A new `Run` from that cursor gets the failed event first, and no event that the first `Run` handled. When `ctx` ends, `Run` returns the error of `ctx`.

A cursor holds transaction IDs of one Postgres cluster. After a move to another cluster, for example with `pg_dump`, start again from `Head()`.

| Field | Default | Description |
| --- | --- | --- |
| `ConnString` | none | The connection string of the control store. |
| `Logger` | `slog.Default()` | The logger. |
| `BatchSize` | 500 | The most events that one read returns. |
| `Recheck` | 50 ms | The wait before the next read while an open transaction holds back events. |
| `Idle` | 30 s | The longest wait for a NOTIFY before the next read, and the timeout of each connect, `LISTEN`, and read. |
| `MaxBackoff` | 5 s | The longest wait between two connection attempts. |

The subscriber writes the log `events subscriber reconnecting` at level warn, with `component=events`, `err`, and `delay`.

### Event errors

| Error | Go value | Cause |
| --- | --- | --- |
| `events: the kind must be lowercase words joined by dots, such as machine.created: "<kind>"` | `ErrBadKind` | `Publish` with a bad kind. |
| `events: the payload must be a JSON object` | `ErrBadPayload` | `Publish` with a payload that is not a JSON object. |
| `events: read: the limit <n> is out of range` | | `Read` with a limit below 1. |
| `events: handle event <id> (<kind>): <err>` | `*HandlerError` | The handler of `Run` failed. |

## audit_log

Migrations `0004_audit_log.sql` and `0012_audit_log_org_id.sql`. One row per audit entry: who did which action. The rows never change after their insert. Triggers refuse each `UPDATE`, `DELETE`, and `TRUNCATE` on the table with the error `audit_log is append-only, so <operation> is refused`. A later migration that adds a retention for the log must replace them. Org owners and org admins read the rows of their org. Instance admins read every row.

| Column | Type | Description |
| --- | --- | --- |
| `id` | `uuid`, primary key | A UUIDv7. |
| `org_id` | `uuid`, can be null | The org of the action. It is null for an action on the whole install. An action of an instance admin on an org names that org, so that the owners of the org see it. |
| `actor_kind` | `text` | The kind of caller. See the table below. |
| `actor_account_id` | `uuid`, can be null | The account behind the call. |
| `actor_credential_id` | `uuid`, can be null | The key, token, or server that made the call. |
| `action` | `text` | The action, as lowercase words joined by dots, for example `machine.create`. |
| `target_kind` | `text` | The kind of object of the action, for example `machine`. Empty for an action with no object. |
| `target_id` | `text` | The ID or the name of the object. Empty for an action with no object. |
| `details` | `jsonb` | An object of string keys and string values with more facts. The default is `{}`. |
| `request_id` | `text` | The ID of the API request, or empty. |
| `created_at` | `timestamptz` | When the transaction wrote the row. |

| Actor kind | Caller |
| --- | --- |
| `anonymous` | A caller with no credential. |
| `session` | A user signed in to the console. |
| `cli_token` | A user through the hopbox CLI. |
| `member_key` | A member API key. |
| `org_key` | An org API key. |
| `admin_key` | An admin API key. |
| `local_root` | Root on a control-plane server, through the admin socket. |
| `server` | A server of the install, with its certificate. |
| `control` | Another control process of the install, with the certificate of its internal listener. |
| `ssh_key` | An account through the SSH front door, with an SSH key of the account. The audit entry `machine.ssh` has it ([the SSH front door](/reference/ssh-front-door/index.html.md#audit)). |
| `system` | hopbox itself, for example a job of the leader. |

### Record an entry

`audit.Record(ctx, tx, entry)` writes the entry in the transaction of the change. Thus the entry and the change commit together, or neither commits.

```go
err := pgx.BeginFunc(ctx, pool, func(tx pgx.Tx) error {
	// ... the change ...
	_, _, err := audit.Record(ctx, tx, audit.Entry{
		OrgID:      &orgID,
		Actor:      audit.Actor{Kind: audit.ActorSession, AccountID: &accountID},
		Action:     "machine.create",
		TargetKind: "machine",
		TargetID:   machineID.String(),
		Details:    map[string]string{"name": "box1"},
	})
	return err
})
```

The audit log never stores a secret value. `Record` refuses an entry in these cases:

- A detail key is a secret word, or ends with `_` and a secret word, for example `old_value`. Case does not matter. The table after this list holds the secret words.
- A detail value holds a hopbox token prefix and a letter, a digit, `-`, or `_` after it. The prefixes are `hbx_`, `hbxat_`, `hbxrt_`, `hbxadm_`, and `hbxe_`.
- A detail value matches a pattern of the pre-commit secret scan in `.pre-commit-config.yaml`. The scan finds AWS access keys, GitHub tokens, JSON web tokens, Anthropic and OpenAI keys, and Slack tokens. It also finds Google API keys, Stripe live keys, and the start of a PEM block. The two lists must change together.

| Group | Secret words |
| --- | --- |
| values | `value`, `secret`, `password`, `token` |
| keys | `private_key`, `key_material`, `api_key`, `apikey` |
| encrypted data | `plaintext`, `ciphertext` |
| credentials | `credential`, `credentials`, `authorization`, `cookie` |

The value of the detail `fingerprint` skips the patterns of the two value rules when it is an SSH key fingerprint: `SHA256:` and 43 base64 characters. A fingerprint is a hash, and its base64 can hold a token prefix such as `eyJ` by chance. Any other value of that detail goes through the patterns.

### Audit errors

| Error | Go value | Cause |
| --- | --- | --- |
| `audit: invalid entry: ...` | `ErrInvalidEntry` | An unknown actor kind, a bad action, or a target kind without a target ID. |
| `audit: the details may hold a secret value, which the audit log never stores: ...` | `ErrSecretInDetails` | A detail key or value that the rules above refuse. |

## idempotency_keys

Migration `0005_idempotency_keys.sql`. One row per idempotency key. An idempotency key is a request ID that a client sends again on each retry of a call. The server keeps the first result of the call for 24 hours.

| Column | Type | Description |
| --- | --- | --- |
| `scope` | `text`, primary key with `key` | The caller, so that two callers never share a key. |
| `key` | `text`, primary key with `scope` | The idempotency key from the client. |
| `request_hash` | `bytea` | The SHA-256 of the method name and the request body. |
| `response` | `bytea` | The encoded response of the first call. |
| `version` | `bigint` | The row version. |
| `created_at` | `timestamptz` | When the first call claimed the key. |
| `expires_at` | `timestamptz` | 24 hours after `created_at`. After this time, the key is free again. |

### Run an idempotent call

`Store.Do(ctx, request, fn)` runs `fn` once for each key, in one transaction. The transaction does these steps:

1. It claims the key with an insert. If another call with the same key is still open, the insert waits for that call. The wait is at most `Wait`, through the Postgres setting `lock_timeout`.
2. If the key was free or expired, `fn` runs in the same transaction. `fn` returns the encoded response, and `Do` stores it with the key. The change of `fn`, the key, and the response then commit together.
3. If the key has a response, `Do` compares the hash of the request with the stored hash. For the same request, `Do` returns the stored response with `replayed` set to true, and `fn` does not run. For another request, `Do` fails with `ErrKeyReused`.

`fn` must be short and must never wait, for example for a machine to reach a state. Its transaction holds back the change events of every later transaction ([Read events in order](#read-events-in-order)), and a retry waits for it.

If `fn` fails, the transaction rolls back, and no key stays. A retry then runs `fn` again, which is safe because nothing committed. The caller chooses the response bytes, for example the encoded protobuf response of the call.

`Hash(method, body)` returns the request hash. It covers the method name, so the same key on another method is a reuse too.

```go
s := &idempotency.Store{DB: pool}
resp, replayed, err := s.Do(ctx, idempotency.Request{
	Scope: "account:" + accountID.String(),
	Key:   key,
	Hash:  idempotency.Hash("CreateMachine", body),
}, func(ctx context.Context, tx pgx.Tx) ([]byte, error) {
	// ... create the machine in tx, then encode the response ...
	return proto.Marshal(machine)
})
```

| Field | Default | Description |
| --- | --- | --- |
| `DB` | none | A `*pgxpool.Pool` or another value with a `Begin` method. |
| `Wait` | 10 s | How long a call waits for an open call with the same key. Then it fails with `ErrKeyBusy`. Only the claim uses this `lock_timeout`. `fn` runs with the `lock_timeout` of the session. |
| `Retention` | 24 h | How long a key stays. |

A key has 1 to 255 characters, all printable ASCII without spaces. A scope has 1 to 255 bytes.

### Sweep the expired keys

`idempotency.Sweep(ctx, db)` deletes the expired keys in batches of 1000, and returns how many it deleted. A call can take over an expired key while the sweep runs. The sweep then waits for that call, checks the expiry again, and keeps the key. A job of the leader calls it every 10 minutes ([hopboxd](/reference/hopboxd/index.html.md#leader-jobs)). An expired key that the sweep did not delete yet is free already: the next call with it runs again.

### Idempotency errors

| Error | Go value | Cause |
| --- | --- | --- |
| `idempotency: this key was used with a different request` | `ErrKeyReused` | The key has a stored response for another method or body. |
| `idempotency: the key stayed locked for the whole wait, so retry later` | `ErrKeyBusy` | Another transaction held the key for all of `Wait`. Most often it is the first call with the key, but it can also be a sweep. Retry later. |
| `idempotency: invalid key: ...` | `ErrInvalidKey` | A key or a scope that breaks the rules, or a request without a hash. |

## install_config

Migration `0006_install_config.sql`. The settings of the whole install. The table holds exactly one row. The migration inserts it, and the primary key `singleton` refuses a second row. Each later setting of the install gets a new column.

| Column | Type | Default | Description |
| --- | --- | --- | --- |
| `singleton` | `boolean`, primary key | `true` | Always `true`. |
| `domain` | `text` | empty | The domain of the install, for example `example.com`. Empty until an instance admin sets it. It must be lowercase. |
| `smtp_host` | `text` | empty | The host of the SMTP server ([SMTP config](/reference/smtp/index.html.md)). With an empty host and the transport `smtp`, the install sends no mail. |
| `smtp_port` | `integer` | `0` | The TCP port of the SMTP server, 0 to 65535. |
| `smtp_tls` | `text` | empty | `starttls`, `tls`, `none`, or empty without an SMTP host. |
| `smtp_username` | `text` | empty | The SMTP login. |
| `smtp_password` | `bytea`, can be null | null | The SMTP password, sealed with the place install-secret `smtp.password` ([Sealing](/reference/sealing/index.html.md#places)). |
| `smtp_from` | `text` | empty | The sender of the mail. |
| `github_client_id` | `text` | empty | The client ID of the GitHub OAuth app for sign-in ([Sign-in with GitHub and Google](/reference/oauth/index.html.md)). Empty while GitHub is off. |
| `github_client_secret` | `bytea`, can be null | null | The client secret of the GitHub OAuth app, sealed with the place install-secret `oauth.github.client_secret` ([Sealing](/reference/sealing/index.html.md#places)). The check `install_config_github_client` keeps it null exactly while the client ID is empty. |
| `google_client_id` | `text` | empty | The client ID of the Google OAuth client for sign-in. Empty while Google is off. |
| `google_client_secret` | `bytea`, can be null | null | The client secret of the Google OAuth client, sealed with the place install-secret `oauth.google.client_secret`. The check `install_config_google_client` keeps it null exactly while the client ID is empty. |
| `org_max_machines` | `integer` | `3` | The default machine limit of each org ([the org API](/reference/org-api/index.html.md#usage-limits)). 0 or more. |
| `org_max_vcpus` | `integer` | `6` | The default vCPU limit of each org. 0 or more. |
| `org_max_memory_bytes` | `bigint` | `25769803776` (24 GiB) | The default RAM limit of each org, in bytes. 0 or more. |
| `org_max_disk_bytes` | `bigint` | `300000000000` (300 GB) | The default disk limit of each org, in bytes. 0 or more. |
| `max_orgs_per_account` | `integer` | `10` | The most orgs that one account owns besides its personal org ([the org API](/reference/org-api/index.html.md#createorg)). 0 or more. |
| `public_addresses` | `text[]` | `'{}'`, empty | The public addresses of the install: the IPv4 and IPv6 addresses at which the edge servers take traffic, in their order ([the admin API](/reference/admin-api/index.html.md#updateinstallconfig)). The zone of the DNS role points the domain at them ([the DNS role](/reference/dns/index.html.md#the-zone-of-the-install)). Never null. |
| `acme_directory_url` | `text` | `https://acme-v02.api.letsencrypt.org/directory` | The ACME directory of the CA of the certificate of the install ([the certificate of the install](/reference/acme/index.html.md#settings)). It must start with `https://`. |
| `acme_email` | `text` | empty | The contact email of the ACME account. It must be lowercase. |
| `acme_terms_agreed` | `boolean` | `false` | `true` once an instance admin agreed to the terms of the CA of `acme_directory_url`. |
| `acme_terms_url` | `text` | empty | The URL of the terms of the CA that the directory named at the last attempt ([the certificate of the install](/reference/acme/index.html.md#settings)). The ACME job records it at its first attempt after the agreement, and records each new URL. A new agreement, a withdrawn agreement, and a new directory clear it. After a `userActionRequired` of the CA, the job clears it and `acme_terms_agreed`. The writes of the job keep `version` and `updated_at`, so an admin update with `expected_version` still works. |
| `mail_transport` | `text` | `smtp` | How the install delivers its mail: `smtp` or `file` ([Mail](/reference/mail/index.html.md#mail-transports)). A check refuses another value. `file` allows magic links without an SMTP host. |
| `maintenance` | `boolean` | `false` | `true` while the install is in maintenance mode ([the API server](/reference/api/index.html.md#maintenance-mode)). Only `SetMaintenance` changes it ([the admin API](/reference/admin-api/index.html.md#setmaintenance)). It lives in this table, so a copy of the store has it too. Its migration allows schema version 43 ([the bundled Postgres](/reference/postgres/index.html.md#two-releases-on-one-store)): a `hopboxd` of that version runs on the store and ignores the flag. A turn to `true` refuses while a live row of [control_processes](#control_processes) has a schema version below 44, and waits for the other live control processes to report that they ended their work. |
| `promoted_kek` | `text` | empty | The ID of the key-encryption key that the last `PromoteKEK` made current, such as `file:2` ([the admin API](/reference/admin-api/index.html.md#promotekek)). Empty until the first promote. `PromoteKEK` sets it in the transaction that publishes the event `keyring.kek_promoted`. Each control process reads it to promote its own staged copy of that key: at its start, at each event, and at each renewal of its row while its key dir has a staged file or while it misses the key ([Sealing](/reference/sealing/index.html.md#several-control-servers)). It stays set after the promote, and a later promote replaces it. Migration `0045_control_processes.sql` adds it. |
| `version` | `bigint` | `1` | The row version. |
| `created_at` | `timestamptz` | the time of the migration | When the migration made the row. |
| `updated_at` | `timestamptz` | the time of the migration | The last change. |

Migration `0011_smtp_config.sql` adds the `smtp_` columns, `0017_org_limits_and_suspension.sql` adds the `org_max_` columns and `max_orgs_per_account`, `0033_dns.sql` adds `public_addresses`, and `0034_acme.sql` adds the `acme_` columns. Migration `0035_oauth_sign_in.sql` adds the `github_` and `google_` columns, `0040_mail_transport.sql` adds `mail_transport`, `0044_maintenance.sql` adds `maintenance`, and `0045_control_processes.sql` adds `promoted_kek`. `SetDefaultOrgLimits` changes them ([the admin API](/reference/admin-api/index.html.md#setdefaultorglimits)), and adds 1 to the version. The admin API reads and changes the row ([the admin API](/reference/admin-api/index.html.md#installconfigservice)). A rewrap of the SMTP password changes only `smtp_password`, and keeps the version ([Sealing](/reference/sealing/index.html.md#rewrap)). `GetInstallInfo` returns the domain to every caller ([the API server](/reference/api/index.html.md#installservice)). A change of `domain` or `public_addresses` raises the serial in `dns_zone` in the same transaction ([dns_zone](#dns_zone)). The inputs of the ACME job are `domain`, `public_addresses`, and the `acme_` columns. A change of one of them publishes the change event `acme.config_changed` in the same transaction ([the certificate of the install](/reference/acme/index.html.md#settings)). A change of `maintenance` adds 1 to the version, and publishes the change event `install.maintenance_changed` in the same transaction.

## accounts

Migration `0007_accounts.sql`. One row per account: the record of one person in the install. The first sign-in creates it ([Sessions, cookies, and sign-in links](/reference/sessions/index.html.md#what-a-sign-in-does)).

| Column | Type | Description |
| --- | --- | --- |
| `id` | `uuid`, primary key | A UUIDv7. |
| `display_name` | `text` | The name that the person chose ([Domain rules](/reference/domain-rules/index.html.md#display-names)). Empty by default. |
| `version` | `bigint` | The row version. |
| `created_at` | `timestamptz` | The first sign-in. |
| `updated_at` | `timestamptz` | The last change. |

An account can go only after its personal org and its row in `instance_admins` are gone. The foreign keys of those two tables have no cascade, so the delete of an account must remove them on purpose. `DeleteAccount` does so ([the sign-in, account, and audit API](/reference/auth-api/index.html.md#deleteaccount)). The other tables of an account cascade.

## account_emails

Migration `0007_accounts.sql`. One row per email of an account.

| Column | Type | Description |
| --- | --- | --- |
| `id` | `uuid`, primary key | A UUIDv7. |
| `account_id` | `uuid` | The account. A delete of the account deletes the row. |
| `email` | `text` | The email, in lower case. Unique for each account. |
| `verified_at` | `timestamptz`, can be null | The time of the proof of the email by a sign-in method. Null for an email that no method proved, such as the email of a login link. A link from a session sets it, unless another account holds the email as verified ([Sessions, cookies, and sign-in links](/reference/sessions/index.html.md#link-a-new-sign-in-method)). |
| `version` | `bigint` | The row version. |
| `created_at` | `timestamptz` | When the email joined the account. |

The unique index `account_emails_verified` holds each verified email once, so a verified email belongs to at most one account. An unverified email can repeat.

## sign_in_methods

Migration `0007_accounts.sql`. One row per sign-in method of an account.

| Column | Type | Description |
| --- | --- | --- |
| `id` | `uuid`, primary key | A UUIDv7. |
| `account_id` | `uuid` | The account. A delete of the account deletes the row. |
| `kind` | `text` | The kind of sign-in: `email_link`, `login_link`, `github`, or `google`. |
| `subject` | `text` | The identity that the method proves: the email of an email link or a login link, the numeric user ID at GitHub, or the `sub` claim at Google ([Sign-in with GitHub and Google](/reference/oauth/index.html.md#providers)). Unique with `kind`. |
| `email` | `text` | The email that the method names, in lower case. Each sign-in with the method sets it again, so it follows a new email at GitHub or Google. |
| `last_used_at` | `timestamptz` | The last sign-in with the method. |
| `version` | `bigint` | The row version. |
| `created_at` | `timestamptz` | When the method joined the account. |

## orgs

Migrations `0008_orgs.sql` and `0017_org_limits_and_suspension.sql`. One row per org.

| Column | Type | Description |
| --- | --- | --- |
| `id` | `uuid`, primary key | A UUIDv7. |
| `name` | `text` | The name of the org ([Domain rules](/reference/domain-rules/index.html.md#org-names)). A personal org starts with the name `Personal`. |
| `personal_account_id` | `uuid`, can be null, unique | The account of a personal org. Null for any other org. The foreign key has no cascade. |
| `version` | `bigint` | The row version. |
| `created_at` | `timestamptz` | When the org was created. |
| `updated_at` | `timestamptz` | The last change. |
| `max_machines` | `integer`, can be null | The machine limit of this org. Null takes `org_max_machines` of `install_config`. 0 or more. |
| `max_vcpus` | `integer`, can be null | The vCPU limit of this org, or null for the default. |
| `max_memory_bytes` | `bigint`, can be null | The RAM limit of this org, in bytes, or null for the default. |
| `max_disk_bytes` | `bigint`, can be null | The disk limit of this org, in bytes, or null for the default. |
| `suspended_at` | `timestamptz`, can be null | The start of the suspension of the org. Null for an org that is not suspended ([the org API](/reference/org-api/index.html.md#suspension)). |
| `suspend_reason` | `text` | The reason of the suspension, or empty. |

A change can take the last owner from an org. Such a change first locks the org rows with `FOR NO KEY UPDATE`. Then it counts the owners in a new statement, which sees the changes that committed during the wait. `FOR NO KEY UPDATE` leaves the key of the org row free, so another transaction can still write a row that names the org, such as a machine. Only `DeleteOrg` takes `FOR UPDATE`, because it deletes the row.

The lock order is the account row, then the org row, then the rows of the resources in the org, such as machines. Every change follows it:

- The delete of an account locks the row of the account first. That lock holds back every new membership of the account. Then it locks every org in which the account has a role, in the order of their IDs. It locks the orgs where the account is not an owner too, because a promotion during the delete can make it one.
- `RemoveOrgMember` and `LeaveOrg` take the key lock (`FOR KEY SHARE`) of the account that leaves, then the row of their one org. A resource that they delete can name that account.
- `SetOrgMemberRole`, `RemoveOrgMember`, and `LeaveOrg` read the role of the caller again after the org lock ([the org API](/reference/org-api/index.html.md#every-org-keeps-an-owner)).
- `DeleteOrg`, `UpdateOrg`, `CreateInvitation`, `RevokeInvitation`, and `AcceptInvitation` lock the row of the org too, so they wait for such a change. `AcceptInvitation` locks the org row before the invitation row, as `CreateInvitation` does.
- `CreateInvitation` and `AcceptInvitation` write a row that names the account of the caller. So they take the key lock of that account before the org lock.
- `CreateOrg` locks the row of the account, so that the cap `max_orgs_per_account` counts each create.
- `SetVar` writes a row that names the account of the caller. So it takes the key lock of that account before the org lock. `DeleteVar` takes only the org lock ([env_vars](#env_vars)).
- The `OrgResource` methods run while their caller holds these locks ([the org API](/reference/org-api/index.html.md#what-a-member-leaves-behind)). The machine methods lock the machine rows of the org with `FOR UPDATE`, in the order of their IDs.
- A change of a machine outside them follows the same order. It takes the key lock of an account first: the creator account of the machine, or the account of the caller for a create. Then it takes the org row, then the machine row with `FOR UPDATE`. A change that adds to the usage of the org, or to the names that it holds in reservation, takes the org row with `FOR NO KEY UPDATE` (`LockOrg`): `CreateMachine`, `StartMachine`, and `RenameMachine`. So two usage checks, or two checks of the cap of reserved names, in one org never pass together. Every other change of a machine takes `FOR SHARE` (`LockOrgShare`), so changes of different machines of one org run at the same time. Both locks read the suspension of the org, and wait for a suspension in progress.
- A rename locks the rows of the old name and the new name in `machine_names` after the machine row, in the order of the names. So two renames that swap two names never deadlock. A claim of a name in a create takes its row after the org row too.
- A placement locks the row of its server in `servers` last, after the org row ([Placement](/reference/placement/index.html.md#the-steps-of-a-placement)). The start of a stopped machine does the same, after the machine row. So the full order is the account key, the org row, the machine rows, the name rows, then the server row.

So two such changes in one org run one after the other, the org keeps an owner, and no two of them deadlock.

## org_usage

Migration `0017_org_limits_and_suspension.sql`. A view with one row per org: what the org uses of its limits ([the org API](/reference/org-api/index.html.md#usage-limits)). `GetOrgUsage`, the admin `ListOrgs`, and the limit checks read only this view.

| Column | Type | Description |
| --- | --- | --- |
| `org_id` | `uuid` | The org. |
| `machines` | `integer` | The machines of the org whose target state is not `deleted`. |
| `vcpus` | `integer` | The vCPUs of the machines whose target state is `running` or `standby`. |
| `memory_bytes` | `bigint` | The RAM of the machines whose target state is `running` or `standby`, in bytes. |
| `disk_bytes` | `bigint` | The disk of the machines whose target state is not `deleted`, in bytes. |

Migration `0028_machines.sql` replaces the view with `CREATE OR REPLACE VIEW`. The view sums the rows of `machines` of each org, as the table says. A machine counts with the numbers of its size that it copied at its create. An org without machines gets 0 for each column. `CREATE OR REPLACE VIEW` refuses a column of another type, and `sum` of an `integer` gives a `bigint`. So the view casts each sum back to the type of its column, such as `coalesce(sum(m.vcpus), 0)::integer`, and keeps the columns of migration `0017`.

## org_invitations

Migration `0018_org_invitations.sql`. One row per invitation to join an org ([the org API](/reference/org-api/index.html.md#invitations)). The leader deletes a row 24 hours after its end: its use, its revoke, or its expiry.

| Column | Type | Description |
| --- | --- | --- |
| `id` | `uuid`, primary key | A UUIDv7. |
| `org_id` | `uuid` | The org. A delete of the org deletes the row. |
| `email` | `text` | The email that the invitation is for, in lower case. |
| `role` | `text` | `member`, `admin`, or `owner`: the role that the invitation gives. |
| `token_hash` | `bytea`, unique | The SHA-256 of the token in the link. The token itself is never stored. |
| `invited_by` | `uuid`, can be null | The account that made the invitation. A delete of the account sets it to null. |
| `expires_at` | `timestamptz` | 7 days after `created_at`. |
| `accepted_at` | `timestamptz`, can be null | The accept of the invitation. |
| `accepted_by` | `uuid`, can be null | The account that accepted it. A delete of the account sets it to null. |
| `revoked_at` | `timestamptz`, can be null | The revoke of the invitation, by `RevokeInvitation` or by a new invitation for the same email. |
| `version` | `bigint` | The row version. |
| `created_at` | `timestamptz` | The invitation. |

An invitation is open while `accepted_at` and `revoked_at` are null. The unique index `org_invitations_open` holds one open invitation for each email in an org.  An open invitation is live while `expires_at` is in the future.

## org_invitation_sends

Migration `0018_org_invitations.sql`. One row for each new invitation, which the rate limits of `CreateInvitation` count ([the org API](/reference/org-api/index.html.md#invitations)). The rows have no foreign keys. So the delete of an org or an account, which deletes its invitations, keeps these rows, and resets no limit. The leader deletes a row 24 hours after its creation. A row never changes, so it has no `version`.

| Column | Type | Description |
| --- | --- | --- |
| `id` | `uuid`, primary key | A UUIDv7. |
| `org_id` | `uuid` | The org of the invitation. |
| `invited_by` | `uuid` | The account that made the invitation. |
| `email_hash` | `bytea` | The SHA-256 of the email of the invitation, in lower case. The table never holds the email itself. |
| `created_at` | `timestamptz` | The invitation. |

## memberships

Migration `0008_orgs.sql`. One row per account in an org, with its org role. The primary key is `(org_id, account_id)`. A delete of the org or of the account deletes the row.

| Column | Type | Description |
| --- | --- | --- |
| `org_id` | `uuid` | The org. |
| `account_id` | `uuid` | The account. |
| `role` | `text` | `member`, `admin`, or `owner` ([Domain rules](/reference/domain-rules/index.html.md#org-roles)). |
| `version` | `bigint` | The row version. |
| `created_at` | `timestamptz` | When the account joined the org. |
| `updated_at` | `timestamptz` | The last change. |

## sessions

Migration `0009_sign_in.sql`. One row per console session ([Sessions, cookies, and sign-in links](/reference/sessions/index.html.md)).

| Column | Type | Description |
| --- | --- | --- |
| `id` | `uuid`, primary key | A UUIDv7. It is the credential ID of the audit entries of the session. |
| `account_id` | `uuid` | The account. A delete of the account deletes the row. |
| `token_hash` | `bytea`, unique | The SHA-256 of the token in the cookie. The token itself is never stored. |
| `method` | `text` | The kind of sign-in that started the session. |
| `last_seen_at` | `timestamptz` | The last use, recorded at most once per minute. |
| `expires_at` | `timestamptz` | The end of the session, 30 days after `created_at`. |
| `version` | `bigint` | The row version. |
| `created_at` | `timestamptz` | The sign-in. The sign-in is recent while it is less than 10 minutes ago ([Sessions, cookies, and sign-in links](/reference/sessions/index.html.md#recent-sign-in)). |

A session is live while `expires_at` is in the future, and `last_seen_at` is less than 7 days ago.

## machine_url_sessions

Migration `0036_machine_url_sessions.sql`. One row per sign-in of a private machine URL in one browser ([the edge role](/reference/edge/index.html.md#private-machine-urls)). `GET /auth/machine` inserts the row with a one-time code. The edge trades the code once for a token, and the row then holds the token instead of the code. Both are stored only as their SHA-256. A row lives only as long as the console session that made it, so a sign-out also ends the machine URLs of the browser.

| Column | Type | Description |
| --- | --- | --- |
| `id` | `uuid`, primary key | A UUIDv7. |
| `session_id` | `uuid` | The console session that asked for the code. A delete of the session deletes the row. |
| `account_id` | `uuid` | The account of the session. A delete of the account deletes the row. |
| `machine_id` | `uuid` | The machine of the URL. A delete of the machine row deletes the row. |
| `code_hash` | `bytea`, unique, can be null | The SHA-256 of the one-time code until the trade, then null. |
| `code_expires_at` | `timestamptz` | 60 s after the insert. The trade fails after it. |
| `token_hash` | `bytea`, unique, can be null | The SHA-256 of the token in the cookie `__Host-hopbox_machine`, from the trade on. Null before it. |
| `version` | `bigint` | The row version. |
| `created_at` | `timestamptz` | The insert. The limit of 30 codes per session and minute counts it. |
| `updated_at` | `timestamptz` | The trade. |

A check keeps exactly one of `code_hash` and `token_hash` set. One statement trades a code, so two trades of one code cannot both get a token. A trade also deletes the older token rows of the same session and machine, because the new cookie replaces the old one in the browser. A token works while its machine holds its name and its console session lives ([Sessions, cookies, and sign-in links](/reference/sessions/index.html.md)). The leader deletes the rows whose code expired before a trade.

| Index | Columns | Use |
| --- | --- | --- |
| `machine_url_sessions_session` | `(session_id, machine_id)` | The count of the codes of a session, and the delete of the older tokens. |
| `machine_url_sessions_codes` | `code_expires_at`, for the rows without a token | The sweep of the expired codes. |

## magic_links

Migration `0009_sign_in.sql`. One row per magic link ([Sessions, cookies, and sign-in links](/reference/sessions/index.html.md#magic-links)). The leader deletes a row 24 hours after its creation. Until then, the row counts for the rate limits.

| Column | Type | Description |
| --- | --- | --- |
| `id` | `uuid`, primary key | A UUIDv7. |
| `token_hash` | `bytea`, unique | The SHA-256 of the token in the link. |
| `email` | `text` | The email that got the link, in lower case. |
| `requested_ip` | `text` | The IP address of the client that asked for the link, or empty. |
| `expires_at` | `timestamptz` | 15 minutes after `created_at`. |
| `used_at` | `timestamptz`, can be null | The sign-in with the link. One statement checks and sets it, so the link works once. |
| `version` | `bigint` | The row version. |
| `created_at` | `timestamptz` | The request. |
| `link_session_id` | `uuid`, can be null | The session that asked for a link of the email to its account ([Sessions, cookies, and sign-in links](/reference/sessions/index.html.md#link-a-new-sign-in-method)). Null for a sign-in link. Only this session can use the link. Migration `0048_sign_in_links.sql` adds it. It has no foreign key, so the row stays for the rate limits after the session ends. The migration runs with schema version 44 ([the bundled Postgres](/reference/postgres/index.html.md#two-releases-on-one-store)). A `hopboxd` of schema version 44 to 47 writes a null, and ignores the column when it ends a link. So a link that such a `hopboxd` ends works as a magic link of that release. |

## login_links

Migration `0009_sign_in.sql`. One row per first-admin login link ([Sessions, cookies, and sign-in links](/reference/sessions/index.html.md#first-admin-login-links)). The leader deletes a row 24 hours after its creation.

| Column | Type | Description |
| --- | --- | --- |
| `id` | `uuid`, primary key | A UUIDv7. |
| `token_hash` | `bytea`, unique | The SHA-256 of the token in the link. |
| `email` | `text` | The email of the link, in lower case. |
| `expires_at` | `timestamptz` | 15 minutes after `created_at`. |
| `used_at` | `timestamptz`, can be null | The sign-in with the link. |
| `version` | `bigint` | The row version. |
| `created_at` | `timestamptz` | When root made the link. |

## oauth_flows

Migration `0035_oauth_sign_in.sql`. One row per sign-in with GitHub or Google that a browser started ([Sign-in with GitHub and Google](/reference/oauth/index.html.md#start)). The leader deletes a row 24 hours after its creation. Until then, the row counts for the rate limit of starts.

| Column | Type | Description |
| --- | --- | --- |
| `id` | `uuid`, primary key | A UUIDv7. |
| `provider` | `text` | `github` or `google`. |
| `state_hash` | `bytea`, unique | The SHA-256 of the state. The state itself is only in the URL and in the cookie of the browser. |
| `code_verifier` | `text` | The PKCE code verifier. It is of no use without the code, which only the browser gets. |
| `nonce` | `text` | The nonce that the ID token of Google must hold. A flow of GitHub has one too, unused. |
| `return_to` | `text` | The path on the install to which the callback sends the browser. It starts with `/`. |
| `requested_ip` | `text` | The IP address of the client that started the flow, or empty. |
| `expires_at` | `timestamptz` | 10 minutes after `created_at`. |
| `used_at` | `timestamptz`, can be null | The callback of the flow. One statement checks and sets it, so the flow works once. |
| `version` | `bigint` | The row version. |
| `created_at` | `timestamptz` | The start. |
| `link_session_id` | `uuid`, can be null | The session that started a link of the identity to its account ([Sign-in with GitHub and Google](/reference/oauth/index.html.md#link-to-a-signed-in-account)). Null for a sign-in. The callback needs this session. Migration `0048_sign_in_links.sql` adds it, without a foreign key. A `hopboxd` of schema version 44 to 47 ignores the column, so a callback that it ends works as a sign-in of that release. |

## instance_admins

Migration `0010_instance_admins.sql`. One row per instance admin. A row never changes: a grant inserts it, and a revoke deletes it. So it has no `version`.

| Column | Type | Description |
| --- | --- | --- |
| `account_id` | `uuid`, primary key | The account. The foreign key has no cascade. |
| `created_at` | `timestamptz` | The grant. |

A grant, a revoke, and the delete of an account lock every row of the table first. So two revokes at the same time cannot both pass the check that keeps one instance admin.

## admin_notices

Migration `0058_admin_notices.sql`. One row per open notice of an instance admin ([the admin API](/reference/admin-api/index.html.md#notices-to-instance-admins)). The audit entry of the row holds what happened. A dismiss deletes the row, so a row never changes, and the table has no `version`.

| Column | Type | Description |
| --- | --- | --- |
| `account_id` | `uuid` | The instance admin. A delete of the account deletes the row. |
| `audit_entry_id` | `uuid` | The audit entry of the action, a row of `audit_log`. It has no foreign key, because `audit_log` only grows. |
| `created_at` | `timestamptz` | The time of the row. |

The primary key is `(account_id, audit_entry_id)`. The index `admin_notices_audit_entry` lists the rows of an audit entry. The migration has the line `-- oldest_schema_version: 57`. A `hopboxd` of schema version 57 never reads the table, so it runs beside the migration, and its calls write no notice.

## pools

Migration `0013_pools.sql`. One row per pool. A pool is a named group of servers. Each enrollment token enrolls into one pool. The migration inserts the pool `default` with the fixed ID `01a143e7-63a4-7035-bf03-6aa9b8f3b78e`, so every install has it.

| Column | Type | Description |
| --- | --- | --- |
| `id` | `uuid`, primary key | The ID of the pool. |
| `name` | `text` | The name of the pool, a lowercase DNS label, for example `default`. The unique index `pools_live_name` allows one live pool of each name. |
| `version` | `bigint` | The row version. |
| `created_at` | `timestamptz` | When the pool was made. |
| `updated_at` | `timestamptz` | The last change. |
| `ovh_plan` | `text` | Not used. Migration `0042_pools_and_operator_sizes.sql` added it for the plan of a provider, with a check of 1 to 64 of `a-z`, `0-9`, `.`, and `-` that start with a letter or a digit. Since N30, no code writes or reads it, and a new pool gets the default, the empty text. The column stays, because a release of schema version 44 still writes it (N30). |
| `hardware` | `jsonb`, null | The hardware of the first server that enrolled into the pool: `cpu_model`, `cpu_threads`, `memory_bytes`, and `disk_bytes`, a list of the sizes of its whole disks. Null until that enrollment. A pool whose servers are all decommissioned takes the hardware of its next server. Migration `0042`. |
| `deleted_at` | `timestamptz`, null | The delete of the pool. A deleted pool keeps its row, because its decommissioned servers and its retired sizes still name it. A check keeps the pool `default` from a delete. Migration `0042`. |

A pool is live while `deleted_at` is null. A delete frees the name, so a new pool can take it. The admin API adds and deletes pools ([the admin API](/reference/admin-api/index.html.md#poolservice)), and enrollment records the hardware ([Enrollment and the install PKI](/reference/enrollment/index.html.md)).

## install_ca

Migration `0014_install_ca.sql`. The install CA, which signs the certificates of the servers and of the internal listener ([Enrollment and the install PKI](/reference/enrollment/index.html.md)). The table holds at most one row. The primary key `singleton` refuses a second one.

| Column | Type | Description |
| --- | --- | --- |
| `singleton` | `boolean`, primary key | Always `true`. |
| `certificate` | `bytea` | The DER certificate of the CA: ECDSA P-256, valid for 10 years. |
| `sealed_key` | `bytea` | The PKCS #8 private key of the CA, sealed with the place `InstallSecret{Name: "install_ca.key"}` ([Sealing](/reference/sealing/index.html.md#places)). The database alone never opens it. |
| `cert_hash` | `text` | The SHA-256 of `certificate`, in lowercase hex. Each enrollment token holds it. |
| `version` | `bigint` | The row version. |
| `created_at` | `timestamptz` | When the CA was made. |
| `updated_at` | `timestamptz` | The last change. |

The first control process that needs the CA makes it: the first `CreateEnrollmentToken`, or the start of the internal listener. It seals the key with the current key-encryption key, so the install needs one first. Two processes can make a CA at the same time. The insert of the second one does nothing, and both read the first one. Each process keeps the CA in memory after its first read, and checks that the hash of the certificate matches `cert_hash`.

## ssh_front_door_keys

Migration `0038_ssh_front_door.sql`. The keys of the SSH front door ([the SSH front door](/reference/ssh-front-door/index.html.md#keys)). The table holds at most one row. The primary key `singleton` refuses a second one.

| Column | Type | Description |
| --- | --- | --- |
| `singleton` | `boolean`, primary key | Always `true`. |
| `host_public_key` | `text` | The public host key in the `authorized_keys` format, such as `ssh-ed25519 AAAA...`. `GetInstallInfo` returns it. |
| `sealed_host_key` | `bytea` | The PKCS #8 Ed25519 host key, sealed with the place `InstallSecret{Name: "ssh_host.key"}` ([Sealing](/reference/sealing/index.html.md#places)). |
| `client_public_key` | `text` | The public client key in the `authorized_keys` format. Each machine spec carries it. |
| `sealed_client_key` | `bytea` | The PKCS #8 Ed25519 client key, sealed with the place `InstallSecret{Name: "ssh_client.key"}`. |
| `version` | `bigint` | The row version. |
| `created_at` | `timestamptz` | When the keys were made. |
| `updated_at` | `timestamptz` | The last change. A rewrap changes only a sealed key. |

The start of the internal listener of the first control process with a key-encryption key makes the keys. If the keys do not exist yet, a read of the machine specs or of the keys for an edge server makes them too. Two processes can make keys at the same time. The insert of the second one does nothing, and both read the first one. Each process keeps the public keys in memory after its first read.

## acme_account

Migration `0034_acme.sql`. The ACME account of the install at its CA ([the certificate of the install](/reference/acme/index.html.md#where-the-certificate-lives)). The table holds at most one row. The ACME job inserts it at its first attempt, and replaces it after a change of the directory.

| Column | Type | Description |
| --- | --- | --- |
| `singleton` | `boolean`, primary key | Always `true`. |
| `directory_url` | `text` | The ACME directory of the CA of the account. Not empty. |
| `email` | `text` | The contact email of the account, or empty. |
| `account_url` | `text` | The URL of the account that the CA gave. Each request of the job names it. Not empty. |
| `sealed_key` | `bytea` | The PKCS #8 key of the account, sealed with the place `InstallSecret{Name: "acme_account.key"}` ([Sealing](/reference/sealing/index.html.md#places)). |
| `version` | `bigint` | The row version. |
| `created_at` | `timestamptz` | When the job stored the first account. |
| `updated_at` | `timestamptz` | The last change. A rewrap changes only `sealed_key`, and keeps the version and this time. |

## install_certificate

Migration `0034_acme.sql`. The wildcard certificate of the install ([the certificate of the install](/reference/acme/index.html.md#where-the-certificate-lives)). The table holds at most one row. A new certificate replaces the row in one transaction, only after its checks pass. So a renewal that fails keeps the certificate that works.

| Column | Type | Description |
| --- | --- | --- |
| `singleton` | `boolean`, primary key | Always `true`. |
| `domain` | `text` | The domain of the install that the certificate covers. Not empty. |
| `names` | `text[]` | The DNS names of the certificate: the domain and its wildcard. |
| `chain_pem` | `text` | The leaf, then the intermediates, in PEM. Not empty. |
| `sealed_key` | `bytea` | The PKCS #8 key of the certificate, sealed with the place `InstallSecret{Name: "install_certificate.key"}` ([Sealing](/reference/sealing/index.html.md#places)). |
| `serial` | `text` | The serial number of the leaf, in lowercase hex. |
| `issuer` | `text` | The subject of the CA certificate that signed the leaf, such as `CN=R12,O=Let's Encrypt,C=US`. |
| `not_before` | `timestamptz` | The start of the validity of the leaf. |
| `not_after` | `timestamptz` | The end of the validity of the leaf. Later than `not_before`. |
| `directory_url` | `text` | The ACME directory that issued the certificate. |
| `version` | `bigint` | The row version. |
| `created_at` | `timestamptz` | When the job stored the first certificate. |
| `updated_at` | `timestamptz` | When the last certificate replaced the one before it. A rewrap changes only `sealed_key`, and keeps the version and this time. |

The transaction of a new certificate also resets `acme_status`, and publishes the change event `certificate.changed` ([events](#events)). A change of an input of the ACME job in `install_config` sets `failures` to 0, in the transaction of the change. It also clears `next_attempt_at`, unless `ca_wait` is `true`. A change of `acme_directory_url` clears `next_attempt_at` and `ca_wait` in every case. Each change keeps `last_error`.

## acme_status

Migration `0034_acme.sql`. The state of the attempts of the ACME job ([the certificate of the install](/reference/acme/index.html.md#renewal-and-backoff)). The table holds exactly one row. The migration inserts it.

| Column | Type | Default | Description |
| --- | --- | --- | --- |
| `singleton` | `boolean`, primary key | `true` | Always `true`. |
| `failures` | `integer` | `0` | The attempts that failed since the last certificate. 0 or more. |
| `last_attempt_at` | `timestamptz`, can be null | null | The start of the last attempt. |
| `last_error` | `text` | empty | The error of the last attempt that failed, at most 1024 bytes. Empty after a certificate. |
| `next_attempt_at` | `timestamptz`, can be null | null | After a failure, the earliest time of the next attempt. Null after a certificate. |
| `action_required` | `boolean` | `false` | `true` after the ACME job cleared the agreement on a `userActionRequired` of the CA. A change of `acme_terms_agreed` or of `acme_directory_url` clears it. |
| `ca_wait` | `boolean` | `false` | If the CA asked for the wait until `next_attempt_at`, with a rate limit or a header `Retry-After`, it is `true`. |
| `version` | `bigint` | `1` | The row version. |
| `updated_at` | `timestamptz` | the time of the migration | The last change. |

## servers

Migrations `0015_servers.sql` and `0030_server_labels.sql`, and these later migrations:

- `0045_control_processes.sql` adds the column `channel_process`.
- `0050_machine_moves.sql` adds the column `disk_transfer_version`.
- `0051_server_drains.sql` adds the states `draining` and `shutting_down`.
- `0052_dns_zone_serial.sql` adds the column `dns_zone_serial`.
- `0053_egress_exit.sql` adds the column `egress_exit_version`.
- `0054_peer_addresses.sql` adds the column `peer_address`.
- `0056_role_removal.sql` adds the reports of the heartbeats, the removal of a role, and `removed_roles`.

The server registry: one row per enrolled server. Enrollment inserts the row, and each heartbeat on the control channel writes the facts of the server ([the control channel](/reference/control-channel/index.html.md)).

| Column | Type | Default | Description |
| --- | --- | --- | --- |
| `id` | `uuid`, primary key | none | The ID of the server. Its certificate names it. |
| `pool_id` | `uuid` | none | The pool of the server, a row of `pools`. |
| `hostname` | `text` | empty | The host name that the server sent at its enrollment. |
| `roles` | `text[]` | none | The roles of the server, at least one, for example `{node}`. |
| `state` | `text` | none | `ready`, `cordoned`, `unresponsive`, `draining`, `shutting_down`, or `decommissioned`. |
| `cordoned` | `boolean` | `false` | The choice of an instance admin. It stays while the server is unresponsive. |
| `status` | `text` | empty | The health of the last heartbeat: `ok` or `degraded`. Empty before the first heartbeat. |
| `problems` | `text[]` | empty | The failed checks of the last heartbeat. |
| `labels` | `jsonb` | `{}` | The labels of the server, an object of strings. Each heartbeat writes them, until `SetServerLabels` sets them. |
| `labels_managed` | `boolean` | `false` | True once `SetServerLabels` set the labels. A heartbeat then keeps `labels` as they are. The admin API calls it `labels_set_by_admin`. |
| `hopboxd_version` | `text` | empty | The version of `hopboxd` on the server. |
| `kernel_release` | `text` | empty | The release of the host kernel. |
| `cpu` | `jsonb` | `{}` | The CPU fingerprint: `arch`, `vendor`, `family`, `model`, `stepping`, `model_name`, `microcode`, `flags_hash`, and `cpu_template`. |
| `compatibility_class` | `text` | empty | The compatibility class of the server. |
| `cpu_threads` | `integer` | `0` | The count of CPU threads. |
| `memory_bytes` | `bigint` | `0` | The size of the memory. |
| `disk_bytes` | `bigint` | `0` | The size of the file system of the data dir. |
| `disk_free_bytes` | `bigint` | `0` | The free space on that file system. |
| `allocations` | `jsonb` | `[]` | The resources that the machines of the server hold: objects with `machine_id`, `vcpus`, `memory_bytes`, and `disk_bytes`. |
| `actions` | `jsonb` | `[]` | The actions in progress: objects with `machine_id`, `action`, and `started_at`. |
| `disk_transfer_version` | `text` | empty | The `hopboxd_version` of the last heartbeat that said `disk_transfer`. Each other heartbeat of a release that knows the field sets it empty. A node is the source or the target of a cold move only while it equals `hopboxd_version` ([Cold move](/reference/cold-move/index.html.md#two-releases)). |
| `last_heartbeat_at` | `timestamptz` | null | The time of the last heartbeat. Null before the first one. |
| `channel_process` | `uuid` | null | The control process that holds the control channel of the server, a row of `control_processes` ([control_processes](#control_processes)). Each recorded heartbeat of a channel that its process still holds sets it. Other heartbeats leave it as it is. The hub clears it when the last channel of the server in that process ends, if it still names that process. Another control process reads it to forward a data stream ([the control channel](/reference/control-channel/index.html.md#streams-through-another-control-process)). Migration `0045_control_processes.sql` adds it. Null before the first heartbeat on a control process of a release with the migration. |
| `dns_zone_serial` | `bigint` | null | The serial of the zones that the DNS role of the server served at its last heartbeat, from 0 to 4294967295 ([the DNS role](/reference/dns/index.html.md#the-serial-of-each-dns-server)). Each heartbeat sets it, and a heartbeat without it sets null. Migration `0052_dns_zone_serial.sql` adds it, with `oldest_schema_version` 44. A control process of schema version 44 to 51 never reads it, and its heartbeats leave it as it is. So while such a process holds the channel of a server, the column keeps the serial of an older heartbeat. |
| `egress_exit_version` | `text` | empty | The `hopboxd_version` of the last heartbeat that said `egress_exit` ([the control channel](/reference/control-channel/index.html.md#heartbeat)). Each other heartbeat of a release that knows the field sets it empty. An egress edge has this value equal to its `hopboxd_version`, and the state `ready`, `cordoned`, or `draining` ([the egress of the machines](/reference/egress/index.html.md#egress-edges)). The read of the mesh gives this to each peer as `egress` ([mesh_members](#mesh_members)). Migration `0053_egress_exit.sql` adds it, with `oldest_schema_version` 44, as `0050_machine_moves.sql` does for `disk_transfer_version`. A `hopboxd` of schema version 44 to 52 never reads it, and its heartbeats leave it as it is. |
| `peer_address` | `inet` | null | The peer address of the server: the public IPv4 address at which the other servers reach it. The enrollment sets it from the token, and `SetServerPeerAddress` sets it. A heartbeat never changes it ([Enrollment and the install PKI](/reference/enrollment/index.html.md#peer-addresses)). It is the endpoint of the server in the mesh ([the mesh](/reference/mesh/index.html.md#the-peers)). Null for a server without one, which stays out of the mesh. Migration `0054_peer_addresses.sql` adds it, with `oldest_schema_version` 44. A `hopboxd` of schema version 44 to 53 never reads it. |
| `features` | `text[]` | empty | The features of the `hopboxd` of the last heartbeat that reported them, such as `remove-role` ([the control channel](/reference/control-channel/index.html.md#heartbeat)). |
| `addresses` | `inet[]` | empty | The public addresses on the interfaces of the server, from the same heartbeat. |
| `floating_addresses` | `inet[]` | empty | The floating addresses of the bootstrap config of an edge server, from the same heartbeat. |
| `reports_version` | `text` | empty | The `hopboxd_version` of the heartbeat that wrote `features`, `addresses`, and `floating_addresses`. A heartbeat of a release that knows the fields writes all four. A check of these fields takes them only while this value equals `hopboxd_version`. So after a rollback, the reports of a newer release never stand for an older one. |
| `removal_roles` | `text[]`, can be null | null | The roles of the pending removal of the server, such as `{edge,dns}`. Null without a removal. |
| `removal_cordoned_before` | `boolean`, can be null | null | True when the server was cordoned before its pending removal started. The end and the cancel of the removal lift only a cordon that the removal set. Null without a removal. |
| `removal_started_at` | `timestamptz`, can be null | null | The start of the pending removal. A move that failed before it strands no machine. Null without a removal. |
| `removal_error` | `text` | empty | Why the pending removal waits now, such as a machine that has not booted yet. Empty otherwise. |
| `removed_roles` | `text[]` | empty | The roles that a removal took from the server. A token adds a role to the enrolled server only when this list holds the role, and the add takes the role out of it. The trigger `servers_role_add` holds this check for each release ([removal_strands](#removal_strands)). |
| `enrolled_at` | `timestamptz` | the time of the insert | When the server enrolled. |
| `version` | `bigint` | `1` | The row version. Each heartbeat adds 1. |
| `created_at` | `timestamptz` | the time of the insert | When the row was made. |
| `updated_at` | `timestamptz` | the time of the insert | The last change. |

A check refuses the state `cordoned` without the flag `cordoned`. The check `servers_peer_address_public` keeps `peer_address` null or one public unicast IPv4 address, with a prefix length of 32. It refuses an address in `0.0.0.0/8`, `10.0.0.0/8` (which holds the mesh range and the machine range), `100.64.0.0/10`, `127.0.0.0/8`, `169.254.0.0/16`, `172.16.0.0/12`, `192.168.0.0/16`, and `224.0.0.0/3` (multicast, the reserved range, and the broadcast address). The unique partial index `servers_peer_address` keeps each peer address on one server that is not decommissioned. The query `ServerOfPeerAddress` reads that server. The query `FirewallPeers` reads the peer address and the roles of each server that is not decommissioned, for the sets `peers` and `jump_hosts` of the host firewall. `SetServerPeerAddress` and the enrollment take the lock of the server registry before they check it. The partial index `servers_heartbeat` on `last_heartbeat_at` covers the `ready`, `cordoned`, `draining`, and `shutting_down` servers, for the leader job.

The state changes in these steps:

| Event | From | To |
| --- | --- | --- |
| Enrollment | none | `unresponsive`. A server is `ready` only once a heartbeat arrived. |
| A heartbeat | `unresponsive` | `draining` if the server has a row in `server_drains`, else `cordoned` if the flag `cordoned` is set, else `ready` |
| A heartbeat | `ready`, `cordoned`, `draining`, or `shutting_down` | no change |
| `CordonServer` | `ready` | `cordoned`. The flag is set in every state but `decommissioned`. |
| `UncordonServer` | `cordoned` | `ready`. The flag is cleared in every state but `decommissioned`. It refuses a server with a row in `server_drains`. |
| `DrainServer` | `ready` or `cordoned` | `draining`, with the flag set. An `unresponsive` server keeps its state with the flag set. |
| The drain, the first empty check | `draining` | `shutting_down` |
| The drain, a check that finds the server not empty, or `DrainServer` again | `shutting_down` | `draining` |
| `DecommissionServer`, and the third empty check of the drain | `ready`, `cordoned`, `unresponsive`, `draining`, or `shutting_down` | `decommissioned`, for good. It refuses a server with a row of `machines` whose `current_state` is not `deleted`, or whose `allocations` are not empty, a server that a move runs from or to, and a server that keeps a copy of a disk after a failed move. It deletes the row of `server_drains`, and clears `cleanup_server_id` of the moves that name the server. It refuses a server that shares its host with the last live control process while another server is not `decommissioned` ([control_processes](#control_processes)). |
| The leader job, 20 s after the last heartbeat | `ready`, `cordoned`, `draining`, or `shutting_down` | `unresponsive` |

A heartbeat of a `decommissioned` server changes nothing. No call moves a server out of `decommissioned`. `DecommissionServer` writes the audit entry `server.decommission` with the details `old_state` and `state`. The third empty check of a drain writes it with the actor `system` and the details `old_state` `shutting_down`, `state` `decommissioned`, and `drain` `true`. `DrainServer` writes `server.drain` with the details `old_state`, `state`, and `retry`. The first empty check of a drain writes `server.mark_shutting_down` with the actor `system` and the details `old_state`, `state`, and `checks`. A check that finds the server not empty again writes `server.mark_draining` with the actor `system` and the details `old_state`, `state`, and `reason`. `SetServerLabels` writes `server.set_labels` with the details `old_labels` and `labels`. `SetServerPeerAddress` writes `server.set_peer_address` with the details `old_peer_address` and `peer_address`. The heartbeat that ends an `unresponsive` spell writes the audit entry `server.heartbeat` with the details `old_state` and `state`. The leader job writes the audit entry `server.mark_unresponsive` with the actor `system` and the detail `last_heartbeat_at`, and its write checks the leader lease ([Fenced writes](#fenced-writes)).

## control_processes

Migration `0045_control_processes.sql`, with `-- oldest_schema_version: 44`. One row per process of the control role. The control role never enrolls, so `servers` does not know it. Each control process records itself here, so the checks that span the control processes can see the others. These checks read the live rows. They are the migration runner ([the bundled Postgres](/reference/postgres/index.html.md#two-releases-on-one-store)), a turn of maintenance mode to on ([the admin API](/reference/admin-api/index.html.md#setmaintenance)), the decommission of the last control server ([the admin API](/reference/admin-api/index.html.md#decommissionserver)), and the rolling upgrade of the Terraform provider ([hopbox Terraform provider](/reference/terraform-provider/index.html.md#rolling-upgrade)). `hopboxd admin control list` shows the rows ([hopboxd](/reference/hopboxd/index.html.md#admin-control-list)).

| Column | Type | Default | Description |
| --- | --- | --- | --- |
| `id` | `uuid`, primary key | none | A random ID. Each start of `hopboxd run` makes a new one. |
| `server_id` | `uuid`, can be null | null | The ID of the enrolled server of the same `hopboxd`. Null when no role of that `hopboxd` enrolled. |
| `hostname` | `text` | none | The host name of the process. |
| `internal_url` | `text` | none | `https://<ip>:<port>` of the internal listener of the process, at which the other control processes reach it. The value is empty if the listener listens on every address or on a loopback address, which no other server reaches. |
| `release` | `text` | none | The version of `hopboxd` of the process. |
| `schema_version` | `integer` | none | The number of the newest migration that the `hopboxd` of the process knows. More than 0. |
| `kek_ids` | `text[]` | `'{}'`, empty | The IDs of the key files that the process can open, the `.key` files first and then the staged files ([Sealing](/reference/sealing/index.html.md#several-control-servers)). A staged file whose key differs from the recorded fingerprint of its ID is left out ([Sealing](/reference/sealing/index.html.md#key-fingerprints)). A promote refuses while a live process does not list the key. |
| `kek_current` | `text` | empty | The ID of the KEK with which the process seals: the highest `.key` file. A promote waits until every live process reports the new key here. |
| `kek_fingerprints` | `text[]` | `'{}'`, empty | The fingerprint of each key file of `kek_ids`, as `<key ID>=<fingerprint>` ([Sealing](/reference/sealing/index.html.md#key-fingerprints)). A promote and a rewrap compare them. Migration `0046_kek_fingerprints.sql` adds it. |
| `maintenance` | `boolean` | `false` | The maintenance mode that the process applies. |
| `maintenance_version` | `bigint` | `0` | The version of `install_config` at which the process read maintenance mode. A turn to on waits until every other live process reports `maintenance`, `drained`, and a `maintenance_version` at least the version that the turn wrote. Migration `0046_kek_fingerprints.sql` adds it. |
| `drained` | `boolean` | `false` | `true` in maintenance mode once no call, control channel, or leader job of the process runs. |
| `started_at` | `timestamptz` | the time of the insert | When the process started. |
| `renewed_at` | `timestamptz` | the time of the insert | The last renewal. |
| `version` | `bigint` | `1` | The row version. Each renewal adds 1. |
| `stopped_at` | `timestamptz`, can be null | null | The clean stop of the process. A row with it is never live. Migration `0047_control_process_stops.sql` adds it, with `-- oldest_schema_version: 44`. |

The index `control_processes_renewed_at` on `renewed_at` serves the live reads and the sweep.

The `store` unit of `hopboxd run` writes the row ([hopboxd](/reference/hopboxd/index.html.md#run)). The migration runner of the control role inserts the first row while it still holds the migration lock, right after the migrations ([the bundled Postgres](/reference/postgres/index.html.md#schema-migrations)). The full row, with the keys, maintenance mode, and the server ID, follows at the end of the unit. The unit renews the row on a fixed ticker every 5 s, and each try takes at most 2 s. It also renews at once after a turn of maintenance mode, after the internal listener starts, at a change event of a promote of a KEK, and once at the end of a drain. At its stop, before it closes its connections, the unit sets `stopped_at` of the row. A stopped row is not live from then on, and stays until the sweep, so a rotation of a KEK still sees the control server ([Sealing](/reference/sealing/index.html.md#several-control-servers)). A row is live for 20 s (`store.ControlProcessLive`) after its last renewal, so it stays live through three periods and a slow try. A process that ends without its stop, for example after a crash, leaves its row. The row is not live after 20 s. The sweep of the leader deletes it 24 h after its last renewal ([hopboxd](/reference/hopboxd/index.html.md#leader-jobs)).

Each renewal is one transaction (`store.RecordControlProcess`). If the row is gone, stopped, or has no more than 5 s left to live (`store.ControlProcessFenceMargin`), it first takes the migration lock `LockKeyMigrations` shared (`pg_advisory_xact_lock_shared`), so a migration runner that checks the live rows never misses a process that comes back ([the bundled Postgres](/reference/postgres/index.html.md#advisory-lock-keys)). Then it reads `schema_migrations`, and fails with the error `store: the database has a migration that does not run with this hopboxd: ...` when the store does not run with this `hopboxd`. Such a failure fences the process and stops `hopboxd` at once. Otherwise it writes the row. So a renewal that works means that the row is live and that the store runs with this `hopboxd`.

When no renewal worked for 15 s, counted from the start of the last renewal that worked, the process fences itself before its row stops being live. It takes no call, serves no control channel, and runs no leader job, until a renewal works ([hopboxd](/reference/hopboxd/index.html.md#run)). A leader that stays fenced for one lease TTL gives up its lease, so another process can run the leader jobs ([hopboxd](/reference/hopboxd/index.html.md#leader-jobs)).

A control process of a release before migration `0045` writes no row, so none of the checks sees it: the migration guard, the maintenance wait, the KEK promote, the decommission check, and the rolling upgrade order. Upgrade every control server before you rely on them. If a migration does not run with such a process, the schema watch still stops it within 10 s ([the bundled Postgres](/reference/postgres/index.html.md#two-releases-on-one-store)).

## kek_fingerprints

Migration `0046_kek_fingerprints.sql`, with `-- oldest_schema_version: 44`. One row per key-encryption key that a control server of the install made or holds ([Sealing](/reference/sealing/index.html.md#key-fingerprints)). The first control server that reports an ID records its fingerprint. A key file of that ID with other bytes then fails a stage, a promote, a renewal of the row of its process, and the start of its control role. A row never changes, so the table has no row version.

| Column | Type | Default | Description |
| --- | --- | --- | --- |
| `key_id` | `text`, primary key | none | The ID of the key, such as `file:2`. |
| `fingerprint` | `text` | none | The first 16 bytes, in hex, of an HMAC-SHA256 of the text `hopbox kek fingerprint v1` under the key. |
| `created_at` | `timestamptz` | the time of the insert | When the install first saw the key. |

## server_certificates

Migration `0015_servers.sql`. One row per certificate that the CA issued to a server.

| Column | Type | Description |
| --- | --- | --- |
| `serial` | `text`, primary key | The serial number of the certificate, in lowercase hex. |
| `server_id` | `uuid` | The server of the certificate, a row of `servers`. |
| `issued` | `bigint`, unique | A number from an identity column. It grows with each certificate, so it orders the certificates of a server also within one clock tick. |
| `certificate` | `bytea` | The DER certificate. A renewal with the key of an unused certificate sends it again. |
| `not_before` | `timestamptz` | The start of the certificate. |
| `not_after` | `timestamptz` | The end of the certificate, 30 days after its issue. |
| `first_used_at` | `timestamptz` | The first call with the certificate. Null before it. |
| `retired_at` | `timestamptz` | When the first use of another certificate of the server retired it. Null while it is live. |
| `alarmed_at` | `timestamptz` | The last alarm of a call with the certificate after its retirement ([Enrollment and the install PKI](/reference/enrollment/index.html.md#client-certificates-on-each-call)). Null before the first. |
| `suppressed_alarms` | `integer` | The calls with the retired certificate since `alarmed_at` that raised no alarm. `0` by default. |
| `version` | `bigint` | The row version. |
| `created_at` | `timestamptz` | When the CA issued it, from `clock_timestamp()`. |

The enrollment inserts the first row of a server, and each new certificate of a renewal adds one. A renewal locks the row of the server in `servers`, and grants no new certificate while the newest row of the server, by `issued`, is less than 1 hour old. The first call with a certificate locks the row of the server too, sets its `first_used_at`, and sets `retired_at` on every other live row of the server: the older ones, and a newer one that the server never used. A first use whose certificate a concurrent first use retired fails. A retired certificate is refused, so after a renewal the old certificate works only until the server uses the new one ([Enrollment and the install PKI](/reference/enrollment/index.html.md#client-certificates-on-each-call)). The internal listener refuses a client certificate whose serial is not in this table, whose row names another server, that expired, that is retired, whose roles differ from the server, or whose server is `decommissioned` ([the internal listener](/reference/internal-listener/index.html.md)). The leader sweep deletes the rows that expired more than 1 day ago, and keeps the newest row of each server, so the registry always knows when the last certificate of a server ends.

## enrollment_tokens

Migration `0016_enrollment_tokens.sql`. One row per enrollment token ([Enrollment and the install PKI](/reference/enrollment/index.html.md)). The table keeps only the SHA-256 of the secret of each token, never the token.

| Column | Type | Description |
| --- | --- | --- |
| `id` | `uuid`, primary key | The ID of the token. The text of the token holds it. |
| `pool_id` | `uuid` | The pool that the token enrolls into. |
| `secret_hash` | `bytea` | The SHA-256 of the 32-byte secret of the token. |
| `roles` | `text[]` | The roles that a server can enroll with by the token. At least one. The default is `{node}`. |
| `expires_at` | `timestamptz` | The end of the token: by default 1 hour after the creation, and at most 24 hours after it. |
| `used_at` | `timestamptz` | When a server used the token. Null while the token is unused. |
| `server_id` | `uuid` | The server that the token enrolled. Null while the token is unused. |
| `for_server_id` | `uuid` | Migration `0037_enrollment_token_server.sql`. The enrolled server that the token adds roles to, or null for a token of a new server. Only that server takes such a token, and no new server does ([Enrollment and the install PKI](/reference/enrollment/index.html.md#add-roles-to-an-enrolled-server)). |
| `peer_address` | `inet` | Migration `0054_peer_addresses.sql`. The peer address of the server that enrolls with the token, or null. The enrollment must come from it ([Enrollment and the install PKI](/reference/enrollment/index.html.md#peer-addresses)). The check `enrollment_tokens_peer_address_public` refuses the same addresses as the check of `servers.peer_address`. |
| `version` | `bigint` | The row version. |
| `created_at` | `timestamptz` | When the token was made. |
| `updated_at` | `timestamptz` | The last change. |

A check makes `used_at` and `server_id` both null or both set. An enrollment locks the row of its token, so two enrollments with one token run one after the other, and the second one finds the token used. A retry of the enrollment with the same key reads `server_id` and the certificates of that server in `server_certificates`, and changes no row ([Enrollment and the install PKI](/reference/enrollment/index.html.md#retry-after-a-lost-answer)). The leader sweep deletes the tokens that expired more than 1 day ago, used or not.

The query `RegisteredPeerAddresses` reads the set `registered`: the distinct peer addresses of the tokens that are not used and not expired, in order. The control role reads it every second for the host firewall of its server, which opens the internal listener to these addresses ([Enrollment and the install PKI](/reference/enrollment/index.html.md#peer-addresses)). A token leaves the set when a server uses it, and when it expires.

## device_authorizations

Migration `0019_cli_sign_in.sql`. One row per device sign-in of the hopbox CLI ([CLI tokens, API keys, admin API keys, and SSH keys](/reference/credentials/index.html.md#the-device-flow)). The table keeps only the SHA-256 of the device code and of the user code. The leader deletes a row 24 hours after its creation. Until then, the row counts for the limits of the device flow.

| Column | Type | Description |
| --- | --- | --- |
| `id` | `uuid`, primary key | A UUIDv7. It is the target ID of the audit entries of the device sign-in. |
| `device_code_hash` | `bytea`, unique | The SHA-256 of the device code. |
| `user_code_hash` | `bytea`, unique | The SHA-256 of the user code, in upper case without the dash. |
| `client_name` | `text` | The name that the CLI gave, or empty. |
| `requested_ip` | `text` | The IP address that started the device sign-in, or empty. |
| `state` | `text` | `pending`, then `approved` after `ApproveDevice`, then `used` after the poll that took the tokens. |
| `account_id` | `uuid`, can be null | The account that approved the code. A check makes it null exactly while the state is `pending`. A delete of the account deletes the row. |
| `approved_at` | `timestamptz`, can be null | The approval. |
| `interval_seconds` | `integer` | The least time between two polls. It starts at 5 and grows by 5 with each poll that comes too soon. |
| `last_polled_at` | `timestamptz`, can be null | The last poll. |
| `expires_at` | `timestamptz` | 10 minutes after `created_at`. |
| `version` | `bigint` | The row version. |
| `created_at` | `timestamptz` | The start. |

One statement approves a pending code, so two approvals at the same time cannot both succeed. A poll locks the row, so two polls of one code run one after the other, and only one of them takes the tokens.

## device_approval_failures

Migration `0019_cli_sign_in.sql`. One row per approval with a wrong user code, for the limit of each account ([CLI tokens, API keys, admin API keys, and SSH keys](/reference/credentials/index.html.md#limits-of-the-device-flow)). The leader deletes a row 24 hours after its creation. A delete of the account deletes its rows.

| Column | Type | Description |
| --- | --- | --- |
| `id` | `uuid`, primary key | A UUIDv7. |
| `account_id` | `uuid` | The account that entered the code. |
| `version` | `bigint` | The row version. |
| `created_at` | `timestamptz` | The failed approval. |

## cli_sign_ins

Migration `0019_cli_sign_in.sql`. One row per CLI sign-in: the family of tokens that one device flow started ([CLI tokens, API keys, admin API keys, and SSH keys](/reference/credentials/index.html.md#cli-tokens)). A delete of the row deletes every token of the family. A sign-out, `EndCLISignIn`, a reuse of a replaced refresh token, and the delete of the account delete the row. The leader deletes a row that has no refresh token left that can renew it.

| Column | Type | Description |
| --- | --- | --- |
| `id` | `uuid`, primary key | A UUIDv7. It is the credential ID of the audit entries of the CLI, and the field `cli_sign_in_id` of `CLITokens`. |
| `account_id` | `uuid` | The account. A delete of the account deletes the row. |
| `client_name` | `text` | The name that the CLI gave at the start of the device flow. |
| `client_ip` | `text` | The IP address that started the device flow, or empty. Migration `0023_cli_sign_in_ip.sql` adds it. |
| `last_used_at` | `timestamptz` | The last call with an access token of the family, recorded at most once per minute. |
| `last_used_ip` | `text` | The IP address of the last call. A call from a new address records it at once. It starts as `client_ip`. Migration `0025_cli_sign_in_last_ip.sql` adds it. |
| `version` | `bigint` | The row version. |
| `created_at` | `timestamptz` | The poll that took the first tokens. |

## cli_refresh_tokens

Migration `0019_cli_sign_in.sql`. One row per refresh token of a CLI sign-in.

| Column | Type | Description |
| --- | --- | --- |
| `id` | `uuid`, primary key | A UUIDv7. |
| `sign_in_id` | `uuid` | The CLI sign-in. A delete of the sign-in deletes the row. |
| `token_hash` | `bytea`, unique | The SHA-256 of the whole token, `hbxrt_` included. |
| `expires_at` | `timestamptz` | 30 days after `created_at`. |
| `replaced_at` | `timestamptz`, can be null | The refresh that used the token. A token with this column set that comes back ends the CLI sign-in. |
| `replaced_by` | `uuid`, can be null | The refresh token that replaced this one. Migration `0024_cli_refresh_grace.sql` adds it. |
| `grace_used_at` | `timestamptz`, can be null | The one use of the token within 10 s of its replacement, while `replaced_by` was unused ([CLI tokens, API keys, admin API keys, and SSH keys](/reference/credentials/index.html.md#cli-tokens)). Migration `0024_cli_refresh_grace.sql` adds it. |
| `version` | `bigint` | The row version. |
| `created_at` | `timestamptz` | The issue of the token. |

A refresh locks the token and its CLI sign-in. So two refreshes with one token run one after the other, and the second one sees the token replaced. Within the grace, the second one gets a new pair. After it, the second one ends the sign-in. Each normal refresh marks every other live refresh token of the sign-in replaced, with `replaced_by` null. A later use of such a token gets no grace and ends the sign-in. A replaced token stays until its 30 days end, then the leader deletes it.

## cli_access_tokens

Migration `0019_cli_sign_in.sql`. One row per access token of a CLI sign-in. The leader deletes the expired rows.

| Column | Type | Description |
| --- | --- | --- |
| `id` | `uuid`, primary key | A UUIDv7. |
| `sign_in_id` | `uuid` | The CLI sign-in. A delete of the sign-in deletes the row. |
| `refresh_token_id` | `uuid`, can be null | The refresh token issued with this access token. A refresh that drops that refresh token deletes the row in the same statement. Migration `0026_cli_access_token_pair.sql` adds it. |
| `token_hash` | `bytea`, unique | The SHA-256 of the whole token, `hbxat_` included. |
| `expires_at` | `timestamptz` | 1 hour after `created_at`. |
| `version` | `bigint` | The row version. |
| `created_at` | `timestamptz` | The issue of the token. |

## api_keys

Migration `0020_api_keys.sql`. One row per API key ([CLI tokens, API keys, admin API keys, and SSH keys](/reference/credentials/index.html.md#api-keys)). The table keeps only the SHA-256 of each key.

| Column | Type | Description |
| --- | --- | --- |
| `id` | `uuid`, primary key | A UUIDv7. It is the credential ID of the audit entries of the key. |
| `org_id` | `uuid` | The org of the key. A delete of the org deletes the row. |
| `kind` | `text` | `member` or `org` ([Domain rules](/reference/domain-rules/index.html.md#api-key-kinds)). |
| `account_id` | `uuid`, can be null | The account of a member key. A check makes it set exactly for a member key. A delete of the account deletes the row. |
| `created_by` | `uuid`, can be null | The account that made the key. A delete of that account sets it to null, so an org key stays. |
| `name` | `text` | The name of the key, never empty. |
| `token_hash` | `bytea`, unique | The SHA-256 of the whole key, `hbx_` included. |
| `expires_at` | `timestamptz` | The end of the key. |
| `last_used_at` | `timestamptz`, can be null | The last call with the key, recorded at most once per minute. |
| `version` | `bigint` | The row version. |
| `created_at` | `timestamptz` | The creation. |

The leader deletes a key 30 days after its expiry. A member key resolves only while its account has a row in `memberships` for the org of the key. So a member that leaves the org loses its keys there at once, without a write to this table.

## admin_keys

Migration `0021_admin_keys.sql`. One row per admin API key ([CLI tokens, API keys, admin API keys, and SSH keys](/reference/credentials/index.html.md#admin-api-keys)). An admin key belongs to no account and no org. The table keeps only the SHA-256 of each key.

| Column | Type | Description |
| --- | --- | --- |
| `id` | `uuid`, primary key | A UUIDv7. It is the credential ID of the audit entries of the key. |
| `name` | `text` | The name of the key, never empty. |
| `token_hash` | `bytea`, unique | The SHA-256 of the whole key, `hbxadm_` included. A second registration of one hash inserts nothing. |
| `expires_at` | `timestamptz` | The end of the key. |
| `last_used_at` | `timestamptz`, can be null | The last call with the key, recorded at most once per minute. |
| `version` | `bigint` | The row version. |
| `created_at` | `timestamptz` | The creation or the registration. |

The leader deletes a key 30 days after its expiry.

## ssh_keys

Migration `0022_ssh_keys.sql`. One row per SSH key of an account ([CLI tokens, API keys, admin API keys, and SSH keys](/reference/credentials/index.html.md#ssh-keys)).

| Column | Type | Description |
| --- | --- | --- |
| `id` | `uuid`, primary key | A UUIDv7. |
| `account_id` | `uuid` | The account. A delete of the account deletes the row. |
| `name` | `text` | The name of the key, or empty. |
| `key_type` | `text` | The key type, such as `ssh-ed25519`. |
| `bits` | `integer` | The size of the key. |
| `public_key` | `text` | The key in the `authorized_keys` form, without its comment. |
| `fingerprint` | `text`, unique | The SHA-256 fingerprint, such as `SHA256:DRdf39Nvm+zAXRzjpCOtwde9E/UUaXIbgz0+g1NA+10`. The unique index makes each key work for one account only, also for two adds at the same time. |
| `version` | `bigint` | The row version. |
| `created_at` | `timestamptz` | The addition. |

## sizes

Migration `0027_sizes_and_images.sql`. One row per size of machine ([Domain rules](/reference/domain-rules/index.html.md#sizes)). The migration inserts the size `default`: 2 vCPUs, 8589934592 bytes of RAM (8 GiB), and 100000000000 bytes of disk (100 GB). A machine copies the numbers of its size at its create. A size never changes, and a retired size keeps its row and its name, because its machines name it. The admin API adds and retires sizes ([the admin API](/reference/admin-api/index.html.md#sizeservice)).

| Column | Type | Description |
| --- | --- | --- |
| `name` | `text`, primary key | The name of the size. A check holds a lowercase DNS label of 1 to 63 characters that starts with a letter. |
| `vcpus` | `integer` | 1 to 32. |
| `memory_bytes` | `bigint` | The RAM in bytes: at least 134217728 (128 MiB), and a whole number of MiB. |
| `disk_bytes` | `bigint` | The sparse disk in bytes: at least 1000000000 (1 GB), and a whole number of 512-byte sectors. |
| `version` | `bigint` | The row version. |
| `created_at` | `timestamptz` | When the size was made. |
| `updated_at` | `timestamptz` | The last change. |
| `pool_id` | `uuid`, null, references `pools` | The pool of the machines of the size, or null for any pool. Migration `0042_pools_and_operator_sizes.sql` adds it. |
| `labels` | `jsonb` | The labels that a node must have to take a machine of the size, such as `{"disk":"nvme"}`. Default `{}`. Migration `0042`. |
| `retired_at` | `timestamptz`, null | The retirement of the size. A retired size takes no new machine. A check keeps the size `default` from a retirement. Migration `0042`. |

## images

Migrations `0027_sizes_and_images.sql` and `0045_control_processes.sql` (the column `release`). One row per image that new machines start from: the metadata of an image artifact ([Artifact format](/reference/artifact-format/index.html.md)). `RegisterImage` of the admin API inserts the rows ([the admin API](/reference/admin-api/index.html.md#registerimage)). A row never changes, so it has no `version`.

| Column | Type | Description |
| --- | --- | --- |
| `id` | `uuid`, primary key | A UUIDv7. |
| `name` | `text` | The name of the image, for example `default`: 1 to 63 lowercase letters, digits, dots, and hyphens, which start and end with a letter or a digit. Each build of an image has the same name. |
| `digest` | `text`, unique | `sha256:` and the sha256 of the bytes of `artifact.json`, in lowercase hex. It names one build. |
| `arch` | `text` | The CPU architecture, `x86_64`. Not empty. |
| `kernel_sha256` | `text` | The sha256 of the guest kernel that the image needs, 64 lowercase hex digits. |
| `disk_size` | `bigint` | The size of the disk in bytes. More than 0. |
| `disk_sha256` | `text` | The sha256 of the disk, 64 lowercase hex digits. |
| `metadata` | `bytea` | The bytes of `artifact.json`, unchanged, so the digest can be checked again. |
| `url` | `text` | The https URL from which nodes download the compressed disk, or empty. |
| `built_at` | `timestamptz` | `created_at` of `artifact.json`: the time of the build. |
| `created_at` | `timestamptz` | The registration. |
| `release` | `text` | The release of `hopboxd` that registered the image as the default image of its release, such as `v0.2.0`. Empty for an image that an operator registered, and for an image that a release before migration `0045` registered. Default empty. Migration `0045_control_processes.sql` adds it. The control role skips its default image when the install has an image of the same name with a newer release ([hopboxd](/reference/hopboxd/index.html.md#the-default-image-of-the-release)). |

The index `images_name` on `(name, created_at DESC)` finds the newest image of a name. A create takes that image: the newest registration, then the highest ID.

## machines

Migration `0028_machines.sql`. One row per machine ([the machine API](/reference/machine-api/index.html.md)). A request sets only `target_state`. The executor of the machine on its node moves `current_state` toward it. Only its reports write `current_state` ([the executor protocol](/reference/executor-protocol/index.html.md#machinereport)). A machine outlives its org and its creator until its executor deleted it, so those keys go null. An org goes only after each of its machines has the target `deleted`.

| Column | Type | Default | Description |
| --- | --- | --- | --- |
| `id` | `uuid`, primary key | none | A UUIDv7. |
| `org_id` | `uuid`, can be null | none | The org of the machine. A delete of the org sets it to null. |
| `creator_account_id` | `uuid`, can be null | none | The account that made the machine. A delete of the account sets it to null. Null for a machine that an org key made. |
| `creator_key_id` | `uuid`, can be null | none | The org API key that made the machine, a row of `api_keys`. Such a machine is shared from the start. A revoke deletes the key, and sets this column to null: the machine stays. |
| `name` | `text` | none | The name of the machine. A machine that is being deleted keeps its name here, while `machine_names` can give the name to a new machine of the same org. |
| `shared` | `boolean` | `false` | `true` for a machine that is open to the whole org. |
| `target_state` | `text` | none | `running`, `standby`, `hibernated`, `stopped`, or `deleted` ([Domain rules](/reference/domain-rules/index.html.md#machine-states)). |
| `current_state` | `text` | none | `pending`, `starting`, `running`, `standby`, `hibernated`, `stopped`, `failed`, `deleted`, or `moving`. |
| `size_name` | `text` | none | The name of the size at the create. |
| `vcpus` | `integer` | none | The vCPUs, copied from the size. More than 0. |
| `memory_bytes` | `bigint` | none | The RAM in bytes, copied from the size. More than 0. |
| `disk_bytes` | `bigint` | none | The disk in bytes, copied from the size. More than 0. |
| `image_id` | `uuid` | none | The image that the machine started from, a row of `images`. |
| `restart_policy` | `text` | none | `always` or `never` ([Domain rules](/reference/domain-rules/index.html.md#restart-policy)). |
| `start_command` | `text` | empty | The start command, or empty. |
| `auto_delete_seconds` | `bigint` | `0` | The auto-delete timeout in seconds. 0 is off. 0 or more. |
| `url_port` | `integer` | `8000` | The port in the machine that its URL reaches, 1 to 65535. |
| `url_visibility` | `text` | `private` | `private` or `public`. |
| `ssh_port` | `integer` | none | The SSH port of the machine, 10000 to 30000. `ssh_ports` holds it. |
| `server_id` | `uuid`, can be null | none | The node of the machine, a row of `servers`. The create sets it from the placement. |
| `reboot_generation` | `bigint` | `0` | The number of boots that requests asked for. A reboot, a rename of a running machine, and a start of a failed machine add 1. A start of a stopped machine with the restart policy `never` adds 1 too. |
| `skipped_env` | `integer` | `0` | Migration `0032_machine_skipped_env.sql`. The count of env values that the guest agent of the running machine left out, from the last report of `running` ([the executor protocol](/reference/executor-protocol/index.html.md#machinereport)). 0 or more. |
| `current_reboot_generation` | `bigint` | `0` | The boot generation of the last report of the executor. It never goes down. The executor boots with `reboot_generation`, so a lower value than `reboot_generation` means a pending boot (`reboot_pending` in the API). So a request can tell that its boot ran. |
| `last_activity_at` | `timestamptz` | the time of the insert | The last use: inbound traffic, an exec, a copy, an SSH session, a start, or a new auto-delete timeout. The auto-delete job reads it. |
| `state_changed_at` | `timestamptz` | the time of the insert | The last change of `current_state`. |
| `delete_requested_at` | `timestamptz`, can be null | none | The first request for the target `deleted`. |
| `version` | `bigint` | `1` | The row version. Each change of a request or a report adds 1. A change of an env var or a secret that reaches the machine adds 1 too, because the env is part of its spec. The specs of the executor protocol carry it, so the executor keeps the newest one. |
| `created_at` | `timestamptz` | the time of the insert | When the machine was made. |
| `updated_at` | `timestamptz` | the time of the insert | The last change of a request or a report. |

The table has three checks across its columns:

| Check | Why |
| --- | --- |
| `org_id` is set, or `target_state` is `deleted`. | Only a machine on its way out can lose its org. |
| `creator_account_id` or `creator_key_id` is null. | A machine has one creator: an account or an org key. Both are null after the delete of the creator account or the revoke of the creator key. |
| `target_state` is `deleted`, and `delete_requested_at` is set. Or neither. | The delete request and its time go together. |

| Index | Columns | Use |
| --- | --- | --- |
| `machines_org` | `(org_id, name)`, where `org_id` is set | The machines of an org, by name. |
| `machines_name` | `name` | The lookup by name. |
| `machines_server` | `server_id`, where it is set | What a node uses ([Placement](/reference/placement/index.html.md#what-a-node-uses)). |
| `machines_auto_delete` | `last_activity_at`, where `auto_delete_seconds` is more than 0 and the target is not `deleted` | The auto-delete job. |
| `machines_purge` | `delete_requested_at`, where `current_state` is `deleted` | The sweep that drops the rows of deleted machines. |

The leader drops a row whose `current_state` is `deleted` once its `delete_requested_at` is 7 days old. The delete cascades to its rows in `machine_names`, `ssh_ports`, `placements`, and `route_records`. The same sweep then deletes its lease.

## machine_names

Migration `0028_machines.sql`. One row per machine name that is live or reserved. Machine names are unique in the whole install ([Domain rules](/reference/domain-rules/index.html.md#machine-names)).

| Column | Type | Description |
| --- | --- | --- |
| `name` | `text`, primary key | The name. A check holds a lowercase DNS label of 1 to 63 characters that starts with a letter, and refuses `ns1` and `ns2`. The service layer checks the full rules first ([Domain rules](/reference/domain-rules/index.html.md#machine-names)). |
| `machine_id` | `uuid` | The machine that holds the name, or held it last. A delete of the machine row deletes the row. The foreign key is deferred to the end of the transaction, because a create claims the name before it inserts the machine. |
| `org_id` | `uuid` | The org of that machine. No foreign key: a reservation outlives a deleted org. |
| `reserved_until` | `timestamptz`, can be null | Null while the name is live. A delete or a rename sets it 7 days ahead ([Domain rules](/reference/domain-rules/index.html.md#name-reservation)). |
| `version` | `bigint` | The row version. |
| `created_at` | `timestamptz` | The last claim of the name. |
| `updated_at` | `timestamptz` | The last change. |

An org holds at most 100 names in reservation at once (`domain.MaxNameReservations`). A rename beyond that fails, and a delete always reserves. The leader sweep that frees the names checks the leader lease ([Fenced writes](#fenced-writes)).

A claim inserts the name. If the name has a row, the claim takes it only in two cases: its reservation ended, or the same org holds it in reservation. A claim of a live name, or of a name that another org holds in reservation, gets no row. Two claims of one name at the same time run one after the other, and the second one sees the first. The unique index `machine_names_live` on `machine_id`, where `reserved_until` is null, gives each machine at most one live name. The index `machine_names_reserved` on `reserved_until` serves the sweep. The leader deletes each row whose reservation ended.

## ssh_ports

Migration `0028_machines.sql`. One row per SSH port that a machine holds, or held until its reservation ends ([Domain rules](/reference/domain-rules/index.html.md#ssh-ports)).

| Column | Type | Description |
| --- | --- | --- |
| `port` | `integer`, primary key | The port, 10000 to 30000. |
| `machine_id` | `uuid`, unique | The machine. A delete of the machine row deletes the row. The foreign key is deferred to the end of the transaction, as in `machine_names`. |
| `reserved_until` | `timestamptz`, can be null | Null while the machine lives. A delete sets it 7 days ahead. |
| `version` | `bigint` | The row version. |
| `created_at` | `timestamptz` | The claim of the port. |
| `updated_at` | `timestamptz` | The last change. |

A claim picks a random port of the range that no row holds, or whose reservation ended. A concurrent claim can take the same port first. The create then tries again, 5 times at most. The index `ssh_ports_reserved` on `reserved_until` serves the sweep. The leader deletes each row whose reservation ended.

## node_subnets

Migration `0039_machine_addresses.sql`. The subnet of each node with machines ([the node network](/reference/node-network/index.html.md#addresses)).

| Column | Type | Description |
| --- | --- | --- |
| `server_id` | `uuid`, primary key | The node, a row of `servers`. |
| `subnet` | `cidr` | A `/20` of `10.240.0.0/12`. The exclusion constraint on `subnet` keeps two subnets from overlapping. |
| `version` | `bigint` | The row version. |
| `created_at` | `timestamptz` | The claim of the subnet. |
| `updated_at` | `timestamptz` | The last change. |

The create of the first machine of a node claims its subnet: the first `/20` of the range that no node holds. The claim takes the transaction lock `LockKeyNodeSubnets` of `internal/store/locks.go`, so two nodes never pick the same subnet.

The leader sweep deletes the row of a decommissioned node once no row of `machine_addresses` holds an address of its `subnet`. An address holds the subnet while `held_until` is null or in the future. A moved machine keeps the address of its first node, so the row stays while such a machine lives. The next claim can take the freed subnet ([the node network](/reference/node-network/index.html.md#subnet-of-a-decommissioned-node)).

## machine_addresses

Migration `0039_machine_addresses.sql`. One row per IPv4 address that a machine holds, or held until its hold ends ([the node network](/reference/node-network/index.html.md#addresses)).

| Column | Type | Description |
| --- | --- | --- |
| `address` | `inet`, primary key | The address, a `/32` of `10.240.0.0/12`. |
| `server_id` | `uuid` | The node of the machine. It is the node whose subnet holds the address, until a cold move hands the machine to another node ([the node network](/reference/node-network/index.html.md#moved-machines)). |
| `machine_id` | `uuid`, unique | The machine. A delete of the machine row deletes the row. The foreign key is deferred to the end of the transaction, as in `machine_names`. |
| `held_until` | `timestamptz`, can be null | Null while the machine lives. The report of `deleted` sets it 5 minutes ahead (`domain.AddressHold`). |
| `version` | `bigint` | The row version. |
| `created_at` | `timestamptz` | The claim of the address. |
| `updated_at` | `timestamptz` | The last change. |

The create of a machine claims the first address of the subnet of its node that no row holds, or whose hold ended. It skips the first and the last address of the subnet. The create holds the row lock of the node, so two creates on one node never pick the same address. The index `machine_addresses_server` on `server_id` lists the addresses of a node. `MachineSpecs` reads the address of each machine for its spec.

## mesh_members

Migrations `0049_mesh_members.sql`, `0055_mesh_on_peer_addresses.sql`, and `0057_drop_vrack_address.sql`. One row for each server in the mesh ([the mesh](/reference/mesh/index.html.md)).

| Column | Type | Description |
| --- | --- | --- |
| `server_id` | `uuid`, primary key | The server, a row of `servers`. |
| `address` | `inet`, unique | The mesh address of the server, a `/32` of `10.239.0.0/16`. It stays with the server for its whole life. |
| `public_key` | `bytea`, unique | The public key of the WireGuard device of the server, 32 bytes. |
| `version` | `bigint` | The row version. |
| `created_at` | `timestamptz` | The first heartbeat of the server with its part of the mesh. |
| `updated_at` | `timestamptz` | The last change. |

The heartbeat of a server writes its row, in the transaction of the heartbeat ([the control channel](/reference/control-channel/index.html.md#mesh-pushes)). The first heartbeat with the field `mesh` of a server with a peer address claims the first address of the mesh range that no row holds, from `10.239.0.1`. A server without a peer address gets no row. The claim takes the transaction lock `LockKeyMeshAddresses` of `internal/store/locks.go`, so two servers never pick the same address. A later heartbeat with another key updates the row, and keeps the address. Each insert and each update writes the audit entry `server.mesh`, with the details `address` and `public_key`. An update also has `old_public_key`.

The endpoint of each member is its `servers.peer_address`. The read of the mesh leaves out a member without a peer address.

Migrations `0049` and `0055` have the line `-- oldest_schema_version: 44`, because a `hopboxd` of an older release never reads the table ([the bundled Postgres](/reference/postgres/index.html.md#two-releases-on-one-store)). The servers of such a release stay out of the mesh until they upgrade. Migration `0057` drops the old column `vrack_address`, which held the vRack address of a server before migration `0055`. It has the line `-- oldest_schema_version: 57`. The builds of schema version 49 to 56 write that column, and no release has a schema version from 44 to 56. So such a build refuses the store at its start, and does not fail later at a write of the mesh.

A key that the row of another server holds changes nothing: the heartbeat goes on without its part of the mesh. A decommissioned server keeps its row, so no other server takes its address. The read of the mesh leaves out the decommissioned servers. It also reads `servers.egress_exit_version`, `hopboxd_version`, and `state`, which give each edge server its field `egress` ([the egress of the machines](/reference/egress/index.html.md#egress-edges)).

## machine_moves

Migration `0050_machine_moves.sql`. One row per cold move of a machine ([Cold move](/reference/cold-move/index.html.md)).

| Column | Type | Description |
| --- | --- | --- |
| `id` | `uuid`, primary key | A UUIDv7. |
| `machine_id` | `uuid` | The machine. It has no foreign key, so a node can still remove an old copy after the leader drops the row of a deleted machine. |
| `source_server_id` | `uuid` | The node that the machine leaves, a row of `servers`. |
| `target_server_id` | `uuid` | The node that the machine moves to, a row of `servers`. It differs from the source. |
| `state` | `text` | `stopping`, `pulling`, `done`, or `failed`. |
| `target_state` | `text` | The target state of the machine before the move: `running` or `stopped`. |
| `pinned_target` | `boolean` | True when the instance admin named the target. A target without room then fails the move, and placement picks no other one. |
| `error` | `text` | Why the move failed. Empty otherwise. |
| `cleanup_server_id` | `uuid`, can be null | The node that still holds an old copy of the files of the machine: the source after `done`, the target after a move that failed in `pulling`. Null once the heartbeat of that node says that it removed them, for a move that failed in `stopping`, and for a failed move that left its machine on the target. |
| `version` | `bigint` | The row version. |
| `created_at` | `timestamptz` | The start of the move. The stop timeout counts from it. |
| `updated_at` | `timestamptz` | The last change. In `pulling`, it is the time of the hand-off or of the last progress of the pull, and the pull deadline counts from it. |

The unique partial index `machine_moves_active` lets one move of a machine run at a time, in `stopping` or `pulling`. The index `machine_moves_serving` lists the disks that a source serves, and `machine_moves_cleanup` the old copies of a node. A change of a move locks the machine row first, then the move row. The rows stay after the move as its record.

The migration adds the column `servers.disk_transfer_version` too. It has the line `-- oldest_schema_version: 44`, because a `hopboxd` of an older release never reads either. Such a release starts no move. While a live row of `control_processes` has a `schema_version` below 50, a newer control role starts no move and hands off none either ([Cold move](/reference/cold-move/index.html.md#two-releases)).

## server_drains

Migration `0051_server_drains.sql`. One row per server that drains ([Cold move](/reference/cold-move/index.html.md#drain-of-a-node)). The row goes with the decommission.

| Column | Type | Default | Description |
| --- | --- | --- | --- |
| `server_id` | `uuid`, primary key | none | The server, a row of `servers`. |
| `started_at` | `timestamptz` | the time of the insert | The start of the drain, or of its last retry. A move that failed before it strands no machine. |
| `empty_checks` | `integer` | `0` | The empty checks in a row: 0 to 3. A check constraint keeps it in that range. |
| `checked_at` | `timestamptz`, can be null | null | The time of the last empty check. |
| `error` | `text` | empty | Why the drain waits now, such as a refused decommission. Empty otherwise. |
| `version` | `bigint` | `1` | The row version. |
| `created_at` | `timestamptz` | the time of the insert | The start of the first drain. |
| `updated_at` | `timestamptz` | the time of the insert | The last change. |

## drain_strands

Migration `0051_server_drains.sql`. One row per machine that the drain of a server cannot move, with the reason. A retry of the drain drops the rows, and the drain tries each machine again.

| Column | Type | Description |
| --- | --- | --- |
| `server_id` | `uuid` | The draining server, a row of `server_drains`. A delete of that row deletes the row. |
| `machine_id` | `uuid` | The machine, a row of `machines`. A delete of the machine row deletes the row. |
| `reason` | `text` | Why the machine cannot move. It is never empty. |
| `version` | `bigint` | The row version. |
| `created_at` | `timestamptz` | The time of the row. |
| `updated_at` | `timestamptz` | The last change. |

The primary key is `(server_id, machine_id)`. The index `drain_strands_machine` lists the rows of a machine.

The migration has the line `-- oldest_schema_version: 44`, so a release of the schema version 44 to 50 runs beside it. Such a release never reads the two tables. It shows `draining` and `shutting_down` as unknown (`SERVER_STATE_UNSPECIFIED`), and places nothing on such a node, because placement takes only `ready` nodes. It keeps the state in its heartbeats, and its leader job never marks such a node unresponsive. Its uncordon can clear the flag `cordoned`, which the next step of the drain sets again. Its leader runs no drain job, and its hand-off of a move from a draining source waits and fails after 10 min. While a live row of `control_processes` has a `schema_version` below 51, `DrainServer` refuses, and the drain job starts no move and decommissions nothing ([Cold move](/reference/cold-move/index.html.md#two-releases)).

## removal_strands

Migration `0056_role_removal.sql`. One row per machine that the removal of the node role from a server cannot move, with the reason, as [drain_strands](#drain_strands) for a drain. The end and the cancel of the removal drop the rows.

| Column | Type | Description |
| --- | --- | --- |
| `server_id` | `uuid` | The server of the removal, a row of `servers`. |
| `machine_id` | `uuid` | The machine, a row of `machines`. A delete of the machine row deletes the row. |
| `reason` | `text` | Why the machine cannot move. It is never empty. |
| `version` | `bigint` | The row version. |
| `created_at` | `timestamptz` | The time of the row. |
| `updated_at` | `timestamptz` | The last change. |

The primary key is `(server_id, machine_id)`. The index `removal_strands_machine` lists the rows of a machine.

The migration also adds the reports of the heartbeats, the removal, and `removed_roles` to [servers](#servers). The check constraint `servers_removal_whole` keeps `removal_roles`, `removal_cordoned_before`, and `removal_started_at` all null or all set, with at least one role. The partial index `servers_removal` lists the servers with a removal. The migration has the line `-- oldest_schema_version: 44`, so a release of the schema version 44 to 55 runs beside it. Such a release never reads the new columns and the new table, and its heartbeats leave the reports as they are. Its enrollment refuses a list that drops a role, so no removal ends through it.

Such a release adds a role to an enrolled server without the check of `removed_roles`. So the trigger `servers_role_add` holds that check in the store, for each release. It runs the function `servers_role_add_gate` before each `UPDATE` of `servers.roles`. It refuses new roles that are not in the old `roles` or the old `removed_roles`, with the code `check_violation` (`23514`). A drop of a role passes. The enrollment of this release checks `removed_roles` first. It sets the roles before it takes the role out of `removed_roles`, so the trigger never refuses it.

## placements

Migration `0028_machines.sql`. One row per placement of a machine on a node ([Placement](/reference/placement/index.html.md)). A row never changes, so it has no `version`.

| Column | Type | Description |
| --- | --- | --- |
| `id` | `uuid`, primary key | A UUIDv7. |
| `machine_id` | `uuid` | The machine. A delete of the machine row deletes the row. |
| `server_id` | `uuid` | The node, a row of `servers`. |
| `lease_epoch` | `bigint` | The epoch of the grant of the machine lease. |
| `lease_token` | `uuid` | The token of that grant. With the epoch, it names the execution of the machine on the node ([leases](#leases)). |
| `tries` | `integer` | How many nodes the placement tried, the last one included. 1 or more. |
| `created_at` | `timestamptz` | The placement. |

The index `placements_machine` on `(machine_id, created_at)` lists the placements of a machine in order.

A heartbeat that grants the lease of a machine again writes a new row too, with `tries` 1 ([the executor protocol](/reference/executor-protocol/index.html.md#grants)).

## route_records

Migration `0031_route_records.sql`. One row per machine that runs. A route record maps a machine to the node that runs it. It carries the epoch and the token of the machine lease of that execution, so traffic for a moved machine never goes to its previous node ([leases](#leases)). The edges get it through the routes that the control role pushes ([the control channel](/reference/control-channel/index.html.md#edge-pushes)).

The report of the executor writes the record in its transaction ([the executor protocol](/reference/executor-protocol/index.html.md#what-the-control-role-does)). A report of `running` inserts or updates it, also a report that repeats the state. A report of any other state deletes it. A heartbeat that grants the lease of the machine again moves the record to the new grant, in its transaction. `SetMachineURLPort` updates `url_port` of a record that exists.

| Column | Type | Description |
| --- | --- | --- |
| `machine_id` | `uuid`, primary key | The machine. A delete of the machine row deletes the record. |
| `server_id` | `uuid` | The node that runs the machine, a row of `servers`. |
| `address` | `inet`, can be null | The address of the machine on its node, from `machine_addresses`. Each report of `running` writes it. It is null only for a machine without an address. Migration `0039_machine_addresses.sql` fills it for the records that exist. |
| `ssh_port` | `integer` | The SSH port of the machine, 10000 to 30000. |
| `url_port` | `integer` | The port in the machine that its URL reaches, 1 to 65535. |
| `lease_epoch` | `bigint` | The epoch of the grant of the execution. |
| `lease_token` | `uuid` | The token of that grant. |
| `ssh_host_keys` | `text[]` | The public SSH host keys of the guest that the record holds, in the `authorized_keys` format. At most 8. Default: empty. Migration `0038_ssh_front_door.sql` adds it, with the GIN index `route_records_ssh_host_keys`. The SSH front door accepts only these keys from the machine ([the SSH front door](/reference/ssh-front-door/index.html.md#connection-to-the-machine)). Each key in it has its row in [route_host_keys](#route_host_keys), so no key is in two records. They are the keys of `claimed_host_keys` that no other record holds, in the same order. |
| `claimed_host_keys` | `text[]` | The keys of the last report of `running`, also those that the record of another machine holds. At most 8. Default: empty. Migration `0041_route_host_keys.sql` adds it, with the GIN index `route_records_claimed_host_keys`, and fills it from `ssh_host_keys`. A key that another record lets go comes to this record through it. |
| `version` | `bigint` | The row version. A report that changes no column writes nothing. |
| `created_at` | `timestamptz` | The first report of `running`. |
| `updated_at` | `timestamptz` | The last change. |

The index `route_records_server` on `server_id` lists the records of a node.

`Service.EdgeRoutes` reads the routes of the edges. It lists each machine that holds its name in `machine_names`, whose target and current state are not `deleted`, with its org. A machine gets a target only in one case. It is `running`, it has a route record, and its lease is the live grant of the record. The lease must have the epoch and the token of the record, the node of the record as its holder, and a future end. The target then holds `address`, `url_port`, `ssh_host_keys`, the grant, and the rest of the lease. Each route also has the `ssh_port` of its machine. A record of an older grant, or of an expired lease, gives no target. So an edge never sends traffic to a node that can have lost the machine.

## route_host_keys

Migration `0041_route_host_keys.sql`. One row per SSH host key that a route record holds (docket item #94). The primary key lets at most one record hold a key, also with several control processes at once.

| Column | Type | Description |
| --- | --- | --- |
| `key` | `text`, primary key | The public SSH host key, in the `authorized_keys` format. |
| `machine_id` | `uuid` | The machine whose route record holds the key. A delete of the route record deletes its rows. |

The index `route_host_keys_machine` on `machine_id` lists the keys of a record.

The migration fills the table from `route_records.ssh_host_keys` under the lock `LockKeyRetiredGuestHostKeys` ([the bundled Postgres](/reference/postgres/index.html.md#advisory-lock-keys)). If two records list one key, the record with the oldest `created_at` keeps it, and the migration removes it from the other record.

A report of the executor locks the route record of its machine first. It then works on each key of the record and of the report in the sort order of the keys, so two reports never wait on each other in a cycle:

1. A key of the report that no row holds gets a row of the record.
2. A key of the report that the row of another record holds stays out of the record. The row stays locked until the end of the transaction, so its holder cannot let it go meanwhile. The service logs `a guest reported an SSH host key of another machine, so the SSH front door does not reach it` at level error, with `machine` and `other_machine`.
3. A key that the record holds and the report does not claim leaves the record. The other record with the oldest `created_at` that claims it in `claimed_host_keys` and does not hold it then gets it, in the same transaction. That record must hold only keys that it claims, and no third record can list the key. A record that another transaction locks is left out. The service logs `a machine got back an SSH host key that another machine let go` at level info, with `machine`, the record that gets the key, and `other_machine`. A record that is left out, or a key that a `hopboxd` of v0.1.0 lets go, gets the key from the return of SSH host keys, a leader job that runs every minute ([hopboxd](/reference/hopboxd/index.html.md#leader-jobs)).

Then the report writes `ssh_host_keys`. A report of another state lets every key of the record go, as in step 3, and then deletes the record. The edges get the new keys with the next push of the routes ([the control channel](/reference/control-channel/index.html.md#edge-pushes)).

### Upgrade from v0.1.0

A `hopboxd` of v0.1.0 knows nothing of this table. It takes `LockKeyRetiredGuestHostKeys` exclusively, looks for a key only in `route_records.ssh_host_keys` of the other records, and writes the keys only there. So for one release, the writer of this release also does these things:

- It takes `LockKeyRetiredGuestHostKeys` as a shared transaction lock before it writes keys. Two writers of this release do not wait on each other, but each one waits for a writer of v0.1.0.
- It keeps writing `route_records.ssh_host_keys`, which a writer of v0.1.0 reads.
- After step 2, it also looks for each key of the report in `ssh_host_keys` of the other records. A key that another record lists stays out, and its row goes.
- It treats a row as stale when the record of its machine does not list the key. A writer of v0.1.0 changes the keys of a record without the rows. A report that claims the key of a stale row moves the row to its record.

A writer of v0.1.0 does not write `claimed_host_keys`. So during the overlap, a record that v0.1.0 overwrote with no keys can get back a key that its guest no longer has. The front door then fails closed for that machine, because the guest shows another key, until the machine reports `running` again.

The release after removes the shared lock, the second lookup, and the check of stale rows.

## env_vars

Migration `0029_env_vars.sql`. One row per env var or secret of an org ([the env var API](/reference/vars-api/index.html.md)). A shared or an all value has no owner. A private value has the account that owns it.

| Column | Type | Description |
| --- | --- | --- |
| `id` | `uuid`, primary key | A UUIDv7. A set that replaces the value keeps it. |
| `org_id` | `uuid` | The org. A delete of the org deletes the row. |
| `scope` | `text` | `shared`, `private`, or `all`. |
| `owner_account_id` | `uuid`, can be null | The account of a private value. Null for a shared or an all value. A delete of the account deletes the row. |
| `name` | `text` | The name of the env var. |
| `secret` | `boolean` | `true` for a sealed value. |
| `value` | `text`, can be null | The value of an env var. Null for a secret. |
| `sealed_value` | `bytea`, can be null | The envelope of a secret ([Sealing](/reference/sealing/index.html.md#envelope-format)). Null for an env var. |
| `value_bytes` | `integer` | The size of the value in bytes, also for a secret. The limits read it, so they never open a secret. |
| `updated_by` | `uuid`, can be null | The account that set the value last. Null after the delete of that account. |
| `version` | `bigint` | The row version. A set adds one. A rewrap keeps it, because the value stays the same. |
| `created_at` | `timestamptz` | The first set. |
| `updated_at` | `timestamptz` | The last set. A rewrap keeps it. |

Checks: a row with the scope `private` has an owner, and a row with another scope has none. A secret has `sealed_value` and no `value`, and an env var has `value` and no `sealed_value`. The unique key `(org_id, owner_account_id, name)` treats null owners as equal (`NULLS NOT DISTINCT`). So an org-level name has one row, whatever its scope, and each account has one private row for a name. `SetVar` is one upsert on this key.

`SetVar` seals a secret before it takes a lock, so a slow keyring never holds the org lock. Then it takes three locks in this order. First it takes the key lock of the account of the caller. Then it takes the row lock of the org, with `FOR NO KEY UPDATE`. Then it changes the rows of the org. `DeleteVar` takes the org lock, then the row. Both read the role of the caller again after the org lock. The org lock makes two sets in one org run one after the other, so they cannot pass a limit together.

## dns_zone

Migration `0033_dns.sql`. The SOA serial of the zones of the DNS role ([the DNS role](/reference/dns/index.html.md)). The table holds exactly one row. The migration inserts it, and the primary key `singleton` refuses a second row.

| Column | Type | Default | Description |
| --- | --- | --- | --- |
| `singleton` | `boolean`, primary key | `true` | Always `true`. |
| `serial` | `bigint` | the Unix time of the migration | The SOA serial of every zone. 1 to 4294967295, so it fits in the 32 bits of a DNS serial. |
| `caa_issuers` | `text[]` | `'{letsencrypt.org}'` | The issuers of the CAA records of the zone ([the DNS role](/reference/dns/index.html.md#caa-records)). Migration `0034_acme.sql` adds it. Never null. |
| `version` | `bigint` | `1` | The row version. |
| `updated_at` | `timestamptz` | the time of the migration | The last change of the serial. |

Each change of a zone raises the serial in the transaction of the change. The new serial is the larger of `serial + 1` and the Unix time now. So the serial grows with each change, and it also grows across an install whose data was erased. The update locks the row, so two changes get two serials in the order of their commits. The same transaction publishes the change event `dns.zone_changed` ([events](#events)). These changes raise the serial:

- an update of the install settings that changes `domain` or `public_addresses` ([the admin API](/reference/admin-api/index.html.md#updateinstallconfig)),
- the add or the remove of a TXT record ([dns_txt_records](#dns_txt_records)),
- a change of `caa_issuers` ([the DNS role](/reference/dns/index.html.md#caa-records)).

## dns_txt_records

Migration `0033_dns.sql`. One row per TXT record of the zone of the install, such as the DNS-01 challenge of ACME at `_acme-challenge` ([the DNS role](/reference/dns/index.html.md#txt-records)). A row never changes after its insert. A new value is a new row, so the table has no `version` column.

| Column | Type | Description |
| --- | --- | --- |
| `name` | `text` | The name of the record, relative to the domain, such as `_acme-challenge`. Not empty, and lowercase. |
| `value` | `text` | The value of the record: 1 to 255 bytes. |
| `created_at` | `timestamptz` | When the transaction wrote the row. |

The primary key is `(name, value)`. So a name can hold more than one value, as an ACME order for the domain and its wildcard needs. [Domain rules](/reference/domain-rules/index.html.md#txt-records) lists the rules of a name and a value. The zone holds at most 64 records.

`AddDNSTXTRecord` and `RemoveDNSTXTRecord` of the service layer change the table. Only a leader job calls them, with its lease, so each write checks the lease ([Fenced writes](#fenced-writes)). No job calls them in this release. Each call that adds or removes a row raises the serial in [dns_zone](#dns_zone), and writes an audit entry with the actor `system`:

| Action | Target kind | Target ID | Details |
| --- | --- | --- | --- |
| `dns.add_txt_record` | `dns_txt_record` | the name | `serial`: the serial of the zones with the record |
| `dns.remove_txt_record` | `dns_txt_record` | the name | `serial`: the serial of the zones without the record |

An add of a record that exists, or a remove of a record that does not exist, changes nothing and writes no audit entry. It returns the serial now. An add to a zone with 64 records fails with `failed precondition: the zone holds 64 TXT records, the most it can hold`.
