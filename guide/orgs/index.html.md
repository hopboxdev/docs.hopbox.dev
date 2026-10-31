# Work in a team org

> How to share an org with a team, and how the operator sets the limits of an org and suspends it.

Every account has a personal org. A team needs an org that several accounts share, with roles. This guide creates an org, invites a second person by email, and gives roles. Then, as the operator, it lists the orgs, sets the limits of the org, and suspends it. [The org API](/reference/org-api/index.html.md) has every call and rule.

This guide calls the API with curl, in the same way as the console does, so it shows each call. The console pages of an org do the same work in a browser ([Use the console](/guide/console/index.html.md#more-pages)).

## What you need

- `hopboxd` with the control role on a laptop, as in [Run the control role on a laptop](/guide/run-hopboxd/index.html.md), with the API on `127.0.0.1:7443`.
- A domain and an SMTP config, as in [First sign-in on a new install](/guide/first-sign-in/index.html.md#turn-on-magic-links). An invitation is for one verified email, so each person signs in with a magic link. A local mail catcher such as [Mailpit](https://mailpit.axllent.org) works.
- Two accounts, user1 and user2, each signed in with a magic link of [First sign-in on a new install](/guide/first-sign-in/index.html.md#turn-on-magic-links). In the default sign-up mode `approval`, an instance admin approves each new account first ([Use the Admin area](/guide/admin-area/index.html.md#approve-or-reject-new-accounts)). Keep the session cookie of each one in a file: `/tmp/hb/user1` and `/tmp/hb/user2`, each with one line of the form `__Host-hopbox_session=<token>`.

Define a function that calls the API as one of them, with the headers of the console:

```sh
API=http://127.0.0.1:7443
as() {
  who=$1; shift
  curl -s -X POST -H 'Content-Type: application/json' -H 'Connect-Protocol-Version: 1' \
    -H 'Origin: https://example.com' -H "Cookie: $(cat /tmp/hb/$who)" "$@"
  echo
}
admin() { ./bin/darwin_arm64/hopboxd admin --config /tmp/hb/hopboxd.toml "$@"; }
```

## Create an org

`CreateOrg` makes an org with you as its owner. A create call takes an idempotency key, so a retry with the same key returns the same org:

```sh
as user1 -H "Idempotency-Key: $(uuidgen)" -d '{"name":"org1"}' $API/hopbox.v1.OrgService/CreateOrg
```

```json
{"org":{"id":"01a14565-a2df-75a1-9401-ed28dfa37fc4","name":"org1","role":"ORG_ROLE_OWNER","version":"1","createTime":"2026-10-16T15:47:31.166821Z"}}
```

Keep the ID:

```sh
ORG=01a14565-a2df-75a1-9401-ed28dfa37fc4
```

`ListOrgs` now shows the personal org of user1 first, then org1:

```sh
as user1 -d '{}' $API/hopbox.v1.OrgService/ListOrgs
```

A call that leaves out `orgId` acts in the personal org of the caller. A personal org belongs to one account, so it takes no invitations.

## Invite a person

An owner invites with any role. An admin invites members only. An admin or an owner can change what runs in the private machines of every member ([Read the members and the roles](#read-the-members-and-the-roles)). So give these roles only to a person whom every member trusts. Invite user2 as an admin:

```sh
as user1 -d "{\"orgId\":\"$ORG\",\"email\":\"user2@example.org\",\"role\":\"ORG_ROLE_ADMIN\"}" \
  $API/hopbox.v1.OrgService/CreateInvitation
```

```json
{"invitation":{"id":"01a14565-a316-74dd-968e-fac72d706827","orgId":"01a14565-a2df-75a1-9401-ed28dfa37fc4","email":"user2@example.org","role":"ORG_ROLE_ADMIN","inviterAccountId":"01a14565-8448-774f-9317-ea6348e9a930","createTime":"2026-10-16T15:47:31.221754Z","expireTime":"2026-10-23T15:47:31.221754Z"},"url":"https://example.com/invite#token=vH_2434xeosR22NllAU0tHE8LoqC08ZYHX1JhHZK4UA","emailSent":true}
```

The response holds the link, and `emailSent` says that the server mailed it to user2 too. Without an SMTP config, `emailSent` is `false`, and you send the link to user2 yourself. The same goes for the file mail transport of a dev install ([Mail](/reference/mail/index.html.md)). The link works once, for 7 days. Treat it as a password until then. A new invitation for the same email replaces this one, so its link stops working.

## Accept the invitation

On an install with a console, user2 opens the link in a browser. The console page `/invite` reads the token after `#token=` and calls `AcceptInvitation`. Here, make that call yourself. The invitation is for `user2@example.org`, so user1 cannot use it:

```sh
TOKEN=vH_2434xeosR22NllAU0tHE8LoqC08ZYHX1JhHZK4UA
as user1 -d "{\"token\":\"$TOKEN\"}" $API/hopbox.v1.OrgService/AcceptInvitation
```

```json
{"code":"failed_precondition","message":"failed precondition: the invitation is for user2@example.org, and the account has no verified email user2@example.org. Sign in with that email, then open the link again"}
```

The refused try leaves the invitation open. Then user2 accepts it:

```sh
as user2 -d "{\"token\":\"$TOKEN\"}" $API/hopbox.v1.OrgService/AcceptInvitation
```

```json
{"org":{"id":"01a14565-a2df-75a1-9401-ed28dfa37fc4","name":"org1","role":"ORG_ROLE_ADMIN","version":"1","createTime":"2026-10-16T15:47:31.166821Z"}}
```

A second accept of the same link fails with `not_found`: the link works once.

## Read the members and the roles

Each member reads the members of the org:

```sh
as user2 -d "{\"orgId\":\"$ORG\"}" $API/hopbox.v1.OrgService/ListOrgMembers
```

The members of an org trust its admins and owners. An admin or an owner sets the env vars and secrets with the scope `all`, and these reach the private machines of every member ([Set env vars and secrets](/guide/env-vars/index.html.md#trust-the-admins-of-your-org)). An admin cannot open a shell in the private machine of another member, but can change what runs there with such a value.

Only an owner makes other owners and admins. So user2, an admin, cannot change a role:

```sh
as user2 -d "{\"orgId\":\"$ORG\",\"accountId\":\"<the ID of user1>\",\"role\":\"ORG_ROLE_MEMBER\"}" \
  $API/hopbox.v1.OrgService/SetOrgMemberRole
```

```json
{"code":"permission_denied","message":"permission denied: a caller of kind session may not do org.member_set_role"}
```

Every org keeps at least one owner. So user1, the only owner, cannot leave yet:

```sh
as user1 -d "{\"orgId\":\"$ORG\"}" $API/hopbox.v1.OrgService/LeaveOrg
```

```json
{"code":"failed_precondition","message":"failed precondition: the account is the only owner of the org, and every org keeps at least one owner. Make another member an owner first, or delete the org"}
```

To leave, user1 first makes user2 an owner with `SetOrgMemberRole` and the role `ORG_ROLE_OWNER`. An account that leaves an org loses what it made in the org and did not share.

## Read the usage and the limits

```sh
as user2 -d "{\"orgId\":\"$ORG\"}" $API/hopbox.v1.OrgService/GetOrgUsage
```

```json
{"usage":{},"limits":{"machines":3,"vcpus":6,"memoryBytes":"25769803776","diskBytes":"300000000000"}}
```

The org has no machines yet, so it uses nothing. The limits are the defaults of the install: 3 machines, 6 vCPUs, 24 GiB of RAM, and 300 GB of disk.

## Manage the orgs as the operator

An instance admin sees the metadata of every org, but not its members or its resources. Root on a control server calls the admin API with `hopboxd admin`:

```sh
admin org list
```

```text
01a14565-8448-7753-9c1f-5ec0a86b8d92	Personal	personal	members=1	active	machines=0/3
01a14565-85d5-7087-a903-d66b5cabce88	Personal	personal	members=1	active	machines=0/3
01a14565-a2df-75a1-9401-ed28dfa37fc4	org1	org	members=2	active	machines=0/3
```

Give org1 more room. A limit that you leave out takes the default of the install:

```sh
admin org set-limits --org-id $ORG --machines 10 --memory-gib 64
```

```text
id	01a14565-a2df-75a1-9401-ed28dfa37fc4
name	org1
members	2
state	active
machines	0/10
vcpus	0/6
memory_bytes	0/68719476736
disk_bytes	0/300000000000
version	2
```

Change the defaults of every other org with `admin org default-limits set`, for example `--machines 5`. Each new org brings its own limits, so an account owns at most 10 orgs besides its personal org. Change that cap with `admin org default-limits set --max-orgs-per-account N`.

To handle abuse, suspend the org:

```sh
admin org suspend --org-id $ORG --reason 'spam reports'
```

The members see the suspension and its reason:

```sh
as user1 -d "{\"orgId\":\"$ORG\"}" $API/hopbox.v1.OrgService/GetOrg
```

```json
{"org":{"id":"01a14565-a2df-75a1-9401-ed28dfa37fc4","name":"org1","role":"ORG_ROLE_OWNER","suspended":true,"suspendTime":"2026-10-16T15:47:41.931248Z","suspendReason":"spam reports","version":"3","createTime":"2026-10-16T15:47:31.166821Z"}}
```

A suspended org sends no invitations and takes no new members. Reads still work. A suspension also stops the machines of the org, and blocks the create, the start, and the delete of a machine ([the org API](/reference/org-api/index.html.md#suspension)). The owners find the entry `org.suspend` of `local_root` in the audit log of the org:

```sh
as user1 -d "{\"orgId\":\"$ORG\",\"pageSize\":1}" $API/hopbox.v1.AuditService/ListAuditEntries
```

End the suspension:

```sh
admin org unsuspend --org-id $ORG
```

## Delete the org

Only an owner deletes an org, and the request must hold its name in `confirm`. A suspended org keeps its data as evidence, so the delete waits for the end of the suspension:

```sh
as user1 -d "{\"orgId\":\"$ORG\",\"confirm\":\"org1\"}" $API/hopbox.v1.OrgService/DeleteOrg
```

```json
{}
```

The delete removes the memberships, the invitations, and every resource of the org. A personal org goes only with its account ([the sign-in, account, and audit API](/reference/auth-api/index.html.md#deleteaccount)).
