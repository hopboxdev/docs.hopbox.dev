# Mail

> The mail transports smtp and file, the mail that hopbox sends, and the file transport of dev installs: its folder, file names, bounds, logs, and how to read the mail.

hopbox sends two kinds of mail: magic links and invitations to orgs. The mail transport of the install says how hopbox delivers each mail. A production install uses the SMTP transport, which sends each mail through the SMTP server of the install ([SMTP config](/reference/smtp/index.html.md)). A dev or test install without an SMTP server can use the file transport. That transport writes each mail to a file on the server and sends nothing. The code is in `internal/mail`.

## Mail transports

The transport lives in the column `install_config.mail_transport` ([Control store tables](/reference/control-store/index.html.md#install_config)). An instance admin sets it with `sudo hopboxd admin config set --mail-transport` ([hopboxd](/reference/hopboxd/index.html.md#admin-config-set)), or with the field `mail_transport` of `UpdateInstallConfig` ([the admin API](/reference/admin-api/index.html.md#updateinstallconfig)).

| Transport | Value in the admin API | What it does |
| --- | --- | --- |
| `smtp` | `MAIL_TRANSPORT_SMTP` | Sends each mail through the SMTP server of the install. It is the default. Without an SMTP config, the install sends no mail. |
| `file` | `MAIL_TRANSPORT_FILE` | Writes each mail to a file in the mail folder of the control server that handles the call. Nothing goes over the network. It needs no SMTP config. |

A change of the transport keeps the SMTP config. So a switch back to `smtp` needs no new config. A change adds 1 to the version of the install config.

The transport shows in these places:

| Place | What it shows |
| --- | --- |
| `hopboxd admin config get` and `GetInstallConfig` | The line `mail_transport` and the field `mail_transport` ([the admin API](/reference/admin-api/index.html.md#installconfigservice)). |
| `hopboxd admin config smtp get` and `GetSMTPConfig` | The line `mail_transport` and the field `mail_transport`, so an operator sees that the install does not use the SMTP server. |
| `GetInstallInfo` | `mail_to_files` is `true` with the file transport ([the API server](/reference/api/index.html.md#getinstallinfo)). Any caller can read it. |
| The sign-in page of the console | After a request for a magic link, the page says that the install writes each email to a file, and names `sudo hopboxd admin mail show` ([Console](/reference/console/index.html.md)). |
| The log of `hopboxd` | A warning at the start of the control role, at each update of the install config, and with each mail ([Logs](#logs)). |

## The mail that hopbox sends

| Mail | When | Content |
| --- | --- | --- |
| Magic link | `RequestMagicLink` ([the sign-in, account, and audit API](/reference/auth-api/index.html.md#requestmagiclink)) | [Sessions, cookies, and sign-in links](/reference/sessions/index.html.md#magic-links) |
| Invitation | `CreateInvitation` ([the org API](/reference/org-api/index.html.md#createinvitation)) | The org, the role, the link, and the end of the link |

Each transport delivers the same message: plain text in UTF-8, in quoted-printable. Its headers are `Date`, `MIME-Version`, `Message-ID`, `Subject`, `From`, `To`, `Content-Transfer-Encoding`, and `Content-Type`. The sender is the `from` of the SMTP config. With the file transport and no SMTP config, the sender is `hopbox <no-reply@<domain>>`, with the domain of the install. The `From` header then reads `"hopbox" <no-reply@<domain>>`, because go-mail quotes the name.

Quoted-printable breaks a line after 76 characters with a `=` at its end, and writes each `=` as `=3D`. A sign-in link is longer than that. So in the raw message, the link is on two lines, and its `#token=` reads `#token=3D`. A mail client undoes this. `hopboxd admin mail show` does too.

## The file transport

### The mail folder

Each control server has its own mail folder, `<data_dir>/mail`, which is `/var/lib/hopbox/mail` by default ([the bootstrap config of hopboxd](/reference/hopboxd-config/index.html.md#keys)). The control server that handles a call writes its mail there. On an install with several control servers, the mail of one person can be on any of them. Read the folder of each.

The files hold sign-in links and invitation links, which are credentials. So the folder and its files have these rules:

| Item | Rule |
| --- | --- |
| Folder | Mode `0700`, owned by the user that runs `hopboxd`, which is root on a server. The first mail makes the folder. If the folder has another mode, the next access sets `0700` again: a write, a list, a read, or a prune. |
| File | Mode `0600`, owned by the same user. |
| Folder that is a symlink | The write fails, and so does the call that sent the mail. |
| Folder of another user | The write fails. |

No API call returns the folder to a person, a session, an API key, or an admin API key. Only root on the control server reads it, through the admin socket ([Read the mail](#read-the-mail)).

### File names

Each file is named `<time>-<random>.eml`, for example `20261023T094531.606059000Z-5cb912a8d0bb45d7.eml`. The time is the UTC time of the write, to the nanosecond. The random part is 16 hex digits, 64 random bits. So the names sort by time, and two writes in the same nanosecond still get two names.

### Atomic writes

`hopboxd` writes each mail to the temp file `.tmp-<name>` in the folder, with mode `0600`. It syncs the file to disk, then renames it to `<name>`. So a reader sees a whole file or no file. A list never shows a temp file. A temp file older than 1 hour is from a crash, and the next prune removes it.

### Bounds

A prune keeps the newest 1000 files and removes any file older than 7 days. 7 days is the lifetime of an invitation, the longest link that hopbox mails. A prune runs after each write, at each `ListMailFiles`, and at the start of the control role. These bounds are fixed. No setting changes them.

### Logs

`hopboxd` logs `mail goes to files, not to recipients` at level warn in these cases:

| When | Attribute |
| --- | --- |
| At the start of the control role, while the transport is `file` | `dir`: the mail folder |
| After each `UpdateInstallConfig` that leaves the transport at `file` | `dir` |
| With each mail | `file`: the path of the new file |

The log never holds the content of a mail, its link, or its token. A failed prune logs `the prune of the mail folder failed` at level error, and the mail still counts as written. Some files of the folder do not read, such as a symlink that leads out of the folder. The list skips each one, and logs `a mail file does not read, so the list skips it` at level error, with `file` and `err`.

### Read the mail

Root on the control server reads the mail with two commands. They call `MailService` through the admin socket ([the admin API](/reference/admin-api/index.html.md#mailservice)).

```sh
sudo hopboxd admin mail list
sudo hopboxd admin mail show [NAME] [--raw]
```

`admin mail list` prints the files, newest first ([hopboxd](/reference/hopboxd/index.html.md#admin-mail-list)). `admin mail show` prints one file, the newest by default, with the decoded text, so the link is whole on one line ([hopboxd](/reference/hopboxd/index.html.md#admin-mail-show)). The audit log records each read with the action `mail_file.get` and the name of the file.

`sudo ls /var/lib/hopbox/mail` and `sudo cat` work too. But `cat` shows the raw message, where the link breaks over two lines.

### Failures

If a write fails, `RequestMagicLink` fails with `unavailable: the server failed to send the email. Retry later`, as it does after a failed SMTP send. `CreateInvitation` still returns the link, with `email_sent` set to `false`. The log has the error.

### Invitations

With the file transport, `CreateInvitation` writes the mail, but returns `email_sent` set to `false`, because the mail reached nobody ([the org API](/reference/org-api/index.html.md#createinvitation)). The inviter passes the link on.

## Example

On a dev install without an SMTP server, turn on the file transport:

```text
$ sudo hopboxd admin config set --mail-transport file
domain	dev.example.org
public_addresses	203.0.113.10
acme_directory_url	https://acme-v02.api.letsencrypt.org/directory
acme_email	-
acme_terms_agreed	true
mail_transport	file
version	7
update_time	2026-10-23T09:40:00Z
```

Ask for a magic link on the sign-in page of the console, then read it:

```text
$ sudo hopboxd admin mail list
20261023T094531.606059000Z-5cb912a8d0bb45d7.eml	2026-10-23T09:45:31Z	<ann@example.org>	Sign in to dev.example.org
$ sudo hopboxd admin mail show
name	20261023T094531.606059000Z-5cb912a8d0bb45d7.eml
create_time	2026-10-23T09:45:31Z
to	<ann@example.org>
subject	Sign in to dev.example.org

Open this link to sign in to dev.example.org:

https://dev.example.org/auth/email#token=Zm9vYmFyYmF6cXV4cXV1eGNvcmdlZ3JhdWx0Z2FycGx5

The link works once, for 15 minutes. If you did not ask for it, you can ignore this email.
```

Turn the file transport off again. The SMTP config is still there:

```sh
sudo hopboxd admin config set --mail-transport smtp
```

## Errors

| Error | Cause |
| --- | --- |
| `--mail-transport "<value>" is not smtp or file` | `hopboxd admin config set` got another value. |
| `invalid_argument: invalid argument: the mail transport <n> is not smtp or file` | `UpdateInstallConfig` with `MAIL_TRANSPORT_UNSPECIFIED` or an unknown value. |
| `invalid_argument: invalid argument: the name is not the name of a mail file, such as 20261023T091530.123456789Z-0123456789abcdef.eml: "<name>"` | `GetMailFile` with a name that no mail file has, such as a path. |
| `not_found: not found: the mail file <name> is not in <dir>. A prune removes a file after 7 days, or when 1000 newer files exist` | `GetMailFile` with a name that is not in the folder. |
| `permission_denied: permission denied: a caller of kind admin_key may not do mail_file.list` | A call of `MailService` with another credential than the admin socket, here an admin API key. |
| `mail: the mail folder <dir> is a symlink` | In the log, after a write. Remove the symlink. |
| `mail: the mail folder <dir> belongs to uid <n>, and hopboxd runs as uid <m>` | In the log, after a write. Give the folder to the user of `hopboxd`. |
