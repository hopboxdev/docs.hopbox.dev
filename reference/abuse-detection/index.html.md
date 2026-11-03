# Abuse detection

> The detection of abuse inside hopboxd: its thresholds, its signals for each account, its leader job, and its notices.

`hopboxd` detects signs of abuse for each account by itself, without Prometheus, as section 3.2 and the decisions Q150 to Q153 of the abuse plan describe. It watches the same signals as the alert rules of the monitoring bundle, but it names the account behind each one. The thresholds are part of the install config.

## Thresholds

Each threshold is a value of the install config ([the admin API](/reference/admin-api/index.html.md#installconfigservice)). An instance admin changes it with `hopboxd admin config set` ([hopboxd](/reference/hopboxd/index.html.md#admin-config-set)), or on the page Config of the Admin area. Each default is the number of the alert of the monitoring bundle with the same signal, from section 4.4 of the abuse plan.

| Setting | Bounds | Default | Alert of the bundle |
| --- | --- | --- | --- |
| `abuse_cpu_percent` | 1 to 100 | 90 | `HopboxMachineCPUSustained` |
| `abuse_cpu_for` | 10 min to 6 h | 2 h | `HopboxMachineCPUSustained` |
| `abuse_egress_bytes_per_second` | 1 to 100,000,000,000 | 50,000,000 | `HopboxEgressHigh` |
| `abuse_egress_for` | 5 min to 6 h | 30 min | `HopboxEgressHigh` |
| `abuse_scan_connections_per_second` | 1 to 1,000,000 | 100 | `HopboxEgressScan` |
| `abuse_scan_for` | 5 min to 6 h | 10 min | `HopboxEgressScan` |
| `abuse_org_burst` | 1 to 10,000 | 5 | `HopboxOrgOrKeyBurst` |
| `abuse_api_key_burst` | 1 to 10,000 | 5 | `HopboxOrgOrKeyBurst` |
| `abuse_long_api_key_burst` | 1 to 10,000 | 2 | `HopboxLongKeyBurst` |
| `abuse_machine_burst` | 1 to 10,000 | 10 | `HopboxMachineBurst` |
| `abuse_url_churn` | 1 to 10,000 | 20 | `HopboxURLChurn` |
| `abuse_new_account_age` | 1 h to 90 days | 7 days | none |

Each duration is whole minutes, the step of the history of samples. `HopboxSignUpBurst` has no signal here, because a new account comes from no account. The windows of a machine end at 6 hours, so one run reads a bounded part of the history, which keeps 24 hours. A value out of its bounds fails with `invalid_argument`, and the install config keeps its old value. A change of a threshold applies from the next run.

## Signals

Each signal has a name, which the notices and the audit entries use. A signal of a machine comes from the history of samples, the values of the metrics once a minute ([Metrics](/reference/metrics/index.html.md)). The other signals come from the audit log and the control store.

| Signal | Active while | Source |
| --- | --- | --- |
| `cpu_sustained` | A running machine uses `abuse_cpu_percent` or more of its vCPUs, averaged over 30 minutes, at each sample of the last `abuse_cpu_for`. | `hopbox_machine_cpu_seconds_total` and the label `vcpus` of `hopbox_machine_info` |
| `egress_high` | A running machine sends `abuse_egress_bytes_per_second` or more, as a rate over 5 minutes, at each sample of the last `abuse_egress_for`. | `hopbox_machine_egress_bytes_total` |
| `egress_scan` | A running machine opens `abuse_scan_connections_per_second` or more new outbound connections per second, as a rate over 5 minutes, at each sample of the last `abuse_scan_for`. | `hopbox_machine_egress_connections_total` |
| `org_burst` | An account made `abuse_org_burst` or more orgs in the last hour. | the audit action `org.create` |
| `api_key_burst` | An account made `abuse_api_key_burst` or more API keys in the last hour. | the audit action `api_key.create` |
| `long_api_key_burst` | An account made `abuse_long_api_key_burst` or more API keys with the preset lifetime of 365 days in the last hour. A key with a custom expiry does not count, as in the label `lifetime` of `hopbox_api_key_creations_total`. | the audit action `api_key.create`, with its detail `lifetime` set to `365d` |
| `machine_burst` | An account made `abuse_machine_burst` or more machines in the last hour. | the audit action `machine.create` |
| `url_churn` | An account changed the port or the visibility of machine URLs `abuse_url_churn` or more times in the last hour. | the audit actions `machine.set_url_port` and `machine.set_url_visibility` |
| `new_account_public_url` | An account younger than `abuse_new_account_age` has a machine with a public URL that is not deleted. | the table `machines`, and `created_at` of the account |

A signal of a machine takes the lowest rate of the points in its window, as `min_over_time` of the rule. A point without two samples in its rate window, such as in a gap of a restart of the node, has no rate and does not count. So such a gap does not end the signal. As `present_over_time` of the rule, a rate must exist in the first part of the window: the first 15 minutes for the CPU, 5 minutes for the egress, and 3 minutes for the connections, or the whole window when it is shorter. With the defaults, a new machine at full CPU thus counts after about 1 hour and 46 minutes, and the rule fires after about 1 hour and 48 minutes: its first rate after 1 minute, its range of `present_over_time` after 106 minutes, and its 2 minutes of `for`. A machine without a sample of `hopbox_machine_info` in the last 3 minutes does not count, because it stopped or its node is gone.

The account of a machine is its creator. For a machine that an org API key made, it is the account that made the key. The account of an audit entry is its actor. For an org API key, it is again the account that made the key. An entry of an admin key, of root, or of the system has no account, and does not count. Only the creator of a machine changes the visibility of its URL, so the creator of a public machine made it public. For a machine that an org API key made, the account that made the key counts. A deleted account counts no more, so its signals end at the next run, and that run closes their notices.

Each run gives one detection for each signal and account. A detection of a machine signal names its machines, the worst first, at most 20, with the count of the others, and the org of each one. Its value is the worst value. A machine without a creator and without the account of its key gives a detection of its org. The value of a burst is its count, and the value of `new_account_public_url` is the count of public machines.

## Reads of each run

Each run reads the thresholds, then three sources. The history of samples gives each family of a machine signal over the window of its own signal and the rate before it, plus 2 minutes, and `hopbox_machine_info` over the last 4 minutes. The longest read is 6 hours and 32 minutes, for the CPU with the longest `abuse_cpu_for`. The audit counts come from the table `audit_log`. The partial index `audit_log_abuse` holds only the five actions that the detection counts, by time ([Control store tables](/reference/control-store/index.html.md#audit_log)). So the count of the last hour reads only the rows of that hour and of those actions. The public machines of new accounts come from the partial index `machines_public` on the creator and the creator key of each machine with a public URL that is not deleted.

## The leader job

The leader of the control role runs the detection every minute, as the leader job `abuse_detection` ([hopboxd](/reference/hopboxd/index.html.md#leader-jobs)). A follower runs nothing. A run first checks the leader lease, and a leader that lost its lease reads and changes nothing. Then it reads the signals outside a transaction. Last, one short transaction checks the lease again and writes the changes. A run takes at most 30 s. A run that this timeout cuts counts as `failed` in `hopbox_leader_job_runs_total{job="abuse_detection"}` ([Metrics](/reference/metrics/index.html.md#metrics-of-the-leader)). Maintenance mode and the fence of the process pause the job, as they pause every leader job.

A run compares the signals that it finds with the active signals of the table `abuse_signals` ([Control store tables](/reference/control-store/index.html.md#abuse_signals)):

1. A signal of an account that has no active row starts. The run writes the audit entry `abuse.detect` and a row of `abuse_signals`, and gives each instance admin a notice of the kind `abuse`.
2. A signal that has an active row repeats. The run updates the value, the machines, and the time `updated_at` of the row. It writes no audit entry. An instance admin who got no notice of the signal yet, such as a new instance admin, gets one now.
3. An active row that the run does not find ended. The run sets `resolved_at`, closes the notices of the signal for every instance admin, and writes the audit entry `abuse.resolve`.
4. The run deletes the rows that ended more than 30 days ago.

The detection acts on no account and no machine (Q124). A wrong stop loses the work of a customer, so an instance admin decides.

Each start logs `an abuse signal started` at level info, with `signal`, `account`, `org`, `value`, `machines`, and `audit_entry`. Each end logs `an abuse signal ended`, with `signal`, `account`, `org`, `started_at`, and `audit_entry`. A run that fails logs `the detection of abuse failed` at level warn, with `component=leader` and `err`.

## Notices

Each signal of an account has one notice for each instance admin while it is active. A repeat updates the notice, and the end closes it. The account is the target of the notice. A signal of an org without an account names the org.

The summary of a notice gives the signal with its value at the last run, the threshold, and the window. For a machine signal, it names the machines. It ends with the time of the last run that found the signal. For example:

```text
The account user1@example.org (<account id>): the machine box1 used 95% or more of the vCPUs, averaged over 30 minutes, for the last 2h0m0s. The threshold is 90%. Last seen at 2026-11-01T12:00:00Z. The notice closes by itself when the signal ends.
```

The advice says that hopbox stops nothing by itself, that legitimate work can look the same, and how to suspend the account: the page Accounts of the Admin area, or `sudo hopboxd admin account suspend --account-id='<id>' --reason='TEXT'`. For an org without an account, it names `sudo hopboxd admin org suspend`. A dismiss closes the notice on the page of one instance admin only. The column `notified` of the signal keeps each instance admin who got the notice, so a repeat of the signal does not open a dismissed notice again.

The signals also share the list of the alerts of the monitoring bundle. While a signal is active, "Needs your attention" of the page Overview lists it next to the alerts, with the tag `abuse` and the link "See signal". After its end, the section "Resolved alerts" of the page Notices lists it for 30 days, with its start and its end ([Console](/reference/console/index.html.md#admin-area)). The view `alert_list` gives both kinds in one order ([Control store tables](/reference/control-store/index.html.md#alert_list)). The tile "Open notices" counts the alerts and the signals apart, such as "1 alert and 2 abuse signals fire". Each signal has the ID of its audit entry `abuse.detect` as its `id` in the API.

## Email

The detection emails the instance admins only while the install has no token of the alert webhook ([Metrics](/reference/metrics/index.html.md#alert-webhook)), so only when no Prometheus sends its alerts to `hopboxd` (Q152). With a token, the alerts of Prometheus reach the admins by their own email, and the detection makes the notices alone.

Each run that starts signals sends one email to each email of each instance admin who got a notice of them. The email names the first 20 new signals, and counts the rest, as the email of a call of Alertmanager does (Q148). A repeat and an end send no email. The email goes out in the background after the commit, with the limits of the email of alerts. A stop of the control role waits for it, at most until the stop timeout. A run that ends while the control role stops sends no email, and logs `the control role stops, so the email of new abuse signals goes out to nobody` at level info, with `signals`. Its notices stay. A failed email writes the audit entry `admin_notice.mail_failed`, as for any notice.

The subject is `Abuse signal: <signal> on <domain>` for one signal, and `Abuse signals: <n> new signals on <domain>` for more. For more than one signal, the text starts with a count of the signals of each name. Then each signal has the line `Signal <signal>.` with the summary of its notice, its start, its audit entry, and its advice:

```text
Subject: Abuse signals: 2 new signals on example.com

This is a notice of the detection of abuse of hopboxd to the instance admins of example.com.

2 new abuse signals: cpu_sustained once, org_burst once.

Signal cpu_sustained. The account user1@example.org (<account id>): the machine box1 used 95% or more of the vCPUs, averaged over 30 minutes, for the last 2h0m0s. The threshold is 90%. The notice closes by itself when the signal ends.
Started: Sun, 01 Nov 2026 12:00:00 UTC
Audit entry: <audit entry id>
hopbox stops nothing by itself. ...
```

## Audit entries

| Action | Actor | Target | Details |
| --- | --- | --- | --- |
| `abuse.detect` | `system` | `account` and its ID, or `org` and its ID for a signal without an account | `signal`, `measured` (the value at the start, such as `95%`), `threshold`, `window` for a signal with a window, and for a machine signal `machines` (the IDs), `machine_names`, `machine_orgs`, and `more_machines` when the signal has more than 20 machines. A signal of an org also has `org`. |
| `abuse.resolve` | `system` | the target of its `abuse.detect` | `signal`, `detect_entry` (the ID of the entry `abuse.detect`), and `started_at` |

The audit log keeps both entries for good, also after the delete of the row of the signal.

