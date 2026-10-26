# Run the control store on an external Postgres

> How to run the control store on an external Postgres, such as managed PostgreSQL: the database, the CA and password files, the config, and the checks. Also how to move a one-server install there with hopboxd admin db export and maintenance mode.

This guide puts the control store of an install on an external Postgres, such as managed PostgreSQL of OVHcloud. An install needs it before it can run more than one control server, because the bundled Postgres listens only on its unix socket. [The bundled Postgres](/reference/postgres/index.html.md#external-postgres) lists every rule.

The guide first sets up a new control server on an empty database. Then it moves an install with one server from its bundled Postgres to the external Postgres with `hopboxd admin db export`.

## What you need

- A Postgres 18 service with TLS, such as managed PostgreSQL with the plan Production. Managed PostgreSQL offers the versions 15 to 18. Pick 18, because hopbox needs the major version of its bundled Postgres or a newer one.
- The host name of the service, its port, and the CA certificate of its TLS certificate. OVHcloud shows them in the control panel of the service.
- A database for hopbox, and a user that owns it. The default user of managed PostgreSQL, `avnadmin`, is no superuser, and hopbox needs none.
- A Debian 13 server with root access, and a release bundle, as in [Install hopbox on a Debian 13 server by hand](/guide/install-debian/index.html.md#what-you-need).
- An allowed address range of the service that holds the address of the control server. Managed PostgreSQL refuses every other client.

hopbox holds one session per connection: it keeps the leader lock and a `LISTEN` on their own connections. So connect directly to the service, or through a pooler in session mode, never in transaction mode.

## 1. Make the database

Make the database `hopbox` in the service. On managed PostgreSQL, use the control panel or the API of OVHcloud. On another Postgres, run as a user that can make databases:

```sql
CREATE DATABASE hopbox OWNER avnadmin;
```

## 2. Write the CA and the password on the control server

Save the CA certificate of the service and the password of the user as root. The password file must be readable by root alone, or `hopboxd` refuses it. `read -rs` keeps the password out of the screen and the shell history, and the `umask` gives the file the mode `0400`:

```sh
sudo install -d -m 0755 /etc/hopbox
sudo install -m 0644 ca.pem /etc/hopbox/postgres-ca.pem
read -rs PGPW
printf '%s\n' "$PGPW" | sudo sh -c 'umask 0377; cat > /etc/hopbox/postgres-password'
unset PGPW
```

## 3. Name the Postgres in the bootstrap config

Write `[postgres]` into `/etc/hopbox/hopboxd.toml` before the install. The install keeps every key that its flags do not set ([hopboxd](/reference/hopboxd/index.html.md#install)):

```sh
sudo tee /etc/hopbox/hopboxd.toml >/dev/null <<'EOF'
[postgres]
url = "postgres://avnadmin@db.example.org:20184/hopbox?sslmode=verify-full"
ca_file = "/etc/hopbox/postgres-ca.pem"
password_file = "/etc/hopbox/postgres-password"
EOF
sudo chmod 0644 /etc/hopbox/hopboxd.toml
```

Use the host, the port, and the user of your service. The URL must have `sslmode=verify-full` and no password ([the bootstrap config of hopboxd](/reference/hopboxd-config/index.html.md#keys)).

## 4. Install

Run the install of the bundle, as in [Install hopbox on a Debian 13 server by hand](/guide/install-debian/index.html.md#4-install):

```sh
sudo ./hopbox-v0.2.0/hopboxd install --roles control,node --domain example.com
```

The install connects to the database first. The database has no schema yet, so this install is the first one of the install, and it needs `--domain`. It makes no Postgres cluster and no user `hopbox-pg`. A wrong CA, a wrong password, or an address outside the allowed range stops the install before it changes anything.

## 5. Check the connection

The first start of `hopboxd` runs the migrations on the database. Check its log:

```sh
sudo journalctl -u hopboxd -o cat | grep 'using the external postgres'
```

```json
{"time":"2026-10-25T10:00:00Z","level":"INFO","msg":"using the external postgres","url":"postgres://avnadmin@db.example.org:20184/hopbox","ca_file":"/etc/hopbox/postgres-ca.pem"}
```

Check that each connection of `hopboxd` uses TLS. The bundled `psql` speaks TLS too:

```sh
PGPASSWORD="$(sudo cat /etc/hopbox/postgres-password)" /opt/hopbox/current/postgres/bin/psql \
  "postgres://avnadmin@db.example.org:20184/hopbox?sslmode=verify-full&sslrootcert=/etc/hopbox/postgres-ca.pem" \
  -c "SELECT a.application_name, s.ssl FROM pg_stat_activity a JOIN pg_stat_ssl s USING (pid) WHERE a.application_name LIKE 'hopboxd%'"
```

Each row shows `t` in the column `ssl`: the pool `hopboxd`, the leader election `hopboxd-leader`, and the event subscriber `hopboxd-events`.

## If the Postgres fails over

`hopboxd` reads the schema of the control store every 10 s. If 12 reads in a row fail, it stops, and systemd starts it again. A read that takes more than 5 s counts as failed, so a store that does not answer for about 2 minutes stops `hopboxd`. A managed PostgreSQL that promotes a replica after 60 s comes back in time, and `hopboxd` only logs `reading the schema of the control store failed` with the number of failed reads. The limit of 12 reads is safe only because of a second guard. A control process that cannot renew its row in the store for 15 s fences itself. It then refuses every call, with `unavailable` and `Retry-After: 30`, and runs no control channel and no leader job, until it reaches the store again. So it never writes while the other control processes no longer count it. A client that retries, or that goes through another control server, loses nothing. See [the bundled Postgres](/reference/postgres/index.html.md#two-releases-on-one-store).

## See the control processes

Each control process of the install records itself in the control store. The list shows the ones on this store, with their release and when they last renewed their row:

```sh
sudo hopboxd admin control list
```

```text
ID                                    LIVE  HOSTNAME  INTERNAL URL              RELEASE  SCHEMA  KEKS     MAINTENANCE  LAST RENEWAL
01a14404-ce7b-72d9-b18e-5006c3e462db  true  control1  https://172.31.0.11:7443  v0.2.0   45      default  off          2026-10-26T09:00:05Z
```

A process that runs an older release can hold back a migration of a newer one. The migration runner waits up to 40 s for a live process that does not run with the migration, and then it fails with the process, its host, and its release. Upgrade or stop the control role of that server first. [hopboxd](/reference/hopboxd/index.html.md#admin-control-list) describes the command.

When you turn maintenance mode on with several control servers on one store, `hopboxd admin maintenance on` returns after every live control process reported that its calls, control channels, and leader jobs ended. A turn to on waits until each other live process reports that it read maintenance mode at the version that the turn wrote, and that it is drained.

A control process of a release before migration `0045` writes no row, so none of these checks sees it: the migration guard, the maintenance wait, the KEK promote, the decommission check, and the rolling upgrade order. Upgrade every control server before you rely on them.

## If the start fails

`hopboxd` exits with status 1, and systemd restarts it. The error names the cause:

| Error | What to do |
| --- | --- |
| `x509: certificate signed by unknown authority` | `postgres.ca_file` holds another CA. Download the CA of the service again. |
| `x509: certificate is valid for <names>, not <host>` | The host of `postgres.url` is not the name of the service. Use the host name from the control panel, not its IP address. |
| `password authentication failed` | The password file holds another password. A newline at its end is fine. |
| `permission denied for schema public` | The user does not own the database. Make it the owner. |
| `postgres.password_file: ... which lets group or others read it` | Run `sudo chmod 0400 /etc/hopbox/postgres-password`. |

[The bundled Postgres](/reference/postgres/index.html.md#external-postgres) and [the bootstrap config of hopboxd](/reference/hopboxd-config/index.html.md#errors) list every error.

## Move a one-server install to the external Postgres

An install that started on one server keeps its control store in the bundled Postgres. `hopboxd admin db export` copies that store into an empty external Postgres ([hopboxd](/reference/hopboxd/index.html.md#admin-db-export)). The install is in maintenance mode from the start of the copy until a control server runs on the copy ([the API server](/reference/api/index.html.md#maintenance-mode)). In that window, the API takes only the reads that write nothing, `hopboxd admin` takes only `maintenance on` and `off`, and the machine URLs and the SSH front door answer no client. An open SSH session to a machine ends in the window, because its edge server cannot check its access again. The machines keep running.

### 1. Prepare the database and the files

Make an empty database, as in [step 1](#1-make-the-database). Write the CA and the password on the server with the bundled Postgres, as in [step 2](#2-write-the-ca-and-the-password-on-the-control-server). The export uses these files, and so does the control server that runs on the copy.

### 2. Copy the store

Run the export on the server with the bundled Postgres. It prints the config lines for the copy on standard output, so keep them in a file:

```sh
sudo hopboxd admin db export \
  --to 'postgres://avnadmin@db.example.org:20184/hopbox?sslmode=verify-full' \
  --ca-file /etc/hopbox/postgres-ca.pem --password-file /etc/hopbox/postgres-password | sudo tee /root/postgres.toml
```

The export checks the target before it changes anything: the TLS certificate, the major version of Postgres, the right to create tables, and an empty database. Then it turns maintenance mode on, dumps the store, restores it, and checks the row count of each table. It prints lines such as these on standard error:

```text
hopboxd: copied the control store to postgres://avnadmin@db.example.org:20184/hopbox (Postgres 18.1): migration 44, 61 tables, 2317 rows, in 4s.
hopboxd: the install stays in maintenance mode. Stop hopboxd on this server now, so that two control planes never run: sudo systemctl stop hopboxd. Then give the new control server these lines in its bootstrap config and the key files of /var/lib/hopbox/kek, start it, and run hopboxd admin maintenance off there.
```

### 3. Stop hopboxd on the source

Stop `hopboxd` at once, so the bundled store never serves again:

```sh
sudo systemctl stop hopboxd
```

The bundled cluster stays in `/var/lib/hopbox/pg`. The copy is in maintenance mode too, because the flag lives in the store.

Do not roll the source back to a release older than schema version 44 while it is in maintenance mode. Such a release does not know the flag, so it serves writes from the bundled store again.

### 4. Start the control role on the copy

The control role needs the key-encryption keys, because the store holds secrets that they seal ([Sealing](/reference/sealing/index.html.md#key-files)). On the same server, the keys are in place. Add the printed keys to the table `[postgres]` of `/etc/hopbox/hopboxd.toml`, next to its other keys, and start `hopboxd`:

```sh
sudo cat /root/postgres.toml
sudo vi /etc/hopbox/hopboxd.toml
sudo systemctl start hopboxd
```

On another control server, copy the key files of `/var/lib/hopbox/kek` with their mode `0400`, the CA file, and the password file first ([Run the control role on a laptop](/guide/run-hopboxd/index.html.md#add-a-second-control-server)). The start of the control role fails when the key files do not open the install CA. Then put the printed keys and `control` in `roles` into its bootstrap config, and start it.

`hopboxd` starts in maintenance mode, and logs `the install is in maintenance mode: the API takes only reads, and the control role serves no control channel. Turn it off with hopboxd admin maintenance off`.

### 5. Turn maintenance mode off

On the control server that runs on the copy:

```sh
sudo hopboxd admin maintenance off
```

The servers dial their control channels again within 30 s. Their heartbeats bring new grants of the machine leases, and the machine URLs and the SSH front door answer again. Check that each server is `ready`:

```sh
sudo hopboxd admin server list
```

### If the export fails

A refusal before the copy changes nothing, and the error ends with `The export changed nothing`. Fix the cause, such as the CA file, and run the export again.

A failure after the start of maintenance mode leaves the install in maintenance mode on its bundled Postgres. The restore runs in one transaction, so a failed restore leaves the target empty. To go back to the bundled Postgres, run on the source:

```sh
sudo hopboxd admin maintenance off
```

A failed check of the copy leaves the tables of the copy in the target. Drop the database of the target and make it again before the next export, because the export takes only an empty database.
