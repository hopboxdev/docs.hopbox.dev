# Sign in with GitHub and Google

> How an operator registers the GitHub OAuth app and the Google OAuth client of an install, gives hopbox their clients, and signs in with GitHub.

People can sign up and sign in to an install with GitHub or Google. hopbox cannot register the OAuth apps for you. Each app has redirect URIs on the domain of one install, and its client secret belongs to you. So you register one app at each provider, and give hopbox the client ID and the secret. This guide does that for GitHub and Google. Then it signs in with GitHub on a laptop install and reads the new account.

The first sign-in with GitHub makes an account and a personal org. This is task 1 of the end of phase 1. [Sign-in with GitHub and Google](/reference/oauth/index.html.md) has every rule of the flow.

## What you need

- `hopboxd` with the control role, with a domain and a key-encryption key, as in [First sign-in on a new install](/guide/first-sign-in/index.html.md). This guide uses the laptop install of that guide, with the domain `example.com` and the API on `127.0.0.1:7080`.
- An account at GitHub, and for Google, a Google Cloud project.
- A web browser and curl.

Define the short name for the admin command, as in [First sign-in on a new install](/guide/first-sign-in/index.html.md):

```sh
admin() { ./bin/darwin_arm64/hopboxd admin --config /tmp/hb/hopboxd.toml "$@"; }
```

## Print the redirect URIs

Each provider sends the browser back to one redirect URI on the domain of the install. Print the URIs to register:

```sh
admin config oauth
```

```text
github	off	-	https://example.com/auth/github/callback
google	off	-	https://example.com/auth/google/callback
```

Register the exact URI, and keep the exact matching of the provider. Only with exact matching does the provider refuse a sign-in with another URI.

## Register the GitHub OAuth app

1. On GitHub, open Settings, then Developer settings, then OAuth Apps, and click New OAuth App. For an app that an organization owns, open the settings of the organization instead.
2. Fill in the form:

   | Field | Value |
   | --- | --- |
   | Application name | `hopbox` |
   | Homepage URL | `https://example.com` |
   | Authorization callback URL | `https://example.com/auth/github/callback` |
   | Enable Device Flow | off |

3. Click Register application. Copy the Client ID.
4. Click Generate a new client secret, and copy the secret. GitHub shows it only once.
5. In the settings of the app, turn off wildcard matching for the callback URL. With wildcard matching, GitHub also takes a redirect URI on a subdomain or in a subdirectory of the callback URL. A machine URL such as `https://box1.example.com` is a subdomain of the install, and it serves the content of a user. So without this step, a link can send the code of a sign-in to a machine. hopbox still sends the exact redirect URI in the token request, and the code needs the state cookie and the PKCE verifier. But the code must not leave the install at all.

Use an OAuth app, not a GitHub App. hopbox asks only for the scope `user:email`, to read the emails and whether GitHub verified them.

## Register the Google OAuth client

1. In the Google Cloud console, select the project, and open Google Auth Platform.
2. On the page Branding, fill in the app name, the user support email, and the developer contact email. Add the domain of the install as an authorized domain, for example `example.com`.
3. On the page Audience, choose the user type External. While the app has the publishing status Testing, only test users can sign in. Add each Google account that must sign in as a test user.
4. On the page Data Access, add the scopes `openid`, `email`, and `profile`. These scopes are not sensitive, so the app needs no verification by Google for them.
5. On the page Clients, click Create client. Choose the application type Web application.
6. Under Authorized redirect URIs, add `https://example.com/auth/google/callback`. Google matches this URI exactly.
7. Click Create. Copy the client ID and the client secret at once. Google shows a new client secret only at its creation. If you lose it, add a new secret to the client.

## Give hopbox the clients

Pass each secret on standard input, so it never shows in the process list or in the shell history:

```sh
read -rs GITHUB_CLIENT_SECRET
printf '%s\n' "$GITHUB_CLIENT_SECRET" | admin config oauth set github \
  --client-id Ov23liExampleClient --client-secret-stdin
```

```text
github	on	Ov23liExampleClient	https://example.com/auth/github/callback
```

```sh
read -rs GOOGLE_CLIENT_SECRET
printf '%s\n' "$GOOGLE_CLIENT_SECRET" | admin config oauth set google \
  --client-id 123-abc.apps.googleusercontent.com --client-secret-stdin
```

```text
google	on	123-abc.apps.googleusercontent.com	https://example.com/auth/google/callback
```

hopbox seals each secret with the current key-encryption key ([Sealing](/reference/sealing/index.html.md)). Without a key, the command fails and asks you to run `admin kek rotate`. On a server, run the same commands as root: `sudo hopboxd admin config oauth set ...`.

