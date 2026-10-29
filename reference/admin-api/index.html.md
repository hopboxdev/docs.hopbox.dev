# The admin API

> The admin API hopbox.admin.v1, its calls and their audit entries, including enrollment tokens, the server calls, the calls on orgs and their limits, and admin API keys, and the admin socket.

The admin API is the API service `hopbox.admin.v1`. Instance admins and root on a control server use it to operate the whole install. The hopbox CLI and the SDKs never include it. This page lists its calls and the admin socket. [The API server](/reference/api/index.html.md) covers the parts that every call shares: the request gate, the policy check, the idempotency keys, and the error codes. The generated reference is [Admin API](/api/hopbox.admin.v1.openapi.yaml).

## Where the admin API is served

| Listener | Callers that the policy allows | Use |
| --- | --- | --- |
| The admin socket, `<run_dir>/admin.sock` | `local_root` | `hopboxd admin` on the server ([hopboxd](/reference/hopboxd/index.html.md#admin)) |
| The API listener, `control.api_listen` | `admin_key`, except for the calls of `KeyringService`, `MailService`, `LoginLinkService`, and `AdminKeyService`. An [admin session](#admin-sessions), for the calls of the Admin area. | The hopbox Terraform provider and scripts, with an admin API key in the header `Authorization: Bearer hbxadm_...` ([CLI tokens, API keys, admin API keys, and SSH keys](/reference/credentials/index.html.md#admin-api-keys)). The Admin area of the console, with the session cookie ([Console](/reference/console/index.html.md#admin-area)). |
| The domain of the install, through the edge servers, which forward it to the internal listener ([the edge role](/reference/edge/index.html.md#forwarded-requests)) | as on the API listener | The same callers from outside the server, such as `https://example.com/hopbox.admin.v1.ServerService/ListServers`. The admin API is public behind the edge, for admin API keys and admin sessions. |

The admin API has these services:

| Service | Calls |
| --- | --- |
| `hopbox.admin.v1.InstallConfigService` | `GetInstallConfig`, `UpdateInstallConfig`, `GetSMTPConfig`, `SetSMTPConfig`, `ClearSMTPConfig`, `GetOAuthConfig`, `SetOAuthClient`, `ClearOAuthClient`, `SetMaintenance` |
| `hopbox.admin.v1.KeyringService` | `RotateKEK`, `StageKEK`, `PromoteKEK`, `ListKEKs`, `RewrapSealedValues` |
| `hopbox.admin.v1.MailService` | `ListMailFiles`, `GetMailFile` |
| `hopbox.admin.v1.LoginLinkService` | `CreateLoginLink` |
| `hopbox.admin.v1.InstanceAdminService` | `ListInstanceAdmins`, `GrantInstanceAdmin`, `RevokeInstanceAdmin`, `ListAdminNotices`, `DismissAdminNotices` |
| `hopbox.admin.v1.AccountService` | `ListAccountSignInMethods`, `RemoveAccountSignInMethod` |
| `hopbox.admin.v1.ServerService` | `CreateEnrollmentToken`, `ListServers`, `GetServer`, `CordonServer`, `UncordonServer`, `SetServerLabels`, `DecommissionServer`, `DrainServer`, `ListControlProcesses` |
| `hopbox.admin.v1.PoolService` | `CreatePool`, `GetPool`, `ListPools`, `DeletePool` |
| `hopbox.admin.v1.SizeService` | `CreateSize`, `GetSize`, `ListSizes`, `RetireSize` |
| `hopbox.admin.v1.OrgService` | `ListOrgs`, `SuspendOrg`, `UnsuspendOrg`, `SetOrgLimits`, `GetDefaultOrgLimits`, `SetDefaultOrgLimits` |
| `hopbox.admin.v1.AdminKeyService` | `CreateAdminKey`, `RegisterAdminKey`, `ListAdminKeys`, `RevokeAdminKey` |
| `hopbox.admin.v1.MachineService` | `ListMachines`, `StopMachine`, `DeleteMachine`, `MoveMachine`, `CancelMove`, `ListMoves` |
| `hopbox.admin.v1.ImageService` | `RegisterImage`, `ListImages` |

### Admin sessions

An admin session is the console session of an instance admin whose sign-in is at most 10 minutes old ([Sessions, cookies, and sign-in links](/reference/sessions/index.html.md#recent-sign-in)). The Admin area of the console calls the admin API with it ([Console](/reference/console/index.html.md#admin-area)). The cookie rules of the public API hold for these calls too. Each call must carry the `Origin` header of the install and the header `Connect-Protocol-Version: 1` ([Sessions, cookies, and sign-in links](/reference/sessions/index.html.md#calls-with-the-cookie)). So a page of another site cannot make an admin call with the cookie of a browser.

An admin session can make these calls, and no other call of the admin API:

| Service | Calls |
| --- | --- |
| `hopbox.admin.v1.InstallConfigService` | `GetInstallConfig`, `UpdateInstallConfig`, `GetSMTPConfig`, `SetSMTPConfig`, `ClearSMTPConfig`, `GetOAuthConfig`, `SetOAuthClient`, `ClearOAuthClient` |
| `hopbox.admin.v1.InstanceAdminService` | `ListInstanceAdmins`, `GrantInstanceAdmin`, `RevokeInstanceAdmin`, `ListAdminNotices`, `DismissAdminNotices` |
| `hopbox.admin.v1.AdminKeyService` | `CreateAdminKey`, `ListAdminKeys`, `RevokeAdminKey` |
| `hopbox.admin.v1.ServerService` | `ListServers`, `GetServer`, `CordonServer`, `UncordonServer`, `SetServerLabels`, `DrainServer` (only of a server that hosts no control process) |
| `hopbox.admin.v1.PoolService` | `CreatePool`, `GetPool`, `ListPools`, `DeletePool` |
| `hopbox.admin.v1.SizeService` | `CreateSize`, `GetSize`, `ListSizes`, `RetireSize` |
| `hopbox.admin.v1.OrgService` | `ListOrgs`, `SuspendOrg`, `UnsuspendOrg`, `SetOrgLimits`, `GetDefaultOrgLimits`, `SetDefaultOrgLimits` |
| `hopbox.admin.v1.MachineService` | `ListMachines`, `StopMachine`, `DeleteMachine` |
| `hopbox.admin.v1.ImageService` | `ListImages` |

The Admin area also reads the audit log of the whole install with `ListAuditEntries` of the public API, under the same rule. The same rule holds for the audit log of an org in which the instance admin is not an owner or an admin ([the sign-in, account, and audit API](/reference/auth-api/index.html.md#listauditentries)).

The other calls hand out a secret, or change what only root on a control server or the hopbox Terraform provider changes:

| Calls | Reason |
| --- | --- |
| `SetMaintenance`, `CreateLoginLink`, `RegisterAdminKey`, and the calls of `KeyringService` and `MailService` | Only root on a control server makes them ([The admin socket](#the-admin-socket)). |
| `CreateEnrollmentToken` and `DecommissionServer` | The hopbox Terraform provider adds and removes servers. An enrollment token is a secret. |
| `RegisterImage`, `GetCertificate`, `ListControlProcesses`, and the calls of `AccountService` | No page of the Admin area uses them. They stay with root on a control server and with admin API keys. |

The policy check of the service layer decides each call, the reads too ([the API server](/reference/api/index.html.md#callers-and-policy)). The console only hides the pages of the Admin area from other accounts. It is not the check.

| Caller | Result |
| --- | --- |
| The session of an instance admin, at most 10 minutes after its sign-in, on a call of the table | The call runs. Its audit entry has the actor kind `session`, the account, and the session. |
| The session of an instance admin, more than 10 minutes after its sign-in, on a call of the table | `permission_denied` with the message `permission denied: this action needs a sign-in within the last 10 minutes. Sign in again, then retry`, and the detail `hopbox.v1.RecentSignInRequired` ([the sign-in, account, and audit API](/reference/auth-api/index.html.md#recentsigninrequired)). The call writes no audit entry. |
| The session of an instance admin, on any other call | `permission_denied` without the detail. A new sign-in does not change it. |
| The session of an account that is not an instance admin | `permission_denied` without the detail, on every call. |
| The CLI token of an instance admin | `permission_denied`. The hopbox CLI never calls the admin API. |
| A session cookie without the `Origin` of the install, or without the Connect protocol | `unauthenticated` with the message `the credential is not valid` ([Sessions, cookies, and sign-in links](/reference/sessions/index.html.md#calls-with-the-cookie)) |

The gate reads the instance admin role and the age of the sign-in with the session on each call. So a revoke of the role closes the admin API to the sessions of the account at once.

The internal listener does not serve the admin API ([the internal listener](/reference/internal-listener/index.html.md)).

In maintenance mode, the admin API takes only `SetMaintenance` ([the API server](/reference/api/index.html.md#maintenance-mode)). Each other call, also each `Get` and `List` call of this page, writes an audit entry, so maintenance mode refuses it with `unavailable` before it changes anything. The reads keep the option `(hopbox.v1.read_only)` all the same, because a retry of them does no harm ([API definitions](/reference/api-definitions/index.html.md#hopboxv1read_only)).

Every call of the admin API writes an audit entry in the transaction of the call, also a call that only reads. The entry has the action of the call, the actor kind of the caller, and the request ID. An entry of `local_root` also has the details `peer_uid` and `peer_pid`: the user and the process that called through the socket. A refused call writes no entry.

## The admin socket

The admin socket is a unix socket that only root can use. With it, whoever controls a control server can always act, even without web access. The code is in `internal/adminsock`.

| Item | Value |
| --- | --- |
| Path | `<run_dir>/admin.sock`, by default `/run/hopbox/admin.sock` ([the bootstrap config of hopboxd](/reference/hopboxd-config/index.html.md#keys)) |
| Owner | the user of `hopboxd`, root on a server |
| Mode | `0600` |
| Protocols | Connect, gRPC, and gRPC-Web, over HTTP/1.1 or HTTP/2 with prior knowledge |
| Services | every service of the admin API, gRPC reflection, and gRPC health |
| Caller | `local_root` |

The socket checks each connection in two ways:

1. **File mode.** The socket file has mode `0600`, so only its owner can connect. `hopboxd` makes the socket under a temp name, `admin.sock.<6 random characters>.tmp`, sets the mode, and then renames it to `admin.sock`. So the final name never has a looser mode.
2. **Peer check.** For each new connection, `hopboxd` asks the kernel for the user and the process of the peer: `SO_PEERCRED` on Linux, `LOCAL_PEERCRED` and `LOCAL_PEERPID` on macOS. It accepts only root (uid 0) and the user that runs `hopboxd`. It closes any other connection at once and logs `admin socket refused a peer`. On a server, `hopboxd` runs as root, so only root is accepted. On a laptop, `hopboxd` runs as you, so you can call it without `sudo`.

The second check holds also after a change of the mode of the file. The request gate of the socket turns the peer into the caller `local_root`, with its user ID and its process ID. Then the policy check and the audit log see that caller ([the API server](/reference/api/index.html.md#callers-and-policy)).

When it starts, `hopboxd` does these checks on the socket:

| Check | Error |
| --- | --- |
| The socket dir must not be a symlink. | `adminsock: the socket dir <dir> is a symlink` |
| The socket dir must be a dir. | `adminsock: the socket dir <dir> is not a dir` |
| Group and others must not be able to write in the socket dir. Such a user can swap the socket for one of their own. | `adminsock: the socket dir <dir> has mode <mode>, which lets group or others change it` |
| The socket dir must belong to the user of `hopboxd`. | `adminsock: the socket dir <dir> must belong to the user of hopboxd` |
| A file at the path must be a socket. | `adminsock: <path> exists and is not a socket, so hopboxd will not replace it` |
| Another process must not serve a socket at the path. A socket that no process serves, for example after a crash, is removed. | `adminsock: another process serves <path>. Stop it first` |
| The path plus its 11-byte temp suffix must fit in the 103 bytes of a unix socket path. | `adminsock: the socket path <path> has <n> bytes, and with its temp suffix it must fit in 92` |

A missing socket dir is created with mode `0755`. When `hopboxd` stops, it removes the socket file, if the file is still its own socket.

### Logs of the socket

The socket writes these records with `component=adminsock`:

| Message | Level | Fields |
| --- | --- | --- |
| `admin socket refused a peer` | warn | `uid`, `pid` |
| `admin socket failed to read the peer` | warn | `err` |

Each call through the socket also gets the access log record `api call` with `caller=local_root` ([the API server](/reference/api/index.html.md#logs)). The HTTP server of the socket writes its own errors at level warn with `component=http`, as the API listener does. These records have no `component=adminsock`.

### Call the socket by hand

`hopboxd admin` is the client of the socket. Other gRPC or Connect clients work too. For example, with curl and the Connect protocol:

```sh
sudo curl -s --unix-socket /run/hopbox/admin.sock \
  -H 'Content-Type: application/json' -d '{}' \
  http://hopboxd/hopbox.admin.v1.InstallConfigService/GetInstallConfig
```

## InstallConfigService

`hopbox.admin.v1.InstallConfigService` reads and changes the config of the whole install. The config lives in the table `install_config` ([Control store tables](/reference/control-store/index.html.md#install_config)).

The message `InstallConfig` holds the general config:

| Field | Type | Description |
| --- | --- | --- |
| `domain` | string | The domain of the install, for example `example.com`. It is empty until an instance admin sets it. |
| `version` | int64 | The version of the install config. It starts at 1 and grows by 1 with each change. |
| `update_time` | `google.protobuf.Timestamp` | The time of the last change. |
| `public_addresses` | repeated string | The IPv4 and IPv6 addresses at which the edge servers take traffic, in their order. The DNS role points the domain, `ns1`, `ns2`, and every machine name at them ([the DNS role](/reference/dns/index.html.md#the-zone-of-the-install)). It is empty until an instance admin sets it. |
| `acme_directory_url` | string | The ACME directory of the CA that issues the wildcard certificate of the install ([the certificate of the install](/reference/acme/index.html.md#settings)). The default is `https://acme-v02.api.letsencrypt.org/directory`, the production CA of Let's Encrypt. |
| `acme_email` | string | The contact email of the ACME account of the install. For an account without a contact, it is empty. |
| `acme_terms_agreed` | bool | `true` once an instance admin agreed to the terms of the CA of `acme_directory_url`. The install asks for no certificate before that. The default is `false`. |
| `mail_transport` | `MailTransport` | How the install delivers its mail ([Mail](/reference/mail/index.html.md#mail-transports)). The default is `MAIL_TRANSPORT_SMTP`. |
| `maintenance` | bool | `true` while the install is in maintenance mode ([the API server](/reference/api/index.html.md#maintenance-mode)). Only [SetMaintenance](#setmaintenance) changes it. The default is `false`. |

The enum `MailTransport` has these values:

| Value | Number | Meaning |
| --- | --- | --- |
| `MAIL_TRANSPORT_UNSPECIFIED` | 0 | Never in a response. A request must not use it. |
| `MAIL_TRANSPORT_SMTP` | 1 | Sends each mail through the SMTP server of the install ([SMTP config](/reference/smtp/index.html.md)). |
| `MAIL_TRANSPORT_FILE` | 2 | Writes each mail to a file on the control server that sends it, and sends nothing. It is for a dev or test install. `MailService` reads the files ([MailService](#mailservice)). |

### GetInstallConfig

Returns the general config of the install.

| Item | Value |
| --- | --- |
| Procedure | `/hopbox.admin.v1.InstallConfigService/GetInstallConfig` |
| Policy action | `install_config.get`: `local_root`, `admin_key`, and an [admin session](#admin-sessions) |
| Idempotent option | no |
| Audit entry | action `install_config.get`, no target |

The request `GetInstallConfigRequest` has no fields. The response `GetInstallConfigResponse` has the field `config`, an `InstallConfig`.

### UpdateInstallConfig

Changes the fields that the request sets, and keeps the others. It returns the config after the change.

| Item | Value |
| --- | --- |
| Procedure | `/hopbox.admin.v1.InstallConfigService/UpdateInstallConfig` |
| Policy action | `install_config.update`: `local_root`, `admin_key`, and an [admin session](#admin-sessions) |
| Idempotent option | no. A second call with the same request sets the same values again. |
| Audit entry | action `install_config.update`, no target, details `domain` (the new domain), `old_domain`, `public_addresses` (the new addresses, joined by commas), and `old_public_addresses`. Also `acme_directory_url`, `acme_email`, `acme_terms_agreed` (`true` or `false`), and `mail_transport` (`smtp` or `file`), each with its `old_` detail. |

The request `UpdateInstallConfigRequest` has these fields:

| Field | Type | Description |
| --- | --- | --- |
| `domain` | optional string | The new domain. It must follow the rules in [Domain rules](/reference/domain-rules/index.html.md#install-domain). Leave the field out to keep the domain. |
| `public_addresses` | `PublicAddresses` | The new public addresses. The list replaces every public address of the install. An empty list removes them. Leave the field out to keep them. |
| `acme_directory_url` | optional string | The new ACME directory: an `https` URL of a host, with at most 2048 characters, and without user info or a fragment. A new directory also sets `acme_terms_agreed` to `false`, unless the request sets `acme_terms_agreed` too. Leave the field out to keep the directory. |
| `acme_email` | optional string | The new contact email of the ACME account. It must be one plain email address, such as `ops@example.org`, and the install keeps it in lowercase. An empty string removes the contact. Leave the field out to keep the email. |
| `acme_terms_agreed` | optional bool | `true` records that the caller agrees to the terms of the CA of the directory. A new agreement covers the terms that the next attempt reads from the directory. `false` stops new certificates. Leave the field out to keep it. |
| `mail_transport` | optional `MailTransport` | The new mail transport. The SMTP config stays, so a switch back to `MAIL_TRANSPORT_SMTP` needs no new config. Leave the field out to keep the transport. |
| `expected_version` | optional int64 | The `version` of the install config that the caller read. If the config has another version now, the call fails with `aborted` and changes nothing. Leave the field out to change the config at any version. |

The message `PublicAddresses` has one field, `addresses`, a repeated string. Each address must follow the rules in [Domain rules](/reference/domain-rules/index.html.md#public-addresses): at most 16 IPv4 or IPv6 addresses, none twice. The install keeps them in the text form of Go `netip`, in their order. So `2001:DB8::10` becomes `2001:db8::10`, and an IPv4 address in IPv6 form, such as `::ffff:203.0.113.10`, becomes `203.0.113.10`.

With `expected_version`, two admins that change the config at the same time cannot overwrite each other: the second one gets `aborted` with the message `conflict: the install config is at version <n>, not <m>. Read it again`. The call locks the row until it commits, so the version check and the change happen as one step.

A request that sets no field gets `invalid_argument` with the message `invalid argument: the update sets no setting`. `expected_version` alone sets no field. A bad domain gets `invalid_argument` with the rule that it breaks, for example `invalid argument: invalid name: the install domain "localhost" must have at least two labels, such as hop.example.org`. A bad address gets `invalid_argument` with the rule that it breaks, for example `invalid argument: invalid public address: "0.0.0.0" is not an address that a client on another host can reach`. A bad ACME setting gets `invalid_argument` too, for example `invalid argument: invalid ACME setting: "http://acme.example.org/directory" is not an https URL of a host without user info or a fragment, such as https://acme-v02.api.letsencrypt.org/directory`. `MAIL_TRANSPORT_UNSPECIFIED` or an unknown transport gets `invalid_argument` with the message `invalid argument: the mail transport <n> is not smtp or file`.

While the transport is `MAIL_TRANSPORT_FILE`, each update logs `mail goes to files, not to recipients` at level warn ([Mail](/reference/mail/index.html.md#logs)).

A change of an input of the ACME job publishes the change event `acme.config_changed` in the transaction of the change ([the certificate of the install](/reference/acme/index.html.md#settings)). It also resets the failures of the ACME job, so the job tries again at once, unless the CA asked for the wait ([the certificate of the install](/reference/acme/index.html.md#renewal-and-backoff)).

The response `UpdateInstallConfigResponse` has the field `config`, an `InstallConfig`.

`GetInstallInfo` returns the new domain to every caller at once ([the API server](/reference/api/index.html.md#getinstallinfo)).

### SetMaintenance

Turns maintenance mode of the install on or off, and returns the config after the change ([the API server](/reference/api/index.html.md#maintenance-mode)). `hopboxd admin maintenance on` and `off` call it ([hopboxd](/reference/hopboxd/index.html.md#admin-maintenance-on)).

| Item | Value |
| --- | --- |
| Procedure | `/hopbox.admin.v1.InstallConfigService/SetMaintenance` |
| Policy action | `install_config.set_maintenance`: `local_root` only. An admin API key could not turn maintenance mode off again, because maintenance mode refuses every write. |
| Idempotent option | no. A call for the state that the install has already changes nothing, and succeeds. |
| Audit entry | action `install_config.set_maintenance`, no target, detail `maintenance` (`true` or `false`). A call that changes nothing writes no entry. |

The request `SetMaintenanceRequest` has one field:

| Field | Type | Description |
| --- | --- | --- |
| `maintenance` | bool | `true` turns maintenance mode on, and `false` turns it off. |

The response `SetMaintenanceResponse` has the field `config`, an `InstallConfig`.

A change sets the column `maintenance` of `install_config`, adds 1 to the version, and publishes the change event `install.maintenance_changed` in one transaction ([Control store tables](/reference/control-store/index.html.md#install_config)). Then the control process turns its own maintenance mode on or off. Maintenance mode lets this call through also while it is on, so the admin socket always takes it.

A call that turns maintenance mode on, also one for the state that the install has already, returns only after the calls, the control channels, and the leader jobs that ran in this process ended. If they do not end before the deadline of the call, it fails with `unavailable` and the message `unavailable: <n> calls, control channels, or leader jobs still run: context deadline exceeded`, and maintenance mode stays on. Call it again. If maintenance mode turns off before the wait ends, the call fails with `unavailable` and the message `unavailable: maintenance mode turned off while it waited for the calls, control channels, and leader jobs to end`, because the work runs again. So a success means that maintenance mode was on and the work had ended.

The turn to on also covers the other control processes of the install, which record themselves in `control_processes` ([Control store tables](/reference/control-store/index.html.md#control_processes)). It runs these steps:

1. Before it changes anything, it checks the live rows. If a live control process knows fewer than 44 migrations, the call fails with `failed_precondition` and the message `the control process <id> on <host> (hopboxd <rel>) knows the migrations up to <n>, and only a hopboxd that knows migration 44 honors maintenance mode. Upgrade it first`. Maintenance mode stays off. Only releases from migration `0045` on write rows, so this check guards the rows of a future release, not an older one.
2. It drains its own process, as the paragraph above says.
3. It waits until every other live control process reports maintenance mode on and drained, with a `maintenance_version` at least the version of `install_config` that the turn wrote. It reads the rows every 250 ms. Each process reports in its row the version at which it read maintenance mode, so the wait compares versions, not times. A process that applies the change renews its row at once, and once more at the end of its drain. A process that ended without its stop counts as gone only 35 s after its last renewal (`store.ControlProcessGone`: the live window of 20 s and the dead peer timeout of 15 s), because a COMMIT that it sent before its fence can still land until the store ends its session. A process that stopped cleanly counts as gone at once. So a turn to on beside a control server that died can wait up to 35 s, longer than the deadline of the call: then it fails with `unavailable`, and you call it again.

If a process does not report before the deadline of the call, the call fails with `unavailable` and the message `maintenance mode is on, but the control process <id> on <host> (hopboxd <rel>) did not report that its calls, control channels, and leader jobs ended: ...`. The rest of the message is the cause, such as `context deadline exceeded`. The flag stays on then. Call it again. A control process of a release before migration `0045` writes no row, so the call does not see it, and neither does any check that reads the rows: the migration guard, the KEK promote, the decommission check, and the rolling upgrade order. Upgrade every control server before you rely on them.

The control process logs `maintenance mode is on: the API takes only reads, and the control role serves no control channel` at level warn after a change to on, and `maintenance mode is off` at level info after a change to off, with `component=service`.

A change of the domain or of the public addresses raises the serial of the DNS zones ([Control store tables](/reference/control-store/index.html.md#dns_zone)). The same transaction publishes the change event `dns.zone_changed`. The control role then pushes the new zones to each server with the DNS role ([the DNS role](/reference/dns/index.html.md#how-the-zones-reach-the-dns-role)). An update that sets the same domain and the same addresses again changes neither, and pushes nothing.

### GetSMTPConfig

Returns the SMTP config of the install, without its password ([SMTP config](/reference/smtp/index.html.md)).

| Item | Value |
| --- | --- |
| Procedure | `/hopbox.admin.v1.InstallConfigService/GetSMTPConfig` |
| Policy action | `smtp_config.get`: `local_root`, `admin_key`, and an [admin session](#admin-sessions) |
| Idempotent option | no |
| Audit entry | action `smtp_config.get`, no target |

The request `GetSMTPConfigRequest` has no fields. The response `GetSMTPConfigResponse` has the field `config`, an `SMTPConfig`:

| Field | Type | Description |
| --- | --- | --- |
| `host` | string | The host name or the IP address of the SMTP server. An install without an SMTP config has an empty host. |
| `port` | int32 | The TCP port. |
| `tls` | `SMTPSecurity` | `SMTP_SECURITY_STARTTLS`, `SMTP_SECURITY_TLS`, or `SMTP_SECURITY_NONE`. `SMTP_SECURITY_UNSPECIFIED` for an install without an SMTP config. |
| `username` | string | The SMTP login, or empty. |
| `password_set` | bool | `true` for a config with a password. The API never returns the password. |
| `from` | string | The sender of the mail. |
| `version` | int64 | The version of the whole install config ([GetInstallConfig](#getinstallconfig)). |
| `update_time` | `google.protobuf.Timestamp` | The time of the last change of the install config. |
| `mail_transport` | `MailTransport` | The mail transport of the install now. With `MAIL_TRANSPORT_FILE`, the install does not use this SMTP server ([Mail](/reference/mail/index.html.md#mail-transports)). |

### SetSMTPConfig

Replaces the SMTP config, and returns it. The password is sealed with the current key-encryption key ([SMTP config](/reference/smtp/index.html.md#password)). `GetInstallInfo` then lists `SIGN_IN_METHOD_EMAIL_LINK`.

| Item | Value |
| --- | --- |
| Procedure | `/hopbox.admin.v1.InstallConfigService/SetSMTPConfig` |
| Policy action | `smtp_config.set`: `local_root`, `admin_key`, and an [admin session](#admin-sessions) |
| Idempotent option | no. A second call with the same request sets the same values again. |
| Audit entry | action `smtp_config.set`, no target. The details are `host`, `port`, `tls`, `username`, `from`, `old_host`, and `password_changed` (`true` or `false`). They never hold the password. |

The request `SetSMTPConfigRequest` has these fields:

| Field | Type | Default | Description |
| --- | --- | --- | --- |
| `host` | string | none, required | The host name or the IP address of the server, without a port. |
| `port` | int32 | the usual port of `tls`: 587, 465, or 25 | The TCP port, 1 to 65535. |
| `tls` | `SMTPSecurity` | `SMTP_SECURITY_STARTTLS` | How the connection is encrypted ([SMTP config](/reference/smtp/index.html.md#tls-modes)). |
| `username` | string | empty | The SMTP login. |
| `password` | optional string | the password that is set now | The SMTP password. Leave it out to keep the password that is set now. An empty password removes it. A new `host` or `username` needs the password again. |
| `from` | string | none, required | The sender, such as `hopbox <no-reply@example.org>`. |
| `expected_version` | optional int64 | none | The `version` of the install config that the caller read. If the install config has another version now, the call fails with `aborted` and changes nothing. |

The response `SetSMTPConfigResponse` has the field `config`, an `SMTPConfig`. A field that breaks a rule gets `invalid_argument` with the rule ([SMTP config](/reference/smtp/index.html.md#errors)). A new password without a key-encryption key gets `failed_precondition`.

### ClearSMTPConfig

Removes the SMTP config and its password, so the install sends no more sign-in links.

| Item | Value |
| --- | --- |
| Procedure | `/hopbox.admin.v1.InstallConfigService/ClearSMTPConfig` |
| Policy action | `smtp_config.clear`: `local_root`, `admin_key`, and an [admin session](#admin-sessions) |
| Idempotent option | no |
| Audit entry | action `smtp_config.clear`, no target, details `old_host` |

The request `ClearSMTPConfigRequest` has no fields. The response `ClearSMTPConfigResponse` has the field `config`, the empty `SMTPConfig` with the new version.

### OAuthClient

The message `OAuthClient` holds the client of one sign-in provider, without its secret ([Sign-in with GitHub and Google](/reference/oauth/index.html.md#config)). The enum `SignInProvider` names the provider:

| Value | Provider |
| --- | --- |
| `SIGN_IN_PROVIDER_UNSPECIFIED` | Never in a response. A request with it fails with `invalid_argument`. |
| `SIGN_IN_PROVIDER_GITHUB` | A GitHub OAuth app. |
| `SIGN_IN_PROVIDER_GOOGLE` | A Google OAuth client for a web application, through OpenID Connect. |

| Field | Type | Description |
| --- | --- | --- |
| `provider` | `SignInProvider` | The provider. |
| `client_id` | string | The client ID that the provider gave the operator. It is empty while the provider is off. |
| `client_secret_set` | bool | `true` for a client with a secret. The API never returns the secret. |
| `redirect_uri` | string | The URI to register at the provider, such as `https://example.com/auth/github/callback` ([Sign-in with GitHub and Google](/reference/oauth/index.html.md#redirect-uris)). It is empty until the install has a domain. |
| `enabled` | bool | `true` for a provider that people can sign in with: the client is set, and the install has a domain. |

### GetOAuthConfig

Returns the client of each provider, without its secret, and the redirect URI to register at each provider.

| Item | Value |
| --- | --- |
| Procedure | `/hopbox.admin.v1.InstallConfigService/GetOAuthConfig` |
| Policy action | `oauth_config.get`: `local_root`, `admin_key`, and an [admin session](#admin-sessions) |
| Idempotent option | no |
| Audit entry | action `oauth_config.get`, no target |

The request `GetOAuthConfigRequest` has no fields. The response `GetOAuthConfigResponse` has these fields:

| Field | Type | Description |
| --- | --- | --- |
| `clients` | repeated `OAuthClient` | One client for each provider, GitHub first. |
| `version` | int64 | The version of the whole install config ([GetInstallConfig](#getinstallconfig)). |
| `update_time` | `google.protobuf.Timestamp` | The time of the last change of the install config. |

### SetOAuthClient

Sets the client of one provider, and returns it. The secret is sealed with the current key-encryption key ([Sealing](/reference/sealing/index.html.md#places)). With a domain, `GetInstallInfo` then lists `SIGN_IN_METHOD_GITHUB` or `SIGN_IN_METHOD_GOOGLE`.

| Item | Value |
| --- | --- |
| Procedure | `/hopbox.admin.v1.InstallConfigService/SetOAuthClient` |
| Policy action | `oauth_config.set`: `local_root`, `admin_key`, and an [admin session](#admin-sessions) |
| Idempotent option | no. A second call with the same request sets the same values again. |
| Audit entry | action `oauth_config.set`, no target. The details are `provider` (`github` or `google`), `client_id`, `old_client_id`, and `secret_changed` (`true` or `false`). They never hold the secret. |

The request `SetOAuthClientRequest` has these fields:

| Field | Type | Default | Description |
| --- | --- | --- | --- |
| `provider` | `SignInProvider` | none, required | The provider. |
| `client_id` | string | none, required | The client ID: 1 to 256 printable ASCII characters, without spaces. |
| `client_secret` | optional string | the secret that is set now | The client secret: 1 to 512 printable ASCII characters, without spaces. Leave it out to keep the secret that is set now. That works only for the same `client_id`. A new client ID needs its secret. |
| `expected_version` | optional int64 | none | The `version` of the install config that the caller read. If the install config has another version now, the call fails with `aborted` and changes nothing. |

The response `SetOAuthClientResponse` has the field `client`, an `OAuthClient`, and the new `version` and `update_time` of the install config. These errors can occur:

| Code | Message | Cause |
| --- | --- | --- |
| `invalid_argument` | `invalid argument: "SIGN_IN_PROVIDER_UNSPECIFIED" is not a sign-in provider. Use github or google` | The provider is unspecified or unknown. |
| `invalid_argument` | `invalid argument: the client ID is empty` | No client ID. |
| `invalid_argument` | `invalid argument: the client secret must hold only printable ASCII characters without spaces` | The client ID or the secret breaks the rule of its characters. |
| `invalid_argument` | `invalid argument: the client ID has more than 256 characters` | The client ID or the secret is too long. |
| `invalid_argument` | `invalid argument: a new client ID needs its client secret` | No secret, and the provider has no client with this client ID. |
| `failed_precondition` | `failed precondition: the keyring has no key-encryption key to seal or open the value. Run hopboxd admin kek rotate to make the first one` | The keyring has no key-encryption key. |
| `aborted` | `conflict: the install config is at version 5, not 4. Read it again` | `expected_version` is old. |

### ClearOAuthClient

Removes the client of one provider and its secret, so nobody can start a sign-in with it. The sign-in methods of the provider stay on their accounts, and work again after a new `SetOAuthClient`.

| Item | Value |
| --- | --- |
| Procedure | `/hopbox.admin.v1.InstallConfigService/ClearOAuthClient` |
| Policy action | `oauth_config.clear`: `local_root`, `admin_key`, and an [admin session](#admin-sessions) |
| Idempotent option | no |
| Audit entry | action `oauth_config.clear`, no target, details `provider` and `old_client_id` |

The request `ClearOAuthClientRequest` has the fields `provider`, a `SignInProvider`, and `expected_version`, as in `SetOAuthClientRequest`. The response `ClearOAuthClientResponse` has the field `client`, the empty `OAuthClient` with its redirect URI, and the new `version` and `update_time`.
## CertificateService

`hopbox.admin.v1.CertificateService` reads the wildcard certificate of the install and the state of the ACME job that gets it ([the certificate of the install](/reference/acme/index.html.md)).

### GetCertificate

Returns the certificate of the install, without its key, and the state of the ACME job.

| Item | Value |
| --- | --- |
| Procedure | `/hopbox.admin.v1.CertificateService/GetCertificate` |
| Policy action | `certificate.get`: `local_root` and `admin_key` |
| Idempotent option | no |
| Audit entry | action `certificate.get`, no target |

The request `GetCertificateRequest` has no fields. The response `GetCertificateResponse` has the field `certificate`, a `Certificate`, and the field `status`, an `ACMEStatus`. Before the first certificate of the install, `certificate` is absent.

The message `Certificate`:

| Field | Type | Description |
| --- | --- | --- |
| `domain` | string | The domain of the install that the certificate covers. |
| `names` | repeated string | The DNS names of the certificate: the domain and its wildcard, such as `example.com` and `*.example.com`. |
| `serial` | string | The serial number, in lowercase hex. |
| `issuer` | string | The subject of the CA certificate that signed the certificate, such as `CN=R12,O=Let's Encrypt,C=US`. |
| `not_before` | `google.protobuf.Timestamp` | The start of the validity. |
| `not_after` | `google.protobuf.Timestamp` | The end of the validity. |
| `renew_time` | `google.protobuf.Timestamp` | The time at which the certificate is due for renewal: after two thirds of its lifetime ([the certificate of the install](/reference/acme/index.html.md#renewal-and-backoff)). |
| `directory_url` | string | The ACME directory that issued the certificate. |
| `update_time` | `google.protobuf.Timestamp` | When the certificate replaced the one before it. |

The message `ACMEStatus`:

| Field | Type | Description |
| --- | --- | --- |
| `failures` | int32 | The attempts that failed since the last certificate, or since the last change of an input of the ACME job. |
| `last_attempt_time` | `google.protobuf.Timestamp` | The start of the last attempt. Absent before the first attempt. |
| `last_error` | string | The error of the last attempt that failed. Empty after a certificate. A change of an input of the ACME job keeps it until the next attempt. |
| `next_attempt_time` | `google.protobuf.Timestamp` | After a failure, the earliest time of the next attempt. Absent after a certificate. |
| `terms_url` | string | The URL of the terms of service of the CA that the directory named at the last attempt ([the certificate of the install](/reference/acme/index.html.md#settings)). The first attempt after an agreement records it, so it is empty before that attempt. |
| `renewals_stopped` | string | Why the ACME job plans no renewal of the certificate: `the install has no domain`, `no instance admin agreed to the terms of the CA`, or, after a `userActionRequired` of the CA, `the CA asked an instance admin to act: read its new terms and agree again`. Empty while the job renews the certificate, and before the first certificate. |
| `ca_wait` | bool | If the CA asked for the wait until `next_attempt_time`, with a rate limit or a header `Retry-After`, it is `true`. A change of an input of the ACME job does not end such a wait, except a change of the directory. The wait lasts at most 7 days. |

## KeyringService

`hopbox.admin.v1.KeyringService` manages the key-encryption keys (KEKs) that seal secrets ([Sealing](/reference/sealing/index.html.md)).

### RotateKEK

Adds a new KEK to the file keyring in `<data_dir>/kek` and makes it the current one. The old KEKs stay, so the values that they sealed still open ([Sealing](/reference/sealing/index.html.md#how-a-rotation-works)). Use it only with one control server. With more than one, use [StageKEK](#stagekek) and [PromoteKEK](#promotekek).

| Item | Value |
| --- | --- |
| Procedure | `/hopbox.admin.v1.KeyringService/RotateKEK` |
| Policy action | `keyring.rotate`: `local_root` only. The key files live on one control server, so only root on that server rotates them. |
| Idempotent option | yes. A call must have an `Idempotency-Key` header ([the API server](/reference/api/index.html.md#idempotency-keys)). |
| Audit entries | `keyring.rotate.start` before the key file exists, then `keyring.rotate` after it. Both have target kind `kek`, target ID the ID of the new KEK, and the detail `idempotency`: the SHA-256, in hex, of the caller scope and the idempotency key. |

The request `RotateKEKRequest` has no fields. The response `RotateKEKResponse` has these fields:

| Field | Type | Description |
| --- | --- | --- |
| `key_id` | string | The ID of the new current KEK, for example `file:2`. |

The call refuses while the install has a row of a control process on another host in `control_processes`, live or not ([Control store tables](/reference/control-store/index.html.md#control_processes)). A control server that stopped cleanly, or that ended without its stop, keeps its row for 24 hours, so it counts too. The call fails with `failed_precondition` and the message `the install has a control server on <host> too, and a rotation makes the new key current only on this server. Run hopboxd admin kek stage, copy the staged file to every control server, then run hopboxd admin kek promote`. The new key gets the next ID of the install: one after the newest key of this keyring and of `kek_fingerprints`, so a key that another control server made, or a stage that an operator gave up, never shares its ID ([Sealing](/reference/sealing/index.html.md#key-fingerprints)). The check comes after the policy check and before the steps below. The call takes these steps. The audit entries of steps 4 and 6 commit in transactions of their own, apart from the transaction of the call. A key file is live as soon as it exists. Each entry commits also after the client gave up. A failed entry is tried 3 times, with waits of 0.1 s and 0.4 s, and each try waits at most 10 s.

1. The policy check.
2. It looks in the audit log for a rotation (`keyring.rotate`) of the last 24 hours with the same idempotency key. If it finds one, it returns the ID of that key and stops.
3. It looks for a start (`keyring.rotate.start`) of the last 24 hours with the same idempotency key. The start names a key. If that key exists as `<n>.key` or `<n>.staged`, and no rotation entry names it, an earlier try made the key and stopped before step 6. The call then takes that key. If the key is still staged, the call records its fingerprint, as step 5 does, and promotes the file to `<n>.key`. Then it goes to step 6.
4. It writes the start entry `keyring.rotate.start`. The entry has the ID of the next key and the hash of the idempotency key.
5. It writes the new key as the staged file `<n>.staged`, and records its fingerprint. If the install has another key of that ID, the call fails with `failed_precondition`, removes the staged file, and the current KEK stays. Then it promotes the file to `<n>.key`. From this moment, the new KEK is current.
6. It writes the rotation entry `keyring.rotate`.
7. It logs `rotated the key-encryption key` at level info, with `component=service`, `key_id`, and `caller`.

So a retry with the same idempotency key never makes a second key, wherever the first try stopped:

| The first try stopped | A retry |
| --- | --- |
| before step 4, or at step 4 | makes the key, as a first try does |
| after step 4, before the key file existed | finds the start, sees no key file, and makes the key |
| in step 5, after the staged file existed and before its promote | finds the start and the staged file, records its fingerprint, promotes it, and records that key. No staged file stays behind. |
| after step 5, before step 6 committed | finds the start and the key file, and records that key |
| after step 6, for example after a failed commit of the idempotency key | finds the rotation entry and returns its key |

Steps 2 and 3 matter because a call that fails leaves no idempotency response, so a retry runs the call again ([the API server](/reference/api/index.html.md#idempotency-keys)). Without an idempotency key, for example from a test of the service layer, the call skips steps 2 to 4.

One `hopboxd` runs one rotation at a time, so two calls at the same time get two keys in turn. The keyring can refuse the rotation, for example because others can read the keyring dir. Then the call gets `failed_precondition` with the message of the keyring ([Sealing](/reference/sealing/index.html.md#errors)). Any other failure of the key file, such as an I/O error, gets `internal`.

If the rotation entry still fails after its tries, the new KEK stays current without its rotation entry. Then `hopboxd` logs `rotated the key-encryption key, but the audit entry failed` at level error, with `key_id` and `err`, and the call fails with `internal`. Each failed try of an entry logs `audit entry failed` at level warn, with `action`, `try`, and `err`. A retry of the call with the same idempotency key records the key at step 3.

A rotation does not rewrap the sealed values. They keep their old KEK until `RewrapSealedValues` moves them. The rewrap also moves the key of the install CA ([Control store tables](/reference/control-store/index.html.md#install_ca)).

Example:

```text
$ sudo hopboxd admin kek rotate
file:2
```

### StageKEK

Adds a new KEK to the file keyring in `<data_dir>/kek` as a staged key file `<n>.staged`. A staged KEK opens values but seals none, so the current KEK stays the same and the other control servers keep working. `<n>` is 1 more than the highest number of the key files and the staged files. The operator then copies the file to every other control server and calls [PromoteKEK](#promotekek) ([Sealing](/reference/sealing/index.html.md#several-control-servers)).

| Item | Value |
| --- | --- |
| Procedure | `/hopbox.admin.v1.KeyringService/StageKEK` |
| Policy action | `keyring.stage`: `local_root` only. The key files live on one control server, so only root on that server stages a key. |
| Idempotent option | no |
| Audit entry | action `keyring.stage`, target kind `kek`, target ID the ID of the staged KEK. It commits in a transaction of its own, after the file exists. |

The request `StageKEKRequest` has no fields. The response `StageKEKResponse` has these fields:

| Field | Type | Description |
| --- | --- | --- |
| `key_id` | string | The ID of the staged KEK, for example `file:2`. |
| `file` | string | The path of the staged key file on this server, for example `/var/lib/hopbox/kek/2.staged`. Copy it to the same path on every other control server. |

The call logs `staged a key-encryption key` at level info, with `component=service`, `key_id`, and `caller`. The keyring can refuse the stage, for example because others can read the keyring dir. Then the call gets `failed_precondition` with the message of the keyring ([Sealing](/reference/sealing/index.html.md#errors)). If the audit entry fails, the staged file stays, the call logs `staged a key-encryption key, but the audit entry failed` at level error with `key_id` and `err`, and the call fails with `internal`.

Example:

```text
$ sudo hopboxd admin kek stage
file:2	/var/lib/hopbox/kek/2.staged
```

### PromoteKEK

Makes a staged KEK the current one on every control server ([Sealing](/reference/sealing/index.html.md#several-control-servers)). The call returns when every live control process seals with the key.

| Item | Value |
| --- | --- |
| Procedure | `/hopbox.admin.v1.KeyringService/PromoteKEK` |
| Policy action | `keyring.promote`: `local_root` only |
| Idempotent option | no. A second call for a key that is current changes nothing on the key files. |
| Audit entry | action `keyring.promote`, target kind `kek`, target ID the promoted key. It commits in the transaction that records the promote. |

The request `PromoteKEKRequest` has the field `key_id`, the ID of the staged KEK, such as `file:2`. The response `PromoteKEKResponse` has these fields:

| Field | Type | Description |
| --- | --- | --- |
| `key_id` | string | The ID of the key that now seals new values. |

The call takes these steps:

1. The policy check.
2. It refuses an ID that is no file key ID, with `invalid_argument` and the message `"<id>" is not the ID of a file key, such as file:2`.
3. It refuses with `failed_precondition` when this server has neither a staged file nor a key file of the key: `this control server has no key file of <id>. Stage it with hopboxd admin kek stage first`.
4. It refuses with `failed_precondition` when a key file with a higher number exists: `the key file <k> is newer than <id>, so <id> would never seal. Stage a new key instead`.
5. It refuses with `failed_precondition` while another live control process does not list the key in `kek_ids` of its row: `the control process <id> on <host> (hopboxd <release>) reports no key file of <key>. Copy the staged file to the key dir of that server with mode 0400, and promote again after its next renewal, within 5 s`.
6. In one transaction, it sets `install_config.promoted_kek` to the key ([Control store tables](/reference/control-store/index.html.md#install_config)), publishes the change event `keyring.kek_promoted`, and writes the audit entry.
7. It promotes the file on this server: it links `<n>.staged` to `<n>.key`, removes the staged name, and syncs the dir. A second promote of the same key is a no-op. If the file is not in the dir, the call gets `failed_precondition` with `seal: <dir>: the keyring has no staged key file of this key-encryption key file:<n>`.
8. It logs `promoted a key-encryption key` at level info, with `component=service`, `key_id`, and `caller`.
9. It waits until every other live control process reports the key as `kek_current`. It reads the rows every 250 ms. If the deadline of the call ends first, it fails with `unavailable` and the message `<key> is promoted, but the control process <id> still seals with <old key>: ...`. The flag stays recorded, so the control processes go on.

Each control process applies the promote by itself, from the flag `promoted_kek`: at its start, at each change event `keyring.kek_promoted`, and at each renewal of its row while its key dir has a staged file or while it misses the promoted key. It then promotes its own staged copy and logs `made the promoted key-encryption key current` at level info, with `component=service` and `key_id`. A process that has no file of the promoted key logs the error `the install promoted a key-encryption key that this control server has no file of. Copy its key file from another control server`, with `component=control`, `key_id`, and `dir`, once for each missing key ([hopboxd](/reference/hopboxd/index.html.md#logs)). One `hopboxd` runs one stage, promote, or rotation at a time.

Example:

```text
$ sudo hopboxd admin kek promote file:2
file:2
```

### ListKEKs

Lists the key-encryption keys of the keyring in `<data_dir>/kek`, and the keys that sealed values name. For each key, it gives the number of values that the key seals. A key that seals no value can go, after a backup ([Sealing](/reference/sealing/index.html.md#rewrap)).

| Item | Value |
| --- | --- |
| Procedure | `/hopbox.admin.v1.KeyringService/ListKEKs` |
| Policy action | `keyring.list`: `local_root` only |
| Idempotent option | no |
| Audit entry | action `keyring.list`, no target |

The request `ListKEKsRequest` has no fields. The response `ListKEKsResponse` has the field `keys`, a repeated `KEK`. The keys of the `.key` files come first, oldest first, then the staged keys. A key that a value names, but that the keyring does not have, comes last.

| `KEK` field | Type | Description |
| --- | --- | --- |
| `id` | string | The ID of the key, such as `file:1`. |
| `current` | bool | `true` for the key that seals new values. A staged key is never current. |
| `staged` | bool | `true` for a staged key, which opens values but seals none until a promote. A staged key has `in_keyring` set to `true`. |
| `in_keyring` | bool | `false` for a key whose key file is gone. A value that it seals does not open any more. |
| `sealed_values` | int64 | The number of sealed values whose data key this key seals. |

A keyring dir that the checks refuse gets `failed_precondition` ([Sealing](/reference/sealing/index.html.md#errors)). A missing keyring dir holds no key.

### RewrapSealedValues

Moves every sealed value of the control store to the current key-encryption key ([Sealing](/reference/sealing/index.html.md#rewrap)). It reseals only the data keys. The values stay as they are.

| Item | Value |
| --- | --- |
| Procedure | `/hopbox.admin.v1.KeyringService/RewrapSealedValues` |
| Policy action | `keyring.rewrap`: `local_root` only |
| Idempotent option | no. A second call finds nothing to move. |
| Audit entry | action `keyring.rewrap`, target kind `kek`, target ID the current key, details `rewrapped`: the number of values that moved |

The request `RewrapSealedValuesRequest` has no fields. The response `RewrapSealedValuesResponse` has these fields:

| Field | Type | Description |
| --- | --- | --- |
| `current_key_id` | string | The ID of the key that now seals the values. |
| `rewrapped` | int64 | The number of values that moved to it. |
| `keys` | repeated `KEK` | Every key after the rewrap, as `ListKEKs` returns them. |

Each value commits on its own. A value that changes during the rewrap, for example through `SetSMTPConfig` or `SetOAuthClient`, keeps its new seal. A value whose key is gone fails the call with `internal`, and the values before it stay moved. A failed rewrap still writes its audit entry, with the details `rewrapped` and `failed` set to `true`. If that entry fails too, the call logs `the audit entry of a failed rewrap failed` at level error, with `component=service` and `err`. A keyring without a key gets `failed_precondition`. The call logs `rewrapped the sealed values` at level info, with `component=service`, `key_id`, `rewrapped`, and `caller`.

```text
$ sudo hopboxd admin kek rewrap
rewrapped 1 to file:2
file:1	0	
file:2	1	current
```

## MailService

`hopbox.admin.v1.MailService` reads the mail files that the file transport wrote on this control server ([Mail](/reference/mail/index.html.md#the-file-transport)). The files hold sign-in links and invitation links. So the policy allows only `local_root`, through the admin socket of the server that holds the files. An admin API key reaches whichever control server answers, so the policy refuses it. Each control server has its own folder, `<data_dir>/mail`.

The message `MailFile` describes one file:

| Field | Type | Description |
| --- | --- | --- |
| `name` | string | The name of the file, such as `20261023T094531.606059000Z-5cb912a8d0bb45d7.eml`. The names sort by the time of the write. |
| `create_time` | `google.protobuf.Timestamp` | The time of the write, from the name. |
| `to` | string | The `To` header, such as `<ann@example.org>`. |
| `subject` | string | The `Subject` header, decoded. |
| `size_bytes` | int64 | The size of the file. |

### ListMailFiles

Lists the mail files of this control server, newest first. It prunes the folder first: it keeps the newest 1000 files, and removes any file older than 7 days ([Mail](/reference/mail/index.html.md#bounds)).

| Item | Value |
| --- | --- |
| Procedure | `/hopbox.admin.v1.MailService/ListMailFiles` |
| Policy action | `mail_file.list`: `local_root` only |
| Idempotent option | no |
| Audit entry | action `mail_file.list`, no target |

The request `ListMailFilesRequest` has no fields. The response `ListMailFilesResponse` has these fields:

| Field | Type | Description |
| --- | --- | --- |
| `dir` | string | The mail folder of the server, such as `/var/lib/hopbox/mail`. |
| `files` | repeated `MailFile` | The files, newest first. Before the first mail, the folder does not exist, and the list is empty. |

A file that does not read stays out of the list, such as a symlink that leads out of the folder. The other files still show ([Mail](/reference/mail/index.html.md#logs)).

The files stay after a switch back to `MAIL_TRANSPORT_SMTP`, until a prune removes them.

### GetMailFile

Returns one mail file of this control server, with its content.

| Item | Value |
| --- | --- |
| Procedure | `/hopbox.admin.v1.MailService/GetMailFile` |
| Policy action | `mail_file.get`: `local_root` only |
| Idempotent option | no |
| Audit entry | action `mail_file.get`, target kind `mail_file`, target ID the name of the file |

The request `GetMailFileRequest` has the field `name`, a string: the name of the file, as `ListMailFiles` returns it. The response `GetMailFileResponse` has these fields:

| Field | Type | Description |
| --- | --- | --- |
| `file` | `MailFile` | The file. |
| `content` | bytes | The whole file: an RFC 5322 message, with the headers and the body that the SMTP transport sends. The body is in quoted-printable, so a long link breaks over two lines. |
| `text` | string | The body, decoded from its transfer encoding, with plain newlines. A link in it is whole. |

A name that no mail file can have, such as a path, gets `invalid_argument`. A name that is not in the folder gets `not_found` ([Mail](/reference/mail/index.html.md#errors)).

## LoginLinkService

`hopbox.admin.v1.LoginLinkService` makes the one-time sign-in links of the first instance admin.

### CreateLoginLink

Makes a first-admin login link for an email. The link works once, for 15 minutes ([Sessions, cookies, and sign-in links](/reference/sessions/index.html.md#first-admin-login-links)). It signs in to the account of earlier login links for the email, or to a new account, and makes that account an instance admin. The email stays unverified, so it never links the account to another sign-in method.

| Item | Value |
| --- | --- |
| Procedure | `/hopbox.admin.v1.LoginLinkService/CreateLoginLink` |
| Policy action | `login_link.create`: `local_root` only |
| Idempotent option | no. Each call makes a new link. |
| Audit entry | action `login_link.create`, target kind `login_link`, target ID the UUID of the link, details `email`. The entry never holds the token. |

The request `CreateLoginLinkRequest` has the field `email`, a string. The response `CreateLoginLinkResponse` has these fields:

| Field | Type | Description |
| --- | --- | --- |
| `url` | string | The link, such as `https://example.com/auth/login-link#token=<token>`. It is a secret until its use or its end. |
| `email` | string | The email, in lower case. |
| `expire_time` | `google.protobuf.Timestamp` | The end of the link. |

| Error | Code |
| --- | --- |
| The email breaks a rule ([Domain rules](/reference/domain-rules/index.html.md#emails)) | `invalid_argument` |
| The install has no domain: `failed precondition: the install has no domain, and the link needs one. Set it with hopboxd admin config set --domain` | `failed_precondition` |

## InstanceAdminService

`hopbox.admin.v1.InstanceAdminService` manages the instance admins: the accounts that operate the install. An install keeps at least one instance admin, after the first one signs in. Each grant and revoke locks the table, so two revokes at the same time cannot remove the last two admins.

### ListInstanceAdmins

Lists the instance admins, in the order of their grant.

| Item | Value |
| --- | --- |
| Procedure | `/hopbox.admin.v1.InstanceAdminService/ListInstanceAdmins` |
| Policy action | `instance_admin.list`: `local_root`, `admin_key`, and an [admin session](#admin-sessions) |
| Idempotent option | no |
| Audit entry | action `instance_admin.list`, no target |

The request `ListInstanceAdminsRequest` has no fields. The response `ListInstanceAdminsResponse` has the field `admins`, a repeated `InstanceAdmin`:

| Field | Type | Description |
| --- | --- | --- |
| `account_id` | string | The UUID of the account. |
| `display_name` | string | The display name of the account. |
| `emails` | repeated `InstanceAdminEmail` | Each email of the account, with `email` and `verified`. |
| `grant_time` | `google.protobuf.Timestamp` | The time of the grant. |

### GrantInstanceAdmin

Makes an account an instance admin. A grant to an instance admin changes nothing and succeeds.

| Item | Value |
| --- | --- |
| Procedure | `/hopbox.admin.v1.InstanceAdminService/GrantInstanceAdmin` |
| Policy action | `instance_admin.grant`: `local_root`, `admin_key`, and an [admin session](#admin-sessions) |
| Idempotent option | no |
| Audit entry | action `instance_admin.grant`, target kind `account`, details `already` (`true` for an account that was an instance admin already) |

The request `GrantInstanceAdminRequest` names the account with one of these fields:

| Field | Type | Description |
| --- | --- | --- |
| `account_id` | string | The UUID of the account. |
| `email` | string | A verified email of the account. An unverified email, such as the email of a login link, names no account, because many accounts can hold it. |

The response `GrantInstanceAdminResponse` has the field `account_id`. A request without a field, or with an ID that is not a UUID, gets `invalid_argument`. An unknown account gets `not_found`.

### RevokeInstanceAdmin

Takes the instance admin role from an account. Its sessions lose the role at their next call. The revoke tells the instance admins and the revoked account ([Notices to instance admins](#notices-to-instance-admins)).

| Item | Value |
| --- | --- |
| Procedure | `/hopbox.admin.v1.InstanceAdminService/RevokeInstanceAdmin` |
| Policy action | `instance_admin.revoke`: `local_root`, `admin_key`, and an [admin session](#admin-sessions) |
| Idempotent option | no |
| Audit entry | action `instance_admin.revoke`, target kind `account` |

The request `RevokeInstanceAdminRequest` has the field `account_id`. The response `RevokeInstanceAdminResponse` has no fields.

| Error | Code |
| --- | --- |
| The ID is not a UUID | `invalid_argument` |
| The account is not an instance admin | `not_found` |
| The account is the last instance admin: `failed precondition: the account <id> is the last instance admin, and an install keeps at least one. Grant the role to another account first` | `failed_precondition` |

### Notices to instance admins

Some calls leave something that lasts after the session that made them. For example, a stolen admin session of 10 minutes can make an admin key that works for 365 days. So these calls tell the instance admins, in the console and by email (plan question Q103):

| Call | When it tells the instance admins |
| --- | --- |
| `CreateAdminKey` | Always. |
| `RegisterAdminKey` | When it makes a key, with `registered` set to `true`. |
| `GrantInstanceAdmin` | When the account was not an instance admin. |
| The sign-in of a login link ([the sign-in, account, and audit API](/reference/auth-api/index.html.md#verifyloginlink)) | When it makes the account an instance admin. |
| `RevokeInstanceAdmin` | Always. Otherwise a stolen session can revoke the other admins first, so that nobody hears of its later actions. The notice comes before the revoke, so the revoked account gets the email and a notice too. Its notice shows only if it gets the role again. |
| `SetSMTPConfig` and `ClearSMTPConfig` | Always. |
| `SetOAuthClient` and `ClearOAuthClient` | Always. |
| `UpdateInstallConfig` | When it changes `mail_transport`, because the transport decides whether the install uses the SMTP server. |

A notice is the audit entry of the call. In the transaction of the call, each instance admin other than the actor gets an open notice in the table `admin_notices` ([Control store tables](/reference/control-store/index.html.md#admin_notices)). The banner of the console and the Admin area show the open notices ([Console](/reference/console/index.html.md#admin-area)), and `ListAdminNotices` returns them. `hopboxd admin` has no command for them, because root has no account and so no notices.

After the commit, hopbox sends an email to each email of each of these instance admins. When the actor is a session, its account gets a copy too. The person whose session someone stole is the person who knows best that the action was not theirs. The sign-in of a login link had no session before the grant, so its account gets no copy. An email that no sign-in method verified gets the notice too, because only root adds such an email to an account, in a login link.

The email has these parts:

- What changed, with the facts of the audit entry, such as the name, the ID, and the end of a new key.
- Who did it: the session and its account, the admin API key, or root on a control server with its user and process.
- The time and the ID of the audit entry.
- How to undo it: the page of the Admin area and the `hopboxd admin` command. For an action of a session, also how to revoke the role of its account, which closes the admin API to each session of the account at once.
- The link to `/admin/notices`, or for the copy of the actor, a line that says why the actor gets it.

The email and the notice never hold a secret: no admin key, no client secret, and no SMTP password. They hold only the facts of the audit entry, which never holds a secret value either.

A change of the mail config sends its email the way that the install sent mail before the change, and the new way only when that fails. The changes of the mail config are `SetSMTPConfig`, `ClearSMTPConfig`, and a change of `mail_transport`. So a new SMTP server that an attacker sets does not swallow the notice of its own change.

A failed email never fails the call. The emails of one notice share one limit of 20 s. For each address that fails, hopbox logs `the email of an admin notice failed` at level error, with `audit_entry`, `account`, and `err`. It also writes the audit entry `admin_notice.mail_failed`, with the actor kind `system`, the target kind `audit_entry` and the ID of the entry of the call, and the details `account`, `email`, and `error`. The error is the reply of the mail server. If that text looks like a credential, the audit log refuses it, so `error` holds a fixed text instead, and only the log has the reply. The notice stays in the Admin area. An install without an SMTP config fails each email this way. With the file transport, the email goes to the mail folder ([Mail](/reference/mail/index.html.md#the-file-transport)). After the sends, hopbox logs `sent the emails of an admin notice` at level info, with `audit_entry`, `action`, `sent`, and `failed`. If it cannot write the audit entries of the failures, it logs `the audit entries of the failed emails of an admin notice failed` at level error.

### AdminNotice

| Field | Type | Description |
| --- | --- | --- |
| `id` | string | The UUID of the audit entry of the action. |
| `action` | string | The action of the audit entry, such as `admin_key.create`. |
| `summary` | string | What changed, in one to three sentences, such as `An admin API key was made: "key1", ID <id>, valid until <time>.` |
| `advice` | string | How to undo the change if nobody that the reader trusts made it. |
| `actor_kind` | string | The actor kind of the audit entry: `session`, `admin_key`, or `local_root`. |
| `actor_account_id` | string | The UUID of the account of a session. Empty for the other actors. |
| `actor_name` | string | The display name of that account with its first email, such as `User 1 <user1@example.org>`, or its first email alone. Empty for the other actors. |
| `actor_credential_id` | string | The UUID of the session or of the admin key. Empty for `local_root`. |
| `target_kind` | string | The target kind of the audit entry, or empty. |
| `target_id` | string | The target ID of the audit entry, or empty. |
| `create_time` | `google.protobuf.Timestamp` | The time of the action. |

### ListAdminNotices

Lists the open notices of the instance admin of the session, newest first, at most 100.

| Item | Value |
| --- | --- |
| Procedure | `/hopbox.admin.v1.InstanceAdminService/ListAdminNotices` |
| Policy action | `admin_notice.list`: an [admin session](#admin-sessions) only. A notice belongs to an account, so `local_root` and an admin API key get `permission_denied`. |
| Idempotent option | no |
| Audit entry | action `admin_notice.list`, no target |

The request `ListAdminNoticesRequest` has no fields. The response `ListAdminNoticesResponse` has these fields:

| Field | Type | Description |
| --- | --- | --- |
| `notices` | repeated `AdminNotice` | The open notices, newest first, at most 100. |
| `total` | int32 | The count of all open notices of the account. It is more than the length of `notices` when the account has more than 100. |

### DismissAdminNotices

Removes notices from the list of the instance admin of the session. The audit log keeps the entries of the actions.

| Item | Value |
| --- | --- |
| Procedure | `/hopbox.admin.v1.InstanceAdminService/DismissAdminNotices` |
| Policy action | `admin_notice.dismiss`: an [admin session](#admin-sessions) only |
| Idempotent option | no. A second call skips the notices that the first call removed. |
| Audit entry | action `admin_notice.dismiss`, no target, details `notices`, the removed IDs joined by commas, and `count` |

The request `DismissAdminNoticesRequest` has the field `ids`, 1 to 100 UUIDs of notices. An ID that names no open notice of the account is skipped, also an open notice of another instance admin. The response `DismissAdminNoticesResponse` has the field `dismissed`, the count of the removed notices.

| Error | Code |
| --- | --- |
| No ID, more than 100 IDs, or an ID that is not a UUID | `invalid_argument` |
| A caller other than an admin session | `permission_denied`, or `unauthenticated` for an anonymous caller |

## AccountService

`hopbox.admin.v1.AccountService` lets an operator recover the sign-in of an account. A person can lose the only sign-in method of the account, for example the GitHub user. A new GitHub user with the same verified email then fails, because the account has an identity of a provider ([Sign-in with GitHub and Google](/reference/oauth/index.html.md#recover-a-lost-identity)). The operator removes the old method. The account then has no identity of a provider, so a magic link or the new GitHub user links through the verified email.

### ListAccountSignInMethods

Lists the sign-in methods of one account, oldest first.

| Item | Value |
| --- | --- |
| Procedure | `/hopbox.admin.v1.AccountService/ListAccountSignInMethods` |
| Policy action | `admin_account.list_sign_in_methods`: `local_root` and `admin_key` |
| Idempotent option | no |
| Audit entry | action `admin_account.list_sign_in_methods`, target kind `account`, in the personal org of the account, so its owner reads the entry |

The request `ListAccountSignInMethodsRequest` has the oneof `account`: `account_id`, the UUID of the account, or `email`, a verified email of the account. An unverified email names no account. The response `ListAccountSignInMethodsResponse` has `account_id` and `methods`, a list of `AccountSignInMethod`:

| Field | Type | Description |
| --- | --- | --- |
| `id` | string | The UUID of the method. |
| `kind` | string | `email_link`, `login_link`, `github`, or `google`. |
| `subject` | string | The identity that the method proves: the email of a link, the numeric user ID at GitHub, or the `sub` claim at Google. |
| `email` | string | The email that the method names. |
| `last_use_time` | `google.protobuf.Timestamp` | The time of the last sign-in with the method. |
| `create_time` | `google.protobuf.Timestamp` | The time of the link of the method to the account. |

| Error | Code |
| --- | --- |
| Both or neither of `account_id` and `email`, an ID that is not a UUID, or an email that breaks the rules of [Domain rules](/reference/domain-rules/index.html.md#emails) | `invalid_argument` |
| No account has the ID, or the verified email | `not_found` |

### RemoveAccountSignInMethod

Removes one sign-in method of an account. The emails of the account stay. While the account has no identity of GitHub or Google, a later sign-in that proves a verified email of the account links its method again ([Sessions, cookies, and sign-in links](/reference/sessions/index.html.md#what-a-sign-in-does)). The removal does not end the sessions or the CLI sign-ins that the method started. The person ends them with `SignOut` and `EndCLISignIn` ([the sign-in, account, and audit API](/reference/auth-api/index.html.md#endclisignin)).

| Item | Value |
| --- | --- |
| Procedure | `/hopbox.admin.v1.AccountService/RemoveAccountSignInMethod` |
| Policy action | `admin_account.remove_sign_in_method`: `local_root` and `admin_key` |
| Idempotent option | no |
| Audit entry | action `admin_account.remove_sign_in_method`, target kind `sign_in_method`, in the personal org of the account. The details are `account`, `kind`, `email`, and `last_method` (`true` or `false`). The owner of the account reads the entry in the audit log of the personal org. |

The request `RemoveAccountSignInMethodRequest` has these fields:

| Field | Type | Default | Description |
| --- | --- | --- | --- |
| `account_id` | string | none, required | The UUID of the account. |
| `method_id` | string | none, required | The UUID of the method, as `ListAccountSignInMethods` returns it. |
| `allow_last` | bool | `false` | Removes also the last method of the account. Then the account signs in only through a new method that proves one of its verified emails. |

The response `RemoveAccountSignInMethodResponse` has no fields.

| Error | Code |
| --- | --- |
| An ID is not a UUID | `invalid_argument` |
| No account has the ID, or the account has no such method | `not_found` |
| The method is the last one, and `allow_last` is `false`: `failed precondition: this is the last sign-in method of the account. Allow the removal of the last method to remove it. ...` | `failed_precondition` |
| The method is the last one, and the account has no verified email, such as the account of a login link. No new method can reach such an account: `failed precondition: the account has no verified email, so no new sign-in method can reach it. Keep its last method` | `failed_precondition` |

## AdminKeyService

`hopbox.admin.v1.AdminKeyService` manages the admin API keys ([CLI tokens, API keys, admin API keys, and SSH keys](/reference/credentials/index.html.md#admin-api-keys)). An admin API key has install scope and calls the admin API on the API listener, for example from the hopbox Terraform provider. Only `local_root` calls this service, so no key makes another key. The keys live in the table `admin_keys` ([Control store tables](/reference/control-store/index.html.md#admin_keys)). `hopboxd admin admin-key` calls this service ([hopboxd](/reference/hopboxd/index.html.md#admin-admin-key-create)).

### AdminKey

| Field | Type | Description |
| --- | --- | --- |
| `id` | string | The UUID of the key. |
| `name` | string | The name that root gave the key, such as `terraform`. |
| `expire_time` | `google.protobuf.Timestamp` | The end of the key. |
| `last_use_time` | `google.protobuf.Timestamp` | The last call with the key, exact to a minute. Unset for a key without a call. |
| `create_time` | `google.protobuf.Timestamp` | The creation of the key. |

The enum `AdminKeyLifetime` has the values `ADMIN_KEY_LIFETIME_UNSPECIFIED` (90 days), `ADMIN_KEY_LIFETIME_30_DAYS`, `ADMIN_KEY_LIFETIME_90_DAYS`, and `ADMIN_KEY_LIFETIME_365_DAYS`.

### CreateAdminKey

Makes an admin API key. The response holds the only copy of the key.

| Item | Value |
| --- | --- |
| Procedure | `/hopbox.admin.v1.AdminKeyService/CreateAdminKey` |
| Policy action | `admin_key.create`: `local_root` and an [admin session](#admin-sessions). An admin API key never manages admin keys. |
| Idempotent option | yes. The install keeps no copy of the secret. So a retry with the same idempotency key fails with `already_exists` and the ID of the key. |
| Audit entry | action `admin_key.create`, target kind `admin_key`, details `name` and `expire_time` |

The request `CreateAdminKeyRequest` has these fields:

| Field | Type | Default | Description |
| --- | --- | --- | --- |
| `name` | string | none, required | A name of 1 to 64 characters, such as `terraform` ([Domain rules](/reference/domain-rules/index.html.md#credential-names)). |
| `lifetime` | `AdminKeyLifetime` | 90 days | A preset lifetime. Give it or `expire_time`, not both. |
| `expire_time` | `google.protobuf.Timestamp` | none | A custom end, in the future and at most 365 days ahead. |

The response `CreateAdminKeyResponse` has the fields `key`, an `AdminKey`, and `secret`, the key itself, which starts with `hbxadm_`.

| Error | Code |
| --- | --- |
| No name, a bad name, an unknown lifetime, or an expiry in the past or more than 365 days ahead | `invalid_argument` |
| A caller other than `local_root` | `permission_denied`, or `unauthenticated` for an anonymous caller |
| A retry with an idempotency key that made a key already | `already_exists` |

### RegisterAdminKey

Makes an admin API key from the SHA-256 of its secret, so the secret never reaches the server. `hopboxd install` registers the first key of an install this way. A second registration of the same hash returns the key of the first one, unchanged, with its first name and expiry.

| Item | Value |
| --- | --- |
| Procedure | `/hopbox.admin.v1.AdminKeyService/RegisterAdminKey` |
| Policy action | `admin_key.register`: `local_root` only |
| Idempotent option | yes. A retry with the same idempotency key returns the first response. |
| Audit entry | action `admin_key.register`, target kind `admin_key`, details `name`, `expire_time`, and `new` (`false` for a hash that a key had already) |

The request `RegisterAdminKeyRequest` has the fields of `CreateAdminKeyRequest`, plus `sha256`: the SHA-256 of the whole key text, `hbxadm_` included, in 64 lowercase hex characters ([CLI tokens, API keys, admin API keys, and SSH keys](/reference/credentials/index.html.md#admin-api-keys)). The response `RegisterAdminKeyResponse` has these fields:

| Field | Type | Description |
| --- | --- | --- |
| `key` | `AdminKey` | The key of the hash. |
| `registered` | bool | `true` for a key that this call made. `false` for a hash that an earlier call registered. |

A hash that is not 64 lowercase hex characters gets `invalid_argument` with the message `invalid argument: the hash must be the SHA-256 of the key in 64 lowercase hex characters`. The hash of an expired key gets `failed_precondition` with the message `failed precondition: the admin key <id> of this hash expired at <time>. Make a new key with a new hash`. The other errors are those of `CreateAdminKey`.

### ListAdminKeys

Lists the admin API keys, oldest first. An expired key stays in the list for 30 days. Then the leader deletes it.

| Item | Value |
| --- | --- |
| Procedure | `/hopbox.admin.v1.AdminKeyService/ListAdminKeys` |
| Policy action | `admin_key.list`: `local_root` and an [admin session](#admin-sessions). An admin API key never manages admin keys. |
| Idempotent option | no |
| Audit entry | action `admin_key.list`, no target |

The request `ListAdminKeysRequest` has no fields. The response `ListAdminKeysResponse` has the field `keys`, a repeated `AdminKey`.

### RevokeAdminKey

Deletes an admin API key at once. The next call with the key gets `unauthenticated`.

| Item | Value |
| --- | --- |
| Procedure | `/hopbox.admin.v1.AdminKeyService/RevokeAdminKey` |
| Policy action | `admin_key.revoke`: `local_root` and an [admin session](#admin-sessions). An admin API key never manages admin keys. |
| Idempotent option | no |
| Audit entry | action `admin_key.revoke`, target kind `admin_key`, details `name` |

The request `RevokeAdminKeyRequest` has the field `id`, the UUID of the key. The response `RevokeAdminKeyResponse` has no fields. An ID that is not a UUID gets `invalid_argument`. An unknown key gets `not_found`.

Example:

```sh
sudo curl -s --unix-socket /run/hopbox/admin.sock \
  -H 'Content-Type: application/json' -d '{}' \
  http://hopboxd/hopbox.admin.v1.AdminKeyService/ListAdminKeys
```

## ServerService

`hopbox.admin.v1.ServerService` enrolls servers and manages the servers of the install. The servers live in the table `servers` ([Control store tables](/reference/control-store/index.html.md#servers)). [Enrollment and the install PKI](/reference/enrollment/index.html.md) describes the enrollment, and [the control channel](/reference/control-channel/index.html.md) the heartbeats that keep each server current.

### Server

The message `Server` is one enrolled server, with the facts of its last heartbeat:

| Field | Type | Description |
| --- | --- | --- |
| `id` | string | The ID of the server, a UUID. |
| `pool` | string | The name of the pool of the server, for example `default`. |
| `hostname` | string | The host name that the server sent at its enrollment. |
| `roles` | repeated string | The roles of the server, for example `node` or `dns`. Its certificate carries them. |
| `state` | `ServerState` | The state of the server. |
| `cordoned` | bool | True while an instance admin keeps the server cordoned, also while it is unresponsive. |
| `status` | `ServerStatus` | The health that the server reported in its last heartbeat. |
| `problems` | repeated string | The failed checks of the last heartbeat, one line each, for example `kvm: /dev/kvm does not exist. ...`. The list also has each check of severity `warn`, also with the `status` `SERVER_STATUS_OK` ([the preflight check](/reference/preflight/index.html.md)). |
| `labels` | map of string to string | The labels of the server, for example `zone=gra`. They come from the bootstrap config of the server, until `SetServerLabels` sets them. |
| `labels_set_by_admin` | bool | True once `SetServerLabels` set the labels. The heartbeats of the server then no longer change them. |
| `machine_count` | int32 | The machines on the server that their executor has not deleted yet, stopped machines included. The control store counts them. Their disks live on the server. |
| `hopboxd_version` | string | The version of `hopboxd` on the server, for example `v0.1.0`. |
| `kernel_release` | string | The release of the host kernel, for example `6.18.57-hopbox1`. |
| `cpu` | `CPUFingerprint` | The CPU of the server. |
| `compatibility_class` | string | The compatibility class: the servers that can restore the memory state of each other, for example `cc-9c6a6f2eb2ae6047` ([the control channel](/reference/control-channel/index.html.md)). |
| `capacity` | `Capacity` | What the server has for machines. |
| `allocations` | repeated `Allocation` | The resources that the running machines of the server hold, from its last heartbeat. Empty for a server that runs no machine executor ([the node executor](/reference/executor/index.html.md)). |
| `actions` | repeated `ActionInProgress` | The actions that the executors of the server ran at its last heartbeat, such as `boot`. Empty for a server that runs no machine executor. |
| `last_heartbeat_time` | `google.protobuf.Timestamp` | The time of the last heartbeat. Absent before the first one. |
| `enroll_time` | `google.protobuf.Timestamp` | The time of the enrollment. |
| `certificate_expire_time` | `google.protobuf.Timestamp` | The end of the newest certificate of the server. |
| `version` | int64 | The row version. It grows by 1 with each change, such as each heartbeat. |
| `update_time` | `google.protobuf.Timestamp` | The time of the last change. |
| `mesh` | `ServerMesh` | The part of the server in the mesh. Absent until the server reports the key of its WireGuard device, and for a server outside the mesh ([the mesh](/reference/mesh/index.html.md)). |
| `dns_zone_serial` | optional uint32 | The serial of the zones that the DNS role of the server served at its last heartbeat. Absent on a server without the DNS role, and before its first zones. Compare it with the serial of the zones of the install to find a DNS server that lags ([the DNS role](/reference/dns/index.html.md#the-serial-of-each-dns-server)). |
| `drain` | `ServerDrain` | The drain of the server (field 26). Absent when the server does not drain, and after the decommission. |
| `peer_address` | string | The peer address of the server (field 27): the public IPv4 address at which the other servers reach it, such as `203.0.113.12` ([Enrollment and the install PKI](/reference/enrollment/index.html.md#peer-addresses)). Empty for a server without one. The enrollment token or `SetServerPeerAddress` sets it. A heartbeat never changes it. |
| `removal` | `ServerRoleRemoval` | The pending removal of roles from the server (field 28) ([RemoveServerRoles](#removeserverroles)). Absent without one. |
| `removed_roles` | repeated string | The roles that a removal took from the server (field 29). A token adds a role to the enrolled server only when this list holds the role ([Enrollment and the install PKI](/reference/enrollment/index.html.md#add-roles-to-an-enrolled-server)). |

The message `ServerMesh` has these fields:

| Field | Type | Description |
| --- | --- | --- |
| `address` | string | The mesh address of the server, in `10.239.0.0/16`, for example `10.239.0.3`. |
| `vrack_address` | string | Deprecated, and always empty. The mesh runs on the peer address of each server, the field `peer_address` of `Server`. |
| `public_key` | string | The public key of the WireGuard device `hbm0` of the server, in base64, as `wg show` prints it. |

The message `ServerDrain` has these fields:

| Field | Type | Description |
| --- | --- | --- |
| `start_time` | `google.protobuf.Timestamp` | The start of the drain, or of its last retry. |
| `moving` | int32 | The moves from the server that run now. |
| `stranded` | repeated `StrandedMachine` | The machines that the drain cannot move. Each one keeps its disk on the server. |
| `empty_checks` | int32 | The empty checks in a row: 0 to 3. |
| `last_check_time` | `google.protobuf.Timestamp` | The time of the last empty check. Absent before the first one. |
| `error` | string | Why the drain waits now, such as a refused decommission. Empty while the drain goes on. |
| `stuck` | bool | True when no move runs, every machine left on the server is stranded, and a machine is stranded or the decommission was refused. A second `DrainServer` tries again. |

The message `ServerRoleRemoval` has these fields:

| Field | Type | Description |
| --- | --- | --- |
| `roles` | repeated string | The roles that go, such as `dns` and `edge`, in the order `node`, `dns`, `edge`. |
| `start_time` | `google.protobuf.Timestamp` | The start of the removal. A move that failed before it strands no machine. |
| `cordoned_before` | bool | True when the server was cordoned before the removal. The end and the cancel of the removal lift only a cordon that the removal set. |
| `moving` | int32 | The moves from the server that run now. Only a removal of `node` has moves. |
| `stranded` | repeated `StrandedMachine` | The machines that the removal of `node` cannot move. Each one keeps its disk on the server. |
| `error` | string | Why the removal waits now, such as a machine that has not booted yet. Empty while the removal goes on. |
| `stuck` | bool | True when no move runs, a machine is stranded, and every machine left on the server is stranded. A cancel ends such a removal ([CancelRoleRemoval](#cancelroleremoval)). |
| `ready` | bool | True when an enrollment of the server with the roles that stay can end the removal. A removal without `node` is ready at once. A removal of `node` is ready once the server holds no machine, no allocation, and no move from or to it. It must also hold no old copy of a moved machine, and no copy of a disk that a failed move left. |

The message `StrandedMachine` has these fields:

| Field | Type | Description |
| --- | --- | --- |
| `machine_id` | string | The ID of the machine. |
| `name` | string | The name of the machine. |
| `reason` | string | Why the machine cannot move. |
| `on_server` | bool | False for a machine on another node whose disk copy a failed move left on this server ([Cold move](/reference/cold-move/index.html.md#drain-of-a-node)). |

A drain and the removal of the role `node` name the machines that they cannot move with this message.

The enum `ServerState` has these values:

| Value | Meaning |
| --- | --- |
| `SERVER_STATE_UNSPECIFIED` | Never in a response. |
| `SERVER_STATE_READY` | The server sends heartbeats and takes new machines. |
| `SERVER_STATE_CORDONED` | An instance admin cordoned the server. It keeps its machines but takes no new one. |
| `SERVER_STATE_UNRESPONSIVE` | The server sent no heartbeat for 20 s, or none since its enrollment. Its machines stay on it. |
| `SERVER_STATE_DRAINING` | A drain cordoned the server and moves its machines to other nodes. The server takes no new machine. |
| `SERVER_STATE_SHUTTING_DOWN` | The drain found the server empty. After 3 empty checks in a row, one minute apart, it decommissions the server. |
| `SERVER_STATE_DECOMMISSIONED` | The server left the install. The install refuses its certificates. |

The enum `ServerStatus` has these values:

| Value | Meaning |
| --- | --- |
| `SERVER_STATUS_UNSPECIFIED` | The server sent no heartbeat yet. |
| `SERVER_STATUS_OK` | Every check of severity `fail` passes. |
| `SERVER_STATUS_DEGRADED` | A check of severity `fail` fails. `problems` says which. |

The message `CPUFingerprint` has these fields:

| Field | Type | Description |
| --- | --- | --- |
| `arch` | string | The CPU architecture, as Go names it, for example `amd64`. |
| `vendor` | string | The `vendor_id` of `/proc/cpuinfo`, for example `GenuineIntel`. |
| `family` | uint32 | The `cpu family` of `/proc/cpuinfo`. |
| `model` | uint32 | The `model` of `/proc/cpuinfo`. |
| `stepping` | uint32 | The `stepping` of `/proc/cpuinfo`. |
| `model_name` | string | The `model name` of `/proc/cpuinfo`, for example `Intel(R) Xeon(R) CPU E5-1650 v4 @ 3.60GHz`. |
| `microcode` | string | The microcode revision, for example `0xb000040`. |
| `flags_hash` | string | The SHA-256, in hex, of the sorted CPU flags. |
| `cpu_template` | string | The Firecracker CPU template of the server. It is `none` in phase 1. |

The message `Capacity` has these fields:

| Field | Type | Description |
| --- | --- | --- |
| `cpu_threads` | uint32 | The count of CPU threads. |
| `memory_bytes` | uint64 | The size of the memory, `MemTotal` of `/proc/meminfo`. |
| `disk_bytes` | uint64 | The size of the file system of the data dir. |
| `disk_free_bytes` | uint64 | The free space on that file system, for an unprivileged user. |

The message `Allocation` has the fields `machine_id` (string, a UUID), `vcpus` (uint32), `memory_bytes` (uint64), and `disk_bytes` (uint64). The message `ActionInProgress` has the fields `machine_id` (string, a UUID), `action` (string, for example `start`), and `start_time` (`google.protobuf.Timestamp`).

### CreateEnrollmentToken

Makes a token with which one new server enrolls into a pool ([Enrollment and the install PKI](/reference/enrollment/index.html.md)). The token works once and expires after its `ttl`, 1 hour by default. The install keeps only the SHA-256 of its secret, so the response holds the only copy of the token. The first call of an install makes the install CA and seals its key with the current key-encryption key.

| Item | Value |
| --- | --- |
| Procedure | `/hopbox.admin.v1.ServerService/CreateEnrollmentToken` |
| Policy action | `enrollment_token.create`: `local_root` and `admin_key` |
| Idempotent option | yes, without a stored response. See below. |
| Audit entry | action `enrollment_token.create`, target kind `enrollment_token`, target ID the token ID, details `pool`, `roles` (joined by commas), `expire_time` (RFC 3339, in UTC), `server`, and `peer_address`. Only a token of an enrolled server has `server`, and only a token with a peer address has `peer_address`. |

The call needs an `Idempotency-Key` header ([the API server](/reference/api/index.html.md#idempotency-keys)). Its response holds the secret of the token, so the control store keeps the response without the field `token`. A retry with the same key makes no second token, and fails with `already_exists` and the message `this idempotency key made the enrollment token <token ID> already, and the install keeps no copy of its secret. Make a new token with a new idempotency key`. Thus the secret never sits in the control store, and a retry never makes a second token. A caller that lost the response makes a new token with a new key. The unused token expires at the end of its `ttl`.

The request `CreateEnrollmentTokenRequest` has these fields:

| Field | Type | Description |
| --- | --- | --- |
| `pool` | string | The name of the live pool that the new server joins ([PoolService](#poolservice)). Empty means the pool `default`. |
| `roles` | repeated string | The roles that the new server can enroll with: `node`, `dns`, `edge`, or more than one. Empty means the role `node` alone. |
| `server_id` | string | The ID of an enrolled server, or empty. The token then adds the roles to that server alone, in its pool. It enrolls no new server ([Enrollment and the install PKI](/reference/enrollment/index.html.md#add-roles-to-an-enrolled-server)). A `pool` other than the pool of the server fails with `invalid_argument`. An unknown server fails with `not_found`, and a decommissioned one with `failed_precondition`. |
| `peer_address` | string | The peer address of the server that enrolls with the token, such as `203.0.113.12`, or empty. It is one public unicast IPv4 address that no other server holds ([Enrollment and the install PKI](/reference/enrollment/index.html.md#peer-addresses)). The enrollment must come from it, and the new server keeps it. |
| `ttl` | `google.protobuf.Duration` | How long the token works, more than 0 and at most 24 hours. Absent or zero means 1 hour. While the token is live, its `peer_address` is in the set `registered` ([Enrollment and the install PKI](/reference/enrollment/index.html.md#peer-addresses)). |

The response `CreateEnrollmentTokenResponse` has these fields:

| Field | Type | Description |
| --- | --- | --- |
| `token` | string | The enrollment token, `hbxe_<id>_<secret>_<cahash>`. It is a secret. Pass it to `hopboxd enroll --token` on the new server ([hopboxd](/reference/hopboxd/index.html.md#enroll)). |
| `token_id` | string | The ID of the token, a UUID, as the audit log names it. |
| `pool` | string | The name of the pool of the token. |
| `expire_time` | `google.protobuf.Timestamp` | The end of the token, `ttl` after the call. |
| `ca_hash` | string | The SHA-256, in hex, of the certificate of the install CA. The token holds it too. |
| `roles` | repeated string | The roles that the new server can enroll with. |
| `server_id` | string | The enrolled server that the token adds roles to, or empty. |
| `peer_address` | string | The address from which the enrollment must come, or empty. |

| Error | Cause |
| --- | --- |
| `failed_precondition` with `failed precondition: the keyring has no key-encryption key to seal or open the value. Run hopboxd admin kek rotate to make the first one` | The install has no key-encryption key yet, so it cannot make its CA. |
| `failed_precondition` with the error of the keyring | The keyring is not safe ([Sealing](/reference/sealing/index.html.md#errors)). |
| `invalid_argument` with `invalid argument: invalid name: the pool name ...` | The pool name is not a lowercase DNS label. |
| `invalid_argument` with `invalid argument: unknown server role: ...` | A role that a server cannot enroll with, or a role twice. |
| `invalid_argument` with `idempotency: invalid key: this method needs an Idempotency-Key header` | The call has no idempotency key. |
| `already_exists` with `this idempotency key made the enrollment token <token ID> already, and the install keeps no copy of its secret. Make a new token with a new idempotency key` | A retry with a key that made a token already. |
| `not_found` with `not found: no pool is named "<pool>"` | No pool has the name. |
| `invalid_argument` with `invalid argument: invalid peer address: ...` | `peer_address` is not an IPv4 address, or not a public unicast one. The rest of the message names the address and the reason, such as `10.0.0.1 is a private address. A peer address is one public IPv4 address on the uplink of the server, with no NAT`. |
| `already_exists` with `already exists: the server <id> has the peer address <address>. A peer address belongs to one server until its decommission` | Another server that is not decommissioned holds `peer_address`. A token with `server_id` can name the peer address of that server. |
| `invalid_argument` with `invalid argument: the server <id> has the peer address <address>, so its token names that address or none` | A `server_id` with a `peer_address` other than the peer address of that server. This is a build default that follows from Q91 and Q92. |
| `failed_precondition` with `failed precondition: the control process <process> knows the migrations up to <n>, and only a hopboxd that knows migration 54 checks the peer address of an enrollment token. Upgrade it first` | A `peer_address` while a live control process runs a release of the schema version below 54. Such a release enrolls the server from any address. A token without `peer_address` works. |
| `invalid_argument` with `invalid argument: the ttl of the token is <ttl>, but it must be more than 0 and at most 24h` | A negative `ttl`, or one above 24 hours. |
| `invalid_argument` with `invalid argument: the ttl of the token is not a valid duration: ...` | The `ttl` is not a valid `google.protobuf.Duration`, such as one with seconds and nanos of opposite signs. |

### ListServers

Returns every enrolled server, oldest enrollment first. The list has no pages.

| Item | Value |
| --- | --- |
| Procedure | `/hopbox.admin.v1.ServerService/ListServers` |
| Policy action | `server.list`: `local_root`, `admin_key`, and an [admin session](#admin-sessions) |
| Idempotent option | no |
| Audit entry | action `server.list`, no target |

The request `ListServersRequest` has no fields. The response `ListServersResponse` has the field `servers`, a list of `Server`.

### ListControlProcesses

Returns the control processes that recorded themselves in the control store, live or not, oldest start first. The control role never enrolls, so `ListServers` does not show it. A process that stopped cleanly, or that ended without its stop, stays in the list with `live` set to `false` until 24 hours after its last renewal. Then the sweep of the leader deletes its row. The list has no pages. `hopboxd admin control list` calls it ([hopboxd](/reference/hopboxd/index.html.md#admin-control-list)).

| Item | Value |
| --- | --- |
| Procedure | `/hopbox.admin.v1.ServerService/ListControlProcesses` |
| Policy action | `control.list_processes`: `local_root` and `admin_key` |
| Idempotent option | no. The method has the option `(hopbox.v1.read_only)`. |
| Audit entry | action `control.list_processes`, no target |

The request `ListControlProcessesRequest` has no fields. The response `ListControlProcessesResponse` has the field `processes`, a list of `ControlProcess`.

#### ControlProcess

| Field | Type | Description |
| --- | --- | --- |
| `id` | string | The ID of the process, a UUID. Each start of the control role has a new one. |
| `server_id` | string | The ID of the enrolled server of the same `hopboxd`. Empty when no role of that `hopboxd` enrolled. |
| `hostname` | string | The host name of the server of the process. |
| `internal_url` | string | The URL of the internal listener of the process, such as `https://203.0.113.11:7443`. Empty when the listener listens on every address or on a loopback address. |
| `release` | string | The version of `hopboxd` of the process, such as `v0.2.0`. |
| `schema_version` | int32 | The number of the newest migration that the `hopboxd` of the process knows. |
| `kek_ids` | list of string | The key-encryption keys that the process can open, staged keys too. |
| `kek_current` | string | The key-encryption key with which the process seals new values. |
| `kek_fingerprints` | list of string | The fingerprint of each key file of `kek_ids`, as `<key ID>=<fingerprint>`, such as `file:2=9f86d081884c7d659a2feaa0c55ad015` ([Sealing](/reference/sealing/index.html.md#key-fingerprints)). It tells whether two key files hold the same key, and nothing of the key. The hopbox Terraform provider checks a copy of the key files against it. A control process of a release before this field reports none. |
| `maintenance` | bool | `true` when the process applies maintenance mode. |
| `drained` | bool | `true` in maintenance mode once no call, control channel, or leader job of the process runs. |
| `start_time` | Timestamp | The start of the process. |
| `renew_time` | Timestamp | The last renewal of the row. |
| `live` | bool | `true` when the row was renewed in the last 20 seconds. |
| `version` | int64 | The row version. It grows by 1 with each renewal. |
| `stop_time` | Timestamp | The clean stop of the process. Absent while it runs, and after an end without a stop. A stopped process keeps its row for 24 hours, and is never live. |

### GetServer

Returns one server.

| Item | Value |
| --- | --- |
| Procedure | `/hopbox.admin.v1.ServerService/GetServer` |
| Policy action | `server.get`: `local_root`, `admin_key`, and an [admin session](#admin-sessions) |
| Idempotent option | no |
| Audit entry | action `server.get`, target kind `server`, target ID the server ID |

The request `GetServerRequest` has the field `id`, the ID of the server. The response `GetServerResponse` has the field `server`, a `Server`.

### CordonServer

Stops new machines from landing on a server. Its machines keep running. The call sets the flag `cordoned`. A `ready` server turns `cordoned`. An `unresponsive` server stays `unresponsive`, and turns `cordoned` with its next heartbeat. A cordon of a cordoned server changes nothing, and its row version stays.

| Item | Value |
| --- | --- |
| Procedure | `/hopbox.admin.v1.ServerService/CordonServer` |
| Policy action | `server.cordon`: `local_root`, `admin_key`, and an [admin session](#admin-sessions) |
| Idempotent option | no. A second call changes nothing. |
| Audit entry | action `server.cordon`, target kind `server`, target ID the server ID, details `old_state` and `state` |

The request `CordonServerRequest` has the field `id`, the ID of the server. The response `CordonServerResponse` has the field `server`, the `Server` after the change.

### UncordonServer

Lets new machines land on a cordoned server again. The call clears the flag `cordoned`. A `cordoned` server turns `ready`. An `unresponsive` server stays `unresponsive`, and turns `ready` with its next heartbeat. An uncordon of a server that is not cordoned changes nothing. A server that drains stays cordoned until its decommission, so the call refuses it ([DrainServer](#drainserver)). The removal of the role `node` keeps its server cordoned too, so the call refuses such a server until the end or the cancel of the removal ([RemoveServerRoles](#removeserverroles)).

| Item | Value |
| --- | --- |
| Procedure | `/hopbox.admin.v1.ServerService/UncordonServer` |
| Policy action | `server.uncordon`: `local_root`, `admin_key`, and an [admin session](#admin-sessions) |
| Idempotent option | no. A second call changes nothing. |
| Audit entry | action `server.uncordon`, target kind `server`, target ID the server ID, details `old_state` and `state` |

The request `UncordonServerRequest` has the field `id`, the ID of the server. The response `UncordonServerResponse` has the field `server`, the `Server` after the change.

### SetServerLabels

Replaces the labels of a server. Placement filters use labels. The hopbox Terraform provider calls it with the labels in its Terraform files ([hopbox Terraform provider](/reference/terraform-provider/index.html.md)). From then on, the install keeps these labels, and the heartbeats of the server no longer change them. So `channel.labels` of the bootstrap config of the server has no effect after the first call ([the bootstrap config of hopboxd](/reference/hopboxd-config/index.html.md#keys)).

| Item | Value |
| --- | --- |
| Procedure | `/hopbox.admin.v1.ServerService/SetServerLabels` |
| Policy action | `server.set_labels`: `local_root`, `admin_key`, and an [admin session](#admin-sessions) |
| Idempotent option | no. A second call with the same labels changes only the row version. |
| Audit entry | action `server.set_labels`, target kind `server`, target ID the server ID. The details `old_labels` and `labels` hold `key=value` pairs, sorted by key and joined by commas. |

The request `SetServerLabelsRequest` has the fields `id`, the ID of the server, and `labels`, a map of string to string. The labels replace every label of the server. An empty map removes them all. The labels follow the rules of `channel.labels`. A server has at most 64 labels. A key has 1 to 63 characters of `a-z`, `0-9`, `.`, `_`, and `-`, and starts with a lowercase letter. A value is valid UTF-8 of at most 255 bytes, without control characters. The response `SetServerLabelsResponse` has the field `server`, the `Server` after the change.

### SetServerPeerAddress

Sets the peer address of a server: the public IPv4 address at which the other servers reach it ([Enrollment and the install PKI](/reference/enrollment/index.html.md#peer-addresses)). It is for a server that enrolled without a peer address in its token, such as the first server of an install, or a server from before peer addresses. `hopboxd install --peer-address` calls it on the first server ([hopboxd](/reference/hopboxd/index.html.md#install)). The call replaces the address of a server that has one. It refuses an address that another server holds, until that server is decommissioned. It refuses every call while a live control process runs a release before migration `0054`. The enrollment token and this call are the only ways to set a peer address. A heartbeat never changes it.

| Item | Value |
| --- | --- |
| Procedure | `/hopbox.admin.v1.ServerService/SetServerPeerAddress` |
| Policy action | `server.set_peer_address`: `local_root` and `admin_key`. The Admin area shows the peer address, but does not set it. |
| Idempotent option | no. A second call with the same address changes only the row version. |
| Audit entry | action `server.set_peer_address`, target kind `server`, target ID the server ID, details `old_peer_address` (empty for a server without one) and `peer_address` |

The request `SetServerPeerAddressRequest` has the fields `id`, the ID of the server, and `peer_address`, one public unicast IPv4 address, such as `203.0.113.12`. The response `SetServerPeerAddressResponse` has the field `server`, the `Server` after the change.

### DecommissionServer

Takes a server out of the install for good. The state turns `decommissioned`, and no call changes it again. The install refuses the certificates, the heartbeats, and the renewals of the server from then on. If the control channel of the server runs in the same control process, that process ends it after the commit. A channel in another control process ends at its next heartbeat ([the control channel](/reference/control-channel/index.html.md)).

The call works from every state but `decommissioned`. A `draining` or `shutting_down` server must be empty. The drain runs this call itself at its third empty check ([Cold move](/reference/cold-move/index.html.md#drain-of-a-node)).

The call refuses a server that still holds machines. The control store counts each machine on the server that its executor has not deleted yet, stopped machines included. The last heartbeat of the server counts its allocations. The call refuses a server that a move runs from or to. It also refuses a server that keeps the copy of a disk which a failed move left, while the machine lives on another node.

The decommission drops the old copies that the server still had to remove. It clears `cleanup_server_id` of those moves, so that a new move of those machines does not wait. The row of the drain goes too, and so does a pending removal of roles, with its stranded machines. `removed_roles` stays.

The call also refuses a server that shares its host with the last live control process, while another server is not decommissioned. The other servers need that control role. The check reads the live rows of `control_processes` ([Control store tables](/reference/control-store/index.html.md#control_processes)). A live process hosts the server when its row names the server ID, or when it is the process that takes the call and its own host has that identity in `<data_dir>/server` ([Enrollment and the install PKI](/reference/enrollment/index.html.md)). The call refuses only while no live control process runs on another host, and other servers still belong to the install. A process on another host counts only when its row names no server, or names a server that is not decommissioned. A process on a decommissioned server goes with its server, so it keeps no control plane for the others. A control process of a release before migration `0045` records no row, so the check does not see it: upgrade every control server before a decommission relies on it. So an install with a second control server decommissions the first server in any order, and the server of the last control process goes last. The check takes a transaction lock before it reads, and so does each enrollment. So two decommissions at the same time, such as in one parallel `terraform apply`, see each other, and a server that enrolls during the check counts. If the control role cannot read that identity, the call fails with `failed_precondition`. The message names the file: `cannot read the server on the host of this control role: read <data_dir>/server/identity.pem: ...`. A decommission of a decommissioned server changes nothing, and its row version stays. When Terraform destroys a server, the hopbox Terraform provider drains it, and the drain calls it ([hopbox Terraform provider](/reference/terraform-provider/index.html.md#delete)). The provider never removes the first server from its state after a decommission outside Terraform.

| Item | Value |
| --- | --- |
| Procedure | `/hopbox.admin.v1.ServerService/DecommissionServer` |
| Policy action | `server.decommission`: `local_root` and `admin_key` |
| Idempotent option | no. A second call changes nothing. |
| Audit entry | action `server.decommission`, target kind `server`, target ID the server ID, details `old_state` and `state` |

The request `DecommissionServerRequest` has the field `id`, the ID of the server. The response `DecommissionServerResponse` has the field `server`, the `Server` after the change.

### DrainServer

Moves every machine of a server cold to other nodes, then decommissions the server ([Cold move](/reference/cold-move/index.html.md#drain-of-a-node), [Drain a node](/guide/drain-a-node/index.html.md)). The call cordons the server and sets its state to `draining`, then returns at once. A leader job moves the machines, at most 2 at once. A machine that cannot move stays on the server with its disk: a running machine with no room keeps running, a machine whose move failed stays stopped. The `drain` of the `Server` names it. Once the server is empty, 3 empty checks in a row, one minute apart, decommission it. A drain never deletes a machine.

A second call starts the drain again, as a retry. It clears the stranded list, restarts `start_time`, and counts the empty checks from zero. A server that drains stays cordoned, so `UncordonServer` refuses it with `failed precondition: the server <id> drains, and stays cordoned until the drain decommissions it. A drain cannot stop. Follow it with hopboxd admin server get, and fix the reasons of its stranded machines. For a server that Terraform manages, run terraform destroy, which waits for the drain`. An `unresponsive` server keeps that state with the flag set, and turns `draining` at its next heartbeat.

| Item | Value |
| --- | --- |
| Procedure | `/hopbox.admin.v1.ServerService/DrainServer` |
| Policy action | `server.drain`: `local_root`, `admin_key`, and an [admin session](#admin-sessions) |
| Idempotent option | no. A second call starts the drain again. |
| Audit entry | action `server.drain`, target kind `server`, target ID the server ID, details `old_state`, `state`, and `retry` (`true` or `false`) |

The call refuses a server with a pending removal of roles ([RemoveServerRoles](#removeserverroles)). A removal and a drain exclude each other on one server.

The request `DrainServerRequest` has the field `id`, the ID of the server. The response `DrainServerResponse` has the field `server`, the `Server` after the change, with its `drain`.

### RemoveServerRoles

Starts the removal of the roles `edge`, `dns`, `node`, or more than one from a running server, the first server too (plan questions Q97 and Q100). The server keeps its ID and the data of its other roles. The call returns at once. `hopboxd admin server remove-role` calls it ([hopboxd](/reference/hopboxd/index.html.md#admin-server-remove-role)), and the Terraform provider calls it when a plan drops a role ([hopbox Terraform provider](/reference/terraform-provider/index.html.md#roles-of-a-running-server)).

The call refuses the role `control` with `unimplemented`. The control role never enrolls, and it goes from a server only through a second control server on the external Postgres. It refuses a removal that leaves the server no role that enrolls (`node`, `dns`, or `edge`), because its certificate needs one. A drain takes a whole server out ([DrainServer](#drainserver)). The role `dns` runs only on edge servers, so the removal of `edge` from a server with `dns` must name `dns` too.

These checks run in the transaction of the call. Each refusal gives `failed_precondition` with the reason:

1. The server runs a release whose heartbeats name the feature `remove-role` ([the control channel](/reference/control-channel/index.html.md#heartbeat)). The control role takes the features only from the heartbeats of the release that the server runs now.
2. Another server keeps each role that goes. That server is `ready`, or `cordoned` for `edge` and `dns`, and has no pending removal of the role. For `edge`, that server holds at least one floating address: its heartbeats report `edge.public_ipv4`, `edge.floating_addresses`, or `edge.public_ipv6`.
3. For `node`, another `ready` node exists. A machine without room elsewhere does not fail the call. It stays on the server, as in a drain.
4. For `edge` and `dns`, the public addresses of the install, which the zone names, hold no address of the server: its peer address, the floating addresses of its heartbeats, and the public addresses on its interfaces ([the API server](/reference/api/index.html.md)). Set the public addresses first with `hopboxd admin config set --public-address`.
5. No live control process runs a release of the schema version below 56.

The call takes the advisory lock of the server registry, so two removals run one after the other, and each sees the other. It records the removal on the server row (`removal_roles`, `removal_cordoned_before`, `removal_started_at`), and writes the audit entry `server.remove_roles_start`. The registry keeps every role until the end of the removal, so the certificate of the server stays valid and its control channel goes on ([Enrollment and the install PKI](/reference/enrollment/index.html.md#client-certificates-on-each-call)).

For `node`, the call cordons the server first. A leader job then moves each machine of the server cold to another node, as the drain does, at most 2 at once ([Cold move](/reference/cold-move/index.html.md#removal-of-the-node-role)). No decommission comes at the end. A machine that cannot move stays on the server with its disk, the removal waits, and `removal.stranded` names the machine. The removal records whether the server was cordoned before it. Its end and its cancel lift only a cordon that it set, so a cordon of the operator stays.

A removal without `node` is ready at once. A removal of `node` is ready once the server holds nothing for a move (`removal.ready`). An enrollment of the server with the roles that stay then ends the removal ([Enrollment and the install PKI](/reference/enrollment/index.html.md#remove-roles-from-an-enrolled-server)). The end adds the roles that went to `removed_roles`.

One removal runs at a time on a server. A second call for a server with a pending removal returns that removal when it names the same roles, in any order. So a retry of the Terraform provider goes on with it. A call with other roles fails. `DrainServer` refuses a server with a pending removal, and this call refuses a server that drains.

| Item | Value |
| --- | --- |
| Procedure | `/hopbox.admin.v1.ServerService/RemoveServerRoles` |
| Policy action | `server.remove_roles_start`: `local_root` and `admin_key` |
| Idempotent option | no. A second call with the same roles returns the pending removal, and writes no second audit entry. |
| Audit entry | action `server.remove_roles_start`, target kind `server`, target ID the server ID, details `roles` and `cordoned_before` (`true` or `false`) |

The request `RemoveServerRolesRequest` has the fields `id`, the ID of the server, and `roles`, the roles that go. The response `RemoveServerRolesResponse` has the field `server`, the `Server` with its `removal`.

### CancelRoleRemoval

Ends the pending removal of roles from a server that cannot finish. An example is a removal of `node` with a machine that has no room elsewhere. No new move starts. Each move from the server that started with a removal of `node` and still runs ends as `CancelMove` ends it ([the machine API](/reference/machine-api/index.html.md)). A removal of `edge` or `dns` has no moves, so its cancel ends no move, also a move that an admin started. A machine that moved stays on its new node. The registry keeps every role of the server, `removed_roles` does not change, and the stranded machines go. The cordon of the removal goes. A cordon that the server had before the removal stays.

`hopboxd admin server cancel-remove-role` calls it ([hopboxd](/reference/hopboxd/index.html.md#admin-server-cancel-remove-role)). After a cancel, put the role back in the Terraform configuration, so the next plan shows no change ([hopbox Terraform provider](/reference/terraform-provider/index.html.md#roles-of-a-running-server)).

| Item | Value |
| --- | --- |
| Procedure | `/hopbox.admin.v1.ServerService/CancelRoleRemoval` |
| Policy action | `server.remove_roles_cancel`: `local_root` and `admin_key` |
| Idempotent option | no. A second call fails, because the removal is gone. |
| Audit entry | action `server.remove_roles_cancel`, target kind `server`, target ID the server ID, details `roles` and `moved`, the count of machines that moved before the cancel. Each move that the cancel ends writes the entry `machine.cancel_move` with the detail `removal`. |

The request `CancelRoleRemovalRequest` has the field `id`, the ID of the server. The response `CancelRoleRemovalResponse` has the field `server`, the `Server` after the cancel.

### Errors of ServerService

`GetServer`, `CordonServer`, `UncordonServer`, `SetServerLabels`, `SetServerPeerAddress`, `DecommissionServer`, `DrainServer`, `RemoveServerRoles`, and `CancelRoleRemoval` check the policy first, and then the ID.

| Error | Cause |
| --- | --- |
| `invalid_argument` with `invalid argument: the server ID "<id>" is not a UUID` | The ID is not a UUID. |
| `not_found` with `not found: no server has the ID <id>` | No server has the ID. |
| `failed_precondition` with `failed precondition: the server <id> is decommissioned` | `CordonServer`, `UncordonServer`, `SetServerLabels`, `SetServerPeerAddress`, or `DrainServer` of a decommissioned server. |
| `already_exists` with `already exists: the server <id> has the peer address <address>. A peer address belongs to one server until its decommission` | `SetServerPeerAddress` with the address of another server that is not decommissioned. |
| `failed_precondition` with `failed precondition: the control process <process> knows the migrations up to <n>, and only a hopboxd that knows migration 54 checks the source address of an enrollment. Upgrade it first` | `SetServerPeerAddress` while a live control process runs a release of the schema version below 54. Such a release enrolls a token without a peer address from any address ([Enrollment and the install PKI](/reference/enrollment/index.html.md#peer-addresses)). |
| `invalid_argument` with `invalid argument: invalid peer address: ...` | `SetServerPeerAddress` with an address that is not one public unicast IPv4 address. The rest of the message names the address and the reason, such as `10.239.0.3 is in the mesh range 10.239.0.0/16`. |
| `invalid_argument` with `invalid argument: invalid label: ...` | `SetServerLabels` with a label that breaks a rule. The rest of the message names the label and the rule. |
| `failed_precondition` with `failed precondition: the server <id> holds <n> machines, such as <names>. Drain it first, so that its machines move to other nodes, or delete them` | `DecommissionServer` of a server with machines. `<names>` holds up to 5 names. Without names, the message reads `holds <n> machines. Drain it first, ...`. |
| `failed_precondition` with `failed precondition: the move <move id> of the machine <machine id> runs from or to the server <id>. Wait for its end, or cancel it` | `DecommissionServer` of a server that a move runs from or to. |
| `failed_precondition` with `failed precondition: the server <id> keeps a copy of the disk of the machine <name>, which a failed move left <state> on another node. Delete the machine, or keep the server` | `DecommissionServer` of a server that keeps the only copy of a disk after a failed move. |
| `failed_precondition` with `failed precondition: the server <id> drains, and stays cordoned until the drain decommissions it. A drain cannot stop. Follow it with hopboxd admin server get, and fix the reasons of its stranded machines. For a server that Terraform manages, run terraform destroy, which waits for the drain` | `UncordonServer` of a server that drains. |
| `failed_precondition` with `failed precondition: the control process <process> knows the migrations up to <n>, and only a hopboxd that knows migration 51 drains servers. Upgrade it first` | `DrainServer` while a live control process runs a release of the schema version below 51. |
| `failed_precondition` with `failed precondition: the server <id> has no role node, so it holds no machine to move. Decommission it instead` | `DrainServer` of a server without the role `node`. |
| `permission_denied` with `permission denied: the server <id> hosts a control process, so only root on a control server or an admin API key drains it` | `DrainServer` by an admin session of a server that hosts a control process. A new sign-in does not change it. |
| `failed_precondition` with `failed precondition: the server <id> shares its host with the last live control process of the install, and <n> other servers still belong to the install. Decommission them first, or start the control role on another server` | `DecommissionServer` of the server on the host of the last live control process, before the other servers. |
| `failed_precondition` with `failed precondition: the server <id> has a pending removal of the roles <roles>. Wait for its end, or cancel it with hopboxd admin server cancel-remove-role <id>` | `DrainServer` of a server with a pending removal, and `RemoveServerRoles` with other roles than the pending removal. |
| `failed_precondition` with `failed precondition: the removal of the node role keeps the server <id> cordoned, so that its machines move off. Cancel the removal first with hopboxd admin server cancel-remove-role <id>` | `UncordonServer` of a server with a pending removal of `node`. |
| `unimplemented` with `unimplemented: the role control cannot leave a running server in this release. The way that exists: move the control store to an external Postgres with hopboxd admin db export, add a second control server on it, and decommission this server` | `RemoveServerRoles` that names `control`. |
| `invalid_argument` with `invalid argument: name at least one role to remove: edge, dns, or node`, `invalid argument: unknown role "<role>". A removal takes edge, dns, or node`, or `invalid argument: the role <role> appears twice` | `RemoveServerRoles` without a role, with an unknown role, or with a role twice. |
| `failed_precondition` with `failed precondition: the server <id> drains, and the drain takes the whole server out` | `RemoveServerRoles` of a server that drains. |
| `failed_precondition` with `failed precondition: the server <id> has no role <role>. It has the roles <roles>` | `RemoveServerRoles` of a role that the server does not have. |
| `failed_precondition` with `failed precondition: the removal of <roles> leaves the server <id> no role that enrolls, and its certificate needs one. To take the whole server out, drain it: hopboxd admin server drain <id>` | `RemoveServerRoles` of every role of the server that enrolls. |
| `failed_precondition` with `failed precondition: the role dns runs only on edge servers, so the removal of edge from the server <id> takes dns too: name edge and dns` | `RemoveServerRoles` of `edge` alone on a server with `dns`. |
| `failed_precondition` with `failed precondition: the control process <process> knows the migrations up to <n>, and only a hopboxd that knows migration 56 removes roles from servers. Upgrade it first` | `RemoveServerRoles` while a live control process runs a release of the schema version below 56. |
| `failed_precondition` with `failed precondition: the roles <roles> cannot leave the server <id>: <reason>` | `RemoveServerRoles` that breaks a check. The reason is one of these: `the server runs hopboxd <version>, whose heartbeats name no feature remove-role. Upgrade the server first`, `no other ready server keeps the role <role>, and the install needs one`, `no other ready edge server holds a floating address, and the public addresses of the install must be floating addresses. Give an edge server a floating address first`, `no other ready node exists, so the machines of the server have nowhere to go`, or `the public addresses of the install name the address <address> of the server, so the zone still sends traffic to it. Set the public addresses without it first: hopboxd admin config set --public-address`. |
| `failed_precondition` with `failed precondition: the server <id> has no pending removal of a role` | `CancelRoleRemoval` of a server without a removal. |
| `aborted` with `conflict: moves from the server <id> kept starting during the cancel. Retry` | `CancelRoleRemoval` while moves of the removal kept starting, or the removal changed, in 3 tries. |

Example:

```sh
sudo curl -s --unix-socket /run/hopbox/admin.sock \
  -H 'Content-Type: application/json' -d '{}' \
  http://hopboxd/hopbox.admin.v1.ServerService/ListServers
```

## PoolService

`hopbox.admin.v1.PoolService` manages the pools. A pool is a named group of servers. Each enrollment token enrolls a server into one pool, and all servers of a pool have identical hardware. An operator size can keep its machines in one pool ([SizeService](#sizeservice)). The pools live in the table `pools` ([Control store tables](/reference/control-store/index.html.md#pools)). Every install has the pool `default`, and an enrollment token without a pool enrolls into it.

A pool records the hardware of its servers. The first server that enrolls into the pool records it. The enrollment of a server with other hardware fails (plan question Q61). A pool records no plan of a provider, because the operator orders each server at its provider.

The message `Pool` holds one pool:

| Field | Type | Description |
| --- | --- | --- |
| `id` | string | The ID of the pool, a UUID. |
| `name` | string | The name of the pool, a lowercase DNS label, such as `default`. |
| `ovh_plan` | string | Deprecated, and always empty. A pool records no plan of a provider. |
| `hardware` | `PoolHardware` | The hardware of the first server that enrolled into the pool. It is absent before that enrollment. A pool whose servers are all decommissioned takes the hardware of its next server ([Enrollment and the install PKI](/reference/enrollment/index.html.md#hardware-of-a-pool)). |
| `server_count` | int32 | The count of servers of the pool that are not decommissioned. |
| `size_count` | int32 | The count of sizes that name the pool and are not retired. |
| `version` | int64 | The row version. It grows by 1 with each change, such as the record of the hardware. |
| `create_time` | `google.protobuf.Timestamp` | The time of the create. |
| `update_time` | `google.protobuf.Timestamp` | The time of the last change. |

The message `PoolHardware` holds the hardware that a server reports at its enrollment ([Enrollment and the install PKI](/reference/enrollment/index.html.md)):

| Field | Type | Description |
| --- | --- | --- |
| `cpu_model` | string | The model name of the CPU, such as `AMD EPYC 4244P 6-Core Processor`. |
| `cpu_threads` | uint32 | The count of CPU threads. |
| `memory_bytes` | uint64 | The RAM that the kernel sees, in bytes. |
| `disk_bytes` | repeated uint64 | The sizes of the whole disks, in bytes, smallest first. |

### CreatePool

Adds a pool.

| Item | Value |
| --- | --- |
| Procedure | `/hopbox.admin.v1.PoolService/CreatePool` |
| Policy action | `pool.create`: `local_root`, `admin_key`, and an [admin session](#admin-sessions) |
| Idempotent option | yes. A retry with the same key returns the first result. |
| Audit entry | action `pool.create`, target kind `pool`, target ID the pool ID, details `name` |

The request `CreatePoolRequest` has these fields:

| Field | Type | Description |
| --- | --- | --- |
| `name` | string | The name of the pool: a lowercase DNS label of 1 to 63 characters that starts with a letter. No other live pool can have it. The name of a deleted pool is free. |
| `ovh_plan` | string | Deprecated. The call ignores it. |

The response `CreatePoolResponse` has the field `pool`, the new `Pool`.

### GetPool

Returns one pool.

| Item | Value |
| --- | --- |
| Procedure | `/hopbox.admin.v1.PoolService/GetPool` |
| Policy action | `pool.get`: `local_root`, `admin_key`, and an [admin session](#admin-sessions) |
| Idempotent option | no |
| Audit entry | action `pool.get`, target kind `pool`, target ID the pool ID |

The request `GetPoolRequest` has the field `name`, the name of the pool. The response `GetPoolResponse` has the field `pool`, a `Pool`.

### ListPools

Returns every pool, by name. The list has no pages, and it leaves out the deleted pools.

| Item | Value |
| --- | --- |
| Procedure | `/hopbox.admin.v1.PoolService/ListPools` |
| Policy action | `pool.list`: `local_root`, `admin_key`, and an [admin session](#admin-sessions) |
| Idempotent option | no |
| Audit entry | action `pool.list`, no target |

The request `ListPoolsRequest` has no fields. The response `ListPoolsResponse` has the field `pools`, a list of `Pool`.

### DeletePool

Deletes a pool and frees its name. The row stays in the table `pools`, because the decommissioned servers and the retired sizes of the pool still name it. A deleted pool takes no enrollment: an unused token of the pool fails ([Enrollment and the install PKI](/reference/enrollment/index.html.md#errors-of-enroll-1)).

The call refuses these pools:

- the pool `default`, because an enrollment token without a pool enrolls into it
- a pool that holds a server that is not decommissioned
- a pool that a size names that is not retired

The call locks the row of the pool. An enrollment into the pool and a create of a size that names the pool wait for the lock, so neither one can slip in during the check.

| Item | Value |
| --- | --- |
| Procedure | `/hopbox.admin.v1.PoolService/DeletePool` |
| Policy action | `pool.delete`: `local_root`, `admin_key`, and an [admin session](#admin-sessions) |
| Idempotent option | no. A second call fails with `not_found`. |
| Audit entry | action `pool.delete`, target kind `pool`, target ID the pool ID, detail `name` |

The request `DeletePoolRequest` has the field `name`, the name of the pool. The response `DeletePoolResponse` has no fields.

### Errors of PoolService

| Error | Cause |
| --- | --- |
| `invalid_argument` with `invalid argument: invalid name: the pool name ...` | `CreatePool` with a name that is not a lowercase DNS label. |
| `invalid_argument` with `idempotency: invalid key: this method needs an Idempotency-Key header` | `CreatePool` without an idempotency key. |
| `already_exists` with `already exists: a pool is named "<name>" already` | `CreatePool` with the name of a live pool. |
| `not_found` with `not found: no pool is named "<name>"` | `GetPool` or `DeletePool` of a name that no live pool has. |
| `failed_precondition` with `failed precondition: the pool default stays, because an enrollment token without a pool enrolls into it` | `DeletePool` of the pool `default`. |
| `failed_precondition` with `failed precondition: the pool <name> holds <n> servers. Decommission them first` | `DeletePool` of a pool with servers. |
| `failed_precondition` with `failed precondition: <n> sizes name the pool <name>. Retire them first` | `DeletePool` of a pool that live sizes name. |

Example:

```sh
sudo curl -s --unix-socket /run/hopbox/admin.sock \
  -H 'Content-Type: application/json' -H "Idempotency-Key: $(uuidgen)" \
  -d '{"name":"big"}' \
  http://hopboxd/hopbox.admin.v1.PoolService/CreatePool
```

## SizeService

`hopbox.admin.v1.SizeService` manages the sizes of machines. A fresh install has only the size `default` ([Domain rules](/reference/domain-rules/index.html.md#sizes)). An instance admin adds bigger sizes with this service. A user picks a size at the create of a machine, and never a pool ([the machine API](/reference/machine-api/index.html.md#createmachine)). The sizes live in the table `sizes` ([Control store tables](/reference/control-store/index.html.md#sizes)).

A size can name a pool and labels. They become the placement filters 4 and 5 of each machine of the size ([Placement](/reference/placement/index.html.md#hard-filters)), so its machines land only on the nodes of that pool that have each label (plan question Q60). A size that names no pool takes any pool.

A size never changes. A machine copies the vCPUs, the RAM, and the disk of its size at its create. To change a size, add a size with a new name and retire the old one. A retired size takes no new machine. Its machines keep it, and its name stays taken, because the machines name it. The size `default` cannot retire, because a create without a size takes it.

The message `Size` holds one size:

| Field | Type | Description |
| --- | --- | --- |
| `name` | string | The name of the size, a lowercase DNS label, such as `default`. |
| `vcpus` | int32 | The count of vCPUs, 1 to 32. |
| `memory_bytes` | int64 | The RAM, in bytes: at least 128 MiB, and a whole number of MiB. |
| `disk_bytes` | int64 | The size of the sparse disk, in bytes: at least 1 GB, and a whole number of 512-byte sectors. |
| `pool` | string | The name of the pool of the machines of the size, or empty for any pool. A retired size can name a deleted pool. |
| `labels` | map of string to string | The labels that a node must have to take a machine of the size, such as `disk=nvme`. |
| `retire_time` | `google.protobuf.Timestamp` | The time of the retirement. It is absent while the size takes new machines. |
| `version` | int64 | The row version. It grows by 1 with the retirement. |
| `create_time` | `google.protobuf.Timestamp` | The time of the create. |
| `update_time` | `google.protobuf.Timestamp` | The time of the last change. |

### CreateSize

Adds a size. The call takes a share lock on the row of the pool that the size names, so a delete of that pool waits for it.

| Item | Value |
| --- | --- |
| Procedure | `/hopbox.admin.v1.SizeService/CreateSize` |
| Policy action | `size.create`: `local_root`, `admin_key`, and an [admin session](#admin-sessions) |
| Idempotent option | yes. A retry with the same key returns the first result. |
| Audit entry | action `size.create`, target kind `size`, target ID the name of the size. The details are `vcpus`, `memory_bytes`, `disk_bytes`, `pool`, and `labels`, the `key=value` pairs sorted by key and joined by commas. |

The request `CreateSizeRequest` has these fields:

| Field | Type | Description |
| --- | --- | --- |
| `name` | string | The name of the size: the syntax of a machine name ([Domain rules](/reference/domain-rules/index.html.md#sizes)). No other size can have it, also no retired one. |
| `vcpus` | int32 | The count of vCPUs, 1 to 32. |
| `memory_bytes` | int64 | The RAM, in bytes: at least 134217728 (128 MiB), and a whole number of MiB. |
| `disk_bytes` | int64 | The size of the sparse disk, in bytes: at least 1000000000 (1 GB), and a whole number of 512-byte sectors. |
| `pool` | string | The name of a live pool, or empty for any pool. |
| `labels` | map of string to string | The labels that a node must have to take a machine of the size. They follow the rules of the labels of a server ([SetServerLabels](#setserverlabels)): at most 64, a key of 1 to 63 of `a-z`, `0-9`, `.`, `_`, and `-` that starts with a letter, and a value of at most 255 bytes of UTF-8 without control characters. |

The response `CreateSizeResponse` has the field `size`, the new `Size`.

### GetSize

Returns one size, also a retired one.

| Item | Value |
| --- | --- |
| Procedure | `/hopbox.admin.v1.SizeService/GetSize` |
| Policy action | `size.get`: `local_root`, `admin_key`, and an [admin session](#admin-sessions) |
| Idempotent option | no |
| Audit entry | action `size.get`, target kind `size`, target ID the name of the size |

The request `GetSizeRequest` has the field `name`, the name of the size. The response `GetSizeResponse` has the field `size`, a `Size`.

### ListSizes

Returns the sizes, smallest first: by vCPUs, then RAM, then name. The list has no pages. [ListMachineSizes](/reference/machine-api/index.html.md#listmachinesizes) gives the users the same list without the retired sizes, and without the pools and the labels.

| Item | Value |
| --- | --- |
| Procedure | `/hopbox.admin.v1.SizeService/ListSizes` |
| Policy action | `size.list`: `local_root`, `admin_key`, and an [admin session](#admin-sessions) |
| Idempotent option | no |
| Audit entry | action `size.list`, no target |

The request `ListSizesRequest` has the field `include_retired`, a bool. With `true`, the list also holds the retired sizes. The response `ListSizesResponse` has the field `sizes`, a list of `Size`.

### RetireSize

Stops new machines of a size. A create with the size fails from then on ([the machine API](/reference/machine-api/index.html.md#createmachine)). A create takes a share lock on the row of its size, so the retirement waits for each create that read the size before it, and a create that starts during the retirement waits and then fails. The machines of the size keep running, and keep their size. A retirement of a retired size changes nothing, and its row version stays.

| Item | Value |
| --- | --- |
| Procedure | `/hopbox.admin.v1.SizeService/RetireSize` |
| Policy action | `size.retire`: `local_root`, `admin_key`, and an [admin session](#admin-sessions) |
| Idempotent option | no. A second call changes nothing. |
| Audit entry | action `size.retire`, target kind `size`, target ID the name of the size, detail `already`: `true` for a size that was retired already |

The request `RetireSizeRequest` has the field `name`, the name of the size. The response `RetireSizeResponse` has the field `size`, the `Size` after the change.

### Errors of SizeService

| Error | Cause |
| --- | --- |
| `invalid_argument` with `invalid argument: invalid size: ...` | `CreateSize` with numbers out of bounds. The rest of the message names the field and the bound. |
| `invalid_argument` with `invalid argument: invalid name: the size name ...` or `invalid argument: invalid name: the pool name ...` | `CreateSize` with a name that is not a lowercase DNS label. |
| `invalid_argument` with `invalid argument: invalid label: ...` | `CreateSize` with a label that breaks a rule. |
| `invalid_argument` with `idempotency: invalid key: this method needs an Idempotency-Key header` | `CreateSize` without an idempotency key. |
| `not_found` with `not found: no pool is named "<pool>"` | `CreateSize` with a pool that no live pool has. |
| `already_exists` with `already exists: a size is named "<name>" already. A retired size keeps its name, so pick a new name` | `CreateSize` with the name of a size, also of a retired one. |
| `not_found` with `not found: no size is named "<name>"` | `GetSize` or `RetireSize` of an unknown name. |
| `failed_precondition` with `failed precondition: the size default stays, because a create without a size takes it` | `RetireSize` of the size `default`. |

Example:

```sh
sudo curl -s --unix-socket /run/hopbox/admin.sock \
  -H 'Content-Type: application/json' -H "Idempotency-Key: $(uuidgen)" \
  -d '{"name":"large","vcpus":8,"memoryBytes":"34359738368","diskBytes":"200000000000","pool":"big","labels":{"disk":"nvme"}}' \
  http://hopboxd/hopbox.admin.v1.SizeService/CreateSize
```

## OrgService

`hopbox.admin.v1.OrgService` lets instance admins see the metadata of every org, suspend an org, and set the usage limits of orgs. It shows no member and no resource of an org. [The org API](/reference/org-api/index.html.md) covers the calls of the members of an org. Each call that acts on one org writes its audit entry with that org, so the owners of the org see it.

The message `Org` holds the metadata of one org:

| Field | Type | Description |
| --- | --- | --- |
| `id` | string | The UUID of the org. |
| `name` | string | The name of the org. |
| `personal_account_id` | string | The UUID of the account of a personal org, or empty for another org. |
| `member_count` | int64 | The number of members. |
| `suspended` | bool | `true` while the org is suspended. |
| `suspend_time` | `google.protobuf.Timestamp` | The start of the suspension. Unset for an org that is not suspended. |
| `suspend_reason` | string | The reason of the suspension, or empty. |
| `limits` | `OrgLimits` | The limits that apply: each override, else the default of the install. |
| `limit_overrides` | `OrgLimitOverrides` | The limits of this org that replace the defaults. |
| `usage` | `OrgUsage` | What the org uses now. |
| `version` | int64 | The version of the org ([the org API](/reference/org-api/index.html.md#org)). |
| `create_time` | `google.protobuf.Timestamp` | The creation time of the org. |

`OrgLimits` and `OrgUsage` have the fields `machines` (int32), `vcpus` (int32), `memory_bytes` (int64), and `disk_bytes` (int64), as in [the org API](/reference/org-api/index.html.md#orglimits-and-orgusage). `OrgLimitOverrides` has the same fields, each one `optional`. A field that is not set takes the default of the install. A limit of 0 allows nothing.

### ListOrgs

Lists every org of the install, oldest first, one page at a time.

| Item | Value |
| --- | --- |
| Procedure | `/hopbox.admin.v1.OrgService/ListOrgs` |
| Policy action | `org.list_all`: `local_root`, `admin_key`, and an [admin session](#admin-sessions) |
| Idempotent option | no |
| Audit entry | action `org.list_all`, no org, no target |

| Field | Type | Default | Description |
| --- | --- | --- | --- |
| `page_size` | int32 | 100 | The most orgs in one response, 1 to 1000. |
| `page_token` | string | empty | The `next_page_token` of the response before. Leave it empty for the first page. |

The response `ListOrgsResponse` has the fields `orgs`, a repeated `Org`, and `next_page_token`. The token is empty on the last page. A page size out of range, or a page token that does not decode, gets `invalid_argument`.

### SuspendOrg

Suspends an org, for example to handle abuse ([the org API](/reference/org-api/index.html.md#suspension)). A suspended org stays as it is, with its first reason, and the call succeeds again without a change.

| Item | Value |
| --- | --- |
| Procedure | `/hopbox.admin.v1.OrgService/SuspendOrg` |
| Policy action | `org.suspend`: `local_root`, `admin_key`, and an [admin session](#admin-sessions) |
| Idempotent option | no. A second call changes nothing. |
| Audit entry | action `org.suspend`, org the org, target kind `org`, details `reason` and `already`. The reason is that of the suspension. `already` is `true` for an org that was suspended already. |

| Field | Type | Description |
| --- | --- | --- |
| `org_id` | string | The UUID of the org. |
| `reason` | string | The reason, which the members of the org see. At most 256 characters, without control characters. The server drops the spaces around it. |

The call stops the resources of the org in its transaction, through `OrgSuspending` ([the org API](/reference/org-api/index.html.md#what-a-member-leaves-behind)). It sets the target `stopped` on each machine of the org, except a hibernated one. Each machine that it changes gets an entry `machine.stop` with the detail `reason` (`org suspended`). The response `SuspendOrgResponse` has the field `org`, the `Org` after the change.

| Error | Code |
| --- | --- |
| `org_id` is not a UUID, or the reason breaks a rule | `invalid_argument` |
| No org has the ID: `not found: no org has the ID <id>` | `not_found` |

### UnsuspendOrg

Ends the suspension of an org. Its machines stay stopped until their users start them. An org that is not suspended stays as it is, and the call succeeds.

| Item | Value |
| --- | --- |
| Procedure | `/hopbox.admin.v1.OrgService/UnsuspendOrg` |
| Policy action | `org.unsuspend`: `local_root`, `admin_key`, and an [admin session](#admin-sessions) |
| Idempotent option | no |
| Audit entry | action `org.unsuspend`, org the org, target kind `org`, details `old_reason` and `already` (`true` for an org that was not suspended) |

The request `UnsuspendOrgRequest` has the field `org_id`. The response `UnsuspendOrgResponse` has the field `org`. The errors are those of `SuspendOrg`.

### SetOrgLimits

Replaces the limit overrides of one org. A limit that the request leaves out takes the default of the install. So a request without overrides puts the org back on the defaults. A new limit below the usage of the org stops nothing. The org then creates and starts no more machines until its usage fits.

| Item | Value |
| --- | --- |
| Procedure | `/hopbox.admin.v1.OrgService/SetOrgLimits` |
| Policy action | `org.set_limits`: `local_root`, `admin_key`, and an [admin session](#admin-sessions) |
| Idempotent option | no. A second call sets the same overrides again. |
| Audit entry | action `org.set_limits`, org the org, target kind `org`. The details are `machines`, `vcpus`, `memory_bytes`, and `disk_bytes`, then the same keys with the prefix `old_`. Each value is a number, or `default` for a limit without an override. |

| Field | Type | Description |
| --- | --- | --- |
| `org_id` | string | The UUID of the org. |
| `limit_overrides` | `OrgLimitOverrides` | The new overrides. Each field is 0 or more. |
| `expected_version` | optional int64 | The `version` of the org that the caller read. If the org has another version now, the call fails with `aborted` and changes nothing. |

The response `SetOrgLimitsResponse` has the field `org`. A limit below 0 gets `invalid_argument`, for example `invalid argument: invalid limit: the vCPU limit -1 is below 0`. An unknown org gets `not_found`.

### GetDefaultOrgLimits

Returns the limits of every org without an override. They live in the row of `install_config` ([Control store tables](/reference/control-store/index.html.md#install_config)).

| Item | Value |
| --- | --- |
| Procedure | `/hopbox.admin.v1.OrgService/GetDefaultOrgLimits` |
| Policy action | `org_limits.get_defaults`: `local_root`, `admin_key`, and an [admin session](#admin-sessions) |
| Idempotent option | no |
| Audit entry | action `org_limits.get_defaults`, no target |

The request `GetDefaultOrgLimitsRequest` has no fields. The response `GetDefaultOrgLimitsResponse` has the field `defaults`, a `DefaultOrgLimits`:

| Field | Type | Description |
| --- | --- | --- |
| `limits` | `OrgLimits` | The default limits. A fresh install has 3 machines, 6 vCPUs, 24 GiB of RAM, and 300 GB of disk. |
| `max_orgs_per_account` | int32 | The most orgs that one account owns besides its personal org ([the org API](/reference/org-api/index.html.md#createorg)). The default is 10. |
| `version` | int64 | The version of the whole install config ([GetInstallConfig](#getinstallconfig)). |
| `update_time` | `google.protobuf.Timestamp` | The time of the last change of the install config. |

### SetDefaultOrgLimits

Changes the limits of every org without an override, and returns them. A limit that the request leaves out keeps its value. A change adds 1 to the version of the install config.

| Item | Value |
| --- | --- |
| Procedure | `/hopbox.admin.v1.OrgService/SetDefaultOrgLimits` |
| Policy action | `org_limits.set_defaults`: `local_root`, `admin_key`, and an [admin session](#admin-sessions) |
| Idempotent option | no |
| Audit entry | action `org_limits.set_defaults`, no target. The details are the new limits and the old ones, as for `SetOrgLimits`, with `max_orgs_per_account` and `old_max_orgs_per_account`. |

| Field | Type | Description |
| --- | --- | --- |
| `machines` | optional int32 | The most machines of an org, 0 or more. |
| `vcpus` | optional int32 | The most vCPUs of an org, 0 or more. |
| `memory_bytes` | optional int64 | The most RAM of an org, in bytes, 0 or more. |
| `disk_bytes` | optional int64 | The most disk of an org, in bytes, 0 or more. |
| `max_orgs_per_account` | optional int32 | The most orgs that one account owns besides its personal org, 0 or more. A lower value deletes no org. It only refuses new orgs above it. |
| `expected_version` | optional int64 | The `version` of the install config that the caller read. If the install config has another version now, the call fails with `aborted` and changes nothing. |

A request that sets no limit gets `invalid_argument` with the message `invalid argument: the update sets no limit`. The response `SetDefaultOrgLimitsResponse` has the field `defaults`.

```text
$ sudo hopboxd admin org default-limits set --machines 5
machines	5
vcpus	6
memory_bytes	25769803776
disk_bytes	300000000000
max_orgs_per_account	10
version	4
update_time	2026-10-16T15:50:02Z
```

## MachineService

`hopbox.admin.v1.MachineService` lets instance admins see the machines of every org, and stop, delete, or move any of them. This also works in a suspended org, for example to handle abuse. It never opens a shell in a machine and never reads its secrets. [The machine API](/reference/machine-api/index.html.md) covers the calls of the members of an org.

The message `Machine` of this service is a machine with the facts that only instance admins see:

| Field | Type | Description |
| --- | --- | --- |
| `machine` | `hopbox.v1.Machine` | The machine as its users see it ([the machine API](/reference/machine-api/index.html.md#machine)), but without its `start_command`, which is always empty here. A start command can hold data of the org, and an instance admin sees the metadata of an org, not its contents. |
| `server_id` | string | The UUID of the node of the machine. |
| `address` | string | The IPv4 address of the machine on the network of its node, such as `10.240.0.2` ([the node network](/reference/node-network/index.html.md#addresses)). Only `ListMachines` fills it. It is empty in the responses of `StopMachine` and `DeleteMachine`. |

A call on one machine names it with `id` or `name`, as in the public API ([the machine API](/reference/machine-api/index.html.md#naming-a-machine)). The admin calls skip the role check and the suspension check of the public API.

### ListMachines

Lists the machines of every org, or of one org, one page at a time, in the order of their IDs. A machine whose current state is `deleted` is not in the list. The list holds metadata only. A machine of a deleted org stays in the list of every org until its executor deleted it. Its `org_id` is then empty.

| Item | Value |
| --- | --- |
| Procedure | `/hopbox.admin.v1.MachineService/ListMachines` |
| Policy action | `machine.list_all`: `local_root`, `admin_key`, and an [admin session](#admin-sessions) |
| Idempotent option | no |
| Audit entry | action `machine.list_all`, no target. With `org_id`, the entry has that org and the detail `org_id`. Without it, the entry has no org. |

| Field | Type | Default | Description |
| --- | --- | --- | --- |
| `org_id` | optional string | none | Lists only the machines of this org. |
| `page_size` | int32 | 100 | The most machines in one response, 1 to 1000. |
| `page_token` | string | empty | The `next_page_token` of the response before. Leave it empty for the first page. |

The response `ListMachinesResponse` has the fields `machines`, a repeated `Machine` of this service, and `next_page_token`. The token is empty on the last page. A page size out of range, a page token that does not decode, or an `org_id` that is not a UUID gets `invalid_argument`. An `org_id` that names no org gets an empty list.

### StopMachine

Sets the target of any machine to `stopped`, and waits for it for up to 60 s. It works as `StopMachine` of the public API ([the machine API](/reference/machine-api/index.html.md#stopmachine)), also in a suspended org. A hibernated machine refuses the stop with `failed_precondition` and the message `failed precondition: machine action not allowed: cannot stop a hibernated machine: wake it first`.

| Item | Value |
| --- | --- |
| Procedure | `/hopbox.admin.v1.MachineService/StopMachine` |
| Policy action | `machine.admin_stop`: `local_root`, `admin_key`, and an [admin session](#admin-sessions) |
| Idempotent option | no. A second call finds the target `stopped`, and changes nothing. |
| Audit entry | action `machine.admin_stop`, org the org of the machine, target kind `machine`, details `name` and `old_target` |

The request `StopMachineRequest` names the machine. The response `StopMachineResponse` has the fields `machine` (a `Machine` of this service), `duration`, and `done`, as in [the machine API](/reference/machine-api/index.html.md#lifecycle-calls). The call gets the default deadline of 61 s, and waits as the public call does ([the machine API](/reference/machine-api/index.html.md#the-wait)).

### DeleteMachine

Sets the target of any machine to `deleted`, and waits for it for up to 60 s. It works as `DeleteMachine` of the public API ([the machine API](/reference/machine-api/index.html.md#deletemachine)), also in a suspended org. The name and the SSH port of the machine stay reserved for its org for 7 days.

| Item | Value |
| --- | --- |
| Procedure | `/hopbox.admin.v1.MachineService/DeleteMachine` |
| Policy action | `machine.admin_delete`: `local_root`, `admin_key`, and an [admin session](#admin-sessions) |
| Idempotent option | no. A second call fails with `failed_precondition`. |
| Audit entry | action `machine.admin_delete`, org the org of the machine, target kind `machine`, details `name` |

The request `DeleteMachineRequest` names the machine. The response `DeleteMachineResponse` has the fields `machine`, `duration`, and `done`. A machine whose target is `deleted` already gets `failed_precondition` with the message `failed precondition: machine action not allowed: cannot delete a machine that is being deleted`.

```text
$ sudo hopboxd admin machine stop --name box1
id	01a1460c-1f6e-7b8a-9c42-3d5e7f8a9b10
name	box1
state	pending
target_state	stopped
duration	39.001s
done	false
```

### MoveMachine

Moves any machine cold to another node ([Cold move](/reference/cold-move/index.html.md)). The node of the machine stops it. The other node then pulls its disk, and boots it if its target is `running`. The machine keeps its name, its URL, its SSH port, and its address. The call returns at once, with the move in the state `stopping`. `ListMoves` follows the move. A move works in a suspended org too.

| Item | Value |
| --- | --- |
| Procedure | `/hopbox.admin.v1.MachineService/MoveMachine` |
| Policy action | `machine.move`: `local_root` and `admin_key` |
| Idempotent option | yes. The call needs an `Idempotency-Key` header. A retry with the same key returns the first move, and starts no second move. A second call with a new key fails with `failed_precondition` while the first move runs. |
| Audit entry | action `machine.move`, org the org of the machine, target kind `machine`, details `name`, `move`, `source`, `target`, and `old_target`. The end of the move writes a second entry with the actor `system` and the details `move` and `result`, and `error` for a failure. |

| Field | Type | Default | Description |
| --- | --- | --- | --- |
| `id`, `name` | string | | Name the machine. |
| `target_server_id` | string | empty | The UUID of the node to move the machine to. Empty lets placement pick a node. |

The response `MoveMachineResponse` has the field `move`, a `Move`. [Cold move](/reference/cold-move/index.html.md#start-of-a-move) lists the refusals. A machine that cannot move, such as a hibernated one, gets `failed_precondition`. A named node that is the node of the machine, or a `target_server_id` that is not a UUID, gets `invalid_argument`. An unknown named node gets `not_found`. When no node takes the machine, the call fails with `unavailable` and the message of [Placement](/reference/placement/index.html.md#when-no-node-has-room). While a live control process runs a `hopboxd` that does not know the moves, the call fails with `failed_precondition` ([Cold move](/reference/cold-move/index.html.md#two-releases)).

### CancelMove

Ends the cold move of a machine that runs ([Cold move](/reference/cold-move/index.html.md#pull-deadline-and-cancel)). The move ends in the state `failed`, with the error `an instance admin canceled the move`. A machine in `stopping` gets its own target back on its node. A machine in `pulling` goes back to its source node with its disk, stopped, and the target node removes what it pulled. A machine that is no longer in `moving` on the target node stays there ([Cold move](/reference/cold-move/index.html.md#a-step-that-an-older-control-process-accepted)).

| Item | Value |
| --- | --- |
| Procedure | `/hopbox.admin.v1.MachineService/CancelMove` |
| Policy action | `machine.cancel_move`: `local_root` and `admin_key` |
| Idempotent option | no. A second call fails with `failed_precondition`, because the move no longer runs. |
| Audit entry | action `machine.cancel_move`, org the org of the machine, target kind `machine`, details `name`, `move`, and `state`, the state of the move before the cancel. A move in `pulling` also gets the entry `machine.move` of a failed move, with the actor `system`. |

| Field | Type | Default | Description |
| --- | --- | --- | --- |
| `id`, `name` | string | | Name the machine. |

The response `CancelMoveResponse` has the field `move`, a `Move`. A machine without a move that runs gets `failed_precondition` with the message `the machine has no move that runs`.

### ListMoves

Lists the cold moves of one machine, or of every machine, newest first, one page at a time.

| Item | Value |
| --- | --- |
| Procedure | `/hopbox.admin.v1.MachineService/ListMoves` |
| Policy action | `machine.list_moves`: `local_root` and `admin_key` |
| Idempotent option | yes |
| Audit entry | action `machine.list_moves`, no target. With `machine_id`, the entry has the detail `machine_id`. |

| Field | Type | Default | Description |
| --- | --- | --- | --- |
| `machine_id` | optional string | none | Lists only the moves of this machine. |
| `page_size` | int32 | 100 | The most moves in one response, 1 to 1000. |
| `page_token` | string | empty | The `next_page_token` of the response before. Leave it empty for the first page. |

The response `ListMovesResponse` has the fields `moves`, a repeated `Move`, and `next_page_token`. The token is empty on the last page. A page size out of range, a page token that does not decode, or a `machine_id` that is not a UUID gets `invalid_argument`.

### Move

| Field | Type | Description |
| --- | --- | --- |
| `id` | string | The UUID of the move. |
| `machine_id` | string | The UUID of the machine. |
| `source_server_id` | string | The UUID of the node that the machine leaves. |
| `target_server_id` | string | The UUID of the node that the machine moves to. |
| `state` | `MoveState` | The state of the move. |
| `target_state` | `hopbox.v1.MachineState` | The target state of the machine before the move. The machine keeps it on the target node. |
| `error` | string | Why the move failed. |
| `cleanup_server_id` | string | The UUID of the node that still holds an old copy of the files of the machine: the source after a move that is done, the target after a move that failed. Empty once that node removed them. |
| `create_time` | Timestamp | When the move started. |
| `update_time` | Timestamp | When the move changed last. |

| `MoveState` | Meaning |
| --- | --- |
| `MOVE_STATE_STOPPING` | The source node stops the machine. |
| `MOVE_STATE_PULLING` | The machine is on the target node, which pulls its disk. |
| `MOVE_STATE_DONE` | The target node has the disk. |
| `MOVE_STATE_FAILED` | The move failed, passed its pull deadline, or an instance admin canceled it. The machine and its disk stay on the source node. |

```text
$ sudo hopboxd admin machine move --name box1
id	01a1460c-2a7b-7c11-8d23-4e5f6a7b8c9d
machine_id	01a1460c-1f6e-7b8a-9c42-3d5e7f8a9b10
source_server_id	01a14600-0a1b-7c2d-8e3f-405162738495
target_server_id	01a14600-1b2c-7d3e-8f40-516273849506
state	stopping
```

## ImageService

`hopbox.admin.v1.ImageService` manages the images that new machines start from: the metadata of image artifacts ([Artifact format](/reference/artifact-format/index.html.md)). The images live in the table `images` ([Control store tables](/reference/control-store/index.html.md#images)). A create takes the newest image of the name that it asks for, `default` by default ([the machine API](/reference/machine-api/index.html.md#createmachine)). The service holds only the metadata. The URL of an image tells the nodes where to download the compressed disk. An empty URL serves a node that has the image in its cache already.

The message `Image` holds one image:

| Field | Type | Description |
| --- | --- | --- |
| `id` | string | The UUID of the image. |
| `name` | string | The name of the image, for example `default`. |
| `digest` | string | `sha256:` and the sha256 of the bytes of the `artifact.json` of the image, in hex. |
| `arch` | string | The CPU architecture of the image, `x86_64`. |
| `kernel_sha256` | string | The sha256 of the guest kernel that the image needs, in hex. |
| `disk_size` | int64 | The size of the disk of the image, in bytes. |
| `disk_sha256` | string | The sha256 of the disk of the image, in hex. |
| `url` | string | The https URL from which nodes download the compressed disk, or empty. |
| `build_time` | `google.protobuf.Timestamp` | The time of the build: `created_at` of `artifact.json`. |
| `register_time` | `google.protobuf.Timestamp` | The time of the registration. |

### RegisterImage

Adds an image from the bytes of its `artifact.json`. The newest image of a name is the one that new machines of that name take. The control role also registers the default image of its release at each start ([hopboxd](/reference/hopboxd/index.html.md#the-default-image-of-the-release)). The same bytes a second time change nothing, and return the image that exists. A call to this method records no release on the image, so the control role never skips because of it ([hopboxd](/reference/hopboxd/index.html.md#the-default-image-of-the-release)). The control role registers the default image of its release with the release of its `hopboxd`, and skips it when the install has an image of the same name from a newer release. Two registrations of the same bytes at the same time also give one image: the second one gets `registered` `false`.

| Item | Value |
| --- | --- |
| Procedure | `/hopbox.admin.v1.ImageService/RegisterImage` |
| Policy action | `image.register`: `local_root` and `admin_key` |
| Idempotent option | no. A second call with the same bytes changes nothing. |
| Audit entry | action `image.register`, no org, target kind `image`, target ID the image ID. The details are `name`, `digest`, and `already`: `true` for an image that the install had already. |

| Field | Type | Description |
| --- | --- | --- |
| `metadata` | bytes | The bytes of the `artifact.json` of the image, unchanged. Their sha256 is the digest, so a reformat makes another image. |
| `url` | string | An https URL of the compressed disk, or empty. |

The call reads `metadata` with the rules of `internal/artifact` ([Artifact format](/reference/artifact-format/index.html.md#artifactjson)). Format version 1 knows only the kind `image`. The response `RegisterImageResponse` has these fields:

| Field | Type | Description |
| --- | --- | --- |
| `image` | `Image` | The image. |
| `registered` | bool | `true` for a new image. `false` for an image that the install had already. A second call with another `url` keeps the first URL. |

| Error | Code |
| --- | --- |
| `metadata` breaks a rule of the artifact format, for example `invalid argument: invalid artifact metadata: kind "snapshot", want "image"` | `invalid_argument` |
| `metadata` has a `format_version` other than 1: `invalid argument: unsupported artifact format version ...` | `invalid_argument` |
| `url` is not an https URL with a host: `invalid argument: the image URL "<url>" is not an https URL` | `invalid_argument` |

```text
$ sudo hopboxd admin image register bin/image/artifact.json --url https://dl.example.org/images/default/disk.raw.zst
id	01a1460a-7c21-7d4e-8f90-1a2b3c4d5e6f
name	default
digest	sha256:422d31d7a4fd5e6958162b22e3d772f71b5a1465cbd6255d253ae8c087873dc8
registered	true
```

### ListImages

Lists the images of the install, by name, newest first.

| Item | Value |
| --- | --- |
| Procedure | `/hopbox.admin.v1.ImageService/ListImages` |
| Policy action | `image.list`: `local_root`, `admin_key`, and an [admin session](#admin-sessions) |
| Idempotent option | no |
| Audit entry | action `image.list`, no target |

The request `ListImagesRequest` has no fields. The response `ListImagesResponse` has the field `images`, a repeated `Image`. The list has no pages.
