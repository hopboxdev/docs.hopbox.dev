# First sign-in on a new install

> How the first instance admin signs in to a new install with a login link, reads the audit log, and turns on magic links.

A new install has no SMTP config and no OAuth app, and hopbox has no passwords. So the first instance admin signs in through a one-time login link that root prints on a control server. This guide signs in that way on a laptop. Then it reads the account and the audit log, and sets up magic links for everyone else.

This guide calls the API with curl, in the same way as the console does, so it shows each call. [Use the console](/guide/console/index.html.md) signs in with the same link in a browser.

## What you need

- `hopboxd` with the control role on a laptop, as in [Run the control role on a laptop](/guide/run-hopboxd/index.html.md). This guide uses its bootstrap config `/tmp/hb/hopboxd.toml` and the API on `127.0.0.1:7080`.
- curl.
- For magic links: an SMTP server. A local mail catcher such as [Mailpit](https://mailpit.axllent.org) works, on `127.0.0.1:1025`. Without one, use the file mail transport, as in [Magic links without an SMTP server](#magic-links-without-an-smtp-server).

Start `hopboxd`, and leave it running:

```sh
./bin/darwin_arm64/hopboxd run --config /tmp/hb/hopboxd.toml
```

In a second terminal, define a short name for the admin command:

```sh
admin() { ./bin/darwin_arm64/hopboxd admin --config /tmp/hb/hopboxd.toml "$@"; }
```

On a server, the same commands run as root: `sudo hopboxd admin ...`.

## Set the domain

A link points at the domain of the install, so the install needs one first:

```sh
admin config set --domain example.com
```

```text
domain	example.com
public_addresses	-
acme_directory_url	https://acme-v02.api.letsencrypt.org/directory
acme_email	-
acme_terms_agreed	false
mail_transport	smtp
version	2
update_time	2026-10-16T09:09:15Z
```

Without a domain, `login-link` fails with `failed_precondition`.

## Print the login link

```sh
admin login-link --email root@example.org
```

```text
https://example.com/auth/login-link#token=tOzJ8YQyibHutrU34tIWLHnGd5XlzYs4WoIBXgSDh5I
The link signs in root@example.org once, until 2026-10-16T09:24:15Z.
```

The link works once, for 15 minutes. Treat it as a password until then. The audit log records that root made it, but never the token ([Sessions, cookies, and sign-in links](/reference/sessions/index.html.md#first-admin-login-links)).

On an install with an edge, you open this link in a browser ([Use the console](/guide/console/index.html.md#sign-in)). The console page reads the token after `#token=` and calls `VerifyLoginLink`. Here, you make that call yourself.

## Sign in with the link

Put the token in a shell variable, and call `VerifyLoginLink`:

```sh
TOKEN=tOzJ8YQyibHutrU34tIWLHnGd5XlzYs4WoIBXgSDh5I
curl -s -D /tmp/hb/headers -X POST -H 'Content-Type: application/json' \
  -H 'Connect-Protocol-Version: 1' -H 'Origin: https://example.com' \
  -d "{\"token\":\"$TOKEN\"}" \
  http://127.0.0.1:7080/hopbox.v1.AuthService/VerifyLoginLink
```

```json
{"account":{"id":"01a143f9-1cee-7ec0-9238-7976abaacfd6", "emails":[{"email":"root@example.org"}], "instanceAdmin":true, "personalOrgId":"01a143f9-1cee-7ec1-a648-71fea47ae54f", "version":"1", "createTime":"2026-10-16T09:09:21.774252Z"}, "newAccount":true, "sessionExpireTime":"2026-11-15T09:09:21.774252Z"}
```

The link created your account, made it an instance admin, and gave it a personal org. Your email stays unverified. So a later magic link or GitHub sign-in with the same email reaches another account ([Sessions, cookies, and sign-in links](/reference/sessions/index.html.md#what-a-sign-in-does)). To sign in to this account with a magic link, link your email from the session ([Link your email to your account](#link-your-email-to-your-account)).

The two headers `Origin` and `Connect-Protocol-Version` are what the console sends. Without them, the call fails with `permission_denied`, so no page of another site can sign a browser in. Try the link a second time, and it fails with `unauthenticated`: it works once.

The response sets the session cookie. The cookie has the `Secure` flag, so curl does not send it back over plain HTTP. Copy it from the saved headers:

```sh
COOKIE=$(sed -n 's/^Set-Cookie: \(__Host-hopbox_session=[^;]*\).*/\1/p' /tmp/hb/headers)
browser() {
  curl -s -X POST -H 'Content-Type: application/json' -H 'Connect-Protocol-Version: 1' \
    -H 'Origin: https://example.com' -H "Cookie: $COOKIE" "$@"
}
```

The session lasts 30 days. After 7 days without a call, it ends before that ([Sessions, cookies, and sign-in links](/reference/sessions/index.html.md#lifetimes)).

## Read your account

```sh
browser -d '{}' http://127.0.0.1:7080/hopbox.v1.AccountService/GetAccount
```

```json
{"account":{"id":"01a143f9-1cee-7ec0-9238-7976abaacfd6", "emails":[{"email":"root@example.org"}], "instanceAdmin":true, "personalOrgId":"01a143f9-1cee-7ec1-a648-71fea47ae54f", "version":"1", "createTime":"2026-10-16T09:09:21.774252Z"}}
```

The same call from another origin fails. A page that a machine serves, such as `https://box1.example.com`, is another origin too:

```sh
curl -s -X POST -H 'Content-Type: application/json' -H 'Connect-Protocol-Version: 1' \
  -H 'Origin: https://evil.example' -H "Cookie: $COOKIE" -d '{}' \
  http://127.0.0.1:7080/hopbox.v1.AccountService/GetAccount
```

```json
{"code":"unauthenticated","message":"the credential is not valid"}
```

## Read the audit log of the install

An instance admin reads the audit log of the whole install:

```sh
browser -d '{"wholeInstall":true}' http://127.0.0.1:7080/hopbox.v1.AuditService/ListAuditEntries
```

The newest entries come first: `instance_admin.grant`, `auth.sign_in`, and `account.create` by your session, then `login_link.create` and `install_config.update` by `local_root` ([the sign-in, account, and audit API](/reference/auth-api/index.html.md#listauditentries)).

## Turn on magic links

Magic links let everyone else sign up and sign in by email. They need an SMTP config ([SMTP config](/reference/smtp/index.html.md)). A password needs a key-encryption key to seal it, so make the first one now:

```sh
admin kek rotate
```

```text
file:1
```

For a local mail catcher without TLS and without a login:

```sh
admin config smtp set --host 127.0.0.1 --port 1025 --tls none --from 'hopbox <no-reply@example.org>'
```

For a real SMTP server, pass the password on standard input, so it never shows in the process list:

```sh
printf '%s\n' "$SMTP_PASSWORD" | admin config smtp set \
  --host smtp.example.org --username hopbox@example.org --password-stdin \
  --from 'hopbox <no-reply@example.org>'
```

The install now lists the email link as a sign-in method:

```sh
curl -s --http2-prior-knowledge \
  'http://127.0.0.1:7080/hopbox.v1.InstallService/GetInstallInfo?encoding=json&message=%7B%7D'
```

```json
{"domain":"example.com", "signInMethods":["SIGN_IN_METHOD_EMAIL_LINK"], "version":"v0.0.0-20261016090852-6a5e58e824e1"}
```

Ask for a link:

```sh
curl -s -X POST -H 'Content-Type: application/json' -d '{"email":"ann@example.org"}' \
  http://127.0.0.1:7080/hopbox.v1.AuthService/RequestMagicLink
```

```json
{}
```

The email holds a link of the form `https://example.com/auth/email#token=<token>`. Sign in with its token as above, with `VerifyMagicLink` in place of `VerifyLoginLink`. The first sign-in creates the account of Ann and her personal org. An email gets at most 5 links an hour ([Sessions, cookies, and sign-in links](/reference/sessions/index.html.md#rate-limits)). If the SMTP server fails, the call gets `unavailable`, and the log of `hopboxd` has the record `the magic link email failed` with the cause.

## Magic links without an SMTP server

A dev or test install often has no SMTP server. The file mail transport then writes each mail to a file on the server, and sends nothing ([Mail](/reference/mail/index.html.md)). A production install uses SMTP, because with files no person gets a mail.

Turn on the file transport. It needs no SMTP config:

```sh
admin config set --mail-transport file
```

The output has the line `mail_transport	file`. `hopboxd` logs `mail goes to files, not to recipients` at level warn, and again with each mail.

Ask for a link, as above:

```sh
curl -s -X POST -H 'Content-Type: application/json' -d '{"email":"ann@example.org"}' \
  http://127.0.0.1:7080/hopbox.v1.AuthService/RequestMagicLink
```

List the mail files of the server, newest first:

```sh
admin mail list
```

```text
20261016T091530.120044000Z-5cb912a8d0bb45d7.eml	2026-10-16T09:15:30Z	<ann@example.org>	Sign in to example.com
```

Print the newest file. The text has the link whole, on one line:

```sh
admin mail show
```

```text
name	20261016T091530.120044000Z-5cb912a8d0bb45d7.eml
create_time	2026-10-16T09:15:30Z
to	<ann@example.org>
subject	Sign in to example.com

Open this link to sign in to example.com:

https://example.com/auth/email#token=Zm9vYmFyYmF6cXV4cXV1eGNvcmdlZ3JhdWx0Z2FycGx5

The link works once, for 15 minutes. If you did not ask for it, you can ignore this email.
```

Sign in with its token as above, with `VerifyMagicLink` in place of `VerifyLoginLink`. On a server, run `sudo hopboxd admin mail show` on the control server that took the request. Only root on that server reads the files, which live in `<data_dir>/mail` with mode `0700`.

When the install gets an SMTP server, set its config, then switch back:

```sh
admin config set --mail-transport smtp
```

## Link your email to your account

The email of your login-link account is unverified, so it links no magic link. Link it from your session instead ([Sessions, cookies, and sign-in links](/reference/sessions/index.html.md#link-a-new-sign-in-method)). The link needs a sign-in within the last 10 minutes. If your sign-in is older, the call fails with `permission_denied`. Then print a new login link, and sign in with it again as in [Sign in with the link](#sign-in-with-the-link).

Ask for a link of your email, with the session cookie and `"link":true`. This example uses the file transport of [Magic links without an SMTP server](#magic-links-without-an-smtp-server), so `admin mail show` prints the mail:

```sh
browser -d '{"email":"root@example.org","link":true}' \
  http://127.0.0.1:7080/hopbox.v1.AuthService/RequestMagicLink
admin mail show
```

```text
name	20261016T091842.551203000Z-0b2e6f8c1d3a4e57.eml
create_time	2026-10-16T09:18:42Z
to	<root@example.org>
subject	Add your email to your account on example.com

Open this link to add root@example.org as a sign-in method of your account on example.com:

https://example.com/auth/email#token=cXV1eGNvcmdlZ3JhdWx0Z2FycGx5d2FsZG9mcmVkcGx1

Open it in the browser where you asked for it. The link works once, for 15 minutes. If you did not ask for it, you can ignore this email.
```

Use the token with the same session cookie:

```sh
TOKEN=cXV1eGNvcmdlZ3JhdWx0Z2FycGx5d2FsZG9mcmVkcGx1
browser -d "{\"token\":\"$TOKEN\"}" http://127.0.0.1:7080/hopbox.v1.AuthService/VerifyMagicLink
```

```json
{"account":{"id":"01a143f9-1cee-7ec0-9238-7976abaacfd6", "emails":[{"email":"root@example.org", "verified":true, "verifyTime":"2026-10-16T09:19:03.402871Z"}], "instanceAdmin":true, "personalOrgId":"01a143f9-1cee-7ec1-a648-71fea47ae54f", "version":"1", "createTime":"2026-10-16T09:09:21.774252Z"}, "linked":true}
```

The response has `linked` and no `Set-Cookie`, so your session stays. Your account now has the email link as a sign-in method, and the email is verified. A magic link to `root@example.org` now signs in to this account. Without the session cookie, the call fails with `permission_denied`, and the link stays unused: only the session that asked for the link can use it.

## Keep a second instance admin

An install keeps at least one instance admin, so `admin instance-admin revoke` refuses the last one. Make a second admin first, for example the account of Ann, by her verified email:

```sh
admin instance-admin grant --email ann@example.org
admin instance-admin list
```

```text
01a143f9-1cee-7ec0-9238-7976abaacfd6	root@example.org	2026-10-16T09:09:21Z
01a14411-5a20-7d3e-9f0e-0c6c2b3e4a51	ann@example.org	2026-10-16T09:20:02Z
```

Root on a control server can always print a new login link, or grant the role again. So a lost session never locks the operator out.

## Sign out

```sh
browser -d '{}' http://127.0.0.1:7080/hopbox.v1.AuthService/SignOut
```

The response clears the cookie, and the session ends. The next call with the old cookie runs as an anonymous caller, and a call that needs a credential gets `unauthenticated`.

## Next

To let people sign up and sign in with GitHub or Google, register an OAuth app at each provider, as in [Sign in with GitHub and Google](/guide/github-google-sign-in/index.html.md).
