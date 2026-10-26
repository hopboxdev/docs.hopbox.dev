# Run the control role on a laptop

> How to run the control role on a laptop, call the API and the admin socket, read the audit log, make an admin API key, open the console behind Caddy, and sign the CLI in with curl.

This guide runs `hopboxd` with the control role on a Mac, without root. It reads the install info over the API. Then it sets the domain of the install and rotates the key-encryption key through the admin socket, and finds both actions in the audit log. It also makes an admin API key, opens the console on the laptop, and signs the CLI in without a browser. On a server, the same steps run as root with the default paths.

The release files of hopbox are not public yet. The hopbox team gives them on request. They hold no `hopboxd` for macOS, so this guide builds `hopboxd` from a checkout of the source of hopbox. The source is not public yet either, so this guide is for the people who work on hopbox. To run an install on a server, use [Install hopbox on a Debian 13 server by hand](/guide/install-debian/index.html.md) or [Install hopbox on an OVHcloud server with Terraform](/guide/install-on-ovhcloud/index.html.md).

## What `hopboxd run` does

`hopboxd run` reads one small file, the bootstrap config. The file lists the roles of the server and a few paths ([the bootstrap config of hopboxd](/reference/hopboxd-config/index.html.md)). Everything else about the install lives in the control store and changes through the admin API.

The control role is a chain of units. Each unit needs the units before it, so `hopboxd` starts them in this order and stops them in the reverse order:

1. `postgres`: the bundled Postgres.
2. `store`: the connection pool and the schema migrations.
3. `leader election`: one control process becomes the leader and runs the leader jobs.
4. `api`: the API listener, for every client.
5. `internal listener`: the cluster API over TLS, for the servers of the install ([the internal listener](/reference/internal-listener/index.html.md)).
6. `admin socket`: the admin API for root on this server.

Every call on every listener passes one request gate, which names the caller. Then one policy check in the service layer decides what that caller can do ([the API server](/reference/api/index.html.md)). On the admin socket, the kernel proves the user of the caller. On a laptop, `hopboxd` runs as you, so the socket accepts you without `sudo`.

## What you need

- A checkout of the source of hopbox, built with `make build`.
- The bundled Postgres: `make pg` builds it on the first run and prints its dir.
- curl 7.47 or later, for `--http2-prior-knowledge`.

## Write the bootstrap config

A unix socket path has at most 103 bytes on macOS, so put the sockets in a short dir under `/tmp`. Group and others must not be able to write in the dir.

```sh
mkdir -p /tmp/hb && chmod 700 /tmp/hb
cat > /tmp/hb/hopboxd.toml <<EOF
roles = ["control"]
data_dir = "$HOME/hopbox-dev"
run_dir = "/tmp/hb"

[postgres]
install = "$(make pg)"
EOF
```

The API listener keeps its default address, `127.0.0.1:7080`, and the internal listener keeps `127.0.0.1:7443`.

## Start hopboxd

```sh
./bin/darwin_arm64/hopboxd run --config /tmp/hb/hopboxd.toml
```

The first start creates the Postgres cluster in `~/hopbox-dev/pg`, which takes a few seconds. `hopboxd` writes one JSON log record per line. The record `hopboxd is ready` means that every unit runs. Leave it running, and use a second terminal for the next steps.

