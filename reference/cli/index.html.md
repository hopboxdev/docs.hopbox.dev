# hopbox CLI

> The commands, flags, env vars, profiles, files, tokens, output, and exit status of the hopbox CLI. It also covers the managed SSH block and the prompt.

`hopbox` is the public CLI of hopbox. Users install it on laptops. It runs on macOS on Apple Silicon, and on Linux on x86_64 and arm64. On Windows, run the Linux build in WSL. The CLI calls the public API of an install through the Go client of [Go client](/reference/go-client/index.html.md). It never reads the control store.

This page lists every command, flag, env var, file, output, and exit status of the CLI. [Install the hopbox CLI](/guide/install-cli/index.html.md) installs the CLI, and [Use the hopbox CLI](/guide/cli/index.html.md) shows it at work on an install. The code is in `cmd/hopbox`, `internal/cli`, and `internal/cli/config`.

## Usage

```sh
hopbox [global flags] <command> [args] [flags]
```

A global flag can come before or after the command. When you run `hopbox` without a command on a terminal, it opens [the prompt](#the-prompt), which runs one command after another. Without a terminal, it prints its usage and an error to stderr, and exits with status 80.

## Global flags

| Flag | Env var | Type | Default | Description |
| --- | --- | --- | --- | --- |
| `-h`, `--help` | | bool | `false` | Prints the help of `hopbox`, or of the command that it follows, and exits with status 0. |
| `--profile=NAME` | `HOPBOX_PROFILE` | string | the active profile, else `default` | The profile of the command: one install and its sign-in. See [Profiles](#profiles). |
| `--org=ORG` | `HOPBOX_ORG` | string | the active org of the profile | The org of the command, by name or UUID. See [The org of a command](#the-org-of-a-command). |
| `--json` | | bool | `false` | Prints the result as JSON on stdout, and an error as JSON on stderr. See [Output](#output). |
| `--config-dir=DIR` | `HOPBOX_CONFIG_DIR` | path | see [Files](#files) | The dir of the config and the CLI tokens. |
| `--debug` | `HOPBOX_DEBUG` | bool | `false` | Logs each retry of a call and each refresh of the CLI tokens to stderr. |
| `--no-ssh-config` | `HOPBOX_NO_SSH_CONFIG` | bool | `false` | Leaves `~/.ssh` alone: no managed block in `~/.ssh/config`, no line in `~/.ssh/known_hosts`, and no new key at `login`. See [SSH files](#ssh-files). |

A flag on the command line wins over its env var.

## Env vars

| Env var | Description |
| --- | --- |
| `HOPBOX_PROFILE` | The value of `--profile`. |
| `HOPBOX_ORG` | The value of `--org`. |
| `HOPBOX_CONFIG_DIR` | The value of `--config-dir`. |
| `HOPBOX_DEBUG` | The value of `--debug`, for example `1`. |
| `HOPBOX_NO_SSH_CONFIG` | The value of `--no-ssh-config`, for example `1`. |
| `HOPBOX_URL` | The value of `login --url`. With `HOPBOX_API_KEY`, it is also the URL of the install for every command, and it is required. |
| `HOPBOX_API_KEY` | An API key ([CLI tokens, API keys, admin API keys, and SSH keys](/reference/credentials/index.html.md#api-keys)). If it is set, each command calls the API with the key, and not with the CLI sign-in of the profile. See [API keys in scripts](#api-keys-in-scripts). |
| `XDG_CONFIG_HOME` | If it is set and `HOPBOX_CONFIG_DIR` is not, the config dir is `$XDG_CONFIG_HOME/hopbox`. |
| `HOME` | The home dir, for the default config dir and for `~/.ssh`. |
| `TERM` | The terminal type that `connect` and `exec --tty` send to the machine. Empty gives `xterm-256color` in the machine. With `dumb`, `hopbox` without a command opens no prompt. |

## Profiles

A profile is one install that the CLI signs in to. It has a name, the URL of the install, and the active org. One CLI can hold many profiles, for example `default` for a hosted install and `dev` for an install on the laptop.

- The profile of a command is `--profile`, else the active profile of the config, else `default`.
- `hopbox login` makes or updates the profile of the command, and makes it the active profile.
- `hopbox profile use` changes the active profile. `hopbox profile remove` deletes a profile.

A profile name has 1 to 64 characters from `A-Z`, `a-z`, `0-9`, `.`, `-`, and `_`. It starts with a letter or a digit. Another name fails with status 80.

## The org of a command

Each request that acts in an org carries an org ([the org API](/reference/org-api/index.html.md#the-org-of-a-request)). The CLI picks it in this order:

1. `--org`, by name or UUID. A name matches one of your orgs. An exact match wins over a match that ignores case. If no org of yours has the name, the command fails with status 4. If two orgs have the name, the command fails with status 80 and lists their UUIDs.
2. The active org of the profile, which `hopbox org use` sets.
3. No org: the server takes the personal org of the account, or the org of an API key.

Sometimes `hopbox org delete` or `hopbox org leave` removes the active org. Then the command also drops the active org, and the profile acts in the personal org again.

## Files

The config dir is `$HOPBOX_CONFIG_DIR`, else `$XDG_CONFIG_HOME/hopbox`, else `~/.config/hopbox`. macOS uses the same path. The CLI makes the dir with mode 0700 at its first write, such as at `hopbox login`. A command that only reads makes no dir.

`hopbox version` reads no config dir. With `HOPBOX_API_KEY`, only `login`, `logout`, `profile`, and `org use` read it. So a script with a key works with any config dir, or with none. `org create --use` and `org join --use` fail with status 80 while `HOPBOX_API_KEY` is set, as `org use` does.

| File | Mode | Content |
| --- | --- | --- |
| `config.toml` | 0600 | The profiles. It holds no secret. |
| `credentials.toml` | 0600 | The CLI tokens of each profile. |
| `lock` | 0600 | An empty file. Each process of the CLI takes an exclusive `flock` on it to change a file of the dir, or to refresh the CLI tokens. |
| `history` | 0600 | The lines of [the prompt](#the-prompt), one per line, oldest first. |

The CLI writes a file to a temporary file in the same dir and renames it into place. So another process never reads half a file. The CLI does not use the keychain of the OS.

The CLI tokens are secrets, so the CLI checks the dir and `credentials.toml` before it uses them, as ssh checks `~/.ssh`. The dir can be a symlink, for example from a dotfile manager. The CLI then checks the dir that the link points to. It resolves the link once, at the start of the command or at its first write, and works only in that dir. So a link that someone points elsewhere during the command moves none of its files. `credentials.toml` itself cannot be a symlink. Each check that fails stops the command with status 1:

| Check | Error |
| --- | --- |
| `credentials.toml` is a symlink. | `<path> is a symlink. The CLI tokens must live in a real dir and file. Set HOPBOX_CONFIG_DIR to a dir of your own` |
| Another user owns the dir, the target of a linked dir, or `credentials.toml`. | `<path> belongs to the user with uid <uid>, not to you. Set HOPBOX_CONFIG_DIR to a dir of your own` |
| Group or others can write the dir, the target of a linked dir, or `credentials.toml`. | `group or others can write <path> (mode <mode>). Run chmod go-w <path>` |
| Group or others can read `credentials.toml`. | `the file <path> holds secrets, but others can access it (mode <mode>). Run chmod 600 <path>` |
| The file system of the dir has no file locks, such as some network file systems. | `the file system has no file locks: the config dir <dir> cannot be locked, so two hopbox commands could refresh the CLI tokens at once and end the sign-in. Set HOPBOX_CONFIG_DIR to a dir on a local disk` |

The CLI opens `lock` and `credentials.toml` without following a symlink. After the open, it checks that the file that opened is the file that it checked. Else it fails with `<path> changed while the CLI read it. Run the command again`.

### config.toml

```toml
active_profile = "default"

[profiles.default]
url = "https://example.com"
org = "01a14a7c-be1e-74cc-9748-ff40254c5d55"
```

| Key | Type | Default | Description |
| --- | --- | --- | --- |
| `active_profile` | string | empty, which means `default` | The profile of a command without `--profile`. |
| `profiles.<name>.url` | string | none | The URL of the install, as [URLs of an install](#urls-of-an-install) gives it. `hopbox login` sets it. |
| `profiles.<name>.org` | string | empty | The UUID of the active org. Empty means the personal org. `hopbox org use` sets it. |

You can edit the file. The CLI writes it again at the next change, without your comments.

### credentials.toml

```toml
[profiles.default]
url = "https://example.com"
account_id = "01a14a7c-813f-7a14-b6bc-44a4e57ac6ff"
cli_sign_in_id = "01a14a7c-abae-7e4d-bde7-34c1e1d6870d"
access_token = "hbxat_<token>"
access_token_expire_time = 2026-10-17T16:30:36Z
refresh_token = "hbxrt_<token>"
refresh_token_expire_time = 2026-11-16T15:30:36Z
```

| Key | Type | Description |
| --- | --- | --- |
| `profiles.<name>.url` | string | The install that issued the tokens. The CLI sends the tokens to this URL only. If the profile has another URL, the profile is not signed in. |
| `profiles.<name>.account_id` | string | The UUID of the account of the tokens. |
| `profiles.<name>.cli_sign_in_id` | string | The UUID of the CLI sign-in ([CLI tokens, API keys, admin API keys, and SSH keys](/reference/credentials/index.html.md#cli-tokens)). |
| `profiles.<name>.access_token` | string | The access token, `hbxat_...`. |
| `profiles.<name>.access_token_expire_time` | datetime | The end of the access token, 1 hour after its issue. |
| `profiles.<name>.refresh_token` | string | The refresh token, `hbxrt_...`. |
| `profiles.<name>.refresh_token_expire_time` | datetime | The end of the refresh token, 30 days after its issue. |

### SSH files

The CLI also keeps three things in `~/.ssh`, where `~` is `$HOME`. With them, `ssh box1.hop`, scp, rsync, and editors reach a machine through the SSH front door of the install. `--no-ssh-config` leaves `~/.ssh` alone.

| File | When the CLI writes it | Content |
| --- | --- | --- |
| `~/.ssh/config` | At `new`, `machine rename`, `machine delete`, and `list` | The managed block: one `Host` entry for each machine |
| `~/.ssh/known_hosts` | At `login`, and with each write of the managed block, once the install has an SSH front door | The managed block: one line with the host key of each install |
| `~/.ssh/hopbox_ed25519` and `hopbox_ed25519.pub` | At `login`, for an account without an SSH key | A new ed25519 key pair. The CLI adds the public key to the account. It takes the public key from the private key, so a stale `.pub` file changes nothing. A `.pub` file without its private key stops the CLI, and it makes no key next to it. |

A managed block starts with the line `# BEGIN hopbox` and ends with the line `# END hopbox`. The CLI changes only the lines between these markers. Every other byte of the file stays as it was. The CLI writes the new file to a temporary file in the same dir. Then it renames that file into place, so ssh never reads half a file. A file that exists keeps its mode. A new file gets mode 0600, and a new `~/.ssh` gets mode 0700. A file that is a symlink, for example from a dotfile manager, stays a symlink: the CLI writes the file that the link points to. For each change, the CLI takes an exclusive `flock` on the dir `~/.ssh`. So two commands at once never lose a change.

A file with a `# BEGIN hopbox` and no `# END hopbox`, a second block, or an `# END hopbox` alone is broken. The CLI then changes nothing in it, and logs `the managed SSH block was not updated` at level warn, with `the hopbox block markers do not pair up`. Fix the markers by hand. A failure to write a file of `~/.ssh` never fails a command, because the machine changed already. The command logs the warning and goes on.

Every value in a managed block comes from the API of an install. A hostile or broken install must not add lines of its own to `~/.ssh`. For example, a newline in a machine name can add a `ProxyCommand`. So the CLI checks each value before it writes it, and leaves out an entry or a key that fails. It then logs `the machine gets no entry in the managed SSH block` at level warn, or skips the whole write.

| Value | Rule |
| --- | --- |
| Machine name | A lowercase DNS label: `a-z`, `0-9`, and `-`, 1 to 63 characters, from a letter, not ending with `-` |
| Domain, `HostName`, and the host in `HostKeyAlias` | DNS labels of letters, digits, and `-`, at most 253 characters, or an IP address without a zone. No comma, space, `%`, `*`, or control character. The CLI writes the domain in lower case, and drops one dot at its end, as in `example.com.`. |
| Port | 1 to 65535 |
| User | `user`, or another lowercase Linux user name |
| Machine ID and org ID | A UUID in its canonical form |
| Host key | Exactly one public key. The CLI writes it in its canonical form, without options or a comment. |

The block of `~/.ssh/config`:

```text
# BEGIN hopbox
# The hopbox CLI writes this block at each create, rename, and delete of a machine. Edits inside it are lost.
Host box1.hop
  # hopbox machine=01a15e16-fe58-74f1-a778-d1350c527612 org=01a14a7c-813f-7a1c-946b-db123b23c342
  HostName example.com
  Port 10001
  User user
  IdentityFile ~/.ssh/hopbox_ed25519
  IdentitiesOnly yes
  HostKeyAlias hopbox-example.com
Host *
# END hopbox
```

| Line | Value |
| --- | --- |
| `Host` | The alias of the machine: its name and `.hop`, such as `box1.hop` |
| `# hopbox` | The UUIDs of the machine and of its org. The CLI reads them back at the next change. |
| `HostName` | The domain of the install, from `GetInstallInfo`. An install without a domain gets the host of its URL. |
| `Port` | The SSH port of the machine on the edge servers. It never changes ([the machine API](/reference/machine-api/index.html.md#machine)). |
| `User` | `user`, the main Linux user of the default image |
| `IdentityFile` | `~/.ssh/hopbox_ed25519`, only while that file exists. |
| `IdentitiesOnly` | `yes`, with `IdentityFile` only. ssh then offers only that key, not each key of the agent. The SSH front door says yes to every offered key, and checks only signed keys. Without it, ssh signs with each key in turn, and a security key asks for a touch each time ([the SSH front door](/reference/ssh-front-door/index.html.md#sign-in)). To use another key, give it with `ssh -i`. |
| `HostKeyAlias` | `hopbox-<domain>`. ssh looks the host key up under this name, without the port. So one line of `known_hosts` covers the SSH port of every machine. |

The entries are in the order of their aliases. A new block goes at the top of the file, because ssh takes the first value that it finds for each option. So a `Host *` of yours with another `User` or `Port` cannot change the entry of a machine. The block ends with `Host *`, so the lines of yours after the block still apply to every host, as they did before. You can move the block elsewhere in the file. The CLI then changes it where it is. When no machine is left, the CLI removes the block and its markers.

Each entry belongs to one machine, of one org, on one install. `new` adds the entry of the machine, and replaces an entry with the same machine UUID or the same alias. Another machine that had the alias, for example on another install, loses it, and the CLI logs `another machine had the SSH alias, and loses it` at level warn. `machine rename` replaces the entry. `machine delete` removes it. `list` makes the entries of the org and the install match the list: it adds the machines that another laptop or the console made, and removes the machines that are gone. The entries of other orgs and other installs stay.

The block of `~/.ssh/known_hosts` goes at the end of the file:

```text
# BEGIN hopbox
# The hopbox CLI writes this block at login and at each change of a machine. Edits inside it are lost.
hopbox-example.com ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAI...
# END hopbox
```

Each line holds the `HostKeyAlias` of an install and the `ssh_host_key` of its `GetInstallInfo` ([the API server](/reference/api/index.html.md)). The key comes over the API, which TLS protects, so the first `ssh box1.hop` asks no question about the host key. A new host key of the install replaces the line at the next write. An install without an SSH front door has no host key, so the CLI writes no line for it.

The certificate of TLS proves only the host of the URL of the install. So over `https`, the domain of `GetInstallInfo` must be the host of the URL. For another domain, the CLI uses the host of the URL in `HostName` and `HostKeyAlias`, and for the host key. It then logs `the install names another domain than its URL, so the managed SSH block uses the host of the URL` at level warn. So one install never writes the entries or the host key of another install. A loopback install without TLS, such as `http://127.0.0.1:7080`, gets its line for its domain. The trust then rests on the laptop, or on the SSH tunnel that leads to the install.

The block works only while ssh reads `~/.ssh/known_hosts`. A `UserKnownHostsFile` or `GlobalKnownHostsFile` in your config that leaves out this file makes ssh skip the block. Then ssh asks about the host key at the first connection, as for any new host.

## URLs of an install

`hopbox login --url` takes the URL of the install. The API serves at the root of the domain of the install, so the URL has no path.

- A bare domain gets `https://`. `example.com` is `https://example.com`.
- The CLI drops a `/` at the end, and writes the host in lower case. It also drops an empty port and the default port: 443 for `https`, 80 for `http`.
- Plain `http://` works only for a loopback host: `localhost`, `127.0.0.0/8`, or `::1`. So a bearer token never crosses a network without TLS. A laptop install uses `http://127.0.0.1:7080`.
- A URL with a path, a query, a fragment, a user name, or a password fails with status 80.

The CLI speaks HTTP/2 to the install. To a loopback install without TLS, it speaks HTTP/2 with prior knowledge, because `exec` and `connect` need a stream in both directions ([Go client](/reference/go-client/index.html.md#make-a-client)). It speaks no HTTP/1.1 there. So a proxy on a loopback address that speaks only HTTP/1.1 fails every call. `hopboxd` speaks both.

## Sign-in and tokens

`hopbox login` signs in with the device flow ([CLI tokens, API keys, admin API keys, and SSH keys](/reference/credentials/index.html.md#the-device-flow)):

1. It calls `StartDeviceAuthorization` with the client name `hopbox CLI on <host name of the laptop>`, cut to 64 characters.
2. It prints the user code and the page `https://<domain>/device?code=<user code>` to stderr.
3. It polls `PollDeviceAuthorization` after each interval of the response, 5 s at first. A `SLOW_DOWN` answer makes the interval longer.
4. When a person approves the code in the console, the poll gets the CLI tokens. The CLI stores them in `credentials.toml`, sets the URL of the profile, and makes the profile active. The approval needs a sign-in to the console within the last 10 minutes ([Sessions, cookies, and sign-in links](/reference/sessions/index.html.md#recent-sign-in)).
5. If the profile had a CLI sign-in before, the CLI ends it with `SignOut` and its old tokens.

Each command then calls the API with the access token. The CLI refreshes the tokens with `RefreshCLIToken` in two cases:

- The access token ends within 1 minute.
- The API refuses the access token with `unauthenticated`. The command then retries its call once with the new token.

A refresh token works once ([CLI tokens, API keys, admin API keys, and SSH keys](/reference/credentials/index.html.md#cli-tokens)). So the CLI refreshes only under the lock of the config dir. It always uses the refresh token that `credentials.toml` holds at that moment, never a copy in memory. A second process that waits for the lock reads the new tokens of the first, and makes no refresh of its own. A process that holds an older access token reads `credentials.toml` again before a refresh. The grace of 10 s of the install covers only one second refresh with the same token. Without the lock, three processes that refresh at once end the sign-in, and the audit log records a reuse that no thief made. So the CLI refuses a config dir on a file system without `flock` ([Files](#files)).

If the refresh token ended, the profile is not signed in. If the install refuses the refresh token, the sign-in ended: someone ended it in the console, or a used refresh token came back. If `credentials.toml` still holds the refresh token that the CLI sent, the CLI then deletes the tokens of the profile. The command fails with status 2. Run `hopbox login` again.

If the CLI cannot write the new tokens, the command still finishes with them. The CLI logs `the refreshed CLI tokens could not be stored, so the next command must sign in again` at level warn. It also tries to delete the old tokens of the profile, so the next command asks for a new sign-in.

A refresh replaces the refresh token on the install before the CLI writes the new one. Two cases leave the old refresh token in `credentials.toml`: the write and the delete both fail, or the process dies between the refresh and the write. The next refresh then sends the old token again. The install ends the CLI sign-in, and its audit log records a refresh token reuse. Run `hopbox login` again. No token leaked, but the install cannot tell this case from a theft.

## API keys in scripts

If `HOPBOX_API_KEY` holds an API key, each command calls the API with that key and ignores the CLI sign-in of the profile.

- The URL of the install must be in `HOPBOX_URL`. Without it, the command fails with status 80. The CLI never sends the key to the URL of a profile, because the profile can name another install.
- A key works in its own org only ([CLI tokens, API keys, admin API keys, and SSH keys](/reference/credentials/index.html.md#api-keys)). The active org of the profile does not apply. `--org` still applies, and the server refuses an org that is not the org of the key.
- `hopbox org use`, and `--use` of `org create` and `org join`, fail with status 80, because the active org has no effect on a key.
- No API key can make, list, or revoke API keys. Such a command fails with status 3.
- `hopbox login` and `hopbox logout` ignore the key. They act on the CLI sign-in of the profile.
- If the install refuses the key, the error says `Check the API key in HOPBOX_API_KEY`.
- The commands for machines work with a key too. They still keep the managed blocks in `~/.ssh`, so a script on a shared runner can add `--no-ssh-config`.

## Retries

Each call goes through the retries of the Go client ([Go client](/reference/go-client/index.html.md#retries)). The CLI retries a call twice after `unavailable`, `resource_exhausted`, or `deadline_exceeded`, if the call is safe to run again. A read call, such as `ListMachines`, and a call with an idempotency key, such as `CreateMachine`, are safe. A call that changes state and has no key, such as `CreateInvitation`, gets a retry only when its try surely did not reach the install: a refused connection, or an answer with `Hopbox-Not-Run: 1` ([the API server](/reference/api/index.html.md#answers-of-calls-that-changed-nothing)). A refresh of the CLI tokens is the one exception. If the answer of a refresh gets lost, the CLI sends the refresh once more, if that retry starts within 10 s of the first try. The install takes that one replay, so the command goes on. If the second answer gets lost too, the command fails with status 6, and the next command ends the CLI sign-in. You then run `hopbox login` again ([CLI tokens, API keys, admin API keys, and SSH keys](/reference/credentials/index.html.md#cli-tokens)). It waits a random time up to 500 ms before the first retry, and up to 1 s before the second. If the server sends `Retry-After`, the CLI waits that time instead, up to 5 minutes. A wait of 10 s or more logs `retrying a call` at level warn, with the wait. A command has no deadline of its own, so a longer `Retry-After` does not stall it. The command fails at once with status 6, and the error ends with `The install asks to wait <time> before a retry`. A refresh of the CLI tokens waits at most 30 s. It holds the lock of the config dir, and every other command waits for that lock. Each create call carries one `Idempotency-Key` on all of its tries ([the API server](/reference/api/index.html.md#idempotency-keys)).

## Output

The CLI writes the result of a command to stdout, and errors and notes to stderr.

Without `--json`, the result is text for people:

- A list is a table with a header row of upper-case names, and columns that spaces align.
- One item is a list of `name value` lines.
- A time is in RFC 3339 form in UTC, such as `2026-10-17T15:30:51Z`. An unset time is `never`.
- An empty value is `-`.

With `--json`, the result is one JSON value and a newline. A list is a JSON array, and one item is a JSON object. The field names are in snake case. A time is an RFC 3339 string in UTC, and an unset time is `null`. Each command below gives its JSON fields.

An error goes to stderr. Without `--json`, it is one line:

```text
hopbox: <message>
```

An error of the API has the form `<code>: <message of the server>`, such as `permission_denied: permission denied: a caller of kind member_key may not do api_key.create`. The codes are those of [the API server](/reference/api/index.html.md#errors). With `--json`, the error is one JSON object:

```json
{"error":{"code":"permission_denied","message":"permission denied: a caller of kind member_key may not do api_key.create","request_id":"01a14a7c-c0a1-7e1e-8a52-0f3c9a1b2d3e"}}
```

| Field | Description |
| --- | --- |
| `code` | The code of the API, such as `not_found`. `signed_out` means that the profile has no live CLI sign-in. `usage` means a bad command line. `failed` is any other error of the CLI. |
| `message` | The message of the server, or of the CLI. |
| `request_id` | The ID of the call in the log of the server ([the API server](/reference/api/index.html.md#the-chain-of-a-call)). A call that did not reach the server has no such field. |

A bad command line with `--json` also gets the JSON error, with the code `usage` and no usage text.

Some commands differ from these rules. `exec` prints the output of the command, and `cp` prints nothing on success. `logs --json` prints one JSON object for each line of the log. `connect` has no JSON output.

With `--debug`, the CLI logs records like `level=DEBUG msg="retrying a call" procedure=/hopbox.v1.OrgService/ListOrgs code=unavailable retry=1 wait=312ms` to stderr. Without it, the CLI logs only warnings.

### Logs

Other sections name the records of their own topic, such as [SSH files](#ssh-files) and [Retries](#retries). The CLI also writes these records:

| Message | Level | Fields | When |
| --- | --- | --- | --- |
| `no home dir, so the CLI leaves ~/.ssh alone` | warn | `err` | `HOME` is empty, and the CLI cannot find the home dir in another way. |
| `refreshed the CLI tokens` | debug | `cli_sign_in` | The CLI traded its refresh token for new tokens ([Sign-in and tokens](#sign-in-and-tokens)). `--debug` shows it. |
| `the install sent a host key that is not one public key, so known_hosts gets none` | warn | `err` | `GetInstallInfo` returned an SSH host key that does not parse. |
| `the host key of the install is not in ~/.ssh/known_hosts` | warn | `err` | At `login`, the CLI did not write the host key of the install. |
| `the CLI cannot read the SSH keys of the account, so it adds none` | warn | `err` | At `login`, `ListSSHKeys` failed. |
| `the CLI made no SSH key. Add one with hopbox ssh-key add` | warn | `err` | At `login`, the account has no SSH key, and the CLI did not make `~/.ssh/hopbox_ed25519`. |
| `the CLI did not add its SSH key to the account. Add it with hopbox ssh-key add` | warn | `file`, `err` | At `login`, `AddSSHKey` failed. `file` is the public key. |
| `the install asked to poll less often` | debug | `interval_seconds` | During `login`, the install answered the device flow with the state slow down. |
| `the replaced CLI sign-in did not end` | debug | `cli_sign_in`, `err` | A new `login` replaced the CLI sign-in of the profile, and `SignOut` of the old sign-in failed. |
| `the archive holds an entry that is not a file, a directory, or a link, so the copy skips it` | warn | `path` | `cp` from a machine got an entry of another type in its archive. |
| `the read of stdin failed, so the command gets no more input` | debug | `err` | `exec` did not read its stdin, for a reason other than its end. |
| `the line is not in the history file` | warn | `err` | The prompt did not add a line to its history file. |
| `the prompt stops to follow the machines` | debug | `err` | The watch of the machines of the prompt ended for good, for example after a sign-out. |
| `the watch of the machines broke, so the prompt opens it again` | debug | `err`, `wait` | The watch of the machines of the prompt failed. The prompt opens it again after `wait`. |

## Machines in commands

A command names a machine by its name, such as `box1`, or by its UUID. A text in the form of a UUID is always a UUID. Machine names are unique in the whole install, so a command on one machine needs no org ([the machine API](/reference/machine-api/index.html.md#naming-a-machine)). `--org` applies to `new`, `list`, and `env`.

`machine show` prints a machine in this form:

```text
id             01a15e16-fe58-74f1-a778-d1350c527612
name           box1
org            01a14a7c-813f-7a1c-946b-db123b23c342
state          running
size           default (2 vCPU, 8 GiB RAM, 100 GB disk)
image          default (sha256:0f3c...)
shared         no
url            https://box1.example.com (private, port 8000)
ssh            ssh box1.hop (port 10001)
restart        always
start command  -
auto-delete    off
last used      2026-10-21T10:52:07Z
created        2026-10-21T10:52:07Z
```

A machine on its way to another state shows both states, such as `stopped, on its way to running`. The line `reboot pending  yes` shows while a reboot waits. The line `skipped env vars  <count>` shows the env vars that the guest agent left out. While the install has no domain, the URL is `-`. `auto-delete` shows the timeout, such as `after 168h0m0s without use`.

The JSON object of a machine:

| Field | Type | Description |
| --- | --- | --- |
| `id` | string | The UUID of the machine. |
| `org_id` | string | The UUID of its org. |
| `name` | string | The name. |
| `state` | string | The current state: `pending`, `starting`, `running`, `standby`, `hibernated`, `stopped`, `failed`, `deleted`, or `moving`. |
| `target_state` | string | The state that the machine must reach. |
| `size` | object | The size: `name`, `vcpus`, `memory_bytes`, and `disk_bytes`. |
| `image_name` | string | The image, such as `default`. |
| `image_digest` | string | The digest of the image, 64 hex digits. |
| `creator_account_id` | string | The account that made the machine, or empty. |
| `creator_key_id` | string | The org key that made the machine, or empty. |
| `shared` | bool | Whether the whole org can use the machine. |
| `restart_policy` | string | `always` or `never`. |
| `start_command` | string | The start command, or empty. |
| `auto_delete_timeout_seconds` | number or null | The auto-delete timeout in seconds. `null` means off. |
| `url` | string | `https://<name>.<domain>`, or empty while the install has no domain. |
| `url_port` | number | The port in the machine that the URL reaches. |
| `url_visibility` | string | `private` or `public`. |
| `ssh_port` | number | The SSH port of the machine on the edge servers. |
| `ssh_alias` | string | The alias of the machine in the managed SSH block, such as `box1.hop` ([SSH files](#ssh-files)). |
| `reboot_pending` | bool | Whether a reboot waits. |
| `skipped_env_vars` | number | The count of env vars that the guest agent left out. |
| `last_activity_time` | string or null | The last use, for the auto-delete timeout. |
| `state_change_time` | string or null | The last change of the state. |
| `delete_request_time` | string or null | The time of the delete request. |
| `version` | number | The version of the machine. |
| `create_time` | string | The creation time. |
| `update_time` | string | The time of the last change by a request. |

A timeout of the CLI, such as `--auto-delete`, has one of three forms: `off`, a Go duration such as `12h`, or days such as `7d`. The API takes 1 minute to 365 days.

## The prompt

The prompt is the interactive mode of the CLI. It reads one command after another from the terminal, and runs each one. Tab completes commands, flags, and machine names. [Work in the prompt of the CLI](/guide/cli-prompt/index.html.md) shows it at work.

```sh
hopbox [global flags]
```

The prompt needs all of these things to open:

- stdin is a terminal.
- stdout is a terminal.
- The command line has no `--json`.
- `TERM` is not `dumb`. The line editor draws with the escape codes of ANSI terminals, which a dumb terminal does not know.

Else `hopbox` without a command prints its usage to stderr, then `hopbox: name a command. The prompt of hopbox without a command opens only with a terminal on stdin and stdout, without --json, and with a TERM other than dumb`, and exits with status 80. With `--json`, it prints the JSON error with the code `usage`, and no usage text. So a script never waits at a prompt.

### The line of the prompt

The prompt is `hopbox> `. If the profile is not `default`, the prompt names it, as in `hopbox (dev)> `. The prompt reads the active profile again before each line, so `hopbox profile use dev` changes the next prompt.

Each line runs as `hopbox` runs its arguments, with the global flags of the prompt in front. For example, in `hopbox --profile dev`, the line `list` runs `hopbox --profile=dev list`. A global flag in the line wins over the flag of the prompt. The output, the errors, and the effects of a command are the same as in a shell. In the prompt, an error of the command line prints only its line, without the usage.

The prompt splits a line into words as a POSIX shell does, with these rules:

| Text | Meaning |
| --- | --- |
| Space or tab | Ends a word. |
| `'text'` | Keeps each character of `text` as it is. |
| `"text"` | Keeps each character of `text`, except `\"` for `"` and `\\` for `\`. |
| `\c` | Outside quotes, keeps the character `c`, for example a space. |
| `~` or `~/path` at the start of an unquoted word | Your home dir, from `HOME`. |

The prompt does no other expansion. `$VAR`, `*`, `|`, `>`, and `;` are plain characters. A quote that does not close fails with `hopbox: the line has a quote that does not close`, and the line does not run.

These words have a meaning of their own:

| Line | Effect |
| --- | --- |
| Empty | Nothing. |
| `hopbox <command> ...` | Runs `<command> ...`. The prompt drops the first word `hopbox`, so a command that you copy from a shell works. |
| `help` | Prints the usage of the CLI, as `hopbox --help` does. |
| `help <command>` | Prints the help of the command, as `hopbox <command> --help` does. |
| `exit` or `quit` | Closes the prompt, with status 0. With an argument, it fails with `hopbox: exit takes no arguments`, or the same for `quit`, and the prompt stays open. |
| `prompt`, or a line of global flags alone | Fails with `hopbox: the prompt is open already. Type a command, such as list, or exit`. |

### Keys

The line editor is [ergochat/readline](https://github.com/ergochat/readline) v0.1.3. It holds the terminal in raw mode only while it reads a line. Before each prompt, it asks the terminal for the column of the cursor. Until the answer comes, the prompt gives the editor no keys, so keys that you type ahead stay in order for their readers. For example, a paste of a command, then the answer to its question, then a next command, goes to the right places. If no answer comes within 1 s, the editor takes column 1. The prompt then drops the answer that comes later, within 30 s, so it reaches neither the editor nor a command. After 30 s, it drops nothing for that query, because some keys, such as Ctrl-F3, send the same bytes as an answer. After three queries in a row without an answer, the prompt takes the terminal for one that never answers. It then asks no more, waits no more, and drops nothing. When a line ends, it puts the terminal back as it was, so a command gets the terminal in its usual mode. When the prompt closes, it puts the terminal back too, also after an error, a signal, or a panic.

| Key | Effect |
| --- | --- |
| Enter | Runs the line. |
| Tab | Completes the word at the cursor. See [Completion](#completion). |
| Left, Ctrl-B | Moves back one character. |
| Right, Ctrl-F | Moves forward one character. |
| Alt-B, Alt-Left | Moves back one word. |
| Alt-F, Alt-Right | Moves forward one word. |
| Home, Ctrl-A | Moves to the start of the line. |
| End, Ctrl-E | Moves to the end of the line. |
| Backspace, Ctrl-H | Deletes the character before the cursor. |
| Delete | Deletes the character at the cursor. |
| Ctrl-D | Deletes the character at the cursor. On an empty line, it closes the prompt with status 0. |
| Ctrl-W | Deletes the word before the cursor. |
| Ctrl-U | Deletes the text before the cursor. |
| Ctrl-K | Deletes the text from the cursor to the end of the line. |
| Ctrl-Y | Puts back the text that the last delete removed. |
| Ctrl-T | Swaps the character at the cursor with the one before it. |
| Ctrl-_ | Undoes the last change of the line. |
| Up, Ctrl-P | Shows the line before in the history. |
| Down, Ctrl-N | Shows the next line in the history. |
| Ctrl-R | Searches the history backward. Each key adds to the search text. Enter takes the line that it found. |
| Ctrl-S | Searches the history forward. |
| Ctrl-G | Stops a search or a completion list. |
| Ctrl-L | Clears the screen. |
| Ctrl-C | Drops the line and shows `^C`. The line does not run, and the prompt stays open. |
| Ctrl-Z | Stops the CLI, as in a shell. `fg` brings the prompt back. |

Many terminals send Ctrl-S to the terminal itself, which stops the output. In that case, run `stty -ixon` first.

### Completion

Tab completes the word at the cursor:

| Word | Completes to |
| --- | --- |
| The first word, or the word after a command group such as `machine` | The commands of that group, and at the start of the line also `exit`, `help`, and `quit`. A hidden command does not show. |
| A word that starts with `-` | The flags of the command and the global flags, and `--help`. A flag that takes a value ends with `=`. |
| The value of a flag with a fixed set of values, as in `--restart ` or `--restart=` | Those values, such as `always` and `never`. |
| An argument named `MACHINE`, as in `connect`, `logs`, and `machine stop` | The names of the machines of the org. |
| The source or the target of `cp` | The names of the machines, each with `:` at its end. Local paths do not complete. |

A word that completes in one way gets a space at its end. If many words fit and they share more letters, Tab adds those letters. Else Tab shows the list of the words. In that list, Tab and the arrow keys move, Enter takes the word, and Ctrl-G or Ctrl-C closes the list. After `--`, as in `exec box1 -- ls`, nothing completes. Tab completes nothing in a terminal with fewer than three rows. A terminal of unknown size counts as 80 columns by 24 rows.

The names of the machines are live. When the prompt opens, it opens a `WatchMachines` stream ([the machine API](/reference/machine-api/index.html.md)) for the org of its commands. It keeps the names that the stream sends, as the console does. It leaves out a name that breaks the rule of machine names in [SSH files](#ssh-files). So an install cannot put a space or a control character into your line. A machine that another client makes or renames thus completes at once. If the stream breaks, the prompt keeps the names that it has. It opens a new stream after 1 s, 2 s, 5 s, 10 s, and then every 30 s. A stream that sends its first list resets these waits. If the install refuses the stream, for example because the profile is signed out, the prompt stops it. After each command, the prompt checks the profile, its install, its sign-in, and the org again. If one of them changed, or the stream stopped, it opens a new stream, for example after `hopbox login` or `hopbox org use` in the prompt. With `--debug`, the prompt logs each break of the stream, and its stop, to stderr.

### History

The prompt keeps the lines that you run in the file `history` in the config dir ([Files](#files)). The file has mode 0600, and it cannot be a symlink. At its open, the prompt gives mode 0600 to a file with another mode. The prompt keeps the last 1000 lines. When it opens, it reads the file, and cuts a longer file to its last 1000 lines, under the lock of the config dir. Then it adds each new line at the end of the file. Two prompts at once both add their lines.

The history leaves out a line that can hold a secret:

- A line that starts with a space or a tab.
- `env set NAME=VALUE`. `env set NAME` without a value stays in the history, because the value comes from stdin.
- `org join`, because the link of an invitation is a secret.
- A line with `-e NAME=VALUE`, `-eNAME=VALUE`, `--env NAME=VALUE`, or `--env=NAME=VALUE`, as in `exec`. This also counts `e` in a group of short flags, as in `-te NAME=VALUE` and `-teNAME=VALUE`.
- A line with a quote that does not close.

If the prompt cannot open the file, it logs `the prompt keeps no history file` at level warn, with the error. Then the history lasts only as long as the prompt.

### Commands and signals in the prompt

A command of the prompt gets the terminal and the signals as in a shell. A Ctrl-C (SIGINT) or a SIGTERM during a command acts as outside the prompt, as [Exit status](#exit-status) and [exec](#exec) say. When the signal stops the command, it stops only that command, and the prompt stays open. A command that does not end can get a second signal, which then ends the CLI at once, as outside the prompt. `connect` and `exec --tty` put the terminal in raw mode, and send the keys to the machine until the shell ends. After that, the next prompt gets the keys again. A command that reads stdin, as `exec` does, gets the keys that you type while it runs. Other keys wait for the next prompt. This is true also for a read of the terminal that a command left behind at its end.

`machine delete` asks its question on the terminal. `env set NAME` reads the value from the terminal with no echo, as in a shell. Each of them reads its answer up to the first CR or LF, and no more. So a paste of `env set NAME`, a value, and a next command, each with its Enter, sets the value and then runs the command. A Ctrl-C at the question ends the command, and the prompt stays open.

At the prompt itself, the keys Ctrl-C and Ctrl-D are keys, not signals. A SIGINT, a SIGTERM, a SIGHUP, or a SIGQUIT that comes from another process closes the prompt, and puts the terminal back. For example, the close of the window of the terminal sends a SIGHUP. The line typed so far then does not run. The prompt prints a newline, and the CLI exits with 128 plus the number of the signal.

| Status | When |
| --- | --- |
| 0 | `exit`, `quit`, or Ctrl-D on an empty line. The status of the last command does not count. |
| 80 | The prompt cannot open. |
| 130 | A SIGINT from another process closed the prompt. |
| 143 | A SIGTERM closed the prompt. |
| 129 | A SIGHUP closed the prompt. |
| 131 | A SIGQUIT closed the prompt. |
| 1 | The line editor did not start, or it failed to read a line. |

## Commands

| Command | Description |
| --- | --- |
| [`new`](#new), `machine create` | Makes a machine, and waits until it runs. |
| [`list`](#list), `ls`, `machine list`, `machine ls` | Lists the machines of the org. |
| [`connect`](#connect) | Opens a shell in a machine through the API, without SSH keys. |
| [`exec`](#exec) | Runs a command in a machine, and exits with its exit code. |
| [`cp`](#cp) | Copies files and directories into and out of a machine. |
| [`logs`](#logs) | Prints the log of the start command of a machine. |
| [`machine show`](#machine-show) | Prints one machine. |
| [`machine start`](#machine-start) | Starts a stopped machine, and waits until it runs. |
| [`machine stop`](#machine-stop) | Stops a machine, and waits until it stops. |
| [`machine reboot`](#machine-reboot) | Boots a running machine again, and waits for the new boot. |
| [`machine rename`](#machine-rename) | Gives a machine a new name. |
| [`machine delete`](#machine-delete), `machine rm` | Deletes a machine and its disk. |
| [`machine share`](#machine-share) | Opens a machine to the whole org. |
| [`machine unshare`](#machine-unshare) | Makes a shared machine private to its creator again. |
| [`machine update`](#machine-update) | Changes the restart policy, the start command, or the auto-delete timeout. |
| [`machine sizes`](#machine-sizes) | Lists the sizes that a machine can have. |
| [`url port`](#url-port) | Sets the port in the machine that its URL reaches. |
| [`url public`](#url-public) | Lets anyone open the URL of a machine. |
| [`url private`](#url-private) | Lets only accounts with access to the machine open its URL. |
| [`env list`](#env-list), `env ls` | Lists the env vars and secrets of the org. |
| [`env set`](#env-set) | Sets an env var or a secret. |
| [`env unset`](#env-unset), `env rm` | Removes an env var or a secret. |
| [`login`](#login) | Signs in to an install with a browser, and makes the profile active. |
| [`logout`](#logout) | Ends the CLI sign-in of the profile, and deletes its tokens. |
| [`whoami`](#whoami) | Prints the account, the org, and the install of the profile. |
| [`org list`](#org-list), `org ls` | Lists your orgs. |
| [`org show`](#org-show) | Prints one org. |
| [`org create`](#org-create) | Makes an org with you as its owner. |
| [`org use`](#org-use) | Makes an org the active org of the profile. |
| [`org rename`](#org-rename) | Changes the name of an org. |
| [`org delete`](#org-delete) | Deletes an org and everything in it. |
| [`org leave`](#org-leave) | Takes you out of an org, and deletes the machines that you made in it and did not share. |
| [`org usage`](#org-usage) | Prints what an org uses, and its limits. |
| [`org member list`](#org-member-list), `org member ls` | Lists the members of the org. |
| [`org member set-role`](#org-member-set-role) | Changes the role of a member. |
| [`org member remove`](#org-member-remove), `org member rm` | Removes a member from the org. |
| [`org invite`](#org-invite) | Invites an email to join the org. |
| [`org invitation list`](#org-invitation-list), `org invitation ls` | Lists the open invitations of the org. |
| [`org invitation revoke`](#org-invitation-revoke) | Ends an open invitation. |
| [`org join`](#org-join) | Joins an org with the link of an invitation. |
| [`api-key create`](#api-key-create) | Makes an API key, and prints its secret once. |
| [`api-key list`](#api-key-list), `api-key ls` | Lists the API keys of the org that you manage. |
| [`api-key revoke`](#api-key-revoke) | Deletes an API key. |
| [`ssh-key add`](#ssh-key-add) | Adds a public key to your account. |
| [`ssh-key list`](#ssh-key-list), `ssh-key ls` | Lists the SSH keys of your account. |
| [`ssh-key remove`](#ssh-key-remove), `ssh-key rm` | Removes an SSH key from your account. |
| [`profile list`](#profile-list), `profile ls` | Lists the profiles. |
| [`profile use`](#profile-use) | Makes a profile the active profile. |
| [`profile remove`](#profile-remove), `profile rm` | Ends the CLI sign-in of a profile, and deletes the profile. |
| [`version`](#version) | Prints the version of the CLI ([the version command](/reference/version/index.html.md)). |

`hopbox prompt` opens [the prompt](#the-prompt), as `hopbox` without a command does. The help of the CLI does not list it.

### new

```sh
hopbox new [NAME] [flags]
```

Makes a machine in the org of the command with `CreateMachine` ([the machine API](/reference/machine-api/index.html.md#createmachine)). The CLI adds the machine to the managed SSH block. Then it watches the machine with `WatchMachines` until the machine runs, and prints it as [machine show](#machine-show) does. `machine create` is the same command. Each try of the call carries the same idempotency key, so a retry never makes a second machine.

| Flag | Default | Description |
| --- | --- | --- |
| `NAME` | a free name, such as `box1` | The name: a lowercase DNS label of 1 to 63 characters that starts with a letter. |
| `--size=SIZE` | `tiny` in the personal org of a limited account, else `default` | A size from [machine sizes](#machine-sizes), which marks the default of the org. |
| `--image=IMAGE` | `default` | The image. |
| `--restart=POLICY` | `always` | The restart policy: `always` or `never`. |
| `--start-command=CMD` | none | A command that the machine runs as `user` after each boot. |
| `--auto-delete=TIMEOUT` | `off` | Deletes the machine after this time without use. |
| `--wait=DURATION` | `5m` | The longest wait until the machine runs. `0` prints the new machine at once, in the state `pending`. |

JSON: the machine object of [Machines in commands](#machines-in-commands).

| Error | Status | Cause |
| --- | --- | --- |
| `already_exists: already exists: the machine name "box1" is taken. A free name is "box2"` | 5 | Another machine holds the name, or another org holds it in reservation. The message names a free name, as the install suggests it. |
| `unavailable: ... no capacity: ...` | 6 | No node has room for the machine ([Placement](/reference/placement/index.html.md)). |
| `the machine <name> does not run after <wait>. It is <state>. hopbox machine show <name> shows its state later` | 6 | The wait ended first. The machine exists, and can still start. |
| `the machine <name> failed to start. hopbox machine show <name> shows it` | 1 | The machine reached the state `failed`. |

### list

```sh
hopbox list
```

Lists the machines of the org that you see, by name. `ls`, `machine list`, and `machine ls` are the same command. A state on its way shows both states, such as `stopped -> running`.

```text
NAME  STATE    SIZE     SHARED  URL                       SSH
box1  running  default  no      https://box1.example.com  box1.hop
```

The list also brings the managed SSH block up to date: the entries of the org on this install then match the list ([SSH files](#ssh-files)). JSON: an array of machine objects.

### connect

```sh
hopbox connect MACHINE
```

Opens the login shell of `user` in a running machine with `Exec`, with `tty` set and no `argv` ([the machine API](/reference/machine-api/index.html.md#exec)). It needs no SSH key, and it uses only the port of the API, 443, so it works from networks with few open ports.

If stdin is a terminal, the CLI does these things:

1. It sends the size of the terminal and the env var `TERM`.
2. It puts the terminal in raw mode. Each key then goes to the shell, Ctrl-C, Ctrl-Z, and Ctrl-D too.
3. At each SIGWINCH, it sends the new size of the terminal.
4. When the shell ends, it puts the terminal back as it was. It does the same after an error and after a signal.

The exit status is the exit status of the shell, so `exit 3` in the shell ends `hopbox connect` with 3. A machine that does not run fails with `failed_precondition`, status 1. Phase 1 has no standby and no hibernate, so `connect` wakes no machine. `connect` has no JSON output: `--json` fails with status 80.

### exec

```sh
hopbox exec [flags] MACHINE [COMMAND [ARG...]]
```

Runs a command in a running machine with `Exec`, and exits with the exit code of the command.

The flags of `exec` go before the command. Each word from the command on belongs to the command, flags too. So `hopbox exec box1 ls -la` runs `ls -la`, and `hopbox exec box1 echo --json` prints `--json`. A `--` before the command also works. Without a command, the login shell of the user runs, and reads its commands from stdin. With `--tty` and no command, `exec` opens a shell, as `connect` does.

| Flag | Default | Description |
| --- | --- | --- |
| `-e`, `--env=NAME[=VALUE]` | none | An env var for the command, on top of the env of the machine. `NAME` alone takes the value of `NAME` on the laptop, also an empty value. If `NAME` is not set on the laptop, the command fails with status 80. Repeat the flag for more env vars. A value can hold commas. |
| `--timeout=DURATION` | none | Ends the command after this time, 1 s to 24 h. The exit code is then 124. |
| `-t`, `--tty` | `false` | Runs the command in a terminal. The command then sends all its output to stdout. If stdin is a terminal, the CLI handles it as `connect` does. |
| `-u`, `--user=USER` | `user` | The Linux user that runs the command. |
| `-w`, `--dir=DIR` | the home dir of the user | The working dir. |

The CLI sends its stdin to the command, as ssh does, and then the end of the input. The stdout of the command goes to stdout, and its stderr to stderr. A command in a shell loop that reads stdin takes the input of the loop. Give it `</dev/null` to stop that.

The exit status of `hopbox exec` is the exit code of the command:

| Exit code | When |
| --- | --- |
| The code of the command | The command ended by itself. A code out of 0 to 255 becomes 255. |
| 124 | `--timeout` ended the command. |
| 126 | The command did not start. The CLI prints `hopbox: <reason>` to stderr. |
| 127 | The program does not exist. The CLI prints `hopbox: <reason>` to stderr. |
| 128 plus the signal number | A signal ended the command, for example 130 for SIGINT. |

The statuses of [Exit status](#exit-status) apply only to a command that did not run. For example, a machine that does not exist gives 4, and a machine that does not run gives 1.

A Ctrl-C (SIGINT) or a SIGTERM goes to the process group of the command, as the signal with the same name. The command can end, or go on. A second Ctrl-C or SIGTERM ends the exec. The API then ends the command with SIGTERM, and after 2 s with SIGKILL. The status is then 130 for SIGINT, or 143 for SIGTERM. With `--tty` and a terminal, Ctrl-C is a key for the command, as in `connect`.

With `--json`, the CLI keeps the output and prints one JSON object at the end:

```json
{"exit_code":3,"signal":"","error":"","timed_out":false,"stdout":"","stderr":"boom\n","stdout_truncated":false,"stderr_truncated":false}
```

| Field | Type | Description |
| --- | --- | --- |
| `exit_code` | number | The exit code of the command. |
| `signal` | string | The signal that ended the command, such as `SIGKILL`, or empty. |
| `error` | string | Why the command did not start, or empty. |
| `timed_out` | bool | Whether `--timeout` ended the command. |
| `stdout`, `stderr` | string | The output. The CLI keeps at most 4 MiB of each (`cli.MaxJSONOutput`). A byte that is not UTF-8 becomes U+FFFD. |
| `stdout_truncated`, `stderr_truncated` | bool | Whether the output had more than 4 MiB. |

The exit status with `--json` is the exit code of the command too. `--json` with `--tty` fails with status 80, because a terminal sends all output to stdout.

### cp

```sh
hopbox cp [-r] [-u USER] SOURCE TARGET
```

Copies a file or a directory into a running machine with `Upload`, or out of it with `Download` ([the machine API](/reference/machine-api/index.html.md#upload)). One of `SOURCE` and `TARGET` is in a machine, in the form `MACHINE:PATH`, such as `box1:app/main.go`. `MACHINE` is a machine name or UUID. A relative `PATH` starts at the home dir of the user. If a local path starts with a word and a colon, such as `notes:old`, write it as `./notes:old`.

| Flag | Default | Description |
| --- | --- | --- |
| `-r`, `--recursive` | `false` | Copies a directory. Its contents go into the target directory. |
| `-u`, `--user=USER` | `user` | The Linux user in the machine. It owns the new files, and a relative path starts at its home dir. |

A copy into a machine:

| `SOURCE` | What the machine gets |
| --- | --- |
| A file | The file `PATH`, with the permission bits of the local file. |
| A directory, with `-r` | A tar stream of the contents of the directory. The machine extracts the contents into the directory `PATH`. It makes a missing `PATH` first. Symlinks stay symlinks. A socket or a device fails the copy. |
| `-` | stdin, as the contents of the file `PATH`. With `-r`, stdin is a tar stream, and the machine extracts it into the directory `PATH`. |

An empty `PATH`, or a `PATH` that ends with `/`, gets the base name of `SOURCE`. So `hopbox cp run.sh box1:bin/` writes `bin/run.sh`, and `hopbox cp -r app box1:` writes the directory `app`. stdin has no name, so `-` needs a `PATH`. If the read of the local file fails, the CLI cancels the upload, so the machine keeps no file of a clean end.

A copy out of a machine:

| `TARGET` | What the laptop gets |
| --- | --- |
| A path | The file, with the permission bits of the file in the machine. A temporary file in the same dir takes the data first, so a failed copy leaves no half file. |
| A directory that exists, or a path that ends with `/` | The file in that directory, with the base name of `PATH`. |
| A directory, with `-r` | The contents of the directory `PATH`. The CLI makes a missing `TARGET` first. |
| `-` | The data on stdout: the file, or with `-r` the tar stream of the directory. |

The CLI keeps the whole tar stream of a directory in a temporary file first. It extracts the stream only after the download ended well. Before the first write, it checks the name of every entry. A name with `..` that leads out of `TARGET`, or an absolute name, fails the copy, and the copy writes nothing. Then the CLI writes each entry through an `os.Root` of `TARGET`, so a write through a symlink that leads out of `TARGET` fails too. A symlink in the archive keeps its target as it is, also a target outside `TARGET`, as `tar` does. Only the writes of the copy stay inside `TARGET`. The CLI skips an entry that is not a file, a directory, or a link, and logs a warning. A download to a laptop has at most 4 MiB ([the machine API](/reference/machine-api/index.html.md#limits)). A larger one fails with `failed_precondition`, status 1.

| Error | Status |
| --- | --- |
| `both paths are in a machine. Copy through this laptop, in two steps` | 80 |
| `neither path is in a machine. Write one as MACHINE:PATH, such as box1:app/main.go` | 80 |
| `<path> is a directory. Copy it with -r` | 80 |
| `stdin has no name. Give the target path in the machine, such as box1:notes.txt` | 80 |
| `the archive holds the path "<name>", which leaves the directory` | 1 |

On success, `cp` prints nothing, as cp does. JSON fields: `source`, `target`, `bytes` (the bytes that the copy moved), and `archive` (whether the data was a tar stream).

### logs

```sh
hopbox logs MACHINE [-f] [-n N] [--since=WHEN]
```

Prints the log of the start command of a machine with `StreamMachineLogs` ([Machine logs](/reference/machine-logs/index.html.md)). A line from the stdout of the start command goes to stdout, and a line from its stderr goes to stderr. The machine can be stopped. When the node lost lines, the CLI prints `hopbox: <count> lines of the log were lost here` to stderr.

| Flag | Default | Description |
| --- | --- | --- |
| `-f`, `--follow` | `false` | After the old lines, prints each new line as it comes. Ctrl-C ends it with status 130. |
| `-n`, `--tail=N` | `0`, all lines | Prints only the last `N` lines, at most 10000. |
| `--since=WHEN` | all lines | Prints only the lines from this time on: a duration back from now, such as `10m`, or an RFC 3339 time. |

With `--json`, the CLI prints one JSON object for each line, on a line of its own (JSON Lines). It prints no array, so `--follow` works with it. Each object has `time`, `stream` (`stdout` or `stderr`), `data`, and `dropped` (the count of lines lost before this one).

### machine show

```sh
hopbox machine show MACHINE
```

Prints one machine, as [Machines in commands](#machines-in-commands) shows.

### machine start

```sh
hopbox machine start MACHINE
```

Starts a stopped machine with `StartMachine`, and waits until it runs.

```text
Started the machine box1 in 1.204s.
```

The install waits for the machine for up to 60 s ([the machine API](/reference/machine-api/index.html.md#the-wait)). If the machine is not there at the end of the wait, the command fails with status 6: `the machine <name> is not <state> after <time>. It is <state>. hopbox machine show <name> shows its state later`. The machine keeps its target, so it can still get there. The commands `machine stop`, `machine reboot`, `machine rename`, and `machine delete` work in the same way.

JSON fields of each of these commands: `machine`, the machine object, `duration_seconds`, how long the call took, and `done`, which is always `true`.

### machine stop

```sh
hopbox machine stop MACHINE
```

Stops a machine with `StopMachine`, and waits until it stops. Its disk stays.

### machine reboot

```sh
hopbox machine reboot MACHINE
```

Boots a running machine again with `RebootMachine`, and waits for the new boot.

### machine rename

```sh
hopbox machine rename MACHINE NAME
```

Gives a machine the new name `NAME` with `RenameMachine`. A running machine reboots, and the command waits for the new boot. The old name stays reserved for the org for 7 days. The CLI moves the machine to its new alias in the managed SSH block. A taken name fails with status 5, as at [new](#new).

### machine delete

```sh
hopbox machine delete MACHINE [--yes]
```

Deletes a machine and its disk with `DeleteMachine`, and waits until it is gone. `machine rm` is the same command. Its name and its SSH port stay reserved for 7 days. The CLI removes the machine from the managed SSH block.

Before the delete, the CLI asks on stderr: `Delete the machine <name> and its disk? Its name stays reserved for 7 days. [y/N]`. The answer ends at the first CR or LF. Only `y` or `yes` deletes. Another answer changes nothing, and fails with `nothing changed`, status 5. A Ctrl-C at the question deletes nothing, and ends the command with status 130. If stdin is not a terminal, or with `--json`, the CLI cannot ask. The command then needs `--yes`, or it fails with status 80.

| Flag | Default | Description |
| --- | --- | --- |
| `-y`, `--yes` | `false` | Deletes without the question. |

### machine share

```sh
hopbox machine share MACHINE
```

Opens a private machine to the whole org with `ShareMachine`, and prints the machine. Only its creator can. If the env of the machine as a shared machine does not resolve, for example because a secret of the org does not open, the share fails, and the machine stays private ([the machine API](/reference/machine-api/index.html.md#sharemachine)).

### machine unshare

```sh
hopbox machine unshare MACHINE
```

Makes a shared machine private to its creator again with `UnshareMachine`, and prints the machine. Only its creator can.

### machine update

```sh
hopbox machine update MACHINE [--restart=POLICY] [--start-command=CMD] [--auto-delete=TIMEOUT]
```

Changes the config of a machine with `UpdateMachine`, and prints the machine. A flag that you leave out keeps its value. Without a flag, the command fails with status 80.

| Flag | Description |
| --- | --- |
| `--restart=POLICY` | The new restart policy: `always` or `never`. |
| `--start-command=CMD` | The new start command. An empty value removes it. The machine runs it from its next boot. |
| `--auto-delete=TIMEOUT` | The new auto-delete timeout, or `off`. A new timeout counts from now. |

### machine sizes

```sh
hopbox machine sizes
```

Lists the sizes that a machine can have, smallest first, with `ListMachineSizes` ([the machine API](/reference/machine-api/index.html.md#listmachinesizes)). The list leaves out the retired sizes. An instance admin adds and retires sizes ([the admin API](/reference/admin-api/index.html.md#sizeservice)). The column `DEFAULT` marks with `yes` the size of `hopbox new` without `--size` in the org of the command: `tiny` in the personal org of a limited account, else `default`.

```text
NAME     VCPUS  MEMORY  DISK    DEFAULT
tiny     1      1 GiB   5 GB
default  2      8 GiB   100 GB  yes
```

JSON: an array of objects with `name`, `vcpus`, `memory_bytes`, `disk_bytes`, and `default`, which is `true` for the default size of the org.

### url port

```sh
hopbox url port MACHINE PORT
```

Sets the port in the machine that its URL reaches, 1 to 65535, with `SetMachineURLPort`. Only the creator of the machine can. The command prints the URL:

```text
https://box1.example.com (private, port 3000)
```

JSON: the machine object.

### url public

```sh
hopbox url public MACHINE
```

Lets anyone open the URL of the machine, without an account, with `SetMachineURLVisibility`. Only the creator of the machine can. It prints the URL as `url port` does.

### url private

```sh
hopbox url private MACHINE
```

Lets only signed-in accounts with access to the machine open its URL. Only the creator of the machine can. A new machine is private.

### env list

```sh
hopbox env list
```

Lists the env vars and secrets of the org that you see, by name, with `ListVars` ([the env var API](/reference/vars-api/index.html.md)). `env ls` is the same command. A secret shows `(secret)` and no value. A value with a newline shows its first line and `...`.

```text
NAME   SCOPE  SECRET  VALUE     UPDATED
TOKEN  all    yes     (secret)  2026-10-21T10:57:09Z
```

JSON: an array of objects. Each has `id`, `org_id`, `name`, `scope`, `secret`, and `value`, which is empty for a secret. Each also has `owner_account_id`, `updater_account_id`, `create_time`, and `update_time`.

### env set

```sh
hopbox env set NAME[=VALUE] [--secret] [--scope=SCOPE]
```

Sets an env var or a secret in the org of the command with `SetVar`. The machines get it at their next boot and in each new login shell. The CLI first checks the name with the rules of [the env var API](/reference/vars-api/index.html.md) for the scope. A bad name fails with status 80, before the CLI reads any value.

Without `=VALUE`, the value comes from stdin, so a secret stays out of the shell history:

- If stdin is a terminal, the CLI writes `Value of NAME: ` to stderr. Then it reads one line with no echo, so the value never shows on the screen. The line ends at the first CR or LF, and the CLI reads nothing after it. So in a paste, the next line stays for the next reader, such as [the prompt](#the-prompt). A Ctrl-C at this question turns the echo back on, and ends the command with status 130.
- Else it reads all of stdin, up to 64 KiB, and drops one newline at its end, as in `hopbox env set TOKEN --secret < token.txt`.

| Flag | Default | Description |
| --- | --- | --- |
| `--secret` | `false` | Seals the value. No call returns it again. A secret needs a key-encryption key on the install ([Sealing](/reference/sealing/index.html.md)). |
| `--scope=SCOPE` | `shared` | `shared`: the shared machines of the org. `private`: your own private machines. `all`: every machine of the org. A personal org refuses `shared`, so use `private` or `all` there. |

```text
Set the secret TOKEN in the scope all.
```

JSON: the object of [env list](#env-list).

### env unset

```sh
hopbox env unset NAME [--scope=SCOPE]
```

Removes the value `NAME` of a scope with `DeleteVar`. `env rm` is the same command. A name without a value fails with status 4. JSON fields: `name` and `scope`.

| Flag | Default | Description |
| --- | --- | --- |
| `--scope=SCOPE` | `shared` | The scope of the value: `shared`, `private`, or `all`. |

### login

```sh
hopbox login [--url=URL]
```

Signs in the profile of the command with the device flow, as [Sign-in and tokens](#sign-in-and-tokens) describes. The command waits until a person approves the code in the console, the code ends after 10 minutes, or you press Ctrl-C.

| Flag | Env var | Default | Description |
| --- | --- | --- | --- |
| `--url=URL` | `HOPBOX_URL` | the URL of the profile | The URL of the install. A new profile needs it. |

A URL that differs from the URL of the profile also drops the active org of the profile. That org belongs to the other install.

The command prints the code to stderr:

```text
To sign in, open this page in a browser and approve the code KWJQ-LBZV:

  https://example.com/device?code=KWJQ-LBZV

Waiting for the approval. Press Ctrl-C to stop.
```

Then the CLI prepares `~/.ssh` for the SSH front door ([SSH files](#ssh-files)). `--no-ssh-config` skips these steps:

1. It writes the host key of the install into `~/.ssh/known_hosts`, once the install has an SSH front door.
2. If the account has no SSH key, it adds the public key of `~/.ssh/hopbox_ed25519` to the account. For a missing file, it makes a new ed25519 key pair first. The key gets the name `hopbox CLI on <host name of the laptop>`. An account with a key keeps its keys as they are. `ssh-key add` adds more.

A failure of these steps does not fail the login. The CLI logs a warning, and the next command works. Then it prints the result to stdout:

```text
Signed in to https://example.com as ann@example.org. The profile default is active.
Added the SSH key ~/.ssh/hopbox_ed25519 (SHA256:rkEGrocWsFODhePtstCZFcyj1D0BUmGYu1rexwam/mo) to your account.
```

Only a login that added a key prints the second line. JSON fields: `profile`, `url`, `account`, the account object of [whoami](#whoami), and `ssh_key`. `ssh_key` is the key that the login added, as [ssh-key add](#ssh-key-add) prints it, or `null`.

| Error | Status | Cause |
| --- | --- | --- |
| `the profile <name> has no install yet. Give its URL: hopbox login --url https://hop.example.org` | 80 | The profile has no URL, and the command gives none. |
| `the install URL is not valid: ...` | 80 | The URL breaks a rule of [URLs of an install](#urls-of-an-install). |
| `start the sign-in at <url>: failed_precondition: ...` | 1 | The install has no domain, so it has no console page for the approval. |
| `the code expired before its approval. Run hopbox login again` | 1 | Nobody approved the code in 10 minutes. |
| `wait for the approval: ...` | per code | A poll failed. |

### logout

```sh
hopbox logout
```

Ends the CLI sign-in of the profile with `SignOut`, and deletes its tokens from `credentials.toml`. The call goes to the install that issued the tokens: the `url` of the tokens in `credentials.toml`. The profile can name another install by now. The profile keeps its URL and its active org. If the install does not answer, the CLI still deletes the tokens. It then logs `the install did not end the CLI sign-in, so only the local tokens go. End it in the console` at level warn. A profile that is not signed in is no error.

```text
Signed out the profile default.
```

JSON fields: `profile` and `signed_out`. For a profile that was not signed in, `signed_out` is `false`.

### whoami

```sh
hopbox whoami
```

Prints the profile, the URL of the install, the account, and the org of the command.

```text
profile         default
url             https://example.com
account         01a14a7c-813f-7a14-b6bc-44a4e57ac6ff
name            -
emails          root@example.org (unverified)
instance admin  yes
org             acme (01a14a7c-be1e-74cc-9748-ff40254c5d55)
role            owner
```

The line `instance admin` shows only for an instance admin. JSON fields:

| Field | Type | Description |
| --- | --- | --- |
| `profile` | string | The profile of the command. |
| `url` | string | The URL of the install. |
| `account` | object or null | The account. It is `null` for an org API key, which has no account. |
| `account.id` | string | The UUID of the account. |
| `account.display_name` | string | The display name, or empty. |
| `account.emails` | array | The emails: `email`, `verified`, and `verify_time`. |
| `account.instance_admin` | bool | Whether the account is an instance admin. |
| `account.personal_org_id` | string | The UUID of the personal org. |
| `account.version` | number | The version of the account. |
| `account.create_time` | string | The time of the first sign-in. |
| `org` | object or null | The org of the command, as [org show](#org-show) prints it. |

The active org of the profile can stop being one of your orgs. Then `whoami` prints the rest with `org` as `null`, and logs `the active org of the profile is not one of your orgs. Pick another with hopbox org use` at level warn. Other commands in that org fail until you run `hopbox org use`.

### org list

```sh
hopbox org list
```

Lists your orgs: the personal org first, then the others, oldest first. A star marks the active org.

```text
   ID                                    NAME      ROLE   PERSONAL  SUSPENDED
   01a14a7c-813f-7a1c-946b-db123b23c342  Personal  owner  yes       no
*  01a14a7c-be1e-74cc-9748-ff40254c5d55  acme      owner  no        no
```

JSON: an array of org objects. Each has the fields of [org show](#org-show).

### org show

```sh
hopbox org show [ORG]
```

Prints one org: `ORG`, by name or UUID, else the org of the command.

```text
id         01a14a7c-be1e-74cc-9748-ff40254c5d55
name       acme
role       owner
personal   no
active     yes
suspended  no
created    2026-10-17T15:30:51Z
```

A suspended org shows `suspended  yes, since <time>: <reason>`. JSON fields:

| Field | Type | Description |
| --- | --- | --- |
| `id` | string | The UUID of the org. |
| `name` | string | The name of the org. |
| `personal` | bool | Whether it is the personal org of the account. |
| `role` | string | Your role: `member`, `admin`, or `owner`. |
| `suspended` | bool | Whether an instance admin suspended the org. |
| `suspend_time` | string or null | The start of the suspension. |
| `suspend_reason` | string | The reason of the suspension, or empty. |
| `version` | number | The version of the org. |
| `create_time` | string | The creation time of the org. |
| `active` | bool | Whether it is the active org of the profile. |

### org create

```sh
hopbox org create NAME [--use]
```

Makes an org named `NAME`, with you as its owner, and prints it as [org show](#org-show) does. A name has 1 to 64 characters ([the org API](/reference/org-api/index.html.md#createorg)).

| Flag | Default | Description |
| --- | --- | --- |
| `--use` | `false` | Also makes the new org the active org of the profile. |

### org use

```sh
hopbox org use ORG
```

Makes `ORG`, by name or UUID, the active org of the profile. The CLI checks with `GetOrg` that you are in the org.

```text
The active org of the profile default is now acme (01a14a7c-be1e-74cc-9748-ff40254c5d55).
```

JSON: the org object of [org show](#org-show).

### org rename

```sh
hopbox org rename ORG NAME
```

Changes the name of `ORG` to `NAME`, and prints the org. Only an owner can.

### org delete

```sh
hopbox org delete ORG --confirm=NAME
```

Deletes `ORG` and every resource in it ([the org API](/reference/org-api/index.html.md#deleteorg)). Only an owner can. Nobody can delete a personal org.

| Flag | Default | Description |
| --- | --- | --- |
| `--confirm=NAME` | none, required | The name of the org. The delete runs only with the right name. A wrong name fails with `invalid_argument`. |

```text
Deleted the org 01a14a7c-be1e-74cc-9748-ff40254c5d55.
```

JSON fields: `id`.

### org leave

```sh
hopbox org leave ORG
```

Takes you out of `ORG`. hopbox deletes the machines that you made in the org and did not share ([the org API](/reference/org-api/index.html.md#what-a-member-leaves-behind)). The only owner of an org cannot leave it, and nobody leaves a personal org.

```text
You left the org 01a14a7c-be1e-74cc-9748-ff40254c5d55.
```

JSON fields: `id`.

### org usage

```sh
hopbox org usage [ORG]
```

Prints what the org uses, and its limits ([the org API](/reference/org-api/index.html.md#getorgusage)). The org is `ORG`, else the org of the command. RAM is in GiB, and disk is in GB, as the sizes of machines are.

```text
RESOURCE       USED   LIMIT
machines       0      3
vcpus          0      6
memory         0 GiB  24 GiB
disk           0 GB   300 GB
public urls           yes
outbound smtp         no
```

The row `public urls` says whether the creator of a machine can make its URL public. The personal org of a limited account says `no` ([the org API](/reference/org-api/index.html.md#usage-limits)). The row `outbound smtp` says whether the machines reach TCP port 25. Only an instance admin opens it, for one org.

JSON fields: `usage` and `limits`. Each has `machines`, `vcpus`, `memory_bytes`, and `disk_bytes`. `limits` also has `public_urls` and `outbound_smtp`.

### org member list

```sh
hopbox org member list
```

Lists the members of the org of the command, in the order that they joined.

```text
ACCOUNT                               EMAILS                         NAME  ROLE   JOINED
01a14a7c-813f-7a14-b6bc-44a4e57ac6ff  root@example.org (unverified)  -     owner  2026-10-17T15:30:51Z
```

JSON: an array of objects with `account_id`, `display_name`, `emails`, `role`, and `join_time`.

### org member set-role

```sh
hopbox org member set-role MEMBER ROLE
```

Changes the role of `MEMBER` to `ROLE`: `member`, `admin`, or `owner`. `MEMBER` is the UUID of the account, or one of its emails. Only an owner can, and the org keeps at least one owner.

```text
The account 01a14a7d-0d55-7a2e-b8f6-0d5c8f1e2a33 is now member.
```

JSON fields: `account_id` and `role`.

### org member remove

```sh
hopbox org member remove MEMBER
```

Takes `MEMBER`, by account UUID or email, out of the org. hopbox deletes the machines that the member made in the org and did not share. An admin removes members, and an owner removes anyone. JSON fields: `account_id`.

### org invite

```sh
hopbox org invite EMAIL [--role=ROLE]
```

Invites `EMAIL` to join the org of the command, and prints the link. With an SMTP config, the install also sends the link to the email. With the file mail transport, it writes the email to a file on the server and sends nothing ([Mail](/reference/mail/index.html.md)). A personal org takes no invitations.

| Flag | Default | Description |
| --- | --- | --- |
| `--role=ROLE` | `member` | The role that the invitation gives: `member`, `admin`, or `owner`. |

```text
https://example.com/invite#token=<token>
The link invites ann@example.org as admin until 2026-10-24T15:30:51Z. It is a secret until then. The install sent no email, so pass the link on yourself.
```

JSON fields: `invitation`, with the fields of [org invitation list](#org-invitation-list), `url`, and `email_sent`.

### org invitation list

```sh
hopbox org invitation list
```

Lists the open invitations of the org of the command, oldest first.

```text
ID                                    EMAIL            ROLE   EXPIRES
01a14a7c-be78-7610-b5cc-c5f5ce32bfee  ann@example.org  admin  2026-10-24T15:30:51Z
```

JSON: an array of objects with `id`, `org_id`, `email`, `role`, `inviter_account_id`, `create_time`, and `expire_time`.

### org invitation revoke

```sh
hopbox org invitation revoke ID
```

Ends the open invitation `ID` of the org of the command, so its link stops working. JSON fields: `id`.

### org join

```sh
hopbox org join LINK [--use]
```

Joins the org of an invitation. `LINK` is the whole link, or the token after `#token=` in it. Your account must hold the email of the invitation as a verified email.

The token of the link is a secret of its install. So the host of a whole link must be the host of the URL of the profile. A profile on a loopback address, such as `http://127.0.0.1:7080`, also takes the domain that `GetInstallInfo` returns, because its links name that domain. For another host, the command fails with status 80:

```text
the invitation is for the install at <host>, but the profile <name> signs in to <url>. Use the profile of that install with --profile
```

A bare token gets no such check.

| Flag | Default | Description |
| --- | --- | --- |
| `--use` | `false` | Also makes the org the active org of the profile. |

```text
You joined the org acme (01a14a7c-be1e-74cc-9748-ff40254c5d55) as admin.
```

JSON: the org object of [org show](#org-show).

### api-key create

```sh
hopbox api-key create NAME [--kind=KIND] [--expires=WHEN]
```

Makes an API key named `NAME` in the org of the command ([CLI tokens, API keys, admin API keys, and SSH keys](/reference/credentials/index.html.md#api-keys)). Without `--json`, the secret alone goes to stdout, and a note goes to stderr. So `KEY=$(hopbox api-key create ci)` works.

| Flag | Default | Description |
| --- | --- | --- |
| `--kind=KIND` | `member` | `member` acts as you in the org. `org` is a key of the org itself. Only org admins and owners make org keys. |
| `--expires=WHEN` | `90d` | The end of the key: `30d`, `90d`, `365d`, a date such as `2027-01-31`, or an RFC 3339 time. A date means midnight UTC at its start. A custom end is at most 365 days ahead. |

The secret goes to stdout:

```text
hbx_<key>
```

The note goes to stderr:

```text
The member key 01a14a7c-beb0-7c35-a4c4-5c81288a9055 (ci) works until 2026-11-16T15:30:51Z. This is its only copy.
```

JSON fields: `key`, with the fields of [api-key list](#api-key-list), and `secret`. The install keeps no copy of the secret. A retry of the call with the same idempotency key fails with `already_exists`, so the CLI never prints a secret twice.

### api-key list

```sh
hopbox api-key list
```

Lists the API keys of the org of the command that you manage: your own member keys as a member, every key of the org as an admin or owner.

```text
ID                                    NAME  KIND    EXPIRES               LAST USED
01a14a7c-beb0-7c35-a4c4-5c81288a9055  ci    member  2026-11-16T15:30:51Z  never
```

JSON: an array of objects. Each has `id`, `org_id`, `kind` (`member` or `org`), `name`, `account_id`, `creator_account_id`, `expire_time`, `last_use_time`, and `create_time`.

### api-key revoke

```sh
hopbox api-key revoke ID
```

Deletes the API key `ID` at once. JSON fields: `id`.

### ssh-key add

```sh
hopbox ssh-key add FILE [--name=NAME]
```

Adds the public key in `FILE` to your account ([CLI tokens, API keys, admin API keys, and SSH keys](/reference/credentials/index.html.md#ssh-keys)). `-` reads the key from stdin. The input must hold one public key line. Blank lines and lines that start with `#` do not count. A private key fails with status 80.

| Flag | Default | Description |
| --- | --- | --- |
| `--name=NAME` | the comment of the key | The name of the key, such as `ann@laptop`. |

```text
id           01a14a7c-e079-7778-9819-39ade2a33482
name         ann@laptop
type         ssh-ed25519 256
fingerprint  SHA256:rkEGrocWsFODhePtstCZFcyj1D0BUmGYu1rexwam/mo
```

JSON fields: `id`, `name`, `type`, `bits`, `public_key`, `fingerprint`, and `create_time`. A key that an account of the install holds already fails with `already_exists`, status 5.

### ssh-key list

```sh
hopbox ssh-key list
```

Lists the SSH keys of your account, oldest first.

```text
ID                                    NAME        TYPE         FINGERPRINT                                         ADDED
01a14a7c-e079-7778-9819-39ade2a33482  ann@laptop  ssh-ed25519  SHA256:rkEGrocWsFODhePtstCZFcyj1D0BUmGYu1rexwam/mo  2026-10-17T15:31:00Z
```

JSON: an array of the objects of [ssh-key add](#ssh-key-add).

### ssh-key remove

```sh
hopbox ssh-key remove KEY
```

Removes an SSH key from your account. `KEY` is the UUID of the key, or its fingerprint, with or without `SHA256:`. JSON fields: `id`.

### profile list

```sh
hopbox profile list
```

Lists the profiles by name. A star marks the active profile. The column `SIGNED IN` says whether the profile holds CLI tokens of its install. The CLI does not ask the install whether the sign-in still works.

```text
   PROFILE  URL                  SIGNED IN  ORG
*  default  https://example.com  yes        01a14a7c-be1e-74cc-9748-ff40254c5d55
```

JSON: an array of objects with `name`, `url`, `org`, `signed_in`, and `active`.

### profile use

```sh
hopbox profile use NAME
```

Makes the profile `NAME` the active profile. A name without a profile fails with status 4. JSON fields: `profile`.

### profile remove

```sh
hopbox profile remove NAME
```

Ends the CLI sign-in of the profile `NAME` as [logout](#logout) does. Then it deletes the profile and its tokens. If it was the active profile, no profile is active, and commands use `default`. JSON fields: `profile`.

### version

```sh
hopbox version
```

Prints the version of the CLI. With `--json`, it prints the JSON object of [the version command](/reference/version/index.html.md).

## Exit status

| Status | Meaning |
| --- | --- |
| 0 | The command succeeded. |
| 1 | The command failed for another reason, for example `invalid_argument` or `failed_precondition`. |
| 2 | Not signed in: the profile has no CLI sign-in, the sign-in ended, or the install refused the credential (`unauthenticated`). |
| 3 | The install refused the call: `permission_denied`. |
| 4 | Something is not there: `not_found`, an org or a member that a name does not match, or a profile that does not exist. |
| 5 | A conflict: `already_exists` or `aborted`. Also an answer other than `y` to the question of `machine delete`. |
| 6 | The install cannot serve the call now, after the retries: `unavailable`, `resource_exhausted`, or `deadline_exceeded`. Try again later. Also a machine that did not reach its state within the wait of `new` or of a lifecycle command. |
| 80 | The command line is not valid, for example an unknown command, an unknown flag, a bad profile name, or a bad URL. Also a command that deletes data without `--yes` and without a terminal for its question. |
| 130 | You stopped the command with Ctrl-C (SIGINT). The command then ends its work. A second Ctrl-C ends the process at once. |
| 143 | SIGTERM stopped the command. |

The prompt has statuses of its own ([Commands and signals in the prompt](#commands-and-signals-in-the-prompt)). `exec` and `connect` exit with the exit code of the command in the machine, once the command ran ([exec](#exec)). So for them, a status such as 1 or 4 can also come from the command. Use `exec --json` to tell the two apart: its object holds the exit code of the command.
