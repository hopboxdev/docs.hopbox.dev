# Use machines from the CLI

> How to make and use a machine with the hopbox CLI: commands, files, the log, a shell, and the SSH alias.

This guide makes a machine with the hopbox CLI, works in it, and deletes it. It runs a command, copies files, reads the log, opens a shell, and reaches the machine by its SSH alias. [hopbox CLI](/reference/cli/index.html.md#machines-in-commands) lists every command, flag, and exit status.

Each machine is a Firecracker microVM on a node of the install. It gets an address on the network of its node ([Check the network of a machine](/guide/node-network/index.html.md)). `exec`, `cp`, `logs`, and `connect` reach the machine through the API. The SSH alias and the URL of a machine reach it through the edge servers of the install.

## What you need

- An account on an install with machines, such as the preview install of [Try hopbox](/guide/quickstart/index.html.md). This guide uses the install `https://example.com`.
- The hopbox CLI, signed in to that install, as in [Install the hopbox CLI](/guide/install-cli/index.html.md).

## What the login did

`hopbox login` prepared `~/.ssh` for the SSH front door. If your account had no SSH key, the login made one and added it:

```text
Signed in to https://example.com as user1@example.org. The profile default is active.
Added the SSH key ~/.ssh/hopbox_ed25519 (SHA256:rkEGrocWsFODhePtstCZFcyj1D0BUmGYu1rexwam/mo) to your account.
```

An account that has a key keeps its keys as they are. Then add your own key with `hopbox ssh-key add ~/.ssh/id_ed25519.pub`.

## Make a machine

```sh
hopbox new box1
```

The CLI makes the machine, and waits until it runs. Then it prints the machine:

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

The first machine of an image on a node takes about 25 s, because the node unpacks the image first. To get the new machine at once, give `--wait 0`. If the machine does not run within 5 minutes, the command fails with status 6. The machine stays in both cases.

A name that another machine holds fails with status 5. The install then names a free name:

```text
hopbox: already_exists: already exists: the machine name "box1" is taken. A free name is "box2"
```

List the machines of the org:

```sh
hopbox list
```

```text
NAME  STATE    SIZE     SHARED  URL                       SSH
box1  running  default  no      https://box1.example.com  box1.hop
```

## Look at the SSH alias

The CLI added the machine to a managed block at the top of `~/.ssh/config`. Your own lines stay below it, as they were:

```sh
sed -n '/# BEGIN hopbox/,/# END hopbox/p' ~/.ssh/config
```

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

With this entry, `ssh box1.hop`, `scp`, `rsync`, and VS Code Remote-SSH reach the machine through the SSH front door ([Reach a machine over SSH](/guide/ssh/index.html.md)). The login also puts the host key of the install into `~/.ssh/known_hosts`, so the first connection asks no question. An install without a key-encryption key has no host key, and `known_hosts` then gets no line.

A rename moves the entry, and a delete removes it. `hopbox list` also adds the machines that the console made, and removes the machines that are gone. Give `--no-ssh-config` to a command that must leave `~/.ssh` alone, such as a command of a CI job.

## Run commands

```sh
hopbox exec box1 uname -sr
hopbox exec box1 -e GREETING=hello -- sh -c 'echo $GREETING from $(hostname)'
```

```text
Linux 6.1.167
hello from box1
```

Each word after the machine belongs to the command, flags too. The exit code of the command is the exit status of the CLI:

```sh
hopbox exec box1 sh -c 'echo out; echo err >&2; exit 3'; echo "status $?"
```

```text
out
err
status 3
```

For a script, `--json` keeps stdout and stderr apart in one object:

```sh
hopbox exec --json box1 sh -c 'echo out; echo err >&2; exit 3'
```

```json
{"exit_code":3,"signal":"","error":"","timed_out":false,"stdout":"out\n","stderr":"err\n","stdout_truncated":false,"stderr_truncated":false}
```

The input of the CLI goes to the command:

```sh
printf 'b\na\n' | hopbox exec box1 sort
```

```text
a
b
```

A Ctrl-C goes to the command as SIGINT. A second Ctrl-C ends the exec, and the machine then ends the command. `--timeout 30s` ends a command after 30 seconds, with the exit code 124.

## Copy files

Copy a file into the machine, and read it back:

```sh
echo 'hello' > notes.txt
hopbox cp notes.txt box1:
hopbox cp box1:notes.txt -
```

```text
hello
```

A target that ends with `/`, or an empty one, takes the name of the source. Copy a directory with `-r`. Its contents go into the target directory:

```sh
mkdir -p site && echo '<h1>hi</h1>' > site/index.html
hopbox cp -r site box1:site
hopbox cp -r box1:site ./site-copy
ls site-copy
```

```text
index.html
```

A copy to the laptop has at most 4 MiB. A larger one fails with status 1, and leaves no file behind.

## Serve the URL

Run a web server as the start command, on the port of the URL:

```sh
hopbox machine update box1 --start-command 'python3 -m http.server 3000'
hopbox url port box1 3000
hopbox machine reboot box1
```

```text
https://box1.example.com (private, port 3000)
Rebooted the machine box1 in 4.812s.
```

The start command writes its output to the log of the machine. Read it, and keep it open with `-f` until Ctrl-C:

```sh
hopbox logs -f box1
```

```text
Serving HTTP on 0.0.0.0 port 3000 (http://0.0.0.0:3000/) ...
```

The URL is private: only signed-in accounts with access to the machine open it. The edge servers of the install serve it ([Run the edge role and open machine URLs](/guide/machine-urls/index.html.md#open-the-url-of-a-machine)). Make it public:

```sh
hopbox url public box1
```

```text
https://box1.example.com (public, port 3000)
```

## Open a shell

```sh
hopbox connect box1
```

`connect` opens the login shell of `user` through the API on port 443. It needs no SSH key. The terminal is in raw mode until the shell ends, so Ctrl-C and Ctrl-Z go to the shell. A resize of the window reaches the shell too. Type `exit` to leave. The CLI then exits with the status of the shell.

## Set env vars

A personal org has no shared machines, so its values are `private` or `all`:

```sh
hopbox env set GREETING=hello --scope private
printf 's3cret' | hopbox env set API_TOKEN --secret --scope all
hopbox env list
```

```text
Set the env var GREETING in the scope private.
Set the secret API_TOKEN in the scope all.
NAME       SCOPE    SECRET  VALUE     UPDATED
API_TOKEN  all      yes     (secret)  2026-10-21T11:02:40Z
GREETING   private  no      hello     2026-10-21T11:02:39Z
```

The value of a secret comes from stdin, so it stays out of the shell history. The machine gets the values at its next boot and in each new login shell.

## Stop and start the machine

A stop shuts the guest down. The disk of the machine stays, with your files:

```sh
hopbox machine stop box1
hopbox list
hopbox machine start box1
hopbox cp box1:notes.txt -
```

```text
Stopped the machine box1 in 512ms.
NAME  STATE    SIZE     SHARED  URL                       SSH
box1  stopped  default  no      https://box1.example.com  box1.hop
Started the machine box1 in 1.204s.
hello
```

A stopped machine holds no vCPUs and no RAM of the org. So the start first checks the limits of the org and the room on the node. Each command waits for the machine for up to 60 s. If the machine does not get there in that time, the command fails with status 6, and the machine keeps its target ([hopbox CLI](/reference/cli/index.html.md#machine-start)).

## Delete the machine

```sh
hopbox machine delete box1
```

```text
Delete the machine box1 and its disk? Its name stays reserved for 7 days. [y/N] y
Deleted the machine box1 in 2.104s.
```

A script has no terminal for the question, so it adds `--yes`. The CLI removed the entry of `box1` from `~/.ssh/config`. It was the last machine, so the block and its markers are gone, and the file holds only your own lines again.
