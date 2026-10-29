# The certificate of the install

> The wildcard certificate of the install: the ACME settings, the leader job that gets and renews it, the name server check, the backoff, where it lives, its logs, and its tests with Pebble.

The edge servers serve HTTPS for the domain of the install and for every machine name below it. One wildcard certificate covers both, such as `example.com` and `*.example.com`. ACME is the protocol that a certificate authority (CA), such as Let's Encrypt, uses to issue certificates automatically (RFC 8555). This page covers the ACME settings of the install, the leader job that gets the certificate and renews it, and where the certificate lives. The code is in `internal/acme` (the job) and `internal/service/acme.go` (the store). The job uses [acmez](https://github.com/mholt/acmez).

## Settings

The ACME settings are part of the settings of the install ([the admin API](/reference/admin-api/index.html.md#installconfigservice)). An instance admin sets them with the command `admin config set` of `hopboxd` ([hopboxd](/reference/hopboxd/index.html.md#admin)).

| Setting | Flag of `admin config set` | Default | Description |
| --- | --- | --- | --- |
| `acme_directory_url` | `--acme-directory-url` | `https://acme-v02.api.letsencrypt.org/directory` | The ACME directory of the CA. The directory is the URL from which an ACME client reads every other URL of the CA. The default is the production CA of Let's Encrypt. It must be an `https` URL of a host, with at most 2048 characters, and without user info or a fragment. It holds no whitespace and no control character. A notice to the instance admins puts it in a shell command in single quotes ([the admin API](/reference/admin-api/index.html.md#notices-to-instance-admins)). |
| `acme_email` | `--acme-email` | empty | The contact email of the ACME account. The CA can send mail about the account to it. An empty value removes the contact from the install, but the CA keeps the old contact of the account. A new email updates the contact at the CA. hopbox keeps it in lowercase. It holds no whitespace and no control character, as the directory. |
| `acme_terms_agreed` | `--acme-agree-terms`, `--no-acme-agree-terms` | `false` | `true` records that an instance admin agrees to the terms of service of the CA of the directory. The install asks for no certificate before that. |

The inputs of the ACME job are the domain, the public addresses, and these three settings. The public addresses name the name servers that the job checks ([The name server check](#the-name-server-check)).

The agreement covers the terms that the directory names at the next attempt of the job (`meta.termsOfService`, RFC 8555 section 7.1.1). The first attempt after the agreement records the URL of those terms. `hopboxd admin certificate get` prints it as `terms_url`. A CA can publish new terms. Let's Encrypt does so about once a year, and does not ask its accounts to agree again. So when the directory names a new terms URL, the job records it, and writes an audit entry and a warning. Then it goes on with its renewals. After the warning, read the new terms. A directory without a terms URL changes nothing.

A CA can want an action of the account holder, such as an agreement to new terms. Then it refuses the requests of the account with the error `urn:ietf:params:acme:error:userActionRequired` (RFC 8555 section 7.3.3). The job then clears the agreement, and records the failure. Until a new agreement or a new directory, `renewals_stopped` says `the CA asked an instance admin to act: read its new terms and agree again`. It asks for no certificate until an instance admin reads the terms and agrees again. A change of the directory clears the agreement too.

A new directory names another CA, with other terms. So a change of `acme_directory_url` sets `acme_terms_agreed` to `false`, unless the same update sets `acme_terms_agreed` too. An update with the directory that the install has already keeps the agreement.

A certificate from another directory is due at once. So a change of the directory replaces the certificate that works with one from the new CA, at the first attempt that passes. After a move from a trusted CA to a test CA, every browser refuses the HTTPS of the install. The staging CA of Let's Encrypt is such a test CA. Test a new CA on an install that has no trusted certificate yet.

Let's Encrypt has a staging CA for tests: `https://acme-staging-v02.api.letsencrypt.org/directory`. Its certificates are not trusted by browsers, and its rate limits are higher.

Each change of an input of the ACME job publishes the change event `acme.config_changed` in the transaction of the change ([Control store tables](/reference/control-store/index.html.md#events)).

Example:

```sh
sudo hopboxd admin config set --acme-email ops@example.org --acme-agree-terms
```

## The ACME job

The leader of the control role runs the ACME job ([hopboxd](/reference/hopboxd/index.html.md#leader-jobs)). The job starts at the start of each term of the leader, and stops at its end. It reads its state from the control store, so a new leader goes on where the old one stopped.

The job gets a certificate in these cases:

| Case | When |
| --- | --- |
| The install has no certificate. | at once |
| The certificate is for another domain, after a change of the domain. | at once |
| The certificate is from another ACME directory, after a change of the directory. | at once |
| The certificate is due for renewal. | after two thirds of its lifetime, such as day 60 of a certificate of 90 days |

It gets no certificate while the install has no domain, or while `acme_terms_agreed` is `false`. After a failed attempt, it waits until the time of its backoff ([Renewal and backoff](#renewal-and-backoff)). A change of an input of the ACME job ends the wait, unless the CA asked for it. Each change of an input of the ACME job publishes the change event `acme.config_changed`, and the job plans again within a second. Without a change, it reads its state again at least once an hour.

An attempt takes these steps:

1. It records the start of the attempt in `acme_status`. The write checks the leader lease. So a leader that lost its lease starts no order.
2. It reads the directory of the CA, and records the URL of its terms ([Settings](#settings)). It sets the CAA issuers of the zone to the CAA identities that the directory names ([the DNS role](/reference/dns/index.html.md#caa-records)). Then it waits until every name server serves the zone with the new serial ([The name server check](#the-name-server-check)).
3. It reads the ACME account of the install from the control store, and asks the CA for its state. For a new directory, or an account that the CA no longer knows, it makes a new account with a new key. The new account agrees to the terms of the CA, and has the contact `mailto:<acme_email>`. After a change of `acme_email`, it updates the contact at the CA. An empty email keeps the contact that the CA has.
4. It removes every value at `_acme-challenge`, which an attempt that stopped early can leave behind ([the DNS role](/reference/dns/index.html.md#txt-records)).
5. It makes a new ECDSA P-256 key, and orders a certificate for `<domain>` and `*.<domain>` with the challenge type DNS-01.
6. For each of the two authorizations, it adds the value of the challenge at `_acme-challenge` with `Service.AddDNSTXTRecord`. One name holds both values, for the domain and for its wildcard (RFC 8555 section 8.4). Before it asks the CA to check the challenge, it waits until every name server serves the serial of the change.
7. After the check of the CA, it removes each value with `Service.RemoveDNSTXTRecord`. It also removes them after a failed check.
8. It finalizes the order, downloads the first chain, and stores the certificate with its key ([Where the certificate lives](#where-the-certificate-lives)).

Each step that changes the control store checks the leader lease in its transaction. If the lease is stale, the job stops until the next term, and writes nothing more. One attempt has at most 10 minutes. Each request to the CA has at most 1 minute.

## The name server check

The CA asks the name servers of the zone for each challenge. So before the CA checks a challenge, the job checks that each name server serves the change. It asks each name server for the SOA record of the domain, without recursion, over UDP, and over TCP after a cut answer. If its answer is authoritative, with the serial of the change or a later one, the name server passes. The job compares serials with the serial arithmetic of RFC 1982.

| Item | Value |
| --- | --- |
| Name servers | The key `acme.name_servers` of `/etc/hopbox/hopboxd.toml` ([the bootstrap config of hopboxd](/reference/hopboxd-config/index.html.md#keys)). Without it, port 53 at each address of `ns1` and `ns2` of the zone: the first and the second public address of each family ([the DNS role](/reference/dns/index.html.md#the-zone-of-the-install)). |
| Time between two queries | 1 s |
| Time of one query | 2 s |
| Wait at most | 2 minutes. Then the attempt fails with `the name servers <addresses> did not serve the serial <n> of the zone in time`, and the error of the last query. |

An install without a public address has no name server, so the attempt fails with `the install has no public address, so no name server serves the zone. Set the public addresses`.

The control servers must reach port 53 of each name server. On an install with one server, the control role asks the public address of that server itself.

## Renewal and backoff

The job renews the certificate after two thirds of its lifetime. A certificate of Let's Encrypt lives 90 days, so the job renews it on day 60, and 30 days stay for renewals that fail.

`hopboxd admin certificate get` prints the certificate and the state of the attempts ([hopboxd](/reference/hopboxd/index.html.md#admin-certificate-get)). The job does not renew a certificate while the install has no domain or no agreement. Then the command prints the reason as `renewals_stopped`.

A failed attempt keeps the certificate that the install has. It adds 1 to `acme_status.failures`, keeps the error in `last_error`, and sets `next_attempt_at`:

| Failure | Wait |
| --- | --- |
| 1st | 1 minute |
| 2nd | 2 minutes |
| nth | 2^(n-1) minutes |
| 10th and later | 6 hours |

A rate limit of the CA (`urn:ietf:params:acme:error:rateLimited`) waits at least 1 hour. If the detail of the error names a time, such as `retry after 2026-10-18 12:00:00 UTC` from Let's Encrypt, it waits at least until then. An answer of the CA with the status 429 or 503 can have a header `Retry-After` (RFC 9110 section 10.2.3). Its value is a number of seconds or an HTTP date. After a failed attempt, the job also waits at least until the latest time that such a header named. The header can come from any answer of the CA during the attempt. That includes an answer to a request that the job then sent again, and that passed. A wait that the CA asks for, from a rate limit or a header `Retry-After`, lasts at most 7 days. So a wrong time never stops the renewals for longer than that.

A new certificate resets the failures. So does a change of an input of the ACME job, because the change can fix the cause of the failures: the job then tries again at once. A wait that the CA asked for stays, and the status marks it with `ca_wait`. A change of the directory ends that wait too, because it applies only to the old CA. The change keeps the last error until the next attempt, so `hopboxd admin certificate get` still shows it.

## Where the certificate lives

The control store keeps the certificate, its key, and the ACME account ([Control store tables](/reference/control-store/index.html.md#install_certificate)). Both keys are ECDSA P-256 keys, sealed with the current key-encryption key ([Sealing](/reference/sealing/index.html.md#places)). So the database alone never opens them, and a rewrap moves them to a new key-encryption key.

| Table | What it holds |
| --- | --- |
| `acme_account` | The account at the CA of the directory: its URL, its contact, and its sealed key. A new directory gets a new account, which replaces the row. |
| `install_certificate` | The certificate for the domain and `*.<domain>`: the chain in PEM, the sealed key, the serial, the issuer, and the validity. |
| `acme_status` | The attempts that failed since the last certificate, the last error, and the time of the next attempt. |

A new certificate replaces the one before it only after these checks pass. If one fails, the certificate before it stays:

| Check | Error |
| --- | --- |
| The chain holds 1 to 8 PEM blocks of type `CERTIFICATE`, and nothing else, in at most 64 KiB. | `ErrInvalidArgument` |
| The leaf belongs to the new key. | `ErrInvalidArgument` |
| The leaf has the extended key usage `serverAuth`, so TLS clients take it from a server. | `ErrInvalidArgument` |
| Each certificate of the chain, the leaf and each intermediate, is valid now. The CA can run up to 5 minutes ahead. | `ErrInvalidArgument` |
| Each certificate of the chain signed the one before it. | `ErrInvalidArgument` |
| The leaf names the domain of the install now, and its wildcard. A change of the domain during an order fails this check. | `ErrFailedPrecondition` |
| The lease of the leader is still live. | `lease.ErrStale` |

The same transaction resets `acme_status`, publishes the change event `certificate.changed`, and writes the audit entry `acme.store_certificate`. The entry has the actor `system`, the target kind `certificate`, and the serial as the target ID. Its details are `domain`, `issuer`, `directory_url`, and `not_after`. A new account writes the audit entry `acme.store_account`, with the target kind `acme_account` and the account URL as the target ID. Its details are `directory_url` and `email`. A new terms URL of the CA writes the audit entry `acme.terms_changed`, without a target, with the details `terms_url` and `old_terms_url`. A `userActionRequired` of the CA writes the audit entry `acme.clear_agreement`, without a target, with the detail `cause`.

The edge role serves the certificate ([the edge role](/reference/edge/index.html.md#tls)). The control role pushes it on the control channel to each server with the edge role: after the first heartbeat of each channel, and after each `certificate.changed` ([the control channel](/reference/control-channel/index.html.md#edge-pushes)). So a renewal reaches every edge server without a restart. The control role reads the certificate with these Go functions of the service layer:

| Function | Description |
| --- | --- |
| `Service.InstallCertificate(ctx) (InstallCertificate, error)` | Returns the certificate with its key open. Before the first certificate, it fails with `ErrNotFound`. After a change of the domain, the certificate covers the old domain until the job replaces it. Then it fails with `ErrFailedPrecondition`, so the edge role never serves a certificate for another domain. `InstallCertificate.TLSCertificate()` returns a `tls.Certificate` with the chain, the key, and the parsed leaf. |
| `Service.EdgeCertificate(ctx, caller) (InstallCertificate, error)` | `InstallCertificate` for a server with the edge role, under the policy action `server.read_edge_certificate`. It fails as `InstallCertificate` does. |

After a change of the domain, the edge stops serving the old certificate at once, because the routes name the new domain. It refuses each handshake until the certificate of the new domain arrives ([the edge role](/reference/edge/index.html.md#tls)).

## Logs

The job writes these records with `component=acme`. The records of acmez, such as `trying to solve challenge` and `authorization finalized`, also have `component=acme`.

| Level | Message | Fields | When |
| --- | --- | --- | --- |
| info | `the install gets no certificate yet` | `reason` | The install has no domain, or no agreement to the terms. Once for each change of the reason. |
| info | `the next attempt for the certificate is planned` | `reason`, `at` | After a certificate, or after a failure. Once for each change of the plan. |
| info | `getting the certificate of the install` | `domain`, `directory_url`, `reason` | An attempt starts. |
| info | `got the certificate of the install` | `domain`, `directory_url`, `serial`, `issuer`, `not_after`, `renew_at` | An attempt stored a new certificate. |
| error | `getting the certificate of the install failed. The certificate before it stays` | `domain`, `directory_url`, `err`, `failures`, `retry_at` | An attempt failed. |
| error | `getting the certificate of the install failed, and its record failed too` | `err`, `record_err` | An attempt failed, and the write of the failure failed. The job waits for the backoff anyway. |
| warn | `the ACME directory names no CAA identity, so the zone has no CAA record, and any CA can issue` | | The directory has no `meta.caaIdentities`. |
| warn | `the CA does not know the ACME account of the install any more, so the job makes a new one` | `account_url`, `err` | The CA refused the stored account. |
| warn | `the ACME account of the install is not valid, so the job makes a new one` | `account_url`, `status` | The CA deactivated or revoked the account. |
| warn | `the CA published new terms of service. The install keeps its agreement, and an instance admin should read them` | `terms_url`, `old_terms_url` | The directory names a new terms URL. This record has `component=service`. |
| error | `the CA asks an instance admin to act, so the install asks for no certificate until an instance admin agrees again` | `cause` | The CA answered `userActionRequired`. This record has `component=service`. |
| error | `clearing the agreement after a request of the CA failed` | `err` | The write that clears the agreement failed. |
| info, warn, or error | `the certificate of the install is not renewed` | `reason`, `renewals_stopped`, `not_after`, `next_attempt` | At each plan of the job, while the install has a certificate and the job does not renew it: its renewals stopped, or it is due for renewal and the last attempt failed. `reason` is the reason of the stop, or the last error. The level is error within 14 days of the expiry. Else it is info while an instance admin turned the agreement off and the certificate is not yet due, and warn in the other cases. |
| warn | `reading the state of the certificate failed` | `err`, `retry_in` | The read of the control store failed. The job tries again in 1 minute. |
| info | `the leader lease is stale, so the job stops until the next term` | | A fenced write found the lease stale. |

The service layer writes these records with `component=service`: `stored the ACME account`, `stored the certificate of the install`, `changed the CAA issuers of the zone`, and the records of the TXT records ([the DNS role](/reference/dns/index.html.md#txt-records)).

Example:

```json
{"level":"INFO","msg":"got the certificate of the install","component":"acme","domain":"example.com","directory_url":"https://localhost:14000/dir","serial":"4553a65221ea631b","issuer":"CN=Pebble Intermediate CA 512ad8","not_after":"2027-01-18T22:58:38Z","renew_at":"2026-12-19T22:58:38.333333333Z"}
```

## Test with Pebble

[Pebble](https://github.com/letsencrypt/pebble) is a small ACME CA of Let's Encrypt for tests. Its certificates are not trusted anywhere. The tests run Pebble in process, with the DNS role of the test answering its challenges:

| Test | Package | What it checks |
| --- | --- | --- |
| `TestIssueAndRenew` | `internal/acme` | The chain of the first certificate leads to the root of Pebble, for the domain and for a machine name. The CAA record follows the directory. The challenges leave the zone. A renewal gets a new certificate with the same account. |
| `TestFailedRenewalKeepsTheCertificate` | `internal/acme` | A name server that never answers makes the renewal fail. The certificate stays, and the failure has its error and a later attempt. |
| `TestStaleLeaseIssuesNothing` | `internal/acme` | A job with a stale lease stops at once, with no attempt and no account. |
| `TestNewTermsKeepTheAgreement` | `internal/acme` | A new terms URL in the directory keeps the agreement. The job gets the certificate and records the new URL. |
| `TestUserActionRequiredClearsTheAgreement` | `internal/acme` | A CA that answers `userActionRequired` makes the job clear the agreement and record the failure. |
| `TestLeaseLostAfterPresent` | `internal/acme` | The lease ends after the first challenge is in the zone. The job stops, and stores no certificate. The next term removes the challenges that stayed, and gets the certificate. |
| `TestNoAttemptWithoutTheAgreement` | `internal/acme` | Without the agreement, the job makes no attempt. After the agreement, it gets the certificate. |
| `TestWildcardCertificate` | `internal/control` | `hopboxd` with the control role and the DNS role gets the certificate. The zone reaches the DNS role over the control channel. |

[Get the wildcard certificate of an install](/guide/acme-certificate/index.html.md) runs the job against a Pebble binary on a Mac.
