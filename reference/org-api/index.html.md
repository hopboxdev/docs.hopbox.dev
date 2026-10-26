# The org API

> The org calls of the public API, OrgService: orgs, roles, members, invitations, usage limits, the suspension, and their audit entries.

The service `hopbox.v1.OrgService` of the public API manages the orgs of the caller, their members, their invitations, and reads their usage and limits. [The admin API](/reference/admin-api/index.html.md#orgservice) covers the calls of instance admins on orgs: the list of every org, the suspension, and the limits. [The API server](/reference/api/index.html.md) covers the parts that every call shares: the request gate, the policy check, the idempotency keys, and the error codes. The generated reference is [Public API](/api/hopbox.v1.openapi.yaml). The code is in `internal/service` (`orgs.go`, `invitations.go`) and `internal/api`.

| Calls | What they do |
| --- | --- |
| `CreateOrg`, `GetOrg`, `ListOrgs`, `UpdateOrg`, `DeleteOrg` | Make, read, rename, and delete orgs. |
| `GetOrgUsage` | Reads what an org uses, and its limits. |
| `ListOrgMembers`, `SetOrgMemberRole`, `RemoveOrgMember`, `LeaveOrg` | Read and change the members of an org. |
| `CreateInvitation`, `ListInvitations`, `RevokeInvitation`, `AcceptInvitation` | Invite a person by email, and join with the invitation. |

Each call acts as the account of a `session` or `cli_token` caller. A `member_key` acts as its account too, but only in the org of the key, and it never creates, joins, or leaves an org. An `org_key` has the rights of a member in its own org, without personal resources: it reads the org, its usage, and its members ([CLI tokens, API keys, admin API keys, and SSH keys](/reference/credentials/index.html.md#api-keys)). An instance admin sees the orgs of others only through the admin API. Here, a session of an instance admin has only the rights of its own roles.

The examples on this page use the shell function `browser` of [the sign-in, account, and audit API](/reference/auth-api/index.html.md), with the session cookie of the caller. [Work in a team org](/guide/orgs/index.html.md) runs the whole flow on a laptop.

## The org of a request

A request that acts in one org has the field `org_id`, an optional string. Leave it out to act in the personal org of the caller ([the API server](/reference/api/index.html.md#callers-and-policy)). `ListOrgs`, `CreateOrg`, and `AcceptInvitation` have no `org_id`.

The service layer reads the role of the caller in the org before the policy check. A caller that is not a member has no role. So an org that does not exist gets the same `permission_denied` as an org of others. An `org_id` that is not a UUID gets `invalid_argument`.

## Roles

The enum `OrgRole` names the role of an account in an org ([Domain rules](/reference/domain-rules/index.html.md#org-roles)). Each role has every right of the roles before it.

| Value | Role | Default |
| --- | --- | --- |
| `ORG_ROLE_UNSPECIFIED` | Never in a response. In a request, the field takes its default. | |
| `ORG_ROLE_MEMBER` | `member` | the role of an invitation without a role |
| `ORG_ROLE_ADMIN` | `admin` | |
| `ORG_ROLE_OWNER` | `owner` | the role of the account that creates the org |

A value that is not in the enum gets `invalid_argument` with the message `invalid argument: the org role <n> is not known`.

### What each role can do

| Call | `member` | `admin` | `owner` |
| --- | --- | --- | --- |
| `GetOrg`, `GetOrgUsage`, `ListOrgMembers`, `LeaveOrg` | yes | yes | yes |
| `ListInvitations`, `RevokeInvitation` | no | yes | yes |
| `CreateInvitation` with the role `member` | no | yes | yes |
| `CreateInvitation` with the role `admin` or `owner` | no | no | yes |
| `RemoveOrgMember` of a `member` | no | yes | yes |
| `RemoveOrgMember` of an `admin` or an `owner` | no | no | yes |
| `SetOrgMemberRole`, `UpdateOrg`, `DeleteOrg` | no | no | yes |

Only an owner makes other owners and admins. So an admin invites and removes members only. A caller without the right gets `permission_denied`, with the message `permission denied: a caller of kind session may not do <action>`.

### Every org keeps an owner

An org always has at least one owner. If a change takes the last owner from the org, it fails with `failed_precondition`. These changes can do so: `SetOrgMemberRole` of the last owner to another role, `RemoveOrgMember` of the last owner, `LeaveOrg` by the last owner, and `DeleteAccount` of the last owner ([the sign-in, account, and audit API](/reference/auth-api/index.html.md#deleteaccount)).

Each such change first locks the row of the org in `orgs`. Then it reads the role of the caller again, checks the policy again, and counts the owners in a new statement ([Control store tables](/reference/control-store/index.html.md#orgs)). So two owners that each step down, leave, or remove the other at the same time leave one owner. The second change sees the first one, and fails with `failed_precondition` or `permission_denied`. The delete of an account locks every org of the account in the same way. So a promotion of the account to owner during its delete waits for the delete, and then fails with `not_found`.

## Messages

### Org

| Field | Type | Description |
| --- | --- | --- |
| `id` | string | The UUID of the org. |
| `name` | string | The name of the org ([Domain rules](/reference/domain-rules/index.html.md#org-names)). A personal org starts with the name `Personal`, and its owner can rename it. |
| `personal` | bool | `true` for the personal org of an account. |
| `role` | `OrgRole` | The role of the caller in the org. |
| `suspended` | bool | `true` while an instance admin suspends the org ([Suspension](#suspension)). |
| `suspend_time` | `google.protobuf.Timestamp` | The start of the suspension. Unset for an org that is not suspended. |
| `suspend_reason` | string | The reason that the instance admin gave, or empty. |
| `version` | int64 | The version of the org. It starts at 1 and grows by 1 with each change, such as a rename, a suspension, or new limits. |
| `create_time` | `google.protobuf.Timestamp` | The creation time of the org. |

### OrgMember

| Field | Type | Description |
| --- | --- | --- |
| `account_id` | string | The UUID of the account. |
| `display_name` | string | The display name of the account. |
| `emails` | repeated `AccountEmail` | The emails of the account ([the sign-in, account, and audit API](/reference/auth-api/index.html.md#accountemail)). |
| `role` | `OrgRole` | The role of the account in the org. |
| `join_time` | `google.protobuf.Timestamp` | The time of the join. |

### OrgLimits and OrgUsage

`OrgLimits` holds the usage limits of an org, and `OrgUsage` holds what the org uses ([Usage limits](#usage-limits)). Both have the same fields.

| Field | Type | Description |
| --- | --- | --- |
| `machines` | int32 | The machines. |
| `vcpus` | int32 | The vCPUs of all machines together. |
| `memory_bytes` | int64 | The RAM of all machines together, in bytes. |
| `disk_bytes` | int64 | The disk of all machines together, in bytes. |

### Invitation

| Field | Type | Description |
| --- | --- | --- |
| `id` | string | The UUID of the invitation. |
| `org_id` | string | The UUID of the org. |
| `email` | string | The email that the invitation is for, in lower case. |
| `role` | `OrgRole` | The role that the invitation gives. |
| `inviter_account_id` | string | The UUID of the account that made the invitation, or empty after the delete of that account. |
| `create_time` | `google.protobuf.Timestamp` | The time of the invitation. |
| `expire_time` | `google.protobuf.Timestamp` | The end of the invitation, 7 days after `create_time`. |

## Orgs

### CreateOrg

Makes an org with the caller as its owner.

| Item | Value |
| --- | --- |
| Procedure | `/hopbox.v1.OrgService/CreateOrg` |
| Policy action | `org.create`: `session` and `cli_token` |
| Idempotent option | yes. A call must have an `Idempotency-Key` header ([the API server](/reference/api/index.html.md#idempotency-keys)). A retry with the same key returns the same org. |
| Audit entry | action `org.create`, org the new org, target kind `org`, details `name` |

The request `CreateOrgRequest` has the field `name`, a string ([Domain rules](/reference/domain-rules/index.html.md#org-names)). The response `CreateOrgResponse` has the field `org`, an `Org`.

An account owns at most 10 orgs besides its personal org. Every new org brings its own usage limits, so this cap keeps one account from many limits. An instance admin changes the cap with `SetDefaultOrgLimits` ([the admin API](/reference/admin-api/index.html.md#setdefaultorglimits)). The cap counts each org in which the account is an owner. The call takes the row lock of the account, so two creates at the same time cannot pass the cap together.

| Error | Code |
| --- | --- |
| A bad name, for example `invalid argument: invalid org name: it must not start or end with a space` | `invalid_argument` |
| The account owns the most orgs: `usage limit exceeded: the account owns 10 orgs besides its personal org, the most that the install allows. Delete one, or ask an instance admin to raise the limit` | `resource_exhausted` |

```sh
browser -H 'Idempotency-Key: 6f1c2f0e-team' -d '{"name":"Team"}' $API/hopbox.v1.OrgService/CreateOrg
```

```json
{"org":{"id":"01a14565-a2df-75a1-9401-ed28dfa37fc4","name":"Team","role":"ORG_ROLE_OWNER","version":"1","createTime":"2026-10-16T15:47:31.166821Z"}}
```

### GetOrg

Returns one org of the caller.

| Item | Value |
| --- | --- |
| Procedure | `/hopbox.v1.OrgService/GetOrg` |
| Policy action | `org.get`: a `session`, `cli_token`, or `member_key` with a role in the org, or an `org_key` of the org |
| Idempotent option | no |
| Audit entry | none |

The request `GetOrgRequest` has the field `org_id`. The response `GetOrgResponse` has the field `org`.

### ListOrgs

Lists the orgs of the caller: the personal org first, then the others, oldest first.

| Item | Value |
| --- | --- |
| Procedure | `/hopbox.v1.OrgService/ListOrgs` |
| Policy action | `org.list`: the `session` or `cli_token` of the account |
| Idempotent option | no |
| Audit entry | none |

The request `ListOrgsRequest` has no fields. The response `ListOrgsResponse` has the field `orgs`, a repeated `Org`.

```sh
browser -d '{}' $API/hopbox.v1.OrgService/ListOrgs
```

```json
{"orgs":[{"id":"01a14565-8448-7753-9c1f-5ec0a86b8d92","name":"Personal","personal":true,"role":"ORG_ROLE_OWNER","version":"1","createTime":"2026-10-16T15:47:23.335394Z"},{"id":"01a14565-a2df-75a1-9401-ed28dfa37fc4","name":"Team","role":"ORG_ROLE_OWNER","version":"1","createTime":"2026-10-16T15:47:31.166821Z"}]}
```

### UpdateOrg

Changes the name of an org, and returns the org after the change.

| Item | Value |
| --- | --- |
| Procedure | `/hopbox.v1.OrgService/UpdateOrg` |
| Policy action | `org.update`: a `session`, `cli_token`, or `member_key` with the role `owner` |
| Idempotent option | no |
| Audit entry | action `org.update`, target kind `org`, details `name` and `old_name` |

| Field | Type | Description |
| --- | --- | --- |
| `org_id` | optional string | The org. |
| `name` | optional string | The new name. Leave it out to keep the name. |
| `expected_version` | optional int64 | The `version` of the org that the caller read. If the org has another version now, the call fails with `aborted` and changes nothing. |

A request that sets no field gets `invalid_argument` with the message `invalid argument: the update sets no field`. The response `UpdateOrgResponse` has the field `org`.

### DeleteOrg

Deletes an org, its memberships, its invitations, and every resource in it. The request must hold the name of the org in `confirm`.

| Item | Value |
| --- | --- |
| Procedure | `/hopbox.v1.OrgService/DeleteOrg` |
| Policy action | `org.delete`: a `session`, `cli_token`, or `member_key` with the role `owner` |
| Idempotent option | no |
| Audit entry | action `org.delete`, org the deleted org, target kind `org`, details `name` |

| Field | Type | Description |
| --- | --- | --- |
| `org_id` | optional string | The org. |
| `confirm` | string | The name of the org, exactly. |

The response `DeleteOrgResponse` has no fields. The audit log keeps the entries of the deleted org, and instance admins read them.

| Error | Code |
| --- | --- |
| `confirm` is not the name: `invalid argument: confirm the delete with the name of the org, "Team"` | `invalid_argument` |
| The personal org: `failed precondition: a personal org goes only with its account. Delete the account instead` | `failed_precondition` |
| A suspended org: `failed precondition: org suspended: an instance admin suspended the org, and nothing deletes its data until the suspension ends. Ask for the end of the suspension first` | `failed_precondition` |

A suspension keeps the org and its resources, for example as the evidence of abuse. So the owners delete the org only after an instance admin ends the suspension.

```sh
browser -d '{"orgId":"01a14565-a2df-75a1-9401-ed28dfa37fc4","confirm":"Team"}' $API/hopbox.v1.OrgService/DeleteOrg
```

### GetOrgUsage

Returns what an org uses, and its limits ([Usage limits](#usage-limits)).

| Item | Value |
| --- | --- |
| Procedure | `/hopbox.v1.OrgService/GetOrgUsage` |
| Policy action | `org.get_usage`: a `session`, `cli_token`, or `member_key` with a role in the org, or an `org_key` of the org |
| Idempotent option | no |
| Audit entry | none |

The request `GetOrgUsageRequest` has the field `org_id`. The response `GetOrgUsageResponse` has the fields `usage`, an `OrgUsage`, and `limits`, an `OrgLimits`.

```sh
browser -d '{"orgId":"01a14565-a2df-75a1-9401-ed28dfa37fc4"}' $API/hopbox.v1.OrgService/GetOrgUsage
```

```json
{"usage":{},"limits":{"machines":3,"vcpus":6,"memoryBytes":"25769803776","diskBytes":"300000000000"}}
```

In the JSON codec, a field with its default value is left out, so a usage of 0 is `{}`.

## Members

### ListOrgMembers

Lists the members of an org, in the order that they joined.

| Item | Value |
| --- | --- |
| Procedure | `/hopbox.v1.OrgService/ListOrgMembers` |
| Policy action | `org.member_list`: a `session`, `cli_token`, or `member_key` with a role in the org, or an `org_key` of the org |
| Idempotent option | no |
| Audit entry | none |

The request `ListOrgMembersRequest` has the field `org_id`. The response `ListOrgMembersResponse` has the field `members`, a repeated `OrgMember`. For the other members, `emails` holds only their verified emails, because nobody proved an unverified email. For the caller, it holds every email.

### SetOrgMemberRole

Changes the role of a member. A change to the role that the member has already changes nothing and succeeds.

| Item | Value |
| --- | --- |
| Procedure | `/hopbox.v1.OrgService/SetOrgMemberRole` |
| Policy action | `org.member_set_role`: a `session`, `cli_token`, or `member_key` with the role `owner` |
| Idempotent option | no |
| Audit entry | action `org.member_set_role`, target kind `account`, details `role` and `old_role` |

| Field | Type | Description |
| --- | --- | --- |
| `org_id` | optional string | The org. |
| `account_id` | string | The UUID of the member. |
| `role` | `OrgRole` | The new role. |

The response `SetOrgMemberRoleResponse` has no fields.

| Error | Code |
| --- | --- |
| `account_id` is not a UUID, or `role` is `ORG_ROLE_UNSPECIFIED` or not known: `invalid argument: unknown org role: "". The role must be member, admin, or owner` | `invalid_argument` |
| The account is not a member: `not found: the account <id> is not a member of the org` | `not_found` |
| The last owner: `failed precondition: the account is the only owner of the org, and every org keeps at least one owner. Make another member an owner first` | `failed_precondition` |

The call checks `account_id` and `role` before it reads the org. So a bad value gets `invalid_argument`, also in an org that does not exist, or of which the caller is not a member. The check reads only the request, so it tells nothing about the org. An anonymous caller still gets `unauthenticated` first.

### RemoveOrgMember

Takes another account out of an org, with what it made in the org and did not share.

| Item | Value |
| --- | --- |
| Procedure | `/hopbox.v1.OrgService/RemoveOrgMember` |
| Policy action | `org.member_remove`: a `session`, `cli_token`, or `member_key` with the role `admin` for a `member`, or with the role `owner` for anyone |
| Idempotent option | no |
| Audit entry | action `org.member_remove`, target kind `account`, details `role` and `reason` (`removed`) |

The request `RemoveOrgMemberRequest` has the fields `org_id` and `account_id`. The response `RemoveOrgMemberResponse` has no fields.

| Error | Code |
| --- | --- |
| The caller names its own account: `invalid argument: to take your own account out of the org, leave it` | `invalid_argument` |
| The account is not a member, or no account has the ID. Both get the same message, `not found: the account <id> is not a member of the org`, so the call tells nothing about which accounts exist. | `not_found` |
| A suspended org, with the message of `DeleteOrg` | `failed_precondition` |
| The last owner | `failed_precondition` |

### LeaveOrg

Takes the account of the caller out of an org, with what it made in the org and did not share.

| Item | Value |
| --- | --- |
| Procedure | `/hopbox.v1.OrgService/LeaveOrg` |
| Policy action | `org.leave`: a `session` or `cli_token` with a role in the org |
| Idempotent option | no |
| Audit entry | action `org.leave`, target kind `account`, details `role` |

The request `LeaveOrgRequest` has the field `org_id`. The response `LeaveOrgResponse` has no fields.

| Error | Code |
| --- | --- |
| The personal org: `failed precondition: an account cannot leave its personal org` | `failed_precondition` |
| The last owner: `failed precondition: the account is the only owner of the org, and every org keeps at least one owner. Make another member an owner first, or delete the org` | `failed_precondition` |
| A suspended org, with the message of `DeleteOrg` | `failed_precondition` |

### What a member leaves behind

A leave, a removal, and the delete of an account take the account out of the org. They also delete what the account made in the org and did not share. The delete of an org deletes every resource of the org. The service layer does this through the Go interface `service.OrgResource`, in the transaction of the change:

| Method | Called by |
| --- | --- |
| `MemberLeaving(org, account)` | `LeaveOrg`, `RemoveOrgMember`, and `DeleteAccount`, once for each org other than the personal org |
| `OrgDeleting(org)` | `DeleteOrg`, and `DeleteAccount` for the personal org |
| `OrgSuspending(org)` | `SuspendOrg` of the admin API ([the admin API](/reference/admin-api/index.html.md#suspendorg)) |

Each call holds the row lock of the org while the method runs. A leave, a removal, and an account delete also hold the lock of the account, which they take first. The lock order is the account row, then the org row, then the rows of the resource, such as machines ([Control store tables](/reference/control-store/index.html.md#orgs)). A change of a resource outside these calls takes the locks in the same order.

The env vars and secrets are built into the service layer, and it runs their methods first. `MemberLeaving` deletes the private values of the account in the org, with a `var.delete` entry for each ([the env var API](/reference/vars-api/index.html.md#what-a-member-leaves-behind)). The other values of the org stay. The delete of an org deletes its values through the foreign key. Each other kind of resource in an org adds its implementation to `Service.OrgResources`. The control role adds the machines ([the machine API](/reference/machine-api/index.html.md)). A method that fails rolls the whole change back, and the call fails with `internal`.

The machines do this in each method. Each method locks the machine rows in the order of their IDs, and skips a machine whose target is `deleted` already.

| Method | What it does to the machines | Audit entry of each machine that changes |
| --- | --- | --- |
| `MemberLeaving(org, account)` | Sets the target `deleted` on each machine that the account made in the org and did not share. A shared machine stays in the org. | `machine.delete`, with the detail `reason` (`member left`) |
| `OrgDeleting(org)` | Sets the target `deleted` on every machine of the org. | `machine.delete`, with the detail `reason` (`org deleted`) |
| `OrgSuspending(org)` | Sets the target `stopped` on every machine of the org, except a machine whose target is `hibernated`. A hibernated machine stays hibernated, because only a delete drops its memory file ([Domain rules](/reference/domain-rules/index.html.md#org-suspension)). | `machine.stop`, with the details `old_target` and `reason` (`org suspended`) |

A delete reserves the name and the SSH port of the machine for 7 days, as `DeleteMachine` does ([the machine API](/reference/machine-api/index.html.md#deletemachine)). The actor of each entry is the caller of the change, and each change publishes the event `machine.updated` ([the machine API](/reference/machine-api/index.html.md#change-events)). A delete of an org sets `org_id` of its machines to null once their target is `deleted`. The machines then wait for their executor ([Control store tables](/reference/control-store/index.html.md#machines)).

## Invitations

An org owner or an org admin invites a person by email, and the person accepts after the sign-in. An invitation is for one email and gives one role.

| Rule | Value |
| --- | --- |
| Link | `https://<domain>/invite#token=<token>`. The token is 32 random bytes in base64url, as for a sign-in link ([Sessions, cookies, and sign-in links](/reference/sessions/index.html.md#tokens)). It is in the fragment, so a browser never sends it to a server. The console page `/invite` reads it and calls `AcceptInvitation`. |
| Storage | The control store keeps only the SHA-256 of the token ([Control store tables](/reference/control-store/index.html.md#org_invitations)). The audit log never holds the token. |
| Lifetime | 7 days |
| Use | once |
| Open invitations | one for each email in an org. A new invitation for the email revokes the open one, so its link stops working. |
| Rate limits | An org makes at most 20 invitations in an hour. An inviting account makes at most 20 in an hour, in all its orgs together. An email gets at most 5 in an hour, from every org together. The control store counts them in `org_invitation_sends`, a table without foreign keys ([Control store tables](/reference/control-store/index.html.md#org_invitation_sends)). So the limits hold across every control server, and the delete of an org or an account resets no limit. Each new invitation takes a transaction lock (`LockKeyInvitations`, [the bundled Postgres](/reference/postgres/index.html.md#advisory-lock-keys)), so invitations at the same time cannot pass a limit together. |
| Email | With an SMTP config, the server sends the link to the email ([SMTP config](/reference/smtp/index.html.md)). Without one, only the inviter gets the link. With the file mail transport, the server writes the email to a file on the server, and the inviter passes the link on ([Mail](/reference/mail/index.html.md#invitations)). |
| Who accepts | A `session` whose account holds the email of the invitation as a verified email. The email of a login link stays unverified, so it does not count ([Sessions, cookies, and sign-in links](/reference/sessions/index.html.md#what-a-sign-in-does)). A link of the email from a session of the account verifies it ([Sessions, cookies, and sign-in links](/reference/sessions/index.html.md#link-a-new-sign-in-method)). |
| Orgs | A personal org belongs to one account, so it takes no invitations. A suspended org sends none and takes no new members. |
| Rights of the inviter | An invitation never gives more than its inviter can give now. A leave, a removal, or an account delete of the inviter revokes each open invitation of the inviter in the org. A role change of the inviter revokes each one with a role that the new role cannot give. |

The leader deletes an invitation 24 hours after its end: its use, its revoke, or its expiry ([hopboxd](/reference/hopboxd/index.html.md#leader-jobs)).

### CreateInvitation

Invites an email to join an org with a role, and returns the link. With an SMTP config, it also sends the link to the email.

| Item | Value |
| --- | --- |
| Procedure | `/hopbox.v1.OrgService/CreateInvitation` |
| Policy action | `org.invitation_create`: a `session`, `cli_token`, or `member_key` with the role `admin` for the role `member`, or with the role `owner` for any role |
| Idempotent option | no. The response holds the link, which must never stay in the table of idempotency keys. A retry revokes the invitation of the first try and makes a new one. |
| Audit entries | action `org.invitation_create`, target kind `invitation`, details `email` and `role`. A replaced invitation gets the action `org.invitation_revoke` with the details `email`, `role`, and `reason` (`replaced`). |

| Field | Type | Default | Description |
| --- | --- | --- | --- |
| `org_id` | optional string | the personal org, which the call refuses | The org. |
| `email` | string | none, required | The email of the person, such as `bob@example.org` ([Domain rules](/reference/domain-rules/index.html.md#emails)). |
| `role` | `OrgRole` | `ORG_ROLE_MEMBER` | The role that the invitation gives. |

The response `CreateInvitationResponse` has these fields:

| Field | Type | Description |
| --- | --- | --- |
| `invitation` | `Invitation` | The new invitation. |
| `url` | string | The link. It is a secret until its use or its end. |
| `email_sent` | bool | `true` after a sent email. It is `false` without an SMTP config, after a failed email, and with the file mail transport, because that email reaches nobody. |

The server sends the email after the invitation commits, and waits at most 20 s for the SMTP server. A failed email does not fail the call, because the inviter has the link. The server then logs `the invitation email failed` at level error, with `component=service`, `invitation`, and `err`.

The email has this form:

```text
From: hopbox <no-reply@example.org>
To: bob@example.org
Subject: Join Team on example.com

You are invited to join the org "Team" on example.com as admin.

Open this link, and sign in with the email bob@example.org to accept:

https://example.com/invite#token=<token>

The link works once, until Fri, 23 Oct 2026 15:47:31 UTC. If you did not expect it, you can ignore this email.
```

| Error | Code |
| --- | --- |
| The email breaks a rule, or the role is not known | `invalid_argument` |
| The install has no domain: `failed precondition: the install has no domain, and the invitation link needs one` | `failed_precondition` |
| The personal org: `failed precondition: a personal org belongs to one account, so it takes no invitations` | `failed_precondition` |
| A suspended org: `failed precondition: org suspended: an instance admin suspended the org, so it sends no invitations` | `failed_precondition` |
| An account with the email as a verified email is a member already | `failed_precondition` |
| The org made 20 invitations in the last hour: `too many requests: this org made the most invitations of an hour. Wait, then retry` | `resource_exhausted` |
| The caller made 20 invitations in the last hour, in any orgs: `too many requests: this account made the most invitations of an hour. Wait, then retry` | `resource_exhausted` |
| The email got 5 invitations in the last hour, from any orgs: `too many requests: this email got the most invitations of an hour. Wait, then retry` | `resource_exhausted` |
| The SMTP password does not open, for example after the loss of its key-encryption key | `internal` |

```sh
browser -d '{"orgId":"01a14565-a2df-75a1-9401-ed28dfa37fc4","email":"bob@example.org","role":"ORG_ROLE_ADMIN"}' \
  $API/hopbox.v1.OrgService/CreateInvitation
```

```json
{"invitation":{"id":"01a14565-a316-74dd-968e-fac72d706827","orgId":"01a14565-a2df-75a1-9401-ed28dfa37fc4","email":"bob@example.org","role":"ORG_ROLE_ADMIN","inviterAccountId":"01a14565-8448-774f-9317-ea6348e9a930","createTime":"2026-10-16T15:47:31.221754Z","expireTime":"2026-10-23T15:47:31.221754Z"},"url":"https://example.com/invite#token=vH_2434xeosR22NllAU0tHE8LoqC08ZYHX1JhHZK4UA","emailSent":true}
```

### ListInvitations

Lists the open invitations of an org, oldest first. An open invitation is not used, not revoked, and not expired.

| Item | Value |
| --- | --- |
| Procedure | `/hopbox.v1.OrgService/ListInvitations` |
| Policy action | `org.invitation_list`: a `session`, `cli_token`, or `member_key` with the role `admin` or `owner` |
| Idempotent option | no |
| Audit entry | none |

The request `ListInvitationsRequest` has the field `org_id`. The response `ListInvitationsResponse` has the field `invitations`, a repeated `Invitation`. It never holds a link.

### RevokeInvitation

Ends an open invitation, so its link stops working.

| Item | Value |
| --- | --- |
| Procedure | `/hopbox.v1.OrgService/RevokeInvitation` |
| Policy action | `org.invitation_revoke`: a `session`, `cli_token`, or `member_key` with the role `admin` or `owner`. An admin also revokes an invitation that gives the role `owner`, because a revoke gives nobody a right. |
| Idempotent option | no |
| Audit entry | action `org.invitation_revoke`, target kind `invitation`, details `email` and `role` |

The request `RevokeInvitationRequest` has the fields `org_id` and `id`, the UUID of the invitation. The response `RevokeInvitationResponse` has no fields. An invitation that the org does not have open gets `not_found` with the message `not found: the org has no open invitation <id>`.

### AcceptInvitation

Joins the org of an invitation with its role. The account of the caller must hold the email of the invitation as a verified email.

| Item | Value |
| --- | --- |
| Procedure | `/hopbox.v1.OrgService/AcceptInvitation` |
| Policy action | `org.invitation_accept`: `session` and `cli_token` |
| Idempotent option | no |
| Audit entry | action `org.invitation_accept`, org the org, target kind `invitation`, details `email` and `role` |

The request `AcceptInvitationRequest` has the field `token`: the value after `#token=` in the link. The response `AcceptInvitationResponse` has the field `org`, the `Org` with the new role of the caller.

The call locks the row of the org, then the row of the invitation, in the order that `CreateInvitation` takes them. So two accepts of one link at the same time make one membership. A refused try that is not a used, revoked, or expired link leaves the invitation open, so the right person can still accept it.

| Error | Code |
| --- | --- |
| The token is unknown, used, revoked, or expired: `not found: the invitation is not valid: it is unknown, used, revoked, or expired. Ask for a new one` | `not_found` |
| The account has no verified email of the invitation: `failed precondition: the invitation is for bob@example.org, and the account has no verified email bob@example.org. Sign in with that email, then open the link again` | `failed_precondition` |
| The account is a member already: `failed precondition: the account is a member of the org already, as <role>` | `failed_precondition` |
| A suspended org: `failed precondition: org suspended: an instance admin suspended the org, so it takes no new members` | `failed_precondition` |

The server logs a refused token as `invitation refused` at level info, with `component=service`, and never the token.

```sh
browser -d '{"token":"vH_2434xeosR22NllAU0tHE8LoqC08ZYHX1JhHZK4UA"}' $API/hopbox.v1.OrgService/AcceptInvitation
```

```json
{"org":{"id":"01a14565-a2df-75a1-9401-ed28dfa37fc4","name":"Team","role":"ORG_ROLE_ADMIN","version":"1","createTime":"2026-10-16T15:47:31.166821Z"}}
```

## Usage limits

Each org has usage limits on machines, vCPUs, RAM, and disk. The install has default limits for every org. An instance admin can change the defaults, and set limits for one org that replace the defaults ([the admin API](/reference/admin-api/index.html.md#setorglimits)).

| Limit | Default |
| --- | --- |
| Machines | 3 |
| vCPUs | 6 |
| RAM | 24 GiB (25769803776 bytes) |
| Disk | 300 GB (300000000000 bytes) |

The defaults fit three machines of the default size ([Domain rules](/reference/domain-rules/index.html.md#sizes)). A limit of 0 allows nothing. [Domain rules](/reference/domain-rules/index.html.md#usage-limits) gives the rules of the check.

| Usage | The machines that count |
| --- | --- |
| Machines and disk | every machine that is not deleted |
| vCPUs and RAM | the machines whose target state is `running` or `standby` |

So a stopped machine keeps its disk and its place in the machine count, but frees its vCPUs and its RAM. A machine whose target is `deleted` counts no more, also while its executor still deletes it. `CreateMachine` and `StartMachine` check the limits ([the machine API](/reference/machine-api/index.html.md#usage-and-limits)). A start adds only the vCPUs and the RAM of a machine that does not count already, so a start never counts a machine twice. A new limit below the usage stops nothing: the org then creates and starts no more machines until its usage fits.

## Suspension

An instance admin can suspend an org, for example to handle abuse. The calls are on the admin API ([the admin API](/reference/admin-api/index.html.md#suspendorg)). The members of the org see the suspension in `GetOrg`. Its owners and admins see its audit entry in the audit log of the org.

| Effect | Since |
| --- | --- |
| Reads still work. | now |
| The org sends no invitations. `CreateInvitation` fails with `failed_precondition`. | now |
| The org takes no new members. `AcceptInvitation` fails with `failed_precondition`. | now |
| Owners can still change roles and rename the org. | now |
| Nothing deletes the data of the org, for example the evidence of abuse. `DeleteOrg`, `LeaveOrg`, and `RemoveOrgMember` in the org fail with `failed_precondition`. So does `DeleteAccount` of a member of the org, or of the account of a suspended personal org. | now |
| The env vars and secrets of the org do not change. `SetVar` and `DeleteVar` fail with `failed_precondition` ([the env var API](/reference/vars-api/index.html.md#suspension)). | now |
| The members cannot delete a machine of the org. `DeleteMachine` fails with `failed_precondition` ([Domain rules](/reference/domain-rules/index.html.md#org-suspension)). | now |
| The machines of the org stop. A hibernated machine stays hibernated ([Domain rules](/reference/domain-rules/index.html.md#org-suspension)). | now, through `OrgSuspending` |
| The org cannot create, start, or reboot a machine. `CreateMachine`, `StartMachine`, and `RebootMachine` fail with `failed_precondition` ([the machine API](/reference/machine-api/index.html.md#suspension)). | now |
| The org cannot resume, wake, fork, or restore a machine. | with the calls of these actions, in phase 3 |
| The auto-delete of the leader skips the machines of the org. | now |
| Instance admins can still stop and delete any machine of the org ([the admin API](/reference/admin-api/index.html.md#machineservice)). | now |
| Exec and file copy are blocked ([the machine API](/reference/machine-api/index.html.md#suspension)). | now |
| The machine URLs are blocked. The edge answers `403`, and `GET /auth/machine` gives no code ([the edge role](/reference/edge/index.html.md#private-machine-urls)). | now |
| SSH is blocked. The SSH front door refuses every key, and ends the open SSH connections of the org ([the SSH front door](/reference/ssh-front-door/index.html.md#checks-of-an-open-connection)). | now |

The end of the suspension starts no machine. The users of the org start them again.

## Audit entries

The org calls write these entries in the transaction of the change. Their org is the org of the change, so its owners and admins read them with `ListAuditEntries` ([the sign-in, account, and audit API](/reference/auth-api/index.html.md#listauditentries)). The actor is the caller.

| Action | Call | Target | Details |
| --- | --- | --- | --- |
| `org.create` | `CreateOrg` | the org | `name` |
| `org.update` | `UpdateOrg` | the org | `name`, `old_name` |
| `org.delete` | `DeleteOrg` | the org | `name` |
| `org.member_set_role` | `SetOrgMemberRole` | the account | `role`, `old_role` |
| `org.member_remove` | `RemoveOrgMember`, and `DeleteAccount` for each org of the account | the account | `role`, `reason` (`removed` or `account deleted`) |
| `org.leave` | `LeaveOrg` | the account | `role` |
| `org.invitation_create` | `CreateInvitation` | the invitation | `email`, `role` |
| `org.invitation_revoke` | `RevokeInvitation`, `CreateInvitation` for the invitation that it replaces, and a leave, a removal, an account delete, or a role change of the inviter | the invitation | `email`, `role`, and for a revoke by another call `reason`: `replaced`, `inviter left`, or `inviter role changed` |
| `org.invitation_accept` | `AcceptInvitation` | the invitation | `email`, `role` |

The admin calls on orgs add `org.suspend`, `org.unsuspend`, and `org.set_limits` to the audit log of the org ([the admin API](/reference/admin-api/index.html.md#orgservice)). The machine calls write their own entries in the audit log of the org ([the machine API](/reference/machine-api/index.html.md#audit-entries)).
