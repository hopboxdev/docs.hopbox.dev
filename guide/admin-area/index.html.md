# Use the Admin area

> How an instance admin uses the Admin area of the console, signs in again after 10 minutes, handles an abuse report, approves or rejects new accounts, and sets the sign-up mode. Also the work that stays with root.

The Admin area is the part of the console for instance admins. An instance admin is an account that operates the whole install. In the Admin area, you see the servers, the pools, the sizes, the images, the accounts, the orgs, and the machines of the install. You cordon and drain nodes. You also manage the instance admins, the admin API keys, and the config of the install, and you read the audit log of the install. [Console](/reference/console/index.html.md#admin-area) lists each page and its calls.

This guide uses the install `https://example.com`, an org `org1`, a machine `box1`, and an account `user1@example.org`.

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

The Admin area does not add or remove a server. The hopbox Terraform provider does that ([Install hopbox on your servers with Terraform](/guide/install-with-terraform/index.html.md)). To take a server out of the install with its machines, drain it first ([Drain a node](/guide/drain-a-node/index.html.md)). The server page has a button "Drain" for it.

## Drain a server

A drain moves every machine of a server to other nodes, and then decommissions the server ([Drain a node](/guide/drain-a-node/index.html.md)). A running machine stops, moves, and boots again, so it loses its memory. Each machine keeps its disk, name, URL, SSH port, and address.

1. Click the name of the server `node1`. The page of the server opens.
2. Click "Drain". The dialog "Drain node1" explains the drain. It says that "Cancel drain" ends the drain and keeps the server. It also says to destroy a server that Terraform manages with Terraform, because the next refresh of Terraform drops a drained server from its state.
3. Click "Drain". The state changes to `draining`. The buttons "Cordon" and "Uncordon" disappear, because a drain keeps the server cordoned.
4. Watch the section "Drain". It shows when the drain started, the moves that run, and the empty checks, such as `1 of 3`. The page reads again every 2 s.
5. When the server holds nothing, the state changes to `shutting down`, and after 3 empty checks, to `decommissioned`.

Only a server with the role `node` has the button. The Admin area cannot drain a server that hosts a control process, because the drain ends in a decommission. Drain such a server as root on a control server, or with `terraform destroy`.

A machine that the drain cannot move shows in a table with its reason. The notice "The drain cannot go on by itself" appears when each machine left on the server is stranded. A machine without room moves once a node has room, because the drain tries it again each minute. For the other reasons, fix the reason, then click "Drain again" to try the machines again.

To keep the server, click "Cancel drain" during the drain. No new move starts, a move whose machine still stops ends, and a move whose disk copies already goes on to its end. The server turns cordoned or ready, as it was before the drain ([Drain a node](/guide/drain-a-node/index.html.md#cancel-a-drain)).

## Handle an abuse report

An abuse report names the machine `box1` in the org `org1`. You stop the machine, and you suspend the org until its owners answer:

1. Open the page "Machines". Each row shows the org, the state, the node, and the address of a machine, such as `10.240.0.2`.
2. Click "Stop" in the row of `box1`. The state changes to `stopped`.
3. Open the page "Orgs". Click "Suspend" in the row of `org1`.
4. Type a reason, such as `abuse report 1`, and click "Suspend". The members of `org1` see the reason. The org stops its machines, and creates and starts no machine until the end of the suspension.
5. When the report is settled, click "Unsuspend".

The account `user1@example.org` made `box1`. If that account caused the abuse, suspend the account too:

1. Open the page "Accounts". Each row shows an account, its emails, its ID, its state, and its tier. Choose "Active" in the state filter to see only the active accounts.
2. Click "Suspend" in the row of `user1@example.org`. The dialog "Suspend user1@example.org" says what a suspend does.
3. Type a reason, such as `abuse report 1`, and click "Suspend". The state changes to `suspended`, and the row shows the reason.

The suspend deletes nothing ([the admin API](/reference/admin-api/index.html.md#suspendaccount)). The account cannot sign in, and its sign-in page shows the reason. Its sessions and its CLI sign-ins end. Its API keys and its SSH keys stop working. The suspend also suspends the personal org of the account, and each org that the account owns alone. Their machines stop. In the other orgs of the account, its private machines stop, and its shared machines keep running.

Then read what the account did:

1. Open the page "Audit log". It shows the entries of the whole install, newest first.
2. Find the entries about the account. The column "Target" shows `account` and the full ID of the account, for example in the entry `admin_account.suspend`. The column "Actor" shows `account` and the first 8 characters of the ID of the account that acted.
3. Read the actions of the account, such as `machine.create` and `org.create`. Keep a note of what you find with the report.

Then decide. Keep the account suspended as evidence, or delete it. To give the account back its access, click "Unsuspend" in its row. The orgs that its suspend suspended become active again, but an org that you suspended on the page "Orgs" stays suspended. The machines stay stopped until their users start them.

The Admin area does not delete an account. The account of the report can be a bot of the sign-up mode `open`. To delete it, run the delete as root on a control server:

1. Run `sudo hopboxd admin account delete --account-id=ID --reason='abuse report 1'`, with the ID from the page "Accounts".
2. If it fails, delete each machine that the error names, with the `sudo hopboxd admin machine delete` command that the error gives for it. These machines are in the personal org of the account, or in an org that the account owns alone. The private machines of the account in its other orgs count too. Then run the delete of the account again.
3. The account is suspended, so the delete ends the evidence of the suspension. The command asks for the account ID a second time. Type the ID again, and press Enter.

```text
$ sudo hopboxd admin account delete --account-id=01a15c02-7d1e-7a4c-9b51-2b1c5e8f0a7d --reason='abuse report 1'
The account 01a15c02-7d1e-7a4c-9b51-2b1c5e8f0a7d is suspended. Its delete also deletes the evidence of the suspension.
Type the account ID again to delete it: 01a15c02-7d1e-7a4c-9b51-2b1c5e8f0a7d
01a15c02-7d1f-7b20-8a3e-4c1d2e3f4a5b
```

The command also deletes each org that the account owns alone. A suspended org does not stop it. The command prints the ID of each deleted org. The audit entry `admin_account.delete` keeps the reason, the IDs of the deleted orgs, and the detail `suspended`. The command deletes no audit entry of the account ([hopboxd](/reference/hopboxd/index.html.md#admin-account-delete)).

A bot often signs up again with a new email. In the sign-up mode `approval`, its new account waits for you. Reject it with a block, so its email and its GitHub or Google identity cannot sign up again ([Approve or reject new accounts](#approve-or-reject-new-accounts)):

```sh
sudo hopboxd admin account reject --account-id=01a15c05-3e4f-7a10-8b21-6c7d8e9f0a1b --block --reason='abuse report 1'
```

If the bot uses many addresses of one email domain, block the whole domain:

```sh
sudo hopboxd admin account block --email-domain=example.net --reason='abuse report 1'
```

The Admin area has no shell and no file access for a machine. It never shows a secret or an env var of an org. An instance admin sees the metadata of an org, not its contents. The audit log of `org1` shows each of these actions to the owners of the org.

To change the limits of one org, click "Limits" in its row. A field that you leave empty takes the default of the install. The section "Default limits" sets those defaults.

## Approve or reject new accounts

The sign-up mode of the install decides how a new person gets an account ([Sessions, cookies, and sign-in links](/reference/sessions/index.html.md#sign-up-modes)). The default is `approval`. In this mode, the first sign-in of a new person makes a pending account. A pending account can sign in, write a note to the instance admins, and delete itself, and nothing else. It waits for you.

Each new pending account gives each instance admin a notice in the console. The emails come as a digest: at most one email every 15 minutes, with the new pending accounts ([the admin API](/reference/admin-api/index.html.md#the-digest-of-new-pending-accounts)). The digest links to the page "Accounts" of the Admin area, where you approve or reject each account.

In the Admin area:

1. Open "Notices", and click "See the pending accounts" under "New accounts". Or open "Accounts", and choose the filter "Pending".
2. Read the row of each account: its emails, its sign-in methods, its note, its client address, and the time of its sign-up. For a GitHub sign-up, the row also shows the GitHub login. A GitHub user made less than 24 hours before the sign-up gets the tag "new GitHub user", which is a hint of a bot.
3. Click "Approve" for an account that you trust, or "Approve as limited" to give it the tier `limited`. If the install sends mail, the person gets an email.
4. Click "Reject" for an account that you do not want. In the dialog, check "Also block this email or identity" to keep its email and its GitHub or Google identity from a new sign-up. Then click "Reject".

The section "Block list of sign-ups" of the same page lists the block list. It blocks a whole email domain with "Block the domain", and removes an entry with "Remove". The page "Config" sets the sign-up mode in its section "Sign-ups".

You can also decide as root on a control server:

1. List the pending accounts with their sign-ups:

   ```sh
   sudo hopboxd admin account list --state=pending --json
   ```

   For each account, `sign_up` shows the method, the client address, and the time. It also shows the note of the person. For a GitHub sign-up, it shows the GitHub login and `github_create_time`. A GitHub user made the same day is a hint of a bot.
2. Approve an account that you trust. If the install sends mail, the person gets an email:

   ```sh
   sudo hopboxd admin account approve --account-id=01a15c05-3e4f-7a10-8b21-6c7d8e9f0a1b
   ```

   Add `--tier=limited` to approve the account with the tier `limited`.
3. Reject an account that you do not want. The reject deletes the account and its personal org. Add `--block` to keep its email and its GitHub or Google identity from a new sign-up:

   ```sh
   sudo hopboxd admin account reject --account-id=01a15c05-3e4f-7a10-8b21-6c7d8e9f0a1b --block --reason='bot sign-up'
   ```

A pending account that nobody approves expires after 30 days, and the leader deletes it. Each client address makes at most 3 new accounts a day, and the install holds at most 50 pending accounts ([Sessions, cookies, and sign-in links](/reference/sessions/index.html.md#limits-of-new-accounts)).

`sudo hopboxd admin account blocks` lists the block list, and `sudo hopboxd admin account unblock --id=ID` removes an entry ([hopboxd](/reference/hopboxd/index.html.md#admin-account-blocks)). A block stops only new accounts. An account that exists still signs in. A block of an email also stops the same address with any +tag, such as `user4+1@example.org` for `user4@example.org`. A block of a domain also stops its subdomains.

To let each new person in at once, with an active account of the tier `limited`, set the mode `open`:

```sh
sudo hopboxd admin config set --sign-up-mode=open
```

Each change of the mode gives each other instance admin a notice and an email, with the command that sets the old mode back. Set `--sign-up-mode=approval` to go back.

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

## Read the security notices

Some actions last after the session that did them:

- a new admin API key, a new instance admin, or the revoke of an instance admin
- a new sign-in method of an instance admin
- the suspend, the unsuspend, or the delete of an instance admin account
- a change of the SMTP server, of the mail transport, or of the domain
- a change of the ACME settings, of the sign-up mode, or of the GitHub or Google app

If another caller does one of them, you get a notice in the console and an email ([the admin API](/reference/admin-api/index.html.md#notices-to-instance-admins)). The email goes to each email of your account, also when the SMTP server just changed. The email of a new domain names the old domain and the new one, and has no link, because the console at the old domain no longer answers. Some mail clients make the new domain clickable. Check the change first, then type the new domain in your browser yourself and open the page "Notices". If you do one of them yourself, you get a copy of the email, and no notice in the console.

While you have an open notice, each page of the console shows a banner with the count, such as "2 security notices for the instance admins". To read and close the notices, do these steps:

1. Click "See the notices" in the banner, or open the page "Notices" of the Admin area.
2. Read each notice. It says what changed, who did it, when, and the ID of its audit entry.
3. If nobody that you trust made the change, do what the notice says. For example, revoke the new key on the page "Admin keys", or revoke the role on the page "Instance admins".
4. Click "Dismiss" on a notice. With two or more notices, the page also has the button "Dismiss all". A dismissed notice leaves your list. The audit log keeps the action.

A dismiss changes only your own list. Each other instance admin has a list of their own.

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
| The delete of another account | `hopboxd admin account delete` |
