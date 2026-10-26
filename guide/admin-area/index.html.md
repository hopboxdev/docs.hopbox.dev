# Use the Admin area

> How an instance admin uses the Admin area of the console, signs in again after 10 minutes, and handles an abuse report. Also the work that stays with root.

The Admin area is the part of the console for instance admins. An instance admin is an account that operates the whole install. In the Admin area, you see the servers, the pools, the sizes, the images, the orgs, and the machines of the install. You also manage the instance admins, the admin API keys, the config of the install, and you read the audit log of the install. [Console](/reference/console/index.html.md#admin-area) lists each page and its calls.

This guide uses the install `https://example.com`, an org `org1`, and a machine `box1`.

## What you need

- An account that is an instance admin. The first instance admin signs in with a login link ([First sign-in on a new install](/guide/first-sign-in/index.html.md)). An instance admin makes another account one on the page "Instance admins".
- A sign-in within the last 10 minutes. The next section gives the reason.

## Open the Admin area

1. Sign in at `https://example.com/` ([Use the console](/guide/console/index.html.md#sign-in)).
2. Click "Admin" at the top right. Only an instance admin sees this link.
3. The Admin area opens on the page "Servers". The tabs lead to the other pages.

A person who is not an instance admin sees no "Admin" link. The address `https://example.com/admin` shows "Page not found" to such a person. The admin API also refuses each call of such a session with `permission_denied` ([the admin API](/reference/admin-api/index.html.md#admin-sessions)).

## Sign in again after 10 minutes

The Admin area acts on the whole install, and a session lives for up to 30 days in a browser. So each call of the Admin area needs a sign-in within the last 10 minutes, the reads too. The server checks this on each call, not the console ([Sessions, cookies, and sign-in links](/reference/sessions/index.html.md#recent-sign-in)).

After 10 minutes, a page or an action of the Admin area shows "This step needs a sign-in within the last 10 min". It also shows the button "Sign in again":

1. Click "Sign in again". The sign-in page opens with the title "Sign in again".
2. Sign in with a method of your account.
3. The console opens the page of the Admin area again. Do the action again.

The new sign-in replaces the session of this browser. The sessions of your account in other browsers stay.

## Cordon a server

A cordoned server takes no new machine. Its machines keep running. Cordon a server before work on it.

1. Open the page "Servers". Each row shows the state of a server, its last heartbeat, its labels, and its count of machines. It also shows the compat class: the group of servers that can restore the memory state of each other.
2. Click "Cordon" in the row of the server. The state changes to `cordoned`.
3. After the work, click "Uncordon". The state changes to `ready`.

To set the labels of a server, click its name. Type the labels as `key=value` pairs, such as `disk=nvme, zone=a`, and click "Set labels". A size can ask for labels, and placement then puts its machines only on servers with them ([Add pools and sizes](/guide/pools-and-sizes/index.html.md)).

The Admin area does not add or remove a server. The hopbox Terraform provider does that ([Install hopbox on an OVHcloud server with Terraform](/guide/install-on-ovhcloud/index.html.md)). The Admin area has no drain button yet, because hopbox has no drain of a server yet. The button comes with the drain.

## Handle an abuse report

An abuse report names the machine `box1` in the org `org1`. You stop the machine, and you suspend the org until its owners answer:

1. Open the page "Machines". Each row shows the org, the state, the node, and the address of a machine, such as `10.240.0.2`.
2. Click "Stop" in the row of `box1`. The state changes to `stopped`.
3. Open the page "Orgs". Click "Suspend" in the row of `org1`.
4. Type a reason, such as `abuse report 1`, and click "Suspend". The members of `org1` see the reason. The org stops its machines, and creates and starts no machine until the end of the suspension.
5. When the report is settled, click "Unsuspend".

The Admin area has no shell and no file access for a machine. It never shows a secret or an env var of an org. An instance admin sees the metadata of an org, not its contents. The audit log of `org1` shows each of these actions to the owners of the org.

To change the limits of one org, click "Limits" in its row. A field that you leave empty takes the default of the install. The section "Default limits" sets those defaults.

## Make an admin API key

An admin API key lets a script or the hopbox Terraform provider call the admin API ([CLI tokens, API keys, admin API keys, and SSH keys](/reference/credentials/index.html.md#admin-api-keys)).

1. Open the page "Admin keys".
2. Type a name, such as `terraform`, pick a lifetime, and click "Create admin key".
3. Copy the key at once. It starts with `hbxadm_`. The install keeps only its hash, so the console shows the key only once.

Click "Revoke" in the row of a key to end it. Every call with the key then fails at once.

## Change the SMTP server

1. Open the page "Config", section "Email".
2. Type the host, the username, the password, and the sender, and click "Save the SMTP server".

The page never shows the password, and the API never returns it. The page says only whether a password is set. Leave the field "Password" empty to keep the password that is set now. The section "Sign-in with GitHub and Google" works the same for the client secret of each app ([Sign in with GitHub and Google](/guide/github-google-sign-in/index.html.md)).

## Change the domain

1. Open the page "Config", section "Domain".
2. Type the new domain, such as `hop.example.org`, and click "Change the domain". Point the DNS records of the new domain to the install first.
3. Click "Change" in the dialog. The API, the console, and every machine URL move to the new domain.
4. Open the console at the new domain, such as `https://hop.example.org/`, and sign in there.

The console at the old domain stops working at once. The session cookie and the `Origin` check belong to one domain ([Sessions, cookies, and sign-in links](/reference/sessions/index.html.md#calls-with-the-cookie)).

## Read the audit log

The page "Audit log" lists the actions of everyone on the install, newest first. Each action of the Admin area is there, with the actor kind `session`. Each read of the admin API writes an entry too. So a page of the Admin area adds entries each time that it loads.

## Call the admin API as the console does

The console calls the admin API with the session cookie, the `Origin` header of the install, and the header `Connect-Protocol-Version: 1`. You can make the same calls with curl on a laptop install. Sign in first, and define the function `browser`, as in [First sign-in on a new install](/guide/first-sign-in/index.html.md#sign-in-with-the-link). Then list the instance admins:

```sh
browser -d '{}' http://127.0.0.1:7080/hopbox.admin.v1.InstanceAdminService/ListInstanceAdmins
```

```json
{"admins":[{"accountId":"01a178cb-9355-7ab8-8ae3-bb81ff83b823", "emails":[{"email":"root@example.org"}], "grantTime":"2026-10-26T15:19:29.877143Z"}]}
```

A call that only root on a control server makes fails, also with a recent sign-in:

```sh
browser -d '{"email":"user1@example.org"}' http://127.0.0.1:7080/hopbox.admin.v1.LoginLinkService/CreateLoginLink
```

```json
{"code":"permission_denied","message":"permission denied: a caller of kind session may not do login_link.create"}
```

More than 10 minutes after the sign-in, each call of the Admin area fails with the detail `hopbox.v1.RecentSignInRequired`:

```json
{"code":"permission_denied","message":"permission denied: this action needs a sign-in within the last 10 minutes. Sign in again, then retry","details":[{"type":"hopbox.v1.RecentSignInRequired","value":"CgMI2AQ","debug":{"maxAge":"600s"}}]}
```

[The admin API](/reference/admin-api/index.html.md#admin-sessions) lists the calls that a session can make.

## What stays with root on a control server

Some work never goes through the Admin area. Run `sudo hopboxd admin` on a control server for it ([hopboxd](/reference/hopboxd/index.html.md#admin)):

| Work | Command |
| --- | --- |
| Maintenance mode | `hopboxd admin maintenance on` and `off` |
| The key-encryption keys | `hopboxd admin kek rotate`, `stage`, `promote`, `list`, and `rewrap` |
| The mail files of the file transport | `hopboxd admin mail list` and `show` |
| A first-admin login link | `hopboxd admin login-link` |
| An enrollment token for a new server | `hopboxd admin enrollment-token create`, or the Terraform provider |
| The register of an admin key from its hash, or of an image | `hopboxd admin admin-key register` and `hopboxd admin image register` |
| The recovery of a lost sign-in method | `hopboxd admin account sign-in-method remove` |
