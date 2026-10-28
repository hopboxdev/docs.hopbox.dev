# The bundled Postgres

> The bundled Postgres: its build, tools/pgdist, paths, OS users, roles, logs, errors, and the schema migrations.

`hopboxd` brings its own Postgres 18 and runs it as a supervised child process. This page lists the build of that Postgres, with the OpenSSL that it links to, and the tool that builds it. It also lists the paths, OS users, database roles, server parameters, logs, and errors of Postgres. The last part covers the schema migrations of the control store.

The package `internal/pgsup` supervises Postgres, and the package `internal/store` runs the migrations. `hopboxd run` starts both for the control role, with the paths on this page ([hopboxd](/reference/hopboxd/index.html.md#run)). The keys `data_dir`, `run_dir`, `postgres.install`, and `postgres.user` of the bootstrap config set the paths and the OS user ([the bootstrap config of hopboxd](/reference/hopboxd-config/index.html.md#keys)). With `postgres.url`, the control role uses an external Postgres in place of the bundled one ([External Postgres](#external-postgres)).

## Version

| Item | Value |
| --- | --- |
| Postgres version | 18.6 |
| Source | `https://ftp.postgresql.org/pub/source/v18.6/postgresql-18.6.tar.bz2` |
| sha256 of the source | `555610c24d53e4316da5b7d3fc25c279d96856d5e0e23ee308c328c5fa881d9f` |
| OpenSSL version | 3.5.9, a long-term support release |
| OpenSSL source | `https://github.com/openssl/openssl/releases/download/openssl-3.5.9/openssl-3.5.9.tar.gz` |
| sha256 of the OpenSSL source | `603f5602e2eef00d77fbd429d34dcd5822bb301757a1bc9cdb24c670f1eb859a` |

The file `build/postgres/source.env` pins each version, URL, and hash. `tools/pgdist` refuses a download with another hash. A fix of OpenSSL reaches the servers as a change of `source.env` in a new release of `hopboxd`.

## Build recipe

The folder `build/postgres` holds the recipe:

| File | What it does |
| --- | --- |
| `source.env` | Pins the source tarballs, one `key=value` per line: `version`, `url`, and `sha256` for Postgres, and `openssl_version`, `openssl_url`, and `openssl_sha256` for OpenSSL. |
| `build.sh` | `build.sh <postgres tarball> <openssl tarball> <empty output dir>`. Builds OpenSSL, then Postgres, and checks the result. |
| `Dockerfile` | Runs `build.sh` for Linux in Debian 12 (`debian:bookworm-slim`, pinned by digest). |

`build.sh` first builds the libraries of OpenSSL. `pg_dump` and `pg_restore` need them to reach an external Postgres over TLS, for `hopboxd admin db export` ([hopboxd](/reference/hopboxd/index.html.md#admin-db-export)). It runs the `Configure` of OpenSSL with these flags:

| Flag | Reason |
| --- | --- |
| `--prefix=/hopbox-ssl-prefix` | A path that no server has, and that only root can make. On macOS, the prefix is a temp dir of the build, because macOS loads a library by its install name, and the Postgres build runs programs that link to OpenSSL. |
| `--openssldir=/etc/ssl` | The config dir of OpenSSL on Debian. A client that asks for the roots of the system, such as `sslrootcert=system`, finds them on a server. `hopboxd admin db export` runs `pg_dump` and `pg_restore` with `OPENSSL_CONF=/dev/null`, so they read no `openssl.cnf` of the host ([hopboxd](/reference/hopboxd/index.html.md#admin-db-export)). |
| `--libdir=lib` | The libraries go to `lib/`, never `lib64/`. |
| `shared` | `libssl` and `libcrypto` are shared libraries, so each program of the install links to one copy. |
| `no-tests` | The build makes no test programs. |

It builds only the libraries (`make build_libs`) and installs them with their headers into a stage dir. Then it runs the `configure` of Postgres with these flags:

| Flag | Reason |
| --- | --- |
| `--disable-rpath` | `build.sh` sets relative library paths after the build. |
| `--without-icu` | The install must not need ICU. The cluster uses the builtin collation provider. |
| `--without-readline` | The install must not need readline. |
| `--without-zlib` | The install must not need zlib. `pg_dump` and `pg_basebackup` cannot compress. |
| `--with-ssl=openssl` | `libpq` speaks TLS, so `pg_dump` and `pg_restore` can use `sslmode=verify-full`. The server of the bundled Postgres keeps `ssl` off, because it listens only on its unix socket. |
| `--with-includes` and `--with-libraries` | They name the headers and the libraries of the OpenSSL stage dir, not those of the system. |

ICU, readline, and zlib are the only libraries that `configure` turns on by default. Every other optional library is off. During the build of Postgres, `LD_LIBRARY_PATH` names the OpenSSL stage dir, because the build runs programs that it links, such as `zic`.

After `make install-strip`, `build.sh` deletes `include/`, `lib/postgresql/pgxs/`, `lib/pkgconfig/`, and the static libraries. It copies the versioned files of `libssl` and `libcrypto` into `lib/`, without their symlinks and headers, and strips them. It then makes the install relocatable:

- On Linux, `patchelf` sets the run path of each file that links to `libpq`, `libecpg`, `libpgtypes`, `libssl`, or `libcrypto`. The run path is `$ORIGIN/../lib` in `bin/`, `$ORIGIN/..` in `lib/postgresql/`, and `$ORIGIN` in `lib/`.
- On macOS, `install_name_tool` changes the install name of each library of Postgres and OpenSSL to `@rpath/<name>`, and adds the run path `@loader_path/../lib`, `@loader_path/..`, or `@loader_path/.`. `codesign` then signs each changed file again.

`build.sh` checks each file. A link to a library outside the install, other than the C library, fails the build. An absolute library path also fails the build. At the end, it runs `--version` on `postgres`, `initdb`, `pg_ctl`, `psql`, `pg_dump`, `pg_restore`, and `pg_upgrade` from the output dir. That dir is not the configured prefix, so the runs prove that the install is relocatable. Last, it runs `psql` with `sslmode=require`. A `libpq` without TLS refuses that mode before it connects, and the build fails with `build.sh: libpq has no TLS support`.

### Targets

| Target | Builder | Runs on | Used by |
| --- | --- | --- | --- |
| `linux/amd64` | Docker, Debian 12 | glibc 2.36 and later, for example Debian 13 and Ubuntu 24.04 | the release bundle, CI |
| `darwin/arm64` | native | macOS on Apple Silicon | tests on a Mac |

The Linux build uses Debian 12 because a binary built against an older glibc runs on newer ones.

### The install

An install is one folder:

```text
bin/      postgres, initdb, pg_ctl, psql, pg_dump, pg_restore, pg_upgrade, and the other client programs
lib/      libpq, libecpg, libpgtypes, libssl, libcrypto, and lib/postgresql/ with the server modules
share/    share/postgresql/ with the catalog files, time zones, and extensions
```

It links only to its own libraries and to the C library of the system: `libc`, `libm`, and the dynamic loader on Linux, and `libSystem` on macOS. The OpenSSL libraries are `lib/libssl.so.3` and `lib/libcrypto.so.3` on Linux, and `lib/libssl.3.dylib` and `lib/libcrypto.3.dylib` on macOS.

## pgdist

`tools/pgdist` builds the install from the recipe, caches it, and prints its path on stdout. Progress logs go to stderr.

```sh
go run ./tools/pgdist [flags]
```

| Flag | Type | Default | Description |
| --- | --- | --- | --- |
| `-target` | `os/arch` | the host, for example `darwin/arm64` | The platform of the build: `darwin/arm64` or `linux/amd64`. |
| `-builder` | string | `docker` for a `linux` target, `native` for `darwin` | `native` runs `build.sh` on the host. It builds only the host target. `docker` runs the `Dockerfile` with `docker build --platform <target> --output type=local`. It builds only `linux` targets. |
| `-cache` | dir | `$HOPBOX_PGDIST_CACHE`, else `hopbox/pgdist` in the user cache dir | Where builds and source tarballs live. |
| `-recipe` | dir | `build/postgres` in the first parent dir of the current dir that has it | The recipe folder. |

| Environment variable | Description |
| --- | --- |
| `HOPBOX_PGDIST_CACHE` | The cache dir. `-cache` overrides it. |

The user cache dir is `~/Library/Caches` on macOS, and `$XDG_CACHE_HOME` or `~/.cache` on Linux.

The cache holds:

| Path | Contents |
| --- | --- |
| `<cache>/postgres-<version>-<os>-<arch>-<hash>/` | One install. `<hash>` is the first 12 hex digits of a sha256 over every file of the recipe folder. Thus each change to the recipe makes a new build. |
| `<cache>/postgres-<...>.lock` | A lock file. Parallel runs of pgdist wait for each other. |
| `<cache>/postgres-<...>.tmp/` | A build in progress. After a good build, pgdist renames it to the install dir. |
| `<cache>/src/postgresql-<version>.tar.bz2` | The source tarball of Postgres. If its hash does not match, pgdist downloads it again. |
| `<cache>/src/openssl-<version>.tar.gz` | The source tarball of OpenSSL, with the same check. |

Two runs of pgdist can share one cache, such as two CI jobs on one runner. A run writes each download to a temp file in `src/` and renames it into place, so a second run never reads half a file. The lock file makes the second run wait for the build of the first, and then reuse it.

If the install dir exists, pgdist prints its path and builds nothing. On each run, pgdist sets the modes of the install to `u=rwX,go=rX`: `0755` for dirs and programs, `0644` for other files. The OS user `hopbox-pg` must read and run them, whatever modes the build or the Docker export left. It exits with status 1 and an error log on any failure.

What each builder needs:

| Builder | Needs |
| --- | --- |
| `native` on macOS | the Xcode command line tools (clang, make, bison, flex, perl, `strip`, `install_name_tool`, `codesign`) |
| `native` on Linux | gcc, make, bison, flex, perl, patchelf, readelf, strip |
| `docker` | Docker with BuildKit |

Example:

```sh
$ go run ./tools/pgdist
/Users/me/Library/Caches/hopbox/pgdist/postgres-18.6-darwin-arm64-82e44458122c
```

## Paths on a server

| Path | Owner | Mode | Contents |
| --- | --- | --- | --- |
| `/var/lib/hopbox/pg` | `hopbox-pg` | `0700` | The Postgres state dir. If it does not exist, pgsup creates it. |
| `/var/lib/hopbox/pg/data` | `hopbox-pg` | `0700` | The cluster, which is the data dir of Postgres. |
| `/var/lib/hopbox/pg/data.init` | `hopbox-pg` | `0700` | A cluster that `initdb` is creating. It exists only during the first start. |
| `/run/hopbox/pg` | `hopbox-pg` | `0700` | The socket dir. It holds `.s.PGSQL.5432` and its lock file. |

pgsup creates a missing parent dir, such as `/var/lib/hopbox` or `/run/hopbox`, with mode `0755`. A parent dir that exists must let `hopbox-pg` enter it. pgsup sets the mode `0700` on the state dir and the socket dir on each start. `hopbox-pg` must also be able to read the Postgres install. The path of the socket file must be at most 103 bytes, the limit of unix sockets on macOS.

## OS users and database roles

| Name | Kind | Description |
| --- | --- | --- |
| `hopbox-pg` | OS user | Postgres refuses to run as root. So when `hopboxd` runs as root, Postgres runs as this user. The user must exist before the first start. When `hopboxd` is not root, Postgres runs as the user of `hopboxd`. |
| `hopbox` | database role | The bootstrap superuser of the cluster. `hopboxd` connects with this role. |
| `hopbox` | database | The control store database. On each start, pgsup makes sure that it exists. |

Postgres accepts connections only on the unix socket, and only with peer authentication. Peer authentication takes the OS user of the client from the kernel. The two files that pgsup writes after `initdb` set this up:

```text
# pg_hba.conf
local all all peer map=hopbox

# pg_ident.conf: maps the OS user of hopboxd to the role hopbox
hopbox root hopbox
```

When `hopboxd` is not root, the second line names the user of `hopboxd` in place of `root`. pgsup writes both files on each start, so a change of the user of `hopboxd` takes effect at the next start. pgsup syncs them to disk, because an empty `pg_hba.conf` after a power loss locks `hopboxd` out.

As root, pgsup works in dirs that `hopbox-pg` owns, and `hopbox-pg` can plant symlinks there. So pgsup does each file operation in the state dir through a handle on that dir that never leaves it. It reads only regular files of at most 4 KiB. It writes a file under a new temp name with `O_EXCL`, changes the owner through the open file, and renames it into place. It never opens an existing path for writing, and never follows a symlink with `chown` or `chmod`.

## initdb flags

pgsup runs `initdb` as the Postgres OS user, with the environment `LC_ALL=C` and `TZ=UTC`:

| Flag | Reason |
| --- | --- |
| `--pgdata=<dir>/data.init` | pgsup renames the dir to `data` only after `initdb` succeeds, so a crash never leaves a partial cluster in `data`. |
| `--username=hopbox` | The bootstrap superuser. |
| `--encoding=UTF8` | |
| `--locale=C` | No dependency on the locales of the OS. |
| `--locale-provider=builtin` and `--builtin-locale=C.UTF-8` | A collation that stays the same across OS and ICU updates. |
| `--auth-local=peer` and `--auth-host=reject` | pgsup replaces `pg_hba.conf` right after, but no other method is ever on. |
| `--no-instructions` | |

Postgres 18 turns on data checksums by default.

## Server parameters

pgsup starts `bin/postgres -D <dir>/data` with these parameters as `-c name=value`. A caller can pass extra parameters, but not these, and not `config_file`, `data_directory`, `hba_file`, `ident_file`, or `unix_socket_group`. Start refuses such a parameter, because it can undo the socket-only and peer-only setup. Postgres reads parameter names without regard to case, so the check does the same: `Hba_File` is refused too.

| Parameter | Value | Reason |
| --- | --- | --- |
| `listen_addresses` | empty | No TCP listener. |
| `unix_socket_directories` | the socket dir | |
| `unix_socket_permissions` | `0700` | |
| `port` | `5432` | Names the socket file `.s.PGSQL.5432`. |
| `log_destination` | `stderr` | pgsup turns each line into a log record. |
| `logging_collector` | `off` | |
| `log_line_prefix` | `[%p] ` | pgsup reads the process ID from it. |
| `timezone` | `UTC` | |
| `log_timezone` | `UTC` | |

Postgres runs in its own process group, with the environment `LC_ALL=C` and `TZ=UTC`. It does not inherit the environment of `hopboxd`.

## Start, restart, and stop

On start, pgsup does these steps in order:

1. It runs `postgres --version` as the Postgres OS user, with the clean environment of the server, and reads the major version of the binaries. This run also proves that `hopbox-pg` can run the install.
2. It refuses a state dir, data dir, or socket dir that is a symlink.
3. It reads `data/PG_VERSION`. If the file holds another major version, pgsup refuses to start and changes nothing on disk. If the file is missing and `data` is not empty, pgsup also refuses.
4. If `data/postmaster.pid` names a running postmaster, pgsup refuses to start. A Postgres can outlive a `hopboxd` that was killed, because it runs in its own process group. The parent death signal on Linux and the watchdog on macOS make that rare (see below), but not impossible: a postmaster that does not stop in time still holds the data dir. A PID file can also outlive its server, for example after a power loss. So pgsup ignores the file in three cases:
   - No process has the PID.
   - The start time in the file is before the boot of the host.
   - On Linux, the process with the PID is not `postgres`.
5. It creates the state dir and the socket dir. As root, it gives them to `hopbox-pg`.
6. If `data` is missing or empty, it creates the cluster. It deletes any `data.init` from an earlier crash and runs `initdb`. Then it writes `pg_hba.conf` and `pg_ident.conf`, and renames `data.init` to `data`. For a cluster that exists, it writes the two files again.
7. It starts the postmaster and connects every 50 ms until a connection succeeds and `postmaster.pid` names the new process. The first start fails after 60 s. An exit of the postmaster also fails it at once.
8. If the database `hopbox` does not exist, it creates it.

After the first start, pgsup restarts Postgres each time it exits without a stop request. It waits 1 s before the first restart, and doubles the wait after each failed attempt, up to 30 s. A server that ran for more than 1 minute before it exited restarts after 1 s again.

On stop, pgsup sends `SIGINT`, the fast shutdown, which rolls back open transactions and writes a checkpoint. After 30 s, it sends `SIGQUIT`, the immediate shutdown. After 5 s more, it sends `SIGKILL` to the process group of Postgres, because a killed postmaster cannot stop its backends. If the caller stops the wait early, pgsup sends `SIGQUIT` and returns the error of the caller. A second stop call then waits until Postgres exits.

On Linux, the postmaster has a parent death signal: when `hopboxd` dies, for example by `SIGKILL`, the kernel sends the postmaster `SIGINT`, the fast shutdown. So Postgres does not wait for the timeout of the stop of `hopboxd.service` before systemd kills it. The kernel sends this signal at the end of the thread that started the postmaster. So pgsup starts each postmaster from a goroutine that holds its own thread until the postmaster exits. Other systems, such as macOS, have no parent death signal. There, pgsup starts a watchdog process for each postmaster: a `/bin/sh` that waits on a pipe. Only `hopboxd` holds the write end of the pipe. When `hopboxd` dies, the kernel closes that end, and the watchdog sends the postmaster `SIGINT`. The watchdog runs in its own process group, so a Ctrl-C in the terminal does not stop it first. When the postmaster exits, its PID can go to another process. So pgsup kills the watchdog first, and then closes the pipe. If the watchdog cannot start, pgsup logs the warning `postgres will outlive a crash of hopboxd` and runs Postgres without it.

The test `TestPostgresStopsWhenHopboxdDies` kills a supervisor process with `SIGKILL`, and checks that its postmaster exits within 10 s. It runs on Linux and on macOS. `make test-linux` with `PG_LINUX` also runs it as root, where Postgres runs as `hopbox-pg`, and CI does so in the job `test-linux`.

## Upgrades

A release bundle carries the Postgres of its release ([Release bundle](/reference/release-bundle/index.html.md)). An upgrade of `hopboxd` that keeps the major version, for example from 18.6 to 18.7, swaps the binaries with the release: `postgres.install` names `/opt/hopbox/current/postgres`, and the next start of Postgres runs the new binaries on the same cluster.

An upgrade must never start a cluster with binaries of another major version. `hopboxd install` is the last step of each upgrade. Before it switches the release, it runs the check of step 3 of [Start, restart, and stop](#start-restart-and-stop) ([hopboxd](/reference/hopboxd/index.html.md#install)). The Go call is `pgsup.CheckVersion`:

- It runs `postgres --version` of the new release as the Postgres OS user, as a start does.
- It reads `PG_VERSION` of the cluster. A dir without a cluster passes.
- It returns the error of a start with `PG_VERSION` of another major version, and it changes nothing on disk.
- It only reads, so it works while the old `hopboxd` runs Postgres.

The install then stops before it changes the config, the current release, or the units, and the running `hopboxd` keeps running. A move to a new major version needs `pg_upgrade`, which `hopboxd admin db upgrade` runs. That command comes with the first change of the major version.

The first install of a control server runs `initdb` through pgsup: it starts Postgres once, which makes the cluster and the database `hopbox`, and stops it ([hopboxd](/reference/hopboxd/index.html.md#install)). So a broken Postgres stops the install, not the first start of the service.

A control server on an external Postgres skips the checks of this section. The operator upgrades that Postgres ([External Postgres](#external-postgres)).

## Logs

pgsup writes slog records with `component=pgsup`.

| Message | Level | Fields |
| --- | --- | --- |
| `creating postgres cluster` | info | `data_dir` |
| `initdb` | debug | `line`, one record per output line of `initdb` |
| `postgres started` | info | `pid`, `data_dir` |
| `postgres ready` | info | `pid`, `socket_dir` |
| `created database` | info | `database` |
| `postgres exited` | error | `pid`, `err`, `uptime` |
| `restarting postgres` | info | `delay` |
| `postgres restart failed` | error | `err` |
| `postgres did not become ready` | warn | `pid`, `err` |
| `stopping postgres` | info | `pid`, `mode`: `fast`, `immediate`, or `kill` |
| `postgres log lost` | warn | `err`. A log line was longer than 1 MiB. pgsup drops the rest of the log of that run. |
| `postgres stopped` | info | `pid`, `duration` |

Each line that Postgres writes becomes one record. The message is the text of the line, and the fields are `pid`, `severity`, and `source=postgres`. The level follows the severity:

| Postgres severity | slog level |
| --- | --- |
| `PANIC`, `FATAL`, `ERROR` | error |
| `WARNING` | warn |
| `DEBUG1` to `DEBUG5` | debug |
| every other severity | info |

Some lines have no prefix, for example the second line of a long statement. Each of them becomes a record with the message `postgres` and the field `line`.

## Errors

| Error | Cause |
| --- | --- |
| `postgres data dir <dir> holds a Postgres <N> cluster, but the bundled binaries are Postgres <M>, so pgsup will not start it` | `PG_VERSION` names another major version. A major upgrade needs `pg_upgrade`, which `hopboxd` does not run yet. An install or an upgrade gives the same error from `pgsup.CheckVersion` ([Upgrades](#upgrades)). |
| `postgres data dir is not empty and has no PG_VERSION file, so pgsup will not use it: <dir>` | `data` holds files that are not a cluster. |
| `pgsup: look up the postgres OS user "hopbox-pg": ...` | `hopboxd` runs as root, and the user `hopbox-pg` does not exist. |
| `pgsup: socket dir <dir> makes a socket path of <n> bytes, more than the 103 that unix sockets allow` | The socket dir path is too long. |
| `pgsup: hopboxd cannot run <install>/bin/postgres --version: ...` | The install has no working `postgres` binary. |
| `pgsup: the postgres OS user hopbox-pg cannot run <install>/bin/postgres --version: ...` | As root: `hopbox-pg` cannot enter a dir of the install path, or cannot read or run a file of it. |
| `pgsup: <dir> is a symlink, so pgsup will not use it` | The state dir, the data dir, or the socket dir is a symlink. |
| `pgsup: <dir>/PG_VERSION does not hold a version number` | `PG_VERSION` holds something other than digits. |
| `pgsup: read PG_VERSION: ...` or `pgsup: read postmaster.pid: ...` | The file is a symlink that leaves the state dir, or it is not a regular file. |
| `pgsup: initdb: ...` | `initdb` failed. The error ends with the last lines of its output. |
| `start postgres: postgres exited before it was ready: ...` | The postmaster exited during the first start. The Postgres log records give the reason. |
| `start postgres: postgres not ready: context deadline exceeded ...` | The first start took more than 60 s. |
| `postgres supervisor is stopped` | A call after `Stop`. |
| `a postgres server already runs on the data dir, so pgsup will not start another: pid <n> in <dir>/postmaster.pid` | A live process holds the data dir. Stop it, or remove a stale `postmaster.pid` whose PID now belongs to another process. |
| `pgsup: the setting <name> is fixed and cannot be in Settings` | The caller passed a parameter from the [Server parameters](#server-parameters) list, or a protected one. |
| `pgsup: stop postgres: context canceled` | The caller stopped the wait of a stop. Postgres got `SIGQUIT`. |

## External Postgres

Several control servers need one shared Postgres, and the bundled Postgres listens only on its unix socket. So the control role can use an external Postgres, such as managed PostgreSQL of OVHcloud or a Postgres with Patroni. `hopboxd admin db export` moves the store of a one-server install there ([hopboxd](/reference/hopboxd/index.html.md#admin-db-export)). The keys `postgres.url`, `postgres.ca_file`, and `postgres.password_file` of the bootstrap config name it ([the bootstrap config of hopboxd](/reference/hopboxd-config/index.html.md#keys)). [Run the control store on an external Postgres](/guide/external-postgres/index.html.md) sets one up.

With `postgres.url`, the control role changes in these ways:

- The unit `postgres` of `hopboxd run` starts no Postgres. It reads `postgres.ca_file` and `postgres.password_file`, and logs `using the external postgres` at level info, with `url`, the URL without its query, and `ca_file` ([hopboxd](/reference/hopboxd/index.html.md#run)). A file that fails a check stops the start.
- The connection pool, the connection of the leader election, and the connection of the event subscriber connect over TCP with TLS and `sslmode=verify-full`. The client checks that the CA of `postgres.ca_file` signed the certificate of the server, and that the certificate names the host of `postgres.url`. The client sends the password only after that check. A failed check stops the unit `store`, and `hopboxd run` exits with status 1.
- No `<data_dir>/pg`, no `<run_dir>/pg`, and no OS user `hopbox-pg` exist. `hopboxd install` makes none of them, and runs no `initdb` ([hopboxd](/reference/hopboxd/index.html.md#install)).
- `hopboxd install` reads the schema version from the external Postgres. A database without the table `schema_migrations` counts as a new control store, so the install needs `--domain`.
- The upgrades of Postgres and its failover are the job of the operator. After a failover, the connections end. The pool connects again for the next call. The leader election loses its lock with its connection, and takes part again. The event subscriber connects again and reads the events that it missed.

The external Postgres must meet these rules:

| Rule | Reason |
| --- | --- |
| Postgres 18 or later: the major version of the bundled Postgres, or a newer one. | The migrations are tested on Postgres 18. `hopboxd admin db export` refuses an older target, because `pg_restore` of a newer dump into an older server can fail. |
| TLS with a certificate that names the host of `postgres.url`. | `sslmode=verify-full` checks both. The server can refuse connections without TLS. |
| The user of `postgres.url` owns the database, or can create tables in its schema `public`. It needs no superuser. | The migrations create tables, indexes, and sequences. They need no extension. Managed PostgreSQL gives no superuser. |
| Each connection is a session of its own: a direct connection, or a pooler in session mode. A pooler in transaction or statement mode does not work. | The leader election holds the session advisory lock `LockKeyLeader` on one connection, and the event subscriber runs `LISTEN` on one connection ([Control store tables](/reference/control-store/index.html.md#leader-election)). A pooler in transaction mode would hand that session to other clients. |
| The user can set `application_name`, `idle_in_transaction_session_timeout`, and `idle_session_timeout` for its session. | Each connection sets them at its start ([Control store tables](/reference/control-store/index.html.md#connections)). Every user can set them, unless the operator forbids it. |
| Room for the connections of each control process: the pool, with at most 4 connections or the CPU count of the server if that is larger, plus 2. | pgx sizes the pool by default, and the leader election and the event subscriber hold one connection each. |

Errors of the unit `store` on an external Postgres include these:

| Error | Cause |
| --- | --- |
| `store: ping: failed to connect to ...: tls: failed to verify certificate: x509: certificate signed by unknown authority` | The CA of `postgres.ca_file` did not sign the certificate of the server. |
| `store: ping: failed to connect to ...: tls: failed to verify certificate: x509: certificate is valid for <names>, not <host>` | The certificate does not name the host of `postgres.url`. |
| `store: ping: failed to connect to ...: server error: FATAL: password authentication failed for user "<user>" (SQLSTATE 28P01)` | The password of `postgres.password_file` is wrong. |
| `store: migration <NNNN_name>: ERROR: permission denied for schema public (SQLSTATE 42501)` | The user cannot create tables in the database. |

## Schema migrations

The control store schema comes from numbered SQL files in `internal/store/migrations/`. The `hopboxd` binary embeds them.

- Each file is named `NNNN_name.sql`. The name has four digits, a `_`, then lowercase letters, digits, and `_`.
- The numbers start at `0001` and have no gaps. Any other file in the folder fails the load.
- A migration never changes after it ships. A schema change is a new file.
- A migration must not control its transaction. The runner refuses a file with a top-level `BEGIN`, `START`, `COMMIT`, `END`, `ROLLBACK`, or `ABORT` statement. The check skips comments, string literals, quoted names, dollar-quoted bodies, and `BEGIN ATOMIC` function bodies.
- A migration can have one line `-- oldest_schema_version: <n>`. See [Two releases on one store](#two-releases-on-one-store).

Migration `0001_schema_migrations.sql` creates the table that records the applied migrations, and migration `0043_oldest_schema_version.sql` adds its last column:

| Column | Type | Description |
| --- | --- | --- |
| `number` | `integer`, primary key | The number of the migration. |
| `name` | `text` | The name part of the file name. |
| `sha256` | `bytea` | The sha256 of the file. |
| `applied_at` | `timestamptz` | When the migration ran. |
| `oldest_schema_version` | `integer`, null | The oldest schema version that runs with the migration, from the line `-- oldest_schema_version` of its file. It is from 1 to `number`. `NULL` means `number`: only a `hopboxd` that knows the migration runs with it. The rows of the migrations before `0043`, and of each migration without the line, hold `NULL`. |

The rows of this table never change after their insert, so the table has no row version column.

[Control store tables](/reference/control-store/index.html.md) lists the tables of the later migrations.

The runner does these steps:

1. It takes the session advisory lock `LockKeyMigrations` on one connection. Other runners wait for it, so many processes can migrate at the same time.
2. If the table `schema_migrations` exists, it reads the table.
3. It stops with an error in two cases. The first case is a database whose newest migration does not run with this binary ([Two releases on one store](#two-releases-on-one-store)). The second case is an applied migration whose name or hash differs from the file of this binary.
4. If the table `control_processes` exists, it waits for the live control processes that the missing migrations do not run with ([Two releases on one store](#two-releases-on-one-store)).
5. It runs each missing migration in its own transaction, together with the insert of its row. A migration that fails leaves no trace. A database with migrations that this binary does not know, and that run with it, gets no migration.
6. The runner of the control role (`store.MigrateAndRecord`) then writes the first row of its process in `control_processes`, while it still holds the lock. The write reads `schema_migrations` again and fails with the error of the first case of step 3 when the store no longer runs with this `hopboxd`. So no newer `hopboxd` can apply a migration that does not allow this one between the migrations and the first row. The full row, with the keys, maintenance mode, and the server ID, follows at the end of the `store` unit ([hopboxd](/reference/hopboxd/index.html.md#run)). The other runners write no row.
7. It frees the lock. If the unlock fails, it closes the connection, which frees the lock too.

### Two releases on one store

An install can run two releases at the same time: release N and release N-1. For example, a rolling upgrade moves one server at a time, and a rollback moves one server back. So a `hopboxd` of N-1 can run on a store that a `hopboxd` of N migrated.

The schema version of a `hopboxd` is the number of the newest migration that it knows. A release bundle records it as `schema_version` ([Release bundle](/reference/release-bundle/index.html.md#manifest)). Each migration records the oldest schema version that runs with it, in `oldest_schema_version`. A `hopboxd` runs on a store only when its schema version is at least the oldest schema version of the newest migration of the store.

A migration names this version in a line of its own:

```sql
-- oldest_schema_version: 43
ALTER TABLE machines ADD COLUMN note text NOT NULL DEFAULT '';
```

Here, a `hopboxd` that knows migration 43 or later runs on a store with this migration. Name the schema version of release N-1 only when the code of N-1 still works with the schema after the migration. For example, a new column with a default value works with N-1, and a dropped column that N-1 reads does not. A migration without the line runs only with a `hopboxd` that knows it, so a rollback across it stops. The guide Write a change in two releases shows how to split a change that N-1 cannot follow.

The load of the migrations checks the line:

- The value is a migration number from 1 to the number of the migration.
- A file has the line at most once.
- The value is never lower than the oldest schema version of the migration before it. A release that does not run with an earlier migration does not run with a later one either. So the newest migration of a store speaks for all of its migrations.

The rule holds at three points:

- The start of the control role. The runner of step 3 refuses the store, and `hopboxd run` exits.
- A running control role. The `store` unit reads the schema every 10 s (`control.SchemaCheckEvery`). When a newer `hopboxd` applies a migration that does not run with this one, the unit logs an error and stops `hopboxd` ([hopboxd](/reference/hopboxd/index.html.md#run)). systemd starts it again, and the start then refuses the store. The unit also stops `hopboxd` when the store has no `schema_migrations`, or when a read fails because a column or a table is gone (SQLSTATE `42703` or `42P01`). Then the role cannot tell whether it may still write. Other failed reads only log, but 12 failed reads in a row stop `hopboxd` too. A read that takes more than 5 s counts as failed, so a store that does not answer for about 2 minutes stops `hopboxd`. This outlasts the failover of a managed PostgreSQL, which promotes a replica after 60 s. A process that cannot reach the store does not wait for this limit: it fences itself within 15 s, as the paragraph after this list says. The limit of 12 reads is safe only because of this fence. A process that cannot reach the store takes no call and runs no job long before the watch stops it, so it never writes while the other processes no longer count it. Until the next read, the older control role can still write for up to 10 s on the new schema. The runner closes most of this window: it waits for older control processes before it applies a migration, as the paragraph after this list says. It does not see a control role of a release before migration `0045`, because that role records no row. So a migration that does not allow N-1 must also not break a write of N-1 in that window.
- An upgrade or a rollback. `hopboxd upgrade` refuses a release that does not run on the store, before it switches ([hopboxd](/reference/hopboxd/index.html.md#upgrade)).

Since migration `0045`, each control process records itself in `control_processes` ([Control store tables](/reference/control-store/index.html.md#control_processes)). Before the runner applies the missing migrations, it looks at this table, if it exists. The runner takes the newest of the missing migrations and its `oldest_schema_version`. It waits while a control process that is not gone has a schema version below that number. A process that ended without its stop counts as gone only 35 s after its last renewal (`store.ControlProcessGone`: the live window of 20 s and the dead peer timeout of 15 s), because a COMMIT that it sent before its fence can still land until the store ends its session. A process that stopped cleanly counts as gone at once. The runner polls each second, for at most 40 s, 5 s more than that. A control process that stops, or whose row is gone, lets the runner go on.

The runner logs the wait once, with the last message of the log table below. If the wait ends with a live process still in the way, the runner fails with the error `store: a live control process runs a hopboxd that does not run with a new migration: ...` of the table below, and applies nothing. Upgrade or stop the control role of that server, then start again.

Each control process renews its row in one transaction (`store.RecordControlProcess`). If the row is gone, stopped, or has no more than 5 s left to live (`store.ControlProcessFenceMargin`), the write first takes `LockKeyMigrations` shared (`pg_advisory_xact_lock_shared`). So it waits for a runner that holds the lock, and a runner that checks the live rows never misses a process that comes back. The write then reads `schema_migrations`, and fails with the error `store: the database has a migration that does not run with this hopboxd: ...` of the table below when the store no longer runs with this `hopboxd`. Only then does it write the row. A renewal that fails this way fences the process and stops `hopboxd` at once, without the wait for the next read of the schema watch. So a renewal that works means that the row is live and that the store runs with this `hopboxd`.

A process that no renewal reached for 15 s (the live window less a margin of 5 s, counted from the start of the last renewal that worked) fences itself before its row stops being live. While it is fenced, the API refuses every call, runs no control channel, and runs no leader job ([hopboxd](/reference/hopboxd/index.html.md#run)). The fence lifts at the next renewal that works.

A control process of a release before migration `0045` records no row, so the runner cannot see it. The schema watch of that process still stops it within 10 s after the migration.

v0.1.0 and the builds before migration `0043` have an older rule: their control role refuses every store with a migration that it does not know. So no control role of v0.1.0 runs on the store of a later release. A rollback of a control server to v0.1.0 always stops, because migration `0043` has no line `-- oldest_schema_version`. The roles without the control role do not read the store. So the hopbox Terraform provider upgrades an install of v0.1.0 one server at a time too: the first server first, then the nodes ([hopbox Terraform provider](/reference/terraform-provider/index.html.md#rolling-upgrade)). In that time, servers of v0.1.0 work with the control role of the later release. No test covers this pair, because the fleet tests start only from a release with the new rule. So watch the servers of v0.1.0 in the first upgrade of a real install.

| Error | Cause |
| --- | --- |
| `store: the database has a migration that does not run with this hopboxd: the database is at migration <n>, which runs only with a hopboxd that knows migration <m> or later, and this hopboxd knows the migrations up to <k>` | A newer `hopboxd` applied a migration that does not allow this one. Upgrade `hopboxd` to a release that knows migration `<m>`. |
| `the control store changed: store: the database has a migration that does not run with this hopboxd: .... Upgrade hopboxd on this server` | The same, found by the `store` unit while `hopboxd` runs. `hopboxd` stops. |
| `the control store changed: schema_migrations is gone`, or `the control store changed: store: read schema_migrations: ERROR: ... (SQLSTATE 42703)` or `(SQLSTATE 42P01)` | The store of a running control role lost `schema_migrations`, or a column or a table that the read needs. The second form comes when the table goes away between the check that it exists and the read of its rows. `hopboxd` stops. |
| `the schema of the control store did not read 12 times in a row: ...` | The `store` unit could not read the schema for 12 checks in a row, about 2 minutes. `hopboxd` stops. |
| `store: a live control process runs a hopboxd that does not run with a new migration: migration <NNNN_name> runs only with a hopboxd that knows migration <m> or later, and the control process <id> on <host> runs hopboxd <rel>, which knows the migrations up to <n>. Upgrade or stop the control role of that server first` | The runner waited 30 s, and the live control process still runs a `hopboxd` that does not run with the new migration. The runner applied nothing. |
| `store: an applied migration changed, but migrations must only grow: <NNNN_name>` | An applied migration file changed or got a new name. |
| `store: migration <NNNN_name>: ...` | The SQL of a migration failed. |
| `store: migration file "<file>" is not NNNN_name.sql` | A file in the folder has a bad name. |
| `store: migration <file> has number <n>, want <m>` | A gap or a doubled number. |
| `store: migration <file> has a top-level <WORD>, but the runner runs each migration in its own transaction` | The file has a statement that ends or starts a transaction. |
| `store: migration <file>: its oldest_schema_version <n> is not a migration number from 1 to <m>` | The line names a number that is 0 or later than the migration. |
| `store: migration <file>: it has more than one line oldest_schema_version` | The file has the line twice. |
| `store: migration <file>: its oldest_schema_version is <n>, but migration <NNNN> before it runs only with <m> or later. ...` | The line names a lower version than the migration before it. |

These are the log records of the runner and of the `store` unit. Their records have `component=store`.

| Message | Level | Fields |
| --- | --- | --- |
| `applied migration` | info | `number`, `name` |
| `waited for the migration lock` | info | `duration`, after a wait of more than 1 s |
| `the database has migrations that this hopboxd does not know, and they run with it` | info | `applied`, `known`, `oldest_schema_version` |
| `a newer hopboxd applied a migration that does not run with this one, so it stops` | error | `applied`, `oldest_schema_version`, `known` from the schema watch. From a renewal of the row of the process, `component=control` and `err` instead. |
| `reading the schema of the control store failed` | warn | `err`, `failures`: the failed reads in a row. The unit reads again 10 s later, and stops `hopboxd` at 12. |
| `the control store lost the schema that this hopboxd migrated, so it stops` | error | `err` |
| `a live control process runs a hopboxd that does not run with a new migration, so the runner waits for it to end` | warn | `process`, `hostname`, `release`, `schema_version`, `migration`, `oldest_schema_version`. Once for each run of the runner. |

### Advisory lock keys

The file `internal/store/locks.go` holds every advisory lock key of hopbox. Each key has `hopbox` in ASCII in its top six bytes, then a 16-bit purpose number. A test checks the prefix and that no two keys are the same.

| Key | Value | Holder |
| --- | --- | --- |
| `LockKeyMigrations` | `0x686f70626f780001` | The migration runner. The write of a row of `control_processes` that is gone or has no more than 5 s left to live takes it too, as a shared transaction lock, so the runner and such a write wait for each other. |
| `LockKeyLeader` | `0x686f70626f780002` | The leader of the control role, on its own connection ([Control store tables](/reference/control-store/index.html.md#leader-election)). |
| `LockKeySignIn` | `0x686f70626f780010` | A sign-in that can create or link an account, as a transaction lock ([Sessions, cookies, and sign-in links](/reference/sessions/index.html.md#what-a-sign-in-does)). |
| `LockKeyMagicLinks` | `0x686f70626f780011` | A request for a magic link, as a transaction lock, so the rate limits count every request ([Sessions, cookies, and sign-in links](/reference/sessions/index.html.md#rate-limits)). |
| `LockKeyInvitations` | `0x686f70626f780012` | A new invitation to an org, as a transaction lock, so the rate limits per inviter and per email count every invitation ([the org API](/reference/org-api/index.html.md#invitations)). |
| `LockKeyDeviceAuthorizations` | `0x686f70626f780013` | A start of a device sign-in, as a transaction lock, so the limits of the device flow count every start ([CLI tokens, API keys, admin API keys, and SSH keys](/reference/credentials/index.html.md#limits-of-the-device-flow)). |
| `LockKeyDeviceApprovals` | `0x686f70626f780014` | An approval of a device sign-in, as a transaction lock, so the limit of wrong codes counts every failure ([CLI tokens, API keys, admin API keys, and SSH keys](/reference/credentials/index.html.md#limits-of-the-device-flow)). |
| `LockKeyServerRegistry` | `0x686f70626f780015` | A transaction lock of each enrollment. A decommission of the node on the host of the control role takes it before it counts the other servers. So a server that enrolls at that time counts ([the admin API](/reference/admin-api/index.html.md#decommissionserver)). |
| `LockKeyOAuthFlows` | `0x686f70626f780016` | A transaction lock of each start of a sign-in with GitHub or Google that the rate limit counts. The limit thus counts every start ([Sign-in with GitHub and Google](/reference/oauth/index.html.md#rate-limits)). |
| `LockKeyRetiredGuestHostKeys` | `0x686f70626f780017` | Retired, do not reuse. Up to v0.1.0, a report of `running` with SSH host keys took it as an exclusive transaction lock. A `hopboxd` of v0.1.0 can still take it during an upgrade. So in the release after v0.1.0, each report takes it as a shared transaction lock, and migration `0041_route_host_keys.sql` takes it exclusively for its fill. The table `route_host_keys` now keeps each key in one route record ([Control store tables](/reference/control-store/index.html.md#route_host_keys)). The release after that drops the shared lock. |
| `LockKeyNodeSubnets` | `0x686f70626f780018` | A transaction lock of each claim of a node subnet. So two nodes that get their first machine at once never pick the same subnet ([Control store tables](/reference/control-store/index.html.md#node_subnets)). |
| `LockKeyReleaseImages` | `0x686f70626f780019` | A transaction lock of each register of the default image of a release at the start of the control role, around the check for an image of a newer release and the insert ([hopboxd](/reference/hopboxd/index.html.md#the-default-image-of-the-release)). |
| `LockKeyMeshAddresses` | `0x686f70626f78001a` | A transaction lock of each claim of a mesh address. So two servers that join the mesh at once never pick the same address ([Control store tables](/reference/control-store/index.html.md#mesh_members)). |

### Queries

sqlc generates Go code for the queries in `internal/store/queries/*.sql` into the package `internal/store/db`. Its schema input is the migrations folder, so the generated types always match the schema that the runner builds. The file `internal/store/sqlc.yaml` holds the config of sqlc.

sqlc maps column types to Go types with these rules:

| Postgres type | Go type | Go type of a `NULL` column |
| --- | --- | --- |
| `uuid` | `uuid.UUID` from the Go standard library | `*uuid.UUID` |
| `timestamptz` | `time.Time` | `*time.Time` |
| every other type | the default type of sqlc for pgx v5, for example `int64` or `string` | a pointer to it, for example `*string` |

IDs are UUIDv7. The Go code makes them with `uuid.NewV7`, not the database. `make generate` runs sqlc after buf. If `internal/store/db` differs from git, `make check-generate` fails.

## Tests

| Environment variable | Description |
| --- | --- |
| `HOPBOX_PG_DIR` | The Postgres install for tests. `make test` sets it to the output of `tools/pgdist`. If it is not set, tests run `go run ./tools/pgdist` themselves. |

The package `internal/store/storetest` starts one Postgres per test package, with `fsync`, `synchronous_commit`, and `full_page_writes` off. It creates the template database `hopbox_template` with every migration applied. Each test gets its own database, copied from the template with `CREATE DATABASE ... TEMPLATE`, and a cleanup drops it.

`pgsuptest.StartTLS` starts a Postgres of the bundled install that plays an external Postgres, as the user of the test. It runs `initdb` in a temp dir, and takes TCP connections on `127.0.0.1` only over TLS, with a password. Its certificate names `127.0.0.1`, so a client can use `sslmode=verify-full`. It owns a database through a user that is no superuser, as on managed PostgreSQL. It also writes the password file and a second CA that did not sign the certificate. A cleanup stops it. Postgres refuses root, so it skips a test that runs as root. `TestControlRoleExternalPostgres` runs the control role on it and checks that each connection uses TLS. `TestControlRoleExternalPostgresWrongCA` checks that the start fails with the second CA.

A test binary that panics, times out, or gets Ctrl-C or `SIGKILL` skips its cleanup and leaves its dirs behind. Its Postgres stops within seconds, by the parent death signal on Linux or the watchdog on macOS ([Start, restart, and stop](#start-restart-and-stop)). So at each start, `storetest.Main` removes old dirs of earlier runs: socket dirs `/tmp/hbpg-*` and data dirs `storetest-*` in the temp dir. A dir goes only on two conditions: it is older than 10 minutes, and its PID file names no live process. A Postgres that still runs keeps its dirs. A test package calls `storetest.Main` from `TestMain`, and `storetest.New` in each test (the build guide of the repo has an example).
