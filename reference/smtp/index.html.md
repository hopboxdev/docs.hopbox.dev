# SMTP config

> The SMTP config that sends the sign-in links: its fields, TLS modes, the sealed password, and its errors.

hopbox sends magic links and invitations to orgs by email through the SMTP server of the install ([the org API](/reference/org-api/index.html.md#invitations)). An instance admin sets the server through the admin API: `sudo hopboxd admin config smtp set` on a control server ([hopboxd](/reference/hopboxd/index.html.md#admin-config-smtp-set)), or the call `SetSMTPConfig` ([the admin API](/reference/admin-api/index.html.md#setsmtpconfig)). Without an SMTP config, the install sends no sign-in links, and `GetInstallInfo` does not list `SIGN_IN_METHOD_EMAIL_LINK`. An invitation then gives its link only to the inviter. The code is in `internal/mail`, which uses [go-mail](https://github.com/wneessen/go-mail).

The install uses the SMTP config while its mail transport is `smtp`, the default. A production install uses SMTP. A dev or test install without an SMTP server can write each mail to a file instead, with the transport `file` ([Mail](/reference/mail/index.html.md)). That transport keeps the SMTP config, and does not use it.

## Fields

The fields of the SMTP config live in the row of `install_config` ([Control store tables](/reference/control-store/index.html.md#install_config)). A change of the SMTP config adds 1 to the version of the install config.

| Field | Type | Default | Rule |
| --- | --- | --- | --- |
| host | string | none, required | A host name of at most 253 characters, or an IP address. It has no port, no spaces, and none of `/ : @ [ ]`, except in an IPv6 address. |
| port | integer | the usual port of the TLS mode | 1 to 65535. |
| TLS mode | `starttls`, `tls`, or `none` | `starttls` | See [TLS modes](#tls-modes). |
| username | string | empty | The SMTP login. A username needs a password, and a password needs a username. |
| password | string | empty | The SMTP password. It needs the TLS mode `starttls` or `tls`, so that it never travels in plain text. hopbox seals it ([Password](#password)). |
| from | string | none, required | The sender of the mail, such as `no-reply@example.org` or `hopbox <no-reply@example.org>`, in the form of RFC 5322. |

With a username, hopbox logs in with the strongest mechanism that the server offers, for example `SCRAM-SHA-256` before `PLAIN`.

## TLS modes

| Mode | Usual port | What it does |
| --- | --- | --- |
| `starttls` | 587 | Connects in plain text, then requires `STARTTLS` before it sends anything else. A server without `STARTTLS` gets no mail. |
| `tls` | 465 | Speaks TLS from the first byte (implicit TLS). |
| `none` | 25 | Never encrypts. Use it only for a relay on the same host or network. It allows no password. |

With TLS, hopbox needs TLS 1.2 or later, and checks the certificate of the server against the system roots and the host name.

## Password

hopbox seals the password with envelope encryption ([Sealing](/reference/sealing/index.html.md)). The place of the value is the install secret `smtp.password`. So a sealed password that someone copies into another column does not open. Only the column `install_config.smtp_password` holds it.

A seal needs a key-encryption key. On a new install without one, `SetSMTPConfig` with a password fails with `failed_precondition` and the hint `Run hopboxd admin kek rotate to make the first one`.

The admin API never returns the password. `GetSMTPConfig` returns only `password_set`. `SetSMTPConfig` without a password keeps the password that is set now, for example to change the sender. A kept password never goes to another server or login: a change of the host or the username needs the password again. The audit log records `password_changed`, never the password.

hopbox opens the password for each email that it sends, and keeps it in memory only for that send. A rotation of the key-encryption key leaves the password sealed by the old key, until a rewrap moves it ([Sealing](/reference/sealing/index.html.md#rewrap)).

## The DNS records of the sender

The DNS role of hopbox serves only the records of hopbox in the zone of the install. So send mail from an address in another zone that you control, such as `no-reply@example.org` for the install `example.com`. Set SPF, DKIM, and DMARC for that zone at its DNS provider. Without them, many mail servers put the sign-in links in the spam folder.

## Mail that hopbox sends

| Mail | When | Content |
| --- | --- | --- |
| Magic link | `RequestMagicLink` ([the sign-in, account, and audit API](/reference/auth-api/index.html.md#requestmagiclink)) | [Sessions, cookies, and sign-in links](/reference/sessions/index.html.md#magic-links) |
| Invitation | `CreateInvitation` ([the org API](/reference/org-api/index.html.md#createinvitation)) | The org, the role, the link, and the end of the link |
| Security notice | A call that leaves something that lasts, such as `CreateAdminKey` ([the admin API](/reference/admin-api/index.html.md#notices-to-instance-admins)) | What changed, who did it, when, the ID of the audit entry, and how to undo the change. It never holds a secret. |

Each mail is plain text in UTF-8, with a `Date` and a `Message-ID` ([Mail](/reference/mail/index.html.md#the-mail-that-hopbox-sends)). hopbox connects with a timeout of 15 s for the connection and each reply. A send waits at most 20 s in all.

## Example

```sh
printf '%s\n' "$SMTP_PASSWORD" | sudo hopboxd admin config smtp set \
  --host smtp.example.org --username hopbox@example.org --password-stdin \
  --from 'hopbox <no-reply@example.org>'
```

```text
host	smtp.example.org
port	587
tls	starttls
username	hopbox@example.org
password_set	true
from	hopbox <no-reply@example.org>
mail_transport	smtp
version	3
update_time	2026-10-16T09:09:37Z
```

## Errors

| Error | Cause |
| --- | --- |
| `invalid argument: invalid SMTP config: the host is empty` | No host. |
| `invalid argument: invalid SMTP config: the host "<host>" is not a host name or an IP address` | The host has a port, a space, or another character that a host name cannot have. |
| `invalid argument: invalid SMTP config: the port <n> is not 1 to 65535` | A bad port. |
| `invalid argument: invalid SMTP config: give both a username and a password, or neither` | A username without a password, or the reverse. |
| `invalid argument: invalid SMTP config: a password needs the TLS mode starttls or tls, so that it never travels in plain text` | A password with the TLS mode `none`. |
| `invalid argument: invalid SMTP config: the sender address "<from>" is not an email address: ...` | A bad sender. |
| `invalid argument: a new host or username needs the password again` | A new host or username without a password, while a password is set. |
| `failed precondition: the keyring has no key-encryption key to seal or open the value. Run hopboxd admin kek rotate to make the first one` | A password on an install without a key-encryption key. |
