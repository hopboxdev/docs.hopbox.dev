# Get the wildcard certificate of an install

> How to get the wildcard certificate of an install, first from the test CA Pebble on a Mac, then from Let's Encrypt on a server.

This guide gets the certificate of an install for its domain and for every machine name, such as `example.com` and `*.example.com`. First it runs the whole flow on a Mac against Pebble, a test CA of Let's Encrypt. Then it gets a real certificate from Let's Encrypt on a server.

[The certificate of the install](/reference/acme/index.html.md) covers the ACME config, the job, and the backoff in full.

The release files of hopbox are not public yet. The hopbox team gives them on request.

## How the certificate gets issued

ACME is the protocol that a CA uses to issue certificates automatically. The CA issues a wildcard certificate only after a DNS-01 challenge: the CA gives a value, and the install puts it in a TXT record at `_acme-challenge.<domain>`. The DNS role of the install serves that record ([Serve the domain of an install with the DNS role](/guide/dns-role/index.html.md)).

The leader of the control role runs the ACME job. The job adds the TXT records to the zone, waits until each name server serves them, and then asks the CA to check them. It stores the certificate and its key, sealed, in the control store. After two thirds of the lifetime of the certificate, it renews it.

The job asks for no certificate until an instance admin agrees to the terms of the CA.

## What you need

- For the steps on a server: a server installed from the release bundle, as in [Install hopbox on a Debian 13 server by hand](/guide/install-debian/index.html.md) or [Install hopbox on your servers with Terraform](/guide/install-with-terraform/index.html.md).
- For the steps on a Mac: a checkout of the source of hopbox, built with `make build`. The source is not public yet, so these steps are for the people who work on hopbox.
- For the steps on a Mac: the bundled Postgres. `make pg` builds it on the first run and prints its dir.
- Go, to install Pebble, and `dig`.

## Run the flow on a Mac with Pebble

### Start Pebble

Install Pebble, and start it from its module dir, where its test config and its test certificates are. `-dnsserver` sends each DNS query of Pebble to the DNS role on port 15353. `PEBBLE_VA_NOSLEEP=1` stops the random wait of Pebble before each check:

```sh
go install github.com/letsencrypt/pebble/v2/cmd/pebble@v2.10.1
PEBBLE_DIR="$(go env GOMODCACHE)/github.com/letsencrypt/pebble/v2@v2.10.1"
cd "$PEBBLE_DIR" && PEBBLE_VA_NOSLEEP=1 "$(go env GOPATH)/bin/pebble" -config test/config/pebble-config.json -dnsserver 127.0.0.1:15353
```

Pebble serves its ACME directory at `https://localhost:14000/dir`. Leave it running, and use a second terminal for the next steps.

### Write the bootstrap config

