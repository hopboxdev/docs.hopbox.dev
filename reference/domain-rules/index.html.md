# Domain rules

> The rules of machine states and actions, restart policies, names and their suggestions and reservation, SSH ports, machine URLs, start commands, auto-delete timeouts, sizes, emails, account states and tiers, orgs, usage limits, API key kinds, credential names, SSH keys, and scopes.

This page lists the rules of the domain types of hopbox. It covers machine states and actions, restart policies, machine names and their reservation, SSH ports, and the settings of a machine. It also covers the install domain, the public addresses, the TXT records of the zone, sizes, emails, and display names. It also covers org names, org roles, usage limits, the org suspension, and API key kinds. Then it covers credential names, SSH keys, and the scopes of env vars and secrets, and the names and values of env vars. The Go package `internal/domain` holds these rules. It has no I/O, so every role of `hopboxd` and the guest agent can use it.

The admin API checks the [install domain](#install-domain) and the [public addresses](#public-addresses) with these rules. The machine API checks the names and the settings of machines with them ([the machine API](/reference/machine-api/index.html.md)). Later releases build the other API calls on them, and the API reference then links here.

## Machine states

A machine is always in one current state. It also has a target state, which is the state that it must reach.

| State | Meaning | Can be a target |
| --- | --- | --- |
| `pending` | The machine exists in the control store, but no node acted on it yet. | no |
| `starting` | The microVM boots. A boot, a reboot, a wake, and a restore pass through this state. | no |
| `running` | The microVM runs. | yes |
| `standby` | The microVM is paused, and its memory stays in the RAM of the node. | yes |
| `hibernated` | The memory is in a memory file on disk, and the microVM does not run. | yes |
| `stopped` | The microVM does not run. The disk stays. | yes |
| `failed` | An action failed. The saved state of the machine stays. | no |
| `deleted` | The machine and its saved state are gone. | yes |
| `moving` | A cold move takes the machine to another node, which pulls its disk ([Cold move](/reference/cold-move/index.html.md)). The control role sets this state at the hand-off of the move. | no |

A new machine starts with the current state `pending` and the target state `running`.

## Steps between current states

Only the executor of a machine changes its current state. Each step goes from one current state to a next one. The executor can make only the steps in this table. `CheckStep(from, to)` refuses any other step with the error `machine state step not allowed: from <from> to <to>`.

| From | To |
| --- | --- |
| `pending` | `starting`, `stopped`, `failed`, `deleted` |
| `starting` | `running`, `stopped`, `failed` |
| `running` | `starting`, `standby`, `hibernated`, `stopped`, `failed`, `moving`, `deleted` |
| `standby` | `running`, `hibernated`, `stopped`, `failed`, `moving`, `deleted` |
| `hibernated` | `starting`, `failed`, `moving`, `deleted` |
| `stopped` | `starting`, `moving`, `deleted` |
| `failed` | `starting`, `stopped`, `deleted` |
| `moving` | `running`, `standby`, `hibernated`, `stopped`, `failed` |
| `deleted` | none |

These rules shape the table:

- A hibernated machine never goes straight to `stopped`. That step drops the memory file, and only a delete removes saved state.
- A machine in `starting` or `moving` first ends its action. Then it can go to `deleted`.
- `deleted` is the end. No step leaves it.
- Every other state can reach `deleted`.

## Lifecycle actions

A lifecycle action only sets the target state. The executor then moves the current state toward the target, one step at a time. `Apply(target, current)` returns the new target state, or the error `machine action not allowed: ...`.

| Action | Allowed target states | Allowed current states | New target state |
| --- | --- | --- | --- |
| `create` | A new machine only. It starts with the target `running`. | | |
| `start` | `stopped` or `running` | any | `running` |
| `stop` | `running`, `standby`, or `stopped` | any | `stopped` |
| `reboot` | `running` | `running` | unchanged |
| `pause` | `running` or `standby` | `running` or `standby` | `standby` |
| `resume` | `standby` | any | `running` |
| `hibernate` | `running`, `standby`, or `hibernated` | any | `hibernated` |
| `wake` | `hibernated` | any | `running` |
| `rename` | any target except `deleted` | any | unchanged. A rename reboots a running machine. |
| `delete` | any target except `deleted` | any | `deleted` |
| `fork` | `running` | `running` | unchanged. The fork is a new machine. |
| `checkpoint` | `running` | `running` | unchanged |
| `restore` | `running`, `standby`, or `stopped` | any state except `hibernated`, `moving`, and `deleted` | `running` |

No action works on a machine whose target is `deleted`. The error is `machine action not allowed: cannot <action> a machine that is being deleted`.

An action that asks for the target that the machine already has succeeds and changes nothing. For example, `stop` on a machine with the target `stopped` succeeds.

## Hibernated machines keep their memory file

Only a delete drops the memory file of a hibernated machine. So a machine with the target `hibernated` refuses `stop` and `restore` with the error `machine action not allowed: cannot <action> a hibernated machine: wake it first`. To stop or restore it, wake it first.

A wake can be pending: the target is `running`, but the current state is still `hibernated`. Then `restore` waits for the wake to end, because a restore replaces the memory file that the wake needs. The error is `machine action not allowed: cannot restore a machine that is hibernated`.

Other paths follow the same rule:

- An admin stop of a hibernated machine gets the same error, and the machine stays hibernated.
- `delete` is the only action that drops a memory file.

### Org suspension

An instance admin can suspend an org ([the org API](/reference/org-api/index.html.md#suspension)). A suspension stops the machines of the org, and blocks the actions that make a machine run or make a new one. It leaves a hibernated machine hibernated. It blocks the wake of the machine until the org is active again. It never stops the machine, because a stop is not allowed for a hibernated machine.

The suspend of an account suspends its personal org and each org that it owns alone, by the same rules ([the admin API](/reference/admin-api/index.html.md#suspendaccount)). In its other orgs, it stops the private machines of the account, and leaves a hibernated one hibernated.

`Action.CheckSuspended()` gives the rule for each lifecycle action. A blocked action gets the error `org suspended: an instance admin suspended the org, so it cannot <action> a machine`.

| Action | In a suspended org |
| --- | --- |
| `create`, `start`, `reboot`, `resume`, `wake`, `fork`, `restore` | blocked |
| `delete` | blocked |
| `stop`, `pause`, `hibernate`, `rename`, `checkpoint` | allowed |

Exec, file copy, SSH, and the machine URLs are blocked too. Exec and file copy check the suspension of the org now ([the machine API](/reference/machine-api/index.html.md#suspension)). The machine URLs check it now: the edge answers `403` ([the edge role](/reference/edge/index.html.md#routing)). SSH checks it now: the SSH front door refuses every key, and ends the open connections of the org ([the SSH front door](/reference/ssh-front-door/index.html.md#checks-of-an-open-connection)).

While an org is suspended, nothing deletes its data, for example the evidence of abuse. So the members cannot delete a machine, and these calls fail with `failed_precondition` and `org suspended: ... Ask for the end of the suspension first` ([the org API](/reference/org-api/index.html.md#suspension)):

- `DeleteOrg` of the org.
- `LeaveOrg` and `RemoveOrgMember` in the org, because they delete what the account made in the org and did not share.
- `DeleteAccount` of an account whose personal org is suspended, or that is a member of a suspended org.

The admin `DeleteAccount` of an operator goes on in a suspended org. Its audit entry keeps the evidence instead ([the admin API](/reference/admin-api/index.html.md#deleteaccount)).

The env vars and secrets of a suspended org do not change either. `SetVar` and `DeleteVar` fail with `failed_precondition` and `org suspended: an instance admin suspended the org, so its env vars and secrets do not change until the suspension ends` ([the env var API](/reference/vars-api/index.html.md#suspension)).

The rule holds for the members of the org. Instance admins can stop or delete a machine of a suspended org, for example to handle abuse. The admin API has these calls ([the admin API](/reference/admin-api/index.html.md#machineservice)).

### Executor contract after a failed wake

The executor follows this rule ([the node executor](/reference/executor/index.html.md#saved-state)). A wake can fail. The machine then has the target `running` and the current state `failed`.

1. The executor keeps the memory file. A failure never deletes saved state.
2. `stop` is still allowed. It sets the target `stopped`, and the executor makes the step `failed` to `stopped`. The memory file stays.
3. A later `start` of the machine first retries the wake from the kept memory file. Only after that wake succeeds does the machine run.
4. Only `delete` drops the memory file.

## Restart policy

A machine can stop by itself. Its restart policy then says what the executor does.

| Policy | What it does |
| --- | --- |
| `always` | The default. After a crash, a shutdown from inside, a failed boot, or a node reboot, the executor starts the machine again. If the start command exits, the executor asks the guest agent to run it again. |
| `never` | The machine and its start command stay stopped. After a shutdown from inside, the target turns `stopped`. After a crash or a failed boot, the machine stays `failed`. A start boots it again. A node reboot does not change the target, so a machine with the target `running` boots again. |

Between restarts, the executor waits 1 s, then doubles the wait after each restart, up to 5 minutes (`RestartBackoff`). A run of at least 5 minutes starts the count again. [The node executor](/reference/executor/index.html.md#restart-policy) has every case.

The machine lease of the executor lives 30 s after its grant or its renewal (`MachineLeaseTTL`, [the executor protocol](/reference/executor-protocol/index.html.md#the-machine-lease)).

## Machine names

A machine name is the first label of the machine URL, such as `box1` in `box1.example.com`. `ValidateMachineName` checks these rules:

- It has 1 to 63 characters.
- It has only the characters `a` to `z`, `0` to `9`, and `-`.
- It starts with a letter.
- It does not end with `-`.
- It has no `--` in the third and fourth places. RFC 5891 keeps that form for international names, such as `xn--bx-0ia`, which a browser shows with other letters.
- It is not `ns1` or `ns2`. The name servers of the install use these names.

| Name | Result |
| --- | --- |
| `box1` | valid |
| `my-box-2` | valid |
| `Box1` | `invalid name: the machine name "Box1" must start with a lowercase letter` |
| `box-` | `invalid name: the machine name "box-" must not end with a hyphen` |
| `xn--bx-0ia` | `invalid name: the machine name "xn--bx-0ia" must not have "--" in the third and fourth places` |
| `ns1` | `reserved machine name: "ns1"` |

A create and a rename also refuse a new name that looks like a credential, with `ValidateNewMachineName`. The check uses the patterns of the audit log ([Control store tables](/reference/control-store/index.html.md#record-an-entry)), in `LooksLikeCredential`. A machine name is public. It is in the machine URL, in DNS, and in the audit entry of the create or the rename. Examples are `xoxb-` with 10 more digits, which has the form of a Slack token, and `dev-sk-proj-` with 20 more letters.

| Name | Result |
| --- | --- |
| `xoxb-123456789` | valid, because a Slack token has at least 10 characters after `xoxb-` |
| `xoxb-` with 10 more digits | `invalid name: the machine name looks like a credential. A machine name is public: it is in the machine URL, in DNS, and in the audit log. Pick another name` |

A lookup by name, the routes of the edge, and the machine URL use only the rules of `ValidateMachineName`. An action on a machine whose name the audit log refuses writes its audit entry without the refused name details: each of `name`, `old_name`, and `new_name` that looks like a credential. The target ID of the entry names the machine. A rename of a machine to its own name changes nothing, and passes before the credential check. So a new pattern in the audit log never cuts off a machine that holds its name already.

### Machine name suggestions

A create without a name, or with a name that is taken, needs other names. `SuggestMachineNames(base, n)` returns up to `n` + 3 valid names close to `base`, in order of preference. The caller keeps the first one that is free. The machine API asks for `n` = 9.

| Base | Suggestions |
| --- | --- |
| A valid name that ends in a number, such as `box1` | It counts on from the number: `box2`, `box3`, and so on. |
| Another valid name, such as `web` | It adds `-2`, `-3`, and so on: `web-2`, `web-3`. |
| An empty or invalid name | `box1`, `box2`, and so on. |

3 names with a random suffix follow the `n` numbered names, for example `web-k3x9q`. The suffix has 5 characters from `a` to `z` and `0` to `9`. These names help in an org whose numbers are all taken. Each suggestion is a valid name, and none looks like a credential. A base whose every suggestion looks like a credential gets `box1`, `box2`, and so on. An example is `xoxb-abcdefgh`: each number or suffix makes it a Slack token. A stem that is too long is cut, so the name fits in 63 characters. A final number of 1 to 9 digits counts, but a final number of 0 or of more digits does not. The list never holds the base itself.

### Name reservation

The name of a deleted machine stays out of reach of other orgs for 7 days (`NameReservation`). The old name of a renamed machine does too. So a URL never leads to the machine of a stranger soon after a change. The org that held the name can take it back during the reservation. The SSH port of a deleted machine stays held for the same 7 days. The leader frees each name and each port at the end of its reservation ([hopboxd](/reference/hopboxd/index.html.md#leader-jobs)).

An org holds at most 100 names in reservation (`MaxNameReservations`). A rename beyond that fails with `usage limit exceeded`, so renames cannot hold back names from other orgs without end. A delete always reserves its name.

## SSH ports

Each machine has one SSH port on the edge servers, from 10000 to 30000 (`SSHPortMin`, `SSHPortMax`). A create picks a random free port. A machine keeps its port for its whole life.

## Machine URLs

The URL of a machine reaches one port in the machine. The default is 8000 (`DefaultURLPort`), as on boxd. `ValidateURLPort` accepts a port from 1 to 65535. Another port gets `invalid machine config: the URL port <port> is not 1 to 65535`.

`URLVisibility` says who can open the URL:

| Visibility | Who can open the URL |
| --- | --- |
| `private` | The default. Only an account that uses the machine, with a console session in its browser, under the policy action `machine.open_url`: the creator of a private machine, or a member of the org of a shared one. An org admin cannot open the private machine of another member. |
| `public` | Anyone, without an account. |

The edge role serves the URLs ([the edge role](/reference/edge/index.html.md)). [The edge role](/reference/edge/index.html.md#private-machine-urls) gives the sign-in of a private URL.

## Start command

The start command is a command that the guest agent runs with a shell after each boot. Empty means no command. `ValidateStartCommand` checks these rules:

| Rule | Error |
| --- | --- |
| At most 4096 bytes (`MaxStartCommandLen`) | `invalid machine config: the start command has <n> bytes, more than 4096` |
| Valid UTF-8 | `invalid machine config: the start command is not valid UTF-8` |
| No NUL byte | `invalid machine config: the start command holds a NUL byte` |

## Auto-delete timeout

The auto-delete timeout is the time without use after which hopbox deletes a machine. 0 turns it off, and is the default. `ValidateAutoDeleteTimeout` checks these rules:

| Rule | Error |
| --- | --- |
| 0, or 1 minute to 365 days | `invalid machine config: the auto-delete timeout <d> is not 0 and not 1m0s to 8760h0m0s` |
| A whole number of seconds | `invalid machine config: the auto-delete timeout <d> is not a whole number of seconds` |

## Install domain

The install domain is the domain of the whole install, such as `example.com`. An instance admin sets it through the admin API ([the admin API](/reference/admin-api/index.html.md#installconfigservice)). `ValidateInstallDomain` checks these rules:

- It has at least two labels, joined by dots, such as `example.com`.
- Each label has 1 to 63 characters.
- Each label has only the characters `a` to `z`, `0` to `9`, and `-`. Upper case letters are refused, so the domain is always lowercase.
- No label starts or ends with `-`.
- The last label is not all digits, so an IPv4 address is refused.
- The whole domain has at most 189 characters. A machine URL puts a machine name of up to 63 characters and a dot before the domain. A DNS name has at most 253 characters.

| Domain | Result |
| --- | --- |
| `example.com` | valid |
| `1password.example.com` | valid |
| `localhost` | `invalid name: the install domain "localhost" must have at least two labels, such as hop.example.org` |
| `Hop.example.org` | `invalid name: the install domain "Hop.example.org": the label "Hop" may hold only a-z, 0-9, and "-"` |
| `192.168.0.1` | `invalid name: the install domain "192.168.0.1" must not end with a label of digits only` |

## Public addresses

The public addresses of the install are the IP addresses at which the edge servers take traffic. The zone of the DNS role points the domain, `ns1`, `ns2`, and every machine name at them ([the DNS role](/reference/dns/index.html.md#the-zone-of-the-install)). An instance admin sets them through the admin API ([the admin API](/reference/admin-api/index.html.md#updateinstallconfig)). `PublicAddresses` checks these rules:

- The list has at most 16 addresses. Each address is in every `A` or `AAAA` answer of the zone, and 16 fit in a UDP answer of 512 bytes. An empty list is valid.
- Each address is an IPv4 or an IPv6 address.
- A client on another host must be able to reach the address. So it is not unspecified, such as `0.0.0.0` or `::`, not multicast, and not link-local, such as `169.254.0.1` or `fe80::1`.
- An IPv6 address has no zone, such as `%eth0`.
- No address appears twice.

Loopback and private addresses pass, for an install on a laptop or on a private network.

`PublicAddresses` returns the addresses in the text form of the Go package `netip`, in their order. So an IPv6 address becomes lowercase and short, and an IPv4 address in IPv6 form becomes IPv4. The check of duplicates uses this form.

| Address | Result |
| --- | --- |
| `203.0.113.10` | valid |
| `2001:DB8::10` | valid, kept as `2001:db8::10` |
| `::ffff:203.0.113.10` | valid, kept as `203.0.113.10` |
| `127.0.0.1`, `10.0.0.5` | valid |
| `example.org` | `invalid public address: "example.org" is not an IPv4 or IPv6 address` |
| `0.0.0.0` | `invalid public address: "0.0.0.0" is not an address that a client on another host can reach` |
| `fe80::1%eth0` | `invalid public address: "fe80::1%eth0" has an IPv6 zone` |
| `203.0.113.10` twice | `invalid public address: 203.0.113.10 appears twice` |
| 17 addresses | `invalid public address: 17 addresses, more than 16` |

## TXT records

The zone of the install can hold TXT records, such as the DNS-01 challenge of ACME at `_acme-challenge` ([the DNS role](/reference/dns/index.html.md#txt-records)). `ValidateTXTRecord` checks the name and the value of each one:

- The name is relative to the domain, such as `_acme-challenge`. It has 1 to 63 characters. With a dot and the longest domain of 189 characters, it fits in a DNS name of 253 characters.
- The name is labels joined by dots. Each label has 2 to 63 characters and starts with `_`. It holds `a` to `z`, `0` to `9`, `-`, and `_`. Upper case letters are refused.
- A label without `_` can be the name of a machine, such as `box1`. A TXT record below a machine name hides the wildcard address of that machine, so each label starts with `_`. No machine name starts with `_`.
- The value has 1 to 255 characters: one character string of a TXT record (RFC 1035 section 3.3).
- The value holds only printable ASCII, from space to `~`, without `"`, `\`, `;`, or `$`. So it needs no escape in the text form of a zone, and the DNS role accepts each value that passes.

The zone holds at most 64 TXT records. An ACME order for the domain and its wildcard needs 2. A name can hold more than one value.

| Name and value | Result |
| --- | --- |
| `_acme-challenge`, `gfj9Xq...Rg85nM` | valid |
| `_ACME-challenge`, `x` | `invalid TXT record: the name "_ACME-challenge" is not labels that start with _ and hold a-z, 0-9, - and _, joined by dots, such as _acme-challenge` |
| `_acme-challenge.box1`, `x` | `invalid TXT record: the name "_acme-challenge.box1" is not labels that start with _ and hold a-z, 0-9, - and _, joined by dots, such as _acme-challenge` |
| empty name | `invalid TXT record: the name "" must have 1 to 63 characters` |
| `_acme-challenge`, empty value | `invalid TXT record: the value must have 1 to 255 characters` |
| `_acme-challenge`, `a;b` | `invalid TXT record: the value holds a character that is not printable ASCII, or one of "\;$` |

## Sizes

A size sets the vCPUs, the RAM, and the disk of a machine. RAM uses binary units (1 GiB is 2^30 bytes). Disk uses decimal units (1 GB is 10^9 bytes).

A fresh install has one size:

| Name | vCPUs | RAM | Disk |
| --- | --- | --- | --- |
| `default` | 2 | 8 GiB | 100 GB, sparse |

An instance admin adds other sizes, and retires them, with the admin API ([the admin API](/reference/admin-api/index.html.md#sizeservice)). An operator size can also name a pool and labels, which become the placement filters of its machines ([Placement](/reference/placement/index.html.md#hard-filters)).

Every size must meet these bounds:

| Field | Rule |
| --- | --- |
| name | The same syntax as a machine name. `ns1` and `ns2` are allowed. |
| vCPUs | 1 to 32. Firecracker gives a microVM at most 32 vCPUs. |
| RAM | At least 128 MiB, and a whole number of MiB. |
| disk | At least 1 GB, and a whole number of 512-byte sectors. |

## Emails

hopbox compares emails in one form, so `Ann@Example.org` and `ann@example.org` name one account. `NormalizeEmail` checks an email and returns that form:

- It drops the spaces around the email, and writes its ASCII letters in lower case. No other letter changes case.
- It refuses a capital letter that is not ASCII, such as `É`, the Kelvin sign `K` (U+212A), or `İ` (U+0130). A Unicode case mapping changes these letters into ASCII letters, so two different addresses become one. For example, a case mapping makes `ceo@icorp.com`, an address at another domain, out of `ceo@İcorp.com`.
- It takes only one plain address, without a display name or angle brackets.
- It has a local part, an `@`, and a domain. A domain in brackets, such as `[127.0.0.1]`, is refused.
- It has at most 254 characters.

| Email | Result |
| --- | --- |
| `ann@example.org` | `ann@example.org` |
| ` Ann@Example.ORG ` | `ann@example.org` |
| `ann+hopbox@example.org` | `ann+hopbox@example.org` |
| `Zoë@Example.org` | `zoë@example.org` |
| `ceo@İcorp.com` | `invalid email: "ceo@İcorp.com" holds the capital letter 'İ', which is not ASCII. Write it in lower case` |
| `Ann <ann@example.org>` | `invalid email: "Ann <ann@example.org>" is not one plain email address, such as ann@example.org` |
| `ann@[127.0.0.1]` | `invalid email: "ann@[127.0.0.1]" must have a name, an @, and a domain` |

## Display names

The display name of an account is the name that its person chose. `ValidateDisplayName` checks these rules:

- It has at most 64 characters, in valid UTF-8.
- It has no control characters, such as a newline.
- It does not start or end with a space.
- It can be empty.

## Account states and tiers

Each account has a state and a tier. The type `AccountState` has the values `pending`, `active`, and `suspended`. `ParseAccountState` refuses any other text with `ErrUnknownAccountState`: `unknown account state: "<text>"`. The type `AccountTier` has the values `limited` and `full`.

| State | Meaning |
| --- | --- |
| `pending` | The account waits for approval by an instance admin. No release makes such an account yet. |
| `active` | The account acts as its credentials allow. |
| `suspended` | An instance admin suspended the account. It has no session and no working credential until the unsuspend. Its data stays ([the admin API](/reference/admin-api/index.html.md#suspendaccount)). |

## Credential names

A person names each API key, admin API key, SSH key, and CLI sign-in, for example `ci` or `ann@laptop`. The name only helps people tell their credentials apart, so it need not be unique. `ValidateCredentialName` checks these rules:

- It has at most 64 characters, in valid UTF-8.
- It has no control characters, such as a tab, and no format characters, such as a direction mark.
- It holds no token prefix of hopbox: `hbx_`, `hbxat_`, `hbxrt_`, `hbxadm_`, or `hbxe_`, in any case. The audit log never records such a text.
- It does not start or end with a space.
- It can be empty. An API key and an admin API key need a name, so their calls refuse the empty name.

## Org names

`ValidateOrgName` checks the name of an org:

- It has 1 to 64 characters, in valid UTF-8.
- It has no control characters, such as a newline.
- It does not start or end with a space.

Org names need not be unique, because the ID names an org.

| Name | Result |
| --- | --- |
| `Team` | valid |
| `Acme Labs` | valid |
| empty | `invalid org name: the name is empty` |
| ` Team` | `invalid org name: it must not start or end with a space` |

## Org roles

The org roles are `member`, `admin`, and `owner`, from the fewest rights to the most. Each role has the rights of the role below it. `AtLeast(min)` reports whether a role has every right of `min`.

## Usage limits

Each org has usage limits on its machines, its vCPUs, its RAM, and its disk. `OrgLimits` holds them, and `OrgUsage` holds what an org uses. RAM and disk are in bytes. A limit of 0 allows nothing.

| Limit | `DefaultOrgLimits` |
| --- | --- |
| Machines | 3 |
| vCPUs | 6 |
| RAM | 24 GiB |
| Disk | 300 GB |

The defaults fit three machines of the default size. The rules of the limits:

- Each limit is 0 or more. `Validate` refuses a limit below 0 with `invalid limit`.
- An org can have its own limits, `LimitOverrides`. Each limit that it does not set takes the default of the install. `Apply(defaults)` returns the limits of the org.
- Every machine that is not deleted counts in the machines and the disk. Only a machine whose target is `running` or `standby` counts in the vCPUs and the RAM. `HoldsCompute()` of a target state reports this case. The placement uses the same rule for the room of a node ([Placement](/reference/placement/index.html.md#what-a-node-uses)).
- `AfterCreate(size)` adds a new machine to a usage. A new machine has the target `running`, so it counts in all four limits.
- `AfterStart(size, counted)` adds the vCPUs and the RAM of a machine that exists. If its target counts it already, `counted` is true and the usage stays as it is. So a start never counts a machine twice.
- `Check(usage)` refuses a usage above a limit with `usage limit exceeded`. It names the first such limit, in the order machines, vCPUs, RAM, and disk. A create checks `limits.Check(usage.AfterCreate(size))`, and a start checks `limits.Check(usage.AfterStart(size, counted))`.

| Limits | Usage | Result |
| --- | --- | --- |
| the defaults | 3 default machines | fits |
| the defaults | 4 default machines | `usage limit exceeded: the org would have 4 machines, and its limit is 3` |
| 2 machines, 4 vCPUs, 8 GiB, 100 GB | 1 machine with 6 vCPUs, 8 GiB, and 100 GB | `usage limit exceeded: the org would use 6 vCPUs, and its limit is 4` |
| 2 machines, 2 vCPUs | 2 default machines, one runs, one is stopped. The stopped one starts. | `usage limit exceeded: the org would use 4 vCPUs, and its limit is 2` |
| 2 machines, 2 vCPUs | the same, and the one that runs starts again | fits |

## API key kinds

| Kind | Meaning |
| --- | --- |
| `member` | Acts as the account that made it, in one org. |
| `org` | A service credential of the org for shared machines. It has the rights of a member without personal resources. |

An API key expires after 30, 90, or 365 days, or at a custom date. The default is 90 days. A custom date must be in the future and at most 365 days ahead, so every key expires within a year. An org holds at most 100 API keys. The limit counts expired keys too, until the sweep deletes them 30 days after their expiry. [CLI tokens, API keys, admin API keys, and SSH keys](/reference/credentials/index.html.md#api-keys) gives the rights of each kind.

## SSH keys

`ParseSSHPublicKey` checks one public key in the `authorized_keys` form, as in a `.pub` file ([CLI tokens, API keys, admin API keys, and SSH keys](/reference/credentials/index.html.md#ssh-keys)):

| Key type | Size |
| --- | --- |
| `ssh-ed25519` | 256 bits |
| `ecdsa-sha2-nistp256`, `ecdsa-sha2-nistp384`, `ecdsa-sha2-nistp521` | the size of the curve |
| `ssh-rsa` | 3072 bits (`MinRSABits`) to 16384 bits (`MaxRSABits`) |

It refuses every other type, such as DSA keys and security keys. It also refuses certificates, key options such as `command=`, and a text of more than 16 KiB. The text must have one line that is not empty, so a comment line before the key fails too. It returns the type, the size, the key without its comment, the SHA-256 fingerprint, and the comment.

## Scopes of env vars and secrets

Each env var and each secret has a scope.

| Scope | Reaches | Who can set it |
| --- | --- | --- |
| `shared` | The shared machines of the org. This scope is the default. | org admins and org owners |
| `private` | The private machines of its user only. | any org member, for their own machines |
| `all` | Every machine of the org, shared or private. | org admins and org owners |

`Scope.Reaches(shared, owner)` says whether a value goes into a machine. `shared` is true for a shared machine. `owner` is true for a private value of the creator of the machine. `Scope.CanSet(role)` says whether a role sets a scope. [The env var API](/reference/vars-api/index.html.md#env-of-a-machine) gives the env of a machine, with the rule for a name in two scopes.

## Env var names and values

`ValidateEnvVarName` checks the name of an env var or a secret:

- It has 1 to 128 characters (`MaxEnvVarNameLen`).
- Each character is an ASCII letter, a digit, or `_`. The first character is not a digit. The check matches case, so `path` is valid.
- It does not look like a credential (`LooksLikeCredential`, the patterns that the audit log refuses). A name is never sealed.
- It is not a reserved name, and it does not start with a reserved prefix. `ReservedEnvVarNames()` and `ReservedEnvVarPrefixes()` return the lists.

| Reserved names | The names of the shell: `ENV`, `IFS`, `OLDPWD`, `PROMPT_COMMAND`, `PS0`, `PS1`, `PS2`, `PS4`, `PWD`, `SHELLOPTS`, and `SHLVL`. The names of a login: `HOME`, `LOGNAME`, `PATH`, `SHELL`, `TERM`, and `USER`. The name of glibc: `GCONV_PATH`. |
| --- | --- |
| Reserved prefixes | `BASH_`, `HOPBOX_`, `LD_` |

The guest agent or the login shell sets these names, or they can make the shell or the dynamic loader run code. No value of any scope takes them. The guest agent runs `ValidateEnvVarName` on each name that it gets too.

`ValidateScopedEnvVarName(scope, name)` checks the name of a value with a scope. A private value has the rules of `ValidateEnvVarName`. A shared or an all value also refuses the names of `OrgReservedEnvVarNames()` and the prefixes of `OrgReservedEnvVarPrefixes()`. A program loads or runs code from these names, or finds its code through them. Examples are `NODE_OPTIONS`, `PYTHONSTARTUP`, `EDITOR`, `GLIBC_TUNABLES`, and every name that starts with `GIT_` or `SSH_`. The prefix `npm_config_` matches in any case, because npm reads it in any case. A value of the scope `all` reaches the private machines of every member, so an org value never takes them. A member can still take them for its own private values. `LANG` and the `LC_*` names stay open.

These lists reduce the risk that an org admin runs code in the private machine of a member. They do not remove it, because any env var can change what a program does. [The env var API](/reference/vars-api/index.html.md#names-and-values) gives each name and the program that reads it.

`ValidateEnvVarValue` checks a value: valid UTF-8 without a NUL byte, at most 16 KiB (`MaxEnvVarValueBytes`). The empty value is valid. A value can hold newlines.

`EnvVarSize(name, value)` is what one value counts toward the byte limits: the bytes of its name and its value.

| Constant | Value | What it bounds |
| --- | --- | --- |
| `MaxOrgEnvVars` | 100 | The shared and all values of one org. |
| `MaxOrgEnvVarBytes` | 262144 (256 KiB) | The names and values of the shared and all values of one org, together. |
| `MaxPrivateEnvVars` | 50 | The private values of one account in one org. |
| `MaxPrivateEnvVarBytes` | 131072 (128 KiB) | The names and values of the private values of one account in one org, together. |

## Errors

Each error starts with one of these texts. Go code matches them with `errors.Is`.

| Error | Go value |
| --- | --- |
| `unknown machine state` | `ErrUnknownState` |
| `machine state step not allowed` | `ErrStepNotAllowed` |
| `unknown machine action` | `ErrUnknownAction` |
| `machine action not allowed` | `ErrActionNotAllowed` |
| `unknown restart policy` | `ErrUnknownRestartPolicy` |
| `invalid name` | `ErrInvalidName` |
| `invalid public address` | `ErrInvalidPublicAddress` |
| `invalid TXT record` | `ErrInvalidTXTRecord` |
| `reserved machine name` | `ErrReservedName` |
| `invalid size` | `ErrInvalidSize` |
| `invalid machine config` | `ErrInvalidMachineConfig` |
| `unknown org role` | `ErrUnknownRole` |
| `unknown API key kind` | `ErrUnknownKeyKind` |
| `unknown scope` | `ErrUnknownScope` |
| `invalid email` | `ErrInvalidEmail` |
| `invalid display name` | `ErrInvalidDisplayName` |
| `invalid org name` | `ErrInvalidOrgName` |
| `invalid limit` | `ErrInvalidLimit` |
| `usage limit exceeded` | `ErrLimitExceeded` |
| `org suspended` | `ErrOrgSuspended` |
| `invalid credential name` | `ErrInvalidCredentialName` |
| `invalid SSH key` | `ErrInvalidSSHKey` |
| `invalid env var` | `ErrInvalidEnvVar` |
| `reserved env var name` | `ErrReservedEnvVar` |
