# Use the console

> How to use the console of an install in a browser: sign in, create a machine, open a shell, approve the hopbox CLI, and the other pages.

The console is the web app of an install. You sign in, create and stop machines, make a URL public, and open a shell in a machine, all in a browser ([Console](/reference/console/index.html.md)). `hopboxd` serves it at the domain of the install, next to the API. This guide uses the install `https://example.com`. For the preview install at https://dev.hopbox.dev, [Try hopbox](/guide/quickstart/index.html.md) gives the same first steps.

## What you need

- A browser.
- An account on an install, or a sign-in method of the install that makes one, such as GitHub.

## Sign in

Open `https://example.com/` in the browser. Without a session, the console sends you to `/signin`. The page shows the sign-in methods of the install:

- GitHub and Google: the buttons "Sign in with GitHub" and "Sign in with Google". The operator sets them up as in [Sign in with GitHub and Google](/guide/github-google-sign-in/index.html.md).
- Magic links: a field "Email" and the button "Email me a sign-in link". The link works once, for 15 minutes.
- No method yet: a new install shows the command that makes a first-admin login link. The operator runs it on a control server, as in [First sign-in on a new install](/guide/first-sign-in/index.html.md).

The first sign-in creates your account and its personal org. In the sign-up mode `approval`, the default, the new account waits for approval by an instance admin ([Sessions, cookies, and sign-in links](/reference/sessions/index.html.md#pending-accounts)). Until then, the console shows only the page of a pending account. After the approval, or at once in the mode `open`, the console opens the machines of your org. The head of the org shows `live`: the list follows the install as it changes.

## Create a machine and open a shell

1. On the page of the org, click "New machine". Leave the name empty for a free name, such as `box1`, or type one. Click "Create".
2. The machine shows `pending`, then `starting`, then `running`, without a reload of the page.
3. Click the name of the machine. Its page has the actions, the URL, and the sharing.
4. Turn on the switch "Public URL". Anyone can now open the URL without an account. Turn it off to make the URL private again. Only the creator of a machine can change the switch.
5. Click "Open terminal". The terminal opens the login shell of the machine. Type `uname -a`, then `exit`.
6. Back on the list, click "Stop", then "Start", then "Delete". The delete asks first.

A machine name is also the first part of its URL, so each name is unique on the install. If another account holds the name, the dialog offers a free name, such as "Use box2".

## Approve the hopbox CLI

The hopbox CLI signs in with a device flow ([Install the hopbox CLI](/guide/install-cli/index.html.md)). Run it on your laptop:

```sh
hopbox login --url https://example.com
```

The CLI prints a link such as `https://example.com/device?code=WDJB-MJHT`. Open it in the browser where you signed in. The page shows the name and the IP address of the client that asks. Check that the code matches the code in your terminal. Approve only a sign-in that you started yourself. Click "Approve", and the CLI gets its tokens.

The approval needs a sign-in within the last 10 minutes ([Sessions, cookies, and sign-in links](/reference/sessions/index.html.md#recent-sign-in)). If you signed in earlier, the page says so and shows "Sign in again". Click it, and sign in. The console then opens the page of the code again. Click "Approve" again.

## Link another sign-in method

Your account can hold several sign-in methods. Once it has GitHub or Google, a new method joins it only through a link from your session ([Sessions, cookies, and sign-in links](/reference/sessions/index.html.md#link-a-new-sign-in-method)). A sign-in with a new method fails then, also with the same email. A link needs a sign-in within the last 10 minutes.

1. Open "Account" at the top right.
2. Under "Sign-in methods", click "Link GitHub" or "Link Google". The console shows a button only for a provider of the install that the account does not hold yet. Approve at the provider. The console opens "Account" again, with the new method in the list.
3. To link an email, such as `ann.work@example.org`, type it in the field "Email" and click "Email me a link". Open the link in the same browser. The console adds the email and opens "Account" again.

If your sign-in is older than 10 minutes, "Link GitHub" and "Link Google" send you to sign in again first, then back to "Account". "Email me a link" shows a notice and the button "Sign in again" instead, which also brings you back to "Account". After the new sign-in, start the link again.

If your account is an instance admin, each new method sends a security notice to every instance admin. Each email of your account gets a copy, also the new one, unless another account holds it as verified ([Use the Admin area](/guide/admin-area/index.html.md#read-the-security-notices)).

## More pages

| Page | Where | What you do there |
| --- | --- | --- |
| Account | the name at the top right | Change the display name. See the emails and the sign-in methods. Link and remove sign-in methods. Add and remove SSH keys. End a CLI sign-in. |
| Orgs | "All orgs" in the org list at the top | See your orgs, and create one. |
| Members | the tab of an org | Change roles. Invite people by email, and copy the link of an invitation. |
| API keys | the tab of an org | Create a key for a script. Copy the key at once: the console shows it only once. |
| Env vars | the tab of an org | Set env vars and secrets for the machines of the org. |
| Settings | the tab of an org | Rename, leave, or delete the org, and see its limits. |
| Admin area | "Admin" at the top right, only for an instance admin | Operate the whole install: servers, pools, sizes, images, orgs, machines, instance admins, admin keys, the configuration of the install, and the audit log ([Use the Admin area](/guide/admin-area/index.html.md)). |

"Sign out" at the top right ends the session.
