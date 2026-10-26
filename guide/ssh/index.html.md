# Reach a machine over SSH

> How to reach a machine through the SSH front door with ssh, scp, sftp, rsync, and port forwards. Also what each failure means.

The SSH front door lets `ssh`, `scp`, `sftp`, `rsync`, port forwards, and VS Code Remote-SSH reach a machine. Each machine has its own SSH port on the edge servers. Your SSH key proves your account. The user name is the Linux user in the machine, `user` by default. [The SSH front door](/reference/ssh-front-door/index.html.md) covers the front door in full.

## How a connection reaches the machine

1. Your SSH client connects to the domain of the install, at the SSH port of the machine, such as `example.com` port 10022.
2. nftables on the edge server sends the connection to the SSH front door. The port names the machine.
3. The front door shows the one host key of the install. The hopbox CLI wrote it to `~/.ssh/known_hosts` at your login, so ssh asks no question.
4. Your client signs in with your SSH key. The control role checks that your account uses the machine, as for `hopbox exec`.
5. The front door opens its own SSH connection to the OpenSSH server in the machine, and passes your channels through it.

An org admin cannot open a private machine of another member, as with `hopbox exec`. A shared machine opens for every member of its org.

## What you need

- An account on an install with machines, such as the preview install of [Try hopbox](/guide/quickstart/index.html.md). This guide uses the install `https://example.com`.
- The hopbox CLI, installed as in [Install the hopbox CLI](/guide/install-cli/index.html.md). Without the CLI, add your SSH key on the page "Account" of the console, and give the SSH port of the machine to ssh ([Try hopbox](/guide/quickstart/index.html.md#connect-over-ssh)).
- OpenSSH on your laptop. macOS and most Linux systems have it.

## Sign in and make a machine

```sh
hopbox login --url https://example.com
hopbox new box1
```

If your account had no SSH key, the login made `~/.ssh/hopbox_ed25519` and added it to your account. It also wrote the host key of the install to `~/.ssh/known_hosts`. `hopbox new` added the entry `box1.hop` to the managed block of `~/.ssh/config` ([Use machines from the CLI](/guide/cli-machines/index.html.md)):

```text
Host box1.hop
  HostName example.com
  Port 10022
  User user
  IdentityFile ~/.ssh/hopbox_ed25519
  IdentitiesOnly yes
  HostKeyAlias hopbox-example.com
```

## Open a shell, and copy files

```sh
ssh box1.hop
ssh box1.hop 'uname -sr; id -un'
scp ./notes.txt box1.hop:/tmp/
sftp box1.hop
rsync -a ./site/ box1.hop:site/
```

The second command prints the kernel of the machine and `user`. The exit status of a remote command is the exit status of `ssh`. For root, use `ssh root@box1.hop`. The main user also has `sudo` without a password.

## Forward ports

```sh
ssh -N -L 8000:127.0.0.1:8000 box1.hop
ssh -N -R 9000:127.0.0.1:3000 box1.hop
ssh -A box1.hop
```

The first command makes port 8000 of the machine reach `http://127.0.0.1:8000` on your laptop. The second one makes port 9000 in the machine reach port 3000 on your laptop. The third one forwards your SSH agent, so `git` in the machine can use your keys. X11 forwarding stays off.

In VS Code, install the Remote-SSH extension, and connect to the host `box1.hop`. VS Code reads the managed block, and uses a local port forward.

## Use another key

```sh
hopbox ssh-key add ~/.ssh/id_ed25519.pub
ssh -i ~/.ssh/id_ed25519 -p 10022 -o HostKeyAlias=hopbox-example.com user@example.com
```

The managed block sets `IdentitiesOnly yes`, so `ssh box1.hop` offers only `~/.ssh/hopbox_ed25519`. Give another key with `-i`, as above. A key works for one account only. A removal with `hopbox ssh-key rm` ends the open SSH connections of the key within seconds.

## See who connected

Each SSH sign-in writes the audit entry `machine.ssh` in the org of the machine. The entry names the account, the key fingerprint, the machine, the Linux user, and the IP address of the client. A refused sign-in writes no entry. The org owners and the org admins read the entries. An org member reads none.

`hopbox new box1` made `box1` in your active org. If you are an owner or an admin of that org, make a member key in it, and read the newest entries:

```sh
KEY=$(hopbox api-key create audit --expires=30d)
curl -s -X POST -H 'Content-Type: application/json' -H "Authorization: Bearer $KEY" \
  -d '{"pageSize":50}' https://example.com/hopbox.v1.AuditService/ListAuditEntries |
  jq -c '.entries[] | select(.action == "machine.ssh") | [.createTime, .details.name, .details.linux_user, .details.client_ip, .details.fingerprint]'
```

```text
["2026-10-25T15:19:59.295802Z","box1","user","192.0.2.10","SHA256:FXuRc2JyZfKx3tXHxoblBpr+f1jfrmFaGfkF4S5o0Ho"]
```

`hopbox ssh-key list` shows the fingerprints of your own keys. The checks of an open connection write no entry, so one connection gives one line. [The SSH front door](/reference/ssh-front-door/index.html.md#audit) lists every field.

## When the connection fails

| What ssh prints | What to do |
| --- | --- |
| `Permission denied (publickey)` | Your key opens no machine at this port for this user. Check `hopbox ssh-key list`, the user name, and that you use the machine. The front door never says which part failed. |
| `channel 0: open failed: connect failed: hopbox: the machine box1 cannot take SSH connections now: it is stopped. Start it with: hopbox machine start box1` | Start the machine, then retry. |
| `... it has no network address on its node. Use hopbox connect instead` | The machine has no address. Run `hopbox connect box1` for a shell. |
| `... its guest agent has not reported the SSH host keys of the machine yet. Retry in a few seconds` | The machine just booted. Retry. |
| `Connection closed` right after a connection | The client held 64 connections, or the edge server has no keys of the front door yet. |
| `WARNING: REMOTE HOST IDENTIFICATION HAS CHANGED!` | The line `hopbox-<domain>` of `~/.ssh/known_hosts` holds another key. Run `hopbox login` again, which writes the key of `GetInstallInfo`. |

## Check the front door on an edge server

This section is for the operator of an install. On an edge server, the redirect and the listener show like this:

```sh
sudo nft list table inet hopbox_edge
curl -s http://127.0.0.1:9464/metrics | grep '^hopbox_edge_ssh_'
```

`hopbox_edge_ssh_connections_total{result="forwarded"}` counts the connections that reached their machine. The other results and the logs are in [the SSH front door](/reference/ssh-front-door/index.html.md#metrics).