The install now offers both providers:

```sh
curl -s -X POST -H 'Content-Type: application/json' -d '{}' \
  http://127.0.0.1:7080/hopbox.v1.InstallService/GetInstallInfo
```

```json
{"domain":"example.com", "signInMethods":["SIGN_IN_METHOD_GITHUB", "SIGN_IN_METHOD_GOOGLE"], "version":"v0.1.0"}
```

## Sign in with GitHub

On an install with an edge and a console, a person clicks Sign in with GitHub, and the browser does every step on its own. The laptop install of this guide runs no edge role, so its API answers only on `127.0.0.1` over plain HTTP. Here you do the two steps of hopbox with curl, and only the step at GitHub in the browser.

Start the sign-in. Save the headers, which hold the URL of GitHub and the state cookie:

```sh
curl -s -o /dev/null -D /tmp/hb/start 'http://127.0.0.1:7080/auth/github/start?return_to=/'
sed -n 's/^Location: //p' /tmp/hb/start
```

```text
https://github.com/login/oauth/authorize?client_id=Ov23liExampleClient&code_challenge=...&code_challenge_method=S256&redirect_uri=https%3A%2F%2Fexample.com%2Fauth%2Fgithub%2Fcallback&response_type=code&scope=user%3Aemail&state=...
```

The domain `example.com` of this guide is a public host that is not your install. Before you open the URL, map the domain to your laptop in `/etc/hosts`:

```sh
echo '127.0.0.1 example.com' | sudo tee -a /etc/hosts
```

Without this line, the browser sends the code and the state of the sign-in to the public host `example.com`. A browser with its own DNS-over-HTTPS resolver, or with `example.com` in its cache, can skip the line. So before the sign-in, open `https://example.com/` in the browser, and make sure that it shows a refused connection, not the public page of `example.com`. If it shows the public page, turn off the secure DNS setting of the browser for this test, restart the browser, and open the page again.

Open that URL in the browser, and click Authorize. GitHub then sends the browser to `https://example.com/auth/github/callback?code=...&state=...`. The name now points at `127.0.0.1`, and the install does not listen on port 443, so the browser shows an error page for a refused connection. Copy the whole URL from the address bar into a shell variable:

```sh
CALLBACK='https://example.com/auth/github/callback?code=...&state=...'
```

The code works once, for a few minutes. Send it to hopbox with the state cookie of the start, as the browser does:

```sh
STATE=$(sed -n 's/^Set-Cookie: \(__Host-hopbox_oauth_github=[^;]*\).*/\1/p' /tmp/hb/start)
curl -s -D /tmp/hb/callback -H "Cookie: $STATE" "http://127.0.0.1:7080${CALLBACK#https://example.com}"
grep -E '^(HTTP/|Location:)' /tmp/hb/callback
```

```text
HTTP/1.1 303 See Other
Location: /
```

