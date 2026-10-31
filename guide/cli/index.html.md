# Use the hopbox CLI

> How to sign the hopbox CLI in, work in an org, make API keys and SSH keys, script it with JSON, keep the sign-in alive, use a second install, and sign out.

The hopbox CLI signs in to an install with your browser. Then it manages your machines, orgs, API keys, and SSH keys. This guide runs each kind of command once against the install `https://example.com`. [Use machines from the CLI](/guide/cli-machines/index.html.md) covers the machines. [hopbox CLI](/reference/cli/index.html.md) lists every command, flag, file, and exit status.

## What you need

- The hopbox CLI, installed and signed in to an install, as in [Install the hopbox CLI](/guide/install-cli/index.html.md).
- `python3`, for one of the examples.

## Sign in

`hopbox login` signs in with a device flow. The CLI prints a code and a link. You open the link in a browser where you are signed in to the console, check the code, and click "Approve" ([Install the hopbox CLI](/guide/install-cli/index.html.md#sign-in-to-an-install)):

```sh
hopbox login --url https://example.com
```

The approval needs a sign-in to the console within the last 10 minutes. After an older sign-in, the page asks you to sign in again, then shows the code again.

The CLI keeps its tokens in `~/.config/hopbox/credentials.toml`, mode 0600, and the URL of the install in `~/.config/hopbox/config.toml` ([hopbox CLI](/reference/cli/index.html.md#files)). Ask who you are:

```sh
hopbox whoami
```

```text
profile  default
url      https://example.com
account  01a14a7c-813f-7a14-b6bc-44a4e57ac6ff
name     user1
emails   user1@example.org
org      Personal (01a14a7c-813f-7a1c-946b-db123b23c342)
role     owner
```

## Work in an org

Each command acts in one org. Until you pick one, that is your personal org. Make an org for a team, and make it the active org of the profile. A limited account creates no org: an instance admin first moves it to the tier full ([the org API](/reference/org-api/index.html.md#usage-limits)).

```sh
hopbox org create org1 --use
hopbox org list
```

```text
id         01a14a7c-be1e-74cc-9748-ff40254c5d55
name       org1
role       owner
personal   no
active     yes
suspended  no
created    2026-10-17T15:30:51Z
   ID                                    NAME      ROLE   PERSONAL  SUSPENDED
   01a14a7c-813f-7a1c-946b-db123b23c342  Personal  owner  yes       no
*  01a14a7c-be1e-74cc-9748-ff40254c5d55  org1      owner  no        no
```

The star marks the active org. Commands now act in `org1`. One command can act in another org with `--org`, by name or UUID, such as `hopbox --org Personal org usage`. See what the org uses, and invite a colleague:

```sh
hopbox org usage
hopbox org invite user2@example.org --role admin
```

```text
RESOURCE       USED   LIMIT
machines       0      3
vcpus          0      6
memory         0 GiB  24 GiB
disk           0 GB   300 GB
public urls           yes
outbound smtp         no
https://example.com/invite#token=<token>
The link invites user2@example.org as admin until 2026-10-24T15:30:51Z. It is a secret until then. The install sent no email, so pass the link on yourself.
```

With an SMTP config, the install also emails the link. The colleague signs in, and joins with `hopbox org join '<link>'`, or opens the link in a browser. Their account must hold the email of the invitation as a verified email. Then `hopbox org member list` shows them, and `hopbox org member set-role user2@example.org member` changes their role. [Work in a team org](/guide/orgs/index.html.md) covers the roles.

## Make an API key for a script

A script uses an API key, not your CLI sign-in. A member key acts as you in the active org:

```sh
KEY=$(hopbox api-key create ci --expires 30d)
hopbox api-key list
```

```text
The member key 01a14a7c-beb0-7c35-a4c4-5c81288a9055 (ci) works until 2026-11-16T15:30:51Z. This is its only copy.
ID                                    NAME  KIND    EXPIRES               LAST USED
01a14a7c-beb0-7c35-a4c4-5c81288a9055  ci    member  2026-11-16T15:30:51Z  never
```

The secret went to `$KEY`, and the note went to stderr. The install keeps only a hash of the key, so this is its only copy. Give the key to the CLI of the script in `HOPBOX_API_KEY`, and the URL of its install in `HOPBOX_URL`. The CLI never sends a key to the URL of a profile, because a profile can name another install:

```sh
HOPBOX_API_KEY=$KEY HOPBOX_URL=https://example.com hopbox whoami
HOPBOX_API_KEY=$KEY HOPBOX_URL=https://example.com hopbox api-key create child; echo "status $?"
```

```text
profile  default
url      https://example.com
account  01a14a7c-813f-7a14-b6bc-44a4e57ac6ff
name     user1
emails   user1@example.org
org      org1 (01a14a7c-be1e-74cc-9748-ff40254c5d55)
role     owner
hopbox: permission_denied: permission denied: a caller of kind member_key may not do api_key.create
status 3
```

The key works in its own org, `org1`. No key can make another key, and the status 3 tells a script that the install refused the call. After the last run of the script, revoke the key with `hopbox api-key revoke <id>`.

An org admin or an org owner also makes org keys, with `--kind org`. An org key is a credential of the org, not of a person. So if its creator leaves the org, the key stays. [CLI tokens, API keys, admin API keys, and SSH keys](/reference/credentials/index.html.md#api-keys) gives the rights of each kind.

## Add an SSH key

The SSH front door finds your account by your SSH key ([Reach a machine over SSH](/guide/ssh/index.html.md)). The login added `~/.ssh/hopbox_ed25519`. To use another key too, add its public key, never the private key:

```sh
hopbox ssh-key add ~/.ssh/id_ed25519.pub
```

```text
id           01a14a7c-e079-7778-9819-39ade2a33482
name         user1@laptop
type         ssh-ed25519 256
fingerprint  SHA256:DRdf39Nvm+zAXRzjpCOtwde9E/UUaXIbgz0+g1NA+10
```

The comment of the key became its name. RSA keys need at least 3072 bits, so Ed25519 is the simple choice. A key works for one account only, so the same key on another account fails with `already_exists`. `hopbox ssh-key add ~/.ssh/id_ed25519` fails with `this is a private key. Give the public key, the .pub file`, and status 80.

## Use the output in a script

Each command prints JSON with `--json`. An error then goes to stderr as JSON too:

```sh
hopbox org list --json | python3 -c 'import json,sys; print([o["name"] for o in json.load(sys.stdin)])'
hopbox --profile nope whoami --json; echo "status $?"
```

```text
['Personal', 'org1']
{"error":{"code":"signed_out","message":"not signed in: the profile nope has no install. Run hopbox login --url <URL>"}}
status 2
```

Status 2 always means that the profile has no live sign-in. [hopbox CLI](/reference/cli/index.html.md#exit-status) lists each status.

## Keep the sign-in alive

The access token lives 1 hour, and the refresh token lives 30 days. The CLI trades them for new ones by itself. With `--debug`, each refresh writes the log line `refreshed the CLI tokens` to stderr.

A refresh token works once. A used one that comes back ends the whole sign-in, because the install takes it for a stolen copy. So the CLI refreshes under a lock of its config dir. Many commands at once, in many terminals, make one refresh, and the others read the new tokens. [CLI tokens, API keys, admin API keys, and SSH keys](/reference/credentials/index.html.md#cli-tokens) gives the rules.

## Sign in to a second install

A profile holds one install. To use a second install, such as the install `hop.example.com` of your company, sign in with another profile:

```sh
hopbox --profile work login --url hop.example.com
```

A login makes its profile active. List the profiles, and switch back:

```sh
hopbox profile list
hopbox profile use default
```

```text
   PROFILE  URL                      SIGNED IN  ORG
   default  https://example.com      yes        01a14a7c-be1e-74cc-9748-ff40254c5d55
*  work     https://hop.example.com  yes        -
The active profile is now default.
```

`HOPBOX_PROFILE=work` picks a profile for one shell, and `--profile work` for one command.

## Sign out

```sh
hopbox logout
hopbox whoami; echo "status $?"
```

```text
Signed out the profile default.
hopbox: not signed in: the profile default has no CLI sign-in. Run hopbox login
status 2
```

`logout` ended the CLI sign-in on the install too, so a copy of its tokens does not work anymore. The profile keeps its URL and its active org, so `hopbox login` alone signs in again.

If you lose a laptop, end its sign-in from another device. Open the page "Account" of the console. The section "CLI sign-ins" lists each sign-in with the name of its client and its last use. Click "End" on the row of the lost laptop.
