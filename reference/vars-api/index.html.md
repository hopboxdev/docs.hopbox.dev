# The env var API

> The env var and secret calls of the public API, VarService: scopes, who can set each one, the name rules and the reserved names, the limits, the env of a machine, sealing, and their audit entries.

The service `hopbox.v1.VarService` of the public API manages the env vars and secrets of an org. hopbox puts both into the machines of the org at boot and into every login shell. A user can read an env var back. A secret is sealed, and no call returns its value. The scope of each value says which machines it reaches.

[The API server](/reference/api/index.html.md) covers the parts that every call shares: the request gate, the policy check, and the error codes. The generated reference is [Public API](/api/hopbox.v1.openapi.yaml). The code is in `internal/service/vars.go`, `internal/api/handlers_vars.go`, and `internal/domain/envvar.go`. [Set env vars and secrets](/guide/env-vars/index.html.md) runs the calls on a laptop.

| Call | What it does |
| --- | --- |
| [`SetVar`](#setvar) | Sets an env var or a secret, and replaces the value with the same name and scope. |
| [`ListVars`](#listvars) | Lists the values that the caller sees, without the values of the secrets. |
| [`DeleteVar`](#deletevar) | Deletes an env var or a secret. |

The examples on this page use the shell function `browser` of [First sign-in on a new install](/guide/first-sign-in/index.html.md#sign-in-with-the-link), with the session cookie of the caller, and `API=http://127.0.0.1:7080`.

## The org of a request

Each request has the field `org_id`, an optional string. Leave it out to act in the org of the API key, or else in the personal org of the caller ([the org API](/reference/org-api/index.html.md#the-org-of-a-request)). The service layer reads the role of the caller in the org before the policy check. So an org that does not exist gets the same `permission_denied` as an org of others.

## Scopes

The enum `VarScope` holds the scope of a value ([Domain rules](/reference/domain-rules/index.html.md#scopes-of-env-vars-and-secrets)).

| Value | Scope | Reaches | Who sets it |
| --- | --- | --- | --- |
| `VAR_SCOPE_UNSPECIFIED` | `shared` in a request. Never in a response. | | |
| `VAR_SCOPE_SHARED` | `shared` | The shared machines of the org. | org admins and org owners |
| `VAR_SCOPE_PRIVATE` | `private` | The private machines of the account that set the value. | any member, for its own machines |
| `VAR_SCOPE_ALL` | `all` | Every machine of the org, shared or private. | org admins and org owners |

A value that is not in the enum gets `invalid_argument`, after the policy check.

These rules hold for the names in one org:

- A shared value or an all value belongs to the org. Its name has one scope. So a shared value and an all value never share a name. A set with the other scope fails with `already_exists`. Delete the value first to change its scope.
- A private value belongs to one account. Two members can each have a private value with the same name. A member can also have a private value with the name of an org value. In the private machines of the member, the private value wins ([Env of a machine](#env-of-a-machine)).
- A personal org has no shared machines. So a shared value in a personal org fails with `invalid_argument`. Use the scope `all` or `private` there.

## Who can call

| Caller | `ListVars` | `SetVar` and `DeleteVar` of a shared or an all value | `SetVar` and `DeleteVar` of a private value |
| --- | --- | --- | --- |
| `session`, `cli_token` of a member | yes | no | its own values |
| `session`, `cli_token` of an admin or an owner | yes | yes | its own values |
| `member_key` | in the org of the key, as its account | in the org of the key, as its account | in the org of the key, its own values |
| `org_key` | the shared and all values of its org | no | no |
| `admin_key`, `local_root`, `server`, `anonymous` | no | no | no |

Nobody sees, sets, or deletes the private values of another account, also not an org owner. The audit log, which org admins read, names a private value only by its ID ([Audit entries](#audit-entries)). An org key has the rights of a member without personal resources, so it has no private values ([CLI tokens, API keys, admin API keys, and SSH keys](/reference/credentials/index.html.md#api-keys)). An instance admin cannot read secrets, so a session of an instance admin has only the rights of its org roles.

## Names and values

A name has 1 to 128 characters. Each character is a letter, a digit, or `_`, and the first character is not a digit. A POSIX shell takes such names. The rules match case, as the shell does. So `path` is a valid name.

No value of any scope takes these names. The guest agent or the login shell sets them, or they can make the shell or the dynamic loader run code. The guest agent checks these rules too. It leaves a value with such a name out of the machine, and logs it. So a wrong push of the control role cannot set it:

| Reserved | Why |
| --- | --- |
| `HOME`, `USER`, `LOGNAME`, `SHELL`, `PATH`, `PWD`, `OLDPWD`, `SHLVL`, `TERM` | The guest agent or the login shell sets them ([Guest link](/reference/guest-link/index.html.md#exec)). |
| `ENV`, `PROMPT_COMMAND`, `PS0`, `PS1`, `PS2`, `PS4`, `SHELLOPTS`, `IFS` | A shell runs code from them, or they change how it reads commands. |
| `GCONV_PATH` | glibc loads code from it. |
| every name that starts with `LD_` | The dynamic loader loads code from them, for example `LD_PRELOAD` and `LD_AUDIT`. |
| every name that starts with `BASH_` | bash runs or loads code from them, for example `BASH_ENV`, `BASH_FUNC_*`, and `BASH_LOADABLES_PATH`. |
| every name that starts with `HOPBOX_` | hopbox keeps these names for the values that it sets itself. |

A shared or an all value comes from an org admin. An all value reaches the private machines of every member. So a shared or an all value also cannot take the names below. A program loads or runs code from them, or finds its code through them. A member can still set them as private values, for its own machines:

| Reserved for shared and all values | Program |
| --- | --- |
| `BASHOPTS`, `ZDOTDIR`, `FPATH`, `PS3` | bash, zsh |
| `NODE_OPTIONS`, `NODE_PATH` | Node.js |
| `PYTHONSTARTUP`, `PYTHONPATH`, `PYTHONHOME`, `PYTHONWARNINGS`, `PYTHONBREAKPOINT` | Python |
| `PERL5OPT`, `PERL5LIB`, `PERLLIB`, `PERL5DB` | Perl |
| `RUBYOPT`, `RUBYLIB` | Ruby |
| `JAVA_TOOL_OPTIONS`, `_JAVA_OPTIONS`, `JDK_JAVA_OPTIONS`, `CLASSPATH` | Java |
| `LUA_PATH`, `LUA_CPATH`, and every name that starts with `LUA_INIT` | Lua |
| `DOTNET_STARTUP_HOOKS`, and every name that starts with `CORECLR_` | .NET |
| `PHPRC`, `PHP_INI_SCAN_DIR` | PHP |
| `GOFLAGS`, `RUSTC_WRAPPER`, `RUSTC_WORKSPACE_WRAPPER` | Go, Rust |
| every name that starts with `npm_config_`, in any case, for example `NPM_CONFIG_PREFIX` | npm. It reads these names in any case. |
| every name that starts with `GIT_` | git, for example `GIT_SSH_COMMAND` |
| every name that starts with `SSH_` | ssh and sshd, for example `SSH_AUTH_SOCK` |
| `PAGER`, `MANPAGER`, `EDITOR`, `VISUAL`, `BROWSER`, `LESS`, `LESSOPEN`, `LESSCLOSE` | Tools that start the program in the value. |
| `GLIBC_TUNABLES`, `GETCONF_DIR`, `HOSTALIASES`, `LOCALDOMAIN`, `LOCPATH`, `MALLOC_TRACE`, `NIS_PATH`, `NLSPATH`, `RESOLV_HOST_CONF`, `RES_OPTIONS`, `TMPDIR`, `TZDIR` | glibc. This is the set that glibc drops for setuid programs. |

`LANG` and the `LC_*` names stay open for every scope.

These lists reduce the risk that an org admin runs code in the private machine of a member. They do not remove it. Any env var can change what a program does, and a program can read a name that no list holds. The members of an org trust its org admins and org owners. So the scope `private` does not protect a machine from the all values of its org.

A name is never sealed. It is in the table `env_vars`, in its backups, in `ListVars`, in logged errors, and in `/etc/hopbox/env`. So no value of any scope takes a name that looks like a credential. The check uses the patterns of the audit log ([Control store tables](/reference/control-store/index.html.md#audit_log)), in `domain.LooksLikeCredential`. Examples are a name that starts with `hbx_` and a GitHub token, such as `ghp_` with 36 more characters. A name in the form of an AWS access key ID is another example. Put the credential in the value of a secret.

A value is valid UTF-8 without a NUL byte, at most 16 KiB (16384 bytes). It can be empty, and it can hold newlines and quotes.

## Limits

| Limit | Value |
| --- | --- |
| Shared and all values in one org | 100 |
| Names and values of the shared and all values of one org, together | 256 KiB (262144 bytes) |
| Private values of one account in one org | 50 |
| Names and values of the private values of one account in one org, together | 128 KiB (131072 bytes) |

A set that replaces a value counts the new value in place of the old one. So at the limit, a set that replaces a value works. A set over a limit fails with `resource_exhausted`. With these limits, the env of a machine stays below 384 KiB. Linux allows 2 MiB for the arguments and the env of one exec. The Go constants are `domain.MaxOrgEnvVars`, `MaxOrgEnvVarBytes`, `MaxPrivateEnvVars`, `MaxPrivateEnvVarBytes`, and `MaxEnvVarValueBytes`.

## Env of a machine

The env of a machine is the list of names and values that reach it, sorted by name:

| Machine | Gets |
| --- | --- |
| A shared machine | the shared values and the all values of its org |
| A private machine | the all values of its org, and the private values of its creator in that org. A private value wins over an all value with the same name. |

A shared value never reaches a private machine, and a private value never reaches a shared machine. The private values of a member never reach the private machines of another member.

For example, an org has these values:

| Name | Scope | Owner | Value |
| --- | --- | --- | --- |
| `REGION` | `all` | | `eu-west` |
| `DATABASE_URL` | `shared` | | a secret |
| `LOG_LEVEL` | `all` | | `info` |
| `LOG_LEVEL` | `private` | Ann | `debug` |
| `GITHUB_TOKEN` | `private` | Ann | a secret |

Then a shared machine gets `DATABASE_URL`, `LOG_LEVEL=info`, and `REGION=eu-west`. A private machine of Ann gets `GITHUB_TOKEN`, `LOG_LEVEL=debug`, and `REGION=eu-west`. A private machine of Bob gets `LOG_LEVEL=info` and `REGION=eu-west`.

The control role builds this list with the Go method `Service.ResolveEnv(ctx, MachineEnv)`. It opens each secret at that time. It also checks the name of each stored value again with the rules of its scope. A value that someone saved before a name joined the reserved lists thus never reaches a machine. `ResolveEnv` leaves such a value out. It logs `left a stored env var out of a machine, because its name no longer passes the rules` at level warn. The fields are `component=service`, `org`, `var`, `scope`, and `reason` (`invalid` or `reserved`). `var` is the ID of the value, not its name. No entry point of the API serves it. So it takes no caller, and the policy of the call that changed the machine has already run. `MachineEnv` has these fields:

| Field | Type | Description |
| --- | --- | --- |
| `Org` | UUID | The org of the machine. It must not be the nil UUID. |
| `Shared` | bool | `true` for a machine that is shared with the org. |
| `Creator` | UUID | The account that made the machine. A private machine needs it. A shared machine ignores it. A machine that an org key makes is shared from the start. |

The node sends the list to the guest agent as the field `env` of the guest-link configuration ([Guest link](/reference/guest-link/index.html.md#configuration)). The guest agent writes it to `/etc/hopbox/env`, which every login shell reads. Exec and the start command get the same values ([hopbox-guest](/reference/hopbox-guest/index.html.md)). The control role sends the env in the spec of each machine to its executor ([the executor protocol](/reference/executor-protocol/index.html.md#machinespec)). A machine whose env does not resolve, for example without a key-encryption key, gets no spec while its target is `running`. A stop or a delete of it still goes on ([the executor protocol](/reference/executor-protocol/index.html.md#a-machine-whose-env-does-not-resolve)).

A share or an unshare changes which values reach a machine. A private machine holds the private values of its creator, and the guest agent wrote them to the disk of the machine. So after `ShareMachine` and `UnshareMachine`, the control role resolves the env again. The executor then pushes it to the guest agent of a running machine ([the node executor](/reference/executor/index.html.md#the-guest-link)). The guest agent then writes the new list in place of the old one. `SetVar` and `DeleteVar` do the same for each machine that the value reaches: they publish the event `machine.updated` for it, with the action `var.set` or `var.delete`. They lock those machine rows in the order of their IDs, add 1 to their version, and publish the events in their transaction. The new env does not remove a value from a copy that a program in the machine made already.

A private machine of a member of the org can thus get an all value that an org admin set. The [reserved names](#names-and-values) keep the known ways to run code out of the org values, such as `LD_PRELOAD`, `BASH_ENV`, and `NODE_OPTIONS`. They reduce the risk, and they do not remove it. Treat the scope `all` as a way to change what runs in the private machines of members.

A share fails closed: `ShareMachine` first resolves the env of the machine as a shared machine in its transaction. If that env does not resolve, the share fails with `failed_precondition`, and the machine stays private ([the machine API](/reference/machine-api/index.html.md#sharemachine)).

## Messages

### Var

| Field | Type | Description |
| --- | --- | --- |
| `id` | string | The UUID of the value. A set that replaces the value keeps it. |
| `org_id` | string | The UUID of the org of the value. |
| `name` | string | The name of the env var in the machine, for example `DATABASE_URL`. |
| `scope` | `VarScope` | The scope of the value. |
| `secret` | bool | `true` for a sealed value. |
| `value` | string | The value of an env var. Empty for a secret. |
| `owner_account_id` | string | The UUID of the account of a private value. Empty for a shared or an all value. |
| `updater_account_id` | string | The UUID of the account that set the value last. Empty after the delete of that account. |
| `create_time` | timestamp | The time of the first set of the value. |
| `update_time` | timestamp | The time of the last set of the value. |

## Calls

### SetVar

Sets a value, and replaces the value with the same name and scope. A replace can turn an env var into a secret, or a secret into an env var. The value of a secret is sealed in the transaction of the call. A retry gives the same end state, so it needs no idempotency key. A retry still adds one to the version of the row, and writes an audit entry.

| Item | Value |
| --- | --- |
| Procedure | `/hopbox.v1.VarService/SetVar` |
| Policy action | `var.set`: a shared or an all value needs the role `admin` or `owner`. A private value needs the role `member` or more, and the account of the caller as its owner. |
| Idempotent option | no |
| Audit entry | action `var.set`, target kind `var`. The details are `scope`, `kind` (`env_var` or `secret`), and `replaced` (`true` or `false`). A shared or an all value also has the detail `name`. |

| Field of `SetVarRequest` | Type | Default | Description |
| --- | --- | --- | --- |
| `org_id` | optional string | the org of the API key, or the personal org | The UUID of the org. |
| `name` | string | required | The name, as in [Names and values](#names-and-values). |
| `value` | string | empty | The value. The field is marked `debug_redact`, so a printed request does not show it. |
| `secret` | bool | `false` | Seals the value, so that no call returns it. |
| `scope` | `VarScope` | `shared` | The scope. |

The response `SetVarResponse` holds the value in `var`, without the value of a secret.

```sh
browser -d '{"name":"APP_ENV","value":"staging","scope":"VAR_SCOPE_ALL"}' $API/hopbox.v1.VarService/SetVar
```

```json
{"var":{"id":"01a14940-ce48-7a24-bc79-e34809e47653","orgId":"01a14940-b457-7ae9-a014-b1dd06363447","name":"APP_ENV","scope":"VAR_SCOPE_ALL","value":"staging","updaterAccountId":"01a14940-b457-7ae5-8232-30378b548d68","createTime":"2026-10-17T09:45:46.309819Z","updateTime":"2026-10-17T09:45:46.309819Z"}}
```

| Error | Code |
| --- | --- |
| The caller cannot set this scope: `permission denied: a caller of kind session may not do var.set` | `permission_denied` |
| An unknown scope: `invalid argument: the scope "<n>" is not shared, private, or all` | `invalid_argument` |
| A bad name: `invalid argument: invalid env var: the name must hold only letters, digits, and underscores, and must not start with a digit`. The message never repeats a refused name. A name can be a pasted token. An empty name and a name of more than 128 characters get their own message. | `invalid_argument` |
| A reserved name: `invalid argument: reserved env var name: LD_PRELOAD. No name can start with LD_`, or `... reserved env var name: PATH. The guest agent or the shell sets it, or it can make the shell or the loader run code` | `invalid_argument` |
| A name that only a private value can take, in a shared or an all value: `invalid argument: reserved env var name: NODE_OPTIONS. A program can load or run code from it, so only a private value can take it`, or `... reserved env var name: GIT_SSH_COMMAND. No shared or all value can start with GIT_, so only a private value can take it` | `invalid_argument` |
| A name that looks like a credential: `invalid argument: invalid env var: the name looks like a credential, and a name is never sealed. Put the credential in the value of a secret` | `invalid_argument` |
| A bad value: `invalid argument: invalid env var: the value has <n> bytes, more than 16384`, `... the value is not valid UTF-8`, or `... the value holds a NUL byte` | `invalid_argument` |
| A shared value in a personal org: `invalid argument: a personal org has no shared machines, so a shared value reaches no machine. Use the scope all or private` | `invalid_argument` |
| The org has the name with the other org scope: `already exists: the org has a value REGION with the scope all. Set it with that scope, or delete it first` | `already_exists` |
| A count limit: `usage limit exceeded: the shared and all values of an org are at most 100. Delete one first`, or `... the private values of an account in an org are at most 50. Delete one first` | `resource_exhausted` |
| A byte limit: `usage limit exceeded: the names and values of the shared and all values of an org hold at most 262144 bytes together, and this value would make <n>` | `resource_exhausted` |
| A suspended org: `failed precondition: org suspended: an instance admin suspended the org, so its env vars and secrets do not change until the suspension ends` | `failed_precondition` |
| A secret on an install without a key-encryption key: `failed precondition: the keyring has no key-encryption key to seal or open the value. Run hopboxd admin kek rotate to make the first one` | `failed_precondition` |

### ListVars

Lists the shared and all values of the org and the private values of the caller, sorted by name. A private value comes after the org value with the same name. The call returns the value of each env var, and no value of a secret.

| Item | Value |
| --- | --- |
| Procedure | `/hopbox.v1.VarService/ListVars` |
| Policy action | `var.list`: a `session`, `cli_token`, or `member_key` with a role in the org, or an `org_key` of the org |
| Idempotent option | no |
| Audit entry | none |

The request `ListVarsRequest` has the field `org_id`. The response `ListVarsResponse` holds the values in `vars`.

```sh
browser -d '{}' $API/hopbox.v1.VarService/ListVars
```

```json
{"vars":[{"id":"01a14940-ce48-7a24-bc79-e34809e47653","orgId":"01a14940-b457-7ae9-a014-b1dd06363447","name":"APP_ENV","scope":"VAR_SCOPE_ALL","value":"staging","updaterAccountId":"01a14940-b457-7ae5-8232-30378b548d68","createTime":"2026-10-17T09:45:46.309819Z","updateTime":"2026-10-17T09:45:46.309819Z"},{"id":"01a14940-ce58-776c-90ff-b6dd427564c2","orgId":"01a14940-b457-7ae9-a014-b1dd06363447","name":"GITHUB_TOKEN","scope":"VAR_SCOPE_PRIVATE","secret":true,"ownerAccountId":"01a14940-b457-7ae5-8232-30378b548d68","updaterAccountId":"01a14940-b457-7ae5-8232-30378b548d68","createTime":"2026-10-17T09:45:46.328096Z","updateTime":"2026-10-17T09:45:46.328096Z"}]}
```

The JSON leaves out the fields with their default value, such as `value` of a secret and `secret` of an env var.

### DeleteVar

Deletes one value. The rules of `SetVar` say who can delete it.

| Item | Value |
| --- | --- |
| Procedure | `/hopbox.v1.VarService/DeleteVar` |
| Policy action | `var.delete`: as `var.set` |
| Idempotent option | no |
| Audit entry | action `var.delete`, target kind `var`, details `scope`, `kind`, and `name` for a shared or an all value |

| Field of `DeleteVarRequest` | Type | Default | Description |
| --- | --- | --- | --- |
| `org_id` | optional string | the org of the API key, or the personal org | The UUID of the org. |
| `name` | string | required | The name of the value. |
| `scope` | `VarScope` | `shared` | The scope of the value. A private value is always a value of the caller. |

The response `DeleteVarResponse` has no fields.

| Error | Code |
| --- | --- |
| A name that no rule allows, with the message of `SetVar`. Examples are an empty name and a name that looks like a credential. A reserved name is allowed. A value can be older than the reservation of its name, and it can still go. | `invalid_argument` |
| No value with this name and scope: `not found: the org has no value REGION with the scope all` | `not_found` |
| A suspended org, with the message of `SetVar` | `failed_precondition` |

## What a member leaves behind

A leave, a removal, and the delete of an account take the account out of an org. They also delete the private values of the account in that org. Each value gets a `var.delete` entry with the detail `reason` `owner left` and no name ([the org API](/reference/org-api/index.html.md#what-a-member-leaves-behind)). The delete of an org deletes every value of the org.

## Suspension

While an instance admin suspends an org, its values do not change ([the org API](/reference/org-api/index.html.md#suspension)). `SetVar` and `DeleteVar` fail with `failed_precondition`. `ListVars` still works.

## Sealing

A secret is sealed with the place `OrgSecret` of its org, scope, owner, and name ([Sealing](/reference/sealing/index.html.md#places)). So a sealed value that someone copies into another row does not open there. The column `env_vars.sealed_value` is in the list of sealed columns, so `hopboxd admin kek rewrap` moves the secrets to the current key-encryption key ([Sealing](/reference/sealing/index.html.md#rewrap)). The control store keeps the values in the table `env_vars` ([Control store tables](/reference/control-store/index.html.md#env_vars)).

## Audit entries

The calls write these entries in the transaction of the change. Their org is the org of the value, so the owners and admins of the org read them with `ListAuditEntries` ([the sign-in, account, and audit API](/reference/auth-api/index.html.md#listauditentries)). No entry holds a value. The entry of a private value also holds no name: the private values of a member are not for the org admins to see. Its target ID names the value. No new name looks like a credential. A name that someone stored before this rule never blocks a change of its value. The audit entry of that change then goes without the name.

| Action | Call | Target | Details |
| --- | --- | --- | --- |
| `var.set` | `SetVar` | the value | `scope`, `kind`, `replaced`, and `name` for a shared or an all value |
| `var.delete` | `DeleteVar`, and the leave, removal, or account delete of the owner of a private value | the value | `scope`, `kind`, `name` for a shared or an all value, and `reason` `owner left` for a value of a member that left |
