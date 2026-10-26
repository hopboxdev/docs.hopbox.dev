# Run the edge role and open machine URLs

> How to run the edge role on a Mac and call the API through it. Also how a private URL, its port, and a public URL work.

The edge role takes the HTTPS traffic of an install. It serves the API and the console on the domain of the install, such as `example.com`. It also serves the URL of each machine, such as `box1.example.com`. This guide runs the edge role on a Mac with the test CA Pebble, and calls the API through it. Then it shows how a machine URL works: the private URL with its sign-in, the URL port, and the public URL. [The edge role](/reference/edge/index.html.md) covers the edge role in full.

## How a request reaches its target

The edge role gets the wildcard certificate of the install from the control role, over the control channel. It refuses a TLS handshake without a server name (SNI), or for a name outside the domain. Then it routes each request by its host:

- The domain goes to the internal listener of the control role, over the mutual TLS of the server. The edge role names the address of the client in the header `Hopbox-Client-Addr`. The control role trusts this header only from a server with the edge role. So the rate limits of sign-in count each client.
- `<name>.<domain>` goes to the URL port of the machine `<name>`, port 8000 by default.
- Plain HTTP goes to HTTPS with a `301` answer.

## What you need

- The setup of [Get the wildcard certificate of an install](/guide/acme-certificate/index.html.md) on a Mac: a build of hopbox, the bundled Postgres, Pebble, and `dig`.
- `curl`.

## Run the edge role on a Mac

### Start Pebble

