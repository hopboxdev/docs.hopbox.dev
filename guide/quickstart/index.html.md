# Try hopbox

> How to try hopbox on the preview install: sign in, create a machine, open a shell, and open its URL. Also the words of these docs.

This guide makes your first machine on the preview install of hopbox. Its address is https://dev.hopbox.dev. You sign in, create a machine, open a shell in it, and open its URL. You need only a browser, and a GitHub account or an email address. The steps take about five minutes.

The preview install is for tests. Its data can go at any time before version 1.0, so keep no work there that you cannot lose.

## Sign in

1. Open https://dev.hopbox.dev in a browser.
2. Click "Sign in with GitHub", and approve the request on GitHub. Or type your email in "Email", and click "Email me a sign-in link". Open the link in the email. It works once, for 15 minutes.

The first sign-in creates your account and its personal org, "Personal". An org holds machines, and a personal org has only you as its member. In the sign-up mode `approval`, the default of an install, the new account waits for approval by an instance admin ([Sessions, cookies, and sign-in links](/reference/sessions/index.html.md#pending-accounts)). Until then, the console shows only the page of a pending account. If the install sends mail, you get an email after the approval. Then the console opens the page "Machines" of your org. Next to the name of the org, the word `live` tells you that the list follows the install as it changes.

The line under the title shows the usage and the limits of the org. By default, an org can hold 3 machines, 6 vCPUs, 24 GiB of RAM, and 300 GB of disk. The operator of the install can change the limits of an org.

## Create a machine

1. Click "New machine".
2. Leave the name empty for a free name, or type a name of your own.
3. Keep the size `default`: 2 vCPUs, 8 GiB of RAM, and a 100 GB disk.
4. Click "Create".

A machine name is unique on the install, because it is also the first part of the machine URL. If another account holds the name that you typed, the dialog says so and offers a free name, such as "Use box2". Click it, then click "Create". This guide calls the machine `box1`. Use the name of your own machine in its place.

The state of the machine goes from `pending` to `starting` to `running`, without a reload of the page. If the node did not run the image before, the create takes about 25 seconds, because the node unpacks the image first.

## Open a shell

1. Click the name of the machine. The page of the machine shows its actions, its URL, its sharing, and its details.
2. Click "Open terminal". The web terminal opens the login shell of the user `user` in the machine.
3. Type `uname -a`, and press Enter. The machine prints its Linux kernel.

The machine runs Ubuntu. `user` has `sudo` without a password. The files that you make stay on the disk of the machine, also after a stop and a start.

## Open the URL of the machine

Each machine gets an HTTPS URL, such as `https://box1.dev.hopbox.dev`. The URL reaches port 8000 in the machine. The default image has nginx on that port, but nginx is off at first. Turn it on in the terminal:

```sh
sudo systemctl enable --now nginx
```

Go back to the page of the machine, and click the URL in the section "URL". The page of nginx opens.

The URL is private at first. Only a signed-in account that can use the machine opens it. Your browser has your session, so the URL opens for you. To let anyone open it without an account, turn on the switch "Public URL". Turn the switch off to make the URL private again.

To serve another port, type it in the field "Port" of the section "URL", and click "Set port".

## Connect over SSH

The SSH front door of the install takes `ssh`, `scp`, `rsync`, and port forwards. It finds your account by your SSH key.

1. Click your name at the top right to open the page "Account".
2. In the section "SSH keys", paste your public key, such as the text of `~/.ssh/id_ed25519.pub`, into "Public key".
3. Click "Add SSH key".
4. On the page of the machine, read the "SSH port" in the section "Details", such as `10001`.

Then connect from a terminal on your computer. Use your own port:

```sh
ssh -p 10001 user@dev.hopbox.dev
```

At the first connection, ssh asks whether you trust the host key of the install, and shows its fingerprint. The API of the install gives the same key in the field `sshHostKey` of `GetInstallInfo`. Print its fingerprint in a second terminal. The command needs `jq`:

```sh
curl -s -X POST -H 'Content-Type: application/json' -d '{}' \
  https://dev.hopbox.dev/hopbox.v1.InstallService/GetInstallInfo | jq -r .sshHostKey | ssh-keygen -lf -
```

If the `SHA256:` fingerprint matches the fingerprint that ssh shows, type `yes`. The hopbox CLI writes the host key and an alias such as `box1.hop` for you ([Install the hopbox CLI](/guide/install-cli/index.html.md)). After `hopbox login` and `hopbox list`, `ssh box1.hop` works without the port. [Reach a machine over SSH](/guide/ssh/index.html.md) shows `scp`, `rsync`, and port forwards.

## Stop, start, and delete the machine

On the page of the machine:

1. Click "Stop". The machine shuts down, and its disk stays. A stopped machine holds no vCPUs and no RAM of the org.
2. Click "Start". The machine boots with the same disk.
3. Click "Delete", then "Delete" in the dialog. The machine and its disk go. Its name and its SSH port stay reserved for your org for 7 days.

"Sign out" at the top right ends the session in this browser.

## What the preview does not offer yet

- The hopbox CLI has no public download yet. [Install the hopbox CLI](/guide/install-cli/index.html.md) gives the steps for the day it does. Until then, use the console.

## Next

- [Install the hopbox CLI](/guide/install-cli/index.html.md): install the hopbox CLI, and sign it in.
- [Use machines from the CLI](/guide/cli-machines/index.html.md): make and use machines from the CLI.
- [Use the console](/guide/console/index.html.md): the other pages of the console, such as orgs, members, API keys, and env vars.
- [Connect an AI agent to hopbox with MCP](/guide/mcp/index.html.md): let an AI agent make and use a machine.

## Words in these docs

- Install: one running copy of hopbox, with its own domain, such as `dev.hopbox.dev`.
- Org: a group of accounts that owns machines and other resources.
- Machine: a persistent Linux computer that hopbox runs as a microVM on a node.
- Node: a server of the install that runs machines.
- Size: the number of vCPUs, the RAM, and the disk of a machine.
- Image: the base disk that a new machine starts from.
- CLI token: the credential of the hopbox CLI. The CLI gets it after you approve its sign-in in the console.
- API key: a credential for scripts, which works in one org.
- Machine URL: the HTTPS URL of a machine, such as `box1.dev.hopbox.dev`.
- SSH front door: the SSH endpoint of the install, which passes your SSH connection to a machine.