The callback compared the state of the URL with the state of the cookie. Then it traded the code at GitHub, read your primary email, and made your account with a personal org. Your email is verified, because GitHub verified it. Without the cookie, the callback fails with `400`, so a link from another site cannot end a sign-in in your browser ([Sign-in with GitHub and Google](/reference/oauth/index.html.md#callback)).

Copy the session cookie, and read your account:

```sh
COOKIE=$(sed -n 's/^Set-Cookie: \(__Host-hopbox_session=[^;]*\).*/\1/p' /tmp/hb/callback)
curl -s -X POST -H 'Content-Type: application/json' -H 'Connect-Protocol-Version: 1' \
  -H 'Origin: https://example.com' -H "Cookie: $COOKIE" -d '{}' \
  http://127.0.0.1:7080/hopbox.v1.AccountService/ListSignInMethods
```

```json
{"methods":[{"id":"01a14f02-6b1e-7d4c-9a51-2b1c5e8f0a7d", "method":"SIGN_IN_METHOD_GITHUB", "email":"ann@example.org", "lastUseTime":"2026-10-21T09:12:04.118Z", "createTime":"2026-10-21T09:12:04.118Z"}]}
```

Your login-link account from [First sign-in on a new install](/guide/first-sign-in/index.html.md) has an unverified email, so the GitHub sign-in made a new account. To make it an instance admin too, grant the role by its verified email:

```sh
admin instance-admin grant --email ann@example.org
```

A sign-in with Google works in the same way, with `/auth/google/start` and the cookie `__Host-hopbox_oauth_google`.

## Link Google to your account

Your account now has an identity of a provider: the GitHub user. So a verified email alone no longer adds a sign-in method to it ([Sessions, cookies, and sign-in links](/reference/sessions/index.html.md#what-a-sign-in-does)). A Google sign-in with `ann@example.org` now fails with `409`. The rule protects the account if the domain of the email lapses and a new owner proves the address. To add Google, link it from your signed-in session. In the console, this is the button Link Google on `/account`.

The link needs a sign-in within the last 10 minutes ([Sessions, cookies, and sign-in links](/reference/sessions/index.html.md#recent-sign-in)). If your GitHub sign-in is older, run the steps of [Sign in with GitHub](#sign-in-with-github) again first.

Start the link with the session cookie and `link=1`:

```sh
curl -s -o /dev/null -D /tmp/hb/link -H "Cookie: $COOKIE" \
  'http://127.0.0.1:7080/auth/google/start?link=1&return_to=%2Faccount'
sed -n 's/^Location: //p' /tmp/hb/link
```

```text
https://accounts.google.com/o/oauth2/v2/auth?client_id=...&code_challenge=...&code_challenge_method=S256&nonce=...&redirect_uri=https%3A%2F%2Fexample.com%2Fauth%2Fgoogle%2Fcallback&response_type=code&scope=openid+email+profile&state=...
```

With an older sign-in, the `Location` is `/signin?again=1&return_to=%2Faccount` instead.

Open the URL in the browser, and choose your Google account. As before, copy the URL of the refused callback. Send it with the state cookie of the link and the session cookie:

```sh
CALLBACK='https://example.com/auth/google/callback?state=...&code=...'
STATE=$(sed -n 's/^Set-Cookie: \(__Host-hopbox_oauth_google=[^;]*\).*/\1/p' /tmp/hb/link)
curl -s -D /tmp/hb/linked -H "Cookie: $STATE; $COOKIE" "http://127.0.0.1:7080${CALLBACK#https://example.com}"
grep -E '^(HTTP/|Location:|Set-Cookie:)' /tmp/hb/linked
```

```text
HTTP/1.1 303 See Other
Location: /account
Set-Cookie: __Host-hopbox_oauth_google=; Path=/; Max-Age=0; HttpOnly; Secure; SameSite=Lax
```

The callback added the Google account to your account. It set no session cookie, so your session stays. `ListSignInMethods` now lists GitHub and Google, and both sign in to the same account. The link works only with the session that started it. Another browser gets `400` at the callback ([Sign-in with GitHub and Google](/reference/oauth/index.html.md#link-to-a-signed-in-account)).

After the sign-ins, remove the line `127.0.0.1 example.com` from `/etc/hosts`. This command works on macOS and Linux, and keeps a copy of the old file in `/etc/hosts.bak`:

```sh
sudo sed -i.bak '/^127\.0\.0\.1 example\.com$/d' /etc/hosts
```

## Try a server before it has an edge

A server without the edge role answers only on `127.0.0.1:7080` ([the edge role](/reference/edge/index.html.md)). Until the server runs the edge role, forward the port, and run the same steps on your laptop against the forward:

```sh
ssh -N -L 7080:127.0.0.1:7080 debian@203.0.113.10
```

Through the tunnel, the commands and the `Origin` header use the domain of the server install.

When the edge role serves the install over HTTPS, open `https://example.com/auth/github/start` in the browser, and the browser does every step. The edge then passes the address of each client, so the rate limits of a client apply ([Sign-in with GitHub and Google](/reference/oauth/index.html.md#rate-limits)).

## When a sign-in fails

The failed step answers with a plain text page that says why, and a request ID for the server log ([Sign-in with GitHub and Google](/reference/oauth/index.html.md#errors)).

| Message | What to do |
| --- | --- |
| `the sign-in is not valid: it did not start in this browser, it was used, or it expired` | Start the sign-in again. A flow lasts 10 minutes, and works once. |
| `the provider has not verified an email of this account` | At GitHub, verify an email and make it the primary email. At Google, use an account with a verified email. |
| `the provider did not approve the sign-in: GitHub refused the sign-in code` | Compare the client secret and the redirect URI with the app at GitHub. The log record `oauth sign-in refused by the provider` has the answer of GitHub. |
| `failed precondition: this install has no GitHub client` | Run `admin config oauth set github`. |
| `the sign-in conflicts with a sign-in method of the account` | The account of the email has GitHub or Google, so a new method needs a link. Sign in with a method of the account, then link the new one on `/account` ([Link Google to your account](#link-google-to-your-account)). If the person lost every method, run `admin account sign-in-method remove` ([Sign-in with GitHub and Google](/reference/oauth/index.html.md#recover-a-lost-identity)). |

To turn a provider off, clear its client. The accounts keep their sign-in methods, which work again after a new set:

```sh
admin config oauth clear google
```
