# Console

> The console: its pages and their calls, its headers, its build, and its tests.

The console is the web app of an install. People use it in a browser to sign in, to manage their machines, and to open a shell in a machine. It is a single-page app in React and TypeScript, and `hopboxd` holds its build. This page gives its pages, its calls, its headers, its build, and its tests. The code is in `web/apps/console` and `internal/console`. [Use the console](/guide/console/index.html.md) shows how to use it.

The first console holds the parts of phase 1:

- sign-in
- the list of machines, with the live state of each one
- the create, stop, start, and delete actions
- the switch that makes a URL public
- the web terminal

It also has the pages that the links of the API open, the account page, and the pages of an org. Phase 2 adds the Admin area for instance admins ([Admin area](#admin-area)).

## Where hopboxd serves it

The API listener of the control role serves the console at the domain of the install, next to the API ([the API server](/reference/api/index.html.md#listener)). Each public handler of the control role holds the console, so the console also answers behind the edge. So the console calls the API from its own origin, and the browser needs no CORS rules.

The console takes every path that no API route takes:

| Request | Who answers |
| --- | --- |
| `POST /hopbox.v1.<Service>/<Method>` and the other API calls | the API |
| `GET /auth/github/start`, `/auth/github/callback`, `/auth/google/start`, `/auth/google/callback` | the sign-in with GitHub and Google ([Sign-in with GitHub and Google](/reference/oauth/index.html.md#endpoints)) |
| `GET /api/ws/exec` | the WebSocket endpoint of exec ([the machine API](/reference/machine-api/index.html.md)) |
| any other `GET` or `HEAD` request whose first path segment has no dot | the console |
| any other request, such as `GET /hopbox.v1.NoSuchService/Get` or `GET /index.html` | the API, with `404 Not Found`, so a Connect or gRPC client reads an unknown procedure |

The console needs HTTPS at `https://<domain>`. The session cookie has the `__Host-` prefix and the `Secure` flag, and each call with the cookie must carry the `Origin` header `https://<domain>` ([Sessions, cookies, and sign-in links](/reference/sessions/index.html.md#calls-with-the-cookie)). Thus the console works only behind a front that ends TLS for the domain of the install, such as the edge. Over plain HTTP, the pages load, but no sign-in works.

## Pages

A page that needs a session sends a browser without one to `/signin?return_to=<path>`, and the sign-in then returns to the page.

| Path | Session | What it does | Calls |
| --- | --- | --- | --- |
| `/signin?return_to=<path>` | no | Shows the sign-in methods of the install. A browser with a live session goes on to `return_to`, or `/`. With `again=1`, the page stays for a live session too, with the title "Sign in again" ([Sign-in](#sign-in)). | `GetInstallInfo`, `GetAccount`, `RequestMagicLink` |
| `/auth/email#token=<token>` | no | The page of a magic link ([Sessions, cookies, and sign-in links](/reference/sessions/index.html.md#magic-links)). It signs in with the token, then opens the saved return path, or `/`. A link that adds an email to the account of the session opens `/account` instead ([Sessions, cookies, and sign-in links](/reference/sessions/index.html.md#link-a-new-sign-in-method)). If the call fails, the page stays, shows the error, and links to `/signin`. For a suspended account, the error says `Permission denied: an instance admin suspended this account, so it cannot sign in. The reason: <reason>`, with the request ID, and the page opens no console ([Sessions, cookies, and sign-in links](/reference/sessions/index.html.md#what-a-sign-in-does)). | `VerifyMagicLink` |
| `/auth/login-link#token=<token>` | no | The page of a first-admin login link ([Sessions, cookies, and sign-in links](/reference/sessions/index.html.md#first-admin-login-links)). It works as `/auth/email`. | `VerifyLoginLink` |
| `/invite#token=<token>` | asks for one | The page of an invitation to an org ([the org API](/reference/org-api/index.html.md#invitations)). Without a session, it shows the sign-in methods. With one, it accepts the invitation and opens the org. | `GetAccount`, `AcceptInvitation` |
| `/pending` | yes, of a pending account | The only page of an account that waits for approval by an instance admin ([Sessions, cookies, and sign-in links](/reference/sessions/index.html.md#pending-accounts)). It says "Your account waits for approval by an admin of <domain>. When it is ready, you get an email." It has the field "Note to the admins", at most 500 characters, with the button "Save the note", and shows the saved note. It also has "Sign out" and "Delete my account". Each other page of a pending account opens `/pending`, and so does each call that fails with the detail `AccountPending`. For any other account, `/pending` opens `/`. After the approval, the next page load opens the console. | `GetAccount`, `GetInstallInfo`, `SetSignUpNote`, `SignOut`, `DeleteAccount` |
| `/device?code=<user code>` | yes | Approves a sign-in of the hopbox CLI ([CLI tokens, API keys, admin API keys, and SSH keys](/reference/credentials/index.html.md)). It shows the name, the IP address, and the start of the client. It asks the person to compare the code with the code in the terminal (RFC 8628 section 5.4). It approves exactly that client. Without `code`, it asks for the code. The approval needs a sign-in within the last 10 minutes. After an older sign-in, the page says so, and its button "Sign in again" opens `/signin?again=1&return_to=/device?code=<code>`. | `GetDeviceAuthorization`, `ApproveDevice` |
| `/` | yes | Opens the last org of this browser, or the personal org. | `GetAccount`, `ListOrgs` |
| `/account` | yes | The display name, the emails, the sign-in methods, the SSH keys, and the CLI sign-ins of the account. Under the sign-in methods, it links a new method of the install ([Sessions, cookies, and sign-in links](/reference/sessions/index.html.md#link-a-new-sign-in-method)). The buttons "Link GitHub" and "Link Google" open `/auth/<provider>/start?link=1&return_to=/account`. A provider that the account holds already gets no button, because an account takes one identity of each provider. The form "Email me a link" sends a link to an email. A link needs a sign-in within the last 10 minutes. After an older sign-in, the form says so, and its button "Sign in again" opens `/signin?again=1&return_to=/account`. | `GetAccount`, `UpdateAccount`, `ListSignInMethods`, `RemoveSignInMethod`, `GetInstallInfo`, `RequestMagicLink`, `ListSSHKeys`, `AddSSHKey`, `RemoveSSHKey`, `ListCLISignIns`, `EndCLISignIn` |
| `/orgs` | yes | The orgs of the account, and a form that creates an org. | `ListOrgs`, `CreateOrg` |
| `/orgs/<org ID>` | yes | The machines of the org with their live states, the usage of the org, and the create, start, stop, and delete actions. The line of the usage ends with `Public URLs: yes` or `no`, and `Outbound TCP port 25: open` or `blocked` ([the org API](/reference/org-api/index.html.md#usage-limits)). The dialog "New machine" has an empty choice of "Size". It names the default size of the org, from `default_size` of `ListMachineSizes` ([the machine API](/reference/machine-api/index.html.md#listmachinesizes)). In the personal org of a limited account, it reads "tiny (default): 1 vCPU, 1 GiB RAM, 5 GB disk". When the size is `default`, it reads "default: <resources>". | `WatchMachines`, `GetOrgUsage`, `ListMachineSizes`, `CreateMachine`, `StartMachine`, `StopMachine`, `DeleteMachine` |
| `/orgs/<org ID>/machines/<name>` | yes | One machine: its actions, its URL with the public switch and the port, its sharing, and its details. In an org without public URLs, the switch of a private URL is off and disabled. The text under it says "The org cannot make a machine URL public. An instance admin can allow it." A public URL can still turn private ([the machine API](/reference/machine-api/index.html.md#setmachineurlvisibility)). While the limits of the org load, the switch takes a change, and the server decides. | `WatchMachines`, `GetOrgUsage`, `StartMachine`, `StopMachine`, `RebootMachine`, `DeleteMachine`, `SetMachineURLVisibility`, `SetMachineURLPort`, `ShareMachine`, `UnshareMachine` |
| `/orgs/<org ID>/machines/<name>/terminal` | yes | The web terminal ([Web terminal](#web-terminal)). | `/api/ws/exec` |
| `/orgs/<org ID>/members` | yes | The members and their roles. Owners and admins also see the invitations, and invite people. | `ListOrgMembers`, `SetOrgMemberRole`, `RemoveOrgMember`, `ListInvitations`, `CreateInvitation`, `RevokeInvitation` |
| `/orgs/<org ID>/api-keys` | yes | The API keys of the org. A new key shows its secret once. | `ListAPIKeys`, `CreateAPIKey`, `RevokeAPIKey` |
| `/orgs/<org ID>/vars` | yes | The env vars and secrets of the org. A secret shows no value. | `ListVars`, `SetVar`, `DeleteVar` |
| `/orgs/<org ID>/settings` | yes | The name, the usage and the limits of the org, and the leave and delete actions. The table of the usage also has the rows "Public URLs", `yes` or `no`, and "Outbound TCP port 25", `open` or `blocked`. | `GetOrg`, `UpdateOrg`, `GetOrgUsage`, `LeaveOrg`, `DeleteOrg` |
| `/admin/...` | yes, of an instance admin | The Admin area ([Admin area](#admin-area)). | the admin API |

Every page of an org also calls `GetOrg`. Each page shows the actions that the role of the account allows. The service layer still decides each call ([the API server](/reference/api/index.html.md)), and the page shows its error.

A form, or the question before a delete or another action that cannot be undone, opens in a modal dialog. The dialog keeps the focus inside it. The title of the dialog is its accessible name. The Cancel button, the Escape key, and a click on the dark backdrop close it. A click inside the dialog, or a drag that starts inside it, does not close it.

A path with no page shows "Page not found". A path with a file extension, or under `/assets/`, that names no file of the build gets `404` with the text `the console has no file <path>`.

## Admin area

The Admin area is the part of the console for instance admins. [Use the Admin area](/guide/admin-area/index.html.md) shows how to use it. The code is in `web/apps/console/src/admin`.

Only an instance admin sees the link "Admin" at the top right. While `admin_notices` of `GetAccount` is more than 0, each page that needs a session also shows a banner under the top bar, such as "2 notices for the instance admins". The count holds the notices of new pending accounts too. Its link "See the notices" opens `/admin/notices`. The banner reads the count with `GetAccount`, which needs no recent sign-in, so it shows also after 10 minutes. The routes under `/admin` read `instance_admin` of `GetAccount` before they load. For any other account, each of them shows "Page not found". This only hides the pages. The server decides each call: the admin API takes a session only from an instance admin with a sign-in within the last 10 minutes ([the admin API](/reference/admin-api/index.html.md#admin-sessions)).

The tabs come in this order: "Overview", "Capacity", "Machines", "Accounts", "Servers", "Pools", "Sizes", "Images", "Orgs", "Instance admins", "Admin keys", "Config", "Audit log", and "Notices". The pages Overview, Capacity, and Machines, and the section "Activity, last 24 h" of Accounts, read their call of `MonitorService` again every 60 s while the browser shows the page, and not while the tab of the browser is in the background. These calls write no audit entry ([the admin API](/reference/admin-api/index.html.md#monitorservice)). After 10 minutes, a read fails and asks for a sign-in again ([the admin API](/reference/admin-api/index.html.md#admin-sessions)). The page then keeps the last numbers, and shows the line "Sign in again to refresh" with the link "Sign in again", in place of the error. A change of the range of Capacity or of the order of Machines after that shows the error, and not the numbers of the old range or order under the new label. They show the links "More in Grafana" to the dashboards `hopbox-overview`, `hopbox-capacity`, `hopbox-machines`, and `hopbox-accounts` only while the install config has `grafana_url` ([the admin API](/reference/admin-api/index.html.md#updateinstallconfig)).

| Path | What it does | Calls |
| --- | --- | --- |
| `/admin` | Opens `/admin/overview`. | none |
| `/admin/overview` | The state of the install from the history of samples ([Metrics](/reference/metrics/index.html.md#history-of-samples)). Six tiles: the servers with the count of `ready` ones, the running machines with the stopped ones, the allocated vCPUs against the capacity, the pending accounts with the age of the oldest, the open notices of this instance admin, and the public URLs with the count of their orgs, "in 2 orgs". The section "Needs your attention" lists the machines that used 90% or more of their vCPUs over the last 30 minutes, with the link "Open machine" to the machines of their org. A machine counts only once its samples cover the 30 minutes. It also lists the pending accounts by email, each server that is not `ready` or that is `degraded`, and the open notices, each with a link to its page. The section "Last 24 h" counts the sign-ups, the new orgs and API keys, the URL changes, the URLs made public, the refusals of the rate limits and of the org limits, the API errors, and the failed executor actions, and gives the suspended orgs now. The section "Health" has a line for "Control plane", "Nodes", "Edge", and "Network". DNS and SSH get their checks with milestone O6: `ok`, `warning`, `critical`, or `unknown`, with its problems. The header gives the time of the newest sample, such as "Samples from 14:32 UTC", or "No sample yet. The history takes one every minute." | `GetOverview` |
| `/admin/capacity` | One card for each node, with a link to its server page, and its pool, roles, and state, such as "pool default · node, dns, edge · ready". The card shows the vCPUs, the RAM, and the disk that the machines reserve, each with a bar against the capacity of the node. Under each one, a line chart of the use: the CPU threads that the machines used against the CPU threads of the node, such as "Used now 41%, peak 61% at 09:10", the RAM that they hold, and the free disk. The buttons "24 h" and "1 h" pick the range. A node without a recent sample of its use says so. | `GetCapacity` |
| `/admin/servers` | The servers, with their state, their last heartbeat, their compat class, their labels, and their count of machines. The cordon and uncordon actions. While a server drains, the list reads again every 2 s. | `ListServers`, `CordonServer`, `UncordonServer` |
| `/admin/servers/<server ID>` | One server: its roles, its pool, its peer address (or `none`), its status and problems, its heartbeat, its compat class, its CPU, its capacity, its versions, and its certificate. Its cordon, its drain, and a form for its labels. While the server drains, the page reads again every 2 s. | `GetServer`, `CordonServer`, `UncordonServer`, `SetServerLabels`, `DrainServer` |
| `/admin/pools` | The pools, with their hardware, and their counts of servers and sizes. A form that creates a pool, and the delete action. | `ListPools`, `CreatePool`, `DeletePool` |
| `/admin/sizes` | The sizes, with their resources, their pool, and their labels. With "Show retired sizes", also the retired ones. A form that creates a size, and the retire action. | `ListSizes`, `ListPools`, `CreateSize`, `RetireSize` |
| `/admin/images` | The machine images, with their digest, their arch, their disk, and their times. It has no action. | `ListImages` |
| `/admin/accounts?state=<state>` | The section "Activity, last 24 h" comes first: a chart of each hour of the last 24 hours, with its total, for the sign-ups, the new orgs, the new API keys, the URL changes, and the refusals of the rate limits. A chart turns to the warning color when an hour reaches the threshold of its alert: 5 sign-ups (`HopboxSignUpBurst`), 5 new orgs or 5 new API keys (`HopboxOrgOrKeyBurst`), 20 changes of the port or the visibility of machine URLs (`HopboxURLChurn`), or 100 refusals (`HopboxRateLimitBurst`). Then the public URLs with the count of their orgs, the URLs made public in the 24 hours, the suspended orgs, and the refusals of each rate limit. Then the accounts, page by page, with a filter by state: "All states", "Active", "Suspended", or "Pending". `state=pending` opens the filter "Pending". Each row has the name of the account, its emails, and its ID. It also has a badge of its state, its tier, and the date of its creation. An instance admin has the tag "instance admin". A suspended account also shows "Reason: <reason>, since <date>". A pending account has the approve and reject actions. An active account has the suspend action, and a suspended account has the unsuspend action. Both also move to the other tier. The page also holds the block list of sign-ups ([Accounts page](#accounts-page)). | `GetAccountActivity`, `ListAccounts`, `ApproveAccount`, `RejectAccount`, `SuspendAccount`, `UnsuspendAccount`, `SetAccountTier`, `ListSignUpBlocks`, `BlockSignUp`, `UnblockSignUp` |
| `/admin/orgs` | The orgs, with their members, their usage, their limits, and their suspension, page by page. The suspend, unsuspend, and limits actions, and a form for the default limits ([Orgs page](#orgs-page)). | `ListOrgs`, `SuspendOrg`, `UnsuspendOrg`, `SetOrgLimits`, `GetDefaultOrgLimits`, `SetDefaultOrgLimits` |
| `/admin/machines?org=<org ID>` | Without `org`, the section "Top machines" comes first: the running machines that use the most, ordered by "Sort by": "CPU, last hour", "RAM", "Disk", or "Egress, 24 h". Each row has the name of the machine, with a link to the machines of its org, the org name, the email of the first owner, and the size. It also has a line chart of the CPU over 24 hours, the CPU now as a share of the vCPUs, the RAM and the disk used against the size, the egress of 24 hours with its connections, and the visibility of the URL. The CPU now is in the error color when the machine used 90% or more of its vCPUs over 30 minutes. Then the section "Every machine": the machines of every org, or of one org with `org` alone, page by page. Each row has the org, the state, the node, the address, and the size. The stop and delete actions. While a machine changes its state, the page reads the list again every 2 seconds. | `ListTopMachines` without `org`, `ListMachines`, `ListServers`, `StopMachine`, `DeleteMachine` |
| `/admin/admins` | The instance admins. A form that makes an account an instance admin by its verified email, and the revoke action. | `ListInstanceAdmins`, `GrantInstanceAdmin`, `RevokeInstanceAdmin`, `GetAccount` |
| `/admin/admin-keys` | The admin API keys. A new key shows its secret once. The revoke action. | `ListAdminKeys`, `CreateAdminKey`, `RevokeAdminKey` |
| `/admin/config` | The domain, the public addresses, the ACME settings, the mail transport, the sign-up mode, and maintenance mode. The section "Sign-ups" changes the sign-up mode: "approval by an instance admin" or "open", with the button "Save the sign-up mode" ([the admin API](/reference/admin-api/index.html.md#updateinstallconfig)). Each change sends a notice to the other instance admins. A form that changes the domain. A form for the SMTP server, and a form for the app of GitHub and of Google. | `GetInstallConfig`, `UpdateInstallConfig`, `GetSMTPConfig`, `SetSMTPConfig`, `ClearSMTPConfig`, `GetOAuthConfig`, `SetOAuthClient`, `ClearOAuthClient` |
| `/admin/audit` | The audit log of the whole install, newest first, page by page. | `ListAuditEntries` with `whole_install` |
| `/admin/notices` | The open notices of this instance admin, newest first ([the admin API](/reference/admin-api/index.html.md#notices-to-instance-admins)). The notices of the kind `sign_up` show under the heading "New accounts", with the link "See the pending accounts" to `/admin/accounts?state=pending`. The other notices show under the heading "Security notices". Each one says what changed, who did it, when, the ID of its audit entry, and how to undo the change. The button "Dismiss" of a notice, and with two or more notices the button "Dismiss all", remove notices from the list of this account only. | `ListAdminNotices`, `DismissAdminNotices`, `GetAccount` |

### Accounts page

The tab "Accounts" sits between "Machines" and "Servers". The name of an account is its display name, else its first email, else its ID. The badge names the state: `active`, `suspended`, or `pending`. The badge of a suspended account is in the error color. The page lists no machine and no org of an account.

The button "Suspend" of an active account opens the dialog "Suspend <name>". The dialog says what a suspend does: the account cannot sign in, its sessions and CLI sign-ins end, and its API keys and SSH keys stop working. Its personal org and each org that it owns alone are suspended, and their machines stop. In its other orgs, its private machines stop. Nothing is deleted. The field "Reason, which the account sees at its sign-in" takes at most 256 characters. The button "Suspend" of the dialog stays off until the reason holds text other than spaces. The button, or Enter in the field, calls `SuspendAccount` ([the admin API](/reference/admin-api/index.html.md#suspendaccount)). The server checks the reason too. An error of the call shows in the dialog. A pending account has no button.

The button "Suspend" is off in two rows, and a tooltip says why:

- The row of the last active instance admin: "This account is the last active instance admin, and an install keeps one. Make another account an instance admin first." The server refuses that suspend. The page knows the active instance admins from `ListAccounts` with the state `active`, up to 1,000 accounts. With more, it leaves the check to the server.
- The row of the account of the session: "This is your own account. Another instance admin can suspend it." A suspend of the own account ends the session of the page.

The button "Unsuspend" of a suspended account calls `UnsuspendAccount` at once, without a dialog ([the admin API](/reference/admin-api/index.html.md#unsuspendaccount)). After each of the two actions, the console reads the accounts, the orgs, and the machines again.

An active or a suspended account also has a button for its tier. A limited account has "Make full", and a full account has "Make limited". The button calls `SetAccountTier` with the other tier and the `version` of the row, at once, without a dialog ([the admin API](/reference/admin-api/index.html.md#setaccounttier)). If the account changed after the page read it, the call fails with `aborted`, and the row shows the error. A move keeps what the account has: its machines, its orgs, and its public URLs. The limits of the new tier refuse only what comes after. After the move, the console reads the accounts and the orgs again, because the tier sets the default limits of the personal org. The text under the title of the page says this too.

A pending account shows the facts of its sign-up: "Sign-in" with its sign-in methods, "GitHub" with the GitHub login, "Note", "Address" with the client address, and "Signed up" with the time. A GitHub user made less than 24 hours before the sign-up gets the tag "new GitHub user". Three buttons act on a pending account:

- "Approve" calls `ApproveAccount` with the tier `full` ([the admin API](/reference/admin-api/index.html.md#approveaccount)).
- "Approve as limited" calls `ApproveAccount` with the tier `limited`.
- "Reject" opens the dialog "Reject <name>". It has the check box "Also block this email or identity" and the field "Reason", at most 256 characters, which can stay empty. The button "Reject" of the dialog calls `RejectAccount` ([the admin API](/reference/admin-api/index.html.md#rejectaccount)).

With the filter "Pending", an empty list says "No account waits for approval."

The section "Block list of sign-ups" lists the block list, with the kind, the value, and the reason of each entry, and the button "Remove" of each entry ([the admin API](/reference/admin-api/index.html.md#listsignupblocks)). A form with the fields of an email domain, such as `example.net`, and of a reason adds a domain with the button "Block the domain". The other kinds of entries come from a reject with a block, or from `hopboxd admin account block` ([hopboxd](/reference/hopboxd/index.html.md#admin-account-block)).

The page has no delete action, because the admin delete is only for root on a control server and for admin API keys ([the admin API](/reference/admin-api/index.html.md#deleteaccount)). [Use the Admin area](/guide/admin-area/index.html.md#handle-an-abuse-report) shows how to delete an account with `hopboxd admin account delete`.

### Orgs page

The row of an org has the name of the org with these tags:

| Tag | When |
| --- | --- |
| `personal` | The org is a personal org. |
| `limited` | The org is the personal org of a limited account, so it takes the limited defaults ([the org API](/reference/org-api/index.html.md#usage-limits)). |
| `suspended` | An instance admin suspended the org. The tag is in the error color. |
| `no public URLs` | The limits of the org allow no public machine URL. |
| `port 25 open` | The machines of the org reach outbound TCP port 25 ([the node network](/reference/node-network/index.html.md#outbound-tcp-port-25)). The tag is in the warning color. |

The button "Limits" opens the dialog "Limits of <name>". It has the fields "Machines", "vCPUs", "RAM in GiB", and "Disk in GB", and two selects:

| Select | Choices |
| --- | --- |
| "Public URLs" | "default: yes" or "default: no", "yes", and "no" |
| "Outbound TCP port 25" | "default: blocked", "open", and "blocked" |

An empty field, or a choice "default", takes the default of the install. The personal org of a limited account takes the limited defaults, and the dialog says so. The placeholder of an empty field shows its default. The line "The limits that apply now" gives the limits of the org. An example is "1 machine, 1 vCPU, 1 GiB RAM, 5 GB disk, public URLs: no, outbound TCP port 25: blocked". The dialog also says two rules. A limit below the usage stops nothing, but the org starts no more machines. A machine takes a change of port 25 at its next start or reboot. The button "Set limits" calls `SetOrgLimits` with the `version` of the row ([the admin API](/reference/admin-api/index.html.md#setorglimits)).

The section "Default limits" has two groups of fields:

| Group | Fields |
| --- | --- |
| "Team orgs and personal orgs of full accounts" | "Machines", "vCPUs", "RAM in GiB", "Disk in GB", "Public URLs", and "Orgs of one full account, besides its personal org" |
| "Personal org of a limited account" | "Machines", "vCPUs", "RAM in GiB", "Disk in GB", and "Public URLs" |

The field "Orgs of one full account, besides its personal org" is the cap of a full account. A limited account owns no org. No default opens port 25, so the section has no field for it. The text of the section says to open port 25 for one org in its limits. The button "Set default limits" stays off until a field changes. It calls `SetDefaultOrgLimits`, with the second group in `limited` ([the admin API](/reference/admin-api/index.html.md#setdefaultorglimits)).

### Drain of a server

The server page has the button "Drain" in the danger style. It opens the dialog "Drain <name>" ([Cold move](/reference/cold-move/index.html.md#drain-of-a-node)). The dialog says these things:

- The drain cordons the server and moves each machine to another node, at most 2 at once.
- A running machine stops, moves, and boots again, so it loses its memory.
- Each machine keeps its disk, name, URL, SSH port, and address.
- Once the server holds nothing, the drain decommissions it.
- "Until then, Cancel drain ends the drain and keeps the server."
- If Terraform manages the server, run terraform destroy instead, because the next refresh of Terraform drops a drained server from its state.

A server without the role `node` has no button "Drain", because it holds no machine and the install refuses its drain. An admin session cannot drain a server that hosts a control process: the call fails with `permission_denied` ([the admin API](/reference/admin-api/index.html.md#drainserver)).

While the server drains, the page changes:

- The button reads "Drain again". It starts the drain again, to try the stranded machines again. Its dialog says that the drain tries each stranded machine again and counts its empty checks from zero.
- The buttons "Cordon" and "Uncordon" are hidden, because a drain keeps the server cordoned.
- The button "Cancel drain" calls `CancelDrain` ([the admin API](/reference/admin-api/index.html.md#canceldrain)) after the dialog "Cancel the drain of <name>". The dialog says that no new move starts, that a move whose machine still stops ends, and that a move whose disk copies already goes on to its end. It names the state that the server gets back: cordoned or ready.
- The section "Drain" shows these facts: Started, Moves, Empty checks (`n of 3`, and how long ago the last one ran), and Waits, the error of the drain.
- A notice says that the drain cannot go on by itself when the drain is stuck. It says that a machine without room moves once a node has room, and that a cancel keeps the server.
- A table lists each stranded machine with its reason. A machine whose disk copy a failed move left on this server shows "on another node" under its name.

The admin API has no stream of servers. So the list of servers and the page of a server read again every 2 s while a drain runs.

The calls of the admin API go to the services `hopbox.admin.v1.*`. Each read of the admin API writes an audit entry. So each load of a page of the Admin area adds entries to the audit log ([the admin API](/reference/admin-api/index.html.md#where-the-admin-api-is-served)).

The Admin area keeps the rules for instance admins:

- It has no shell, no web terminal, and no file access for a machine. The machine list has no link to the page of a machine.
- It shows no secret value. No call of the admin API returns the SMTP password, an OAuth client secret, or key material. No call returns the secret of an admin key after its create. The forms of a password and a client secret are write-only: they start empty, and an empty field keeps the value that is set now. After a save, the page empties the field, and the cache of the console drops the request.
- The secret of a new admin key shows once, as `hopboxd admin admin-key create` prints it once. When the page closes, the cache of the console drops it.

Each call needs a sign-in within the last 10 minutes. After an older one, a call fails with `permission_denied` and the detail `RecentSignInRequired`. The page then shows "This step needs a sign-in within the last 10 min" and the button "Sign in again" ([Sign-in](#sign-in)).

## Look

The console has the look of the public site [hopbox.dev](https://hopbox.dev). It uses the same colors, fonts, buttons, cards, and icon:

| Part | Value |
| --- | --- |
| Background | `#0A0E0B`, with panels in `#111712` |
| Text | `#ECF2EC`, and `#A3B1A5` for secondary text |
| Accent | lime `#C4F05A`, for links, the main button, the focus ring, and the state `running` |
| Warning | amber `#FFB443`, for a public URL, a state that changes, and the `not verified` tag of an email. Also for the tag `port 25 open` of an org, and the banner and the notices of an instance admin. |
| Error | `#FF7A6B`, for errors and the delete buttons |
| Text font | Bricolage Grotesque |
| Code font | JetBrains Mono, for names, URLs, and the web terminal |

The console has a dark theme only, as the site has. It does not follow the light or dark setting of the system.

The look keeps the console accessible:

- Text has a contrast of at least 4.5:1 on its background (WCAG 2.2 level AA). A disabled control is the only exception.
- The control that has the keyboard focus shows a lime ring.
- Each state has a word next to its color, such as `running` or `Public URL: public`.
- A link inside a sentence has an underline, so it differs from the text by more than its color.

The web terminal takes its colors from the same values. If a program picks a text color with a contrast below 4.5:1, xterm.js makes it lighter.

## Browsers

The console works in these browsers:

| Browser | Version |
| --- | --- |
| Chrome and Edge | 111 or later |
| Firefox | 114 or later |
| Safari | 16.4 or later |

Vite builds the code for this set, its default build target of Baseline Widely Available browsers.

The terminal page fills the height of the window with the CSS selector `:has()`. Firefox supports it from version 121. In Firefox 114 to 120, the terminal and its bar are 360 pixels high together, and do not fill the window. Rows of a table with actions also do not light up on hover there, because that rule uses `:has()` too. The other parts of the console work the same.

On a short window, such as a phone in landscape, the terminal gets smaller, until it and its bar are 360 pixels high together. Then the page scrolls. The head and the tabs of the org keep their height.

## Sign-in

The sign-in page asks `GetInstallInfo` for the sign-in methods:

- For GitHub and Google, it shows a link to `/auth/<provider>/start?return_to=<path>`. The browser opens that link, and the API sends it on to the provider ([Sign-in with GitHub and Google](/reference/oauth/index.html.md#start)). The console encodes the path once with `encodeURIComponent`, and never decodes it ([Sign-in with GitHub and Google](/reference/oauth/index.html.md#return-path)).
- For magic links, it shows an email form, which calls `RequestMagicLink`. After the call, it says that the link went to the email. With `mail_to_files` in `GetInstallInfo`, it says instead that the install writes each email to a file on its server. An operator then reads the link with `sudo hopboxd admin mail show` ([Mail](/reference/mail/index.html.md#read-the-mail)).

- Without a method, it shows the command that makes a first-admin login link: `sudo hopboxd admin login-link --email <email>`.

A magic link or a login link opens its own page, often in a new tab. So the sign-in page keeps its return path in the local storage of the browser for 15 minutes, the lifetime of a link. The page of the link then opens that path.

Some steps need a recent sign-in, at most 10 minutes old ([Sessions, cookies, and sign-in links](/reference/sessions/index.html.md#recent-sign-in)). In the console, these steps are the approval of a CLI sign-in on `/device` and the link of a sign-in method on `/account`. Each call of the Admin area is such a step too. A link of GitHub or Google starts on the browser endpoint, which sends an older session to `/signin?again=1&return_to=/account` itself ([Sign-in with GitHub and Google](/reference/oauth/index.html.md#link-to-a-signed-in-account)). A call of such a step with an older session fails with `permission_denied` and the detail `RecentSignInRequired`. The page then links to `/signin?again=1&return_to=<path of the page>`. With `again=1`, the sign-in page shows the methods to a person who is signed in already. The new sign-in replaces the session of the browser, and the page of the step opens again.

The console checks a return path with the rules of the server. A good path has these properties:

- It is one path on the install, with at most 512 characters.
- It starts with one `/`.
- It has no backslash, no space, and no control character.
- It has no `.` or `..` segment.

The console drops any other return path, and opens `/`.

The pages of the links read the token from the fragment, then remove the fragment from the address bar and the history before the call. A browser never sends a fragment to a server, so the token never reaches a log or a `Referer` header.

The page of an invitation keeps its token in the session storage of the tab while the person signs in. A sign-in with GitHub or Google in the same tab then comes back to it. A magic link opens in another tab without the token. Go back to the first tab. When the tab gets the focus, it reads the account again and shows the accept button.

A call that fails with `unauthenticated` on a page that needs a session sends the browser to the sign-in page. "Sign out" calls `SignOut`, which ends the session and clears the cookie.

## Calls

The console calls the API with the TypeScript SDK `@hopbox/sdk` of the workspace, in its browser runtime ([TypeScript SDK](/reference/ts-sdk/index.html.md)). The session cookie authenticates each call, so the client has no credential. Each call is a Connect `POST` with the binary format.

The SDK never includes the admin API. So the Admin area has its own generated code of `hopbox.admin.v1` in `web/apps/console/src/gen` ([API definitions](/reference/api-definitions/index.html.md#generators-and-outputs)). It calls the admin API on the same transport of the SDK, with the same retries and idempotency keys.

The pages read and change data with TanStack Query and connect-query on the transport of the SDK. The transport already retries a call that is safe to run again, and puts an idempotency key on each create. A read call gets a retry after `unavailable`, `resource_exhausted`, or `deadline_exceeded`. A change without an idempotency key gets one only after an answer with `Hopbox-Not-Run: 1`, because `fetch` does not tell a refused connection from a lost answer ([TypeScript SDK](/reference/ts-sdk/index.html.md#retries)). So TanStack Query never retries a call itself. A read stays fresh for 15 seconds. A change reads the data of its service again.

## Live machine list

Each page of an org follows the machines of the org with the stream `WatchMachines`. The stream sends each machine, then a `synced` message, then each change. The console shows the first list at the `synced` message. A machine in the state `deleted`, or a `gone_machine_id`, leaves the list.

When the stream breaks, the console keeps the old list on show. It opens the stream again after 1, 2, 5, 10, and then 30 seconds. The new first list replaces the old one at its `synced` message, so a machine that went away in the meantime leaves the list then. The head of the org shows the state of the stream: `connecting`, `live`, or `reconnecting`. An `unauthenticated` error ends the stream and opens the sign-in page.

## Web terminal

The web terminal opens the login shell of the main user of a running machine, as `hopbox connect` does. It runs exec over the WebSocket endpoint `/api/ws/exec` with these start fields ([the machine API](/reference/machine-api/index.html.md)):

| Field | Value |
| --- | --- |
| `machine` | the name of the machine |
| `argv` | empty, for the login shell |
| `tty` | true |
| `size` | the rows and the columns of the terminal |
| `term` | `xterm-256color` |

The terminal is xterm.js. Each key goes to the shell as stdin, and each change of the size of the page sends a resize. The output of the shell goes to the terminal. When the shell ends, the terminal shows how it ended, such as `the shell ended with exit code 2`, and offers "Open a new shell". The button "Open terminal" needs the state `running`.

The browser sends the session cookie with the WebSocket, and the server checks its `Origin` header against `https://<domain>` ([Sessions, cookies, and sign-in links](/reference/sessions/index.html.md#calls-with-the-cookie)). A page of a machine URL thus cannot open a shell with the cookie of a person.

## Browser storage

The console keeps no credential in the storage of the browser. It keeps these keys:

| Key | Storage | Value | Lifetime |
| --- | --- | --- | --- |
| `hopbox.lastOrg` | local | the ID of the org that the browser opened last, for `/` | until "Sign out" |
| `hopbox.returnTo` | local | the return path of the last sign-in page, with its time | 15 minutes, and one read |
| `hopbox.invite` | session | the token of an invitation that waits for a sign-in | until the accept, "Not now", an accept error that a retry cannot fix, or the end of the tab |

A browser without storage loses only these conveniences.

## Headers

Each response of the console has these headers:

| Header | Value |
| --- | --- |
| `Content-Security-Policy` | `default-src 'none'; script-src 'self'; style-src 'self' 'unsafe-inline'; img-src 'self' data:; font-src 'self'; connect-src 'self'; manifest-src 'self'; base-uri 'none'; form-action 'self'; frame-ancestors 'none'` |
| `X-Content-Type-Options` | `nosniff` |
| `X-Frame-Options` | `DENY` |
| `Referrer-Policy` | `no-referrer` |
| `Cross-Origin-Opener-Policy` | `same-origin` |
| `Cross-Origin-Resource-Policy` | `same-origin` |
| `Permissions-Policy` | `camera=(), microphone=(), geolocation=(), payment=(), usb=()` |

The policy allows scripts only from the console itself: no inline script, no `eval`, and no other site. The build has no inline script and no data URL. The console loads nothing from a CDN. No site can show the console in a frame.

The fonts are files of the build, so `font-src 'self'` covers them. Each file is a variable font in WOFF2, so one file gives every weight that the console uses:

| Font | Characters | Size |
| --- | --- | --- |
| Bricolage Grotesque | the Latin subset that hopbox.dev uses | 75 KiB |
| Bricolage Grotesque | the Latin Extended subset that hopbox.dev uses, such as `ł` and `ő`. The browser gets this file only for a page that shows such a character. | 30 KiB |
| JetBrains Mono 2.304 | the Latin and Latin Extended-A characters, arrows, geometric shapes, dingbats such as `✓` and `❯`, box-drawing and block characters, and the Powerline symbols, for terminal programs such as `htop` and `tmux` | 45 KiB |

The policy allows inline styles. xterm.js writes style elements and style attributes while it runs, for the size of the cells and the colors of the text. Without inline styles, the terminal shows the text without its colors and its layout. An inline style cannot run code.

The cache headers depend on the file:

| File | `Cache-Control` |
| --- | --- |
| `index.html`, and each page of the app | `no-cache` |
| a file under `/assets/`, whose name holds the hash of its content | `public, max-age=31536000, immutable` |
| a file that does not exist | `no-store` |

Each file has an `ETag` from its SHA-256, and a request with a matching `If-None-Match` gets `304 Not Modified`. A file of text, script, JSON, or SVG of at least 1 KiB also has a gzip copy. A request with `Accept-Encoding: gzip` gets that copy with `Content-Encoding: gzip` and `Vary: Accept-Encoding`, unless it asks for a range.

## Build

`make console` builds the console:

1. It builds the SDK into `web/packages/sdk/dist/` with `tsc`.
2. It type-checks the console with `tsc`.
3. Vite writes the build to `internal/console/dist/`: `index.html`, and the hashed files under `assets/`, the icon and the two fonts among them. The terminal and xterm.js load in a chunk of their own, only on the terminal page. When an instance admin opens a page of the Admin area, the page loads in a chunk of its own.

`make build` runs `make console` first, so each `hopboxd` holds the console of its own commit. `tools/bundle` refuses a `hopboxd` without the console, so no release ships without it ([Release bundle](/reference/release-bundle/index.html.md)). `internal/console` embeds `internal/console/dist/` with `go:embed`. Git ignores the build, apart from the empty file `.gitkeep`, which `go:embed` needs. The same lockfile gives the same files, so the build is reproducible.

A `hopboxd` built without `make console`, for example with `go build`, holds no console. Its API listener answers each path of the console with `404`. The text is `This hopboxd has no console. Build it with make console, then make build.` At its start, `hopboxd run` logs the record `this hopboxd has no console. Build it with make console, then make build` at level warn.

The console pins each package to an exact version in `web/apps/console/package.json`:

| Package | Version |
| --- | --- |
| `react`, `react-dom` | 19.3.1 |
| `@tanstack/react-router` | 1.170.45 |
| `@tanstack/react-query` | 5.104.5 |
| `@connectrpc/connect-query` | 2.3.1 |
| `@xterm/xterm` | 6.0.0 |
| `@xterm/addon-fit` | 0.11.0 |
| `vite` | 8.3.4 |
| `typescript` | 7.0.2 |
| `vitest` | 5.0.5 |
| `@playwright/test` | 1.64.1 |

## Tests

| Test | What it checks | Command |
| --- | --- | --- |
| `web/apps/console/test/*.test.ts` | With vitest: the rules of the return path and the browser storage, the live machine list, the labels, and the error texts, with the detail `AccountPending`. Also the tag "new GitHub user", the labels of the sign-in methods and of the block list, and the rows without a suspend: the last active instance admin and the own account. Also the clicks that close a dialog, and the colors that the web terminal takes from the tokens. For the Admin area: the form of labels and of amounts, the labels of the server states, the copy of the machine states, and the actor and the count of the notices. Also the labels of the account states and tiers, the name of an account, and the order of the tabs. For the pages Overview, Capacity, and Machines, and the activity of Accounts: the percents and ages, the labels of the health lines, the list "Needs your attention", the points of the line charts, the thresholds of the bursts, the links to Grafana, and the choice to keep the numbers and ask for a sign-in again after 10 minutes. Also the buttons of the tier moves, the yes-or-no limits and the line of the limits that apply. Also the size choices of the dialog "New machine", and the public URL switch in an org without public URLs. | `make test-console`, or `node_modules/.bin/vitest run` in `web/apps/console` |
| `internal/console` | the routes, the headers, the cache, gzip, and the page of a binary without the console | `make test` |
| `TestConsoleTakesOnlyTheFreePaths` in `internal/api` | that the console hides no API route | `make test` |
| `TestConsole` in `internal/control` | the whole console in Chromium with Playwright, `web/apps/console/e2e` | `make test-console` only |

`TestConsole` runs a control role with a node that runs machines with the real guest agent. Before the suite, it writes an hour of samples of the history, in which `box1` uses 95% of its 2 vCPUs. It serves the API with the console of the test binary at `https://hop.example.org`. An HTTP proxy of the test sits in front, and ends TLS with a certificate that the test makes. Chromium reaches the console through that proxy. The suite:

1. Checks the headers and the sign-in page without a session, and checks that both fonts load under the policy.
2. Signs in with a login link, comes back to the page that asked for the sign-in, and finds `box1` running. It checks that the scrolling row of tabs does not cut off the focus ring of a tab or the underline of the current tab.
3. Creates, stops, starts, and deletes a machine, and checks the name suggestion of a taken name.
4. Makes the URL of `box1` public, then private.
5. Runs `echo` in the web terminal, then `exit`. Before that, it checks at 844 by 390 and 1280 by 560 pixels that the tabs of the org keep their height.
6. Adds and removes an SSH key, creates an API key, and sets an env var.
7. Opens each dialog, and checks it at a desktop and a phone width. The title starts at the left edge, the text stays inside the dialog, and the buttons sit at the bottom right. The dialogs are "New machine", "Delete box1", "Revoke the API key", "Remove the SSH key", "Remove the member", "Leave Team", and "Delete Team".
8. Checks that a click on the backdrop and the Escape key close a dialog, and that a click or a drag inside it does not.
9. Creates an org, and finds it in the org switcher.
10. Signs out, and checks that each page then asks for a sign-in.
11. Drops the cookies of the browser, and opens `/device?code=<code>` without a session. It signs in with a new login link, comes back to the code, and approves it.
12. Signs in, makes the session 11 minutes old, and opens `/device?code=<code>` with a second code. The approval fails and asks for a sign-in again. The suite signs in with a new login link, comes back to the code, and approves it. After the run, the Go test polls both device sign-ins and gets the CLI tokens of the account.
13. Opens a second browser with the session of `user1@example.org`, who is not an instance admin. The console shows no "Admin" link, and each page under `/admin` shows "Page not found". A call of the admin API from that page gets `permission_denied` without the detail `RecentSignInRequired`.
14. Opens the Admin area, which opens on `/admin/overview`, and goes to "Servers". It cordons and uncordons the server of the test, and sets its labels. A label without `=` shows an error.
15. Creates a pool and a size in it, retires the size, deletes the pool, and finds the image `default`.
16. Sets the limits of the org `Team`, suspends it with a reason, and unsuspends it. It checks both dialogs at a desktop and a phone width. The dialog "Limits of Team" shows "default: yes" for "Public URLs" and the limits that apply now. The suite sets 5 machines, "Public URLs" "no", and "Outbound TCP port 25" "open". The row then shows "0 of 5" and the tags "no public URLs" and "port 25 open". In "Default limits", the suite sets the machines of the group "Personal org of a limited account" to 2. It finds the value after a reload, and sets it back to 1. The page "Settings" of `Team` shows "no" for "Public URLs" and "open" for "Outbound TCP port 25". It finds `box1` in the machine list with its address, finds no link or button of a terminal or a shell, and stops `box1`.
17. Makes `user1@example.org` an instance admin, and revokes the role. Creates an admin key, which shows its secret once: after a visit to another page, the page no longer holds the secret. Then it revokes the key.
18. Checks that the row of `ann@example.org`, the account of the session, has the button "Suspend" off, with its tooltip. Suspends `user1@example.org` on `/admin/accounts` with the reason `abuse report 1`, and checks the dialog at a desktop and a phone width. The button "Suspend" of the dialog stays off for an empty reason, and Enter in the field with only spaces does nothing. Enter with the reason suspends the account. The row shows the state `suspended` and the reason, and the filter "Suspended" lists that one account. A second browser with the old session of `user1@example.org` gets the sign-in page. An email link of `user1@example.org` then shows the error of the suspension with its reason on `/auth/email`, and no console. The Go test makes that link without a mail, through `HOPBOX_E2E_MAGIC_LINK`. Then the suite unsuspends the account, and the filter "All states" shows it `active` again. The Playwright test is "the Admin area suspends an account, which then has no session, and ends the suspension".
19. Clicks "Make limited" in the row of `user1@example.org`, and the tier changes from `full` to `limited`. On the page "Orgs", one personal org has the tags "limited" and "no public URLs". Its dialog "Limits of Personal" says "The limits that apply now: 1 machine, 1 vCPU, 1 GiB RAM, 5 GB disk, public URLs: no, outbound TCP port 25: blocked." and shows "default: no" for "Public URLs". Then "Make full" sets the tier `full` again, and no org has the tag "limited". The Playwright test is "an instance admin moves an account to the tier limited and back to full".
20. Finds the banner of the notices, and opens `/admin/notices` from it. The notices hold the admin key `notice1`, which the Go test registered as root, and the login-link grants of the setup. The suite clicks "Dismiss all", and the banner goes on each page.
21. Sets the SMTP server and the GitHub app with a password and a client secret. The fields empty after the save, and the page says only that each secret is set. The page and the audit log never hold either secret.
22. Checks that each page of the Admin area fits a phone width of 360 pixels without a horizontal scroll, also Overview and Capacity.
23. Makes the session 11 minutes old, and clicks "Cordon". The action fails and asks for a sign-in again, and the server stays `ready`. The suite signs in with a new login link, comes back to `/admin/servers`, and the cordon works. After the run, the Go test reads the audit log: each action of the Admin area has the actor kind `session`. The test also finds the new SMTP server of the install.
24. Turns on the file mail transport, and links the email `ann.work@example.org` on `/account`. It opens the link from the mail file, and finds the email link in the sign-in methods. Then it makes the session 11 minutes old, and checks that a second link asks for a sign-in again.
25. Drains a second server of the test, which holds no machine, from its page. The empty checks of the test come 2 s apart. The badge shows `draining` or `shutting down`, the section "Drain" shows, and the button "Uncordon" is gone. Then the badge shows `decommissioned`, and the page has no drain button and no button "Cancel drain". The audit log has `server.drain` with the actor kind `session`, and `server.decommission` with the actor `system`. The Playwright test is "the drain button drains a server, which then leaves the install".
26. Opens `/admin/overview`. It finds the six tiles and the four health lines. It finds `box1` under "Needs your attention" at 95% of its vCPUs, from the samples that the Go test writes, and the notices and the cordoned second server there too. It finds no link to Grafana, sets `grafana_url` with `UpdateInstallConfig`, and then finds the link "More in Grafana". The Playwright test is "the Overview shows the tiles, what needs the admin, and the health lines, and links Grafana once it is set".
27. Opens `/admin/capacity`, finds the card of the node with its three gauges, and switches the range from "24 h" to "1 h". The charts then name the last hour.
28. Opens `/admin/machines`, finds `box1` under "Top machines" with its CPU and RAM, sorts by RAM, and finds the link to Grafana and the section "Every machine".
29. Opens `/admin/overview` 375 pixels wide, and checks that the tiles sit in two columns, without a horizontal scroll.
30. Opens `/admin/accounts`, and finds the section "Activity, last 24 h" with its five charts, the public URLs, the suspended orgs, and the link to Grafana.
31. Opens `/admin/overview` in a browser with a fake clock, makes the session 11 minutes old, and moves the clock past the next read. The tiles stay, and the line "Sign in again to refresh" shows, with no error.
32. Sets the sign-up mode `approval` on `/admin/config`. An email link of the new person `user2@example.org` opens `/pending` in a browser of its own. The person saves a note, and the API refuses `ListOrgs` with the detail `AccountPending`. The admin finds the notice under "New accounts" and the row with the note under the filter "Pending", and clicks "Approve". After a reload, the person sees the machines of the personal org. The Playwright test is "in approval mode, a new account waits on its own page until an instance admin approves it".
33. Rejects the new person `user3@example.org` with "Also block this email or identity" and a reason. The entry shows in the block list, the session of the person ends, and a new email link of the same email fails. Then the suite blocks the domain `example.net` and removes it again. The Playwright test is "an instance admin rejects a new account and blocks its email, and the block list adds and removes a domain".
34. Signs up `user4@example.org`, who signs out on `/pending`, signs in again, and deletes the account with "Delete my account". The row of `user4@example.org` is gone from the filter "Pending". Then the admin approves `user5@example.org` with "Approve as limited". The row shows `active`, `limited`, and the button "Make full". The usage line of the personal org of `user5@example.org` says "Public URLs: no." In the dialog "New machine", the size reads "tiny (default): 1 vCPU, 1 GiB RAM, 5 GB disk", and the person creates `box5`. On the page of `box5`, the public URL switch is disabled, with the text "The org cannot make a machine URL public. An instance admin can allow it". The Playwright test is "a pending account signs out and deletes itself, and an instance admin approves another one as limited".
35. Checks that no page error and no policy violation happened.

Steps 12 and 14 to 31 each start with a new sign-in through a login link, and step 11 starts without cookies. So they need no session from the steps before them, and their order does not matter. Each sign-in goes through the page of the link and the real sign-in of the API, as for a person.

`TestConsole` gives Playwright the install in env vars, so you do not set them yourself. `HOPBOX_E2E_PROXY` is the proxy that serves the console and the API. `HOPBOX_E2E_LOGIN_LINK` is the first login link of `ann@example.org`, the account of the suite and an instance admin. A `POST` to `HOPBOX_E2E_LOGIN`, a URL on loopback, returns a new login link of that account. `HOPBOX_E2E_USER_COOKIE` is the session token of `user1@example.org`, an account with a verified email that is not an instance admin. `HOPBOX_E2E_USER_CODE` and `HOPBOX_E2E_USER_CODE_2` are the codes of the two device sign-ins. A `POST` to `HOPBOX_E2E_AGE_SESSIONS`, a URL on loopback, makes every session of `ann@example.org` 11 minutes old. A `POST` to `HOPBOX_E2E_MAIL`, a URL on loopback, turns on the file mail transport. A `GET` of it returns the link in the newest mail file. A `POST` to `HOPBOX_E2E_MAGIC_LINK?email=<email>`, a URL on loopback, makes a magic link for the email without a mail, and returns the link. `HOPBOX_E2E_TEAM_ORG` is the ID of the org `Team`, where `bob@example.org` is a member of the account of the suite. `HOPBOX_E2E_DRAIN_SERVER` is the ID of a second server, cordoned and without machines, which the suite drains. `HOPBOX_E2E_PEER_ADDRESS` is the peer address of the first server, which the server page shows. The others are `HOPBOX_E2E_URL`, `HOPBOX_E2E_SSH_KEY`, and `HOPBOX_E2E_MACHINE`.

Without `HOPBOX_TEST_CONSOLE`, `TestConsole` skips. `make test-console` sets the variable, and `make test` does not, so `make test` needs no browser. The test needs `corepack pnpm install`, a build of the console, and the Chromium of Playwright: run `web/apps/console/node_modules/.bin/playwright install chromium` once. If one of them is missing, the test fails.