The server runs the control role and the DNS role. The DNS role listens on a high port, so `acme.name_servers` tells the job where to check the zone. Pebble serves HTTPS with a certificate of its own test CA, so `acme.ca_file` makes the job trust that CA ([the bootstrap config of hopboxd](/reference/hopboxd-config/index.html.md#keys)):

```sh
PEBBLE_DIR="$(go env GOMODCACHE)/github.com/letsencrypt/pebble/v2@v2.10.1"
mkdir -p /tmp/hba && chmod 700 /tmp/hba
cat > /tmp/hba/hopboxd.toml <<EOF
roles = ["control", "dns"]
data_dir = "$HOME/hopbox-acme-dev"
run_dir = "/tmp/hba"

[dns]
listen = ["127.0.0.1:15353"]

[acme]
name_servers = ["127.0.0.1:15353"]
ca_file = "$PEBBLE_DIR/test/certs/pebble.minica.pem"

[postgres]
install = "$(make pg)"
EOF
```

Run the `cat` command in the checkout of hopbox, so `make pg` finds the recipe of Postgres.

### Start hopboxd

```sh
./bin/darwin_arm64/hopboxd run --config /tmp/hba/hopboxd.toml
```

The install has no domain yet, so the job waits:

```json
{"level":"INFO","msg":"the install gets no certificate yet","component":"acme","reason":"the install has no domain"}
```

Leave it running, and use a third terminal. Set two variables, so the commands stay short:

```sh
H=./bin/darwin_arm64/hopboxd
C=/tmp/hba/hopboxd.toml
```

### Enroll the DNS role

The DNS role gets its zone over the control channel, so the server enrolls with the control role of the same `hopboxd`:

```sh
$H admin --config $C kek rotate
TOKEN="$($H admin --config $C enrollment-token create --roles dns)"
HOPBOX_ENROLLMENT_TOKEN="$TOKEN" $H enroll --config $C
```

### Set the domain and the ACME config

The public address is the address of `ns1` and `ns2` in the zone. The ACME directory is the one of Pebble. `--acme-agree-terms` agrees to the terms of the CA, so the job starts:

```text
$ $H admin --config $C config set --domain example.com --public-address 127.0.0.1 \
    --acme-directory-url https://localhost:14000/dir --acme-email ops@example.org --acme-agree-terms
domain	example.com
public_addresses	127.0.0.1
acme_directory_url	https://localhost:14000/dir
acme_email	ops@example.org
acme_terms_agreed	true
mail_transport	smtp
version	2
update_time	2026-10-20T22:58:38Z
```

Within some seconds, the log of `hopboxd run` shows the attempt. The job sets the CAA record, makes the account, solves the challenge of the domain and of the wildcard, and stores the certificate:

```json
{"level":"INFO","msg":"getting the certificate of the install","component":"acme","domain":"example.com","directory_url":"https://localhost:14000/dir","reason":"the install has no certificate"}
{"level":"INFO","msg":"changed the CAA issuers of the zone","component":"service","caa_issuers":["pebble.letsencrypt.org"],"old_caa_issuers":["letsencrypt.org"],"serial":1792537119}
{"level":"INFO","msg":"stored the ACME account","component":"service","directory_url":"https://localhost:14000/dir","account_url":"https://localhost:14000/my-account/55ce84861e84b3ee"}
{"level":"INFO","msg":"trying to solve challenge","component":"acme","identifier":"example.com","challenge_type":"dns-01"}
{"level":"INFO","msg":"trying to solve challenge","component":"acme","identifier":"*.example.com","challenge_type":"dns-01"}
{"level":"INFO","msg":"got the certificate of the install","component":"acme","domain":"example.com","serial":"4553a65221ea631b","issuer":"CN=Pebble Intermediate CA 512ad8","not_after":"2027-01-18T22:58:38Z","renew_at":"2026-12-19T22:58:38.333333333Z"}
{"level":"INFO","msg":"the next attempt for the certificate is planned","component":"acme","reason":"the certificate is due for renewal","at":"2026-12-19T23:58:38.333333333+01:00"}
```

### Read the certificate

```text
$ $H admin --config $C certificate get
domain	example.com
names	example.com,*.example.com
serial	4553a65221ea631b
issuer	CN=Pebble Intermediate CA 512ad8
not_before	2026-10-20T22:58:39Z
not_after	2027-01-18T22:58:38Z
renew_time	2026-12-19T22:58:38Z
directory_url	https://localhost:14000/dir
update_time	2026-10-20T22:58:44Z
failures	0
last_attempt_time	2026-10-20T22:58:38Z
last_error	-
next_attempt_time	-
ca_wait	false
terms_url	data:text/plain,Do%20what%20thou%20wilt
renewals_stopped	-
```

`terms_url` names the terms that your agreement covers. Pebble has no real terms, so it names a short text.

The job renews the certificate at `renew_time`, after two thirds of its lifetime.

### Check the zone

The CAA record now names the CAA identity of Pebble, and the challenges left the zone:

```text
$ dig @127.0.0.1 -p 15353 example.com CAA +short
0 issue "pebble.letsencrypt.org"
$ dig @127.0.0.1 -p 15353 _acme-challenge.example.com TXT +short
```

### Stop

Press Ctrl-C in the terminal of `hopboxd`, then in the terminal of Pebble. To start from nothing, remove `~/hopbox-acme-dev` and `/tmp/hba`.

## Get the certificate from Let's Encrypt on a server

In this example, the install serves `example.com`. Before the job can get a certificate, the domain must be delegated to the DNS role of the install ([Serve the domain of an install with the DNS role](/guide/dns-role/index.html.md#delegate-the-domain-of-the-install)). Let's Encrypt asks the name servers of the delegation, and those are the servers with the DNS role.

1. Check the delegation from your laptop. The answer must come from the DNS role of the install:

   ```sh
   dig +trace example.com SOA
   ```

2. Test against the staging CA of Let's Encrypt first. Its rate limits are higher, and its certificates are not trusted, so a mistake costs nothing. Do this only while the install has no trusted certificate. A change of the directory replaces the certificate at once. After a move from the production CA to the staging CA, every browser refuses the HTTPS of the install. Set the contact email of the operator, and agree to the terms of Let's Encrypt at <https://letsencrypt.org/repository/>:

   ```sh
   sudo hopboxd admin config set --acme-directory-url https://acme-staging-v02.api.letsencrypt.org/directory \
     --acme-email ops@example.org --acme-agree-terms
   ```

3. Read the certificate. Within a minute, it names the staging CA of Let's Encrypt as the issuer. The log shows each step of the job:

   ```sh
   sudo hopboxd admin certificate get
   journalctl -u hopboxd | grep '"component":"acme"'
   ```

4. Move to the production CA. A new directory clears the agreement, so give `--acme-agree-terms` again. The job gets a new certificate at once, because the certificate is from another directory:

   ```sh
   sudo hopboxd admin config set --acme-directory-url https://acme-v02.api.letsencrypt.org/directory --acme-agree-terms
   ```

The CAA record of the zone then names `letsencrypt.org`:

```sh
dig @1.1.1.1 example.com CAA +short
```

If an attempt fails, `sudo hopboxd admin certificate get` shows the error in `last_error`, and the time of the next attempt in `next_attempt_time`. The log record `getting the certificate of the install failed. The certificate before it stays` has the same facts. The job waits 1 minute after the first failure, and doubles the wait after each failure, up to 6 hours ([the certificate of the install](/reference/acme/index.html.md#renewal-and-backoff)). A common cause is a name server that the control role cannot reach on port 53. A change of the domain, the public addresses, or an ACME setting ends the wait, unless the CA asked for it with a rate limit.

## Serve the certificate

A server with the edge role serves the certificate. The control role pushes each new certificate to it over the control channel, so a renewal needs no restart ([the control channel](/reference/control-channel/index.html.md#edge-pushes)). Once the edge role has it, check the certificate that the edge serves:

```sh
openssl s_client -connect example.com:443 -servername example.com </dev/null 2>/dev/null | openssl x509 -noout -subject -enddate
```

The metric `hopbox_edge_certificate_not_after_seconds` gives the same end ([Metrics](/reference/metrics/index.html.md#metrics-of-the-edge-role)). [Run the edge role and open machine URLs](/guide/machine-urls/index.html.md) opens a machine URL with it.
