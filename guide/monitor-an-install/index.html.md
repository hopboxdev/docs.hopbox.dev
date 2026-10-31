# Monitor an install with Prometheus and Grafana

> How to set up the monitoring bundle on the control server: Prometheus, Alertmanager, node_exporter, and Grafana from Debian packages, the alert email, the outside check, a test alert, and what each alert means.

This guide sets up the monitoring bundle on the control server of an install. The monitoring bundle is the set of files that hopbox ships for standard monitoring tools. Prometheus scrapes `hopboxd` and node_exporter, and runs the alert rules. Alertmanager sends each alert by email, and sends the alert `Watchdog` to an outside check. Grafana shows the dashboards D1 to D4, and it is optional.

The tools come from Debian packages. hopbox does not run them or upgrade them. Each tool listens on a loopback address only, as the metrics listener of `hopboxd` does ([Metrics](/reference/metrics/index.html.md#listener)):

| Tool | Debian 13 package | Listen address |
| --- | --- | --- |
| Prometheus | `prometheus` 2.53.3+ds1-2 | `127.0.0.1:9090` |
| Alertmanager | `prometheus-alertmanager` 0.28.1+ds-1 | `127.0.0.1:9093` |
| node_exporter | `prometheus-node-exporter` 1.9.0-1+b4 | `127.0.0.1:9100` |
| Grafana | `grafana` 13.3.0 from the Grafana apt repository, because Debian has no Grafana package | `127.0.0.1:3000` |

The versions come from these sources on 2026-10-31:

- The Debian versions are those of the trixie Packages index at `https://deb.debian.org/debian/dists/trixie/main/binary-amd64/Packages.xz`. trixie-updates and trixie-security had no newer version. A Debian 13 container installed these versions, and ran the commands of steps 1 to 5 without systemd and without an SMTP server.
- `make check-monitoring` runs promtool and amtool from the Docker Hub images `prom/prometheus:v2.53.3` and `prom/alertmanager:v0.28.1`, the same upstream versions as the Debian packages. The `Makefile` pins each image by the digest that Docker Hub gives for its tag.
- Grafana 13.3.0 was the newest version in the Packages index of `https://apt.grafana.com`, at `dists/stable/main/binary-amd64`. The dashboards load in Grafana 13.3.0.

The host firewall of hopbox also blocks inbound traffic to these ports ([the host firewall](/reference/host-firewall/index.html.md)). Prometheus keeps the samples for 30 days, and scrapes every 15 seconds. On ks, the stack takes less than 0.5 GB of RAM and about 1.2 GB of disk for 30 days (plan section 4.7).

## What you need

- A control server of the install, with the release bundle in `/opt/hopbox/current` ([Install hopbox on a Debian 13 server by hand](/guide/install-debian/index.html.md)). Run every command of this guide as root on that server.
- The SMTP server of the install, with a login for the alert email. `hopboxd admin config smtp get` prints the host, the port, and the login of the install ([hopboxd](/reference/hopboxd/index.html.md#admin-config-smtp-get)). The password of that login is not in the output.
- An email address that gets the alerts, such as `admin@example.org`.
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
| `grafana/grafana-server.conf` | `/etc/systemd/system/grafana-server.service.d/hopbox.conf` |
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

## 3. Set the email and the outside check

Open `/etc/prometheus/alertmanager.yml`, and replace each `example.org` value:

| Key | Value |
| --- | --- |
| `smtp_smarthost` | The host and the port of the SMTP server, such as `smtp.example.org:587`. Port 587 uses STARTTLS, and port 465 uses TLS from the first byte. |
| `smtp_from` | The sender of the alerts, such as `hopbox alerts <alerts@example.org>` |
| `smtp_auth_username` | The SMTP login. For a relay on the same host without a login, remove this line and the line `smtp_auth_password_file`. |
| `to` of the receiver `email` | The address that gets the alerts |

Alertmanager reads the SMTP password and the ping URL from files of their own, outside the sealed store of hopbox. Only root and the user `prometheus` can read them. Write the password: type it at the prompt, then press Enter.

```sh
install -m 0640 -o root -g prometheus /dev/null /etc/prometheus/alertmanager-smtp-password
read -rs password && printf '%s' "$password" > /etc/prometheus/alertmanager-smtp-password; unset password
```

Write the ping URL of the outside check in the same way, such as `https://hc-ping.com/<uuid>`:

```sh
install -m 0640 -o root -g prometheus /dev/null /etc/prometheus/alertmanager-watchdog-url
read -r url && printf '%s' "$url" > /etc/prometheus/alertmanager-watchdog-url; unset url
```

Check the two configurations:

```sh
promtool check config /etc/prometheus/prometheus.yml
amtool check-config /etc/prometheus/alertmanager.yml
```

Each tool prints `SUCCESS`. `promtool` also checks the two rule files, and prints `SUCCESS: 22 rules found` and `SUCCESS: 32 rules found`.

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

Within 1 minute, the address of step 3 gets an email whose subject starts with `[FIRING:1] HopboxTestAlert`. The test alert ends by itself after 5 minutes, and a second email says `[RESOLVED]`. If no email arrives, read the log of Alertmanager:

```sh
journalctl -u prometheus-alertmanager --since '-10 min' | grep -i 'notify'
```

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

`apt-mark hold` keeps `apt-get upgrade` from a move to a new Grafana. Copy the files of the bundle, and start Grafana:

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

Grafana listens on loopback, so you reach it through SSH. On your computer, open a tunnel to the control server:

```sh
ssh -L 3000:127.0.0.1:3000 root@control1.example.org
```

A control server that is not an edge server takes SSH only through an edge server as a jump host ([the host firewall](/reference/host-firewall/index.html.md#the-jump-hosts)). Then add the jump host:

```sh
ssh -J root@edge1.example.org -L 3000:127.0.0.1:3000 root@control1.example.org
```

Open `http://127.0.0.1:3000` in a browser, and sign in as `admin`. The folder `hopbox` holds the four dashboards:

| Dashboard | Content |
| --- | --- |
| hopbox D1 Overview | Targets down, firing alerts, nodes, running machines, the host firewall, the time until the certificate ends, API errors, executor failures, and the memory of `hopboxd` |
| hopbox D2 Capacity | vCPUs, RAM, and disk per node and for the install, allocated against capacity, and the CPU and RAM of each host |
| hopbox D3 Machines | The top 10 machines by CPU share, RAM, disk, egress, inbound traffic, and new outbound connections. Each line shows the name of the machine and the ID of its org. |
| hopbox D4 Accounts and abuse | New accounts, orgs, API keys, machines, and URL changes in the last hour, from the recording rules that the alerts read. Also accounts by state, rate limits, org limits, and audit actions. |

The same tunnel works for Prometheus on port 9090 and Alertmanager on port 9093.

The pages "Overview", "Capacity", "Machines", and "Accounts" of the Admin area can link to these dashboards. For the links, set the URL of Grafana in the install config on a control server ([Use the Admin area](/guide/admin-area/index.html.md)). For the tunnel above, the URL is `http://127.0.0.1:3000`:

```sh
sudo hopboxd admin config set --grafana-url http://127.0.0.1:3000
```

## After an upgrade of hopboxd

An upgrade of `hopboxd` does not copy the files of the bundle. The rules of a release read the metrics of that release, so copy the new files after each upgrade. Do not copy `alertmanager.yml` again: the copy has no values of step 3, so every alert email fails, and `Watchdog` still pings.

1. Compare the files of the release with the files of the server. `diff` prints nothing for a file that did not change.

   ```sh
   m=/opt/hopbox/current/monitoring
   diff /etc/default/prometheus "$m/prometheus/prometheus"
   diff /etc/default/prometheus-alertmanager "$m/alertmanager/prometheus-alertmanager"
   diff /etc/default/prometheus-node-exporter "$m/node-exporter/prometheus-node-exporter"
   diff /etc/prometheus/alertmanager.yml "$m/alertmanager.yml"
   ```

2. If the last `diff` shows a change apart from your values of step 3, add that change to `/etc/prometheus/alertmanager.yml` by hand.

3. Copy the other files of step 2.

   ```sh
   install -m 0644 "$m/prometheus.yml" /etc/prometheus/prometheus.yml
   install -m 0644 "$m/prometheus/prometheus" /etc/default/prometheus
   install -m 0644 "$m/rules/hopbox-recording.yml" "$m/rules/hopbox-alerts.yml" /etc/prometheus/rules/
   install -m 0644 "$m/alertmanager/prometheus-alertmanager" /etc/default/prometheus-alertmanager
   install -m 0644 "$m/node-exporter/prometheus-node-exporter" /etc/default/prometheus-node-exporter
   ```

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

7. If you run Grafana, copy the Grafana files of step 6 again, then run `systemctl daemon-reload` and `systemctl restart grafana-server`.

## Change a rule

The thresholds of the alerts are defaults that you can change. To change one on an install, edit `/etc/prometheus/rules/hopbox-alerts.yml`, check it, and reload Prometheus:

```sh
promtool check rules /etc/prometheus/rules/hopbox-alerts.yml
systemctl reload prometheus
```

The copy of the next release replaces the file, so make the change again after each copy.

To change a rule of hopbox itself, edit `monitoring/rules/hopbox-alerts.yml` in the repository, and its test in `monitoring/rules/hopbox-alerts_test.yml`. Each alert has a case that fires and a case that does not. Then run `make check-monitoring`. It runs `promtool test rules` and `amtool` in Docker, at the versions of Debian 13, and CI runs it too. A new alert also needs a case in a rule test and an entry in the next section, or the docs check fails.

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
- Alertmanager fails to send email, and holds back `Watchdog` while HopboxAlertEmailFailing fires.

Read the log of Alertmanager first. If the command shows `Notify attempt failed` for the receiver `email`, the email path fails: go to HopboxAlertEmailFailing. If the command fails or shows nothing, check that the control server, Prometheus, and Alertmanager run.

```sh
journalctl -u prometheus-alertmanager --since '-30 min' | grep -i 'notify'
```

### HopboxAlertEmailFailing

Warning. Alertmanager failed to send alert email in the last 15 minutes. The email of this alert also fails, so Alertmanager then holds back `Watchdog`. The outside check gets no more pings, and emails you by its own path. Read the log of Alertmanager, and check the values and the password file of step 3:

```sh
journalctl -u prometheus-alertmanager --since '-30 min' | grep -i 'notify'
```

### HopboxRuleFailures

Warning. Prometheus failed to evaluate the rules of a group in the last 15 minutes, so the alerts of that group cannot fire. The label `rule_group` names the file and the group. Read the log of Prometheus. A common cause is a rule that you changed by hand ([Change a rule](#change-a-rule)).

```sh
journalctl -u prometheus --since '-30 min' | grep -i 'rule'
```

### Alerts that come later

Milestone O7 adds the alerts of the P2 metrics, each with an entry here (plan section 5). Some read metrics that `hopboxd` does not serve yet: `HopboxLeaderJobFailing`, `HopboxServerUnresponsive`, and `HopboxMailFailing`. Others read metrics that exist, but are not P1 metrics: `HopboxEdgeConnectionsHigh`, `HopboxSSHRefusalBurst`, `HopboxSSHLostOpens`, `HopboxFirewallRepairs`, `HopboxMeshPeerStale`, and `HopboxDNSStale`.
