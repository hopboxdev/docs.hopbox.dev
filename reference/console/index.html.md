# Console

> The console: its pages and their calls, its headers, its build, and its tests.

The console is the web app of an install. People use it in a browser to sign in, to manage their machines, and to open a shell in a machine. It is a single-page app in React and TypeScript, and `hopboxd` holds its build. This page gives its pages, its calls, its headers, its build, and its tests. The code is in `web/apps/console` and `internal/console`. [Use the console](/guide/console/index.html.md) shows how to use it.

The first console holds the parts of phase 1:

- sign-in
- the list of machines, with the live state of each one
- the create, stop, start, and delete actions
- the switch that makes a URL public
- the web terminal

It also has the pages that the links of the API open, the account page, and the pages of an org. The Admin area for instance admins comes in phase 2.

## Where hopboxd serves it

The API listener of the control role serves the console at the domain of the install, next to the API ([the API server](/reference/api/index.html.md#listener)). Each public handler of the control role holds the console, so the console also answers behind the edge. So the console calls the API from its own origin, and the browser needs no CORS rules.

The console takes every path that no API route takes:

| Request | Who answers |
| --- | --- |
| `POST /hopbox.v1.<Service>/<Method>` and the other API calls | the API |
| `GET /auth/github/start`, `/auth/github/callback`, `/auth/google/start`, `/auth/google/callback` | the sign-in with GitHub and Google ([Sign-in with GitHub and Google](/reference/oauth/index.html.md#endpoints)) |
| `GET /api/ws/exec` | the WebSocket endpoint of exec ([the machine API](/reference/machine-api/index.html.md)) |
| any other `GET` or `HEAD` request whose first path segment has no dot | the console |
| any other request, such as `GET /hopbox.v1.NoSuchService/Get` or `GET /index.html` | the API, with `404 Not Found`, so a Connect or gRPC client reads an unknown procedure |

The console needs HTTPS at `https://<domain>`. The session cookie has the `__Host-` prefix and the `Secure` flag, and each call with the cookie must carry the `Origin` header `https://<domain>` ([Sessions, cookies, and sign-in links](/reference/sessions/index.html.md#calls-with-the-cookie)). Thus the console works only behind a front that ends TLS for the domain of the install, such as the edge. Over plain HTTP, the pages load, but no sign-in works.

## Pages

A page that needs a session sends a browser without one to `/signin?return_to=<path>`, and the sign-in then returns to the page.

| Path | Session | What it does | Calls |
| --- | --- | --- | --- |
| `/signin?return_to=<path>` | no | Shows the sign-in methods of the install. A browser with a live session goes on to `return_to`, or `/`. With `again=1`, the page stays for a live session too, with the title "Sign in again" ([Sign-in](#sign-in)). | `GetInstallInfo`, `GetAccount`, `RequestMagicLink` |
| `/auth/email#token=<token>` | no | The page of a magic link ([Sessions, cookies, and sign-in links](/reference/sessions/index.html.md#magic-links)). It signs in with the token, then opens the saved return path, or `/`. | `VerifyMagicLink` |
| `/auth/login-link#token=<token>` | no | The page of a first-admin login link ([Sessions, cookies, and sign-in links](/reference/sessions/index.html.md#first-admin-login-links)). It works as `/auth/email`. | `VerifyLoginLink` |
| `/invite#token=<token>` | asks for one | The page of an invitation to an org ([the org API](/reference/org-api/index.html.md#invitations)). Without a session, it shows the sign-in methods. With one, it accepts the invitation and opens the org. | `GetAccount`, `AcceptInvitation` |
| `/device?code=<user code>` | yes | Approves a sign-in of the hopbox CLI ([CLI tokens, API keys, admin API keys, and SSH keys](/reference/credentials/index.html.md)). It shows the name, the IP address, and the start of the client. It asks the person to compare the code with the code in the terminal (RFC 8628 section 5.4). It approves exactly that client. Without `code`, it asks for the code. The approval needs a sign-in within the last 10 minutes. After an older sign-in, the page says so, and its button "Sign in again" opens `/signin?again=1&return_to=/device?code=<code>`. | `GetDeviceAuthorization`, `ApproveDevice` |
| `/` | yes | Opens the last org of this browser, or the personal org. | `GetAccount`, `ListOrgs` |
| `/account` | yes | The display name, the emails, the sign-in methods, the SSH keys, and the CLI sign-ins of the account. | `GetAccount`, `UpdateAccount`, `ListSignInMethods`, `RemoveSignInMethod`, `ListSSHKeys`, `AddSSHKey`, `RemoveSSHKey`, `ListCLISignIns`, `EndCLISignIn` |
| `/orgs` | yes | The orgs of the account, and a form that creates an org. | `ListOrgs`, `CreateOrg` |
| `/orgs/<org ID>` | yes | The machines of the org with their live states, the usage of the org, and the create, start, stop, and delete actions. | `WatchMachines`, `GetOrgUsage`, `ListMachineSizes`, `CreateMachine`, `StartMachine`, `StopMachine`, `DeleteMachine` |
| `/orgs/<org ID>/machines/<name>` | yes | One machine: its actions, its URL with the public switch and the port, its sharing, and its details. | `WatchMachines`, `StartMachine`, `StopMachine`, `RebootMachine`, `DeleteMachine`, `SetMachineURLVisibility`, `SetMachineURLPort`, `ShareMachine`, `UnshareMachine` |
| `/orgs/<org ID>/machines/<name>/terminal` | yes | The web terminal ([Web terminal](#web-terminal)). | `/api/ws/exec` |
| `/orgs/<org ID>/members` | yes | The members and their roles. Owners and admins also see the invitations, and invite people. | `ListOrgMembers`, `SetOrgMemberRole`, `RemoveOrgMember`, `ListInvitations`, `CreateInvitation`, `RevokeInvitation` |
| `/orgs/<org ID>/api-keys` | yes | The API keys of the org. A new key shows its secret once. | `ListAPIKeys`, `CreateAPIKey`, `RevokeAPIKey` |
| `/orgs/<org ID>/vars` | yes | The env vars and secrets of the org. A secret shows no value. | `ListVars`, `SetVar`, `DeleteVar` |
| `/orgs/<org ID>/settings` | yes | The name, the usage and the limits of the org, and the leave and delete actions. | `GetOrg`, `UpdateOrg`, `GetOrgUsage`, `LeaveOrg`, `DeleteOrg` |

Every page of an org also calls `GetOrg`. Each page shows the actions that the role of the account allows. The service layer still decides each call ([the API server](/reference/api/index.html.md)), and the page shows its error.

A form, or the question before a delete or another action that cannot be undone, opens in a modal dialog. The dialog keeps the focus inside it. The title of the dialog is its accessible name. The Cancel button, the Escape key, and a click on the dark backdrop close it. A click inside the dialog, or a drag that starts inside it, does not close it.

A path with no page shows "Page not found". A path with a file extension, or under `/assets/`, that names no file of the build gets `404` with the text `the console has no file <path>`.

## Look

The console has the look of the public site [hopbox.dev](https://hopbox.dev). It uses the same colors, fonts, buttons, cards, and icon:

| Part | Value |
| --- | --- |
| Background | `#0A0E0B`, with panels in `#111712` |
| Text | `#ECF2EC`, and `#A3B1A5` for secondary text |
| Accent | lime `#C4F05A`, for links, the main button, the focus ring, and the state `running` |
| Warning | amber `#FFB443`, for a public URL, a state that changes, and the `not verified` tag of an email |
| Error | `#FF7A6B`, for errors and the delete buttons |
| Text font | Bricolage Grotesque |
| Code font | JetBrains Mono, for names, URLs, and the web terminal |

The console has a dark theme only, as the site has. It does not follow the light or dark setting of the system.

The look keeps the console accessible:

- Text has a contrast of at least 4.5:1 on its background (WCAG 2.2 level AA). A disabled control is the only exception.
- The control that has the keyboard focus shows a lime ring.
- Each state has a word next to its color, such as `running` or `Public URL: public`.
- A link inside a sentence has an underline, so it differs from the text by more than its color.

The web terminal takes its colors from the same values. If a program picks a text color with a contrast below 4.5:1, xterm.js makes it lighter.

## Browsers

The console works in these browsers:

| Browser | Version |
| --- | --- |
| Chrome and Edge | 111 or later |
| Firefox | 114 or later |
| Safari | 16.4 or later |

Vite builds the code for this set, its default build target of Baseline Widely Available browsers.

The terminal page fills the height of the window with the CSS selector `:has()`. Firefox supports it from version 121. In Firefox 114 to 120, the terminal and its bar are 360 pixels high together, and do not fill the window. Rows of a table with actions also do not light up on hover there, because that rule uses `:has()` too. The other parts of the console work the same.

On a short window, such as a phone in landscape, the terminal gets smaller, until it and its bar are 360 pixels high together. Then the page scrolls. The head and the tabs of the org keep their height.

## Sign-in

The sign-in page asks `GetInstallInfo` for the sign-in methods:

- For GitHub and Google, it shows a link to `/auth/<provider>/start?return_to=<path>`. The browser opens that link, and the API sends it on to the provider ([Sign-in with GitHub and Google](/reference/oauth/index.html.md#start)). The console encodes the path once with `encodeURIComponent`, and never decodes it ([Sign-in with GitHub and Google](/reference/oauth/index.html.md#return-path)).
- For magic links, it shows an email form, which calls `RequestMagicLink`. After the call, it says that the link went to the email. With `mail_to_files` in `GetInstallInfo`, it says instead that the install writes each email to a file on its server. An operator then reads the link with `sudo hopboxd admin mail show` ([Mail](/reference/mail/index.html.md#read-the-mail)).

- Without a method, it shows the command that makes a first-admin login link: `sudo hopboxd admin login-link --email <email>`.

A magic link or a login link opens its own page, often in a new tab. So the sign-in page keeps its return path in the local storage of the browser for 15 minutes, the lifetime of a link. The page of the link then opens that path.

Some steps need a recent sign-in, at most 10 minutes old ([Sessions, cookies, and sign-in links](/reference/sessions/index.html.md#recent-sign-in)). In the console, the approval of a CLI sign-in on `/device` is such a step. A call of such a step with an older session fails with `permission_denied` and the detail `RecentSignInRequired`. The page then links to `/signin?again=1&return_to=<path of the page>`. With `again=1`, the sign-in page shows the methods to a person who is signed in already. The new sign-in replaces the session of the browser, and the page of the step opens again.

The console checks a return path with the rules of the server. A good path has these properties:

- It is one path on the install, with at most 512 characters.
- It starts with one `/`.
- It has no backslash, no space, and no control character.
- It has no `.` or `..` segment.

The console drops any other return path, and opens `/`.

The pages of the links read the token from the fragment, then remove the fragment from the address bar and the history before the call. A browser never sends a fragment to a server, so the token never reaches a log or a `Referer` header.

The page of an invitation keeps its token in the session storage of the tab while the person signs in. A sign-in with GitHub or Google in the same tab then comes back to it. A magic link opens in another tab without the token. Go back to the first tab. When the tab gets the focus, it reads the account again and shows the accept button.

A call that fails with `unauthenticated` on a page that needs a session sends the browser to the sign-in page. "Sign out" calls `SignOut`, which ends the session and clears the cookie.

## Calls

The console calls the API with the TypeScript SDK `@hopbox/sdk` of the workspace, in its browser runtime ([TypeScript SDK](/reference/ts-sdk/index.html.md)). The session cookie authenticates each call, so the client has no credential. Each call is a Connect `POST` with the binary format.

The pages read and change data with TanStack Query and connect-query on the transport of the SDK. The transport already retries a call that is safe to run again, and puts an idempotency key on each create. A read call gets a retry after `unavailable`, `resource_exhausted`, or `deadline_exceeded`. A change without an idempotency key gets one only after an answer with `Hopbox-Not-Run: 1`, because `fetch` does not tell a refused connection from a lost answer ([TypeScript SDK](/reference/ts-sdk/index.html.md#retries)). So TanStack Query never retries a call itself. A read stays fresh for 15 seconds. A change reads the data of its service again.

## Live machine list

Each page of an org follows the machines of the org with the stream `WatchMachines`. The stream sends each machine, then a `synced` message, then each change. The console shows the first list at the `synced` message. A machine in the state `deleted`, or a `gone_machine_id`, leaves the list.

When the stream breaks, the console keeps the old list on show. It opens the stream again after 1, 2, 5, 10, and then 30 seconds. The new first list replaces the old one at its `synced` message, so a machine that went away in the meantime leaves the list then. The head of the org shows the state of the stream: `connecting`, `live`, or `reconnecting`. An `unauthenticated` error ends the stream and opens the sign-in page.

## Web terminal

The web terminal opens the login shell of the main user of a running machine, as `hopbox connect` does. It runs exec over the WebSocket endpoint `/api/ws/exec` with these start fields ([the machine API](/reference/machine-api/index.html.md)):

| Field | Value |
| --- | --- |
| `machine` | the name of the machine |
| `argv` | empty, for the login shell |
| `tty` | true |
| `size` | the rows and the columns of the terminal |
| `term` | `xterm-256color` |

The terminal is xterm.js. Each key goes to the shell as stdin, and each change of the size of the page sends a resize. The output of the shell goes to the terminal. When the shell ends, the terminal shows how it ended, such as `the shell ended with exit code 2`, and offers "Open a new shell". The button "Open terminal" needs the state `running`.

The browser sends the session cookie with the WebSocket, and the server checks its `Origin` header against `https://<domain>` ([Sessions, cookies, and sign-in links](/reference/sessions/index.html.md#calls-with-the-cookie)). A page of a machine URL thus cannot open a shell with the cookie of a person.

## Browser storage

The console keeps no credential in the storage of the browser. It keeps these keys:

| Key | Storage | Value | Lifetime |
| --- | --- | --- | --- |
| `hopbox.lastOrg` | local | the ID of the org that the browser opened last, for `/` | until "Sign out" |
| `hopbox.returnTo` | local | the return path of the last sign-in page, with its time | 15 minutes, and one read |
| `hopbox.invite` | session | the token of an invitation that waits for a sign-in | until the accept, "Not now", an accept error that a retry cannot fix, or the end of the tab |

A browser without storage loses only these conveniences.

## Headers

Each response of the console has these headers:

| Header | Value |
| --- | --- |
| `Content-Security-Policy` | `default-src 'none'; script-src 'self'; style-src 'self' 'unsafe-inline'; img-src 'self' data:; font-src 'self'; connect-src 'self'; manifest-src 'self'; base-uri 'none'; form-action 'self'; frame-ancestors 'none'` |
| `X-Content-Type-Options` | `nosniff` |
| `X-Frame-Options` | `DENY` |
| `Referrer-Policy` | `no-referrer` |
| `Cross-Origin-Opener-Policy` | `same-origin` |
| `Cross-Origin-Resource-Policy` | `same-origin` |
| `Permissions-Policy` | `camera=(), microphone=(), geolocation=(), payment=(), usb=()` |

The policy allows scripts only from the console itself: no inline script, no `eval`, and no other site. The build has no inline script and no data URL. The console loads nothing from a CDN. No site can show the console in a frame.

The fonts are files of the build, so `font-src 'self'` covers them. Each file is a variable font in WOFF2, so one file gives every weight that the console uses:

| Font | Characters | Size |
| --- | --- | --- |
| Bricolage Grotesque | the Latin subset that hopbox.dev uses | 75 KiB |
| Bricolage Grotesque | the Latin Extended subset that hopbox.dev uses, such as `ł` and `ő`. The browser gets this file only for a page that shows such a character. | 30 KiB |
| JetBrains Mono 2.304 | the Latin and Latin Extended-A characters, arrows, geometric shapes, dingbats such as `✓` and `❯`, box-drawing and block characters, and the Powerline symbols, for terminal programs such as `htop` and `tmux` | 45 KiB |

The policy allows inline styles. xterm.js writes style elements and style attributes while it runs, for the size of the cells and the colors of the text. Without inline styles, the terminal shows the text without its colors and its layout. An inline style cannot run code.

The cache headers depend on the file:

| File | `Cache-Control` |
| --- | --- |
| `index.html`, and each page of the app | `no-cache` |
| a file under `/assets/`, whose name holds the hash of its content | `public, max-age=31536000, immutable` |
| a file that does not exist | `no-store` |

Each file has an `ETag` from its SHA-256, and a request with a matching `If-None-Match` gets `304 Not Modified`. A file of text, script, JSON, or SVG of at least 1 KiB also has a gzip copy. A request with `Accept-Encoding: gzip` gets that copy with `Content-Encoding: gzip` and `Vary: Accept-Encoding`, unless it asks for a range.

## Build

`make console` builds the console:

1. It builds the SDK into `web/packages/sdk/dist/` with `tsc`.
2. It type-checks the console with `tsc`.
3. Vite writes the build to `internal/console/dist/`: `index.html`, and the hashed files under `assets/`, the icon and the two fonts among them. The terminal and xterm.js load in a chunk of their own, only on the terminal page.

`make build` runs `make console` first, so each `hopboxd` holds the console of its own commit. `tools/bundle` refuses a `hopboxd` without the console, so no release ships without it ([Release bundle](/reference/release-bundle/index.html.md)). `internal/console` embeds `internal/console/dist/` with `go:embed`. Git ignores the build, apart from the empty file `.gitkeep`, which `go:embed` needs. The same lockfile gives the same files, so the build is reproducible.

A `hopboxd` built without `make console`, for example with `go build`, holds no console. Its API listener answers each path of the console with `404`. The text is `This hopboxd has no console. Build it with make console, then make build.` At its start, `hopboxd run` logs the record `this hopboxd has no console. Build it with make console, then make build` at level warn.

The console pins each package to an exact version in `web/apps/console/package.json`:

| Package | Version |
| --- | --- |
| `react`, `react-dom` | 19.3.1 |
| `@tanstack/react-router` | 1.170.45 |
| `@tanstack/react-query` | 5.104.5 |
| `@connectrpc/connect-query` | 2.3.1 |
| `@xterm/xterm` | 6.0.0 |
| `@xterm/addon-fit` | 0.11.0 |
| `vite` | 8.3.4 |
| `typescript` | 7.0.2 |
| `vitest` | 5.0.5 |
| `@playwright/test` | 1.64.1 |

## Tests

| Test | What it checks | Command |
| --- | --- | --- |
| `web/apps/console/test/*.test.ts` | With vitest: the rules of the return path and the browser storage, the live machine list, the labels, and the error texts. Also the clicks that close a dialog, and the colors that the web terminal takes from the tokens. | `make test-console`, or `node_modules/.bin/vitest run` in `web/apps/console` |
| `internal/console` | the routes, the headers, the cache, gzip, and the page of a binary without the console | `make test` |
| `TestConsoleTakesOnlyTheFreePaths` in `internal/api` | that the console hides no API route | `make test` |
| `TestConsole` in `internal/control` | the whole console in Chromium with Playwright, `web/apps/console/e2e` | `make test-console` only |

`TestConsole` runs a control role with a node that runs machines with the real guest agent. It serves the API with the console of the test binary at `https://hop.example.org`. An HTTP proxy of the test sits in front, and ends TLS with a certificate that the test makes. Chromium reaches the console through that proxy. The suite:

1. Checks the headers and the sign-in page without a session, and checks that both fonts load under the policy.
2. Signs in with a login link, comes back to the page that asked for the sign-in, and finds `box1` running. It checks that the scrolling row of tabs does not cut off the focus ring of a tab or the underline of the current tab.
3. Creates, stops, starts, and deletes a machine, and checks the name suggestion of a taken name.
4. Makes the URL of `box1` public, then private.
5. Runs `echo` in the web terminal, then `exit`. Before that, it checks at 844 by 390 and 1280 by 560 pixels that the tabs of the org keep their height.
6. Adds and removes an SSH key, creates an API key, and sets an env var.
7. Opens each dialog, and checks it at a desktop and a phone width. The title starts at the left edge, the text stays inside the dialog, and the buttons sit at the bottom right. The dialogs are "New machine", "Delete box1", "Revoke the API key", "Remove the SSH key", "Remove the member", "Leave Team", and "Delete Team".
8. Checks that a click on the backdrop and the Escape key close a dialog, and that a click or a drag inside it does not.
9. Creates an org, and finds it in the org switcher.
10. Signs out, and checks that each page then asks for a sign-in.
11. Opens `/device?code=<code>` without a session, signs in with a second login link, comes back to the code, and approves it.
12. Makes the session 11 minutes old, and opens `/device?code=<code>` with a second code. The approval fails and asks for a sign-in again. The suite signs in with a third login link, comes back to the code, and approves it. After the run, the Go test polls both device sign-ins and gets the CLI tokens of the account.
13. Checks that no page error and no policy violation happened.

`TestConsole` gives Playwright the install in env vars, so you do not set them yourself. `HOPBOX_E2E_PROXY` is the proxy that serves the console and the API. `HOPBOX_E2E_LOGIN_LINK`, `HOPBOX_E2E_LOGIN_LINK_2`, and `HOPBOX_E2E_LOGIN_LINK_3` are the three login links. `HOPBOX_E2E_USER_CODE` and `HOPBOX_E2E_USER_CODE_2` are the codes of the two device sign-ins. A `POST` to `HOPBOX_E2E_AGE_SESSIONS`, a URL on loopback, makes every session of the install 11 minutes old. `HOPBOX_E2E_TEAM_ORG` is the ID of the org `Team`, where `bob@example.org` is a member of the account of the suite. The others are `HOPBOX_E2E_URL`, `HOPBOX_E2E_SSH_KEY`, and `HOPBOX_E2E_MACHINE`.

Without `HOPBOX_TEST_CONSOLE`, `TestConsole` skips. `make test-console` sets the variable, and `make test` does not, so `make test` needs no browser. The test needs `corepack pnpm install`, a build of the console, and the Chromium of Playwright: run `web/apps/console/node_modules/.bin/playwright install chromium` once. If one of them is missing, the test fails.
