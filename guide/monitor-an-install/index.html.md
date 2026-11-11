# Monitor an install with Prometheus and Grafana

> How to set up the monitoring bundle on the control server: Prometheus, Alertmanager, node_exporter, and Grafana from Debian packages, the alert token of the webhook to hopboxd, the outside check, a test alert, the Grafana host and its rollback, and what each alert means.

This guide sets up the monitoring bundle on the control server of an install. The monitoring bundle is the set of files that hopbox ships for standard monitoring tools. Prometheus scrapes `hopboxd` and node_exporter, and runs the alert rules. Alertmanager sends each alert to the alert webhook of `hopboxd`, which makes a notice for each instance admin and emails them through the mail of the install ([Metrics](/reference/metrics/index.html.md#alert-webhook)). Alertmanager sends the alert `Watchdog` to an outside check. Grafana shows the dashboards D1 to D8, and it is optional. An instance admin opens Grafana through an SSH tunnel, or at its own host `grafana.<domain>`.

The tools come from Debian packages. hopbox does not run them or upgrade them. Each tool listens on a loopback address only, as the metrics listener of `hopboxd` does ([Metrics](/reference/metrics/index.html.md#listener)):

| Tool | Debian 13 package | Listen address |
| --- | --- | --- |
| Prometheus | `prometheus` 2.53.3+ds1-2 | `127.0.0.1:9090` |
| Alertmanager | `prometheus-alertmanager` 0.28.1+ds-1 | `127.0.0.1:9093` |
| node_exporter | `prometheus-node-exporter` 1.9.0-1+b4 | `127.0.0.1:9100` |
| postgres_exporter, optional | `prometheus-postgres-exporter` 0.17.1-1+b4 | `127.0.0.1:9187` |
| Grafana | `grafana` 13.3.0 from the Grafana apt repository, because Debian has no Grafana package | `127.0.0.1:3000` |

The versions come from these sources on 2026-10-31:

- The Debian versions are those of the trixie Packages index at `https://deb.debian.org/debian/dists/trixie/main/binary-amd64/Packages.xz`. trixie-updates and trixie-security had no newer version. A Debian 13 container installed these versions, and ran the commands of steps 1 to 5 without systemd and without an SMTP server.
- `make check-monitoring` runs promtool and amtool from the Docker Hub images `prom/prometheus:v2.53.3` and `prom/alertmanager:v0.28.1`, the same upstream versions as the Debian packages. The `Makefile` pins each image by the digest that Docker Hub gives for its tag.
- Grafana 13.3.0 was the newest version in the Packages index of `https://apt.grafana.com`, at `dists/stable/main/binary-amd64`. The dashboards load in Grafana 13.3.0.

The host firewall of hopbox also blocks inbound traffic to these ports ([the host firewall](/reference/host-firewall/index.html.md)). Prometheus keeps the samples for 30 days, and scrapes every 15 seconds. On ks, the stack takes less than 0.5 GB of RAM and about 1.2 GB of disk for 30 days (plan section 4.7).

## What you need

- A control server of the install, with the release bundle in `/opt/hopbox/current` ([Install hopbox on a Debian 13 server by hand](/guide/install-debian/index.html.md)). Run every command of this guide as root on that server.
- The mail of the install, which sends the alert emails to the instance admins. `hopboxd admin config smtp get` prints the SMTP server of the install ([hopboxd](/reference/hopboxd/index.html.md#admin-config-smtp-get)). Each instance admin gets the email of each new alert at each email of the account.
- A check of a free outside check service, such as healthchecks.io. Set its period to 2 minutes and its grace time to 5 minutes, and copy its ping URL. This check is the dead man's switch of the install: when the pings stop, the service emails you.

The release bundle carries the files under `/opt/hopbox/current/monitoring` ([Release bundle](/reference/release-bundle/index.html.md#contents)):

| File | Copy to |
| --- | --- |
| `prometheus.yml` | `/etc/prometheus/prometheus.yml` |
| `prometheus/prometheus` | `/etc/default/prometheus` |
| `rules/hopbox-recording.yml`, `rules/hopbox-alerts.yml` | `/etc/prometheus/rules/` |
| `alertmanager.yml` | `/etc/prometheus/alertmanager.yml` |
| `alertmanager/prometheus-alertmanager` | `/etc/default/prometheus-alertmanager` |
| `node-exporter/prometheus-node-exporter` | `/etc/default/prometheus-node-exporter` |
| `postgres-exporter/prometheus-postgres-exporter` | `/etc/default/prometheus-postgres-exporter`, for [postgres_exporter](#add-postgres_exporter) |
| `scrape-configs/postgres.yml` | `/etc/prometheus/scrape-configs/postgres.yml`, for [postgres_exporter](#add-postgres_exporter) |
| `grafana/grafana-server.conf` | `/etc/systemd/system/grafana-server.service.d/hopbox.conf` |
| `grafana/grafana-host.conf` | `/etc/systemd/system/grafana-server.service.d/hopbox-host.conf`, for [the Grafana host](#8-turn-on-the-grafana-host) |
| `grafana/provisioning/datasources/hopbox.yml` | `/etc/grafana/provisioning/datasources/hopbox.yml` |
| `grafana/provisioning/dashboards/hopbox.yml` | `/etc/grafana/provisioning/dashboards/hopbox.yml` |
| `grafana/hopbox-*.json` | `/var/lib/grafana/dashboards/hopbox/` |

The files `rules/*_test.yml` are the tests of the rules. The server does not need them.

## 1. Install the packages

```sh
apt-get update
apt-get install prometheus promtool prometheus-alertmanager prometheus-node-exporter
```

The packages start each tool at once with the configuration of Debian. That configuration listens on all addresses. The host firewall blocks these ports, and step 2 moves each tool to loopback.

## 2. Copy the files of the bundle

```sh
m=/opt/hopbox/current/monitoring
install -m 0644 "$m/prometheus.yml" /etc/prometheus/prometheus.yml
install -m 0644 "$m/prometheus/prometheus" /etc/default/prometheus
install -d -m 0755 /etc/prometheus/rules
install -m 0644 "$m/rules/hopbox-recording.yml" "$m/rules/hopbox-alerts.yml" /etc/prometheus/rules/
install -m 0644 "$m/alertmanager.yml" /etc/prometheus/alertmanager.yml
install -m 0644 "$m/alertmanager/prometheus-alertmanager" /etc/default/prometheus-alertmanager
install -m 0644 "$m/node-exporter/prometheus-node-exporter" /etc/default/prometheus-node-exporter
```

## 3. Set the alert token and the outside check

Alertmanager calls the alert webhook of `hopboxd` at `http://127.0.0.1:9464/alerts`, with a token in the header `Authorization`. Make the token:

```sh
hopboxd admin config alert-token set
```

The command keeps only the SHA-256 of the token in the control store. It writes the token to `/etc/prometheus/alertmanager-hopbox-token`, which only root and the group `prometheus` can read ([hopboxd](/reference/hopboxd/index.html.md#admin-config-alert-token-set)). The receiver `hopbox` of `alertmanager.yml` reads that file. A second run makes a new token, and the old one stops working at once. After each run, reload Alertmanager, as step 4 does.

Each new alert gives each instance admin a notice on the page "Notices" of the Admin area ([Use the Admin area](/guide/admin-area/index.html.md#alerts)). Each call of Alertmanager sends one group of alerts, and gives each instance admin one email that lists the new alerts of the call. A repeat of the alert sends no email. When the alert ends, its notice closes by itself.

Alertmanager reads the ping URL of the outside check from a file of its own, outside the sealed store of hopbox. Only root and the user `prometheus` can read it. Write the ping URL, such as `https://hc-ping.com/<uuid>`:

```sh
install -m 0640 -o root -g prometheus /dev/null /etc/prometheus/alertmanager-watchdog-url
read -r url && printf '%s' "$url" > /etc/prometheus/alertmanager-watchdog-url; unset url
```

Check the two configurations:

```sh
promtool check config /etc/prometheus/prometheus.yml
amtool check-config /etc/prometheus/alertmanager.yml
```

Each tool prints `SUCCESS`. `promtool` also checks the two rule files, and prints `SUCCESS: 29 rules found` and `SUCCESS: 81 rules found`.

### Use the email of Alertmanager instead

An install can let Alertmanager send the alert email itself, through the SMTP server of the install, in place of the notices of `hopboxd`. Then the alerts do not show on the page "Notices". Alertmanager needs the SMTP password in a file of its own, outside the sealed store of hopbox. Use one path only, so that the admins do not get each alert twice.

1. In `/etc/prometheus/alertmanager.yml`, set `receiver: email` in place of `receiver: hopbox` on the two lines that name it: the top of `route`, and the route of `severity="info"`.

2. Replace each `example.org` value:

   | Key | Value |
   | --- | --- |
   | `smtp_smarthost` | The host and the port of the SMTP server, such as `smtp.example.org:587`. Port 587 uses STARTTLS, and port 465 uses TLS from the first byte. |
   | `smtp_from` | The sender of the alerts, such as `hopbox alerts <alerts@example.org>` |
   | `smtp_auth_username` | The SMTP login. For a relay on the same host without a login, remove this line and the line `smtp_auth_password_file`. |
   | `to` of the receiver `email` | The address that gets the alerts |

3. Write the SMTP password: type it at the prompt, then press Enter.

   ```sh
   install -m 0640 -o root -g prometheus /dev/null /etc/prometheus/alertmanager-smtp-password
   read -rs password && printf '%s' "$password" > /etc/prometheus/alertmanager-smtp-password; unset password
   ```

4. Remove the token of the webhook, so that nothing can make a notice with it:

   ```sh
   hopboxd admin config alert-token clear
   ```

5. Run `amtool check-config /etc/prometheus/alertmanager.yml` again.

## 4. Restart the tools and check the targets

```sh
systemctl restart prometheus-node-exporter prometheus prometheus-alertmanager
ss -ltn | grep -E ':(9090|9093|9100) '
```

`ss` prints three lines, each with a local address on `127.0.0.1`. If a line shows `*` or `0.0.0.0`, the file of step 2 in `/etc/default/` is missing. Copy it again, and restart that tool.

Wait 30 seconds for the first scrapes. Then read the targets:

```sh
promtool query instant http://127.0.0.1:9090 up
```

```text
up{instance="127.0.0.1:9090", job="prometheus"} => 1 @[1793460000]
up{instance="127.0.0.1:9093", job="alertmanager"} => 1 @[1793460000]
up{instance="127.0.0.1:9100", job="node"} => 1 @[1793460000]
up{instance="127.0.0.1:9464", job="hopboxd"} => 1 @[1793460000]
```

Each target has the value 1. A value 0 means that Prometheus cannot scrape that target. For the job `hopboxd`, the bootstrap configuration of `hopboxd` can set another `metrics.listen` ([the bootstrap config of hopboxd](/reference/hopboxd-config/index.html.md#keys)). Then change the target of the job `hopboxd` in `/etc/prometheus/prometheus.yml` to that address, and restart Prometheus.

## 5. Send a test alert

Send an alert to Alertmanager by hand:

```sh
amtool --alertmanager.url=http://127.0.0.1:9093 alert add HopboxTestAlert severity=warning \
  --annotation='summary="A test alert of the monitoring guide"'
```

Within 1 minute, the page "Notices" of the Admin area lists the alert under "Alerts", and each instance admin gets an email with the subject `Alert: HopboxTestAlert (warning) on <domain>`. The test alert ends by itself after 5 minutes. Its notice then closes, and the alert moves to "Resolved alerts" on the same page. No second email comes. If no notice shows, read the log of Alertmanager and the log of `hopboxd`:

```sh
journalctl -u prometheus-alertmanager --since '-10 min' | grep -i 'notify'
journalctl -u hopboxd --since '-10 min' | grep -i 'alert'
```

A line `refused a call of the alert webhook` in the log of `hopboxd` means that Alertmanager sends no token or an old one: run step 3 again, then `systemctl reload prometheus-alertmanager`. With the email of Alertmanager, the address of the receiver `email` gets an email whose subject starts with `[FIRING:1] HopboxTestAlert`, and a second email that says `[RESOLVED]`.

Then make sure that the outside check gets its pings. The alert `Watchdog` always fires, and Alertmanager sends it to the ping URL every 1 to 2 minutes. The page of the check on the outside service shows a ping in the last 2 minutes. Alertmanager shows the alert too:

```sh
amtool --alertmanager.url=http://127.0.0.1:9093 alert query alertname=Watchdog
```

## 6. Install Grafana

Grafana is optional. The alerts work without it, and the Prometheus web UI also draws graphs. The Debian package of Prometheus has no React web UI. To add it, run `/usr/share/prometheus/install-ui.sh`, which downloads it from GitHub.

Add the Grafana apt repository, and install the pinned version:

```sh
apt-get install gnupg
mkdir -p /etc/apt/keyrings
curl -fsSL https://apt.grafana.com/gpg.key | gpg --dearmor -o /etc/apt/keyrings/grafana.gpg
echo 'deb [signed-by=/etc/apt/keyrings/grafana.gpg] https://apt.grafana.com stable main' > /etc/apt/sources.list.d/grafana.list
apt-get update
apt-get install grafana=13.3.0
apt-mark hold grafana
```

`apt-mark hold` keeps `apt-get upgrade` from a move to a new Grafana, so a security fix of Grafana does not reach the server by itself. The owner of hopbox reads the Grafana advisories at each hopbox release, and moves the pin. The same check reads whether Safari sends `Sec-Purpose` yet, or turned on speculation rules by default ([the edge role](/reference/edge/index.html.md#requests-of-other-sites)). A new pin goes with a rerun of `make check-grafana` and `make test-grafana-browser`. A new Grafana version can change the paths that the [Grafana host](#8-turn-on-the-grafana-host) needs.

Copy the files of the bundle, and start Grafana:

```sh
m=/opt/hopbox/current/monitoring
install -D -m 0644 "$m/grafana/grafana-server.conf" /etc/systemd/system/grafana-server.service.d/hopbox.conf
install -m 0644 "$m/grafana/provisioning/datasources/hopbox.yml" /etc/grafana/provisioning/datasources/hopbox.yml
install -m 0644 "$m/grafana/provisioning/dashboards/hopbox.yml" /etc/grafana/provisioning/dashboards/hopbox.yml
install -d -m 0755 /var/lib/grafana/dashboards/hopbox
install -m 0644 "$m"/grafana/hopbox-*.json /var/lib/grafana/dashboards/hopbox/
systemctl daemon-reload
systemctl enable --now grafana-server
```

Wait until Grafana answers. Then set the password of the Grafana user `admin`: type it at the prompt, then press Enter. The prompt does not show the password.

```sh
until curl -fs http://127.0.0.1:3000/api/health >/dev/null; do sleep 1; done
read -rs password && printf '%s\n' "$password" | grafana cli --homepath /usr/share/grafana --config /etc/grafana/grafana.ini admin reset-admin-password --password-from-stdin; unset password
```

## 7. Open Grafana through an SSH tunnel

Grafana listens on loopback, so you reach it through SSH, or at [the Grafana host](#8-turn-on-the-grafana-host). When `hopboxd` is down, the SSH tunnel at `http://127.0.0.1:3000/` stays the way in, with the login form of Grafana. The tunnel depends on no part of `hopboxd`. On your computer, open a tunnel to the control server:

```sh
ssh -L 3000:127.0.0.1:3000 root@control1.example.org
```

A control server that is not an edge server takes SSH only through an edge server as a jump host ([the host firewall](/reference/host-firewall/index.html.md#the-jump-hosts)). Then add the jump host:

```sh
ssh -J root@edge1.example.org -L 3000:127.0.0.1:3000 root@control1.example.org
```

Open `http://127.0.0.1:3000` in a browser, and sign in as `admin`. The folder `hopbox` holds the nine dashboards:

| Dashboard | Content |
| --- | --- |
| hopbox D1 Overview | Targets down, firing alerts, nodes, running machines, the host firewall, the time until the certificate ends, API errors, executor failures, the memory of `hopboxd`, the units of `hopboxd run` that do not run, and the releases of `hopboxd` |
| hopbox D2 Capacity | vCPUs, RAM, and disk per node and for the install, allocated against capacity, and the CPU and RAM of each host |
| hopbox D3 Machines | The top 10 machines by CPU share, RAM, disk, egress, inbound traffic, and new outbound connections. Each line shows the name of the machine and the ID of its org. |
| hopbox D4 Accounts and abuse | New accounts, orgs, API keys, machines, and URL changes in the last hour, from the recording rules that the alerts read. Also accounts by state, rate limits, org limits, and audit actions. |
| hopbox D5 Control plane | The leader, the fence, the API calls with their error share and p99, the leader jobs with their failures and the time since their last success, the pool and the errors of the control store, the change events, the bundled Postgres, the control channels, the enrollments, the emails, and ACME. The row "Postgres" needs [postgres_exporter](#add-postgres_exporter). |
| hopbox D6 Nodes | The executor actions and boot phases, the lease losses, the guest links, the image caches, the cold moves with their pulls, and the state and restarts of `hopboxd.service` |
| hopbox D7 Edge, DNS, and SSH | The requests of the edge, the requests to machines and the upstream errors, the routes, the connections against their limit, the refusals, the certificate, the DNS queries and zone serials, and the SSH front door |
| hopbox D8 Network | The egress exits and uplinks, the traffic and dropped packets of the machines, the repairs of the node network, the mesh peers with their handshakes and traffic, and the host firewall |
| hopbox D9 Memory | The machines in standby and hibernated, the machines by state, the failed wakes and resumes, the count and the p99 of pauses, resumes, hibernates, and wakes, the p95 of the saves and loads of the VMM driver. Also the saved state of the nodes, in all and by node and kind, and the saves that the nodes refused, by reason. Also the wakes that the edges start for connections with the held connections, and the p95 of the wait of a held connection. |

The same tunnel works for Prometheus on port 9090 and Alertmanager on port 9093.

The panel "Running machines per node" of D3 leaves out the hibernated machines, whose series of `hopbox_machine_info` have the state `saved` ([Metrics](/reference/metrics/index.html.md#metrics-of-the-machines)). The panel "Machines by state" of D1 and "Machines per node" of D2 leave out the state `deleted`. "Moves by state" of D6 leaves out the state `done`. These counts stay until the sweep drops the row, so they set the scale and hide the live states.

Each stat reads one value, at the end of the time range. It reads its series through a window of 45 seconds, about 3 scrapes. So one or two missed scrapes do not change it, and a series that ended loses its value within 45 seconds. The dashboards refresh once a minute, so the change can take up to one more minute to show. After an upgrade, the release stats of D1 can show both releases for those 45 seconds.

These stats differ from that rule:

- "Targets down" of D1 changes at each missed scrape, because a failed scrape writes the value 0 to `up`. It reads that failure until the next good scrape. No other stat changes.
- "API error share" of D5 and "Upstream errors" of D7 read a rate over 5 minutes, and "Lease losses" of D6 reads an increase over 15 minutes. The counts of D4 read an increase over 1 hour. Each one keeps a value for its window after its series ended.
- After a restart of Prometheus, "Nodes", "Free vCPUs", and "Disk free" can keep a value for about 5 minutes after the stop, because their recording rules read a series that ended while Prometheus did not run. After a stop of more than 45 seconds, the other stats on a raw series have no value until the first scrape, up to about 20 seconds after the start. The counts of D4, "Nodes", "Free vCPUs", and "Disk free" read a recording rule through the same window, so they also have no value until the first run of their rules, within about the same time. "API error share", "Upstream errors", and "Lease losses" keep their value, because they read a range over the stop.
- Just after a start of Prometheus, "Targets down" and "Firing alerts" read 0 with nothing behind them. "Targets down" knows nothing until the first scrape of each target. "Firing alerts" knows nothing until the rules run on new samples, and an alert with a `for` can wait again for that time. So a 0 on these two stats in the first minutes after a start does not say that all is well.

A stat with no value shows a word in a neutral color. The word says only that no scraped target serves the metric. It does not say that all is well:

| Dashboard | Stat | Word | Normal case | Fault case |
| --- | --- | --- | --- | --- |
| D1 | Units down | no value | None. Each `hopboxd` serves this metric. | The scraped `hopboxd` is of an older release. |
| D1 | Certificate ends in | no value | The install has no edge. | The edge runs on a server that Prometheus does not scrape. An edge before its first certificate reads `0 s` in red. |
| D2 | Disk free | no value | The install has no node, or it is in maintenance mode. | The install has no leader, or the leader fails to read its stats. |
| D5 | Bundled Postgres | no value | The install uses an external Postgres, or the scraped server has no control role. | The start of a unit before `postgres` hangs, which "Units down" of D1 shows. A bundled Postgres whose start hangs reads `0` in red. |
| D6 | Stranded machines | no value | The install is in maintenance mode. | The install has no leader, or the leader fails to read its stats. |
| D6 | hopboxd not active | no value | The install has no node_exporter. | |
| D8 | Egress exits | no value | The scraped server has the edge role, or no node role. | A node without the edge role runs on a server that Prometheus does not scrape. On a scraped node, the value `0` stays red. |
| D8 | Uplinks down | no value | The scraped server has no edge role. | The edge runs on a server that Prometheus does not scrape. |
| D8 | Missing mesh peers | no value | The scraped server is outside the mesh, or the control role sent no mesh yet. | A mesh device that is gone reads the count of expected peers in orange. |
| D8 | Oldest mesh handshake | no value | The scraped server is outside the mesh, such as on an install with one server. | The mesh runs on a server that Prometheus does not scrape. A peer without a handshake reads `never` in orange. |

Each of these stats keeps its value through one or two missed scrapes, and has no value after a scrape keeps failing for about 45 seconds. The stat "Targets down" of D1 shows a failed scrape from the first failure. It does not show a server that is no target of Prometheus, and the bundle has one target for `hopboxd`, the control server.

Each server serves `hopbox_edge_certificate_not_after_seconds` and `hopbox_egress_exits` itself. So on a fleet, "Certificate ends in", "Egress exits", HopboxCertificateExpiry, and HopboxEgressFailClosed do not see an edge or a node on another server. The leader serves the disk metrics for each node. So "Disk free", HopboxDiskFull, and HopboxDiskOvercommit see each node, while the scraped control process leads. "Oldest mesh handshake" and HopboxMeshPeerStale read the handshakes of the scraped server with its peers, so they do not see a link between two other servers.

The counts of D4 read 0 before the first event, while a control process answers. If its scrape keeps failing for about 45 seconds, or a recording rule fails for that time, they read "No data". So "No data" on D4 means that the count is unknown, and 0 means that no event happened. "Nodes" of D1 and "Free vCPUs" of D2 read 0 on an install without a node, and "No data" while the leader serves no stats, as in maintenance mode.

The pages "Overview", "Capacity", "Machines", and "Accounts" of the Admin area can link to these dashboards. For the links, set the URL of Grafana in the install config on a control server ([Use the Admin area](/guide/admin-area/index.html.md)). For the tunnel above, the URL is `http://127.0.0.1:3000`. Step 8 of [the Grafana host](#8-turn-on-the-grafana-host) sets the URL of the host instead.

```sh
sudo hopboxd admin config set --grafana-url http://127.0.0.1:3000
```

## 8. Turn on the Grafana host

The Grafana host `grafana.<domain>` opens Grafana in the browser of an instance admin, after a sign-in on the console. It needs no tunnel and no second password ([the edge role](/reference/edge/index.html.md#the-grafana-host)). It is optional. Grafana stays on loopback. The wildcard certificate and the wildcard DNS record of the install cover the host, so no certificate, DNS record, or port changes. `hopboxd` needs no restart. Until this server has the key, the host answers `404` to an instance admin.

Grafana works on one control server only (Q163 of the abuse plan). The edge servers send the requests in turn to each control server. So a control server without the key answers `404` on its share.

Pick your case:

- A new install: run the steps below after section 7. A new machine cannot take the name `grafana`, so step 1 finds no such machine.
- An existing install: first upgrade `hopboxd`, and follow [After an upgrade of hopboxd](#after-an-upgrade-of-hopboxd). A machine can hold the name from an earlier release. That machine keeps its URL, and the host stays off on this install until the machine is renamed or deleted. Until then, `https://example.com/auth/grafana` answers `409 Conflict`, and the host serves that machine.

Run these steps in this order:

1. Make sure that no machine holds the name. The second column of the output must not show `grafana`. If it does, the owner of that machine renames it or deletes it first.

   ```sh
   hopboxd admin machine list
   ```

2. Make sure that DNS and the certificate serve the host. `dig` prints the public addresses of the install. `curl` prints `303`.

   ```sh
   dig +short grafana.example.com
   curl -s -o /dev/null -w '%{http_code}\n' https://grafana.example.com/
   ```

3. Make the key. `ls` must show the public key before the next step, because Grafana does not start without that file while the drop-in names it.

   ```sh
   hopboxd admin grafana-key set
   ls -l /etc/grafana/hopbox-identity.pub
   ```

4. Copy the drop-in, and restart Grafana.

   ```sh
   install -m 0644 /opt/hopbox/current/monitoring/grafana/grafana-host.conf /etc/systemd/system/grafana-server.service.d/hopbox-host.conf
   systemctl daemon-reload
   systemctl restart grafana-server
   ```

5. Make sure that Grafana stays on loopback and refuses a forged header. `ss` must show `127.0.0.1`, and the last `curl` must print `401`.

   ```sh
   until curl -fs http://127.0.0.1:3000/api/health >/dev/null; do sleep 1; done
   ss -ltn | grep ':3000 '
   curl -s -o /dev/null -w '%{http_code}\n' -H 'X-Hopbox-Identity: admin' http://127.0.0.1:3000/api/user
   ```

6. Make sure that the tunnel still works. Open it as in [section 7](#7-open-grafana-through-an-ssh-tunnel), go to `http://127.0.0.1:3000/`, and sign in as `admin` with the password of section 6.

7. Sign in on the console, open `https://grafana.example.com/`, click the link on the page that opens, and open the dashboards D1 to D9.

8. Set the links of the console. They then open `https://grafana.example.com/d/hopbox-overview` and the other dashboards. A change of the domain needs a new URL here.

   ```sh
   hopboxd admin config set --grafana-url https://grafana.example.com
   ```

A link in the Admin area opens Grafana with one click. A typed address or a bookmark takes one more click when the cookie of the host is gone: a small page holds one link, and the click on it opens the path that you asked for. A browser that sends no `Sec-Fetch-Site`, such as Safari before 16.4, cannot open the host. Use the SSH tunnel there.

A tab of Grafana works for up to 1 hour after your sign-in. Each request reads the session, so a revoke or a sign-out acts at the next request. After the hour, a reload asks you to sign in again. The sign-in ends in the Admin area, so open the link again. Prometheus and Alertmanager stay behind the tunnel.

### Roll back the Grafana host

Run these commands in this exact order.

```sh
hopboxd admin config set --grafana-url http://127.0.0.1:3000
rm /etc/systemd/system/grafana-server.service.d/hopbox-host.conf
systemctl daemon-reload
systemctl restart grafana-server
hopboxd admin grafana-key clear
```

Warning: a clear of the public key while the drop-in exists stops Grafana, and you lose the tunnel too. `hopboxd admin grafana-key clear` removes only the private key, and leaves the file `/etc/grafana/hopbox-identity.pub`. Do not remove that file before the drop-in.

Grafana then runs as it did before step 4. The host answers `404` to an instance admin. The table `grafana_sessions` and the user `hopbox` in the database of Grafana stay, and they do nothing.

## Add postgres_exporter

postgres_exporter serves the statistics of the bundled Postgres: connections, deadlocks, cache hits, WAL, and long transactions. It is optional. The row "Postgres" of D5 reads it, and no alert does. Run these steps as root on the control server, after steps 1 to 4 of this guide.

The Debian package runs postgres_exporter as the OS user `prometheus`. That user connects to the database `postgres` with the role `hopbox_monitor`, and reads the statistics of the whole cluster ([the bundled Postgres](/reference/postgres/index.html.md#what-the-role-hopbox_monitor-can-do)).

Once the OS group `prometheus` exists, each start of `hopboxd` adds the bundled Postgres to that group, also without postgres_exporter. Postgres can then read the files of that group, such as the token of the alert webhook ([the bundled Postgres](/reference/postgres/index.html.md#who-can-reach-the-socket) lists them).

1. Install the package. It starts postgres_exporter at once on all addresses, without a database. The host firewall blocks port 9187, and step 2 moves postgres_exporter to loopback.

   ```sh
   apt-get install prometheus-postgres-exporter
   ```

2. Copy the arguments and the scrape job of the bundle.

   ```sh
   m=/opt/hopbox/current/monitoring
   install -m 0640 -o root -g prometheus "$m/postgres-exporter/prometheus-postgres-exporter" /etc/default/prometheus-postgres-exporter
   install -d -m 0755 /etc/prometheus/scrape-configs
   install -m 0644 "$m/scrape-configs/postgres.yml" /etc/prometheus/scrape-configs/postgres.yml
   ```

3. Read the socket dir of Postgres.

   ```sh
   stat -c '%A %G' /run/hopbox/pg
   ```

4. If step 3 prints `drwx------`, restart `hopboxd`. It opens the socket to the group `prometheus` only at its start. A restart stops the API and the leader jobs of this server for some seconds, and the machines keep running. Step 3 then prints `drwxr-x--- prometheus`.

   ```sh
   systemctl restart hopboxd
   ```

5. Restart postgres_exporter, and reload Prometheus.

   ```sh
   systemctl restart prometheus-postgres-exporter
   systemctl reload prometheus
   ```

6. Make sure that postgres_exporter reaches Postgres. The command prints `pg_up 1`.

   ```sh
   curl -s http://127.0.0.1:9187/metrics | grep '^pg_up'
   ```

If the command prints `pg_up 0`, read the log of postgres_exporter with `journalctl -u prometheus-postgres-exporter`. `Permission denied` means that the socket dir stays closed: go to step 3.

An install with an external Postgres has no socket and no role `hopbox_monitor`. Make a role of your own on that Postgres, as a member of `pg_monitor`, and name that server and role in `DATA_SOURCE_NAME` of `/etc/default/prometheus-postgres-exporter`. Only root and the group `prometheus` can read that file, as with the file of the Debian package, because the value can hold a password.

## After an upgrade of hopboxd

An upgrade of `hopboxd` does not copy the files of the bundle. The rules of a release read the metrics of that release, so copy the new files after each upgrade. Do not copy `alertmanager.yml` again: the copy has no values of your changes, such as the email of Alertmanager, so the alerts can stop and `Watchdog` still pings.

1. Compare the files of the release with the files of the server. `diff` prints nothing for a file that did not change.

   ```sh
   m=/opt/hopbox/current/monitoring
   diff /etc/default/prometheus "$m/prometheus/prometheus"
   diff /etc/default/prometheus-alertmanager "$m/alertmanager/prometheus-alertmanager"
   diff /etc/default/prometheus-node-exporter "$m/node-exporter/prometheus-node-exporter"
   diff /etc/prometheus/alertmanager.yml "$m/alertmanager.yml"
   ```

2. If the last `diff` shows a change apart from your own values, add that change to `/etc/prometheus/alertmanager.yml` by hand. An `alertmanager.yml` from before the alert webhook sends each alert by email. To move it to the webhook, add the receiver `hopbox` and the rules of `inhibit_rules` with `HopboxAlertWebhookFailing` and `HopboxDown`, and set `receiver: hopbox` on the two lines that name `email`. Then run step 3 for the token. Keep `email` on these lines only if you prefer the email of Alertmanager, because the admins otherwise get each alert twice.

3. Copy the other files of step 2.

   ```sh
   install -m 0644 "$m/prometheus.yml" /etc/prometheus/prometheus.yml
   install -m 0644 "$m/prometheus/prometheus" /etc/default/prometheus
   install -m 0644 "$m/rules/hopbox-recording.yml" "$m/rules/hopbox-alerts.yml" /etc/prometheus/rules/
   install -m 0644 "$m/alertmanager/prometheus-alertmanager" /etc/default/prometheus-alertmanager
   install -m 0644 "$m/node-exporter/prometheus-node-exporter" /etc/default/prometheus-node-exporter
   ```

   If you run postgres_exporter, also copy its two files of [Add postgres_exporter](#add-postgres_exporter), then run `systemctl restart prometheus-postgres-exporter`.

4. Make sure that the configurations are valid. Each tool prints `SUCCESS`.

   ```sh
   promtool check config /etc/prometheus/prometheus.yml
   amtool check-config /etc/prometheus/alertmanager.yml
   ```

5. Reload Prometheus and Alertmanager, which then read their new configurations.

   ```sh
   systemctl reload prometheus prometheus-alertmanager
   ```

6. If step 1 showed a change in a file of `/etc/default/`, restart that tool. A reload does not read the arguments. For example, run `systemctl restart prometheus` after a change of `/etc/default/prometheus`.

7. If you run Grafana, copy the Grafana files of step 6 again. If this server has the Grafana host on, also copy `grafana-host.conf` as in step 4 of [section 8](#8-turn-on-the-grafana-host). Do not copy it to a server without the public key, because Grafana then does not start. Then run `systemctl daemon-reload` and `systemctl restart grafana-server`.

8. An upgrade does not turn on the Grafana host. To turn it on, follow [section 8](#8-turn-on-the-grafana-host).

## Change a rule

The thresholds of the alerts are defaults that you can change. To change one on an install, edit `/etc/prometheus/rules/hopbox-alerts.yml`, check it, and reload Prometheus:

```sh
promtool check rules /etc/prometheus/rules/hopbox-alerts.yml
systemctl reload prometheus
```

The copy of the next release replaces the file, so make the change again after each copy.

To change a rule of hopbox itself, edit `monitoring/rules/hopbox-alerts.yml` in the repository, and its test in `monitoring/rules/*_test.yml`. Each alert has a case that fires and a case that does not. Then run `make check-monitoring`. It runs `promtool test rules` and `amtool` in Docker, at the versions of Debian 13, and CI runs it too. A new alert also needs a case in a rule test and an entry in the next section, or the docs check fails.

Alertmanager can also open one GitHub issue for each alert group. For that path, set up a relay on the control server, which turns the body of Alertmanager into a GitHub issue. An alert group is a set of alerts that Alertmanager sends together, here all the alerts of one name. The receiver `github` of `alertmanager.yml` sends to `http://127.0.0.1:9095/alerts`. Remove the `#` of the route `github` in `alertmanager.yml`, and keep the token of the relay in a file of its own.

## What each alert means

Each alert has the label `severity`: `critical`, `warning`, `info`, or `none`. A critical alert needs action now. A warning needs action within a day. An info alert is a fact to look at. While a critical alert fires, Alertmanager holds back the warning of the same name and server. Each email links to the entry of its alert below. The alerts on the accounts count over the whole install. To find the account, read the audit log on the page "Audit log" of the Admin area ([Use the Admin area](/guide/admin-area/index.html.md#handle-an-abuse-report)). No alert acts by itself. After you stop a machine, its alerts HopboxMachineCPUSustained, HopboxEgressHigh, and HopboxEgressScan keep firing for about 10 minutes. These alerts read the last 10 minutes of `hopbox_machine_info`, so a restart of `hopboxd` does not end them.

### HopboxMachineCPUSustained

Warning. A machine used 90% or more of its vCPUs, averaged over 30 minutes, for 2 hours. The alert names the machine, its name, and the ID of its org. A build or an AI agent also uses CPU for hours, so this alone is no proof of abuse. Look at the egress of the machine on D3 and at its org. A crypto miner uses all vCPUs for days, with little traffic to a mining pool. To stop the machine, run `hopboxd admin machine stop --id=ID`, and handle it as an abuse report ([Use the Admin area](/guide/admin-area/index.html.md#handle-an-abuse-report)).

### HopboxNodeCPUBusy

Warning. The CPU of a host was busy 90% or more of the time for 30 minutes, from node_exporter. The machines of the node compete for the CPU. Look at the top machines on D3, and at the CPU of the machines against the host on D2.

### HopboxNodeVCPUsFull

Warning. The machines of a node hold 90% or more of its vCPU capacity, with the overcommit, for 10 minutes. Placement soon puts no new machine on the node. Delete or stop machines that nobody uses, or add a node ([Enroll the node role on one server](/guide/enroll-node/index.html.md)).

### HopboxNodeRAMFull

Warning. The label `cause` says which check fired. `allocated`: the machines of a node hold 90% or more of its RAM for 10 minutes, and placement soon puts no new machine on it. `host`: the available RAM of the host stayed below 10% for 10 minutes, so the kernel can kill a process. Look at the top machines by RAM on D3, and at the RAM of the host on D2.

### HopboxDiskFull

The file system of the data dir of a node has little space left. The label `cause` says which check fired. `free` with the severity warning: less than 15% is free. `free` with the severity critical: less than 5% is free. `trend`, critical: at the rate of the last 6 hours, the file system is full within 24 hours. A full data dir stops the writes of the machine disks. Look at the top disks on D3, and delete the machines that nobody uses.

### HopboxDiskOvercommit

Info. The disk sizes of the machines on a node are more than 3 times the size of its data dir. The disks are sparse, so this is no fault yet. A machine that writes its whole disk can fill the data dir. Watch HopboxDiskFull.

### HopboxNoCapacity

Warning. A create of a machine found no node with room in the last 15 minutes. `hopbox_placements_total` gives the reason, for example `ram` ([Metrics](/reference/metrics/index.html.md#metrics-of-placement)). Add a node, or free room on a node.

### HopboxDown

Critical. The label `cause` says which check fired. `scrape`: Prometheus got no answer from `hopboxd` for 5 minutes. On the server of the alert, run `systemctl status hopboxd`, and read the log of `hopboxd` ([hopboxd](/reference/hopboxd/index.html.md#logs)). `no_job`: `prometheus.yml` has no job `hopboxd`, so no other alert of hopbox can fire. Copy the file of step 2 again.

### HopboxUnitDown

Critical. A unit of `hopboxd run` had no value 1 at any scrape of the last 5 minutes, and that stayed so for 5 more minutes. So a unit that ran fires after about 10 minutes. A unit with a new series has no earlier value 1, so it fires after 5 minutes. The label `unit` names the unit ([Metrics](/reference/metrics/index.html.md#metrics-of-the-units)).

The start of that unit hangs, and `hopboxd` starts no unit after it. The metrics listener still answers, so HopboxDown does not fire. An example is the unit `store` with a Postgres that accepts the connection and does not answer. A start that hangs ends at one of these limits:

- The bundled Postgres must be ready 60 seconds after `hopboxd` runs it. Otherwise the start of the unit `postgres` fails.
- The Postgres pool gives a new connection 120 seconds, unless `postgres.url` sets `connect_timeout`. Then the start of the unit `store` fails.
- systemd ends the whole start of `hopboxd` after 180 seconds, also for a unit without a limit of its own.

In each case, `hopboxd` exits, systemd starts it again after 5 seconds, and the alert stays on through these restarts.

A start that fails at once does not fire this alert. An example is a Postgres that refuses connections. `hopboxd` exits at each start, so the scrapes rarely reach it, and the unit gets no sample. HopboxDown fires instead. A scrape that reaches a short run starts the 5 minutes of HopboxDown again, so it can come after more than 5 minutes. HopboxdRestarting also reports this case. It also reports a unit that starts well and fails later, which makes `hopboxd` exit.

Read the log of `hopboxd` on the server of the alert:

```sh
journalctl -u hopboxd --since '-15 min' | grep -E 'starting|started|start failed'
```

The last line `starting` without a line `started` names the unit that hangs. A line `start failed` gives the error. Each unit has its row in [hopboxd](/reference/hopboxd/index.html.md#run).

While the unit `postgres` or `store` does not run on the control server, `hopboxd` answers the alert webhook with 503. So this alert reaches you as HopboxAlertWebhookFailing, which holds back `Watchdog`, so the outside check emails you. It does not reach you as a notice.

### HopboxExecutorFailures

Warning. Executor actions on a node failed for 15 minutes, such as a start or a stop of a machine ([the node executor](/reference/executor/index.html.md#metrics)). D1 shows the action. Read the log of `hopboxd` on that node.

### HopboxCertificateExpiry

The certificate of an edge ends soon. The label `cause` says which check fired. `expiry` with the severity warning: the certificate ends within 14 days. `expiry` with the severity critical: it ends within 3 days. The leader renews it after two thirds of its lifetime, so the renewals failed for days. `missing`, critical: the edge served no certificate for 30 minutes, so HTTPS to the install fails. Run `hopboxd admin certificate get` to see the state of the attempts ([Get the wildcard certificate of an install](/guide/acme-certificate/index.html.md)).

### HopboxHostFirewallDown

Critical. The install of the table `inet hopbox_host` failed on a server, so the ports of that server can be open. Read the log of `hopboxd`, and check the table ([the host firewall](/reference/host-firewall/index.html.md#check-the-firewall)).

### HopboxSignUpBurst

Warning. 5 or more people signed up in the last hour, without the refused sign-ins. A bot can make accounts in a burst. In the sign-up mode `approval`, each new account waits for an instance admin. Look at the pending accounts on the page "Accounts" of the Admin area, and reject the bots ([Use the Admin area](/guide/admin-area/index.html.md)).

### HopboxPendingWaiting

Info. A pending account waited for more than 1 day. Approve or reject it on the page "Accounts" of the Admin area. A pending account expires after 30 days.

### HopboxOrgOrKeyBurst

Warning. The label `cause` says which check fired. `orgs`: 5 or more orgs were made in the last hour. `api_keys`: 5 or more API keys were made in the last hour. A bot can make orgs to get more quota, and keys to run from scripts. Find the actor of the entries `org.create` or `api_key.create` in the audit log.

### HopboxLongKeyBurst

Warning. 2 or more API keys with a lifetime of 365 days were made in the last hour. A long key outlives a suspend of its session. Find the actor of the entries `api_key.create` in the audit log.

### HopboxMachineBurst

Warning. 10 or more machines were made in the last hour. Find the actor of the entries `machine.create` in the audit log, and look at the new machines on D3.

### HopboxURLChurn

Info. 20 or more changes of the port or the visibility of machine URLs in the last hour. A phishing page can move from machine to machine. Find the actor of the entries `machine.set_url_port` and `machine.set_url_visibility` in the audit log.

### HopboxRateLimitBurst

Info. The rate limits of hopbox refused 100 or more requests in the last hour. The panel "Rate-limit refusals" of D4 shows the refusals of each limit, such as `sign_in_links` or `account_creations`. A burst of one limit is often a bot. The limits hold it back, so this alert needs no action. Look for the same client in the log of `hopboxd`.

### HopboxLimitPressure

Info. The usage limits of the orgs refused 20 or more calls in the last hour. The panel "Org limit refusals" of D4 shows the refusals of each limit, such as `machines` or `vcpus`. A team that grows can hit its limits, and a bot can too. If an org needs more, raise its limits with `hopboxd admin org set-limits` ([hopboxd](/reference/hopboxd/index.html.md#admin-org-set-limits)).

### HopboxEgressHigh

Warning. A machine sent 50 MB/s or more for 30 minutes. The alert names the machine and its org. A large upload is fine, but a flood of traffic to one address is a sign of an attack. Look at the egress of the machine on D3. If it is an attack, handle it as an abuse report.

### HopboxEgressScan

Warning. A machine opened 100 or more new outbound connections per second for 10 minutes. That is a sign of a port scan or of spam. Stop the machine with `hopboxd admin machine stop --id=ID`, and handle it as an abuse report ([Use the Admin area](/guide/admin-area/index.html.md#handle-an-abuse-report)).

### HopboxPublicTrafficJump

Info. In the last hour, public machine URLs got more than 10 times the requests of the 7-day level. The alert also needs more than 0.1 requests per second. A machine can serve a phishing page or a file that spreads. Look at the public URLs on D4, and at the inbound traffic of each machine on D3.

### Watchdog

None. This alert always fires, and Alertmanager sends it to the outside check every 1 to 2 minutes. The pings stop for one of two causes, and then the outside service emails you:

- The server, Prometheus, or Alertmanager stopped. Then no alert of the install can reach you.
- The path of the alerts fails. Alertmanager holds back `Watchdog` while HopboxAlertWebhookFailing, HopboxDown, or HopboxAlertEmailFailing fires: `hopboxd` refuses the calls of its webhook, `hopboxd` is down, or the email of Alertmanager fails.

Read the log of Alertmanager first. If the command shows `Notify attempt failed` for the receiver `hopbox`, the alert webhook fails: go to HopboxAlertWebhookFailing and HopboxAlertmanagerWebhookFailing. For the receiver `email`, the email path fails: go to HopboxAlertEmailFailing. If the command fails or shows nothing, check that the control server, Prometheus, and Alertmanager run.

```sh
journalctl -u prometheus-alertmanager --since '-30 min' | grep -i 'notify'
```

### HopboxAlertEmailFailing

Warning. Alertmanager failed to send alert email in the last 15 minutes. Only the email of Alertmanager uses this path, not the emails of `hopboxd` ([Use the email of Alertmanager instead](#use-the-email-of-alertmanager-instead)). The email of this alert also fails, so Alertmanager then holds back `Watchdog`. The outside check gets no more pings, and emails you by its own path. Read the log of Alertmanager, and check the values and the password file of the email path:

```sh
journalctl -u prometheus-alertmanager --since '-30 min' | grep -i 'notify'
```

### HopboxAlertWebhookFailing

Warning. `hopboxd` answered calls of its alert webhook with an error, and no call with success, in the last 15 minutes, for 5 minutes, from `hopbox_alert_webhook_requests_total` ([Metrics](/reference/metrics/index.html.md#alert-webhook)). A single error, such as a 503 while `hopboxd` restarts, does not fire it, because the next try of Alertmanager works. A wrong token fails each call, and fires it. The alerts of those calls did not reach the instance admins. While this alert fires, Alertmanager holds back `Watchdog`, so the outside check gets no more pings and emails you by its own path. HopboxDown does the same, because the webhook stops with `hopboxd`. Read the log of `hopboxd`:

```sh
journalctl -u hopboxd --since '-30 min' | grep -i 'alert webhook'
```

The label `code` of the metric names the answer. A 401 means a token that `hopboxd` does not know: run `hopboxd admin config alert-token set`, then `systemctl reload prometheus-alertmanager`. A 503 means that the control store of `hopboxd` was not open, or that the install was in maintenance mode. A 500 means that a write to the control store failed. A 400 or a 413 means a body that `hopboxd` refuses: compare the version of Alertmanager with the guide.

### HopboxAlertmanagerWebhookFailing

Warning. Alertmanager failed to call a webhook for 15 minutes. Alertmanager counts its webhooks together: the alert webhook of `hopboxd`, the outside check, and the GitHub relay. This alert holds back nothing, because a failed outside check is no failure of the path of the alerts. When `hopboxd` gets no call at all, for example with a wrong URL or port in the receiver `hopbox`, only this alert fires. Its notice then does not reach `hopboxd` either. Read the log of Alertmanager:

```sh
journalctl -u prometheus-alertmanager --since '-30 min' | grep -i 'notify'
```

The line names the receiver. For `hopbox`, check that its URL is the address of the metrics listener, `http://127.0.0.1:9464/alerts` by default, and that `hopboxd` runs. For `watchdog`, the outside check fails, and its own service tells you. For `github`, check the relay.

### HopboxRuleFailures

Warning. Prometheus failed to evaluate the rules of a group in the last 15 minutes, so the alerts of that group cannot fire. The label `rule_group` names the file and the group. Read the log of Prometheus. A common cause is a rule that you changed by hand ([Change a rule](#change-a-rule)).

```sh
journalctl -u prometheus --since '-30 min' | grep -i 'rule'
```

### HopboxNoLeader

Critical. No control process held the leader lease for 1 minute. Without a leader, no leader job runs: no sweep, no check of silent servers, no cold move, and no certificate renewal. Run `hopboxd admin control list` on a control server to see the control processes, and read the log of `hopboxd` for the lines with `component=leader` ([hopboxd](/reference/hopboxd/index.html.md#leader-jobs)). A fenced process does not try for the lease, so look at HopboxControlFenced too.

### HopboxLeaderFlapping

Warning. A control process took the leader lease 4 or more times in the last hour. Each new term starts the jobs again, with the sweep first. A restart of `hopboxd` also starts a new term, so a loop of restarts fires this alert. Look at HopboxdRestarting and HopboxControlFenced, and read the log of `hopboxd` for the lines with `component=leader`.

### HopboxLeaderJobFailing

Warning. A leader job failed 3 or more times in a row, and no run of it worked. The label `exported_job` names the job, such as `sweep` or `moves`. Prometheus names the label `exported_job`, because the label `job` of the scrape wins. The window is 5 minutes for the jobs that run every minute or more often, and 35 minutes for `sweep` and `pending_expiry`, which run every 10 minutes ([Metrics](/reference/metrics/index.html.md#metrics-of-the-leader-jobs)). Read the log of `hopboxd` on the leader for the errors of that job.

### HopboxLeaderJobStale

Warning. A leader job had no run that worked for 3 of its periods. The label `exported_job` names the job. The jobs of the sweep get 30 minutes and `acme` gets 3 hours. The other jobs get 3 minutes, also the jobs that run every few seconds, because Prometheus reads the time of the last success only every 15 seconds. Look at HopboxLeaderJobFailing for the same job, and read the log of `hopboxd` on the leader.

### HopboxACMEFailing

Warning. The last 3 or more attempts for the certificate of the install failed. The edge keeps the certificate that it has until that certificate ends. Run `hopboxd admin certificate get` to see the last error and the next attempt ([Get the wildcard certificate of an install](/guide/acme-certificate/index.html.md)).

### HopboxMovesFailing

Warning. 2 or more cold moves failed or gave their machine back in the last hour. A move that failed leaves the machine on its source node, so a drain or the removal of a node role waits. Run `hopboxd admin machine moves` to see the moves and their errors ([Cold move](/reference/cold-move/index.html.md#end-of-a-move)).

### HopboxDrainStuck

Warning. The drain of a server, or the removal of its node role, did not move some machines for 30 minutes. The label `server` names the server. Run `hopboxd admin server get` with that ID, and see [Drain a node](/guide/drain-a-node/index.html.md) for the causes, such as no other node with room.

### HopboxMailFailing

Warning. `hopboxd` failed to send emails in the last 30 minutes, and sent none. Sign-in links, invitations, and the emails of the notices then do not arrive. The email of this alert also uses the mail of the install, so read its notice on the page "Notices". Run `hopboxd admin config smtp get` to see the SMTP server, and read the log of `hopboxd` for the error of the SMTP server ([SMTP config](/reference/smtp/index.html.md#errors)).

### HopboxAPIErrors

Warning. 5 or more API calls of the last 10 minutes, and more than 5% of them, ended with the code `internal`, `unavailable`, `unknown`, or `data_loss`. The minimum keeps 1 failed call of 3 on a quiet install from an alert. The first call of a procedure and a code counts too. The alert waits 1 minute before it fires, so it comes up to 2 minutes after the health line. The health line of the control plane on the page "Overview" counts in the same way. Maintenance mode refuses calls with `unavailable` by design, so this alert does not fire in maintenance mode, or in the 10 minutes after it, while those calls stay in the window. A maintenance shorter than one scrape gives Prometheus no flag to read, so this alert can fire after it while the health line stays quiet. A fence of the control process also refuses calls with `unavailable`, so look at HopboxControlFenced first. D1 shows the failed calls by code, and D5 shows the share of failed calls. Read the log of `hopboxd`.

### HopboxAPISlow

Warning. The 99th percentile of the duration of the unary API calls stayed above 2 seconds for 15 minutes. The streams, such as `Exec` and the control channels, do not count. A full pool of the control store or a slow disk often causes it. Look at HopboxStorePoolFull and at the store panels of D5.

### HopboxControlFenced

Critical. A control process did not reach the control store for 1 minute, so it refuses every call and runs no leader job ([hopboxd](/reference/hopboxd/index.html.md#run)). The fence lifts at the next renewal that works. Check that Postgres runs, and read the log of `hopboxd` for the errors of the store.

### HopboxMaintenanceLong

Info. Maintenance mode stayed on for 2 hours. The API refuses every call that writes, and the leader jobs pause. If the maintenance is over, run `hopboxd admin maintenance off`.

### HopboxMachinesStuck

Warning. Machines stayed in a state other than their target state for 15 minutes or more, such as a machine that stays `failed` with the target `running`. A machine with an open cold move does not count. Run `hopboxd admin machine list` to find the machines, and read the log of `hopboxd` on their node.

### HopboxErrorLogs

Warning. `hopboxd` wrote error records of one component for 15 minutes. The labels `instance` and `component` name the server and the part of `hopboxd`. Read the log of `hopboxd` on that server for that component:

```sh
journalctl -u hopboxd --since '-30 min' | grep '"component":"executor"' | grep '"level":"ERROR"'
```

Replace `executor` with the value of the label.

### HopboxStorePoolFull

Warning. Every connection of the pool of the control store was in use for 5 minutes, so calls wait for a connection ([Metrics](/reference/metrics/index.html.md#metrics-of-the-control-store)). A slow query or a lock holds the connections. Look at HopboxAPISlow and at the store panels of D5.

### HopboxStoreErrors

Warning. Queries to the control store failed for 10 minutes. The label `kind` of `hopbox_store_errors_total` says why: `connection`, `timeout`, `serialization`, or `other`. D5 shows each kind. Check that Postgres runs, and read the log of `hopboxd`.

### HopboxEventsLag

Warning. The last change event that a control process read stayed older than 30 seconds for 5 minutes. The process then serves old state, such as old routes to the edge servers. A lost connection to the store often causes it. Read the log of `hopboxd` on that server for the errors of the change events and of the store ([Control store tables](/reference/control-store/index.html.md#events)).

### HopboxPostgresDown

Critical. The bundled Postgres took no connection for 1 minute. `hopboxd` restarts it by itself ([the bundled Postgres](/reference/postgres/index.html.md#start-restart-and-stop)). Read the log of `hopboxd` for the lines of Postgres, and check the free space of `/var/lib/hopbox`.

### HopboxStoreGrowth

Warning. At the rate of the last day, the control store outgrows the free space of its file system within 7 days. A full file system stops Postgres, and with it the control plane. `hopbox_postgres_disk_free_bytes` gives the free space of the file system that holds `/var/lib/hopbox/pg/data` ([Metrics](/reference/metrics/index.html.md#metrics-of-the-control-store)). D5 shows the size of the database and the free space. Add disk space, or look at the audit entries and the other tables that grow. An external Postgres has no such value, so this alert does not fire for it.

### HopboxServerCertExpiry

Warning. The certificate of a server ends within 7 days. The server renews it over its control channel, so the renewals failed. Check that the server holds its control channel, and read its log for the renewals ([the control channel](/reference/control-channel/index.html.md#certificate-renewal)).

### HopboxInstallCAExpiry

Warning. The install CA ends within 90 days. The CA signs the certificate of each enrolled server, and each renewal of it ([Enrollment and the install PKI](/reference/enrollment/index.html.md#install-ca)). The reference pages describe no renewal of the install CA, so report the alert to the maintainers of hopbox.

### HopboxHeartbeatLate

Warning. The last heartbeat of a server stayed older than 15 seconds for 1 minute. The server sends one every 5 seconds. At 20 seconds, the leader marks the server unresponsive, and HopboxServerUnresponsive fires. This alert does not fire in maintenance mode, when the control role ends each channel as planned. Check that `hopboxd` runs on the server and that it reaches the control server.

### HopboxServerUnresponsive

Critical. The leader marked servers unresponsive, and they stayed so for 2 minutes. An unresponsive server sent no heartbeat for 20 seconds. Its machines stay on it, but their leases end, and their machine URLs answer 503. Run `hopboxd admin server list` to find the servers, and check that `hopboxd` runs on each one.

### HopboxServerDegraded

Warning. The last heartbeat of a server had the status `degraded`: a check of severity `fail` failed, such as a missing `/dev/kvm` ([the preflight check](/reference/preflight/index.html.md#checks)). Placement puts no new machine on the server. Run `hopboxd preflight` on that server to see the check.

### HopboxVersionSkew

Info. The servers of the install ran two or more releases of `hopboxd` for more than 24 hours. An upgrade that stopped halfway leaves this state. A gap of less than 1 hour in `hopbox_server_info`, such as a short maintenance window, changes nothing. A longer gap pauses a firing alert from 1 hour after the start of the gap until the samples return. The alert then pauses a second time, from 24 hours after the start of the gap until about 23 hours after its end. After a gap of a day or more, the two pauses join, so the alert stays quiet for about 23 hours of new samples, also when the releases still differ. Run `hopboxd admin server list` to see the release of each server, and upgrade the rest ([Install hopbox on a Debian 13 server by hand](/guide/install-debian/index.html.md#upgrade)).

### HopboxChannelDown

Critical. A server held no control channel for 1 minute. Without its channel, the server gets no new routes, zones, or leases. This alert does not fire in maintenance mode, when the control role ends each channel as planned. Check that the control server runs and that the server reaches it. Read the log of `hopboxd` on the server for the reason.

### HopboxLeaseLosses

Warning. A node lost machine leases in the last 15 minutes. The executor then stops its actions on those machines ([the node executor](/reference/executor/index.html.md#metrics)). This alert does not fire in maintenance mode, or in the 16 minutes after it, because no lease renews in maintenance mode. A lost control channel often causes it. Look at HopboxChannelDown and HopboxHeartbeatLate.

### HopboxMachineCrashLoop

Warning. The restart policy restarted a machine 3 or more times in the last 15 minutes. The alert names the machine, its name, and the ID of its org. The label `reason` of `hopbox_machine_restarts_total` says why: `crash`, `guest_shutdown`, or `start_command_exit`. Read the logs of the machine. A start command that exits at once is a common cause.

### HopboxCPUContention

Warning. A process of a machine waited for a CPU more than 20% of the time for 30 minutes. The machines of the node compete for the CPU. Look at the top machines by CPU on D3, and at HopboxNodeCPUBusy.

### HopboxMachineOOM

Warning. The kernel of the node killed processes of the unit of a machine for a lack of memory in the last 15 minutes, such as its Firecracker process. A kill inside the guest does not count. Look at the RAM of the node on D2, and at HopboxNodeRAMFull.

### HopboxDiskWriteHeavy

Info. A machine wrote more than 100 MB/s to the disks of the node for 30 minutes. It slows the disks of the other machines, and fills its sparse disk. Look at the disk of the machine on D3, and at HopboxDiskFull.

### HopboxdRestarting

Warning. systemd restarted `hopboxd.service` 3 or more times in the last 15 minutes, after it failed. A restart by `systemctl restart` does not count. Read the log of `hopboxd` for the error before each stop:

```sh
journalctl -u hopboxd --since '-30 min' | grep '"level":"ERROR"'
```

### HopboxImageFetchFailing

Warning. 2 or more fetches of an image for a boot failed on a node in the last hour. Boots of machines with that image fail. Run `hopboxd admin image list` to check the images, and read the log of `hopboxd` on the node.

### HopboxGuestLinkDown

Warning. Machines ran on a node without an authenticated guest link for 5 minutes. Exec, file copies, and the start command of those machines do not work. A machine in standby does not count, because the node closed its link on purpose. This alert also fires for a resume whose guest agent does not connect again: the guest runs, and `HopboxWakesFailing` stays quiet, because such a resume counts one failure. The machine stays in the state `standby` until its link comes back. Read the logs of the machines, and the log of `hopboxd` on the node.

### HopboxWakesFailing

Warning. Wakes or resumes of machines failed on a node for 10 minutes. Each machine keeps its memory, and its executor tries again. A machine in the state `standby` stays paused until a resume works. A machine whose wake failed is `failed`, and keeps its memory file. Its executor tries the wake again only with the restart policy `always`. Read the log of `hopboxd` on the node for `cannot run the machine again` ([the node executor](/reference/executor/index.html.md#standby)), and for `a machine action failed` with the error of the wake ([the node executor](/reference/executor/index.html.md#the-end-of-a-wake)). Read the journal of the unit of the machine too. A memory file that never wakes stays until its creator drops it ([Standby and hibernate](/guide/standby-and-hibernate/index.html.md#a-failed-wake)).

### HopboxSavedStateDiskLow

Warning. A node refused a save of the memory of a machine in the last 10 minutes. The save would leave less free disk than the floor of the node, `vmm.save_free_percent` and `vmm.save_free_bytes` ([the bootstrap config of hopboxd](/reference/hopboxd-config/index.html.md#keys)). The machine runs on, and its hibernate fails with `resource_exhausted`. The alert ends 10 minutes after the last refusal. Read the log of `hopboxd` on the node for `refused the save of a hibernate, because the node has too little free disk`, which names the free bytes and the floor ([the node executor](/reference/executor/index.html.md#the-gate-of-a-save)). Check the free disk of `<data_dir>/machines` with `df -h /var/lib/hopbox/machines`, and the panel "Saved state by kind" of D9. Free disk on the node, or move machines to another node. Lower the floor only on a small node whose disk holds less than the default of 50 GiB.

### HopboxGuestLinkProofFailed

Warning. A guest agent on a node failed the proof of its guest link in the last 15 minutes. The proof does not match the link key of the machine. A guest agent that someone changed inside the machine can cause it. Find the machine in the log of `hopboxd` on the node.

### HopboxSpoofing

Warning. A node dropped packets of a machine with a forged source for 5 minutes: a source that is not an address of the machine. The node drops them, but the machine tries to hide its traffic. Stop the machine with `hopboxd admin machine stop --id=ID`, and handle it as an abuse report.

### HopboxEgressFailClosed

Critical. A node had no egress edge for 1 minute, so the outbound traffic of its machines fails closed. Look at HopboxUplinkDown on the edge servers, and check the egress ([the egress of the machines](/reference/egress/index.html.md#check-the-egress)).

### HopboxUplinkDown

Critical. The check of the uplink of an edge server failed for 1 minute. The edge is then no egress exit, so the nodes use the other egress edges. Check the network of the edge server ([the egress of the machines](/reference/egress/index.html.md#the-uplink-of-an-egress-edge)).

### HopboxEdgeConnectionsHigh

Warning. The open connections of a listener of an edge stayed above 80% of its limit for 10 minutes. At the limit, the edge refuses new connections. A flood of clients can cause it. D7 shows the connections and the refusals of each listener.

### HopboxEdgeUpstreamErrors

Warning. More than 10% of the requests to machine URLs of running machines got a 502 or a 503 for 10 minutes. The label `reason` of `hopbox_edge_upstream_errors_total` says why: `no_address`, `no_answer`, or `expired_lease` ([the edge role](/reference/edge/index.html.md#metrics)). `no_answer` means that the machine does not answer on its URL port. The reasons `not_running` and `waking` do not count, because a bot that polls a stopped or sleeping machine makes them. `hopbox_edge_wakes_total` and HopboxWakesFailing cover the wakes. D7 shows each reason, also `not_running`. This alert does not fire in maintenance mode, when each machine URL answers `expired_lease` as planned.

### HopboxEdgeRoutesStale

Critical. The leases of targets of machine URLs ended at an edge for 2 minutes, so those URLs answer 503. The node of those machines did not confirm them lately. This alert does not fire in maintenance mode, when the leases end as planned. Look at HopboxServerUnresponsive and HopboxChannelDown.

### HopboxWakeBurst

Warning. An edge sent more than 30 resumes and wakes of sleeping machines a minute to their nodes for 10 minutes: the results `woken`, `failed`, and `timeout`. A refused wake moves no RAM, so it does not count. A wake starts only after the check of access of a connection, so a scan of the SSH ports wakes nothing. Many requests to public URLs, or a client that connects in a loop, cause it. Each wake takes the RAM of its machine on its node. Read the log of `hopboxd` on the edge for the machines: each wake logs `waking a machine for a connection` ([the edge role](/reference/edge/index.html.md#wake-on-connect)). D9 shows the wakes by result.

### HopboxSSHRefusalBurst

Info. The SSH front door refused 500 or more keys in the last hour. A bot that tries keys causes it. The limits of the front door hold it back, so this alert needs no action. Read the log of `hopboxd` on the edge for the clients.

### HopboxSSHLostOpens

Warning. More than 1,000 channel opens of the SSH front door got no answer and still wait after their connection ended ([the SSH front door](/reference/ssh-front-door/index.html.md#channels-and-requests)). They hold memory of the edge. Restart `hopboxd` on the edge at a quiet time.

### HopboxDNSStale

Warning. A DNS server served an older serial of the zones than another DNS server for 5 minutes. It lost its control channel, or it serves the zones on its disk ([the DNS role](/reference/dns/index.html.md#the-serial-of-each-dns-server)). Look at HopboxChannelDown for that server. An install with one DNS server never fires this alert.

### HopboxFirewallRepairs

Warning. `hopboxd` installed the table `inet hopbox_host` of a server again in the last 15 minutes, after it lost a part outside `hopboxd`. Another program, such as a firewall tool, changed the rules. Find that program, and check the table ([the host firewall](/reference/host-firewall/index.html.md#check-the-firewall)).

### HopboxMeshPeerStale

Warning. The last WireGuard handshake of a server with a peer of the mesh stayed older than 3 minutes for 2 minutes. The label `server` names the peer. A peer without any handshake fires too. Each peer has a persistent keepalive of 25 seconds, so a healthy peer makes a new handshake about every 2 minutes, also without traffic. The 2 minutes keep one lost handshake packet from an alert. So this alert means a broken link, also before traffic needs that link. Run `wg show hbm0` on the server, and check that the peer reaches UDP port 51820 ([the mesh](/reference/mesh/index.html.md#values)).

### HopboxMeshPeerMissing

Warning. The mesh device of a server had fewer peers than the last push of the control role named, for 5 minutes. The device refused the mesh, or lost a peer. Read the log of `hopboxd` for `the mesh device refused the mesh of the control plane` ([the mesh](/reference/mesh/index.html.md#refused-mesh)).
