# Set env vars and secrets

> How to set env vars and secrets with the hopbox CLI in each scope, list them, share a machine with them, and rewrap the secrets after a rotation of the key-encryption key.

An env var or a secret goes into the machines of an org at boot and into every login shell. You can read an env var back. A secret is sealed, and nobody can read it back. This guide sets values with each scope, lists them, shows which machines get which value, and shares a machine. [The env var API](/reference/vars-api/index.html.md) has every call and rule.

This guide uses the hopbox CLI. The tab "Env vars" of an org in the console does the same work in a browser ([Use the console](/guide/console/index.html.md#more-pages)).

## What you need

- The hopbox CLI, signed in to an install, as in [Install the hopbox CLI](/guide/install-cli/index.html.md). This guide uses the install `https://example.com`.
- For secrets: a key-encryption key on the install. The operator makes the first one with `hopboxd admin kek rotate` ([Run the control role on a laptop](/guide/run-hopboxd/index.html.md#act-through-the-admin-socket)). Without it, a secret fails with `failed_precondition`.

## Pick a scope

Each value has a scope. The scope says which machines get the value:

| Scope | Reaches | Who sets it |
| --- | --- | --- |
| `shared`, the default | the shared machines of the org | org admins and org owners |
| `private` | your own private machines in the org | any member |
| `all` | every machine of the org | org admins and org owners |

If a private value and an all value have the same name, your private machines get the private value.

## Set values in your personal org

A command without `--org` acts in the active org of the profile, at first your personal org. A personal org has no shared machines, so a value with the default scope reaches no machine. hopbox refuses it:

```sh
hopbox env set REGION=eu-west; echo "status $?"
```

```text
hopbox: invalid_argument: invalid argument: a personal org has no shared machines, so a shared value reaches no machine. Use the scope all or private
status 1
```

Set an env var for every machine of the org:

```sh
hopbox env set REGION=eu-west --scope all
```

```text
Set the env var REGION in the scope all.
```

Set a secret for your own private machines. Without `=VALUE`, the CLI asks for the value and does not show it, so the value stays out of the shell history:

```sh
hopbox env set GITHUB_TOKEN --secret --scope private
```

```text
Value of GITHUB_TOKEN: 
Set the secret GITHUB_TOKEN in the scope private.
```

A script gives the value on stdin instead, such as `hopbox env set GITHUB_TOKEN --secret --scope private < token.txt`. No later command returns the value of a secret.

A private value can have the name of an all value. Your private machines then get the private value:

```sh
hopbox env set REGION=eu-central --scope private
```

Some names, such as `EDITOR` and `NODE_OPTIONS`, make a program run code. Only a private value can take them, for your own machines:

```sh
hopbox env set EDITOR=vim --scope all; echo "status $?"
hopbox env set EDITOR=nvim --scope private
```

```text
hopbox: reserved env var name: EDITOR. A program can load or run code from it, so only a private value can take it
status 80
Set the env var EDITOR in the scope private.
```

The CLI checks the name before it sends the value. A set with the same name and scope replaces the value.

## List the values

```sh
hopbox env list
```

```text
NAME          SCOPE    SECRET  VALUE       UPDATED
EDITOR        private  no      nvim        2026-10-17T10:25:49Z
GITHUB_TOKEN  private  yes     (secret)    2026-10-17T10:25:49Z
REGION        all      no      eu-west     2026-10-17T10:25:49Z
REGION        private  no      eu-central  2026-10-17T10:25:49Z
```

The list holds the values of the org and your own private values, sorted by name. Nobody else sees your private values, also not an owner of the org.

## See what each machine gets

hopbox builds the env of each machine from these values. With the values above, your private machines in this org get:

```text
EDITOR=nvim
GITHUB_TOKEN=<your token>
REGION=eu-central
```

The guest agent writes these values to `/etc/hopbox/env`, which every login shell reads. Exec and the start command get them too. Make a machine, and read a value in a login shell:

```sh
hopbox new box1
hopbox exec box1 bash -lc 'echo "$REGION"'
```

```text
eu-central
```

[The env var API](/reference/vars-api/index.html.md#env-of-a-machine) gives the rules for every kind of machine. Delete the machine with `hopbox machine delete box1 --yes`. Its name stays reserved for your personal org for 7 days, so the next section uses `box2`.

## Share values with a team

Make an org for a team, `org1`, as in [Use the hopbox CLI](/guide/cli/index.html.md#work-in-an-org), and make it the active org:

```sh
hopbox org create org1 --use
```

As the owner, set a secret for the shared machines, and an env var for every machine:

```sh
printf 'postgres://db.internal/app' | hopbox env set DATABASE_URL --secret
hopbox env set REGION=eu-west --scope all
```

A name of the org has one scope. So a shared value cannot take the name of an all value:

```sh
hopbox env set REGION=us-east; echo "status $?"
```

```text
hopbox: already_exists: already exists: the org has a value REGION with the scope all. Set it with that scope, or delete it first
status 5
```

A member of the team sets only private values. A shared value or an all value from a member fails with `permission_denied`.

An org key reads the names of the org values, as a member does, and sets nothing:

```sh
ORG_KEY=$(hopbox api-key create reader --kind org --expires 30d)
HOPBOX_API_KEY=$ORG_KEY HOPBOX_URL=https://example.com hopbox env set X=1; echo "status $?"
```

```text
The org key 01a14965-7890-7d12-9c3a-1f2e3d4c5b6a (reader) works until 2026-11-16T10:25:49Z. This is its only copy.
hopbox: permission_denied: permission denied: a caller of kind org_key may not do var.set
status 3
```

Revoke the key after the test with `hopbox api-key revoke <id>`.

## Share a machine with the team

A machine is private to its creator until the creator shares it. A share changes which values reach the machine. Your private values leave it, and the shared values of the org reach it. Create a machine in `org1`, and share it:

```sh
hopbox new box2
hopbox machine share box2
```

Before the share, `box2` gets `REGION=eu-west`. After the share, it gets `DATABASE_URL` and `REGION=eu-west`. hopbox pushes the new env to the running machine, so a new login shell reads it:

```sh
hopbox exec box2 bash -lc 'echo "$DATABASE_URL"'
```

```text
postgres://db.internal/app
```

Before the share, hopbox resolves the env that the machine gets as a shared machine. If that env does not resolve, the share fails, and the machine stays private. Two causes are a secret of the org that does not open, and a keyring without a key-encryption key. For example, on an install that lost its key-encryption key, the share fails with:

```text
hopbox: failed_precondition: failed precondition: the env of the machine as a shared machine does not resolve, so the machine stays private: the keyring has no key-encryption key to seal or open the value. Run hopboxd admin kek rotate to make the first one
```

The machine keeps its old env, with your private values, and only you can use it. Ask the operator of the install to put the key file back from a backup, then share again. If the key is lost, the operator makes a new one. Then set each secret of the org again, and share again. [The machine API](/reference/machine-api/index.html.md#sharemachine) lists each cause.

## Names that hopbox refuses

No value of any scope takes a name that the shell or the loader runs code from, or a name that hopbox sets itself:

```sh
hopbox env set LD_PRELOAD=/tmp/x.so --scope private
```

```text
hopbox: reserved env var name: LD_PRELOAD. No name can start with LD_
```

A value of the scope `all` reaches the private machines of every member. So a shared or an all value also cannot take a name that makes a language runtime, a tool, or glibc load or run code:

```sh
hopbox env set NODE_OPTIONS='--require /tmp/x.js' --scope all
```

```text
hopbox: reserved env var name: NODE_OPTIONS. A program can load or run code from it, so only a private value can take it
```

This list reduces the risk that an org admin runs code in your private machines. It does not remove it, because any env var can change what a program does. [The env var API](/reference/vars-api/index.html.md#names-and-values) lists every reserved name.

## Trust the admins of your org

The members of an org trust its org admins and org owners. The org admins and org owners set the all values, and an all value reaches your private machines too. So the scope `private` does not protect a machine from the all values of its org. An all value can change what a program in the machine does, even with a name that no list reserves. An org admin cannot open a shell in your private machine, but can change what runs there with an all value.

To see the all values that reach your private machines, list the values of the org:

```sh
hopbox env list
```

```text
NAME          SCOPE   SECRET  VALUE     UPDATED
DATABASE_URL  shared  yes     (secret)  2026-10-17T10:25:49Z
REGION        all     no      eu-west   2026-10-17T10:25:49Z
```

Each value with the scope `all` reaches your private machines in this org, unless you have a private value with the same name. Here, `REGION` is such a value. To replace an all value in your private machines, set a private value with the same name.

You are the only member of your personal org. So in your personal org, only you set the all values.

## Find the changes in the audit log

Each set and each delete writes an audit entry, without the value. The org admins read the audit log, so the entry of a private value also leaves out its name. An owner or an admin of the org reads the entries with a member key of the org:

```sh
KEY=$(hopbox api-key create audit --expires 30d)
curl -s -X POST -H 'Content-Type: application/json' -H "Authorization: Bearer $KEY" \
  -d '{"pageSize":50}' https://example.com/hopbox.v1.AuditService/ListAuditEntries |
  jq -c '.entries[] | select(.action == "var.set") | .details'
```

```text
{"kind":"env_var","name":"REGION","replaced":"false","scope":"all"}
{"kind":"secret","name":"DATABASE_URL","replaced":"false","scope":"shared"}
```

The newest entries come first.

## Delete a value

Name the value and its scope:

```sh
hopbox env unset REGION --scope all
```

A second unset fails with status 4, because the value is gone.

When a member leaves the org, hopbox deletes the private values of the member in that org.

## Move the secrets to a new key

This section is for the operator of the install. A rotation of the key-encryption key leaves each secret on its old key. The rewrap moves them, with the other sealed values of the install, such as the SMTP password. Run both commands as root on a control server:

```sh
sudo hopboxd admin kek rotate
sudo hopboxd admin kek rewrap
```

```text
file:2
rewrapped 2 to file:2
file:1	0	
file:2	2	current
```

Your numbers depend on the earlier rotations and on the other sealed values of the install. Each line names a key and the number of values that it seals. A key that seals no value can go, after a backup ([Sealing](/reference/sealing/index.html.md#rewrap)).

If the install has more than one control server, `kek rotate` refuses, because it makes the new key current only on its own server. Then stage the key, copy the staged file to each control server, and promote it before the rewrap, as [Run the control role on a laptop](/guide/run-hopboxd/index.html.md#add-a-second-control-server) shows.