The install has no key-encryption key yet, so it has no CA, and the internal listener has no certificate. `hopboxd` logs `the internal listener has no certificate yet` at level warn. The rotation in [Act through the admin socket](#act-through-the-admin-socket) makes the first key. The next TLS handshake of the internal listener then makes the CA.

## Read the install info

`GetInstallInfo` needs no credential:

```sh
curl -s --http2-prior-knowledge \
  'http://127.0.0.1:7080/hopbox.v1.InstallService/GetInstallInfo?encoding=json&message=%7B%7D'
```

```json
{"version":"v0.0.0-20261016030431-02092f6e6161"}
```

The domain is empty, so the JSON leaves it out. Your version names your commit.

An admin call on the API listener needs an admin API key, and this install has none yet. So the gate leaves the call anonymous, and the policy refuses it:

```sh
curl -s -X POST -H 'Content-Type: application/json' -d '{}' \
  http://127.0.0.1:7080/hopbox.admin.v1.InstallConfigService/GetInstallConfig
```

```json
{"code":"unauthenticated","message":"this call needs a credential"}
```

[Make an admin API key](#make-an-admin-api-key) makes one, so that the same call works.

## Act through the admin socket

Set the domain of the install, then rotate the key-encryption key:

```sh
./bin/darwin_arm64/hopboxd admin --config /tmp/hb/hopboxd.toml config set --domain example.com
./bin/darwin_arm64/hopboxd admin --config /tmp/hb/hopboxd.toml kek rotate
```

```text
domain	example.com
public_addresses	-
acme_directory_url	https://acme-v02.api.letsencrypt.org/directory
acme_email	-
acme_terms_agreed	false
mail_transport	smtp
version	2
update_time	2026-10-16T09:00:00Z
file:1
```

`admin` finds the socket in the `run_dir` of the config, here `/tmp/hb/admin.sock`. The rotation wrote the key file `~/hopbox-dev/kek/1.key`. Read the install info again, and it now has the domain:

```sh
curl -s --http2-prior-knowledge \
  'http://127.0.0.1:7080/hopbox.v1.InstallService/GetInstallInfo?encoding=json&message=%7B%7D'
```

```json
{"domain":"example.com","version":"v0.0.0-20261016030431-02092f6e6161"}
```

## Find the actions in the audit log

Every admin call writes an audit entry. Read the entries with the `psql` of the bundled Postgres, over its socket:

```sh
"$(make pg)/bin/psql" -h /tmp/hb/pg -U hopbox hopbox \
  -c "SELECT actor_kind, action, target_id, details FROM audit_log ORDER BY id"
```

```text
 actor_kind |        action         | target_id |      details
------------+-----------------------+-----------+------------------------------------------------------------------------
 local_root | install_config.update |           | {"domain": "example.com", "peer_pid": "49450", "peer_uid": "501", "old_domain": "", ...}
 local_root | keyring.rotate.start  | file:1    | {"peer_pid": "49449", "peer_uid": "501", "idempotency": "5046b3c8...2fccc2d"}
 local_root | keyring.rotate        | file:1    | {"peer_pid": "49449", "peer_uid": "501", "idempotency": "5046b3c8...2fccc2d"}
(3 rows)
```

The output here is cut short. The entry `install_config.update` holds each setting before and after the change, such as `old_domain` and `domain`. A rotation writes two entries: `keyring.rotate.start` before it writes the key file, and `keyring.rotate` after it ([the admin API](/reference/admin-api/index.html.md#rotatekek)). `idempotency` is the SHA-256 of the idempotency key of the call. `peer_uid` and `peer_pid` name the user and the process that called through the socket.

## Read the metrics

`hopboxd` counts every API call in its Prometheus metrics on `127.0.0.1:9464` ([Metrics](/reference/metrics/index.html.md)):

```sh
curl -s http://127.0.0.1:9464/metrics | grep '^hopbox_api_calls_total'
```

```text
hopbox_api_calls_total{code="ok",listener="admin_socket",procedure="/hopbox.admin.v1.InstallConfigService/UpdateInstallConfig"} 1
hopbox_api_calls_total{code="ok",listener="admin_socket",procedure="/hopbox.admin.v1.KeyringService/RotateKEK"} 1
hopbox_api_calls_total{code="ok",listener="api",procedure="/hopbox.v1.InstallService/GetInstallInfo"} 2
hopbox_api_calls_total{code="unauthenticated",listener="api",procedure="/hopbox.admin.v1.InstallConfigService/GetInstallConfig"} 1
```

## Make an admin API key

The admin API on the API listener takes an admin API key ([CLI tokens, API keys, admin API keys, and SSH keys](/reference/credentials/index.html.md#admin-api-keys)). Only root on a control server makes one, through the admin socket. On the laptop, that is you:

```sh
admin() { ./bin/darwin_arm64/hopboxd admin --config /tmp/hb/hopboxd.toml "$@"; }
ADMIN_KEY="$(admin admin-key create --name terraform)"
curl -s -X POST -H 'Content-Type: application/json' -H "Authorization: Bearer $ADMIN_KEY" -d '{}' \
  http://127.0.0.1:7080/hopbox.admin.v1.InstallConfigService/GetInstallConfig
```

```text
The admin key 01a145b6-3fc1-7ba1-a54d-d1dceac38930 (terraform) works until 2027-01-14T17:15:34Z. This is its only copy.
{"config":{"domain":"example.com","version":"2","updateTime":"2026-10-16T17:14:47.254670Z"}}
```

The key operates the install, but the keyring stays with root on the server:

```sh
curl -s -X POST -H 'Content-Type: application/json' -H "Authorization: Bearer $ADMIN_KEY" -d '{}' \
  http://127.0.0.1:7080/hopbox.admin.v1.KeyringService/ListKEKs
```

```json
{"code":"permission_denied","message":"permission denied: a caller of kind admin_key may not do keyring.list"}
```

The hopbox Terraform provider never sends its key to the server. It makes the key itself, and the server registers only its SHA-256. Do the same by hand:

```sh
KEY="hbxadm_$(head -c 32 /dev/urandom | base64 | tr '+/' '-_' | tr -d '=')"
admin admin-key register --name bootstrap --sha256 "$(printf '%s' "$KEY" | shasum -a 256 | cut -d' ' -f1)"
admin admin-key list
```

```text
01a145b6-400d-7c41-979e-14666fc2fe21	bootstrap	2027-01-14T17:15:34Z	never
01a145b6-3fc1-7ba1-a54d-d1dceac38930	terraform	2027-01-14T17:15:34Z	2026-10-16T17:15:34Z
01a145b6-400d-7c41-979e-14666fc2fe21	bootstrap	2027-01-14T17:15:34Z	never
```

The first line is the output of `register`, and the next two are the list. `$KEY` now calls the admin API. Run `register` again with the same hash, and it prints the same key: a repeated install makes no second key. Delete a key with `admin admin-key revoke --id <id>`. The audit log has `admin_key.create`, `admin_key.register`, and `admin_key.list` with the actor `local_root`.

## Open the console on the laptop

`make build` builds the console into `hopboxd`, after `corepack pnpm install`. The console needs HTTPS. Its session cookie has the `Secure` flag, and each call with the cookie must come from the origin `https://<domain>` ([Sessions, cookies, and sign-in links](/reference/sessions/index.html.md#calls-with-the-cookie)). On an install, the edge ends TLS. On the laptop, [Caddy](https://caddyserver.com) 2.8 or later does that job.

With `hopboxd` running, ask the API listener for a page of the console:

```sh
curl -sI http://127.0.0.1:7080/orgs | grep -i -e '^HTTP' -e '^content-security-policy'
```

```text
HTTP/1.1 200 OK
Content-Security-Policy: default-src 'none'; script-src 'self'; style-src 'self' 'unsafe-inline'; img-src 'self' data:; font-src 'self'; connect-src 'self'; manifest-src 'self'; base-uri 'none'; form-action 'self'; frame-ancestors 'none'
```

If you get `404` and the text `This hopboxd has no console`, the binary came from `go build`. Run `make build` again.

A browser treats each name under `localhost` as the laptop itself. So `hop.localhost` works as the domain of the install without a DNS record:

```sh
admin config set --domain hop.localhost
```

```text
domain	hop.localhost
public_addresses	-
...
```

Save this file as `/tmp/hb/Caddyfile`:

```text
hop.localhost {
	tls internal
	reverse_proxy 127.0.0.1:7080
}
```

`tls internal` makes Caddy sign a certificate for `hop.localhost` with its own local CA. Make the browser trust that CA once, then start Caddy in a third terminal:

```sh
caddy trust
caddy run --config /tmp/hb/Caddyfile
```

`caddy trust` asks for your password, because it adds the CA to the trust store of macOS. Caddy listens on port 443 and sends each request on to the API listener. It keeps the `Host` and `Origin` headers, so the checks of the session cookie pass.

Open `https://hop.localhost/` in the browser. The console sends you to `/signin`. The install has no SMTP server and no OAuth app yet, so the page shows the command for a login link. Make the link, and open it in the browser:

```sh
admin login-link --email root@example.org
```

```text
https://hop.localhost/auth/login-link#token=tOzJ8YQyibHutrU34tIWLHnGd5XlzYs4WoIBXgSDh5I
The link signs in root@example.org once, until 2026-10-21T14:57:56Z.
```

The page reads the token after `#token=`, removes it from the address bar, and signs you in. Then the steps of [Use the console](/guide/console/index.html.md#create-a-machine-and-open-a-shell) work, but a machine needs a node. The laptop runs only the control role, so a create fails with an error about capacity. Enroll a node first ([Enroll the node role on one server](/guide/enroll-node/index.html.md)). The CLI signs in to the same console with `./bin/darwin_arm64/hopbox login --url https://hop.localhost`.

Stop Caddy with Ctrl-C. To remove the CA of Caddy from the trust store, run `caddy untrust`. To go back to the domain of the other guides, run `admin config set --domain example.com`.

## Sign in the CLI with curl

A person approves a sign-in of the hopbox CLI on the console page `/device` ([Install the hopbox CLI](/guide/install-cli/index.html.md#sign-in-to-an-install)). To test the device flow without a browser, approve it with curl, as that page does. This section needs the domain `example.com`, a signed-in instance admin, and the shell function `browser` of [First sign-in on a new install](/guide/first-sign-in/index.html.md#sign-in-with-the-link). [CLI tokens, API keys, admin API keys, and SSH keys](/reference/credentials/index.html.md#the-device-flow) gives each step of the flow.

Build the CLI, and give it a short name. On Linux, use `bin/linux_amd64` or `bin/linux_arm64`:

```sh
make build
hopbox() { ./bin/darwin_arm64/hopbox "$@"; }
API=http://127.0.0.1:7080
```

Start the sign-in in the background, so the same shell can approve it:

```sh
hopbox login --url $API > /tmp/hb/login.out 2> /tmp/hb/login.err &
sleep 1; cat /tmp/hb/login.err
```

```text
To sign in, open this page in a browser and approve the code KWJQ-LBZV:

  https://example.com/device?code=KWJQ-LBZV

Waiting for the approval. Press Ctrl-C to stop.
```

The URL uses plain `http`, which the CLI accepts only for a loopback address. Now be the person at the console. Look up the code, check that the client is your own CLI, and approve it with the start time that the lookup showed:

```sh
CODE=$(sed -n 's/.*approve the code \([A-Z-]*\):/\1/p' /tmp/hb/login.err)
browser -d "{\"userCode\":\"$CODE\"}" $API/hopbox.v1.AuthService/GetDeviceAuthorization | tee /tmp/hb/shown.json
CREATED=$(python3 -c 'import json; print(json.load(open("/tmp/hb/shown.json"))["createTime"])')
browser -d "{\"userCode\":\"$CODE\",\"createTime\":\"$CREATED\"}" $API/hopbox.v1.AuthService/ApproveDevice
```

```json
{"clientName":"hopbox CLI on user1-laptop", "clientIp":"127.0.0.1", "createTime":"2026-10-17T15:30:36.792730Z", "expireTime":"2026-10-17T15:40:36.792730Z"}
{"clientName":"hopbox CLI on user1-laptop", "clientIp":"127.0.0.1", "createTime":"2026-10-17T15:30:36.792730Z"}
```

The approval needs a sign-in within the last 10 minutes ([Sessions, cookies, and sign-in links](/reference/sessions/index.html.md#recent-sign-in)). If your sign-in is older, the call fails with `permission_denied`:

```json
{"code":"permission_denied","message":"permission denied: this action needs a sign-in within the last 10 minutes. Sign in again, then retry","details":[{"type":"hopbox.v1.RecentSignInRequired","value":"CgMI2AQ","debug":{"maxAge":"600s"}}]}
```

The code keeps waiting. Print a new login link, and put the part of the link after `#token=` in `TOKEN`. Then sign in with your old cookie, as a browser does. The new sign-in ends the old session and starts a new one. Take the new cookie, and approve again:

```sh
admin login-link --email root@example.org
TOKEN='<the part of the link after #token=>'
curl -s -D /tmp/hb/headers -X POST -H 'Content-Type: application/json' \
  -H 'Connect-Protocol-Version: 1' -H 'Origin: https://example.com' -H "Cookie: $COOKIE" \
  -d "{\"token\":\"$TOKEN\"}" $API/hopbox.v1.AuthService/VerifyLoginLink
COOKIE=$(sed -n 's/^Set-Cookie: \(__Host-hopbox_session=[^;]*\).*/\1/p' /tmp/hb/headers)
browser -d "{\"userCode\":\"$CODE\",\"createTime\":\"$CREATED\"}" $API/hopbox.v1.AuthService/ApproveDevice
```

The CLI polls every 5 seconds, so it sees the approval within a few seconds:

```sh
wait; cat /tmp/hb/login.out
```

```text
Signed in to http://127.0.0.1:7080 as root@example.org (unverified). The profile default is active.
Added the SSH key ~/.ssh/hopbox_ed25519 (SHA256:rkEGrocWsFODhePtstCZFcyj1D0BUmGYu1rexwam/mo) to your account.
```

To see a refresh of the CLI tokens at once, mark the access token as ended in the file. On Linux, use `sed -i` without `''`:

```sh
sed -i '' 's/^\(    access_token_expire_time = \).*/\12026-01-01T00:00:00Z/' ~/.config/hopbox/credentials.toml
hopbox --debug whoami 2>&1 | head -1
```

```text
time=2026-10-17T17:31:00.445+02:00 level=DEBUG msg="refreshed the CLI tokens" cli_sign_in=01a14a7c-abae-7e4d-bde7-34c1e1d6870d
```

## Call the access calls on the laptop

The API listener of the laptop listens on `127.0.0.1:7080` without TLS. So the grpcurl calls of [Run commands and copy files in a machine](/guide/exec-and-files/index.html.md) need `-plaintext` there, and an API key of your org in `KEY`:

```sh
API=127.0.0.1:7080
hb() { grpcurl -plaintext -H "Authorization: Bearer $KEY" "$@"; }
```

Each access call but the log needs a running machine, so the laptop also needs a node ([Enroll the node role on one server](/guide/enroll-node/index.html.md)). Without one, a new machine stays `pending`, and the calls answer `failed_precondition`.

## Add a second control server

Two control servers share one control store ([Run the control store on an external Postgres](/guide/external-postgres/index.html.md)), and they need the same key-encryption keys. Copy the key dir of the first server to the second one before the second one starts for the first time. Do not run `hopboxd install` on it first with the control role and no keys, because that makes a first key that the other server does not have. The start of the control role checks that its key files open the install CA, and fails when they do not.

On the first server, `cp1`, copy the dir as root. The dir has the mode `0700`, and each key file has the mode `0400` and belongs to root:

```sh
sudo rsync -a /var/lib/hopbox/kek/ root@cp2:/var/lib/hopbox/kek/
ssh root@cp2 ls -l /var/lib/hopbox/kek
```

```text
total 4
-r-------- 1 root root 32 Oct 16 09:00 1.key
```

Start `hopboxd` on `cp2`. If the keys are wrong, it stops with `control: the key files in /var/lib/hopbox/kek do not open the install CA of the control store: ... Copy the key files of a running control server there`.

Later, add a key to both servers with a stage and a promote. A plain `kek rotate` fails while the other server is live:

```sh
sudo hopboxd admin kek stage
sudo scp -p /var/lib/hopbox/kek/2.staged root@cp2:/var/lib/hopbox/kek/2.staged
sudo hopboxd admin kek promote file:2
sudo hopboxd admin kek rewrap
```

```text
file:2	/var/lib/hopbox/kek/2.staged
file:2
rewrapped 3 to file:2
file:1	0	
file:2	3	current
```

[Sealing](/reference/sealing/index.html.md#several-control-servers) gives the steps and the errors. The hopbox Terraform provider copies the key files to a new control server that it sets up ([hopbox Terraform provider](/reference/terraform-provider/index.html.md#control-servers)). You copy a staged key yourself, over SSH.

## Stop hopboxd

Press Ctrl-C in the first terminal. `hopboxd` stops the units in the reverse order: the admin socket, the internal listener, the API, the leader election, the store, Postgres, and the metrics listener last. It removes `/tmp/hb/admin.sock` and exits with status 0. The next start keeps the cluster, the domain, and the key.

To start again from nothing, stop `hopboxd`, then remove the dirs `~/hopbox-dev` and `/tmp/hb`.

## Next

The node role runs machines, so it needs Linux with KVM, and its enrollment fails on a Mac. [Enroll the node role on one server](/guide/enroll-node/index.html.md) enrolls the node role of a Linux server with the control role of the same server. [Serve the domain of an install with the DNS role](/guide/dns-role/index.html.md) runs the DNS role.