Start Pebble as in [Get the wildcard certificate of an install](/guide/acme-certificate/index.html.md#start-pebble), and leave it running:

```sh
PEBBLE_DIR="$(go env GOMODCACHE)/github.com/letsencrypt/pebble/v2@v2.10.1"
cd "$PEBBLE_DIR" && PEBBLE_VA_NOSLEEP=1 "$(go env GOPATH)/bin/pebble" -config test/config/pebble-config.json -dnsserver 127.0.0.1:15353
```

### Write the bootstrap config

The server runs the control role, the DNS role, and the edge role. Ports 443 and 80 need root, so the edge role listens on high ports of loopback ([the bootstrap config of hopboxd](/reference/hopboxd-config/index.html.md#keys)). A Mac has no nftables, so the SSH front door listens only on its own port, and reaches no machine ([the SSH front door](/reference/ssh-front-door/index.html.md)). Run the `cat` command in the checkout of hopbox, so `make pg` finds the recipe of Postgres:

```sh
PEBBLE_DIR="$(go env GOMODCACHE)/github.com/letsencrypt/pebble/v2@v2.10.1"
mkdir -p ~/hbe/run && chmod 700 ~/hbe ~/hbe/run
cat > ~/hbe/hopboxd.toml <<EOF
roles = ["control", "dns", "edge"]
data_dir = "$HOME/hbe/data"
run_dir = "$HOME/hbe/run"

[dns]
listen = ["127.0.0.1:15353"]

[edge]
https_listen = "127.0.0.1:8443"
http_listen = "127.0.0.1:8080"
ssh_listen = "127.0.0.1:2222"
ssh_redirect = false

[acme]
name_servers = ["127.0.0.1:15353"]
ca_file = "$PEBBLE_DIR/test/certs/pebble.minica.pem"

[postgres]
install = "$(make pg)"
EOF
```

### Start hopboxd

```sh
./bin/darwin_arm64/hopboxd run --config ~/hbe/hopboxd.toml
```

The edge role opens its listeners, and waits for its certificate:

```json
{"level":"INFO","msg":"edge listening","component":"edge","https":"127.0.0.1:8443","http":"127.0.0.1:8080"}
```

Leave it running, and use a third terminal. Set two variables, so the commands stay short:

```sh
H=./bin/darwin_arm64/hopboxd
C=~/hbe/hopboxd.toml
```

### Enroll the DNS role and the edge role

Both roles get their data over the control channel of the server. So the server enrolls with the control role of the same `hopboxd`:

```sh
$H admin --config $C kek rotate
TOKEN="$($H admin --config $C enrollment-token create --roles dns,edge)"
HOPBOX_ENROLLMENT_TOKEN="$TOKEN" $H enroll --config $C
```

### Set the domain and get the certificate

```sh
$H admin --config $C config set --domain example.com --public-address 127.0.0.1 \
  --acme-directory-url https://localhost:14000/dir --acme-email ops@example.org --acme-agree-terms
```

The edge role gets the domain with its first routes. Within some seconds, the ACME job gets the certificate, and the edge role serves it:

```json
{"level":"INFO","msg":"serving the machine URLs of a domain","component":"edge","domain":"example.com","machines":0}
{"level":"INFO","msg":"got the certificate of the install","component":"acme","domain":"example.com","serial":"2a61b67498df3f2a","issuer":"CN=Pebble Intermediate CA 12b00b","not_after":"2027-01-19T11:18:51Z"}
{"level":"INFO","msg":"serving a certificate","component":"edge","domain":"example.com","not_after":"2027-01-19T11:18:51Z","serial":"2a61b67498df3f2a"}
```

### Call the API through the edge

Pebble makes a new root CA at each start. Save it, so curl trusts the certificate. The flag `--resolve` sends each name to the edge role on loopback, with the right server name:

```sh
curl -sk https://localhost:15000/roots/0 > ~/hbe/pebble-root.pem
R=(--resolve example.com:8443:127.0.0.1 --resolve box1.example.com:8443:127.0.0.1 --cacert ~/hbe/pebble-root.pem)
```

Call `InstallService.GetInstallInfo` on the domain. The answer comes from the control role, over HTTP/2, with HSTS:

```text
$ curl -s "${R[@]}" -D - -H 'Content-Type: application/json' -d '{}' \
    https://example.com:8443/hopbox.v1.InstallService/GetInstallInfo
HTTP/2 200
strict-transport-security: max-age=31536000; includeSubDomains
...
{"domain":"example.com", "version":"v0.1.0"}
```

The edge role refuses the requests that it does not serve:

```text
$ curl -s -o /dev/null -w '%{http_code} %{redirect_url}\n' \
    --resolve box1.example.com:8080:127.0.0.1 'http://box1.example.com:8080/a?b=1'
301 https://box1.example.com/a?b=1
$ curl -s -o /dev/null -w '%{http_code} %{redirect_url}\n' "${R[@]}" https://box1.example.com:8443/
303 https://example.com/auth/machine?machine=box1&path=%2F
$ curl -s "${R[@]}" -H 'Content-Type: application/json' -d '{}' \
    https://example.com:8443/hopbox.cluster.v1.EdgeService/AuthorizeMachineRequest
No such page.
$ curl -s --cacert ~/hbe/pebble-root.pem https://127.0.0.1:8443/; echo "exit $?"
exit 35
```

The first redirect goes to port 443, because the edge role of an install listens there. No machine has the name `box1`. The edge role still answers as for a private URL, so a client cannot tell which names have a machine. The last request has no server name, so the handshake fails.

### Sign in through the edge

Sign in with a first-admin login link, as the console does ([First sign-in on a new install](/guide/first-sign-in/index.html.md)). The session cookie comes back through the edge:

```text
$ LINK=$($H admin --config $C login-link --email ann@example.org | grep -o 'https://[^ ]*')
$ curl -s "${R[@]}" -D - -o /dev/null -H 'Content-Type: application/json' -H 'Connect-Protocol-Version: 1' \
    -H 'Origin: https://example.com' -d "{\"token\":\"${LINK#*#token=}\"}" \
    https://example.com:8443/hopbox.v1.AuthService/VerifyLoginLink | grep -i set-cookie
set-cookie: __Host-hopbox_session=<token>; Path=/; Expires=Fri, 20 Nov 2026 11:19:47 GMT; Max-Age=2591999; HttpOnly; Secure; SameSite=Lax
```

The access log of the control role names the address of curl as the peer of each call, such as `"peer":"127.0.0.1:57382"`. The port changes with each run of curl, because it is the port of the client, not the port of the edge role.

### Stop

Press Ctrl-C in the terminal of `hopboxd`, then in the terminal of Pebble. To start from nothing, remove `~/hbe`.

## Open the URL of a machine

A Mac has no KVM, so it runs no machine. These steps need an install with a node. The node gives each machine its address, and the edge role reaches the machine there ([the node network](/reference/node-network/index.html.md)).

The steps call the API with an API key ([Use the hopbox CLI](/guide/cli/index.html.md#make-an-api-key-for-a-script)). Keep the key in `HOPBOX_API_KEY`, and define a function:

```sh
api() {
  curl -s -X POST -H 'Content-Type: application/json' -H "Authorization: Bearer $HOPBOX_API_KEY" \
    "https://example.com/hopbox.v1.MachineService/$1" -d "$2"
  echo
}
```

### A private URL

The URL of a new machine is private. Open `https://box1.example.com/` in a browser:

1. Without a sign-in for the machine, the edge role sends the browser to `https://example.com/auth/machine`.
2. The browser has a console session of an account that uses the machine. So the control role sends it back to the machine host with a one-time code.
3. The edge role trades the code for the cookie `__Host-hopbox_machine`, and opens the page that the browser asked for.

Without a console session, step 2 shows `sign in to the console of this install first, then open the machine URL again`. Sign in, then open the machine URL again. The creator of a private machine opens its URL. Once the creator shares the machine, every member of its org does too. An org admin cannot open the private URL of a private machine of another member.

A sign-out ends the machine URLs of the browser too. So does an unshare, for each member that is not the creator.

### The URL port

The URL reaches port 8000 in the machine by default. The creator picks another port:

```sh
api SetMachineURLPort '{"name":"box1","port":3000}'
```

The edge role gets the new port with the next routes, within a second or two.

### A public URL

The creator makes the URL public, so it needs no sign-in:

```sh
api SetMachineURLVisibility '{"name":"box1","visibility":"URL_VISIBILITY_PUBLIC"}'
curl -s https://box1.example.com/
```

To make it private again, set `URL_VISIBILITY_PRIVATE`.

### What the code of the machine sees

The request reaches the machine over plain HTTP, with these headers:

- `Host: box1.example.com`.
- `X-Forwarded-For`: the IP address of the client.
- `X-Forwarded-Proto: https`.

The edge role takes each cookie of hopbox out of the request, so the machine never gets a credential of the install. It also drops a `Set-Cookie` of the machine for another host than its own, such as `Domain=example.com`. Such a cookie can reach the console and the other machines of the install.

### When the URL does not open

| Page | Cause |
| --- | --- |
| `404` `no machine has the name "box1"` on the domain, after the sign-in | No machine holds the name, or the account does not see the machine. |
| `403` | The account has no access to the machine, or an instance admin suspended its org. |
| `503` `The machine box1 is stopped.` | The machine does not run. Start it. |
| `503` `... has no network address on its node` | The machine has no address. |
| `502` `The machine did not answer on its URL port.` | No server listens on the URL port in the machine. Check the port, and start the server in the machine. |

[The edge role](/reference/edge/index.html.md#machine-urls) lists every answer.
