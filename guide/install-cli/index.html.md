# Install the hopbox CLI

> How to download the hopbox CLI, check it, put it on the PATH, and sign it in to an install.

The hopbox CLI makes and uses machines from a terminal. This guide downloads the CLI, checks the download, puts it on your `PATH`, and signs it in to an install. [Use the hopbox CLI](/guide/cli/index.html.md) then shows each kind of command.

The download link of the CLI is not public yet, so `<cli-download-url>` below stands for it. Until the link is public, use the console of your install ([Try hopbox](/guide/quickstart/index.html.md)).

## What you need

- A Mac with Apple Silicon, or a Linux computer on x86_64 or arm64. On Windows, use the Linux build in WSL.
- An account on an install, and a browser where you are signed in to its console. The preview install is at https://dev.hopbox.dev. This guide uses the install `https://example.com`.

## Download the CLI

Each release has one file of the CLI for each platform, and the file `SHA256SUMS` with the sha256 of each file:

| Platform | File |
| --- | --- |
| macOS on Apple Silicon | `hopbox-cli-v0.1.0-darwin-arm64.tar.gz` |
| Linux on x86_64 | `hopbox-cli-v0.1.0-linux-amd64.tar.gz` |
| Linux on arm64 | `hopbox-cli-v0.1.0-linux-arm64.tar.gz` |

Download the file of your platform and the sums. On a Mac:

```sh
curl -fLO <cli-download-url>/hopbox-cli-v0.1.0-darwin-arm64.tar.gz
curl -fLO <cli-download-url>/SHA256SUMS
```

## Check the download

Compare the sha256 of the file with its line in `SHA256SUMS`:

```sh
grep hopbox-cli-v0.1.0-darwin-arm64.tar.gz SHA256SUMS | shasum -a 256 -c
```

```text
hopbox-cli-v0.1.0-darwin-arm64.tar.gz: OK
```

On Linux, use `sha256sum -c` in place of `shasum -a 256 -c`. If the command prints `FAILED`, delete the file and download it again. Do not run a file that fails the check.

## Put the CLI on your PATH

The file holds one program, `hopbox`. Unpack it, and copy it to a dir of your `PATH`:

```sh
tar -xzf hopbox-cli-v0.1.0-darwin-arm64.tar.gz
sudo install -m 0755 hopbox /usr/local/bin/hopbox
hopbox version
```

```text
hopbox v0.1.0 (commit 1a2b3c4d5e6f7a8b9c0d1e2f3a4b5c6d7e8f9a0b, built 2026-10-16T09:00:00Z, go1.27.1 darwin/arm64)
```

Without `sudo`, copy the program to a dir of your own, such as `~/.local/bin`, and add that dir to your `PATH`.

## Sign in to an install

Give the CLI the URL of the install. A bare domain gets `https://`:

```sh
hopbox login --url https://example.com
```

The CLI prints a code and a link, then waits:

```text
To sign in, open this page in a browser and approve the code KWJQ-LBZV:

  https://example.com/device?code=KWJQ-LBZV

Waiting for the approval. Press Ctrl-C to stop.
```

1. Open the link in the browser where you are signed in to the console.
2. Check that the code on the page matches the code in your terminal.
3. Check the client: it is `hopbox CLI on` and the name of your computer. Approve only a sign-in that you started yourself.
4. Click "Approve".

The approval needs a sign-in to the console within the last 10 minutes. After an older sign-in, the page says so after your click on "Approve", and shows "Sign in again". Click it, and sign in. The console then shows the same code. Click "Approve" again.

The CLI sees the approval within a few seconds:

```text
Signed in to https://example.com as user1@example.org. The profile default is active.
Added the SSH key ~/.ssh/hopbox_ed25519 (SHA256:rkEGrocWsFODhePtstCZFcyj1D0BUmGYu1rexwam/mo) to your account.
```

The code works for 10 minutes. After that time, run `hopbox login` again.

The login also prepares `~/.ssh` for the SSH front door. If your account had no SSH key, the CLI made the key `~/.ssh/hopbox_ed25519` and added it to your account. It also wrote the host key of the install to `~/.ssh/known_hosts`. `--no-ssh-config` leaves `~/.ssh` alone ([hopbox CLI](/reference/cli/index.html.md#ssh-files)).

Check who you are:

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

## Next

- [Use machines from the CLI](/guide/cli-machines/index.html.md): make a machine, run commands in it, copy files, and open a shell.
- [Use the hopbox CLI](/guide/cli/index.html.md): work in an org, make API keys and SSH keys, and script the CLI.
- [Reach a machine over SSH](/guide/ssh/index.html.md): reach a machine with ssh, scp, rsync, and port forwards.
